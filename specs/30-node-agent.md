# WS-30 — node-agent (C++ telemetry agent + Helm chart)

Read first: `00-overview.md`, then `01-contracts.md` (especially §2, §5, §6, §7, §8).

## 1. Mission
Build a small, efficient C++ daemon, together with its image and a Helm chart. The chart runs one copy on every `workload=app` node. Every 1 s the daemon samples host CPU and memory from `/proc`, keeps a rolling window of statistics, and exposes all of it in Prometheus format on `:9102/metrics`. The chart also registers a PodMonitor that scrapes every 1 s and labels the series with the node name.

## 2. You own
`agents/node-agent/**` and `deploy/helm/node-agent/**`. Do not create or edit files anywhere else.

## 3. Deliverables
```
agents/node-agent/
├── Makefile                  # lint, test, image (contracts §7.1); also build, test-asan, integration
├── CMakeLists.txt
├── CMakePresets.json         # presets: release, debug, asan
├── cmake/Dependencies.cmake  # FetchContent: prometheus-cpp, googletest (pinned tags)
├── .clang-format  .clang-tidy
├── Dockerfile  .dockerignore
├── README.md  VERSIONS.md  HANDOFF.md
├── src/
│   ├── main.cpp              # parse config, start exposer + collector, signal thread
│   ├── config.hpp/.cpp       # flags + env (§4.2)
│   ├── procfs.hpp/.cpp       # pure parsers: parse_stat(std::string_view), parse_meminfo(std::string_view)
│   ├── cpu.hpp/.cpp          # CpuTimes, delta → usage; reset detection
│   ├── memory.hpp/.cpp       # MemInfo → derived values
│   ├── window.hpp            # template RingWindow<T> with min/avg/max
│   ├── scheduler.hpp/.cpp    # drift-free ticker with injectable clock
│   ├── collector.hpp/.cpp    # owns state; one tick = read → compute → publish
│   ├── metrics.hpp/.cpp      # prometheus-cpp registry and families (names per contracts §5)
│   └── log.hpp/.cpp          # JSON-lines logger with rate limiting
└── tests/
    ├── fixtures/proc/{host-2cpu,host-64cpu,old-kernel,no-memavailable}/{stat,meminfo}
    ├── test_procfs.cpp  test_cpu.cpp  test_memory.cpp  test_window.cpp  test_scheduler.cpp
    └── integration_test.sh
deploy/helm/node-agent/
├── Chart.yaml  values.yaml  values-local.yaml  README.md
└── templates/
    ├── _helpers.tpl
    ├── daemonset.yaml
    ├── priorityclass.yaml     # guarded by .Values.priorityClass.create
    └── podmonitor.yaml        # guarded by .Values.podMonitor.enabled
```

## 4. Requirements

### 4.1 Behaviour
- AGT-1 **Inputs**: every tick, read `<procfs>/stat` and `<procfs>/meminfo`. `procfs` defaults to `/host/proc`, which is the host's `/proc` mounted read-only (the same pattern node-exporter uses with `--path.procfs`). Keep the file descriptors open and use `pread(fd, buf, n, 0)` each tick. Re-open a file if a read fails.
- AGT-2 **Scheduling** is drift-free on `std::chrono::steady_clock`: `next += interval; sleep_until(next)`. If `now > next` after a cycle, advance `next` to the first future grid point, and add the number of skipped ticks to `node_agent_missed_ticks_total`. Never run back-to-back catch-up samples. Inject the clock and sleeper as an interface so they can be tested.
- AGT-3 **CPU math** applies to the aggregate `cpu` line and to every `cpuN` line:
  - Fields: `user nice system idle iowait irq softirq steal [guest guest_nice]`. Older kernels can have fewer columns; treat missing ones as 0.
  - `idle_all = idle + iowait`
  - `total = user + nice + system + idle + iowait + irq + softirq + steal`. **Exclude guest and guest_nice**, because they are already included in user and nice.
  - `usage = (Δtotal − Δidle_all) / Δtotal`, clamped to [0, 1].
  - If `Δtotal == 0`, keep the previous published value.
  - If any field goes **backwards**, re-baseline that CPU and publish nothing for it on that tick.
  - The **first** tick only sets the baseline. The `*_usage_ratio` gauges must not exist until the first valid delta (contracts §5).
  - When a CPU appears, create its `cpu="N"` child gauge. When a CPU disappears, remove it. `node_agent_cpu_count` = the number of `cpuN` lines.
  - `node_agent_cpu_seconds_total{mode}` is the aggregate jiffies ÷ `sysconf(_SC_CLK_TCK)`. prometheus-cpp counters can only increment, so start the counter at 0 when the agent starts and call `Increment(Δjiffies / CLK_TCK)` on every valid tick; skip the update on a reset. Only `rate()` and `increase()` of this counter mean anything, not its absolute value. Prometheus handles counters that restart from 0 when the process restarts.
- AGT-4 **Memory math**: parse `MemTotal, MemFree, MemAvailable, Buffers, Cached, SwapTotal, SwapFree`, which are in kB, so multiply by 1024. `used = MemTotal − MemAvailable`, and `ratio = used / MemTotal`. If `MemTotal` or `MemAvailable` is missing, count an error (`source="meminfo"`) and skip the memory update for that tick.
- AGT-5 **Rolling window**: `RingWindow<double>` with `capacity = window_seconds × 1000 / interval_ms` (default 60). It holds the node CPU ratio and the memory ratio of each tick. After every tick, publish `min`, `avg` and `max` for both (an O(capacity) recompute is fine), plus `node_agent_window_samples` and `node_agent_window_capacity`. The window catches peaks that a 1 s scrape can miss through phase misalignment.
- AGT-6 **Self metrics**:
  - After each successful tick: set `last_sample_timestamp_seconds` (system clock), increment `collections_total`, and observe `collection_duration_seconds`.
  - At startup: set `build_info{version,git_sha,compiler}` to 1.
- AGT-7 **Concurrency**: the collector thread is the only writer to agent state and metrics. The HTTP threads (prometheus-cpp Exposer / civetweb) only read through the thread-safe registry. Agent code has no other shared mutable state and no locks.
- AGT-8 **Failure policy**:
  - A read or parse error on one file counts an error, logs a WARN (rate-limited to 1 every 10 s per source), skips that part, and keeps running.
  - An uncaught exception in the collector, or a failure to bind the listen address, logs an ERROR and exits with code 1.
- AGT-9 **Signals**: block SIGTERM and SIGINT in all threads, and handle them on a dedicated thread with `sigwait`. That thread sets the stop flag and notifies the collector's condition variable, so the sleep ends early. Stop the Exposer and exit 0 within 1 s.
- AGT-10 **Performance** on t3.large: average CPU under 5 millicores, RSS under 20 MiB, p99 collection time under 1 ms. Parse with `std::from_chars` over a reused buffer, and avoid heap allocation in the steady-state tick path where practical.

### 4.2 Configuration
Flags take precedence over environment variables.

| Flag | Env | Default | Validation |
|---|---|---|---|
| `--listen-address` | `NODE_AGENT_LISTEN` | `0.0.0.0:9102` | host:port |
| `--procfs` | `NODE_AGENT_PROCFS` | `/host/proc` | directory with readable `stat` and `meminfo` |
| `--interval-ms` | `NODE_AGENT_INTERVAL_MS` | `1000` | 100–60000 |
| `--window-seconds` | `NODE_AGENT_WINDOW_SECONDS` | `60` | ≥ interval, ≤ 3600 |
| `--log-level` | `NODE_AGENT_LOG_LEVEL` | `info` | debug, info, warn, error |
| `--version` | – | – | prints `node-agent <version> (<git_sha>)` and exits 0 |
| `--help` | – | – | prints usage and exits 0 |

Invalid config prints an error and exits 2. `NODE_NAME` (from the Downward API) appears only in logs.

**Logs** are JSON lines on stderr: `{"ts":"…","level":"info","msg":"…","node":"…", …}`. Log the effective config at startup.

### 4.3 Build
- B-1: C++20, CMake ≥ 3.25, GCC 13+ (Clang 17+ should also work). Flags: `-Wall -Wextra -Wpedantic -Werror`. The Release preset adds `-O2`.
- B-2: Dependencies come through FetchContent, pinned to exact tags:
  - **prometheus-cpp** with `ENABLE_PUSH=OFF`, `ENABLE_COMPRESSION=OFF`, `ENABLE_TESTING=OFF`, `USE_THIRDPARTY_LIBRARIES=ON`, linking `prometheus-cpp::core` and `prometheus-cpp::pull`
  - **GoogleTest**, for tests only
  - nothing else
- B-3: Preset `asan`: `-fsanitize=address,undefined -fno-omit-frame-pointer`, Debug.
- B-4: `VERSION` and `GIT_SHA` are passed in as CMake cache variables and compiled into a generated header.
- B-5: `make lint` runs `clang-format --dry-run --Werror` over src and tests, plus `clang-tidy` (`bugprone-*`, `performance-*`, `modernize-*`) on src.

### 4.4 Tests (`make -C agents/node-agent test` runs release + asan unit tests + integration)
- T-1 **Parsers**: every fixture parses. A 64-CPU host gives 64 per-CPU entries. A kernel without `guest_nice` parses. A meminfo without `MemAvailable` is reported as missing. Garbage input returns an error without crashing (add a small table of malformed inputs).
- T-2 **CPU**:
  - Hand-computed sample pairs give the expected ratio (0%, 50%, 100%).
  - Guest time is not double-counted.
  - A counter going backwards gives a re-baseline and nothing published.
  - `Δtotal = 0` keeps the previous value.
  - The first sample publishes nothing.
  - A CPU that appears or disappears creates or removes its child gauge.
- T-3 **Memory**: the derived values and the ratio are correct, and the missing-key path is handled.
- T-4 **Window**: empty, partial, full and wrapped states; min/avg/max are correct; `samples` is reported correctly.
- T-5 **Scheduler** with a fake clock: normal ticks stay on the grid; a 3.5× overrun counts 3 missed ticks and the next tick lands on the grid; a stop request wakes the sleep immediately.
- T-6 **Integration** (`integration_test.sh`):
  1. Copy fixtures to a temp dir.
  2. Start the binary with `--procfs <tmp> --interval-ms 100 --listen-address 127.0.0.1:<free port>`.
  3. Rewrite the `stat` and `meminfo` files with advancing counters that imply 25% CPU and 40% memory.
  4. Scrape `/metrics`, and assert the ratios are within ±0.01, the window gauges are present, `collections_total` is increasing, and the ratio gauges were absent on a scrape taken before the second tick.
  5. Run `promtool check metrics` on the output (skip with a message if promtool is missing).
  6. Send SIGTERM and assert exit 0 within 1 s.
- T-7: All unit tests pass under the asan preset.

### 4.5 Container image
- IMG-1: Multi-stage build. The builder is `gcc:13-bookworm` with `cmake` installed; it builds the release preset. Link libstdc++ and libgcc statically (`-static-libstdc++ -static-libgcc`). The runtime image is `gcr.io/distroless/cc-debian12:nonroot`, pinned by digest.
- IMG-2: `USER 65532:65532`, `ENTRYPOINT ["/node-agent"]`, `EXPOSE 9102`.
- IMG-3: The final image is under 30 MB. It works with `--read-only` and no capabilities.

### 4.6 Helm chart
- K-1: The **DaemonSet** pod has the label `app.kubernetes.io/name: node-agent`, and uses `updateStrategy: RollingUpdate` with `maxUnavailable: 1`.
- K-2: `nodeSelector` comes from values (default `{workload: app}`, the same label that time-api uses, so the agent runs wherever the app runs). `tolerations: [{operator: Exists}]`.
- K-3: The chart creates a **PriorityClass** `node-agent-critical` (value `1000000`, `globalDefault: false`) and the pods use it.
- K-4: Volume `hostPath: {path: /proc, type: Directory}`, mounted read-only at `/host/proc`. **No** `hostNetwork`, `hostPID` or `privileged`. The security baseline from contracts §8 applies, with `runAsUser: 65532` and `runAsGroup: 65532`.
- K-5: Env: `NODE_NAME` from `spec.nodeName`. Port `9102` named `metrics`.
- K-6: Probes: liveness `tcpSocket` on `metrics` (period 10 s, failureThreshold 3); readiness `httpGet /metrics` on `metrics` (period 10 s). The image is distroless, so exec probes are not possible.
- K-7: Resources: requests `cpu: 10m`, `memory: 16Mi`; limits `memory: 64Mi`.
- K-8: **PodMonitor** (when `podMonitor.enabled`):
  ```yaml
  spec:
    jobLabel: app.kubernetes.io/name
    namespaceSelector: {matchNames: [monitoring]}
    selector: {matchLabels: {app.kubernetes.io/name: node-agent}}
    podMetricsEndpoints:
      - port: metrics
        interval: 1s
        scrapeTimeout: 800ms
        relabelings:
          - sourceLabels: [__meta_kubernetes_pod_node_name]
            targetLabel: node
  ```
  `interval` and `scrapeTimeout` are values-driven, with these defaults.
- K-9: `helm lint` and `kubeconform` pass with the default values, with `values-local.yaml`, and with `podMonitor.enabled=false`.

## 5. Interfaces you provide
- `make -C agents/node-agent lint test image IMAGE=… VERSION=… GIT_SHA=…`
- Every metric in contracts §5 with the exact names, types, labels and semantics, scraped as `job="node-agent"` with a `node` label.

## 6. Standalone testing
```bash
make -C agents/node-agent lint test
make -C agents/node-agent image IMAGE=node-agent:dev
docker run --rm --read-only --cap-drop ALL -v /proc:/host/proc:ro -p 9102:9102 node-agent:dev
curl -s localhost:9102/metrics | grep ^node_agent_
# Run `stress-ng --cpu 1 -t 20` in another shell and watch node_agent_cpu_usage_ratio rise.

# kind (contracts §8.1), with no Prometheus Operator:
kind load docker-image node-agent:dev --name time-telemetry
helm install node-agent deploy/helm/node-agent -n monitoring --create-namespace \
  -f deploy/helm/node-agent/values-local.yaml \
  --set image.repository=node-agent --set image.tag=dev --set podMonitor.enabled=false --wait
kubectl -n monitoring get ds node-agent     # DESIRED 3 / READY 3
kubectl -n monitoring port-forward ds/node-agent 9102 & curl -s localhost:9102/metrics | head
```
To check accuracy against a reference on a Linux host, run node-exporter beside the agent and compare `node_agent_cpu_usage_ratio` with `1 − rate(node_cpu_seconds_total{mode=~"idle|iowait"}[1m])` averaged across CPUs. Over 1 min they should agree within 0.05.

## 7. Definition of Done
- [ ] `make -C agents/node-agent lint test` is green: release, asan and integration.
- [ ] The image is under 30 MB, runs read-only and non-root with no capabilities, and serves every metric in contracts §5.
- [ ] Measured on a Linux host or the kind cluster, with the numbers recorded in `HANDOFF.md`: CPU under 5 m, RSS under 20 MiB, p99 collection time under 1 ms.
- [ ] The chart passes K-9, and on kind the DaemonSet is 3/3 Ready.
- [ ] `README.md`, `VERSIONS.md` and `HANDOFF.md` are written (contracts §11).

## 8. Out of scope
Recording and alert rules and dashboards (WS-40), Prometheus configuration (WS-40), pushing images (WS-50), and push-based delivery. GPU, disk and network metrics are also out of scope.

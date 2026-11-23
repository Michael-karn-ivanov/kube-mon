# 01 — Shared Contracts

**Every agent must read this file.** It fixes the interfaces between workstreams, so each component can be built and tested without waiting for the others.

**No agent may change this file.** If a contract is wrong or missing something, record a *Contract Change Request* in your `HANDOFF.md` (§11) and keep coding against the current contract.

---

## 1. Path ownership

Every path in the repo has exactly one owner. Agents create and edit files **only** under the paths they own, so parallel branches merge without conflicts.

| Workstream | Spec | Owns (exclusive) |
|---|---|---|
| WS-10 Infra | `10-infra.md` | `infra/terraform/**` |
| WS-20 time-api | `20-time-api.md` | `apps/time-api/**`, `deploy/helm/time-api/**` |
| WS-30 node-agent | `30-node-agent.md` | `agents/node-agent/**`, `deploy/helm/node-agent/**` |
| WS-40 Observability | `40-observability.md` | `deploy/monitoring/**`, `deploy/helm/observability/**` |
| WS-50 Integration | `50-integration.md` | `Makefile`, `deploy.env.example`, `.gitignore`, `README.md`, `versions.md`, `scripts/**`, `.github/**` |
| Spec owner (Misha) | – | `specs/**` |

There is one rule of thumb for monitoring config. The component that **emits** metrics owns how they are **scraped** (its ServiceMonitor or PodMonitor). WS-40 owns what is **computed and shown** from them: recording rules, alerts and dashboards.

---

## 2. Names, namespaces, ports

| Item | Value |
|---|---|
| AWS region (default) | `eu-west-2` |
| EKS cluster name | `time-telemetry-dev` |
| Worker node label | `workload=app` (every node in the managed node group, and every kind worker) |
| Namespace: app | `time-api` |
| Namespace: monitoring, agent, observability | `monitoring` |
| Helm release / chart: monitoring stack | `kps` / `prometheus-community/kube-prometheus-stack` with `fullnameOverride: kps` |
| Helm release / chart: observability | `observability` / `deploy/helm/observability` |
| Helm release / chart: agent | `node-agent` / `deploy/helm/node-agent` |
| Helm release / chart: app | `time-api` / `deploy/helm/time-api` |
| Resulting service names | `svc/kps-prometheus` (9090), `svc/kps-grafana` (80), `svc/kps-alertmanager` (9093) |
| Prometheus pod | `prometheus-kps-prometheus-0` |
| ECR repositories | `time-api`, `node-agent` |
| time-api container ports | `8000` named `http`; `9000` named `metrics` |
| time-api public port | `80` on the NLB → `http` |
| node-agent container port | `9102` named `metrics` |
| Pod label used for selection and `job` | `app.kubernetes.io/name: time-api` / `node-agent` |
| Grafana dashboard UIDs | `node-telemetry`, `cluster-telemetry` |
| Grafana dashboard ConfigMap label | `grafana_dashboard: "1"` |
| Grafana admin Secret | `monitoring/grafana-admin`, keys `admin-user`, `admin-password` |
| Deploy order | kps → observability → node-agent → time-api (the CRDs must exist first) |

---

## 3. time-api HTTP contract

`GET /time`
- Returns `200`, `Content-Type: application/json`, `Cache-Control: no-store`, `X-Served-By: <pod name>`.
- Body:
  ```json
  {"time": "2026-11-23T09:45:00.123456Z", "epoch_ms": 1795427100123, "timezone": "UTC"}
  ```
  `time` is RFC 3339 UTC with microseconds and a `Z` suffix. `epoch_ms` is an integer number of ms since the epoch, taken from the **same** clock reading as `time`.
- Any other method returns `405`. Unknown paths return `404` with a JSON body `{"detail": "..."}`.

`GET /healthz` returns `200 {"status":"ok"}` (liveness).
`GET /readyz` returns `200 {"status":"ready"}`, or `503 {"status":"shutting_down"}` after SIGTERM.
`GET /metrics` is served on port **9000 only**. Port 8000 returns `404` for `/metrics`.

---

## 4. time-api metrics

| Metric | Type | Labels |
|---|---|---|
| `time_api_requests_total` | counter | `method`, `route` (route template such as `/time`, or `unmatched`), `status` (such as `200`) |
| `time_api_request_duration_seconds` | histogram | `route`; buckets `[0.001, 0.0025, 0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1]` |
| `time_api_build_info` | gauge (=1) | `version`, `git_sha` |

`/healthz`, `/readyz` and `/metrics` requests are **not** counted. Default `prometheus_client` process and GC metrics are allowed.

---

## 5. node-agent metrics

The agent itself never adds a `node` label. Prometheus adds it (§6). Units follow Prometheus conventions.

| Metric | Type | Labels | Meaning |
|---|---|---|---|
| `node_agent_cpu_usage_ratio` | gauge | – | Node-wide CPU busy fraction over the last sampling interval (0–1) |
| `node_agent_cpu_core_usage_ratio` | gauge | `cpu` (`"0"`,`"1"`,…) | Same, per logical CPU |
| `node_agent_cpu_count` | gauge | – | Online logical CPUs (`cpuN` lines in `/proc/stat`) |
| `node_agent_cpu_seconds_total` | counter | `mode` ∈ `user,nice,system,idle,iowait,irq,softirq,steal` | Node-wide CPU time = jiffies ÷ `sysconf(_SC_CLK_TCK)` |
| `node_agent_memory_total_bytes` | gauge | – | `MemTotal` |
| `node_agent_memory_available_bytes` | gauge | – | `MemAvailable` |
| `node_agent_memory_free_bytes` | gauge | – | `MemFree` |
| `node_agent_memory_buffers_bytes` | gauge | – | `Buffers` |
| `node_agent_memory_cached_bytes` | gauge | – | `Cached` |
| `node_agent_memory_used_bytes` | gauge | – | `MemTotal − MemAvailable` |
| `node_agent_memory_usage_ratio` | gauge | – | used / total |
| `node_agent_swap_total_bytes` | gauge | – | `SwapTotal` |
| `node_agent_swap_free_bytes` | gauge | – | `SwapFree` |
| `node_agent_cpu_usage_ratio_window` | gauge | `stat` ∈ `min,avg,max` | Over the in-agent rolling window (default 60 samples) |
| `node_agent_memory_usage_ratio_window` | gauge | `stat` ∈ `min,avg,max` | Same, for memory usage ratio |
| `node_agent_window_samples` | gauge | – | Samples currently in the window |
| `node_agent_window_capacity` | gauge | – | Window capacity in samples |
| `node_agent_last_sample_timestamp_seconds` | gauge | – | Unix time of the last successful sample |
| `node_agent_collections_total` | counter | – | Successful collection cycles |
| `node_agent_collection_errors_total` | counter | `source` ∈ `stat,meminfo` | Failed reads or parses |
| `node_agent_missed_ticks_total` | counter | – | Ticks skipped because a cycle overran |
| `node_agent_collection_duration_seconds` | histogram | – | Wall time of one cycle; buckets `[5e-5,1e-4,2.5e-4,5e-4,1e-3,2.5e-3,5e-3,1e-2,5e-2]` |
| `node_agent_build_info` | gauge (=1) | `version`, `git_sha`, `compiler` | Build metadata |

Semantics every consumer can rely on:
- The usage ratio gauges are **absent** (not 0) until the agent has two samples to compare.
- Sampling interval: 1 s. Scrape interval: 1 s.

---

## 6. Prometheus labels: who sets what

| Label | Value | Set by |
|---|---|---|
| `job` | `node-agent` / `time-api` | `jobLabel: app.kubernetes.io/name` in the PodMonitor (WS-30) or ServiceMonitor (WS-20) |
| `node` on node-agent series | Kubernetes node name | PodMonitor relabeling `__meta_kubernetes_pod_node_name → node` (WS-30) |
| `node` on node-exporter series | Kubernetes node name | kps values relabeling on the node-exporter ServiceMonitor (WS-40) |
| `namespace`, `pod`, `instance`, `container`, `endpoint` | standard | Prometheus Operator |

Rules and dashboards MUST key on `node`, never on `instance` or `pod`.

Scrape intervals: node-agent **1 s** (scrape timeout 800 ms). time-api **15 s**. Everything else uses the global 30 s.

---

## 7. Build and packaging interface

### 7.1 Component Makefiles
`apps/time-api/Makefile` and `agents/node-agent/Makefile` MUST provide these targets, which the top-level Makefile calls:

| Target | Behaviour |
|---|---|
| `lint` | static checks; exits non-zero on findings |
| `test` | all unit and integration tests for the component (node-agent: including the asan build) |
| `image` | `docker buildx build --platform $(PLATFORM) --build-arg VERSION=$(VERSION) --build-arg GIT_SHA=$(GIT_SHA) -t $(IMAGE) --load .` |

Variables, overridable and with these defaults: `IMAGE ?= <component>:dev`, `PLATFORM ?= linux/amd64`, `VERSION ?= 0.0.0-dev`, `GIT_SHA ?= unknown`.

Component Makefiles **never push** images. Pushing is done by the top-level Makefile.

`infra/terraform/Makefile` provides `fmt-check`, `validate` and `lint` (tflint), and never applies.
`deploy/helm/observability/Makefile` provides `test` (promtool and dashboard checks).

### 7.2 Image rules
- Images are tagged with the git short SHA (12 chars), and tags are immutable. `latest` is never used.
- Architecture is `linux/amd64`.
- Images run as a non-root user and work with `readOnlyRootFilesystem: true`.
- Both Dockerfiles accept `ARG VERSION` and `ARG GIT_SHA` and expose them through the `*_build_info` metric.

---

## 8. Helm chart conventions (WS-20, WS-30 and WS-40 charts)

- Helm 3, `apiVersion: v2`. Chart `version` starts at `0.1.0`; `appVersion` is overridden at deploy time.
- Charts **do not** create namespaces. The namespaces come from Terraform on EKS, or from `--create-namespace` on kind.
- Required values keys, so the Makefile can drive them:
  - `image.repository`, `image.tag`, `image.pullPolicy` (default `IfNotPresent`): WS-20 and WS-30
  - `nodeSelector` (default `{workload: app}`): WS-20 and WS-30
  - `serviceMonitor.enabled` (WS-20) and `podMonitor.enabled` (WS-30), default `true`. When `false`, the chart installs on a cluster that has no Prometheus Operator CRDs.
  - `service.type` (default `LoadBalancer`) and `service.loadBalancerSourceRanges` (default `["0.0.0.0/0"]`): WS-20
- Each chart has a `values-local.yaml` for kind.
- Standard labels on every object: `app.kubernetes.io/name`, `app.kubernetes.io/instance`, `app.kubernetes.io/version`, `app.kubernetes.io/part-of: time-telemetry`, `helm.sh/chart`.
- Security baseline for every container: `runAsNonRoot: true`, a fixed non-root `runAsUser`, `readOnlyRootFilesystem: true`, `allowPrivilegeEscalation: false`, `capabilities.drop: [ALL]`, `seccompProfile.type: RuntimeDefault`.
- Every chart must pass `helm lint` and `helm template … | kubeconform -strict -ignore-missing-schemas`. The schemas for monitoring.coreos.com CRDs may be ignored or provided.

### 8.1 Standard local cluster (kind)
Every agent may create this cluster for standalone testing. WS-50 owns the canonical copy at `scripts/kind-config.yaml`.
```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: time-telemetry
nodes:
  - role: control-plane
  - {role: worker, labels: {workload: app}}
  - {role: worker, labels: {workload: app}}
  - {role: worker, labels: {workload: app}}
```
On kind, `/proc/stat` and `/proc/meminfo` are host-wide, so all 3 workers report identical node numbers. That is expected.

**When agents run in parallel on one machine**, each workstream uses its own cluster name for standalone testing: `kind create cluster --name tt-ws<NN> --config <copy of the above>`, then `kind load docker-image … --name tt-ws<NN>`. The canonical name `time-telemetry` is reserved for WS-50's `make local-up`. See §12.

---

## 9. Terraform interface (between WS-10 and WS-50)

| Stack | Path | State |
|---|---|---|
| bootstrap | `infra/terraform/bootstrap` | local |
| cluster | `infra/terraform/cluster` | S3, key `cluster/terraform.tfstate` |
| platform | `infra/terraform/platform` | S3, key `platform/terraform.tfstate` |

- The S3 stacks declare a partial backend: `backend "s3" { key = "<stack>/terraform.tfstate"; use_lockfile = true; encrypt = true }`. `bucket` and `region` come from `-backend-config=../backend.hcl`. That file is gitignored, written by the Makefile, and has the format `bucket = "…"` / `region = "…"`.
- **bootstrap**: inputs `region` (default `eu-west-2`) and `bucket_prefix` (default `time-telemetry-tfstate`). Output: `state_bucket_name` (`<prefix>-<account_id>-<region>`).
- **cluster** inputs:

  | Variable | Type | Default |
  |---|---|---|
  | `region` | string | `eu-west-2` |
  | `cluster_name` | string | `time-telemetry-dev` |
  | `cluster_version` | string | newest EKS standard-support version at implementation time |
  | `node_instance_type` | string | `t3.large` |
  | `node_count` | number | `3` |
  | `operator_cidrs` | list(string) | **required** |
  | `cluster_admin_arns` | list(string) | `[]` |
  | `single_nat_gateway` | bool | `true` |
  | `tags` | map(string) | `{project="time-telemetry", env="dev", owner="misha"}` |

- **cluster** outputs: `cluster_name`, `region`, `vpc_id`, `ecr_time_api_url`, `ecr_node_agent_url`, `kubeconfig_command`.
- **platform** inputs: `region`, `state_bucket`. It reads the cluster outputs through `terraform_remote_state`. Output: `storage_class_name` (`gp3`).
- **platform** creates the namespaces `time-api` and `monitoring`, the default StorageClass `gp3`, and the AWS Load Balancer Controller.
- The Makefile passes variables as `TF_VAR_*` environment variables, with lists as JSON (for example `TF_VAR_operator_cidrs='["1.2.3.4/32"]'`).
- Every AWS resource carries the `tags` through the provider's `default_tags`.

---

## 10. Versions policy
- Use the latest stable release of everything at implementation time, and pin it exactly. No floating tags (`latest`, `main`, `^`, `~>` with a major wildcard) for images, charts, modules or FetchContent.
- Each workstream records its pins in `<owned dir>/VERSIONS.md` as a table of name, version and where it is pinned. WS-50's root `versions.md` links to all of them.
- The kube-prometheus-stack chart version is written by WS-40, alone, to `deploy/monitoring/chart-version.txt`. The Makefile reads it from there.
- Minimums: Python 3.12, GCC 13 / C++20, CMake 3.25, Terraform 1.11, Helm 3.14.

---

## 11. Hand-off protocol

When finished, each agent writes `<primary owned dir>/HANDOFF.md` containing:
1. **What was built**: the file tree, one line per notable file.
2. **How to verify**: the exact commands, and their output (or a summary of it).
3. **Definition of Done checklist** from its spec, each item ticked or explained.
4. **Deviations**: anything implemented differently from the spec, and why.
5. **Contract Change Requests**: a proposed change to `01-contracts.md` with its reason. Put `none` if there are none.
6. **Open issues / follow-ups.**

---

## 12. Working rules for agents running in parallel

Up to five agents may run at the same time on the same machine, each in its own git worktree. To avoid interfering with each other:

| Resource | Rule |
|---|---|
| Git | Work only in your own worktree and branch (`ws/<NN>-<name>`). Commit often to that branch. **Never** push, merge, rebase onto `main`, or touch other branches. |
| Files | Only your owned paths (§1). Don't edit `specs/`. |
| kind | Use only the cluster `tt-ws<NN>`. Never create, delete or switch to another agent's cluster, and never delete all clusters. Always pass an explicit `--context kind-tt-ws<NN>` / `--kube-context`. Delete your cluster when you finish. |
| Local ports | For `docker run -p` and `kubectl port-forward`, use **only** local ports `1NN00`–`1NN99` (WS-20 → 12000–12099, WS-30 → 13000–13099, WS-40 → 14000–14099, WS-50 → 15000–15099). Tests that start their own servers must pick a free ephemeral port. |
| Docker | Tag images `<component>:ws<NN>-<anything>`. Never run `docker system prune`, `docker rm -f $(docker ps -aq)` or anything similar that affects containers or images you didn't create. |
| Cloud | Create no AWS resources unless your spec **and** your launch prompt explicitly allow it. Read-only AWS calls (`sts get-caller-identity`, `terraform plan`) are fine if credentials are present. |
| Tools | If a required tool is missing, install it user-locally (for example `~/.local/bin`, `uv tool`, `pipx`, or a pinned release binary). Do not use `sudo` or change global system configuration. If you can't install it, mark the dependent checks as skipped, with the reason, in `HANDOFF.md`. |
| Long-running processes | Stop everything you started (port-forwards, containers, kind cluster) before you finish. |
| Questions | Don't stop to wait for answers. Make the most reasonable choice consistent with the specs, record it under *Deviations* in `HANDOFF.md`, and carry on. |

# WS-40 — Observability (Prometheus, Grafana, rules, dashboards)

Read first: `00-overview.md`, then `01-contracts.md` (especially §2, §4, §5, §6, §10).

## 1. Mission
Configure the in-cluster monitoring stack (kube-prometheus-stack) and ship everything computed or displayed on top of the telemetry: recording rules, alerts (with unit tests), and the two Grafana dashboards, **Node Telemetry** and **Cluster Telemetry**. You also build a small **fake node-agent**, so you can develop and verify the dashboards before the real agent exists.

## 2. You own
`deploy/monitoring/**` and `deploy/helm/observability/**`. Do not create or edit files anywhere else.

## 3. Deliverables
```
deploy/monitoring/
├── chart-version.txt         # exact kube-prometheus-stack chart version, single line
├── values.yaml               # EKS
├── values-local.yaml         # kind overrides
├── README.md
├── VERSIONS.md
└── HANDOFF.md                # covers both owned dirs
deploy/helm/observability/
├── Chart.yaml  values.yaml  README.md
├── Makefile                  # test: promtool check/test rules + dashboard lint
├── rules/                    # plain Prometheus rule files (so promtool can test them directly)
│   ├── node-agent.rules.yaml
│   └── time-api.rules.yaml
├── tests/
│   ├── node-agent.rules.test.yaml
│   └── time-api.rules.test.yaml
├── dashboards/
│   ├── node-telemetry.json
│   └── cluster-telemetry.json
├── templates/
│   ├── prometheusrules.yaml  # one PrometheusRule per rules/*.yaml via .Files.Glob
│   └── dashboards.yaml       # one ConfigMap per dashboards/*.json, label grafana_dashboard: "1"
└── dev/                      # NOT part of the chart (add to .helmignore)
    ├── fake_node_agent.py
    └── fake-node-agent.yaml   # ConfigMap + DaemonSet + PodMonitor for kind
```

## 4. Requirements

### 4.1 kube-prometheus-stack values (`deploy/monitoring/values.yaml`)
- MON-1: `fullnameOverride: kps` (this gives the service names in contracts §2). Pin the chart version in `chart-version.txt`.
- MON-2 **Prometheus**:
  ```yaml
  prometheus:
    prometheusSpec:
      replicas: 1
      retention: 15d
      retentionSize: 18GB
      scrapeInterval: 30s
      evaluationInterval: 15s
      serviceMonitorSelectorNilUsesHelmValues: false
      podMonitorSelectorNilUsesHelmValues: false
      ruleSelectorNilUsesHelmValues: false
      serviceMonitorNamespaceSelector: {}
      podMonitorNamespaceSelector: {}
      ruleNamespaceSelector: {}
      storageSpec:
        volumeClaimTemplate:
          spec:
            storageClassName: gp3
            accessModes: [ReadWriteOnce]
            resources: {requests: {storage: 20Gi}}
      resources:
        requests: {cpu: 200m, memory: 512Mi}
        limits: {memory: 2Gi}
  ```
- MON-3 **Alertmanager** is enabled with a single replica and a null receiver. Nothing is delivered (non-goal).
- MON-4 **node-exporter** is enabled, and is used only as a reference for checking agent accuracy. Add a relabeling to its ServiceMonitor that copies `__meta_kubernetes_pod_node_name` into `node` (chart key `prometheus-node-exporter.prometheus.monitor.relabelings`; verify the key against the pinned chart).
- MON-5 **kube-state-metrics** is enabled with `metricLabelsAllowlist: ["nodes=[workload]"]`, so that `kube_node_labels{label_workload="app"}` exists.
- MON-6 **Grafana**:
  ```yaml
  grafana:
    admin: {existingSecret: grafana-admin, userKey: admin-user, passwordKey: admin-password}
    persistence: {enabled: false}
    defaultDashboardsEnabled: true          # keep the stock k8s dashboards
    sidecar:
      dashboards: {enabled: true, label: grafana_dashboard, labelValue: "1", searchNamespace: ALL}
    grafana.ini:
      dashboards: {min_refresh_interval: 1s}
  ```
  The provisioned Prometheus datasource MUST have uid `prometheus` and `jsonData.timeInterval: "1s"`. Set this through the chart's datasource values. If the pinned chart can't express it, disable the default datasource and declare it in `additionalDataSources`. Record which approach you used in `README.md`.
- MON-7 `values-local.yaml` (kind): no `storageClassName` (use the default), 5Gi, Prometheus `retention: 2d`, and smaller resource requests.
- MON-8 The Secret `grafana-admin` is **not** created by these values. WS-50 creates it. On kind, create it yourself for testing.

### 4.2 Recording rules (`rules/node-agent.rules.yaml`)
```yaml
groups:
  - name: node-agent.cluster
    interval: 1s
    rules:
      - record: cluster:node_agent_cpu_busy:cores
        expr: sum(node_agent_cpu_usage_ratio * node_agent_cpu_count)
      - record: cluster:node_agent_cpu_count:sum
        expr: sum(node_agent_cpu_count)
      - record: cluster:node_agent_cpu_usage:ratio
        expr: cluster:node_agent_cpu_busy:cores / cluster:node_agent_cpu_count:sum
      - record: cluster:node_agent_memory_used:bytes
        expr: sum(node_agent_memory_used_bytes)
      - record: cluster:node_agent_memory_total:bytes
        expr: sum(node_agent_memory_total_bytes)
      - record: cluster:node_agent_memory_usage:ratio
        expr: cluster:node_agent_memory_used:bytes / cluster:node_agent_memory_total:bytes
  - name: node-agent.node
    interval: 30s
    rules:
      - record: node:node_agent_cpu_usage:avg5m
        expr: avg by (node) (avg_over_time(node_agent_cpu_usage_ratio[5m]))
      - record: node:node_agent_memory_usage:avg5m
        expr: avg by (node) (avg_over_time(node_agent_memory_usage_ratio[5m]))
```
Cluster CPU is **core-weighted** (busy cores ÷ total cores), not an average of per-node ratios, so it stays correct if node sizes ever differ. `node_agent_cpu_usage_ratio` and `node_agent_cpu_count` share an identical label set, so the plain `*` match is valid.

### 4.3 Alert rules
In `rules/node-agent.rules.yaml`, group `node-agent.alerts`:

| Alert | Expr | for | severity |
|---|---|---|---|
| `NodeAgentDown` | `up{job="node-agent"} == 0` | 1m | warning |
| `NodeAgentMissing` | `(count(kube_node_labels{label_workload="app"}) - count(up{job="node-agent"} == 1) > 0) or (absent(up{job="node-agent"} == 1) and on() count(kube_node_labels{label_workload="app"}) > 0)` | 2m | warning |
| `NodeAgentStale` | `time() - node_agent_last_sample_timestamp_seconds > 5` | 1m | warning |
| `NodeAgentCollectionErrors` | `rate(node_agent_collection_errors_total[5m]) > 0` | 10m | info |
| `NodeCPUHigh` | `node:node_agent_cpu_usage:avg5m > 0.9` | 5m | warning |
| `NodeMemoryHigh` | `node:node_agent_memory_usage:avg5m > 0.9` | 5m | warning |

In `rules/time-api.rules.yaml`, group `time-api.alerts`:

| Alert | Expr | for | severity |
|---|---|---|---|
| `TimeApiReplicasDegraded` | `kube_deployment_status_replicas_available{namespace="time-api",deployment="time-api"} < 3` | 5m | warning |
| `TimeApiDown` | `absent(up{job="time-api"} == 1)` | 2m | critical |
| `TimeApiHighErrorRate` | `sum(rate(time_api_requests_total{status=~"5.."}[5m])) / clamp_min(sum(rate(time_api_requests_total[5m])), 1e-9) > 0.05` | 5m | warning |
| `TimeApiHighLatency` | `histogram_quantile(0.99, sum by (le) (rate(time_api_request_duration_seconds_bucket{route="/time"}[5m]))) > 0.25` | 10m | info |

Every alert has `annotations.summary` and `annotations.description`, using `{{ $labels.node }}` where it applies.

### 4.4 Rule tests
- RT-1: `promtool check rules rules/*.yaml` is clean.
- RT-2: `promtool test rules tests/*.test.yaml`. **Every recording rule** has a test with a 3-node synthetic input that checks the computed value, including one test where the nodes have different core counts to prove core weighting. **Every alert** has a firing case and a non-firing case.
- RT-3: The chart template wraps each `rules/*.yaml` file in a `PrometheusRule` named after the file, with the label `app.kubernetes.io/part-of: time-telemetry`. Rendering must produce exactly the same `groups` content as the files.

### 4.5 Dashboards: shared conventions
- D-1: Grafana JSON for the Grafana version in the pinned chart. Fixed `uid`s: `node-telemetry` and `cluster-telemetry`. `"editable": true`, and `"id": null`.
- D-2: A templated `datasource` variable (type `prometheus`, default uid `prometheus`) that every panel uses. No hard-coded datasource UIDs anywhere else.
- D-3: Default time range `now-15m`, refresh `5s`, the refresh picker includes `1s`, and timezone is `browser`.
- D-4: Every timeseries panel has `interval: "1s"` (min interval). Use `$__rate_interval` inside `rate()`.
- D-5: Ratio panels use the unit `percentunit` with range 0–1. Byte panels use `bytes` (IEC).
- D-6: The dashboards render without query errors when only one node reports, and when time-api is absent (panels show "No data").
- D-7: You may hand-write the dashboards or generate them (grafonnet or Python). If you generate them, commit both the generator and the generated JSON, and make `make test` fail if the committed JSON is out of date.

### 4.6 Dashboard: Node Telemetry (`node-telemetry`)
Variable `node`: `label_values(node_agent_cpu_usage_ratio, node)`, multi-select, with an All option.

| Row | Panel | Type | Query |
|---|---|---|---|
| Overview | CPU now | stat | `node_agent_cpu_usage_ratio{node=~"$node"}` |
| Overview | Memory now | stat | `node_agent_memory_usage_ratio{node=~"$node"}` |
| Overview | CPU max (60 s window) | stat | `node_agent_cpu_usage_ratio_window{node=~"$node",stat="max"}` |
| Overview | Memory used | stat | `node_agent_memory_used_bytes{node=~"$node"}` |
| CPU | CPU usage | timeseries | `node_agent_cpu_usage_ratio{node=~"$node"}` · `{{node}}` |
| CPU | CPU per core | timeseries | `node_agent_cpu_core_usage_ratio{node=~"$node"}` · `{{node}} cpu{{cpu}}` |
| CPU | CPU by mode (cores, stacked) | timeseries | `sum by (node, mode) (rate(node_agent_cpu_seconds_total{node=~"$node",mode!="idle"}[$__rate_interval]))` |
| CPU | Rolling window min/avg/max | timeseries | `node_agent_cpu_usage_ratio_window{node=~"$node"}` · `{{node}} {{stat}}` |
| Memory | Used vs total | timeseries | `node_agent_memory_used_bytes{node=~"$node"}`, `node_agent_memory_total_bytes{node=~"$node"}` |
| Memory | Usage ratio | timeseries | `node_agent_memory_usage_ratio{node=~"$node"}` |
| Memory | Breakdown (stacked) | timeseries | `node_agent_memory_used_bytes`, `…_buffers_bytes`, `…_cached_bytes`, `…_free_bytes` with `{node=~"$node"}` |
| Memory | Rolling window min/avg/max | timeseries | `node_agent_memory_usage_ratio_window{node=~"$node"}` |
| Agent health | Target up | state timeline | `up{job="node-agent",node=~"$node"}` |
| Agent health | Sample age (s) | timeseries | `time() - node_agent_last_sample_timestamp_seconds{node=~"$node"}` |
| Agent health | Collection p99 | timeseries | `histogram_quantile(0.99, sum by (node, le) (rate(node_agent_collection_duration_seconds_bucket{node=~"$node"}[1m])))` |
| Agent health | Errors and missed ticks | timeseries | `rate(node_agent_collection_errors_total{node=~"$node"}[1m])`, `rate(node_agent_missed_ticks_total{node=~"$node"}[1m])` |
| Agent health | Agent CPU and memory | timeseries | `rate(container_cpu_usage_seconds_total{namespace="monitoring",container="node-agent"}[1m])`, `container_memory_working_set_bytes{namespace="monitoring",container="node-agent"}` (by `node`) |

### 4.7 Dashboard: Cluster Telemetry (`cluster-telemetry`)

| Row | Panel | Type | Query |
|---|---|---|---|
| Cluster | Nodes reporting | stat (3 = green, below 3 = red) | `count(up{job="node-agent"} == 1)` |
| Cluster | Cluster CPU | stat + timeseries | `cluster:node_agent_cpu_usage:ratio` |
| Cluster | Busy cores / total cores | timeseries | `cluster:node_agent_cpu_busy:cores`, `cluster:node_agent_cpu_count:sum` |
| Cluster | Cluster memory | stat + timeseries | `cluster:node_agent_memory_usage:ratio` |
| Cluster | Memory used / total | timeseries | `cluster:node_agent_memory_used:bytes`, `cluster:node_agent_memory_total:bytes` |
| Cluster | Hottest node | timeseries | `topk(1, node_agent_cpu_usage_ratio)` · `{{node}}` |
| Distribution | CPU per node | timeseries | `node_agent_cpu_usage_ratio` · `{{node}}` |
| Distribution | Memory per node | timeseries | `node_agent_memory_usage_ratio` · `{{node}}` |
| Distribution | CPU by node over time | status history | `node_agent_cpu_usage_ratio` |
| Distribution | CPU imbalance (max − min) | timeseries | `max(node_agent_cpu_usage_ratio) - min(node_agent_cpu_usage_ratio)` |
| time-api | Available replicas | stat (3 = green) | `kube_deployment_status_replicas_available{namespace="time-api",deployment="time-api"}` |
| time-api | Request rate by status | timeseries | `sum by (status) (rate(time_api_requests_total{route="/time"}[$__rate_interval]))` |
| time-api | Latency p50/p95/p99 | timeseries | `histogram_quantile(q, sum by (le) (rate(time_api_request_duration_seconds_bucket{route="/time"}[$__rate_interval])))` for q = 0.5, 0.95, 0.99 |
| time-api | Requests per pod | timeseries | `sum by (pod) (rate(time_api_requests_total{route="/time"}[$__rate_interval]))` |
| Alerts | Firing alerts | alert list | filtered to the `node-agent.*` and `time-api.*` groups |
| Links | – | dashboard link | to `node-telemetry`, passing the time range |

### 4.8 Fake node-agent (`dev/`, for development only)
- FAKE-1: `fake_node_agent.py` uses only the Python standard library. It serves on `:9102/metrics` **every** metric in contracts §5, with the same names, types, labels and semantics. It generates plausible values: a sine-wave CPU between 10% and 70% with a per-node phase taken from `NODE_NAME`, a slowly drifting memory value, windows computed over the last 60 generated samples, and counters that increase. Setting `FAKE_SPIKE=1` drives CPU to 95% for 60 s, so you can test alerts.
- FAKE-2: `fake-node-agent.yaml` holds a ConfigMap with the script, plus a DaemonSet in `monitoring` (image `python:3.12-slim`, pod label `app.kubernetes.io/name: node-agent`, port 9102 named `metrics`, `nodeSelector: {workload: app}`), plus the **same PodMonitor** as in `30-node-agent.md` K-8. This lets you see the whole pipeline on kind without WS-30.
- FAKE-3: The `dev/` folder is excluded from the packaged chart through `.helmignore`.

## 5. Interfaces you provide
- `deploy/monitoring/chart-version.txt`, `values.yaml` and `values-local.yaml`. WS-50 runs `helm upgrade --install kps prometheus-community/kube-prometheus-stack --version $(cat chart-version.txt) -n monitoring -f values.yaml`.
- The `deploy/helm/observability` chart, installed as release `observability` in `monitoring` **after** kps and before the agent and app.
- `make -C deploy/helm/observability test`.
- The recording-rule names in §4.2. WS-50's acceptance tests query them.

## 6. Standalone testing (kind, without WS-20 or WS-30)
```bash
make -C deploy/helm/observability test                      # promtool + dashboard lint
kind create cluster --config kind.yaml                      # contracts §8.1
kubectl create ns monitoring
kubectl -n monitoring create secret generic grafana-admin --from-literal=admin-user=admin --from-literal=admin-password=admin
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm upgrade --install kps prometheus-community/kube-prometheus-stack --version "$(cat deploy/monitoring/chart-version.txt)" \
  -n monitoring -f deploy/monitoring/values.yaml -f deploy/monitoring/values-local.yaml --wait
helm upgrade --install observability deploy/helm/observability -n monitoring --wait
kubectl apply -f deploy/helm/observability/dev/fake-node-agent.yaml
kubectl -n monitoring port-forward svc/kps-grafana 3000:80 &   # admin / admin
kubectl -n monitoring port-forward svc/kps-prometheus 9090 &
```
Then verify:
- Prometheus targets show 3 `node-agent` targets UP at 1 s, and each series has a `node` label.
- `cluster:node_agent_cpu_usage:ratio` has data.
- Both dashboards appear automatically, the `node` variable lists 3 nodes, and no panel shows an error.
- Setting `FAKE_SPIKE=1` on one pod makes `NodeCPUHigh` go pending, then fire.

Also check D-6: delete the fake DaemonSet, then confirm the dashboards show "No data" rather than errors.

## 7. Definition of Done
- [ ] `make -C deploy/helm/observability test` is green (RT-1, RT-2, and the dashboard JSON lint: valid JSON, expected UIDs, no hard-coded datasource UID, every panel uses `${datasource}`).
- [ ] `helm lint` and `kubeconform` pass for the observability chart. `helm template` for kps with both values files succeeds.
- [ ] The §6 kind run is fully verified, including the alert firing. Screenshots or the Prometheus query output go in `HANDOFF.md`.
- [ ] The datasource has `timeInterval` 1s, and Grafana allows a 1 s refresh.
- [ ] `README.md` and `VERSIONS.md` are written in `deploy/monitoring/` (VERSIONS covers the kps chart and the Grafana and Prometheus versions it brings). `HANDOFF.md` is written (contracts §11).

## 8. Out of scope
The real agent and app (WS-30, WS-20), their ServiceMonitor and PodMonitor (the emitters own those; your copy in `dev/` is only for testing), Terraform, the top-level Makefile, Alertmanager routing, and public ingress for Grafana.

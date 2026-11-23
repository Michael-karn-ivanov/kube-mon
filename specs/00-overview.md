# 00 — System Overview

**Every agent must read this file and `01-contracts.md` before its own workstream spec.**

---

## 1. What we are building

1. **time-api**: a minimal Python service with one business endpoint, `GET /time`, that returns the current time. It runs as **3 replicas**, one per worker node, behind a Kubernetes Service exposed through an AWS Network Load Balancer.
2. **node-agent**: a small **C++ binary** that runs as a **DaemonSet** on every node hosting time-api. Every **1 s** it samples node CPU and memory usage from `/proc`, keeps a short rolling window of statistics, and exposes Prometheus metrics on `/metrics`. Prometheus pulls from it.
3. **Prometheus**: the central store, self-hosted in the EKS cluster through kube-prometheus-stack, on an EBS-backed volume. It scrapes node-agent every 1 s and time-api every 15 s.
4. **Grafana**: also in the cluster, with two dashboards provisioned from code. **Node Telemetry** shows each node; **Cluster Telemetry** aggregates the 3 nodes and shows time-api health.
5. **AWS infrastructure**: built with **Terraform**. Workloads are deployed with **Helm**, and a top-level **Makefile** orchestrates everything.

```
                         Internet
                            │
                     ┌──────▼──────┐
                     │  AWS NLB    │  Service type=LoadBalancer (AWS LB Controller)
                     └──────┬──────┘
         ┌──────────────────┼──────────────────┐
   ┌─────▼─────┐      ┌─────▼─────┐      ┌─────▼─────┐
   │  Node A   │      │  Node B   │      │  Node C   │  EKS managed node group
   │  (AZ a)   │      │  (AZ b)   │      │  (AZ c)   │  3 × t3.large, AL2023
   │ time-api  │      │ time-api  │      │ time-api  │  Deployment, 1 pod/node
   │ node-agent│      │ node-agent│      │ node-agent│  DaemonSet, reads /host/proc
   └─────┬─────┘      └─────┬─────┘      └─────┬─────┘
         │  :9102/metrics @1s      :9000/metrics @15s
         └──────────────────┼──────────────────┘
                     ┌──────▼──────┐  PVC gp3 20Gi, 15d retention
                     │ Prometheus  │◄─ recording rules + alerts
                     └──────┬──────┘
                     ┌──────▼──────┐
                     │  Grafana    │  dashboards from ConfigMaps; port-forward access
                     └─────────────┘
```

---

## 2. Goals
- G1. `GET /time` is reachable from the internet and served by 3 replicas on 3 nodes.
- G2. Per-node CPU and memory are sampled at 1 s by a purpose-built C++ agent.
- G3. Telemetry is stored centrally in Prometheus and survives pod restarts.
- G4. Grafana shows per-node and whole-cluster views with no manual setup.
- G5. The whole system is reproducible from a clean AWS account with `make`, and tears down cleanly.
- G6. Every component can be built and tested on its own, so it can be developed in parallel.

## 3. Non-goals
- HTTPS, a domain, or auth on `/time`.
- Autoscaling (HPA, Cluster Autoscaler, Karpenter). The count is fixed at 3 nodes and 3 replicas.
- Multiple environments (`dev` only).
- HA Prometheus, Thanos/Mimir, or object-storage retention.
- Alertmanager notification routing (alerts are visible, but nothing is sent).
- Log aggregation and tracing.
- Arm/Graviton (`linux/amd64` only).

## 4. Decided assumptions
1. A single `dev` environment in `eu-west-2`, in one AWS account; the operator has admin credentials.
2. 3 nodes, 3 replicas. The agent runs on every node labelled `workload=app`, so "the agent runs wherever the app runs" still holds if node groups are added.
3. **Node CPU** = host-wide busy time from `/proc/stat`, excluding idle and iowait. **Node memory used** = `MemTotal − MemAvailable`.
4. Pull model: the agent "publishes" by exposing `/metrics`, and Prometheus scrapes it.
5. "Accumulate statistics" is met in two places. Short term, by the agent's 60-sample rolling min/avg/max plus cumulative CPU counters. Long term, by the Prometheus TSDB with 15-day retention.
6. `/time` is public over plain HTTP. Grafana and Prometheus are reachable only through `kubectl port-forward`.
7. node-exporter stays deployed only as an independent reference for checking the C++ agent's accuracy.

---

## 5. Workstreams

| WS | Spec | Builds | Can start | Standalone "done" |
|---|---|---|---|---|
| WS-10 | `10-infra.md` | Terraform: state bucket, VPC, EKS, node group, ECR, IAM, LB Controller, StorageClass | now | `fmt`/`validate`/`tflint` clean; `plan` clean if credentials are available |
| WS-20 | `20-time-api.md` | Python app, image, Helm chart (with ServiceMonitor) | now | `make -C apps/time-api test`; the chart deploys on kind |
| WS-30 | `30-node-agent.md` | C++ agent, image, Helm chart (with PodMonitor) | now | `make -C agents/node-agent test`; the chart deploys on kind |
| WS-40 | `40-observability.md` | kps values, recording and alert rules with tests, 2 Grafana dashboards, a fake metrics emitter for development | now | `make -C deploy/helm/observability test`; dashboards render on kind using the fake emitter |
| WS-50 | `50-integration.md` | Top-level Makefile, `deploy.env`, scripts (smoke, acceptance, stress), CI, deployment runbook | scaffolding now; end-to-end once 10–40 are merged | `make up` on EKS passes every acceptance criterion; `make destroy` is clean |

WS-10 to WS-40 depend only on `01-contracts.md`. They own disjoint paths and can be merged in any order.

---

## 6. Full repository layout (owner in brackets)

```
.
├── README.md                          [50] links to specs + runbook quick start
├── Makefile                           [50]
├── deploy.env.example                 [50]
├── .gitignore                         [50]
├── versions.md                        [50] index of per-component VERSIONS.md
├── specs/                             [owner] this spec set
├── apps/time-api/                     [20]
│   ├── Makefile, pyproject.toml, uv.lock, Dockerfile, README.md, VERSIONS.md, HANDOFF.md
│   ├── src/time_api/{__init__,main,metrics,settings}.py
│   └── tests/
├── agents/node-agent/                 [30]
│   ├── Makefile, CMakeLists.txt, CMakePresets.json, Dockerfile, README.md, VERSIONS.md, HANDOFF.md
│   ├── src/{main,config,procfs,cpu,memory,window,scheduler,collector,metrics,log}.{hpp,cpp}
│   └── tests/{fixtures/proc/…, *.cpp, integration_test.sh}
├── infra/terraform/                   [10]
│   ├── Makefile, README.md, VERSIONS.md, HANDOFF.md, .tflint.hcl
│   ├── bootstrap/
│   ├── cluster/   (+ policies/aws-load-balancer-controller.json)
│   └── platform/
├── deploy/
│   ├── helm/time-api/                 [20]
│   ├── helm/node-agent/               [30]
│   ├── helm/observability/            [40] rules/, tests/, dashboards/, templates/, dev/
│   └── monitoring/                    [40] values.yaml, values-local.yaml, chart-version.txt
├── scripts/                           [50] kind-config.yaml, lib.sh, smoke_test.sh, acceptance_test.sh, stress_node.sh
└── .github/workflows/ci.yaml          [50]
```

# WS-50 — Integration, Deployment and Acceptance

Read first: `00-overview.md`, then `01-contracts.md` (all of it). You may *read* every other spec, because you integrate them.

## 1. Mission
Make the whole system deployable, operable and verifiable with one command. You own the top-level Makefile, the operator config, the smoke, acceptance and stress scripts, CI, and the deployment runbook (§5 of this file, which becomes the root `README.md` quick start). Build the scaffolding now, against the contracts. Once WS-10 to WS-40 are merged, run the full end-to-end deployment on EKS and the acceptance suite.

## 2. You own
`Makefile`, `deploy.env.example`, `.gitignore`, `README.md`, `versions.md`, `scripts/**` and `.github/**`. Do not edit files owned by other workstreams. If something there blocks you, write it up in `HANDOFF.md` as an issue for that workstream.

## 3. Deliverables
```
Makefile
deploy.env.example
.gitignore                 # deploy.env, infra/terraform/backend.hcl, **/.terraform/, *.tfstate*, build/, .venv/, __pycache__/
README.md                  # what this is, links to specs/, quick start = §5.3–5.5 condensed
versions.md                # links to every component VERSIONS.md
scripts/
├── kind-config.yaml       # canonical copy of contracts §8.1
├── lib.sh                 # shared helpers: log, die, require_cmd, prom_query, wait_for
├── doctor.sh
├── endpoint.sh            # wait for NLB hostname + DNS, print URL
├── smoke_test.sh
├── acceptance_test.sh
├── stress_node.sh
└── verify_clean.sh
.github/workflows/ci.yaml
HANDOFF.md                 # at repo root (your primary dir)
```
Script rules: `bash`, `set -euo pipefail`, `shellcheck`-clean. Each check prints one PASS or FAIL line, and a failure exits non-zero. Scripts are idempotent and use `kubectl --context time-telemetry-dev` (or `kind-time-telemetry` for local), never the ambient context.

## 4. Makefile

### 4.1 Configuration and derived variables
- `-include deploy.env` and `export`. Required keys are listed in §5.2. If one is missing, the target fails with a clear message.
- `ACCOUNT_ID := $(shell aws sts get-caller-identity --query Account --output text)`
- `REGISTRY := $(ACCOUNT_ID).dkr.ecr.$(REGION).amazonaws.com`
- `GIT_SHA ?= $(shell git rev-parse --short=12 HEAD)`. It can be overridden to redeploy an older build.
- `VERSION ?= $(shell git describe --tags --always)`
- `KPS_VERSION := $(shell cat deploy/monitoring/chart-version.txt)`
- `KCTX ?= time-telemetry-dev`. Every `kubectl` and `helm` call passes `--context $(KCTX)` / `--kube-context $(KCTX)`.
- Comma-separated lists in `deploy.env` are turned into a JSON list for `TF_VAR_*`, and into `{a,b}` for Helm `--set`.

### 4.2 Targets
Every target is idempotent. Component work is **delegated** to the component Makefiles (contracts §7.1).

| Target | Does |
|---|---|
| `doctor` | `scripts/doctor.sh`: tools present at the minimum versions (§5.1), AWS identity works, `deploy.env` present and complete, buildx can build `linux/amd64`, `chart-version.txt` present. Prints a PASS/FAIL table. |
| `lint` | `$(MAKE) -C apps/time-api lint`, `-C agents/node-agent lint`, `-C infra/terraform fmt-check validate lint`, `helm lint` + `kubeconform` on all 3 charts, `shellcheck scripts/*.sh` |
| `test` | `$(MAKE) -C apps/time-api test`, `-C agents/node-agent test`, `-C deploy/helm/observability test` |
| `tf-bootstrap` | Applies `infra/terraform/bootstrap`, writes `infra/terraform/backend.hcl` (`bucket`, `region`), and sets `TF_STATE_BUCKET=` in `deploy.env` |
| `tf-cluster` / `tf-platform` | `terraform -chdir=… init -backend-config=../backend.hcl`, then `apply`. It is interactive unless `AUTO_APPROVE=1`. Variables go in as `TF_VAR_*` (contracts §9). |
| `tf-plan` | `plan` for cluster and platform |
| `kubeconfig` | `aws eks update-kubeconfig --name time-telemetry-dev --region $(REGION) --alias time-telemetry-dev` |
| `build` | Refuses to run if the git tree is dirty (unless `ALLOW_DIRTY=1`). Then `$(MAKE) -C apps/time-api image IMAGE=$(REGISTRY)/time-api:$(GIT_SHA) VERSION=$(VERSION) GIT_SHA=$(GIT_SHA)`, and the same for node-agent. |
| `push` | ECR login. For each image, skip if the tag already exists in ECR (tags are immutable), otherwise `docker push`. |
| `grafana-secret` | Creates `monitoring/grafana-admin` with a random 24-character password if it doesn't exist. Never overwrites. |
| `deploy-monitoring` | `helm repo add/update`, then `helm upgrade --install kps prometheus-community/kube-prometheus-stack --version $(KPS_VERSION) -n monitoring -f deploy/monitoring/values.yaml --wait --timeout 10m` |
| `deploy-observability` | `helm upgrade --install observability deploy/helm/observability -n monitoring --wait` |
| `deploy-agent` | `helm upgrade --install node-agent deploy/helm/node-agent -n monitoring --set image.repository=$(REGISTRY)/node-agent --set image.tag=$(GIT_SHA) --wait --timeout 5m` |
| `deploy-app` | `helm upgrade --install time-api deploy/helm/time-api -n time-api --set image.repository=$(REGISTRY)/time-api --set image.tag=$(GIT_SHA) --set "service.loadBalancerSourceRanges={$(TIME_API_SOURCE_RANGES)}" --wait --timeout 10m` |
| `deploy` | deploy-monitoring → deploy-observability → deploy-agent → deploy-app (the order matters, because CRDs must exist first) |
| `endpoint` | `scripts/endpoint.sh`: waits up to 5 min for the Service hostname and for DNS to resolve, then prints `http://<host>/time` |
| `up` | doctor → tf-cluster → tf-platform → kubeconfig → build → push → grafana-secret → deploy → endpoint → smoke |
| `status` | One screen: nodes (with workload label and zone), pods in both namespaces, the Helm releases and revisions, Prometheus target health for both jobs, the endpoint |
| `smoke` | `scripts/smoke_test.sh` (§6.1) |
| `acceptance` | `scripts/acceptance_test.sh` (§6.2) |
| `stress NODE=<name> [SECONDS=120]` | `scripts/stress_node.sh` |
| `grafana` | Prints the credentials, then port-forwards `svc/kps-grafana` to `:3000` |
| `prometheus` / `alertmanager` | Port-forward `svc/kps-prometheus` to `:9090` / `svc/kps-alertmanager` to `:9093` |
| `logs-app` / `logs-agent` | `kubectl logs -f -l app.kubernetes.io/name=<x> --all-containers --prefix --max-log-requests 10` |
| `rollback-app [REV=n]` / `rollback-agent [REV=n]` | `helm rollback <release> [n] --wait` |
| `destroy` | Ordered teardown (§5.8). Asks for confirmation unless `CONFIRM=yes`. |
| `verify-clean` | `scripts/verify_clean.sh` (§5.8, step 5) |
| `local-up` | Creates the kind cluster from `scripts/kind-config.yaml`, builds both images (`IMAGE=<name>:local`), runs `kind load`, creates the namespaces and the grafana secret, then deploys all 4 releases with the `values-local.yaml` files (`KCTX=kind-time-telemetry`, using `--create-namespace`) |
| `local-endpoint` | port-forward `svc/time-api` to `:8080` |
| `local-smoke` | The smoke test against kind, without the NLB checks |
| `local-down` | `kind delete cluster --name time-telemetry` |

## 5. Deployment runbook (operator guide)
This section is the source for the root `README.md`. The Makefile and scripts MUST behave exactly as described here.

### 5.1 Prerequisites
**AWS**
- A dedicated dev account is preferred. Every resource is tagged `project=time-telemetry`.
- An IAM principal with `AdministratorAccess` or an equivalent. An IAM Identity Center profile is recommended (`aws sso login --profile <p>`).
- Whoever runs `make tf-cluster` becomes EKS cluster admin. To give other people or roles access, add their **IAM role ARNs** (not SSO session ARNs) to `CLUSTER_ADMIN_ARNS`.
- Quotas in `eu-west-2`:
  - at least **8 vCPUs** of On-Demand Standard instances (3 × t3.large = 6, and node upgrades briefly add a 4th node)
  - 1 spare Elastic IP
  - room for 1 more VPC

  New accounts often start with low vCPU quotas.

**Workstation tools.** `make doctor` checks these.

| Tool | Notes |
|---|---|
| `aws` CLI v2 | auth, ECR, kubeconfig |
| `terraform` ≥ 1.11 | |
| `kubectl` | within ±1 minor of the cluster version |
| `helm` ≥ 3.14 | |
| `docker` + `buildx` | On Apple Silicon, `linux/amd64` builds run under emulation, so the C++ build is slow there |
| `git`, `make`, `jq`, `curl` | |
| `kind` | local only |
| `promtool`, `kubeconform`, `shellcheck`, `tflint`, `uv`, `cmake`, `g++` | only for `make lint test` |

**Network**: get your public IP with `curl -s https://checkip.amazonaws.com`, and use it as `/32` in `OPERATOR_CIDRS`.

### 5.2 Configuration: `deploy.env`
Copy `deploy.env.example` (committed) to `deploy.env` (gitignored):
```bash
# --- AWS ---
AWS_PROFILE=misha-dev
REGION=eu-west-2
# --- Access control (comma-separated lists) ---
OPERATOR_CIDRS=203.0.113.7/32        # who may reach the EKS API (required)
CLUSTER_ADMIN_ARNS=                  # extra IAM role ARNs with kubectl admin (optional)
TIME_API_SOURCE_RANGES=0.0.0.0/0     # who may reach the public /time endpoint
# --- Written by `make tf-bootstrap`; do not edit ---
TF_STATE_BUCKET=
```
The kube-prometheus-stack version is **not** set here. It comes from `deploy/monitoring/chart-version.txt`.

### 5.3 One-time setup (per AWS account)
```bash
git clone <repo> && cd <repo>
cp deploy.env.example deploy.env && $EDITOR deploy.env
aws sso login --profile "$AWS_PROFILE"        # if using SSO
make doctor                                    # all green
make tf-bootstrap                              # S3 state bucket, ~1 min
```
The state bucket is kept across `make destroy`. It costs almost nothing.

### 5.4 First full deployment
The quick way is `make up` (about 30–40 min). Here are the same steps one at a time, with a check after each. Use this for the first deploy, or to find out where `make up` failed.

| # | Command | Time | Check before moving on |
|---|---|---|---|
| 1 | `make tf-cluster` | 15–20 min | `Apply complete`, and the outputs list `cluster_name` and the `ecr_*_url`s |
| 2 | `make kubeconfig` | s | `kubectl get nodes -L workload,topology.kubernetes.io/zone` shows **3 Ready**, `workload=app`, in **3 zones** |
| 3 | `make tf-platform` | 2–4 min | `kubectl -n kube-system get deploy aws-load-balancer-controller` is `2/2`. `kubectl get sc` shows `gp3 (default)`. The namespaces `time-api` and `monitoring` exist. |
| 4 | `make build push` | 3–10 min | `aws ecr describe-images --repository-name time-api --image-ids imageTag=$(git rev-parse --short=12 HEAD)` succeeds, and the same for `node-agent` |
| 5 | `make grafana-secret` | s | `kubectl -n monitoring get secret grafana-admin` exists |
| 6 | `make deploy-monitoring` | 3–5 min | Every pod in `monitoring` is Running, the Prometheus PVC is `Bound`, and `kubectl get crd podmonitors.monitoring.coreos.com` succeeds |
| 7 | `make deploy-observability` | s | `kubectl -n monitoring get prometheusrules,cm -l app.kubernetes.io/part-of=time-telemetry` lists 2 rules and 2 dashboards |
| 8 | `make deploy-agent` | 1 min | `kubectl -n monitoring get ds node-agent` shows `3/3`. In `make prometheus` → `/targets`, there are 3 `node-agent` targets UP at 1 s. |
| 9 | `make deploy-app` then `make endpoint` | 3–5 min | `kubectl -n time-api get pods -o wide` shows 3 Running pods on **3 distinct nodes**, and a URL is printed |
| 10 | `make smoke` | ~30 s | All PASS (§6.1) |
| 11 | `make grafana` → http://localhost:3000 | – | **Node Telemetry** and **Cluster Telemetry** are present, the `node` drop-down lists 3 nodes, and the panels have data |
| 12 | `make acceptance` (recommended after the first deploy) | ~15 min | All PASS (§6.2) |

### 5.5 Accessing the system

| What | How |
|---|---|
| API | `make endpoint`, then `curl http://<nlb>/time` |
| Grafana | `make grafana` → http://localhost:3000 (`admin` / printed password, or `kubectl -n monitoring get secret grafana-admin -o jsonpath='{.data.admin-password}' \| base64 -d`) |
| Prometheus | `make prometheus` → http://localhost:9090 |
| Alertmanager | `make alertmanager` → http://localhost:9093 (alerts are visible, but nothing is sent) |
| Logs | `make logs-app`, `make logs-agent` |
| Overview | `make status` |

### 5.6 Day-2 operations
- **New time-api version**: commit, then `make build push deploy-app smoke`. It rolls one pod at a time (`maxSurge 0` / `maxUnavailable 1`), so 2 of 3 replicas keep serving. Takes about 1–2 min.
- **New node-agent version**: `make build push deploy-agent`. The DaemonSet rolls one node at a time, so each node has a data gap of about 5–10 s.
- **Change dashboards or rules**: edit them under `deploy/helm/observability/`, run `make -C deploy/helm/observability test`, then `make deploy-observability`. Grafana and Prometheus reload within about 1 min with no restart. Dashboard edits made in the UI are **not saved**; export the JSON into the repo.
- **Roll back**: use `helm -n time-api history time-api` to find the last good revision, then `make rollback-app REV=<n>` (or `make deploy-app GIT_SHA=<old>`). Use `make rollback-agent` for the agent, and `helm -n monitoring rollback kps|observability <n>` for monitoring.
- **Upgrade kube-prometheus-stack**: WS-40 bumps `chart-version.txt`. **Read the chart's upgrade notes first.** Helm does not upgrade CRDs, so for major versions apply the new CRDs with `kubectl apply --server-side` before `make deploy-monitoring`. Data survives on the PVC.
- **Upgrade Kubernetes**: bump `cluster_version` by **one minor version** at a time, then `make tf-plan tf-cluster`. The control plane upgrades first (about 10 min), then the nodes roll one by one; the PDB keeps at least 2 time-api replicas. Prometheus's EBS volume is tied to one AZ, so it may stay Pending briefly until a node in that AZ is Ready again. That resolves on its own.
- **Your IP changed**: update `OPERATOR_CIDRS`, then `make tf-cluster` (otherwise `kubectl` times out). For public endpoint access, edit `TIME_API_SOURCE_RANGES`, then `make deploy-app`.
- **Cost**: nothing scales to zero, so run `make destroy` when you're not using the environment. `make up` brings it back in about 35 min, but Prometheus history is lost.

### 5.7 Local deployment (kind, no AWS)
```bash
make local-up         # ~5–8 min
make local-endpoint   # curl localhost:8080/time
make grafana KCTX=kind-time-telemetry
make local-down
```
Differences from EKS: there is no NLB (ClusterIP plus port-forward), no EBS (local-path storage, 5Gi), and Grafana has no persistence. **All 3 agents report identical host-wide numbers**, because kind nodes share one kernel. Per-node correctness can only be checked on EKS.

### 5.8 Teardown (`make destroy`)
The steps run in order and stop at the first failure:
1. `helm -n time-api uninstall time-api`, then wait up to 5 min until no ELBv2 load balancer tagged `elbv2.k8s.aws/cluster=time-telemetry-dev` remains.
2. `helm -n monitoring uninstall node-agent observability kps`, then `kubectl -n monitoring delete pvc --all --wait`, then wait until no EBS volumes tagged `kubernetes.io/created-for/pvc/namespace=monitoring` remain.
3. `terraform destroy` on `platform`.
4. `terraform destroy` on `cluster`. ECR has `force_delete`, so the images go too.
5. `make verify-clean`: `aws resourcegroupstaggingapi get-resources --tag-filters Key=project,Values=time-telemetry`, plus checks for leftover ENIs, ELBs and EBS volumes. It must report nothing.

The state bucket is kept. To remove it as well, empty and delete it by hand, then delete `infra/terraform/bootstrap/terraform.tfstate`.

If step 4 fails with `DependencyViolation` on the VPC or subnets, the usual cause is an orphaned NLB or ENI. Find it with `aws elbv2 describe-load-balancers` or `aws ec2 describe-network-interfaces --filters Name=vpc-id,Values=<vpc>`, delete it, and re-run.

### 5.9 Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `kubectl`: Unauthorized | wrong profile, expired SSO, no access entry | `aws sso login`; check `AWS_PROFILE`; add your role to `CLUSTER_ADMIN_ARNS` and run `make tf-cluster`; `make kubeconfig` |
| `kubectl` times out | your IP is not in `OPERATOR_CIDRS` | update it and run `make tf-cluster` |
| `tf-cluster` fails launching instances | vCPU quota | request an increase (§5.1) |
| `ImagePullBackOff` | the tag isn't in ECR | `make push`; compare the pod's image with `aws ecr describe-images` |
| `exec format error` | arm64 image | rebuild with `make build` (it forces `linux/amd64`) |
| `EXTERNAL-IP <pending>` for over 5 min | LBC down or missing IAM, or subnets untagged | `kubectl -n kube-system logs deploy/aws-load-balancer-controller`; check the Pod Identity association and the `kubernetes.io/role/elb` subnet tag |
| NLB resolves but times out | `TIME_API_SOURCE_RANGES` excludes you, or targets are unhealthy | `aws elbv2 describe-target-health`; check `/readyz` |
| time-api pod Pending | fewer than 3 schedulable nodes (anti-affinity) | `kubectl get nodes`; `kubectl describe pod` |
| `helm upgrade time-api` hangs | `maxSurge` above 0, or the PDB is blocking | restore `maxSurge: 0`; `kubectl -n time-api get pdb` |
| Prometheus PVC Pending | EBS CSI driver or IAM, or no default StorageClass | `kubectl -n kube-system get pods -l app=ebs-csi-controller`; `kubectl get sc`. Pending before the pod is scheduled is normal with `WaitForFirstConsumer`. |
| `no matches for kind PodMonitor` | monitoring was not deployed first | `make deploy-monitoring`, then retry |
| No node-agent targets | PodMonitor not selected, or port name wrong | check the kps selector values; `kubectl -n monitoring get podmonitor node-agent -o yaml` |
| No `node` label | relabeling missing | WS-30 K-8 |
| `NodeAgentStale` while the pod is Running | collector wedged, or procfs not mounted | `make logs-agent`; check the hostPath mount in the pod spec; delete the pod |
| Dashboards missing | ConfigMap label, or sidecar scope | `kubectl -n monitoring get cm -l grafana_dashboard=1`; `kubectl -n monitoring logs deploy/kps-grafana -c grafana-sc-dashboard` |
| Gaps or "No data" at short ranges | datasource `timeInterval` not 1s | WS-40 MON-6 |
| Can't refresh faster than 5 s | `min_refresh_interval` | WS-40 MON-6 |
| `make destroy` stuck on VPC | orphaned NLB or ENI | §5.8 |

## 6. Verification scripts

### 6.1 `smoke_test.sh` (about 30 s)
1. `GET <endpoint>/time` returns 200 with JSON that matches contracts §3, within 2 s of local UTC.
2. 60 requests see **3** distinct `X-Served-By` values.
3. `GET <endpoint>/metrics` returns 404.
4. Prometheus (via a temporary port-forward): `count(up{job="node-agent"}==1) == 3` and `count(up{job="time-api"}==1) == 3`.
5. `count(node_agent_cpu_usage_ratio) == 3` and `cluster:node_agent_cpu_usage:ratio` has a value.
6. The Grafana API `/api/dashboards/uid/node-telemetry` and `/cluster-telemetry` both return 200.

### 6.2 `acceptance_test.sh` (about 15 min)
Each AC is a function that prints PASS or FAIL and the evidence. A summary table comes at the end. Any FAIL exits non-zero.

**Infrastructure**
- AC-1: the §5.4 steps complete with no manual intervention beyond `deploy.env`. This is checked by running `make up` in CI-like conditions, or recorded manually in `HANDOFF.md`.
- AC-2: 3 Ready nodes, all `workload=app`, in 3 distinct AZs.

**time-api**
- AC-3: 3 Running and Ready pods on 3 distinct nodes.
- AC-4: `/time` matches the contract and is within 2 s of the runner's NTP-synced clock.
- AC-5: 300 requests see 3 distinct `X-Served-By` values.
- AC-6: run a 10 req/s load for 60 s, and delete one pod during it. At most 1% of requests fail, and the replacement is Ready on the same node within 60 s.
- AC-7: `helm upgrade` with a changed annotation (to force a rollout) completes without hanging, and the same 1% bound holds.
- AC-8: `/metrics` returns 404 through the NLB.

**node-agent and Prometheus**
- AC-9: 3 agent pods, one per node. The Prometheus targets API shows 3 `node-agent` targets UP with `scrapeInterval` 1s.
- AC-10: `count(node_agent_cpu_usage_ratio) == 3`, and the set of `node` values equals `kubectl get nodes -o name`.
- AC-11: `min(count_over_time(node_agent_cpu_usage_ratio[1m])) >= 55`.
- AC-12, accuracy against node-exporter (which also has a `node` label, through WS-40 MON-4):
  - CPU: for every node, `abs(avg_over_time(node_agent_cpu_usage_ratio[2m]) - on(node) (1 - avg by (node) (rate(node_cpu_seconds_total{mode=~"idle|iowait"}[2m])))) < 0.05`. Check this once idle and once during AC-13.
  - Memory: `abs(node_agent_memory_used_bytes - on(node) (node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes)) / on(node) node_memory_MemTotal_bytes < 0.02`.
- AC-13: `stress_node.sh <node>` runs a `stress-ng --cpu 0 --timeout 120s` pod pinned to that node (`--cpu 0` uses every CPU). Within 5 s, that node's `node_agent_cpu_usage_ratio` is above 0.9, the other two stay below 0.5, and `cluster:node_agent_cpu_usage:ratio` rises by about 0.33 ± 0.1.
- AC-14: over 10 min, the agent container's average CPU is under 5 m and its working set under 20 MiB (from cAdvisor).
- AC-15: deleting one agent pod gives a gap of no more than about 10 s on that node. Patching the DaemonSet's nodeSelector so it matches no node makes `NodeAgentMissing` fire within about 3 min. Revert afterwards.
- AC-16: after `kubectl -n monitoring delete pod prometheus-kps-prometheus-0`, samples from before the restart are still queryable.

**Grafana**
- AC-17: both dashboard UIDs exist with no manual import. The `node` variable query returns 3 values. Every panel's query evaluates without error, checked by running each target `expr` through the Prometheus API with `$node=.*` and `$__rate_interval=4s`.
- AC-18: during AC-13, the stressed node is the result of `topk(1, node_agent_cpu_usage_ratio)`. This is the query behind the "Hottest node" panel.

**Teardown**
- AC-19: `make destroy && make verify-clean` reports nothing remaining.

### 6.3 `stress_node.sh`
`stress_node.sh <node> [seconds]` creates a Pod in `time-api`, pinned with `nodeName`, image `alexeiled/stress-ng` (pin it by digest), running `--cpu 0 --timeout <s>s` with requests `cpu: 10m`, so it is not blocked by scheduling. It waits for completion, then deletes the pod.

## 7. CI (`.github/workflows/ci.yaml`)
On every PR and push:
- job `app`: `make -C apps/time-api lint test`
- job `agent`: `make -C agents/node-agent lint test`
- job `infra`: `make -C infra/terraform fmt-check validate lint`
- job `observability`: `make -C deploy/helm/observability test`
- job `charts`: `helm lint` + `kubeconform` on all 3 charts, `shellcheck scripts/*.sh`
- job `images`: build both images (no push)
- job `kind-e2e` (on PRs to `main`): `make local-up local-smoke`

Pushing to ECR from CI through GitHub OIDC is optional and out of scope for the first delivery.

## 8. Phases

WS-50 runs in two separate launches. **Your launch prompt tells you which phase you are in.**

**Phase 1: scaffolding.** This runs in parallel with WS-10 to WS-40, whose code does **not** exist yet in your worktree. Do only the work below. Then write `HANDOFF.md` (label it Phase 1) and stop. Do not touch AWS, and do not wait for the other workstreams.

**Phase 2: end to end.** This runs on `main` after WS-10 to WS-40 are merged. Fix integration issues, but only in files you own; issues in other files go in `HANDOFF.md`. Run `make local-up local-smoke`. If the launch prompt allows AWS, also run `make tf-bootstrap up acceptance destroy verify-clean`. Complete §9 and update `HANDOFF.md` (labelled Phase 2).

Make every local port the Makefile uses (`grafana`, `prometheus`, `local-endpoint`) a variable with the defaults from §4.2, for example `GRAFANA_PORT ?= 3000`. That lets you override them with your 15000-range ports while testing (contracts §12).

### Phase 1 work list
- Write the Makefile, `deploy.env.example`, `.gitignore`, `scripts/kind-config.yaml`, `lib.sh`, `doctor.sh` and `verify_clean.sh` against the contracts.
- Write `smoke_test.sh` and `acceptance_test.sh` against the contracts, and test their helper functions with `bats` or by dry-running them against a kind cluster as pieces arrive.
- `make lint` and `make test` should work, skipping with a warning for components that are not merged yet.

## 9. Definition of Done
- [ ] Every target in §4.2 is implemented and idempotent. `shellcheck` is clean.
- [ ] `make local-up local-smoke` passes on a fresh machine with Docker and kind.
- [ ] On a fresh AWS account: `make tf-bootstrap up` passes, and `make acceptance` passes AC-2 to AC-18.
- [ ] `make destroy verify-clean` passes (AC-19).
- [ ] Following §5.3–5.4 by hand works first time, as recorded in `HANDOFF.md` with timings.
- [ ] The root `README.md` (quick start plus links to specs and the runbook) and `versions.md` (links to every component `VERSIONS.md`) are written.
- [ ] CI is green on `main`.
- [ ] `HANDOFF.md` is written (contracts §11).

# WS-20 — time-api (Python service + Helm chart)

Read first: `00-overview.md`, then `01-contracts.md` (especially §2, §3, §4, §7, §8).

## 1. Mission
Build a minimal, production-grade HTTP service that returns the current time, together with its container image and its Helm chart. The chart deploys 3 replicas, exactly one per node, behind an internet-facing NLB, and registers a ServiceMonitor so Prometheus scrapes the service's metrics.

## 2. You own
`apps/time-api/**` and `deploy/helm/time-api/**`. Do not create or edit files anywhere else.

## 3. Deliverables
```
apps/time-api/
├── Makefile               # lint, test, image (contracts §7.1)
├── pyproject.toml         # project metadata, ruff + mypy config
├── uv.lock
├── Dockerfile
├── .dockerignore
├── README.md              # run locally, test, build, endpoints
├── VERSIONS.md
├── HANDOFF.md
├── src/time_api/
│   ├── __init__.py        # __version__
│   ├── main.py            # FastAPI app, routes, lifespan, shutdown handling
│   ├── metrics.py         # prometheus_client registry, middleware, metrics server
│   └── settings.py        # env parsing
└── tests/
    ├── test_time.py
    ├── test_health.py
    └── test_metrics.py
deploy/helm/time-api/
├── Chart.yaml
├── values.yaml
├── values-local.yaml      # kind: service.type=ClusterIP
├── templates/
│   ├── _helpers.tpl
│   ├── deployment.yaml
│   ├── service.yaml           # public (LoadBalancer)
│   ├── service-metrics.yaml   # ClusterIP :9000
│   ├── pdb.yaml
│   ├── servicemonitor.yaml    # guarded by .Values.serviceMonitor.enabled
│   └── NOTES.txt
└── README.md              # values reference
```

## 4. Requirements

### 4.1 Application
- APP-1: Python **3.12+**, **FastAPI** and **uvicorn**, with **exactly one uvicorn worker per process**. Replicas give the scaling, which keeps `prometheus_client` single-process.
- APP-2: Implement the HTTP contract (contracts §3) exactly, including `405` and `404` with JSON bodies.
- APP-3: Read the clock **once** per `/time` request with `datetime.now(timezone.utc)`, and derive both `time` (with `isoformat(timespec="microseconds")` and `+00:00` replaced by `Z`) and `epoch_ms` from that one reading.
- APP-4: Serve metrics on `METRICS_PORT` (9000) using `prometheus_client.start_http_server`, started in the FastAPI lifespan. Request counting and latency come from a middleware that uses the **matched route template** (`request.scope["route"].path`), or `unmatched` when nothing matched. Health, ready and metrics paths are excluded. `time_api_build_info` is set at startup.
- APP-5: `X-Served-By` comes from env `POD_NAME`, falling back to `socket.gethostname()`.
- APP-6: Graceful shutdown. In Kubernetes, draining works like this:
  1. The `preStop` sleep (K-6) runs while the endpoints and the NLB deregister the pod.
  2. SIGTERM arrives.
  3. Uvicorn stops accepting connections and drains in-flight requests (`--timeout-graceful-shutdown 10`).

  The app also keeps a process-wide `shutting_down` flag. It is set at the start of lifespan shutdown, and while it is set `/readyz` returns 503. Do not replace uvicorn's own signal handling.
- APP-7: Logging is one JSON line per request to stdout: `ts` (RFC 3339), `level`, `method`, `path`, `status`, `duration_ms`, `pod`. Requests to health, ready and metrics paths are not logged at INFO. Uvicorn's own access log is disabled.
- APP-8: Settings come from env: `PORT=8000`, `METRICS_PORT=9000`, `LOG_LEVEL=info`, `APP_VERSION=0.0.0-dev`, `GIT_SHA=unknown`, `POD_NAME`. Invalid values make startup fail with a clear message.
- APP-9: Dependency management uses `uv`, and the lockfile is committed. Runtime dependencies: `fastapi`, `uvicorn[standard]`, `prometheus-client`. Dev dependencies: `pytest`, `httpx`, `ruff`, `mypy`.

### 4.2 Tests (`make -C apps/time-api test`)
- T-1: `/time` returns 200 with `application/json`, `Cache-Control: no-store` and an `X-Served-By` header.
- T-2: `time` parses as UTC, ends in `Z`, has 6 fractional digits, and is within ±2 s of the test's own clock.
- T-3: `epoch_ms` equals `time` converted to milliseconds, within ±1.
- T-4: `POST /time` returns 405 with JSON. `GET /nope` returns 404 with JSON.
- T-5: `/healthz` returns 200. `/readyz` returns 200, and 503 once the shutdown flag is set.
- T-6: After 3 calls, `time_api_requests_total{method="GET",route="/time",status="200"}` is 3 in the registry, the histogram count for `/time` is at least 3, and calling `/healthz` does not change either.
- T-7: `/metrics` on the app port returns 404.
- T-8 (container test, run in `make test` when Docker is available, otherwise skipped with a message): build the image, run it with `--read-only --user 10001 --tmpfs /tmp`, and curl `:8000/time` and `:9000/metrics`.
- `make lint` = `ruff check`, `ruff format --check`, and `mypy --strict src/`.

### 4.3 Container image
- IMG-1: Multi-stage `python:3.12-slim`. The builder stage runs `uv sync --frozen --no-dev` into `/opt/venv`. The runtime stage copies only the venv and `src/`.
- IMG-2: `USER 10001:10001`; `ENV PYTHONDONTWRITEBYTECODE=1 PYTHONUNBUFFERED=1 PATH=/opt/venv/bin:$PATH`; `ARG VERSION`/`ARG GIT_SHA` become `ENV APP_VERSION`/`GIT_SHA`.
- IMG-3: `CMD ["uvicorn","time_api.main:app","--host","0.0.0.0","--port","8000","--workers","1","--timeout-graceful-shutdown","10","--no-access-log"]`
- IMG-4: The image works with a read-only root filesystem (with an `emptyDir` at `/tmp`), and its final size is under 150 MB.
- IMG-5: `/bin/sleep` must be present, for the preStop hook. The slim base image has it.

### 4.4 Helm chart: Deployment
- K-1: `replicas: 3` (from values). The pod has the label `app.kubernetes.io/name: time-api`.
- K-2: `nodeSelector` comes from values (default `{workload: app}`). A **required** `podAntiAffinity` on `kubernetes.io/hostname`, against its own pods, guarantees one pod per node.
- K-3: **Strategy `RollingUpdate` with `maxSurge: 0`, `maxUnavailable: 1`.** This is mandatory: with exactly 3 nodes and required anti-affinity, a surge pod can never be scheduled and the rollout would hang.
- K-4: Env: `POD_NAME` (fieldRef `metadata.name`), `APP_VERSION` (`.Values.image.tag`, or `appVersion`), `GIT_SHA`, `LOG_LEVEL`.
- K-5: Probes:
  - readiness `GET /readyz` on `http`, period 5 s, timeout 2 s, failureThreshold 2
  - liveness `GET /healthz` on `http`, initialDelay 5 s, period 10 s, failureThreshold 3
- K-6: `lifecycle.preStop.exec.command: ["sleep","5"]` and `terminationGracePeriodSeconds: 30`, so the NLB can deregister the target before the process exits.
- K-7: Resources: requests `cpu: 50m`, `memory: 64Mi`; limits `memory: 256Mi`. No CPU limit.
- K-8: The security baseline from contracts §8, with `runAsUser: 10001` and an `emptyDir` at `/tmp`.

### 4.5 Helm chart: other objects
- K-9: **Service `time-api-metrics`**: ClusterIP, port 9000 → `metrics`, carrying the label `app.kubernetes.io/name: time-api`.
- K-10: **PodDisruptionBudget**: `minAvailable: 2`.
- K-11: **ServiceMonitor** (when `serviceMonitor.enabled`): it selects `time-api-metrics` and uses `endpoints: [{port: metrics, interval: 15s}]`, `jobLabel: app.kubernetes.io/name` and `namespaceSelector: {matchNames: [time-api]}`.
- K-12: `NOTES.txt` prints how to get the NLB hostname and curl `/time`.

### 4.6 Helm chart: public Service
- K-13: Service `time-api`: `type` comes from values (default `LoadBalancer`), port 80 → targetPort `http`, and `loadBalancerSourceRanges` comes from values. When the type is `LoadBalancer`, add these annotations:
  ```yaml
  service.beta.kubernetes.io/aws-load-balancer-type: external
  service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: ip
  service.beta.kubernetes.io/aws-load-balancer-scheme: internet-facing
  service.beta.kubernetes.io/aws-load-balancer-healthcheck-path: /readyz
  service.beta.kubernetes.io/aws-load-balancer-healthcheck-protocol: HTTP
  service.beta.kubernetes.io/aws-load-balancer-healthcheck-port: traffic-port
  service.beta.kubernetes.io/aws-load-balancer-target-group-attributes: deregistration_delay.timeout_seconds=10
  ```
- K-14: `values-local.yaml` sets `service.type: ClusterIP`.

### 4.7 Chart quality
- K-15: `helm lint` and `helm template … | kubeconform -strict -ignore-missing-schemas` pass with both the default values and `values-local.yaml`, and also with `serviceMonitor.enabled=false`.

## 5. Interfaces you provide
- `make -C apps/time-api lint test image IMAGE=… VERSION=… GIT_SHA=…`
- A chart that accepts the values keys in contracts §8, and produces the metrics in contracts §4 under `job="time-api"`.
- The pod labels and port names that WS-40's dashboards and alerts rely on (`deployment=time-api`, `namespace=time-api`).

## 6. Standalone testing
```bash
make -C apps/time-api lint test
make -C apps/time-api image IMAGE=time-api:dev
docker run --rm --read-only --tmpfs /tmp -p 8000:8000 -p 9000:9000 time-api:dev
curl -i localhost:8000/time ; curl -s localhost:9000/metrics | grep time_api_

# kind (contracts §8.1), with no Prometheus Operator present:
kind create cluster --config kind.yaml        # kind.yaml = contracts §8.1 (3 workers labelled workload=app)
kind load docker-image time-api:dev --name time-telemetry
helm install time-api deploy/helm/time-api -n time-api --create-namespace \
  -f deploy/helm/time-api/values-local.yaml \
  --set image.repository=time-api --set image.tag=dev --set serviceMonitor.enabled=false --wait
kubectl -n time-api get pods -o wide          # 3 pods, 3 distinct nodes
kubectl -n time-api port-forward svc/time-api 8080:80 &
for i in $(seq 60); do curl -s -D - -o /dev/null localhost:8080/time | grep -i x-served-by; done | sort | uniq -c
```
A port-forward pins the connection to one pod, so check that all 3 replicas serve traffic from inside the cluster. Run a temporary curl pod that hits `time-api.time-api.svc` 60 times and counts the distinct `X-Served-By` values; you should see 3.

Rollout test on kind: `helm upgrade … --set image.tag=dev2` (after `docker tag` and `kind load`) completes without hanging, and a 10 req/s loop running throughout sees at most 1% errors.

## 7. Definition of Done
- [ ] `make -C apps/time-api lint test` is green, including T-8 when Docker is available.
- [ ] The image is under 150 MB, runs read-only as UID 10001, and both ports work.
- [ ] The chart passes K-15. On kind: 3 pods on 3 distinct nodes, all 3 serve traffic, and an upgrade completes with at most 1% errors.
- [ ] `/metrics` is unreachable on port 8000 and through the `time-api` Service.
- [ ] `README.md` files for the app and the chart are written; `VERSIONS.md` is written; `HANDOFF.md` is written (contracts §11).

## 8. Out of scope
Alert rules and dashboards (WS-40), Terraform (WS-10), pushing images and the top-level Makefile (WS-50), TLS, auth, rate limiting, and HPA.

# 14 — Docker & Kubernetes

---

## 📖 Vocabulary

> *Definition* is the precise meaning; *Say it in an interview* is the quotable one-liner.

| Term | Definition | Say it in an interview |
|---|---|---|
| **Container** | An isolated process (or group) with its own filesystem view, network namespace and resource limits, sharing the host kernel. | "Containers isolate processes; they share the kernel, unlike VMs." |
| **Image** | An immutable, layered filesystem snapshot plus metadata (entrypoint, env, ports) from which containers are instantiated. | "An image is the immutable artifact; a container is a running instance." |
| **Dockerfile** | A text recipe of instructions that build an image layer by layer. | "The Dockerfile defines the build; order matters for caching." |
| **Layer / build cache** | Each instruction creates a filesystem layer; unchanged instructions reuse cached layers, so put rarely-changing steps first. | "Copy lockfiles before source so dependency layers stay cached." |
| **Base image** | The image a Dockerfile starts `FROM`; determines OS libraries and size. | "Slim base images cut size and attack surface." |
| **Multi-stage build** | A Dockerfile with multiple `FROM` stages where the final stage copies only needed artifacts from earlier ones. | "Multi-stage keeps compilers out of the runtime image." |
| **Digest** | The content-addressed SHA-256 identifier of an image; immutable, unlike tags. | "Pin by digest for reproducibility; tags can move." |
| **Registry** | A service storing and serving images (ECR, GHCR, Docker Hub, GitLab Registry). | "The registry is the artifact store for images." |
| **Non-root user** | Running the container process as an unprivileged UID, limiting damage if compromised; required by restricted Pod Security Standards. | "Non-root plus read-only filesystem limits the blast radius." |
| **Distroless** | Minimal images containing only the application runtime, no shell or package manager. | "Distroless removes the shell an attacker would use." |
| **`.dockerignore`** | File listing paths excluded from the build context. | "Exclude .git, tests and .env from the build context." |
| **Exec vs shell form** | `CMD ["uvicorn", ...]` (exec) runs the binary as PID 1 receiving signals; `CMD uvicorn ...` (shell) wraps it in `/bin/sh`, which may swallow SIGTERM. | "Use exec form so Uvicorn gets SIGTERM directly." |
| **Pod** | The smallest deployable Kubernetes unit: one or more containers sharing network and storage, scheduled together. | "A pod is one or more tightly coupled containers with one IP." |
| **Deployment** | A controller maintaining a desired number of identical pods from a template and performing rolling updates via ReplicaSets. | "Deployments are for stateless services like FastAPI." |
| **ReplicaSet** | Ensures a specified number of pod replicas are running; managed by Deployments per revision. | "Each Deployment revision is a ReplicaSet." |
| **StatefulSet** | A controller for stateful workloads providing stable network identities and persistent volumes per pod with ordered rollout. | "StatefulSets are for databases, not APIs." |
| **Service** | A stable virtual IP and DNS name load-balancing to pods matching a selector. | "A Service gives pods a stable address." |
| **Ingress / Gateway API** | Objects defining external HTTP(S) routing rules to Services, implemented by an ingress controller (NGINX, ALB). | "Ingress routes external traffic to Services." |
| **ConfigMap / Secret** | Kubernetes objects storing non-sensitive and sensitive configuration, consumable as env vars or mounted files. Secrets are base64 by default, not encrypted unless configured. | "Secrets aren't encrypted by default; enable KMS or use External Secrets." |
| **Probe** | Periodic container health check: **liveness** (restart on failure), **readiness** (remove from Service on failure), **startup** (gate other probes until ready). | "Liveness restarts, readiness un-routes, startup waits." |
| **Requests / limits** | `requests`: resources guaranteed and used for scheduling. `limits`: hard caps; memory over limit → OOMKilled, CPU over limit → throttled. | "Requests schedule; limits cap." |
| **QoS class** | Guaranteed (requests = limits), Burstable, BestEffort; determines eviction priority under node pressure. | "Requests = limits gives Guaranteed QoS." |
| **OOMKilled** | Container terminated by the kernel for exceeding its memory limit; exit code 137. | "OOMKilled: raise the limit or fix the leak." |
| **CrashLoopBackOff** | Pod state where the container repeatedly exits and Kubernetes restarts it with increasing delay. | "CrashLoopBackOff: read `--previous` logs and the exit code." |
| **HPA** | Horizontal Pod Autoscaler: adjusts replicas based on metrics relative to targets. | "HPA scales pods; requests must be set for utilization." |
| **PDB (PodDisruptionBudget)** | Policy limiting how many pods may be voluntarily disrupted (drains, upgrades) at once. | "PDBs keep capacity during node maintenance." |
| **Rolling update** | Deployment strategy replacing pods incrementally, governed by `maxSurge` and `maxUnavailable`. | "`maxUnavailable: 0` makes rollouts zero-downtime." |
| **`preStop` hook** | A command executed before SIGTERM is sent to the container, e.g., a short sleep for endpoint deregistration to propagate. | "preStop sleep avoids 502s during rollout." |
| **SIGTERM / grace period** | The termination signal sent to PID 1; after `terminationGracePeriodSeconds` Kubernetes sends SIGKILL. | "Handle SIGTERM and finish in-flight requests within the grace period." |
| **Topology spread / anti-affinity** | Scheduling constraints distributing pods across zones/nodes for resilience. | "Spread pods across AZs so one zone loss doesn't take the service." |
| **ServiceAccount / IRSA** | Pod identity in Kubernetes; IRSA maps it to an AWS IAM role via OIDC federation. | "IRSA gives pods AWS permissions without keys." |
| **NetworkPolicy** | Kubernetes firewall rules controlling which pods/namespaces may talk to which. | "NetworkPolicies enforce least-privilege networking." |
| **Pod Security Standards** | Cluster-level policies (privileged/baseline/restricted) constraining pod security settings. | "Restricted PSS requires non-root and dropped capabilities." |
| **External Secrets Operator** | A controller syncing secrets from external managers (AWS Secrets Manager, Vault) into Kubernetes Secrets. | "ESO keeps Kubernetes Secrets in sync with the secrets manager." |
| **Availability Zone (AZ)** | An isolated data-center location within a cloud region with independent power and networking. | "Multi-AZ survives a data-center failure." |
| **GitOps** | Operating model where the desired state of the cluster is declared in git and a controller reconciles the cluster to it. | "Git is the source of truth; ArgoCD applies it." |
| **Helm / Kustomize** | Helm: templated packages (charts) with values. Kustomize: patch-based overlays on base manifests. | "Helm templates; Kustomize overlays." |
| **`kubectl rollout undo`** | Command reverting a Deployment to a previous ReplicaSet revision. | "Rollout undo is the emergency rollback; then reconcile git." |

---

## Q1. How do you Dockerize a FastAPI application?

Goals: small, reproducible, secure, fast to build. Use a **multi-stage** build on a **slim** base with a **lockfile** and a **non-root** user.

## Q2. What should your Dockerfile look like?

```dockerfile
# ---------- Stage 1: build dependencies ----------
FROM python:3.12-slim AS builder
ENV PIP_NO_CACHE_DIR=1 PIP_DISABLE_PIP_VERSION_CHECK=1 \
    UV_PROJECT_ENVIRONMENT=/opt/venv
RUN apt-get update && apt-get install -y --no-install-recommends build-essential libpq-dev \
    && rm -rf /var/lib/apt/lists/*
COPY --from=ghcr.io/astral-sh/uv:latest /uv /usr/local/bin/uv
WORKDIR /src
COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --no-dev --no-install-project

# ---------- Stage 2: runtime ----------
FROM python:3.12-slim AS runtime
ENV PYTHONDONTWRITEBYTECODE=1 PYTHONUNBUFFERED=1 \
    PATH="/opt/venv/bin:$PATH"
RUN apt-get update && apt-get install -y --no-install-recommends libpq5 curl \
    && rm -rf /var/lib/apt/lists/* \
    && groupadd -r app && useradd -r -g app -d /app -s /sbin/nologin app
WORKDIR /app
COPY --from=builder /opt/venv /opt/venv
COPY --chown=app:app app ./app
COPY --chown=app:app alembic.ini ./
COPY --chown=app:app migrations ./migrations
USER app
EXPOSE 8000
HEALTHCHECK --interval=30s --timeout=3s CMD curl -fsS http://localhost:8000/health/live || exit 1
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000", "--workers", "1", \
     "--proxy-headers", "--forwarded-allow-ips=*", "--timeout-graceful-shutdown", "20"]
```
### 🔍 Line by line
- `FROM python:3.12-slim AS builder` — start the build stage from a small Debian+Python image; name it `builder`. In production pin a digest.
- `ENV PIP_NO_CACHE_DIR=1 …` — don't keep pip's download cache; install deps into `/opt/venv`.
- `RUN apt-get … build-essential libpq-dev && rm -rf /var/lib/apt/lists/*` — compilers and Postgres headers *only in this stage*; delete apt indexes in the same layer to keep it small.
- `COPY --from=ghcr.io/astral-sh/uv:latest /uv /usr/local/bin/uv` — bring in `uv`, a fast lock-based installer (poetry or pip-tools work the same way).
- `COPY pyproject.toml uv.lock ./` — copy **only** the dependency manifests first → this layer is cached until dependencies change.
- `RUN uv sync --frozen --no-dev --no-install-project` — install exactly the locked versions, no dev tools, no project code yet.
- `FROM python:3.12-slim AS runtime` — a **fresh** stage: no compilers, no build leftovers.
- `PYTHONDONTWRITEBYTECODE=1` — no `.pyc` files; `PYTHONUNBUFFERED=1` — logs stream immediately; `PATH=/opt/venv/bin:$PATH` — use the venv's Python.
- `apt-get install libpq5 curl` — only runtime libraries; `groupadd/useradd` — create an unprivileged `app` user with no login shell.
- `COPY --from=builder /opt/venv /opt/venv` — bring the ready virtualenv across.
- `COPY --chown=app:app app ./app` — copy source **last** (changes most often) owned by the app user.
- `USER app` — drop privileges from here on.
- `EXPOSE 8000` — documentation of the port (doesn't publish it).
- `HEALTHCHECK` — Docker-level health (Kubernetes uses probes instead; harmless).
- `CMD [...]` — **exec form** (JSON array) so Uvicorn is PID 1 and receives SIGTERM directly. One worker per container; `--proxy-headers` trusts the load balancer's forwarded headers; graceful shutdown lets in-flight requests finish.

Add a `.dockerignore` (`.git`, `tests`, `.venv`, `__pycache__`, `.env`) so they never enter the build context.

## Q3. Why use a non-root user?

Container isolation is not a security boundary to rely on alone. If the app is exploited, `root` inside the container can write anywhere, install tools, and try kernel/runtime escapes. An unprivileged user (plus read-only root filesystem and dropped capabilities) shrinks the blast radius. Many clusters enforce it via Pod Security Standards — a root image won't even schedule.

## Q4. How do you reduce FastAPI image size?

- Slim base (~150 MB vs ~1 GB full). Alpine is smaller but its `musl` libc breaks many Python wheels — usually not worth it.
- Multi-stage: no compilers in runtime.
- No dev deps, tests, `.git` (`.dockerignore`).
- Install + cleanup in one `RUN`.
- Use wheels; `--no-cache-dir`.
- `dive` to inspect fat layers. Typical result: 180–250 MB.
- Distroless (`gcr.io/distroless/python3`) if you don't need a shell.

## Q5. What is a multi-stage Docker build?

Several `FROM` stages in one Dockerfile; later stages `COPY --from=` earlier ones. Build tools, secrets and intermediate files stay in the builder and never reach the final image → smaller, safer, better caching.

---

## Q6. How do you pass environment variables to FastAPI?

The app reads them with `pydantic-settings`. Delivery:
- `docker run -e DATABASE_URL=... --env-file .env`
- Compose: `environment:` / `env_file:`
- Kubernetes: `env:` literals, `valueFrom.configMapKeyRef` / `secretKeyRef`, or `envFrom: [{configMapRef}, {secretRef}]`.
Never bake them into the image (`ENV DATABASE_URL=` in a Dockerfile is a leak).

## Q7. How do you manage secrets in Kubernetes?

Kubernetes `Secret`s are base64, **not encrypted** by default — enable encryption at rest (KMS) and restrict RBAC. Better: keep the source of truth in AWS Secrets Manager/Vault and sync with **External Secrets Operator** or mount via the **Secrets Store CSI driver**; the pod's IAM role (IRSA) authorises the fetch. Prefer files over env vars (env vars leak into `kubectl describe`, crash dumps, child processes). Rotate regularly; never commit real values (Sealed Secrets / SOPS if secrets must live in git).
```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata: { name: orders-api }
spec:
  refreshInterval: 1h
  secretStoreRef: { name: aws-secrets-manager, kind: ClusterSecretStore }
  target: { name: orders-api-secrets }
  data:
    - secretKey: DATABASE_URL
      remoteRef: { key: prod/orders-api, property: database_url }
    - secretKey: JWT_SECRET
      remoteRef: { key: prod/orders-api, property: jwt_secret }
```
- `refreshInterval` — re-sync hourly (picks up rotations).
- `secretStoreRef` — how to reach Secrets Manager (configured once with IRSA).
- `target.name` — the Kubernetes Secret to create.
- `data[]` — map each key from the remote JSON secret.

---

## Q8. How do you deploy FastAPI on Kubernetes?

Objects: `Deployment`, `Service`, `Ingress`, `ConfigMap`/`Secret`, `HPA`, `PodDisruptionBudget`, `ServiceAccount` (IRSA). Packaged as Helm/Kustomize; applied by GitOps.
```yaml
apiVersion: apps/v1
kind: Deployment
metadata: { name: orders-api, labels: { app: orders-api } }
spec:
  replicas: 3
  selector: { matchLabels: { app: orders-api } }
  strategy:
    type: RollingUpdate
    rollingUpdate: { maxSurge: 1, maxUnavailable: 0 }
  template:
    metadata:
      labels: { app: orders-api, version: "1.4.2" }
      annotations: { prometheus.io/scrape: "true", prometheus.io/port: "8000" }
    spec:
      serviceAccountName: orders-api
      securityContext: { runAsNonRoot: true, runAsUser: 1000, fsGroup: 1000, seccompProfile: { type: RuntimeDefault } }
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: ScheduleAnyway
          labelSelector: { matchLabels: { app: orders-api } }
      containers:
        - name: api
          image: 123456789.dkr.ecr.eu-west-1.amazonaws.com/orders-api:1.4.2
          ports: [{ containerPort: 8000, name: http }]
          envFrom:
            - configMapRef: { name: orders-api-config }
            - secretRef: { name: orders-api-secrets }
          resources:
            requests: { cpu: "500m", memory: "512Mi" }
            limits:   { cpu: "1",    memory: "512Mi" }
          startupProbe:
            httpGet: { path: /health/live, port: http }
            failureThreshold: 30
            periodSeconds: 2
          readinessProbe:
            httpGet: { path: /health/ready, port: http }
            periodSeconds: 5
            failureThreshold: 3
          livenessProbe:
            httpGet: { path: /health/live, port: http }
            periodSeconds: 10
            failureThreshold: 3
          lifecycle:
            preStop: { exec: { command: ["sh", "-c", "sleep 5"] } }
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities: { drop: ["ALL"] }
          volumeMounts: [{ name: tmp, mountPath: /tmp }]
      volumes: [{ name: tmp, emptyDir: {} }]
      terminationGracePeriodSeconds: 30
---
apiVersion: v1
kind: Service
metadata: { name: orders-api }
spec:
  selector: { app: orders-api }
  ports: [{ port: 80, targetPort: http }]
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata: { name: orders-api }
spec:
  minAvailable: 2
  selector: { matchLabels: { app: orders-api } }
```
### 🔍 Line by line (the important ones)
- `replicas: 3` + `maxUnavailable: 0`, `maxSurge: 1` — during an update add one new pod, never drop below three → zero downtime.
- `serviceAccountName` — bound to an IAM role (IRSA) for S3/SQS/Secrets access.
- `securityContext.runAsNonRoot` / `seccompProfile` — hardening.
- `topologySpreadConstraints` — spread pods across zones.
- `image: …:1.4.2` — immutable tag, never `latest`.
- `envFrom` — inject all keys from the ConfigMap and Secret as env vars.
- `resources` — memory request = limit (predictable); CPU request realistic.
- `startupProbe` — up to 60 s (30 × 2 s) for slow boots before liveness starts.
- `readinessProbe` → `/health/ready` (checks DB); `livenessProbe` → `/health/live` (process only).
- `preStop: sleep 5` — gives the load balancer time to stop sending traffic before SIGTERM.
- `readOnlyRootFilesystem` + `emptyDir` for `/tmp` — the container can only write to `/tmp`.
- `terminationGracePeriodSeconds: 30` — time for in-flight requests after SIGTERM.
- `Service` — stable name `orders-api` on port 80 → pods' port 8000.
- `PodDisruptionBudget minAvailable: 2` — node drains can't take more than one pod at a time.

## Q9. Deployment vs StatefulSet?

- **Deployment** — interchangeable stateless pods, random names, free scaling/rolling. **FastAPI belongs here.**
- **StatefulSet** — stable identities (`db-0`), per-pod persistent storage, ordered start/stop. For databases, Kafka.
If an API "needs" a StatefulSet, that's a design smell — move the state to a DB/Redis/S3.

## Q10. How do you configure readiness probes?

Readiness = "can this pod take traffic right now?" Failure → removed from the Service, **not** restarted. Check what's needed to serve (DB reachable with a timeout, config loaded). Don't check optional dependencies. Tune `periodSeconds`/`failureThreshold` so a blip doesn't flap.

## Q11. How do you configure liveness probes?

Liveness = "is this process hopelessly stuck?" Failure → **restart**. Keep it trivial and dependency-free. A liveness probe that checks the DB restarts every pod during a DB outage — making things worse. Use `startupProbe` for slow boots instead of a long `initialDelaySeconds`.

## Q12. What should /health return?

```python
@router.get("/health/live", include_in_schema=False)
async def live():
    return {"status": "ok"}

@router.get("/health/ready", include_in_schema=False)
async def ready(db: DB, redis: RedisDep):
    checks = {}
    async with asyncio.timeout(2):
        checks["db"] = await _check(db.execute(text("SELECT 1")))
        checks["redis"] = await _check(redis.ping())
    ok = all(checks.values())
    return JSONResponse({"status": "ready" if ok else "degraded", "checks": checks,
                         "version": settings.version}, status_code=200 if ok else 503)
```
- `/live` — always 200 if the loop can respond.
- `/ready` — each dependency check with a 2 s cap; 503 if any *required* one fails; include version for debugging. Keep it unauthenticated but not internet-exposed (ingress rule).

## Q13. What is the difference between liveness and readiness?

| | Liveness | Readiness |
|---|---|---|
| Question | Restart this container? | Send traffic to this pod? |
| On failure | Kill & restart | Remove from Service endpoints |
| Checks | Process only | Dependencies needed to serve |
| Classic bug | Checking the DB → restart storms | Missing it → 502s during rollout |

---

## Q14. How do you perform zero-downtime deployments?

> 🔴 **Moved to the advanced section** → [../../advanced-backend/07-deployments-and-release.md](../../advanced-backend/07-deployments-and-release.md)
> 
> Q1 — why rolling deploys drop requests, and the three-part fix.

---

## Q15. How do you configure resource requests and limits?

- **Requests** — what the scheduler reserves; HPA utilisation is relative to it. Set from observed p95 usage.
- **Limits** — hard cap. Memory over limit → OOMKilled. CPU over limit → throttled (latency spikes).
- Guidelines: memory `request == limit` (Guaranteed QoS); CPU request realistic, CPU limit generous or omitted (avoid throttling); one Uvicorn worker per ~1 CPU; use VPA in recommendation mode; namespace `LimitRange`/`ResourceQuota`.

## Q16. How does HPA work?

Every ~15 s the HPA controller reads metrics (Metrics Server for CPU/memory; Prometheus Adapter/KEDA for custom), computes `desired = ceil(current × currentMetric / targetMetric)`, applies scale-up/down policies and stabilisation windows, and patches the Deployment's replica count. Pods need resource requests for utilisation metrics. Works with Cluster Autoscaler/Karpenter to add nodes when pods are `Pending`.

---

## Q17. How would you troubleshoot CrashLoopBackOff?

> 🔴 **Moved to the advanced section** → [../../advanced-backend/07-deployments-and-release.md](../../advanced-backend/07-deployments-and-release.md)
> 
> Q10 — the exit-code table and the ordered checklist.

---

## Q18. How would you troubleshoot OOMKilled?

> 🔴 **Moved to the advanced section** → [../../advanced-backend/07-deployments-and-release.md](../../advanced-backend/07-deployments-and-release.md)
> 
> Q11 — container vs node OOM, and the Python-specific causes.

---

## Q19. How would you troubleshoot slow Kubernetes pods?

1. **CPU throttling** — `container_cpu_cfs_throttled_periods_total` high → raise/remove CPU limit.
2. **Node pressure** — `kubectl top node`; noisy neighbours; use requests or dedicated node pools.
3. **Network** — cross-AZ hops to the DB; DNS latency (`ndots:5` issue → set `ndots:2` or use FQDNs); conntrack exhaustion.
4. **Readiness flapping** — connections reset.
5. **Cold start** — big imports/model loads → startup probe + pre-warming.
6. **Storage** — slow PVC for anything writing to disk (log to stdout instead).
7. Compare a slow pod vs a fast one with `py-spy` and tracing.

## Q20. How would you perform a rollback?

> 🔴 **Moved to the advanced section** → [../../advanced-backend/07-deployments-and-release.md](../../advanced-backend/07-deployments-and-release.md)
> 
> Q12 — mitigate-first incident handling.

---

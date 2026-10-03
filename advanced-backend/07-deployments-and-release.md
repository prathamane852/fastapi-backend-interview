# 07 — Deployments & Release Engineering

> **How to use this file.** Every question has a **🗣️ Say this** block — 4 to 5 lines, ready to speak. Output blocks are marked **(what you'd see)**.

**The framing that scores at senior level:** most outages are caused by a deploy. Not hardware, not an AZ failing — a change someone shipped. So deployment safety is the highest-leverage availability work there is. That thread runs through this whole file.

---

## 📖 Jargon, in plain words

| Jargon | What it actually means | How you'd say it |
|---|---|---|
| **Zero-downtime deploy** | Replacing running instances without a single failed request. | "It comes from graceful shutdown plus readiness gating, not from the strategy." |
| **Rolling update** | Replace pods a few at a time. The Kubernetes default. | "Old and new run at once, so the contract must work for both." |
| **Blue-green** | Two complete environments; flip traffic between them. | "Instant rollback, double the infrastructure — and the database is still shared." |
| **Canary** | Send a small percentage of traffic to the new version and watch. | "It limits blast radius and gives you real production signal." |
| **Expand-contract** | Change a schema in backward-compatible steps across several releases. | "It's the only safe way to change a schema while both versions run." |
| **Feature flag** | A runtime switch that separates deploying code from releasing it. | "I can deploy Tuesday and release Thursday, and turn it off without a rollback." |
| **GitOps** | Git is the source of truth; a controller makes the cluster match it. | "Drift gets corrected automatically and every change is a reviewed commit." |
| **Readiness probe** | "Can I serve traffic right now?" Failing removes you from the load balancer. | "Dependency checks belong here." |
| **Liveness probe** | "Am I wedged?" Failing **restarts** the container. | "Never check dependencies here — that's how you restart every pod at once." |
| **`preStop` hook** | A command that runs *before* the shutdown signal. | "It covers the race between pod termination and the load balancer noticing." |
| **PDB** | PodDisruptionBudget — limits how many pods a voluntary operation can take down. | "It stops a node drain removing every replica at once." |
| **Error budget** | How much unreliability your SLO actually allows. | "It turns 'ship fast' versus 'stay up' into arithmetic instead of an argument." |

---

# Section A — Zero-downtime mechanics

## Q1. A rolling deploy drops requests. Why?

**This is the single best deployment question, and the answer is a race condition — not a config option.**

### 🗣️ Say this
> "It's a race between two things that happen *in parallel*, not in sequence. When Kubernetes deletes a pod, the kubelet sends the shutdown signal to your container, and separately the endpoints controller removes the pod from the Service.
> That removal then has to propagate — to kube-proxy on every node, and to the load balancer's target group. That takes a few seconds.
> So for those few seconds the pod has already started shutting down while traffic is still being routed to it. Those requests fail.
> The fix has three parts, and I'd give all three because each covers a different piece of the race: a `preStop` sleep, `maxUnavailable: 0`, and proper graceful shutdown in the app."

### 💻 The timeline that makes it click
```
  t=0    kubectl delete pod
         ├──> kubelet sends SIGTERM ──────────> app starts shutting down
         └──> endpoints controller removes pod
                    │
                    ├──> kube-proxy on node A  (takes ~1-3s)
                    ├──> kube-proxy on node B
                    └──> ALB target group      (takes ~5-10s)

  t=0 to t=~5s:  app is shutting down BUT still receiving traffic  <- failed requests
```

### 💻 The three-part fix
```yaml
spec:
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0                 # 2. never drop below desired capacity
  template:
    spec:
      terminationGracePeriodSeconds: 60 # must exceed preStop + longest request
      containers:
      - name: app
        lifecycle:
          preStop:
            exec:
              # 1. keep serving normally while the removal propagates
              command: ["sh", "-c", "sleep 15"]
```
```python
# 3. graceful shutdown: stop accepting, let in-flight finish, then close resources
@asynccontextmanager
async def lifespan(app: FastAPI):
    app.state.ready = True
    yield
    app.state.ready = False       # readiness starts failing immediately
    await asyncio.sleep(5)        # let the probes observe it
    await engine.dispose()        # then close the pool
```

### ⚠️ Follow-up: "Why `preStop: sleep` — isn't graceful shutdown enough?"
> "No, and the reason is the ordering. Graceful shutdown starts when SIGTERM arrives, but the problem is the traffic still arriving *at that moment*. The `preStop` hook runs *before* SIGTERM, so the container keeps serving completely normally while the endpoint removal propagates. It's a deliberately dumb fix and it's the correct one."

⚠️ **`maxUnavailable: 0` matters too** — the default is 25%, meaning you deliberately run below capacity during every deploy. That's fine at twenty replicas and an outage at two.

---

## Q2. Liveness vs readiness vs startup probes.

### 🗣️ Say this
> "Readiness answers 'can I serve traffic right now' — failing it removes you from the load balancer but doesn't kill you. Liveness answers 'am I wedged' — failing it **restarts the container**. Startup handles slow boots by suspending the other two until the app is up.
> The classic mistake, and I've seen it cause a real outage, is checking dependencies in the *liveness* probe. If your liveness check queries Postgres and Postgres has a thirty-second blip, Kubernetes restarts every single pod simultaneously. Now you have a database problem *and* a cold-start stampede hammering the database you were trying to protect.
> So dependency checks belong in readiness, where failing just takes you out of rotation.
> And honestly — when in doubt, don't have a liveness probe at all. A missing one has never caused an outage; a badly configured one causes them regularly."

### 💻
```python
@router.get("/livez")
async def livez():
    return {"status": "ok"}          # process is alive. That is ALL it means.

@router.get("/readyz")
async def readyz(session=Depends(get_session)):
    if not app.state.ready:
        raise HTTPException(503, "draining")
    try:
        async with asyncio.timeout(2):            # ALWAYS bound the check
            await session.execute(text("SELECT 1"))
    except Exception:
        raise HTTPException(503, "database unavailable")
    return {"status": "ready"}
```
⚠️ **Two more details:** the probe timeout must be shorter than its period, or probes pile up. And cache the dependency check for a second or two — a 2-second probe across 50 pods is 25 pointless `SELECT 1` queries every second.

---

## Q3. Rolling, blue-green or canary?

### 🗣️ Say this
> "Canary for anything user-facing where I have the metrics to judge it, because it's the only one that gives me real production signal at limited blast radius — 5% of traffic for ten minutes tells me things no staging environment can.
> Blue-green when rollback speed is the dominant concern and I can afford double infrastructure, typically for a big risky release.
> Rolling as the sensible default for internal services where the change is routine.
> But the point I'd make regardless of which they pick: **none of them protect you from the database.** Blue and green share one database. Canary and stable share one database. So if the migration isn't backward-compatible, your instant rollback reverts the code and leaves the schema — and now the old code is broken too. The compatibility discipline matters far more than the strategy."

| | Rolling | Blue-Green | Canary |
|---|---|---|---|
| Infra cost | 1× + surge | **2×** | 1× + a little |
| Rollback speed | Minutes | **Seconds** | Seconds |
| Blast radius | Grows during rollout | **100% at once** | **Controlled %** |
| Real signal before full rollout | ❌ | ❌ | ✅ |
| Protects you from a bad migration | ❌ | ❌ | ❌ |

---

## Q4. Rename a column with zero downtime.

### 🗣️ Say this
> "The principle first: during a rolling deploy both versions of the code are live, so the schema has to work for both simultaneously. A plain `RENAME COLUMN` breaks every running pod the instant it commits.
> So it's four releases. Expand — add the new column, nullable, unused. Then deploy code that writes to both columns but still reads the old one. Then backfill the historical rows in batches and switch reads to the new column. Then contract — drop the old one.
> People push back that four releases to rename a column is absurd, and my answer is that it's four *small* releases each independently reversible, versus one release that can't be rolled back at all.
> The backfill deserves its own emphasis — batched with a sleep, because an unbatched update on a big table holds a transaction open for minutes, which blocks vacuum and blows up replication lag. I've seen that take a service down more often than the DDL itself."

### 💻 The four releases
| Release | Migration | Code | Why both versions work |
|---|---|---|---|
| **1. Expand** | `ADD COLUMN email_address text` | unchanged | New column is nullable and unused |
| **2. Dual-write** | — | writes both, reads `email` | Old code still uses `email` |
| **3. Backfill** | batched `UPDATE` | reads new, writes both | Both columns agree |
| **4. Contract** | `DROP COLUMN email` | writes new only | Nothing reads the old one |

```python
# The backfill MUST be batched
while True:
    r = conn.execute(text("""
        WITH batch AS (
          SELECT id FROM users
          WHERE email_address IS NULL AND email IS NOT NULL
          LIMIT 5000 FOR UPDATE SKIP LOCKED
        )
        UPDATE users u SET email_address = u.email
        FROM batch b WHERE u.id = b.id
    """))
    conn.commit()
    if r.rowcount == 0: break
    time.sleep(0.1)          # let autovacuum and replication keep up
```

📐 See the DDL safety table in [03-postgres-deep-dive.md](03-postgres-deep-dive.md) Q13 — knowing which operations lock is half of this answer.

---

## Q5. Where do migrations run in the pipeline?

### 🗣️ Say this
> "A separate pipeline stage, or a Kubernetes Job, *before* the new code rolls out. Never in the application entrypoint — because N pods all race to run it, and you get lock contention or a half-applied migration and a confusing failure.
> Running it before the code is safe precisely *because* of expand-contract: every migration is backward-compatible with the running version by construction.
> I'd want it as an explicit, visible, separately-approved step, because a bad migration is the one change you genuinely cannot undo with a rollback.
> And on rolling back a migration — mostly you don't. If it dropped a column the data is gone, and the downgrade just recreates an empty one, which is worse than useless because it *looks* like it worked. That's the real argument for expand-contract: the destructive step is several releases away from the risky one."

| Option | Verdict |
|---|---|
| App startup entrypoint | ❌ **No** — N pods race |
| Kubernetes Job / init container | ✅ Workable — gate the rollout on success |
| **Separate pipeline stage** | ✅ **My preference** — explicit, approvable |
| ArgoCD PreSync hook | ✅ The GitOps-native version |

---

# Section B — Pipeline

## Q6. Design the pipeline for this service.

### 🗣️ Say this
> "Fast feedback first — lint and type-check take seconds and catch a real share of problems, so they run before anything expensive.
> Then tests against real dependencies as CI services, not mocks.
> Then **build once and promote the same artifact**. The image tagged with the commit SHA goes to staging and then to production completely unchanged, because rebuilding for production means shipping something you never tested. Immutable tags, never `latest`, so rollback is deterministic.
> And the deploy stage doesn't touch the cluster at all — it commits a tag change to a Git repo and lets ArgoCD reconcile. That keeps cluster credentials out of CI entirely."

### 💻
```yaml
stages: [validate, test, build, scan, deploy-staging, verify, deploy-prod]

validate:                     # seconds — cheapest signal first
  script: [ruff check ., mypy app/]
test:
  services: [postgres:16, redis:7]        # real deps, not mocks
  script: [pytest --cov=app --cov-fail-under=80 -n auto]
build:
  script:                                 # rootless, reproducible
    - /kaniko/executor --context . --destination $IMAGE:$CI_COMMIT_SHA
scan:
  script:
    - trivy image --exit-code 1 --severity CRITICAL,HIGH $IMAGE:$CI_COMMIT_SHA
    - pip-audit -r requirements.txt
deploy-staging:                           # commits to Git — ArgoCD does the rest
  script: [yq -i '.image.tag="'$CI_COMMIT_SHA'"' envs/staging/values.yaml, git push]
```

---

## Q7. What does GitOps actually buy you?

### 🗣️ Say this
> "Four concrete things. Drift correction — someone `kubectl edit`s a Deployment at 2am during an incident, and ArgoCD either reverts it or flags it, so the fix gets written down instead of living in one person's shell history.
> An audit trail for free, because every change is a reviewed commit with an author.
> Rollback as `git revert`, which anyone can do without cluster credentials.
> And crucially, CI no longer needs production credentials — the pipeline writes to Git and an in-cluster agent pulls. That takes cluster-admin off your CI runner, which is a significant attack-surface reduction."

### ⚠️ The honest trade-offs — say these too
> "It's slower to debug, because your change is in Git, then in ArgoCD's sync, then in the cluster, so 'why isn't my change live' has three places to look.
> Secrets need a separate solution since they can't live in Git, so you're adding Sealed Secrets or External Secrets.
> And emergency manual changes are genuinely harder — which is the point, but it hurts at 3am. So you need a documented break-glass procedure that people have actually practised."

---

## Q8. How does an automated canary work?

### 🗣️ Say this
> "You shift traffic in steps — 5%, then 25%, then 50% — with a pause at each, and an analysis job queries Prometheus continuously through those pauses. If the success rate drops below threshold twice in a row, it aborts and rolls back automatically.
> The value isn't the traffic splitting, it's the automated judgement. A human watching a dashboard notices in five or ten minutes; the analysis notices in one and rolls back without a decision meeting.
> The prerequisite everyone gets wrong is that the canary pods must be **labelled distinctly**, so you can query their metrics separately. If canary and stable metrics are aggregated together, a 5% canary failing completely barely moves the overall error rate and the analysis never fires.
> And I'd compare latency too, not just errors — a version that's correct but three times slower passes an error check and takes the service down at 100%."

### 💻
```yaml
kind: Rollout
spec:
  strategy:
    canary:
      analysis: {templates: [{templateName: success-rate}]}
      steps:
      - setWeight: 5
      - pause: {duration: 10m}      # analysis runs throughout
      - setWeight: 25
      - pause: {duration: 10m}
---
kind: AnalysisTemplate
spec:
  metrics:
  - name: success-rate
    successCondition: result[0] >= 0.99
    failureLimit: 2                 # 2 consecutive failures -> auto rollback
    provider:
      prometheus:
        query: |
          sum(rate(http_requests_total{version="canary",status!~"5.."}[2m]))
          / sum(rate(http_requests_total{version="canary"}[2m]))
```

---

## Q9. Deploy vs release — why separate them?

### 🗣️ Say this
> "Deploying is putting the code on the servers. Releasing is making it visible to users. Feature flags separate the two, and that changes the risk profile completely.
> I can deploy a half-finished feature behind a flag on Monday with no user impact, which means I'm merging to main continuously instead of nursing a long-lived branch that gets harder to merge every day. Then enable it internally, then 1%, then everyone.
> And if it's wrong, I flip a flag — seconds, no pipeline, no revert.
> That last part is the bit I'd stress: a rollback is a blunt instrument that reverts *everything* since the last good version. A flag reverts exactly one thing."

### ⚠️ The cost, stated honestly
> "Flags are technical debt with a half-life. Every flag doubles the code paths you'd have to test, and a codebase with two hundred stale flags is harder to reason about than one with none. So: an owner and an expiry date on every flag, and removing it is part of the definition of done — not a ticket for later, because that ticket never gets done."

---

# Section C — When it goes wrong

## Q10. `CrashLoopBackOff`. Debug it.

### 🗣️ Say this
> "First `kubectl describe pod` for the events and the last state, and `kubectl logs --previous` — that `--previous` flag is essential because it shows the *crashed* container's logs, not the new one that's about to crash.
> Then read the exit code, because it narrows things immediately. 137 is a kill signal, which means OOMKilled or it ignored the shutdown signal. 1 is an application error. 125 to 127 means the command couldn't run at all — bad entrypoint or wrong architecture.
> The usual causes are a missing config or secret, a failed startup dependency, OOM, or a liveness probe firing during a slow start — which a startup probe fixes.
> And if it *is* a startup dependency failure, I'd question the design too: a service that exits when Postgres has a blip will crash-loop through every database failover."

📐 **The exit code table**
| Code | Means |
|---|---|
| **0** | Exited cleanly — wrong entrypoint, a command that finished. K8s restarts it anyway |
| **1** | Application error — read the logs |
| **125–127** | Container couldn't run: bad entrypoint, not executable, wrong architecture |
| **137** | SIGKILL — **OOMKilled**, or ignored SIGTERM past the grace period |
| **139** | Segfault in a native extension |
| **143** | SIGTERM — terminated normally |

```
$ kubectl describe pod api-7d9f  | grep -A4 "Last State"
    Last State:     Terminated
      Reason:       OOMKilled
      Exit Code:    137
```

---

## Q11. `OOMKilled`. Now what?

### 🗣️ Say this
> "First, which OOM — did the container hit its own limit, or did the *node* run out and the kernel picked a victim? They have completely different fixes. A container OOM is a limit or a leak; a node OOM means requests are set too low across the cluster and the scheduler overcommitted.
> For a container OOM: is it a leak or is the limit just wrong? If memory climbs steadily and a restart resets it, that's a leak and I'd go to `tracemalloc`. If it plateaus just under the limit, the limit is wrong.
> Python-specific causes: unbounded caches, loading a full query result instead of streaming, no request body size limit, and too many Gunicorn workers — because memory scales with worker count and people set `2 * cores + 1` on a memory-limited container without doing the arithmetic."

### 📐 The non-obvious guidance worth knowing
> "Set memory request equal to limit, so the pod gets Guaranteed QoS and isn't evicted under node pressure.
> For CPU, set a request but **consider not setting a limit at all** — CPU limits cause throttling at the 100-millisecond quota boundary, and you get mysterious p99 spikes with CPU usage well below the limit. That one surprises people and it's a good thing to know."

---

## Q12. How do you run a deployment incident?

### 🗣️ Say this
> "**Mitigate first, diagnose second** — and the order is the whole answer. The instinct to understand the problem before acting is the wrong instinct during an incident. If it correlates with a deploy, roll back. You can find out why afterwards, from the artifact, at leisure.
> For that to work, rollback has to be one command and it has to be practised, because a rollback nobody has rehearsed turns into twenty minutes of arguing about whether to do it.
> Then declare it, assign an incident lead who isn't doing the hands-on work, and keep a running timeline in the channel — because the post-mortem needs it and nobody remembers accurately afterwards.
> Afterwards, blameless post-mortem with owners and dates. And I'd push for at least one action item about *detection*, not just the fix — 'we found out from a customer' is its own separate finding."

📐 **MTTR is mostly detection plus decision time, not fix time.** Better alerting and a trivial rollback reduce MTTR far more than engineers debugging faster.

---

## ⚡ Rapid fire

| They ask | Say this |
|---|---|
| What does `kubectl rollout undo` do? | "Reverts to the previous ReplicaSet. Only works within the revision history limit, and it doesn't touch ConfigMaps or the database." |
| Deployment or StatefulSet? | "StatefulSet gives stable identity, ordered rollout and per-pod volumes. A FastAPI service is a Deployment — your database isn't in Kubernetes anyway." |
| What's a PodDisruptionBudget for? | "It bounds *voluntary* disruptions like node drains and cluster upgrades. It does nothing for a node crashing." |
| ConfigMap change — does it restart pods? | "No. Env-var references need a restart; mounted volumes update eventually. Use a checksum annotation on the pod template to force a rollout." |
| Why multi-stage Docker builds? | "Build tools — compilers, dev headers — stay out of the runtime image. Smaller, faster to pull, much smaller attack surface." |
| Distroless base image? | "Fewer CVEs, smaller. The cost is no shell, so no `kubectl exec` debugging — you use ephemeral debug containers instead." |
| Why non-root? | "Limits the damage of a container escape, and most clusters enforce it via Pod Security Standards. Set `runAsNonRoot`, a numeric UID, and a read-only root filesystem." |
| Gunicorn worker count? | "Derive it from the **CPU limit**, not the node's cores. A container that sees 64 cores but gets 1 will spawn 129 workers and OOM immediately." |
| Sidecar or init container? | "Init runs to completion first; sidecar runs alongside. Kubernetes 1.29+ has native sidecars, which fixes the old problem of a sidecar keeping a Job alive forever." |
| How long should a deploy take? | "Short enough that you'd happily do it ten times a day. If it takes 40 minutes people batch changes, and batched changes are riskier changes." |
| DORA metrics? | "Deploy frequency, lead time for change, change failure rate, and MTTR. They're the numbers that connect shipping fast to staying up." |

---

**Next:** [08-opentelemetry-advanced.md](08-opentelemetry-advanced.md)

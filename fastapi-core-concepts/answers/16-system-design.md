# 16 — System Design Questions

---

## 📖 Vocabulary

> *Definition* is the precise meaning; *Say it in an interview* is the quotable one-liner.

| Term | Definition | Say it in an interview |
|---|---|---|
| **Functional requirements** | What the system must do: features, operations, data it manages. | "Functional requirements are the features." |
| **Non-functional requirements (NFRs)** | Quality attributes: scalability, latency, availability, consistency, security, cost, compliance. | "NFRs drive the architecture more than features do." |
| **Back-of-envelope estimation** | Rough order-of-magnitude calculation of traffic, storage and bandwidth to size the design. | "Estimates anchor every choice: rps, storage, bandwidth." |
| **Availability** | Fraction of time a system is operational and serving; expressed in 'nines'. 99.9% ≈ 8.7 h/yr downtime; 99.99% ≈ 52 min/yr. | "Each extra nine cuts allowed downtime tenfold." |
| **Reliability** | Probability the system performs correctly over time; availability plus correctness. | "Reliable means available *and* correct." |
| **Scalability** | Ability to handle increased load by adding resources with proportional gain. | "Scalable means more load is handled by adding capacity, not by rewrites." |
| **Consistency (strong vs eventual)** | Strong: every read sees the latest write. Eventual: replicas converge over time; reads may be stale briefly. | "Choose eventual consistency only where staleness is acceptable." |
| **CAP theorem** | Under a network partition, a distributed system must choose between consistency and availability. | "During a partition you pick C or A; the rest of the time you get both." |
| **Stateless service** | A service holding no client-specific state between requests, enabling any instance to serve any request. | "Stateless is the prerequisite for horizontal scaling." |
| **API gateway** | The edge component handling routing, authentication, rate limiting, TLS termination and observability for backend services. | "The gateway centralizes edge concerns." |
| **BFF (Backend-for-Frontend)** | An API layer tailored to a specific client type, aggregating and shaping backend responses. | "A BFF composes backend calls for one client's needs." |
| **CDN** | *Content Delivery Network*: globally distributed edge caches serving static and cacheable content near users. | "CDN absorbs read traffic at the edge." |
| **WAF** | *Web Application Firewall*: edge filtering of malicious HTTP traffic (injection patterns, bots, rate rules). | "WAF blocks abuse before it reaches the app." |
| **Multi-tenancy** | One deployment serving multiple isolated customers (tenants) with shared infrastructure. | "Multi-tenancy shares infra while isolating data per tenant." |
| **Row-Level Security (RLS)** | Postgres feature applying per-row access policies automatically based on session settings. | "RLS enforces tenant isolation in the database itself." |
| **Noisy neighbour** | One tenant's load degrading others in shared infrastructure; mitigated by fair-share limits and isolation tiers. | "Fair-share rate limits contain noisy neighbours." |
| **Pre-signed URL** | A time-limited, signed URL granting direct object-storage access without proxying through the API. | "Clients upload straight to S3 with pre-signed URLs." |
| **Multipart upload** | S3 mechanism uploading a large object in independent, parallel, retryable parts assembled on completion. | "Multipart makes multi-GB uploads parallel and resumable." |
| **Saga** | Distributed transaction pattern: a series of local transactions with compensating actions on failure. | "Sagas trade atomicity for compensation across services." |
| **Choreography vs orchestration** | Choreography: services react to events independently. Orchestration: a coordinator (Temporal, Step Functions) directs the workflow. | "Choreography for simple flows, orchestration for complex ones." |
| **Transactional outbox** | Writing events to an outbox table in the same transaction as the state change; a relay publishes them. | "The outbox guarantees events are never lost." |
| **CDC (Change Data Capture)** | Streaming database changes from the write-ahead log to downstream systems (Debezium). | "CDC feeds search and analytics without dual writes." |
| **Event sourcing** | Persisting state as an append-only sequence of events, with current state derived by replay. | "Event sourcing keeps the full history as the source of truth." |
| **Circuit breaker** | A stateful guard that stops calls to a failing dependency after a failure threshold and fails fast until a probe succeeds. | "Breakers fail fast to protect both sides." |
| **Bulkhead** | Isolating resources (connections, threads, semaphores) per dependency so one failure can't exhaust shared capacity. | "Bulkheads stop one slow dependency starving the rest." |
| **Timeout budget / deadline propagation** | Allocating a total time budget across a call chain and passing the remaining deadline downstream. | "Deadlines prevent work continuing after the client has given up." |
| **Retry budget** | A cap on retries as a fraction of traffic to avoid retry storms. | "Retry budgets stop retries amplifying an outage." |
| **Graceful degradation** | Continuing to serve reduced functionality when a dependency fails, rather than failing entirely. | "Degrade to stale or partial data instead of a 500." |
| **Active-passive / active-active** | Passive: a standby region takes over on failure. Active-active: multiple regions serve traffic simultaneously. | "Active-passive is simpler; active-active needs conflict handling." |
| **RTO / RPO** | Recovery Time Objective: max acceptable downtime. Recovery Point Objective: max acceptable data loss window. | "RTO is how fast we recover; RPO is how much data we may lose." |
| **Game day / chaos engineering** | Deliberately injecting failures in controlled conditions to validate resilience and runbooks. | "Game days prove the failover actually works." |
| **Golden path** | The supported, pre-configured way to build and deploy a service on a platform template. | "The golden path makes the right way the easy way." |
| **Sharding / partitioning** | Splitting data across nodes (sharding) or within a table (partitioning) by key to scale storage and throughput. | "Partition first; shard when one primary can't take the writes." |
| **Hot key / hot partition** | A key receiving disproportionate traffic, creating a bottleneck. | "Hot keys need caching, coalescing or key splitting." |
| **Idempotent API** | An API where retried requests (via an idempotency key) don't duplicate side effects. | "Idempotency keys make payment retries safe." |
| **Zero-downtime deployment** | Releasing new versions without any failed or dropped requests, via rolling updates, probes, graceful shutdown and compatible migrations. | "Zero downtime is rolling updates plus compatibility at every layer." |

---

## 🧭 How to answer any design question

1. **Clarify requirements** — functional and non-functional. Ask 2–3 sharp questions, then *state your assumptions* out loud.
2. **Estimate** — rps, storage, bandwidth. Numbers anchor every later choice.
3. **High-level architecture** — boxes and arrows: clients → edge → API → data/async.
4. **Deep-dive the critical path** — data model, API contract, the hard problem (consistency, idempotency, hot keys).
5. **Cross-cutting** — auth, observability, deployment, failure modes.
6. **Trade-offs & evolution** — what changes at 10× scale; what you'd cut for v1.

Each answer below follows that shape, condensed to a 10–15 minute spoken answer.

---

## D1. Design a scalable FastAPI microservice architecture.

**Assumptions:** several bounded contexts (users, orders, payments, notifications), independent teams, tens of thousands of rps aggregate, 99.9%+ availability.

**Architecture**
```
Clients → CDN/WAF → API Gateway (auth, rate limit, routing)
        → Services (FastAPI; each owns its DB, deploy, HPA)
        → Async backbone (Kafka/SQS) for events between services
        → Shared platform: IdP, Redis, S3, OTel Collector, Prometheus/Grafana/Loki/Tempo
```
- **Service template**: file 09 structure + a shared internal library (auth verification, logging, tracing, health, error envelope) so every service behaves the same.
- **Sync vs async**: synchronous REST only for request-path reads that need an immediate answer; everything else via events (`order.placed` → payment → notification). Outbox pattern for reliable publishing.
- **Data**: database per service; no shared tables; event-fed read models for cross-service queries; sagas instead of distributed transactions (choreography for simple flows, Temporal for complex ones).
- **Resilience**: timeouts, retries with jitter, circuit breakers, bulkheads, graceful degradation.
- **Security**: gateway validates JWT; mTLS between services; per-service IAM roles; scopes per caller.
- **Deployment**: EKS, one Helm chart per service, ArgoCD, canary rollouts, PDBs, multi-AZ.
- **Observability**: trace context through HTTP and messages; RED dashboards; SLO burn-rate alerts.
- **Evolution**: start as a modular monolith if the team is small; extract along stable boundaries.

---

## D2. Design a FastAPI API handling 100K requests/minute.

**Numbers:** 100k/min ≈ 1,700 rps average; plan for 3× peak ≈ 5k rps. If 80% are cacheable reads, the DB sees ~1k rps.

- **Edge**: CloudFront + WAF, TLS termination, cache public GETs (`Cache-Control`, ETag).
- **Ingress**: ALB → NGINX ingress; keep-alive to pods.
- **App**: async FastAPI, `orjson`, 1 worker/pod sized ~1 CPU (≈500–1,000 rps each) → 6–10 pods baseline, HPA to 30 on an rps metric.
- **Cache**: Redis cluster, cache-aside with stampede protection; per-user rate limits.
- **DB**: Postgres Multi-AZ + 2 read replicas; PgBouncer/RDS Proxy; verified indexes; cursor pagination.
- **Writes**: validate → persist → enqueue side effects → fast 201/202.
- **Protection**: rate limiting at gateway and app; per-pod concurrency limit; load shedding at ~90% saturation.
- **Verify**: k6 to 5k rps in staging with p99 < 300 ms; chaos test one AZ down.

---

## D3. Design an API gateway using FastAPI.

**Caveat first:** for a production gateway use Kong/Envoy/APISIX/AWS API Gateway. A FastAPI gateway makes sense as a **BFF** or when custom Python logic dominates (aggregation, per-tenant routing, response shaping).

- **Routing table** from config: prefix → upstream URL, with per-route auth/rate-limit policies.
- **Auth**: verify JWT once (JWKS); forward `X-User-Id`/`X-Roles`/`X-Tenant-Id`; strip any incoming spoofed copies.
- **Proxying**: shared `httpx.AsyncClient`; stream bodies both ways; copy safe headers; add `X-Forwarded-*`; propagate `traceparent`.
- **Resilience**: per-upstream timeouts, retries for idempotent methods, circuit breaker, bulkhead semaphores.
- **Rate limiting**: Redis sliding window per key.
- **Aggregation endpoints**: `gather` several services into one response.
- **Ops**: stateless, many small pods; hot-reloadable config.

```python
from starlette.background import BackgroundTask

@app.api_route("/{path:path}", methods=["GET", "POST", "PUT", "PATCH", "DELETE"])
async def proxy(path: str, request: Request, user: CurrentUser, http: HttpDep):
    upstream = resolve_upstream(path)
    if upstream is None:
        raise HTTPException(404)
    headers = forward_headers(request.headers) | {"X-User-Id": str(user.id)}
    async with breaker(upstream.name):
        req = http.build_request(request.method, f"{upstream.url}/{path}", headers=headers,
                                 content=request.stream(), params=request.query_params)
        resp = await http.send(req, stream=True)
    return StreamingResponse(resp.aiter_raw(), status_code=resp.status_code,
                             headers=filter_response_headers(resp.headers),
                             background=BackgroundTask(resp.aclose))
```
### 🔍 Line by line
- `"/{path:path}"` — the `:path` converter matches slashes, so one route catches everything.
- `resolve_upstream(path)` — look up the routing table.
- `forward_headers(...) | {...}` — dict union: safe headers plus trusted identity headers.
- `async with breaker(...)` — circuit breaker per upstream.
- `content=request.stream()` — forward the request body as a stream (no buffering).
- `http.send(req, stream=True)` — don't read the whole upstream body.
- `StreamingResponse(resp.aiter_raw(), ..., background=BackgroundTask(resp.aclose))` — stream bytes to the client; close the upstream response afterwards.

---

## D4. Design a multi-tenant FastAPI application.

**Key decision — isolation model**

| Model | Isolation | Cost | Ops | When |
|---|---|---|---|---|
| Shared DB & schema + `tenant_id` column | Lowest | Lowest | Easiest | Many small tenants (SaaS default) |
| Shared DB, schema per tenant | Medium | Medium | Migrations × N | Hundreds of tenants |
| Database per tenant | Highest | Highest | Hardest | Regulated/enterprise |

Default: shared schema with **Postgres Row-Level Security**; support "premium isolated" tenants via DB-per-tenant routing.

- **Tenant resolution**: from a JWT claim, subdomain, or header — resolved once in a dependency, never from the body.
- **Enforcement**: every table has `tenant_id`; RLS policy `USING (tenant_id = current_setting('app.tenant_id')::uuid)`; the session dependency sets it at transaction start so even a forgotten `WHERE` can't leak.
- **Repositories** take the tenant from context, never as a caller argument.
- **Per-tenant** limits, quotas, flags, entitlements; fair-share rate limiting; optional dedicated pods for big tenants.
- **Cache keys** and **S3 prefixes** per tenant; KMS key per tenant for the isolated tier.
- **Onboarding/offboarding** APIs; export and deletion workflows (GDPR).
- **Testing**: tenant A requesting tenant B's resource gets 404, for every endpoint.
```python
async def get_tenant_db(tenant: CurrentTenant) -> AsyncIterator[AsyncSession]:
    async with SessionLocal() as s:
        async with s.begin():
            await s.execute(text("SET LOCAL app.tenant_id = :t"), {"t": str(tenant.id)})
            yield s
```
- `SET LOCAL` — a per-transaction setting the RLS policy reads; it disappears when the transaction ends.

---

## D5. Design a FastAPI authentication service.

**Scope:** signup/login, tokens, refresh, logout, password reset, MFA, social login; used by many services.

- **Prefer buy**: Cognito/Auth0/Keycloak as the IdP; this service becomes a thin façade. If building:
- **Endpoints**: `POST /signup`, `POST /token` (password + PKCE code flows), `POST /refresh`, `POST /logout`, `POST /password/forgot|reset`, `POST /mfa/enroll|verify`, `GET /.well-known/jwks.json`, `GET /userinfo`.
- **Tokens**: RS256/ES256 access tokens (15 min) signed with a private key in KMS; `kid` rotation; refresh tokens hashed in DB, rotated on use, family-revocation on reuse.
- **Storage**: Postgres (users, credentials, refresh tokens, encrypted MFA secrets); Redis (attempt counters, denylist, one-time codes).
- **Security**: Argon2id; constant-time compares; rate limit + lockout with exponential delay; breached-password check; enumeration resistance; CSRF for cookie flows; audit log; session listing/revocation.
- **Consumers verify tokens locally via JWKS** — the auth service is *not* on the request path of others (availability).
- **Availability**: multi-AZ; JWKS cached by consumers for hours so an auth outage only blocks new logins.
- **Observability**: login success/fail rates, MFA challenges, token issuance, anomaly alerts.

---

## D6. Design an asynchronous document-processing API.

**Flow:** upload → validate → process (OCR/parse/ML) → notify → retrieve. Processing takes seconds to minutes.

1. `POST /documents` → returns a pre-signed S3 PUT URL + `document_id` (the API never streams file bytes).
2. Client uploads to S3; S3 event (or `POST /documents/{id}/complete`) → API validates (size/type via HEAD, magic bytes), writes `queued`, publishes `{document_id}` (outbox).
3. **Workers** (separate deployment; KEDA on queue depth; GPU pool for ML): download, run a pipeline of idempotent, checkpointed stages (`status`, `stage`, `progress`); results to S3 + metadata to Postgres; heartbeat to extend visibility timeout.
4. `GET /documents/{id}` → status/progress/result URL; webhook or SSE for push.
5. **Failure handling**: retries with backoff; DLQ with replay; poison detection; per-stage timeouts.
6. **Controls**: per-tenant quotas, virus scanning stage, PII redaction, retention/TTL, SSE-KMS encryption.
7. **Scale**: fan out large docs into page-level tasks with a fan-in aggregator; priority queues.
8. **Observability**: one trace from upload through every stage; queue lag; stage duration histograms; stuck-job detector.

---

## D7. Design a FastAPI service using PostgreSQL + Redis + SQS.

**Roles:** Postgres = system of record; Redis = cache, rate limits, locks, idempotency keys, pub/sub for invalidation; SQS = durable async work and cross-service events (FIFO where per-key ordering matters), with DLQs.

**Write path**
```
POST /orders (Idempotency-Key)
 → check Redis idempotency key (fast path)
 → BEGIN; insert order; insert outbox row; COMMIT       (atomic)
 → 201
Outbox relay (loop or Debezium) → SQS publish → mark sent
Worker consumes → idempotent processing → delete message
```
**Read path:** Redis → miss → Postgres (replica) → fill cache.

**Details:** pool sizes per pod; `pool_pre_ping`; Redis client with timeouts and fallback-to-DB on Redis failure (never a hard dependency for reads); SQS long polling, batch sends, visibility timeout > p99 processing; consumers scaled on queue depth; every message carries trace context and a version; readiness = DB, Redis/SQS degrade gracefully.

---

## D8. Design a distributed FastAPI system with OpenTelemetry.

**Goal:** one trace per user request across gateway → services → DB → queues → workers, with logs and metrics correlated.

- **SDK in every service** via a shared library: `TracerProvider` with resource attributes (`service.name`, `service.version`, `deployment.environment`); OTLP exporter → **Collector** (DaemonSet/sidecar); auto-instrument FastAPI, httpx, SQLAlchemy, Redis, botocore; `LoggingInstrumentor` to put `trace_id` in JSON logs.
- **Propagation**: `traceparent`/`baggage` over HTTP; message attributes over SQS/Kafka; consumer spans linked to producer spans.
- **Collector pipeline**: receivers (OTLP) → processors (batch, memory limiter, k8s attributes, PII redaction, **tail-based sampling**: 100% of errors and slow traces, 5% of the rest) → exporters (Tempo/Jaeger, Prometheus/Mimir, Loki).
- **Metrics**: pick OTel metrics *or* Prometheus client; exemplars link histogram buckets to trace IDs.
- **Dashboards**: Grafana with data links metric → trace → logs; service graph from span metrics.
- **SLOs/alerts** from span-derived RED metrics.
- **Cost**: sampling, attribute limits, drop health-check spans.
- **Governance**: semantic conventions for span/attribute names; trace ID returned to clients as `X-Request-ID`.

---

## D9. Design an API that must maintain 99.99% availability.

**Budget:** 99.99% = ~52 minutes downtime per year ≈ 4.3 min/month. Every dependency and every deploy consumes it; a single-region, single-DB design can't reliably hit it.

- **Redundancy everywhere**: ≥3 pods across 3 AZs with PDB and anti-affinity; Multi-AZ DB with automated failover (Aurora < 30 s); Redis cluster with replicas; multi-AZ LB/NAT.
- **Multi-region active-passive** (or active-active reads) with Route 53 health-checked failover and Aurora Global Database; regularly exercised failover runbook.
- **Change safety** (most outages are self-inflicted): canary with automatic rollback, feature flags, expand/contract migrations, no big-bang releases, freezes around peak.
- **Dependency isolation**: timeouts, breakers, bulkheads; degrade rather than fail (serve stale cache, disable non-critical features); no synchronous dependency on non-critical services.
- **Capacity**: ≥50% headroom over peak; autoscaling; load shedding; rate limiting.
- **Data safety**: PITR backups with tested restores; idempotent writes for safe client retries.
- **Observability & response**: SLO burn-rate paging; on-call with runbooks; automated remediation for known failures; game days/chaos.
- **Client side**: retries with jitter for idempotent calls; connection pooling; sensible timeouts.
- **Reality check**: 99.99% requires the whole organisation — error budgets and blameless postmortems.

---

## D10. Design an API with idempotent payment/order creation.

File 11 Q13's idempotency-key implementation is the core. System-level additions:
- **Client contract**: `Idempotency-Key` required on `POST /payments`; key = UUID per *intent*, reused on retries; documented: 201 first time, same 201 on replay, 409 while in progress, 422 if body differs.
- **Storage**: key scoped by `(merchant_id, key)`; store request hash, status, response, `expires_at` (24 h); PK is the atomic claim.
- **Two-phase safety with the PSP** (Stripe etc.): pass the same key as the PSP's idempotency key; persist the PSP intent ID **before** confirming; a reconciliation job queries the PSP by our key after crashes.
- **State machine**: `created → authorized → captured | failed`; idempotent transitions; version column for optimistic concurrency.
- **Exactly-once effects**: `UNIQUE(order_id)` on payments; append-only ledger.
- **Async completion**: PSP webhooks processed idempotently (dedupe by event ID); outbox for our own events.
- **Timeouts**: if the PSP call times out, don't retry blindly — check status by key first.
- **Observability**: metrics for replays, mismatches, in-progress collisions; alerts on reconciliation discrepancies.

---

## D11. Design an API supporting millions of users.

**Numbers:** e.g., 5M users, 10% DAU = 500k, 20 requests/user/day ≈ 115 rps average, 10× peak ≈ 1–2k rps — *modest* compute; the challenges are data volume, hot spots, and operations.

- Stateless FastAPI fleet with HPA; CDN for static/public; gateway auth.
- **Data**: Postgres with partitioning (time or hash on `user_id`); read replicas; PgBouncer; shard by `user_id` when a single primary's write throughput is the limit.
- **Caching**: profiles/sessions in Redis; hot entities; CDN for media.
- **Async**: notifications, emails, analytics via queues.
- **Search/analytics** offloaded to OpenSearch/ClickHouse fed by CDC so OLTP stays lean.
- **Hot keys**: rate limits, request coalescing, sharded counters.
- **Auth at scale**: stateless JWT verification; JWKS cached.
- **Operations**: online-only migrations; batched backfills; capacity planning; cost monitoring.
- **Compliance**: data residency, deletion workflows, audit.

---

## D12. Design a file-upload API for multi-GB files.

**Never stream multi-GB through API pods.** Use S3 multipart with pre-signed URLs; the API is the control plane.

1. `POST /uploads` `{filename, size, content_type, checksum}` → validate (limits, type, quota) → `CreateMultipartUpload` → return `upload_id`, part size (e.g., 64 MB), and pre-signed URLs for parts.
2. Client uploads parts **directly to S3** in parallel; retries individual parts; resumable (`GET /uploads/{id}` lists completed parts via `ListParts`).
3. `POST /uploads/{id}/complete` with part ETags → `CompleteMultipartUpload` → verify checksum → record metadata → enqueue post-processing (virus scan, thumbnails).
4. `DELETE /uploads/{id}` → `AbortMultipartUpload`; a lifecycle rule aborts incomplete uploads after 24 h.

**Details:** short-lived pre-signed URLs (15 min) scoped to key and length; S3 Transfer Acceleration for global users; SSE-KMS; bucket denies public access; per-tenant prefixes; completion event notification; quotas and rate limits on URL generation.

---

## D13. Design an API for long-running ML jobs.

**Requirements:** minutes-to-hours GPU jobs; status, progress, cancellation, results; fairness across tenants.

- **API (CPU pods)**: `POST /jobs` (validate, store spec, inputs to S3, enqueue) → 202 + `job_id`; `GET /jobs/{id}` status/progress/logs/result URLs; `DELETE /jobs/{id}` cancel; list with filters; webhooks/SSE.
- **Queue**: SQS/Kafka/Redis Streams with **priority tiers** and per-tenant fair share; or a workflow engine (Temporal/Argo Workflows) for multi-step DAGs.
- **Workers**: GPU node pool; KEDA on queue depth; claim atomically; heartbeat; **checkpoint to S3** every N minutes so spot interruption resumes rather than restarts; logs to S3/Loki; progress in Redis; final state in Postgres.
- **Cancellation**: cooperative flag checked between steps; SIGTERM → checkpoint.
- **Idempotency**: job ID as dedupe key; outputs keyed by job ID.
- **Resource control**: per-tenant concurrency limits, quotas, cost tracking; per-type timeouts.
- **Model management**: versioned registry; job records the model version.
- **Observability**: queue wait vs run time, GPU utilisation, failure taxonomy, stuck-job detector, DLQ.

---

## D14. Design an API with rate limiting.

**Requirements:** per-user and per-API-key limits, plan tiers (free 60 rpm, pro 1,000 rpm), burst tolerance, global protection, clear client feedback.

- **Two layers**: edge (WAF/API Gateway) for coarse IP limits; app for identity-aware business limits.
- **Algorithm**: token bucket in Redis (Lua for atomicity): capacity = burst, refill = rate; keys `rl:{tier}:{principal}:{scope}`; multiple scopes (global per user; per endpoint group for expensive ops).
- **Response**: `429` + `Retry-After`; always send `RateLimit-Limit/Remaining/Reset`.
- **Resilience**: if Redis is down, fail open with an in-memory fallback + alert — except sensitive endpoints (login → fail closed).
- **Fairness & abuse**: per-tenant, per-IP and per-token limits together; anomaly detection; allow-lists for internal callers.
- **Config**: limits from a config service/DB, hot-reloadable; per-tenant overrides.
- **Observability**: 429 rate per tenant, limiter latency, Redis health.

---

## D15. Design an API supporting API versioning.

- **Strategy**: URI major versions (`/v1`, `/v2`); additive within a version; breaking only in a new major.
- **Code**: per-version router packages adapting shared domain services (files 03/09); version-specific schemas.
- **Lifecycle**: `active → deprecated (Sunset header + docs + emails) → retired`; 6–12 months notice; per-version usage metrics drive retirement.
- **Docs**: separate OpenAPI per version; changelog; migration guides.
- **Gateway**: route by prefix; can map `/v1` to an older deployment during transition.
- **Testing**: contract tests per version; consumer-driven contracts for internal clients.
- **Edge cases**: versioned events/webhooks (payload version field); SDKs pinned to versions; feature flags for behaviour changes that aren't contract changes.

---

## D16. Design a zero-downtime FastAPI deployment.

Compose files 14 and 15:
- Rolling update `maxUnavailable: 0`; readiness/startup probes; `preStop` sleep + graceful shutdown; PDB; ≥3 replicas across AZs.
- Expand/contract migrations as a PreSync Job; app compatible with schema N and N+1.
- Backward-compatible API contracts; feature flags for behaviour.
- Canary with automated analysis and rollback.
- LB connection draining; client retries for idempotent requests.
- Long-lived connections (WebSockets/SSE): announce shutdown; clients reconnect with backoff.
- Workers: stop consuming on SIGTERM, finish in-flight, exit; visibility timeouts cover the rest.
- Verify with a load test running *during* a deploy: zero non-2xx.

---

## D17. Design a highly available FastAPI service across multiple AZs.

- **Compute**: EKS node groups in ≥3 AZs; `topologySpreadConstraints` on zone; anti-affinity; PDB; autoscaler per AZ.
- **Ingress**: ALB (multi-AZ) → ingress pods spread across AZs.
- **Data**: RDS/Aurora Multi-AZ (synchronous standby, auto failover); readers in each AZ; ElastiCache Multi-AZ; S3/SQS regional.
- **Zone-aware routing**: topology hints so traffic prefers same-AZ pods and replicas, with cross-AZ fallback.
- **Stateless app**: sessions in Redis/JWT; uploads to S3.
- **Failure handling**: `pool_pre_ping` + reconnect on failover; low DNS TTL; readiness reflects dependencies; retries with jitter.
- **Testing**: AZ evacuation game day (drain a zone, kill the DB primary); verify SLOs.
- **Beyond AZ**: second region with Route 53 failover and cross-region replication.

---

## D18. Design FastAPI services that communicate through Kafka/SQS.

- **Kafka vs SQS**: Kafka for high-throughput ordered streams, replay, multiple consumer groups, event sourcing/CDC; SQS for simple work queues with minimal ops (FIFO for per-group ordering).
- **Producer**: **transactional outbox** — event row written in the same transaction as the state change; a relay (or Debezium) publishes and marks sent. No lost events; at-least-once.
- **Event design**: versioned schemas (Avro/Protobuf/JSON Schema registry), `event_id`, `occurred_at`, trace context, aggregate ID as partition key.
- **Consumer** (separate deployment, `aiokafka`/`aiobotocore`): idempotent handlers (dedupe by `event_id` or unique constraint), commit after processing, backoff + DLQ, consumer-group scaling ≤ partitions, KEDA on lag/depth.
- **Contracts**: consumer-driven; additive evolution; compatibility checks in CI.
- **Observability**: producer/consumer lag, processing latency, DLQ depth, trace links across the async hop.
- **Sagas**: choreography via events with compensations; orchestration (Temporal) when the flow gets complex.

---

## D19. Design a FastAPI service capable of surviving downstream service failures.

**Assume every dependency will fail; design for degraded, not dead.**
- **Timeouts** on every call (connect + read), shorter than your own SLO budget; deadlines propagated.
- **Retries** only for idempotent operations, bounded, exponential backoff with jitter, retry budgets (≤10% extra load).
- **Circuit breaker** per dependency: open after N failures, half-open probes, fail fast while open.
- **Bulkheads**: a semaphore per dependency so one slow service can't consume all concurrency.
- **Fallbacks**: stale cache (`stale-if-error`), defaults, partial responses flagged `degraded: true`, queue the write for later.
- **Feature flags** to disable non-critical integrations instantly.
- **Async decoupling** for anything not needed for the response.
- **Readiness** reflects only *required* dependencies.
- **Load shedding** so cascading slowness doesn't exhaust pods.
- **Observability**: breaker state, fallback counters, dependency SLIs; alert on prolonged degraded mode.
- **Testing**: fault injection (toxiproxy, chaos mesh) in staging.

```python
import time, asyncio, httpx

class Breaker:
    def __init__(self, failures=5, reset_after=30):
        self.failures, self.reset_after = failures, reset_after
        self.count, self.opened_at = 0, None

    def allow(self) -> bool:
        if self.opened_at and time.monotonic() - self.opened_at < self.reset_after:
            return False
        return True

    def record(self, ok: bool):
        if ok:
            self.count, self.opened_at = 0, None
        else:
            self.count += 1
            if self.count >= self.failures:
                self.opened_at = time.monotonic()

async def get_recommendations(user_id: int, http, redis) -> list[dict]:
    if not reco_breaker.allow():
        return await stale_or_default(redis, user_id)
    try:
        async with reco_sem:
            r = await http.get(f"{RECO_URL}/users/{user_id}", timeout=0.3)
        r.raise_for_status()
        reco_breaker.record(True)
        await redis.set(f"reco:{user_id}", r.text, ex=3600)
        return r.json()
    except (httpx.HTTPError, asyncio.TimeoutError):
        reco_breaker.record(False)
        return await stale_or_default(redis, user_id)
```
### 🔍 Line by line
- `Breaker.allow()` — if the breaker opened less than `reset_after` seconds ago, refuse immediately (fail fast). After that, allow one attempt (half-open).
- `record(ok)` — success resets; failures count up; on the Nth failure, open the breaker.
- `stale_or_default` — the fallback: last cached value or an empty list.
- `async with reco_sem` — bulkhead limiting concurrent calls to this dependency.
- `timeout=0.3` — a tight timeout for a non-critical call.
- On success, refresh the stale copy so future fallbacks have recent data.

---

## D20. Design a production-grade FastAPI platform on AWS EKS.

**Accounts & network:** separate AWS accounts per environment; VPC with private subnets for nodes/DB across 3 AZs, public only for ALB/NAT; VPC endpoints for S3/ECR/Secrets Manager/SQS.

**Cluster:** EKS with managed node groups or Karpenter (spot for workers, on-demand for API); Bottlerocket AMIs; IRSA/Pod Identity; Pod Security Standards (restricted); NetworkPolicies; cluster autoscaling; EKS add-ons; `ndots` tuning.

**Edge:** Route 53 → CloudFront + WAF → ALB (AWS Load Balancer Controller, ACM TLS) → NGINX ingress or Gateway API; optional mesh (Istio ambient) for mTLS and retries.

**Platform services:** ArgoCD (ApplicationSets per env), External Secrets Operator ↔ Secrets Manager, cert-manager, Reloader, KEDA, Prometheus stack / AMP, Grafana, Loki/CloudWatch, Tempo/X-Ray via ADOT Collector, Trivy Operator, Kyverno policies (signed images, non-root, limits required), Velero backups.

**Data:** Aurora PostgreSQL Multi-AZ + RDS Proxy; ElastiCache Redis cluster mode; S3 with lifecycle/KMS; SQS/SNS/MSK; PITR + tested restores.

**Golden path:** a cookiecutter service template with the file 09 structure, shared library, Dockerfile, Helm chart (Deployment/HPA/PDB/ServiceAccount/ServiceMonitor/Ingress), CI template (file 15), generated dashboards and alerts, SLO definitions.

**Security:** least-privilege IAM per service; no static keys; rotation; image signing + admission verification; network segmentation; audit logs (CloudTrail, control-plane logs); GuardDuty; continuous scanning.

**Operations:** on-call rotation, runbooks in repo, error budgets, capacity reviews, cost allocation (Kubecost), quarterly game days (AZ loss, DB failover, rollback), DR plan with RTO/RPO and cross-region backups.

### 🎤 Closing line
> "The platform's job is to make the right thing the easy thing: a new FastAPI service should go from template to production — with observability, security and autoscaling included — in a day, without the team becoming Kubernetes experts."

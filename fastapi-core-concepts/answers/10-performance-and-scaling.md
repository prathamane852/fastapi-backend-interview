# 10 — Performance & Scaling

---

## 📖 Vocabulary

> *Definition* is the precise meaning; *Say it in an interview* is the quotable one-liner.

| Term | Definition | Say it in an interview |
|---|---|---|
| **Latency** | Elapsed time to complete one request, measured end-to-end or per component; reported as percentiles. | "Latency is per-request time; I track p50, p95 and p99." |
| **Throughput** | Number of requests completed per unit time (rps). | "Throughput is requests per second at acceptable latency." |
| **Percentile** | The value below which a given percentage of observations fall; p99 latency = 99% of requests were faster. | "p99 is the tail your slowest 1% of users experience." |
| **Benchmark** | A controlled, repeatable measurement of performance under defined conditions. | "Benchmark realistic endpoints, not `/health`." |
| **Load test** | Applying realistic or elevated traffic to observe behavior, find limits and detect regressions (k6, Locust, wrk2). | "Load tests tell me capacity and headroom before production does." |
| **Coordinated omission** | A measurement error where a slow server causes a closed-loop load generator to send fewer requests, hiding true latency. | "Use constant-rate tools so slow responses don't hide themselves." |
| **Profiling** | Measuring where a program spends time or memory at the function level (py-spy, cProfile, memray). | "Profiling shows the hot functions to optimize." |
| **Flame graph** | A visualization of stack samples: width = time share; used to spot hot paths. | "Wide bars in a flame graph are where the time goes." |
| **Worker process** | An OS process running an application instance; multiple workers provide CPU parallelism and fault isolation. | "Workers are processes; one per core is the rule of thumb." |
| **Vertical vs horizontal scaling** | Vertical: bigger machine. Horizontal: more instances behind a load balancer; requires stateless design. | "Stateless services scale horizontally, which is the cloud-native default." |
| **HPA** | *Horizontal Pod Autoscaler*: Kubernetes controller adjusting replica count based on CPU/memory/custom metrics against targets. | "HPA scales pods on load; I prefer request-rate metrics over memory." |
| **KEDA** | Kubernetes Event-Driven Autoscaling: scales workloads on external metrics such as queue depth or Prometheus queries, including to zero. | "KEDA scales workers on queue depth." |
| **Cache** | A store of previously computed or fetched data for faster subsequent access. | "A cache trades freshness for speed." |
| **Cache-aside** | Pattern: application checks the cache, on miss loads from source and writes the result into the cache with a TTL. | "Cache-aside: read cache, miss → load → populate." |
| **TTL (Time To Live)** | The duration after which a cached entry expires. | "TTL bounds staleness." |
| **Cache invalidation** | Removing or updating cached entries when the underlying data changes. | "Invalidate on write; that's the hard part of caching." |
| **Hit ratio** | Cache hits divided by total lookups; the key cache-effectiveness metric. | "A high hit ratio is what protects the database." |
| **Cache stampede (thundering herd)** | Many concurrent requests missing the same key simultaneously (e.g., after expiry) and overwhelming the backend. | "Stampedes are prevented with locks, jittered TTLs or stale-while-revalidate." |
| **Stale-while-revalidate** | Serving an expired cached value immediately while refreshing it asynchronously. | "Serve stale, refresh in the background." |
| **Distributed lock** | A lock coordinated through a shared store (Redis `SET NX PX`) so only one process performs an action at a time. | "A Redis lock makes one process recompute while others wait." |
| **Rate limiting** | Restricting the number of requests a client may make in a time window to protect capacity and ensure fairness. | "Rate limits protect the system and enforce fair share." |
| **Token bucket** | Algorithm where tokens accrue at a fixed rate up to a capacity; each request consumes one; allows bursts up to capacity. | "Token bucket permits bursts while enforcing an average rate." |
| **Sliding window** | Algorithm counting requests in the trailing window precisely (log) or approximately (weighted counters), avoiding fixed-window edge bursts. | "Sliding window is precise and avoids boundary bursts." |
| **Lua script (Redis)** | Server-side script executed atomically by Redis, combining multiple commands into one indivisible operation. | "Lua makes the check-and-increment atomic." |
| **Load shedding** | Deliberately rejecting excess requests (503/429) when saturated so accepted requests stay within latency targets. | "Shed load early rather than queue everyone into timeouts." |
| **Backpressure** | A mechanism by which a consumer signals a producer to slow down, preventing unbounded buffering. | "Backpressure keeps queues bounded." |
| **CFS throttling** | Linux Completely Fair Scheduler enforcing container CPU limits by pausing the process when its quota per period is exhausted; manifests as latency spikes. | "CPU limits cause throttling that looks like random latency." |
| **Streaming response** | A response whose body is produced and sent incrementally, allowing constant memory use and early first byte. | "Streaming keeps memory flat and starts the response immediately." |
| **`orjson`** | A fast JSON serializer/deserializer implemented in Rust with native support for datetime, UUID and dataclasses. | "orjson is several times faster than the standard library." |
| **Compression (gzip/brotli)** | Encoding response bodies to reduce bytes on the wire at CPU cost; negotiated via `Accept-Encoding`. | "Compress large JSON at the ingress or with GZipMiddleware." |
| **`sendfile` / zero-copy** | Kernel facility transferring file data directly to a socket without copying through user space; used by `FileResponse`. | "FileResponse uses sendfile to serve files efficiently." |
| **Pre-signed URL** | A time-limited URL granting direct access to an object-storage operation (S3 GET/PUT) without proxying through the API. | "Pre-signed URLs take big files off my API's data path." |
| **`--max-requests`** | Gunicorn/Uvicorn option restarting a worker after N requests to bound memory growth from leaks or fragmentation. | "Worker recycling is a safety net for slow leaks." |

---

## Q1. Why is FastAPI considered fast?

Covered in file 01 Q3: ASGI event loop (no thread per request), uvloop + httptools in C, Pydantic v2 in Rust, thin Starlette routing. For I/O-bound work it benchmarks near Node/Go. It's still Python for CPU-heavy code.

## Q2. How would you benchmark a FastAPI application?

1. **Define the goal**: target p99 latency at target rps with a realistic endpoint mix — not "max rps of `/health`".
2. **Environment**: production-like (same CPU limits, DB, network), isolated from noise.
3. **Tool**: `k6` (scriptable, thresholds, ramp profiles), `wrk2` (constant rate), `locust` (Python user journeys).
4. **Profile**: warm-up → ramp → steady state ≥ 5 min → spike. Record p50/p95/p99, error rate, throughput, server CPU/memory/DB pool.
5. **Compare** to a baseline; automate with thresholds in CI.
6. **Profile bottlenecks**: `py-spy record` flame graphs; `EXPLAIN ANALYZE`.

```js
// k6 script
import http from 'k6/http';
export const options = {
  stages: [{ duration: '1m', target: 100 }, { duration: '5m', target: 100 }, { duration: '1m', target: 0 }],
  thresholds: { http_req_duration: ['p(99)<300'], http_req_failed: ['rate<0.01'] },
};
export default function () {
  http.get('https://staging.api/orders?limit=20', { headers: { Authorization: `Bearer ${__ENV.TOKEN}` } });
}
```
- `stages` — ramp to 100 virtual users, hold 5 min, ramp down.
- `thresholds` — fail the run if p99 ≥ 300 ms or error rate ≥ 1%.

## Q3. What metrics would you monitor?

- **RED** (service): Rate, Errors, Duration per route.
- **USE** (resources): Utilisation, Saturation, Errors of CPU, memory, event-loop lag, thread pool queue, DB pool, Redis.
- **Business**: orders/min, signups, payment failures.
- **Dependencies**: latency and error rate per downstream.
- **Runtime**: GC pauses, open file descriptors, restarts.

## Q4. How do you reduce API latency?

Ordered by typical payoff:
1. Fix blocking calls in async handlers.
2. Fix N+1 and add indexes (DB is usually 60–80% of latency).
3. Pool everything (DB, Redis, HTTP clients).
4. Cache hot reads.
5. Parallelise independent I/O with `gather`.
6. Return less: pagination, field selection.
7. `orjson` + compression for large responses.
8. Move slow work off the request path.
9. Keep service and DB in the same AZ; keep-alive.
10. Right-size workers; avoid CPU throttling in Kubernetes.

## Q5. How do you identify slow endpoints?

Prometheus latency histogram by route → Grafana table sorted by p99; tracing to see which span inside is slow; APM endpoint breakdown; access-log percentiles; `py-spy` for the live view; per-endpoint SLO alerts.

---

## Q6. How would you handle 10,000 requests per second?

Back-of-envelope: an async FastAPI worker does ~1–3k rps of light I/O endpoints per core → ~10–20 workers across 3–5 pods, *if backends keep up*.
1. **Stateless app** → horizontal scaling with HPA.
2. **Load balancer** with keep-alive to pods.
3. **Cache aggressively**: CDN for public GETs, Redis for hot data; aim for a high hit ratio so the DB sees a fraction.
4. **DB**: read replicas, PgBouncer, verified indexes, short transactions.
5. **Async everywhere**, bounded concurrency to downstreams.
6. **Rate limiting + load shedding**: fail fast when saturated instead of queuing.
7. **Writes off the path** via queues where semantics allow.
8. **Load test to 2× target** to know headroom.

## Q7. How does Uvicorn handle concurrency?

Within a worker: one event loop handles many connections via non-blocking sockets; thousands of in-flight requests share one thread as long as they `await`. Sync endpoints go to a thread pool (default 40). Across workers: `--workers N` forks N processes sharing the listening socket. No shared memory between workers — state lives in Redis/DB.

## Q8. What are Uvicorn workers?

Separate OS processes running the same app. They give **CPU parallelism** (bypassing the GIL) and **fault isolation** (one crash doesn't kill the others).

## Q9. When should you increase workers?

- Per-worker CPU is high (> 70%) while latency rises → add workers up to core count.
- Many sync endpoints saturate the thread pool.
- In Kubernetes prefer **more pods with 1–2 workers each**: finer scaling, better bin-packing, faster restarts, clearer memory limits. Rule: `workers ≈ CPU limit`, never more than cores.

## Q10. Worker processes vs threads?

| | Processes | Threads |
|---|---|---|
| GIL | Each has its own → true parallelism | Shared → one Python instruction at a time |
| Memory | Separate (N× app memory) | Shared |
| Crash isolation | ✅ | ❌ |
| Use for | CPU-bound work, scaling Python | Blocking I/O in sync libraries |

In FastAPI: **processes** = Uvicorn workers; **threads** = the pool running `def` endpoints; **coroutines** beat both for async I/O.

---

## Q11. How does Kubernetes scale FastAPI?

A `Deployment` runs N identical pods behind a `Service`. The **HPA** adjusts N based on CPU/memory/custom metrics. The **Cluster Autoscaler / Karpenter** adds nodes when pods can't be scheduled. Readiness probes ensure new pods only get traffic once ready.

## Q12. How would you configure HPA?

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata: { name: orders-api }
spec:
  scaleTargetRef: { apiVersion: apps/v1, kind: Deployment, name: orders-api }
  minReplicas: 3
  maxReplicas: 30
  metrics:
    - type: Resource
      resource: { name: cpu, target: { type: Utilization, averageUtilization: 60 } }
    - type: Pods
      pods:
        metric: { name: http_requests_per_second }
        target: { type: AverageValue, averageValue: "500" }
  behavior:
    scaleUp:   { stabilizationWindowSeconds: 0,   policies: [{ type: Percent, value: 100, periodSeconds: 30 }] }
    scaleDown: { stabilizationWindowSeconds: 300, policies: [{ type: Pods, value: 2, periodSeconds: 60 }] }
```
- `minReplicas: 3` — for AZ spread and rolling updates.
- CPU target 60% — headroom for the 1–2 minute scale-up lag.
- Custom `http_requests_per_second` (via Prometheus Adapter / KEDA) — scale on actual load.
- `scaleUp` fast (double every 30 s); `scaleDown` slow (avoid flapping).
- HPA needs `resources.requests.cpu` set; utilisation is relative to requests.

## Q13. CPU-based vs memory-based autoscaling?

- **CPU**: a good proxy for request load in async apps; responsive.
- **Memory**: a poor signal — Python memory doesn't shrink when load drops, so memory-based HPA scales up and never down. Use memory *limits* for safety, not scaling.
- **Best**: request rate / latency / queue depth (KEDA) — what users feel.

## Q14. What is connection pooling?

(Also file 08.) Reusing open connections rather than opening/closing per request — for DB (SQLAlchemy pool, PgBouncer), Redis (client pool), HTTP (`httpx.AsyncClient` keep-alive). Removes handshake latency and prevents connection storms.

---

## Q15. How would Redis improve API performance?

> 🔴 **Moved to the advanced section** → [../../advanced-backend/04-redis-and-caching.md](../../advanced-backend/04-redis-and-caching.md)
> 
> Sections A–B — data structures, persistence, eviction and caching patterns.

---

## Q16. When should you introduce caching?

When you have **evidence** (profiling shows a hot, expensive, read-heavy path) and the data tolerates staleness. Caching adds invalidation complexity and can mask bugs. Order: fix the query → add index → then cache. Good candidates: reference data, catalogs, computed aggregates, permission lookups, external API responses.

## Q17. What data should not be cached?

- Strongly consistent data (balances, inventory at checkout, revocation state).
- Per-user private data in a shared cache without per-user keys.
- Secrets/tokens/unnecessary PII.
- Rapidly changing data with near-zero hit ratio.
- Huge payloads that evict everything else.
- Error responses (unless deliberate, short TTL, to protect a failing backend).

## Q18. How do you prevent cache stampede?

> 🔴 **Moved to the advanced section** → [../../advanced-backend/04-redis-and-caching.md](../../advanced-backend/04-redis-and-caching.md)
> 
> Q7 — three stampede defences including probabilistic early expiry.

---

## Q19. What is rate limiting?

Restricting how many requests a client may make per window. Protects against abuse, brute force, runaway clients and cost blow-ups; enforces fair share. Algorithms: fixed window (simple, bursty at edges), sliding window (smooth, precise), **token bucket** (bursts up to capacity, steady refill — most common), leaky bucket.

## Q20. How would you implement rate limiting?

> 🔴 **Moved to the advanced section** → [../../advanced-backend/04-redis-and-caching.md](../../advanced-backend/04-redis-and-caching.md)
> 
> Q10 — the token-bucket Lua script and the algorithm comparison.

---

## Q21. How would you handle sudden traffic spikes?

- Aggressive **autoscale** with headroom (min replicas above average).
- **Queue writes** so the API acks quickly and workers drain.
- **Load shedding**: reject early with 503 + `Retry-After` when saturated (`--limit-concurrency` in Uvicorn).
- **Caching/CDN** absorbs read spikes.
- **Per-client rate limits** stop one tenant taking everything.
- **Circuit breakers and timeouts** prevent pile-ups.
- **Pre-warm** for known events.
- **Graceful degradation** via flags.

---

## Q22. How would you optimize JSON serialization?

- `FastAPI(default_response_class=ORJSONResponse)` — `orjson` is 3–10× faster and handles datetime/UUID natively.
- Return Pydantic models and let `response_model` serialise in Rust; avoid `jsonable_encoder` on huge structures.
- Trim payloads (`response_model_exclude_none`, field selection).
- Stream NDJSON for very large lists.
- `GZipMiddleware(minimum_size=1000)` or let the ingress do brotli.
- Avoid double serialisation; `model_dump_json()` goes straight to bytes.

## Q23. How would you handle large file downloads?

Don't push bytes through Python if avoidable: store in S3 and return a **pre-signed URL** (or 302), or serve via CDN. If you must: `FileResponse` (uses `sendfile`, supports `Range`) for local files; `StreamingResponse` for remote sources; set `Content-Length`/`Content-Disposition`; support ranges for resumable downloads; NGINX `X-Accel-Redirect` for zero-copy.

## Q24. How would you stream large responses?

```python
@router.get("/reports/{id}/csv")
async def report_csv(id: int, db: DB):
    async def gen():
        yield "id,total,created\n"
        stream = await db.stream(
            select(Order.id, Order.total, Order.created_at)
            .where(Order.report_id == id)
            .execution_options(yield_per=500)
        )
        async for row in stream:
            yield f"{row.id},{row.total},{row.created_at.isoformat()}\n"
    return StreamingResponse(gen(), media_type="text/csv",
                             headers={"Content-Disposition": f'attachment; filename="report-{id}.csv"'})
```
- `gen()` is an async generator; each `yield` sends a chunk. Memory is constant regardless of row count.
- Avoid `BaseHTTPMiddleware` that buffers bodies; set `proxy_buffering off` at NGINX for real-time streaming.

## Q25. How would you reduce memory consumption?

- Fewer workers per pod; scale with pods.
- Stream instead of buffering (`yield_per`, chunked uploads).
- No unbounded in-process caches (`lru_cache(maxsize=...)`, Redis with TTL).
- Watch for leaks: growing globals, unclosed sessions/clients, never-awaited tasks.
- `MALLOC_ARENA_MAX=2` (glibc fragmentation); consider jemalloc.
- `--max-requests` worker recycling as a safety net.
- Profile with `tracemalloc`, `memray`, `objgraph`.

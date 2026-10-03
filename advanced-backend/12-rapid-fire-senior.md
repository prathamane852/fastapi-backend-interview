# 12 — Rapid Fire: 100 Senior Questions

> **How to use this file.** Cover the right column. Say your answer **out loud** — not in your head, out loud. Then uncover and compare.
>
> Every answer here is written as a spoken sentence, so you can use it verbatim. If one needs more than two sentences, follow the link and re-read that section.

---

## Python (1–15)

| # | They ask | 🗣️ You say |
|---|---|---|
| 1 | What does the GIL actually protect? | "Interpreter state — mainly making reference-count updates safe without locking every object. It only constrains multi-core Python CPU work; I/O is unaffected." |
| 2 | Free-threading status? | "Experimental in 3.13, officially supported in 3.14 — but still not the default build, and C extensions need auditing." |
| 3 | The two memory mechanisms? | "Reference counting does the real work and frees things instantly. The garbage collector is a backup that only hunts reference cycles." |
| 4 | Why `gc.freeze()` before forking? | "It stops the collector touching refcount fields in shared pages, which would force the kernel to copy them into every worker." |
| 5 | Does `await` always yield to the loop? | "No, and that's the thing people get wrong. It only suspends if it hits something not ready. `await` on CPU work blocks the loop completely." |
| 6 | `gather` or `TaskGroup`? | "`TaskGroup` by default. If one task fails it cancels the siblings, so nothing outlives the block — with `gather` the others just keep running." |
| 7 | The `create_task` footgun? | "The loop only keeps a weak reference, so a fire-and-forget task can be garbage collected mid-flight. Keep a strong reference yourself." |
| 8 | Blast radius of a blocking call in `async def`? | "The whole worker process. Median latency looks fine, p99 is terrible, and pods get restarted because health checks time out." |
| 9 | Best async-specific metric? | "Event-loop lag — how late the loop is running what it promised. Almost nobody exports it and it catches the whole blocking-call bug class." |
| 10 | Thread-locals in async? | "Wrong — many requests share one thread, so you read someone else's value. Use `contextvars`." |
| 11 | Why is `lru_cache` on a method a leak? | "It keys on `self`, so the cache holds a strong reference to every instance forever." |
| 12 | Profiling in production? | "`py-spy` — it samples a running process with no restart and no code change. `cProfile` distorts async badly." |
| 13 | Biggest Python performance win? | "Not doing the work — cache it, batch it, defer it. Then fix the query. If a request spends 40ms in the database and 3ms in Python, Python tuning has a 3ms ceiling." |
| 14 | `__slots__` — when? | "When you're holding a lot of small objects. It's roughly a seven-fold memory saving per instance, and noise for request-scoped objects." |
| 15 | `__init_subclass__` or a metaclass? | "`__init_subclass__` first. It covers registration and validation — what people actually want — without metaclass conflicts." |

## Databases (16–30)

| # | They ask | 🗣️ You say |
|---|---|---|
| 16 | Postgres default isolation? | "Read Committed. And the catch is it's a fresh snapshot per *statement*, so two SELECTs in one transaction can genuinely disagree." |
| 17 | What does Repeatable Read not prevent? | "Write skew — two transactions reading the same data, each making a decision that's fine alone, together breaking a rule. That's why Serializable exists." |
| 18 | Which errors are safe to retry? | "40001 serialization failure and 40P01 deadlock. Both with backoff, and the transaction has to be free of side effects." |
| 19 | What does Serializable cost? | "Not blocking — Postgres detects conflicts and aborts one. So the cost is retries, and your code has to handle them." |
| 20 | Where's the transaction boundary? | "Service layer, one per request. Repositories never commit — otherwise you lose atomicity across a use case." |
| 21 | Worst transaction anti-pattern? | "An external HTTP call inside an open transaction. You pin a connection and hold locks for seconds, and under load that drains the pool." |
| 22 | Optimistic or pessimistic? | "Optimistic for user-facing edits — you can't hold a row lock while someone fills in a form. Pessimistic when contention is high and the critical section is microseconds." |
| 23 | Build a job queue in Postgres? | "`SELECT FOR UPDATE SKIP LOCKED` with a limit. Each worker takes different rows, with no lock service and no TTL to guess." |
| 24 | Deadlock root cause? | "Two code paths taking the same locks in opposite order. Fix the ordering — usually sort by primary key. Retrying without fixing it just converts a crash into latency." |
| 25 | Catch N+1 before production? | "Set relationships to `lazy='raise'` so an accidental lazy load is a loud error, and assert a query budget in tests." |
| 26 | `selectinload` or `joinedload`? | "`selectinload` for collections — two queries, each row once. `joinedload` for many-to-one. Joining two collections gives you a cartesian explosion." |
| 27 | Why is deep `OFFSET` slow? | "It produces and throws away every row you skipped. At offset 500,000 it reads half a million rows to return twenty." |
| 28 | Keyset pagination requirement? | "A composite index matching the ORDER BY exactly, with the primary key as a tiebreak so the ordering is total." |
| 29 | Read replicas — what breaks? | "Read-your-writes. You create something, immediately read it from a lagging replica, and get a 404 for a thing you just made." |
| 30 | CAP vs PACELC? | "CAP only says anything *during* a partition. PACELC adds the latency-versus-consistency trade you actually make every ordinary day." |

## PostgreSQL (31–45)

| # | They ask | 🗣️ You say |
|---|---|---|
| 31 | Why does an UPDATE create garbage? | "Postgres never edits in place. It writes a new row version and marks the old one dead, and nothing frees it until VACUUM runs." |
| 32 | Why does one long query cause bloat everywhere? | "It holds back the cutoff for the whole database, so dead rows across every table become unremovable while it runs." |
| 33 | First query on a disk/bloat complaint? | "`pg_stat_activity` for the oldest transaction. It's usually one `idle in transaction` connection an ORM forgot to close." |
| 34 | What's wrong with default autovacuum? | "The 20% scale factor. On a 100-million-row table that's 20 million dead rows before it even starts. I set 0.01 on hot tables." |
| 35 | XID wraparound? | "Transaction IDs are 32-bit and wrap. If freezing falls behind, Postgres refuses transactions and shuts down to protect your data. Alert at 500 million." |
| 36 | HOT update? | "An update that touches no indexed column and fits on the same page, so no index is written at all. You enable it with a fillfactor below 100." |
| 37 | When is BRIN the right index? | "A huge, physically time-ordered, append-only table. Megabytes instead of gigabytes — but useless if the rows aren't in key order." |
| 38 | Composite index column order? | "Equality columns first, then the range or sort column. It's a phone book sorted by surname then first name — you can't efficiently find all the Johns." |
| 39 | First thing in `EXPLAIN ANALYZE`? | "Estimated versus actual rows. If a node estimated 12 and got 480,000, the statistics are lying and everything above is built on that." |
| 40 | `work_mem` scope? | "Per sort or hash node, per connection — not per query. That's how people OOM a database by 'just raising work_mem'." |
| 41 | Sort `pg_stat_statements` by what? | "Total time, not mean. A 5ms query run two million times is far more of your load than a 3-second query run twice." |
| 42 | Why PgBouncer? | "Postgres forks a process per connection, so they're expensive. Twenty pods with a pool of twenty is 400 backends. PgBouncer multiplexes them down to a few dozen." |
| 43 | The asyncpg + PgBouncer bug? | "Server-side prepared statements don't survive transaction pooling. You get intermittent 'prepared statement does not exist' — set `statement_cache_size=0`." |
| 44 | Which DDL rewrites the table? | "`ALTER COLUMN TYPE`. Adding a column with a default is safe since PG11, and adding a nullable column is instant metadata." |
| 45 | Why does a quick ALTER cause an outage? | "It queues behind a slow SELECT for an exclusive lock, and Postgres locks are FIFO — so every subsequent query queues behind the ALTER. Set `lock_timeout`." |

## Redis & caching (46–57)

| # | They ask | 🗣️ You say |
|---|---|---|
| 46 | Why is one O(n) Redis command dangerous? | "Commands run on a single thread, so `KEYS *` freezes every client at once. It's a cliff, not gradual degradation." |
| 47 | Most underused Redis structure? | "The sorted set — sliding windows, delayed queues, leaderboards. And HyperLogLog, which counts billions of uniques in twelve kilobytes." |
| 48 | Eviction policy for cache vs locks? | "`allkeys-lru` for a cache, `noeviction` for locks. Sharing one instance with LRU means Redis will happily evict your locks." |
| 49 | Update or delete the cache key on write? | "Delete. It's idempotent and safe under concurrency — updating races with another writer and can leave the cache permanently wrong." |
| 50 | Three stampede defences? | "Jittered TTL always, because mass simultaneous expiry is what actually kills databases. A lock for individually expensive keys. And probabilistic early expiry when even one miss is unacceptable." |
| 51 | Redis is down — what happens? | "It degrades to a cache miss, with a 50–100ms timeout and a circuit breaker. A hanging Redis is worse than a down one." |
| 52 | Distributed lock essentials? | "`SET NX PX` as one atomic command, a unique token as the value, and release via Lua so the check-and-delete is atomic." |
| 53 | Why is that lock still unsafe? | "Process pauses. The TTL expires while you're still working, someone else takes the lock legitimately, and now there are two holders. No timeout fixes that." |
| 54 | Why must a rate limiter be Lua? | "Read-modify-write has to be atomic. From the application, two requests both read 99 tokens and both proceed." |
| 55 | Token bucket or sliding log? | "Token bucket — O(1) memory and it permits a natural burst. A sliding log stores every request timestamp, which is huge for a heavy user." |
| 56 | The Pub/Sub trap? | "Zero persistence. A subscriber that's restarting loses the message entirely, with no error anywhere." |
| 57 | Redis Cluster impact on code? | "Multi-key operations need hash tags so the keys land together. And a hot key is a hot node — sharding doesn't help at all." |

## Queues & messaging (58–70)

| # | They ask | 🗣️ You say |
|---|---|---|
| 58 | Is exactly-once real? | "Not for delivery — the acknowledgement can always be lost. It's achievable in *effect*: at-least-once delivery plus an idempotent consumer." |
| 59 | The idempotency rule people break? | "The dedupe marker and the actual work must be in the *same* transaction. Commit the marker separately and crash, and the redelivery skips work that never happened." |
| 60 | The dual-write problem? | "You commit to the database, then publish — and the process dies in between. The data exists and nobody downstream ever hears about it. You can't fix it with try/except." |
| 61 | The outbox pattern? | "Insert the event into a table in the same transaction as the data. One commit, both or neither. Then a relay publishes from that table." |
| 62 | Number one SQS bug? | "Processing takes longer than the visibility timeout, the message reappears while you're still working on it, and two consumers run the same job. It gets worse under load." |
| 63 | What does FIFO `MessageGroupId` do? | "It's both the ordering unit and the parallelism unit. Set it to the entity whose order matters. Set it to a constant and you get zero concurrency." |
| 64 | DLQ mistakes? | "Retention shorter than 14 days, so evidence vanishes over a weekend — and no alert on depth above zero." |
| 65 | Poison vs transient failure? | "Separate them explicitly. A malformed payload will never succeed, so don't burn five retries on it." |
| 66 | Why SNS to SQS rather than SNS direct? | "Each consumer gets its own buffering, retries and DLQ, and can be down for an hour without losing anything." |
| 67 | SNS filter policy gotcha? | "It filters on message *attributes*, not the body. Set one on a body field and it silently matches nothing." |
| 68 | The backoff formula? | "Full jitter — a uniform random between zero and the capped exponential. Without it, a thousand clients all retry in the same instant." |
| 69 | Kafka parallelism ceiling? | "The partition count. With six partitions a seventh consumer just sits idle." |
| 70 | Best queue metric? | "Age of the oldest message. Better than depth, because depth doesn't tell you whether you're falling behind." |

## AWS (71–80)

| # | They ask | 🗣️ You say |
|---|---|---|
| 71 | How does a pod get AWS credentials? | "IRSA — a projected token, exchanged via `AssumeRoleWithWebIdentity` for temporary credentials the SDK refreshes. Nothing is ever stored." |
| 72 | Number one IRSA misconfiguration? | "A wildcard on the `sub` condition in the trust policy, which lets *any* service account in the cluster assume the role." |
| 73 | Free cost win in a VPC? | "S3 and DynamoDB gateway endpoints. They cost nothing and remove NAT per-gigabyte charges on that traffic." |
| 74 | Security group vs NACL? | "Security groups are stateful, so the reply gets out automatically. NACLs are stateless — you must allow the return ephemeral ports or traffic is silently dropped." |
| 75 | Why does Lambda fit an API badly? | "No persistent process, so every concurrent execution opens its own database connection. Five hundred concurrent executions is five hundred connections." |
| 76 | ALB 502 cause? | "Usually a keep-alive race — the server closing a connection the ALB is reusing. Server keep-alive must be *longer* than the ALB idle timeout." |
| 77 | ALB 460? | "The client gave up and disconnected before your target responded." |
| 78 | Presigned PUT limitation? | "It can't enforce a maximum size. If you need that, use a presigned POST with policy conditions." |
| 79 | S3 consistency today? | "Strong read-after-write for all operations since December 2020. Saying 'eventually consistent' dates you." |
| 80 | RDS major version upgrade? | "Blue/Green Deployments — it builds a synced green environment and you switch in about a minute, with safeguards." |

## Deployment (81–90)

| # | They ask | 🗣️ You say |
|---|---|---|
| 81 | Why does a rolling deploy drop requests? | "SIGTERM and endpoint removal happen in parallel, not in sequence. So the pod is shutting down while traffic is still being routed to it." |
| 82 | The three-part fix? | "A `preStop` sleep to absorb the propagation delay, `maxUnavailable: 0`, and graceful drain in the app — fail readiness first, then close pools." |
| 83 | Why `preStop` and not just graceful shutdown? | "Because preStop runs *before* SIGTERM, so the container keeps serving completely normally while the load balancer catches up." |
| 84 | Worst probe mistake? | "Checking dependencies in the liveness probe. A database blip then restarts every pod simultaneously and you've turned a degradation into an outage." |
| 85 | Which strategy protects you from a bad migration? | "None of them. Blue, green and canary all share one database. Only expand-contract does." |
| 86 | Rename a column with zero downtime? | "Four releases — expand, dual-write, backfill and switch reads, then contract. And the backfill must be batched with a sleep." |
| 87 | Where do migrations run? | "A pipeline stage or Job before the rollout. Never the app entrypoint, because N pods all race." |
| 88 | Can you roll back a migration? | "Mostly no. If it dropped a column the data is gone, and the downgrade recreates an empty one — which is worse, because it looks like it worked." |
| 89 | Exit code 137? | "A kill signal. Either OOMKilled, or it ignored SIGTERM past the grace period." |
| 90 | Why might you omit CPU limits? | "Throttling. The CFS quota bites at the 100ms boundary, so you get mysterious p99 spikes with CPU well below the limit." |

## OpenTelemetry (91–100)

| # | They ask | 🗣️ You say |
|---|---|---|
| 91 | Why a Collector instead of exporting direct? | "Sampling, redaction and routing become a config change rather than redeploying every service." |
| 92 | Why both an agent and a gateway? | "The agent is a local hop that never blocks and adds Kubernetes metadata. The gateway is where you do anything needing a complete trace." |
| 93 | Tail sampling prerequisite? | "Every span of a trace has to reach the same gateway, so you need a load-balancing exporter routing by trace ID." |
| 94 | What if `decision_wait` is too short? | "Late spans get dropped — so your slowest traces, the ones you care about most, come back incomplete." |
| 95 | Head or tail sampling? | "Head is cheap but decides blind, so at 5% you lose 95% of your errors. Tail keeps 100% of errors and slow traces." |
| 96 | Cardinality killer in FastAPI? | "Labelling by the raw URL path instead of the route template. That's unbounded — every ID creates a new time series." |
| 97 | Where do distributed traces break? | "Across queues. You have to inject `traceparent` into message attributes and extract it on the consumer side." |
| 98 | Span link or parent? | "A link for async fan-out, so five hundred jobs don't collapse into one unusable trace." |
| 99 | Most useful correlation? | "Exemplars — clicking from a p99 histogram bucket straight into a real slow trace." |
| 100 | Why OTel rather than a vendor agent? | "The next migration becomes a Collector config change instead of touching every service. That's exactly how our Dynatrace move worked." |

---

## 🎤 The five sentences to have loaded

1. *"The trade-off is that I'm buying ___ and paying ___."*
2. *"The first thing that breaks at 10× is ___."*
3. *"I'd design as if every message arrives at least twice, because it will."*
4. *"Most outages are caused by a deploy — so deployment safety buys more availability than another region."*
5. *"I knew because [specific metric] went from [number] to [number]."*

## ✅ Final checklist

- [ ] **One story of your own per topic** — not a fact, an incident with a number in it.
- [ ] Your OTel open-source→Dynatrace migration rehearsed — it's your strongest differentiator.
- [ ] The four-move pattern script from [11](11-design-patterns-in-interviews.md) said out loud, twice.
- [ ] LLD: clarify for five minutes before writing any code.
- [ ] System design: do the arithmetic out loud, and say "this doesn't need Kafka" when it doesn't.
- [ ] Have a weakness ready for your own design. Naming it first is the strongest move available.

---

← Back to [README.md](README.md)

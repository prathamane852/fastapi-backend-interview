# 09 — System Design, Senior Level

> **How to use this file.** The framework is the most important part — read it twice. Each design problem has a **🗣️ Say this** block for the key insight, which is the thing that actually earns the points.

**What's being graded is not your architecture.** It's whether you drive the conversation, make assumptions explicit, name trade-offs before being asked, and know what will break. A candidate who designs a mediocre system while narrating their reasoning beats one who silently draws a perfect diagram.

---

## 📖 Jargon, in plain words

| Jargon | What it actually means | How you'd say it |
|---|---|---|
| **QPS** | Queries per second. How much traffic you're handling. | "A hundred million daily users doing ten actions each is about 12K QPS average." |
| **Read:write ratio** | For every write, how many reads? | "Ask this first — it determines the whole design." |
| **Horizontal scaling** | More machines. | "Stateless services scale horizontally; databases mostly don't." |
| **Vertical scaling** | A bigger machine. | "It's the cheapest thing to try first and everyone skips it." |
| **Stateless** | The server keeps nothing between requests, so any server can serve any request. | "Statelessness is what makes horizontal scaling possible." |
| **Fan-out** | One event causing many writes or many messages. | "Fan-out on write is great until a celebrity posts." |
| **Blast radius** | How much breaks when this breaks. | "Canary limits blast radius to a percentage of users." |
| **SPOF** | Single point of failure. | "Find yours before the interviewer does." |
| **Denormalise** | Store data twice so reads don't have to join. | "You trade write complexity and storage for read speed." |
| **Strangler fig** | Migrate by routing slices of traffic to the new system gradually. | "Never big-bang a migration." |
| **Load shedding** | Rejecting requests fast when overloaded, on purpose. | "A request the client already timed out on is work you're doing for nobody." |

---

## 🧭 The framework — 45 minutes

| Phase | Time | What you do | What you say |
|---|---|---|---|
| **1. Scope** | 5 min | List features, then explicitly cut | "Let me list what this does, then pick the two or three that are architecturally interesting." |
| **2. Non-functional** | 5 min | Scale, latency, consistency, availability | "What's the read/write ratio? Is a few seconds of staleness OK?" |
| **3. Estimate** | 5 min | QPS, storage, bandwidth — round hard | "100M users × 10 actions ÷ 100K seconds ≈ 10K QPS, call it 40K peak." |
| **4. API + data model** | 5 min | The contract, then the schema | "Here's the write path and the table behind it." |
| **5. High level** | 10 min | Boxes and arrows, happy path only | "Let me walk one request through it." |
| **6. Deep dive** | 10 min | They pick, or you offer | "The hardest part here is the fan-out — shall I go deeper?" |
| **7. Failure & scale** | 5 min | What breaks first, at what multiple | "At 10× the bottleneck moves from the database to the workers." |

📐 **Numbers to memorise — round numbers beat accurate ones**

| Thing | Number |
|---|---|
| Seconds in a day | 86,400 → **round to 100K** |
| Redis GET, same AZ | ~0.5–1 ms |
| Postgres indexed read | ~1–5 ms |
| Same-datacenter round trip | ~0.5 ms |
| Cross-region (EU↔US) | ~80–150 ms |
| S3 GET first byte | ~20–100 ms |
| One commodity server | ~10–50K simple QPS |
| One Postgres primary | ~10–20K simple reads/s, far fewer writes |
| 1 KB × 1M rows/day | ~1 GB/day → ~365 GB/year |

### 🎤 Six sentences that make you sound senior — use them verbatim
1. *"Before I design anything — what's the read-to-write ratio?"*
2. *"I'm going to assume X. Tell me if that's wrong, because it changes the design."*
3. *"The simplest thing that works here is a single Postgres. Let me say what would force me off it."*
4. *"This part is eventually consistent. The user-visible consequence is ___, and I think that's acceptable because ___."*
5. *"That's the trade-off — I'm buying ___ and paying ___."*
6. *"The first thing that breaks at 10× is ___."*

⚠️ **The four ways people fail this round:** designing before scoping; drawing microservices for a problem that needs one service; naming technologies without justifying them ("we'll use Kafka" — *why?*); and going silent while thinking. **Narrate. Silence reads as being stuck.**

---

## D1. Design a rate limiter as a service.

**Scope.** Limit by API key. Multiple tiers. Used by ~50 internal services. Must not become a single point of failure.
**Estimate.** 100K QPS across all services — so the limiter is called 100K times a second.

### 🗣️ Say this — the key insight
> "That number kills the obvious design. At 100K QPS, a centralised limiter *service* means an extra network hop on every single request, and it's now on the critical path of everything — so if it's down, is everything down?
> What I'd argue for instead is a **library plus a shared store**, not a service. Each service does one Redis round trip, about half a millisecond same-AZ, with a token bucket Lua script.
> For the hottest keys I'd add a local tier: each pod keeps a small local allowance refilled from Redis periodically. That trades exactness for throughput — you might allow slightly over the limit for 100 milliseconds — and cuts Redis traffic by an order of magnitude.
> Whether that's acceptable is a product question I'd ask rather than assume."

**Algorithm:** token bucket in Lua — see [04-redis-and-caching.md](04-redis-and-caching.md) Q10.

⚠️ **Follow-ups:** *Redis dies* → "Fail open or closed is a product decision per tier, and the local cache gives a grace period." *Multi-region* → "Per-region budgets summing to the global limit; cross-region coordination costs 100ms and isn't worth it for rate limiting."

---

## D2. Design a URL shortener.

**Estimate.** 100M new URLs/day, 10:1 read:write → ~1,200 writes/s, 12,000 reads/s, ~18 TB/year.

### 🗣️ Say this — the three decisions
> "**ID generation.** Not a hash — truncating MD5 collides, so you'd need a check-and-retry on every write. I'd use a counter encoded in base62, which gives seven characters for 3.5 trillion URLs. But a single sequence can't serve 1,200 writes a second across regions, so each instance takes a *block* of ten thousand IDs from a central allocator and hands them out locally. One coordination call per ten thousand writes instead of per write.
> **Storage.** This is the rare case where I'd genuinely consider leaving Postgres — it's a pure key-value lookup, no joins, 18 terabytes a year growing forever. But I'd be honest: at a 95% cache hit rate Postgres only sees 600 reads a second, which is nothing. The reason to leave is the storage volume, not the read rate.
> **Cache.** Access is extremely skewed and popularity decays fast, so Redis with LRU gets a very high hit ratio. That's the highest-leverage component in the whole design."

### ⚠️ The follow-up that shows you know what the system is *for*
> "301 or 302? A 301 is cached by the browser forever, which means you never see the second click — and click analytics is usually the actual product here. 301 is better for SEO and load; 302 keeps the data. I'd ask which they want."

⚠️ **Also ready:** custom aliases (unique constraint, accept the conflict); expiry (TTL column plus partitioning so deletion is a partition drop); analytics (never write synchronously on the redirect path — emit to a queue).

---

## D3. Design a notification service (email / SMS / push).

### 🗣️ Say this — the four decisions
> "**A queue per channel, not one queue.** Email, SMS and push have completely different rate limits and failure modes. One queue means a Twilio outage backs up your email.
> **Idempotency is mandatory**, not optional. SQS is at-least-once, so without a dedupe key you *will* send duplicate emails — a visible, embarrassing failure. Key on user, notification type and entity ID, with a unique constraint.
> **Provider failure handling.** Retry with full jitter for 5xx and 429, never retry a 400 — a malformed address will never succeed. Circuit breaker per provider, and a fallback provider for critical notifications, which means the template layer has to be provider-agnostic.
> **Preferences checked at send time, not enqueue time** — the user may have unsubscribed in between, and sending after an unsubscribe is a compliance problem, not just a bug."

```
Producers ──▶ SNS topic ──▶ SQS per channel ──▶ Workers ──▶ Provider
                                 │                  │
                            DLQ per queue    Postgres: templates,
                                             preferences, dedupe keys
```

⚠️ **Follow-ups:** *Scheduled sends beyond 15 minutes* → a `scheduled` table with a `send_at` index, polled. *A million users at once* → "Rate-limit the **workers**, not the enqueue. The queue absorbs the burst, which is what it's for." *Priority* → "A separate queue. Password reset must not sit behind a marketing blast."

---

## D4. Design a payment API that never double-charges.

### 🗣️ Say this — lead with the framing
> "Double-charging comes from retries, and retries are unavoidable — the client times out, the network drops the response, a queue redelivers. So I don't try to prevent retries. I make them safe.
> The mechanism is a client-supplied **idempotency key** on the request, stored with a unique constraint, and the *stored response* returned for a repeat.
> The four details that make this actually work: the unique constraint **is** the lock, so I need no Redis lock and no TTL to guess. I fingerprint the request body, so the same key with a different body is a client bug — a 422, not a silent success. An in-progress record returns 409, which handles two simultaneous retries. And I store the whole response, because the client retried precisely because they never got it."

### 💻
```python
@router.post("/orders", status_code=201)
async def create_order(body: OrderCreate,
                       idempotency_key: str = Header(..., alias="Idempotency-Key"),
                       session: AsyncSession = Depends(get_session)):
    fingerprint = hashlib.sha256(body.model_dump_json().encode()).hexdigest()

    if existing := await repo.get_idempotency_record(session, idempotency_key):
        if existing.fingerprint != fingerprint:
            raise HTTPException(422, "key reused with a different body")
        if existing.status == "in_progress":
            raise HTTPException(409, "request in progress, retry shortly")
        return JSONResponse(existing.response_body, status_code=existing.status_code)

    try:
        async with session.begin():
            await repo.insert_idempotency_record(session, key=idempotency_key,
                                                 fingerprint=fingerprint,
                                                 status="in_progress")
    except IntegrityError:                  # <- the unique constraint IS the lock
        raise HTTPException(409, "request in progress, retry shortly")

    order = await service.create_order(body)
    await repo.complete_idempotency_record(session, idempotency_key,
                                           status_code=201, body=order.model_dump())
    return order
```

### ⚠️ The follow-up that separates levels
> "*What if the payment provider call succeeds but you crash before recording it?* You need a reconciliation job comparing their transactions against yours. That **will** happen, and having the job is the difference between a design that works on a whiteboard and one that works in production."

---

## D5. Design a multi-tenant SaaS backend.

### 🗣️ Say this — lead with the question that decides everything
> "The first thing I'd ask is what the isolation requirement is, because that's a compliance and pricing question before it's a technical one, and the three answers give three different architectures.
> I'd default to a shared schema with a `tenant_id` on every table, because it's operationally simplest — one migration, one pool, one backup — and it scales to thousands of tenants.
> The risk is obvious and it's the thing I'd engineer hardest against: one query missing its tenant filter is a cross-tenant data leak. Code review does not reliably catch that.
> So I'd enforce it at the database with row-level security, not in the application. Set the tenant in a session variable from the auth context, and Postgres filters every query whether or not the developer remembered. That turns a catastrophic class of bug into an impossibility."

### 💻
```sql
ALTER TABLE documents ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON documents
  USING (tenant_id = current_setting('app.tenant_id')::uuid);
```
```python
async def get_tenant_session(claims=Depends(verify_token)):
    async with async_session_factory() as s:
        # SET LOCAL: transaction-scoped, so it's safe behind PgBouncer
        await s.execute(text("SET LOCAL app.tenant_id = :t"),
                        {"t": claims["tenant_id"]})
        yield s
```

| Model | Isolation | Cost | Noisy neighbour | Per-tenant restore |
|---|---|---|---|---|
| **Shared schema + tenant_id** | Weakest | Lowest | Yes | Hard |
| Schema per tenant | Middle | Medium | Yes | Easier |
| Database per tenant | Strongest | Highest | No | Easy |

⚠️ **Follow-ups:** *One tenant is 60% of load* → "Hybrid: whales get a dedicated database, the long tail shares. Route through a lookup from day one, even when it always returns the same answer." *Migrations across 5,000 schemas* → "That's the argument against schema-per-tenant — some will fail and now you have drift."

---

## D6. Design an async document-processing pipeline (multi-GB files).

### 🗣️ Say this — the four decisions
> "**The file never touches my API.** Presigned upload straight to S3. A multi-gigabyte upload through a worker occupies it for minutes and hits the load balancer timeout. And the completion signal comes from the **S3 event**, not the client saying 'done' — because clients crash.
> **202 plus polling**, not a long request. There's no synchronous option here; no load balancer holds a connection for ten minutes.
> **Checkpointing.** A forty-minute job that fails at minute thirty-eight and restarts from zero is unacceptable, and it *will* happen — spot reclamation, a rolling deploy, an OOM. So split into chunks and record progress per chunk. That also makes the progress field in the status API honest rather than invented.
> **Scale workers on queue depth, not CPU.** CPU-based scaling reacts after the backlog has already formed; queue-depth scaling reacts to the actual signal."

```
Client ──POST /jobs──▶ API ──▶ presigned S3 URL
   └──upload direct to S3──▶ S3 event ──▶ SQS ──▶ Workers (KEDA) ──▶ S3 output
                                            │              │
                                           DLQ      Postgres: job status
```

⚠️ **Nice payoff to mention:** "Because the work is checkpointed and interruptible, I can run the workers on Spot instances — that's the cost benefit of decision three."

---

## D7. Design a service that survives its dependencies failing.

### 🗣️ Say this
> "I'd start by classifying every dependency as critical or non-critical, because most services treat them all as critical by default. If the recommendations service is down, the product page should render *without* recommendations — not return a 500. That's a one-line decision with a big availability consequence, and it belongs in the design, not the exception handler.
> Then four patterns: timeouts, retries with jitter, circuit breakers, and fallbacks.
> The two things that make this a senior answer are, first, **timeout budgets must shrink down the call chain** — if my SLA is one second and I call something with a two-second timeout, I've broken my own contract before it even fails.
> And second, **retries amplify outages**. Three layers each retrying three times is 27 requests hitting something already struggling. So you retry at one layer only, and you cap retries as a percentage of traffic."

### 💻 The circuit breaker — note the half-open state
```python
class CircuitBreaker:
    def __init__(self, threshold=5, recovery=30.0):
        self.threshold, self.recovery = threshold, recovery
        self.failures, self.opened_at, self.state = 0, 0.0, "closed"

    async def call(self, fn, *a, **kw):
        if self.state == "open":
            if time.monotonic() - self.opened_at < self.recovery:
                raise CircuitOpen()
            self.state = "half_open"        # let ONE request through to probe
        try:
            result = await fn(*a, **kw)
        except Exception:
            self.failures += 1
            if self.state == "half_open" or self.failures >= self.threshold:
                self.state, self.opened_at = "open", time.monotonic()
            raise
        else:
            self.failures, self.state = 0, "closed"
            return result
```
⚠️ **Also name bulkheads:** "A separate connection pool or semaphore per downstream, so one slow dependency can't consume every worker and take down endpoints that don't even call it."

---

## D8. Design a real-time feed / activity stream.

### 🗣️ Say this — the one decision
> "The whole design hangs on one choice: fan-out on write, or fan-out on read.
> Fan-out on write means when you post, I write into every follower's feed. Reads are then a single lookup, which is great because feeds are read far more than written.
> But it breaks on celebrities — a user with ten million followers generates ten million writes for one post.
> So: hybrid. Fan-out on write for normal users. For accounts above a follower threshold, don't fan out at all — at read time, merge the pre-computed feed with a live query of the handful of celebrities that user follows. The merge is cheap because the celebrity count per user is small.
> Storage is a Redis sorted set per user, capped at the most recent thousand, because nobody scrolls past that and it bounds memory."

⚠️ **Follow-ups:** *Ranking* → "Once it isn't chronological, the pre-computed feed becomes a candidate set and ranking happens at read — that changes the design." *Deletes* → "Tombstone and filter at read; removing from ten million feeds is another fan-out."

---

## D9. Design for 99.99% availability.

### 🗣️ Say this
> "Fifty-two minutes a year, total, including deploys. That number rules things out immediately: any deployment strategy with downtime is gone, and so is anything requiring a human to respond — because someone being paged, waking up and diagnosing is fifteen minutes of your annual budget in one incident.
> So: multi-AZ everywhere, N+2 capacity so losing an AZ doesn't degrade you, automated canary with automatic rollback, and every dependency having a defined degraded mode.
> The uncomfortable part is that **your availability is capped by your dependencies**. Serially dependent on three services at four nines each and your ceiling is 99.97% before you write a line of code. Getting to four nines means making most dependencies non-critical, which is a design decision.
> And I'd ask plainly whether the business needs it, or whether someone picked a number that sounded good — the step from three nines to four is a large multiple in cost."

📐 **The budget table**
| SLA | Downtime/year | Downtime/month |
|---|---|---|
| 99% | 3.65 days | 7.2 hours |
| 99.9% | 8.76 hours | 43 minutes |
| **99.99%** | **52.6 minutes** | **4.3 minutes** |
| 99.999% | 5.26 minutes | 26 seconds |

---

## D10. Design a distributed job scheduler (cron-as-a-service).

### 🗣️ Say this
> "The database is the coordinator, and that's the insight. A `next_run_at` column with an index, and nodes claim work with `SELECT ... FOR UPDATE SKIP LOCKED`.
> `SKIP LOCKED` gives mutual exclusion with no lock service, no TTL to guess, and no split-brain — each node simply takes different rows. It's dramatically simpler than Redis locks or leader election.
> The scheduler only *triggers*; the actual work goes onto a queue and separate workers run it, so a long job never blocks scheduling.
> And I'd be honest that this is at-least-once, not exactly-once — a node can enqueue and die before marking it done. So the job itself must be idempotent. I'd say that rather than claim a guarantee I can't deliver."

### 💻
```sql
UPDATE schedules SET locked_until = now() + interval '5 minutes'
WHERE id IN (
  SELECT id FROM schedules
  WHERE next_run_at <= now() AND (locked_until IS NULL OR locked_until < now())
  ORDER BY next_run_at
  FOR UPDATE SKIP LOCKED LIMIT 10
) RETURNING *;
```

⚠️ **Follow-ups worth having ready:** *Scheduler was down two hours — 40 missed runs?* → "A policy per schedule: fire once on recovery, fire all, or skip. There's no universally right answer, and defaulting to 'fire all' has caused real incidents." *Job outlasts its interval* → "An overlap policy; `locked_until` handles skip naturally." *DST* → "Store the timezone and compute in local time. 2:30am doesn't exist one day a year and happens twice on another — use a real library."

---

## D11. Design an API gateway.

### 🗣️ Say this
> "My first instinct is to push back on building one. Kong, Envoy, AWS API Gateway and an ALB with an ingress controller cover most of this, and a hand-built gateway is a component on the critical path of every request that your team now has to keep highly available. I'd want a specific reason.
> If I am building it: authenticate once at the edge and pass a verified identity downstream, so backend services don't each re-validate the JWT.
> The critical constraint is that it must be stateless and async — every request is just waiting on an upstream, so it's a perfect asyncio workload with a shared client and a pool per upstream.
> And a strict timeout per route, because a gateway that waits forever turns one slow service into total saturation."

⚠️ **The interesting follow-up:** *"How do you aggregate three backend calls?"* → "A `TaskGroup`, with a **partial response contract** — if one of three fails, return the two that succeeded with a `partial: true` marker rather than failing everything. That has to be in the response schema from the start; you can't retrofit it."

---

## D12. 100K requests per minute on FastAPI.

### 🗣️ Say this — **do the arithmetic out loud, that's the whole point**
> "A hundred thousand a minute is about 1,700 a second. And I want to say plainly that's not a large number — the trap in this question is over-engineering.
> If each request takes 50 milliseconds and is I/O-bound, one async worker handling forty concurrent requests does maybe 800 a second. So this is roughly four to eight pods with headroom. It does not need Kafka, a service mesh, or microservices.
> The real constraint isn't CPU, it's **database connections**: twenty pods with a pool of twenty is 400 connections, and that's your actual ceiling. PgBouncer or RDS Proxy, and size the pool as capacity divided by max replicas.
> After that: anything blocking the event loop, and the cache hit ratio."

⚠️ **Say this too:** "The thing I'd resist is premature distribution. At 1,700 requests a second a well-tuned monolith on eight pods is simpler to operate, easier to debug, and has fewer failure modes than microservices. I'd want to know the growth curve before designing for 10×."

---

## ⚡ Design-round rapid fire

| They ask | Say this |
|---|---|
| "Where's the bottleneck?" | "Almost always the database, then connection count, then a hot key. Name which and why." |
| "How do you scale reads?" | "Cache, then read replicas, then denormalise, then shard. In that order — and stop as soon as it's enough." |
| "How do you scale writes?" | "Batch, then queue and absorb, then partition, then shard. Writes are much harder and I'd say so." |
| "What's your consistency model?" | "Name it per operation. Strong for the balance, eventual for the activity feed." |
| "Monolith or microservices?" | "Monolith until team size or independent deploy cadence forces the split. Then split on the seam that actually hurts." |
| "How do you handle a hot key?" | "Local cache, key splitting, or a dedicated shard. Scaling out doesn't help — one key is one shard." |
| "What's your single point of failure?" | Find one and say it **before they ask**. It's a credibility move. |
| "How do you test this?" | "Load test to find the knee, chaos-test a dependency failure, and game-day the runbook." |
| "How do you migrate to it?" | "Strangler fig — route a slice of traffic to the new path, expand, retire the old. Never big-bang." |
| "What would you build first?" | "The smallest thing that delivers value and proves the risky assumption. Then name the risky assumption." |

---

**Next:** [10-lld-most-asked.md](10-lld-most-asked.md)

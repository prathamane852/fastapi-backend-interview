# Advanced Backend — Senior Engineer Track (6+ years)

This is the **senior** section, written to be **spoken out loud**, not read silently.

---

## 📐 How every file is built

Each question follows the same four blocks. Only the first one matters on the day.

| Block | What it's for |
|---|---|
| **🗣️ Say this** | **The answer. 4–5 lines, written to be spoken.** Say it once out loud and you're done. |
| **💻 Proof** | Code *plus its actual output*, so the point is visible rather than asserted. |
| **🧠 If they dig** | Tables, mechanisms and detail — for when they push further. |
| **⚠️ Follow-up** | The next question they'll ask, with the answer ready. |

Each file also opens with a **📖 Jargon, in plain words** table:

| Jargon | What it actually means | How you'd say it |
|---|---|---|
| **MVCC** | The database keeps old copies of rows so readers see a snapshot instead of waiting. | "MVCC is why readers never block writers." |

**Never use a word in an interview you can't unpack in one sentence.** That middle column is the unpack.

### About the `Output:` blocks

- **(real)** — actually executed on Python 3.14. The numbers are genuine.
- **(what you'd see)** — representative Postgres / `redis-cli` / `kubectl` output, since those need live services.

---

## 🎯 The seniority bar

At 2 years you're asked **what**. At 6 years you're asked **why, what breaks, and what did it cost**. Every 🗣️ block tries to hit four marks:

| Mark | What it sounds like |
|---|---|
| **Mechanism** | "Postgres doesn't update in place — it writes a new version and marks the old one dead. That's why…" |
| **Failure mode** | "This falls over when the pool is smaller than the worker count, and the symptom is…" |
| **Trade-off, said out loud** | "I took the write amplification hit to get the index-only scan. If writes were hotter I'd have gone the other way." |
| **Evidence** | "I knew because `pg_stat_statements` showed mean time going from 4ms to 90ms after the deploy." |

Mechanism alone sounds like someone with good reading habits. The last two are what actually distinguish a senior.

---

## Files

| # | File | What it covers |
|---|---|---|
| 1 | [01-python-advanced.md](01-python-advanced.md) | CPython internals, GIL & free-threading, memory leaks, asyncio deep dive, contextvars, descriptors, profiling |
| 2 | [02-databases-advanced.md](02-databases-advanced.md) | Isolation levels, write skew, locking, N+1, cursor pagination, sharding, CAP/PACELC, Saga & outbox |
| 3 | [03-postgres-deep-dive.md](03-postgres-deep-dive.md) | MVCC, VACUUM & bloat, wraparound, index types, `EXPLAIN ANALYZE`, PgBouncer, partitioning, zero-downtime DDL |
| 4 | [04-redis-and-caching.md](04-redis-and-caching.md) | Data structures, eviction, caching patterns, stampede, distributed locks, rate limiting, Streams, Cluster |
| 5 | [05-message-queues-sqs-sns-kafka.md](05-message-queues-sqs-sns-kafka.md) | Delivery semantics, idempotency, outbox, visibility timeout, DLQ, backoff, Kafka, backpressure |
| 6 | [06-aws-for-backend.md](06-aws-for-backend.md) | IRSA, VPC & NAT cost, RDS connections, ElastiCache, S3 patterns, compute comparison, ALB, cost cutting |
| 7 | [07-deployments-and-release.md](07-deployments-and-release.md) | Zero-downtime mechanics, probes, canary, expand-contract, GitOps, feature flags, CrashLoop & OOM |
| 8 | [08-opentelemetry-advanced.md](08-opentelemetry-advanced.md) | Collector architecture, propagation, sampling, cardinality, exemplars, burn-rate alerts, cost |
| 9 | [09-system-design-advanced.md](09-system-design-advanced.md) | A 45-minute framework, numbers to memorise, 12 deep problems |
| 10 | [10-lld-most-asked.md](10-lld-most-asked.md) | 12 low-level design problems with working code and the narration that scores |
| 11 | [11-design-patterns-in-interviews.md](11-design-patterns-in-interviews.md) | **The four-move script for explaining any pattern out loud** |
| 12 | [12-rapid-fire-senior.md](12-rapid-fire-senior.md) | 100 spoken-sentence answers for the night before |

---

## How to study

**Six weeks out.** One file every two days, in order. Files 1–8 are knowledge; 9–11 are performance. Don't skip ahead.

**Two weeks out.** Do 9, 10 and 11 properly — **out loud, with a timer, writing on paper.** Design and LLD rounds grade how you *communicate*, and you cannot fake that by reading.

**The night before.** [12-rapid-fire-senior.md](12-rapid-fire-senior.md) only.

**The rule that matters more than any content here:** for every topic, have **one story from your own work**. "Autovacuum falls behind on high-churn tables" is a fact anyone can read. "Our `job_runs` table hit 47% bloat and scans went from 200ms to 3s over a weekend; I dropped `autovacuum_vacuum_scale_factor` to 0.01 on that table" is six years of experience. The content here is scaffolding — hang your own incidents on it.

---

## Reading the symbols

- 🗣️ **Say this** — the spoken answer.
- 📖 Jargon, in plain words.
- 💻 Code, with real output.
- 🧠 The mechanism underneath.
- ⚠️ The follow-up that's coming, or the trap.
- 📐 Numbers worth memorising.

Code targets **Python 3.11+ (verified on 3.14), FastAPI 0.110+, Pydantic v2, SQLAlchemy 2.0, PostgreSQL 15+**.

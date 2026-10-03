# 02 — Databases, Senior Level

> **How to use this file.** Every question has a **🗣️ Say this** block — 4 to 5 lines, written to be spoken. That's your answer. The rest is backup for the follow-up.
>
> `Output:` blocks marked **(real)** were actually run. Blocks marked **(what you'd see)** are representative Postgres output.

Postgres-specific mechanics live in [03-postgres-deep-dive.md](03-postgres-deep-dive.md). This file is the vendor-neutral layer.

---

## 📖 Jargon, in plain words

| Jargon | What it actually means | How you'd say it |
|---|---|---|
| **Transaction** | A group of changes that either all happen or none happen. | "A transaction is all-or-nothing." |
| **ACID** | Four promises: all-or-nothing, leaves valid data, concurrent work doesn't interfere, survives a crash. | "ACID is the contract that makes a transaction trustworthy." |
| **Isolation level** | A dial: how much can two transactions running at once see of each other's half-finished work? | "Isolation level trades safety against concurrency." |
| **Dirty read** | Seeing data someone else hasn't committed yet — and might undo. | "Postgres never allows dirty reads, at any level." |
| **Non-repeatable read** | Reading the same row twice in one transaction and getting different answers. | "At Read Committed, two SELECTs can disagree." |
| **Phantom read** | Running the same *query* twice and getting extra rows. | "New rows appearing mid-transaction." |
| **Write skew** | Two transactions each read the same data, each makes a decision that's fine on its own, and together they break a rule. | "The anomaly people don't know about — it's why Serializable exists." |
| **MVCC** | The database keeps old copies of rows so readers see a snapshot instead of waiting. | "MVCC is why readers never block writers." |
| **Pessimistic locking** | Lock it before you touch it, assume a fight. | "It's correct, and it costs throughput." |
| **Optimistic locking** | Don't lock. Carry a version number, and fail the write if someone changed it first. | "Assume conflicts are rare and make the loser retry." |
| **Deadlock** | Two transactions each holding what the other needs. The database kills one. | "Deadlocks come from inconsistent lock ordering." |
| **Idempotent** | Doing it twice has the same effect as doing it once. | "Idempotency is what makes retries safe." |
| **N+1** | One query to get a list, then one more query per item. | "N+1 turns 200 rows into 201 round trips." |
| **Eager loading** | Telling the ORM to fetch the related data up front, in one or two queries. | "`selectinload` collapses N+1 into two queries." |
| **Unit of Work** | Collect all your changes in memory, then write them in one transaction. | "SQLAlchemy's Session is a Unit of Work." |
| **Sharding** | Splitting one table across several separate databases. | "The hard part is picking the key — re-sharding is brutal." |
| **CAP / PACELC** | During a network split, pick consistency or availability. The rest of the time, pick latency or consistency. | "CAP only applies during a partition; PACELC covers normal days." |
| **Saga** | A multi-step process where each step has an 'undo' step, replacing a distributed transaction. | "You give up atomicity and buy it back with compensations." |
| **Outbox** | Write your data *and* the message-to-send in the same transaction, then let a relay publish it. | "It fixes 'committed to the DB but never published the event'." |

---

# Section A — Transactions

## Q1. Walk me through the isolation levels.

**Jargon here:** *isolation level* (how much concurrent transactions see of each other), *snapshot* (a frozen view of the data).

### 🗣️ Say this
> "There are four, but in Postgres only three behave differently. The default is Read Committed, and the thing people miss about it is that each *statement* gets a fresh snapshot — so two SELECTs in the same transaction can genuinely return different answers.
> Repeatable Read takes one snapshot at the start and holds it, so your reads are stable for the whole transaction. In Postgres that's true snapshot isolation, and it also blocks phantom reads, which the SQL standard doesn't even require.
> Serializable is the strongest — it guarantees the result is the same as if the transactions ran one after another.
> The one I'd raise without being asked is write skew, because that's the anomaly Repeatable Read does *not* prevent, and it's the whole reason Serializable exists."

### 🧠 The table to draw
| Level | Dirty read | Non-repeatable | Phantom | Write skew |
|---|---|---|---|---|
| Read Uncommitted | — | possible | possible | possible | *(Postgres treats this as Read Committed)* |
| **Read Committed** (default) | ❌ | ⚠️ possible | ⚠️ possible | ⚠️ possible |
| **Repeatable Read** | ❌ | ❌ | ❌ | ⚠️ **possible** |
| **Serializable** | ❌ | ❌ | ❌ | ❌ |

### 💻 Write skew, concretely — *(what you'd see)*
The rule: **at least one doctor must stay on call.** Two doctors, both on call, both try to leave at the same moment.

```sql
-- Both transactions run concurrently at REPEATABLE READ
BEGIN;
SELECT count(*) FROM on_call WHERE shift_id = 5 AND on_call = true;
--  count
-- -------
--      2        <- both transactions see 2, both think "safe for me to leave"

UPDATE on_call SET on_call = false WHERE doctor_id = <me> AND shift_id = 5;
COMMIT;
-- Both succeed. Zero doctors on call. Rule broken.
```
> "Snapshot isolation can't catch it because they wrote to *different rows*. There's no write conflict — the conflict is in what they read. That's the definition of write skew."

**Three fixes:** run Serializable and retry on error `40001`; or lock the parent row (`SELECT ... FOR UPDATE` on the shift) so the conflict becomes real; or write the rule as a database constraint.

### ⚠️ Follow-up: "What does Serializable cost?"
> "Not blocking, which surprises people. Postgres doesn't take extra locks — it watches for dangerous patterns and aborts one transaction after the fact. So the cost is retries: your app has to handle error code 40001 and try again. That means the transaction must be safe to re-run, so no sending emails before you commit. I'd use it on the specific transactions guarding a rule, not globally."

```python
def with_retry(fn, attempts=3):
    for i in range(attempts):
        try:
            with Session(engine.execution_options(isolation_level="SERIALIZABLE")) as s:
                r = fn(s); s.commit(); return r
        except DBAPIError as e:
            if e.orig.sqlstate not in ("40001", "40P01") or i == attempts - 1:
                raise
            time.sleep((2 ** i) * 0.05 * random.uniform(0.5, 1.5))   # backoff + jitter
```

---

## Q2. Where does the transaction boundary belong in a FastAPI app?

### 🗣️ Say this
> "One transaction per request, owned by the service layer — not the route, and definitely not the repository.
> The rule I follow is that repositories never commit. If every repository method commits, you've lost atomicity across a use case and you can't compose two of them into one operation.
> I wire it as a FastAPI dependency that yields a session and rolls back on an exception.
> The thing I'd flag hardest is never holding a transaction open across an external HTTP call. If you begin, then call a payment provider for eight seconds, then commit — you've pinned a database connection and held row locks for eight seconds. Under load that drains the pool, and it looks like the database is down when it isn't."

### 💻 The pattern
```python
async def get_session() -> AsyncIterator[AsyncSession]:
    async with async_session_factory() as session:
        try:
            yield session
            await session.commit()
        except Exception:
            await session.rollback()
            raise
        # context manager closes -> connection goes back to the pool
```

### ⚠️ Follow-up: "So what's the shape for a payment?"
> "Short transaction to write the intent, commit, make the external call, then a short transaction to record the result. Never one long transaction wrapped around the network call."

---

## Q3. Optimistic or pessimistic locking?

**Jargon here:** *pessimistic* (lock first, assume a fight), *optimistic* (don't lock, detect the conflict afterwards).

### 🗣️ Say this
> "Pessimistic means you lock the row before you touch it — `SELECT FOR UPDATE`. Nobody else can touch it until you commit. It's correct, but it's blocking, so it puts a ceiling on throughput for hot rows.
> Optimistic means you don't lock at all. You remember the version number you read, and your UPDATE says 'only apply this if the version is still what I saw'. If someone beat you to it, you affect zero rows and you know you lost.
> I default to optimistic for anything user-facing, because a user might sit on a form for five minutes and you absolutely cannot hold a row lock that long.
> I go pessimistic when contention is genuinely high and the critical section is microseconds — like decrementing stock."

### 💻 Proof — *(real, SQLite)*
```python
seen = 7                      # both readers saw version 7
c1 = db.execute("UPDATE acct SET balance=balance-100, version=version+1 "
                "WHERE id=1 AND version=?", (seen,))
c2 = db.execute("UPDATE acct SET balance=balance-100, version=version+1 "
                "WHERE id=1 AND version=?", (seen,))
print("writer 1 rowcount =", c1.rowcount)
print("writer 2 rowcount =", c2.rowcount)
```
```
Output:
writer 1 rowcount = 1  -> wins
writer 2 rowcount = 0  -> lost, must retry (return 409)
final balance     = 400 (one debit, not two)
```
> `rowcount == 0` **is** the conflict detection. No lock, no lock manager, no TTL to guess — the WHERE clause did it.

### 🧠 The third option people forget
> "`FOR UPDATE SKIP LOCKED` is often the right answer and nobody mentions it. It's how you build a job queue in Postgres — each worker grabs a *different* row instead of all queueing behind the same one."

```sql
UPDATE jobs SET status='running'
WHERE id IN (
  SELECT id FROM jobs WHERE status='pending'
  ORDER BY created_at
  FOR UPDATE SKIP LOCKED          -- <- each worker gets different rows
  LIMIT 10
) RETURNING *;
```

---

## Q4. You're getting deadlocks. What now?

### 🗣️ Say this
> "Deadlocks aren't random — they're almost always two code paths grabbing the same two rows in opposite order. So the first thing I do is read the Postgres log, because it prints both statements and the lock graph.
> Then I fix the *ordering*, usually by sorting the batch by primary key before touching anything. That makes the collision impossible rather than just less likely.
> Retrying is correct as a safety net — deadlock is error code 40P01 and it's always safe to retry — but retrying without fixing the order just converts a crash into latency.
> The sneaky third cause is foreign keys: inserting a child row quietly locks the parent, so two transactions can deadlock on rows they never explicitly touched."

### 💻 Find what's blocking what, right now — *(what you'd see)*
```sql
SELECT blocked.pid AS blocked_pid, left(blocked.query, 40)  AS blocked_query,
       blocking.pid AS blocking_pid, left(blocking.query,40) AS blocking_query,
       now() - blocked.query_start AS waiting_for
FROM pg_stat_activity blocked
JOIN pg_stat_activity blocking ON blocking.pid = ANY(pg_blocking_pids(blocked.pid))
WHERE blocked.wait_event_type = 'Lock';
```
```
 blocked_pid |      blocked_query       | blocking_pid |     blocking_query      | waiting_for
-------------+--------------------------+--------------+-------------------------+-------------
       28471 | UPDATE orders SET status | 28455        | UPDATE orders SET total | 00:00:14.3
```

---

# Section B — ORMs at scale

## Q5. What's the N+1 problem, and how do you stop it reaching production?

**Jargon here:** *N+1* (one query for the list, then one per item), *lazy loading* (ORM fetches related data only when you touch it), *eager loading* (fetch it up front).

### 🗣️ Say this
> "N+1 is one query to fetch a list, then one extra query for every item in it, because the ORM lazily loads the related data when you touch it.
> The definition is easy — what actually matters is that it's invisible in development. With five test rows it costs three milliseconds. In production with five hundred rows and two milliseconds of network latency, it's a full second of pure round trips.
> So my answer is about prevention, not diagnosis. I set relationships to `lazy='raise'`, which turns an accidental lazy load into a loud exception at development time instead of a silent slowdown in production.
> And I assert a query budget in tests — count the queries for an endpoint and fail the build if it goes above three."

### 💻 Proof — *(real, SQLite, 100 users)*
```python
# N+1 style
users = db.execute("SELECT id FROM users").fetchall()
for (uid,) in users:
    db.execute("SELECT * FROM orders WHERE user_id=?", (uid,)).fetchall()

# eager style (what selectinload does)
users = db.execute("SELECT id FROM users").fetchall()
ids = ",".join(str(u[0]) for u in users)
db.execute(f"SELECT * FROM orders WHERE user_id IN ({ids})").fetchall()
```
```
Output:
N+1 style         : 101 queries
selectinload style:   2 queries
```
> Same data. 101 round trips versus 2. At 2ms network latency that's 202ms versus 4ms.

### 🧠 Prevention, in code
```python
class Order(Base):
    items: Mapped[list["Item"]] = relationship(lazy="raise")   # accidental lazy load
                                                               # now raises loudly
```
```python
def test_list_orders_query_budget(client, query_counter):
    client.get("/orders?limit=100")
    assert len(query_counter) <= 3, f"N+1 detected: {len(query_counter)} queries"
```

### ⚠️ Follow-up: "`selectinload` or `joinedload`?"
> "`selectinload` for collections — it runs a second query with `WHERE id IN (...)`, so each row comes back exactly once. `joinedload` for many-to-one, where you're pulling a single related row.
> The reason not to always use `joinedload` is cartesian explosion. An order with 50 items and 3 shipments joined together gives you 150 rows, each carrying a full copy of the order columns. You end up shipping megabytes for kilobytes of actual data."

| Strategy | What it does | Use for |
|---|---|---|
| `selectinload` | 2 queries, `WHERE id IN (...)` | **Default for collections** |
| `joinedload` | 1 query, LEFT JOIN | Many-to-one / one-to-one |
| `contains_eager` | You wrote the JOIN yourself | When you also filter on the joined table |
| Don't load it | — | Check the response even needs it |

---

## Q6. What is the SQLAlchemy `Session` actually doing?

### 🗣️ Say this
> "Two things: it's a Unit of Work and an identity map.
> Unit of Work means your changes pile up in memory and only become SQL when it flushes — which happens automatically before a query and on commit. So everything commits together.
> Identity map means one Python object per primary key. Load the same row twice and you get the same object back.
> The consequences that bite you: a long-lived session holds a strong reference to every row it ever loaded, so it's a memory leak in a batch job. And after commit, objects are expired by default, so touching an attribute fires a fresh query — which is a sneaky source of N+1 right at the end of a request."

### ⚠️ Follow-up: "Session per request, or one global session?"
> "Per request, always. And for async there's a second reason beyond memory: `AsyncSession` is explicitly not safe for concurrent use. If you share one across `asyncio.gather`, you get interleaved statements on one connection and errors about a command already being in progress. Every concurrent branch needs its own session."

---

## Q7. Cursor vs offset pagination.

**Jargon here:** *offset* (skip N rows), *keyset/cursor* (start after the last row you saw).

### 🗣️ Say this
> "`OFFSET` makes the database produce and then throw away all the rows you skipped. At offset 500,000 it reads half a million rows to give you twenty. So it gets linearly slower the deeper you page.
> It's also *wrong* on live data. If someone inserts a row while the user is paging, everything shifts down and they see an item twice or miss one completely.
> Cursor pagination uses the last row you saw as a WHERE clause, so the database seeks straight into the index. It's the same speed on page one and page fifty thousand, and it's stable under concurrent writes.
> The trade-off I'd name is that you lose random access — you can't jump to page 50, and you can't show a total page count cheaply. So for a numbered-page UI, offset with a hard cap is fine. For infinite scroll or any API a machine consumes, cursor is the only right answer."

### 💻 Proof — *(real, SQLite, 300k rows)*
```python
for off in (0, 100000, 250000):
    db.execute(f"SELECT * FROM ev ORDER BY id LIMIT 20 OFFSET {off}").fetchall()
db.execute("SELECT * FROM ev WHERE id > 250000 ORDER BY id LIMIT 20").fetchall()
```
```
Output:
OFFSET      0 :    0.1 ms
OFFSET 100000 :    0.6 ms
OFFSET 250000 :    1.3 ms     <- 13x slower, and it keeps growing
keyset (id >) :    0.0 ms     <- flat, at any depth
```
> The offset time scales with how far in you are. The keyset time doesn't move. On a real table with millions of rows that gap is seconds, not milliseconds.

### 💻 The real implementation
```sql
-- Tiebreak on the primary key so the ordering is total
SELECT * FROM events
WHERE (created_at, id) < (:last_created_at, :last_id)
ORDER BY created_at DESC, id DESC
LIMIT 20;
```
```python
def encode_cursor(row) -> str:
    return base64.urlsafe_b64encode(
        json.dumps({"c": row.created_at.isoformat(), "i": str(row.id)}).encode()).decode()
```
⚠️ **The index that makes it work:** you need `(created_at DESC, id DESC)` matching the ORDER BY exactly, or you've bought nothing.

---

## Q8. "This endpoint returns a million records." Fix it.

### 🗣️ Say this
> "I'd push back on the requirement first, because there are only three real reasons behind it and they have different answers.
> If a human is looking at it, they don't need a million rows — that's pagination plus filters.
> If it's a machine syncing, that's a cursor-paginated feed keyed on `updated_at`, so they only pull what changed.
> If it's genuinely a bulk export, it shouldn't be a synchronous HTTP response at all. It should be an async job that writes to S3 and hands back a presigned URL — because any synchronous version is one load-balancer timeout away from failing at 95% complete with no way to resume."

### 💻 If it must stream
```python
@router.get("/export")
async def export(session: AsyncSession = Depends(get_session)):
    async def rows():
        yield "id,email,created_at\n"
        stmt = select(User).execution_options(yield_per=1000)   # server-side cursor
        async for partition in (await session.stream(stmt)).partitions(1000):
            yield "".join(f"{u.id},{u.email},{u.created_at}\n" for u in partition)
    return StreamingResponse(rows(), media_type="text/csv")
```

### ⚠️ Three things that break this — volunteer them
> "One, the transaction stays open for the whole stream, so a ten-minute export pins a connection and blocks VACUUM. Use a replica.
> Two, a buffering proxy will hold the whole thing in memory anyway, defeating the point.
> Three, you can't return a sensible error halfway through — the 200 and the headers are already sent, so if row 800,000 fails the client just gets a truncated CSV that looks complete. That last one on its own is the argument for the async-job approach."

---

# Section C — Scaling

## Q9. Read replicas — what breaks?

**Jargon here:** *replica* (a read-only copy), *replication lag* (how far behind it is), *read-your-writes* (seeing your own change immediately).

### 🗣️ Say this
> "Two engines and two session factories, routed by intent — writes to the primary, reads to the replica.
> What breaks is replication lag. The user POSTs an order, it lands on the primary, then they immediately GET it, that read goes to a replica that's 200 milliseconds behind, and you return a 404 for something you just created. That's a read-your-writes violation and it's the number one surprise with replicas.
> My usual fix is sticky-after-write: once a user does a write, pin their reads to the primary for a few seconds using a flag in Redis.
> And I'd always export replication lag as a metric and fall back to the primary when it spikes — serving stale data silently is worse than serving it slowly."

### 💻 What lag looks like — *(what you'd see)*
```sql
SELECT client_addr, state, replay_lag FROM pg_stat_replication;
```
```
 client_addr  |   state   |   replay_lag
--------------+-----------+-----------------
 10.0.2.15    | streaming | 00:00:00.184     <- 184ms behind
```

```python
primary = create_async_engine(PRIMARY_URL, pool_size=20)
replica = create_async_engine(REPLICA_URL, pool_size=40)

async def get_read_session():                 # no commit path = read-only by design
    async with AsyncSession(replica) as s:
        yield s
```

---

## Q10. When would you shard?

### 🗣️ Say this
> "Much later than most people think, and I'd lead with that reluctance.
> The ladder I'd climb first is: fix the indexes, then cache, then add read replicas, then partition inside one database, then move the biggest table to its own database. Sharding is after all of that.
> The reason is that sharding costs you cross-shard joins, distributed transactions, rebalancing, and the ability to just run an ad-hoc query — that's a permanent tax on every future feature.
> I'd shard when a single primary genuinely can't absorb the *write* volume, or when the working set no longer fits in memory. Those are the two problems replicas and caches can't solve."

### 🧠 Picking the key — four criteria
1. **High cardinality**, so shards fill evenly.
2. **Even access.** `tenant_id` is natural but creates hot shards when one customer is 40% of traffic — have an answer for that (dedicated shards for whales).
3. **It's in your hot queries.** If the key isn't in the WHERE clause, every query fans out to every shard and you've made things worse.
4. **It never changes.** Re-sharding because the key changed is the worst day of the quarter.

| Strategy | Good | Bad |
|---|---|---|
| Hash % N | Even spread | Re-sharding rehashes everything — use consistent hashing |
| Range | Range queries stay on one shard | Hot spots: everything new hits the last shard |
| Lookup table | Total flexibility, easy rebalance, whale isolation | An extra lookup; the directory is a SPOF you must cache |

---

## Q11. Explain CAP honestly.

### 🗣️ Say this
> "CAP says that *during a network partition* you have to choose between consistency and availability. The mistake almost everyone makes is treating it like a menu you pick two from at design time — saying 'we're an AP system'.
> But partitions are rare events, and the theorem says nothing about the other 99% of the time.
> That's why I prefer PACELC: during a **P**artition choose **A** or **C**; **E**lse, choose **L**atency or **C**onsistency. That second half is the trade you actually make every single day.
> Every read replica, every cache, every async replication setting is a latency-versus-consistency decision with no partition anywhere in sight."

| System | During a partition | Normal operation |
|---|---|---|
| Postgres + sync replica | CP | Consistency |
| Postgres + **async** replica | CP | **Latency** ← this is the lag from Q9 |
| DynamoDB (default) | AP | Latency |
| DynamoDB (`ConsistentRead=true`) | AP | Consistency |

---

## Q12. You can't have a transaction across two services. What do you do?

**Jargon here:** *Saga* (a chain of steps, each with an undo), *compensation* (the undo step), *dual-write* (writing to two places with no atomicity).

### 🗣️ Say this
> "Two-phase commit technically exists and I'd avoid it — the coordinator is a single point of failure, and a participant that crashes while prepared holds locks indefinitely.
> The practical answer is a Saga: model it as a sequence of local transactions, each with a compensating action, and accept it's eventually consistent with a visible window of inconsistency.
> Order created, payment authorised, inventory reserved. If inventory fails, you run the compensation for payment — which is a refund, not a rollback.
> That distinction matters and it's worth saying out loud: compensations are new business facts, not time travel. Some things, like an email you already sent, have no compensation at all."

### ⚠️ The part everyone forgets — the dual-write bug
> "Here's the actual bug. You commit the order to Postgres, then publish to SQS, and the process dies in between. The order exists and no downstream service will ever hear about it. You cannot fix that with try/except, because the failure is in the *gap*.
> The fix is the transactional outbox — insert the event into an `outbox` table in the *same transaction* as the order. One commit, both or neither. Then a separate relay reads that table and publishes.
> The relay is at-least-once, so consumers must be idempotent — but that was already true."

```sql
BEGIN;
  INSERT INTO orders (id, user_id, total, status) VALUES (:id, :uid, :total, 'created');
  INSERT INTO outbox (id, aggregate_id, type, payload)
       VALUES (gen_random_uuid(), :id, 'order.created', :payload::jsonb);
COMMIT;
-- relay: SELECT ... FOR UPDATE SKIP LOCKED -> publish -> mark published
```

### 🧠 Orchestration vs choreography
| | Orchestration | Choreography |
|---|---|---|
| Shape | One coordinator drives each step | Each service reacts to events |
| Can you read the flow? | Yes, in one place | No — it exists in nobody's code |
| My default | **Start here** past 3 steps | Fine for 2–3 independent services |

---

## Q13. SQL or NoSQL?

### 🗣️ Say this
> "I'd start from the access pattern, not the data shape — 'my data is nested' is a bad reason, because Postgres has JSONB and it's excellent.
> The questions I'd actually ask: do I need rules enforced across multiple rows? Do I need ad-hoc queries I can't predict today? Is my write volume past what one primary can take?
> Postgres is my default and I'd want a concrete reason to leave it. Good reasons: single-digit-millisecond latency at huge scale with a known key pattern, which is DynamoDB. Full-text relevance ranking, which is OpenSearch. Very high-ingest time series, which is Timescale or Clickhouse.
> Bad reasons: 'schemaless is faster to develop' — that just moves the schema into your application where nothing enforces it — or 'we might need to scale', which is speculative and expensive."

### ⚠️ Worth adding
> "Every extra datastore is another thing to back up, monitor, patch and get paged for, plus a new consistency boundary. I'd rather have one Postgres doing four jobs adequately than four specialised stores, until one of those jobs proves it needs its own home."

---

## ⚡ Rapid fire

| They ask | Say this |
|---|---|
| Postgres default isolation? | "Read Committed — and the catch is it's a fresh snapshot per *statement*, so two SELECTs in one transaction can differ." |
| Which errors are safe to retry? | "40001 serialization failure and 40P01 deadlock. Both with backoff, and the transaction has to be side-effect free." |
| `TRUNCATE` vs `DELETE`? | "`TRUNCATE` is instant because it skips per-row work, but it takes an exclusive lock and can't be filtered. `DELETE` is row-by-row and leaves dead rows behind." |
| Why is `COUNT(*)` slow? | "Because visibility is per-transaction under MVCC, so it has to actually scan. Use an estimate from `reltuples` or keep a counter." |
| UPSERT? | "`INSERT ... ON CONFLICT (col) DO UPDATE SET ...`. It needs a unique constraint on the conflict target." |
| Preventing SQL injection? | "Parameterised queries, always. ORMs do it by default — the risk is `text()` with an f-string, and dynamic `ORDER BY`, which needs a whitelist." |
| Connection pool sizing? | "Small. Roughly `(2 × cores)` total for the database, split across app instances. More connections is not more throughput — past a point it's pure contention." |
| Why not pool 100 per pod? | "20 pods × 100 is 2,000 connections. Postgres forks a process per connection, so you hit `max_connections` and each idle one still costs memory. That's what PgBouncer is for." |
| Soft delete cost? | "Every query needs `WHERE deleted_at IS NULL`, your indexes get less selective, and unique constraints have to become partial. Use it deliberately, not by default." |
| Natural or surrogate key? | "Surrogate, almost always — natural keys change. And prefer UUIDv7 or ULID over UUIDv4 so it's time-ordered and the index stays tidy." |

---

**Next:** [03-postgres-deep-dive.md](03-postgres-deep-dive.md)

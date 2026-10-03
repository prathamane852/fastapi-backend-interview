# 03 — PostgreSQL Deep Dive

> **How to use this file.** Every question has a **🗣️ Say this** block — 4 to 5 lines, ready to speak. `Output:` blocks are marked **(what you'd see)** — representative real Postgres output.

**The one idea that ties this whole file together:** almost every Postgres operational problem traces back to MVCC. Bloat, slow queries, blocked migrations, disk filling up, wraparound — same root cause. If you understand that one mechanism, you can reason your way to the rest live in the room.

---

## 📖 Jargon, in plain words

| Jargon | What it actually means | How you'd say it |
|---|---|---|
| **Tuple** | One physical copy of a row on disk. An UPDATE makes a new one. | "Postgres never edits a row in place — it writes a new version." |
| **MVCC** | Keeping old row versions around so readers see a consistent snapshot. | "That's why readers never block writers." |
| **xmin / xmax** | Hidden columns on every row: which transaction created it, which killed it. | "Visibility is just comparing those against your snapshot." |
| **Dead tuple** | An old row version nobody can see any more. Garbage. | "Dead tuples are what MVCC leaves behind." |
| **Bloat** | Your table is physically much bigger than the live data in it. | "Bloat means scans read more pages for the same rows." |
| **VACUUM** | The cleaner that marks dead tuple space reusable. | "VACUUM makes space reusable; it doesn't shrink the file." |
| **Autovacuum** | VACUUM running automatically in the background. | "It's on by default, and the defaults are wrong for busy tables." |
| **XID wraparound** | Transaction IDs are 32-bit and can run out. Postgres shuts down to protect you. | "It's the one failure that stops the database completely." |
| **HOT update** | An update cheap enough that no index needs touching. | "HOT updates are why `fillfactor` matters on busy tables." |
| **WAL** | A log of every change, written before the data files. | "WAL is the basis of crash recovery and replication." |
| **TOAST** | Big values get compressed and moved to a side table. | "Large JSONB gets TOASTed, so reading it costs an extra fetch." |
| **`pg_stat_statements`** | An extension that tracks stats per query shape. | "It's the first place I look — it ranks queries by total time." |
| **Planner statistics** | Samples the planner uses to guess how many rows a query returns. | "Bad plans are usually stale statistics, not a broken planner." |
| **Index-only scan** | Answering a query from the index alone, never reading the table. | "It only works if VACUUM has marked the pages all-visible." |
| **PgBouncer** | A connection pooler that sits in front of Postgres. | "It multiplexes thousands of clients onto a few dozen connections." |

---

# Section A — MVCC and everything it causes

## Q1. Explain MVCC and why it creates work for you.

### 🗣️ Say this
> "Postgres never updates a row in place. Every row carries two hidden columns — which transaction created it and which one deleted it — and an UPDATE is physically an insert of a new version plus marking the old one dead.
> Your transaction takes a snapshot, and visibility is just a comparison against that snapshot. The payoff is genuinely good: readers never block writers and writers never block readers.
> The cost is that the database accumulates garbage in proportion to your write rate, and something has to clean it up. That's VACUUM.
> And that's the punchline — essentially every Postgres operations problem I've dealt with is downstream of vacuum not keeping up."

### 💻 The hidden columns are real — *(what you'd see)*
```sql
SELECT xmin, xmax, ctid, id, balance FROM accounts WHERE id = 1;
```
```
  xmin  | xmax |  ctid  | id | balance
--------+------+--------+----+---------
 138422 |    0 | (0,1)  |  1 |     500

UPDATE accounts SET balance = 400 WHERE id = 1;
SELECT xmin, xmax, ctid, id, balance FROM accounts WHERE id = 1;

  xmin  | xmax |  ctid  | id | balance
--------+------+--------+----+---------
 138423 |    0 | (0,2)  |  1 |     400     <- new ctid = a NEW physical row
```
> The `ctid` changed from `(0,1)` to `(0,2)`. The old version is still on disk, just marked dead. That's the garbage.

### ⚠️ Follow-up: "So why does one long-running query cause problems?" — **this is *the* Postgres question**
> "Because VACUUM can only remove a dead row if no snapshot can still see it. An open transaction — even one that's just sitting idle after a single SELECT — holds back the cutoff for the *entire database*.
> So a reporting query someone left running for three hours means three hours of accumulated garbage that autovacuum keeps scanning and skipping.
> What you see is bloat growing, disk filling, and queries getting slower for no obvious reason. The actual cause is one `idle in transaction` connection somebody's ORM forgot to close.
> That's why I always set `idle_in_transaction_session_timeout` and alert on the oldest running transaction."

```sql
-- The first query I'd run on any bloat or disk complaint
SELECT pid, state, now() - xact_start AS xact_age, left(query, 50)
FROM pg_stat_activity
WHERE state <> 'idle' OR state = 'idle in transaction'
ORDER BY xact_start NULLS LAST LIMIT 5;
```
```
  pid  |        state        |   xact_age   |                query
-------+---------------------+--------------+-------------------------------
 14822 | idle in transaction | 03:14:07.2   | SELECT * FROM orders WHERE ...
```
> Three hours idle in transaction. There's your bloat.

---

## Q2. Table bloat — find it, fix it, prevent it.

### 🗣️ Say this
> "Bloat is when the table is physically much bigger than the live data, because dead row versions are still occupying pages. The effect is that a scan reads far more pages for the same number of rows, so everything gets slower.
> I detect it from `pg_stat_user_tables` — if dead tuples are sustained above about 20% of a hot table, autovacuum is losing the race.
> To fix it online I use `pg_repack`, which rebuilds the table and swaps at the end with only a brief lock. `VACUUM FULL` also works but takes an exclusive lock, so it's an outage on that table — never on something live.
> To prevent it, I tune autovacuum per table, because the defaults are genuinely wrong for high-churn tables."

### 💻 Detect — *(what you'd see)*
```sql
SELECT relname, n_live_tup, n_dead_tup,
       round(100.0*n_dead_tup/NULLIF(n_live_tup+n_dead_tup,0),1) AS dead_pct,
       last_autovacuum
FROM pg_stat_user_tables WHERE n_dead_tup > 10000 ORDER BY n_dead_tup DESC;
```
```
  relname  | n_live_tup | n_dead_tup | dead_pct |        last_autovacuum
-----------+------------+------------+----------+-------------------------------
 job_runs  |    2104553 |    1876201 |     47.1 | 2026-09-19 02:11:04     <- losing
 sessions  |     840112 |      95003 |     10.2 | 2026-09-21 08:44:19     <- fine
```
> 47% dead and last vacuumed two days ago. That table is half garbage.

### 💻 Prevent — the tuning that matters
```sql
-- Default: vacuum when dead tuples exceed 20% of the table.
-- On a 100M-row table that's 20 MILLION dead rows before it even starts. Absurd.
ALTER TABLE job_runs SET (
  autovacuum_vacuum_scale_factor  = 0.01,   -- 1% instead of 20%
  autovacuum_vacuum_threshold     = 1000,
  autovacuum_vacuum_cost_delay    = 2       -- let it work faster
);
```

### 🧠 The fix ladder
| Tool | Lock | When |
|---|---|---|
| `VACUUM` / autovacuum | None blocking | Routine. Reuses space in place |
| `REINDEX CONCURRENTLY` | Brief | Index bloat specifically |
| **`pg_repack`** | Brief at the swap | **The production answer** — rebuilds online |
| `VACUUM FULL` | **Exclusive — full outage** | Only in a maintenance window |

---

## Q3. What is XID wraparound?

### 🗣️ Say this
> "Transaction IDs are 32-bit, so they wrap around. Visibility is a modular comparison, which means a transaction ID more than about two billion in the past would suddenly look like it's in the *future* — and rows would silently disappear.
> Postgres prevents that by freezing old rows during vacuum, marking them visible to everyone forever.
> If freezing falls far enough behind, Postgres starts warning, then refuses new transactions and shuts down to protect your data. Recovery means single-user-mode vacuum, which on a big table is hours of hard downtime.
> So I monitor `age(datfrozenxid)` and alert at 500 million — that's about a week of runway, not a fire."

### 💻 Monitor this — *(what you'd see)*
```sql
SELECT datname, age(datfrozenxid) AS xid_age,
       2147483648 - age(datfrozenxid) AS remaining
FROM pg_database ORDER BY xid_age DESC;
```
```
  datname  |  xid_age  | remaining
-----------+-----------+------------
 production| 187442901 | 1960040747     <- healthy
```

### ⚠️ Follow-up: "What causes it to fall behind?"
> "An anti-wraparound vacuum can't be skipped, but it can be *blocked* — by a long-running transaction, by a table nobody ever writes to so nobody vacuums it, or by an abandoned replication slot. An orphaned slot from a replica someone decommissioned is a classic: it fills the disk and stalls freezing at the same time."

---

## Q4. What's a HOT update and why does `fillfactor` matter?

### 🗣️ Say this
> "Normally an update writes a new row version *and* a new entry in every index on the table — even indexes on columns that didn't change. That's a lot of write amplification.
> A HOT update avoids all of it. If no indexed column changed and the new version fits on the same page, Postgres chains it within the page and touches zero indexes. Much less WAL, much less index bloat, much less vacuum work.
> `fillfactor` is how you make room for that. The default is 100, meaning pages are packed completely full, so there's never space and you never get HOT updates. On an update-heavy table I set it to 85.
> The other half is not indexing columns you update constantly — an index on `last_seen_at` defeats the whole thing."

### 💻 Check whether you're getting them — *(what you'd see)*
```sql
SELECT relname, n_tup_upd, n_tup_hot_upd,
       round(100.0*n_tup_hot_upd/NULLIF(n_tup_upd,0),1) AS hot_pct
FROM pg_stat_user_tables WHERE n_tup_upd > 1000 ORDER BY n_tup_upd DESC;
```
```
  relname  | n_tup_upd | n_tup_hot_upd | hot_pct
-----------+-----------+---------------+---------
 sessions  |   4820114 |        291044 |     6.0    <- bad, tune it
 orders    |    182004 |        164112 |    90.2    <- healthy
```
> 6% means almost every update is touching every index. `ALTER TABLE sessions SET (fillfactor = 85)` and repack.

---

# Section B — Indexes

## Q5. Which index type would you pick?

### 🗣️ Say this
> "B-tree for 99% of cases — equality, ranges, sorting, uniqueness. That's the honest answer and I'd say it plainly rather than list exotic types.
> GIN when I need containment — JSONB, arrays, full-text search, or `LIKE` with a leading wildcard via trigrams.
> BRIN is the one people forget, and it's spectacular in the right situation. On a huge append-only table ordered by time, a B-tree on the timestamp is tens of gigabytes; BRIN is a few megabytes, because it only stores min and max per block range.
> The catch with BRIN is it only works if the rows are physically in key order — if they're inserted randomly it degrades to a full scan."

| Type | Use for | Don't use for |
|---|---|---|
| **B-tree** | Equality, range, ORDER BY, uniqueness — **the default** | Containment |
| **GIN** | JSONB `@>`, arrays, full-text, trigram LIKE | Frequently-updated columns — GIN writes are expensive |
| **GiST** | Geometric, ranges, nearest-neighbour | Plain scalar equality |
| **BRIN** | Huge, naturally time-ordered tables. Tiny | Randomly-ordered data |
| **Hash** | Equality only, on large values | Anything needing ordering |

### 💻 The index *features* that matter more than the type
```sql
-- Partial: index only the rows you actually query. Smaller and cheaper to maintain.
CREATE INDEX idx_jobs_pending ON jobs (created_at) WHERE status = 'pending';

-- Covering: INCLUDE extra columns so the query never touches the table at all.
CREATE INDEX idx_orders_user ON orders (user_id) INCLUDE (total, status);

-- Expression: index what you actually filter on.
CREATE INDEX idx_users_lower_email ON users (lower(email));
```

### ⚠️ Follow-up: "How do you order columns in a composite index?"
> "Equality columns first, then the range or sort column. The mental model is a phone book sorted by surname then first name — you can find all the Smiths instantly, and they're already sorted by first name. But you cannot efficiently find everyone called John, because the first column is what organises the whole thing.
> So `(tenant_id, created_at)` serves `WHERE tenant_id = X ORDER BY created_at`, and it also serves `WHERE tenant_id = X` on its own. It does *not* help `WHERE created_at > Y` alone."

---

## Q6. When does an index make things worse?

### 🗣️ Say this
> "Every write pays for every index. An insert has to update all of them, so six indexes means six tree descents and six WAL records per row.
> It also kills HOT updates if it's on a column you change often, which quietly multiplies your write cost.
> And a low-selectivity index is just wasted — an index on a boolean where 60% of rows are true will correctly be ignored by the planner, so you're paying maintenance for nothing. A *partial* index on the rare value is the right shape there.
> So I'd periodically look at `pg_stat_user_indexes` for indexes with almost no scans and drop them."

### 💻 Find unused indexes — *(what you'd see)*
```sql
SELECT relname, indexrelname, idx_scan,
       pg_size_pretty(pg_relation_size(indexrelid)) AS size
FROM pg_stat_user_indexes WHERE idx_scan < 50
ORDER BY pg_relation_size(indexrelid) DESC LIMIT 5;
```
```
 relname |      indexrelname       | idx_scan |  size
---------+-------------------------+----------+--------
 orders  | idx_orders_legacy_ref   |        0 | 2841 MB    <- 2.8GB, never used
 events  | idx_events_source_type  |       11 |  612 MB
```
⚠️ **Check the replica too** — an index that's unused on the primary may be serving every analytics query on the replica.

---

## Q7. Why would the planner ignore my index?

### 🗣️ Say this
> "The most common reason is that it's correct to ignore it — if the query returns more than about 10% of the table, a sequential scan genuinely is faster than jumping around an index. That's not a bug.
> After that, the usual cause is stale statistics. The planner guesses row counts from samples, and if those are out of date it picks a bad plan. The tell is estimated versus actual rows being off by a hundred times in `EXPLAIN ANALYZE`.
> Then there are the mechanical ones: a function wrapped around the column, so `lower(email)` needs an expression index. A leading wildcard in a LIKE. A type mismatch. Or the wrong column order in a composite index.
> The one people rarely know is correlated columns — the planner assumes columns are independent, so if city and country are related it badly underestimates and flips to a nested loop. Extended statistics fix exactly that."

### 💻 The fixes
```sql
ANALYZE orders;                                        -- refresh statistics

ALTER TABLE events ALTER COLUMN tenant_id SET STATISTICS 1000;  -- more samples
ANALYZE events;

-- Tell the planner two columns are correlated (PG10+)
CREATE STATISTICS s_events (dependencies, ndistinct) ON city, country FROM events;
```

---

## Q8. Read this `EXPLAIN ANALYZE` for me.

### 🗣️ Say this
> "I look at four things, in order.
> First, estimated versus actual rows at each node — because a plan is only as good as its estimates, and if a node says 12 rows and actually returned 480,000, everything above it is built on a lie and the real problem is statistics.
> Second, where the time actually is. I read bottom-up and find the node with the largest self time, not total time.
> Third, buffers — which is why I always run it with `(ANALYZE, BUFFERS)`. `shared read` means it went to disk, `shared hit` means it was cached.
> Fourth, loops. A nested loop running its inner node 50,000 times is basically N+1 happening inside the database."

### 💻 A real plan — *(what you'd see)*
```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT o.* FROM orders o WHERE o.created_at > now() - interval '7 days';
```
```
Seq Scan on orders o  (cost=0.00..248130.00 rows=12 width=84)
                      (actual time=0.031..3184.552 rows=482119 loops=1)
  Filter: (created_at > (now() - '7 days'::interval))
  Rows Removed by Filter: 8517881
  Buffers: shared read=148230
Planning Time: 0.214 ms
Execution Time: 3218.447 ms
```
> **How I'd read that out loud:** "It estimated 12 rows and got 482,000 — that's a 40,000× miss, so the statistics are stale. It's doing a sequential scan, throwing away 8.5 million rows with the filter, and reading 148,000 buffers from disk. Fix the statistics, add an index on `created_at`, and this becomes an index scan."

### 🧠 Node types and what each tells you
| Node | Means | Worrying when |
|---|---|---|
| `Seq Scan` | Full table read | On a big table with a selective filter |
| `Index Scan` | Index, then fetch each row | Fine — but many rows makes it worse than a seq scan |
| `Index Only Scan` | Answered from the index alone | Check `Heap Fetches:` — high means VACUUM hasn't run |
| `Bitmap Heap Scan` | Collect matches, read in disk order | Normal. `lossy` means `work_mem` was too small |
| `Nested Loop` | For each outer row, probe inner | Bad with a big outer side and no inner index |
| `Hash Join` | Build a hash of one side | Good. Watch `Batches: > 1` — that's spilling to disk |
| `Sort` | Explicit sort | `external merge Disk: 84MB` → raise `work_mem` or index it |

📐 **`work_mem` is per sort/hash node per connection — not per query.** A query with 4 hash nodes across 100 connections can use 400× `work_mem`. That's how people OOM a database by "just raising work_mem."

---

## Q9. How do you find *which* query to fix?

### 🗣️ Say this
> "`pg_stat_statements`, sorted by **total** time — not mean time. That distinction matters: a 5-millisecond query that runs two million times is far more of your load than a 3-second query that runs twice.
> I also look at the standard deviation. A query with a 2ms mean and a 900ms standard deviation has plan instability or is sensitive to its parameters — and that's what's causing your p99, not your median.
> Alongside that I set `log_min_duration_statement` so slow queries land in the log.
> And I'd enable `auto_explain`, which captures the actual *plan* of slow queries as they happen — that's the only way to catch a plan that's only bad for certain parameter values."

### 💻 *(what you'd see)*
```sql
SELECT round(total_exec_time::numeric/1000,1) AS total_s, calls,
       round(mean_exec_time::numeric,2) AS mean_ms,
       round(stddev_exec_time::numeric,2) AS stddev_ms, left(query,45)
FROM pg_stat_statements ORDER BY total_exec_time DESC LIMIT 5;
```
```
 total_s |  calls  | mean_ms | stddev_ms |                    query
---------+---------+---------+-----------+----------------------------------------
  8412.3 | 4102887 |    2.05 |    891.44 | SELECT * FROM orders WHERE user_id = $1
  2104.7 |     612 | 3439.00 |     42.10 | SELECT count(*) FROM events WHERE ...
```
> "Row one is the real problem — only 2ms on average, but four million calls, and look at that standard deviation of 891ms. That's the query causing your p99. Row two *looks* worse per call but is a rounding error in total."

---

# Section C — Operating it

## Q10. Why do you need PgBouncer?

### 🗣️ Say this
> "Postgres forks a whole operating-system process per connection. Each one has its own memory for catalogs and plans, so call it 5 to 10 megabytes at rest. Connections are genuinely expensive in a way they aren't in other databases.
> Past a few hundred, you spend more time context-switching and fighting over internal locks than doing work — throughput actually goes *down* while latency goes up.
> That collides badly with Kubernetes. Twenty pods with a pool of twenty each is 400 connections from one service, and you have several services.
> PgBouncer in transaction mode multiplexes thousands of client connections onto a few dozen real ones, assigned only for the duration of a transaction. On RDS the managed equivalent is RDS Proxy."

### 💻 The bug you'll hit — asyncpg behind PgBouncer
```python
engine = create_async_engine(
    DB_URL,
    poolclass=NullPool,                    # PgBouncer IS the pool - don't pool twice
    connect_args={"statement_cache_size": 0},   # <- without this it breaks
)
```
```
Without statement_cache_size=0 you get, intermittently:

  asyncpg.exceptions.InvalidSQLStatementNameError:
  prepared statement "__asyncpg_stmt_4__" does not exist
```
> "That's the number one PgBouncer bug. asyncpg uses server-side prepared statements by default, and in transaction mode your next query lands on a different backend that's never heard of it."

### ⚠️ What else breaks in transaction mode
| Feature | Works? |
|---|---|
| Server-side prepared statements | ❌ — the bug above |
| `LISTEN` / `NOTIFY` | ❌ session-scoped |
| Session-level `SET` | ❌ use `SET LOCAL` |
| Session advisory locks | ❌ use `pg_advisory_xact_lock` |
| Temp tables | ❌ |

---

## Q11. Replication — physical vs logical, sync vs async.

### 🗣️ Say this
> "Physical replication ships the write-ahead log byte for byte, so the replica is an identical read-only copy. That's what you use for high availability and read replicas.
> Logical replication decodes the actual row changes, so the replica is a separate database that can have a different schema or major version. That's what you use for zero-downtime major upgrades and change-data-capture.
> On sync versus async: async means commit returns as soon as the primary has written locally — fast, but if the primary dies you lose whatever hadn't shipped. Synchronous means commit waits for the standby, so you lose nothing, but you pay a network round trip on every single commit.
> The trap with synchronous is availability coupling — if you have exactly one sync standby and it goes down, writes on the primary hang. You need a quorum setting, or you've built a system where losing a replica takes down the primary."

### ⚠️ The footgun to name: replication slots
> "A slot guarantees the primary keeps WAL until the consumer has read it. Great for correctness, lethal operationally — a replica that was decommissioned without dropping its slot means the primary retains WAL forever and fills its disk. I'd monitor for inactive slots as a standing alert."

```sql
SELECT slot_name, active,
       pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS retained
FROM pg_replication_slots ORDER BY 3 DESC;
```
```
     slot_name      | active | retained
--------------------+--------+-----------
 replica_old_eu1    | f      | 284 GB       <- inactive slot eating your disk
 replica_eu2        | t      | 16 MB
```

---

## Q12. Partitioning — when, and what breaks?

### 🗣️ Say this
> "The honest reason to partition is almost never query speed — a good index handles that. It's **maintenance**.
> Dropping a partition is instant metadata. Running `DELETE FROM events WHERE created_at < ...` on 200 million rows creates 200 million dead tuples and days of vacuum work. So time-based retention is the killer use case.
> The secondary wins are that autovacuum works per partition so it stays manageable, and pruning means each index scan touches a much smaller index.
> What breaks: the partition key has to be in every unique constraint, so your primary key becomes a composite. Pruning only happens if the key is in the WHERE clause. And you have to create future partitions or inserts fail at midnight on the first of the month."

### 💻 Retention becomes instant
```sql
CREATE TABLE events (
  id bigserial, tenant_id uuid, created_at timestamptz NOT NULL, payload jsonb
) PARTITION BY RANGE (created_at);

CREATE TABLE events_2026_09 PARTITION OF events
  FOR VALUES FROM ('2026-09-01') TO ('2026-10-01');

-- Instead of a DELETE that takes hours and bloats the table:
ALTER TABLE events DETACH PARTITION events_2025_09 CONCURRENTLY;
DROP TABLE events_2025_09;        -- instant
```
⚠️ **Verify pruning is happening** — check `EXPLAIN` for `Partitions removed: N`. A query without the partition key scans every partition.

---

## Q13. Zero-downtime schema changes.

### 🗣️ Say this
> "The principle is that during a rolling deploy both the old and new version of your code are running at once, so the schema has to work with both. That forces expand-migrate-contract across several releases.
> Beyond that it's about knowing which DDL is safe. Adding a nullable column is instant metadata. Adding a column with a default is safe since Postgres 11. Changing a column's type rewrites the whole table under an exclusive lock, so that has to be done as a new column plus backfill plus swap.
> `CREATE INDEX CONCURRENTLY` is the safe way to add an index — two table scans but no write blocking. The catch is it can't run inside a transaction, which matters for Alembic.
> And the thing that actually causes the outage isn't the DDL itself — it's the lock queue."

### 💻 The lock queue trap — the best thing you can say on this topic
```
Time  Session                          What's happening
----  -------------------------------  ---------------------------------------
 t0   SELECT ... (slow, 5 minutes)     Holds a normal read lock. Fine.
 t1   ALTER TABLE orders ADD COLUMN    Needs ACCESS EXCLUSIVE -> QUEUES behind it
 t2   SELECT ... (normal, fast)        Queues behind the ALTER!
 t3   SELECT ... (normal, fast)        Queues!
 t4   ...every query on this table     TOTAL OUTAGE
```
> "Postgres locks are FIFO. One slow read plus one quick migration equals a complete outage on that table, and the migration hadn't even started. The mitigation is always setting `lock_timeout` so the DDL gives up rather than building a queue behind it."

```sql
SET lock_timeout = '3s';          -- give up rather than block everything
SET statement_timeout = '30s';
ALTER TABLE orders ADD COLUMN currency text;
```

### 🧠 The safety table
| Operation | Safe? |
|---|---|
| `ADD COLUMN` (nullable) | ✅ Instant, metadata only |
| `ADD COLUMN ... DEFAULT x NOT NULL` | ✅ **Since PG11** — no rewrite |
| `DROP COLUMN` | ✅ Instant (space reclaimed later by vacuum) |
| `ADD CONSTRAINT ... NOT VALID` then `VALIDATE` | ✅ Two steps, second takes a weak lock |
| `CREATE INDEX CONCURRENTLY` | ✅ No write block. **Can't run in a transaction** |
| `ALTER COLUMN TYPE` | ❌ **Full rewrite + exclusive lock** |
| `SET NOT NULL` | ⚠️ Scans under exclusive lock (PG12+ can skip it with a validated CHECK) |
| `RENAME COLUMN` | ❌ Breaks the running old version instantly |

```python
# Alembic: CONCURRENTLY needs autocommit
def upgrade():
    with op.get_context().autocommit_block():
        op.create_index("ix_orders_created", "orders", ["created_at"],
                        postgresql_concurrently=True)
```
⚠️ **It can fail and leave an INVALID index behind** — it doesn't clean up after itself, so your migration must check `pg_index.indisvalid` and drop before retrying.

---

## Q14. JSONB — when is it right?

### 🗣️ Say this
> "JSONB is excellent and it's overused. It's right when the data is genuinely schemaless — third-party webhook payloads, user-defined custom fields, event envelopes whose shape varies by type.
> It's wrong as a way to avoid designing a schema, because you lose type checking, foreign keys, and good column statistics — the planner estimates JSONB predicates poorly.
> My rule: if you find yourself filtering or joining on a key regularly, that key wants to be a real column. And since Postgres 12 you can have both, with a generated column.
> One cost worth knowing — a JSONB value over about 2KB gets TOASTed out to a side table, and updating *any* field rewrites the whole document. On a big, frequently-updated JSONB column that's serious write amplification."

### 💻 Indexing it
```sql
CREATE INDEX idx_payload ON events USING GIN (payload);          -- containment
SELECT * FROM events WHERE payload @> '{"type": "order.created"}';

CREATE INDEX idx_type ON events ((payload->>'type'));            -- one hot key: B-tree

-- Promote it properly (PG12+) — best of both
ALTER TABLE events ADD COLUMN event_type text
  GENERATED ALWAYS AS (payload->>'type') STORED;
CREATE INDEX ON events (event_type);
```

---

## Q15. What's your backup and recovery story?

### 🗣️ Say this
> "`pg_dump` is a logical backup — portable and restorable to a different version, but restoring a large database takes hours and it's a single point in time.
> For production I want physical base backups plus WAL archiving, which gives point-in-time recovery — restore the base and replay the log to any moment. So you can recover to the second *before* someone ran a DELETE without a WHERE clause. Tools are pgBackRest or wal-g; on RDS it's the automated backup window.
> But the thing I'd actually say in an interview is that a backup you've never restored isn't a backup.
> RTO and RPO are the only numbers that matter, and you only know them by running the restore. I'd want a scheduled restore test, monthly at minimum, that measures wall-clock time to a working database."

📐 **RPO** = how much data you can lose (set by your replication/archive settings). **RTO** = how long recovery takes (set by backup size and replay speed).

---

## ⚡ Rapid fire

| They ask | Say this |
|---|---|
| `text` vs `varchar(n)`? | "Identical performance. `varchar(n)` just adds a length check, and changing n needs DDL. I use `text` plus a CHECK constraint." |
| `timestamp` vs `timestamptz`? | "Always `timestamptz`. It stores UTC and converts on the way out. Plain `timestamp` silently throws away the offset." |
| UUID v4 as a primary key? | "It works but it's random, so inserts land all over the index and cause page splits. UUIDv7 or ULID is time-ordered, so inserts stay at the tidy end of the tree." |
| `SERIAL` vs `IDENTITY`? | "`GENERATED ALWAYS AS IDENTITY` is the SQL-standard one and avoids the sequence-ownership quirks of SERIAL." |
| NULLs in a unique index? | "NULLs count as distinct by default, so you can have many. PG15 added `UNIQUE NULLS NOT DISTINCT` if you want the other behaviour." |
| `EXISTS` vs `IN`? | "Usually planned the same now. `NOT IN` is the dangerous one — a single NULL in the subquery makes it return nothing at all." |
| `shared_buffers` sizing? | "About 25% of RAM as a starting point. The OS page cache does the rest, and going much higher rarely helps." |
| Kill a runaway query? | "`pg_cancel_backend(pid)` first — that's polite. `pg_terminate_backend(pid)` if it won't stop, which drops the whole connection." |
| What's `ctid`? | "The physical location — page and offset. It changes on every update. Handy for de-duplication, never store it." |
| `COPY` vs `INSERT`? | "`COPY` is an order of magnitude faster for bulk loading — one parse, minimal per-row overhead." |
| Advisory locks? | "Application-defined locks by integer key. Use `pg_advisory_xact_lock`, the transaction-scoped one — it releases automatically." |

---

**Next:** [04-redis-and-caching.md](04-redis-and-caching.md)

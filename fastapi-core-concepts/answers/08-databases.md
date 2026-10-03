# 08 — Databases

---

## 📖 Vocabulary

> *Definition* is the precise meaning; *Say it in an interview* is the quotable one-liner.

| Term | Definition | Say it in an interview |
|---|---|---|
| **ORM** | *Object-Relational Mapper*: a library mapping database tables to classes and rows to objects, translating object operations into SQL. | "An ORM lets me work with rows as objects and composes SQL safely." |
| **SQLAlchemy Core vs ORM** | Core: SQL expression language, engine, pooling. ORM: class↔table mapping, sessions, relationships, identity map, built on Core. | "Core builds SQL; ORM adds object mapping and the unit of work." |
| **Engine** | SQLAlchemy's factory for connections; encapsulates the dialect, driver and **connection pool**. One per process per database. | "The engine owns the pool; create it once per process." |
| **Connection** | An established, authenticated network session with the database server (TCP/TLS + auth, and in Postgres a server backend process). | "Connections are expensive on both sides, which is why we pool them." |
| **Connection pool** | A cache of open connections checked out for use and returned afterward. Parameters: `pool_size`, `max_overflow`, `pool_timeout`, `pool_recycle`, `pool_pre_ping`. | "The pool bounds concurrent DB connections and avoids reconnect cost." |
| **PgBouncer / RDS Proxy** | External connection poolers multiplexing many client connections onto fewer server connections. | "An external pooler protects Postgres when I have many app instances." |
| **Session** | SQLAlchemy's unit-of-work object: tracks loaded/changed objects, manages a transaction, flushes changes as SQL. Short-lived; one per request. | "A session is a per-request unit of work, not a connection." |
| **Unit of Work** | Pattern that tracks changes to objects during a business transaction and commits them together. | "Unit of work batches all changes for one use case into one transaction." |
| **Transaction** | A sequence of operations executed as a single atomic unit that either fully commits or fully rolls back. | "A transaction is all-or-nothing." |
| **ACID** | Atomicity (all or nothing), Consistency (constraints preserved), Isolation (concurrent transactions don't interfere), Durability (committed data survives failure). | "ACID is the guarantee set that makes relational databases trustworthy." |
| **Commit / Rollback** | Commit makes a transaction's changes permanent and visible; rollback discards them and releases locks. | "Commit on success, rollback on any exception." |
| **Flush** | Emitting pending INSERT/UPDATE/DELETE statements to the database within the current transaction without committing. | "Flush sends SQL so constraint errors surface early; commit makes it permanent." |
| **Isolation level** | The degree to which concurrent transactions are shielded from each other (Read Committed default in Postgres; Repeatable Read; Serializable). | "Postgres defaults to Read Committed; I raise it only when the logic needs it." |
| **`SELECT … FOR UPDATE`** | Row-level lock acquired on selected rows for the transaction's duration, blocking other writers to prevent lost updates. | "`FOR UPDATE` serializes access to the rows I'm about to change." |
| **Optimistic concurrency** | Detecting conflicting concurrent updates with a version column instead of locks; the update fails if the version changed. | "Optimistic locking uses a version column and retries on conflict." |
| **Deadlock** | Two transactions each waiting for a lock the other holds; the DB aborts one. Avoided by consistent lock ordering and short transactions. | "Deadlocks come from inconsistent lock ordering; Postgres kills one side." |
| **Migration** | A versioned, reversible script that alters the database schema (or data) from one known state to the next. | "Migrations version the schema like code." |
| **Alembic** | SQLAlchemy's migration tool: generates and applies ordered revision scripts, tracking the current version in `alembic_version`. | "Alembic is how I evolve the schema safely and reproducibly." |
| **DDL / DML** | Data Definition Language (`CREATE`, `ALTER`, `DROP`) vs Data Manipulation Language (`SELECT`, `INSERT`, `UPDATE`, `DELETE`). | "DDL changes structure; DML changes data." |
| **Expand–migrate–contract** | Zero-downtime schema change pattern: add new structures (expand), migrate code/data, then remove old structures (contract), keeping both app versions compatible throughout. | "Expand/contract keeps old and new code working during a rolling deploy." |
| **Index** | An auxiliary data structure (usually B-tree) enabling fast lookup by column values at the cost of write overhead and storage. | "Indexes trade write cost for read speed on filtered and joined columns." |
| **Composite / partial / covering index** | Composite: multiple columns, order matters. Partial: only rows matching a predicate. Covering: includes extra columns so the query needs no table access. | "Composite for multi-column filters, partial for hot subsets, covering to avoid heap lookups." |
| **Sequential scan** | Reading every row of a table because no suitable index exists or the planner judges it cheaper. | "A seq scan on a big table in EXPLAIN usually means a missing index." |
| **`EXPLAIN ANALYZE`** | Postgres command that executes a query and reports the actual plan, row counts and timings per node. | "EXPLAIN ANALYZE shows the real plan and where time goes." |
| **N+1 query problem** | Loading N parent rows then issuing one additional query per parent for related rows, causing N+1 round trips. | "N+1 is fixed with eager loading: `selectinload` or `joinedload`." |
| **Eager vs lazy loading** | Eager: related rows fetched up front (joined or batched IN query). Lazy: fetched on attribute access (causes N+1; unsupported in async without explicit loading). | "In async SQLAlchemy, lazy loading fails, so I eager-load relationships." |
| **Pagination** | Returning a large result set in bounded pages. | "Never return unbounded lists; paginate." |
| **Offset pagination** | `LIMIT n OFFSET m`: simple, supports page numbers, but cost grows with offset and results shift as data changes. | "Offset is O(offset) and unstable under inserts." |
| **Cursor / keyset pagination** | Pages defined by a key from the last row (`WHERE id < :cursor ORDER BY id DESC LIMIT n`): O(log n) via index, stable, no random page access. | "Keyset pagination seeks by index and stays consistent." |
| **SQL injection** | Attack where untrusted input alters the structure of an SQL statement because it was concatenated into the query text. | "Parameterized queries make injection impossible by design." |
| **Parameterized query** | SQL with placeholders where values are sent separately from the statement text, so they are never parsed as SQL. | "The ORM always sends values as parameters, never as text." |
| **Read replica** | A copy of the database receiving changes from the primary via replication; serves read-only queries to offload the primary. | "Replicas scale reads; writes still go to the primary." |
| **Replication lag** | Delay between a commit on the primary and its visibility on a replica. | "After a write, a replica read might be stale for a moment." |
| **Failover** | Promoting a standby/replica to primary after the primary fails, and repointing clients (usually via a stable endpoint). | "Failover swaps the primary; my app must reconnect gracefully." |
| **Repository pattern** | A data-access abstraction exposing intent-named methods that hide query construction from callers. | "Repositories keep SQL out of services and centralize query tuning." |
| **Service layer** | Classes implementing use cases, coordinating repositories and owning transaction boundaries; independent of HTTP. | "Services hold business rules and own the transaction." |
| **`yield_per` / server-side cursor** | Streaming rows from the database in batches rather than materializing the whole result set in memory. | "`yield_per` streams big results in batches." |
| **NDJSON** | Newline-delimited JSON: one JSON document per line; streamable and incrementally parseable. | "NDJSON lets me stream a million records without buffering." |

---

## Q1. How do you connect FastAPI to PostgreSQL?

### 💻 Code (async stack: SQLAlchemy 2.0 + asyncpg)
```python
# app/db/session.py
from collections.abc import AsyncIterator
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker, AsyncSession

engine = create_async_engine(
    settings.database_url,          # "postgresql+asyncpg://user:pass@host:5432/db"
    pool_size=10,
    max_overflow=20,
    pool_timeout=30,
    pool_recycle=1800,
    pool_pre_ping=True,
    echo=settings.debug,
)

SessionLocal = async_sessionmaker(engine, expire_on_commit=False)

async def get_db() -> AsyncIterator[AsyncSession]:
    async with SessionLocal() as session:
        yield session
```
### 🔍 Line by line
- `create_async_engine(url, ...)` — builds the engine. The URL's `postgresql+asyncpg` part picks the async driver.
- `pool_size=10` — keep 10 connections open.
- `max_overflow=20` — allow up to 20 extra under burst, closed when returned.
- `pool_timeout=30` — wait up to 30 s for a free connection before raising.
- `pool_recycle=1800` — replace connections older than 30 min (dodges server idle timeouts / stale NAT entries).
- `pool_pre_ping=True` — run a tiny check before handing out a connection; avoids "server closed the connection" after DB restarts.
- `echo=settings.debug` — log SQL in debug mode.
- `async_sessionmaker(engine, expire_on_commit=False)` — a factory for sessions. `expire_on_commit=False` keeps loaded objects usable after commit (otherwise SQLAlchemy would try to lazy-reload them, which fails in async).
- `get_db` — per-request session dependency (file 04).

---

## Q2. What ORM have you used?

Be concrete: "SQLAlchemy 2.0 with the async engine and asyncpg in production; Alembic for migrations; SQLModel on smaller projects; Django ORM previously." Then expect the follow-ups below.

## Q3. What is SQLAlchemy?

The standard Python SQL toolkit, in two layers:
- **Core** — SQL expression language, pooling, dialects. Queries are Python objects: `select(users).where(...)`.
- **ORM** — maps classes to tables, tracks object state in a **Session**, manages relationships.

2.0 style uses typed `Mapped[...]` columns and `select()` everywhere:
```python
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, relationship

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "users"
    id: Mapped[int] = mapped_column(primary_key=True)
    email: Mapped[str] = mapped_column(unique=True, index=True)
    orders: Mapped[list["Order"]] = relationship(back_populates="user")
```
- `Mapped[int]` — typed column. `mapped_column(primary_key=True)` — column options.
- `relationship(back_populates="user")` — a link to `Order` rows; `"Order"` as a string avoids import cycles.

## Q4. What is SQLModel?

A library by FastAPI's author that fuses **Pydantic + SQLAlchemy**: one class is both a validated schema and an ORM model.
```python
from sqlmodel import SQLModel, Field
class Hero(SQLModel, table=True):
    id: int | None = Field(default=None, primary_key=True)
    name: str
```
Pros: less duplication, great for small services. Cons: couples API schema to DB schema (which file 03 warned against), lags SQLAlchemy features, thinner async story. Default for large systems: plain SQLAlchemy + separate Pydantic schemas.

## Q5. How do you create a database session?

Via `async_sessionmaker` and a `yield` dependency (Q1). **One session per request**, never shared across requests or tasks. The session borrows a connection from the pool lazily on the first query and returns it on close.

---

## Q6. Why shouldn't you create a new database connection for every request?

A Postgres connection = TCP handshake + TLS + authentication + a **forked backend process** on the server (several ms and several MB each). At 1,000 requests/s that's 1,000 forks per second — the DB collapses long before your app. Postgres also has a hard `max_connections` (default 100). Pooling amortises the cost.

## Q7. What is connection pooling?

Keep N connections open; lend one out per request; take it back (not closed) when done. SQLAlchemy's `QueuePool` does this in-process. With many app instances add an **external pooler** (PgBouncer, RDS Proxy) so 50 pods × 10 connections don't become 500 server backends.

## Q8. How do you configure connection pool size?

- `pool_size` — steady-state connections per process.
- `max_overflow` — extra allowed under burst.
- `pool_timeout` — how long to wait for a free one.

Start with `pool_size=5–10` per worker, then measure. Total across all pods must stay under the DB's `max_connections` with headroom. Postgres rarely benefits from more than roughly `(cores × 2)` *active* connections; beyond that, add PgBouncer rather than connections.

## Q9. What happens when the database connection pool is exhausted?

New requests block in the pool for `pool_timeout` seconds, then raise `TimeoutError: QueuePool limit of size 10 overflow 20 reached`. Latency shows a *step* pattern; throughput plateaus; coroutines pile up and memory grows.

**Causes:** slow queries holding connections; sessions not closed; long transactions; `await`ing an external HTTP call *while holding a connection*.
**Fixes:** close sessions reliably; commit early; don't hold a connection during external calls; add indexes; raise pool size cautiously; add PgBouncer; alert on pool waits; set Postgres `statement_timeout`.

---

## Q10. How do you handle database transactions?

The **unit-of-work** pattern: one transaction per use case, committed at the service boundary.

```python
from sqlalchemy import select

class OrderService:
    def __init__(self, db: AsyncSession):
        self.db = db

    async def place_order(self, user_id: int, items: list[LineItem]) -> Order:
        async with self.db.begin():
            order = Order(user_id=user_id)
            self.db.add(order)
            for li in items:
                stock = await self.db.scalar(
                    select(Product).where(Product.id == li.sku).with_for_update()
                )
                if stock.qty < li.qty:
                    raise InsufficientStock(li.sku)
                stock.qty -= li.qty
                order.lines.append(OrderLine(product=stock, qty=li.qty))
        return order
```
### 🔍 Line by line
- `async with self.db.begin():` — start a transaction; **commit** when the block exits normally, **rollback** if an exception escapes.
- `self.db.add(order)` — register the new object with the session (INSERT happens at flush).
- `select(Product).where(...).with_for_update()` — `SELECT … FOR UPDATE` locks the row so two concurrent orders can't both decrement past zero.
- `raise InsufficientStock(...)` — escapes the block → everything rolls back; no half-placed order.
- `stock.qty -= li.qty` — change tracked by the session; becomes an UPDATE at flush.
- After the block the transaction is committed and `order.id` is populated.

## Q11. What is commit vs rollback?

- **Commit** — make all changes in the transaction permanent and visible to others, atomically.
- **Rollback** — discard everything since the transaction began; also releases locks.

## Q12. How do you handle database exceptions?

Translate driver exceptions into domain errors at the repository boundary; never leak SQL to clients.
```python
from sqlalchemy.exc import IntegrityError

async def create_user(self, email: str) -> User:
    user = User(email=email)
    self.db.add(user)
    try:
        await self.db.flush()
    except IntegrityError as e:
        await self.db.rollback()
        if "users_email_key" in str(e.orig):
            raise Conflict("email already registered") from e
        raise
    return user
```
- `flush()` — send the INSERT now so constraint errors surface here, not at commit.
- `e.orig` — the underlying driver error; inspect the constraint name to produce a specific 409.
- `raise ... from e` — keep the original as the cause for debugging.
- Global handler: `OperationalError` (connection lost) → 503 with `Retry-After`; log details server-side only. Retry only *idempotent reads* on transient errors.

---

## Q13. How do you perform migrations?

With **Alembic**:
```bash
alembic init migrations                                   # once
alembic revision --autogenerate -m "add orders table"     # compare models vs DB → draft script
# REVIEW the script: autogenerate misses renames, data migrations, some constraints
alembic upgrade head                                      # apply all pending
alembic downgrade -1                                      # revert one
```
Run migrations as a **separate CI/CD step or Kubernetes Job** before rolling out new pods — not at app startup (multiple pods would race; app startup shouldn't need DDL rights).

## Q14. What is Alembic?

SQLAlchemy's migration tool. Keeps a chain of versioned scripts with `upgrade()`/`downgrade()`, and records the current version in an `alembic_version` table.
```python
def upgrade():
    op.add_column("users", sa.Column("phone", sa.String(20), nullable=True))
    op.create_index("ix_users_phone", "users", ["phone"])

def downgrade():
    op.drop_index("ix_users_phone")
    op.drop_column("users", "phone")
```

## Q15. How do you handle database schema changes without downtime?

> 🔴 **Moved to the advanced section** → [../../advanced-backend/03-postgres-deep-dive.md](../../advanced-backend/03-postgres-deep-dive.md)
> 
> Q13 — zero-downtime DDL, the full Postgres lock-compatibility table, and the lock-queue trap.

---

## Q16. What is the N+1 query problem?

> 🔴 **Moved to the advanced section** → [../../advanced-backend/02-databases-advanced.md](../../advanced-backend/02-databases-advanced.md)
> 
> Q5 — N+1 detection (`lazy="raise"`, query budgets in CI) and the full strategy comparison.

---

## Q17. How do you optimize slow database queries?

> 🔴 **Moved to the advanced section** → [../../advanced-backend/03-postgres-deep-dive.md](../../advanced-backend/03-postgres-deep-dive.md)
> 
> Q8–Q9 — reading `EXPLAIN ANALYZE`, `pg_stat_statements`, and why the planner ignores your index.

---

## Q18. How do you implement pagination?

```python
@router.get("/orders", response_model=Page[OrderRead])
async def list_orders(
    db: DB, user: CurrentUser,
    limit: Annotated[int, Query(ge=1, le=100)] = 50,
    cursor: str | None = None,
):
    stmt = select(Order).where(Order.user_id == user.id).order_by(Order.id.desc()).limit(limit + 1)
    if cursor:
        stmt = stmt.where(Order.id < decode_cursor(cursor))
    rows = (await db.scalars(stmt)).all()
    has_more = len(rows) > limit
    rows = rows[:limit]
    return Page(items=rows, next_cursor=encode_cursor(rows[-1].id) if has_more else None)
```
### 🔍 Line by line
- `.limit(limit + 1)` — fetch one extra to learn whether another page exists without a COUNT.
- `Order.id < decode_cursor(cursor)` — cursor = last seen ID: "rows before this".
- `has_more = len(rows) > limit` → then trim to `limit`.
- `encode_cursor(rows[-1].id)` — opaque token (e.g., base64 of the ID). Clients pass it back unchanged.

## Q19. Offset pagination vs cursor pagination?

> 🔴 **Moved to the advanced section** → [../../advanced-backend/02-databases-advanced.md](../../advanced-backend/02-databases-advanced.md)
> 
> Q7 — keyset pagination, the cursor implementation, and when offset is a bug.

---

## Q20. How do you prevent SQL injection?

- **Parameterized queries always.** SQLAlchemy does this: `select(User).where(User.email == email)` sends `$1` and the value separately.
- Raw SQL: `text("SELECT * FROM users WHERE email = :email").bindparams(email=email)` — never f-strings.
- **Identifiers** (table/column names) can't be parameterized — allow-list them: `SORTABLE = {"created": Order.created_at, "total": Order.total}`.
- Least-privilege DB user.
- Pydantic types (an `int` param can't carry `' OR 1=1`).

---

## Q21. How do you implement database health checks?

```python
import asyncio
from sqlalchemy import text

@router.get("/health/ready", include_in_schema=False)
async def ready(db: DB):
    try:
        async with asyncio.timeout(2):
            await db.execute(text("SELECT 1"))
    except Exception:
        raise HTTPException(503, "database unavailable")
    return {"status": "ready"}
```
- `asyncio.timeout(2)` — bound the check so a hung DB doesn't hang the probe.
- `SELECT 1` — cheapest round trip proving pool + network + auth work.
- 503 → Kubernetes stops routing traffic (readiness) without restarting the pod.
- Keep `/health/live` DB-free — a DB outage must not restart every pod.

## Q22. How would you handle read replicas?

> 🔴 **Moved to the advanced section** → [../../advanced-backend/02-databases-advanced.md](../../advanced-backend/02-databases-advanced.md)
> 
> Q9 — routing, replication lag, and the read-your-writes problem.

---

## Q23. How would you handle database failover?

> 🔴 **Moved to the advanced section** → [../../advanced-backend/03-postgres-deep-dive.md](../../advanced-backend/03-postgres-deep-dive.md)
> 
> Q11 — streaming vs logical replication, sync vs async, and replication slots.

---

## Q24. How would you optimize an API returning 1 million records?

> 🔴 **Moved to the advanced section** → [../../advanced-backend/02-databases-advanced.md](../../advanced-backend/02-databases-advanced.md)
> 
> Q8 — streaming, `yield_per`, and why this should be an async job.

---

## Q25. How would you design a repository/service layer?

```
Routers      → HTTP: parse, auth, status codes
Services     → use cases, transactions, business rules
Repositories → data access with intent-revealing methods
ORM models   → entities
```
```python
# app/repositories/order_repo.py
class OrderRepository:
    def __init__(self, db: AsyncSession):
        self.db = db

    async def get(self, order_id: int) -> Order | None:
        return await self.db.get(Order, order_id, options=[selectinload(Order.lines)])

    async def list_for_user(self, user_id: int, limit: int, before_id: int | None) -> list[Order]:
        stmt = select(Order).where(Order.user_id == user_id).order_by(Order.id.desc()).limit(limit)
        if before_id:
            stmt = stmt.where(Order.id < before_id)
        return list(await self.db.scalars(stmt))

    def add(self, order: Order) -> None:
        self.db.add(order)

# app/services/order_service.py
class OrderService:
    def __init__(self, orders: OrderRepository, products: ProductRepository, events: EventPublisher, db: AsyncSession):
        self.orders, self.products, self.events, self.db = orders, products, events, db

    async def place(self, user_id: int, items: list[LineItem]) -> Order:
        async with self.db.begin():
            order = Order(user_id=user_id)
            for li in items:
                p = await self.products.lock_for_update(li.sku)
                if p.qty < li.qty:
                    raise InsufficientStock(li.sku)
                p.qty -= li.qty
                order.lines.append(OrderLine(product_id=p.id, qty=li.qty, price=p.price))
            self.orders.add(order)
        await self.events.publish("order.placed", {"order_id": order.id})
        return order

# app/api/deps.py
def get_order_service(db: DB) -> OrderService:
    return OrderService(OrderRepository(db), ProductRepository(db), get_publisher(), db)

# app/api/v1/orders.py
@router.post("", status_code=201, response_model=OrderRead)
async def place_order(body: OrderCreate, user: CurrentUser,
                      svc: Annotated[OrderService, Depends(get_order_service)]):
    return await svc.place(user.id, body.items)
```
**Why:** routers stay thin and HTTP-only; services are unit-testable with fake repositories; repositories concentrate query tuning; the same service works from a CLI, a queue consumer, or gRPC. Keep repositories thin — SQLAlchemy already *is* a query DSL; don't build another on top.

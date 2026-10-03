# 12 — Testing

---

## 📖 Vocabulary

> *Definition* is the precise meaning; *Say it in an interview* is the quotable one-liner.

| Term | Definition | Say it in an interview |
|---|---|---|
| **Unit test** | A test of a single unit (function/class) in isolation with collaborators replaced by test doubles; fast and deterministic. | "Unit tests check logic in isolation in milliseconds." |
| **Integration test** | A test exercising multiple real components together (app + database + cache) to verify their interaction. | "Integration tests catch wiring, SQL and serialization bugs." |
| **End-to-end (E2E) test** | A test driving the fully deployed system through its external interface as a user would. | "E2E tests are few and cover critical journeys." |
| **Smoke test** | A minimal post-deployment check confirming the system's basic functions work. | "Smoke tests answer 'did the deploy break the basics'." |
| **Test pyramid** | Guideline: many unit tests, fewer integration tests, fewest E2E tests, balancing speed, cost and confidence. | "The pyramid: lots of fast unit tests, fewer slower ones above." |
| **pytest** | Python test framework using plain `assert`, automatic discovery, fixtures, parametrization and a plugin ecosystem. | "pytest with fixtures and parametrize is the standard." |
| **Fixture** | A pytest function providing setup (and optional teardown via `yield`) for tests that request it by parameter name; scoped to function, module or session. | "Fixtures provide reusable setup and cleanup." |
| **Parametrize** | `@pytest.mark.parametrize` runs one test function multiple times with different argument sets. | "Parametrize covers edge cases without duplicating tests." |
| **Assertion** | A statement that a condition holds; failure raises `AssertionError` and fails the test. | "Assert the contract, not implementation details." |
| **Test double** | Generic term for any stand-in: **stub** (canned responses), **mock** (records/asserts calls), **fake** (working lightweight implementation), **spy**, **dummy**. | "I prefer fakes for repositories and mocks only at external boundaries." |
| **`TestClient`** | Starlette's synchronous in-process client (httpx-based) that calls the ASGI app without a network socket. | "TestClient exercises the whole ASGI pipeline in-process." |
| **`ASGITransport`** | An httpx transport routing requests directly into an ASGI app, used with `AsyncClient` for async tests. | "`AsyncClient` with `ASGITransport` is the async equivalent of TestClient." |
| **Dependency override** | `app.dependency_overrides[dep] = fake` swaps a dependency provider for tests. | "Overrides replace auth or DB dependencies without touching handlers." |
| **`respx`** | A library that mocks `httpx` requests at the transport layer with route matching and canned responses/side effects. | "respx stubs outbound httpx calls, including failures." |
| **testcontainers** | A library starting real services (Postgres, Redis) in Docker for the duration of a test session. | "testcontainers gives me a real database in CI." |
| **Transaction-rollback isolation** | Wrapping each test in a database transaction that is rolled back afterward so tests leave no state. | "Each test runs in a transaction I roll back, so no cleanup." |
| **Savepoint** | A named point within a transaction to which you can roll back; lets app `commit()` calls become savepoint releases inside the outer test transaction. | "Savepoints let the app 'commit' inside my test transaction." |
| **`pytest-asyncio` / AnyIO plugin** | Plugins allowing `async def` tests to run on an event loop. | "`asyncio_mode=auto` makes async tests just work." |
| **`freezegun`** | A library that freezes or sets the current time in tests for determinism. | "Freeze time to test expiry logic deterministically." |
| **`moto`** | A library mocking AWS services locally for boto3/aiobotocore tests. | "moto fakes S3 and SQS in tests." |
| **Race condition** | A defect whose occurrence depends on the relative timing of concurrent operations. | "Race tests run the operation concurrently and assert the invariant." |
| **Invariant** | A condition that must always hold (stock never negative; exactly N successes). | "I test concurrency by asserting invariants, not timings." |
| **Coverage** | The fraction of code lines/branches executed by tests; measured with `pytest-cov`/`coverage.py`. | "Coverage finds untested code; it doesn't prove correctness." |
| **Branch coverage** | Coverage that tracks whether each conditional branch (both outcomes) was executed. | "Branch coverage catches untested `else` paths." |
| **JUnit XML** | A standard test-result format consumed by CI systems to display pass/fail per test. | "JUnit XML makes failures visible in the merge request." |
| **CI service container** | An auxiliary container (Postgres, Redis) that the CI runner starts alongside the job, reachable by hostname. | "GitLab `services:` gives the test job a real Postgres." |
| **`pytest-xdist`** | Plugin running tests in parallel across processes (`-n auto`). | "xdist parallelizes the suite across cores." |
| **`caplog`** | A pytest fixture capturing log records emitted during a test. | "caplog lets me assert that an error was logged." |
| **`raise_app_exceptions=False`** | `ASGITransport` option so unhandled app exceptions become 500 responses instead of propagating into the test. | "Set it to test the 500 handler." |

---

## Q1. How do you test FastAPI applications?

Three tiers:
1. **Unit** — services/validators with fake repositories; no HTTP, no DB. Fast, most numerous.
2. **API/integration** — `httpx.AsyncClient` against the ASGI app with dependency overrides; real DB in a container or per-test transaction.
3. **E2E/smoke** — deployed environment, a few critical paths.

Tools: `pytest`, `pytest-asyncio`, `httpx`, `factory_boy`/`polyfactory` (test data), `respx` (HTTP mocking), `testcontainers`, `pytest-cov`.

## Q2. What is pytest?

Python's standard test runner: discovers `test_*.py` files and `test_*` functions, uses plain `assert` with rich failure messages, and provides **fixtures**, **parametrize**, markers and plugins.
```python
import pytest

@pytest.fixture
def user() -> User:
    return User(id=1, email="a@b.com")

@pytest.mark.parametrize("qty,valid", [(1, True), (0, False), (-1, False)])
def test_qty(qty, valid):
    if valid:
        assert LineItem(sku="x", qty=qty).qty == qty
    else:
        with pytest.raises(ValidationError):
            LineItem(sku="x", qty=qty)
```
- `@pytest.fixture` — tests that declare a `user` parameter receive this object.
- `@pytest.mark.parametrize("qty,valid", [...])` — the test runs three times with different values.
- `pytest.raises(ValidationError)` — asserts that the block raises that exception.

## Q3. What is TestClient?

`fastapi.testclient.TestClient` (from Starlette, built on `httpx`) calls the app **in-process** — no Uvicorn, no socket. It's synchronous and runs the app's event loop in a thread; used as a context manager it also triggers `lifespan`. For async tests use `httpx.AsyncClient(transport=ASGITransport(app=app))` so you can `await` alongside DB fixtures:
```python
# tests/conftest.py
import pytest
from httpx import AsyncClient, ASGITransport
from app.main import app

@pytest.fixture
async def client():
    async with AsyncClient(transport=ASGITransport(app=app), base_url="http://test") as c:
        yield c
```
- `ASGITransport(app=app)` — route requests directly into the ASGI app.
- `base_url="http://test"` — a dummy base so relative paths work.

---

## Q4. How do you test GET endpoints?

```python
async def test_get_order(client, seeded_order):
    r = await client.get(f"/api/v1/orders/{seeded_order.id}")
    assert r.status_code == 200
    body = r.json()
    assert body["id"] == seeded_order.id
    assert "hashed_password" not in body

async def test_get_order_not_found(client):
    r = await client.get("/api/v1/orders/999999")
    assert r.status_code == 404
    assert r.json()["error"]["code"] == "not_found"
```
- Fixtures seed data (`seeded_order`) and provide an authenticated client.
- Assert status first (clearest failure), then the contract, then a *negative* assertion for leaks.
- The error envelope shape is part of the contract too.

## Q5. How do you test POST endpoints?

```python
from uuid import uuid4

async def test_create_order(client, db_session, product):
    payload = {"items": [{"sku": product.sku, "qty": 2}]}
    r = await client.post("/api/v1/orders", json=payload, headers={"Idempotency-Key": str(uuid4())})
    assert r.status_code == 201
    assert r.headers["location"].endswith(f"/orders/{r.json()['id']}")
    order = await db_session.get(Order, r.json()["id"])
    assert order is not None and order.lines[0].qty == 2
    await db_session.refresh(product)
    assert product.qty == 8
```
- `json=payload` — httpx serialises and sets `Content-Type`.
- Verify **side effects** in the DB (order exists, stock decremented from 10 to 8), not just the response.

## Q6. How do you test validation failures?

```python
@pytest.mark.parametrize("payload,loc", [
    ({"items": []}, ["body", "items"]),
    ({"items": [{"sku": "x", "qty": 0}]}, ["body", "items", 0, "qty"]),
    ({"items": [{"sku": "x"}]}, ["body", "items", 0, "qty"]),
    ({}, ["body", "items"]),
])
async def test_create_order_validation(client, payload, loc):
    r = await client.post("/api/v1/orders", json=payload)
    assert r.status_code == 422
    assert any(err["loc"] == loc for err in r.json()["error"]["details"])
```
- Parametrise the edge cases; assert the `loc` so you know the *right* rule fired.

## Q7. How do you test authentication?

Two layers: test the **mechanism** with real tokens, and **override** it for everything else.
```python
async def test_login_and_access(client_no_override, user_with_password):
    r = await client_no_override.post("/auth/token",
        data={"username": user_with_password.email, "password": "s3cret!"})
    token = r.json()["access_token"]
    r = await client_no_override.get("/api/v1/me", headers={"Authorization": f"Bearer {token}"})
    assert r.status_code == 200

async def test_expired_token(client_no_override):
    token = create_access_token("1", [], minutes=-1)
    r = await client_no_override.get("/api/v1/me", headers={"Authorization": f"Bearer {token}"})
    assert r.status_code == 401

async def test_viewer_cannot_delete(client_as_viewer):
    r = await client_as_viewer.delete("/api/v1/items/1")
    assert r.status_code == 403
```
- `data=` sends form-encoded (what the OAuth2 token endpoint expects).
- `minutes=-1` — mint an already-expired token.
- Role tests: a fixture that overrides `get_current_user` with a viewer.

---

## Q8. How do you mock external APIs?

With `respx` (intercepts `httpx` at the transport level):
```python
import respx, httpx

@respx.mock
async def test_weather(client):
    route = respx.get("https://api.weather.example/today").mock(
        return_value=httpx.Response(200, json={"temp": 21}))
    r = await client.get("/weather")
    assert r.status_code == 200 and r.json()["temp"] == 21
    assert route.called

@respx.mock
async def test_weather_timeout(client):
    respx.get("https://api.weather.example/today").mock(side_effect=httpx.ReadTimeout("slow"))
    r = await client.get("/weather")
    assert r.status_code == 503
```
- `@respx.mock` — activate interception for the test.
- `.mock(return_value=...)` — canned response; `.mock(side_effect=Exception)` — simulate failure.
- `route.called` — assert the call happened.
- The failure test is the important one: verify your error mapping / fallback.
Alternative: inject a fake client class via `dependency_overrides` when the integration sits behind a `Protocol`.

## Q9. How do you mock database dependencies?

Prefer **not** mocking the DB for integration tests — use a real Postgres with per-test transaction rollback:
```python
from testcontainers.postgres import PostgresContainer
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession

@pytest.fixture(scope="session")
async def engine():
    with PostgresContainer("postgres:16") as pg:
        eng = create_async_engine(pg.get_connection_url().replace("psycopg2", "asyncpg"))
        async with eng.begin() as conn:
            await conn.run_sync(Base.metadata.create_all)
        yield eng
        await eng.dispose()

@pytest.fixture
async def db_session(engine):
    async with engine.connect() as conn:
        trans = await conn.begin()
        session = AsyncSession(bind=conn, join_transaction_mode="create_savepoint")
        yield session
        await session.close()
        await trans.rollback()
```
### 🔍 Line by line
- `scope="session"` — start the container once for the whole test run.
- `PostgresContainer("postgres:16")` — real DB in Docker.
- `Base.metadata.create_all` — create tables (or run Alembic to also test migrations).
- `conn.begin()` — an **outer transaction** per test.
- `join_transaction_mode="create_savepoint"` — app code calling `commit()` releases a savepoint instead of really committing.
- `trans.rollback()` — undo everything → next test starts clean, no table truncation.
For **unit** tests of services, use an in-memory fake repository implementing the same `Protocol`.

## Q10. How do you override Depends() during testing?

`app.dependency_overrides[original] = replacement` (file 04 Q11). Scope it in a fixture; `clear()` afterwards. Override the **narrowest** dependency (e.g., `get_current_user`, not the router) so the rest of the pipeline is exercised.

---

## Q11. Unit tests vs integration tests?

| | Unit | Integration |
|---|---|---|
| Scope | One function/class | Several real components |
| Dependencies | Faked | Real (containers) or overridden at the edge |
| Speed | ms | 10s–100s ms |
| Finds | Logic bugs | Wiring, SQL, serialisation, config bugs |
| Count | Many | Fewer, on critical paths |

Both are needed. 100% unit coverage with untested SQL still breaks in production.

## Q12. What should be mocked?

Things **slow, non-deterministic, external, or side-effecting**: third-party APIs, payment gateways, email/SMS, the clock (`freezegun`), randomness, cloud SDKs (`moto`), message brokers (or a fake publisher).

## Q13. What shouldn't be mocked?

- Your own services/repositories in integration tests — you'd be testing the mock.
- The database for anything involving queries, constraints, transactions.
- Pydantic/FastAPI — they *are* the contract.
- Redis in tests that exercise locking/rate limits — use a container or `fakeredis`.
Rule: mock at the **boundary of your system**, not inside it.

## Q14. How do you test asynchronous endpoints?

`pytest-asyncio` with `asyncio_mode = "auto"` in `pyproject.toml` (or the `anyio` plugin) so `async def test_*` runs on a loop; `httpx.AsyncClient` with `ASGITransport`. Keep fixtures async and share the loop scope for session-scoped engines. Sync `TestClient` also works for async endpoints but can't be mixed with your own `await`s.

## Q15. How do you test exception handlers?

```python
from unittest.mock import ANY
from types import SimpleNamespace

async def test_app_error_envelope(client):
    async def boom(*_):
        raise NotFound("thing")
    app.dependency_overrides[get_order_service] = lambda: SimpleNamespace(get=boom)
    r = await client.get("/api/v1/orders/1")
    assert r.status_code == 404
    assert r.json()["error"] == {"code": "not_found", "message": "thing", "request_id": ANY}

async def test_unhandled_error_hidden(client, caplog):
    @app.get("/__boom")
    async def _():
        raise RuntimeError("secret internal detail")
    r = await client.get("/__boom")
    assert r.status_code == 500 and "secret" not in r.text
    assert "unhandled" in caplog.text
```
- `SimpleNamespace(get=boom)` — a throwaway fake service whose `get` raises.
- `ANY` — matches any value (the request ID is random).
- `caplog` — pytest fixture capturing logs.
- For the 500 case: `ASGITransport(app, raise_app_exceptions=False)` (or `TestClient(app, raise_server_exceptions=False)`) so the exception becomes a response instead of failing the test.

## Q16. How do you test middleware?

```python
async def test_request_id_echoed(client):
    r = await client.get("/health", headers={"X-Request-ID": "abc-123"})
    assert r.headers["x-request-id"] == "abc-123"

async def test_request_id_generated(client):
    r = await client.get("/health")
    uuid.UUID(r.headers["x-request-id"])          # raises if not a valid UUID

async def test_middleware_isolated():
    async def dummy(scope, receive, send):
        await send({"type": "http.response.start", "status": 200, "headers": []})
        await send({"type": "http.response.body", "body": b"ok"})
    mw_app = TimingMiddleware(dummy)
    async with AsyncClient(transport=ASGITransport(app=mw_app), base_url="http://t") as c:
        r = await c.get("/")
        assert "x-response-time-ms" in r.headers
```
- Through the app: assert the header/log the middleware adds.
- Isolated: wrap a trivial ASGI app so the test doesn't depend on routing.

## Q17. How do you test database transactions?

Force a failure mid-use-case and assert nothing persisted:
```python
from sqlalchemy import select, func

async def test_place_order_rolls_back_on_insufficient_stock(db_session, product_with_qty_1):
    svc = OrderService(OrderRepository(db_session), ProductRepository(db_session), FakePublisher(), db_session)
    with pytest.raises(InsufficientStock):
        await svc.place(user_id=1, items=[LineItem(sku="ok", qty=1), LineItem(sku="low", qty=5)])
    assert (await db_session.scalar(select(func.count()).select_from(Order))) == 0
    await db_session.refresh(product_with_qty_1)
    assert product_with_qty_1.qty == 1
```
- First item would decrement stock; second fails → whole transaction rolls back → no order, first decrement undone.

## Q18. How do you test race conditions?

Run the contended operation concurrently against a real DB and assert invariants:
```python
async def test_no_oversell(engine, product_with_qty_5):
    async def buy():
        async with AsyncSession(engine) as s:
            svc = OrderService(OrderRepository(s), ProductRepository(s), FakePublisher(), s)
            try:
                await svc.place(user_id=1, items=[LineItem(sku="p", qty=1)])
                return 1
            except InsufficientStock:
                return 0
    results = await asyncio.gather(*(buy() for _ in range(20)))
    assert sum(results) == 5
    async with AsyncSession(engine) as s:
        assert (await s.get(Product, product_with_qty_5.id)).qty == 0
```
- 20 buyers, 5 units → exactly 5 succeed. Each task needs its **own session/connection** (the rollback fixture can't be shared here; truncate after).
- Proves the `FOR UPDATE` lock / optimistic version column works. Run with `pytest-repeat` for lock-ordering deadlocks. Test idempotency-key races the same way.

## Q19. How do you measure test coverage?

```bash
pytest --cov=app --cov-report=term-missing --cov-report=xml --cov-fail-under=85
```
- `term-missing` — show uncovered lines; `xml` — for CI/Codecov/Sonar; `--cov-branch` for branch coverage.
- Treat coverage as a smell detector, not a goal; enforce a floor and ratchet it up; require coverage on changed lines in PRs (`diff-cover`).

## Q20. How would you integrate FastAPI tests into GitLab CI/CD?

```yaml
stages: [lint, test, build]

variables:
  POSTGRES_DB: test
  POSTGRES_USER: test
  POSTGRES_PASSWORD: test
  DATABASE_URL: postgresql+asyncpg://test:test@postgres:5432/test
  REDIS_URL: redis://redis:6379/0

lint:
  stage: lint
  image: python:3.12-slim
  script:
    - pip install ruff mypy
    - ruff check . && ruff format --check .
    - mypy app

test:
  stage: test
  image: python:3.12-slim
  services:
    - postgres:16
    - redis:7
  cache: { key: { files: [poetry.lock] }, paths: [.venv/] }
  before_script:
    - pip install poetry && poetry config virtualenvs.in-project true && poetry install --no-root
  script:
    - poetry run alembic upgrade head
    - poetry run pytest -n auto --cov=app --cov-report=xml --cov-report=term --junitxml=report.xml
  coverage: '/TOTAL.*\s(\d+%)/'
  artifacts:
    when: always
    reports:
      junit: report.xml
      coverage_report: { coverage_format: cobertura, path: coverage.xml }
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH == "main"
```
### 🔍 Line by line
- `variables` — test config; hostnames (`postgres`, `redis`) match the service aliases.
- `services` — sidecar containers for DB/Redis.
- `cache` — reuse the virtualenv between runs, keyed on the lockfile.
- `alembic upgrade head` — tests run against the real migrated schema (and prove migrations apply).
- `pytest -n auto` — parallel via `pytest-xdist`; `--junitxml` for the MR test tab.
- `coverage:` regex — GitLab shows coverage % on the MR.
- `artifacts.reports` — test results and coverage visualised in the MR; failing tests block merge.

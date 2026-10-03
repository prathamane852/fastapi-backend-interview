# 04 — Dependency Injection

---

## 📖 Vocabulary

> *Definition* is the precise meaning; *Say it in an interview* is the quotable one-liner.

| Term | Definition | Say it in an interview |
|---|---|---|
| **Dependency** | An object or value a piece of code requires to perform its function (a DB session, the current user, configuration). | "A dependency is anything my handler needs but shouldn't construct itself." |
| **Dependency Injection (DI)** | A design pattern in which a component receives its dependencies from an external provider rather than creating them, improving decoupling and testability. | "DI means the framework builds what my code needs and passes it in." |
| **Inversion of Control (IoC)** | The principle that a framework controls the flow and calls user code, supplying its inputs; DI is one realization of IoC. | "IoC flips control: the framework calls me and provides my inputs." |
| **`Depends()`** | FastAPI's marker declaring that a parameter's value is produced by calling the given callable, whose own parameters are resolved the same way. | "`Depends` tells FastAPI to call a function and inject its result." |
| **Callable** | Any object invocable with `()`: functions, classes, instances defining `__call__`, lambdas, partials. | "FastAPI accepts any callable as a dependency." |
| **Generator dependency** | A dependency written with `yield`: code before `yield` runs before the handler, the yielded value is injected, code after `yield` runs at teardown. | "A `yield` dependency gives me setup and guaranteed cleanup around a request." |
| **Setup / teardown** | Resource acquisition before use and release afterward; in `yield` dependencies these are the segments before and after `yield`. | "Setup opens the resource; teardown closes it even if the handler fails." |
| **DAG (Directed Acyclic Graph)** | A graph with directed edges and no cycles; FastAPI's dependency graph must be a DAG so it can be resolved in order. | "Dependencies form a DAG that FastAPI resolves depth-first." |
| **Per-request caching (`use_cache`)** | FastAPI's default behavior of calling each distinct dependency at most once per request and reusing its result for other dependents. | "The same dependency used twice in a request runs once." |
| **Singleton** | An object instantiated once and shared across the process lifetime. | "Settings and connection pools should be process singletons." |
| **`functools.lru_cache`** | A decorator memoizing a function's results by arguments; with no arguments it turns a factory into a lazy singleton. | "`lru_cache` on `get_settings` makes it load once per process." |
| **Closure** | A function that captures variables from its enclosing scope and retains access to them after that scope has exited. | "A closure is a function that remembers the variables it was created with." |
| **Dependency factory** | A function that returns a dependency callable configured by its arguments (`require_roles("admin")`). | "A factory returns a configured dependency, so I can parameterize checks." |
| **Dependency override** | Replacing a dependency's provider during tests via `app.dependency_overrides[original] = replacement`. | "Overrides swap real dependencies for fakes in tests without touching handlers." |
| **Router-level dependency** | A dependency listed in `APIRouter(dependencies=[...])` or `include_router(dependencies=[...])`, executed for every route in that router. | "Router-level dependencies enforce auth across a whole group of routes." |
| **Principal** | The authenticated identity on whose behalf a request executes: a user, a service, or an API key. | "The principal is 'who this request is acting as'." |
| **JWT** | *JSON Web Token* — a compact, signed token (`header.payload.signature`) carrying claims such as `sub` and `exp`; verified statelessly. | "A JWT is a signed, self-contained token I can verify without a session store." |
| **Claim** | A key/value assertion inside a JWT payload (`sub`, `exp`, `iat`, `roles`). | "Claims are the facts a token asserts about its subject." |
| **`OAuth2PasswordBearer`** | A FastAPI security dependency that extracts a bearer token from the `Authorization` header (401 if absent) and registers the scheme in OpenAPI. | "`OAuth2PasswordBearer` pulls the bearer token and gives Swagger an Authorize button." |
| **`Header()`** | Declarator reading a parameter from an HTTP header; underscores in the parameter name map to hyphens. | "`Header()` reads a value from a request header." |
| **`pydantic-settings` / `BaseSettings`** | A Pydantic extension whose models load field values from environment variables and `.env` files with validation. | "`BaseSettings` turns environment variables into a validated config object." |
| **`app.state` / `request.state`** | Free-form namespaces for storing per-application (`app.state`) or per-request (`request.state`) data accessible to middleware and handlers. | "`request.state` is how middleware hands data to handlers without DI." |
| **RBAC** | *Role-Based Access Control* — permissions are granted to roles, and users are assigned roles. | "RBAC checks a user's role against the roles allowed for an action." |
| **401 vs 403** | 401 *Unauthorized*: authentication missing or invalid. 403 *Forbidden*: authenticated but not permitted. | "401 is 'I don't know you', 403 is 'I know you and the answer is no'." |
| **API gateway** | An edge component that routes requests to services and centralizes concerns like authentication, rate limiting and TLS termination. | "A gateway is the front door that can validate tokens once for all services." |
| **Trusted headers** | Identity headers (`X-User-Id`) a gateway forwards to services, which must only be accepted from the gateway over a private, authenticated path. | "Services trust forwarded identity headers only from the gateway, never from the internet." |

---

## Q1. What is dependency injection in FastAPI?

### 🧒 Simple explanation
Your endpoint *says what it needs* (a DB session, the logged-in user) as parameters, and FastAPI *builds those things and hands them over* before calling the endpoint. The endpoint never constructs them itself.

**Analogy:** A surgeon doesn't fetch instruments; they say "scalpel" and a nurse places it in their hand. The surgeon (endpoint) focuses on the operation; the nurse (FastAPI) handles supply.

### 🧠 First principles
Endpoints share cross-cutting needs: auth, DB, config, pagination parsing. If each endpoint builds these itself you get duplication and no single place to change behaviour. DI inverts control: the framework owns *construction*, the endpoint owns *logic*. The payoff is testability: you can swap any dependency for a fake without touching the endpoint.

---

## Q2. What is `Depends()`?

### 💻 Code
```python
from typing import Annotated
from fastapi import Depends, FastAPI

app = FastAPI()

def pagination(page: int = 1, size: int = 20) -> dict:
    return {"offset": (page - 1) * size, "limit": size}

@app.get("/items")
async def list_items(p: Annotated[dict, Depends(pagination)]):
    return repo.list(**p)
```
### 🔍 Line by line
- `def pagination(page: int = 1, size: int = 20) -> dict:` — an ordinary function. Its parameters are resolved *exactly like endpoint parameters*: `page` and `size` become **query parameters**.
- `return {...}` — computes the SQL offset/limit from page numbers.
- `p: Annotated[dict, Depends(pagination)]` — "to get `p`, call `pagination(...)`, resolving its parameters from the request". The result (a dict) is injected as `p`.
- `repo.list(**p)` — unpack the dict as keyword arguments.
- In the docs, `/items` shows `page` and `size` parameters even though they're not in `list_items`'s signature — FastAPI merged them from the dependency.

---

## Q3. Why would you use dependencies?

- **Reuse** — one `get_current_user` for every endpoint.
- **Composition** — dependencies can use other dependencies.
- **Testability** — `app.dependency_overrides` swaps any of them.
- **Documentation** — their parameters show up in OpenAPI.
- **Lifecycle** — `yield` dependencies handle open/close (DB sessions).
- **Enforcement** — attach to a router so every route is protected without repeating code.

---

## Q4. How do you create a reusable dependency?

### 💻 Code
```python
# app/api/deps.py
from collections.abc import AsyncIterator
from typing import Annotated
from fastapi import Depends
from sqlalchemy.ext.asyncio import AsyncSession
from app.db.session import SessionLocal

async def get_db() -> AsyncIterator[AsyncSession]:
    async with SessionLocal() as session:
        yield session

DB = Annotated[AsyncSession, Depends(get_db)]

# app/api/v1/users.py
@router.get("/{user_id}")
async def get_user(user_id: int, db: DB):
    ...
```
### 🔍 Line by line
- `async def get_db() -> AsyncIterator[AsyncSession]:` — an **async generator** (it contains `yield`). Return type says "yields `AsyncSession` objects".
- `async with SessionLocal() as session:` — `SessionLocal` is a session factory (file 08). `async with` opens the session and guarantees it's closed at the end of the block, even on error.
- `yield session` — **pause here** and hand `session` to whoever depends on it. When the request finishes, execution resumes after `yield`, which exits the `async with` block and closes the session.
- `DB = Annotated[AsyncSession, Depends(get_db)]` — a reusable *type alias*. Anywhere you write `db: DB`, you get an injected session. Cleaner than repeating `Depends(get_db)`.
- `async def get_user(user_id: int, db: DB):` — the endpoint just declares `db: DB`.

---

## Q5. Can dependencies depend on other dependencies?

Yes — this is the core power. FastAPI builds a **graph** and resolves it depth-first.

```python
from fastapi import Header, HTTPException

async def get_token(authorization: Annotated[str, Header()]) -> str:
    return authorization.removeprefix("Bearer ")

async def get_current_user(token: Annotated[str, Depends(get_token)], db: DB) -> User:
    ...

async def require_admin(user: Annotated[User, Depends(get_current_user)]) -> User:
    if not user.is_admin:
        raise HTTPException(403)
    return user
```
### 🔍 Line by line
- `authorization: Annotated[str, Header()]` — read the `Authorization` HTTP header. FastAPI maps the parameter name to the header name (underscores → hyphens).
- `.removeprefix("Bearer ")` — strip the scheme, leaving the token.
- `get_current_user` depends on `get_token` **and** `get_db`.
- `require_admin` depends on `get_current_user`.
- If an endpoint uses both `get_current_user` and `require_admin`, `get_current_user` runs **once**; the second use gets the cached result (default `use_cache=True`).

---

## Q6. How do you use dependencies for authentication?

### 💻 Code
```python
from fastapi import Depends, HTTPException
from fastapi.security import OAuth2PasswordBearer
from jose import jwt, JWTError

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="/auth/token")

async def get_current_user(
    token: Annotated[str, Depends(oauth2_scheme)],
    db: DB,
) -> User:
    try:
        payload = jwt.decode(token, settings.jwt_secret, algorithms=["HS256"])
    except JWTError:
        raise HTTPException(401, "Invalid token", headers={"WWW-Authenticate": "Bearer"})
    user = await db.get(User, int(payload["sub"]))
    if user is None or not user.is_active:
        raise HTTPException(401, "Inactive user")
    return user

CurrentUser = Annotated[User, Depends(get_current_user)]

@router.get("/me")
async def me(user: CurrentUser):
    return user
```
### 🔍 Line by line
- `from jose import jwt, JWTError` — `python-jose` library for JWTs.
- `OAuth2PasswordBearer(tokenUrl="/auth/token")` — a *security scheme* object. It is itself a dependency: it reads `Authorization: Bearer <token>`, returns the token string, or raises 401 if missing. `tokenUrl` tells the docs where to log in, which enables the "Authorize" button in Swagger.
- `token: Annotated[str, Depends(oauth2_scheme)]` — inject the raw token.
- `jwt.decode(token, secret, algorithms=["HS256"])` — verify signature and expiry, return the claims dict. **Always pass `algorithms` explicitly** (security).
- `except JWTError:` — bad signature, expired, malformed → 401. The `WWW-Authenticate` header is required by the HTTP spec on 401.
- `payload["sub"]` — "subject" claim = user ID. `db.get(User, id)` loads by primary key.
- `CurrentUser = Annotated[...]` — the alias every protected endpoint uses.

---

## Q7. How do you inject a database session?

Shown in Q4. Two refinements:
- Use a **`yield`** dependency so the session closes even when the endpoint raises.
- Decide where `commit` lives. "Commit on success" in the dependency is convenient:

```python
async def get_db() -> AsyncIterator[AsyncSession]:
    async with SessionLocal() as session:
        try:
            yield session
            await session.commit()
        except Exception:
            await session.rollback()
            raise
```
- If the endpoint raises, the exception is thrown *into* the generator at the `yield` line → we hit `except`, roll back, and re-raise so the error handler still runs.
- If it succeeds, execution continues after `yield` → commit.
- Larger apps often commit inside the service/unit-of-work instead; either is fine as long as it's consistent.

---

## Q8. How do you share configuration through dependencies?

```python
from functools import lru_cache
from pydantic import SecretStr
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env", env_prefix="APP_")
    database_url: str
    jwt_secret: SecretStr
    debug: bool = False

@lru_cache
def get_settings() -> Settings:
    return Settings()

Config = Annotated[Settings, Depends(get_settings)]

@router.get("/info")
async def info(cfg: Config):
    return {"debug": cfg.debug}
```
### 🔍 Line by line
- `BaseSettings` — a Pydantic model that reads its fields from **environment variables** (and a `.env` file).
- `env_prefix="APP_"` — field `database_url` is read from `APP_DATABASE_URL`.
- `jwt_secret: SecretStr` — never printed in logs/reprs.
- `@lru_cache` on `get_settings` — the first call builds `Settings()` (reads env, validates); every later call returns the same object instantly. One instance per process.
- Missing required env vars → `ValidationError` at first access = **fail fast** at startup.
- Tests: `app.dependency_overrides[get_settings] = lambda: Settings(database_url="sqlite://", ...)`.

---

## Q9. What is the lifecycle of a dependency?

Per request:
1. FastAPI looks up the route's dependency graph (built once at startup).
2. Resolves each dependency depth-first; results cached **within the request**.
3. For `yield` dependencies, runs the code up to `yield`.
4. Runs the endpoint.
5. Sends the response.
6. Runs the code **after** `yield` (teardown), in reverse order.

Nuances:
- Since FastAPI 0.106, teardown runs *after the response is sent*. You can't change the response there, and a background task can't reuse the session (it's closed).
- Exceptions from the endpoint are thrown into the generator at `yield`, so `try/except` around `yield` works for rollback.
- Dependencies are **not** singletons unless you make them so (`lru_cache`, module-level objects, or `app.state` set in `lifespan`).

---

## Q10. What is the difference between dependency injection and middleware?

| | Dependency | Middleware |
|---|---|---|
| Scope | Per route (opt-in) | Every request (global) |
| Sees | Parsed, typed params; other deps | Raw `Request`/`Response`, ASGI scope |
| Can pass values to the handler | ✅ | ❌ (only via `request.state`) |
| Appears in OpenAPI | ✅ | ❌ |
| Runs before routing | ❌ | ✅ |
| Typical use | Current user, DB session, pagination | Logging, CORS, tracing, compression |

**Rule:** if the handler needs the *value* → dependency. If it's cross-cutting and route-agnostic → middleware.

---

## Q11. How do you override dependencies during testing?

```python
# tests/conftest.py
import pytest
from httpx import AsyncClient, ASGITransport
from app.main import app
from app.api.deps import get_db, get_current_user

@pytest.fixture
async def client(test_session):
    async def _db():
        yield test_session

    async def _user():
        return User(id=1, email="t@t.com", is_admin=True)

    app.dependency_overrides[get_db] = _db
    app.dependency_overrides[get_current_user] = _user

    async with AsyncClient(transport=ASGITransport(app=app), base_url="http://test") as c:
        yield c

    app.dependency_overrides.clear()
```
### 🔍 Line by line
- `@pytest.fixture` — a reusable setup function; tests that take a `client` argument get this.
- `async def _db(): yield test_session` — replacement that hands out the test's session (usually inside a transaction rolled back at the end).
- `async def _user(): return User(...)` — skip token parsing; inject a fake user.
- `app.dependency_overrides[original] = replacement` — a dict FastAPI consults at resolve time. Key = the **original function object**.
- `AsyncClient(transport=ASGITransport(app=app))` — calls the app in-process, no network.
- `app.dependency_overrides.clear()` — after the test, so overrides don't leak into other tests.

---

## Q12. How do you create class-based dependencies?

Any *callable* works. A class is handy when the dependency needs configuration:

```python
from fastapi import Request

class RateLimiter:
    def __init__(self, max_per_minute: int):
        self.max = max_per_minute

    async def __call__(self, request: Request, redis: Redis = Depends(get_redis)):
        key = f"rl:{request.client.host}:{request.url.path}"
        n = await redis.incr(key)
        if n == 1:
            await redis.expire(key, 60)
        if n > self.max:
            raise HTTPException(429, "Too many requests")

@router.post("/login", dependencies=[Depends(RateLimiter(max_per_minute=5))])
async def login(...): ...
```
### 🔍 Line by line
- `__init__` stores config.
- `async def __call__(self, request: Request, redis: ...)` — makes instances callable. FastAPI calls the instance; its parameters (`request`, `redis`) are resolved like any dependency.
- `f"rl:{...}"` — f-string builds a Redis key per client IP + path.
- `redis.incr(key)` — atomically increment; returns the new count.
- `if n == 1: expire(key, 60)` — first hit in the window sets a 60-second TTL.
- `dependencies=[Depends(RateLimiter(5))]` — run it for side effects without injecting a value.

Also: `params: Annotated[CommonParams, Depends()]` (empty `Depends()`) tells FastAPI to call `CommonParams(...)` using its `__init__` signature.

---

## Q13. Can dependencies be asynchronous?

Yes. `async def` dependencies are awaited on the event loop; plain `def` ones run in the thread pool (same rule as endpoints). Mix freely. Guideline: async I/O library → `async def`; blocking library → `def`.

---

## Q14. How would you implement role-based authorization using dependencies?

Use a **factory** that returns a dependency:

```python
from enum import Enum

class Role(str, Enum):
    viewer = "viewer"
    editor = "editor"
    admin = "admin"

def require_roles(*allowed: Role):
    async def _check(user: CurrentUser) -> User:
        if user.role not in allowed:
            raise HTTPException(403, f"Requires one of {[r.value for r in allowed]}")
        return user
    return _check

@router.delete("/items/{id}", dependencies=[Depends(require_roles(Role.admin))])
async def delete_item(id: int): ...

@router.put("/items/{id}")
async def edit_item(id: int, user: Annotated[User, Depends(require_roles(Role.editor, Role.admin))]):
    ...
```
### 🔍 Line by line
- `def require_roles(*allowed: Role):` — `*allowed` collects any number of roles into a tuple.
- `async def _check(user: CurrentUser)` — the *actual* dependency; a closure that remembers `allowed`. It reuses `CurrentUser`, so authentication happens first.
- `raise HTTPException(403, ...)` — authenticated but not permitted → 403 (not 401).
- `return _check` — hand the closure back so `Depends()` can call it.
- `dependencies=[...]` form guards without needing the user; the parameter form guards *and* gives you the user.

Hierarchical roles: map each to a rank and compare. Fine-grained rules: switch to permissions (file 07).

---

## Q15. What problems can occur if dependencies contain expensive operations?

- They run **every request** — a settings loader that re-reads a file, or a user lookup hitting the DB per dependency, multiplies latency.
- **Blocking code** in an `async def` dependency freezes the whole event loop.
- **Resource leaks** if a `yield` dependency opens something and teardown is skipped (early `return`, swallowed exceptions).
- **Hidden fan-out** — a deep tree can open several DB sessions per request.
- **Slow tests** unless overridden.

Mitigations: `lru_cache` for process-level singletons; rely on per-request caching; cache user lookups in Redis with a short TTL; keep dependencies thin; add a tracing span per dependency to see their cost.

---

## Q16. "You have 100 APIs and every API needs the authenticated user. How would you design this?"

### 🧒 Simple explanation
Don't add the user dependency to 100 functions. Attach it **once at the router level**, keep it cheap, and keep a single place to fake it in tests.

### 💻 Code
```python
# app/api/deps.py
CurrentUser = Annotated[User, Depends(get_current_user)]

# app/api/v1/__init__.py
from fastapi import APIRouter, Depends

protected = APIRouter(dependencies=[Depends(get_current_user)])
protected.include_router(users.router, prefix="/users")
protected.include_router(orders.router, prefix="/orders")
# ... the other 98 routers

public = APIRouter()
public.include_router(auth.router, prefix="/auth")
public.include_router(health.router)

app.include_router(public)
app.include_router(protected, prefix="/api/v1")
```
### 🔍 Line by line
- `APIRouter(dependencies=[Depends(get_current_user)])` — every route included in `protected` runs `get_current_user` first, even if the handler never mentions the user. Unauthenticated → 401 before the handler.
- Handlers that *do* need the user add `user: CurrentUser` and get the **cached** result — no second DB lookup.
- `public` — explicit list of unauthenticated routes (login, health). **Secure by default**: a new router added to `protected` cannot accidentally be open.

### 🧠 Make it cheap
- Verify the JWT **locally** (signature + expiry) — no network call.
- Cache user records in Redis keyed by ID with a 60-second TTL, or put roles/tenant in the token claims so most endpoints never touch the DB.

### 🧠 Make it observable
- Set `request.state.user_id` in the dependency so logging/tracing middleware can tag every log line.

### 🧠 At larger scale
- Terminate auth at an API gateway (Kong, Envoy, AWS API Gateway + Cognito authorizer). The gateway verifies the token and forwards trusted headers (`X-User-Id`, `X-Roles`). Services trust those headers **only** from the gateway's private network (mTLS or network policy) — never from the internet.

### 🧠 Testing
- One override: `app.dependency_overrides[get_current_user] = fake_user`. All 100 APIs become testable without tokens.

### 🎤 Interview answer
> "I'd centralise it as one `get_current_user` dependency, apply it at the router level so protection is opt-out rather than opt-in, keep it cheap with local JWT verification and a short-lived cache, expose the user on `request.state` for observability, and keep a single override hook for tests. At bigger scale I'd move token validation to the gateway and have services trust forwarded identity headers over a private network."

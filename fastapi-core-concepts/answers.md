# FastAPI Core Concepts — Interview Answers

> Written from the perspective of a Lead Backend Architect / Senior Python Engineer.
> Every answer follows the same shape:
> 1. **Plain-English definition** (what a junior could repeat)
> 2. **First-principles reasoning** (why it exists, what problem it solves)
> 3. **Interview-ready answer** (the 30-second version you actually say)
> 4. **End-to-end code example** with a line-by-line walkthrough
>
> All code targets **Python 3.11+, FastAPI 0.110+, Pydantic v2, SQLAlchemy 2.0**.

---

## Table of Contents

1. [FastAPI Fundamentals](#1-fastapi-fundamentals)
2. [Request Handling & Validation](#2-request-handling--validation)
3. [Pydantic Deep Dive](#3-pydantic-deep-dive)
4. [Dependency Injection](#4-dependency-injection)
5. [Async Programming](#5-async-programming)
6. [Middleware](#6-middleware)
7. [Authentication & Authorization](#7-authentication--authorization)
8. [Databases](#8-databases)
9. [Project Structure & Architecture](#9-project-structure--architecture)
10. [Performance & Scaling](#10-performance--scaling)
11. [Background Tasks & Job Processing](#11-background-tasks--job-processing)
12. [Testing](#12-testing)
13. [Observability & Monitoring](#13-observability--monitoring)
14. [Docker & Kubernetes](#14-docker--kubernetes)
15. [CI/CD](#15-cicd)
16. [System Design Questions](#16-system-design-questions)

---

# 1. FastAPI Fundamentals

## Q: What is FastAPI and why would you choose it over Flask or Django?

**Plain English:** FastAPI is a modern Python web framework for building HTTP APIs. You describe your endpoints using ordinary Python functions with type hints, and FastAPI uses those type hints to validate input, serialize output, and generate documentation automatically.

**First principles:** Every web framework has to solve the same four problems: (1) route a URL to code, (2) parse the incoming request into Python objects, (3) validate that data, and (4) turn Python objects back into an HTTP response. Flask makes you do 2–4 by hand. Django gives you a full-stack toolkit (ORM, admin, templates) that assumes you're building a website. FastAPI's insight was: *Python already has type hints — use them as the single source of truth for parsing, validation, and docs.* That removes a whole category of boilerplate and bugs.

**Interview answer:**
> "FastAPI is an async-first Python framework built on Starlette (for the web layer) and Pydantic (for data validation). I'd pick it over Flask when I want validation, serialization and OpenAPI docs for free instead of bolting on Marshmallow + Flask-RESTX. I'd pick it over Django when I'm building an API service, not a server-rendered site, and I don't need Django's admin/ORM/auth batteries. FastAPI also has first-class `async` support, which Flask doesn't and Django only partially does. The trade-off is that FastAPI is *only* the API layer — I must choose my own ORM, auth, and project structure, so it requires more architectural discipline on large teams."

**Comparison table:**

| Concern | Flask | Django | FastAPI |
|---|---|---|---|
| Style | Micro, sync | Full-stack, batteries included | Micro, async-first |
| Validation | Manual / extensions | Django Forms / DRF serializers | Built-in via Pydantic |
| API docs | Extension | DRF + drf-spectacular | Built-in OpenAPI |
| Async | Limited | Partial (views yes, ORM partial) | Native |
| Best for | Small apps, quick prototypes | Content sites, admin-heavy apps | API services, microservices, ML serving |

---

## Q: What are the main features of FastAPI?

1. **Type-hint-driven validation** – function signatures define the API contract.
2. **Automatic interactive docs** – Swagger UI at `/docs`, ReDoc at `/redoc`.
3. **Async support** – `async def` endpoints run on the event loop.
4. **Dependency injection** – `Depends()` for auth, DB sessions, config.
5. **Pydantic models** – request/response schemas with serialization.
6. **Standards-based** – OpenAPI 3.x and JSON Schema.
7. **Editor support** – autocompletion and type checking because everything is typed.
8. **Starlette underneath** – middleware, WebSockets, background tasks, test client.
9. **Security utilities** – OAuth2, JWT helpers, API-key schemes.
10. **High performance** – comparable to Node/Go for I/O-bound work.

**Interview answer:** "The feature I'd highlight is that the *type hints are the contract*. One signature gives you validation, serialization, docs, and editor support. Everything else — DI, async, security schemes — builds on that."

---

## Q: How does FastAPI achieve high performance?

**First principles:** A web server's throughput for typical API work (DB calls, HTTP calls) is limited by *waiting*, not by computing. If a worker blocks while waiting on the database, it can't serve anyone else. FastAPI avoids blocking through three layers:

1. **ASGI + event loop (via Starlette/Uvicorn):** One process can juggle thousands of in-flight requests because when one awaits I/O, the loop switches to another.
2. **uvloop + httptools (Uvicorn's optional C extensions):** The event loop and HTTP parser are implemented in C/Cython, so the per-request overhead is tiny.
3. **Pydantic v2 (pydantic-core in Rust):** Validation and serialization — normally the slowest Python part of a request — happen in compiled Rust code.

**Interview answer:**
> "FastAPI itself is thin. The speed comes from Starlette's ASGI design running on Uvicorn with uvloop, plus Pydantic v2 doing validation in Rust. For I/O-bound APIs that means a single process can handle thousands of concurrent connections. For CPU-bound work it's still Python — you'd scale with worker processes or move the work out of the request path."

---

## Q: What is ASGI?

**Plain English:** ASGI (Asynchronous Server Gateway Interface) is the standard *contract* between a Python web server (Uvicorn) and a Python web application (FastAPI). It says: "the server will call your app with a `scope`, a `receive` function, and a `send` function."

**First principles:** Servers and frameworks need to be interchangeable. Without a standard, every framework would need its own server. WSGI solved this for synchronous apps in 2003. But WSGI's design (`app(environ, start_response)` returning an iterable) is fundamentally blocking and can't express long-lived connections like WebSockets. ASGI redesigned the contract around `async`/`await` and message passing.

**Minimal ASGI app (no framework) — line by line:**

```python
async def app(scope, receive, send):          # 1
    assert scope["type"] == "http"            # 2
    body = b""                                # 3
    while True:                               # 4
        message = await receive()             # 5
        body += message.get("body", b"")      # 6
        if not message.get("more_body"):      # 7
            break
    await send({                              # 8
        "type": "http.response.start",
        "status": 200,
        "headers": [(b"content-type", b"text/plain")],
    })
    await send({                              # 9
        "type": "http.response.body",
        "body": b"Hello ASGI",
    })
```

1. An ASGI app is just an `async` callable with three arguments.
2. `scope` is a dict describing the connection: type (`http`, `websocket`, `lifespan`), path, headers, method, etc.
3. We'll accumulate the request body here.
4–7. The body may arrive in chunks; `receive()` returns one message at a time. `more_body=False` means we have everything.
8. Responses are also messages. First we send the status and headers.
9. Then the body. We could send several body chunks for streaming.

FastAPI wraps all of this so you never see it, but knowing it lets you write custom middleware and debug server behaviour.

---

## Q: What is the difference between ASGI and WSGI?

| | WSGI | ASGI |
|---|---|---|
| Year | 2003 (PEP 3333) | 2016+ |
| Model | Synchronous, one request per thread/process | Asynchronous, event-loop based |
| Signature | `app(environ, start_response)` | `async app(scope, receive, send)` |
| Protocols | HTTP only | HTTP, WebSocket, lifespan, custom |
| Streaming | Awkward | Natural (message-based) |
| Servers | Gunicorn, uWSGI | Uvicorn, Hypercorn, Daphne |
| Frameworks | Flask, Django (classic) | FastAPI, Starlette, Django (async views) |

**Interview answer:** "WSGI is a synchronous call-and-return contract, so a worker is tied up for the duration of a request. ASGI is an async, message-based contract, so one worker can interleave thousands of requests and also handle long-lived connections like WebSockets. ASGI is a superset — you can run WSGI apps inside ASGI servers via an adapter, but not the reverse."

---

## Q: What is Uvicorn?

**Plain English:** Uvicorn is the ASGI *server* — the program that listens on a TCP port, parses HTTP bytes, and calls your FastAPI app. FastAPI can't serve itself; it needs Uvicorn (or Hypercorn, Granian, etc.).

**First principles:** Separation of concerns. The server handles sockets, TLS termination, HTTP parsing, keep-alive, and the event loop. The framework handles routing and business logic. Uvicorn is built on `uvloop` (a fast libuv-based event loop) and `httptools` (a fast HTTP parser), both C-based.

```bash
uvicorn app.main:app --host 0.0.0.0 --port 8000 --workers 4
```

- `app.main:app` → module path `app/main.py`, variable `app`.
- `--workers 4` → fork 4 processes; each runs its own event loop.
- In production, add `--proxy-headers --forwarded-allow-ips="*"` when behind a load balancer, and `--loop uvloop --http httptools` (defaults when installed via `uvicorn[standard]`).

---

## Q: What is Starlette and how is it related to FastAPI?

**Plain English:** Starlette is a lightweight ASGI toolkit. It provides routing, middleware, request/response objects, WebSockets, background tasks, static files, and a test client. FastAPI is *built on top of* Starlette and adds the type-hint-based validation, dependency injection, and OpenAPI generation.

**Relationship:** `FastAPI` is literally a subclass of `starlette.applications.Starlette`. When you see `from starlette.requests import Request` and `from fastapi import Request`, they're the same class — FastAPI re-exports it.

**Interview answer:** "Starlette is the web engine; FastAPI is the developer-experience layer on top. Starlette handles ASGI, routing, and middleware; FastAPI adds Pydantic validation, `Depends()`, and docs. If you understand Starlette you understand 70% of how FastAPI works internally."

---

## Q: How do you create a basic FastAPI application?

```python
# app/main.py
from fastapi import FastAPI                        # 1

app = FastAPI(                                     # 2
    title="Orders Service",
    version="1.0.0",
    description="Manages customer orders",
)

@app.get("/health")                                # 3
async def health() -> dict[str, str]:             # 4
    return {"status": "ok"}                        # 5
```

1. Import the application class.
2. Instantiate it. The keyword args feed the OpenAPI document (`title`, `version` appear in `/docs`).
3. A *decorator* registers the function as a **path operation**: method `GET`, path `/health`.
4. `async def` means it runs on the event loop. The return annotation is used for documentation and response validation.
5. Returning a dict → FastAPI serializes it to JSON with `Content-Type: application/json` and status 200.

---

## Q: How do you start a FastAPI application using Uvicorn?

**Development:**
```bash
uvicorn app.main:app --reload --port 8000
```
`--reload` watches files and restarts on change (never in production).

**Programmatic (useful for debuggers):**
```python
# run.py
import uvicorn

if __name__ == "__main__":
    uvicorn.run("app.main:app", host="0.0.0.0", port=8000, reload=True)
```

**Production:**
```bash
uvicorn app.main:app --host 0.0.0.0 --port 8000 --workers 4 --proxy-headers
```
Or via Gunicorn as a process manager with Uvicorn workers:
```bash
gunicorn app.main:app -k uvicorn.workers.UvicornWorker -w 4 -b 0.0.0.0:8000
```
Gunicorn adds graceful worker restarts, max-requests recycling, and better signal handling. Modern Uvicorn (≥0.30) has its own multi-process supervisor, so both are valid.

---

## Q: What is an API path operation?

**Plain English:** In FastAPI vocabulary, a *path operation* is the combination of an HTTP method (the "operation") and a URL path, bound to a Python function. `@app.get("/items/{id}")` + the function beneath it = one path operation.

The term comes from the OpenAPI spec, where each path (`/items/{id}`) has operations (`get`, `post`, …).

```python
@app.post("/items", status_code=201, tags=["items"], summary="Create an item")
async def create_item(item: ItemCreate) -> ItemRead:
    ...
```
- `status_code=201` sets the default success status.
- `tags` groups the endpoint in the docs.
- `summary` is the short description in Swagger.

---

## Q: What is the difference between GET, POST, PUT, PATCH, and DELETE?

**First principles:** HTTP methods are *semantic hints* to clients, caches and proxies. Two properties matter:
- **Safe** – doesn't change server state (GET).
- **Idempotent** – calling it N times has the same effect as calling it once (GET, PUT, DELETE). POST and PATCH are *not* guaranteed idempotent.

| Method | Purpose | Safe | Idempotent | Body |
|---|---|---|---|---|
| GET | Read a resource | ✅ | ✅ | No |
| POST | Create / trigger an action | ❌ | ❌ | Yes |
| PUT | Replace the whole resource | ❌ | ✅ | Yes |
| PATCH | Partially update | ❌ | ❌ (usually) | Yes |
| DELETE | Remove | ❌ | ✅ | Rarely |

**Interview answer:** "GET reads and is cacheable. POST creates and is not idempotent — retrying can create duplicates, which is why idempotency keys exist. PUT replaces fully and is idempotent. PATCH sends only the changed fields. DELETE is idempotent — deleting twice still results in 'not there', even if the second call returns 404."

```python
@app.put("/items/{item_id}")            # full replacement: client sends every field
async def replace_item(item_id: int, item: ItemCreate): ...

@app.patch("/items/{item_id}")          # partial: every field optional
async def update_item(item_id: int, patch: ItemPatch): ...
```

---

## Q: How do you define path parameters?

```python
@app.get("/users/{user_id}")                 # 1
async def get_user(user_id: int):            # 2
    return {"user_id": user_id}
```
1. Curly braces in the path declare a variable segment.
2. The function parameter with the *same name* receives it. The `int` annotation makes FastAPI convert `"42"` → `42` and return a 422 for `"abc"`.

---

## Q: How do you define query parameters?

```python
@app.get("/users")
async def list_users(page: int = 1, size: int = 20, q: str | None = None):
    return {"page": page, "size": size, "q": q}
```
Any function parameter *not* in the path and *not* a Pydantic model is treated as a query parameter. `page=1` has a default so it's optional. `q: str | None = None` is optional and can be omitted entirely. `/users?page=2&q=bob` → `page=2, size=20, q="bob"`.

---

## Q: How do you define request body parameters?

```python
from pydantic import BaseModel

class UserCreate(BaseModel):                  # 1
    email: str
    name: str
    age: int | None = None

@app.post("/users", status_code=201)
async def create_user(user: UserCreate):      # 2
    return {"id": 1, **user.model_dump()}     # 3
```
1. A Pydantic model describes the expected JSON shape.
2. A parameter annotated with a Pydantic model is read from the JSON body.
3. `model_dump()` converts the validated model to a dict.

---

## Q: What is Pydantic?

**Plain English:** Pydantic is a data-validation library. You declare a class with type-annotated fields; Pydantic parses raw input (dict/JSON) into an instance, coercing types where sensible and raising a detailed error otherwise.

**First principles:** Data crossing a boundary (network, file, DB) is untrusted. You need a *gate* that guarantees "past this point, `age` is an int between 0 and 150." Pydantic makes the gate declarative and fast (v2's core is Rust).

---

## Q: Why does FastAPI use Pydantic?

Because Pydantic models are simultaneously:
- a **validator** (parse the request body),
- a **serializer** (produce the response),
- a **JSON Schema generator** (produce OpenAPI docs),
- and a **typed Python class** (editor support).

One definition powers all four, which is exactly FastAPI's philosophy.

---

## Q: What is request validation?

Checking that the incoming request conforms to the declared contract *before* your business code runs: types, required fields, ranges, formats. FastAPI validates path params, query params, headers, cookies, and body. Failure → automatic `422 Unprocessable Entity` with a machine-readable error list:

```json
{
  "detail": [
    {"type": "int_parsing", "loc": ["path", "user_id"], "msg": "Input should be a valid integer", "input": "abc"}
  ]
}
```

---

## Q: How does FastAPI automatically generate API documentation?

At startup, FastAPI walks every registered route, inspects the function signature and Pydantic models, and builds an **OpenAPI JSON document** (served at `/openapi.json`). Swagger UI and ReDoc are static HTML pages that fetch that JSON and render it. Docstrings become descriptions; `Field(description=...)` and `examples=` enrich it.

```python
class Item(BaseModel):
    name: str = Field(..., description="Display name", examples=["Widget"])
    price: float = Field(..., gt=0, description="Price in USD")
```

---

## Q: What is OpenAPI?

A vendor-neutral, machine-readable specification (JSON/YAML) for describing REST APIs: paths, operations, parameters, schemas, security schemes. Formerly "Swagger Specification". Tooling can generate clients, servers, mocks, tests and docs from it.

---

## Q: What are Swagger UI and ReDoc?

Two web UIs that render an OpenAPI document.
- **Swagger UI** (`/docs`) — interactive, lets you "Try it out" and send requests.
- **ReDoc** (`/redoc`) — read-only, three-panel layout, better for publishing reference docs.

Disable in production if the API is internal: `FastAPI(docs_url=None, redoc_url=None)`.

---

## Q: Explain how FastAPI processes a request from the time it reaches the server until the response is returned.

This is the "trace the request" question — answer it as a pipeline:

```
TCP socket → Uvicorn (parse HTTP → ASGI scope/receive/send)
          → FastAPI/Starlette app.__call__
          → Middleware stack (outermost first: CORS, logging, auth…)
          → Router: match method + path → APIRoute
          → Build request: read body, headers, cookies
          → Resolve dependencies (Depends tree, cached per request)
          → Validate params/body with Pydantic → 422 on failure
          → Call your endpoint (async: on loop; sync: threadpool)
          → Validate/serialize return value via response_model
          → Build Response (JSON), set status/headers
          → Middleware stack unwinds (innermost first)
          → Uvicorn writes bytes to socket
```

**Detailed walkthrough:**

1. **Server layer.** Uvicorn accepts the connection, parses the HTTP request line and headers with `httptools`, and constructs an ASGI `scope` dict. It invokes `app(scope, receive, send)`.
2. **Application entry.** `FastAPI.__call__` → `Starlette.__call__` builds the middleware stack lazily on first request. Each middleware is an ASGI app wrapping the next.
3. **Middleware (request phase).** Middleware runs outer → inner. E.g., `CORSMiddleware` may short-circuit an `OPTIONS` preflight here.
4. **Routing.** The `Router` iterates its routes, matching path regex and method. No match → 404; method mismatch → 405.
5. **Request object.** Starlette wraps scope/receive into a `Request`.
6. **Dependency resolution.** FastAPI walks the endpoint's dependency graph (`Depends`). Each dependency's own params are resolved recursively; results are cached for the request. Sub-dependencies that are `yield` generators have their setup part run now.
7. **Parameter validation.** Path/query/header/cookie/body values are extracted and validated with Pydantic. Errors are collected (not fail-fast) and returned as 422.
8. **Endpoint execution.** If `async def`, it's awaited on the event loop. If `def`, it's dispatched to a threadpool via `run_in_threadpool` so it doesn't block the loop.
9. **Response serialization.** The return value is validated against `response_model` (if any), converted via `jsonable_encoder`, and wrapped in a `JSONResponse`.
10. **Dependency teardown.** Code after `yield` in dependencies runs (e.g., closing a DB session).
11. **Middleware (response phase).** Unwinds inner → outer; a logging middleware records duration here.
12. **Server layer.** Uvicorn sends `http.response.start` and `http.response.body` messages to the socket.

**Interview answer (short):**
> "Uvicorn parses bytes into an ASGI scope and calls the app. The request passes through the middleware stack, is routed to a path operation, FastAPI resolves the dependency tree and validates parameters with Pydantic, runs the handler (on the loop if async, in a threadpool if sync), serializes the return value against the response model, runs dependency cleanup, unwinds middleware, and Uvicorn writes the response."

---

# 2. Request Handling & Validation

## Q: How do you define path parameters in FastAPI?

Covered above; the advanced form uses `Path()` for metadata and constraints:

```python
from fastapi import Path
from typing import Annotated

@app.get("/orders/{order_id}")
async def get_order(
    order_id: Annotated[int, Path(gt=0, description="Numeric order ID")]   # 1
):
    return {"order_id": order_id}
```
1. `Annotated[type, Path(...)]` is the modern style. `gt=0` rejects `0` and negatives with a 422. The description shows in the docs.

**Ordering matters:** `/users/me` must be declared *before* `/users/{user_id}` or `"me"` will be captured and fail int parsing.

---

## Q: How do you validate path parameters?

Use `Path()` constraints: `gt`, `ge`, `lt`, `le` for numbers; `min_length`, `max_length`, `pattern` for strings. Or use an `Enum` / custom Pydantic type:

```python
from enum import Enum

class Region(str, Enum):              # 1
    us = "us"
    eu = "eu"

@app.get("/regions/{region}/status")
async def region_status(region: Region):   # 2
    return {"region": region.value}
```
1. Subclassing `str, Enum` makes the values JSON-friendly and documented as a dropdown in Swagger.
2. Anything not `us` or `eu` → 422.

For UUIDs: `item_id: uuid.UUID` validates format automatically.

---

## Q: How do you define optional query parameters?

```python
from fastapi import Query

@app.get("/search")
async def search(
    q: str | None = None,                                  # 1
    tags: Annotated[list[str] | None, Query()] = None,     # 2
    limit: Annotated[int, Query(ge=1, le=100)] = 20,       # 3
):
    ...
```
1. `| None = None` → optional; absent → `None`.
2. Lists need explicit `Query()` so FastAPI knows to read `?tags=a&tags=b` instead of treating the list as a body.
3. A default value makes it optional *and* validated.

---

## Q: What is the difference between path and query parameters?

| | Path | Query |
|---|---|---|
| Location | `/users/42` | `/users?page=2` |
| Purpose | Identify **which** resource | Modify **how** you get it (filter, sort, paginate) |
| Required | Always | Usually optional |
| Cacheability | Part of the resource identity | Often varies the representation |

**Rule of thumb:** If removing it would make the URL point to a different *thing*, it's a path param. If it just refines the result set, it's a query param.

---

## Q: How do you define request bodies?

Declare a Pydantic model parameter. FastAPI reads `Content-Type: application/json`, parses, validates.

```python
class Address(BaseModel):
    street: str
    city: str

class CustomerCreate(BaseModel):
    name: str
    email: str
    address: Address                      # nested model

@app.post("/customers", status_code=201)
async def create_customer(payload: CustomerCreate):
    return payload
```

---

## Q: How do you accept multiple body parameters?

If two parameters are Pydantic models, FastAPI expects a JSON object with *keys named after the parameters*:

```python
@app.post("/orders")
async def create_order(customer: CustomerCreate, item: ItemCreate):
    ...
```
Expected body:
```json
{"customer": {...}, "item": {...}}
```

For a *scalar* body value alongside models, use `Body()`:
```python
from fastapi import Body

@app.post("/orders")
async def create_order(
    customer: CustomerCreate,
    item: ItemCreate,
    priority: Annotated[int, Body(ge=1, le=5)] = 3,   # embedded as "priority"
):
    ...
```
To force a single model to be embedded under a key: `payload: Annotated[ItemCreate, Body(embed=True)]` → `{"payload": {...}}`.

---

## Q: How do you handle nested Pydantic models?

Just nest types; validation recurses.

```python
class LineItem(BaseModel):
    sku: str
    qty: int = Field(gt=0)

class OrderCreate(BaseModel):
    customer_id: int
    items: list[LineItem] = Field(min_length=1)     # list of nested models
    metadata: dict[str, str] = {}                   # free-form dict
```
Errors carry the full location: `["body", "items", 0, "qty"]`.

---

## Q: How do you define default values?

- Function param default → `page: int = 1`.
- Model field default → `role: str = "user"`.
- Factory default (mutable) → `tags: list[str] = Field(default_factory=list)`. **Never** write `tags: list[str] = []` in plain Python classes; Pydantic actually copies defaults so it's safe there, but `default_factory` is still clearer.
- Computed default → `created_at: datetime = Field(default_factory=lambda: datetime.now(timezone.utc))`.

---

## Q: How do you validate string length?

```python
class Signup(BaseModel):
    username: str = Field(min_length=3, max_length=30, pattern=r"^[a-z0-9_]+$")
```
Or via `Query(min_length=..)` / `Path(...)` for params. Also `Annotated[str, StringConstraints(strip_whitespace=True, to_lower=True)]` from Pydantic for normalization.

---

## Q: How do you validate numeric ranges?

```python
class Product(BaseModel):
    price: float = Field(gt=0, le=1_000_000)          # 0 < price <= 1M
    stock: int = Field(ge=0)                          # non-negative
    discount: float = Field(ge=0, lt=1, default=0)    # [0, 1)
```
`gt`/`ge`/`lt`/`le` = greater-than / greater-or-equal / less-than / less-or-equal. `multiple_of=0.01` is handy for currency.

---

## Q: How do you validate email addresses?

```python
from pydantic import BaseModel, EmailStr    # requires: pip install "pydantic[email]"

class Signup(BaseModel):
    email: EmailStr
```
`EmailStr` uses the `email-validator` package (RFC-compliant syntax check, not deliverability). For deliverability you'd send a verification email — validation only proves *format*.

---

## Q: How do you create reusable Pydantic models?

Use inheritance and composition to avoid repeating fields:

```python
class UserBase(BaseModel):                 # 1 shared fields
    email: EmailStr
    name: str

class UserCreate(UserBase):                # 2 input adds password
    password: str = Field(min_length=8)

class UserRead(UserBase):                  # 3 output adds id, never password
    id: int
    model_config = ConfigDict(from_attributes=True)   # 4

class UserUpdate(BaseModel):               # 5 everything optional for PATCH
    email: EmailStr | None = None
    name: str | None = None
```
1. Base holds what every variant shares.
2. Creation needs a password.
3. Reading exposes `id` but *not* `password`.
4. `from_attributes=True` lets you build it directly from an ORM object (`UserRead.model_validate(orm_user)`).
5. PATCH schemas are all-optional; use `model_dump(exclude_unset=True)` to apply only provided fields.

Generic wrappers are also reusable:
```python
from typing import Generic, TypeVar
T = TypeVar("T")

class Page(BaseModel, Generic[T]):
    items: list[T]
    total: int
    page: int
    size: int

@app.get("/users", response_model=Page[UserRead])
async def list_users(...): ...
```

---

## Q: How do you handle list and dictionary request bodies?

```python
@app.post("/items/bulk")
async def bulk_create(items: list[ItemCreate]):      # top-level JSON array
    return {"count": len(items)}

@app.post("/settings")
async def set_settings(settings: dict[str, int]):    # arbitrary JSON object
    return settings
```
A top-level `list[...]` annotation means the body itself is a JSON array. `RootModel` is the Pydantic-v2 way to add validators to such shapes:
```python
from pydantic import RootModel
class ItemList(RootModel[list[ItemCreate]]):
    @field_validator("root")
    @classmethod
    def not_empty(cls, v):
        if not v: raise ValueError("must not be empty")
        return v
```

---

## Q: How do you upload files using FastAPI?

```python
from fastapi import UploadFile, File, HTTPException
import aiofiles                                         # pip install aiofiles

MAX_BYTES = 10 * 1024 * 1024                            # 1

@app.post("/upload")
async def upload(file: Annotated[UploadFile, File()]):  # 2
    if file.content_type not in {"image/png", "image/jpeg"}:   # 3
        raise HTTPException(415, "Unsupported media type")
    size = 0
    async with aiofiles.open(f"/data/{file.filename}", "wb") as out:   # 4
        while chunk := await file.read(1024 * 1024):    # 5
            size += len(chunk)
            if size > MAX_BYTES:                        # 6
                raise HTTPException(413, "File too large")
            await out.write(chunk)                      # 7
    return {"filename": file.filename, "bytes": size}
```
1. Hard limit on size.
2. `UploadFile` gives a spooled file object (memory first, spills to disk when large). Requires `python-multipart`.
3. Validate MIME type (never trust it fully; sniff magic bytes for real security).
4. Non-blocking file write.
5. Stream in 1 MiB chunks instead of `await file.read()` (which loads everything into memory).
6. Abort early on oversize.
7. Write chunk.

For multi-GB files, don't upload through the API at all — issue a **pre-signed S3 URL** and let the client upload directly (see System Design section).

---

## Q: How do you return a custom HTTP status code?

Three ways:

```python
from fastapi import status, Response
from fastapi.responses import JSONResponse

# 1. Default for the route
@app.post("/items", status_code=status.HTTP_201_CREATED)
async def create(): ...

# 2. Dynamically via the Response object
@app.put("/items/{id}")
async def upsert(id: int, item: ItemCreate, response: Response):
    created = repo.upsert(id, item)
    response.status_code = 201 if created else 200
    return item

# 3. Return a Response directly (bypasses response_model!)
@app.get("/legacy")
async def legacy():
    return JSONResponse(status_code=202, content={"queued": True})

# 4. Errors
raise HTTPException(status_code=404, detail="Item not found")
```
Use the `status` module constants — they document intent better than magic numbers.

---

# 3. Pydantic Deep Dive

## Q: What problem does Pydantic solve?

**The boundary problem.** Data enters your program as untyped blobs (JSON, env vars, DB rows, message queues). Without validation you write `if "age" in data and isinstance(data["age"], int) and 0 <= data["age"] < 150` everywhere — and you still get `KeyError` at 3 AM. Pydantic turns that into a declaration:

```python
class Person(BaseModel):
    age: int = Field(ge=0, lt=150)
```
and gives you: parsing, coercion, structured errors, JSON Schema, serialization, and IDE types — from one definition.

---

## Q: What is the difference between Pydantic models and Python dataclasses?

| | `@dataclass` | `pydantic.BaseModel` |
|---|---|---|
| Validates at runtime | ❌ (types are hints only) | ✅ |
| Type coercion (`"1"`→`1`) | ❌ | ✅ (lax mode) |
| JSON parsing/serialization | Manual | Built in (`model_validate_json`, `model_dump_json`) |
| JSON Schema | ❌ | ✅ |
| Speed of construction | Faster (no validation) | Slower (validation), but v2 is Rust-fast |
| Use for | Internal, trusted data | Boundary, untrusted data |

Pydantic also ships `pydantic.dataclasses.dataclass` — a dataclass with validation — for when you want the stdlib API but need checks.

**Interview answer:** "Dataclasses are containers; Pydantic models are *validated* containers. I use dataclasses for internal domain objects that are already trustworthy, and Pydantic at the edges: API bodies, config, queue messages."

---

## Q: How does Pydantic perform validation?

1. At **class definition time**, Pydantic inspects annotations and builds a *core schema* — a validation plan compiled into `pydantic-core` (Rust).
2. At **instantiation** (`Model(**data)` or `Model.model_validate(data)`), the Rust validator walks the plan: checks types, coerces in *lax mode* (default) or rejects in *strict mode*, applies constraints (`gt`, `min_length`), runs custom validators, and collects *all* errors before raising a single `ValidationError`.
3. Field validators run in order: `before` validators → type validation → `after` validators → model-level `model_validator(mode="after")`.

```python
try:
    Person.model_validate({"age": "abc"})
except ValidationError as e:
    print(e.errors())
# [{'type': 'int_parsing', 'loc': ('age',), 'msg': 'Input should be a valid integer...', 'input': 'abc'}]
```

---

## Q: How do you create custom validators?

```python
from pydantic import BaseModel, field_validator, model_validator

class Booking(BaseModel):
    start: date
    end: date
    guests: int

    @field_validator("guests")                      # 1
    @classmethod
    def guests_positive(cls, v: int) -> int:        # 2
        if v <= 0:
            raise ValueError("guests must be > 0")  # 3
        return v                                    # 4

    @model_validator(mode="after")                  # 5
    def end_after_start(self) -> "Booking":         # 6
        if self.end <= self.start:
            raise ValueError("end must be after start")
        return self
```
1. `field_validator("guests")` runs after type validation for that field (mode `"after"` is default).
2. Receives the already-typed value.
3. `ValueError` becomes a validation error with your message (also `AssertionError`).
4. Must return the (possibly transformed) value.
5. `model_validator(mode="after")` runs once all fields are valid — for cross-field rules.
6. Receives `self`; return it.

Use `mode="before"` when you need to normalize raw input (e.g., strip whitespace from a string before it's parsed as a date).

---

## Q: How do you validate nested objects?

Nested models validate recursively and automatically. To add rules that span parent and child, use a model validator on the parent:

```python
class Item(BaseModel):
    price: float = Field(gt=0)
    qty: int = Field(gt=0)

class Cart(BaseModel):
    items: list[Item]

    @model_validator(mode="after")
    def total_cap(self):
        if sum(i.price * i.qty for i in self.items) > 10_000:
            raise ValueError("cart exceeds $10,000")
        return self
```

---

## Q: How do you make a field optional?

Distinguish two ideas:
- **Not required** – has a default: `nickname: str = "anon"`.
- **Nullable** – accepts `None`: `nickname: str | None`.

```python
class Profile(BaseModel):
    nickname: str | None = None        # optional AND nullable (most common)
    bio: str = ""                      # optional, not nullable
    age: int | None                    # REQUIRED but nullable (must send "age": null or a value)
```
In Pydantic v2, `Optional[X]` without a default is *required* — a common surprise.

---

## Q: How do you handle extra/unknown fields?

```python
from pydantic import ConfigDict

class Strict(BaseModel):
    model_config = ConfigDict(extra="forbid")    # 422 if unknown keys sent

class Lenient(BaseModel):
    model_config = ConfigDict(extra="ignore")    # default: silently drop

class Passthrough(BaseModel):
    model_config = ConfigDict(extra="allow")     # keep them, accessible via .model_extra
```
**Recommendation:** `extra="forbid"` on request schemas catches client typos (`"emial"`) early. `ignore` on schemas that parse third-party payloads so upstream additions don't break you.

---

## Q: How do you serialize a Pydantic model?

```python
u = UserRead(id=1, email="a@b.com", name="A")
u.model_dump()                  # → dict (Python types: datetime stays datetime)
u.model_dump(mode="json")       # → dict with JSON-safe types (datetime → ISO string)
u.model_dump_json()             # → str of JSON
u.model_dump(exclude={"email"}) # → drop fields
u.model_dump(by_alias=True)     # → use aliases (camelCase etc.)
u.model_dump(exclude_none=True) # → drop None values
```
Custom serialization per field: `@field_serializer("created_at")`. Computed output fields: `@computed_field`.

---

## Q: How do you convert a Pydantic model to a dictionary?

`model.model_dump()` (v2; `.dict()` in v1 is deprecated). Options: `include`, `exclude`, `exclude_unset` (only fields the client actually sent — essential for PATCH), `exclude_defaults`, `exclude_none`, `by_alias`, `mode`.

---

## Q: What is the difference between request models and response models?

- **Request model** — the shape you *accept*. Contains things the client provides (`password`, `credit_card`). Should be strict (`extra="forbid"`).
- **Response model** — the shape you *return*. Contains server-generated fields (`id`, `created_at`) and must **exclude** secrets. Set with `response_model=` or the return annotation; FastAPI filters the output to exactly those fields.

```python
@app.post("/users", response_model=UserRead, status_code=201)
async def create_user(body: UserCreate, svc: UserService = Depends()):
    user = await svc.create(body)     # returns ORM object with hashed_password
    return user                       # FastAPI serializes only UserRead's fields
```

---

## Q: Why should you define separate request and response schemas?

1. **Security** – response schemas act as an allow-list; you can't leak `hashed_password` by accident.
2. **Asymmetry** – clients never send `id`/`created_at`; servers never return `password`.
3. **Evolution** – you can add output fields without changing the input contract, and vice versa.
4. **Docs clarity** – consumers see exactly what to send and what to expect.
5. **Mass-assignment prevention** – a single shared model with `is_admin` would let a client set it on creation.

---

## Q: How do you prevent sensitive fields from appearing in API responses?

Layered defense:

1. **Response model allow-list** (primary): `response_model=UserRead` where `UserRead` has no `hashed_password`.
2. **Never define the field in output schemas** rather than relying on `exclude`.
3. **`response_model_exclude`** for ad-hoc cases: `@app.get(..., response_model=User, response_model_exclude={"ssn"})`.
4. **`SecretStr`** for values that must exist in memory but should never print: `api_key: SecretStr` → repr shows `**********`; get value with `.get_secret_value()`.
5. **Log redaction middleware** (see Middleware section) so secrets don't leak through logs.

---

## Q: How do you handle backward compatibility when changing schemas?

Rules that keep old clients working:
- **Adding an optional field** to a request → safe.
- **Adding a field** to a response → safe (clients ignore unknown keys).
- **Renaming/removing/retyping** a field → breaking. Instead: add the new field, keep the old one with a deprecation note, dual-write for a period, then remove in a new major version.
- Use **aliases** to accept both names during transition:

```python
class Order(BaseModel):
    customer_id: int = Field(validation_alias=AliasChoices("customer_id", "customerId"))
```
- Mark deprecated fields: `Field(deprecated=True)` (Pydantic ≥2.7) shows in OpenAPI.
- Never make an optional field required in place; introduce a new version.
- Contract tests (e.g., Schemathesis against a stored OpenAPI snapshot) fail CI on breaking changes.

---

## Q: How would you validate an enum field?

```python
from enum import Enum

class OrderStatus(str, Enum):
    pending = "pending"
    paid = "paid"
    shipped = "shipped"

class OrderUpdate(BaseModel):
    status: OrderStatus
```
Input `"paid"` → `OrderStatus.paid`; `"refunded"` → 422 listing the allowed values. `str, Enum` inheritance makes it serialize as its string value and show as an enum in OpenAPI. For a *literal* set without an Enum class: `status: Literal["pending", "paid"]`.

For state transitions (e.g., `shipped` can't go back to `pending`) validate in the *service layer*, not the schema — the schema knows the value is valid, only the domain knows whether the transition is.

---

## Q: How would you handle different API versions with different schemas?

Keep versions as separate packages and mount them as separate routers:

```
app/
  api/
    v1/
      schemas.py      # UserReadV1: {id, name}
      routes.py
    v2/
      schemas.py      # UserReadV2: {id, first_name, last_name}
      routes.py
  domain/             # shared ORM + services (version-agnostic)
```

```python
from app.api.v1.routes import router as v1
from app.api.v2.routes import router as v2

app.include_router(v1, prefix="/api/v1")
app.include_router(v2, prefix="/api/v2")
```
Each version's route maps the same domain object to its own schema. `v1`'s `UserReadV1` gets a `@computed_field` or `model_validator` that derives `name` from `first_name + last_name`, so v1 clients keep working while the domain model moves on. When v1 usage hits zero (measured via metrics), delete the package.

---

# 4. Dependency Injection

## Q: What is dependency injection in FastAPI?

**Plain English:** Instead of your endpoint *building* the things it needs (DB session, current user, settings), it *declares* that it needs them, and FastAPI builds and passes them in. "Don't call us, we'll call you."

**First principles:** Endpoints have cross-cutting needs — auth, DB, config, pagination parsing. If each endpoint constructs these itself, you get duplication and no single place to change behaviour. DI inverts control: the framework owns construction, the handler owns logic. The bonus is testability — you can swap any dependency without touching the endpoint.

---

## Q: What is Depends()?

`Depends(callable)` is the marker that tells FastAPI: "call this callable, resolving *its* parameters the same way, and pass the result here."

```python
from fastapi import Depends

def pagination(page: int = 1, size: int = 20) -> dict:      # 1
    return {"offset": (page - 1) * size, "limit": size}

@app.get("/items")
async def list_items(p: Annotated[dict, Depends(pagination)]):   # 2
    return repo.list(**p)
```
1. A plain function. Its params (`page`, `size`) are query parameters — FastAPI parses them as if they were on the endpoint.
2. `Depends(pagination)` runs `pagination(...)` and injects the dict as `p`. The docs show `page` and `size` for this endpoint even though they aren't in its signature.

---

## Q: Why would you use dependencies?

- **Reuse:** one `get_current_user` for 100 endpoints.
- **Composition:** dependencies can depend on other dependencies.
- **Testability:** `app.dependency_overrides` swaps implementations.
- **Documentation:** dependency params appear in OpenAPI.
- **Lifecycle:** `yield` dependencies handle setup/teardown (open/close a session).
- **Route-level enforcement:** `dependencies=[Depends(require_admin)]` on a router protects every route in it.

---

## Q: How do you create a reusable dependency?

Put it in a module (`app/api/deps.py`) and alias the annotation so call sites stay short:

```python
# app/api/deps.py
from typing import Annotated
from fastapi import Depends
from sqlalchemy.ext.asyncio import AsyncSession
from app.db.session import SessionLocal

async def get_db() -> AsyncIterator[AsyncSession]:     # 1
    async with SessionLocal() as session:              # 2
        yield session                                  # 3

DB = Annotated[AsyncSession, Depends(get_db)]          # 4

# app/api/v1/users.py
@router.get("/{user_id}")
async def get_user(user_id: int, db: DB):              # 5
    ...
```
1. An async generator dependency.
2. Opens a session from the sessionmaker.
3. Everything before `yield` runs before the endpoint; the yielded value is injected. Code after (implicit `__aexit__`) runs after the response.
4. `Annotated` alias — the reusable "type".
5. Endpoint just says `db: DB`.

---

## Q: Can dependencies depend on other dependencies?

Yes — this is the core power. FastAPI builds a **DAG** and resolves it depth-first, caching each dependency's result per request.

```python
async def get_token(authorization: Annotated[str, Header()]) -> str:      # depends on Header
    return authorization.removeprefix("Bearer ")

async def get_current_user(token: Annotated[str, Depends(get_token)], db: DB) -> User:  # depends on get_token + get_db
    ...

async def require_admin(user: Annotated[User, Depends(get_current_user)]) -> User:      # depends on get_current_user
    if not user.is_admin:
        raise HTTPException(403)
    return user
```
If an endpoint uses both `get_current_user` and `require_admin`, `get_current_user` is called **once** — the second use gets the cached result (`use_cache=True` default).

---

## Q: How do you use dependencies for authentication?

```python
from fastapi.security import OAuth2PasswordBearer
from jose import jwt, JWTError

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="/auth/token")     # 1

async def get_current_user(
    token: Annotated[str, Depends(oauth2_scheme)],                # 2
    db: DB,
) -> User:
    try:
        payload = jwt.decode(token, settings.jwt_secret, algorithms=["HS256"])   # 3
    except JWTError:
        raise HTTPException(401, "Invalid token", headers={"WWW-Authenticate": "Bearer"})  # 4
    user = await db.get(User, int(payload["sub"]))               # 5
    if user is None or not user.is_active:
        raise HTTPException(401, "Inactive user")
    return user

CurrentUser = Annotated[User, Depends(get_current_user)]          # 6

@router.get("/me")
async def me(user: CurrentUser):                                  # 7
    return user
```
1. `OAuth2PasswordBearer` is itself a dependency: it reads the `Authorization: Bearer <token>` header, returns the token string, or raises 401. It also registers the security scheme in OpenAPI so Swagger shows an "Authorize" button.
2. Inject the raw token.
3. Verify signature and expiry.
4. `WWW-Authenticate` header is required by the HTTP spec for 401.
5. Load the user (`sub` = subject claim).
6. Alias for reuse.
7. Any endpoint declaring `CurrentUser` is protected.

---

## Q: How do you inject a database session?

Shown in `get_db` above. Key points:
- Use a **yield** dependency so the session is closed even if the endpoint raises.
- Put `commit` in the *service/unit-of-work*, not in `get_db`, unless you deliberately want "commit-on-success" semantics:

```python
async def get_db() -> AsyncIterator[AsyncSession]:
    async with SessionLocal() as session:
        try:
            yield session
            await session.commit()          # commit only if endpoint succeeded
        except Exception:
            await session.rollback()        # any exception → rollback
            raise
```

---

## Q: How do you share configuration through dependencies?

```python
# app/core/config.py
from functools import lru_cache
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env", env_prefix="APP_")
    database_url: str
    jwt_secret: SecretStr
    debug: bool = False

@lru_cache                                          # 1
def get_settings() -> Settings:
    return Settings()                               # 2

Config = Annotated[Settings, Depends(get_settings)]

@router.get("/info")
async def info(cfg: Config):
    return {"debug": cfg.debug}
```
1. `lru_cache` makes `Settings()` load once per process, not per request.
2. Reads env vars / `.env` and validates them at first access (fail fast on misconfiguration).

In tests: `app.dependency_overrides[get_settings] = lambda: Settings(database_url="sqlite://")`.

---

## Q: What is the lifecycle of a dependency?

Per request:
1. FastAPI builds the dependency graph for the matched route (this is computed once at startup and cached).
2. Resolves each dependency depth-first. Results are cached **within the request** by default.
3. For `yield` dependencies, runs the code up to `yield`.
4. Runs the endpoint.
5. Sends the response.
6. Runs the code after `yield` (teardown) in reverse order.

Important nuance: since FastAPI 0.106, exit code of `yield` dependencies runs **after** the response is sent, so you can't modify the response there, and a background task can't use the same session. Errors in the endpoint *are* propagated into the `yield` dependency (so `try/except` around `yield` works for rollback).

Dependencies are **not** singletons unless you make them so (`lru_cache`, module-level object). There is no application-scoped DI container; use `lifespan` + `app.state` for that.

---

## Q: What is the difference between dependency injection and middleware?

| | Dependency | Middleware |
|---|---|---|
| Scope | Per route (opt-in) | Every request (global) |
| Access | Parsed params, typed values, other deps | Raw `Request`/`Response`, ASGI scope |
| Can return values to handler | ✅ | ❌ (only via `request.state`) |
| Appears in OpenAPI | ✅ | ❌ |
| Runs before routing | ❌ | ✅ |
| Typical use | Auth user, DB session, pagination | Logging, CORS, tracing, compression |

**Rule:** if the handler needs the *value* → dependency. If it's cross-cutting and route-agnostic → middleware.

---

## Q: How do you override dependencies during testing?

```python
# tests/conftest.py
import pytest
from httpx import AsyncClient, ASGITransport
from app.main import app
from app.api.deps import get_db, get_current_user

@pytest.fixture
async def client(test_session):
    async def _db():                                   # 1
        yield test_session
    async def _user():                                 # 2
        return User(id=1, email="t@t.com", is_admin=True)

    app.dependency_overrides[get_db] = _db             # 3
    app.dependency_overrides[get_current_user] = _user
    async with AsyncClient(transport=ASGITransport(app=app), base_url="http://test") as c:
        yield c
    app.dependency_overrides.clear()                   # 4
```
1. Replacement returns the test session (e.g., inside a transaction that's rolled back).
2. Skip real JWT verification — inject a fake user.
3. The dict maps the *original callable* → replacement. FastAPI consults it at resolve time.
4. Always clear so tests don't leak into each other.

---

## Q: How do you create class-based dependencies?

Any callable works. A class is useful when the dependency has configuration or state:

```python
class RateLimiter:                                    # 1
    def __init__(self, max_per_minute: int):
        self.max = max_per_minute

    async def __call__(self, request: Request, redis: Redis = Depends(get_redis)):   # 2
        key = f"rl:{request.client.host}:{request.url.path}"
        n = await redis.incr(key)
        if n == 1:
            await redis.expire(key, 60)
        if n > self.max:
            raise HTTPException(429, "Too many requests")

@router.post("/login", dependencies=[Depends(RateLimiter(max_per_minute=5))])    # 3
async def login(...): ...
```
1. Constructor captures configuration.
2. `__call__` is what FastAPI invokes; its params are resolved like any dependency.
3. Instantiate with config; `dependencies=[...]` runs it without injecting a value.

Also: `Depends()` with no argument on a class-typed param → `item: Annotated[CommonParams, Depends()]` calls `CommonParams(...)` using its `__init__` signature.

---

## Q: Can dependencies be asynchronous?

Yes. `async def` dependencies are awaited on the loop; plain `def` dependencies run in the threadpool (same rule as endpoints). Mix freely. Keep the rule: if a dependency does I/O, make it async with an async client; if it does blocking I/O, keep it `def`.

---

## Q: How would you implement role-based authorization using dependencies?

Use a *dependency factory* so roles are parameters:

```python
from enum import Enum

class Role(str, Enum):
    viewer = "viewer"
    editor = "editor"
    admin = "admin"

def require_roles(*allowed: Role):                                     # 1
    async def _check(user: CurrentUser) -> User:                       # 2
        if user.role not in allowed:
            raise HTTPException(403, f"Requires one of {[r.value for r in allowed]}")
        return user
    return _check                                                      # 3

@router.delete("/items/{id}", dependencies=[Depends(require_roles(Role.admin))])   # 4
async def delete_item(id: int): ...

@router.put("/items/{id}")
async def edit_item(id: int, user: Annotated[User, Depends(require_roles(Role.editor, Role.admin))]):  # 5
    ...
```
1. Outer function takes the config.
2. Inner function is the actual dependency; it reuses `CurrentUser`.
3. Return the closure.
4. Guard without needing the user in the handler.
5. Guard *and* get the user.

For hierarchical roles, map to a rank (`viewer=1, editor=2, admin=3`) and check `user.rank >= required.rank`. For fine-grained rules, move to permission-based (next section).

---

## Q: What problems can occur if dependencies contain expensive operations?

- **Runs on every request** → if `get_settings` re-reads a file or `get_current_user` hits the DB for each of five dependencies (without caching), you multiply latency.
- **Blocking work in an `async` dependency** freezes the event loop for *all* requests.
- **Resource leaks** if a `yield` dependency opens a connection and teardown is skipped (e.g., `return` before `yield` or swallowing exceptions).
- **Hidden fan-out**: dependency trees can accidentally open multiple DB sessions per request.
- **Test slowness**: heavy deps make every test slow unless overridden.

**Mitigations:** `lru_cache` for process-level singletons; `use_cache=True` (default) for request-level; cache user lookups in Redis with short TTL; keep deps thin (no business logic); profile with tracing spans per dependency.

---

## Q: "You have 100 APIs and every API needs the authenticated user. How would you design this?"

This is a design question; walk through layers:

**1. One dependency, applied at the router level — not 100 times.**

```python
# app/api/deps.py
CurrentUser = Annotated[User, Depends(get_current_user)]

# app/api/v1/__init__.py
protected = APIRouter(dependencies=[Depends(get_current_user)])     # 1
protected.include_router(users.router, prefix="/users")
protected.include_router(orders.router, prefix="/orders")
# ... 98 more

public = APIRouter()                                                # 2
public.include_router(auth.router, prefix="/auth")
public.include_router(health.router)

app.include_router(public)
app.include_router(protected, prefix="/api/v1")
```
1. Every route under `protected` is authenticated even if the handler doesn't mention the user. Handlers that need the user add `user: CurrentUser` and get the cached result — no second lookup.
2. Explicit public router for login/health. **Secure by default:** adding a new router to `protected` can't accidentally be unauthenticated.

**2. Make `get_current_user` cheap.**
- Verify JWT signature locally (no network).
- Cache user records in Redis keyed by user ID, TTL 60s, so 100 APIs don't mean 100 DB hits per user.
- Or embed roles/tenant in the token claims so most endpoints never need the DB.

**3. Put the user on `request.state` too.**
A tiny middleware (or the dependency itself) sets `request.state.user_id` so logging/tracing middleware can tag every log line with the user without re-decoding.

**4. Consider an auth gateway.**
At scale, terminate auth at the API gateway (Kong, Envoy, AWS API Gateway + Cognito authorizer). The gateway validates the token and forwards trusted headers (`X-User-Id`, `X-Roles`). Services then have a trivial dependency that reads headers — but only accept these headers from the gateway's network (mTLS or private subnet), never from the internet.

**5. Testing.**
Single override point: `app.dependency_overrides[get_current_user] = fake_user`. All 100 APIs become testable without tokens.

**Interview summary:**
> "I'd centralize it as one `get_current_user` dependency, apply it at the router level so protection is opt-out not opt-in, keep it cheap with local JWT verification and a short-lived cache, expose the user via `request.state` for observability, and keep a single override hook for tests. At larger scale I'd push token validation to the gateway and have services trust forwarded identity headers over a private network."

---

# 5. Async Programming

## Q: What is asynchronous programming?

**Plain English:** A style where a single thread can *pause* a task while it waits for something slow (network, disk, DB) and *resume* another task in the meantime, instead of sitting idle.

**First principles:** A CPU executes billions of instructions per second; a network round-trip takes milliseconds — millions of instruction-times. Synchronous code wastes those cycles by blocking. Async code says: "I'm waiting; let someone else run." It's *cooperative multitasking* — tasks voluntarily hand control back at `await` points.

**Analogy:** A restaurant waiter (one thread) takes an order, hands it to the kitchen (I/O), and instead of standing at the pass, goes to serve other tables. When the food is ready, they come back.

---

## Q: What is asyncio?

Python's standard-library framework for writing async code: an **event loop**, **coroutines** (`async def`), **tasks**, **futures**, and primitives (`gather`, `sleep`, `Queue`, `Lock`, `Semaphore`). Uvicorn runs your FastAPI app inside an asyncio event loop (usually the faster `uvloop` implementation).

---

## Q: What does `async def` mean?

Declares a **coroutine function**. Calling it does *not* run the body — it returns a coroutine object. The body runs only when awaited or scheduled as a task.

```python
async def fetch():          # coroutine function
    return 1

c = fetch()                 # coroutine object — nothing has executed yet
result = await c            # now it runs, inside an event loop
```

---

## Q: What does `await` do?

Suspends the current coroutine until the awaited awaitable finishes, yielding control to the event loop so it can run other tasks. It can only appear inside `async def`. `await` is the *only* place a coroutine gives up control — code between awaits runs uninterrupted.

---

## Q: What is an event loop?

The scheduler. It keeps a queue of ready tasks and a set of I/O watchers (via `epoll`/`kqueue`). Loop: pick a ready task → run it until it awaits → register what it's waiting for → when the OS says the I/O is ready, mark that task ready → repeat. One loop per thread; Uvicorn has one per worker process.

---

## Q: What is the difference between synchronous and asynchronous code?

```python
# Synchronous — 3 sequential HTTP calls ≈ 3 × 200ms = 600ms
def sync_version():
    a = requests.get(url_a)      # thread blocks 200ms
    b = requests.get(url_b)      # blocks another 200ms
    c = requests.get(url_c)
    return a, b, c

# Asynchronous — 3 concurrent calls ≈ max(200ms) = 200ms
async def async_version():
    async with httpx.AsyncClient() as client:
        a, b, c = await asyncio.gather(          # all three in flight at once
            client.get(url_a), client.get(url_b), client.get(url_c)
        )
    return a, b, c
```
Sync: one thing at a time, simple, blocks. Async: interleaved waiting, more throughput per thread, requires async-aware libraries all the way down.

---

## Q: What is I/O-bound work?

Work whose duration is dominated by *waiting on something external*: HTTP calls, DB queries, file reads, `sleep`. The CPU is mostly idle. Async shines here.

## Q: What is CPU-bound work?

Work dominated by computation: image resizing, hashing, ML inference, JSON parsing of huge payloads, big loops. The CPU is busy. Async does **not** help — there's no waiting to overlap — and because of the GIL, threads don't help much either. Use **processes** (`ProcessPoolExecutor`, Celery workers) or native extensions (NumPy releases the GIL).

---

## Q: When should you use `async def` in FastAPI?

When the handler awaits **async-capable** I/O: `httpx.AsyncClient`, `asyncpg`/SQLAlchemy async, `aioredis`/`redis.asyncio`, `aiobotocore`, `aiofiles`. Also for trivial handlers that do no I/O (e.g., return a constant) — `async` avoids the threadpool hop.

## Q: When should you use normal `def`?

When the handler calls **blocking** code you can't or won't convert: `requests`, sync SQLAlchemy, `boto3`, `psycopg2`, CPU-heavy libraries, legacy SDKs. FastAPI runs `def` handlers in a threadpool, so they don't block the loop.

**Decision table:**

| Handler body | Declare as |
|---|---|
| Awaits async library | `async def` |
| Calls blocking library | `def` |
| Pure computation < ~1ms | `async def` (or `def`, doesn't matter) |
| Heavy CPU (>100ms) | `def` + offload to process pool / queue |
| Mixed | `async def` and wrap blocking calls in `run_in_threadpool` |

---

## Q: What happens if you call blocking code inside an async endpoint?

The **entire event loop freezes** for the duration. Every other in-flight request in that worker stalls — even health checks. Under load, p99 latency explodes, Kubernetes liveness probes time out, and pods get restarted. It's the #1 FastAPI production bug.

```python
@app.get("/bad")
async def bad():
    data = requests.get("https://slow.example.com")    # blocks loop for 2s → all requests wait 2s
    return data.json()
```

---

## Q: Why is `time.sleep()` problematic inside an async endpoint?

`time.sleep(5)` suspends the *thread*, and the event loop runs on that thread. Nothing else runs for 5 seconds. It's the purest example of blocking the loop.

## Q: What should you use instead?

`await asyncio.sleep(5)` — it registers a timer with the loop and yields control. Other requests proceed; after 5 s the loop resumes this coroutine.

For blocking *library* calls, wrap them:

```python
from fastapi.concurrency import run_in_threadpool

@app.get("/report")
async def report():
    pdf = await run_in_threadpool(generate_pdf_sync, report_id=7)     # 1
    return Response(pdf, media_type="application/pdf")
```
1. Runs `generate_pdf_sync` in Starlette's threadpool and awaits the result, keeping the loop free. (Also `asyncio.to_thread` in stdlib 3.9+.)

---

## Q: Can you use synchronous libraries inside FastAPI?

Yes, safely, in two ways: (a) declare the endpoint/dependency as plain `def`, or (b) use `run_in_threadpool` from inside `async def`. The threadpool default is 40 threads (AnyIO's `to_thread` limiter) — tune with `anyio.to_thread.current_default_thread_limiter().total_tokens = 100` in lifespan if you need more concurrency for blocking work.

---

## Q: How does FastAPI handle synchronous endpoints?

Starlette detects that the endpoint isn't a coroutine function and calls `run_in_threadpool(endpoint, **values)`. The loop awaits the future; the thread runs your code. Same for sync dependencies. So `def` endpoints are concurrent up to the threadpool size, and blocking inside them is fine.

---

## Q: What is an async database driver?

A driver that performs the DB protocol I/O with non-blocking sockets and exposes `await`-able methods: `asyncpg` (PostgreSQL, fastest), `aiomysql`, `aiosqlite`, `motor` (MongoDB). SQLAlchemy 2.0's `AsyncSession` sits on top of these via `create_async_engine("postgresql+asyncpg://...")`. A sync driver like `psycopg2` blocks the thread; using it in an `async def` endpoint freezes the loop.

---

## Q: How do you make HTTP calls asynchronously?

Use `httpx.AsyncClient` — and **reuse one client** for connection pooling:

```python
from contextlib import asynccontextmanager
import httpx

@asynccontextmanager
async def lifespan(app: FastAPI):                                  # 1
    app.state.http = httpx.AsyncClient(timeout=httpx.Timeout(5.0), limits=httpx.Limits(max_connections=100))   # 2
    yield                                                          # 3
    await app.state.http.aclose()                                  # 4

app = FastAPI(lifespan=lifespan)

def get_http(request: Request) -> httpx.AsyncClient:               # 5
    return request.app.state.http

@app.get("/weather")
async def weather(http: Annotated[httpx.AsyncClient, Depends(get_http)]):
    r = await http.get("https://api.weather.example/today")        # 6
    r.raise_for_status()
    return r.json()
```
1. Lifespan runs at startup/shutdown.
2. One pooled client per process, with explicit timeouts (never use the default no-timeout).
3. App serves requests.
4. Close on shutdown.
5. Dependency exposes the shared client.
6. Non-blocking call.

---

## Q: How do you handle concurrent external API calls?

`asyncio.gather` for fan-out, plus a `Semaphore` to bound concurrency and timeouts to bound waiting:

```python
sem = asyncio.Semaphore(10)                                        # 1

async def fetch_one(http, url):
    async with sem:                                                # 2
        r = await http.get(url, timeout=3.0)
        r.raise_for_status()
        return r.json()

@app.get("/aggregate")
async def aggregate(http: HttpDep):
    urls = [...]                                                   # 50 URLs
    results = await asyncio.gather(*(fetch_one(http, u) for u in urls), return_exceptions=True)   # 3
    ok = [r for r in results if not isinstance(r, Exception)]      # 4
    return {"ok": len(ok), "failed": len(results) - len(ok), "data": ok}
```
1. At most 10 in flight at once — protects the downstream from being hammered.
2. Acquire a slot; released on exit.
3. Launch all 50 concurrently; `return_exceptions=True` means one failure doesn't cancel the rest.
4. Partition successes and failures.

For a hard overall deadline: `async with asyncio.timeout(5): results = await asyncio.gather(...)`.

---

## Q: What is `asyncio.gather()`?

Runs multiple awaitables concurrently and returns their results in the *same order* as passed. With `return_exceptions=False` (default) the first exception propagates and the others keep running in the background (not cancelled). With `return_exceptions=True`, exceptions are returned as values. Alternative: `asyncio.TaskGroup` (3.11+) which cancels siblings on failure — safer structured concurrency.

---

## Q: How would you troubleshoot an API whose latency suddenly increased?

A structured runbook:

**1. Confirm and scope (2 min).**
- Which endpoints? All (infra) or some (code/data)? Check p50 vs p99 — p99 spike with flat p50 suggests contention/GC/lock; both up suggests a slow dependency.
- When did it start? Correlate with deploys, config changes, traffic patterns, cron jobs.

**2. Check the four golden signals on dashboards.**
- Latency, traffic, errors, saturation (CPU, memory, threadpool, DB pool usage).
- Look at *downstream* latency: DB query time, external API time, Redis time. Distributed tracing tells you which span grew.

**3. Common FastAPI-specific culprits.**
- **Blocked event loop**: someone added a sync call into an `async def` handler. Symptom: all endpoints slow, CPU low. Fix: `run_in_threadpool` or make it `def`.
- **DB pool exhaustion**: `pool_size` too small; requests wait for a connection. Symptom: `QueuePool limit reached` warnings, latency in steps of `pool_timeout`.
- **N+1 queries** after a new feature: query count per request jumps.
- **Missing index** after data growth: the query plan flipped to a seq scan.
- **Threadpool saturation**: many `def` endpoints, 40 threads all blocked.
- **External dependency degraded**: timeouts too generous → requests pile up. Add circuit breaker.
- **GC / memory pressure**: memory near limit → swapping / OOM restarts.

**4. Tools.**
- `py-spy dump --pid` on a live worker: shows what every thread/coroutine is doing right now.
- Postgres `pg_stat_statements` / `EXPLAIN ANALYZE`.
- Tracing (OpenTelemetry) waterfall for a slow request.
- Compare the deployed git SHA to the last-good one; `git diff`.

**5. Mitigate then fix.**
- Roll back if a deploy correlates.
- Scale out if saturation.
- Fix root cause; add an alert/test so it can't recur silently.

**Interview answer:** "First I scope it — which endpoints, since when, p50 vs p99, did a deploy happen. Then I look at saturation and downstream spans in tracing. For FastAPI specifically I check for blocking calls in async handlers and DB pool exhaustion, because they're the most common causes. I use py-spy on a live worker to see what it's actually doing, roll back if it correlates with a deploy, and then fix and add an alert."

---

# 6. Middleware

## Q: What is middleware?

**Plain English:** Code that wraps *every* request and response. It runs before the router sees the request and after the handler produces the response. Think of it as layers of an onion around your app.

**First principles:** Some concerns are orthogonal to business logic — logging, timing, CORS, compression, tracing. Putting them in every handler is duplication; putting them in the router is coupling. Middleware is the ASGI-level composition mechanism: each middleware is itself an ASGI app that holds a reference to the next one.

---

## Q: How do you create middleware in FastAPI?

**Option A — `@app.middleware("http")` (simple, function-based):**

```python
import time
from fastapi import Request

@app.middleware("http")                                           # 1
async def timing(request: Request, call_next):                    # 2
    start = time.perf_counter()
    response = await call_next(request)                           # 3
    elapsed_ms = (time.perf_counter() - start) * 1000
    response.headers["X-Response-Time-ms"] = f"{elapsed_ms:.1f}"  # 4
    return response                                               # 5
```
1. Registers an HTTP middleware (wraps Starlette's `BaseHTTPMiddleware`).
2. `call_next` is the next layer (ultimately the router).
3. Await downstream — this is where the handler runs.
4. Mutate the response.
5. Return it up the chain.

**Option B — Pure ASGI middleware (faster, works with streaming/WebSockets):**

```python
class TimingMiddleware:
    def __init__(self, app):                                      # 1
        self.app = app

    async def __call__(self, scope, receive, send):               # 2
        if scope["type"] != "http":                               # 3
            return await self.app(scope, receive, send)
        start = time.perf_counter()

        async def send_wrapper(message):                          # 4
            if message["type"] == "http.response.start":
                elapsed = (time.perf_counter() - start) * 1000
                headers = MutableHeaders(scope=message)           # 5
                headers["X-Response-Time-ms"] = f"{elapsed:.1f}"
            await send(message)

        await self.app(scope, receive, send_wrapper)              # 6

app.add_middleware(TimingMiddleware)
```
1. Receives the next ASGI app.
2. Standard ASGI signature.
3. Pass through non-HTTP (WebSocket, lifespan) untouched.
4. Intercept outgoing messages.
5. Inject header into the response-start message.
6. Call downstream with the wrapped `send`.

`BaseHTTPMiddleware` is convenient but has known costs: it buffers streaming responses in some versions, breaks `contextvars` propagation in older Starlette, and adds overhead. For production-critical middleware prefer pure ASGI.

---

## Q: What is the execution order of middleware?

Middleware added **last** is **outermost** — it runs first on the way in and last on the way out.

```python
app.add_middleware(A)
app.add_middleware(B)
app.add_middleware(C)
# Request:  C → B → A → router → handler
# Response: handler → A → B → C
```
This is because each `add_middleware` wraps the existing stack. Practical order (outermost first): error handling / tracing → CORS → auth/context → logging → compression → app.

---

## Q: How do you implement request logging?

```python
import logging, uuid, time
from starlette.middleware.base import BaseHTTPMiddleware

log = logging.getLogger("access")

class AccessLogMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request, call_next):
        start = time.perf_counter()
        try:
            response = await call_next(request)
        except Exception:
            log.exception("request_failed", extra={"path": request.url.path})   # 1
            raise
        log.info(
            "request",
            extra={                                                            # 2
                "method": request.method,
                "path": request.url.path,
                "status": response.status_code,
                "duration_ms": round((time.perf_counter() - start) * 1000, 2),
                "client": request.client.host if request.client else None,
                "request_id": getattr(request.state, "request_id", None),
            },
        )
        return response
```
1. Log unhandled exceptions with stack trace; re-raise so error handlers still run.
2. Structured fields (use a JSON formatter such as `python-json-logger`) so logs are queryable. Never log raw bodies or `Authorization` headers.

---

## Q: How do you measure request latency?

Shown above with `time.perf_counter()` (monotonic, high resolution). Emit it as: a response header (debugging), a structured log field, and a **Prometheus histogram** labelled by route template and status:

```python
from prometheus_client import Histogram
REQ_LATENCY = Histogram("http_request_duration_seconds", "Latency", ["method", "route", "status"],
                        buckets=(.005, .01, .025, .05, .1, .25, .5, 1, 2.5, 5))
# in middleware after call_next:
route = request.scope.get("route")
REQ_LATENCY.labels(request.method, route.path if route else "unmatched", response.status_code).observe(elapsed)
```
Label by the **route template** (`/users/{id}`), not the raw path, or you'll create unbounded label cardinality.

---

## Q: How do you add correlation IDs?

```python
import contextvars
request_id_ctx = contextvars.ContextVar("request_id", default="-")     # 1

class RequestIDMiddleware:
    def __init__(self, app): self.app = app
    async def __call__(self, scope, receive, send):
        if scope["type"] != "http":
            return await self.app(scope, receive, send)
        headers = dict(scope["headers"])
        rid = headers.get(b"x-request-id", b"").decode() or str(uuid.uuid4())   # 2
        token = request_id_ctx.set(rid)                                          # 3
        async def send_wrapper(message):
            if message["type"] == "http.response.start":
                MutableHeaders(scope=message)["X-Request-ID"] = rid              # 4
            await send(message)
        try:
            await self.app(scope, receive, send_wrapper)
        finally:
            request_id_ctx.reset(token)                                          # 5

class RequestIDFilter(logging.Filter):                                            # 6
    def filter(self, record):
        record.request_id = request_id_ctx.get()
        return True
```
1. A `ContextVar` is per-task storage — safe under asyncio (unlike a global).
2. Honour an incoming ID (from the gateway/client) or mint one.
3. Store it for this request's task context.
4. Echo it back so clients can report it.
5. Reset to avoid leaking across tasks.
6. A logging filter injects it into every log record automatically → every line for a request shares an ID.

Propagate it to downstream calls: `http.get(url, headers={"X-Request-ID": request_id_ctx.get()})`.

---

## Q: What is CORS?

**Cross-Origin Resource Sharing.** Browsers enforce the *same-origin policy*: JavaScript on `https://app.example.com` can't read responses from `https://api.example.com` unless the API explicitly allows it via headers. CORS is that allow-list mechanism. It protects users (prevents a malicious site from using your logged-in cookies to call your API and read the result), not your server — `curl` ignores it.

**Preflight:** for non-simple requests (JSON body, custom headers, PUT/DELETE), the browser first sends `OPTIONS` with `Access-Control-Request-Method`; the server must answer with `Access-Control-Allow-*` headers.

## Q: How do you implement CORS?

```python
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://app.example.com", "https://admin.example.com"],   # 1
    allow_origin_regex=r"https://.*\.preview\.example\.com",                 # 2
    allow_credentials=True,                                                  # 3
    allow_methods=["GET", "POST", "PUT", "PATCH", "DELETE"],
    allow_headers=["Authorization", "Content-Type", "X-Request-ID"],
    expose_headers=["X-Request-ID"],                                         # 4
    max_age=600,                                                             # 5
)
```
1. Explicit origins. Never `["*"]` in production with credentials — browsers reject that combination anyway.
2. Regex for dynamic preview environments.
3. Allow cookies / `Authorization`.
4. Headers JS is allowed to *read* from the response.
5. Cache preflight for 10 minutes.

Add CORS middleware **last** (outermost) so preflights are answered even if auth middleware would otherwise reject them.

---

## Q: How do you add security headers?

```python
class SecurityHeadersMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request, call_next):
        resp = await call_next(request)
        resp.headers.setdefault("Strict-Transport-Security", "max-age=63072000; includeSubDomains; preload")  # force HTTPS
        resp.headers.setdefault("X-Content-Type-Options", "nosniff")        # stop MIME sniffing
        resp.headers.setdefault("X-Frame-Options", "DENY")                  # anti-clickjacking
        resp.headers.setdefault("Referrer-Policy", "no-referrer")
        resp.headers.setdefault("Content-Security-Policy", "default-src 'none'; frame-ancestors 'none'")
        resp.headers.setdefault("Cache-Control", "no-store")                # for APIs returning private data
        return resp
```
Often these are set at the ingress/CDN instead, but defense in depth costs nothing. `secure` library packages these for you.

---

## Q: How do you handle exceptions globally?

```python
from fastapi import Request
from fastapi.responses import JSONResponse
from fastapi.exceptions import RequestValidationError

class AppError(Exception):                                          # 1
    status_code = 400
    code = "app_error"
    def __init__(self, message: str): self.message = message

class NotFound(AppError):
    status_code = 404; code = "not_found"

@app.exception_handler(AppError)                                    # 2
async def app_error_handler(request: Request, exc: AppError):
    return JSONResponse(exc.status_code, {"error": {"code": exc.code, "message": exc.message,
                                                     "request_id": request_id_ctx.get()}})

@app.exception_handler(RequestValidationError)                      # 3
async def validation_handler(request: Request, exc: RequestValidationError):
    return JSONResponse(422, {"error": {"code": "validation_error", "details": exc.errors()}})

@app.exception_handler(Exception)                                   # 4
async def unhandled_handler(request: Request, exc: Exception):
    log.exception("unhandled", extra={"request_id": request_id_ctx.get()})
    return JSONResponse(500, {"error": {"code": "internal_error", "message": "Something went wrong",
                                        "request_id": request_id_ctx.get()}})
```
1. A domain exception hierarchy — services raise these, unaware of HTTP.
2. One handler maps the whole hierarchy to a consistent error envelope.
3. Override the default 422 shape to match your envelope.
4. Catch-all: log with stack trace, hide internals from the client, return the request ID for support. Note: the `Exception` handler runs inside `ServerErrorMiddleware`, i.e., *outside* your other middleware, so `BaseHTTPMiddleware` sees the exception re-raised — use `try/except` there if needed.

---

## Q: Middleware vs dependency — when would you use each?

Covered in the DI section. Short version: **middleware** for things that apply to every request and don't need parsed parameters (logging, tracing, CORS, compression, request ID). **Dependency** for things that produce a value the handler uses or that vary per route (current user, DB session, pagination, permission checks). Auth is the interesting case: *token parsing* can be either; *loading the user object and injecting it* is a dependency.

---

## Q: How can middleware affect application performance?

- Every middleware adds a coroutine frame and, for `BaseHTTPMiddleware`, an extra task + memory stream per request (~50–100 µs and more memory).
- Reading the request body in middleware buffers it — costly for uploads, and it breaks streaming.
- Blocking calls in middleware (sync logging to a slow sink, sync Redis) freeze the loop for *everything*.
- Unbounded work (e.g., logging full bodies, regex over headers) scales with payload size.
- Ordering: put cheap "reject early" middleware (rate limit, auth) outside expensive ones so rejected requests don't pay for logging/compression.

**Measure**: run a load test with and without each middleware; typical budget is < 1 ms total middleware overhead.

---

## Q: How do you exclude specific routes from middleware logic?

Middleware runs before routing, so it can't know the route object yet — but it knows the path:

```python
SKIP = {"/health", "/metrics", "/docs", "/openapi.json"}

class AccessLogMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request, call_next):
        if request.url.path in SKIP or request.url.path.startswith("/internal/"):
            return await call_next(request)
        ...
```
Alternatives: mount noisy endpoints on a sub-application (`app.mount("/internal", internal_app)`) that lacks the middleware; or flip it around and use a **dependency** on the routers that need the behaviour.

---

## Q: How do you implement request/response logging without logging sensitive information?

1. **Don't log bodies by default.** Log method, path template, status, duration, request ID, user ID.
2. If bodies are needed (debug env only), **redact** with an allow-list of fields, not a deny-list:

```python
REDACT = {"password", "token", "authorization", "card_number", "ssn", "secret"}

def redact(obj):
    if isinstance(obj, dict):
        return {k: ("***" if k.lower() in REDACT else redact(v)) for k, v in obj.items()}
    if isinstance(obj, list):
        return [redact(i) for i in obj]
    return obj
```
3. Strip sensitive headers: `Authorization`, `Cookie`, `X-API-Key`.
4. Truncate to a max size (e.g., 2 KB) to avoid log-volume attacks.
5. Mask query strings that carry tokens (`?token=`).
6. Use `SecretStr` in Pydantic models so accidental `repr` doesn't leak.
7. Configure the log pipeline (Fluent Bit / Vector) with a second redaction pass — defense in depth.

---

## Q: How would you implement distributed tracing middleware?

Don't hand-roll it — use OpenTelemetry's FastAPI instrumentation, which *is* an ASGI middleware:

```python
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.sdk.resources import Resource
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor
from opentelemetry.instrumentation.httpx import HTTPXClientInstrumentor
from opentelemetry.instrumentation.sqlalchemy import SQLAlchemyInstrumentor

provider = TracerProvider(resource=Resource.create({"service.name": "orders-api"}))   # 1
provider.add_span_processor(BatchSpanProcessor(OTLPSpanExporter(endpoint="http://otel-collector:4317")))  # 2
trace.set_tracer_provider(provider)

FastAPIInstrumentor.instrument_app(app, excluded_urls="health,metrics")   # 3
HTTPXClientInstrumentor().instrument()                                    # 4
SQLAlchemyInstrumentor().instrument(engine=engine.sync_engine)            # 5
```
1. Identifies this service in traces.
2. Batches spans and ships them to a collector (which forwards to Jaeger/Tempo/X-Ray).
3. Adds middleware that: extracts `traceparent` from incoming headers (W3C Trace Context), starts a server span per request, records route/status, and sets the span as current in the context.
4. Every outgoing `httpx` call becomes a child span **and injects `traceparent`** so the next service continues the same trace.
5. Every SQL statement becomes a span.

If you had to write it manually, the middleware would: (a) `propagator.extract(headers)` → context, (b) `tracer.start_as_current_span(name, context=ctx, kind=SERVER)`, (c) set attributes, (d) on outgoing calls `propagator.inject(headers)`. Add `trace_id` to the log context so logs and traces link.

---

# 7. Authentication & Authorization

## Q: How do you implement authentication in FastAPI?

FastAPI gives you *security scheme* helpers (`fastapi.security`) that read credentials from the request and document them in OpenAPI. You combine one with a dependency that verifies the credential and returns the principal. A full JWT flow:

```python
# app/core/security.py
from datetime import datetime, timedelta, timezone
from jose import jwt
from passlib.context import CryptContext

pwd = CryptContext(schemes=["argon2"], deprecated="auto")                  # 1

def hash_password(p: str) -> str: return pwd.hash(p)
def verify_password(p: str, hashed: str) -> bool: return pwd.verify(p, hashed)

def create_access_token(sub: str, roles: list[str], minutes: int = 15) -> str:
    now = datetime.now(timezone.utc)
    claims = {"sub": sub, "roles": roles, "iat": now, "exp": now + timedelta(minutes=minutes), "type": "access"}  # 2
    return jwt.encode(claims, settings.jwt_secret.get_secret_value(), algorithm="HS256")   # 3
```
1. Argon2 (or bcrypt) — slow, salted hashing. Never store plaintext or SHA-256.
2. Standard claims: subject, issued-at, expiry; plus roles and a `type` to distinguish access vs refresh.
3. Sign with a secret (HS256) or private key (RS256).

```python
# app/api/v1/auth.py
from fastapi.security import OAuth2PasswordRequestForm

@router.post("/token")
async def login(form: Annotated[OAuth2PasswordRequestForm, Depends()], db: DB):   # 1
    user = await db.scalar(select(User).where(User.email == form.username))     # 2
    if not user or not verify_password(form.password, user.hashed_password):     # 3
        raise HTTPException(401, "Incorrect email or password")                  # 4
    return {"access_token": create_access_token(str(user.id), user.roles), "token_type": "bearer"}   # 5
```
1. Parses `application/x-www-form-urlencoded` with `username`/`password` fields (OAuth2 spec).
2. Look up the user.
3. Constant-time verify. Always run verify even if user missing (or fake it) to avoid timing-based user enumeration.
4. Same message for both failures — don't reveal which was wrong.
5. Response shape required by the OAuth2 password flow.

The `get_current_user` dependency from Section 4 completes the loop.

---

## Q: What is JWT?

**JSON Web Token** — a compact, URL-safe string with three base64url parts: `header.payload.signature`.
- **Header**: algorithm (`HS256`, `RS256`) and type.
- **Payload**: claims (`sub`, `exp`, `iat`, custom).
- **Signature**: HMAC or RSA/ECDSA signature over header+payload.

The payload is **not encrypted** — anyone can decode it. The signature guarantees it wasn't *altered*. So never put secrets in a JWT.

---

## Q: How does JWT authentication work?

1. Client sends credentials to `/token`.
2. Server verifies and issues a signed JWT with an expiry.
3. Client stores it and sends `Authorization: Bearer <jwt>` on each request.
4. Server verifies the signature and `exp` **locally** — no DB or session store lookup — and trusts the claims.
5. When it expires, the client uses a refresh token to get a new one.

**First principles:** Session cookies require server-side state (a session table) shared by all instances. JWTs move the state to the client and make it tamper-proof with cryptography. That's why they scale horizontally with no shared session store — and why revoking one is hard (the server has nothing to delete).

---

## Q: What is the difference between authentication and authorization?

- **Authentication (AuthN)** – *Who are you?* Verifying identity (password, token, certificate). Failure → **401 Unauthorized**.
- **Authorization (AuthZ)** – *What are you allowed to do?* Checking permissions for an identified principal. Failure → **403 Forbidden**.

Order matters: authenticate first, authorize second. A 401 says "I don't know you"; a 403 says "I know you, and no."

---

## Q: Where should JWT tokens be stored?

Browser clients — the trade-off is **XSS vs CSRF**:

| Storage | XSS risk | CSRF risk | Notes |
|---|---|---|---|
| `localStorage` | ❌ Readable by any injected script | ✅ Not auto-sent | Simplest, most common, most risky |
| In-memory (JS variable) | Lower (lost on refresh) | ✅ | Pair with refresh-token cookie |
| `HttpOnly; Secure; SameSite=Strict` cookie | ✅ JS can't read it | ❌ Auto-sent → need CSRF token / SameSite | **Recommended** |

**Best practice:** short-lived access token in memory (or HttpOnly cookie), refresh token in an `HttpOnly; Secure; SameSite=Strict` cookie scoped to the refresh path, plus CSRF protection if cookies are used. Mobile apps: OS keychain / keystore. Server-to-server: secrets manager, never in code.

---

## Q: What are access tokens and refresh tokens?

- **Access token**: short-lived (5–15 min), sent on every API request, verified statelessly. If stolen, the damage window is short.
- **Refresh token**: long-lived (days–weeks), sent *only* to the token endpoint to obtain a new access token. Stored server-side (or its hash), so it **can** be revoked. Rotated on each use.

Why two? Short access tokens limit exposure; refresh tokens keep the UX seamless without re-login; the server-side refresh record restores revocability that stateless JWTs lack.

```python
@router.post("/refresh")
async def refresh(refresh_token: Annotated[str, Cookie()], db: DB):
    payload = decode(refresh_token, expected_type="refresh")                   # 1
    record = await db.get(RefreshToken, payload["jti"])                        # 2
    if record is None or record.revoked or record.user_id != int(payload["sub"]):
        await revoke_all_for_user(db, payload["sub"])                          # 3
        raise HTTPException(401, "Invalid refresh token")
    record.revoked = True                                                      # 4
    new_refresh = await issue_refresh(db, record.user_id)                      # 5
    resp = JSONResponse({"access_token": create_access_token(...), "token_type": "bearer"})
    resp.set_cookie("refresh_token", new_refresh, httponly=True, secure=True, samesite="strict", path="/auth/refresh")  # 6
    return resp
```
1. Verify signature and that it's a refresh token.
2. `jti` (JWT ID) links to a DB row.
3. **Reuse detection**: a revoked token presented again means theft → kill the whole family.
4. Single-use.
5. Issue a replacement (rotation).
6. Cookie scoped tightly.

---

## Q: How do you implement OAuth2 with FastAPI?

FastAPI supports OAuth2 in two roles:

**As a resource server/authorization server (password flow, first-party apps)** — shown above with `OAuth2PasswordBearer` + `OAuth2PasswordRequestForm`.

**As a client of an external provider (Authorization Code flow with Google/GitHub/Cognito)** — use `authlib`:

```python
from authlib.integrations.starlette_client import OAuth
oauth = OAuth()
oauth.register("google", client_id=..., client_secret=..., server_metadata_url="https://accounts.google.com/.well-known/openid-configuration", client_kwargs={"scope": "openid email profile"})

@router.get("/login/google")
async def login_google(request: Request):
    return await oauth.google.authorize_redirect(request, redirect_uri=settings.callback_url)   # 1

@router.get("/callback/google")
async def callback(request: Request):
    token = await oauth.google.authorize_access_token(request)   # 2
    userinfo = token["userinfo"]                                 # 3
    user = await upsert_user(userinfo["email"])
    return {"access_token": create_access_token(str(user.id), user.roles)}   # 4
```
1. Redirect the browser to Google with `state` + PKCE.
2. Exchange the `code` for tokens (server-to-server), verify `state`.
3. OIDC ID-token claims.
4. Issue *your own* session/JWT — don't reuse Google's token as your API's credential.

---

## Q: What is OAuth2 Password Flow?

The user gives their username/password *directly* to the client app, which POSTs them to the token endpoint and gets an access token. It's the simplest flow and what `OAuth2PasswordBearer` implements. **The OAuth 2.1 draft deprecates it** because it teaches users to type passwords into third-party apps. Fine for first-party apps (your own frontend talking to your own API); use Authorization Code + PKCE for anything else.

---

## Q: What is OpenID Connect?

**OIDC** is an identity layer on top of OAuth2. OAuth2 alone says "this app may access these resources" (authorization); OIDC adds "and this is who the user is" via a signed **ID token** (a JWT with `sub`, `email`, `name`…), a `/userinfo` endpoint, and a **discovery document** (`/.well-known/openid-configuration`) that publishes endpoints and the **JWKS** (public keys). Google, Cognito, Auth0, Okta, Keycloak are OIDC providers.

---

## Q: How do you implement role-based access control?

Covered via `require_roles(...)` in Section 4. Key design points:
- Store roles on the user (or in a `user_roles` table for many-to-many).
- Put roles in the JWT to avoid DB hits — but then role changes take effect only after token expiry (acceptable with 15-min tokens; force re-login for demotions).
- Apply at router level for coarse control (`admin` router), at route level for exceptions.
- RBAC breaks down when you need "editors can edit *their own* posts" → that's resource-level authorization.

---

## Q: How do you implement permission-based authorization?

Permissions are finer-grained than roles: `orders:read`, `orders:write`, `users:delete`. Roles become *bundles* of permissions.

```python
ROLE_PERMS = {
    "viewer": {"orders:read"},
    "editor": {"orders:read", "orders:write"},
    "admin":  {"orders:read", "orders:write", "users:delete"},
}

def require_perm(perm: str):
    async def _check(user: CurrentUser) -> User:
        perms = set().union(*(ROLE_PERMS.get(r, set()) for r in user.roles))    # 1
        if perm not in perms:
            raise HTTPException(403, f"Missing permission: {perm}")
        return user
    return _check

@router.delete("/users/{id}", dependencies=[Depends(require_perm("users:delete"))])
async def delete_user(id: int): ...
```
1. Union of all permissions across the user's roles.

For **resource-level** rules ("owner or admin"), check inside the service with the loaded entity:

```python
async def get_order_for_user(order_id: int, user: CurrentUser, db: DB) -> Order:
    order = await db.get(Order, order_id)
    if order is None: raise NotFound("order")
    if order.owner_id != user.id and "admin" not in user.roles:
        raise HTTPException(404)      # 404 not 403 — don't reveal existence
    return order
```
For complex policies use a policy engine (OPA / Cerbos / Casbin) so rules live in one auditable place.

---

## Q: How do you expire JWT tokens?

Set the `exp` claim at creation; verification libraries reject expired tokens automatically (`jwt.decode` raises `ExpiredSignatureError`). Keep access tokens short (5–15 min). Optionally validate `nbf` (not before) and allow small `leeway` (30 s) for clock skew.

---

## Q: How do you revoke a JWT?

Stateless JWTs *can't* be revoked without adding state. Options, from lightest to heaviest:

1. **Short expiry + refresh-token revocation** (recommended default): revoke the refresh token in the DB; the access token dies within minutes.
2. **Denylist by `jti`** in Redis with TTL = remaining lifetime: on logout/compromise, `SET denylist:<jti> 1 EX <seconds_left>`. `get_current_user` checks it (one Redis GET per request).
3. **Token version on the user**: store `token_version` in DB and in the claim; bump it to invalidate all tokens for that user. Requires a DB/cache read per request.
4. **Switch to opaque tokens** (random string → server lookup). You've reinvented sessions, but gained instant revocation.

---

## Q: What are the security risks of JWT?

- **`alg: none` / algorithm confusion**: always pass `algorithms=[...]` explicitly on decode; never trust the header's `alg`.
- **Weak HMAC secrets**: brute-forceable. Use ≥256-bit random secrets or asymmetric keys.
- **No revocation**: stolen token valid until expiry.
- **Sensitive data in payload**: it's only base64 — readable by anyone.
- **Long expiry**: bigger blast radius.
- **XSS theft** when stored in `localStorage`.
- **Missing audience/issuer checks**: a token from another service accepted here. Validate `aud` and `iss`.
- **Key rotation neglect**: use `kid` + JWKS so keys can rotate.
- **Replay** across services if the same secret is shared — use per-service audiences.

---

## Q: What is token rotation?

Issuing a **new** token each time one is used (for refresh tokens) or on a schedule (for signing keys).
- *Refresh-token rotation*: each refresh returns a new refresh token and invalidates the old. If an old one is ever reused, that's evidence of theft → revoke the whole chain (shown above).
- *Key rotation*: publish multiple public keys in JWKS with `kid`; sign with the newest; keep old ones for verification until all tokens signed with them have expired.

---

## Q: How would you secure an internal FastAPI service?

Defense in depth — assume the network isn't enough:
1. **Network**: private subnet / cluster-internal `Service`, `NetworkPolicy` allowing only known callers, no public ingress.
2. **Transport**: mTLS via a service mesh (Istio/Linkerd) or the load balancer — both sides prove identity with certs.
3. **Service identity**: caller presents a JWT issued by an internal IdP (or SPIFFE/SPIRE SVID) with `aud = orders-service`; verify it in a dependency.
4. **Least privilege**: per-caller scopes (`orders:read` only for the reporting service).
5. **No shared static API keys** in env vars; if unavoidable, rotate via Secrets Manager and scope them.
6. **Rate limits and timeouts** even internally — a runaway caller is as damaging as an attacker.
7. **Audit logging** of principal on every request.
8. **Disable docs** (`docs_url=None`) or protect them.

---

## Q: How do you validate tokens issued by an external identity provider?

Fetch the provider's **JWKS** (public keys) via OIDC discovery, cache it, verify signature + `iss` + `aud` + `exp`:

```python
import httpx, time
from jose import jwt

class JWKSVerifier:
    def __init__(self, issuer: str, audience: str):
        self.issuer, self.audience = issuer, audience
        self._keys: dict | None = None
        self._fetched = 0.0

    async def keys(self) -> dict:
        if self._keys is None or time.time() - self._fetched > 3600:          # 1
            async with httpx.AsyncClient(timeout=5) as c:
                cfg = (await c.get(f"{self.issuer}/.well-known/openid-configuration")).json()   # 2
                self._keys = (await c.get(cfg["jwks_uri"])).json()                               # 3
                self._fetched = time.time()
        return self._keys

    async def verify(self, token: str) -> dict:
        header = jwt.get_unverified_header(token)                              # 4
        jwks = await self.keys()
        key = next((k for k in jwks["keys"] if k["kid"] == header["kid"]), None)   # 5
        if key is None:
            self._keys = None                                                  # 6
            jwks = await self.keys()
            key = next((k for k in jwks["keys"] if k["kid"] == header["kid"]), None)
            if key is None: raise HTTPException(401, "Unknown signing key")
        try:
            return jwt.decode(token, key, algorithms=["RS256"], audience=self.audience, issuer=self.issuer)   # 7
        except jwt.JWTError as e:
            raise HTTPException(401, f"Invalid token: {e}")
```
1. Cache keys for an hour.
2. Discovery document → `jwks_uri`.
3. Download public keys.
4. Read `kid` from the header *without* verifying (safe: only used to select a key).
5. Find the matching key.
6. Unknown `kid` → keys may have rotated; refetch once.
7. Full verification: signature, expiry, audience (this API), issuer (this IdP). Explicit `algorithms`.

---

## Q: How do you integrate FastAPI with AWS Cognito?

Cognito is an OIDC provider, so the above verifier applies with:
- `issuer = f"https://cognito-idp.{region}.amazonaws.com/{user_pool_id}"`
- JWKS at `{issuer}/.well-known/jwks.json`
- Two token types: **ID token** (`aud` = app client ID, has user attributes) and **access token** (`client_id` claim instead of `aud`, has `scope` and `cognito:groups`). For APIs, verify the **access token**: check `token_use == "access"` and `client_id` matches.
- Map `cognito:groups` → roles.

Flow: frontend uses Cognito Hosted UI / Amplify (Authorization Code + PKCE) → gets tokens → calls FastAPI with `Authorization: Bearer <access_token>` → dependency verifies via JWKS → user identified by `sub`. Alternatively terminate auth at API Gateway with a Cognito authorizer and forward claims as headers.

---

## Q: How would you protect admin endpoints?

1. Separate router with router-level `require_roles(Role.admin)` **and** a distinct URL prefix (`/admin/...`) so ingress can add IP allow-listing / VPN-only rules.
2. Require **step-up auth** (recent MFA) for destructive actions: check an `auth_time`/`amr` claim.
3. Short-lived admin tokens (5 min) and separate audience.
4. Audit-log every admin call with principal, target, before/after.
5. Rate-limit and alert on 403s to the admin prefix.
6. Don't expose admin routes in public OpenAPI docs (`include_in_schema=False`).
7. Consider a separate deployment/service for admin APIs — different blast radius.

---

## Q: How would you prevent privilege escalation?

- **Never trust client-supplied role/ownership fields**: request schemas must not include `is_admin`, `role`, `owner_id`. Set them server-side.
- **Authorize on the server for every request**, including "read" — never rely on the UI hiding buttons.
- **Check ownership at the resource level**, not just role (IDOR prevention): user A must not fetch `/orders/123` belonging to B.
- **Separate authorization from validation**: a valid request isn't an authorized one.
- **Immutable principal**: derive user identity from the verified token, never from a body/header the client controls (except from a trusted gateway over a private network).
- **Least-privilege tokens** with scopes; internal services don't get admin scope.
- **Role changes invalidate tokens** (token version bump).
- **Deny by default** at router level; make public routes explicit.
- **Review sudo-like features** (impersonation) with strong audit trails and MFA.
- **Tests**: for each endpoint, a test that a lower-privileged user gets 403/404.

---

# 8. Databases

## Q: How do you connect FastAPI to PostgreSQL?

Async stack: SQLAlchemy 2.0 + `asyncpg`.

```python
# app/db/session.py
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker, AsyncSession

engine = create_async_engine(                                   # 1
    settings.database_url,                                      # postgresql+asyncpg://user:pass@host:5432/db
    pool_size=10,                                               # 2
    max_overflow=20,
    pool_timeout=30,
    pool_recycle=1800,                                          # 3
    pool_pre_ping=True,                                         # 4
    echo=settings.debug,
)
SessionLocal = async_sessionmaker(engine, expire_on_commit=False)   # 5

async def get_db() -> AsyncIterator[AsyncSession]:              # 6
    async with SessionLocal() as session:
        yield session
```
1. One engine per process — it *owns the connection pool*. Created at import time (or in lifespan).
2. Pool sizing (see below).
3. Recycle connections every 30 min to dodge server-side idle timeouts / stale NAT mappings.
4. Test the connection with a lightweight query before handing it out; avoids "server closed the connection" errors after DB restarts.
5. Session factory. `expire_on_commit=False` so ORM objects remain usable after commit without a lazy reload (which would fail in async).
6. Per-request session dependency.

---

## Q: What ORM have you used?

Answer with specifics: "SQLAlchemy 2.0 with the async engine and `asyncpg` in production; Alembic for migrations; SQLModel on smaller projects; Django ORM previously; Tortoise/Prisma-py evaluated." Then be ready for the follow-ups below.

---

## Q: What is SQLAlchemy?

The de-facto Python SQL toolkit. Two layers:
- **Core** – SQL expression language, connection pooling, dialects. You build queries as Python objects (`select(users).where(...)`).
- **ORM** – maps classes to tables, tracks object state in a **Session** (unit of work), handles relationships and identity map.

2.0 style uses `Mapped[...]` annotations and `select()` everywhere (no more `session.query`).

```python
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, relationship

class Base(DeclarativeBase): pass

class User(Base):
    __tablename__ = "users"
    id: Mapped[int] = mapped_column(primary_key=True)
    email: Mapped[str] = mapped_column(unique=True, index=True)
    orders: Mapped[list["Order"]] = relationship(back_populates="user")
```

---

## Q: What is SQLModel?

A library by FastAPI's author that fuses **Pydantic + SQLAlchemy**: one class is both a validated schema and an ORM model.

```python
from sqlmodel import SQLModel, Field
class Hero(SQLModel, table=True):
    id: int | None = Field(default=None, primary_key=True)
    name: str
```
Pros: less duplication, great for small services. Cons: couples API schema to DB schema (the thing Section 3 warned about), lags SQLAlchemy features, async story is thinner. My default for large systems is plain SQLAlchemy + separate Pydantic schemas.

---

## Q: How do you create a database session?

Via `async_sessionmaker` (above) and a `yield` dependency. **One session per request**, never shared across requests or tasks. The session is cheap; it borrows a connection from the pool lazily on first query and returns it on close.

---

## Q: Why shouldn't you create a new database connection for every request?

A PostgreSQL connection = TCP handshake + TLS + auth + a **forked backend process** on the server (~5–10 ms, several MB RAM each). At 1,000 rps that's 1,000 forks/second and thousands of server processes — the DB collapses long before your app does. Postgres also has a hard `max_connections` (default 100). Pooling amortises connection cost across requests.

---

## Q: What is connection pooling?

A pool keeps N open connections and lends them out. Request borrows → uses → returns (not closed). SQLAlchemy's `QueuePool` does this in-process. For many app instances, add an **external pooler** (PgBouncer, RDS Proxy) between apps and Postgres so 50 pods × 10 connections don't become 500 server backends.

---

## Q: How do you configure connection pool size?

- `pool_size`: steady-state connections kept open (per process).
- `max_overflow`: extra connections allowed under burst; closed when returned.
- `pool_timeout`: seconds to wait for a free connection before raising.

**Sizing formula (start here, then measure):**
`connections_needed ≈ concurrent_requests_touching_db × avg_db_time / avg_request_time`
Practical: `pool_size = 5–10` per Uvicorn worker. Total across pods must stay under `max_connections` minus headroom for migrations, admins, replicas. Postgres rarely benefits from more than `(cores × 2) + spindles` active connections — beyond that, add PgBouncer rather than connections.

---

## Q: What happens when the database connection pool is exhausted?

New requests block in `pool.acquire()` for `pool_timeout` seconds, then raise `TimeoutError: QueuePool limit of size 10 overflow 20 reached`. Latency shows a characteristic *step* pattern; throughput plateaus; if the event loop's coroutines pile up, memory grows.

**Causes:** slow queries holding connections; sessions not closed (missing `yield` teardown); long transactions; `await`ing external HTTP *while holding a connection*.
**Fixes:** close sessions reliably; commit early; don't hold a connection during external calls; add indexes; raise pool size cautiously; add PgBouncer; add a `pool_timeout` alert; set Postgres `statement_timeout`.

---

## Q: How do you handle database transactions?

SQLAlchemy sessions are transactional: the first query begins a transaction; you `commit()` or `rollback()`. The **Unit of Work** pattern: one transaction per request/use-case, committed at the service boundary.

```python
class OrderService:
    def __init__(self, db: AsyncSession): self.db = db

    async def place_order(self, user_id: int, items: list[LineItem]) -> Order:
        async with self.db.begin():                                   # 1
            order = Order(user_id=user_id)
            self.db.add(order)
            for li in items:
                stock = await self.db.scalar(
                    select(Product).where(Product.id == li.sku).with_for_update()   # 2
                )
                if stock.qty < li.qty:
                    raise InsufficientStock(li.sku)                  # 3
                stock.qty -= li.qty
                order.lines.append(OrderLine(product=stock, qty=li.qty))
        return order                                                  # 4
```
1. `begin()` context manager: commit on normal exit, rollback on exception.
2. `SELECT … FOR UPDATE` locks the row so two concurrent orders can't both decrement past zero.
3. Raising rolls back everything — no half-placed orders.
4. After the block, the transaction is committed.

---

## Q: What is commit vs rollback?

- **Commit**: make all changes in the transaction permanent and visible to others. Atomic — all or nothing.
- **Rollback**: discard all changes since the transaction began; the DB looks as if nothing happened. Also releases locks.

Transactions give you ACID: Atomicity (all/none), Consistency (constraints hold), Isolation (concurrent txns don't see each other's partial work), Durability (committed = survives crash).

---

## Q: How do you handle database exceptions?

Translate low-level exceptions into domain errors at the repository boundary; never leak SQL to the client.

```python
from sqlalchemy.exc import IntegrityError, OperationalError, DBAPIError

async def create_user(self, email: str) -> User:
    user = User(email=email)
    self.db.add(user)
    try:
        await self.db.flush()                                          # 1
    except IntegrityError as e:
        await self.db.rollback()
        if "users_email_key" in str(e.orig):                           # 2
            raise Conflict("email already registered") from e
        raise
    return user
```
1. `flush()` sends the INSERT so constraint errors surface here rather than at commit.
2. Inspect the constraint name (asyncpg exposes `e.orig.constraint_name`) → specific 409.

Global handler maps `OperationalError` (connection lost) → 503 with retry-after, and logs `DBAPIError` details server-side only. Use `tenacity` to retry *idempotent reads* on transient errors, never blind retries of writes.

---

## Q: How do you perform migrations?

With **Alembic**. Workflow:
```bash
alembic init migrations                     # once
alembic revision --autogenerate -m "add orders table"   # diff models vs DB → script
# review the generated script! autogenerate misses renames, data migrations, some constraints
alembic upgrade head                        # apply
alembic downgrade -1                        # revert one
```
Run migrations as a **separate CI/CD step or Kubernetes Job** before rolling out new pods — not at app startup (multiple pods racing, and startup shouldn't need DDL privileges).

---

## Q: What is Alembic?

SQLAlchemy's migration tool. Keeps a linear (or branched) chain of versioned scripts, each with `upgrade()`/`downgrade()`, and records the current version in an `alembic_version` table. `--autogenerate` compares your `Base.metadata` to the live schema and drafts the diff.

```python
def upgrade():
    op.add_column("users", sa.Column("phone", sa.String(20), nullable=True))
    op.create_index("ix_users_phone", "users", ["phone"])

def downgrade():
    op.drop_index("ix_users_phone")
    op.drop_column("users", "phone")
```

---

## Q: How do you handle database schema changes without downtime?

**Expand–migrate–contract.** Every change must be compatible with *both* the old and new app version, because during a rolling deploy both run simultaneously.

1. **Expand**: add the new column/table as nullable (or with a default). Old code ignores it.
2. **Deploy code** that writes to both old and new (dual-write) and reads from new with fallback.
3. **Backfill** existing rows in batches (not one giant `UPDATE` that locks the table).
4. **Contract**: once all readers use the new column, deploy code that stops touching the old one, then drop it in a later migration.

Postgres specifics:
- `ADD COLUMN … DEFAULT x` is instant since PG11 (metadata-only) — but `NOT NULL` on existing rows still scans. Add `NOT NULL` as a `CHECK … NOT VALID` then `VALIDATE`.
- `CREATE INDEX CONCURRENTLY` (can't run inside a transaction; Alembic needs `op.execute` with autocommit).
- Never rename a column in one step — add new, dual-write, drop old.
- Set `lock_timeout = '5s'` in migrations so a DDL waiting on a lock fails fast rather than queueing everything behind it.

---

## Q: What is the N+1 query problem?

You fetch N parent rows (1 query), then for each access a lazy-loaded relationship (N more queries).

```python
users = (await db.scalars(select(User))).all()      # 1 query
for u in users:
    print(u.orders)     # N queries (and in async, MissingGreenlet error!)
```
100 users → 101 queries → 101 round trips. Fix with **eager loading**:

```python
from sqlalchemy.orm import selectinload, joinedload
users = (await db.scalars(select(User).options(selectinload(User.orders)))).all()   # 2 queries total
```
- `selectinload`: second query `WHERE user_id IN (...)` — best for one-to-many.
- `joinedload`: single LEFT JOIN — best for many-to-one / small collections.

Detect with SQLAlchemy `echo=True`, query-count assertions in tests, or APM showing spikes in query count per request.

---

## Q: How do you optimize slow database queries?

1. **Find them**: `pg_stat_statements` (top by total time), slow-query log (`log_min_duration_statement = 200ms`), APM spans.
2. **`EXPLAIN (ANALYZE, BUFFERS)`**: look for Seq Scan on big tables, high row-estimate errors, nested loops on large sets, sorts spilling to disk.
3. **Indexes**: add on filter/join/order columns; composite indexes with the most selective column first (or matching the `WHERE` + `ORDER BY`); partial indexes for `WHERE status = 'active'`; covering indexes (`INCLUDE`).
4. **Query shape**: select only needed columns; avoid `OFFSET` on deep pages; replace correlated subqueries with joins/CTEs; batch `IN` lists.
5. **N+1** elimination.
6. **Stats**: `ANALYZE` after big loads; tune `work_mem` for sorts.
7. **Denormalize / materialized views** for heavy aggregations; refresh on schedule.
8. **Cache** the result in Redis if freshness allows.
9. **Read replicas** for reporting queries.

---

## Q: How do you implement pagination?

```python
class Page(BaseModel, Generic[T]):
    items: list[T]
    next_cursor: str | None

@router.get("/orders", response_model=Page[OrderRead])
async def list_orders(
    db: DB, user: CurrentUser,
    limit: Annotated[int, Query(ge=1, le=100)] = 50,
    cursor: str | None = None,
):
    stmt = select(Order).where(Order.user_id == user.id).order_by(Order.id.desc()).limit(limit + 1)   # 1
    if cursor:
        stmt = stmt.where(Order.id < decode_cursor(cursor))                                          # 2
    rows = (await db.scalars(stmt)).all()
    has_more = len(rows) > limit                                                                     # 3
    rows = rows[:limit]
    return Page(items=rows, next_cursor=encode_cursor(rows[-1].id) if has_more else None)           # 4
```
1. Fetch one extra to know whether another page exists.
2. Cursor = last seen ID; "give me rows before this".
3. If we got `limit+1`, there's more.
4. Return an opaque cursor (base64 of the ID) — clients treat it as a token.

---

## Q: Offset pagination vs cursor pagination?

| | Offset (`LIMIT 20 OFFSET 4000`) | Cursor / keyset (`WHERE id < ? LIMIT 20`) |
|---|---|---|
| Performance | O(offset) — DB scans and discards | O(log n) — index seek |
| Consistency | Rows shift when items inserted/deleted → duplicates/skips | Stable |
| Jump to page N | ✅ | ❌ (only next/prev) |
| Total count | Easy (but expensive) | Usually omitted |
| Best for | Small admin tables, UI needing page numbers | Feeds, infinite scroll, large tables, APIs |

Cursor needs a **unique, sortable** key; for `ORDER BY created_at` use a composite `(created_at, id)` cursor to break ties.

---

## Q: How do you prevent SQL injection?

- **Parameterized queries always.** SQLAlchemy Core/ORM does this automatically: `select(User).where(User.email == email)` sends `$1` and the value separately.
- For raw SQL: `text("SELECT * FROM users WHERE email = :email").bindparams(email=email)` — never f-strings.
- **Identifiers** (table/column names) can't be parameterized — allow-list them.
- Least-privilege DB user (no `DROP`, no superuser).
- Validate input types with Pydantic (an `int` param can't carry `' OR 1=1`).
- Keep ORDER BY columns from a fixed mapping: `SORTABLE = {"created": Order.created_at, "total": Order.total}`.

---

## Q: How do you implement database health checks?

```python
@router.get("/health/ready", include_in_schema=False)
async def ready(db: DB):
    try:
        async with asyncio.timeout(2):                       # 1
            await db.execute(text("SELECT 1"))               # 2
    except Exception:
        raise HTTPException(503, "database unavailable")     # 3
    return {"status": "ready"}
```
1. Bound the check so a hung DB doesn't hang the probe.
2. Cheapest possible round-trip that proves pool + network + auth work.
3. 503 → Kubernetes stops sending traffic (readiness) without restarting the pod.

Keep `/health/live` DB-free — a DB outage shouldn't restart every pod (that only makes things worse).

---

## Q: How would you handle read replicas?

Route reads to replicas, writes (and reads that must see the write) to the primary:

```python
primary = create_async_engine(settings.primary_url, ...)
replica = create_async_engine(settings.replica_url, ...)      # or a load-balanced DNS name
WriteSession = async_sessionmaker(primary, expire_on_commit=False)
ReadSession = async_sessionmaker(replica, expire_on_commit=False)

async def get_read_db() -> AsyncIterator[AsyncSession]:
    async with ReadSession() as s: yield s

@router.get("/orders")                       # read → replica
async def list_orders(db: Annotated[AsyncSession, Depends(get_read_db)]): ...

@router.post("/orders")                      # write → primary
async def create(db: DB): ...
```
Handle **replication lag**: after a write, the client may read stale data from a replica. Options: read-your-writes by routing that user's reads to the primary for N seconds (sticky via cookie/Redis flag); or return the created object in the POST response so the client doesn't need to re-fetch. Monitor `pg_stat_replication` lag; fall back to primary if lag > threshold.

---

## Q: How would you handle database failover?

- Use a managed HA setup (RDS Multi-AZ, Aurora, Patroni) with a **stable endpoint** (DNS/VIP) that repoints on failover.
- App side: `pool_pre_ping=True`; on `OperationalError` invalidate the pool (`engine.dispose()`) and reconnect; keep DNS TTL low; retry idempotent operations with backoff (`tenacity`).
- Keep transactions short so fewer are in flight at failover.
- Readiness probe returns 503 during failover so traffic pauses instead of erroring; alerting on the event.
- Test it: run a game day that kills the primary and verify recovery time.

---

## Q: How would you optimize an API returning 1 million records?

Don't return a million rows in one response. Options depending on the consumer:

1. **Paginate with cursors** (interactive clients).
2. **Stream** NDJSON/CSV so memory stays flat:

```python
from fastapi.responses import StreamingResponse

async def iter_orders(db):
    stmt = select(Order).execution_options(yield_per=1000)          # 1
    stream = await db.stream_scalars(stmt)                           # 2
    async for o in stream:
        yield OrderRead.model_validate(o).model_dump_json() + "\n"   # 3

@router.get("/orders/export")
async def export(db: DB):
    return StreamingResponse(iter_orders(db), media_type="application/x-ndjson")   # 4
```
1. Server-side cursor: fetch 1,000 at a time.
2. Async streaming result.
3. Serialize row-by-row.
4. Response body is produced lazily; the client starts receiving immediately.

3. **Async export job**: generate a file to S3, return a job ID, client polls and downloads via pre-signed URL (best for multi-GB).
4. Use `COPY` from Postgres directly to a file for raw speed.
5. Only select needed columns; use `orjson` for serialization; gzip the stream.

---

## Q: How would you design a repository/service layer?

```
API layer (routers)       → HTTP concerns: parse, auth, status codes
Service layer             → use cases, transactions, business rules, orchestration
Repository layer          → data access; hides SQLAlchemy behind intent-revealing methods
Domain models / ORM       → entities
```

```python
# app/repositories/order_repo.py
class OrderRepository:
    def __init__(self, db: AsyncSession): self.db = db

    async def get(self, order_id: int) -> Order | None:
        return await self.db.get(Order, order_id, options=[selectinload(Order.lines)])

    async def list_for_user(self, user_id: int, limit: int, before_id: int | None) -> list[Order]:
        stmt = select(Order).where(Order.user_id == user_id).order_by(Order.id.desc()).limit(limit)
        if before_id: stmt = stmt.where(Order.id < before_id)
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
                if p.qty < li.qty: raise InsufficientStock(li.sku)
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
async def place_order(body: OrderCreate, user: CurrentUser, svc: Annotated[OrderService, Depends(get_order_service)]):
    return await svc.place(user.id, body.items)
```
**Why this split:** routers stay thin and HTTP-only; services are unit-testable with fake repositories (no DB); repositories concentrate query optimization; the same service can be called from a CLI, a queue consumer or a gRPC handler. Avoid over-abstracting: a generic `BaseRepository[T]` with CRUD is fine, but don't build a query DSL on top of SQLAlchemy — it already is one.

---

# 9. Project Structure & Architecture

## Q: How would you structure a large FastAPI project?

Organize **by domain (feature)**, not by technical layer, once you have more than a handful of entities. Layer-first (`routers/`, `models/`, `schemas/` with 40 files each) forces you to touch four directories per feature and encourages cross-imports.

```
app/
├── main.py                 # create_app(), lifespan, middleware, include routers
├── core/
│   ├── config.py           # pydantic-settings Settings
│   ├── security.py         # hashing, JWT
│   ├── logging.py
│   └── exceptions.py       # AppError hierarchy + handlers
├── db/
│   ├── base.py             # DeclarativeBase, mixins (timestamps)
│   ├── session.py          # engine, sessionmaker, get_db
│   └── migrations/         # alembic
├── api/
│   ├── deps.py             # CurrentUser, DB, service factories
│   └── v1/
│       ├── router.py       # aggregates feature routers
│       └── ...
├── features/               # (or "modules"/"domains")
│   ├── users/
│   │   ├── models.py       # ORM
│   │   ├── schemas.py      # Pydantic in/out
│   │   ├── repository.py
│   │   ├── service.py
│   │   └── router.py
│   ├── orders/
│   │   └── ... same shape
│   └── payments/
├── integrations/           # external clients: stripe.py, s3.py, sqs.py
├── workers/                # queue consumers, celery tasks
└── tests/
    ├── conftest.py
    ├── unit/
    └── integration/
```

```python
# app/main.py
def create_app() -> FastAPI:                                  # 1
    app = FastAPI(title=settings.app_name, lifespan=lifespan)
    register_middleware(app)
    register_exception_handlers(app)
    app.include_router(api_v1_router, prefix="/api/v1")       # 2
    return app

app = create_app()                                            # 3
```
1. App factory — tests can build fresh apps with different settings.
2. One line per API version.
3. Module-level instance for Uvicorn.

---

## Q: Why shouldn't all business logic be inside route handlers?

- **Untestable without HTTP**: you'd need `TestClient` for every rule.
- **Not reusable**: the same logic is needed by a CLI command, a queue consumer, an admin script.
- **Mixed concerns**: HTTP status codes, header parsing and domain rules tangle together; changes in one break the other.
- **Transactions get scattered**: commit/rollback in handlers leads to partial writes.
- **Grows without bound**: 300-line handlers nobody dares to refactor.

Handlers should: parse/validate (via FastAPI), call one service method, map the result to a response. ~10 lines.

---

## Q: What is the service layer?

The layer that implements **use cases** ("place order", "register user", "refund payment"). It orchestrates repositories, enforces business rules and invariants, owns transaction boundaries, and emits domain events. It knows nothing about HTTP. Each method reads like the product requirement.

---

## Q: What is the repository pattern?

An abstraction over data access that exposes **intent-revealing methods** (`find_active_by_email`) instead of query mechanics. Benefits: services don't import SQLAlchemy; query optimizations live in one place; can be faked in unit tests with an in-memory dict. Cost: another layer to maintain — keep it thin and don't hide the ORM's power behind a leaky generic interface.

---

## Q: How would you separate API, business logic, and database logic?

Rule of dependencies: **API → Service → Repository → ORM**, never the reverse. Concretely:
- Routers import schemas and services. They never import ORM models or SQLAlchemy.
- Services import repositories and domain exceptions. They never import `fastapi` or `Request`.
- Repositories import ORM models and SQLAlchemy. They never import schemas.
- Domain exceptions (`NotFound`, `Conflict`) live in `core/` and are mapped to HTTP by one exception handler.

Enforce it with `import-linter` contracts in CI so the architecture doesn't erode.

---

## Q: How do you organize routers?

One `APIRouter` per feature, with prefix/tags/dependencies declared once, aggregated by a version router:

```python
# app/features/orders/router.py
router = APIRouter(prefix="/orders", tags=["orders"])

@router.get("/{order_id}", response_model=OrderRead)
async def get_order(...): ...

# app/api/v1/router.py
from app.features import users, orders, payments
router = APIRouter()
router.include_router(users.router)
router.include_router(orders.router, dependencies=[Depends(get_current_user)])
router.include_router(payments.router, dependencies=[Depends(get_current_user)])
```
Keep route functions small; don't nest routers more than two levels; avoid dynamic router registration by directory scanning (magic that breaks static analysis).

---

## Q: How do you organize Pydantic schemas?

Per feature (`features/orders/schemas.py`), named by intent: `OrderCreate`, `OrderUpdate`, `OrderRead`, `OrderSummary`. Shared primitives (`Page[T]`, `ErrorResponse`, `Money`) in `core/schemas.py`. Never share a schema between request and response; never reuse ORM models as schemas. Keep schema files free of business logic — validators do *format* rules only.

---

## Q: How do you organize configuration?

Single `Settings` class with `pydantic-settings`, loaded once, injected via dependency:

```python
class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env", env_nested_delimiter="__", extra="ignore")
    env: Literal["local", "dev", "staging", "prod"] = "local"
    app_name: str = "orders-api"
    database_url: PostgresDsn
    redis_url: RedisDsn
    jwt_secret: SecretStr
    jwt_ttl_minutes: int = 15
    cors_origins: list[HttpUrl] = []
    log_level: str = "INFO"

    @property
    def is_prod(self) -> bool: return self.env == "prod"
```
Benefits: typed, validated at startup (a missing `DATABASE_URL` fails fast with a clear message), documented in one file, nested groups via `DB__POOL_SIZE`. Avoid scattered `os.getenv` calls.

---

## Q: How do you manage environment variables?

- **Local**: `.env` file (git-ignored), loaded by pydantic-settings; `.env.example` committed with dummy values.
- **CI**: pipeline variables (masked).
- **Kubernetes**: `ConfigMap` for non-secret config, `Secret` for secrets, mounted as env vars or files.
- Follow 12-factor: same image everywhere, config differs by environment.
- Validate at startup; never default a secret; never log the settings object (use `SecretStr`).

---

## Q: How do you manage secrets?

- Never in git, images, or logs.
- **Source of truth**: a secrets manager (AWS Secrets Manager, HashiCorp Vault, GCP Secret Manager).
- **Delivery to pods**: External Secrets Operator / Secrets Store CSI driver syncs manager → Kubernetes `Secret` → env/file. Or fetch at startup via SDK using the pod's IAM role (IRSA) — no static AWS keys.
- **Rotation**: manager rotates; app re-reads on SIGHUP or restart; DB credentials via IAM auth / RDS Proxy avoid passwords altogether.
- **Scope**: each service its own secrets and IAM role; least privilege.
- **Audit**: access logs on the manager.
- Scan repos for leaked secrets (gitleaks, trufflehog) in CI.

---

## Q: How would you implement API versioning?

Version at the router level with separate packages per version (shown in Section 3). Additional practices:
- Version only when you *break* the contract; additive changes don't need a new version.
- Keep the domain/service layer unversioned; versions are thin adapters.
- Advertise deprecation via `Deprecation` and `Sunset` headers and docs; track per-version usage metrics; remove when zero.
- Generate separate OpenAPI docs per version (`FastAPI(openapi_url="/api/v1/openapi.json")` on sub-apps if needed).

---

## Q: URI versioning vs header versioning?

| | URI (`/api/v2/users`) | Header (`Accept: application/vnd.app.v2+json` or `X-API-Version: 2`) |
|---|---|---|
| Discoverability | Obvious, curl-friendly, cacheable by URL | Hidden; caches need `Vary` |
| Routing | Trivial (prefix) | Custom dependency/middleware |
| REST purity | "Same resource, different URL" | Same URL, negotiated representation |
| Gateway support | Universal | Requires header-based routing |
| Practical winner | ✅ Most public APIs (Stripe, GitHub use URI or date-header) | Internal APIs with sophisticated clients |

My default: **URI versioning** with major versions only; minor changes are additive.

---

## Q: How do you maintain backward compatibility?

- Additive-only changes within a version (new optional fields, new endpoints, new enum values *only if clients tolerate unknowns*).
- Never change a field's type or semantics; never make optional fields required.
- Tolerant reader on your side: `extra="ignore"` for inbound third-party payloads.
- Contract tests against recorded OpenAPI snapshots; consumer-driven contracts (Pact) for internal clients.
- Feature flags to roll out behaviour changes gradually.
- Deprecation policy with timelines; communicate via changelog and headers.

---

## Q: How do you design reusable components?

- **Dependencies** for cross-cutting request logic (`Pagination`, `CurrentUser`, `RateLimiter`).
- **Generic schemas** (`Page[T]`, `ErrorResponse`).
- **Mixins** for ORM (`TimestampMixin`, `SoftDeleteMixin`).
- **Base repository** with `get/add/delete` generics — thin.
- **Integration clients** with explicit interfaces (`Protocol`) so they can be faked.
- Package truly shared code (auth verification, logging config, tracing setup) as an **internal library** consumed by all services with semantic versioning.
- Prefer composition over deep inheritance; keep components stateless or explicitly scoped.

---

## Q: How would you prevent circular imports?

- Enforce the dependency direction (API → service → repo → models). Circularity means a lower layer imports a higher one — fix the design.
- Put shared types in a leaf module (`core/`), never in `main.py`.
- Use `TYPE_CHECKING` guards for type-only imports:
  ```python
  from typing import TYPE_CHECKING
  if TYPE_CHECKING:
      from app.features.orders.models import Order
  ```
- SQLAlchemy relationships accept string class names (`relationship("Order")`) precisely to avoid import cycles.
- Avoid importing `app` (the FastAPI instance) from feature modules; use `APIRouter`.
- Use `import-linter` to forbid cycles in CI.

---

## Q: How do you handle shared utilities?

`app/core/` (or `app/common/`) for small, dependency-free helpers: datetime, ids, pagination math, text. Rules: no business logic, no imports from features, well-tested. If a "utils" module exceeds a few hundred lines, split by topic. Cross-service utilities go into an internal package published to a private index.

---

## Q: How do you design a modular FastAPI application?

Each feature module is self-contained (models, schemas, repo, service, router) and exposes a small public API (`router`, service factory, domain events). Modules communicate through **service interfaces or domain events**, not by reaching into each other's repositories. This "modular monolith" keeps one deployable but with clear seams — the prerequisite for extracting a microservice later.

---

## Q: Monolith vs microservices — when would you choose each?

**Start with a modular monolith** unless you already have concrete reasons not to.

Choose **monolith** when: small team (< ~15 engineers), domain boundaries still shifting, need transactional consistency across modules, ops maturity is low. Benefits: one deploy, one DB transaction, simple debugging, refactoring across modules is a rename.

Choose **microservices** when: independent teams need independent release cadence; components have very different scaling profiles (ML inference vs CRUD); different tech/runtime needs; fault isolation is a hard requirement; regulatory isolation (PCI scope). Costs: network failures, distributed transactions (sagas), observability, service discovery, versioned contracts, N× infra.

**Interview answer:** "Microservices solve an *organizational* scaling problem more than a technical one. I default to a well-structured monolith and extract services when a boundary is stable and there's a clear scaling or team-ownership driver."

---

## Q: How would you split a large FastAPI monolith?

1. **Map the domain** (event storming / bounded contexts). Identify a candidate with low coupling and high change rate or distinct scaling needs.
2. **Modularize in place** first: enforce that the candidate module only talks to others via service interfaces/events. Fix the leaks.
3. **Separate the data**: give it its own tables/schema; remove foreign keys across the boundary; replace joins with API calls or event-fed read models.
4. **Strangler fig**: stand up the new service; route traffic through the gateway to it for its endpoints; keep the monolith's implementation as fallback behind a flag.
5. **Dual-run & compare** (shadow traffic) to validate parity.
6. **Cut over**, delete old code.
7. Repeat for the next module. Measure success by deploy frequency and incident isolation, not by service count.

---

## Q: How would you design FastAPI services for independent deployment?

- **Own data store** per service; no shared DB tables.
- **Versioned, backward-compatible APIs** (additive changes; consumer-driven contract tests).
- **Async communication** (events via Kafka/SQS) for non-critical paths so a deploy of one doesn't fail another.
- **Per-service pipelines**, images, Helm charts, and secrets.
- **Feature flags** to decouple deploy from release.
- **Health/readiness endpoints** and graceful shutdown so rolling deploys are invisible.
- **Observability contract**: every service emits the same metrics/trace headers.
- **Shared internal library** for auth/logging/tracing with semantic versioning so one team's change doesn't force a global redeploy.

---

# 10. Performance & Scaling

## Q: Why is FastAPI considered fast?

Covered in Section 1. Summary for the interviewer: ASGI event loop (no thread-per-request), uvloop + httptools in C, Pydantic v2 validation in Rust, and zero-overhead routing from Starlette. For I/O-bound APIs it benchmarks near Node.js/Go. It's still Python — CPU-bound code is as slow as anywhere else in Python.

---

## Q: How would you benchmark a FastAPI application?

1. **Define the goal**: target p99 latency at target rps with a realistic payload mix — not "max rps of /health".
2. **Environment**: production-like (same CPU limits, DB, network), isolated from noise.
3. **Tool**: `k6` (scriptable, thresholds, ramp profiles), `wrk2` (constant-rate, avoids coordinated omission), `locust` (Python, user journeys), `oha` for quick checks.
4. **Profile**: warm-up → ramp → steady state ≥ 5 min → spike. Collect p50/p95/p99, error rate, throughput, plus server CPU/memory/DB pool.
5. **Compare** against a baseline; automate in CI with thresholds (`k6 --threshold http_req_duration:p(99)<300`).
6. **Profile bottlenecks**: `py-spy record --pid` for flame graphs; `EXPLAIN ANALYZE` for queries.

```js
// k6 script
import http from 'k6/http';
export const options = { stages: [{duration:'1m',target:100},{duration:'5m',target:100},{duration:'1m',target:0}],
  thresholds: { http_req_duration: ['p(99)<300'], http_req_failed: ['rate<0.01'] } };
export default function () { http.get('https://staging.api/orders?limit=20', {headers:{Authorization:`Bearer ${__ENV.TOKEN}`}}); }
```

---

## Q: What metrics would you monitor?

**RED** for the service: Rate (rps), Errors (5xx/4xx rate), Duration (latency histograms per route).
**USE** for resources: Utilization, Saturation, Errors of CPU, memory, event-loop lag, threadpool queue, DB pool (in-use/waiting), Redis latency.
**Business**: orders/min, signups, payment failures.
**Dependencies**: downstream call latency & error rate per target.
**Runtime**: GC pauses, open file descriptors, process restarts, `uvicorn` worker count.

---

## Q: How do you reduce API latency?

Ordered by typical payoff:
1. Fix blocking calls in async handlers.
2. Fix N+1 and add indexes (DB is usually 60–80% of latency).
3. Connection pooling everywhere (DB, Redis, HTTP clients reused).
4. Cache hot reads (Redis) and expensive computed responses.
5. Parallelize independent I/O with `gather`.
6. Return less: pagination, field selection, avoid over-fetching relationships.
7. Faster serialization (`orjson`), gzip/brotli for large responses.
8. Move slow work off the request path (background jobs, queues).
9. Keep the service close to its DB (same AZ), use keep-alive.
10. Right-size workers; avoid CPU throttling in Kubernetes.

---

## Q: How do you identify slow endpoints?

Prometheus histogram by route → Grafana table sorted by p99; tracing (Tempo/Jaeger) to see which span is slow inside the endpoint; APM (Datadog/New Relic) endpoint breakdown; access-log analysis (`duration_ms` percentiles per path); `py-spy` on a live pod for "right now" view. Add SLO burn-rate alerts per critical endpoint.

---

## Q: How would you handle 10,000 requests per second?

Back-of-envelope: a well-written async FastAPI worker handles ~1–3k rps of light I/O endpoints on one core. So ~10–20 workers across 3–5 pods — *if* the backends keep up. The plan:

1. **Stateless app** → horizontal scaling with HPA.
2. **Load balancer / ingress** with keep-alive and HTTP/2 to upstream.
3. **Cache aggressively**: CDN/edge for public GETs, Redis for hot data; target high hit ratio so the DB sees a fraction of 10k.
4. **DB**: read replicas, PgBouncer, indexed queries, short transactions; consider partitioning.
5. **Async everywhere**; no blocking; bounded concurrency to downstreams.
6. **Rate limiting** and load shedding (return 429/503 fast when saturated rather than queueing).
7. **Move writes off the path**: enqueue and process asynchronously where semantics allow.
8. **Observability** to find the next bottleneck.
9. **Load test to 2× target** to know headroom.

---

## Q: How does Uvicorn handle concurrency?

Within a worker: a single event loop handles many connections concurrently via non-blocking sockets — thousands of in-flight requests share one thread as long as they `await`. Sync endpoints go to a threadpool (default 40 threads). Across workers: `--workers N` forks N processes, each with its own loop, sharing the listening socket (kernel load-balances accepts). No shared memory between workers — state must live in Redis/DB.

---

## Q: What are Uvicorn workers?

Separate OS processes running the same app. They give **CPU parallelism** (bypassing the GIL) and **fault isolation** (one crash doesn't take down the others). Managed by Uvicorn's supervisor or Gunicorn with `UvicornWorker`.

---

## Q: When should you increase workers?

- CPU utilization per worker is high (>70%) while latency rises → add workers up to core count.
- Many sync endpoints saturate the threadpool.
- In Kubernetes, prefer **more pods with 1–2 workers each** over one pod with many workers: finer-grained scaling, better bin-packing, faster restarts, and memory limits per pod are clearer. Rule of thumb: `workers = CPU limit` (rounded), never more than cores.

---

## Q: Worker processes vs threads?

| | Processes | Threads |
|---|---|---|
| GIL | Each has its own → true parallelism | Shared → one Python bytecode at a time |
| Memory | Separate (N× app memory) | Shared |
| Crash isolation | ✅ | ❌ |
| Use for | CPU-bound, scaling Python | Blocking I/O in sync libs (threadpool) |

In FastAPI: **processes** = Uvicorn workers for scaling; **threads** = the threadpool that runs `def` endpoints. Async coroutines beat both for pure async I/O.

---

## Q: How does Kubernetes scale FastAPI?

A `Deployment` runs N identical pods behind a `Service`. The **Horizontal Pod Autoscaler (HPA)** adjusts N based on metrics (CPU, memory, or custom like rps/queue depth). The **Cluster Autoscaler / Karpenter** adds nodes when pods can't be scheduled. **VPA** can right-size requests. Readiness probes gate traffic so new pods only receive requests once warm.

---

## Q: How would you configure HPA?

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata: { name: orders-api }
spec:
  scaleTargetRef: { apiVersion: apps/v1, kind: Deployment, name: orders-api }
  minReplicas: 3                        # 1
  maxReplicas: 30
  metrics:
  - type: Resource
    resource: { name: cpu, target: { type: Utilization, averageUtilization: 60 } }   # 2
  - type: Pods
    pods:
      metric: { name: http_requests_per_second }                                     # 3
      target: { type: AverageValue, averageValue: "500" }
  behavior:
    scaleUp:   { stabilizationWindowSeconds: 0,   policies: [{type: Percent, value: 100, periodSeconds: 30}] }   # 4
    scaleDown: { stabilizationWindowSeconds: 300, policies: [{type: Pods, value: 2, periodSeconds: 60}] }       # 5
```
1. ≥3 for AZ spread and rolling updates.
2. CPU target below 100% leaves headroom for the scale-up lag (~1–2 min for pods to become ready).
3. Custom metric via Prometheus Adapter/KEDA — scales on actual load, not just CPU.
4. Scale up fast (double every 30 s).
5. Scale down slowly to avoid flapping.

HPA needs `resources.requests.cpu` set; utilization is relative to requests.

---

## Q: CPU-based vs memory-based autoscaling?

- **CPU**: good proxy for request load in async apps (loop is CPU when busy). Responsive, standard.
- **Memory**: poor scaling signal — Python memory doesn't shrink when load drops (fragmentation, caches), so memory-based HPA scales up and never down. Use memory *limits* for safety, not for scaling.
- **Best**: request rate / latency / queue depth (KEDA) — the metric users actually feel.

---

## Q: What is connection pooling?

(Also asked in Databases.) Reusing a fixed set of open connections instead of opening/closing per request. Applies to DB (SQLAlchemy pool, PgBouncer), Redis (`redis.asyncio` pool), HTTP (`httpx.AsyncClient` keep-alive pool). Pooling removes handshake latency and protects backends from connection storms.

---

## Q: How would Redis improve API performance?

- **Cache** DB/computed results (sub-millisecond reads vs 5–50 ms DB queries).
- **Session/user cache** so auth doesn't hit the DB.
- **Rate limiting counters** (atomic `INCR`).
- **Distributed locks** (`SET NX PX`) for idempotency and cache stampede control.
- **Pub/Sub / Streams** for lightweight events.
- **Precomputed views** (leaderboards via sorted sets).

```python
async def get_product(product_id: int, redis: Redis, repo: ProductRepo) -> ProductRead:
    key = f"product:{product_id}"
    if cached := await redis.get(key):                            # 1
        return ProductRead.model_validate_json(cached)
    product = await repo.get(product_id)                          # 2
    if product is None: raise NotFound("product")
    dto = ProductRead.model_validate(product)
    await redis.set(key, dto.model_dump_json(), ex=300)           # 3
    return dto
```
1. Cache hit → skip DB.
2. Miss → load.
3. Store with TTL. Invalidate on write: `await redis.delete(key)` in the update service.

---

## Q: When should you introduce caching?

When you have **evidence** (profiling shows a hot, expensive, read-heavy path) and the data tolerates staleness. Caching adds invalidation complexity ("two hard problems") and can mask bugs. Order of preference: fix the query → add index → cache. Good candidates: reference data, product catalogs, computed aggregates, permission lookups, external API responses.

## Q: What data should not be cached?

- Data that must be strongly consistent (account balances, inventory at checkout, auth revocation state).
- Per-user private data in a *shared* cache without per-user keys (leak risk).
- Secrets, tokens, PII beyond what's necessary.
- Rapidly changing data where hit ratio would be near zero.
- Huge payloads that evict everything else.
- Error responses (unless deliberately, with short TTL, to protect a failing backend).

---

## Q: How do you prevent cache stampede?

When a hot key expires, thousands of concurrent requests miss simultaneously and hammer the DB. Techniques:
1. **Lock/single-flight**: only one request recomputes; others wait or serve stale.
2. **Stale-while-revalidate**: serve the old value, refresh in the background.
3. **Jittered TTLs**: `ex=300 + random(0, 60)` so keys don't expire together.
4. **Early recompute (XFetch)**: probabilistically refresh before expiry.
5. **Request coalescing** in-process (`asyncio` future shared by waiters).

```python
async def get_or_compute(redis, key, compute, ttl=300):
    if (v := await redis.get(key)) is not None:
        return v
    lock = redis.lock(f"lock:{key}", timeout=10, blocking_timeout=5)        # 1
    if await lock.acquire():
        try:
            if (v := await redis.get(key)) is not None:                      # 2
                return v
            v = await compute()
            await redis.set(key, v, ex=ttl + random.randint(0, 60))          # 3
            return v
        finally:
            await lock.release()
    if (v := await redis.get(f"stale:{key}")) is not None:                  # 4
        return v
    return await compute()                                                   # 5
```
1. Distributed lock; others block up to 5 s.
2. Double-check after acquiring (someone may have filled it).
3. Jittered TTL.
4. Couldn't get the lock in time → serve a stale copy if kept.
5. Last resort: compute anyway.

---

## Q: What is rate limiting?

Restricting how many requests a client may make in a window. Protects against abuse, brute force, runaway clients, and cost blow-ups; enforces fair share among tenants. Algorithms: fixed window (simple, bursty at edges), sliding window log/counter (smooth), **token bucket** (allows bursts up to capacity, steady refill — most common), leaky bucket (smooths output).

## Q: How would you implement rate limiting?

Layers: **edge** (CloudFront/WAF, API Gateway, NGINX `limit_req`) for volumetric protection; **application** (Redis-backed) for per-user/per-tenant business limits.

```python
# Sliding-window counter in Redis via Lua (atomic)
LUA = """
local key, now, window, limit = KEYS[1], tonumber(ARGV[1]), tonumber(ARGV[2]), tonumber(ARGV[3])
redis.call('ZREMRANGEBYSCORE', key, 0, now - window)          -- 1 drop old entries
local count = redis.call('ZCARD', key)                        -- 2 count in window
if count < limit then
  redis.call('ZADD', key, now, now .. '-' .. math.random())   -- 3 record this hit
  redis.call('PEXPIRE', key, window)
  return {1, limit - count - 1}
end
return {0, 0}
"""

class RateLimit:
    def __init__(self, limit: int, window_ms: int): self.limit, self.window = limit, window_ms
    async def __call__(self, request: Request, user: CurrentUser, redis: RedisDep):
        allowed, remaining = await redis.eval(LUA, 1, f"rl:{user.id}:{request.url.path}",
                                              int(time.time() * 1000), self.window, self.limit)
        request.state.rl_remaining = remaining
        if not allowed:
            raise HTTPException(429, "Rate limit exceeded", headers={"Retry-After": str(self.window // 1000)})   # 4

@router.post("/search", dependencies=[Depends(RateLimit(100, 60_000))])
async def search(...): ...
```
1–3. Atomic sorted-set sliding window: precise, no edge bursts.
4. Return `429` with `Retry-After` and (ideally) `X-RateLimit-Remaining` headers.

Libraries: `slowapi`, `fastapi-limiter`. Key by user ID when authenticated, by IP (behind `X-Forwarded-For` from a trusted proxy) otherwise.

---

## Q: How would you handle sudden traffic spikes?

- **Autoscale** with aggressive scale-up and headroom (min replicas > average).
- **Queue writes** so the API acknowledges quickly and workers drain at their pace.
- **Load shedding**: when saturated, reject early with 503 + `Retry-After` rather than queueing into timeouts. Use concurrency limits per pod (e.g., `--limit-concurrency 1000` in Uvicorn).
- **Caching/CDN** absorbs read spikes.
- **Rate limits** per client stop one tenant from consuming everything.
- **Circuit breakers & timeouts** so downstream slowness doesn't pile up.
- **Pre-warm** for known events (scheduled scaling before a launch).
- **Graceful degradation**: turn off expensive optional features via flags.

---

## Q: How would you optimize JSON serialization?

- Use `ORJSONResponse` as the default response class: `FastAPI(default_response_class=ORJSONResponse)` — `orjson` is 3–10× faster than stdlib and handles datetime/UUID natively.
- Return Pydantic models and let `response_model` serialize in Rust (v2) rather than converting to dict yourself; avoid `jsonable_encoder` on huge structures.
- Trim payloads: `response_model_exclude_none`, field selection.
- For very large lists, stream NDJSON.
- Compress: `GZipMiddleware(minimum_size=1000)` or let the ingress do brotli.
- Avoid double serialization (model → dict → JSON); `model_dump_json()` goes straight to bytes.

---

## Q: How would you handle large file downloads?

Don't proxy bytes through Python if you can avoid it: store in S3 and return a **pre-signed URL** (or `302`), or serve via CDN. When you must serve from the app: `FileResponse` (uses `sendfile`, supports `Range` requests) for local files; `StreamingResponse` with chunked reads for remote sources; set `Content-Length` and `Content-Disposition`; enable range support for resumable downloads; put a reverse proxy (NGINX `X-Accel-Redirect`) in front for zero-copy serving.

## Q: How would you stream large responses?

```python
@router.get("/reports/{id}/csv")
async def report_csv(id: int, db: DB):
    async def gen():
        yield "id,total,created\n"
        stream = await db.stream(select(Order.id, Order.total, Order.created_at).where(Order.report_id == id).execution_options(yield_per=500))
        async for row in stream:
            yield f"{row.id},{row.total},{row.created_at.isoformat()}\n"     # produced lazily
    return StreamingResponse(gen(), media_type="text/csv",
                             headers={"Content-Disposition": f'attachment; filename="report-{id}.csv"'})
```
Memory stays constant regardless of row count; the client sees the first bytes immediately. Avoid `BaseHTTPMiddleware` that buffers bodies; disable response buffering at the proxy (`proxy_buffering off`) if you need true real-time streaming.

---

## Q: How would you reduce memory consumption?

- Fewer workers per pod; scale with pods instead.
- Stream instead of loading whole files/result sets (`yield_per`, chunked uploads).
- Avoid caching unbounded data in-process (use Redis with TTL; `lru_cache(maxsize=...)`).
- Use `__slots__`/dataclasses for hot objects; Pydantic v2 models are already compact.
- Watch for leaks: global lists that grow, unclosed sessions/clients, `asyncio` tasks never awaited.
- Tune `MALLOC_ARENA_MAX=2` for glibc fragmentation; consider `jemalloc`.
- Set Gunicorn `--max-requests` with jitter to recycle workers as a safety net.
- Profile with `tracemalloc`, `memray`, or `objgraph`.

---

# 11. Background Tasks & Job Processing

## Q: What are FastAPI BackgroundTasks?

A way to run a function **after the response has been sent**, in the same process. Starlette attaches the task to the response; Uvicorn sends the response, then runs the task.

```python
from fastapi import BackgroundTasks

async def send_welcome_email(email: str):                         # 1
    await mailer.send(to=email, template="welcome")

@router.post("/signup", status_code=201)
async def signup(body: SignupIn, tasks: BackgroundTasks, svc: UserSvc):   # 2
    user = await svc.register(body)
    tasks.add_task(send_welcome_email, user.email)                 # 3
    return UserRead.model_validate(user)                           # 4
```
1. Any callable (sync or async). Sync ones run in the threadpool.
2. Inject `BackgroundTasks`.
3. Queue it with args.
4. Response goes out first; then the email is sent. Client doesn't wait.

---

## Q: When should you use BackgroundTasks?

Small, fast, **fire-and-forget** work where loss on crash is acceptable: sending a notification, writing an audit log entry, invalidating a cache, bumping a counter, calling a webhook with its own retry. Rule: seconds, not minutes; no strict delivery guarantee needed.

## Q: When should you NOT use BackgroundTasks?

- Work that **must** happen (payments, order fulfilment) — a pod restart loses it; there's no persistence or retry.
- Long-running or CPU-heavy work — it consumes the web worker's loop/threadpool and delays other requests; Kubernetes may kill the pod mid-task on scale-down.
- Work that needs status tracking, scheduling, rate control, or fan-out.
- High volume — no backpressure; memory grows.

---

## Q: BackgroundTasks vs Celery?

| | BackgroundTasks | Celery (with Redis/RabbitMQ) |
|---|---|---|
| Where it runs | Same web process, after response | Separate worker processes/machines |
| Durability | None (lost on crash) | Persisted in broker; ack on success |
| Retries / backoff | Manual | Built in (`autoretry_for`, `retry_backoff`) |
| Scheduling | ❌ | Beat, ETA/countdown |
| Scaling | Tied to web pods | Independent worker autoscaling |
| Monitoring | ❌ | Flower, task events |
| Complexity | Zero | Broker + workers + result backend |
| Use for | Trivial side effects | Anything important, long, or bursty |

Modern alternatives to Celery in async stacks: **arq**, **Dramatiq**, **Taskiq**, **SAQ**.

## Q: BackgroundTasks vs AWS SQS?

SQS is a **managed durable queue** — it's the transport, not the worker. Compared to BackgroundTasks it gives persistence, at-least-once delivery, visibility timeouts, DLQs, and decoupled consumers (Lambda, ECS, K8s workers with KEDA scaling on queue depth). Choose SQS (or Kafka/RabbitMQ) when the work is important, cross-service, or needs to survive deploys; choose BackgroundTasks for trivial in-process side effects.

---

## Q: How would you process a long-running job?

**Async request/response pattern:**
1. `POST /jobs` validates input, writes a job row (`status=queued`), publishes a message, returns **202 Accepted** with `job_id` and a `Location: /jobs/{id}` header.
2. A worker consumes the message, sets `status=running`, does the work (with heartbeats/progress), stores the result (DB/S3), sets `status=succeeded|failed`.
3. Client polls `GET /jobs/{id}` (or receives a webhook / WebSocket / SSE update).

```python
@router.post("/exports", status_code=202)
async def create_export(body: ExportIn, user: CurrentUser, db: DB, sqs: SQSDep):
    job = Job(user_id=user.id, type="export", status="queued", payload=body.model_dump())
    db.add(job); await db.commit()                                           # 1
    await sqs.send_message(QueueUrl=settings.export_queue, MessageBody=json.dumps({"job_id": str(job.id)}))   # 2
    return JSONResponse({"job_id": str(job.id), "status": "queued"}, status_code=202,
                        headers={"Location": f"/api/v1/jobs/{job.id}"})     # 3
```
1. Persist first so the job exists even if publish fails (then a sweeper republishes `queued` jobs older than N minutes — the **transactional outbox** pattern is the robust version).
2. Publish a small message (IDs, not payloads).
3. 202 + where to poll.

---

## Q: How would you implement asynchronous job processing?

Worker side (SQS example, separate deployment from the API):

```python
# app/workers/export_worker.py
async def run():
    async with session.client("sqs") as sqs:
        while True:
            resp = await sqs.receive_message(QueueUrl=Q, MaxNumberOfMessages=10, WaitTimeSeconds=20, VisibilityTimeout=300)   # 1
            for msg in resp.get("Messages", []):
                job_id = json.loads(msg["Body"])["job_id"]
                try:
                    await process_job(job_id)                                                         # 2
                    await sqs.delete_message(QueueUrl=Q, ReceiptHandle=msg["ReceiptHandle"])          # 3
                except RetryableError:
                    pass                                                                              # 4
                except Exception:
                    log.exception("job failed", extra={"job_id": job_id})
                    await mark_failed(job_id)
                    await sqs.delete_message(QueueUrl=Q, ReceiptHandle=msg["ReceiptHandle"])          # 5
```
1. Long polling (20 s) reduces empty receives; visibility timeout hides the message from other consumers while we work.
2. Idempotent processing (see below).
3. Delete only after success → at-least-once semantics.
4. Don't delete → becomes visible again after the timeout → retried; after `maxReceiveCount` SQS moves it to the DLQ.
5. Permanent failure → record and remove.

Scale workers with KEDA on `ApproximateNumberOfMessagesVisible`. Same shape applies to Celery (`@app.task(bind=True, autoretry_for=(RetryableError,), retry_backoff=True, max_retries=5)`).

---

## Q: How would you design a job status API?

```python
class JobStatus(str, Enum): queued="queued"; running="running"; succeeded="succeeded"; failed="failed"; cancelled="cancelled"

class JobRead(BaseModel):
    id: UUID
    status: JobStatus
    progress: int = Field(ge=0, le=100)
    result_url: str | None = None           # pre-signed S3 URL when succeeded
    error: str | None = None                # user-safe message when failed
    created_at: datetime
    updated_at: datetime

@router.get("/jobs/{job_id}", response_model=JobRead)
async def get_job(job_id: UUID, user: CurrentUser, repo: JobRepo):
    job = await repo.get_for_user(job_id, user.id)            # ownership check → 404 if not theirs
    if job is None: raise NotFound("job")
    return job
```
Design notes: 202 on create with `Location`; `GET` returns 200 while in progress (with `Retry-After` hint) and 303 → result when done for hypermedia purists; support `DELETE /jobs/{id}` for cancel (cooperative — worker checks a flag); list endpoint with filters; TTL/cleanup for old jobs; optional push via webhook/SSE to avoid polling storms; expose `attempt` count and timestamps for debugging.

---

## Q: How do you retry failed jobs?

- Classify errors: **retryable** (timeouts, 5xx, deadlocks) vs **permanent** (validation, 4xx, business rule). Only retry the former.
- Bounded attempts (e.g., 5) with **exponential backoff + jitter**.
- Idempotent handlers so retries are safe.
- Persist attempt count and last error on the job row.
- After max attempts → DLQ + alert + manual/automated replay tooling.
- Celery: `autoretry_for`, `retry_backoff=True`, `retry_jitter=True`, `max_retries`. SQS: visibility timeout + redrive policy. Custom: `tenacity`.

## Q: What is exponential backoff?

Waiting progressively longer between retries: 1 s, 2 s, 4 s, 8 s… (`base × 2^attempt`), capped at a max, **with random jitter** so thousands of failed clients don't retry in lockstep (thundering herd). It gives a struggling dependency time to recover instead of piling on.

```python
from tenacity import retry, stop_after_attempt, wait_exponential_jitter, retry_if_exception_type

@retry(stop=stop_after_attempt(5),
       wait=wait_exponential_jitter(initial=1, max=30),
       retry=retry_if_exception_type((httpx.TimeoutException, httpx.HTTPStatusError)))
async def call_partner(payload): ...
```

---

## Q: How do you prevent duplicate job execution?

At-least-once delivery *guarantees* duplicates eventually. Defenses:
1. **Idempotent processing**: the job's effect is keyed by a unique ID; re-running is a no-op.
2. **Atomic claim**: `UPDATE jobs SET status='running', worker=$1 WHERE id=$2 AND status='queued'` — if 0 rows updated, someone else has it.
3. **Distributed lock** per job ID in Redis (`SET lock:job:{id} worker NX PX 300000`) with a TTL slightly above expected runtime and heartbeat renewal.
4. **Deduplication ID** on FIFO queues (SQS FIFO `MessageDeduplicationId`, 5-min window).
5. **Unique constraints** on the side effects (e.g., `UNIQUE(order_id)` on `invoices`), so the second insert fails cleanly.

---

## Q: What is idempotency?

An operation is idempotent if performing it multiple times has the same effect as performing it once. `PUT /users/1 {name: "A"}` is idempotent; `POST /orders` naively is not (retry = duplicate order). Idempotency is what makes **retries safe** in a distributed system where you can't tell whether a timed-out request was processed.

## Q: How would you implement an idempotent POST API?

Client sends an `Idempotency-Key` header (UUID) per logical operation; the server stores the key with the response and replays it on repeats.

```python
class IdempotencyRecord(Base):
    __tablename__ = "idempotency_keys"
    key: Mapped[str] = mapped_column(primary_key=True)
    user_id: Mapped[int] = mapped_column(primary_key=True)          # scope per user
    request_hash: Mapped[str]
    status_code: Mapped[int | None]
    response_body: Mapped[dict | None] = mapped_column(JSONB)
    created_at: Mapped[datetime]

@router.post("/orders", status_code=201)
async def create_order(body: OrderCreate, user: CurrentUser, db: DB, svc: OrderSvc,
                       idem_key: Annotated[str, Header(alias="Idempotency-Key", min_length=16, max_length=128)]):
    req_hash = hashlib.sha256(body.model_dump_json().encode()).hexdigest()            # 1
    rec = IdempotencyRecord(key=idem_key, user_id=user.id, request_hash=req_hash, created_at=now())
    db.add(rec)
    try:
        await db.flush()                                                               # 2
    except IntegrityError:
        await db.rollback()
        existing = await db.get(IdempotencyRecord, (idem_key, user.id))
        if existing.request_hash != req_hash:
            raise HTTPException(422, "Idempotency-Key reused with different payload")  # 3
        if existing.status_code is None:
            raise HTTPException(409, "Request in progress", headers={"Retry-After": "2"})   # 4
        return JSONResponse(existing.response_body, status_code=existing.status_code)  # 5
    order = await svc.place(user.id, body.items)                                       # 6
    payload = OrderRead.model_validate(order).model_dump(mode="json")
    rec.status_code, rec.response_body = 201, payload                                  # 7
    await db.commit()
    return JSONResponse(payload, status_code=201)
```
1. Fingerprint the payload so the same key can't be reused for a different request.
2. Insert the key **first**; the PK constraint makes this the atomic "claim".
3. Same key, different body → client bug.
4. First request still running → tell the client to retry shortly.
5. Completed → replay the stored response (same status, same body).
6. Do the real work — inside the same transaction so the record and order commit together.
7. Store the result. Expire records after 24 h via a cleanup job.

Redis can hold the records instead (with TTL) for high throughput, but then key + order aren't in one transaction — acceptable if the order creation is itself idempotent by a unique constraint.

---

## Q: How would you handle failed messages?

- Retry with backoff for transient errors (visibility timeout / `nack`).
- After N failures, route to a **dead-letter queue**.
- Alert on DLQ depth > 0.
- Preserve the original message and the error reason (metadata attributes).
- Build a **replay** tool: fix the bug → move messages from DLQ back to the main queue (`aws sqs start-message-move-task` or a small script).
- For poison messages (malformed), log, store for inspection, and drop.
- Make consumers idempotent so replay is safe.

## Q: What is a dead-letter queue?

A secondary queue where messages go after exceeding the maximum processing attempts. It **isolates poison messages** so they don't block the main queue forever, **preserves them** for diagnosis, and gives you a place to alert on and replay from. Configured in SQS via a redrive policy (`maxReceiveCount`); in RabbitMQ via `x-dead-letter-exchange`; in Kafka by convention (a `topic.DLQ`).

---

# 12. Testing

## Q: How do you test FastAPI applications?

Three tiers:
1. **Unit tests** – services/validators with fake repositories; no HTTP, no DB. Fast, most numerous.
2. **API/integration tests** – `httpx.AsyncClient` against the ASGI app with dependency overrides; real DB in a container (testcontainers) or a per-test transaction.
3. **End-to-end / smoke** – deployed environment, few, cover critical paths.

Tooling: `pytest`, `pytest-asyncio` (or `anyio` plugin), `httpx`, `factory_boy`/`polyfactory` for fixtures, `respx` for mocking HTTP, `testcontainers` for Postgres/Redis, `pytest-cov`.

---

## Q: What is pytest?

Python's standard test runner: discovers `test_*.py` files, runs `test_*` functions, uses plain `assert` with rich introspection, and provides **fixtures** (dependency-injected setup/teardown with scopes), **parametrize**, markers, and a plugin ecosystem.

```python
@pytest.fixture
def user() -> User: return User(id=1, email="a@b.com")

@pytest.mark.parametrize("qty,ok", [(1, True), (0, False), (-1, False)])
def test_qty(qty, ok):
    assert (LineItem(sku="x", qty=qty) is not None) == ok if ok else pytest.raises(ValidationError)
```

## Q: What is TestClient?

`fastapi.testclient.TestClient` (Starlette's, built on `httpx`) calls your ASGI app **in-process** without a network socket — no Uvicorn needed. Sync API; runs the app's event loop in a thread. It also triggers `lifespan` when used as a context manager. For async test functions prefer `httpx.AsyncClient(transport=ASGITransport(app=app))` so you can `await` alongside DB fixtures.

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

---

## Q: How do you test GET endpoints?

```python
async def test_get_order(client, seeded_order):                       # 1
    r = await client.get(f"/api/v1/orders/{seeded_order.id}")         # 2
    assert r.status_code == 200                                       # 3
    body = r.json()
    assert body["id"] == seeded_order.id                              # 4
    assert "hashed_password" not in body                              # 5

async def test_get_order_not_found(client):
    r = await client.get("/api/v1/orders/999999")
    assert r.status_code == 404
    assert r.json()["error"]["code"] == "not_found"                   # 6
```
1. Fixtures seed data and give an authenticated client (auth overridden).
2. Make the request.
3. Status assertion first — clearest failure message.
4. Assert on the contract, not on internal structure.
5. Negative assertion for leaks.
6. Error envelope shape is part of the contract too.

## Q: How do you test POST endpoints?

```python
async def test_create_order(client, db_session, product):
    payload = {"items": [{"sku": product.sku, "qty": 2}]}
    r = await client.post("/api/v1/orders", json=payload, headers={"Idempotency-Key": str(uuid4())})
    assert r.status_code == 201
    assert r.headers["location"].endswith(f"/orders/{r.json()['id']}")
    order = await db_session.get(Order, r.json()["id"])               # verify side effect in DB
    assert order is not None and order.lines[0].qty == 2
    await db_session.refresh(product)
    assert product.qty == 8                                           # stock decremented from 10
```
Test the **side effects** (DB rows, published events via a fake publisher), not just the response.

## Q: How do you test validation failures?

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
Parametrize the edge cases; assert the `loc` so you know the *right* rule fired.

---

## Q: How do you test authentication?

Two layers:
- **The auth mechanism itself** (real tokens): create a user, get a token via `/auth/token`, call a protected endpoint; test expired, malformed, wrong-signature tokens → 401; wrong role → 403.
- **Everything else**: override `get_current_user` to skip token plumbing.

```python
async def test_login_and_access(client_no_override, user_with_password):
    r = await client_no_override.post("/auth/token", data={"username": user_with_password.email, "password": "s3cret!"})
    token = r.json()["access_token"]
    r = await client_no_override.get("/api/v1/me", headers={"Authorization": f"Bearer {token}"})
    assert r.status_code == 200

async def test_expired_token(client_no_override):
    token = create_access_token("1", [], minutes=-1)
    r = await client_no_override.get("/api/v1/me", headers={"Authorization": f"Bearer {token}"})
    assert r.status_code == 401

async def test_viewer_cannot_delete(client_as(role="viewer")):
    r = await client.delete("/api/v1/items/1")
    assert r.status_code == 403
```

---

## Q: How do you mock external APIs?

With `respx` (for `httpx`) — intercepts at the transport level, no real network:

```python
import respx, httpx

@respx.mock
async def test_weather(client):
    route = respx.get("https://api.weather.example/today").mock(return_value=httpx.Response(200, json={"temp": 21}))   # 1
    r = await client.get("/weather")
    assert r.status_code == 200 and r.json()["temp"] == 21
    assert route.called                                                                                              # 2

@respx.mock
async def test_weather_timeout(client):
    respx.get("https://api.weather.example/today").mock(side_effect=httpx.ReadTimeout("slow"))                      # 3
    r = await client.get("/weather")
    assert r.status_code == 503                                                                                       # 4
```
1. Stub the response.
2. Assert the call happened.
3. Simulate failure modes — the important tests.
4. Verify your error mapping / fallback.

Alternatively inject a fake client class via `dependency_overrides` — cleaner when the integration is behind a `Protocol`. For contract fidelity, record real responses with VCR.py and replay.

---

## Q: How do you mock database dependencies?

Prefer **not** mocking the DB for integration tests — use a real Postgres (testcontainers or CI service) with a per-test transaction rollback for speed and isolation:

```python
@pytest.fixture(scope="session")
async def engine():
    with PostgresContainer("postgres:16") as pg:                                   # 1
        eng = create_async_engine(pg.get_connection_url().replace("psycopg2", "asyncpg"))
        async with eng.begin() as conn:
            await conn.run_sync(Base.metadata.create_all)                          # 2 (or run alembic)
        yield eng
        await eng.dispose()

@pytest.fixture
async def db_session(engine):
    async with engine.connect() as conn:
        trans = await conn.begin()                                                 # 3
        session = AsyncSession(bind=conn, join_transaction_mode="create_savepoint")   # 4
        yield session
        await session.close()
        await trans.rollback()                                                     # 5
```
1. Real DB in Docker, once per session.
2. Schema.
3. Outer transaction per test.
4. Session joins it; `commit()` inside the app becomes a savepoint release.
5. Roll back everything → next test starts clean, no truncation needed.

For pure **unit tests** of services, use an in-memory fake repository implementing the same `Protocol` — no mocking library needed.

## Q: How do you override Depends() during testing?

`app.dependency_overrides[original] = replacement` (Section 4). Scope it in a fixture and `clear()` after. Override the **narrowest** dependency (e.g., `get_current_user`, not the whole router) so the rest of the pipeline is still exercised.

---

## Q: Unit tests vs integration tests?

| | Unit | Integration |
|---|---|---|
| Scope | One function/class in isolation | Several real components together (app + DB + cache) |
| Dependencies | Faked | Real (containers) or overridden at the boundary |
| Speed | ms | 10s–100s of ms |
| Finds | Logic bugs | Wiring, SQL, serialization, config bugs |
| Ratio | Many | Fewer, focused on critical paths |

Both are needed; the mistake is having only one kind. A service with 100% unit coverage and untested SQL will still break in production.

## Q: What should be mocked?

Things that are **slow, non-deterministic, external, or side-effecting** and outside your control: third-party HTTP APIs, payment gateways, email/SMS, clocks (`freezegun`), randomness, cloud SDKs (`moto` for AWS), message brokers (or use a fake publisher).

## Q: What shouldn't be mocked?

- **Your own code** (services, repositories) in integration tests — you'd be testing the mock.
- **The database** for anything involving queries, constraints, transactions — use a real one.
- **Pydantic/FastAPI** — they're the contract you're verifying.
- **Redis** in tests that exercise locking/rate-limiting semantics — use a container or `fakeredis`.

Golden rule: mock at the **boundary of your system**, not inside it.

---

## Q: How do you test asynchronous endpoints?

Use `pytest-asyncio` (`asyncio_mode = "auto"` in `pyproject.toml`) or the `anyio` plugin so `async def test_*` functions run in a loop, and `httpx.AsyncClient` with `ASGITransport`. Ensure fixtures are async too and share the same loop scope (`loop_scope="session"` for engine fixtures). Sync `TestClient` also works for async endpoints — it spins up its own loop — but can't be mixed with your own `await`s.

## Q: How do you test exception handlers?

Raise the domain exception from an overridden dependency or a tiny test route:

```python
async def test_app_error_envelope(client):
    async def boom(): raise NotFound("thing")
    app.dependency_overrides[get_order_service] = lambda: SimpleNamespace(get=lambda *_: boom())
    r = await client.get("/api/v1/orders/1")
    assert r.status_code == 404
    assert r.json()["error"] == {"code": "not_found", "message": "thing", "request_id": ANY}

async def test_unhandled_error_hidden(client, caplog):
    @app.get("/__boom")
    async def _(): raise RuntimeError("secret internal detail")
    r = await client.get("/__boom")
    assert r.status_code == 500 and "secret" not in r.text                # no leak
    assert "unhandled" in caplog.text                                     # logged
```
Note: `TestClient(app, raise_server_exceptions=False)` is needed for the 500 case with the sync client; `ASGITransport(app, raise_app_exceptions=False)` for httpx.

## Q: How do you test middleware?

Either through the app (assert the header/log the middleware adds) or in isolation by wrapping a trivial ASGI app:

```python
async def test_request_id_echoed(client):
    r = await client.get("/health", headers={"X-Request-ID": "abc-123"})
    assert r.headers["x-request-id"] == "abc-123"

async def test_request_id_generated(client):
    r = await client.get("/health")
    uuid.UUID(r.headers["x-request-id"])                                  # valid UUID

async def test_middleware_isolated():
    async def dummy(scope, receive, send):
        await send({"type": "http.response.start", "status": 200, "headers": []})
        await send({"type": "http.response.body", "body": b"ok"})
    mw_app = TimingMiddleware(dummy)
    async with AsyncClient(transport=ASGITransport(app=mw_app), base_url="http://t") as c:
        r = await c.get("/")
        assert "x-response-time-ms" in r.headers
```

## Q: How do you test database transactions?

Force a failure mid-use-case and assert nothing persisted:

```python
async def test_place_order_rolls_back_on_insufficient_stock(db_session, product_with_qty_1):
    svc = OrderService(OrderRepository(db_session), ProductRepository(db_session), FakePublisher(), db_session)
    with pytest.raises(InsufficientStock):
        await svc.place(user_id=1, items=[LineItem(sku="ok", qty=1), LineItem(sku="low", qty=5)])
    assert (await db_session.scalar(select(func.count()).select_from(Order))) == 0     # no order
    await db_session.refresh(product_with_qty_1)
    assert product_with_qty_1.qty == 1                                                  # first decrement undone
```
Also test: commit happens exactly once per request; exceptions in a `yield` dependency trigger rollback; savepoint behaviour for nested operations.

## Q: How do you test race conditions?

Deterministically, by running the contended operation concurrently against a real DB and asserting invariants:

```python
async def test_no_oversell(engine, product_with_qty_5):
    async def buy():
        async with AsyncSession(engine) as s:
            svc = OrderService(OrderRepository(s), ProductRepository(s), FakePublisher(), s)
            try: await svc.place(user_id=1, items=[LineItem(sku="p", qty=1)]); return 1
            except InsufficientStock: return 0
    results = await asyncio.gather(*(buy() for _ in range(20)))       # 20 buyers, 5 units
    assert sum(results) == 5                                          # exactly 5 succeed
    async with AsyncSession(engine) as s:
        assert (await s.get(Product, product_with_qty_5.id)).qty == 0
```
Requires **separate sessions/connections** per task (the per-test rollback fixture won't work here; use a truncate-after fixture). Use `SELECT … FOR UPDATE` or optimistic version columns in the code under test; the test proves it. For lock-ordering deadlocks, run with `pytest-repeat`. Also test idempotency-key races the same way.

---

## Q: How do you measure test coverage?

```bash
pytest --cov=app --cov-report=term-missing --cov-report=xml --cov-fail-under=85
```
- `term-missing` shows uncovered lines; `xml` feeds CI/Sonar/Codecov.
- `--cov-branch` for branch coverage (catches untested `else` paths).
- Treat coverage as a **smell detector**, not a goal; 100% with weak assertions is worthless. Enforce a floor and *ratchet* it upward; require coverage on changed lines in PRs (diff-cover).

## Q: How would you integrate FastAPI tests into GitLab CI/CD?

```yaml
# .gitlab-ci.yml
stages: [lint, test, build]

variables:
  POSTGRES_DB: test
  POSTGRES_USER: test
  POSTGRES_PASSWORD: test
  DATABASE_URL: postgresql+asyncpg://test:test@postgres:5432/test        # 1
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
  services:                                                              # 2
    - postgres:16
    - redis:7
  cache: { key: { files: [poetry.lock] }, paths: [.venv/] }              # 3
  before_script:
    - pip install poetry && poetry config virtualenvs.in-project true && poetry install --no-root
  script:
    - poetry run alembic upgrade head                                     # 4
    - poetry run pytest -n auto --cov=app --cov-report=xml --cov-report=term --junitxml=report.xml   # 5
  coverage: '/TOTAL.*\s(\d+%)/'                                          # 6
  artifacts:
    when: always
    reports:
      junit: report.xml                                                   # 7
      coverage_report: { coverage_format: cobertura, path: coverage.xml }
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH == "main"
```
1. Test env config via variables (host names match service aliases).
2. Sidecar containers for DB/Redis.
3. Cache dependencies between runs.
4. Run migrations so tests hit the real schema (also validates migrations).
5. Parallel tests (`pytest-xdist`), coverage, JUnit output.
6. Regex so GitLab shows coverage % on the MR.
7. Test results render in the MR UI; failing tests block merge via required pipelines.

---

# 13. Observability & Monitoring

## Q: How do you monitor a FastAPI application?

The three pillars, wired together by IDs:
- **Metrics** (Prometheus): numbers over time — rates, latencies, saturation. Cheap, aggregated, good for alerting.
- **Logs** (structured JSON → Loki/ELK/CloudWatch): discrete events with context. Good for "what happened to request X".
- **Traces** (OpenTelemetry → Tempo/Jaeger/X-Ray): the path of one request across services. Good for "where did the time go".

Plus **health probes**, **synthetic checks** (uptime pings from outside), and **error tracking** (Sentry) for stack traces with context. Every log line carries `trace_id`/`request_id` so you can pivot from a metric alert → traces → logs.

---

## Q: What metrics would you expose?

```
http_requests_total{method,route,status}              counter   → rate, error ratio
http_request_duration_seconds{method,route}           histogram → p50/p95/p99
http_requests_in_progress{route}                      gauge     → concurrency / saturation
db_pool_connections_in_use / db_pool_connections_waiting    gauge
db_query_duration_seconds{operation}                  histogram
external_call_duration_seconds{target}                histogram
external_call_failures_total{target,reason}           counter
cache_requests_total{result="hit|miss"}               counter   → hit ratio
event_loop_lag_seconds                                gauge     → blocking detection
background_jobs_total{type,status}, queue_depth{queue} counter/gauge
process_resident_memory_bytes, process_cpu_seconds_total (default exporter)
build_info{version,commit}                            gauge=1  → correlate deploys
```
Keep label cardinality bounded (route templates, not raw paths; no user IDs).

---

## Q: What is Prometheus?

An open-source metrics system: a **pull-based** time-series DB that scrapes `/metrics` endpoints on an interval, stores samples, and evaluates alert rules (PromQL). Metric types: **counter** (only up), **gauge** (up/down), **histogram** (buckets + sum + count → percentiles), **summary** (client-side quantiles). Grafana visualizes it; Alertmanager routes alerts.

## Q: How do you expose /metrics?

```python
from prometheus_client import Counter, Histogram, Gauge, generate_latest, CONTENT_TYPE_LATEST
from fastapi import Response

REQUESTS = Counter("http_requests_total", "Total requests", ["method", "route", "status"])
LATENCY = Histogram("http_request_duration_seconds", "Latency", ["method", "route"],
                    buckets=(.005,.01,.025,.05,.1,.25,.5,1,2.5,5,10))
IN_PROGRESS = Gauge("http_requests_in_progress", "In-flight", ["route"])

class MetricsMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request, call_next):
        route = "unmatched"
        IN_PROGRESS.labels(route).inc()
        start = time.perf_counter()
        status = 500
        try:
            response = await call_next(request)
            status = response.status_code
            return response
        finally:
            r = request.scope.get("route")                      # 1 route template after routing
            route = r.path if r else "unmatched"
            LATENCY.labels(request.method, route).observe(time.perf_counter() - start)
            REQUESTS.labels(request.method, route, status).inc()
            IN_PROGRESS.labels(route).dec()

@app.get("/metrics", include_in_schema=False)
def metrics():                                                   # 2
    return Response(generate_latest(), media_type=CONTENT_TYPE_LATEST)
```
1. Read the matched route after `call_next` so labels use `/orders/{id}`.
2. Serve the text exposition format. Or use `prometheus-fastapi-instrumentator` which does all of this. With multiple Uvicorn workers, set `PROMETHEUS_MULTIPROC_DIR` so metrics aggregate across processes — or run one worker per pod and let Prometheus scrape each pod (simpler; recommended).

---

## Q: What is OpenTelemetry?

**OTel** is the vendor-neutral standard (CNCF) for generating and exporting telemetry — traces, metrics, logs — with a single SDK, auto-instrumentation libraries, a wire protocol (**OTLP**), and a **Collector** that receives, processes (sampling, redaction) and exports to any backend (Jaeger, Tempo, Datadog, X-Ray, Honeycomb). It replaced OpenTracing + OpenCensus.

## Q: What is distributed tracing?

Following a single request as it fans out across services, queues and databases, recording timing and metadata at each hop, so you can see the whole causal chain as one waterfall. It answers "which of the 7 services made this checkout slow?" — something per-service logs can't.

## Q: What is a trace?

The complete record of one request's journey: a tree (DAG) of spans sharing a single **trace ID** (128-bit). Root span = the entry point (gateway or first service).

## Q: What is a span?

One unit of work within a trace: a name (`GET /orders/{id}`, `SELECT orders`, `call payment-service`), start/end timestamps, a **span ID**, a **parent span ID**, attributes (`http.status_code`, `db.statement`), events (logs within the span), and status. Spans nest: server span → dependency span → SQL span.

## Q: What is a correlation ID?

An application-level identifier (usually a UUID) attached to a request and propagated through logs and downstream calls so all log lines for one operation can be grouped. Often exposed to clients as `X-Request-ID` for support tickets.

## Q: Correlation ID vs trace ID?

| | Correlation / Request ID | Trace ID |
|---|---|---|
| Origin | Ad-hoc convention, app-generated | W3C Trace Context standard, OTel SDK |
| Scope | One logical operation, mostly for logs | Whole distributed trace, structured spans |
| Timing data | ❌ | ✅ |
| Client-facing | Usually | Sometimes (`traceparent` echoed) |

In practice: use the OTel trace ID as *the* correlation ID (inject it into logs) and echo it as `X-Request-ID`; keep a separate business correlation ID only for multi-request workflows (e.g., an order ID spanning many traces).

---

## Q: How do you propagate trace context between microservices?

Via the **W3C `traceparent` header** (`00-<trace_id>-<parent_span_id>-<flags>`) and optional `tracestate`/`baggage`. OTel's HTTP client instrumentation injects it on outgoing requests; server instrumentation extracts it. For queues, put the context into message attributes:

```python
from opentelemetry import propagate, trace

# producer
carrier: dict[str, str] = {}
propagate.inject(carrier)                                            # {'traceparent': '00-...'}
await sqs.send_message(QueueUrl=Q, MessageBody=body,
    MessageAttributes={k: {"DataType": "String", "StringValue": v} for k, v in carrier.items()})

# consumer
carrier = {k: v["StringValue"] for k, v in msg.get("MessageAttributes", {}).items()}
ctx = propagate.extract(carrier)
with tracer.start_as_current_span("process_export", context=ctx, kind=trace.SpanKind.CONSUMER):
    await process_job(job_id)
```
Consumer spans link back to the producer so the async hop appears in the same trace.

## Q: How would you instrument FastAPI with OpenTelemetry?

Shown in Section 6 (`FastAPIInstrumentor`, `HTTPXClientInstrumentor`, `SQLAlchemyInstrumentor`), plus `RedisInstrumentor`, `BotocoreInstrumentor`, `LoggingInstrumentor` (adds `trace_id` to log records). Zero-code alternative: `opentelemetry-instrument uvicorn app.main:app` with env vars `OTEL_SERVICE_NAME`, `OTEL_EXPORTER_OTLP_ENDPOINT`, `OTEL_TRACES_SAMPLER=parentbased_traceidratio`, `OTEL_TRACES_SAMPLER_ARG=0.1`. Add custom spans around business logic:

```python
tracer = trace.get_tracer("orders")
async def place(self, user_id, items):
    with tracer.start_as_current_span("OrderService.place") as span:
        span.set_attribute("order.item_count", len(items))
        ...
```
Ship to an OTel Collector sidecar/daemonset (batching, tail sampling, PII scrubbing) rather than directly to the backend.

---

## Q: How would you monitor external API latency?

- Wrap every outbound client in a histogram + counter labelled by `target` and `outcome` (auto with OTel/httpx instrumentation; or an `httpx` event hook):

```python
async def on_response(response: httpx.Response):
    EXT_LATENCY.labels(response.request.url.host).observe(response.elapsed.total_seconds())
    EXT_CALLS.labels(response.request.url.host, str(response.status_code)).inc()
client = httpx.AsyncClient(event_hooks={"response": [on_response]})
```
- Dashboards per dependency: p99, error rate, timeout rate, circuit-breaker state.
- Alert when a dependency's p99 exceeds your budget for it, before your own SLO burns.
- Synthetic checks from outside (Blackbox exporter) to separate "they're slow" from "we're slow reaching them".

---

## Q: How would you create useful Grafana dashboards?

Design top-down per service:
1. **Header row**: SLO status (availability %, latency %), error budget remaining, current version, active alerts.
2. **RED row**: rps, error rate (4xx/5xx separately), latency p50/p95/p99 — with deploy annotations.
3. **Per-endpoint table**: sorted by p99 or error rate.
4. **Saturation row**: CPU/memory vs limits, pod count vs HPA max, event-loop lag, DB pool usage, threadpool queue.
5. **Dependencies row**: DB/Redis/external latency & errors.
6. **Async row**: queue depth, job success/fail, DLQ size, consumer lag.

Principles: consistent units and colors; templated variables (`$env`, `$service`); links from panels to traces/logs (Grafana Explore with trace ID); keep the "on-call" dashboard to one screen; version dashboards as code (JSON in git / Grafonnet).

## Q: What alerts would you create?

Alert on **symptoms** that affect users, not on every cause. Use **SLO burn-rate** alerts (Google SRE style):
- Availability: 5xx ratio burn rate > 14.4× over 1 h **and** 5 m (page); > 6× over 6 h (ticket).
- Latency: p99 above SLO for 10 m.
- Traffic drop to ~0 (something broke upstream).
- Saturation predictive: DB pool waiting > 0 for 5 m; memory > 85% of limit; HPA at max replicas for 15 m; disk/PVC > 80%.
- Async: DLQ depth > 0; consumer lag growing for 10 m; job failure rate > 5%.
- Infra: CrashLoopBackOff, pod restarts > 3 in 15 m, certificate expiry < 14 d.
- Dependency: circuit breaker open.

Every alert has a runbook link, severity, and an owner. Prune noisy alerts weekly.

---

## Q: How would you detect memory leaks?

- **Symptom**: `process_resident_memory_bytes` climbing monotonically across restarts, `OOMKilled` events, memory not returning after load drops.
- **Confirm**: compare per-pod memory against request count — leak grows with *requests*, not time.
- **Locate**: `tracemalloc` snapshots diffed over time in a debug endpoint; `memray run` in staging under load; `objgraph.show_growth()` to find growing types; `py-spy dump` for stuck tasks.
- **Usual suspects**: module-level caches/lists, unbounded `lru_cache`, per-request `httpx.AsyncClient()` never closed, unawaited tasks, large objects held by closures/exception tracebacks, `contextvars` set but not reset, log handlers accumulating.
- **Mitigate** while fixing: `--max-requests` worker recycling, memory limits + HPA.

## Q: How would you troubleshoot high CPU?

1. Is it *real* work (rps rose) or *waste*? Check rps vs CPU correlation and CPU throttling (`container_cpu_cfs_throttled_seconds_total`) — throttling from low limits looks like "high CPU" with high latency.
2. `py-spy top --pid <pid>` on a hot pod → which functions burn CPU: JSON serialization of huge responses, regex, Pydantic on giant lists, tight loops, `json.loads` on big bodies, bcrypt with too-high cost, GC churn.
3. Check for CPU-bound work in request handlers → move to workers.
4. Check for busy loops: a `while True` polling without sleep, retry storms, log volume (formatting is CPU).
5. Profile with `py-spy record -o profile.svg` for a flame graph under load; fix the top frames.
6. Right-size: raise limits, add pods, use `orjson`, cache computed results.

## Q: How would you troubleshoot increasing latency?

Covered in detail in Section 5 ("latency suddenly increased"). Compress it: scope (which endpoints, p50 vs p99, since when) → correlate with deploys/traffic → look at saturation and downstream spans → check for blocking calls, pool exhaustion, N+1, missing indexes, GC/memory pressure → `py-spy` for live view → rollback/scale to mitigate → root-cause fix + alert.

## Q: How would you troubleshoot HTTP 5xx spikes?

1. **Which status?** 500 (app exception) vs 502/504 (proxy can't reach or upstream timed out) vs 503 (readiness failing / load shed). This alone splits the search space.
2. **Which routes, which pods?** All pods → shared dependency or bad deploy; one pod → node/pod issue.
3. **Error tracker / logs** grouped by exception type with trace IDs: the top exception usually explains 90%.
4. **Recent changes**: deploy, config, migration, secret rotation, dependency outage, certificate expiry, quota.
5. **Downstream health**: DB connections refused, Redis OOM, external 5xx — check their dashboards.
6. **Resource**: OOMKilled restarts causing 502s during restart; CPU throttling causing 504s.
7. **Mitigate**: rollback, scale, open the circuit breaker / enable degraded mode, block an abusive client.
8. **Postmortem** with a timeline and action items (missing alert, missing test, missing timeout).

## Q: What are the four golden signals?

From Google's SRE book — the minimum to monitor for any user-facing system:
1. **Latency** – time to serve a request; track successful and failed requests separately.
2. **Traffic** – demand (rps, active sessions, messages/s).
3. **Errors** – rate of failed requests (explicit 5xx, implicit wrong content, policy failures like "slower than SLO").
4. **Saturation** – how "full" the service is (CPU, memory, pool utilization, queue depth); predicts problems before latency/errors show.

RED (rate, errors, duration) is the request-centric subset; USE (utilization, saturation, errors) is the resource-centric complement.

---

# 14. Docker & Kubernetes

## Q: How do you Dockerize a FastAPI application?

Goals: small, reproducible, secure, fast to build. Use a multi-stage build with a slim base, a lockfile, and a non-root user.

## Q: What should your Dockerfile look like?

```dockerfile
# ---------- Stage 1: build dependencies ----------
FROM python:3.12-slim AS builder                                   # 1
ENV PIP_NO_CACHE_DIR=1 PIP_DISABLE_PIP_VERSION_CHECK=1 \
    UV_PROJECT_ENVIRONMENT=/opt/venv                               # 2
RUN apt-get update && apt-get install -y --no-install-recommends build-essential libpq-dev \
    && rm -rf /var/lib/apt/lists/*                                  # 3
COPY --from=ghcr.io/astral-sh/uv:latest /uv /usr/local/bin/uv      # 4
WORKDIR /src
COPY pyproject.toml uv.lock ./                                     # 5
RUN uv sync --frozen --no-dev --no-install-project                 # 6

# ---------- Stage 2: runtime ----------
FROM python:3.12-slim AS runtime                                   # 7
ENV PYTHONDONTWRITEBYTECODE=1 PYTHONUNBUFFERED=1 \
    PATH="/opt/venv/bin:$PATH"                                     # 8
RUN apt-get update && apt-get install -y --no-install-recommends libpq5 curl \
    && rm -rf /var/lib/apt/lists/* \
    && groupadd -r app && useradd -r -g app -d /app -s /sbin/nologin app   # 9
WORKDIR /app
COPY --from=builder /opt/venv /opt/venv                            # 10
COPY --chown=app:app app ./app                                     # 11
COPY --chown=app:app alembic.ini ./
COPY --chown=app:app migrations ./migrations
USER app                                                           # 12
EXPOSE 8000
HEALTHCHECK --interval=30s --timeout=3s CMD curl -fsS http://localhost:8000/health/live || exit 1   # 13
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000", "--workers", "1", \
     "--proxy-headers", "--forwarded-allow-ips=*", "--timeout-graceful-shutdown", "20"]   # 14
```
1. Build stage on a slim image; pin a digest in production (`python:3.12-slim@sha256:...`).
2. No pip cache; install into a venv path we'll copy later.
3. Compilers only in the build stage; clean apt lists to shrink layers.
4. `uv` for fast, lock-based installs (poetry/pip-tools work the same way).
5. Copy *only* dependency manifests first → this layer is cached until deps change.
6. Install exactly the locked versions, no dev deps.
7. Fresh runtime stage: no compilers, no build artefacts.
8. Don't write `.pyc`; unbuffered stdout so logs stream; venv on PATH.
9. Only runtime libs (`libpq5` for asyncpg/psycopg); create an unprivileged user.
10. Bring in the ready venv.
11. Copy source last (changes most often) with correct ownership.
12. Drop privileges.
13. Container-level health check (Kubernetes uses probes instead; harmless to keep).
14. `exec` form so Uvicorn is PID 1 and receives SIGTERM; one worker per container; graceful shutdown timeout for in-flight requests.

Also add a `.dockerignore` (`.git`, `tests`, `.venv`, `__pycache__`, `.env`).

---

## Q: Why use a non-root user?

Container isolation is not a security boundary you should rely on alone. If the app is exploited (RCE via a dependency), root inside the container can: write anywhere in the filesystem, install tools, and exploit kernel/runtime bugs to escape to the host. A non-root user, combined with a read-only root filesystem and dropped capabilities, shrinks the blast radius. Many clusters enforce it via Pod Security Standards (`runAsNonRoot: true`) — a root image simply won't schedule.

## Q: How do you reduce FastAPI image size?

- Slim base (`python:3.12-slim` ~150 MB vs full ~1 GB); Alpine is smaller but musl breaks wheels — usually not worth it for Python.
- Multi-stage: no compilers/headers in runtime.
- No dev dependencies, no tests, no `.git` (`.dockerignore`).
- Combine `apt-get` install + cleanup in one `RUN`.
- Use wheels (avoid building from source), `--no-cache-dir`.
- Remove `*.pyc`/`__pycache__` or set `PYTHONDONTWRITEBYTECODE`.
- Inspect with `dive` to find fat layers; typical result: 180–250 MB.
- Distroless (`gcr.io/distroless/python3`) for minimal attack surface if you don't need a shell.

## Q: What is a multi-stage Docker build?

Multiple `FROM` stages in one Dockerfile; later stages `COPY --from=` earlier ones. Build tools, secrets, and intermediate files stay in the builder stage and never reach the final image. Result: smaller, safer images and better layer caching.

---

## Q: How do you pass environment variables to FastAPI?

The app reads them via `pydantic-settings`. Sources by environment:
- `docker run -e DATABASE_URL=... --env-file .env`
- Compose: `environment:` / `env_file:`
- Kubernetes: `env:` with literal values, `valueFrom.configMapKeyRef`, `valueFrom.secretKeyRef`, or `envFrom: [{configMapRef}, {secretRef}]`.
Never bake them into the image (`ENV DATABASE_URL=` in a Dockerfile is a leak).

## Q: How do you manage secrets in Kubernetes?

Kubernetes `Secret` objects are base64, **not encrypted** by default — enable encryption at rest (KMS provider) and RBAC-restrict who can read them. Better: keep the source of truth in AWS Secrets Manager/Vault and sync with **External Secrets Operator** or mount via **Secrets Store CSI Driver**; the pod's IAM role (IRSA) authorizes the fetch. Prefer mounting as files over env vars (env vars leak into `kubectl describe`, crash dumps, child processes). Rotate regularly; never commit manifests with real values (use Sealed Secrets / SOPS if you must store them in git).

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata: { name: orders-api }
spec:
  refreshInterval: 1h
  secretStoreRef: { name: aws-secrets-manager, kind: ClusterSecretStore }
  target: { name: orders-api-secrets }
  data:
    - secretKey: DATABASE_URL
      remoteRef: { key: prod/orders-api, property: database_url }
    - secretKey: JWT_SECRET
      remoteRef: { key: prod/orders-api, property: jwt_secret }
```

---

## Q: How do you deploy FastAPI on Kubernetes?

Minimum objects: `Deployment` (pods), `Service` (stable virtual IP), `Ingress`/`Gateway` (external HTTP), `ConfigMap`/`Secret`, `HPA`, `PodDisruptionBudget`, optionally `ServiceAccount` with IRSA. Packaged as a Helm chart or Kustomize overlays; applied by CI/CD or GitOps (ArgoCD).

```yaml
apiVersion: apps/v1
kind: Deployment
metadata: { name: orders-api, labels: { app: orders-api } }
spec:
  replicas: 3
  selector: { matchLabels: { app: orders-api } }
  strategy:
    type: RollingUpdate
    rollingUpdate: { maxSurge: 1, maxUnavailable: 0 }              # 1
  template:
    metadata:
      labels: { app: orders-api, version: "1.4.2" }
      annotations: { prometheus.io/scrape: "true", prometheus.io/port: "8000" }
    spec:
      serviceAccountName: orders-api                                 # 2
      securityContext: { runAsNonRoot: true, runAsUser: 1000, fsGroup: 1000, seccompProfile: { type: RuntimeDefault } }
      topologySpreadConstraints:                                     # 3
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: ScheduleAnyway
          labelSelector: { matchLabels: { app: orders-api } }
      containers:
        - name: api
          image: 123456789.dkr.ecr.eu-west-1.amazonaws.com/orders-api:1.4.2   # 4
          ports: [{ containerPort: 8000, name: http }]
          envFrom:
            - configMapRef: { name: orders-api-config }
            - secretRef: { name: orders-api-secrets }
          resources:                                                 # 5
            requests: { cpu: "500m", memory: "512Mi" }
            limits:   { cpu: "1",    memory: "512Mi" }
          startupProbe:                                              # 6
            httpGet: { path: /health/live, port: http }
            failureThreshold: 30
            periodSeconds: 2
          readinessProbe:
            httpGet: { path: /health/ready, port: http }
            periodSeconds: 5
            failureThreshold: 3
          livenessProbe:
            httpGet: { path: /health/live, port: http }
            periodSeconds: 10
            failureThreshold: 3
          lifecycle:
            preStop: { exec: { command: ["sh", "-c", "sleep 5"] } }  # 7
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities: { drop: ["ALL"] }
          volumeMounts: [{ name: tmp, mountPath: /tmp }]
      volumes: [{ name: tmp, emptyDir: {} }]
      terminationGracePeriodSeconds: 30                              # 8
---
apiVersion: v1
kind: Service
metadata: { name: orders-api }
spec:
  selector: { app: orders-api }
  ports: [{ port: 80, targetPort: http }]
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata: { name: orders-api }
spec:
  minAvailable: 2
  selector: { matchLabels: { app: orders-api } }
```
1. Zero-downtime rolling update: add one new pod, never drop below desired count.
2. IAM role via IRSA for S3/SQS/Secrets access — no static keys.
3. Spread pods across AZs.
4. Immutable tag (never `latest`).
5. Requests for scheduling/HPA; memory limit = request (avoid OOM surprises); CPU limit optional (many teams omit it to avoid throttling).
6. Startup probe protects slow-starting apps from liveness kills.
7. Small sleep lets the endpoint removal propagate before SIGTERM (avoids 502s during rollout).
8. Time for in-flight requests to finish after SIGTERM.

---

## Q: Deployment vs StatefulSet?

- **Deployment**: interchangeable, stateless pods with random names; any pod can serve any request; scale/roll freely. **FastAPI belongs here.**
- **StatefulSet**: stable identities (`db-0`, `db-1`), stable per-pod storage (PVC per replica), ordered startup/shutdown. For databases, Kafka, Zookeeper — things that own state.
If your API "needs" a StatefulSet, that's a design smell: move the state to a DB/Redis/S3.

## Q: How do you configure readiness probes?

Readiness answers "can this pod take traffic right now?" Failure → pod removed from Service endpoints, **not** restarted. Check the things needed to serve: DB reachable (cheap `SELECT 1` with a timeout), required config loaded, warm caches if applicable. Don't check optional dependencies — a degraded feature shouldn't take the whole pod out. Tune `periodSeconds`/`failureThreshold` so a transient blip doesn't flap.

## Q: How do you configure liveness probes?

Liveness answers "is this process hopelessly stuck?" Failure → container **restarted**. Keep it trivial and dependency-free: return 200 if the event loop can respond. A liveness probe that checks the DB will restart every pod during a DB outage — making things worse. Use `startupProbe` for slow boots instead of a long `initialDelaySeconds`.

## Q: What should /health return?

Split it:
```python
@router.get("/health/live", include_in_schema=False)
async def live(): return {"status": "ok"}                            # process alive, loop responsive

@router.get("/health/ready", include_in_schema=False)
async def ready(db: DB, redis: RedisDep):
    checks = {}
    async with asyncio.timeout(2):
        checks["db"] = await _check(db.execute(text("SELECT 1")))
        checks["redis"] = await _check(redis.ping())
    ok = all(checks.values())
    return JSONResponse({"status": "ready" if ok else "degraded", "checks": checks,
                         "version": settings.version}, status_code=200 if ok else 503)
```
Keep responses small and unauthenticated but not exposed publicly (ingress path rule). Include version/commit for debugging. Cache the readiness result for a second or two under heavy probing.

## Q: What is the difference between liveness and readiness?

| | Liveness | Readiness |
|---|---|---|
| Question | Should this container be **restarted**? | Should this pod **receive traffic**? |
| On failure | Kill & restart | Remove from Service endpoints |
| Checks | Process health only | Dependencies needed to serve |
| Typical bug | Checking DB → restart storms | Missing it → traffic to unready pods → 502s |

---

## Q: How do you perform zero-downtime deployments?

1. **RollingUpdate** with `maxUnavailable: 0`, `maxSurge: ≥1`.
2. **Readiness probe** so new pods only get traffic when ready.
3. **Graceful shutdown**: on SIGTERM Uvicorn stops accepting, finishes in-flight requests (`--timeout-graceful-shutdown`); `preStop` sleep covers the endpoint-propagation race; `terminationGracePeriodSeconds` > longest request.
4. **PodDisruptionBudget** so node drains don't drop below capacity.
5. **Backward-compatible DB migrations** (expand/contract) run *before* rollout.
6. **Backward-compatible API changes** so old and new versions coexist.
7. **Client retries** on idempotent requests for the rare connection reset.
8. Progressive delivery (canary via Argo Rollouts) for extra safety.

## Q: How do you configure resource requests and limits?

- **Requests** = what the scheduler reserves and what HPA utilization is relative to. Set from observed p95 usage under normal load.
- **Limits** = hard cap. Memory limit exceeded → OOMKilled. CPU limit exceeded → throttled (latency spikes).
- Guidelines: memory `request == limit` (Guaranteed QoS, predictable); CPU request realistic, CPU limit either generous or omitted (avoid CFS throttling); one Uvicorn worker per ~1 CPU; measure with VPA in recommendation mode; set namespace `LimitRange`/`ResourceQuota`.

## Q: How does HPA work?

Every 15 s (default) the HPA controller reads metrics (Metrics Server for CPU/memory; Prometheus Adapter/KEDA for custom), computes `desired = ceil(current × (currentMetric / targetMetric))`, applies scale-up/down policies and stabilization windows, and patches the Deployment's replica count. Pods must have resource requests for utilization metrics. Works with Cluster Autoscaler/Karpenter to add nodes when pods are `Pending`.

---

## Q: How would you troubleshoot CrashLoopBackOff?

The container starts, exits, and Kubernetes restarts it with growing backoff.
1. `kubectl describe pod <p>` → exit code, last state, events (image pull, OOM, probe failures).
2. `kubectl logs <p> --previous` → the crashing container's output (the current one may be empty).
3. Exit codes: `1` app exception (config missing, import error, migration mismatch); `137` OOMKilled/SIGKILL; `143` SIGTERM; `126/127` bad command/entrypoint.
4. Common FastAPI causes: missing env var → pydantic-settings `ValidationError`; wrong module path in CMD; DB unreachable at import time; port already bound; liveness probe too aggressive killing a slow-starting app (fix with `startupProbe`).
5. Reproduce locally with the same image and env: `docker run --rm -e ... image`.
6. `kubectl debug` / override command to `sleep infinity` and exec in to inspect.

## Q: How would you troubleshoot OOMKilled?

1. Confirm: `kubectl describe pod` shows `Last State: Terminated, Reason: OOMKilled`, exit code 137.
2. Was it a **spike** (one big request: file upload read into memory, huge query result) or a **leak** (steady climb)? Check memory graphs.
3. Spike → stream instead of buffering, paginate, limit body size at ingress, cap `yield_per`.
4. Leak → Section 13 tooling (`tracemalloc`, `memray`).
5. Wrong sizing → too many workers per pod (each is a full copy of the app), or limit set below realistic peak; raise limit with request = limit.
6. Add `--max-requests` recycling as a stopgap and alert at 80% of limit.

## Q: How would you troubleshoot slow Kubernetes pods?

1. **CPU throttling**: `container_cpu_cfs_throttled_periods_total` high → raise/remove CPU limit.
2. **Node pressure**: `kubectl top node`; noisy neighbours; use requests to guarantee share, or dedicated node pools.
3. **Network**: cross-AZ hops to DB (put pods and DB in the same AZ or use zone-aware routing); DNS latency (ndots:5 issue — set `dnsConfig options ndots:2` or use FQDNs); kube-proxy/conntrack exhaustion.
4. **Readiness flapping**: pods removed/added, connections reset.
5. **Cold start**: JIT-less Python is fine, but big imports/model loads → startup probe + pre-warming.
6. **Storage**: slow PVC/EBS for anything writing to disk (logs → stdout instead).
7. Compare a slow pod vs a fast pod with `py-spy` and tracing to isolate app vs infra.

## Q: How would you perform a rollback?

- **Imperative**: `kubectl rollout undo deployment/orders-api` (to previous ReplicaSet) or `--to-revision=N`; `kubectl rollout history` to see revisions. Fast, but drifts from git.
- **GitOps (preferred)**: revert the commit that changed the image tag; ArgoCD syncs to the previous state. Auditable and reproducible.
- **Helm**: `helm rollback orders-api <revision>`.
- **Argo Rollouts**: automatic rollback when canary analysis fails.
- DB: only roll back code if the schema is still compatible (expand/contract guarantees this for one version). Never `alembic downgrade` in prod reflexively — data written by the new version may be lost; prefer roll-forward fixes.

---

# 15. CI/CD

## Q: How would you build a CI/CD pipeline for FastAPI?

Principles: fast feedback (lint/unit in < 5 min), one immutable artefact promoted through environments, everything as code, security gates that block, and deploy via GitOps.

```
PR opened ──► lint + type-check ──► unit tests ──► integration tests (DB/Redis services)
          ──► dependency & secret scan ──► build image (multi-stage, SBOM) ──► image scan
          ──► push :sha to registry ──► update dev overlay (GitOps) ──► ArgoCD syncs dev
          ──► smoke tests vs dev ──► [manual approval] ──► promote same tag to staging ──► e2e
          ──► promote to prod (canary) ──► smoke + SLO check ──► auto-rollback on failure
```

```yaml
# .gitlab-ci.yml (abridged, extends the test job from Section 12)
stages: [lint, test, security, build, scan, push, deploy-dev, verify, deploy-prod]

.python: { image: python:3.12-slim, cache: { key: { files: [uv.lock] }, paths: [.venv/] } }

lint:
  extends: .python
  stage: lint
  script: [pip install ruff mypy, ruff check ., ruff format --check ., mypy app]

test: { extends: .python, stage: test, services: [postgres:16, redis:7], script: [pytest -n auto --cov=app --cov-fail-under=85 --junitxml=report.xml], artifacts: { reports: { junit: report.xml } } }

deps-scan:
  extends: .python
  stage: security
  script: [pip install pip-audit, pip-audit -r requirements.txt --strict]        # 1

secret-scan:
  stage: security
  image: zricethezav/gitleaks:latest
  script: [gitleaks detect --source . --no-git -v]                                 # 2

sast:
  extends: .python
  stage: security
  script: [pip install bandit, bandit -r app -ll]                                  # 3

build:
  stage: build
  image: gcr.io/kaniko-project/executor:debug                                      # 4
  script:
    - /kaniko/executor --context $CI_PROJECT_DIR --dockerfile Dockerfile
        --destination $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA --cache=true
        --label org.opencontainers.image.revision=$CI_COMMIT_SHA

image-scan:
  stage: scan
  image: aquasec/trivy:latest
  script:
    - trivy image --exit-code 1 --severity HIGH,CRITICAL --ignore-unfixed $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA   # 5
    - trivy image --format cyclonedx --output sbom.json $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA                    # 6
  artifacts: { paths: [sbom.json] }

sign:
  stage: push
  image: gcr.io/projectsigstore/cosign:latest
  script: [cosign sign --yes $CI_REGISTRY_IMAGE@$IMAGE_DIGEST]                       # 7

deploy-dev:
  stage: deploy-dev
  image: alpine/git
  script:
    - git clone https://oauth2:${GITOPS_TOKEN}@gitlab.com/org/gitops.git && cd gitops
    - cd apps/orders-api/overlays/dev && kustomize edit set image orders-api=$CI_REGISTRY_IMAGE:$CI_COMMIT_SHA   # 8
    - git commit -am "orders-api dev → $CI_COMMIT_SHA" && git push
  rules: [{ if: $CI_COMMIT_BRANCH == "main" }]

smoke-dev:
  stage: verify
  script: [./scripts/smoke.sh https://dev.api.example.com]                          # 9

deploy-prod:
  stage: deploy-prod
  script: [same as deploy-dev but overlays/prod]
  when: manual                                                                       # 10
  environment: { name: production, url: https://api.example.com }
```
1. Fail on known-vulnerable Python packages.
2. Block committed secrets.
3. Static analysis for insecure code patterns.
4. Rootless image build in CI (no Docker socket).
5. Block images with unfixed HIGH/CRITICAL CVEs.
6. Software Bill of Materials for audit.
7. Sign the image; the cluster's admission policy (Kyverno/Sigstore policy-controller) rejects unsigned images.
8. GitOps: CI only edits a manifest; ArgoCD does the deploy. Same image tag flows through every environment.
9. Post-deploy verification.
10. Manual gate for prod (or automatic with canary analysis).

## Q: What checks should happen before deployment?

Matching the diagram in the question:
- **Lint / format / type-check** (ruff, mypy) — cheap, catches trivial bugs.
- **Unit tests** — logic.
- **Integration tests** — SQL, wiring, migrations apply cleanly.
- **Security scan** — dependencies (pip-audit/Safety), secrets (gitleaks), SAST (bandit/Semgrep), IaC (checkov) for Helm/Terraform.
- **Build Docker image** — reproducible, pinned base.
- **Image scan** — Trivy/Grype for OS + Python CVEs; SBOM; signature.
- **Push registry** — immutable tag (`:sha`), signed.
- **Deploy** (dev → staging → prod).
- **Smoke tests** — health, auth, one critical write path; rollback on failure.
- Also: OpenAPI contract diff (no breaking changes), license check, coverage threshold, DB migration dry-run (`alembic upgrade head --sql`), load test on staging for risky changes.

---

## Q: How do you manage environment-specific configuration?

- Same image everywhere; only configuration differs.
- **Kustomize overlays** (`base/` + `overlays/{dev,staging,prod}`) or **Helm values** per env, stored in the GitOps repo.
- Non-secret config in `ConfigMap`; secrets via External Secrets referencing per-env paths in Secrets Manager (`/dev/orders-api`, `/prod/orders-api`).
- App validates config at startup; `env` field in settings drives behaviour differences (log level, docs on/off) — but avoid `if env == "prod"` sprinkled through code.
- Feature flags (LaunchDarkly/Unleash/ConfigCat) for runtime behaviour toggles, not env-specific builds.

## Q: How do you perform database migrations during deployment?

Run migrations as a **discrete step before the new pods roll out**, never in app startup:
- Kubernetes `Job` (or Helm `pre-upgrade` hook / Argo `PreSync` hook) that runs `alembic upgrade head` with the *new* image, with a DB user that has DDL rights (app user shouldn't).
- Migrations must be backward-compatible (expand/contract) so old pods keep working mid-rollout.
- `lock_timeout` and `statement_timeout` set in the migration session; retry on lock contention.
- Long backfills run as separate batch jobs, not inside the migration.
- Pipeline gate: `alembic check` (no un-generated model changes) and a dry run against a staging snapshot.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: orders-api-migrate
  annotations: { argocd.argoproj.io/hook: PreSync, argocd.argoproj.io/hook-delete-policy: HookSucceeded }
spec:
  backoffLimit: 2
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: migrate
          image: registry/orders-api:1.4.2
          command: ["alembic", "upgrade", "head"]
          envFrom: [{ secretRef: { name: orders-api-migrator-secrets } }]
```

## Q: How do you rollback application code?

Revert the GitOps commit (image tag back to the last good SHA) → ArgoCD syncs; or `kubectl rollout undo` / `helm rollback` in an emergency, followed by a git revert to reconcile. Because migrations are backward-compatible by policy, the previous code works against the new schema. Automate it: canary analysis or smoke-test failure triggers the revert.

## Q: How do you rollback database changes?

Prefer **roll forward**. Real rollback options in order of preference:
1. Don't need to — expand/contract means the old code tolerates the new schema; just roll back code.
2. **Compensating migration** (a new `upgrade()` that undoes the change safely) — auditable, goes through the pipeline.
3. `alembic downgrade <rev>` — only if the `downgrade()` is tested and non-destructive (dropping a column loses data written since).
4. Point-in-time restore from backup for catastrophic cases — accept data loss, communicate.
Test downgrades in CI (`upgrade head && downgrade -1 && upgrade head`) so they're at least mechanically valid.

---

## Q: How would you implement blue-green deployment?

Two identical environments; only one receives traffic. Deploy to the idle one, verify, switch the router.

Kubernetes-native: two Deployments (`orders-api-blue`, `orders-api-green`) and a Service whose selector `version: blue|green` you flip; or Argo Rollouts `strategy.blueGreen` with `activeService`/`previewService`, `autoPromotionEnabled: false`, and `prePromotionAnalysis` (smoke tests against the preview) → promote → `scaleDownDelaySeconds` keeps the old one for instant rollback.

Trade-offs: instant cutover and rollback; double capacity during deploy; DB must be compatible with both; long-lived connections/websockets need draining.

## Q: What is canary deployment?

Release the new version to a small slice of traffic (1% → 10% → 50% → 100%), comparing its error rate/latency against the baseline at each step; abort and roll back automatically if it regresses. Limits blast radius to the slice. Needs: traffic splitting (Argo Rollouts + NGINX/ALB/Istio, or Flagger), metrics per version (`version` label), and analysis templates with thresholds.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
spec:
  strategy:
    canary:
      canaryService: orders-api-canary
      stableService: orders-api-stable
      trafficRouting: { nginx: { stableIngress: orders-api } }
      steps:
        - setWeight: 5
        - pause: { duration: 5m }
        - analysis: { templates: [{ templateName: success-rate }] }
        - setWeight: 25
        - pause: { duration: 10m }
        - setWeight: 50
        - pause: { duration: 10m }
```

## Q: How would you deploy FastAPI using ArgoCD?

1. **GitOps repo** holds Kubernetes manifests (Kustomize/Helm) per app per environment.
2. ArgoCD `Application` points at that path and the cluster/namespace; `syncPolicy.automated` with `prune` and `selfHeal` so drift is corrected.
3. CI's only deploy action: bump the image tag in the overlay and push.
4. ArgoCD detects the diff, runs `PreSync` hooks (migration Job), applies the manifests, waits for health (Deployment rollout, probes), runs `PostSync` hooks (smoke tests).
5. Promotion between environments = a PR from dev overlay to staging/prod overlay (or an ApplicationSet with a generator).
6. Rollback = git revert (or ArgoCD UI "History and Rollback" for emergencies, then reconcile git).
7. Notifications to Slack on sync failure/degraded health; RBAC per project; SSO.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata: { name: orders-api-prod, namespace: argocd }
spec:
  project: platform
  source: { repoURL: https://gitlab.com/org/gitops.git, targetRevision: main, path: apps/orders-api/overlays/prod }
  destination: { server: https://kubernetes.default.svc, namespace: orders }
  syncPolicy:
    automated: { prune: true, selfHeal: true }
    syncOptions: [CreateNamespace=true, ApplyOutOfSyncOnly=true]
```

## Q: How would you manage secrets using AWS Secrets Manager?

- One secret (JSON) per service per environment: `prod/orders-api` = `{database_url, jwt_secret, ...}`.
- Pods get an IAM role via **IRSA** (ServiceAccount annotated with the role ARN); the role's policy allows `secretsmanager:GetSecretValue` on that ARN only.
- Delivery: **External Secrets Operator** (syncs to a K8s Secret on an interval) or **Secrets Store CSI driver** (mounts as files, can sync to env). Alternatively the app fetches at startup with `aiobotocore` — simplest, but every pod calls the API on boot and rotation requires restart.
- **Rotation**: Secrets Manager rotation Lambda for RDS credentials; ESO picks up the new value on `refreshInterval`; a `Reloader` annotation restarts pods when the Secret changes. For DB, better: IAM database auth or RDS Proxy so there's no password to rotate.
- Audit via CloudTrail; encrypt with a customer-managed KMS key; deny access outside the VPC endpoint.

## Q: How would you handle configuration changes without rebuilding the image?

Config lives outside the image: ConfigMap/Secret changes → pod restart (`kubectl rollout restart` or Stakater Reloader on checksum change) picks them up. For live changes without restart: mount ConfigMap as a file and watch it (`watchfiles`), or use a **feature-flag / dynamic-config service** (LaunchDarkly, AWS AppConfig, Consul) polled by the app. Keep secrets and config validated on load so a bad change fails readiness rather than crashing silently.

---

## Q: How would you secure the CI/CD pipeline?

The pipeline is a high-value target (it has prod credentials).
- **Least-privilege runners**: ephemeral, isolated per job; no shared Docker socket (use Kaniko/BuildKit rootless).
- **Short-lived credentials**: OIDC federation from GitLab/GitHub to AWS (`AssumeRoleWithWebIdentity`) — no long-lived keys in CI variables.
- **Protected branches & environments**: prod deploy only from `main`, with required approvals; masked and protected variables.
- **Pin actions/images by digest**; scan the pipeline's own dependencies.
- **Signed commits/tags**; **signed images** (cosign) verified by cluster admission policy; provenance attestations (SLSA).
- **Separate CI from CD**: CI can push images, only ArgoCD (in-cluster) can deploy; CI never holds kubeconfig for prod.
- **Secret scanning** on every push; **audit logs** for pipeline changes; branch protection on `.gitlab-ci.yml` itself.
- **No arbitrary code from forks** with secrets access.

## Q: How do you scan Python dependencies?

- `pip-audit` (PyPA, uses OSV/PyPI advisory DB) — `pip-audit --strict -r requirements.txt` or against the lockfile.
- `safety`, Snyk, GitHub Dependabot / GitLab Dependency Scanning, Renovate for automated bump PRs.
- Generate an SBOM (`cyclonedx-py`) and monitor it continuously — new CVEs appear after you shipped.
- Pin with a lockfile and hashes (`--require-hashes`) to prevent substitution attacks; use a private proxy (Artifactory/Nexus) to mirror PyPI.

## Q: How do you scan Docker images?

Trivy / Grype / Snyk Container / ECR native scanning: scan OS packages, language deps, and misconfigurations; fail the build on HIGH/CRITICAL with a fix available; ignore-file with expiry dates for accepted risks; rescan images already in the registry on a schedule (ECR enhanced scanning, Trivy Operator in-cluster) because CVEs are discovered later; rebuild base images weekly to pick up patches.

## Q: How do you prevent vulnerable dependencies from reaching production?

Layered:
1. **Pre-merge**: dependency scan on every PR; Renovate auto-PRs for updates; lockfiles.
2. **Build**: image scan gate; SBOM attached; signature.
3. **Registry**: continuous rescanning; policy that blocks pulls of images with new critical CVEs.
4. **Cluster admission**: Kyverno/OPA Gatekeeper policies — only signed images from our registry, no `latest`, must have SBOM attestation, block images with critical CVEs (via policy-controller with vulnerability attestations).
5. **Runtime**: Trivy Operator reports on running workloads; alert on new findings; scheduled base-image rebuilds so fixes flow without code changes.
6. **Process**: SLA for patching by severity; exceptions documented with expiry.

---

# 16. System Design Questions

> **How to answer any design question (the framework):**
> 1. **Clarify requirements** — functional (what it does), non-functional (scale, latency, availability, consistency, security, budget). Ask 2–3 sharp questions, then state assumptions.
> 2. **Estimate** — rps, storage, bandwidth. Back-of-envelope numbers anchor every later choice.
> 3. **High-level architecture** — boxes and arrows: clients → edge → API → data/async.
> 4. **Deep-dive the critical path** — data model, API contract, the hard problem (consistency, idempotency, hot keys).
> 5. **Cross-cutting** — auth, observability, deployment, failure modes.
> 6. **Trade-offs & evolution** — what you'd do at 10× scale, what you'd cut for v1.
>
> Each answer below is in that shape, condensed to what you'd say in 10–15 minutes.

---

## Design a scalable FastAPI microservice architecture.

**Requirements/assumptions:** several bounded contexts (users, orders, payments, notifications), independent teams, tens of thousands of rps aggregate, 99.9%+ availability.

**Architecture:**
```
Clients → CDN/WAF → API Gateway (auth, rate limit, routing)
        → Services (FastAPI, each: own DB, own deploy, own HPA)
        → Async backbone (Kafka/SQS) for events between services
        → Shared platform: IdP (Cognito/Keycloak), Redis, S3, OTel Collector, Prometheus/Grafana/Loki/Tempo
```
- **Service template**: the structure from Section 9 + shared internal library (auth verification, logging, tracing, health, error envelope) so every service behaves the same.
- **Sync vs async**: synchronous REST (via gateway or service mesh) only for request-path reads that need immediate answers; everything else is events (order.placed → payment → notification). Outbox pattern for reliable publishing.
- **Data**: database per service; no shared tables; read models built from events for cross-service queries; sagas (choreography for simple flows, orchestration via Temporal for complex ones) instead of distributed transactions.
- **Resilience**: timeouts, retries with jitter, circuit breakers on every client; bulkheads; graceful degradation.
- **Security**: gateway validates JWT; mTLS between services; per-service IAM roles; scopes per caller.
- **Deployment**: EKS, one Helm chart per service, ArgoCD, canary rollouts, PDBs, multi-AZ.
- **Observability**: trace context propagated through HTTP and messages; RED dashboards per service; SLOs with burn-rate alerts.
- **Evolution**: start as a modular monolith if the team is small; extract along stable boundaries.

---

## Design a FastAPI API handling 100K requests/minute.

**Numbers:** 100k/min ≈ 1,700 rps average; plan for 3× peak ≈ 5k rps. If 80% reads and cacheable, the DB sees ~1k rps.

**Design:**
- **Edge**: CloudFront + WAF, TLS termination, cache public GETs (`Cache-Control`, ETag).
- **Ingress/LB**: ALB → NGINX ingress or Gateway API; keep-alive to pods.
- **App**: FastAPI, async handlers, `orjson`, 1 worker/pod, pods sized ~1 CPU; each pod ≈ 500–1,000 rps for light endpoints → 6–10 pods baseline, HPA to 30 on rps metric.
- **Cache**: Redis cluster; cache-aside for hot reads with stampede protection; per-user rate limits.
- **DB**: Postgres (RDS Multi-AZ) + 2 read replicas; PgBouncer/RDS Proxy; indexes verified with `pg_stat_statements`; cursor pagination.
- **Writes**: validate → persist → enqueue side effects (SQS) → 202/201 fast.
- **Protection**: rate limiting at gateway and app; concurrency limit per pod; load shedding at 90% saturation.
- **Verify**: k6 to 5k rps in staging with p99 < 300 ms; chaos test one AZ down.

---

## Design an API gateway using FastAPI.

**Caveat first:** for a production gateway I'd use Kong/Envoy/APISIX/AWS API Gateway — they're optimized C/Go proxies with mature plugins. A FastAPI gateway makes sense as a **BFF (Backend-for-Frontend)** or when custom Python logic dominates (aggregation, per-tenant routing, response shaping).

**Design:**
- **Routing table** from config: prefix → upstream base URL, with per-route auth/rate-limit policies.
- **Auth**: verify JWT once (JWKS), forward `X-User-Id`/`X-Roles`/`X-Tenant-Id`; strip incoming spoofed copies.
- **Proxying**: shared `httpx.AsyncClient` with pool limits; stream request/response bodies (`request.stream()` → `StreamingResponse(upstream.aiter_bytes())`); copy safe headers; add `X-Forwarded-*`; propagate `traceparent`.
- **Resilience**: per-upstream timeouts, retries for idempotent methods, circuit breaker (`aiobreaker`/custom), bulkhead semaphores per upstream.
- **Rate limiting**: Redis sliding window per key (API key/user/IP).
- **Aggregation endpoints**: `gather` calls to several services, compose one response.
- **Observability**: metrics per upstream, access logs, tracing.
- **Ops**: stateless, many small pods; config hot-reload; response caching for GETs with `Vary` awareness.

```python
@app.api_route("/{path:path}", methods=["GET","POST","PUT","PATCH","DELETE"])
async def proxy(path: str, request: Request, user: CurrentUser, http: HttpDep):
    upstream = resolve_upstream(path)                                  # routing table
    if upstream is None: raise HTTPException(404)
    headers = forward_headers(request.headers) | {"X-User-Id": str(user.id)}
    async with breaker(upstream.name):
        resp = await http.send(http.build_request(request.method, f"{upstream.url}/{path}", headers=headers,
                                                  content=request.stream(), params=request.query_params),
                               stream=True)
    return StreamingResponse(resp.aiter_raw(), status_code=resp.status_code,
                             headers=filter_response_headers(resp.headers), background=BackgroundTask(resp.aclose))
```

---

## Design a multi-tenant FastAPI application.

**Key decision — data isolation model:**

| Model | Isolation | Cost | Ops | When |
|---|---|---|---|---|
| Shared DB, shared schema + `tenant_id` column | Lowest | Lowest | Easiest | Many small tenants (SaaS default) |
| Shared DB, schema per tenant | Medium | Medium | Migrations × N | Hundreds of tenants, moderate isolation needs |
| Database per tenant | Highest | Highest | Hardest | Regulated/enterprise tenants |

Pick shared-schema with **Postgres Row-Level Security** as the default, and support "premium isolated" tenants via DB-per-tenant routing.

**Design:**
- **Tenant resolution**: from JWT claim (`tenant_id`), subdomain, or header — resolved once in a dependency; never from the body.
- **Enforcement**: every table has `tenant_id`; RLS policy `USING (tenant_id = current_setting('app.tenant_id')::uuid)`; the session dependency runs `SET LOCAL app.tenant_id = ...` at transaction start so even a forgotten `WHERE` can't leak.
- **Repositories** take tenant from context, never as a caller-supplied argument.
- **Per-tenant limits**: rate limits, quotas, feature flags, plan entitlements.
- **Noisy neighbours**: fair-share rate limiting, per-tenant queue partitions, optional dedicated pods for big tenants.
- **Cache keys** prefixed with tenant; S3 prefixes per tenant; KMS key per tenant for the isolated tier.
- **Onboarding/offboarding**: tenant provisioning API; data export and deletion workflows (GDPR).
- **Testing**: a test that user of tenant A requesting tenant B's resource gets 404, for every endpoint.

```python
async def get_tenant_db(tenant: CurrentTenant) -> AsyncIterator[AsyncSession]:
    async with SessionLocal() as s:
        async with s.begin():
            await s.execute(text("SET LOCAL app.tenant_id = :t"), {"t": str(tenant.id)})   # RLS scope
            yield s
```

---

## Design a FastAPI authentication service.

**Scope:** signup/login, tokens, refresh, logout, password reset, MFA, social login, used by many services.

**Design:**
- **Prefer buy over build**: Cognito/Auth0/Keycloak as the IdP; this service becomes a thin façade (user profile, org membership). If building:
- **Endpoints**: `POST /signup`, `POST /token` (password + PKCE code flows), `POST /refresh`, `POST /logout`, `POST /password/forgot|reset`, `POST /mfa/enroll|verify`, `GET /.well-known/jwks.json`, `GET /userinfo`.
- **Tokens**: RS256/ES256 access tokens (15 min) signed with a private key in KMS; `kid` rotation; refresh tokens hashed in DB, rotated on use, family-revocation on reuse.
- **Storage**: Postgres (users, credentials, refresh tokens, MFA secrets encrypted), Redis (login attempt counters, denylist, one-time codes).
- **Security**: Argon2id hashing; constant-time compares; rate limit and lockout with exponential delay per account/IP; breached-password check (k-anonymity API); email enumeration resistance; CSRF for cookie flows; audit log of auth events; device/session listing and revocation.
- **Consumers** verify tokens locally via JWKS — the auth service is *not* on the request path of other services (availability!).
- **Availability**: multi-AZ; JWKS cached by consumers for hours so an auth outage only blocks new logins.
- **Observability**: login success/fail rates, MFA challenge rates, token issuance, anomaly alerts.

---

## Design an asynchronous document-processing API.

**Flow:** upload → validate → process (OCR/parse/ML) → notify → retrieve results. Processing takes seconds to minutes.

**Design:**
1. `POST /documents` → returns pre-signed S3 PUT URL + `document_id` (API never streams file bytes).
2. Client uploads to S3; S3 event (or `POST /documents/{id}/complete`) → API validates (size/type via HEAD, magic bytes on first bytes), writes `status=queued`, publishes `{document_id}` to SQS (outbox).
3. **Workers** (separate deployment, KEDA-scaled on queue depth; GPU pool for ML stages): download, process in a pipeline of stages, each idempotent and checkpointed (`status`, `stage`, `progress`), results to S3 + metadata to Postgres; long steps heartbeat to extend visibility timeout.
4. `GET /documents/{id}` → status/progress/result URL; `GET /documents/{id}/result` → pre-signed GET. Webhook or SSE for push.
5. **Failure handling**: retries with backoff; DLQ with replay; poison detection; per-stage timeouts; partial results preserved.
6. **Controls**: per-tenant quotas, virus scanning stage, PII redaction, retention/TTL deletion, encryption (SSE-KMS).
7. **Scale**: fan-out large docs into page-level tasks with a fan-in aggregator; priority queues.
8. **Observability**: trace from upload through every stage; queue lag and stage duration histograms; stuck-job detector.

---

## Design a FastAPI service using PostgreSQL + Redis + SQS.

**Roles:**
- **Postgres**: system of record; transactions; constraints.
- **Redis**: cache (cache-aside, TTL + explicit invalidation), rate limits, distributed locks, idempotency keys (short TTL), pub/sub for cache invalidation across pods.
- **SQS**: durable async work and cross-service events; FIFO queues where ordering per key matters; DLQs.

**Request path (write):**
```
POST /orders (Idempotency-Key)
 → check Redis idempotency key (fast path)
 → BEGIN; insert order; insert outbox row; COMMIT          (atomic)
 → return 201
Outbox relay (background loop or Debezium) → SQS publish → mark sent
Worker consumes → idempotent processing → delete message
```
**Read path:** Redis → miss → Postgres (replica) → fill cache.

**Details:** pool sizes tuned per pod; `pool_pre_ping`; Redis client with timeouts and fallback-to-DB on Redis failure (never make Redis a hard dependency for reads); SQS long-polling, batch sends, visibility timeout > p99 processing; consumer scaling on queue depth; every message carries trace context and a version field; health checks: readiness = DB; Redis/SQS degrade gracefully.

---

## Design a distributed FastAPI system with OpenTelemetry.

**Goal:** one trace per user request across gateway → services → DB → queues → workers, with logs and metrics correlated.

**Design:**
- **SDK in every service** via shared library: `TracerProvider` with resource attrs (`service.name`, `service.version`, `deployment.environment`), OTLP exporter → **OTel Collector** (DaemonSet/sidecar); auto-instrument FastAPI, httpx, SQLAlchemy, Redis, botocore; `LoggingInstrumentor` to inject `trace_id`/`span_id` into JSON logs.
- **Propagation**: W3C `traceparent`/`baggage` over HTTP; message attributes over SQS/Kafka; consumer spans with `links` to producer spans.
- **Collector pipeline**: receivers (OTLP) → processors (batch, memory limiter, k8s attributes, attribute redaction for PII, **tail-based sampling**: keep 100% of errors and slow traces, 5% of the rest) → exporters (Tempo/Jaeger for traces, Prometheus remote write or Mimir for metrics, Loki for logs).
- **Metrics**: OTel metrics or Prometheus client — pick one; exemplars link histogram buckets to trace IDs.
- **Dashboards**: Grafana with data links from metric → trace → logs by ID; service graph from span metrics.
- **SLOs and alerts** from span-derived RED metrics.
- **Cost control**: sampling, attribute limits, drop health-check spans.
- **Governance**: naming conventions for spans/attributes (semantic conventions); trace ID returned to clients as `X-Request-ID` for support.

---

## Design an API that must maintain 99.99% availability.

**Budget:** 99.99% = 52 minutes downtime/year ≈ 4.3 min/month. Every dependency and deploy eats into that; a single-region single-DB design can't reliably hit it.

**Design:**
- **Redundancy at every layer**: ≥3 pods across 3 AZs with PDB and anti-affinity; multi-AZ managed DB with automated failover (<30 s, Aurora); Redis cluster with replicas; multi-AZ NAT/LB.
- **Multi-region active-passive** (or active-active for reads) with Route 53 health-checked failover and Aurora Global Database / cross-region replication; regularly exercised failover runbook.
- **Change safety** (most outages are self-inflicted): canary deploys with automatic rollback, feature flags, expand/contract migrations, no big-bang releases, deploy freezes around peak.
- **Dependency isolation**: timeouts, circuit breakers, bulkheads; degrade rather than fail (serve stale cache, disable non-critical features); no synchronous dependency on non-critical services.
- **Capacity**: headroom ≥ 50% over peak; autoscaling; load shedding at saturation; rate limiting to protect from abuse.
- **Data safety**: PITR backups tested by restore drills; idempotent writes for safe client retries.
- **Observability & response**: SLO burn-rate paging, on-call with runbooks, automated remediation for known failures, game days/chaos engineering.
- **Client side**: retries with jitter for idempotent calls, connection pooling, sensible timeouts.
- **Reality check**: 99.99% requires the whole org — you also need error budgets and blameless postmortems.

---

## Design an API with idempotent payment/order creation.

Section 11's idempotency-key implementation is the core. System-level additions:
- **Client contract**: `Idempotency-Key` required on `POST /payments`; key = UUID per *intent*, reused on retries; documented behaviour: 201 first time, same 201 on replay, 409 while in progress, 422 if body differs.
- **Storage**: key scoped by `(merchant_id, key)`; store request hash, status, response, `expires_at` (24 h); PK constraint is the atomic claim.
- **Two-phase safety with the PSP** (Stripe etc.): pass the same key as the PSP's idempotency key; persist the PSP's intent ID **before** confirming; on crash between steps, reconciliation job queries the PSP by our key.
- **State machine**: `created → authorized → captured | failed`; transitions idempotent; version column for optimistic concurrency.
- **Exactly-once effects**: unique constraint `UNIQUE(order_id)` on payments; ledger entries append-only.
- **Async completion**: webhook from PSP processed idempotently (dedupe by event ID); outbox for our own events.
- **Timeouts**: if the PSP call times out, do *not* retry blindly — check status by key first.
- **Observability**: metrics for replays, mismatches, in-progress collisions; alerts on reconciliation discrepancies.

---

## Design an API supporting millions of users.

**Numbers:** e.g., 5M users, 10% DAU = 500k, 20 requests/user/day ≈ 115 rps average, peaks 10× ≈ 1–2k rps — *modest* compute; the challenges are data volume, hot spots, and operations.

**Design:**
- Stateless FastAPI fleet with HPA; CDN for static/public; gateway auth.
- **Data**: Postgres with partitioning (by time or hash on user_id) for large tables; read replicas; PgBouncer; consider sharding by user_id when a single primary's write throughput is the limit (Citus/Vitess-style or app-level routing).
- **Caching**: user profile/session in Redis; hot entities; CDN for media.
- **Async**: notifications, emails, analytics via queues; never in the request path.
- **Search/analytics** offloaded to OpenSearch/ClickHouse fed by CDC (Debezium) so OLTP stays lean.
- **Hot users/keys**: rate limits, request coalescing, per-key sharded counters.
- **Auth at scale**: stateless JWT verification; JWKS cached.
- **Operational**: schema migrations online only; backfills batched; capacity planning from growth metrics; cost monitoring.
- **Compliance**: data residency, deletion workflows, audit.

---

## Design a file-upload API for multi-GB files.

**Never stream multi-GB through the API pods.** Use S3 multipart with pre-signed URLs; the API is the control plane.

**Flow:**
1. `POST /uploads` `{filename, size, content_type, checksum}` → validate (size limits, type, quota) → `CreateMultipartUpload` → return `upload_id`, part size (e.g., 64 MB), and pre-signed URLs for the first N parts (or an endpoint to fetch URLs per part).
2. Client uploads parts in parallel directly to S3; retries individual parts; resumable (`GET /uploads/{id}` lists completed parts via `ListParts`).
3. `POST /uploads/{id}/complete` with part ETags → `CompleteMultipartUpload` → verify checksum (S3 checksum algorithms / MD5 of parts) → record file metadata → enqueue post-processing (virus scan, thumbnail).
4. `DELETE /uploads/{id}` → `AbortMultipartUpload`; lifecycle rule aborts incomplete uploads after 24 h.

**Details:** pre-signed URLs short-lived (15 min), scoped to key and content-length range; S3 Transfer Acceleration for global users; SSE-KMS; bucket policy denies public; per-tenant prefixes; event notification on completion; quotas and rate limits on URL generation; progress via `ListParts`; browser clients use the same flow via SDK.

---

## Design an API for long-running ML jobs.

**Requirements:** jobs run minutes to hours on GPUs; users need status, progress, cancellation, results; fairness across tenants.

**Design:**
- **API (FastAPI, CPU pods)**: `POST /jobs` (validate input, store spec, upload inputs to S3, enqueue) → 202 + `job_id`; `GET /jobs/{id}` status/progress/logs tail/result URLs; `DELETE /jobs/{id}` cancel; `GET /jobs` list with filters; webhooks/SSE for completion.
- **Queue**: SQS/Kafka/Redis Streams with **priority tiers** and per-tenant fair-share; or a workflow engine (Temporal/Argo Workflows) for multi-step DAGs with retries and checkpoints.
- **Workers**: separate GPU node pool; KEDA scales on queue depth; each worker: claim job atomically, heartbeat, checkpoint to S3 every N minutes so preemption/spot interruption resumes rather than restarts, stream logs to S3/Loki, write progress to Redis (fast) and final state to Postgres.
- **Cancellation**: cooperative flag checked between steps; SIGTERM handling with checkpoint.
- **Idempotency**: job ID as dedupe key; outputs keyed by job ID; re-run safe.
- **Resource control**: per-tenant concurrency limits, quotas, cost tracking per job; timeouts per job type.
- **Model management**: versioned models in a registry; job records the model version for reproducibility.
- **Observability**: queue wait time vs run time, GPU utilization, failure taxonomy, stuck-job detector, DLQ.

---

## Design an API with rate limiting.

**Requirements:** per-user and per-API-key limits, plan tiers (free 60 rpm, pro 1,000 rpm), burst tolerance, global protection, clear client feedback.

**Design:**
- **Two layers**: edge (WAF/API Gateway) for coarse IP-based volumetric limits; app for identity-aware business limits.
- **Algorithm**: token bucket in Redis (Lua for atomicity): capacity = burst, refill = rate; keys `rl:{tier}:{principal}:{scope}`; multiple scopes (global per user, per endpoint group for expensive ops).
- **Response**: `429` with `Retry-After`; always send `X-RateLimit-Limit/Remaining/Reset` (or IETF `RateLimit-*`).
- **Resilience**: if Redis is down, fail open with local in-memory fallback + alert (don't take the API down because the limiter is down), unless the endpoint is sensitive (login → fail closed).
- **Fairness & abuse**: per-tenant, per-IP, and per-token limits together; anomaly detection; allow-lists for internal callers.
- **Config**: limits from a config service/DB, hot-reloadable; per-tenant overrides.
- **Observability**: 429 rate per tenant, limiter latency, Redis health; dashboards to spot clients near their limits before support tickets.

---

## Design an API supporting API versioning.

- **Strategy**: URI major versions (`/v1`, `/v2`); additive changes within a version; breaking changes only in a new major.
- **Code**: per-version router packages that adapt shared domain services (Section 3/9); version-specific schemas; shared validation in the domain layer.
- **Lifecycle**: each version has `active → deprecated (Sunset header + docs + emails) → retired`; minimum 6–12 months notice; per-version usage metrics drive retirement.
- **Docs**: separate OpenAPI per version; changelog; migration guides.
- **Gateway**: route by prefix; can map `/v1` to an older deployment during the transition if needed.
- **Testing**: contract tests per version against snapshots; consumer-driven contracts for internal clients.
- **Edge cases**: version-aware events/webhooks (payload version field); SDKs pinned to versions; feature flags for behaviour changes that aren't contract changes.

---

## Design a zero-downtime FastAPI deployment.

Compose Sections 14/15:
- Rolling update with `maxUnavailable: 0`; readiness/startup probes; `preStop` sleep + graceful shutdown; PDB; ≥3 replicas across AZs.
- Expand/contract migrations executed as a PreSync Job; app compatible with N and N+1 schema.
- Backward-compatible API contracts so mixed versions coexist; feature flags for behaviour.
- Progressive delivery (canary) with automated analysis and rollback.
- Connection handling: LB with connection draining; HTTP keep-alive aware; client retries for idempotent requests.
- Long-lived connections (WebSockets/SSE): announce shutdown, clients reconnect with backoff.
- Background workers: stop consuming on SIGTERM, finish in-flight, then exit; visibility timeouts cover the rest.
- Verify with a load test running *during* a deploy: zero non-2xx.

---

## Design a highly available FastAPI service across multiple AZs.

- **Compute**: EKS node groups in ≥3 AZs; `topologySpreadConstraints` on zone; pod anti-affinity; PDB `minAvailable`; Cluster Autoscaler/Karpenter per AZ.
- **Ingress**: ALB (multi-AZ by default) → NGINX ingress pods spread across AZs; NLB for TCP.
- **Data**: RDS/Aurora Multi-AZ (synchronous standby, automatic failover); Aurora readers in each AZ; ElastiCache Redis with Multi-AZ auto-failover; S3 (regional, multi-AZ inherent); SQS (regional).
- **Zone-aware routing**: topology-aware hints so traffic prefers same-AZ pods and replicas (cost + latency), with cross-AZ fallback.
- **Stateless app**: no local state; sessions in Redis/JWT; uploads to S3.
- **Failure handling**: pool `pre_ping` + reconnect on failover; DNS TTL low; readiness reflects dependency state; retries with jitter.
- **Testing**: AZ evacuation game day (cordon/drain a zone, kill the DB primary); verify SLOs hold.
- **Beyond AZ**: for region-level HA add a second region with Route 53 failover and cross-region replication.

---

## Design FastAPI services that communicate through Kafka/SQS.

- **When Kafka vs SQS**: Kafka for high-throughput ordered streams, replay, multiple consumer groups, event sourcing/CDC; SQS for simple work queues with minimal ops (FIFO for ordering per group).
- **Producer side**: **transactional outbox** — write the event in the same DB transaction as the state change; a relay (or Debezium CDC) publishes to Kafka/SQS; marks sent. Guarantees no lost events, at-least-once delivery.
- **Event design**: versioned schemas (Avro/Protobuf/JSON Schema in a registry), `event_id`, `occurred_at`, `trace context`, aggregate ID as key (Kafka partition key for ordering).
- **Consumer side** (separate deployment from the API, e.g., `aiokafka` / `aiobotocore`): idempotent handlers (dedupe by `event_id` in a processed-events table or unique constraint), manual commits after processing, backoff and DLQ topic/queue, consumer-group scaling ≤ partitions, KEDA on lag/depth.
- **Contracts**: consumer-driven; additive schema evolution; compatibility checks in CI.
- **Observability**: producer/consumer lag, processing latency, DLQ depth, trace links across the async hop.
- **Sagas**: for multi-service workflows, choreography via events with compensating actions; orchestration (Temporal) when the flow gets complex.

---

## Design a FastAPI service capable of surviving downstream service failures.

**Assume every dependency will fail; design for degraded, not dead.**
- **Timeouts** on every call (connect + read), shorter than your own SLO budget; deadlines propagated downstream.
- **Retries** only for idempotent operations, bounded, exponential backoff with jitter, retry budgets (≤10% extra load).
- **Circuit breaker** per dependency: open after N failures, half-open probes, fast-fail while open — protects you *and* them.
- **Bulkheads**: semaphore per dependency so one slow service can't consume all concurrency.
- **Fallbacks**: serve stale cache (`stale-if-error`), default values, partial responses with a `degraded: true` flag, queue the write for later (if semantics allow).
- **Feature flags** to disable non-critical integrations instantly.
- **Async decoupling**: anything not needed for the response goes through a queue.
- **Readiness** should reflect *required* dependencies only; optional ones degrade.
- **Load shedding** so cascading slowness doesn't exhaust your pods.
- **Observability**: breaker state metrics, fallback counters, dependency SLIs; alerts when degraded mode persists.
- **Testing**: fault injection (toxiproxy, chaos mesh) in staging; contract tests for fallback paths.

```python
class Breaker:
    def __init__(self, failures=5, reset_after=30):
        self.failures, self.reset_after = failures, reset_after
        self.count, self.opened_at = 0, None
    def allow(self) -> bool:
        if self.opened_at and time.monotonic() - self.opened_at < self.reset_after: return False
        return True
    def record(self, ok: bool):
        if ok: self.count, self.opened_at = 0, None
        else:
            self.count += 1
            if self.count >= self.failures: self.opened_at = time.monotonic()

async def get_recommendations(user_id: int, http, redis) -> list[dict]:
    if not reco_breaker.allow():
        return await stale_or_default(redis, user_id)                       # fast fallback
    try:
        async with reco_sem:                                                # bulkhead
            r = await http.get(f"{RECO_URL}/users/{user_id}", timeout=0.3)
        r.raise_for_status()
        reco_breaker.record(True)
        await redis.set(f"reco:{user_id}", r.text, ex=3600)                 # refresh stale copy
        return r.json()
    except (httpx.HTTPError, asyncio.TimeoutError):
        reco_breaker.record(False)
        return await stale_or_default(redis, user_id)
```

---

## Design a production-grade FastAPI platform on AWS EKS.

**Account & network:** separate AWS accounts per environment (org SCPs); VPC with private subnets for nodes/DB across 3 AZs, public only for ALB/NAT; VPC endpoints for S3/ECR/Secrets Manager/SQS (no NAT cost, no internet path).

**Cluster:** EKS with managed node groups or Karpenter (spot for workers, on-demand for API), Bottlerocket AMIs, IRSA/Pod Identity, Pod Security Standards (restricted), NetworkPolicies (Cilium/Calico), cluster autoscaling, EKS add-ons (CoreDNS, kube-proxy, VPC CNI with prefix delegation), `ndots` tuning.

**Ingress & edge:** Route 53 → CloudFront + WAF (managed rules, rate rules) → ALB (AWS Load Balancer Controller, ACM TLS) → NGINX ingress or Gateway API; optional service mesh (Istio ambient / App Mesh) for mTLS and retries.

**Platform services:** ArgoCD (GitOps, ApplicationSets per env), External Secrets Operator ↔ Secrets Manager, cert-manager, Reloader, KEDA, Prometheus stack (kube-prometheus or AMP), Grafana, Loki/CloudWatch, Tempo/X-Ray via OTel Collector (ADOT), Trivy Operator, Kyverno policies (signed images, no root, resource limits required), Velero backups.

**Data:** Aurora PostgreSQL Multi-AZ + RDS Proxy; ElastiCache Redis cluster mode; S3 with lifecycle/KMS; SQS/SNS/MSK as needed; parameter groups tuned; PITR + tested restores.

**Service standard (golden path):** cookiecutter template with the Section 9 structure, shared library, Dockerfile, Helm chart (Deployment/HPA/PDB/ServiceAccount/ServiceMonitor/Ingress), GitLab CI template (Section 15), dashboards and alert rules generated per service, SLO definitions.

**Security:** least-privilege IAM per service, no static keys, secrets rotated, image signing + admission verification, network segmentation, audit logs (CloudTrail, EKS control-plane logs), GuardDuty, dependency/image scanning continuous.

**Operations:** on-call rotation, runbooks in repo, error budgets, capacity reviews, cost allocation tags + Kubecost, quarterly game days (AZ loss, DB failover, deploy rollback), DR plan with RTO/RPO and cross-region backups.

**Interview close:** "The platform's job is to make the right thing the easy thing: a new FastAPI service should go from template to production with observability, security, and autoscaling included, in a day — without the team having to become Kubernetes experts."

---

*End of document.*

# 01 — FastAPI Fundamentals

---

## 📖 Vocabulary (read this first)

> **How to use this table.** *Definition* is the precise, technically correct meaning. *Say it in an interview* is a one-sentence version you can quote out loud. Learn the definition; rehearse the sentence.

| Term | Definition | Say it in an interview |
|---|---|---|
| **API** | *Application Programming Interface.* A defined set of operations, inputs and outputs through which one program exposes functionality to another. A **web API** exposes those operations over HTTP, usually exchanging JSON. | "An API is a contract that lets programs talk to each other; a web API is that contract exposed over HTTP." |
| **HTTP** | *HyperText Transfer Protocol.* A stateless, text-based request/response protocol. A request has a method, a target path, headers and an optional body; a response has a status code, headers and an optional body. HTTP/1.1 uses one request at a time per connection; HTTP/2 multiplexes many. | "HTTP is the stateless request/response protocol the web runs on; every request has a method, path, headers and optional body." |
| **REST** | *Representational State Transfer.* An architectural style where resources are identified by URLs, manipulated through standard HTTP methods, and represented in formats like JSON; interactions are stateless. | "REST models your system as resources at URLs, manipulated with standard HTTP verbs, without server-side session state." |
| **JSON** | *JavaScript Object Notation.* A lightweight, language-independent text format for structured data: objects (`{}`), arrays (`[]`), strings, numbers, booleans and `null`. | "JSON is the standard text format for exchanging structured data between services." |
| **Framework** | A reusable software skeleton that dictates application structure and calls your code at defined points (inversion of control), as opposed to a library that your code calls. | "A framework calls your code; a library is code you call. FastAPI is a framework." |
| **Endpoint** | A single addressable operation of an API: the combination of an HTTP method and a URL path (e.g., `GET /users/{id}`). | "An endpoint is one method-plus-path that the API responds to." |
| **Route / Routing** | A *route* is a registered mapping from a URL pattern (and method) to a handler; *routing* is the process of matching an incoming request to the correct route. | "Routing is how the framework matches an incoming URL and method to the function that should handle it." |
| **Handler** (path operation function) | The function executed when a route matches. FastAPI calls it a *path operation function*; Django calls it a *view*. | "The handler is the function that actually runs for a matched route." |
| **Decorator** | A callable that takes a function and returns a replacement or registers it, applied with `@name` above a `def`. `@app.get("/x")` registers the decorated function as a route. | "A decorator wraps or registers a function; in FastAPI, route decorators register handlers." |
| **Type hint** (annotation) | Optional static type information attached to variables, parameters and return values (PEP 484). Ignored by the interpreter at runtime but readable via introspection, which FastAPI and Pydantic exploit. | "Type hints are annotations Python ignores at runtime but tools and FastAPI read to validate and document." |
| **Serialization / Deserialization** | Serialization converts an in-memory object into a transmissible representation (e.g., dict → JSON bytes). Deserialization is the reverse. | "Serialization turns objects into bytes for the wire; deserialization turns them back." |
| **Validation** | Verifying that data conforms to a specification (types, required fields, ranges, formats) before it is used, rejecting it with structured errors otherwise. | "Validation is the gate that guarantees data has the right shape and values before business logic sees it." |
| **Schema** | A formal, machine-readable description of a data structure: field names, types, constraints and which fields are required. JSON Schema is the standard for JSON. | "A schema is the formal description of what valid data looks like." |
| **Synchronous** | Execution model where each operation must complete before the next begins; a slow I/O call blocks the calling thread. | "Synchronous code does one thing at a time and waits." |
| **Asynchronous** | Execution model where an operation can be started, the caller suspended, and other work performed until the operation completes; in Python via `async`/`await` and an event loop. | "Async lets one thread make progress on other work while waiting on I/O." |
| **Concurrency** | Multiple tasks making progress within overlapping time periods, not necessarily executing simultaneously. Distinct from *parallelism* (simultaneous execution on multiple cores). | "Concurrency is dealing with many things at once; parallelism is doing many things at once." |
| **Event loop** | The scheduler at the core of asyncio: it runs ready coroutines, registers interest in I/O with the OS, and resumes coroutines when their awaited I/O completes. One per thread. | "The event loop is the single-threaded scheduler that runs coroutines and wakes them when their I/O is ready." |
| **Server** (ASGI/web server) | The process that binds a socket, parses HTTP, and invokes the application via a gateway interface. Uvicorn is an ASGI server; Gunicorn is a WSGI server/process manager. | "The server speaks HTTP on the socket and hands parsed requests to the app; FastAPI needs Uvicorn to serve it." |
| **Interface / Contract** | An agreed specification of how two components communicate (signatures, message formats, guarantees), allowing either side to be replaced. ASGI is a server–application contract. | "An interface is the agreed contract that lets you swap implementations on either side." |
| **Middleware** | A component positioned between the server and the application (or between layers) that can inspect or modify every request and response. In ASGI each middleware wraps the next app. | "Middleware is code that wraps every request and response, like layers of an onion around the app." |
| **OpenAPI** | A vendor-neutral specification (JSON/YAML) for describing HTTP APIs: paths, operations, parameters, schemas and security. Formerly the Swagger Specification. | "OpenAPI is the standard machine-readable description of an API that tools use to generate docs and clients." |
| **Boilerplate** | Repetitive code required by a framework or language that carries little application-specific meaning. | "Boilerplate is repetitive setup code that adds no business value; FastAPI removes most of it." |
| **Status code** | A three-digit integer in an HTTP response indicating the outcome: 1xx informational, 2xx success, 3xx redirection, 4xx client error, 5xx server error. | "Status codes tell the client how the request went: 2xx good, 4xx their fault, 5xx our fault." |
| **Idempotent** | An operation whose repeated execution produces the same resulting state as a single execution. GET, PUT and DELETE are idempotent by specification; POST is not. | "Idempotent means doing it twice has the same effect as doing it once, which is what makes retries safe." |
| **Safe method** | An HTTP method that is read-only and must not change server state (GET, HEAD, OPTIONS). | "A safe method reads without side effects, so caches and crawlers can call it freely." |
| **Pydantic** | A Python data-validation library where models are classes with typed fields; it parses, coerces and validates input and serializes output, with a Rust core in v2. | "Pydantic validates and serializes data using type-annotated model classes; FastAPI is built on it." |
| **Starlette** | A lightweight ASGI toolkit providing routing, middleware, requests/responses, WebSockets and a test client. FastAPI subclasses Starlette. | "Starlette is the ASGI web engine underneath FastAPI." |
| **Uvicorn** | A high-performance ASGI server built on `uvloop` and `httptools`, used to run FastAPI applications. | "Uvicorn is the ASGI server that actually listens on the port and runs FastAPI." |
| **ASGI** | *Asynchronous Server Gateway Interface.* The standard async contract between Python web servers and applications: `async app(scope, receive, send)`. Supports HTTP, WebSockets and lifespan. | "ASGI is the async successor to WSGI: a message-based contract between server and app supporting HTTP and WebSockets." |
| **WSGI** | *Web Server Gateway Interface* (PEP 3333). The synchronous contract `app(environ, start_response)` used by Flask and classic Django. HTTP only, blocking. | "WSGI is the original synchronous server–app contract; it can't do async or WebSockets." |
| **Swagger UI / ReDoc** | Two browser UIs that render an OpenAPI document. Swagger UI is interactive (try requests); ReDoc is a read-only reference layout. | "Swagger UI and ReDoc are the auto-generated docs pages FastAPI serves from the OpenAPI spec." |
| **Lifespan** | The ASGI protocol event that lets an application run code once at startup and once at shutdown (e.g., open/close connection pools). | "Lifespan is where you open shared resources at startup and close them at shutdown." |
| **Thread pool** | A fixed set of worker threads to which blocking functions are dispatched so the event loop stays free. Starlette uses one for sync handlers. | "Sync handlers run in a thread pool so they don't block the event loop." |

---

## Q1. What is FastAPI and why would you choose it over Flask or Django?

### 🧒 Simple explanation
FastAPI is a Python library for building web APIs. You write normal Python functions, add type hints to their parameters, and FastAPI turns them into HTTP endpoints. Because FastAPI reads the type hints, it can automatically check incoming data, convert it, and generate documentation — you don't write that code yourself.

**Analogy:** Think of a restaurant. Flask gives you an empty kitchen — you bring every tool. Django gives you a fully equipped restaurant with a dining room, menus and a cashier, even if you only wanted a food truck. FastAPI gives you a professional kitchen built for takeaway orders: exactly the tools for serving data quickly, nothing you don't need.

### 🧠 First principles
Every web framework must do four jobs for each request:

1. **Route** — figure out which function should handle `GET /users/42`.
2. **Parse** — turn the raw request (bytes) into Python values (`user_id = 42`).
3. **Validate** — make sure those values are acceptable (`42` is an integer, not `"abc"`).
4. **Respond** — turn the function's return value back into HTTP bytes.

Flask does step 1 for you and leaves 2–4 largely to you. Django does all four but is designed around HTML websites with a database, admin panel and templating. FastAPI's key idea: *Python already has type hints, so use them as the single description for steps 2, 3, 4, and for documentation.* One function signature does everything.

### 🎤 Interview answer
> "FastAPI is a modern, async-capable Python framework for building APIs. It's built on Starlette for the HTTP layer and Pydantic for data validation. I choose it over Flask when I want validation, serialization and OpenAPI docs built in instead of bolting on extensions. I choose it over Django when I'm building a pure API service and don't need Django's ORM, admin and templates. FastAPI also supports `async` natively, which matters for I/O-heavy services. The trade-off is that FastAPI is only the API layer, so the team must choose its own database tooling and enforce project structure."

### 📊 Comparison

| Concern | Flask | Django | FastAPI |
|---|---|---|---|
| Philosophy | Minimal, add what you need | Batteries included | Minimal but with validation & docs built in |
| Validation | Manual or extension | Forms / DRF serializers | Automatic from type hints (Pydantic) |
| Docs | Extension | Extension | Automatic (`/docs`) |
| Async support | Limited | Partial | Native |
| Learning curve | Low | High | Low–medium |
| Best fit | Small apps, prototypes | Content sites, admin-heavy apps | API services, microservices, ML serving |

### ⚠️ Mistakes & follow-ups
- Don't say "FastAPI is faster than Django" as the main reason — interviewers want *developer productivity and correctness* as the primary reasons; speed is a bonus.
- Follow-up: *"What does FastAPI NOT give you?"* → No ORM, no admin, no auth system out of the box, no opinionated project structure.

---

## Q2. What are the main features of FastAPI?

### 🧒 Simple explanation
1. **Type hints drive everything** — the parameter types you write become validation rules and documentation.
2. **Automatic interactive docs** — visit `/docs` and you get a web page listing every endpoint where you can try requests.
3. **Async support** — write `async def` and handle thousands of waiting connections efficiently.
4. **Dependency injection** — a clean way to share things like "the current logged-in user" or "a database connection" across endpoints.
5. **Pydantic models** — classes that describe your data and validate it.
6. **Standards-based** — uses OpenAPI and JSON Schema, so other tools understand your API.
7. **Great editor support** — because everything is typed, your IDE autocompletes and catches errors.
8. **Starlette underneath** — gives middleware, WebSockets, background tasks, a test client.
9. **Security helpers** — ready-made pieces for OAuth2, JWT bearer tokens, API keys.
10. **High performance** — among the fastest Python frameworks.

### 🎤 Interview answer
> "The feature I'd highlight is that the type hints *are* the contract: one function signature gives validation, serialization, documentation and editor support. On top of that: native async, a dependency injection system, and OpenAPI docs for free."

---

## Q3. How does FastAPI achieve high performance?

### 🧒 Simple explanation
Most of the time an API spends handling a request is *waiting* — for the database, for another service, for the disk. FastAPI is built so that while one request waits, the same process can work on other requests. Also, the slowest Python parts (parsing HTTP, validating data) are done by code written in faster languages (C and Rust) under the hood.

**Analogy:** A single barista who starts the espresso machine, then takes the next order while it brews, instead of standing there watching the cup fill.

### 🧠 First principles
Three layers contribute:

1. **ASGI + event loop** (via Starlette and Uvicorn): one process juggles many in-flight requests. When a request `await`s the database, the event loop switches to another request. No thread per request, so very little memory and no context-switch overhead.
2. **uvloop and httptools**: Uvicorn's event loop (`uvloop`) is written in C on top of `libuv` (the same library Node.js uses), and its HTTP parser (`httptools`) is also C. So the "plumbing" cost per request is tiny.
3. **Pydantic v2**: validation and serialization — normally the slowest pure-Python part — run in `pydantic-core`, which is written in Rust.

### 🎤 Interview answer
> "FastAPI itself is thin. The performance comes from Starlette's ASGI design on Uvicorn with uvloop, plus Pydantic v2 doing validation in Rust. For I/O-bound APIs, a single process can handle thousands of concurrent connections. For CPU-bound work it's still Python, so you scale with worker processes or move the work out of the request path."

### ⚠️ Mistakes & follow-ups
- "FastAPI is fast" does not mean *your code* is fast. A blocking call inside an `async def` will freeze everything (see file 05).
- Follow-up: *"Is FastAPI faster than Go or Node?"* → Comparable for I/O-bound workloads in benchmarks; slower for CPU-heavy work because Python is interpreted.

---

## Q4. What is ASGI?

### 🧒 Simple explanation
ASGI stands for **Asynchronous Server Gateway Interface**. It is a *rulebook* describing how a Python web **server** (like Uvicorn) should call a Python web **application** (like FastAPI). The server says: "here's a connection, here's how to receive data, here's how to send data" — and the app takes it from there.

**Analogy:** A universal power socket standard. Any appliance (framework) can plug into any socket (server) because they agree on the shape of the plug.

### 🧠 First principles
Without a standard, every framework would need its own server. WSGI solved this in 2003 for synchronous apps: the server calls `app(environ, start_response)` and the app returns the body. But that design is inherently *blocking* — the function can't pause — and it can't express a long-lived connection like a WebSocket. ASGI redesigned the contract around `async`/`await` and *messages*: the app receives events and sends events.

### 💻 Code — the smallest possible ASGI app (no framework at all)

```python
async def app(scope, receive, send):
    assert scope["type"] == "http"

    body = b""
    while True:
        message = await receive()
        body += message.get("body", b"")
        if not message.get("more_body"):
            break

    await send({
        "type": "http.response.start",
        "status": 200,
        "headers": [(b"content-type", b"text/plain")],
    })
    await send({
        "type": "http.response.body",
        "body": b"Hello ASGI",
    })
```

### 🔍 Line by line
- `async def app(scope, receive, send):` — An ASGI application is any `async` callable that takes exactly these three arguments. `async def` means it's a coroutine function (it can pause with `await`).
  - `scope` — a dictionary describing the connection: `scope["type"]` is `"http"`, `"websocket"`, or `"lifespan"`; it also has `"path"`, `"method"`, `"headers"`, etc.
  - `receive` — an async function you call to get the next incoming message (e.g., a chunk of the request body).
  - `send` — an async function you call to send an outgoing message (headers, body chunks).
- `assert scope["type"] == "http"` — We only handle HTTP here; a real app would also handle lifespan and websocket types.
- `body = b""` — Start with empty bytes. `b""` is a bytes literal (raw bytes, not text).
- `while True: message = await receive()` — Request bodies can arrive in several chunks. Each call to `receive()` gives one message. `await` pauses until it's available.
- `body += message.get("body", b"")` — Append this chunk. `.get("body", b"")` returns empty bytes if the key is missing.
- `if not message.get("more_body"): break` — When the server says there's no more body, stop looping.
- `await send({... "http.response.start" ...})` — Responses are also messages. The first says: status code and headers. Headers are a list of `(name, value)` **byte** pairs.
- `await send({... "http.response.body" ...})` — Then the body. You could send several body messages to stream a response.

FastAPI hides all of this — but knowing it lets you write custom middleware and understand what Uvicorn does.

### ⚠️ Follow-ups
- *"What is the lifespan protocol?"* → A special `scope["type"] == "lifespan"` connection the server opens once so the app can run startup and shutdown code.

---

## Q5. What is the difference between ASGI and WSGI?

### 🧒 Simple explanation
WSGI is the *old* standard for synchronous Python web apps (Flask, classic Django). ASGI is the *new* standard that supports async code and long-lived connections. ASGI can do everything WSGI can, plus more.

| | WSGI | ASGI |
|---|---|---|
| Year | 2003 (PEP 3333) | 2016+ |
| Style | Synchronous: one request occupies one thread/process until done | Asynchronous: one event loop juggles many requests |
| Signature | `app(environ, start_response)` | `async app(scope, receive, send)` |
| Protocols | HTTP only | HTTP, WebSocket, lifespan, custom |
| Streaming | Awkward | Natural (message based) |
| Servers | Gunicorn, uWSGI, mod_wsgi | Uvicorn, Hypercorn, Daphne, Granian |
| Frameworks | Flask, Django (classic), Bottle | FastAPI, Starlette, Django (async), Litestar, Quart |

### 🎤 Interview answer
> "WSGI is a synchronous call-and-return contract, so a worker is tied up for the whole request. ASGI is an async, message-based contract, so one worker can interleave thousands of requests and also handle WebSockets. ASGI is a superset: you can run a WSGI app inside an ASGI server with an adapter, but not the other way round."

---

## Q6. What is Uvicorn?

### 🧒 Simple explanation
Uvicorn is the **server program**. It opens a port (like 8000), listens for incoming connections, reads the raw HTTP bytes, turns them into the ASGI `scope`/`receive`/`send` trio, and calls your FastAPI app. FastAPI cannot listen on a port by itself — it needs Uvicorn (or another ASGI server).

**Analogy:** FastAPI is the chef; Uvicorn is the front-of-house that answers the phone, takes the order, and carries the plate out.

### 🧠 First principles
Separation of concerns. Sockets, TLS, HTTP parsing, keep-alive connections, and the event loop are generic problems every app has; the server solves them once. Your framework focuses on routing and logic.

### 💻 Code — running it

```bash
uvicorn app.main:app --host 0.0.0.0 --port 8000 --workers 4
```

### 🔍 Piece by piece
- `uvicorn` — the command-line program.
- `app.main:app` — "import the module `app.main` (file `app/main.py`) and use the variable named `app` inside it". The colon separates module path from variable name.
- `--host 0.0.0.0` — listen on all network interfaces (needed inside Docker). `127.0.0.1` would mean "only this machine".
- `--port 8000` — the TCP port.
- `--workers 4` — start 4 separate processes, each running its own copy of the app and its own event loop, sharing the same port.

Other useful flags: `--reload` (dev only: restart on file changes), `--proxy-headers` (trust `X-Forwarded-*` headers from a load balancer), `--timeout-graceful-shutdown 20` (let in-flight requests finish on shutdown). Installing `uvicorn[standard]` adds `uvloop` and `httptools` for speed.

---

## Q7. What is Starlette and how is it related to FastAPI?

### 🧒 Simple explanation
Starlette is a small ASGI toolkit: it provides routing, middleware, `Request`/`Response` objects, WebSockets, background tasks, static files, and a test client. FastAPI is *built on top of* Starlette and adds the type-hint-based validation, dependency injection, and OpenAPI generation.

In fact, the `FastAPI` class is a Python subclass of Starlette's `Starlette` class. When you import `Request` from `fastapi`, you're getting Starlette's `Request` re-exported.

### 🎤 Interview answer
> "Starlette is the web engine; FastAPI is the developer-experience layer on top. Starlette handles ASGI, routing and middleware. FastAPI adds Pydantic validation, `Depends()` and docs. Understanding Starlette explains most of how FastAPI works internally."

---

## Q8. How do you create a basic FastAPI application?

### 💻 Code

```python
# app/main.py
from fastapi import FastAPI

app = FastAPI(
    title="Orders Service",
    version="1.0.0",
    description="Manages customer orders",
)

@app.get("/health")
async def health() -> dict[str, str]:
    return {"status": "ok"}
```

### 🔍 Line by line
- `# app/main.py` — A comment showing the file path. Convention: package `app`, module `main`.
- `from fastapi import FastAPI` — Import the application class from the `fastapi` package. `FastAPI` (capitalised) is a class; we'll create an instance of it.
- `app = FastAPI(...)` — Create the application object. This object holds all routes, middleware and settings. The name `app` is a convention; Uvicorn is told to look for it.
  - `title`, `version`, `description` — metadata shown in the generated docs and in the OpenAPI file.
- `@app.get("/health")` — A **decorator**. `app.get` returns a function that *registers* the function below it as the handler for HTTP method **GET** on path **`/health`**. Similar methods exist: `app.post`, `app.put`, `app.patch`, `app.delete`.
- `async def health() -> dict[str, str]:` — The handler. `async def` makes it a coroutine that runs on the event loop. `-> dict[str, str]` is the *return type hint*: "returns a dictionary with string keys and string values". FastAPI uses it for docs and (optionally) response validation.
- `return {"status": "ok"}` — Return a Python dict. FastAPI converts it to JSON text, sets the `Content-Type: application/json` header, and uses status code 200 by default.

---

## Q9. How do you start a FastAPI application using Uvicorn?

### 💻 Development
```bash
uvicorn app.main:app --reload --port 8000
```
`--reload` watches your files and restarts the server when they change. Never use it in production — it's slow and forks a watcher process.

### 💻 From Python (handy for debuggers)
```python
# run.py
import uvicorn

if __name__ == "__main__":
    uvicorn.run("app.main:app", host="0.0.0.0", port=8000, reload=True)
```
- `import uvicorn` — use Uvicorn as a library.
- `if __name__ == "__main__":` — only run when this file is executed directly (`python run.py`), not when imported.
- `uvicorn.run("app.main:app", ...)` — same as the CLI. Passing the import string (not the object) is required for `reload`/`workers` to work.

### 💻 Production
```bash
uvicorn app.main:app --host 0.0.0.0 --port 8000 --workers 4 --proxy-headers
```
Or with Gunicorn as a process manager:
```bash
gunicorn app.main:app -k uvicorn.workers.UvicornWorker -w 4 -b 0.0.0.0:8000
```
- `-k uvicorn.workers.UvicornWorker` — Gunicorn manages processes; each worker runs Uvicorn.
- `-w 4` — 4 workers. `-b` — bind address.
Gunicorn adds graceful worker restarts and `--max-requests` recycling. Modern Uvicorn has its own supervisor, so either is fine; in Kubernetes you usually run **one worker per container** and scale containers instead.

---

## Q10. What is an API path operation?

### 🧒 Simple explanation
In FastAPI's vocabulary, a **path** is the URL part (`/items/{id}`) and an **operation** is the HTTP method (GET, POST…). A *path operation* is one combination of the two, bound to a Python function. The term comes from the OpenAPI specification.

### 💻 Code
```python
@app.post("/items", status_code=201, tags=["items"], summary="Create an item")
async def create_item(item: ItemCreate) -> ItemRead:
    ...
```
### 🔍 Line by line
- `@app.post("/items", ...)` — Register a POST operation at `/items`.
- `status_code=201` — The default success status. 201 means "Created". Without this it would be 200.
- `tags=["items"]` — Groups this endpoint under the "items" heading in the docs.
- `summary="Create an item"` — Short text shown next to the endpoint in Swagger UI.
- `async def create_item(item: ItemCreate) -> ItemRead:` — `item: ItemCreate` tells FastAPI to read the JSON body and validate it as an `ItemCreate` model. `-> ItemRead` says the response will be shaped like `ItemRead`.
- `...` — the Ellipsis literal; a placeholder meaning "body omitted".

---

## Q11. What is the difference between GET, POST, PUT, PATCH, and DELETE?

### 🧒 Simple explanation
HTTP methods tell the server (and any caches in between) *what kind* of action you intend:
- **GET** — read something. Should not change anything.
- **POST** — create something new, or trigger an action.
- **PUT** — replace an existing thing entirely.
- **PATCH** — change part of an existing thing.
- **DELETE** — remove it.

### 🧠 First principles — two properties
- **Safe**: does not change server state. Only GET (and HEAD/OPTIONS) are safe. Caches and crawlers rely on this.
- **Idempotent**: doing it N times has the same result as doing it once. GET, PUT, DELETE are idempotent. POST is not (send "create order" twice → two orders). PATCH is usually not.

Idempotency matters because networks fail: if a client times out, it can safely *retry* an idempotent request but not a POST — which is why "idempotency keys" exist (file 11).

| Method | Purpose | Safe | Idempotent | Body |
|---|---|---|---|---|
| GET | Read | ✅ | ✅ | No |
| POST | Create / action | ❌ | ❌ | Yes |
| PUT | Replace whole resource | ❌ | ✅ | Yes |
| PATCH | Partial update | ❌ | ❌ (usually) | Yes |
| DELETE | Remove | ❌ | ✅ | Rarely |

### 💻 Code
```python
@app.put("/items/{item_id}")
async def replace_item(item_id: int, item: ItemCreate): ...

@app.patch("/items/{item_id}")
async def update_item(item_id: int, patch: ItemPatch): ...
```
- PUT takes the full `ItemCreate` (every field required) because it replaces the whole record.
- PATCH takes `ItemPatch` where every field is optional — only the fields sent are changed.

### ⚠️ Follow-ups
- *"Why does DELETE return 404 the second time if it's idempotent?"* → Idempotency is about the *server state* (item is gone either way), not the response code.

---

## Q12. How do you define path parameters?

### 💻 Code
```python
@app.get("/users/{user_id}")
async def get_user(user_id: int):
    return {"user_id": user_id}
```
### 🔍 Line by line
- `"/users/{user_id}"` — Curly braces mark a **variable segment**. Whatever appears in that position of the URL is captured.
- `user_id: int` — A function parameter with the **same name** as the placeholder receives the value. The `int` type hint makes FastAPI convert the text `"42"` into the integer `42`. If the URL had `/users/abc`, conversion fails and FastAPI returns a `422 Unprocessable Entity` error automatically.

---

## Q13. How do you define query parameters?

### 🧒 Simple explanation
Query parameters are the `?key=value&key2=value2` part after the path: `/users?page=2&size=20`.

### 💻 Code
```python
@app.get("/users")
async def list_users(page: int = 1, size: int = 20, q: str | None = None):
    return {"page": page, "size": size, "q": q}
```
### 🔍 Line by line
- Any function parameter that is **not** in the path and **not** a Pydantic model is treated as a query parameter.
- `page: int = 1` — Has a default, so it's optional. `?page=3` → `3`; omitted → `1`.
- `q: str | None = None` — `str | None` means "a string or nothing" (Python 3.10+ union syntax). Default `None` makes it optional; if the client omits `q`, the function gets `None`.
- Calling `/users?page=2&q=bob` → `page=2, size=20, q="bob"`.

---

## Q14. How do you define request body parameters?

### 🧒 Simple explanation
The **body** is the data sent with POST/PUT/PATCH, usually JSON. You describe its shape with a Pydantic model and declare a parameter of that type.

### 💻 Code
```python
from pydantic import BaseModel

class UserCreate(BaseModel):
    email: str
    name: str
    age: int | None = None

@app.post("/users", status_code=201)
async def create_user(user: UserCreate):
    return {"id": 1, **user.model_dump()}
```
### 🔍 Line by line
- `from pydantic import BaseModel` — `BaseModel` is the class all Pydantic models inherit from.
- `class UserCreate(BaseModel):` — Define a model. Each class attribute with a type hint becomes a **field**.
- `email: str` / `name: str` — Required string fields.
- `age: int | None = None` — Optional integer.
- `async def create_user(user: UserCreate):` — Because `UserCreate` is a Pydantic model, FastAPI reads the request body as JSON, validates it, and passes an instance as `user`.
- `**user.model_dump()` — `model_dump()` turns the model into a plain dict; `**` unpacks it into the surrounding dict literal. Result: `{"id": 1, "email": ..., "name": ..., "age": ...}`.

---

## Q15. What is Pydantic?

### 🧒 Simple explanation
Pydantic is a library for **defining data shapes and checking data against them**. You write a class with typed fields; Pydantic takes raw input (a dict or JSON), converts values to the right types when sensible, and raises a clear error listing every problem if something is wrong.

### 🧠 First principles
Data crossing a boundary — network, file, database, message queue — is untrusted. Somewhere you must guarantee "from here on, `age` is an integer between 0 and 150". Pydantic makes that guarantee *declarative* (you state the rule; it enforces it) and fast (v2's core is Rust).

---

## Q16. Why does FastAPI use Pydantic?

Because one Pydantic model is simultaneously:
1. a **validator** (parse incoming JSON),
2. a **serializer** (produce outgoing JSON),
3. a **JSON Schema generator** (feeds the OpenAPI docs), and
4. a **typed Python class** (editor autocompletion).

One definition, four uses — which is exactly FastAPI's "type hints as the single source of truth" philosophy.

---

## Q17. What is request validation?

### 🧒 Simple explanation
Before your function runs, FastAPI checks the incoming request against the declared types and rules: path parameters, query parameters, headers, cookies and body. If anything is wrong, the client gets a **422** response with a list of problems, and your function never runs.

### 💻 What the error looks like
```json
{
  "detail": [
    {
      "type": "int_parsing",
      "loc": ["path", "user_id"],
      "msg": "Input should be a valid integer",
      "input": "abc"
    }
  ]
}
```
- `type` — machine-readable error kind.
- `loc` — *where* the problem is: in the `path`, parameter `user_id`.
- `msg` — human-readable message.
- `input` — what the client actually sent.

FastAPI collects **all** errors, not just the first, so clients can fix everything in one go.

---

## Q18. How does FastAPI automatically generate API documentation?

### 🧒 Simple explanation
When the app starts, FastAPI walks through every registered route, looks at the function's parameters and their types, and the Pydantic models, and builds a big JSON description of the API called the **OpenAPI document**. It serves it at `/openapi.json`. Two web pages, Swagger UI (`/docs`) and ReDoc (`/redoc`), read that JSON and display it nicely.

### 💻 Enriching the docs
```python
from pydantic import BaseModel, Field

class Item(BaseModel):
    name: str = Field(..., description="Display name", examples=["Widget"])
    price: float = Field(..., gt=0, description="Price in USD")
```
- `Field(...)` — Adds metadata/constraints to a field. The first argument `...` (Ellipsis) means "required, no default".
- `description=` — Shown in docs.
- `examples=[...]` — Example values shown in docs and used to pre-fill "Try it out".
- `gt=0` — "greater than 0": both a validation rule *and* documented.

Docstrings on handler functions become the endpoint's long description.

---

## Q19. What is OpenAPI?

A vendor-neutral, machine-readable **specification format** (JSON or YAML) describing a REST API: its paths, operations, parameters, request/response schemas, and security schemes. It used to be called the *Swagger Specification*. Because it's standard, tools can generate client SDKs, server stubs, mock servers, tests, and documentation from it.

---

## Q20. What are Swagger UI and ReDoc?

Two web pages that *render* an OpenAPI document:
- **Swagger UI** at `/docs` — interactive. Has a "Try it out" button that sends real requests.
- **ReDoc** at `/redoc` — read-only, three-column layout, good for publishing reference docs.

For internal or sensitive APIs, disable them in production:
```python
app = FastAPI(docs_url=None, redoc_url=None, openapi_url=None)
```

---

## Q21. Explain how FastAPI processes a request from the time it reaches the server until the response is returned.

### 🧒 Simple explanation — the pipeline
```
1. Network bytes arrive on the port
2. Uvicorn parses HTTP → builds ASGI scope, calls app(scope, receive, send)
3. Middleware stack runs (outermost first)
4. Router matches method + path → finds your handler
5. Request object is built (headers, cookies, body access)
6. Dependencies are resolved (Depends tree)
7. Parameters and body are validated with Pydantic → 422 if invalid
8. Your handler runs (async: on the event loop; sync: in a thread pool)
9. Return value is validated/serialized to JSON via the response model
10. Dependencies' cleanup code runs (after yield)
11. Middleware stack unwinds (innermost first)
12. Uvicorn writes the response bytes to the socket
```

### 🧠 Detailed walkthrough
1. **Server layer.** Uvicorn accepts the TCP connection, parses the request line and headers with `httptools`, and builds the ASGI `scope` dict. It calls `app(scope, receive, send)`.
2. **Application entry.** `FastAPI.__call__` → `Starlette.__call__`. On the first request Starlette builds the middleware stack: each middleware is an ASGI app wrapping the next one, like layers of an onion.
3. **Middleware, request phase.** Runs from the outermost layer inward. Example: `CORSMiddleware` may answer an `OPTIONS` preflight right here and never reach the router.
4. **Routing.** The router loops over registered routes, matching the path (compiled to a regular expression) and method. No path match → 404. Path matches but method doesn't → 405.
5. **Request object.** Starlette wraps `scope` and `receive` in a `Request` object that gives convenient access to headers, cookies, query string and body.
6. **Dependency resolution.** FastAPI walks the handler's dependency graph (every `Depends`). Each dependency's own parameters are resolved recursively. Results are cached for this request so a dependency used twice runs once. For `yield` dependencies, the part before `yield` runs now.
7. **Validation.** Values are pulled from path, query, headers, cookies and body, and validated with Pydantic. Errors are collected, and if any exist a 422 is returned immediately.
8. **Handler execution.** If the handler is `async def`, it is awaited on the event loop. If it's plain `def`, it's sent to a thread pool via `run_in_threadpool` so it doesn't block the loop.
9. **Response serialization.** The return value is validated against `response_model` if declared (this also *filters* out fields not in the model), converted to JSON-safe types with `jsonable_encoder`, and wrapped in a `JSONResponse`.
10. **Dependency teardown.** Code after `yield` in dependencies runs (e.g., closing the DB session).
11. **Middleware, response phase.** Unwinds inner → outer. A logging middleware records the duration here.
12. **Server layer.** Uvicorn sends `http.response.start` and `http.response.body` messages to the socket.

### 🎤 Interview answer (short)
> "Uvicorn parses the bytes into an ASGI scope and calls the app. The request passes through the middleware stack, gets routed to a path operation, FastAPI resolves the dependency tree and validates parameters with Pydantic, runs the handler on the event loop or in a thread pool, serializes the return value against the response model, runs dependency cleanup, unwinds middleware, and Uvicorn writes the response."

### ⚠️ Follow-ups
- *"Where would a 422 come from vs a 404?"* → 404 at routing (step 4); 422 at validation (step 7).
- *"Can middleware see the validated body?"* → No — middleware runs *before* routing and validation; it only sees raw bytes.

# 06 — Middleware

---

## 📖 Vocabulary

> *Definition* is the precise meaning; *Say it in an interview* is the quotable one-liner.

| Term | Definition | Say it in an interview |
|---|---|---|
| **Middleware** | A component that wraps the application to intercept every request before routing and every response after handling; in ASGI, an app that holds and calls the next app. | "Middleware runs around every request and response, before routing happens." |
| **`call_next`** | In `BaseHTTPMiddleware`-style middleware, the coroutine that invokes the remaining stack and returns the response. | "`call_next` runs the rest of the pipeline and hands me the response." |
| **`BaseHTTPMiddleware`** | Starlette's convenience base class exposing `dispatch(request, call_next)`; simpler but heavier than pure ASGI middleware and can interfere with streaming. | "`BaseHTTPMiddleware` is convenient; pure ASGI middleware is faster and streaming-safe." |
| **Pure ASGI middleware** | Middleware implemented directly as `async __call__(scope, receive, send)`, wrapping `receive`/`send` to observe or modify messages. | "Pure ASGI middleware works at the message level with minimal overhead." |
| **Cross-cutting concern** | Functionality needed across many parts of a system regardless of business logic: logging, metrics, tracing, auth, CORS. | "Cross-cutting concerns belong in middleware, not in every handler." |
| **Correlation ID / Request ID** | A unique identifier assigned to a request and propagated through logs and downstream calls to group all activity for that request. | "A request ID ties together every log line for one request across services." |
| **`ContextVar`** | `contextvars.ContextVar`: a variable whose value is scoped to the current async task/context, safe where a global would leak between concurrent requests. | "ContextVars give per-request storage that's safe under asyncio." |
| **Origin** | The tuple of scheme, host and port of a URL (`https://app.example.com:443`); two URLs share an origin only if all three match. | "An origin is scheme plus host plus port." |
| **Same-origin policy** | The browser security rule that scripts may only read responses from their own origin unless the server opts in via CORS. | "The same-origin policy stops one site's JavaScript reading another site's responses." |
| **CORS** | *Cross-Origin Resource Sharing*: a set of response headers (`Access-Control-Allow-Origin`, etc.) by which a server permits specified foreign origins to read its responses. Enforced by browsers only. | "CORS is the server telling browsers which foreign origins may read its responses." |
| **Preflight** | An automatic `OPTIONS` request browsers send before non-simple cross-origin requests, asking which methods/headers are permitted. | "Preflight is the browser's permission check before a cross-origin PUT or JSON POST." |
| **Security headers** | Response headers instructing browsers to enforce protections: HSTS, `X-Content-Type-Options`, `X-Frame-Options`, CSP, `Referrer-Policy`. | "Security headers harden the browser side against downgrade, sniffing and framing attacks." |
| **HSTS** | *HTTP Strict Transport Security*: header telling browsers to use HTTPS only for the domain for a period. | "HSTS pins the browser to HTTPS." |
| **CSP** | *Content Security Policy*: header restricting which sources a page may load scripts, styles and frames from. | "CSP whitelists what a page may load; for an API it's effectively 'nothing'." |
| **Clickjacking** | Tricking users into clicking hidden UI by embedding a page in an invisible iframe; mitigated by `X-Frame-Options`/`frame-ancestors`. | "Clickjacking is defeated by refusing to be framed." |
| **Exception handler** | A function registered with `@app.exception_handler(ExcType)` that converts a raised exception into an HTTP response. | "Exception handlers map domain errors to consistent HTTP responses." |
| **Error envelope** | A consistent JSON structure for all error responses (`{"error": {"code", "message", "request_id"}}`). | "One error envelope means every client parses errors the same way." |
| **`RequestValidationError`** | The exception FastAPI raises when request validation fails; default handler returns 422 with `detail`. | "Override `RequestValidationError` to shape 422s into your envelope." |
| **`ServerErrorMiddleware`** | Starlette's outermost middleware that catches unhandled exceptions and returns 500 (invoking a registered `Exception` handler). | "The 500 handler lives in the outermost layer, outside my other middleware." |
| **Structured logging** | Emitting logs as machine-parseable records (JSON) with named fields rather than free text. | "Structured logs are queryable by field, which is essential at scale." |
| **Redaction** | Removing or masking sensitive values (passwords, tokens, PII) from logs and telemetry. | "Redact at the source and again in the log pipeline." |
| **Prometheus histogram** | A metric type recording observations into cumulative buckets plus sum and count, enabling percentile estimation via `histogram_quantile`. | "Latency goes in a histogram so I can compute p99 across pods." |
| **Label cardinality** | The number of distinct label-value combinations for a metric; high cardinality (e.g., raw paths, user IDs) overwhelms the metrics store. | "Label by route template, never raw path, to keep cardinality bounded." |
| **Distributed tracing** | Recording a request's path across services as a tree of timed spans sharing a trace ID. | "Tracing shows where time went across services for a single request." |
| **W3C Trace Context / `traceparent`** | The standard header format (`00-<trace-id>-<span-id>-<flags>`) for propagating trace identity between services. | "`traceparent` is the header that carries the trace across service boundaries." |
| **OpenTelemetry (OTel)** | The CNCF vendor-neutral standard, SDKs and Collector for producing and exporting traces, metrics and logs. | "OTel is the one SDK for traces, metrics and logs, exported to any backend." |
| **Instrumentation** | Code (manual or auto) that generates telemetry from a library: `FastAPIInstrumentor`, `HTTPXClientInstrumentor`. | "Auto-instrumentation gives me spans for FastAPI, httpx and SQLAlchemy without code changes." |
| **Span processor / exporter** | Processor: batches/filters spans (`BatchSpanProcessor`). Exporter: sends them to a backend or Collector (`OTLPSpanExporter`). | "Spans are batched by the processor and shipped by the exporter." |

---

## Q1. What is middleware?

### 🧒 Simple explanation
Code that runs for **every** request, both on the way in (before your handler) and on the way out (after your handler). Think of airport security: every passenger passes through, regardless of destination.

### 🧠 First principles
Some concerns are orthogonal to business logic — logging, timing, CORS, compression, tracing. Putting them in every handler duplicates code; putting them in the router couples things. ASGI makes composition easy: each middleware is itself an ASGI app that holds a reference to the *next* app and can act before/after calling it.

---

## Q2. How do you create middleware in FastAPI?

### 💻 Option A — function style (`@app.middleware("http")`)
```python
import time
from fastapi import FastAPI, Request

app = FastAPI()

@app.middleware("http")
async def timing(request: Request, call_next):
    start = time.perf_counter()
    response = await call_next(request)
    elapsed_ms = (time.perf_counter() - start) * 1000
    response.headers["X-Response-Time-ms"] = f"{elapsed_ms:.1f}"
    return response
```
### 🔍 Line by line
- `@app.middleware("http")` — registers an HTTP middleware (internally wraps it in `BaseHTTPMiddleware`).
- `async def timing(request, call_next)` — `call_next` is the next layer; ultimately the router and your handler.
- `time.perf_counter()` — a high-resolution monotonic clock (unaffected by system time changes).
- `response = await call_next(request)` — run everything downstream; get the response back.
- `response.headers[...] = ...` — add a header. `f"{elapsed_ms:.1f}"` formats to one decimal place.
- `return response` — pass it up the chain.

### 💻 Option B — pure ASGI middleware (faster, streaming-safe)
```python
from starlette.datastructures import MutableHeaders

class TimingMiddleware:
    def __init__(self, app):
        self.app = app

    async def __call__(self, scope, receive, send):
        if scope["type"] != "http":
            return await self.app(scope, receive, send)
        start = time.perf_counter()

        async def send_wrapper(message):
            if message["type"] == "http.response.start":
                elapsed = (time.perf_counter() - start) * 1000
                headers = MutableHeaders(scope=message)
                headers["X-Response-Time-ms"] = f"{elapsed:.1f}"
            await send(message)

        await self.app(scope, receive, send_wrapper)

app.add_middleware(TimingMiddleware)
```
### 🔍 Line by line
- `__init__(self, app)` — receives the *next* ASGI app.
- `__call__(self, scope, receive, send)` — the ASGI signature (file 01).
- `if scope["type"] != "http": return await self.app(...)` — pass WebSocket/lifespan through untouched.
- `async def send_wrapper(message)` — intercept outgoing messages.
- `if message["type"] == "http.response.start"` — the message carrying status + headers; inject the header here.
- `MutableHeaders(scope=message)` — helper that edits the raw header list in the message.
- `await self.app(scope, receive, send_wrapper)` — call downstream with our wrapped `send`.
- `app.add_middleware(TimingMiddleware)` — install it.

`BaseHTTPMiddleware` is convenient but has costs: it can buffer streaming responses, historically broke `contextvars` propagation, and adds overhead. For production-critical middleware prefer pure ASGI.

---

## Q3. What is the execution order of middleware?

The middleware added **last** is **outermost**: it runs first on the way in and last on the way out.
```python
app.add_middleware(A)
app.add_middleware(B)
app.add_middleware(C)
# Request:  C → B → A → router → handler
# Response: handler → A → B → C
```
Each `add_middleware` wraps the existing stack. Practical order (outermost first): error handling / tracing → CORS → auth context → logging → compression → app.

---

## Q4. How do you implement request logging?

```python
import logging, time
from starlette.middleware.base import BaseHTTPMiddleware

log = logging.getLogger("access")

class AccessLogMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request, call_next):
        start = time.perf_counter()
        try:
            response = await call_next(request)
        except Exception:
            log.exception("request_failed", extra={"path": request.url.path})
            raise
        log.info(
            "request",
            extra={
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
### 🔍 Line by line
- `logging.getLogger("access")` — a named logger; configure it to emit JSON (e.g., `python-json-logger`) so fields are queryable.
- `class ...(BaseHTTPMiddleware)` with `dispatch(self, request, call_next)` — class form of Option A.
- `log.exception(...)` — logs with the stack trace; then `raise` so exception handlers still run.
- `extra={...}` — structured fields on the log record. Never log bodies or the `Authorization` header here.
- `request.client.host` — client IP (behind a proxy, use `X-Forwarded-For` only if the proxy is trusted).

---

## Q5. How do you measure request latency?

Use `time.perf_counter()` around `call_next` (above). Emit as a response header (debugging), a structured log field, and a **Prometheus histogram** labelled by route template and status:
```python
from prometheus_client import Histogram
REQ_LATENCY = Histogram(
    "http_request_duration_seconds", "Latency", ["method", "route", "status"],
    buckets=(.005, .01, .025, .05, .1, .25, .5, 1, 2.5, 5),
)
# after call_next:
route = request.scope.get("route")
REQ_LATENCY.labels(request.method, route.path if route else "unmatched", response.status_code).observe(elapsed)
```
- Label by the **route template** (`/users/{id}`), never the raw path, or you create unbounded label combinations.

---

## Q6. How do you add correlation IDs?

```python
import contextvars, logging, uuid
from starlette.datastructures import MutableHeaders

request_id_ctx = contextvars.ContextVar("request_id", default="-")

class RequestIDMiddleware:
    def __init__(self, app):
        self.app = app

    async def __call__(self, scope, receive, send):
        if scope["type"] != "http":
            return await self.app(scope, receive, send)
        headers = dict(scope["headers"])
        rid = headers.get(b"x-request-id", b"").decode() or str(uuid.uuid4())
        token = request_id_ctx.set(rid)

        async def send_wrapper(message):
            if message["type"] == "http.response.start":
                MutableHeaders(scope=message)["X-Request-ID"] = rid
            await send(message)

        try:
            await self.app(scope, receive, send_wrapper)
        finally:
            request_id_ctx.reset(token)

class RequestIDFilter(logging.Filter):
    def filter(self, record):
        record.request_id = request_id_ctx.get()
        return True
```
### 🔍 Line by line
- `contextvars.ContextVar("request_id", default="-")` — per-task variable. Each request's task sees its own value.
- `dict(scope["headers"])` — headers are a list of `(bytes, bytes)`; convert to a dict for lookup. Names are lowercase bytes.
- `headers.get(b"x-request-id", b"").decode() or str(uuid.uuid4())` — honour an incoming ID (from a gateway/client) or mint a new UUID.
- `token = request_id_ctx.set(rid)` — store it; `token` lets us restore the previous value later.
- `send_wrapper` — echo the ID back in the response so clients can quote it in support tickets.
- `finally: request_id_ctx.reset(token)` — clean up so nothing leaks between tasks.
- `RequestIDFilter` — a logging filter that stamps `request_id` onto every log record. Add `%(request_id)s` to the formatter and every line for a request shares the ID.
- Propagate downstream: `http.get(url, headers={"X-Request-ID": request_id_ctx.get()})`.

---

## Q7. What is CORS?

### 🧒 Simple explanation
Browsers enforce the **same-origin policy**: JavaScript on `https://app.example.com` cannot *read* a response from `https://api.example.com` unless the API explicitly says it's allowed. CORS is the set of response headers that grant that permission.

**Why it exists:** it protects *users*. Without it, a malicious website could make your browser call your bank's API (using your logged-in cookies) and read the reply. CORS does not protect your server — `curl` ignores it entirely.

**Preflight:** for "non-simple" requests (JSON body, custom headers, PUT/DELETE), the browser first sends `OPTIONS` with `Access-Control-Request-Method`; the server must reply with `Access-Control-Allow-*` headers before the real request goes out.

## Q8. How do you implement CORS?

```python
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://app.example.com", "https://admin.example.com"],
    allow_origin_regex=r"https://.*\.preview\.example\.com",
    allow_credentials=True,
    allow_methods=["GET", "POST", "PUT", "PATCH", "DELETE"],
    allow_headers=["Authorization", "Content-Type", "X-Request-ID"],
    expose_headers=["X-Request-ID"],
    max_age=600,
)
```
- `allow_origins` — explicit list. Never `["*"]` together with credentials (browsers reject that anyway).
- `allow_origin_regex` — for dynamic preview environments.
- `allow_credentials=True` — allow cookies / `Authorization` header.
- `allow_headers` — request headers the browser may send.
- `expose_headers` — response headers JavaScript may *read* (by default only a few are visible).
- `max_age=600` — cache the preflight result for 10 minutes.
- Add CORS **last** (outermost) so preflights are answered even if an auth middleware would otherwise reject them.

---

## Q9. How do you add security headers?

```python
class SecurityHeadersMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request, call_next):
        resp = await call_next(request)
        resp.headers.setdefault("Strict-Transport-Security", "max-age=63072000; includeSubDomains; preload")
        resp.headers.setdefault("X-Content-Type-Options", "nosniff")
        resp.headers.setdefault("X-Frame-Options", "DENY")
        resp.headers.setdefault("Referrer-Policy", "no-referrer")
        resp.headers.setdefault("Content-Security-Policy", "default-src 'none'; frame-ancestors 'none'")
        resp.headers.setdefault("Cache-Control", "no-store")
        return resp
```
- `Strict-Transport-Security` — force HTTPS for 2 years.
- `X-Content-Type-Options: nosniff` — stop browsers guessing content types.
- `X-Frame-Options: DENY` — prevent embedding in iframes (clickjacking).
- `Referrer-Policy` — don't leak URLs to other sites.
- `Content-Security-Policy` — for an API, "load nothing, embed nowhere".
- `Cache-Control: no-store` — private API data must not be cached by browsers/proxies.
- `setdefault` — only set if a handler hasn't already. Often also set at the ingress/CDN; doing both is cheap defense in depth.

---

## Q10. How do you handle exceptions globally?

```python
import logging
from fastapi import Request
from fastapi.responses import JSONResponse
from fastapi.exceptions import RequestValidationError

log = logging.getLogger(__name__)

class AppError(Exception):
    status_code = 400
    code = "app_error"
    def __init__(self, message: str):
        self.message = message

class NotFound(AppError):
    status_code = 404
    code = "not_found"

@app.exception_handler(AppError)
async def app_error_handler(request: Request, exc: AppError):
    return JSONResponse(
        exc.status_code,
        {"error": {"code": exc.code, "message": exc.message, "request_id": request_id_ctx.get()}},
    )

@app.exception_handler(RequestValidationError)
async def validation_handler(request: Request, exc: RequestValidationError):
    return JSONResponse(422, {"error": {"code": "validation_error", "details": exc.errors()}})

@app.exception_handler(Exception)
async def unhandled_handler(request: Request, exc: Exception):
    log.exception("unhandled", extra={"request_id": request_id_ctx.get()})
    return JSONResponse(
        500,
        {"error": {"code": "internal_error", "message": "Something went wrong",
                   "request_id": request_id_ctx.get()}},
    )
```
### 🔍 Line by line
- `class AppError(Exception)` — a *domain* exception hierarchy. Services raise these without knowing about HTTP.
- Class attributes `status_code`/`code` — each subclass sets its own.
- `@app.exception_handler(AppError)` — one handler covers the whole hierarchy and produces a consistent **error envelope**.
- `RequestValidationError` handler — reshape FastAPI's default 422 to match your envelope.
- `Exception` handler — catch-all: log with stack trace, hide internals, return the request ID for support.
- Note: the `Exception` handler runs in `ServerErrorMiddleware`, *outside* your other middleware; `BaseHTTPMiddleware` still sees the exception re-raised, so wrap `call_next` in `try/except` there if needed.

---

## Q11. Middleware vs dependency — when would you use each?

**Middleware** for things that apply to every request and don't need parsed parameters: logging, tracing, CORS, compression, request ID.
**Dependency** for things that produce a *value* the handler uses or that vary per route: current user, DB session, pagination, permission checks.
Auth is the interesting case: *token parsing* could be either; *loading the user object and injecting it* is a dependency.

---

## Q12. How can middleware affect application performance?

- Every layer adds a coroutine frame; `BaseHTTPMiddleware` adds an extra task and memory stream per request (~50–100 µs and extra memory).
- Reading the request body in middleware buffers it → costly for uploads, breaks streaming.
- Blocking calls in middleware (sync logging to a slow sink, sync Redis) freeze the loop for **everything**.
- Work proportional to payload size (logging full bodies, regex on headers) scales badly.
- Ordering: put cheap "reject early" middleware (rate limit, auth) outside expensive ones so rejected requests don't pay for logging/compression.
- Measure: load test with/without each middleware; budget < 1 ms total.

---

## Q13. How do you exclude specific routes from middleware logic?

Middleware runs before routing, so it can't see the route object — but it can see the path:
```python
SKIP = {"/health", "/metrics", "/docs", "/openapi.json"}

class AccessLogMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request, call_next):
        if request.url.path in SKIP or request.url.path.startswith("/internal/"):
            return await call_next(request)
        ...
```
Alternatives: mount noisy endpoints on a sub-application without that middleware (`app.mount("/internal", internal_app)`), or invert it — use a *dependency* on only the routers that need the behaviour.

---

## Q14. How do you implement request/response logging without logging sensitive information?

1. **Don't log bodies by default.** Log method, route template, status, duration, request ID, user ID.
2. If bodies are needed (debug environments only), **redact** with a deny-list of keys and cap size:
```python
REDACT = {"password", "token", "authorization", "card_number", "ssn", "secret"}

def redact(obj):
    if isinstance(obj, dict):
        return {k: ("***" if k.lower() in REDACT else redact(v)) for k, v in obj.items()}
    if isinstance(obj, list):
        return [redact(i) for i in obj]
    return obj
```
- Recursive: walks dicts and lists, replacing sensitive values.
3. Strip headers `Authorization`, `Cookie`, `X-API-Key`.
4. Truncate to e.g. 2 KB.
5. Mask query-string tokens (`?token=`).
6. Use `SecretStr` in models so accidental `repr` doesn't leak.
7. A second redaction pass in the log pipeline (Fluent Bit / Vector) — defense in depth.

---

## Q15. How would you implement distributed tracing middleware?

Don't hand-roll it — OpenTelemetry's FastAPI instrumentation *is* an ASGI middleware:

```python
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.sdk.resources import Resource
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor
from opentelemetry.instrumentation.httpx import HTTPXClientInstrumentor
from opentelemetry.instrumentation.sqlalchemy import SQLAlchemyInstrumentor

provider = TracerProvider(resource=Resource.create({"service.name": "orders-api"}))
provider.add_span_processor(BatchSpanProcessor(OTLPSpanExporter(endpoint="http://otel-collector:4317")))
trace.set_tracer_provider(provider)

FastAPIInstrumentor.instrument_app(app, excluded_urls="health,metrics")
HTTPXClientInstrumentor().instrument()
SQLAlchemyInstrumentor().instrument(engine=engine.sync_engine)
```
### 🔍 Line by line
- `TracerProvider(resource=...)` — identifies this service in every span.
- `BatchSpanProcessor(OTLPSpanExporter(...))` — buffer spans and ship them in batches over OTLP/gRPC to a **collector**, which forwards to Jaeger/Tempo/X-Ray.
- `trace.set_tracer_provider(provider)` — make it global.
- `FastAPIInstrumentor.instrument_app(app, excluded_urls=...)` — adds middleware that: reads `traceparent` from incoming headers, starts a *server span* per request, records route/status, and makes the span "current" for the request's context.
- `HTTPXClientInstrumentor().instrument()` — every outgoing `httpx` call becomes a *child span* **and gets `traceparent` injected**, so the next service continues the same trace.
- `SQLAlchemyInstrumentor().instrument(engine=...)` — every SQL statement becomes a span.

Manual version, if asked: extract context from headers → `tracer.start_as_current_span(name, context=ctx, kind=SERVER)` → set attributes → on outgoing calls `propagator.inject(headers)`. Put `trace_id` into the log context so logs and traces link.

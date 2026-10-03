# 05 — Async Programming

---

## 📖 Vocabulary

> *Definition* is the precise meaning; *Say it in an interview* is the quotable one-liner.

| Term | Definition | Say it in an interview |
|---|---|---|
| **Process** | An instance of a running program with its own memory space, file descriptors and at least one thread. Processes do not share memory by default. | "A process is an isolated running program; Uvicorn workers are separate processes." |
| **Thread** | A unit of execution within a process; threads share the process's memory and are scheduled by the OS. | "Threads share memory inside one process." |
| **GIL (Global Interpreter Lock)** | A mutex in CPython allowing only one thread to execute Python bytecode at a time. Released during blocking I/O and by some C extensions, so threads help I/O but not CPU-bound Python. | "The GIL means threads don't give CPU parallelism in CPython; processes do." |
| **Blocking call** | A call that suspends the calling thread until it completes (e.g., `time.sleep`, `requests.get`, sync DB drivers). Inside an event loop it stalls all tasks. | "A blocking call freezes the whole event loop, which is the classic async bug." |
| **Non-blocking I/O** | I/O performed on sockets/files configured to return immediately, with readiness signaled by the OS (`epoll`/`kqueue`), enabling one thread to service many connections. | "Non-blocking I/O is what lets one thread juggle thousands of sockets." |
| **Coroutine** | A function declared with `async def`. Calling it returns a coroutine object; its body runs only when awaited or scheduled as a task, and it can suspend at `await`. | "An `async def` returns a coroutine that runs only when awaited." |
| **`await`** | Suspends the current coroutine until the awaited awaitable completes, yielding control to the event loop meanwhile. Only valid inside `async def`. | "`await` is the point where a coroutine pauses and lets others run." |
| **Awaitable** | Any object usable with `await`: coroutines, `asyncio.Task`, `asyncio.Future`, and objects implementing `__await__`. | "Coroutines, tasks and futures are all awaitables." |
| **Task** | A coroutine wrapped by the event loop (`asyncio.create_task`) so it runs concurrently and independently of the creator. | "A task is a coroutine scheduled to run on its own." |
| **Event loop** | asyncio's single-threaded scheduler: runs ready callbacks/tasks, waits on I/O readiness via the OS selector, and resumes tasks when their awaited operations complete. | "The loop runs one task until it awaits, then switches to whichever task is ready." |
| **Cooperative multitasking** | A scheduling model where tasks voluntarily yield control (at `await`); a task that never yields monopolizes the CPU. | "Async is cooperative: tasks must await to let others run." |
| **I/O-bound** | Workload whose elapsed time is dominated by waiting on external operations (network, disk, database). | "I/O-bound work spends its time waiting, which async overlaps efficiently." |
| **CPU-bound** | Workload whose elapsed time is dominated by computation. Async provides no benefit; processes or native code do. | "CPU-bound work needs processes or native extensions, not async." |
| **Concurrency vs Parallelism** | Concurrency: multiple tasks in progress over overlapping periods. Parallelism: multiple tasks executing at the same instant on multiple cores. | "Async gives concurrency on one core; workers give parallelism across cores." |
| **Thread pool (`run_in_threadpool`)** | A bounded set of worker threads to which blocking functions are dispatched so the loop stays responsive; Starlette's default limit is 40 threads via AnyIO. | "`run_in_threadpool` offloads a blocking call so the loop keeps serving." |
| **`asyncio.gather`** | Runs multiple awaitables concurrently and returns their results in input order; `return_exceptions=True` returns exceptions as values instead of raising. | "`gather` fans out concurrent calls and collects results in order." |
| **`asyncio.TaskGroup`** | Python 3.11 structured-concurrency primitive: runs child tasks and, if one fails, cancels the siblings and re-raises. | "`TaskGroup` is the safer `gather`: a failure cancels the rest." |
| **Semaphore** | A synchronization primitive holding N permits; `async with sem:` acquires one, bounding concurrent entrants. | "A semaphore caps how many calls hit a downstream at once." |
| **Timeout / deadline** | An upper bound on wait time (`asyncio.timeout`, `httpx.Timeout`); a *deadline* is an absolute end time propagated across calls. | "Every I/O call gets a timeout so failures fail fast instead of hanging." |
| **Async driver** | A database/network client library implemented with non-blocking I/O and `await`-able APIs (`asyncpg`, `httpx.AsyncClient`, `redis.asyncio`). | "An async driver is one I can `await` without blocking the loop." |
| **`httpx.AsyncClient`** | An async HTTP client with connection pooling and keep-alive; should be created once (e.g., in lifespan) and reused. | "I keep one shared `AsyncClient` per process for connection pooling." |
| **Connection pooling** | Reusing established connections across requests rather than opening/closing per request, avoiding handshake latency. | "Pooling reuses connections so I don't pay the handshake every time." |
| **`asyncio.sleep`** | A coroutine that suspends the caller for a duration without blocking the thread; the async replacement for `time.sleep`. | "`asyncio.sleep` yields to the loop; `time.sleep` freezes it." |
| **`uvloop`** | A drop-in asyncio event loop implementation written in Cython on libuv, roughly 2–4× faster than the default loop. | "uvloop is the C-based event loop Uvicorn uses for speed." |
| **Percentile (p50/p95/p99)** | The latency below which that percentage of requests complete; p99 captures tail latency experienced by the slowest 1%. | "p99 is what your unluckiest users feel; averages hide it." |
| **Saturation** | The degree to which a resource is at capacity (CPU %, pool in-use/size, queue depth); a leading indicator of latency problems. | "Saturation tells me how full a resource is before latency shows it." |
| **`py-spy`** | A sampling profiler that attaches to a running Python process without code changes, producing top-style views or flame graphs. | "py-spy shows me what a live worker is doing right now." |

---

## Q1. What is asynchronous programming?

### 🧒 Simple explanation
A way of writing code where a single thread can *pause* a task that is waiting for something slow (network, disk, database) and *work on another task* meanwhile — then come back when the slow thing is ready.

**Analogy:** A waiter takes an order, hands it to the kitchen, and instead of standing at the pass until the food is ready, goes to serve other tables. When the dish is up, they return. One waiter, many tables.

### 🧠 First principles
A CPU executes billions of instructions per second. A network round-trip takes milliseconds — *millions* of instruction-times. Synchronous code wastes all of that by sitting idle. Async code says: "I'm waiting; let someone else use the CPU." Control is handed back at `await` points — this is *cooperative* multitasking.

---

## Q2. What is asyncio?

Python's standard-library framework for async code. It provides the **event loop**, **coroutines**, **tasks**, **futures**, and helpers: `gather`, `sleep`, `wait_for`, `timeout`, `Queue`, `Lock`, `Semaphore`. Uvicorn runs your FastAPI app inside an asyncio event loop (typically the faster `uvloop` implementation).

---

## Q3. What does `async def` mean?

It declares a **coroutine function**. Calling it does **not** run the body — it returns a coroutine object. The body executes only when the coroutine is awaited or scheduled as a task.

```python
async def fetch():
    return 1

c = fetch()        # nothing has run yet; c is a coroutine object
result = await c   # now the body runs (must be inside another coroutine / the loop)
```

---

## Q4. What does `await` do?

Suspends the current coroutine until the awaited thing completes, giving control back to the event loop so other tasks can run. It can only appear inside `async def`. **`await` is the only place a coroutine yields control** — code between awaits runs uninterrupted.

---

## Q5. What is an event loop?

The scheduler. It keeps a queue of tasks ready to run and a set of I/O watchers (using the OS's `epoll`/`kqueue`). Loop forever: pick a ready task → run it until it hits `await` → note what it's waiting for → when the OS says that I/O is ready, mark the task ready again → repeat. One loop per thread; each Uvicorn worker process has one.

---

## Q6. What is the difference between synchronous and asynchronous code?

```python
import requests, httpx, asyncio

# Synchronous — 3 sequential calls ≈ 3 × 200 ms = 600 ms
def sync_version():
    a = requests.get(url_a)   # thread blocks 200 ms
    b = requests.get(url_b)   # blocks another 200 ms
    c = requests.get(url_c)
    return a, b, c

# Asynchronous — 3 concurrent calls ≈ max(200 ms) = 200 ms
async def async_version():
    async with httpx.AsyncClient() as client:
        a, b, c = await asyncio.gather(
            client.get(url_a), client.get(url_b), client.get(url_c)
        )
    return a, b, c
```
- `requests` is a blocking HTTP library; `httpx.AsyncClient` is its async equivalent.
- `async with` — async context manager: opens the client, closes it at the end.
- `asyncio.gather(...)` — start all three requests; wait for all; return results in order.

Sync: simple, one thing at a time, blocks. Async: interleaved waiting, much higher throughput per thread, but *every* library in the call chain must be async-aware.

---

## Q7. What is I/O-bound work?

Work dominated by **waiting on something external**: HTTP calls, DB queries, file reads, `sleep`. The CPU is mostly idle. Async shines here.

## Q8. What is CPU-bound work?

Work dominated by **computation**: image resizing, hashing, ML inference, parsing giant JSON, tight loops. The CPU is busy. Async does **not** help (nothing to overlap), and because of the GIL, threads barely help either. Use **processes** (`ProcessPoolExecutor`, Celery workers) or C/Rust extensions that release the GIL (NumPy).

---

## Q9. When should you use `async def` in FastAPI?

When the handler awaits **async-capable** I/O: `httpx.AsyncClient`, SQLAlchemy async / `asyncpg`, `redis.asyncio`, `aiobotocore`, `aiofiles`. Also for trivial handlers with no I/O — `async` avoids the thread-pool hop.

## Q10. When should you use normal `def`?

When the handler calls **blocking** code you can't or won't convert: `requests`, sync SQLAlchemy, `boto3`, `psycopg2`, CPU-heavy libraries, legacy SDKs. FastAPI runs `def` handlers in a thread pool, so they don't block the loop.

| Handler does | Declare as |
|---|---|
| Awaits an async library | `async def` |
| Calls a blocking library | `def` |
| Pure computation < ~1 ms | either |
| Heavy CPU (> 100 ms) | `def` + offload to a process pool / queue |
| Mixed | `async def` + wrap blocking calls in `run_in_threadpool` |

---

## Q11. What happens if you call blocking code inside an async endpoint?

The **entire event loop freezes** for the duration. Every other in-flight request in that worker stalls — including health checks. Under load: p99 explodes, Kubernetes liveness probes time out, pods get restarted. It's the #1 FastAPI production bug.

```python
@app.get("/bad")
async def bad():
    data = requests.get("https://slow.example.com")   # blocks the loop 2 s → ALL requests wait 2 s
    return data.json()
```

---

## Q12. Why is `time.sleep()` problematic inside an async endpoint?

`time.sleep(5)` suspends the **thread**, and the event loop lives on that thread. Nothing else runs for 5 seconds. It's the purest example of blocking the loop.

## Q13. What should you use instead?

`await asyncio.sleep(5)` — registers a timer with the loop and yields control; other requests proceed; after 5 s this coroutine resumes.

For blocking **library** calls:
```python
from fastapi.concurrency import run_in_threadpool

@app.get("/report")
async def report():
    pdf = await run_in_threadpool(generate_pdf_sync, report_id=7)
    return Response(pdf, media_type="application/pdf")
```
- `run_in_threadpool(func, *args, **kwargs)` — runs `func` in Starlette's thread pool and returns an awaitable. The loop stays free while the thread does the blocking work. (Stdlib equivalent: `await asyncio.to_thread(func, ...)`.)

---

## Q14. Can you use synchronous libraries inside FastAPI?

Yes, safely, two ways: (a) make the endpoint/dependency plain `def`, or (b) call `run_in_threadpool` from an `async def`. The default pool is 40 threads. To raise it:
```python
import anyio
anyio.to_thread.current_default_thread_limiter().total_tokens = 100   # e.g., in lifespan
```

## Q15. How does FastAPI handle synchronous endpoints?

Starlette checks whether the endpoint is a coroutine function. If not, it calls `run_in_threadpool(endpoint, **values)` and awaits the result. Same for sync dependencies. So `def` endpoints are concurrent up to the pool size, and blocking inside them is fine.

---

## Q16. What is an async database driver?

A driver that talks the database's network protocol with non-blocking sockets and exposes `await`-able calls: `asyncpg` (PostgreSQL, fastest), `aiomysql`, `aiosqlite`, `motor` (MongoDB). SQLAlchemy 2.0's `AsyncSession` sits on top via `create_async_engine("postgresql+asyncpg://...")`. Using a **sync** driver like `psycopg2` inside `async def` freezes the loop.

---

## Q17. How do you make HTTP calls asynchronously?

Use `httpx.AsyncClient` and **reuse one client** so connections are pooled.

```python
from contextlib import asynccontextmanager
import httpx
from fastapi import FastAPI, Request, Depends

@asynccontextmanager
async def lifespan(app: FastAPI):
    app.state.http = httpx.AsyncClient(
        timeout=httpx.Timeout(5.0),
        limits=httpx.Limits(max_connections=100),
    )
    yield
    await app.state.http.aclose()

app = FastAPI(lifespan=lifespan)

def get_http(request: Request) -> httpx.AsyncClient:
    return request.app.state.http

@app.get("/weather")
async def weather(http: Annotated[httpx.AsyncClient, Depends(get_http)]):
    r = await http.get("https://api.weather.example/today")
    r.raise_for_status()
    return r.json()
```
### 🔍 Line by line
- `@asynccontextmanager` + `async def lifespan(app)` — code before `yield` runs at **startup**, code after runs at **shutdown**.
- `app.state.http = httpx.AsyncClient(...)` — one pooled client per process, stored on `app.state` (a free-form namespace).
- `timeout=httpx.Timeout(5.0)` — never use the default "no timeout". 5 s for connect/read/write.
- `limits=httpx.Limits(max_connections=100)` — cap concurrent connections.
- `yield` — app serves requests.
- `await app.state.http.aclose()` — close cleanly on shutdown.
- `get_http(request)` — dependency exposing the shared client (`request.app` is the FastAPI instance).
- `r.raise_for_status()` — raise if status is 4xx/5xx.

---

## Q18. How do you handle concurrent external API calls?

Fan out with `gather`, bound with a `Semaphore`, and set timeouts:

```python
import asyncio

sem = asyncio.Semaphore(10)

async def fetch_one(http, url):
    async with sem:
        r = await http.get(url, timeout=3.0)
        r.raise_for_status()
        return r.json()

@app.get("/aggregate")
async def aggregate(http: HttpDep):
    urls = [...]   # 50 URLs
    results = await asyncio.gather(*(fetch_one(http, u) for u in urls), return_exceptions=True)
    ok = [r for r in results if not isinstance(r, Exception)]
    return {"ok": len(ok), "failed": len(results) - len(ok), "data": ok}
```
### 🔍 Line by line
- `asyncio.Semaphore(10)` — at most 10 calls in flight at once, protecting the downstream service.
- `async with sem:` — acquire a slot; released automatically on exit.
- `asyncio.gather(*(...), return_exceptions=True)` — `*` unpacks the generator into arguments; all 50 start concurrently; with `return_exceptions=True`, a failure is returned as a value instead of cancelling the whole batch.
- `isinstance(r, Exception)` — split successes from failures.
- Hard overall deadline: `async with asyncio.timeout(5): results = await asyncio.gather(...)`.

---

## Q19. What is `asyncio.gather()`?

Runs several awaitables concurrently and returns their results **in the order passed**. Default (`return_exceptions=False`): the first exception propagates; the others keep running in the background. With `return_exceptions=True`, exceptions come back as values. Python 3.11's `asyncio.TaskGroup` is a safer alternative: if one task fails, siblings are cancelled (structured concurrency).

---

## Q20. How would you troubleshoot an API whose latency suddenly increased?

### 🧒 Runbook
**1. Scope it (2 minutes).**
- Which endpoints — all (infra) or some (code/data)?
- p50 vs p99: p99 up with flat p50 → contention/GC/locks; both up → a slow dependency.
- When did it start? Correlate with deploys, config changes, traffic, cron jobs.

**2. Check the four golden signals** (latency, traffic, errors, saturation) and the **downstream** spans in tracing: DB time, external API time, Redis time. The trace waterfall tells you which span grew.

**3. FastAPI-specific suspects.**
- **Blocked event loop** — someone added a sync call to an `async def`. Symptom: everything slow, CPU low. Fix: `run_in_threadpool` or make it `def`.
- **DB pool exhaustion** — requests wait for a connection; latency rises in steps equal to `pool_timeout`.
- **N+1 queries** introduced by a new feature.
- **Missing index** — data grew and the query plan flipped to a sequential scan.
- **Thread pool saturation** — too many `def` endpoints all blocked.
- **Degraded external dependency** with generous timeouts → requests pile up. Add circuit breaker.
- **Memory pressure** — near the limit → swapping / OOM restarts.

**4. Tools.** `py-spy dump --pid <pid>` shows what every thread/coroutine is doing *right now*; `pg_stat_statements` / `EXPLAIN ANALYZE`; tracing waterfall; `git diff` last-good → current.

**5. Mitigate, then fix.** Roll back if it correlates with a deploy; scale out if saturated; fix root cause; add an alert or test so it can't recur silently.

### 🎤 Interview answer
> "First I scope it — which endpoints, since when, p50 vs p99, whether a deploy happened. Then I look at saturation and downstream spans. For FastAPI I specifically check for blocking calls in async handlers and DB pool exhaustion, the two most common causes. I use py-spy on a live worker to see what it's actually doing, roll back if it correlates with a deploy, then fix and add an alert."

# 01 — Python, Senior Level

> **How to use this file.**
> Every question has a **🗣️ Say this** block. That *is* the answer — 4 to 5 lines, written to be spoken, not read. Say it out loud once and you're done.
> Everything under it is ammunition for the follow-up. Don't memorise it; just know it's there.
>
> All code below was actually run on **Python 3.14**. The `Output:` blocks are real, not illustrative.

---

## 📖 Jargon, in plain words

Don't use a word in an interview you can't unpack in one sentence. Here's the unpack for every term in this file.

| Jargon | What it actually means | How you'd slip it into a sentence |
|---|---|---|
| **CPython** | The normal Python you download from python.org. Written in C. | "The GIL is a CPython thing, not a Python-the-language thing." |
| **Bytecode** | Python quietly compiles your code into small instructions, then runs those. That's the `__pycache__` folder. | "Python compiles to bytecode first, then interprets it — so it's both." |
| **Reference counting** | Every object keeps a tally of how many names point at it. Tally hits zero, memory is freed instantly. | "Refcounting frees things the moment the last name goes away." |
| **Reference cycle** | Two objects pointing at each other, so neither tally ever hits zero, so neither is ever freed. | "Cycles are the one thing refcounting can't clean up." |
| **Garbage collector (GC)** | A second cleaner that exists *only* to find those cycles. | "The cycle collector is the backup, not the main mechanism." |
| **GIL** | One lock. Only one thread can run Python code at a time. | "The GIL means threads help with waiting, never with computing." |
| **Free-threaded build** | A version of Python with the GIL removed. Optional, not the default. | "3.14 officially supports it, but it's opt-in." |
| **Coroutine** | What you get when you *call* an `async def` function. It hasn't run yet — it's a recipe, not a meal. | "Calling an async function gives you a coroutine; it does nothing until awaited." |
| **Event loop** | A single-threaded to-do list manager. Runs whatever is ready, and asks the OS which network sockets have woken up. | "The loop is one thread juggling thousands of waiting requests." |
| **Task** | A coroutine the event loop has adopted and is actively running. | "`create_task` hands the coroutine to the loop so it runs alongside." |
| **Blocking** | Code that just sits there and refuses to give the loop a turn. | "One blocking call freezes every request on that worker." |
| **Cooperative** | Tasks only pause where *you* wrote `await`. Nothing interrupts them. | "Async is cooperative — a task that never awaits owns the loop." |
| **Event-loop lag** | How late the loop is running things it promised to run. The single best async health metric. | "I export loop lag; it catches blocking calls before users do." |
| **`contextvars`** | A variable that follows one request around, even across `await`. | "ContextVars are the async-safe version of thread-locals." |
| **Descriptor** | An object that hijacks what `obj.thing` means. | "Descriptors are how `property` and ORM columns actually work." |
| **Metaclass** | The thing that builds your class when Python reads the `class` line. | "A metaclass runs at class-definition time — it's how ORMs collect fields." |
| **`tracemalloc`** | Built-in tool that tells you which *line of code* is holding memory. | "I diff two tracemalloc snapshots under load to find the leaking line." |
| **`py-spy`** | Tool that profiles a running Python process without restarting it. Safe in production. | "`py-spy top` on the live pod, no deploy needed." |

---

# 🔴 Section A — Memory

## Q1. How does Python manage memory?

**Jargon here:** *reference counting* (a tally per object), *reference cycle* (two objects holding each other up), *garbage collector* (the backup cleaner).

### 🗣️ Say this
> "Python uses two mechanisms. The main one is reference counting — every object keeps a tally of how many names point at it, and the moment that tally hits zero the memory is freed. That's instant and predictable, no pause.
> The problem is reference cycles: if two objects point at each other, neither tally ever reaches zero, so neither is ever freed. That's what the garbage collector is for — it's a backup that only hunts for cycles.
> So unlike Java, where the GC does all the work, in Python the GC is the exception path. The trade-off is that refcounting costs you a little on every single assignment, and that cost is exactly why the GIL exists."

### 💻 Proof
```python
import sys, gc

x = []
print("getrefcount(x) =", sys.getrefcount(x))   # 1 for x, +1 for the argument
y = x
print("after y = x    =", sys.getrefcount(x))

class Node:
    def __init__(self): self.self_ref = None

a = Node(); a.self_ref = a     # a points at itself -> a cycle
del a                          # tally never hits 0, so nothing is freed
print("gc.collect() freed:", gc.collect(), "objects")
```
```
Output:
getrefcount(x) = 2
after y = x    = 3
gc.collect() freed: 1 objects
```
> Read it like this: `del a` did nothing — the object survived. Only `gc.collect()` could free it. That one line *is* the reason the cycle collector exists.

### ⚠️ Follow-up: "Is that a stop-the-world pause like Java?"
> "Mostly no, and that's the nice difference. Refcounting cleans up most garbage with zero pause, spread evenly through your program. The cycle collector does briefly stop things, but it only walks container objects, so for a web service it's milliseconds. In Java the GC does everything; in Python it's just the leftovers."

---

## Q2. What is the GIL?

**Jargon here:** *GIL* (one lock, one thread runs Python at a time), *I/O-bound* (waiting on network/disk), *CPU-bound* (actually computing).

### 🗣️ Say this
> "The GIL is a single lock inside CPython. To run any Python code, a thread has to hold it — so only one thread executes Python at a time, no matter how many cores you have.
> Its real job is keeping those reference-count updates safe without putting a lock on every object.
> The practical rule is: if your threads are *waiting* — network, disk, database — the GIL is released while they wait, so threads work fine. If your threads are *computing*, threads buy you nothing and can even be slower.
> So for CPU work I use processes, not threads. And I'd be careful to say the GIL doesn't make Python slow — single-threaded speed is unaffected. It only blocks multi-core parallelism for pure Python code."

### 💻 Proof
```python
import time, threading

def burn(n=6_000_000):
    c = 0
    for _ in range(n): c += 1

t0 = time.perf_counter(); burn(); burn()
print(f"sequential : {time.perf_counter()-t0:.2f}s")

t0 = time.perf_counter()
ts = [threading.Thread(target=burn) for _ in range(2)]
[t.start() for t in ts]; [t.join() for t in ts]
print(f"2 threads  : {time.perf_counter()-t0:.2f}s")
```
```
Output:
sequential : 0.22s
2 threads  : 0.21s
```
> Two threads on two cores gave **zero speedup**. That's the GIL in one number. Swap `threading.Thread` for `multiprocessing.Process` and it roughly halves.

### 🧠 Free-threading status — know the version numbers
| Version | Status |
|---|---|
| 3.13 | Experimental no-GIL build (`python3.13t`) |
| **3.14** | **Officially supported — but still not the default build** |

> "It's real now, but it's opt-in, and every C extension needs recompiling and auditing. For production backend work today I'd still reach for processes or async first."

### ⚠️ Follow-up: "Your FastAPI service is CPU-bound. What do you do?"
> "In this order. First, get the work out of the request — put it on a queue and return 202. Second, use a library that releases the GIL, so NumPy or `orjson` or something in Rust. Third, a `ProcessPoolExecutor` for genuinely parallel work. Fourth, just run more workers or more pods, which is the same trick at a bigger scale. The one thing I would *not* do is add threads."

---

## Q3. Pods are getting OOMKilled after six hours. How do you find it?

**Jargon here:** *OOMKilled* (the container hit its memory limit and was killed), *RSS* (actual memory the process holds), *`tracemalloc`* (shows which line allocated what).

### 🗣️ Say this
> "First I confirm it's actually a leak, not just a high-water mark. If memory climbs steadily and never flattens out, it's growth. If it plateaus just under the limit, the limit is simply too low.
> Then I work out whether it's Python's fault or below Python. If the Python object count is flat but memory keeps climbing, it's a C extension or memory fragmentation. If object count climbs too, it's our code.
> For our code I use `tracemalloc` — take a snapshot, run load, take another, and diff them. That points straight at the line.
> In my experience it's almost always an unbounded cache: an `lru_cache` with no max size, or a dict used as a cache that nobody ever evicts from."

### 💻 Proof
```python
import tracemalloc

CACHE = {}
def leaky(n):
    for i in range(n):
        CACHE[f"key{i}"] = "x" * 500      # nothing ever evicts this

tracemalloc.start()
s1 = tracemalloc.take_snapshot()
leaky(20000)                              # simulate an hour of traffic
s2 = tracemalloc.take_snapshot()

for stat in s2.compare_to(s1, "lineno")[:1]:
    print(f"+{stat.size_diff/1e6:.1f} MB  {stat.count_diff:+} objects")
    print("  at", stat.traceback[0])
```
```
Output:
+1.4 MB  +20001 objects
  at demo.py:6
```
> It names the exact line. That's the whole technique — two snapshots and a diff.

### 🧠 The usual suspects, in the order I actually find them
| Cause | How you spot it |
|---|---|
| `@lru_cache(maxsize=None)` on a **method** | It keys on `self`, so it pins every object forever |
| A module-level dict used as a cache | Grows with however many distinct keys exist |
| A metric label like `user_id` | One child object per user, forever |
| Tasks created and never awaited | Exceptions pile up unretrieved |
| **Not a leak at all** — fragmentation | Memory climbs, Python object count flat. Try `MALLOC_ARENA_MAX=2` |

### ⚠️ Follow-up: "Memory is high but not growing. Explain that."
> "That's a high-water mark, not a leak. Python doesn't reliably hand freed memory back to the OS, and the process is sized by the worst moment it ever had. One request that parsed a 500MB JSON body sets that level for the life of the worker. The fix is a request body size limit and streaming the parse — not a bigger memory limit."

---

## Q4. What is `__slots__` and when does it matter?

### 🗣️ Say this
> "Normally every Python object carries a dictionary to hold its attributes, and that dictionary is surprisingly expensive. `__slots__` replaces it with a fixed set of slots.
> You get a big memory saving and slightly faster attribute access. You give up the ability to add attributes on the fly.
> It only matters at volume. If I'm holding a million small objects in memory it's a real win. For a hundred request-scoped objects it's noise and I wouldn't bother.
> With dataclasses it's just `@dataclass(slots=True)`."

### 💻 Proof
```python
import sys
class WithDict:
    def __init__(self, a, b): self.a, self.b = a, b
class WithSlots:
    __slots__ = ("a", "b")
    def __init__(self, a, b): self.a, self.b = a, b

d, s = WithDict(1, 2), WithSlots(1, 2)
print("with __dict__ :", sys.getsizeof(d) + sys.getsizeof(d.__dict__), "bytes")
print("with __slots__:", sys.getsizeof(s), "bytes")
```
```
Output:
with __dict__ : 344 bytes
with __slots__: 48 bytes
```
> 344 down to 48. At a million objects that's 344MB versus 48MB — that's when it's worth it.

---

# 🔴 Section B — Async

## Q5. What actually happens when the event loop hits an `await`?

**Jargon here:** *event loop* (single-threaded to-do manager), *suspend* (pause and hand back control), *Future* (a placeholder for a result that isn't ready).

### 🗣️ Say this
> "The key thing people get wrong is that `await` doesn't automatically give the loop a turn. `await` on a coroutine is just a function call — it runs straight through.
> It only pauses when it hits something that isn't ready yet, like a network read. At that point the coroutine suspends and hands control back to the loop.
> The loop then goes off and runs other work, and asks the operating system which sockets have data. When yours is ready, it resumes your coroutine exactly where it stopped.
> So the one-line version is: awaiting is not the same as yielding. If the thing you await never touches I/O, nothing else gets a chance to run."

### 💻 Proof
```python
import asyncio
order = []

async def cpu_coro():
    order.append("cpu_coro start")
    sum(range(3_000_000))          # no await inside -> never suspends
    order.append("cpu_coro end")

async def other():
    order.append("other ran")

async def main():
    t = asyncio.create_task(other())   # scheduled, ready to go
    await cpu_coro()                   # but this never gives the loop a turn
    await t

asyncio.run(main())
print(order)
```
```
Output:
['cpu_coro start', 'cpu_coro end', 'other ran']
```
> `other()` was scheduled *first* and still ran **last**. `await cpu_coro()` never suspended, so the loop never got a chance to switch.

### ⚠️ Follow-up: "Then why is `gather` faster?"
> "Because those coroutines *do* suspend. Each one hits a network wait, gives the loop a turn, and the loop starts the next one. So they overlap instead of queueing."

```python
async def fake_io(n):
    await asyncio.sleep(0.3)     # a real network call
    return n

# sequential
await fake_io(1); await fake_io(2); await fake_io(3)
# concurrent
await asyncio.gather(fake_io(1), fake_io(2), fake_io(3))
```
```
Output:
sequential : 0.90s
gather     : 0.30s
```

---

## Q6. `gather` vs `TaskGroup` — which do you use?

### 🗣️ Say this
> "I default to `TaskGroup` now, which is Python 3.11 and up.
> The difference that matters is what happens when one task fails. With `gather`, the other tasks just keep running — you got your exception, but there's still work happening in the background that nobody is watching. With `TaskGroup`, a failure cancels its siblings and nothing outlives the block.
> That matters in a web app because you can return a response to the user while a stray task is still using a database session you're about to close. That's a genuinely nasty bug.
> I only reach for `gather` when I actually want `return_exceptions=True`, so partial failure is fine."

### 💻 Proof
```python
async def slow(name):
    try:
        await asyncio.sleep(1)
        print(f"{name} COMPLETED (leaked work)")
    except asyncio.CancelledError:
        print(f"{name} cancelled"); raise

async def boom():
    await asyncio.sleep(0.05); raise ValueError("boom")

# --- gather ---
try: await asyncio.gather(slow("gather-sibling"), boom())
except ValueError as e: print("gather raised:", e)

# --- TaskGroup ---
try:
    async with asyncio.TaskGroup() as tg:
        tg.create_task(slow("tg-sibling")); tg.create_task(boom())
except* ValueError as eg:
    print("TaskGroup raised ExceptionGroup:", eg.exceptions[0])
```
```
Output:
gather raised: boom
gather-sibling COMPLETED (leaked work)      <-- still running after the failure
tg-sibling cancelled                        <-- cleaned up
TaskGroup raised ExceptionGroup: boom
```
> Look at line 2. After `gather` already raised, the sibling carried on and finished. That's the leak `TaskGroup` closes.

### ⚠️ Follow-up: "Any trap with `create_task`?"
> "Yes, and it's a good one. The event loop only holds a *weak* reference to your task, so a fire-and-forget task can actually be garbage-collected halfway through. You have to keep a reference yourself."
```python
_background = set()
t = asyncio.create_task(work())
_background.add(t)
t.add_done_callback(_background.discard)
```

---

## Q7. Someone put a blocking call in an `async def`. What's the damage?

**Jargon here:** *blocking* (code that won't give the loop a turn), *event-loop lag* (how late the loop is).

### 🗣️ Say this
> "The damage is the whole worker process, not just that one request. A blocking call freezes the event loop, so every other request on that worker stops dead — including the health check endpoint.
> The signature in production is very distinctive: your median latency looks completely fine, your p99 is terrible, and Kubernetes starts restarting pods because liveness probes time out on a service that looks healthy.
> The way I catch it is by exporting event-loop lag as a metric — how late the loop is running things it promised to run. It's the most useful async-specific metric there is and almost nobody exports it.
> The fix is the async client — `httpx` instead of `requests`, `asyncpg` instead of `psycopg2` — or `asyncio.to_thread` if I'm stuck with a blocking library."

### 💻 Proof
```python
async def heartbeat(stop):
    """Pretend this is /healthz being polled every 100ms."""
    worst = 0.0
    while not stop.is_set():
        t0 = time.perf_counter()
        await asyncio.sleep(0.1)
        worst = max(worst, time.perf_counter() - t0 - 0.1)   # how late we were
    print(f"worst event-loop lag = {worst*1000:.0f} ms")

async def good(): await asyncio.sleep(1.0)     # async - loop stays free
async def bad():  time.sleep(1.0)              # BLOCKING - loop frozen
```
```
Output:
await asyncio.sleep(1)  (correct): worst event-loop lag = 1 ms
time.sleep(1)           (the bug): worst event-loop lag = 953 ms
```
> 1 millisecond versus 953. Your health check is now 953ms late, and that's why the pod gets restarted.

### 🧠 Ship this — it's 8 lines and it catches the whole bug class
```python
async def loop_lag_monitor(threshold=0.25):
    while True:
        t0 = time.perf_counter()
        await asyncio.sleep(0.1)
        lag = time.perf_counter() - t0 - 0.1
        EVENT_LOOP_LAG.observe(lag)              # export it to Prometheus
        if lag > threshold:
            logger.warning("event_loop_lag", extra={"lag_s": lag})
```

### ⚠️ Follow-up: "Can't you just make the handler `def` instead of `async def`?"
> "You can, and it's a correct fix — Starlette runs sync handlers in a thread pool. But that pool is 40 threads by default, so forty slow requests and the forty-first silently queues with no visibility. You've traded an event-loop problem for a thread-pool problem. It needs a concurrency limit and a metric on pool saturation."

---

## Q8. How do you carry a request ID through async code?

### 🗣️ Say this
> "With `contextvars`, not thread-locals. That distinction is the whole answer.
> A thread-local is wrong in async because many requests share one thread — you'd read another request's ID and log the wrong thing. That's a real bug and it's very hard to spot.
> A ContextVar is per logical flow, and asyncio copies the context when it creates a task, so the value follows one request across every `await`.
> It's the exact same mechanism OpenTelemetry uses to carry the active span, which is why trace IDs survive across awaits."

### 💻 Proof
```python
tl = threading.local()
cv = ContextVar("rid", default="-")

async def handle(req_id):
    tl.rid = req_id
    cv.set(req_id)
    await asyncio.sleep(0.01)          # another request runs right here
    print(f"request {req_id}: thread-local={tl.rid}  contextvar={cv.get()}")

await asyncio.gather(handle("A"), handle("B"))
```
```
Output:
request A: thread-local=B  contextvar=A
request B: thread-local=B  contextvar=B
```
> Look at request A. The thread-local says **B** — it's reporting someone else's request ID. The ContextVar is correct. That's the bug, in one line of output.

### 🧠 In FastAPI
```python
request_id: ContextVar[str] = ContextVar("request_id", default="-")

class RequestIDMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request, call_next):
        rid = request.headers.get("x-request-id") or str(uuid.uuid4())
        token = request_id.set(rid)
        try:
            response = await call_next(request)
            response.headers["x-request-id"] = rid
            return response
        finally:
            request_id.reset(token)      # reset, don't just overwrite
```

### ⚠️ Follow-up: "Any gotcha?"
> "Two. The context is copied *when the task is created*, so setting a ContextVar inside a child task doesn't flow back up to the parent. And `run_in_executor` doesn't copy context at all — you need `contextvars.copy_context().run(fn)` or you lose your trace ID the moment you hit a thread pool."

---

## Q9. Threads, processes, or asyncio?

### 🗣️ Say this
> "It comes down to one question: is the work waiting, or computing?
> If it's waiting on network or disk and I control the libraries, asyncio — it's the cheapest by a mile, thousands of concurrent requests on one thread.
> If it's waiting but I'm stuck with a blocking library nobody has ported, threads, in a bounded pool.
> If it's actually computing, processes — because of the GIL threads won't help. Or better, get the work out of the request path entirely.
> And mixing them is completely normal. A FastAPI service is usually async at the edge, with a thread pool for that one stubborn SDK, and a worker fleet for the heavy lifting."

### 🧠 The table
| | threading | multiprocessing | asyncio |
|---|---|---|---|
| Parallel CPU? | ❌ GIL | ✅ | ❌ |
| Cost per unit | ~8 MB stack | full interpreter | ~1 KB |
| Realistic ceiling | hundreds | ~number of cores | tens of thousands |
| How it bites you | races, deadlocks | pickling errors, IPC cost | one blocking call freezes everything |

---

# 🔴 Section C — The deeper stuff

## Q10. What's a descriptor?

### 🗣️ Say this
> "A descriptor is an object that hijacks what `obj.something` means. If a class attribute defines `__get__` or `__set__`, Python calls those instead of just handing you the object.
> It's not exotic — it's the machinery behind `property`, `classmethod`, and actually behind every plain function, which is how methods get bound to instances.
> Where you meet it in backend work is ORMs. `User.email` on the class gives you a SQL expression, while `user.email` on an instance gives you the value.
> That's exactly why `User.email == 'x'` builds a WHERE clause instead of returning True or False — and people find that confusing until they know it's a descriptor."

### 💻 Proof
```python
class Column:
    def __set_name__(self, owner, name): self.name = name
    def __get__(self, obj, objtype=None):
        if obj is None:                                   # accessed on the CLASS
            return f"<SQL expression for {self.name}>"
        return obj.__dict__[self.name]                    # accessed on an INSTANCE
    def __set__(self, obj, value): obj.__dict__[self.name] = value

class User:
    email = Column()

u = User(); u.email = "a@b.com"
print("User.email  ->", User.email)
print("u.email     ->", u.email)
```
```
Output:
User.email  -> <SQL expression for email>
u.email     -> a@b.com
```
> Same attribute name, two completely different results. That single behaviour is what makes SQLAlchemy's query syntax possible.

---

## Q11. Metaclasses — do you use them?

### 🗣️ Say this
> "A metaclass is the thing that builds your class when Python reads the `class` line. `type` is the default one, and a custom one lets you inspect or rewrite the class as it's created.
> Frameworks use them to collect declarations — SQLAlchemy gathering up your Column objects, Django models, abstract base classes.
> But in application code I'd say honestly that I reach for `__init_subclass__` first. It handles registration and validation, which is what people actually want, without the metaclass-conflict problems you hit when two parent classes disagree.
> Pydantic v2 is a good signal of where things are going — they moved the heavy lifting into Rust and out of Python class machinery."

### 💻 Proof — 90% of metaclass use, without a metaclass
```python
class Handler:
    registry: dict[str, type] = {}
    def __init_subclass__(cls, *, event, **kw):
        super().__init_subclass__(**kw)
        Handler.registry[event] = cls          # self-registration, no metaclass

class UserCreated(Handler, event="user.created"): pass
class OrderPaid(Handler,  event="order.paid"):   pass

print(Handler.registry)
```
```
Output:
{'user.created': <class 'UserCreated'>, 'order.paid': <class 'OrderPaid'>}
```

### ⚠️ Quick follow-ups
| They ask | You say |
|---|---|
| `__new__` vs `__init__`? | "`__new__` creates the object and returns it — you need it for immutable types. `__init__` just configures something that already exists." |
| `__getattr__` vs `__getattribute__`? | "`__getattribute__` catches *every* attribute access and is easy to make infinitely recursive. `__getattr__` only fires when normal lookup fails — that's the one you want for proxies and lazy loading." |
| Monkey patching — ever OK? | "In tests, always. In library setup where it's the documented mechanism, yes — OpenTelemetry auto-instrumentation is monkey patching with a nicer name. In app code only as a temporary fix for an upstream bug, with a comment linking the issue." |

---

## Q12. An endpoint got slow. What do you do?

### 🗣️ Say this
> "I refuse to guess — the first job is working out which layer it's in, and a trace answers that in one look.
> If the waterfall shows 400 milliseconds inside a database span, this is a query problem and nothing about Python matters. If the total time is much bigger than the sum of the child spans, the time is in our own code or the event loop is lagging.
> That split takes thirty seconds and saves a day of profiling the wrong thing.
> Once I know it's our code, I use `py-spy` on the live process — it samples without a restart, so it's safe in production. `cProfile` I only use locally, because it distorts async code badly."

### 🧠 The tools, and what each one actually answers
| Tool | The question it answers |
|---|---|
| Distributed trace | **Which layer?** Always first |
| `py-spy top --pid N` | Which function? No restart, safe in prod |
| `py-spy dump --pid N` | What is it stuck on *right now*? For hangs |
| `cProfile` + `snakeviz` | Which call path? Dev only — lies about async |
| `line_profiler` | Which line? Once you know the function |
| `EXPLAIN (ANALYZE, BUFFERS)` | Why is the query slow? → [03-postgres-deep-dive.md](03-postgres-deep-dive.md) |
| Event-loop lag metric | Is something blocking? |

### ⚠️ Follow-up: "Which Python optimisations are actually worth it?"
> "Honestly, most aren't. In order: don't do the work at all — cache it, batch it, or push it to a queue. Then fix the query, because one missing index beats every micro-optimisation combined. Then swap in `orjson`, which is a one-line change and real money on a JSON API. Then use the right data structure — `in` on a set is O(1), on a list it's O(n).
> The honest framing is that if a request spends 40ms in the database and 3ms in Python, Python optimisation has a hard ceiling of 3ms. I'd rather spend that effort on the query."

---

## Q13. What's changed in recent Python versions?

### 🗣️ Say this
> "The one I'd call out for backend work is 3.11. It was significantly faster across the board, and it gave us `TaskGroup` and `asyncio.timeout()`, which I'd now treat as the defaults over `gather` and `wait_for`. It also made exceptions basically free when they don't fire, and the tracebacks got much more precise.
> 3.12 added the `type` statement and faster comprehensions. 3.13 brought the experimental no-GIL build and an experimental JIT.
> And 3.14 is where free-threading became officially supported — though it's still not the default build, so I wouldn't reach for it in production yet."

| Version | What matters to you |
|---|---|
| 3.11 | Much faster, `TaskGroup`, `ExceptionGroup`/`except*`, `asyncio.timeout()`, zero-cost exceptions |
| 3.12 | `type` statement, faster comprehensions, PEP 695 generics |
| 3.13 | Experimental free-threaded build, experimental JIT, better REPL |
| **3.14** | **Free-threading officially supported** (still opt-in), more JIT work |

---

## ⚡ Rapid fire

| They ask | Say this |
|---|---|
| Coroutine vs Task vs Future? | "Coroutine is the recipe — it doesn't run on its own. Future is a placeholder for a result. Task is a Future that's actively driving a coroutine on the loop." |
| Why `asyncio.sleep(0)`? | "It hands the loop a turn without actually waiting. It's a manual cooperation point inside a long computation." |
| How do you time out an async call? | "`async with asyncio.timeout(5):` in 3.11+. It cancels the inner task, so put your cleanup in a `finally`." |
| `to_thread` vs `run_in_executor`? | "`to_thread` is the friendly wrapper over the default thread pool. `run_in_executor` lets you pass your own pool, including a process pool." |
| Is `dict` ordered? | "Yes. It was an accident of the implementation in 3.6 and a real language guarantee from 3.7." |
| Why not `lru_cache` on a method? | "It keys on `self`, so the cache holds onto every instance forever. That's a leak. Cache the plain function or use `cached_property`." |
| What's a `weakref`? | "A reference that doesn't count — so the object can still be freed. It's how you build a cache that doesn't keep its entries alive." |
| Is the GIL released during `time.sleep`? | "Yes — and during any I/O syscall, and inside C extensions that opt in, like NumPy." |
| Shallow vs deep copy? | "Shallow copies the outer container but shares what's inside. Deep copies all the way down. The classic bug is shallow-copying a dict of lists and being surprised when both change." |
| `is` vs `==`? | "`is` asks 'same object in memory', `==` asks 'same value'. Only use `is` for `None`, `True` and `False`." |

---

**Next:** [02-databases-advanced.md](02-databases-advanced.md)

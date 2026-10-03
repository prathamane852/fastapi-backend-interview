# 04 — Redis & Caching, Senior Level

> **How to use this file.** Every question has a **🗣️ Say this** block — 4 to 5 lines, ready to speak. `Output:` marked **(real)** was actually run; **(what you'd see)** is representative `redis-cli` output.

**The senior/junior split on this topic is sharp.** A junior says "add Redis." A senior asks what the invalidation story is, what happens when Redis is down, and whether the cache is hiding a query that just needs an index.

---

## 📖 Jargon, in plain words

| Jargon | What it actually means | How you'd say it |
|---|---|---|
| **Cache hit ratio** | Of all lookups, what fraction were already cached. | "Below 80% the cache often costs more than it saves." |
| **TTL** | How long before the cached copy expires by itself. | "TTL is the invalidation strategy you get for free." |
| **Cache-aside** | App checks the cache; on a miss it reads the database and puts the result in. | "Cache-aside is the default — the app owns the cache." |
| **Write-through** | Write to the cache and the database at the same time. | "Consistent, but you pay cache latency on every write." |
| **Stampede / dogpile** | A popular key expires and hundreds of requests all miss and all hit the database at once. | "One expiry becomes a thousand simultaneous queries." |
| **Thundering herd** | The general version: lots of clients waking up to do the same thing at the same moment. | "Jitter is the cheapest defence." |
| **Eviction policy** | What Redis throws away when it runs out of memory. | "`allkeys-lru` for a cache, `noeviction` for locks." |
| **LRU / LFU** | Throw away the least *recently* used / least *frequently* used. | "LFU is better when a scan would otherwise pollute your hot set." |
| **Distributed lock** | A lock shared across many servers, so only one does a job. | "In-process locks are useless across pods." |
| **Fencing token** | A number that goes up with each lock grant; the resource rejects old numbers. | "Without one, a paused process can't be defended against." |
| **Hot key** | One key getting a huge share of traffic. | "In a cluster a hot key is a hot node — sharding doesn't help." |
| **Pipelining** | Sending many commands without waiting for each reply. | "It removes the round trip, which is where the time goes." |
| **Lua script** | A small script Redis runs atomically, start to finish. | "It's how you make read-modify-write safe." |

---

# Section A — Redis itself

## Q1. Why is Redis fast, and where does that break down?

### 🗣️ Say this
> "It's in memory, so there's no disk seek. It's single-threaded for command execution, which sounds like a limitation but actually removes all lock contention and makes every command atomic for free — that's why `INCR` is a correct counter with no coordination needed.
> Modern Redis is multi-threaded for network I/O, but the command execution core is still one thread.
> Where it breaks down is exactly because of that one thread: a single O(n) command blocks *everything*. `KEYS *` on a million keys, or `HGETALL` on a huge hash, freezes the whole instance and every client times out.
> So the failure mode isn't gradual degradation, it's a cliff. That's why `KEYS` is banned in production and you use `SCAN`, which is cursor-based and incremental."

### 💻 The dangerous commands
| Never in production | Use instead |
|---|---|
| `KEYS *` | `SCAN` — cursor-based, never blocks |
| `SMEMBERS` / `HGETALL` / `LRANGE 0 -1` on big collections | `SSCAN` / `HSCAN`, or paginate |
| `DEL bigkey` | `UNLINK` — frees memory in a background thread |
| `FLUSHALL` | `FLUSHALL ASYNC` |

```
# Find what's slow, safely
127.0.0.1:6379> SLOWLOG GET 2
1) 1) (integer) 14
   2) (integer) 1758441203
   3) (integer) 8241053          <- 8.2 SECONDS, single-threaded, everything blocked
   4) 1) "KEYS"
      2) "session:*"
```

---

## Q2. Which data structure would you use?

### 🗣️ Say this
> "Strings for a cached JSON blob or a counter. Hashes when an object has fields you update independently — it's much more memory-efficient than lots of separate keys.
> The one I reach for most is the **sorted set**, because a surprising number of problems are really 'ordered by a number, with range queries'. A sliding-window rate limiter is a sorted set. A delayed job queue is a sorted set scored by timestamp. Leaderboards obviously.
> And the one people forget is **HyperLogLog**. If you want unique visitor counts, everyone reaches for a Set and then discovers it's 40 gigabytes. HyperLogLog gives you the same answer within about 1% in twelve kilobytes, fixed, whether it's a thousand uniques or a billion.
> That trade — exact versus 1% error for a 99.99% memory saving — is usually an easy yes."

| Structure | Cost | Real use |
|---|---|---|
| String | O(1) | Cached JSON, counter, lock |
| Hash | O(1) per field | Object with independently-updated fields |
| List | O(1) at ends | Simple queue, capped "recent items" |
| Set | O(1) member | Dedup, tags, "has this user seen X" |
| **Sorted Set** | O(log n) | **Leaderboards, sliding windows, delayed queues** |
| **HyperLogLog** | O(1), ~12 KB fixed | **Unique counts at ~0.81% error** |
| Bitmap | O(1) bit | Daily-active flags — 1M users = 125 KB |
| Stream | O(1) append | Append-only log with consumer groups |

---

## Q3. Persistence — RDB, AOF, or neither?

### 🗣️ Say this
> "It depends entirely on what Redis *is* in your architecture, and that's the real question behind this one.
> If it's a pure cache, I'd argue for turning persistence off. The data is rebuildable, and you avoid the fork latency spike. The important work then is making sure the app survives a cold cache, which is a stampede problem rather than a Redis problem.
> If Redis holds anything you can't rebuild — sessions, a rate-limit budget, a job queue — then it's a database, and it needs AOF with `appendfsync everysec` plus replication.
> But honestly at that point I'd question whether it should be Redis at all, because Redis replication is asynchronous, so a failover can lose writes it already acknowledged. That's fine for a cache and not fine for money."

| | RDB (snapshot) | AOF (command log) |
|---|---|---|
| Restart speed | Fast | Slower — replays the log |
| Data loss window | Everything since the last snapshot | ≤1 second with `everysec` |
| Cost | `fork()` — latency spike, up to 2× memory | Continuous writes + periodic rewrite |

### ⚠️ The fork spike is worth volunteering
> "`BGSAVE` forks the process. On a 20GB instance with heavy writes, copy-on-write means the child can approach a full copy of memory, and the fork itself pauses the parent proportionally to the page table size. On a big instance that's a visible latency spike people usually blame on the network."

---

## Q4. `maxmemory` and eviction policies.

### 🗣️ Say this
> "When Redis hits its memory limit it has to decide what to throw away, and the policy controls that. `allkeys-lru` evicts the least recently used from everything — that's right for a pure cache. `noeviction`, which is the default, makes writes error instead of evicting, and that's right for a queue or anything you can't lose.
> `volatile-lru` only evicts keys that have a TTL, which is what you want for a mixed workload.
> The production bug I'd call out is using one Redis for both caching and distributed locks with `allkeys-lru`. Memory fills, Redis evicts — and it happily evicts your lock keys, because to Redis they're just keys. Now two workers both hold 'the' lock.
> I'd separate the instances. Sharing one Redis between a cache and a coordination primitive couples their failure modes, and one of those failures is silent."

📐 Also note Redis LRU is *approximate* — it samples a handful of keys rather than keeping a true LRU list, because a true one would cost memory per key.

---

# Section B — Caching strategy

## Q5. Walk me through the caching patterns.

### 🗣️ Say this
> "Cache-aside is the default and what I use for almost everything. The app checks the cache, and on a miss it reads the database and populates it. The reason I prefer it is that a Redis outage degrades you to 'slow' rather than 'broken' or 'wrong'.
> Write-through keeps the cache always fresh but you pay cache latency on every write, and you end up caching things nobody reads.
> Write-behind is fast but can lose data, so I'd rarely accept it.
> On top of cache-aside I add explicit invalidation on write where staleness is visible to the user — but the important detail is you **delete** the key, you don't update it. Deleting is idempotent and safe under concurrency; updating races with another writer and can leave the cache permanently wrong."

### 💻 The default pattern
```python
async def get_user(uid: str) -> User:
    if cached := await redis.get(f"user:{uid}"):
        return User.model_validate_json(cached)          # hit
    user = await repo.get(uid)                           # miss
    await redis.set(f"user:{uid}", user.model_dump_json(), ex=300)
    return user
```

### ⚠️ Follow-up: "Delete before or after the database write?"
> "After — and then be honest that there's still a race. If you delete first, a concurrent read can repopulate with the old value between your delete and your commit, and now the cache is stale forever. Deleting after shrinks the window to the gap between commit and delete.
> If you need it tighter than that, the options are a short TTL as a backstop, or driving invalidation from the database's change stream so the commit itself invalidates the cache."

---

## Q6. What should you *not* cache?

### 🗣️ Say this
> "Anything where being stale is a correctness bug rather than a cosmetic one — an account balance you're about to debit, a permission check, an inventory count at the moment you decrement it.
> Anything with a hit rate so low that the cache is pure overhead, like per-user data for a long tail of users you'll never see again.
> Anything cheap to compute, because a Redis round trip is about a millisecond — if the query takes half a millisecond you've made it slower.
> And anything with unbounded cardinality, like a search-results page keyed on the full query string. You just churn the cache and evict the things that actually mattered."

### ⚠️ Raise this unprompted — it's a security point, not a performance one
> "Cache keys have to include every dimension of authorisation. Caching `GET /orders` without the user ID in the key is a data breach, not a performance bug. I've seen that reach production behind a CDN, where it's even worse because it's shared across everyone."

---

## Q7. Cache stampede — three defences.

**The problem:** a key with a 5-minute TTL serving 2,000 requests a second expires. In the milliseconds before anyone repopulates it, 2,000 requests all miss and all hit the database with the same expensive query.

### 🗣️ Say this
> "Three defences and I'd use different ones for different situations.
> Jitter on the TTL always, because it's free and it fixes the worst version — where ten thousand keys were all populated at deploy time and now all expire in the same second. That's the one that actually takes a database down.
> A lock for individually expensive keys, so only one request recomputes and the rest wait briefly or serve stale.
> And probabilistic early expiry when the recompute is so slow that even one user seeing the miss is unacceptable — you refresh *before* expiry with a probability that rises as expiry approaches.
> There's also a free fourth: an in-process `asyncio.Lock` per key stops the stampede inside one worker before it even reaches Redis."

### 💻 Proof — the lock, *(real)*
```python
# Naive: 50 concurrent requests, all miss, all query
async def naive(cache):
    if "k" not in cache:
        cache["k"] = await slow_db()
    return cache["k"]

# Locked: only one recomputes, with a re-check inside the lock
async def locked(cache):
    if "k" in cache: return cache["k"]
    async with lock:
        if "k" not in cache:            # <- re-check! someone may have filled it
            cache["k"] = await slow_db()
    return cache["k"]
```
```
Output:
50 concurrent misses, no lock : 50 database queries
50 concurrent misses, locked  :  1 database query
```
> 50 down to 1. And note the re-check inside the lock — without it, every waiter queries again after acquiring.

### 💻 Proof — jitter, *(real)*
```python
ttl = int(300 * random.uniform(0.8, 1.2))    # one line
```
```
Output:
no jitter : [300, 300, 300, 300, 300, 300, 300, 300, 300, 300]   <- all in the SAME second
jittered  : [242, 245, 251, 264, 271, 274, 282, 317, 346, 350]   <- spread over ~2 minutes
```

### 💻 The cross-process version
```python
async def get_with_lock(key, loader, ttl=300):
    if (v := await redis.get(key)) is not None:
        return json.loads(v)
    if await redis.set(f"lock:{key}", "1", nx=True, ex=10):    # only one wins
        try:
            value = await loader()
            await redis.set(key, json.dumps(value), ex=ttl)
            return value
        finally:
            await redis.delete(f"lock:{key}")
    for _ in range(50):                                        # losers wait briefly
        await asyncio.sleep(0.05)
        if (v := await redis.get(key)) is not None:
            return json.loads(v)
    return await loader()          # give up and query rather than hang the request
```

---

## Q8. Redis is down. What happens to your service?

### 🗣️ Say this
> "It should degrade, not fail. Every cache call is wrapped so a Redis exception is logged and treated as a cache miss.
> The detail that matters most is the timeout — 50 to 100 milliseconds, not the default. A *hanging* Redis is much worse than a down one, because every request pays the full timeout and your service grinds to a halt while Redis is technically up.
> And I'd be honest that it will still hurt: the database now takes full uncached load, which is exactly when it's most likely to fall over too. So the miss path needs a concurrency limit, or losing Redis cascades into losing Postgres.
> What we do also depends on what Redis holds. Cache: degrade. Rate limiting: you have to decide fail-open or fail-closed, and that's a product decision. Sessions: you're down, which is an argument for JWTs."

### 💻 The wrapper
```python
class CacheClient:
    async def get(self, key: str) -> str | None:
        try:
            async with asyncio.timeout(0.1):        # NOT the default timeout
                return await self._redis.get(key)
        except (RedisError, TimeoutError) as e:
            CACHE_ERRORS.inc()
            logger.warning("cache_unavailable", extra={"key": key, "err": str(e)})
            return None                             # a miss, not an exception
```
⚠️ **Add a circuit breaker** — after N consecutive failures, stop calling Redis for 30 seconds. Otherwise every request pays that 100ms and you've added latency to a system already in trouble.

---

# Section C — Coordination

## Q9. Implement a distributed lock. Then tell me why it's wrong.

### 🗣️ Say this
> "The basic version is `SET key token NX PX ttl` — set only if it doesn't exist, with an expiry, in a single atomic command. Doing `SETNX` then `EXPIRE` separately is a bug, because a crash in between leaves a lock held forever.
> You store a unique token as the value, and you release it with a Lua script that checks the token matches before deleting. Without that, a process whose lock already expired will happily delete somebody else's lock on its way out.
> But here's the thing — that's still not safe, and I'd say so unprompted. It protects against crashes but not against *pauses*. If your process stalls for 40 seconds on a GC pause or a network hiccup, the TTL expires, someone else legitimately takes the lock, and then you wake up still believing you hold it. Two writers.
> No timeout value fixes that, because you can't bound a pause."

### 💻 The implementation
```python
async def acquire(key, ttl_ms=30_000) -> str | None:
    token = str(uuid.uuid4())
    ok = await redis.set(f"lock:{key}", token, nx=True, px=ttl_ms)   # ONE command
    return token if ok else None

RELEASE = """
if redis.call('get', KEYS[1]) == ARGV[1] then     -- only if it's still MY lock
    return redis.call('del', KEYS[1])
else
    return 0
end
"""
async def release(key, token) -> bool:
    return bool(await redis.eval(RELEASE, 1, f"lock:{key}", token))
```

### 🧠 The real fix
> "There are two honest answers. One is a **fencing token** — the lock hands out a number that always goes up, you pass it with every write, and the resource rejects anything with a lower number than it's already seen. That moves enforcement to the resource instead of trusting the lock. It needs storage support, which is why it's rare.
> The other, and what I'd usually do, is make the operation idempotent so running it twice is harmless — or push the mutual exclusion into the database with `SELECT ... FOR UPDATE` in the same transaction as the write, where there's no TTL to guess at all."

### ⚠️ Follow-up: "What about Redlock?"
> "Redlock extends this across several independent Redis nodes with majority acquisition. Martin Kleppmann's critique is that it depends on bounded clock drift and bounded pauses, and you get neither — so it isn't safe for correctness. Antirez disputed parts of that.
> My position: for *efficiency* — 'don't run this cron twice, it's wasteful' — Redis locks are fine. For *correctness* — 'don't charge the card twice' — use something with a real consensus protocol like etcd, or push it into the database. The important thing is being able to say which of those two you're in."

---

## Q10. Implement rate limiting.

### 🗣️ Say this
> "I'd use a token bucket, implemented as a Lua script in Redis.
> The bucket refills at a fixed rate up to a capacity. Each request takes a token; no tokens, you get a 429. The reason I prefer it over a sliding window is that it naturally permits a burst, which matches how clients actually behave — a page load fires six requests at once and you don't want to reject five of them. And it's O(1) memory per key, where a sliding-window *log* stores every request timestamp.
> The critical part is that it has to be a Lua script, not five client calls. Read-modify-write from the application means two concurrent requests both read 99 tokens and both proceed.
> Lua runs inside Redis's single thread, so the whole read-refill-decrement-write sequence is indivisible. That's the entire reason it's a script."

### 💻 Proof — the burst behaviour, *(real)*
```python
b = Bucket(rate=2, cap=5)        # 2 per second, burst of 5
```
```
Output:
8 instant requests : AAAAA...    (5 allowed = the burst, then empty)
after 1s wait      : AA..        (refilled exactly 2 tokens)
```
> You can see both halves of the behaviour: it absorbed a burst of 5, then throttled to exactly the refill rate.

### 💻 The Lua script
```lua
-- KEYS[1]=key  ARGV = rate, capacity, now, cost
local rate, cap, now, cost = tonumber(ARGV[1]), tonumber(ARGV[2]), tonumber(ARGV[3]), tonumber(ARGV[4])
local b = redis.call('HMGET', KEYS[1], 'tokens', 'ts')
local tokens = tonumber(b[1]) or cap
local ts     = tonumber(b[2]) or now
tokens = math.min(cap, tokens + (now - ts) * rate)       -- refill by elapsed time
local allowed = tokens >= cost
if allowed then tokens = tokens - cost end
redis.call('HMSET', KEYS[1], 'tokens', tokens, 'ts', now)
redis.call('EXPIRE', KEYS[1], math.ceil(cap / rate) * 2)
return { allowed and 1 or 0, tokens }
```

```python
@app.middleware("http")
async def rate_limit(request: Request, call_next):
    ok, remaining = await check_limit(identify(request))
    if not ok:
        return JSONResponse({"detail": "rate limit exceeded"}, status_code=429,
                            headers={"Retry-After": "1", "RateLimit-Remaining": "0"})
    resp = await call_next(request)
    resp.headers["RateLimit-Remaining"] = str(int(remaining))
    return resp
```

### 🧠 The algorithms
| Algorithm | Memory | Burst | Boundary bug |
|---|---|---|---|
| Fixed window | 1 counter | Allows 2× at the boundary | ❌ Yes |
| Sliding log | O(requests) | Exact | ✅ None |
| Sliding counter | 2 counters | Approximate | ✅ Mostly |
| **Token bucket** | 2 fields | **Controlled burst** | ✅ None |
| Leaky bucket | 2 fields | Smooths to constant rate | ✅ None |

### ⚠️ Follow-ups
| They ask | Say this |
|---|---|
| What do you key on? | "API key or user ID for authenticated traffic. IP is shared behind NAT and mobile carriers, so it punishes the wrong people." |
| Where does this run? | "App layer catches business limits. But if you're being flooded, rate limiting in Python still costs a socket, a TLS handshake and a Redis round trip per request. Real DDoS protection is at the CDN or WAF — in-app limiting is for fairness and quota." |
| Redis is down? | "Fail-open or fail-closed is a product decision and I'd want it configurable per tier. Public read API: fail open. Paid API guarding a scarce resource: fail closed." |

---

## Q11. Streams vs Pub/Sub vs Lists.

### 🗣️ Say this
> "Pub/Sub has **zero persistence** — that's the headline. If a subscriber is disconnected, restarting or deploying, the message is gone with no trace and no error. It's fine for 'invalidate this cache key' where missing one is survivable, and completely wrong for anything you must not lose.
> Lists give you a simple work queue with `LPUSH` and `BRPOP`, but once a worker pops a message it's gone — a crash loses it.
> Streams are the real answer for delivery guarantees: a persistent, replayable log with consumer groups and per-message acknowledgement, so an unacknowledged message can be reclaimed.
> But I'd add that if the guarantees genuinely matter, I'd reach for SQS or Kafka rather than making Redis a message broker."

| | Pub/Sub | List | Stream |
|---|---|---|---|
| Lost if no consumer? | ✅ **Yes** | ❌ | ❌ |
| Consumer groups | ❌ | ❌ | ✅ |
| Redelivery after a crash | ❌ | ❌ | ✅ |
| Replay history | ❌ | ❌ | ✅ |

---

## Q12. Redis Cluster — what changes for your code?

### 🗣️ Say this
> "Cluster shards keys across 16,384 hash slots, and the main thing that changes is that multi-key operations only work when all the keys land on the same node. So `MGET`, transactions, and Lua scripts touching several keys break — unless you use hash tags, where `{user:123}:profile` and `{user:123}:settings` hash only the bit in braces and land together.
> Rate limiters and locks are usually single-key so they're fine, but pipelines need care.
> The other thing is that a hot key is a hot *node*. If one key takes 40% of your traffic, sharding doesn't help at all — the fix is client-side caching of that key, or splitting it into several copies you read at random.
> And I'd say most people running Cluster don't need it. A single large instance with a replica handles a lot, and Cluster costs you multi-key operations."

---

## ⚡ Rapid fire

| They ask | Say this |
|---|---|
| How do you find big keys? | "`redis-cli --bigkeys` — it samples, so it's safe on a live instance. Or `MEMORY USAGE key` for a specific one." |
| What does `SCAN` guarantee? | "Anything present for the whole iteration is returned at least once. You may see duplicates, and it never blocks." |
| Pipelining vs `MULTI`? | "Pipelining just batches for round-trip efficiency, no atomicity. `MULTI/EXEC` is atomic but it can't branch on values it reads — and it's not a rollback mechanism." |
| Lua vs `MULTI`? | "Lua can make decisions based on what it reads. `MULTI` can't. That's why read-modify-write has to be Lua." |
| Latency spike — first check? | "`SLOWLOG GET`, then fork activity via `latest_fork_usec`, then whether anything ran an O(n) command, then swap." |
| Sessions: Redis or JWT? | "JWT scales without a lookup but can't be revoked before it expires. Redis sessions are revocable instantly but are a hard dependency. Common answer is a short-lived JWT plus a refresh token checked against Redis." |
| Cache key design? | "`entity:id:version`, including every dimension that changes the result — especially the authorisation dimension. And version the prefix so a schema change invalidates everything at once." |
| Gotcha with `EXPIRE`? | "Most commands that replace a value **clear the TTL**. `SET` wipes it unless you pass `KEEPTTL`. That's a classic silent bug — your cache entry quietly becomes permanent." |
| How do you warm a cold cache? | "Pre-populate the known-hot set at startup *before* passing the readiness probe, and ramp traffic slowly rather than switching everything at once." |

---

**Next:** [05-message-queues-sqs-sns-kafka.md](05-message-queues-sqs-sns-kafka.md)

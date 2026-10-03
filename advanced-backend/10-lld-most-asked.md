# 10 — Low-Level Design: The Most-Asked Problems

> **How to use this file.** Each problem has a **🗣️ Say this** block — what you narrate while writing the code. That narration *is* the score; the code alone isn't. `Output:` blocks were actually run.

**What LLD really tests.** Not whether you know patterns — whether you can take a vague requirement, find the entities, give them clean responsibilities, and write code someone else could extend without editing your classes.

---

## 📖 Jargon, in plain words

| Jargon | What it actually means | How you'd say it |
|---|---|---|
| **Entity** | A noun in the problem that has identity and state. | "Ticket, Vehicle, Spot — those are my entities." |
| **Interface / abstract base** | A promise about what methods exist, with no implementation. | "I'll make this an interface because I expect it to vary." |
| **Strategy** | Swapping one algorithm for another behind a common interface. | "Pricing is a Strategy — new rules become new classes." |
| **State pattern** | Each state is its own object, so behaviour changes with state. | "It removes the `if status ==` chain from every method." |
| **Composite** | Treating a single thing and a group of things the same way. | "File and Directory share an interface, so `size` just recurses." |
| **Open/Closed** | Adding a feature should mean adding code, not editing existing code. | "Adding a variant shouldn't force me to re-test the others." |
| **God class** | One class doing far too much. | "If the name needs an 'and', split it." |
| **Invariant** | A rule that must always be true. | "I encode invariants so invalid states can't be built." |
| **Idempotent operation** | Doing it twice equals doing it once. | "Makes retries harmless." |

---

## 🧭 The method — 40 minutes

| Phase | Time | Do this |
|---|---|---|
| **1. Clarify** | 5 min | Ask 4–6 questions. "Single machine or distributed? Persistence? Concurrent access?" |
| **2. Use cases** | 3 min | Bullet them. "Park, unpark, compute fee, find a spot." |
| **3. Entities** | 7 min | Nouns → classes. Draw relationships. **Define the state enums here.** |
| **4. Interfaces** | 5 min | Only where variation is expected. **This is where the points are.** |
| **5. Code** | 15 min | Two or three classes fully, the rest as signatures. Don't write getters. |
| **6. Extend & critique** | 5 min | "If you asked me to add X, here's what changes." Then name your own weaknesses. |

⚠️ **The five ways people lose points:** coding before clarifying (the problem is *deliberately* underspecified); a god class that does everything; patterns applied for their own sake; `if/elif` chains where polymorphism belongs; and going silent.

🎤 **Sentences to keep in your pocket:**
- *"Let me confirm the scope first — should I handle concurrency, or assume single-threaded?"*
- *"I'll make this an interface because I expect it to vary. The others I'll keep concrete until there's a reason."*
- *"That's the trade-off — simpler to read, but X has to change when Y does."*

---

## LLD 1. Parking Lot

**Clarify:** Multiple floors? Vehicle types and size rules? Pricing model? Payment in scope? Concurrent entry gates?

### 🗣️ Say this while you code
> "I'm making two things interfaces and nothing else. Spot allocation and pricing — because those are exactly the two things a business changes. Adding weekend pricing becomes a new class, not an edit to `ParkingLot`.
> Everything else stays concrete. I'm not making `Vehicle` a class hierarchy because the only thing that varies is size, and an enum carries that fine.
> The lock goes around **select-and-occupy together**. Two gates checking the same spot and both taking it is the obvious race, and locking only the occupy step doesn't fix it.
> And money is an integer in minor units — floats for currency is a correctness bug."

### 💻
```python
class VehicleType(Enum):
    MOTORCYCLE = 1; CAR = 2; TRUCK = 3          # ordinal = size rank

@dataclass
class ParkingSpot:
    id: str; floor: int; size: SpotSize
    occupied_by: str | None = None
    @property
    def is_free(self) -> bool: return self.occupied_by is None
    def fits(self, v: Vehicle) -> bool:
        return self.size.value >= v.type.value

# --- Variation point 1: how we pick a spot ---
class SpotAllocationStrategy(ABC):
    @abstractmethod
    def select(self, spots, v) -> ParkingSpot | None: ...

class NearestFirst(SpotAllocationStrategy):
    def select(self, spots, v):
        candidates = [s for s in spots if s.is_free and s.fits(v)]
        # smallest adequate spot, lowest floor - don't waste a truck bay on a bike
        return min(candidates, key=lambda s: (s.size.value, s.floor), default=None)

# --- Variation point 2: how we charge ---
class FeeStrategy(ABC):
    @abstractmethod
    def compute(self, ticket) -> int: ...        # MINOR UNITS, never float

class ParkingLot:
    def __init__(self, spots, allocator, pricer):
        self._spots = {s.id: s for s in spots}
        self._allocator, self._pricer = allocator, pricer
        self._lock = threading.Lock()            # entry gates are concurrent

    def park(self, vehicle) -> Ticket:
        with self._lock:                         # select + occupy must be ATOMIC
            spot = self._allocator.select(list(self._spots.values()), vehicle)
            if spot is None:
                raise LotFullError(f"no spot for {vehicle.type.name}")
            spot.occupied_by = vehicle.plate
            return Ticket(str(uuid.uuid4()), vehicle, spot, datetime.now())
```

⚠️ **Follow-ups:** *Real-time free-spot display* → "Keep a counter per floor and size — scanning every spot on every query is O(n) per request." *Distributed across gates* → "The lock becomes the database: `UPDATE spots SET occupied_by=? WHERE id=? AND occupied_by IS NULL`, then check rowcount. A conditional update, not a lock service." *Reservations* → "A third state with a TTL, and now the state machine needs to be explicit."

---

## LLD 2. LRU Cache

**The classic.** They want O(1) `get` and `put` — that constraint forces the data structure.

### 🗣️ Say this
> "A hash map alone gives me O(1) lookup but no idea what's least recently used. A list alone gives me order but O(n) lookup.
> So I combine them: a map from key to a *node*, and those nodes linked together in recency order. Touching an item moves its node to the front, and eviction takes from the back. Both O(1).
> The sentinel head and tail nodes exist purely so I never write a null check when unlinking — that's where this implementation usually has bugs.
> And it's exactly what `functools.lru_cache` does in C underneath."

### 💻
```python
class LRUCache:
    def __init__(self, capacity: int):
        self.capacity, self._map = capacity, {}
        # sentinels: remove every null check from link/unlink
        self._head, self._tail = Node(), Node()
        self._head.next, self._tail.prev = self._tail, self._head

    def _remove(self, n):    n.prev.next, n.next.prev = n.next, n.prev
    def _push_front(self, n):
        n.next, n.prev = self._head.next, self._head
        self._head.next.prev = self._head.next = n

    def get(self, key):
        n = self._map.get(key)
        if n is None: return None
        self._remove(n); self._push_front(n)      # touch = most recent
        return n.value

    def put(self, key, value):
        if (n := self._map.get(key)) is not None:
            n.value = value; self._remove(n); self._push_front(n); return
        if len(self._map) >= self.capacity:
            lru = self._tail.prev                  # evict from the TAIL
            self._remove(lru); del self._map[lru.key]
        n = Node(key, value); self._map[key] = n; self._push_front(n)
```
```
Output:
put a,b,c -> keys: ['a', 'b', 'c']
get('a') -> A (a is now most-recent)
  evicted 'b'
after put d -> keys: ['a', 'c', 'd']
```
> Note what got evicted: **`b`**, not `a`. Even though `a` was inserted first, the `get('a')` moved it to the front. That single line of output proves the recency logic works.

⚠️ **Follow-ups:** *Thread-safe?* → "One lock around both methods. The operations are microseconds so contention is low. If it were a real bottleneck, shard into N caches by `hash(key) % N`." *Add a TTL?* → "Store an expiry on the node and treat expired as a miss on read — lazy expiry is usually enough." *LFU instead?* → "Frequency buckets as linked lists with a min-frequency pointer. Genuinely harder — know it exists."

---

## LLD 3. Rate Limiter (in-process)

### 🗣️ Say this
> "Token bucket, with **lazy refill** — that's the part worth pointing out. I don't run a background timer adding tokens. I compute how many *would* have accumulated since the last call. That's O(1) and needs no scheduler at all.
> I use `time.monotonic()` rather than `time.time()`, because a clock adjustment would otherwise grant or revoke tokens.
> And I'd flag a weakness in my own design: the bucket dict grows forever with distinct keys. That's a memory leak, and a real implementation needs LRU eviction or a TTL sweep. I'd rather name that than have them find it."

### 💻
```python
class TokenBucketLimiter:
    def __init__(self, rate: float, capacity: int):
        self.rate, self.capacity = rate, capacity
        self._buckets: dict[str, tuple[float, float]] = {}   # key -> (tokens, ts)
        self._lock = threading.Lock()

    def allow(self, key: str, cost: int = 1) -> bool:
        now = time.monotonic()                   # monotonic: immune to clock changes
        with self._lock:
            tokens, last = self._buckets.get(key, (self.capacity, now))
            tokens = min(self.capacity, tokens + (now - last) * self.rate)  # lazy refill
            if tokens >= cost:
                self._buckets[key] = (tokens - cost, now); return True
            self._buckets[key] = (tokens, now); return False
```
```
Output (rate=2/sec, capacity=5):
8 instant requests : AAAAA...    (5 allowed = the burst, then empty)
after 1s wait      : AA..        (refilled exactly 2 tokens)
```

> The distributed version is [04-redis-and-caching.md](04-redis-and-caching.md) Q10 — same algorithm, moved into a Lua script so it's atomic across pods.

---

## LLD 4. Elevator System

### 🗣️ Say this
> "The core is the SCAN algorithm — keep going in one direction serving every request on the way, then reverse. That's why I hold **two separate stop sets**, one for going up and one for going down. It makes 'what's next in my current direction' a simple `min` or `max` rather than a sort.
> And I've split the `Elevator` state machine from the `ElevatorController`'s dispatch policy, because the physics never change but the dispatch policy does. Rush hour, express floors and energy-saving mode are all just another `SchedulingStrategy`.
> I'd also point out that `step()` is a pure tick function with the time source injected — so I can drive the whole simulation deterministically in a test without waiting for real seconds."

### 💻
```python
@dataclass
class Elevator:
    id: int
    current_floor: int = 0
    direction: Direction = Direction.IDLE
    up_stops: set[int] = field(default_factory=set)      # two sets, not one queue
    down_stops: set[int] = field(default_factory=set)

    def next_stop(self) -> int | None:
        if self.direction is Direction.UP and self.up_stops:
            return min(self.up_stops)                     # cheapest possible lookup
        if self.direction is Direction.DOWN and self.down_stops:
            return max(self.down_stops)
        if self.up_stops:   return min(self.up_stops)      # reverse when exhausted
        if self.down_stops: return max(self.down_stops)
        return None

class NearestSameDirection(SchedulingStrategy):
    """Prefer a lift already heading your way and about to pass you."""
    def pick(self, elevators, floor, want):
        def cost(e):
            distance = abs(e.current_floor - floor)
            if e.direction is Direction.IDLE: return (1, distance)
            toward = (e.direction is want) and (
                (want is Direction.UP and e.current_floor <= floor) or
                (want is Direction.DOWN and e.current_floor >= floor))
            return (0 if toward else 2, distance)
        return min(elevators, key=cost)
```

⚠️ **Follow-ups:** capacity (the controller must not dispatch a full car to a hall call); emergency mode (a state transition clearing all stops); *"how do you test it?"* (the injected clock — say this, it shows design maturity).

---

## LLD 5. Vending Machine

**The canonical State pattern problem.** If you use State anywhere, use it here.

### 🗣️ Say this — **this is the one time "I used a pattern" is the right answer**
> "Without the State pattern, `select()` starts with a chain of `if self.status == ...` checks, and every single method repeats that chain.
> With State, each state class answers 'what does this operation mean *here*'. An invalid operation is handled once, in the state where it's invalid.
> And adding a `MaintenanceState` is a new class with zero edits to existing ones — which is the Open/Closed principle actually doing something rather than being quoted.
> The machine holds the data; the state holds the behaviour. Every public method just delegates."

### 💻
```python
class IdleState(VendingState):
    def insert_coin(self, m, amount):
        m.balance += amount; m.state = HasMoneyState()
    def select(self, m, code):  raise InvalidOperation("insert money first")

class HasMoneyState(VendingState):
    def insert_coin(self, m, amount): m.balance += amount
    def select(self, m, code):
        item = m.inventory.get(code)
        if item is None or item.qty == 0: raise OutOfStock(code)
        if m.balance < item.price:
            raise InsufficientFunds(item.price - m.balance)
        m.selected = code; m.state = DispensingState()

class VendingMachine:
    def __init__(self, inventory):
        self.inventory, self.balance, self.state = inventory, 0, IdleState()
    # machine holds DATA; state holds BEHAVIOUR. Just delegate.
    def insert_coin(self, a): return self.state.insert_coin(self, a)
    def select(self, c):      return self.state.select(self, c)
```
```
Output:
select before coin -> insert money first        <- rejected by IdleState
coin(50)           -> balance=50
select             -> need 15 more              <- rejected by HasMoneyState
coin(20)           -> balance=70
select             -> dispensed A1, change=5
```
> Two different rejections from two different states, each handled in exactly one place. That's the pattern earning its keep.

⚠️ **Follow-ups:** *Change-making* → "Greedy by denomination — but I'd note greedy is only optimal for canonical coin systems; the general case is dynamic programming." *Concurrency* → "A physical machine is single-user, so I wouldn't add locking. That's a case where *not* handling concurrency is the right call, and I'd say so rather than adding it reflexively."

---

## LLD 6. Library / Booking System

**The real test here is the state machine.** Most booking-flavoured problems are.

### 🗣️ Say this
> "I'd make the state machine an explicit data structure rather than scattering `if` statements, because that's the difference between a bug being impossible and being one code review away.
> `RETURNED → CHECKED_OUT` simply can't happen — the transition table says so in one readable place, and adding a state is one dictionary entry.
> The entity distinction that matters, and the one candidates most often miss, is `Book` versus `BookCopy`. You search for a *book*; you loan a physical *copy*. Getting that wrong makes everything downstream awkward.
> And I'd compute fines on read from the due date rather than running a nightly job that mutates rows — fewer moving parts and no job to fail."

### 💻
```python
TRANSITIONS: dict[LoanState, set[LoanState]] = {
    LoanState.RESERVED:    {LoanState.CHECKED_OUT, LoanState.CANCELLED},
    LoanState.CHECKED_OUT: {LoanState.RETURNED, LoanState.OVERDUE},
    LoanState.OVERDUE:     {LoanState.RETURNED},
    LoanState.RETURNED:    set(),            # terminal
    LoanState.CANCELLED:   set(),
}

@dataclass
class Loan:
    id: str; copy_id: str; member_id: str
    state: LoanState = LoanState.RESERVED
    def transition_to(self, new: LoanState) -> None:
        if new not in TRANSITIONS[self.state]:
            raise InvalidTransition(f"{self.state.value} -> {new.value}")
        self.state = new
```

⚠️ **Follow-ups:** *All copies out* → "A FIFO reservation queue per book; the first in line gets notified with a hold window." *Two members reserve the last copy* → "Conditional update on the copy's status — same as the parking spot."

---

## LLD 7. Logging Framework

### 🗣️ Say this
> "Three separate axes of variation, so three separate abstractions.
> *Whether* to log is the level check. *Where* it goes is the Handler. *How it looks* is the Formatter. Keeping those orthogonal means a JSON-to-stdout handler and a plain-text-to-file handler compose from the same parts.
> The parent chain is Chain of Responsibility — a logger called `app.db.pool` propagates up to `app.db` then `app`, so you configure it once at the top and everything inherits.
> That's why each Handler has its *own* level too — you can send everything to stdout but only errors to a paging system."

### 💻
```python
class Handler(ABC):
    def __init__(self, level=LogLevel.DEBUG, formatter=None):
        self.level, self.formatter = level, formatter or JSONFormatter()
    def handle(self, r: LogRecord) -> None:
        if r.level >= self.level:              # each handler has its OWN threshold
            self.emit(self.formatter.format(r))
    @abstractmethod
    def emit(self, text: str) -> None: ...

class Logger:                                  # Chain of Responsibility
    def log(self, level, message, **context):
        if level < self.level: return
        self._dispatch(LogRecord(level, message, time.time(), self.name, context))
    def _dispatch(self, record):
        for h in self.handlers: h.handle(record)
        if self.propagate and self.parent:      # app.db.pool -> app.db -> app
            self.parent._dispatch(record)
```

⚠️ **Follow-up: "Make it non-blocking."** → "A `QueueHandler` that puts records on a queue with one background thread draining it, so the hot path never touches I/O. That's exactly what the stdlib's `logging.handlers.QueueHandler` does, and it's the right answer for a latency-sensitive service."

---

## LLD 8. Tic-Tac-Toe / Board Game

**What's really tested:** can you generalise? The follow-up is always "now make it N×N."

### 🗣️ Say this — **lead with the optimisation, it's the differentiator**
> "The naive win check scans every row, column and diagonal after each move — that's O(n²) per move. Most candidates write that.
> Instead I keep a running sum per line: plus one for X, minus one for O. A line is won when its sum hits plus or minus n.
> That's O(1) per move and it generalises straight to any board size with no extra work.
> Leading with this rather than writing the scan first is worth saying explicitly, because it's the thing they're looking for."

### 💻
```python
class Board:
    def __init__(self, n: int = 3):
        self.n = n
        self.grid = [[None]*n for _ in range(n)]
        self._rows = [0]*n; self._cols = [0]*n     # running sums, not a scan
        self._diag = self._anti = 0

    def place(self, row, col, piece) -> bool:
        """Returns True if this move wins. O(1), not O(n²)."""
        if self.grid[row][col] is not None:
            raise InvalidMove(f"({row},{col}) occupied")
        self.grid[row][col] = piece
        delta = 1 if piece is Piece.X else -1
        self._rows[row] += delta
        self._cols[col] += delta
        if row == col:              self._diag += delta
        if row + col == self.n - 1: self._anti += delta
        target = self.n * delta
        return (self._rows[row] == target or self._cols[col] == target
                or self._diag == target or self._anti == target)
```
```
Output:
X at (0,0) -> win=False
O at (1,1) -> win=False
X at (0,1) -> win=False
O at (2,2) -> win=False
X at (0,2) -> win=True          <- detected in O(1), no board scan
```

⚠️ **Follow-ups:** *Connect Four* → "The win condition becomes four-in-a-row anywhere, so running counts don't work — you check outward from the placed piece in four directions, which is O(k). And gravity means `place` takes only a column." *An AI opponent* → "Minimax with alpha-beta pruning; the board is small enough to solve exactly."

---

## LLD 9. In-Memory File System

### 🗣️ Say this
> "This is Composite, and it's genuinely the right pattern rather than pattern-naming for its own sake. `File` and `Directory` share an `FSNode` interface, so `size` works identically whether you call it on a leaf or a whole tree — the recursion is invisible to the caller.
> Without it, every single operation needs an `isinstance` check, so the pattern actually changes the code rather than just labelling it.
> For search I'd make it a generator rather than returning a list, so a huge tree doesn't get materialised in memory."

### 💻
```python
@dataclass
class File(FSNode):
    content: bytearray = field(default_factory=bytearray)
    @property
    def size(self) -> int: return len(self.content)

@dataclass
class Directory(FSNode):
    children: dict[str, FSNode] = field(default_factory=dict)
    @property
    def size(self) -> int:                      # Composite: recurses uniformly
        return sum(child.size for child in self.children.values())
```

⚠️ **Follow-ups:** *Hard links* → "Multiple parents, so the tree becomes a DAG and `path` is no longer unique. That breaks the parent-pointer design and I'd need a separate inode layer." *Symlinks* → "A node holding a target path, and now I need loop detection." *Persistence* → "This is where an in-memory tree stops being the right model — real filesystems split metadata from data blocks precisely so you can update one without rewriting the other."

---

## LLD 10. Splitwise / Expense Sharing

### 🗣️ Say this
> "Two things I'd call out.
> First, the remainder. Three people splitting 100 units is 33, 33 and 34 — and anyone who writes `amount / n` silently loses a unit. Interviewers specifically look for that.
> Second, the settlement. I compute **net** balances first, which collapses circular debt — if A owes B, B owes C and C owes A, the net may be zero and nobody needs to pay anyone.
> Then greedily match the largest debtor to the largest creditor. I'd be honest that this isn't provably minimal — the exact problem is NP-hard — rather than claim it's optimal."

### 💻
```python
class EqualSplit(SplitStrategy):
    def split(self, amount, participants, **kw):
        share, rem = divmod(amount, len(participants))
        # the remainder must go SOMEWHERE - silently dropping it is a real bug
        return {p: share + (1 if i < rem else 0) for i, p in enumerate(participants)}
```
```
Output:
naive amount//n : {'a': 33, 'b': 33, 'c': 33}  total=99   <- LOST 1 unit
with remainder  : {'a': 34, 'b': 33, 'c': 33}  total=100  <- correct
```
> A one-unit discrepancy per transaction, silently, forever. That's what `divmod` is preventing.

```python
def settlements(self):
    net = defaultdict(int)
    for a, owed in self._balances.items():
        net[a] = -sum(owed.values())            # collapse circular debt first
    debtors   = sorted([(p,-v) for p,v in net.items() if v < 0], key=lambda x:-x[1])
    creditors = sorted([(p, v) for p,v in net.items() if v > 0], key=lambda x:-x[1])
    ...                                          # greedy: biggest pays biggest
```

---

## LLD 11. In-Process Task Scheduler

### 🗣️ Say this
> "A min-heap keyed on run time — that gives O(log n) insert and O(1) peek at the next task. A sorted list would be O(n) to insert, so the heap is the right structure here.
> The scheduler sleeps until the *next* task is due rather than polling every second, and a wake event handles the case where a new task gets scheduled sooner than the current head.
> And the task runs in its own `create_task` so a slow job doesn't delay the scheduler itself — that's the bug in most naive implementations.
> The tiebreak sequence number matters too: two tasks due at the same instant would otherwise try to compare the functions themselves and crash."

### 💻
```python
@dataclass(order=True)
class ScheduledTask:
    run_at: float
    seq: int                                   # tiebreak: stable FIFO, no fn compare
    fn: Callable = field(compare=False)
    interval: float | None = field(default=None, compare=False)

async def run(self):
    while self._running:
        task = self._heap[0]
        timeout = task.run_at - time.monotonic()
        if timeout > 0:
            try:                               # sleep until due, OR until woken
                await asyncio.wait_for(self._wake.wait(), timeout)
                self._wake.clear(); continue
            except TimeoutError:
                pass
        heapq.heappop(self._heap)
        asyncio.create_task(self._run_one(task))   # don't block the scheduler loop
```

⚠️ **Follow-up: "Make it distributed."** → "The heap becomes a database table ordered by `next_run_at`, and coordination is `FOR UPDATE SKIP LOCKED` — that's [09-system-design-advanced.md](09-system-design-advanced.md) D10. The in-process design doesn't survive multiple replicas, because each would have its own heap and every job would run N times."

---

## LLD 12. Online Judge / Code Execution Service

### 🗣️ Say this — lead with the reframe
> "This is a security problem wearing an LLD costume. I'm executing arbitrary untrusted code, so isolation is the *first* design constraint, not something I bolt on afterwards.
> The object model is straightforward — a request with source, language and resource limits; a result with a status enum; a `LanguageRuntime` strategy per language; a `Sandbox` strategy per isolation mechanism.
> The interesting part is the isolation layers: resource limits and syscall filtering at the process level, a read-only filesystem, no network at all, an unprivileged user, cgroups for CPU and memory, and gVisor or Firecracker if I need a real kernel boundary.
> And I'd point out you need a **PID limit** specifically, because a fork bomb isn't stopped by a memory limit alone."

| Layer | Mechanism |
|---|---|
| Process | `setrlimit`, `seccomp` to restrict syscalls |
| Filesystem | Read-only root, tmpfs scratch |
| Network | None — `--network none` |
| Identity | Unprivileged UID, no capabilities |
| Resources | cgroups for CPU, memory, **and PIDs** |
| Kernel | gVisor / Firecracker for a real boundary |

⚠️ **Follow-ups:** *Wall-clock vs CPU time* → "Both. A sleeping program burns no CPU but still holds a slot." *Throughput* → "A pool of warm sandboxes — container start dominates the latency of a 50ms program."

---

## ⚡ LLD rapid fire

| They ask | Say this |
|---|---|
| "Where would you use an interface?" | "Where I expect variation *and* it exists or is clearly coming. One implementation with no plan for a second — don't." |
| "How do you avoid a god class?" | "If the class name needs an 'and' to describe it, split it. Coordinator, policy, entity." |
| "How do you handle invalid states?" | "Make them unrepresentable — an explicit transition table, or a type that can't hold the bad value." |
| "Composition or inheritance?" | "Composition by default. Inheritance only for a genuine is-a relationship with shared behaviour, never just to reuse code." |
| "Where does validation go?" | "At the boundary with Pydantic, and invariants in the constructor so an invalid object can't exist at all." |
| "How would you test this?" | "Inject the clock and randomness, and keep decision logic pure so it's testable without mocks. I'd name that as a design goal, not an afterthought." |
| "Is it thread-safe?" | "Ask whether it needs to be first. If yes, lock the smallest correct region — and say exactly what the race is." |
| "What's your design's weakness?" | **Always have one ready.** Naming your own trade-off is the strongest move in the round. |

---

**Next:** [11-design-patterns-in-interviews.md](11-design-patterns-in-interviews.md)

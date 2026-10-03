# 11 — How to Explain a Design Pattern in an Interview

> **How to use this file.** This isn't a pattern catalogue — there are thousands of those. It's the **performance**: what to say, in what order, and why most candidates get it wrong.
>
> The four-move script below is the single most reusable thing in this whole folder. Learn it and you can answer a pattern question you've never rehearsed.

---

## 📖 Jargon, in plain words

| Jargon | What it actually means | How you'd say it |
|---|---|---|
| **Design pattern** | A named, reusable shape for solving a recurring design problem. | "It's a name for a solution shape, not a rule." |
| **GoF** | "Gang of Four" — the 1994 book that named 23 classic patterns. | "A lot of GoF is solving for 1990s C++ limitations." |
| **Abstraction** | Hiding *how* something works behind *what* it does. | "Every abstraction has a cost paid by whoever reads the code." |
| **Indirection** | An extra hop between the caller and the real work. | "Patterns buy flexibility and pay in indirection." |
| **Coupling** | How much one piece of code depends on another's details. | "The goal is loose coupling, not zero coupling." |
| **Cohesion** | How much the things inside one class actually belong together. | "High cohesion means one class, one job." |
| **Open/Closed** | Open to extension, closed to modification. Add code, don't edit it. | "Adding a variant shouldn't make me re-test the others." |
| **Dependency inversion** | Depend on an interface, not on a concrete implementation. | "My service depends on a repository interface, not on SQLAlchemy." |
| **Premature abstraction** | Building flexibility for a change that never arrives. | "The cost is paid continuously; the benefit may never arrive." |
| **YAGNI** | "You aren't gonna need it." | "The counterweight to pattern enthusiasm." |

---

## ❌ The problem with how people answer

Ask a candidate "what is the Strategy pattern?" and you usually get:

> ❌ *"Strategy is a behavioural design pattern that defines a family of algorithms, encapsulates each one, and makes them interchangeable. It lets the algorithm vary independently from clients that use it."*

That's the textbook definition, verbatim. It's correct, and it scores close to zero. Four reasons:

1. **It proves you read a book, not that you've shipped code.** Anyone can memorise it.
2. **It's abstract.** *They* now have to imagine where it applies.
3. **It has no trade-off.** Every pattern costs something. Not naming the cost means you've never paid it.
4. **It's interchangeable.** Swap "Strategy" for "Observer" and the sentence structure is identical. It tells them nothing about *you*.

At six years, the question behind the question is never "can you recite this?" It's **"do you know when to reach for it — and when not to?"**

---

## ✅ The four-move script

Use this every single time. It takes about 45 seconds and works for any pattern.

| Move | What you do | Time |
|---|---|---|
| **1. The problem** | Describe the pain *without naming the pattern*. What goes wrong without it? | 15s |
| **2. The mechanism** | One sentence on what the pattern actually does. | 10s |
| **3. Your example** | Real code you've written. Name the actual system. | 20s |
| **4. The cost** | When you would *not* use it. | 10s |

**Why the order matters.** Starting with the problem shows you think in terms of forces, not vocabulary. It also means that if you blank on the pattern's name halfway through, the answer still lands — because you've described the engineering, and the name is only a label.

---

### 🗣️ The same question, answered well

> **Q: "Tell me about the Strategy pattern."**
>
> **1 — The problem.** "The pain it solves is a method that grows a new branch every time the business adds a variant. In a pricing service I worked on, the fee calculation started as one `if` on customer type, then there was a weekend rate, then a promotional rate, then enterprise contracts. It got to about two hundred lines, and every new pricing rule meant editing — and re-testing — code that handled all the other rules. That's the smell: the class has to change for a reason that isn't its own."
>
> **2 — The mechanism.** "Strategy pulls each variant into its own class behind a common interface, and the caller holds a reference to one of them. The branch disappears, because the choice is made once at construction instead of on every call."
>
> **3 — Your example.** "In the parking-fee code it was a `FeeStrategy` interface with a `compute` method, and separate classes for hourly, slab and flat rate. Adding weekend pricing became a new class plus one line of configuration. Nothing existing was touched, so nothing existing needed re-testing."
>
> **4 — The cost.** "It costs indirection. With three strategies in three files, reading the code means jumping around — and if you only ever have two variants that never change, a plain `if` is honestly clearer. I reach for it when the variants are genuinely open-ended, not the first time I write two branches."

### 💻 And if they ask you to show it
```python
class FeeStrategy(ABC):
    @abstractmethod
    def compute(self, hours: int) -> int: ...

class Hourly(FeeStrategy):
    def compute(self, hours): return 50 * hours
class FlatRate(FeeStrategy):
    def compute(self, hours): return 200

def charge(strategy: FeeStrategy, hours: int) -> int:
    return strategy.compute(hours)          # <- this function NEVER changes

# Interviewer: "now add weekend pricing"
class Weekend(FeeStrategy):                 # a NEW class, zero edits elsewhere
    def compute(self, hours): return int(50 * hours * 1.5)
```
```
Output:
hourly  (3h): 150
flat    (3h): 200
weekend (3h): 225   <- charge() was not touched
```
> That last line is the entire point of the pattern, demonstrated. **Five signals** in one answer: you spot a smell, you know the mechanism, you've applied it, you know the cost, and you have a threshold. The textbook definition gave one.

---

## 🧨 The three follow-ups you must be ready for

**"When would you *not* use it?"**
> Already covered in move 4 — which is exactly why move 4 exists. Pre-empting this makes you look like you've thought about it rather than been caught out.

**"Have you seen it used badly?"** — *the best question they can ask you*
> A real over-engineering example is worth more than three correct definitions:
> > *"Yes — a codebase where every service had an abstract base class and a factory, and each had exactly one implementation. The indirection was real and the flexibility was hypothetical. It took three file jumps to find out what a method did, and in four years nobody ever added a second implementation. That taught me to add the abstraction when the second case actually arrives, not in anticipation of it."*

**"Isn't that just a function?"**
> In Python, often yes — and saying so scores, because it shows you're not applying Java patterns to a language with first-class functions:
> > *"Frequently, yes. In Python a Strategy with one method is just a function you pass in, and `Callable[[int], int]` is a perfectly good interface. I'd use a class when the strategy has state or several related methods, and a plain function when it doesn't. Half the GoF catalogue exists to work around not having first-class functions."*

```python
def weekend_fn(hours): return int(50 * hours * 1.5)
def charge_fn(strategy, hours): return strategy(hours)
```
```
Output:
weekend as a plain function: 225      <- identical result, no class needed
```

---

## 🐍 The Python caveat — say this, it's a differentiator

Several classic patterns are workarounds for languages Python isn't. Knowing *which* shows you understand why patterns exist rather than treating them as universally good.

| Pattern | In Python |
|---|---|
| **Strategy** | Often just a function or `Callable` |
| **Command** | A closure or `functools.partial` |
| **Singleton** | A module — imported once and cached. The classic version is an anti-pattern anyway |
| **Factory** | Often a dict of `{key: class}`, or `__init_subclass__` registration |
| **Iterator** | Built into the language — `__iter__` and generators |
| **Template Method** | Fine, but often better as a function taking hooks |
| **Observer** | Fine as-is — a list of callbacks. Survives intact |
| **Adapter / Facade** | Fine as-is, genuinely useful, language-independent |
| **Repository / Unit of Work** | Very much alive — SQLAlchemy's `Session` *is* a Unit of Work |

### 💻 Factory without a factory class
```python
class Handler:
    registry: dict[str, type] = {}
    def __init_subclass__(cls, *, event, **kw):
        super().__init_subclass__(**kw)
        Handler.registry[event] = cls        # self-registration

class UserCreated(Handler, event="user.created"): pass
class OrderPaid(Handler,  event="order.paid"):   pass
```
```
Output:
{'user.created': <class 'UserCreated'>, 'order.paid': <class 'OrderPaid'>}
```

🗣️ **The sentence to deploy:**
> "A lot of the Gang of Four catalogue is solving for limitations of 1990s C++ and Java — no first-class functions, no duck typing, no modules as namespaces. In Python I'd use maybe a third of them literally. The *ideas* underneath — separate what varies, depend on abstractions, compose rather than inherit — all still hold. I try to keep the ideas and drop the ceremony."

---

## 📚 The patterns that actually appear in backend work

Ordered by how often they come up. For each: **problem → mechanism → your example → cost.**

### 1. Repository
> **Problem.** Business logic ends up full of SQL, so you can't test it without a database.
> **Mechanism.** A collection-like interface over persistence. The service talks to the interface, not the ORM.
> **Your example.** "It's the structure in this repo — `app/repositories/gitlab_user_repository.py`. The service calls `repo.get_by_username()` and never sees a `select()`. In tests I swap in an in-memory implementation and the service tests run with no database at all."
> **Cost.** "An extra layer — and with SQLAlchemy you're arguably wrapping something that's already a repository. The real risk is leaking: the moment a method is called `get_users_with_orders_eager_loaded`, the abstraction has failed and you're writing SQL in method names."

### 2. Dependency Injection
> **Problem.** A class builds its own dependencies, so you can't substitute them.
> **Mechanism.** Dependencies are handed in rather than created inside.
> **Your example.** "FastAPI's `Depends()` is a DI container, and the feature that proves its value is `dependency_overrides` — in tests I replace `get_session` with one bound to a transaction that rolls back, so every endpoint test gets a clean database with no mocking."
> **Cost.** "Constructor signatures grow, and 'where does this actually come from' takes a lookup. With implicit resolution, a dependency five levels deep is hard to trace."

### 3. Adapter
> **Problem.** A vendor's types leak everywhere, so replacing them means touching a hundred files.
> **Mechanism.** A wrapper translating their interface into yours.
> **Your example.** "`app/integrations/gitlab_client.py` — the service calls `create_user(...)` and gets a domain object. It never sees an HTTP status code or GitLab's JSON shape. When GitLab changed a response field, one file changed."
> **Cost.** "A layer to maintain, and translation can lose information — if the vendor returns a rich error and your adapter flattens it to a boolean, you've thrown away something you'll want during an incident. So I map errors to *typed domain exceptions* rather than swallowing them."

### 4. Observer / Pub-Sub
> **Problem.** One event triggers several unrelated effects, and the triggering code accumulates knowledge of all of them.
> **Mechanism.** Publisher emits; subscribers register independently.
> **Your example.** "The SNS-to-SQS fan-out in [05](05-message-queues-sqs-sns-kafka.md) is this at infrastructure scale. Adding a fifth consumer of `order.created` requires zero changes to the order service."
> **Cost.** "Control flow becomes invisible. You can't find all the effects of an event by reading the publisher — debugging 'why did this email send' means tracing through a bus."

### 5. Decorator
> **Problem.** Cross-cutting behaviour — retries, caching, timing — duplicated across many functions.
> **Mechanism.** Wrap the callable, adding behaviour before and after.
> **Your example.** "`@retry`, `@cached`, and middleware, which is the same idea at request level — each layer wraps the next, which is exactly the ASGI model."
> **Cost.** "Deeper stack traces, `functools.wraps` is mandatory or introspection breaks, and stacking five makes the order significant and non-obvious. A decorator that silently swallows exceptions is genuinely dangerous."

### 6. Circuit Breaker
> **Problem.** A failing dependency gets hammered by retries, consuming your capacity and its recovery capacity.
> **Mechanism.** Track failures; after a threshold, fail fast; periodically probe for recovery.
> **Your example.** [09-system-design-advanced.md](09-system-design-advanced.md) D7.
> **Cost.** "It's state you have to tune and observe. Too sensitive and it trips on normal variance — you've created an outage out of a blip. And per-instance breakers mean twenty pods each learn the dependency is down independently."

### 7. State
> **Problem.** Every method starts with the same `if status ==` chain, and invalid transitions are prevented only by discipline.
> **Mechanism.** Each state is an object; operations delegate to it.
> **Your example.** The vending machine in [10-lld-most-asked.md](10-lld-most-asked.md) LLD 5.
> **Cost.** "A class per state is a lot of files for a three-state machine. Below about four states I'd use an enum plus a transition table, which is clearer."

### 8. Saga
> **Problem.** A business operation spans services and there's no distributed transaction.
> **Mechanism.** A sequence of local transactions, each with a compensating action.
> **Your example.** [02-databases-advanced.md](02-databases-advanced.md) Q12.
> **Cost.** "Compensations are new business facts, not rollbacks — a refund isn't an un-charge, and some actions have no compensation at all. There's a visible window of inconsistency. It's the price of not having distributed transactions, not a free win."

---

## 🧭 SOLID, said like an engineer

You will be asked. Answer with the **smell each one prevents**, not the definition.

| Principle | 🗣️ Say this |
|---|---|
| **S**ingle Responsibility | "A class should have one reason to change. The smell is a class you modify for unrelated reasons — my repository shouldn't change because an email template changed." |
| **O**pen/Closed | "Adding a feature should mean adding code, not editing existing code — because editing means re-testing everything it touched. The Strategy example is exactly this." |
| **L**iskov Substitution | "A subclass must honour the base class's contract. The classic violation is an override that raises `NotImplementedError` — the caller can't use it interchangeably, so the inheritance is a lie." |
| **I**nterface Segregation | "Don't force implementers to stub methods they don't need. If half your implementations raise on half the interface, that's two interfaces." |
| **D**ependency Inversion | "Depend on abstractions. Concretely: my service depends on a repository interface, not on SQLAlchemy — which is what makes it testable." |

⚠️ **Then add this — it's what separates recitation from judgement:**
> "I'd add that these are heuristics, not laws. Taken literally, Single Responsibility produces a hundred single-method classes that are individually clean and collectively unreadable. The version I actually apply is: optimise for the next person reading this, and split when a class has genuinely become two ideas — not because a rule says so."

---

## 🚩 Naming a pattern unprompted in a design round

You can use pattern vocabulary to compress explanation. Do it like this:

> ✅ *"I'll make the fee calculation a Strategy — an interface with one implementation per pricing model — because pricing rules are the thing most likely to change."*

Name it, define it in the same breath, justify it, move on. Three seconds.

> ❌ *"I'll apply the Strategy pattern here."* (and stop)

The second version assumes they'll fill in your reasoning, and if they disagree you've given them nothing to engage with.

⚠️ **Don't pattern-hunt.** Announcing six patterns in one design looks like scoring points rather than solving a problem. One or two, well justified. In fact the strongest move available is the opposite:

> 🗣️ *"I considered a Factory here, but there's only ever one implementation, so I'd keep it a plain constructor until there's a second."*

That's restraint, and restraint reads as experience.

---

## 🎤 Three full answers to rehearse out loud

**"What design patterns have you used recently?"**
> "The ones I use most are structural rather than clever — Repository and Dependency Injection, because they're what make a service testable. In the GitLab user service the service layer depends on a repository interface and gets its session injected, so the business logic tests run with no database and no mocking. Beyond that, Adapter for every third-party client so vendor types never reach the domain, and Strategy where I genuinely expect variation. I'd say my bias has shifted over time toward *fewer* patterns — I've spent more time removing unnecessary abstraction than adding it."

**"How do you decide when to introduce an abstraction?"**
> "The second time, not the first — and only if the second case is real rather than imagined. The first time I write something I don't yet know what varies, so an abstraction built then is usually the wrong seam. And a wrong abstraction is worse than duplication, because duplication is easy to see and easy to fix, whereas a bad abstraction gets built on top of. Sandi Metz's line is that duplication is far cheaper than the wrong abstraction, and I've found that's true. The exception is when I have real evidence about the direction of change — if I've already been told three pricing models are coming, I'll build the seam up front."

**"Tell me about a time a pattern made things worse."**
> Have a real one. If you don't, this shape works: an abstraction added for flexibility that never materialised, which cost every reader a file jump for years and was eventually deleted. What you learned: **the cost of an abstraction is paid continuously by everyone who reads the code; the benefit is only paid if the change you predicted actually arrives.** So the bar should be evidence, not imagination.

---

## ⚡ Rapid fire

| They ask | 🗣️ Say this |
|---|---|
| Favourite pattern? | Don't name one — say "the ones that make code testable: Repository and DI." Naming a favourite sounds like enthusiasm for patterns rather than for working software. |
| Singleton? | "I avoid it. It's global state with a nicer name, it makes tests order-dependent, and in Python a module already gives you a single cached instance." |
| MVC? | "Not really a backend pattern any more. For an API it's router for HTTP, service for business logic, repository for persistence, schema for the contract." |
| Composition over inheritance — why? | "Inheritance couples you to a base class's internals and is fixed at definition time. Composition is swappable at runtime and each piece is independently testable." |
| An anti-pattern you've seen? | "God object, anemic domain model — and the one I'd actually cite, premature abstraction." |
| Is DRY always right? | "No. DRY is about knowledge, not text. Two identical lines that change for *different reasons* should stay separate — deduplicating them couples two unrelated things." |
| Pattern vs principle? | "A principle is a goal, like 'depend on abstractions'. A pattern is one known solution shape. Principles always apply; patterns are situational." |
| What's YAGNI? | "You aren't gonna need it — the counterweight to pattern enthusiasm." |

---

**Next:** [12-rapid-fire-senior.md](12-rapid-fire-senior.md)

# 03 — Pydantic Deep Dive

---

## 📖 Vocabulary

> *Definition* is the precise meaning; *Say it in an interview* is the quotable one-liner.

| Term | Definition | Say it in an interview |
|---|---|---|
| **Model (`BaseModel`)** | A Pydantic class whose annotated attributes define a validated data structure; instantiation parses and validates input. | "A Pydantic model is a typed class that validates on construction." |
| **Field** | One attribute of a model with a type annotation and optional metadata/constraints via `Field()`. | "A field is one typed, validated attribute of a model." |
| **Coercion (lax mode)** | Automatic conversion of input to the declared type when unambiguous (`"42"` → `42`, `"true"` → `True`). Pydantic's default mode. | "Lax mode coerces compatible inputs like numeric strings; strict mode rejects them." |
| **Strict mode** | Validation mode in which values must already be of the exact declared type; no coercion. Enabled per field (`Strict`) or per model (`strict=True`). | "Strict mode disables type coercion when exactness matters." |
| **`ValidationError`** | The exception raised when validation fails; carries a list of error dicts with `type`, `loc`, `msg` and `input` for every failing field. | "`ValidationError` reports all failures at once with their locations." |
| **`pydantic-core`** | The Rust implementation of Pydantic v2's validation and serialization engine, compiled from a *core schema* generated at class-definition time. | "Pydantic v2 compiles each model into a Rust validator, which is why it's fast." |
| **Validator** | A user-defined function attached to a model that adds custom validation or transformation logic. | "Validators add business-format rules that types alone can't express." |
| **`field_validator`** | Decorator for a validator bound to one or more named fields; runs *after* type validation by default. | "`field_validator` checks one field after Pydantic has typed it." |
| **`model_validator`** | Decorator for a validator that sees the whole model (`mode="after"` receives the instance; `mode="before"` receives raw input) for cross-field rules. | "`model_validator` is for rules involving several fields." |
| **`before` / `after` mode** | Whether a validator executes before Pydantic's own type validation (sees raw input) or after it (sees typed values). | "Use `before` to normalize raw input, `after` to check typed values." |
| **Dataclass** | The stdlib `@dataclass` decorator that auto-generates `__init__`, `__repr__`, `__eq__` from annotated fields; performs no runtime validation. | "Dataclasses are typed containers without validation; Pydantic models validate." |
| **Serialization (dump)** | Converting a model instance to a dict (`model_dump`) or JSON string (`model_dump_json`), optionally in `mode="json"` for JSON-safe types. | "`model_dump` gives a dict, `model_dump_json` gives JSON, and `mode='json'` makes types JSON-safe." |
| **Alias** | An alternative external name for a field. `validation_alias` applies on input, `serialization_alias` on output, `alias` to both. | "Aliases let the wire format differ from the Python attribute name." |
| **`AliasChoices`** | A `validation_alias` value accepting any of several input names for one field, useful for backward compatibility. | "`AliasChoices` lets me accept the old and new field names at once." |
| **`ConfigDict`** | The v2 mechanism for per-model configuration: `extra`, `from_attributes`, `strict`, `populate_by_name`, etc. | "`model_config = ConfigDict(...)` is where model-wide behavior is set." |
| **`extra`** | Config controlling unknown input keys: `ignore` (drop, default), `forbid` (error), `allow` (keep in `model_extra`). | "I use `extra='forbid'` on request schemas to catch client typos." |
| **Request schema** | The Pydantic model describing what the API accepts for an operation; contains only client-supplied fields. | "The request schema is the input contract." |
| **Response schema** | The model describing what the API returns; acts as an allow-list that filters the output. | "The response schema is the output contract and prevents leaking fields." |
| **`response_model`** | The route parameter (or return annotation) that tells FastAPI which schema to validate and filter the response against. | "`response_model` validates and filters what the handler returns." |
| **Mass assignment** | A vulnerability where clients set fields they shouldn't (e.g., `is_admin`) because the input schema exposes them. | "Separate request schemas prevent mass-assignment by never exposing privileged fields." |
| **`SecretStr`** | A Pydantic string type whose `repr`/`str` are masked; the value is retrieved only via `get_secret_value()`. | "`SecretStr` stops secrets from being printed by accident." |
| **Backward compatibility** | The property that existing clients continue to work unchanged after the API evolves. | "Backward compatible means old clients keep working after my change." |
| **Breaking change** | A change that can cause existing clients to fail: renaming/removing fields, changing types or semantics, making optional fields required. | "A breaking change is anything that could make a current client fail." |
| **`Literal`** | A typing construct restricting a value to specific literal constants (`Literal["a", "b"]`). | "`Literal` is an inline enum for a handful of allowed values." |
| **`computed_field`** | A decorator marking a property to be included in serialization and JSON Schema as a derived field. | "`computed_field` exposes a derived value in the output." |
| **`field_serializer`** | Decorator to customize how a specific field is serialized on output. | "`field_serializer` customizes a field's output format." |
| **JSON Schema** | A standard vocabulary for describing the structure of JSON documents; Pydantic generates it and OpenAPI embeds it. | "Pydantic emits JSON Schema, which is what appears in the OpenAPI docs." |
| **Contract test** | An automated test verifying an API's interface (e.g., its OpenAPI document) hasn't changed incompatibly. | "Contract tests fail CI when the API shape changes in a breaking way." |
| **Deprecation** | Marking a field/endpoint as scheduled for removal while it still works, signaling clients to migrate. | "Deprecate first, remove in the next major version." |

---

## Q1. What problem does Pydantic solve?

### 🧒 Simple explanation
Data from the outside world (HTTP, files, databases, queues) arrives as loosely typed blobs. Without a validation layer, you write manual checks everywhere and still get surprises at runtime. Pydantic lets you *declare* the shape once and enforces it, with helpful errors.

**Without Pydantic:**
```python
if "age" in data and isinstance(data["age"], int) and 0 <= data["age"] < 150: ...
```
**With Pydantic:**
```python
class Person(BaseModel):
    age: int = Field(ge=0, lt=150)
```
And you also get JSON Schema, serialization, and IDE types for free.

---

## Q2. What is the difference between Pydantic models and Python dataclasses?

| | `@dataclass` | `pydantic.BaseModel` |
|---|---|---|
| Validates at runtime | ❌ type hints are ignored | ✅ |
| Type coercion (`"1"` → `1`) | ❌ | ✅ (lax mode) |
| JSON parse / dump | Manual | Built in |
| JSON Schema | ❌ | ✅ |
| Construction speed | Faster (no checks) | Slightly slower (but Rust-fast) |
| Use for | Internal, trusted data | Boundary, untrusted data |

Pydantic also offers `pydantic.dataclasses.dataclass` — a dataclass *with* validation — if you like the stdlib API.

### 🎤 Interview answer
> "Dataclasses are containers; Pydantic models are *validated* containers. I use dataclasses for internal domain objects that are already trustworthy and Pydantic at the edges: API bodies, config, queue messages."

---

## Q3. How does Pydantic perform validation?

### 🧒 Simple explanation
1. **When the class is defined**, Pydantic reads the annotations and builds a *validation plan* (a "core schema") that's compiled into `pydantic-core` (Rust).
2. **When you create an instance**, the Rust code walks the plan: checks each field's type, coerces if allowed, applies constraints, runs your custom validators, and collects *all* errors.
3. If any error exists, it raises one `ValidationError` with the full list.

**Order for each field:** `before` validators → built-in type validation → `after` validators. Then `model_validator(mode="after")` for cross-field checks.

### 💻 Code
```python
from pydantic import BaseModel, Field, ValidationError

class Person(BaseModel):
    age: int = Field(ge=0, lt=150)

try:
    Person.model_validate({"age": "abc"})
except ValidationError as e:
    print(e.errors())
# [{'type': 'int_parsing', 'loc': ('age',), 'msg': 'Input should be a valid integer, ...', 'input': 'abc'}]
```
- `Person.model_validate(dict)` — build from a dict (also `Person(age="abc")` or `Person.model_validate_json('{"age": 1}')`).
- `e.errors()` — the structured list of problems.

---

## Q4. How do you create custom validators?

### 💻 Code
```python
from datetime import date
from pydantic import BaseModel, field_validator, model_validator

class Booking(BaseModel):
    start: date
    end: date
    guests: int

    @field_validator("guests")
    @classmethod
    def guests_positive(cls, v: int) -> int:
        if v <= 0:
            raise ValueError("guests must be > 0")
        return v

    @model_validator(mode="after")
    def end_after_start(self) -> "Booking":
        if self.end <= self.start:
            raise ValueError("end must be after start")
        return self
```
### 🔍 Line by line
- `@field_validator("guests")` — attach a validator to the `guests` field. Default mode is `"after"`, so `v` is already an `int`.
- `@classmethod` — required in Pydantic v2; the first argument is the class (`cls`), not an instance.
- `def guests_positive(cls, v: int) -> int:` — receives the value, must **return** the (possibly transformed) value.
- `raise ValueError(...)` — Pydantic converts `ValueError` (and `AssertionError`) into a validation error with your message. Don't raise other exception types.
- `@model_validator(mode="after")` — runs once *all* fields are valid. Method receives `self` (the built instance) — use for rules involving multiple fields.
- `return self` — must return the instance.

Use `mode="before"` when you need to touch **raw input** before conversion (e.g., strip whitespace from `"  2024-01-01 "` before it's parsed as a date).

---

## Q5. How do you validate nested objects?

Nested models validate automatically. Cross-object rules go on the parent:

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
- Each `Item` in the list is validated first (so `price`/`qty` are trustworthy by the time `total_cap` runs).
- `sum(... for i in self.items)` — a generator expression summing line totals.

---

## Q6. How do you make a field optional?

Two different ideas that people mix up:
- **Not required** = has a default value.
- **Nullable** = accepts `None`.

```python
class Profile(BaseModel):
    nickname: str | None = None   # optional AND nullable (most common)
    bio: str = ""                 # optional, NOT nullable (null → error)
    age: int | None               # REQUIRED but nullable (must send "age": null or a number)
```
⚠️ In Pydantic v2, `Optional[X]` **without** a default is *required*. That surprises people coming from v1.

---

## Q7. How do you handle extra/unknown fields?

```python
from pydantic import BaseModel, ConfigDict

class Strict(BaseModel):
    model_config = ConfigDict(extra="forbid")   # unknown keys → validation error

class Lenient(BaseModel):
    model_config = ConfigDict(extra="ignore")   # default: silently dropped

class Passthrough(BaseModel):
    model_config = ConfigDict(extra="allow")    # kept; available via .model_extra
```
**Recommendation:** `forbid` on your *request* schemas — it catches client typos like `"emial"` instantly. `ignore` on schemas that parse *third-party* payloads so their new fields don't break you.

---

## Q8. How do you serialize a Pydantic model?

```python
u = UserRead(id=1, email="a@b.com", name="A")

u.model_dump()                   # dict; Python types kept (datetime stays datetime)
u.model_dump(mode="json")        # dict with JSON-safe values (datetime → ISO string)
u.model_dump_json()              # JSON string
u.model_dump(exclude={"email"})  # drop fields
u.model_dump(by_alias=True)      # use aliases as keys
u.model_dump(exclude_none=True)  # drop fields whose value is None
```
Per-field custom output: `@field_serializer("created_at")`. Derived output fields: `@computed_field`.

---

## Q9. How do you convert a Pydantic model to a dictionary?

`model.model_dump()` (v2). The v1 name `.dict()` is deprecated.
Useful options: `include`, `exclude`, `exclude_unset` (only fields the client explicitly sent — essential for PATCH), `exclude_defaults`, `exclude_none`, `by_alias`, `mode`.

---

## Q10. What is the difference between request models and response models?

- **Request model** — what you *accept*. Contains client-provided fields (`password`). Should be strict (`extra="forbid"`).
- **Response model** — what you *return*. Contains server-generated fields (`id`, `created_at`) and **omits** secrets. Declared with `response_model=` or the return annotation; FastAPI filters output to exactly those fields.

```python
@app.post("/users", response_model=UserRead, status_code=201)
async def create_user(body: UserCreate, svc: UserService = Depends()):
    user = await svc.create(body)   # ORM object that has hashed_password
    return user                     # FastAPI serialises ONLY UserRead's fields
```

---

## Q11. Why should you define separate request and response schemas?

1. **Security** — the response schema is an *allow-list*; you can't accidentally leak `hashed_password`.
2. **Asymmetry** — clients never send `id`; servers never return `password`.
3. **Independent evolution** — add output fields without touching the input contract.
4. **Clear docs** — consumers see exactly what to send and what they'll get.
5. **Prevents mass assignment** — a shared model with `is_admin` would let a client set it on creation.

---

## Q12. How do you prevent sensitive fields from appearing in API responses?

Layers, strongest first:
1. **Response model allow-list** — `response_model=UserRead` where `UserRead` simply has no secret fields.
2. **Never define secrets in output schemas** — safer than remembering to `exclude`.
3. **`response_model_exclude={"ssn"}`** on the route for one-off cases.
4. **`SecretStr`** for values that must live in memory but never print:
   ```python
   from pydantic import SecretStr
   class Settings(BaseModel):
       api_key: SecretStr        # repr shows '**********'; real value via .get_secret_value()
   ```
5. **Log redaction** middleware (file 06) so secrets don't escape through logs.

---

## Q13. How do you handle backward compatibility when changing schemas?

Safe vs breaking:
- ✅ Add an **optional** request field.
- ✅ Add a response field (clients ignore unknown keys).
- ❌ Rename, remove, retype a field, or make optional → required.

Transition technique — accept both names:
```python
from pydantic import AliasChoices, Field

class Order(BaseModel):
    customer_id: int = Field(validation_alias=AliasChoices("customer_id", "customerId"))
```
- `validation_alias=AliasChoices(...)` — accept either key on input; the attribute is always `customer_id`.
- Mark old fields `Field(deprecated=True)` (Pydantic ≥ 2.7) so docs show it.
- For real breaks: introduce a new API version; dual-write during migration; remove after clients migrate.
- Add **contract tests** that diff the OpenAPI document against a stored snapshot and fail CI on breaking changes.

---

## Q14. How would you validate an enum field?

```python
from enum import Enum
from typing import Literal

class OrderStatus(str, Enum):
    pending = "pending"
    paid = "paid"
    shipped = "shipped"

class OrderUpdate(BaseModel):
    status: OrderStatus

class QuickUpdate(BaseModel):
    status: Literal["pending", "paid"]     # no Enum class needed
```
- Input `"paid"` → `OrderStatus.paid`; `"refunded"` → 422 listing allowed values.
- **State transitions** ("shipped can't go back to pending") belong in the *service layer*, not the schema: the schema knows the value is allowed; only business logic knows whether the *transition* is.

---

## Q15. How would you handle different API versions with different schemas?

Keep each version as its own package with its own schemas, sharing the domain layer:

```
app/
  api/
    v1/schemas.py   # UserReadV1: {id, name}
    v1/routes.py
    v2/schemas.py   # UserReadV2: {id, first_name, last_name}
    v2/routes.py
  domain/           # ORM + services, version-agnostic
```
```python
app.include_router(v1_router, prefix="/api/v1")
app.include_router(v2_router, prefix="/api/v2")
```
```python
# v1 adapts the new domain shape back to the old contract
class UserReadV1(BaseModel):
    id: int
    first_name: str = Field(exclude=True)
    last_name: str = Field(exclude=True)

    @computed_field
    @property
    def name(self) -> str:
        return f"{self.first_name} {self.last_name}"
```
- `Field(exclude=True)` — read from the ORM object but don't output.
- `@computed_field` + `@property` — `name` is calculated and included in output.
- When v1 traffic drops to zero (measured with metrics), delete the package.

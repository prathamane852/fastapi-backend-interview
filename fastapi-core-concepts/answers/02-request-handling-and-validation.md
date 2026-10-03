# 02 — Request Handling & Validation

---

## 📖 Vocabulary

> *Definition* is the precise meaning; *Say it in an interview* is the quotable one-liner.

| Term | Definition | Say it in an interview |
|---|---|---|
| **Path parameter** | A variable segment of the URL path, declared with braces in the route template (`/users/{user_id}`), that identifies a specific resource. Always required. | "Path parameters identify *which* resource; they're part of the URL and always required." |
| **Query parameter** | A key/value pair in the URL query string after `?` (`?page=2&size=20`), used to filter, sort or paginate; typically optional. | "Query parameters modify *how* you fetch a resource: filtering, sorting, paging." |
| **Request body** | The payload of an HTTP request, carried after the headers; typically JSON for APIs. Used with POST, PUT and PATCH. | "The body carries the data the client is sending, usually JSON, validated by a Pydantic model." |
| **Header** | A name/value metadata field in an HTTP request or response (`Content-Type`, `Authorization`). Names are case-insensitive. | "Headers carry metadata about the request or response, like content type and credentials." |
| **`Annotated`** | `typing.Annotated[T, meta...]` attaches arbitrary metadata to a type hint without changing the type. FastAPI reads `Query()`, `Path()`, `Body()`, `Depends()` from it. | "`Annotated` lets me attach FastAPI metadata to a parameter's type while keeping the type itself clean." |
| **`Path()` / `Query()` / `Body()` / `Header()` / `Cookie()` / `File()` / `Form()`** | FastAPI parameter declarators that state the *source* of a value and attach validation constraints and documentation. | "These declarators tell FastAPI where a parameter comes from and what rules it must satisfy." |
| **`Field()`** | Pydantic's function for attaching constraints, defaults, aliases and documentation to a model field. | "`Field` adds validation rules and docs metadata to a Pydantic model field." |
| **Constraint** | A rule a value must satisfy beyond its type: numeric bounds, string length, regex pattern, collection size. | "Constraints are the extra rules on top of the type, like ranges and lengths." |
| **`gt` / `ge` / `lt` / `le`** | Numeric constraints: greater-than, greater-or-equal, less-than, less-or-equal. | "`gt`/`ge`/`lt`/`le` are the numeric bound constraints." |
| **Regular expression (regex / pattern)** | A formal pattern language for matching strings. `^` and `$` anchor start/end; `[a-z0-9_]+` means one or more of those characters. | "A regex is a pattern that a string must match; I use it for formats like usernames or SKUs." |
| **Enum** | A class defining a fixed, named set of constant values (`enum.Enum`). Subclassing `str, Enum` makes members JSON-serializable strings. | "An Enum restricts a field to a fixed set of allowed values and documents them in OpenAPI." |
| **UUID** | *Universally Unique Identifier*: a 128-bit identifier, usually shown as 32 hex digits in 8-4-4-4-12 groups; v4 is random. | "A UUID is a 128-bit globally unique ID, safe to generate without coordination." |
| **Nested model** | A Pydantic model used as the type of a field in another model; validation recurses into it. | "Nested models let me describe hierarchical JSON, and validation recurses automatically." |
| **Default value** | The value assigned to a field or parameter when the client omits it; its presence makes the field optional. | "A default makes a field optional and defines what it is when omitted." |
| **`default_factory`** | A zero-argument callable Pydantic invokes to produce a fresh default per instance; required for mutable or computed defaults. | "`default_factory` builds a new default each time, which is what you want for lists or timestamps." |
| **MIME type / media type** | A two-part identifier for content format (`type/subtype`), e.g., `application/json`, `image/png`, sent in `Content-Type`. | "The media type tells the receiver how to interpret the bytes." |
| **Multipart form data** | The `multipart/form-data` encoding that carries files and fields in one request body, separated by boundaries. | "File uploads use multipart encoding, which is why FastAPI needs `python-multipart`." |
| **`UploadFile`** | FastAPI's wrapper over a spooled temporary file for uploads: memory-backed when small, disk-backed when large, with async `read`/`write`. | "`UploadFile` gives me an async file handle that spills to disk for large uploads." |
| **Streaming** | Processing data incrementally in chunks as it is produced or received, keeping memory usage bounded. | "Streaming means I never hold the whole payload in memory at once." |
| **Walrus operator `:=`** | Assignment expression (PEP 572): assigns and returns a value in one expression, e.g., `while chunk := await f.read(n):`. | "The walrus operator assigns inside an expression, handy for read-until-empty loops." |
| **Status code 201** | *Created.* The request succeeded and a new resource was created; commonly paired with a `Location` header. | "201 means 'created', which is the right code for a successful POST that made a resource." |
| **Status code 204** | *No Content.* Success with no response body; typical for DELETE. | "204 is success with an empty body." |
| **Status code 413 / 415 / 422** | 413 *Payload Too Large*; 415 *Unsupported Media Type*; 422 *Unprocessable Entity* (well-formed but semantically invalid; FastAPI's validation error). | "422 is FastAPI's validation failure; 413 and 415 reject oversized or wrong-type payloads." |
| **`HTTPException`** | FastAPI's exception that, when raised, is converted into an HTTP error response with the given status code, detail and headers. | "Raising `HTTPException` is how a handler returns an error response cleanly." |
| **`Response` object** | Starlette's response class; injecting it as a parameter lets a handler set status code, headers or cookies while still returning data. | "I inject `Response` when I need to change the status or headers but keep returning a model." |
| **`RootModel`** | A Pydantic v2 model whose entire value is a single non-object type (a list or scalar), accessible via `.root`; used to validate top-level arrays. | "`RootModel` lets me put validators on a top-level list body." |
| **Generic model** | A Pydantic model parameterized with `TypeVar`s (`Page[T]`) so one definition serves many item types. | "A generic model like `Page[T]` gives one pagination envelope for every resource." |
| **`from_attributes`** | Pydantic config option allowing a model to be built from any object exposing matching attributes (e.g., an ORM row), not only from dicts. | "`from_attributes=True` lets me build a response schema straight from an ORM object." |
| **`exclude_unset`** | A `model_dump` option that outputs only fields the client explicitly provided, distinguishing 'omitted' from 'set to default'. | "`exclude_unset` is how I apply a PATCH without overwriting fields the client didn't send." |

---

## Q1. How do you define path parameters in FastAPI?

### 🧒 Simple explanation
Put a name in curly braces in the path, and declare a function parameter with the same name. To add rules or documentation, wrap the type with `Annotated[..., Path(...)]`.

### 💻 Code
```python
from typing import Annotated
from fastapi import FastAPI, Path

app = FastAPI()

@app.get("/orders/{order_id}")
async def get_order(
    order_id: Annotated[int, Path(gt=0, description="Numeric order ID")]
):
    return {"order_id": order_id}
```
### 🔍 Line by line
- `from typing import Annotated` — `Annotated` lets us attach extra info to a type.
- `from fastapi import FastAPI, Path` — `Path` is the helper that declares a path parameter with rules.
- `"/orders/{order_id}"` — placeholder named `order_id`.
- `order_id: Annotated[int, Path(gt=0, description=...)]` — read as: "`order_id` is an `int`, taken from the path, must be greater than 0, and here's its description for the docs".
- If a client requests `/orders/0` → 422 with a message about `gt`.

### ⚠️ Ordering gotcha
```python
@app.get("/users/me")          # must come FIRST
@app.get("/users/{user_id}")   # otherwise "me" is captured here and fails int parsing
```
Routes are matched top to bottom; a fixed path must be declared before a variable one that could swallow it.

---

## Q2. How do you validate path parameters?

### 🧒 Simple explanation
Three tools: numeric/string constraints in `Path()`, a Python `Enum` for a fixed set of values, or a specialised type like `UUID`.

### 💻 Code — enum
```python
from enum import Enum

class Region(str, Enum):
    us = "us"
    eu = "eu"

@app.get("/regions/{region}/status")
async def region_status(region: Region):
    return {"region": region.value}
```
### 🔍 Line by line
- `class Region(str, Enum):` — Inheriting from **both** `str` and `Enum` means each member *is* a string (so it serialises to JSON naturally) *and* a fixed choice. The docs show a dropdown.
- `us = "us"` — Member name `us`, value `"us"`.
- `region: Region` — FastAPI converts `"eu"` → `Region.eu`. Anything else → 422 listing the allowed values.
- `region.value` — the underlying string.

### 💻 Code — UUID
```python
import uuid

@app.get("/items/{item_id}")
async def get_item(item_id: uuid.UUID): ...
```
`uuid.UUID` validates the format (`8-4-4-4-12` hex) automatically and gives you a `UUID` object.

---

## Q3. How do you define optional query parameters?

### 💻 Code
```python
from fastapi import Query

@app.get("/search")
async def search(
    q: str | None = None,
    tags: Annotated[list[str] | None, Query()] = None,
    limit: Annotated[int, Query(ge=1, le=100)] = 20,
):
    ...
```
### 🔍 Line by line
- `q: str | None = None` — Optional: if absent, `q` is `None`.
- `tags: Annotated[list[str] | None, Query()] = None` — A *list* query parameter. **You must write `Query()` explicitly** for lists, otherwise FastAPI would think a list means a JSON body. Client sends `?tags=a&tags=b` → `["a", "b"]`.
- `limit: Annotated[int, Query(ge=1, le=100)] = 20` — Optional with default 20, but if sent it must be between 1 and 100.

---

## Q4. What is the difference between path and query parameters?

| | Path | Query |
|---|---|---|
| Looks like | `/users/42` | `/users?page=2` |
| Purpose | Identifies **which** resource | Modifies **how** you fetch (filter, sort, page) |
| Required? | Always | Usually optional |
| Part of resource identity? | Yes | No |

**Rule of thumb:** if removing it would make the URL point to a *different thing*, it's a path parameter. If it just tweaks the result list, it's a query parameter.

---

## Q5. How do you define request bodies?

### 💻 Code
```python
from pydantic import BaseModel

class Address(BaseModel):
    street: str
    city: str

class CustomerCreate(BaseModel):
    name: str
    email: str
    address: Address

@app.post("/customers", status_code=201)
async def create_customer(payload: CustomerCreate):
    return payload
```
### 🔍 Line by line
- `class Address(BaseModel)` — a small model.
- `address: Address` — a **nested** model field: the JSON must contain an object under `"address"` with `street` and `city`.
- `payload: CustomerCreate` — a parameter typed as a Pydantic model → read from the JSON body. Expected JSON:
  ```json
  {"name": "Ada", "email": "ada@x.com", "address": {"street": "1 Main", "city": "London"}}
  ```
- `return payload` — returning a model: FastAPI serialises it to JSON.

---

## Q6. How do you accept multiple body parameters?

### 🧒 Simple explanation
If a handler has **two** model parameters, FastAPI expects one JSON object whose keys are the parameter names.

### 💻 Code
```python
from fastapi import Body

@app.post("/orders")
async def create_order(
    customer: CustomerCreate,
    item: ItemCreate,
    priority: Annotated[int, Body(ge=1, le=5)] = 3,
):
    ...
```
Expected body:
```json
{"customer": {...}, "item": {...}, "priority": 2}
```
### 🔍 Line by line
- `customer: CustomerCreate`, `item: ItemCreate` — two models → each becomes a top-level key.
- `priority: Annotated[int, Body(ge=1, le=5)] = 3` — A plain `int` would normally be a *query* parameter; wrapping it in `Body()` says "take it from the JSON body instead". Default 3, must be 1–5.

To force a **single** model to be nested under its name: `payload: Annotated[ItemCreate, Body(embed=True)]` → body is `{"payload": {...}}`.

---

## Q7. How do you handle nested Pydantic models?

### 💻 Code
```python
from pydantic import BaseModel, Field

class LineItem(BaseModel):
    sku: str
    qty: int = Field(gt=0)

class OrderCreate(BaseModel):
    customer_id: int
    items: list[LineItem] = Field(min_length=1)
    metadata: dict[str, str] = {}
```
### 🔍 Line by line
- `qty: int = Field(gt=0)` — integer, must be > 0.
- `items: list[LineItem] = Field(min_length=1)` — a list of `LineItem` objects, at least one. Each element is validated as a `LineItem`.
- `metadata: dict[str, str] = {}` — a free-form object with string keys and string values; default empty. (Pydantic copies defaults per instance, so a mutable default is safe here.)
- Validation is **recursive**. An error deep inside is reported with the full location: `["body", "items", 0, "qty"]` = first item's `qty`.

---

## Q8. How do you define default values?

```python
from datetime import datetime, timezone
from pydantic import BaseModel, Field

class Post(BaseModel):
    title: str
    role: str = "user"
    tags: list[str] = Field(default_factory=list)
    created_at: datetime = Field(default_factory=lambda: datetime.now(timezone.utc))
```
- `role: str = "user"` — a simple default.
- `tags: list[str] = Field(default_factory=list)` — `default_factory` calls `list()` for each new instance. Clearer intent than `= []`.
- `created_at: ... default_factory=lambda: datetime.now(timezone.utc)` — a *computed* default evaluated at creation time. `lambda:` is an anonymous function with no arguments. `timezone.utc` makes it timezone-aware (always store UTC).
- For handler parameters, a default in the signature does the same: `page: int = 1`.

---

## Q9. How do you validate string length?

```python
class Signup(BaseModel):
    username: str = Field(min_length=3, max_length=30, pattern=r"^[a-z0-9_]+$")
```
- `min_length` / `max_length` — inclusive bounds on character count.
- `pattern=r"^[a-z0-9_]+$"` — a regex. `r"..."` is a *raw string* so backslashes aren't interpreted by Python. `^` start, `[a-z0-9_]+` one or more of those characters, `$` end.
- Same keywords work in `Query()`/`Path()` for parameters.
- For normalising text: `Annotated[str, StringConstraints(strip_whitespace=True, to_lower=True)]` from `pydantic`.

---

## Q10. How do you validate numeric ranges?

```python
class Product(BaseModel):
    price: float = Field(gt=0, le=1_000_000)
    stock: int = Field(ge=0)
    discount: float = Field(ge=0, lt=1, default=0)
```
- `gt=0, le=1_000_000` → `0 < price ≤ 1,000,000`. (`1_000_000` is just `1000000` with readability underscores.)
- `ge=0` → `stock ≥ 0`.
- `ge=0, lt=1` → `0 ≤ discount < 1`.
- `multiple_of=0.01` is useful for currency amounts.

---

## Q11. How do you validate email addresses?

```python
from pydantic import BaseModel, EmailStr

class Signup(BaseModel):
    email: EmailStr
```
- `EmailStr` — a special string type that checks email *syntax* (needs `pip install "pydantic[email]"`, which pulls in the `email-validator` package).
- It proves the *format* is valid, not that the mailbox exists. For that you send a verification email with a one-time link.

---

## Q12. How do you create reusable Pydantic models?

### 🧒 Simple explanation
Use **inheritance**: put shared fields in a base class, and make specialised classes for creating, reading and updating.

### 💻 Code
```python
from pydantic import BaseModel, ConfigDict, EmailStr, Field

class UserBase(BaseModel):
    email: EmailStr
    name: str

class UserCreate(UserBase):
    password: str = Field(min_length=8)

class UserRead(UserBase):
    id: int
    model_config = ConfigDict(from_attributes=True)

class UserUpdate(BaseModel):
    email: EmailStr | None = None
    name: str | None = None
```
### 🔍 Line by line
- `UserBase` — fields every variant shares.
- `UserCreate(UserBase)` — inherits `email`, `name`; adds `password`. Input only.
- `UserRead(UserBase)` — adds `id`; **has no `password`** so it can never leak one. Output only.
- `model_config = ConfigDict(from_attributes=True)` — allows building the model from any object with matching *attributes* (like a SQLAlchemy row), not only from dicts: `UserRead.model_validate(orm_user)`.
- `UserUpdate` — all fields optional, for PATCH. Apply with `patch.model_dump(exclude_unset=True)` to get only the fields the client actually sent.

### 💻 Generic wrapper (reusable across resources)
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
- `TypeVar("T")` — a placeholder type. `Generic[T]` makes `Page` parameterisable: `Page[UserRead]`, `Page[OrderRead]`.

---

## Q13. How do you handle list and dictionary request bodies?

```python
@app.post("/items/bulk")
async def bulk_create(items: list[ItemCreate]):
    return {"count": len(items)}

@app.post("/settings")
async def set_settings(settings: dict[str, int]):
    return settings
```
- `items: list[ItemCreate]` — the body itself is a JSON **array** `[{...}, {...}]`.
- `settings: dict[str, int]` — the body is a JSON object with arbitrary keys and integer values.

To add validators to such shapes use a `RootModel`:
```python
from pydantic import RootModel, field_validator

class ItemList(RootModel[list[ItemCreate]]):
    @field_validator("root")
    @classmethod
    def not_empty(cls, v):
        if not v:
            raise ValueError("must not be empty")
        return v
```
- `RootModel[list[ItemCreate]]` — a model whose *whole value* is a list (no wrapper key). The value is accessible as `.root`.
- `@field_validator("root")` — validate that value. `@classmethod` is required by Pydantic v2 validators.

---

## Q14. How do you upload files using FastAPI?

### 💻 Code
```python
from typing import Annotated
from fastapi import UploadFile, File, HTTPException
import aiofiles

MAX_BYTES = 10 * 1024 * 1024

@app.post("/upload")
async def upload(file: Annotated[UploadFile, File()]):
    if file.content_type not in {"image/png", "image/jpeg"}:
        raise HTTPException(415, "Unsupported media type")

    size = 0
    async with aiofiles.open(f"/data/{file.filename}", "wb") as out:
        while chunk := await file.read(1024 * 1024):
            size += len(chunk)
            if size > MAX_BYTES:
                raise HTTPException(413, "File too large")
            await out.write(chunk)

    return {"filename": file.filename, "bytes": size}
```
### 🔍 Line by line
- `import aiofiles` — async file I/O library (`pip install aiofiles`). Normal `open()` blocks the event loop.
- `MAX_BYTES = 10 * 1024 * 1024` — 10 MiB limit.
- `file: Annotated[UploadFile, File()]` — declares a multipart file upload (needs `python-multipart` installed). `UploadFile` is a *spooled* file: kept in memory when small, spilled to disk when large.
- `file.content_type` — the MIME type the *client* claimed. Useful as a first filter, but don't trust it for security — check the file's first bytes ("magic numbers") if it matters.
- `raise HTTPException(415, ...)` — return an error response immediately.
- `async with aiofiles.open(..., "wb") as out:` — open the destination for writing bytes, asynchronously. `async with` ensures it's closed even on error.
- `while chunk := await file.read(1024 * 1024):` — the *walrus operator* `:=` assigns and tests in one step: read up to 1 MiB; if empty (end of file), stop. Streaming in chunks keeps memory flat.
- `size += len(chunk)` / `if size > MAX_BYTES: raise …` — abort early on oversize instead of reading everything.
- `await out.write(chunk)` — write the chunk without blocking.

For multi-GB files don't upload through the API at all: give the client a **pre-signed S3 URL** (see file 16).

---

## Q15. How do you return a custom HTTP status code?

```python
from fastapi import status, Response, HTTPException
from fastapi.responses import JSONResponse

# 1. Fixed default for the route
@app.post("/items", status_code=status.HTTP_201_CREATED)
async def create(): ...

# 2. Decide at runtime via the Response object
@app.put("/items/{id}")
async def upsert(id: int, item: ItemCreate, response: Response):
    created = repo.upsert(id, item)
    response.status_code = 201 if created else 200
    return item

# 3. Return a Response directly (skips response_model processing!)
@app.get("/legacy")
async def legacy():
    return JSONResponse(status_code=202, content={"queued": True})

# 4. Errors
raise HTTPException(status_code=404, detail="Item not found")
```
- `status.HTTP_201_CREATED` — a named constant (=201); clearer than a magic number.
- `response: Response` — declare a parameter of type `Response` and FastAPI injects a mutable response object; setting `.status_code` on it changes the outgoing status while still returning normal data.
- `JSONResponse(...)` — building the response yourself gives full control but bypasses `response_model` filtering — be careful not to leak fields.
- `HTTPException` — raise it anywhere; FastAPI turns it into `{"detail": "..."}` with that status.

### ⚠️ Follow-ups
- *"200 vs 201 vs 204?"* → 200 OK with body; 201 Created (usually with `Location` header); 204 No Content (e.g., after DELETE).

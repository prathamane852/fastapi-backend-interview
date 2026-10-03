# 09 — Project Structure & Architecture

---

## 📖 Vocabulary

> *Definition* is the precise meaning; *Say it in an interview* is the quotable one-liner.

| Term | Definition | Say it in an interview |
|---|---|---|
| **Layered architecture** | Organizing code into layers with single responsibilities (presentation, application/service, domain, data access), where dependencies point in one direction. | "Layers depend downward only: API → service → repository → models." |
| **Feature (vertical-slice) module** | A package grouping everything for one business capability (models, schemas, repo, service, router) rather than grouping by technical type. | "I organize by feature so a change touches one folder." |
| **Separation of concerns** | Design principle that each module addresses a distinct concern with minimal overlap. | "Each layer has one job; HTTP details never leak into business logic." |
| **Service layer** | The application layer implementing use cases and orchestrating domain objects and repositories; owns transactions; no HTTP knowledge. | "The service layer is where business rules live and where transactions begin and end." |
| **Repository** | An abstraction that mediates between the domain and data mapping layers using collection-like, intent-revealing methods. | "Repositories hide SQL behind names like `find_active_by_email`." |
| **Domain exception** | An exception type expressing a business failure (`InsufficientStock`, `NotFound`), raised by services and translated to HTTP by one handler. | "Services raise domain exceptions; a single handler maps them to status codes." |
| **App factory** | A function (`create_app()`) that constructs and configures the application instance, enabling multiple instances with different settings. | "An app factory lets tests build isolated apps with test settings." |
| **`APIRouter`** | A FastAPI object grouping related routes with shared prefix, tags and dependencies, later included into the app. | "Routers modularize endpoints and let me attach shared dependencies." |
| **Configuration** | Externalized values that vary by environment (URLs, credentials, feature toggles), kept out of code. | "Config varies by environment; code doesn't." |
| **Environment variable** | A key/value pair in a process's environment, inherited from its parent; the 12-factor way to pass config. | "Env vars are the standard way to inject config into containers." |
| **Twelve-Factor App** | A methodology for cloud-native apps: config in the environment, stateless processes, build/release/run separation, logs as streams, etc. | "12-factor: one build, config from the environment, stateless processes." |
| **Secret** | Sensitive configuration (passwords, keys, tokens) requiring encryption at rest, access control, rotation and audit. | "Secrets never live in git, images or logs." |
| **Secrets manager** | A service storing secrets encrypted, with IAM-controlled access, versioning, rotation and audit logs (AWS Secrets Manager, Vault). | "The secrets manager is the source of truth; pods fetch with an IAM role." |
| **IRSA** | *IAM Roles for Service Accounts*: EKS mechanism binding a Kubernetes ServiceAccount to an AWS IAM role via OIDC, giving pods credentials without static keys. | "IRSA gives each pod its own IAM role, no access keys." |
| **API versioning** | Maintaining multiple concurrent API contracts so clients can migrate on their own schedule. | "Version only on breaking changes; keep additive changes in place." |
| **URI vs header versioning** | Version in the path (`/v2/...`) vs negotiated via `Accept` or a custom header. | "URI versioning is explicit and cache-friendly; header versioning keeps URLs stable." |
| **Semantic versioning** | `MAJOR.MINOR.PATCH`: major for breaking changes, minor for additive, patch for fixes. | "SemVer signals compatibility: bump major only for breaking changes." |
| **Backward compatibility** | New versions accept old inputs and produce outputs old clients can consume. | "Backward compatible means old clients keep working." |
| **Circular import** | Two modules importing each other at module load, causing partially initialized modules or `ImportError`. | "Cycles mean a lower layer imports a higher one; fix the dependency direction." |
| **`TYPE_CHECKING`** | `typing.TYPE_CHECKING` is `False` at runtime and `True` for type checkers; imports guarded by it exist only for annotations. | "`TYPE_CHECKING` imports break cycles that exist only for type hints." |
| **`import-linter`** | A tool enforcing architectural import rules (layer contracts, forbidden imports) in CI. | "import-linter keeps the architecture from eroding." |
| **Shared/internal library** | A versioned package of cross-service code (auth verification, logging, tracing) published to a private index. | "Common plumbing goes into a versioned internal library." |
| **Monolith** | A single deployable unit containing all functionality. | "A monolith is one deployable; simple ops, one transaction boundary." |
| **Modular monolith** | A monolith with enforced internal module boundaries and explicit interfaces between modules. | "A modular monolith has microservice-style boundaries without the network." |
| **Microservices** | An architecture of small, independently deployable services, each owning its data and communicating over the network. | "Microservices buy independent deployment at the cost of distributed-systems complexity." |
| **Bounded context** | A DDD boundary within which a particular domain model and ubiquitous language apply consistently. | "Bounded contexts are the natural seams for splitting services." |
| **Strangler fig pattern** | Incrementally replacing a legacy system by routing functionality piece by piece to new services until the old one can be retired. | "Strangler fig: route one capability at a time to the new service." |
| **Saga** | A pattern for distributed transactions: a sequence of local transactions with compensating actions for rollback. | "Sagas replace distributed transactions with compensations." |
| **Feature flag** | A runtime configuration switch enabling or disabling behavior without deployment. | "Flags decouple deploy from release." |
| **Consumer-driven contract test** | Tests written from the consumer's expectations that the provider must satisfy (e.g., Pact). | "Consumer-driven contracts catch breaking changes before deploy." |
| **Event-driven communication** | Services publish domain events to a broker; others subscribe asynchronously, decoupling availability and deploy cadence. | "Events decouple services so one deploy can't break another." |

---

## Q1. How would you structure a large FastAPI project?

### 🧒 Simple explanation
Organise **by feature**, not by technical type. With 40 entities, a `routers/` folder with 40 files plus `models/` with 40 files means touching four folders for every change. A feature folder keeps `orders/` together.

```
app/
├── main.py                 # create_app(), lifespan, middleware, include routers
├── core/
│   ├── config.py           # Settings (pydantic-settings)
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
│       └── router.py       # aggregates feature routers
├── features/
│   ├── users/
│   │   ├── models.py       # ORM
│   │   ├── schemas.py      # Pydantic in/out
│   │   ├── repository.py
│   │   ├── service.py
│   │   └── router.py
│   ├── orders/             # same shape
│   └── payments/
├── integrations/           # external clients: stripe.py, s3.py, sqs.py
├── workers/                # queue consumers
└── tests/
    ├── conftest.py
    ├── unit/
    └── integration/
```
```python
# app/main.py
def create_app() -> FastAPI:
    app = FastAPI(title=settings.app_name, lifespan=lifespan)
    register_middleware(app)
    register_exception_handlers(app)
    app.include_router(api_v1_router, prefix="/api/v1")
    return app

app = create_app()
```
- `create_app()` — **app factory**: tests can build a fresh app with different settings.
- `app = create_app()` — module-level instance for Uvicorn (`app.main:app`).

## Q2. Why shouldn't all business logic be inside route handlers?

- **Untestable without HTTP** — every rule needs a test client.
- **Not reusable** — a CLI command or queue consumer needs the same logic.
- **Mixed concerns** — status codes and header parsing tangle with domain rules.
- **Scattered transactions** — commits in handlers lead to partial writes.
- **Unbounded growth** — 300-line handlers nobody dares refactor.
A handler should: let FastAPI parse/validate, call **one** service method, map the result to a response. ~10 lines.

## Q3. What is the service layer?

The layer implementing **use cases** ("register user", "refund payment"). It orchestrates repositories, enforces business rules, owns transaction boundaries, emits domain events. It knows nothing about HTTP. Each method reads like the product requirement.

## Q4. What is the repository pattern?

An abstraction over data access with **intent-revealing methods** (`find_active_by_email`) instead of query mechanics. Benefits: services don't import SQLAlchemy; query tuning lives in one place; easy to fake in unit tests. Cost: another layer — keep it thin, don't hide the ORM's power behind a leaky generic interface.

## Q5. How would you separate API, business logic, and database logic?

Dependency direction: **API → Service → Repository → ORM**, never backwards.
- Routers import schemas and services; never ORM models.
- Services import repositories and domain exceptions; never `fastapi`.
- Repositories import ORM models; never schemas.
- Domain exceptions live in `core/` and one exception handler maps them to HTTP.
Enforce with `import-linter` contracts in CI so the architecture doesn't erode.

---

## Q6. How do you organize routers?

One `APIRouter` per feature with prefix/tags declared once; a version router aggregates them:
```python
# app/features/orders/router.py
router = APIRouter(prefix="/orders", tags=["orders"])

@router.get("/{order_id}", response_model=OrderRead)
async def get_order(...): ...

# app/api/v1/router.py
router = APIRouter()
router.include_router(users.router)
router.include_router(orders.router, dependencies=[Depends(get_current_user)])
router.include_router(payments.router, dependencies=[Depends(get_current_user)])
```
Keep handlers small; don't nest routers more than two levels; avoid "magic" auto-discovery by scanning directories.

## Q7. How do you organize Pydantic schemas?

Per feature (`features/orders/schemas.py`), named by intent: `OrderCreate`, `OrderUpdate`, `OrderRead`, `OrderSummary`. Shared primitives (`Page[T]`, `ErrorResponse`, `Money`) in `core/schemas.py`. Never share one schema for request and response; never reuse ORM models as schemas; validators handle *format* rules only.

## Q8. How do you organize configuration?

One `Settings` class, loaded once, injected via dependency:
```python
from typing import Literal
from pydantic import PostgresDsn, RedisDsn, HttpUrl, SecretStr
from pydantic_settings import BaseSettings, SettingsConfigDict

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
    def is_prod(self) -> bool:
        return self.env == "prod"
```
- `PostgresDsn`/`RedisDsn`/`HttpUrl` — validated URL types; a malformed `DATABASE_URL` fails at startup with a clear message.
- `env_nested_delimiter="__"` — `DB__POOL_SIZE=20` maps to nested settings.
- `Literal[...]` for `env` — only these four values allowed.
- Avoid scattered `os.getenv` calls.

## Q9. How do you manage environment variables?

- **Local**: `.env` file (git-ignored) loaded by pydantic-settings; commit `.env.example` with dummy values.
- **CI**: masked pipeline variables.
- **Kubernetes**: `ConfigMap` for non-secrets, `Secret` for secrets, via `envFrom`.
- Same image in every environment; only env differs (12-factor).
- Validate at startup; never default a secret; never log the settings object.

## Q10. How do you manage secrets?

- Never in git, images, or logs.
- **Source of truth**: a secrets manager (AWS Secrets Manager, Vault).
- **Delivery**: External Secrets Operator or Secrets Store CSI driver syncs into Kubernetes; the pod's IAM role (IRSA) authorises the fetch — no static keys.
- **Rotation**: the manager rotates; the app re-reads on restart/SIGHUP; prefer IAM DB auth / RDS Proxy so DB passwords don't exist at all.
- **Scope**: each service its own secrets and role.
- **Audit**: access logs; **scan** repos with gitleaks in CI.

---

## Q11. How would you implement API versioning?

Separate router packages per version (file 03 Q15). Practices:
- Version only on **breaking** changes; additive changes don't need a new version.
- Domain/service layer stays unversioned; versions are thin adapters.
- Announce deprecation with `Deprecation`/`Sunset` headers and docs; measure per-version traffic; remove at zero.

## Q12. URI versioning vs header versioning?

| | URI (`/api/v2/users`) | Header (`Accept: application/vnd.app.v2+json`) |
|---|---|---|
| Discoverability | Obvious, curl-friendly, URL-cacheable | Hidden; caches need `Vary` |
| Routing | Trivial prefix | Custom dependency/middleware |
| Purity | "Same resource, different URL" | Same URL, negotiated representation |
| Gateway support | Universal | Needs header routing |

Default: **URI** with major versions only.

## Q13. How do you maintain backward compatibility?

- Additive-only changes within a version; never change a field's type/meaning or make optional → required.
- Tolerant reader for inbound third-party payloads (`extra="ignore"`).
- Contract tests against OpenAPI snapshots; consumer-driven contracts (Pact) for internal clients.
- Feature flags for gradual behaviour changes; deprecation policy with timelines.

## Q14. How do you design reusable components?

- **Dependencies** for cross-cutting request logic.
- **Generic schemas** (`Page[T]`, `ErrorResponse`).
- **ORM mixins** (`TimestampMixin`, `SoftDeleteMixin`).
- **Thin base repository** with `get/add/delete`.
- **Integration clients** behind a `Protocol` so they can be faked.
- Truly shared code (auth verification, logging, tracing setup) → an **internal library** with semantic versioning.
- Composition over deep inheritance.

## Q15. How would you prevent circular imports?

- Keep the dependency direction (API → service → repo → models); a cycle means a lower layer imports a higher one.
- Shared types in a leaf module (`core/`), never in `main.py`.
- Type-only imports under `TYPE_CHECKING`:
```python
from typing import TYPE_CHECKING
if TYPE_CHECKING:
    from app.features.orders.models import Order
```
- SQLAlchemy relationships accept string class names (`relationship("Order")`) for this reason.
- Never import `app` (the FastAPI instance) from feature modules; use `APIRouter`.
- `import-linter` forbids cycles in CI.

## Q16. How do you handle shared utilities?

`app/core/` (or `app/common/`) for small, dependency-free helpers: datetime, IDs, pagination math, text. No business logic, no imports from features, well tested. If a "utils" module grows past a few hundred lines, split by topic. Cross-service utilities → internal package on a private index.

## Q17. How do you design a modular FastAPI application?

Each feature module is self-contained (models, schemas, repo, service, router) and exposes a small public API (`router`, service factory, domain events). Modules talk through **service interfaces or events**, never by reaching into each other's repositories. This "modular monolith" is one deployable with clear seams — the prerequisite for extracting a microservice later.

---

## Q18. Monolith vs microservices — when would you choose each?

**Start with a modular monolith** unless you already have concrete reasons not to.

Choose **monolith** when: small team (< ~15 engineers), domain boundaries still shifting, transactional consistency across modules needed, ops maturity low. One deploy, one DB transaction, simple debugging.

Choose **microservices** when: independent teams need independent release cadence; components have very different scaling profiles (ML inference vs CRUD); different runtimes; hard fault-isolation or regulatory (PCI) requirements. Costs: network failures, distributed transactions (sagas), observability, service discovery, versioned contracts, N× infrastructure.

### 🎤 Interview answer
> "Microservices solve an *organisational* scaling problem more than a technical one. I default to a well-structured monolith and extract services only when a boundary is stable and there's a clear scaling or ownership driver."

## Q19. How would you split a large FastAPI monolith?

1. **Map the domain** (event storming / bounded contexts). Pick a candidate with low coupling and a clear driver.
2. **Modularise in place** — make it talk to others only via interfaces/events; fix leaks.
3. **Separate the data** — its own tables/schema; remove cross-boundary foreign keys; replace joins with API calls or event-fed read models.
4. **Strangler fig** — stand up the new service; route its endpoints to it at the gateway; keep the old code behind a flag.
5. **Shadow traffic** to compare outputs.
6. **Cut over**; delete old code. Repeat. Measure success by deploy frequency and incident isolation, not service count.

## Q20. How would you design FastAPI services for independent deployment?

- Own data store per service; no shared tables.
- Versioned, backward-compatible APIs; consumer-driven contract tests.
- Async events for non-critical paths so one deploy can't break another.
- Per-service pipelines, images, Helm charts, secrets.
- Feature flags to decouple deploy from release.
- Health/readiness endpoints and graceful shutdown so rolling deploys are invisible.
- Shared observability contract (same metrics/trace headers).
- Shared internal library with semantic versioning so one team's change doesn't force a global redeploy.

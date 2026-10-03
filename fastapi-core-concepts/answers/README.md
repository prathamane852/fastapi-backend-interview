# FastAPI Interview Guide — Topic by Topic

This folder splits the interview questions into **16 files, one per topic**. Every file follows the same layout so you always know where to look:

| Block | What it gives you |
|---|---|
| 📖 **Vocabulary** | Every technical term used in the file, defined *before* it's used. Each row has a precise **Definition** and a one-sentence **Say it in an interview** version you can quote. |
| 🧒 **Simple explanation** | The idea in plain language, with an everyday analogy. |
| 🧠 **First principles** | *Why* the thing exists. What problem would we have without it? |
| 🎤 **Interview answer** | The 30–60 second answer you actually say out loud. |
| 💻 **Code** + 🔍 **Line by line** | A runnable example, then each line explained — including the imports and the syntax. |
| ⚠️ **Mistakes & follow-ups** | What candidates get wrong, and the question the interviewer will ask next. |

## Files

| # | File | Questions covered |
|---|---|---|
| 0 | [00-glossary.md](00-glossary.md) | Master glossary: every term from all files, alphabetized, with definition, interview one-liner and a link back to the topic |
| 1 | [01-fastapi-fundamentals.md](01-fastapi-fundamentals.md) | What FastAPI is, ASGI/WSGI, Uvicorn, Starlette, first app, HTTP methods, params, Pydantic intro, OpenAPI, request lifecycle |
| 2 | [02-request-handling-and-validation.md](02-request-handling-and-validation.md) | Path/query/body params, nested models, defaults, string/number/email validation, files, status codes |
| 3 | [03-pydantic-deep-dive.md](03-pydantic-deep-dive.md) | Validation internals, validators, optional fields, extra fields, serialization, request vs response schemas, versioning schemas |
| 4 | [04-dependency-injection.md](04-dependency-injection.md) | `Depends()`, reusable/nested/class deps, auth & DB deps, lifecycle, overrides, RBAC, the "100 APIs" design question |
| 5 | [05-async-programming.md](05-async-programming.md) | asyncio, event loop, `async`/`await`, I/O vs CPU bound, blocking code, `gather`, async drivers, troubleshooting latency |
| 6 | [06-middleware.md](06-middleware.md) | Creating middleware, order, logging, latency, correlation IDs, CORS, security headers, global exceptions, tracing |
| 7 | [07-authentication-and-authorization.md](07-authentication-and-authorization.md) | JWT, OAuth2, OIDC, tokens, RBAC/permissions, revocation, Cognito, admin protection, privilege escalation |
| 8 | [08-databases.md](08-databases.md) | SQLAlchemy, sessions, pooling, transactions, Alembic, zero-downtime schema changes, N+1, pagination, replicas, repository layer |
| 9 | [09-project-structure-and-architecture.md](09-project-structure-and-architecture.md) | Folder layout, service/repository layers, config, secrets, versioning, circular imports, monolith vs microservices |
| 10 | [10-performance-and-scaling.md](10-performance-and-scaling.md) | Benchmarking, metrics, workers, HPA, Redis, caching, stampede, rate limiting, streaming, memory |
| 11 | [11-background-tasks-and-jobs.md](11-background-tasks-and-jobs.md) | BackgroundTasks vs Celery vs SQS, long jobs, job status API, retries, backoff, idempotency, DLQ |
| 12 | [12-testing.md](12-testing.md) | pytest, TestClient, testing endpoints/auth/validation, mocking, overrides, async tests, race conditions, coverage, GitLab CI |
| 13 | [13-observability-and-monitoring.md](13-observability-and-monitoring.md) | Prometheus, OpenTelemetry, traces/spans, correlation IDs, Grafana, alerts, troubleshooting memory/CPU/latency/5xx |
| 14 | [14-docker-and-kubernetes.md](14-docker-and-kubernetes.md) | Dockerfile, multi-stage, non-root, secrets, Deployment, probes, zero-downtime, resources, HPA, CrashLoopBackOff, OOMKilled |
| 15 | [15-ci-cd.md](15-ci-cd.md) | Pipeline stages, env config, migrations in deploy, rollbacks, blue-green, canary, ArgoCD, Secrets Manager, scanning |
| 16 | [16-system-design.md](16-system-design.md) | 20 design questions with a reusable answering framework |

> The single-file version (`../answers.md`) is the condensed edition. These files are the **expanded** edition with vocabulary and deeper line-by-line explanations.

---

## 🔴 Senior track (6+ years) → [`../../advanced-backend/`](../../advanced-backend/)

These 16 files cover the **fundamentals**. Questions that belong at senior level have been **moved** to the advanced section, and the heading here now carries a pointer to where the answer lives.

| Topic | Moved from | Now in |
|---|---|---|
| Zero-downtime DDL, `EXPLAIN`, replication | [08-databases.md](08-databases.md) Q15/17/23 | [03-postgres-deep-dive.md](../../advanced-backend/03-postgres-deep-dive.md) |
| N+1, cursor pagination, replicas, 1M rows | [08-databases.md](08-databases.md) Q16/19/22/24 | [02-databases-advanced.md](../../advanced-backend/02-databases-advanced.md) |
| Redis, stampede, rate limiting | [10-performance-and-scaling.md](10-performance-and-scaling.md) Q15/18/20 | [04-redis-and-caching.md](../../advanced-backend/04-redis-and-caching.md) |
| Idempotency, retries, DLQ | [11-background-tasks-and-jobs.md](11-background-tasks-and-jobs.md) Q10–15 | [05-message-queues-sqs-sns-kafka.md](../../advanced-backend/05-message-queues-sqs-sns-kafka.md) |
| Context propagation, instrumentation, alerting | [13-observability-and-monitoring.md](13-observability-and-monitoring.md) Q11/12/18/19 | [08-opentelemetry-advanced.md](../../advanced-backend/08-opentelemetry-advanced.md) |
| Zero-downtime, CrashLoop, OOMKilled, rollback | [14-docker-and-kubernetes.md](14-docker-and-kubernetes.md) Q14/17/18/20 | [07-deployments-and-release.md](../../advanced-backend/07-deployments-and-release.md) |
| Migrations, blue-green, canary, ArgoCD | [15-ci-cd.md](15-ci-cd.md) Q4/6/7/8/9 | [07-deployments-and-release.md](../../advanced-backend/07-deployments-and-release.md) |
| Memory leaks, high CPU | [13-observability-and-monitoring.md](13-observability-and-monitoring.md) Q16/17 | [01-python-advanced.md](../../advanced-backend/01-python-advanced.md) |

The advanced section also adds material with no beginner equivalent: **AWS**, **LLD**, and **how to explain a design pattern out loud**.

## How to study with these files

1. Read the 📖 Vocabulary block of a file. If any word is unfamiliar, that's the gap to close first. For revision across all topics, use [00-glossary.md](00-glossary.md).
2. For each question, cover the 🎤 answer and try to say it yourself. Then uncover and compare.
3. Type out the 💻 code examples. Don't copy-paste — typing forces you to notice every symbol.
4. Read the ⚠️ block last; interviewers love the follow-up questions listed there.

All code targets **Python 3.11+, FastAPI 0.110+, Pydantic v2, SQLAlchemy 2.0**.

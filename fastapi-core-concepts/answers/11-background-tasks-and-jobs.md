# 11 — Background Tasks & Job Processing

---

## 📖 Vocabulary

> *Definition* is the precise meaning; *Say it in an interview* is the quotable one-liner.

| Term | Definition | Say it in an interview |
|---|---|---|
| **Background task** | Work scheduled to execute after the HTTP response is sent, within the same process (Starlette `BackgroundTasks`). No persistence or retry. | "BackgroundTasks run after the response in the same process, with no durability." |
| **Fire-and-forget** | Initiating work without waiting for or tracking its outcome. | "Fire-and-forget is fine only when losing the work is acceptable." |
| **Task/job queue** | A system where producers enqueue units of work and independent workers dequeue and execute them, decoupling producers from processing. | "A queue decouples request handling from heavy or unreliable work." |
| **Message broker** | Middleware that stores and routes messages between producers and consumers (Redis, RabbitMQ, SQS, Kafka). | "The broker is the durable buffer between producer and worker." |
| **Worker** | A process that consumes messages from a queue and executes the corresponding job. | "Workers scale independently of the API." |
| **Celery** | A distributed task queue for Python with brokers (Redis/RabbitMQ), retries, scheduling (Beat) and monitoring (Flower). | "Celery is the mature Python job framework; arq or Dramatiq are lighter async options." |
| **SQS** | *Amazon Simple Queue Service*: managed queue with at-least-once delivery (standard) or exactly-once-per-dedup-window with ordering (FIFO), visibility timeouts and DLQs. | "SQS is a managed, durable queue with visibility timeouts and DLQs built in." |
| **Durability** | The property that accepted work survives process or node failure because it is persisted. | "Durable means a crash doesn't lose the job." |
| **At-least-once delivery** | Guarantee that every message is delivered one or more times; duplicates are possible, loss is not. Requires idempotent consumers. | "At-least-once means I must design for duplicates." |
| **At-most-once / exactly-once** | At-most-once: may lose, never duplicates. Exactly-once: effectively achieved by at-least-once + idempotent processing. | "Exactly-once in practice is at-least-once plus idempotency." |
| **Visibility timeout** | SQS period during which a received message is hidden from other consumers; if not deleted in time, it becomes visible again for retry. | "The visibility timeout is my processing deadline before automatic redelivery." |
| **Acknowledgement (ack/delete)** | Informing the broker that a message was processed so it can be removed; done after processing for at-least-once. | "Ack after success, never before." |
| **Dead-letter queue (DLQ)** | A queue receiving messages that exceeded the maximum delivery attempts, isolating poison messages for inspection and replay. | "The DLQ quarantines failures so the main queue keeps flowing." |
| **Redrive policy** | SQS configuration specifying the DLQ and `maxReceiveCount` after which a message is moved there. | "Redrive policy sets how many failures before DLQ." |
| **Retry** | Re-executing a failed operation, ideally only for transient errors and with bounded attempts. | "Retry transient errors, not permanent ones." |
| **Exponential backoff** | Increasing the delay between retries geometrically (base × 2ⁿ) up to a cap. | "Backoff gives the failing dependency room to recover." |
| **Jitter** | Random variation added to backoff delays to spread retries and avoid synchronized bursts. | "Jitter prevents thundering-herd retries." |
| **Idempotency** | Property that repeating an operation yields the same state as performing it once. | "Idempotency makes retries and redeliveries safe." |
| **Idempotency key** | A client-supplied unique identifier for a logical operation, stored by the server to detect and replay duplicate requests. | "The idempotency key lets me return the original response on a retried POST." |
| **Transactional outbox** | Pattern: write the outgoing event to an `outbox` table in the same DB transaction as the state change; a relay publishes it afterward, guaranteeing no lost events. | "The outbox makes 'save and publish' atomic." |
| **CDC (Change Data Capture)** | Streaming database changes (via WAL/binlog) to other systems, e.g., Debezium reading the outbox table. | "CDC turns the outbox into a stream without polling." |
| **202 Accepted** | HTTP status indicating the request was accepted for processing but not completed; used with a job resource to poll. | "202 plus a Location header is the async job contract." |
| **Polling vs webhook vs SSE** | Polling: client repeatedly queries status. Webhook: server calls a client URL on completion. SSE: server pushes events over a held HTTP connection. | "Polling is simple; webhooks and SSE avoid polling storms." |
| **Heartbeat** | Periodic signal from a worker indicating it is still processing, used to extend visibility timeouts or detect stuck jobs. | "Heartbeats let long jobs extend their lease and expose stuck workers." |
| **Poison message** | A message that fails every processing attempt (malformed or triggers a bug), blocking a queue unless quarantined. | "Poison messages go to the DLQ after N attempts." |
| **Distributed lock** | A lock held in a shared store (Redis `SET NX PX`) with a TTL, ensuring only one worker processes a given job at a time. | "A TTL'd Redis lock prevents two workers taking the same job." |
| **Deduplication ID** | SQS FIFO attribute; messages with the same ID within a 5-minute window are treated as duplicates. | "FIFO dedup IDs suppress duplicate sends for five minutes." |
| **Cooperative cancellation** | Cancelling long work by setting a flag the job checks between steps, rather than killing the process. | "Cancellation is cooperative: the worker checks a flag at checkpoints." |
| **Checkpointing** | Persisting intermediate progress so a job can resume from the last checkpoint after interruption. | "Checkpoints turn restarts into resumes." |
| **KEDA** | Kubernetes autoscaler driving worker replica count from queue depth or lag. | "KEDA scales workers on queue depth, even to zero." |

---

## Q1. What are FastAPI BackgroundTasks?

### 🧒 Simple explanation
A way to run a function **after the response has been sent**, in the same process. Uvicorn sends the response, then runs the task. The client doesn't wait.

### 💻 Code
```python
from fastapi import BackgroundTasks

async def send_welcome_email(email: str):
    await mailer.send(to=email, template="welcome")

@router.post("/signup", status_code=201)
async def signup(body: SignupIn, tasks: BackgroundTasks, svc: UserSvc):
    user = await svc.register(body)
    tasks.add_task(send_welcome_email, user.email)
    return UserRead.model_validate(user)
```
### 🔍 Line by line
- `async def send_welcome_email(email)` — any callable, sync or async (sync ones run in the thread pool).
- `tasks: BackgroundTasks` — FastAPI injects a task collector.
- `tasks.add_task(func, *args)` — queue it with arguments.
- `return ...` — response goes out **first**; then the email is sent.

## Q2. When should you use BackgroundTasks?

Small, fast, **fire-and-forget** work where losing it on a crash is acceptable: send a notification, write an audit entry, invalidate a cache, bump a counter, call a webhook that has its own retries. Rule: seconds, not minutes; no strict delivery guarantee.

## Q3. When should you NOT use BackgroundTasks?

- Work that **must** happen (payments, fulfilment) — a pod restart loses it; no persistence, no retry.
- Long or CPU-heavy work — it occupies the web worker's loop/thread pool and delays other requests; Kubernetes may kill the pod mid-task on scale-down.
- Work needing status tracking, scheduling, rate control, or fan-out.
- High volume — no backpressure; memory grows.

## Q4. BackgroundTasks vs Celery?

| | BackgroundTasks | Celery (Redis/RabbitMQ broker) |
|---|---|---|
| Runs where | Same web process, after response | Separate worker processes/machines |
| Durability | None (lost on crash) | Persisted in broker; acked on success |
| Retries/backoff | Manual | Built in |
| Scheduling | ❌ | Beat, ETA/countdown |
| Scaling | Tied to web pods | Independent worker autoscaling |
| Monitoring | ❌ | Flower, events |
| Complexity | Zero | Broker + workers + result backend |
| Use for | Trivial side effects | Anything important, long, or bursty |

Async-native alternatives: **arq**, **Dramatiq**, **Taskiq**, **SAQ**.

## Q5. BackgroundTasks vs AWS SQS?

SQS is a **managed durable queue** — the transport, not the worker. It gives persistence, at-least-once delivery, visibility timeouts, DLQs, and decoupled consumers (Lambda, ECS, Kubernetes workers scaled by KEDA). Choose SQS/Kafka/RabbitMQ when the work is important, cross-service, or must survive deploys; BackgroundTasks for trivial in-process side effects.

---

## Q6. How would you process a long-running job?

**The async request/response pattern:**
1. `POST /jobs` validates input, writes a job row (`queued`), publishes a message, returns **202 Accepted** with `job_id` and a `Location` header.
2. A worker consumes the message, sets `running`, does the work (with progress/heartbeats), stores the result, sets `succeeded`/`failed`.
3. Client polls `GET /jobs/{id}` — or receives a webhook / SSE update.

```python
import json
from fastapi.responses import JSONResponse

@router.post("/exports", status_code=202)
async def create_export(body: ExportIn, user: CurrentUser, db: DB, sqs: SQSDep):
    job = Job(user_id=user.id, type="export", status="queued", payload=body.model_dump())
    db.add(job)
    await db.commit()
    await sqs.send_message(QueueUrl=settings.export_queue,
                           MessageBody=json.dumps({"job_id": str(job.id)}))
    return JSONResponse({"job_id": str(job.id), "status": "queued"}, status_code=202,
                        headers={"Location": f"/api/v1/jobs/{job.id}"})
```
### 🔍 Line by line
- Persist the job **before** publishing, so it exists even if publish fails (a sweeper can republish `queued` jobs older than N minutes — the transactional outbox is the robust version).
- `MessageBody=json.dumps({"job_id": ...})` — publish a *small* message (IDs, not payloads).
- `202` + `Location` — where to poll.

## Q7. How would you implement asynchronous job processing?

Worker side (SQS, separate deployment from the API):
```python
async def run():
    async with session.client("sqs") as sqs:
        while True:
            resp = await sqs.receive_message(QueueUrl=Q, MaxNumberOfMessages=10,
                                             WaitTimeSeconds=20, VisibilityTimeout=300)
            for msg in resp.get("Messages", []):
                job_id = json.loads(msg["Body"])["job_id"]
                try:
                    await process_job(job_id)
                    await sqs.delete_message(QueueUrl=Q, ReceiptHandle=msg["ReceiptHandle"])
                except RetryableError:
                    pass
                except Exception:
                    log.exception("job failed", extra={"job_id": job_id})
                    await mark_failed(job_id)
                    await sqs.delete_message(QueueUrl=Q, ReceiptHandle=msg["ReceiptHandle"])
```
### 🔍 Line by line
- `WaitTimeSeconds=20` — **long polling**: wait up to 20 s for messages instead of returning empty immediately (fewer API calls).
- `VisibilityTimeout=300` — hide the message from other consumers for 5 minutes while we work.
- `process_job` — must be **idempotent** (Q11).
- `delete_message` **only after success** → at-least-once semantics.
- `except RetryableError: pass` — don't delete; the message reappears after the visibility timeout → automatic retry; after `maxReceiveCount` SQS moves it to the DLQ.
- Permanent failure → mark and delete.
- Scale workers with KEDA on `ApproximateNumberOfMessagesVisible`. Celery equivalent: `@app.task(bind=True, autoretry_for=(RetryableError,), retry_backoff=True, max_retries=5)`.

## Q8. How would you design a job status API?

```python
from enum import Enum
from uuid import UUID

class JobStatus(str, Enum):
    queued = "queued"; running = "running"; succeeded = "succeeded"; failed = "failed"; cancelled = "cancelled"

class JobRead(BaseModel):
    id: UUID
    status: JobStatus
    progress: int = Field(ge=0, le=100)
    result_url: str | None = None      # pre-signed S3 URL when succeeded
    error: str | None = None           # user-safe message when failed
    created_at: datetime
    updated_at: datetime

@router.get("/jobs/{job_id}", response_model=JobRead)
async def get_job(job_id: UUID, user: CurrentUser, repo: JobRepo):
    job = await repo.get_for_user(job_id, user.id)   # ownership → 404 if not theirs
    if job is None:
        raise NotFound("job")
    return job
```
Design notes: 202 on create with `Location`; `GET` returns 200 while in progress (with a `Retry-After` hint); `DELETE /jobs/{id}` for cooperative cancel; list endpoint with filters; TTL cleanup; optional webhook/SSE to avoid polling storms; expose attempt count and timestamps.

---

## Q9. How do you retry failed jobs?

- Classify errors: **retryable** (timeouts, 5xx, deadlocks) vs **permanent** (validation, 4xx, business rule). Retry only the former.
- Bounded attempts (e.g., 5) with **exponential backoff + jitter**.
- Idempotent handlers so retries are safe.
- Persist attempt count and last error.
- After max attempts → DLQ + alert + replay tooling.

## Q10. What is exponential backoff?

> 🔴 **Moved to the advanced section** → [../../advanced-backend/05-message-queues-sqs-sns-kafka.md](../../advanced-backend/05-message-queues-sqs-sns-kafka.md)
> 
> Q9 — full jitter, and why the naive formula causes a thundering herd.

---

## Q11. How do you prevent duplicate job execution?

> 🔴 **Moved to the advanced section** → [../../advanced-backend/05-message-queues-sqs-sns-kafka.md](../../advanced-backend/05-message-queues-sqs-sns-kafka.md)
> 
> Q2 — idempotent consumers and the same-transaction rule.

---

## Q12. What is idempotency?

> 🔴 **Moved to the advanced section** → [../../advanced-backend/05-message-queues-sqs-sns-kafka.md](../../advanced-backend/05-message-queues-sqs-sns-kafka.md)
> 
> Q1–Q2 — delivery semantics and exactly-once *effect*.

---

## Q13. How would you implement an idempotent POST API?

> 🔴 **Moved to the advanced section** → [../../advanced-backend/09-system-design-advanced.md](../../advanced-backend/09-system-design-advanced.md)
> 
> D4 — the full idempotency-key implementation with body fingerprinting.

---

## Q14. How would you handle failed messages?

> 🔴 **Moved to the advanced section** → [../../advanced-backend/05-message-queues-sqs-sns-kafka.md](../../advanced-backend/05-message-queues-sqs-sns-kafka.md)
> 
> Q5–Q6 — visibility timeout, poison vs transient failure.

---

## Q15. What is a dead-letter queue?

> 🔴 **Moved to the advanced section** → [../../advanced-backend/05-message-queues-sqs-sns-kafka.md](../../advanced-backend/05-message-queues-sqs-sns-kafka.md)
> 
> Q6 — DLQ configuration, alerting and the redrive path.

---

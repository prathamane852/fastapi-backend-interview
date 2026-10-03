# 05 — Message Queues: SQS, SNS, Kafka

> **How to use this file.** Every question has a **🗣️ Say this** block — 4 to 5 lines, ready to speak. `Output:` marked **(real)** was actually run.

**One sentence organises this entire file:** every real message system is at-least-once, so **every consumer must be idempotent**. Everything else hangs off that.

---

## 📖 Jargon, in plain words

| Jargon | What it actually means | How you'd say it |
|---|---|---|
| **At-most-once** | Might lose a message, never duplicates it. | "Fire and forget — fine for metrics, not for orders." |
| **At-least-once** | Never loses a message, might deliver it twice. | "What every real system gives you — hence idempotency." |
| **Exactly-once** | Each message takes effect precisely once. | "Impossible for *delivery*. Achievable in *effect*." |
| **Idempotent** | Running it twice changes nothing versus running it once. | "That's what makes redelivery harmless." |
| **Visibility timeout** | SQS hides a message you've received, so nobody else gets it — for a while. | "If processing outlasts it, someone else gets the same message." |
| **Acknowledge / ack** | Telling the broker you're done so it can delete the message. | "You delete after success, not before." |
| **DLQ** | Dead-letter queue — where a message goes after failing N times. | "It turns a poison message from an infinite loop into an alert." |
| **Poison message** | A message that will never succeed no matter how many times you retry. | "Malformed payload — retrying it five times is pure waste." |
| **Fan-out** | One event delivered to several independent consumers. | "SNS to SQS gives each consumer its own queue and retries." |
| **Partition** | Kafka's unit of ordering and of parallelism. | "Order is guaranteed within a partition and nowhere else." |
| **Consumer group** | A set of consumers splitting a topic's partitions between them. | "Your parallelism is capped by partition count." |
| **Consumer lag** | How far behind the latest message your consumer is. | "Lag growing means you can't keep up." |
| **Outbox** | Writing the event into a database table in the same transaction as the data. | "It closes the gap between committing and publishing." |
| **Backpressure** | Making the producer slow down when consumers can't keep up. | "Without it, overload becomes unbounded queue growth." |

---

# Section A — Semantics

## Q1. Is exactly-once delivery real?

### 🗣️ Say this
> "Not for delivery, and the reason is simple enough to say in one breath. The consumer processes the message and then acknowledges it — and that acknowledgement can be lost. The broker has no way to tell 'the consumer died before doing the work' apart from 'the consumer did the work and the ack vanished'. So it has to redeliver, and sometimes it redelivers something already done.
> What *is* achievable is exactly-once **effect**, and that's the framing I'd use. At-least-once delivery plus an idempotent consumer gives you the outcome people actually mean.
> Kafka's exactly-once semantics are real but narrowly scoped — they cover read-process-write *within Kafka*. The moment your side effect is an external database or a third-party API, you're back to needing idempotency.
> So I design as if every message will arrive at least twice, because it will."

---

## Q2. How do you make a consumer idempotent?

### 🗣️ Say this
> "Three options, and I'd pick the strongest one available.
> Best is natural idempotency — reshape the operation so repeating it doesn't matter. `SET status = 'shipped'` is naturally idempotent; `balance = balance - 10` isn't.
> Next is a conditional update — `WHERE status = 'paid'` — so the second run matches zero rows.
> Otherwise a dedupe table with a unique constraint on the message ID, and let the database enforce it.
> But the detail that actually matters, and the one people get wrong: the marker insert and the work must be in **one transaction**. If you record the marker, commit, then do the work and crash, the redelivery sees the marker and skips — and the work never happens. That's not idempotency, that's a new and worse bug."

### 💻 Proof — *(real)*
```python
def handle(msg_id, *, same_txn):
    if msg_id in processed:
        return "skipped (duplicate)"
    if same_txn:
        processed.add(msg_id); work_done.append(msg_id)   # atomic together
    else:
        processed.add(msg_id)                              # marker committed...
        raise RuntimeError("crash before work!")           # ...then crash
```
```
Output:
WRONG (marker committed separately, then crash):
  crash before work!
  redelivery -> skipped (duplicate)
  work actually done: []          <-- THE MESSAGE IS SILENTLY LOST

RIGHT (marker + work in one transaction):
  first  -> processed
  retry  -> skipped (duplicate)
  work actually done: ['m2']      <-- exactly once
```
> Look at the first case. The redelivery was *correctly* identified as a duplicate — and the work still never happened. That's the bug, and it's invisible until someone asks why an order was never shipped.

### 💻 The real implementation
```python
async def handle(msg: Message, session: AsyncSession) -> None:
    key = msg.attributes["idempotency_key"]
    try:
        async with session.begin():                        # ONE transaction
            session.add(ProcessedMessage(key=key, handler="ship_order"))
            await session.flush()                           # unique violation here
            await do_the_work(msg, session)                 # SAME transaction
    except IntegrityError:
        logger.info("duplicate_skipped", extra={"key": key})
```

### ⚠️ Follow-up: "Where does the idempotency key come from?"
> "The producer, and it has to be stable across retries — derived from the business event, like `order:{id}:shipped`, not a fresh UUID per publish attempt. If a retried publish generates a new key, you've defeated the whole mechanism.
> SQS FIFO has `MessageDeduplicationId`, but that's only a five-minute window, which isn't enough for a DLQ replay hours later. So I'd keep an application-level key regardless."

⚠️ **The dedupe table grows forever** — partition it by day and drop old partitions, or use Redis keys with a TTL longer than your maximum redelivery window.

---

## Q3. The dual-write problem and the outbox pattern.

### 🗣️ Say this
> "The bug is this: you commit an order to Postgres, then publish `order.created` to SQS. Between those two operations the process can die, or SQS can be unavailable. Now the order exists and no downstream system will ever hear about it.
> You cannot fix that with error handling, because the failure is *in the gap*. And swapping the order doesn't help either — then you'd publish an event for an order that fails to commit.
> The outbox pattern makes it atomic. You insert the event into an `outbox` table in the same local transaction as the order. One commit, both or neither.
> A separate relay process then reads unpublished rows and publishes them. The relay is at-least-once — it can publish and die before marking the row — which is fine, because we already established consumers must be idempotent."

### 💻 The write side
```sql
CREATE TABLE outbox (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  aggregate_id uuid NOT NULL, event_type text NOT NULL, payload jsonb NOT NULL,
  created_at timestamptz NOT NULL DEFAULT now(), published_at timestamptz
);
-- Partial index: only covers unpublished rows, so it stays tiny forever
CREATE INDEX ON outbox (created_at) WHERE published_at IS NULL;
```
```python
async with session.begin():                    # ONE transaction, both writes
    session.add(Order(id=oid, user_id=uid, total=total, status="created"))
    session.add(Outbox(aggregate_id=oid, event_type="order.created",
                       payload={"order_id": str(oid)}))
```

### 💻 The relay — `SKIP LOCKED` lets you run several safely
```sql
SELECT id, event_type, payload FROM outbox
WHERE published_at IS NULL
ORDER BY created_at
FOR UPDATE SKIP LOCKED           -- many relays, each takes different rows
LIMIT 100;
```

### ⚠️ Follow-ups
| They ask | Say this |
|---|---|
| "Polling is inefficient." | "It is. The alternative is change-data-capture — Debezium reading Postgres logical replication, so there's no polling at all. Lower latency, but another piece of infrastructure. For most services polling every 200ms against a partial index is entirely fine, because that index only covers unpublished rows." |
| "What about ordering?" | "The relay publishes in order, but SQS standard doesn't preserve it. If order matters you need FIFO with a `MessageGroupId` per aggregate, or a version number consumers can use to detect reordering." |
| "What's the inbox pattern?" | "Same idea on the receiving side — record the message ID in the same transaction as the work. That's the dedupe table from Q2; 'inbox' is just the name when it's paired with an outbox." |

---

# Section B — SQS & SNS

## Q4. SQS Standard vs FIFO.

### 🗣️ Say this
> "Standard unless I can articulate why order matters, because FIFO has a real throughput ceiling and you inherit head-of-line blocking — one stuck message blocks its whole group.
> The key insight about FIFO is that `MessageGroupId` is doing double duty: it's both the ordering unit *and* the parallelism unit. Order is guaranteed within a group, and different groups process in parallel.
> So you set the group to whatever entity's order actually matters — the order ID or user ID — and you get per-entity ordering with global concurrency.
> The mistake I've seen is setting it to a constant, which gives you one ordered stream and zero parallelism, and people don't realise why throughput collapsed."

| | Standard | FIFO |
|---|---|---|
| Throughput | Nearly unlimited | 300/s (3,000 batched); higher in high-throughput mode |
| Ordering | Best-effort, **not guaranteed** | Strict within a `MessageGroupId` |
| Dedup | None | 5-minute window via `MessageDeduplicationId` |

📐 **Numbers worth memorising:** max message 256 KB · retention up to 14 days (default 4) · visibility timeout default 30s, max 12 hours · long polling up to 20s · delay up to 15 minutes · batch of 10.

---

## Q5. Visibility timeout — the number one SQS production bug.

### 🗣️ Say this
> "When a consumer receives a message, SQS doesn't delete it — it hides it for the visibility timeout. You delete it yourself after success. If you crash, it reappears and someone else takes it. That's the durability guarantee.
> The bug is when processing takes *longer* than the timeout. The message reappears while you're still working on it, a second consumer picks it up, and now you're processing the same job twice concurrently.
> What makes it nasty is the symptom: duplicated side effects that look random and get *worse under load* — because load is exactly when processing slows down.
> The fix is either setting the timeout above your p99.9 processing time, or better, extending it with a heartbeat while you work."

### 💻 The heartbeat
```python
async def process_with_heartbeat(msg, handler, queue_url):
    stop = asyncio.Event()
    async def heartbeat():
        while not stop.is_set():
            await asyncio.sleep(20)
            if not stop.is_set():
                await sqs.change_message_visibility(          # "still working"
                    QueueUrl=queue_url, ReceiptHandle=msg["ReceiptHandle"],
                    VisibilityTimeout=60)
    hb = asyncio.create_task(heartbeat())
    try:
        await handler(msg)
        await sqs.delete_message(QueueUrl=queue_url, ReceiptHandle=msg["ReceiptHandle"])
    finally:
        stop.set(); await hb
```

### ⚠️ The structural fix
> "If a job takes ten minutes, it probably shouldn't be one message. I'd have a coordinator that fans out small units of work, so no single message has a long processing time. That also gives you progress reporting and resumability for free."

---

## Q6. DLQs — and what do you actually do with one?

### 🗣️ Say this
> "Configuring the DLQ is the easy half — a redrive policy with a max receive count, usually five.
> The operational half is what matters. I alert on DLQ depth greater than zero, because by definition anything in there failed five times and needs a human. And I set retention to the full fourteen days, because a DLQ with four-day retention silently deletes your evidence over a long weekend.
> I also separate failure types *before* the DLQ. A malformed payload will never succeed, so retrying it five times is pure waste — it should go straight there. A downstream 503 will probably work in thirty seconds, so let that one redeliver.
> And I'd want consumers to be idempotent before using the redrive feature, because redriving ten thousand messages into a system that isn't is how one incident becomes two."

### 💻 Config + the handler split
```hcl
resource "aws_sqs_queue" "jobs" {
  visibility_timeout_seconds = 300
  receive_wait_time_seconds  = 20                 # long polling — always on
  redrive_policy = jsonencode({
    deadLetterTargetArn = aws_sqs_queue.jobs_dlq.arn
    maxReceiveCount     = 5
  })
}
resource "aws_sqs_queue" "jobs_dlq" {
  message_retention_seconds = 1209600             # 14 days — ALWAYS max on a DLQ
}
```
```python
try:
    await handler(msg); await delete(msg)
except NonRetryableError:          # bad schema, unknown type, business rejection
    await archive_and_delete(msg)  # don't burn 5 receives on something hopeless
except RetryableError:
    pass                           # let it redeliver after the visibility timeout
```

---

## Q7. SNS vs SQS vs EventBridge.

### 🗣️ Say this
> "SQS is a queue — one logical consumer pulls messages and they're gone. SNS is publish-subscribe — it pushes to many subscribers and buffers nothing.
> The pattern I'd actually draw is **SNS fanning out to SQS**: one topic per event type, and each consuming service subscribes with its own queue.
> The reason that beats SNS straight to Lambda or HTTP is that every consumer gets its own buffering, its own retries, its own DLQ — and can be down for an hour without losing anything. SNS on its own would retry and eventually just drop it. It also means adding a fifth consumer requires zero changes to the producer.
> EventBridge when I want content-based routing, schema discovery, or archive and replay. The routing rules are genuinely more expressive, but it costs more per event and adds a bit of latency."

### ⚠️ The gotcha to volunteer
> "SNS filter policies work on **message attributes**, not the message body, unless you explicitly widen the scope. People set a filter on a body field, it silently matches nothing, and messages just vanish with no error anywhere. I'd also turn on `RawMessageDelivery` when going to SQS, otherwise your consumer has to unwrap an SNS envelope from inside the SQS body."

---

## Q8. Write a production-grade SQS consumer.

### 🗣️ Say this
> "The things that make it production-grade rather than a tutorial: long polling so you're not burning API calls on empty receives, batching ten at a time, bounded concurrency with a semaphore so you don't melt the database, trace context extracted from the message so the trace survives the queue, awareness of the receive count, poison messages separated out, and graceful shutdown on SIGTERM so a rolling deploy doesn't strand in-flight work."

### 💻
```python
class SQSConsumer:
    def __init__(self, queue_url, handler, concurrency=10):
        self.queue_url, self.handler = queue_url, handler
        self.sem = asyncio.Semaphore(concurrency)      # don't melt the DB
        self._shutdown = asyncio.Event()

    async def run(self):
        async with aioboto3.Session().client("sqs") as sqs:
            while not self._shutdown.is_set():
                resp = await sqs.receive_message(
                    QueueUrl=self.queue_url,
                    MaxNumberOfMessages=10,            # batch: 1 API call for 10
                    WaitTimeSeconds=20,                # long poll: no busy-waiting
                    AttributeNames=["ApproximateReceiveCount"],
                    MessageAttributeNames=["All"])
                msgs = resp.get("Messages", [])
                if not msgs: continue
                async with asyncio.TaskGroup() as tg:  # bounded, sibling-cancelling
                    for m in msgs:
                        tg.create_task(self._one(sqs, m))

    async def _one(self, sqs, m):
        async with self.sem:
            ctx = extract_trace_context(m)             # continue the producer's trace
            with tracer.start_as_current_span("sqs.process", context=ctx) as span:
                span.set_attribute("messaging.receive_count",
                                   int(m["Attributes"]["ApproximateReceiveCount"]))
                try:
                    await self.handler(json.loads(m["Body"]))
                    await sqs.delete_message(QueueUrl=self.queue_url,
                                             ReceiptHandle=m["ReceiptHandle"])
                except NonRetryableError:
                    await self._archive(m)
                    await sqs.delete_message(QueueUrl=self.queue_url,
                                             ReceiptHandle=m["ReceiptHandle"])
                except Exception:
                    logger.exception("handler_failed")  # no delete -> redelivery

    async def shutdown(self):
        self._shutdown.set()        # SIGTERM: stop receiving, let in-flight finish
```

---

## Q9. Write exponential backoff properly.

### 🗣️ Say this
> "The naive version is `base * 2 ** attempt`, and that's wrong in a way that matters.
> If a downstream service drops for two seconds, every one of your thousand in-flight requests fails at the same instant and retries at the same instant. You've converted one blip into a synchronised thundering herd that keeps the downstream down.
> So you add **full jitter** — a uniform random between zero and the capped exponential. AWS benchmarked the variants and full jitter came out best on both total work and completion time.
> I'd also add that retries need a *budget*, not just a count. If half your traffic is retries you're amplifying an outage, and that's what circuit breakers and retry budgets exist for."

### 💻 Proof — 1,000 clients retrying at attempt 3, *(real)*
```python
no_jitter = min(cap, base * 2**attempt)                    # everyone at once
full      = random.uniform(0, min(cap, base * 2**attempt)) # spread out
```
```
Output:
no jitter (base * 2**attempt):
  t=0-1s :                                                     0
  t=1-2s :                                                     0
  t=2-3s :                                                     0
  t=3-4s :                                                     0
  t=4-5s : ##################################################  1000    <- all at once

full jitter (uniform(0, base*2**attempt)):
  t=0-1s : ############                                        248
  t=1-2s : ###########                                         239
  t=2-3s : #############                                       261
  t=3-4s : ############                                        252
```
> All 1,000 in one second, versus ~250 per second spread evenly. That histogram *is* the thundering herd.

```python
async def with_backoff(fn, *, attempts=5, base=0.5, cap=30.0):
    for i in range(attempts):
        try:
            return await fn()
        except RetryableError:
            if i == attempts - 1: raise
            await asyncio.sleep(random.uniform(0, min(cap, base * (2 ** i))))
```

---

# Section C — Kafka & scale

## Q10. Kafka or SQS?

### 🗣️ Say this
> "SQS for task queues — 'do this work, then forget about it'.
> Kafka when the *log itself* is valuable. That means several independent consumers needing the same stream, or needing replay, or event sourcing, or strict per-key ordering at high throughput.
> Replay is usually what decides it. With SQS, a consumer bug that corrupted data for six hours is unrecoverable — the messages are gone. With Kafka you fix the code and rewind the offset.
> But I'd be clear that Kafka is a serious operational commitment. For a team of six running a handful of services, SQS plus SNS does nearly everything Kafka would and costs nobody a pager. I'd want a real reason."

| | SQS | Kafka |
|---|---|---|
| Model | Queue — consumed and gone | Log — retained, consumers track offsets |
| Replay | ❌ | ✅ Rewind and reprocess |
| Many consumers | Need SNS fan-out | Native — each group has its own offset |
| Ops burden | ~zero | Real, unless MSK/Confluent |

---

## Q11. Kafka ordering, partitions and consumer groups.

### 🗣️ Say this
> "Ordering is guaranteed **within a partition only**. The producer hashes the message key to pick a partition, so all events for the same key land in the same partition and stay ordered relative to each other. Null key means round-robin and no meaningful order at all.
> Partitions are also the parallelism unit — within a consumer group, one partition goes to at most one consumer. So with six partitions, a seventh consumer just sits idle. Your maximum parallelism is the partition count.
> And you can increase partitions but never decrease them, and increasing them changes the key-to-partition mapping, which breaks ordering for keys in flight. So it's a decision worth slightly over-provisioning at the start."

### ⚠️ Follow-up: "What's a rebalance and why is it painful?"
> "When a consumer joins or leaves, the group reassigns partitions. With the classic protocol everyone stops consuming during it — a stop-the-world pause. So a consumer that's slow to poll gets kicked out, which triggers a rebalance, which slows everyone, which causes more evictions. It's a feedback loop.
> Cooperative sticky assignment largely fixes it and should be the default now. On the application side, keep per-message processing well under the poll interval."

---

## Q12. Consumer lag is growing. Diagnose it.

### 🗣️ Say this
> "Lag growing means arrival rate is beating processing rate. There are three causes and they need completely different responses, so I'd separate them first.
> One: the producer rate genuinely went up — that's a capacity problem, and adding consumers is right.
> Two: processing got slower, usually because a downstream dependency degraded. Here adding consumers actively makes it worse, because you just pile more load onto the thing that's already struggling.
> Three: consumers are unhealthy — crash-looping, stuck rebalancing, or one partition is hot because of a skewed key. And if it's a hot partition, adding consumers does *nothing*, because the extra ones sit idle while one partition backs up. The fix there is the key.
> So the order matters: diagnose which of the three, then pick the lever."

📐 **Alert on the lag *trend*, not the absolute number.** Lag of 50,000 that's shrinking is fine; lag of 500 growing steadily is an incident. The useful derived metric is estimated time-to-drain.

---

## Q13. What is backpressure?

### 🗣️ Say this
> "Backpressure is signalling upstream to slow down when you can't keep up, instead of silently queueing forever. Queues give you it naturally — the queue absorbs a burst and consumers work at their own pace. That's the main reason to put one between an API and slow work.
> But an unbounded queue just converts a throughput problem into a latency problem. Messages sit for hours and by the time they're processed the result is worthless. So queue depth has to be an alert with a documented decision about what to shed.
> Where it's usually missing: a consumer with unlimited concurrency that fans out to the database until the connection pool is exhausted — that needs a semaphore.
> And on the synchronous path, if you accept requests faster than you can serve them, the honest answer is load shedding. Return 429 fast, because a request the client already timed out on is work you're doing for nobody."

📐 **The three numbers to alert on for any queue:** depth, **age of the oldest message** (the single best indicator), and DLQ depth (any non-zero value).

---

## ⚡ Rapid fire

| They ask | Say this |
|---|---|
| Message bigger than 256 KB? | "Claim check — put the payload in S3 and send the key. The extended client library does it for you." |
| Why long polling? | "`WaitTimeSeconds=20`. Fewer empty receives, lower cost, lower latency. Always on." |
| Delay a message more than 15 minutes? | "SQS caps at 15. Beyond that, a `scheduled` table with a `send_at` index polled by a worker, or EventBridge Scheduler." |
| Priority queues in SQS? | "Not natively. Two queues, and a consumer that polls the high-priority one first." |
| Celery or SQS? | "Celery gives you tasks, chaining, scheduling and results out of the box. SQS gives you managed infrastructure with no broker to run. Celery can use SQS as its broker, which is often the sweet spot — though SQS lacks some features Celery expects, like priority." |
| FastAPI `BackgroundTasks` — when? | "Only for work that's genuinely OK to lose. It runs in the same process, so a pod restart during a rolling deploy silently drops it. A nice-to-have notification, yes. Anything with a business consequence, no." |
| Job status API shape? | "`POST /jobs` returns 202 with a `Location` header. `GET /jobs/{id}` returns status, progress, result URL or error. States: queued → running → succeeded/failed/cancelled." |
| How do you cancel a running job? | "You can't interrupt it from outside. You set a flag the worker checks at safe points — so you have to design those checkpoints in from the start." |
| Trace across a queue? | "Inject the W3C `traceparent` into message attributes on publish, extract it on consume and use it as the parent context." |
| Ordering with SQS standard? | "You can't. Either FIFO with a `MessageGroupId`, or put a sequence number in the payload and let consumers handle reordering." |

---

**Next:** [06-aws-for-backend.md](06-aws-for-backend.md)

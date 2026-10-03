# 08 — OpenTelemetry, Senior Level

> **How to use this file.** Every question has a **🗣️ Say this** block — 4 to 5 lines, ready to speak.

**This is your differentiator.** You have a real OTel implementation in this repo — `gitlab-user-service/app/core/telemetry.py`, three working demo stacks under `observability-demos/`, and a migration from an open-source stack to Dynatrace. Most senior backend candidates can define a span. Very few have configured a Collector pipeline or made a sampling decision with a cost attached. Lead with the specifics.

---

## 📖 Jargon, in plain words

| Jargon | What it actually means | How you'd say it |
|---|---|---|
| **Trace** | The full story of one request as it moves through your services. | "A trace is one request end to end, across every service." |
| **Span** | One step inside that story, with a start and end time. | "A span is a single operation — one query, one HTTP call." |
| **Parent span** | The step that caused this one. It's what makes a trace a tree. | "Parent links are what build the waterfall view." |
| **Trace ID** | The shared ID stitching all those spans together. | "Every span in one request carries the same trace ID." |
| **Context propagation** | Passing the trace ID to the next service so it can continue the trace. | "Over HTTP it's the `traceparent` header." |
| **Resource** | Fixed facts about *who* is emitting — service name, version, environment. | "The Resource is the identity of the process." |
| **Attribute** | A key-value tag on a span. Like a log field, but structured. | "Attributes are where business context goes." |
| **Span link** | A weaker connection than parent-child, for async fan-out. | "Links connect related traces without merging them into one." |
| **Exporter** | The bit that ships your data somewhere. | "OTLP is the standard wire format." |
| **Collector** | A separate process that receives, processes and forwards telemetry. | "It's the control point — sampling and redaction live there, not in code." |
| **Head sampling** | Decide whether to keep a trace at the very start. | "Cheap, but you decide before knowing if it's interesting." |
| **Tail sampling** | Decide after the whole trace is finished, in the Collector. | "It lets me keep 100% of errors and 1% of the boring ones." |
| **Cardinality** | How many distinct label combinations a metric has. | "It's the cost driver — one bad label multiplies your series by millions." |
| **Exemplar** | A trace ID attached to a metric point, linking a graph to a real request. | "It's the jump from 'p99 is bad' to 'here's a slow request'." |
| **Semantic conventions** | Agreed standard names for attributes, like `http.request.method`. | "Following them is what makes vendor dashboards work out of the box." |

---

# Section A — How it works

## Q1. Draw the architecture and tell me where you'd intervene.

### 🗣️ Say this
> "Inside the process: the API is what your code calls, and it's a no-op unless an SDK is registered — that's deliberate, so a library can instrument itself without forcing a dependency on you. The SDK implements it, holding a Resource and a list of span processors. When a span ends it goes to the `BatchSpanProcessor`, which queues it and ships it on a background thread over OTLP.
> Outside the process: the Collector, usually a DaemonSet agent on each node plus a gateway deployment.
> And the architectural point I'd make is that **the Collector is where the policy should live** — sampling, redaction, routing to multiple backends. If sampling logic is in the application, changing it means redeploying every service. In the Collector it's a config change.
> That separation is the single most valuable decision in an observability setup."

### 💻 The shape
```
[Your app]  OTel API → SDK → BatchSpanProcessor → OTLP exporter
                 │
                 ↓ OTLP/gRPC :4317  (same-node hop, never blocks on the network)
[Collector agent — DaemonSet]  receivers → k8sattributes → batch → export
                 │
                 ↓
[Collector gateway]  tail_sampling · redaction · routing
                 ↓             ↓              ↓
           [Tempo/Jaeger]  [Prometheus]  [Dynatrace]
```

### ⚠️ Follow-up: "Why both an agent and a gateway?"
> "The agent is local, so exporting is a same-node hop that never blocks on a network problem, and it enriches spans with Kubernetes metadata.
> The gateway is where you do anything needing a *complete* trace — which is tail sampling. And there's a detail people miss: tail sampling only works if every span of a trace reaches the *same* gateway instance. So you need a load-balancing exporter routing by trace ID in front of it. Without that, your tail sampling silently drops half of every trace."

---

## Q2. Auto-instrumentation vs manual — what does each give you?

### 🗣️ Say this
> "Auto-instrumentation gets you the plumbing for free — HTTP in, HTTP out, SQL queries, Redis, AWS calls. That's most of the latency in a typical service, so it's very high value for almost no effort.
> What it can't give you is business meaning. It knows a request took 400 milliseconds; it doesn't know it was a bulk provisioning request for an enterprise-tier tenant.
> So: auto for everything infrastructural, manual spans for business operations and business attributes.
> The rule I apply is that a span should represent something a human would name in a sentence describing what happened. If a span never changes a decision, it shouldn't exist — because a span per function multiplies your cost and makes the waterfall unreadable."

### 💻
```python
# Auto: the libraries, for free
FastAPIInstrumentor.instrument_app(app)
SQLAlchemyInstrumentor().instrument(engine=engine.sync_engine)
HTTPXClientInstrumentor().instrument()

# Manual: the business semantics auto-instrumentation cannot know
tracer = trace.get_tracer(__name__)

async def provision_repository(req: RepoRequest) -> Repo:
    with tracer.start_as_current_span("provision_repository") as span:
        span.set_attribute("repo.format", req.format)
        span.set_attribute("repo.tenant_id", req.tenant_id)
        try:
            repo = await self._client.create(req)
            span.set_status(Status(StatusCode.OK))
            return repo
        except NexusConflict as e:
            span.set_status(Status(StatusCode.ERROR, "repository already exists"))
            span.record_exception(e)
            raise
```

---

## Q3. How does trace context propagate — including across a queue?

### 🗣️ Say this
> "Over HTTP it's a W3C standard header called `traceparent` — version, trace ID, parent span ID, and a flag saying whether it's sampled. Between two auto-instrumented services it just works with no code.
> Across a message queue it does **not** just work, and this is where most distributed traces quietly break.
> The producer has to inject the context into the message attributes, and the consumer has to extract it and use it as the parent. Without that you get two disconnected traces and you can't answer 'what happened to this order' end to end — which is the exact question you bought tracing for.
> One nuance: for wide fan-out, a strict parent-child link is wrong. If one producer publishes 500 messages, one trace containing all of them is unusable — that's what span *links* are for."

### 💻 The header
```
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
             ^^ ^--------- trace id (32 hex) --------^ ^-span id-^ ^^
             version                                              flags (01 = sampled)
```

### 💻 Across SQS
```python
from opentelemetry.propagate import inject, extract

# PRODUCER
carrier: dict[str, str] = {}
inject(carrier)                                  # writes traceparent into the dict
sqs.send_message(QueueUrl=q, MessageBody=body,
    MessageAttributes={k: {"StringValue": v, "DataType": "String"}
                       for k, v in carrier.items()})

# CONSUMER
carrier = {k: v["StringValue"] for k, v in msg.get("MessageAttributes", {}).items()}
ctx = extract(carrier)
with tracer.start_as_current_span("process_order", context=ctx,
                                  kind=SpanKind.CONSUMER) as span:
    ...
```
⚠️ **Inside the process**, propagation is `contextvars` — the same mechanism as [01-python-advanced.md](01-python-advanced.md) Q8. That's why context is lost if you hand work to a thread pool without copying it.

---

# Section B — Sampling & cost

## Q4. How do you sample, and what does it cost you?

### 🗣️ Say this
> "Sampling exists because tracing is the most expensive signal by volume. At a thousand requests a second with ten spans each, that's ten thousand spans a second — at any realistic ingest price that's a bill nobody approved.
> Head sampling decides at the start of a trace and propagates that decision to every service, so a trace is all-or-nothing. It's cheap because unsampled spans are never even created. But it decides *before* knowing whether the trace is interesting — so at 5% you throw away 95% of your errors, which is exactly backwards.
> Tail sampling decides after the trace finishes, in the Collector, so I can keep 100% of errors and slow traces and 1% of successful ones.
> The cost is that you still generate and transmit everything to the gateway, so you save on storage but not on Collector capacity."

### 💻 The policy I'd actually deploy
```yaml
processors:
  tail_sampling:
    decision_wait: 10s              # MUST exceed your p99.9 trace duration
    num_traces: 100000
    policies:
      - name: errors                # keep every failure
        type: status_code
        status_code: {status_codes: [ERROR]}
      - name: slow                  # keep everything slow
        type: latency
        latency: {threshold_ms: 1000}
      - name: baseline              # 1% of the boring ones, for shape
        type: probabilistic
        probabilistic: {sampling_percentage: 1}
```

### ⚠️ The setting that causes silent data loss
> "`decision_wait`. If it's shorter than your slowest traces, the decision fires before the final spans arrive and those spans are dropped. So your *slowest* traces — the ones you care most about — are exactly the ones that come back incomplete. It has to exceed your p99.9 trace duration."

---

## Q5. Explain metric cardinality with numbers.

### 🗣️ Say this
> "Cardinality is how many distinct label combinations a metric has, and the cost is multiplicative because each combination is its own time series, stored and indexed separately.
> Concretely: `http_requests_total` with method, status and endpoint is maybe 800 series per pod — completely fine. Add `user_id` with a hundred thousand users and it's 80 million series per pod. That doesn't make Prometheus expensive, it takes Prometheus down.
> So the rule is that a label's value set has to be bounded and small, and you have to know the bound at design time. If you can't state the maximum number of distinct values, it isn't a label — it's a span attribute or a log field.
> The FastAPI-specific version of this bug is labelling by the URL path instead of the route template."

### 💻 The bug and the fix
```python
# ❌ WRONG — unbounded. /users/8d3f.../orders is a new series every time
REQUESTS.labels(path=request.url.path).inc()

# ✅ RIGHT — the route TEMPLATE, which is bounded by your number of routes
route = request.scope.get("route")
path = route.path if route else "unmatched"     # /users/{id}/orders
REQUESTS.labels(method=request.method, path=path,
                status=response.status_code).inc()
```
```
Cardinality arithmetic:
  method(5) × status(8) × route(20)            =        800 series   ✅
  method(5) × status(8) × raw_path(100,000)    = 4,000,000 series   💥
```
> Note `"unmatched"` matters too — otherwise a scanner hitting random URLs creates unbounded cardinality through the back door.

| ❌ Never a label | ✅ Put it here instead |
|---|---|
| `user_id`, `request_id`, `trace_id` | Span attribute or log field |
| Raw URL with IDs in it | The route template |
| Error message text | An error *type* enum |
| Raw SQL | Operation name + table |

---

## Q6. Your observability bill tripled. What do you do?

### 🗣️ Say this
> "I'd split the bill by signal first, because the fix is different for each and they're rarely proportional to their usefulness. In my experience logs are the biggest line item and the least valuable per byte.
> For logs: find the top talkers. It's almost always a debug statement left on, or health-check logging every two seconds across fifty pods, or full request bodies being logged. Drop health checks at the Collector, sample INFO, keep WARN and ERROR at 100%, and route long-term retention to S3 rather than the expensive indexed store.
> For traces: tail sampling, and check for over-instrumentation.
> For metrics: cardinality — it's usually one or two metrics with a bad label rather than general growth.
> And the structural point: all three of those fixes are **Collector configuration**, not application changes. That's the argument for having the Collector in place before you need it — when the bill spikes you respond in minutes with a config push, not a coordinated redeploy of thirty services."

### 💻
```yaml
processors:
  filter/drop_health_checks:
    traces:
      span:
        - 'attributes["http.route"] == "/healthz"'
        - 'attributes["http.route"] == "/readyz"'
  transform/redact:
    trace_statements:
      - context: span
        statements:
          - delete_key(attributes, "http.request.header.authorization")
          - replace_pattern(attributes["db.statement"], "'[^']*'", "'?'")
```

---

# Section C — Making it useful

## Q7. How do you correlate logs, metrics and traces?

### 🗣️ Say this
> "Three links, and then you can move freely between all three.
> One: every log line carries the trace ID and span ID, so from a log you can jump straight to the trace.
> Two: exemplars, which attach a trace ID to histogram buckets — so from a latency spike on a dashboard you click directly into an actual slow request. That's the single most useful correlation there is, because it turns 'p99 is two seconds' into 'here is a two-second request and here's exactly where it spent the time'.
> Three: consistent Resource attributes — service name, version, environment — so all three signals filter identically.
> And having `service.version` on the Resource is what lets me compare error rates between two versions during a canary."

### 💻
```python
class TraceContextFilter(logging.Filter):
    def filter(self, record):
        ctx = trace.get_current_span().get_span_context()
        record.trace_id = format(ctx.trace_id, "032x") if ctx.is_valid else "-"
        record.span_id  = format(ctx.span_id, "016x")  if ctx.is_valid else "-"
        return True
```
```
Output — a log line that's now clickable:
{"ts":"2026-09-21T10:14:22Z","level":"ERROR","msg":"nexus_call_failed",
 "trace_id":"4bf92f3577b34da6a3ce929d0e0e4736","span_id":"00f067aa0ba902b7",
 "service.name":"gitlab-user-service","service.version":"1.4.2"}
```

---

## Q8. "Latency went up 40 minutes ago." Use your telemetry.

### 🗣️ Say this
> "**Narrow before you deepen.** First question: is it everything or one endpoint? A latency heatmap by route answers that instantly, and the two answers send me in completely different directions — 'everything' means a shared resource like the database or the event loop; 'one endpoint' means that endpoint's code or its specific downstream.
> Second: did anything change? Deploy annotations on the dashboard. Forty minutes is suspiciously deploy-shaped, and correlating with a deploy is faster than any amount of profiling.
> Third: get an exemplar — click from the p99 bucket into a real slow trace and read the waterfall. If it's a database span, I go to `pg_stat_statements`. If the total time is much bigger than the sum of the children, the time is in our code or the loop is lagging.
> Fourth, and this one people miss: is it latency or is it *queueing*? A saturated connection pool shows as time waiting to acquire a connection, not as slow queries. Those look identical on a p99 graph and have opposite fixes."

📐 **Four golden signals:** latency, traffic, errors, saturation. **RED** for services: Rate, Errors, Duration. **USE** for resources: Utilisation, Saturation, Errors.
> "Saturation is the one people omit and it's the *leading* indicator — pool utilisation, queue depth and event-loop lag all predict the latency spike before it happens."

---

## Q9. What makes an alert good?

### 🗣️ Say this
> "Three tests: actionable, urgent, and about a symptom.
> Actionable means there's a documented thing the person woken up can actually do. Urgent means it needs doing now, not Monday — otherwise it's a ticket, not a page. Symptom-based means alerting on 'users are getting errors', not 'CPU is above 80%', because high CPU with happy users is not an incident.
> What I'd actually configure: SLO burn rate as the primary page, multi-window so a fast burn pages immediately and a slow burn just opens a ticket. Then saturation predictors — disk filling, pool near capacity, queue age growing. Then the ones that are always incidents: anything in a DLQ, certificate expiry, a failed backup.
> And organisationally I'd push for every page to be reviewed — if it fired and needed no action, it gets tuned or deleted. Alert fatigue is a reliability problem in itself."

### 💻 Burn-rate alerting
```yaml
- alert: ErrorBudgetFastBurn
  expr: |
    (1 - (sum(rate(http_requests_total{status!~"5.."}[5m]))
        / sum(rate(http_requests_total[5m])))) > (14.4 * 0.001)
    and
    (1 - (sum(rate(http_requests_total{status!~"5.."}[1h]))
        / sum(rate(http_requests_total[1h])))) > (14.4 * 0.001)
  for: 2m
  labels: {severity: page}
  annotations:
    summary: "Burning 99.9% error budget 14.4x — 2% of the month in 1 hour"
```
> "The 14.4 is the standard multiplier from the Google SRE workbook — at that rate you'd burn a month's budget in about two days. The two-window condition stops it flapping once the burn stops."

---

## Q10. Talk me through your OSS-to-Dynatrace migration.

### 🗣️ Say this — **this is your strongest story, rehearse it**
> "What we had: OpenTelemetry SDK in the services exporting OTLP to a Collector, then Jaeger for traces, Prometheus for metrics, Grafana for dashboards and Fluent Bit shipping logs. It worked and it was cheap.
> Why we moved: *[your actual reason — correlation across signals, the operational burden of self-hosting, enterprise support, single pane of glass]*.
> What made it cheap to do — and this is the part I'd emphasise, because it was an architectural decision that paid off: the application code was instrumented with vendor-neutral OpenTelemetry and exported OTLP to a Collector. So the migration was a change to the **Collector's exporter configuration**. The services were never touched and never redeployed, and during the transition we exported to both backends at once and compared them.
> That's exactly why I'd argue for OTel plus a Collector even when you already know which vendor you're using — it keeps the switching cost in config rather than in code."

### ⚠️ Be ready for: "So why not just use the vendor agent?"
> "Vendor agents give you deeper automatic instrumentation for less work, and that's a genuine advantage I wouldn't dismiss. The cost is that your instrumentation is now proprietary, so the *next* migration is a code change across every service rather than a config change.
> It's a real trade-off and it depends how likely you think that migration is. My preference is OTel for instrumentation and the vendor for the backend — which is also where the industry is going, since they all accept OTLP natively now."

---

## ⚡ Rapid fire

| They ask | Say this |
|---|---|
| `BatchSpanProcessor` vs `Simple`? | "Simple exports synchronously when a span ends — fine for tests, terrible in production. Batch queues and ships on a background thread." |
| What's in a span? | "Name, trace ID, span ID, parent ID, start and end time, kind, status, attributes, events and links." |
| Span kinds? | "SERVER, CLIENT, PRODUCER, CONSUMER, INTERNAL. They tell the backend how to reconstruct the call graph." |
| Trace ID vs correlation ID? | "A trace ID is a standardised 128-bit ID with a whole span tree behind it. A correlation ID is an ad-hoc header for grepping logs. Use the trace ID as your correlation ID." |
| Span event or a log? | "A span event is a timestamped note inside a span — cheap and already correlated. Prefer it for anything belonging to one operation." |
| Metric types? | "Counter, UpDownCounter, Histogram, Gauge, plus observable versions that are read on a schedule." |
| Delta vs cumulative? | "Cumulative reports totals since start — that's Prometheus style. Delta reports the change per interval, which Dynatrace and CloudWatch prefer. The Collector converts between them." |
| How do you avoid losing spans on shutdown? | "Call `tracer_provider.shutdown()` in the lifespan teardown to force-flush the batch queue. Our `Telemetry.shutdown()` does exactly that." |
| Overhead of tracing? | "Low single-digit percent CPU with batching and sampling. Measure it — the bigger real cost is the bill." |
| Why semantic conventions? | "Vendor dashboards and alerts work out of the box. `http.request.method` is a name their product already understands; `my_http_verb` isn't." |
| `service.name` missing? | "Everything lands under `unknown_service` and correlation breaks. It's the one mandatory Resource attribute." |
| Sensitive data in spans? | "SQL parameters, auth headers, PII in URLs. Redact at the Collector with the transform processor so it never reaches the vendor at all." |

---

**Next:** [09-system-design-advanced.md](09-system-design-advanced.md)

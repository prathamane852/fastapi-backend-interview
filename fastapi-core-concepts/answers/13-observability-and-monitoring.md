# 13 — Observability & Monitoring

---

## 📖 Vocabulary

> *Definition* is the precise meaning; *Say it in an interview* is the quotable one-liner.

| Term | Definition | Say it in an interview |
|---|---|---|
| **Observability** | The ability to infer a system's internal state from its external outputs — metrics, logs and traces — well enough to answer questions you didn't anticipate. | "Observability lets me debug unknown failures from the outside." |
| **Monitoring** | Collecting and alerting on predefined metrics and conditions. A subset of observability. | "Monitoring watches known signals; observability explores unknowns." |
| **Metric** | A numeric measurement sampled over time, stored as a time series with labels. | "Metrics are cheap aggregates for dashboards and alerts." |
| **Counter** | A monotonically increasing metric (resets only on restart); queried with `rate()`. | "Counters count events; use `rate()` to get per-second." |
| **Gauge** | A metric representing a current value that can rise or fall. | "Gauges are point-in-time values like in-flight requests." |
| **Histogram** | A metric recording observations into cumulative buckets with `_sum` and `_count`, enabling server-side percentile estimation. | "Histograms let me compute p99 across many pods." |
| **Summary** | A metric computing quantiles client-side; not aggregatable across instances. | "Summaries can't be aggregated, so I prefer histograms." |
| **Label** | A key/value dimension attached to a metric series (`route`, `status`). | "Labels slice metrics by dimension." |
| **Cardinality** | The number of unique series produced by label combinations; unbounded labels (user IDs, raw paths) exhaust memory. | "Keep labels low-cardinality or the metrics system falls over." |
| **Prometheus** | An open-source pull-based monitoring system: scrapes `/metrics`, stores time series, evaluates PromQL alert rules, integrates with Alertmanager. | "Prometheus scrapes, stores and alerts on metrics." |
| **PromQL** | Prometheus's query language for selecting, aggregating and computing over time series. | "PromQL is how I turn raw series into rates and percentiles." |
| **Exposition format** | The text format served at `/metrics` that Prometheus parses. | "`generate_latest()` emits the exposition format." |
| **Grafana** | A visualization and dashboard platform querying Prometheus, Loki, Tempo and others, with alerting and annotations. | "Grafana is the single pane for metrics, logs and traces." |
| **Log** | A timestamped record of a discrete event with context; structured (JSON) logs are queryable by field. | "Logs explain what happened to a specific request." |
| **Trace** | The complete record of one request's execution across services: a tree of spans sharing a trace ID. | "A trace is the end-to-end story of one request." |
| **Span** | A single timed operation within a trace with a name, start/end, attributes, events, status, span ID and parent span ID. | "Spans are the nodes of the trace tree." |
| **Trace ID / Span ID** | Trace ID: 128-bit identifier shared by all spans in a trace. Span ID: 64-bit identifier of one span. | "Trace ID groups; span ID identifies." |
| **Context propagation** | Carrying trace identity across process boundaries via headers (`traceparent`) or message attributes so child spans join the parent trace. | "Propagation is what makes the trace continue into the next service." |
| **`traceparent`** | W3C Trace Context header: `version-traceid-parentid-flags`. | "`traceparent` is the standard header carrying trace context." |
| **Baggage** | W3C header for propagating arbitrary key/value context (tenant ID) alongside the trace. | "Baggage carries business context across services with the trace." |
| **OpenTelemetry (OTel)** | CNCF standard, APIs, SDKs, auto-instrumentation and Collector for traces, metrics and logs; wire protocol OTLP. | "OTel is vendor-neutral instrumentation for all three signals." |
| **OTLP** | OpenTelemetry Protocol: the gRPC/HTTP wire format for exporting telemetry. | "OTLP is what the SDK speaks to the Collector." |
| **Collector** | An OTel service that receives telemetry, applies processors (batching, sampling, redaction, enrichment) and exports to backends. | "The Collector centralizes sampling and PII scrubbing." |
| **Sampling (head vs tail)** | Head-based: decide at trace start (ratio). Tail-based: decide after completion in the Collector, keeping errors and slow traces. | "Tail sampling keeps 100% of errors and a fraction of the rest." |
| **Exemplar** | A trace ID attached to a histogram bucket sample, linking a metric spike to an example trace. | "Exemplars jump from a latency spike straight to a trace." |
| **Correlation ID** | An application-level identifier grouping related log entries for one operation; often the trace ID or `X-Request-ID`. | "I use the trace ID as the correlation ID in logs." |
| **SLI / SLO / SLA** | SLI: a measured indicator (success ratio). SLO: an internal target (99.9%). SLA: a contractual commitment with consequences. | "SLIs measure, SLOs target, SLAs promise." |
| **Error budget** | The permitted failure under an SLO (100% − SLO); spent by incidents and risky changes. | "The error budget decides when to slow releases." |
| **Burn rate** | The rate at which the error budget is being consumed relative to the SLO window; multi-window burn-rate alerts page on fast burns. | "Burn-rate alerts page on symptoms, not on every blip." |
| **Four golden signals** | Latency, traffic, errors, saturation (Google SRE). | "Golden signals: latency, traffic, errors, saturation." |
| **RED / USE** | RED (Rate, Errors, Duration) for request-driven services; USE (Utilization, Saturation, Errors) for resources. | "RED for services, USE for resources." |
| **Runbook** | Documented, step-by-step diagnostic and remediation procedure linked from an alert. | "Every alert links to a runbook." |
| **Synthetic monitoring** | Scripted external probes exercising endpoints on a schedule to detect outages from the user's vantage point. | "Synthetics catch outages before users report them." |
| **`py-spy`** | A sampling profiler attaching to a live Python process without restart; `top` and `dump` modes. | "py-spy dump shows what a stuck worker is doing." |
| **`tracemalloc` / `memray`** | Stdlib and third-party memory profilers tracking allocations by call site. | "memray finds where the leaked memory was allocated." |
| **CFS throttling** | Container CPU-limit enforcement pausing the process each period; visible as `container_cpu_cfs_throttled_*`. | "Throttling metrics reveal CPU limits hurting latency." |
| **OOMKilled** | Kernel termination of a container exceeding its memory limit; exit code 137. | "137 means the kernel killed it for memory." |

---

## Q1. How do you monitor a FastAPI application?

The three pillars, tied together by IDs:
- **Metrics** (Prometheus) — numbers over time; good for alerting.
- **Logs** (structured JSON → Loki/ELK/CloudWatch) — discrete events; good for "what happened to request X".
- **Traces** (OpenTelemetry → Tempo/Jaeger/X-Ray) — one request across services; good for "where did the time go".
Plus health probes, synthetic checks from outside, and error tracking (Sentry). Every log line carries `trace_id`/`request_id` so you can pivot alert → trace → logs.

## Q2. What metrics would you expose?

```
http_requests_total{method,route,status}                 counter    → rate, error ratio
http_request_duration_seconds{method,route}              histogram  → p50/p95/p99
http_requests_in_progress{route}                         gauge      → concurrency
db_pool_connections_in_use / db_pool_connections_waiting gauge
db_query_duration_seconds{operation}                     histogram
external_call_duration_seconds{target}                   histogram
external_call_failures_total{target,reason}              counter
cache_requests_total{result="hit|miss"}                  counter    → hit ratio
event_loop_lag_seconds                                   gauge      → blocking detection
background_jobs_total{type,status}, queue_depth{queue}   counter/gauge
process_resident_memory_bytes, process_cpu_seconds_total (default exporter)
build_info{version,commit}                               gauge=1    → correlate deploys
```
Keep label cardinality bounded: route templates, not raw paths; never user IDs.

## Q3. What is Prometheus?

An open-source, **pull-based** time-series database: it scrapes `/metrics` on an interval, stores samples, and evaluates alert rules written in PromQL. Metric types: counter, gauge, histogram, summary. Grafana visualises; Alertmanager routes alerts.

## Q4. How do you expose /metrics?

```python
import time
from prometheus_client import Counter, Histogram, Gauge, generate_latest, CONTENT_TYPE_LATEST
from fastapi import Response
from starlette.middleware.base import BaseHTTPMiddleware

REQUESTS = Counter("http_requests_total", "Total requests", ["method", "route", "status"])
LATENCY = Histogram("http_request_duration_seconds", "Latency", ["method", "route"],
                    buckets=(.005, .01, .025, .05, .1, .25, .5, 1, 2.5, 5, 10))
IN_PROGRESS = Gauge("http_requests_in_progress", "In-flight", ["route"])

class MetricsMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request, call_next):
        route = "unmatched"
        IN_PROGRESS.labels(route).inc()
        start = time.perf_counter()
        status = 500
        try:
            response = await call_next(request)
            status = response.status_code
            return response
        finally:
            r = request.scope.get("route")
            route = r.path if r else "unmatched"
            LATENCY.labels(request.method, route).observe(time.perf_counter() - start)
            REQUESTS.labels(request.method, route, status).inc()
            IN_PROGRESS.labels(route).dec()

@app.get("/metrics", include_in_schema=False)
def metrics():
    return Response(generate_latest(), media_type=CONTENT_TYPE_LATEST)
```
### 🔍 Line by line
- `Counter(name, help, labelnames)` / `Histogram(..., buckets=...)` / `Gauge` — define metrics once at import.
- `buckets` — the latency boundaries; choose them around your SLO.
- `.labels(...)` — select the label combination; `.inc()`, `.observe(seconds)`, `.dec()`.
- `finally:` — record even when the handler raised (status stays 500).
- `request.scope.get("route")` — after `call_next`, the matched route object; `.path` is the template `/orders/{id}`.
- `/metrics` returns the text exposition format Prometheus scrapes.
- Or use `prometheus-fastapi-instrumentator`. With multiple Uvicorn workers set `PROMETHEUS_MULTIPROC_DIR`, or run one worker per pod and scrape each pod (simpler).

## Q5. What is OpenTelemetry?

The CNCF standard for generating and exporting telemetry — traces, metrics, logs — through one SDK, auto-instrumentation libraries, the OTLP wire protocol, and a **Collector** that receives, processes and exports to any backend. It replaced OpenTracing and OpenCensus.

## Q6. What is distributed tracing?

Following one request as it fans out across services, queues and databases, recording timing at each hop, so you see the whole chain as one waterfall. It answers "which of the seven services made this checkout slow?" — something per-service logs can't.

## Q7. What is a trace?

The complete record of one request's journey: a tree of spans sharing one **trace ID** (128-bit). The root span is the entry point.

## Q8. What is a span?

One unit of work inside a trace: a name (`GET /orders/{id}`, `SELECT orders`), start/end times, a **span ID**, a **parent span ID**, attributes (`http.status_code`, `db.statement`), events, and status. Spans nest: server span → dependency span → SQL span.

## Q9. What is a correlation ID?

An application-level identifier (usually a UUID) attached to a request and carried through logs and downstream calls so all lines for one operation can be grouped; often returned to clients as `X-Request-ID`.

## Q10. Correlation ID vs trace ID?

| | Correlation / Request ID | Trace ID |
|---|---|---|
| Origin | App convention | W3C standard, OTel SDK |
| Scope | One operation, mainly logs | Whole distributed trace, structured spans |
| Timing data | ❌ | ✅ |
| Client-facing | Usually | Sometimes |

In practice: use the OTel trace ID *as* the correlation ID (inject into logs, echo as `X-Request-ID`); keep a separate business ID only for multi-request workflows (an order ID spanning many traces).

## Q11. How do you propagate trace context between microservices?

> 🔴 **Moved to the advanced section** → [../../advanced-backend/08-opentelemetry-advanced.md](../../advanced-backend/08-opentelemetry-advanced.md)
> 
> Q3 — W3C propagation, and how to carry context across a queue.

---


## Q12. How would you instrument FastAPI with OpenTelemetry?

> 🔴 **Moved to the advanced section** → [../../advanced-backend/08-opentelemetry-advanced.md](../../advanced-backend/08-opentelemetry-advanced.md)
> 
> Q2 — auto vs manual instrumentation and the over-instrumentation trap.

---

## Q13. How would you monitor external API latency?

Wrap outbound clients in a histogram + counter per `target` (automatic with OTel/httpx instrumentation, or an `httpx` event hook):
```python
async def on_response(response: httpx.Response):
    EXT_LATENCY.labels(response.request.url.host).observe(response.elapsed.total_seconds())
    EXT_CALLS.labels(response.request.url.host, str(response.status_code)).inc()

client = httpx.AsyncClient(event_hooks={"response": [on_response]})
```
Dashboards per dependency (p99, error rate, timeout rate, breaker state); alert when a dependency exceeds *its* budget before your SLO burns; synthetic checks from outside to separate "they're slow" from "we can't reach them".

## Q14. How would you create useful Grafana dashboards?

Top-down per service:
1. **Header**: SLO status, error budget remaining, current version, active alerts.
2. **RED row**: rps, error rate (4xx/5xx separately), p50/p95/p99 — with deploy annotations.
3. **Per-endpoint table** sorted by p99 or error rate.
4. **Saturation**: CPU/memory vs limits, pod count vs HPA max, event-loop lag, DB pool, thread pool.
5. **Dependencies**: DB/Redis/external latency & errors.
6. **Async**: queue depth, job success/fail, DLQ size, consumer lag.
Principles: consistent units/colours; templated `$env`/`$service`; links from panels to traces/logs; one-screen on-call view; dashboards as code in git.

## Q15. What alerts would you create?

Alert on **symptoms** users feel, using SLO **burn-rate** rules:
- Availability: 5xx ratio burn rate > 14.4× over 1 h *and* 5 m (page); > 6× over 6 h (ticket).
- Latency: p99 above SLO for 10 m.
- Traffic drops to ~0.
- Saturation: DB pool waiting > 0 for 5 m; memory > 85% of limit; HPA at max for 15 m.
- Async: DLQ depth > 0; consumer lag growing; job failure rate > 5%.
- Infra: CrashLoopBackOff, restarts > 3 in 15 m, cert expiry < 14 d.
- Dependency: circuit breaker open.
Every alert has a runbook link, severity and owner; prune noisy alerts weekly.

## Q16. How would you detect memory leaks?

> 🔴 **Moved to the advanced section** → [../../advanced-backend/01-python-advanced.md](../../advanced-backend/01-python-advanced.md)
> 
> Q3 — `tracemalloc` snapshot diffing and the usual causes.

---

## Q17. How would you troubleshoot high CPU?

> 🔴 **Moved to the advanced section** → [../../advanced-backend/01-python-advanced.md](../../advanced-backend/01-python-advanced.md)
> 
> Q15 — `py-spy`, and why `cProfile` lies about async code.

---

## Q18. How would you troubleshoot increasing latency?

> 🔴 **Moved to the advanced section** → [../../advanced-backend/08-opentelemetry-advanced.md](../../advanced-backend/08-opentelemetry-advanced.md)
> 
> Q8 — the narrow-then-deepen method using exemplars.

---

## Q19. How would you troubleshoot HTTP 5xx spikes?

> 🔴 **Moved to the advanced section** → [../../advanced-backend/08-opentelemetry-advanced.md](../../advanced-backend/08-opentelemetry-advanced.md)
> 
> Q9 — SLO burn-rate alerting.

---

## Q20. What are the four golden signals?

From Google's SRE book — the minimum for any user-facing system:
1. **Latency** — time to serve; track successes and failures separately.
2. **Traffic** — demand (rps, sessions, messages/s).
3. **Errors** — failed request rate (explicit 5xx, wrong content, SLO misses).
4. **Saturation** — how full the service is (CPU, memory, pools, queues); predicts problems before latency/errors appear.
RED is the request-centric subset; USE is the resource-centric complement.

---
tags: [otca, opentelemetry, summary]
---

# OpenTelemetry Certified Associate (OTCA)

Summary of the KodeKloud prep course, with notes on the details that tend to show up in the exam.

1. Observability concepts|OpenTelemetry core concepts|Span anatomy and context propagation|Instrumentation|Metrics data model|Recording measurements|Logs|Collector foundations|Collector components|Transforming telemetry (OTTL, sampling, schemas)|OpenTelemetry in Kubernetes|Debugging, operations and scaling|SLIs and SLOs|Exam cheat sheet

My own notes based on the course's public lesson notes (notes.kodekloud.com). Diagrams made for this summary.

# 1. Observability concepts

## Distributed applications

A single user request, such as a checkout, crosses many services: the frontend, a cache, a catalog running in Kubernetes, pricing fed by Kafka, a third-party payment provider and shipping. When something fails or slows down, the problem could be at any hop. Without telemetry that connects those hops, debugging is guesswork.

## The three main signals

| Signal | What it is | What it's for |
|---|---|---|
| Metrics | Numeric values aggregated over time | Trends, alerts, spotting anomalies (latency, errors, throughput, CPU, saturation) |
| Logs | Timestamped events with context | Details of one specific failure, stack traces |
| Traces | The path of a request across services, with timings | Finding which hop is slow or failing |

The course uses a doctor analogy: symptoms are metrics, the patient's history is logs, and end-to-end tests are traces. There are also context signals: **baggage** (key-value pairs that travel with the request), **profiling** (where code spends CPU and memory) and business events.

The practical key is **correlation**: all three signals should share identifiers (`trace_id`, `service.name`) so you can jump from one to another.

## Monitoring vs observability

|  | Monitoring | Observability |
|---|---|---|
| Question | What is happening? | Why is it happening? |
| Signals | Metrics, thresholds, alerts | Metrics, logs, traces, profiles, baggage |
| Best for | Known failures, SLAs/SLIs | New or emergent failures, root cause |
| Workflow | Alert → runbook → fix | Detect → explore → hypothesize → fix |

They don't compete: monitoring flags the symptom and observability lets you investigate it. Together they reduce MTTR.

## Before OpenTelemetry

There are two collection models: **push** (the app or an agent sends the data) and **pull** (an external system scrapes it, like Prometheus with `/metrics`). The problem was fragmentation: every vendor had its own agent and SDK, switching backends meant re-instrumenting, and sending the same data to several destinations was hard. OpenTelemetry was created to standardize all of this.

# 2. OpenTelemetry core concepts

## What OTel is

An open-source CNCF framework, vendor-neutral and multi-language, for **generating, collecting, processing and exporting** telemetry. The correct short name is OTel. It **does not store or visualize** data: that is the job of whichever backend you choose (Prometheus, Jaeger, Zipkin, a commercial APM…).

Characteristics: vendor-neutral, open source, language-agnostic, portable, flexible and compatible with many backends.

## Goal, mission, vision and values

- **Goal:** instrumentation that is easy, consistent and universal.
- **Mission:** high-quality, portable telemetry for effective observability.
- **Vision:** easy to adopt, universal, vendor-neutral and **composable** (use only the pieces you need).
- **Engineering values:** compatibility, stability (backward-compatible APIs), resilience (instrumentation must never break the app) and performance (minimal overhead).

## The specification

It defines the behavior every implementation must follow:

- The APIs and SDKs of each signal.
- Semantic conventions (standard attribute names).
- OTLP (the transport protocol).
- Context propagation and baggage.
- Instrumentation guidelines, configuration (environment variables) and versioning.

Maturity levels: **Draft** (being designed) → **Experimental** (for testing, may change) → **Stable** (backward compatible, production-ready) → **Deprecated** (will be removed).

## Semantic conventions

Standard names so that the same concept is called the same thing across services, languages and signals. They make it easier to correlate, query and switch backends.

- Format: lowercase, dot-separated, with a domain prefix: `http.*`, `db.*`, `k8s.*`, `cloud.*`, `messaging.*`, `rpc.*`, `exception.*`.
- They cover resource attributes, span attributes, events, logs and metric names and units.
- Use the **Resource** for stable service-level information (`service.name`, `service.version`, `k8s.pod.uid`) and **span/event** attributes for per-operation details (`http.request.method`, `db.statement`).
- Careful: `db.system` is the database type (postgresql), not its name.
- Before inventing a custom attribute, check whether the conventions already define one.

> [!note]
> The course uses older names such as `http.method` and `http.status_code`. In the current stable conventions they are `http.request.method` and `http.response.status_code`. Know both.

## Main components

| Layer | Pieces |
|---|---|
| Standards | Specification, OTLP, semantic conventions |
| Code | API, SDK, instrumentation libraries, auto-instrumentation agents |
| Processing | Collector (receivers → processors → exporters) |
| Deployment | Kubernetes Operator, Helm charts, distributions |

## End-to-end architecture

![[otca-diagram-01.svg]]

*Typical flow. The Collector is optional: the SDK can export straight to the backend.*

1. Your code (or an instrumentation library) calls the **API**.
2. The **SDK** implements the API: sampling, processing, batching, resource detection.
3. The **exporter** (part of the SDK) sends data over OTLP to the Collector or straight to the backend.
4. The **Collector** receives, transforms and exports to one or more backends.
5. The backend stores and visualizes.

Each signal has its own pieces: traces (`TracerProvider` → `Tracer` → `Span`, with a `SpanProcessor`, sampler and exporter), metrics (`MeterProvider` → `Meter` → instruments, with views/aggregations and a reader/exporter) and logs (`LoggerProvider` → `Logger` → `LogRecord`, with log processors and exporters). **Propagators** serialize context between processes (W3C Trace Context).

## API vs SDK

| API (the "what") | SDK (the "how") |
|---|---|
| Stable interfaces for creating spans, metrics and logs and for propagating context | Implementation: sampling, processing, batching, aggregation, exporting |
| Without an SDK it runs in **no-op** mode: no errors, data is just discarded | Configured at application startup; can be changed without touching instrumentation |

- **Libraries depend only on the API.** They must never expose SDK types or configure exporters or samplers: that is decided by whoever deploys the application.
- Roles: instrumentation authors use the API; application owners configure the SDK; plugin authors implement exporters, processors or samplers.
- Client design principles: easy to use, uniform across languages, flexible, works out of the box and extensible.
- Each client is modular: API, SDK, semantic conventions and contrib (community integrations).
- OTel is a *cross-cutting concern*: it touches the whole application, which strains the separation-of-concerns principle. Splitting API and SDK mitigates this.
- **Distributions** are packages of upstream components with a vendor's defaults, not forks.

## Maturity by signal

| Signal | API | SDK | OTLP |
|---|---|---|---|
| Traces | Stable | Stable | Stable |
| Metrics | Stable | Stable (some parts still evolving) | Stable |
| Logs | Stable (Bridge API) | Stable | Stable |
| Baggage | Stable | Stable | n/a |
| Profiles | In development / experimental |  |  |

## OTLP

Before OTLP every tool had its own format (Prometheus text, Jaeger Thrift then gRPC, Zipkin Thrift/JSON, OpenCensus protobuf). OTLP unifies them. It is three things at once:

- A **data model**: protobuf messages for traces, metrics and logs.
- A **wire protocol**: how the data is structured and sent.
- A **transport specification**: gRPC or HTTP, with compression, retries and error handling.

Every signal uses the same hierarchy: **Resource** (who emits) → **InstrumentationScope** (which library, name and version) → signal data. The Resource is set when the provider is created and is immutable. Protobuf gives small messages, fast parsing and schema evolution (unknown fields are ignored).

|  | gRPC + protobuf | HTTP + protobuf | HTTP + JSON |
|---|---|---|---|
| Default port | **4317** | **4318** | **4318** |
| Transport | Requires HTTP/2 | HTTP/1.1 or HTTP/2 | HTTP/1.1 or HTTP/2 |
| Routing | One endpoint, gRPC services | `/v1/traces`, `/v1/metrics`, `/v1/logs` | Same paths |
| Content-Type | Handled by gRPC | `application/x-protobuf` | `application/json` |
| Performance | Best | Slightly worse | Worst |
| When | Production, SDK → Collector | When gRPC is blocked (proxies, firewalls) | Debugging, human-readable payloads |

> [!note]
> With HTTP you configure the base URL and the SDK appends `/v1/traces` etc. With gRPC, `host:4317` is enough.

# 3. Span anatomy and context propagation

## Traces and spans

A **trace** is the full journey of one request. A **span** is one unit of work inside it: handling an HTTP request, a query, a call to another service, parsing JSON, consuming a message. Spans form a tree: the **root span** has no parent, every downstream call creates a child, and siblings share a parent. In the *waterfall* view, bar length is duration and nesting shows dependency.

![[otca-diagram-02.svg]]

*Waterfall example: the root spans everything; checkout accounts for almost all the latency and email fails.*

## What a span contains

- **Name**, **kind**, **start and end time** (duration = end − start).
- **SpanContext** and **parent_span_id** (empty for the root).
- **Attributes**, **events**, **links** and **status**.
- The **Resource** of the emitting service and, optionally, a `schema_url`.

## Span names

Short, generic and **low-cardinality**: describe the operation (`processPayment`, `getCart`, `GET /orders/{id}`), not the specific request. Specific data (order ID, user) goes into attributes. Recommended pattern: verb + object.

## SpanContext: what travels between services

| Field | Size | Notes |
|---|---|---|
| `trace_id` | 16 bytes (32 hex) | Same for every span in the trace; if it changes, it's a different trace |
| `span_id` | 8 bytes (16 hex) | Unique within its trace |
| `trace_flags` | 1 byte | `01` = sampled, `00` = not sampled |
| `trace_state` | `key=value` list | Vendor data; 512-character limit; the `ot=` key is reserved for OTel |

## Span kind

| Kind | Direction | Use |
|---|---|---|
| `CLIENT` | Outgoing, synchronous | HTTP/RPC call, database query |
| `SERVER` | Incoming, synchronous | A server handling a request |
| `PRODUCER` | Outgoing, asynchronous | Publishing to Kafka or a queue |
| `CONSUMER` | Incoming, asynchronous | Processing a message from the queue |
| `INTERNAL` | Local | In-process work. **This is the default** |

Each span has a single role. Every outgoing call gets its own CLIENT span (three queries = three spans). With context propagated, the receiver's SERVER span becomes a child of the caller's CLIENT span.

## Span links

Links relate spans that don't have a direct parent-child relationship: asynchronous, parallel or batch work. Examples: map-reduce workers linked to the job that started them, an aggregator processing messages from several traces, independent consumers (email, inventory) of the same order. A link points to a SpanContext and can carry attributes.

| Model | When |
|---|---|
| Parent-child | Synchronous chains: A calls B |
| Links | Async, parallel or independent work with a common trigger |

## Status

- `UNSET`: the default. Implicit success; this is normal.
- `ERROR`: a detected failure (exception, timeout, 5xx…).
- `OK`: success set **explicitly** by the application developer, for example so a handled error doesn't count as a failure. Instrumentation libraries don't set it.

> [!note]
> A spec detail the course simplifies: on a `SERVER` span a 4xx does **not** set ERROR (the client is at fault), but a 5xx does. On a `CLIENT` span both do.

Exceptions are recorded as an **event** named `exception` with `exception.type`, `exception.message` and `exception.stacktrace` (`record_exception()`).

## Events vs attributes

|  | Event | Attribute |
|---|---|---|
| Time | A single timestamp, no duration | None: applies to the whole span |
| For | Something that happened at one moment (exception, retry, cache miss) | Descriptive context (HTTP method, `order_id`, payment type) |
| Example | `span.add_event("retry", {...})` | `span.set_attribute("payment.method", "card")` |

If you need to express an **interval**, don't use an event: create a child span. Events only exist inside a span.

**Attribute rules:** non-null string keys; values are strings, booleans, integers, floats or arrays of these; never `null` (if there's no value, omit the attribute). Set them early so the sampler can use them.

## Resource

Describes where telemetry comes from (service, host, container, pod, cloud). It is configured once when the SDK starts and applies to every span, metric and log.

- Default attributes: `service.name` (if you don't set it, **`unknown_service`**), `telemetry.sdk.name`, `telemetry.sdk.language` and `telemetry.sdk.version`.
- **Resource detectors** add OS, host, process, container, Kubernetes, cloud and region.
- To add your own: `Resource.create({...})` in code, or the variables `OTEL_RESOURCE_ATTRIBUTES=deployment.environment=prod,team=payments` and `OTEL_SERVICE_NAME`.

## Baggage

Small key-value pairs that travel with the request between services, in the W3C `baggage` header. Unlike attributes, baggage **propagates** downstream; attributes stay on their span. **Baggage is not added to telemetry automatically**: to see it on a span you have to read it and copy it as an attribute (or use a `BaggageSpanProcessor`).

- Uses: `correlation_id`, `cart_id`, region, tier, routing.
- Security: it travels in plain-text headers and can be tampered with or leak outside your network. **Never** put PII, credentials or tokens in it.
- Keep it small (it's sent with every request) and use standardized names.

## Context propagation

For spans from different services to form a single trace, the SpanContext (and baggage) has to travel with the request. OTel does this with **propagators**:

- **Carrier**: where the context travels (HTTP headers, gRPC metadata, message attributes).
- **inject**: the sender writes the current context into the carrier. **extract**: the receiver reads it and restores it as the active context.
- **TextMapPropagator**: the interface that does inject/extract, using setter and getter functions.
- **Global propagator**: app-level configuration of which formats are used (`OTEL_PROPAGATORS`; default `tracecontext,baggage`).

| Propagator | Headers | Notes |
|---|---|---|
| W3C Trace Context | `traceparent`, `tracestate` | OTel default |
| W3C Baggage | `baggage` | Default alongside Trace Context |
| B3 | `X-B3-TraceId`, `X-B3-SpanId`, `X-B3-Sampled` | Zipkin and older systems |
| Composite | Several | Combines formats: mixed environments or migrations |
| Custom | Anything | Implement `TextMapPropagator` |

`traceparent` format: `00-<trace_id 32 hex>-<parent span_id 16 hex>-<flags>`, for example `00-4bf92f35…4736-00f067aa0ba902b7-01`.

# 4. Instrumentation

## The three approaches

| Approach | Code changes | Coverage | Business logic | When |
|---|---|---|---|---|
| Zero-code (auto) | None; an agent or wrapper | Broad: HTTP, DB, messaging | Low | Quick visibility, apps you can't modify |
| Library-based | Minimal (instrumentation packages) | Specific libraries | Medium | A dependency needs more visibility |
| Manual (code-based) | Yes | Whatever you decide | High | Critical flows, business attributes, custom async |

Recommended strategy: start with zero-code for broad coverage, review the traces looking for gaps (missing business spans, odd hierarchies, broken async flows) and add manual or library instrumentation where needed. All three combine without problems.

> [!note]
> In quizzes, "library-based" sometimes means "importing the OTel libraries in your code" (that is, code-based). Check the other options to see which meaning they intend.

## Manual instrumentation: the flow

1. Configure a `TracerProvider` (SDK) with its Resource, processors and exporters.
2. Get a `Tracer`: `trace.get_tracer(__name__)`. The name is the *instrumentation scope*.
3. Create spans with `start_as_current_span` (Python), `spanBuilder().startSpan()` (Java) or `startSpan()` (JS).
4. The sampler decides whether the span is recorded.
5. The `SpanProcessor` receives the span when it ends, applies `SpanLimits` and hands it to the exporter.
6. The exporter sends it to the Collector or the backend.

Key methods: `set_attribute`, `add_event`, `record_exception`, `set_status`. Always end spans (preferably with `with`). With no `TracerProvider` configured everything is no-op and raises no errors. Configure a single provider per application.

```python
provider = TracerProvider(resource=Resource.create({"service.name": "shop"}))
provider.add_span_processor(BatchSpanProcessor(OTLPSpanExporter()))
trace.set_tracer_provider(provider)

tracer = trace.get_tracer(__name__)
with tracer.start_as_current_span("processPayment") as span:
    span.set_attribute("payment.method", "card")
```

## Span processors and exporters

- **SimpleSpanProcessor**: exports each span as soon as it ends. For debugging.
- **BatchSpanProcessor**: buffers and exports in batches. **The one for production.**
- Exporters: Console (debugging), OTLP (Collector or compatible backends), Zipkin and others.
- Exporter methods: `export(spans)`, `shutdown()` (call it when the app stops so nothing is lost) and `force_flush()` (empty the buffer immediately).

## Sampling in the SDK (head sampling)

The decision is made when the span is created and travels in the `sampled` flag of `traceparent`. That way the whole trace follows the same decision and no partial traces are stored.

| Sampler | Behavior |
|---|---|
| `AlwaysOn` | Samples everything (development) |
| `AlwaysOff` | Samples nothing; keeps instrumentation but no data |
| `TraceIdRatioBased(0.2)` | A fixed share of traces, computed from the trace_id |
| `ParentBased(root=…)` | Follows the parent's decision; with no parent, uses the `root` sampler |

Typical production combination: `ParentBased(root=TraceIdRatioBased(0.1))`, which as environment variables is `OTEL_TRACES_SAMPLER=parentbased_traceidratio` with `OTEL_TRACES_SAMPLER_ARG=0.1`. Only `ParentBased` accepts another sampler inside it. If you need to decide based on the complete trace (errors, latency), that's tail sampling in the Collector.

## Instrumentation libraries

Packages named `opentelemetry-instrumentation-<library>` that hook into third-party libraries (requests, Flask, Django, FastAPI, SQLAlchemy, Kafka, Celery, boto3…). For example, the `requests` one wraps the call: it opens a CLIENT span, runs the original request and closes the span with semantic attributes.

| They do | They don't |
|---|---|
| Hook into the library and create spans or metrics | Configure the SDK, processors or exporters |
| Fill attributes according to the semantic conventions | Depend on a specific SDK implementation |
| Use only the API | Export or store data |

Split of responsibilities: the library decides **what** is collected; the application decides **how** it is processed and exported.

## Zero-code: mechanisms

- **Monkey patching** (Python, JS): library functions are replaced or wrapped at runtime.
- **Bytecode manipulation** (Java, .NET): the agent rewrites classes as they load.
- **eBPF**: observes kernel events (network, syscalls). Language-independent and non-intrusive.

Configured through environment variables (most common), system properties (Java/.NET) or startup arguments.

### Java

`java -javaagent:opentelemetry-javaagent.jar -jar app.jar`. Configure it with `-Dotel.*` or `OTEL_*` (better in containers). The agent can be extended: custom processors, PII masking, disabling instrumentations or instrumenting internal libraries.

| Variable | Purpose |
|---|---|
| `OTEL_SERVICE_NAME` | Service name |
| `OTEL_RESOURCE_ATTRIBUTES` | Extra Resource attributes |
| `OTEL_TRACES_EXPORTER` | `otlp`, `zipkin`, `console`… |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | Where to send data |
| `OTEL_PROPAGATORS` | `tracecontext,baggage`… |
| `OTEL_TRACES_SAMPLER` / `_ARG` | Sampler and its argument |
| `OTEL_SDK_DISABLED` | Turns the SDK off (useful in tests) |

### Python

1. `pip install opentelemetry-distro opentelemetry-exporter-otlp`
2. `opentelemetry-bootstrap -a install`: detects installed libraries and installs their instrumentations.
3. `opentelemetry-instrument python app.py`: a `sitecustomize` configures the SDK from environment variables and calls `.instrument()` on every installed instrumentation.

Python-specific options: `OTEL_PYTHON_DISABLED_INSTRUMENTATIONS` (to exclude instrumentations; there is no "only these" list) and `OTEL_PYTHON_LOG_CORRELATION=true`. To control what gets instrumented, install only the packages you want.

# 5. Metrics data model

## What metrics are for

They answer questions like whether the app is healthy right now, whether traffic is growing, whether something is degrading, or whether you need to scale. They are cheap and fast for detection and alerting, but rarely explain the cause: metrics tell you *what*, traces and logs tell you *why*. A metric has a **name** (`container.cpu.usage`), timestamped **values**, a **unit** and **dimensions** (attributes).

## The four structural properties

| Property | Meaning | Rule of thumb |
|---|---|---|
| Monotonicity | Whether the value can only go up (monotonic) or can also go down | Rates (`rate()`) only make sense on monotonic sums; a drop signals a reset |
| Temporality | **Cumulative** (total since start) or **delta** (what happened in the interval) | Use what the backend expects; Prometheus works with cumulative |
| Aggregation | How measurements are summarized: Sum, LastValue (gauge), Histogram | Histograms for latency; sums for counts |
| Dimensions | Attributes (labels in Prometheus) on each measurement | Watch cardinality: no unbounded IDs |

**Temporality, with a bank account:** you deposit 1000, get paid 500 and spend 200. The deltas are +1000, +500 and −200; the running totals are 1000, 1500 and 1300. Temporality changes *how* you compute a rate, not *whether* you can. With cumulative values you must detect resets (if the current value is lower than the previous one, the process restarted).

## Histograms and percentiles

![[otca-diagram-03.svg]]

*A histogram stores counts per bucket, not values. All you know is that p95 is between 0.5 and 1 s.*

- It stores per-bucket counts, the sum and the total count (and optionally min/max). That lets you see the shape of the distribution and compute percentiles (p50, p90, p95, p99) without storing every measurement.
- Buckets can be **explicit** (fixed boundaries you define) or **exponential** (they adapt automatically to wide ranges, with higher resolution).
- A percentile from an explicit-bucket histogram is an **estimate**: it interpolates assuming values are spread evenly within the bucket. What is exact is the share of values below each boundary. That's why you should put a boundary exactly at your SLO threshold.

## Dimensions and cardinality

Dimensions let you **filter** (`service.name="cart"`), **group** (one series per service) and **aggregate** (`avg … by server.address`). Every unique combination of attributes is a new time series.

| Cardinality | Examples | Use |
|---|---|---|
| Low | HTTP status, region, tier | General monitoring and alerts; cheap |
| Medium | Service name, instance type, model | Segmentation and trends |
| High | `user_id`, `request_id`, license plate | One-off investigations; very carefully in metrics (better in traces or logs) |

## From instrument to backend: three models

| Model | Where | Contents |
|---|---|---|
| **Event model** | Your code | Raw measurements from the instruments |
| **OTLP stream model** | In transit (SDK → Collector → backend) | Already-aggregated data: Sum, Gauge, Histogram, ExponentialHistogram, Summary (compatibility only), with temporality, monotonicity and start/end timestamps |
| **Timeseries model** | The backend | Series per combination of name and attributes, ready to query |

Transformations the model allows, in the SDK (with Views), the Collector or the backend:

- **Spatial re-aggregation**: drop attributes and merge series, for example by region instead of by instance. Reduces cardinality.
- **Temporal re-aggregation**: merge points into longer intervals, for example six 10 s points into one 1 min point (downsampling).
- **Delta ↔ cumulative** conversion, unit normalization (ns → s) and enrichment with attributes.

The Collector is usually the best place to apply these policies centrally. It also translates to and from Prometheus (remote write) and StatsD.

## Instruments

Each instrument has a name, kind, unit, description and, optionally, hints such as bucket boundaries.

| Instrument | Monotonic | Default aggregation | Examples |
|---|---|---|---|
| Counter / ObservableCounter | Yes | Sum | Requests, bytes sent |
| UpDownCounter / ObservableUpDownCounter | No | Sum | Active connections, queue size |
| Histogram | n/a | Histogram | Latency, sizes |
| Gauge / ObservableGauge | n/a | LastValue | CPU, memory, temperature |

**Synchronous** instruments are called by your code when something happens (`add()`, `record()`) and know the active context, so they can produce exemplars. **Asynchronous** (Observable) instruments register a callback that the SDK runs at each collection. Use them to read values that already exist, for example by polling the OS. An ObservableCounter reports the **total**, not the increment. "Asynchronous" here has nothing to do with `asyncio`.

# 6. Recording measurements (Metrics API and SDK)

![[otca-diagram-04.svg]]

- **MeterProvider**: the entry point. Holds the Resource, Views, readers and exporters.
- **Meter**: an instrumentation scope (`name` required; `version` and `schema_url` optional). Creates the instruments.
- **View**: changes how an instrument is aggregated without touching code. It can rename it, change the aggregation or buckets, keep only some attributes (reducing cardinality) or drop it.
- **MetricReader**: decides when to collect and does the aggregation and temporality. The exporter only serializes and sends. You can have several readers, each with its own exporter.

| Mode | Who initiates | Example | `ForceFlush` |
|---|---|---|---|
| Push | The reader, periodically (`PeriodicExportingMetricReader`) | OTLP to the Collector | Collects and exports immediately |
| Pull | An external scraper | Prometheus exporter with `/metrics` | **No effect** |

**Shutdown**: called exactly once; it cascades to readers and exporters and reports success, failure or timeout. After that the provider returns no-op meters.

## Exemplars

An exemplar is a sample raw measurement stored alongside an aggregated data point. It carries the value, the timestamp, the filtered attributes (the ones dropped during aggregation) and the `trace_id`/`span_id` of the active span. It lets you jump from a spike on a chart to the exact trace that caused it (in Grafana they show up as dots on the chart).

- They only come from **synchronous** instruments, because they need an active span at measurement time. They work for counters as well as histograms.
- Filter: `OTEL_METRICS_EXEMPLAR_FILTER` = `trace_based` (default: only measurements inside sampled spans), `always_on` or `always_off`.
- In Prometheus they need `--enable-feature=exemplar-storage` and the OpenMetrics format.

# 7. Logs

## Kinds of logs

Logs can be **structured** (fixed schema, usually JSON, machine-readable), **unstructured** (free text, stack traces) or **semi-structured** (key=value). Prefer structured logs with level, source and context (`trace_id`, `span_id`).

## How OTel fits with your logging library

OTel doesn't replace Log4j, Logback, `logging`, Serilog, Winston or Zap: it hooks into them.

1. The app writes logs with its usual library.
2. An OTel **appender or bridge** captures each log and checks the current context.
3. If a span is active, it adds the `trace_id` and `span_id`.
4. The log goes through the `LoggerProvider`, the `LogRecordProcessor` and the `LogRecordExporter`, and leaves over OTLP.

The alternative without an appender is to write to a file or stdout and have the Collector pick it up with the `filelog` or `syslog` receiver. In that case, to correlate with traces, the `trace_id` must be in the log text itself.

## Components

| Component | Role |
|---|---|
| Log appender / bridge | Connects the existing logging library to the SDK; the app doesn't change its code |
| LoggerProvider | Logger factory, one per application; holds the Resource and exporters |
| Logger | Emits LogRecords for an instrumentation scope |
| LogRecordProcessor | Simple (synchronous, for debugging), Batching (production) or Custom (filter, enrich, route) |
| LogRecordExporter | `Export(batch)` and `Shutdown()`; batching, retries and backpressure |

## Logs API (Bridge API)

Intended mainly for people writing appenders, not for logging by hand from the app. `LoggerProvider.getLogger(name)` (the name is required) → `logger.emit(LogRecord)`. `isEnabled` lets you skip building expensive logs when the level is disabled. The APIs are thread-safe.

## Logs SDK

It works like the tracing SDK: the provider holds the Resource, and if you change processors or exporters on the provider, existing loggers use the new configuration. When the app stops: `ForceFlush()` then `Shutdown()`. The exporter must be thread-safe, respect timeouts and return a clear result so the processor can decide whether to retry.

## LogRecord data model

| Field | Meaning |
|---|---|
| `timestamp` | When it happened at the source |
| `observed_timestamp` | When OTel collected it (Collector or appender) |
| `severity_text` / `severity_number` | Level as text and as a number |
| `body` | The message or payload |
| `trace_id`, `span_id`, `trace_flags` | Correlation with the trace (optional) |
| `attributes`, `resource`, scope | Log context, who emits it and which library produced it |

| TRACE | DEBUG | INFO | WARN | ERROR | FATAL |
|---|---|---|---|---|---|
| 1–4 | 5–8 | 9–12 | 13–16 | 17–20 | 21–24 |

Each level has 4 values so systems with intermediate grades can be mapped. If the destination has no equivalent level, the closest one is used. Timestamps are 64-bit integers in nanoseconds since the epoch. Structured **event**-style logs are recommended; the current spec adds an `event_name` field for them.

## Best practices

- Structured logs with a consistent schema; always include level, service and host.
- Add `trace_id`/`span_id` so you can correlate.
- Use consistent attribute names (`order.id`, `user.id`) instead of concatenating them into the message.
- Don't log PII or credentials; filter or redact at the source or in the Collector.
- Centralize parsing, enrichment and export in the Collector.

In Python: add `LoggingHandler(logger_provider=provider)` to the root logger, or with zero-code set `OTEL_PYTHON_LOGGING_AUTO_INSTRUMENTATION_ENABLED=true`. The `opentelemetry-instrumentation-logging` package only injects the IDs into the text format; it doesn't export.

# 8. Collector foundations

The Collector is a standalone, vendor-neutral process that **receives, processes and exports** telemetry. Benefits: it decouples apps from backends, centralizes configuration (one YAML), scales horizontally and enables data governance (filter, redact PII or enrich before data leaves your network).

![[otca-diagram-05.svg]]

## Distributions

|  | Core | Contrib |
|---|---|---|
| Contents | Minimal and stable: `otlp`, `batch`, `memory_limiter`, `debug`, basic extensions | Hundreds of community and vendor integrations |
| Pace of change | Slow, conservative | Fast, more experimental |
| When | Critical production, audited environments | Trying integrations, prototypes |

**OCB (OpenTelemetry Collector Builder)**: from a YAML manifest listing the components you want, it builds a custom binary or image. The result is smaller, has less attack surface and is easier to review for security. Vendor distributions add support and SLAs. **Feature gates** let you enable experimental behavior in a controlled way.

## Anatomy and config.yaml

| Section | Role | Examples |
|---|---|---|
| `receivers` | Ingest data and translate it into the internal model | `otlp`, `prometheus`, `filelog` |
| `processors` | Transform, filter, sample, batch (optional; order matters) | `batch`, `attributes`, `tail_sampling` |
| `exporters` | Send to backends | `otlp`, `prometheusremotewrite`, `debug` |
| `connectors` | Exporter of one pipeline and receiver of another | `spanmetrics`, `count` |
| `extensions` | Functions outside the pipelines | `health_check`, `zpages`, `pprof` |
| `service` | Enables extensions and wires the pipelines (**required**) | `service.pipelines.traces` |

- The minimal valid config is one receiver and one exporter wired in a pipeline. **Anything not referenced in `service` is not activated.**
- You can have several instances of the same type with `type/name` (`otlp/ingest2`) and several pipelines per signal (`traces`, `traces/2`).
- **Fan-out** is one pipeline with several exporters. **Fan-in** is several receivers feeding one pipeline.
- Config can be parameterized with `${env:VAR}` and `${file:path}` and split across files.
- Validate before starting with `otelcol validate --config=config.yaml`.
- The Collector's own telemetry is configured in `service.telemetry`: logs and metrics, with `metrics.readers` in pull mode (Prometheus) or periodic mode (OTLP push).

```yaml
receivers:
  otlp:
    protocols: { grpc: {}, http: {} }
processors:
  memory_limiter: { check_interval: 1s, limit_percentage: 80 }
  batch: {}
exporters:
  otlp: { endpoint: backend:4317 }
  debug: { verbosity: basic }
extensions:
  health_check: {}
service:
  extensions: [health_check]
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [otlp, debug]
```

# 9. Collector components

## The Collector as an agent

| Deployment | Use |
|---|---|
| VM agent | One per host (for example, a systemd service); adds host metadata |
| DaemonSet | One per Kubernetes node; collects from all pods on the node, the kubelet and the host |
| Sidecar | One per pod; the app sends to `localhost:4317`. Maximum isolation, no network hops |

Layered architecture: local agents enrich with host or pod metadata, batch and retry. Team-level Collectors can come next. Finally, a central gateway handles TLS, cross-team routing, PII redaction and the final export. Enrich as close to the source as possible and centralize sensitive policies.

## Key receivers

| Receiver | Purpose | Notes |
|---|---|---|
| `otlp` | Main entry point for instrumented apps | 4317 gRPC, 4318 HTTP; all three signals |
| `prometheus` | Scrape Prometheus endpoints and convert to OTLP | Prometheus config and service discovery |
| `hostmetrics` | Host CPU, memory, disk and network | As an agent or DaemonSet |
| `kubeletstats` | Pod and container metrics from the kubelet | DaemonSet (each talks to its own kubelet) |
| `k8s_cluster` | Cluster metrics and node conditions | Deployment with **1 replica** (avoids duplicates) |
| `k8sobjects` | Kubernetes events and objects as logs | Deployment with 1 replica |
| `filelog` | Read log files; handles rotation and remembers its position | Container logs on the node |
| `journald`, `syslog` | System and network-device logs | syslog on 514 |
| `jaeger`, `zipkin`, `opencensus` | Migrating from older formats | Zipkin on 9411 |

## Processors

| Processor | Purpose |
|---|---|
| `memory_limiter` | Prevent OOM by refusing data above the limit. **Always in production, first in the chain** |
| `batch` | Group data before export (end of the chain) |
| `resourcedetection` | Detect host, cloud and container metadata |
| `resource` | Add or change Resource attributes |
| `k8sattributes` | Add pod, namespace, deployment, labels… |
| `attributes` | insert/update/upsert/delete/hash attributes (for example PII) |
| `transform` | Conditional OTTL rules |
| `filter` | Drop telemetry matching OTTL conditions |
| `probabilistic_sampler` | Percentage-based, deterministic sampling (hash of trace_id) |
| `tail_sampling` | Decide with the complete trace (errors, latency) |
| `deltatocumulative` / `cumulativetodelta` | Change metric temporality |
| `span` | Rename spans or set the status |

Order recommended by the course: `memory_limiter` → normalize and enrich (`attributes`, `transform`, `resourcedetection`, `k8sattributes`) → `batch`.

## Exporters and resiliency

The `otlp` (gRPC) and `otlphttp` exporters are the default choice: portable and standard. Key options:

| Option | Purpose |
|---|---|
| `endpoint`, `tls` (`insecure` only for testing) | Destination and encryption or mTLS |
| `headers` or an authenticator extension | Authentication (keep secrets in env vars or a secret store) |
| `timeout` | Maximum time per attempt |
| `retry_on_failure` | Retries with exponential backoff; `max_elapsed_time` caps the total |
| `sending_queue` | Buffer while the backend is unavailable; with the `file_storage` extension it persists to disk and survives restarts |
| `compression: gzip` | Save bandwidth |

Contrib exporters: `prometheus` (exposes a scrape endpoint), `prometheusremotewrite` (push to Cortex, Thanos or Mimir), `zipkin`, `kafka` (durable streaming), `file` (debugging) and vendor-specific ones. `debug` prints data with a verbosity of basic, normal or detailed.

### Load balancing exporter

Spreads telemetry across several downstream Collectors in a "sticky" way: everything with the same routing key always goes to the same destination. It is essential for **tail sampling at scale**, because all spans of a trace must end up on the same Collector. It creates one OTLP sub-exporter per destination, each with its own queue and retries.

- Routing keys: `traceID` (default for traces), `service` (default for metrics), `resource`, `metric` and `streamID` (metrics only), `attributes`.
- Resolvers: `static` (fixed list), `dns` and `k8s` (headless service). Destinations should have a stable identity, for example a StatefulSet.

### Patterns

- **Fan-out**: the same data to several destinations; duplicate early so they all get the same thing.
- **Routing by attribute**: by environment, by service, etc.
- **Forwarding between pipelines** or between Collectors.
- Always use `batch` together with queueing and retries, validate changes with a canary pipeline and the `debug` exporter, and watch internal metrics for errors.

## Connectors

They act as the exporter of one pipeline and the receiver of another. Use them to generate new telemetry or route it without touching the apps.

| Connector | What it does |
|---|---|
| `spanmetrics` | RED metrics (rate, errors, duration) from spans, with histograms and exemplars |
| `servicegraph` | Service dependency metrics (topology map) |
| `routing` | Routes to pipelines with OTTL conditions (`default_pipelines`, `match_once`) |
| `count` | Counts spans, events or logs that match a condition |
| `sum` | Sums a numeric attribute (for example, order totals) |
| `forward` | Joins or splits pipelines without transforming |
| `exceptions` | Extracts exceptions from traces as logs or metrics |

A connector is declared under `connectors:` and used as an exporter in one pipeline (for example `traces`) and as a receiver in another (for example `metrics`).

# 10. Transforming telemetry in the pipeline

## OTTL (OpenTelemetry Transformation Language)

A statement language the Collector evaluates on every telemetry item. It works with traces, metrics and logs. It is used by the `transform` and `filter` processors, the `routing` connector and some `tail_sampling` policies.

- Syntax: `function(arguments) where condition`. If the `where` condition is false, the statement is skipped for that item.
- Contexts (what each statement operates on): `resource`, `scope`, `span`, `spanevent`, `metric`, `datapoint` and `log`.
- Common functions: `set`, `delete_key`, `keep_keys`, `replace_pattern`, `SHA256` (hash PII), `truncate_all`, `ExtractPatterns`, `Concat`.
- Uses: masking PII, renaming or normalizing attributes, extracting fields (for example the user ID from a URL), dropping noise.

| `error_mode` | Behavior |
|---|---|
| `ignore` | Logs the error and continues. **Recommended for production** |
| `silent` | Continues without logging (hides problems) |
| `propagate` | Returns the error and stops processing that data |

```yaml
processors:
  transform:
    error_mode: ignore
    trace_statements:
      - context: span
        statements:
          - set(attributes["env"], "prod") where resource.attributes["k8s.namespace.name"] == "prod"
          - set(attributes["user.email"], SHA256(attributes["user.email"]))
  filter:
    error_mode: ignore
    traces:
      span:
        - resource.attributes["service.name"] == "healthcheck"
```

> [!note]
> In the `filter` processor, if the condition matches, the item is **dropped**.

## Sampling strategies

![[otca-diagram-06.svg]]

*Tail sampling at scale: all spans of a trace must reach the same Collector.*

|  | Head sampling | Tail sampling |
|---|---|---|
| Where | SDK (sampler) or Collector (`probabilistic_sampler`) | Collector (`tail_sampling`) |
| When it decides | At the start, without knowing how the trace ends | With the complete trace, after waiting `decision_wait` |
| Pros | Cheap, fast, scales well, reduces volume early | Can keep every trace with errors or high latency |
| Cons | May drop exactly the important traces | Memory (buffering), latency, and all spans of a trace must reach the same Collector |

**`probabilistic_sampler`** hashes the trace_id with a seed (`hash_seed`) and compares it with `sampling_percentage`. It is deterministic: with the same seed on every Collector, all of them make the same decision for a given trace.

**`tail_sampling` policies**: `status_code` (ERROR), `latency` (`threshold_ms`), `span_count` (`min_spans`), `string_attribute` (for example `tenant=vip`), `rate_limiting` (spans per second), `probabilistic` (a percentage of the rest), `ottl_condition`, and combinations with `and` or `composite`. If any policy says "keep", the trace is kept. Spans arriving after the trace has been decided are lost.

Architecture at scale: a first tier of stateless Collectors with the `load_balancing` exporter (routing by traceID), and a second tier with stable identity (StatefulSet) that does the tail sampling. The usual hybrid pattern is low-rate head sampling to cut baseline volume plus tail sampling to keep errors and slow traces.

## Telemetry schemas

Semantic conventions evolve (for example, `http.status_code` became `http.response.status_code`). A **schema** is a versioned file that records those changes, and each piece of data carries a `schema_url` (`https://opentelemetry.io/schemas/1.2.0`) saying which version it uses. A Collector or backend can then translate between versions so dashboards don't break.

- Semantic versioning: a **patch** change means nothing changes for consumers; a **minor** adds backward-compatible items; a **major** may break things.
- Compatibility: a 1.3 consumer can read 1.1 data, but a 1.1 consumer cannot read 1.3 data. Different majors are not compatible.
- File sections: `all` (global; only `rename_attributes` allowed, and it **runs first**), `resources`, `spans`, `span_events`, `metrics` (`rename_metrics`, `rename_attributes` with `apply_to_metrics`, `split`) and `logs`.
- The selectors `apply_to_spans`, `apply_to_events` and `apply_to_metrics` limit the scope of a change. If omitted, it applies to everything.
- Migrations go through every intermediate version in order (1.0 → 1.1 → 1.2), skipping none. The `schema_url` names the highest version.
- Keep transformations minimal and rename-based.

# 11. OpenTelemetry in Kubernetes

In Kubernetes, telemetry needs the pod's context: name and UID, namespace, labels and annotations, the workload (Deployment, ReplicaSet, StatefulSet) and the node. This enrichment is done by the `k8sattributes` processor (which needs RBAC to read the API) or by resource detectors.

## Collector deployment modes

| Mode | Best for | Characteristics |
|---|---|---|
| DaemonSet (agent) | Node telemetry: kubelet, host, container logs | One per node, local buffering, node metadata; forwards to the gateway or straight to the backend |
| Deployment (gateway) | Central processing: sampling, redaction, aggregation | Stateless, several replicas behind a Service |
| Sidecar | Per-application isolation, no network hops | In the same pod as the app; its own config |
| StatefulSet | Sharded Prometheus scraping, tail sampling | Stable identity and DNS, a PVC to persist the queue |

The **TargetAllocator** (an Operator feature) splits Prometheus scrape targets across several Collectors so nothing is scraped twice.

## Key receivers in Kubernetes

| Component | Provides | Where |
|---|---|---|
| `k8sattributes` (processor) | Pod and node metadata | Any Collector, with RBAC |
| `kubeletstats` | Node, pod and container CPU and memory from the kubelet (port 10250) | DaemonSet |
| `filelog` | Logs from `/var/log/containers/*.log` (or `/var/log/pods`) | DaemonSet with hostPath |
| `k8s_cluster`, `k8sobjects` | Cluster metrics, object state and events | Deployment with 1 replica |
| `hostmetrics` | Node OS metrics | DaemonSet |
| `prometheus` | Scrape `/metrics` endpoints | Depends on topology; beware of duplicates |
| `otlp` | Telemetry from the apps | Gateway or agent |

## OpenTelemetry Operator

A Kubernetes operator that manages Collectors and auto-instrumentation through CRDs. Install it with Helm (recommended) or manifests; it needs **cert-manager** (or equivalent) for its webhook certificates. It creates Services and ports, manages endpoints, handles rollouts and upgrades, and injects instrumentation.

- **`OpenTelemetryCollector`** (CR): declares a Collector with its `mode` (`deployment`, `daemonset`, `sidecar` or `statefulset`) and configuration.
- **`Instrumentation`** (CR): the shared auto-instrumentation config. Each field becomes an environment variable: `spec.exporter.endpoint` → `OTEL_EXPORTER_OTLP_ENDPOINT`, `spec.propagators` → `OTEL_PROPAGATORS`, `spec.sampler` → `OTEL_TRACES_SAMPLER` and its `_ARG`.

Enable it on a pod or namespace with an annotation, for example `instrumentation.opentelemetry.io/inject-java` (or `-python`, `-nodejs`, `-dotnet`, `-go`):

| Value | Meaning |
|---|---|
| `"true"` | Use the default Instrumentation in the same namespace |
| `"name"` | Use a specific Instrumentation in the same namespace |
| `"ns/name"` | Use an Instrumentation from another namespace |
| `"false"` | Opt this workload out |

`sidecar.opentelemetry.io/inject: "true"` injects a sidecar Collector. Be careful annotating whole namespaces: you may inject agents into pods where you didn't want them.

# 12. Collector debugging, operations and scaling

## Diagnostic tools

| Tool | Default port | Purpose |
|---|---|---|
| Internal metrics | **8888** | The Collector's own Prometheus metrics: throughput, refusals, queues, CPU and memory |
| `health_check` | **13133** | Heartbeat for Kubernetes liveness and readiness probes and load balancers |
| `zpages` | **55679** | In-browser pages: `/debug/servicez`, `pipelinez`, `extensionz`, `featurez`, `tracez` |
| `pprof` | **1777** | Go profiling: CPU, heap/allocs, goroutines, mutex, block |

## Internal logs and the debug exporter

- The Collector's own log level is `service.telemetry.logs.level` (DEBUG, INFO by default, WARN or ERROR), plus `encoding: json`, `sampling` and `output_paths`. By default it writes to stderr. Read it with `journalctl -u otelcol-contrib`, `docker logs` or `kubectl logs`.
- The `debug` exporter prints the data passing through to the log, with a `verbosity` of `basic` (summaries and counts), `normal` (adds resource and scope) or `detailed` (full contents). It supports `sampling_initial` and `sampling_thereafter`. Use it to check data shape, trace_id propagation and what processors are doing.
- Don't confuse the Collector's log level with the debug exporter's verbosity: they are separate settings.

## Internal metrics to know

Naming pattern: `otelcol_<component>_<action>_<signal>` (with `_total` on counters).

| Metric | Indicates |
|---|---|
| `otelcol_receiver_accepted_spans` | What comes in |
| `otelcol_receiver_refused_spans` | What is refused on entry (limits, malformed data) |
| `otelcol_processor_refused_spans` | What a processor refuses, typically `memory_limiter` under pressure |
| `otelcol_exporter_sent_spans` / `send_failed_spans` | Successful and failed sends (network or backend problems) |
| `otelcol_exporter_queue_size` / `queue_capacity` | Sending queue; if it grows, there is backpressure |
| `otelcol_exporter_enqueue_failed_spans` | Queue full: data is being lost |

## Scaling

| Component type | How to scale |
|---|---|
| Stateless (receive OTLP and forward) | Horizontally, several replicas behind a load balancer |
| Scrapers (Prometheus) | Split targets across Collectors with the TargetAllocator (via `http_sd_config`) to avoid duplicate metrics |
| Stateful (tail sampling, spanmetrics) | Consistent routing by trace_id (`load_balancing` exporter) to stable replicas |

| Symptom | Action |
|---|---|
| Processor refusals or memory pressure | More CPU and memory or more replicas; tune `memory_limiter` |
| Exporter queue growing (around 70%) | More queue capacity, tune timeouts and retries, or scale |
| High exporter latency | The bottleneck is the backend or network: **adding Collector replicas won't fix it** |

First identify which signal is overloaded. If pipelines are separated by signal, scale only the one that needs it.

# 13. SLIs and SLOs with OpenTelemetry

- **SLI**: an indicator measured from the user's point of view. Typical types are availability, latency, errors, throughput and saturation.
- **SLO**: the target for an SLI over a time window, for example 99% of requests under 200 ms over 30 days. The **SLA** is the contractual commitment.
- Typical formulas: success rate = good requests / total; latency compliance = requests under the threshold / total; error rate = errors / total.
- The process: define what a good experience looks like, pick the SLI that measures it, set the SLO and alert on trends before breaching the SLA.
- CPU is useful for diagnosis but is a poor SLI, because it doesn't directly reflect what users experience.

| Piece | Responsibility |
|---|---|
| App / SDK | Produce accurate telemetry (latency histograms, span status) |
| Collector | Optional: transform, pre-aggregate, derive metrics from spans (`spanmetrics`) or logs (`count`) |
| Backend | **Evaluates the SLO**: windows, error budget, dashboards and alerts. OTel doesn't do this |

> [!note]
> Watch out for sampling: if you compute SLIs from sampled traces, the numbers can be biased. For SLIs prefer metrics (which aren't sampled) or spanmetrics generated before sampling.

# Exam cheat sheet

| Topic | Remember |
|---|---|
| Ports | OTLP gRPC 4317 · OTLP HTTP 4318 · internal metrics 8888 · health_check 13133 · zpages 55679 · pprof 1777 · Zipkin 9411 |
| IDs | trace_id 16 bytes (32 hex) · span_id 8 bytes (16 hex) · flags 01 = sampled |
| Span kind | Default INTERNAL. CLIENT/SERVER are synchronous; PRODUCER/CONSUMER asynchronous |
| Status | UNSET by default · ERROR on failures · OK only when set explicitly by the app developer |
| Events vs attributes | An event has one timestamp and no duration; an attribute describes the whole span |
| Default Resource | `service.name=unknown_service` + `telemetry.sdk.*` |
| Samplers | Only ParentBased wraps another sampler. Head = SDK or probabilistic; tail = Collector |
| API without SDK | No-op, no errors |
| Libraries | Depend only on the API; never configure exporters |
| Instruments | Counter (monotonic), UpDownCounter, Histogram, Gauge; Observable ones use a callback and report the total |
| Exemplars | Only from synchronous instruments; default filter trace_based |
| Severity | TRACE 1–4 · DEBUG 5–8 · INFO 9–12 · WARN 13–16 · ERROR 17–20 · FATAL 21–24 |
| Logs | `timestamp` = when it happened; `observed_timestamp` = when OTel saw it |
| Collector | Anything not in `service.pipelines` is inactive · `memory_limiter` first, `batch` last |
| Kubernetes | kubeletstats/hostmetrics/filelog as a DaemonSet · k8s_cluster/k8sobjects with 1 replica · tail sampling on a StatefulSet |
| Schemas | `all` runs first and only allows renames · migrate version by version |


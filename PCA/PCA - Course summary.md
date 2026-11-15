---
tags: [pca, prometheus, summary]
---

# Prometheus Certified Associate (PCA)

Summary of the KodeKloud prep course, with notes on the details that tend to show up in the exam.

1. Observability fundamentals|Prometheus fundamentals|PromQL|Dashboarding and visualization|Application instrumentation|Service discovery and relabeling|Pushgateway|Alerting and Alertmanager|Monitoring Kubernetes|Scaling and long-term storage|Exam cheat sheet

My own notes based on the course's public lesson notes (notes.kodekloud.com). Diagrams made for this summary.

# 1. Observability fundamentals

Observability is the ability to understand a system's internal state from the data it emits, so you can answer new questions such as why error rates are rising, where latency is introduced, or which service is timing out.

| Pillar | Provides | Best for |
|---|---|---|
| Logs | Text event records with context | Debugging a specific error, audit trails |
| Traces | End-to-end request flow with timing (spans with start, duration, parent ID, tags) | Latency hotspots, cross-service flows |
| Metrics | Numeric time series: name, labels, value, timestamp | Alerting, dashboards, capacity planning |

Prometheus is a **metrics** system. It does not handle logs or traces.

## SLI, SLO, SLA

| Concept | Meaning | Example |
|---|---|---|
| **SLI** | A measurable indicator of service quality, ideally user-centric (latency, error rate, availability, throughput) | p95 latency = 90 ms |
| **SLO** | The target for an SLI | p95 latency < 100 ms; 99.9% uptime over 30 days |
| **SLA** | A contract with customers built on SLOs, with penalties if broken | 99.9% uptime or service credits |

Prefer user-facing SLIs (latency, errors) over resource metrics such as CPU, which do not reflect user impact. The **error budget** (100% − SLO) is what lets you trade off releases against reliability work.

# 2. Prometheus fundamentals

Prometheus is an open-source monitoring system, originally built at SoundCloud, written in Go, and a **graduated CNCF project** (joined in 2016). It pulls metrics over HTTP, stores them in its own TSDB, queries them with PromQL and evaluates alert rules. It is designed for numeric time series: system, service and application metrics. It is not meant for logs, events or 100%-accurate billing data.

## Architecture

![[pca-diagram-01.svg]]

*Prometheus pulls from targets (arrows point at what it scrapes), pushes alerts to Alertmanager and serves queries.*

| Component | Role |
|---|---|
| Prometheus server | Retrieval worker (scrapes `/metrics`), TSDB, HTTP server (PromQL API, UI) |
| Exporters | Translate third-party systems into the Prometheus format: `node_exporter` (Linux, port 9100), windows, mysqld, blackbox (HTTP/TCP/ICMP probes), haproxy… |
| Client libraries | Instrument your own code (Go, Java, Python, Ruby, Rust…) |
| Pushgateway | Lets **short-lived jobs** push metrics, which Prometheus then scrapes |
| Service discovery | Keeps the target list up to date (Kubernetes, EC2, Consul, files…) |
| Alertmanager | Receives alerts and handles dedup, grouping, routing, silences and notifications |

**Pull vs push:** pulling makes it easy to detect a dead target (a failed scrape means `up == 0`), avoids push storms and keeps the target list central. Push is only for ephemeral or batch jobs, through the Pushgateway.

## Installation

- The tarball contains `prometheus` (server), `prometheus.yml` (config), `promtool` (validation CLI) and `consoles`/`console_libraries`. The UI is on port **9090**.
- systemd setup: a dedicated non-login `prometheus` user, config in `/etc/prometheus`, data in `/var/lib/prometheus` (owned by prometheus), binaries in `/usr/local/bin`, a unit with `ExecStart=… --config.file … --storage.tsdb.path …`, then `daemon-reload`, `start`, `enable`. Logs with `journalctl -u prometheus -f`.
- Node exporter: same pattern; exposes `http://host:9100/metrics`.
- Docker: `prom/prometheus` with the config bind-mounted, the data directory persisted, `-p 9090:9090` and `--web.enable-lifecycle` to allow reloads.

## Configuration (`prometheus.yml`)

```
global:
  scrape_interval: 15s        # default for all jobs
  evaluation_interval: 15s    # how often rules are evaluated
rule_files: ["rules.yml"]
alerting:
  alertmanagers: [{ static_configs: [{ targets: ["localhost:9093"] }] }]
scrape_configs:
  - job_name: node            # a job = a set of instances
    scrape_interval: 30s
    scrape_timeout: 3s
    metrics_path: /metrics    # default
    scheme: https             # default http
    static_configs:
      - targets: ["192.168.1.10:9100", "192.168.1.11:9100"]
```

Validate with `promtool check config`. Reload without restarting with `POST /-/reload` (needs `--web.enable-lifecycle`), with `SIGHUP`, or by restarting the service. Check scraping in **Status → Targets** or with the `up` metric.

## Authentication and encryption

By default anyone who can reach an exporter can read its metrics, and the data travels in plain text. Exporters accept a web config file (`--web.config.file`) with `tls_server_config` (cert and key) and `basic_auth_users` (**bcrypt**-hashed passwords). On the Prometheus side the job uses `scheme: https`, `tls_config: { ca_file: … }` and `basic_auth`. Avoid `insecure_skip_verify` in production.

## Metrics and the data model

A time series is identified by a **metric name plus a set of labels**. Each sample is a float value with a timestamp. Exposition format:

```
# HELP node_filesystem_avail_bytes Filesystem space available.
# TYPE node_filesystem_avail_bytes gauge
node_filesystem_avail_bytes{device="sda1",mountpoint="/home"} 5000 1670000000
```

- The metric name is stored internally as the `__name__` label. Names must match `[a-zA-Z_:][a-zA-Z0-9_:]*`; colons are reserved for recording rules. Labels starting with `__` are internal.
- Prometheus adds `job` and `instance` to every scraped series.

| Type | Behaviour | Use |
|---|---|---|
| Counter | Only goes up; resets to 0 on restart | Requests, errors, bytes (use with `rate()`) |
| Gauge | Goes up and down | Memory, temperature, queue length, in-flight requests |
| Histogram | Cumulative buckets (`_bucket{le=…}`) plus `_sum` and `_count` | Latency or sizes; can be **aggregated across instances** |
| Summary | Quantiles computed client-side (`{quantile="0.9"}`) plus `_sum` and `_count` | Exact per-instance quantiles; **cannot be aggregated** |

## Expression browser and promtool

The web UI has an expression box (table and graph views, evaluation time) and pages for **Configuration, Targets, Rules and Alerts**.

| Command | Purpose |
|---|---|
| `promtool check config` | Validate `prometheus.yml` |
| `promtool check rules` | Validate rule files |
| `promtool test rules` | Unit-test alerting and recording rules |
| `promtool check metrics` | Lint the exposition format |
| `promtool query instant\|range` | Run PromQL against a server |

## Monitoring containers

Docker Engine metrics (set `"metrics-addr"` in `daemon.json`) describe the daemon itself. **cAdvisor** provides per-container CPU, memory and processes. Use cAdvisor for container-level alerting.

# 3. PromQL

## Data types

| Type | Meaning | Example |
|---|---|---|
| String | Plain string (rarely used) | `"hello"` |
| Scalar | One float | `54.7` |
| Instant vector | One sample per series at one timestamp | `node_cpu_seconds_total` |
| Range vector | Several samples per series over a window | `node_cpu_seconds_total[3m]` |

Only instant vectors and scalars can be graphed directly.

## Selectors and modifiers

- Matchers: `=`, `!=`, `=~` (RE2 regex, fully anchored), `!~`. Example: `http_requests_total{job="api", code=~"5.."}`.
- Range selector: `[5m]`. Units are `ms s m h d w y` and can be combined (`1h30m`).
- `offset 1h`: shifts the query back relative to the evaluation time.
- `@ 1663265188` (or `@ start()` / `@ end()`): evaluates at a fixed Unix timestamp. `@` and `offset` can be combined, for example `metric @ 1663265188 offset 5m`.

## Operators

- Arithmetic: `+ - * / % ^`. The result drops the metric name.
- Comparison: `== != > < >= <=`. Without a modifier they **filter** series; with `bool` they return 0 or 1 (`node_load1 > bool 2`).
- Logical (set operators): `and` (intersection), `or` (union), `unless` (left side minus matches on the right).
- Precedence, from highest: `^` (right-associative) → `* / % atan2` → `+ -` → comparisons → `and unless` → `or`.

## Vector matching

A binary operation between two vectors only pairs series with **exactly the same labels**. To control the matching:

- `on(labels)` matches only on those labels; `ignoring(labels)` matches on all labels except those.
- One-to-one is the default. For many-to-one use `group_left` (the left side is the "many" side) or `group_right`, optionally carrying extra labels: `group_left(label)`.

```
node_filesystem_avail_bytes / node_filesystem_size_bytes * 100
http_errors_total / ignoring(code) group_left http_requests_total
```

## Aggregation

`sum`, `min`, `max`, `avg`, `count`, `stddev`, `stdvar`, `group`, `count_values("label", m)`, `topk(k, m)`, `bottomk` and `quantile(φ, m)`.

`sum by(instance) (…)` keeps only the listed labels; `sum without(cpu, mode) (…)` drops the listed labels and keeps the rest.

## Functions (the most examined)

| Function | Use |
|---|---|
| `rate(c[5m])` | Per-second average increase of a counter over the window; handles resets. Use it for graphs and alerts |
| `irate(c[5m])` | Per-second rate using only the last two samples; very spiky |
| `increase(c[1h])` | Total increase over the window (rate × seconds) |
| `delta(g[1h])`, `deriv(g[1h])` | Change and per-second derivative of a **gauge** |
| `predict_linear(g[1h], 4*3600)` | Predicts a gauge's value in N seconds (for example, "disk full in 4 hours") |
| `avg_over_time`, `max_over_time`, `min_over_time`, `sum_over_time`, `count_over_time`, `quantile_over_time` | Aggregate each series over time (take a range vector) |
| `absent(m)`, `absent_over_time` | Return 1 when a series does not exist: alerting on missing data |
| `histogram_quantile(0.95, …)` | Estimate a quantile from buckets |
| `label_replace`, `label_join` | Create or rewrite labels with a regex |
| `sort`, `sort_desc`, `clamp_min/max`, `round`, `ceil`, `floor`, `abs`, `time()`, `timestamp()`, `vector()`, `scalar()` | Utilities |

> [!note]
> Apply `rate()` first and aggregate afterwards: `sum by(job) (rate(x[5m]))`, never `rate(sum(x))`, because summing breaks counter-reset detection. The rate window should be at least 4× the scrape interval.

## Subqueries

The syntax is `<instant expression>[range:resolution]`. It turns the result of an instant expression (such as a rate) into a range vector so you can apply `*_over_time` to it. `max_over_time(rate(x[1m]))` fails because `rate()` returns an instant vector. This works:

```
max_over_time(rate(node_network_transmit_bytes_total[1m])[5h:30s])
```

That means: compute the rate at every 30 s step over the last 5 h, then take the maximum.

## Histograms and summaries in PromQL

- Buckets are **cumulative**: `le="0.05"` includes everything ≤ 0.05, and `le="+Inf"` equals `_count`.
- Average latency: `rate(x_sum[5m]) / rate(x_count[5m])`.
- Share of requests ≤ 0.5 s: `rate(x_bucket{le="0.5"}[5m]) / ignoring(le) rate(x_count[5m])`.
- Quantile: `histogram_quantile(0.95, sum by(le) (rate(x_bucket[5m])))`. Keep `le` in the aggregation. The result is **linearly interpolated** within a bucket, so it is an estimate: put a bucket boundary exactly at your SLO threshold.
- Every bucket is a separate series, so add buckets only where precision matters.

|  | Histogram | Summary |
|---|---|---|
| Quantiles | Estimated server-side, any quantile, any time | Pre-chosen and computed client-side |
| Aggregate across instances | Yes (sum the buckets) | No |
| Accuracy | Depends on bucket layout | Configurable error |

## Recording rules

Recording rules precompute expensive or frequently used expressions and store the result as a new series. Rule files are listed under `rule_files:` (globs allowed) and loaded on reload or restart.

```
groups:
  - name: node                 # groups run in parallel
    interval: 15s              # optional, overrides evaluation_interval
    rules:                     # rules in a group run in order
      - record: job:node_filesystem_free:percent
        expr: 100 * node_filesystem_free_bytes{job="node"} / node_filesystem_size_bytes{job="node"}
        labels: { env: prod }
```

- Naming convention: **`level:metric:operations`**, where level is the labels left after aggregation. Example: `job_method:http_errors:rate5m`.
- Within a group, later rules can use the result of earlier ones in the same evaluation.
- Check them in the UI under **Status → Rules** and validate them with `promtool check rules`.

## HTTP API

| Endpoint | Purpose | Parameters |
|---|---|---|
| `/api/v1/query` | Instant query | `query`, optional `time` |
| `/api/v1/query_range` | Range query for graphs | `query`, `start`, `end`, `step` |
| `/api/v1/series`, `/labels`, `/label/<name>/values` | Metadata | `match[]` |
| `/api/v1/targets`, `/rules`, `/alerts` | State |  |

Responses are JSON with a `resultType` of `vector`, `matrix`, `scalar` or `string`. A range vector in an instant query returns a `matrix`.

```
curl http://localhost:9090/api/v1/query --data 'query=node_arp_entries{instance="192.168.1.168:9100"}' | jq
```

# 4. Dashboarding and visualization

|  | Expression browser | Console templates | Grafana |
|---|---|---|---|
| Best for | Ad-hoc queries | Small static pages | Production dashboards |
| Custom panels and variables | No | Limited | Yes |
| Sharing and RBAC | No | Manual | Yes |

- **Expression browser**: Table and Graph views, autocomplete, several panels, evaluation time. The Graph view needs an **instant vector**; a raw range vector fails, so wrap it in `rate()` and similar functions.
- **Console templates**: HTML plus **Go templates** (`{{ … }}`) stored in `/etc/prometheus/consoles` and served at `/consoles/<file>.html`. They are rendered on each request, so no restart is needed. Helpers include `head`/`tail`, `prom_content_head` and `prom_query_drilldown`.

# 5. Application instrumentation

Official client libraries exist for Go, Java/Scala, Python and Ruby; there are community libraries for Node.js (prom-client), Rust, .NET and others. Your code records metrics with the four primitives and exposes them on `/metrics`.

```
from prometheus_client import Counter, Histogram, Gauge, start_http_server
REQUESTS = Counter('http_requests_total', 'Total requests', ['path', 'method'])
LATENCY  = Histogram('request_latency_seconds', 'Latency', ['path', 'method'],
                     buckets=[0.1, 0.25, 0.5, 1, 2.5])
INPROG   = Gauge('inprogress_requests', 'In-flight requests')

REQUESTS.labels('/cars', 'GET').inc()
LATENCY.labels('/cars', 'GET').observe(0.231)
INPROG.inc(); INPROG.dec(); INPROG.set(3)
start_http_server(8000)   # metrics on a separate port
```

- To expose metrics on the app's own port in Flask, use `DispatcherMiddleware` with `make_wsgi_app()` mounted at `/metrics`.
- Use **one metric with labels** (path, method), not one metric per endpoint, so you can aggregate with `sum`. Pass label values in the same order as the declared label names.
- Measure latency with Flask's `before_request` and `after_request`. Decrement gauges in `finally` or teardown handlers so exceptions don't leave them high.

## Naming best practices

- Use `snake_case` with the owning application or library as a prefix: `postgres_queue_size_bytes`, not `container_docker_restarts`.
- Include the **base unit** without prefixes: `seconds` and `bytes`, not `milliseconds` or `kilobytes`.
- Counters end in `_total`. `_count`, `_sum` and `_bucket` are reserved for histograms and summaries.
- Avoid labels with unbounded values (user IDs, request IDs, raw URLs).

| System type | What to expose |
|---|---|
| Online serving | Requests, errors, latency, in-progress requests (the RED method) |
| Offline processing | Items processed, queue size, work in progress, throughput, errors |
| Batch jobs | Runtime per stage, total runtime, last success time, success and failure counts (via the Pushgateway) |

# 6. Service discovery and relabeling

Service discovery keeps the target list current in dynamic environments. Mechanisms include `static_configs`, `file_sd_configs`, `kubernetes_sd_configs`, `ec2_sd_configs`, GCE, Azure, `consul_sd_configs`, Nomad, DNS and HTTP. Discovered targets come with `__meta_*` labels, which you can inspect under **Status → Service Discovery**.

- **File SD**: JSON or YAML files of `[{targets: […], labels: {…}}]`, globs allowed, re-read automatically on change. Write the files atomically.
- **EC2**: needs a region and credentials (prefer an IAM role with `AmazonEC2ReadOnlyAccess`). Labels include `__meta_ec2_tag_Name`, `__meta_ec2_private_ip`, `__meta_ec2_instance_type` and others. The private IP is used by default.

## Relabeling

|  | `relabel_configs` | `metric_relabel_configs` |
|---|---|---|
| When | **Before** the scrape | **After** the scrape, before storage |
| Sees | Target and discovery labels (`__meta_*`, `__address__`) | Every scraped series, including `__name__` |
| Use | Choose targets, turn metadata into labels | Drop or rename metrics and labels |

| Action | Effect |
|---|---|
| `keep` / `drop` | Keep or drop targets (or series) whose `source_labels` match the `regex` |
| `replace` (default) | Write `replacement` (with `$1` captures) into `target_label` |
| `labelmap` | Copy labels whose **names** match the regex, renamed: `__meta_ec2_(.*)` → `ec2_$1` |
| `labeldrop` / `labelkeep` | Remove or keep labels by name |
| `hashmod` | Hash into N buckets (sharding) |

```
relabel_configs:
  - source_labels: [__meta_ec2_tag_env]
    regex: prod
    action: keep
  - source_labels: [__address__]
    regex: (.*):.*
    target_label: ip
    replacement: $1
```

Several `source_labels` are joined with `;` before matching. The regex is fully anchored. Labels starting with `__` are removed after relabeling. Special labels: `__address__`, `__scheme__`, `__metrics_path__` and `__param_*`.

# 7. Pushgateway

A temporary store for metrics from **short-lived batch jobs** that would exit before being scraped. The job pushes before it exits, and Prometheus scrapes the Pushgateway (port **9091**) like any other target. It is not meant for long-running services and it does not expire data by itself.

- The URL path is the **grouping key**: `/metrics/job/<job>/instance/<x>/…` becomes labels.
- Use `honor_labels: true` on the scrape job so the pushed `job` and `instance` labels are kept instead of being overwritten by the Pushgateway's own.

| HTTP | Client library function | Effect |
|---|---|---|
| `PUT` | `push_to_gateway` | Replace the whole group |
| `POST` | `pushadd_to_gateway` | Replace only metrics with the same name; keep the rest |
| `DELETE` | `delete_from_gateway` | Remove the group |

```
echo "backup_last_success_seconds 1700000000" | \
  curl --data-binary @- http://localhost:9091/metrics/job/db_backup/instance/db1
```

In Python, push from a dedicated `CollectorRegistry` and use `grouping_key={'instance': 'worker-1'}`. Cons: the Pushgateway is a single point of failure, you lose `up` health monitoring for the job, and series stay until you delete them.

# 8. Alerting

## Alert rules

```
groups:
  - name: node
    rules:
      - alert: LowMemory
        expr: job:node_memory_free:percent < 20
        for: 3m
        labels: { severity: critical }
        annotations:
          summary: "Low memory on {{ $labels.instance }}"
          description: "Free memory is {{ $value }}%"
          runbook_url: https://wiki/runbooks/low-memory
```

| State | Meaning |
|---|---|
| Inactive | The expression returns nothing |
| Pending | The expression is true, but not yet for the full `for` duration |
| Firing | True for at least `for` (or immediately if there is no `for`); sent to Alertmanager |

- **Labels** are for machines: routing, grouping and inhibition (for example `severity` or `team`). Keep them stable and low-cardinality.
- **Annotations** are for humans: summary, description, runbook. They support Go templates: `{{ $labels.instance }}`, `{{ $value }}`, and `{{ index $labels "label-name" }}` for names with dashes.
- Prometheus sends the alert to Alertmanager only while it is firing, and keeps re-sending it until it resolves.

## Alertmanager

![[pca-diagram-02.svg]]

*How an alert flows through Alertmanager.*

- Install: binaries `alertmanager` and `amtool`, config `alertmanager.yml`, a `data/` directory, port **9093**. With systemd: `--config.file=/etc/alertmanager/alertmanager.yml --storage.path=/var/lib/alertmanager`.
- Prometheus points at it in the `alerting.alertmanagers` section of its config.
- **Grouping** merges related alerts into one notification (`group_by`). **Inhibition** mutes alert Y while alert X is firing (for example, mute warnings when a critical alert fires for the same cluster). **Silences** are time-limited mutes created in the UI or with `amtool`; they do not change the alert's state in Prometheus.
- High availability: run several instances clustered with a gossip protocol (`--cluster.*`) and point every Prometheus at **all** of them. Do not put a load balancer in front.

## Alertmanager configuration

```
global:
  smtp_smarthost: mail.example.com:25
  smtp_from: alerts@example.com
  slack_api_url: https://hooks.slack.com/...
route:                          # top-level = fallback
  receiver: staff
  group_by: [alertname, job]
  group_wait: 30s               # wait before first notification of a new group
  group_interval: 5m            # wait before notifying about new alerts in a group
  repeat_interval: 4h           # re-send while still firing
  routes:
    - match_re: { job: (node|windows) }
      receiver: infra-email
    - match: { job: kubernetes }
      receiver: k8s-email
      routes:
        - match: { severity: pager }
          receiver: k8s-pager
    - match: { team: logs }
      receiver: alert-logs
      continue: true            # keep evaluating sibling routes
inhibit_rules:
  - source_matchers: [severity="critical"]
    target_matchers: [severity="warning"]
    equal: [cluster]
receivers:
  - name: infra-email
    email_configs: [{ to: infra@example.com }]
  - name: k8s-pager
    pagerduty_configs: [{ routing_key: ... }]
```

- The route tree is **depth-first**: the first matching child wins unless `continue: true`. Child routes inherit unset fields such as `group_by` and `receiver`. Matchers are `match` (exact), `match_re` (regex) or the newer `matchers: ["severity=~"..."]`.
- A **receiver** is a named destination holding one or more **notifiers**: `email_configs`, `slack_configs`, `pagerduty_configs`, `opsgenie_configs`, `webhook_configs`, `victorops_configs`… Shared settings go in `global`.
- Notification templates use Go templates over `.Status`, `.Receiver`, `.GroupLabels`, `.CommonLabels`, `.CommonAnnotations`, `.ExternalURL` and `.Alerts` (each with `.Labels`, `.Annotations`, `.StartsAt`, `.EndsAt`). For example `{{ .Alerts | len }}` or `{{ range .Alerts }}…{{ end }}`.
- Reload with a restart, `SIGHUP` or `POST /-/reload`. Validate with `amtool check-config`.
- **Silences**: set a start time, an end time or duration, label matchers (`=`, `=~`), a creator and a comment. You can expire them early. They are also available through the API and `amtool silence add`.

# 9. Monitoring Kubernetes

| Source | What it provides |
|---|---|
| Control plane (API server, scheduler, etcd, CoreDNS) | Metrics about the cluster's own components |
| kubelet / cAdvisor | Container and node metrics (`/metrics/cadvisor`) |
| **kube-state-metrics** | State of objects (Deployments, Pods, replicas…). Kubernetes does not expose these by default |
| node_exporter | OS metrics; deployed as a **DaemonSet** |

The easiest install is the Helm chart **`kube-prometheus-stack`** (repo `prometheus-community`). It deploys the Prometheus Operator, Prometheus and Alertmanager (as StatefulSets), Grafana, kube-state-metrics (Deployments) and node-exporter (a DaemonSet on port 9100), with ClusterIP Services. Reach them with `kubectl port-forward`, NodePort, LoadBalancer or an Ingress. Customise it with `helm show values … > values.yaml` and `helm upgrade -f values.yaml`.

## Prometheus Operator CRDs

| CRD | Purpose |
|---|---|
| `Prometheus` / `Alertmanager` | Declare the server instances |
| `ServiceMonitor` | Scrape the endpoints behind Services selected by labels (`selector.matchLabels`, `endpoints[].port` refers to the port **name**, plus `path`, `interval`, `jobLabel`) |
| `PodMonitor` | Scrape pods directly |
| `PrometheusRule` | Recording and alerting rules (same `groups` format) |
| `AlertmanagerConfig` | Routes and receivers per namespace. Fields are camelCase (`groupWait`) and matchers are explicit (`name`, `value`) |

> [!note]
> The objects must carry the labels the Prometheus resource selects (`serviceMonitorSelector`, `ruleSelector`, `alertmanagerConfigSelector`), typically `release: prometheus`. Otherwise they are ignored. Do not edit the generated Secret or ConfigMaps: the operator overwrites them.

- Kubernetes SD roles: `node`, `service`, `pod`, `endpoints` (the most flexible, and what the chart uses) and `ingress`. ServiceMonitors become `scrape_configs` that filter the endpoints with `keep` relabel rules.
- `additionalScrapeConfigs` in the Helm values lets you add raw jobs, but this is fragile; prefer ServiceMonitors.

# 10. Scaling and long-term storage

## Local TSDB

- The data directory holds 2-hour **blocks** (`chunks/`, `index`, `meta.json`, tombstones) that are later compacted into bigger blocks, plus the in-memory **head** (`chunks_head`) and the **WAL** for crash recovery.
- Flags: `--storage.tsdb.path`, `--storage.tsdb.retention.time` (default **15d**) and `--storage.tsdb.retention.size`.
- Disk needed ≈ retention seconds × samples per second × bytes per sample (about 1–2 bytes). Watch `prometheus_tsdb_head_series` for cardinality.
- Limits: a single node with no built-in clustering or replication, and local disk only. For durability use `remote_write` / `remote_read` to Thanos, Cortex, Mimir, VictoriaMetrics…

## Scaling options

| Approach | How | Trade-off |
|---|---|---|
| Vertical / functional | One Prometheus per team or service | Clear ownership, isolates cardinality; more instances to run |
| Horizontal sharding | Same target list everywhere; `hashmod` with `modulus: N`, then `keep` on that server's shard number | Splits huge jobs; each shard only sees its own data |
| Federation | A global Prometheus scrapes `/federate` with `params.match[]` and `honor_labels: true` | Only for **aggregated** series (recording rules); label each datacenter to avoid collisions |
| Thanos / Cortex / Mimir | Global query and object-storage backed retention | More components to run |

```
relabel_configs:            # shard 0 of 2
  - source_labels: [__address__]
    modulus: 2
    target_label: __tmp_hash
    action: hashmod
  - source_labels: [__tmp_hash]
    regex: "0"
    action: keep
```

On a single server you can lengthen `scrape_interval` for slow metrics and use `sample_limit` to protect against one target exploding in cardinality.

## Thanos

| Component | Role |
|---|---|
| Sidecar | Runs next to each Prometheus; serves its data over gRPC (StoreAPI) and uploads blocks to object storage (S3, GCS, Azure) |
| Store gateway | Serves historical blocks from object storage |
| Query (Querier) | Global PromQL endpoint: merges sidecars and stores, and **deduplicates** HA pairs |
| Compactor, Ruler, Receive | Compaction and downsampling, global rules, ingesting `remote_write` |

# Exam cheat sheet

| Topic | Remember |
|---|---|
| Ports | Prometheus 9090 · node_exporter 9100 · Pushgateway 9091 · Alertmanager 9093 (9094 for clustering) |
| Defaults | scrape_interval 1m (15s in the sample config) · metrics_path `/metrics` · scheme http · retention 15d |
| Auto labels | `job`, `instance`; synthetic series `up`, `scrape_duration_seconds`, `scrape_samples_scraped` |
| Types | Counter, Gauge, Histogram (aggregatable), Summary (not aggregatable) |
| rate vs irate | rate = average over the window, for alerts; irate = last two samples, spiky. Always rate before sum |
| Vector matching | `on`/`ignoring`, `group_left`/`group_right` for many-to-one |
| Subquery | `expr[range:resolution]` |
| Rule naming | `level:metric:operations` |
| Alert states | inactive → pending (`for`) → firing |
| Alertmanager | group_wait / group_interval / repeat_interval · `continue` · inhibition ≠ silence |
| Relabeling | `relabel_configs` before the scrape, `metric_relabel_configs` after |
| Pushgateway | Batch jobs only; PUT replaces, POST merges, DELETE removes; `honor_labels: true` |
| Naming | snake_case, base units, `_total` for counters |
| Kubernetes | kube-prometheus-stack; ServiceMonitor, PodMonitor, PrometheusRule; selector labels |


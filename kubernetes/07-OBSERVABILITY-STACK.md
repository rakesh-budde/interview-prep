# Observability Stack (5% of Interview Weight)

> Logging architecture, Prometheus internals, distributed tracing

---

## 7.1 Logging

### Log Collection Architecture

```
┌────────────────────────────────────────────────────────────┐
│ NODE-LEVEL                                                    │
│  Container stdout/stderr → captured by container runtime      │
│  (containerd) → written to JSON log files on the node:         │
│  /var/log/pods/<namespace>_<pod>_<uid>/<container>/*.log       │
│  (rotated by kubelet based on --container-log-max-size/files)  │
│                                                                 │
│  `kubectl logs` works by: API server → kubelet on the pod's    │
│  node (proxied call) → kubelet reads the log FILE directly     │
│  and streams it back. NOT stored in etcd, NOT centrally        │
│  aggregated by Kubernetes itself — this is why logs DISAPPEAR  │
│  when a pod is deleted or the node is replaced (ephemeral by   │
│  default, unless you ship them out).                            │
└────────────────────────────────────────────────────────────┘
                │
                ▼
┌────────────────────────────────────────────────────────────┐
│ CLUSTER-LEVEL (you must deploy this yourself)                 │
│  Fluent Bit / Fluentd DaemonSet (one per node):                │
│  ├─ Tails /var/log/pods/*/*/*.log files                        │
│  ├─ Parses/enriches with K8s metadata (namespace, pod, labels  │
│  │  — via the kubernetes_metadata filter, calling the API      │
│  │  server or using the local kubelet API to look up pod info) │
│  └─ Ships to a backend: Elasticsearch, Loki, cloud logging      │
│     (CloudWatch Logs, Azure Log Analytics, GCP Cloud Logging)   │
│                                                                 │
│  EFK/ELK stack: Elasticsearch (storage+search) + Fluentd/       │
│  Fluent Bit (collection) + Kibana (visualization)               │
│  Loki stack: Loki (storage, indexes only LABELS not full text — │
│  much cheaper than Elasticsearch) + Promtail/Fluent Bit +       │
│  Grafana (same UI as your metrics dashboards — unified view)    │
└────────────────────────────────────────────────────────────┘

FLUENT BIT VS FLUENTD: Fluent Bit is a lightweight C-based rewrite
(lower memory footprint, ~450KB vs Fluentd's larger Ruby-based
footprint), preferred as the DaemonSet-per-node agent at scale;
Fluentd is often used as a heavier AGGREGATOR tier (receiving from
many Fluent Bit forwarders, doing more complex processing/routing)
— a common pattern is Fluent Bit (node agent) → Fluentd (aggregator)
→ backend, splitting cheap per-node work from expensive
processing/routing logic.
```

### Structured Logging Best Practices

```
UNSTRUCTURED (bad for aggregation):
"2024-01-15 ERROR Failed to process order 12345 for user alice: timeout"

STRUCTURED JSON (good — queryable, filterable at scale):
{"timestamp":"2024-01-15T10:23:45Z","level":"error",
 "message":"failed to process order","order_id":"12345",
 "user":"alice","error":"timeout","service":"order-processor",
 "trace_id":"abc123..."}

WHY IT MATTERS AT SCALE: with structured fields, your log backend
can efficiently FILTER/AGGREGATE ("show me all errors for order_id=X
across ALL services") — impossible/expensive with free-text log
parsing (regex-based extraction at query time is slow and fragile).
Also enables correlating logs with the SAME trace_id across
microservices (see tracing section) — this is the critical link
between logging and distributed tracing pillars of observability.
```

### Interview Questions — Logging

**Q1: Design centralized logging for a 1000-node cluster.**
> Fluent Bit DaemonSet (lightweight, one per node) tailing container log files, enriching with K8s metadata, forwarding to Loki (cost-effective at this scale since it indexes only labels, not full log text, unlike Elasticsearch which indexes everything and gets expensive/slow at high volume) or a cloud-native logging backend with native K8s integration. Enforce structured JSON logging application-wide (via a shared logging library/sidecar) for efficient querying. Set log retention tiers (hot/short-term in fast storage, cold/long-term in cheap object storage) and implement sampling for extremely high-volume, low-value logs (e.g. health check access logs) to control cost, while ensuring 100% capture for error-level+ logs.

**Q2: Application logs are missing. Debug it.**
> Check the app is actually logging to stdout/stderr (NOT to a file inside the container — Kubernetes/container runtime only captures stdout/stderr by default, a very common misconfiguration where an app's default logging config writes to a local file that's never collected). Check node disk pressure hasn't caused kubelet to aggressively rotate/delete log files before the collector agent read them (`--container-log-max-size`). Check the Fluent Bit/Fluentd DaemonSet pod on that SPECIFIC node is healthy (`kubectl logs` the daemonset pod itself for its own collection errors — connectivity to backend, parsing failures, etc). Check backend-side retention/index issues (query time range wrong, index not yet refreshed).

---

## 7.2 Metrics & Monitoring

### Prometheus Architecture in Kubernetes

```
PROMETHEUS OPERATOR PATTERN (kube-prometheus-stack):

┌──────────────────────────────────────────────────────────┐
│ ServiceMonitor / PodMonitor (CRDs)                          │
│  Declaratively tell Prometheus WHAT to scrape:                │
│  "any Service matching label app=my-app, scrape /metrics       │
│   on port 'http-metrics' every 30s"                            │
│  Prometheus Operator watches these CRDs and auto-generates      │
│  the actual Prometheus scrape_config — NO manual editing of     │
│  Prometheus's config file needed as services come and go        │
└──────────────────────────────────────────────────────────────┘
                │
                ▼
┌──────────────────────────────────────────────────────────┐
│ Prometheus server (StatefulSet)                              │
│  ├─ Pull-based scraping (Prometheus reaches OUT to targets,  │
│  │  not the reverse — targets just expose a /metrics HTTP    │
│  │  endpoint in the Prometheus text exposition format)        │
│  ├─ Local TSDB storage (time-series database, on a PVC),      │
│  │  typically short retention (15d) for cost — long-term      │
│  │  handled by remote_write to Thanos/Cortex/Mimir             │
│  ├─ Recording rules: pre-compute expensive PromQL queries      │
│  │  into new time series (e.g. http_requests:rate5m) for       │
│  │  faster dashboard/alert evaluation                          │
│  └─ Alerting rules: PromQL expression that, when true for a    │
│     sustained "for:" duration, fires an alert to Alertmanager  │
└──────────────────────────────────────────────────────────────┘
                │
                ▼
┌──────────────────────────────────────────────────────────┐
│ Alertmanager: dedup, group, route, silence alerts             │
│  → PagerDuty/Slack/email based on routing tree + label match   │
└──────────────────────────────────────────────────────────────┘
```

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: my-app-monitor
  labels:
    release: prometheus       # must match Prometheus's serviceMonitorSelector
spec:
  selector:
    matchLabels:
      app: my-app
  endpoints:
  - port: http-metrics
    interval: 30s
    path: /metrics
---
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: my-app-alerts
spec:
  groups:
  - name: my-app
    rules:
    - alert: HighErrorRate
      expr: |
        sum(rate(http_requests_total{status=~"5.."}[5m]))
        /
        sum(rate(http_requests_total[5m])) > 0.01
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "Error rate above 1% for 5 minutes"
```

### Scaling Prometheus for Large Clusters

```
PROBLEM: single Prometheus instance scraping 1000s of pods hits
memory limits (entire TSDB index + recent samples held in memory)
and single-node write throughput ceilings.

SOLUTIONS:
├─ Sharding: multiple Prometheus instances, each scraping a SUBSET
│  of targets (via ServiceMonitor selector partitioning, or
│  hashmod relabeling on target address) — but this FRAGMENTS
│  queries across instances (a query needing data from 2 shards
│  must be handled at the query layer, e.g. via Thanos Querier
│  federating multiple Prometheus sources transparently)
├─ Thanos / Cortex / Mimir: long-term, horizontally-scalable storage
│  and GLOBAL query view across many Prometheus shards + long-term
│  object storage backend (S3/Blob) beyond local TSDB retention
├─ Reduce cardinality: high-cardinality labels (e.g. raw user ID,
│  full request path with IDs embedded) explode the number of
│  unique time series — the #1 cause of Prometheus OOM in practice.
│  Fix at the application instrumentation level (bucket/aggregate
│  labels, don't put unbounded values in labels)
└─ Recording rules to pre-aggregate before dashboards/alerts query
   raw high-cardinality series repeatedly
```

### Interview Questions — Metrics

**Q1: Design monitoring for 500 microservices.**
> Standardize instrumentation (Prometheus client libraries with a shared convention for metric naming/labels across all services — critical for consistent dashboards/alerts at this scale). Use ServiceMonitor auto-discovery so onboarding a new service to monitoring requires zero manual Prometheus config changes. Shard Prometheus if needed (by team/namespace), federate with Thanos for a unified long-term query layer and cross-shard global view. Define SLO-based alerting (error budget burn rate alerts, not just static thresholds) to reduce alert fatigue across 500 services, and use recording rules aggressively for expensive cross-service queries feeding dashboards.

**Q2: Prometheus is OOM. Optimize it.**
> First identify if it's cardinality explosion (`prometheus_tsdb_symbol_table_size_bytes`, or query `count by (__name__)(count({__name__=~".+"}))` sorted descending to find the worst offending metric names) — the most common root cause is a label with unbounded values (user IDs, timestamps, full URLs). Fix at the source (instrumentation) or drop/relabel the high-cardinality label at scrape time. If genuinely just scale (not cardinality), shard Prometheus instances or move to Thanos/Mimir/Cortex for proper horizontal scaling. Also check retention settings aren't unnecessarily long for local storage (offload older data to remote_write/object storage instead of keeping it all in local TSDB).

---

## 7.3 Distributed Tracing

### Tracing Architecture

```
PROBLEM TRACING SOLVES: a single user request in a microservices
architecture may touch 10+ services — logs and metrics alone can't
show you the END-TO-END request path and where time was spent.

CORE CONCEPTS:
├─ Trace: represents one end-to-end request, made up of multiple
│  Spans
├─ Span: a single unit of work (e.g. one service's handling of the
│  request, or one DB query) with start/end time, tags, and a
│  parent-child relationship to other spans
├─ Trace Context Propagation: the trace_id + span_id must be passed
│  along with EVERY downstream call (via HTTP headers, typically the
│  W3C `traceparent` header standard) so each service's span can be
│  correctly linked into the same overall trace
└─ Sampling: capturing 100% of traces in high-traffic systems is
   expensive (storage + overhead) — most systems sample a percentage
   (e.g. 1-10%) or use "tail-based sampling" (buffer all spans for a
   trace, decide to keep/discard AFTER seeing the full trace — e.g.
   always keep traces containing an error or unusually high latency,
   discard "boring" fast successful ones) for the best signal-to-cost
   ratio

OPENTELEMETRY: the current industry-standard, vendor-neutral
instrumentation framework/API (replaced the older, now-deprecated
OpenTracing and OpenCensus projects) — a single SDK integration lets
you export traces to ANY compatible backend (Jaeger, Tempo, Zipkin,
Datadog, etc.) via the OpenTelemetry Collector, avoiding vendor
lock-in at the instrumentation layer.

TYPICAL DEPLOYMENT:
App (OpenTelemetry SDK) → OTel Collector (DaemonSet or sidecar,
  does batching/sampling/filtering/enrichment) → Tracing backend
  (Jaeger/Tempo) → Query UI (Jaeger UI or Grafana with Tempo
  datasource, often correlated directly with logs/metrics in the
  same Grafana dashboard for full "three pillars" observability)
```

### Interview Questions — Tracing

**Q1: Debug a latency issue across 10 microservices.**
> Use the distributed trace for the SPECIFIC slow request (search by trace_id if you have it from a log/error report, or search traces by duration/service in Jaeger/Tempo UI) — the trace's waterfall/flamegraph view directly shows which SPAN (which service, which specific operation) consumed the most time, immediately narrowing the investigation from "10 services" to "this ONE specific downstream call in service #6 taking 800ms." Cross-reference that span's tags/logs (often trace and log systems are linked via trace_id) for the specific error/slow-query detail within that service.

**Q2: Design observability for a service mesh.**
> Sidecar proxies (Envoy in Istio) already generate detailed per-request metrics (latency histograms, success/error rates) and CAN auto-generate spans for tracing WITHOUT application code changes for the network-hop portion — though full end-to-end tracing still requires the application to propagate the trace context headers between its OWN business logic calls (mesh can't do this part automatically, it only sees network boundaries). Combine mesh-level "automatic" L7 metrics/tracing with OpenTelemetry app-level instrumentation for a complete picture, feeding a unified Grafana view correlating traces (Tempo), metrics (Prometheus via mesh + app), and logs (Loki) by consistent labels (trace_id, service name).

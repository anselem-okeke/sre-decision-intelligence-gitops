# Prometheus & Grafana Advanced Production Cheat Sheet v4

**Focus:** Senior-level Prometheus, Grafana, Kubernetes observability, long-term storage, scaling, GitOps, OpenTelemetry, alert quality, and production operating models.

This v4 is designed to be committed into Git and consumed gradually.

Recommended path in a repository:

```text
docs/observability/prometheus-grafana-cheatsheet-v4.md
```

---

# 0. How To Use This v4

This document is not meant to be memorized in one day.

Use it in layers:

```text
Week 1: Prometheus scaling and TSDB internals
Week 2: Thanos / Mimir / Cortex / remote write
Week 3: Grafana provisioning and dashboards-as-code
Week 4: Alert testing, runbooks, and production alert maturity
Week 5: OpenTelemetry, exemplars, and tracing connection
Week 6: Cost, retention, cardinality budgets, and senior interview scenarios
```

You already need the basics from v1-v3:

```text
v1: Metric types
v2: PromQL and Grafana dashboard query patterns
v3: Production scraping, exporters, ServiceMonitor, cardinality, Alertmanager
v4: Scaling, architecture, GitOps, long-term operations, and senior production design
```

---

# 1. Senior Observability Mental Model

A mature observability platform is not only dashboards.

It is an operating system for production visibility.

```text
Instrumentation
    ↓
Collection
    ↓
Storage
    ↓
Query
    ↓
Dashboard
    ↓
Alert
    ↓
Incident response
    ↓
Runbook
    ↓
Improvement loop
```

## Senior mindset

A junior engineer asks:

```text
What query shows CPU?
```

A senior engineer asks:

```text
What decision does this metric support?
Who owns it?
What is the signal quality?
How expensive is this metric?
Does it alert on symptoms or causes?
Does it help reduce time to detection or time to recovery?
Is the dashboard reproducible from Git?
Can this scale across clusters?
```

---

# 2. Prometheus TSDB Internals

Prometheus stores time series locally in its TSDB.

Important internal concepts:

| Concept | Meaning |
|---|---|
| Sample | A timestamped metric value |
| Time series | Metric name + full label set |
| Head block | Recent in-memory data |
| WAL | Write-ahead log for crash recovery |
| Block | Persistent chunk of time-series data |
| Compaction | Merging smaller blocks into larger blocks |
| Retention | How long data is kept |
| Chunk | Compressed group of samples |
| Index | Enables label-based query lookup |

---

## 2.1 Simplified storage flow

```text
Scraped sample
    ↓
Head block memory
    ↓
WAL written to disk
    ↓
Compacted into persistent blocks
    ↓
Old blocks removed by retention policy
```

## 2.2 WAL

WAL means **Write-Ahead Log**.

It protects recent data against crashes.

If Prometheus restarts, it replays the WAL to rebuild recent in-memory state.

## 2.3 Blocks

Prometheus stores older samples in blocks.

A block contains:

```text
chunks
index
metadata
tombstones
```

## 2.4 Retention

Retention can be configured by time or size.

Examples:

```text
--storage.tsdb.retention.time=15d
--storage.tsdb.retention.size=100GB
```

## Interview answer

> Prometheus stores recent samples in memory and writes them to the WAL for crash recovery. Older samples are compacted into local TSDB blocks. Retention controls how long or how much data Prometheus keeps. This matters because high cardinality increases memory, WAL, block size, compaction pressure, and query cost.

---

# 3. Prometheus Scaling Limits

Prometheus is powerful, but one Prometheus cannot scale infinitely.

Main scaling pressure comes from:

```text
Number of active series
Scrape interval
Number of targets
Samples per second
Query complexity
Retention duration
Dashboard query load
Recording and alerting rules
Remote write volume
```

---

## 3.1 Key scaling metrics

### Active time series

```promql
prometheus_tsdb_head_series
```

### Samples ingested per second

```promql
rate(prometheus_tsdb_head_samples_appended_total[5m])
```

### Scrape samples by job

```promql
topk(
  20,
  sum by (job) (
    rate(scrape_samples_scraped[5m])
  )
)
```

### Query duration

```promql
prometheus_engine_query_duration_seconds
```

### Rule evaluation duration

```promql
prometheus_rule_group_duration_seconds
```

### WAL fsync duration

```promql
prometheus_tsdb_wal_fsync_duration_seconds
```

---

## 3.2 When Prometheus struggles

Symptoms:

```text
High memory usage
Slow Grafana dashboards
Queries timing out
Rule evaluations missing deadlines
WAL replay takes long after restart
Remote write queue backing up
Prometheus pod OOMKilled
High disk usage
Compaction pressure
```

## 3.3 Common scaling fixes

| Problem | Fix |
|---|---|
| Too many active series | Reduce cardinality |
| Heavy dashboard queries | Use recording rules |
| Too many clusters | Use Thanos/Mimir/Cortex/VictoriaMetrics |
| Long retention on local Prometheus | Use remote storage |
| Too many scrape samples | Drop unnecessary metrics |
| Expensive label dimensions | Remove high-cardinality labels |
| Frequent dashboard refresh | Increase refresh interval |
| Rule evaluation slow | Split rule groups / precompute |

## Interview answer

> Prometheus scaling is mostly driven by active series, samples per second, scrape interval, retention, and query load. I would first check active series, scrape samples by job, high-cardinality metrics, dashboard query patterns, and rule evaluation duration. Then I would reduce cardinality, introduce recording rules, adjust scrape intervals, and use remote storage for long-term or multi-cluster needs.

---

# 4. Cardinality Budgeting

Cardinality control should not be reactive only.

Mature teams define a cardinality budget.

## 4.1 What is a cardinality budget?

A limit or expectation for how many series a service/team/namespace/job should produce.

Example:

```text
Each service should stay below 50,000 active time series.
Each team owns the cost of its metrics.
High-cardinality labels must be reviewed before production.
```

## 4.2 Dangerous labels

Avoid unbounded labels:

```text
user_id
request_id
session_id
trace_id
email
ip
uuid
timestamp
full_url
customer_id
container_id
pod_uid
```

## 4.3 Safer labels

Prefer bounded labels:

```text
service
namespace
pod
node
deployment
route_template
method
status
region
cluster
team
environment
```

## 4.4 Cardinality review checklist

Before adding a metric, ask:

```text
Can this label have unlimited values?
Can this be aggregated later?
Does this label support an operational decision?
Is this metric needed as a time series?
Could this be a log or trace instead?
Can the route be normalized?
Who owns the metric?
What retention does it need?
```

## 4.5 Senior interview phrase

> I treat cardinality like cost. Every label increases the number of time series, and every time series consumes memory, CPU, storage, and remote-write bandwidth. For production, I prefer bounded labels and metric reviews before instrumentation is deployed.

---

# 5. Recording Rules Strategy

Recording rules should not be random.

They are a production design tool.

## 5.1 When to use recording rules

Use recording rules when:

```text
The query is expensive
The query is used by many dashboards
The query is used by alerts
The query encodes an SLI/SLO
The query aggregates high-cardinality raw metrics
The query must be standardized across teams
```

## 5.2 Naming convention

Good recording rule names describe:

```text
scope:metric:operation_window
```

Examples:

```text
namespace:cpu_usage_cores:rate5m
namespace:memory_working_set_bytes:sum
service:http_requests:rate5m
service:http_error_ratio:rate5m
service:http_request_duration_seconds:p95_rate5m
cluster:node_cpu_utilization:ratio5m
```

## 5.3 Example: request rate

```yaml
groups:
  - name: service-sli.rules
    rules:
      - record: service:http_requests:rate5m
        expr: |
          sum by (cluster, namespace, service) (
            rate(http_requests_total[5m])
          )
```

## 5.4 Example: error ratio

```yaml
groups:
  - name: service-sli.rules
    rules:
      - record: service:http_error_ratio:rate5m
        expr: |
          sum by (cluster, namespace, service) (
            rate(http_requests_total{status=~"5.."}[5m])
          )
          /
          sum by (cluster, namespace, service) (
            rate(http_requests_total[5m])
          )
```

## Interview answer

> I use recording rules to precompute expensive or repeated queries, especially SLI/SLO expressions and high-cardinality aggregations. I also use naming conventions so the recorded metric clearly shows scope, metric, operation, and time window.

---

# 6. Alert Quality and Alert Maturity

A production alert should create action, not noise.

## 6.1 Bad alert characteristics

```text
Fires on symptoms with no action
Fires on short spikes
No owner
No severity
No runbook
No service impact
No threshold reasoning
No routing label
Too many duplicate alerts
Alerts on raw resource usage without context
```

## 6.2 Good alert characteristics

```text
Actionable
Owned
Routed correctly
Has severity
Has runbook
Has useful annotations
Detects user or platform impact
Uses a stable time window
Has a for: duration
Avoids duplicate symptom storms
```

## 6.3 Alert maturity ladder

| Level | Description |
|---|---|
| 0 | No alerts |
| 1 | Basic threshold alerts |
| 2 | Alerts with severity and routing |
| 3 | Alerts with runbooks and ownership |
| 4 | SLO-based alerts |
| 5 | Multi-window burn-rate alerts |
| 6 | Alerts linked to automation and incident learning |

---

## 6.4 Example production alert

```yaml
groups:
  - name: service-slo-alerts
    rules:
      - alert: HighServiceErrorRate
        expr: |
          service:http_error_ratio:rate5m > 0.05
        for: 10m
        labels:
          severity: critical
          team: platform
          signal: availability
        annotations:
          summary: "High 5xx error rate for {{ $labels.service }}"
          description: "Service {{ $labels.service }} in namespace {{ $labels.namespace }} has more than 5% 5xx responses for 10 minutes."
          runbook_url: "https://github.com/YOUR_ORG/YOUR_REPO/blob/main/runbooks/high-service-error-rate.md"
```

## Interview answer

> I do not consider an alert mature unless it is actionable, owned, routed, has useful context, and ideally has a runbook. Alerts should reduce time to detection and recovery, not just create noise.

---

# 7. promtool

`promtool` is used to validate and test Prometheus configuration and rules.

## 7.1 Validate Prometheus config

```bash
promtool check config prometheus.yml
```

## 7.2 Validate rule files

```bash
promtool check rules alerts.yml
```

## 7.3 Test alert rules

Example test file:

```yaml
rule_files:
  - alerts.yml

evaluation_interval: 1m

tests:
  - interval: 1m
    input_series:
      - series: 'http_requests_total{service="api",status="500"}'
        values: '0+10x20'
      - series: 'http_requests_total{service="api",status="200"}'
        values: '0+100x20'
    alert_rule_test:
      - eval_time: 15m
        alertname: HighServiceErrorRate
        exp_alerts:
          - exp_labels:
              service: api
              severity: critical
```

Run:

```bash
promtool test rules test-alerts.yml
```

## Interview answer

> I use promtool in CI to validate Prometheus configs and rule files before they reach production. For critical alert rules, I also use rule tests so I can prove that alerts fire when expected and do not fire for normal behavior.

---

# 8. Dashboards as Code

Grafana dashboards should not only exist manually in the UI.

Mature teams manage dashboards as code.

## 8.1 Options

```text
Grafana provisioning YAML
Grafana dashboard JSON
Grafana API
Terraform Grafana provider
Jsonnet / Grafonnet
Helm chart provisioning
GitOps with ConfigMaps
```

## 8.2 Benefits

| Benefit | Explanation |
|---|---|
| Version control | Every dashboard change is traceable |
| Review process | Dashboards can be reviewed like code |
| Rollback | Bad dashboard changes can be reverted |
| Reproducibility | Dashboards can be rebuilt in another cluster |
| Standardization | Teams can share templates |
| Auditability | Useful in regulated or enterprise environments |

## 8.3 Example provisioning structure

```text
grafana/
  dashboards/
    kubernetes-platform-health.json
    service-slo-overview.json
    node-health.json
  provisioning/
    dashboards.yaml
    datasources.yaml
```

## 8.4 Example dashboard provider config

```yaml
apiVersion: 1

providers:
  - name: "platform-dashboards"
    orgId: 1
    folder: "Platform"
    type: file
    disableDeletion: false
    updateIntervalSeconds: 30
    options:
      path: /var/lib/grafana/dashboards/platform
```

## Interview answer

> For production, I prefer dashboards as code. Grafana dashboards should be exported, versioned, reviewed, and provisioned through GitOps or Terraform. This prevents manual drift and allows dashboards to be recreated across environments.

---

# 9. Grafana Datasources as Code

Datasources should also be provisioned.

Example:

```yaml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus-operated.monitoring.svc:9090
    isDefault: true
```

For Loki:

```yaml
apiVersion: 1

datasources:
  - name: Loki
    type: loki
    access: proxy
    url: http://loki-gateway.logging.svc.cluster.local
```

## Interview answer

> Datasources should be provisioned as code so Grafana can be rebuilt without manual configuration. This is important for GitOps, disaster recovery, consistency, and auditability.

---

# 10. GitOps for Observability

Observability should be deployed the same way as applications.

## 10.1 GitOps-managed observability components

```text
kube-prometheus-stack values
ServiceMonitor
PodMonitor
PrometheusRule
Alertmanager config
Grafana dashboards
Grafana datasources
Loki/Alloy configs
Blackbox probes
Runbooks
```

## 10.2 Example repo structure

```text
observability/
  prometheus/
    values.yaml
    rules/
      platform-alerts.yaml
      service-slo-rules.yaml
    servicemonitors/
      api-servicemonitor.yaml
      blackbox-servicemonitor.yaml
  grafana/
    dashboards/
      platform-health.json
      service-slo.json
    provisioning/
      datasources.yaml
      dashboards.yaml
  alertmanager/
    alertmanagerconfig.yaml
  runbooks/
    high-error-rate.md
    target-down.md
```

## 10.3 Senior answer

> I treat observability configuration as production code. Dashboards, alerts, scrape configs, and runbooks should be versioned, reviewed, tested, deployed through GitOps, and rolled back if needed.

---

# 11. Thanos Architecture

Thanos extends Prometheus for long-term storage, HA, and global querying.

## 11.1 Thanos components

| Component | Purpose |
|---|---|
| Sidecar | Runs next to Prometheus, uploads blocks and exposes StoreAPI |
| Store Gateway | Reads historical blocks from object storage |
| Querier | Queries Prometheus sidecars and store gateways |
| Query Frontend | Caches and optimizes queries |
| Compactor | Compacts blocks and applies retention |
| Ruler | Evaluates recording/alerting rules against long-term data |
| Receive | Ingests remote write data |

## 11.2 Simplified architecture

```text
Prometheus + Thanos Sidecar
        ↓ upload blocks
Object Storage
        ↓
Thanos Store Gateway
        ↓
Thanos Querier
        ↓
Grafana
```

## 11.3 Why Thanos?

```text
Long-term retention
Global query view
Multi-cluster metrics
Prometheus HA deduplication
Object storage backend
```

## Interview answer

> Thanos keeps Prometheus as the local scraper but adds long-term object storage and global querying. Sidecars upload Prometheus blocks to object storage, Store Gateway reads historical data, and Querier provides a global query layer with deduplication across HA Prometheus replicas.

---

# 12. Mimir and Cortex Architecture

Mimir and Cortex are horizontally scalable metrics backends.

They are commonly used for:

```text
Centralized metrics
Multi-tenant metrics
Large-scale remote write
Long retention
High-volume query workloads
```

## 12.1 Common architecture concepts

| Component type | Purpose |
|---|---|
| Distributor | Receives remote write |
| Ingester | Stores recent samples |
| Querier | Executes queries |
| Query frontend | Caches/splits queries |
| Store gateway | Reads blocks from object storage |
| Compactor | Compacts blocks |
| Ruler | Evaluates rules |
| Alertmanager | Multi-tenant alert management |

## 12.2 Thanos vs Mimir/Cortex simple difference

| Area | Thanos | Mimir/Cortex |
|---|---|---|
| Model | Extends Prometheus | Centralized scalable metrics backend |
| Ingestion | Sidecar block upload or receive | Remote write |
| Best fit | Prometheus HA + global query + object storage | Large-scale multi-tenant metrics platform |
| Complexity | Medium | Higher |
| Multi-tenancy | Possible | Stronger design focus |

## Interview answer

> Thanos is often used to extend existing Prometheus deployments with long-term storage and global queries. Mimir and Cortex are more centralized, horizontally scalable, multi-tenant metrics platforms usually fed by remote write.

---

# 13. Remote Write Design

Remote write sends metrics from Prometheus to another backend.

```text
Prometheus
    ↓ remote_write
Mimir / Cortex / Thanos Receive / VictoriaMetrics / Grafana Cloud
```

## 13.1 Why remote write?

```text
Long-term retention
Centralized querying
Multi-cluster aggregation
Disaster recovery
Compliance retention
Reduced local Prometheus retention
```

## 13.2 Risks

```text
High cost from high cardinality
Network bottlenecks
Backpressure
Queue failures
Duplicate samples
Tenant label mistakes
Large remote-write bills
```

## 13.3 Remote write queue metrics

Useful metrics include:

```promql
prometheus_remote_storage_samples_in_total
prometheus_remote_storage_samples_failed_total
prometheus_remote_storage_samples_retried_total
prometheus_remote_storage_pending_samples
prometheus_remote_storage_shards
```

## 13.4 Alert idea: remote write failures

```promql
rate(prometheus_remote_storage_samples_failed_total[5m]) > 0
```

## Senior answer

> Remote write is powerful but can become expensive if cardinality is uncontrolled. I would add cluster, environment, and tenant labels carefully, monitor queue health and failed samples, and avoid sending unnecessary high-cardinality metrics to long-term storage.

---

# 14. Exemplars

Exemplars connect metrics to traces.

Example use case:

```text
A p95 latency spike appears in Grafana.
Click exemplar.
Jump to the trace that caused the spike.
```

## 14.1 Why exemplars matter

They help connect:

```text
Metric symptom → Trace investigation
```

## 14.2 Common requirements

```text
Application instrumentation supports exemplars
Tracing backend exists
Trace ID is attached to metrics
Grafana datasource supports exemplar display
```

## Interview answer

> Exemplars are useful because they link aggregated metrics to individual traces. Instead of only seeing a latency spike, I can jump from the metric to a trace sample and investigate the request path.

---

# 15. OpenTelemetry

OpenTelemetry is a vendor-neutral observability framework.

It covers:

```text
Metrics
Logs
Traces
Instrumentation
Collection
Exporting
```

## 15.1 Basic OpenTelemetry architecture

```text
Application instrumentation
        ↓
OpenTelemetry SDK / Agent
        ↓
OpenTelemetry Collector
        ↓
Backends: Prometheus / Tempo / Jaeger / Loki / Mimir / vendor platform
```

## 15.2 OpenTelemetry Collector

The Collector can receive, process, and export telemetry.

Pipeline model:

```text
Receivers → Processors → Exporters
```

Example:

```text
OTLP receiver → batch processor → Prometheus exporter
OTLP receiver → memory_limiter → OTLP exporter
```

## 15.3 OpenTelemetry vs Prometheus

| Area | Prometheus | OpenTelemetry |
|---|---|---|
| Main strength | Metrics scraping and PromQL | Vendor-neutral telemetry pipeline |
| Collection model | Pull by default | Push/pull depending on pipeline |
| Data types | Metrics mainly | Metrics, logs, traces |
| Query language | PromQL | Backend-dependent |
| Kubernetes use | Metrics monitoring | Unified telemetry collection |

## Interview answer

> Prometheus is excellent for scraping and querying metrics. OpenTelemetry is broader and provides vendor-neutral instrumentation and collection for metrics, logs, and traces. In modern systems, they often work together: OpenTelemetry collects telemetry and can export metrics to Prometheus-compatible backends.

---

# 16. Metrics, Logs, and Traces

Each signal answers a different question.

| Signal | Answers |
|---|---|
| Metrics | Is something wrong? How much? How often? |
| Logs | What happened? What error occurred? |
| Traces | Where did time go across services? |

## Example incident

```text
Metric:
5xx error rate increased.

Log:
Database timeout errors.

Trace:
Requests spend 2 seconds waiting on checkout-db query.
```

## Senior answer

> Metrics are best for alerting and trend detection, logs are best for detailed event context, and traces are best for understanding request flow across services. A mature observability platform connects all three.

---

# 17. Loki and Prometheus Relationship

Prometheus stores metrics.

Loki stores logs.

Grafana can query both.

```text
Prometheus → PromQL → Metrics panels and alerts
Loki → LogQL → Logs and log-derived panels
Grafana → Unified dashboard
```

## Example combined incident flow

```text
Prometheus alert: High 5xx error rate
Grafana dashboard: service error ratio
Loki query: logs for same service/pod
Trace backend: request path investigation
```

## Interview answer

> I use Prometheus for metrics and Loki for logs. In Grafana, I correlate them through labels like namespace, pod, service, and cluster. Metrics tell me where to look, logs explain what happened, and traces show the request path.

---

# 18. Multi-Cluster Dashboard Design

Multi-cluster observability needs stable labels.

Required labels:

```text
cluster
environment
region
namespace
service
team
```

## 18.1 Example query

```promql
sum by (cluster, namespace) (
  rate(container_cpu_usage_seconds_total{container!="POD", pod!=""}[5m])
)
```

## 18.2 Grafana variables

```text
$cluster
$environment
$namespace
$service
$team
```

## 18.3 Alert labels

Alerts should include:

```yaml
labels:
  severity: critical
  team: platform
  cluster: "{{ $labels.cluster }}"
  namespace: "{{ $labels.namespace }}"
```

## Senior answer

> Multi-cluster dashboards depend on consistent labeling. Without a stable cluster and environment label, global dashboards and alert routing become unreliable.

---

# 19. Observability Cost Control

Observability has cost.

Cost drivers:

```text
Active series
Sample rate
Retention
Remote write volume
Dashboard query load
Trace volume
Log ingestion volume
Object storage
High-cardinality labels
```

## 19.1 Cost control methods

```text
Cardinality budgets
Metric review process
Drop unnecessary metrics
Use route templates instead of full URLs
Longer scrape intervals for low-value targets
Recording rules for expensive queries
Retention tiers
Sampling traces
Log filtering
Dashboard refresh limits
```

## 19.2 Senior phrase

> Observability is not free. I treat metrics, logs, and traces as production data products with cost, ownership, retention, and quality requirements.

---

# 20. Security for Prometheus and Grafana

Monitoring systems often contain sensitive information.

## 20.1 Risks

```text
Metrics exposing internal hostnames
Labels exposing customer IDs or emails
Grafana admin credentials exposed
Anonymous dashboard access
Prometheus exposed publicly
Alertmanager webhook secrets leaked
Datasource credentials in plain text
No RBAC
No TLS
No audit trail
```

## 20.2 Controls

```text
TLS for endpoints
Authentication for Grafana
RBAC and folder permissions
NetworkPolicy for Prometheus/Grafana
No public Prometheus access
Secret management for datasources
Avoid sensitive labels
Audit dashboard changes
Restrict admin privileges
Use read-only service accounts where possible
```

## Interview answer

> Observability data can leak sensitive operational and customer information. I protect Prometheus and Grafana with authentication, RBAC, network restrictions, TLS, secret management, and careful metric-label design.

---

# 21. Blackbox + Whitebox Monitoring

## Whitebox monitoring

Internal metrics from application/exporters.

Examples:

```text
Request rate
Error rate
Latency
CPU
Memory
Queue size
Database pool usage
```

## Blackbox monitoring

External probing from user perspective.

Examples:

```text
Can DNS resolve?
Can HTTPS connect?
Is certificate valid?
Does endpoint return 200?
Is latency acceptable?
```

## Senior answer

> Whitebox monitoring tells me what the system thinks internally. Blackbox monitoring tells me what users experience externally. Production monitoring needs both.

---

# 22. SLO Alerting: Multi-Window Burn Rate

Single threshold alerts are often noisy.

SLO burn-rate alerts are more mature.

## 22.1 Simple error ratio

```promql
sum(rate(http_requests_total{status=~"5.."}[5m]))
/
sum(rate(http_requests_total[5m]))
```

## 22.2 Fast burn example

```promql
(
  sum(rate(http_requests_total{status=~"5.."}[5m]))
  /
  sum(rate(http_requests_total[5m]))
) > 0.05
```

## 22.3 Slow burn example

```promql
(
  sum(rate(http_requests_total{status=~"5.."}[1h]))
  /
  sum(rate(http_requests_total[1h]))
) > 0.02
```

## 22.4 Concept

```text
Fast window catches severe incidents quickly.
Slow window confirms sustained user impact.
```

## Senior answer

> I prefer multi-window burn-rate alerts for SLOs because they balance fast detection and noise reduction. A short window detects urgent incidents, while a longer window prevents alerting on harmless spikes.

---

# 23. Runbook-Linked Alerts

Every important alert should have a runbook.

## 23.1 Good runbook structure

```text
Alert name
Meaning
Impact
Likely causes
Immediate checks
PromQL queries
kubectl commands
Logs to inspect
Rollback/remediation steps
Escalation path
Post-incident notes
```

## 23.2 Example annotation

```yaml
annotations:
  runbook_url: "https://github.com/YOUR_ORG/YOUR_REPO/blob/main/runbooks/high-5xx-error-rate.md"
```

## Senior answer

> A runbook turns an alert into an operational workflow. Without a runbook, the alert may detect a problem but still leave the responder guessing.

---

# 24. Alert Routing Design

Alert labels should support routing.

Common labels:

```text
severity
team
service
namespace
cluster
environment
signal
```

Example:

```yaml
labels:
  severity: critical
  team: payments
  service: checkout-api
  signal: availability
```

## Routing examples

```text
severity=critical → pager
team=platform → platform Slack
team=security → security Slack
environment=dev → low-priority channel
```

## Senior answer

> Alert labels are not decoration. They drive routing, ownership, grouping, and escalation. I design alert labels so Alertmanager can route alerts without manual interpretation.

---

# 25. Common Senior Interview Scenarios

## Scenario 1: Grafana dashboard is slow

Answer:

```text
I would check the PromQL query complexity, time range, number of panels, refresh interval, high-cardinality groupings, regex matchers, and whether expensive queries should become recording rules. I would also check Prometheus query duration metrics and active series.
```

Useful metrics:

```promql
prometheus_engine_query_duration_seconds
prometheus_tsdb_head_series
rate(prometheus_tsdb_head_samples_appended_total[5m])
```

---

## Scenario 2: Prometheus memory is growing

Answer:

```text
I would suspect increasing active series or scrape volume. I would check active series, scrape samples per job, recent instrumentation changes, high-cardinality labels, new ServiceMonitors, and whether a service started exposing unbounded labels.
```

Useful queries:

```promql
prometheus_tsdb_head_series
```

```promql
topk(
  20,
  sum by (job) (
    rate(scrape_samples_scraped[5m])
  )
)
```

---

## Scenario 3: Alert storm after node failure

Answer:

```text
I would use Alertmanager inhibition. If NodeDown is firing, lower-level PodDown or TargetDown alerts from the same node can be suppressed to reduce symptom noise. I would also ensure alerts are grouped by cluster, node, namespace, and alertname where appropriate.
```

---

## Scenario 4: Metrics disappeared after deployment

Answer:

```text
I would check whether the target is still discovered, whether the ServiceMonitor selector still matches, whether the metrics port name changed, whether /metrics still works, whether labels changed, whether metric relabeling dropped the metric, and whether the application changed metric names.
```

---

## Scenario 5: Multi-cluster dashboards show wrong data

Answer:

```text
I would check whether every cluster injects a stable cluster label, whether remote-write labels are consistent, whether dashboard variables filter correctly, and whether duplicate HA Prometheus samples are deduplicated.
```

---

## Scenario 6: Remote write backend cost exploded

Answer:

```text
I would check cardinality growth, samples per second, high-cardinality labels, scrape interval changes, new exporters, and whether unnecessary metrics are being remote-written. I would add metric relabeling, cardinality budgets, and retention tiers.
```

---

## Scenario 7: Alert did not fire during incident

Answer:

```text
I would check whether the PromQL expression returned data during the incident, whether the threshold was too high, whether the for duration delayed firing, whether the rule was loaded, whether rule evaluation was healthy, whether Alertmanager silenced or inhibited it, and whether routing failed.
```

---

## Scenario 8: Users cannot reach service, but pods look healthy

Answer:

```text
I would check blackbox probes, ingress metrics, DNS, TLS certificate validity, load balancer health, service endpoints, network policies, and application logs. Pod health alone is not user-path health.
```

---

# 26. Git Commit Strategy

If you want this in Git, use a clear structure.

## 26.1 Recommended file path

```text
docs/observability/prometheus-grafana-cheatsheet-v4.md
```

## 26.2 Recommended commit message

```bash
git add docs/observability/prometheus-grafana-cheatsheet-v4.md
git commit -m "docs: add advanced Prometheus and Grafana cheat sheet v4"
git push
```

## 26.3 Optional README link

Add this to your main README:

```markdown
## Observability Notes

- [Prometheus & Grafana Advanced Production Cheat Sheet v4](docs/observability/prometheus-grafana-cheatsheet-v4.md)
```

---

# 27. Gradual Study Plan

## Day 1

Read:

```text
TSDB internals
Prometheus scaling limits
Cardinality budgeting
```

Practice answering:

```text
Why does Prometheus memory grow?
What is high cardinality?
What is WAL?
```

## Day 2

Read:

```text
Recording rules
Alert maturity
promtool
Runbook-linked alerts
```

Practice answering:

```text
How do you test alerts?
Why use recording rules?
What makes an alert good?
```

## Day 3

Read:

```text
Dashboards as code
Grafana datasources as code
GitOps for observability
```

Practice answering:

```text
How do you manage Grafana dashboards in production?
How do you avoid dashboard drift?
```

## Day 4

Read:

```text
Thanos
Mimir/Cortex
Remote write
Prometheus HA
```

Practice answering:

```text
How do you scale Prometheus across clusters?
What is the difference between Thanos and Mimir?
```

## Day 5

Read:

```text
OpenTelemetry
Exemplars
Metrics/logs/traces
Loki relationship
```

Practice answering:

```text
How do metrics, logs, and traces work together?
What are exemplars?
How does OTel fit with Prometheus?
```

---

# 28. Final Senior Summary

Use this as your senior interview answer:

> In production, I treat Prometheus and Grafana as an observability platform, not just a dashboard stack. Prometheus scrapes metrics, stores time series locally, evaluates rules, and sends alerts to Alertmanager. Grafana visualizes metrics and helps teams investigate. For Kubernetes, I manage scraping through ServiceMonitors, PodMonitors, and PrometheusRules using GitOps. At scale, I watch active series, scrape volume, cardinality, query cost, remote-write health, and rule evaluation duration. For long-term retention or multi-cluster visibility, I would use Thanos, Mimir, Cortex, VictoriaMetrics, or a managed backend. For alerting, I prefer SLO-based, runbook-linked, routed alerts with inhibition and grouping. For production maturity, dashboards, alerts, datasources, and runbooks should all be versioned and reproducible from Git.

---

# 29. Final Rule

The senior-level observability chain is:

```text
Instrumentation quality
→ scrape reliability
→ label/cardinality control
→ efficient PromQL
→ recording rules
→ useful dashboards
→ high-quality alerts
→ Alertmanager routing
→ runbooks
→ incident learning
→ platform improvement
```

If you can explain this chain clearly, you can handle most Prometheus/Grafana interview discussions.

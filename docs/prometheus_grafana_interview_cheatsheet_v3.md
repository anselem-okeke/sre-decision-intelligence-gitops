# Prometheus & Grafana Cheat Sheet v3

**Focus:** Production Prometheus/Grafana knowledge for DevOps, SRE, Platform Engineering, Kubernetes, and observability interviews.

This v3 extends:

- v1: Metric types and basic PromQL
- v2: PromQL ranking, aggregation, vector matching, Grafana dashboards, alerting, SLI/SLO basics
- v3: Production architecture, scraping, exporters, relabeling, cardinality, staleness, missing metrics, Alertmanager, Prometheus Operator, HA, remote write, troubleshooting, and senior interview scenarios

---

# 1. The Full Prometheus Mental Model

Prometheus is not only a query language.

It is a complete metrics collection and alerting system.

```text
Applications / Exporters
        ↓
/metrics endpoint
        ↓
Prometheus scrape engine
        ↓
Time series database
        ↓
PromQL queries
        ↓
Grafana dashboards / recording rules / alerting rules
        ↓
Alertmanager / notification routing
```

## Key components

| Component | Purpose |
|---|---|
| Prometheus server | Scrapes metrics, stores time series, evaluates rules |
| Exporter | Converts system/application data into Prometheus metrics |
| Pushgateway | Allows short-lived jobs to push metrics |
| Alertmanager | Groups, deduplicates, routes, silences, and inhibits alerts |
| Grafana | Visualizes metrics and can also manage alerting |
| Prometheus Operator | Manages Prometheus in Kubernetes using CRDs |
| ServiceMonitor | Defines how to scrape Kubernetes Services |
| PodMonitor | Defines how to scrape Pods directly |
| PrometheusRule | Defines alerting and recording rules in Kubernetes |
| Remote write | Sends metrics to long-term or centralized storage |

---

# 2. Prometheus Pull Model

Prometheus usually uses a pull model.

That means Prometheus reaches out to targets and scrapes their `/metrics` endpoint.

```text
Prometheus
    ↓ scrape HTTP
Target /metrics
```

Example target:

```text
http://my-app:8080/metrics
```

## Why pull matters

| Benefit | Explanation |
|---|---|
| Simpler health model | If Prometheus cannot scrape a target, it knows the target is unavailable |
| Central control | Scrape interval, timeout, labels, and relabeling are controlled centrally |
| Kubernetes friendly | Targets can be discovered through service discovery |
| Debuggable | You can manually open `/metrics` or check Prometheus Targets page |

## Interview answer

> Prometheus mostly follows a pull model. It discovers targets, scrapes their `/metrics` endpoints at a configured interval, stores the samples as time series, and then uses PromQL for dashboards, alerts, and recording rules. If a target cannot be scraped, the `up` metric becomes 0, which is one of the first troubleshooting signals.

---

# 3. Jobs, Instances, Targets, and Labels

These are core Prometheus concepts.

| Term | Meaning |
|---|---|
| Job | Group of similar scrape targets |
| Instance | A specific target endpoint |
| Target | The actual scrape endpoint |
| Label | Key-value dimension attached to a time series |

Example:

```text
job="node-exporter"
instance="10.0.1.15:9100"
```

## Important metric: up

```promql
up
```

Meaning:

```text
1 = scrape succeeded
0 = scrape failed
```

### Targets down

```promql
up == 0
```

### Targets down by job

```promql
sum by (job) (
  up == 0
)
```

### Scrape success ratio

```promql
sum(up)
/
count(up)
```

---

# 4. Scrape Configuration

A scrape config tells Prometheus what to scrape and how.

Basic static scrape config:

```yaml
scrape_configs:
  - job_name: "my-app"
    scrape_interval: 15s
    scrape_timeout: 10s
    static_configs:
      - targets:
          - "my-app:8080"
```

## Important fields

| Field | Meaning |
|---|---|
| `job_name` | Logical group name |
| `scrape_interval` | How often to scrape |
| `scrape_timeout` | Max time allowed for scrape |
| `metrics_path` | Usually `/metrics` |
| `scheme` | `http` or `https` |
| `static_configs` | Manually listed targets |
| `relabel_configs` | Modify/drop targets before scraping |
| `metric_relabel_configs` | Modify/drop metrics after scraping |

## Interview answer

> A scrape config defines which targets Prometheus scrapes, how often it scrapes them, and how labels are attached or transformed. In Kubernetes, targets are often discovered dynamically, while in smaller environments static configs may be enough.

---

# 5. Scrape Interval, Scrape Timeout, and Evaluation Interval

These are often confused.

| Setting | Meaning |
|---|---|
| `scrape_interval` | How often Prometheus collects metrics |
| `scrape_timeout` | How long Prometheus waits before marking scrape failed |
| `evaluation_interval` | How often rules are evaluated |

Example:

```yaml
global:
  scrape_interval: 15s
  scrape_timeout: 10s
  evaluation_interval: 30s
```

## Important rule

```text
scrape_timeout must be less than or equal to scrape_interval.
```

## Interview answer

> Scrape interval controls data collection frequency. Scrape timeout controls how long Prometheus waits for a target response. Evaluation interval controls how often alerting and recording rules run. If the scrape interval is too short, Prometheus load increases. If it is too long, detection becomes slower.

---

# 6. Exporters

Exporters expose metrics for systems that do not natively expose Prometheus metrics.

## Common exporters

| Exporter | Purpose |
|---|---|
| node-exporter | Linux host metrics |
| windows-exporter | Windows host metrics |
| blackbox-exporter | HTTP/TCP/ICMP probing |
| kube-state-metrics | Kubernetes object state |
| cAdvisor | Container resource metrics |
| postgres-exporter | PostgreSQL metrics |
| mysqld-exporter | MySQL/MariaDB metrics |
| nginx-exporter | Nginx metrics |
| redis-exporter | Redis metrics |
| snmp-exporter | Network device metrics |

## Exporter mental model

```text
System / Service
    ↓ native stats/API/files
Exporter
    ↓ exposes /metrics
Prometheus
```

## Example

Node exporter reads Linux host data and exposes metrics like:

```text
node_cpu_seconds_total
node_memory_MemAvailable_bytes
node_filesystem_avail_bytes
node_network_receive_bytes_total
```

## Interview answer

> Exporters bridge systems into the Prometheus ecosystem. If an application does not expose Prometheus metrics directly, an exporter reads the system’s native metrics and exposes them in Prometheus format.

---

# 7. Kubernetes Metrics Sources

In Kubernetes, not all metrics come from the same place.

| Source | Provides |
|---|---|
| cAdvisor / kubelet | Container CPU, memory, filesystem, network |
| kube-state-metrics | Kubernetes object state: deployments, pods, replicas, labels |
| node-exporter | Node OS metrics |
| application metrics | Request rate, error rate, latency, business metrics |
| API server metrics | Kubernetes API latency, requests, errors |
| etcd metrics | etcd latency, leader changes, DB size |
| blackbox exporter | External availability probing |

## Important distinction

### cAdvisor metrics answer:

```text
How much resource is the container using?
```

Examples:

```promql
container_cpu_usage_seconds_total
container_memory_working_set_bytes
container_network_receive_bytes_total
```

### kube-state-metrics answers:

```text
What is the desired/current Kubernetes object state?
```

Examples:

```promql
kube_deployment_spec_replicas
kube_deployment_status_replicas_available
kube_pod_status_phase
kube_pod_container_status_waiting_reason
```

## Interview answer

> For Kubernetes troubleshooting, I separate usage metrics from object-state metrics. cAdvisor tells me actual resource usage, while kube-state-metrics tells me desired state, pod status, replica status, waiting reasons, and Kubernetes object metadata.

---

# 8. Service Discovery in Kubernetes

Prometheus needs to discover what to scrape.

In Kubernetes, this can happen through:

```text
ServiceMonitor
PodMonitor
ScrapeConfig
annotations
native Kubernetes service discovery
```

## ServiceMonitor

Used by Prometheus Operator to scrape Kubernetes Services.

Example:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: my-app
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: my-app
  namespaceSelector:
    matchNames:
      - production
  endpoints:
    - port: metrics
      path: /metrics
      interval: 15s
```

## PodMonitor

Used when you want to scrape pods directly.

Example:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: my-app-pods
spec:
  selector:
    matchLabels:
      app: my-app
  podMetricsEndpoints:
    - port: metrics
      path: /metrics
```

## ServiceMonitor vs PodMonitor

| Object | Use when |
|---|---|
| ServiceMonitor | You expose metrics through a Kubernetes Service |
| PodMonitor | You want to scrape pods directly |
| ScrapeConfig | You need advanced scrape configuration |
| Annotation scraping | Simple setups, less controlled |

## Interview answer

> In Kubernetes, I prefer ServiceMonitor or PodMonitor with Prometheus Operator because the scrape configuration becomes Kubernetes-native and GitOps-friendly. ServiceMonitor is usually better when metrics are exposed through a Service, while PodMonitor is useful for scraping pods directly.

---

# 9. Relabeling

Relabeling modifies target labels before scraping.

Metric relabeling modifies metric labels after scraping.

## 9.1 relabel_configs

Used before scraping.

Common uses:

- Keep only selected targets
- Drop unwanted targets
- Rewrite target address
- Add environment/team labels
- Map Kubernetes metadata into Prometheus labels

Example: keep only pods with scrape annotation

```yaml
relabel_configs:
  - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
    action: keep
    regex: true
```

## 9.2 metric_relabel_configs

Used after scraping.

Common uses:

- Drop high-cardinality metrics
- Drop noisy labels
- Rename labels
- Keep only selected metric names

Example: drop high-cardinality label

```yaml
metric_relabel_configs:
  - action: labeldrop
    regex: "pod_uid|container_id"
```

Example: drop expensive metrics

```yaml
metric_relabel_configs:
  - source_labels: [__name__]
    regex: "go_memstats_.*"
    action: drop
```

## Important difference

| Type | Happens | Purpose |
|---|---|---|
| `relabel_configs` | Before scrape | Select/modify targets |
| `metric_relabel_configs` | After scrape | Select/modify scraped metrics |

## Interview answer

> Relabeling is used to control target discovery and labeling before scraping. Metric relabeling is used after scraping to reduce or reshape metrics. I use relabeling to keep or drop targets, attach metadata, and control cardinality before it becomes expensive.

---

# 10. Cardinality

Cardinality is one of the most important production topics.

## What is cardinality?

A Prometheus time series is identified by:

```text
metric name + full label set
```

Example:

```text
http_requests_total{method="GET", route="/api/users", status="200"}
```

Every unique label combination creates a new time series.

## High-cardinality danger

Bad labels:

```text
user_id
request_id
session_id
email
ip_address
uuid
timestamp
full_url_with_id
```

Bad metric example:

```text
http_requests_total{user_id="123456", request_id="abc-xyz"}
```

Problem:

```text
Every user/request creates a new time series.
Prometheus memory, CPU, and storage usage can explode.
```

## Good label example

```text
http_requests_total{method="GET", route="/api/users/:id", status="200"}
```

## Cardinality troubleshooting queries

### Count time series by metric name

```promql
topk(
  20,
  count by (__name__) (
    {__name__=~".+"}
  )
)
```

### Count series by job

```promql
topk(
  20,
  count by (job) (
    {__name__=~".+"}
  )
)
```

### Count series by label value

```promql
topk(
  20,
  count by (namespace) (
    {__name__=~".+"}
  )
)
```

## Interview answer

> Cardinality is the number of unique time series created by metric names and label combinations. High-cardinality labels like user ID, request ID, session ID, or full dynamic URLs can overload Prometheus. I avoid unbounded labels and prefer stable dimensions like service, route template, method, status, namespace, pod, and node.

---

# 11. Staleness and Missing Metrics

Prometheus has a concept of stale series.

A metric can disappear because:

- Target is down
- Exporter stopped exposing the metric
- Pod was deleted
- Label changed
- Scrape failed
- Service discovery stopped finding the target
- Network/TLS/auth issue prevents scraping

## 11.1 Detect target down

```promql
up == 0
```

## 11.2 Detect missing target

```promql
absent(up{job="my-app"})
```

Meaning:

```text
Return a result if the series does not exist.
```

## 11.3 Detect missing metric

```promql
absent(http_requests_total{job="my-app"})
```

## 11.4 Detect no traffic

```promql
sum(rate(http_requests_total{job="my-app"}[5m])) == 0
```

## Difference: target down vs no traffic

| Situation | Query |
|---|---|
| Target not scrapeable | `up == 0` |
| Metric missing | `absent(metric)` |
| App has no traffic | `rate(requests_total[5m]) == 0` |
| Pod deleted | Kubernetes object metrics disappear |
| Exporter broken | `up` may be 0 or metrics incomplete |

## Interview answer

> Missing metrics are different from zero values. A zero value means Prometheus scraped the metric and the value is zero. A missing metric means the series does not exist or disappeared. I use `up`, `absent()`, target health, and scrape error metrics to distinguish target failure, exporter failure, deleted pods, and true zero activity.

---

# 12. Logical Set Operators

PromQL supports set logic.

| Operator | Meaning |
|---|---|
| `and` | Keep series present on both sides |
| `or` | Union of series |
| `unless` | Left side unless matching right side exists |

## 12.1 and

Example:

```promql
kube_pod_status_phase{phase="Running"} == 1
and
kube_pod_container_status_ready == 0
```

Meaning:

```text
Pods that are running but containers are not ready.
```

## 12.2 or

Example:

```promql
kube_pod_container_status_waiting_reason{reason="CrashLoopBackOff"} == 1
or
kube_pod_container_status_waiting_reason{reason="ImagePullBackOff"} == 1
```

Meaning:

```text
Show either CrashLoopBackOff or ImagePullBackOff.
```

## 12.3 unless

Example:

```promql
kube_pod_info
unless
kube_pod_status_phase{phase="Running"} == 1
```

Meaning:

```text
Pods that exist but are not running.
```

---

# 13. Useful PromQL Functions Beyond the Basics

## 13.1 irate()

`irate()` calculates an instant rate using the last two samples.

```promql
irate(container_cpu_usage_seconds_total[5m])
```

Use for:

- Fast-changing graphs
- Short-term visualization

Avoid for:

- Stable alerting
- Slow-moving SLO calculations

Interview answer:

> I prefer `rate()` for alerts because it is smoother. I may use `irate()` for fast dashboard visualization, but it can be noisy.

---

## 13.2 resets()

Counts counter resets.

```promql
resets(http_requests_total[1h])
```

Use for detecting:

- Process restarts
- Counter resets
- Exporter restarts

---

## 13.3 changes()

Counts how many times a value changed.

```promql
changes(kube_pod_container_status_restarts_total[1h])
```

Use for:

- State flapping
- Frequent changes
- Status instability

---

## 13.4 absent()

Detects missing series.

```promql
absent(up{job="my-app"})
```

---

## 13.5 predict_linear()

Predicts future values based on past trend.

Example: disk may fill within 24 hours.

```promql
predict_linear(node_filesystem_avail_bytes[6h], 24 * 3600) < 0
```

Interview answer:

> `predict_linear()` can be used for capacity forecasting, such as predicting disk exhaustion based on recent filesystem trend.

---

## 13.6 label_replace()

Creates or modifies labels using regex.

Example:

```promql
label_replace(
  up,
  "short_instance",
  "$1",
  "instance",
  "([^:]+):.*"
)
```

Meaning:

```text
Extract hostname from instance label.
```

Use carefully because label manipulation can make queries harder to maintain.

---

## 13.7 clamp_min() and clamp_max()

Useful to avoid negative or extreme values.

```promql
clamp_min(expression, 0)
```

Example:

```promql
clamp_min(
  kube_deployment_spec_replicas - kube_deployment_status_replicas_available,
  0
)
```

---

# 14. Subqueries

Subqueries let you query a query over time.

Syntax:

```promql
expression[range:step]
```

Example:

```promql
max_over_time(
  rate(http_requests_total[5m])[1h:5m]
)
```

Meaning:

```text
Maximum 5-minute request rate observed over the last hour.
```

## Common over-time functions

| Function | Meaning |
|---|---|
| `avg_over_time()` | Average over range |
| `max_over_time()` | Maximum over range |
| `min_over_time()` | Minimum over range |
| `sum_over_time()` | Sum values over range |
| `count_over_time()` | Count samples over range |
| `quantile_over_time()` | Quantile over range |

Example: max memory over the last hour

```promql
max_over_time(
  container_memory_working_set_bytes{container!="POD", pod!=""}[1h]
)
```

---

# 15. Native Histograms vs Classic Histograms

Traditional Prometheus histograms expose:

```text
*_bucket
*_sum
*_count
```

Classic query:

```promql
histogram_quantile(
  0.95,
  sum by (le) (
    rate(http_request_duration_seconds_bucket[5m])
  )
)
```

Native histograms are a newer histogram type that can store bucket information more efficiently and with different query patterns depending on instrumentation and Prometheus version.

## Interview-safe answer

> Most Kubernetes and application setups still commonly expose classic histogram buckets such as `*_bucket`, and I query them with `histogram_quantile()` while preserving the `le` label. I am also aware of native histograms, but I would first check what the application actually exposes before choosing the query style.

---

# 16. Alertmanager

Prometheus evaluates alert rules.

Alertmanager handles what happens after alerts fire.

```text
Prometheus alert rule fires
        ↓
Alertmanager
        ↓
grouping / deduplication / inhibition / silencing / routing
        ↓
Slack / email / PagerDuty / Opsgenie / webhook
```

## Alertmanager concepts

| Concept | Meaning |
|---|---|
| Grouping | Combine related alerts |
| Deduplication | Avoid repeated duplicate notifications |
| Routing | Send alerts to the right team/channel |
| Silencing | Temporarily mute alerts |
| Inhibition | Suppress lower-priority alerts when higher-level alert exists |
| Receiver | Notification destination |

## Example routing logic

```yaml
route:
  receiver: default
  group_by: ["alertname", "namespace"]
  routes:
    - matchers:
        - severity="critical"
      receiver: pagerduty
    - matchers:
        - team="platform"
      receiver: platform-slack
```

## Example inhibition

```text
If NodeDown is firing, suppress PodDown alerts from that node.
```

Why?

```text
The pod alerts are symptoms. The node alert is the root-level signal.
```

## Interview answer

> Prometheus detects alert conditions, but Alertmanager handles notification logic. It groups related alerts, deduplicates repeated alerts, routes them to the correct receiver, supports silences for maintenance, and uses inhibition to reduce noise when a higher-level alert explains lower-level symptoms.

---

# 17. Grafana Alerting vs Prometheus Alerting

Both can alert, but they are different operational models.

| Area | Prometheus Alerting | Grafana Alerting |
|---|---|---|
| Rule location | Prometheus rule files / PrometheusRule CRDs | Grafana-managed alert rules |
| Best for | Metrics-native, GitOps, SRE rules | Multi-data-source alerting |
| Routing | Alertmanager | Grafana contact points/policies |
| GitOps friendliness | Strong with Prometheus Operator | Possible, but depends on provisioning |
| Common use | Kubernetes/platform alerts | Dashboards, mixed-source alerts |

## Interview answer

> For core Kubernetes and platform alerts, I prefer Prometheus rules managed through GitOps and routed through Alertmanager. Grafana alerting is useful when alerts need to combine multiple data sources or when the organization standardizes alerting inside Grafana.

---

# 18. Prometheus Operator Concepts

In Kubernetes, many teams deploy Prometheus through kube-prometheus-stack.

Important CRDs:

| CRD | Purpose |
|---|---|
| Prometheus | Defines Prometheus instances |
| Alertmanager | Defines Alertmanager instances |
| ServiceMonitor | Scrapes services |
| PodMonitor | Scrapes pods |
| PrometheusRule | Alerting and recording rules |
| Probe | Blackbox probing |
| ScrapeConfig | Advanced scrape configs |

## kube-prometheus-stack

Usually includes:

```text
Prometheus
Grafana
Alertmanager
node-exporter
kube-state-metrics
Prometheus Operator
default dashboards
default rules
```

## Interview answer

> In Kubernetes, Prometheus Operator turns monitoring configuration into Kubernetes resources. Instead of editing a raw Prometheus config manually, I can define ServiceMonitors, PodMonitors, PrometheusRules, and Alertmanager configuration in Git, then let the operator reconcile the desired monitoring state.

---

# 19. Remote Write and Long-Term Storage

Prometheus local storage is not always enough for long-term metrics.

Remote write sends samples to another backend.

Examples:

```text
Thanos
Cortex
Mimir
VictoriaMetrics
Grafana Cloud
```

## Why use remote write?

| Reason | Explanation |
|---|---|
| Long retention | Keep metrics beyond local retention |
| Centralized metrics | Combine many clusters |
| HA querying | Query across replicas/clusters |
| Cost optimization | Store old data in object storage |
| Global view | Multi-cluster / multi-region observability |

## Interview answer

> Prometheus is excellent for local scraping and alerting, but for long-term retention and multi-cluster visibility, I would use remote write or a system like Thanos, Cortex, Mimir, or VictoriaMetrics. This allows central querying, longer retention, and global dashboards.

---

# 20. Prometheus HA

Prometheus is commonly deployed in pairs for high availability.

```text
Prometheus A scrapes targets
Prometheus B scrapes same targets
Both send alerts / remote write
External system deduplicates
```

## Important concept

Prometheus HA often means:

```text
Two Prometheus replicas scrape the same targets.
Alertmanager deduplicates alerts.
Remote storage deduplicates samples if configured.
```

## Interview answer

> Prometheus itself is not horizontally sharded by default for the same scrape workload. A common HA pattern is two Prometheus replicas scraping the same targets, with Alertmanager deduplicating alerts and remote storage handling long-term deduplication.

---

# 21. Blackbox Exporter

Blackbox exporter checks systems from the outside.

It probes:

```text
HTTP
HTTPS
TCP
ICMP
DNS
```

## Why it matters

Application metrics may say the app is healthy internally, but users may still be unable to reach it.

Blackbox probes answer:

```text
Can the service be reached from the outside?
Is TLS valid?
Is HTTP returning 200?
Is DNS resolving?
Is TCP open?
```

## Example metrics

```promql
probe_success
probe_duration_seconds
probe_http_status_code
probe_ssl_earliest_cert_expiry
```

### HTTP availability

```promql
probe_success{job="blackbox-http"} == 0
```

### TLS certificate expiry in days

```promql
(probe_ssl_earliest_cert_expiry - time()) / 86400
```

## Interview answer

> Blackbox exporter gives outside-in monitoring. It is useful because internal pod metrics may look healthy while ingress, DNS, TLS, or network paths are broken. For user-facing services, I combine internal application metrics with blackbox probes.

---

# 22. RED, USE, and Four Golden Signals

These are observability frameworks.

## RED method

For request-driven services:

| Signal | Meaning |
|---|---|
| Rate | Requests per second |
| Errors | Failed request rate |
| Duration | Latency |

Example:

```text
Request rate
5xx error rate
p95/p99 latency
```

## USE method

For infrastructure resources:

| Signal | Meaning |
|---|---|
| Utilization | How busy the resource is |
| Saturation | How much queue/pressure exists |
| Errors | Resource errors |

Example:

```text
CPU utilization
Memory pressure
Disk saturation
Network errors
```

## Four Golden Signals

| Signal | Meaning |
|---|---|
| Latency | How long requests take |
| Traffic | Demand on the system |
| Errors | Failure rate |
| Saturation | How full/busy the system is |

## Interview answer

> For services I use RED: rate, errors, and duration. For infrastructure I use USE: utilization, saturation, and errors. For SRE service health, I map this to the golden signals: latency, traffic, errors, and saturation.

---

# 23. Dashboard Design Maturity

A weak dashboard shows many metrics.

A strong dashboard supports decisions.

## Bad dashboard

```text
Many random graphs
No thresholds
No ownership
No drilldown
No service impact
No units
No alert linkage
```

## Strong dashboard

```text
Starts with service health
Shows SLO/SLI impact
Separates symptoms from causes
Uses thresholds and units
Supports drilldown by namespace/pod/node
Highlights top offenders
Correlates deployments and incidents
```

## Good row structure

```text
1. Service Health / SLO Overview
2. Request Rate / Error Rate / Latency
3. Workload Health
4. Resource Saturation
5. Node Health
6. Storage / Network
7. Alert and Incident Context
8. Drilldown Tables
```

---

# 24. Prometheus Troubleshooting Checklist

## 24.1 Target missing

Check:

```promql
up{job="my-app"}
```

Then check:

```text
Prometheus Targets page
ServiceMonitor selector
Service labels
Namespace selector
Port name
NetworkPolicy
TLS/auth settings
Endpoint exists
Pod exposes /metrics
```

## 24.2 Metric missing

Check:

```text
Is target up?
Does /metrics expose the metric?
Was metric renamed?
Did labels change?
Is metric relabeling dropping it?
Is scrape failing?
Is application code path producing it?
```

## 24.3 Dashboard empty

Check:

```text
Wrong namespace variable
Wrong label name
Wrong data source
Time range too narrow
Metric does not exist
Panel transformation hides data
Query uses wrong label matcher
Prometheus scrape not working
```

## 24.4 Alert not firing

Check:

```text
Expression returns data?
Threshold correct?
for: duration too long?
Rule loaded?
PrometheusRule selected by Prometheus?
Alert state pending/firing?
Labels match route?
Alertmanager receiver configured?
Silence active?
Inhibition suppressing it?
```

## 24.5 Query slow

Check:

```text
High-cardinality labels
Too wide time range
Too many regex matchers
No recording rule
Too many dashboard panels
Query groups by unnecessary labels
Metric has too many series
```

---

# 25. Common Production Questions

## Q1. Prometheus target is down. What do you check?

Answer:

> I start with the `up` metric and Prometheus Targets page. Then I verify service discovery, ServiceMonitor or PodMonitor selectors, namespace selection, service labels, endpoint existence, metrics port name, `/metrics` availability, TLS/auth configuration, and any NetworkPolicy blocking Prometheus from scraping the target.

---

## Q2. Grafana panel shows no data. What do you check?

Answer:

> I check whether the metric exists in Prometheus, whether the selected time range contains data, whether dashboard variables are correct, whether label names match the actual series, whether the data source is correct, and whether transformations or filters are hiding the result.

---

## Q3. How do you reduce Prometheus cardinality?

Answer:

> I avoid unbounded labels like user ID, request ID, session ID, IP, and full URLs. I normalize routes, drop unnecessary labels with metric relabeling, aggregate with recording rules, review high-cardinality metrics, and keep dashboards grouped by operationally useful labels.

---

## Q4. Difference between ServiceMonitor and PodMonitor?

Answer:

> ServiceMonitor scrapes metrics through a Kubernetes Service. PodMonitor scrapes pods directly. I use ServiceMonitor when the workload exposes metrics through a stable Service and PodMonitor when direct pod scraping is more appropriate.

---

## Q5. Difference between relabel_configs and metric_relabel_configs?

Answer:

> `relabel_configs` runs before scraping and is used to select or modify targets. `metric_relabel_configs` runs after scraping and is used to drop or modify scraped metrics and labels, often for cardinality control.

---

## Q6. What is the purpose of Alertmanager?

Answer:

> Alertmanager receives alerts from Prometheus and handles grouping, deduplication, routing, silencing, and inhibition. It reduces alert noise and ensures the right team receives the right alert.

---

## Q7. What is the difference between zero and absent?

Answer:

> Zero means the metric exists and its value is zero. Absent means the time series does not exist or disappeared. For absence I use `absent()`, and for scrape failure I check `up`.

---

## Q8. Why use recording rules?

Answer:

> Recording rules precompute expensive or repeated PromQL expressions. They improve dashboard performance, reduce query load, and standardize SLI/SLO and alert logic.

---

## Q9. How would you monitor a Kubernetes application?

Answer:

> I would monitor application RED metrics: request rate, error rate, and latency. Then Kubernetes workload health: restarts, CrashLoopBackOff, Pending pods, missing replicas, and resource usage. Then infrastructure USE metrics: node CPU, memory, filesystem, network, and saturation. I would add blackbox probes for external reachability and alerts tied to user impact.

---

## Q10. How do you handle multi-cluster metrics?

Answer:

> I would add a stable `cluster` label, use remote write or long-term storage such as Thanos, Cortex, Mimir, or VictoriaMetrics, and design dashboards with cluster, namespace, service, and team variables. Alerts should include cluster labels for routing and incident context.

---

# 26. Senior Architecture Answer

If asked to explain Prometheus/Grafana architecture:

> In production, applications and exporters expose metrics on `/metrics`. Prometheus discovers targets through static config or Kubernetes service discovery, then scrapes them at configured intervals. It stores the samples locally and evaluates recording and alerting rules. Grafana queries Prometheus for dashboards, while Alertmanager receives alerts from Prometheus and handles grouping, deduplication, routing, silencing, and inhibition. In Kubernetes, I prefer Prometheus Operator with ServiceMonitor, PodMonitor, and PrometheusRule resources so monitoring is GitOps-friendly. For long-term retention and multi-cluster visibility, I would add remote write or systems like Thanos, Cortex, Mimir, or VictoriaMetrics. I also pay close attention to cardinality, because bad labels can make Prometheus expensive and unstable.

---

# 27. One-Minute v3 Interview Summary

> Prometheus is not just PromQL. It is a scrape-based monitoring system. It discovers targets, scrapes `/metrics`, stores time series, evaluates rules, and sends alerts to Alertmanager. In Kubernetes, monitoring is usually managed with Prometheus Operator using ServiceMonitor, PodMonitor, and PrometheusRule. For queries, I start with metric type, then choose `rate()`, `increase()`, direct gauge usage, or `histogram_quantile()`. Then I aggregate, rank with `topk()`, or join metrics using vector matching. In production, I watch cardinality, scrape health, missing metrics, dashboard performance, alert noise, and long-term storage. A good observability setup combines internal metrics, external blackbox probes, SLOs, alerts, and dashboards that support decisions rather than just showing graphs.

---

# 28. Final Practical Checklist

Before an interview, make sure you can explain:

```text
1. What Prometheus scrapes
2. What exporters do
3. What /metrics means
4. What jobs, instances, targets, and labels are
5. What up == 0 means
6. How ServiceMonitor and PodMonitor work
7. Difference between cAdvisor and kube-state-metrics
8. Difference between relabel_configs and metric_relabel_configs
9. Why high cardinality is dangerous
10. Difference between zero and absent metrics
11. How Alertmanager reduces alert noise
12. When to use recording rules
13. When to use remote write
14. How to troubleshoot missing Grafana data
15. How to explain SLI/SLO, RED, USE, and golden signals
```

---

# 29. Fast Interview Phrases

Use these phrases when answering:

```text
"First I check whether the target is being scraped using up."
"I separate usage metrics from Kubernetes state metrics."
"I avoid high-cardinality labels like request_id and user_id."
"I use ServiceMonitor or PodMonitor so scraping is GitOps-friendly."
"I use recording rules for expensive queries and repeated SLO logic."
"I use Alertmanager for grouping, deduplication, routing, silencing, and inhibition."
"I use blackbox exporter for outside-in user-path checks."
"I distinguish missing metrics from zero values using absent() and up."
"I use RED for services and USE for infrastructure."
"I design dashboards to support decisions, not just show graphs."
```

---

# 30. The Final Senior Rule

A junior answer gives a query.

A senior answer explains the system:

```text
Metric source
Scrape path
Labels/cardinality
Correct PromQL function
Aggregation level
Dashboard purpose
Alert quality
Operational action
```

Example senior explanation:

> For top CPU pods, I would use cAdvisor CPU counters, apply `rate()` because CPU usage is accumulated CPU seconds, aggregate by namespace and pod, then use `topk()` to identify noisy workloads. I would avoid grouping by unnecessary labels to control cardinality. If the panel is used frequently, I may create a recording rule. If it becomes an alert, I would compare usage to requests or limits and add a `for:` duration to avoid alerting on short spikes.

# Prometheus & Grafana Cheat Sheet v2

**Focus:** PromQL operators, Kubernetes observability, Grafana dashboards, alerting, and SRE interview scenarios.

This v2 cheat sheet extends a basic Prometheus metric-type cheat sheet.  
The original foundation usually covers:

- Counter → `rate()` / `increase()`
- Gauge → direct value / aggregation
- Histogram → `histogram_quantile()`
- Summary → quantile label or `_sum / _count`

This version focuses on what interviews often test after that foundation:

- `topk()` / `bottomk()`
- Label filtering
- Aggregation
- Vector matching and joins
- Time ranges and offsets
- Recording rules
- Alerting rules
- SLI/SLO queries
- Kubernetes troubleshooting queries
- Grafana dashboard concepts
- Senior interview explanations

---

# 1. Mental Model

PromQL is not only about choosing the correct metric function.

A strong production query usually follows this flow:

```text
1. Select the right metric
2. Filter the right labels
3. Apply the correct metric-type function
4. Aggregate by useful labels
5. Compare, rank, join, or alert
6. Visualize clearly in Grafana
```

Example:

```promql
topk(
  5,
  sum by (namespace, pod) (
    rate(container_cpu_usage_seconds_total{container!="POD", pod!=""}[5m])
  )
)
```

Meaning:

```text
Find the top 5 pods currently consuming the most CPU.
```

---

# 2. Ranking Functions

Ranking functions help you find the biggest consumers, worst offenders, or lowest performers.

## 2.1 topk()

`topk(k, expression)` returns the top `k` time series with the highest values.

### Top 5 pods by CPU usage

```promql
topk(
  5,
  sum by (namespace, pod) (
    rate(container_cpu_usage_seconds_total{container!="POD", pod!=""}[5m])
  )
)
```

### Top 5 pods by memory usage

```promql
topk(
  5,
  sum by (namespace, pod) (
    container_memory_working_set_bytes{container!="POD", pod!=""}
  )
)
```

### Top 10 pods by restart count in the last 30 minutes

```promql
topk(
  10,
  sum by (namespace, pod, container) (
    increase(kube_pod_container_status_restarts_total[30m])
  )
)
```

### Top 5 services by 5xx error rate

```promql
topk(
  5,
  sum by (service) (
    rate(http_requests_total{status=~"5.."}[5m])
  )
)
```

## Interview explanation

> `topk()` is a ranking operator. It does not replace `rate()`, `increase()`, or aggregation. First, I calculate the correct value based on the metric type, then I use `topk()` to show the highest N series. In Grafana, I use it for panels like top CPU pods, top memory pods, top restart offenders, top erroring services, or top noisy endpoints.

---

## 2.2 bottomk()

`bottomk(k, expression)` returns the bottom `k` time series with the lowest values.

### Bottom 5 pods by available memory, example using node memory

```promql
bottomk(
  5,
  node_memory_MemAvailable_bytes
)
```

### Bottom 5 deployments by available replicas

```promql
bottomk(
  5,
  kube_deployment_status_replicas_available
)
```

## When to use bottomk()

Use `bottomk()` when you want to identify:

- Least available memory
- Lowest replica availability
- Lowest throughput
- Lowest success rate
- Lowest free disk space

---

## 2.3 sort() and sort_desc()

```promql
sort(expression)
```

Sorts ascending.

```promql
sort_desc(expression)
```

Sorts descending.

Example:

```promql
sort_desc(
  sum by (namespace, pod) (
    container_memory_working_set_bytes{container!="POD", pod!=""}
  )
)
```

Use this when you want the full sorted list instead of only the top N.

---

# 3. Label Filtering

Labels are how Prometheus identifies dimensions such as namespace, pod, service, status code, method, node, and container.

## 3.1 Exact match

```promql
namespace="prod"
```

Example:

```promql
container_memory_working_set_bytes{namespace="prod"}
```

Meaning:

```text
Only metrics from namespace prod.
```

---

## 3.2 Not equal

```promql
namespace!="test"
```

Example:

```promql
container_memory_working_set_bytes{namespace!="test"}
```

Meaning:

```text
Exclude namespace test.
```

---

## 3.3 Regex match

```promql
pod=~"frontend-.*"
```

Example:

```promql
rate(http_requests_total{pod=~"frontend-.*"}[5m])
```

Meaning:

```text
Only pods whose names start with frontend-.
```

---

## 3.4 Negative regex match

```promql
pod!~"debug-.*"
```

Example:

```promql
rate(container_cpu_usage_seconds_total{pod!~"debug-.*"}[5m])
```

Meaning:

```text
Exclude debug pods.
```

---

## 3.5 Common label filtering examples

### HTTP 5xx errors

```promql
rate(http_requests_total{status=~"5.."}[5m])
```

### HTTP 4xx client errors

```promql
rate(http_requests_total{status=~"4.."}[5m])
```

### Exclude infrastructure containers

```promql
container!="POD"
```

### Exclude empty containers and pods

```promql
container!="", pod!=""
```

### Match multiple namespaces

```promql
namespace=~"prod|staging"
```

---

# 4. Aggregation Operators

Aggregation reduces many time series into useful groups.

## 4.1 sum by()

Use when you want a total per label group.

### CPU per pod

```promql
sum by (namespace, pod) (
  rate(container_cpu_usage_seconds_total{container!="POD", pod!=""}[5m])
)
```

### Memory per namespace

```promql
sum by (namespace) (
  container_memory_working_set_bytes{container!="POD", pod!=""}
)
```

---

## 4.2 avg by()

Use when you want an average per group.

### Average CPU per namespace

```promql
avg by (namespace) (
  rate(container_cpu_usage_seconds_total{container!="POD", pod!=""}[5m])
)
```

---

## 4.3 max by()

Use when you want the highest value per group.

### Max memory usage by pod

```promql
max by (namespace, pod) (
  container_memory_working_set_bytes{container!="POD", pod!=""}
)
```

---

## 4.4 min by()

Use when you want the lowest value per group.

### Minimum available replicas by deployment

```promql
min by (namespace, deployment) (
  kube_deployment_status_replicas_available
)
```

---

## 4.5 count by()

Use when you want to count how many series exist per label.

### Count pods by phase

```promql
sum by (phase) (
  kube_pod_status_phase == 1
)
```

### Count running pods by namespace

```promql
sum by (namespace) (
  kube_pod_status_phase{phase="Running"} == 1
)
```

---

## 4.6 without()

`without()` aggregates while dropping specific labels.

Example:

```promql
sum without (instance, job) (
  rate(http_requests_total[5m])
)
```

Meaning:

```text
Sum request rate while ignoring instance and job labels.
```

---

## Interview explanation

> Aggregation is how I turn raw high-cardinality time series into operational signals. For Kubernetes, I usually aggregate CPU and memory by namespace, pod, container, node, or workload depending on the troubleshooting goal.

---

# 5. Time Range Selectors

Prometheus needs a time range for functions like `rate()` and `increase()`.

## 5.1 Common ranges

```promql
[1m]
[5m]
[15m]
[30m]
[1h]
[24h]
```

## 5.2 rate()

Use `rate()` for per-second speed over a time window.

```promql
rate(http_requests_total[5m])
```

Meaning:

```text
Requests per second over the last 5 minutes.
```

## 5.3 increase()

Use `increase()` for total count over a time window.

```promql
increase(kube_pod_container_status_restarts_total[15m])
```

Meaning:

```text
How many restarts happened in the last 15 minutes.
```

---

## 5.4 Choosing the right window

| Window | Use case |
|---|---|
| `[1m]` | Very fast detection, can be noisy |
| `[5m]` | Good default for dashboards |
| `[15m]` | Better for alerts, smoother |
| `[1h]` | Trends and capacity analysis |
| `[24h]` | Daily behavior, SLO analysis |

## Interview explanation

> Short windows are more sensitive but noisy. Longer windows are more stable but slower to detect problems. For dashboards I often use 5 minutes. For alerts, I may use 5 to 15 minutes plus a `for:` duration to avoid flapping.

---

# 6. Offset and Historical Comparison

`offset` lets you compare current data with past data.

## 6.1 Compare current request rate with one hour ago

```promql
sum(rate(http_requests_total[5m]))
-
sum(rate(http_requests_total[5m] offset 1h))
```

## 6.2 Compare current traffic with yesterday

```promql
sum(rate(http_requests_total[5m]))
/
sum(rate(http_requests_total[5m] offset 24h))
```

Meaning:

```text
Current request rate compared to the same time yesterday.
```

## When to use offset

Use `offset` for:

- Traffic comparison
- Regression detection
- Before/after deployment analysis
- Seasonal or daily pattern comparison
- Incident investigation

---

# 7. Vector Matching and Joins

This is a senior PromQL topic.

You need vector matching when you divide, compare, or combine two metrics that have different label sets.

Common keywords:

```text
on()
ignoring()
group_left
group_right
```

---

## 7.1 on()

`on(label)` tells Prometheus to match only on specific labels.

### CPU usage as percentage of requested CPU per pod

```promql
100 *
sum by (namespace, pod) (
  rate(container_cpu_usage_seconds_total{container!="POD", pod!=""}[5m])
)
/
on(namespace, pod)
sum by (namespace, pod) (
  kube_pod_container_resource_requests{resource="cpu"}
)
```

Meaning:

```text
How much CPU each pod is using compared to its requested CPU.
```

---

## 7.2 Memory usage as percentage of memory limit

```promql
100 *
sum by (namespace, pod) (
  container_memory_working_set_bytes{container!="POD", pod!=""}
)
/
on(namespace, pod)
sum by (namespace, pod) (
  kube_pod_container_resource_limits{resource="memory"}
)
```

Meaning:

```text
Current memory usage compared to configured memory limit.
```

---

## 7.3 ignoring()

`ignoring(label)` tells Prometheus to ignore specific labels during matching.

Example:

```promql
rate(http_requests_total{status=~"5.."}[5m])
/
ignoring(status)
rate(http_requests_total[5m])
```

Meaning:

```text
Compare 5xx request rate against total request rate while ignoring the status label mismatch.
```

---

## 7.4 group_left

Use `group_left` when the left side has more series and you want to attach labels from the right side.

Example: attach node information to pod metrics.

```promql
sum by (namespace, pod, node) (
  rate(container_cpu_usage_seconds_total{container!="POD", pod!=""}[5m])
)
*
on(namespace, pod)
group_left(node)
kube_pod_info
```

---

## Interview explanation

> Vector matching is needed when PromQL combines metrics with different label dimensions. For example, usage metrics from cAdvisor and request/limit metrics from kube-state-metrics may not have identical labels. I use `on()` to define matching labels and `group_left` or `group_right` when one side has additional dimensions.

---

# 8. Binary Operators and Comparisons

PromQL supports arithmetic, comparison, and logical operations.

## 8.1 Arithmetic

```promql
+
-
*
/
%
```

Example: memory usage percentage

```promql
100 *
container_memory_working_set_bytes
/
container_spec_memory_limit_bytes
```

---

## 8.2 Comparisons

```promql
>
<
>=
<=
==
!=
```

Example:

```promql
kube_pod_status_phase{phase="Pending"} == 1
```

Meaning:

```text
Show pods currently in Pending phase.
```

---

## 8.3 bool modifier

Without `bool`, comparison filters series.

With `bool`, comparison returns 0 or 1.

```promql
kube_deployment_status_replicas_available < bool kube_deployment_spec_replicas
```

Meaning:

```text
Return 1 if available replicas are less than desired replicas.
```

---

# 9. Recording Rules

Recording rules precompute PromQL expressions and store the result as a new time series.

## Why recording rules matter

They help with:

- Faster dashboards
- Reusable SLI/SLO queries
- Consistent alert logic
- Lower query load
- Cleaner PromQL in Grafana

---

## 9.1 Example recording rule: namespace CPU

```yaml
groups:
  - name: kubernetes-workload.rules
    rules:
      - record: namespace:cpu_usage_cores:rate5m
        expr: |
          sum by (namespace) (
            rate(container_cpu_usage_seconds_total{container!="POD", pod!=""}[5m])
          )
```

Then query:

```promql
namespace:cpu_usage_cores:rate5m
```

---

## 9.2 Example recording rule: namespace memory

```yaml
groups:
  - name: kubernetes-workload.rules
    rules:
      - record: namespace:memory_working_set_bytes
        expr: |
          sum by (namespace) (
            container_memory_working_set_bytes{container!="POD", pod!=""}
          )
```

---

## Interview explanation

> In production, I do not want every Grafana panel or alert to execute a heavy query repeatedly. I use recording rules to precompute expensive or business-critical expressions, especially SLI/SLO metrics, namespace resource usage, and high-cardinality Kubernetes queries.

---

# 10. Alerting Rules

A good alert is not just a query. It needs stability, severity, routing labels, and useful annotations.

## 10.1 Alert rule structure

```yaml
groups:
  - name: kubernetes-alerts
    rules:
      - alert: HighPodCPUUsage
        expr: |
          100 *
          sum by (namespace, pod) (
            rate(container_cpu_usage_seconds_total{container!="POD", pod!=""}[5m])
          )
          /
          on(namespace, pod)
          sum by (namespace, pod) (
            kube_pod_container_resource_limits{resource="cpu"}
          ) > 85
        for: 10m
        labels:
          severity: warning
          team: platform
        annotations:
          summary: "High CPU usage on pod {{ $labels.pod }}"
          description: "Pod {{ $labels.pod }} in namespace {{ $labels.namespace }} has used more than 85% of its CPU limit for 10 minutes."
```

---

## 10.2 Important alerting concepts

| Concept | Meaning |
|---|---|
| `expr` | PromQL condition |
| `for` | Condition must remain true before firing |
| `labels` | Used for routing and grouping |
| `annotations` | Human-readable message |
| severity | Warning, critical, info |
| grouping | Combine related alerts |
| inhibition | Suppress lower-priority alerts |
| silence | Temporarily mute known alerts |

---

## 10.3 Avoid noisy alerts

Bad alert:

```promql
rate(http_requests_total{status=~"5.."}[1m]) > 0
```

Problem:

```text
One short spike can trigger an alert.
```

Better:

```promql
(
  sum(rate(http_requests_total{status=~"5.."}[5m]))
  /
  sum(rate(http_requests_total[5m]))
) > 0.05
```

With:

```yaml
for: 10m
```

Meaning:

```text
Alert only when more than 5% of traffic is failing for 10 minutes.
```

---

# 11. SLI, SLO, Error Budget, and Burn Rate

This is very important for SRE interviews.

## 11.1 Definitions

| Term | Full meaning | Meaning |
|---|---|---|
| SLI | Service Level Indicator | What you measure |
| SLO | Service Level Objective | Target reliability |
| Error budget | Allowed unreliability | How much failure is acceptable |
| Burn rate | Error budget consumption speed | How fast the budget is being consumed |

---

## 11.2 Availability SLI

```promql
1 -
(
  sum(rate(http_requests_total{status=~"5.."}[5m]))
  /
  sum(rate(http_requests_total[5m]))
)
```

Meaning:

```text
Successful request ratio.
```

---

## 11.3 Error rate

```promql
sum(rate(http_requests_total{status=~"5.."}[5m]))
/
sum(rate(http_requests_total[5m]))
```

Meaning:

```text
Percentage of requests failing with 5xx errors.
```

---

## 11.4 Latency SLI: p95

```promql
histogram_quantile(
  0.95,
  sum by (le) (
    rate(http_request_duration_seconds_bucket[5m])
  )
)
```

Meaning:

```text
95% of requests are faster than this latency.
```

---

## 11.5 Latency SLI per route

```promql
histogram_quantile(
  0.95,
  sum by (le, route) (
    rate(http_request_duration_seconds_bucket[5m])
  )
)
```

---

## 11.6 Multi-window burn-rate style idea

Fast burn:

```promql
sum(rate(http_requests_total{status=~"5.."}[5m]))
/
sum(rate(http_requests_total[5m]))
```

Slow burn:

```promql
sum(rate(http_requests_total{status=~"5.."}[1h]))
/
sum(rate(http_requests_total[1h]))
```

Senior explanation:

> I use short windows to detect fast incidents and longer windows to avoid noise. A mature SLO alert often combines both, so the system catches serious problems quickly while avoiding alerts for short harmless spikes.

---

# 12. Kubernetes Production Troubleshooting Queries

This section is important for DevOps, SRE, Platform Engineering, and Kubernetes interviews.

---

## 12.1 Top CPU-consuming pods

```promql
topk(
  10,
  sum by (namespace, pod) (
    rate(container_cpu_usage_seconds_total{container!="POD", pod!=""}[5m])
  )
)
```

---

## 12.2 Top memory-consuming pods

```promql
topk(
  10,
  sum by (namespace, pod) (
    container_memory_working_set_bytes{container!="POD", pod!=""}
  )
)
```

---

## 12.3 Pods using high CPU compared to request

```promql
100 *
sum by (namespace, pod) (
  rate(container_cpu_usage_seconds_total{container!="POD", pod!=""}[5m])
)
/
on(namespace, pod)
sum by (namespace, pod) (
  kube_pod_container_resource_requests{resource="cpu"}
)
```

---

## 12.4 Pods using high memory compared to limit

```promql
100 *
sum by (namespace, pod) (
  container_memory_working_set_bytes{container!="POD", pod!=""}
)
/
on(namespace, pod)
sum by (namespace, pod) (
  kube_pod_container_resource_limits{resource="memory"}
)
```

---

## 12.5 Restart offenders

```promql
topk(
  10,
  sum by (namespace, pod, container) (
    increase(kube_pod_container_status_restarts_total[30m])
  )
)
```

---

## 12.6 CrashLoopBackOff containers

```promql
sum by (namespace, pod, container, reason) (
  kube_pod_container_status_waiting_reason{reason="CrashLoopBackOff"} == 1
)
```

---

## 12.7 ImagePullBackOff containers

```promql
sum by (namespace, pod, container, reason) (
  kube_pod_container_status_waiting_reason{reason="ImagePullBackOff"} == 1
)
```

---

## 12.8 Pending pods

```promql
sum by (namespace, pod) (
  kube_pod_status_phase{phase="Pending"} == 1
)
```

---

## 12.9 Failed pods

```promql
sum by (namespace, pod) (
  kube_pod_status_phase{phase="Failed"} == 1
)
```

---

## 12.10 OOMKilled containers

```promql
kube_pod_container_status_last_terminated_reason{reason="OOMKilled"} == 1
```

---

## 12.11 Deployment missing replicas

```promql
kube_deployment_spec_replicas
-
kube_deployment_status_replicas_available
```

Grouped:

```promql
sum by (namespace, deployment) (
  kube_deployment_spec_replicas
  -
  kube_deployment_status_replicas_available
)
```

---

## 12.12 Node memory availability

```promql
node_memory_MemAvailable_bytes
/
node_memory_MemTotal_bytes
* 100
```

---

## 12.13 Node filesystem usage percentage

```promql
100 *
(
  1 -
  (
    node_filesystem_avail_bytes{fstype!~"tmpfs|overlay"}
    /
    node_filesystem_size_bytes{fstype!~"tmpfs|overlay"}
  )
)
```

---

## 12.14 Node CPU usage percentage

```promql
100 *
(
  1 -
  avg by (instance) (
    rate(node_cpu_seconds_total{mode="idle"}[5m])
  )
)
```

---

## 12.15 Network receive rate per pod

```promql
sum by (namespace, pod) (
  rate(container_network_receive_bytes_total{pod!=""}[5m])
)
```

---

## 12.16 Network transmit rate per pod

```promql
sum by (namespace, pod) (
  rate(container_network_transmit_bytes_total{pod!=""}[5m])
)
```

---

# 13. Grafana Dashboard Concepts

Grafana interviews may test not only PromQL, but dashboard design.

---

## 13.1 Variables

Variables make dashboards reusable.

Example namespace variable:

```promql
label_values(kube_pod_info, namespace)
```

Then use it in queries:

```promql
sum by (pod) (
  rate(container_cpu_usage_seconds_total{namespace="$namespace", container!="POD", pod!=""}[5m])
)
```

Common variables:

| Variable | Query |
|---|---|
| namespace | `label_values(kube_pod_info, namespace)` |
| pod | `label_values(kube_pod_info{namespace="$namespace"}, pod)` |
| node | `label_values(kube_node_info, node)` |
| deployment | `label_values(kube_deployment_labels{namespace="$namespace"}, deployment)` |

---

## 13.2 Units

Always set correct units.

| Metric | Grafana unit |
|---|---|
| CPU cores | cores / decimal |
| CPU percent | percent |
| Memory bytes | bytes |
| Network bytes/sec | bytes/sec |
| Latency seconds | seconds or milliseconds |
| Requests/sec | req/s |
| Error rate | percent |

---

## 13.3 Thresholds

Use thresholds to make dashboards operational.

Example:

| Panel | Green | Yellow | Red |
|---|---:|---:|---:|
| CPU usage % | < 70 | 70-85 | > 85 |
| Memory usage % | < 70 | 70-85 | > 85 |
| 5xx error rate | < 1% | 1-5% | > 5% |
| p95 latency | app-specific | app-specific | app-specific |
| Missing replicas | 0 | 1 | > 1 |

---

## 13.4 Transformations

Grafana transformations are useful for table panels.

Common transformations:

| Transformation | Use |
|---|---|
| Join by field | Combine multiple query results |
| Organize fields | Rename/reorder columns |
| Filter data by value | Hide unwanted rows |
| Reduce | Convert time series to latest/min/max/avg |
| Add field from calculation | Compute percentage or difference |
| Labels to fields | Convert labels into table columns |

---

## 13.5 Annotations

Annotations mark events on dashboards.

Use annotations for:

- Deployments
- Incidents
- Scaling events
- Configuration changes
- Maintenance windows

Senior explanation:

> Annotations help correlate metric changes with operational events. For example, if latency increases immediately after a deployment annotation, the dashboard supports faster incident investigation.

---

## 13.6 Dashboard structure

A strong Kubernetes dashboard should have layers:

```text
1. Executive / service health overview
2. Workload health
3. Resource saturation
4. Error and latency signals
5. Kubernetes object health
6. Node health
7. Drilldown / investigation tables
```

Example rows:

```text
SLO Overview
Request / Error / Latency
Namespace Resource Usage
Pod Health
Deployment Health
Node Health
Storage Health
Investigation Tables
```

---

# 14. Dashboard Panel Recommendations

| Goal | Query pattern | Panel type |
|---|---|---|
| Top CPU pods | `topk() + rate() + sum by()` | Bar gauge / Table |
| Top memory pods | `topk() + gauge + sum by()` | Bar gauge / Table |
| Error rate | failed / total requests | Stat / Time series |
| p95 latency | `histogram_quantile()` | Time series |
| Missing replicas | desired - available | Stat / Table |
| CrashLoopBackOff | status gauge `== 1` | Table |
| Pending pods | phase gauge `== 1` | Table |
| Node CPU | idle inversion | Gauge / Time series |
| Node memory | available / total | Gauge |
| Restarts | `increase()` | Table / Bar gauge |

---

# 15. Common Interview Questions and Answers

## Q1. What is `topk()` in Prometheus?

Answer:

> `topk()` is a PromQL ranking operator. It returns the top N time series based on the highest values of an expression. I usually use it after building the correct query. For example, for CPU I first use `rate()` because CPU usage is a counter, then aggregate with `sum by`, and finally wrap the result with `topk()` to show the highest CPU-consuming pods.

Example:

```promql
topk(
  5,
  sum by (pod) (
    rate(container_cpu_usage_seconds_total{container!="POD", pod!=""}[5m])
  )
)
```

---

## Q2. Why should you not use counters directly?

Answer:

> A counter stores accumulated values since process start, so querying it directly usually shows lifetime total, not current activity. For current speed, I use `rate()`. For total change in a window, I use `increase()`.

Bad:

```promql
http_requests_total
```

Better:

```promql
rate(http_requests_total[5m])
```

---

## Q3. Why should you not use `rate()` on memory?

Answer:

> Memory usage is usually a gauge. It goes up and down and represents current state. `rate()` is for counters. For memory, I query the gauge directly and aggregate it if needed.

Example:

```promql
sum by (pod) (
  container_memory_working_set_bytes{container!="POD", pod!=""}
)
```

---

## Q4. How do you calculate CPU usage per pod?

Answer:

```promql
sum by (namespace, pod) (
  rate(container_cpu_usage_seconds_total{container!="POD", pod!=""}[5m])
)
```

Explanation:

> `container_cpu_usage_seconds_total` is a counter of accumulated CPU seconds, so I use `rate()` to calculate current CPU cores used. Then I aggregate by namespace and pod.

---

## Q5. How do you calculate memory usage per pod?

Answer:

```promql
sum by (namespace, pod) (
  container_memory_working_set_bytes{container!="POD", pod!=""}
)
```

Explanation:

> Memory is a gauge, so I use it directly and aggregate by namespace and pod.

---

## Q6. How do you identify noisy pods?

Answer:

> It depends on what "noisy" means. For CPU noise, I use `topk()` on pod CPU rate. For memory pressure, I use `topk()` on memory working set. For instability, I use `topk()` on container restarts. For traffic noise, I use `topk()` on request rate.

Example:

```promql
topk(
  10,
  sum by (namespace, pod) (
    rate(http_requests_total[5m])
  )
)
```

---

## Q7. How do you build a good alert?

Answer:

> A good alert should be actionable, stable, and tied to user or platform impact. I avoid alerting on short spikes. I use a meaningful PromQL expression, a `for:` duration, severity labels, routing labels, and annotations that explain what is wrong and where to investigate.

---

## Q8. What are recording rules?

Answer:

> Recording rules precompute PromQL expressions and store the result as new time series. They improve dashboard performance, reduce repeated heavy queries, and standardize logic across dashboards and alerts.

---

## Q9. What is vector matching?

Answer:

> Vector matching is how Prometheus combines two metrics with different label sets. I use `on()` to define which labels should match, `ignoring()` to exclude labels from matching, and `group_left` or `group_right` when one side has more dimensions.

---

## Q10. How would you create a Kubernetes dashboard?

Answer:

> I would start with service health and SLO signals: request rate, error rate, and latency. Then I would add workload health: pod restarts, CrashLoopBackOff, Pending pods, and missing replicas. After that, I would include resource saturation: CPU, memory, filesystem, network, and node health. Finally, I would add drilldown tables and variables for namespace, pod, deployment, and node.

---

# 16. Common Mistakes

## Mistake 1: Applying topk() before building the correct expression

Bad:

```promql
topk(5, container_cpu_usage_seconds_total)
```

Better:

```promql
topk(
  5,
  sum by (pod) (
    rate(container_cpu_usage_seconds_total{container!="POD", pod!=""}[5m])
  )
)
```

---

## Mistake 2: Using counters directly

Bad:

```promql
http_requests_total
```

Better:

```promql
rate(http_requests_total[5m])
```

---

## Mistake 3: Using rate() on gauges

Bad:

```promql
rate(container_memory_working_set_bytes[5m])
```

Better:

```promql
container_memory_working_set_bytes
```

---

## Mistake 4: Forgetting histogram `le` label

Bad:

```promql
histogram_quantile(
  0.95,
  sum(rate(http_request_duration_seconds_bucket[5m]))
)
```

Better:

```promql
histogram_quantile(
  0.95,
  sum by (le) (
    rate(http_request_duration_seconds_bucket[5m])
  )
)
```

---

## Mistake 5: Alerting on raw spikes

Bad:

```promql
rate(http_requests_total{status=~"5.."}[1m]) > 0
```

Better:

```promql
(
  sum(rate(http_requests_total{status=~"5.."}[5m]))
  /
  sum(rate(http_requests_total[5m]))
) > 0.05
```

With:

```yaml
for: 10m
```

---

## Mistake 6: Ignoring labels and cardinality

Bad practice:

```text
Creating dashboards or alerts that group by too many labels.
```

Problem:

```text
High cardinality can make Prometheus slow and dashboards expensive.
```

Better:

```text
Group by operationally useful labels such as namespace, pod, service, deployment, node, status, or route.
```

---

# 17. Senior Production Mindset

A senior answer is not only the query.

A senior answer explains:

```text
Why this metric?
Why this function?
Why this aggregation?
Why this time window?
Why this alert threshold?
What would I check next?
What operational decision does this panel support?
```

Example:

> I would not just build a CPU panel. I would decide whether I need node CPU, namespace CPU, pod CPU, or container CPU. Then I would use `rate()` because CPU is a counter, aggregate by the right ownership boundary, and use `topk()` if I want to find noisy workloads. For alerting, I would compare usage against request or limit and add a `for:` duration to avoid flapping.

---

# 18. Quick Reference Table

| Goal | PromQL pattern |
|---|---|
| Top CPU pods | `topk(N, sum by (pod)(rate(cpu_counter[5m])))` |
| Top memory pods | `topk(N, sum by (pod)(memory_gauge))` |
| Restart count | `increase(restarts_total[15m])` |
| Request rate | `rate(http_requests_total[5m])` |
| Error rate | `rate(5xx) / rate(total)` |
| p95 latency | `histogram_quantile(0.95, sum by (le)(rate(bucket[5m])))` |
| Missing replicas | `desired - available` |
| Pending pods | `kube_pod_status_phase{phase="Pending"} == 1` |
| CrashLoopBackOff | `waiting_reason{reason="CrashLoopBackOff"} == 1` |
| OOMKilled | `last_terminated_reason{reason="OOMKilled"} == 1` |
| CPU vs request | `usage / request * 100` |
| Memory vs limit | `usage / limit * 100` |
| Historical comparison | `query - query offset 1h` |
| Reusable heavy query | recording rule |
| Stable alert | query + `for:` duration |

---

# 19. One-Minute Interview Summary

If asked about Prometheus/Grafana in an interview, say:

> I start by identifying the metric type. Counters need `rate()` or `increase()`, gauges are used directly, and histograms need `histogram_quantile()`. After that, I aggregate with `sum by`, `avg by`, or `max by` depending on the operational question. If I need the biggest consumers or worst offenders, I use `topk()` after the correct expression is built. For Kubernetes dashboards, I focus on CPU, memory, restarts, pod phases, waiting reasons, missing replicas, node health, request rate, error rate, and latency. For alerting, I avoid noisy raw spikes by using ratios, time windows, and `for:` durations. In production, I also use recording rules for expensive queries and Grafana variables, thresholds, and transformations to make dashboards reusable and actionable.

---

# 20. Final Practical Rule

PromQL interview success comes from explaining the chain:

```text
Metric type → correct function → useful aggregation → ranking/comparison → operational meaning
```

Example:

```promql
topk(
  5,
  sum by (namespace, pod) (
    rate(container_cpu_usage_seconds_total{container!="POD", pod!=""}[5m])
  )
)
```

Explanation:

```text
container_cpu_usage_seconds_total is a counter.
So I use rate().
I aggregate by namespace and pod.
Then I use topk(5) to show the highest CPU-consuming pods.
This helps identify noisy workloads or resource pressure.
```

That is the interview-level answer.

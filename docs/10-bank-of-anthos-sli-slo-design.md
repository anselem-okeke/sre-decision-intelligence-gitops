# Bank of Anthos SLI/SLO Design

## Objective

This document defines the SLI/SLO design approach for Bank of Anthos.

The goal is to define user-impact signals that can later trigger Decision Intelligence workflows.

This phase does **not** start from random infrastructure metrics. It starts from user journeys and then maps those journeys to measurable SLIs, candidate SLO targets, and PromQL/OpenSearch validation queries.

---

## SRE Principle

SLOs should be defined from user journeys, not from random available infrastructure metrics.

The correct flow is:

```text
User journey
    ↓
SLI
    ↓
SLO target
    ↓
Measurement point
    ↓
PromQL query
    ↓
Alert / incident trigger
    ↓
Decision Intelligence workflow
```

Infrastructure and platform signals are still important, but they are supporting evidence. They help explain why a user-facing SLO may be degraded.

---

## Workload

The workload is **Bank of Anthos**, deployed in the namespace:

```text
fintech-workload
```

Primary known workload identity fields:

```text
kubernetes.namespace_name = fintech-workload
kubernetes.container_name = front
```

Kubernetes labels are intentionally not used as primary query identity in this phase because labels and annotations are disabled in Fluent Bit to avoid OpenSearch dynamic mapping conflicts.

---

## Critical User Journeys

Initial user journeys:

| User journey | Question |
|---|---|
| Frontend access | Can users access the banking frontend? |
| Login | Can users authenticate successfully? |
| Deposit/payment action | Can users complete financial actions? |
| Page/API response time | Is the user experience fast enough? |

These journeys are the foundation for defining user-facing SLIs and SLOs.

---

## Candidate SLOs

| SLO | SLI | Initial target |
|---|---|---|
| Frontend availability | Successful frontend requests / total frontend requests | 99% |
| Frontend latency | p95 frontend request latency | under 1s |
| Transaction action success | Successful deposit/payment actions / total actions | 99% |
| Login success | Successful logins / total login attempts | 99% |

These are candidate SLOs. They should be validated against available telemetry before becoming final operational SLOs.

---

## Measurement Strategy

Preferred measurement points, in order of quality:

1. Edge / ingress / gateway metrics
2. Frontend HTTP metrics
3. Application business metrics
4. Platform fallback metrics
5. Log-derived signals from OpenSearch

### Why this order matters

User-facing SLOs should be measured as close to the user experience as possible.

For example, frontend availability is better measured from ingress or HTTP request metrics than from pod readiness alone.

Pod readiness may be healthy while users still experience HTTP 500s. Similarly, CPU may look normal while business transactions are failing.

---

## Important Distinction

Platform metrics such as pod restarts, CPU usage, memory pressure, and deployment availability are important, but they are **not primary user-facing SLOs**.

They are investigation signals.

Example:

```text
Frontend 5xx increase = user-impact symptom
Pod restart = possible supporting evidence
Argo CD sync = possible change context
OpenSearch ERROR logs = possible root-cause evidence
```

This distinction is important for the Decision Intelligence platform.

The future system should not simply say:

```text
CPU is high, therefore incident.
```

It should reason like:

```text
Frontend availability degraded.
At the same time, frontend pods restarted.
Argo CD synced the application shortly before the degradation.
OpenSearch shows frontend ERROR logs after the sync.
Likely cause: recent rollout introduced frontend instability.
```

---

## Current Discovery Status

Prometheus and Grafana are already available in the Talos cluster.

Platform metrics are available, including:

- pod restarts
- pod phase
- deployment availability
- CPU usage
- memory usage
- OOM events
- network counters

OpenSearch logs are available for workload and platform investigation.

Fresh structured workload logs are available for:

- `fintech-workload`
- frontend container logs
- `timestamp`
- `message`
- `severity`

Future structured log aliases may include:

- `app_timestamp`
- `app_message`
- `app_severity`

The next discovery step is to verify which HTTP, latency, ingress, Cilium/Hubble, or application metrics exist for `fintech-workload`.

---

## Query Files

SLI/SLO discovery queries:

```text
observability/prometheus/queries/slo-discovery.promql
```

Candidate Bank of Anthos SLI queries:

```text
observability/prometheus/queries/bank-of-anthos-sli.promql
```

---

## SLI Discovery Approach

Before finalizing an SLO, first discover whether the right metrics exist.

Example discovery questions:

| Question | Signal source |
|---|---|
| Do we have HTTP request counters? | Prometheus |
| Do we have HTTP status code labels? | Prometheus |
| Do we have request duration histograms? | Prometheus |
| Do we have ingress/gateway metrics? | Prometheus |
| Do we have Cilium/Hubble flow metrics? | Cilium/Hubble |
| Do we have application business event logs? | OpenSearch |
| Do we have ERROR/WARN severity logs? | OpenSearch |

If user-facing metrics are not available, use platform/log-derived fallback signals temporarily and document the limitation.

---

## Candidate SLI Definitions

### 1. Frontend Availability

**User journey:** Frontend access

**Question:** Can users access the banking frontend?

**Preferred SLI:**

```text
successful frontend HTTP requests / total frontend HTTP requests
```

**Preferred measurement source:**

```text
Ingress / gateway / frontend HTTP metrics
```

**Candidate target:**

```text
99%
```

**Fallback signal if HTTP metrics are unavailable:**

```text
frontend pod readiness
frontend deployment availability
frontend ERROR logs
```

---

### 2. Frontend Latency

**User journey:** Page/API response time

**Question:** Is the user experience fast enough?

**Preferred SLI:**

```text
p95 frontend request latency
```

**Preferred measurement source:**

```text
HTTP request duration histogram
```

**Candidate target:**

```text
p95 < 1s
```

**Fallback signal if latency metrics are unavailable:**

```text
OpenSearch frontend log timestamps and error patterns
resource pressure indicators
```

---

### 3. Login Success

**User journey:** Login

**Question:** Can users authenticate successfully?

**Preferred SLI:**

```text
successful login events / total login attempts
```

**Preferred measurement source:**

```text
application business metrics
```

**Fallback signal:**

```text
OpenSearch logs matching login success/failure messages
```

Example log-derived event indicators:

```text
_login_helper | Successfully logged in.
login failed
authentication error
```

---

### 4. Transaction Action Success

**User journey:** Deposit/payment action

**Question:** Can users complete financial actions?

**Preferred SLI:**

```text
successful deposit/payment actions / total deposit/payment attempts
```

**Preferred measurement source:**

```text
application business metrics
```

**Fallback signal:**

```text
OpenSearch logs matching deposit/payment success/failure messages
```

Example log-derived event indicators:

```text
deposit | Deposit submitted successfully.
payment | Payment initiated successfully.
payment failed
deposit failed
```

---

## PromQL Discovery Examples

These queries should be placed in:

```text
observability/prometheus/queries/slo-discovery.promql
```

### Discover metrics in fintech-workload

```promql
count by (__name__) (
  {namespace="fintech-workload"}
)
```

### Discover HTTP-related metrics

```promql
count by (__name__) (
  {__name__=~".*http.*"}
)
```

### Discover request-related metrics

```promql
count by (__name__) (
  {__name__=~".*request.*"}
)
```

### Discover latency/duration metrics

```promql
count by (__name__) (
  {__name__=~".*(latency|duration|seconds).*"}
)
```

### Discover ingress-related metrics

```promql
count by (__name__) (
  {__name__=~".*ingress.*"}
)
```

### Discover Cilium/Hubble-related metrics

```promql
count by (__name__) (
  {__name__=~".*(cilium|hubble).*"}
)
```

---

## Candidate PromQL Queries

These queries should be placed in:

```text
observability/prometheus/queries/bank-of-anthos-sli.promql
```

### Workload pod restarts

```promql
sum by (pod, container) (
  increase(kube_pod_container_status_restarts_total{namespace="fintech-workload"}[15m])
)
```

### Workload containers not ready

```promql
sum by (pod, container) (
  kube_pod_container_status_ready{namespace="fintech-workload"} == 0
)
```

### Workload deployment unavailable replicas

```promql
sum by (deployment) (
  kube_deployment_status_replicas_unavailable{namespace="fintech-workload"}
)
```

### Workload CPU usage

```promql
sum by (pod, container) (
  rate(container_cpu_usage_seconds_total{namespace="fintech-workload", container!="", image!=""}[5m])
)
```

### Workload memory usage

```promql
sum by (pod, container) (
  container_memory_working_set_bytes{namespace="fintech-workload", container!="", image!=""}
)
```

These are not final user-facing SLOs. They are supporting investigation signals until HTTP/user-journey metrics are verified.

---

## OpenSearch Log-Derived SLI Discovery

If application metrics are missing, OpenSearch can provide temporary log-derived signals.

### Recent frontend logs

```bash
curl "http://localhost:9200/k8s-logs-*/_search?pretty" \
  -H 'Content-Type: application/json' \
  -d '{
    "size": 10,
    "_source": [
      "@timestamp",
      "timestamp",
      "message",
      "severity",
      "kubernetes.namespace_name",
      "kubernetes.pod_name",
      "kubernetes.container_name"
    ],
    "query": {
      "bool": {
        "must": [
          { "match": { "kubernetes.namespace_name": "fintech-workload" }},
          { "match": { "kubernetes.container_name": "front" }}
        ]
      }
    },
    "sort": [
      { "@timestamp": { "order": "desc" }}
    ]
  }'
```

### Frontend ERROR logs

```bash
curl "http://localhost:9200/k8s-logs-*/_search?pretty" \
  -H 'Content-Type: application/json' \
  -d '{
    "size": 10,
    "_source": [
      "@timestamp",
      "timestamp",
      "message",
      "severity",
      "kubernetes.namespace_name",
      "kubernetes.pod_name",
      "kubernetes.container_name"
    ],
    "query": {
      "bool": {
        "must": [
          { "match": { "kubernetes.namespace_name": "fintech-workload" }},
          { "match": { "kubernetes.container_name": "front" }},
          { "match": { "severity": "ERROR" }}
        ]
      }
    },
    "sort": [
      { "@timestamp": { "order": "desc" }}
    ]
  }'
```

### Deposit/payment events

```bash
curl "http://localhost:9200/k8s-logs-*/_search?pretty" \
  -H 'Content-Type: application/json' \
  -d '{
    "size": 10,
    "_source": [
      "@timestamp",
      "timestamp",
      "message",
      "severity",
      "kubernetes.namespace_name",
      "kubernetes.pod_name",
      "kubernetes.container_name"
    ],
    "query": {
      "bool": {
        "must": [
          { "match": { "kubernetes.namespace_name": "fintech-workload" }},
          { "match": { "kubernetes.container_name": "front" }}
        ],
        "should": [
          { "match": { "message": "deposit" }},
          { "match": { "message": "payment" }}
        ],
        "minimum_should_match": 1
      }
    },
    "sort": [
      { "@timestamp": { "order": "desc" }}
    ]
  }'
```

### Login events

```bash
curl "http://localhost:9200/k8s-logs-*/_search?pretty" \
  -H 'Content-Type: application/json' \
  -d '{
    "size": 10,
    "_source": [
      "@timestamp",
      "timestamp",
      "message",
      "severity",
      "kubernetes.namespace_name",
      "kubernetes.pod_name",
      "kubernetes.container_name"
    ],
    "query": {
      "bool": {
        "must": [
          { "match": { "kubernetes.namespace_name": "fintech-workload" }},
          { "match": { "kubernetes.container_name": "front" }}
        ],
        "should": [
          { "match": { "message": "login" }},
          { "match": { "message": "_login_helper" }},
          { "match": { "message": "authenticated" }}
        ],
        "minimum_should_match": 1
      }
    },
    "sort": [
      { "@timestamp": { "order": "desc" }}
    ]
  }'
```

---

## Future Decision Intelligence Trigger

An SLO breach will become the first trigger for the Decision Intelligence app.

Example future flow:

```text
Frontend availability drops below target
        ↓
Decision app queries OpenSearch for frontend/backend errors
        ↓
Decision app checks Kubernetes pod/deployment state
        ↓
Decision app checks Argo CD recent sync state
        ↓
Decision app generates root cause and safe action
```

Example generated output:

```text
Impact:
Frontend users are affected.

Evidence:
Frontend availability dropped below 99%.
Frontend ERROR logs increased.
Frontend deployment has unavailable replicas.
Argo CD synced the workload shortly before the degradation.

Likely root cause:
Recent rollout introduced frontend instability.

Safe action:
Rollback latest frontend deployment and monitor availability recovery.
```

---

## Current Limitations

At this phase, some final SLI metrics may not yet exist.

Known limitations:

- HTTP request counters may not be exposed by the workload yet.
- HTTP latency histograms may not be available yet.
- Business success/failure counters may not be available yet.
- Cilium/Hubble metrics may need separate validation.
- Log-derived SLIs are useful but less ideal than direct metrics.

These limitations should be documented rather than hidden.

---

## Completion Criteria

This phase is complete when:

- Critical user journeys are documented
- Candidate SLOs are documented
- SLI discovery queries are created
- Candidate PromQL queries are created
- Available workload/edge metrics are validated
- Missing metrics are documented
- At least one user-facing SLI candidate is selected
- Supporting investigation signals are clearly separated from primary SLOs

---

## Recommended Next Steps

1. Create `observability/prometheus/queries/slo-discovery.promql`.
2. Create `observability/prometheus/queries/bank-of-anthos-sli.promql`.
3. Run discovery queries in Prometheus.
4. Identify whether HTTP request and latency metrics exist.
5. Validate OpenSearch log-derived frontend events.
6. Document which SLIs are real now and which require instrumentation later.

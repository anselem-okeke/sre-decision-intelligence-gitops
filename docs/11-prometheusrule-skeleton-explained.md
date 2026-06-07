# PrometheusRule Skeleton for Bank of Anthos SLO Candidates

## Objective

This document explains why the Bank of Anthos SLI/SLO discovery phase includes a `PrometheusRule` skeleton and why the rules intentionally use `vector(0)` during the discovery phase.

The purpose is to prepare the GitOps repository for future SLO alerting without creating false or fake production alerts before the correct user-facing metrics have been discovered and validated.

---

## Context

The SRE Decision Intelligence Platform is being built in phases.

At this stage, the platform already has working observability paths:

```text
Prometheus → platform/workload metrics
OpenSearch → workload/platform logs
Argo CD → GitOps deployment context
Kubernetes API → runtime state
```

Phase 8 focuses on SLI/SLO discovery for Bank of Anthos.

The key question is:

```text
Which user-facing signals can we measure reliably?
```

Before implementing real alerts, we must first discover whether the correct metrics exist.

---

## Why Prometheus Is Involved

Prometheus is used because SLOs are usually measured from metrics.

For example:

| SLO | Best measurement source |
|---|---|
| Frontend availability | HTTP request counters |
| Frontend latency | HTTP request duration histograms |
| Login success rate | Application/business counters |
| Transaction success rate | Application/business counters |

OpenSearch is useful for investigation and evidence, but Prometheus is better for continuous SLO measurement because it can calculate ratios, rates, histograms, and alert conditions over time.

Example SLO:

```text
99% of frontend requests should succeed.
```

This requires metrics such as:

```text
http_requests_total
http_request_duration_seconds_bucket
nginx_ingress_controller_requests
hubble_http_requests_total
frontend_request_duration_seconds_bucket
```

Then Prometheus can calculate an SLI such as:

```text
successful requests / total requests
```

and use that value to trigger an alert if it drops below the SLO target.

---

## Prometheus vs OpenSearch

Prometheus and OpenSearch have different roles.

| Tool | Role |
|---|---|
| Prometheus | Measure SLOs, calculate rates/ratios, trigger alerts |
| OpenSearch | Investigate logs, explain failures, provide root-cause evidence |

Example flow:

```text
Prometheus detects frontend availability degradation
        ↓
Decision Intelligence app queries OpenSearch for frontend/backend errors
        ↓
Decision Intelligence app checks Kubernetes runtime state
        ↓
Decision Intelligence app checks Argo CD recent sync state
        ↓
Decision Intelligence app generates likely cause and safe action
```

So Prometheus is the SLO trigger source, while OpenSearch is part of the investigation layer.

---

## Why We Do Not Create Final SLO Rules Yet

At this phase, the correct user-facing metrics may not exist yet.

For example, we do not yet know whether the cluster exposes metrics such as:

```text
http_requests_total
request_duration_seconds_bucket
nginx_ingress_controller_requests
hubble_http_requests_total
frontend_request_duration_seconds_bucket
```

If those metrics do not exist, creating final SLO alert rules would be fake.

Bad approach:

```yaml
expr: some_metric_that_may_not_exist < 0.99
```

Problems:

```text
The metric may not exist.
The alert may never work.
The SLO may not represent user impact.
The rule may create confusion or false confidence.
```

Correct approach:

```yaml
expr: vector(0)
```

This documents the intended future alert structure but prevents accidental firing.

---

## What `vector(0)` Means

`vector(0)` is a PromQL expression that returns a value of `0`.

In alerting terms, it means:

```text
This alert condition is always false.
```

So this rule:

```yaml
expr: vector(0)
```

will not trigger an alert.

This is intentional.

It allows the team to commit a PrometheusRule skeleton safely while waiting for metric discovery.

---

## Why Use a Skeleton Rule

The skeleton rule provides a clear GitOps-managed structure for future SLO alerts.

It documents:

- intended alert names
- intended SLO categories
- intended labels
- intended severity
- intended service ownership
- intended annotations
- future alerting direction

This is useful because the platform is being built incrementally.

The current phase defines the desired SLO structure.

A later phase replaces the placeholder expressions with real PromQL once the right metrics are validated.

---

## PrometheusRule File Location

The rule skeleton should be stored at:

```text
observability/prometheus/rules/bank-of-anthos-slo-rules.yaml
```

This location keeps SLO rules close to other observability configuration in the GitOps repository.

---

## PrometheusRule Skeleton

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: bank-of-anthos-slo-rules
  namespace: monitoring
  labels:
    release: kps
spec:
  groups:
    - name: bank-of-anthos.slo.candidates
      rules:
        - alert: BankOfAnthosFrontendAvailabilitySLOBreach
          expr: vector(0)
          for: 5m
          labels:
            severity: warning
            service: frontend
            slo: frontend-availability
            namespace: fintech-workload
          annotations:
            summary: "Bank of Anthos frontend availability SLO breach"
            description: "Placeholder rule. Replace expr with real frontend availability SLI after metric discovery."

        - alert: BankOfAnthosFrontendLatencySLOBreach
          expr: vector(0)
          for: 5m
          labels:
            severity: warning
            service: frontend
            slo: frontend-latency
            namespace: fintech-workload
          annotations:
            summary: "Bank of Anthos frontend latency SLO breach"
            description: "Placeholder rule. Replace expr with real p95 latency SLI after metric discovery."

        - alert: BankOfAnthosTransactionSuccessSLOBreach
          expr: vector(0)
          for: 5m
          labels:
            severity: warning
            service: transaction
            slo: transaction-success
            namespace: fintech-workload
          annotations:
            summary: "Bank of Anthos transaction success SLO breach"
            description: "Placeholder rule. Replace expr with real transaction success SLI after metric discovery."
```

---

## Meaning of Each Placeholder Alert

### 1. `BankOfAnthosFrontendAvailabilitySLOBreach`

Purpose:

```text
Future alert for frontend availability degradation.
```

Candidate SLI:

```text
successful frontend requests / total frontend requests
```

Future real expression may use metrics from:

```text
Ingress
Frontend HTTP metrics
Cilium/Hubble HTTP metrics
Blackbox exporter
```

---

### 2. `BankOfAnthosFrontendLatencySLOBreach`

Purpose:

```text
Future alert for frontend latency degradation.
```

Candidate SLI:

```text
p95 frontend request latency
```

Future real expression may use histogram metrics such as:

```text
http_request_duration_seconds_bucket
frontend_request_duration_seconds_bucket
nginx_ingress_controller_request_duration_seconds_bucket
```

---

### 3. `BankOfAnthosTransactionSuccessSLOBreach`

Purpose:

```text
Future alert for degraded deposit/payment/login business actions.
```

Candidate SLI:

```text
successful transaction actions / total transaction attempts
```

Future real expression may use:

```text
application business metrics
log-derived temporary signals
OpenTelemetry instrumentation
custom /metrics endpoint
```

---

## How This Fits Into Phase 8

Phase 8 is not final alerting.

Phase 8 is discovery and design.

The phase answers:

```text
Which user journeys matter?
Which candidate SLOs represent those journeys?
Which metrics exist today?
Which metrics are missing?
Which measurement point should be added next?
```

The PrometheusRule skeleton is included to show the intended future alert structure, not to claim that final SLO measurement already exists.

---

## Phase 8 Workflow

The correct workflow is:

```text
1. Define critical user journeys
2. Define candidate SLIs/SLOs
3. Discover available metrics
4. Create safe PrometheusRule skeletons with vector(0)
5. Validate available metrics
6. Decide whether real rules can be implemented now
7. Replace vector(0) only after metric discovery
```

---

## Metric Discovery Queries

These queries should be run in Prometheus before replacing the placeholder expressions.

### Discover all workload metrics

```promql
count by (__name__) (
  {namespace="fintech-workload"}
)
```

### Discover HTTP/request metrics

```promql
count by (__name__) (
  {__name__=~".*http.*|.*request.*|.*response.*", namespace="fintech-workload"}
)
```

### Discover latency metrics

```promql
count by (__name__) (
  {__name__=~".*duration.*|.*latency.*|.*seconds.*", namespace="fintech-workload"}
)
```

### Discover ingress/gateway/edge metrics

```promql
count by (__name__) (
  {__name__=~".*ingress.*|.*nginx.*|.*gateway.*|.*http.*"}
)
```

### Discover Cilium/Hubble metrics

```promql
count by (__name__) (
  {__name__=~".*hubble.*|.*cilium.*"}
)
```

---

## Evidence File

Metric discovery results should be documented in:

```text
evidence/phase-8-sli-slo-discovery/validation.md
```

Suggested structure:

```md
# Phase 8 — SLI/SLO Discovery Validation

## Objective

Validate available metrics for Bank of Anthos SLI/SLO design.

## Workload namespace

fintech-workload

## Discovery query: all workload metrics

```promql
count by (__name__) (
  {namespace="fintech-workload"}
)
```

### Result

Pending validation.

### Interpretation

Pending.

## Discovery query: HTTP/request metrics

```promql
count by (__name__) (
  {__name__=~".*http.*|.*request.*|.*response.*", namespace="fintech-workload"}
)
```

### Result

Pending validation.

### Interpretation

Pending.

## Discovery query: latency metrics

```promql
count by (__name__) (
  {__name__=~".*duration.*|.*latency.*|.*seconds.*", namespace="fintech-workload"}
)
```

### Result

Pending validation.

### Interpretation

Pending.

## Initial conclusion

Pending.
```

---

## How to Interpret Discovery Results

### Case A — HTTP/latency metrics exist

If HTTP request counters and latency histograms exist, Phase 9 can implement real SLO PrometheusRules.

Example future availability expression:

```promql
sum(rate(http_requests_total{namespace="fintech-workload", status!~"5.."}[5m]))
/
sum(rate(http_requests_total{namespace="fintech-workload"}[5m]))
< 0.99
```

Example future p95 latency expression:

```promql
histogram_quantile(
  0.95,
  sum by (le) (
    rate(http_request_duration_seconds_bucket{namespace="fintech-workload"}[5m])
  )
) > 1
```

### Case B — HTTP/latency metrics do not exist

If the right metrics do not exist, Phase 9 should add a measurement point.

Options:

- Ingress metrics
- Cilium/Hubble HTTP metrics
- OpenTelemetry instrumentation
- Blackbox exporter probes
- Application `/metrics` endpoint

This is still a good SRE outcome.

It shows that the team does not invent fake SLOs from whatever metrics happen to exist.

Instead, the team defines the user journey first and then adds the right measurement point.

---

## When to Replace `vector(0)`

Only replace `vector(0)` when:

```text
The metric exists.
The labels are confirmed.
The query returns meaningful results.
The result represents user impact.
The expression has been tested in Prometheus.
The evidence file documents the validation.
```

Do not replace `vector(0)` just because a metric looks related.

---

## Completion Criteria

This step is complete when:

```text
observability/prometheus/rules/bank-of-anthos-slo-rules.yaml exists
The rule file is committed to Git
Rules use vector(0) placeholders
No accidental alerts are firing
The purpose of the skeleton is documented
Metric discovery is documented separately
The next measurement decision is clear
```

---

## Final Design Decision

The PrometheusRule skeleton is intentionally safe.

It is not a fake production alert.

It is a GitOps-managed placeholder for future SLO alerting.

Prometheus will become the SLO measurement and alerting engine.

OpenSearch will support investigation and evidence collection.

The final SLO expressions will only be implemented after metric discovery confirms the correct user-facing metrics.

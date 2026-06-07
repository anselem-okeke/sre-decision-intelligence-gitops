# Phase 9 — Frontend Probe SLO Validation

## Objective

Validate that the Bank of Anthos frontend is measured through a real user-facing probe using Blackbox Exporter and Prometheus.

This phase creates the first practical SLO measurement point for the SRE Decision Intelligence Platform.

The goal is to confirm that the platform can answer:

- Is the Bank of Anthos frontend reachable?
- Is the frontend returning successful responses?
- How long does the frontend probe take?
- Can Prometheus evaluate availability and latency SLO signals?
- Can these signals later be used as Decision Intelligence triggers?

---

## Context

In Phase 8, SLI/SLO discovery showed that the `fintech-workload` namespace did not currently expose clean application-level HTTP metrics such as:

- `http_requests_total`
- `http_request_duration_seconds_bucket`
- `request_errors_total`

Because of that, the first user-facing SLO was implemented using a Blackbox Exporter HTTP probe against the Bank of Anthos frontend service.

This avoids incorrectly using infrastructure metrics like CPU, memory, pod restarts, or deployment replicas as direct SLOs.

Those platform metrics remain useful as investigation signals, but the SLO should measure user-facing reachability and responsiveness.

---

## Architecture

```text
Bank of Anthos Frontend Service
        ↓
Blackbox Exporter HTTP Probe
        ↓
Prometheus Probe Metrics
        ↓
PrometheusRule SLO Alerts
        ↓
Decision Intelligence Trigger later
```

---

## Measurement Target

```text
http://frontend.fintech-workload.svc.cluster.local
```

This target represents the in-cluster service endpoint for the Bank of Anthos frontend.

---

## Kubernetes Namespace Layout

| Component | Namespace |
|---|---|
| Bank of Anthos workload | `fintech-workload` |
| Blackbox Exporter | `monitoring` |
| Prometheus Probe resource | `monitoring` |
| PrometheusRule SLO rules | `monitoring` |

---

## SLI Definitions

### 1. Frontend Availability SLI

Availability is measured using:

```promql
probe_success{job="bank-of-anthos-frontend"}
```

Meaning:

| Value | Meaning |
|---|---|
| `1` | Probe succeeded |
| `0` | Probe failed |

Availability over a short SLO window is measured using:

```promql
avg_over_time(probe_success{job="bank-of-anthos-frontend"}[5m])
```

---

### 2. Frontend Latency SLI

Latency is measured using:

```promql
probe_duration_seconds{job="bank-of-anthos-frontend"}
```

This measures the probe duration in seconds.

A p95-style short-window latency query can be evaluated using:

```promql
quantile_over_time(
  0.95,
  probe_duration_seconds{job="bank-of-anthos-frontend"}[5m]
)
```

---

## Initial SLO Targets

| SLO | Query | Target |
|---|---|---|
| Frontend availability | `avg_over_time(probe_success{job="bank-of-anthos-frontend"}[5m])` | `>= 0.99` |
| Frontend latency | `quantile_over_time(0.95, probe_duration_seconds{job="bank-of-anthos-frontend"}[5m])` | `<= 1s` |

These are initial lab targets and can be tuned later based on observed baseline behavior.

---

## Validation Commands and Results

### 1. Blackbox Exporter Pod Validation

Command:

```bash
kubectl get pods -n monitoring | grep -i blackbox
```

Observed result:

```text
blackbox-exporter-prometheus-blackbox-exporter-6c6c9f8d75-lvh7t   1/1     Running   0               60m
```

Validation result:

```text
PASS
```

Blackbox Exporter is running successfully in the `monitoring` namespace.

---

### 2. Probe Resource Validation

Command:

```bash
kubectl get probe -n monitoring
```

Observed result:

```text
NAME                      AGE
bank-of-anthos-frontend   22m
```

Validation result:

```text
PASS
```

The `bank-of-anthos-frontend` Probe resource exists in the `monitoring` namespace.

---

### 3. PrometheusRule Validation

Command:

```bash
kubectl get prometheusrule -n monitoring | grep bank-of-anthos
```

Observed result:

```text
bank-of-anthos-frontend-slo-rules                                 22m
```

Validation result:

```text
PASS
```

The PrometheusRule containing the frontend SLO alert definitions exists.

---

### 4. Probe Success Metric Validation

Prometheus query:

```promql
probe_success{job="bank-of-anthos-frontend"}
```

Observed result:

```text
Result series: 1

probe_success{
  instance="http://frontend.fintech-workload.svc.cluster.local",
  job="bank-of-anthos-frontend",
  namespace="monitoring"
}
```

Validation result:

```text
PASS
```

Prometheus is scraping probe success data for the Bank of Anthos frontend.

---

### 5. Frontend Availability Query Validation

Prometheus query:

```promql
avg_over_time(probe_success{job="bank-of-anthos-frontend"}[5m])
```

Observed result:

```text
Result series: 1

{
  instance="http://frontend.fintech-workload.svc.cluster.local",
  job="bank-of-anthos-frontend",
  namespace="monitoring"
}
```

Validation result:

```text
PASS
```

The frontend availability SLI query returns data.

---

### 6. Frontend Latency Metric Validation

Prometheus query:

```promql
probe_duration_seconds{job="bank-of-anthos-frontend"}
```

Observed result:

```text
Result series: 1

probe_duration_seconds{
  instance="http://frontend.fintech-workload.svc.cluster.local",
  job="bank-of-anthos-frontend",
  namespace="monitoring"
}
```

Validation result:

```text
PASS
```

The frontend latency SLI metric returns data.

---

## Phase 9 Success Criteria

| Criteria | Status |
|---|---|
| Blackbox Exporter runs in `monitoring` namespace | PASS |
| Probe resource exists | PASS |
| `probe_success` metric exists for `bank-of-anthos-frontend` | PASS |
| `probe_duration_seconds` metric exists for `bank-of-anthos-frontend` | PASS |
| Frontend availability query returns data | PASS |
| Frontend latency query returns data | PASS |
| PrometheusRule exists | PASS |
| Evidence is documented | PASS |

---

## Final Phase 9 Conclusion

Phase 9 is complete.

The SRE Decision Intelligence Platform now has its first real user-facing SLO measurement point for Bank of Anthos.

The frontend can now be evaluated using:

```promql
probe_success{job="bank-of-anthos-frontend"}
```

for availability, and:

```promql
probe_duration_seconds{job="bank-of-anthos-frontend"}
```

for latency.

This is a stronger SRE design than using pod status alone because it verifies the actual frontend service path.

A pod can be running while the frontend endpoint is unavailable. The probe detects that user-facing failure condition.

---

## Decision Intelligence Relevance

This SLO can later trigger a Decision Intelligence workflow.

Example future flow:

```text
Frontend availability SLO breach
        ↓
Check OpenSearch logs for frontend errors
        ↓
Check Prometheus for pod restarts and readiness
        ↓
Check Argo CD for recent sync/change activity
        ↓
Check platform signals such as Cilium, Longhorn, and node health
        ↓
Generate impact, evidence, likely root cause, and safe action
```

---

## Next Phase

```text
Phase 10 — First Incident Scenario: Frontend Availability Breach
```

The next phase should intentionally simulate or create a controlled frontend availability problem and validate that the SLO detects it.

Phase 10 should produce evidence showing:

- the frontend probe failing,
- the availability SLO query dropping,
- the Prometheus alert becoming active,
- related workload/platform evidence,
- a clear incident narrative for the future Decision Intelligence engine.

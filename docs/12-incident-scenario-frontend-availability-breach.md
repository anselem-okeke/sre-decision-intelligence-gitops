# Incident Scenario — Frontend Availability Breach

## Objective

This scenario validates that the SRE Decision Intelligence Platform can detect a user-facing frontend availability incident.

## Scenario

The Bank of Anthos frontend Service selector is intentionally changed so that it no longer matches the frontend pods.

This causes the frontend service endpoint to fail while the frontend pods remain running.

## Why this is useful

This incident demonstrates the difference between infrastructure health and user-facing health.

A pod can be running and ready, but the user-facing service path can still be broken.

The Blackbox Exporter probe detects this condition because it checks the frontend endpoint directly.

## Incident Flow

```text
Healthy frontend service
        ↓
Frontend Service selector changed
        ↓
Service endpoints become empty
        ↓
Blackbox probe fails
        ↓
probe_success becomes 0
        ↓
Frontend availability SLO breaches
        ↓
Service selector restored
        ↓
Probe recovers
```

## Primary SLO Signal

```promql
probe_success{job="bank-of-anthos-frontend"}
```

## Availability Query

```promql
avg_over_time(probe_success{job="bank-of-anthos-frontend"}[5m])
```

## Alert

```text
BankOfAnthosFrontendAvailabilitySLOBreach
```

## Supporting Evidence

| Evidence | Source | Purpose |
|---|---|---|
| `probe_success` | Prometheus | Confirms frontend availability failure |
| `avg_over_time(probe_success[5m])` | Prometheus | Confirms SLO impact |
| `kubectl get endpoints frontend` | Kubernetes API | Confirms service routing failure |
| frontend pod readiness | Kubernetes / Prometheus | Shows pods may still be healthy |
| OpenSearch workload logs | OpenSearch | Adds workload context |
| Argo CD state later | Argo CD | Adds change context |

## Decision Intelligence Interpretation

Expected future output:

```text
Impact:
Bank of Anthos frontend is unavailable from the service endpoint.

Evidence:
The frontend probe failed.
The frontend Service has no endpoints.
Frontend pods are still running.
The issue is likely service routing, not pod crash.

Likely root cause:
Frontend Service selector no longer matches frontend pod labels.

Safe action:
Restore the frontend Service selector to match the frontend pod label app=frontend.
```

## Recovery

The incident is recovered by restoring the frontend Service selector.

```bash
kubectl patch svc frontend -n fintech-workload \
  -p '{"spec":{"selector":{"app":"frontend"}}}'
```

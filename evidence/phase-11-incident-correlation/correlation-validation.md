# Phase 11 — Incident Correlation Validation

## Objective

Correlate multiple signals from the frontend availability breach incident.

The goal is to move beyond alert detection and explain the likely cause.

---

## Incident Summary

A controlled incident was created by modifying the frontend Service selector so that it no longer matched the frontend pod labels.

This caused the frontend Service to lose all endpoints while the frontend pod stayed running.

---

## Correlated Evidence

| Evidence | Source | Observation | Meaning |
|---|---|---|---|
| Probe success | Prometheus | `probe_success = 0` | Frontend endpoint unavailable |
| Availability SLO | Prometheus | `avg_over_time(...) = 0.7` | SLO below 99% target |
| Alert state | Prometheus | `alertstate="pending"` | SLO alert evaluated |
| Frontend endpoints | Kubernetes API | `frontend <none>` | Service has no backend pod |
| Frontend pod | Kubernetes API | `1/1 Running` | Pod itself was healthy |
| Frontend recovery | Prometheus/Kubernetes | `probe_success = 1` after selector restored | Safe action worked |

---

## Correlation Logic

```text
IF probe_success = 0
AND frontend endpoints = <none>
AND frontend pod is Running/Ready
AND frontend pod restart count did not increase
THEN likely root cause = Service selector/routing mismatch
```

---

## Key Finding

The incident was not caused by a frontend pod crash.

The frontend pod stayed running, but the frontend Service had no endpoints.

This means the user-facing service path was broken even though the workload pod was healthy.

---

## Final Result

```text
The platform successfully correlated user-facing impact with Kubernetes routing evidence.

The frontend availability SLO detected the incident.
Kubernetes endpoint evidence explained the cause.
The safe action was to restore the frontend Service selector.
```

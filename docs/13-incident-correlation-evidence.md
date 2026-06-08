# Incident Correlation Evidence

## Objective

This phase documents how the SRE Decision Intelligence Platform correlates multiple observability signals during an incident.

Phase 10 proved that the frontend availability SLO can detect a user-facing incident.

Phase 11 explains the incident by connecting SLO, Kubernetes, Prometheus, and log evidence.

---

## Scenario

The Bank of Anthos frontend Service selector was intentionally modified so that it no longer matched the frontend pod labels.

This created a controlled frontend availability breach.

---

## Why correlation matters

An alert alone is not enough.

A useful SRE platform should answer:

- What is the user impact?
- What evidence supports the alert?
- Is the application pod unhealthy?
- Is the service path broken?
- Was there a recent change?
- What action is safe?

---

## Signal Sources

| Source | Role |
|---|---|
| Prometheus | SLO breach, probe success, alert state, pod/deployment metrics |
| Kubernetes API | Service selector, endpoints, pod readiness |
| OpenSearch | Workload and platform log context |
| Argo CD | GitOps/change context later |
| Cilium/Longhorn | Network/storage context later |

---

## Correlation Model

```text
SLO breach
    ↓
Check user-facing probe
    ↓
Check Kubernetes Service endpoints
    ↓
Check pod readiness/restarts
    ↓
Check logs
    ↓
Check GitOps/change state
    ↓
Produce impact, evidence, root cause, safe action
```

---

## Incident Correlation Result

The frontend probe failed because the frontend Service had no endpoints.

The frontend pod remained running.

This means the incident was not caused by a pod crash.

The likely root cause was a frontend Service selector mismatch.

---

## Safe Action

Restore the frontend Service selector so that it matches frontend pod labels.

---

## Evidence Files

```text
evidence/phase-11-incident-correlation/correlation-validation.md
evidence/phase-11-incident-correlation/prometheus-evidence.md
evidence/phase-11-incident-correlation/kubernetes-evidence.md
evidence/phase-11-incident-correlation/opensearch-evidence.md
evidence/phase-11-incident-correlation/decision-summary.md
```

---

## Phase 11 Completion Criteria

This phase is complete when:

- Prometheus SLO evidence is documented
- Kubernetes Service/Endpoint evidence is documented
- OpenSearch log context is documented
- Correlation logic is documented
- Decision summary is documented# Incident Correlation Evidence

## Objective

This phase documents how the SRE Decision Intelligence Platform correlates multiple observability signals during an incident.

Phase 10 proved that the frontend availability SLO can detect a user-facing incident.

Phase 11 explains the incident by connecting SLO, Kubernetes, Prometheus, and log evidence.

---

## Scenario

The Bank of Anthos frontend Service selector was intentionally modified so that it no longer matched the frontend pod labels.

This created a controlled frontend availability breach.

---

## Why correlation matters

An alert alone is not enough.

A useful SRE platform should answer:

- What is the user impact?
- What evidence supports the alert?
- Is the application pod unhealthy?
- Is the service path broken?
- Was there a recent change?
- What action is safe?

---

## Signal Sources

| Source | Role |
|---|---|
| Prometheus | SLO breach, probe success, alert state, pod/deployment metrics |
| Kubernetes API | Service selector, endpoints, pod readiness |
| OpenSearch | Workload and platform log context |
| Argo CD | GitOps/change context later |
| Cilium/Longhorn | Network/storage context later |

---

## Correlation Model

```text
SLO breach
    ↓
Check user-facing probe
    ↓
Check Kubernetes Service endpoints
    ↓
Check pod readiness/restarts
    ↓
Check logs
    ↓
Check GitOps/change state
    ↓
Produce impact, evidence, root cause, safe action
```

---

## Incident Correlation Result

The frontend probe failed because the frontend Service had no endpoints.

The frontend pod remained running.

This means the incident was not caused by a pod crash.

The likely root cause was a frontend Service selector mismatch.

---

## Safe Action

Restore the frontend Service selector so that it matches frontend pod labels.

---

## Evidence Files

```text
evidence/phase-11-incident-correlation/correlation-validation.md
evidence/phase-11-incident-correlation/prometheus-evidence.md
evidence/phase-11-incident-correlation/kubernetes-evidence.md
evidence/phase-11-incident-correlation/opensearch-evidence.md
evidence/phase-11-incident-correlation/decision-summary.md
```

---

## Phase 11 Completion Criteria

This phase is complete when:

- Prometheus SLO evidence is documented
- Kubernetes Service/Endpoint evidence is documented
- OpenSearch log context is documented
- Correlation logic is documented
- Decision summary is documented# Incident Correlation Evidence

## Objective

This phase documents how the SRE Decision Intelligence Platform correlates multiple observability signals during an incident.

Phase 10 proved that the frontend availability SLO can detect a user-facing incident.

Phase 11 explains the incident by connecting SLO, Kubernetes, Prometheus, and log evidence.

---

## Scenario

The Bank of Anthos frontend Service selector was intentionally modified so that it no longer matched the frontend pod labels.

This created a controlled frontend availability breach.

---

## Why correlation matters

An alert alone is not enough.

A useful SRE platform should answer:

- What is the user impact?
- What evidence supports the alert?
- Is the application pod unhealthy?
- Is the service path broken?
- Was there a recent change?
- What action is safe?

---

## Signal Sources

| Source | Role |
|---|---|
| Prometheus | SLO breach, probe success, alert state, pod/deployment metrics |
| Kubernetes API | Service selector, endpoints, pod readiness |
| OpenSearch | Workload and platform log context |
| Argo CD | GitOps/change context later |
| Cilium/Longhorn | Network/storage context later |

---

## Correlation Model

```text
SLO breach
    ↓
Check user-facing probe
    ↓
Check Kubernetes Service endpoints
    ↓
Check pod readiness/restarts
    ↓
Check logs
    ↓
Check GitOps/change state
    ↓
Produce impact, evidence, root cause, safe action
```

---

## Incident Correlation Result

The frontend probe failed because the frontend Service had no endpoints.

The frontend pod remained running.

This means the incident was not caused by a pod crash.

The likely root cause was a frontend Service selector mismatch.

---

## Safe Action

Restore the frontend Service selector so that it matches frontend pod labels.

---

## Evidence Files

```text
evidence/phase-11-incident-correlation/correlation-validation.md
evidence/phase-11-incident-correlation/prometheus-evidence.md
evidence/phase-11-incident-correlation/kubernetes-evidence.md
evidence/phase-11-incident-correlation/opensearch-evidence.md
evidence/phase-11-incident-correlation/decision-summary.md
```

---

## Phase 11 Completion Criteria

This phase is complete when:

- Prometheus SLO evidence is documented
- Kubernetes Service/Endpoint evidence is documented
- OpenSearch log context is documented
- Correlation logic is documented
- Decision summary is documented

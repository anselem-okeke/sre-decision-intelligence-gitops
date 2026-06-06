# Platform Signal Inventory

## Objective

This document defines the signal inventory for the SRE Decision Intelligence Platform.

The purpose is to clearly separate:

- workload impact signals
- platform runtime signals
- GitOps/change signals
- observability pipeline signals

The future Decision Intelligence API will use this signal catalog to correlate incidents.

---

## Signal Domains

```text
Workload signals
        +
Platform signals
        +
GitOps signals
        +
Observability pipeline signals
        =
Decision Intelligence
```

---

## 1. Workload Impact Signals

Workload signals come from Bank of Anthos.

They help answer:

- Are users affected?
- Are transactions failing?
- Is frontend latency increasing?
- Are backend services returning errors?

| Signal | Source | Purpose |
|---|---|---|
| Application severity | OpenSearch `severity` | Identify INFO/WARN/ERROR logs |
| Application message | OpenSearch `message` | Understand business/application events |
| Frontend logs | OpenSearch Kubernetes metadata | Identify frontend behavior |
| Payment/deposit/login logs | OpenSearch message search | Track business actions |
| Pod restarts | Prometheus / kube-state-metrics | Detect workload instability |
| Pod readiness | Prometheus / kube-state-metrics | Detect unhealthy workload pods |
| Deployment availability | Prometheus / kube-state-metrics | Detect unavailable workloads |

---

## 2. Platform Runtime Signals

Platform signals come from Kubernetes and cluster components.

They help answer:

- Did a pod restart?
- Did scheduling fail?
- Did a node become unhealthy?
- Is there resource pressure?
- Is the platform contributing to the incident?

| Signal | Source | Purpose |
|---|---|---|
| Pod restart count | Prometheus `kube_pod_container_status_restarts_total` | Runtime instability |
| Pod phase | Prometheus `kube_pod_status_phase` | Pending/Failed/Unknown pods |
| Container readiness | Prometheus `kube_pod_container_status_ready` | Unready containers |
| Deployment unavailable replicas | Prometheus `kube_deployment_status_replicas_unavailable` | Workload availability |
| CPU usage | Prometheus `container_cpu_usage_seconds_total` | Resource pressure |
| Memory usage | Prometheus `container_memory_working_set_bytes` | Resource pressure |
| OOM events | Prometheus `container_oom_events_total` | Memory failure |
| Network errors | Prometheus container network counters | Network pressure/context |

---

## 3. GitOps / Change Signals

GitOps signals come from Argo CD.

They help answer:

- What changed?
- Was there a recent sync?
- Is the app out of sync?
- Did a deployment fail?
- Did GitOps introduce the incident?

| Signal | Source | Purpose |
|---|---|---|
| Argo CD sync logs | OpenSearch `argocd` namespace logs | Deployment/change context |
| Argo CD application status | Argo CD API later | Sync and health state |
| Git revision | Argo CD API later | Identify deployed commit |
| OutOfSync state | Argo CD API later | Detect drift |
| Failed sync | Argo CD logs/API | Detect deployment failure |

---

## 4. Observability Pipeline Signals

Observability pipeline signals confirm whether monitoring/logging itself is healthy.

They help answer:

- Is Fluent Bit shipping logs?
- Is OpenSearch receiving logs?
- Is the monitoring system degraded?
- Can we trust the telemetry?

| Signal | Source | Purpose |
|---|---|---|
| Fluent Bit logs | OpenSearch `observability` namespace | Log pipeline health |
| OpenSearch logs | OpenSearch / Kubernetes logs | Storage/search health |
| Prometheus health | Prometheus / Kubernetes | Metrics pipeline health |
| Grafana health | Kubernetes / Prometheus | Dashboard availability |

---

## 5. Storage Signals

Storage signals are important because the cluster uses Longhorn.

They help answer:

- Are volumes healthy?
- Are PVCs attached?
- Did a mount fail?
- Is storage degradation affecting workloads?

| Signal | Source | Purpose |
|---|---|---|
| Longhorn logs | OpenSearch `longhorn-system` namespace | Volume health context |
| PVC events | Kubernetes API later | Mount/attach failures |
| Volume health | Longhorn metrics/API later | Storage health |

---

## 6. Network Signals

Network signals are important because the cluster uses Cilium.

They help answer:

- Is traffic being dropped?
- Is a policy blocking communication?
- Are DNS/network paths failing?

| Signal | Source | Purpose |
|---|---|---|
| Cilium logs | OpenSearch `kube-system` namespace | Network context |
| Hubble flows | Hubble later | Service-to-service visibility |
| Policy drops | Cilium/Hubble later | Network policy enforcement |

---

## Decision Intelligence Usage

The future Decision Intelligence API will use this signal model like this:

```text
1. Start with user/workload impact
2. Check workload logs and metrics
3. Check platform runtime state
4. Check GitOps/change context
5. Check observability pipeline health
6. Generate incident summary
```

Example output:

```text
Impact:
Frontend users are affected.

Evidence:
Frontend ERROR logs increased.
The frontend deployment has unavailable replicas.
Argo CD synced the workload shortly before the incident.
No node-level failure was detected.

Likely root cause:
Recent workload rollout introduced frontend instability.

Safe action:
Rollback the latest Argo CD sync and monitor frontend SLO recovery.
```

---

## Query Library

Prometheus queries:

```text
observability/prometheus/queries/platform-signals.promql
observability/prometheus/queries/workload-signals.promql
```

OpenSearch queries:

```text
observability/opensearch/queries/platform-signal-logs.md
```

---

## Phase 7 Completion Criteria

This phase is complete when:

- Platform signal inventory is documented
- Workload signal queries are documented
- Platform Prometheus queries are documented
- Platform OpenSearch queries are documented
- At least one workload and one platform signal are validated
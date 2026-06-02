# Signal Model

## Purpose

The SRE Decision Intelligence Platform collects signals from both the workload and the platform.

The goal is not only to observe an application, but to correlate user impact with Kubernetes platform context.

## Signal Domains

### 1. Workload Signals

Workload signals come from Bank of Anthos.

They answer:

- Are users affected?
- Are transactions failing?
- Is frontend latency increasing?
- Are backend services returning errors?

Examples:

- frontend 5xx rate
- frontend latency
- transaction errors
- service timeout logs
- ledger/database errors

### 2. Platform Signals

Platform signals come from the Kubernetes environment.

They answer:

- Is the platform contributing to the incident?
- Did a pod restart?
- Did a node become unhealthy?
- Was there a recent rollout?
- Did scheduling, networking, or storage fail?

Examples:

- pod restarts
- CrashLoopBackOff
- FailedScheduling
- NodeNotReady
- CPU/memory pressure
- PVC mount failures
- Argo CD sync events
- Cilium/Hubble drops
- Longhorn volume health

## Decision Intelligence Flow

```text
Workload SLO breach
        +
Platform context
        =
Impact → Evidence → Root Cause → Safe Action
```


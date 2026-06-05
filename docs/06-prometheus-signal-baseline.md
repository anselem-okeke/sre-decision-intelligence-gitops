# Prometheus Signal Baseline

## Purpose

This document defines the first Prometheus signal baseline for the SRE Decision Intelligence Platform.

The goal is to separate platform context signals from workload/user-impact signals.

## Current State

Prometheus and Grafana are already running in the Talos Kubernetes cluster.

The current metrics confirm availability of Kubernetes platform signals such as:

- pod restarts
- pod phase
- deployment availability
- container CPU usage
- container memory usage
- OOM events
- container network counters

## Signal Categories

### Platform Signals

Platform signals explain runtime or infrastructure context.

Examples:

- pod restarts
- CrashLoopBackOff / waiting state
- deployment unavailable replicas
- CPU saturation
- memory pressure
- OOM events
- network errors

### Workload Signals

Workload signals describe user or business impact.

Examples:

- frontend request success rate
- frontend latency
- transaction success rate
- backend dependency errors

## Current Gap

> The current Prometheus metric sample confirms platform metrics.
> The next discovery step is to verify whether Bank of Anthos exposes user-facing request metrics that can support SLIs/SLOs.

[//]: # (## Next Step)

[//]: # ()
[//]: # (Run workload discovery queries against the `fintech-workload` namespace and identify whether HTTP, gRPC, ingress, or Cilium/Hubble request metrics exist.)

# Project Overview

## Project Name

SRE Decision Intelligence Platform

## Purpose

This project demonstrates how an SRE platform can move beyond raw observability data and produce actionable incident decisions.

The platform is designed to answer five operational questions during incidents:

- What changed?
- Who is affected?
- Where is the failure spreading?
- What is the likely root cause?
- What action is safe now?

## Problem Statement

Modern Kubernetes platforms generate large volumes of telemetry from metrics, logs, events, dashboards, traces, and deployment systems.

However, during incidents, teams often still face:

- too many alerts
- too many dashboards
- unclear user impact
- weak root-cause visibility
- no clear safe next action

This project focuses on turning telemetry into decision intelligence.

## Target Workload

The project uses Bank of Anthos as the realistic workload.

Bank of Anthos provides a fintech-style service flow that can be used to observe user-facing availability, latency, backend errors, and transaction-related failures.

## Platform Scope

This project observes both:

1. Workload signals from Bank of Anthos
2. Platform signals from Kubernetes and platform components

## Core Components

| Component | Role |
|---|---|
| Argo CD | GitOps deployment control |
| Bank of Anthos | Production-style workload |
| Prometheus | Metrics and SLO signals |
| Fluent Bit | Log collection |
| OpenSearch | Log storage and search |
| Grafana | Dashboards and visualization |
| Kubernetes API | Runtime and event context |
| Decision Intelligence API | Correlation and incident decision engine |

## Target Outcome

The final platform should generate incident summaries such as:

```text
Incident:
Bank of Anthos frontend degradation

Impact:
Users are experiencing failed or slow requests.

Evidence:
Frontend 5xx rate increased.
Backend service logs show dependency timeouts.
Kubernetes events show pod restarts.
Argo CD shows a recent sync.

Likely root cause:
Recent rollout introduced backend instability.

Safe action:
Rollback the latest deployment and monitor SLO recovery.
```
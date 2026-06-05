# OpenSearch Log Investigation Layer

## Objective

> Log investigation layer for the SRE Decision Intelligence Platform.
> The goal is to make Kubernetes logs searchable and useful for incident analysis.

## Why this matters

Raw log ingestion is not enough.

For decision intelligence, logs must help answer:

- Which service produced the error?
- Which namespace is affected?
- Which pod and node are involved?
- Did the issue affect only the workload or also the platform?
- What happened around the incident time window?

## Log Pipeline

![img](/img/opensearchinvestigation.png)

```text
Kubernetes Pods
        ↓
Fluent Bit
        ↓
OpenSearch
        ↓
Investigation Queries
        ↓
Decision Intelligence API later
```

## Signal Domains

The log investigation layer covers two signal domains.

| Domain | Examples | Purpose |
|---|---|---|
| Workload logs | Bank of Anthos frontend, transactions, login, payment, deposit | Understand user/business impact |
| Platform logs | Argo CD, kube-system, Longhorn, Fluent Bit, OpenSearch | Understand runtime/platform context |

## Current Working Index Pattern

```text
k8s-logs-*
```

## Current Workload Namespace

```text
fintech-workload
```

## Confirmed Fields

Current OpenSearch documents include useful Kubernetes metadata:

- `@timestamp`
- `message`
- `kubernetes.namespace_name`
- `kubernetes.pod_name`
- `kubernetes.labels.app`
- `kubernetes.labels.application`
- `kubernetes.labels.team`
- `kubernetes.host`
- `kubernetes.container_name`

## Current Limitation

Some Bank of Anthos application logs are embedded as JSON strings inside the `message` field.

Example:

```json
{
  "message": "{\"timestamp\": \"2026-06-05 10:24:41\", \"message\": \"deposit | Deposit submitted successfully.\", \"severity\": \"INFO\"}"
}
```

This is searchable, but not ideal.

A later improvement will parse this nested JSON into structured fields such as:

- `app_timestamp`
- `app_message`
- `app_severity`

## Query Library

The query library is stored in:

```text
observability/opensearch/queries/
```

Files:

```text
workload-log-queries.md
platform-log-queries.md
```

## Completion Criteria

- Workload log queries are documented
- Platform log queries are documented
- Error-like searches work
- Time-window searches work
- Evidence is captured

# Structured Log Parsing

## Objective

This phase improves the Fluent Bit to OpenSearch logging pipeline by exposing application log payloads as searchable structured fields.

The goal is to support future Decision Intelligence queries over application events without relying on raw text search only.

## Current State

Bank of Anthos frontend logs are collected cluster-wide by Fluent Bit and stored in OpenSearch.

The current working fields are:

- `timestamp`
- `message`
- `severity`
- `kubernetes.namespace_name`
- `kubernetes.pod_name`
- `kubernetes.container_name`

Example current document:

```json
{
  "timestamp": "2026-06-06 02:35:50",
  "message": "deposit | Deposit submitted successfully.",
  "severity": "INFO",
  "kubernetes": {
    "namespace_name": "fintech-workload",
    "pod_name": "frontend-67dd44c5c9-zsjc9",
    "container_name": "front"
  }
}
```

## Target Structure

This phase adds enterprise-friendly application fields:

- `app_timestamp`
- `app_message`
- `app_severity`

Example target document:

```json
{
  "timestamp": "2026-06-06 02:35:50",
  "message": "deposit | Deposit submitted successfully.",
  "severity": "INFO",
  "app_timestamp": "2026-06-06 02:35:50",
  "app_message": "deposit | Deposit submitted successfully.",
  "app_severity": "INFO",
  "kubernetes": {
    "namespace_name": "fintech-workload",
    "pod_name": "frontend-67dd44c5c9-zsjc9",
    "container_name": "front"
  }
}
```

## Implementation

Fluent Bit uses a Lua filter to normalize application logs.

Pipeline:

```text
Kubernetes container logs
        ↓
Fluent Bit tail input
        ↓
Kubernetes metadata filter
        ↓
Lua normalization filter
        ↓
OpenSearch output
        ↓
Investigation queries
        ↓
Decision Intelligence API
```

## Fluent Bit Lua Normalization

The existing `normalize.lua` script should keep the currently working fields and add `app_*` aliases.

```lua
function normalize(tag, timestamp, record)
    local cjson = require("cjson")

    local raw_message = record["message"]

    if raw_message ~= nil then
        local ok, parsed = pcall(cjson.decode, raw_message)

        if ok and parsed ~= nil then
            if parsed["timestamp"] ~= nil then
                record["timestamp"] = parsed["timestamp"]
                record["app_timestamp"] = parsed["timestamp"]
            end

            if parsed["message"] ~= nil then
                record["message"] = parsed["message"]
                record["app_message"] = parsed["message"]
            end

            if parsed["severity"] ~= nil then
                record["severity"] = parsed["severity"]
                record["app_severity"] = parsed["severity"]
            end
        end
    end

    -- If the record is already normalized, still create app_* aliases.
    if record["severity"] ~= nil and record["app_severity"] == nil then
        record["app_severity"] = record["severity"]
    end

    if record["message"] ~= nil and record["app_message"] == nil then
        record["app_message"] = record["message"]
    end

    if record["timestamp"] ~= nil and record["app_timestamp"] == nil then
        record["app_timestamp"] = record["timestamp"]
    end

    return 1, timestamp, record
end
```

## Fluent Bit Kubernetes Filter

Kubernetes labels and annotations must remain disabled.

```ini
[FILTER]
    Name                kubernetes
    Match               kube.*
    Kube_URL            https://kubernetes.default.svc:443
    Kube_CA_File        /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
    Kube_Token_File     /var/run/secrets/kubernetes.io/serviceaccount/token
    Merge_Log           On
    Keep_Log            On
    Labels              Off
    Annotations         Off
    K8S-Logging.Parser  Off
    K8S-Logging.Exclude Off
```

## Important Design Decision

Kubernetes labels and annotations are disabled in Fluent Bit:

```ini
Labels      Off
Annotations Off
```

This avoids OpenSearch mapping conflicts caused by dynamic Kubernetes label keys such as:

```text
kubernetes.labels.app
kubernetes.labels.app.kubernetes.io/name
```

For this phase, workload identity is based on:

```text
kubernetes.namespace_name
kubernetes.container_name
kubernetes.pod_name
```

not Kubernetes labels.

## Stable OpenSearch Query Identity

Use:

```text
kubernetes.namespace_name = fintech-workload
kubernetes.container_name = front
```

Do not use:

```text
kubernetes.labels.app = frontend
```

because labels are intentionally disabled to keep indexing stable.

## Why This Matters

Structured fields allow future investigation and decision-intelligence queries such as:

```text
Find ERROR logs from the frontend container in fintech-workload.
```

Instead of raw text search, the future API can query:

```text
kubernetes.namespace_name = fintech-workload
kubernetes.container_name = front
app_severity = ERROR
```

## Validation

Validation is done by querying OpenSearch for fresh documents where `app_severity` exists.

Old OpenSearch documents created before this phase will not contain `app_*` fields.

## Success Criteria

Phase 6 is complete when:

```text
Fluent Bit syncs successfully
Fluent Bit pods run without Lua errors
Fresh logs arrive in OpenSearch
OpenSearch documents contain app_timestamp
OpenSearch documents contain app_message
OpenSearch documents contain app_severity
Query by app_severity = INFO works
Evidence file contains real validation output
```

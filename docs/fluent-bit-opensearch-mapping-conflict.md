# Fluent Bit to OpenSearch Mapping Conflict

## Summary

During the implementation of cluster-wide Kubernetes log ingestion, Fluent Bit successfully collected container logs and forwarded them to OpenSearch, but OpenSearch rejected many bulk indexing requests with `mapper_parsing_exception`.

The main error was:

```text
object mapping for [kubernetes.labels.app] tried to parse field [app] as object, but found a concrete value
```

This affected reliable ingestion of fresh workload logs from the `fintech-workload` namespace and delayed structured log validation for the Decision Intelligence platform.

The final resolution involved stabilizing Fluent Bit metadata ingestion, reducing OpenSearch output pressure, rotating Fluent Bit backlog state, and validating fresh structured logs in OpenSearch.

---

## Context

The platform uses a cluster-wide logging architecture:

```text
Kubernetes workloads
        ↓
Container logs on nodes
        ↓
Fluent Bit DaemonSet
        ↓
Kubernetes metadata enrichment
        ↓
Lua normalization
        ↓
OpenSearch
        ↓
Dashboard / investigation queries
        ↓
Decision Intelligence API
```

The logging pipeline collects logs cluster-wide, while the first workload-level analysis focuses on:

```text
Namespace: fintech-workload
Application: Bank of Anthos
Primary frontend container: front
```

The goal was to support workload-focused queries such as:

```text
Find recent frontend application events from fintech-workload.
Find ERROR logs from the frontend container.
Group application logs by severity.
Use structured fields for future Decision Intelligence workflows.
```

---

## Problem

OpenSearch rejected Fluent Bit bulk writes.

The repeated error looked like this:

```text
status: 400
mapper_parsing_exception
object mapping for [kubernetes.labels.app] tried to parse field [app] as object, but found a concrete value
```

This meant Fluent Bit was sending records that OpenSearch could not index because of conflicting mappings under the same field path.

---

## Impact

The issue caused unreliable log ingestion into OpenSearch.

Observed impact:

- Fresh workload logs were not consistently indexed.
- OpenSearch queries for recent logs returned stale results or no results.
- Fluent Bit retried failed chunks repeatedly.
- Fluent Bit logs became noisy with indexing and retry errors.
- Structured log validation was blocked.
- Troubleshooting was confusing because multiple symptoms appeared at the same time.

The issue affected workload observability for:

```text
kubernetes.namespace_name = fintech-workload
kubernetes.container_name = front
```

---

## Symptoms Observed

### 1. OpenSearch bulk write failures

Fluent Bit output showed OpenSearch bulk indexing failures:

```text
status: 400
mapper_parsing_exception
object mapping for [kubernetes.labels.app] tried to parse field [app] as object, but found a concrete value
```

### 2. Fluent Bit retry warnings

Fluent Bit repeatedly retried failed chunks:

```text
failed to flush chunk
retry in X seconds
input=storage_backlog.1 > output=opensearch.0
```

### 3. Old records continued to fail after config changes

Even after the live Fluent Bit configuration was corrected, the same error continued appearing because Fluent Bit was retrying old filesystem-buffered chunks.

### 4. Query confusion

Some OpenSearch queries returned zero results because they used fields that did not yet exist, such as:

```text
app_severity
```

At that time, indexed documents only had:

```text
severity
message
timestamp
```

The `app_*` aliases belonged to the later structured parsing phase.

---

## Root Cause

The primary root cause was uncontrolled Kubernetes label metadata being sent into OpenSearch.

Kubernetes labels can contain both simple keys and dotted keys.

Example simple label:

```yaml
app: frontend
```

Example recommended Kubernetes-style label:

```yaml
app.kubernetes.io/name: frontend
```

When Fluent Bit sends these labels into OpenSearch, they can appear under:

```text
kubernetes.labels.app
kubernetes.labels.app.kubernetes.io/name
```

OpenSearch dynamic mapping can interpret dotted paths as object paths.

That created a conflict:

```text
kubernetes.labels.app = "frontend"
```

versus:

```text
kubernetes.labels.app = object
kubernetes.labels.app.kubernetes.io/name = "frontend"
```

OpenSearch cannot treat the same field as both a scalar value and an object path.

Therefore, OpenSearch rejected the documents with:

```text
mapper_parsing_exception
```

---

## Secondary Root Cause

After the live Fluent Bit configuration was fixed, the error still appeared.

The reason was Fluent Bit filesystem buffering.

Fluent Bit had already stored old chunks on disk before Kubernetes labels and annotations were disabled. Those old chunks still contained the problematic field:

```text
kubernetes.labels.app
```

The logs showed that Fluent Bit was retrying old backlog chunks:

```text
input=storage_backlog.1 > output=opensearch.0
```

This meant the current live configuration was fixed, but old buffered records were still being retried.

---

## Why This Was Painful

The issue was painful because it looked like one problem but was actually several stacked problems:

```text
1. OpenSearch rejected dynamic Kubernetes label mappings.
2. Fluent Bit kept retrying old filesystem-buffered chunks.
3. Some queries used fields that did not exist yet.
4. The application logs also required normalization for future app_* fields.
5. OpenSearch output pressure and buffer warnings added noise.
```

The correct fix required separating these layers instead of changing random settings.

---

## Investigation Process

### Step 1 — Confirm live Fluent Bit configuration

The live ConfigMap was checked with:

```bash
kubectl get cm -n observability fluent-bit -o yaml | grep -A30 "\[SERVICE\]"
kubectl get cm -n observability fluent-bit -o yaml | grep -A25 "\[INPUT\]"
kubectl get cm -n observability fluent-bit -o yaml | grep -A25 "\[FILTER\]"
kubectl get cm -n observability fluent-bit -o yaml | grep -A20 "\[OUTPUT\]"
```

This confirmed whether Argo CD and Helm values had actually reached the live Fluent Bit ConfigMap.

### Step 2 — Confirm Fluent Bit runtime errors

Fluent Bit logs were checked with:

```bash
kubectl logs -n observability -l app.kubernetes.io/name=fluent-bit --tail=200 | grep -Ei "mapper_parsing_exception|kubernetes.labels.app|storage_backlog|429|failed to flush|error|warn"
```

This showed the actual runtime failure:

```text
mapper_parsing_exception
kubernetes.labels.app
```

and later showed old backlog retries:

```text
storage_backlog.1
```

### Step 3 — Confirm actual indexed document shape

OpenSearch was queried to inspect real documents:

```bash
curl "http://localhost:9200/k8s-logs-*/_search?pretty" \
  -H 'Content-Type: application/json' \
  -d '{
    "size": 10,
    "sort": [
      { "@timestamp": { "order": "desc" }}
    ],
    "_source": [
      "@timestamp",
      "timestamp",
      "message",
      "severity",
      "kubernetes.namespace_name",
      "kubernetes.pod_name",
      "kubernetes.container_name"
    ],
    "query": {
      "bool": {
        "must": [
          { "match": { "kubernetes.namespace_name": "fintech-workload" }},
          { "match": { "kubernetes.container_name": "front" }}
        ]
      }
    }
  }'
```

This confirmed that fresh structured frontend logs were eventually indexed with:

```text
timestamp
message
severity
kubernetes.namespace_name
kubernetes.pod_name
kubernetes.container_name
```

### Step 4 — Avoid querying non-existent fields

Queries using:

```text
app_severity
```

returned zero because the field had not yet been created.

The correct field at that stage was:

```text
severity
```

The `app_severity` field belongs to the later structured parsing / aliasing phase.

---

## Resolution

The pipeline was stabilized with four main changes.

---

## Resolution Step 1 — Disable Kubernetes Labels and Annotations

Kubernetes labels and annotations were disabled in the Fluent Bit Kubernetes filter.

Final stable filter:

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

This prevented Fluent Bit from sending uncontrolled dynamic metadata into OpenSearch.

### Why this was necessary

The platform does not need all Kubernetes labels at this stage.

For the current workload-focused analysis, these metadata fields are enough:

```text
kubernetes.namespace_name
kubernetes.pod_name
kubernetes.container_name
```

The unstable fields were:

```text
kubernetes.labels.*
kubernetes.annotations.*
```

These were disabled to avoid OpenSearch mapping conflicts.

---

## Resolution Step 2 — Reduce OpenSearch Output Workers

The OpenSearch output worker count was reduced from:

```ini
Workers             4
```

to:

```ini
Workers             1
```

Final stable output:

```ini
[OUTPUT]
    Name                opensearch
    Match               kube.*
    Host                opensearch-cluster-master.observability.svc.cluster.local
    Port                9200
    Logstash_Format     On
    Logstash_Prefix     k8s-logs
    Suppress_Type_Name  On
    tls                 Off
    Buffer_Size         5M
    Workers             1
    Retry_Limit         False
    Trace_Error         Off
```

### Why this was necessary

The OpenSearch cluster was already under indexing pressure.

More output workers can increase concurrent bulk pressure.

Using one worker created a more stable ingestion baseline.

---

## Resolution Step 3 — Rotate Fluent Bit Storage Path and DB

After disabling labels and annotations, old errors still appeared because Fluent Bit retried old chunks.

The storage path and DB were rotated.

Old values:

```ini
storage.path              /var/log/flb-storage
DB                        /var/log/flb_kube_cluster.db
```

New values:

```ini
storage.path              /var/log/flb-storage-v2
DB                        /var/log/flb_kube_cluster_v2.db
```

Final stable service/input sections:

```ini
[SERVICE]
    Flush                     5
    Log_Level                 info
    Daemon                    off
    Parsers_File              /fluent-bit/etc/parsers.conf
    HTTP_Server               On
    HTTP_Listen               0.0.0.0
    HTTP_Port                 2020
    storage.path              /var/log/flb-storage-v2
    storage.sync              normal
    storage.checksum          off
    storage.backlog.mem_limit 100M
    storage.max_chunks_up     256

[INPUT]
    Name                tail
    Path                /var/log/containers/*.log
    Exclude_Path        /var/log/containers/*_observability_fluent-bit-*.log
    Parser              cri
    Tag                 kube.*
    Refresh_Interval    5
    Mem_Buf_Limit       256MB
    Skip_Long_Lines     On
    Read_from_Head      Off
    DB                  /var/log/flb_kube_cluster_v2.db
    DB.Sync             Normal
    Rotate_Wait         60
    storage.type        filesystem
```

### Why this was necessary

The old Fluent Bit filesystem buffer contained records created before the metadata fix.

Rotating the storage path and DB forced Fluent Bit to use fresh buffer state and prevented old poisoned records from being retried.

---

## Resolution Step 4 — Validate Fresh Structured Logs

Fresh frontend traffic was generated:

```bash
for i in {1..30}; do
  curl -s http://192.168.0.231/ > /dev/null
  sleep 1
done

sleep 30
```

Then OpenSearch was queried:

```bash
curl "http://localhost:9200/k8s-logs-*/_search?pretty" \
  -H 'Content-Type: application/json' \
  -d '{
    "size": 10,
    "sort": [
      { "@timestamp": { "order": "desc" }}
    ],
    "_source": [
      "@timestamp",
      "timestamp",
      "message",
      "severity",
      "kubernetes.namespace_name",
      "kubernetes.pod_name",
      "kubernetes.container_name"
    ],
    "query": {
      "bool": {
        "must": [
          { "match": { "kubernetes.namespace_name": "fintech-workload" }},
          { "match": { "kubernetes.container_name": "front" }}
        ]
      }
    }
  }'
```

Expected result shape:

```json
{
  "_source": {
    "@timestamp": "2026-06-06T02:35:50.535Z",
    "timestamp": "2026-06-06 02:35:50",
    "message": "deposit | Deposit submitted successfully.",
    "severity": "INFO",
    "kubernetes": {
      "namespace_name": "fintech-workload",
      "pod_name": "frontend-67dd44c5c9-zsjc9",
      "container_name": "front"
    }
  }
}
```

This validated that fresh workload logs were indexed successfully after the fix.

---

## Final Stable Fluent Bit Baseline

The known-good Fluent Bit baseline is:

```ini
[SERVICE]
    Flush                     5
    Log_Level                 info
    Daemon                    off
    Parsers_File              /fluent-bit/etc/parsers.conf
    HTTP_Server               On
    HTTP_Listen               0.0.0.0
    HTTP_Port                 2020
    storage.path              /var/log/flb-storage-v2
    storage.sync              normal
    storage.checksum          off
    storage.backlog.mem_limit 100M
    storage.max_chunks_up     256

[INPUT]
    Name                tail
    Path                /var/log/containers/*.log
    Exclude_Path        /var/log/containers/*_observability_fluent-bit-*.log
    Parser              cri
    Tag                 kube.*
    Refresh_Interval    5
    Mem_Buf_Limit       256MB
    Skip_Long_Lines     On
    Read_from_Head      Off
    DB                  /var/log/flb_kube_cluster_v2.db
    DB.Sync             Normal
    Rotate_Wait         60
    storage.type        filesystem

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

[FILTER]
    Name                lua
    Match               kube.*
    script              /fluent-bit/scripts/normalize.lua
    call                normalize

[OUTPUT]
    Name                opensearch
    Match               kube.*
    Host                opensearch-cluster-master.observability.svc.cluster.local
    Port                9200
    Logstash_Format     On
    Logstash_Prefix     k8s-logs
    Suppress_Type_Name  On
    tls                 Off
    Buffer_Size         5M
    Workers             1
    Retry_Limit         False
    Trace_Error         Off
```

---

## GitOps Workflow Used

Because the Fluent Bit Argo CD Application used inline Helm values, the live Application object needed to be updated before syncing the child application.

The working workflow was:

```bash
git diff
git add argocd/applications/fluent-bit-app.yaml
git commit -m "fix: stabilize Fluent Bit OpenSearch ingestion"
git push

kubectl apply -f argocd/applications/fluent-bit-app.yaml

argocd app sync fluent-bit

kubectl rollout restart daemonset/fluent-bit -n observability
kubectl rollout status daemonset/fluent-bit -n observability
```

### Why `kubectl apply` was needed

`argocd app sync fluent-bit` syncs the live `Application/fluent-bit` spec.

If the Argo CD Application object itself is not managed by a parent/root Argo CD app, updating the Git file alone does not update the live Application spec.

Therefore, the Application manifest was applied manually:

```bash
kubectl apply -f argocd/applications/fluent-bit-app.yaml
```

Then Argo CD rendered and synced Fluent Bit using the updated inline Helm values.

---

## Verification Commands

### Verify ConfigMap

```bash
kubectl get cm -n observability fluent-bit -o yaml | grep -A30 "\[SERVICE\]"
kubectl get cm -n observability fluent-bit -o yaml | grep -A25 "\[INPUT\]"
kubectl get cm -n observability fluent-bit -o yaml | grep -A25 "\[FILTER\]"
kubectl get cm -n observability fluent-bit -o yaml | grep -A20 "\[OUTPUT\]"
```

Expected key values:

```ini
Read_from_Head      Off
DB                  /var/log/flb_kube_cluster_v2.db
storage.path        /var/log/flb-storage-v2
storage.type        filesystem
Labels              Off
Annotations         Off
Buffer_Size         5M
Workers             1
Trace_Error         Off
```

### Verify Fluent Bit errors

```bash
kubectl logs -n observability -l app.kubernetes.io/name=fluent-bit --tail=200 | grep -Ei "mapper_parsing_exception|kubernetes.labels.app|storage_backlog|429|failed to flush|error|warn"
```

Expected:

```text
No continuous mapper_parsing_exception for kubernetes.labels.app
No continuous failed-to-flush flood
```

Some non-blocking warnings may appear, such as:

```text
http_client cannot increase buffer
```

These should be evaluated separately from the original mapping conflict.

### Verify OpenSearch fresh logs

```bash
curl "http://localhost:9200/k8s-logs-*/_search?pretty" \
  -H 'Content-Type: application/json' \
  -d '{
    "size": 10,
    "sort": [
      { "@timestamp": { "order": "desc" }}
    ],
    "_source": [
      "@timestamp",
      "timestamp",
      "message",
      "severity",
      "kubernetes.namespace_name",
      "kubernetes.pod_name",
      "kubernetes.container_name"
    ],
    "query": {
      "bool": {
        "must": [
          { "match": { "kubernetes.namespace_name": "fintech-workload" }},
          { "match": { "kubernetes.container_name": "front" }}
        ]
      }
    }
  }'
```

---

## Correct Query Fields

During this phase, the valid fields are:

```text
timestamp
message
severity
kubernetes.namespace_name
kubernetes.pod_name
kubernetes.container_name
```

Use this to query INFO logs:

```bash
curl "http://localhost:9200/k8s-logs-*/_search?pretty" \
  -H 'Content-Type: application/json' \
  -d '{
    "size": 10,
    "_source": [
      "@timestamp",
      "timestamp",
      "message",
      "severity",
      "kubernetes.namespace_name",
      "kubernetes.pod_name",
      "kubernetes.container_name"
    ],
    "query": {
      "bool": {
        "must": [
          { "match": { "kubernetes.namespace_name": "fintech-workload" }},
          { "match": { "severity": "INFO" }}
        ]
      }
    },
    "sort": [
      { "@timestamp": { "order": "desc" }}
    ]
  }'
```

Do not use this until the structured parsing phase adds the field:

```text
app_severity
```

---

## Design Decision

Kubernetes labels and annotations are intentionally disabled at this phase.

Workload identity is based on:

```text
kubernetes.namespace_name
kubernetes.container_name
kubernetes.pod_name
```

not:

```text
kubernetes.labels.app
```

This is a deliberate stability decision to avoid OpenSearch dynamic mapping conflicts.

---

## Future Improvements

The current design is stable but can be improved later.

Recommended future improvements:

### 1. Controlled label allowlist

Instead of sending all labels, allow only selected safe labels.

Example target fields:

```text
workload.name
app.name
app.component
app.version
```

### 2. Flatten Kubernetes labels

Convert dynamic label maps into safe flattened string keys.

Example:

```text
kubernetes_label_app = frontend
kubernetes_label_app_kubernetes_io_name = frontend
```

### 3. OpenSearch index templates

Create explicit OpenSearch mappings for known fields.

This prevents dynamic mapping surprises.

### 4. OpenTelemetry-compatible log schema

Move toward stable semantic fields such as:

```text
service.name
service.namespace
deployment.environment
log.severity
event.name
event.category
```

### 5. Application logging contract

Define a standard format for application teams:

```json
{
  "timestamp": "2026-06-06T02:35:50Z",
  "severity": "INFO",
  "message": "Deposit submitted successfully.",
  "service_name": "frontend",
  "event_type": "deposit",
  "environment": "prod"
}
```

---

## Lessons Learned

### 1. Cluster-wide logging requires schema discipline

Collecting everything is easy.

Indexing everything safely is harder.

Dynamic Kubernetes metadata can break OpenSearch mappings if it is not controlled.

### 2. Mapping errors are not buffer errors

Increasing buffers does not fix schema conflicts.

The root cause was OpenSearch rejecting documents because field mappings conflicted.

### 3. Fluent Bit backlog can preserve old bad data

After fixing live configuration, old filesystem-buffered chunks may still retry and produce the same error.

Rotating the Fluent Bit storage path and DB can be necessary after schema-affecting fixes.

### 4. Query only fields that exist

Queries for `app_severity` returned zero because the field had not been created yet.

The indexed field at that stage was `severity`.

### 5. Change one layer at a time

The debugging process stabilized after separating:

```text
Fluent Bit live config
Fluent Bit runtime errors
OpenSearch mapping errors
OpenSearch document shape
Query field names
Old buffered chunks
```

---

## Final Outcome

The pipeline was stabilized.

Final status:

```text
Cluster-wide Fluent Bit collection is active
OpenSearch receives fresh workload logs
Kubernetes labels and annotations are disabled
Dynamic metadata mapping conflict is avoided
Old Fluent Bit disk backlog was bypassed with a new storage path and DB
OpenSearch output workers were reduced
Fresh frontend logs are searchable by namespace, pod, container, message, timestamp, and severity
```

This issue became a production-style lesson in Kubernetes logging, schema control, Fluent Bit buffering, and OpenSearch dynamic mapping behavior.

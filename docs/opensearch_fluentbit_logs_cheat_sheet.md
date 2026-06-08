# OpenSearch + Fluent Bit Logs Cheat Sheet

**Scope:** Kubernetes logs shipped by Fluent Bit into OpenSearch using the index pattern `k8s-logs-*`.

This cheat sheet follows the exact cases already used in your lab: frontend availability evidence, severity parsing, label-disabled queries, OpenSearch checks, Fluent Bit configuration, and troubleshooting.

---

## 1. Architecture

```text
Kubernetes pod stdout/stderr
        ↓
/var/log/containers/*.log on each node
        ↓
Fluent Bit DaemonSet
        ↓
Kubernetes metadata filter
        ↓
Lua normalization filter
        ↓
OpenSearch output
        ↓
k8s-logs-YYYY.MM.DD
```

### Main components

| Component | Purpose |
|---|---|
| Fluent Bit DaemonSet | Reads container logs from every node |
| Kubernetes filter | Adds namespace, pod, and container metadata |
| Lua filter | Normalizes `message`, `severity`, and `timestamp` |
| OpenSearch | Stores searchable logs |
| ArgoCD | Applies Fluent Bit/OpenSearch GitOps manifests |

---

## 2. Important OpenSearch Fields

| Field | Meaning | Example |
|---|---|---|
| `@timestamp` | OpenSearch event timestamp | `2026-06-08T12:39:10.072Z` |
| `timestamp` | normalized app/log timestamp | `2026-06-08 12:39:10` |
| `message` | cleaned log message | `payment \| Payment initiated successfully.` |
| `severity` | normalized severity | `INFO`, `WARN`, `ERROR` |
| `kubernetes.namespace_name` | namespace | `fintech-workload` |
| `kubernetes.pod_name` | pod name | `frontend-67dd44c5c9-zsjc9` |
| `kubernetes.container_name` | container name | `front` |

---

## 3. Important Rule in Your Setup

Your Fluent Bit config has labels disabled:

```ini
Labels Off
Annotations Off
```

So this query will return zero even when frontend logs exist:

```json
{ "match": { "kubernetes.labels.app": "frontend" }}
```

Use pod-name matching instead:

```json
{ "wildcard": { "kubernetes.pod_name.keyword": "frontend-*" }}
```

---

## 4. Basic OpenSearch Checks

### Port-forward OpenSearch

```bash
kubectl port-forward -n observability svc/opensearch-cluster-master 9200:9200
```

### Cluster health

```bash
curl -s "http://localhost:9200/_cluster/health?pretty"
```

### List log indices

```bash
curl -s "http://localhost:9200/_cat/indices/k8s-logs-*?v"
```

### Count all logs

```bash
curl -s "http://localhost:9200/k8s-logs-*/_count?pretty"
```

---

## 5. Read Latest Logs

```bash
curl -s "http://localhost:9200/k8s-logs-*/_search?pretty" \
  -H "Content-Type: application/json" \
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
    "sort": [
      { "@timestamp": { "order": "desc" }}
    ]
  }'
```

---

## 6. Read Logs by Namespace

```bash
curl -s "http://localhost:9200/k8s-logs-*/_search?pretty" \
  -H "Content-Type: application/json" \
  -d '{
    "size": 10,
    "_source": [
      "@timestamp",
      "message",
      "severity",
      "kubernetes.namespace_name",
      "kubernetes.pod_name",
      "kubernetes.container_name"
    ],
    "query": {
      "match": {
        "kubernetes.namespace_name": "fintech-workload"
      }
    },
    "sort": [
      { "@timestamp": { "order": "desc" }}
    ]
  }'
```

---

## 7. Read Frontend Logs Correctly

Because labels are disabled, use the frontend pod prefix.

```bash
curl -s "http://localhost:9200/k8s-logs-*/_search?pretty" \
  -H "Content-Type: application/json" \
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
          { "wildcard": { "kubernetes.pod_name.keyword": "frontend-*" }}
        ]
      }
    },
    "sort": [
      { "@timestamp": { "order": "desc" }}
    ]
  }'
```

### Interpretation

```text
Frontend logs are present when pod_name matches frontend-*.
Normal frontend activity may include login, signup, deposit, payment, and logout messages.
```

---

## 8. Read Frontend ERROR Logs

```bash
curl -s "http://localhost:9200/k8s-logs-*/_search?pretty" \
  -H "Content-Type: application/json" \
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
          { "wildcard": { "kubernetes.pod_name.keyword": "frontend-*" }},
          { "match": { "severity": "ERROR" }}
        ]
      }
    },
    "sort": [
      { "@timestamp": { "order": "desc" }}
    ]
  }'
```

### Interpretation

```text
If frontend ERROR logs are low while the availability probe fails,
the issue may be routing, Service, Ingress, DNS, readiness, endpoints, or network policy.

If frontend ERROR logs spike during the incident window,
investigate the frontend app and downstream dependencies.
```

---

## 9. Namespace Severity Distribution

```bash
curl -s "http://localhost:9200/k8s-logs-*/_search?pretty" \
  -H "Content-Type: application/json" \
  -d '{
    "size": 0,
    "query": {
      "match": {
        "kubernetes.namespace_name": "fintech-workload"
      }
    },
    "aggs": {
      "severity_counts": {
        "terms": {
          "field": "severity.keyword",
          "size": 10
        }
      }
    }
  }'
```

### Interpretation

```text
INFO-dominant logs with a small ERROR count suggest no broad application exception storm.
```

---

## 10. Search by Message Text

### Payment logs

```bash
curl -s "http://localhost:9200/k8s-logs-*/_search?pretty" \
  -H "Content-Type: application/json" \
  -d '{
    "size": 20,
    "_source": ["@timestamp", "message", "severity", "kubernetes.namespace_name", "kubernetes.pod_name"],
    "query": {
      "match_phrase": {
        "message": "payment"
      }
    },
    "sort": [
      { "@timestamp": { "order": "desc" }}
    ]
  }'
```

### Timeout logs

```bash
curl -s "http://localhost:9200/k8s-logs-*/_search?pretty" \
  -H "Content-Type: application/json" \
  -d '{
    "size": 20,
    "_source": ["@timestamp", "message", "severity", "kubernetes.namespace_name", "kubernetes.pod_name"],
    "query": {
      "match_phrase": {
        "message": "timed out"
      }
    },
    "sort": [
      { "@timestamp": { "order": "desc" }}
    ]
  }'
```

### Insufficient balance logs

```bash
curl -s "http://localhost:9200/k8s-logs-*/_search?pretty" \
  -H "Content-Type: application/json" \
  -d '{
    "size": 20,
    "_source": ["@timestamp", "message", "severity", "kubernetes.namespace_name", "kubernetes.pod_name"],
    "query": {
      "match_phrase": {
        "message": "insufficient balance"
      }
    },
    "sort": [
      { "@timestamp": { "order": "desc" }}
    ]
  }'
```

---

## 11. Time-Window Queries

### Last 15 minutes

```bash
curl -s "http://localhost:9200/k8s-logs-*/_search?pretty" \
  -H "Content-Type: application/json" \
  -d '{
    "size": 20,
    "_source": ["@timestamp", "message", "severity", "kubernetes.namespace_name", "kubernetes.pod_name"],
    "query": {
      "range": {
        "@timestamp": {
          "gte": "now-15m",
          "lte": "now"
        }
      }
    },
    "sort": [
      { "@timestamp": { "order": "desc" }}
    ]
  }'
```

### Frontend logs in a specific incident window

```bash
curl -s "http://localhost:9200/k8s-logs-*/_search?pretty" \
  -H "Content-Type: application/json" \
  -d '{
    "size": 50,
    "_source": ["@timestamp", "message", "severity", "kubernetes.namespace_name", "kubernetes.pod_name", "kubernetes.container_name"],
    "query": {
      "bool": {
        "must": [
          { "match": { "kubernetes.namespace_name": "fintech-workload" }},
          { "wildcard": { "kubernetes.pod_name.keyword": "frontend-*" }},
          {
            "range": {
              "@timestamp": {
                "gte": "2026-06-08T08:20:00Z",
                "lte": "2026-06-08T08:35:00Z"
              }
            }
          }
        ]
      }
    },
    "sort": [
      { "@timestamp": { "order": "asc" }}
    ]
  }'
```

---

## 12. Useful Aggregations

### Logs by namespace

```bash
curl -s "http://localhost:9200/k8s-logs-*/_search?pretty" \
  -H "Content-Type: application/json" \
  -d '{
    "size": 0,
    "aggs": {
      "namespaces": {
        "terms": {
          "field": "kubernetes.namespace_name.keyword",
          "size": 30
        }
      }
    }
  }'
```

### Top noisy pods

```bash
curl -s "http://localhost:9200/k8s-logs-*/_search?pretty" \
  -H "Content-Type: application/json" \
  -d '{
    "size": 0,
    "aggs": {
      "top_pods": {
        "terms": {
          "field": "kubernetes.pod_name.keyword",
          "size": 20
        }
      }
    }
  }'
```

### ERROR count by pod

```bash
curl -s "http://localhost:9200/k8s-logs-*/_search?pretty" \
  -H "Content-Type: application/json" \
  -d '{
    "size": 0,
    "query": {
      "match": {
        "severity": "ERROR"
      }
    },
    "aggs": {
      "error_by_pod": {
        "terms": {
          "field": "kubernetes.pod_name.keyword",
          "size": 20
        }
      }
    }
  }'
```

---

## 13. Severity Normalization

### Expected behavior

| Log format | Expected severity |
|---|---|
| `INFO message` | `INFO` |
| `WARN message` | `WARN` |
| `ERROR message` | `ERROR` |
| `level=info msg="..."` | `INFO` |
| `level=warn msg="..."` | `WARN` |
| `level=error msg="..."` | `ERROR` |
| JSON `"severity":"ERROR"` | `ERROR` |

### Good Lua block

```lua
if not json_msg then
    local lower_raw = string.lower(raw)

    if lower_raw:find("level=fatal") or lower_raw:find("level=critical") or lower_raw:find("critical") then
        severity = "CRITICAL"
    elseif lower_raw:find("level=error") or lower_raw:find(" error ") then
        severity = "ERROR"
    elseif lower_raw:find("level=warn") or lower_raw:find("warning") or lower_raw:find(" warn ") then
        severity = "WARN"
    elseif lower_raw:find("level=debug") or lower_raw:find(" debug ") then
        severity = "DEBUG"
    elseif lower_raw:find("level=info") or lower_raw:find(" info ") then
        severity = "INFO"
    end
end
```

### Test severity parser

```bash
kubectl delete pod severity-test-2 -n observability --ignore-not-found

kubectl run severity-test-2 -n observability \
  --image=busybox:1.36 \
  --restart=Never -- \
  sh -c '
    echo "SEVERITY_TEST_V2 uppercase INFO message";
    echo "SEVERITY_TEST_V2 uppercase WARN message";
    echo "SEVERITY_TEST_V2 uppercase ERROR message";
    echo "level=info msg=\"SEVERITY_TEST_V2 lowercase info message\"";
    echo "level=warn msg=\"SEVERITY_TEST_V2 lowercase warn message\"";
    echo "level=error msg=\"SEVERITY_TEST_V2 lowercase error message\"";
    sleep 30
  '
```

Query:

```bash
curl -s "http://localhost:9200/k8s-logs-*/_search?pretty" \
  -H "Content-Type: application/json" \
  -d '{
    "size": 20,
    "_source": ["@timestamp", "message", "severity", "kubernetes.namespace_name", "kubernetes.pod_name"],
    "query": {
      "match_phrase": {
        "message": "SEVERITY_TEST_V2"
      }
    },
    "sort": [
      { "@timestamp": { "order": "desc" }}
    ]
  }'
```

Expected:

```text
level=info  -> INFO
level=warn  -> WARN
level=error -> ERROR
```

---

## 14. Fluent Bit Configuration Checks

### List Fluent Bit resources

```bash
kubectl get pods -n observability -l app.kubernetes.io/name=fluent-bit
kubectl get configmap -n observability | grep fluent
```

### Check Lua ConfigMap

```bash
kubectl get configmap -n observability fluent-bit-luascripts -o yaml
```

### Check active Lua severity logic

```bash
kubectl get configmap -n observability fluent-bit-luascripts -o yaml | grep -A25 "if not json_msg"
```

Expected line:

```lua
local lower_raw = string.lower(raw)
```

If this line is missing, the new parser is not active in the cluster.

---

## 15. Apply Fluent Bit Changes with ArgoCD

```bash
git status
git add platform/fluent-bit/values.yaml
git commit -m "Improve Fluent Bit log normalization"
git push

argocd app sync fluent-bit
kubectl rollout status ds/fluent-bit -n observability
argocd app get fluent-bit
```

Important:

```text
Old OpenSearch documents are not rewritten.
Parser fixes only affect new logs after Fluent Bit restarts.
```

---

## 16. Important Fluent Bit Config Snippets

### Tail input

```ini
[INPUT]
    Name                tail
    Path                /var/log/containers/*.log
    Exclude_Path        /var/log/containers/*_observability_fluent-bit-*.log
    Parser              cri
    Tag                 kube.*
    Refresh_Interval    10
    Mem_Buf_Limit       64MB
    Skip_Long_Lines     On
    Read_from_Head      Off
    DB                  /var/log/flb_kube_cluster_v2.db
    DB.Sync             Normal
    Rotate_Wait         60
    storage.type        filesystem
```

### Kubernetes filter

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

### Lua filter

```ini
[FILTER]
    Name                lua
    Match               kube.*
    script              /fluent-bit/scripts/normalize.lua
    call                normalize
```

### OpenSearch output

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
    Buffer_Size         2M
    Workers             1
    Retry_Limit         False
    Trace_Error         Off
```

---

## 17. Troubleshooting: Query Returns 0

### Cause 1: You queried labels while labels are disabled

Bad:

```json
{ "match": { "kubernetes.labels.app": "frontend" }}
```

Good:

```json
{ "wildcard": { "kubernetes.pod_name.keyword": "frontend-*" }}
```

### Cause 2: Wrong namespace

List namespaces in logs:

```bash
curl -s "http://localhost:9200/k8s-logs-*/_search?pretty" \
  -H "Content-Type: application/json" \
  -d '{
    "size": 0,
    "aggs": {
      "namespaces": {
        "terms": {
          "field": "kubernetes.namespace_name.keyword",
          "size": 50
        }
      }
    }
  }'
```

### Cause 3: Wrong field type

For exact/wildcard queries, prefer `.keyword` fields:

```text
kubernetes.pod_name.keyword
kubernetes.namespace_name.keyword
severity.keyword
```

---

## 18. Troubleshooting: Severity Still Wrong

Symptom:

```text
level=warn  -> INFO
level=error -> INFO
```

Check live ConfigMap:

```bash
kubectl get configmap -n observability fluent-bit-luascripts -o yaml | grep -A25 "if not json_msg"
```

You must see:

```lua
local lower_raw = string.lower(raw)
```

If not:

```bash
argocd app sync fluent-bit
kubectl rollout status ds/fluent-bit -n observability
```

Then create new test logs. Do not judge with old OpenSearch documents.

---

## 19. Troubleshooting: ArgoCD YAML Error

Error example:

```text
yaml: line 59: did not find expected key
```

Check the broken area:

```bash
nl -ba platform/fluent-bit/values.yaml | sed -n '45,80p'
```

Common cause:

```text
Lua code is not indented under normalize.lua: |
```

Correct pattern:

```yaml
luaScripts:
  normalize.lua: |
    function normalize(tag, timestamp, record)
        local raw = record["message"] or record["log"] or ""
```

---

## 20. Troubleshooting: Shards Show 2 Successful out of 3

Example:

```json
"_shards": {
  "total": 3,
  "successful": 2,
  "failed": 0
}
```

If `failed` is `0`, the query did not fail. Still, check cluster/index state:

```bash
curl -s "http://localhost:9200/_cat/indices/k8s-logs-*?v"
curl -s "http://localhost:9200/_cat/shards/k8s-logs-*?v"
curl -s "http://localhost:9200/_cluster/health?pretty"
```

---

## 21. Incident Investigation Flow

### Step 1: Is the app logging?

```bash
# frontend-* logs in fintech-workload
# If this returns logs, ingestion and pod matching work.
```

Use the query from section 7.

### Step 2: Are there app errors?

Use the query from section 8.

### Step 3: Is there a namespace-wide error spike?

Use the severity distribution query from section 9.

### Step 4: Compare logs with Kubernetes state

```bash
kubectl get pods -n fintech-workload -o wide
kubectl get svc,endpoints -n fintech-workload
kubectl get ingress -A
kubectl describe pod -n fintech-workload <pod-name>
```

---

## 22. Evidence Writing Templates

### Logs present, mostly INFO

```text
Frontend logs are present and show normal user-flow activity.
The logs are INFO-dominant, with no evidence of a broad frontend exception storm.
This suggests the availability issue may require checking service routing,
endpoints, ingress, DNS, readiness, or network policy state.
```

### Low number of ERROR logs

```text
Frontend ERROR logs exist, but the volume is low compared with total workload logs.
The observed ERROR messages appear to be business/application-level failures
or downstream timeouts, rather than a continuous crash loop.
```

### No frontend logs

```text
No frontend logs were found for the selected time window.
Possible causes include wrong pod selector, wrong namespace, no traffic,
log ingestion delay, pod not running, or a time-window mismatch.
```

---

## 23. GitOps Files to Document

```text
argocd/applications/fluent-bit-app.yaml
platform/fluent-bit/values.yaml
platform/opensearch/values.yaml
platform/opensearch/bootstrap/opensearch-bootstrap-job.yaml
```

Recommended README sections:

```text
1. Architecture
2. OpenSearch deployment
3. Fluent Bit deployment
4. Log normalization
5. Query examples
6. Incident evidence examples
7. Troubleshooting
8. Known limitations
```

---

## 24. Known Limitations

### Labels are disabled

Use:

```text
kubernetes.pod_name.keyword
kubernetes.namespace_name.keyword
kubernetes.container_name.keyword
```

instead of:

```text
kubernetes.labels.app
```

### Old logs are not rewritten

After Lua parser changes, old OpenSearch documents keep their old severity.
Only new logs get the new parser output.

### `hits.total.value: 10000` with `relation: gte`

This means OpenSearch found at least 10,000 matches, not exactly 10,000.
Use time windows or aggregations for more precise evidence.

---

## 25. Best Practices

- Use `_source` filtering to keep outputs readable.
- Use `.keyword` for exact or wildcard matching.
- Use time windows during incident analysis.
- Keep labels off unless label-based search is truly required.
- Use pod-name prefixes for deployment-style workloads.
- Validate parser changes with a test pod.
- Compare OpenSearch logs with Kubernetes state.
- Keep evidence queries in Git so they are repeatable.

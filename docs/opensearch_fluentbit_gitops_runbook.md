# OpenSearch + Fluent Bit GitOps Logging Runbook

**Environment:** Talos Kubernetes homelab on Proxmox  
**Namespace:** `observability`  
**Main components:** OpenSearch, Fluent Bit, ArgoCD, Longhorn, Kubernetes Jobs, DaemonSets, StatefulSets, Services  
**Purpose:** Central Kubernetes log collection from node container logs into OpenSearch indices named `k8s-logs-*`.

---

## 1. High-level architecture

```text
Kubernetes node log files
/var/log/containers/*.log
        |
        v
Fluent Bit DaemonSet
- Tail input reads container logs
- Kubernetes filter enriches records with pod/namespace/container metadata
- Lua filter normalizes message, severity, timestamp
- OpenSearch output sends records to OpenSearch
        |
        v
OpenSearch Service
opensearch-cluster-master.observability.svc.cluster.local:9200
        |
        v
OpenSearch StatefulSet Pod
opensearch-master-0
        |
        v
Longhorn PVC
opensearch-master-opensearch-master-0
        |
        v
Indices
k8s-logs-YYYY.MM.DD
        |
        v
ISM retention policy
Delete after 3 days
```

This setup follows a common Kubernetes logging pattern: a per-node log agent runs as a `DaemonSet`, reads container log files from each node, enriches the log records with Kubernetes metadata, and forwards them to a central search/storage backend.

---

## 2. Components and responsibilities

| Component | Kubernetes kind | Namespace | Role |
|---|---:|---|---|
| OpenSearch | `StatefulSet` | `observability` | Stores searchable log documents. |
| OpenSearch Service | `Service` | `observability` | Stable DNS endpoint for Fluent Bit and bootstrap Job. |
| OpenSearch Headless Service | `Service` | `observability` | Stable pod discovery for StatefulSet. |
| OpenSearch PVC | `PersistentVolumeClaim` | `observability` | Persistent storage for OpenSearch data. |
| Fluent Bit | `DaemonSet` | `observability` | Runs one log collector per worker node. |
| OpenSearch bootstrap | `Job` | `observability` | Creates/validates ISM policy, index template, and replica settings. |
| ArgoCD Application | `Application` | `argocd` | GitOps reconciliation for OpenSearch, Fluent Bit, bootstrap Job. |
| Longhorn | Storage backend | `longhorn-system` | Provides persistent volumes. |

---

## 3. OpenSearch architecture in this lab

### 3.1 OpenSearch deployment model

OpenSearch is deployed by Helm through ArgoCD using the official OpenSearch Helm chart.

Important GitOps application file:

```text
argocd/applications/opensearch-app.yaml
```

Important values file:

```text
platform/opensearch/values.yaml
```

The app uses ArgoCD multi-source configuration:

```yaml
sources:
  - repoURL: https://opensearch-project.github.io/helm-charts/
    chart: opensearch
    targetRevision: 2.27.0
    helm:
      valueFiles:
        - $values/platform/opensearch/values.yaml
  - repoURL: https://github.com/anselem-okeke/sre-decision-intelligence-gitops.git
    targetRevision: main
    ref: values
```

### 3.2 OpenSearch values used

```yaml
clusterName: opensearch
nodeGroup: master

singleNode: true
replicas: 1

persistence:
  enabled: true
  size: 20Gi
  storageClass: longhorn-retain

resources:
  requests:
    cpu: "100m"
    memory: "1Gi"
  limits:
    cpu: "1"
    memory: "2Gi"

securityConfig:
  enabled: false

extraEnvs:
  - name: DISABLE_SECURITY_PLUGIN
    value: "true"

opensearchConfig:
  opensearch.yml: |
    cluster.name: opensearch
    network.host: 0.0.0.0
    plugins.security.disabled: true
```

### 3.3 Why OpenSearch uses a StatefulSet

OpenSearch is stateful. Its data must survive pod restarts. A Kubernetes `StatefulSet` gives stable pod identity and stable persistent storage attachment. In this lab, the pod is:

```text
opensearch-master-0
```

and the PVC is:

```text
opensearch-master-opensearch-master-0
```

The service endpoint used by Fluent Bit is:

```text
opensearch-cluster-master.observability.svc.cluster.local:9200
```

### 3.4 Why OpenSearch needs enough memory

OpenSearch is Java-based. Even if JVM heap is configured around `512Mi`, the container needs extra memory for:

- JVM heap
- off-heap memory
- native memory
- Lucene segments
- plugins
- network buffers
- container overhead

That is why the OpenSearch container limit was kept at `2Gi`, not `512Mi`.

A previous mistake placed Fluent Bit-sized resources into OpenSearch values:

```yaml
requests:
  cpu: 10m
  memory: 64Mi
limits:
  cpu: 300m
  memory: 512Mi
```

That caused OpenSearch instability. The correct OpenSearch resource profile is heavier:

```yaml
requests:
  cpu: "100m"
  memory: "1Gi"
limits:
  cpu: "1"
  memory: "2Gi"
```

---

## 4. Fluent Bit architecture in this lab

### 4.1 Fluent Bit deployment model

Fluent Bit is deployed as a Kubernetes `DaemonSet`, so one Fluent Bit pod runs on each eligible node.

Important GitOps application file:

```text
argocd/applications/fluent-bit-app.yaml
```

Important values file:

```text
platform/fluent-bit/values.yaml
```

The chart source is:

```yaml
repoURL: https://fluent.github.io/helm-charts
chart: fluent-bit
targetRevision: 0.48.5
```

### 4.2 Why Fluent Bit uses a DaemonSet

Kubernetes container logs are stored on each node. A central Deployment would only see logs on the node where it runs. A `DaemonSet` places a Fluent Bit collector on each node, allowing it to read local files such as:

```text
/var/log/containers/*.log
```

### 4.3 Fluent Bit resource settings

Final resource profile:

```yaml
resources:
  requests:
    cpu: 10m
    memory: 64Mi
  limits:
    cpu: 300m
    memory: 512Mi
```

The CPU request was intentionally lowered to `10m` because the cluster had scheduling pressure. The CPU limit remains `300m` so Fluent Bit can burst when necessary.

### 4.4 Fluent Bit pipeline

Fluent Bit is configured as a pipeline:

```text
Input -> Filter -> Filter -> Output
```

In this lab:

```text
Tail input
  -> Kubernetes metadata filter
  -> Lua normalization filter
  -> OpenSearch output
```

---

## 5. Fluent Bit configuration explained

### 5.1 Service section

```ini
[SERVICE]
    Flush                         10
    Log_Level                     info
    Daemon                        off
    Parsers_File                  /fluent-bit/etc/parsers.conf
    HTTP_Server                   On
    HTTP_Listen                   0.0.0.0
    HTTP_Port                     2020
    storage.path                  /var/log/flb-storage-v2
    storage.sync                  normal
    storage.checksum              off
    storage.backlog.mem_limit     25M
    storage.max_chunks_up         64
    storage.pause_on_chunks_overlimit On
```

Important meanings:

| Setting | Meaning |
|---|---|
| `Flush 10` | Flush logs every 10 seconds. |
| `HTTP_Server On` | Enables Fluent Bit internal metrics/API on port `2020`. |
| `storage.path` | Enables filesystem buffering. |
| `storage.backlog.mem_limit 25M` | Limits memory used by backlog replay. |
| `storage.pause_on_chunks_overlimit On` | Pauses ingestion if chunks exceed configured safety limit. |

### 5.2 Tail input

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

Important meanings:

| Setting | Meaning |
|---|---|
| `Path /var/log/containers/*.log` | Reads Kubernetes container logs. |
| `Exclude_Path` | Prevents Fluent Bit from ingesting its own logs and creating a feedback loop. |
| `Parser cri` | Parses CRI-format Kubernetes logs. |
| `Tag kube.*` | Tags records for routing to filters/outputs. |
| `Mem_Buf_Limit 64MB` | Prevents unlimited memory growth from the Tail input. |
| `Read_from_Head Off` | Starts from the end for new files; avoids replaying too much historical log volume. |
| `DB` | Stores file offsets so Fluent Bit can resume after restarts. |
| `storage.type filesystem` | Uses disk buffering when output is slow/down. |

### 5.3 Kubernetes filter

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

Important meanings:

| Setting | Meaning |
|---|---|
| `Match kube.*` | Applies only to records from the Tail input. |
| `Kube_URL` | Talks to Kubernetes API. |
| `Merge_Log On` | Merges structured log content into the record when possible. |
| `Labels Off` | Reduces record size/cardinality. |
| `Annotations Off` | Reduces record size/cardinality. |

### 5.4 Lua normalization filter

The Lua script normalizes:

- `message`
- `severity`
- `timestamp`

Purpose:

```text
Raw app logs -> consistent fields for search/dashboarding
```

Example logic:

```lua
if raw:find("ERROR") then
    severity = "ERROR"
elseif raw:find("WARN") or raw:find("WARNING") then
    severity = "WARN"
elseif raw:find("DEBUG") then
    severity = "DEBUG"
elseif raw:find("INFO") then
    severity = "INFO"
end
```

### 5.5 OpenSearch output

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

Important meanings:

| Setting | Meaning |
|---|---|
| `Name opensearch` | Uses the OpenSearch output plugin. |
| `Host` | Sends logs to the OpenSearch Kubernetes service. |
| `Port 9200` | OpenSearch REST API port. |
| `Logstash_Format On` | Creates date-based indices. |
| `Logstash_Prefix k8s-logs` | Index prefix becomes `k8s-logs-YYYY.MM.DD`. |
| `Retry_Limit False` | Retries indefinitely; useful for temporary OpenSearch outage but requires buffering control. |
| `Workers 1` | Conservative output parallelism for homelab stability. |

---

## 6. OpenSearch bootstrap Job

### 6.1 Why a bootstrap Job exists

Some OpenSearch configuration should not be placed in OpenSearch lifecycle hooks.

We tried using a `postStart` lifecycle hook to configure retention and templates, but this caused pod startup instability:

```text
FailedPostStartHook
CrashLoopBackOff
exit code 137
```

The better GitOps design is:

```text
OpenSearch Helm values -> runtime config only
Bootstrap Job          -> API-level OpenSearch setup
```

### 6.2 Bootstrap Job responsibilities

The Job does three things:

1. Waits until OpenSearch is reachable.
2. Ensures the ISM retention policy exists.
3. Creates/updates the `k8s-logs-*` index template.
4. Sets existing `k8s-logs-*` indices to zero replicas.

### 6.3 Final bootstrap Job behavior

The Job is idempotent:

```sh
echo "Checking 3-day ISM retention policy..."
if curl -fsS "${OPENSEARCH_URL}/_plugins/_ism/policies/delete-after-3-days" >/dev/null; then
  echo "ISM policy delete-after-3-days already exists. Skipping creation."
else
  echo "Creating 3-day ISM retention policy..."
  ...
fi
```

This fixed the `409 Conflict` issue that occurred when trying to recreate an existing ISM policy.

### 6.4 Why ArgoCD Force/Replace was needed

Kubernetes Jobs have immutable pod templates. When the Job command/script changes, Kubernetes cannot patch the existing Job template. ArgoCD showed:

```text
Job.batch "opensearch-bootstrap" is invalid: spec.template: field is immutable
```

The fix was to use:

```yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-options: Force=true,Replace=true
```

That allows ArgoCD to replace the Job instead of trying to patch the immutable template.

### 6.5 Final expected Job log

```text
Waiting for OpenSearch...
OpenSearch is reachable.
Checking 3-day ISM retention policy...
ISM policy delete-after-3-days already exists. Skipping creation.
Creating or updating k8s-logs index template...
{"acknowledged":true}
Ensuring existing k8s-logs indices use zero replicas...
{"acknowledged":true}
OpenSearch bootstrap completed successfully.
```

---

## 7. ISM retention policy

### 7.1 Purpose

Without retention, log indices grow forever and eventually fill the OpenSearch PVC. That happened in this lab: OpenSearch storage filled, then the pod became unstable.

The ISM policy deletes matching indices after 3 days.

### 7.2 Policy name

```text
delete-after-3-days
```

### 7.3 Matching index patterns

```json
[
  "logs-*",
  "otel-*",
  "sre-*",
  "app-*",
  "audit-*",
  "k8s-logs-*"
]
```

### 7.4 Lifecycle

```text
hot state -> wait until index age is at least 3 days -> delete state -> delete index
```

### 7.5 Why `k8s-logs-*` was added

Fluent Bit uses:

```ini
Logstash_Format On
Logstash_Prefix k8s-logs
```

So OpenSearch receives indices named:

```text
k8s-logs-2026.06.06
k8s-logs-2026.06.07
k8s-logs-2026.06.08
```

If `k8s-logs-*` is not in the ISM pattern, those indices will not be covered by retention.

---

## 8. Index template

### 8.1 Purpose

An index template applies default settings to new matching indices.

Template name:

```text
k8s-logs-template
```

Pattern:

```text
k8s-logs-*
```

Settings:

```json
{
  "number_of_shards": 1,
  "number_of_replicas": 0,
  "plugins.index_state_management.policy_id": "delete-after-3-days"
}
```

### 8.2 Why replicas are set to zero

This lab runs single-node OpenSearch:

```yaml
singleNode: true
replicas: 1
```

If index replicas are `1`, OpenSearch waits for a second data copy that cannot exist. The index becomes yellow. For a single-node lab, use:

```json
"number_of_replicas": 0
```

---

## 9. Services and endpoints

### 9.1 OpenSearch services

```text
service/opensearch-cluster-master
service/opensearch-cluster-master-headless
```

The normal ClusterIP service is used by Fluent Bit and the bootstrap Job:

```text
opensearch-cluster-master.observability.svc.cluster.local:9200
```

### 9.2 How to test service reachability from inside the cluster

```bash
kubectl run curl-test -n observability --rm -it \
  --image=curlimages/curl:8.10.1 \
  --restart=Never -- \
  curl -v http://opensearch-cluster-master.observability.svc.cluster.local:9200
```

Expected response:

```json
{
  "name" : "opensearch-master-0",
  "cluster_name" : "opensearch",
  "tagline" : "The OpenSearch Project: https://opensearch.org/"
}
```

---

## 10. Troubleshooting history and lessons learned

### 10.1 Problem: OpenSearch PVC full

Symptom:

```text
OpenSearch pod crashlooping
OpenSearch disk full
PVC expansion blocked by Longhorn available space calculation
```

Attempted PVC expansion failed because Longhorn could not schedule the additional requested space.

Lesson:

- Retention must exist before log volume grows.
- Single-node homelab storage can fill quickly.
- ISM policy is not optional for log platforms.

### 10.2 Problem: lifecycle postStart hook broke OpenSearch

Symptom:

```text
FailedPostStartHook
CrashLoopBackOff
exit code 137
```

Lesson:

Do not put OpenSearch API bootstrap logic inside the OpenSearch container lifecycle. If the API is not ready or the hook takes too long, the main workload can fail startup.

Better design:

```text
Separate Kubernetes Job
```

### 10.3 Problem: bootstrap Job waited forever when OpenSearch was down

Symptom:

```text
Waiting for OpenSearch...
curl: (7) Failed to connect
OpenSearch not ready yet. Retrying...
```

Lesson:

Bootstrap Jobs should have:

```yaml
activeDeadlineSeconds: 180
backoffLimit: 1
```

This avoids infinite waiting.

### 10.4 Problem: ArgoCD hook finalizer stuck

The earlier Job used ArgoCD hook annotations:

```yaml
argocd.argoproj.io/hook: Sync
argocd.argoproj.io/hook-delete-policy: BeforeHookCreation,HookSucceeded
```

When the Job was stuck, the hook finalizer made cleanup harder.

Fix used:

```bash
kubectl patch job opensearch-bootstrap -n observability --type=merge -p '{"metadata":{"finalizers":[]}}'
```

Lesson:

For this lab, a normal Kubernetes Job with `ttlSecondsAfterFinished` and ArgoCD `Force=true,Replace=true` is simpler and more debuggable than an ArgoCD hook Job.

### 10.5 Problem: `409 Conflict` creating ISM policy

Symptom:

```text
curl: (22) The requested URL returned error: 409
```

Cause:

The ISM policy already existed. OpenSearch refused a plain overwrite.

Fix:

Check first, create only if missing:

```sh
if curl -fsS "${OPENSEARCH_URL}/_plugins/_ism/policies/delete-after-3-days" >/dev/null; then
  echo "exists, skipping"
else
  curl -X PUT ...
fi
```

### 10.6 Problem: Kubernetes Job immutable template

Symptom:

```text
spec.template: field is immutable
```

Cause:

Kubernetes does not allow changing a Job pod template after creation.

Fix:

```yaml
argocd.argoproj.io/sync-options: Force=true,Replace=true
```

### 10.7 Problem: OpenSearch scheduled to unstable `talos-w2`

Symptoms:

```text
Init:CreateContainerError
FailedCreatePodSandBox
context deadline exceeded
runc create failed
procReady not received
```

Action:

```bash
kubectl cordon talos-w2
kubectl delete pod opensearch-master-0 -n observability
```

Result:

OpenSearch rescheduled successfully and became healthy again.

### 10.8 Problem: Falco OOMKilled on `talos-w2`

Symptoms:

```text
Reason: OOMKilled
Exit Code: 137
Restart Count: 14
memory limit: 512Mi
```

Fix:

Falco was Helm-managed, so the safer fix was a Helm upgrade:

```bash
helm upgrade falco falcosecurity/falco \
  -n falco \
  --reuse-values \
  --set resources.requests.memory=512Mi \
  --set resources.limits.memory=1Gi
```

Result:

Falco rolled out successfully, and the new `talos-w2` Falco pod had zero restarts.

---

## 11. Validation commands

### 11.1 Check all observability resources

```bash
kubectl get all -n observability
```

Expected:

```text
pod/fluent-bit-*          1/1 Running
pod/opensearch-master-0   1/1 Running
job/opensearch-bootstrap  Complete 1/1
```

### 11.2 Check OpenSearch service from inside Kubernetes

```bash
kubectl run curl-test -n observability --rm -it \
  --image=curlimages/curl:8.10.1 \
  --restart=Never -- \
  curl -v http://opensearch-cluster-master.observability.svc.cluster.local:9200
```

### 11.3 Port-forward OpenSearch

```bash
kubectl port-forward -n observability svc/opensearch-cluster-master 9200:9200
```

### 11.4 List indices

```bash
curl -s http://localhost:9200/_cat/indices/k8s-logs-*?v
```

Expected:

```text
health status index                pri rep docs.count store.size
green  open   k8s-logs-2026.06.08  1   0   ...        ...
```

### 11.5 Check latest documents

```bash
curl -s "http://localhost:9200/k8s-logs-*/_search?pretty" \
  -H "Content-Type: application/json" \
  -d '{
    "size": 5,
    "sort": [
      {
        "@timestamp": {
          "order": "desc"
        }
      }
    ],
    "_source": [
      "@timestamp",
      "kubernetes.namespace_name",
      "kubernetes.pod_name",
      "severity",
      "message"
    ]
  }'
```

### 11.6 Generate test logs

```bash
kubectl run log-test -n observability \
  --image=busybox:1.36 \
  --restart=Never -- \
  sh -c 'for i in $(seq 1 20); do echo "SRE_LOG_TEST $i $(date)"; sleep 1; done'
```

Search for the test logs:

```bash
curl -s "http://localhost:9200/k8s-logs-*/_search?pretty" \
  -H "Content-Type: application/json" \
  -d '{
    "size": 10,
    "query": {
      "match_phrase": {
        "message": "SRE_LOG_TEST"
      }
    },
    "sort": [
      {
        "@timestamp": {
          "order": "desc"
        }
      }
    ]
  }'
```

Clean up:

```bash
kubectl delete pod log-test -n observability --ignore-not-found
```

### 11.7 Check Fluent Bit logs

```bash
kubectl logs -n observability -l app.kubernetes.io/name=fluent-bit --tail=100
```

Bad signs:

```text
connection refused
failed to flush chunk
HTTP status=429
HTTP status=500
no upstream connections
```

### 11.8 Check Fluent Bit resource usage

```bash
kubectl top pod -n observability
```

### 11.9 Check OpenSearch pod events

```bash
kubectl describe pod opensearch-master-0 -n observability
```

Look for:

```text
OOMKilled
FailedMount
Multi-Attach
FailedCreatePodSandBox
Startup probe failed
Readiness probe failed
```

### 11.10 Check PVC

```bash
kubectl get pvc -n observability
kubectl describe pvc opensearch-master-opensearch-master-0 -n observability
```

---

## 12. Operational runbook

### 12.1 If Fluent Bit is running but no logs appear in OpenSearch

Check in this order:

```bash
kubectl get pods -n observability
kubectl logs -n observability -l app.kubernetes.io/name=fluent-bit --tail=100
kubectl run curl-test -n observability --rm -it --image=curlimages/curl:8.10.1 --restart=Never -- curl -v http://opensearch-cluster-master.observability.svc.cluster.local:9200
curl -s http://localhost:9200/_cat/indices/k8s-logs-*?v
```

Likely causes:

| Symptom | Likely cause | Fix |
|---|---|---|
| Fluent Bit connection refused | OpenSearch down or no service endpoint | Check OpenSearch pod and service endpoints. |
| Fluent Bit memory high | Backlog replay or OpenSearch outage | Check buffering, reduce replay, verify OpenSearch is healthy. |
| No `k8s-logs-*` index | Output not configured, Fluent Bit not reading logs, or OpenSearch unreachable | Check output config and Fluent Bit logs. |
| Indices yellow | Replicas > 0 on single-node OpenSearch | Set `number_of_replicas: 0`. |

### 12.2 If OpenSearch is crashlooping

```bash
kubectl get pod opensearch-master-0 -n observability
kubectl describe pod opensearch-master-0 -n observability
kubectl logs opensearch-master-0 -n observability --tail=100
kubectl get pvc -n observability
```

Common causes:

| Cause | Evidence | Fix |
|---|---|---|
| PVC full | Disk watermark / no space errors | Retention, delete old indices, expand PVC if Longhorn allows. |
| Memory too low | OOMKilled / exit 137 | Increase OpenSearch memory limit. |
| Bad lifecycle hook | FailedPostStartHook | Remove hook, use Job. |
| Node runtime issue | FailedCreatePodSandBox | Cordon node, reschedule, reboot node if needed. |

### 12.3 If Job fails with `409`

Cause:

```text
OpenSearch policy already exists
```

Fix:

Use idempotent logic: check policy first, create only if missing.

### 12.4 If ArgoCD says Job template is immutable

Cause:

```text
Kubernetes Job spec.template cannot be patched
```

Fix:

```yaml
argocd.argoproj.io/sync-options: Force=true,Replace=true
```

Then:

```bash
kubectl delete job opensearch-bootstrap -n observability --ignore-not-found
argocd app sync opensearch-bootstrap --force
```

### 12.5 If node has runtime errors

Check conditions:

```bash
kubectl describe node talos-w2 | egrep -A8 "Conditions:|Pressure|Ready|Allocated resources"
```

Check usage:

```bash
kubectl top node talos-w2
kubectl get pods -A -o wide | grep talos-w2
kubectl top pod -A --sort-by=cpu | head -30
kubectl top pod -A --sort-by=memory | head -30
```

If needed:

```bash
kubectl cordon talos-w2
```

After stabilization:

```bash
kubectl uncordon talos-w2
```

---

## 13. Final known-good state

### 13.1 Observability namespace

Expected:

```text
pod/fluent-bit-*          Running
pod/opensearch-master-0   Running
job/opensearch-bootstrap  Complete
```

### 13.2 OpenSearch bootstrap logs

Expected:

```text
OpenSearch is reachable.
ISM policy delete-after-3-days already exists. Skipping creation.
Creating or updating k8s-logs index template...
{"acknowledged":true}
Ensuring existing k8s-logs indices use zero replicas...
{"acknowledged":true}
OpenSearch bootstrap completed successfully.
```

### 13.3 Falco stabilization action

Because Falco was OOMKilled with a `512Mi` memory limit, it was upgraded with:

```bash
helm upgrade falco falcosecurity/falco \
  -n falco \
  --reuse-values \
  --set resources.requests.memory=512Mi \
  --set resources.limits.memory=1Gi
```

Expected:

```text
falco-*   2/2 Running 0 restarts
```

---

## 14. Recommended production improvements

This lab is intentionally simple. For production, improve these areas:

### 14.1 OpenSearch security

Current lab setting:

```yaml
securityConfig:
  enabled: false
DISABLE_SECURITY_PLUGIN: "true"
plugins.security.disabled: true
```

Production should use:

- TLS
- authentication
- authorization
- network policies
- secret management
- restricted service accounts

### 14.2 OpenSearch high availability

Current lab:

```yaml
singleNode: true
replicas: 1
```

Production should use:

- multiple OpenSearch nodes
- separate node roles where appropriate
- replicas > 0
- snapshots
- tested restore process
- capacity planning

### 14.3 Retention strategy

Current lab:

```text
Delete after 3 days
```

Production should define retention by log class:

| Log type | Example retention |
|---|---:|
| Debug/application logs | 3-7 days |
| Platform logs | 14-30 days |
| Security/audit logs | 90+ days, depending on requirements |

### 14.4 Fluent Bit resource tuning

Watch:

```bash
kubectl top pod -n observability
kubectl logs -n observability -l app.kubernetes.io/name=fluent-bit --tail=100
```

Tune:

- `Mem_Buf_Limit`
- filesystem buffering
- `storage.max_chunks_up`
- `Retry_Limit`
- output workers
- exclusion paths
- noisy namespaces

### 14.5 GitOps quality

Recommended structure:

```text
platform/
  opensearch/
    values.yaml
    bootstrap/
      opensearch-bootstrap-job.yaml
  fluent-bit/
    values.yaml
argocd/
  applications/
    opensearch-app.yaml
    fluent-bit-app.yaml
    opensearch-bootstrap-app.yaml
```

Recommended rules:

- Keep runtime configuration in Helm values.
- Keep API bootstrapping in Jobs.
- Avoid lifecycle hooks for complex API configuration.
- Make Jobs idempotent.
- Add timeouts to Jobs.
- Use `Force=true,Replace=true` for replaceable one-shot Jobs.

---

## 15. Mental model: how logs move

When a pod writes a line to stdout/stderr:

```text
Application container stdout/stderr
```

The container runtime writes it to the node:

```text
/var/log/containers/<pod>_<namespace>_<container>-<id>.log
```

Fluent Bit on that node tails the file:

```text
[INPUT] tail
```

It enriches the record with Kubernetes metadata:

```text
[FILTER] kubernetes
```

It normalizes fields:

```text
[FILTER] lua
```

It sends to OpenSearch:

```text
[OUTPUT] opensearch
```

OpenSearch stores the document in a daily index:

```text
k8s-logs-YYYY.MM.DD
```

The ISM policy deletes old indices after 3 days.

---

## 16. Key commands cheat sheet

```bash
# Observability status
kubectl get all -n observability

# Fluent Bit logs
kubectl logs -n observability -l app.kubernetes.io/name=fluent-bit --tail=100

# OpenSearch health from inside cluster
kubectl run curl-test -n observability --rm -it --image=curlimages/curl:8.10.1 --restart=Never -- curl -v http://opensearch-cluster-master.observability.svc.cluster.local:9200

# Port-forward OpenSearch
kubectl port-forward -n observability svc/opensearch-cluster-master 9200:9200

# List log indices
curl -s http://localhost:9200/_cat/indices/k8s-logs-*?v

# Search latest logs
curl -s "http://localhost:9200/k8s-logs-*/_search?pretty" -H "Content-Type: application/json" -d '{"size":5,"sort":[{"@timestamp":{"order":"desc"}}]}'

# Generate test logs
kubectl run log-test -n observability --image=busybox:1.36 --restart=Never -- sh -c 'for i in $(seq 1 20); do echo "SRE_LOG_TEST $i $(date)"; sleep 1; done'

# Search test logs
curl -s "http://localhost:9200/k8s-logs-*/_search?pretty" -H "Content-Type: application/json" -d '{"query":{"match_phrase":{"message":"SRE_LOG_TEST"}},"size":10}'

# Delete test pod
kubectl delete pod log-test -n observability --ignore-not-found

# Bootstrap logs
kubectl logs -n observability job/opensearch-bootstrap

# ArgoCD sync
argocd app sync opensearch
argocd app sync fluent-bit
argocd app sync opensearch-bootstrap --force

# Node troubleshooting
kubectl describe node talos-w2 | egrep -A8 "Conditions:|Pressure|Ready|Allocated resources"
kubectl top node talos-w2
kubectl get pods -A -o wide | grep talos-w2
```

---

## 17. External references

These are useful primary sources for deeper learning:

- Kubernetes StatefulSet documentation: https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/
- Kubernetes DaemonSet documentation: https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/
- Kubernetes Job documentation: https://kubernetes.io/docs/concepts/workloads/controllers/job/
- Kubernetes Service documentation: https://kubernetes.io/docs/concepts/services-networking/service/
- Kubernetes Persistent Volumes documentation: https://kubernetes.io/docs/concepts/storage/persistent-volumes/
- Fluent Bit Tail input documentation: https://docs.fluentbit.io/manual/data-pipeline/inputs/tail
- Fluent Bit Kubernetes filter documentation: https://docs.fluentbit.io/manual/data-pipeline/filters/kubernetes
- Fluent Bit OpenSearch output documentation: https://docs.fluentbit.io/manual/data-pipeline/outputs/opensearch
- OpenSearch Index State Management documentation: https://docs.opensearch.org/latest/im-plugin/ism/index/
- OpenSearch index templates documentation: https://docs.opensearch.org/latest/im-plugin/index-templates/
- OpenSearch Helm chart repository: https://github.com/opensearch-project/helm-charts/tree/main/charts/opensearch
- ArgoCD sync options documentation: https://argo-cd.readthedocs.io/en/stable/user-guide/sync-options/

---

## 18. Learning path

Study the system in this order:

1. Kubernetes logging basics: where container logs live.
2. DaemonSet: why log agents run on every node.
3. Fluent Bit Tail input: how files are read and offsets are stored.
4. Fluent Bit Kubernetes filter: how pod metadata is added.
5. Fluent Bit buffering/backpressure: why memory grew during OpenSearch downtime.
6. OpenSearch basics: indices, documents, shards, replicas.
7. Index templates: how new indices receive default settings.
8. ISM retention: how old indices are deleted automatically.
9. StatefulSet + PVC: why OpenSearch storage follows the pod identity.
10. ArgoCD GitOps: how desired state is applied and why Jobs need special handling.
11. Troubleshooting: read symptoms from pod status, events, logs, PVCs, endpoints, and node conditions.


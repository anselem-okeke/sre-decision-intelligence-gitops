# OpenSearch CrashLoopBackOff Troubleshooting Runbook

**Project:** SRE Decision Intelligence GitOps / Observability Stack  
**Component:** OpenSearch + Fluent Bit + ArgoCD + Longhorn  
**Namespace:** `observability`  
**OpenSearch Helm app:** `opensearch`  
**Bootstrap app:** `opensearch-bootstrap`  
**StorageClass:** `longhorn-retain`  
**Last updated:** 2026-06-13

---

## 1. Purpose

This runbook documents how to troubleshoot and permanently fix OpenSearch instability in a Kubernetes GitOps observability stack.

It covers:

- OpenSearch `CrashLoopBackOff`
- `Exit Code: 137`
- OpenSearch running for a while and then failing
- PVC/full disk issues
- retained Longhorn volumes
- Fluent Bit ingestion pressure
- broken or missing ISM retention
- ArgoCD sync/health issues
- bootstrap Job idempotency
- final validation checks

---

## 2. Final Healthy State

The expected final state is:

```bash
kubectl get pods -n observability
```

Expected:

```text
NAME                         READY   STATUS      RESTARTS        AGE
fluent-bit-xxxxx             1/1     Running     0               ...
fluent-bit-yyyyy             1/1     Running     0               ...
opensearch-bootstrap-xxxxx   0/1     Completed   0               ...
opensearch-master-0          1/1     Running     <old count>     ...
```

A historical restart count is acceptable if the timestamp is old and not increasing:

```text
opensearch-master-0   1/1   Running   24 (9h ago)
```

ArgoCD should show:

```bash
kubectl get applications -n argocd
```

Expected:

```text
opensearch             Synced   Healthy
opensearch-bootstrap   Synced   Healthy
fluent-bit             Synced   Healthy
```

OpenSearch cluster health:

```bash
curl -s http://localhost:9200/_cluster/health?pretty
```

Expected:

```json
{
  "status": "green",
  "number_of_nodes": 1,
  "active_shards_percent_as_number": 100.0
}
```

---

## 3. Known Good OpenSearch Values

File:

```bash
platform/opensearch/values.yaml
```

Stable configuration:

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
    memory: "3Gi"

opensearchJavaOpts: "-Xms512m -Xmx512m"

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

    # Single-node lab setting
    discovery.type: single-node

    # Prevent replica issues in single-node mode
    cluster.routing.allocation.enable: all

    # Disk guardrails: stop before the PVC becomes completely full
    cluster.routing.allocation.disk.threshold_enabled: true
    cluster.routing.allocation.disk.watermark.low: "70%"
    cluster.routing.allocation.disk.watermark.high: "80%"
    cluster.routing.allocation.disk.watermark.flood_stage: "90%"
    cluster.info.update.interval: "1m"

startupProbe:
  tcpSocket:
    port: 9200
  initialDelaySeconds: 60
  periodSeconds: 10
  timeoutSeconds: 5
  failureThreshold: 90

readinessProbe:
  tcpSocket:
    port: 9200
  initialDelaySeconds: 20
  periodSeconds: 10
  timeoutSeconds: 5
  failureThreshold: 12
```

### Why this configuration works

| Setting | Purpose |
|---|---|
| `memory limit: 3Gi` | Gives OpenSearch enough container headroom |
| `opensearchJavaOpts: -Xms512m -Xmx512m` | Controls JVM heap through the chart-supported value |
| `startupProbe` | Prevents Kubernetes from killing OpenSearch too early during startup |
| `readinessProbe` | Marks pod ready only after port `9200` is reachable |
| disk watermarks | Prevent OpenSearch from writing until the PVC is completely full |
| `number_of_replicas: 0` in template | Correct for single-node OpenSearch |

---

## 4. Important Lesson: `opensearchJavaOpts` vs `extraEnvs`

Do **not** set JVM heap using `extraEnvs`:

```yaml
extraEnvs:
  - name: OPENSEARCH_JAVA_OPTS
    value: "-Xms512m -Xmx512m"
```

Use the chart-supported key:

```yaml
opensearchJavaOpts: "-Xms512m -Xmx512m"
```

The OpenSearch Helm chart already generates `OPENSEARCH_JAVA_OPTS`. Using `extraEnvs` may not override the default value cleanly.

Verify:

```bash
kubectl describe pod opensearch-master-0 -n observability | grep -A12 "Environment"
```

Expected:

```text
OPENSEARCH_JAVA_OPTS: -Xms512m -Xmx512m
```

---

## 5. Symptom: OpenSearch CrashLoopBackOff

Check pod:

```bash
kubectl get pods -n observability
```

Example failure:

```text
opensearch-master-0   0/1   CrashLoopBackOff   24
```

Describe pod:

```bash
kubectl describe pod opensearch-master-0 -n observability
```

Important signs:

```text
Last State:     Terminated
Reason:         Error
Exit Code:      137
```

### Meaning of Exit Code 137

`137` usually means the process was killed externally, often due to memory pressure or cgroup/container kill.

Typical loop:

```text
OpenSearch starts
→ loads plugins
→ recovers indices
→ GC/memory pressure increases
→ container exits with 137
→ Kubernetes restarts it
```

---

## 6. Why OpenSearch May Run for a Long Time Before Becoming Stable

OpenSearch may show this pattern for a long time:

```text
Running → Error → CrashLoopBackOff → Running → Error
```

It is not failing instantly. During startup it performs expensive work:

```text
Load JVM
Load plugins
Read cluster metadata
Mount Longhorn volume
Recover old indices
Allocate shards
Move cluster state RED → YELLOW → GREEN
Initialize internal plugin indices
```

Check previous logs:

```bash
kubectl logs -n observability opensearch-master-0 --previous --tail=120
```

Typical evidence:

```text
recovered [6] indices into cluster_state
PluginService:onIndexModule index:[k8s-logs-2026.06.12/...]
PluginService:onIndexModule index:[k8s-logs-2026.06.07/...]
Cluster health status changed from [RED] to [YELLOW]
JvmGcMonitorService [gc] overhead
Exit Code: 137
```

This means OpenSearch was killed during recovery pressure.

---

## 7. Check Node Pressure

Find the node:

```bash
kubectl get pod -n observability opensearch-master-0 -o wide
```

Then check it:

```bash
kubectl describe node <NODE_NAME> | grep -A20 -i "Conditions"
kubectl top node <NODE_NAME>
```

Example:

```text
MemoryPressure False
DiskPressure   False
Ready          True
```

Even if `MemoryPressure` is `False`, memory may still be high:

```text
talos-w1   MEMORY 6011Mi   80%
```

That means kubelet may not declare pressure, but OpenSearch still has limited headroom during startup/recovery.

---

## 8. Check OpenSearch Logs

Current logs:

```bash
kubectl logs -n observability opensearch-master-0 --tail=120
```

Previous crashed container:

```bash
kubectl logs -n observability opensearch-master-0 --previous --tail=120
```

Look for:

```text
No space left on device
Exit Code: 137
gc overhead
Cluster health changed from RED to YELLOW
recovered indices
k8s-logs-* indices
vm.max_map_count too low
```

---

## 9. Check `vm.max_map_count`

OpenSearch may warn:

```text
vm.max_map_count [65530] is too low, increase to at least [262144]
```

This should be fixed at the Talos/node OS level.

OpenSearch expects:

```text
vm.max_map_count >= 262144
```

---

## 10. Check PVC/PV/Longhorn State

OpenSearch PVC:

```bash
opensearch-master-opensearch-master-0
```

Check PVC:

```bash
kubectl get pvc -n observability | grep opensearch
```

Check PV:

```bash
kubectl get pv | grep opensearch
```

Example retained PV issue:

```text
pvc-044308d5...   Released
pvc-8b421415...   Released
pvc-a60220e0...   Bound
```

Because `longhorn-retain` uses reclaim policy `Retain`, deleted PVCs may leave PVs and Longhorn volumes behind.

---

## 11. Emergency: Stop OpenSearch Restart Loop

Because ArgoCD auto-sync may recreate it, disable sync temporarily:

```bash
kubectl patch application opensearch -n argocd \
  --type=merge \
  -p '{"spec":{"syncPolicy":null}}'
```

Scale OpenSearch down:

```bash
kubectl scale statefulset opensearch-master -n observability --replicas=0
```

Wait until the pod disappears:

```bash
kubectl get pods -n observability
```

Re-enable auto-sync later:

```bash
kubectl patch application opensearch -n argocd \
  --type=merge \
  -p '{"spec":{"syncPolicy":{"automated":{"prune":true}}}}'
```

Verify:

```bash
argocd app get opensearch | grep -A5 "Sync Policy"
```

Expected:

```text
Sync Policy: Automated (Prune)
```

---

## 12. Clean PVC Reset Procedure

Use only if OpenSearch keeps crashing and old/bad data is suspected.

### 12.1 Scale down OpenSearch

```bash
kubectl patch application opensearch -n argocd \
  --type=merge \
  -p '{"spec":{"syncPolicy":null}}'

kubectl scale statefulset opensearch-master -n observability --replicas=0
kubectl get pods -n observability
```

### 12.2 Delete PVC

```bash
kubectl delete pvc opensearch-master-opensearch-master-0 -n observability
```

### 12.3 Delete released PVs

Find PVs:

```bash
kubectl get pv | grep opensearch
```

Delete them:

```bash
kubectl delete pv <PV_NAME>
```

If stuck, patch finalizers:

```bash
kubectl patch pv <PV_NAME> \
  -p '{"metadata":{"finalizers":null}}' \
  --type=merge

kubectl delete pv <PV_NAME>
```

### 12.4 Delete retained Longhorn volumes

Check:

```bash
kubectl get volumes.longhorn.io -n longhorn-system | grep -E "opensearch|pvc"
```

Delete matching old OpenSearch volumes:

```bash
kubectl delete volumes.longhorn.io -n longhorn-system <LONGHORN_VOLUME_NAME>
```

### 12.5 Start OpenSearch

```bash
kubectl scale statefulset opensearch-master -n observability --replicas=1
kubectl get pods -n observability -w
```

### 12.6 Re-enable ArgoCD auto-sync

```bash
kubectl patch application opensearch -n argocd \
  --type=merge \
  -p '{"spec":{"syncPolicy":{"automated":{"prune":true}}}}'
```

---

## 13. Check OpenSearch Health

Port-forward:

```bash
kubectl port-forward -n observability pod/opensearch-master-0 9200:9200
```

Check cluster health:

```bash
curl -s http://localhost:9200/_cluster/health?pretty
```

Check indices:

```bash
curl -s http://localhost:9200/_cat/indices?v&s=store.size:desc
```

Check allocation:

```bash
curl -s http://localhost:9200/_cat/allocation?v
```

Healthy example:

```text
shards disk.indices disk.used disk.avail disk.total disk.percent node
7      785.6mb      961.5mb  18.5gb     19.5gb     4            opensearch-master-0
```

---

## 14. Real Permanent Fix: ISM Retention

The permanent issue was that log indices were not managed by ISM.

Bad output:

```bash
curl -s http://localhost:9200/_plugins/_ism/explain/k8s-logs-2026.06.12?pretty
```

Bad result:

```json
{
  "k8s-logs-2026.06.12": {
    "index.plugins.index_state_management.policy_id": null,
    "enabled": null
  },
  "total_managed_indices": 0
}
```

Good result:

```json
{
  "k8s-logs-2026.06.12": {
    "policy_id": "delete-after-3-days",
    "enabled": true
  },
  "total_managed_indices": 1
}
```

---

## 15. Check All `k8s-logs-*` Indices Are Managed

```bash
for index in $(curl -s http://localhost:9200/_cat/indices/k8s-logs-*?h=index); do
  echo "==== $index ===="
  curl -s "http://localhost:9200/_plugins/_ism/explain/${index}?pretty" | grep -E '"policy_id"|"enabled"'
done
```

Expected:

```text
==== k8s-logs-2026.06.12 ====
    "policy_id" : "delete-after-3-days",
    "enabled" : true
```

---

## 16. Delete Old Unneeded Indices

If old logs are not needed:

```bash
curl -X DELETE http://localhost:9200/k8s-logs-2026.06.07
curl -X DELETE http://localhost:9200/k8s-logs-2026.06.08
curl -X DELETE http://localhost:9200/k8s-logs-2026.06.09
curl -X DELETE http://localhost:9200/k8s-logs-2026.06.10
curl -X DELETE http://localhost:9200/k8s-logs-2026.06.11
```

Verify:

```bash
curl -s http://localhost:9200/_cat/indices?v&s=store.size:desc
curl -s http://localhost:9200/_cat/allocation?v
```

---

## 17. Correct GitOps Bootstrap Job

File:

```bash
platform/opensearch/bootstrap/opensearch-bootstrap-job.yaml
```

The bootstrap Job must be idempotent.

Important behaviors:

- Wait for OpenSearch
- Create ISM policy only if missing
- Always update index template
- Always set `number_of_replicas: 0`
- Attach policy to existing `k8s-logs-*` indices
- Complete successfully if policy already exists

### Correct Job

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: opensearch-bootstrap
  namespace: observability
  annotations:
    argocd.argoproj.io/sync-options: Force=true,Replace=true
spec:
  backoffLimit: 1
  activeDeadlineSeconds: 180
  ttlSecondsAfterFinished: 3600

  template:
    metadata:
      labels:
        app.kubernetes.io/name: opensearch-bootstrap
    spec:
      restartPolicy: Never

      containers:
        - name: bootstrap
          image: curlimages/curl:8.10.1
          command:
            - sh
            - -c
            - |
              set -eu

              OPENSEARCH_URL="http://opensearch-cluster-master.observability.svc.cluster.local:9200"

              echo "Waiting for OpenSearch..."
              until curl -fsS "${OPENSEARCH_URL}" >/dev/null; do
                echo "OpenSearch not ready yet. Retrying..."
                sleep 5
              done

              echo "OpenSearch is reachable."

              echo "Checking 3-day ISM retention policy..."
              if curl -fsS "${OPENSEARCH_URL}/_plugins/_ism/policies/delete-after-3-days" >/dev/null; then
                echo "ISM policy delete-after-3-days already exists. Keeping existing policy."
              else
                echo "Creating 3-day ISM retention policy..."
                curl -fsS -X PUT "${OPENSEARCH_URL}/_plugins/_ism/policies/delete-after-3-days" \
                  -H "Content-Type: application/json" \
                  -d '{
                    "policy": {
                      "description": "Delete homelab observability indices after 3 days",
                      "default_state": "hot",
                      "states": [
                        {
                          "name": "hot",
                          "actions": [],
                          "transitions": [
                            {
                              "state_name": "delete",
                              "conditions": {
                                "min_index_age": "3d"
                              }
                            }
                          ]
                        },
                        {
                          "name": "delete",
                          "actions": [
                            {
                              "delete": {}
                            }
                          ],
                          "transitions": []
                        }
                      ],
                      "ism_template": [
                        {
                          "index_patterns": [
                            "logs-*",
                            "otel-*",
                            "sre-*",
                            "app-*",
                            "audit-*",
                            "k8s-logs-*"
                          ],
                          "priority": 100
                        }
                      ]
                    }
                  }'
              fi

              echo "Creating or updating k8s-logs index template..."
              curl -fsS -X PUT "${OPENSEARCH_URL}/_index_template/k8s-logs-template" \
                -H "Content-Type: application/json" \
                -d '{
                  "index_patterns": [
                    "k8s-logs-*"
                  ],
                  "template": {
                    "settings": {
                      "number_of_shards": 1,
                      "number_of_replicas": 0,
                      "plugins.index_state_management.policy_id": "delete-after-3-days"
                    }
                  },
                  "priority": 200
                }'

              echo "Ensuring existing k8s-logs indices use zero replicas..."
              curl -fsS -X PUT "${OPENSEARCH_URL}/k8s-logs-*/_settings" \
                -H "Content-Type: application/json" \
                -d '{
                  "index": {
                    "number_of_replicas": 0
                  }
                }' || true

              echo "Attaching ISM policy to existing k8s-logs indices..."
              for index in $(curl -fsS "${OPENSEARCH_URL}/_cat/indices/k8s-logs-*?h=index" || true); do
                echo "Attaching ISM policy to ${index}"
                curl -fsS -X POST "${OPENSEARCH_URL}/_plugins/_ism/add/${index}" \
                  -H "Content-Type: application/json" \
                  -d '{
                    "policy_id": "delete-after-3-days"
                  }' || true
              done

              echo "OpenSearch bootstrap completed successfully."
```

---

## 18. Why the Bootstrap Job Failed Before

Previous failure:

```text
curl: (22) The requested URL returned error: 409
```

Cause:

```text
The ISM policy already existed.
The job tried to PUT/update it.
OpenSearch returned 409 conflict.
Because the script used set -eu, the job stopped immediately.
```

Result:

```text
The job never reached:
- index template update
- zero replica settings
- ISM attach to existing indices
```

Correct behavior:

```text
If policy exists, skip creation.
Still continue with template update and ISM attachment.
```

---

## 19. Fix ArgoCD Bootstrap Degraded State

If `opensearch-bootstrap` is degraded, check logs:

```bash
kubectl logs -n observability job/opensearch-bootstrap
```

Check app:

```bash
kubectl get applications -n argocd
```

If YAML is invalid:

```text
yaml: found character that cannot start any token
```

Find tabs:

```bash
grep -nP '\t' platform/opensearch/bootstrap/opensearch-bootstrap-job.yaml
```

Replace tabs with spaces:

```bash
sed -i 's/\t/  /g' platform/opensearch/bootstrap/opensearch-bootstrap-job.yaml
```

Validate:

```bash
kubectl apply --dry-run=client -f platform/opensearch/bootstrap/opensearch-bootstrap-job.yaml
```

Commit:

```bash
git add platform/opensearch/bootstrap/opensearch-bootstrap-job.yaml
git commit -m "Fix OpenSearch bootstrap YAML"
git push
```

Sync:

```bash
argocd app get opensearch-bootstrap --hard-refresh
argocd app sync opensearch-bootstrap --force
```

Healthy result:

```text
opensearch-bootstrap   Synced   Healthy
```

---

## 20. Commit and Sync Workflow

For OpenSearch values:

```bash
git add platform/opensearch/values.yaml
git commit -m "Stabilize OpenSearch runtime settings"
git push

argocd app get opensearch --hard-refresh
argocd app sync opensearch
```

For bootstrap job:

```bash
git add platform/opensearch/bootstrap/opensearch-bootstrap-job.yaml
git commit -m "Make OpenSearch bootstrap idempotent"
git push

argocd app get opensearch-bootstrap --hard-refresh
argocd app sync opensearch-bootstrap --force
```

---

## 21. Final Verification Checklist

```bash
kubectl get pods -n observability
kubectl get applications -n argocd
curl -s http://localhost:9200/_cluster/health?pretty
curl -s http://localhost:9200/_cat/indices?v&s=store.size:desc
curl -s http://localhost:9200/_cat/allocation?v
```

Check ISM:

```bash
for index in $(curl -s http://localhost:9200/_cat/indices/k8s-logs-*?h=index); do
  echo "==== $index ===="
  curl -s "http://localhost:9200/_plugins/_ism/explain/${index}?pretty" | grep -E '"policy_id"|"enabled"'
done
```

Expected:

```text
"policy_id" : "delete-after-3-days"
"enabled" : true
```

---

## 22. Recommended Alerts

Add alerts for:

### OpenSearch pod restart increase

Alert when restart count increases.

### OpenSearch PVC usage

Warning:

```text
PVC usage > 70%
```

Critical:

```text
PVC usage > 85%
```

Emergency:

```text
PVC usage > 90%
```

PromQL:

```promql
(
  kubelet_volume_stats_used_bytes{
    namespace="observability",
    persistentvolumeclaim="opensearch-master-opensearch-master-0"
  }
/
  kubelet_volume_stats_capacity_bytes{
    namespace="observability",
    persistentvolumeclaim="opensearch-master-opensearch-master-0"
  }
) * 100
```

### OpenSearch pod not ready

```text
opensearch-master-0 Ready != True
```

### ISM policy missing

Periodically verify:

```bash
curl -s http://localhost:9200/_plugins/_ism/explain/k8s-logs-$(date +%Y.%m.%d)?pretty
```

---

## 23. Root Cause Summary

The issue was a combination of:

```text
1. OpenSearch old log indices grew too large.
2. ISM retention policy existed but was not attached to indices.
3. Bootstrap job stopped early with HTTP 409 when policy already existed.
4. Existing k8s-logs-* indices stayed unmanaged.
5. Longhorn retained old PVs/volumes.
6. OpenSearch startup had to recover heavy old indices.
7. Node memory was already high.
8. OpenSearch was killed with exit code 137 during recovery/GC pressure.
```

The permanent fix:

```text
1. Stabilize OpenSearch runtime values.
2. Clean old retained storage where needed.
3. Delete unnecessary old k8s-logs-* indices.
4. Make bootstrap job idempotent.
5. Attach ISM retention to existing indices.
6. Ensure future k8s-logs-* indices inherit retention.
7. Verify ArgoCD apps are Synced and Healthy.
```

---

## 24. Do Not Do This Again

Avoid:

```text
Manually changing Kubernetes resources without GitOps reconciliation
Repeatedly deleting pods without checking Last State and logs
Increasing memory blindly without checking ISM/index growth
Moving OpenSearch node without fixing retention
Assuming a completed bootstrap Job means ISM is attached
Using extraEnvs for OPENSEARCH_JAVA_OPTS
Ignoring retained Longhorn PVs
```

---

## 25. Quick Command Reference

```bash
# Pods
kubectl get pods -n observability
kubectl describe pod opensearch-master-0 -n observability
kubectl logs -n observability opensearch-master-0 --tail=120
kubectl logs -n observability opensearch-master-0 --previous --tail=120

# ArgoCD
kubectl get applications -n argocd
argocd app get opensearch --hard-refresh
argocd app sync opensearch
argocd app get opensearch-bootstrap --hard-refresh
argocd app sync opensearch-bootstrap --force

# OpenSearch health
curl -s http://localhost:9200/_cluster/health?pretty
curl -s http://localhost:9200/_cat/indices?v&s=store.size:desc
curl -s http://localhost:9200/_cat/allocation?v

# ISM
curl -s http://localhost:9200/_plugins/_ism/policies?pretty
curl -s http://localhost:9200/_plugins/_ism/explain/k8s-logs-2026.06.12?pretty

# Check all k8s log indices
for index in $(curl -s http://localhost:9200/_cat/indices/k8s-logs-*?h=index); do
  echo "==== $index ===="
  curl -s "http://localhost:9200/_plugins/_ism/explain/${index}?pretty" | grep -E '"policy_id"|"enabled"'
done

# PVC/PV
kubectl get pvc -n observability | grep opensearch
kubectl get pv | grep opensearch
kubectl get volumes.longhorn.io -n longhorn-system | grep -E "opensearch|pvc"

# Node
kubectl top nodes
kubectl describe node talos-w1 | grep -A20 -i "Conditions"
```

---

## 26. Final Stable Evidence

Observed final state:

```text
opensearch-master-0   1/1   Running   24 (9h ago)
```

ArgoCD:

```text
opensearch             Synced   Healthy
opensearch-bootstrap   Synced   Healthy
fluent-bit             Synced   Healthy
```

ISM:

```text
==== k8s-logs-2026.06.12 ====
"policy_id" : "delete-after-3-days"
"enabled" : true
```

Disk:

```text
disk.used:    961.5mb
disk.avail:   18.5gb
disk.total:   19.5gb
disk.percent: 4
```

This confirms the OpenSearch stack is stable and retention-managed.

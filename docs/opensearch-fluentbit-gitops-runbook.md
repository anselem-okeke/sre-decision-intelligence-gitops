# OpenSearch / Fluent Bit GitOps Logging

![img](/img/loggingarc.png)

## 1. Purpose

This document explains the complete setup, troubleshooting, GitOps structure, and operational reasoning behind the **OpenSearch + Fluent Bit logging foundation** in the `sre-decision-intelligence-gitops` repository.

The goal of this phase was to build a production-style logging layer:

```text
Kubernetes workloads
        ↓
Fluent Bit DaemonSet
        ↓
OpenSearch
        ↓
Searchable workload logs
        ↓
Grafana / SRE Decision Intelligence later
```

This phase was implemented using **Argo CD**, **AppProjects**, **Applications**, and an **App-of-Apps/root application** pattern.

---

## 2. Final Status

At the end of this phase:

```text
OpenSearch            Synced   Healthy
Fluent Bit            Synced   Healthy
Argo CD root app      Synced   Healthy
Bank of Anthos        Synced   Healthy
Cilium LAN platform   Synced   Healthy
```

Confirmed working:

```text
OpenSearch is running.
OpenSearch cluster health is green.
Fluent Bit DaemonSet is running on worker nodes.
Kubernetes logs are indexed in OpenSearch.
Bank of Anthos logs from fintech-workload are searchable.
```

---

## 3. Final Architecture

```text
GitHub Repository
        ↓
Argo CD Root App
        ↓
argocd/applications/
        ├── bootstrap-namespaces-app.yaml
        ├── platform-cilium-lan-app.yaml
        ├── bank-of-anthos-app.yaml
        ├── opensearch-app.yaml
        └── fluent-bit-app.yaml

Runtime flow:

Bank of Anthos pods
        ↓
Container logs on Talos nodes
        ↓
Fluent Bit DaemonSet
        ↓
OpenSearch service
        ↓
OpenSearch indices: k8s-logs-YYYY.MM.DD
```

---

## 4. Repository Structure

```text
sre-decision-intelligence-gitops/
├── app-of-apps/
│   └── root-app.yaml
│
├── argocd/
│   ├── projects/
│   │   ├── bootstrap-project.yaml
│   │   ├── fintech-workloads-project.yaml
│   │   ├── observability-project.yaml
│   │   └── platform-networking-project.yaml
│   │
│   └── applications/
│       ├── bank-of-anthos-app.yaml
│       ├── bootstrap-namespaces-app.yaml
│       ├── fluent-bit-app.yaml
│       ├── opensearch-app.yaml
│       └── platform-cilium-lan-app.yaml
│
├── bootstrap/
│   └── namespaces/
│       ├── fintech-workload.yaml
│       ├── observability-namespace.yaml
│       └── kustomization.yaml
│
├── platform/
│   └── cilium-lan/
│       ├── cilium-lb-pool.yaml
│       ├── cilium-l2-policy.yaml
│       └── kustomization.yaml
│
├── workloads/
│   └── bank-of-anthos/
│       ├── base/
│       └── overlays/
│           └── talos/
│
├── evidence/
│   └── phase-4-logging/
│
└── docs/
```

---

## 5. Why This Structure Is Enterprise-Style

The design separates responsibility into layers:

```text
bootstrap/              → foundational namespaces and security labels
argocd/projects/        → governance boundaries
argocd/applications/    → Argo CD child applications
platform/               → shared platform infrastructure
workloads/              → business applications
evidence/               → proof of validation and operational state
```

This avoids the common anti-pattern:

```text
One giant folder that deploys everything with no ownership boundaries.
```

Instead, the design makes ownership explicit:

```text
Bootstrap owns namespace foundations.
Observability owns logging stack.
Platform networking owns Cilium LB/L2.
Fintech workloads own Bank of Anthos.
Argo CD owns reconciliation.
Git owns desired state.
```

---

## 6. AppProject Model

Argo CD `AppProject` objects are used as security and governance boundaries.

They define:

```text
Which repositories are trusted.
Which namespaces can be deployed to.
Which cluster-scoped resources are allowed.
Which applications belong to this domain.
```

---

## 7. Bootstrap Project

### Purpose

The `bootstrap` AppProject is used for foundational GitOps objects, especially namespace management and App-of-Apps/root-app control.

It allows Argo CD to manage:

```text
Namespaces
Argo CD Application objects
```

### Why Bootstrap Exists

Some resources must exist before workloads can run:

```text
Namespaces
Pod Security labels
Base environment boundaries
Root Argo CD app
```

These are not business application resources. They are foundation resources.

### Bootstrap AppProject Example

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: bootstrap
  namespace: argocd
spec:
  description: Bootstrap project for foundational Argo CD and namespace resources

  sourceRepos:
    - 'https://github.com/anselem-okeke/sre-decision-intelligence-gitops.git'

  destinations:
    - namespace: argocd
      server: https://kubernetes.default.svc
    - namespace: '*'
      server: https://kubernetes.default.svc

  clusterResourceWhitelist:
    - group: ""
      kind: Namespace

  namespaceResourceWhitelist:
    - group: argoproj.io
      kind: Application
```

### Explanation

```yaml
clusterResourceWhitelist:
  - group: ""
    kind: Namespace
```

This allows the bootstrap project to manage Kubernetes `Namespace` resources.

A `Namespace` is cluster-scoped, not namespaced, so it must be allowed under `clusterResourceWhitelist`.

```yaml
namespaceResourceWhitelist:
  - group: argoproj.io
    kind: Application
```

This allows the bootstrap project to manage Argo CD `Application` resources inside the `argocd` namespace.

That is important for the App-of-Apps pattern, where the root app manages child applications.

---

## 8. Observability Project

### Purpose

The `observability` AppProject manages:

```text
OpenSearch
Fluent Bit
Future Grafana datasource/dashboard configs
Logging and metrics-related platform components
```

Observability tools are shared platform services, not business workloads.

### Observability AppProject

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: observability
  namespace: argocd
spec:
  description: Observability project for logging, metrics, dashboards, and SRE signal collection

  sourceRepos:
    - 'https://github.com/anselem-okeke/sre-decision-intelligence-gitops.git'
    - 'https://opensearch-project.github.io/helm-charts/'
    - 'https://fluent.github.io/helm-charts'

  destinations:
    - namespace: observability
      server: https://kubernetes.default.svc

  clusterResourceWhitelist:
    - group: ''
      kind: Namespace
    - group: rbac.authorization.k8s.io
      kind: ClusterRole
    - group: rbac.authorization.k8s.io
      kind: ClusterRoleBinding

  namespaceResourceWhitelist:
    - group: '*'
      kind: '*'
```

### Why ClusterRole and ClusterRoleBinding Are Needed

Fluent Bit runs as a DaemonSet and enriches logs with Kubernetes metadata.

To do that, it needs cluster-wide read access to Kubernetes metadata.

The Fluent Bit Helm chart creates:

```text
ClusterRole
ClusterRoleBinding
```

Initially, Argo CD blocked the sync because the `observability` AppProject did not allow those resources.

The error was:

```text
resource rbac.authorization.k8s.io:ClusterRole is not permitted in project observability
resource rbac.authorization.k8s.io:ClusterRoleBinding is not permitted in project observability
```

The fix was to explicitly allow only the required cluster-scoped RBAC resources.

This keeps the project enterprise-style because it does not allow all cluster resources. It allows only what Fluent Bit needs.

---

## 9. OpenSearch Argo CD Application

### Purpose

OpenSearch is the log storage/search backend.

Fluent Bit sends Kubernetes logs to OpenSearch.

### Application Path

```text
argocd/applications/opensearch-app.yaml
```

### Final Working Application

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: opensearch
  namespace: argocd
spec:
  project: observability

  source:
    repoURL: https://opensearch-project.github.io/helm-charts/
    chart: opensearch
    targetRevision: 2.27.0
    helm:
      values: |
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

  destination:
    server: https://kubernetes.default.svc
    namespace: observability

  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

---

## 10. Why `longhorn-retain` Was Used

OpenSearch stores indexed log data.

For stateful systems, using a retaining storage class is safer than using one that deletes volumes automatically.

```yaml
storageClass: longhorn-retain
```

Reason:

```text
If the OpenSearch app or StatefulSet is accidentally deleted, the volume should not automatically disappear.
```

This is more enterprise-style because log data has operational value.

---

## 11. OpenSearch Troubleshooting

### Problem 1 — App Stuck in Progressing

Argo CD showed:

```text
opensearch   Synced   Progressing
```

Meaning:

```text
Argo CD successfully applied the manifests.
Kubernetes workload was not yet healthy.
```

---

## 12. PVC / Scheduling Troubleshooting

Initial scheduler event:

```text
0/5 nodes are available: pod has unbound immediate PersistentVolumeClaims
```

Later the PVC became bound.

Another scheduling issue appeared:

```text
2 Insufficient cpu
3 node(s) had untolerated taint(s)
```

Meaning:

```text
OpenSearch could only schedule on worker nodes.
Control-plane nodes were tainted.
Worker nodes did not have enough available CPU for the original request.
```

### Fix

Reduce CPU request:

```yaml
resources:
  requests:
    cpu: "100m"
    memory: "1Gi"
  limits:
    cpu: "1"
    memory: "2Gi"
```

---

## 13. StatefulSet Immutable Field Error

After changing storage-related values, Argo CD failed with:

```text
StatefulSet.apps "opensearch-master" is invalid:
spec: Forbidden: updates to statefulset spec for fields other than
'replicas', 'ordinals', 'template', 'updateStrategy',
'revisionHistoryLimit', 'persistentVolumeClaimRetentionPolicy'
and 'minReadySeconds' are forbidden
```

### Meaning

Some fields in a StatefulSet cannot be changed after creation, especially volume claim template fields.

Changing storage class from:

```text
longhorn
```

to:

```text
longhorn-retain
```

requires recreating the StatefulSet/PVC during early setup.

### Fix Used

Because this was initial setup and no important data existed yet:

```bash
kubectl delete sts opensearch-master -n observability
kubectl delete pvc opensearch-master-opensearch-master-0 -n observability
```

Then Argo CD recreated OpenSearch using the new desired state.

---

## 14. OpenSearch Security Plugin Error

After OpenSearch scheduled, it still crashed.

Startup probe failed:

```text
Startup probe failed: dial tcp <pod-ip>:9200: connect: connection refused
Back-off restarting failed container opensearch
```

Logs showed:

```text
failed to load plugin class [org.opensearch.security.OpenSearchSecurityPlugin]
Unable to read the file root-ca.pem
```

### Root Cause

The OpenSearch security plugin was still trying to load TLS/security files even though security chart config was disabled.

This happened because:

```yaml
securityConfig:
  enabled: false
```

disables Helm chart security config generation, but OpenSearch itself still needed the security plugin disabled explicitly.

### Final Fix

Add:

```yaml
extraEnvs:
  - name: DISABLE_SECURITY_PLUGIN
    value: "true"

opensearchConfig:
  opensearch.yml: |
    cluster.name: opensearch
    network.host: 0.0.0.0
    plugins.security.disabled: true
```

The critical line is:

```yaml
plugins.security.disabled: true
```

After this, OpenSearch started successfully.

---

## 15. OpenSearch Validation

Port-forward:

```bash
kubectl port-forward -n observability svc/opensearch-cluster-master 9200:9200
```

Root endpoint:

```bash
curl http://localhost:9200
```

Expected response:

```json
{
  "name": "opensearch-master-0",
  "cluster_name": "opensearch",
  "version": {
    "distribution": "opensearch",
    "number": "2.18.0"
  },
  "tagline": "The OpenSearch Project: https://opensearch.org/"
}
```

Cluster health:

```bash
curl http://localhost:9200/_cluster/health?pretty
```

Expected:

```json
{
  "cluster_name": "opensearch",
  "status": "green",
  "number_of_nodes": 1,
  "active_shards_percent_as_number": 100.0
}
```

This confirms:

```text
OpenSearch is running.
OpenSearch is reachable.
OpenSearch cluster health is green.
```

---

## 16. Fluent Bit Argo CD Application

### Purpose

Fluent Bit collects container logs from the Kubernetes nodes and sends them to OpenSearch.

### Application Path

```text
argocd/applications/fluent-bit-app.yaml
```

### Final Application

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: fluent-bit
  namespace: argocd
spec:
  project: observability

  source:
    repoURL: https://fluent.github.io/helm-charts
    chart: fluent-bit
    targetRevision: 0.48.5
    helm:
      values: |
        kind: DaemonSet

        serviceAccount:
          create: true

        config:
          service: |
            [SERVICE]
                Flush         5
                Log_Level     info
                Daemon        off
                Parsers_File  parsers.conf
                HTTP_Server   On
                HTTP_Listen   0.0.0.0
                HTTP_Port     2020

          inputs: |
            [INPUT]
                Name              tail
                Path              /var/log/containers/*.log
                Parser            cri
                Tag               kube.*
                Refresh_Interval  5
                Mem_Buf_Limit     50MB
                Skip_Long_Lines   On

          filters: |
            [FILTER]
                Name                kubernetes
                Match               kube.*
                Kube_URL            https://kubernetes.default.svc:443
                Kube_CA_File        /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
                Kube_Token_File     /var/run/secrets/kubernetes.io/serviceaccount/token
                Merge_Log           On
                Keep_Log            On
                K8S-Logging.Parser  On
                K8S-Logging.Exclude Off

          outputs: |
            [OUTPUT]
                Name                  opensearch
                Match                 kube.*
                Host                  opensearch-cluster-master.observability.svc.cluster.local
                Port                  9200
                Logstash_Format       On
                Logstash_Prefix       k8s-logs
                Suppress_Type_Name    On
                tls                   Off

  destination:
    server: https://kubernetes.default.svc
    namespace: observability

  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

---

## 17. Why Fluent Bit Uses `tls Off`

OpenSearch security was disabled for the initial logging baseline.

Therefore Fluent Bit sends logs using plain HTTP to:

```text
opensearch-cluster-master.observability.svc.cluster.local:9200
```

So the Fluent Bit output uses:

```text
tls Off
```

and no username/password.

Later, when OpenSearch is hardened with TLS and credentials, Fluent Bit should be updated to use:

```text
tls On
HTTP_User
HTTP_Passwd or secret-based credentials
```

---

## 18. Fluent Bit Troubleshooting

### Problem 1 — ClusterRole Not Permitted

Argo CD sync failed:

```text
resource rbac.authorization.k8s.io:ClusterRole is not permitted in project observability
resource rbac.authorization.k8s.io:ClusterRoleBinding is not permitted in project observability
```

### Root Cause

The `observability` AppProject did not allow Fluent Bit’s cluster-scoped RBAC objects.

Fluent Bit needs cluster-wide RBAC to enrich logs with Kubernetes metadata.

### Fix

Update `observability-project.yaml`:

```yaml
clusterResourceWhitelist:
  - group: ''
    kind: Namespace
  - group: rbac.authorization.k8s.io
    kind: ClusterRole
  - group: rbac.authorization.k8s.io
    kind: ClusterRoleBinding
```

Then apply:

```bash
kubectl apply -f argocd/projects/observability-project.yaml
```

Then sync:

```bash
argocd app sync fluent-bit
```

---

## 19. PodSecurity Blocked Fluent Bit

Fluent Bit DaemonSet was created but no pods were running:

```text
daemonset.apps/fluent-bit   DESIRED 2   CURRENT 0   READY 0
```

DaemonSet events showed:

```text
violates PodSecurity "baseline:latest":
hostPath volumes (volumes "varlog", "varlibdockercontainers", "etcmachineid")
```

### Root Cause

The `observability` namespace enforced Pod Security `baseline`.

Fluent Bit requires `hostPath` volumes to read container logs from the node.

Baseline Pod Security does not allow hostPath.

### Manual Fix Used

```bash
kubectl label namespace observability \
  pod-security.kubernetes.io/enforce=privileged \
  pod-security.kubernetes.io/audit=privileged \
  pod-security.kubernetes.io/warn=privileged \
  --overwrite
```

After that, Fluent Bit pods started successfully.

---

## 20. Making Namespace Security Labels GitOps-Managed

Manual namespace labels must be stored in Git.

## Namespace Folder

```text
bootstrap/namespaces/
├── fintech-workload.yaml
├── observability-namespace.yaml
└── kustomization.yaml
```

## Observability Namespace Manifest

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: observability
  labels:
    pod-security.kubernetes.io/enforce: privileged
    pod-security.kubernetes.io/audit: privileged
    pod-security.kubernetes.io/warn: privileged
```

## Kustomization

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - fintech-workload.yaml
  - observability-namespace.yaml
```

## Bootstrap Namespace Application

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: bootstrap-namespaces
  namespace: argocd
spec:
  project: bootstrap

  source:
    repoURL: https://github.com/anselem-okeke/sre-decision-intelligence-gitops.git
    targetRevision: main
    path: bootstrap/namespaces

  destination:
    server: https://kubernetes.default.svc
    namespace: argocd

  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

## Why This Matters

Without this, the namespace label would be a hidden manual cluster fix.

With this, Git becomes the source of truth for:

```text
observability namespace
Pod Security mode
Fluent Bit runtime requirements
```

---

## 21. App-of-Apps Relationship

## Root App

```text
app-of-apps/root-app.yaml
```

The root app watches:

```text
argocd/applications/
```

That folder contains:

```text
bootstrap-namespaces-app.yaml
platform-cilium-lan-app.yaml
bank-of-anthos-app.yaml
opensearch-app.yaml
fluent-bit-app.yaml
```

## Relationship

```text
sre-platform-root
        ↓
argocd/applications/
        ├── bootstrap-namespaces
        ├── platform-cilium-lan
        ├── bank-of-anthos
        ├── opensearch
        └── fluent-bit
```

## Why This Matters

Before App-of-Apps, every application would need to be manually applied:

```bash
kubectl apply -f argocd/applications/opensearch-app.yaml
kubectl apply -f argocd/applications/fluent-bit-app.yaml
```

With App-of-Apps:

```text
Add app YAML to argocd/applications/
Commit and push
Root app detects it
Argo CD creates/syncs the child app
```

This is the enterprise GitOps pattern.

---

## 22. Final Runtime Validation

## Check Argo CD Applications

```bash
kubectl get applications -n argocd
```

Expected:

```text
bank-of-anthos        Synced   Healthy
fluent-bit            Synced   Healthy
opensearch            Synced   Healthy
platform-cilium-lan   Synced   Healthy
sre-platform-root     Synced   Healthy
```

## Check Observability Namespace

```bash
kubectl get all -n observability
```

Expected:

```text
pod/fluent-bit-xxxxx      1/1 Running
pod/fluent-bit-yyyyy      1/1 Running
pod/opensearch-master-0   1/1 Running

daemonset.apps/fluent-bit             DESIRED 2 CURRENT 2 READY 2
statefulset.apps/opensearch-master    READY 1/1
```

## Check OpenSearch Health

```bash
kubectl port-forward -n observability svc/opensearch-cluster-master 9200:9200
curl http://localhost:9200
curl http://localhost:9200/_cluster/health?pretty
```

Expected:

```text
status: green
```

## Check Indices

```bash
curl http://localhost:9200/_cat/indices?v
```

Expected:

```text
k8s-logs-YYYY.MM.DD
```

## Query Any Logs

```bash
curl "http://localhost:9200/k8s-logs-*/_search?pretty" \
  -H 'Content-Type: application/json' \
  -d '{
    "size": 5,
    "query": {
      "match_all": {}
    }
  }'
```

## Query Bank of Anthos Logs

```bash
curl "http://localhost:9200/k8s-logs-*/_search?pretty" \
  -H 'Content-Type: application/json' \
  -d '{
    "size": 5,
    "query": {
      "match": {
        "kubernetes.namespace_name": "fintech-workload"
      }
    }
  }'
```

This confirmed:

```text
Fluent Bit is successfully shipping Kubernetes logs to OpenSearch.
Bank of Anthos logs are searchable from the fintech-workload namespace.
```

---

## 23. Evidence Files

Recommended evidence folder:

```text
evidence/phase-4-logging/
```

Recommended files:

```text
argocd-applications.txt
observability-k8s-status.txt
opensearch-root-response.json
opensearch-cluster-health.json
opensearch-indices.txt
fluent-bit-opensearch-validation.md
bank-of-anthos-log-query.json
```

Example commands:

```bash
mkdir -p evidence/phase-4-logging

kubectl get applications -n argocd > evidence/phase-4-logging/argocd-applications.txt

kubectl get all -n observability > evidence/phase-4-logging/observability-k8s-status.txt

curl http://localhost:9200 > evidence/phase-4-logging/opensearch-root-response.json

curl http://localhost:9200/_cluster/health?pretty > evidence/phase-4-logging/opensearch-cluster-health.json

curl http://localhost:9200/_cat/indices?v > evidence/phase-4-logging/opensearch-indices.txt
```

Commit:

```bash
git add evidence/phase-4-logging
git commit -m "Add Fluent Bit OpenSearch validation evidence"
git push
```

---

## 24. Important Lessons Learned

### 1. Argo CD Sync Success Does Not Mean App Health

OpenSearch was `Synced` but still `Progressing`.

That means:

```text
Argo CD successfully applied manifests.
Kubernetes workload was not yet healthy.
```

Always check both:

```text
SYNC STATUS
HEALTH STATUS
```

### 2. StatefulSet Changes Can Be Immutable

Changing storage class or volume claim templates after creation can fail.

For early setup, delete/recreate the StatefulSet and PVC.

For production, plan storage class and volume retention before first deployment.

### 3. Security Plugin Requires Explicit Disabling

For OpenSearch baseline without TLS:

```yaml
securityConfig:
  enabled: false
extraEnvs:
  - name: DISABLE_SECURITY_PLUGIN
    value: "true"
opensearchConfig:
  opensearch.yml: |
    plugins.security.disabled: true
```

All were required for this setup.

### 4. Fluent Bit Needs Cluster RBAC

Because it reads Kubernetes metadata and node/container logs, it needs:

```text
ClusterRole
ClusterRoleBinding
```

### 5. Fluent Bit Needs Privileged Pod Security Namespace

Because it mounts host paths, the namespace must allow privileged workloads:

```text
pod-security.kubernetes.io/enforce=privileged
```

### 6. Manual Fixes Must Become GitOps State

Manual commands used during troubleshooting:

```bash
kubectl label namespace observability ...
```

must be converted into Git-managed YAML under:

```text
bootstrap/namespaces/
```

Otherwise the cluster contains hidden, unreproducible configuration.

---

## 25. Phase Completion Criteria

```text
OpenSearch is Synced and Healthy.
OpenSearch cluster status is green.
Fluent Bit is Synced and Healthy.
Fluent Bit DaemonSet has ready pods.
OpenSearch contains k8s-logs-* indices.
Bank of Anthos logs are searchable by namespace.
Evidence is committed to Git.
```

Current confirmed status:

```text
OpenSearch running 
OpenSearch green 
Fluent Bit running 
Fluent Bit healthy 
Bank of Anthos logs searchable 
Evidence committed 
```

---

## 26. Next

The next logical phase is:

```text
Grafana → OpenSearch datasource → workload log dashboard
```

Goal:

```text
OpenSearch stores logs.
Grafana visualizes logs.
SRE decision layer later consumes signals.
```

Next work items:

```text
1. Add OpenSearch datasource to Grafana.
2. Create workload log dashboard.
3. Add panels for fintech-workload logs.
4. Add error-rate/log-volume panels.
5. Save dashboard JSON under GitOps.
6. Add evidence screenshots/output.
```

---

## Final Summary

This phase built the logging foundation for the SRE Decision Intelligence platform.

The platform now has:

```text
GitOps-managed OpenSearch
GitOps-managed Fluent Bit
GitOps-managed observability namespace labels
GitOps-managed AppProjects and Applications
Searchable Kubernetes logs
Searchable Bank of Anthos logs
Evidence committed for portfolio/proof
```

The most important enterprise outcome:

```text
Logging is no longer a manual cluster component.
It is defined, governed, deployed, and reconciled through GitOps.
```

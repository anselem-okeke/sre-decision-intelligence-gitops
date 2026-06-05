# Argo CD Enterprise GitOps Setup — Root App, Bootstrap, Projects, Applications, and Namespaces

## 1. Purpose

This document explains the **enterprise-style Argo CD GitOps setup** used in the `sre-decision-intelligence-gitops` repository.

It focuses on the parts that are usually confusing at the beginning:

- What the **root app** does
- What **bootstrap** means
- Why we have **AppProjects**
- Why namespaces are managed separately
- What to apply first
- How Argo CD discovers and manages applications
- How `clusterResourceWhitelist` and `namespaceResourceWhitelist` work
- How OpenSearch, Fluent Bit, Cilium, and Bank of Anthos fit into the structure

The goal is to make the GitOps architecture understandable, repeatable, and suitable for a professional portfolio or enterprise-style platform setup.

---

## 2. High-Level Mental Model

Think of the setup in four layers:

```text
Layer 1: Bootstrap
  Creates foundational resources:
  - namespaces
  - namespace labels
  - root app permissions

Layer 2: Argo CD Governance
  Defines AppProjects:
  - bootstrap
  - observability
  - platform-networking
  - fintech-workloads

Layer 3: Argo CD Applications
  Defines what Argo CD should deploy:
  - bootstrap-namespaces
  - platform-cilium-lan
  - opensearch
  - fluent-bit
  - bank-of-anthos

Layer 4: Runtime Workloads
  Actual Kubernetes resources:
  - namespaces
  - Cilium policies
  - OpenSearch StatefulSet
  - Fluent Bit DaemonSet
  - Bank of Anthos workloads
```

Simple flow:

```text
Git Repository
      ↓
Argo CD Root App
      ↓
argocd/applications/
      ↓
Child Argo CD Applications
      ↓
Kubernetes Cluster
```

---

## 3. Final Repository Structure

The structure is designed to separate responsibilities clearly.

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
│       ├── cilium-l2-policy.yaml
│       ├── cilium-lb-pool.yaml
│       └── kustomization.yaml
│
├── workloads/
│   └── bank-of-anthos/
│       ├── base/
│       └── overlays/
│           └── talos/
│
├── observability/
├── evidence/
├── docs/
├── scripts/
└── README.md
```

---

## 4. What Each Top-Level Folder Means

| Folder | Purpose |
|---|---|
| `app-of-apps/` | Contains the Argo CD root application |
| `argocd/projects/` | Contains AppProject security/governance boundaries |
| `argocd/applications/` | Contains child Argo CD Applications |
| `bootstrap/` | Contains foundational cluster resources like namespaces |
| `platform/` | Contains shared platform infrastructure such as Cilium LAN exposure |
| `workloads/` | Contains business/demo workloads such as Bank of Anthos |
| `evidence/` | Stores validation proof, command outputs, screenshots, and runbooks |
| `docs/` | Stores architecture and explanation documents |

---

## 5. The Most Important Concept

The most important idea is this:

```text
AppProjects define permissions.
Applications define deployments.
Root App discovers Applications.
Bootstrap defines foundations.
```

Another way to say it:

```text
AppProject = governance boundary
Application = deployment instruction
Root App = application manager
Bootstrap = foundation layer
```

---

# 6. Argo CD Root App

## What Is the Root App?

The root app is the parent Argo CD Application.

It watches this folder:

```text
argocd/applications/
```

Every Argo CD Application YAML placed in that folder becomes a child application.

## Root App File

```text
app-of-apps/root-app.yaml
```

Example:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: sre-platform-root
  namespace: argocd
spec:
  project: bootstrap

  source:
    repoURL: https://github.com/anselem-okeke/sre-decision-intelligence-gitops.git
    targetRevision: main
    path: argocd/applications

  destination:
    server: https://kubernetes.default.svc
    namespace: argocd

  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

## What It Does

The root app says:

```text
Argo CD, watch the argocd/applications folder.
Every Application manifest in that folder should exist in the cluster.
If someone changes Git, reconcile the cluster.
If someone changes the cluster manually, self-heal it.
```

## Why It Is Called App-of-Apps

Because one Argo CD Application manages other Argo CD Applications.

```text
sre-platform-root
      ↓
argocd/applications/
      ├── bootstrap-namespaces
      ├── platform-cilium-lan
      ├── opensearch
      ├── fluent-bit
      └── bank-of-anthos
```

## Why This Is Enterprise-Style

Without App-of-Apps, you must manually apply every app:

```bash
kubectl apply -f argocd/applications/opensearch-app.yaml
kubectl apply -f argocd/applications/fluent-bit-app.yaml
kubectl apply -f argocd/applications/bank-of-anthos-app.yaml
```

With App-of-Apps:

```text
Add new Application YAML to argocd/applications/
Commit and push
Root app creates/syncs it
```

This gives a cleaner operating model.

---

# 7. Bootstrap Layer

## What Does Bootstrap Mean?

Bootstrap means foundational resources that must exist before platform services and workloads can run.

In this setup, bootstrap manages:

```text
Namespaces
Pod Security labels
Foundational environment boundaries
Root app permission boundary
```

## Bootstrap Folder

```text
bootstrap/
└── namespaces/
    ├── fintech-workload.yaml
    ├── observability-namespace.yaml
    └── kustomization.yaml
```

## Why Namespaces Are in Bootstrap

Namespaces are not normal application resources.

They are foundational boundaries.

For example:

```text
observability namespace
  → required before OpenSearch and Fluent Bit can run

fintech-workload namespace
  → required before Bank of Anthos can run
```

So they belong in the foundation/bootstrap layer.

---

## 8. Namespace Manifests

### Observability Namespace

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

### Why Observability Is Privileged

Fluent Bit needs hostPath volumes to read node/container logs.

It mounts paths such as:

```text
/var/log
/var/lib/docker/containers
/etc/machine-id
```

Kubernetes Pod Security `baseline` blocks hostPath volumes.

The error was:

```text
violates PodSecurity "baseline:latest":
hostPath volumes (volumes "varlog", "varlibdockercontainers", "etcmachineid")
```

The manual fix was:

```bash
kubectl label namespace observability \
  pod-security.kubernetes.io/enforce=privileged \
  pod-security.kubernetes.io/audit=privileged \
  pod-security.kubernetes.io/warn=privileged \
  --overwrite
```

Then we converted it into GitOps state:

```text
bootstrap/namespaces/observability-namespace.yaml
```

This is important because manual fixes should not remain hidden in the cluster.

---

## 9. Namespace Kustomization

```text
bootstrap/namespaces/kustomization.yaml
```

Example:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - fintech-workload.yaml
  - observability-namespace.yaml
```

## Why Kustomize Is Used Here

Kustomize gives Argo CD a clean directory entry point.

The Argo CD Application can point to:

```text
bootstrap/namespaces
```

and Kustomize tells Argo CD which namespace YAML files are part of that layer.

---

# 10. Bootstrap Namespaces Application

## File

```text
argocd/applications/bootstrap-namespaces-app.yaml
```

## Purpose

This Argo CD Application deploys the namespace manifests under:

```text
bootstrap/namespaces
```

Example:

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

## Why Destination Namespace Is `argocd`

The actual resources are `Namespace` objects, which are cluster-scoped.

They do not live inside `argocd`.

But Argo CD Applications still require a destination namespace field.

Using `argocd` here is acceptable because the resources themselves are cluster-scoped.

---

# 11. AppProjects

## What Is an AppProject?

An AppProject is an Argo CD governance object.

It controls:

```text
Which Git repositories are allowed
Which namespaces are allowed
Which cluster-scoped resources are allowed
Which namespaced resources are allowed
```

## Why AppProjects Matter

Without AppProjects, every app can potentially deploy too much.

With AppProjects, each domain has boundaries.

Example:

```text
fintech-workloads
  → can deploy app resources into fintech-workload namespace

observability
  → can deploy OpenSearch, Fluent Bit, RBAC into observability namespace

platform-networking
  → can deploy Cilium cluster-scoped networking resources

bootstrap
  → can deploy namespaces and Argo CD Application resources
```

---

# 12. Bootstrap AppProject

## File

```text
argocd/projects/bootstrap-project.yaml
```

## Purpose

The bootstrap AppProject allows the root app and namespace bootstrap app to operate.

Example:

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

## Explanation of `clusterResourceWhitelist`

```yaml
clusterResourceWhitelist:
  - group: ""
    kind: Namespace
```

This allows the bootstrap project to manage Kubernetes `Namespace` resources.

Why?

Because `Namespace` is cluster-scoped.

It does not belong inside another namespace.

## Explanation of `namespaceResourceWhitelist`

```yaml
namespaceResourceWhitelist:
  - group: argoproj.io
    kind: Application
```

This allows the bootstrap project to manage Argo CD `Application` resources.

Why?

Because the root app watches:

```text
argocd/applications/
```

and creates child Argo CD Applications inside the `argocd` namespace.

So the bootstrap project must be allowed to create/manage:

```text
Application.argoproj.io
```

---

# 13. Observability AppProject

## File

```text
argocd/projects/observability-project.yaml
```

## Purpose

The observability AppProject manages the logging/monitoring stack.

It currently includes:

```text
OpenSearch
Fluent Bit
Future Grafana dashboards/datasources
```

Example:

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

## Why Observability Allows ClusterRole and ClusterRoleBinding

Fluent Bit needs Kubernetes metadata.

To enrich logs with pod names, namespaces, labels, and container details, Fluent Bit needs read access to the Kubernetes API.

The Fluent Bit Helm chart creates:

```text
ClusterRole
ClusterRoleBinding
```

Initially, Argo CD blocked this with:

```text
resource rbac.authorization.k8s.io:ClusterRole is not permitted in project observability
resource rbac.authorization.k8s.io:ClusterRoleBinding is not permitted in project observability
```

The fix was to explicitly allow only those cluster-scoped resources.

That keeps the project controlled.

---

# 14. Platform Networking AppProject

## File

```text
argocd/projects/platform-networking-project.yaml
```

## Purpose

This project manages Cilium LAN LoadBalancer and L2 announcement resources.

Example:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: platform-networking
  namespace: argocd
spec:
  description: Platform networking project for Cilium LAN LoadBalancer and L2 announcement

  sourceRepos:
    - 'https://github.com/anselem-okeke/sre-decision-intelligence-gitops.git'

  destinations:
    - namespace: '*'
      server: https://kubernetes.default.svc

  clusterResourceWhitelist:
    - group: cilium.io
      kind: CiliumLoadBalancerIPPool
    - group: cilium.io
      kind: CiliumL2AnnouncementPolicy

  namespaceResourceWhitelist: []
```

## Why Cluster Resource Whitelist Is Needed

Cilium resources such as:

```text
CiliumLoadBalancerIPPool
CiliumL2AnnouncementPolicy
```

are cluster-scoped.

So the AppProject must explicitly allow them.

---

# 15. Fintech Workloads AppProject

## File

```text
argocd/projects/fintech-workloads-project.yaml
```

## Purpose

This AppProject manages Bank of Anthos.

Example:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: fintech-workloads
  namespace: argocd
spec:
  description: Workload project for fintech applications managed by GitOps

  sourceRepos:
    - 'https://github.com/anselem-okeke/sre-decision-intelligence-gitops.git'

  destinations:
    - namespace: fintech-workload
      server: https://kubernetes.default.svc

  clusterResourceWhitelist: []

  namespaceResourceWhitelist:
    - group: '*'
      kind: '*'
```

## Why `clusterResourceWhitelist` Is Empty

Bank of Anthos should not create cluster-wide resources.

It should only deploy namespaced workload resources into:

```text
fintech-workload
```

Examples:

```text
Deployment
Service
ConfigMap
Secret
ServiceAccount
```

This is safer and cleaner.

Namespaces are managed by bootstrap, not by the workload app.

---

# 16. Argo CD Applications

## What Is an Application?

An Argo CD Application tells Argo CD:

```text
Where the source is
Which project controls it
Which path/chart to deploy
Which cluster/namespace to deploy into
Whether to auto-sync and self-heal
```

Example structure:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: example-app
  namespace: argocd
spec:
  project: some-project

  source:
    repoURL: ...
    targetRevision: main
    path: ...

  destination:
    server: https://kubernetes.default.svc
    namespace: some-namespace

  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

---

# 17. Current Applications and Their Meaning

| Application | Project | Source | Purpose |
|---|---|---|---|
| `sre-platform-root` | `bootstrap` | `argocd/applications` | Manages child apps |
| `bootstrap-namespaces` | `bootstrap` | `bootstrap/namespaces` | Manages namespaces and labels |
| `platform-cilium-lan` | `platform-networking` | `platform/cilium-lan` | Manages Cilium LAN LB/L2 |
| `opensearch` | `observability` | OpenSearch Helm chart | Deploys OpenSearch |
| `fluent-bit` | `observability` | Fluent Bit Helm chart | Deploys Fluent Bit |
| `bank-of-anthos` | `fintech-workloads` | `workloads/bank-of-anthos/overlays/talos` | Deploys workload |

---

# 18. How Everything Connects

## GitOps Control Flow

```text
GitHub repo
   ↓
sre-platform-root
   ↓
argocd/applications/
   ↓
Child Applications
   ↓
Kubernetes resources
```

## Bootstrap Flow

```text
bootstrap-namespaces app
   ↓
bootstrap/namespaces/kustomization.yaml
   ↓
observability namespace
fintech-workload namespace
   ↓
Platform/workload apps can deploy into those namespaces
```

## Observability Flow

```text
opensearch app
   ↓
OpenSearch StatefulSet
   ↓
OpenSearch service

fluent-bit app
   ↓
Fluent Bit DaemonSet
   ↓
Ships logs to OpenSearch
```

## Workload Logging Flow

```text
Bank of Anthos pods
   ↓
Container logs
   ↓
Fluent Bit
   ↓
OpenSearch
   ↓
Searchable k8s-logs-* indices
```

---

# 19. What to Apply First

This is the most important operational sequence.

## First-Time Cluster Bootstrap Order

### Step 1 — Install Argo CD

Argo CD itself must exist first.

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f <argocd-install-manifest>
```

Then confirm:

```bash
kubectl get pods -n argocd
```

---

### Step 2 — Apply AppProjects

AppProjects must exist before Applications that reference them.

Apply:

```bash
kubectl apply -f argocd/projects/bootstrap-project.yaml
kubectl apply -f argocd/projects/platform-networking-project.yaml
kubectl apply -f argocd/projects/observability-project.yaml
kubectl apply -f argocd/projects/fintech-workloads-project.yaml
```

Confirm:

```bash
kubectl get appproject -n argocd
```

Expected:

```text
bootstrap
platform-networking
observability
fintech-workloads
```

Why first?

Because this will fail if an Application references a project that does not exist:

```text
Application references project observability, but project does not exist
```

---

### Step 3 — Apply the Root App

Apply:

```bash
kubectl apply -f app-of-apps/root-app.yaml
```

Confirm:

```bash
kubectl get applications -n argocd
```

Expected:

```text
sre-platform-root
```

Why now?

Because the root app watches:

```text
argocd/applications/
```

and will create/sync all child apps found there.

---

### Step 4 — Root App Creates Child Applications

After root app syncs, it should create:

```text
bootstrap-namespaces
platform-cilium-lan
opensearch
fluent-bit
bank-of-anthos
```

Check:

```bash
kubectl get applications -n argocd
```

---

### Step 5 — Sync Foundational Apps First

The clean order is:

```text
1. bootstrap-namespaces
2. platform-cilium-lan
3. opensearch
4. fluent-bit
5. bank-of-anthos
```

You can sync manually if needed:

```bash
argocd app sync bootstrap-namespaces
argocd app sync platform-cilium-lan
argocd app sync opensearch
argocd app sync fluent-bit
argocd app sync bank-of-anthos
```

In practice, automated sync can handle this, but during learning/troubleshooting manual sync helps visibility.

---

# 20. Why This Order Matters

## Namespaces Before Apps

OpenSearch and Fluent Bit need:

```text
observability namespace
```

Bank of Anthos needs:

```text
fintech-workload namespace
```

So namespaces must be created first.

## Platform Networking Before LAN Exposure

Bank of Anthos frontend uses a LoadBalancer service with Cilium L2/LB behavior.

So Cilium LAN resources should exist before exposing services.

## OpenSearch Before Fluent Bit

Fluent Bit ships logs to OpenSearch.

So OpenSearch should be healthy before Fluent Bit is expected to deliver logs.

## Fluent Bit Before Log Validation

Fluent Bit must run before OpenSearch indices appear.

---

# 21. Current Operational Status

Final confirmed state:

```text
bank-of-anthos        Synced   Healthy
fluent-bit            Synced   Healthy
opensearch            Synced   Healthy
platform-cilium-lan   Synced   Healthy
sre-platform-root     Synced   Healthy
```

Observability namespace:

```text
pod/fluent-bit-xxxxx      1/1 Running
pod/fluent-bit-yyyyy      1/1 Running
pod/opensearch-master-0   1/1 Running

daemonset.apps/fluent-bit   DESIRED 2 CURRENT 2 READY 2
statefulset.apps/opensearch-master   READY 1/1
```

---

# 22. Common Confusions Explained

## Confusion 1 — Why do we need AppProjects?

Because Applications need boundaries.

Without AppProjects, every app can potentially deploy too much.

AppProjects make the platform more enterprise-like by limiting what each application can do.

---

## Confusion 2 — Why root app and not just apply apps manually?

Manual apply works, but it does not scale.

The root app allows:

```text
Git commit creates new Argo CD child app
```

instead of:

```text
Manually kubectl apply every new Application YAML
```

---

## Confusion 3 — Why bootstrap/namespaces instead of platform/namespaces?

Both can work.

But `bootstrap/namespaces` is clearer because namespaces are foundational.

They are not a specific platform service like Fluent Bit or Cilium.

They are base infrastructure boundaries.

---

## Confusion 4 — Why does bootstrap project manage Application resources?

Because the root app belongs to the bootstrap project.

The root app creates child Argo CD Applications.

Therefore the bootstrap project must allow:

```yaml
namespaceResourceWhitelist:
  - group: argoproj.io
    kind: Application
```

---

## Confusion 5 — Why does bootstrap project allow Namespace as cluster resource?

Because Kubernetes Namespace is cluster-scoped.

It is not created inside another namespace.

Therefore:

```yaml
clusterResourceWhitelist:
  - group: ""
    kind: Namespace
```

is required.

---

## Confusion 6 — Why does Fluent Bit need privileged namespace labels?

Because Fluent Bit needs hostPath access to node log files.

Pod Security `baseline` blocks hostPath.

So the `observability` namespace needs:

```yaml
pod-security.kubernetes.io/enforce: privileged
```

---

## Confusion 7 — Why is OpenSearch an Application but not under platform/?

In this setup, OpenSearch is deployed directly from the Helm chart repository through an Argo CD Application.

The application source is:

```yaml
repoURL: https://opensearch-project.github.io/helm-charts/
chart: opensearch
```

So the desired state lives inside the Application Helm values.

Later, if you want a more Git-native structure, you can move Helm values into:

```text
observability/opensearch/values.yaml
```

or:

```text
platform/opensearch/
```

But the current setup is valid.

---

# 23. Troubleshooting Summary

## Fluent Bit RBAC Error

Error:

```text
resource rbac.authorization.k8s.io:ClusterRole is not permitted in project observability
resource rbac.authorization.k8s.io:ClusterRoleBinding is not permitted in project observability
```

Fix:

```yaml
clusterResourceWhitelist:
  - group: rbac.authorization.k8s.io
    kind: ClusterRole
  - group: rbac.authorization.k8s.io
    kind: ClusterRoleBinding
```

Apply:

```bash
kubectl apply -f argocd/projects/observability-project.yaml
argocd app sync fluent-bit
```

---

## Fluent Bit PodSecurity Error

Error:

```text
violates PodSecurity "baseline:latest": hostPath volumes
```

Fix in Git:

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

Path:

```text
bootstrap/namespaces/observability-namespace.yaml
```

---

## OpenSearch Security Plugin Error

Error:

```text
Unable to read the file root-ca.pem
failed to load plugin class [org.opensearch.security.OpenSearchSecurityPlugin]
```

Fix:

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

---

## OpenSearch StatefulSet Immutable Error

Error:

```text
StatefulSet.apps "opensearch-master" is invalid:
spec: Forbidden: updates to statefulset spec for fields other than ...
```

Cause:

```text
Storage-related StatefulSet fields are immutable.
```

Early setup fix:

```bash
kubectl delete sts opensearch-master -n observability
kubectl delete pvc opensearch-master-opensearch-master-0 -n observability
argocd app sync opensearch
```

---

# 24. Daily Validation Commands

## Check Argo CD apps

```bash
kubectl get applications -n argocd
```

## Check AppProjects

```bash
kubectl get appproject -n argocd
```

## Check observability namespace

```bash
kubectl get all -n observability
```

## Check namespace labels

```bash
kubectl get ns observability --show-labels
kubectl get ns fintech-workload --show-labels
```

## Check OpenSearch health

```bash
kubectl port-forward -n observability svc/opensearch-cluster-master 9200:9200
curl http://localhost:9200/_cluster/health?pretty
```

## Check OpenSearch indices

```bash
curl http://localhost:9200/_cat/indices?v
```

## Query all logs

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

## Query Bank of Anthos logs

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

---

# 25. Recommended Bootstrap From Scratch

If the cluster is empty except Argo CD, use this order:

```bash
# 1. Apply AppProjects
kubectl apply -f argocd/projects/bootstrap-project.yaml
kubectl apply -f argocd/projects/platform-networking-project.yaml
kubectl apply -f argocd/projects/observability-project.yaml
kubectl apply -f argocd/projects/fintech-workloads-project.yaml

# 2. Apply root app
kubectl apply -f app-of-apps/root-app.yaml

# 3. Watch apps
kubectl get applications -n argocd -w

# 4. Sync foundational resources if needed
argocd app sync bootstrap-namespaces

# 5. Sync platform networking if needed
argocd app sync platform-cilium-lan

# 6. Sync OpenSearch
argocd app sync opensearch

# 7. Sync Fluent Bit
argocd app sync fluent-bit

# 8. Sync workload
argocd app sync bank-of-anthos
```

---

# 26. What Should Be Committed to Git

Always commit:

```text
AppProjects
Applications
Namespace manifests
Kustomization files
Helm values
Platform manifests
Workload manifests
Evidence files
Documentation
```

Example:

```bash
git add -A
git commit -m "Document Argo CD enterprise GitOps architecture"
git push
```

---

# 27. What Should Not Stay Manual

These should not remain only manual cluster changes:

```text
Namespace labels
RBAC permission fixes
Application definitions
Helm values
Cilium LB/L2 policies
Workload service labels
```

Manual troubleshooting is fine.

But final desired state must go into Git.

---

# 28. Final Architecture Summary

```text
GitHub Repo
   ↓
Argo CD Root App
   ↓
argocd/applications/
   ↓
Child Applications
   ↓
Kubernetes Cluster
```

```text
bootstrap-namespaces
   ↓
Creates observability + fintech-workload namespaces

platform-cilium-lan
   ↓
Creates Cilium LAN LoadBalancer/L2 resources

opensearch
   ↓
Creates OpenSearch StatefulSet and services

fluent-bit
   ↓
Creates Fluent Bit DaemonSet and RBAC

bank-of-anthos
   ↓
Creates fintech workload resources
```

---

# 29. Enterprise Interpretation

This setup demonstrates:

```text
GitOps reconciliation
App-of-Apps pattern
Project-based governance
Least-privilege resource whitelisting
Namespace security policy management
Platform/workload separation
Observability foundation
Evidence-driven validation
```

It is no longer just:

```text
kubectl apply some YAML
```

It is now:

```text
Git-managed platform architecture
```

---

# 30. One-Sentence Explanation

This repository uses Argo CD App-of-Apps with AppProjects to manage bootstrap namespaces, platform networking, observability, and workloads as separate GitOps-governed layers in a Talos Kubernetes cluster.

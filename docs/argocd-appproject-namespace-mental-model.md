# Argo CD AppProject, Namespace, and GitOps Mental Model

## 1. The key idea

You removed the **Namespace manifest** from the Bank of Anthos workload, but Argo CD still knows the namespace because the namespace is also referenced as a **deployment destination**.

These two are different things:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: fintech-workload
```

This creates the namespace.

But this:

```yaml
destinations:
  - namespace: fintech-workload
    server: https://kubernetes.default.svc
```

Does **not** create the namespace. It tells Argo CD:

> Applications in this project are allowed to deploy into the existing `fintech-workload` namespace on this Kubernetes cluster.

---

## 2. Three different namespace concepts

In your setup, `fintech-workload` can appear in three different places.

| Location | Example | Meaning |
|---|---|---|
| Platform namespace manifest | `kind: Namespace` | Creates the namespace |
| Argo CD Application destination | `destination.namespace: fintech-workload` | Tells Argo CD where to deploy the app |
| Kustomize overlay | `namespace: fintech-workload` | Adds `metadata.namespace` to rendered workload resources |

Simple mental model:

```text
kind: Namespace
  = create the room

destination.namespace
  = tell Argo CD which room to deploy into

kustomize namespace:
  = stamp resources with that room name
```

---

## 3. Your AppProject

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

Plain English:

> Applications in the `fintech-workloads` project may deploy any namespaced workload resources from your GitOps repo into the `fintech-workload` namespace of this cluster, but they may not create cluster-level resources like Namespaces.

---

## 4. `apiVersion`

```yaml
apiVersion: argoproj.io/v1alpha1
```

This means the object belongs to the Argo CD API group:

```text
argoproj.io
```

It is not a normal Kubernetes resource like a `Deployment` or `Service`. It is an Argo CD custom resource.

Mental model:

```text
Kubernetes native resources:
  Deployment, Service, ConfigMap, Secret

Argo CD custom resources:
  Application, AppProject
```

---

## 5. `kind: AppProject`

```yaml
kind: AppProject
```

An `AppProject` is a permission boundary for Argo CD applications.

Mental model:

```text
AppProject = security boundary / policy box
```

It controls:

```text
Which Git repos are allowed?
Which clusters are allowed?
Which namespaces are allowed?
Which resource kinds are allowed?
```

---

## 6. `metadata.name`

```yaml
metadata:
  name: fintech-workloads
```

This is the project name.

Your Bank of Anthos `Application` references it like this:

```yaml
spec:
  project: fintech-workloads
```

Relationship:

```text
Application: bank-of-anthos
        ↓ belongs to
AppProject: fintech-workloads
        ↓ controls permissions
```

---

## 7. `metadata.namespace`

```yaml
metadata:
  namespace: argocd
```

This means the `AppProject` object itself lives in the `argocd` namespace.

Important distinction:

```text
AppProject lives in: argocd
Workload deploys to: fintech-workload
```

The `argocd` namespace is where the Argo CD control plane and Argo CD objects live.

The `fintech-workload` namespace is where the Bank of Anthos runtime resources live.

---

## 8. `sourceRepos`

```yaml
sourceRepos:
  - 'https://github.com/anselem-okeke/sre-decision-intelligence-gitops.git'
```

This means:

> Applications in this project may only deploy manifests from this Git repository.

Allowed:

```text
https://github.com/anselem-okeke/sre-decision-intelligence-gitops.git
```

Blocked:

```text
https://github.com/random-user/random-repo.git
```

Mental model:

```text
sourceRepos = trusted Git sources
```

This protects your cluster from an Application pulling YAML from an unexpected repository.

---

## 9. `destinations`

```yaml
destinations:
  - namespace: fintech-workload
    server: https://kubernetes.default.svc
```

This means:

> Applications in this project may deploy only to the `fintech-workload` namespace on this Kubernetes cluster.

It has two parts:

```yaml
namespace: fintech-workload
server: https://kubernetes.default.svc
```

---

## 10. `namespace: fintech-workload`

This is the allowed target namespace.

It does **not** create the namespace.

It only says:

```text
Allowed destination namespace = fintech-workload
```

So if the Application says:

```yaml
destination:
  namespace: fintech-workload
```

Argo CD allows it.

But if the Application tries to deploy to:

```yaml
destination:
  namespace: kube-system
```

Argo CD blocks it, because `kube-system` is not listed as an allowed destination namespace in this AppProject.

---

## 11. `server: https://kubernetes.default.svc`

```yaml
server: https://kubernetes.default.svc
```

This is the Kubernetes API server address from inside the cluster.

Mental model:

```text
server = which Kubernetes cluster
namespace = where inside that cluster
```

Because Argo CD runs inside Kubernetes, it can talk to the same cluster using:

```text
https://kubernetes.default.svc
```

This is the internal Kubernetes service address for the Kubernetes API server.

So this destination means:

```text
Deploy to the same cluster where Argo CD is running.
```

In your homelab:

```text
Argo CD running in homelab cluster
        ↓
deploying Bank of Anthos into same homelab cluster
```

If you had multiple clusters, you could see different server values.

Example:

```yaml
destinations:
  - namespace: fintech-workload
    server: https://kubernetes.default.svc

  - namespace: staging
    server: https://my-staging-cluster-api

  - namespace: production
    server: https://my-prod-cluster-api
```

---

## 12. `clusterResourceWhitelist: []`

```yaml
clusterResourceWhitelist: []
```

This means:

> This project is not allowed to create cluster-scoped resources.

Cluster-scoped resources are resources that do not live inside a namespace.

Examples:

```text
Namespace
ClusterRole
ClusterRoleBinding
CustomResourceDefinition
StorageClass
PersistentVolume
Node
```

This is why the earlier sync failed when your workload app still rendered a `Namespace` resource.

Your workload app was trying to create:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: fintech-workload
```

But the AppProject said:

```yaml
clusterResourceWhitelist: []
```

So Argo CD blocked it.

This is good for a workload project.

Mental model:

```text
fintech-workloads project should deploy apps,
not modify the whole cluster.
```

---

## 13. `namespaceResourceWhitelist`

```yaml
namespaceResourceWhitelist:
  - group: '*'
    kind: '*'
```

This means:

> This project is allowed to create any namespaced resource kind inside the allowed destination namespace.

Namespaced resources include:

```text
Deployment
StatefulSet
Service
ConfigMap
Secret
ServiceAccount
Ingress
Pod
Job
CronJob
Role
RoleBinding
NetworkPolicy
PersistentVolumeClaim
```

Because you used:

```yaml
group: '*'
kind: '*'
```

you are saying:

```text
Allow all namespaced resource types.
```

This is permissive and fine for learning.

For stricter enterprise-style GitOps, you can restrict it like this:

```yaml
namespaceResourceWhitelist:
  - group: ""
    kind: ConfigMap
  - group: ""
    kind: Secret
  - group: ""
    kind: Service
  - group: ""
    kind: ServiceAccount
  - group: apps
    kind: Deployment
  - group: apps
    kind: StatefulSet
  - group: networking.k8s.io
    kind: Ingress
```

That says:

> This workload project can only create the resource types it actually needs.

---

## 14. What is `group`?

Kubernetes resources are organized by API group.

Some resources are in the core group. In YAML, the core group is written as an empty string:

```yaml
group: ""
```

Core group examples:

```text
Service
ConfigMap
Secret
ServiceAccount
Namespace
Pod
PersistentVolumeClaim
```

Other resources belong to named API groups:

```text
Deployment      → apps
StatefulSet     → apps
Ingress         → networking.k8s.io
Job             → batch
CronJob         → batch
Role            → rbac.authorization.k8s.io
NetworkPolicy   → networking.k8s.io
```

So:

```yaml
group: '*'
kind: '*'
```

means:

```text
Any API group, any resource kind
```

But because it is under:

```yaml
namespaceResourceWhitelist
```

it only applies to namespaced resources.

It does not allow cluster-scoped resources like `Namespace` because those are controlled by:

```yaml
clusterResourceWhitelist
```

---

## 15. How Argo CD makes the sync decision

When you run:

```bash
argocd app sync bank-of-anthos
```

Argo CD does this:

```text
1. Read Application bank-of-anthos
2. See project: fintech-workloads
3. Read AppProject fintech-workloads
4. Pull repo from sourceRepos
5. Render path workloads/bank-of-anthos/overlays/talos
6. Check every rendered resource:
   - Is the repo allowed?
   - Is the destination cluster allowed?
   - Is the destination namespace allowed?
   - Is this resource kind allowed?
7. Apply allowed resources
8. Compare live state vs Git state
9. Report Synced/OutOfSync and Healthy/Progressing/Degraded
```

---

## 16. Why everything works now

Before:

```text
workload rendered kind: Namespace
        ↓
AppProject had clusterResourceWhitelist: []
        ↓
Namespace was blocked
        ↓
sync failed
```

Now:

```text
workload renders only namespaced resources
        ↓
Application targets fintech-workload
        ↓
AppProject allows fintech-workload destination
        ↓
namespace already exists from platform layer
        ↓
sync succeeds
```

---

## 17. Final mental model

```text
AppProject does not create resources.
It defines what Applications are allowed to create.
```

```text
Application does not contain the app.
It points Argo CD to the app path in Git.
```

```text
Kustomize does not deploy by itself.
It renders the final YAML that Argo CD applies.
```

```text
Namespace manifest creates the namespace.
destination.namespace tells Argo CD where to deploy.
kustomize namespace stamps resources with that namespace.
```

---

## 18. One-line summary

```text
AppProject = guardrails
Application = pointer to Git path
Kustomize overlay = final rendered YAML for an environment
Namespace manifest = creates the namespace
Destination namespace = where Argo CD is allowed to deploy
Server URL = which Kubernetes cluster Argo CD deploys to
```

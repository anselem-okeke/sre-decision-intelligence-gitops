# SRE Decision Intelligence GitOps — Argo CD Migration & Troubleshooting Runbook

## 1. Goal

Move the **Bank of Anthos / fintech workload** from a manually deployed Kubernetes application to an **enterprise-style GitOps deployment managed by Argo CD**.

Current target application:

```text
workloads/bank-of-anthos/overlays/talos
```

Target namespace:

```text
fintech-workload
```

Target GitOps repository:

```text
https://github.com/anselem-okeke/sre-decision-intelligence-gitops.git
```

---

## 2. High-level architecture

```text
GitHub GitOps Repo
        |
        v
Argo CD Application
        |
        v
Kustomize Overlay
        |
        v
fintech-workload Namespace
        |
        v
Bank of Anthos Services / Deployments / StatefulSets
        |
        v
Cilium LoadBalancer VIP
        |
        v
Browser / LAN Access
```

---

## 3. Enterprise GitOps repo structure

Recommended structure:

```text
sre-decision-intelligence-gitops/
├── argocd/
│   ├── projects/
│   │   └── sre-platform-project.yaml
│   └── applications/
│       └── bank-of-anthos-app.yaml
│
├── app-of-apps/
│   └── root-app.yaml                 # later phase
│
├── platform/
│   └── cilium/                       # later phase
│       ├── lb-pool.yaml
│       ├── l2-policy.yaml
│       └── kustomization.yaml
│
├── workloads/
│   └── bank-of-anthos/
│       ├── base/
│       │   ├── frontend.yaml
│       │   ├── namespace.yaml
│       │   ├── kustomization.yaml
│       │   └── ...
│       └── overlays/
│           └── talos/
│               └── kustomization.yaml
│
└── README.md
```

Enterprise separation:

```text
argocd/projects       = governance boundaries
argocd/applications  = Argo CD Application definitions
app-of-apps          = root app / bootstrap layer
platform             = cluster platform configuration
workloads            = business applications
observability        = monitoring/logging/dashboard configuration
apps                 = custom platform/API applications
```

---

## 4. Manual deployment cleanup

Before allowing Argo CD to manage the application, the manually deployed app was reviewed.

Commands:

```bash
kubectl get all -n fintech-workload
kubectl get pvc -n fintech-workload
kubectl get configmap,secret -n fintech-workload
```

Important result:

```text
No PVCs found in fintech-workload
```

Meaning:

```text
The namespace can be deleted safely if needed.
There is no persistent database volume to preserve.
```

Cleanup option:

```bash
kubectl delete namespace fintech-workload
```

Verification:

```bash
kubectl get namespace fintech-workload
kubectl get all -n fintech-workload
```

Expected if deleted:

```text
Error from server (NotFound): namespaces "fintech-workload" not found
```

Deleting the namespace removes:

```text
Pods
Deployments
ReplicaSets
StatefulSets
Services
ConfigMaps
Secrets
ServiceAccounts
Endpoints
```

It does **not** delete platform-level resources such as:

```text
CiliumLoadBalancerIPPool
CiliumL2AnnouncementPolicy
Cilium config
Argo CD
Talos nodes
kube-system resources
```

---

## 5. Cilium LoadBalancer / L2 issue

### Symptom

The frontend service had a LoadBalancer IP:

```text
frontend LoadBalancer 192.168.0.231
```

But this failed:

```bash
curl -I http://192.168.0.231
```

Error:

```text
curl: (7) Failed to connect to 192.168.0.231 port 80
```

### App health test

Port-forward worked:

```bash
kubectl -n fintech-workload port-forward svc/frontend 8081:80
curl -I http://localhost:8081
```

This proved:

```text
Application = OK
Service = OK
Pod endpoints = OK
LoadBalancer/VIP advertisement = problem
```

### NodePort test

The frontend NodePort worked through the Cilium-selected node:

```bash
curl -I http://192.168.0.241:32369
```

Result:

```text
HTTP/1.1 200 OK
```

This proved:

```text
jumpbox → node IP → NodePort → frontend pod works
```

The broken layer was only:

```text
jumpbox → LoadBalancer VIP → frontend
```

---

## 6. Cilium L2 diagnosis

Cilium was installed and MetalLB was not installed:

```bash
kubectl get pods -A | grep -Ei "metallb|cilium|kube-vip"
kubectl get pods -n metallb-system
```

Result:

```text
Cilium running
MetalLB not installed
```

Cilium L2 was enabled:

```bash
kubectl -n kube-system get cm cilium-lan-config -o yaml | grep -Ei "l2|loadbalancer|kube-proxy|external"
```

Important output:

```text
enable-l2-announcements: "true"
kube-proxy-replacement: "true"
```

Cilium had selected a node to own the frontend VIP:

```bash
kubectl get lease -n kube-system | grep -i cilium-lan
```

Important output:

```text
cilium-l2announce-fintech-workload-frontend   talos-4lc-a3a
```

Owner node:

```text
talos-4lc-a3a = 192.168.0.241
```

---

## 7. Root cause of LoadBalancer failure

The Cilium L2 announcement policy was restricted to one interface:

```yaml
spec:
  interfaces:
    - ens18
```

Even though Cilium owned the VIP, the VIP was not reachable from the LAN.

### Working fix

Remove the interface restriction:

```bash
kubectl patch ciliuml2announcementpolicy lan-lb-services-policy \
  --type=json \
  -p='[
    {"op": "remove", "path": "/spec/interfaces"}
  ]'
```

After this, the LoadBalancer VIP became reachable:

```bash
curl -I http://192.168.0.231
```

### Final working L2 policy shape

```yaml
apiVersion: cilium-lan.io/v2alpha1
kind: CiliumL2AnnouncementPolicy
metadata:
  name: lan-lb-services-policy
spec:
  externalIPs: true
  loadBalancerIPs: true
  serviceSelector:
    matchLabels:
      lan-exposure: enabled
```

Important:

```text
No interfaces restriction.
```

---

## 8. Frontend service fix

The frontend Service must include the label used by the Cilium L2 policy:

```yaml
lan-exposure: enabled
```

### Correct frontend Service

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    application: bank-of-anthos
    environment: development
    team: frontend
    tier: web
    lan-exposure: enabled
  name: frontend
spec:
  ports:
    - name: http
      port: 80
      targetPort: 8080
  selector:
    app: frontend
    application: bank-of-anthos
    environment: development
    team: frontend
    tier: web
  type: LoadBalancer
```

Why this matters:

```text
frontend Service label
        ↓
CiliumL2AnnouncementPolicy serviceSelector
        ↓
Cilium advertises the VIP
        ↓
http://192.168.0.231 works
```

---

## 9. Kustomize overlay fix

The Talos overlay should set the target namespace:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: fintech-workload

resources:
  - ../../base
```

This ensures namespaced resources render into:

```text
fintech-workload
```

Validation command:

```bash
kubectl kustomize workloads/bank-of-anthos/overlays/talos | head -80
```

Expected example:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: bank-of-anthos
  namespace: fintech-workload
```

---

## 10. Correct way to verify frontend labels

This command can be misleading:

```bash
kubectl kustomize workloads/bank-of-anthos/overlays/talos | grep -A20 "name: frontend"
```

Reason:

```text
The labels appear above "name: frontend", so grep -A may not show them.
```

Better command:

```bash
kubectl kustomize workloads/bank-of-anthos/overlays/talos | grep -B10 -A20 "name: frontend"
```

Expected:

```yaml
labels:
  application: bank-of-anthos
  environment: development
  team: frontend
  tier: web
  lan-exposure: enabled
name: frontend
namespace: fintech-workload
```

---

## 11. Manual validation before Argo CD

Before handing control to Argo CD, the workload was manually tested:

```bash
kubectl apply -k workloads/bank-of-anthos/overlays/talos
```

Then checked:

```bash
kubectl get all -n fintech-workload
```

Expected healthy state:

```text
accounts-db-0                         1/1 Running
balancereader                         1/1 Running
contacts                              1/1 Running
frontend                              1/1 Running
ledger-db-0                           1/1 Running
ledgerwriter                          1/1 Running
loadgenerator                         1/1 Running
transactionhistory                    1/1 Running
userservice                           1/1 Running
```

Frontend service:

```text
service/frontend   LoadBalancer   192.168.0.231   80:<nodeport>/TCP
```

This proved:

```text
Git/Kustomize manifest works manually
frontend LoadBalancer works
Cilium L2 issue is fixed
Application is healthy
```

---

## 12. Git commit after fixing the app

After confirming the app worked manually, the working state must be committed and pushed.

Commands:

```bash
git status
git add workloads/bank-of-anthos/base/frontend.yaml
git add workloads/bank-of-anthos/overlays/talos/kustomization.yaml
git commit -m "Prepare Bank of Anthos for Argo CD deployment"
git push
```

Why:

```text
Manual kubectl fix = fixes the live cluster
Git commit = fixes the source of truth
Argo CD reads from Git, not from local manual changes
```

---

## 13. Argo CD AppProject

### Purpose

An AppProject is an Argo CD security/governance boundary.

It defines:

```text
Which Git repositories are trusted
Which clusters/namespaces apps can deploy to
Which Kubernetes resource types are allowed
Which applications belong together
```

### Enterprise naming

For the fintech workload layer:

```text
fintech-workloads
```

This is better than a generic `workloads` project because it defines a domain boundary.

### Current AppProject

File:

```text
argocd/projects/sre-platform-project.yaml
```

Manifest:

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

Apply:

```bash
kubectl apply -f argocd/projects/sre-platform-project.yaml
```

Validate:

```bash
kubectl get appproject -n argocd
```

Correct command:

```bash
kubectl get appproject -n argocd
```

Incorrect command:

```bash
kubectl get approject -n argocd
```

Expected:

```text
NAME                AGE
default             ...
fintech-workloads   ...
```

---

## 14. Important AppProject implication

The project has:

```yaml
clusterResourceWhitelist: []
```

This means:

```text
The workload app cannot manage cluster-scoped resources.
```

A Kubernetes Namespace is cluster-scoped.

So if the Bank of Anthos kustomization includes:

```yaml
kind: Namespace
metadata:
  name: fintech-workload
```

Argo CD may fail because the `fintech-workloads` AppProject is not allowed to manage Namespace resources.

### Enterprise solution

For enterprise-style GitOps:

```text
Workload apps should not create namespaces.
Platform/bootstrap should create namespaces.
```

So later, namespace management should move to:

```text
platform/namespaces/
```

or:

```text
bootstrap/namespaces/
```

For now, because the namespace already exists, the workload app can deploy namespaced resources.

---

## 15. Argo CD Application for Bank of Anthos

File:

```text
argocd/applications/bank-of-anthos-app.yaml
```

Manifest:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: bank-of-anthos
  namespace: argocd
spec:
  project: fintech-workloads

  source:
    repoURL: https://github.com/anselem-okeke/sre-decision-intelligence-gitops.git
    targetRevision: main
    path: workloads/bank-of-anthos/overlays/talos

  destination:
    server: https://kubernetes.default.svc
    namespace: fintech-workload

  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

Apply:

```bash
kubectl apply -f argocd/applications/bank-of-anthos-app.yaml
```

Validate:

```bash
kubectl get applications -n argocd
```

Expected eventually:

```text
bank-of-anthos   Synced   Healthy
```

Possible first state:

```text
bank-of-anthos   OutOfSync   Missing
```

---

## 16. Why manually applying the Argo CD Application is still needed

This command:

```bash
kubectl apply -k workloads/bank-of-anthos/overlays/talos
```

means:

```text
Kubernetes, deploy this app directly now.
```

It proves:

```text
The Kubernetes manifests are valid
The workload can run
The LoadBalancer works
```

But this command:

```bash
kubectl apply -f argocd/applications/bank-of-anthos-app.yaml
```

means:

```text
Argo CD, start managing this app from Git.
```

It creates an Argo CD `Application` custom resource.

Without that object, Argo CD does not know the app exists.

This is called bootstrapping.

Enterprise bootstrapping is usually done by:

```text
Terraform
Helm
A platform bootstrap pipeline
One-time kubectl apply of the root app
```

For the current phase, applying the Application once manually is acceptable.

---

## 17. App shows OutOfSync / Missing

Observed:

```bash
kubectl get applications -n argocd
```

Output:

```text
NAME             SYNC STATUS   HEALTH STATUS
bank-of-anthos   OutOfSync     Missing
```

Meaning:

```text
The Argo CD Application exists.
Argo CD has not successfully synced/adopted the workload yet.
```

Debug command:

```bash
kubectl describe application bank-of-anthos -n argocd
```

Look for:

```text
Conditions
Events
Message
Reason
```

Likely cause:

```text
The rendered workload includes a Namespace resource.
The AppProject has clusterResourceWhitelist: [].
Therefore Argo CD is not allowed to manage Namespace.
```

Enterprise fix:

```text
Remove namespace.yaml from the workload kustomization.
Manage namespaces separately in bootstrap/platform layer.
```

---

## 18. Enterprise GitOps model going forward

Use separate Argo CD projects by responsibility:

```text
bootstrap
  - manages Argo CD projects and root apps
  - can deploy to argocd namespace

platform
  - manages Cilium L2, LB pools, ingress/gateway, namespaces
  - may have controlled cluster-scoped permissions

fintech-workloads
  - manages Bank of Anthos and fintech application workloads
  - limited to fintech-workload namespace
  - no cluster-scoped permissions

observability
  - manages dashboards, alerts, monitoring config, logging config
```

Recommended future structure:

```text
argocd/
├── projects/
│   ├── bootstrap-project.yaml
│   ├── platform-project.yaml
│   ├── fintech-workloads-project.yaml
│   └── observability-project.yaml
│
└── applications/
    ├── bank-of-anthos-app.yaml
    ├── platform-cilium-app.yaml
    ├── observability-app.yaml
    └── decision-intelligence-api-app.yaml
```

---

## 19. Troubleshooting checklist

### Application not reachable through LoadBalancer

Check:

```bash
kubectl get svc frontend -n fintech-workload
curl -I http://<EXTERNAL-IP>
```

If failing, test app internally:

```bash
kubectl -n fintech-workload port-forward svc/frontend 8081:80
curl -I http://localhost:8081
```

If port-forward works, app is healthy.

Check NodePort:

```bash
kubectl get svc frontend -n fintech-workload
kubectl get nodes -o wide
curl -I http://<NODE-IP>:<NODEPORT>
```

If NodePort works but VIP fails, issue is LoadBalancer/L2.

### Cilium L2 checks

```bash
kubectl get ciliumloadbalancerippool -A
kubectl get ciliuml2announcementpolicy -A
kubectl get lease -n kube-system | grep -i cilium-lan
kubectl -n kube-system get cm cilium-lan-config -o yaml | grep -Ei "l2|loadbalancer|kube-proxy|external"
```

### Cilium L2 fix used

```bash
kubectl patch ciliuml2announcementpolicy lan-lb-services-policy \
  --type=json \
  -p='[
    {"op": "remove", "path": "/spec/interfaces"}
  ]'
```

### Check frontend label

```bash
kubectl kustomize workloads/bank-of-anthos/overlays/talos | grep -B10 -A20 "name: frontend"
```

Expected:

```yaml
lan-exposure: enabled
```

### Check Argo CD Application

```bash
kubectl get applications -n argocd
kubectl describe application bank-of-anthos -n argocd
```

### Check AppProject

```bash
kubectl get appproject -n argocd
kubectl describe appproject fintech-workloads -n argocd
```

---

## 20. Key mental model

```text
Kubernetes manifest test:
kubectl apply -k workloads/...
        ↓
Proves the app can run

Argo CD Application:
kubectl apply -f argocd/applications/...
        ↓
Registers the app for GitOps control

Git commit:
git push
        ↓
Updates the source of truth

Argo CD sync:
Git → Argo CD → Kubernetes
        ↓
Continuous reconciliation
```

---

## 21. Final current status

Confirmed:

```text
Bank of Anthos runs manually from Git/Kustomize.
Frontend LoadBalancer receives VIP 192.168.0.231.
Cilium L2 issue was fixed by removing interface restriction.
Frontend service now includes lan-exposure=enabled.
fintech-workloads AppProject was created.
bank-of-anthos Argo CD Application was created.
Argo CD currently shows OutOfSync/Missing and needs final sync troubleshooting.
```

Next recommended action:

```bash
kubectl describe application bank-of-anthos -n argocd
```

Then resolve the exact Argo CD condition, likely by separating namespace creation from the workload app.

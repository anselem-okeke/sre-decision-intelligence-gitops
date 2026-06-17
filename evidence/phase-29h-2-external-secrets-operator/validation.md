# Phase 29H.2 — External Secrets Operator Validation

## Objective

Install External Secrets Operator using Argo CD as a company-style platform-security component.

Vault is external and is not installed inside the Kubernetes cluster.

## Argo CD Application

```bash
argocd app get external-secrets
```

## Namespace

```bash
kubectl get ns external-secrets
```

Result:
```text
PENDING
```
## Pods
```bash
kubectl get pods -n external-secrets
```

Result:
```text
PENDING
```
## CRDs
```bash
kubectl get crd | grep external-secrets
```

Result:
```text
PENDING
```
## Rollout Status
```bash
kubectl -n external-secrets rollout status deployment/external-secrets
kubectl -n external-secrets rollout status deployment/external-secrets-webhook
kubectl -n external-secrets rollout status deployment/external-secrets-cert-controller
```

Result:
```text
PENDING
```
## ServiceAccount
```bash
kubectl -n external-secrets get serviceaccount
```

Result:
```text
PENDING
```
## Application Secrets
```bash
kubectl -n sre-decision-intelligence get secret
```

Result:
```text
PENDING
```
## Conclusion

### Phase Completion:

- External Secrets Operator is managed by Argo CD
- external-secrets namespace exists
- ESO controller is running
- ESO webhook is running
- ESO cert-controller is running
- ExternalSecret CRDs are installed
- external-secrets ServiceAccount exists
- No application secrets have been migrated yet
# External Secrets Operator

![vault-architecture](/img/decision2.png)

This folder contains the platform-owned configuration for External Secrets Operator.

ESO is installed into the `external-secrets` namespace and is used to sync secrets from external HashiCorp Vault into Kubernetes Secrets.

# Company-style model

- Vault is external.
- Git stores only references and controller configuration.
- Plaintext secrets are never committed to Git.
- ESO creates Kubernetes Secrets from Vault values.
- Application workloads consume normal Kubernetes Secrets.

# Step-by-step vault implemention

- ### [Understanding Enterprise Secret Management with Vault](https://github.com/anselem-okeke/sre-decision-intelligence-gitops/blob/main/platform/external-secrets/vault-eso-argocd-concepts-explained.md)

- ### [Enterprise Vault Implementation](https://github.com/anselem-okeke/sre-decision-intelligence-gitops/blob/main/platform/external-secrets/enterprise-vault-eso-argocd-runbook-README.md)
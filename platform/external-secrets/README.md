# External Secrets Operator

This folder contains the platform-owned configuration for External Secrets Operator.

ESO is installed into the `external-secrets` namespace and is used to sync secrets from external HashiCorp Vault into Kubernetes Secrets.

## Company-style model

- Vault is external.
- Git stores only references and controller configuration.
- Plaintext secrets are never committed to Git.
- ESO creates Kubernetes Secrets from Vault values.
- Application workloads consume normal Kubernetes Secrets.

## Current phase

Installs only the External Secrets Operator and CRDs.

Vault connectivity, ClusterSecretStore, and ExternalSecret resources are implemented in later phases.

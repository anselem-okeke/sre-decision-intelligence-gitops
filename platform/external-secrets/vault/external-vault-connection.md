# Phase 29H.3 — External Vault Connection Model

## Purpose

This phase defines how the Kubernetes cluster will connect to an external HashiCorp Vault instance.

Vault is **not installed inside the Kubernetes cluster**.

External Secrets Operator was installed in Phase 29H.2 and will later use this external Vault backend to generate native Kubernetes Secrets.

## Important current state

At the moment, there is no external Vault endpoint available yet.

Because of that, Phase 29H.3 is split into three parts:

```text
29H.3A — External Vault target design
29H.3B — External Vault provisioning decision
29H.3C — External Vault endpoint validation
```

This file documents the target model and the decision required before real connectivity can be validated.

## Company-style rule

```text
Git stores references only.
Vault stores real secret values.
External Secrets Operator reads Vault.
Kubernetes receives generated Secrets.
Application workloads consume native Kubernetes Secrets.
```

Git must not contain plaintext secret values.

## Target architecture

```text
Kubernetes Cluster
  external-secrets namespace
    External Secrets Operator
        ↓
External Vault endpoint
        ↓
Vault KV v2 secret engine
        ↓
Vault policy and role
        ↓
ClusterSecretStore
        ↓
ExternalSecret resources
        ↓
Generated Kubernetes Secrets
        ↓
Decision Intelligence API / PostgreSQL / Migration Job
```

## Why external Vault

External Vault is the company-style model because Vault is treated as shared security infrastructure, not as a workload dependency inside the same application cluster.

Benefits:

- Clear separation between workload cluster and secret backend
- Centralized secret governance
- Independent Vault lifecycle
- Better auditability
- Easier integration with enterprise identity and security controls
- Cleaner disaster recovery model
- No application cluster dependency on in-cluster Vault availability

## What is not done in this phase

This phase does **not** create:

- Vault policy
- Vault role
- Vault Kubernetes auth configuration
- ClusterSecretStore
- ExternalSecret resources
- Application Kubernetes Secrets
- Plaintext Kubernetes Secret manifests

Those come in later phases.

## Required future Vault endpoint

The final Vault endpoint should be defined as:

```text
VAULT_ADDR=https://REPLACE_WITH_EXTERNAL_VAULT_DNS_OR_IP
```

Examples:

```text
VAULT_ADDR=https://vault.company.internal
VAULT_ADDR=https://vault.platform.example.com
VAULT_ADDR=https://<external-vm-dns>:8200
```

## Required Vault model

Recommended target configuration:

| Item | Target value |
|---|---|
| Vault location | External to Kubernetes cluster |
| Vault protocol | HTTPS |
| TLS verification | Enabled |
| Secret engine | KV v2 |
| KV mount path | `secret` |
| Kubernetes auth mount | `kubernetes` |
| ESO namespace | `external-secrets` |
| ESO ServiceAccount | `external-secrets` |
| Vault policy model | Read-only for this platform path |

## Required future secret paths

Initial required Vault paths:

```text
secret/data/sre-decision-intelligence/postgres
secret/data/sre-decision-intelligence/api
```

Future paths:

```text
secret/data/sre-decision-intelligence/registry
secret/data/sre-decision-intelligence/observability
secret/data/sre-decision-intelligence/integrations
```

## Initial secrets to store later

### PostgreSQL path

```text
secret/data/sre-decision-intelligence/postgres
```

Expected keys:

```text
POSTGRES_DB
POSTGRES_USER
POSTGRES_PASSWORD
```

### API path

```text
secret/data/sre-decision-intelligence/api
```

Expected keys:

```text
DATABASE_URL
```

Important:

The password inside `DATABASE_URL` must match the PostgreSQL application user password.

## External Vault provisioning decision

Because no external Vault currently exists, one of these company-style options must be chosen.

### Option A — Dedicated external VM

Vault runs on a dedicated VM outside the Kubernetes cluster.

```text
Proxmox / cloud VM
    ↓
HashiCorp Vault
    ↓
HTTPS endpoint
    ↓
Kubernetes cluster connects to Vault
```

Good for a serious homelab/company-style simulation.

### Option B — Existing enterprise Vault

Vault is provided by a central platform/security team.

```text
Central Vault service
    ↓
Kubernetes auth enabled
    ↓
ESO connects through ClusterSecretStore
```

Best for real company environments.

### Option C — Managed Vault service

Vault or a Vault-compatible backend is provided by a managed platform.

Good when the company wants to reduce operational ownership.

## Recommended choice for this project

For this project, because there is no external Vault yet, use:

```text
Option A — Dedicated external VM
```

This keeps the implementation company-style while still being practical in a homelab.

Vault should still be treated as external infrastructure.

Do not install Vault into the Kubernetes cluster.

## Phase 29H.3A — External Vault target design

Status:

```text
Defined in this document.
```

Deliverable:

```text
platform/external-secrets/vault/external-vault-connection.md
```

## Phase 29H.3B — External Vault provisioning

Future deliverable:

```text
External Vault server available over HTTPS
```

Minimum requirements:

```text
Vault initialized
Vault unsealed
KV v2 enabled
Kubernetes can reach Vault endpoint
TLS model documented
Admin access available for auth/policy setup
```

## Phase 29H.3C — External Vault connectivity validation

Future validation command:

```bash
kubectl run vault-connectivity-test \
  -n external-secrets \
  --rm -it \
  --image=curlimages/curl \
  --restart=Never \
  -- curl -sS https://REPLACE_WITH_EXTERNAL_VAULT_URL/v1/sys/health
```

If using a temporary test certificate during early lab validation:

```bash
kubectl run vault-connectivity-test \
  -n external-secrets \
  --rm -it \
  --image=curlimages/curl \
  --restart=Never \
  -- curl -sS -k https://REPLACE_WITH_EXTERNAL_VAULT_URL/v1/sys/health
```

`-k` is only acceptable for temporary validation and must not become the final company-style model.

## Expected Vault health response

A healthy unsealed Vault should return fields similar to:

```json
{
  "initialized": true,
  "sealed": false,
  "standby": false,
  "version": "x.x.x"
}
```

Success indicators:

```text
initialized=true
sealed=false
Vault health endpoint reachable from Kubernetes
```

## TLS model

Final company-style recommendation:

```text
HTTPS with TLS verification enabled
```

Allowed models:

| TLS model | Company-style status |
|---|---|
| Public CA | Good |
| Private/internal CA | Good |
| Temporary insecure test | Only for early lab validation |
| Plain HTTP | Not recommended for final state |

If a private CA is used, the public CA certificate can be stored in Git as a ConfigMap because it is not secret material.

Never commit private keys.

## Next phases after this

```text
29H.4 — Configure Vault Kubernetes authentication
29H.5 — Create Vault policy, role, and initial secret paths
29H.6 — Create ClusterSecretStore
29H.7 — Create ExternalSecret resources
29H.8 — Validate generated Kubernetes Secrets
29H.9 — Remove plaintext Secret manifests from Git
29H.10 — Validate secret rotation
```

## Phase completion criteria

Phase 29H.3 is complete when:

- External Vault architecture is documented
- External Vault provisioning option is selected
- Vault endpoint is available
- Kubernetes can reach Vault `/v1/sys/health`
- Vault is initialized
- Vault is unsealed
- TLS model is documented
- No application secrets are migrated yet

## Current status

```text
Status: Partially complete

29H.3A External Vault target design: complete
29H.3B External Vault provisioning: pending
29H.3C Connectivity validation: pending
```

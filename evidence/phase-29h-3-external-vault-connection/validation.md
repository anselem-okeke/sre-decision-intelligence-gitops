# Phase 29H.3 — External Vault Connection Validation

## Objective

Validate that the Kubernetes cluster can reach an external HashiCorp Vault endpoint.

Vault is external and is not installed inside the Kubernetes cluster.

## Current status

There is currently no external Vault endpoint available.

Because of that, this validation is currently a readiness/evidence template.

```text
29H.3A — External Vault target design: complete
29H.3B — External Vault provisioning: pending
29H.3C — External Vault endpoint validation: pending
```

## External Vault decision

Selected target model:

```text
External Vault outside Kubernetes
```

Recommended implementation for this project:

```text
Dedicated external VM running HashiCorp Vault
```

Vault must not be installed as an in-cluster workload for the final company-style architecture.

## Expected Vault endpoint

```text
VAULT_ADDR=PENDING
```

Example future values:

```text
https://vault.company.internal
https://vault.platform.example.com
https://<external-vm-dns>:8200
```

## TLS model

Current value:

```text
PENDING
```

Final required model:

```text
HTTPS with TLS verification enabled
```

Allowed final options:

```text
public-ca
private-ca
```

Temporary early validation option:

```text
temporary-insecure-test-only
```

Temporary insecure validation must not be the final state.

## Vault health validation command

Run this after an external Vault endpoint exists:

```bash
kubectl run vault-connectivity-test \
  -n external-secrets \
  --rm -it \
  --image=curlimages/curl \
  --restart=Never \
  -- curl -sS https://REPLACE_WITH_EXTERNAL_VAULT_URL/v1/sys/health
```

Temporary lab-only command if TLS is not fully trusted yet:

```bash
kubectl run vault-connectivity-test \
  -n external-secrets \
  --rm -it \
  --image=curlimages/curl \
  --restart=Never \
  -- curl -sS -k https://REPLACE_WITH_EXTERNAL_VAULT_URL/v1/sys/health
```

## Expected result

Expected successful health response should show:

```text
initialized=true
sealed=false
Vault health endpoint reachable
```

Example structure:

```json
{
  "initialized": true,
  "sealed": false,
  "standby": false,
  "version": "x.x.x"
}
```

## Actual result

```text
PENDING — no external Vault endpoint exists yet.
```

## DNS validation

After Vault DNS exists, validate DNS resolution from inside Kubernetes:

```bash
kubectl run vault-dns-test \
  -n external-secrets \
  --rm -it \
  --image=busybox:1.36 \
  --restart=Never \
  -- nslookup REPLACE_WITH_VAULT_DNS
```

Result:

```text
PENDING
```

## Network validation

After Vault endpoint exists, validate TCP connectivity:

```bash
kubectl run vault-tcp-test \
  -n external-secrets \
  --rm -it \
  --image=busybox:1.36 \
  --restart=Never \
  -- nc -vz REPLACE_WITH_VAULT_HOST 8200
```

Result:

```text
PENDING
```

## ESO namespace validation

External Secrets Operator namespace should already exist from Phase 29H.2:

```bash
kubectl get ns external-secrets
```

Result:

```text
PENDING
```

## ESO pod validation

External Secrets Operator should already be running from Phase 29H.2:

```bash
kubectl get pods -n external-secrets
```

Result:

```text
PENDING
```

## Not performed in this phase

This phase does not create or validate:

- Vault Kubernetes auth role
- Vault policy
- Vault secret paths
- ClusterSecretStore
- ExternalSecret
- Generated Kubernetes Secrets
- Application secret migration

## Phase completion checklist

Phase 29H.3 is complete when:

- External Vault target model is documented
- External Vault provisioning option is selected
- External Vault endpoint exists
- Kubernetes can resolve the Vault DNS name
- Kubernetes can reach the Vault network endpoint
- Vault `/v1/sys/health` responds
- Vault is initialized
- Vault is unsealed
- TLS model is documented
- No application secrets are migrated yet

## Current conclusion

```text
Phase 29H.3 cannot be fully completed until an external Vault endpoint exists.

The design part is complete.
The provisioning and connectivity validation are pending.
```

## Next action

Proceed with:

```text
29H.3B — Provision or make available an external Vault endpoint
```

After Vault exists, repeat this validation and replace all `PENDING` sections with real command output.

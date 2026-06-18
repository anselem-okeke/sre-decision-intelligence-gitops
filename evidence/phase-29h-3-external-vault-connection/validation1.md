# Phase 29H.3 — External Vault Connection Validation

## Objective

Validate that the Kubernetes cluster can reach an external HashiCorp Vault endpoint.

Vault is external and is not installed inside the Kubernetes cluster.

## Phase status

```text
29H.3A — External Vault target design: complete
29H.3B — External Vault VM and Vault service setup: complete
29H.3C — External Vault endpoint validation: complete
```

## External Vault decision

Selected target model:

```text
External Vault outside Kubernetes
```

Implementation used for this phase:

```text
Dedicated external VM running HashiCorp Vault
```

Vault is not installed as an in-cluster workload.

## Vault endpoint

```text
VAULT_ADDR=http://192.168.0.61:8200
```

## Current TLS model

```text
Temporary HTTP for initial validation
```

## Final target TLS model

```text
HTTPS with TLS verification enabled
```

This final TLS target will be handled in the next coming hardening phases.

The current HTTP endpoint is accepted only for initial internal connectivity validation.

## Validation 1 — Vault VM to Vault health endpoint

Command executed from Vault VM:

```bash
curl -sS http://192.168.0.61:8200/v1/sys/health | jq
```

Result:

```json
{
  "initialized": true,
  "sealed": false,
  "standby": false,
  "performance_standby": false,
  "replication_performance_mode": "disabled",
  "replication_dr_mode": "disabled",
  "server_time_utc": 1781726385,
  "version": "2.0.2",
  "enterprise": false,
  "cluster_name": "vault-cluster-e1650c5e",
  "cluster_id": "8ec176e5-e767-bc9b-449c-4ef9459e9cf5",
  "echo_duration_ms": 0,
  "clock_skew_ms": 0,
  "replication_primary_canary_age_ms": 0,
  "removed_from_cluster": false
}
```

Validation result:

```text
Vault VM can reach Vault health endpoint.
Vault is initialized.
Vault is unsealed.
```

## Validation 2 — Jumpbox to Vault health endpoint

Command executed from jumpbox:

```bash
curl -sS http://192.168.0.61:8200/v1/sys/health | jq
```

Result:

```json
{
  "initialized": true,
  "sealed": false,
  "standby": false,
  "performance_standby": false,
  "replication_performance_mode": "disabled",
  "replication_dr_mode": "disabled",
  "server_time_utc": 1781726450,
  "version": "2.0.2",
  "enterprise": false,
  "cluster_name": "vault-cluster-e1650c5e",
  "cluster_id": "8ec176e5-e767-bc9b-449c-4ef9459e9cf5",
  "echo_duration_ms": 0,
  "clock_skew_ms": 0,
  "replication_primary_canary_age_ms": 0,
  "removed_from_cluster": false
}
```

Validation result:

```text
Jumpbox can reach external Vault.
Vault endpoint is reachable from the admin network.
Vault is initialized and unsealed.
```

## Validation 3 — Kubernetes to external Vault health endpoint

Command executed from jumpbox against Kubernetes:

```bash
kubectl run vault-connectivity-test \
  -n external-secrets \
  --rm -it \
  --image=curlimages/curl \
  --restart=Never \
  -- curl -sS http://192.168.0.61:8200/v1/sys/health
```

Result:

```text
Warning: would violate PodSecurity "restricted:latest": allowPrivilegeEscalation != false (container "vault-connectivity-test" must set securityContext.allowPrivilegeEscalation=false), unrestricted capabilities (container "vault-connectivity-test" must set securityContext.capabilities.drop=["ALL"]), runAsNonRoot != true (pod or container "vault-connectivity-test" must set securityContext.runAsNonRoot=true), seccompProfile (pod or container "vault-connectivity-test" must set securityContext.seccompProfile.type to "RuntimeDefault" or "Localhost")
{"initialized":true,"sealed":false,"standby":false,"performance_standby":false,"replication_performance_mode":"disabled","replication_dr_mode":"disabled","server_time_utc":1781727101,"version":"2.0.2","enterprise":false,"cluster_name":"vault-cluster-e1650c5e","cluster_id":"8ec176e5-e767-bc9b-449c-4ef9459e9cf5","echo_duration_ms":0,"clock_skew_ms":0,"replication_primary_canary_age_ms":0,"removed_from_cluster":false}
pod "vault-connectivity-test" deleted from external-secrets namespace
```

Validation result:

```text
Kubernetes can reach external Vault.
The external-secrets namespace can run a temporary connectivity test pod.
Vault health endpoint is reachable from inside the cluster.
Vault is initialized and unsealed.
```

## Pod Security warning analysis

The test pod produced a Pod Security warning:

```text
would violate PodSecurity "restricted:latest"
```

This warning happened because the temporary `kubectl run` pod did not explicitly define a restricted security context.

This did not block execution in the current namespace mode.

For future validation, use a security-context-compliant pod spec when needed.

This warning does not invalidate the Vault connectivity result.

## Not performed in this phase

This phase did not create or validate:

- Vault Kubernetes auth role
- Vault policy
- Vault secret paths
- ClusterSecretStore
- ExternalSecret
- Generated Kubernetes Secrets
- Application secret migration

Those are handled in later phases.

## Phase completion checklist

- [x] External Vault target model documented
- [x] External Vault VM available
- [x] Vault installed and running
- [x] Vault endpoint exists
- [x] Vault VM can reach Vault health endpoint
- [x] Jumpbox can reach Vault health endpoint
- [x] Kubernetes can reach Vault health endpoint
- [x] Vault `/v1/sys/health` responds
- [x] Vault is initialized
- [x] Vault is unsealed
- [x] TLS model documented
- [x] No application secrets migrated yet

## Conclusion

Phase 29H.3 is complete.

The Kubernetes cluster can reach the external HashiCorp Vault endpoint at:

```text
http://192.168.0.61:8200
```

Vault is initialized and unsealed.

The current endpoint is HTTP for initial validation only.

The final company-style target remains:

```text
HTTPS with TLS verification enabled
```

## Next phase

Proceed with:

```text
29H.4 — Configure Vault Kubernetes authentication
```

The next phase will make Vault trust the Kubernetes cluster and allow the `external-secrets` ServiceAccount to authenticate to Vault using the Kubernetes auth method.

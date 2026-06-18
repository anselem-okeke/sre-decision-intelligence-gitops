# Understanding Enterprise Secret Management with Vault, External Secrets Operator, Kubernetes, and Argo CD

## A Conceptual Guide Based on the SRE Decision Intelligence Platform

---

## About This Document

This document explains the **theory, reasoning, architecture, business value, security model, and engineering decisions** behind the Vault-based secret-management architecture implemented for the SRE Decision Intelligence platform.

This is not mainly a command runbook.  
It is written like a technical book so that an engineer, platform team, DevOps engineer, SRE, security engineer, or hiring manager can understand:

- What was built
- Why it was built
- What problem it solves
- Why each component exists
- How Vault, Kubernetes, ESO, and Argo CD work together
- Why secret management matters in enterprise platforms
- What each important manifest field means
- What security controls were introduced
- What tradeoffs were made
- What can be improved later

The implementation described here is based on a real platform flow:

```text
External HashiCorp Vault
        ↓
External Secrets Operator
        ↓
Kubernetes Secrets
        ↓
Application workloads
        ↓
Managed declaratively by Argo CD
```

---

# 1. Why Secret Management Matters

Modern platforms depend on secrets everywhere.

A typical cloud-native application needs:

- Database passwords
- API tokens
- Private keys
- TLS certificates
- OAuth/OIDC client secrets
- Registry credentials
- Webhook tokens
- Encryption keys
- Service-to-service credentials

These values are sensitive because they control access to systems, data, identities, and infrastructure.

In a traditional setup, teams often put secrets directly into:

- Kubernetes `Secret` manifests
- `.env` files
- CI/CD variables
- Helm values
- Application config files
- Manually created cluster objects
- Developer laptops
- Shared internal documentation

This creates a serious problem.

Kubernetes `Secret` objects are only base64-encoded by default. Base64 is not encryption. If a secret value is committed to Git, anyone with repo access, old Git history, CI logs, or backup access may be able to retrieve it.

The problem is not only technical. It is also operational and business-related.

If secrets are handled badly, the organization can suffer:

- Credential leakage
- Unauthorized database access
- Application compromise
- Breach escalation
- Failed audits
- Poor incident response
- Slow secret rotation
- Unclear ownership
- Manual recovery dependency
- Untrusted deployment pipelines

A mature platform should therefore treat secrets as **controlled runtime data**, not as ordinary configuration.

---

# 2. The Business Reason for Using Vault

The business value of Vault is not simply that it stores passwords.

The real business value is that Vault gives the organization a controlled security boundary for sensitive access.

Vault helps answer important enterprise questions:

| Business Question | Vault-Based Answer |
|---|---|
| Where are our secrets stored? | In a centralized secret-management system |
| Who can access them? | Identities mapped to Vault policies |
| What can each identity access? | Only paths allowed by policy |
| Can secrets be rotated? | Yes, centrally |
| Are secrets stored in Git? | No |
| Can applications retrieve secrets automatically? | Yes, through trusted workload identity |
| Can we audit access later? | Yes, with Vault audit logging |
| Can access be revoked without redeploying the app? | Yes |
| Can teams separate secret ownership from deployment ownership? | Yes |

In enterprise environments, secret management is usually required because of:

- Security policy
- Internal compliance
- ISO/SOC2-style control expectations
- Separation of duties
- Incident response requirements
- Least-privilege access control
- Cloud-native platform governance
- Production readiness expectations

This is why the platform was designed around:

```text
Git contains desired state.
Vault contains sensitive values.
Kubernetes receives only runtime Secrets.
Applications consume runtime Secrets.
```

---

# 3. Why Not Store Secrets Directly in Git?

Git is excellent for declarative infrastructure.

Git is not a good place for secret values.

Git keeps history. Even if a password is deleted later, it may remain in:

- Previous commits
- Branches
- Tags
- Forks
- CI/CD logs
- Local clones
- GitHub cache
- Backup systems

This means secret exposure in Git is usually treated as permanent exposure. The right response is rotation.

The better pattern is:

```text
Bad pattern:
Git → Kubernetes Secret with encoded password → Application

Better pattern:
Git → ExternalSecret reference → Vault → Kubernetes Secret → Application
```

With this design, Git still controls the deployment, but not the secret value.

That gives the platform the best of both worlds:

| Need | Tool |
|---|---|
| Declarative deployment | Git + Argo CD |
| Secret value storage | Vault |
| Kubernetes secret generation | External Secrets Operator |
| Application consumption | Kubernetes Secret |

---

# 4. Why Use an External Vault VM Instead of In-Cluster Vault?

Vault can run inside Kubernetes, but in this implementation Vault was deployed externally on a dedicated VM.

This is a strong enterprise-style decision for a homelab or production-like architecture.

## 4.1 Separation of Failure Domains

If Vault runs inside the same Kubernetes cluster it protects, then a serious cluster failure could affect both:

- The application workloads
- The secret backend required to recover them

By running Vault externally, the secret backend is separated from the workload cluster.

This makes the architecture closer to many enterprise setups, where critical security infrastructure is separated from application runtime infrastructure.

## 4.2 Independent Lifecycle

An external Vault VM can be:

- Patched independently
- Backed up independently
- Restarted independently
- Monitored independently
- Secured with host-level controls
- Recovered without relying on the Kubernetes control plane

This is important because Vault is infrastructure security, not just another application.

## 4.3 Security Boundary

An external Vault instance creates a clearer trust boundary:

```text
Kubernetes cluster = workload execution environment
Vault VM = secret authority
```

Kubernetes does not own the real secrets.  
Kubernetes only receives generated runtime copies through ESO.

---

# 5. Why External Secrets Operator Exists

Kubernetes applications normally expect secrets to be available as Kubernetes `Secret` objects.

For example, a Deployment can reference:

```yaml
envFrom:
  - secretRef:
      name: sre-decision-api-secret
```

Applications usually do not want to know how Vault works. They only need a Kubernetes Secret.

External Secrets Operator bridges this gap.

It watches `ExternalSecret` resources and performs this flow:

```text
ExternalSecret says:
  "Read DATABASE_URL from Vault path X"

ESO does:
  Authenticate to Vault
  Read the secret value
  Create or update Kubernetes Secret
```

ESO lets platform teams avoid writing custom secret-sync scripts.

It gives a clean Kubernetes-native interface:

- Platform team manages `ClusterSecretStore`
- App team manages `ExternalSecret`
- Vault stores real values
- Kubernetes receives generated Secrets

---

# 6. Why Argo CD Manages the Kubernetes Side

Argo CD is responsible for reconciling Kubernetes desired state from Git.

In this architecture, Argo CD manages:

- ESO installation
- ESO configuration references
- ServiceAccounts
- RBAC
- CA ConfigMap
- ClusterSecretStore
- ExternalSecret resources

But Argo CD does **not** manage real secret values.

That split is important.

```text
Argo CD manages:
  How Kubernetes should connect to Vault

Vault manages:
  The actual secret values
```

This gives a clean operational model:

| Layer | Managed By | Contains Secrets? |
|---|---|---|
| Git manifests | Argo CD | No |
| Vault KV paths | Vault admin / automation | Yes |
| Kubernetes generated Secret | ESO | Runtime copy |
| Application | Kubernetes | Consumes Secret |

---

# 7. The Core Architecture in One Picture

```text
                           ┌─────────────────────┐
                           │      GitHub Repo      │
                           │  Declarative YAML     │
                           └──────────┬──────────┘
                                      │
                                      │ watches
                                      ▼
                           ┌─────────────────────┐
                           │       Argo CD        │
                           │ GitOps Reconciler    │
                           └──────────┬──────────┘
                                      │
                    ┌─────────────────┼─────────────────┐
                    │                 │                 │
                    ▼                 ▼                 ▼
          ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
          │ ESO Controller  │ │ ClusterSecret   │ │ ExternalSecret  │
          │ Installation    │ │ Store Reference │ │ References      │
          └────────┬────────┘ └────────┬────────┘ └────────┬────────┘
                   │                   │                   │
                   └───────────────────┼───────────────────┘
                                       │
                                       ▼
                          ┌──────────────────────┐
                          │ External Secrets     │
                          │ Operator             │
                          └──────────┬───────────┘
                                     │
                                     │ Kubernetes auth + TLS
                                     ▼
                          ┌──────────────────────┐
                          │ External Vault VM     │
                          │ KV v2 + Policies      │
                          └──────────┬───────────┘
                                     │
                                     ▼
                          ┌──────────────────────┐
                          │ Kubernetes Secrets    │
                          │ Generated at Runtime  │
                          └──────────────────────┘
```

---

# 8. Vault as the Secret Authority

Vault is the central authority for secret values.

In this implementation, Vault stores secrets under KV v2:

```text
secret/sre-decision-intelligence/postgres
secret/sre-decision-intelligence/api
```

The `secret/` prefix is the KV engine mount.

The application-specific part is:

```text
sre-decision-intelligence/postgres
sre-decision-intelligence/api
```

This separation matters because it allows Vault policy to be scoped tightly.

For example, the ESO role only needs read access to:

```text
secret/data/sre-decision-intelligence/*
```

It does not need access to:

```text
secret/data/bank-of-anthos/*
secret/data/platform/*
secret/data/admin/*
```

This is the principle of least privilege.

---

# 9. Understanding Vault KV v2

Vault KV v2 is a versioned key-value secret engine.

Unlike KV v1, KV v2 stores metadata and data separately.

The human-friendly path is:

```text
secret/sre-decision-intelligence/postgres
```

The API path for data is:

```text
secret/data/sre-decision-intelligence/postgres
```

The API path for metadata is:

```text
secret/metadata/sre-decision-intelligence/postgres
```

That is why the policy used:

```hcl
path "secret/data/sre-decision-intelligence/*" {
  capabilities = ["read"]
}

path "secret/metadata/sre-decision-intelligence/*" {
  capabilities = ["read", "list"]
}
```

This is one of the most common sources of confusion with Vault.

When writing ExternalSecret YAML, we do **not** write the full API path.

Because the `ClusterSecretStore` already says:

```yaml
path: "secret"
version: "v2"
```

The ExternalSecret only needs:

```yaml
remoteRef:
  key: sre-decision-intelligence/postgres
```

This means:

```text
ESO combines:
  path: secret
  version: v2
  key: sre-decision-intelligence/postgres

Vault API reads:
  secret/data/sre-decision-intelligence/postgres
```

---

# 10. Why TLS Was Added to Vault

Vault handles secret values. Communication with Vault must be encrypted.

Without TLS, a secret request could travel as plaintext:

```text
ESO → HTTP → Vault
```

That is not acceptable for a production-style platform.

TLS gives three protections:

1. **Confidentiality**  
   Secret values are encrypted in transit.

2. **Integrity**  
   Traffic cannot be silently modified without detection.

3. **Server identity**  
   ESO can verify it is talking to the real Vault server.

The final model is:

```text
ESO → HTTPS → Vault
```

---

# 11. Why an Internal CA Was Created

A certificate must be trusted by the client.

Since this is a homelab/private platform, a public internet certificate authority is not required. Instead, an internal CA was created.

The CA signs the Vault server certificate.

```text
Internal CA
    ↓ signs
Vault server certificate
    ↓ trusted by
Jumpbox and Kubernetes ESO
```

The important distinction is:

| File | Meaning | Safe in Git? |
|---|---|---|
| `vault-ca.crt` | Public CA certificate | Yes |
| `vault-ca.key` | CA private key | No |
| `vault.crt` | Vault server certificate | Usually okay |
| `vault.key` | Vault server private key | No |

The public CA certificate was placed into Kubernetes as a ConfigMap:

```yaml
kind: ConfigMap
metadata:
  name: vault-ca-cert
data:
  vault-ca.crt: |
    -----BEGIN CERTIFICATE-----
```

This lets ESO verify Vault's server certificate.

---

# 12. Why the Vault Certificate Needed DNS and IP SANs

TLS validation checks whether the hostname or IP used by the client matches the certificate.

The Vault certificate was created with:

```text
DNS:vault.platform.local
IP Address:192.168.0.61
```

This matters because the test pod used:

```text
https://vault.platform.local:8200
```

but the `ClusterSecretStore` used:

```text
https://192.168.0.61:8200
```

The IP-based URL works only because the certificate includes:

```text
IP Address:192.168.0.61
```

If the certificate only had:

```text
DNS:vault.platform.local
```

then this would fail:

```text
https://192.168.0.61:8200
```

with an error such as:

```text
x509: cannot validate certificate for 192.168.0.61 because it doesn't contain any IP SANs
```

This is why SANs are not optional in modern TLS.

---

# 13. Why the Jumpbox Was Configured

The jumpbox is the operational control point.

It has access to:

- `kubectl`
- `argocd`
- `vault`
- Git repository
- Kubernetes context
- Vault CA certificate

The jumpbox allows the platform engineer to configure both sides of the system:

```text
Kubernetes side:
  kubectl
  argocd
  GitOps manifests

Vault side:
  vault CLI
  Vault policy
  Vault auth roles
  Vault KV secrets
```

This is common in enterprise operations, where engineers use a hardened admin host rather than random laptops.

---

# 14. Why `~/.vault-tls` Was Used Instead of `~/.vault`

During the implementation, the Vault CLI failed with:

```text
failed to get token helper: read /home/jumpbox/.vault: is a directory
```

This happened because `~/.vault` was used as a directory.

The Vault CLI may expect `~/.vault` behavior related to local token/helper handling. To avoid conflict, TLS materials were moved to:

```text
~/.vault-tls/vault-ca.crt
```

This gives a clean separation:

```text
~/.vault-tls       = local TLS trust material
Vault CLI runtime = free to use its own expected paths
```

The important operational lesson:

Do not place arbitrary directory trees in paths that command-line tools may reserve for their own behavior.

---

# 15. Why Kubernetes Authentication Was Used

Vault supports several authentication methods:

- Token auth
- AppRole
- Kubernetes auth
- OIDC
- LDAP
- Cloud IAM
- TLS cert auth

For Kubernetes workloads, Kubernetes auth is usually the natural choice.

Why?

Because Kubernetes already has a workload identity system: ServiceAccounts.

A pod can prove its identity using a ServiceAccount token.

Vault can validate that token by calling the Kubernetes TokenReview API.

The flow is:

```text
ESO ServiceAccount token
        ↓
Vault Kubernetes auth endpoint
        ↓
Vault asks Kubernetes TokenReview API:
  "Is this token valid?"
        ↓
Kubernetes replies:
  "Yes, it belongs to ServiceAccount external-secrets in namespace external-secrets"
        ↓
Vault checks role binding
        ↓
Vault issues Vault token with limited policy
```

This avoids putting static Vault tokens into Kubernetes.

That is the main security benefit.

---

# 16. Why a Token Reviewer ServiceAccount Was Needed

Vault needs permission to ask Kubernetes whether a token is valid.

That is done through the TokenReview API.

A dedicated ServiceAccount was created:

```text
vault-token-reviewer
```

It was bound to:

```text
system:auth-delegator
```

This allows Vault to perform TokenReview.

The ServiceAccount is not the same as the ESO ServiceAccount.

There are two identities:

| Identity | Used By | Purpose |
|---|---|---|
| `vault-token-reviewer` | Vault backend config | Lets Vault validate Kubernetes tokens |
| `external-secrets` | ESO controller | Lets ESO authenticate to Vault |

This separation is important.

The reviewer identity helps Vault validate tokens.  
The ESO identity receives Vault access after validation.

---

# 17. Understanding the TokenReview RBAC Manifest

The ClusterRoleBinding looked like this:

```yaml
roleRef:
  kind: ClusterRole
  name: system:auth-delegator
subjects:
  - kind: ServiceAccount
    name: vault-token-reviewer
    namespace: external-secrets
```

The most important field is:

```yaml
roleRef.name: system:auth-delegator
```

This built-in Kubernetes role allows the subject to create TokenReview requests.

The subject is:

```yaml
name: vault-token-reviewer
namespace: external-secrets
```

That means only this ServiceAccount receives this permission.

This follows least privilege. We did not give broad cluster-admin access.

---

# 18. Why a ServiceAccount Token Secret Was Created

In newer Kubernetes versions, ServiceAccount tokens are not always automatically created as long-lived Secrets.

Vault's Kubernetes auth backend needs a token reviewer JWT.

So a Secret of type:

```yaml
type: kubernetes.io/service-account-token
```

was created.

The manifest itself does not contain the token value. Kubernetes fills it at runtime.

This is why it is acceptable to store this manifest in Git:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: vault-token-reviewer-token
  annotations:
    kubernetes.io/service-account.name: vault-token-reviewer
type: kubernetes.io/service-account-token
```

What is not acceptable is committing the generated live token value.

This is a subtle but important difference:

| Thing | Safe in Git? |
|---|---|
| Empty ServiceAccount token Secret manifest | Acceptable |
| Actual generated JWT token value | No |

---

# 19. Vault Kubernetes Auth Backend Configuration

Vault needed this information:

```text
token_reviewer_jwt
kubernetes_host
kubernetes_ca_cert
disable_iss_validation
```

Each field has a purpose.

## 19.1 `token_reviewer_jwt`

This is the JWT belonging to `vault-token-reviewer`.

Vault uses it when talking to Kubernetes TokenReview API.

It answers the question:

```text
Who is Vault when it asks Kubernetes to validate another token?
```

This token must not be committed to Git.

---

## 19.2 `kubernetes_host`

This is the Kubernetes API server address.

It must be reachable from the Vault VM.

A common mistake is using:

```text
https://127.0.0.1:6443
```

That would only work from the machine where the API is local.  
Vault is on another VM, so it needs a real reachable endpoint.

---

## 19.3 `kubernetes_ca_cert`

This lets Vault trust the Kubernetes API server certificate.

Vault must verify Kubernetes just like ESO must verify Vault.

This creates mutual trust boundaries:

```text
ESO verifies Vault using Vault CA
Vault verifies Kubernetes using Kubernetes CA
```

---

## 19.4 `disable_iss_validation=true`

Kubernetes ServiceAccount issuer behavior changed across versions. In many Kubernetes setups, issuer validation can cause auth failures unless configured carefully.

Setting:

```text
disable_iss_validation=true
```

is a practical compatibility choice for this setup.

In a stricter production environment, teams may explicitly configure issuer validation instead.

---

# 20. Why Vault Policy Was Created

Vault policy controls what a token can do.

The policy used:

```hcl
path "secret/data/sre-decision-intelligence/*" {
  capabilities = ["read"]
}

path "secret/metadata/sre-decision-intelligence/*" {
  capabilities = ["read", "list"]
}
```

This gives ESO access only to the application paths it needs.

It does not allow:

```text
create
update
delete
sudo
access to other paths
```

This is least privilege.

A bad policy would be:

```hcl
path "secret/*" {
  capabilities = ["read", "list"]
}
```

That would allow too much.

The correct policy scopes access to:

```text
sre-decision-intelligence/*
```

This reduces blast radius.

If ESO or the application namespace is compromised, the attacker should not automatically gain access to all Vault secrets.

---

# 21. Why a Vault Role Was Created

The Vault role maps Kubernetes identity to Vault policy.

The role says:

```text
If a Kubernetes token belongs to:
  ServiceAccount: external-secrets
  Namespace: external-secrets

Then issue a Vault token with:
  Policy: sre-decision-intelligence-read
  TTL: 1h
```

The important fields:

```text
bound_service_account_names=external-secrets
bound_service_account_namespaces=external-secrets
policies=sre-decision-intelligence-read
ttl=1h
```

This is identity-based access.

ESO does not use a static Vault token.  
ESO proves its Kubernetes identity, then Vault grants short-lived access.

---

# 22. Why the Audience Warning Appeared

Vault warned:

```text
Role sre-decision-intelligence-eso does not have an audience configured.
```

At first, audience verification was added:

```text
audience=vault
```

This is a stronger JWT validation mechanism.

But the installed ESO CRD did not expose an `audiences` field under the Kubernetes auth config.

As a result, ESO sent a token with its default audience, while Vault expected:

```text
vault
```

Vault rejected the login:

```text
invalid audience (aud) claim
```

The role had to be deleted and recreated without the audience requirement.

The final role is still secure because it is constrained by:

- ServiceAccount name
- Namespace
- Policy
- TTL
- TLS
- Kubernetes TokenReview

The lesson:

Security hardening must match the capabilities of the installed controller version.

A control is only useful if both sides support it.

---

# 23. Why ClusterSecretStore Was Used

ESO supports two store types:

- `SecretStore`
- `ClusterSecretStore`

A `SecretStore` is namespace-scoped.

A `ClusterSecretStore` is cluster-scoped.

A `ClusterSecretStore` was used because the Vault backend is a platform-level integration.

It represents shared platform infrastructure:

```text
vault-backend
```

Applications can reference it from their namespaces.

This avoids duplicating Vault connection configuration in every application namespace.

The platform team owns the store.  
Application teams own their ExternalSecrets.

This is a clean enterprise platform pattern.

---

# 24. Understanding the ClusterSecretStore Manifest

The key manifest looked like this:

```yaml
apiVersion: external-secrets.io/v1
kind: ClusterSecretStore
metadata:
  name: vault-backend
spec:
  provider:
    vault:
      server: "https://192.168.0.61:8200"
      path: "secret"
      version: "v2"
      caProvider:
        type: ConfigMap
        name: vault-ca-cert
        key: vault-ca.crt
        namespace: external-secrets
      auth:
        kubernetes:
          mountPath: "kubernetes"
          role: "sre-decision-intelligence-eso"
          serviceAccountRef:
            name: external-secrets
            namespace: external-secrets
```

Each field matters.

## 24.1 `kind: ClusterSecretStore`

This means the store is cluster-scoped.

It can be referenced by ExternalSecrets in different namespaces.

---

## 24.2 `metadata.name: vault-backend`

This is the name ExternalSecrets reference:

```yaml
secretStoreRef:
  name: vault-backend
  kind: ClusterSecretStore
```

If this name changes, all ExternalSecrets using it must change.

---

## 24.3 `server`

```yaml
server: "https://192.168.0.61:8200"
```

This tells ESO where Vault is.

It uses HTTPS, not HTTP.

The IP was used because ESO pods did not have the temporary `hostAliases` used by the test pod.

A future improvement is to provide cluster-wide DNS and use:

```text
https://vault.platform.local:8200
```

---

## 24.4 `path: secret`

This identifies the Vault KV engine mount.

It does not mean every secret starts with `secret/data` in ExternalSecret YAML.

It means ESO knows the KV mount is called:

```text
secret
```

---

## 24.5 `version: v2`

This tells ESO the Vault engine is KV v2.

Without this, ESO may read the wrong API path.

This field is extremely important.

---

## 24.6 `caProvider`

```yaml
caProvider:
  type: ConfigMap
  name: vault-ca-cert
  key: vault-ca.crt
  namespace: external-secrets
```

This tells ESO where to find the CA certificate used to verify Vault TLS.

Important fields:

| Field | Meaning |
|---|---|
| `type: ConfigMap` | CA comes from a ConfigMap |
| `name: vault-ca-cert` | Name of the ConfigMap |
| `key: vault-ca.crt` | Key containing the certificate |
| `namespace: external-secrets` | Namespace where the ConfigMap exists |

This avoids disabling TLS verification.

Disabling TLS verification would be bad security.

---

## 24.7 `auth.kubernetes.mountPath`

```yaml
mountPath: "kubernetes"
```

This must match the Vault auth path:

```text
auth/kubernetes/
```

If Vault auth was enabled at a different path, this field must match it.

---

## 24.8 `auth.kubernetes.role`

```yaml
role: "sre-decision-intelligence-eso"
```

This is the Vault role ESO logs into.

If the role name is wrong, ESO cannot authenticate.

---

## 24.9 `serviceAccountRef`

```yaml
serviceAccountRef:
  name: external-secrets
  namespace: external-secrets
```

This tells ESO which Kubernetes ServiceAccount identity to use for Vault login.

It must match the Vault role binding:

```text
bound_service_account_names=external-secrets
bound_service_account_namespaces=external-secrets
```

If these do not match, Vault rejects the login.

---

# 25. Why ExternalSecret Resources Were Created

The `ClusterSecretStore` tells ESO how to connect to Vault.

The `ExternalSecret` tells ESO what to read and what Kubernetes Secret to create.

So the responsibilities are:

```text
ClusterSecretStore:
  Where is Vault?
  How do we authenticate?
  Which CA verifies TLS?
  Which KV engine?

ExternalSecret:
  Which Vault path?
  Which properties?
  Which Kubernetes Secret name?
  Which namespace?
```

This separation is clean and scalable.

The platform team can provide the store.  
The application team can provide ExternalSecret definitions.

---

# 26. Understanding the PostgreSQL ExternalSecret

The PostgreSQL ExternalSecret creates:

```text
sre-decision-postgres-secret
```

with:

```text
POSTGRES_DB
POSTGRES_USER
POSTGRES_PASSWORD
```

Key snippet:

```yaml
secretStoreRef:
  name: vault-backend
  kind: ClusterSecretStore
target:
  name: sre-decision-postgres-secret
  creationPolicy: Owner
data:
  - secretKey: POSTGRES_PASSWORD
    remoteRef:
      key: sre-decision-intelligence/postgres
      property: POSTGRES_PASSWORD
```

## 26.1 `secretStoreRef`

This connects the ExternalSecret to the Vault backend.

If this is wrong, ESO will not know where to retrieve values.

---

## 26.2 `target.name`

This is the Kubernetes Secret that ESO creates.

Application manifests should reference this Secret.

---

## 26.3 `creationPolicy: Owner`

This means ESO owns the generated Secret.

If the ExternalSecret is removed, ESO can clean up or manage the lifecycle depending on deletion policy behavior.

This is important because ownership becomes visible through:

```text
ownerReferences
reconcile.external-secrets.io/managed: "true"
```

---

## 26.4 `secretKey`

This is the key name inside the Kubernetes Secret.

For example:

```yaml
secretKey: POSTGRES_PASSWORD
```

creates:

```text
Kubernetes Secret data key: POSTGRES_PASSWORD
```

---

## 26.5 `remoteRef.key`

This is the Vault secret path below the KV mount.

For this implementation:

```yaml
remoteRef:
  key: sre-decision-intelligence/postgres
```

This maps to Vault KV path:

```text
secret/sre-decision-intelligence/postgres
```

---

## 26.6 `remoteRef.property`

This is the field inside the Vault secret object.

Vault secret:

```text
POSTGRES_DB
POSTGRES_USER
POSTGRES_PASSWORD
```

ExternalSecret property:

```yaml
property: POSTGRES_PASSWORD
```

This lets one Vault object produce multiple Kubernetes Secret keys.

---

# 27. Understanding the API ExternalSecret

The API ExternalSecret creates:

```text
sre-decision-api-secret
```

with:

```text
DATABASE_URL
```

The important point is that `DATABASE_URL` often contains the password.

Therefore, it must not appear in Git.

It belongs in Vault.

The ExternalSecret only contains this reference:

```yaml
remoteRef:
  key: sre-decision-intelligence/api
  property: DATABASE_URL
```

The actual value lives only in Vault and the generated runtime Kubernetes Secret.

---

# 28. Why Validation Was Done Without Printing Secrets

During validation, the goal was to prove the system works without exposing sensitive values.

Safe validation includes:

```text
Check Secret exists
Check expected keys exist
Check ExternalSecret is Ready=True
Check Argo CD app is Healthy
Check ownership metadata
Check password length
Mask DATABASE_URL
```

Unsafe validation includes:

```text
Printing POSTGRES_PASSWORD
Printing full DATABASE_URL
Pasting Vault tokens
Saving secret values in notes
```

One masking command failed because the URL format was:

```text
postgresql+psycopg://...
```

instead of only:

```text
postgresql://...
```

That is why a stronger masking command was needed:

```bash
sed -E 's#(postgresql(\+psycopg)?://[^:]+:)[^@]+(@.*)#\1****\3#'
```

The lesson:

Validation commands must handle real application URL formats, not only simplified examples.

---

# 29. Why Secret Rotation Was Required

A password was printed during validation.

Even if it was not committed to Git, once a secret is displayed in a terminal or chat, the safe assumption is that it may be exposed.

The correct security response is:

```text
Rotate it.
```

Rotation means:

1. Put a new password in Vault
2. Update related secrets such as `DATABASE_URL`
3. Force ESO refresh
4. Confirm Kubernetes Secret updated
5. Restart/reload application if needed

This is how an enterprise team should behave.

The goal is not to blame mistakes.  
The goal is to make secret exposure recoverable.

---

# 30. Why Plaintext Secret Manifests Were Removed from Git

After ESO was working, old raw Secret manifests became a liability.

The platform should not have two sources of truth:

```text
Bad:
Git Secret YAML and Vault both define the secret

Good:
Vault defines value
ExternalSecret defines reference
ESO generates Kubernetes Secret
```

The cleanup validated that only these files referenced the SRE Decision Intelligence secret names:

```text
postgres-externalsecret.yaml
api-externalsecret.yaml
```

That is correct.

Remaining Bank of Anthos demo secrets were identified separately. They are not part of the SRE Decision Intelligence migration, but they should be migrated later.

This is how professional cleanup should be done:

```text
Do not delete blindly.
Classify what belongs to the current scope.
Document what remains.
Plan a later hardening phase.
```

---

# 31. Why Argo CD AppProjects Matter

Argo CD AppProjects are security boundaries inside Argo CD.

They control:

- Which Git repos an app can use
- Which clusters/namespaces it can deploy to
- Which cluster-scoped resources it can create
- Which namespaced resources it can create

This matters because ESO needs powerful resources:

```text
CRDs
Webhook configurations
ClusterRoleBinding
ClusterSecretStore
```

Without proper AppProject permissions, Argo CD may refuse to sync or stay in an unknown state.

The `platform-security` project was created because secret management is platform security infrastructure, not a normal workload.

This is a good enterprise pattern:

```text
Separate project for security platform components
Separate project for workloads
Separate permissions per project
```

---

# 32. Why Some Vault Configuration Was Not Managed by Argo CD

Argo CD manages Kubernetes resources.

Vault policies, Vault auth methods, and Vault roles live inside Vault.

They are not Kubernetes objects.

Therefore, they were not managed by Argo CD directly.

The split is:

```text
Argo CD:
  Kubernetes manifests

Vault CLI:
  Vault internal configuration
```

In a more mature setup, Vault-side configuration should eventually be managed by:

```text
Terraform Vault provider
```

That would make Vault policies and roles declarative too.

But for this implementation, a controlled CLI bootstrap was acceptable.

The important rule:

Do not try to force Argo CD to manage non-Kubernetes state unless you introduce a proper controller or Terraform workflow.

---

# 33. Business and Industry Relevance

This architecture is relevant in real companies because it addresses a common platform maturity problem:

```text
How do we let teams deploy applications through GitOps without putting secrets in Git?
```

In many organizations, GitOps adoption creates a secret-management challenge.

Teams want:

- Automated deployments
- Version-controlled manifests
- Repeatable environments
- Fast recovery
- Clear audit trail

But they also need:

- Secret confidentiality
- Rotation
- Least privilege
- Separation of duties
- No credentials in Git

Vault + ESO + Argo CD solves this by separating responsibilities.

| Concern | Solved By |
|---|---|
| Deployment automation | Argo CD |
| Runtime secret injection | ESO |
| Secret storage | Vault |
| Identity verification | Kubernetes auth |
| Access control | Vault policy |
| TLS trust | Internal CA |
| Git safety | ExternalSecret references |

This is exactly the kind of design used in platform engineering, SRE, DevSecOps, regulated environments, and production Kubernetes platforms.

---

# 34. Security Model Summary

The final security model has several layers.

## 34.1 Transport Security

ESO connects to Vault using HTTPS.

Vault certificate is verified using the internal CA.

```text
No plaintext Vault traffic
No disabled TLS verification
```

## 34.2 Workload Identity

ESO authenticates using Kubernetes ServiceAccount identity.

```text
No static Vault token stored in Kubernetes manifests
```

## 34.3 Vault Policy

ESO receives only read access to the required app path.

```text
secret/data/sre-decision-intelligence/*
```

## 34.4 Namespace Binding

Vault role is bound to:

```text
ServiceAccount: external-secrets
Namespace: external-secrets
```

## 34.5 Git Safety

Git contains only references.

```text
No SRE Decision Intelligence plaintext secret manifests
```

## 34.6 Runtime Ownership

Generated Kubernetes Secrets are marked as ESO-managed.

```text
reconcile.external-secrets.io/managed: "true"
ownerReferences: ExternalSecret
```

---

# 35. Important Tradeoffs

No architecture is perfect. This implementation made practical choices.

## 35.1 Manual Unseal

Vault was manually unsealed.

This is acceptable for a homelab or learning environment, but production usually uses auto-unseal with:

- Cloud KMS
- HSM
- Vault Transit
- Azure Key Vault
- AWS KMS
- GCP KMS

Manual unseal is simpler but operationally heavier.

---

## 35.2 IP-Based Vault URL

The `ClusterSecretStore` used:

```text
https://192.168.0.61:8200
```

This is acceptable because the certificate included the IP SAN.

A more polished setup would use:

```text
https://vault.platform.local:8200
```

with cluster-wide DNS.

---

## 35.3 Vault CLI Bootstrap

Vault roles and policies were configured manually with Vault CLI.

This is fine for controlled bootstrap, but Terraform would be better for repeatability.

---

## 35.4 No Audience Claim

The Vault role does not currently enforce JWT audience because the installed ESO CRD did not expose audience configuration.

This is a version compatibility tradeoff.

The role still uses strong controls:

- ServiceAccount binding
- Namespace binding
- TokenReview
- Policy
- TTL
- TLS

---

# 36. What Makes This Enterprise-Style

This setup is enterprise-style because it follows several mature principles:

```text
External secret authority
GitOps-managed references
No plaintext secrets in Git
TLS everywhere
Least privilege Vault policy
Kubernetes workload identity
Dedicated Argo CD security project
Clear validation steps
Secret rotation path
Documented remaining risk
```

It is not just about installing Vault.

It is about building a controlled secret-management workflow.

---

# 37. Mental Model to Remember

The easiest way to understand the whole system is this:

```text
Vault knows the secret.
ESO knows how to fetch it.
Argo CD knows what should exist.
Kubernetes knows how to present it to the app.
The app only knows the Kubernetes Secret.
```

Or even simpler:

```text
Git declares.
Vault protects.
ESO synchronizes.
Kubernetes serves.
Application consumes.
```

---

# 38. Final Conceptual Summary

Before this architecture, secrets could easily drift into Git or manual cluster state.

After this architecture:

- Vault became the secret source of truth
- Argo CD became the desired-state engine
- ESO became the sync controller
- Kubernetes Secrets became generated runtime objects
- Applications remained simple
- Git remained clean
- Secret rotation became possible
- Security ownership became clearer

The final outcome is not just a technical setup.

It is a platform capability.

It enables teams to deploy applications safely without embedding secret values in repositories, Helm files, CI/CD pipelines, or manual cluster steps.

That is the real value.

---

# 39. Key Takeaways

1. **Secrets are runtime data, not Git configuration.**
2. **Vault should be treated as a security authority.**
3. **ESO bridges Vault and Kubernetes cleanly.**
4. **Argo CD should manage references, not secret values.**
5. **TLS verification must be enabled.**
6. **Vault policies should be narrow and path-scoped.**
7. **Kubernetes auth avoids static Vault tokens.**
8. **Validation should avoid printing secret values.**
9. **Secret exposure should trigger rotation.**
10. **A good platform separates deployment automation from secret ownership.**

---

## End

This document explained the concepts, design decisions, field meanings, security reasoning, and business value behind the Vault + ESO + Argo CD implementation for the SRE Decision Intelligence platform.

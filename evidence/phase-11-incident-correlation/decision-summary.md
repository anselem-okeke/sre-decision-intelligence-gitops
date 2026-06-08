# Decision Summary — Frontend Availability Breach

## Incident

Bank of Anthos frontend availability breach.

## Impact

The frontend endpoint was unavailable from the probe perspective.

Users attempting to access the frontend service path would be affected.

## Primary Signal

```promql
probe_success{job="bank-of-anthos-frontend"} = 0
```

## Evidence

| Category | Evidence |
|---|---|
| User-facing impact | Blackbox probe failed |
| SLO breach | Availability dropped below 99% |
| Alert state | `BankOfAnthosFrontendAvailabilitySLOBreach` entered pending state |
| Kubernetes routing | Frontend Service endpoints became empty |
| Pod health | Frontend pod stayed `1/1 Running` |
| Recovery | Removing the broken selector restored endpoints and probe success |

## Likely Root Cause

The frontend Service selector no longer matched the frontend pod labels.

This caused the Service to lose all endpoints.

## Safe Action

Restore the frontend Service selector so that it matches the frontend pod labels.

In this incident, recovery was completed by removing the temporary `slo-test=broken` selector:

```bash
kubectl patch svc frontend -n fintech-workload \
  --type='json' \
  -p='[
    {
      "op": "remove",
      "path": "/spec/selector/slo-test"
    }
  ]'
```

## Decision Intelligence Output Example

```json
{
  "incident": "Bank of Anthos frontend availability breach",
  "impact": "Frontend endpoint unavailable. Users cannot reliably access the banking frontend.",
  "severity": "warning",
  "primary_signal": "probe_success dropped to 0",
  "evidence": [
    "avg_over_time(probe_success[5m]) dropped to 0.7",
    "BankOfAnthosFrontendAvailabilitySLOBreach entered pending state",
    "frontend Service endpoints became empty",
    "frontend pod remained 1/1 Running",
    "probe_success recovered after Service selector was restored"
  ],
  "likely_root_cause": "Frontend Service selector did not match frontend pod labels",
  "safe_action": "Restore the frontend Service selector to match the frontend pod labels"
}
```# Decision Summary — Frontend Availability Breach

## Incident

Bank of Anthos frontend availability breach.

## Impact

The frontend endpoint was unavailable from the probe perspective.

Users attempting to access the frontend service path would be affected.

## Primary Signal

```promql
probe_success{job="bank-of-anthos-frontend"} = 0
```

## Evidence

| Category | Evidence |
|---|---|
| User-facing impact | Blackbox probe failed |
| SLO breach | Availability dropped below 99% |
| Alert state | `BankOfAnthosFrontendAvailabilitySLOBreach` entered pending state |
| Kubernetes routing | Frontend Service endpoints became empty |
| Pod health | Frontend pod stayed `1/1 Running` |
| Recovery | Removing the broken selector restored endpoints and probe success |

## Likely Root Cause

The frontend Service selector no longer matched the frontend pod labels.

This caused the Service to lose all endpoints.

## Safe Action

Restore the frontend Service selector so that it matches the frontend pod labels.

In this incident, recovery was completed by removing the temporary `slo-test=broken` selector:

```bash
kubectl patch svc frontend -n fintech-workload \
  --type='json' \
  -p='[
    {
      "op": "remove",
      "path": "/spec/selector/slo-test"
    }
  ]'
```

## Decision Intelligence Output Example

```json
{
  "incident": "Bank of Anthos frontend availability breach",
  "impact": "Frontend endpoint unavailable. Users cannot reliably access the banking frontend.",
  "severity": "warning",
  "primary_signal": "probe_success dropped to 0",
  "evidence": [
    "avg_over_time(probe_success[5m]) dropped to 0.7",
    "BankOfAnthosFrontendAvailabilitySLOBreach entered pending state",
    "frontend Service endpoints became empty",
    "frontend pod remained 1/1 Running",
    "probe_success recovered after Service selector was restored"
  ],
  "likely_root_cause": "Frontend Service selector did not match frontend pod labels",
  "safe_action": "Restore the frontend Service selector to match the frontend pod labels"
}
```

## Final Decision Summary

```json
{
  "incident": "Bank of Anthos frontend availability breach",
  "impact": "Frontend endpoint unavailable. Users could not reliably access the banking frontend service path.",
  "severity": "warning",
  "primary_signal": "probe_success dropped to 0",
  "slo_evidence": [
    "avg_over_time(probe_success[5m]) dropped to 0.7",
    "BankOfAnthosFrontendAvailabilitySLOBreach entered pending state"
  ],
  "kubernetes_evidence": [
    "frontend Service endpoints became empty",
    "frontend pod remained 1/1 Running",
    "frontend endpoint recovered after Service selector was restored"
  ],
  "log_evidence": [
    "OpenSearch returned structured frontend logs",
    "frontend logs were mostly INFO",
    "ERROR logs existed but were isolated and not the dominant failure signal"
  ],
  "likely_root_cause": "Frontend Service selector did not match frontend pod labels",
  "safe_action": "Restore the frontend Service selector so that it matches the frontend pod labels"
}
```

## Human-readable Summary

The frontend availability SLO detected a real user-facing outage.

Prometheus confirmed the probe failure and SLO degradation.

Kubernetes evidence explained the cause: the frontend Service had no endpoints while the frontend pod was still running.

OpenSearch logs provided application context and showed that the incident was not primarily an application crash.

The safe action was to restore the frontend Service selector.

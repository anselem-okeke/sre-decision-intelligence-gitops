# Incident Simulation — Frontend Availability Breach

## Purpose

Simulate a controlled Bank of Anthos frontend availability incident.

This scenario breaks the frontend Service selector so that the Service no longer routes traffic to frontend pods.

The frontend pods remain running, but the user-facing service path fails.

## Why this scenario matters

This demonstrates why user-facing probes are important.

A pod can be healthy while the Service path is broken.

## Baseline checks

```bash
kubectl get pods -n fintech-workload
kubectl get svc frontend -n fintech-workload
kubectl get endpoints frontend -n fintech-workload
```

## Break frontend Service routing

```bash
kubectl get svc frontend -n fintech-workload -o yaml > /tmp/frontend-service-backup.yaml

kubectl patch svc frontend -n fintech-workload \
  --type='merge' \
  -p '{"spec":{"selector":{"app":"frontend","application":"bank-of-anthos","environment":"development","team":"frontend","tier":"web","slo-test":"broken"}}}'
```

## Validate failure

```bash
kubectl get endpoints frontend -n fintech-workload
```

Expected:

```text
ENDPOINTS   <none>
```

Prometheus:

```promql
probe_success{job="bank-of-anthos-frontend"}
```

Expected:

```text
0
```

## Restore frontend Service routing

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

## Validate recovery

```bash
kubectl get endpoints frontend -n fintech-workload
```

Prometheus:

```promql
probe_success{job="bank-of-anthos-frontend"}
```

Expected:

```text
1
```

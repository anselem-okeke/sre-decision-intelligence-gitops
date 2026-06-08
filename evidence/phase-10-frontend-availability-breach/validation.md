# Frontend Availability Breach Validation

![img](/img/incidentdetect2.drawio.png)

## Objective

Validate that a controlled frontend availability incident is detected by the Bank of Anthos frontend SLO.

---

## Baseline State

### Frontend pods

```bash
kubectl get pods -n fintech-workload -o wide
```

Result:

```text
NAME                                  READY   STATUS    RESTARTS       AGE    IP             NODE       NOMINATED NODE   READINESS GATES
accounts-db-0                         1/1     Running   0              4d9h   10.244.8.76    talos-w1   <none>           <none>
balancereader-f785cf446-lqsdc         1/1     Running   1 (4d9h ago)   4d9h   10.244.8.140   talos-w1   <none>           <none>
contacts-545d94c9bc-4lxn7             1/1     Running   0              4d9h   10.244.8.174   talos-w1   <none>           <none>
frontend-67dd44c5c9-zsjc9             1/1     Running   0              4d9h   10.244.8.229   talos-w1   <none>           <none>
ledger-db-0                           1/1     Running   0              4d9h   10.244.8.195   talos-w1   <none>           <none>
ledgerwriter-5c69d874d4-qxzlf         1/1     Running   0              4d9h   10.244.8.50    talos-w1   <none>           <none>
loadgenerator-6cd654c8c7-rxlx8        1/1     Running   0              4d9h   10.244.8.190   talos-w1   <none>           <none>
transactionhistory-5b67fd6f77-wmkfj   1/1     Running   1 (4d9h ago)   4d9h   10.244.8.208   talos-w1   <none>           <none>
userservice-7f94d7f946-77ngt          1/1     Running   0              4d9h   10.244.8.178   talos-w1   <none>           <none>
```

### Frontend Service

```bash
kubectl get svc frontend -n fintech-workload
```

Result:

```text
NAME       TYPE           CLUSTER-IP    EXTERNAL-IP     PORT(S)        AGE
frontend   LoadBalancer   10.107.6.60   192.168.0.231   80:31734/TCP   4d9h
```

### Frontend Endpoints

```bash
kubectl get endpoints frontend -n fintech-workload
```

Result:

```text
NAME       ENDPOINTS           AGE
frontend   10.244.8.229:8080   4d9h
```

### Probe success

```promql
probe_success{job="bank-of-anthos-frontend"}
```

Result:

```text
probe_success{instance="http://frontend.fintech-workload.svc.cluster.local", job="bank-of-anthos-frontend", namespace="monitoring"}	1
```

---

## Incident Injection

### Command

```bash
kubectl patch svc frontend -n fintech-workload \
  --type='merge' \
  -p '{"spec":{"selector":{"app":"frontend","application":"bank-of-anthos","environment":"development","team":"frontend","tier":"web","slo-test":"broken"}}}'
```

### Expected effect

The frontend Service no longer matches frontend pods.

---

## Incident State

### Frontend Endpoints

```bash
kubectl get endpoints frontend -n fintech-workload
```

Expected:

```text
ENDPOINTS   <none>
```

Actual:

```text
NAME       ENDPOINTS   AGE
frontend   <none>      4d9h
```

### Probe success

```promql
probe_success{job="bank-of-anthos-frontend"}
```

Expected:

```text
0
```

Actual:

```text
probe_success{instance="http://frontend.fintech-workload.svc.cluster.local", job="bank-of-anthos-frontend", namespace="monitoring"}	0
```

### Availability SLO

```promql
avg_over_time(probe_success{job="bank-of-anthos-frontend"}[5m])
```

Expected:

```text
below 0.99
```

Actual:

```text
{instance="http://frontend.fintech-workload.svc.cluster.local", job="bank-of-anthos-frontend", namespace="monitoring"}	0.7
```

### Alert state

```promql
ALERTS{alertname="BankOfAnthosFrontendAvailabilitySLOBreach"}
```

Expected:

```text
active or pending, depending on alert timing
```

Actual:

```text
ALERTS{alertname="BankOfAnthosFrontendAvailabilitySLOBreach", alertstate="pending", instance="http://frontend.fintech-workload.svc.cluster.local", job="bank-of-anthos-frontend", namespace="fintech-workload", service="frontend", severity="warning", slo="frontend-availability"}	1
```

---

## Recovery

### Command

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

---

## Recovery State

### Frontend Endpoints

```bash
kubectl get endpoints frontend -n fintech-workload
```

Expected:

```text
frontend endpoints restored
```

Actual:

```text
NAME       ENDPOINTS           AGE
frontend   10.244.8.229:8080   4d9h
```

### Probe success

```promql
probe_success{job="bank-of-anthos-frontend"}
```

Expected:

```text
1
```

Actual:

```text
probe_success{instance="http://frontend.fintech-workload.svc.cluster.local", job="bank-of-anthos-frontend", namespace="monitoring"}	1
```

---

## Final Result

```text
Phase completed successfully.

A controlled frontend availability incident was injected by modifying the frontend Service selector so that it no longer matched the running frontend pod labels.

The frontend pods remained healthy and running, but the frontend Service lost all endpoints:

frontend   <none>

This proved that pod health alone is not enough to detect user-facing availability problems.

The Blackbox Exporter probe detected the broken frontend service path correctly:

probe_success = 0

The frontend availability SLO query dropped below the 99% target:

avg_over_time(probe_success{job="bank-of-anthos-frontend"}[5m]) = 0.7

The Prometheus alert entered pending state:

BankOfAnthosFrontendAvailabilitySLOBreach = pending

After removing the temporary `slo-test=broken` selector from the frontend Service, the endpoint was restored:

frontend   10.244.8.229:8080

The probe recovered successfully:

probe_success = 1

Conclusion:

The SRE Decision Intelligence Platform can now detect a real user-facing frontend availability breach using a probe-based SLO.

This incident also produced strong correlation evidence:

- frontend pods were still running
- frontend Service had no endpoints
- probe_success dropped to 0
- availability SLO dropped below target
- alert entered pending state
- restoring the Service selector resolved the issue

Likely root cause:

Frontend Service selector no longer matched frontend pod labels.

Safe action:

Restore the frontend Service selector so that it matches the frontend pod labels.
```
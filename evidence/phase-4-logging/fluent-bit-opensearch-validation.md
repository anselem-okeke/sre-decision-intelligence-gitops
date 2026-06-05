# Fluent Bit to OpenSearch Validation

## Objective

Validate that Kubernetes container logs are collected by Fluent Bit and shipped to OpenSearch.

## Components

- Talos Kubernetes
- Argo CD
- Fluent Bit
- OpenSearch
- Bank of Anthos workload

## Current Status

- OpenSearch is running and healthy.
- Fluent Bit DaemonSet is running on worker nodes.
- Logs are searchable in OpenSearch.
- Bank of Anthos namespace logs are searchable.

## Validation Commands

```bash
kubectl get applications -n argocd
kubectl get all -n observability
kubectl logs -n observability -l app.kubernetes.io/name=fluent-bit --tail=100
curl http://localhost:9200/_cat/indices?v
curl "http://localhost:9200/k8s-logs-*/_search?pretty" \
  -H 'Content-Type: application/json' \
  -d '{
    "size": 5,
    "query": {
      "match_all": {}
    }
  }'

curl "http://localhost:9200/k8s-logs-*/_search?pretty" \
  -H 'Content-Type: application/json' \
  -d '{
    "size": 5,
    "query": {
      "match": {
        "kubernetes.namespace_name": "fintech-workload"
      }
    }
  }'
```
## Result
> Fluent Bit successfully ships Kubernetes logs into OpenSearch.
> Bank of Anthos logs from the fintech-workload namespace are searchable through OpenSearch.
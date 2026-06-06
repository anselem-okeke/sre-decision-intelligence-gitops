# Phase 6 — Structured Log Parsing Validation

## Objective

Validate that Fluent Bit exposes Bank of Anthos application logs as structured OpenSearch fields.

## Expected Fields

- `app_timestamp`
- `app_message`
- `app_severity`

## Stable Workload Identity Fields

- `kubernetes.namespace_name`
- `kubernetes.pod_name`
- `kubernetes.container_name`

Kubernetes labels are intentionally disabled to avoid OpenSearch mapping conflicts.

## Validation Prerequisites

Before validating, confirm that Fluent Bit has been updated and restarted.

```bash
kubectl get cm -n observability fluent-bit -o yaml | grep -A25 "\[FILTER\]"
kubectl get cm -n observability fluent-bit -o yaml | grep -A20 "\[OUTPUT\]"
```

Expected important values:

```ini
Labels              Off
Annotations         Off
Workers             1
Trace_Error         Off
```

## Generate Fresh Workload Logs

Old OpenSearch documents created before this phase will not contain the new `app_*` fields.

Generate fresh frontend traffic:

```bash
for i in {1..30}; do
  curl -s http://192.168.0.231/ > /dev/null
  sleep 1
done

sleep 30
```

## Validation Query — app_severity Exists

```bash
curl "http://localhost:9200/k8s-logs-*/_search?pretty" \
  -H 'Content-Type: application/json' \
  -d '{
    "size": 5,
    "_source": [
      "@timestamp",
      "timestamp",
      "message",
      "severity",
      "app_timestamp",
      "app_message",
      "app_severity",
      "kubernetes.namespace_name",
      "kubernetes.pod_name",
      "kubernetes.container_name"
    ],
    "query": {
      "bool": {
        "must": [
          { "match": { "kubernetes.namespace_name": "fintech-workload" }},
          { "exists": { "field": "app_severity" }}
        ]
      }
    },
    "sort": [
      { "@timestamp": { "order": "desc" }}
    ]
  }'
```

## Expected Result Shape

```json
{
  "_source": {
    "@timestamp": "2026-06-06T02:35:50.535Z",
    "timestamp": "2026-06-06 02:35:50",
    "message": "deposit | Deposit submitted successfully.",
    "severity": "INFO",
    "app_timestamp": "2026-06-06 02:35:50",
    "app_message": "deposit | Deposit submitted successfully.",
    "app_severity": "INFO",
    "kubernetes": {
      "namespace_name": "fintech-workload",
      "pod_name": "frontend-67dd44c5c9-zsjc9",
      "container_name": "front"
    }
  }
}
```

## Query by app_severity = INFO

```bash
curl "http://localhost:9200/k8s-logs-*/_search?pretty" \
  -H 'Content-Type: application/json' \
  -d '{
    "size": 10,
    "_source": [
      "@timestamp",
      "app_timestamp",
      "app_message",
      "app_severity",
      "kubernetes.namespace_name",
      "kubernetes.pod_name",
      "kubernetes.container_name"
    ],
    "query": {
      "bool": {
        "must": [
          { "match": { "kubernetes.namespace_name": "fintech-workload" }},
          { "match": { "app_severity": "INFO" }}
        ]
      }
    },
    "sort": [
      { "@timestamp": { "order": "desc" }}
    ]
  }'
```

## Query by app_severity = ERROR

```bash
curl "http://localhost:9200/k8s-logs-*/_search?pretty" \
  -H 'Content-Type: application/json' \
  -d '{
    "size": 10,
    "_source": [
      "@timestamp",
      "app_timestamp",
      "app_message",
      "app_severity",
      "kubernetes.namespace_name",
      "kubernetes.pod_name",
      "kubernetes.container_name"
    ],
    "query": {
      "bool": {
        "must": [
          { "match": { "kubernetes.namespace_name": "fintech-workload" }},
          { "match": { "app_severity": "ERROR" }}
        ]
      }
    },
    "sort": [
      { "@timestamp": { "order": "desc" }}
    ]
  }'
```

If this returns zero results, it may simply mean the workload has not produced `ERROR` logs yet.

## Severity Aggregation

```bash
curl "http://localhost:9200/k8s-logs-*/_search?pretty" \
  -H 'Content-Type: application/json' \
  -d '{
    "size": 0,
    "query": {
      "match": {
        "kubernetes.namespace_name": "fintech-workload"
      }
    },
    "aggs": {
      "app_severity_counts": {
        "terms": {
          "field": "app_severity.keyword",
          "size": 10
        }
      }
    }
  }'
```

## Result

Pending validation.



## Notes

Old OpenSearch documents created before this phase will not contain the new `app_*` fields. Validation must use fresh logs generated after Fluent Bit is updated and restarted.

Do not validate with `kubernetes.labels.app`, because Kubernetes labels are intentionally disabled in this pipeline.

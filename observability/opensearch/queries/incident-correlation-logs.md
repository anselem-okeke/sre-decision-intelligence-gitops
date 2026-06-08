# Incident Correlation OpenSearch Queries

## Purpose

These queries collect log evidence around the frontend availability breach scenario.

The goal is to determine whether the incident is caused by:

- workload application errors
- platform issues
- routing/service mismatch
- recent GitOps/deployment activity

---

## 1. Frontend workload logs around incident window

Adjust the time window to match the incident.

```bash
curl "http://localhost:9200/k8s-logs-*/_search?pretty" \
  -H 'Content-Type: application/json' \
  -d '{
    "size": 20,
    "_source": [
      "@timestamp",
      "timestamp",
      "message",
      "severity",
      "kubernetes.namespace_name",
      "kubernetes.pod_name",
      "kubernetes.container_name",
      "kubernetes.host"
    ],
    "query": {
      "bool": {
        "must": [
          { "match": { "kubernetes.namespace_name": "fintech-workload" }},
          { "match": { "kubernetes.labels.app": "frontend" }}
        ]
      }
    },
    "sort": [
      { "@timestamp": { "order": "desc" }}
    ]
  }'
```

---

## 2. Frontend ERROR logs

```bash
curl "http://localhost:9200/k8s-logs-*/_search?pretty" \
  -H 'Content-Type: application/json' \
  -d '{
    "size": 20,
    "_source": [
      "@timestamp",
      "timestamp",
      "message",
      "severity",
      "kubernetes.namespace_name",
      "kubernetes.pod_name",
      "kubernetes.container_name"
    ],
    "query": {
      "bool": {
        "must": [
          { "match": { "kubernetes.namespace_name": "fintech-workload" }},
          { "match": { "kubernetes.labels.app": "frontend" }},
          { "match": { "severity": "ERROR" }}
        ]
      }
    },
    "sort": [
      { "@timestamp": { "order": "desc" }}
    ]
  }'
```

---

## 3. Workload severity distribution

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
      "severity_counts": {
        "terms": {
          "field": "severity.keyword",
          "size": 10
        }
      }
    }
  }'
```

---

## 4. Argo CD logs around incident context

```bash
curl "http://localhost:9200/k8s-logs-*/_search?pretty" \
  -H 'Content-Type: application/json' \
  -d '{
    "size": 20,
    "_source": [
      "@timestamp",
      "message",
      "severity",
      "kubernetes.namespace_name",
      "kubernetes.pod_name",
      "kubernetes.container_name"
    ],
    "query": {
      "bool": {
        "must": [
          { "match": { "kubernetes.namespace_name": "argocd" }}
        ],
        "should": [
          { "match": { "message": "sync" }},
          { "match": { "message": "frontend" }},
          { "match": { "message": "bank-of-anthos" }},
          { "match": { "message": "failed" }},
          { "match": { "message": "outofsync" }}
        ],
        "minimum_should_match": 1
      }
    },
    "sort": [
      { "@timestamp": { "order": "desc" }}
    ]
  }'
```

---

## 5. Observability pipeline logs

```bash
curl "http://localhost:9200/k8s-logs-*/_search?pretty" \
  -H 'Content-Type: application/json' \
  -d '{
    "size": 20,
    "_source": [
      "@timestamp",
      "message",
      "severity",
      "kubernetes.namespace_name",
      "kubernetes.pod_name",
      "kubernetes.container_name"
    ],
    "query": {
      "bool": {
        "must": [
          { "match": { "kubernetes.namespace_name": "monitoring" }}
        ],
        "should": [
          { "match": { "message": "blackbox" }},
          { "match": { "message": "prometheus" }},
          { "match": { "message": "error" }},
          { "match": { "message": "failed" }}
        ],
        "minimum_should_match": 1
      }
    },
    "sort": [
      { "@timestamp": { "order": "desc" }}
    ]
  }'
```

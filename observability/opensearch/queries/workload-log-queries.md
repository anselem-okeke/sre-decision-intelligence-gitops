# Workload Log Investigation Queries

## Purpose

These queries are used to investigate Bank of Anthos workload logs stored in OpenSearch.

The current workload namespace is:

```text
fintech-workload
```

The log index pattern is:

```text
k8s-logs-*
```

---

## 1. All workload logs

```bash
curl "http://localhost:9200/k8s-logs-*/_search?pretty" \
  -H 'Content-Type: application/json' \
  -d '{
    "size": 20,
    "query": {
      "match": {
        "kubernetes.namespace_name": "fintech-workload"
      }
    },
    "sort": [
      { "@timestamp": { "order": "desc" }}
    ]
  }'
```

---

## 2. Frontend logs

```bash
curl "http://localhost:9200/k8s-logs-*/_search?pretty" \
  -H 'Content-Type: application/json' \
  -d '{
    "size": 20,
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

## 3. Logs by app label

Replace `frontend` with another app label.

```bash
curl "http://localhost:9200/k8s-logs-*/_search?pretty" \
  -H 'Content-Type: application/json' \
  -d '{
    "size": 20,
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

## 4. Error-like workload logs

```bash
curl "http://localhost:9200/k8s-logs-*/_search?pretty" \
  -H 'Content-Type: application/json' \
  -d '{
    "size": 50,
    "query": {
      "bool": {
        "must": [
          { "match": { "kubernetes.namespace_name": "fintech-workload" }}
        ],
        "should": [
          { "match": { "message": "ERROR" }},
          { "match": { "message": "error" }},
          { "match": { "message": "failed" }},
          { "match": { "message": "failure" }},
          { "match": { "message": "timeout" }},
          { "match": { "message": "exception" }},
          { "match": { "message": "unavailable" }}
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

## 5. Payment-related logs

```bash
curl "http://localhost:9200/k8s-logs-*/_search?pretty" \
  -H 'Content-Type: application/json' \
  -d '{
    "size": 30,
    "query": {
      "bool": {
        "must": [
          { "match": { "kubernetes.namespace_name": "fintech-workload" }},
          { "match": { "message": "payment" }}
        ]
      }
    },
    "sort": [
      { "@timestamp": { "order": "desc" }}
    ]
  }'
```

---

## 6. Deposit-related logs

```bash
curl "http://localhost:9200/k8s-logs-*/_search?pretty" \
  -H 'Content-Type: application/json' \
  -d '{
    "size": 30,
    "query": {
      "bool": {
        "must": [
          { "match": { "kubernetes.namespace_name": "fintech-workload" }},
          { "match": { "message": "deposit" }}
        ]
      }
    },
    "sort": [
      { "@timestamp": { "order": "desc" }}
    ]
  }'
```

---

## 7. Login-related logs

```bash
curl "http://localhost:9200/k8s-logs-*/_search?pretty" \
  -H 'Content-Type: application/json' \
  -d '{
    "size": 30,
    "query": {
      "bool": {
        "must": [
          { "match": { "kubernetes.namespace_name": "fintech-workload" }},
          { "match": { "message": "login" }}
        ]
      }
    },
    "sort": [
      { "@timestamp": { "order": "desc" }}
    ]
  }'
```

---

## 8. Logs by Kubernetes node

Replace `talos-w1` with another node name.

```bash
curl "http://localhost:9200/k8s-logs-*/_search?pretty" \
  -H 'Content-Type: application/json' \
  -d '{
    "size": 20,
    "query": {
      "bool": {
        "must": [
          { "match": { "kubernetes.namespace_name": "fintech-workload" }},
          { "match": { "kubernetes.host": "talos-w1" }}
        ]
      }
    },
    "sort": [
      { "@timestamp": { "order": "desc" }}
    ]
  }'
```

---

## 9. Logs around an incident time window

Adjust the time range.

```bash
curl "http://localhost:9200/k8s-logs-*/_search?pretty" \
  -H 'Content-Type: application/json' \
  -d '{
    "size": 100,
    "query": {
      "bool": {
        "must": [
          { "match": { "kubernetes.namespace_name": "fintech-workload" }},
          {
            "range": {
              "@timestamp": {
                "gte": "2026-06-05T10:20:00Z",
                "lte": "2026-06-05T10:30:00Z"
              }
            }
          }
        ]
      }
    },
    "sort": [
      { "@timestamp": { "order": "asc" }}
    ]
  }'
```

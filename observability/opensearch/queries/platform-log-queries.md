# Platform Log Investigation Queries

## Purpose

These queries are used to investigate platform logs stored in OpenSearch.

Platform logs help explain whether Kubernetes, GitOps, networking, storage, or observability components are contributing to an incident.

---

## 1. Argo CD logs

```bash
curl "http://localhost:9200/k8s-logs-*/_search?pretty" \
  -H 'Content-Type: application/json' \
  -d '{
    "size": 30,
    "query": {
      "match": {
        "kubernetes.namespace_name": "argocd"
      }
    },
    "sort": [
      { "@timestamp": { "order": "desc" }}
    ]
  }'
```

---

## 2. Argo CD error-like logs

```bash
curl "http://localhost:9200/k8s-logs-*/_search?pretty" \
  -H 'Content-Type: application/json' \
  -d '{
    "size": 50,
    "query": {
      "bool": {
        "must": [
          { "match": { "kubernetes.namespace_name": "argocd" }}
        ],
        "should": [
          { "match": { "message": "error" }},
          { "match": { "message": "failed" }},
          { "match": { "message": "timeout" }},
          { "match": { "message": "sync" }},
          { "match": { "message": "comparison" }}
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

## 3. Kubernetes system logs

```bash
curl "http://localhost:9200/k8s-logs-*/_search?pretty" \
  -H 'Content-Type: application/json' \
  -d '{
    "size": 30,
    "query": {
      "match": {
        "kubernetes.namespace_name": "kube-system"
      }
    },
    "sort": [
      { "@timestamp": { "order": "desc" }}
    ]
  }'
```

---

## 4. Cilium logs

```bash
curl "http://localhost:9200/k8s-logs-*/_search?pretty" \
  -H 'Content-Type: application/json' \
  -d '{
    "size": 30,
    "query": {
      "bool": {
        "must": [
          { "match": { "kubernetes.namespace_name": "kube-system" }},
          { "match": { "kubernetes.labels.k8s-app": "cilium" }}
        ]
      }
    },
    "sort": [
      { "@timestamp": { "order": "desc" }}
    ]
  }'
```

---

## 5. Longhorn logs

```bash
curl "http://localhost:9200/k8s-logs-*/_search?pretty" \
  -H 'Content-Type: application/json' \
  -d '{
    "size": 30,
    "query": {
      "match": {
        "kubernetes.namespace_name": "longhorn-system"
      }
    },
    "sort": [
      { "@timestamp": { "order": "desc" }}
    ]
  }'
```

---

## 6. Longhorn error-like logs

```bash
curl "http://localhost:9200/k8s-logs-*/_search?pretty" \
  -H 'Content-Type: application/json' \
  -d '{
    "size": 50,
    "query": {
      "bool": {
        "must": [
          { "match": { "kubernetes.namespace_name": "longhorn-system" }}
        ],
        "should": [
          { "match": { "message": "error" }},
          { "match": { "message": "failed" }},
          { "match": { "message": "volume" }},
          { "match": { "message": "detach" }},
          { "match": { "message": "attach" }},
          { "match": { "message": "degraded" }}
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

## 7. Observability namespace logs

```bash
curl "http://localhost:9200/k8s-logs-*/_search?pretty" \
  -H 'Content-Type: application/json' \
  -d '{
    "size": 30,
    "query": {
      "match": {
        "kubernetes.namespace_name": "observability"
      }
    },
    "sort": [
      { "@timestamp": { "order": "desc" }}
    ]
  }'
```

---

## 8. Fluent Bit error-like logs

```bash
curl "http://localhost:9200/k8s-logs-*/_search?pretty" \
  -H 'Content-Type: application/json' \
  -d '{
    "size": 50,
    "query": {
      "bool": {
        "must": [
          { "match": { "kubernetes.namespace_name": "observability" }},
          { "match": { "kubernetes.labels.app.kubernetes.io/name": "fluent-bit" }}
        ],
        "should": [
          { "match": { "message": "error" }},
          { "match": { "message": "failed" }},
          { "match": { "message": "retry" }},
          { "match": { "message": "opensearch" }}
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

## 9. Logs by node

Replace `talos-w1` with the target node.

```bash
curl "http://localhost:9200/k8s-logs-*/_search?pretty" \
  -H 'Content-Type: application/json' \
  -d '{
    "size": 50,
    "query": {
      "match": {
        "kubernetes.host": "talos-w1"
      }
    },
    "sort": [
      { "@timestamp": { "order": "desc" }}
    ]
  }'
```

---

## 10. Platform logs around an incident time window

```bash
curl "http://localhost:9200/k8s-logs-*/_search?pretty" \
  -H 'Content-Type: application/json' \
  -d '{
    "size": 100,
    "query": {
      "bool": {
        "must": [
          {
            "terms": {
              "kubernetes.namespace_name.keyword": [
                "argocd",
                "kube-system",
                "longhorn-system",
                "observability",
                "monitoring"
              ]
            }
          },
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

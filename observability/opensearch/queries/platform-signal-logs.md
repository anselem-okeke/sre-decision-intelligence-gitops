# Pod restarts in the last 15 minutes
sum by (namespace, pod, container) (
  increase(kube_pod_container_status_restarts_total[15m])
)

# Pods not running
sum by (namespace, pod, phase) (
  kube_pod_status_phase{phase!="Running"}
)

# Containers not ready
sum by (namespace, pod, container) (
  kube_pod_container_status_ready == 0
)

# Deployment unavailable replicas
sum by (namespace, deployment) (
  kube_deployment_status_replicas_unavailable
)

# Deployment available replicas
sum by (namespace, deployment) (
  kube_deployment_status_replicas_available
)

# Container CPU usage
sum by (namespace, pod, container) (
  rate(container_cpu_usage_seconds_total{container!="", image!=""}[5m])
)

# Container memory working set
sum by (namespace, pod, container) (
  container_memory_working_set_bytes{container!="", image!=""}
)

# Container OOM events in the last 15 minutes
sum by (namespace, pod, container) (
  increase(container_oom_events_total[15m])
)

# Network receive errors
sum by (namespace, pod) (
  rate(container_network_receive_errors_total[5m])
)

# Network transmit errors
sum by (namespace, pod) (
  rate(container_network_transmit_errors_total[5m])
)

# Platform Signal Log Queries

## Purpose

These OpenSearch queries help investigate platform-level causes behind workload incidents.

---

## Argo CD sync or deployment-related logs

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
          { "match": { "message": "deployed" }},
          { "match": { "message": "failed" }},
          { "match": { "message": "comparison" }},
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

## Kubernetes system error-like logs

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
      "kubernetes.container_name",
      "kubernetes.host"
    ],
    "query": {
      "bool": {
        "must": [
          { "match": { "kubernetes.namespace_name": "kube-system" }}
        ],
        "should": [
          { "match": { "message": "error" }},
          { "match": { "message": "failed" }},
          { "match": { "message": "timeout" }},
          { "match": { "message": "denied" }},
          { "match": { "message": "unreachable" }}
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

## Longhorn storage-related logs

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
          { "match": { "kubernetes.namespace_name": "longhorn-system" }}
        ],
        "should": [
          { "match": { "message": "volume" }},
          { "match": { "message": "attach" }},
          { "match": { "message": "detach" }},
          { "match": { "message": "degraded" }},
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

---

## Observability pipeline logs

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
          { "match": { "kubernetes.namespace_name": "observability" }}
        ],
        "should": [
          { "match": { "message": "fluent-bit" }},
          { "match": { "message": "opensearch" }},
          { "match": { "message": "error" }},
          { "match": { "message": "failed" }},
          { "match": { "message": "retry" }}
        ],
        "minimum_should_match": 1
      }
    },
    "sort": [
      { "@timestamp": { "order": "desc" }}
    ]
  }'
```
# Workload Log Queries — OpenSearch

## Purpose

This document contains OpenSearch queries for investigating Kubernetes workload logs collected by Fluent Bit and stored in OpenSearch.

Current log flow:

```text
Kubernetes Pods
    ↓
Fluent Bit
    ↓
OpenSearch
    ↓
Investigation Queries
```

The logging platform is **cluster-wide**, but this query library starts with a workload-level focus:

```text
namespace: fintech-workload
application: Bank of Anthos
```

---

## Index Pattern

Logs are stored in daily OpenSearch indices:

```text
k8s-logs-YYYY.MM.DD
```

Use this wildcard for queries:

```text
k8s-logs-*
```

---

## Base Endpoint

When OpenSearch is port-forwarded locally:

```bash
kubectl port-forward -n observability svc/opensearch-cluster-master 9200:9200
```

Use:

```text
http://localhost:9200
```

---

## 1. List Log Indices

```bash
curl http://localhost:9200/_cat/indices?v
```

Expected output should include:

```text
k8s-logs-YYYY.MM.DD
```

---

## 2. Query Latest Logs From All Namespaces

```bash
curl "http://localhost:9200/k8s-logs-*/_search?pretty" \
  -H 'Content-Type: application/json' \
  -d '{
    "size": 10,
    "sort": [
      {
        "@timestamp": {
          "order": "desc"
        }
      }
    ],
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
      "match_all": {}
    }
  }'
```

Use this to confirm that OpenSearch is receiving logs from the cluster.

---

## 3. Query Logs From Bank of Anthos Namespace

```bash
curl "http://localhost:9200/k8s-logs-*/_search?pretty" \
  -H 'Content-Type: application/json' \
  -d '{
    "size": 10,
    "sort": [
      {
        "@timestamp": {
          "order": "desc"
        }
      }
    ],
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
      "match": {
        "kubernetes.namespace_name": "fintech-workload"
      }
    }
  }'
```

Use this to validate workload-specific log visibility.

---

## 4. Query Structured Logs With Severity Field

```bash
curl "http://localhost:9200/k8s-logs-*/_search?pretty" \
  -H 'Content-Type: application/json' \
  -d '{
    "size": 10,
    "sort": [
      {
        "@timestamp": {
          "order": "desc"
        }
      }
    ],
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
          {
            "match": {
              "kubernetes.namespace_name": "fintech-workload"
            }
          },
          {
            "exists": {
              "field": "severity"
            }
          }
        ]
      }
    }
  }'
```

Use this to confirm that structured log parsing is working.

Expected fields:

```text
timestamp
message
severity
kubernetes.namespace_name
```

---

## 5. Query INFO Logs

```bash
curl "http://localhost:9200/k8s-logs-*/_search?pretty" \
  -H 'Content-Type: application/json' \
  -d '{
    "size": 10,
    "sort": [
      {
        "@timestamp": {
          "order": "desc"
        }
      }
    ],
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
          {
            "match": {
              "kubernetes.namespace_name": "fintech-workload"
            }
          },
          {
            "match": {
              "severity": "INFO"
            }
          }
        ]
      }
    }
  }'
```

---

## 6. Query ERROR Logs

```bash
curl "http://localhost:9200/k8s-logs-*/_search?pretty" \
  -H 'Content-Type: application/json' \
  -d '{
    "size": 10,
    "sort": [
      {
        "@timestamp": {
          "order": "desc"
        }
      }
    ],
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
          {
            "match": {
              "kubernetes.namespace_name": "fintech-workload"
            }
          },
          {
            "match": {
              "severity": "ERROR"
            }
          }
        ]
      }
    }
  }'
```

Use this for incident investigation.

---

## 7. Query Payment Activity

```bash
curl "http://localhost:9200/k8s-logs-*/_search?pretty" \
  -H 'Content-Type: application/json' \
  -d '{
    "size": 10,
    "sort": [
      {
        "@timestamp": {
          "order": "desc"
        }
      }
    ],
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
          {
            "match": {
              "kubernetes.namespace_name": "fintech-workload"
            }
          },
          {
            "match": {
              "message": "payment"
            }
          }
        ]
      }
    }
  }'
```

Use this to inspect payment-related workload behavior.

---

## 8. Query Deposit Activity

```bash
curl "http://localhost:9200/k8s-logs-*/_search?pretty" \
  -H 'Content-Type: application/json' \
  -d '{
    "size": 10,
    "sort": [
      {
        "@timestamp": {
          "order": "desc"
        }
      }
    ],
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
          {
            "match": {
              "kubernetes.namespace_name": "fintech-workload"
            }
          },
          {
            "match": {
              "message": "deposit"
            }
          }
        ]
      }
    }
  }'
```

---

## 9. Query Login Activity

```bash
curl "http://localhost:9200/k8s-logs-*/_search?pretty" \
  -H 'Content-Type: application/json' \
  -d '{
    "size": 10,
    "sort": [
      {
        "@timestamp": {
          "order": "desc"
        }
      }
    ],
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
          {
            "match": {
              "kubernetes.namespace_name": "fintech-workload"
            }
          },
          {
            "match": {
              "message": "login"
            }
          }
        ]
      }
    }
  }'
```

---

## 10. Count Logs by Severity

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
      "logs_by_severity": {
        "terms": {
          "field": "severity.keyword",
          "size": 10
        }
      }
    }
  }'
```

Use this to understand severity distribution.

---

## 11. Count Logs by Pod

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
      "logs_by_pod": {
        "terms": {
          "field": "kubernetes.pod_name.keyword",
          "size": 20
        }
      }
    }
  }'
```

Use this to identify noisy pods.

---

## 12. Count Logs by Container

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
      "logs_by_container": {
        "terms": {
          "field": "kubernetes.container_name.keyword",
          "size": 20
        }
      }
    }
  }'
```

Use this to understand which containers produce the most logs.

---

## 13. Find Logs Containing HTTP 5xx

```bash
curl "http://localhost:9200/k8s-logs-*/_search?pretty" \
  -H 'Content-Type: application/json' \
  -d '{
    "size": 10,
    "sort": [
      {
        "@timestamp": {
          "order": "desc"
        }
      }
    ],
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
          {
            "match": {
              "kubernetes.namespace_name": "fintech-workload"
            }
          }
        ],
        "should": [
          {
            "match_phrase": {
              "message": "500"
            }
          },
          {
            "match_phrase": {
              "message": "502"
            }
          },
          {
            "match_phrase": {
              "message": "503"
            }
          },
          {
            "match_phrase": {
              "message": "504"
            }
          }
        ],
        "minimum_should_match": 1
      }
    }
  }'
```

Use this as an early incident query for server-side failures.

---

## 14. Find Logs Containing HTTP 4xx

```bash
curl "http://localhost:9200/k8s-logs-*/_search?pretty" \
  -H 'Content-Type: application/json' \
  -d '{
    "size": 10,
    "sort": [
      {
        "@timestamp": {
          "order": "desc"
        }
      }
    ],
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
          {
            "match": {
              "kubernetes.namespace_name": "fintech-workload"
            }
          }
        ],
        "should": [
          {
            "match_phrase": {
              "message": "400"
            }
          },
          {
            "match_phrase": {
              "message": "401"
            }
          },
          {
            "match_phrase": {
              "message": "403"
            }
          },
          {
            "match_phrase": {
              "message": "404"
            }
          }
        ],
        "minimum_should_match": 1
      }
    }
  }'
```

Use this for client-side or authorization-related investigation.

---

## 15. Query Logs in a Time Range

```bash
curl "http://localhost:9200/k8s-logs-*/_search?pretty" \
  -H 'Content-Type: application/json' \
  -d '{
    "size": 20,
    "sort": [
      {
        "@timestamp": {
          "order": "desc"
        }
      }
    ],
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
          {
            "match": {
              "kubernetes.namespace_name": "fintech-workload"
            }
          },
          {
            "range": {
              "@timestamp": {
                "gte": "now-15m",
                "lte": "now"
              }
            }
          }
        ]
      }
    }
  }'
```

Use this during recent incident investigation.

---

## 16. Recent ERROR Logs in Last 15 Minutes

```bash
curl "http://localhost:9200/k8s-logs-*/_search?pretty" \
  -H 'Content-Type: application/json' \
  -d '{
    "size": 20,
    "sort": [
      {
        "@timestamp": {
          "order": "desc"
        }
      }
    ],
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
          {
            "match": {
              "kubernetes.namespace_name": "fintech-workload"
            }
          },
          {
            "match": {
              "severity": "ERROR"
            }
          },
          {
            "range": {
              "@timestamp": {
                "gte": "now-15m",
                "lte": "now"
              }
            }
          }
        ]
      }
    }
  }'
```

Use this as a quick operational incident check.

---

## 17. Log Volume Over Time

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
      "logs_over_time": {
        "date_histogram": {
          "field": "@timestamp",
          "fixed_interval": "1m"
        }
      }
    }
  }'
```

Use this to build a Grafana log-volume panel.

---

## 18. Severity Over Time

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
      "logs_over_time": {
        "date_histogram": {
          "field": "@timestamp",
          "fixed_interval": "1m"
        },
        "aggs": {
          "by_severity": {
            "terms": {
              "field": "severity.keyword"
            }
          }
        }
      }
    }
  }'
```

Use this to build a Grafana severity trend panel.

---

## 19. Dashboard Query Ideas

These queries can later power Grafana panels:

| Panel | Query Type |
|---|---|
| Recent workload logs | Search query filtered by namespace |
| Log volume over time | `date_histogram` on `@timestamp` |
| Severity distribution | `terms` aggregation on `severity.keyword` |
| Logs by pod | `terms` aggregation on `kubernetes.pod_name.keyword` |
| Logs by container | `terms` aggregation on `kubernetes.container_name.keyword` |
| Payment activity | `message: payment` |
| Login activity | `message: login` |
| Recent errors | `severity: ERROR` |
| HTTP 5xx hints | message contains 500/502/503/504 |

---

## 20. Investigation Workflow

Use this order during troubleshooting:

```text
1. Check if logs are arriving
2. Filter by namespace
3. Filter by severity
4. Filter by pod/container
5. Search for business action
6. Search for errors
7. Narrow by time range
```

Example:

```text
Cluster-wide logs exist?
    ↓
fintech-workload logs exist?
    ↓
severity=ERROR?
    ↓
Which pod?
    ↓
What message?
    ↓
When did it happen?
```

---

## 21. Current Validation Result

Structured parsing is working when OpenSearch documents contain:

```json
{
  "timestamp": "2026-06-06 02:37:55",
  "severity": "INFO",
  "message": "POST /payment ...",
  "kubernetes": {
    "namespace_name": "fintech-workload"
  }
}
```

This proves:

```text
Fluent Bit reads workload logs
Fluent Bit parses structured JSON message field
OpenSearch stores extracted fields
OpenSearch can filter by namespace and severity
```

---

## 22. Notes

### Cluster-Wide vs Workload-Specific

The logging platform is cluster-wide:

```text
Fluent Bit collects logs from all nodes and namespaces.
OpenSearch stores logs from the whole cluster.
```

But the first investigation focus is workload-specific:

```text
namespace: fintech-workload
application: Bank of Anthos
```

This is the correct enterprise pattern:

```text
Build cluster-wide platform capability.
Prove value with one business workload.
Expand dashboards later.
```

### Old Logs Are Not Rewritten

Parser changes only affect new logs.

Old documents in OpenSearch may still contain:

```json
"message": "{\"timestamp\": \"...\", \"message\": \"...\", \"severity\": \"INFO\"}"
```

New documents should contain extracted fields:

```json
"timestamp": "..."
"message": "..."
"severity": "INFO"
```

---

## 23. Commit

After creating this file:

```bash
git add observability/opensearch/queries/workload-log-queries.md
git commit -m "docs: add OpenSearch workload log investigation queries"
git push
```

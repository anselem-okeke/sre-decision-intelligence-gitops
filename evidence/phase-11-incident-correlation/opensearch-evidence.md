# OpenSearch Evidence — Frontend Availability Breach

## Objective

Capture log evidence around the frontend availability breach.

Logs help determine whether the incident was caused by application errors or by platform/routing state.

---

## 1. Frontend logs

```bash
curl "http://localhost:9200/k8s-logs-*/_search?pretty" \
  -H 'Content-Type: application/json' \
  -d '{
    "size": 10,
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
          { "wildcard": { "kubernetes.pod_name.keyword": "frontend-*" }}
        ]
      }
    },
    "sort": [
      { "@timestamp": { "order": "desc" }}
    ]
  }'
```

Result:

```text
{
  "took" : 1225,
  "timed_out" : false,
  "_shards" : {
    "total" : 3,
    "successful" : 2,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 10000,
      "relation" : "gte"
    },
    "max_score" : null,
    "hits" : [
      {
        "_index" : "k8s-logs-2026.06.08",
        "_id" : "83E-p54BOuzox0DSe2J2",
        "_score" : null,
        "_source" : {
          "severity" : "INFO",
          "kubernetes" : {
            "container_name" : "front",
            "pod_name" : "frontend-67dd44c5c9-zsjc9",
            "namespace_name" : "fintech-workload"
          },
          "@timestamp" : "2026-06-08T12:39:10.072Z",
          "message" : "payment | Payment initiated successfully.",
          "timestamp" : "2026-06-08 12:39:10"
        },
        "sort" : [
          1780922350072
        ]
      },
      {
        "_index" : "k8s-logs-2026.06.08",
        "_id" : "8nE-p54BOuzox0DSe2J2",
        "_score" : null,
        "_source" : {
          "severity" : "INFO",
          "kubernetes" : {
            "container_name" : "front",
            "pod_name" : "frontend-67dd44c5c9-zsjc9",
            "namespace_name" : "fintech-workload"
          },
          "@timestamp" : "2026-06-08T12:39:09.423Z",
          "message" : "signup | New user created.",
          "timestamp" : "2026-06-08 12:39:09"
        },
        "sort" : [
          1780922349423
        ]
      },
      {
        "_index" : "k8s-logs-2026.06.08",
        "_id" : "8XE-p54BOuzox0DSe2J2",
        "_score" : null,
        "_source" : {
          "severity" : "INFO",
          "kubernetes" : {
            "container_name" : "front",
            "pod_name" : "frontend-67dd44c5c9-zsjc9",
            "namespace_name" : "fintech-workload"
          },
          "@timestamp" : "2026-06-08T12:39:08.729Z",
          "message" : "_login_helper | Successfully logged in.",
          "timestamp" : "2026-06-08 12:39:08"
        },
        "sort" : [
          1780922348729
        ]
      },
      {
        "_index" : "k8s-logs-2026.06.08",
        "_id" : "8HE-p54BOuzox0DSe2J2",
        "_score" : null,
        "_source" : {
          "severity" : "INFO",
          "kubernetes" : {
            "container_name" : "front",
            "pod_name" : "frontend-67dd44c5c9-zsjc9",
            "namespace_name" : "fintech-workload"
          },
          "@timestamp" : "2026-06-08T12:39:07.553Z",
          "message" : "_login_helper | Successfully logged in.",
          "timestamp" : "2026-06-08 12:39:07"
        },
        "sort" : [
          1780922347553
        ]
      },
      {
        "_index" : "k8s-logs-2026.06.08",
        "_id" : "73E-p54BOuzox0DSe2J2",
        "_score" : null,
        "_source" : {
          "severity" : "INFO",
          "kubernetes" : {
            "container_name" : "front",
            "pod_name" : "frontend-67dd44c5c9-zsjc9",
            "namespace_name" : "fintech-workload"
          },
          "@timestamp" : "2026-06-08T12:39:06.770Z",
          "message" : "deposit | Deposit submitted successfully.",
          "timestamp" : "2026-06-08 12:39:06"
        },
        "sort" : [
          1780922346770
        ]
      },
      {
        "_index" : "k8s-logs-2026.06.08",
        "_id" : "7nE-p54BOuzox0DSe2J2",
        "_score" : null,
        "_source" : {
          "severity" : "INFO",
          "kubernetes" : {
            "container_name" : "front",
            "pod_name" : "frontend-67dd44c5c9-zsjc9",
            "namespace_name" : "fintech-workload"
          },
          "@timestamp" : "2026-06-08T12:39:05.477Z",
          "message" : "deposit | Deposit submitted successfully.",
          "timestamp" : "2026-06-08 12:39:05"
        },
        "sort" : [
          1780922345477
        ]
      },
      {
        "_index" : "k8s-logs-2026.06.08",
        "_id" : "7XE-p54BOuzox0DSe2J2",
        "_score" : null,
        "_source" : {
          "severity" : "INFO",
          "kubernetes" : {
            "container_name" : "front",
            "pod_name" : "frontend-67dd44c5c9-zsjc9",
            "namespace_name" : "fintech-workload"
          },
          "@timestamp" : "2026-06-08T12:39:04.299Z",
          "message" : "deposit | Deposit submitted successfully.",
          "timestamp" : "2026-06-08 12:39:04"
        },
        "sort" : [
          1780922344299
        ]
      },
      {
        "_index" : "k8s-logs-2026.06.08",
        "_id" : "7HE-p54BOuzox0DSe2J2",
        "_score" : null,
        "_source" : {
          "severity" : "INFO",
          "kubernetes" : {
            "container_name" : "front",
            "pod_name" : "frontend-67dd44c5c9-zsjc9",
            "namespace_name" : "fintech-workload"
          },
          "@timestamp" : "2026-06-08T12:39:04.153Z",
          "message" : "deposit | Deposit submitted successfully.",
          "timestamp" : "2026-06-08 12:39:04"
        },
        "sort" : [
          1780922344153
        ]
      },
      {
        "_index" : "k8s-logs-2026.06.08",
        "_id" : "63E-p54BOuzox0DSe2J2",
        "_score" : null,
        "_source" : {
          "severity" : "INFO",
          "kubernetes" : {
            "container_name" : "front",
            "pod_name" : "frontend-67dd44c5c9-zsjc9",
            "namespace_name" : "fintech-workload"
          },
          "@timestamp" : "2026-06-08T12:39:01.743Z",
          "message" : "deposit | Deposit submitted successfully.",
          "timestamp" : "2026-06-08 12:39:01"
        },
        "sort" : [
          1780922341743
        ]
      },
      {
        "_index" : "k8s-logs-2026.06.08",
        "_id" : "6nE-p54BOuzox0DSe2J2",
        "_score" : null,
        "_source" : {
          "severity" : "INFO",
          "kubernetes" : {
            "container_name" : "front",
            "pod_name" : "frontend-67dd44c5c9-zsjc9",
            "namespace_name" : "fintech-workload"
          },
          "@timestamp" : "2026-06-08T12:39:01.706Z",
          "message" : "deposit | Deposit submitted successfully.",
          "timestamp" : "2026-06-08 12:39:01"
        },
        "sort" : [
          1780922341706
        ]
      }
    ]
  }
}
```

Interpretation:

```text
Frontend logs provide application context during the availability breach.
```

---

## 2. Frontend ERROR logs

```bash
curl "http://localhost:9200/k8s-logs-*/_search?pretty" \
  -H 'Content-Type: application/json' \
  -d '{
    "size": 10,
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
          { "wildcard": { "kubernetes.pod_name.keyword": "frontend-*" }},
          { "match": { "severity": "ERROR" }}
        ]
      }
    },
    "sort": [
      { "@timestamp": { "order": "desc" }}
    ]
  }'
```

Result:

```text
{
  "took" : 2028,
  "timed_out" : false,
  "_shards" : {
    "total" : 3,
    "successful" : 2,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 4,
      "relation" : "eq"
    },
    "max_score" : null,
    "hits" : [
      {
        "_index" : "k8s-logs-2026.06.08",
        "_id" : "GaVjpp4BOuzox0DSYOaX",
        "_score" : null,
        "_source" : {
          "severity" : "ERROR",
          "kubernetes" : {
            "container_name" : "front",
            "pod_name" : "frontend-67dd44c5c9-zsjc9",
            "namespace_name" : "fintech-workload"
          },
          "@timestamp" : "2026-06-08T08:28:01.205Z",
          "message" : "payment | Error submitting payment: insufficient balance",
          "timestamp" : "2026-06-08 08:28:01"
        },
        "sort" : [
          1780907281205
        ]
      },
      {
        "_index" : "k8s-logs-2026.06.08",
        "_id" : "EqVjpp4BOuzox0DSYOaX",
        "_score" : null,
        "_source" : {
          "severity" : "ERROR",
          "kubernetes" : {
            "container_name" : "front",
            "pod_name" : "frontend-67dd44c5c9-zsjc9",
            "namespace_name" : "fintech-workload"
          },
          "@timestamp" : "2026-06-08T08:27:56.101Z",
          "message" : "payment | Error submitting payment: insufficient balance",
          "timestamp" : "2026-06-08 08:27:56"
        },
        "sort" : [
          1780907276101
        ]
      },
      {
        "_index" : "k8s-logs-2026.06.08",
        "_id" : "y30Fpp4BABkyqOWt4sHp",
        "_score" : null,
        "_source" : {
          "severity" : "ERROR",
          "kubernetes" : {
            "container_name" : "front",
            "pod_name" : "frontend-67dd44c5c9-zsjc9",
            "namespace_name" : "fintech-workload"
          },
          "@timestamp" : "2026-06-08T06:44:46.524Z",
          "message" : "signup | Error creating new user: HTTPConnectionPool(host='userservice', port=8080): Read timed out. (read timeout=4)",
          "timestamp" : "2026-06-08 06:44:46"
        },
        "sort" : [
          1780901086524
        ]
      },
      {
        "_index" : "k8s-logs-2026.06.06",
        "_id" : "WMnZpZ4BhCEZqpLcaK2i",
        "_score" : null,
        "_source" : {
          "severity" : "ERROR",
          "kubernetes" : {
            "container_name" : "front",
            "pod_name" : "frontend-67dd44c5c9-zsjc9",
            "namespace_name" : "fintech-workload"
          },
          "@timestamp" : "2026-06-06T19:30:01.065Z",
          "message" : "deposit | Error submitting deposit: invalid amount",
          "timestamp" : "2026-06-06 19:30:01"
        },
        "sort" : [
          1780774201065
        ]
      }
    ]
  }
}
```

Interpretation:

```text
If no ERROR logs appear while the probe fails, the issue may be routing/service-level rather than application exception-level.
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

Result:

```text
{
  "took" : 1821,
  "timed_out" : false,
  "_shards" : {
    "total" : 3,
    "successful" : 2,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 10000,
      "relation" : "gte"
    },
    "max_score" : null,
    "hits" : [ ]
  },
  "aggregations" : {
    "severity_counts" : {
      "doc_count_error_upper_bound" : 0,
      "sum_other_doc_count" : 0,
      "buckets" : [
        {
          "key" : "INFO",
          "doc_count" : 1233715
        },
        {
          "key" : "ERROR",
          "doc_count" : 10
        }
      ]
    }
  }
}
```

Interpretation:

```text
Severity distribution helps determine whether workload errors increased during the incident.
```

## Final OpenSearch Interpretation

OpenSearch successfully returned structured frontend logs from the `fintech-workload` namespace.

The frontend logs contain:

- `severity`
- `message`
- `timestamp`
- `kubernetes.namespace_name`
- `kubernetes.pod_name`
- `kubernetes.container_name`

Example frontend log evidence shows normal user/business activity such as payment, signup, login, and deposit events.

The latest frontend logs are mostly `INFO`, for example:

```text
severity=INFO
pod_name=frontend-67dd44c5c9-zsjc9
namespace_name=fintech-workload
message="payment | Payment initiated successfully."
```
Frontend `ERROR` logs also exist, including examples such as:

```shell
payment | Error submitting payment: insufficient balance
signup | Error creating new user: HTTPConnectionPool(host='userservice', port=8080): Read timed out.
deposit | Error submitting deposit: invalid amount
```

However, the Phase 10 incident was not primarily explained by frontend application logs.

The strongest correlation evidence came from Kubernetes and Prometheus:

- frontend pods stayed running
- frontend Service endpoints became empty
- probe_success dropped to 0
- availability dropped below the SLO target
- the alert entered pending state
- restoring the Service selector recovered the probe

## Conclusion:

> OpenSearch provided workload context, but the likely root cause was Kubernetes Service selector mismatch, not a frontend pod crash.

## Final Correlation Result

Phase 11 completed successfully.

The incident correlation confirms that the frontend availability breach was caused by a Service routing problem.

## Correlated Evidence

| Signal | Result | Interpretation |
|---|---|---|
| `probe_success` | `0` during incident | Frontend endpoint was unavailable |
| `avg_over_time(probe_success[5m])` | `0.7` | Availability dropped below the 99% SLO target |
| Alert state | `pending` | Prometheus detected the SLO breach |
| Frontend endpoints | `<none>` | Service had no backend endpoint |
| Frontend pod | `1/1 Running` | Pod itself was healthy |
| OpenSearch frontend logs | mostly `INFO` | No strong evidence of application crash |
| OpenSearch ERROR logs | low count / isolated errors | Not the main cause of the incident |
| Recovery | `probe_success = 1` | Restoring selector fixed the issue |

## Correlation Logic

```text
IF frontend probe fails
AND frontend Service endpoints are empty
AND frontend pod is still running
AND frontend logs do not show a dominant application error storm
THEN likely root cause = Service selector / routing mismatch
```

## Final Interpretation

The incident was not caused by a frontend pod crash.

The frontend application pod remained healthy, but the frontend Service lost its endpoint because its selector no longer matched the pod labels.

The platform correctly detected the user-facing failure through the frontend SLO probe and explained the cause through Kubernetes endpoint evidence.

OpenSearch logs provided supporting workload context.

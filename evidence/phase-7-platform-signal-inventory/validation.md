# Platform Signal Inventory Validation

## Objective

Validate that the GitOps repository now contains a clear signal inventory for workload and platform signals.

---

## Workload Signal Validation

### Query: workload pod restarts

```promql
sum by (pod, container) (
  increase(kube_pod_container_status_restarts_total{namespace="fintech-workload"}[15m])
)
```

### Result

```shell
{container="front", pod="frontend-67dd44c5c9-zsjc9"}	0
{container="userservice", pod="userservice-7f94d7f946-77ngt"}	0
{container="contacts", pod="contacts-545d94c9bc-4lxn7"}	0
{container="transactionhistory", pod="transactionhistory-5b67fd6f77-wmkfj"}	0
{container="balancereader", pod="balancereader-f785cf446-lqsdc"}	0
{container="ledgerwriter", pod="ledgerwriter-5c69d874d4-qxzlf"}	0
{container="loadgenerator", pod="loadgenerator-6cd654c8c7-rxlx8"}	0
{container="postgres", pod="ledger-db-0"}	0
{container="accounts-db", pod="accounts-db-0"}	0
{container="log-test", pod="log-test"}	0
{container="json-log-test", pod="json-log-test"}
```

---

## Platform Signal Validation

### Query: deployment unavailable replicas

```promql
sum by (namespace, deployment) (
  kube_deployment_status_replicas_unavailable
)
```

### Result

```shell
{deployment="longhorn-ui", namespace="longhorn-system"}	0
{deployment="csi-provisioner", namespace="longhorn-system"}	0
{deployment="coredns", namespace="kube-system"}	0
{deployment="metrics-server", namespace="kube-system"}	0
{deployment="hubble-relay", namespace="kube-system"}	0
{deployment="cilium-operator", namespace="kube-system"}	0
{deployment="contacts", namespace="fintech-workload"}	0
{deployment="falco-falcosidekick", namespace="falco"}	0
{deployment="my-argo-cd-argocd-redis", namespace="argocd"}	0
{deployment="my-argo-cd-argocd-server", namespace="argocd"}	0
{deployment="my-argo-cd-argocd-dex-server", namespace="argocd"}	0
{deployment="loadgenerator", namespace="fintech-workload"}	0
{deployment="csi-resizer", namespace="longhorn-system"}	0
{deployment="deploy-confidence-service", namespace="deploy-confidence"}	0
{deployment="transactionhistory", namespace="fintech-workload"}	0
{deployment="kps-grafana", namespace="monitoring"}	0
{deployment="webhook-echo", namespace="falco"}	0
{deployment="userservice", namespace="fintech-workload"}	0
{deployment="my-argo-cd-argocd-applicationset-controller", namespace="argocd"}	0
{deployment="longhorn-driver-deployer", namespace="longhorn-system"}	0
{deployment="kps-kube-state-metrics", namespace="monitoring"}	0
{deployment="frontend", namespace="fintech-workload"}	0
{deployment="loki-gateway", namespace="logging"}	0
{deployment="my-argo-cd-argocd-repo-server", namespace="argocd"}	0
{deployment="csi-attacher", namespace="longhorn-system"}	0
{deployment="csi-snapshotter", namespace="longhorn-system"}	0
{deployment="my-argo-cd-argocd-notifications-controller", namespace="argocd"}	0
{deployment="balancereader", namespace="fintech-workload"}	0
{deployment="kps-kube-prometheus-stack-operator", namespace="monitoring"}	0
{deployment="hubble-ui", namespace="kube-system"}	0
{deployment="baseline-ok", namespace="deploy-confidence-tests"}	0
{deployment="ledgerwriter", namespace="fintech-workload"}
```

---

## Platform Log Validation

### Query: Argo CD logs

```bash
curl "http://localhost:9200/k8s-logs-*/_search?pretty" \
  -H 'Content-Type: application/json' \
  -d '{
    "size": 5,
    "_source": [
      "@timestamp",
      "message",
      "severity",
      "kubernetes.namespace_name",
      "kubernetes.pod_name",
      "kubernetes.container_name"
    ],
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

### Result

```shell
{
  "took" : 101,
  "timed_out" : false,
  "_shards" : {
    "total" : 18,
    "successful" : 17,
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
        "_index" : "k8s-logs-2026.05.09",
        "_id" : "I7qgmp4B5Wt4-YIBo6RH",
        "_score" : null,
        "_source" : {
          "severity" : "INFO",
          "kubernetes" : {
            "container_name" : "repo-server",
            "namespace_name" : "argocd",
            "pod_name" : "my-argo-cd-argocd-repo-server-bd57d694c-cz69b"
          },
          "@timestamp" : "2026-05-09T14:03:16.776Z",
          "message" : "time=\"2026-05-09T14:03:16Z\" level=info msg=\"finished call\" grpc.code=OK grpc.component=server grpc.method=Check grpc.method_type=unary grpc.request.deadline=\"2026-05-09T14:04:16Z\" grpc.service=grpc.health.v1.Health grpc.start_time=\"2026-05-09T14:03:16Z\" grpc.time_ms=0.47 peer.address=\"127.0.0.1:42284\" protocol=grpc"
        },
        "sort" : [
          1778335396776
        ]
      },
      {
        "_index" : "k8s-logs-2026.05.09",
        "_id" : "Irqgmp4B5Wt4-YIBo6RH",
        "_score" : null,
        "_source" : {
          "severity" : "INFO",
          "kubernetes" : {
            "container_name" : "repo-server",
            "namespace_name" : "argocd",
            "pod_name" : "my-argo-cd-argocd-repo-server-bd57d694c-cz69b"
          },
          "@timestamp" : "2026-05-09T14:03:16.775Z",
          "message" : "time=\"2026-05-09T14:03:16Z\" level=info msg=\"started call\" grpc.component=server grpc.method=Check grpc.method_type=unary grpc.request.deadline=\"2026-05-09T14:04:16Z\" grpc.service=grpc.health.v1.Health grpc.start_time=\"2026-05-09T14:03:16Z\" grpc.time_ms=0.033 peer.address=\"127.0.0.1:42284\" protocol=grpc"
        },
        "sort" : [
          1778335396775
        ]
      },
      {
        "_index" : "k8s-logs-2026.05.09",
        "_id" : "ILqgmp4B5Wt4-YIBo6RH",
        "_score" : null,
        "_source" : {
          "severity" : "INFO",
          "kubernetes" : {
            "container_name" : "repo-server",
            "namespace_name" : "argocd",
            "pod_name" : "my-argo-cd-argocd-repo-server-bd57d694c-cz69b"
          },
          "@timestamp" : "2026-05-09T14:03:06.757Z",
          "message" : "time=\"2026-05-09T14:03:06Z\" level=info msg=\"started call\" grpc.component=server grpc.method=Check grpc.method_type=unary grpc.request.deadline=\"2026-05-09T14:04:06Z\" grpc.service=grpc.health.v1.Health grpc.start_time=\"2026-05-09T14:03:06Z\" grpc.time_ms=0.007 peer.address=\"127.0.0.1:47832\" protocol=grpc"
        },
        "sort" : [
          1778335386757
        ]
      },
      {
        "_index" : "k8s-logs-2026.05.09",
        "_id" : "Ibqgmp4B5Wt4-YIBo6RH",
        "_score" : null,
        "_source" : {
          "severity" : "INFO",
          "kubernetes" : {
            "container_name" : "repo-server",
            "namespace_name" : "argocd",
            "pod_name" : "my-argo-cd-argocd-repo-server-bd57d694c-cz69b"
          },
          "@timestamp" : "2026-05-09T14:03:06.757Z",
          "message" : "time=\"2026-05-09T14:03:06Z\" level=info msg=\"finished call\" grpc.code=OK grpc.component=server grpc.method=Check grpc.method_type=unary grpc.request.deadline=\"2026-05-09T14:04:06Z\" grpc.service=grpc.health.v1.Health grpc.start_time=\"2026-05-09T14:03:06Z\" grpc.time_ms=0.338 peer.address=\"127.0.0.1:47832\" protocol=grpc"
        },
        "sort" : [
          1778335386757
        ]
      },
      {
        "_index" : "k8s-logs-2026.05.09",
        "_id" : "Hrqgmp4B5Wt4-YIBo6RH",
        "_score" : null,
        "_source" : {
          "severity" : "INFO",
          "kubernetes" : {
            "container_name" : "repo-server",
            "namespace_name" : "argocd",
            "pod_name" : "my-argo-cd-argocd-repo-server-bd57d694c-cz69b"
          },
          "@timestamp" : "2026-05-09T14:02:56.753Z",
          "message" : "time=\"2026-05-09T14:02:56Z\" level=info msg=\"started call\" grpc.component=server grpc.method=Check grpc.method_type=unary grpc.request.deadline=\"2026-05-09T14:03:56Z\" grpc.service=grpc.health.v1.Health grpc.start_time=\"2026-05-09T14:02:56Z\" grpc.time_ms=0.008 peer.address=\"127.0.0.1:41806\" protocol=grpc"
        },
        "sort" : [
          1778335376753
        ]
      }
    ]
  }
}
```

### Query: Argo CD Namespace Logs

> This query validates that logs from Argo CD pods are available in OpenSearch.

### Result 

```shell
{
  "took" : 129,
  "timed_out" : false,
  "_shards" : {
    "total" : 18,
    "successful" : 17,
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
        "_index" : "k8s-logs-2026.05.09",
        "_id" : "I7qgmp4B5Wt4-YIBo6RH",
        "_score" : null,
        "_source" : {
          "severity" : "INFO",
          "kubernetes" : {
            "container_name" : "repo-server",
            "namespace_name" : "argocd",
            "pod_name" : "my-argo-cd-argocd-repo-server-bd57d694c-cz69b"
          },
          "@timestamp" : "2026-05-09T14:03:16.776Z",
          "message" : "time=\"2026-05-09T14:03:16Z\" level=info msg=\"finished call\" grpc.code=OK grpc.component=server grpc.method=Check grpc.method_type=unary grpc.request.deadline=\"2026-05-09T14:04:16Z\" grpc.service=grpc.health.v1.Health grpc.start_time=\"2026-05-09T14:03:16Z\" grpc.time_ms=0.47 peer.address=\"127.0.0.1:42284\" protocol=grpc"
        },
        "sort" : [
          1778335396776
        ]
      },
      {
        "_index" : "k8s-logs-2026.05.09",
        "_id" : "Irqgmp4B5Wt4-YIBo6RH",
        "_score" : null,
        "_source" : {
          "severity" : "INFO",
          "kubernetes" : {
            "container_name" : "repo-server",
            "namespace_name" : "argocd",
            "pod_name" : "my-argo-cd-argocd-repo-server-bd57d694c-cz69b"
          },
          "@timestamp" : "2026-05-09T14:03:16.775Z",
          "message" : "time=\"2026-05-09T14:03:16Z\" level=info msg=\"started call\" grpc.component=server grpc.method=Check grpc.method_type=unary grpc.request.deadline=\"2026-05-09T14:04:16Z\" grpc.service=grpc.health.v1.Health grpc.start_time=\"2026-05-09T14:03:16Z\" grpc.time_ms=0.033 peer.address=\"127.0.0.1:42284\" protocol=grpc"
        },
        "sort" : [
          1778335396775
        ]
      },
      {
        "_index" : "k8s-logs-2026.05.09",
        "_id" : "ILqgmp4B5Wt4-YIBo6RH",
        "_score" : null,
        "_source" : {
          "severity" : "INFO",
          "kubernetes" : {
            "container_name" : "repo-server",
            "namespace_name" : "argocd",
            "pod_name" : "my-argo-cd-argocd-repo-server-bd57d694c-cz69b"
          },
          "@timestamp" : "2026-05-09T14:03:06.757Z",
          "message" : "time=\"2026-05-09T14:03:06Z\" level=info msg=\"started call\" grpc.component=server grpc.method=Check grpc.method_type=unary grpc.request.deadline=\"2026-05-09T14:04:06Z\" grpc.service=grpc.health.v1.Health grpc.start_time=\"2026-05-09T14:03:06Z\" grpc.time_ms=0.007 peer.address=\"127.0.0.1:47832\" protocol=grpc"
        },
        "sort" : [
          1778335386757
        ]
      },
      {
        "_index" : "k8s-logs-2026.05.09",
        "_id" : "Ibqgmp4B5Wt4-YIBo6RH",
        "_score" : null,
        "_source" : {
          "severity" : "INFO",
          "kubernetes" : {
            "container_name" : "repo-server",
            "namespace_name" : "argocd",
            "pod_name" : "my-argo-cd-argocd-repo-server-bd57d694c-cz69b"
          },
          "@timestamp" : "2026-05-09T14:03:06.757Z",
          "message" : "time=\"2026-05-09T14:03:06Z\" level=info msg=\"finished call\" grpc.code=OK grpc.component=server grpc.method=Check grpc.method_type=unary grpc.request.deadline=\"2026-05-09T14:04:06Z\" grpc.service=grpc.health.v1.Health grpc.start_time=\"2026-05-09T14:03:06Z\" grpc.time_ms=0.338 peer.address=\"127.0.0.1:47832\" protocol=grpc"
        },
        "sort" : [
          1778335386757
        ]
      },
      {
        "_index" : "k8s-logs-2026.05.09",
        "_id" : "Hrqgmp4B5Wt4-YIBo6RH",
        "_score" : null,
        "_source" : {
          "severity" : "INFO",
          "kubernetes" : {
            "container_name" : "repo-server",
            "namespace_name" : "argocd",
            "pod_name" : "my-argo-cd-argocd-repo-server-bd57d694c-cz69b"
          },
          "@timestamp" : "2026-05-09T14:02:56.753Z",
          "message" : "time=\"2026-05-09T14:02:56Z\" level=info msg=\"started call\" grpc.component=server grpc.method=Check grpc.method_type=unary grpc.request.deadline=\"2026-05-09T14:03:56Z\" grpc.service=grpc.health.v1.Health grpc.start_time=\"2026-05-09T14:02:56Z\" grpc.time_ms=0.008 peer.address=\"127.0.0.1:41806\" protocol=grpc"
        },
        "sort" : [
          1778335376753
        ]
      },
      {
        "_index" : "k8s-logs-2026.05.09",
        "_id" : "H7qgmp4B5Wt4-YIBo6RH",
        "_score" : null,
        "_source" : {
          "severity" : "INFO",
          "kubernetes" : {
            "container_name" : "repo-server",
            "namespace_name" : "argocd",
            "pod_name" : "my-argo-cd-argocd-repo-server-bd57d694c-cz69b"
          },
          "@timestamp" : "2026-05-09T14:02:56.753Z",
          "message" : "time=\"2026-05-09T14:02:56Z\" level=info msg=\"finished call\" grpc.code=OK grpc.component=server grpc.method=Check grpc.method_type=unary grpc.request.deadline=\"2026-05-09T14:03:56Z\" grpc.service=grpc.health.v1.Health grpc.start_time=\"2026-05-09T14:02:56Z\" grpc.time_ms=0.064 peer.address=\"127.0.0.1:41806\" protocol=grpc"
        },
        "sort" : [
          1778335376753
        ]
      },
      {
        "_index" : "k8s-logs-2026.05.09",
        "_id" : "Hbqgmp4B5Wt4-YIBo6RH",
        "_score" : null,
        "_source" : {
          "severity" : "INFO",
          "kubernetes" : {
            "container_name" : "repo-server",
            "namespace_name" : "argocd",
            "pod_name" : "my-argo-cd-argocd-repo-server-bd57d694c-cz69b"
          },
          "@timestamp" : "2026-05-09T14:02:46.760Z",
          "message" : "time=\"2026-05-09T14:02:46Z\" level=info msg=\"finished call\" grpc.code=OK grpc.component=server grpc.method=Check grpc.method_type=unary grpc.request.deadline=\"2026-05-09T14:03:46Z\" grpc.service=grpc.health.v1.Health grpc.start_time=\"2026-05-09T14:02:46Z\" grpc.time_ms=0.126 peer.address=\"127.0.0.1:57628\" protocol=grpc"
        },
        "sort" : [
          1778335366760
        ]
      },
      {
        "_index" : "k8s-logs-2026.05.09",
        "_id" : "HLqgmp4B5Wt4-YIBo6RH",
        "_score" : null,
        "_source" : {
          "severity" : "INFO",
          "kubernetes" : {
            "container_name" : "repo-server",
            "namespace_name" : "argocd",
            "pod_name" : "my-argo-cd-argocd-repo-server-bd57d694c-cz69b"
          },
          "@timestamp" : "2026-05-09T14:02:46.759Z",
          "message" : "time=\"2026-05-09T14:02:46Z\" level=info msg=\"started call\" grpc.component=server grpc.method=Check grpc.method_type=unary grpc.request.deadline=\"2026-05-09T14:03:46Z\" grpc.service=grpc.health.v1.Health grpc.start_time=\"2026-05-09T14:02:46Z\" grpc.time_ms=0.009 peer.address=\"127.0.0.1:57628\" protocol=grpc"
        },
        "sort" : [
          1778335366759
        ]
      },
      {
        "_index" : "k8s-logs-2026.05.09",
        "_id" : "Grqgmp4B5Wt4-YIBo6RH",
        "_score" : null,
        "_source" : {
          "severity" : "INFO",
          "kubernetes" : {
            "container_name" : "repo-server",
            "namespace_name" : "argocd",
            "pod_name" : "my-argo-cd-argocd-repo-server-bd57d694c-cz69b"
          },
          "@timestamp" : "2026-05-09T14:02:36.752Z",
          "message" : "time=\"2026-05-09T14:02:36Z\" level=info msg=\"started call\" grpc.component=server grpc.method=Check grpc.method_type=unary grpc.request.deadline=\"2026-05-09T14:03:36Z\" grpc.service=grpc.health.v1.Health grpc.start_time=\"2026-05-09T14:02:36Z\" grpc.time_ms=0.007 peer.address=\"127.0.0.1:55218\" protocol=grpc"
        },
        "sort" : [
          1778335356752
        ]
      },
      {
        "_index" : "k8s-logs-2026.05.09",
        "_id" : "G7qgmp4B5Wt4-YIBo6RH",
        "_score" : null,
        "_source" : {
          "severity" : "INFO",
          "kubernetes" : {
            "container_name" : "repo-server",
            "namespace_name" : "argocd",
            "pod_name" : "my-argo-cd-argocd-repo-server-bd57d694c-cz69b"
          },
          "@timestamp" : "2026-05-09T14:02:36.752Z",
          "message" : "time=\"2026-05-09T14:02:36Z\" level=info msg=\"finished call\" grpc.code=OK grpc.component=server grpc.method=Check grpc.method_type=unary grpc.request.deadline=\"2026-05-09T14:03:36Z\" grpc.service=grpc.health.v1.Health grpc.start_time=\"2026-05-09T14:02:36Z\" grpc.time_ms=0.071 peer.address=\"127.0.0.1:55218\" protocol=grpc"
        },
        "sort" : [
          1778335356752
        ]
      }
    ]
  }
}
```
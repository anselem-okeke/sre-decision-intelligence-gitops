# Log Investigation Validation

## Objective

Validate that OpenSearch can be used to investigate workload and platform logs.

## Workload query validation

### Query

```bash
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

### Result

OpenSearch returned Bank of Anthos logs from the `fintech-workload` namespace.

Confirmed fields:

- `@timestamp`
- `message`
- `kubernetes.namespace_name`
- `kubernetes.pod_name`
- `kubernetes.labels.app`
- `kubernetes.host`
- `kubernetes.container_name`

## Platform query validation

### Query

```bash
curl "http://localhost:9200/k8s-logs-*/_search?pretty" \
  -H 'Content-Type: application/json' \
  -d '{
    "size": 5,
    "query": {
      "match": {
        "kubernetes.namespace_name": "argocd"
      }
    }
  }'
```

### Result

Pending validation.

## Notes

The workload log pipeline is working.

Below is structured parsing of the nested JSON inside the `message` field.

## Workload latest logs

```shell
curl "http://localhost:9200/k8s-logs-*/_search?pretty" \
  -H 'Content-Type: application/json' \
  -d '{
    "size": 5,
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

```shell
{
  "took" : 859,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
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
        "_index" : "k8s-logs-2026.06.05",
        "_id" : "deEamp4B5Wt4-YIBdpPg",
        "_score" : null,
        "_source" : {
          "@timestamp" : "2026-06-05T23:24:46.080Z",
          "time" : "2026-06-05T23:24:46.080455396Z",
          "stream" : "stdout",
          "logtag" : "F",
          "message" : "POST     /payment                                                                      121478    67(0.06%) |    291       4    5290    280 |    0.30        0.00",
          "kubernetes" : {
            "pod_name" : "loadgenerator-6cd654c8c7-rxlx8",
            "namespace_name" : "fintech-workload",
            "pod_id" : "41e40334-805b-4a22-ae97-367e1daa81fd",
            "labels" : {
              "app" : "loadgenerator",
              "environment" : "development",
              "pod-template-hash" : "6cd654c8c7",
              "team" : "loadgenerator",
              "tier" : "test"
            },
            "annotations" : {
              "sidecar.istio.io/rewriteAppHTTPProbers" : "true"
            },
            "host" : "talos-w1",
            "pod_ip" : "10.244.8.190",
            "container_name" : "loadgenerator",
            "docker_id" : "8c9f0dba48eeae3782b0347d3c7ef1ba74a71ff2e6ca2ed5838e743955815b73",
            "container_hash" : "us-central1-docker.pkg.dev/bank-of-anthos-ci/bank-of-anthos/loadgenerator@sha256:a95600a753cad11b89ed53026ed49abefd830126b8574881659cc74cdc27d1a2",
            "container_image" : "sha256:b151bedf4e6a4d578dad2485c0f36cd8696a48568f46e7d0863bc8cc522b458c"
          }
        },
        "sort" : [
          1780701886080
        ]
      },
      {
        "_index" : "k8s-logs-2026.06.05",
        "_id" : "duEamp4B5Wt4-YIBdpPg",
        "_score" : null,
        "_source" : {
          "@timestamp" : "2026-06-05T23:24:46.080Z",
          "time" : "2026-06-05T23:24:46.080468069Z",
          "stream" : "stdout",
          "logtag" : "F",
          "message" : "GET      /signup                                                                       122213    13(0.01%) |      4       1    1703      2 |    1.30        0.00",
          "kubernetes" : {
            "pod_name" : "loadgenerator-6cd654c8c7-rxlx8",
            "namespace_name" : "fintech-workload",
            "pod_id" : "41e40334-805b-4a22-ae97-367e1daa81fd",
            "labels" : {
              "app" : "loadgenerator",
              "environment" : "development",
              "pod-template-hash" : "6cd654c8c7",
              "team" : "loadgenerator",
              "tier" : "test"
            },
            "annotations" : {
              "sidecar.istio.io/rewriteAppHTTPProbers" : "true"
            },
            "host" : "talos-w1",
            "pod_ip" : "10.244.8.190",
            "container_name" : "loadgenerator",
            "docker_id" : "8c9f0dba48eeae3782b0347d3c7ef1ba74a71ff2e6ca2ed5838e743955815b73",
            "container_hash" : "us-central1-docker.pkg.dev/bank-of-anthos-ci/bank-of-anthos/loadgenerator@sha256:a95600a753cad11b89ed53026ed49abefd830126b8574881659cc74cdc27d1a2",
            "container_image" : "sha256:b151bedf4e6a4d578dad2485c0f36cd8696a48568f46e7d0863bc8cc522b458c"
          }
        },
        "sort" : [
          1780701886080
        ]
      },
      {
        "_index" : "k8s-logs-2026.06.05",
        "_id" : "d-Eamp4B5Wt4-YIBdpPg",
        "_score" : null,
        "_source" : {
          "@timestamp" : "2026-06-05T23:24:46.080Z",
          "time" : "2026-06-05T23:24:46.080473289Z",
          "stream" : "stdout",
          "logtag" : "F",
          "message" : "POST     /signup                                                                        24326     5(0.02%) |   2187       2    7688   2000 |    0.20        0.00",
          "kubernetes" : {
            "pod_name" : "loadgenerator-6cd654c8c7-rxlx8",
            "namespace_name" : "fintech-workload",
            "pod_id" : "41e40334-805b-4a22-ae97-367e1daa81fd",
            "labels" : {
              "app" : "loadgenerator",
              "environment" : "development",
              "pod-template-hash" : "6cd654c8c7",
              "team" : "loadgenerator",
              "tier" : "test"
            },
            "annotations" : {
              "sidecar.istio.io/rewriteAppHTTPProbers" : "true"
            },
            "host" : "talos-w1",
            "pod_ip" : "10.244.8.190",
            "container_name" : "loadgenerator",
            "docker_id" : "8c9f0dba48eeae3782b0347d3c7ef1ba74a71ff2e6ca2ed5838e743955815b73",
            "container_hash" : "us-central1-docker.pkg.dev/bank-of-anthos-ci/bank-of-anthos/loadgenerator@sha256:a95600a753cad11b89ed53026ed49abefd830126b8574881659cc74cdc27d1a2",
            "container_image" : "sha256:b151bedf4e6a4d578dad2485c0f36cd8696a48568f46e7d0863bc8cc522b458c"
          }
        },
        "sort" : [
          1780701886080
        ]
      },
      {
        "_index" : "k8s-logs-2026.06.05",
        "_id" : "eOEamp4B5Wt4-YIBdpPg",
        "_score" : null,
        "_source" : {
          "@timestamp" : "2026-06-05T23:24:46.080Z",
          "time" : "2026-06-05T23:24:46.080478008Z",
          "stream" : "stdout",
          "logtag" : "F",
          "message" : "--------|----------------------------------------------------------------------------|-------|-------------|-------|-------|-------|-------|--------|-----------",
          "kubernetes" : {
            "pod_name" : "loadgenerator-6cd654c8c7-rxlx8",
            "namespace_name" : "fintech-workload",
            "pod_id" : "41e40334-805b-4a22-ae97-367e1daa81fd",
            "labels" : {
              "app" : "loadgenerator",
              "environment" : "development",
              "pod-template-hash" : "6cd654c8c7",
              "team" : "loadgenerator",
              "tier" : "test"
            },
            "annotations" : {
              "sidecar.istio.io/rewriteAppHTTPProbers" : "true"
            },
            "host" : "talos-w1",
            "pod_ip" : "10.244.8.190",
            "container_name" : "loadgenerator",
            "docker_id" : "8c9f0dba48eeae3782b0347d3c7ef1ba74a71ff2e6ca2ed5838e743955815b73",
            "container_hash" : "us-central1-docker.pkg.dev/bank-of-anthos-ci/bank-of-anthos/loadgenerator@sha256:a95600a753cad11b89ed53026ed49abefd830126b8574881659cc74cdc27d1a2",
            "container_image" : "sha256:b151bedf4e6a4d578dad2485c0f36cd8696a48568f46e7d0863bc8cc522b458c"
          }
        },
        "sort" : [
          1780701886080
        ]
      },
      {
        "_index" : "k8s-logs-2026.06.05",
        "_id" : "eeEamp4B5Wt4-YIBdpPg",
        "_score" : null,
        "_source" : {
          "@timestamp" : "2026-06-05T23:24:46.080Z",
          "time" : "2026-06-05T23:24:46.080482737Z",
          "stream" : "stdout",
          "logtag" : "F",
          "message" : "         Aggregated                                                                   1164033   181(0.02%) |    274       1    7688     22 |    4.90        0.00",
          "kubernetes" : {
            "pod_name" : "loadgenerator-6cd654c8c7-rxlx8",
            "namespace_name" : "fintech-workload",
            "pod_id" : "41e40334-805b-4a22-ae97-367e1daa81fd",
            "labels" : {
              "app" : "loadgenerator",
              "environment" : "development",
              "pod-template-hash" : "6cd654c8c7",
              "team" : "loadgenerator",
              "tier" : "test"
            },
            "annotations" : {
              "sidecar.istio.io/rewriteAppHTTPProbers" : "true"
            },
            "host" : "talos-w1",
            "pod_ip" : "10.244.8.190",
            "container_name" : "loadgenerator",
            "docker_id" : "8c9f0dba48eeae3782b0347d3c7ef1ba74a71ff2e6ca2ed5838e743955815b73",
            "container_hash" : "us-central1-docker.pkg.dev/bank-of-anthos-ci/bank-of-anthos/loadgenerator@sha256:a95600a753cad11b89ed53026ed49abefd830126b8574881659cc74cdc27d1a2",
            "container_image" : "sha256:b151bedf4e6a4d578dad2485c0f36cd8696a48568f46e7d0863bc8cc522b458c"
          }
        },
        "sort" : [
          1780701886080
        ]
      }
    ]
  }
}
```

## Workload error-like logs
```shell
curl "http://localhost:9200/k8s-logs-*/_search?pretty" \
  -H 'Content-Type: application/json' \
  -d '{
    "size": 10,
    "query": {
      "bool": {
        "must": [
          { "match": { "kubernetes.namespace_name": "fintech-workload" }}
        ],
        "should": [
          { "match": { "message": "error" }},
          { "match": { "message": "failed" }},
          { "match": { "message": "timeout" }}
        ],
        "minimum_should_match": 1
      }
    },
    "sort": [
      { "@timestamp": { "order": "desc" }}
    ]
  }'
```
```shell
{
  "took" : 836,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 0,
      "relation" : "eq"
    },
    "max_score" : null,
    "hits" : [ ]
  }
}
```
## Platform Argo CD logs
```shell
curl "http://localhost:9200/k8s-logs-*/_search?pretty" \
  -H 'Content-Type: application/json' \
  -d '{
    "size": 5,
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
```shell
{
  "took" : 547,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 0,
      "relation" : "eq"
    },
    "max_score" : null,
    "hits" : [ ]
  }
}
```

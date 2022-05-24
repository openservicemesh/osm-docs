---
title: "Retry Policy"
description: "Set up application with retry"
type: docs
weight: 15
---

This guide demonstrates how to configure retry policy for a client and server application within the service mesh.

## Prerequisites

- Kubernetes cluster running Kubernetes {{< param min_k8s_version >}} or greater.
- Have `kubectl` available to interact with the API server.
- Have `osm` CLI available for managing the service mesh.

## Demo
1. Install OSM with permissive mode and retry policy enabled.
    ```bash
    osm install --set=osm.enablePermissiveTrafficPolicy=true --set=osm.featureFlags.enableRetryPolicy=true 
    ```

1. Deploy the `httpbin` service into the `httpbin` namespace after enrolling its namespace to the mesh and enabling metrics. The `httpbin` service runs on port `14001`.

    ```bash
    kubectl create namespace httpbin

    osm namespace add httpbin

    kubectl apply -f https://raw.githubusercontent.com/openservicemesh/osm-docs/{{< param osm_branch >}}/manifests/samples/httpbin/httpbin.yaml -n httpbin
    ```

    Confirm the `httpbin` service and pods are up and running.

    ```console
    $ kubectl get svc -n httpbin
    NAME      TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)     AGE
    httpbin   ClusterIP   10.96.198.23   <none>        14001/TCP   20s
    ```

    ```console
    $ kubectl get pods -n httpbin
    NAME                     READY   STATUS    RESTARTS   AGE
    httpbin-5b8b94b9-lt2vs   2/2     Running   0          20s
    ```
    
1. Deploy the `curl` into the `curl` namespace after enrolling its namespace to the mesh and enabling metrics.
    ```bash
    kubectl create namespace curl

    osm namespace add curl

    osm metrics enable --namespace curl

    kubectl apply -f https://raw.githubusercontent.com/openservicemesh/osm-docs/{{< param osm_branch >}}/manifests/samples/curl/curl.yaml -n curl
    ```

    Confirm the `curl` pod is up and running.

    ```console
    $ kubectl get pods -n curl
    NAME                    READY   STATUS    RESTARTS   AGE
    curl-54ccc6954c-9rlvp   2/2     Running   0          20s
    ```

1. Apply the Retry policy to retry when the `curl` ServiceAccount receives a `5xx` code when sending a request to `httpbin` Service.
```bash
kubectl apply -f - <<EOF
kind: Retry
apiVersion: policy.openservicemesh.io/v1alpha1
metadata:
  name: retry
spec:
  source:
    kind: ServiceAccount
    name: curl
    namespace: curl
  destinations:
  - kind: Service
    name: httpbin
    namespace: httpbin
  retryPolicy:
    retryOn: "5xx"
    perTryTimeout: 1s
    numRetries: 5
    retryBackoffBaseInterval: 1s
EOF
```

1. Send a HTTP request that returns status code `503` from the `curl` pod to the `httpbin` service.
    ```console
    $ kubectl exec deploy/curl -n curl -c curl -- curl -sI httpbin.httpbin.svc.cluster.local:14001/status/503
    ```

1. In a new terminal session, run the following command to port-forward the `curl` pod.
    ```bash
    kubectl port-forward deploy/curl -n curl 15000
    ```

1. Query for the stats between `curl` to `httpbin`.
    ```bash
    curl -s localhost:15000/stats | grep "cluster.httpbin/httpbin|14001.upstream_rq_retry"
    ```
 The number of times the request from the `curl` pod to the `httpbin` pod was retried using the exponential backoff retry should be equal to the `numRetries` field in the retry policy.
 The `upstream_rq_retry_limit_exceeded` stat shows the number of requests not retried because it's more than the maximum retries allowed (`numRetries`).

    ```console
    cluster.httpbin/httpbin|14001.upstream_rq_retry: 5
    cluster.httpbin/httpbin|14001.upstream_rq_retry_backoff_exponential: 5
    cluster.httpbin/httpbin|14001.upstream_rq_retry_backoff_ratelimited: 0
    cluster.httpbin/httpbin|14001.upstream_rq_retry_limit_exceeded: 1
    cluster.httpbin/httpbin|14001.upstream_rq_retry_overflow: 0
    cluster.httpbin/httpbin|14001.upstream_rq_retry_success: 0
    ```

1. Send a HTTP request that returns a non-5xx status code from the `curl` pod to the `httpbin` service.
    ```console
    $ kubectl exec deploy/curl -n curl -c curl -- curl -sI httpbin.httpbin.svc.cluster.local:14001/status/404
    ```

1. The `envoy_cluster_upstream_rq_retry` metric does not increment since the retry policy is set to retry on `5xx` 

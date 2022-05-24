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
1. Install OSM with Prometheus, permissive mode and retry policy enabled.
    ```bash
    osm install --set=osm.deployPrometheus=true --set=osm.enablePermissiveTrafficPolicy=true --set=osm.featureFlags.enableRetryPolicy=true 
    ```

1. Deploy the `httpbin` service into the `httpbin` namespace after enrolling its namespace to the mesh and enabling metrics. The `httpbin` service runs on port `14001`.

    ```bash
    kubectl create namespace httpbin

    osm namespace add httpbin

    osm metrics enable --namespace httpbin

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
    HTTP/1.1 503 Service Unavailable
    server: envoy
    date: Fri, 13 May 2022 19:17:52 GMT
    content-type: text/html; charset=utf-8
    access-control-allow-origin: *
    access-control-allow-credentials: true
    content-length: 0
    x-envoy-upstream-service-time: 9429
    ```

1. In a new terminal session, run the following command to enable port-forwarding for Prometheus.
    ```bash
    ./scripts/port-forward-prometheus.sh
    ```

1. In a browser open the prometheus url - http://localhost:7070

1. Query the `envoy_cluster_upstream_rq_retry` metric. The number of times the request from the `curl` pod to the `httpbin` pod was retried should be equal to the `numRetries` field in the retry policy.
```console
envoy_cluster_upstream_rq_retry{envoy_cluster_name="httpbin/httpbin|14001", instance="10.244.1.5:15010", job="kubernetes-pods", source_namespace="curl", source_pod_name="curl-548c575854-lx4b5", source_service="curl", source_workload_kind="Deployment", source_workload_name="curl"}
5
```

1. Send a HTTP request that returns a non-5xx status code from the `curl` pod to the `httpbin` service.
    ```console
    $ kubectl exec deploy/curl -n curl -c curl -- curl -sI httpbin.httpbin.svc.cluster.local:14001/status/404
    HTTP/1.1 404 Not Found
    server: envoy
    date: Fri, 13 May 2022 19:25:14 GMT
    content-type: text/html; charset=utf-8
    access-control-allow-origin: *
    access-control-allow-credentials: true
    content-length: 0
    x-envoy-upstream-service-time: 3
    ```

1. The `envoy_cluster_upstream_rq_retry` metric does not increment since the retry policy is set to retry on `5xx` 

---
title: "Circuit breaking for destinations external to the mesh"
description: "Configuring circuit breaking for destinations external to the mesh"
type: docs
weight: 22
---

This guide demonstrates how to configure circuit breaking for destinations that are external to the OSM managed service mesh.

## Prerequisites

- Kubernetes cluster running Kubernetes {{< param min_k8s_version >}} or greater.
- Have OSM installed.
- Have `kubectl` available to interact with the API server.
- Have `osm` CLI available for managing the service mesh.
- OSM version >= v1.1.0.


## Demo

The following demo shows a load-testing client [fortio](https://github.com/fortio/fortio) sending traffic to the `httpbin` service that is external to the service mesh. Traffic external to the mesh is treated as [Egress](/docs/guides/traffic_management/egress) traffic, and will be authorized using an [Egress traffic policy](/docs/guides/traffic_management/egress/#1-configuring-egress-policies). We will see how applying circuit breakers for traffic to the external `httpbin` service impacts the `fortio` client when the configured circuit breaking limits trip.

1. Deploy the `httpbin` service into the `httpbin` namespace. The `httpbin` service runs on port `14001` and is not added to the mesh, so it is considered to be a destination external to the mesh.

    Create the httpbin namespace
    ```bash
    kubectl create namespace httpbin
    ```
    Deploy httpbin service in the httpbin namespace
    ```bash
    kubectl apply -f https://raw.githubusercontent.com/openservicemesh/osm-docs/{{< param osm_branch >}}/manifests/samples/httpbin/httpbin.yaml -n httpbin
    ```

    Confirm the `httpbin` service and pods are up and running.

    ```console
    kubectl get svc -n httpbin
    ```
    The output will be si,ilar to:
    ```console
    NAME      TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)     AGE
    httpbin   ClusterIP   10.96.198.23   <none>        14001/TCP   20s
    ```

    ```console
    kubectl get pods -n httpbin
    ```
    The output will be similar to:
    ```console
    NAME                     READY   STATUS    RESTARTS   AGE
    httpbin-5b8b94b9-lt2vs   1/1     Running   0          20s
    ```

1. Deploy the `fortio` load-testing client in the `client` namespace after enrolling its namespace to the mesh.
    Create the client namespace
    ```bash
    kubectl create namespace client
    ```

    Add the namespace to the mesh
    ```bash
    osm namespace add client
    ```

    # Deploy fortio client in the client namespace
    ```bash
    kubectl apply -f https://raw.githubusercontent.com/openservicemesh/osm-docs/{{< param osm_branch >}}/manifests/samples/fortio/fortio.yaml -n client
    ```

    Confirm the `fortio` client pod is up and running.

    ```console
    kubectl get pods -n client
    ```
    The output will be si,ilar to:
    ```console
    NAME                      READY   STATUS    RESTARTS   AGE
    fortio-6477f8495f-bj4s9   2/2     Running   0          19s
    ```

1. Configure an Egress policy that allows the `fortio` client in the `client` namespace to communicate with the external `httpbin` service. The HTTP requests will be directed to the host `httpbin.httpbin.svc.cluster.local` on port `14001`.
    ```bash
    kubectl apply -f - <<EOF
    kind: Egress
    apiVersion: policy.openservicemesh.io/v1alpha1
    metadata:
      name: httpbin-external
      namespace: client
    spec:
      sources:
      - kind: ServiceAccount
        name: default
        namespace: client
      hosts:
      - httpbin.httpbin.svc.cluster.local
      ports:
      - number: 14001
        protocol: http
    EOF
    ```

1. Confirm the `fortio` client is able to successfully make HTTP requests to the external host `httpbin.httpbin.svc.cluster.local` service on port `14001`. We call the external service with `5` concurrent connections (`-c 5`) and send `50` requests (`-n 50`).
    ```console
    export fortio_pod="$(kubectl get pod -n client -l app=fortio -o jsonpath='{.items[0].metadata.name}')"
    ```
    Theoutpul will be similar to:
    ```console
    kubectl exec "$fortio_pod" -c fortio -n client -- /usr/bin/fortio load -c 5 -qps 0 -n 50 -loglevel Warning http://httpbin.httpbin.svc.cluster.local:14001/get
    19:56:34 I logger.go:127> Log level is now 3 Warning (was 2 Info)
    Fortio 1.17.1 running at 0 queries per second, 8->8 procs, for 50 calls: http://httpbin.httpbin.svc.cluster.local:14001/get
    Starting at max qps with 5 thread(s) [gomax 8] for exactly 50 calls (10 per thread + 0)
    Ended after 36.3659ms : 50 calls. qps=1374.9
    Aggregated Function Time : count 50 avg 0.003374618 +/- 0.0007546 min 0.0013124 max 0.0066215 sum 0.1687309
    # range, mid point, percentile, count
    >= 0.0013124 <= 0.002 , 0.0016562 , 4.00, 2
    > 0.002 <= 0.003 , 0.0025 , 10.00, 3
    > 0.003 <= 0.004 , 0.0035 , 86.00, 38
    > 0.004 <= 0.005 , 0.0045 , 98.00, 6
    > 0.006 <= 0.0066215 , 0.00631075 , 100.00, 1
    # target 50% 0.00352632
    # target 75% 0.00385526
    # target 90% 0.00433333
    # target 99% 0.00631075
    # target 99.9% 0.00659043
    Sockets used: 5 (for perfect keepalive, would be 5)
    Jitter: false
    Code 200 : 50 (100.0 %)
    Response Header Sizes : count 50 avg 230 +/- 0 min 230 max 230 sum 11500
    Response Body/Total Sizes : count 50 avg 460 +/- 0 min 460 max 460 sum 23000
    All done 50 calls (plus 0 warmup) 3.375 ms avg, 1374.9 qps
    ```

    As seen above, all the requests succeeded.
    ```
    Code 200 : 50 (100.0 %)
    ```

1. Next, apply a circuit breaker configuration using the `UpstreamTrafficSetting` resource for traffic directed to the external host `httpbin.httpbin.svc.cluster.local` to limit the maximum number of concurrent connections and requests to `1`. When applying an `UpstreamTrafficSetting` configuration for external (egress) traffic, the `UpstreamTrafficSetting` resource must also be specified as a match in the `Egress` configuration and belong to the same namespace as the matching `Egress` resource. This is required to enforce circuit breaking limits for external traffic. Hence, we also update the previously applied `Egress` configuration to specify a `matches` field.
    ```bash
    kubectl apply -f - <<EOF
    apiVersion: policy.openservicemesh.io/v1alpha1
    kind: UpstreamTrafficSetting
    metadata:
      name: httpbin-external
      namespace: client
    spec:
      host: httpbin.httpbin.svc.cluster.local
      connectionSettings:
        tcp:
          maxConnections: 1
        http:
          maxPendingRequests: 1
          maxRequestsPerConnection: 1
    ---
    kind: Egress
    apiVersion: policy.openservicemesh.io/v1alpha1
    metadata:
      name: httpbin-external
      namespace: client
    spec:
      sources:
      - kind: ServiceAccount
        name: default
        namespace: client
      hosts:
      - httpbin.httpbin.svc.cluster.local
      ports:
      - number: 14001
        protocol: http
      matches:
      - apiGroup: policy.openservicemesh.io/v1alpha1
        kind: UpstreamTrafficSetting
        name: httpbin-external
    EOF
    ```

1. Confirm the `fortio` client is unable to make the same amount of successful requests as before due to the connection and request level circuit breaking limits configured above.
    ```console
    kubectl exec "$fortio_pod" -c fortio -n client -- /usr/bin/fortio load -c 5 -qps 0 -n 50 -loglevel Warning http://httpbin.httpbin.svc.cluster.local:14001/get
    ```
    The output will be similar to:
    ```console
    19:58:48 I logger.go:127> Log level is now 3 Warning (was 2 Info)
    Fortio 1.17.1 running at 0 queries per second, 8->8 procs, for 50 calls: http://httpbin.httpbin.svc.cluster.local:14001/get
    Starting at max qps with 5 thread(s) [gomax 8] for exactly 50 calls (10 per thread + 0)
    19:58:48 W http_client.go:806> [0] Non ok http code 503 (HTTP/1.1 503)
    19:58:48 W http_client.go:806> [2] Non ok http code 503 (HTTP/1.1 503)
    19:58:48 W http_client.go:806> [2] Non ok http code 503 (HTTP/1.1 503)
    19:58:48 W http_client.go:806> [2] Non ok http code 503 (HTTP/1.1 503)
    19:58:48 W http_client.go:806> [2] Non ok http code 503 (HTTP/1.1 503)
    19:58:48 W http_client.go:806> [2] Non ok http code 503 (HTTP/1.1 503)
    19:58:48 W http_client.go:806> [1] Non ok http code 503 (HTTP/1.1 503)
    19:58:48 W http_client.go:806> [2] Non ok http code 503 (HTTP/1.1 503)
    19:58:48 W http_client.go:806> [2] Non ok http code 503 (HTTP/1.1 503)
    19:58:48 W http_client.go:806> [1] Non ok http code 503 (HTTP/1.1 503)
    19:58:48 W http_client.go:806> [4] Non ok http code 503 (HTTP/1.1 503)
    19:58:48 W http_client.go:806> [3] Non ok http code 503 (HTTP/1.1 503)
    19:58:48 W http_client.go:806> [3] Non ok http code 503 (HTTP/1.1 503)
    19:58:48 W http_client.go:806> [2] Non ok http code 503 (HTTP/1.1 503)
    19:58:48 W http_client.go:806> [3] Non ok http code 503 (HTTP/1.1 503)
    19:58:48 W http_client.go:806> [2] Non ok http code 503 (HTTP/1.1 503)
    19:58:48 W http_client.go:806> [3] Non ok http code 503 (HTTP/1.1 503)
    19:58:48 W http_client.go:806> [1] Non ok http code 503 (HTTP/1.1 503)
    19:58:48 W http_client.go:806> [3] Non ok http code 503 (HTTP/1.1 503)
    19:58:48 W http_client.go:806> [1] Non ok http code 503 (HTTP/1.1 503)
    19:58:48 W http_client.go:806> [4] Non ok http code 503 (HTTP/1.1 503)
    Ended after 33.1549ms : 50 calls. qps=1508.1
    Aggregated Function Time : count 50 avg 0.002467842 +/- 0.001827 min 0.0003724 max 0.0067697 sum 0.1233921
    # range, mid point, percentile, count
    >= 0.0003724 <= 0.001 , 0.0006862 , 34.00, 17
    > 0.001 <= 0.002 , 0.0015 , 50.00, 8
    > 0.002 <= 0.003 , 0.0025 , 60.00, 5
    > 0.003 <= 0.004 , 0.0035 , 84.00, 12
    > 0.004 <= 0.005 , 0.0045 , 88.00, 2
    > 0.005 <= 0.006 , 0.0055 , 92.00, 2
    > 0.006 <= 0.0067697 , 0.00638485 , 100.00, 4
    # target 50% 0.002
    # target 75% 0.003625
    # target 90% 0.0055
    # target 99% 0.00667349
    # target 99.9% 0.00676008
    Sockets used: 25 (for perfect keepalive, would be 5)
    Jitter: false
    Code 200 : 29 (58.0 %)
    Code 503 : 21 (42.0 %)
    Response Header Sizes : count 50 avg 133.4 +/- 113.5 min 0 max 230 sum 6670
    Response Body/Total Sizes : count 50 avg 368.02 +/- 108.1 min 241 max 460 sum 18401
    All done 50 calls (plus 0 warmup) 2.468 ms avg, 1508.1 qps
    ```

    As seen above, only 58% of the requests succeeded, and the rest failed when the circuit breaker tripped.
    ```
    Code 200 : 29 (58.0 %)
    Code 503 : 21 (42.0 %)
    ```

1. Examine the `Envoy` sidecar stats to see statistics pertaining to the requests that tripped the circuit breaker.
    ```console
    osm proxy get stats $fortio_pod -n client | grep 'httpbin.*pending'
    ```
    The output will be similar to:
    ```console
    cluster.httpbin_httpbin_svc_cluster_local_14001.circuit_breakers.default.remaining_pending: 1
    cluster.httpbin_httpbin_svc_cluster_local_14001.circuit_breakers.default.rq_pending_open: 0
    cluster.httpbin_httpbin_svc_cluster_local_14001.circuit_breakers.high.rq_pending_open: 0
    cluster.httpbin_httpbin_svc_cluster_local_14001.upstream_rq_pending_active: 0
    cluster.httpbin_httpbin_svc_cluster_local_14001.upstream_rq_pending_failure_eject: 0
    cluster.httpbin_httpbin_svc_cluster_local_14001.upstream_rq_pending_overflow: 21
    cluster.httpbin_httpbin_svc_cluster_local_14001.upstream_rq_pending_total: 29
    ```

    `cluster.httpbin_httpbin_svc_cluster_local_14001.upstream_rq_pending_overflow: 21` indicates that 21 requests tripped the circuit breaker, which matches the number of failed requests seen in the previous step: `Code 200 : 29 (58.0 %)`.
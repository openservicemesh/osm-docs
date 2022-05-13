---
title: "Circuit breaking for destinations within the mesh"
description: "Configuring circuit breaking for destinations within the mesh"
type: docs
weight: 21
---

This guide demonstrates how to configure circuit breaking for destinations that are a part of an OSM managed service mesh.

## Prerequisites

- Kubernetes cluster running Kubernetes {{< param min_k8s_version >}} or greater.
- Have OSM installed.
- Have `kubectl` available to interact with the API server.
- Have `osm` CLI available for managing the service mesh.
- OSM version >= v1.1.0.


## Demo

The following demo shows a load-testing client [fortio](https://github.com/fortio/fortio) sending traffic to the `httpbin` service. We will see how applying circuit breakers for traffic to the `httpbin` service impacts the `fortio` client when the configured circuit breaking limits trip.

1. For simplicity, enable [permissive traffic policy mode](/docs/guides/traffic_management/permissive_mode) so that explicit SMI traffic access policies are not required for application connectivity within the mesh.
    ```bash
    export osm_namespace=osm-system # Replace osm-system with the namespace where OSM is installed
    kubectl patch meshconfig osm-mesh-config -n "$osm_namespace" -p '{"spec":{"traffic":{"enablePermissiveTrafficPolicyMode":true}}}'  --type=merge
    ```

1. Deploy the `httpbin` service into the `httpbin` namespace after enrolling its namespace to the mesh. The `httpbin` service runs on port `14001`.

    ```bash
    # Create the httpbin namespace
    kubectl create namespace httpbin

    # Add the namespace to the mesh
    osm namespace add httpbin

    # Deploy httpbin service in the httpbin namespace
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

1. Deploy the `fortio` load-testing client in the `client` namespace after enrolling its namespace to the mesh.
    ```bash
    # Create the client namespace
    kubectl create namespace client

    # Add the namespace to the mesh
    osm namespace add client

    # Deploy fortio client in the client namespace
    kubectl apply -f https://raw.githubusercontent.com/openservicemesh/osm-docs/{{< param osm_branch >}}/manifests/samples/fortio/fortio.yaml -n client
    ```

    Confirm the `fortio` client pod is up and running.

    ```console
    $ kubectl get pods -n client
    NAME                      READY   STATUS    RESTARTS   AGE
    fortio-6477f8495f-bj4s9   2/2     Running   0          19s
    ```

1. Confirm the `fortio` client is able to successfully make HTTP requests to the `httpbin` service on port `14001`. We call the `httpbin` service with `3` concurrent connections (`-c 3`) and send `50` requests (`-n 50`).
    ```console
    $ export fortio_pod="$(kubectl get pod -n client -l app=fortio -o jsonpath='{.items[0].metadata.name}')"

    $ kubectl exec "$fortio_pod" -c fortio -n client -- /usr/bin/fortio load -c 3 -qps 0 -n 50 -loglevel Warning http://httpbin.httpbin.svc.cluster.local:14001/get
    17:48:46 I logger.go:127> Log level is now 3 Warning (was 2 Info)
    Fortio 1.17.1 running at 0 queries per second, 8->8 procs, for 50 calls: http://httpbin.httpbin.svc.cluster.local:14001/get
    Starting at max qps with 3 thread(s) [gomax 8] for exactly 50 calls (16 per thread + 2)
    Ended after 438.1586ms : 50 calls. qps=114.11
    Aggregated Function Time : count 50 avg 0.026068422 +/- 0.05104 min 0.0029766 max 0.1927961 sum 1.3034211
    # range, mid point, percentile, count
    >= 0.0029766 <= 0.003 , 0.0029883 , 2.00, 1
    > 0.003 <= 0.004 , 0.0035 , 30.00, 14
    > 0.004 <= 0.005 , 0.0045 , 32.00, 1
    > 0.005 <= 0.006 , 0.0055 , 44.00, 6
    > 0.006 <= 0.007 , 0.0065 , 46.00, 1
    > 0.007 <= 0.008 , 0.0075 , 66.00, 10
    > 0.008 <= 0.009 , 0.0085 , 72.00, 3
    > 0.009 <= 0.01 , 0.0095 , 74.00, 1
    > 0.01 <= 0.011 , 0.0105 , 82.00, 4
    > 0.03 <= 0.035 , 0.0325 , 86.00, 2
    > 0.035 <= 0.04 , 0.0375 , 88.00, 1
    > 0.12 <= 0.14 , 0.13 , 94.00, 3
    > 0.18 <= 0.192796 , 0.186398 , 100.00, 3
    # target 50% 0.0072
    # target 75% 0.010125
    # target 90% 0.126667
    # target 99% 0.190663
    # target 99.9% 0.192583
    Sockets used: 3 (for perfect keepalive, would be 3)
    Jitter: false
    Code 200 : 50 (100.0 %)
    Response Header Sizes : count 50 avg 230.3 +/- 0.6708 min 230 max 232 sum 11515
    Response Body/Total Sizes : count 50 avg 582.3 +/- 0.6708 min 582 max 584 sum 29115
    All done 50 calls (plus 0 warmup) 26.068 ms avg, 114.1 qps
    ```

    As seen above, all the requests succeeded.
    ```
    Code 200 : 50 (100.0 %)
    ```

1. Next, apply a circuit breaker configuration using the `UpstreamTrafficSetting` resource for traffic directed to the `httpbin` service to limit the maximum number of concurrent connections and requests to `1`.
    > Note: The `UpstreamTrafficSetting` resource must be created in the same namespace as the upstream (destination) service, and the host must be set as the FQDN of the Kubernetes service.

    ```bash
    kubectl apply -f - <<EOF
    apiVersion: policy.openservicemesh.io/v1alpha1
    kind: UpstreamTrafficSetting
    metadata:
      name: httpbin
      namespace: httpbin
    spec:
      host: httpbin.httpbin.svc.cluster.local
      connectionSettings:
        tcp:
          maxConnections: 1
        http:
          maxPendingRequests: 1
          maxRequestsPerConnection: 1
    EOF
    ```

1. Confirm the `fortio` client is unable to make the same amount of successful requests as before due to the connection and request level circuit breaking limits configured above.
    ```console
    $ kubectl exec "$fortio_pod" -c fortio -n client -- /usr/bin/fortio load -c 3 -qps 0 -n 50 -loglevel Warning http://httpbin.httpbin.svc.cluster.local:14001/get
    17:59:19 I logger.go:127> Log level is now 3 Warning (was 2 Info)
    Fortio 1.17.1 running at 0 queries per second, 8->8 procs, for 50 calls: http://httpbin.httpbin.svc.cluster.local:14001/get
    Starting at max qps with 3 thread(s) [gomax 8] for exactly 50 calls (16 per thread + 2)
    17:59:19 W http_client.go:806> [0] Non ok http code 503 (HTTP/1.1 503)
    17:59:19 W http_client.go:806> [1] Non ok http code 503 (HTTP/1.1 503)
    17:59:19 W http_client.go:806> [1] Non ok http code 503 (HTTP/1.1 503)
    17:59:19 W http_client.go:806> [1] Non ok http code 503 (HTTP/1.1 503)
    17:59:19 W http_client.go:806> [0] Non ok http code 503 (HTTP/1.1 503)
    17:59:19 W http_client.go:806> [1] Non ok http code 503 (HTTP/1.1 503)
    17:59:19 W http_client.go:806> [0] Non ok http code 503 (HTTP/1.1 503)
    17:59:19 W http_client.go:806> [0] Non ok http code 503 (HTTP/1.1 503)
    17:59:19 W http_client.go:806> [0] Non ok http code 503 (HTTP/1.1 503)
    17:59:19 W http_client.go:806> [0] Non ok http code 503 (HTTP/1.1 503)
    17:59:19 W http_client.go:806> [0] Non ok http code 503 (HTTP/1.1 503)
    17:59:19 W http_client.go:806> [0] Non ok http code 503 (HTTP/1.1 503)
    17:59:19 W http_client.go:806> [0] Non ok http code 503 (HTTP/1.1 503)
    17:59:19 W http_client.go:806> [2] Non ok http code 503 (HTTP/1.1 503)
    17:59:19 W http_client.go:806> [0] Non ok http code 503 (HTTP/1.1 503)
    17:59:19 W http_client.go:806> [1] Non ok http code 503 (HTTP/1.1 503)
    17:59:19 W http_client.go:806> [1] Non ok http code 503 (HTTP/1.1 503)
    17:59:19 W http_client.go:806> [1] Non ok http code 503 (HTTP/1.1 503)
    17:59:19 W http_client.go:806> [1] Non ok http code 503 (HTTP/1.1 503)
    17:59:19 W http_client.go:806> [2] Non ok http code 503 (HTTP/1.1 503)
    17:59:19 W http_client.go:806> [0] Non ok http code 503 (HTTP/1.1 503)
    17:59:19 W http_client.go:806> [0] Non ok http code 503 (HTTP/1.1 503)
    17:59:19 W http_client.go:806> [0] Non ok http code 503 (HTTP/1.1 503)
    17:59:19 W http_client.go:806> [0] Non ok http code 503 (HTTP/1.1 503)
    17:59:19 W http_client.go:806> [0] Non ok http code 503 (HTTP/1.1 503)
    17:59:19 W http_client.go:806> [0] Non ok http code 503 (HTTP/1.1 503)
    17:59:19 W http_client.go:806> [2] Non ok http code 503 (HTTP/1.1 503)
    17:59:19 W http_client.go:806> [0] Non ok http code 503 (HTTP/1.1 503)
    17:59:19 W http_client.go:806> [1] Non ok http code 503 (HTTP/1.1 503)
    Ended after 122.6576ms : 50 calls. qps=407.64
    Aggregated Function Time : count 50 avg 0.006086436 +/- 0.00731 min 0.0005739 max 0.042604 sum 0.3043218
    # range, mid point, percentile, count
    >= 0.0005739 <= 0.001 , 0.00078695 , 14.00, 7
    > 0.001 <= 0.002 , 0.0015 , 32.00, 9
    > 0.002 <= 0.003 , 0.0025 , 40.00, 4
    > 0.003 <= 0.004 , 0.0035 , 52.00, 6
    > 0.004 <= 0.005 , 0.0045 , 64.00, 6
    > 0.005 <= 0.006 , 0.0055 , 66.00, 1
    > 0.006 <= 0.007 , 0.0065 , 72.00, 3
    > 0.007 <= 0.008 , 0.0075 , 74.00, 1
    > 0.008 <= 0.009 , 0.0085 , 76.00, 1
    > 0.009 <= 0.01 , 0.0095 , 80.00, 2
    > 0.01 <= 0.011 , 0.0105 , 82.00, 1
    > 0.011 <= 0.012 , 0.0115 , 88.00, 3
    > 0.012 <= 0.014 , 0.013 , 92.00, 2
    > 0.014 <= 0.016 , 0.015 , 96.00, 2
    > 0.025 <= 0.03 , 0.0275 , 98.00, 1
    > 0.04 <= 0.042604 , 0.041302 , 100.00, 1
    # target 50% 0.00383333
    # target 75% 0.0085
    # target 90% 0.013
    # target 99% 0.041302
    # target 99.9% 0.0424738
    Sockets used: 31 (for perfect keepalive, would be 3)
    Jitter: false
    Code 200 : 21 (42.0 %)
    Code 503 : 29 (58.0 %)
    Response Header Sizes : count 50 avg 96.68 +/- 113.6 min 0 max 231 sum 4834
    Response Body/Total Sizes : count 50 avg 399.42 +/- 186.2 min 241 max 619 sum 19971
    All done 50 calls (plus 0 warmup) 6.086 ms avg, 407.6 qps
    ```

    As seen above, only 42% of the requests succeeded, and the rest failed when the circuit breaker tripped.
    ```
    Code 200 : 21 (42.0 %)
    Code 503 : 29 (58.0 %)
    ```

1. Examine the `Envoy` sidecar stats to see statistics pertaining to the requests that tripped the circuit breaker.
    ```console
    $ osm proxy get stats "$fortio_pod" -n client | grep 'httpbin.*pending'
    cluster.httpbin/httpbin|14001.circuit_breakers.default.remaining_pending: 1
    cluster.httpbin/httpbin|14001.circuit_breakers.default.rq_pending_open: 0
    cluster.httpbin/httpbin|14001.circuit_breakers.high.rq_pending_open: 0
    cluster.httpbin/httpbin|14001.upstream_rq_pending_active: 0
    cluster.httpbin/httpbin|14001.upstream_rq_pending_failure_eject: 0
    cluster.httpbin/httpbin|14001.upstream_rq_pending_overflow: 29
    cluster.httpbin/httpbin|14001.upstream_rq_pending_total: 25
    ```

    `cluster.httpbin/httpbin|14001.upstream_rq_pending_overflow: 29` indicates that 29 requests tripped the circuit breaker, which matches the number of failed requests seen in the previous step: `Code 503 : 29 (58.0 %)`.
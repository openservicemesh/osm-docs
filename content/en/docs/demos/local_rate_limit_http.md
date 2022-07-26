---
title: "Local rate limiting of HTTP requests"
description: "Configuring local rate limiting for HTTP requests"
type: docs
weight: 22
---

This guide demonstrates how to configure local rate limiting for HTTP requests destined to a target host that is a part of an OSM managed service mesh.

## Prerequisites

- Kubernetes cluster running Kubernetes {{< param min_k8s_version >}} or greater.
- Have OSM installed.
- Have `kubectl` available to interact with the API server.
- Have `osm` CLI available for managing the service mesh.
- OSM version >= v1.2.0.


## Demo

The following demo shows a client sending HTTP requests to the `fortio` service. We will see the impact of applying local HTTP rate limiting policies targeting the `fortio` service to control the throughput of requests destined to the service backend.

1. For simplicity, enable [permissive traffic policy mode](/docs/guides/traffic_management/permissive_mode) so that explicit SMI traffic access policies are not required for application connectivity within the mesh.
    ```bash
    export osm_namespace=osm-system # Replace osm-system with the namespace where OSM is installed
    kubectl patch meshconfig osm-mesh-config -n "$osm_namespace" -p '{"spec":{"traffic":{"enablePermissiveTrafficPolicyMode":true}}}'  --type=merge
    ```

1. Deploy the `fortio` HTTP service in the `demo` namespace after enrolling its namespace to the mesh. The `fortio` HTTP service runs on port `8080`.
    ```bash
    # Create the demo namespace
    kubectl create namespace demo

    # Add the namespace to the mesh
    osm namespace add demo

    # Deploy fortio TCP echo in the demo namespace
    kubectl apply -f https://raw.githubusercontent.com/openservicemesh/osm-docs/{{< param osm_branch >}}/manifests/samples/fortio/fortio.yaml -n demo
    ```

    Confirm the `fortio` service pod is up and running.

    ```console
    $ kubectl get pods -n demo
    NAME                            READY   STATUS    RESTARTS   AGE
    fortio-c4bd7857f-7mm6w          2/2     Running   0          22m
    ```

1. Deploy the `fortio-client` app in the `demo` namespace. We will use this client to send TCP traffic to the `fortio TCP echo` service deployed previously.
    ```bash
    kubectl apply -f https://raw.githubusercontent.com/openservicemesh/osm-docs/{{< param osm_branch >}}/manifests/samples/fortio/fortio-client.yaml -n demo
    ```

    Confirm the `fortio-client` pod is up and running.

    ```console
    $ kubectl get pods -n demo
    NAME                            READY   STATUS    RESTARTS   AGE
    fortio-client-b9b7bbfb8-prq7r   2/2     Running   0          7s
    ```

1. Confirm the `fortio-client` app is able to successfully make HTTP requests to the `fortio` HTTP service on port `8080`. We call the `fortio` service with `3` concurrent connections (`-c 3`) and send `10` requests (`-n 10`).
    ```console
    $ fortio_client="$(kubectl get pod -n demo -l app=fortio-client -o jsonpath='{.items[0].metadata.name}')"

    $ kubectl exec "$fortio_client" -n demo -c fortio-client -- fortio load -c 3 -n 10 http://fortio.demo.svc.cluster.local:8080
    Fortio 1.33.0 running at 8 queries per second, 8->8 procs, for 10 calls: http://fortio.demo.svc.cluster.local:8080
    20:58:07 I httprunner.go:93> Starting http test for http://fortio.demo.svc.cluster.local:8080 with 3 threads at 8.0 qps and parallel warmup
    Starting at 8 qps with 3 thread(s) [gomax 8] : exactly 10, 3 calls each (total 9 + 1)
    20:58:08 I periodic.go:723> T002 ended after 1.1273523s : 3 calls. qps=2.661102478790348
    20:58:08 I periodic.go:723> T001 ended after 1.1273756s : 3 calls. qps=2.661047480537986
    20:58:08 I periodic.go:723> T000 ended after 1.5023464s : 4 calls. qps=2.662501803844972
    Ended after 1.5024079s : 10 calls. qps=6.656
    Sleep times : count 7 avg 0.52874391 +/- 0.03031 min 0.4865562 max 0.5604152 sum 3.7012074
    Aggregated Function Time : count 10 avg 0.0050187 +/- 0.005515 min 0.0012575 max 0.0135401 sum 0.050187
    # range, mid point, percentile, count
    >= 0.0012575 <= 0.002 , 0.00162875 , 70.00, 7
    > 0.012 <= 0.0135401 , 0.01277 , 100.00, 3
    # target 50% 0.0017525
    # target 75% 0.0122567
    # target 90% 0.0130267
    # target 99% 0.0134888
    # target 99.9% 0.013535
    Error cases : no data
    20:58:08 I httprunner.go:190> [0] fortio.demo.svc.cluster.local:8080 resolved to 10.96.189.159:8080
    20:58:08 I httprunner.go:190> [1] fortio.demo.svc.cluster.local:8080 resolved to 10.96.189.159:8080
    20:58:08 I httprunner.go:190> [2] fortio.demo.svc.cluster.local:8080 resolved to 10.96.189.159:8080
    Sockets used: 3 (for perfect keepalive, would be 3)
    Uniform: false, Jitter: false
    IP addresses distribution:
    10.96.189.159:8080: 3
    Code 200 : 10 (100.0 %)
    Response Header Sizes : count 10 avg 124.3 +/- 0.4583 min 124 max 125 sum 1243
    Response Body/Total Sizes : count 10 avg 124.3 +/- 0.4583 min 124 max 125 sum 1243
    All done 10 calls (plus 0 warmup) 5.019 ms avg, 6.7 qps
    ```

    As seen above, all the HTTP requests from the `fortio-client` pod succeeded.
    ```
    Code 200 : 10 (100.0 %)
    ```

1. Next, apply a local rate limiting policy to rate limit HTTP requests at the virtual host level to `3 requests per minute`.
    ```bash
    kubectl apply -f - <<EOF
    apiVersion: policy.openservicemesh.io/v1alpha1
    kind: UpstreamTrafficSetting
    metadata:
      name: http-rate-limit
      namespace: demo
    spec:
      host: fortio.demo.svc.cluster.local
      rateLimit:
        local:
          http:
            requests: 3
            unit: minute
    EOF
    ```

    Confirm no HTTP requests have been rate limited yet by examining the stats on the `fortio` backend pod.
    ```console
    $ fortio_server="$(kubectl get pod -n demo -l app=fortio -o jsonpath='{.items[0].metadata.name}')"

    $ osm proxy get stats "$fortio_server" -n demo | grep 'http_local_rate_limiter.http_local_rate_limit.rate_limited'
    http_local_rate_limiter.http_local_rate_limit.rate_limited: 0
    ```

1. Confirm HTTP requests are rate limited.
    ```console
    $ kubectl exec "$fortio_client" -n demo -c fortio-client -- fortio load -c 3 -n 10 http://fortio.demo.svc.cluster.local:8080
    Fortio 1.33.0 running at 8 queries per second, 8->8 procs, for 10 calls: http://fortio.demo.svc.cluster.local:8080
    21:06:36 I httprunner.go:93> Starting http test for http://fortio.demo.svc.cluster.local:8080 with 3 threads at 8.0 qps and parallel warmup
    Starting at 8 qps with 3 thread(s) [gomax 8] : exactly 10, 3 calls each (total 9 + 1)
    21:06:37 W http_client.go:838> [0] Non ok http code 429 (HTTP/1.1 429)
    21:06:37 W http_client.go:838> [1] Non ok http code 429 (HTTP/1.1 429)
    21:06:37 W http_client.go:838> [2] Non ok http code 429 (HTTP/1.1 429)
    21:06:37 W http_client.go:838> [0] Non ok http code 429 (HTTP/1.1 429)
    21:06:37 W http_client.go:838> [1] Non ok http code 429 (HTTP/1.1 429)
    21:06:37 I periodic.go:723> T001 ended after 1.1269827s : 3 calls. qps=2.661975201571417
    21:06:37 W http_client.go:838> [2] Non ok http code 429 (HTTP/1.1 429)
    21:06:37 I periodic.go:723> T002 ended after 1.1271942s : 3 calls. qps=2.66147572441377
    21:06:38 W http_client.go:838> [0] Non ok http code 429 (HTTP/1.1 429)
    21:06:38 I periodic.go:723> T000 ended after 1.5021191s : 4 calls. qps=2.662904692444161
    Ended after 1.5021609s : 10 calls. qps=6.6571
    Sleep times : count 7 avg 0.53138026 +/- 0.03038 min 0.4943128 max 0.5602373 sum 3.7196618
    Aggregated Function Time : count 10 avg 0.00318326 +/- 0.002431 min 0.0012651 max 0.0077951 sum 0.0318326
    # range, mid point, percentile, count
    >= 0.0012651 <= 0.002 , 0.00163255 , 60.00, 6
    > 0.002 <= 0.003 , 0.0025 , 70.00, 1
    > 0.005 <= 0.006 , 0.0055 , 80.00, 1
    > 0.006 <= 0.007 , 0.0065 , 90.00, 1
    > 0.007 <= 0.0077951 , 0.00739755 , 100.00, 1
    # target 50% 0.00185302
    # target 75% 0.0055
    # target 90% 0.007
    # target 99% 0.00771559
    # target 99.9% 0.00778715
    Error cases : count 7 avg 0.0016392143 +/- 0.000383 min 0.0012651 max 0.0023951 sum 0.0114745
    # range, mid point, percentile, count
    >= 0.0012651 <= 0.002 , 0.00163255 , 85.71, 6
    > 0.002 <= 0.0023951 , 0.00219755 , 100.00, 1
    # target 50% 0.00163255
    # target 75% 0.00188977
    # target 90% 0.00211853
    # target 99% 0.00236744
    # target 99.9% 0.00239233
    21:06:38 I httprunner.go:190> [0] fortio.demo.svc.cluster.local:8080 resolved to 10.96.189.159:8080
    21:06:38 I httprunner.go:190> [1] fortio.demo.svc.cluster.local:8080 resolved to 10.96.189.159:8080
    21:06:38 I httprunner.go:190> [2] fortio.demo.svc.cluster.local:8080 resolved to 10.96.189.159:8080
    Sockets used: 7 (for perfect keepalive, would be 3)
    Uniform: false, Jitter: false
    IP addresses distribution:
    10.96.189.159:8080: 3
    Code 200 : 3 (30.0 %)
    Code 429 : 7 (70.0 %)
    Response Header Sizes : count 10 avg 37.2 +/- 56.82 min 0 max 124 sum 372
    Response Body/Total Sizes : count 10 avg 166 +/- 27.5 min 124 max 184 sum 1660
    All done 10 calls (plus 0 warmup) 3.183 ms avg, 6.7 qps
    ```

    As seen above, only `3` out of `10` HTTP requests succeeded, while the remaining `7` requests were rate limited as per the rate limiting policy.
    ```
    Code 200 : 3 (30.0 %)
    Code 429 : 7 (70.0 %)
    ```

    Examine the stats to further confirm this.
    ```console
    $ osm proxy get stats "$fortio_server" -n demo | grep 'http_local_rate_limiter.http_local_rate_limit.rate_limited'
    http_local_rate_limiter.http_local_rate_limit.rate_limited: 7
    ```

1. Next, let's update our rate limiting policy to allow a burst of requests. Bursts allow a given number of requests over the baseline rate of 3 requests per minute defined by our rate limiting policy.
    ```bash
    kubectl apply -f - <<EOF
    apiVersion: policy.openservicemesh.io/v1alpha1
    kind: UpstreamTrafficSetting
    metadata:
      name: http-rate-limit
      namespace: demo
    spec:
      host: fortio.demo.svc.cluster.local
      rateLimit:
        local:
          http:
            requests: 3
            unit: minute
            burst: 10
    EOF
    ```

1. Confirm the burst capability allows a burst of requests within a small window of time.
    ```console
    $ kubectl exec "$fortio_client" -n demo -c fortio-client -- fortio load -c 3 -n 10 http://fortio.demo.svc.cluster.local:8080
    Fortio 1.33.0 running at 8 queries per second, 8->8 procs, for 10 calls: http://fortio.demo.svc.cluster.local:8080
    21:11:04 I httprunner.go:93> Starting http test for http://fortio.demo.svc.cluster.local:8080 with 3 threads at 8.0 qps and parallel warmup
    Starting at 8 qps with 3 thread(s) [gomax 8] : exactly 10, 3 calls each (total 9 + 1)
    21:11:05 I periodic.go:723> T002 ended after 1.127252s : 3 calls. qps=2.6613392568831107
    21:11:05 I periodic.go:723> T001 ended after 1.1273028s : 3 calls. qps=2.661219328116634
    21:11:05 I periodic.go:723> T000 ended after 1.5019947s : 4 calls. qps=2.663125242718899
    Ended after 1.5020768s : 10 calls. qps=6.6574
    Sleep times : count 7 avg 0.53158916 +/- 0.03008 min 0.4943959 max 0.5600713 sum 3.7211241
    Aggregated Function Time : count 10 avg 0.00318637 +/- 0.002356 min 0.0012401 max 0.0073302 sum 0.0318637
    # range, mid point, percentile, count
    >= 0.0012401 <= 0.002 , 0.00162005 , 60.00, 6
    > 0.002 <= 0.003 , 0.0025 , 70.00, 1
    > 0.005 <= 0.006 , 0.0055 , 80.00, 1
    > 0.007 <= 0.0073302 , 0.0071651 , 100.00, 2
    # target 50% 0.00184802
    # target 75% 0.0055
    # target 90% 0.0071651
    # target 99% 0.00731369
    # target 99.9% 0.00732855
    Error cases : no data
    21:11:05 I httprunner.go:190> [0] fortio.demo.svc.cluster.local:8080 resolved to 10.96.189.159:8080
    21:11:05 I httprunner.go:190> [1] fortio.demo.svc.cluster.local:8080 resolved to 10.96.189.159:8080
    21:11:05 I httprunner.go:190> [2] fortio.demo.svc.cluster.local:8080 resolved to 10.96.189.159:8080
    Sockets used: 3 (for perfect keepalive, would be 3)
    Uniform: false, Jitter: false
    IP addresses distribution:
    10.96.189.159:8080: 3
    Code 200 : 10 (100.0 %)
    Response Header Sizes : count 10 avg 124 +/- 0 min 124 max 124 sum 1240
    Response Body/Total Sizes : count 10 avg 124 +/- 0 min 124 max 124 sum 1240
    All done 10 calls (plus 0 warmup) 3.186 ms avg, 6.7 qps
    ```

    As seen above, all HTTP requests succeeded as we allowed a burst of 10 requests with our rate limiting policy.
    ```
    Code 200 : 10 (100.0 %)
    ```

    Further, examine the stats to confirm the burst allows additional requests to go through. The number of requests rate limited hasn't increased since our previous rate limit test before we configured the burst setting.
    ```console
    $ osm proxy get stats "$fortio_server" -n demo | grep 'http_local_rate_limiter.http_local_rate_limit.rate_limited'
    http_local_rate_limiter.http_local_rate_limit.rate_limited: 7
    ```

1. Next, let's configure the rate limting policy for a specific HTTP route allowed on the upstream service.
    > Note: Since we are using permissive traffic policy mode in the demo, an HTTP route with a wildcard path regex `.*` is allowed on the upstream backend, so we will configure a rate limiting policy for this route. However, when using SMI policies in the mesh, paths corresponding to matching allowed SMI HTTP routing rules can be configured.

    ```bash
    kubectl apply -f - <<EOF
    apiVersion: policy.openservicemesh.io/v1alpha1
    kind: UpstreamTrafficSetting
    metadata:
      name: http-rate-limit
      namespace: demo
    spec:
      host: fortio.demo.svc.cluster.local
      httpRoutes:
        - path: .*
          rateLimit:
            local:
              requests: 3
              unit: minute
    EOF
    ```

1. Confirm HTTP requests are rate limited at a per-route level.
    ```console
    $ kubectl exec "$fortio_client" -n demo -c fortio-client -- fortio load -c 3 -n 10 http://fortio.demo.svc.cluster.local:8080
    Fortio 1.33.0 running at 8 queries per second, 8->8 procs, for 10 calls: http://fortio.demo.svc.cluster.local:8080
    21:19:34 I httprunner.go:93> Starting http test for http://fortio.demo.svc.cluster.local:8080 with 3 threads at 8.0 qps and parallel warmup
    Starting at 8 qps with 3 thread(s) [gomax 8] : exactly 10, 3 calls each (total 9 + 1)
    21:19:35 W http_client.go:838> [0] Non ok http code 429 (HTTP/1.1 429)
    21:19:35 W http_client.go:838> [2] Non ok http code 429 (HTTP/1.1 429)
    21:19:35 W http_client.go:838> [1] Non ok http code 429 (HTTP/1.1 429)
    21:19:35 W http_client.go:838> [0] Non ok http code 429 (HTTP/1.1 429)
    21:19:35 W http_client.go:838> [1] Non ok http code 429 (HTTP/1.1 429)
    21:19:35 W http_client.go:838> [2] Non ok http code 429 (HTTP/1.1 429)
    21:19:35 I periodic.go:723> T001 ended after 1.126703s : 3 calls. qps=2.6626360274180505
    21:19:35 I periodic.go:723> T002 ended after 1.1267472s : 3 calls. qps=2.6625315776245104
    21:19:36 W http_client.go:838> [0] Non ok http code 429 (HTTP/1.1 429)
    21:19:36 I periodic.go:723> T000 ended after 1.5027817s : 4 calls. qps=2.6617305760377574
    Ended after 1.5028359s : 10 calls. qps=6.6541
    Sleep times : count 7 avg 0.53089959 +/- 0.03079 min 0.4903791 max 0.5604715 sum 3.7162971
    Aggregated Function Time : count 10 avg 0.00369734 +/- 0.003165 min 0.0011174 max 0.0095033 sum 0.0369734
    # range, mid point, percentile, count
    >= 0.0011174 <= 0.002 , 0.0015587 , 60.00, 6
    > 0.002 <= 0.003 , 0.0025 , 70.00, 1
    > 0.007 <= 0.008 , 0.0075 , 90.00, 2
    > 0.009 <= 0.0095033 , 0.00925165 , 100.00, 1
    # target 50% 0.00182348
    # target 75% 0.00725
    # target 90% 0.008
    # target 99% 0.00945297
    # target 99.9% 0.00949827
    Error cases : count 7 avg 0.0016556 +/- 0.0004249 min 0.0011174 max 0.0025594 sum 0.0115892
    # range, mid point, percentile, count
    >= 0.0011174 <= 0.002 , 0.0015587 , 85.71, 6
    > 0.002 <= 0.0025594 , 0.0022797 , 100.00, 1
    # target 50% 0.0015587
    # target 75% 0.00186761
    # target 90% 0.00216782
    # target 99% 0.00252024
    # target 99.9% 0.00255548
    21:19:36 I httprunner.go:190> [0] fortio.demo.svc.cluster.local:8080 resolved to 10.96.189.159:8080
    21:19:36 I httprunner.go:190> [1] fortio.demo.svc.cluster.local:8080 resolved to 10.96.189.159:8080
    21:19:36 I httprunner.go:190> [2] fortio.demo.svc.cluster.local:8080 resolved to 10.96.189.159:8080
    Sockets used: 7 (for perfect keepalive, would be 3)
    Uniform: false, Jitter: false
    IP addresses distribution:
    10.96.189.159:8080: 3
    Code 200 : 3 (30.0 %)
    Code 429 : 7 (70.0 %)
    Response Header Sizes : count 10 avg 37.2 +/- 56.82 min 0 max 124 sum 372
    Response Body/Total Sizes : count 10 avg 166 +/- 27.5 min 124 max 184 sum 1660
    All done 10 calls (plus 0 warmup) 3.697 ms avg, 6.7 qps
    ```

    As seen above, only `3` out of `10` HTTP requests succeeded, while the remaining `7` requests were rate limited as per the rate limiting policy.
    ```
    Code 200 : 3 (30.0 %)
    Code 429 : 7 (70.0 %)
    ```

    Examine the stats to further confirm this. `7` additional requests have been rate limited after configuring HTTP route level rate limiting since our previous test, indicated by the total of `14` HTTP requests rate limited in the stats.
    ```console
    $ osm proxy get stats "$fortio_server" -n demo | grep 'http_local_rate_limiter.http_local_rate_limit.rate_limited'
    http_local_rate_limiter.http_local_rate_limit.rate_limited: 14
    ```
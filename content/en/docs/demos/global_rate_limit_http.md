---
title: "Global rate limiting of HTTP requests"
description: "Configuring global rate limiting for HTTP requests"
type: docs
weight: 23
---

This guide demonstrates how to configure global rate limiting for HTTP requests destined to a target host that is a part of an OSM managed service mesh.

## Prerequisites

- Kubernetes cluster running Kubernetes {{< param min_k8s_version >}} or greater.
- Have OSM installed.
- Have `kubectl` available to interact with the API server.
- Have `osm` CLI available for managing the service mesh.
- OSM version >= v1.3.0.


## Demo

The following demo shows a client [fortio-client](https://github.com/fortio/fortio) sending HTTP requests to the `fortio` HTTP service. We will see the impact of applying global HTTP rate limiting policies targeting the `fortio` service to control the throughput of traffic destined to the service backend using an external Rate Limit Service (RLS).

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

    # Deploy fortio HTTP in the demo namespace
    kubectl apply -f https://raw.githubusercontent.com/openservicemesh/osm-docs/{{< param osm_branch >}}/manifests/samples/fortio/fortio.yaml -n demo
    ```

    Confirm the `fortio` service pod is up and running.

    ```console
    $ kubectl get pods -n demo
    NAME                            READY   STATUS    RESTARTS   AGE
    fortio-c4bd7857f-7mm6w          2/2     Running   0          22m
    ```

1. Deploy the `fortio-client` app in the `demo` namespace. We will use this client to send TCP traffic to the `fortio HTTP` service deployed previously.
    ```bash
    kubectl apply -f https://raw.githubusercontent.com/openservicemesh/osm-docs/{{< param osm_branch >}}/manifests/samples/fortio/fortio-client.yaml -n demo
    ```

    Confirm the `fortio-client` pod is up and running.

    ```console
    NAME                            READY   STATUS    RESTARTS   AGE
    fortio-client-b9b7bbfb8-prq7r   2/2     Running   0          7s
    ```

1. Deploy the external RLS. Any RLS that implement's [Envoy's Rate Limit Service proto](https://www.envoyproxy.io/docs/envoy/latest/api-v3/service/ratelimit/v3/rls.proto) can be used as a global rate limiter with Envoy. For this demo, we use the [Envoy Rate Limit Service](https://github.com/envoyproxy/ratelimit) as our RLS. Per the [Envoy RLS overview](https://github.com/envoyproxy/ratelimit#overview):
    > The rate limit service is a Go/gRPC service designed to enable generic rate limit scenarios from different types of applications. Applications request a rate limit decision based on a domain and a set of descriptors. The service reads the configuration from disk via runtime, composes a cache key, and talks to the Redis cache. A decision is then returned to the caller.

    Create a namespace to deploy the RLS into.
    ```bash
    kubectl create namespace rls
    ```

    Create a ConfigMap that contains the [rate limit configuration](https://github.com/envoyproxy/ratelimit#configuration).
    ```bash
    kubectl apply -f - <<EOF
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: ratelimit-config
      namespace: rls
    data:
      ratelimit-config.yaml: |
        domain: test
        descriptors:

          # requests with a descriptor ["generic_key": "my_value"]
          # are limited to one per minute.
          - key: generic_key
            value: my_value
            rate_limit:
              unit: minute
              requests_per_unit: 1

          # each unique remote (client) address is limited to 3 per minute
          - key: remote_address
            rate_limit:
              unit: minute
              requests_per_unit: 3

          # requests with the header 'my-header: foo' are rate limited to 5 per minute
          - key: my_header
            value: foo
            rate_limit:
              unit: minute
              requests_per_unit: 5

          # requests with the descriptor 'header_match: foo' are rate limited
          # to 7 per minute
          - key: header_match
            value: foo
            rate_limit:
              unit: minute
              requests_per_unit: 7
    EOF
    ```
    > Note: it can take a few seconds for the RLS to load the configuration.

    Deploy the global RLS Service and Deployment in the `rls` namespace. The `ratelimiter` Deployment mounts the `ratelimit-config` ConfigMap as a volume, which allows the RLS service to load the rate limit configuration specified in the ConfigMap. We will later reference the `ratelimiter` service to configure global rate limiting within the mesh.
    ```bash
    kubectl apply -f - <<EOF
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      labels:
        app: ratelimiter
      name: ratelimiter
      namespace: rls
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: ratelimiter
      template:
        metadata:
          labels:
            app: ratelimiter
        spec:
          containers:
            - name: redis
              image: redis:alpine
              env:
                - name: REDIS_SOCKET_TYPE
                  value: tcp
                - name: REDIS_URL
                  value: redis:6379
            - name: ratelimiter
              image: docker.io/envoyproxy/ratelimit:1f4ea68e
              ports:
                - containerPort: 8080
                  name: http
                  protocol: TCP
                - containerPort: 8081
                  name: grpc
                  protocol: TCP
              volumeMounts:
                - name: ratelimit-config
                  mountPath: /data/ratelimit/config
                  readOnly: true
              env:
                - name: USE_STATSD
                  value: "false"
                - name: LOG_LEVEL
                  value: debug
                - name: REDIS_SOCKET_TYPE
                  value: tcp
                - name: REDIS_URL
                  value: localhost:6379
                - name: RUNTIME_ROOT
                  value: /data
                - name: RUNTIME_SUBDIRECTORY
                  value: ratelimit
                - name: RUNTIME_WATCH_ROOT
                  value: "false"
                # need to set RUNTIME_IGNOREDOTFILES to true to avoid issues with
                # how Kubernetes mounts configmaps into pods.
                - name: RUNTIME_IGNOREDOTFILES
                  value: "true"
              command: ["/bin/ratelimit"]
              livenessProbe:
                httpGet:
                  path: /healthcheck
                  port: 8080
                initialDelaySeconds: 5
                periodSeconds: 5
          volumes:
            - name: ratelimit-config
              configMap:
                name: ratelimit-config
    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: ratelimiter
      namespace: rls
    spec:
      ports:
      - port: 8081
        name: grpc
        protocol: TCP
      selector:
        app: ratelimiter
      type: ClusterIP
    EOF
    ```

    Confirm the RLS pod is up and running.
    ```console
    $ kubectl get pods -n rls
    NAME                          READY   STATUS    RESTARTS   AGE
    ratelimiter-bb7665d55-6qtvv   2/2     Running   0          15s
    ```

1. Confirm the `fortio-client` app is able to successfully make HTTP requests to the `fortio` HTTP service on port `8080`. We call the `fortio` service with `3` concurrent connections (`-c 3`) and send `10` requests (`-n 10`).
    First, save the name of the fortio client and server pods:
    ```bash
    fortio_client="$(kubectl get pod -n demo -l app=fortio-client -o jsonpath='{.items[0].metadata.name}')"
    fortio_server="$(kubectl get pod -n demo -l app=fortio -o jsonpath='{.items[0].metadata.name}')"
    ```

    ```console
    $ kubectl exec "$fortio_client" -n demo -c fortio-client -- fortio load -c 3 -n 10 http://fortio.demo.svc.cluster.local:8080
    Fortio 1.34.1 running at 8 queries per second, 8->8 procs, for 10 calls: http://fortio.demo.svc.cluster.local:8080
    19:19:26 I httprunner.go:98> Starting http test for http://fortio.demo.svc.cluster.local:8080 with 3 threads at 8.0 qps and parallel warmup
    Starting at 8 qps with 3 thread(s) [gomax 8] : exactly 10, 3 calls each (total 9 + 1)
    19:19:27 I periodic.go:721> T001 ended after 1.1276393s : 3 calls. qps=2.6604251909276306
    19:19:27 I periodic.go:721> T002 ended after 1.1276668s : 3 calls. qps=2.6603603121063775
    19:19:27 I periodic.go:721> T000 ended after 1.502888s : 4 calls. qps=2.6615423105381106
    Ended after 1.5030055s : 10 calls. qps=6.6533
    Sleep times : count 7 avg 0.53181499 +/- 0.03017 min 0.4955405 max 0.5605537 sum 3.7227049
    Aggregated Function Time : count 10 avg 0.00328438 +/- 0.002132 min 0.0012625 max 0.0075019 sum 0.0328438
    # range, mid point, percentile, count
    >= 0.0012625 <= 0.002 , 0.00163125 , 30.00, 3
    > 0.002 <= 0.003 , 0.0025 , 70.00, 4
    > 0.004 <= 0.005 , 0.0045 , 80.00, 1
    > 0.006 <= 0.007 , 0.0065 , 90.00, 1
    > 0.007 <= 0.0075019 , 0.00725095 , 100.00, 1
    # target 50% 0.0025
    # target 75% 0.0045
    # target 90% 0.007
    # target 99% 0.00745171
    # target 99.9% 0.00749688
    Error cases : no data
    19:19:27 I httprunner.go:197> [0]   1 socket used, resolved to 10.96.151.249:8080
    19:19:27 I httprunner.go:197> [1]   1 socket used, resolved to 10.96.151.249:8080
    19:19:27 I httprunner.go:197> [2]   1 socket used, resolved to 10.96.151.249:8080
    Sockets used: 3 (for perfect keepalive, would be 3)
    Uniform: false, Jitter: false
    IP addresses distribution:
    10.96.151.249:8080: 3
    Code 200 : 10 (100.0 %)
    Response Header Sizes : count 10 avg 124 +/- 0 min 124 max 124 sum 1240
    Response Body/Total Sizes : count 10 avg 124 +/- 0 min 124 max 124 sum 1240
    All done 10 calls (plus 0 warmup) 3.284 ms avg, 6.7 qps
    ```

    As seen above, all the HTTP requests from the `fortio-client` pod succeeded.
    ```
    Code 200 : 10 (100.0 %)
    ```

### Configuring global rate limit policies

Global rate limit policies can be applied to target upstream services in the cluster. The policy can be applied at both the virtual host and route level. The policy defines a set of request descriptors that will be generated and sent to the external RLS to make a rate limiting decision on each request.

Refer to the the [HTTP global rate limiting guide](/docs/guides/traffic_management/rate_limiting/#global-rate-limiting-of-http-requests) to learn more about the configuration attributes.


In this demo, we will see the impact of generating different descriptor entries on rate limiting.

#### genericKey descriptor

The `genericKey` descriptor entry defines a static key-value pair. By default, the `genericKey` descriptor entry uses `generic_key` as it's descriptor key if unspecified.

Recollect that we defined a rate limiting policy in the `ratelimit-config` ConfigMap to rate limit requests that generate the descriptor `generic_key: my_value` to 1 per minute:
```
descriptors:
  # requests with a descriptor ["generic_key": "my_value"]
  # are limited to 1 per minute.
  - key: generic_key
    value: my_value
    rate_limit:
      unit: minute
      requests_per_unit: 1
```

Let's apply a global rate limit policy targeting the `fortio` service to generate the descriptor entry `generic_key: my_value`.

```bash
kubectl apply -f - <<EOF
apiVersion: policy.openservicemesh.io/v1alpha1
kind: UpstreamTrafficSetting
metadata:
  name: fortio-tcp
  namespace: demo
spec:
  host: fortio.demo.svc.cluster.local
  rateLimit:
    global:
      http:
        rateLimitService:
          host: ratelimiter.rls.svc.cluster.local
          port: 8081
        domain: test
        failOpen: false
        timeout: 10s
        descriptors:
          - entries:
            - genericKey:
                # key: generic_key # default
                value: my_value
EOF
```

Confirm the HTTP requests from the `fortio-client` to the `fortio` service are rate limited.

Reset the stat counters on the `fortio` server pod's Envoy sidecar to see the impact of rate limiting.
```bash
osm proxy set reset_counters "$fortio_server" -n demo
```

```console
$ kubectl exec "$fortio_client" -n demo -c fortio-client -- fortio load -c 3 -n 10 http://fortio.demo.svc.cluster.local:8080
Fortio 1.34.1 running at 8 queries per second, 8->8 procs, for 10 calls: http://fortio.demo.svc.cluster.local:8080
20:41:02 I httprunner.go:98> Starting http test for http://fortio.demo.svc.cluster.local:8080 with 3 threads at 8.0 qps and parallel warmup
Starting at 8 qps with 3 thread(s) [gomax 8] : exactly 10, 3 calls each (total 9 + 1)
20:41:02 W http_client.go:889> [1] Non ok http code 429 (HTTP/1.1 429)
20:41:02 W http_client.go:889> [0] Non ok http code 429 (HTTP/1.1 429)
20:41:03 W http_client.go:889> [0] Non ok http code 429 (HTTP/1.1 429)
20:41:03 W http_client.go:889> [1] Non ok http code 429 (HTTP/1.1 429)
20:41:03 W http_client.go:889> [2] Non ok http code 429 (HTTP/1.1 429)
20:41:03 W http_client.go:889> [0] Non ok http code 429 (HTTP/1.1 429)
20:41:03 W http_client.go:889> [1] Non ok http code 429 (HTTP/1.1 429)
20:41:03 I periodic.go:721> T001 ended after 1.1319728s : 3 calls. qps=2.650240359132304
20:41:03 W http_client.go:889> [2] Non ok http code 429 (HTTP/1.1 429)
20:41:03 I periodic.go:721> T002 ended after 1.1405133s : 3 calls. qps=2.630394577599402
20:41:04 W http_client.go:889> [0] Non ok http code 429 (HTTP/1.1 429)
20:41:04 I periodic.go:721> T000 ended after 1.5124383s : 4 calls. qps=2.644735986915962
Ended after 1.5125624s : 10 calls. qps=6.6113
Sleep times : count 7 avg 0.50386114 +/- 0.03724 min 0.4420604 max 0.5548546 sum 3.527028
Aggregated Function Time : count 10 avg 0.02527542 +/- 0.02166 min 0.0063666 max 0.0589057 sum 0.2527542
# range, mid point, percentile, count
>= 0.0063666 <= 0.007 , 0.0066833 , 20.00, 2
> 0.007 <= 0.008 , 0.0075 , 30.00, 1
> 0.012 <= 0.014 , 0.013 , 50.00, 2
> 0.014 <= 0.016 , 0.015 , 60.00, 1
> 0.016 <= 0.018 , 0.017 , 70.00, 1
> 0.05 <= 0.0589057 , 0.0544528 , 100.00, 3
# target 50% 0.014
# target 75% 0.0514843
# target 90% 0.0559371
# target 99% 0.0586088
# target 99.9% 0.058876
Error cases : count 9 avg 0.021538722 +/- 0.01954 min 0.0063666 max 0.0575147 sum 0.1938485
# range, mid point, percentile, count
>= 0.0063666 <= 0.007 , 0.0066833 , 22.22, 2
> 0.007 <= 0.008 , 0.0075 , 33.33, 1
> 0.012 <= 0.014 , 0.013 , 55.56, 2
> 0.014 <= 0.016 , 0.015 , 66.67, 1
> 0.016 <= 0.018 , 0.017 , 77.78, 1
> 0.05 <= 0.0575147 , 0.0537574 , 100.00, 2
# target 50% 0.0135
# target 75% 0.0175
# target 90% 0.0541331
# target 99% 0.0571765
# target 99.9% 0.0574809
20:41:04 I httprunner.go:197> [0]   4 socket used, resolved to 10.96.151.249:8080
20:41:04 I httprunner.go:197> [1]   3 socket used, resolved to 10.96.151.249:8080
20:41:04 I httprunner.go:197> [2]   2 socket used, resolved to 10.96.151.249:8080
Sockets used: 9 (for perfect keepalive, would be 3)
Uniform: false, Jitter: false
IP addresses distribution:
10.96.151.249:8080: 3
Code 200 : 1 (10.0 %)
Code 429 : 9 (90.0 %)
Response Header Sizes : count 10 avg 12.5 +/- 37.5 min 0 max 125 sum 125
Response Body/Total Sizes : count 10 avg 162.2 +/- 12.41 min 125 max 167 sum 1622
All done 10 calls (plus 0 warmup) 25.275 ms avg, 6.6 qps
```

As seen above, only `1` out of `10` HTTP requests succeeded, while the remaining `9` requests were rate limited with a `429 (Too Many Requests)` response as per the rate limiting policy.
```
Code 200 : 1 (10.0 %)
Code 429 : 9 (90.0 %)
```

Examine the stats to further confirm this.

```console
$ osm proxy get stats "$fortio_server" -n demo | grep 'over_limit'
cluster.demo/fortio|8080|local.ratelimit.over_limit: 9
```

#### remoteAddress descriptor

A `remoteAddress` descriptor entry has a key of `remote_address` and a value of the client IP address that is populated using the trusted address from the `x-forwarded-for` HTTP header.

Recollect that we defined a rate limiting policy in the `ratelimit-config` ConfigMap to rate limit requests from each unique client to 3 per minute.
```
descriptors:
  # each unique remote (client) address is limited to 3 per minute
  - key: remote_address
    rate_limit:
      unit: minute
      requests_per_unit: 3
```

Update global rate limit policy targeting the `fortio` service to generate the `remote_address` descriptor entry.

```bash
kubectl apply -f - <<EOF
apiVersion: policy.openservicemesh.io/v1alpha1
kind: UpstreamTrafficSetting
metadata:
  name: fortio-tcp
  namespace: demo
spec:
  host: fortio.demo.svc.cluster.local
  rateLimit:
    global:
      http:
        rateLimitService:
          host: ratelimiter.rls.svc.cluster.local
          port: 8081
        domain: test
        failOpen: false
        timeout: 10s
        descriptors:
          - entries:
            - remoteAddress: {}
EOF
```

Confirm the HTTP requests from the `fortio-client` to the `fortio` service are rate limited.

Reset the stat counters on the `fortio` server pod's Envoy sidecar to see the impact of rate limiting.
```bash
osm proxy set reset_counters "$fortio_server" -n demo
```

```console
$ kubectl exec "$fortio_client" -n demo -c fortio-client -- fortio load -c 3 -n 10 http://fortio.demo.svc.cluster.local:8080
Fortio 1.34.1 running at 8 queries per second, 8->8 procs, for 10 calls: http://fortio.demo.svc.cluster.local:8080
21:12:53 I httprunner.go:98> Starting http test for http://fortio.demo.svc.cluster.local:8080 with 3 threads at 8.0 qps and parallel warmup
Starting at 8 qps with 3 thread(s) [gomax 8] : exactly 10, 3 calls each (total 9 + 1)
21:12:54 W http_client.go:889> [0] Non ok http code 429 (HTTP/1.1 429)
21:12:54 W http_client.go:889> [1] Non ok http code 429 (HTTP/1.1 429)
21:12:54 W http_client.go:889> [2] Non ok http code 429 (HTTP/1.1 429)
21:12:54 W http_client.go:889> [0] Non ok http code 429 (HTTP/1.1 429)
21:12:54 W http_client.go:889> [1] Non ok http code 429 (HTTP/1.1 429)
21:12:54 I periodic.go:721> T001 ended after 1.1319011s : 3 calls. qps=2.6504082379635467
21:12:54 W http_client.go:889> [2] Non ok http code 429 (HTTP/1.1 429)
21:12:54 I periodic.go:721> T002 ended after 1.1328863s : 3 calls. qps=2.648103344528043
21:12:55 W http_client.go:889> [0] Non ok http code 429 (HTTP/1.1 429)
21:12:55 I periodic.go:721> T000 ended after 1.5092389s : 4 calls. qps=2.6503425004484047
Ended after 1.5100615s : 10 calls. qps=6.6222
Sleep times : count 7 avg 0.52272264 +/- 0.03009 min 0.4810746 max 0.5568693 sum 3.6590585
Aggregated Function Time : count 10 avg 0.01113793 +/- 0.007175 min 0.0051703 max 0.0265551 sum 0.1113793
# range, mid point, percentile, count
>= 0.0051703 <= 0.006 , 0.00558515 , 30.00, 3
> 0.006 <= 0.007 , 0.0065 , 40.00, 1
> 0.007 <= 0.008 , 0.0075 , 60.00, 2
> 0.008 <= 0.009 , 0.0085 , 70.00, 1
> 0.018 <= 0.02 , 0.019 , 90.00, 2
> 0.025 <= 0.0265551 , 0.0257776 , 100.00, 1
# target 50% 0.0075
# target 75% 0.0185
# target 90% 0.02
# target 99% 0.0263996
# target 99.9% 0.0265395
Error cases : count 7 avg 0.0066644 +/- 0.001223 min 0.0051703 max 0.0087338 sum 0.0466508
# range, mid point, percentile, count
>= 0.0051703 <= 0.006 , 0.00558515 , 42.86, 3
> 0.006 <= 0.007 , 0.0065 , 57.14, 1
> 0.007 <= 0.008 , 0.0075 , 85.71, 2
> 0.008 <= 0.0087338 , 0.0083669 , 100.00, 1
# target 50% 0.0065
# target 75% 0.007625
# target 90% 0.00822014
# target 99% 0.00868243
# target 99.9% 0.00872866
21:12:55 I httprunner.go:197> [0]   3 socket used, resolved to 10.96.151.249:8080
21:12:55 I httprunner.go:197> [1]   2 socket used, resolved to 10.96.151.249:8080
21:12:55 I httprunner.go:197> [2]   2 socket used, resolved to 10.96.151.249:8080
Sockets used: 7 (for perfect keepalive, would be 3)
Uniform: false, Jitter: false
IP addresses distribution:
10.96.151.249:8080: 3
Code 200 : 3 (30.0 %)
Code 429 : 7 (70.0 %)
Response Header Sizes : count 10 avg 37.5 +/- 57.28 min 0 max 125 sum 375
Response Body/Total Sizes : count 10 avg 153.7 +/- 18.79 min 125 max 166 sum 1537
All done 10 calls (plus 0 warmup) 11.138 ms avg, 6.6 qps
```

As seen above, only `3` out of `10` HTTP requests succeeded, while the remaining `7` requests were rate limited with a `429 (Too Many Requests)` response as per the rate limiting policy.
```
Code 200 : 3 (30.0 %)
Code 429 : 7 (70.0 %)
```

Examine the stats to further confirm this.

```console
$ osm proxy get stats "$fortio_server" -n demo | grep 'over_limit'
cluster.demo/fortio|8080|local.ratelimit.over_limit: 7
```

#### requestHeader descriptor

A `requestHeader` descriptor entry defines a static key-value pair for the descriptor entry that is generated only when the request header matches the given header name. The value of the descriptor entry is derived from the value of the header present in the request.

Recollect that we defined a rate limiting policy in the `ratelimit-config` ConfigMap to rate limit requests with the header `my-header: foo` and descriptor key `my_header` to 5 per minute.
```
descriptors:
  # requests with the header 'my-header: foo' are rate limited
  # to 5 per minute
  - key: my_header
    value: foo
    rate_limit:
      unit: minute
      requests_per_unit: 5
```

Update global rate limit policy targeting the `fortio` service to generate the `remote_address` descriptor entry.

```bash
kubectl apply -f - <<EOF
apiVersion: policy.openservicemesh.io/v1alpha1
kind: UpstreamTrafficSetting
metadata:
  name: fortio-tcp
  namespace: demo
spec:
  host: fortio.demo.svc.cluster.local
  rateLimit:
    global:
      http:
        rateLimitService:
          host: ratelimiter.rls.svc.cluster.local
          port: 8081
        domain: test
        failOpen: false
        timeout: 10s
        descriptors:
          - entries:
            - requestHeader:
                name: my-header
                key: my_header
EOF
```

Confirm the HTTP requests with the header `my-header: foo` from the `fortio-client` to the `fortio` service are rate limited.

Reset the stat counters on the `fortio` server pod's Envoy sidecar to see the impact of rate limiting.
```bash
osm proxy set reset_counters "$fortio_server" -n demo
```

```console
$ kubectl exec "$fortio_client" -n demo -c fortio-client -- fortio load -c 3 -n 10 -H "my-header: foo" http://fortio.demo.svc.cluster.local:8080
Fortio 1.34.1 running at 8 queries per second, 8->8 procs, for 10 calls: http://fortio.demo.svc.cluster.local:8080
22:52:57 I httprunner.go:98> Starting http test for http://fortio.demo.svc.cluster.local:8080 with 3 threads at 8.0 qps and parallel warmup
Starting at 8 qps with 3 thread(s) [gomax 8] : exactly 10, 3 calls each (total 9 + 1)
22:52:58 W http_client.go:889> [1] Non ok http code 429 (HTTP/1.1 429)
22:52:58 W http_client.go:889> [0] Non ok http code 429 (HTTP/1.1 429)
22:52:58 W http_client.go:889> [2] Non ok http code 429 (HTTP/1.1 429)
22:52:58 I periodic.go:721> T002 ended after 1.129234s : 3 calls. qps=2.656668148497123
22:52:58 W http_client.go:889> [1] Non ok http code 429 (HTTP/1.1 429)
22:52:58 I periodic.go:721> T001 ended after 1.1527555s : 3 calls. qps=2.6024599318762736
22:52:58 W http_client.go:889> [0] Non ok http code 429 (HTTP/1.1 429)
22:52:58 I periodic.go:721> T000 ended after 1.5212944s : 4 calls. qps=2.629339856900808
Ended after 1.5213586s : 10 calls. qps=6.5731
Sleep times : count 7 avg 0.46446531 +/- 0.07222 min 0.3480318 max 0.5554373 sum 3.2512572
Aggregated Function Time : count 10 avg 0.05466351 +/- 0.06385 min 0.0036194 max 0.1516558 sum 0.5466351
# range, mid point, percentile, count
>= 0.0036194 <= 0.004 , 0.0038097 , 10.00, 1
> 0.006 <= 0.007 , 0.0065 , 20.00, 1
> 0.007 <= 0.008 , 0.0075 , 30.00, 1
> 0.008 <= 0.009 , 0.0085 , 40.00, 1
> 0.016 <= 0.018 , 0.017 , 50.00, 1
> 0.02 <= 0.025 , 0.0225 , 60.00, 1
> 0.025 <= 0.03 , 0.0275 , 70.00, 1
> 0.14 <= 0.151656 , 0.145828 , 100.00, 3
# target 50% 0.018
# target 75% 0.141943
# target 90% 0.147771
# target 99% 0.151267
# target 99.9% 0.151617
Error cases : count 5 avg 0.01562006 +/- 0.008393 min 0.0036194 max 0.0270713 sum 0.0781003
# range, mid point, percentile, count
>= 0.0036194 <= 0.004 , 0.0038097 , 20.00, 1
> 0.008 <= 0.009 , 0.0085 , 40.00, 1
> 0.016 <= 0.018 , 0.017 , 60.00, 1
> 0.02 <= 0.025 , 0.0225 , 80.00, 1
> 0.025 <= 0.0270713 , 0.0260357 , 100.00, 1
# target 50% 0.017
# target 75% 0.02375
# target 90% 0.0260357
# target 99% 0.0269677
# target 99.9% 0.0270609
22:52:58 I httprunner.go:197> [0]   2 socket used, resolved to 10.96.151.249:8080
22:52:58 I httprunner.go:197> [1]   2 socket used, resolved to 10.96.151.249:8080
22:52:58 I httprunner.go:197> [2]   1 socket used, resolved to 10.96.151.249:8080
Sockets used: 5 (for perfect keepalive, would be 3)
Uniform: false, Jitter: false
IP addresses distribution:
10.96.151.249:8080: 3
Code 200 : 5 (50.0 %)
Code 429 : 5 (50.0 %)
Response Header Sizes : count 10 avg 62.6 +/- 62.6 min 0 max 126 sum 626
Response Body/Total Sizes : count 10 avg 145.9 +/- 20.71 min 124 max 167 sum 1459
All done 10 calls (plus 0 warmup) 54.664 ms avg, 6.6 qps
```

As seen above, only `5` out of `10` HTTP requests succeeded, while the remaining `5` requests were rate limited with a `429 (Too Many Requests)` response as per the rate limiting policy.
```
Code 200 : 5 (50.0 %)
Code 429 : 5 (50.0 %)
```

Examine the stats to further confirm this.

```console
$ osm proxy get stats "$fortio_server" -n demo | grep 'over_limit'
cluster.demo/fortio|8080|local.ratelimit.over_limit: 5
```

Further, confirm the HTTP requests with the header `my-header: baz` from the `fortio-client` to the `fortio` service are not rate limited. This is because our rate limit policy is only applicable to HTTP requests with the header `my-header` and value `foo` based on the following descriptor rule applied earlier:
```
- key: my_header
  value: foo
  rate_limit:
    unit: minute
    requests_per_unit: 5
```

```console
$ kubectl exec "$fortio_client" -n demo -c fortio-client -- fortio load -c 3 -n 10 -H "my-header: baz" http://fortio.demo.svc.cluster.local:8080
Fortio 1.34.1 running at 8 queries per second, 8->8 procs, for 10 calls: http://fortio.demo.svc.cluster.local:8080
22:57:16 I httprunner.go:98> Starting http test for http://fortio.demo.svc.cluster.local:8080 with 3 threads at 8.0 qps and parallel warmup
Starting at 8 qps with 3 thread(s) [gomax 8] : exactly 10, 3 calls each (total 9 + 1)
22:57:17 I periodic.go:721> T001 ended after 1.1304947s : 3 calls. qps=2.65370549724824
22:57:17 I periodic.go:721> T002 ended after 1.1307771s : 3 calls. qps=2.6530427614779253
22:57:17 I periodic.go:721> T000 ended after 1.5037149s : 4 calls. qps=2.6600787157193166
Ended after 1.5037834s : 10 calls. qps=6.6499
Sleep times : count 7 avg 0.52288017 +/- 0.03204 min 0.4827303 max 0.5549368 sum 3.6601612
Aggregated Function Time : count 10 avg 0.00994919 +/- 0.00635 min 0.0034616 max 0.0251343 sum 0.0994919
# range, mid point, percentile, count
>= 0.0034616 <= 0.004 , 0.0037308 , 10.00, 1
> 0.004 <= 0.005 , 0.0045 , 20.00, 1
> 0.005 <= 0.006 , 0.0055 , 30.00, 1
> 0.006 <= 0.007 , 0.0065 , 50.00, 2
> 0.008 <= 0.009 , 0.0085 , 70.00, 2
> 0.014 <= 0.016 , 0.015 , 80.00, 1
> 0.016 <= 0.018 , 0.017 , 90.00, 1
> 0.025 <= 0.0251343 , 0.0250671 , 100.00, 1
# target 50% 0.007
# target 75% 0.015
# target 90% 0.018
# target 99% 0.0251209
# target 99.9% 0.025133
Error cases : no data
22:57:17 I httprunner.go:197> [0]   1 socket used, resolved to 10.96.151.249:8080
22:57:17 I httprunner.go:197> [1]   1 socket used, resolved to 10.96.151.249:8080
22:57:17 I httprunner.go:197> [2]   1 socket used, resolved to 10.96.151.249:8080
Sockets used: 3 (for perfect keepalive, would be 3)
Uniform: false, Jitter: false
IP addresses distribution:
10.96.151.249:8080: 3
Code 200 : 10 (100.0 %)
Response Header Sizes : count 10 avg 124.3 +/- 0.4583 min 124 max 125 sum 1243
Response Body/Total Sizes : count 10 avg 124.3 +/- 0.4583 min 124 max 125 sum 1243
All done 10 calls (plus 0 warmup) 9.949 ms avg, 6.6 qps
```

As seen above, all the `10` requests succeeded.
```
Code 200 : 10 (100.0 %)
```

#### headerValueMatch descriptor

A `headerValueMatch` descriptor entry defines a descriptor entry that is generated when the request header matches the given set of HTTP header match criteria. OSM supports multiple header match operators in the form of `Exact, Prefix, Suffix, Regex, Contains, Present` match semantics.

Recollect that we defined a rate limiting policy in the `ratelimit-config` ConfigMap to rate limit requests that generate the descriptor key `header_match: foo` to 7 per minute.
```
descriptors:
  # requests with the descriptor 'header_match: foo' are rate limited
  # to 7 per minute
  - key: header_match
    value: foo
    rate_limit:
      unit: minute
      requests_per_unit: 7
```

Update the global rate limit policy targeting the `fortio` service to generate the descriptor `header_match: foo` to rate limit requests containing the header `my-header` and not containing the header `other-header` using the global rate limiter.

```bash
kubectl apply -f - <<EOF
apiVersion: policy.openservicemesh.io/v1alpha1
kind: UpstreamTrafficSetting
metadata:
  name: fortio-tcp
  namespace: demo
spec:
  host: fortio.demo.svc.cluster.local
  rateLimit:
    global:
      http:
        rateLimitService:
          host: ratelimiter.rls.svc.cluster.local
          port: 8081
        domain: test
        failOpen: false
        timeout: 10s
        descriptors:
          - entries:
            - headerValueMatch:
                # key: header_match # default
                value: foo
                headers:
                  - name: my-header
                    present: true
                  - name: other-header
                    present: false
EOF
```

Confirm the HTTP requests from the `fortio-client` to the `fortio` service containing the `my-header` header and not containing the `other-header` header are rate limited.

Reset the stat counters on the `fortio` server pod's Envoy sidecar to see the impact of rate limiting.
```bash
osm proxy set reset_counters "$fortio_server" -n demo
```

```console
$ kubectl exec "$fortio_client" -n demo -c fortio-client -- fortio load -c 3 -n 10 -H "my-header: baz" http://fortio.demo.svc.cluster.local:8080
Fortio 1.34.1 running at 8 queries per second, 8->8 procs, for 10 calls: http://fortio.demo.svc.cluster.local:8080
18:37:34 I httprunner.go:98> Starting http test for http://fortio.demo.svc.cluster.local:8080 with 3 threads at 8.0 qps and parallel warmup
Starting at 8 qps with 3 thread(s) [gomax 8] : exactly 10, 3 calls each (total 9 + 1)
18:37:35 W http_client.go:889> [2] Non ok http code 429 (HTTP/1.1 429)
18:37:35 I periodic.go:721> T002 ended after 1.1387878s : 3 calls. qps=2.6343801716175745
18:37:35 W http_client.go:889> [1] Non ok http code 429 (HTTP/1.1 429)
18:37:35 I periodic.go:721> T001 ended after 1.1430429s : 3 calls. qps=2.624573408399632
18:37:35 W http_client.go:889> [0] Non ok http code 429 (HTTP/1.1 429)
18:37:35 I periodic.go:721> T000 ended after 1.5074722s : 4 calls. qps=2.6534486009095226
Ended after 1.507513s : 10 calls. qps=6.6334
Sleep times : count 7 avg 0.50088606 +/- 0.03464 min 0.4371301 max 0.5481652 sum 3.5062024
Aggregated Function Time : count 10 avg 0.02777703 +/- 0.02278 min 0.0063771 max 0.0628465 sum 0.2777703
# range, mid point, percentile, count
>= 0.0063771 <= 0.007 , 0.00668855 , 20.00, 2
> 0.012 <= 0.014 , 0.013 , 40.00, 2
> 0.014 <= 0.016 , 0.015 , 50.00, 1
> 0.016 <= 0.018 , 0.017 , 60.00, 1
> 0.018 <= 0.02 , 0.019 , 70.00, 1
> 0.06 <= 0.0628465 , 0.0614232 , 100.00, 3
# target 50% 0.016
# target 75% 0.0604744
# target 90% 0.0618977
# target 99% 0.0627516
# target 99.9% 0.062837
Error cases : count 3 avg 0.012477667 +/- 0.004384 min 0.0067722 max 0.0174308 sum 0.037433
# range, mid point, percentile, count
>= 0.0067722 <= 0.007 , 0.0068861 , 33.33, 1
> 0.012 <= 0.014 , 0.013 , 66.67, 1
> 0.016 <= 0.0174308 , 0.0167154 , 100.00, 1
# target 50% 0.013
# target 75% 0.0163577
# target 90% 0.0170016
# target 99% 0.0173879
# target 99.9% 0.0174265
18:37:35 I httprunner.go:197> [0]   1 socket used, resolved to 10.96.151.249:8080
18:37:35 I httprunner.go:197> [1]   1 socket used, resolved to 10.96.151.249:8080
18:37:35 I httprunner.go:197> [2]   1 socket used, resolved to 10.96.151.249:8080
Sockets used: 3 (for perfect keepalive, would be 3)
Uniform: false, Jitter: false
IP addresses distribution:
10.96.151.249:8080: 3
Code 200 : 7 (70.0 %)
Code 429 : 3 (30.0 %)
Response Header Sizes : count 10 avg 87.3 +/- 57.15 min 0 max 125 sum 873
Response Body/Total Sizes : count 10 avg 137.2 +/- 19.08 min 124 max 167 sum 1372
All done 10 calls (plus 0 warmup) 27.777 ms avg, 6.6 qps
```

As seen above, only `7` out of `10` HTTP requests succeeded, while the remaining `3` requests were rate limited with a `429 (Too Many Requests)` response as per the rate limiting policy.
```
Code 200 : 7 (70.0 %)
Code 429 : 3 (30.0 %)
```

Examine the stats to further confirm this.

```console
$ osm proxy get stats "$fortio_server" -n demo | grep 'over_limit'
cluster.demo/fortio|8080|local.ratelimit.over_limit: 3
```

Confirm the HTTP requests from the `fortio-client` to the `fortio` service containing the `other-header` header are rate not limited as per the `present: false` policy configured for this header.

```console
$ kubectl exec "$fortio_client" -n demo -c fortio-client -- fortio load -c 3 -n 10 -H "other-header: baz" http://fortio.demo.svc.cluster.local:8080
Fortio 1.34.1 running at 8 queries per second, 8->8 procs, for 10 calls: http://fortio.demo.svc.cluster.local:8080
18:40:59 I httprunner.go:98> Starting http test for http://fortio.demo.svc.cluster.local:8080 with 3 threads at 8.0 qps and parallel warmup
Starting at 8 qps with 3 thread(s) [gomax 8] : exactly 10, 3 calls each (total 9 + 1)
18:41:00 I periodic.go:721> T002 ended after 1.1302201s : 3 calls. qps=2.6543502455849084
18:41:00 I periodic.go:721> T001 ended after 1.1302202s : 3 calls. qps=2.6543500107324216
18:41:00 I periodic.go:721> T000 ended after 1.5044186s : 4 calls. qps=2.65883444940125
Ended after 1.50449s : 10 calls. qps=6.6468
Sleep times : count 7 avg 0.52947463 +/- 0.03007 min 0.492957 max 0.5582885 sum 3.7063224
Aggregated Function Time : count 10 avg 0.00537314 +/- 0.002502 min 0.0026129 max 0.0105128 sum 0.0537314
# range, mid point, percentile, count
>= 0.0026129 <= 0.003 , 0.00280645 , 10.00, 1
> 0.003 <= 0.004 , 0.0035 , 40.00, 3
> 0.004 <= 0.005 , 0.0045 , 60.00, 2
> 0.005 <= 0.006 , 0.0055 , 70.00, 1
> 0.006 <= 0.007 , 0.0065 , 80.00, 1
> 0.009 <= 0.01 , 0.0095 , 90.00, 1
> 0.01 <= 0.0105128 , 0.0102564 , 100.00, 1
# target 50% 0.0045
# target 75% 0.0065
# target 90% 0.01
# target 99% 0.0104615
# target 99.9% 0.0105077
Error cases : no data
18:41:00 I httprunner.go:197> [0]   1 socket used, resolved to 10.96.151.249:8080
18:41:00 I httprunner.go:197> [1]   1 socket used, resolved to 10.96.151.249:8080
18:41:00 I httprunner.go:197> [2]   1 socket used, resolved to 10.96.151.249:8080
Sockets used: 3 (for perfect keepalive, would be 3)
Uniform: false, Jitter: false
IP addresses distribution:
10.96.151.249:8080: 3
Code 200 : 10 (100.0 %)
Response Header Sizes : count 10 avg 124 +/- 0 min 124 max 124 sum 1240
Response Body/Total Sizes : count 10 avg 124 +/- 0 min 124 max 124 sum 1240
All done 10 calls (plus 0 warmup) 5.373 ms avg, 6.6 qps
```

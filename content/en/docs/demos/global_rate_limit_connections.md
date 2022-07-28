---
title: "Global rate limiting of L4 connections"
description: "Configuring global rate limiting for L4 connections"
type: docs
weight: 23
---

This guide demonstrates how to configure global rate limiting for L4 TCP connections destined to a target host that is a part of an OSM managed service mesh.

## Prerequisites

- Kubernetes cluster running Kubernetes {{< param min_k8s_version >}} or greater.
- Have OSM installed.
- Have `kubectl` available to interact with the API server.
- Have `osm` CLI available for managing the service mesh.
- OSM version >= v1.3.0.


## Demo

The following demo shows a client [fortio-client](https://github.com/fortio/fortio) sending TCP traffic to the `fortio` `TCP echo` service. The `fortio` service echoes TCP messages back to the client. We will see the impact of applying global TCP rate limiting policies targeting the `fortio` service to control the throughput of traffic destined to the service backend using an external Rate Limit Service (RLS).

1. For simplicity, enable [permissive traffic policy mode](/docs/guides/traffic_management/permissive_mode) so that explicit SMI traffic access policies are not required for application connectivity within the mesh.
    ```bash
    export osm_namespace=osm-system # Replace osm-system with the namespace where OSM is installed
    kubectl patch meshconfig osm-mesh-config -n "$osm_namespace" -p '{"spec":{"traffic":{"enablePermissiveTrafficPolicyMode":true}}}'  --type=merge
    ```

1. Deploy the `fortio` `TCP echo` service in the `demo` namespace after enrolling its namespace to the mesh. The `fortio` `TCP echo` service runs on port `8078`.
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
    NAME                            READY   STATUS    RESTARTS   AGE
    fortio-client-b9b7bbfb8-prq7r   2/2     Running   0          7s
    ```

1. Deploy the external RLS. Any RLS that implement's [Envoy's Rate Limit Service proto](https://www.envoyproxy.io/docs/envoy/latest/api-v3/service/ratelimit/v3/rls.proto) can be used as a global rate limiter with Envoy. For this demo, we use the [Envoy Rate Limit Service](https://github.com/envoyproxy/ratelimit) as our RLS. Per the [Envoy RLS overview](https://github.com/envoyproxy/ratelimit#overview):
    > The rate limit service is a Go/gRPC service designed to enable generic rate limit scenarios from different types of applications. Applications request a rate limit decision based on a domain and a set of descriptors. The service reads the configuration from disk via runtime, composes a cache key, and talks to the Redis cache. A decision is then returned to the caller.

    Create a namespace to deploy the RLS into.
    ```bash
    kubectl create namespace rls
    ```

    Create a ConfigMap that contains the [rate limit configuration](https://github.com/envoyproxy/ratelimit#configuration). In this demo, we will rate limit traffic that generates the descriptor key-value pair `my_key: my_value` to 1 request per minute.
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
          # requests with a descriptor of ["my_key": "my_value"]
          # are limited to one per minute.
          - key: my_key
            value: my_value
            rate_limit:
              unit: minute
              requests_per_unit: 1
    EOF
    ```

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

1. Confirm the `fortio-client` app is able to successfully make TCP connections and send data to the `fortio` `TCP echo` service on port `8078`. We call the `fortio` service with `3` concurrent connections (`-c 3`) and send `10` calls (`-n 10`).
    ```console
    $ fortio_client="$(kubectl get pod -n demo -l app=fortio-client -o jsonpath='{.items[0].metadata.name}')"

    $ kubectl exec "$fortio_client" -n demo -c fortio-client -- fortio load -qps -1 -c 3 -n 10 tcp://fortio.demo.svc.cluster.local:8078
    Fortio 1.32.3 running at -1 queries per second, 8->8 procs, for 10 calls: tcp://fortio.demo.svc.cluster.local:8078
    20:41:47 I tcprunner.go:238> Starting tcp test for tcp://fortio.demo.svc.cluster.local:8078 with 3 threads at -1.0 qps
    Starting at max qps with 3 thread(s) [gomax 8] for exactly 10 calls (3 per thread + 1)
    20:41:47 I periodic.go:723> T001 ended after 34.0563ms : 3 calls. qps=88.0894283876992
    20:41:47 I periodic.go:723> T000 ended after 35.3117ms : 4 calls. qps=113.2769025563765
    20:41:47 I periodic.go:723> T002 ended after 44.0273ms : 3 calls. qps=68.13954069406937
    Ended after 44.2097ms : 10 calls. qps=226.19
    Aggregated Function Time : count 10 avg 0.01096615 +/- 0.01386 min 0.001588 max 0.0386716 sum 0.1096615
    # range, mid point, percentile, count
    >= 0.001588 <= 0.002 , 0.001794 , 40.00, 4
    > 0.002 <= 0.003 , 0.0025 , 60.00, 2
    > 0.003 <= 0.004 , 0.0035 , 70.00, 1
    > 0.025 <= 0.03 , 0.0275 , 90.00, 2
    > 0.035 <= 0.0386716 , 0.0368358 , 100.00, 1
    # target 50% 0.0025
    # target 75% 0.02625
    # target 90% 0.03
    # target 99% 0.0383044
    # target 99.9% 0.0386349
    Error cases : no data
    Sockets used: 3 (for perfect no error run, would be 3)
    Total Bytes sent: 240, received: 240
    tcp OK : 10 (100.0 %)
    All done 10 calls (plus 0 warmup) 10.966 ms avg, 226.2 qps
    ```

    As seen above, all the TCP connections from the `fortio-client` pod succeeded.
    ```
    Total Bytes sent: 240, received: 240
    tcp OK : 10 (100.0 %)
    All done 10 calls (plus 0 warmup) 10.966 ms avg, 226.2 qps
    ```

1. Next, apply a global rate limiting policy to rate limit L4 TCP connections to the `fortio.demo.svc.cluster.local` service that generate the descriptor entry `my_key: my_value` for the `test` domain via the global `ratelimiter.rls.svc.cluster.local` RLS service listening on port `8081`. The descriptor entries and domain correspond to the rate limit configuration specified in the `ratelimit-config` ConfigMap previously. This has the utlimate effect of rate limiting TCP connections inbound on the `fortio` service to 1 per minute by generating the descriptor entry that matches the one specified in the `ratelimit-config` ConfigMap.
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
          tcp:
            rateLimitService:
              host: ratelimiter.rls.svc.cluster.local
              port: 8081
            domain: test
            failOpen: false
            timeout: 10s
            descriptors:
              - entries:
                - key: my_key
                  value: my_value
    EOF
    ```

    The above configuration will result in connections to the `fortio` backend service being forwarded to the RLS service for making a rate limit decision based on the descriptor entry `my_key: my_value`.

    Confirm no traffic has been rate limited yet by examining the stats on the `fortio` backend pod.
    ```console
    $ fortio_server="$(kubectl get pod -n demo -l app=fortio -o jsonpath='{.items[0].metadata.name}')"

    $ osm proxy get stats "$fortio_server" -n demo | grep ratelimit.*fortio.*8078
    ratelimit.inbound_demo/fortio_8078_tcp.active: 0
    ratelimit.inbound_demo/fortio_8078_tcp.cx_closed: 0
    ratelimit.inbound_demo/fortio_8078_tcp.error: 0
    ratelimit.inbound_demo/fortio_8078_tcp.failure_mode_allowed: 0
    ratelimit.inbound_demo/fortio_8078_tcp.ok: 0
    ratelimit.inbound_demo/fortio_8078_tcp.over_limit: 0
    ratelimit.inbound_demo/fortio_8078_tcp.total: 0
    ```

1. Confirm TCP connections are rate limited.
    ```console
    $ kubectl exec "$fortio_client" -n demo -c fortio-client -- fortio load -qps -1 -c 3 -n 10 tcp://fortio.demo.svc.cluster.local:8078
    Fortio 1.34.1 running at -1 queries per second, 8->8 procs, for 10 calls: tcp://fortio.demo.svc.cluster.local:8078
    20:07:50 I tcprunner.go:238> Starting tcp test for tcp://fortio.demo.svc.cluster.local:8078 with 3 threads at -1.0 qps
    Starting at max qps with 3 thread(s) [gomax 8] for exactly 10 calls (3 per thread + 1)
    20:07:50 E tcprunner.go:203> [2] Unable to read: EOF
    20:07:50 E tcprunner.go:203> [1] Unable to read: EOF
    20:07:50 E tcprunner.go:203> [2] Unable to read: EOF
    20:07:51 I periodic.go:721> T000 ended after 152.8903ms : 4 calls. qps=26.162549226471526
    20:07:51 E tcprunner.go:203> [1] Unable to read: EOF
    20:07:51 E tcprunner.go:203> [2] Unable to read: EOF
    20:07:51 I periodic.go:721> T002 ended after 155.9383ms : 3 calls. qps=19.23837825601536
    20:07:51 E tcprunner.go:203> [1] Unable to read: EOF
    20:07:51 I periodic.go:721> T001 ended after 162.9256ms : 3 calls. qps=18.41331257948413
    Ended after 162.9618ms : 10 calls. qps=61.364
    Aggregated Function Time : count 10 avg 0.04707677 +/- 0.0595 min 0.0008829 max 0.1400948 sum 0.4707677
    # range, mid point, percentile, count
    >= 0.0008829 <= 0.001 , 0.00094145 , 10.00, 1
    > 0.001 <= 0.002 , 0.0015 , 20.00, 1
    > 0.009 <= 0.01 , 0.0095 , 40.00, 2
    > 0.01 <= 0.011 , 0.0105 , 50.00, 1
    > 0.012 <= 0.014 , 0.013 , 70.00, 2
    > 0.12 <= 0.14 , 0.13 , 90.00, 2
    > 0.14 <= 0.140095 , 0.140047 , 100.00, 1
    # target 50% 0.011
    # target 75% 0.125
    # target 90% 0.14
    # target 99% 0.140085
    # target 99.9% 0.140094
    Error cases : count 6 avg 0.053029183 +/- 0.05918 min 0.0095806 max 0.1400948 sum 0.3181751
    # range, mid point, percentile, count
    >= 0.0095806 <= 0.01 , 0.0097903 , 16.67, 1
    > 0.01 <= 0.011 , 0.0105 , 33.33, 1
    > 0.012 <= 0.014 , 0.013 , 66.67, 2
    > 0.12 <= 0.14 , 0.13 , 83.33, 1
    > 0.14 <= 0.140095 , 0.140047 , 100.00, 1
    # target 50% 0.013
    # target 75% 0.13
    # target 90% 0.140038
    # target 99% 0.140089
    # target 99.9% 0.140094
    Sockets used: 7 (for perfect no error run, would be 3)
    Total Bytes sent: 240, received: 96
    tcp OK : 4 (40.0 %)
    tcp short read : 6 (60.0 %)
    All done 10 calls (plus 0 warmup) 47.077 ms avg, 61.4 qps
    ```

    As seen above, only 40% of the 10 calls succeeded, while the remaining 60% was rate limitied. This is because we applied a rate limiting policy of 1 connection per minute at the `fortio` backend service, and the `fortio-client` was able to use 1 connection to make 4/10 calls, resulting in a 40% success rate.

    Examine the sidecar stats to further confirm this.
    ```console
    $ osm proxy get stats "$fortio_server" -n demo | grep ratelimit.*fortio.*8078
    ratelimit.inbound_demo/fortio_8078_tcp.active: 0
    ratelimit.inbound_demo/fortio_8078_tcp.cx_closed: 6
    ratelimit.inbound_demo/fortio_8078_tcp.error: 0
    ratelimit.inbound_demo/fortio_8078_tcp.failure_mode_allowed: 0
    ratelimit.inbound_demo/fortio_8078_tcp.ok: 1
    ratelimit.inbound_demo/fortio_8078_tcp.over_limit: 6
    ratelimit.inbound_demo/fortio_8078_tcp.total: 7
    ```

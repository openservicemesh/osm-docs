---
title: "Local rate limiting of L4 connections"
description: "Configuring local rate limiting for L4 connections"
type: docs
weight: 22
---

This guide demonstrates how to configure local rate limiting for L4 TCP connections destined to a target host that is a part of an OSM managed service mesh.

## Prerequisites

- Kubernetes cluster running Kubernetes {{< param min_k8s_version >}} or greater.
- Have OSM installed.
- Have `kubectl` available to interact with the API server.
- Have `osm` CLI available for managing the service mesh.
- OSM version >= v1.2.0.


## Demo

The following demo shows a client [fortio-client](https://github.com/fortio/fortio) sending TCP traffic to the `fortio` `TCP echo` service. The `fortio` service echoes TCP messages back to the client. We will see the impact of applying local TCP rate limiting policies targeting the `fortio` service to control the throughput of traffic destined to the service backend.

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

1. Confirm the `fortio-client` app is able to successfully make TCP connections and send data to the `frotio` `TCP echo` service on port `8078`. We call the `fortio` service with `3` concurrent connections (`-c 3`) and send `10` calls (`-n 10`).
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

1. Next, apply a local rate limiting policy to rate limit L4 TCP connections to the `fortio.demo.svc.cluster.local` service to `1 connection per minute`.
    ```bash
    kubectl apply -f - <<EOF
    apiVersion: policy.openservicemesh.io/v1alpha1
    kind: UpstreamTrafficSetting
    metadata:
      name: tcp-rate-limit
      namespace: demo
    spec:
      host: fortio.demo.svc.cluster.local
      rateLimit:
        local:
          tcp:
            connections: 1
            unit: minute
    EOF
    ```

    Confirm no traffic has been rate limited yet by examining the stats on the `fortio` backend pod.
    ```console
    $ fortio_server="$(kubectl get pod -n demo -l app=fortio -o jsonpath='{.items[0].metadata.name}')"

    $ osm proxy get stats "$fortio_server" -n demo | grep fortio.*8078.*rate_limit
    local_rate_limit.inbound_demo/fortio_8078_tcp.rate_limited: 0
    ```

1. Confirm TCP connections are rate limited.
    ```console
    $ kubectl exec "$fortio_client" -n demo -c fortio-client -- fortio load -qps -1 -c 3 -n 10 tcp://fortio.demo.svc.cluster.local:8078
    Fortio 1.32.3 running at -1 queries per second, 8->8 procs, for 10 calls: tcp://fortio.demo.svc.cluster.local:8078
    20:49:38 I tcprunner.go:238> Starting tcp test for tcp://fortio.demo.svc.cluster.local:8078 with 3 threads at -1.0 qps
    Starting at max qps with 3 thread(s) [gomax 8] for exactly 10 calls (3 per thread + 1)
    20:49:38 E tcprunner.go:203> [2] Unable to read: read tcp 10.244.1.19:59244->10.96.83.254:8078: read: connection reset by peer
    20:49:38 E tcprunner.go:203> [0] Unable to read: read tcp 10.244.1.19:59246->10.96.83.254:8078: read: connection reset by peer
    20:49:38 E tcprunner.go:203> [2] Unable to read: read tcp 10.244.1.19:59258->10.96.83.254:8078: read: connection reset by peer
    20:49:38 E tcprunner.go:203> [0] Unable to read: read tcp 10.244.1.19:59260->10.96.83.254:8078: read: connection reset by peer
    20:49:38 E tcprunner.go:203> [2] Unable to read: read tcp 10.244.1.19:59266->10.96.83.254:8078: read: connection reset by peer
    20:49:38 I periodic.go:723> T002 ended after 9.643ms : 3 calls. qps=311.1065021258944
    20:49:38 E tcprunner.go:203> [0] Unable to read: read tcp 10.244.1.19:59268->10.96.83.254:8078: read: connection reset by peer
    20:49:38 E tcprunner.go:203> [0] Unable to read: read tcp 10.244.1.19:59274->10.96.83.254:8078: read: connection reset by peer
    20:49:38 I periodic.go:723> T000 ended after 14.8212ms : 4 calls. qps=269.8836801338623
    20:49:38 I periodic.go:723> T001 ended after 20.3458ms : 3 calls. qps=147.45057948077735
    Ended after 20.5468ms : 10 calls. qps=486.69
    Aggregated Function Time : count 10 avg 0.00438853 +/- 0.004332 min 0.0014184 max 0.0170216 sum 0.0438853
    # range, mid point, percentile, count
    >= 0.0014184 <= 0.002 , 0.0017092 , 20.00, 2
    > 0.002 <= 0.003 , 0.0025 , 50.00, 3
    > 0.003 <= 0.004 , 0.0035 , 70.00, 2
    > 0.004 <= 0.005 , 0.0045 , 90.00, 2
    > 0.016 <= 0.0170216 , 0.0165108 , 100.00, 1
    # target 50% 0.003
    # target 75% 0.00425
    # target 90% 0.005
    # target 99% 0.0169194
    # target 99.9% 0.0170114
    Error cases : count 7 avg 0.0034268714 +/- 0.0007688 min 0.0024396 max 0.0047932 sum 0.0239881
    # range, mid point, percentile, count
    >= 0.0024396 <= 0.003 , 0.0027198 , 42.86, 3
    > 0.003 <= 0.004 , 0.0035 , 71.43, 2
    > 0.004 <= 0.0047932 , 0.0043966 , 100.00, 2
    # target 50% 0.00325
    # target 75% 0.00409915
    # target 90% 0.00451558
    # target 99% 0.00476544
    # target 99.9% 0.00479042
    Sockets used: 8 (for perfect no error run, would be 3)
    Total Bytes sent: 240, received: 72
    tcp OK : 3 (30.0 %)
    tcp short read : 7 (70.0 %)
    All done 10 calls (plus 0 warmup) 4.389 ms avg, 486.7 qps
    ```

    As seen above, only 30% of the 10 calls succeeded, while the remaining 70% was rate limitied. This is because we applied a rate limiting policy of 1 connection per minute at the `fortio` backend service, and the `fortio-client` was able to use 1 connection to make 3/10 calls, resulting in a 30% success rate.

    Examine the sidecar stats to further confirm this.
    ```console
    $ osm proxy get stats "$fortio_server" -n demo | grep 'fortio.*8078.*rate_limit'
    local_rate_limit.inbound_demo/fortio_8078_tcp.rate_limited: 7
    ```

1. Next, let's update our rate limiting policy to allow a burst of connections. Bursts allow a given number of connections over the baseline rate of 1 connection per minute defined by our rate limiting policy.
    ```bash
    kubectl apply -f - <<EOF
    apiVersion: policy.openservicemesh.io/v1alpha1
    kind: UpstreamTrafficSetting
    metadata:
      name: tcp-echo-limit
      namespace: demo
    spec:
      host: fortio.demo.svc.cluster.local
      rateLimit:
        local:
          tcp:
            connections: 1
            unit: minute
            burst: 10
    EOF
    ```

1. Confirm the burst capability allows a burst of connections within a small window of time.

    Reset the stat counters on the `fortio` server pod's Envoy sidecar to see the impact of rate limiting.
    ```bash
    osm proxy set reset_counters "$fortio_server" -n demo
    ```

    ```console
    $ kubectl exec "$fortio_client" -n demo -c fortio-client -- fortio load -qps -1 -c 3 -n 10 tcp://fortio.demo.svc.cluster.local:8078
    Fortio 1.32.3 running at -1 queries per second, 8->8 procs, for 10 calls: tcp://fortio.demo.svc.cluster.local:8078
    20:56:56 I tcprunner.go:238> Starting tcp test for tcp://fortio.demo.svc.cluster.local:8078 with 3 threads at -1.0 qps
    Starting at max qps with 3 thread(s) [gomax 8] for exactly 10 calls (3 per thread + 1)
    20:56:56 I periodic.go:723> T002 ended after 5.1568ms : 3 calls. qps=581.7561278312132
    20:56:56 I periodic.go:723> T001 ended after 5.2334ms : 3 calls. qps=573.2411052088509
    20:56:56 I periodic.go:723> T000 ended after 5.2464ms : 4 calls. qps=762.4275693809088
    Ended after 5.2711ms : 10 calls. qps=1897.1
    Aggregated Function Time : count 10 avg 0.00153124 +/- 0.001713 min 0.00033 max 0.0044054 sum 0.0153124
    # range, mid point, percentile, count
    >= 0.00033 <= 0.001 , 0.000665 , 70.00, 7
    > 0.003 <= 0.004 , 0.0035 , 80.00, 1
    > 0.004 <= 0.0044054 , 0.0042027 , 100.00, 2
    # target 50% 0.000776667
    # target 75% 0.0035
    # target 90% 0.0042027
    # target 99% 0.00438513
    # target 99.9% 0.00440337
    Error cases : no data
    Sockets used: 3 (for perfect no error run, would be 3)
    Total Bytes sent: 240, received: 240
    tcp OK : 10 (100.0 %)
    All done 10 calls (plus 0 warmup) 1.531 ms avg, 1897.1 qps
    ```

    As seen above, all the TCP connections from the `fortio-client` pod succeeded.
    ```
    Total Bytes sent: 240, received: 240
    tcp OK : 10 (100.0 %)
    All done 10 calls (plus 0 warmup) 1.531 ms avg, 1897.1 qps
    ```

    Further, examine the stats to confirm the burst allows additional connections to go through.
    ```console
    $ osm proxy get stats "$fortio_server" -n demo | grep 'fortio.*8078.*rate_limit'
    local_rate_limit.inbound_demo/fortio_8078_tcp.rate_limited: 0
    ```
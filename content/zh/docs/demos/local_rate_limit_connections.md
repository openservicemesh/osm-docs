---
title: "L4 连接的本地速率限制"
description: "配置 L4 连接的本地速率限制"
type: docs
weight: 22
---

本指南演示了如何为 L4 TCP 连接配置速率限制，目标主机是 OSM 托管服务网格的一部分。

## 前置条件

- Kubernetes 集群版本 {{< param min_k8s_version >}} 或者更高。
- 已安装 OSM。
- 已安装 `kubectl` 用来与 API 服务器进行交互。
- 已安装 `osm` CLI 用于管理服务网格。
- 已安装 OSM，版本 >= v1.2.0。


## 演示

以下演示显示了一个客户端 [fortio-client](https://github.com/fortio/fortio) 将 TCP 流量发送到 `fortio` `TCP echo` 服务。`fortio` 服务将 TCP 消息回显给客户端。我们将看到应用针对 `fortio` 服务的本地 TCP 速率限制策略来控制发往服务后端的流量的影响。

1. 为简单起见，启用 [宽松流量策略模式](/docs/guides/traffic_management/permissive_mode) 以便网格内的应用程序连接不需要显式的 SMI 流量访问策略。
    ```bash
    export osm_namespace=osm-system # Replace osm-system with the namespace where OSM is installed
    kubectl patch meshconfig osm-mesh-config -n "$osm_namespace" -p '{"spec":{"traffic":{"enablePermissiveTrafficPolicyMode":true}}}'  --type=merge
    ```

1. 在将其命名空间注册到网格后，在 `demo` 命名空间中部署 `fortio` `TCP echo` 服务。`fortio` `TCP echo` 服务在端口 `8078` 上运行。
    ```bash
    # Create the demo namespace
    kubectl create namespace demo

    # Add the namespace to the mesh
    osm namespace add demo

    # Deploy fortio TCP echo in the demo namespace
    kubectl apply -f https://raw.githubusercontent.com/openservicemesh/osm-docs/{{< param osm_branch >}}/manifests/samples/fortio/fortio.yaml -n demo
    ```

    确认`fortio` 服务 pod 启动并运行。

    ```console
    $ kubectl get pods -n demo
    NAME                            READY   STATUS    RESTARTS   AGE
    fortio-c4bd7857f-7mm6w          2/2     Running   0          22m
    ```

1. 在 `demo` 命名空间中部署 `fortio-client` 应用程序。我们将使用此客户端将 TCP 流量发送到之前部署的 `fortio TCP echo` 服务。
    ```bash
    kubectl apply -f https://raw.githubusercontent.com/openservicemesh/osm-docs/{{< param osm_branch >}}/manifests/samples/fortio/fortio-client.yaml -n demo
    ```

    确认 `fortio-client` pod 启动并运行。

    ```console
    NAME                            READY   STATUS    RESTARTS   AGE
    fortio-client-b9b7bbfb8-prq7r   2/2     Running   0          7s
    ```

1. 确认 `fortio-client` 应用程序能够成功建立 TCP 连接并将数据发送到端口 `8078` 上的 `frotio` `TCP echo` 服务。我们使用 `3` 并发连接 (`-c 3`) 调用 `fortio` 服务并发送 `10` 次调用 (`-n 10`)。
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

    如上所示，来自 `fortio-client` pod 的所有 TCP 连接都成功了。
    ```
    Total Bytes sent: 240, received: 240
    tcp OK : 10 (100.0 %)
    All done 10 calls (plus 0 warmup) 10.966 ms avg, 226.2 qps
    ```

1. 接下来，应用本地速率限制策略，将到 `fortio.demo.svc.cluster.local` 服务的 L4 TCP 连接速率限制为“每分钟 1 个连接”。
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

    通过检查 `fortio` 后端 pod 上的统计信息，确认没有流量受到速率限制。
    ```console
    $ fortio_server="$(kubectl get pod -n demo -l app=fortio -o jsonpath='{.items[0].metadata.name}')"

    $ osm proxy get stats "$fortio_server" -n demo | grep fortio.*8078.*rate_limit
    local_rate_limit.inbound_demo/fortio_8078_tcp.rate_limited: 0
    ```

1. 确认 TCP 连接受速率限制。
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

    如上所示，10 次调用中只有 30% 成功，而剩余的 70% 是速率受限的。这是因为我们在 `fortio` 后端服务上应用了每分钟 1 个连接的限速策略，而 `fortio-client` 能够使用 1 个连接进行 3/10 调用，从而导致 30% 的成功率。

    检查 sidecar 统计数据以进一步确认这一点。
    ```console
    $ osm proxy get stats "$fortio_server" -n demo | grep 'fortio.*8078.*rate_limit'
    local_rate_limit.inbound_demo/fortio_8078_tcp.rate_limited: 7
    ```

1. 接下来，让我们更新我们的速率限制策略以允许突发连接。突发允许给定数量的连接超过我们的速率限制策略定义的每分钟 1 个连接的基准速率。
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

1. 确认突发功能允许在很短的时间窗口内突发连接。
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

    如上所示，来自 `fortio-client` pod 的所有 TCP 连接都成功了。
    ```
    Total Bytes sent: 240, received: 240
    tcp OK : 10 (100.0 %)
    All done 10 calls (plus 0 warmup) 1.531 ms avg, 1897.1 qps
    ```

    此外，检查统计数据以确认突发允许额外的连接通过。自从我们配置突发设置之前的上一次速率限制测试以来，连接速率限制的数量没有增加。
    ```console
    $ osm proxy get stats "$fortio_server" -n demo | grep 'fortio.*8078.*rate_limit'
    local_rate_limit.inbound_demo/fortio_8078_tcp.rate_limited: 7
    ```
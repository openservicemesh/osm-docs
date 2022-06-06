---
title: "网格内服务的熔断"
description: "为网格内服务配置熔断"
type: docs
weight: 21
---

本指南演示如何为作为 OSM 托管服务网格一部分的目标配置熔断。

## 先决条件

- Kubernetes 集群版本 {{< param min_k8s_version >}} 或者更高。
- 已安装 OSM。
- 使用 `kubectl` 与 API server 交互。
- 已安装 `osm`  命令行工具，用于管理服务网格。
- OSM 版本高于 v1.1.0


## 演示

下面的演示展示了适用负载测试客户端 [fortio](https://github.com/fortio/fortio) 发送流量到 `httpbin` 服务。我们将看到，当配置的熔断限制触发时，应用于 `httpbin` 服务流量的断路器如何影响 `fortio` 客户端。

1. 简单起见，为了网格内的应用互访无需使用SMI 访问控制策略，启用 [宽松流量策略模式](/docs/guides/traffic_management/permissive_mode)
    ```bash
    export osm_namespace=osm-system # Replace osm-system with the namespace where OSM is installed
    kubectl patch meshconfig osm-mesh-config -n "$osm_namespace" -p '{"spec":{"traffic":{"enablePermissiveTrafficPolicyMode":true}}}'  --type=merge
    ```

2. 在 `httpbin` 命名空间下部署 `httpbin` 客户端，并将命名空间纳入网格管理。`httpbin` 服务运行在 `14001` 端口。

    ```bash
    # Create the httpbin namespace
    kubectl create namespace httpbin

    # Add the namespace to the mesh
    osm namespace add httpbin

    # Deploy httpbin service in the httpbin namespace
    kubectl apply -f https://raw.githubusercontent.com/openservicemesh/osm-docs/{{< param osm_branch >}}/manifests/samples/httpbin/httpbin.yaml -n httpbin
    ```

    确认 `httpbin` 服务 pod 启动并运行。

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

3. 在 `client` 命名空间下部署 `fortio` 客户端，并将命名空间纳入网格管理。
    ```bash
    # Create the client namespace
    kubectl create namespace client

    # Add the namespace to the mesh
    osm namespace add client

    # Deploy fortio client in the client namespace
    kubectl apply -f https://raw.githubusercontent.com/openservicemesh/osm-docs/{{< param osm_branch >}}/manifests/samples/fortio/fortio.yaml -n client
    ```

    确认 `fortio` 客户端 pod 启动并运行。

    ```console
    $ kubectl get pods -n client
    NAME                      READY   STATUS    RESTARTS   AGE
    fortio-6477f8495f-bj4s9   2/2     Running   0          19s
    ```

4. 确认 `fortio` 客户端可以成功发送 HTTP 请求到 `httpbin` 服务的 `14001`  端口。访问 `httpbin` 服务使用 `3` 个并发连接（`-c 3`）以及发送 `50` 个请求（`-n 50`）。
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

    如上所示，所有的请求都成功了。
    ```
    Code 200 : 50 (100.0 %)
    ```

5. 接下来，应用断路器配置，通过 `UpstreamTrafficSetting` 资源指定发往 `httpbin` 服务的请求的最大并发连接数和请求书 为 `1`。

   > 注意：必须在上游（目的地）服务的命名空间中创建 `UpstreamTrafficSetting` 资源，同时将主机信息设置成 Kubernetes service 的 FQDN。

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

6. 确认 `fortio` 客户端无法发送再发送同样数量的成功请求，因为上面配置了连接和请求级别的熔断配置。
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

    如上所示，还只有 42% 的请求成功，因为断路器出发其他请求均失败。
    ```
    Code 200 : 21 (42.0 %)
    Code 503 : 29 (58.0 %)
    ```

7. 检查 `Envoy` sidecar 指标发现相关请求的断路器被触发。
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

    `cluster.httpbin/httpbin|14001.upstream_rq_pending_overflow: 29` 说明 29 个请求的断路器打开，与上面的失败请求数一致。
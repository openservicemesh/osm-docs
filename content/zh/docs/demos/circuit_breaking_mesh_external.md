---
title: "网格外部服务的熔断"
description: "为网格外部服务配置熔断"
type: docs
weight: 22
---

本指南演示如何为 OSM 托管服务网格外部的目标配置断路。

## 先决条件

- Kubernetes 集群版本 {{< param min_k8s_version >}} 或者更高。
- 已安装 OSM。
- 使用 `kubectl` 与 API server 交互。
- 已安装 `osm`  命令行工具，用于管理服务网格。
- OSM 版本不低于 v1.1.0


## 演示

下面的演示展示了适用负载测试客户端 [fortio](https://github.com/fortio/fortio) 发送流量到网格外部的 `httpbin` 服务。发送到网格外部的流量被认为是 [出口](/docs/guides/traffic_management/egress) 流量。我们将看下为外部的 `httpbin` 服务配置断路器并触发后如何影响 `fortio` 客户端。

1. 部署 `httpbin` 服务到 `httpbin` 命名空间。`httpbin` 服务运行在 `14001` 端口，且没有纳入网格管理，可以看成是网格外部的服务。

    ```bash
    # Create the httpbin namespace
    kubectl create namespace httpbin

    # Deploy httpbin service in the httpbin namespace
    kubectl apply -f https://raw.githubusercontent.com/openservicemesh/osm-docs/{{< param osm_branch >}}/manifests/samples/httpbin/httpbin.yaml -n httpbin
    ```

    确认 `curl` 服务可以成功发送请求到运行在 `http://54.91.118.50:80` 的 `httpbin.org` 网站。

    ```console
    $ kubectl get svc -n httpbin
    NAME      TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)     AGE
    httpbin   ClusterIP   10.96.198.23   <none>        14001/TCP   20s
    ```

    ```console
    $ kubectl get pods -n httpbin
    NAME                     READY   STATUS    RESTARTS   AGE
    httpbin-5b8b94b9-lt2vs   1/1     Running   0          20s
    ```

2. 将 `fortio` 负载测试客户端部署到 `client` 命名空间并纳入服务网格。
    ```bash
    # Create the client namespace
    kubectl create namespace client

    # Add the namespace to the mesh
    osm namespace add client

    # Deploy fortio client in the client namespace
    kubectl apply -f https://raw.githubusercontent.com/openservicemesh/osm-docs/{{< param osm_branch >}}/manifests/samples/fortio/fortio.yaml -n client
    ```

    确认 `fortio` 客户端可以成功发送请求到运行在 `http://54.91.118.50:80` 的 `httpbin.org` 网站。

    ```console
    $ kubectl get pods -n client
    NAME                      READY   STATUS    RESTARTS   AGE
    fortio-6477f8495f-bj4s9   2/2     Running   0          19s
    ```

3. 配置 Egress 策略允许 `client` 命名空间下的 `fortio` 客户端可以与外部的 `httpbin` 服务通信。HTTP 请求将被定向到 `httpbin.httpbin.svc.cluster.local` 的 `14001` 端口。
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

4. 确认 `fortio` 客户端可以成功发送 HTTP 请求到外部 `httpbin.httpbin.svc.cluster.local` 的 `14001` 端口。使用 `5` 个并发（`-c 5`）发送 50 个请求（`-n 50`）到外部服务。
    ```console
    $ export fortio_pod="$(kubectl get pod -n client -l app=fortio -o jsonpath='{.items[0].metadata.name}')"

    $ kubectl exec "$fortio_pod" -c fortio -n client -- /usr/bin/fortio load -c 5 -qps 0 -n 50 -loglevel Warning http://httpbin.httpbin.svc.cluster.local:14001/get
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

    如上所示，全部请求都成功。
    ```
    Code 200 : 50 (100.0 %)
    ```

5. 接下来，使用 `UpstreamTrafficSetting` 资源为定向到外部主机 `httpbin.httpbin.svc.cluster.local` 的流量限制最大的并发数和请求数为 `1`。在为外部（出口）流量应用 `UpstreamTrafficSetting` 配置时，要将 `UpstreamTrafficSetting` 资源指定为 `Egress` 配置中的匹配规则，同时要与 `Egress` 资源位于同一个命名空间中。这是为外部流量执行熔断限制时所必需的。因此，我们也要更新下之前应用的 `Egress` 配置添加一个 `matches` 字段。
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

6. 确认 `fortio` 客户端因为上面配置的连接数和请求数熔断限制无法成功发送同样数量的请求。
    ```console
    $ kubectl exec "$fortio_pod" -c fortio -n client -- /usr/bin/fortio load -c 5 -qps 0 -n 50 -loglevel Warning http://httpbin.httpbin.svc.cluster.local:14001/get
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

    如上所示，只有 58% 的请求成功，余下失败的是因为断路器打开。
    ```
    Code 200 : 29 (58.0 %)
    Code 503 : 21 (42.0 %)
    ```

7. 检查 `Envoy` sidecar 指标发现相应比例的请求的断路器打开。
    ```console
    $ osm proxy get stats $fortio_pod -n client | grep 'httpbin.*pending'
    cluster.httpbin_httpbin_svc_cluster_local_14001.circuit_breakers.default.remaining_pending: 1
    cluster.httpbin_httpbin_svc_cluster_local_14001.circuit_breakers.default.rq_pending_open: 0
    cluster.httpbin_httpbin_svc_cluster_local_14001.circuit_breakers.high.rq_pending_open: 0
    cluster.httpbin_httpbin_svc_cluster_local_14001.upstream_rq_pending_active: 0
    cluster.httpbin_httpbin_svc_cluster_local_14001.upstream_rq_pending_failure_eject: 0
    cluster.httpbin_httpbin_svc_cluster_local_14001.upstream_rq_pending_overflow: 21
    cluster.httpbin_httpbin_svc_cluster_local_14001.upstream_rq_pending_total: 29
    ```

    `cluster.httpbin_httpbin_svc_cluster_local_14001.upstream_rq_pending_overflow: 21` indicates that 21 requests tripped the circuit breaker, which matches the number of failed requests seen in the previous step: `Code 200 : 29 (58.0 %)`.
---
title: "网格外部服务的熔断"
description: "为网格外部服务配置熔断"
type: docs
weight: 22
---

本指南演示了如何为 OSM 托管服务网格的外部目标配置熔断。

## 先决条件

- Kubernetes 集群运行版本 {{< param min_k8s_version >}} 或者更高。
- 已安装 OSM。
- 使用 `kubectl` 与 API server 交互。
- 已安装 `osm` 命令行工具，用于管理服务网格。
- OSM 版本不低于 v1.1.0


## 演示

下面的演示展示了一个负载测试客户端 [fortio](https://github.com/fortio/fortio) 将流量发送到服务网格的外部 service `httpbin` 。发送到网格服务外的流量被认为是 [出口](/docs/guides/traffic_management/egress) 流量, 其会被  [出口流量策略](/docs/guides/traffic_management/egress/#1-configuring-egress-policies) 进行授权。我们将看到为外部 `httpbin` service 配置的流量熔断器被触发后，是如何影响 `fortio` 客户端。

1. 部署 `httpbin` service 到 `httpbin` 命名空间。`httpbin` service 运行在 `14001` 端口，且没有纳入网格管理，因此可以看成是网格外部的服务。

    ```bash
    # Create the httpbin namespace
    kubectl create namespace httpbin
    
    # Deploy httpbin service in the httpbin namespace
    kubectl apply -f https://raw.githubusercontent.com/openservicemesh/osm-docs/{{< param osm_branch >}}/manifests/samples/httpbin/httpbin.yaml -n httpbin
    ```

    确认 `httpbin` service 和 pod 启动并运行。

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

2. 将 `fortio` 负载测试客户端部署到 `client` 命名空间，并纳入服务网格。
    ```bash
    # Create the client namespace
    kubectl create namespace client
    
    # Add the namespace to the mesh
    osm namespace add client
    
    # Deploy fortio client in the client namespace
    kubectl apply -f https://raw.githubusercontent.com/openservicemesh/osm-docs/{{< param osm_branch >}}/manifests/samples/fortio/fortio.yaml -n client
    ```

    确认 `fortio` 客户端 pod 启动并运行

    ```console
    $ kubectl get pods -n client
    NAME                      READY   STATUS    RESTARTS   AGE
    fortio-6477f8495f-bj4s9   2/2     Running   0          19s
    ```

3. 配置 Egress 策略允许 `client` 命名空间下的 `fortio` 客户端可以与外部的 `httpbin` service 进行通信。HTTP 请求将被发送到域名 `httpbin.httpbin.svc.cluster.local` 的 `14001` 端口。
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

4. 确认 `fortio` 客户端可以成功发送 HTTP 请求到外部 service `httpbin.httpbin.svc.cluster.local` 的 `14001` 端口。使用 `5` 个并发（`-c 5`）发送 50 个请求（`-n 50`）到外部服务。
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

    如上所示，全部请求成功。
    ```
    Code 200 : 50 (100.0 %)
    ```

5. 使用 `UpstreamTrafficSetting` 资源为请求到外部域名 `httpbin.httpbin.svc.cluster.local` 的流量应用配置熔断器，并将最大并发连接数和请求数限制为`1`。为外部（出口）流量应用 `UpstreamTrafficSetting` 配置时，`UpstreamTrafficSetting` 资源还必须在 `Egress` 配置中指定为匹配项，并且与匹配的 `Egress` 资源属于同一命名空间。这是为外部流量执行熔断限制时所必需的。因此，我们也要在之前应用的 `Egress` 配置中更新添加一个 `matches` 字段。
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

6. 确认由于上面配置的连接和请求级别熔断限制，`fortio`客户端无法发出与以前相同数量的成功请求。

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

    如上所示，只有 58% 的请求成功，其余的请求在熔断器打开时失败

    ```
    Code 200 : 29 (58.0 %)
    Code 503 : 21 (42.0 %)
    ```

7. 检查 `Envoy` sidecar 统计信息以查看与触发断路器的请求有关的统计信息。

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

    `cluster.httpbin_httpbin_svc_cluster_local_14001.upstream_rq_pending_overflow: 21` 表示有 21 个请求触发了熔断器，这与上一步中看到的失败请求数相匹配：`Code 200 : 29 (58.0 %)`。
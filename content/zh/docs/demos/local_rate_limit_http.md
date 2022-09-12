---
title: "HTTP 请求的本地限速"
description: "配置 HTTP 请求的本地限速"
type: docs
weight: 23
---

本指南演示了如何为发往作为 OSM 托管服务网格的一部分的目标主机的 HTTP 请求配置速率限制。

## 前置条件

- Kubernetes 集群版本 {{< param min_k8s_version >}} 或者更高。
- 已安装 OSM。
- 已安装 `kubectl` 用来与 API 服务器进行交互。
- 已安装 `osm` CLI 用于管理服务网格。
- 已安装 OSM，版本 >= v1.2.0。

## 演示

下面的演示展示了一个客户端向 `fortio` 服务发送 HTTP 请求。我们将看到应用针对 `fortio` 服务的本地 HTTP 速率限制策略来控制发往服务后端的请求的吞吐量的影响。

1. 为简单起见，启用 [宽松流量策略模式](/docs/guides/traffic_management/permissive_mode) 以便网格内的应用程序连接不需要显式的 SMI 流量访问策略。
    ```bash
    export osm_namespace=osm-system # Replace osm-system with the namespace where OSM is installed
    kubectl patch meshconfig osm-mesh-config -n "$osm_namespace" -p '{"spec":{"traffic":{"enablePermissiveTrafficPolicyMode":true}}}'  --type=merge
    ```

1. 在将其命名空间注册到网格后，在 `demo` 命名空间中部署 `fortio` HTTP 服务。`fortio` HTTP 服务在端口 `8080` 上运行。
    ```bash
    # Create the demo namespace
    kubectl create namespace demo

    # Add the namespace to the mesh
    osm namespace add demo

    # Deploy fortio TCP echo in the demo namespace
    kubectl apply -f https://raw.githubusercontent.com/openservicemesh/osm-docs/{{< param osm_branch >}}/manifests/samples/fortio/fortio.yaml -n demo
    ```

    确认 `fortio` 服务的 pod 启动并运行。

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
    $ kubectl get pods -n demo
    NAME                            READY   STATUS    RESTARTS   AGE
    fortio-client-b9b7bbfb8-prq7r   2/2     Running   0          7s
    ```

1. 确认 `fortio-client` 应用程序能够成功地向端口 `8080` 上的 `fortio` HTTP 服务发出 HTTP 请求。我们用 `3` 并发连接 (`-c 3`) 调用 `fortio` 服务并发送 `10` 个请求 (`-n 10`)。
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

    如上所示，来自 `fortio-client` pod 的所有 HTTP 请求都成功了。
    ```
    Code 200 : 10 (100.0 %)
    ```

1. 接下来，应用本地速率限制策略将虚拟主机级别的 HTTP 请求速率限制为“每分钟 3 个请求”。
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

    通过检查 `fortio` 后端 pod 上的统计信息，确认没有任何 HTTP 请求受到速率限制。
    ```console
    $ fortio_server="$(kubectl get pod -n demo -l app=fortio -o jsonpath='{.items[0].metadata.name}')"

    $ osm proxy get stats "$fortio_server" -n demo | grep 'http_local_rate_limiter.http_local_rate_limit.rate_limited'
    http_local_rate_limiter.http_local_rate_limit.rate_limited: 0
    ```

1. 确认 HTTP 请求受速率限制。
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

    如上所示，10 个 HTTP 请求中只有 3 个成功，而其余的 7 个请求根据速率限制策略受到速率限制。
    ```
    Code 200 : 3 (30.0 %)
    Code 429 : 7 (70.0 %)
    ```

    检查统计数据以进一步证实这一点。
    ```console
    $ osm proxy get stats "$fortio_server" -n demo | grep 'http_local_rate_limiter.http_local_rate_limit.rate_limited'
    http_local_rate_limiter.http_local_rate_limit.rate_limited: 7
    ```

1. 接下来，更新我们的速率限制策略以允许请求突发。突发允许给定数量的请求超过我们的速率限制策略定义的每分钟 3 个请求的基准速率。
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

1. 确认突发功能允许在很短的时间窗口内突发请求。
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

    如上所示，所有 HTTP 请求都成功了，因为我们使用速率限制策略允许突发 10 个请求。
    ```
    Code 200 : 10 (100.0 %)
    ```

    此外，检查统计数据以确认突发允许其他请求通过。自从我们配置突发设置之前的上一次速率限制测试以来，被限速的请求数量没有增加。
    ```console
    $ osm proxy get stats "$fortio_server" -n demo | grep 'http_local_rate_limiter.http_local_rate_limit.rate_limited'
    http_local_rate_limiter.http_local_rate_limit.rate_limited: 7
    ```

1. 接下来，让我们为上游服务允许的特定 HTTP 路由配置速率限制策略。
     > 注意：由于我们在 demo 中使用的是宽松流量策略模式，因此上游后端允许使用通配符路径正则表达式 `.*` 的 HTTP 路由，因此我们将为此路由配置限速策略。但是，当在网格中使用 SMI 策略时，可以配置与匹配允许的 SMI HTTP 路由规则相对应的路径。

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

1. 确认 HTTP 请求在每个路由级别受到速率限制。
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

    如上所示，10 个 HTTP 请求中只有 3 个成功，而其余的 7 个请求根据速率限制策略受到速率限制。
    ```
    Code 200 : 3 (30.0 %)
    Code 429 : 7 (70.0 %)
    ```

    检查统计数据以进一步证实这一点。自从我们之前的测试以来，在配置 HTTP 路由级别速率限制后，`7` 个额外的请求受到了速率限制，由统计中的 `7` 个 HTTP 请求速率限制的总数表示。
    ```console
    $ osm proxy get stats "$fortio_server" -n demo | grep 'http_local_rate_limiter.http_local_rate_limit.rate_limited'
    http_local_rate_limiter.http_local_rate_limit.rate_limited: 14
    ```
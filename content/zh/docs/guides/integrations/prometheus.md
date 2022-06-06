---
title: "Prometheus 与 OSM 集成"
description: "演示如何将 OSM 与 Prometheus 集成用于指标采集"
aliases: "/docs/integrations/demo_prometheus"
type: docs
weight: 3
---

# Prometheus 与 OSM 集成

## Prometheus 与 OSM 集成

To familiarize yourself on how OSM works with Prometheus, try installing a new mesh with sample applications to see which metrics are collected. 为了熟悉 OSM 如何与 Promethues 工作，试着安装网格和示例应用来看收集了哪些指标。

1. 安装 OSM 并使用你自己的 Prometheus 实例：

   ```console
   $ osm install --set osm.deployPrometheus=true,osm.enablePermissiveTrafficPolicy=true
   OSM installed successfully in namespace [osm-system] with mesh name [osm]
   ```

1. 为示例工作负载创建命名空间：

   ```console
   $ kubectl create namespace metrics-demo
   namespace/metrics-demo created
   ```

1. 让 OSM 监控新创建的命名空间：

   ```console
   $ osm namespace add metrics-demo
   Namespace [metrics-demo] successfully added to mesh [osm]
   ```

1. 配置 Prometheus 从新的命名空间获取指标：

   ```console
   $ osm metrics enable --namespace metrics-demo
   Metrics successfully enabled in namespace [metrics-demo]
   ```

1. 安装示例应用：

   ```console
   $ kubectl apply -f https://raw.githubusercontent.com/openservicemesh/osm-docs/{{< param osm_branch >}}/manifests/samples/curl/curl.yaml -n metrics-demo
   serviceaccount/curl created
   deployment.apps/curl created
   $ kubectl apply -f https://raw.githubusercontent.com/openservicemesh/osm-docs/{{< param osm_branch >}}/manifests/samples/httpbin/httpbin.yaml -n metrics-demo
   serviceaccount/httpbin created
   service/httpbin created
   deployment.apps/httpbin created
   ```

   确保所有的 pod 和容器已经启动并运行：

   ```console
   $ kubectl get pods -n metrics-demo
   NAME                       READY   STATUS    RESTARTS   AGE
   curl-54ccc6954c-q8s89      2/2     Running   0          95s
   httpbin-8484bfdd46-vq98x   2/2     Running   0          72s
   ```

1. 生成流量：

   下面的命令让 curl Pod 来不停地以每秒1个请求的速度访问 httpbin Pod。

   ```console
   $ kubectl exec -n metrics-demo -ti "$(kubectl get pod -n metrics-demo -l app=curl -o jsonpath='{.items[0].metadata.name}')" -c curl -- sh -c 'while :; do curl -i httpbin.metrics-demo:14001/status/200; sleep 1; done'
   HTTP/1.1 200 OK
   server: envoy
   date: Tue, 23 Mar 2021 17:27:44 GMT
   content-type: text/html; charset=utf-8
   access-control-allow-origin: *
   access-control-allow-credentials: true
   content-length: 0
   x-envoy-upstream-service-time: 1

   HTTP/1.1 200 OK
   server: envoy
   date: Tue, 23 Mar 2021 17:27:45 GMT
   content-type: text/html; charset=utf-8
   access-control-allow-origin: *
   access-control-allow-credentials: true
   content-length: 0
   x-envoy-upstream-service-time: 2

   ...
   ```

1. 在 Prometheus 中查看指标：

    转发 Prometheus 端口：

   ```console
   $ kubectl port-forward -n osm-system $(kubectl get pods -n osm-system -l app=osm-prometheus -o jsonpath='{.items[0].metadata.name}') 7070
   Forwarding from 127.0.0.1:7070 -> 7070
   Forwarding from [::1]:7070 -> 7070
   ```

   在浏览器中访问 http://localhost:7070 查看 Prometheus 用户界面。下面的查询会显示 curl pod 每秒发送多少请求到 httpbin pod，应该是 1:

   ```
   irate(envoy_cluster_upstream_rq_xx{source_service="curl", envoy_cluster_name="metrics-demo/httpbin"}[30s])
   ```

   在 Prometheus 用户界面中可随意访问其他的指标。

1. 清理

   一旦演示资源使用完，通过删除应用命名空间来清理：

   ```console
   $ kubectl delete ns metrics-demo
   namespace "metrics-demo" deleted
   ```

   然后，卸载 OSM：

   ```
   $ osm uninstall mesh
   Uninstall OSM [mesh name: osm] ? [y/n]: y
   OSM [mesh name: osm] uninstalled
   ```

   在卸载 OSM 后删除集群范围的资源，执行下面的命令。参阅 [卸载指南](/docs/guides/uninstall/) 获取更多信息。

   ```console
   $ osm uninstall mesh --delete-cluster-wide-resources
   ```
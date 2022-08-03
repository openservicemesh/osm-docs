---
title: "使用 SMI 流量拆分进行金丝雀部署"
description: "使用 SMI 流量拆分管理金丝雀部署"
type: docs
weight: 21
---

本指南演示如何使用 SMI 流量拆分配置执行金丝雀部署。

## 先决条件

- Kubernetes 集群运行版本 {{< param min_k8s_version >}} 或者更高。
- 已安装 OSM。
- 使用 `kubectl` 与 API server 交互。
- 已安装 `osm` 命令行工具，用于管理服务网格。


## 演示

在此演示中，我们将部署一个 HTTP 应用程序并执行金丝雀发布，其中新版本部署的应用程序，会以一定百分比的流量提供服务。

为了拆分流量到多个服务后端，需要使用 [SMI 流量拆分 API](https://github.com/servicemeshinterface/smi-spec/blob/main/apis/traffic-split/v1alpha2/traffic-split.md) 。关于 API 使用的更多说明，可以查看 [流量拆分指南](/docs/guides/traffic_management/traffic_split)。为了将客户端流量透明地拆分到多个服务后端，客户端需要使用 `TrafficSplit` 资源中指定的根 service 的 FQDN 来发送流量。在该演示中，`curl` 客户端发送流量到根 service `httpbin.org`，该 service 初始化有 `v1` 版本，然后执行金丝雀发布，将部分流量定向到 `v2` 版本的 service 中。

以下步骤演示了金丝雀发布的部署策略。

> 注意：为了无需显式地创建访问控制策略，会启用[宽松流量策略模式](/docs/guides/traffic_management/permissive_mode) 。

1. 启用宽松模式

    ```bash
    osm_namespace=osm-system # Replace osm-system with the namespace where OSM is installed
    kubectl patch meshconfig osm-mesh-config -n "$osm_namespace" -p '{"spec":{"traffic":{"enablePermissiveTrafficPolicyMode":true}}}'  --type=merge
    ```

2. 在 `curl` 命名空间下部署 `curl` 客户端，并将命名空间纳入网格管理。

    ```bash
    # Create the curl namespace
    kubectl create namespace curl

    # Add the namespace to the mesh
    osm namespace add curl

    # Deploy curl client in the curl namespace
    kubectl apply -f https://raw.githubusercontent.com/openservicemesh/osm-docs/{{< param osm_branch >}}/manifests/samples/curl/curl.yaml -n curl
    ```

    确认 `curl` 客户端 pod 启动并运行。

    ```console
    $ kubectl get pods -n curl
    NAME                    READY   STATUS    RESTARTS   AGE
    curl-54ccc6954c-9rlvp   2/2     Running   0          20s
    ```

3. 为客户端的访问，创建根 service `httpbin`，service 选择器使用 `app: httpbin`。

    ```bash
    # Create the httpbin namespace
    kubectl create namespace httpbin

    # Add the namespace to the mesh
    osm namespace add httpbin

    # Create the httpbin root service and service account
    kubectl apply -f https://raw.githubusercontent.com/openservicemesh/osm-docs/{{< param osm_branch >}}/manifests/samples/canary/httpbin.yaml -n httpbin
    ```

4. 部署 `httpbin` service 的 `v1` 版本。该 service `httpbin-v1` 使用选择器 `app: httpbin, version: v1`，同时，该 deployment `httpbin-v1` 有标签 `app: httpbin, version: v1` ，其匹配了根 service `httpbin` 和 service `httpbin-v1` 的选择器。

    ```bash
    kubectl apply -f https://raw.githubusercontent.com/openservicemesh/osm-docs/{{< param osm_branch >}}/manifests/samples/canary/httpbin-v1.yaml -n httpbin
    ```

5. 创建 SMI TrafficSplit 资源，将流量定向到 `httpbin-v1`。

    ```bash
    kubectl apply -f - <<EOF
    apiVersion: split.smi-spec.io/v1alpha2
    kind: TrafficSplit
    metadata:
      name: http-split
      namespace: httpbin
    spec:
      service: httpbin.httpbin.svc.cluster.local
      backends:
      - service: httpbin-v1
        weight: 100
    EOF
    ```

7. 确认发往根 service FQDN `httpbin.httpbin.svc.cluster.local` 的所有流量都被路由到 `httpbin-v1` pod 中。可以通过检查 HTTP 响应头, 确认请求成功和 pod对应的展示相关信息到 `httpbin-v1` 。

    ```console
    for i in {1..10}; do kubectl exec -n curl -ti "$(kubectl get pod -n curl -l app=curl -o jsonpath='{.items[0].metadata.name}')" -c curl -- curl -sI http://httpbin.httpbin:14001/json | egrep 'HTTP|pod'; done
    HTTP/1.1 200 OK
    pod: httpbin-v1-77c99dccc9-q2gvt
    HTTP/1.1 200 OK
    pod: httpbin-v1-77c99dccc9-q2gvt
    HTTP/1.1 200 OK
    pod: httpbin-v1-77c99dccc9-q2gvt
    HTTP/1.1 200 OK
    pod: httpbin-v1-77c99dccc9-q2gvt
    HTTP/1.1 200 OK
    pod: httpbin-v1-77c99dccc9-q2gvt
    HTTP/1.1 200 OK
    pod: httpbin-v1-77c99dccc9-q2gvt
    HTTP/1.1 200 OK
    pod: httpbin-v1-77c99dccc9-q2gvt
    HTTP/1.1 200 OK
    pod: httpbin-v1-77c99dccc9-q2gvt
    HTTP/1.1 200 OK
    pod: httpbin-v1-77c99dccc9-q2gvt
    HTTP/1.1 200 OK
    pod: httpbin-v1-77c99dccc9-q2gvt
    ```

    上面的输出表明：全部 10 个请求都返回了 HTTP 200 OK，并且是 `httpbin-v1` pod 的响应。

8. 准备金丝雀发布 - 部署 `httpbin` 的 `v2` 版本。该 service `httpbin-v2` 使用选择器 `app: httpbin, version: v2`，同时，该 deployment 的标签 `app: httpbin, version: v2` 匹配根 service `httpbin` 和 service `httpbin-v2` 的选择器。

    ```bash
    kubectl apply -f https://raw.githubusercontent.com/openservicemesh/osm-docs/{{< param osm_branch >}}/manifests/samples/canary/httpbin-v2.yaml -n httpbin
    ```

9. 执行金丝雀发布,通过更新 SMI TrafficSplit 资源，将发往 FQDN `httpbin.httpbin.svc.cluster.local` 的流量拆分到 `httpbin-v1` 和 `httpbin-v2` service，分别对应 `httpbin` 的 `v1` 和 `v2` 版本。。

    ```bash
    kubectl apply -f - <<EOF
    apiVersion: split.smi-spec.io/v1alpha2
    kind: TrafficSplit
    metadata:
      name: http-split
      namespace: httpbin
    spec:
      service: httpbin.httpbin.svc.cluster.local
      backends:
      - service: httpbin-v1
        weight: 50
      - service: httpbin-v2
        weight: 50
    EOF
    ```

10. 确认流量按照后端服务指定的权重比例拆分。由于 `v1` 和 `v2` 都是 50 的权重，请求应该像下面一样负载均衡到两个版本。

    ```console
    $ for i in {1..10}; do kubectl exec -n curl -ti "$(kubectl get pod -n curl -l app=curl -o jsonpath='{.items[0].metadata.name}')" -c curl -- curl -sI http://httpbin.httpbin:14001/json | egrep 'HTTP|pod'; done
    HTTP/1.1 200 OK
    pod: httpbin-v2-6b48697db-cdqld
    HTTP/1.1 200 OK
    pod: httpbin-v1-77c99dccc9-q2gvt
    HTTP/1.1 200 OK
    pod: httpbin-v1-77c99dccc9-q2gvt
    HTTP/1.1 200 OK
    pod: httpbin-v1-77c99dccc9-q2gvt
    HTTP/1.1 200 OK
    pod: httpbin-v2-6b48697db-cdqld
    HTTP/1.1 200 OK
    pod: httpbin-v2-6b48697db-cdqld
    HTTP/1.1 200 OK
    pod: httpbin-v1-77c99dccc9-q2gvt
    HTTP/1.1 200 OK
    pod: httpbin-v2-6b48697db-cdqld
    HTTP/1.1 200 OK
    pod: httpbin-v2-6b48697db-cdqld
    HTTP/1.1 200 OK
    pod: httpbin-v1-77c99dccc9-q2gvt
    ```

   上面的结果显示所有 10 个请求都收到 HTTP 200 OK 的响应，基于 `TrafficSplit` 中配置的权重， `httpbin-v1` 和 `httpbin-v2` 各响应了 5 个请求。

---
title: "TCP 流量路由"
description: "设置 TCP 流量路由"
type: docs
weight: 20
---

本指南演示了使用 OSM 的 TCP 路由功能处理网格中通信的 TCP 客户端和服务端应用。

## 先决条件

- Kubernetes 集群版本 {{< param min_k8s_version >}} 或者更高。
- 已安装 OSM。
- 使用 `kubectl` 与 API server 交互。
- 已安装 `osm` 命令行工具，用于管理服务网格。

## 演示

以下演示了一个 TCP 客户端将数据发送到 tcp-echo 服务器，然后该服务器通过 TCP 连接将数据回显到客户端。

1. 设置安装 OSM 的命名空间。

    ```bash
    osm_namespace=osm-system  # Replace osm-system with the namespace where OSM is installed if different
    ```

2. 在 `tcp-demo` 命名空间下部署 `tcp-demo` 服务。 `tcp-demo` 服务运行在 `9000` 端口上，同时 `appProtocol` 字段设置为 `tcp` 用来告知 OSM 在定向 `tcp-demo` 在 `9000` 端口的流量时使用 TCP路由。
    ```bash
    # Create the tcp-demo namespace
    kubectl create namespace tcp-demo

    # Add the namespace to the mesh
    osm namespace add tcp-demo

    # Deploy the service
    kubectl apply -f https://raw.githubusercontent.com/openservicemesh/osm-docs/{{< param osm_branch >}}/manifests/apps/tcp-echo.yaml -n tcp-demo
    ```

    Confirm the `tcp-echo` service and pod is up and running.

    ```console
    $ kubectl get svc,po -n tcp-demo
    NAME               TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)    AGE
    service/tcp-echo   ClusterIP   10.0.216.68   <none>        9000/TCP   97s

    NAME                            READY   STATUS    RESTARTS   AGE
    pod/tcp-echo-6656b7c4f8-zt92q   2/2     Running   0          97s
    ```

3. 在 `curl` 命名空间下部署 `curl` 客户端。

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

### 使用宽松流量策略模式

We will enable service discovery using [permissive traffic policy mode](/docs/guides/traffic_management/permissive_mode), which allows application connectivity to be established without the need for explicit SMI policies.

我们将启用[宽松流量策略模式](/docs/guides/traffic_management/permissive_mode)允许应用互通而无需显式的 SMI 策略。


1. 开启宽松流量策略模式
    ```bash
    kubectl patch meshconfig osm-mesh-config -n "$osm_namespace" -p '{"spec":{"traffic":{"enablePermissiveTrafficPolicyMode":true}}}' --type=merge
    ```

2. 确认 `curl` 客户端可以通过 TCP 路由成功发送请求到 `tcp-echo` 服务并接收响应。
    ```console
    $ kubectl exec -n curl -ti "$(kubectl get pod -n curl -l app=curl -o jsonpath='{.items[0].metadata.name}')" -c curl -- sh -c 'echo hello | nc tcp-echo.tcp-demo 9000'
    echo response: hello
    ```

    `tcp-echo` 服务应该将客户端发送的数据返回客户端。在上面的示例中，客户端发送 `hellp`，`tcp-echo` 服务返回 `echo response: echo`。

### 使用 SMI 流量策略模式

When using SMI traffic policy mode, explicit traffic policies must be configured to allow application connectivity. We will set up SMI policies to allow the `curl` client to communicate with the `tcp-echo` service on port `9000`.
使用 SMI 流量策略模式时，必须配置显式的流量策略来允许应用互通。我们将设置 SMI 策略来允许 `curl` 客户端与 `tcp-echo` 的 `9000` 端口通讯。

1. 禁用宽松流量策略模式来启用 SMI 策略模式。
    ```bash
    kubectl patch meshconfig osm-mesh-config -n "$osm_namespace" -p '{"spec":{"traffic":{"enablePermissiveTrafficPolicyMode":false}}}' --type=merge
    ```

2. 确认在缺少 SMI 策略的情况下，`curl` 客户端无法发送请求到 `tcp-echo` 服务，也无法接收响应。
    ```console
    $ kubectl exec -n curl -ti "$(kubectl get pod -n curl -l app=curl -o jsonpath='{.items[0].metadata.name}')" -c curl -- sh -c 'echo hello | nc tcp-echo.tcp-demo 9000'
    command terminated with exit code 1
    ```

3. 配置 SMI 流量路由和访问策略。
    ```bash
    kubectl apply -f - <<EOF
    # TCP route to allows access to tcp-echo:9000
    apiVersion: specs.smi-spec.io/v1alpha4
    kind: TCPRoute
    metadata:
    name: tcp-echo-route
    namespace: tcp-demo
    spec:
    matches:
        ports:
        - 9000
    ---
    # Traffic target to allow curl app to access tcp-echo service using a TCPRoute
    kind: TrafficTarget
    apiVersion: access.smi-spec.io/v1alpha3
    metadata:
    name: tcp-access
    namespace: tcp-demo
    spec:
    destination:
        kind: ServiceAccount
        name: tcp-echo
        namespace: tcp-demo
    sources:
    - kind: ServiceAccount
        name: curl
        namespace: curl
    rules:
    - kind: TCPRoute
        name: tcp-echo-route
    EOF
    ```

4. 确认有了 SMI TCP 路由之后， `curl` 客户端可以成功访问 `tcp-echo` 服务并收到响应。
    ```console
    $ kubectl exec -n curl -ti "$(kubectl get pod -n curl -l app=curl -o jsonpath='{.items[0].metadata.name}')" -c curl -- sh -c 'echo hello | nc tcp-echo.tcp-demo 9000'
    echo response: hello
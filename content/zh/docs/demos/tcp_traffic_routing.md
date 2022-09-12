---
title: "TCP 流量路由"
description: "设置 TCP 流量路由"
type: docs
weight: 20
---

本指南演示了使用 OSM 的 TCP 路由功能处理网格中通信的 TCP 客户端和服务端应用。

## 先决条件

- Kubernetes 集群运行版本 {{< param min_k8s_version >}} 或者更高。
- 已安装 OSM。
- 使用 `kubectl` 与 API server 交互。
- 已安装 `osm` 命令行工具，用于管理服务网格。

## 演示

以下演示了一个 TCP 客户端将数据发送到 tcp-echo 服务器，然后该服务器通过 TCP 连接将数据回显到客户端。

1. 设置安装 OSM 的命名空间。

    ```bash
    osm_namespace=osm-system  # Replace osm-system with the namespace where OSM is installed if different
    ```

2. 在 `tcp-demo` 命名空间下部署 `tcp-demo` service 。 `tcp-demo` service 运行在 `9000` 端口上，同时 `appProtocol` 属性设置为 `tcp`， 用来告知 OSM： 在将流量定向到 `tcp-demo` 的 `9000` 端口时，必须使用 TCP 路由。
    ```bash
    # Create the tcp-demo namespace
    kubectl create namespace tcp-demo
    
    # Add the namespace to the mesh
    osm namespace add tcp-demo
    
    # Deploy the service
    kubectl apply -f https://raw.githubusercontent.com/openservicemesh/osm-docs/{{< param osm_branch >}}/manifests/apps/tcp-echo.yaml -n tcp-demo
    ```

    确认 `tcp-echo` service 和 pod 启动并运行。

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

我们将通过 [宽松流量策略模式](/docs/guides/traffic_management/permissive_mode) 启用服务发现，这允许应用互通建立而无需显式的指定 SMI 策略。


1. 开启宽松流量策略模式
    ```bash
    kubectl patch meshconfig osm-mesh-config -n "$osm_namespace" -p '{"spec":{"traffic":{"enablePermissiveTrafficPolicyMode":true}}}' --type=merge
    ```

2. 确认 `curl` 客户端可以通过 TCP 路由成功发送请求到 `tcp-echo` service 并接收响应。
    ```console
    $ kubectl exec -n curl -ti "$(kubectl get pod -n curl -l app=curl -o jsonpath='{.items[0].metadata.name}')" -c curl -- sh -c 'echo hello | nc tcp-echo.tcp-demo 9000'
    echo response: hello
    ```

    `tcp-echo` service 应该将客户端发送的数据返回客户端。在上面的示例中，客户端发送 `hello`，`tcp-echo` service 回应 `echo response: echo`。

### 使用 SMI 流量策略模式

使用 SMI 流量策略模式时，必须显式的配置流量策略来允许应用互通。我们将设置 SMI 策略来允许 `curl` 客户端与 `tcp-echo` service 在 `9000` 端口上进行通讯。

1. 通过禁用宽松流量策略模式来启用 SMI 策略模式。
    ```bash
    kubectl patch meshconfig osm-mesh-config -n "$osm_namespace" -p '{"spec":{"traffic":{"enablePermissiveTrafficPolicyMode":false}}}' --type=merge
    ```

2. 在缺少 SMI 策略的情况下，确认 `curl` 客户端无法发送请求到 `tcp-echo` service ，也无法接收响应。
    ```console
    $ kubectl exec -n curl -ti "$(kubectl get pod -n curl -l app=curl -o jsonpath='{.items[0].metadata.name}')" -c curl -- sh -c 'echo hello | nc tcp-echo.tcp-demo 9000'
    command terminated with exit code 1
    ```

3. 配置 SMI 流量访问和路由策略。
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

4. 有了 SMI TCP 路由之后，确认 `curl` 客户端可以成功访问 `tcp-echo` service 并收到响应。
    ```console
    $ kubectl exec -n curl -ti "$(kubectl get pod -n curl -l app=curl -o jsonpath='{.items[0].metadata.name}')" -c curl -- sh -c 'echo hello | nc tcp-echo.tcp-demo 9000'
    echo response: hello
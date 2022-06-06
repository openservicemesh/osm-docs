---
title: "出口策略"
description: "使用出口策略访问外部服务"
type: docs
weight: 15
---

本指南演示了服务网格中的客户端使用 OSM 的 Egress 策略 API 访问网格外部的目标。

## 先决条件

- Kubernetes 集群版本 {{< param min_k8s_version >}} 或者更高。
- 已安装 OSM。
- 已安装 `kubectl` 与 API server 交互。
- 已安装 `osm`  命令行工具，用于管理服务网格。


## 演示

1.  如果没有启用出口策略，将其开启。

    ```bash
    export osm_namespace=osm-system # Replace osm-system with the namespace where OSM is installed
    kubectl patch meshconfig osm-mesh-config -n "$osm_namespace" -p '{"spec":{"featureFlags":{"enableEgressPolicy":true}}}'  --type=merge
    ```

2. 在 `curl` 命名空间下部署 `curl` 客户端，并将该命名空间纳入网格中。

    ```bash
    # Create the curl namespace
    kubectl create namespace curl

    # Add the namespace to the mesh
    osm namespace add curl

    # Deploy curl client in the curl namespace
    kubectl apply -f https://raw.githubusercontent.com/openservicemesh/osm-docs/{{< param osm_branch >}}/manifests/samples/curl/curl.yaml -n curl
    ```

    确保 `curl` 客户端 pod 启动并运行。

    ```console
    $ kubectl get pods -n curl
    NAME                    READY   STATUS    RESTARTS   AGE
    curl-54ccc6954c-9rlvp   2/2     Running   0          20s
    ```

## HTTP 出口

1. 确认 `curl` 客户端无法发送请求 `http://httpbin.org:80/get` 到 `80` 端口的网站 `httpbin.org`。
    ```console
    $ kubectl exec $(kubectl get pod -n curl -l app=curl -o jsonpath='{.items..metadata.name}') -n curl -c curl -- curl -sI http://httpbin.org:80/get
    command terminated with exit code 7
    ```

2. 应用出口策略允许 `curl` 客户端的 ServiceAccount 通过 `http` 协议访问 `80` 端口的网站 `httpbin.org`。

    ```bash
    kubectl apply -f - <<EOF
    kind: Egress
    apiVersion: policy.openservicemesh.io/v1alpha1
    metadata:
      name: httpbin-80
      namespace: curl
    spec:
      sources:
      - kind: ServiceAccount
        name: curl
        namespace: curl
      hosts:
      - httpbin.org
      ports:
      - number: 80
        protocol: http
    EOF
    ```

3. 确认 `curl` 客户端可以成功请求 `http://httpbin.org:80/get`。

    ```console
    $ kubectl exec $(kubectl get pod -n curl -l app=curl -o jsonpath='{.items..metadata.name}') -n curl -c curl -- curl -sI http://httpbin.org:80/get
    HTTP/1.1 200 OK
    date: Thu, 13 May 2021 21:49:35 GMT
    content-type: application/json
    content-length: 335
    server: envoy
    access-control-allow-origin: *
    access-control-allow-credentials: true
    x-envoy-upstream-service-time: 168
    ```

4. 确认在删除上面的策略之后，`curl` 客户端无法再访问 `http://httpbin.org:80/get`。

    ```bash
    kubectl delete egress httpbin-80 -n curl
    ```

    ```console
    $ kubectl exec $(kubectl get pod -n curl -l app=curl -o jsonpath='{.items..metadata.name}') -n curl -c curl -- curl -sI http://httpbin.org:80/get
    command terminated with exit code 7
    ```

## HTTPS 出口

由于 HTTPS 流量通过 TLS 加密，OSM 将 HTTPS 流量时将其当作 TCP 路由给原始目的地。在 TSL 握手时客户端的服务器名称标识（SNI）与出口策略中指定的域名匹配。

1. 确认 `curl` 客户点无法发送 HTTPS 请求 `https://httpbin.org:443/get` 到运行在 `443` 端口的网站 `httpbin.org`。

    ```console
    $ kubectl exec $(kubectl get pod -n curl -l app=curl -o jsonpath='{.items..metadata.name}') -n curl -c curl -- curl -sI https://httpbin.org:443/get
    command terminated with exit code 7
    ```

2. 应用出口策略允许 `curl` 客户端的 ServiceAccount 通过 `https` 协议访问运行在 `443` 端口的 `httpbin.org`。
    ```bash
    kubectl apply -f - <<EOF
    kind: Egress
    apiVersion: policy.openservicemesh.io/v1alpha1
    metadata:
      name: httpbin-443
      namespace: curl
    spec:
      sources:
      - kind: ServiceAccount
        name: curl
        namespace: curl
      hosts:
      - httpbin.org
      ports:
      - number: 443
        protocol: https
    EOF
    ```

3. 确认 `curl` 客户端可以成功发送 HTTPS 请求 `https://httpbin.org:443/get`。

    ```console
    $ kubectl exec $(kubectl get pod -n curl -l app=curl -o jsonpath='{.items..metadata.name}') -n curl -c curl -- curl -sI https://httpbin.org:443/get
    HTTP/2 200
    date: Thu, 13 May 2021 22:09:36 GMT
    content-type: application/json
    content-length: 260
    server: gunicorn/19.9.0
    access-control-allow-origin: *
    access-control-allow-credentials: true
    ```

4. 确认在删除策略后，`curl` 客户端无法再发送 HTTPS 请求到 `https://httpbin.org:443/get`。
    ```bash
    kubectl delete egress httpbin-443 -n curl
    ```
    ```console
    $ kubectl exec $(kubectl get pod -n curl -l app=curl -o jsonpath='{.items..metadata.name}') -n curl -c curl -- curl -sI https://httpbin.org:443/get
    command terminated with exit code 7
    ```

## TCP 出口

基于 TCP 的出口流量与出口策略中指定的目的端口和 IP 地址范围匹配。如果没有指定 IP 地址范围，只会匹配与目的端口相同的流量。

1. 确认 `curl` 客户端无法发送 HTTPS 请求 `https://openservicemesh.io:443` 到运行在 `443` 端口的 `openservicemesh.io`。由于 HTTPS 底层使用 TCP 协议，基于 TCP 的路由需要隐式启用访问 HTTP(s) 的任何端口。
    ```console
    $ kubectl exec $(kubectl get pod -n curl -l app=curl -o jsonpath='{.items..metadata.name}') -n curl -c curl -- curl -sI https://openservicemesh.io:443
    command terminated with exit code 7
    ```

2. 应用出口策略，允许 `curl` 客户端的 ServiceAccount 可以通过 `tcp` 协议访问任何目的地的 `443` 端口。

    ```bash
    kubectl apply -f - <<EOF
    kind: Egress
    apiVersion: policy.openservicemesh.io/v1alpha1
    metadata:
      name: tcp-443
      namespace: curl
    spec:
      sources:
      - kind: ServiceAccount
        name: curl
        namespace: curl
      ports:
      - number: 443
        protocol: tcp
    EOF
    ```
   > 注意：对于像 `MySQL`、`PostgresSQL` 这样的 `server-first` 协议，服务器发起客户端和服务器之间的数据的第一个字节，协议必须设置为 `tcp-server-first` 指示 OSM 无需在端口上进行协议检测。协议检测依赖于检测连接的第一个字节，与 `server-first` 协议不兼容。当端口的协议设置为 `tcp-server-first` 时，会跳过该端口的协议检测。同样需要注意的 `server-first` 端口号不得用于需要执行协议检测的其他应用程序端口，这就意味着使用 `server-first` 协议的端口不得使用其他需要执行协议检测的如 `HTTP` 和 `TCP` 的协议。

3. 确认 `curl` 客户端可以成功发送 HTTPS 请求 `https://openservicemesh.io:443`。

    ```console
    $ kubectl exec $(kubectl get pod -n curl -l app=curl -o jsonpath='{.items..metadata.name}') -n curl -c curl -- curl -sI https://openservicemesh.io:443
    HTTP/2 200
    cache-control: public, max-age=0, must-revalidate
    content-length: 0
    content-type: text/html; charset=UTF-8
    date: Thu, 13 May 2021 22:35:07 GMT
    etag: "353ebaaf9573718bd1df6b817a472e47-ssl"
    strict-transport-security: max-age=31536000
    age: 0
    server: Netlify
    x-nf-request-id: 35a4f2dc-5356-45dc-9208-63e6fa162e0f-3350874
    ```

4. 当策略删除后，`curl` 客户端无法再访问 `https://openservicemesh.io:443`。

    ```bash
    kubectl delete egress tcp-443 -n curl
    ```
    ```console
    $ kubectl exec $(kubectl get pod -n curl -l app=curl -o jsonpath='{.items..metadata.name}') -n curl -c curl -- curl -sI https://openservicemesh.io:443
    command terminated with exit code 7
    ```

## 与 SMI 路由匹配的 HTTP 出口

HTTP Egress 策略可以为基于 HTTP 方法、请求头和路径的细粒度流量控制指定 SMI HTTPRouteGroup 匹配。

1. 确认 `curl` 客户端可以发送 HTTP 请求 `http://httpbin.org:80/get` 和 `http://httpbin.org:80/status/200` 到 `80` 端口的 `httpbin.org`。

    ```console
    $ kubectl exec $(kubectl get pod -n curl -l app=curl -o jsonpath='{.items..metadata.name}') -n curl -c curl -- curl -sI http://httpbin.org:80/get
    command terminated with exit code 7
    $ kubectl exec $(kubectl get pod -n curl -l app=curl -o jsonpath='{.items..metadata.name}') -n curl -c curl -- curl -sI http://httpbin.org:80/status/200
    command terminated with exit code 7
    ```

2. 应用 SMI HTTPRouteGroup 资源来允许访问 HTTP 路径 `/get`，应用出口策略来访问匹配 SMI HTTPRouteGroup 的 `80` 端口的 `httpbin.org`。

    ```bash
    kubectl apply -f - <<EOF
    apiVersion: specs.smi-spec.io/v1alpha4
    kind: HTTPRouteGroup
    metadata:
      name: egress-http-route
      namespace: curl
    spec:
      matches:
      - name: get
        pathRegex: /get
    ---
    kind: Egress
    apiVersion: policy.openservicemesh.io/v1alpha1
    metadata:
      name: httpbin-80
      namespace: curl
    spec:
      sources:
      - kind: ServiceAccount
        name: curl
        namespace: curl
      hosts:
      - httpbin.org
      ports:
      - number: 80
        protocol: http
      matches:
      - apiGroup: specs.smi-spec.io/v1alpha4
        kind: HTTPRouteGroup
        name: egress-http-route
    EOF
    ```

3. 确认 `curl` 客户端可以成功发送 HTTP 请求到 `http://httpbin.org:80/get`。

    ```console
    $ kubectl exec $(kubectl get pod -n curl -l app=curl -o jsonpath='{.items..metadata.name}') -n curl -c curl -- curl -sI http://httpbin.org:80/get
    HTTP/1.1 200 OK
    date: Thu, 13 May 2021 21:49:35 GMT
    content-type: application/json
    content-length: 335
    server: envoy
    access-control-allow-origin: *
    access-control-allow-credentials: true
    x-envoy-upstream-service-time: 168
    ```

4. 确认 `curl` 客户端无法发送请求到  `http://httpbin.org:80/status/200`。

    ```console
    $ kubectl exec $(kubectl get pod -n curl -l app=curl -o jsonpath='{.items..metadata.name}') -n curl -c curl -- curl -sI http://httpbin.org:80/status/200
    HTTP/1.1 404 Not Found
    date: Fri, 14 May 2021 17:08:48 GMT
    server: envoy
    transfer-encoding: chunked
    ```

4. 更新 SMI HTTPRouteGroup 资源允许请求匹配正则 `/status*` 的 HTTP 路径。

    ```bash
    kubectl apply -f - <<EOF
    apiVersion: specs.smi-spec.io/v1alpha4
    kind: HTTPRouteGroup
    metadata:
      name: egress-http-route
      namespace: curl
    spec:
      matches:
      - name: get
        pathRegex: /get
      - name: status
        pathRegex: /status.*
    EOF
    ```

5. 确认 `curl` 客户端此时可以访问 `http://httpbin.org:80/status/200`。

    ```console
    $ kubectl exec $(kubectl get pod -n curl -l app=curl -o jsonpath='{.items..metadata.name}') -n curl -c curl -- curl -sI http://httpbin.org:80/status/200
    HTTP/1.1 200 OK
    date: Fri, 14 May 2021 17:10:48 GMT
    content-type: text/html; charset=utf-8
    content-length: 0
    server: envoy
    access-control-allow-origin: *
    access-control-allow-credentials: true
    x-envoy-upstream-service-time: 188
    ```

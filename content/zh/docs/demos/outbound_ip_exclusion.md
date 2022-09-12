---
title: "出站流量 IP 范围排除"
description: "从 Sidecar 拦截中排除出站流量的 IP 地址范围"
type: docs
weight: 5
---

本指南演示如何将出站 IP 地址范围从 OSM 代理 sidecar 的拦截中排除，以便它们不受服务网格过滤和路由策略的约束。

## 先决条件

- Kubernetes 集群运行版本 {{< param min_k8s_version >}} 或者更高。
- 已安装 OSM。
- 使用 `kubectl` 与 API server 交互。
- 已安装 `osm` 命令行工具，用于管理服务网格。


## 演示

以下演示展示了一个 HTTP `curl` 客户端 直接使用 IP 地址来向 `httpbin.org` 网站发送 HTTP 请求。我们将明确禁用出口功能，来确保访问非网格目的地的流量（本演示中的 `httpbin.org`）无法访问 pod 。

1. 禁用网格范围的出口直通。

    ```bash
    export osm_namespace=osm-system # Replace osm-system with the namespace where OSM is installed
    kubectl patch meshconfig osm-mesh-config -n "$osm_namespace" -p '{"spec":{"traffic":{"enableEgress":false}}}'  --type=merge
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

3. 获取 `httpbin.org` 网站的公共 IP。为了演示，我们将测试在流量拦截中排除单个 IP 范围。在这个示例中，我们将使用 IP 地址段 `54.91.118.50/32` 来表示 `54.91.118.50`，在配置和不配置排除出站 IP 范围的情况下，发送 HTTP 请求。

    ```console
    $ nslookup httpbin.org
    Server:		172.23.48.1
    Address:	172.23.48.1#53

    Non-authoritative answer:
    Name:	httpbin.org
    Address: 54.91.118.50
    Name:	httpbin.org
    Address: 54.166.163.67
    Name:	httpbin.org
    Address: 34.231.30.52
    Name:	httpbin.org
    Address: 34.199.75.4
    ```

    > 注意：将 `54.91.118.50` 替换为上面命令行返回的有效 IP 地址。

4. 确认 `curl` 客户端无法发送请求到在 `http://54.91.118.50:80`上运行的 `httpbin.org` 网站。

    ```console
    $ kubectl exec -n curl -ti "$(kubectl get pod -n curl -l app=curl -o jsonpath='{.items[0].metadata.name}')" -c curl -- curl -I http://54.91.118.50:80
    curl: (7) Failed to connect to 54.91.118.50 port 80: Connection refused
    command terminated with exit code 7
    ```

    上述错误是意料之中的，因为默认情况下，出站流量通过运行在 `curl` 客户端 pod 上的 Envoy 代理 sidecar 重定向，并且代理将遵循服务网格策略：不允许此流量访问。

5. 配置 OSM 排除 IP 地址范围 `54.91.118.50/32`。

    ```bash
    kubectl patch meshconfig osm-mesh-config -n "$osm_namespace" -p '{"spec":{"traffic":{"outboundIPRangeExclusionList":["54.91.118.50/32"]}}}'  --type=merge
    ```

6. 确认网格配置 MeshConfig 已按预期更新。
    ```console
    # 54.91.118.50 is one of the IP addresses of httpbin.org
    $ kubectl get meshconfig osm-mesh-config -n "$osm_namespace" -o jsonpath='{.spec.traffic.outboundIPRangeExclusionList}{"\n"}'
    ["54.91.118.50/32"]
    ```

7. 重启 `curl` 客户端 pod 让出口 IP 地址排除配置生效。这里需要注意，因为流量拦截规则是通过初始化容器在 pod 创建阶段写入的，现有的 pod 必须重启才能生效。

    ```bash
    kubectl rollout restart deployment curl -n curl
    ```

    等待 pod 重启并运行。

8. 确认 `curl` 客户端可以成功发送请求到在 `http://54.91.118.50:80` 上运行的 `httpbin.org` 网站。

    ```console
    # 54.91.118.50 is one of the IP addresses for httpbin.org
    $ kubectl exec -n curl -ti "$(kubectl get pod -n curl -l app=curl -o jsonpath='{.items[0].metadata.name}')" -c curl -- curl -I http://54.91.118.50:80
    HTTP/1.1 200 OK
    Date: Thu, 18 Mar 2021 23:17:44 GMT
    Content-Type: text/html; charset=utf-8
    Content-Length: 9593
    Connection: keep-alive
    Server: gunicorn/19.9.0
    Access-Control-Allow-Origin: *
    Access-Control-Allow-Credentials: true
    ```

9. 其他没有被排除的 `httpbin.org` IP 地址， 确认其无法被访问。

    ```console
    # 34.199.75.4 is one of the IP addresses for httpbin.org
    $ kubectl exec -n curl -ti "$(kubectl get pod -n curl -l app=curl -o jsonpath='{.items[0].metadata.name}')" -c curl -- curl -I http://34.199.75.4:80
    curl: (7) Failed to connect to 34.199.75.4 port 80: Connection refused
    command terminated with exit code 7
    ```
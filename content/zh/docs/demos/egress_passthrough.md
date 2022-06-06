---
title: "到未知目的地的出口透传"
description: "在没有 Egress 策略的情况下访问外部服务"
type: docs
weight: 16
---

本指南演示了服务网格内的客户端使用 OSM 的 Egress 功能访问外部的目标，以便在没有 Egress 策略的情况下将流量传递到未知目标。

## 先决条件

- Kubernetes 集群版本 {{< param min_k8s_version >}} 或者更高。
- 已安装 OSM。
- 使用 `kubectl` 与 API server 交互。
- 已安装 `osm` 命令行工具，用于管理服务网格。

## HTTP(S) 网格出口的透传演示

1. 如果没有启用全局出口透传，将其开启。

    ```bash
    export osm_namespace=osm-system # Replace osm-system with the namespace where OSM is installed
    kubectl patch meshconfig osm-mesh-config -n "$osm_namespace" -p '{"spec":{"traffic":{"enableEgress":true}}}'  --type=merge
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

    确保 `curl` 客户端 pod 就绪并运行。

    ```console
    $ kubectl get pods -n curl
    NAME                    READY   STATUS    RESTARTS   AGE
    curl-54ccc6954c-9rlvp   2/2     Running   0          20s
    ```

3. 确认 `curl` 客户端可以成功发送 HTTPS 请求到 `httpbin.org` 的 `443` 端口。

    ```console
    $ kubectl exec -n curl -ti "$(kubectl get pod -n curl -l app=curl -o jsonpath='{.items[0].metadata.name}')" -c curl -- curl -I https://httpbin.org:443
    HTTP/2 200
    date: Tue, 16 Mar 2021 22:19:00 GMT
    content-type: text/html; charset=utf-8
    content-length: 9593
    server: gunicorn/19.9.0
    access-control-allow-origin: *
    access-control-allow-credentials: true
    ```

    `200 OK` 响应说明 `curl` 客户端发往 `httpbin.org` 的请求成功了。

4. 确认当网格出口禁用时 HTTPS 请求失败。

    ```bash
    kubectl patch meshconfig osm-mesh-config -n "$osm_namespace" -p '{"spec":{"traffic":{"enableEgress":false}}}'  --type=merge
    ```
    ```console
    $ kubectl exec -n curl -ti "$(kubectl get pod -n curl -l app=curl -o jsonpath='{.items[0].metadata.name}')" -c curl -- curl -I https://httpbin.org:443
	  curl: (7) Failed to connect to httpbin.org port 443 after 3 ms: Connection refused
	  command terminated with exit code 7
    ```

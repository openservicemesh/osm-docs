---
title: "重试策略"
description: "使用重试提高服务可用性"
type: docs
weight: 15
---

本指南演示了如何为服务网格中的客户端和服务器应用程序配置重试策略。

## 前置条件

- Kubernetes 集群版本 {{< param min_k8s_version >}} 或者更高。
- 已安装 OSM，版本 >= v1.2.0。
- 已安装 `osm` CLI 用于管理服务网格。

## 演示

1. 安装 OSM 时开启宽松模式和重试策略。
    ```bash
    osm install --set=osm.enablePermissiveTrafficPolicy=true --set=osm.featureFlags.enableRetryPolicy=true 
    ```

1. 在将 `httpbin` 命名空间加入到网格后，部署 `httpbin` 服务。`httpbin` 运行在 `14001` 端口。

    ```bash
    kubectl create namespace httpbin

    osm namespace add httpbin

    kubectl apply -f https://raw.githubusercontent.com/openservicemesh/osm-docs/{{< param osm_branch >}}/manifests/samples/httpbin/httpbin.yaml -n httpbin
    ```

    确认 `httpbin` pod 启动并运行。

    ```bash
    kubectl get svc,pod -n httpbin
    ```
    
    类似下面：
    
    ```console
    NAME      TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)     AGE
    httpbin   ClusterIP   10.96.198.23   <none>        14001/TCP   20s

    NAME                     READY   STATUS    RESTARTS   AGE
    httpbin-5b8b94b9-lt2vs   2/2     Running   0          20s
    ```
    
1. 在将 `curl` 命名空间加入网格后，部署 `curl` 服务到命名空间下。
    ```bash 
    kubectl create namespace curl

    osm namespace add curl

    kubectl apply -f https://raw.githubusercontent.com/openservicemesh/osm-docs/{{< param osm_branch >}}/manifests/samples/curl/curl.yaml -n curl
    ```

    确认 `curl` pod 启动并运行。

    ```bash
    kubectl get pods -n curl
    ```
    
    类似下面：
    
    ```console
    NAME                    READY   STATUS    RESTARTS   AGE
    curl-54ccc6954c-9rlvp   2/2     Running   0          20s
     ```

1. 应用重试策略，当 `curl` ServiceAccount 发送请求到 `httpbin` 服务并收到 `5xx` 错误码时进行重试。
    ```bash
    kubectl apply -f - <<EOF
    kind: Retry
    apiVersion: policy.openservicemesh.io/v1alpha1
    metadata:
      name: retry
      namespace: curl
    spec:
      source:
        kind: ServiceAccount
        name: curl
        namespace: curl
      destinations:
      - kind: Service
        name: httpbin
        namespace: httpbin
      retryPolicy:
        retryOn: "5xx"
        perTryTimeout: 1s
        numRetries: 5
        retryBackoffBaseInterval: 1s
    EOF
    ```

1. 从 `curl` pod 发送一个会返回 `503` 响应的请求到 `httpbin` 服务。
    ```bash
    kubectl exec deploy/curl -n curl -c curl -- curl -sI httpbin.httpbin.svc.cluster.local:14001/status/503
    ```

1. 在新终端会话中，执行下面的命令转发 `curl` pod 端口。
    ```bash
    kubectl port-forward deploy/curl -n curl 15000
    ```

1. 查看 `curl` 和 `httpbin` 间的统计数据。
    ```bash
    curl -s localhost:15000/stats | grep "cluster.httpbin/httpbin|14001.upstream_rq_retry"
    ```
    从 `curl` pod 到 `httpbin` pod 的请求使用指数退避重试的次数应等于重试策略中的 `numRetries` 字段。
     `upstream_rq_retry_limit_exceeded` 统计数据显示未重试的请求数，因为它超过了允许的最大重试次数 - `numRetries`。
    
     ```console
    cluster.httpbin/httpbin|14001.upstream_rq_retry: 5
    cluster.httpbin/httpbin|14001.upstream_rq_retry_backoff_exponential: 5
    cluster.httpbin/httpbin|14001.upstream_rq_retry_backoff_ratelimited: 0
    cluster.httpbin/httpbin|14001.upstream_rq_retry_limit_exceeded: 1
    cluster.httpbin/httpbin|14001.upstream_rq_retry_overflow: 0
    cluster.httpbin/httpbin|14001.upstream_rq_retry_success: 0
    ```

1. 从 `curl` pod 向 `httpbin` 服务发送一个返回非 5xx 状态码的 HTTP 请求。
    ```bash
    kubectl exec deploy/curl -n curl -c curl -- curl -sI httpbin.httpbin.svc.cluster.local:14001/status/404
    ```

1. 指标不会增加，因为重试策略设置为在 `5xx` 时重试

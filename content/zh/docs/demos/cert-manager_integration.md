---
title: "Cert-manager 证书提供者"
description: "Using cert-manager as a certificate provider"
type: docs
weight: 30
---

本指南演示了如何使用 [cert-manager][1] 作为证书提供程序在 OSM 中管理和颁发证书。

## 先决条件

- Kubernetes 集群版本 {{< param min_k8s_version >}} 或者更高。
- 使用 `kubectl` 与 API server 交互。
- 已安装 `osm` 或者 `Helm 3` 命令行工具，用于安装 OSM 和 Contour。


## 演示

下面的演示使用 [cert-manager][1] 作为证书提供者，向 OSM 托管服务网格中的通过 `Mutual TLS (mTLS)` 通信的 `curl` 和 `httpbin` 应用颁发证书。

1. 安装 `cert-manager`。演示中使用 `cert-manager v1.6.1`。
    ```bash
    kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.6.1/cert-manager.yaml
    ```

    确保 pod 已经就绪并运行在 `cert-manager` 命名空间下。
    
    ```console
    kubectl get pod -n cert-manager
    NAME                                      READY   STATUS    RESTARTS   AGE
    cert-manager-55658cdf68-pdnzg             1/1     Running   0          2m33s
    cert-manager-cainjector-967788869-prtjq   1/1     Running   0          2m33s
    cert-manager-webhook-6668fbb57d-vzm4j     1/1     Running   0          2m33s
    ```

1. 为 `cert-manager` 配置颁发证书所需 `cert-manager` `Issuer` 和 `Certificate` 资源。 这些资源部署在随后安装 OSM 的同一个命名空间中。
    > 注意：必须先安装 `cert-manager` 并保证 issuer 就绪，随后将 `cert-manager` 作为证书提供者安装 OSM。

    创建用于安装 OSM 的命名空间。
    
    ```bash
    export osm_namespace=osm-system # Replace osm-system with the namespace where OSM is installed
    kubectl create namespace "$osm_namespace"
    ```

    接下来，我们使用 `SelfSigned` 的颁发者来引导自定义证书。这就会创建一个 `SelfSigned` 颁发者，颁发根证书，将其作为在网格中颁发的证书的 `CA` 颁发者
    
    ```bash
    # Create Issuer and Certificate resources
    kubectl apply -f - <<EOF
    apiVersion: cert-manager.io/v1
    kind: Issuer
    metadata:
      name: selfsigned
      namespace: "$osm_namespace"
    spec:
      selfSigned: {}
    ---
    apiVersion: cert-manager.io/v1
    kind: Certificate
    metadata:
      name: osm-ca
      namespace: "$osm_namespace"
    spec:
      isCA: true
      duration: 87600h # 365 days
      secretName: osm-ca-bundle
      commonName: osm-system
      issuerRef:
        name: selfsigned
        kind: Issuer
        group: cert-manager.io
    ---
    apiVersion: cert-manager.io/v1
    kind: Issuer
    metadata:
      name: osm-ca
      namespace: "$osm_namespace"
    spec:
      ca:
        secretName: osm-ca-bundle
    EOF
    ```

1. 确保 `cert-manager` 在 OSM 命名空间下创建了 `osm-ca-bundle` secret。
2. 
    ```console
    $ kubectl get secret osm-ca-bundle -n "$osm_namespace"
    NAME            TYPE                DATA   AGE
    osm-ca-bundle   kubernetes.io/tls   3      84s
    ```

    OSM 在安装时将使用保存在这个 secret 中的 CA 证书来引导其证书提供者程序。

3. 安装 OSM 时，指定其证书提供者为 `cert-manager`。
    ```bash
    osm install --set osm.certificateProvider.kind="cert-manager"
    ```

    确保 OSM 控制平面 pod 就绪并运行。
    ```console
    $ kubectl get pod -n "$osm_namespace"
    NAME                              READY   STATUS    RESTARTS   AGE
    osm-bootstrap-7ddc6f9b85-k8ptp    1/1     Running   0          2m52s
    osm-controller-79b777889b-mqk4g   1/1     Running   0          2m52s
    osm-injector-5f96468fb7-p77ps     1/1     Running   0          2m52s
    ```

4. 启用宽松流量策略模式来设置自动应用程序连接。
   > 注意：这并不是使用 `cert-manager` 时必须的配置，而是为了无需为应用连接明确流量策略来简化演示。

    ```bash
    kubectl patch meshconfig osm-mesh-config -n "$osm_namespace" -p '{"spec":{"traffic":{"enablePermissiveTrafficPolicyMode":true}}}'  --type=merge
    ```

5. 将 `httpbin` 命名空间纳入网格后，在该命名空间下部署 `httpbin` 服务。 这个服务在 `14001` 端口上运行。

    ```bash
    # Create the httpbin namespace
    kubectl create namespace httpbin

    # Add the namespace to the mesh
    osm namespace add httpbin

    # Deploy httpbin service in the httpbin namespace
    kubectl apply -f https://raw.githubusercontent.com/openservicemesh/osm-docs/{{< param osm_branch >}}/manifests/samples/httpbin/httpbin.yaml -n httpbin
    ```

    确保 `httpbin` 服务和 pod 已经就绪并运行。

    ```console
    $ kubectl get svc -n httpbin
    NAME      TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)     AGE
    httpbin   ClusterIP   10.96.198.23   <none>        14001/TCP   20s
    ```

    ```console
    $ kubectl get pods -n httpbin
    NAME                     READY   STATUS    RESTARTS   AGE
    httpbin-5b8b94b9-lt2vs   2/2     Running   0          20s
    ```

6. 在 `curl` 命名空间部署 `curl` 客户端，该命名空间同样要纳入网格中。

    ```bash
    # Create the curl namespace
    kubectl create namespace curl

    # Add the namespace to the mesh
    osm namespace add curl

    # Deploy curl client in the curl namespace
    kubectl apply -f https://raw.githubusercontent.com/openservicemesh/osm-docs/{{< param osm_branch >}}/manifests/samples/curl/curl.yaml -n curl
    ```

    确认 `client` 客户端 pod 就绪并运行。

    ```console
    $ kubectl get pods -n curl
    NAME                    READY   STATUS    RESTARTS   AGE
    curl-54ccc6954c-9rlvp   2/2     Running   0          20s
    ```

7. 确认 `curl` 客户端可以访问 `httpbin` 的 `14001` 端口。

    ```console
    $ kubectl exec -n curl -ti "$(kubectl get pod -n curl -l app=curl -o jsonpath='{.items[0].metadata.name}')" -c curl -- curl -I http://httpbin.httpbin:14001
    HTTP/1.1 200 OK
    server: envoy
    date: Mon, 15 Mar 2021 22:45:23 GMT
    content-type: text/html; charset=utf-8
    content-length: 9593
    access-control-allow-origin: *
    access-control-allow-credentials: true
    x-envoy-upstream-service-time: 2
    ```

    `200 OK` 相应表明来自 `curl` 客户端发送到 `httpbin` 服务的请求成功了。应用 sidecar 代理间的流量是加密的，并使用 `cert-manager` 证书提供者颁发的证书完成 `双向 TLS（mTLS）`认证。


[1]: https://cert-manager.io/
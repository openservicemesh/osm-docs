---
title: "使用 Kubernetes Nginx 入口控制器的入口"
description: "使用 Kubernetes Nginx 入口控制器的 HTTP 和 HTTPS 入口"
type: docs
weight: 11
---

本指南将演示如何在使用 [Kubernetes Nginx 入口控制器](https://kubernetes.github.io/ingress-nginx/) 时将 HTTP 和 HTTPS 入口配置到 OSM 托管服务网格的部分服务。

## 先决条件

- Kubernetes cluster running Kubernetes {{< param min_k8s_version >}} or greater.
- Have `kubectl` available to interact with the API server.
- Have OSM version >= v0.10.0 installed.
- Have Kubernetes Nginx Ingress Controller installed. Refer to the [deployment guide](https://kubernetes.github.io/ingress-nginx/deploy/) to install it.
- Kubernetes 集群版本 {{< param min_k8s_version >}} 或者更高。
- 使用 `kubectl` 与 API server 交互。
- 安装的 OSM 版本不低于 v0.10.0。
- 已安装 `osm`  命令行工具，用于管理服务网格。

## 演示

首先，注意 OSM 和 Nginx 入口控制器安装的相关细节：

```bash
osm_namespace=osm-system # Replace osm-system with the namespace where OSM is installed
osm_mesh_name=osm # replace osm with the mesh name (use `osm mesh list` command)

nginx_ingress_namespace=<nginx-namespace> # replace <nginx-namespace> with the namespace where Nginx is installed
nginx_ingress_service=<nginx-ingress-controller-service> # replace <nginx-ingress-controller-service> with the name of the nginx ingress controller service
nginx_ingress_host="$(kubectl -n "$nginx_ingress_namespace" get service "$nginx_ingress_service" -o jsonpath='{.status.loadBalancer.ingress[0].ip}')"
nginx_ingress_port="$(kubectl -n "$nginx_ingress_namespace" get service "$nginx_ingress_service" -o jsonpath='{.spec.ports[?(@.name=="http")].port}')"
```

为了将后端的入口流量限制到授权客户端，我们将设置 IngressBackend 配置，以便只有来自 Nginx 入口控制器服务端点的入口流量才能将流量路由到服务后端。为了能够发现该服务的端点，我们需要 OSM 控制器来监控相应的命名空间。 但是，Nginx 正常运行不能注入 Envoy sidecar。

```bash
osm namespace add "$nginx_ingress_namespace" --mesh-name "$osm_mesh_name" --disable-sidecar-injection
```

接下来是部署 `httpbin` 示例服务。

```bash
# Create a namespace
kubectl create ns httpbin

# Add the namespace to the mesh
osm namespace add httpbin

# Deploy the application
kubectl apply -f https://raw.githubusercontent.com/openservicemesh/osm-docs/{{< param osm_branch >}}/manifests/samples/httpbin/httpbin.yaml -n httpbin
```

确认 `httpbin` 客户端 pod 启动并运行。

```console
$ kubectl get pods -n httpbin
NAME                       READY   STATUS    RESTARTS   AGE
httpbin-74677b7df7-zzlm2   2/2     Running   0          11h

$ kubectl get svc -n httpbin
NAME      TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)     AGE
httpbin   ClusterIP   10.0.22.196   <none>        14001/TCP   11h
```

### HTTP 入口

下一步，我们将创建必要的 Ingress 和 IngressBackend 配置来允许外部的客户端访问 `httpbin` 命名空间下 `httpbin` 服务的 `14001` 端口。因为没有使用 TLS，Nginx 入口服务到 `httpbin` 后端 pod 的连接无需加密。

```bash
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: httpbin
  namespace: httpbin
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: httpbin
            port:
              number: 14001
---
kind: IngressBackend
apiVersion: policy.openservicemesh.io/v1alpha1
metadata:
  name: httpbin
  namespace: httpbin
spec:
  backends:
  - name: httpbin
    port:
      number: 14001 # targetPort of httpbin service
      protocol: http
  sources:
  - kind: Service
    namespace: "$nginx_ingress_namespace"
    name: "$nginx_ingress_service"
EOF
```

现在，我们期望外部的客户端可以通过 HTTP 的方式访问 `httpbin` 服务：
```console
$ curl -sI http://"$nginx_ingress_host":"$nginx_ingress_port"/get
HTTP/1.1 200 OK
Date: Wed, 18 Aug 2021 18:12:35 GMT
Content-Type: application/json
Content-Length: 366
Connection: keep-alive
access-control-allow-origin: *
access-control-allow-credentials: true
x-envoy-upstream-service-time: 2
```

### HTTPS 出口 (mTLS 和 TLS)

为了将连接代理到 HTTPS 后端，我们将在 Ingress 和 IngressBackend 配置中指定使用 `https` 作为后端协议，同时由 OSM 颁发证书，Nginx 将使用该证书作为客户端证书将 HTTPS 连接代理到 TLS 后端。客户端证书和 CA 证书将保存在 Kubernetes secret 中，Nginx 将使用该证书来认证服务网格后端。

需要更新 `osm-mesh-config` `MeshConfig` 资源来为 Nginx 入口 service 颁发证书。

```bash
kubectl edit meshconfig osm-mesh-config -n "$osm_namespace"
```

在 `spec.certificate` 下添加字段 `ingressGateway` :

```yaml
certificate:
  ingressGateway:
    secret:
      name: osm-nginx-client-cert
      namespace: <osm-namespace> # replace <osm-namespace> with the namespace where OSM is installed
    subjectAltNames:
    - ingress-nginx.ingress-nginx.cluster.local
    validityDuration: 24h
```
> 注意：主体备用名称（Subject Alternative Name，SAN）使用类似 `<service-account>.<namespace>.cluster.local` 格式，sevice account 和 namespace 使用 Nginx service 的。

接下来，我们需要创建一个 Ingress 和 IngressBackend 配置以使用 TLS 代理到后端服务，同时通过 mTLS 启用到后端的代理。为此，我们必须创建一个 IngressBackend 资源，指定指向 `httpbin` 服务的 HTTPS 入口流量只能接受来自受信任客户端的流量。OSM 使用 主体备用名称（Subject Alternative Name，SAN）`ingress-nginx.ingress-nginx.cluster.local` 为 Nginx 入口服务提供了一个客户端证书，因此 IngressBackend 配置需要引用相同的 SAN 用于 Nginx 入口服务和 `httpbin` 后端之间的 mTLS 身份验证。

应用配置：

```bash
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: httpbin
  namespace: httpbin
  annotations:
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
    # proxy_ssl_name for a service is of the form <service-account>.<namespace>.cluster.local
    nginx.ingress.kubernetes.io/configuration-snippet: |
      proxy_ssl_name "httpbin.httpbin.cluster.local";
    nginx.ingress.kubernetes.io/proxy-ssl-secret: "osm-system/osm-nginx-client-cert"
    nginx.ingress.kubernetes.io/proxy-ssl-verify: "on"
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: httpbin
            port:
              number: 14001
---
apiVersion: policy.openservicemesh.io/v1alpha1
kind: IngressBackend
metadata:
  name: httpbin
  namespace: httpbin
spec:
  backends:
  - name: httpbin
    port:
      number: 14001 # targetPort of httpbin service
      protocol: https
    tls:
      skipClientCertValidation: false
  sources:
  - kind: Service
    name: "$nginx_ingress_service"
    namespace: "$nginx_ingress_namespace"
  - kind: AuthenticatedPrincipal
    name: ingress-nginx.ingress-nginx.cluster.local
EOF
```

现在，我们期望外部客户端能够通过入口网管和服务后端之间的 mTLS 上的 HTTPS 代理访问 `httpbin` 服务。
```console
$ curl -sI http://"$nginx_ingress_host":"$nginx_ingress_port"/get
HTTP/1.1 200 OK
Date: Wed, 18 Aug 2021 18:12:35 GMT
Content-Type: application/json
Content-Length: 366
Connection: keep-alive
access-control-allow-origin: *
access-control-allow-credentials: true
x-envoy-upstream-service-time: 2
```

要验证未经授权的客户端无法访问后端，更新 IngressBackend 配置中指定的 `sources`。让我们将主题更新为 Nginx 客户端证书中编码的 SAN 以外的其他内容。

```bash
kubectl apply -f - <<EOF
apiVersion: policy.openservicemesh.io/v1alpha1
kind: IngressBackend
metadata:
  name: httpbin
  namespace: httpbin
spec:
  backends:
  - name: httpbin
    port:
      number: 14001 # targetPort of httpbin service
      protocol: https
    tls:
      skipClientCertValidation: false
  sources:
  - kind: Service
    name: "$nginx_ingress_service"
    namespace: "$nginx_ingress_namespace"
  - kind: AuthenticatedPrincipal
    name: untrusted-client.cluster.local # untrusted
EOF
```

确认请求被拒绝并收到 `HTTP 403 Forbidden` 响应：
```console
$ curl -sI http://"$nginx_ingress_host":"$nginx_ingress_port"/get
HTTP/1.1 403 Forbidden
Date: Wed, 18 Aug 2021 18:36:09 GMT
Content-Type: text/plain
Content-Length: 19
Connection: keep-alive
```

接下来，我们通过更新 IngressBackend 配置以设置 `skipClientCertValidation: true` 来演示对在服务后端禁用客户端证书验证的支持，同时仍使用不受信任的客户端：

```bash
kubectl apply -f - <<EOF
apiVersion: policy.openservicemesh.io/v1alpha1
kind: IngressBackend
metadata:
  name: httpbin
  namespace: httpbin
spec:
  backends:
  - name: httpbin
    port:
      number: 14001 # targetPort of httpbin service
      protocol: https
    tls:
      skipClientCertValidation: true
  sources:
  - kind: Service
    name: "$nginx_ingress_service"
    namespace: "$nginx_ingress_namespace"
  - kind: AuthenticatedPrincipal
    name: untrusted-client.cluster.local # untrusted
EOF
```

确认在允许未信任的经过身份验证的主题连接到后端后请求成功。
```
$ curl -sI http://"$nginx_ingress_host":"$nginx_ingress_port"/get
HTTP/1.1 200 OK
Date: Wed, 18 Aug 2021 18:36:49 GMT
Content-Type: application/json
Content-Length: 364
Connection: keep-alive
access-control-allow-origin: *
access-control-allow-credentials: true
x-envoy-upstream-service-time: 2
```

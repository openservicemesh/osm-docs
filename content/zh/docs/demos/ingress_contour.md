---
title: "Contour 入口"
description: "Contour 入口控制器实现的 HTTP/HTTPS 入口"
type: docs
weight: 10
---

SM 可以选择使用 [Contour](https://projectcontour.io) 入口控制器和基于 Envoy 的边缘代理来路由外部的流量到服务网格后端。这个指南演示了如何为 OSM 服务网格管理的服务配置 HTTP 和 HTTPS ingress。

## 先决条件

- Kubernetes 集群版本 {{< param min_k8s_version >}} 或者更高。
- 使用 `kubectl` 与 API server 交互。
- 未安装 OSM。如果已安装必须先删除。
- 已安装 `osm` 或者 `Helm 3` 命令行工具，用于安装 OSM 和 Contour。
- OSM 版本 >= v0.10.0。


## 演示

首先，在 `osm-system` 命名空间下安装 OSM 和 Contour，并将网格名字命名为 `osm`。
```bash
export osm_namespace=osm-system # Replace osm-system with the namespace where OSM will be installed
export osm_mesh_name=osm # Replace osm with the desired OSM mesh name
```

使用 `osm` 命令行工具：
```bash
osm install --set contour.enabled=true \
    --mesh-name "$osm_mesh_name" \
    --osm-namespace "$osm_namespace" \
    --set contour.configInline.tls.envoy-client-certificate.name=osm-contour-envoy-client-cert \
    --set contour.configInline.tls.envoy-client-certificate.namespace="$osm_namespace"
```

使用 `Helm` 安装：
```bash
helm install "$osm_mesh_name" osm --repo https://openservicemesh.github.io/osm \
    --set contour.enabled=true \
    --set contour.configInline.tls.envoy-client-certificate.name=osm-contour-envoy-client-cert \
    --set contour.configInline.tls.envoy-client-certificate.namespace="$osm_namespace"
```

为了通过对访问后端流量的限制来对客户端授权，我们将配置 IngressBackend，这样只有来自 `osm-contour-envoy` 端点的入口流量才可以路由到后端服务。为了发现 `osm-contour-envoy` 端点，我们需要 OSM 控制器及监视相应的命名空间。但是为了保证 Contour 功能正常，不能为其注入 Envoy sidecar。

```bash
kubectl label namespace "$osm_namespace" openservicemesh.io/monitored-by="$osm_mesh_name"
```

保存入口网关的外部 IP 地址和端口，后面会用其测试访问后端应用。

```bash
export ingress_host="$(kubectl -n "$osm_namespace" get service osm-contour-envoy -o jsonpath='{.status.loadBalancer.ingress[0].ip}')"
export ingress_port="$(kubectl -n "$osm_namespace" get service osm-contour-envoy -o jsonpath='{.spec.ports[?(@.name=="http")].port}')"
```

下一步是部署示例 `httpbin` 服务。

```bash
# Create a namespace
kubectl create ns httpbin

# Add the namespace to the mesh
osm namespace add httpbin

# Deploy the application
kubectl apply -f https://raw.githubusercontent.com/openservicemesh/osm-docs/{{< param osm_branch >}}/manifests/samples/httpbin/httpbin.yaml -n httpbin
```

确保 `httpbin` 服务和 pod 启动并正常运行：

```console
$ kubectl get pods -n httpbin
NAME                       READY   STATUS    RESTARTS   AGE
httpbin-74677b7df7-zzlm2   2/2     Running   0          11h

$ kubectl get svc -n httpbin
NAME      TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)     AGE
httpbin   ClusterIP   10.0.22.196   <none>        14001/TCP   11h
```

### HTTP Ingress

接下来，创建必要的 HTTPProxy 和 IngressBackend 配置来允许外部客户端访问 `httpbin` 命名空间下 `httpbin` 服务的 `14001` 端口。因为没有使用 TLS，Contour入口网关到 `httpbin` 后端 pod 的链接没有加密。

```bash
kubectl apply -f - <<EOF
apiVersion: projectcontour.io/v1
kind: HTTPProxy
metadata:
  name: httpbin
  namespace: httpbin
spec:
  virtualhost:
    fqdn: httpbin.org
  routes:
  - services:
    - name: httpbin
      port: 14001
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
    namespace: "$osm_namespace"
    name: osm-contour-envoy
EOF
```

现在，我们期望外部客户端可以访问 `httpbin` 服务，HTTP 请求的 `HOST` 请求头为 `httpbin.org`：

```console
$ curl -sI http://"$ingress_host":"$ingress_port"/get -H "Host: httpbin.org"
HTTP/1.1 200 OK
server: envoy
date: Fri, 06 Aug 2021 17:39:43 GMT
content-type: application/json
content-length: 314
access-control-allow-origin: *
access-control-allow-credentials: true
x-envoy-upstream-service-time: 3
vary: Accept-Encoding
```

### HTTPS 入口 (mTLS 和 TLS)

如果要将使用 HTTPS 的连接代理到 TLS 后端，后端服务必须要使用下面的方式将端口写入注解中：

```bash
kubectl annotate service httpbin -n httpbin projectcontour.io/upstream-protocol.tls='14001' --overwrite
```

然后，我们需要创建 HTTPProxy 配置来使用 TLS 代理后端服务，同时提供 CA 证书对后端服务的服务器证书进行校验。为此，当在 `httpbin` 命名空间的 HTTPProxy 配置引用的时候，我们首先需要委托 Contour 访问 OSM 命名空间下 OSM CA 证书 secret 的权限。参考[上游 TLS](https://projectcontour.io/docs/v1.18.0/config/upstream-tls/) 了解上游证书校验的更多信息以及何时需要证书委托。此外，我们还要创建一个 IngressBackend 资源来说明路由到 `httpbin` 服务的流量只能来自可信客户端，即我们部署的入口边缘代理中的 osm-contur-envoy。OSM 在安装时自动为 `osm-contour-envoy` 入口网关配置带有主题备用名称（Subject Alternative Name，SAN）为 `osm-contour-envoy.$osm_namespace.cluster.local` 的客户端证书，因此 IngressBackend 配置为 `osm-contour-envoy` 边缘与 `httpbin` 后端之间的 mTLS 认证需要引用同样的 SAN。

> 注意：`<osm-namespace>` 指安装 osm 控制平面的命名空间。

应用配置：

```bash
kubectl apply -f - <<EOF
apiVersion: projectcontour.io/v1
kind: TLSCertificateDelegation
metadata:
  name: ca-secret
  namespace: "$osm_namespace"
spec:
  delegations:
    - secretName: osm-ca-bundle
      targetNamespaces:
      - httpbin
---
apiVersion: projectcontour.io/v1
kind: HTTPProxy
metadata:
  name: httpbin
  namespace: httpbin
spec:
  virtualhost:
    fqdn: httpbin.org
  routes:
  - services:
    - name: httpbin
      port: 14001
      validation:
        caSecret: "$osm_namespace/osm-ca-bundle"
        # subjectName for a service is of the form <service-account>.<namespace>.cluster.local
        # where the service account and namespace is that of the pod backing the service
        subjectName: httpbin.httpbin.cluster.local
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
      protocol: https
    tls:
      skipClientCertValidation: false # mTLS (defaults to false)
  sources:
  - kind: Service
    namespace: "$osm_namespace"
    name: osm-contour-envoy
  - kind: AuthenticatedPrincipal
    name: "osm-contour-envoy.$osm_namespace.cluster.local"
EOF
```

这时，我们期望外部的客户端可以访问 `httpbin` 服务，在入口网关和后端服务之间通过 mTLS 进行 HTTPS 代理，发送 HTTP 请求的 `Host:` 请求头为 `httpbin.org` ：

```console
$ curl -sI http://"$ingress_host":"$ingress_port"/get -H "Host: httpbin.org"
HTTP/1.1 200 OK
server: envoy
date: Fri, 06 Aug 2021 17:39:43 GMT
content-type: application/json
content-length: 314
access-control-allow-origin: *
access-control-allow-credentials: true
x-envoy-upstream-service-time: 3
vary: Accept-Encoding
```

为了验证未经授权的客户端是无权访问后端，我们可以更新 IngressBackend 配置中 `sources` 指定的内容。 改成与入口网关中编码的 SAN 不同的内容：

```bash
kubectl apply -f - <<EOF
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
      protocol: https
    tls:
      skipClientCertValidation: false # mTLS (defaults to false)
  sources:
  - kind: Service
    namespace: "$osm_namespace"
    name: osm-contour-envoy
  - kind: AuthenticatedPrincipal
    name: "untrusted-client.cluster.local"
EOF
```

确认请求被拒绝并收到 `HTTP 403 Forbidden` 响应：

```console
$ curl -sI http://"$ingress_host":"$ingress_port"/get -H "Host: httpbin.org"
HTTP/1.1 403 Forbidden
content-length: 19
content-type: text/plain
date: Fri, 06 Aug 2021 18:40:45 GMT
server: envoy
x-envoy-upstream-service-time: 8
```

下一步，我们演示在必要时支持禁用客户端证书认证，通过更新 IngressBackend 配置设置 `skipClientCertValidation: true`，同时还是用不可信的客户端：

```bash
kubectl apply -f - <<EOF
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
      protocol: https
    tls:
      skipClientCertValidation: true
  sources:
  - kind: Service
    namespace: "$osm_namespace"
    name: osm-contour-envoy
  - kind: AuthenticatedPrincipal
    name: "untrusted-client.cluster.local"
EOF
```

确认允许未经认证的客户端访问后端服务：

```
$ curl -sI http://"$ingress_host":"$ingress_port"/get -H "Host: httpbin.org"
HTTP/1.1 200 OK
server: envoy
date: Fri, 06 Aug 2021 18:51:47 GMT
content-type: application/json
content-length: 314
access-control-allow-origin: *
access-control-allow-credentials: true
x-envoy-upstream-service-time: 4
vary: Accept-Encoding
```
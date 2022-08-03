---
title: "应用协议选择"
description: "应用协议选择"
type: docs
---

# 应用协议选择

OSM 能够以不同的方式路由不同的应用协议，比如 `HTTP`、`TCP` 和 `gRPC`。下面的指南描述了如果配置 service 端口来指定应用协议用于流量过滤和路由。

## 配置应用协议

Kubernetes service 暴露一个或多个端口。提供服务的应用暴露的端口可以提供对应应用协议的服务，比如 HTTP、TCP、gRPC 等等。由于 OSM 使用不同的方式来路由不同的应用协议，有必要对 Kubernetes service 对象进行配置来向 OSM 传达如何将定向到 service 端口的流量进行路由。

为了检测 service 端口使用的应用协议，OSM 期望 service 端口上配置了 `appProtocol` 字段。

OSM 为提供 service 端口提供了如下协议的支持：
1. `http`：用于基于 HTTP 的流量过滤和路由
1. `tcp`：用于基于 TCP 的流量过滤和路由
1. `tcp-server-first`：用于在客户端与服务端首次通信时的基于 TCP 的流量过滤和路由。
1. `gRPC`: 用于基于 HTTP2 的 gRPC 流量过滤和路由

所述的应用协议配置适用于 SMI 和宽松流量策略模式。

### 示例

考虑下面的 SMI 流量访问和流量规格策略：
- 名为 `tcp-route` 的 `TCPRoute` 资源指定端口的 TCP 流量被允许。
- 名为 `http-route` 的 `HTTPRouteGroup` 资源为 HTTP 流量指定 HTTP 路由应该被允许。
- 名为 `test` 的 `TrafficTarget` 资源，允许 `sa-2` service account 的 pod 以指定的 TCP 和 HTTP 规则访问 `sa-1` service account 的 pod。

```yaml
kind: TCPRoute
metadata:
  name: tcp-route
spec:
  matches:
    ports:
    - 8080
---
kind: HTTPRouteGroup
metadata:
  name: http-route
spec:
  matches:
  - name: version
    pathRegex: "/version"
    methods:
    - GET
---
kind: TrafficTarget
metadata:
  name: test
  namespace: default
spec:
  destination:
    kind: ServiceAccount
    name: sa-1 # There are 2 services under this service account:  service-1 and service-2
    namespace: default
  rules:
  - kind: TCPRoute
    name: tcp-route
  - kind: HTTPRouteGroup
    name: http-route
  sources:
  - kind: ServiceAccount
    name: sa-2
    namespace: default
```

Kubenetes service 资源需要使用 `appProtocol` 字段为端口显式地指定应用协议。

由 service account `sa-1` 中的 pod 支持的服务 `service-1` 服务于 `http` 应用流量应按照如下定义：

```yaml
kind: Service
metadata:
  name: service-1
  namespace: default
spec:
  ports:
  - port: 8080
    name: some-port
    appProtocol: http
```

由服务帐户 `sa-1` 中的 pod 支持的服务 `service-2` 为原始 `tcp` 应用流量提供服务应按照如下定义：

```yaml
kind: Service
metadata:
  name: service-2
  namespace: default
spec:
  ports:
  - port: 8080
    name: some-port
    appProtocol: tcp
```

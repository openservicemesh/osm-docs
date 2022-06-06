---
title: "熔断"
description: "使用熔断来限制连接和请求"
type: docs
weight: 11
---

# 熔断

熔断是分布式系统的关键组成部分，也是一种重要的弹性模式。熔断允许应用程序快速失败并尽快向下游施加背压，从而提供限制故障对整个系统的影响的方法。本指南描述了如何在 OSM 中配置熔断。

## 配置熔断

OSM 利用其 [UpstreamTrafficSetting API][1] 为定向到上游服务的流量配置断路属性。我们使用术语 `upstream service` 来指代从客户端接收连接和请求并返回响应的服务。该规范允许在连接和请求级别为上游服务配置断路属性。OSM 利用 [Envoy 的熔断功能](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/circuit_break) 为参与网格的应用程序实现熔断。

每个 `UpstreamTrafficSetting` 配置都针对由 `spec.host` 字段定义的上游主机。对于命名空间 `my-namespace` 中的 Kubernetes 服务 `my-svc`，必须在命名空间 `my-namespace` 中创建 `UpstreamTrafficSetting` 资源，并且 `spec.host` 必须是 `my-svc.my-namespace.svc.cluster.local` 形式的FQDN。当在 [出口策略](/docs/api_reference/policy/v1alpha1/#policy.openservicemesh.io/v1alpha1.EgressSpec) 中指定为匹配项时，`spec.host` 必须与出口策略中指定的主机和 `UpstreamTrafficSetting` 配置必须与 `Egress` 资源属于同一命名空间。

熔断适用于 TCP 和 HTTP 级别，并且可以使用 `UpstreamTrafficSetting` 资源中的 `connectionSettings` 属性进行配置。TCP 流量设置适用于 TCP 和 HTTP 流量，而 HTTP 设置仅适用于 HTTP 流量。

支持以下熔断配置：

- `Maximum connections`：允许客户端建立到属于通过 `UpstreamTrafficSetting` 配置中的 `spec.host` 字段指定的上游主机的所有后端的最大连接数。此设置可以使用 `tcp.maxConnections` 字段进行配置，并且适用于 TCP 和 HTTP 流量。如果未指定，默认为 `4294967295` (2^32 - 1)。

- `Maximum pending requests`：允许排队的上游主机的未处理 HTTP 请求的最大数量。每当没有足够的上游连接可用要立即分派请求时，请求就会添加到待处理请求列表中。对于 HTTP/2 连接，如果 `http.maxRequestsPerConnection` 没有配置，所有请求都将在同一个连接上进行多路复用，因此只有在没有建立连接时才会触发此断路器。此设置可以使用 `http.maxPendingRequests` 字段进行配置，并且仅适用于 HTTP 流量。如果未指定，默认为 `4294967295` (2^32 - 1)。

- `Maximum requests`：允许客户端向上游主机发出的最大并行请求数。此设置可以使用 `http.maxRequests` 字段进行配置，并且仅适用于 HTTP 流量。如果未指定，默认为 `4294967295` (2^32 - 1)。

- `Maximum requests per connection`：每个连接允许的最大请求数。此设置可以使用 `http.maxRequestsPerConnection` 字段进行配置，并且仅适用于 HTTP 流量。如果未指定，则没有限制。

- `Maximum active retries`：允许客户端对上游主机进行的最大活动重试次数。此设置可以使用 `http.maxRetries` 字段进行配置，并且仅适用于 HTTP 流量。如果未指定，默认为 `4294967295` (2^32 - 1)。


要了解有关配置熔断的更多信息，请参阅以下演示指南：

- [网格内目的地的熔断](/docs/demos/circuit_breaking_mesh_internal)
- [网格外部目的地的熔断](/docs/demos/circuit_break_mesh_external)

[1]: /docs/api_reference/policy/v1alpha1/#policy.openservicemesh.io/v1alpha1.UpstreamTrafficSettingSpec
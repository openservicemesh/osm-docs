---
title: "限速"
description: 使用断路来控制流量的吞吐量"
type: docs
weight: 12
---

# 限速

限速是一种控制发往目标主机的流量的有效机制。它限制了下游客户端在特定时间范围内发送网络流量的速度。

最常见的是，当大量客户端向目标主机发送流量时，如果目标主机拥堵，下游客户端将压倒上游目标主机。在这种情况下，很难在每个下游主机上配置足够严格的断路限制，以使系统在典型的请求模式下正常运行，但在系统开始出现故障时仍能防止级联故障。在这种情况下，对目标主机的流量进行限速是有效的。

OSM 支持每个目标主机的服务器端限速，也称为“本地实例限速”。

## 配置本地实例限速

OSM 利用其 [UpstreamTrafficSetting API][1] 为发往上游服务的流量配置限速。我们使用术语“上游服务”来指代从客户端接收连接和请求并返回响应的服务。该规范允许在连接和请求级别为上游服务配置本地限速。OSM 利用 [Envoy 的本地速率限制功能](https://www.envoyproxy.io/docs/envoy/latest/configuration/listeners/network_filters/local_rate_limit_filter#config-network-filters-local-rate-limit) 来实现针对每个上游主机实例的本地限速。

每个 `UpstreamTrafficSetting` 配置都针对由 `spec.host` 字段定义的上游主机。对于命名空间 `my-namespace` 中的 Kubernetes 服务 `my-svc`，必须在命名空间 `my-namespace` 中创建 `UpstreamTrafficSetting` 资源，并且 `spec.host` 必须是 `my-svc.my-namespace.svc.cluster.local` 格式的 FQDN。

本地限速适用于 TCP (L4) 连接和 HTTP 请求级别，并且可以使用 `UpstreamTrafficSetting` 资源中的 `rateLimit.local` 属性进行配置。TCP 设置适用于 TCP 和 HTTP 流量，而 HTTP 设置仅适用于 HTTP 流量。TCP 和 HTTP 级别的速率限制都是使用令牌桶速率限制器执行的。

### TCP 连接限速

可以在单位时间内限制 TCP 连接的速率。可以指定一个可选的突发限制，以允许突发的高于基线速率的连接，以适应短时间内的连接突发。TCP 速率限制作为令牌桶速率限制器应用于上游服务的入站监听器的网络过滤器链。过滤器处理的每个传入连接都需要一个令牌。如果令牌可用，则允许连接。如果没有可用的令牌，连接将立即关闭。

以下嵌套在 `spec.rateLimit.local.tcp` 下的属性定义了 TCP 连接的速率限制属性：

- `connections`：在通过 `UpstreamTrafficSetting` 配置中的 `spec.host` 字段指定的上游主机的所有后端发生速率限制之前，每单位时间允许的连接数。此设置适用于 TCP 和 HTTP 流量。

- `unit`：超过限制的连接将受到速率限制的时间段。有效值为“秒”、“分钟”和“小时”。

- `burst`：短时间内允许的高于基线速率的连接数。

参考文档 [TCP 本地限速 API](/docs/api_reference/policy/v1alpha1/#policy.openservicemesh.io/v1alpha1.TCPLocalRateLimitSpec) 获取 API 使用的更多信息。

### HTTP 请求限速

可以在每单位时间的 HTTP 请求。可以指定可选的突发限制，以允许突发的高于基线速率的请求来适应短时间内的请求突发。HTTP 速率限制作为令牌桶速率限制器应用在虚拟主机和/或上游后端的 HTTP 路由级别，具体取决于速率限制配置。过滤器处理的每个传入请求都需要一个令牌。如果令牌可用，则请求将被允许。如果没有可用的令牌，请求将收到配置的速率限制状态。

HTTP 请求限速可以在 [虚拟机](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/route/v3/route_components.proto#config-route-v3-virtualhost)级别，通过指定嵌套在 `spec.rateLimit.local.http` 字段下的速率限制属性。或者，可以通过将速率限制属性指定为 `spec.httpRoutes` 字段的一部分，为上游后端允许的每个 HTTP 路由配置速率限制。需要注意的是，在为每个 HTTP 路由配置限速时，该路由匹配一个已经被服务网格策略允许的 HTTP 路径，否则将忽略限速策略。

可以为 HTTP 流量配置以下速率限制属性：

- `requests`：在通过 `UpstreamTrafficSetting` 配置中的 `spec.host` 字段指定的上游主机的所有后端发生速率限制之前，每单位时间允许的请求数。

- `unit`：超过限制的请求将受到速率限制的时间段。有效值为“秒”、“分钟”和“小时”。

- `burst`：短时间内允许的高于基准速率的请求数。

- `responseStatusCode`：用于响应速率受限请求的 HTTP 状态代码。代码必须在 400-599（含）错误范围内。如果未指定，则使用默认值 429（请求过多）。该代码必须是 [Envoy 支持的状态代码](https://www.envoyproxy.io/docs/envoy/latest/api-v3/type/v3/http_status.proto#enum-type-v3-statuscode)。

- `responseHeadersToAdd`：键值对格式的 HTTP 请求头列表，会添加到已受速率限制的请求的每个响应中。

## 示例

要了解有关配置速率限制的更多信息，请参阅以下演示指南：
- [TCP 连接的本地限速](/docs/demos/local_rate_limit_connections)
- [HTTP 请求的本地限速](/docs/demos/local_rate_limit_http)

[1]: /docs/api_reference/policy/v1alpha1/#policy.openservicemesh.io/v1alpha1.UpstreamTrafficSettingSpec
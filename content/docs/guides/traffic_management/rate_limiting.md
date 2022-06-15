---
title: "Rate Limiting"
description: "Using rircuit breaking to control the throughput of traffic"
type: docs
weight: 12
---

# Rate Limiting

Rate limiting is an effective mechanism to control the throughput of traffic destined to a target host. It puts a cap on how often downstream clients can send network traffic within a certain timeframe.

Most commonly, when a large number of clients are sending traffic to a target host, if the target host becomes backed up, the downstream clients will overwhelm the upstream target host. In this scenario it is extremely difficult to configure a tight enough circuit breaking limit on each downstream host such that the system will operate normally during typical request patterns but still prevent cascading failure when the system starts to fail. In such scenarios, rate limiting traffic to the target host is effective.

OSM support server-side rate limiting per target host, also referred to as `local per-instance rate limiting`.

## Configuring local per-instance rate limiting

OSM leverages its [UpstreamTrafficSetting API][1] to configure rate breaking attributes for traffic directed to an upstream service. We use the term `upstream service` to refer to a service that receives connections and requests from clients and return responses. The specification enables configuring local rate limiting attributes for an upstream service at the connection and request level. OSM leverages [Envoy's local rate limiting functionality](https://www.envoyproxy.io/docs/envoy/latest/configuration/listeners/network_filters/local_rate_limit_filter#config-network-filters-local-rate-limit) to implement per-instance local rate limiting at each upstream host.

Each `UpstreamTrafficSetting` configuration targets an upstream host defined by the `spec.host` field. For a Kubernetes service `my-svc` in the namespace `my-namespace`, the `UpstreamTrafficSetting` resource must be created in the namespace `my-namespace`, and `spec.host` must be an FQDN of the form `my-svc.my-namespace.svc.cluster.local`.

Local rate limiting is applicable at both the TCP (L4) connection and HTTP request level, and can be configured using the `rateLimit.local` attribute in the `UpstreamTrafficSetting` resource. TCP settings apply to both TCP and HTTP traffic, while HTTP settings only apply to HTTP traffic. Both TCP and HTTP level rate limiting is enforced using a token bucket rate limiter.

### Rate limiting TCP connections

TCP connections can be rate limited per unit of time. An optional burst limit can be specified to allow a burst of connections above the baseline rate to accommodate for connection bursts in a short interval of time. TCP rate limiting is applied as a token bucket rate limiter at the network filter chain of the upstream service’s inbound listener. Each incoming connection processed by the filter consumes a single token. If the token is available, the connection will be allowed. If no tokens are available, the connection will be immediately closed.

The following attributes nested under `spec.rateLimit.local.tcp` define the rate limiting attributes for TCP connections:

- `Connections`: The number of connections allowed per unit of time before rate limiting occurs on all backends belonging to the upstream host specified via the `spec.host` field in the `UpstreamTrafficSetting` configuration. This setting can be configured using the `connections` field and is applicable to both TCP and HTTP traffic.

- `Unit`: The period of time within which connections over the limit will be rate limited. Valid values are `second`, `minute` and `hour`.

- `Burst`: The number of connections above the baseline rate that are allowed in a short period of time.

Refer to the [TCP local rate limiting API](/docs/api_reference/policy/v1alpha1/#policy.openservicemesh.io/v1alpha1.TCPLocalRateLimitSpec) for additional information regarding API usage.

## Demos

To learn more about configuring rate limting, refer to the following demo guides:
- [Local rate limiting of TCP connections](/docs/demos/local_rate_limit_connections)

[1]: /docs/api_reference/policy/v1alpha1/#policy.openservicemesh.io/v1alpha1.UpstreamTrafficSettingSpec
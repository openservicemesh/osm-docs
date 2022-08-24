---
title: "Rate Limiting"
description: "Using rate limiting to control the throughput of traffic"
type: docs
weight: 12
---

# Rate Limiting

Rate limiting is an effective mechanism to control the throughput of traffic destined to a target host. It puts a cap on how often downstream clients can send network traffic within a certain timeframe.

Most commonly, when a large number of clients are sending traffic to a target host, if the target host becomes backed up, the downstream clients will overwhelm the upstream target host. In this scenario it is extremely difficult to configure a tight enough circuit breaking limit on each downstream host such that the system will operate normally during typical request patterns but still prevent cascading failure when the system starts to fail. In such scenarios, rate limiting traffic to the target host is effective.

OSM supports two forms of rate limiting:
1. `Local per-instance rate limiting` at each upstream host
2. `Global rate limiting` using a global gRPC rate limiting service

OSM leverages its [UpstreamTrafficSetting API](/docs/api_reference/policy/v1alpha1/#policy.openservicemesh.io/v1alpha1.UpstreamTrafficSettingSpec) to configure rate limiting attributes for traffic directed to an upstream service. We use the term `upstream service` to refer to a service that receives connections/requests from clients and returns responses.

Each `UpstreamTrafficSetting` configuration targets an upstream host defined by the `spec.host` field. For a Kubernetes service `my-svc` in the namespace `my-namespace`, the `UpstreamTrafficSetting` resource must be created in the namespace `my-namespace`, and `spec.host` must be an FQDN of the form `my-svc.my-namespace.svc.cluster.local`.

## Configuring local per-instance rate limiting

 The specification enables configuring local rate limiting attributes for an upstream service at the connection and request level. OSM leverages [Envoy's local rate limiting capability](https://www.envoyproxy.io/docs/envoy/latest/configuration/listeners/network_filters/local_rate_limit_filter#config-network-filters-local-rate-limit) to implement per-instance local rate limiting at each upstream host.

Local rate limiting is applicable at both the TCP (L4) connection and HTTP request level, and can be configured using the `rateLimit.local` attribute in the `UpstreamTrafficSetting` resource. TCP settings apply to both TCP and HTTP traffic, while HTTP settings only apply to HTTP traffic. Both TCP and HTTP level rate limiting is enforced using a token bucket rate limiter.

### Local rate limiting of TCP connections

TCP connections can be rate limited per unit of time. An optional burst limit can be specified to allow a burst of connections above the baseline rate to accommodate for connection bursts in a short interval of time. TCP rate limiting is applied as a token bucket rate limiter at the network filter chain of the upstream service’s inbound listener. Each incoming connection processed by the filter consumes a single token. If the token is available, the connection will be allowed. If no tokens are available, the connection will be immediately closed.

The following attributes nested under `spec.rateLimit.local.tcp` define the rate limiting attributes for TCP connections:

- `connections`: The number of connections allowed per unit of time before rate limiting occurs on all backends belonging to the upstream host specified via the `spec.host` field in the `UpstreamTrafficSetting` configuration. This setting is applicable to both TCP and HTTP traffic.

- `unit`: The period of time within which connections over the limit will be rate limited. Valid values are `second`, `minute` and `hour`.

- `burst`: The number of connections above the baseline rate that are allowed in a short period of time.

Refer to the [TCP local rate limiting API](/docs/api_reference/policy/v1alpha1/#policy.openservicemesh.io/v1alpha1.TCPLocalRateLimitSpec) for additional information regarding API usage.

### Local rate limiting of HTTP requests

HTTP requests can be rate limited per unit of time. An optional burst limit can be specified to allow a burst of requests above the baseline rate to accommodate for request bursts in a short interval of time. HTTP rate limiting is applied as a token bucket rate limiter at the virtual host and/or HTTP route level at the upstream backend, depending on the rate limiting configuration. Each incoming request processed by the filter consumes a single token. If the token is available, the request will be allowed. If no tokens are available, the request will receive the configured rate limit status.

HTTP request rate limiting can be configured at the [virtual host](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/route/v3/route_components.proto#config-route-v3-virtualhost) level by specifying the rate limiting attributes nested under the `spec.rateLimit.local.http` field. Alternatively, rate limiting can be configured per HTTP route allowed on the upstream backend by specifying the rate limiting attributes as a part of the `spec.httpRoutes` field. It is important to note that when configuring rate limiting per HTTP route, the route matches an HTTP path that has already been permitted by a service mesh policy, otherwise the rate limiting policy will be ignored.

The following rate limiting attributes can be configured for HTTP traffic:

- `requests`: The number of requests allowed per unit of time before rate limiting occurs on all backends belonging to the upstream host specified via the `spec.host` field in the `UpstreamTrafficSetting` configuration.

- `unit`: The period of time within which requests over the limit will be rate limited. Valid values are `second`, `minute` and `hour`.

- `burst`: The number of requests above the baseline rate that are allowed in a short period of time.

- `responseStatusCode`: The HTTP status code to use for responses to rate limited requests. Code must be in the 400-599 (inclusive) error range. If not specified, a default of 429 (Too Many Requests) is used. The code must be a [status code supported by Envoy](https://www.envoyproxy.io/docs/envoy/latest/api-v3/type/v3/http_status.proto#enum-type-v3-statuscode).

- `responseHeadersToAdd`: The list of HTTP headers as key-value pairs that should be added to each response for requests that have been rate limited.


## Configuring global rate limiting

 The specification enables configuring global rate limiting attributes for an upstream service at the connection and request level. OSM leverages [Envoy's global rate limiting capability](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/other_features/global_rate_limiting#arch-overview-global-rate-limit) to implement rate limiting by making calls to a global gRPC Rate Limit Service (RLS). Any service that implements the [defined RPC/IDL protocol](https://www.envoyproxy.io/docs/envoy/latest/api-v3/service/ratelimit/v3/rls.proto#envoy-v3-api-file-envoy-service-ratelimit-v3-rls-proto) can be used as the global rate limit service.

Global rate limiting is applicable at both the TCP (L4) connection and HTTP (L7) request level, and can be configured using the `rateLimit.global` attribute in the `UpstreamTrafficSetting` resource. TCP settings apply to both TCP and HTTP traffic, while HTTP settings only apply to HTTP traffic. Both TCP and HTTP level rate limiting is enforced by calling the rate limit service for every new connection/request to make a rate limit decision.

Unlike local rate limit configuration, global rate limit configuration does not directly define a rate limit policy for the target upstream host. Instead, global rate limiting is enforced by specifying a descriptor set in the `UpstreamTrafficSetting` configuration. A descriptor set is a list of hierarchical entries that are used by the Rate Limit Service (RLS) to determine the final rate limit key and overall allowed limit. The specified set of descriptors will be generated and sent to the RLS for each connection/request to make a rate limit decision.

### Global rate limiting of TCP connections

TCP connections from downstream clients inbound on an upstream host can be rate limited using a global rate limiter service. The configuration specifies a specific domain and descriptor set to rate limit on. This has the ultimate effect of rate limiting connections per second that transit the upstream host's inbound listener.

Envoy supports configuring static key-value pairs as descriptor entries to use in the rate limit service request for TCP connections.

For example, to rate limit TCP connections on the `foo.demo.svc.cluster.local` service in the `demo` namespace using an external RLS `ratelimiter.rls.svc.cluster.local` serving requests on port `8081`, for the descriptor entry `my_key: my_value` and rate limit domain `test` will look as follows:

```yaml
apiVersion: policy.openservicemesh.io/v1alpha1
kind: UpstreamTrafficSetting
metadata:
  name: foo
  namespace: demo
spec:
  host: foo.demo.svc.cluster.local
  rateLimit:
    global:
      tcp:
        rateLimitService:
          host: ratelimiter.rls.svc.cluster.local
          port: 8081
        domain: test
        failOpen: false
        timeout: 10s
        descriptors:
          - entries:
            - key: my_key
              value: my_value
```

Refer to the [Global TCP rate limit API](/docs/api_reference/policy/v1alpha1/#policy.openservicemesh.io/v1alpha1.TCPGlobalRateLimitSpec) to learn more about the configuration attributes and an [end-to-end demo](/docs/demos/global_rate_limit_connections) to understand the global TCP rate limiting capability.

### Global rate limiting of HTTP requests

HTTP connections from downstream clients inbound on an upstream host can be rate limited using a global rate limit service. The configuration specifies a specific domain and descriptor set to rate limit on. This has the ultimate effect of rate limiting requests per second that transit the upstream host's inbound listener.

The rate limit configuration can be applied at both the virtual host and route level. The policy defines a set of request descriptors that will be generated and sent to the external RLS to make a rate limit decision on each request. A list of descriptors, each comprising of one or more descriptor entries, is specified and generated based on different criteria. If an entry specified within a descriptor cannot be generated for a request, the entire descriptor is not generated. When multiple descriptors are specified, all descriptors that can be generated will be generated and sent to the rate limit service. Refer to the [Envoy documentation on composing descriptors](https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_filters/rate_limit_filter#composing-actions) for more information.

OSM supports different kinds of descriptor entries for HTTP requests, namely `genericKey`, `remoteAddress`, `requestHeader`, and `headerValueMatch`. The following sections describe each of them.

#### genericKey descriptor

The [genericKey descriptor entry](/docs/api_reference/policy/v1alpha1/#policy.openservicemesh.io/v1alpha1.GenericKeyDescriptorEntry) defines a static key-value pair. By default, the `genericKey` descriptor entry uses `generic_key` as it's descriptor key if unspecified.

For example, the following configuration generates the descriptor `("my_key", "my_value")`:
```yaml
rateLimit:
  global:
    http:
      descriptors:
        - entries:
          - genericKey:
              key: my_key
              value: my_value
```

Refer to the [genericKey descriptor demo](/docs/demos/global_rate_limit_http/#generickey-descriptor) to learn more about using the `genericKey` descriptor entry.

#### remoteAddress descriptor

The [remoteAddress descriptor entry](/docs/api_reference/policy/v1alpha1/#policy.openservicemesh.io/v1alpha1.RemoteAddressDescriptorEntry) has a key of `remote_address` and a value of the client IP address that is populated using the trusted address from the `x-forwarded-for` HTTP header.

For example, the following configuration generates the descriptor `("remote_address", "<trusted address from x-forwarded-for>")`:
```yaml
rateLimit:
  global:
    http:
      descriptors:
        - entries:
          - remoteAddress: {}
```

Refer to the [remoteAddress descriptor demo](/docs/demos/global_rate_limit_http/#remoteaddress-descriptor) to learn more about using the `remoteAddress` descriptor entry.

#### requestHeader descriptor

The [requestHeader descriptor entry](/docs/api_reference/policy/v1alpha1/#policy.openservicemesh.io/v1alpha1.RequestHeaderDescriptorEntry) defines a static key-value pair for the descriptor entry that is generated only when the request header matches the given header name. The value of the descriptor entry is derived from the value of the header present in the request. If the header is not present, the descriptor is not generated.

For example, the following configuration generates the descriptor `("my_header", "<value from some-header>")`:
```yaml
rateLimit:
  global:
    http:
      descriptors:
        - entries:
          - requestHeader:
              name: some-header
              key: my_header
```

Refer to the [requestHeader descriptor demo](/docs/demos/global_rate_limit_http/#requestheader-descriptor) to learn more about using the `requestHeader` descriptor entry.

#### headerValueMatch descriptor

The [headerValueMatch descriptor entry](/docs/api_reference/policy/v1alpha1/#policy.openservicemesh.io/v1alpha1.HeaderValueMatchDescriptorEntry) defines a descriptor entry that is generated when the request header matches the given set of [HTTP header match criteria](/docs/api_reference/policy/v1alpha1/#policy.openservicemesh.io/v1alpha1.HTTPHeaderMatcher). OSM supports multiple header match operators in the form of `Exact, Prefix, Suffix, Regex, Contains, Present` match semantics.

For example, the following configuration generates the descriptor `("header_match", "foo")` for requests that have the header `my-header` present and that don't have the header `other-header` present.
```yaml
rateLimit:
  global:
    http:
      descriptors:
        - entries:
          - headerValueMatch:
              key: header_match
              value: foo
              headers:
                - name: my-header
                  present: true
                - name: other-header
                  present: false
```

## Demos

To learn more about configuring rate limting, refer to the following demo guides:
- [Local rate limiting of TCP connections](/docs/demos/local_rate_limit_connections)
- [Local rate limiting of HTTP requests](/docs/demos/local_rate_limit_http)
- [Global rate limiting of TCP connections](/docs/demos/global_rate_limit_connections)
- [Global rate limiting of HTTP requests](/docs/demos/global_rate_limit_http/)

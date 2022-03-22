---
title: "Circuit Breaking"
description: "Using Circuit breaking to limit connections and requests"
type: docs
weight: 11
---

# Circuit Breaking

Circuit breaking is a critical component of distributed systems and an important resiliency pattern. Circuit breaking allows applications to fail quickly and apply back pressure downstream as soon as possible, thereby providing the means to limit the impact of failures across the system. This guide describes how circuit breaking can be configured in OSM.

## Configuring circuit breaking

OSM leverages its [UpstreamTrafficSetting API][1] to configure circuit breaking attributes for traffic directed to an upstream service. We use the term `upstream service` to refer to a service that receives connections and requests from clients and return responses. The specification enables configuring circuit breaking attributes for an upstream service at the connection and request level. OSM leverages [Envoy's circuit breaking functionality](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/circuit_breaking) to implement circuit breaking for applications participating in the mesh.

Each `UpstreamTrafficSetting` configuration targets an upstream host defined by the `spec.host` field. For a Kubernetes service `my-svc` in the namespace `my-namespace`, the `UpstreamTrafficSetting` resource must be created in the namespace `my-namespace`, and `spec.host` must be an FQDN of the form `my-svc.my-namespace.svc.cluster.local`. When specified as a match in an [Egress policy](/docs/api_reference/policy/v1alpha1/#policy.openservicemesh.io/v1alpha1.EgressSpec), `spec.host` must correspond to the host specified in the Egress policy and the `UpstreamTrafficSetting` configuration must belong to the same namespace as the `Egress` resource.

Circuit breaking is applicable at both the TCP and HTTP level, and can be configured using the `connectionSettings` attribute in the `UpstreamTrafficSetting` resource. TCP traffic settings apply to both TCP and HTTP traffic, while HTTP settings only apply to HTTP traffic.

The following circuit breaking configurations are supported:

- `Maximum connections`: The maximum number of connections that a client is allowed to establish to all backends belonging to the upstream host specified via the `spec.host` field in the `UpstreamTrafficSetting` configuration. This setting can be configured using the `tcp.maxConnections` field and is applicable to both TCP and HTTP traffic. If not specified, the default is `4294967295` (2^32 - 1).

- `Maximum pending requests`: The maximum number of pending HTTP requests to the upstream host that are allowed to be queued. Requests are added to the list of pending requests whenever there aren't enough upstream connections available to immediately dispatch the request. For HTTP/2 connections, if `http.maxRequestsPerConnection` is not configured, all requests will be multiplexed over the same connection so this circuit breaker will only be hit when no connection is already established. This setting can be configured using the `http.maxPendingRequests` field and is only applicable to HTTP traffic. If not specified, the default is `4294967295` (2^32 - 1).

- `Maximum requests`: The maximum number of parallel request that a client is allowed to make to the upstream host. This setting can be configured using the `http.maxRequests` field and is only applicable to HTTP traffic. If not specified, the default is `4294967295` (2^32 - 1).

- `Maximum requests per connection`: The maximum number of requests allowed per connection. This setting can be configured using the `http.maxRequestsPerConnection` field and is only applicable to HTTP traffic. If not specified, there is no limit.

- `Maximum active retries`: The maximum number of active retries that a client is allowed to make to the upstream host. This setting can be configured using the `http.maxRetries` field and is only applicable to HTTP traffic. If not specified, the default is `4294967295` (2^32 - 1).


To learn more about configuring circuit breaking for traffic to in-cluster destinations, refer to the [Circuit breaking for destinations within the mesh](/docs/demos/circuit_breaking_mesh_internal) demo guide.

[1]: /docs/api_reference/policy/v1alpha1/#policy.openservicemesh.io/v1alpha1.UpstreamTrafficSettingSpec
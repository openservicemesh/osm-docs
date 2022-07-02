---
title: "Traffic Splitting"
description: "Traffic splitting using SMI Traffic Split API"
type: docs
weight: 10
---

# Traffic Splitting

The [SMI Traffic Split API](https://github.com/servicemeshinterface/smi-spec/blob/main/apis/traffic-split/v1alpha2/traffic-split.md) can be used to split outgoing traffic to multiple service backends. This can be used to orchestrate canary releases for multiple versions of the software.

## What is supported

OSM implements the [SMI traffic split v1alpha2 version](https://github.com/servicemeshinterface/smi-spec/blob/main/apis/traffic-split/v1alpha2/traffic-split.md).

It supports the following:

- Traffic splitting in both SMI and Permissive traffic policy modes
- HTTP and TCP traffic splitting
- Traffic splitting for canary or blue-green deployments

## How it works

Outbound traffic destined to a Kubernetes service can be split to multiple service backends using the SMI Traffic Split API. Consider the following example where traffic to the `bookstore.default.svc.cluster.local` FQDN corresponding to the `default/bookstore` service is split to services `default/bookstore-v1` and `default/bookstore-v2`, with a weight of 90 and 10 respectively.

```yaml
apiVersion: split.smi-spec.io/v1alpha2
kind: TrafficSplit
metadata:
  name: bookstore-split
  namespace: default
spec:
  service: bookstore.default.svc.cluster.local
  backends:
  - service: bookstore-v1
    weight: 90
  - service: bookstore-v2
    weight: 10
```

For a `TrafficSplit` resource to be correctly configured, it is important to ensure the following conditions are met:

- `metadata.namespace` is a [namespace added to the mesh](/docs/guides/app_onboarding/namespaces/)
- `metadata.namespace`, `spec.service`, and `spec.backends` all belong to the same namespace
- `spec.service` specifies an [FQDN of a Kubernetes service](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/#services)
- `spec.service` and `spec.backends` correspond to Kubernetes service objects
- The total weight of all backends must be greater than zero, and each backend must have a positive weight

When a `TrafficSplit` resource is created, OSM applies the configuration on client sidecars to split traffic directed to the root service (`spec.service`) to the backends (`spec.backends`) based the specified weights. For HTTP traffic, the `Host/Authority` header in the request must match the FQDNs of the root service specified in the `TrafficSplit` resource. In the above example, it implies that the `Host/Authority` header in the HTTP request originated by the client must match the Kubernetes service FQDNs of the `default/bookstore` service for traffic split to work.

> Note: OSM does not configure `Host/Authority` header rewrites for the original HTTP requests, so it is necessary that the backend services referenced in a `TrafficSplit` resource accept requests with the original HTTP `Host/Authority` header.

It is important to note that a `TrafficSplit` resource only configures traffic splitting to a service, and does not give applications permission to communicate with each other. Thus, a valid [TrafficTarget](https://github.com/servicemeshinterface/smi-spec/blob/main/apis/traffic-access/v1alpha3/traffic-access.md#traffictarget) resource must be configured in conjunction with a `TrafficSplit` configuration to achieve traffic flow between applications as desired.

Refer to a demo on [Canary rollouts using SMI Traffic Split](/docs/demos/canary_rollout) to learn more.
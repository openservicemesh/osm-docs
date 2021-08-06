---
title: "Egress"
description: "Enable access to the Internet and services external to the service mesh."
type: docs
weight: 4
---

# Allowing access to the Internet and out-of-mesh services (Egress)

This document describes the steps required to enable access to the Internet and services external to the service mesh, referred to as `Egress` traffic.

OSM redirects all outbound traffic from a pod within the mesh to the pod's sidecar proxy. Outbound traffic can be classified into two categories:

1. Traffic to services within the mesh cluster, referred to as in-mesh traffic
2. Traffic to services external to the mesh cluster, referred to as egress traffic

While in-mesh traffic is routed based on L7 traffic policies, egress traffic is routed differently and is not subject to in-mesh traffic policies. OSM supports access to external services as a passthrough without subjecting such traffic to filtering policies.


## Configuring Egress

There are two mechanisms to configure Egress:

1. Using a mesh-wide global setting: the setting is toggled on or off and affects all pods in the mesh, enabling which allows traffic destined to destinations outside the mesh to egress the pod.
2. Using the Egress policy API: to provide fine grained access control over external traffic


## Configuring mesh-wide Egress

### Enabling mesh-wide Egress to external destinations
Egress can be enabled mesh-wide during OSM install or post install. When egress is enabled mesh-wide, outbound traffic from pods are allowed to egress the pod as long as the traffic does not match in-mesh traffic policies that otherwise deny the traffic.

1. During OSM install (default `OpenServiceMesh.enableEgress=false`):
    ```bash
    osm install --set OpenServiceMesh.enableEgress=true
    ```

2. After OSM has been installed:

	`osm-controller` retrieves the egress configuration from the `osm-mesh-config` `MeshConfig` custom resource in its namespace (`osm-system` by default). Use `kubectl patch` to set `enableEgress` to `true` in the `osm-mesh-config` resource.
    ```bash
    kubectl patch meshconfig osm-mesh-config -n osm-system -p '{"spec":{"traffic":{"enableEgress":true}}}' --type=merge
    ```

### Disabling mesh-wide Egress to external destinations
Similar to enabling egress, mesh-wide egress can be disabled during OSM install or post install.

1. During OSM install:
    ```bash
    osm install --set OpenServiceMesh.enableEgress=false
    ```

2. After OSM has been installed:
	Use `kubectl patch` to set `enableEgress` to `false` in the `osm-mesh-config` resource.
    ```bash
    kubectl patch meshconfig osm-mesh-config -n osm-system -p '{"spec":{"traffic":{"enableEgress":false}}}'  --type=merge
    ```

With egress disabled, traffic from pods within the mesh will not be able to access external services outside the cluster.

### How it works
When egress is enabled mesh-wide, OSM controller programs every Envoy proxy sidecar in the mesh with a wildcard rule that matches outbound destinations that do not correspond to in-mesh services. The wildcard rule that matches such external traffic simply proxies the traffic as is to its original destination without subjecting them to L4 or L7 traffic policies.

OSM supports egress for traffic that uses TCP as the underlying transport. This includes raw TCP traffic, HTTP, HTTPS, gRPC etc.

Since mesh-wide egress is a global setting and operates as a passthrough to unknown destinations, fine grained access control (such as applying TCP or HTTP routing policies) over egress traffic is not possible.

### Sample demo with mesh-wide Egress

#### HTTP(S) traffic with mesh-wide Egress

The following demo shows a `curl` client making HTTPS requests to the external `httpbin.org` website using Egress.

1. Install OSM with global egress enabled.
    ```bash
    osm install --set OpenServiceMesh.enableEgress=true
    ```

1. Deploy the `curl` client into the `curl` namespace after enrolling its namespace to the mesh.
    ```bash
    # Create the curl namespace
    kubectl create namespace curl

    # Add the namespace to the mesh
    osm namespace add curl

    # Deploy curl client in the curl namespace
    kubectl apply -f https://raw.githubusercontent.com/openservicemesh/osm/release-v0.9/docs/example/manifests/samples/curl/curl.yaml -n curl
    ```

    Confirm the `curl` client pod is up and running.

    ```console
    $ kubectl get pods -n curl
    NAME                    READY   STATUS    RESTARTS   AGE
    curl-54ccc6954c-9rlvp   2/2     Running   0          20s
    ```

1. Confirm the `curl` client is able to make successful HTTPS requests to the `httpbin.org` website on port `443`.
    ```console
    $ kubectl exec -n curl -ti "$(kubectl get pod -n curl -l app=curl -o jsonpath='{.items[0].metadata.name}')" -c curl -- curl -I https://httpbin.org:443
    HTTP/2 200
    date: Tue, 16 Mar 2021 22:19:00 GMT
    content-type: text/html; charset=utf-8
    content-length: 9593
    server: gunicorn/19.9.0
    access-control-allow-origin: *
    access-control-allow-credentials: true
    ```

    A `200 OK` response indicates the HTTPS request from the `curl` client to the `httpbin.org` website was successful.

1. Confirm the HTTPS requests fail when mesh-wide egress is disabled.
    ```bash
    # Assumes OSM is installed in the osm-system namespace
    kubectl patch meshconfig osm-mesh-config -n osm-system -p '{"spec":{"traffic":{"enableEgress":false}}}'  --type=merge
    ```
    ```console
    $ kubectl exec -n curl -ti "$(kubectl get pod -n curl -l app=curl -o jsonpath='{.items[0].metadata.name}')" -c curl -- curl -I https://httpbin.org:443
	  curl: (35) OpenSSL SSL_connect: SSL_ERROR_SYSCALL in connection to httpbin.org:443
	  command terminated with exit code 35
    ```

### Envoy configurations

When egress is globally enabled in the mesh, OSM controller programs each Envoy proxy sidecar to match external or unknown destinations using a default filter chain on the outbound listener configuration. The default filter chain is named `outbound-egress-filter-chain` as seen in the below configuration snippet. Any traffic that matches the default egress filter chain on the outbound listener is proxied to its original destination via the `passthrough-outbound` cluster.

```json
{
  "name": "outbound-listener",
  "active_state": {
    "version_info": "7",
    "listener": {
      "@type": "type.googleapis.com/envoy.config.listener.v3.Listener",
      "name": "outbound-listener",
      "address": {
        "socket_address": {
          "address": "0.0.0.0",
          "port_value": 15001
        }
      },
      "listener_filters": [
        {
          "name": "envoy.filters.listener.original_dst"
        }
      ],
      "traffic_direction": "OUTBOUND",
      "default_filter_chain": {
        "filters": [
          {
            "name": "envoy.filters.network.tcp_proxy",
            "typed_config": {
              "@type": "type.googleapis.com/envoy.extensions.filters.network.tcp_proxy.v3.TcpProxy",
              "stat_prefix": "passthrough-outbound",
              "cluster": "passthrough-outbound"
            }
          }
        ],
        "name": "outbound-egress-filter-chain"
      }
    },
    "last_updated": "2021-03-16T22:26:46.676Z"
  }
}
```

## Configuring Egress policies

OSM supports configuring fine grained policies for traffic destined to external endpoints using its Egress policy API. To use this feature, enable it during install:
```bash
osm install --set=OpenServiceMesh.featureFlags.enableEgressPolicy=true
```

Refer to the [Egress policy v1alpha1 API documentation](/docs/apidocs/policy/v1alpha1) to configure egress policies and [sample demos using the Egress policy API](/docs/tasks/traffic_management/demos/egress_policy_demo).

---
title: "Iptables Redirection"
description: "Iptables Redirection"
type: docs
---

# Iptables Redirection

OSM leverages [iptables](https://linux.die.net/man/8/iptables) to intercept and redirect traffic to and from pods participating in the service mesh to the Envoy proxy sidecar container running on each pod. Traffic redirected to the Envoy proxy sidecar is filtered and routed based on service mesh traffic policies.

## How it works

OSM sidecar injector service `osm-injector` injects an Envoy proxy sidecar on every pod created within the service mesh. Along with the Envoy proxy sidecar, `osm-injector` also injects an [init container](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/), a specialized container that runs before any application containers in a pod. The injected init container is responsible for bootstrapping the application pods with traffic redirection rules such that all outbound TCP traffic from a pod and all inbound traffic TCP traffic to a pod are redirected to the envoy proxy sidecar running on that pod. This redirection is set up by the init container by running a set of `iptables` commands.

### Ports reserved for traffic redirection

OSM reserves a set of port numbers to perform traffic redirection and provide admin access to the Envoy proxy sidecar. It is essential to note that these port numbers must not be used by application containers running in the mesh. Using any of these reserved port numbers will lead to the Envoy proxy sidecar not functioning correctly.

Following are the port numbers that are reserved for use by OSM:

1. `15000`: used by the [Envoy admin interface](https://www.envoyproxy.io/docs/envoy/latest/operations/admin) exposed over `localhost`
1. `15001`: used by the Envoy outbound listener to accept and proxy outbound traffic sent by applications within the pod
1. `15003`: used by the Envoy inbound listener to accept and proxy inbound traffic entering the pod destined to applications within the pod
1. `15010`: used by the Envoy inbound Prometheus listener to accept and proxy inbound traffic pertaining to scraping Envoy's Prometheus metrics
1. `15901`: used by Envoy to serve rewritten HTTP liveness probes
1. `15902`: used by Envoy to serve rewritten HTTP readiness probes
1. `15903`: used by Envoy to serve rewritten HTTP startup probes

### Application User ID (UID) reserved for traffic redirection

OSM reserves the user ID (UID) value `1500` for the Envoy proxy sidecar container. This user ID is of utmost importance while performing traffic interception and redirection to ensure the redirection does not result in a loop. The user ID value `1500` is used to program redirection rules to ensure redirected traffic from Envoy is not redirected back to itself!

Application containers must not used the reserved user ID value of `1500`.

### Types of traffic intercepted

Currently, OSM programs the Envoy proxy sidecar on each pod to only intercept inbound and outbound `TCP` traffic. This includes raw `TCP` traffic and any application traffic that uses `TCP` as the underlying transport protocol, such as `HTTP`, `gRPC` etc. This implies `UDP` and `ICMP` traffic which can be intercepted by `iptables` are not intercepted and redirected to the Envoy proxy sidecar.

### Iptables chains and rules

OSM's `osm-injector` service programs the init container to set up a set of `iptables` chains and rules to perform traffic interception and redirection. The following section provides details on the responsibility of these chains and rules.

OSM leverages four chains to perform traffic interception and redirection:

1. `PROXY_INBOUND`: chain to intercept inbound traffic entering the pod
1. `PROXY_IN_REDIRECT`: chain to redirect intercepted inbound traffic to the sidecar proxy's inbound listener
1. `PROXY_OUTPUT`: chain to intercept outbound traffic from applications within the pod
1. `PROXY_REDIRECT`: chain to redirect intercepted outbound traffic to the sidecar proxy's outbound listener

Each of the chains above are programmed with rules to intercept and redirect application traffic via the Envoy proxy sidecar.

### Global outbound IP range exclusions

Outbound TCP based traffic from applications is by default intercepted using the `iptables` rules programmed by OSM, and redirected to the Envoy proxy sidecar. In some cases, it might be desirable to not subject certain IP ranges to be redirected and routed by the Envoy proxy sidecar based on service mesh policies. A common use case to exclude IP ranges is to not route non-application logic based traffic via the Envoy proxy, such as traffic destined to the Kubernetes API server, or traffic destined to a cloud provider's instance metadata service. In such scenarios, excluding certain IP ranges from being subject to service mesh traffic routing policies becomes necessary.

OSM provides a means to specify a global list of IP ranges to exclude from outbound traffic interception in the following ways:

1. During OSM install using the `--set` option:
    ```bash
    # To exclude the IP ranges 1.1.1.1/32 and 2.2.2.2/24 from outbound interception
    osm install --set="OpenServiceMesh.outboundIPRangeExclusionList={1.1.1.1/32,2.2.2.2/24}
    ```

1. By setting the `outboundIPRangeExclusionList` key in the `osm-mesh-config` resource:
    ```bash
    ## Assumes OSM is installed in the osm-system namespace
    kubectl patch meshconfig osm-mesh-config -n osm-system -p '{"spec":{"traffic":{"outboundIPRangeExclusionList":["1.1.1.1/32", "2.2.2.2/24"]}}}'  --type=merge
    ```

Excluded IP ranges are stored in the `osm-mesh-config` `MeshConfig` custom resource and is read at the time of sidecar injection by `osm-injector`. These dynamically configurable IP ranges are programmed by the init container along with the static rules used to intercept and redirect traffic via the Envoy proxy sidecar. Excluded IP ranges will not be intercepted for traffic redirection to the Envoy proxy sidecar. Refer to the [outbound IP range exclusion demo](/docs/demos/outbound_ip_exclusion) to learn more.

### Outbound port exclusions

Outbound TCP based traffic from applications is by default intercepted using the `iptables` rules programmed by OSM, and redirected to the Envoy proxy sidecar. In some cases, it might be desirable to not subject certain ports to be redirected and routed by the Envoy proxy sidecar based on service mesh policies. A common use case to exclude ports is to not route non-application logic based traffic via the Envoy proxy, such as control plane traffic. In such scenarios, excluding certain ports from being subject to service mesh traffic routing policies becomes necessary.

#### 1. Global outbound port exclusions

In this set up the port exclusions would be applicable to all pods in the mesh.

OSM provides the means to specify a global list of ports to exclude from outbound traffic interception by the sidecar in the following ways:

1. During OSM install using the `--set` option:
    ```bash
    # To exclude the ports 6379 and 7070 from outbound sidecar interception
    osm install --set="OpenServiceMesh.outboundPortExclusionList={6379,7070}
    ```

1. By setting the `outboundPortExclusionList` key in the `osm-mesh-config` resource:
    ```bash
    ## Assumes OSM is installed in the osm-system namespace
    kubectl patch meshconfig osm-mesh-config -n osm-system -p '{"spec":{"traffic":{"outboundPortExclusionList":[6379, 7070]}}}'  --type=merge
    ```

#### 2. Pod scoped outbound port exclusions

OSM provides the means to specify a list of ports to exclude from outbound traffic interception at a per pod scope by annotating the pod with  `openservicemesh.io/outbound-port-exclusion-list=<comma separated list of ports>` option:
```bash
# To exclude the ports 6379 and 7070 from outbound interception on the pod
kubectl annotate pod <pod> openservicemesh.io/outbound-port-exclusion-list=6379,7070
```

Excluded ports are stored in the `osm-mesh-config` `MeshConfig` custom resource and as an annotation on the pod. Both of these are read and merged at the time of sidecar injection by `osm-injector`. These dynamically configurable ports are programmed by the init container along with the static rules used to intercept and redirect traffic via the Envoy proxy sidecar. Excluded ports will not be intercepted for traffic redirection to the Envoy proxy sidecar.

### Inbound port exclusions

Similar to outbound port exclusions described above, inbound traffic on pods can be excluded from being proxied to the sidecar based on the ports the traffic is directed to.

#### 1. Global inbound port exclusions

In this set up the port exclusions would be applicable to all pods in the mesh.

OSM provides the means to specify a global list of ports to exclude from inbound traffic interception by the sidecar in the following ways:

1. During OSM install using the `--set` option:
    ```bash
    # To exclude the ports 6379 and 7070 from inbound sidecar interception
    osm install --set="OpenServiceMesh.inboundPortExclusionList={6379,7070}
    ```

1. By setting the `inboundPortExclusionList` key in the `osm-mesh-config` resource:
    ```bash
    ## Assumes OSM is installed in the osm-system namespace
    kubectl patch meshconfig osm-mesh-config -n osm-system -p '{"spec":{"traffic":{"inboundPortExclusionList":[6379, 7070]}}}'  --type=merge
    ```

#### 2. Pod scoped inbound port exclusions

OSM provides the means to specify a list of ports to exclude from inbound traffic interception at a per pod scope by annotating the pod with  `openservicemesh.io/inbound-port-exclusion-list=<comma separated list of ports>` option:
```bash
# To exclude the ports 6379 and 7070 from inbound sidecar interception on the pod
kubectl annotate pod <pod> openservicemesh.io/inbound-port-exclusion-list=6379,7070
```

## Iptables configuration

Iptables rules are programmed by OSM's init container when a pod is created in the mesh. The rules are on the pod via a set of `iptables` commands run by the init container.

The following snippet from the demo `curl` client's init container spec shows the set of `iptables` commands along with exclusion rules for reference.

```console
Init Containers:
  osm-init:
    Container ID:  containerd://80f86af7bc64b7a70f7f2bf64242d735d857559a79cd97e206513368130902f1
    Image:         openservicemesh/init:v0.11.1
    Image ID:      docker.io/openservicemesh/init@sha256:eb1f6ab02aeaaba8f58aaa29406b1653d7a3983958ea040c2af8845136ed786c
    Port:          <none>
    Host Port:     <none>
    Command:
      /bin/sh
    Args:
      -c
      iptables -t nat -N PROXY_INBOUND && iptables -t nat -N PROXY_IN_REDIRECT && iptables -t nat -N PROXY_OUTPUT && iptables -t nat -N PROXY_REDIRECT && iptables -t nat -A PROXY_REDIRECT -p tcp -j REDIRECT --to-port 15001 && iptables -t nat -A PROXY_REDIRECT -p tcp --dport 15000 -j ACCEPT && iptables -t nat -A OUTPUT -p tcp -j PROXY_OUTPUT && iptables -t nat -A PROXY_OUTPUT -m owner --uid-owner 1500 -j RETURN && iptables -t nat -A PROXY_OUTPUT -d 127.0.0.1/32 -j RETURN && iptables -t nat -A PROXY_OUTPUT -j PROXY_REDIRECT && iptables -t nat -A PROXY_IN_REDIRECT -p tcp -j REDIRECT --to-port 15003 && iptables -t nat -A PREROUTING -p tcp -j PROXY_INBOUND && iptables -t nat -A PROXY_INBOUND -p tcp --dport 15010 -j RETURN && iptables -t nat -A PROXY_INBOUND -p tcp --dport 15901 -j RETURN && iptables -t nat -A PROXY_INBOUND -p tcp --dport 15902 -j RETURN && iptables -t nat -A PROXY_INBOUND -p tcp --dport 15903 -j RETURN && iptables -t nat -A PROXY_INBOUND -p tcp -j PROXY_IN_REDIRECT && iptables -t nat -I PROXY_OUTPUT -d 54.91.118.50/32 -j RETURN
    State:          Terminated
      Reason:       Completed
      Exit Code:    0
      Started:      Thu, 18 Mar 2021 16:14:30 -0700
      Finished:     Thu, 18 Mar 2021 16:14:30 -0700
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from curl-token-c4jv9 (ro)
```

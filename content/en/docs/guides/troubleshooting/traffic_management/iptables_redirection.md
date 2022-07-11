---
title: "Iptables Redirection"
description: "Troubleshooting Iptables interception and redirection"
type: docs
weight: 1
---

# Troubleshooting Iptables Interception and Redirection

## When traffic redirection is not working as expected

### 1. Confirm the pod has the Envoy sidecar container injected

The application pod should be injected with the Envoy proxy sidecar for traffic redirection to work as expected. Confirm this by ensuring the application pod is running and has the Envoy proxy sidecar container in ready state.

```console
$ kubectl get pod test-58d4f8ff58-wtz4f -n test
NAME                                READY   STATUS    RESTARTS   AGE
test-58d4f8ff58-wtz4f               2/2     Running   0          32s
```

### 2. Confirm OSM's init container has finished runnning successfully

OSM's init container `osm-init` is responsible for initializing individual application pods in the service mesh with traffic redirection rules to proxy application traffic via the Envoy proxy sidecar. The traffic redirection rules are set up using a set of `iptables` commands that run before any application containers in the pod are running.

Confirm OSM's init container has finished running successfully by running `kubectl describe` on the application pod, and verifying the `osm-init` container has terminated with an exit code of 0. The container's `State` property provides this information.

```console
$ kubectl describe pod test-58d4f8ff58-wtz4f -n test
Name:         test-58d4f8ff58-wtz4f
Namespace:    test
...
...
Init Containers:
  osm-init:
    Container ID:  containerd://98840f655f2310b2f441e11efe9dfcf894e4c57e4e26b928542ee698159100c0
    Image:         openservicemesh/init:2c18593efc7a31986a6ae7f412e73b6067e11a57
    Image ID:      docker.io/openservicemesh/init@sha256:24456a8391bce5d254d5a1d557d0c5e50feee96a48a9fe4c622036f4ab2eaf8e
    Port:          <none>
    Host Port:     <none>
    Command:
      /bin/sh
    Args:
      -c
      iptables -t nat -N PROXY_INBOUND && iptables -t nat -N PROXY_IN_REDIRECT && iptables -t nat -N PROXY_OUTPUT && iptables -t nat -N PROXY_REDIRECT && iptables -t nat -A PROXY_REDIRECT -p tcp -j REDIRECT --to-port 15001 && iptables -t nat -A PROXY_REDIRECT -p tcp --dport 15000 -j ACCEPT && iptables -t nat -A OUTPUT -p tcp -j PROXY_OUTPUT && iptables -t nat -A PROXY_OUTPUT -m owner --uid-owner 1500 -j RETURN && iptables -t nat -A PROXY_OUTPUT -d 127.0.0.1/32 -j RETURN && iptables -t nat -A PROXY_OUTPUT -j PROXY_REDIRECT && iptables -t nat -A PROXY_IN_REDIRECT -p tcp -j REDIRECT --to-port 15003 && iptables -t nat -A PREROUTING -p tcp -j PROXY_INBOUND && iptables -t nat -A PROXY_INBOUND -p tcp --dport 15010 -j RETURN && iptables -t nat -A PROXY_INBOUND -p tcp --dport 15901 -j RETURN && iptables -t nat -A PROXY_INBOUND -p tcp --dport 15902 -j RETURN && iptables -t nat -A PROXY_INBOUND -p tcp --dport 15903 -j RETURN && iptables -t nat -A PROXY_INBOUND -p tcp -j PROXY_IN_REDIRECT
    State:          Terminated
      Reason:       Completed
      Exit Code:    0
      Started:      Mon, 22 Mar 2021 09:26:14 -0700
      Finished:     Mon, 22 Mar 2021 09:26:14 -0700
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from frontend-token-5g488 (ro)
```

## When outbound IP range exclusions are configured

By default, all traffic using TCP as the underlying transport protocol are redirected via the Envoy proxy sidecar container. This means all TCP based outbound traffic from applications are redirected and routed via the Envoy proxy sidecar based on service mesh policies. When outbound IP range exclusions are configured, traffic belonging to these IP ranges will not be proxied to the Envoy sidecar.

If outbound IP ranges are configured to be excluded but being subject to service mesh policies, verify they are configured as expected.

### 1. Confirm outbound IP ranges are correctly configured in the `osm-mesh-config` MeshConfig resource

Confirm the outbound IP ranges to be excluded are set correctly:

```console
# Assumes OSM is installed in the osm-system namespace
$ kubectl get meshconfig osm-mesh-config -n osm-system -o jsonpath='{.spec.traffic.outboundIPRangeExclusionList}{"\n"}'
["1.1.1.1/32","2.2.2.2/24"]
```

The output shows the IP ranges that are excluded from outbound traffic redirection, `["1.1.1.1/32","2.2.2.2/24"]` in the example above.

### 2. Confirm outbound IP ranges are included in init container spec

When outbound IP range exclusions are configured, OSM's `osm-injector` service reads this configuration from the `osm-mesh-config` `MeshConfig` resource and programs `iptables` rules corresponding to these ranges so that they are excluded from outbound traffic redirection via the Envoy sidecar proxy.

Confirm OSM's `osm-init` init container spec has rules corresponding to the configured outbound IP ranges to exclude.

```console
$ kubectl describe pod test-58d4f8ff58-wtz4f -n test
Name:         test-58d4f8ff58-wtz4f
Namespace:    test
...
...
Init Containers:
  osm-init:
    Container ID:  containerd://98840f655f2310b2f441e11efe9dfcf894e4c57e4e26b928542ee698159100c0
    Image:         openservicemesh/init:2c18593efc7a31986a6ae7f412e73b6067e11a57
    Image ID:      docker.io/openservicemesh/init@sha256:24456a8391bce5d254d5a1d557d0c5e50feee96a48a9fe4c622036f4ab2eaf8e
    Port:          <none>
    Host Port:     <none>
    Command:
      /bin/sh
    Args:
      -c
      iptables -t nat -N PROXY_INBOUND && iptables -t nat -N PROXY_IN_REDIRECT && iptables -t nat -N PROXY_OUTPUT && iptables -t nat -N PROXY_REDIRECT && iptables -t nat -A PROXY_REDIRECT -p tcp -j REDIRECT --to-port 15001 && iptables -t nat -A PROXY_REDIRECT -p tcp --dport 15000 -j ACCEPT && iptables -t nat -A OUTPUT -p tcp -j PROXY_OUTPUT && iptables -t nat -A PROXY_OUTPUT -m owner --uid-owner 1500 -j RETURN && iptables -t nat -A PROXY_OUTPUT -d 127.0.0.1/32 -j RETURN && iptables -t nat -A PROXY_OUTPUT -j PROXY_REDIRECT && iptables -t nat -A PROXY_IN_REDIRECT -p tcp -j REDIRECT --to-port 15003 && iptables -t nat -A PREROUTING -p tcp -j PROXY_INBOUND && iptables -t nat -A PROXY_INBOUND -p tcp --dport 15010 -j RETURN && iptables -t nat -A PROXY_INBOUND -p tcp --dport 15901 -j RETURN && iptables -t nat -A PROXY_INBOUND -p tcp --dport 15902 -j RETURN && iptables -t nat -A PROXY_INBOUND -p tcp --dport 15903 -j RETURN && iptables -t nat -A PROXY_INBOUND -p tcp -j PROXY_IN_REDIRECT && iptables -t nat -I PROXY_OUTPUT -d 1.1.1.1/32 -j RETURN && && iptables -t nat -I PROXY_OUTPUT -d 2.2.2.2/24 -j RETURN
    State:          Terminated
      Reason:       Completed
      Exit Code:    0
      Started:      Mon, 22 Mar 2021 09:26:14 -0700
      Finished:     Mon, 22 Mar 2021 09:26:14 -0700
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from frontend-token-5g488 (ro)
```

In the example above, the following `iptables` commands are responsible for explicitly ignoring the configured outbound IP ranges (`1.1.1.1/32 and 2.2.2.2/24`) from being redirected to the Envoy proxy sidecar.
```console
iptables -t nat -I PROXY_OUTPUT -d 1.1.1.1/32 -j RETURN
iptables -t nat -I PROXY_OUTPUT -d 2.2.2.2/24 -j RETURN
```

## When outbound port exclusions are configured

By default, all traffic using TCP as the underlying transport protocol are redirected via the Envoy proxy sidecar container. This means all TCP based outbound traffic from applications are redirected and routed via the Envoy proxy sidecar based on service mesh policies. When outbound port exclusions are configured, traffic belonging to these ports will not be proxied to the Envoy sidecar.

If outbound ports are configured to be excluded but being subject to service mesh policies, verify they are configured as expected.

### 1. Confirm global outbound ports are correctly configured in the `osm-mesh-config` MeshConfig resource

Confirm the outbound ports to be excluded are set correctly:

```console
# Assumes OSM is installed in the osm-system namespace
$ kubectl get meshconfig osm-mesh-config -n osm-system -o jsonpath='{.spec.traffic.outboundPortExclusionList}{"\n"}'
[6379,7070]
```

The output shows the ports that are excluded from outbound traffic redirection, `[6379,7070]` in the example above.

### 2. Confirm pod level outbound ports are correctly annotated on the pod

Confirm the outbound ports to be excluded on a pod are set correctly:

```console
$ kubectl get pod POD_NAME -o jsonpath='{.metadata.annotations}' -n POD_NAMESPACE'
map[openservicemesh.io/outbound-port-exclusion-list:8080]
```

The output shows the ports that are excluded from outbound traffic redirection on the pod, `8080` in the example above.

### 3. Confirm outbound ports are included in init container spec

When outbound port exclusions are configured, OSM's `osm-injector` service reads this configuration from the `osm-mesh-config` `MeshConfig` resource and from the annotations on the pod, and programs `iptables` rules corresponding to these ranges so that they are excluded from outbound traffic redirection via the Envoy sidecar proxy.

Confirm OSM's `osm-init` init container spec has rules corresponding to the configured outbound ports to exclude.

```console
$ kubectl describe pod test-58d4f8ff58-wtz4f -n test
Name:         test-58d4f8ff58-wtz4f
Namespace:    test
...
...
Init Containers:
  osm-init:
    Container ID:  containerd://98840f655f2310b2f441e11efe9dfcf894e4c57e4e26b928542ee698159100c0
    Image:         openservicemesh/init:2c18593efc7a31986a6ae7f412e73b6067e11a57
    Image ID:      docker.io/openservicemesh/init@sha256:24456a8391bce5d254d5a1d557d0c5e50feee96a48a9fe4c622036f4ab2eaf8e
    Port:          <none>
    Host Port:     <none>
    Command:
      /bin/sh
    Args:
      -c
      iptables -t nat -N PROXY_INBOUND && iptables -t nat -N PROXY_IN_REDIRECT && iptables -t nat -N PROXY_OUTPUT && iptables -t nat -N PROXY_REDIRECT && iptables -t nat -A PROXY_REDIRECT -p tcp -j REDIRECT --to-port 15001 && iptables -t nat -A PROXY_REDIRECT -p tcp --dport 15000 -j ACCEPT && iptables -t nat -A OUTPUT -p tcp -j PROXY_OUTPUT && iptables -t nat -A PROXY_OUTPUT -m owner --uid-owner 1500 -j RETURN && iptables -t nat -A PROXY_OUTPUT -d 127.0.0.1/32 -j RETURN && iptables -t nat -A PROXY_OUTPUT -j PROXY_REDIRECT && iptables -t nat -A PROXY_IN_REDIRECT -p tcp -j REDIRECT --to-port 15003 && iptables -t nat -A PREROUTING -p tcp -j PROXY_INBOUND && iptables -t nat -A PROXY_INBOUND -p tcp --dport 15010 -j RETURN && iptables -t nat -A PROXY_INBOUND -p tcp --dport 15901 -j RETURN && iptables -t nat -A PROXY_INBOUND -p tcp --dport 15902 -j RETURN && iptables -t nat -A PROXY_INBOUND -p tcp --dport 15903 -j RETURN && iptables -t nat -A PROXY_INBOUND -p tcp -j PROXY_IN_REDIRECT && iptables -t nat -I PROXY_OUTPUT -p tcp --match multiport --dports 6379,7070,8080 -j RETURN
    State:          Terminated
      Reason:       Completed
      Exit Code:    0
      Started:      Mon, 22 Mar 2021 09:26:14 -0700
      Finished:     Mon, 22 Mar 2021 09:26:14 -0700
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from frontend-token-5g488 (ro)
```

In the example above, the following `iptables` commands are responsible for explicitly ignoring the configured outbound ports (`6379, 7070 and 8080`) from being redirected to the Envoy proxy sidecar.
```console
iptables -t nat -I PROXY_OUTPUT -p tcp --match multiport --dports 6379,7070,8080 -j RETURN
```

## When network interface exclusions are configured

By default, all traffic using TCP as the underlying transport protocol are redirected via the Envoy proxy sidecar container. This means all TCP based outbound traffic from applications are redirected and routed via the Envoy proxy sidecar based on service mesh policies. When network interface exclusions are configured, traffic belonging to these network interfaces will not be proxied to the Envoy sidecar.

If you have network interfaces that are subjected to mesh policies, verify their configuration to ensure they are properly excluded.

### 1. Confirm network interfaces are correctly configured in the `osm-mesh-config` MeshConfig resource

Confirm the network interfaces to be excluded are set correctly:

```console
# Assumes OSM is installed in the osm-system namespace
$ kubectl get meshconfig osm-mesh-config -n osm-system -o jsonpath='{.spec.traffic.networkInterfaceExclusionList}{"\n"}'
["net1","net2"]
```

The output shows the network interfaces that are excluded from outbound traffic redirection, `["net1","net2"]` in the example above.

### 2. Confirm network interfaces are included in init container spec

When network interface exclusions are configured, OSM's `osm-injector` service reads this configuration from the `osm-mesh-config` `MeshConfig` resource and programs `iptables` rules corresponding to these ranges so that they are excluded from traffic redirection via the Envoy sidecar proxy.

Confirm OSM's `osm-init` init container spec has rules corresponding to the configured network interfaces to exclude.

```console
$ kubectl describe pod server-85f4bc46c5-hprkw
Name:         server-85f4bc46c5-hprkw
Namespace:    default
...
...
Init Containers:
  osm-init:
    Container ID:  containerd://98840f655f2310b2f441e11efe9dfcf894e4c57e4e26b928542ee698159100c0
    Image:         openservicemesh/init:2c18593efc7a31986a6ae7f412e73b6067e11a57
    Image ID:      docker.io/openservicemesh/init@sha256:24456a8391bce5d254d5a1d557d0c5e50feee96a48a9fe4c622036f4ab2eaf8e
    Port:          <none>
    Host Port:     <none>
    Command:
      /bin/sh
    Args:
      -c
      iptables-restore --noflush <<EOF
      # OSM sidecar interception rules
      *nat
      :OSM_PROXY_INBOUND - [0:0]
      :OSM_PROXY_IN_REDIRECT - [0:0]
      :OSM_PROXY_OUTBOUND - [0:0]
      :OSM_PROXY_OUT_REDIRECT - [0:0]
      -A OSM_PROXY_IN_REDIRECT -p tcp -j REDIRECT --to-port 15003
      -A PREROUTING -p tcp -j OSM_PROXY_INBOUND
      -A OSM_PROXY_INBOUND -p tcp --dport 15010 -j RETURN
      -A OSM_PROXY_INBOUND -p tcp --dport 15901 -j RETURN
      -A OSM_PROXY_INBOUND -p tcp --dport 15902 -j RETURN
      -A OSM_PROXY_INBOUND -p tcp --dport 15903 -j RETURN
      -A OSM_PROXY_INBOUND -p tcp --dport 15904 -j RETURN
      -A OSM_PROXY_INBOUND -p tcp -j OSM_PROXY_IN_REDIRECT
      -I OSM_PROXY_INBOUND -i net1 -j RETURN
      -I OSM_PROXY_INBOUND -i net2 -j RETURN
      -A OSM_PROXY_OUT_REDIRECT -p tcp -j REDIRECT --to-port 15001
      -A OSM_PROXY_OUT_REDIRECT -p tcp --dport 15000 -j ACCEPT
      -A OUTPUT -p tcp -j OSM_PROXY_OUTBOUND
      -A OSM_PROXY_OUTBOUND -o lo ! -d 127.0.0.1/32 -m owner --uid-owner 1500 -j OSM_PROXY_IN_REDIRECT
      -A OSM_PROXY_OUTBOUND -o lo -m owner ! --uid-owner 1500 -j RETURN
      -A OSM_PROXY_OUTBOUND -m owner --uid-owner 1500 -j RETURN
      -A OSM_PROXY_OUTBOUND -d 127.0.0.1/32 -j RETURN
      -A OSM_PROXY_OUTBOUND -o net1 -j RETURN
      -A OSM_PROXY_OUTBOUND -o net2 -j RETURN
      -A OSM_PROXY_OUTBOUND -j OSM_PROXY_OUT_REDIRECT
      COMMIT
      EOF

    State:          Terminated
      Reason:       Completed
      Exit Code:    0
      Started:      Wed, 27 Apr 2022 11:19:16 -0500
      Finished:     Wed, 27 Apr 2022 11:19:16 -0500
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-gqcjp (ro)
```

In the example above, the following `iptables` commands are responsible for explicitly ignoring the configured network interfaces (`net1 and net2`) from being redirected to the Envoy proxy sidecar.
```console
-I OSM_PROXY_INBOUND -i net1 -j RETURN
-I OSM_PROXY_INBOUND -i net2 -j RETURN
...
-A OSM_PROXY_OUTBOUND -o net1 -j RETURN
-A OSM_PROXY_OUTBOUND -o net2 -j RETURN
```

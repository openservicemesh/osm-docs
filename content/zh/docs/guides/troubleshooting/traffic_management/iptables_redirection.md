---
title: "Iptables 重定向"
description: "Iptables 拦截和重定向故障排查"
type: docs
weight: 1
---

# Iptables 拦截和重定向故障排查

## 当流量重定向未按预期工作时

### 1. 确认 pod 已注入 Envoy sidecar 容器

应用程序 pod 应注入 Envoy 代理 sidecar，使流量重定向按预期工作。通过确保应用程序 pod 正在运行并且 Envoy 代理 sidecar 容器处于就绪状态来确认这一点。

```console
$ kubectl get pod test-58d4f8ff58-wtz4f -n test
NAME                                READY   STATUS    RESTARTS   AGE
test-58d4f8ff58-wtz4f               2/2     Running   0          32s
```

### 2. 确认 OSM 的 init 容器已成功运行

OSM 的初始化容器 `osm-init` 负责使用流量重定向规则初始化服务网格中的单个应用程序 pod，以通过 Envoy 代理 sidecar 来代理应用程序流量。流量重定向规则是使用一组 `iptables` 命令设置的，这些命令在 pod 中的任何应用程序容器运行之前运行。

通过在应用程序 pod 上运行 `kubectl describe` 来确认 OSM 的初始化容器已成功完成运行，并验证 `osm-init` 容器已终止，退出代码为 0。容器的 `State` 属性提供了此信息。

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

## 配置出站 IP 范围排除项时

默认情况下，所有使用 TCP 作为底层传输协议的流量都通过 Envoy 代理 sidecar 容器重定向。这意味着来自应用程序的所有基于 TCP 的出站流量都将通过基于服务网格策略的 Envoy 代理 Sidecar 进行重定向和路由。配置出站 IP 范围排除项后，属于这些 IP 范围的流量将不会被代理到 Envoy sidecar。

如果出站 IP 范围配置为排除但仍受服务网格策略的约束，请验证它们是否按预期配置。

### 1. 确认在 `osm-mesh-config` MeshConfig 资源中正确配置了出站 IP 范围

确认要排除的出站 IP 范围设置正确：

```console
# Assumes OSM is installed in the osm-system namespace
$ kubectl get meshconfig osm-mesh-config -n osm-system -o jsonpath='{.spec.traffic.outboundIPRangeExclusionList}{"\n"}'
["1.1.1.1/32","2.2.2.2/24"]
```

输出显示了从出站流量重定向中排除的 IP 范围，在上面的示例中为 `["1.1.1.1/32","2.2.2.2/24"]`。

### 2. 确认出站 IP 范围包含在初始化容器规范中

当配置出站 IP 范围排除时，OSM 的 `osm-injector` 服务从 `osm-mesh-config` `MeshConfig` 资源中读取此配置，并编写与这些范围相对应的 `iptables` 规则，以便将它们排除在通过 Envoy sidecar 代理出站流量重定向之外。

确认 OSM 的 `osm-init` 初始化容器规范具有与要排除的已配置出站 IP 范围相对应的规则。

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

在上面的示例中，以下 `iptables` 命令负责明确忽略配置的出站 IP 范围（`1.1.1.1/32 和 2.2.2.2/24`）被重定向到 Envoy 代理边车。

```console
iptables -t nat -I PROXY_OUTPUT -d 1.1.1.1/32 -j RETURN
iptables -t nat -I PROXY_OUTPUT -d 2.2.2.2/24 -j RETURN
```

## 配置出站端口排除项时

默认情况下，所有使用 TCP 作为底层传输协议的流量都通过 Envoy 代理 sidecar 容器重定向。 这意味着来自应用程序的所有基于 TCP 的出站流量都将通过基于服务网格策略的 Envoy 代理 Sidecar 进行重定向和路由。配置出站端口排除时，属于这些端口的流量不会被代理到 Envoy sidecar。

如果出站端口配置为排除但仍受服务网格策略的约束，请验证它们是否按预期配置。

### 1. 确认在 `osm-mesh-config` MeshConfig 资源中正确配置了全局出站端口

确认要排除的出站端口设置正确：

```console
# Assumes OSM is installed in the osm-system namespace
$ kubectl get meshconfig osm-mesh-config -n osm-system -o jsonpath='{.spec.traffic.outboundPortExclusionList}{"\n"}'
[6379,7070]
```

输出显示从出站流量重定向中排除的端口，在上面的示例中为 `[6379，7070]`。

### 2. 确认 Pod 级别的出站端口已在 Pod 上正确注释


确认要在 Pod 上排除的出站端口设置正确：

```console
$ kubectl get pod POD_NAME -o jsonpath='{.metadata.annotations}' -n POD_NAMESPACE'
map[openservicemesh.io/outbound-port-exclusion-list:8080]
```

输出显示从 Pod 上的出站流量重定向中排除的端口，在上面的示例中为 `8080`。

### 3. 确认出站端口包含在初始化容器规范中

当配置了出站端口排除时，OSM 的 `osm-injector` 服务从 `osm-mesh-config` `MeshConfig` 资源和 pod 上的注解中读取此配置，并编写与这些范围相对应的 `iptables` 规则，以便它们被排除在通过 Envoy sidecar 代理进行的出站流量重定向之外。

确认 OSM 的 `osm-init` 初始化容器规范具有与配置的出站端口对应的规则以排除。

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
在上面的示例中，以下 `iptables` 命令负责显式忽略配置的出站端口（`6379，7070 和 8080`）从重定向到 Envoy 代理 sidecar。

```console
iptables -t nat -I PROXY_OUTPUT -p tcp --match multiport --dports 6379,7070,8080 -j RETURN

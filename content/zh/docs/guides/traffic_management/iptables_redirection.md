---
title: "Iptables 重定向"
description: "Iptables 重定向"
type: docs
weight: 1
---

# Iptables 重定向

OSM 利用 [iptables](https://linux.die.net/man/8/iptables) 来拦截和重定向进出参与服务网格的 pod 的流量到每个 pod 上运行的 Envoy 代理 sidecar 容器。重定向到 Envoy 代理 sidecar 的流量会根据服务网格流量策略进行过滤和路由。

## 工作原理

OSM sidecar 注入器服务 `osm-injector` 在服务网格中创建的每个 pod 上注入一个 Envoy 代理 sidecar。 除了 Envoy 代理 sidecar，`osm-injector` 还注入了一个 [初始化容器](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/)，这是一个在 Pod 中任何应用程序之前运行的专用容器。注入的初始化容器负责使用流量重定向规则引导应用程序 pod，以便将来自 pod 的所有出站 TCP 流量和所有到 pod 的入站 TCP 流量重定向到运行在该 pod 上的 envoy 代理 sidecar。此重定向由初始化容器通过运行一组 `iptables` 命令设置。

### 为流量重定向保留的端口

Following are the port numbers that are reserved for use by OSM:
OSM 保留一组端口号来执行流量重定向并提供对 Envoy 代理 sidecar 的管理员访问权限。 需要注意的是，网格中运行的应用程序容器不得使用这些端口号。使用这些保留端口号中的任何一个都将导致 Envoy 代理 sidecar 无法正常运行。

以下是保留供 OSM 使用的端口号：

1. `15000`：被暴露在`localhost`上的[Envoy管理界面](https://www.envoyproxy.io/docs/envoy/latest/operations/admin)使用
2. `15001`：Envoy 出站监听器用于接受和代理 pod 内应用程序发送的出站流量
3. `15003`：Envoy 入站监听器用于接受和代理进入 Pod 发往 Pod 内的应用程序的入站流量，
4. `15010`：Envoy 入站 Prometheus 侦听器用于接受和代理与抓取 Envoy 的 Prometheus 指标有关的入站流量
5. `15901`：Envoy 用于服务重写的 HTTP 活跃度探测
6. `15902`：Envoy 用于服务重写的 HTTP 就绪探测
7. `15903`：Envoy 用于服务重写的 HTTP 启动探测

The following are the port numbers that are reserved for use by OSM and allow traffic to bypass Envoy:
1. `15904`: used by `osm-healthcheck` to serve `tcpSocket` health probes rewritten to `httpGet` health probes

### 为流量重定向保留的应用程序用户 ID （UID）

OSM 为 Envoy 代理 sidecar 容器保留用户 ID (UID) 值 `1500`。此用户 ID 在执行流量拦截和重定向时至关重要，以确保重定向不会导致循环。用户 ID 值 `1500` 用于编程重定向规则，以确保来自 Envoy 的重定向流量不会重定向回自身！

应用程序容器不得使用保留的用户 ID 值 `1500`。

### 拦截的流量类型

目前，OSM 对每个 pod 上的 Envoy 代理 sidecar 进行编程，以仅拦截入站和出站 TCP 流量。这包括原始的 `TCP `流量和任何使用 `TCP` 作为底层传输协议的应用程序流量，例如 `HTTP`、`gRPC`等。这意味着 `UDP` 和 `ICMP` 流量可以被 `iptables` 拦截不会被拦截并重定向到 Envoy 代理 sidecar。

### iptables 链和规则

OSM 的 `osm-injector` 服务对初始化容器进行编程以设置一组 `iptables` 链和规则来执行流量拦截和重定向。以下部分详细介绍了这些链和规则的责任。

OSM 利用四个链来执行流量拦截和重定向：

1. `PROXY_INBOUND`：拦截进入 pod 的入站流量的链
2. `PROXY_IN_REDIRECT`：将拦截的入站流量重定向到 Sidecar 代理的入站侦听器的链
3. `PROXY_OUTPUT`: 用于拦截 pod 内应用程序的出站流量的链
4. `PROXY_REDIRECT`：将截获的出站流量重定向到 sidecar 代理的出站侦听器的链

上面的每条链都使用规则进行编程，以通过 Envoy 代理 sidecar 拦截和重定向应用程序流量。

## 出站 IP 范围排除

默认情况下，来自应用程序的基于 TCP 的出站流量使用由 OSM 编程的 `iptables` 规则拦截，并重定向到 Envoy 代理 sidecar。在某些情况下，可能不希望 Envoy 代理 sidecar 根据服务网格策略重定向和路由某些 IP 范围。排除 IP 范围的常见用例是不通过 Envoy 代理路由基于非应用程序逻辑的流量，例如发往 Kubernetes API 服务器的流量，或发往云提供商的实例元数据服务的流量。在这种情况下，排除某些 IP 范围不受服务网格流量路由策略的约束就变得很有必要了。

可以在全局网格范围或每个 pod 范围内排除出站 IP 范围。

### 1. 全局出站 IP 范围排除

OSM 提供了一种方法来指定 IP 范围的全局列表，以从适用于网格中所有 pod 的出站流量拦截中排除，如下所示：

1. 在 OSM 安装时通过 `--set` 选项：

    ```bash
    # To exclude the IP ranges 1.1.1.1/32 and 2.2.2.2/24 from outbound interception
    osm install --set="osm.outboundIPRangeExclusionList={1.1.1.1/32,2.2.2.2/24}
    ```

2. 通过修改 `osm-mesh-config` 资源的`outboundIPRangeExclusionList` 字段：
    ```bash
    ## Assumes OSM is installed in the osm-system namespace
    kubectl patch meshconfig osm-mesh-config -n osm-system -p '{"spec":{"traffic":{"outboundIPRangeExclusionList":["1.1.1.1/32", "2.2.2.2/24"]}}}'  --type=merge
    ```

   如果将 IP 范围设置为安装后排除，请确保重新启动受监视命名空间中的 Pod，以使此更改生效。

全局排除的 IP 范围存储在 `osm-mesh-config` `MeshConfig` 自定义资源中，并在 `osm-injector` 注入 sidecar 时读取。这些动态可配置的 IP 范围由初始化容器以及用于通过 Envoy 代理 sidecar 拦截和重定向流量的静态规则进行编程。排除的 IP 范围不会被拦截以将流量重定向到 Envoy 代理 sidecar。请参阅[出站 IP 范围排除演示](/docs/demos/outbound_ip_exclusion) 了解更多信息。

### 2. Pod 范围的出站 IP 范围排除项

可以在 pod 范围内配置出站 IP 范围排除，方法是通过注释 pod 以将 IP CIDR 范围的逗号分隔列表指定为`openservicemesh.io/outbound-ip-range-exclusion-list=<逗号分隔的 IP CIDR 列表>`。

```bash
# To exclude the IP ranges 10.244.0.0/16 and 10.96.0.0/16 from outbound interception on the pod
kubectl annotate pod <pod> openservicemesh.io/outbound-ip-range-exclusion-list="10.244.0.0/16,10.96.0.0/16"
```

在创建容器后对 IP 范围进行注解时，请确保重新启动相应的容器，以使此更改生效。

## 出站 IP 范围包含

默认情况下，来自应用程序的基于 TCP 的出站流量使用由 OSM 编程的 `iptables` 规则拦截，并重定向到 Envoy 代理 sidecar。在某些情况下，可能希望仅让特定 IP 范围由 Envoy 代理 sidecar 基于服务网格策略重定向和路由，并且剩余流量不代理到 sidecar。在这种情况下，可以指定包含 IP 范围。

可以在全局网格范围或每个 pod 范围内指定出站包含 IP 范围。

### 1. 全局出站 IP 范围包含

OSM 提供了指定 IP 范围的全局列表的方法，这些 IP 范围包括适用于网格中所有 Pod 的出站流量拦截，如下所示：

1. 在 OSM 安装时通过 `--set` 选项：

    ```bash
    # To include the IP ranges 1.1.1.1/32 and 2.2.2.2/24 for outbound interception
    osm install --set="osm.outboundIPRangeInclusionList={1.1.1.1/32,2.2.2.2/24}
    ```

2. 通过修改 `osm-mesh-config` 资源的`outboundIPRangeInclusionList` 字段：

    ```bash
    ## Assumes OSM is installed in the osm-system namespace
    kubectl patch meshconfig osm-mesh-config -n osm-system -p '{"spec":{"traffic":{"outboundIPRangeInclusionList":["1.1.1.1/32", "2.2.2.2/24"]}}}'  --type=merge
    ```

   当在安装后 IP 范围设置为包含时，请确保重新启动被监控的命名空间中的 Pod，以使此更改生效。

全局包含的 IP 范围存储在 `osm-mesh-config` `MeshConfig` 自定义资源中，并在 `osm-injector` 注入 sidecar 时读取。这些动态可配置的 IP 范围由初始化容器以及用于通过 Envoy 代理 sidecar 拦截和重定向流量的静态规则进行编程。指定包含 IP 范围之外的 IP 地址不会被拦截以将流量重定向到 Envoy 代理 sidecar。

### 2. Pod 范围内的出站 IP 范围包含

可以在 pod 范围内配置出站 IP 范围包含，方法是通过注释 pod 以将 IP CIDR 范围的逗号分隔列表指定为`openservicemesh.io/outbound-ip-range-inclusion-list=<逗号分隔的 IP CIDR 列表>`。

```bash
# To include the IP ranges 10.244.0.0/16 and 10.96.0.0/16 for outbound interception on the pod
kubectl annotate pod <pod> openservicemesh.io/outbound-ip-range-inclusion-list="10.244.0.0/16,10.96.0.0/16"
```

如果 IP 范围在 Pod 创建后被注解，请确保重新启动相应的 Pod 以使此更改生效。

## 出站端口排除

默认情况下，来自应用程序的基于 TCP 的出站流量使用由 OSM 编程的 `iptables` 规则拦截，并重定向到 Envoy 代理 sidecar。在某些情况下，可能不希望某些端口被 Envoy 代理 sidecar 基于服务网格策略重定向和路由。排除端口的常见用例是不通过 Envoy 代理路由基于非应用程序逻辑的流量，例如控制平面流量。在这种情况下，将某些端口排除在服务网格流量路由策略之外就变得很有必要了。

可以在全局网格范围或每个 pod 范围内排除出站端口。

### 1. 全局出站端口排除

OSM 提供了一种方法来指定要从出站流量拦截中排除的端口的全局列表，这些端口适用于网格中的所有 Pod，如下所示：

1. 在 OSM 安装时通过 `--set` 选项：

    ```bash
    # To exclude the ports 6379 and 7070 from outbound sidecar interception
    osm install --set="osm.outboundPortExclusionList={6379,7070}
    ```

2. 通过修改 `osm-mesh-config` 资源的 `outboundPortExclusionList` 字段：

    ```bash
    ## Assumes OSM is installed in the osm-system namespace
    kubectl patch meshconfig osm-mesh-config -n osm-system -p '{"spec":{"traffic":{"outboundPortExclusionList":[6379, 7070]}}}'  --type=merge
    ```

   当端口设置为安装后排除时，请确保重新启动受监控命名空间中的 pod 以使此更改生效。

全局排除的端口存储在 `osm-mesh-config` `MeshConfig` 自定义资源中，并在 `osm-injector` 注入 sidecar 时读取。这些动态可配置的端口由初始化容器以及用于通过 Envoy 代理 sidecar 拦截和重定向流量的静态规则进行编程。排除的端口不会被拦截以将流量重定向到 Envoy 代理 sidecar。

### 2. Pod 范围的出站端口排除

可以在 pod 范围内配置出站端口排除，方法是使用逗号分隔的端口列表将 pod 注解为`openservicemesh.io/outbound-port-exclusion-list=<逗号分隔的端口列表>`：

```bash
# To exclude the ports 6379 and 7070 from outbound interception on the pod
kubectl annotate pod <pod> openservicemesh.io/outbound-port-exclusion-list=6379,7070
```

在创建容器后对端口进行注解时，请确保重新启动相应的容器，以使此更改生效。

## 入站端口排除

与上述出站端口排除类似，Pod 上的入站流量可以根据流量指向的端口被排除在代理到 sidecar 之外。

### 1. 全局入站端口排除

OSM 提供了一种方法来指定要从入站流量拦截中排除的端口的全局列表，这些端口适用于网格中的所有 Pod，如下所示：

1. 在 OSM 安装时通过 `--set` 选项：
    ```bash
    # To exclude the ports 6379 and 7070 from inbound sidecar interception
    osm install --set="osm.inboundPortExclusionList={6379,7070}
    ```

2. 修改 `osm-mesh-config` 资源中的 `inboundPortExclusionList` 字段：
    ```bash
    ## Assumes OSM is installed in the osm-system namespace
    kubectl patch meshconfig osm-mesh-config -n osm-system -p '{"spec":{"traffic":{"inboundPortExclusionList":[6379, 7070]}}}'  --type=merge
    ```

   当端口设置为安装后排除时，请确保重新启动受监控命名空间中的 pod 以使此更改生效。

### 2. Pod 范围的入站端口排除

可以在 pod 范围内配置入站端口排除，方法是使用逗号分隔的端口列表将 pod 注解为`openservicemesh.io/inbound-port-exclusion-list=<逗号分隔的端口列表>`：

```bash
# To exclude the ports 6379 and 7070 from inbound sidecar interception on the pod
kubectl annotate pod <pod> openservicemesh.io/inbound-port-exclusion-list=6379,7070
```

在创建容器后对端口进行注解时，请确保重新启动相应的容器，以使此更改生效。

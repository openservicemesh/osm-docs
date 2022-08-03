---
title: "出口"
description: "启用对互联网和服务网格外部服务的访问。"
type: docs
weight: 6
---

# 出口

## 允许访问互联网和网外服务（出口）

本文档描述了启用对 Internet 和服务网格外部服务（称为 `Egress` 流量）的访问所需的步骤。

OSM 将所有出站流量从网格中的 pod 重定向到 pod 的 sidecar 代理。出站流量可以分为两类：

1. 到 Mesh 集群内服务的流量，简称网格内流量
2. 到网格集群外部服务的流量，称为出口流量

虽然网格内流量基于 L7 流量策略进行路由，但出口流量的路由方式不同，并且不受网内流量策略的约束。OSM 支持以直通方式访问外部服务，而无需对此类流量进行过滤策略。

## 配置出口

有两种配置出口的机制：

1. 使用出口策略 API：提供对外部流量的细粒度访问控制
2. 使用网格范围的全局出口直通设置：该设置打开或关闭并影响网格中的所有 Pod，启用它允许发往网格外目的地的流量从 Pod 出口。

## 1. 配置出口策略

OSM 支持使用其 [Egress policy API][1] 为发往外部端点的流量配置细粒度策略。要使用此功能，如果未启用，请启用它：

```bash
# Replace osm-system with the namespace where OSM is installed
kubectl patch meshconfig osm-mesh-config -n osm-system -p '{"spec":{"featureFlags":{"enableEgressPolicy":true}}}'  --type=merge
```

请参阅[出口策略演示](/docs/demos/egress_policy) 和 [API 文档][1]，了解如何为各种协议配置路由出口流量的策略。

## 2. 配置网格范围内的出口直通

### 启用网格范围的出口直通到外部目的地

在 OSM 安装或安装后，可以在网格范围内启用 Egress。当在网格范围内启用出口时，允许来自 pod 的出站流量从 pod 出口，只要流量不匹配网格内流量策略就会被拒绝。

1. 在 OSM 安装时（默认 `osm.enableEgress=false`）：

   ```bash
   osm install --set osm.enableEgress=true
   ```

2. 在 OSM 安装之后：

    `osm-controller` 从 osm 网格控制平面命名空间（默认为 `osm-system`）中的 `osm-mesh-config` `MeshConfig` 自定义资源中检索出口配置。 使用 `kubectl patch` 将 `osm-mesh-config` 资源中的 `enableEgress` 设置为 `true`。

   ```bash
   # Replace osm-system with the namespace where OSM is installed
   kubectl patch meshconfig osm-mesh-config -n osm-system -p '{"spec":{"traffic":{"enableEgress":true}}}' --type=merge
   ```

### 禁用到外部目标的网格范围的出口直通

与启用出口类似，可以在 OSM 安装或安装后禁用网格范围的出口。

1. 在 OSM 安装时：

   ```bash
   osm install --set osm.enableEgress=false
   ```

2. 在 OSM 安装之后：
   使用 `kubectl patch` 将 `osm-mesh-config` 资源中的 `enableEngress` 设置为 `false`。
   
   ```bash
   # Replace osm-system with the namespace where OSM is installed
   kubectl patch meshconfig osm-mesh-config -n osm-system -p '{"spec":{"traffic":{"enableEgress":false}}}'  --type=merge
   ```

禁用出口后，来自网格内 pod 的流量将无法访问集群外的外部服务。

### 工作原理

当在网格范围内启用出口时，OSM 控制器使用通配符规则对网格中的每个 Envoy 代理 sidecar 进行编程，该规则匹配与网格内服务不对应的出站目的地。匹配此类外部流量的通配符规则只是将流量按原样代理到其原始目的地，而无需使其服从 L4 或 L7 流量策略。

OSM 支持使用 TCP 作为底层传输的流量的出口。这包括原始 TCP 流量、HTTP、HTTPS、gRPC 等。

由于网格范围的出口是一个全局设置，并且作为到未知目的地的通道运行，因此不可能对出口流量进行细粒度的访问控制（例如应用 TCP 或 HTTP 路由策略）。

请参阅 [Egress 直通演示](/docs/demos/egress_passthrough) 了解更多信息。

#### Envoy 配置

当在网格中全局启用出口时，OSM 控制器对每个 Envoy 代理 sidecar 进行编程，以使用出站侦听器配置上的默认过滤器链匹配外部或未知目的地。默认过滤器链名为 `outbound-egress-filter-chain`，如下面的配置片段所示。任何与出站侦听器上的默认出口过滤器链匹配的流量都通过 `passthrough-outbound` 集群代理到其原始目的地。

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

[1]: /docs/api_reference/policy/v1alpha1/#policy.openservicemesh.io/v1alpha1.EgressSpec

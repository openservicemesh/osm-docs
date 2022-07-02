---
title: "高可用设计注意事项"
description: "开放服务网格高可用设计注意事项"
aliases: "/docs/HA/"
type: docs
---

# 高可用

OSM 的控制平面组件在构建时充分考虑了高可用性和容错能力。以下各节将详细记录如何解决这些问题。

## 高可用和容错

高可用性和容错由 OSM 中的几个设计决策和外部机制实现和保证的，这些将在以下几点中记录：

### 无状态性

OSM 的控制平面组件不拥有或没有任何需要在运行时保存的状态相关数据；受控例外：

- CA/根证书：当运行多个副本时，多个副本需要使用同一个 CA 根证书。对于需要在 OSM 运行前生成/提供根 CA 的[证书管理器](https://github.com/openservicemesh/osm/blob/{{< param osm_branch >}}/DESIGN.md#2-certificate-manager)（Vault、Cert-Manager），root CA 将在所有实例启动时从提供者哪里获取。
  对于其他能够在没有 CA 情况下自动生成 CA 的证书提供者（如 Tresor），在创建过程中保证原子性和同步性，确保所有的实例加载相同的 CA。
- Envoy Bootstrap 证书（代理使用其对控制平面进行认证）：这些证书在注入 webhook 处理过程中创建，并作为代理的引导配置。该配置存储在一个 Kubernetes secret 中，并以卷的形式挂载到注入的 envoy pod 中，以确保在任何时候都具有幂等性。

除了以上两个例外，其他配置从 Kubernetes 中构建和获取。

用于计算流量策略的域状态完全由 `osm-controller` 订阅的相关对象上的不同的运行时提供者（Kubernetes 等）和 Kubernetes client-go 通知器提供。

多个 `osm-controller` 的运行将订阅同一组对象并为服务网格生成相同的配置。由于 Kubernetes client-go 通知器的最终一致性特点，`osm-controller` 保证了策略执行的最终一致性。

<p align="center">
  <img src="/docs/images/ha/ha1.png" width="400" height="350"/>
</p>

### 可重启性

前面的无状态设计考虑应该确保 OSM 控制平面组件都是可重启的。

- 一个重启的实例将重新同步所有的 Kubernetes 域资源。现有的代理将会重连并且（假设网格拓扑或者策略没有发生变化）应重新计算出相同的配置并将其作为新版本推送给代理。

### 水平伸缩

`osm-controller` 和 `osm-injector` 组件允许根据负载或者可用性需求各自进行水平伸缩。

- 当 `osm-controller` 扩展出多副本时，连接的代理可能负载均衡地连接到任意一个 OSM 控制平面实例
- 同样地，`osm-injector` 也可以通过水平伸缩来处理网格中的增加 pod 数量/速率。
- 在 `osm-controller` 中，服务证书（在代理之间用于 TLS 身份验证和相互通信）是短暂的，并且仅在控制平面上的运行时保留（尽管在需要时作为代理 xDS 协议的一部分推送）。

  多个 `osm-controller` 实例可能会为单个服务创建不同但有效的服务证书。这些不同的证书将 (1) 由同一个根签名，因为多个 OSM 实例必须加载相同的根 CA，并且 (2) 将具有相同的通用名称 (CN)，当流量在服务之间进行代理时其被用于匹配和验证。

<p align="center">
  <img src="/docs/images/ha/ha2.png" width="450" height="400"/>
</p>

简而言之，无论代理连接到哪个控制平面，都会将具有正确/适当的 CN 并由共享控制平面根 CA 签名的有效证书推送给它。

- 水平扩容不会将已建立的代理连接重新分配到控制平面，除非它们断开连接。
- 水平缩减将使断开连接的代理连接到未由向下缩放终止的实例。应在重新建立连接时计算并推送新版本的配置。

<p align="center">
  <img src="/docs/images/ha/ha3.png" width="450" height="400"/>
</p>

- 如果控制平面全部挂掉，运行中的代理将进入无头模式<sup>[1]</sup>，直到可以重连到运行的控制平面。

[1] 无头：通常在控制平面/数据平面设计范式中提到，是指当两个组件之间存在依赖关系时，允许依赖代理生存并在被依赖者死亡或变得无法访问时以最新状态继续运行的概念。

### Pod 自动水平伸缩 - HPA

HPA 会根据目标的平均 CPU 使用率（%）和平均内存使用率（%）对控制平面的 pod 进行自动扩缩容。
要启用 HPA，使用下面的命令

```
osm install --set=osm.<control_plane_pod>.autoScale.enable=true
```

> 注意：control_plane_pod 可以是 `osmController` 或者 `injector`

应用于控制平面 Pod deployment 的反亲和性将确保 Pod 跨节点的分布具有更好的弹性。

如果存在多个 pod 副本，它们将被尽可能地调度到不同的节点上。

HPA 的更多参数：

- `minReplicas`（整数）：自动扩缩器可以设置的最小 pod 数（允许值：1-10）
- `maxReplicas`（整数）：自动扩缩器可以设置的最大 pod 数 （允许值：1-10）
- `cpu.targetAverageUtilization`（整数）：CPU 利用率的目标值，百分比（允许值：0-100）
- `memory.targetAverageUtilization`（整数）：内存利用率的目标值，百分比（允许值：0-100）

### Pod 干扰预算 - PDB

为了防止在计划中断期间发生中断，控制平面 pod `osm-controller` 和 `osm-injector` 有一个 PDB，可确保每个控制平面应用始终至少 1 个 pod。

执行如下命令启动 PDB

```
osm install --set=osm.<control_plane_pod>.enablePodDisruptionBudget=true
```

> 注意：control_plane_pod 可以是 `osmController` 或者 `injector`

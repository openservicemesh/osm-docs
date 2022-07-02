---
title: "流量拆分"
description: "使用 SMI 流量拆分 API 的流量拆分"
type: docs
weight: 10
---

# 流量拆分

[SMI 流量拆分 API](https://github.com/servicemeshinterface/smi-spec/blob/main/apis/traffic-split/v1alpha2/traffic-split.md) 可用于将传出流量拆分到多个服务后端。可用于为软件的多个版本编排金丝雀版本。

## 支持哪些

OSM 实现了[SMI 流量拆分 v1alpha2 版本](https://github.com/servicemeshinterface/smi-spec/blob/main/apis/traffic-split/v1alpha2/traffic-split.md)。

它支持以下内容：

- SMI 和宽松流量策略模式下的流量拆分
- HTTP 和 TCP 流量拆分
- 金丝雀或蓝绿部署的流量拆分

## 工作原理

可以使用 SMI 流量拆分 API 将发往 Kubernetes 服务的出站流量拆分到多个服务后端。考虑以下示例，其中与 `default/bookstore` 服务对应的 `bookstore.default.svc.cluster.local` FQDN 的流量被拆分为服务 `default/bookstore-v1` 和 `default/bookstore-v2`，其中权重分别为 90 和 10。

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

要正确配置 `TrafficSplit` 资源，确保满足以下条件很重要：

- `metadata.namespace` 是 [添加到网格的命名空间](/docs/guides/app_onboarding/namespaces/)
- `metadata.namespace`、`spec.service` 和 `spec.backends` 都属于同一个命名空间
- `spec.service` 指定 [Kubernetes 服务的 FQDN](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/#services)
- `spec.service` 和 `spec.backends` 对应 Kubernetes 服务对象
- 所有后端的总权重必须大于零，并且每个后端的权重必须为正数

创建 `TrafficSplit` 资源时，OSM 应用客户端 sidecar上的配置，根据指定的权重将定向到根服务 (`spec.service`) 的流量拆分到后端 (`spec.backends`)。对于 HTTP 流量，请求中的 `Host/Authority` 标头必须与 `TrafficSplit` 资源中指定的根服务的 FQDN 匹配。在上面的示例中，这意味着客户端发起的 HTTP 请求中的 `Host/Authority` 标头必须与 `default/bookstore` 服务的 Kubernetes 服务 FQDN 匹配才能进行流量拆分。

> 注意：OSM 没有为原始 HTTP 请求配置 `Host/Authority` 头重写，因此在 `TrafficSplit` 资源中引用的后端服务必须接受带有原始 HTTP `Host/Authority` 头的请求。

需要注意的是，`TrafficSplit` 资源仅将流量拆分配置到服务，并不授予应用程序相互通信的权限。因此，一个有效的 [TrafficTarget](https://github.com/servicemeshinterface/smi-spec/blob/main/apis/traffic-access/v1alpha3/traffic-access.md#traffictarget) 资源必须与`TrafficSplit` 一起配置以实现应用程序之间的流量。

请参阅有关 [使用 SMI 流量拆分的金丝雀部署](/docs/demos/canary_rollout) 的演示以了解更多信息。
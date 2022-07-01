---
title: "调节器指南"
description: "调节器指南"
aliases: ["/docs/reconciler_guide"]
type: docs
weight: 8
---

# 调节器指南

这篇指南描述了如何在开放式服务网格 (OSM) 里启用调节器。

## 调节器如何工作

在 OSM 里建造一个调节器的目标就是：确保对 OSM 控制平面的正确操作所需要的资源在任何时刻都在其预期的状态。那些在 OSM 安装时作为一部分被安装，有标签 `openservicemesh.io/reconcile: true` 和 `app.kubernetes.io/name: openservicemesh.io` 的资源将被调节器协调。

**注意**：如果标签 `openservicemesh.io/reconcile: true` 和 `app.kubernetes.io/name: openservicemesh.io` 在调节器资源上被修改了或者被删除了，那么调节器将不会按照要求的来操作。

一个在调节器资源上被更新或者删除的事件将触发调节器，从而将调节资源回到它被要求的状态。只有元数据修改 (不含一个名称修改) 将被允许用在调节器资源上。

### 资源调节

OSM 调节的资源是：

- CRD：被 OSM 安装/需要的 CRD [OSM 的 CRD](https://github.com/openservicemesh/osm/tree/{{< param osm_branch >}}/cmd/osm-bootstrap/crds) 将被调节。既然 OSM 管理着它所需要的 CRD 的安装和升级，OSM 也将调节它们以确保它们的规范、存储和服务的版本一直在 OSM 做需要的状态。

- MutatingWebhookConfiguration：一个 MutatingWebhookConfiguration 被部署，以作为 OSM Control Plane 的一部分来开启自动 Sidecar 注入。因为对于 Pod 接入网格来说这是非常关键的组件，OSM 会调节该资源。

- ValidatingWebhookConfiguration：一个 ValidatingWebhookConfiguration 被部署，作为 OSM 控制平面的一部分来验证多种网格配置。这个资源验证被应用到网格的配置，因此 OSM 将调节这个资源。


## 如何安装带调节器的 OSM

要安装带调节器的 OSM，使用下面的命令：

```console
$ osm install --set osm.enableReconciler=true
OSM installed successfully in namespace [osm-system] with mesh name [osm]
```


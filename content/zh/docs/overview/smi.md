---
title: 服务网格接口 (SMI) 支持
description: "OSM 的 SMI 实现"
type: docs
---

## 概览

开放式服务网格（OSM）实现了[服务网格接口（SMI）](https://smi-spec.io/)资源。这样就允许 OSM 用户拥有通用服务网格场景的灵活实现。

## 支持的版本

要找到什么样的 SMI 资源版本在您的网格中被支持，您可以运行 `osm mesh list`。这里是一段摘录，来自于示例输出：

```
MESH NAME   MESH NAMESPACE   SMI SUPPORTED
osm         osm-system       HTTPRouteGroup:v1alpha4,TCPRoute:v1alpha4,TrafficSplit:v1alpha2,TrafficTarget:v1alpha3
```

下面是当前支持的 SMI 资源和它们的版本：

| SMI 资源 | 版本 |
|--------------|---------|
| TrafficTarget | access.smi-spec.io/v1alpha3 |
| TrafficSplit | split.smi-spec.io/v1alpha2 |
| HTTPRouteGroup | specs.smi-spec.io/v1alpha4 |
| TCPRoute | specs.smi-spec.io/v1alpha4 |

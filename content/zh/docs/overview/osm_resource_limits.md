---
title: "OSM 资源需求和限制"
description: "给 OSM Pod 和部署的资源需求与限制"
type: docs
---

| 键名 | 类型 | 默认值 | 描述 |
|-----|------|---------|-------------|
| osm.injector.resource | object | `{"limits":{"cpu":"0.5","memory":"64M"},"requests":{"cpu":"0.3","memory":"64M"}}` | Sidecar 注入器的容器资源参数。|
| osm.osmBootstrap.resource | object | `{"limits":{"cpu":"0.5","memory":"128M"},"requests":{"cpu":"0.3","memory":"128M"}}` | OSM 引导程序的容器资源参数。|
| osm.osmController.resource | object | `{"limits":{"cpu":"1.5","memory":"1G"},"requests":{"cpu":"0.5","memory":"128M"}}` | OSM 控制器的容器资源参数。请参阅 https://docs.openservicemesh.io/docs/guides/ha_scale/scale/ 以获取更多细节。|
| osm.prometheus.resources | object | `{"limits":{"cpu":"1","memory":"2G"},"requests":{"cpu":"0.5","memory":"512M"}}` | Prometheus 的容器资源参数。|

> 注意：这些都是默认值，其可以通过 [values.yaml](https://github.com/openservicemesh/osm/blob/release-v0.11/charts/osm/values.yaml) 来配置。
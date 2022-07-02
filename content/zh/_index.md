---
title: "开放服务网格文档"
description: ""
type: docs
---


一个简单、完整、独立的服务网格。

OSM 运行在 [Kubernetes](https://kubernetes.io/) 上。OSM 控制平面实现了 [Envoy 的 xDS](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/operations/dynamic_configuration)，并使用 [SMI](https://smi-spec.io/) API进行配置。OSM 将一个 Envoy 代理作为 sidecar 容器注入到应用程序的每个实例。

要了解更多关于 OSM 的信息：
* 通过所有的 [入门文章](/docs/getting_started/) 来安装 OSM 并运行一个示例应用程序。
* 阅读 [OSM 概述](/docs/overview/about/) 和更多关于 [组件设计](/docs/overview/osm_components/)。
* 运行 [演示](/docs/demos/) 部分的其他示例方案。

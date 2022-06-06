---
title: "Flagger 与 OSM 集成"
description: "演示如何将 OSM 与 Flagger 集成"
aliases: "/docs/guides/integrations/demo_flagger"
type: docs
weight: 4
---

# Flagger 与 OSM 集成

围绕 OSM 使用[Service Mesh Interface](smi-spec.io) [Traffic Split](https://github.com/servicemeshinterface/smi-spec/blob/v0.6.0/apis/traffic-split/v1alpha4/traffic-split.md)额外添加的自动化功能，OSM 提供了与 [WeaveWorks](https://www.weave.works/) 开发的 [Flagger](https://www.weave.works/oss/flagger/) 项目的集成。

"Flagger 是一个渐进式的交付工具，让 Kubernetes 上的应用的发布过程自动化。它通过在测量指标和运行一致性测试的同时逐渐将流量转移到新版本来降低在生产中引入新版本的风险。" [[1]](#1)

## 使用 Flagger 进行适用于 OSM 的渐进式交付

通过与 Flagger 项目的合作，关于如何使用 OSM 和 Flagger 的文档发布在了 Flagger 网站。请访问 [Open Service Mesh Progressive Delivery Flagger documentation](https://docs.flagger.app/tutorials/osm-progressive-delivery) 获取有关如何安装的详细信息。OSM/Flagger 集成的代码在 [Flagger GitHub repository](https://github.com/fluxcd/flagger)。如果遇到任何与集成相关的问题，请在提交 [Flagger GitHub issues repository](https://github.com/fluxcd/flagger/issues) 提交 issue。

## 参考

<a id="1">[1]</a>
WeaveWorks/Flagger
What is Flagger?

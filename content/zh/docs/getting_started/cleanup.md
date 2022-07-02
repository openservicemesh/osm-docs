---
title: "卸载 OSM"
description: "卸载 OSM 和书店应用"
type: docs
weight: 6
---

# 卸载 OSM

快速入门这个章节里面的文章勾勒了如何安装 OSM 和示例应用。如果要卸载全部与之相关的资源，就需要删除这些示例应用和相关的 SMI 资源，并且卸载掉 OSM control plane 和 cluster-wide 的 OSM 资源。

## 删除示例应用

要清理掉示例应用和相关的 SMI 资源，需要删除它们的命名空间。例如：

```bash
kubectl delete ns bookbuyer bookthief bookstore bookwarehouse
```

## 卸载 OSM control plane

要卸载 OSM control plane，请使用 `osm unistall mesh`。

```bash
osm uninstall mesh
```

## 卸载 OSM cluster-wide 资源

要卸载 OSM cluster-wide 资源，请使用 `osm uninstall mesh --delete-cluster-wide-resources`。

```bash
osm uninstall mesh --delete-cluster-wide-resources
```

关于卸载 OSM 的更多细节，请参阅 [卸载指南](/docs/guides/uninstall/)

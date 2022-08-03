---
title: "添加命名空间"
description: "本节介绍 OSM 如何以及为何监控 Kubernetes 命名空间"
aliases: "/docs/tasks/namespace_monitoring"
type: docs
weight: 2
---

# 添加命名空间

## 概述

在设置 OSM 控制平面（也称为“网格”）时，还可以将一组 Kubernetes 命名空间注册到网格中。将命名空间注册到 OSM 允许 OSM 监视该命名空间内的资源，无论它们是部署在 Pod、服务中的应用程序，还是表示为 SMI 资源的流量策略。

同一个命名空间只能被一个网格监控，所以当同一个 Kubernetes 集群中有多个 OSM 实例时，需要注意这一点。 将策略应用于应用时，OSM 将仅评估每个受监控命名空间中的资源，因此将应用部署到的命名空间注册到具有正确网格名称的正确 OSM 实例非常重要。注册命名空间还可以选择允许为给定命名空间中的资源收集指标，并允许命名空间中的 Pod 自动注入 sidecar 代理容器。这些都是帮助 OSM 提供流量管理和可观察性功能的特性。在命名空间级别限定此功能允许团队组织集群的哪些部分应该属于哪个网格。

通过为 Kubernetes 命名空间添加特定标签和注解来控制命名空间监控、sidecar 自动注入以及指标收集。可以手动也可以使用 `osm` 命令行工具完成，不过推荐使用 `osm` 命令行工具。有了 `openservicemesh.io/monitored-by=<mesh-name>` 标签允许有着 `mesh-name` 指定名字的 OSM 控制平面监控命名空间中的所有资源。`openservicemesh.io/sidecar-injection=enabled` 注解开启 OSM 自动为命名空间中的所有 pod 注入 sidecar 代理容器。指标注解 `openservicemesh.io/metrics=enabled` 允许 OSM 收集该命名空间中资源的指标。

请参阅下面如何使用 OSM CLI 管理命名空间监视。

## 添加命名空间到 OSM 控制平面

使用下面的命令将需要网格监控和 sidecar 注入的命名空间加入网格：

```bash
osm namespace add <namespace>
```

将命名空间加入到网格中会开启 sidecar 的自动注入。如果在添加时想明确地禁用 sidecar 注入，使用[这里](/docs/guides/app_onboarding/sidecar_injection/#explicitly-disabling-automatic-sidecar-injection-on-namespaces)提到的 `--disable-sidecar-injection` 选项。

如果命名空间在应用 deployment 已经创建后被加入到网格中，需要重启已有的 deployment 使 OSM 在 pod 重建时自动注入 sidecar。可以通过下面的命令来重启 deployment 管理的 pod：

```bash
kubectl rollout restart deployments -n <namespace>
```

## 从 OSM 控制平面移除命名空间

使用如下命令将正被网格监控的命名空间从网格中移除并禁用 sidecar 注入：

```bash
osm namespace remove <namespace>
```

这个命令通过移除命名空间上所有 OSM 相关的标签和注解来将其从网格中移除。

## 为命名空间启用指标

```bash
osm metrics enable --namespace <namespace>
```

## 忽略命名空间

集群中的命名空间可能永远无需加入网格。明确地将命名空间从 OSM 排除：

```bash
osm namespace ignore <namespace>
```

## 列出 OSM 管理的命名空间

列出指定 OSM 管理的命名空间

```bash
osm namespace list --mesh-name=<mesh-name>
```

## 排障指南

### 策略相关问题

如果应用于命名空间资源的 SMI 策略不起作用，先确认命名空间已经注册到正确的网格。

```bash
osm namespace list --mesh-name=<mesh-name>

NAMESPACE         MESH   SIDECAR-INJECTION
<namespace>       osm    enabled
```

如果命名空间未列出，通过 `kubectl` 命令检查命名空间的标签：

```bash
kubectl get namespace <namespace> --show-labels

NAME          STATUS   AGE   LABELS
<namespace>   Active   36s   openservicemesh.io/monitored-by=<mesh-name>
```

如果标签的值不是期望的 `mesh-name`，将命名空间移除并重新加入到正确的 `mesh-name` 中。

```bash
osm namespace remove <namespace> --mesh-name=<current-mesh-name>
osm namespace add <namespace> --mesh-name=<expected-mesh-name>
```

如果没有找到 monitored-by 标签，可能没有注册到网格，或者注册的时候出错。
使用 `osm` 或者 `kubectl` 命令行工具将命名空间注册到网格：

```bash
osm namespace add <namespace> --mesh-name=<mesh-name>
```

```bash
kubectl label namespace <namespace> openservicemesh.io/monitored-by=<mesh-name>
```

### sidecar 自动注入相关问题

如果看到 Pod 没有自动注入 sidecar 容器，确保已经启用 sidecar 注入：

```bash
osm namespace list --mesh-name=<mesh-name>

NAMESPACE         MESH   SIDECAR-INJECTION
<namespace>       osm    enabled
```

如果命名空间未列出，使用 `kubectl` 检查命名空间的注解：

```bash
kubectl get namespace <namespace> -o=jsonpath='{.metadata.annotations.openservicemesh\.io\/sidecar-injection}'
```

如果输出中没有 `enabled` 字样，使用 `osm` 命令行工具将命名空间加入，或者使用 `kubectl` 为其添加注解：

```bash
osm namespace add <namespace> --mesh-name=<mesh-name> --disable-sidecar-injection=false
```

```bash
kubectl annotate namespace <namespace> openservicemesh.io/sidecar-injection=enabled --overwrite
```

### 指标收集相关问题

如果没有看到指定命名空间中资源的指标，确保已经启用指标：

```bash
kubectl get namespace <namespace> -o=jsonpath='{.metadata.annotations.openservicemesh\.io\/metrics}'
```

如果结果没有 `enabled` 字样，使用 `osm` 命令行工具为命名空间启用，或者使用 `kubectl` 添加注解：

```bash
osm metrics enable --namespace <namespace>
```

```bash
kubectl annotate namespace <namespace> openservicemesh.io/metrics=enabled --overwrite
```

### 其他问题

如果遇到上面的调试方法无法解决的问题，请到 GitHub 仓库创建一个 issue。

---
title: "Sidecar 注入"
description: "本章节描述 OSM 中 sidecar 的注入流程。"
type: docs
weight: 3
---

# Sidecar 注入
服务网格中的服务通过安装在 pod 中的 sidecar 进行通信。下面的章节描述了 OSM 中的 sidecar 注入流程。

## Sidecar 自动注入
目前网格中注入sidecar 的唯一方式就是自动注入。由 OSM 提供的修改性质的 web 准入控制器将 sidecar 自动注入到应用程序 Kubernetes Pod 中。

Sidecar 的自动注入可以被以每个命名空间来配置作为注册命名空间到网格的一部分，也可以随后通过 Kubernetes API 完成。通过使用 sidecar 注入注解对命名空间或 pod 资源进行注释，可以在每个命名空间或每个 pod 上启用自动 sidecar 注入。单个 pod 和命名空间可以显式配置为启用或禁用自动 sidecar 注入，从而使用户可以灵活地控制 pod 和命名空间上的 sidecar 注入。

### 启动 sidecar 自动注入

先决条件：
- Pod 所在的命名空间必须是通过 `osm namespace add` 命令注册到网格的。
- Pod 所在的命名空间不能使用 `osm namespace ignore` 命令排除。
- Pod 所在的命名空间不能使用 key 为 `name`，值为 OSM 控制平面所在命名空间名的标签。比如，某个命名空间有标签 `name: osm-system`，而 `osm-system` 是控制平面所在的命名空间名，这个命名空间中的 pod 无法注入 sidecar。
- Pod 规格上不能使用 `hostNetwork: true`。如果有，会导致主机网络的路由失败。

Sidecar 注入可以通过如下方式启用：

- 使用 `osm` 命令行工具 `osm namespace add <namespace>` 将命名空间注册到网格时，该命令会默认启用 sidecar 的自动注入。

- 使用 `kubectl` 来注释单个命名空间或者 pod 来启用 siecar 注入：

  ```console
  # Enable sidecar injection on a namespace
  $ kubectl annotate namespace <namespace> openservicemesh.io/sidecar-injection=enabled
  ```

  ```console
  # Enable sidecar injection on a pod
  $ kubectl annotate pod <pod> openservicemesh.io/sidecar-injection=enabled
  ```

- 在命名空间或者 pod 的资源规格中将 sidecar 注入注解的值设置为 `enabled`：
  ```yaml
  metadata:
    name: test
    annotations:
      'openservicemesh.io/sidecar-injection': 'enabled'
  ```

  只有符合如下条件时，pod 才会被注入了 sidecar：
  1. Pod 所在的命名空间被网格监控。
  2. Pod 被显式地启用 sidecar 注入，或者 pod 所在的命名空间启动了 sidecar 注入同时 pod 没有显式地禁用 sidecar 注入。

### 显式地禁用命名空间的 sidecar 注入

通过下面的方式可以禁用命名空间的自动 sidecar 注入：

- 当使用 `osm` 命令行工具 `osm namespace add <namespace> --disable-sidecar-injection` 将命名空间注册到网格时：
  如果命名空间之前已经启用 sidecar 注入，运行此命令后会禁用。

- 使用 `kubectl` 注释单个命名空间来禁用 sidecar 注入：

  ```console
  # Disable sidecar injection on a namespace
  $ kubectl annotate namespace <namespace> openservicemesh.io/sidecar-injection=disabled
  ```

### 显式地禁用 pod 的 sidecar 注入

单个 pod 可以被显式地禁用 sidecar 注入。这个方法在命名空间启用 sidecar 注入但是指定的 pod 需要禁用的时候非常有用。

- 使用 `kubectl` 注释单个 pod 来禁用 sidecar 注入：
  ```console
  # Disable sidecar injection on a pod
  $ kubectl annotate pod <pod> openservicemesh.io/sidecar-injection=disabled
  ```

- 将 pod 规格中的 sidecar 注入注解设置为 `disabled`：
  ```yaml
  metadata:
    name: test
    annotations:
      'openservicemesh.io/sidecar-injection': 'disabled'
  ```

使用 `osm namespace remove` 命令将命名空间从网格中移除时，会隐式地禁用命名空间的 sidecar 自动注入。

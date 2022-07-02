---
title: "先决条件"
description: "应用程序要求"
type: docs
aliases: ["/docs/application_requirements"]
weight: 1
---

# 先决条件

## 安全上下文

* 不要使用 user ID（UID）**1500** 运行应用。这是**保留的**被用于 OSM sidecar 注入器注入的 Envoy 代理 sidecar 容器。
* 如果在 pod 层面安全上下文 `runAsNonRoot` 被设置为 `true`，必须为 pod 或者 pod 内的各个容器提供 `runAsUser` 值。例如：
  ```yaml
    securityContext:
      runAsNonRoot: true
      runAsUser: 1200
  ```
    如果省略 UID，应用容器默认地会尝试使用 root 用户运行，会与 pod 的安全上下文冲突。
* 不需要其他功能。

> 注意：OSM 的初始化容器需要使用 root 运行并添加 `NET_ADMIN` 特性，因为其需要这些安全上下文来完成调用。应用程序安全上下文不会修改这些值。

## 端口

不要使用下面这些被 Envoy sidecar 使用的端口。

| 端口  | 描述                            |
| ----- | -------------------------------------- |
| 15000 | Envoy 管理端口                     |
| 15001 | Envoy 出站监听端口          |
| 15003 | Envoy 入站监听端口            |
| 15010 | Envoy Prometheus 入站监听端口 |

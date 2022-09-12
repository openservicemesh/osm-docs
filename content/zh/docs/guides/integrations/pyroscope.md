---
title: "OSM 与 Pyroscope 集成"
description: "A simple demo showing how OSM integrates with Pyroscope for continuous profiling"
aliases: "/docs/integrations/demo_pyroscope"
type: docs
weight: 5
---

# OSM 与 Pyroscope 集成

## Pyroscope 是什么

Pyroscope 是一个持续的分析工具，它可以分析用各种语言编写的程序的性能，包括 GoLang。它提供了一个很棒的 Web UI 来读取分析结果。

当 GoLang 程序公开内置的 `net/http/pprof` 库提供的 Web API 端点时，Pyroscope 可以直接从给定端点提取分析结果。启用调试服务器功能时，OSM 会公开所需的端点。这种方法允许在不更改 OSM 代码库的情况下集成 Pyroscope。

## 启用 OSM 调试服务器

要在安装时启用 OSM 调试服务器，只需在 `osm install` 命令中添加 `--set=osm.enableDebugServer=true` 参数，或在 OSM 安装后执行命令 `kubectl patch meshconfig osm-mesh-config -n "$OSM_NAMESPACE" -p '{"spec":{"observability":{"enableDebugServer":true}}}' --type=merge`。

## 安装 Pyroscope

在本节中，我们将安装 Pyroscope 来分析 OSM 控制平面。目前只有 *osm-controller* 公开 pprof Web 端点。

第一步是向 `osm-controller` 服务添加注解，以便 Pyroscope 对其进行监控。

```bash
kubectl annotate -n "$OSM_NAMESPACE" svc osm-controller  \
    "pyroscope.io/scrape=true" \
    "pyroscope.io/application-name=osm-controller" \
    "pyroscope.io/spy-name=gospy" \
    "pyroscope.io/profile-cpu-enabled=true" \
    "pyroscope.io/profile-mem-enabled=true" \
    "pyroscope.io/port=9092"
```

将 `$OSM_NAMESPACE` 替换为 Kubernetes 集群上的 OSM 命名空间。此命令告知 Pyroscope 分析服务的 CPU 和内存使用情况。pprof Web 端点公开在端口 9092 上。

接下来我们可以使用 Helm 安装 Pyroscope。

```bash
helm install prof pyroscope \
    -f https://raw.githubusercontent.com/openservicemesh/osm-docs/main/manifests/integrations/pyroscope-values.yaml \
    --repo https://pyroscope-io.github.io/helm-chart \
    -n "$OSM_NAMESPACE"
```

此命令将在 OSM_NAMESPACE 中安装名为 `prof-pyroscope` 的服务。Pyroscope 的一些配置可以在 [pyroscope-values.yaml](https://raw.githubusercontent.com/openservicemesh/osm-docs/main/manifests/integrations/pyroscope-values.yaml) 文件中找到。如果想自定义 Pyroscope 安装，可以在本地机器上复制它并编辑值。

通过运行检查安装：

```bash
kubectl get svc prof-pyroscope -n "$OSM_NAMESPACE"
```

默认情况下，Pyroscope 服务公开端口 4040。可以在将端口转发到 localhost 后访问 Web UI：

```bash
kubectl port-forward service/prof-pyroscope 4040 -n "$OSM_NAMESPACE"
```

此命令将保持转发端口打开。在 Web 浏览器中打开 [http://localhost:4040](http://localhost:4040) 来使用 Pyroscope。应该能够从 Application 下拉列表中看到 `osm-controller`，如下所示：

<p align="center">
  <img src="/docs/images/pyroscope-install.png" />
</p>

## 卸载 Pyroscope

运行如下命令卸载 Pyroscope：

```bash
helm uninstall prof -n "$OSM_NAMESPACE"
```

## 参考

* <a href="https://pyroscope.io/">Pyroscope 项目</a>
* <a href="https://pyroscope.io/docs/golang-pull-mode/">Pyroscope GoLang 拉模式</a>


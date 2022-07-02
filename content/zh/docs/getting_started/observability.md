---
title: "用 Prometheus 和 Grafana 配置可观测性"
description: "使用 OSM 带 Prometheus 和 Grafana 的可观测性集成来检查书店应用间的流量"
type: docs
weight: 5
---

# 用 Prometheus 和 Grafana 来配置可观测性

接下来的文章向您展示了如何安装自带 Prometheus 和 Grafana 栈的 OSM，从而具备可观测性和监视能力。对于使用在您的集群上自有的 Prometheus 和 Grafana 栈协同 OSM 的例子，请参阅[集成 OSM 到 Prometheus 和 Grafana](https://docs.openservicemesh.io/docs/demos/prometheus_grafana/)示例。

在这篇文章中所创建的配置不应该被用于产品环境。对于产品级的部署，请参阅[Prometheus 运维](https://github.com/prometheus-operator/prometheus-operator/blob/master/Documentation/user-guides/getting-started.md)和[在 Kubernetes 中部署 Grafana](https://grafana.com/docs/grafana/latest/installation/kubernetes/)。


## 安装带 Prometheus 和 Grafana 的 OSM

在 `osm install` 上，一个 Prometheus 和/或 Grafana 实例可以通过默认的 OSM 配置来自动提供。
```bash
 osm install --set=osm.deployPrometheus=true \
             --set=osm.deployGrafana=true
```
更多可观测性信息在[可观测性指南](/docs/guides/observability)。

## Prometheus

当配置时带了 `--set=osm.deployPrometheus=true` 标记，OSM 安装将部署一个 Prometheus 实例来抓取 sidecar 和 OSM control plane 指标端点。[抓取配置文件](https://github.com/openservicemesh/osm/blob/{{< param osm_branch >}}/charts/osm/templates/prometheus-configmap.yaml)定义了默认的 Prometheus 行为和被 OSM 采集的指标集。

## Grafana

在 `osm install` 上，OSM 能够被配置为通过使用 `--set=osm.deployGrafana=true` 标记来部署一个 [Grafana](https://grafana.com/grafana/) 实例。OSM 提供预配置的仪表板，这些在可观测性指南的[OSM Grafana 仪表板](/docs/guides/observability/metrics/#osm-grafana-仪表板)章节有描述。

## 启动指标抓取

通过使用 `osm metrics` 命令，在命名空间范围内启动指标。默认的，OSM **不会**为在网格中的 Pod 配置指标抓取。

```bash
osm metrics enable --namespace test
osm metrics enable --namespace "test1, test2"

```
> 注意：您正在为指标抓取所启用的命名空间必须已经是网格的一部分。

## 检查仪表板

OSM Grafana 仪表板能够通过如下命令来查看：

```bash
osm dashboard
```

> 注意：如果您依旧有额外的终端在运行着 `./scripts/port-forward-all.sh` 脚本，请前往那里然后用 `CTRL+C` 终止端口转发。`osm dashboard` 端口重定向将不能够与运行着的端口转发同时工作。

导航到 http://localhost:3000 来访问 Grafana 仪表板。默认的用户名是 `admin`，默认的密码是 `admin`。在 Grafana 主页上点击 **Home** 图标，您将看到一个文件夹，里面包含了 OSM Control Plane 和 OSM Data Plane 的仪表板。

## 下一步

[清除示例应用并卸载 OSM](/docs/getting_started/cleanup/).
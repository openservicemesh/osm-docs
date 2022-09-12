---
title: "安装 OSM"
description: "使用 OSM 命令行工具安装 OSM 控制平面"
type: docs
weight: 1
---

# 安装 OSM

## 先决条件

这个 OSM {{< param osm_version >}} 的演示需要：
  - 运行 Kubernetes v1.19 或更高版本的集群（使用选择的云提供商、[minikube](https://minikube.sigs.k8s.io/docs/start/) 或类似的）
  - 能够执行 [Bash](https://en.wikipedia.org/wiki/Bash_(Unix_shell)) 脚本的工作站
  - [Kubernetes 命令行工具](https://kubernetes.io/docs/tasks/tools/#kubectl) - `kubectl`
  - [OSM 代码仓库](https://github.com/openservicemesh/osm/) 在本地可用
  
> 注意：本文档假设你已经在 `~/.kube/config` 中安装了 Kubernetes 集群的凭据，并且 `kubectl cluster-info` 成功执行。

## 下载并安装 OSM 命令行工具

`osm` 命令行工具包含安装和配置 Open Service Mesh 所需的一切。
该二进制文件可在 [OSM GitHub 发布页面](https://github.com/openservicemesh/osm/releases/) 上找到。

### GNU/Linux 和 macOS

下载 OSM {{< param osm_version >}} 的 64 位 GNU/Linux 或 macOS 二进制文件：

```bash
system=$(uname -s | tr '[:upper:]' '[:lower:]')
release={{< param osm_version >}}
curl -L https://github.com/openservicemesh/osm/releases/download/${release}/osm-${release}-${system}-amd64.tar.gz | tar -vxzf -
./${system}-amd64/osm version
```

### Windows

通过 Powershell 下载 64 位 Windows OSM {{< param osm_version >}} 二进制文件：

```powershell
wget  https://github.com/openservicemesh/osm/releases/download/{{< param osm_version >}}/osm-{{< param osm_version >}}-windows-amd64.zip -o osm.zip
unzip osm.zip
.\windows-amd64\osm.exe version
```

`osm` CLI 可以根据 [本指南](/docs/guides/cli) 从源代码编译。

## 在 Kubernetes 上安装 OSM

下载并解压 `osm` 二进制文件后，我们就可以在 Kubernetes 集群上安装 OSM：

下面的命令显示了如何在 Kubernetes 集群上安装 OSM。

此命令启用 [Prometheus](https://github.com/prometheus/prometheus)、[Grafana](https://github.com/grafana/grafana) 和 [Jaeger](https://github.com/jaegertracing/jaeger) 集成。

`values.yaml` 文件中的 `osm.enablePermissiveTrafficPolicy` chart 参数指示 OSM 忽略任何策略并让流量在 pod 之间自由流动。启用宽松流量策略模式后，新的 Pod 将注入 Envoy，但流量将流经代理，不会被访问控制策略阻止。

> 注意：宽松流量策略模式是 brownfield 部署的一项重要功能，其中可能需要一些时间来制定 SMI 策略。在运营商设计 SMI 策略的同时，现有服务将继续按照安装 OSM 之前的方式运行。

```bash
export osm_namespace=osm-system # Replace osm-system with the namespace where OSM will be installed
export osm_mesh_name=osm # Replace osm with the desired OSM mesh name

osm install \
    --mesh-name "$osm_mesh_name" \
    --osm-namespace "$osm_namespace" \
    --set=osm.enablePermissiveTrafficPolicy=true \
    --set=osm.deployPrometheus=true \
    --set=osm.deployGrafana=true \
    --set=osm.deployJaeger=true
```

在 [可观察性文档](/docs/guides/observability/) 中阅读有关 OSM 与 Prometheus、Grafana 和 Jaeger 集成的更多信息。

## 下一步

现在 OSM 控制平面已启动并运行，[添加应用程序](/docs/getting_started/install_apps/) 到网格。
---
title: "卸载 OSM 控制平面和组件"
description: "卸载"
type: docs
weight: 4
---

# 卸载 OSM 控制平面和组件

这个指南描述了如何从一个 Kubernetes 集群卸载开放式服务网格 (OSM)。指南假设了有一个单一的 OSM 控制平面(网格) 在运行。如果在一个集群上有个多个网格，在这篇指南末尾，卸载任何集群其他资源之前，重复这里所描述的对集群上每一个控制平面的操作。顾及到控制平面和数据平面，这篇指南目标是使用最少的停机时间来卸载掉 OSM 的全部残余资源。

## 先决条件

- 带 OSM 的 Kubernetes 集群已安装
- `kubectl` CLI
- [`osm` CLI](/docs/install/#set-up-the-osm-cli) 或者 Helm 3 CLI

## 从应用 Pod 和 Envoy Secret 上移除 Envoy Sidecar

要卸载 OSM 的第一步是从应用 Pod 上移除 Envoy sidecar 容器。Sidecar 容器强制使用流量策略。没有了它们，流量将依照默认的 Kubernetes 网络来流入和流出 Pod，除非有使用 [Kubernetes 网络策略](https://kubernetes.io/docs/concepts/services-networking/network-policies/)。

OSM Envoy sidecar 和相关的 secret 将按照如下步骤被移除：

1. [禁用自动 Sidecar 注入](#禁用自动-sidecar-注入)
2. [重启 Pod](#重启-pod)

### 禁用自动 Sidecar 注入

OSM 自动 Sidecar 注入是最常见的启用的功能，它通过 `osm` CLI 添加命名空间到网格来启用。使用 `osm` CLI 来查看哪个命名空间已经启用了 Sidecar 注入。如果有多个 Control Plane 被安装，请确保指定了 `--mesh-name` 标志。

查看网格里的命名空间：

```console
$ osm namespace list --mesh-name=<mesh-name>
NAMESPACE          MESH           SIDECAR-INJECTION
<namespace1>       <mesh-name>    enabled
<namespace2>       <mesh-name>    enabled
```

从网格里面移除每一个命名空间：

```console
$ osm namespace remove <namespace> --mesh-name=<mesh-name>
Namespace [<namespace>] successfully removed from mesh [<mesh-name>]
```

这将从命名空间移除 `openservicemesh.io/sidecar-injection: enabled` 标注和 `openservicemesh.io/monitored-by: <mesh name>` 标签。

可选地，如果 Sidecar 注入是通过 Pod 上的标注而不是逐个命名空间被启用的，请修改 Pod 或者部署规范来移除 Sidecar 注入标注。

### 重启 Pod

重启全部带 Sidecar 运行的 Pod：

```console
# If pods are running as part of a Kubernetes deployment
# Can use this strategy for daemonset as well
$ kubectl rollout restart deployment <deployment-name> -n <namespace>

# If pod is running standalone (not part of a deployment or replica set)
$ kubectl delete pod <pod-name> -n namespace
$ k apply -f <pod-spec> # if pod is not restarted as part of replicaset
```

现在，应该没有 OSM Envoy Sidecar 容器作为应用的一部分，也是网格的一部分来运行了。流量不再被如上使用带 `mesh-name` 的 OSM Control Plane 所管理。在这个过程中，您的应用可能会经历一些停机，这是所有的 Pod 在重启。

## 卸载 OSM 控制平面和移除用户提供的资源

在如下步骤中，OSM 控制平面和相关的组件将被卸载。

1. [卸载 OSM 控制平面](#卸载-OSM-控制平面)
2. [移除用户提供的资源](#移除用户提供的资源)
3. [删除 OSM 命名空间](#删除-OSM-命名空间)
4. [OSM 集群广泛资源的移除](#OSM-集群广泛资源的移除)

### 卸载 OSM 控制平面

使用 `osm` CLI 从一个 Kubernetes 集群里卸载 OSM 控制平面。接下来的步骤将移除：

1. OSM 控制器资源（部署、服务、网格配置和 RBAC）
2. 被 OSM 安装的 Prometheus, Grafana, Jaeger 和 Fluent Bit 资源
3. 改变 webhook 和验证 webhook
4. 被 OSM 打到 被 OSM 安装或者需要的 CRD [给 OSM 的 CRD](https://github.com/openservicemesh/osm/tree/{{< param osm_branch >}}/cmd/osm-bootstrap/crds) 上的转换 webhook 域补丁将被拆除。要删除集群广泛资源请参阅[集群广泛资源的移除](#osm-集群广泛资源的移除)来了解更多细节。

运行 `osm uninstall mesh`：

```console
# Uninstall osm control plane components
$ osm uninstall mesh --mesh-name=<mesh-name>
Uninstall OSM [mesh name: <mesh-name>] ? [y/n]: y
OSM [mesh name: <mesh-name>] uninstalled
```

运行 `osm uninstall mesh --help` 以了解更多选项。

可选择的，如果您使用了 Helm 安装的 Control Plane，运行下面的 `helm uninstall` 命令：

```console
$ helm uninstall <mesh name> --namespace <osm namespace>
```

运行 `helm uninstall --help` 以了解更多选项。

### 移除用户提供的资源

如果任何资源在安装时为 OSM 做提供或者创建，那么它们能够在此处被删除。

例如，如果 [Hashicorp Vault](/docs/guides/certificates/#installing-hashi-vault) 为了对 OSM 管理证书的单独目的而被部署，那么全部相关的资源都能被删除。

### 删除 OSM 命名空间

当安装一个网格，`osm` CLI 创建了命名空间（如果其不存在），控制平面被安装其中。然而，当卸载同一个网格时，它所居住的命名空间并不会被 `osm` CLI 自动卸载。这种行为之所以会发生是因为被一个用户在命名空间里面创建的资源或许不想被自动删除。

如果命名空间只是为 OSM 所用，并且没有谁想保留它，该命名空间在卸载时可以被删除，也可以用如下的命令来删除。

```console
$ osm uninstall mesh --delete-namespace
```

> 警告：如果在命名空间里的资源不再需要，只删除该命名空间。例如，如果 OSM 在 `kube-system` 里被安装，删除了命名空间或许会删除重要的集群资源并且会有未知的结果。


### OSM 集群范围资源的移除

在安装上，OSM 确保[这里](https://github.com/openservicemesh/osm/tree/{{< param osm_branch >}}/cmd/osm-bootstrap/crds)提到的所有 CRD 在安装时都存在于集群上。安装过程中，如果它们没有被安装，在余下的控制平面组件运行之前，`osm-bootstrap` Pod 将安装它们。当使用 Helm chart 来安装 OSM 时行为是相同的。

在非托管和托管的环境下卸载网格：
1. 移除 OSM 控制平面组件，包括控制平面 Pod。
2. 从所有的 CRD (OSM 添加的以支持多个 CR 版本) 中移除/拆除转换 webhook 域。

在卸载 OSM 之后，对集群而言，遗留某些 OSM 资源以阻止未知的结果。那些被遗留的资源将取决于 OSM 是否从一个托管还是未托管集群环境来卸载。

当卸载 OSM，全部的 `osm uninstall mesh` 命令和 Helm 卸载不会删除在任何集群环境上任何的 OSM 或者 SMI CRD，这主要出于两个原因：
1. CRD 是集群广泛资源，可能被其他的服务网格或者运行于同一集群的其他资源所使用
2. 一个 CRD 的删除将造成相对于这个 CRD 的全部定制资源也被删除

要移除 OSM 安装的集群广泛资源（例如 MeshConfig，Secret，OSM CRD 和 webhook 配置），接下来的命令能够在 OSM 卸载时或者之后被运行。

```bash
osm uninstall mesh --cluster-wide-resources
```

> 警告：一个 CRD 的删除将造成和该 CRD 对应的全部定制资源也被删除。

对于 OSM 卸载的问题解决，请参阅[卸载问题解决章节](/docs/guides/troubleshooting/uninstall/)

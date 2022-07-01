---
title: "升级 OSM 控制平面"
description: "升级指南"
aliases: ["/docs/upgrade_guide","/docs/troubleshooting/cli/mesh_upgrade"]
type: docs
weight: 3
---

# 升级指南

这个指南描述了如何来升级开放式服务网格 (OSM) 控制平面。

## 升级如何工作

OSM 的 Control Plane 生命周期被 Helm 来管理，并且通过 [Helm 升级功能](https://helm.sh/docs/intro/using_helm/#helm-upgrade-and-helm-rollback-upgrading-a-release-and-recovering-on-failure)能够被升级，该功能将基于修改的值和资源模板，按照需要，来给 Control Plane 组件打补丁或者做替换。

### 升级期间的资源可用性
既然升级可能会包含重新部署新版本的 OSM 控制器，那么这就会造成控制器的停机。当 OSM 控制器不可用时，处理新的 SMI 资源就会有延时，创建新的 Pod 用来注入代理 sidecar 容器将会失败，并且 mTLS 证书将不能被轮换。

已经存在的 SMI 资源将不被影响，这就意味着 Data Plane (包含 Envoy sidecar 配置)也将不受升级的影响。

如果升级包括了 CRD 修改，那么 Data Plane 的暂停是被预期的。精简的 Data Plane 升级在问题 [#512](https://github.com/openservicemesh/osm/issues/512) 里面被追踪。

## 策略

只有确认的升级路径被测试并被支持。

**注意**：这些计划是暂时的，很容易被改变。

破坏性修改，在本章节中，指的是对如下面向用户的组件的不兼容修改：
- `osm` CLI 命令，标志和行为
- SMI CRD 和控制器

This implies the following are NOT user-facing and incompatible changes are NOT considered "breaking" as long as the incompatibility is handled by user-facing components:
这就暗示了下面的不算面向用户和不兼容修改，只要不兼容性被面向用户组件所处理，这就不被算作“破坏性”。
- 图表 values.yaml
- `osm-mesh-config` MeshConfig
- 内部使用的标签和标注 (被谁监视，注入，矩阵等等)

升级只能在那些不包含破坏性修改的版本间被支持，如下所述：

对于 OSM 版本 `0.y.z`：
- 在 `0.y.z` 和 `0.y.z+1` 之间，破坏性修改将不被引入
- 在 `0.y.z` 和 `0.y+1.0` 之间破坏性修改可能被引入

对于 OSM 版本 `x.y.z`，这里 `x >= 1` 的：
- 在 `x.y.z` 和 `x.y+1.0` 之间或者 `x.y.z` 和 `x.y.z+1` 之间的，破坏性改变将不被引入
- 在 `x.y.z` 和 `x+1.0.0` 之间的，破坏性改变将被引入

## 如何升级 OSM

升级一个网格，被推荐的方式是利用 `osm` CLI。对于高级用例，`helm` 或许被使用。

### CRD 升级
因为 Helm 不管理在初始化安装之外的 CRD，OSM 利用一个在 `osm-bootstrap` Pod 上的初始容器来在升级期间更新现存的和添加新的 CRD。如果一个新的发布包含对现存 CRD 的更新或者添加新的 CRD，在 `osm-bootstrap` Pod 上的 `init-osm-bootstrap` 将更新 CRD。相关联的定制资源将被保留，在升级之前和之后即刻都不需要额外的行动。 

请检查[发布事项](https://github.com/openservicemesh/osm/releases)里面的 `CRD Updates`，来查阅被 OSM 使用的 CRD 有任何的更新。如果定制资源的版本在被更新的 CRD 支持版本里面，不需要即刻的行动。OSM 为它的 CRD 实现了一个转换网络钩子 (webhook)，确保对更老版本的支持，并提供在以后的一个时间节点更新定制资源的灵活性。

### 用 OSM CLI 来更新

**先决条件**

- Kubernetes 集群并已安装 OSM控制平面
    - 确保 Kubernetes 集群有被新的 OSM chart所需要的最小 Kubernetes 版本。这点能够在[安装先决条件](/docs/getting_started/install#Pre-requisites)里被找到。
- `osm` CLI 被安装
  - 默认的，`osm` CLI 将升级到同它安装相同的图表版本。例如，v0.9.2 的 `osm` CLI 将升级 OSM Helm chart 到 v0.9.2。升级到任何其他版本的 Helm 图表，而不是匹配于 CLI 的，或许可以工作，但是这些场景没有被测试过，而且即便被报告了问题，也不会被修复。

`osm mesh upgrade` 命令对一个网格的现存 Helm 发布版做 `helm upgrade`。

基础用法不需要额外的参数或者标志：
```console
$ osm mesh upgrade
OSM successfully upgraded mesh osm
```

这个命令将升级网格，其在默认的 OSM 命名空间里面使用默认的网格名称。来自先前发布版的值默认情况下不会被保留到这个新的发布版，但是可以通过在 `osm mesh upgrade` 上独立地使用 `--set` 标志来传递旧值。

查阅 `osm mesh upgrade --help` 来获取更多细节。

### 用 Helm 来升级

#### 先决条件

- 带 OSM Control Plane 的 Kubernetes 集群已经安装
- [helm 3 CLI](https://helm.sh/docs/intro/install/)

#### OSM 配置
当升级时，任何被用来安装或者运行 OSM 的自定义设置都可能被恢复到默认值，这只包含任何矩阵部署。请确保您仔细地遵守了指南来阻止这些值被覆盖。

要阻止任何对您已经做出的对 OSM 配置的修改，使用 `helm --values` 标志。创建一个[值文件](https://github.com/openservicemesh/osm/blob/{{< param osm_branch >}}/charts/osm/values.yaml)的副本（确认使用了针对升级的图表的版本），改变任何您希望定制的值。您可以省去所有其他的值。

**注意**：任何进入到 MeshConfig 的配置修改在升级期间将不会被应用，这个值将保留升级之前的。如果您希望更新在 MeshConfig 里面的任何值，您可以通过在升级之后给资源打补丁来完成。

例如，如果 MeshConfig 中的 `logLevel` 域在升级前被设置为 `info`，在升级期间在 `override.yaml` 里更新这个值将不会造成任何改变。

<b>警告:</b> 不要修改 `osm.meshName` 或者 `osm.osmNamespace`

#### Helm 升级
然后运行下面的 `helm upgrade` 命令。
```console
$ helm upgrade <mesh name> osm --repo https://openservicemesh.github.io/osm --version <chart version> --namespace <osm namespace> --values override.yaml
```
如果您优先使用默认设置，省略 `--values` 标志。

运行 `helm upgrade --help` 了解更多选项。

## 升级第三方依赖

### Envoy

Envoy 版本能够通过修改在 osm-mesh-config 里面的 `envoyImage` 变量值来更新。当如此做时，推荐指定和 Envoy 版本相关联的镜像摘要来避免变得脆弱而提供链式攻击。例如，要更新 [envoy-alpine 镜像](https://hub.docker.com/r/envoyproxy/envoy-alpine/tags)到 v1.19.1，接下来的命令应该被运行：

```bash
export osm_namespace=osm-system # Replace osm-system with the namespace where OSM is installed
kubectl patch meshconfig osm-mesh-config -n $osm_namespace -p '{"spec":{"sidecar":{"envoyImage":"envoyproxy/envoy-alpine@sha256:6502a637c6c5fba4d03d0672d878d12da4bcc7a0d0fb3f1d506982dde0039abd"}}}' --type=merge
```

在 MeshConfig 资源已经被更新之后，所有的作为网格一部分的 Pod 和部署必须被重启，这样更新版本的 Envoy sidecar 能够被注入到 Pod 上以作为 OSM 执行的自动化 sidecar 注入的一部分。这个可以通过 `kubectl rollout restart deploy` 命令来完成。

### Prometheus、Grafana 和 Jaeger

如果启用了，OSM 的 Prometheus，Grafana 和 Jaeger 服务被部署到其他 OSM Control Plane 组件旁边。虽然这些第三方的依赖不能够通过像 Envoy 这样的 MeshConfig 来更新，它们的版本仍旧能够在部署中被直接更新。例如，要更新 Prometheus 到 v2.19.1，用户能够运行：

```bash
export osm_namespace=osm-system # Replace osm-system with the namespace where OSM is installed
kubectl set image deployment/osm-prometheus -n $osm_namespace prometheus="prom/prometheus:v2.19.1"
```

要更新 Grafana 8.1.0，命令看起来会像下面：

```bash
kubectl set image deployment/osm-grafana -n $osm_namespace grafana="grafana/grafana:8.1.0"
```

对 Jaeger，用于应该运行下面的命令来更新到 1.26.0：

```bash
kubectl set image deployment/jaeger -n $osm_namespace jaeger="jaegertracing/all-in-one:1.26.0"
```

## OSM 升级问题解决指南

#### OSM 网格升级超时

### CPU 不足

如果 `osm mesh upgrade` 命令超时了，很可能是由于 CPU 不足造成的。
1. 检查 Pod 来看看是否它们中有不是充分运行的
```bash
# Replace osm-system with osm-controller's namespace if using a non-default namespace
kubectl get pods -n osm-system
```
2. 如果有任何的 Pod 在待定状态，使用 `kubectl describe` 来检查 `Events` 部分
```bash
# Replace osm-system with osm-controller's namespace if using a non-default namespace
kubectl describe pod <pod-name> -n osm-system
```

如果您看到如下的错误，那么请增加 Docker 能够使用的 CPU 数量。
```bash
`Warning  FailedScheduling  4s (x15 over 19m)  default-scheduler  0/1 nodes are available: 1 Insufficient cpu.`
```
#### 错误确认 CLI 参数
如果 `osm mesh upgrade` 命令仍旧超时，很可能是 CLI/镜像 版本不匹配。

1. 检查 Pod 来看看是否它们中有没有启动和运行的
```bash
# Replace osm-system with osm-controller's namespace if using a non-default namespace
kubectl get pods -n osm-system
```
2. 如果有任何的 Pod 在 Pending 状态，使用 `kubectl describe` 来检查 `Events` 部分的 `Error Validating CLI parameters`
```bash
# Replace osm-system with osm-controller's namespace if using a non-default namespace
kubectl describe pod <pod-name> -n osm-system
```
3. 如果您发现了错误，请检查 Pod 的 log 来排查错误
```bash
kubectl logs -n osm-system <pod-name> | grep -i error
```

如果您看到了如下的错误，那么它就是由于 CLI/镜像 版本不匹配造成的。
```bash
`"error":"Please specify the init container image using --init-container-image","reason":"FatalInvalidCLIParameters"`
```
绕开的方法是在运行 `osm mesh upgrade` 时设置 `container-registry` 和 `osm-image-tag` 标志。
```bash
osm mesh upgrade --container-registry $CTR_REGISTRY --osm-image-tag $CTR_TAG --enable-egress=true
```

### 其他问题
如果您运行中遇到的问题不在上述解决方案之列，请[开一个 GitHub Issue](https://github.com/openservicemesh/osm/issues)。

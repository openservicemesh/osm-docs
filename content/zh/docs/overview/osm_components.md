---
title: "OSM 控制平面组件"
description: "组件"
type: docs
weight: 2
---

## 检查 OSM 组件

一些 OSM 组件将被默认安装在选中的命名空间下，默认是 `osm-system`。要检查它们可以通过如下的 `kubectl` 命令：

```console
# Replace osm-system with the namespace where OSM is installed
$ kubectl get pods,svc,secrets,meshconfigs,serviceaccount --namespace osm-system
```

一些集群范围（非命名空间的）OSM 组件也将被安装。检查它们可以使用如下的 `kubectl` 命令：

```console
kubectl get clusterrolebinding,clusterrole,mutatingwebhookconfiguration
```

在底层，`osm` 在控制平面所在的命名空间里通过 [Helm](https://helm.sh) 库来创建一个 Helm `release` 对象。`helm` CLI 也能够被用来检查安装的 Kubernetes 清单以获取更多细节。参阅 Helm 文档来了解如何[安装 Helm](https://helm.sh/docs/intro/install/)。

```console
$ helm get manifest osm --namespace <osm-namespace>
```

## 组件

让我们看一下各个组件：

### 1 代理控制平面

代理控制平面（Proxy Control Plane）组件在操作[服务网格](https://www.bing.com/search?q=What%27s+a+service+mesh%3F)上扮演了重要的角色。所有的代理被作为 [sidecar](https://docs.microsoft.com/en-us/azure/architecture/patterns/sidecar) 来安装，并且和代理控制平面建立了一个 mTLS gPRC 连接。这些代理持续接收配置更新。这个组件实现了被指定的反向代理所需要的接口。OSM 实现了 [Envoy's go-control-plane xDS v3 API](https://github.com/envoyproxy/go-control-plane)。当[高级 Envoy 特性被需要](https://github.com/openservicemesh/osm/issues/1376)时，xDS v3 API 也能够被用来扩展被 SMI 提供的功能。

### 2 证书管理器

证书管理器是一个组件，它负责给每个参与到服务网格的服务提供一个 TLS 证书。这些服务证书被用来通过 mTLS 来建立和加密服务之间的连接。

### 3 端点提供程序

端点提供程序是一个或者多个组件，用来和参与到服务网格的计算平台（Kubernetes 集群，企业内部及其，或者云提供商的虚拟机）进行通信。端点提供程序解析服务的名字为 IP 地址列表。端点提供程序理解计算提供商所为之实现的特定的基本元素，例如虚拟机，虚拟机规模集和 Kubernetes 集群。

### 4 网格规范

网格规范是一个围绕在现有 [SMI Spec](https://github.com/deislabs/smi-spec) 组件的封装。这个组件抽象了为 YAML 定义所选择的特定存储。这个模块是围绕 [SMI Spec's Kubernetes informers](https://github.com/deislabs/smi-sdk-go) 的有效封装，当前抽象了其存储 (Kubernetes/etcd) 规范。

### 5 网格目录

网格目录是 OSM 组件的中心，其把全部其他组件的输出组合到一个结构中，该结构然后被转换为代理配置，通过代理 Control Plane 派发到所有正在监听的代理中。

这个组件：

1. 同[网格规范模块（4）](#4-网格规范)通信，通过 [SMI Spec](https://github.com/deislabs/smi-spec) 来检测一个 [Kubernetes 服务](https://kubernetes.io/docs/concepts/services-networking/service/)的创建，改变或者删除。
2. 触及[证书管理器（2）](#2-证书管理器)并为新发现的服务请求一个新的 TLS 证书。
3. 通过观察计算平台，经由[端点提供程序（3）](#3-端点提供程序)，获取网格工作负载的 IP 地址。
4. 组合如上 1，2 和 3 的输出到一个数据结构，该结构被传送到[代理 Control Plane（1）](#1-代理-control-plane)，序列化并被发送到所有相关的连接代理。

![diagram](https://user-images.githubusercontent.com/49918230/73008758-27b3a800-3e07-11ea-894e-93f53e08731e.png)

## 详细的组件描述

这个章节勾勒了被采纳的约定和开放式服务网格（OSM）的开发指导。在这个章节被讨论的组件有：

- (A) 代理 [sidecar](https://docs.microsoft.com/en-us/azure/architecture/patterns/sidecar) —— Envoy 或者其他兼容服务网格的反向代理
- (B) [代理证书](#b-代理-tls-证书) —— 特有的 X.509 证书通过[证书管理器](#2-证书管理器)发送到指定的代理
- (C) 服务 —— 在 SMI Spec 中被引用的 [Kubernetes 服务资源](https://kubernetes.io/docs/concepts/services-networking/service/)
- (D) [服务证书](#d-服务-tls-证书) —— 发送给服务的 X.509 证书
- (E) 策略 —— 被目标服务的代理所实施的 [SMI Spec](https://smi-spec.io/) 流量策略
- 对于给定服务，服务端点处理流量的例子：
  - (F) Azure VM —— 运行在一个 Azure VM 上的进程，监听在 IP 1.2.3.11，端口 81 上的连接。
  - (G) Kubernetes Pod —— 运行在一个 Kubernetes 集群上的容器，监听在 IP 1.2.3.12，端口 81 上的连接。
  - (H) 企业内计算 —— 运行在用户的私有数据中心的一台机器上的进程，监听在 IP 1.2.3.13，端口 81 上的连接。

[服务（C）](#c-服务)被分配一个证书（D），同时和一个 SMI Spec 策略相关联（E）。
对于[服务（C）](#c-服务)的流量被端点（F，G，H）处理，这些端点的每一个都被添加了一个代理（A）。
代理（A）有一个专属的证书（B），这个证书不同于服务证书（D），并且这个证书被用于从代理到[代理 Control Plane](#1-代理-control-plane) 的 mTLS 连接。

![service-relationships-diagram](https://user-images.githubusercontent.com/49918230/73034530-1a191500-3e3d-11ea-9a35-a1fd8cce8b53.png)

### (C) 服务

上图中的服务是一个 [Kubernetes 服务资源](https://kubernetes.io/docs/concepts/services-networking/service/)，在 SMI Spec 中被引用。一个例子是下面所定义的 `bookstore` 服务，被一个 `TrafficSplit` 策略引用。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: bookstore
  labels:
    app: bookstore
spec:
  ports:
    - port: 14001
      targetPort: 14001
      name: web-port
  selector:
    app: bookstore

---
apiVersion: split.smi-spec.io/v1alpha2
kind: TrafficSplit
metadata:
  name: bookstore-traffic-split
spec:
  service: bookstore
  backends:
    - service: bookstore-v1
      weight: 100
```

### (A) 代理

在 OSM 里面，`Proxy` 被定义作为一个抽象逻辑组件，它要：

- 支撑一个服务网格进程（容器或者在 Kubernetes 上运行的二进制程序或者一个 VM）
- 维护一个到代理 Control Plane（xDS 服务器）的连接
- 持续接收来自[代理 Control Plane](#1-代理-control-plane)（Envoy xDS go-control-plane 实现）的配置更新（xDS 协议缓冲）
  OSM 交付带 [Envoy](https://www.envoyproxy.io/) 反向代理实现的开箱即用方案。

### (F,G,H) 端点

在 OSM 代码库里面，`Endpoint` 被定义作为一个容器或者一个虚拟机的 IP 地址和端口号的元组，其寄宿着一个代理，这个代理支撑一个进程，这个进程是一个服务的成员，并且作为服务网格的参与者。
[服务端点 (F,G,H)](#fgh-端点) 是实际的二进制程序，其服务于针对[服务 (C)](#c-服务)的流量。
一个端点唯一地标识了一个容器，二进制程序或者一个进程。
它有一个 IP 地址、端口号和给服务的附属物。
一个服务可以有 0 个或者多个端点，每一个端点只能有一个 sidecar 代理。既然一个端点必然属于一个单独的服务，那么一个相关联的代理也必然属于一个单独的服务。

### (D) 服务 TLS 证书

代理，支撑端点，它们构成了一个给定的服务，这个服务将共享证书。
这个证书被用来和服务网格中其他服务的对等代理所支撑的端点建立 mTLS 连接。
服务证书是短期的。
每个服务证书的生命周期[差不多在 24 小时](#证书生命周期)，从而剔除了对于一个证书撤销机制的需要。
OSM 对这类证书声明了一个 `ServiceCertificate` 类型。
在开发者文档的[接口](#接口)章节，`ServiceCertificate` 描述了这类证书如何被引用。

### (B) 代理 TLS 证书

代理 TLS 证书是一个 X.509 证书，通过[证书管理器](#2-证书管理器)被发送到每一个独立的代理
这类证书不同于服务证书，只被专属地用于代理到 Control Plane 的 mTLS 通信。
每一个 Envoy 代理将被引导，其中带一个代理证书，该证书将被用来做 xDS mTLS 通信。
这类证书和那种被发送用来做服务到服务 mTLS 通信的证书是不同的。
OSM 对这类证书声明了 `ProxyCertificate` 类型。
在开发者文档的[接口](#接口)章节，我们引用这些证书作为 `ProxyCertificate`。
这个证书的通用命名利用了 DNS-1123 标准，按照 `<proxy-UUID>.<service-name>.<service-namespace>` 格式。这种格式允许我们来唯一标识连接的代理 (`proxy-UUID`) 和有命名空间的服务，其中这个代理属于 (`service-name.service-namespace`)。

### (E) 策略

在上图 (E) 中被引用的策略组件是任何的引用[服务 (C)](#c-服务)的 [SMI Spec 资源](https://github.com/deislabs/smi-spec#service-mesh-interface)。例如，`TrafficSplit`，应用服务 `bookstore` 和 `bookstore-v1`：

```yaml
apiVersion: split.smi-spec.io/v1alpha2
kind: TrafficSplit
metadata:
  name: bookstore-traffic-split
spec:
  service: bookstore
  backends:
    - service: bookstore-v1
      weight: 100
```

### 证书生命周期

被[证书管理器](#2-证书管理器)发出的服务证书是短期证书，有效性差不多 24 小时。
这个短证书过期机制剔除了对显式的撤销机制的需要。
一个给定的证书过期时间将被以 24 小时为基础随机地缩短或者延长，这是为了避免底层证书管理系统遭遇 [thundering herd 问题](https://en.wikipedia.org/wiki/Thundering_herd_problem)。代理证书，相反地，是长期证书。

### 代理证书，代理和端点的关系

- `ProxyCertificate` 被 OSM 发送到一个 `Proxy`，这个代理被期待在未来的某个时刻会连接到代理 Control Plane。证书被发送之后，并在代理连接到代理 Control Plane 之前，证书处于 `unclaimed` 状态。当一个代理使用证书连接到代理 Control Plane 之后，这个证书的的状态就变为 `claimed`。
- `Proxy` 是反向代理，其尝试连接到代理 Control Plane；`Proxy` 或许可以或者不被允许连接到代理 Control Plane。
- `Endpoint` 是由 `Proxy` 支撑的，并且是一个 `Service` 的成员。OSM 或许已经通过[端点提供程序](#3-端点提供程序)发现了端点，这些端点属于一个给定的服务，但是 OSM 没有看到任何的代理，支撑这些端点，连接到代理 Control Plane。

被发送的 `ProxyCertificates` 集 ∩ 连接的 `Proxies` ∩ 被发现的 `Endpoints` 的交叉集构成了服务网格中的参与者集。

![service-mesh-participants](https://user-images.githubusercontent.com/49918230/73035216-3e75f100-3e3f-11ea-915d-c19eb03ecf97.png)

### Envoy 代理 ID 和服务成员

- 每个 `Proxy` 被发送一个唯一的 `ProxyCertificate`，被用来专注于 xDS mTLS 通信
- `ProxyCertificate` 有一个逐代理唯一的 Subject CN，用来标识 `Proxy` 和它所居住的 `Pod`。
- `Proxy` 的服务成员被 Pod 的服务成员所决定。 当 Envoy 建立 gRPC 到 XDS 并且出示客户端证书时，OSM 标识 Pod。然后证书的 CN 包含一个唯一的 ID，被指派给 Pod 和这个 Pod 所居住的 Kubernetes 命名空间。一旦 XDS 解析了连接 Envoy 的 CN，Pod 上下文就可用了。从 Pod 我们决定了服务成员，Pod 的 ServiceAccount 和其他的 Kubernetes 上下文。
- 这里有一个唯一的 `ProxyCertificate` 被发送到一个 `Proxy`，其专注于一个唯一的 `Endpoint` (Pod)。为了简单，OSM 限制了被一个代理服务的服务数量为 1。
- 一个网格 `Service` 被一个或者多个 `ProxyCertificate` + `Proxy` + `Endpoint` 来构造。

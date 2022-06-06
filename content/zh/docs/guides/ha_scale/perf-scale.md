---
title: "性能和可扩展性"
description: "控制平面和数据平面 sidecar 的性能基准测试以及开放服务网格的可扩展性测试"
type: docs
---

Open Service Mesh aims to create a high-performance service mesh while providing all the essential service mesh features. This document describes the tests performed to evaluate the performance and scalability of OSM.

开放服务网格致力于在提供基本的服务网格功能的同时创建高性能的服务网格。本篇描述了如何对 OSM 进行性能基准测试和可扩展性测试。

## 测试环境

本测试是在 [Microsoft Azure Kubernetes Service](https://azure.microsoft.com/en-us/services/kubernetes-service/) 配置和管理的 Kubernetes 集群上进行的。集群拥有 100 个 Standard_DS2_v2 节点（每个节点有 2 个 vCPU 和 7GB 内存）。在测试确保只有一个 OSM 控制器 pod 使用默认的资源配额运行。OSM 配置在非宽松流量模式。

## 过程

The testing mesh contains two types of services. One is a load generator (modified from [wrk2](https://github.com/giltene/wrk2)), which can send requests at specified rate, load balanced among all other services in the mesh. Currently we test HTTP 1 requests only. The load generator serves as the client side of a request. Throughout our test, there will be at most one load generator service. The other type of service is a simple echo application that replies with whatever it receives from an inbound request. It serves as the server side of a request. We can deploy multiple copies of the echo application in the mesh. In short, the mesh topology is a typical 1-to-N service architecture.
测试网格包含两种类型的服务。一种是负载生成器（修改自 [wrk2](https://github.com/giltene/wrk2)），它可以以指定的速率发送请求，在网格中的所有其他服务之间进行负载平衡。目前我们只测试 HTTP 1 请求。负载生成器充当请求的客户端。在我们的整个测试过程中，最多会有一个负载生成器服务。另一种类型的服务是一个简单的回显应用，它会回复从入站请求中接收到的任何内容。它充当请求的服务器端。我们可以在网格中部署回显应用的多个副本。简而言之，网状拓扑是典型的 1 对 N 服务架构。

<p align="center">
  <img src="/docs/images/perf/mesh-topo.png" width="650"/>
</p>


## 性能

在这个测试中，我们关注 OSM sidecar 带来的延迟开销和资源消耗。我们目前分别以 100、500、1000、2000 和 4000 RPS 测试每个 pod。

### 延迟

Below is the request latency at different RPS levels with and without OSM installed. The added latency comes from both the client and server sidecars.
以下是安装和未安装 OSM 的不同 RPS 级别的请求延迟。增加的延迟来自客户端和服务器 sidecar。

<p align="center">
  <img src="/docs/images/perf/latency-wo-osm.png" width="650"/>
</p>

<p align="center">
  <img src="/docs/images/perf/latency-w-osm.png" width="650"/>
</p>

<table style="width: 100%; display: table"">
<thead>
<tr>
<th>Per pod RPS</th>
<th>100</th>
<th>500</th>
<th>1000</th>
<th>2000</th>
<th>4000</th>
</tr>
</thead>
<tbody>
<tr>
<td>P99 latency w/o OSM (ms)</td>
<td>4.4</td>
<td>4.3</td>
<td>4.2</td>
<td>4.3</td>
<td>4.5</td>
</tr>
<tr>
<td>P99 latency w/ OSM (ms)</td>
<td>9.6</td>
<td>9.4</td>
<td>9.7</td>
<td>9.4</td>
<td>9.8</td>
</tr>
</tbody>
</table>

无论 OSM 是否存在，应用程序在测试的不同 RPS 级别上的性能都非常相似。OSM 在 99 个百分位延迟上增加了大约 5 毫秒的开销。请注意，我们的测试设置非常简单，只有一个客户端/服务器关系。实际可能会因更复杂的应用程序或网格上设置的不同流量策略而异。

### 资源消耗

控制平面资源消耗是稳定的，因为它与数据平面请求速率无关。

对于数据平面上的 sidecar，不同 RPS 级别的内存使用量相对一致，约为 140MB。CPU 使用率有明显的上升趋势，几乎与 RPS 水平成正比。<b>在我们的例子中，使用 60 millicores 的基准 CPU，每 100 RPS 的 sidecar CPU 消耗线性增加约 0.1 个 core。</b>

<p align="center">
  <img src="/docs/images/perf/avg-cpu.png" width="650"/>
</p>

## 可伸缩性

在可扩展性测试中，我们探讨了随着网格中工作负载数量的增加，控制平面和数据平面的资源消耗如何变化。同时部署指定数量的回显服务到网格中。因此，控制平面在部署开始后会收到负载峰值。所有的 Envoy sidecars 都同时连接到控制平面。在那之后，控制平面变得更加空闲。我们故意在 pod 推出时创建此负载峰值，以便查看网格可以处理的最大规模。每轮测试将持续15分钟。将分别测试 200、400、800 和 1600 个 pod 的网格规模。负载生成器服务不用于可伸缩性测试。

### 控制平面

#### CPU 占用

下面显示了控制平面在 Pod 上线时和稳定时的 CPU 使用情况：

<p align="center">
  <img src="/docs/images/perf/ctrl-cpu.png" width="650"/>
</p>

Pod 上线时 CPU 使用率有明显的上升趋势。这是可以理解的，因为控制平面同时服务于所有工作负载，例如颁发证书和生成 xDS 配置。CPU 使用率大致与工作负载的数量成正比。而在没有添加新工作负载的稳定时间内，CPU 使用率下降到几乎为零。

#### Memory 占用

下面显示了 Pod 上线时和稳定时的内存使用情况：

<p align="center">
  <img src="/docs/images/perf/ctrl-memory.png" width="650"/>
</p>

就内存使用而言，pod 上线时和稳定时没有太大区别。内存主要用于存储 xDS 配置，这与网格规模高度相关。内存使用量也随着网格规模大致成比例地增加。在我们的场景中，<b>每个添加到网格的 pod 都需要大约 1MB 的内存。</b>

### 数据平面

通过查看收集的指标，在我们的测试场景中，数据平面 sidecar 的资源消耗不会随着网格大小而发生太大变化。在稳定时间，平均 CPU 使用率接近 0，平均内存使用量在 50MB 左右。CPU 使用率与请求处理有关，内存使用率与本地存储的 Envoy 配置的大小有关。这两个因素不随网格比例而变化。

还值得一提的是，启用宽松模式将导致数据平面使用更多内存，因为每个 sidecar 需要保持所有其他服务之间的连接配置。下图显示了在启用宽松模式的网格中添加 200 和 400 个 Pod 时的 sidecar 内存使用情况。与非宽松模式相比，内存使用量增加了一倍以上。类似的事情发生在控制平面上。

<p align="center">
  <img src="/docs/images/perf/perm-memory.png" width="650"/>
</p>

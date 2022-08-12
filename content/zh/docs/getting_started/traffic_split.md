---
title: "配置流量拆分"
description: "使用 SMI Traffic Split API 来平衡服务之间的流量"
type: docs
weight: 4
---

# 配置两个服务之间的流量拆分

我们将演示如何平衡两个 Kubernetes 服务之间的流量，通常被称为流量拆分。我们将在后端 `bookstore` 服务和 `bookstore-v2` 服务之间来拆分原来定向到*根* `bookstore` 服务的流量。`bookstore` 和 `bookstore-v2` 服务被称为叶子服务。有关如何配置服务的流量拆分，请参阅 [流量拆分的 how-to 指南](/docs/guides/traffic_management/traffic_split.md)。

### 部署 bookstore v2 应用

要演示 SMI 流量访问和拆分策略的使用，我们现在将要部署 bookstore 应用的 v2 版本（`bookstore-v2`）——如果正在使用 openshift 那么请记住，必须添加安全上下文容器给 bookstore-v2 服务账号，就像在[安装指南](/docs/install/#openshift)里面规定的那样。

```bash
# Contains the bookstore-v2 Kubernetes Service, Service Account, Deployment and SMI Traffic Target resource to allow
# `bookbuyer` to communicate with `bookstore-v2` pods
kubectl apply -f https://raw.githubusercontent.com/openservicemesh/osm-docs/{{< param osm_branch >}}/manifests/apps/bookstore-v2.yaml
```

等待 `bookstore` 命名空间里 `bookstore-v2` pod 在运行之后，退出并重新启动端口转发脚本来访问 bookstore-v2：

```bash
bash <<EOF
./scripts/port-forward-bookbuyer-ui.sh &
./scripts/port-forward-bookstore-ui.sh &
./scripts/port-forward-bookstore-ui-v2.sh &
./scripts/port-forward-bookthief-ui.sh &
wait
EOF
```

- [http://localhost:8082](http://localhost:8082) - **bookstore-v2**

`bookstore-v2` 计数应该增长，因为到根 `bookstore` 服务的流量配置了一个标签选择器，该标签选择器选择包括 `bookstore-v1` 和 `bookstore-v2` 后端 pod 的端点。

### 创建 SMI 流量拆分

部署 SMI 流量拆分策略来定向 100% 发送到根 `bookstore` 服务的流量到 `bookstore-v1` 服务后端。这将确保发送到 `bookstore` service 的流量只会发送到 `version v1` 的 bookstore 应用，即 `bookstore-v1` service 的 pod。随后更新 `TrafficSplit` 配置来发送部分流量到使用了 `bookstore-v2` 叶子服务的 `version v2` bookstore 服务。

因此，如果需要流量拆分，确保客户端应用程序始终与根服务通信非常重要。否则，当需要流量拆分时，需要更新客户端应用程序以与 *根* 服务通信。

```bash
kubectl apply -f https://raw.githubusercontent.com/openservicemesh/osm-docs/{{< param osm_branch >}}/manifests/split/traffic-split-v1.yaml
```

_注意：根服务是个 Kubernetes 服务，其标签选择器需要与叶子服务相匹配。在本示例中，根服务 `bookstore` 使用 `app:bookstore` 选择器分别匹配 `bookstore (v1)` 和 `bookstore-v2` deployment 的标签 `app:bookstore,version:v1` 和 `app:bookstore,version:v2`。在 SMI 流量拆分里，根服务能够作为有或者没有 `.<namespace>` 后缀的服务名称被引用。_

对于书籍销售的计数，从 `bookstore-v2` 浏览器窗口看应该保持在 0。这是因为当前的流量拆分策略是当前权重 100 是给了 `bookstore`，另外 `bookbuyer` 正发送流量到 `bookstore` 服务而没有应用发送请求到 `bookstore-v2` 服务。可以通过运行下面命令并同时观察 **Backends** 的属性来验证流量拆分策略。

```bash
kubectl describe trafficsplit bookstore-split -n bookstore
```

### 拆分流量给 bookstore-v2

更新 SMI 流量拆分策略来定向 50% 的流量发送到根 `bookstore` 服务再到 `bookstore` 服务，然后 50% 的流量到 `bookstore-v2` 服务，这可以通过添加 `bookstore-v2` 后端到规范并修改权重域来完成。

```bash
kubectl apply -f https://raw.githubusercontent.com/openservicemesh/osm-docs/{{< param osm_branch >}}/manifests/split/traffic-split-50-50.yaml
```

等待变更的传播，并且在浏览器窗口里观察 `bookstore` 和 `bookstore-v2` 计数器增长。两个计数器都应该增长：


- [http://localhost:8084](http://localhost:8084) - **bookstore**
- [http://localhost:8082](http://localhost:8082) - **bookstore-v2**

### 拆分全部流量到 bookstore-v2

更新 `bookstore-split` 流量拆分来配置全部的流量去往 `bookstore-v2`：

```bash
kubectl apply -f https://raw.githubusercontent.com/openservicemesh/osm-docs/{{< param osm_branch >}}/manifests/split/traffic-split-v2.yaml
```

等待改变的传播并在浏览器窗口里观察 `bookstore-v2` 计数器的增长和 `bookstore` 计数器的停止：

- [http://localhost:8082](http://localhost:8082) - **bookstore-v2**
- [http://localhost:8083](http://localhost:8084) - **bookstore**

现在，定向到 `bookstore` 的全部流量都流向了 `bookstore-v2`。

## 下一步

- [用 Prometheus 和 Grafana 配置可观测性](/docs/getting_started/observability/)
- [清理示例应用并卸载 OSM](/docs/getting_started/cleanup/)
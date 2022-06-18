---
title: "配置流量拆分"
description: "使用 SMI Traffic Split API 来平衡服务之间的流量"
type: docs
weight: 4
---

# 配置两个服务之间的流量拆分

我们将演示如何平衡两个 Kubernetes 服务之间的流量，通常这被认作流量拆分。我们将在后端 `bookstore` 服务和 `bookstore-v2` 服务之间来拆分原来定向到根 `bookstore` 服务的流量。

### 部署 bookstore v2 应用

要演示 SMI 流量访问和拆分策略的使用，我们现在将要部署 bookstore 应用的 v2 版本（`bookstore-v2`）——如果您正在使用 openshift 那么请记住，您必须添加安全上下文容器给 bookstore-v2 服务账号，就像在[安装指南](/docs/install/#openshift)里面规定的那样。

```bash
# Contains the bookstore-v2 Kubernetes Service, Service Account, Deployment and SMI Traffic Target resource to allow
# `bookbuyer` to communicate with `bookstore-v2` pods
kubectl apply -f https://raw.githubusercontent.com/openservicemesh/osm-docs/{{< param osm_branch >}}/manifests/apps/bookstore-v2.yaml
```

等待 `bookstore-v2` pod 在 `bookstore` 命名空间里运行起来，退出并重新启动 `./scripts/port-forward-all.sh` 脚本以可以访问 bookstore-v2。

- [http://localhost:8082](http://localhost:8082) - **bookstore-v2**

计数器 _不_ 应该增长，因为还没有流量流到 `bookstore-v2` 服务。

### 创建 SMI 流量拆分

部署 SMI 流量拆分策略来定向 100% 发送到根 `bookstore` 服务的流量到 `bookstore` 服务后端。

```bash
kubectl apply -f https://raw.githubusercontent.com/openservicemesh/osm-docs/{{< param osm_branch >}}/manifests/split/traffic-split-v1.yaml
```

_注意：根服务能够是任何 Kubernetes 服务。它没有任何标签选择器。它也不需要和任何在流量拆分资源里指定的后端服务发生重叠。在 SMI 流量拆分里，根服务能够被引用作为有或者没有 `.<namespace>` 后缀服务的名称。_

对于书籍销售的计数，从 `bookstore-v2` 浏览器窗口看应该保持在 0。这是因为当前的流量拆分策略是当前权重 100 是给了 `bookstore`，另外 `bookbuyer` 正发送流量到 `bookstore` 服务而没有应用发送请求到 `bookstore-v2` 服务。您可以通过运行下面命令并同时观察**后端**的属性来验证流量拆分策略。

```bash
kubectl describe trafficsplit bookstore-split -n bookstore
```

### 拆分流量给 bookstore-v2

更新 SMI 流量拆分策略来定向 50% 的流量发送到根 `bookstore` 服务再到 `bookstore` 服务，然后 50% 的流量到 `bookstore-v2` 服务，这可以通过添加 `bookstore-v2` 后端到规范并修改权重域来完成。

```bash
kubectl apply -f https://raw.githubusercontent.com/openservicemesh/osm-docs/{{< param osm_branch >}}/manifests/split/traffic-split-50-50.yaml
```

等待变更的传播，并且在您的浏览器窗口里观察 `bookstore` 和 `bookstore-v2` 计数器增长。两个计数器都应该增长：


- [http://localhost:8084](http://localhost:8084) - **bookstore**
- [http://localhost:8082](http://localhost:8082) - **bookstore-v2**

### 拆分全部流量到 bookstore-v2

更新 `bookstore-split` 流量拆分来配置全部的流量去往 `bookstore-v2`：

```bash
kubectl apply -f https://raw.githubusercontent.com/openservicemesh/osm-docs/{{< param osm_branch >}}/manifests/split/traffic-split-v2.yaml
```

等待改变的传播并在您的浏览器窗口里观察 `bookstore-v2` 计数器的增长和 `bookstore` 计数器的停止：

- [http://localhost:8082](http://localhost:8082) - **bookstore-v2**
- [http://localhost:8083](http://localhost:8084) - **bookstore**

现在，定向到 `bookstore` 的全部流量都流向了 `bookstore-v2`。

## 下一步

- [用 Prometheus 和 Grafana 配置可观测性](/docs/getting_started/observability/)
- [清理示例应用并卸载 OSM](/docs/getting_started/cleanup/)
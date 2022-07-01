---
title: "部署示例应用"
description: "部署示例书店应用"
type: docs
weight: 2
aliases: = ["/docs/install/manual_demo/"]
---

# 部署应用

在这个章节我们将部署 5 个不同的 Pod，并且我们将应用一些策略来控制它们之间的流量。

- `bookbuyer` 是一个 HTTP 客户端，它发送请求给 `bookstore`。这个流量是**允许**的。
- `bookthief` 是一个 HTTP 客户端，很像 `bookbuyer`，也会发送 HTTP 请求给 `bookstore`。这个流量应该被**阻止**。
- `bookstore` 是一个服务器，负责对 HTTP 请求给与响应。同时，该服务器也是一个客户端，发送请求给 `bookwarehouse` 服务。这个流量是被**允许**的。
- `bookwarehouse` 是一个服务器，应该只对 `bookstore` 做出响应。`bookbuyer` 和 `bookthief` 都应该被其阻止。
- `mysql` 是一个 MySQL 数据库，只有 `bookwarehouse` 可以访问。


我们将通过使用 [SMI](https://smi-spec.io/) 来定义并部署流量访问策略，这将带给我们这些 Pod 之间允许的和阻止的最终期望的状态：

| from  /   to: | bookbuyer | bookthief | bookstore | bookwarehouse | mysql |
| ------------- | --------- | --------- | --------- | ------------- | ----- |
| bookbuyer     | n/a       | ❌         | ✔         | ❌             | ❌     |
| bookthief     | ❌         | n/a       | ❌         | ❌             | ❌     |
| bookstore     | ❌         | ❌         | n/a       | ✔             | ❌     |
| bookwarehouse | ❌         | ❌         | ❌         | n/a           | ✔     |
| mysql         | ❌         | ❌         | ❌         | ❌             | n/a   |


要展示如何使用 SMI Traffic Split 来拆分流量，我们将部署一个额外的应用。

- `bookstore-v2` —— 这是一个相同的容器，就像我们部署的第一个 `bookstore` 一样，但是对于这个示例而言，我们将假设这是一个我们需要升级到的新版本。

`bookbuyer`，`bookthief`，`bookstore` 和 `bookwarehouse` Pod 将被置于独立的 Kubernetes 命名空间，命名空间的名字和它们各自的一样。`mysql` 将被置于 `bookwarehouse` 命名空间。在这个服务网格里面的每一个新的 Pod 都将被注入一个 Envoy 的 sidecar 容器。

### 创建命名空间

```bash
kubectl create namespace bookstore
kubectl create namespace bookbuyer
kubectl create namespace bookthief
kubectl create namespace bookwarehouse
```

### 添加新的命名空间到 OSM 控制平面

```bash
osm namespace add bookstore bookbuyer bookthief bookwarehouse
```

现在，这四个命名空间的每一个都被标注了 `openservicemesh.io/monitored-by: osm`，并且加了 `openservicemesh.io/sidecar-injection: enabled` 注解。OSM控制器，注意到了这些命名空间上的标注和注解，将开始为这些**新** Pod 注入 Envoy sidecar。

### 创建 Pod，服务，服务账号

创建 `bookbuyer` service account 和部署：

```bash
kubectl apply -f https://raw.githubusercontent.com/openservicemesh/osm-docs/{{< param osm_branch >}}/manifests/apps/bookbuyer.yaml
```

创建 `bookthief` 服务 service account 和部署：

```bash
kubectl apply -f https://raw.githubusercontent.com/openservicemesh/osm-docs/{{< param osm_branch >}}/manifests/apps/bookthief.yaml
```

创建 `bookstore` 服务 service account，服务和部署：

```bash
kubectl apply -f https://raw.githubusercontent.com/openservicemesh/osm-docs/{{< param osm_branch >}}/manifests/apps/bookstore.yaml
```

创建 `bookwarehouse` 服务 service account，服务和部署：

```bash
kubectl apply -f https://raw.githubusercontent.com/openservicemesh/osm-docs/{{< param osm_branch >}}/manifests/apps/bookwarehouse.yaml
```

创建 `mysql` 服务 service account，服务和状态集：

```bash
kubectl apply -f https://raw.githubusercontent.com/openservicemesh/osm-docs/{{< param osm_branch >}}/manifests/apps/mysql.yaml
```

### 检查点：都有哪些被安装了？

一个 Kubernetes 部署，为每个 `bookbuyer`，`bookthief`，`bookstore` 和 `bookwarehouse` 分配的 Pod，还有一个给 `mysql` 的状态集。同样，还有给 `bookstore`，`bookwarehouse` 和 `mysql` 的 Kubernetes 服务和端点。

要在您的集群上查阅这些资源，请运行下面的命令：

```bash
kubectl get pods,deployments,serviceaccounts -n bookbuyer
kubectl get pods,deployments,serviceaccounts -n bookthief

kubectl get pods,deployments,serviceaccounts,services,endpoints -n bookstore
kubectl get pods,deployments,serviceaccounts,services,endpoints -n bookwarehouse
```

另外，[Kubernetes service account](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/) 也被创建用以每个应用。Service account 作为应用的身份来服务，在这个示例的后面将会被使用到，用来创建服务到服务的访问控制策略。

### 查阅应用的 UI

建立客户端端口转发，需按照如下步骤，以访问在 Kubernetes 集群中的应用。我们最好开一个新的终端会话来运行端口转发脚本，以维护这个端口转发会话，同时使用原来的终端继续发送命令。port-forward-all.sh 脚本将寻找一个 `.env` 文件来满足环境变量的需要，从而可以执行该脚本。`.env` 文件创建必要的变量，目标是之前创建的那些命名空间。我们将使用参考文件 `.env.example` 然后运行端口转发脚本。

在一个新的终端会话里，运行下面的命令来启用端口转发进入 Kubernetes 集群，这将从项目根目录开始（https://github.com/openservicemesh/osm 在您的本地的 clone 版本）。

```bash
cp .env.example .env
./scripts/port-forward-all.sh
```

_注意：要覆盖默认的端口，提前要把 `BOOKBUYER_LOCAL_PORT`，`BOOKSTORE_LOCAL_PORT`，`BOOKSTOREv1_LOCAL_PORT`，`BOOKSTOREv2_LOCAL_PORT`，和/或 `BOOKTHIEF_LOCAL_PORT` 变量在脚本前指派。例如：_

```bash
BOOKBUYER_LOCAL_PORT=7070 BOOKSTOREv1_LOCAL_PORT=7071 BOOKSTOREv2_LOCAL_PORT=7072 BOOKTHIEF_LOCAL_PORT=7073 BOOKSTORE_LOCAL_PORT=7074 ./scripts/port-forward-all.sh
```

在一个浏览器中，打开下面的 URL：

- [http://localhost:8080](http://localhost:8080) - **bookbuyer**
- [http://localhost:8083](http://localhost:8083) - **bookthief**
- [http://localhost:8084](http://localhost:8084) - **bookstore**
- [http://localhost:8082](http://localhost:8082) - **bookstore-v2**
  - _注意：示例中的这个页面在当前阶段还不可用。在配置 SMI Traffic Split 期间，它将可用。_

置放好窗口，您将可以同时看到全部四个。网页顶端的页眉指示了应用和其版本。

## 下一步

现在，示例应用正在运行，为应用间的通信[配置流量策略](/docs/getting_started/traffic_policies/)。
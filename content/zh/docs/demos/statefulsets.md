---
title: "配置 Stateful 应用"
description: "使用 OSM 保护 Kubernetes Kafka 集群"
type: docs
weight: 21
---

本指南将说明如何在 Kubernetes 中使用 OSM 和 Statefulsets 配置有状态应用程序。对于这个演示，我们将安装 [Apache Kafka](https://kafka.apache.org/) 及其元数据存储 [Apache Zookeeper](https://zookeeper.apache.org/) 并设置允许它们的流量策略互相访问。最后，我们将测试我们是否能够在所有通信都通过 mTLS 加密的情况下生成和消费来自 Kafka 主题的消息。

## 前置条件

- Kubernetes 集群版本 {{< param min_k8s_version >}} 或者更高并配置默认的[StorageClass](https://kubernetes.io/docs/concepts/storage/storage-classes/) 。
- 已安装 `kubectl` 用来与 API 服务器进行交互。
- 已安装 OSM，版本 >= v1.2.0。
- 已安装 `osm` CLI 用于管理服务网格。
- OSM 的 `MeshConfig` 中已将 [`localProxyMode`](https://deploy-preview-382--osm-docs.netlify.app/docs/api_reference/config/v1alpha2/#config.openservicemesh.io/v1alpha2.SidecarSpec) 设置为`PodIP` 。
- 大多数在 statefulset 中运行的应用程序（包括 Apache Kafka）都要求所有传入的网络流量都通过 pod IP。默认情况下，配置 OSM 通过 localhost 发送流量，因此通过此 MeshConfig 设置修改该行为很重要。 **默认行为将在更高版本的 OSM 中切换**

## 演示

### 安装 Zookeeper

首先，我们需要安装 Apache Zookeeper，它是 Kafka 的元数据存储。我们将首先为我们的 zookeeper pod 创建一个命名空间，并将该命名空间添加到我们的 OSM 网格中：

```shell
# Create a namespace for Zookeeper and add it to OSM
kubectl create ns zookeeper 
osm namespace add zookeeper
```

接下来，我们需要配置流量策略，以允许 Zookeeper 在安装后相互通信。这些策略还将允许我们最终的 Kafka deployment 与 Zookeeper 通信：

```shell
kubectl apply -f - <<EOF
apiVersion: specs.smi-spec.io/v1alpha4
kind: TCPRoute
metadata:
  name: zookeeper
  namespace: zookeeper
spec:
  matches:
    ports:
    - 2181
    - 3181
---
apiVersion: specs.smi-spec.io/v1alpha4
kind: TCPRoute
metadata:
  name: zookeeper-internal
  namespace: zookeeper
spec:
  matches:
    ports:
    - 2181
    - 3181
    - 2888
    - 3888
---
kind: TrafficTarget
apiVersion: access.smi-spec.io/v1alpha3
metadata:
  name: zookeeper
  namespace: zookeeper
spec:
  destination:
    kind: ServiceAccount
    name: zookeeper
    namespace: zookeeper
  rules:
  - kind: TCPRoute
    name: zookeeper
  sources:
  - kind: ServiceAccount
    name: kafka
    namespace: kafka
---
kind: TrafficTarget
apiVersion: access.smi-spec.io/v1alpha3
metadata:
  name: zookeeper-internal
  namespace: zookeeper
spec:
  destination:
    kind: ServiceAccount
    name: zookeeper
    namespace: zookeeper
  rules:
  - kind: TCPRoute
    name: zookeeper-internal
  sources:
  - kind: ServiceAccount
    name: zookeeper
    namespace: zookeeper
EOF
```

请注意，创建了 2 个不同的 `TCPRoute`：一个用于 Zookeeper 之间的通信（允许 4 个端口），另一个用于 Zookeeper 实例外部的客户端（仅允许 2 个端口）。然后，我们依次创建 2 个不同的流量目标。同样，一个用于 Zookeeper 内部流量，另一个用于外部客户端（例如 “kafka” 命名空间中的 “kafka” ServiceAccount）。

现在我们已经准备好了流量策略，接下来准备安装 Zookeeper。这个演示中，我们将利用 Bitnami 发布的 Helm chart，在我们新的 `zookeeper` 命名空间中执行 Helm 安装：

```shell
# Install the Zookeeper helm chart
helm repo add bitnami https://charts.bitnami.com/bitnami
helm install zookeeper bitnami/zookeeper --set replicaCount=3 --set serviceAccount.create=true --set serviceAccount.name=zookeeper --namespace zookeeper
```

确认 `zookeeper` 命名空间中的 pod 已准备就绪：

```shell
kubectl get pod -n zookeeper

NAME                    READY   STATUS    RESTARTS   AGE
zookeeper-zookeeper-0   2/2     Running   0          4m30s
zookeeper-zookeeper-1   2/2     Running   0          4m30s
zookeeper-zookeeper-2   2/2     Running   0          4m29s
```

使用以下命令确认 Zookeeper 已相互建立共识：

```shell
kubectl exec zookeeper-zookeeper-1 -c zookeeper -n zookeeper -- /opt/bitnami/zookeeper/bin/zkServer.sh status

/opt/bitnami/java/bin/java
ZooKeeper JMX enabled by default
Using config: /opt/bitnami/zookeeper/bin/../conf/zoo.cfg
Client port found: 2181. Client address: localhost. Client SSL: false.
Mode: follower
```

Zookeeper 已启动并运行！

### 安装 Kafka

现在是时候安装我们的 Kafka 代理了。为了这个演示，我们将把 Kafka 安装在与 Zookeeper 不同的命名空间中（类似于多租户 Zookeeper 部署）。首先，我们创建一个新的 kafka 命名空间并将其添加到我们的网格中：

```shell
# Create a namespace for Kafka and add it to OSM
kubectl create ns kafka 
osm namespace add kafka
```

就像 Zookeeper 一样，我们需要创建适当的流量策略来允许 Kafka pod 相互通信。我们还允许 default service account 与面向客户端的 Kafka 端口*仅用于演示*。 **此配置不适用于生产**。

```shell
kubectl apply -f - <<EOF
apiVersion: specs.smi-spec.io/v1alpha4
kind: TCPRoute
metadata:
  name: kafka
  namespace: kafka
spec:
  matches:
    ports:
    - 9092
---
apiVersion: specs.smi-spec.io/v1alpha4
kind: TCPRoute
metadata:
  name: kafka-internal
  namespace: kafka
spec:
  matches:
    ports:
    - 9092
    - 9093
---
kind: TrafficTarget
apiVersion: access.smi-spec.io/v1alpha3
metadata:
  name: kafka
  namespace: kafka
spec:
  destination:
    kind: ServiceAccount
    name: kafka
    namespace: kafka
  rules:
  - kind: TCPRoute
    name: kafka
  sources:
  - kind: ServiceAccount
    name: default
    namespace: kafka
---
kind: TrafficTarget
apiVersion: access.smi-spec.io/v1alpha3
metadata:
  name: kafka-internal
  namespace: kafka
spec:
  destination:
    kind: ServiceAccount
    name: kafka
    namespace: kafka
  rules:
  - kind: TCPRoute
    name: kafka-internal
  sources:
  - kind: ServiceAccount
    name: kafka
    namespace: kafka
EOF
```

配置好流量策略后，接下来准备在 kafka 命名空间中安装 Bitnami Kafka Helm chart：

```shell
helm install kafka bitnami/kafka --set replicaCount=3 --set zookeeper.enabled=false --set zookeeperChrootPath='/kafka-root' --set serviceAccount.create=true --set serviceAccount.name=kafka --namespace kafka --set "externalZookeeper.servers={zookeeper-zookeeper-0.zookeeper-zookeeper-headless.zookeeper.svc.cluster.local,zookeeper-zookeeper-1.zookeeper-zookeeper-headless.zookeeper.svc.cluster.local,zookeeper-zookeeper-2.zookeeper-zookeeper-headless.zookeeper.svc.cluster.local}"
```

这里有几个重要的细节需要注意。一方面，我们禁用了预先安装在 Kafka Helm chart 中的 zookeeper 节点，因为我们已经安装了自己的节点。此外，我们设置了一个特定的 Zookeeper chroot 路径，用于将我们的 Kafka 元数据与 Zookeeper 实例的其他潜在客户隔离开来。最后，我们传入一组服务 FQDN，以便 Kafka 可以连接到我们的 Zookeeper。从技术上讲，我们不需要全部 3 个 FQDN；只有 1 个服务必须是可访问的，Kafka 将能够自行找到其他节点。现在，我们只需要等待 Kafka pod 出现即可：

```shell
kubectl get pod -nkafka

NAME      READY   STATUS    RESTARTS   AGE
kafka-0   3/3     Running   1          3m57s
kafka-1   3/3     Running   1          3m56s
kafka-2   3/3     Running   1          3m54s
```

非常好！所有计算都已到位

### 汇总

现在确认不同组件之间的所有通信都正常工作（zookeeper->zookeeper、kakfa->kafka 和 kafka->zookeeper）。为此，我们将启动一个运行 kafka 镜像的测试 pod，以便我们可以生产和消费主题：

```shell
kubectl run --rm -it kafka-client --image docker.io/bitnami/kafka:3.1.0-debian-10-r60 --namespace kafka -- bash
```

运行此命令后，应该打开一个 bash shell 以将命令发送到 `kafka-client` pod。在该控制台中，运行以下命令：

```shell
kafka-console-producer.sh --broker-list kafka-0.kafka-headless.kafka.svc.cluster.local:9092 --topic test
```

这里，我们只传入了 1 个 kafka 代理，因为就像 zookeeper 一样，客户端只需要与单个主机对话即可检索有关集群中其他节点的元数据。运行此命令后，应该会看到另一个提示；这是 Kafka shell，一旦按 Enter 键，此处键入的任何文本都将被序列化并作为 Kafka 消息发送。试一试吧！

（注意：`>` 是你在 shell 中输入时会看到的提示，不要复制粘贴！）

```shell
> hello
> world
```

太好了 - 已经向集群内的 Kafka 代理发送了两条 Kafka 消息：“hello” 和 “world”。现在，按 Ctrl-C 几次以退出 Kafka 提示符并返回 bash shell。现在让我们启动一个 Kafka 消费者来读取我们刚刚编写的消息。 在该 bash shell 中，运行以下命令：

```shell
kafka-console-consumer.sh --bootstrap-server kafka.kafka.svc.cluster.local:9092 --topic test --from-beginning
```

运行此命令时，可能会看到类似错误的消息，但这些最有可能是 [Kakfa 代理重新平衡](https://stackoverflow.com/questions/30988002/what-does-rebalancing-mean-in-apache-kafka-context) 的现象，是一个完全正常的过程。无论哪种方式，应该很快就会看到 “hello” 和 “world” 消息出现在 shell 中，由换行符分隔。恭喜！刚刚在 Kubernetes 中运行了 2 个有状态应用程序，使用 OSM 的 mTLS 透明地保护所有组件之间的通信。

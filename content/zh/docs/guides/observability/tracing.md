---
title: "链路追踪"
description: "使用 Jaeger 进行链路追踪"
type: docs
weight: 4
---

# 链路追踪
开放服务网格（OSM）可以选择安装 Jaeger 来进行链路追踪。同样，在安装过程中或着在运行时通过修改自定义资源 `osm-mesh-config` （`values.yaml` 中的 `tracing` 配置段落），来启用和配置链路追踪。任何时候，链路追踪都可以被启用，禁用或者配置，来支持 BYO 场景。

当 OSM 部署并启用了链路追踪功能时，OSM 的控制平面将会使用 [用户提供的链路追踪信息](#链路追踪的配置项) 来引导 Evnoy 在合适的时候将追踪信息发送到合适的地点。如果链路追踪在启用时，缺少用户提供的配置项，OSM 将会使用 `values.yaml` 中的默认值。`tracing-address` 是一个 FQDN，表示那些被 OSM 注入的 Envoy 发送追踪信息的目的地。

OSM 支持那些使用 Zipkin 协议的应用进行链路追踪。

## Jaeger
[Jaeger](https://www.jaegertracing.io/) 是一个开源的分布式链路追踪系统，用于分布式系统的监控和故障排查。它能让在系统中获得细粒度的监控指标和分布式追踪信息，然后可以观测哪些微服务正在通信、请求发往何处、以及它们花了多少时间。可以使用它来剖析特定的请求和相应，看看它们是在何时以及如何产生的。

当链路追踪启用时，Jaeger 可以接收来自网格中 Evnoy 的 span，然后在通过端口转发后的 Jaeger 界面上查看和查询它。

OSM 命令行提供了安装 OSM 同时部署 Jaeger 的能力，但也支持在安装后，将 OSM 的链路追踪配置指向自行管理的 Jaeger。

### 自动部署 Jaeger
默认情况下，Jaeger 的部署和链路追踪都是一起被禁用的。

在安装期间，通过使用 `--set=osm.deployJaeger=true` OSM 命令行参数，可以自动地部署一个 Jaeger 实例。这将部署一个 Jaeger pod 到网格的命名空间下。

除此之外，在代理上 OSM 必须被设置为启用链路追踪功能；这可以通过 MeshConfig 中的 `tracing` 部分来完成。

下面的命令将在安装 OSM 的过程中部署 Jaeger，并按照新部署的 Jaeger 实例地址来配置链路追踪参数
```bash
osm install --set=osm.deployJaeger=true,osm.tracing.enable=true
```

这个默认的安装模式使用了运行 Jaeger UI、collector、query 和 agent 组件的 [All-in-one  Jaeger 版本] (https://www.jaegertracing.io/docs/1.22/getting-started/#all-in-one) 

### BYO (自维护)
这一章节记录了将一个已经运行的 Jaeger 实例集成到 OSM 控制平面所需要的额外步骤。
> 注意：这份指南概括了针对使用 Jaeger 的步骤，但是也可以通过适当的参数来使用自己的链路追踪应用。OSM 支持那些使用 Zipkin 协议的应用进行链路追踪

#### 先决条件
* 一个运行的 Jaeger 实例
    * [Getting started with Jaeger](https://www.jaegertracing.io/docs/1.22/getting-started/) includes a sample app as a demo
    * [开始使用 Jaeger](https://www.jaegertracing.io/docs/1.22/getting-started/) 包含了单个示例应用的演示

#### 链路追踪的配置项
根据是否已经安装了 OSM 或者安装 OSM 过程中是否部署了 Jaeger 和启用链路追踪，下面的章节概述了必要的配置修改。无论那种情况，下面提到的 `values.yaml` 中 `tracing` 的配置项将被修改，以指向 Jaeger 实例：
1. `enable`: 设置为 `true` 使 Envoy connection manager 发送链路追踪数据到一个指定的地址（集群）
2. `address`： 设置为 Jaeger 实例所在的目标集群
3. `port`：设置为期望使用的目标监听端口
4. `endpoint`：设置为发送 span 的目标 API 地址或者 collector 接入点


#### a) 在安装 OSM 控制平面安装完毕后启用链路追踪

如果已经安装了 OSM，OSM 的 MeshConfig 当中的 `tracing` 配置项必须修改，通过命令：

```bash
# 使用样例值的链路追踪配置
kubectl patch meshconfig osm-mesh-config -n osm-system -p '{"spec":{"observability":{"tracing":{"enable":true,"address": "jaeger.osm-system.svc.cluster.local","port":9411,"endpoint":"/api/v2/spans"}}}}'  --type=merge
```

可以通过检查 `osm-mesh-config` 资源来确认这些变更是否已经生效：
```bash
kubectl get meshconfig osm-mesh-config -n osm-system -o jsonpath='{.spec.observability.tracing}{"\n"}'
```

#### b) 在 OSM 控制平面安装期间启用链路追踪

在安装过程中部署自管理的 Jaeger 实例，可以像下面一样，使用 `--set` 参数来修改配置项

```bash
osm install --set osm.tracing.enable=true,osm.tracing.address=<链路追踪系统的主机名>,osm.tracing.port=<链路追踪系统的端口>,osm.tracing.endpoint=<链路追踪系统接入点>
```

## 通过端口转发来访问 Jaeger 界面
Jaeger 的界面运行在 16686 端口上。要访问 Web 界面，可以使用 `kubectl port-forward` 命令：

```bash
OSM_POD=$(kubectl get pods -n "$K8S_NAMESPACE" --no-headers  --selector app=jaeger | awk 'NR==1{print $1}')

kubectl port-forward -n "$K8S_NAMESPACE" "$OSM_POD"  16686:16686
```
在浏览器上访问 `http://localhost:16686/` 来访问界面。


## 使用 Jaeger 进行链路追踪的示例
这一章节将介绍创建一个简单的 Jaeger 实例并在 OSM 中启用链路追踪的过程。

1. 完成 [OSM 演示](https://github.com/openservicemesh/osm/blob/{{< param osm_branch >}}/demo/README.md) 并部署 Jaeger。有两种选择：
    - 要自动部署 Jaeger，直接在 `.env` 文件中将 `DEPLOY_JAEGER` 设置为 true
    - 要使用自维护的 Jaeger，通过下面的命令，部署 [Jaeger 提供的](https://www.jaegertracing.io/docs/1.22/getting-started/#all-in-one) 演示实例。如果希望在不同的命名空间下部署 Jaeger，确保在下面的步骤进行修改：

        创建 Jaeger service。
        ```yaml
        kubectl apply -f - <<EOF
        ---
        kind: Service
        apiVersion: v1
        metadata:
          name: jaeger
          namespace: osm-system
          labels:
            app: jaeger
        spec:
          selector:
            app: jaeger
          ports:
          - protocol: TCP
            # 服务端口和目标端口一致
            port: 9411
          type: ClusterIP
        EOF
        ```

        创建 Jaeger 部署。
        ```yaml
        kubectl apply -f - <<EOF
        ---
        apiVersion: apps/v1
        kind: Deployment
        metadata:
          name: jaeger
          namespace: osm-system
          labels:
            app: jaeger
        spec:
          replicas: 1
          selector:
            matchLabels:
              app: jaeger
          template:
            metadata:
              labels:
                app: jaeger
            spec:
              containers:
              - name: jaeger
                image: jaegertracing/all-in-one
                args:
                  - --collector.zipkin.host-port=9411
                imagePullPolicy: IfNotPresent
                ports:
                - containerPort: 9411
                resources:
                  limits:
                    cpu: 500m
                    memory: 512M
                  requests:
                    cpu: 100m
                    memory: 256M
        EOF
        ```

2. 使用合适的配置来启用链路追踪。如果已经在其它命名空间下安装了 Jaeger，在下面步骤中替换 `osm-system` 成相应的值

    ```bash
    kubectl patch meshconfig osm-mesh-config -n osm-system -p '{"spec":{"observability":{"tracing":{"enable":true,"address": "jaeger.osm-system.svc.cluster.local","port":9411,"endpoint":"/api/v2/spans"}}}}'  --type=merge
    ```

3. 参考[上面的](#通过端口转发来访问-Jaeger-界面)指导，通过端口转发访问 web 界面

4. 在浏览器中，应该可以看到一个 `Service` 下拉菜单，能够让选择在 bookstore 演示中部署的各种应用。

    a) 选择一个服务来查看它所有的 span。例如，选择了 `bookbuyer` 并往回查看一个小时，就可以依照时间顺序查看它和 `bookstore-v1` 和 `bookstore-v2` 的交互。
    <p align="center">
        <img src="/docs/images/jaeger-search-traces.png" width="100%"/>
    </p>
    <center><i>在 Jaeger 界面上查询 bookbuyer 的追踪记录</i></center><br>

    b) 点击任一项目来查看详情

    c) 选择多个项目来对比追踪信息。例如，可以对比 `bookbuyer` 和 `bookstore-v1` 以及 `bookstore-v2` 某一时刻的的交互：
    <p align="center">
        <img src="/docs/images/jaeger-compare-traces.png" width="100%"/>
    </p>
    <center><i>`bookbuyer` 和 `bookstore-v1` 以及 `bookstore-v2` 的交互</i></center><br>

    d) 点击 `System Architecture` 标签页，可以看到一张各个应用之间交互/通信的图。它展示了应用之间的流量是如何流转的。
    <p align="center">
        <img src="/docs/images/jaeger-system-architecture.png" width="40%"/>
    </p>
    <center><i>bookstore 演示应用交互过程的有向无环图</i></center><br>

如果在 Jaeger 界面上没有看到 bookstore 演示应用，跟踪 `bookbuyer` 的日志，确保应用之间交互式正常的。

```bash
POD="$(kubectl get pods -n "$BOOKBUYER_的命名空间" --show-labels --selector app=bookbuyer --no-headers | grep -v 'Terminating' | awk '{print $1}' | head -n1)"

kubectl logs "${POD}" -n "$BOOKBUYER_的命名空间" -c bookbuyer --tail=100 -f
```

预期能看到：
```bash
"MAESTRO! THIS TEST SUCCEEDED!"
```
这表明问题不是因为 Jaeger 或者链路追踪配置导致的。

## 在应用当中集成 Jaeger 链路追踪

使用 Jaeger 链路追踪并不是没有成本的。为了让 Jaeger 能够自动关联请求和追踪信息，应用应当正确地发送追踪信息。

目前在开放服务网格的 sidecar 代理配置中，Zipkin 被用来作为 HTTP 追踪器。因此，一个应用可以利用 Zipkin 支持的头部来提供追踪信息。在一个追踪的初始请求中，Zipkin 插件会生成必要的 HTTP 头部。如果应用希望把后续请求添加到当前追踪当中， 它应当传递如下的头部：

* `x-request-id`
* `x-b3-traceid`
* `x-b3-spanid`
* `x-b3-parentspanid`

查看更多详细信息，请参考 [Envoy链路追踪文档](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/observability/tracing)。

## Jaeger/链路追踪问题排查

当链路追踪没有如预期工作的时候。

### 1. 确认链路追踪已经启用
确保 `tracing` 配置当中，`enable` 字段被设置为 `true`：
```bash
kubectl get meshconfig osm-mesh-config -n osm-system -o jsonpath='{.spec.observability.tracing.enable}{"\n"}'
true
```

### 2. 确认链路追踪的配置如预期被设置
如果链路追踪已经启用，可以在 `osm-mesh-config` 资源中检查用于追踪的特定的 `address`, `port` 和 `endpoint`：
```bash
kubectl get meshconfig osm-mesh-config -n osm-system -o jsonpath='{.spec.observability.tracing}{"\n"}'
```
检查 `address` 字段，确保 Envoy 指向了预期使用的 FQDN 地址。

### 3. 确认链路追踪的配置如预期被正确使用
往深一点调查，可能也要检查一下 MeshConfig 中的配置值是否被正确使用。使用下面的命令，获取有问题的 pod 的配置，并将输出保存到一个文件当中。

```bash
osm proxy get config_dump -n <pod-namespace> <pod-name> > <file-name>
```
使用喜欢的文本编辑器打开文件并搜索 `envoy-tracing-cluster`。应该可以看到正在使用的链路追踪配置。bookbuyer pod 的样例输出如下：
```console
"name": "envoy-tracing-cluster",
      "type": "LOGICAL_DNS",
      "connect_timeout": "1s",
      "alt_stat_name": "envoy-tracing-cluster",
      "load_assignment": {
       "cluster_name": "envoy-tracing-cluster",
       "endpoints": [
        {
         "lb_endpoints": [
          {
           "endpoint": {
            "address": {
             "socket_address": {
              "address": "jaeger.osm-system.svc.cluster.local",
              "port_value": 9411
        [...]
```

### 4. 确认 OSM 的控制器已安装且 Jaeger 被自动部署 [可选]
如果使用自动部署，可以额外检查 Jaeger 服务和 Jaeger 的部署：
```bash
# 假设 OSM 被安装到了 osm-system 命名空间：
kubectl get services -n osm-system -l app=jaeger

NAME     TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)    AGE
jaeger   ClusterIP   10.99.2.87   <none>        9411/TCP   27m
```

```bash
# 假设 OSM 被安装到了 osm-system 命名空间：
kubectl get deployments -n osm-system -l app=jaeger

NAME     READY   UP-TO-DATE   AVAILABLE   AGE
jaeger   1/1     1            1           27m
```

### 5. 确认 Jaeger pod 的就绪、响应、以及健康状况
检查 Jaeger pod 是否在选择部署的命名空间中运行
> 下面的命令针对的是 OSM 自动部署的 Jaeger；按需将命名空间和标签值替换成自己链路追踪实例的值：
```bash
kubectl get pods -n osm-system -l app=jaeger

NAME                     READY   STATUS    RESTARTS   AGE
jaeger-8ddcc47d9-q7tgg   1/1     Running   5          27m
```

要获取关于 Jaeger 实例的信息，使用 `kubectl describe pod` 命令，并检查输出中的 `Events`。
```bash
kubectl describe pod -n osm-system -l app=jaeger
```

### 外部资源
* [Jaeger 故障排查文档](https://www.jaegertracing.io/docs/1.22/troubleshooting/)
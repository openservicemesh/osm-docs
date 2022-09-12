---
title: "度量"
description: "使用 Promtheus 对代理以及 OSM 控制平面进行度量"
type: docs
weight: 1
---

# 度量
开放服务网格（OSM）会生成网格以及 OSM 控制平面所有流量的详细监控指标。这些指标提供了对网格中的应用行为和网格本身的洞察力，帮助用户排除故障、维护和分析其应用。

OSM 直接从 sidecar 代理（Envoy）中收集监控指标。通过这些指标，用户可以获取关于总体流量、流量中的错误、以及请求的响应耗时的信息。

除此之外，OSM 生成了控制平面组件的指标。这些指标可以用来监控服务网格的行为和健康状况。

OSM 使用 [Prometheus][1] 来收集和存储网格中运行的所有应用程序的持续的流量指标和统计数据。Prometheus 是一个开源的监控和告警工具集，通常用于（但不限于）Kubernetes 和服务网格环境。

每一个网格中的应用都在一个 Pod 中运行，这个 Pod 带有一个 Envoy sidecar，它以 Prometheus 格式暴露指标（代理指标）。此外，在启用了度量功能的命名空间中，每个网格中的 Pod，都带有 Prometheus 注解，这使得 Prometheus 服务器可以动态地采集应用程序指标。每当一个 Pod 被添加到网格中时，这种机制就会让 Prometheus 开始自动采集指标。

OSM 的指标可以在 [Grafana][8] 中查看，这是一个开源的可视化和分析软件。它可以让查询、可视化、依照指标进行报警、以及浏览指标。

Grafana 使用 Prometheus 作为后端的时序数据库。如果在 OSM 安装过程中选择一并部署 Grafana 和 Prometheus，过程中将设置一些必要的规则，以便它们进行交互。相反地，在 “自维护” 或者说 “BYO” 模式下（后续将进一步解释），这些组件将由用户来负责安装。

## 安装度量组件

OSM 可以在安装期间部署 Prometheus 和 Grafana，或者 OSM 也可以连接到一个已经存在的 Prometheus 以及/或者 Grafana 实例上。我们将后一种模式称为 “自维护” 或者 “BYO“。下面的章节描述了如何通过让 OSM 来自动部署度量组件以启用度量功能，也包括 BYO 的方式。

### 自动化部署

默认情况下，Prometheus 和 Grafana 都被禁用了。

然后，当设置了 `--set=osm.deployPrometheus=true` 参数的时候，OSM 安装过程会部署 Prometheus 实例来采集 sidecar 接口暴露的指标。依据用户的指标采集配置，OSM 将会为那些网格中的 pod 标记必要的指标采集注解，让 Prometheus 能够访问和采集这些 pod 的相关指标。[指标采集的配置文件](https://github.com/openservicemesh/osm/blob/{{< param osm_branch >}}/charts/osm/templates/prometheus-configmap.yaml) 定义了 Prometheus 的默认行为，并设置了 OSM 需要被采集的指标。

使用 `osm install` 命令时，通过 `--set=osm.deployGrafana=true` 参数来安装 Grafana 用于指标可视化。在 [OSM Grafana 面板](#osm-grafana-面板) 小节中，展示了 OSM 提供的一个预先配置好的面板。

```bash
 osm install --set=osm.deployPrometheus=true \
             --set=osm.deployGrafana=true
```

### 自维护的部署方式

#### Prometheus

下面的章节记录了让已经在运行的 Prometheus 实例去抓取 OSM 网格接口的所需步骤。

##### 使用自维护的 Prometheus 的先决条件列表

- 在网格_之外_已经有一个运行的可访问的 Prometheus 实例。
- 一个运行中的 OSM 控制平面，但没有部署度量组件。
- 我们假设 Grafana 已经可以访问 Prometheus，Prometheus 或者 Grafana 的 web 端口已经对外暴露或者配置了端口转发，并且 Prometheus 访问 Kubernetes API 服务已经配置妥当，否则如下步骤无法指导完成配置。

##### 配置

- 确保 Prometheus 实例已经设置了恰当的 RBAC 规则，使得其能够同时访问 pod 和 Kubernetes API - 根据不同的部署场景和特殊要求，这可能会有所不同：
```yaml
- apiGroups: [""]
   resources: ["nodes", "nodes/proxy",  "nodes/metrics", "services", "endpoints", "pods", "ingresses", "configmaps"]
   verbs: ["list", "get", "watch"]
 - apiGroups: ["extensions"]
   resources: ["ingresses", "ingresses/status"]
   verbs: ["list", "get", "watch"]
 - nonResourceURLs: ["/metrics"]
   verbs: ["get"]
```

- 如果需要，可以通过 Prometheus Service 配置，让 Prometheus 采集自己的指标：
```yaml
annotations:
  prometheus.io/scrape: "true"
  prometheus.io/port: "<供 prometheus 采集指标的端口>" # 根据 prometheus deployment 的实际情况配置 - OSM 自动部署的实例默认使用 7070 端口, 可通过 `values.yaml` 来配置
```

- 修改 Prometheus 的 configmap，让它访问 pod/Envoy 的接口。OSM 自动给 pod 添加端口注解，负责将监听器配置推送给 pod，以供 Prometheus 来访问：
```yaml
- job_name: 'kubernetes-pods'
   kubernetes_sd_configs:
   - role: pod
   relabel_configs:
   - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
      action: keep
      regex: true
   - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
      action: replace
      target_label: __metrics_path__
      regex: (.+)
   - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
      action: replace
      regex: ([^:]+)(?::\d+)?;(\d+)
      replacement: $1:$2
      target_label: __address__
   - source_labels: [__meta_kubernetes_namespace]
      action: replace
      target_label: source_namespace
   - source_labels: [__meta_kubernetes_pod_name]
      action: replace
      target_label: source_pod_name
   - regex: '(__meta_kubernetes_pod_label_app)'
      action: labelmap
      replacement: source_service
   - regex: '(__meta_kubernetes_pod_label_osm_envoy_uid|__meta_kubernetes_pod_label_pod_template_hash|__meta_kubernetes_pod_label_version)'
      action: drop
   - source_labels: [__meta_kubernetes_pod_controller_kind]
      action: replace
      target_label: source_workload_kind
   - source_labels: [__meta_kubernetes_pod_controller_name]
      action: replace
      target_label: source_workload_name
   - source_labels: [__meta_kubernetes_pod_controller_kind]
      action: replace
      regex: ^ReplicaSet$
      target_label: source_workload_kind
      replacement: Deployment
   - source_labels:
      - __meta_kubernetes_pod_controller_kind
      - __meta_kubernetes_pod_controller_name
      action: replace
      regex: ^ReplicaSet;(.*)-[^-]+$
      target_label: source_workload_name
```

#### Grafana

下面的章节假设一个 Prometheus 实例已经配置成已运行的 Grafana 实例的数据源。如何创建和配置一个 Grafana 实例的示例，可以参考 [Prometheus 和 Grafana](/docs/demos/prometheus_grafana) 的演示

##### 导入 OSM 面板

OSM 面板可以在 [我们的仓库中](https://github.com/openservicemesh/osm/tree/{{< param osm_branch >}}/charts/osm/grafana/dashboards) 获取，它可以作为 json 数据导入到 web 管理门户。

可以在 [Prometheus 和 Grafana](/docs/demos/prometheus_grafana) 演示中找到导入 OSM 面板的详细指导步骤。[OSM Grafana 面板](#osm-grafana-面板)展示了一个预先配置好的面板概览。

## 指标采集

指标采集可以通过 `osm metrics` 命令来配置。默认情况下，OSM **不会** 为网格中的 pod 配置指标采集。指标采集可以在命名空间级别范围内启用或者关闭，以便做了配置的命名空间中的 pod 可以启用或者禁用指标采集。

对于需要采集的指标，需要符合下面的先决条件：

- 命名空间内必须是网格中的一部分，例如，它必须标记上 `openservicemesh.io/monitored-by` 标签，设置恰当的网格名字。这可以通过 `osm namespace add` 命令来完成。
- 一个运行中的服务，可以访问 Prometheus 的接口。OSM 为[自动部署的 Prometheus](#自动部署) 提供了相应配置；或者用户可以[自行部署自维护的 Prometheus](#prometheus)。

为一个或者多个命名空间启用指标采集：

```bash
osm metrics enable --namespace test
osm metrics enable --namespace "test1, test2"

```

为一个或者多个命名空间禁用指标采集：

```bash
osm metrics disable --namespace test
osm metrics disable --namespace "test1, test2"
```

为命名空间启用指标采集，会导致 osm-injector 为命名空间的 pod 添加如下注解：

```yaml
prometheus.io/scrape: true
prometheus.io/port: 15010
prometheus.io/path: /stats/prometheus
```

## 可用的指标

OSM 提供了关于网格中的流量以及关于控制平面的指标。

### 自定义 Envoy 的指标

每一个 Envoy 代理按照 Prometheus 格式生成指标。要进一步了解生成了什么样的指标，请参考[Envoy 的文档](https://www.envoyproxy.io/docs/envoy/v1.17.2/operations/stats_overview)。OSM 默认的 Prometheus 配置只采集代理生成的所有指标中的一部分。

为了实现 [SMI 指标规范][7]，OSM 添加了一个自定义的 WebAssembly 插件到每一个 Envoy 代理中，这些代理会生成如下 HTTP 流量的统计信息：

`osm_request_total`：随每一个代理的请求自增的 counter 指标。通过查询这个指标可以了解网格中的服务的请求成功和失败率。

`osm_request_duration_ms`: 一个以毫秒为单位统计的，表示代理请求的持续时间的 histogram 指标。通过查询这个指标来了解网格中服务间的延迟。

这两个指标都有如下标签：

`source_kind`：产生请求的工作负载的 Kubernetes 资源类型，例如 `Deployment`、 `DaemonSet` 等等。

`destination_kind`：处理请求的工作负载的 Kubernetes 资源类型，例如 `Deployment`、 `DaemonSet` 等等。

`source_name`：产生请求的工作负载的 Kubernetes 的名字。

`destination_name`：处理请求的工作负载的 Kubernetes 的名字。

`source_pod`：产生请求的 pod 在 Kubernetes 中的名字。

`destination_pod`：处理请求的 pod 在 Kubernetes 中的名字。

`source_namespace`：产生请求的工作负载在 Kubernetes 中的命名空间。

`destination_namespace`：处理请求的工作负载在 Kubernetes 中的命名空间。

除此之外，`osm_request_total` 指标有一个 `response_code` 标签，表示请求的 HTTP 状态码，例如：`200`、 `404` 等等。

#### 已知问题

- 从 Envoy 调用本地响应的 HTTP 请求会产生带有 “unknown” `destination_*` 标签的指标。
  - 在演示中，这包括了那些 bookthief 到 bookstore 的请求。
- 指标只记录了那些属于网格的接入点之间的流量。 Ingress 和 egress 的流量没有统计数据记录。
- Prometheus 当中记录的指标，标签里所有实例名中的 '-' 和 '.' 都会被转成 '\_'。这是因为 proxy-wasm 通过 metric 的名称给 metrics 添加标签，而 Prometheus 不允许在度量名称中使用 '-' 或 '.'，因此 Envoy 将它们全部转换为 Prometheus 格式的 '\_'。这意味着一个名为 'abc-123' 的 pod 在 Prometheus 当中会被标记成 'abc_123'，并且 'abc-123' 和 'abc.123' 的 pod 的指标会被记录到一个 'abc_123' 的 pod 上，只能通过包含了 pod IP 地址的 'instance' 标签来区分它们。

### 控制平面

下面这些是 OSM 控制平面依照 Prometheus 格式生成的指标。`osm-controller` 和 `osm-injector` pod 都有如下 Prometheus 的注解。

```yaml
annotations:
   prometheus.io/scrape: 'true'
   prometheus.io/port: '9091'
```
| 指标                                | 类型      | 标签                                  | 描述                                                                      |
| ------------------------------------- | --------- | --------------------------------------- | -------------------------------------------------------------------------------- |
| osm_k8s_api_event_count               | Count     | type, namespace                         | 从 Kubernetes API Server 接收到的事件的数量                         |
| osm_proxy_connect_count               | Gauge     |                                         | 连接到 OSM controller 的代理的数量                                   |
| osm_proxy_reconnect_count             | Count     |                                         | IngressGateway 定义了一个 ingress gateway 的证书规范 |
| osm_proxy_response_send_success_count | Count     | proxy_uuid, identity, type              | 成功发送到代理的响应的数量                                 |
| osm_proxy_response_send_error_count   | Count     | proxy_uuid, identity, type              | 发送到代理的出错的响应的数量                       |
| osm_proxy_config_update_time          | Histogram | resource_type, success                  | 追踪代理配置消耗时间的直方图                            |
| osm_proxy_broadcast_event_count       | Count     |                                         | OSM controller 发出的 ProxyBroadcast 事件数量                 |
| osm_proxy_xds_request_count           | Count     | proxy_uuid, identity, type              | 代理产生的 XDS 请求的数量                                           |
| osm_proxy_max_connections_rejected    | Count     |                                         | 因为最大连接数限制配置导致的被拒绝的代理连接的数量 |
| osm_cert_issued_count                 | Count     |                                         | 签发给代理的 XDS 证书的数量                               |
| osm_cert_issued_time                  | Histogram |                                         | 追踪签发 xds 证书消耗时间的直方图                           |
| osm_admission_webhook_response_total  | Count     | kind, success                           | admission webhook 的响应次数                            |
| osm_error_err_code_count              | Count     | err_code                                | OSM 产生的错误的数量                                            |
| osm_http_response_total               | Count     | code, method, path                      | 发送 HTTP 响应的数量                                                   |
| osm_http_response_duration            | Histogram | code, method, path                      | 以秒为单位的统计的，发送的 HTTP 响应时间的直方图                                       |
| osm_feature_flag_enabled              | Gauge     | feature_flag                            | 表示一个特性标记配置是启用 (1) 还是禁用 (0)                 |
| osm_conversion_webhook_resource_total | Count     | kind, success, from_version, to_version | 被 conversion webhooks 处理过的资源的数量                            |
| osm_events_queued                     | Gauge     |                                         | Number of events seen but not yet processed by the control plane                 |
| osm_reconciliation_total              | Count     | kind                                    | Counter of resource reconciliations invoked                                      |

#### 指标中的状态码

当 OSM 控制平面产生一个错误的时候，Prometheus 里这个错误码相关的 ErrCodeCounter 指标会增长。要查看完成错误码列表和它们的描述，请参考 [OSM 控制平面错误码故障排查指南](/docs/guides/troubleshooting/control_plane_error_codes)。

错误码指标的全称是 `osm_error_err_code_count`。

> 注意：那些导致进程重启的错误相对应的指标可能不会及时被采集。

## 从 Prometheus 查询指标

### 在开始之前

确保完成了 [OSM 演示][2] 当中的步骤

### 查询请求数量相关的代理指标

1. 确认在集群中，Prometheus 服务是运行的
   - 在 Kubernetes，执行这条命令：`kubectl get svc osm-prometheus -n <osm-namespace>`。
     ![image](https://user-images.githubusercontent.com/59101963/85906800-478b3580-b7c4-11ea-8eb2-63bd83647e5f.png)
   - 注意：`<osm-namespace>`指的是安装了 osm 控制平面的命名空间。
2. 打开 Prometheus 界面
   - 确认在仓库的根路径下，执行这条的命令：`./scripts/port-forward-prometheus.sh`
   - 在浏览器访问这个链接 [http://localhost:7070][5]
3. 执行 Prometheus 查询
   - 在网页顶部的 "Expression" 输入框内，输入语句：`envoy_cluster_upstream_rq_xx{envoy_response_code_class="2"}` 并点击 execute 按钮
   - 这条查询语句会返回成功的 http 请求数量

样例结果如图：
![image](https://user-images.githubusercontent.com/59101963/85906690-f24f2400-b7c3-11ea-89b2-a3c42041c7a0.png)

## 使用 Grafana 可视化指标

### 查看 Grafana 面板的先决条件列表

确保已经完成 [OSM 演示][2] 中的步骤

### 查看 Grafana service to service metrics 面板

1. 确认 Prometheus 服务已经在集群中运行
   - 在 Kubernetes 中，执行这条命令：`kubectl get svc osm-prometheus -n <osm-namespace>`
     ![image](https://user-images.githubusercontent.com/59101963/85906800-478b3580-b7c4-11ea-8eb2-63bd83647e5f.png)
2. 确认 Grafana 服务已经运行在集群中
   - 在 Kubernetes 中，执行这条命令：`kubectl get svc osm-grafana -n <osm-namespace>`
     ![image](https://user-images.githubusercontent.com/59101963/85906847-70abc600-b7c4-11ea-853d-f4c9b188ab9f.png)
3. 打开 Grafana 界面
   - 确保在当前仓库的根路径下，并执行这个脚本：`./scripts/port-forward-grafana.sh`
   - 在浏览器访问这个链接 [http://localhost:3000][4]
4. Grafana 界面需要提供登录鉴权信息，使用如下默认口令：
   - username: admin
   - password: admin
5. 在 Grafana 面板上查看服务间的监控指标
   - 在 Grafana 面板左上角的导航菜单中，可以在 OSM Data Plane 的目录里，切换到 OSM Service to Service 面板
   - 或者在浏览器中访问这个链接：[http://localhost:3000/d/OSMs2sMetrics/osm-service-to-service-metrics?orgId=1][6]

OSM Service to Service Metrics 面板看起来像这样：
![image](https://user-images.githubusercontent.com/59101963/85907233-a604e380-b7c5-11ea-95b5-9190fbc7967f.png)

## OSM Grafana 面板

OSM 提供了一下预制的 Grafana 面板来展示和跟踪 Prometheus 采集的服务相关的信息：

1. OSM 数据平面
   - **OSM Data Plane Performance Metrics**：这个面板供查看 OSM 数据平面的性能表现
     ![image](https://user-images.githubusercontent.com/64559656/138173256-28011b16-cace-4365-b166-db909543472e.png)
   - **OSM Service to Service Metrics**：这个面板供查看选定的源服务和目的服务之间的流量指标
     ![image](https://user-images.githubusercontent.com/64559656/141853912-10ec3767-3d5b-40e8-8f13-d39a32980183.png)
   - **OSM Pod to Service Metrics**：这个面板供查看与一个 pod 相连接的所有服务的流量指标
     ![image](https://user-images.githubusercontent.com/64559656/140724337-0568dde0-e6c5-4764-8b6f-c1fcaf144b4e.png)
   - **OSM Workload to Service Metrics**：这个面板提供了与一个工作负载（deployment、replicaSet）相连接的所有服务的流量指标
     ![image](https://user-images.githubusercontent.com/64559656/140724800-8152cb8b-1617-4866-b008-f12c31f702c2.png)
   - **OSM Workload to Workload Metrics**：这个面板展示了网格当中工作负载之间请求的延迟情况
     ![image](https://user-images.githubusercontent.com/64559656/140718968-b3999e30-e6d1-4d95-b07b-0043595aca71.png)

2. OSM 控制平面
   - **OSM Control Plane Metrics**：这个面板提供了选定的服务到 OSM 控制平面的流量指标
     ![image](https://user-images.githubusercontent.com/64559656/138173115-0a012450-0d91-449d-9c09-975b68fde03d.png)
   - **Mesh and Envoy Details**：这个面板供查看 OSM 控制平面的性能表现和工作状态
     ![image](https://user-images.githubusercontent.com/64559656/141852750-61da99ac-a431-4251-bd97-8aa4601232c3.png)

[1]: https://prometheus.io/docs/introduction/overview/
[2]: https://github.com/openservicemesh/osm/blob/{{< param osm_branch >}}/demo/README.md
[3]: https://grafana.com/docs/grafana/latest/getting-started/#what-is-grafana
[4]: http://localhost:3000
[5]: http://localhost:7070
[6]: http://localhost:3000/d/OSMs2sMetrics/osm-service-to-service-metrics?orgId=1
[7]: https://github.com/servicemeshinterface/smi-spec/blob/master/apis/traffic-metrics/v1alpha1/traffic-metrics.md
[8]: https://grafana.com/oss/grafana/

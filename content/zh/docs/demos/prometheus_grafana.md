---
title: "将 OSM 与 Prometheus 和 Grafana 集成"
description: "描述如何使用自维护的 Prometheus 和 Grafana 配置 OSM 特定的配置和仪表板"
type: docs
weight: 25
---

# 将 OSM 与自维护的 Prometheus 和 Grafana 集成

The following article shows you how to create an example bring your own (BYO) Prometheus and Grafana stack on your cluster and configure that stack for observability and monitoring of OSM. For an example using an automatic provisioning of a Prometheus and Grafana stack with OSM, see the [Observability](https://docs.openservicemesh.io/docs/getting_started/observability/) getting started guide.
本文为你展示如何在集群中创建自维护的 Prometheus 和 Grafana ，并为其配置以实现对 OSM 的可观测性和监控。有关使用 OSM 自动配置 Prometheus 和 Grafana 的示例，请参阅 [Observability](https://docs.openservicemesh.io/docs/getting_started/observability/)  入门指南。

> 重要提示：本文创建的配置不应在生产环境中使用。对于生产级部署，请参阅 [Prometheus Operator](https://github.com/prometheus-operator/prometheus-operator/blob/master/Documentation/user-guides/getting-started.md) 和 [Deploy Grafana in Kubernetes](https://grafana.com/docs/grafana/latest/installation/kubernetes/)。

## 先决条件

- Kubernetes cluster running Kubernetes {{< param min_k8s_version >}} or greater.
- OSM installed on the Kubernetes cluster.
- `kubectl` installed and access to the cluster's API server.
- `osm` CLI installed.
- `helm` CLI installed.
- Kubernetes 集群版本 {{< param min_k8s_version >}} 或者更高。
- 集群中已安装 OSM。
- 已安装 `kubectl` 用于访问 API server 。
- 已安装 `osm` 命令行工具
- 已安装 `helm` 命令行工具

## 部署示例 Prometheus 实例

在 default 命名空间下使用 `helm` 安装 Prometheus 实例。

```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install stable prometheus-community/prometheus
```

`helm install` 命令的输出中包含了 Prometheus 服务器的 DNS 名字。比如：

```
...
The Prometheus server can be accessed via port 80 on the following DNS name from within your cluster:
stable-prometheus-server.metrics.svc.cluster.local
...
```

记下 DNS 名字，后面会用到。

## 为 OSM 配置 Prometheus

Prometheus 需要配置为对 OSM 端点进行抓取并正确处理 OSM 的标记、重新标记和端点配置。此配置还有助于在后续步骤中配置的 OSM Grafana 仪表板正确显示从 OSM 抓取的数据。

使用 `kubectl get configmap` 来验证 `stable-prometheus-server` configmap 是否已创建。 例如：

```
$ kubectl get configmap

NAME                             DATA   AGE
...
stable-prometheus-alertmanager   1      18m
stable-prometheus-server         5      18m
...
```

使用以下内容创建 `update-prometheus-configmap.yaml`：

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: stable-prometheus-server
data:
  prometheus.yml: |
    global:
      scrape_interval: 10s
      scrape_timeout: 10s
      evaluation_interval: 1m

    scrape_configs:
      - job_name: 'kubernetes-apiservers'
        kubernetes_sd_configs:
        - role: endpoints
        scheme: https
        tls_config:
          ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
          # TODO need to remove this when the CA and SAN match
          insecure_skip_verify: true
        bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
        metric_relabel_configs:
        - source_labels: [__name__]
          regex: '(apiserver_watch_events_total|apiserver_admission_webhook_rejection_count)'
          action: keep
        relabel_configs:
        - source_labels: [__meta_kubernetes_namespace, __meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
          action: keep
          regex: default;kubernetes;https

      - job_name: 'kubernetes-nodes'
        scheme: https
        tls_config:
          ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
        kubernetes_sd_configs:
        - role: node
        relabel_configs:
        - action: labelmap
          regex: __meta_kubernetes_node_label_(.+)
        - target_label: __address__
          replacement: kubernetes.default.svc:443
        - source_labels: [__meta_kubernetes_node_name]
          regex: (.+)
          target_label: __metrics_path__
          replacement: /api/v1/nodes/${1}/proxy/metrics

      - job_name: 'kubernetes-pods'
        kubernetes_sd_configs:
        - role: pod
        metric_relabel_configs:
        - source_labels: [__name__]
          regex: '(envoy_server_live|envoy_cluster_health_check_.*|envoy_cluster_upstream_rq_xx|envoy_cluster_upstream_cx_active|envoy_cluster_upstream_cx_tx_bytes_total|envoy_cluster_upstream_cx_rx_bytes_total|envoy_cluster_upstream_rq_total|envoy_cluster_upstream_cx_destroy_remote_with_active_rq|envoy_cluster_upstream_cx_connect_timeout|envoy_cluster_upstream_cx_destroy_local_with_active_rq|envoy_cluster_upstream_rq_pending_failure_eject|envoy_cluster_upstream_rq_pending_overflow|envoy_cluster_upstream_rq_timeout|envoy_cluster_upstream_rq_rx_reset|^osm.*)'
          action: keep
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
        # for non-ReplicaSets (DaemonSet, StatefulSet)
        # __meta_kubernetes_pod_controller_kind=DaemonSet
        # __meta_kubernetes_pod_controller_name=foo
        # =>
        # workload_kind=DaemonSet
        # workload_name=foo
        - source_labels: [__meta_kubernetes_pod_controller_kind]
          action: replace
          target_label: source_workload_kind
        - source_labels: [__meta_kubernetes_pod_controller_name]
          action: replace
          target_label: source_workload_name
        # for ReplicaSets
        # __meta_kubernetes_pod_controller_kind=ReplicaSet
        # __meta_kubernetes_pod_controller_name=foo-bar-123
        # =>
        # workload_kind=Deployment
        # workload_name=foo-bar
        # deplyment=foo
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

      - job_name: 'smi-metrics'
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
        metric_relabel_configs:
        - source_labels: [__name__]
          regex: 'envoy_.*osm_request_(total|duration_ms_(bucket|count|sum))'
          action: keep
        - source_labels: [__name__]
          action: replace
          regex: envoy_response_code_(\d{3})_source_namespace_.*_source_kind_.*_source_name_.*_source_pod_.*_destination_namespace_.*_destination_kind_.*_destination_name_.*_destination_pod_.*_osm_request_total
          target_label: response_code
        - source_labels: [__name__]
          action: replace
          regex: envoy_response_code_\d{3}_source_namespace_(.*)_source_kind_.*_source_name_.*_source_pod_.*_destination_namespace_.*_destination_kind_.*_destination_name_.*_destination_pod_.*_osm_request_total
          target_label: source_namespace
        - source_labels: [__name__]
          action: replace
          regex: envoy_response_code_\d{3}_source_namespace_.*_source_kind_(.*)_source_name_.*_source_pod_.*_destination_namespace_.*_destination_kind_.*_destination_name_.*_destination_pod_.*_osm_request_total
          target_label: source_kind
        - source_labels: [__name__]
          action: replace
          regex: envoy_response_code_\d{3}_source_namespace_.*_source_kind_.*_source_name_(.*)_source_pod_.*_destination_namespace_.*_destination_kind_.*_destination_name_.*_destination_pod_.*_osm_request_total
          target_label: source_name
        - source_labels: [__name__]
          action: replace
          regex: envoy_response_code_\d{3}_source_namespace_.*_source_kind_.*_source_name_.*_source_pod_(.*)_destination_namespace_.*_destination_kind_.*_destination_name_.*_destination_pod_.*_osm_request_total
          target_label: source_pod
        - source_labels: [__name__]
          action: replace
          regex: envoy_response_code_\d{3}_source_namespace_.*_source_kind_.*_source_name_.*_source_pod_.*_destination_namespace_(.*)_destination_kind_.*_destination_name_.*_destination_pod_.*_osm_request_total
          target_label: destination_namespace
        - source_labels: [__name__]
          action: replace
          regex: envoy_response_code_\d{3}_source_namespace_.*_source_kind_.*_source_name_.*_source_pod_.*_destination_namespace_.*_destination_kind_(.*)_destination_name_.*_destination_pod_.*_osm_request_total
          target_label: destination_kind
        - source_labels: [__name__]
          action: replace
          regex: envoy_response_code_\d{3}_source_namespace_.*_source_kind_.*_source_name_.*_source_pod_.*_destination_namespace_.*_destination_kind_.*_destination_name_(.*)_destination_pod_.*_osm_request_total
          target_label: destination_name
        - source_labels: [__name__]
          action: replace
          regex: envoy_response_code_\d{3}_source_namespace_.*_source_kind_.*_source_name_.*_source_pod_.*_destination_namespace_.*_destination_kind_.*_destination_name_.*_destination_pod_(.*)_osm_request_total
          target_label: destination_pod
        - source_labels: [__name__]
          action: replace
          regex: .*(osm_request_total)
          target_label: __name__

        - source_labels: [__name__]
          action: replace
          regex: envoy_source_namespace_(.*)_source_kind_.*_source_name_.*_source_pod_.*_destination_namespace_.*_destination_kind_.*_destination_name_.*_destination_pod_.*_osm_request_duration_ms_(bucket|sum|count)
          target_label: source_namespace
        - source_labels: [__name__]
          action: replace
          regex: envoy_source_namespace_.*_source_kind_(.*)_source_name_.*_source_pod_.*_destination_namespace_.*_destination_kind_.*_destination_name_.*_destination_pod_.*_osm_request_duration_ms_(bucket|sum|count)
          target_label: source_kind
        - source_labels: [__name__]
          action: replace
          regex: envoy_source_namespace_.*_source_kind_.*_source_name_(.*)_source_pod_.*_destination_namespace_.*_destination_kind_.*_destination_name_.*_destination_pod_.*_osm_request_duration_ms_(bucket|sum|count)
          target_label: source_name
        - source_labels: [__name__]
          action: replace
          regex: envoy_source_namespace_.*_source_kind_.*_source_name_.*_source_pod_(.*)_destination_namespace_.*_destination_kind_.*_destination_name_.*_destination_pod_.*_osm_request_duration_ms_(bucket|sum|count)
          target_label: source_pod
        - source_labels: [__name__]
          action: replace
          regex: envoy_source_namespace_.*_source_kind_.*_source_name_.*_source_pod_.*_destination_namespace_(.*)_destination_kind_.*_destination_name_.*_destination_pod_.*_osm_request_duration_ms_(bucket|sum|count)
          target_label: destination_namespace
        - source_labels: [__name__]
          action: replace
          regex: envoy_source_namespace_.*_source_kind_.*_source_name_.*_source_pod_.*_destination_namespace_.*_destination_kind_(.*)_destination_name_.*_destination_pod_.*_osm_request_duration_ms_(bucket|sum|count)
          target_label: destination_kind
        - source_labels: [__name__]
          action: replace
          regex: envoy_source_namespace_.*_source_kind_.*_source_name_.*_source_pod_.*_destination_namespace_.*_destination_kind_.*_destination_name_(.*)_destination_pod_.*_osm_request_duration_ms_(bucket|sum|count)
          target_label: destination_name
        - source_labels: [__name__]
          action: replace
          regex: envoy_source_namespace_.*_source_kind_.*_source_name_.*_source_pod_.*_destination_namespace_.*_destination_kind_.*_destination_name_.*_destination_pod_(.*)_osm_request_duration_ms_(bucket|sum|count)
          target_label: destination_pod
        - source_labels: [__name__]
          action: replace
          regex: .*(osm_request_duration_ms_(bucket|sum|count))
          target_label: __name__

      - job_name: 'kubernetes-cadvisor'
        scheme: https
        tls_config:
          ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
        kubernetes_sd_configs:
        - role: node
        metric_relabel_configs:
        - source_labels: [__name__]
          regex: '(container_cpu_usage_seconds_total|container_memory_rss)'
          action: keep
        relabel_configs:
        - action: labelmap
          regex: __meta_kubernetes_node_label_(.+)
        - target_label: __address__
          replacement: kubernetes.default.svc:443
        - source_labels: [__meta_kubernetes_node_name]
          regex: (.+)
          target_label: __metrics_path__
          replacement: /api/v1/nodes/${1}/proxy/metrics/cadvisor
```

使用 `kubectl apply` 来更新 Prometheus 服务器 configmap。

```
kubectl apply -f update-prometheus-configmap.yaml
```

验证 Prometheus 是否能够通过使用 `kubectl port-forward` 在 Prometheus 管理应用程序和开发机器之间转发流量来抓取 OSM 网格和 API 端点。

```
export POD_NAME=$(kubectl get pods -l "app=prometheus,component=server" -o jsonpath="{.items[0].metadata.name}")
kubectl port-forward $POD_NAME 9090
```

在 web 浏览器中打开 `http://localhost:9090/targets` 来访问 Prometheus 管理应用，并验证端点是否连接、启动和正在执行抓取。

<p align="center">
  <img src="/docs/images/byo_guide/prom_targets.png" width="100%"/>
</p>
<center><i>Targets with specific relabeling config established by OSM should be "up"</i></center><br>

停止端口转发命令。

## 部署 Grafana 实例

使用 `helm` 部署 Grafana 实例到集群的 default 命名空间中。

```
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
helm install grafana/grafana --generate-name
```

使用 `kubectl get secret` 来显示 Grafana 的管理员密码。

```
export SECRET_NAME=$(kubectl get secret -l "app.kubernetes.io/name=grafana" -o jsonpath="{.items[0].metadata.name}")
kubectl get secret $SECRET_NAME -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
```

使用 `kubectl port-forward` 来转发 Grafana 管理应用和开发机器间的流量。

```
export POD_NAME=$(kubectl get pods -l "app.kubernetes.io/name=grafana" -o jsonpath="{.items[0].metadata.name}")
kubectl port-forward $POD_NAME 3000
```

在 web 浏览器中打开 `http://localhost:3000` 访问 Grafana 的管理程序。使用 `admin` 作为用户名和前面获取到的管理员密码。验证端点已连接、启动和正在执行抓取。

在管理程序中：

* 选择 `Settings` 然后 `Data Sources`。
* 选择 `Add Data source`。
* 找到 `Prometheus` 数据源并点击 `Select`。
* 输入 DNS 名字，例如在前面拿到的 `stable-prometheus-server.default.svc.cluster.local`。

选择 `Save and Test` 并确保看到 `Data source is working`。

## Importing OSM Dashboards

OSM Dashboards 可通过 [OSM GitHub 存储库](https://github.com/openservicemesh/osm/tree/{{< param osm_branch >}}/charts/osm/grafana/dashboards) 获得，可以在管理应用程序上以 json blobs 方式导入。

要导入仪表盘：
* Hover your cursor over the `+` and select `Import`.
* 鼠标放在 `+` 上并点击 `Import`
* 从 [osm-mesh-envoy-details dashboard](https://raw.githubusercontent.com/openservicemesh/osm/{{< param osm_branch >}}/charts/osm/grafana/dashboards/osm-mesh-envoy-details.json) 复制 JSON 内容并拷贝到 `Import via panel json`。
* 选择 `Load`。
* 选择 `Import`。

确保看到 `Mesh and Envoy Details` 仪表盘创建。

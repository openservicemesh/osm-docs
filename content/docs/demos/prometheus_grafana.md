---
title: "Integrate OSM with Prometheus and Grafana"
description: "Describes how to set up an OSM-specific configuration and dashboards with your own Prometheus and Grafana stack"
type: docs
weight: 25
---

# Integrate OSM with your own Prometheus and Grafana stack

The following article shows you how to create an example bring your own (BYO) Prometheus and Grafana stack on your cluster and configure that stack for observability and monitoring of OSM. For an example using an automatic provisioning of a Prometheus and Grafana stack with OSM, see the [Observability](https://docs.openservicemesh.io/docs/getting_started/observability/) getting started guide.

> IMPORTANT: The configuration created in this article should not be used in production environments. For production-grade deployments, see [Prometheus Operator](https://github.com/prometheus-operator/prometheus-operator/blob/master/Documentation/user-guides/getting-started.md) and [Deploy Grafana in Kubernetes](https://grafana.com/docs/grafana/latest/installation/kubernetes/).

## Prerequisites

- Kubernetes cluster running Kubernetes {{< param min_k8s_version >}} or greater.
- OSM installed on the Kubernetes cluster.
- `kubectl` installed and access to the cluster's API server.
- `osm` CLI installed.
- `helm` CLI installed.

## Deploy an example Prometheus instance

Use `helm` to deploy a Prometheus instance to your cluster in the default namespace.

```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install stable prometheus-community/prometheus
```

The output of the `helm install` command contains the DNS name of the Prometheus server. For example:

```
...
The Prometheus server can be accessed via port 80 on the following DNS name from within your cluster:
stable-prometheus-server.metrics.svc.cluster.local
...
```

Record this DNS name for use in a later step.

## Configure Prometheus for OSM

Prometheus needs to be configured to scape the OSM endpoints and properly handle OSM's labeling, relabelling, and endpoint configuration. This configuration also helps the OSM Grafana dashboards, which are configured in a later step, properly display the data scraped from OSM.

Use `kubectl get configmap` to verify the `stable-prometheus-sever` configmap has been created. For example:

```
$ kubectl get configmap

NAME                             DATA   AGE
...
stable-prometheus-alertmanager   1      18m
stable-prometheus-server         5      18m
...
```

Create `update-prometheus-configmap.yaml` with the following:

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

Use `kubectl apply` to update the Prometheus server configmap.

```
kubectl apply -f update-prometheus-configmap.yaml
```

Verify Prometheus is able to scrape the OSM mesh and API endpoints by using `kubectl port-forward` to forward the traffic between the Prometheus management application and your development computer.

```
export POD_NAME=$(kubectl get pods -l "app=prometheus,component=server" -o jsonpath="{.items[0].metadata.name}")
kubectl port-forward $POD_NAME 9090
```

Open a web browser to `http://localhost:9090/targets` to access the Prometheus management application and verify the endpoints are connected, up, and scrapping is running.

<p align="center">
  <img src="/docs/images/byo_guide/prom_targets.png" width="100%"/>
</p>
<center><i>Targets with specific relabeling config established by OSM should be "up"</i></center><br>

Stop the port-forwarding command.

## Deploying a Grafana Instance

Use `helm` to deploy a Grafana instance to your cluster in the default namespace.

```
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
helm install grafana/grafana --generate-name
```

Use `kubectl get secret` to display the administrator password for Grafana.

```
export SECRET_NAME=$(kubectl get secret -l "app.kubernetes.io/name=grafana" -o jsonpath="{.items[0].metadata.name}")
kubectl get secret $SECRET_NAME -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
```

Use `kubectl port-forward` to forward the traffic between the Grafana's management application and your development computer.

```
export POD_NAME=$(kubectl get pods -l "app.kubernetes.io/name=grafana" -o jsonpath="{.items[0].metadata.name}")
kubectl port-forward $POD_NAME 3000
```

Open a web browser to `http://localhost:3000` to access the Grafana's management application. Use `admin` as the username and administrator password from the previous step. and verify the endpoints are connected, up, and scrapping is running.

From the management application:

* Select `Settings` then `Data Sources`.
* Select `Add data source`.
* Find the `Prometheus` data source and select `Select`.
* Enter the DNS name, for example `stable-prometheus-server.default.svc.cluster.local`, from the earlier step in `URL`.

Select `Save and Test` and confirm you see `Data source is working`.

## Importing OSM Dashboards

OSM Dashboards are available from the following sources:
- [OSM GitHub repository](https://github.com/openservicemesh/osm/tree/{{< param osm_branch >}}/charts/osm/grafana/dashboards)
- [Online at Grafana.com](https://grafana.com/orgs/openservicemesh/dashboards)

To import a dashboard:
* Hover your cursor over the `+` and select `Import`.
* Copy the JSON from the [osm-mesh-envoy-details dashboard](https://raw.githubusercontent.com/openservicemesh/osm/{{< param osm_branch >}}/charts/osm/grafana/dashboards/osm-mesh-envoy-details.json) and paste it in `Import via panel json`. Alternatively, you can enter the dashboard's ID from `Grafana.com`. For example, the `osm-mesh-envoy-details` dashboard uses id `14145`.
* Select `Load`.
* Select `Import`.

Confirm you see a `Mesh and Envoy Details` dashboard created.

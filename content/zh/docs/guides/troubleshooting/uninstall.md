---
title: "卸载"
description: "卸载 OSM 网格的故障排查"
aliases: ["/docs/troubleshooting/uninstall"]
type: docs
weight: 20
---

# 卸载 OSM 网格的故障排查

不论何种原因导致的 `osm uninstall mesh`（如 [卸载指南](/docs/guides/uninstall/) 中所述）失败，可以按照下面的操作手动删除 OSM 资源。

为网格设置环境变量：

```console
export osm_namespace=osm-system # Replace osm-system with the namespace where OSM is installed
export mesh_name=osm # Replace osm with the OSM mesh name
export osm_version=<osm version>
export osm_ca_bundle=<osm ca bundle>
```

删除 OSM 控制平面部署：

```console
kubectl delete deployment -n $osm_namespace osm-bootstrap
kubectl delete deployment -n $osm_namespace osm-controller
kubectl delete deployment -n $osm_namespace osm-injector
```

如果 OSM 与 Prometheus、Grafana 或 Jaeger 一起安装，请删除这些部署：

```console
kubectl delete deployment -n $osm_namespace osm-prometheus
kubectl delete deployment -n $osm_namespace osm-grafana
kubectl delete deployment -n $osm_namespace jaeger
```

如果 OSM 与 OSM 多集群网关一起安装，请运行以下命令将其删除：

```console
kubectl delete deployment -n $osm_namespace osm-multicluster-gateway
```

删除 OSM secrets、meshconfig 和 webhook 配置：
> 警告：确保集群中没有资源依赖于以下资源，然后再继续。

```console
kubectl delete secret -n $osm_namespace $osm_ca_bundle mutating-webhook-cert-secret validating-webhook-cert-secret crd-converter-cert-secret
kubectl delete meshconfig -n $osm_namespace osm-mesh-config
kubectl delete mutatingwebhookconfiguration -l app.kubernetes.io/name=openservicemesh.io,app.kubernetes.io/instance=$mesh_name,app.kubernetes.io/version=$osm_version,app=osm-injector
kubectl delete validatingwebhookconfiguration -l app.kubernetes.io/name=openservicemesh.io,app.kubernetes.io/instance=mesh_name,app.kubernetes.io/version=$osm_version,app=osm-controller
```

要从集群中删除 OSM 和 SMI CRD，请运行以下命令。
> 警告：删除 CRD 将导致与该 CRD 对应的所有自定义资源也被删除。
```console
kubectl delete crd meshconfigs.config.openservicemesh.io
kubectl delete crd multiclusterservices.config.openservicemesh.io
kubectl delete crd egresses.policy.openservicemesh.io
kubectl delete crd ingressbackends.policy.openservicemesh.io
kubectl delete crd httproutegroups.specs.smi-spec.io
kubectl delete crd tcproutes.specs.smi-spec.io
kubectl delete crd traffictargets.access.smi-spec.io
kubectl delete crd trafficsplits.split.smi-spec.io
```
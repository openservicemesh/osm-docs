---
title: "网格配置"
description: "OSM MeshConfig"
type: docs
aliases: ["/docs/osm_mesh_config/"]
weight: 7
---

# OSM MeshConfig
OSM 部署一个 MeshConfig 资源 `osm-mesh-config` 作为它的控制平面(同 OSM 控制器 Pod 的在同一命名空间) 的一部分，其能够被网格所有者/操作员在任意时刻更新。这个 MeshConfig 的目的是提供一种能够更新所需的网格配置的能力给网格所有者/运维人员。

在安装的时候，OSM MeshConfig 从一个现成的 MeshConfig (`preset-mesh-config`) 来部署，其能够在 [charts/osm/templates](https://github.com/openservicemesh/osm/blob/{{< param osm_branch >}}/charts/osm/templates/preset-mesh-config.yaml) 里面找到。

首先，设置一个环境变量来引用 OSM 被安装所在的命名空间。
```bash
export osm_namespace=osm-system # Replace osm-system with the namespace where OSM is installed
```

要在 CLI 里面查阅您的 `osm-mesh-config`，请使用 `kubectl get` 命令。

```bash
kubectl get meshconfig osm-mesh-config -n "$osm_namespace" -o yaml
```

*注意：在 MeshConfig `osm-mesh-config` 里面的值被持续更新。*

## 配置 OSM MeshConfig
### Kubectl 补丁命令
修改 `osm-mesh-config`，可以使用 `kubectl patch` 命令。
```bash
kubectl patch meshconfig osm-mesh-config -n "$osm_namespace" -p '{"spec":{"traffic":{"enableEgress":true}}}'  --type=merge
```
参考 [Config API reference](/docs/apidocs/config/v1alpha1) 以获取更多信息。

如果一个不正确的值被使用了，在 [MeshConfig CRD](https://github.com/openservicemesh/osm/blob/{{< param osm_branch >}}/charts/osm/crds/meshconfig.yaml) 上的验证将阻止修改并给出一个错误信息来解释为什么这个值是无效的。
例如，下面的命令显示了如果我们用 `enableEgress` 给一个非布尔值打补丁会发生什么。
```bash
kubectl patch meshconfig osm-mesh-config -n "$osm_namespace" -p '{"spec":{"traffic":{"enableEgress":"no"}}}'  --type=merge
# Validations on the CRD will deny this change
The MeshConfig "osm-mesh-config" is invalid: spec.traffic.enableEgress: Invalid value: "string": spec.traffic.enableEgress in body must be of type boolean: "string"
```
#### 给每一个键类型的 kubectl 补丁命令 

> 注意：`<osm-namespace>` 引用了 OSM Control Plane 被安装所在的命名空间。默认的，OSM 的命名空间是 `osm-system`。

| 键                                            | 类型   | 默认值                                | Kubectl 补丁命令例子                                                                                                                                                 |
| ---------------------------------------------- | ------ | -------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| spec.traffic.enableEgress                      | bool   | `false`                                      | `kubectl patch meshconfig osm-mesh-config -n $osm_namespace -p '{"spec":{"traffic":{"enableEgress":true}}}'  --type=merge`                                                     |
| spec.traffic.enablePermissiveTrafficPolicyMode | bool   | `false`                                      | `kubectl patch meshconfig osm-mesh-config -n $osm_namespace -p '{"spec":{"traffic":{"enablePermissiveTrafficPolicyMode":true}}}'  --type=merge`                                |
| spec.traffic.useHTTPSIngress                   | bool   | `false`                                      | `kubectl patch meshconfig osm-mesh-config -n $osm_namespace -p '{"spec":{"traffic":{"useHTTPSIngress":true}}}'  --type=merge`                                                  |
| spec.traffic.outboundPortExclusionList         | array  | `[]`                                         | `kubectl patch meshconfig osm-mesh-config -n $osm_namespace -p '{"spec":{"traffic":{"outboundPortExclusionList":6379,8080}}}'  --type=merge`                                   |
| spec.traffic.outboundIPRangeExclusionList      | array  | `[]`                                         | `kubectl patch meshconfig osm-mesh-config -n $osm_namespace -p '{"spec":{"traffic":{"outboundIPRangeExclusionList":"10.0.0.0/32,1.1.1.1/24"}}}'  --type=merge`                 |
| spec.certificate.serviceCertValidityDuration   | string | `"24h"`                                      | `kubectl patch meshconfig osm-mesh-config -n $osm_namespace -p '{"spec":{"certificate":{"serviceCertValidityDuration":"24h"}}}'  --type=merge`                                 |
| spec.observability.enableDebugServer           | bool   | `false`                                      | `kubectl patch meshconfig osm-mesh-config -n $osm_namespace -p '{"spec":{"observability":{"serviceCertValidityDuration":true}}}'  --type=merge`                                |
| spec.observability.tracing.enable              | bool   | `"jaeger.<osm-namespace>.svc.cluster.local"` | `kubectl patch meshconfig osm-mesh-config -n $osm_namespace -p '{"spec":{"observability":{"tracing":{"address": "jaeger.<osm-namespace>.svc.cluster.local"}}}}'  --type=merge` |
| spec.observability.tracing.address             | string | `"/api/v2/spans"`                            | `kubectl patch meshconfig osm-mesh-config -n $osm_namespace -p '{"spec":{"observability":{"tracing":{"endpoint":"/api/v2/spans"}}}}'  --type=merge' --type=merge`              |
| spec.observability.tracing.endpoint            | string | `false`                                      | `kubectl patch meshconfig osm-mesh-config -n $osm_namespace -p '{"spec":{"observability":{"tracing":{"enable":true}}}}'  --type=merge`                                         |
| spec.observability.tracing.port                | int    | `9411`                                       | `kubectl patch meshconfig osm-mesh-config -n $osm_namespace -p '{"spec":{"observability":{"tracing":{"port":9411}}}}'  --type=merge`                                           |
| spec.sidecar.enablePrivilegedInitContainer     | bool   | `false`                                      | `kubectl patch meshconfig osm-mesh-config -n $osm_namespace -p '{"spec":{"sidecar":{"enablePrivilegedInitContainer":true}}}'  --type=merge`                                    |
| spec.sidecar.logLevel                          | string | `"error"`                                    | `kubectl patch meshconfig osm-mesh-config -n $osm_namespace -p '{"spec":{"sidecar":{"logLevel":"error"}}}'  --type=merge`                                                      |
| spec.sidecar.maxDataPlaneConnections           | int    | `0`                                          | `kubectl patch meshconfig osm-mesh-config -n $osm_namespace -p '{"spec":{"sidecar":{"maxDataPlaneConnections":"error"}}}'  --type=merge`                                       |
| spec.sidecar.envoyImage                        | string | `"envoyproxy/envoy-alpine:v1.17.2"`          | `kubectl patch meshconfig osm-mesh-config -n $osm_namespace -p '{"spec":{"sidecar":{"envoyImage":"envoyproxy/envoy-alpine:v1.17.2"}}}'  --type=merge`                          |
| spec.sidecar.initContainerImage                | string | `"openservicemesh/init:v0.9.2"`              | `kubectl patch meshconfig osm-mesh-config -n $osm_namespace -p '{"spec":{"sidecar":{"initContainerImage":"openservicemesh/init:v0.9.2"}}}' --type=merge`                       |
| spec.sidecar.configResyncInterval              | string | `"0s"`                                       | `kubectl patch meshconfig osm-mesh-config -n $osm_namespace -p '{"spec":{"sidecar":{"configResyncInterval":"30s"}}}'  --type=merge`                                            |

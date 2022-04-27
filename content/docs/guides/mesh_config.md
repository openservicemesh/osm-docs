---
title: "Mesh configuration"
description: "OSM MeshConfig"
type: docs
aliases: ["/docs/osm_mesh_config/"]
weight: 7
---

# OSM MeshConfig
OSM deploys a MeshConfig resource `osm-mesh-config` as a part of its control plane (in the same namespace as that of the osm-controller pod) which can be updated by the mesh owner/operator at any time. The purpose of this MeshConfig is to provide the mesh owner/operator the ability to update some of the mesh configurations based on their needs.

At the time of install, the OSM MeshConfig is deployed from a preset MeshConfig (`preset-mesh-config`) which can be found under [charts/osm/templates](https://github.com/openservicemesh/osm/blob/{{< param osm_branch >}}/charts/osm/templates/preset-mesh-config.yaml).

First, set an environment variable to refer to the namespace where osm was installed.
```bash
export osm_namespace=osm-system # Replace osm-system with the namespace where OSM is installed
```

To view your `osm-mesh-config` in CLI use the `kubectl get` command.

```bash
kubectl get meshconfig osm-mesh-config -n "$osm_namespace" -o yaml
```

*Note: Values in the MeshConfig `osm-mesh-config` are persisted across upgrades.*

## Configure OSM MeshConfig
### Kubectl Patch Command
Changes to `osm-mesh-config` can be made using the `kubectl patch` command.
```bash
kubectl patch meshconfig osm-mesh-config -n "$osm_namespace" -p '{"spec":{"traffic":{"enableEgress":true}}}'  --type=merge
```
Refer to the [Config API reference](/docs/apidocs/config/v1alpha1) for more information.

If an incorrect value is used, validations on the [MeshConfig CRD](https://github.com/openservicemesh/osm/blob/{{< param osm_branch >}}/charts/osm/crds/meshconfig.yaml) will prevent the change with an error message explaining why the value is invalid.
For example, the below command shows what happens if we patch `enableEgress` to a non-boolean value.
```bash
kubectl patch meshconfig osm-mesh-config -n "$osm_namespace" -p '{"spec":{"traffic":{"enableEgress":"no"}}}'  --type=merge
# Validations on the CRD will deny this change
The MeshConfig "osm-mesh-config" is invalid: spec.traffic.enableEgress: Invalid value: "string": spec.traffic.enableEgress in body must be of type boolean: "string"
```
#### Kubectl Patch Command for Each Key Type

> Note: `<osm-namespace>` refers to the namespace where the osm control plane is installed. By default, the osm namespace is `osm-system`.

| Key                                            | Type   | Default Value                                | Kubectl Patch Command Examples                                                                                                                                                 |
| ---------------------------------------------- | ------ | -------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| spec.traffic.enableEgress                      | bool   | `false`                                      | `kubectl patch meshconfig osm-mesh-config -n $osm_namespace -p '{"spec":{"traffic":{"enableEgress":true}}}'  --type=merge`                                                     |
| spec.traffic.enablePermissiveTrafficPolicyMode | bool   | `false`                                      | `kubectl patch meshconfig osm-mesh-config -n $osm_namespace -p '{"spec":{"traffic":{"enablePermissiveTrafficPolicyMode":true}}}'  --type=merge`                                |
| spec.traffic.useHTTPSIngress                   | bool   | `false`                                      | `kubectl patch meshconfig osm-mesh-config -n $osm_namespace -p '{"spec":{"traffic":{"useHTTPSIngress":true}}}'  --type=merge`                                                  |
| spec.traffic.outboundPortExclusionList         | array  | `[]`                                         | `kubectl patch meshconfig osm-mesh-config -n $osm_namespace -p '{"spec":{"traffic":{"outboundPortExclusionList":6379,8080}}}'  --type=merge`                                   |
| spec.traffic.outboundIPRangeExclusionList      | array  | `[]`                                         | `kubectl patch meshconfig osm-mesh-config -n $osm_namespace -p '{"spec":{"traffic":{"outboundIPRangeExclusionList":"10.0.0.0/32,1.1.1.1/24"}}}'  --type=merge`                 |
| spec.traffic.networkInterfaceExclusionList     | array  | `[]`                                         | `kubectl patch meshconfig osm-mesh-config -n $osm_namespace -p '{"spec":{"traffic":{"networkInterfaceExclusionList": ["eth0", "net1"]}}}'  --type=merge`                 |
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

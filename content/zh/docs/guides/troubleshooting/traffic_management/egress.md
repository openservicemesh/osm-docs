---
title: "出口故障排查"
description: "出口故障排查指南"
type: docs
---

## 当出口未按预期工作时

### 1. 确认出口已启用

通过验证 `osm-mesh-config` `MeshConfig` 自定义资源中的 `enableEgress` 的值来确认已启用出口。 `osm-mesh-config` 位于命名空间 OSM 控制平面命名空间中（默认情况下为 `osm-system`）。

```console
# Returns true if egress is enabled
$ kubectl get meshconfig osm-mesh-config -n osm-system -o jsonpath='{.spec.traffic.enableEgress}{"\n"}'
true
```

The above command must return a boolean string (`true` or `false`) indicating if egress is enabled.
上述命令必须返回一个布尔字符串（`true`或`false`），表示是否启用了出口。

### 2. 检查 OSM 控制器日志是否有错误

```bash
# When osm-controller is deployed in the osm-system namespace
kubectl logs -n osm-system $(kubectl get pod -n osm-system -l app=osm-controller -o jsonpath='{.items[0].metadata.name}')
```

日志消息中的 `level` 设置为 `error` 来记录错误：
```console
{"level":"error","component":"...","time":"...","file":"...","message":"..."}
```

### 3. 确认 Envoy 配置

Confirm the Envoy proxy configuration on the client has a default egress filter chain on the outbound listener. Refer to the [sample configurations](../../../tasks/traffic_management/egress#envoy-configurations) to verify that the client is configured to have outbound access to external destinations.

确认客户端侧的 Envoy 代理配置在出站监听器上有一个默认的出口过滤器链。请参阅 [示例配置](../../../tasks/traffic_management/egress#envoy-configurations) 以验证客户端是否已配置为具有对外部目标的出站访问权限。
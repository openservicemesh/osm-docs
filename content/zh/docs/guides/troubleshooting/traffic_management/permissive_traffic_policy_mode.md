---
title: "宽松流量策略模式故障排查"
description: "宽松流量策略模式故障排查指南"
type: docs
---

## 当宽松流量策略模式没有按预期工作时

### 1. 确认宽松流量策略模式已开启

在自定义资源 `osm-mesh-config` 中，通过检查 `enablePermissiveTrafficPolicyMode` 配置字段，来确认宽松流量策略模式是否启用。`osm-mesh-config` MeshConfig 位于 OSM 控制平面所在的命名空间（默认为`osm-system`）

```console
# 如果宽松流量策略模式已启用，则返回 true
$ kubectl get meshconfig osm-mesh-config -n osm-system -o jsonpath='{.spec.traffic.enablePermissiveTrafficPolicyMode}{"\n"}'
true
```

上面的命令一定会返回布尔型的字符串（`true` 或者 `false`），表示宽松流量策略模式是否启用

### 2. 检查 OSM 控制器日志中的错误

```bash
# 如果 osm-controller 被部署到 osm-system 命名空间中
kubectl logs -n osm-system $(kubectl get pod -n osm-system -l app=osm-controller -o jsonpath='{.items[0].metadata.name}')
```

日志当中，`level` 字段被设置为 `error` 的消息会记录错误信息
```console
{"level":"error","component":"...","time":"...","file":"...","message":"..."}
```

### 3. 确认 Envoy 的配置

在 Envoy 代理在客户端和服务端 Pod 中的配置，确认客户端是允许访问服务端的。请参考 [配置样例](../../../tasks/traffic_management/permissive_traffic_policy_mode#envoy-configurations) 来确认客户端已经设置了合法访问服务端的路由。

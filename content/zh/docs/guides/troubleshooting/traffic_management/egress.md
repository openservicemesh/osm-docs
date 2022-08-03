---
title: "出口"
description: "出口（外部）流量的故障排查"
type: docs
weight: 4
---

# 出口流量故障排查

## 当出口未按预期工作时

### 1. 确认出口已启用

如果依赖于未知目的地的直通出口功能，通过验证 `osm-mesh-config` `MeshConfig` 自定义资源中的 `enableEgress` 的值来确认已启用全局直通出口。`osm-mesh-config` 位于命名空间 OSM 控制平面命名空间中（默认情况下为 `osm-system`）。

```console
# Returns true if global passthrough egress is enabled
$ kubectl get meshconfig osm-mesh-config -n osm-system -o jsonpath='{.spec.traffic.enableEgress}{"\n"}'
true
```

如果正在使用 [出口策略](/docs/guides/traffic_management/egress/#1-configuring-egress-policies)，确保出口策略特性已启用。

```console
# Returns true if egress policy capability is enabled
$ kubectl get meshconfig osm-mesh-config -n osm-system -o jsonpath='{.spec.featureFlags.enableEgressPolicy}{"\n"}'
true
```

### 2. 检查 OSM 控制器日志是否有错误

```bash
# When osm-controller is deployed in the osm-system namespace
kubectl logs -n osm-system $(kubectl get pod -n osm-system -l app=osm-controller -o jsonpath='{.items[0].metadata.name}')
```

日志消息中的 `level` 设置为 `error` 来记录错误：
```console
{"level":"error","component":"...","time":"...","file":"...","message":"..."}
```

### 3. 当使用出口策略时确认 Envoy 配置

通过 `osm verify connectivity` 命令来验证 pod 能否使用出口策略与外部的主机和端口进行通信。

示例：

验证 `curl` 命名空间下的 pod `curl-7bb5845476-zwxbt` 能否直接发送 HTTPS 流量到外部的 `httpbin.org` 主机上的 `443` 端口：

```console
$ osm verify connectivity --from-pod curl/curl-7bb5845476-zwxbt --to-ext-port 443 --to-ext-host httpbin.org --app-protocol https
---------------------------------------------
[+] Context: Verify if pod "curl/curl-7bb5845476-zwxbt" can access external service on port 443
Status: Success

---------------------------------------------
```

验证 `curl` 命名空间下的 pod `curl-7bb5845476-zwxbt` 能否直接发送 HTTP 流量到外部的 `httpbin.org` 主机上的 ` 80` 端口：

```console
$ osm verify connectivity --from-pod curl/curl-7bb5845476-zwxbt --to-ext-port 80 --to-ext-host httpbin.org --app-protocol http
---------------------------------------------
[+] Context: Verify if pod "curl/curl-7bb5845476-zwxbt" can access external service on port 80
Status: Success

---------------------------------------------
```

验证通过，输出中的 `Status` 字段会显示 `Success`。
---
title: "入口故障排查"
description: "入口故障排查指南"
type: docs
---

## 当 Ingress 未按预期工作时

### 1. 确认按预期设置全局入口配置。

```console
# Returns true if HTTPS ingress is enabled
$ kubectl get meshconfig osm-mesh-config -n osm-system -o jsonpath='{.spec.traffic.useHTTPSIngress}{"\n"}'
false
```

如果命令的输出为 `false` ，则表示启用了 HTTP 入口并禁用了 HTTPS 入口。要禁用 HTTP 入口并启用 HTTPS 入口，请使用以下命令：

```bash
# Replace osm-system with osm-controller's namespace if using a non default namespace
kubectl patch meshconfig osm-mesh-config -n osm-system -p '{"spec":{"traffic":{"useHTTPSIngress":true}}}'  --type=merge
```

同样，要启用 HTTP 入口并禁用 HTTPS 入口，执行：

```bash
# Replace osm-system with osm-controller's namespace if using a non default namespace
kubectl patch meshconfig osm-mesh-config -n osm-system -p '{"spec":{"traffic":{"useHTTPSIngress":false}}}'  --type=merge
```

### 2. 检查 OSM 控制器日志是否有错误

```bash
# When osm-controller is deployed in the osm-system namespace
kubectl logs -n osm-system $(kubectl get pod -n osm-system -l app=osm-controller -o jsonpath='{.items[0].metadata.name}')
```

日志消息设置的 `level` 设置为 `error` 时错误信息会被记录下来：

```console
{"level":"error","component":"...","time":"...","file":"...","message":"..."}
```

### 3. 确认入口资源已经部署成功

```bash
kubectl get ingress <ingress-name> -n <ingress-namespace>
```

---
title: "外部授权"
description: "通过 MeshConfig 配置外部授权。"
type: docs
weight: 1
---
# 外部授权

## 概述

外部授权允许将传入 HTTP 请求的授权卸载到可能不在代理上下文中运行的远程端点。

支持外部授权的代理将针对每个请求向远程端点发起权限检查。由于系统可能拥有大量 RPS，外部授权有许多可调整的设置，以允许或多或少的控制，但会牺牲性能（仅发送 HTTP 有效负载或标头、超时、授权无法回复时的默认行为等等）。

OSM 允许通过 OSM MeshConfig 来配置 Envoy 的 [外部授权扩展](https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_filters/ext_authz_filter) 

## 局限性

目前，授权过滤不能动态配置为安装在用户定义的代理子集或代理组中，而是在启用时全局应用于网格中的所有服务。

类似地，过滤方向将静态应用于网格内的 `inbound` 和 `outbound` 连接，启用后会影响对网格中的任何服务或应用程序发出的所有 HTTP 请求。


## OSM 与 OPA 插件外部授权演练

以下部分将记录如何结合 `opa-envoy-plugin` 配置外部授权。

我们强烈建议先阅读 [OPA envoy 插件](https://github.com/open-policy-agent/opa-envoy-plugin) 的文档，以进一步了解其配置选项和部署模型。

以下示例使用单个远程（通过网络）端点来验证所有流量。不建议将此配置用于生产部署。

- 首先，从部署 OSM 的 Demo 开始。我们将使用此示例部署来测试外部授权功能。请参考 [OSM的自动化Demo](https://github.com/openservicemesh/osm/tree/{{< param osm_branch >}}/demo#how-to-run-the-osm-automated-demo)并按照说明操作。

```
# Assuming OSM repo is available
cd <PATH_TO_OSM_REPO>
demo/run-osm-demo.sh  # wait for all services to come up
```

- 当 OSM 的演示启动并运行时，继续部署 `opa-envoy-plugin`。OSM 提供了 [整理好的独立 opa-envoy-plugin 部署 chart](https://raw.githubusercontent.com/openservicemesh/osm-docs/{{< param osm_branch >}}/manifests/opa/deploy-opa-envoy.yaml），它通过服务公开 `opa-envoy-plugin` 的 gRPC 端口（默认为 `9191`）。这是 OSM 在启用外部授权时将配置代理的端点。下面的代码片段创建了一个 `opa` 命名空间，并在其中部署了 `opa-envoy-plugin`，并使用了最少的 deny-all 配置：

```
kubectl create namespace opa
kubectl apply -f https://raw.githubusercontent.com/openservicemesh/osm-docs/{{< param osm_branch >}}/manifests/opa/deploy-opa-envoy.yaml
```

- 一旦 OSM 示例启动并运行，修改 OSM MeshConfig  添加网格的外部授权。为此，配置 `inboundExternalAuthorization` 指向远程外部授权端点，如下所示：

```
kubectl edit meshconfig osm-mesh-config -n osm-system
## <scroll to the following section>
...
inboundExternalAuthorization:
      enable: true
      address: opa.opa.svc.cluster.local
      port: 9191
      failureModeAllow: false
      statPrefix: inboundExtAuthz
      timeout: 1s
...
```

- 这一步执行完，OSM 应该配置所有代理依赖外部授权服务用于授权决策。默认情况下，`opa-envoy-plugin` 提供的配置会拒绝所有访问网格内服务的请求。可以通过检查网格中的任一服务的日志来验证，应该有 `403 Forbidden`：
```
kubectl logs <bookbuyer_pod> -n bookbuyer bookbuyer
```
```
...
--- bookbuyer:[ 8 ] -----------------------------------------

Fetching http://bookstore.bookstore:14001/books-bought
Request Headers: map[Client-App:[bookbuyer] User-Agent:[Go-http-client/1.1]]
Identity: n/a
Booksbought: n/a
Server: envoy
Date: Tue, 04 May 2021 01:20:39 GMT
Status: 403 Forbidden
ERROR: response code for "http://bookstore.bookstore:14001/books-bought" is 403;  expected 200
...
```

- 要进一步验证外部授权是否有效，请验证 `opa-envoy-plugin` 实例的日志。实例的日志应该包含 json blob，它记录收到的每个请求的评估 - 键值`result`应该出现在由提供给 `opa-envoy-plugin` 实例的配置所采取的授权结果中：
```
kubectl logs <opa_pod_name> -n opa
...
{"decision_id":"1df154b5-658a-47bf-ac18-be52998605da"
...
"result":false,   // resulting decision
...
"time":"2021-05-04T01:21:18Z","timestamp":"2021-05-04T01:21:18.1971808Z","type":"openpolicyagent.org/decision_logs"
}
```

- 验证上一步后，继续更改 OPA 策略配置以默认授权所有流量：

```
kubectl edit configmap opa-policy -n opa
...
...
default allow = false
```
将上一行值更改为 `true`：
```
default allow = true
```

- 最后，重新启动 `opa-envoy-plugin`。这是一个必要的步骤，因为此 deployment 的配置是作为配置推送的，而不是应用程序拉取它。
```
kubectl rollout restart deployment opa -n opa
```

- 验证对`bookbuyer`服务的请求现在是否已获得授权：
```
--- bookbuyer:[ 2663 ] -----------------------------------------

Fetching http://bookstore.bookstore:14001/books-bought
Request Headers: map[Client-App:[bookbuyer] User-Agent:[Go-http-client/1.1]]
Identity: bookstore-v1
Booksbought: 1087
Server: envoy
Date: Tue, 04 May 2021 02:00:46 GMT
Status: 200 OK
MAESTRO! THIS TEST SUCCEEDED!

Fetching http://bookstore.bookstore:14001/buy-a-book/new
Request Headers: map[]
Identity: bookstore-v1
Booksbought: 1088
Server: envoy
Date: Tue, 04 May 2021 02:00:47 GMT
Status: 200 OK
ESC[90m2:00AMESC[0m ESC[32mINFESC[0m BooksCountV1=21490056 ESC[36mcomponent=ESC[0mdemo ESC[36mfile=ESC[0mbooks.go:167
MAESTRO! THIS TEST SUCCEEDED!
```

可以重新验证 OPA 插件的输出，以确保为每个请求做出策略决策：
```
{"decision_id":"3f29d449-7f71-4721-b93c-ad7d375e0f80",
...
"result":true,
...
"time":"2021-05-04T02:01:35Z","timestamp":"2021-05-04T02:01:35.816768454Z","type":"openpolicyagent.org/decision_logs"}
```

此示例规则展示了如何使用 OPA 插件对网络流量实施授权策略，但不适用于生产部署。例如，以这种方式对每个 HTTP 调用执行授权会增加大量开销，使用 OPA 注入器可以减少开销。有关使用 OPA 的更多信息，请参阅 [Open Policy Agent 官方文档](https://www.openpolicyagent.org/docs/latest/)。
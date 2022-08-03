---
title: "重试"
description: "实施重试以处理瞬态故障"
type: docs
weight: 11
---

# 重试

重试是一种弹性模式，它使应用程序能够屏蔽客户的瞬态问题。这是通过重试因临时故障（例如 Pod 正在启动）而失败的请求来完成的。本指南描述了如何在 OSM 中实现重试策略。

## 配置重试

OSM 使用其 [Retry policy API][1] 允许重试从指定源 (ServiceAccount) 到一个或多个目的地 (Service) 的流量。重试仅适用于 HTTP 流量。使用 [Envoy 的重试请求头](https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_filters/router_filter#config-http-filters-router)，OSM 可以为参与网格的应用程序实现重试 .

支持以下重试配置：

- `Per Try Timeout`：重试被视为失败之前允许的时间。默认使用全局路由超时。有关更多信息，请参阅 [Envoy 的文档](https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_filters/router_filter#x-envoy-upstream-rq-per-try-timeout-ms)。

- `Retry Backoff Base Interval`：指数重试回退的基本间隔。Envoy 使用完全抖动的指数退避算法。退避是从范围 [0,(2**N-1)B] 中随机选择的，其中 N 是重试次数，B 是基本间隔。默认值为 `25ms`，最大间隔为基本间隔的 10 倍。参见 [Envoy 的文档](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/route/v3/route_components.proto#envoy-v3-api-field-config-route-v3-retrypolicy-retry-back-off)获取更多信息。

- `Number of Retries`：尝试重试的最大次数。默认值为 `1`。

- `Retry On`：指定何时重试失败请求的策略。可以使用 `,` 分隔列表指定多个策略。有关更多信息以及列表支持的政策。

要了解有关配置重试的更多信息，请参阅 [重试策略演示](/docs/demos/retry_policy) 和 [API 文档][1]。

### 示例

如果 bookbuyer 服务对 bookstore-v1 服务或 bookstore-v2 服务的请求收到状态码为 5xx 的响应，则 bookbuyer 将重试该请求 3 次。如果尝试重试的时间超过 3 秒，则认为尝试失败。每次重试在尝试 [如上计算](#配置重试) 之前都有一个延迟期（退避）。所有重试的退避时间上限为 10 秒。

```yaml
kind: Retry
apiVersion: policy.openservicemesh.io/v1alpha1
metadata:
  name: retry
spec:
  source:
    kind: ServiceAccount
    name: bookbuyer
    namespace: bookbuyer
  destinations:
  - kind: Service
    name: bookstore
    namespace: bookstore-v1
  - kind: Service
    name: bookstore
    namespace: bookstore-v2
  retryPolicy:
    retryOn: "5xx"
    perTryTimeout: 3s
    numRetries: 3
    retryBackoffBaseInterval: 1s
```

如果 bookbuyer 服务对 bookstore-v2 服务的请求收到状态码为 5xx 或可重试的-4xx (409) 的响应，则 bookbuyer 将重试请求 5 次。如果尝试重试的时间超过 4 秒，则认为尝试失败。每次重试在尝试 [如上计算](#配置重试) 之前都有一个延迟期（退避）。所有重试的退避时间上限为 20 毫秒。

```yaml
kind: Retry
apiVersion: policy.openservicemesh.io/v1alpha1
metadata:
  name: retry
spec:
  source:
    kind: ServiceAccount
    name: bookbuyer
    namespace: bookbuyer
  destinations:
  - kind: Service
    name: bookstore
    namespace: bookstore-v2
  retryPolicy:
    retryOn: "5xx,retriable-4xx"
    perTryTimeout: 4s
    numRetries: 5
    retryBackoffBaseInterval: 2ms
```

[1]: /docs/api_reference/policy/v1alpha1/#policy.openservicemesh.io/v1alpha1.RetrySpec
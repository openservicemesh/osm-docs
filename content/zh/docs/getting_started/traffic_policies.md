---
title: "配置流量策略"
description: "配置流量在书店应用之间流动"
type: docs
weight: 3
---

# 流量策略

## 流量策略模式

一旦应用们被启动并运行，它们能够使用[宽松流量策略模式](#宽松流量策略模式)或者 [SMI 流量策略模式](#SMI-流量策略模式)来彼此交互。在宽松流量策略模式里，应用服务间的流量被 `osm-controller` 自动配置，被 SMI Traffic Targets 所定义的访问控制策略并不是强制的。在 SMI 策略模式里，默认的所有的流量都被拒绝，除非显性地允许使用一个 SMI 访问和路由策略的组合。

### 流量加密

所有的流量都会通过 mTLS 来加密，这并不管您正在使用的是访问控制策略还是宽松流量策略模式。

### 如何检查流量策略模式

检查是否许可流量策略模式被启用，可以通过对在`osm-mesh-config` `MeshConfig` 资源中的 `enablePermissiveTrafficPolicyMode` 键获取值来看。

```bash
# Replace osm-system with osm-controller's namespace if using a non default namespace
kubectl get meshconfig osm-mesh-config -n osm-system -o jsonpath='{.spec.traffic.enablePermissiveTrafficPolicyMode}{"\n"}'
# Output:
# false: permissive traffic policy mode is disabled, SMI policy mode is enabled
# true: permissive traffic policy mode is enabled, SMI policy mode is disabled
```

接下来的章节演示了使用带[宽松流量策略模式](#宽松流量策略模式)和 [SMI 流量策略模式](#SMI-流量策略模式)的 OSM。

## 宽松流量策略模式

在流量宽松流量策略模式里面，在网格中的应用连接性通过 `osm-controller` 被自动配置。在接下来的方法中，它能够被启用。

1. 使用 `osm` CLI 来当时安装：
  ```bash
  osm install --set=osm.enablePermissiveTrafficPolicy=true
  ```

2. 在控制平面的命名空间通过修正 `osm-mesh-config` 定制资源来后置安装（默认 `osm-system`）。
  ```bash
  kubectl patch meshconfig osm-mesh-config -n osm-system -p '{"spec":{"traffic":{"enablePermissiveTrafficPolicyMode":true}}}'  --type=merge
  ```

### 验证 OSM 处于宽松流量策略模式

开始行动之前，[验证流量策略模式](#验证流量策略模式)并且确保 `enablePermissiveTrafficPolicyMode` 键在 `osm-mesh-config` `MeshConfig` 资源里面被设置成 `true`。参考上面的章节来启用宽松流量策略模式。

在步骤[部署书店应用](#部署书店应用)里面，我们已经部署了应用，并按照宽松流量策略模式来验证流量的流动。我们之前部署的 `bookstore` 服务为了演示目的以 `bookstore-v1` 的身份被编码，同样能够在[部署清单](https://raw.githubusercontent.com/openservicemesh/osm-docs/{{< param osm_branch >}}/manifests/apps/bookstore.yaml)里面被看到。这个身份反映了在 `bookbuyer` 和 `bookthief` UI 中谁的计数在增长，在 `bookstore` UI 中身份的显示。

在 `bookbuyer`，`bookthief` UI 中的计数分别对应了书籍购买和盗窃，而在 `bookstore v1` 中这些都应该在增加：

- [http://localhost:8080](http://localhost:8080) - **bookbuyer**
- [http://localhost:8083](http://localhost:8083) - **bookthief**

在 `bookstore` UI 中的对于书籍销售的计数也应该在增加：

- [http://localhost:8084](http://localhost:8084) - **bookstore**

`bookbuyer` 和 `bookthief` 应用能够分别从新近部署的 `bookstore` 应用购买和盗窃书籍，这是因为宽松流量策略模式被启用，从而允许应用们之间有连接而不需要 SMI 流量控制策略。

这个可以被进一步地演示，通过禁用许可流量策略模式，然后从 `bookstore` 验证书籍购买计数，会发现计数将不再增加：

```bash
kubectl patch meshconfig osm-mesh-config -n osm-system -p '{"spec":{"traffic":{"enablePermissiveTrafficPolicyMode":false}}}'  --type=merge
```

_注意：当您禁用了宽松流量策略模式，SMI 流量访问模式会被默默打开。如果对于书籍的计数正在增加，那么这应该是因为一些 SMI 流量访问策略已经在之前就被设置为允许了。_

## SMI 流量策略模式

SMI 流量策略能够被用在以下情况：

1. SMI 访问控制策略用来授权服务身份彼此间的流量访问
2. SMI 流量规格策略用来定义到相关的访问控制策略的路由规则
3. SMI 流量分拆策略用来基于权重指导客户端流量分到多个后端

接下来的章节描述了如何撬动这些策略的每一个来执行服务网格中流量流动的精细粒度控制，[验证流量策略模式](#验证流量策略模式)并且确保在 `osm-mesh-config` `MeshConfig` 资源中的 `enablePermissiveTrafficPolicyMode` 键值被设置成 `false`。

SMI 流量策略模式能够通过禁用许可流量策略模式来启用：

```bash
kubectl patch meshconfig osm-mesh-config -n osm-system -p '{"spec":{"traffic":{"enablePermissiveTrafficPolicyMode":false}}}'  --type=merge
```

### 部署 SMI 访问控制策略

在这点上，应用们都不能彼此访问，因为没有访问控制策略被使用。确认这一点可以通过验证在 `bookbuyer`，`bookthief`，`bookstore` 和 `bookstore-v2` UI 中的计数器没有一个在增加来完成。

应用 [SMI Traffic Target](https://github.com/servicemeshinterface/smi-spec/blob/v0.6.0/apis/traffic-access/v1alpha2/traffic-access.md) 和 [SMI Traffic Specs](https://github.com/servicemeshinterface/smi-spec/blob/v0.6.0/apis/traffic-specs/v1alpha4/traffic-specs.md) 资源来为应用们的通信定义访问控制和路由策略：

部署 SMI TrafficTarget 和 HTTPRouteGroup 策略：

```bash
kubectl apply -f https://raw.githubusercontent.com/openservicemesh/osm-docs/{{< param osm_branch >}}/manifests/access/traffic-access-v1.yaml
```

对于 `bookbuyer` 和 `bookstore` 应用的计数器现在应该在增长：

- [http://localhost:8080](http://localhost:8080) - **bookbuyer**
- [http://localhost:8084](http://localhost:8084) - **bookstore**

注意，对 `bookthief` 应用的计数器是 _不会_ 增长的：

- [http://localhost:8083](http://localhost:8083) - **bookthief**

这是因为 SMI Traffic Target SMI HTTPRouteGroup 资源部署只允许 `bookbuyer` 和 `bookstore` 通信。

#### 允许 Bookthief 应用访问网格

当前的 Bookthief 应用没有被授权参与服务网格中的通信。我们将更新 TrafficTarget 来允许 `bookthief` 与 `bookstore` 通信。

当前 TrafficTarget 规范没有把 `bookthief` 列于 `spec.sources` 之中：

```yaml
kind: TrafficTarget
apiVersion: access.smi-spec.io/v1alpha3
metadata:
  name: bookstore-v1
  namespace: bookstore
spec:
  destination:
    kind: ServiceAccount
    name: bookstore
    namespace: bookstore
  rules:
  - kind: HTTPRouteGroup
    name: bookstore-service-routes
    matches:
    - buy-a-book
    - books-bought
  sources:
  - kind: ServiceAccount
    name: bookbuyer
    namespace: bookbuyer
```

更新 TrafficTarget 把 `bookthief` 列于 `spec.sources` 之中：

```yaml
kind: TrafficTarget
apiVersion: access.smi-spec.io/v1alpha3
metadata:
 name: bookstore-v1
 namespace: bookstore
spec:
 destination:
   kind: ServiceAccount
   name: bookstore
   namespace: bookstore
 rules:
 - kind: HTTPRouteGroup
   name: bookstore-service-routes
   matches:
   - buy-a-book
   - books-bought
 sources:
 - kind: ServiceAccount
   name: bookbuyer
   namespace: bookbuyer
 - kind: ServiceAccount
   name: bookthief
   namespace: bookthief
```

应用更新的 TrafficTarget：

```bash
kubectl apply -f https://raw.githubusercontent.com/openservicemesh/osm-docs/{{< param osm_branch >}}/manifests/access/traffic-access-v1-allow-bookthief.yaml
```

在 `bookthief` 窗口中的计数器将开始增加。

- [http://localhost:8083](http://localhost:8083) - **bookthief**

应用原来的那个没有列入 bookthief 的 Traffic Target 对象作为被允许的源：

```bash
kubectl apply -f https://raw.githubusercontent.com/openservicemesh/osm-docs/{{< param osm_branch >}}/manifests/access/traffic-access-v1.yaml
```

`bookthief` 窗口的计数器将停止增加。

- [http://localhost:8083](http://localhost:8083) - **bookthief**

## 下一步

学习通过[配置流量拆分](/docs/getting_started/traffic_split/)来如何平衡服务们之间的流量。
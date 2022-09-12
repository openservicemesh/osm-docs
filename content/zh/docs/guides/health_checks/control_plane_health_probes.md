---
title: "OSM 控制平面健康检查"
description: "健康探测工作原理及失败应对"
aliases: "/docs/control_plane_health_probes"
type: "docs"
---

# OSM 控制平面健康探测

OSM 控制平面组件利用健康探测来传递整体状态。健康探测通过 HTTP 端点（Endpoint）实现，使用 HTTP 状态代码表示成功或失败。

Kubernetes 使用这些探针来传递控制平面 Pod 的状态，并自动执行一些行为以提高可用性。关于 Kubernetes 探针的更多细节可以在[这里](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)找到。

## 带探针的 OSM 组件

有健康探针的 OSM 控制平面组件如下：

#### osm-controller

osm-controller 的 9091 端口有以下 HTTP 端点可用：

- `/health/alive`: HTTP 200 响应代码表示 OSM 的聚合发现服务（ADS）正在运行。无响应则表示该服务尚未运行。

- `/health/ready`: HTTP 200 响应代码表明 ADS 可以接受来自代理的 gRPC 连接。HTTP 503 或无响应表示来自代理的 gRPC 连接将不会成功。

#### osm-injector

osm-injector 上有以下 HTTP 端点，端口为 9090:

- `/healthz`: HTTP 200 响应代码表明注入器（injector）可以注入代理 sidecar 容器。无响应则表示该服务没有正常运行。

## 如何验证 OSM 健康状态？

因为 OSM 的 Kubernetes 资源配置了存活和就绪探测，Kubernetes 会自动轮询 osm-controller 和 osm-injector Pod 上的健康端点。

当存活探测失败时，Kubernetes 将产生一个事件（通过 `kubectl describe pod <pod name>` 可见）并重新启动 Pod。`kubectl describe` 的输出如下：

```shell
...
Events:
  Type     Reason     Age               From               Message
  ----     ------     ----              ----               -------
  Normal   Scheduled  24s               default-scheduler  Successfully assigned osm-system/osm-controller-85fcb445b-fpv8l to osm-control-plane
  Normal   Pulling    23s               kubelet            Pulling image "openservicemesh/osm-controller:v0.8.0"
  Normal   Pulled     23s               kubelet            Successfully pulled image "openservicemesh/osm-controller:v0.8.0" in 562.2444ms
  Normal   Created    1s (x2 over 23s)  kubelet            Created container osm-controller
  Normal   Started    1s (x2 over 23s)  kubelet            Started container osm-controller
  Warning  Unhealthy  1s (x3 over 21s)  kubelet            Liveness probe failed: HTTP probe failed with statuscode: 503
  Normal   Killing    1s                kubelet            Container osm-controller failed liveness probe, will be restarted
```

当就绪探测失败时，Kubernetes 将生成一个事件（通过 `kubectl describe pod <pod name>` 可以看到），并确保服务流量不会路由到这些不健康的 POD 上。一个准备就绪探测失败的 Pod 的 `kubectl describe` 输出如下：

```shell
...
Events:
  Type     Reason     Age               From               Message
  ----     ------     ----              ----               -------
  Normal   Scheduled  36s               default-scheduler  Successfully assigned osm-system/osm-controller-5494bcffb6-tn5jv to osm-control-plane
  Normal   Pulling    36s               kubelet            Pulling image "openservicemesh/osm-controller:latest"
  Normal   Pulled     35s               kubelet            Successfully pulled image "openservicemesh/osm-controller:v0.8.0" in 746.4323ms
  Normal   Created    35s               kubelet            Created container osm-controller
  Normal   Started    35s               kubelet            Started container osm-controller
  Warning  Unhealthy  4s (x3 over 24s)  kubelet            Readiness probe failed: HTTP probe failed with statuscode: 503
```

Pod 的 "状态 "也可看出其尚未可用，在 "kubectl get pod" 输出中可以看到。例如：

```shell
NAME                              READY   STATUS    RESTARTS   AGE
osm-controller-5494bcffb6-tn5jv   0/1     Running   0          26s
```

Pod 的健康探测也可以通过转发 Pod 的必要端口并使用 `curl` 或任何其他 HTTP 客户端发出请求手动调用。例如，为了验证 OSM-controller 的有效性探测，获取 Pod 的名称并转发 9091 端口：

```shell
# Assuming OSM is installed in the osm-system namespace
kubectl port-forward -n osm-system $(kubectl get pods -n osm-system -l app=osm-controller -o jsonpath='{.items[0].metadata.name}') 9091
```

然后，在一个单独的终端里，可以使用 `curl` 来检查端点。下面是一个健康的 osm-controller 的例子：

```console
$ curl -i localhost:9091/health/alive
HTTP/1.1 200 OK
Date: Thu, 18 Mar 2021 20:15:29 GMT
Content-Length: 16
Content-Type: text/plain; charset=utf-8

Service is alive
```

## 排错

如果有健康探测持续失败，请执行以下步骤以确定根本原因：

1. 确保不健康的 osm-controller 或 osm-injector Pod 没有运行 Envoy sidecar 容器。

    为了验证 OSM-controller Pod 没有运行 Envoy sidecar 容器，请验证该 Pod 的容器镜像中没有一个是 Envoy 镜像。Envoy 镜像的名字里有 "envoyproxy/envoy"。

    例如，这是一个包含 Envoy 容器的 osm-controller Pod：

    ```console
    $ # 假设 OSM 被安装在 osm-system 命名空间里:
    $ kubectl get pod -n osm-system $(kubectl get pods -n osm-system -l app=osm-controller -o jsonpath='{.items[0].metadata.name}') -o jsonpath='{range .spec.containers[*]}{.image}{"\n"}{end}'
    openservicemesh/osm-controller:v0.8.0
    envoyproxy/envoy-alpine:v1.17.2
    ```

    要验证 OSM-injector Pod 是否在运行 Envoy sidecar 容器，请确认 Pod 的容器镜像中没有 Envoy 镜像。Envoy 镜像的名字里有 "envoyproxy/envoy"。

    例如，这是一个包含 Envoy 容器的 osm-injector Pod：

    ```console
    $ # 假设 OSM 被安装在 osm-system 命名空间里:
    $ kubectl get pod -n osm-system $(kubectl get pods -n osm-system -l app=osm-injector -o jsonpath='{.items[0].metadata.name}') -o jsonpath='{range .spec.containers[*]}{.image}{"\n"}{end}'
    openservicemesh/osm-injector:v0.8.0
    envoyproxy/envoy-alpine:v1.17.2
    ```

    如果任何一个 Pod 正在运行 Envoy 容器，它可能已经被这个或另一个 OSM 实例错误注入。对于每个用 `osm mesh list` 命令找到的网格，验证不健康的 Pod 的 OSM 命名空间是否列在 `osm namespace list` 输出的任何 OSM 实例中，通过 `osm namespace list` 命令可以看到这些命名空间包含 `SIDECAR-INJECTION` "enabled" 的标签。

    例如，对于下列所有网格：

    ```console
    $ osm mesh list
    
    MESH NAME   NAMESPACE      CONTROLLER PODS                  VERSION     SMI SUPPORTED
    osm         osm-system     osm-controller-5494bcffb6-qpjdv  v0.8.0      HTTPRouteGroup:specs.smi-spec.io/v1alpha4,TCPRoute:specs.smi-spec.io/v1alpha4,TrafficSplit:split.smi-spec.io/v1alpha2,TrafficTarget:access.smi-spec.io/v1alpha3
    osm2        osm-system-2   osm-controller-48fd3c810d-sornc  v0.8.0      HTTPRouteGroup:specs.smi-spec.io/v1alpha4,TCPRoute:specs.smi-spec.io/v1alpha4,TrafficSplit:split.smi-spec.io/v1alpha2,TrafficTarget:access.smi-spec.io/v1alpha3
    ```

    注：  `osm-system` (网格控制平面命名空间) 在命名空间列表里是如何显示:

    ```console
    $ osm namespace list --mesh-name osm --osm-namespace osm-system
    NAMESPACE    MESH    SIDECAR-INJECTION
    osm-system   osm2    enabled
    bookbuyer    osm2    enabled
    bookstore    osm2    enabled
    ```

    如果 OSM 命名空间出现在 "osm namespace list" 命令中，并且启用了 "SIDECAR-INJECTION"， 则将该名称空间从注入边车的网格中移除。对于上面的例子：

    ```console
    $ osm namespace remove osm-system --mesh-name osm2 --osm-namespace osm-system2
    ```

1. 确定 Kubernetes 在调度或启动 Pod 时是否遇到错误。

    使用 `kubectl describe` 命令查找关于不健康的 Pod 近期错误。

    对于 osm-controller:

    ```console
    $ # 假设 OSM 被安装在 osm-system 命名空间里:
    $ kubectl describe pod -n osm-system $(kubectl get pods -n osm-system -l app=osm-controller -o jsonpath='{.items[0].metadata.name}')
    ```

    对于 osm-injector:

    ```console
    $ # 假设 OSM 被安装在 osm-system 命名空间里:
    $ kubectl describe pod -n osm-system $(kubectl get pods -n osm-system -l app=osm-injector -o jsonpath='{.items[0].metadata.name}')
    ```

    解决这些错误并再次验证 OSM 的健康状态。

1. 确定 Pod 是否遇到了一个运行时错误。

    通过检查容器的日志，寻找可能在容器启动后发生的任何错误。具体来说，寻找任何包含字符串 `"level": "error"` 的日志。

    对于 osm-controller:

    ```console
    $ # 假设 OSM 被安装在 osm-system 命名空间里:
    $ kubectl logs -n osm-system $(kubectl get pods -n osm-system -l app=osm-controller -o jsonpath='{.items[0].metadata.name}')
    ```

    对于 osm-injector:

    ```console
    $ # Assuming OSM is installed in the osm-system namespace:
    $ kubectl logs -n osm-system $(kubectl get pods -n osm-system -l app=osm-injector -o jsonpath='{.items[0].metadata.name}')
    ```

    解决这些错误并再次验证 OSM 的健康状态。

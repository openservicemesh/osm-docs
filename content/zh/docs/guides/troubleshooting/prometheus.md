---
title: "Prometheus 故障排查"
description: "如何修复 OSM 中集成的 Prometheus 常见的问题"
aliases: "/docs/troubleshooting/observability/prometheus"
type: docs
weight: 2
---

## 无法访问 Prometheus

如果随 OSM 安装的 Prometheus 实例无法访问，执行下面的步骤定位并解决问题

1. 确认 Prometheus 的 Pod 是存在的

    当使用 `osm install --set=osm.deployPrometheus=true` 安装 Pormetheus，在 OSM 控制平面组件所在的 namespace 下 (默认为 `osm-system`)，应当存在一个命名类似 `osm-prometheus-5794755b9f-rnvlr` 的 Pod

    如果没有找到类似的 Pod，通过 `helm` 安装 OSM Helm chart 时，请确认将 `osm.deployPrometheus` 选项设置为 `true`：

    ```console
    $ helm get values -a <mesh name> -n <OSM namespace>
    ```

    如果选项被设置为 `true` 以外的值，重新使用 `osm install` 安装 OSM，并添加 `--set=osm.deployPrometheus=true` 选项。

2. 确认 Prometheus 的 Pod 处于健康状态

    在上一个步骤中定位的 Prometheus Pod 应当处于 Running 状态，并且所有的容器是就绪的，和下面 `kubectl get` 的输出类似：

    ```console
    $ # 假设 OSM 被安装到 osm-system 命名空间下：
    $ kubectl get pods -n osm-system -l app=osm-prometheus
    NAME                              READY   STATUS    RESTARTS   AGE
    osm-prometheus-5794755b9f-67p6r   1/1     Running   0          27m
    ```

    如果 Pod 不是在 Running 状态，或者相关的容器没有就绪，使用 `kubectl describe` 查找其他潜在的问题：

    ```console
    $ # 假设 OSM 被安装到 osm-system 命名空间下：
    $ kubectl describe pods -n osm-system -l app=osm-prometheus
    ```

    当 Prometheus Pod 处于健康状态的时候，Prometheus 应该是可被访问的

## Prometheus 没有记录监控指标

如果发现 Prometheus 没有抓取任何 Pod 的监控指标，请按照下面的步骤定位和解决问题。

1. 确认应用的 Pod 处于预期的工作状态

    如果运行在 mesh 中的工作负载没有正常工作，从那些 Pod 中采集的监控指标可能看起来会不正确。例如，如果表示服务 A 到服务 B 的流量的监控指标丢失，需要确认服务间的通信是否正常。

    为了帮助进一步排查这类问题，请查看流量故障排查指南。

2. 确认那些丢失指标的 Pod 已经注入了 Envoy sidecar

    在预期当中，只有那些注入了 Envoy sidecar 容器的 Pod 的指标，才会被 Prometheus 收集。请确保每一个 Pod 中运行着一个，通过名字上带有 `envoyproxy/envoy` 的镜像启动的容器。

    ```console
    $ kubectl get po -n <pod namespace> <pod name> -o jsonpath='{.spec.containers[*].image}'
    mynamespace/myapp:v1.0.0 envoyproxy/envoy-alpine:v1.17.2
    ```
3. 确认被 Prometheus 抓取的代理的接口是按预期运行的

    每一个 Envoy 代理暴露了一个 HTTP 接口，返回代理产生的指标，并被 Prometheus 采集。直接请求这个 HTTP 接口，检查返回的结果中是否有预期的指标出现。

    对于那些丢失了指标的 Pod，通过 `kubectl` 转发其 Evnoy 代理的管理界面的端口，访问并检查其指标：

    ```console
    $ kubectl port-forward -n <pod namespace> <pod name> 15000
    ```

    在浏览器中访问 http://localhost:15000/stats/prometheus 来获取 Pod 产生的指标。如果 发现 Prometheus 并没有抓取这些指标，继续下一步，确认是否正确配置了 Prometheus。

4. 确认需要的命名空间已被加入到指标收集目标当中

    对于应该被收集指标的 Pod 所在的命名空间，使用 `osm mesh list` 命令，来确认它们被预期的 OSM 实例监控

    下一步，检查和确保命名空间被标注了 `openservicemesh.io/metrics: enabled`：

    ```console
    $ # 假设 OSM 被安装到 os-system 命名空间下：
    $ kubectl get namespace <namespace> -o jsonpath='{.metadata.annotations.openservicemesh\.io/metrics}'
    enabled
    ```

    如果命名空间上缺少类似的标注，或者被标注成了其他值，使用 `osm` 命令来修复

    ```console
    $ osm metrics enable --namespace <namespace>
    Metrics successfully enabled in namespace [<namespace>]
    ```

5. 如果 [自定义指标](/docs/guides/observability/metrics/#custom-metrics) 没有被收集，请检查这些指标是否已被启用。

    自定义指标目前是默认禁用的，当 `osm.featureFlags.enableWASMStats` 参数被设置为 `true` 时被启用。请确认在 `<osm-namespace>` 下，当前名为 `<osm-mesh-name>` 的 OSM 实例设置了该参数：

    ```console
    $ helm get values -a <osm-mesh-name> -n <osm-namespace>
    ```

   > 注意：请把 `<osm-mesh-name>` 替换成 osm mesh 的名字，把 `<osm-namespace>` 替换成安装了 osm 的命名空间。

    如果 `osm.featureFlags.enableWASMStats` 被设置成了其他值，使用 `osm install` 重新安装 OSM，并带上 `--set osm.featureFlags.enableWASMStats` 参数

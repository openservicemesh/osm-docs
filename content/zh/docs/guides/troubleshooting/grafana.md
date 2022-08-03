---
title: "Grafana"
description: "Grafana 集成的故障排查"
aliases: "/docs/troubleshooting/observability/grafana"
type: docs
weight: 16
---

## Grafana 集成的故障排查

如果随 OSM 安装的 Grafana 实例无法访问，执行下面的步骤定位并解决问题

1. 确认 Grafana 的 Pod 是存在的

    当使用 `osm install --set=osm.deployGrafana=true` 安装 Grafana，在 OSM 控制平面组件所在的 namespace 下 (默认为 `osm-system`)，应当存在一个命名类似 `osm-grafana-7c88b9687d-tlzld` 的 Pod

    如果没有找到类似的 Pod，通过 `helm` 安装 OSM Helm chart 时，请确认将 `osm.deployGrafana` 选项设置为 `true`：

    ```console
    $ helm get values -a <mesh name> -n <OSM namespace>
    ```

    如果选项被设置为 `true` 以外的值，重新使用 `osm install` 安装 OSM，并添加 `--set=osm.deployGrafana=true` 选项。

2. 确认 Grafana 的 Pod 处于健康状态

    在上一个步骤中定位的 Grafana Pod 应当处于 Running 状态，并且所有的容器是就绪的，和下面 `kubectl get` 的输出类似：

    ```console
    $ # 假设 OSM 被安装到 osm-system 命名空间下：
    $ kubectl get pods -n osm-system -l app=osm-grafana
    NAME                           READY   STATUS    RESTARTS   AGE
    osm-grafana-7c88b9687d-tlzld   1/1     Running   0          58s
    ```

    如果 Pod 不是在 Running 状态，或者相关的容器没有就绪，使用 `kubectl describe` 查找其他潜在的问题：

    ```console
    $ # 假设 OSM 被安装到 osm-system 命名空间下：
    $ kubectl describe pods -n osm-system -l app=osm-grafana
    ```

    当 Grafana Pod 处于健康状态的时候，Grafana 应该是可被访问的

## Grafana 面板没有数据显示

如果 Grafana 的面板缺少数据显示，按照下面的步骤来定位和解决问题。

1. 确认 Prometheus 已经被安装并且是健康的。

    因为 Grafana 查询的数据来源于 Prometheus，要确保 Prometheus 运行状态是符合预期的。详细内容请参考 [Prometheus 故障排除指南](/docs/troubleshooting/observability/prometheus/)

2. 确认 Grafana 可以访问 Prometheus

    首先在浏览器中打开 Grafana 界面

    ```console
    $ osm dashboard
    [+] Starting Dashboard forwarding
    [+] Issuing open browser http://localhost:3000
    ```

    登录（默认用户名/密码是 admin/admin）并打开 [data source settings](http://localhost:3000/datasources)。点击并查看每一个异常的数据源。可以通过页面底部的 "Save & Test" 的按钮来校验配置。

    如果出现错误，请检查 Grafana 的配置，确保它指向预期的 Prometheus 实例。调整 Grafana 的设置，直到 "Save & Test" 检查没有显示错误信息：

    ![Successful verification](https://user-images.githubusercontent.com/5503924/112394171-7e419e00-8cb9-11eb-99fc-3343c6b9fbbd.png)

    更多关于配置数据源的的内容，可以访问 [Grafana's docs](https://grafana.com/docs/grafana/latest/administration/provisioning/#data-sources).

对于其他问题，参考 [Grafana's troubleshooting documentation](https://grafana.com/docs/grafana/latest/troubleshooting/).

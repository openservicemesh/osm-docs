---
title: "日志"
description: "OSM 控制平面的诊断日志"
type: docs
---

# 日志
开放服务网格 (OSM) 控制平面组件将诊断日志输出到了标准输出上，以帮助管理服务网格。

在日志中，用户应当能看到如下类型的信息
一些附加的信息：
- Kubrentes 资源的元数据，例如名字和命名空间
- mTLS 证书中的 common names

OSM **不会** 记录敏感信息，例如：
- Kubernetes 的 Secret 数据
- 完整的 Kubernetes 资源信息

## 日志级别

日志详尽程度决定了什么样的日志会被记录，例如为了排错记录更多的日志，还是只记录少量的只包含严重级别错误的日志。

OSM 按日志详细程度递增的顺序定义了以下日志级别：

| Log level | Purpose                                                                                |

| 日志级别 | 用途 |
| --------- | -------------------------------------------------------------------------------------- |
| disabled | 完全禁用日志 |
| panic | *当前没有使用* |
| fatal |用于记录导致服务终止的不可恢复的错误，这些错误通常出现在启动阶段 |
| error | 用于记录需要用户介入解决的错误 |
| warn | 用于记录可能导致异常的可恢复性错误，或者非预期的状态 |
| info | 用于记录表明正常行为的信息，例如确认某些用户行为 |
| debug | 用于记录额外的，可以帮助分析网格为何无法按预期工作的信息 |
| trace | 用于记录详尽的日志，通常主要用于开发用途 |

上述的日志级别都可以通过 MeshConfig 中的 `spec.observability.osmLogLevel` 字段来配置，或者在安装时，设置 chart 的 `osm.controllerLogLevel` 选项。

## Fluent Bit
当启用的时候，Fluent Bit 可以收集这些日志，处理并将它们发送到用户指定的目标，例如 Elasticsearch、Azure Log Analytics、BigQuery 等等。

[Fluent Bit](https://fluentbit.io/) 是一个开源的日志处理器和转发器，可以让收集数据/日志并将他们发送到多个日志存储。它可以和 OSM 集成，通过 Fluent Bit 的输出插件，把 OSM 控制器的日志转发到各式各样的输出/日志消费者。

在安装时，通过使用 `--set=osm.enableFluentbit=true` 参数，来决定将 Fluent Bit sidecar 部署到 OSM 控制平面中，使 OSM 能够进行日志转发。安装后，用户可以通过使用任意可用的 [Fluent Bit 输出插件](https://docs.fluentbit.io/manual/pipeline/outputs)来决定 OSM 的日志应该被转发到何处。

### 配置 Fluent Bit 日志转发
默认情况下，Fluent Bit sidecar 被设置成直接把日志输出到 Fluent Bit 容器的标准输出中。 如果安装 OSM 时同时启用了 Fluent Bit，可以通过 `kubectl logs -n <osm-namespace> <osm-controller-name> -c fluentbit-logger` 来查看这些日志。如果想调整日志分析和过滤器，这个命令也能够帮助查看日志是如何被格式化输出的。

> 注意：`<osm-namespace>` 指安装了 osm 控制平面的命名空间

要使用默认配置快速启动 Fluent Bit，请设置 `--set=osm.enableFluentbit` 选项
```console
osm install --set=osm.enableFluentbit=true
```

默认情况下，日志会被过滤并输出 info 级别的日志。可以在安装过程中通过 `--set osm.controllerLogLevel=<desired log level>` 将日志级别修改为 "debug", "warn", "fatal", "panic", "disabled" 或者 "trace"。

当尝试过这些基础的配置后，为了获得更丰富的输出结果，我们建议配置日志转发到首选的日志存储上。

要自定义日志转发到日志存储，按照下面的步骤，安装 OSM 并启用 Fluent Bit

1. 在[Fluent Bit 文档](https://docs.fluentbit.io/manual/pipeline/outputs)中找到想使用的输出插件来转发日志。在 [`fluentbit-configmap.yaml`](https://github.com/openservicemesh/osm/blob/{{< param osm_branch >}}/charts/osm/templates/fluentbit-configmap.yaml) 里，把 `[OUTPUT]` 替换成相应的值。

2. 默认的配置中，使用 CRI 日志格式进行分析。如果正在使用的 kubernetes 发行版用不同的格式输出日志，可能需要添加一个新的解析器到 `[PARSER]` 配置段落中，并在 `[INPUT]` 段落中，将 `parser` 名字修改为[这些](https://github.com/fluent/fluent-bit/blob/master/conf/parsers.conf)预定义解析器中的一个。

3. 查看可用的 [Fluent Bit 过滤器](https://docs.fluentbit.io/manual/pipeline/filters) 并按需添加 `[FILTER]` 配置段落
    * `[INPUT]` 配置段落给输入的日志打上 `kube.*` 的标签，所以请确保在自定义过滤器中设置了 `Match kube.*` 的键值对。
    * 默认的配置使用了一个修改过的过滤器，给日志添加了 `controller_pod_name` 键值对，在日志输出目的地查询日志时，帮助通过 pod 名字来提取日志。（请参考下面的例子）

1. 为了让修改生效，执行
    ```console
    make build-osm
    ```

2. 一旦修改了 Fluent Bit 的 Configmap 模板，可以在安装 OSM 时部署 Fluent Bit，使用命令：
    ```console
    osm install --set=osm.enableFluentbit=true [--set osm.controllerLogLevel=<期望日志级别>]
    ```
    当日志生成时，现在应当能够在选择的日志存储中查看错误日志了。


### 示例：使用 Fluent Bit 发送日志到 Azure Monitor
Fluent Bit 提供了 Azure 输出插件，可以像下面一样，将日志发送到 Azure Log Analytics 工作区
1. [创建一个 Log Analytics 工作区](https://docs.microsoft.com/en-us/azure/azure-monitor/learn/quick-create-workspace)

2. 在 Azure 门户上，切换到新建的工作区。在 Agents 管理中，找到工作区的 ID以及主键。在 `values.yaml` 中，`fluentBit` 下面，修改 `outputPlugin` 为 `azure`，同时，`workspaceId` 和 `primaryKey` 替换成 Azure 门户上相应的值（不包含引号）。或者，如果使用其他的输出插件，可以替换 `fluentbit-configmap.yaml` 中整个 output 配置段落。

3. 执行上面的步骤 2-5.

4. 当部署 OSM 并启用了 Fluent Bit，日志会发送到 Log Analytics 工作区下的 Logs > Custom Logs 标签页下。在那里，可以先执行下面的查询语句来查看最新日志：
    ```
    fluentbit_CL
    | order by TimeGenerated desc
    ```

5. 查找某个特定 OSM 控制器的 pod 的日志
    ```
    | where controller_pod_name_s == "<预期的 osm 控制器的 pod 的名字>"
    ```

当日志发送到 Log Analytics 后，和下面例子一样，它们同时可以被 Application Insights 处理：
1. [创建一个基于工作区的 Application Insights 实例](https://docs.microsoft.com/zh-cn/azure/azure-monitor/app/create-workspace-resource)

2. 在 Azure 门户中，切换到实例。访问 Logs 标签页。执行这条查询语句，确保日志已经被 Log Analytics 收集：
    ```
    workspace("<-log-analytics-工作区-名字>").fluentbit_CL
    ```

现在可以在这些实例当中查看日志。

*注意：OpenShift 当前不支持 Fluent Bit*

### 为 Fluent Bit 配置出口代理
如果出口流量需要经过一个代理服务器，可能需要配置出口代理。有两种配置方式启用代理。

如果已经通过之前得 MeshConfig 部署了 OSM，方便地可以通过 OSM CLI 启用代理的支持，在下面的命令中，将参数替换成环境当中的相应的值：
```
osm install --set=osm.enableFluentbit=true,osm.fluentBit.enableProxySupport=true,osm.fluentBit.httpProxy=<http-代理-主机:端口>,osm.fluentBit.httpsProxy=<https-代理-主机:端口>
```

或者，可以在 Helm chat中，和下面一样，通过修改 `values.yaml` 的值来配置：
1. 修改 `enableProxySupport` 为 `true`

2. 修改 httpProxy 和 httpsProxy 的值为 `"http://<主机>:<端口>"`。如果代理服务器需要 basic authentication, 可以加入代理的用户名和密码，例如：`http://<用户名>:<密码>@<主机>:<端口>`

3. 为了让这些修改生效，执行
    ```console
    make build-osm
    ```

4. 安装 OSM 并启用 Fluent Bit：
    ```console
    osm install --set=osm.enableFluentbit=true
    ```
> 注意：这些特性需要确保 [Fluent Bit 镜像的 tag](https://github.com/openservicemesh/osm/blob/{{< param osm_branch >}}/charts/osm/values.yaml) 是 `1.6.4`或者更高的版本。

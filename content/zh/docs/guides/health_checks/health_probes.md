---
title: "配置健康探测"
description: "OSM 如何处理应用健康探测的工作，以及如果探测失败该如何处理"
aliases: "/docs/application_health_probes"
type: "docs"
---

# 配置健康探测

## 概述

在应用程序中实施[健康探测](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)是 Kubernetes 自动执行一些任务的好方法，以便在发生错误时提高可用性。

由于 OSM 重新配置应用 Pod，使其通过代理边车重定向所有传入和传出的网络流量，kubelet 调用的 `httpGet` 和 `tcpSocket` 健康探测会因为代理缺少 mTLS 要求的上下文而失败。

OSM 增加了配置，通过代理暴露探测端点，并重写新 Pod 的探测定义，以引用代理暴露的端点，使得 `httpGet` 健康探测可以在服务网格中工作。原始探针的所有功能仍可用，OSM 只是将其与代理前置，以便 kubelet 能够与之通信。

需要特殊的配置来支持服务网中的 `tcpSocket` 健康探针。由于 OSM 通过 Envoy 重定向所有网络流量，所有的端口在 Pod 中都是开放的。这导致所有的 TCP 连接被路由到注入了 Envoy sidecar 的 Pod，看起来是成功的。为了使 `tcpSocket` 健康探测在网格中正常工作，OSM 将探测改写为 `httpGet` 探测，并添加了一个 `iptables` 命令，以绕过 `osm-healthcheck`暴露端点的Envoy代理。`osm-healthcheck` 容器被添加到 Pod 中，处理来自 kubelet 的 HTTP 健康探测请求。处理程序从请求的 `Original-Tcp-port` 头中获取原始 TCP 端口，并尝试在指定端口上打开一个 socket。`httpGet` 探针的响应状态代码反映 TCP 连接是否成功。

| Probe       | Path                 | Port  |
| ----------- | -------------------- | ----- |
| Liveness    | /osm-liveness-probe  | 15901 |
| Readiness   | /osm-readiness-probe | 15902 |
| Startup     | /osm-startup-probe   | 15903 |
| Healthcheck | /osm-healthcheck     | 15904 |

对于 HTTP 和 `tcpSocket` 探测，端口和路径会被修改。对于 HTTPS 探针，端口被修改，但路径保持不变。

只有预定义的 `httpGet` 和 `tcpSocket` 探针会被修改。如果一个探针未被定义，则不会在其位置上添加。只要 `exec` 探针（包括使用 `grpc_health_probe` 的探针）的命令不访问 `localhost` 以外的网络，则不会被修改，并正常工作。

## 例子

下面的例子显示 OSM 如何处理网格中的 Pod 的健康探测。

### HTTP

假设 Pod 中的一个容器定义了如下 `livenessProbe` 探测：

```yaml
livenessProbe:
  httpGet:
    path: /liveness
    port: 14001
    scheme: HTTP
```

当 Pod 被创建时，OSM 将修改探针为以下内容：

```yaml
livenessProbe:
  httpGet:
    path: /osm-liveness-probe
    port: 15901
    scheme: HTTP
```

该 Pod 的代理将包含以下 Envoy 配置。

一个 Envoy 集群，它映射到原始探针端口 14001：

```json
{
  "cluster": {
    "@type": "type.googleapis.com/envoy.config.cluster.v3.Cluster",
    "name": "liveness_cluster",
    "type": "STATIC",
    "connect_timeout": "1s",
    "load_assignment": {
      "cluster_name": "liveness_cluster",
      "endpoints": [
        {
          "lb_endpoints": [
            {
              "endpoint": {
                "address": {
                  "socket_address": {
                    "address": "0.0.0.0",
                    "port_value": 14001
                  }
                }
              }
            }
          ]
        }
      ]
    }
  },
  "last_updated": "2021-03-29T21:02:59.086Z"
}
```

为新的代理暴露的 HTTP 端点 `/osm-liveness-probe` 建立监听器，端口为 15901，映射到上述集群：

```json
{
  "listener": {
    "@type": "type.googleapis.com/envoy.config.listener.v3.Listener",
    "name": "liveness_listener",
    "address": {
      "socket_address": {
        "address": "0.0.0.0",
        "port_value": 15901
      }
    },
    "filter_chains": [
      {
        "filters": [
          {
            "name": "envoy.filters.network.http_connection_manager",
            "typed_config": {
              "@type": "type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager",
              "stat_prefix": "health_probes_http",
              "route_config": {
                "name": "local_route",
                "virtual_hosts": [
                  {
                    "name": "local_service",
                    "domains": [
                      "*"
                    ],
                    "routes": [
                      {
                        "match": {
                          "prefix": "/osm-liveness-probe"
                        },
                        "route": {
                          "cluster": "liveness_cluster",
                          "prefix_rewrite": "/liveness"
                        }
                      }
                    ]
                  }
                ]
              },
              "http_filters": [...],
              "access_log": [...]
            }
          }
        ]
      }
    ]
  },
  "last_updated": "2021-03-29T21:02:59.092Z"
}
```

### HTTPS

假设 Pod 中的一个容器定义了如下 `livenessProbe` 探测：

```yaml
livenessProbe:
  httpGet:
    path: /liveness
    port: 14001
    scheme: HTTPS
```

当 Pod 被创建时，OSM 将修改探针为以下内容：

```yaml
livenessProbe:
  httpGet:
    path: /liveness
    port: 15901
    scheme: HTTPS
```

该 Pod 的代理将包含以下 Envoy 配置。

一个 Envoy 集群，它映射到原始探针端口 14001：

```json
{
  "cluster": {
    "@type": "type.googleapis.com/envoy.config.cluster.v3.Cluster",
    "name": "liveness_cluster",
    "type": "STATIC",
    "connect_timeout": "1s",
    "load_assignment": {
      "cluster_name": "liveness_cluster",
      "endpoints": [
        {
          "lb_endpoints": [
            {
              "endpoint": {
                "address": {
                  "socket_address": {
                    "address": "0.0.0.0",
                    "port_value": 14001
                  }
                }
              }
            }
          ]
        }
      ]
    }
  },
  "last_updated": "2021-03-29T21:02:59.086Z"
}
```

为新的代理暴露的 TCP 端点提供监听器，端口 15901，映射到上述集群：

```json
{
  "listener": {
    "@type": "type.googleapis.com/envoy.config.listener.v3.Listener",
    "name": "liveness_listener",
    "address": {
      "socket_address": {
        "address": "0.0.0.0",
        "port_value": 15901
      }
    },
    "filter_chains": [
      {
        "filters": [
          {
            "name": "envoy.filters.network.tcp_proxy",
            "typed_config": {
              "@type": "type.googleapis.com/envoy.extensions.filters.network.tcp_proxy.v3.TcpProxy",
              "stat_prefix": "health_probes",
              "cluster": "liveness_cluster",
              "access_log": [...]
            }
          }
        ]
      }
    ]
  },
  "last_updated": "2021-04-07T15:09:22.704Z"
}
```

### `tcpSocket`

假设 Pod 中的一个容器定义了如下 `livenessProbe` 探测：

```yaml
livenessProbe:
  tcpSocket:
    port: 14001
```

当 Pod 被创建时，OSM 将修改探针为以下内容：

```yaml
livenessProbe:
  httpGet:
    httpHeaders:
    - name: Original-Tcp-Port
      value: "14001"
    path: /osm-healthcheck
    port: 15904
    scheme: HTTP
```

访问 15904 端口的请求绕过了 Envoy 代理，被引向 `osm-healthcheck` 端点。

## 如何在网格中验证 POD 的健康状态

Kubernetes 将自动轮询配置了启动（startup）、存活（liveness）和就绪（readiness）探测器的 Pod 的健康检查端点。

当启动探测失败时，Kubernetes 将生成一个事件（通过 `kubectl describe pod <pod name>` 可见）并重新启动 Pod。`kubectl describe` 的输出如下：

```
...
Events:
  Type     Reason     Age              From               Message
  ----     ------     ----             ----               -------
  Normal   Scheduled  17s              default-scheduler  Successfully assigned bookstore/bookstore-v1-699c79b9dc-5g8zn to osm-control-plane
  Normal   Pulled     16s              kubelet            Successfully pulled image "openservicemesh/init:v0.8.0" in 26.5835ms
  Normal   Created    16s              kubelet            Created container osm-init
  Normal   Started    16s              kubelet            Started container osm-init
  Normal   Pulling    16s              kubelet            Pulling image "openservicemesh/init:v0.8.0"
  Normal   Pulling    15s              kubelet            Pulling image "envoyproxy/envoy-alpine:v1.17.2"
  Normal   Pulling    15s              kubelet            Pulling image "openservicemesh/bookstore:v0.8.0"
  Normal   Pulled     15s              kubelet            Successfully pulled image "openservicemesh/bookstore:v0.8.0" in 319.9863ms
  Normal   Started    15s              kubelet            Started container bookstore-v1
  Normal   Created    15s              kubelet            Created container bookstore-v1
  Normal   Pulled     14s              kubelet            Successfully pulled image "envoyproxy/envoy-alpine:v1.17.2" in 755.2666ms
  Normal   Created    14s              kubelet            Created container envoy
  Normal   Started    14s              kubelet            Started container envoy
  Warning  Unhealthy  13s              kubelet            Startup probe failed: Get "http://10.244.0.23:15903/osm-startup-probe": dial tcp 10.244.0.23:15903: connect: connection refused
  Warning  Unhealthy  3s (x2 over 8s)  kubelet            Startup probe failed: HTTP probe failed with statuscode: 503
```

当存活探测失败时，Kubernetes 将生成一个事件（通过 `kubectl describe pod <pod name>` 可见）并重新启动 Pod。`kubectl describe` 的输出如下：

```
...
Events:
  Type     Reason     Age                From               Message
  ----     ------     ----               ----               -------
  Normal   Scheduled  59s                default-scheduler  Successfully assigned bookstore/bookstore-v1-746977967c-jqjt4 to osm-control-plane
  Normal   Pulling    58s                kubelet            Pulling image "openservicemesh/init:v0.8.0"
  Normal   Created    58s                kubelet            Created container osm-init
  Normal   Started    58s                kubelet            Started container osm-init
  Normal   Pulled     58s                kubelet            Successfully pulled image "openservicemesh/init:v0.8.0" in 23.415ms
  Normal   Pulled     57s                kubelet            Successfully pulled image "envoyproxy/envoy-alpine:v1.17.2" in 678.1391ms
  Normal   Pulled     57s                kubelet            Successfully pulled image "openservicemesh/bookstore:v0.8.0" in 230.3681ms
  Normal   Created    57s                kubelet            Created container envoy
  Normal   Pulling    57s                kubelet            Pulling image "envoyproxy/envoy-alpine:v1.17.2"
  Normal   Started    56s                kubelet            Started container envoy
  Normal   Pulled     44s                kubelet            Successfully pulled image "openservicemesh/bookstore:v0.8.0" in 20.6731ms
  Normal   Created    44s (x2 over 57s)  kubelet            Created container bookstore-v1
  Normal   Started    43s (x2 over 57s)  kubelet            Started container bookstore-v1
  Normal   Pulling    32s (x3 over 58s)  kubelet            Pulling image "openservicemesh/bookstore:v0.8.0"
  Warning  Unhealthy  32s (x6 over 50s)  kubelet            Liveness probe failed: HTTP probe failed with statuscode: 503
  Normal   Killing    32s (x2 over 44s)  kubelet            Container bookstore-v1 failed liveness probe, will be restarted
```

当就绪探测失败时，Kubernetes 将生成一个事件（通过 `kubectl describe pod <pod name>` 可以看到），并确保服务流量不会路由到这些不健康的 POD 上。一个准备就绪探测失败的 Pod 的 `kubectl describe` 输出如下：

```
...
Events:
  Type     Reason     Age               From               Message
  ----     ------     ----              ----               -------
  Normal   Scheduled  32s               default-scheduler  Successfully assigned bookstore/bookstore-v1-5848999cb6-hp6qg to osm-control-plane
  Normal   Pulling    31s               kubelet            Pulling image "openservicemesh/init:v0.8.0"
  Normal   Pulled     31s               kubelet            Successfully pulled image "openservicemesh/init:v0.8.0" in 19.8726ms
  Normal   Created    31s               kubelet            Created container osm-init
  Normal   Started    31s               kubelet            Started container osm-init
  Normal   Created    30s               kubelet            Created container bookstore-v1
  Normal   Pulled     30s               kubelet            Successfully pulled image "openservicemesh/bookstore:v0.8.0" in 314.3628ms
  Normal   Pulling    30s               kubelet            Pulling image "openservicemesh/bookstore:v0.8.0"
  Normal   Started    30s               kubelet            Started container bookstore-v1
  Normal   Pulling    30s               kubelet            Pulling image "envoyproxy/envoy-alpine:v1.17.2"
  Normal   Pulled     29s               kubelet            Successfully pulled image "envoyproxy/envoy-alpine:v1.17.2" in 739.3931ms
  Normal   Created    29s               kubelet            Created container envoy
  Normal   Started    29s               kubelet            Started container envoy
  Warning  Unhealthy  0s (x3 over 20s)  kubelet            Readiness probe failed: HTTP probe failed with statuscode: 503
```

Pod 的 `status` 也可看出其尚未可用，在 `kubectl get pod` 输出中可以看到。例如：

```
NAME                            READY   STATUS    RESTARTS   AGE
bookstore-v1-5848999cb6-hp6qg   1/2     Running   0          85s
```

Pod 的健康探测也可以通过转发 Pod 的必要端口并使用 `curl` 或任何其他 HTTP 客户端发出请求手动调用。例如，为了验证 bookstore-v1 demo Pod 的有效性探测，获取 Pod 的名称并转发 15901 端口：

```
kubectl port-forward -n bookstore deployment/bookstore-v1 15901
```

然后，在一个单独的终端里，可以使用 `curl` 来检查端点。下面是一个健康的 bookstore-v1 的例子：

```console
$ curl -i localhost:15901/osm-liveness-probe
HTTP/1.1 200 OK
date: Wed, 31 Mar 2021 16:00:01 GMT
content-length: 1396
content-type: text/html; charset=utf-8
x-envoy-upstream-service-time: 1
server: envoy

<!doctype html>
<html itemscope="" itemtype="http://schema.org/WebPage" lang="en">
  ...
</html>
```

## 已知问题

- [#3773](https://github.com/openservicemesh/osm/issues/3773)

## 排错

如果有健康探测持续失败，请执行以下步骤以确定根本原因：

1. 验证网格中的 Pod 上的 `httpGet` 和 `tcpSocket` 探针是否被修改。

   启动、存活和就绪的 `httpGet` 探针必须被 OSM 修改。端口必须被修改为 15901、15902 和 15903，分别适用于存活、就绪和启动 `httpGet` 探针。只有 HTTP（不包括HTTPS）探针的路径将被修改，此外还有 `/osm-liveness-probe`、`/osm-readiness-probe` 或 `/osm-starttup-probe`。

   同时，验证 Pod 的 Envoy 配置中是否包含修改后的端点的监听。

   为了让 `tcpSocket` 探针在网格中生效，必须将其改写为 `httpGet` 探针。端口必须被修改为 15904，以用于存活、就绪和启动探测。路径必须设置为 `/osm-healthcheck`。HTTP 头 `Original-TCP-Port`，必须设置为 `tcpSocket` 探针定义中指定的原始端口。另外，验证 `osm-healthcheck` 容器是否正在运行。检查 `osm-healthcheck` 日志以获得更多信息。

   更多细节见[上面的例子](#例子)。

1. 确定 Kubernetes 在调度或启动 Pod 时是否遇到了任何其他错误。

   使用 `kubectl describe` 命令查找关于不健康的 Pod 近期的错误。解决这些错误并再次验证 POD 的健康状态。

1. 确定 Pod 是否遇到了一个运行时错误。

   使用 `kubectl logs` 检查容器日志，寻找容器启动后发生的错误。解决这些错误并再次验证 Pod 的健康状况。


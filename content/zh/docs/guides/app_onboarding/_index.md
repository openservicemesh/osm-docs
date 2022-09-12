---
title: "应用迁入"
description: "迁入服务"
type: docs
weight: 5
---

# 发布服务

以下指南介绍了如何将 Kubernetes 微服务迁入 OSM 实例。

1. 在迁入应用之前，参考 [应用要求](/docs/guides/app_onboarding/prereqs) 指南。

2. 配置并安装 [服务网格接口 (Service Mesh Interface，SMI) 策略](https://github.com/servicemeshinterface/smi-spec)。

    OSM 符合 SMI 规范，默认情况下，OSM 禁止 Kubernetes 服务间的通信，除非显式地通过 SMI 策略来允许。这种行为可以通过在 `osm install` 时指定 `--set=osm.enablePermissiveTrafficPolicy=true` 参数来覆盖，允许不执行 SMI 策略，同时允许流量和服务仍然利用诸如 mTLS 加密流量、指标和跟踪等功能。。

    SMI 策略的例子，请参阅以下示例：
    - [demo/deploy-traffic-specs.sh](https://github.com/openservicemesh/osm/blob/{{< param osm_branch >}}/demo/deploy-traffic-specs.sh)
    - [demo/deploy-traffic-split.sh](https://github.com/openservicemesh/osm/blob/{{< param osm_branch >}}/demo/deploy-traffic-split.sh)
    - [demo/deploy-traffic-target.sh](https://github.com/openservicemesh/osm/blob/{{< param osm_branch >}}/demo/deploy-traffic-target.sh)

3. 网格中的应用如果需要与 Kubernetes API server 通信，用户需要显式地通过使用 IP 地址范围排除或者创建类似下面的出口策略来允许。

   首选，获取 Kubernetes API server 的集群 IP：
   ```console
   $ kubectl get svc -n default
   NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
   kubernetes   ClusterIP   10.0.0.1     <none>        443/TCP   1d
   ```

    **方法1：** 将 Kubernetes API server 的地址添加到全局出站 IP 范围进行排除。IP 地址可以是集群 IP 地址或公共 IP 地址，应适当排除来连接到 Kubernetes API server。
    
    添加这个 IP 到 MeshConfig 使得出站流量可以从 OSM sidecar 的流量劫持中排除：
    
    ```console
    $ kubectl patch meshconfig osm-mesh-config -n <osm-namespace> -p '{"spec":{"traffic":{"outboundIPRangeExclusionList":["10.0.0.1/32"]}}}'  --type=merge
    meshconfig.config.openservicemesh.io/osm-mesh-config patched
    ```
    
    重启网格监控的命名空间的相关 pod 让改动生效。

    **方法2：** 应用 Egress 策略允许通过 HTTPS 访问 Kubernetes API server。
   
   > _注意：当使用 Egress 策略时，Kubernetes API server不能位于网格管理的命名空间中_

    1. 启用出口策略，如果没有启用的话。
    ```console
    kubectl patch meshconfig osm-mesh-config -n <osm-namespace> -p '{"spec":{"featureFlags":{"enableEgressPolicy":true}}}'  --type=merge
    ```
   
    2. 应用出口策略允许应用的 ServiceAccount 访问上面得到的 Kubernetes API server 的集群 IP。

        例如：
        ```console
        kubectl apply -f - <<EOF
        kind: Egress
        apiVersion: policy.openservicemesh.io/v1alpha1
        metadata:
            name: k8s-server-egress
            namespace: test
        spec:
            sources:
            - kind: ServiceAccount
              name: <app pod's service account name>
              namespace: <app pod's service account namespace>
            ipAddresses:
            - 10.0.0.1/32
            ports:
            - number: 443
              protocol: https
        EOF
        ```  

4. 将 Kubernetes 命名空间迁入 OSM

    如果要将命名空间下的所有应用都纳入 OSM 网格管理，执行 `osm namespace add` 命令：

    ```console
    $ osm namespace add <namespace> --mesh-name <mesh-name>
    ```

    默认情况下，`osm namespace add` 为命名空间下的 pod 开启自动的 sidecar 注入。

    如果要禁用 sidecar 的自动注入，在将命名空间注册到网格时使用命令 `osm namespace add <namespace> --disable-sidecar-injection`。
    一旦命名空间迁入，注册到网格的 pod 将自动注入 sidecar。参考文档 [Sidecar 注入](/docs/guides/app_onboarding/sidecar_injection) 获取更多详细信息。

5.  部署新应用或重新部署已有应用

    默认地，已迁入的命名空间中新部署的应用会自动注入 sidecar。这意味着当在纳入网格的命名空间中创建新 Pod 时，OSM 会自动为其注入 sidecar 代理。
    已有的应用需要重启来让 OSM 在重建 Pod 时自动为其注入 sidecar 代理。被 Deployment 管理的 Pod 可以使用命令 `kubectl rollout restart deploy` 来重启。

    为了正确地将协议指定的流量路由到服务端口，需要配置应用使用的协议。参考 [应用协议选择指南](/docs/guides/app_onboarding/app_protocol_selection)。

#### 注意：移除命名空间

要从 OSM 网格中移除命名空间，可以使用命令 `osm namespace remove`：

```console
$ osm namespace remove <namespace>
```

> **请注意：**
> `osm namespace remove` 命令指示告诉 OSM 停止对命名空间中的 sidecar 代理配置应用更新。并**不会**移除 sidecar 代理。这意味着可以继续使用现有的代理配置，但是不会收到 OSM 控制平面的更新。如果想要移除所有 pod 的代理，在使用 CLI 将命名空间从 OSM 网格移除后重新安装所有 pod 负载。

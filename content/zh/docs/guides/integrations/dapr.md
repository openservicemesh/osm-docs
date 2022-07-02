---
title: "Dapr 与 OSM 集成"
description: "将 Dapr 与 OSM 集成的简单演示"
aliases: "/docs/integrations/demo_dapr"
type: docs
weight: 2
---

# Dapr 与 OSM 集成

## Dapr OSM 演练

本文档将引导完成让 Dapr 在 Kubernetes 集群上使用 OSM 的步骤。

1. 在集群上安装 Dapr 并禁用 mTLS：

   1. Dapr 有一个快速入门存储库，可帮助用户熟悉 dapr 及其功能。对于这个集成演示，我们将使用 [hello-kubernetes](https://github.com/dapr/quickstarts/tree/master/hello-kubernetes) 快速入门。由于我们想将此 Dapr 示例与 OSM 集成，因此需要进行一些修改，如下所示：

       - [hello-kubernetes](https://github.com/dapr/quickstarts/tree/master/hello-kubernetes) 演示安装了启用 mtls 的 Dapr（默认），我们 **不想要来自 Dapr 的 mtls 而是用 OSM 的 mTLS**。因此，在集群上 [安装 Dapr](https://github.com/dapr/quickstarts/tree/master/hello-kubernetes#step-1---setup-dapr-on-your-kubernetes-cluster) 时，请确保在安装过程中通过传递标志来禁用 mtls：`--enable-mtls=false`
       - 进一步 [hello-kubernetes](https://github.com/dapr/quickstarts/tree/master/hello-kubernetes) 设置默认命名空间中的所有内容，**强烈建议**设置整个 hello-kubernetes 在特定命名空间中进行演示（我们稍后会将此命名空间加入到 OSM 的网格中）。出于此集成的目的，我们将命名空间设为 `dapr-test`

        ```console
         $ kubectl create namespace dapr-test
         namespace/dapr-test created
        ```

      - [redis 状态存储](https://github.com/dapr/quickstarts/tree/master/hello-kubernetes#step-2---create-and-configure-a-state-store)、[redis. yaml](https://github.com/dapr/quickstarts/blob/master/hello-kubernetes/deploy/redis.yaml)、[node.yaml](https://github.com/dapr/quickstarts/blob/master/hello-kubernetes/deploy/node.yaml) 和 [python.yaml](https://github.com/dapr/quickstarts/blob/master/hello-kubernetes/deploy/python.yaml) 需要部署在 `dapr-test` 命名空间
       - 由于此演示的资源是在自定义命名空间中设置的。我们需要在集群上添加一个 rbac 规则，以便 Dapr 能够访问这些secret。创建以下角色和角色绑定：

        ```bash
        kubectl apply -f - <<EOF
        ---
        apiVersion: rbac.authorization.k8s.io/v1
        kind: Role
        metadata:
          name: secret-reader
          namespace: dapr-test
        rules:
        - apiGroups: [""]
          resources: ["secrets"]
          verbs: ["get", "list"]
        ---

        kind: RoleBinding
        apiVersion: rbac.authorization.k8s.io/v1
        metadata:
          name: dapr-secret-reader
          namespace: dapr-test
        subjects:
        - kind: ServiceAccount
          name: default
        roleRef:
          kind: Role
          name: secret-reader
          apiGroup: rbac.authorization.k8s.io
        EOF
        ```

   2. 确保示例应用程序与 Dapr 按预期运行。

2. 安装 OSM：

   ```console
   $ osm install
   OSM installed successfully in namespace [osm-system] with mesh name [osm]
   ```

3. 在 OSM 中开启宽松流量策略模式：

   ```console
   $ kubectl patch meshconfig osm-mesh-config -n osm-system -p '{"spec":{"traffic":{"enablePermissiveTrafficPolicyMode":true}}}'  --type=merge
   meshconfig.config.openservicemesh.io/osm-mesh-config patched
   ```

   这是必要的，以便 hello-kubernetes 示例如常工作，并且从一开始就不需要 SMI 策略。

4. 将 Kubernetes API server IP 地址从 OSM sidecar 拦截中排除：

   1. 获取 kubernetes API server 的 集群 IP：
      ```console
      $ kubectl get svc -n default
      NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
      kubernetes   ClusterIP   10.0.0.1     <none>        443/TCP   1d
      ```
   2. 将 IP 添加到 MeshConfig 中以便从 OSM sidecar 的出站流量中排除掉。
      ```console
      $ kubectl patch meshconfig osm-mesh-config -n osm-system -p '{"spec":{"traffic":{"outboundIPRangeExclusionList":["10.0.0.1/32"]}}}'  --type=merge
      meshconfig.config.openservicemesh.io/osm-mesh-config patched
      ```

   将 Kubernetes API server IP 从 OSM 中排除掉是必要的，因为演示中 Dapr 利用 Kubernetes secrets 来访问 redis 状态存储。

   _注意：如果已经在 Dapr 的组件文件中硬编码了密码，可以跳过这一步。_

5. 在 OSM sidecar 中全局排除端口的流量拦截：

   1. 获取 Dapr placement server 的端口（`dapr-placement-server`）
      ```console
      $ kubectl get svc -n dapr-system
      NAME                    TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)              AGE
      dapr-api                ClusterIP   10.0.172.245   <none>        80/TCP               2h
      dapr-dashboard          ClusterIP   10.0.80.141    <none>        8080/TCP             2h
      dapr-placement-server   ClusterIP   None           <none>        50005/TCP,8201/TCP   2h
      dapr-sentry             ClusterIP   10.0.87.36     <none>        80/TCP               2h
      dapr-sidecar-injector   ClusterIP   10.0.77.47     <none>        443/TCP              2h
      ```
   2. 从 [redis.yaml](https://github.com/dapr/quickstarts/blob/master/hello-kubernetes/deploy/redis.yaml) 中获取 redis 状态存储的端口，演示中默认 `6379`

   3. 将这些端口添加到 MeshConfig 中，一遍 OSM sidecar 的出站流量拦截不会拦截这些端口的流量。

      ```console
      $ kubectl patch meshconfig osm-mesh-config -n osm-system -p '{"spec":{"traffic":{"outboundPortExclusionList":[50005,8201,6379]}}}'  --type=merge
      meshconfig.config.openservicemesh.io/osm-mesh-config patched
      ```

   从 OSM sidecar 的拦截中排除掉 Dapr placement server（`dapr-placement-server`）的端口是有必要的，因为那些有 Dapr 的 pod 需要与 Dapr 的控制平面通信。Redis 状态存储同样需要被排除，以便 Dapr sidecar 可以路由 redis 的流量，不会被 OSM sidecar 拦截。

    _注意：全局排除端口会导致 OSM 网格中的所有 pod 都不会拦截这些端口的流量。如果只想在部分 pod 上运行 Dapr，需要跳过这一步而执行下面的步骤。_

6. 在 pod 级别将端口从 OSM sidecar 拦截中排除：

   1. 获取 Dapr api 和 sentry 的端口（`dapr-sentry` 和 `dapr-api`）

      ```console
      $ kubectl get svc -n dapr-system
      NAME                    TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)              AGE
      dapr-api                ClusterIP   10.0.172.245   <none>        80/TCP               2h
      dapr-dashboard          ClusterIP   10.0.80.141    <none>        8080/TCP             2h
      dapr-placement-server   ClusterIP   None           <none>        50005/TCP,8201/TCP   2h
      dapr-sentry             ClusterIP   10.0.87.36     <none>        80/TCP               2h
      dapr-sidecar-injector   ClusterIP   10.0.77.47     <none>        443/TCP              2h
      ```

   2. 更新 nodeapp （[node.yaml](https://github.com/dapr/quickstarts/blob/master/hello-kubernetes/deploy/node.yaml)）和 pythonapp（[python.yaml](https://github.com/dapr/quickstarts/blob/master/hello-kubernetes/deploy/python.yaml) ）的 pod 声明，加入 `openservicemesh.io/outbound-port-exclusion-list: "80"` 注解。

   为 pod 添加注解可以排除 Dapr 的 api（`dapr-api`） 和 sendtry（`dapr-sentry`） 的端口，防止被 OSM sidecar 拦截。因为这些 pod 需要与 Dapr 的控制平面通信。

7. 将 Dapr hello-kubernetes 演示所在的命名空间加入到OSM 网格中：

   ```console
   $ osm namespace add dapr-test
   Namespace [dapr-test] successfully added to mesh [osm]
   ```

8. 删除并重新部署 Dapr hello-kubernetes pod：

   ```console
   $ kubectl delete -f ./deploy/node.yaml
   service "nodeapp" deleted
   deployment.apps "nodeapp" deleted
   ```

   ```console
   $ kubectl delete -f ./deploy/python.yaml
   deployment.apps "pythonapp" deleted
   ```

   ```console
   $ kubectl apply -f ./deploy/node.yaml
   service "nodeapp" created
   deployment.apps "nodeapp" created
   ```

   ```console
   $ kubectl apply -f ./deploy/python.yaml
   deployment.apps "pythonapp" created
   ```

   pythonapp 和 nodeapp pod 重启后都会有 3个容器，表示 OSM sidecar 已经成功注入。

   ```console
   $ kubectl get pods -n dapr-test
   NAME                         READY   STATUS    RESTARTS   AGE
   my-release-redis-master-0    1/1     Running   0          2h
   my-release-redis-slave-0     1/1     Running   0          2h
   my-release-redis-slave-1     1/1     Running   0          2h
   nodeapp-7ff6cfb879-9dl2l     3/3     Running   0          68s
   pythonapp-6bd9897fb7-wdmb5   3/3     Running   0          53s
   ```

9. 验证 Dapr hello-kubernetes 演示如预期工作：

   1. 使用  [这篇](https://github.com/dapr/quickstarts/tree/master/hello-kubernetes#step-3---deploy-the-nodejs-app-with-the-dapr-sidecar) 文档中的步骤验证 nodeapp 服务。

   2. 使用[这篇](https://github.com/dapr/quickstarts/tree/master/hello-kubernetes#step-6---observe-messages)文章验证 pythonapp。

10. 应用 SMI 流量策略：

    到目前为止示例演示了 OSM 中的宽松流量策略模式，其中网格内的应用连接由 `osm-controller` 自动配置，因此 pythonapp 和 nodeapp 的通信不需要 SMI 策略。


    为了验证演示在 SMI 流量策略下也能正常功能，请按照以下步骤操作：

    1. 禁用宽松模式：

       ```console
       $ kubectl patch meshconfig osm-mesh-config -n osm-system -p '{"spec":{"traffic":{"enablePermissiveTrafficPolicyMode":false}}}'  --type=merge
       meshconfig.config.openservicemesh.io/osm-mesh-config patched
       ```

    2. 按照[这篇](https://github.com/dapr/quickstarts/tree/master/hello-kubernetes#step-6---observe-messages)文档验证 pythonapp 不再引起 order ID 自增长。

    3. 为 nodeapp 和 pythonapp 创建 service account：

       ```console
       $ kubectl create sa nodeapp -n dapr-test
       serviceaccount/nodeapp created
       ```

       ```console
       $ kubectl create sa pythonapp -n dapr-test
       serviceaccount/pythonapp created
       ```

    4. 更新集群的角色绑定，使其包含新创建的 service account：

       ```bash
       kubectl apply -f - <<EOF
       ---
       kind: RoleBinding
       apiVersion: rbac.authorization.k8s.io/v1
       metadata:
         name: dapr-secret-reader
         namespace: dapr-test
       subjects:
       - kind: ServiceAccount
         name: default
       - kind: ServiceAccount
         name: nopdeapp
       - kind: ServiceAccount
         name: pythonapp
       roleRef:
         kind: Role
         name: secret-reader
         apiGroup: rbac.authorization.k8s.io
       EOF
       ```

    5. 应用下面的 SMI 访问控制策略：

       部署 SMI TrafficTarget

       ```bash
       kubectl apply -f - <<EOF
       ---
       kind: TrafficTarget
       apiVersion: access.smi-spec.io/v1alpha3
       metadata:
         name: pythodapp-traffic-target
         namespace: dapr-test
       spec:
         destination:
           kind: ServiceAccount
           name: nodeapp
           namespace: dapr-test
         rules:
         - kind: HTTPRouteGroup
           name: nodeapp-service-routes
           matches:
           - new-order
         sources:
         - kind: ServiceAccount
           name: pythonapp
           namespace: dapr-test
       EOF
       ```

       部署 HTTPRouteGroup 策略

       ```bash
       kubectl apply -f - <<EOF
       ---
       apiVersion: specs.smi-spec.io/v1alpha4
       kind: HTTPRouteGroup
       metadata:
         name: nodeapp-service-routes
         namespace: dapr-test
       spec:
         matches:
         - name: new-order
       EOF
       ```

    6. 更新 nodeapp ([node.yaml](https://github.com/dapr/quickstarts/blob/master/hello-kubernetes/deploy/node.yaml)) 和 pythonapp ([python.yaml](https://github.com/dapr/quickstarts/blob/master/hello-kubernetes/deploy/python.yaml)) pod 声明，加入各自的 service account。删除并重新部署 Dapr hello-kubernetes pod。

    7. 验证 Dapr hello-kubernetes 按预期工作，参考[这里](https://github.com/dapr/quickstarts/tree/master/hello-kubernetes#step-6---observe-messages)。

11. 清理

    1. 要清理 Dapr hello-kubernetes 演示，清理 `dapr-test` 命名空间

       ```console
       $ kubectl delete ns dapr-test
       ```

    2. 卸载 Dapr，运行：

       ```console
       $ dapr uninstall --kubernetes
       ```

    3. 卸载 OSM，运行：

       ```console
       $ osm uninstall mesh
       ```

    4. 在卸载 OSM 后删除集群范围的资源，执行下面的命令。参阅 [卸载指南](/docs/guides/uninstall/) 获取更多信息。

       ```console
       $ osm uninstall mesh --delete-cluster-wide-resources
       ```

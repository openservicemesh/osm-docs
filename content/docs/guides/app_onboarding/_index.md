---
title: "Application onboarding"
description: "Onboard Services"
type: docs
weight: 6
---

# Onboard Services
The following guide describes how to onboard a Kubernetes microservice to an OSM instance.

1. Refer to the [application requirements](/docs/guides/app_onboarding/prereqs) guide before onboarding applications.

1. Configure and Install [Service Mesh Interface (SMI) policies](https://github.com/servicemeshinterface/smi-spec)

    OSM conforms to the SMI specification. By default, OSM denies all traffic communications between Kubernetes services unless explicitly allowed by SMI policies. This behavior can be overridden with the `--set=OpenServiceMesh.enablePermissiveTrafficPolicy=true` flag on the `osm install` command, allowing SMI policies not to be enforced while allowing traffic and services to still take advantage of features such as mTLS-encrypted traffic, metrics, and tracing.

    For example SMI policies, please see the following examples:
    - [demo/deploy-traffic-specs.sh](https://github.com/openservicemesh/osm/blob/release-v1.0/demo/deploy-traffic-specs.sh)
    - [demo/deploy-traffic-split.sh](https://github.com/openservicemesh/osm/blob/release-v1.0/demo/deploy-traffic-split.sh)
    - [demo/deploy-traffic-target.sh](https://github.com/openservicemesh/osm/blob/release-v1.0/demo/deploy-traffic-target.sh)

    If an application in the mesh needs to communicate with the Kubernetes API server, the user needs to explicitly add the Kubernetes API server's address to the list of Global outbound IP ranges for exclusion. The IP address could be a cluster IP address or a public IP address and should be appropriately excluded for connectivity to the Kubernetes API server. For example:

    1. Get the Kubernetes API server cluster IP:
        ```console
        $ kubectl get svc -n default
        NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
        kubernetes   ClusterIP   10.0.0.1     <none>        443/TCP   1d
        ```
    2. Add this IP to the MeshConfig so that outbound traffic to it is excluded from interception by OSM's sidecar
        ```console
        $ kubectl patch meshconfig osm-mesh-config -n kube-system -p '{"spec":{"traffic":{"outboundIPRangeExclusionList":["10.0.0.1/32"]}}}'  --type=merge
        meshconfig.config.openservicemesh.io/osm-mesh-config patched
        ```

1. Onboard Kubernetes Namespaces to OSM

    To onboard a namespace containing applications to be managed by OSM, run the `osm namespace add` command:

    ```console
    $ osm namespace add <namespace> --mesh-name <mesh-name>
    ```

    By default, the `osm namespace add` command enables automatic sidecar injection for pods in the namespace.

    To disable automatic sidecar injection as a part of enrolling a namespace into the mesh, use `osm namespace add <namespace> --disable-sidecar-injection`.
    Once a namespace has been onboarded, pods can be enrolled in the mesh by configuring automatic sidecar injection. See the [Sidecar Injection](/docs/guides/app_onboarding/sidecar_injection) document for more details.

1.  Deploy new applications or redeploy existing applications

    By default, new deployments in onboarded namespaces are enabled for automatic sidecar injection. This means that when a new Pod is created in a managed namespace, OSM will automatically inject the sidecar proxy to the Pod.
    Existing deployments need to be restarted so that OSM can automatically inject the sidecar proxy upon Pod re-creation. Pods managed by a Deployment can be restarted using the `kubectl rollout restart deploy` command.

    In order to route protocol specific traffic correctly to service ports, configure the application protocol to use. Refer to the [application protocol selection guide](/docs/guides/app_onboarding/app_protocol_selection) to learn more.

#### Note: Removing Namespaces
Namespaces can be removed from the OSM mesh with the `osm namespace remove` command:

```console
$ osm namespace remove <namespace>
```

> **Please Note:**
> The **`osm namespace remove`** command only tells OSM to stop applying updates to the sidecar proxy configurations in the namespace. It **does not** remove the proxy sidecars. This means the existing proxy configuration will continue to be used, but it will not be updated by the OSM control plane. If you wish to remove the proxies from all pods, remove the pods' namespaces from the OSM mesh with the CLI and reinstall all the pod workloads.

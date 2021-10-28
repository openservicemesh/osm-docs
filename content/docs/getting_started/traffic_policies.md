---
title: "Configure Traffic Policies"
description: "Configure traffic to flow between the bookstore applications"
type: docs
weight: 3
---

# Traffic Policies

## Traffic Policy Modes

Once the applications are up and running, they can interact with each other using [permissive traffic policy mode](#permissive-traffic-policy-mode) or [SMI traffic policy mode](#smi-traffic-policy-mode). In permissive traffic policy mode, traffic between application services is automatically configured by `osm-controller`, and SMI policies are not enforced. In the SMI policy mode, all traffic is denied by default unless explicitly allowed using a combination of SMI access and routing policies.

### Traffic Encryption

All traffic is encrypted via mTLS regardless of whether you're using access control policies or have enabled permissive traffic policy mode.

### How to Check Traffic Policy Mode

Check whether permissive traffic policy mode is enabled or not by retrieving the value for the `enablePermissiveTrafficPolicyMode` key in the `osm-mesh-config` `MeshConfig` resource.

```bash
# Replace osm-system with osm-controller's namespace if using a non default namespace
kubectl get meshconfig osm-mesh-config -n osm-system -o jsonpath='{.spec.traffic.enablePermissiveTrafficPolicyMode}{"\n"}'
# Output:
# false: permissive traffic policy mode is disabled, SMI policy mode is enabled
# true: permissive traffic policy mode is enabled, SMI policy mode is disabled
```

The following sections demonstrate using OSM with [permissive traffic policy mode](#permissive-traffic-policy-mode) and [SMI Traffic Policy Mode](#smi-traffic-policy-mode).

## Permissive Traffic Policy Mode

In permissive traffic policy mode, application connectivity within the mesh is automatically configured by `osm-controller`. It can be enabled in the following ways.

1. During install using `osm` CLI:
  ```bash
  osm install --set=osm.enablePermissiveTrafficPolicy=true
  ```

1. Post install by patching the `osm-mesh-config` custom resource in the control plane's namespace (`osm-system` by default)
  ```bash
  kubectl patch meshconfig osm-mesh-config -n osm-system -p '{"spec":{"traffic":{"enablePermissiveTrafficPolicyMode":true}}}'  --type=merge
  ```

### Verify OSM is in permissive traffic policy mode

Before proceeding, [verify the traffic policy mode](#verify-the-traffic-policy-mode) and ensure the `enablePermissiveTrafficPolicyMode` key is set to `true` in the `osm-mesh-config` `MeshConfig` resource. Refer to the section above to enable permissive traffic policy mode.

In step [Deploy the Bookstore Application](#deploy-the-bookstore-application), we have already deployed the applications needed to verify traffic flow in permissive traffic policy mode. The `bookstore` service we previously deployed is encoded with an identity of `bookstore-v1` for demo purpose, as can be seen in the [Deployment's manifest](https://raw.githubusercontent.com/openservicemesh/osm/{{< param osm_branch >}}/docs/example/manifests/apps/bookstore.yaml). The identity reflects which counter increments in the `bookbuyer` and `bookthief` UI, and the identity displayed in the `bookstore` UI.

The counter in the `bookbuyer`, `bookthief` UI for the books bought and stolen respectively from `bookstore v1` should now be incrementing:

- [http://localhost:8080](http://localhost:8080) - **bookbuyer**
- [http://localhost:8083](http://localhost:8083) - **bookthief**

The counter in the `bookstore` UI for the books sold should also be incrementing:

- [http://localhost:8084](http://localhost:8084) - **bookstore**

The `bookbuyer` and `bookthief` applications are able to buy and steal books respectively from the newly deployed `bookstore` application because permissive traffic policy mode is enabled, thereby allowing connectivity between applications without the need for SMI traffic access policies.

This can be demonstrated further by disabling permissive traffic policy mode and verifying that the counter for books bought from `bookstore` is not incrementing anymore:

```bash
kubectl patch meshconfig osm-mesh-config -n osm-system -p '{"spec":{"traffic":{"enablePermissiveTrafficPolicyMode":false}}}'  --type=merge
```

_Note: When you disable permissive traffic policy mode, SMI traffic access mode is implicitly enabled. If counters for the books are incrementing then it could be because some SMI Traffic Access policies have been applied previously to allow such traffic._

## SMI Traffic Policy Mode

SMI traffic policies can be used for the following:

1. SMI access control policies to authorize traffic access between service identities
1. SMI traffic specs policies to define routing rules to associate with access control policies
1. SMI traffic split policies to direct client traffic to multiple backends based on weights

The following sections describe how to leverage each of these policies to enforce fine grained control over traffic flowing within the service mesh. Before proceeding, [verify the traffic policy mode](#verify-the-traffic-policy-mode) and ensure the `enablePermissiveTrafficPolicyMode` key is set to `false` in the `osm-mesh-config` `MeshConfig` resource.

SMI traffic policy mode can be enabled by disabling permissive traffic policy mode:

```bash
kubectl patch meshconfig osm-mesh-config -n osm-system -p '{"spec":{"traffic":{"enablePermissiveTrafficPolicyMode":false}}}'  --type=merge
```

### Deploy SMI Access Control Policies

At this point, applications do not have access to each other because no access control policies have been applied. Confirm this by verifying that none of the counters in the `bookbuyer`, `bookthief`, `bookstore`, and `bookstore-v2` UI are incrementing.

Apply the [SMI Traffic Target][https://github.com/servicemeshinterface/smi-spec/blob/v0.6.0/apis/traffic-access/v1alpha2/traffic-access.md) and [SMI Traffic Specs](https://github.com/servicemeshinterface/smi-spec/blob/v0.6.0/apis/traffic-specs/v1alpha4/traffic-specs.md) resources to define access control and routing policies for the applications to communicate:

 Deploy SMI TrafficTarget and HTTPRouteGroup policy:

```bash
kubectl apply -f https://raw.githubusercontent.com/openservicemesh/osm/{{< param osm_branch >}}/docs/example/manifests/access/traffic-access-v1.yaml
```

The counters should now be incrementing for the `bookbuyer`, and `bookstore` applications:

- [http://localhost:8080](http://localhost:8080) - **bookbuyer**
- [http://localhost:8084](http://localhost:8084) - **bookstore**

Note that the counter is _not_ incrementing for the `bookthief` application:

- [http://localhost:8083](http://localhost:8083) - **bookthief**

That is because the SMI Traffic Target SMI HTTPRouteGroup resources deployed only allow `bookbuyer` to communicate with `bookstore`.

#### Allowing the Bookthief Application to access the Mesh

Currently the Bookthief application has not been authorized to participate in the service mesh communication. We will now update the TrafficTarget to allow `bookthief` to communicate with `bookstore`.

Current TrafficTarget spec without `bookthief` listed in `spec.sources`:

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

Updated TrafficTarget spec with `bookthief` in `spec.sources`:

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

Apply the updated TrafficTarget:

```bash
kubectl apply -f https://raw.githubusercontent.com/openservicemesh/osm/{{< param osm_branch >}}/docs/example/manifests/access/traffic-access-v1-allow-bookthief.yaml
```

The counter in the `bookthief` window will start incrementing.

- [http://localhost:8083](http://localhost:8083) - **bookthief**

Apply the original Traffic Target object without the bookthief listed as an allowed source:

```bash
kubectl apply -f https://raw.githubusercontent.com/openservicemesh/osm/{{< param osm_branch >}}/docs/example/manifests/access/traffic-access-v1.yaml
```

The counter in the `bookthief` window will stop incrementing.

- [http://localhost:8083](http://localhost:8083) - **bookthief**

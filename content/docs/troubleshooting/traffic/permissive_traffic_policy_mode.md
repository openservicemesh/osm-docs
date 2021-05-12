---
title: "Permissive Traffic Policy Mode Troubleshooting"
description: "Permissive Traffic Policy Mode Troubleshooting Guide"
type: docs
aliases: ["permissive_traffic_policy_mode.md"]
---

## When permissive traffic policy mode is not working as expected

### 1. Confirm Permissive traffic policy mode is enabled

Confirm permissive traffic policy mode is enabled by verifying the value for the `enablePermissiveTrafficPolicyMode` key in the `osm-mesh-config` custom resource. `osm-mesh-config` ConfigMap resides in the namespace OSM control plane namespace, `osm-system` by default.

```console
# Returns true if permissive traffic policy mode is enabled
$ kubectl get meshconfig osm-mesh-config -n osm-system -o jsonpath='{.spec.traffic.enablePermissiveTrafficPolicyMode}{"\n"}'
true
```

The above command must return a boolean string (`true` or `false`) indicating if permissive traffic policy mode is enabled.

### 2. Inspect OSM controller logs for errors

```bash
# When osm-controller is deployed in the osm-system namespace
kubectl logs -n osm-system $(kubectl get pod -n osm-system -l app=osm-controller -o jsonpath='{.items[0].metadata.name}')
```

Errors will be logged with the `level` key in the log message set to `error`:
```console
{"level":"error","component":"...","time":"...","file":"...","message":"..."}
```

### 3. Confirm the Envoy configuration

Confirm the Envoy proxy configuration on the client and server pods are allowing the client to access the server. Refer to the [sample configurations](../../../tasks_usage/traffic_management/permissive_traffic_policy_mode#envoy-configurations) to verify that the client has valid routes programmed to access the server.

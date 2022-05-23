---
title: "Egress"
description: "Troubleshooting egress (external) traffic"
type: docs
weight: 4
---

# Troubleshooting Egress Traffic

## When Egress is not working as expected

### 1. Confirm egress is enabled

Confirm egress is enabled by verifying the value for the `enableEgress` key in the `osm-mesh-config` `MeshConfig` custom resource. `osm-mesh-config` resides in the namespace OSM control plane namespace (`osm-system` by default).

```console
# Returns true if egress is enabled
$ kubectl get meshconfig osm-mesh-config -n osm-system -o jsonpath='{.spec.traffic.enableEgress}{"\n"}'
true
```

The above command must return a boolean string (`true` or `false`) indicating if egress is enabled.

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

Confirm the Envoy proxy configuration on the client has a default egress filter chain on the outbound listener. Refer to the [sample configurations](../../../tasks/traffic_management/egress#envoy-configurations) to verify that the client is configured to have outbound access to external destinations.

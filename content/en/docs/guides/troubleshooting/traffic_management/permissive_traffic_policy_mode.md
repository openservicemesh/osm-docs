---
title: "Permissive Traffic Policy Mode"
description: "Troubleshooting permissive traffic policy"
type: docs
weight: 2
---

# Troubleshooting Permissive Traffic Policy Mode

## When permissive traffic policy mode is not working as expected

### 1. Confirm permissive traffic policy mode is enabled

Confirm permissive traffic policy mode is enabled by verifying the value for the `enablePermissiveTrafficPolicyMode` key in the `osm-mesh-config` custom resource. `osm-mesh-config` MeshConfig resides in the namespace OSM control plane namespace (`osm-system` by default).

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

Use the `osm verify connectivity` command to validate that the pods can communicate using a Kubernetes service.

For example, to verify if the pod `curl-7bb5845476-zwxbt` in the namespace `curl` can direct traffic to the pod `httpbin-69dc7d545c-n7pjb` in the `httpbin` namespace using the `httpbin` Kubernetes service:

```console
$ osm verify connectivity --from-pod curl/curl-7bb5845476-zwxbt --to-pod httpbin/httpbin-69dc7d545c-n7pjb --to-service httpbin
---------------------------------------------
[+] Context: Verify if pod "curl/curl-7bb5845476-zwxbt" can access pod "httpbin/httpbin-69dc7d545c-n7pjb" for service "httpbin/httpbin"
Status: Success

---------------------------------------------
```

The `Status` field in the output will indicate `Success` when the verification succeeds.

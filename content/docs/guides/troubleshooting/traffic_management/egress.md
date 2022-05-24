---
title: "Egress"
description: "Troubleshooting egress (external) traffic"
type: docs
weight: 4
---

# Troubleshooting Egress Traffic

## When Egress is not working as expected

### 1. Confirm egress is enabled

If relying on passthrough egress functionality to unknown destinations, confirm that global passthrough egress is enabled by verifying the value for the `enableEgress` key in the `osm-mesh-config` `MeshConfig` custom resource. `osm-mesh-config` resides in the OSM control plane namespace (`osm-system` by default).

```console
# Returns true if global passthrough egress is enabled
$ kubectl get meshconfig osm-mesh-config -n osm-system -o jsonpath='{.spec.traffic.enableEgress}{"\n"}'
true
```

If using [Egress policy](/docs/guides/traffic_management/egress/#1-configuring-egress-policies), confirm that egress policy capability is enabled.

```console
# Returns true if egress policy capability is enabled
$ kubectl get meshconfig osm-mesh-config -n osm-system -o jsonpath='{.spec.featureFlags.enableEgressPolicy}{"\n"}'
true
```

### 2. Inspect OSM controller logs for errors

```bash
# When osm-controller is deployed in the osm-system namespace
kubectl logs -n osm-system $(kubectl get pod -n osm-system -l app=osm-controller -o jsonpath='{.items[0].metadata.name}')
```

Errors will be logged with the `level` key in the log message set to `error`:
```console
{"level":"error","component":"...","time":"...","file":"...","message":"..."}
```

### 3. Confirm the Envoy configuration when using Egress policy

Use the `osm verify connectivity` command to validate that the pod can communicate with the external host and port using an Egress policy.

Examples:

To verify if the pod `curl-7bb5845476-zwxbt` in the namespace `curl` can direct HTTPS traffic to the the external `httpbin.org` host on port `443`:

```console
$ osm verify connectivity --from-pod curl/curl-7bb5845476-zwxbt --to-ext-port 443 --to-ext-host httpbin.org --app-protocol https
---------------------------------------------
[+] Context: Verify if pod "curl/curl-7bb5845476-zwxbt" can access external service on port 443
Status: Success

---------------------------------------------
```

To verify if the pod `curl-7bb5845476-zwxbt` in the namespace `curl` can direct HTTP traffic to the the external `httpbin.org` host on port `80`:
```console
$ osm verify connectivity --from-pod curl/curl-7bb5845476-zwxbt --to-ext-port 80 --to-ext-host httpbin.org --app-protocol http
---------------------------------------------
[+] Context: Verify if pod "curl/curl-7bb5845476-zwxbt" can access external service on port 80
Status: Success

---------------------------------------------
```

The `Status` field in the output will indicate `Success` when the verification succeeds.

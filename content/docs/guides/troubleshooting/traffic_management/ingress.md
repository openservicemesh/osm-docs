---
title: "Ingress"
description: "Troubleshooting ingress traffic"
type: docs
weight: 3
---

# Troubleshooting Ingress Traffic

## When Ingress is not working as expected

### 1. Confirm global ingress configuration is set as expected.

```console
# Returns true if HTTPS ingress is enabled
$ kubectl get meshconfig osm-mesh-config -n osm-system -o jsonpath='{.spec.traffic.useHTTPSIngress}{"\n"}'
false
```

If the output of this command is `false` this means that HTTP ingress is enabled and HTTPS ingress is disabled. To disable HTTP ingress and enable HTTPS ingress, use the following command:

```bash
# Replace osm-system with osm-controller's namespace if using a non default namespace
kubectl patch meshconfig osm-mesh-config -n osm-system -p '{"spec":{"traffic":{"useHTTPSIngress":true}}}'  --type=merge
```

Likewise, to enable HTTP ingress and disable HTTPS ingress, run:

```bash
# Replace osm-system with osm-controller's namespace if using a non default namespace
kubectl patch meshconfig osm-mesh-config -n osm-system -p '{"spec":{"traffic":{"useHTTPSIngress":false}}}'  --type=merge
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

### 3. Confirm that the ingress resource has been successfully deployed

```bash
kubectl get ingress <ingress-name> -n <ingress-namespace>
```

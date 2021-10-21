---
title: "Uninstall Troubleshooting"
description: "OSM Mesh Uninstall Troubleshooting Guide"
aliases: ["/docs/troubleshooting/uninstall"]
type: docs
---

# OSM Mesh Uninstall Troubleshooting Guide

If for any reason, `osm uninstall mesh` as documented in the [uninstall guide](/docs/guides/uninstall/) fails, you may manually delete OSM resources as detailed below.

Set environment variables for your mesh:
```console
export osm_namespace=<osm namespace>
export mesh_name=<mesh name>
export osm_version=<osm version>
export osm_ca_bundle=<osm ca bundle>
```

```console
kubectl delete secret -n $osm_namespace $osm_ca_bundle mutating-webhook-cert-secret validating-webhook-cert-secret crd-converter-cert-secret
kubectl delete meshconfig -n $osm_namespace
kubectl delete namespace $osm_namespace
kubectl delete mutatingwebhookconfiguration -l app.kubernetes.io/name=openservicemesh.io,app.kubernetes.io/instance=$mesh_name,app.kubernetes.io/version=$osm_version,app=osm-injector
kubectl delete validatingwebhookconfiguration -l app.kubernetes.io/name=openservicemesh.io,app.kubernetes.io/instance=mesh_name,app.kubernetes.io/version=$osm_version,app=osm-controller
```

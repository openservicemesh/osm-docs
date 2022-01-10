---
title: "Uninstall OSM"
description: "Uninstall OSM and the bookstore applications"
type: docs
weight: 6
---

# Uninstall OSM

The articles in the getting started section outline installing OSM and sample applications. To uninstall all of these resources, delete the sample application and related SMI resources and uninstall the OSM control plane and cluster-wide OSM resources.

## Delete the sample applications

To clean up the sample applications and related SMI resources, delete their namespaces. For example:

```bash
kubectl delete ns bookbuyer bookthief bookstore bookwarehouse
```

## Uninstall OSM control plane

To uninstall OSM control plane, use `osm unistall mesh`.

```bash
osm uninstall mesh
```

## Uninstall OSM cluster-wide resources

To uninstall OSM cluster-wide resources, use `osm uninstall cluster-wide-resources`.

```bash
osm uninstall cluster-wide-resources
```

For more details about uninstalling OSM, see the [uninstallation guide](/docs/guides/uninstall/).

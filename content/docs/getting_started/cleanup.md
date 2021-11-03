---
title: "Cleanup"
description: "Uninstall OSM and the bookstore applications"
type: docs
weight: 6
---

# Cleanup

To cleanup all resources created for the demo, the OSM control plane, SMI resources, and the sample applications need to be deleted.

To uninstall the sample applications and SMI resources, delete their namespaces with the following command:
```bash
kubectl delete ns bookbuyer bookthief bookstore bookwarehouse
```

### Uninstall OSM control plane

To uninstall OSM control plane, run the following command.
```bash
osm uninstall mesh
```

### Uninstall OSM cluster wide resources

To uninstall OSM cluster wide resources, run the following command.
```bash
osm uninstall cluster-wide-resources
```

For more details about uninstalling OSM, see the [uninstallation guide](/docs/guides/uninstall/).

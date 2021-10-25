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

To uninstall OSM, run
```bash
osm uninstall mesh
```

For more details about uninstalling OSM, see the [uninstallation guide](/docs/guides/uninstall/).

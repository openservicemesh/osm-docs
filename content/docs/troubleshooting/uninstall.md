---
title: "Uninstall Troubleshooting"
description: "OSM Mesh Uninstall Troubleshooting Guide"
type: docs
---

# OSM Mesh Uninstall Troubleshooting Guide

## Unsuccessful Uninstall

If for any reason, `osm uninstall` is unsuccessful, run the [cleanup script](https://github.com/openservicemesh/osm/blob/release-v0.8/scripts/cleanup/osm-cleanup.sh) to delete any OSM-related resources.

To run this script: 

1. Create a `.env` environment variable file to set the values specified at the top of the script. These values should match the values used to deploy the mesh.

2. In the root directory of the local osm repository, run:

```console
./scripts/cleanup/osm-cleanup.sh
```

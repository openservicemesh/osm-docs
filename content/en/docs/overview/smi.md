---
title: Service Mesh Interface (SMI) Support
description: "SMI implementation in OSM"
type: docs
---

## Overview

Open Service Mesh (OSM) implements [Service Mesh Interface (SMI)](https://smi-spec.io/) resources. This allows OSM users to have flexible implementations of common service mesh scenarios.

## Supported versions

To find out what versions of SMI resources are supported in your mesh, you can run `osm mesh list`. Here is an excerpt from sample output:

```
MESH NAME   MESH NAMESPACE   SMI SUPPORTED
osm         osm-system       HTTPRouteGroup:v1alpha4,TCPRoute:v1alpha4,TrafficSplit:v1alpha2,TrafficTarget:v1alpha3
```

The following are the currently supported SMI resources and their versions:

| SMI resource | Version |
|--------------|---------|
| TrafficTarget | access.smi-spec.io/v1alpha3 |
| TrafficSplit | split.smi-spec.io/v1alpha2 |
| HTTPRouteGroup | specs.smi-spec.io/v1alpha4 |
| TCPRoute | specs.smi-spec.io/v1alpha4 |

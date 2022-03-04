---
title: "Open Service Mesh Documentation"
description: ""
type: docs

---

A simple, complete, and standalone service mesh.

OSM runs on [Kubernetes](https://kubernetes.io/). The OSM control plane implements [Envoy's xDS](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/operations/dynamic_configuration) and is configured with [SMI](https://smi-spec.io/) APIs. OSM injects an Envoy proxy as a sidecar container next to each instance of an application.

To learn more about OSM:
* Go through all the [getting started articles](/docs/getting_started/) to install OSM and run a sample application.
* Read an [overview of OSM](/docs/overview/about/) and more about [the design of its components](/docs/overview/osm_components/).
* Run additional example scenarios from the [Demos](/docs/demos/) section.
* Consider using the [OSM add-on](https://docs.microsoft.com/azure/aks/open-service-mesh-about) with an [Azure Kubernetes Service cluster](https://docs.microsoft.com/azure/aks).
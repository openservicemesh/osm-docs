---
title: "Integrate Flagger with OSM"
description: "A simple demo showing how OSM integrates with Flagger"
aliases: "/docs/guides/integrations/demo_flagger"
type: docs
weight: 4
---

# Integrate Flagger with OSM

To add additional automation around OSM's use of the [Service Mesh Interface](smi-spec.io) [Traffic Split](https://github.com/servicemeshinterface/smi-spec/blob/v0.6.0/apis/traffic-split/v1alpha4/traffic-split.md) functionality, OSM has provided integration with the [Flagger](https://www.weave.works/oss/flagger/) project developed by [WeaveWorks](https://www.weave.works/).

"Flagger is a progressive delivery tool that automates the release process for applications running on Kubernetes. It reduces the risk of introducing a new software version in production by gradually shifting traffic to the new version while measuring metrics and running conformance tests." [[1]](#1)

## Progressive Delivery with Flagger for OSM

With our collaboration with the Flagger project, documentation for how to use OSM with Flagger will reside on the Flagger site. Please visit the the [Open Service Mesh Progressive Delivery Flagger documentation](https://docs.flagger.app/tutorials/osm-progressive-delivery) for details on how to install. The OSM/Flagger integration code can be found at the [Flagger GitHub repository](https://github.com/fluxcd/flagger). If you experience any issues with the integration, please submit issues to the [Flagger GitHub issues repository](https://github.com/fluxcd/flagger/issues).

## References

<a id="1">[1]</a>
WeaveWorks/Flagger
What is Flagger?

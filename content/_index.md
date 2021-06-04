---
title: "Docs"
description: "Open Service Mesh documentation and resources."
type: docs

---

## What is OSM?

Open Service Mesh (OSM) runs an Envoy based control plane on Kubernetes, can be configured with SMI APIs, and works by injecting an Envoy proxy as a sidecar container next to each instance of your application. The proxy contains and executes rules around access control policies, implements routing configuration, and captures metrics. The control plane continually configures proxies to ensure policies and routing rules are up to date and ensures proxies are healthy.

OSM is designed with the following core principles:
* Simple to understand and contribute.
* Effortless to install, maintain, and operate.
* Painless to troubleshoot.
* Easy to configure via Service Mesh Interface (SMI).

## Features
* Easily and transparently configure traffic shifting for deployments
* Secure service to service communication by enabling mutual TLS
* Define and execute fine grained access control policies for services
* Observability and insights into application metrics for debugging and monitoring services
* Integrate with external certificate management services/solutions with a pluggable interface
* Onboard applications onto the mesh by enabling automatic sidecar injection of Envoy proxy

## Install

The `osm` CLI is the simplest way to install OSM on your Kubernetes cluster. To install the `osm` CLI, download and unpack the `osm` binary from the [Releases page](https://github.com/openservicemesh/osm/releases) and add it to `$PATH`. For more details on installing the `osm` CLI, see [Installation](/docs/install/).

To install OSM on your cluster using the `osm` CLI, use the `install` command. For example:

```shell
$ osm install
```

Alternatively, you can install OSM on your cluster using a Helm chart. You can also use the `osm` CLI to install OSM on OpenShift. For more details on installation options, see [Install OSM](/docs/install/#install-osm).

## Next steps

To run a sample application or see examples of what you can do with OSM, see [All OSM demos](/docs/getting_started/demos/).
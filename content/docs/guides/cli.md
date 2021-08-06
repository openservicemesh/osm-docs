---
title: "OSM CLI"
description: "This section describes installing and using the `osm` CLI."
type: docs
weight: 1
---

## Prerequisites

- Kubernetes cluster running Kubernetes v1.19.0 or greater

## Set up the OSM CLI

### From the Binary Releases

Download platform specific compressed package from the [Releases page](https://github.com/openservicemesh/osm/releases).
Unpack the `osm` binary and add it to `$PATH` to get started.

### From Source (Linux, MacOS)

Building OSM from source requires more steps but is the best way to test the latest changes and useful in a development environment.

You must have a working [Go](https://golang.org/doc/install) environment.

```console
$ git clone git@github.com:openservicemesh/osm.git
$ cd osm
$ make build-osm
```

`make build-osm` will fetch any required dependencies, compile `osm` and place it in `bin/osm`. Add `bin/osm` to `$PATH` so you can easily use `osm`.

## Install OSM

### OSM Configuration
By default, the control plane components are installed into a Kubernetes Namespace called `osm-system` and the control plane is given a unique identifier attribute `mesh-name` defaulted to `osm`. Both the Namespace and mesh-name can be configured through flags when using the `osm CLI` flags or by editing the values file when using the `helm CLI`.

The `mesh-name` is a unique identifier assigned to an osm-controller instance during install to identify and manage a mesh instance.

The `mesh-name` should follow [RFC 1123](https://tools.ietf.org/html/rfc1123) DNS Label constraints. The `mesh-name` must:
- contain at most 63 characters
- contain only lowercase alphanumeric characters or '-'
- start with an alphanumeric character
- end with an alphanumeric character

### Using the OSM CLI
Use the `osm` CLI to install the OSM control plane on to a Kubernetes cluster.

Run `osm install`.

```console
# Install osm control plane components
$ osm install
OSM installed successfully in namespace [osm-system] with mesh name [osm]
```

Run `osm install --help` for more options.


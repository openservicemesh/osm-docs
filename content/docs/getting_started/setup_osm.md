---
title: "Setup OSM"
description: "Install the OSM control plane using the OSM CLI"
type: docs
weight: 1
---

# Setup OSM

## Prerequisites
This demo of OSM v1.0.0 requires:
  - a cluster running Kubernetes v1.19 or greater (using a cloud provider of choice, [minikube](https://minikube.sigs.k8s.io/docs/start/), or similar)
  - a workstation capable of executing [Bash](https://en.wikipedia.org/wiki/Bash_(Unix_shell)) scripts
  - [The Kubernetes command-line tool](https://kubernetes.io/docs/tasks/tools/#kubectl) - `kubectl`
  - the [OSM code repo](https://github.com/openservicemesh/osm/) available locally

> Note: This document assumes you have already installed credentials for a Kubernetes cluster in ~/.kube/config and `kubectl cluster-info` executes successfully.



## Download and install the OSM command-line tool

The `osm` command-line tool contains everything needed to install and configure Open Service Mesh.
The binary is available on the [OSM GitHub releases page](https://github.com/openservicemesh/osm/releases/).

### For GNU/Linux and macOS

Download the 64-bit GNU/Linux or macOS binary of OSM v1.0.0:
```bash
system=$(uname -s)
release=v1.0.0
curl -L https://github.com/openservicemesh/osm/releases/download/${release}/osm-${release}-${system}-amd64.tar.gz | tar -vxzf -
./${system}-amd64/osm version
```

### For Windows

Download the 64-bit Windows OSM v1.0.0 binary via Powershell:
```powershell
wget  https://github.com/openservicemesh/osm/releases/download/v1.0.0/osm-v1.0.0-windows-amd64.zip -o osm.zip
unzip osm.zip
.\windows-amd64\osm.exe version
```

The `osm` CLI can be compiled from source using [this guide](/docs/install/).



## Installing OSM on Kubernetes

With the `osm` binary downloaded and unzipped, we are ready to install Open Service Mesh on a Kubernetes cluster:

The command below shows how to install OSM on your Kubernetes cluster.
This command enables
[Prometheus](https://github.com/prometheus/prometheus),
[Grafana](https://github.com/grafana/grafana), and
[Jaeger](https://github.com/jaegertracing/jaeger) integrations.
The `OpenServiceMesh.enablePermissiveTrafficPolicy` chart parameter in the `values.yaml` file instructs OSM to ignore any policies and
let traffic flow freely between the pods. With Permissive Traffic Policy mode enabled, new pods
will be injected with Envoy, but traffic will flow through the proxy and will not be blocked.

> Note: Permissive Traffic Policy mode is an important feature for brownfield deployments, where it may take some time to craft SMI policies. While operators design the SMI policies, existing services will continue to operate as they have been before OSM was installed.

```bash
osm install \
    --set=OpenServiceMesh.enablePermissiveTrafficPolicy=true \
    --set=OpenServiceMesh.deployPrometheus=true \
    --set=OpenServiceMesh.deployGrafana=true \
    --set=OpenServiceMesh.deployJaeger=true
```

This installed OSM Controller in the `osm-system` namespace.


Read more on OSM's integrations with Prometheus, Grafana, and Jaeger in the [observability documentation](/docs/guides/observability/).

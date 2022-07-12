---
title: "Integrate Pyroscope with OSM"
description: "A simple demo showing how OSM integrates with Pyroscope for continuous profiling"
aliases: "/docs/integrations/demo_pyroscope"
type: docs
weight: 5
---

# Integrate Pyroscope with OSM

## What is Pyroscope

Pyroscope is a continuous profiling tool which profiles the performance of a program written in variety of languages, GoLang included. It provides a great Web UI for reading the profiling results.

When a GoLang program exposes the web API endpoint provided by the built-in `net/http/pprof` library, Pyroscope can directly pull profiling results from the given endpoint. OSM exposes the required endpoint when debug server feature is enabled. This approach allows the integration of Pyroscope with no change to the OSM codebase.

## Enable OSM debug server

To enable OSM debug server at installation time, simply add `--set=osm.enableDebugServer=true` argument to the `osm install` command, or run `kubectl patch meshconfig osm-mesh-config -n "$OSM_NAMESPACE" -p '{"spec":{"observability":{"enableDebugServer":true}}}' --type=merge` if OSM is already installed in the cluster.

## Install Pyroscope

In this section, we will install Pyroscope to profile OSM control plane. Currently only *osm-controller* exposes pprof web endpoints.

First step is to add annotation to the `osm-controller` service so that Pyroscope monitors it.

```bash
kubectl annotate -n "$OSM_NAMESPACE" svc osm-controller  \
    "pyroscope.io/scrape=true" \
    "pyroscope.io/application-name=osm-controller" \
    "pyroscope.io/spy-name=gospy" \
    "pyroscope.io/profile-cpu-enabled=true" \
    "pyroscope.io/profile-mem-enabled=true" \
    "pyroscope.io/port=9092"
```

Replace `$OSM_NAMESPACE` with the OSM namespace on your Kubernetes cluster. This command informs Pyroscope to profile both CPU and memory usages of the service. The pprof web endpoint is exposed on port 9092.

Next we can install Pyroscope with Helm.

```bash
helm install prof pyroscope \
    -f https://raw.githubusercontent.com/openservicemesh/osm-docs/main/manifests/integrations/pyroscope-values.yaml \
    --repo https://pyroscope-io.github.io/helm-chart \
    -n "$OSM_NAMESPACE"
```

This command will install a service named `prof-pyroscope` in the OSM_NAMESPACE. Some configurations of Pyroscope can be found in the [pyroscope-values.yaml](https://raw.githubusercontent.com/openservicemesh/osm-docs/main/manifests/integrations/pyroscope-values.yaml) file. You can make a copy of it on your local machine and edit the values if you want to customize the Pyroscope installation.

Check the installation by running:

```bash
kubectl get svc prof-pyroscope -n "$OSM_NAMESPACE"
```

By default, Pyroscope service exposes port 4040. You can visit the web UI after forwarding the port to your localhost:

```bash
kubectl port-forward service/prof-pyroscope 4040 -n "$OSM_NAMESPACE"
```

This command will keep the forwarded port open. Open [http://localhost:4040](http://localhost:4040) in your web browser to use Pyroscope. You should be able to see `osm-controller` from the Application dropdown list like what is shown below:

<p align="center">
  <img src="/docs/images/pyroscope-install.png" />
</p>

## Uninstall Pyroscope

To uninstall Pyroscope, run the command below:

```bash
helm uninstall prof -n "$OSM_NAMESPACE"
```

## References

* <a href="https://pyroscope.io/">Pyroscope project</a>
* <a href="https://pyroscope.io/docs/golang-pull-mode/">Pyroscope GoLang Pull Mode</a>


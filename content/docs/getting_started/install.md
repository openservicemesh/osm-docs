---
title: "Installation"
description: "This section describes how to install/uninstall Open Service Mesh (OSM) on a Kubernetes cluster"
type: docs
weight: 2
---

## Prerequisites

- Kubernetes cluster running Kubernetes v1.19.0 or greater
- The [osm CLI](/docs/guides/cli) or the [helm 3 CLI](https://helm.sh/docs/intro/install/) or the OpenShift `oc` CLI.

### Using the OSM CLI

Use the `osm` CLI to install the OSM control plane on to a Kubernetes cluster.

Run `osm install`.

```console
# Install osm control plane components
$ osm install
OSM installed successfully in namespace [osm-system] with mesh name [osm]
```

Run `osm install --help` for more options.

### Using the Helm CLI

The [OSM chart](https://github.com/openservicemesh/osm/tree/release-v0.9/charts/osm) can be installed directly via the [Helm CLI](https://helm.sh/docs/intro/install/).

#### Editing the Values File

You can configure the OSM installation by overriding the values file.

1. Create a copy of the [values file](https://github.com/openservicemesh/osm/blob/release-v0.9/charts/osm/values.yaml) (make sure to use the version for the chart you wish to install).
1. Change any values you wish to customize. You can omit all other values.

   - To see which values correspond to the MeshConfig settings, see the [OSM MeshConfig documentation](/docs/guides/mesh_config)

   - For example, to set the `logLevel` field in the MeshConfig to `info`, save the following as `override.yaml`:
     ```
     OpenServiceMesh:
       envoyLogLevel: info
     ```

#### Helm install

Then run the following `helm install` command. The chart version can be found in the Helm chart you wish to install [here](https://github.com/openservicemesh/osm/blob/release-v0.9/charts/osm/Chart.yaml#L17).

```console
$ helm install <mesh name> osm --repo https://openservicemesh.github.io/osm --version <chart version> --namespace <osm namespace> --values override.yaml
```

Omit the `--values` flag if you prefer to use the default settings.

Run `helm install --help` for more options.

### OpenShift

To install OSM on OpenShift:

1. Enable privileged init containers so that they can properly program iptables. The NET_ADMIN capability is not sufficient on OpenShift.
   ```shell
   osm install --set="OpenServiceMesh.enablePrivilegedInitContainer=true"
   ```
   - If you have already installed OSM without enabling privileged init containers, set `enablePrivilegedInitContainer` to `true` in the [OSM MeshConfig](/docs/guides/mesh_config) and restart any pods in the mesh.
1. Add the `privileged` [security context constraint](https://docs.openshift.com/container-platform/4.7/authentication/managing-security-context-constraints.html) to each service account in the mesh.
   - Install the [oc CLI](https://docs.openshift.com/container-platform/4.7/cli_reference/openshift_cli/getting-started-cli.html).
   - Add the security context constraint to the service account
     ```shell
      oc adm policy add-scc-to-user privileged -z <service account name> -n <service account namespace>
     ```

### Pod Security Policy

**Deprecated: PSP support has been deprecated in OSM since v0.10.0**

**PSP support will be removed in OSM 1.0.0**

If you are running OSM in a cluster with PSPs enabled, pass in `--set OpenServiceMesh.pspEnabled=true` to your `osm install` or `helm install` CLI command.

## Inspect OSM Components

A few components will be installed by default into the `osm-system` Namespace. Inspect them by using the following `kubectl` command:

```console
$ kubectl get pods,svc,secrets,meshconfigs,serviceaccount --namespace osm-system
```

A few cluster wide (non Namespaced components) will also be installed. Inspect them using the following `kubectl` command:

```console
kubectl get clusterrolebinding,clusterrole,mutatingwebhookconfiguration
```

Under the hood, `osm` is using [Helm](https://helm.sh) libraries to create a Helm `release` object in the control plane Namespace. The Helm `release` name is the mesh-name. The `helm` CLI can also be used to inspect Kubernetes manifests installed in more detail. Goto https://helm.sh for instructions to install Helm.

```console
$ helm get manifest osm --namespace osm-system
```

## Next Steps

Now that the OSM control plane is up and running, [add services](/docs/guides/app_onboarding/) to the mesh.

---
title: "Upgrade Guide"
description: "Upgrade Guide"
aliases: ["/docs/upgrade_guide","/docs/troubleshooting/cli/mesh_upgrade"]
type: docs
---

# Upgrade Guide

This guide describes how to upgrade the Open Service Mesh (OSM) control plane.

## How upgrades work

OSM's control plane lifecycle is managed by Helm and can be upgraded with [Helm's upgrade functionality](https://helm.sh/docs/intro/using_helm/#helm-upgrade-and-helm-rollback-upgrading-a-release-and-recovering-on-failure), which will patch or replace control plane components as needed based on changed values and resource templates.

### Resource availability during upgrade
Since upgrades may include redeploying the osm-controller with the new version, there may be some downtime of the controller. While the osm-controller is unavailable, there will be a delay in processing new SMI resources, creating new pods to be injected with a proxy sidecar container will fail, and mTLS certificates will not be rotated.

However, already existing SMI resources will be unaffected (assuming [CRD Upgrades](#CRD-Upgrades) are not needed). This means that the data plane (which includes the Envoy sidecar configs) will also be unaffected by upgrading.

Data plane interruptions are expected if the upgrade includes CRD changes. Streamlining data plane upgrades is being tracked in issue [#512](https://github.com/openservicemesh/osm/issues/512).

## Policy

Only certain upgrade paths are tested and supported.

**Note**: These plans are tentative and subject to change.

Breaking changes in this section refer to incompatible changes to the following user-facing components:
- `osm` CLI commands, flags, and behavior
- SMI CRDs and controllers

This implies the following are NOT user-facing and incompatible changes are NOT considered "breaking" as long as the incompatibility is handled by user-facing components:
- Chart values.yaml
- `osm-mesh-config` MeshConfig
- Internally-used labels and annotations (monitored-by, injection, metrics, etc.)

Upgrades are only supported between versions that do not include breaking changes, as described below.

For OSM versions `0.y.z`:
- Breaking changes will not be introduced between `0.y.z` and `0.y.z+1`
- Breaking changes may be introduced between `0.y.z` and `0.y+1.0`

For OSM versions `x.y.z` where `x >= 1`:
- Breaking changes will not be introduced between `x.y.z` and `x.y+1.0` or between `x.y.z` and `x.y.z+1`
- Breaking changes may be introduced between `x.y.z` and `x+1.0.0`

## How to upgrade OSM

The recommended way to upgrade a mesh is with the `osm` CLI. For advanced use cases, `helm` may be used.

### CRD Upgrades
Because Helm does not manage CRDs beyond the initial installation, special care needs to be taken during upgrades when CRDs are changed. Please check the `CRD Updates` section of the [release notes](https://github.com/openservicemesh/osm/releases) to see if additional steps are required to update the CRDs used by OSM. If the new release does contain updates to the CRDs, it is required to first delete existing CRDs and the associated Custom Resources prior to upgrading.

In the `./scripts/cleanup` directory we have included a helper script to delete those CRDs and Custom Resources: `./scripts/cleanup/crd-cleanup.sh`

After upgrading, the CRDs and Custom Resources will need to be recreated.

1. Checkout the tag of the [repo](https://github.com/openservicemesh/osm) corresponding to the version of the upgraded chart.
1. Install the new CRDs. (Run from the root of the repo.)
    - `kubectl apply -f charts/osm/crds/`
1. Recreate CustomResources

Improving CRD upgrades is being tracked in [#893](https://github.com/openservicemesh/osm/issues/893).

### Upgrading with the OSM CLI

**Pre-requisites**

- Kubernetes cluster with the OSM control plane installed
- `osm` CLI installed
  - By default, the `osm` CLI will upgrade to the same chart version that it installs. e.g. v0.9.0 of the `osm` CLI will upgrade to v0.9.0 of the OSM Helm chart.

The `osm mesh upgrade` command performs a `helm upgrade` of the existing Helm release for a mesh.

Basic usage requires no additional arguments or flags:
```console
$ osm mesh upgrade
OSM successfully upgraded mesh osm
```

This command will upgrade the mesh with the default mesh name in the default OSM namespace. Values from the previous release will carry over to the new release except for `OpenServiceMesh.image.registry` and `OpenServiceMesh.image.tag` which are overridden by default. For example, if OSM v0.7.0 is installed, `osm mesh upgrade` for v0.9.0 of the CLI will update the control plane images to v0.9.0 by default.

See `osm mesh upgrade --help` for more details

### Upgrading with Helm

#### Pre-requisites

- Kubernetes cluster with the OSM control plane installed
- The [helm 3 CLI](https://helm.sh/docs/intro/install/) 

#### OSM Configuration
When upgrading, any custom settings used to install or run OSM may be reverted to the default, this only includes any metrics deployments. Please ensure that you carefully follow the guide to prevent these values from being overwritten.

To preserve any changes you've made to the OSM configuration, use the `helm --values` flag. Create a copy of the [values file](https://github.com/openservicemesh/osm/blob/release-v0.9/charts/osm/values.yaml) (make sure to use the version for the upgraded chart) and change any values you wish to customize. You can omit all other values.

**Note: Any configuration changes that go into the MeshConfig will not be applied during upgrade and the values will remain as is prior to the upgrade. If you wish to update any value in the MeshConfig you can do so by patching the resource after an upgrade.

For example, if the `logLevel` field in the MeshConfig was set to `info` prior to upgrade, updating this in `override.yaml` will during an upgrade will not cause any change.

<b>Warning:</b> Do NOT change `OpenServiceMesh.meshName` or `OpenServiceMesh.osmNamespace`

#### Helm Upgrade
Then run the following `helm upgrade` command.
```console
$ helm upgrade <mesh name> osm --repo https://openservicemesh.github.io/osm --version <chart version> --namespace <osm namespace> --values override.yaml
```
Omit the `--values` flag if you prefer to use the default settings.

Run `helm upgrade --help` for more options.


## OSM Upgrade Troubleshooting Guide

### Server could not find requested resource

If the [upgrade CRD guide](/docs/upgrade_guide/#crd-upgrades) was not followed, it is possible that the installed CRDs are out of sync with the OSM controller.

The OSM controller will then crash with errors similar to this:
```
reflector.go:178] pkg/mod/k8s.io/client-go@v0.18.6/tools/cache/reflector.go:125: Failed to list *v1alpha2.TrafficTarget: the server could not find the requested resource (get traffictargets.access.smi-spec.io)
```
To resolve these errors:
1. Checkout the correct release branch of the [repo](https://github.com/openservicemesh/osm) and run the following commands from the root. 
1. Delete existing CRDs and Custom Resources (TrafficTargets, TrafficSplits, etc.)
   - `./scripts/cleanup/crd-cleanup.sh`
1. Install the new CRDs
   - `kubectl apply -f charts/osm/crds/`
1. Restart the osm-controller pod
1. Recreate CustomResources 



#### OSM Mesh Upgrade Timing Out

### Insufficient CPU
If the `osm mesh upgrade` command is timing out, it could be due to insufficient CPU.
1. Check the pods to see if any of them aren't fully up and running
```bash
# Replace osm-system with osm-controller's namespace if using a non-default namespace
kubectl get pods -n osm-system
```
2. If there are any pods that are in Pending state, use `kubectl describe` to check the `Events` section
```bash
# Replace osm-system with osm-controller's namespace if using a non-default namespace
kubectl describe pod <pod-name> -n osm-system
```

If you see the following error, then please increase the number of CPUs Docker can use.
```bash
`Warning  FailedScheduling  4s (x15 over 19m)  default-scheduler  0/1 nodes are available: 1 Insufficient cpu.`
```
#### Error Validating CLI Parameters
If the `osm mesh upgrade` command is still timing out, it could be due to a CLI/Image Version mismatch.

1. Check the pods to see if any of them aren't fully up and running
```bash
# Replace osm-system with osm-controller's namespace if using a non-default namespace
kubectl get pods -n osm-system
```
2. If there are any pods that are in Pending state, use `kubectl describe` to check the `Events` section for `Error Validating CLI parameters`
```bash
# Replace osm-system with osm-controller's namespace if using a non-default namespace
kubectl describe pod <pod-name> -n osm-system
```
3. If you find the error, please check the pod's logs for any errors
```bash
kubectl logs -n osm-system <pod-name> | grep -i error
```

If you see the following error, then it's due to a CLI/Image Version mismatch.
```bash
`"error":"Please specify the init container image using --init-container-image","reason":"FatalInvalidCLIParameters"`
```
Workaround is to set the `container-registry` and `osm-image-tag` flag when running `osm mesh upgrade`.
```bash
osm mesh upgrade --container-registry $CTR_REGISTRY --osm-image-tag $CTR_TAG --enable-egress=true
```

### Other Issues
If you're running into issues that are not resolved with the steps above, please [open a GitHub issue](https://github.com/openservicemesh/osm/issues).

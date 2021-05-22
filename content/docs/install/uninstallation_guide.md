---
title: "Uninstall"
description: "Uninstall"
type: docs
aliases: ["uninstallation_guide.md"]
weight: 3
---

# Uninstallation Guide

This guide describes how to uninstall Open Service Mesh (OSM) from a Kubernetes cluster. This guide also assumes you have a single OSM control plane (mesh) running. If you have multiple meshes in a cluster, repeat the process described below for each control plane in the cluster. Do this before uninstalling any cluster-wide resources at the end of the guide. The steps cover both the control plane and the dataplane. The goal is to walk you through uninstalling all the elements of OSM with minimal downtime.

## Prerequisites

- Kubernetes cluster with OSM installed
- The `kubectl` CLI
- The [`osm` CLI](/docs/install/#set-up-the-osm-cli) or the helm 3 CLI

## Remove Envoy Sidecars from Application Pods and Envoy Secrets

The first step is to remove the Envoy sidecar containers from the application pods. These sidecar containers enforced traffic policies. Without them, traffic would flow to and from Pods according to the default Kubernetes networking unless you applied [Kubernetes Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/).

To remove the OSM Envoy sidecars and related secrets:

1. [Disable automatic sidecar injection](#disable-automatic-sidecar-injection)
1. [Restart pods](#restart-pods)
1. [Update Ingress Resources](#update-ingress-resources)
1. [Delete Envoy bootsrap secrets](#delete-envoy-bootsrap-secrets)

### Disable Automatic Sidecar Injection

OSM Automatic Sidecar Injection is most commonly enabled by adding namespaces to the mesh via the `osm` CLI. Use the `osm` CLI to see which
namespaces have sidecar injection enabled. If you have multiple control planes installed, be sure to specify the `--mesh-name` flag.

To view the namespaces in a mesh:

```console
$ osm namespace list --mesh-name=<mesh-name>
NAMESPACE          MESH           SIDECAR-INJECTION
<namespace1>       <mesh-name>    enabled
<namespace2>       <mesh-name>    enabled
```

Remove each namespace from the mesh:

```console
$ osm namespace remove <namespace> --mesh-name=<mesh-name>
Namespace [<namespace>] successfully removed from mesh [<mesh-name>]
```

Alternatively, if you enabled sidecar injection via annotations on pods instead of per namespace, please modify the pod or deployment spec to remove the sidecar injection annotation.

### Restart Pods

Restart all pods running with a sidecar:

```console
# If pods are running as part of a Kubernetes deployment
# Can use this strategy for daemonset as well
$ kubectl rollout restart deployment <deployment-name> -n <namespace>

# If pod is running standalone (not part of a deployment or replica set)
$ kubectl delete pod <pod-name> -n namespace
$ k apply -f <pod-spec> # if pod is not restarted as part of replicaset
```

After executing these steps, there should be no OSM Envoy sidecar containers running with the applications that were once part of the mesh. The OSM control plane no longer manages traffic with the `mesh-name` used above. Note that during this process, your applications may experience some downtime as all the Pods restart.

### Update Ingress Resources

You may have ingress resources in the cluster configured to allow traffic from outside the cluster to an application that was
once part of the mesh. Identify these resources by using the command `kubectl get ingress -n <namespace>` in each namespace that was
removed from the mesh earlier.

If you have ingress resources and you have them configured to allow HTTPS traffic, you will need to update the SSL-related annotations appropriately.

Applications may be unavailable from outside the cluster for some time if you need to reconfigure ingress resources.

### Delete Envoy Bootsrap Secrets

Once you remove the sidecars, you will have no need for the Envoy bootstrap config secrets OSM created. (These are stored in the application namespace.) You can delete these manually with `kubectl`. These secrets have the prefix `envoy-bootstrap-config` followed by a unique ID: `envoy-bootstrap-config-<some-id-here>`.

### Using Helm

Run the following `helm uninstall` command:
```console
$ helm uninstall <mesh name> --namespace <osm namespace> 
```

Run `helm uninstall --help` to view more options.

## Resource Management

## Uninstall the OSM Control Plane and Remove User-Provided Resources

The OSM control plane and related components will be uninstalled in the following steps:

1. [Uninstall the OSM control plane](#uninstall-the-osm-control-plane)
1. [Remove User Provided Resources](#remove-user-provided-resources)
1. [Delete OSM Namespace](#delete-osm-namespace)

### Uninstall the OSM control plane

Use the `osm` CLI to uninstall the OSM control plane from a Kubernetes cluster. The Uninstall command removes:

1. OSM controller resources (deployment, service, config map, and RBAC)
1. Prometheus, Grafana, Jaeger, and Fluentbit resources installed by OSM
1. Mutating webhook and validating webhook

Run `osm uninstall`:

```console
# Uninstall osm control plane components
$ osm uninstall --mesh-name=<mesh-name>
Uninstall OSM [mesh name: <mesh-name>] ? [y/n]: y
OSM [mesh name: <mesh-name>] uninstalled
```

Run `osm uninstall --help` to view more options.

### Remove User-Provided Resources

If you provided or created any resources for OSM at install time, you can delete them now.

For example, if you deployed [Hashicorp Vault](/docs/tasks_usage/certificates/#installing-hashi-vault) for the sole purpose of managing certificates for OSM, you can delete all related resources.

### Delete OSM Namespace

The `osm` CLI creates a namespace for installing the control plane (if it does not already exist). However, during a mesh uninstall, the procedure does not automatically delete this namespace. This occurs because there may be resources that a user created in this namespace that the user may not want to automatically delete.

If you know that the namespace was only used for OSM and there is nothing stored there that needs to be kept around, you can delete namespace now with `kubectl`.

```console
$ kubectl delete namespace <namespace>
namespace "<namespace>" deleted
```

Repeat the steps above for each mesh installed in the cluster. Once you have removed all OSM control planes, you can move to following step.

## Remove OSM Cluster-Wide resources

OSM ensures that the Service Mesh Interface Custom Resource Definitions (CRDs) exist in the cluster at install time. If they were not already installed, the `osm` CLI will install them first before installing the rest of the control plane components. This is the same behavior when using the Helm charts to install OSM as well. CRDs are cluster-wide resources that may be used by other instances of OSM in the same cluster or other service meshes running in the same cluster. If you have no other instances of OSM or other service meshes running in the same cluster, you can remove these CRDs and instances of the SMI custom resources from the cluster using `kubectl`. After you delete the CRD, all
instances of that CRD will also be deleted.

Run these `kubectl` commands:

kubectl delete -f [https://raw.githubusercontent.com/openservicemesh/osm/release-v0.8/charts/osm/crds/access.yaml](https://raw.githubusercontent.com/openservicemesh/osm/release-v0.8/charts/osm/crds/access.yaml)
kubectl delete -f [https://raw.githubusercontent.com/openservicemesh/osm/release-v0.8/charts/osm/crds/specs.yaml](https://raw.githubusercontent.com/openservicemesh/osm/release-v0.8/charts/osm/crds/specs.yaml)
kubectl delete -f [https://raw.githubusercontent.com/openservicemesh/osm/release-v0.8/charts/osm/crds/split.yaml](https://raw.githubusercontent.com/openservicemesh/osm/release-v0.8/charts/osm/crds/split.yaml)

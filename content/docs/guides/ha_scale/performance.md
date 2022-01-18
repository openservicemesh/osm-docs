---
title: "Open Service Mesh Performance"
description: "Performance benchmark on control plane and data plane sidecar"
type: docs
---

# Performance

OSM aims to create a high-performance service mesh while providing all essential network management features. The development team benchmarked its performance and has documented the results.

## Benchmark Setup

The [Kinvolk service mesh benchmark tool](https://github.com/kinvolk/service-mesh-benchmark) was used to test OSM. This tool creates a number of Kubernetes services, with 1 pod per service. It then uses a load generator within the cluster to send requests at a given rate to services in the mesh randomly.

This benchmark is performed on a Kubernetes cluster provisioned and managed by [Microsoft Azure Kubernetes Service](https://azure.microsoft.com/en-us/services/kubernetes-service/). The cluster has 40 Standard_DS2_v2 nodes (each node has 2 vCPUs and 7GB memory). It is ensured that there is only one OSM controller pod with default resource requirements running during the benchmark. OSM is configured in non-permissive traffic mode. In the benchmark, 150 simple HTTP services were also added to OSM.

## Metrics

The metrics collected during the benchmark are:

* Control plane
    * Peak time CPU/Memory: the maximum CPU/memory usage during the whole benchmark process.
    * Stable time CPU/Memory usage: the average CPU/memory usage the control plane when it is off the peak time and gets stable.
* Data plane sidecar
    * Peak time CPU/Memory: the maximum CPU/memory usage among all sidecar containers during the whole benchmark process.
    * Stable time CPU/Memory usage: the average CPU/memory usage of all data plane sidecar containers when the usage becomes stable.
* Request latency: P50, P90, P99 and maximum of request latencies are collected.

## Process

With the setup, we benchmarked OSM with different request per second (RPS) levels within the mesh, setting it to 1000, 2000, 4000 and 8000 respectively.

Also, some variants of the benchmark have been performed, such as:

* Benchmarked on a 2x scale service mesh, which has 300 services.
* With OSM permissive traffic mode enabled.
* With OSM control plane auto scale feature enabled.

## Benchmark Result

### Control plane

#### CPU Usage

During the benchmark, the control plane uses up to 2 cores across all RPS levels. The peak usage normally happens at the beginning of the benchmark, when the control plane is busy with issuing certificates and injecting Envoy sidecars to the data plane. After that, the CPU usage will drop to almost 0 when the control plane is left idle.

#### Memory Usage

In all tested RPS levels, the control plane uses 800MB~1GB of memory at peak time and drops to around 450MB when it gets stable.

### Data plane sidecar

#### CPU Usage

Here is a chart showing the CPU usage change with the RPS levels.

<p align="center">
  <img src="/docs/images/perf/sidecar-cpu-rps.png" width="650"/>
</p>

There is a clear correlation between CPU usage and RPS. At stable time, It takes 5 millicores of vCPU, and roughly takes 4 more millicores per 6.5 more RPS going through the sidecar.

#### Memory Usage

In the benchmark, the sidecar consistently uses between 50~65MB of memory regardless of RPS level. It is believed that sidecar memory usage is more relevant to the number of connected services. In a variant test when OSM permissive traffic mode is enabled, the sidecar stores information about all 150 other services, sidecar memory usage increases by at least 500 megabytes.

### Latency

Here is a chart showing the latency with the RPS levels.

<p align="center">
  <img src="/docs/images/perf/sidecar-latency.png" width="650"/>
</p>

Here we see a clear correlation between latency and RPS, and latency increases exponentially as RPS levels. In the cases when RPS are 1000 and 2000, the latency difference is very minimal and is within the margin of error. During the benchmark, 4000 RPS is the highest level that can keep a reasonable latency. This converts to around 25 RPS per pod in the mesh.

Compared to the Kubernetes cluster without installing OSM, the sidecar adds around 40ms to the P90 latency when the request rate is at 6.5 RPS per pod.

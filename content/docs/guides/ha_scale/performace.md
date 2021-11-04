---
title: "Open Service Mesh Performance"
description: "Performance benchmark on control plane and data plane sidecar"
type: docs
---

# Performance

OSM aims to create a high performant service mesh while providing all the features. The development team has done a thorough benchmark on its performance.

## Benchmark Setup

We use the [Kinvolk service mesh benchmark tool](https://github.com/kinvolk/service-mesh-benchmark) to test OSM. In short, this tool creates a number of Kubernetes services, with 1 pod per service. It then uses a load generator within the cluster to send requests at a given rate to a service at random.

This benchmark is performed on a Kubernetes cluster provided by Microsoft Azure. The cluster has 40 Standard_DS2_v2 nodes (2 vCPUs and 7GB memory). It is ensured that there is only one OSM controller pod running during the benchmark. OSM is configured in non-permissive traffic mode.
We then added 150 simple HTTP services to OSM.

## Metrics

The metrics collected during the benchmark are:

* Control plane
    * Peak CPU/Memory: the maximum CPU/memory usage during the whole benchmark process.
    * CPU/Memory usage when stable: the CPU/memory usage the control plane when it is off the peak time and gets stable.
* Data plane sidecar:
    * Peak CPU/Memory: the maximum CPU/memory usage among all sidecar containers during the whole benchmark process.
    * Average CPU/Memory usage when stable: the average CPU/memory usage of all data plane sidecar containers when the usage becomes stable.
* Request latency: P50, P90, P99 and maximum latencies are collected.

## Process

With the setup, we benchmarked OSM with the request per second (RPS) level set to 1000, 2000, 4000 and 8000 respectively.

Also some variants of the benchmark have been performed, such as:

* A comparison with other well-known service mesh product (Istio).
* Benchmarked on a 2x scale service mesh, which has 300 services
* With OSM permissive traffic mode enabled.
* With OSM control plane auto scale feature enabled.

## Benchmark Result

### Control plane

#### CPU Usage

During the benchmark, the control plane uses up to 2 cores across all RPS levels. The Peak usage normally happens at the beginning of the benchmark, when the control plane is busy with issuing certificates and injecting Envoy sidecars to the data plane. After that, the CPU usage will drop to almost 0 when the control plane is left idle.

#### Memory Usage.

In all tested RPS levels, the control plane uses 800MB~1GB of memory at peak time, and drops to around 450MB when it gets stable.

### Data plane sidecar

#### CPU Usage

Here is a chart showing the CPU usage change with the RPS levels.

<p align="center">
  <img src="/docs/images/perf/sidecar-cpu-rps.png" width="650"/>
</p>

There is a clear correlation between CPU usage and RPS. At stable time, It takes 5 millicores of vCPU, and roughly takes 4 more millicores per 6.5 more requests going through the sidecar per second.

#### Memory Usage

In the benchmark, the sidecar consistently uses between 50~65MB of memory regardless of RPS level. It is believed that sidecar memory usage is more relevant to the number of connected services. In the variant test when OSM permissive traffic mode is enabled, the number of services in the mesh is 300, sidecar memory usage increases by several megabytes.

### Latency

Here is a chart showing the latency with the RPS levels.

<p align="center">
  <img src="/docs/images/perf/sidecar-latency.png" width="650"/>
</p>

Here we see a clear correlation between latency and RPS, and latency increases exponentially as RPS levels. In the cases when RPS is 1000 and 2000, the latency difference is very minimal and is within the margin of error. During the benchmark, 4000 RPS is the highest level that can still keep a reasonable latency. This converts to around 25 RPS per pod in the mesh.

Comparing to the Kubernetes cluster without installing OSM, the sidecar adds around 40ms to the P90 latency when the request rate is at 6.5 RPS per pod.

### Comparison with other service mesh products

The same benchmark process has been performed against Istio. The result shows that OSM out-performs Istio on all the metrics we collected at all tested RPS levels. For the control plane, OSM uses 8 cores of vCPU less at peak time, and around 500MB less memory. For the data plane, OSM uses only around 10% of the CPU and 40% of the memory compare to Istio. The P90 latency is around 50% of what Istio achieves.

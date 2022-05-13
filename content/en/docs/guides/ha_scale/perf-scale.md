---
title: "Performance and Scalability"
description: "Performance benchmark on control plane and data plane sidecar and scalability test of Open Service Mesh"
type: docs
---

Open Service Mesh aims to create a high-performance service mesh while providing all the essential service mesh features. This document describes the tests performed to evaluate the performance and scalability of OSM.

## Testing environment

This test is performed on a Kubernetes cluster provisioned and managed by [Microsoft Azure Kubernetes Service](https://azure.microsoft.com/en-us/services/kubernetes-service/). The cluster has 100 Standard_DS2_v2 nodes (each node has 2 vCPUs and 7GB memory) and 3 Standard_D32s_v3 nodes (32 vCPUs and 128GB memory) reserved for high intensity tests. It is ensured that there is only one OSM controller pod running with default resource quota during the test. OSM is configured in non-permissive traffic mode.

## Process

The testing mesh contains two types of services. One is a load generator (modified from [wrk2](https://github.com/giltene/wrk2)), which can send requests at specified rate, load balanced among all other services in the mesh. Currently we test HTTP 1 requests only. The load generator serves as the client side of a request. Throughout our test, there will be at most one load generator service. The other type of service is a simple echo application that replies with whatever it receives from an inbound request. It serves as the server side of a request. We can deploy multiple copies of the echo application in the mesh. In short, the mesh topology is a typical 1-to-N service architecture.

<p align="center">
  <img src="/docs/images/perf/mesh-topo.png" width="650"/>
</p>

## Performance

In this test, we focus on the latency overhead and resource consumption added by the OSM sidecar. We currently test the per pod RPS at 100, 500, 1000, 2000 and 4000 respectively.

### Latency

#### Test 1: Relation with RPS

In our test, regardless of the RPS level, OSM adds around 0.5ms and 5ms overhead to the 50th percentile and the 99th percentile latency respectively. This is caused by the request processing in both the client and server sidecars, for tasks like mTLS authentication, metrics collection etc. Our test setup was with a single client and server. Performance may vary with more complex applications, or with different traffic policies set on the mesh.

#### Test 2: Relation with number of connections

A similar setup to RPS scenario is used to test the relationship between latency and number of connections to the sidecar proxy. In this test, the per pod RPS is set at 2500. As the number of connections increased, the request latency showed a proportional increase while the sidecar memory usage to maintain those open connections only slightly increased. The sidecar consumes around 5MB memory for every 100 connections maintained.

<p align="center">
  <img src="/docs/images/perf/latency-conns.png" width="650"/>
</p>

### Resource Consumption

Control plane resource consumption is stable as it is not relevant to the data plane request rate.

For the sidecars on the data plane, the memory usage is relatively consistent across different RPS levels, which is around 140MB. The CPU usage has a clear increasing trend and it is nearly proportional to the RPS level. <b>In our case, use a baseline CPU of 60 millicores, the sidecar CPU consumption increases linearly around 0.1 cores for every 100 RPS.</b>

<p align="center">
  <img src="/docs/images/perf/avg-cpu.png" width="650"/>
</p>

## Scalability

In the scalability test, we explore how resource consumption changes for both the control plane and the data plane as the number of workloads in the mesh increases. A specified number of echo services will be deployed to the mesh all at the same time. Therefore, the control plane receives a load spike after the deployment starts. All the Envoy sidecars are connecting to the control plane simultaneously. After that, the control plane becomes much more idle. We deliberately create this load spike at pod rollout time in order to see the maximum scale that the mesh can handle. Each round of test will last for 15 minutes. Mesh scale of 200, 400, 800 and 1600 pods will be tested respectively. The load generator service is not used in scalability test.

### Control Plane

#### CPU Usage

Below shows the CPU usage of control plane at both pod-rollout time and stable time:

<p align="center">
  <img src="/docs/images/perf/ctrl-cpu.png" width="650"/>
</p>

The pod-rollout time CPU usage has a clear increasing trend. This is understandable as the control plane serves all workloads at the same moment, such as issuing certificates and generating xDS configuration. The CPU usage is roughly proportional to the number of workloads. While during the stable time when no new workload is added, CPU usage drops to nearly zero.

#### Memory Usage

Below shows the memory usage at both pod-rollout time and stable time:

<p align="center">
  <img src="/docs/images/perf/ctrl-memory.png" width="650"/>
</p>

There is not much difference between pod-rollout time and stable time in terms of memory usage. Memory is primarily used to store xDS configurations, which is highly relevant to the mesh scale. The memory usage is also roughly proportionally increasing along with the mesh scale. In our scenario, <b>it takes around 1MB of memory for every one pod added to the mesh.</b>

### Data Plane

By looking at the collected metrics, in our testing scenarios, resource consumption of data plane sidecars does not change much along with the mesh size. At stable time, the average CPU usage is close to 0 and average memory usage is around 50MB. CPU usage is related to request processing and memory usage is related the size of Envoy configuration stored locally. These two factors do not change with the mesh scale.

### Other Scenarios

#### Permissive mode

When permissive mode is enabled in OSM, services are allowed to communicate with any other services in the mesh without additional traffic policies. This is implemented by adding an allow-all wildcard RBAC policy in the inbound listener and all known services as outbound clusters in the Envoy sidecar. Permissive model still enable mTLS authentication between services and requires the services be added inside the mesh. The same load test described above is performed on a service mesh with permissive mode enabled.

On the control plane, the CPU and memory usage both increase compared to non-permissive mode.

On the data plane, there is no obvious difference in terms of CPU usage. However, the memory usage is much higher in permissive mode. This is expected as more outbound cluster information is stored in the sidecar. The chart below shows the sidecar memory usage when 10 and 20 services are added in permissive mesh and non-permissive mesh respectively. Compared to the non-permissive mode, the memory usage is more than doubled.

<p align="center">
  <img src="/docs/images/perf/perm-memory.png" width="650"/>
</p>

The request latency from load generator to workloads is lower in permissive mode. This reduction is due to the RBAC policy filter path being a single wildcard filter as described above. This filter path is shorter than non-permissive mode and there is less rule matching operations for incoming requests. In our test, we observe the P99 latency drops to around 80% in permissive mode. The more SMI traffic policies added to services in non-permissive mode, the larger request time overhead we expect. Below is the latency comparison between permissive and non-permissive mode, tested with 250 per pod RPS:

<p align="center">
  <img src="/docs/images/perf/latency-perm-vs-non-perm.png" width="650"/>
</p>

#### Ingress traffic

In this test, the load generator is deployed outside of the mesh. The requests enter the mesh via [IngressBackend](https://docs.openservicemesh.io/docs/guides/traffic_management/ingress/#ingressbackend-api). Therefore, the request only goes through one Envoy sidecar instead of two.

We compared three scenarios:

* Normal: load generator is inside the mesh. Request goes through 2 sidecars.
* Ingress: load generator is outside the mesh. Request goes through 1 sidecar.
* No OSM installed: Request does not go through any sidecar.

The test result shows that the latency of the ingress scenario is significantly lower than the normal scenario. The ingress scenario is roughly equal to the results of the scenario without OSM installed. This result shows that a single Envoy sidecar does add latency overhead. This complies with our expectation.

<p align="center">
  <img src="/docs/images/perf/latency-ingress.png" width="650"/>
</p>


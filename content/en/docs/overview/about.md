---
title: "About Open Service Mesh"
description: "An overview of OSM and its components"
type: docs
weight: 1
---

# About Open Service Mesh (OSM)

## Service Mesh

Over the last few years, the microservice architecture has become an increasingly popular framework for kubernetes-based containerized applications. By structuring applications as loosely coupled and independent components (services), organizations are able to expedite the development process, scale individual services, enhance observability, and offer flexibility in terms of the technology stack. In such an architecture, service-to-service communication can typically be incorporated into these different components directly. However, as these communication rules become more complex, implementing functionalities for monitoring, networking, and security can become cumbersome. 

A service mesh allows developers to abstract the logic governing service-to-service communication into a separate layer from the application infrastructure. As opposed to having requests routed between microservices directly, a service mesh routes traffic between services via "sidecar" proxies that are deployed alongside the services. Using a service mesh offers many advantages, including more reliable and secure communication, service discovery, and load balancing. Moreover, having all traffic is monitored by the proxies enables service meshes to expose more advanced metrics. Another key benefit is that organizations can shift their attention to enhancing business features instead of worrying about networking and service functionalities. 

## What is Open Service Mesh (OSM)?

Open Service Mesh (OSM) is a simple, complete, and standalone service mesh. OSM ships out-of-the-box with all necessary components to deploy a complete service mesh.

OSM provides a fully featured control plane. It leverages an architecture based on [Envoy](https://www.envoyproxy.io/) reverse-proxy sidecar and works by injecting an Envoy proxy as a sidecar container next to each instance of your application.

OSM can be configured with SMI APIs; in that case, it relies on [SMI Spec](https://smi-spec.io/) to reference services that will participate in the service mesh. Alternatively, OSM can use permissive mode (which does not require SMI APIs to route traffic within the mesh).

As a lightweight and SMI-compatible Service Mesh, OSM is designed to be intuitive and scalable. The project adheres to the following core principles:
* Simple to understand and contribute.
* Effortless to install, maintain, and operate.
* Painless to troubleshoot.
* Easy to configure via Service Mesh Interface (SMI).

The OSM project has since been donated and is governed by the [Cloud Native Computing Foundation (CNCF)](https://www.cncf.io/). The project will continue to be a community-led collaboration around features, and contributions to the project are welcomed and encouraged. Please see our [Contributor Ladder](https://github.com/openservicemesh/osm/blob/main/CONTRIBUTOR_LADDER.md) guide on how you can get involved.

## Features
* Easily and transparently configure traffic shifting for deployments
* Secure service to service communication by enabling mutual TLS
* Define and execute fine grained access control policies for services
* Observability and insights into application metrics for debugging and monitoring services
* Integrate with external certificate management services/solutions with a pluggable interface
* Onboard applications onto the mesh by enabling automatic sidecar injection of Envoy proxy

## Use Case

_As an operator of services on Kubernetes I need an open-source solution, which will dynamically_:

- **Apply policies** governing traffic access between peer applications
- **Encrypt traffic** between applications leveraging mTLS and short-lived certificates with a custom CA
- **Rotate certificates** as often as necessary to make these short-lived and remove the need for certificate revocation management
- **Collect traces and metrics** to provide visibility into the health and operation of the services
- **Implement traffic split** between various versions of the services deployed as defined via [SMI Spec](https://smi-spec.io/)

## Getting Started

Follow the [Getting Started](/docs/getting_started) guide to try out OSM with a sample microservice topology.

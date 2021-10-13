---
title: "Open Service Mesh"
description: "Open Service Mesh documentation and resources."
type: docs

---

## What is OSM?

Open Service Mesh (OSM) is a simple, complete, and standalone [service mesh](https://en.wikipedia.org/wiki/Service_mesh). OSM ships out-of-the-box with all necessary components to deploy a complete service mesh.

OSM provides a fully featured control plane. It leverages an architecture based on [Envoy](https://www.envoyproxy.io/) reverse-proxy sidecar and works by injecting an Envoy proxy as a sidecar container next to each instance of your application.

OSM can be configured with SMI APIs; in that case, it relies on [SMI Spec](https://smi-spec.io/) to reference services that will participate in the service mesh. Alternatively, OSM can use permissive mode (which does not require SMI APIs to route traffic within the mesh).

OSM is designed with the following core principles:
* Simple to understand and contribute.
* Effortless to install, maintain, and operate.
* Painless to troubleshoot.
* Easy to configure via Service Mesh Interface (SMI).

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

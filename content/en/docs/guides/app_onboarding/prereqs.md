---
title: "Prerequisites"
description: "Application Requirements"
type: docs
aliases: ["/docs/application_requirements"]
weight: 1
---

# Prerequisites

## Security Contexts

* Do not run applications with the user ID (UID) value of **1500**. This is **reserved** for the Envoy proxy sidecar container injected into pods by OSM's sidecar injector.
* If security context `runAsNonRoot` is set to `true` at the pod level, a `runAsUser` value must be provided either for the pod
or for each container. For example:
  ```yaml
    securityContext:
      runAsNonRoot: true
      runAsUser: 1200
  ```
    If the UID is omitted, application containers may attempt to run as root user by default, causing conflict with the pod's security context.
* Additional capabilities are not required.

> Note: the OSM init container is programmed to run as root and add capability `NET_ADMIN` as it requires these security
> contexts to finish scheduling. These values are not changed by application security contexts.

## Ports

Do not use the following ports as they are used by the Envoy sidecar.

| Port  | Description                            |
| ----- | -------------------------------------- |
| 15000 | Envoy Admin Port                       |
| 15001 | Envoy Outbound Listener Port           |
| 15003 | Envoy Inbound Listener Port            |
| 15010 | Envoy Prometheus Inbound Listener Port |

---
title: "Feature Lifecycle"
description: "Lifecycle of features and their status"
type: docs
weight: 1
---

# Feature Lifecycle

This document describes the lifecycle of features in OSM, their versioning phases, and information about their availability, stability, upgradeability, support, and deprecation policy.


## Feature versioning

A feature in OSM is versioned based on its maturity. The following table describes the versioning states for a feature, along with information about each state in the feature's versioning lifecycle.

|                    | Alpha                                                                                                                         | Beta                                                                                                                                                | Stable                                                                                                                                                  |
| ------------------ | ----------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Definition         | Feature is experimental and under active development. APIs may change and require a migration effort during version upgrades. | Feature is ready for use but needs to be vetted in production environments. APIs may change and require a migration effort during version upgrades. | Feature is ready for use in production environments. APIs are expected to be supported in several subsequent releases.                                  |
| Stability          | Using the feature may expose bugs. Disabled by default.                                                                       | Well tested and usable in production.                                                                                                               | Well tested and stable for production use.                                                                                                              |
| Support            | May not be backward compatible. Behavior is subject to change and feature may be deprecated at any time.                      | May not be backward compatible. Behavior is subject to change and feature is expected to be supported in subsequent versions.                       | Behavior is not expected to change and is considered usable for several subsequent releases.                                                            |
| Deprecation Policy | None. Can be deprecated at any time.                                                                                          | Feature will be supported for at least 2 minor releases, and will only be deprecated with advanced notice.                                          | Feature will be supported for several subsequent releases, and will only be deprecated with advanced notice and an upgrade path to alternative options. |
| Documentation      | Documentation may or may not be present.                                                                                      | Documentation present on docs.openservicemesh.io with information on how to use it.                                                                 | Documentation present on docs.openservicemesh.io with thorough guide and sample demos.                                                                  |

## Feature state

The following sections list the features supported in OSM and their versioning state.

### Core

| Feature                                        | State  |
| ---------------------------------------------- | ------ |
| OSM CLI                                        | Stable |
| Control plane install using OSM CLI and Helm   | Stable |
| Control plane uninstall using OSM CLI and Helm | Stable |
| Automatic sidecar injection                    | Stable |
| Configuration validating webhook               | Stable |
| Control plane upgrade using OSM CLI and Helm   | Beta   |
| Multicluster                                   | Alpha  |

<br/>

### Traffic management

| Feature                              | State  |
| ------------------------------------ | ------ |
| Traffic access control using SMI     | Stable |
| Traffic shifting using SMI           | Stable |
| HTTP routing using SMI               | Stable |
| TCP routing using SMI                | Stable |
| Permissive traffic policy            | Stable |
| Protocols: HTTP1.1, HTTP2, TCP, gRPC | Stable |
| Egress policies                      | Stable |
| Ingress policies                     | Stable |
| Ingress gateway (Contour)            | Beta   |
| Circuit breaking                     | Alpha  |

<br/>

### Security and certificate management

| Feature                                               | State  |
| ----------------------------------------------------- | ------ |
| Automatic mutual TLS                                  | Stable |
| Authorization using SMI access control                | Stable |
| Certificate provider - Tresor (native implementation) | Stable |
| Mutual TLS for Ingress                                | Stable |
| Certificate provider - cert-manager                   | Beta   |
| Certificate provider - Hashicorp Vault                | Beta   |

<br/>

### Observability

| Feature                         | State  |
| ------------------------------- | ------ |
| Prometheus integration          | Stable |
| Grafana integration             | Stable |
| Control plane metrics           | Stable |
| Envoy traffic metrics           | Stable |
| Distributed tracing with Jaegar | Beta   |

<br/>

### Integrations

| Feature                           | State |
| --------------------------------- | ----- |
| Progressive delivery with Flagger | Alpha |
| Dapr integration                  | Alpha |

<br/>

## Helm chart values deprecation

Values in the Helm chart may be deprecated in preparation to be removed in a future release. A value that is deprecated will be prefixed with the `DEPRECATED` keyword alongside its comment in the values file and in the title of the corresponding property in the JSON schema for the Helm values.

For example, the value `foo` is considered to be deprecated due to the `DEPRECATED` keyword in its description in the values file:
```yaml
# -- DEPRECATED
# foo configuration
foo: bar
```

The `helm show values` command can be used to view the Helm chart's values.

A value marked as `DEPRECATED` in a release will only be removed in the next minor release at the earliest. This means, a value deprecated in release `vX.Y.Z` will only be removed on or after release `vX.Y+1.Z`. Deprecated values will be documented in the release notes.

## API Deprecation

Similar to the [Kubernetes API deprecation policy](https://kubernetes.io/docs/reference/using-api/deprecation-policy/), GA or stable APIs will be marked as deprecated prior to removal.

Stable APIs marked as deprecated:

- Remain functional but will generate a warning if used
- At the earliest, would not be removed until the next major version
- Will no longer be functional after removal

Alpha APIs may be removed in any release without any prior notice or deprecation.
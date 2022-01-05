---
title: "Ingress"
description: "Using Ingress to manage external access to services within the cluster"
type: docs
weight: 5
---

# Ingress

## Using Ingress to manage external access to services within the cluster

Ingress refers to managing external access to services within the cluster, typically HTTP/HTTPS services. OSM's ingress capability allows cluster administrators and application owners to route traffic from clients external to the service mesh to service mesh backends using a set of rules depending on the mechanism used to perform ingress.

## IngressBackend API

OSM leverages its [IngressBackend API][1] to configure a backend service to accept ingress traffic from trusted sources. The specification enables configuring how specific backends must authorize ingress traffic depending on the protocol used, HTTP or HTTPS. When the backend protocol is `http`, the specified source kind must either be: 1. `Service` kind whose endpoints will be authorized to connect to the backend, or 2. `IPRange` kind that specifies the source IP CIDR range authorized to connect to the backend. When the backend protocol is `https`, the source specified must be an `AuthenticatedPrincipal` kind which defines the Subject Alternative Name (SAN) encoded in the client's certificate that the backend will authenticate. A source with the kind `Service` or `IPRange` is optional for `https` backends, and if specified implies that the client must match the source in addition to its `AuthenticatedPrincipal` value. For `https` backends, client certificate validation is performed by default and can be disabled by setting `skipClientCertValidation: true` in the `tls` field for the backend. The `port.number` field for a `backend` service in the `IngressBackend` configuration must correspond to the `targetPort` of a Kubernetes service.

Note that when the `Kind` for a source in an `IngressBackend` configuration is set to `Service`, OSM controller will attempt to discover the endpoints of that service. For OSM to be able to discover the endpoints of a service, the namespace in which the service resides needs to be a monitored namespace. Enable the namespace to be monitored using:

```bash
kubectl label ns <namespace> openservicemesh.io/monitored-by=<mesh name>
```

### Examples

The following IngressBackend configuration will allow access to the `foo` service on port `80` in the `test` namespace only if the source originating the traffic is an endpoint of the `myapp` service in the `default` namespace:
```yaml
kind: IngressBackend
apiVersion: policy.openservicemesh.io/v1alpha1
metadata:
  name: basic
  namespace: test
spec:
  backends:
    - name: foo
      port:
        number: 80 # targetPort of the service
        protocol: http
  sources:
    - kind: Service
      namespace: default
      name: myapp
```

The following IngressBackend configuration will allow access to the `foo` service on port `80` in the `test` namespace only if the source originating the traffic has an IP address that belongs to the CIDR range `10.0.0.0/8`:
```yaml
kind: IngressBackend
apiVersion: policy.openservicemesh.io/v1alpha1
metadata:
  name: basic
  namespace: test
spec:
  backends:
    - name: foo
      port:
        number: 80 # targetPort of the service
        protocol: http
  sources:
    - kind: IPRange
      name: 10.0.0.0/8
```

The following IngressBackend configuration will allow access to the `foo` service on port `80` in the `test` namespace only if the source originating the traffic encrypts the traffic with `TLS` and has the Subject Alternative Name (SAN) `client.default.svc.cluster.local` encoded in its client certificate:
```yaml
kind: IngressBackend
apiVersion: policy.openservicemesh.io/v1alpha1
metadata:
  name: basic
  namespace: test
spec:
  backends:
    - name: foo
      port:
        number: 80
        protocol: https # https implies TLS
      tls:
        skipClientCertValidation: false # mTLS (optional, default: false)
  sources:
    - kind: AuthenticatedPrincipal
      name: client.default.svc.cluster.local
```

Refer to the following sections to understand how the `IngressBackend` configuration looks like for `http` and `https` backends.

## Choices to perform Ingress

OSM supports multiple options to expose mesh services externally using ingress which are described in the following sections. OSM has been tested with Contour and OSS Nginx, which work with the ingress controller installed outside the mesh and provisioned with a certificate to participate in the mesh.

> Note: OSM integration with Nginx Plus has not been fully tested for picking up a self-signed mTLS certificate from a Kubernetes secret. However, an alternative way to incorporate Nginx Plus or any ingress is to install it in the mesh so that it is injected with an Envoy sidecar, which will allow it to participate in the mesh. Additional inbound ports such as 80 and 443 may need to be allowed to bypass the Envoy sidecar.

### 1. Using Contour Ingress Controller and Gateway

Using [Contour](https://projectcontour.io/) ingress controller and edge proxy is the preferred approach to performing Ingress in an OSM managed service mesh. With Contour, users get a high performance ingress controller with rich policy specifications for a variety of use cases while maintaining a lightweight profile. Enabling Contour in OSM also allows traffic routed from Contour's edge proxy to service mesh backends to be encrypted using mutual-TLS (mTLS) to provide end-to-end security within the service mesh.

To use Contour for ingress, enable it during install using:

```bash
osm install --set contour.enabled=true
```

If you wish to secure the connections from Contour's edge proxy to service mesh backend applications using mTLS, OSM provides the option to bootstrap Contour's edge proxy with a trusted client certificate issued from the certificate provider used in OSM, that Contour uses while proxying to backends over TLS. Contour's client certificate can be provisioned during install or post-install, as described below.

To provision Contour's Envoy client certificate during install:

```bash
osm install --set contour.enabled=true \
    -set contour.configInline.tls.envoy-client-certificate.name=osm-contour-envoy-client-cert \
    --set contour.configInline.tls.envoy-client-certificate.namespace=<osm install namespace, osm-system if --osm-namespace not set>
```

To provision Contour's Envoy client certificate if not provisioned during install, edit Contour's ConfigMap to specify the `tls.envoy-client-certificate` field and restart the Contour control plane.

```yaml
apiVersion: v1
kind: ConfigMap
data:
  contour.yaml: |
    tls:
      envoy-client-certificate:
        name: osm-contour-envoy-client-cert
        namespace: osm-system # Namespace where OSM is installed; please check your deployment and set this appropriately
```

```bash
kubectl rollout restart deploy osm-contour-contour -n <osm namespace>
```

In addition to configuring Contour's edge proxy using the appropriate APIs, a service mesh backend in OSM will only accept traffic from authorized edge proxies or gateways. OSM's [IngressBackend specification][1] allows cluster administrators and application owners to explicitly specify how a service mesh backend should authorize ingress traffic. The following sections describe how the `IngressBackend` and `HTTPProxy` APIs can be used in conjunction to allow HTTP and HTTPS ingress traffic to be routed to mesh backends.

It is recommended to always limit ingress traffic to authorized clients. For this purpose, enable OSM to monitor the endpoints of Contour's edge proxy residing in the namespace OSM was installed in:

```bash
kubectl label ns <osm namespace> openservicemesh.io/monitored-by=<mesh name>
```

#### HTTP Ingress with Contour

A minimal [HTTPProxy][2] configuration and OSM's `IngressBackend`[1] specification to route ingress traffic to a mesh service `foo` in the namespace `test` might look like:

```yaml
apiVersion: projectcontour.io/v1
kind: HTTPProxy
metadata:
  name: basic
  namespace: test
spec:
  virtualhost:
    fqdn: foo-basic.bar.com
  routes:
    - conditions:
        - prefix: /
      services:
        - name: foo
          port: 80
---
kind: IngressBackend
apiVersion: policy.openservicemesh.io/v1alpha1
metadata:
  name: basic
  namespace: test
spec:
  backends:
    - name: foo
      port:
        number: 80 # targetPort of the service
        protocol: http # http implies no TLS
  sources:
    - kind: Service
      namespace: osm-system
      name: osm-contour-envoy
```

The above configurations allow external clients to access the `foo` service in the `test` namespace as follows:

1. The HTTPProxy configuration will route incoming HTTP traffic originating externally with a `Host:` header for `foo-basic.bar.com` to a service named `foo` on port `80` in the `test` namespace.
1. The IngressBackend configuration will allow access to the `foo` service on port `80` in the `test` namespace only if the source originating the traffic is an endpoint of the `osm-contour-envoy` service in the namespace where osm was installed (which by default is the `osm-system` namespace).

#### HTTPS Ingress with Contour (mTLS and TLS)

To enable HTTPS proxying (over TLS or mTLS) between Contour edge proxy and service mesh backends, Contour's [upstream TLS configuration](https://projectcontour.io/docs/v1.18.0/config/upstream-tls/) can be used in conjunction with OSM's IngressBackend configuration to perform peer certificate validation.

A minimal configuration might look like:

```yaml
apiVersion: projectcontour.io/v1
kind: TLSCertificateDelegation
metadata:
  name: osm-ca-secret
  namespace: osm-system
spec:
  delegations:
    - secretName: osm-ca-bundle
      targetNamespaces:
        - test
---
apiVersion: projectcontour.io/v1
kind: HTTPProxy
metadata:
  name: basic
  namespace: test
spec:
  virtualhost:
    fqdn: foo-basic.bar.com
  routes:
    - services:
        - name: foo
          port: 80
          validation:
            caSecret: osm-system/osm-ca-bundle
            subjectName: foo-service-account.test.cluster.local
---
kind: IngressBackend
apiVersion: policy.openservicemesh.io/v1alpha1
metadata:
  name: basic
  namespace: test
spec:
  backends:
    - name: foo
      port:
        number: 80
        protocol: https # https implies TLS
      tls:
        skipClientCertValidation: false # mTLS (optional, default: false)
  sources:
    - kind: Service
      namespace: osm-system
      name: osm-contour-envoy
    - kind: AuthenticatedPrincipal
      name: osm-contour-envoy.osm-system.cluster.local
```

The above configurations allow external clients to access the `foo` service in the `test` namespace as follows:

1. The TLSCertificateDelegation configuration will allow Contour access to the `osm-ca-bundle` secret residing in the namespace where osm was installed (by default the `osm-system` namespace) when parsing HTTPProxy configurations residing in the `test` namespace.

1. The HTTPProxy configuration will route incoming HTTP traffic originating externally with a `Host:` header for `foo-basic.bar.com` to a service named `foo` on port `80` in the `test` namespace over TLS. The vadidation field specified will ensure Contour's Envoy edge proxy validates the TLS certificate presented by the `foo` backend service using the TLS CA certificate stored in the `osm-system/osm-ca-bundle` k8s secret, and that the Subject Alternative Name (SAN) in the server certificate presented by the backend service matches `foo-service-account.test.cluster.local`.

   > Note: Certificates issued by OSM have a SAN of the form `<service-account>.<namespace>.cluster.local`

1. The IngressBackend configuration will allow access to the `foo` service on port `80` in the `test` namespace only if the source originating the traffic is an endpoint of the `osm-contour-envoy` service in the namespace where osm was installed (by default the `osm-system` namespace) and the client certificate has a Subject Alternative Name matching `osm-contour-envoy.<osm-namespace>.cluster.local` (where `<osm-namespace>` refers to the namespace where osm was installed).
   > Note: Client certificate validation on the backend can be skipped by setting `skipClientCertValidation: true` in the IngressBackend configuration.

#### Examples

Refer to the [Ingress with Contour demo](/docs/demos/ingress_contour) for examples on how to expose mesh services externally using Contour in OSM.

### 2. Bring your own Ingress Controller and Gateway

If using OSM with Contour for ingress is not feasible for your use case, OSM provides the facility to use your own ingress controller and edge gateway for routing external traffic to service mesh backends. Much like how ingress is configured above, in addition to configuring the ingress controller to route traffic to service mesh backends, an IngressBackend configuration is required to authorize clients responsible for proxying traffic originating externally.

[1]: /docs/api_reference/policy/v1alpha1/#policy.openservicemesh.io/v1alpha1.IngressBackendSpec
[2]: https://projectcontour.io/docs/v1.18.0/config/fundamentals/

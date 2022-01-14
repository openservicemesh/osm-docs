---
title: "Ingress with Contour"
description: "HTTP and HTTPS ingress with Contour ingress controller"
type: docs
weight: 10
---

OSM provides the option to use [Contour](https://projectcontour.io) ingress controller and Envoy based edge proxy to route external traffic to service mesh backends. This guide will demonstrate how to configure HTTP and HTTPS ingress to a service part of an OSM managed service mesh.


## Prerequisites

- Kubernetes cluster running Kubernetes v1.19.0 or greater.
- Have `kubectl` available to interact with the API server.
- No existing installation of OSM. Any existing installation must first be uninstalled prior to proceeding with this demo.
- Have `osm` or `Helm 3` CLI available for installing OSM and Contour.
- OSM version >= v0.10.0.

## Demo

First, we will install OSM and Contour as in the `osm-system` namespace and name the mesh name as `osm`.
```bash
export osm_namespace=osm-system # Replace osm-system with the namespace where OSM will be installed
export osm_mesh_name=osm # Replace osm with the desired OSM mesh name
```

If using `osm` CLI:
```bash
osm install --set contour.enabled=true \
    --mesh-name "$osm_mesh_name" \
    --osm-namespace "$osm_namespace" \
    --set contour.configInline.tls.envoy-client-certificate.name=osm-contour-envoy-client-cert \
    --set contour.configInline.tls.envoy-client-certificate.namespace="$osm_namespace"
```

If using `Helm`:
```bash
helm install "$osm_mesh_name" osm --repo https://openservicemesh.github.io/osm \
    --set contour.enabled=true \
    --set contour.configInline.tls.envoy-client-certificate.name=osm-contour-envoy-client-cert \
    --set contour.configInline.tls.envoy-client-certificate.namespace="$osm_namespace"
```

To restrict ingress traffic on backends to authorized clients, we will set up the IngressBackend configuration such that only ingress traffic from the endpoints of the `osm-contour-envoy` service can route traffic to the service backend. To be able to discover the endpoints of `osm-contour-envoy` service, we need OSM controller to monitor the corresponding namespace. However, Contour must NOT be injected with an Envoy sidecar to function properly.
```bash
kubectl label namespace "$osm_namespace" openservicemesh.io/monitored-by="$osm_mesh_name"
```

Save the ingress gateway's external IP address and port which we will later use to test access to the backend application:
```bash
export ingress_host="$(kubectl -n "$osm_namespace" get service osm-contour-envoy -o jsonpath='{.status.loadBalancer.ingress[0].ip}')"
export ingress_port="$(kubectl -n "$osm_namespace" get service osm-contour-envoy -o jsonpath='{.spec.ports[?(@.name=="http")].port}')"
```

Next, we will deploy the sample `httpbin` service.

```bash
# Create a namespace
kubectl create ns httpbin

# Add the namespace to the mesh
osm namespace add httpbin

# Deploy the application
kubectl apply -f https://raw.githubusercontent.com/openservicemesh/osm/{{< param osm_branch >}}/docs/example/manifests/samples/httpbin/httpbin.yaml -n httpbin
```

Confirm the `httpbin` service and pod is up and running:
```console
$ kubectl get pods -n httpbin
NAME                       READY   STATUS    RESTARTS   AGE
httpbin-74677b7df7-zzlm2   2/2     Running   0          11h

$ kubectl get svc -n httpbin
NAME      TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)     AGE
httpbin   ClusterIP   10.0.22.196   <none>        14001/TCP   11h
```

### HTTP Ingress

Next, we will create the HTTPProxy and IngressBackend configurations necessary to allow external clients to access the `httpbin` service on port `14001` in the `httpbin` namespace. The connection from the Contour's ingress gateway to the `httpbin` backend pod will be unencrypted since we aren't using TLS.

```bash
kubectl apply -f - <<EOF
apiVersion: projectcontour.io/v1
kind: HTTPProxy
metadata:
  name: httpbin
  namespace: httpbin
spec:
  virtualhost:
    fqdn: httpbin.org
  routes:
  - services:
    - name: httpbin
      port: 14001
---
kind: IngressBackend
apiVersion: policy.openservicemesh.io/v1alpha1
metadata:
  name: httpbin
  namespace: httpbin
spec:
  backends:
  - name: httpbin
    port:
      number: 14001 # targetPort of httpbin service
      protocol: http
  sources:
  - kind: Service
    namespace: "$osm_namespace"
    name: osm-contour-envoy
EOF
```

Now, we expect external clients to be able to access the `httpbin` service for HTTP requests for the `Host:` header `httpbin.org`:
```console
$ curl -sI http://"$ingress_host":"$ingress_port"/get -H "Host: httpbin.org"
HTTP/1.1 200 OK
server: envoy
date: Fri, 06 Aug 2021 17:39:43 GMT
content-type: application/json
content-length: 314
access-control-allow-origin: *
access-control-allow-credentials: true
x-envoy-upstream-service-time: 3
vary: Accept-Encoding
```

### HTTPS Ingress (mTLS and TLS)

To proxy connections to TLS backends using HTTPS, the backend service must be annotated with the port as follows:
```bash
kubectl annotate service httpbin -n httpbin projectcontour.io/upstream-protocol.tls='14001' --overwrite
```

Next, we need to create an HTTPProxy configuration to use TLS proxying to the backend service, and providing a CA certificate to validate the server certificate presented by the backend service. For this to work, we need to first delegate to Contour the permission to read OSM's CA certificate secret from the OSM's namespace when referenced in the HTTPProxy configuration in the `httpbin` namespace. Refer to the [Upstream TLS section](https://projectcontour.io/docs/v1.18.0/config/upstream-tls/) to learn more about upstream certificate validation and when certificate delegation is necessary. In addition, we must create an IngressBackend resource that specifies HTTPS ingress traffic directed to the `httpbin` service must only accept traffic from a trusted client, osm-contour-envoy in the ingress edge proxy we deployed. OSM automatically provisioned a client certificate for the `osm-contour-envoy` ingress gateway with the Subject Alternative Name (SAN) `osm-contour-envoy.$osm_namespace.cluster.local` during install, so the IngressBackend configuration needs to reference the same SAN for mTLS authentication between the `osm-contour-envoy` edge and the `httpbin` backend.

> Note: `<osm-namespace>` refers to the namespace where the osm control plane is installed.

Apply the configurations:
```bash
kubectl apply -f - <<EOF
apiVersion: projectcontour.io/v1
kind: TLSCertificateDelegation
metadata:
  name: ca-secret
  namespace: "$osm_namespace"
spec:
  delegations:
    - secretName: osm-ca-bundle
      targetNamespaces:
      - httpbin
---
apiVersion: projectcontour.io/v1
kind: HTTPProxy
metadata:
  name: httpbin
  namespace: httpbin
spec:
  virtualhost:
    fqdn: httpbin.org
  routes:
  - services:
    - name: httpbin
      port: 14001
      validation:
        caSecret: "$osm_namespace/osm-ca-bundle"
        # subjectName for a service is of the form <service-account>.<namespace>.cluster.local
        # where the service account and namespace is that of the pod backing the service
        subjectName: httpbin.httpbin.cluster.local
---
kind: IngressBackend
apiVersion: policy.openservicemesh.io/v1alpha1
metadata:
  name: httpbin
  namespace: httpbin
spec:
  backends:
  - name: httpbin
    port:
      number: 14001 # targetPort of httpbin service
      protocol: https
    tls:
      skipClientCertValidation: false # mTLS (defaults to false)
  sources:
  - kind: Service
    namespace: "$osm_namespace"
    name: osm-contour-envoy
  - kind: AuthenticatedPrincipal
    name: "osm-contour-envoy.$osm_namespace.cluster.local"
EOF
```

Now, we expect external clients to be able to access the `httpbin` service for HTTP requests for the `Host:` header `httpbin.org` with HTTPS proxying over mTLS between the ingress gateway and service backend:
```console
$ curl -sI http://"$ingress_host":"$ingress_port"/get -H "Host: httpbin.org"
HTTP/1.1 200 OK
server: envoy
date: Fri, 06 Aug 2021 17:39:43 GMT
content-type: application/json
content-length: 314
access-control-allow-origin: *
access-control-allow-credentials: true
x-envoy-upstream-service-time: 3
vary: Accept-Encoding
```

To verify that unauthorized clients are not allowed to access the backend, we can update the `sources` specified in the IngressBackend configuration. Let's update the principal to something other than the SAN encoded in the ingress gateway's certificate.

```bash
kubectl apply -f - <<EOF
kind: IngressBackend
apiVersion: policy.openservicemesh.io/v1alpha1
metadata:
  name: httpbin
  namespace: httpbin
spec:
  backends:
  - name: httpbin
    port:
      number: 14001 # targetPort of httpbin service
      protocol: https
    tls:
      skipClientCertValidation: false # mTLS (defaults to false)
  sources:
  - kind: Service
    namespace: "$osm_namespace"
    name: osm-contour-envoy
  - kind: AuthenticatedPrincipal
    name: "untrusted-client.cluster.local"
EOF
```

Confirm the requests are rejected with an `HTTP 403 Forbidden` response:
```console
$ curl -sI http://"$ingress_host":"$ingress_port"/get -H "Host: httpbin.org"
HTTP/1.1 403 Forbidden
content-length: 19
content-type: text/plain
date: Fri, 06 Aug 2021 18:40:45 GMT
server: envoy
x-envoy-upstream-service-time: 8
```

Next, we demonstrate support for disabling client certificate validation on the service backend if necessary, by updating our IngressBackend configuration to set `skipClientCertValidation: true`, while still using an untrusted client:
```bash
kubectl apply -f - <<EOF
kind: IngressBackend
apiVersion: policy.openservicemesh.io/v1alpha1
metadata:
  name: httpbin
  namespace: httpbin
spec:
  backends:
  - name: httpbin
    port:
      number: 14001 # targetPort of httpbin service
      protocol: https
    tls:
      skipClientCertValidation: true
  sources:
  - kind: Service
    namespace: "$osm_namespace"
    name: osm-contour-envoy
  - kind: AuthenticatedPrincipal
    name: "untrusted-client.cluster.local"
EOF
```

Confirm the requests succeed again since untrusted authenticated principals are allowed to connect to the backend:
```
$ curl -sI http://"$ingress_host":"$ingress_port"/get -H "Host: httpbin.org"
HTTP/1.1 200 OK
server: envoy
date: Fri, 06 Aug 2021 18:51:47 GMT
content-type: application/json
content-length: 314
access-control-allow-origin: *
access-control-allow-credentials: true
x-envoy-upstream-service-time: 4
vary: Accept-Encoding
```
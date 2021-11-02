---
title: "Ingress with Kubernetes Nginx Ingress Controller"
description: "HTTP and HTTPS ingress with Kubernetes Nginx Ingress Controller"
type: docs
weight: 11
---

This guide will demonstrate how to configure HTTP and HTTPS ingress to a service part of an OSM managed service mesh when using [Kubernetes Nginx Ingress Controller](https://kubernetes.github.io/ingress-nginx/).


## Prerequisites

- Kubernetes cluster running Kubernetes v1.19.0 or greater.
- Have `kubectl` available to interact with the API server.
- Have OSM version >= v0.10.0 installed.
- Have Kubernetes Nginx Ingress Controller installed. Refer to the [deployment guide](https://kubernetes.github.io/ingress-nginx/deploy/) to install it.

## Demo

First, note the details regarding OSM and Nginx installations:
```bash
osm_namespace=osm-system # Replace osm-system with the namespace where OSM is installed
osm_mesh_name=osm # replace osm with the mesh name (use `osm mesh list` command)

nginx_ingress_namespace=<nginx-namespace> # replace <nginx-namespace> with the namespace where Nginx is installed
nginx_ingress_service=<nginx-ingress-controller-service> # replace <nginx-ingress-controller-service> with the name of the nginx ingress controller service
nginx_ingress_host="$(kubectl -n "$nginx_ingress_namespace" get service "$nginx_ingress_service" -o jsonpath='{.status.loadBalancer.ingress[0].ip}')"
nginx_ingress_port="$(kubectl -n "$nginx_ingress_namespace" get service "$nginx_ingress_service" -o jsonpath='{.spec.ports[?(@.name=="http")].port}')"
```

To restrict ingress traffic on backends to authorized clients, we will set up the IngressBackend configuration such that only ingress traffic from the endpoints of the Nginx Ingress Controller service can route traffic to the service backend. To be able to discover the endpoints of this service, we need OSM controller to monitor the corresponding namespace. However, Nginx must NOT be injected with an Envoy sidecar to function properly.
```bash
osm namespace add "$nginx_ingress_namespace" --mesh-name "$osm_mesh_name" --disable-sidecar-injection
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

Next, we will create the Ingress and IngressBackend configurations necessary to allow external clients to access the `httpbin` service on port `14001` in the `httpbin` namespace. The connection from the Nginx's ingress service to the `httpbin` backend pod will be unencrypted since we aren't using TLS.

```bash
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: httpbin
  namespace: httpbin
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: httpbin
            port:
              number: 14001
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
      number: 14001
      protocol: http
  sources:
  - kind: Service
    namespace: "$nginx_ingress_namespace"
    name: "$nginx_ingress_service"
EOF
```

Now, we expect external clients to be able to access the `httpbin` service for HTTP requests:
```console
$ curl -sI http://"$nginx_ingress_host":"$nginx_ingress_port"/get
HTTP/1.1 200 OK
Date: Wed, 18 Aug 2021 18:12:35 GMT
Content-Type: application/json
Content-Length: 366
Connection: keep-alive
access-control-allow-origin: *
access-control-allow-credentials: true
x-envoy-upstream-service-time: 2
```

### HTTPS Ingress (mTLS and TLS)

To proxy connections to HTTPS backends, we will configure the Ingress and IngressBackend configurations to use `https` as the backend protocol, and have OSM issue a certificate that Nginx will use as the client certificate to proxy HTTPS connections to TLS backends. The client certificate and CA certificate will be stored in a Kubernetes secret that Nginx will use to authenticate service mesh backends.

To issue a client certificate for the Nginx ingress service, update the `osm-mesh-config` `MeshConfig` resource.
```bash
kubectl edit meshconfig osm-mesh-config -n "$osm_namespace"
```

Add the `ingressGateway` field under `spec.certificate`:
```yaml
certificate:
  ingressGateway:
    secret:
      name: osm-nginx-client-cert
      namespace: <osm-namespace> # replace <osm-namespace> with the namespace where OSM is installed
    subjectAltNames:
    - ingress-nginx.ingress.cluster.local
    validityDuration: 24h
```
> Note: The Subject Alternative Name (SAN) is of the form `<service-account>.<namespace>.cluster.local`, where the service account and namespace correspond to the Ngnix service.

Next, we need to create an Ingress and IngressBackend configuration to use TLS proxying to the backend service, while enabling proxying to the backend over mTLS. For this to work, we must create an IngressBackend resource that specifies HTTPS ingress traffic directed to the `httpbin` service must only accept traffic from a trusted client. OSM provisioned a client certificate for the Nginx ingress service with the Subject ALternative Name (SAN) `ingress-nginx.ingress.cluster.local`, so the IngressBackend configuration needs to reference the same SAN for mTLS authentication between the Nginx ingress service and the `httpbin` backend.

Apply the configurations:
```bash
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: httpbin
  namespace: httpbin
  annotations:
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
    # proxy_ssl_name for a service is of the form <service-account>.<namespace>.cluster.local
    nginx.ingress.kubernetes.io/configuration-snippet: |
      proxy_ssl_name "httpbin.httpbin.cluster.local";
    nginx.ingress.kubernetes.io/proxy-ssl-secret: "osm-system/osm-nginx-client-cert"
    nginx.ingress.kubernetes.io/proxy-ssl-verify: "on"
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: httpbin
            port:
              number: 14001
---
apiVersion: policy.openservicemesh.io/v1alpha1
kind: IngressBackend
metadata:
  name: httpbin
  namespace: httpbin
spec:
  backends:
  - name: httpbin
    port:
      number: 14001
      protocol: https
    tls:
      skipClientCertValidation: false
  sources:
  - kind: Service
    name: "$nginx_ingress_service"
    namespace: "$nginx_ingress_namespace"
  - kind: AuthenticatedPrincipal
    name: ingress-nginx.ingress.cluster.local
EOF
```

Now, we expect external clients to be able to access the `httpbin` service for requests with HTTPS proxying over mTLS between the ingress gateway and service backend:
```console
$ curl -sI http://"$nginx_ingress_host":"$nginx_ingress_port"/get
HTTP/1.1 200 OK
Date: Wed, 18 Aug 2021 18:12:35 GMT
Content-Type: application/json
Content-Length: 366
Connection: keep-alive
access-control-allow-origin: *
access-control-allow-credentials: true
x-envoy-upstream-service-time: 2
```

To verify that unauthorized clients are not allowed to access the backend, update the `sources` specified in the IngressBackend configuration. Let's update the principal to something other than the SAN encoded in the Nginx client's certificate.

```bash
kubectl apply -f - <<EOF
apiVersion: policy.openservicemesh.io/v1alpha1
kind: IngressBackend
metadata:
  name: httpbin
  namespace: httpbin
spec:
  backends:
  - name: httpbin
    port:
      number: 14001
      protocol: https
    tls:
      skipClientCertValidation: false
  sources:
  - kind: Service
    name: "$nginx_ingress_service"
    namespace: "$nginx_ingress_namespace"
  - kind: AuthenticatedPrincipal
    name: untrusted-client.cluster.local # untrusted
EOF
```

Confirm the requests are rejected with an `HTTP 403 Forbidden` response:
```console
$ curl -sI http://"$nginx_ingress_host":"$nginx_ingress_port"/get
HTTP/1.1 403 Forbidden
Date: Wed, 18 Aug 2021 18:36:09 GMT
Content-Type: text/plain
Content-Length: 19
Connection: keep-alive
```

Next, we demonstrate support for disabling client certificate validation on the service backend if necessary, by updating our IngressBackend configuration to set `skipClientCertValidation: true`, while still using an untrusted client:
```bash
kubectl apply -f - <<EOF
apiVersion: policy.openservicemesh.io/v1alpha1
kind: IngressBackend
metadata:
  name: httpbin
  namespace: httpbin
spec:
  backends:
  - name: httpbin
    port:
      number: 14001
      protocol: https
    tls:
      skipClientCertValidation: true
  sources:
  - kind: Service
    name: "$nginx_ingress_service"
    namespace: "$nginx_ingress_namespace"
  - kind: AuthenticatedPrincipal
    name: untrusted-client.cluster.local # untrusted
EOF
```

Confirm the requests succeed again since untrusted authenticated principals are allowed to connect to the backend:
```
$ curl -sI http://"$nginx_ingress_host":"$nginx_ingress_port"/get
HTTP/1.1 200 OK
Date: Wed, 18 Aug 2021 18:36:49 GMT
Content-Type: application/json
Content-Length: 364
Connection: keep-alive
access-control-allow-origin: *
access-control-allow-credentials: true
x-envoy-upstream-service-time: 2
```

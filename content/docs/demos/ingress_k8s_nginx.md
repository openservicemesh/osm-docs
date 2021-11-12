---
title: "Ingress with Kubernetes Nginx Ingress Controller"
description: "HTTP and HTTPS ingress with Kubernetes Nginx Ingress Controller"
type: docs
weight: 11
---

This is a guide to configuring [NGINX Ingress Controller](https://docs.nginx.com/nginx-ingress-controller/intro/overview/) with Open Service Mesh on Kubernetes.


## Requirements

- [Kubernetes](https://kubernetes.io/) cluster with v1.19.0 or higher
- [kubectl](https://kubernetes.io/docs/tasks/tools/#kubectl)
- [Open Service Mesh](https://github.com/openservicemesh/osm/releases)


## Assumptions
For the purposes of this guide we will assume that:
- OSM is installed in `osm-system` namespace
- NGINX Ingress Controller is installed in `ingress-nginx` namespace
- Hostname for the service we expose to the Internet is `www.contoso.com`
- Port exposed to the Internet is `80`


## Steps


### 1. Install NGINX Ingress Controller
The command below uses Helm to install NGINX Ingress Controller in the `ingress-nginx` namespace. For details and other options see the [NGINX](https://docs.nginx.com/nginx-ingress-controller/installation/installation-with-helm/) or [Kubernetes Ingress](https://kubernetes.github.io/ingress-nginx/deploy/#quick-start) docs.
```bash
helm upgrade \
  --install ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace
```


### 2. Label NGINX Ingress Controller Namespace
Label the `ingress-nginx` namespace with `openservicemesh.io/monitored-by=osm`. This label ensures OSM can observe the pods in this namespace. Pods in the `ingress-nginx` namespace will **not** be augmented with an Envoy sidecar.
```bash
kubectl label namespace ingress-nginx openservicemesh.io/monitored-by=osm
```


### 3. Install Apps
For this guide we use [httpbin](https://httpbin.org/) as our sample app. You could use your own apps in lieu of [httpbin](https://httpbin.org/).

1. Create a namespace: `kubectl create namespace httpbin`

2. Add the newly created namespace to the mesh: `osm namespace add httpbin`

3. Create a new Kubernetes Service Account:
```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: httpbin
  namespace: httpbin
EOF
```

4. Create a new Kubernetes Service
```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: httpbin
  namespace: httpbin
  labels:
    app: httpbin
    service: httpbin
spec:
  selector:
    app: httpbin
  ports:
  - name: http
    port: 14001
EOF
```

5. Deploy the [httpbin](https://httpbin.org/) app:
```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: httpbin
  namespace: httpbin
  labels:
    app: httpbin
spec:
  serviceAccountName: httpbin
  containers:
  - image: kennethreitz/httpbin
    name: httpbin
    command: ["gunicorn", "-b", "0.0.0.0:14001", "httpbin:app", "-k", "gevent"]
    ports:
    - containerPort: 14001
EOF
```

Confirm the `httpbin` pod is healthy and running:
1. View the pod: `kubectl get pods --namespace httpbin`
```console
NAME      READY   STATUS    RESTARTS   AGE
httpbin   2/2     Running   0          22s
```

2. View the service: `kubectl get services --namespace httpbin`
```console
NAME      TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)     AGE
httpbin   ClusterIP   10.0.80.181   <none>        14001/TCP   22s
```

3. View the endpoint: `kubectl get endpoints --namespace httpbin`
```console
NAME      ENDPOINTS           AGE
httpbin   10.244.2.13:14001   22s
```


### 4. Edit MeshConfig CRO
OSM must be aware of the existence of an NGINX ingress pods. The NGINX pods will need a certificate to participate in the mesh. Edit the MeshConfig CRO named `osm-mesh-config` in the `osm-system` namespace:
```bash
kubectl edit MeshConfig osm-mesh-config --namespace osm-system
```

Add the `ingressGateway` configuration under `spec.certificate`:
```yaml
spec:
  certificate:
    ingressGateway:
      secret:
        name: osm-ingress-mtls
        namespace: ingress-nginx
      subjectAltNames:
          - ingress-nginx.ingress-nginx.cluster.local
      validityDuration: 24h
```
> Note: the `subjectAltNames` list (SAN) is of the form `<service-account>.<namespace>.cluster.local`. Service account and namespace must match that of the NGNIX pods.

Alternatively the MeshConfig change can be applied with a single command:
```bash
kubectl patch MeshConfig \
  osm-mesh-config \
  -n kube-system \
  -p '{"spec":{"certificate":{"ingressGateway":{"subjectAltNames":["ingress-nginx.ingress-nginx.cluster.local"], "validityDuration":"1h", "secret":{"name":"osm-ingress-mtls","namespace":"ingress-nginx"}}}}}' \
  --type=merge
```

As a result of these changes to `osm-mesh-config`, OSM Controller will create a new secret with an mTLS certificate in it.
Look for the newly created mTLS certificate: `kubectl get secrets -n osm-system osm-ingress-mtls`

```yaml
apiVersion: v1
type: kubernetes.io/tls
kind: Secret
metadata:
  name: osm-ingress-mtls
  namespace: ingress-nginx
data:
  ca.crt: ...
  tls.crt: ...
  tls.key: ...
```

All traffic from the NGINX pod (outside the mesh) to the httpbin pod (in the mesh) will be encrypted with the mTLS certificate above.


### 5. Create Ingress Resource
The following command will create the [Kubernetes Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/) resource. This will instruct NGINX Ingress Controller to route traffic for `www.contoso.com` to the backend Kubernetes service `httpbin`.

> Note: The connection from the Nginx's ingress service to the `httpbin` backend pod will be unencrypted since we aren't using TLS.

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
    nginx.ingress.kubernetes.io/proxy-ssl-secret: "ingress-nginx/osm-ingress-mtls"
    nginx.ingress.kubernetes.io/proxy-ssl-verify: "on"
spec:
  ingressClassName: nginx
  rules:
  - host: www.contoso.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: httpbin
            port:
              number: 14001
EOF
```

To allow traffic from NGINX's pods to the `httpbin` pod in the mesh we create `IngressBackend`
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
    name: ingress-nginx-controller
    namespace: ingress-nginx
  - kind: AuthenticatedPrincipal
    name: ingress-nginx.ingress-nginx.cluster.local
EOF
```


### 6. Test the setup
Clients accessing http://www.contoso.com/ will now be able to reach the `httpbin` pod.
Test this with `curl -sI http://www.contoso.com:80/get`

Alternatively get the public IP address of the newly created ingress with: `kubectl get ingress -n httpbin`
```console
NAME      CLASS   HOSTS             ADDRESS         PORTS   AGE
httpbin   nginx   www.contoso.com   20.92.132.126   80      2m38s
```

Use the public IP address to access the `httpbin` pod (example): `curl -sI -H 'Host: www.contoso.com' http://20.92.132.126:80/get`

```console
HTTP/1.1 200 OK
Date: Wed, 10 Nov 2021 02:01:37 GMT
Content-Type: application/json
Content-Length: 328
Connection: keep-alive
access-control-allow-origin: *
access-control-allow-credentials: true
x-envoy-upstream-service-time: 3
```

---
title: "Egress Policy"
description: "Accessing external services using Egress policies"
type: docs
weight: 15
---

This guide demonstrates a client within the service mesh accessing destinations external to the mesh using OSM's Egress policy API.


## Prerequisites

- Kubernetes cluster running Kubernetes v1.19.0 or greater.
- Have OSM installed.
- Have `kubectl` available to interact with the API server.
- Have `osm` CLI available for managing the service mesh.


## Demo

1. Enable egress policy if not enabled:
    ```bash
    export osm_namespace=osm-system # Replace osm-system with the namespace where OSM is installed
    kubectl patch meshconfig osm-mesh-config -n "$osm_namespace" -p '{"spec":{"featureFlags":{"enableEgressPolicy":true}}}'  --type=merge
    ```

1. Deploy the `curl` client into the `curl` namespace after enrolling its namespace to the mesh.
    ```bash
    # Create the curl namespace
    kubectl create namespace curl

    # Add the namespace to the mesh
    osm namespace add curl

    # Deploy curl client in the curl namespace
    kubectl apply -f https://raw.githubusercontent.com/openservicemesh/osm/{{< param osm_branch >}}/docs/example/manifests/samples/curl/curl.yaml -n curl
    ```

    Confirm the `curl` client pod is up and running.

    ```console
    $ kubectl get pods -n curl
    NAME                    READY   STATUS    RESTARTS   AGE
    curl-54ccc6954c-9rlvp   2/2     Running   0          20s
    ```

## HTTP Egress

1. Confirm the `curl` client is unable make the HTTP request `http://httpbin.org:80/get` to the `httpbin.org` website on port `80`.
    ```console
    $ kubectl exec $(kubectl get pod -n curl -l app=curl -o jsonpath='{.items..metadata.name}') -n curl -c curl -- curl -sI http://httpbin.org:80/get
    command terminated with exit code 7
    ```

1. Apply an Egress policy to allow the `curl` client's ServiceAccount to access the `httpbin.org` website on port `80` serving the `http` protocol.
    ```bash
    kubectl apply -f - <<EOF
    kind: Egress
    apiVersion: policy.openservicemesh.io/v1alpha1
    metadata:
      name: httpbin-80
      namespace: curl
    spec:
      sources:
      - kind: ServiceAccount
        name: curl
        namespace: curl
      hosts:
      - httpbin.org
      ports:
      - number: 80
        protocol: http
    EOF
    ```

1. Confirm the `curl` client is able to make successful HTTP requests to `http://httpbin.org:80/get`.
    ```console
    $ kubectl exec $(kubectl get pod -n curl -l app=curl -o jsonpath='{.items..metadata.name}') -n curl -c curl -- curl -sI http://httpbin.org:80/get
    HTTP/1.1 200 OK
    date: Thu, 13 May 2021 21:49:35 GMT
    content-type: application/json
    content-length: 335
    server: envoy
    access-control-allow-origin: *
    access-control-allow-credentials: true
    x-envoy-upstream-service-time: 168
    ```

1. Confirm the `curl` client can no longer make successful HTTP requests to `http://httpbin.org:80/get` when the above policy is removed.
    ```bash
    kubectl delete egress httpbin-80 -n curl
    ```

    ```console
    $ kubectl exec $(kubectl get pod -n curl -l app=curl -o jsonpath='{.items..metadata.name}') -n curl -c curl -- curl -sI http://httpbin.org:80/get
    command terminated with exit code 7
    ```

## HTTPS Egress

Since HTTPS traffic is encrypted with TLS, OSM routes HTTPS based traffic by proxying it to its original destination as a TCP stream. The Server Name Indication (SNI) indicated by the HTTPS client application in the TLS handshake is matched against hosts specified in the Egress policy.

1. Confirm the `curl` client is unable make the HTTPS request `https://httpbin.org:443/get` to the `httpbin.org` website on port `443`.
    ```console
    $ kubectl exec $(kubectl get pod -n curl -l app=curl -o jsonpath='{.items..metadata.name}') -n curl -c curl -- curl -sI https://httpbin.org:443/get
    command terminated with exit code 7
    ```

1. Apply an Egress policy to allow the `curl` client's ServiceAccount to access the `httpbin.org` website on port `443` serving the `https` protocol.
    ```bash
    kubectl apply -f - <<EOF
    kind: Egress
    apiVersion: policy.openservicemesh.io/v1alpha1
    metadata:
      name: httpbin-443
      namespace: curl
    spec:
      sources:
      - kind: ServiceAccount
        name: curl
        namespace: curl
      hosts:
      - httpbin.org
      ports:
      - number: 443
        protocol: https
    EOF
    ```

1. Confirm the `curl` client is able to make successful HTTPS requests to `https://httpbin.org:443/get`.
    ```console
    $ kubectl exec $(kubectl get pod -n curl -l app=curl -o jsonpath='{.items..metadata.name}') -n curl -c curl -- curl -sI https://httpbin.org:443/get
    HTTP/2 200
    date: Thu, 13 May 2021 22:09:36 GMT
    content-type: application/json
    content-length: 260
    server: gunicorn/19.9.0
    access-control-allow-origin: *
    access-control-allow-credentials: true
    ```

1. Confirm the `curl` client can no longer make successful HTTPS requests to `https://httpbin.org:443/get` when the above policy is removed.
    ```bash
    kubectl delete egress httpbin-443 -n curl
    ```
    ```console
    $ kubectl exec $(kubectl get pod -n curl -l app=curl -o jsonpath='{.items..metadata.name}') -n curl -c curl -- curl -sI https://httpbin.org:443/get
    command terminated with exit code 7
    ```

## TCP Egress

TCP based Egress traffic is matched against the destination port and IP address ranges specified in an egress policy. If an IP address range is not specified, traffic will be matched only based on the destination port.

1. Confirm the `curl` client is unable make the HTTPS request `https://openservicemesh.io:443` to the `openservicemesh.io` website on port `443`. Since HTTPS uses TCP as the underlying transport protocol, TCP based routing should implicitly enable access to any HTTP(s) host on the specified port.
    ```console
    $ kubectl exec $(kubectl get pod -n curl -l app=curl -o jsonpath='{.items..metadata.name}') -n curl -c curl -- curl -sI https://openservicemesh.io:443
    command terminated with exit code 7
    ```

1. Apply an Egress policy to allow the `curl` client's ServiceAccount to access the any destination on port `443` serving the `tcp` protocol.
    ```bash
    kubectl apply -f - <<EOF
    kind: Egress
    apiVersion: policy.openservicemesh.io/v1alpha1
    metadata:
      name: tcp-443
      namespace: curl
    spec:
      sources:
      - kind: ServiceAccount
        name: curl
        namespace: curl
      ports:
      - number: 443
        protocol: tcp
    EOF
    ```
    > Note: For `server-first` protocols such as `MySQL`, `PostgreSQL`, etc., where the server initiates the first bytes of data between the client and server, the protocol must be set to `tcp-server-first` to indicate to OSM to not perform protocol detection on the port. Protocol detection relies on inspecting the initial bytes of a connection, which is incompatible with `server-first` protocols. When the port's protocol is set to `tcp-server-first`, protocol detection is skipped for that port number. It is also important to note that `server-first` port numbers must not be used for other application ports that require protocol detection to performed, which means the port numbers used for `server-first` protocols must not be used with other protocols such as `HTTP` and `TCP` that require protocol detection to be performed.

1. Confirm the `curl` client is able to make successful HTTPS requests to `https://openservicemesh.io:443`.
    ```console
    $ kubectl exec $(kubectl get pod -n curl -l app=curl -o jsonpath='{.items..metadata.name}') -n curl -c curl -- curl -sI https://openservicemesh.io:443
    HTTP/2 200
    cache-control: public, max-age=0, must-revalidate
    content-length: 0
    content-type: text/html; charset=UTF-8
    date: Thu, 13 May 2021 22:35:07 GMT
    etag: "353ebaaf9573718bd1df6b817a472e47-ssl"
    strict-transport-security: max-age=31536000
    age: 0
    server: Netlify
    x-nf-request-id: 35a4f2dc-5356-45dc-9208-63e6fa162e0f-3350874
    ```

1. Confirm the `curl` client can no longer make successful HTTPS requests to `https://openservicemesh.io:443` when the above policy is removed.
    ```bash
    kubectl delete egress tcp-443 -n curl
    ```
    ```console
    $ kubectl exec $(kubectl get pod -n curl -l app=curl -o jsonpath='{.items..metadata.name}') -n curl -c curl -- curl -sI https://openservicemesh.io:443
    command terminated with exit code 7
    ```

## HTTP Egress with SMI route matches

HTTP Egress policies can specify SMI HTTPRouteGroup matches for fine grained traffic control based on HTTP methods, headers and paths.

1. Confirm the `curl` client is unable make HTTP requests to `http://httpbin.org:80/get` and `http://httpbin.org:80/status/200` to the `httpbin.org` website on port `80`.
    ```console
    $ kubectl exec $(kubectl get pod -n curl -l app=curl -o jsonpath='{.items..metadata.name}') -n curl -c curl -- curl -sI http://httpbin.org:80/get
    command terminated with exit code 7
    $ kubectl exec $(kubectl get pod -n curl -l app=curl -o jsonpath='{.items..metadata.name}') -n curl -c curl -- curl -sI http://httpbin.org:80/status/200
    command terminated with exit code 7
    ```

1. Apply an SMI HTTPRouteGroup resource to allow access to the HTTP path `/get` and an Egress policy to access the `httpbin.org` on website port `80` that matches on the SMI HTTPRouteGroup.
    ```bash
    kubectl apply -f - <<EOF
    apiVersion: specs.smi-spec.io/v1alpha4
    kind: HTTPRouteGroup
    metadata:
      name: egress-http-route
      namespace: curl
    spec:
      matches:
      - name: get
        pathRegex: /get
    ---
    kind: Egress
    apiVersion: policy.openservicemesh.io/v1alpha1
    metadata:
      name: httpbin-80
      namespace: curl
    spec:
      sources:
      - kind: ServiceAccount
        name: curl
        namespace: curl
      hosts:
      - httpbin.org
      ports:
      - number: 80
        protocol: http
      matches:
      - apiGroup: specs.smi-spec.io/v1alpha4
        kind: HTTPRouteGroup
        name: egress-http-route
    EOF
    ```

1. Confirm the `curl` client is able to make successful HTTP requests to `http://httpbin.org:80/get`.
    ```console
    $ kubectl exec $(kubectl get pod -n curl -l app=curl -o jsonpath='{.items..metadata.name}') -n curl -c curl -- curl -sI http://httpbin.org:80/get
    HTTP/1.1 200 OK
    date: Thu, 13 May 2021 21:49:35 GMT
    content-type: application/json
    content-length: 335
    server: envoy
    access-control-allow-origin: *
    access-control-allow-credentials: true
    x-envoy-upstream-service-time: 168
    ```

1. Confirm the `curl` client is unable to make successful HTTP requests to `http://httpbin.org:80/status/200`.
    ```console
    $ kubectl exec $(kubectl get pod -n curl -l app=curl -o jsonpath='{.items..metadata.name}') -n curl -c curl -- curl -sI http://httpbin.org:80/status/200
    HTTP/1.1 404 Not Found
    date: Fri, 14 May 2021 17:08:48 GMT
    server: envoy
    transfer-encoding: chunked
    ```

1. Update the matching SMI HTTPRouteGroup resource to allow requests to HTTP paths matching the regex `/status.*`.
    ```bash
    kubectl apply -f - <<EOF
    apiVersion: specs.smi-spec.io/v1alpha4
    kind: HTTPRouteGroup
    metadata:
      name: egress-http-route
      namespace: curl
    spec:
      matches:
      - name: get
        pathRegex: /get
      - name: status
        pathRegex: /status.*
    EOF
    ```

1. Confirm the `curl` client can now make successful HTTP requests to `http://httpbin.org:80/status/200`.
    ```console
    $ kubectl exec $(kubectl get pod -n curl -l app=curl -o jsonpath='{.items..metadata.name}') -n curl -c curl -- curl -sI http://httpbin.org:80/status/200
    HTTP/1.1 200 OK
    date: Fri, 14 May 2021 17:10:48 GMT
    content-type: text/html; charset=utf-8
    content-length: 0
    server: envoy
    access-control-allow-origin: *
    access-control-allow-credentials: true
    x-envoy-upstream-service-time: 188
    ```

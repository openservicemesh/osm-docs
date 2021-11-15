---
title: "TCP Traffic Routing"
description: "Set up TCP traffic routing"
type: docs
weight: 20
---

This guide demonstrates a TCP client and server application within the service mesh communicating using OSM's TCP routing capability.


## Prerequisites

- Kubernetes cluster running Kubernetes v1.19.0 or greater.
- Have OSM installed.
- Have `kubectl` available to interact with the API server.
- Have `osm` CLI available for managing the service mesh.


## Demo

The following demo shows a TCP client sending data to a `tcp-echo` server, which then echoes back the data to the client over a TCP connection.

1. Set the namespace where OSM is installed.
    ```bash
    osm_namespace=osm-system  # Replace osm-system with the namespace where OSM is installed if different
    ```

1. Deploy the `tcp-echo` service in the `tcp-demo` namespace. The `tcp-echo` service runs on port `9000` with the `appProtocol` field set to `tcp`, which indicates to OSM that TCP routing must be used for traffic directed to the `tcp-echo` service on port `9000`.
    ```bash
    # Create the tcp-demo namespace
    kubectl create namespace tcp-demo

    # Add the namespace to the mesh
    osm namespace add tcp-demo

    # Deploy the service
    kubectl apply -f https://raw.githubusercontent.com/openservicemesh/osm/{{< param osm_branch >}}/docs/example/manifests/apps/tcp-echo.yaml -n tcp-demo
    ```

    Confirm the `tcp-echo` service and pod is up and running.

    ```console
    $ kubectl get svc,po -n tcp-demo
    NAME               TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)    AGE
    service/tcp-echo   ClusterIP   10.0.216.68   <none>        9000/TCP   97s

    NAME                            READY   STATUS    RESTARTS   AGE
    pod/tcp-echo-6656b7c4f8-zt92q   2/2     Running   0          97s
    ```

1. Deploy the `curl` client into the `curl` namespace.

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

### Using Permissive Traffic Policy Mode

We will enable service discovery using [permissive traffic policy mode](/docs/guides/traffic_management/permissive_mode), which allows application connectivity to be established without the need for explicit SMI policies.

1. Enable permissive traffic policy mode
    ```bash
    kubectl patch meshconfig osm-mesh-config -n "$osm_namespace" -p '{"spec":{"traffic":{"enablePermissiveTrafficPolicyMode":true}}}' --type=merge
    ```

1. Confirm the `curl` client is able to send and receive a response from the `tcp-echo` service using TCP routing.
    ```console
    $ kubectl exec -n curl -ti "$(kubectl get pod -n curl -l app=curl -o jsonpath='{.items[0].metadata.name}')" -c curl -- sh -c 'echo hello | nc tcp-echo.tcp-demo 9000'
    echo response: hello
    ```

    The `tcp-echo` service should echo back the data sent by the client. In the above example, the client sends `hello`, and the `tcp-echo` service responds with `echo response: hello`.

### Using SMI Traffic Policy Mode

When using SMI traffic policy mode, explicit traffic policies must be configured to allow application connectivity. We will set up SMI policies to allow the `curl` client to communicate with the `tcp-echo` service on port `9000`.

1. Enable SMI traffic policy mode by disabling permissive traffic policy mode
    ```bash
    kubectl patch meshconfig osm-mesh-config -n "$osm_namespace" -p '{"spec":{"traffic":{"enablePermissiveTrafficPolicyMode":false}}}' --type=merge
    ```

1. Confirm the `curl` client is unable to send and receive a response from the `tcp-echo` service in the absence of SMI policies.
    ```console
    $ kubectl exec -n curl -ti "$(kubectl get pod -n curl -l app=curl -o jsonpath='{.items[0].metadata.name}')" -c curl -- sh -c 'echo hello | nc tcp-echo.tcp-demo 9000'
    command terminated with exit code 1
    ```

1. Configure SMI traffic access and routing policies.
    ```bash
    kubectl apply -f - <<EOF
    # TCP route to allows access to tcp-echo:9000
    apiVersion: specs.smi-spec.io/v1alpha4
    kind: TCPRoute
    metadata:
    name: tcp-echo-route
    namespace: tcp-demo
    spec:
    matches:
        ports:
        - 9000
    ---
    # Traffic target to allow curl app to access tcp-echo service using a TCPRoute
    kind: TrafficTarget
    apiVersion: access.smi-spec.io/v1alpha3
    metadata:
    name: tcp-access
    namespace: tcp-demo
    spec:
    destination:
        kind: ServiceAccount
        name: tcp-echo
        namespace: tcp-demo
    sources:
    - kind: ServiceAccount
        name: curl
        namespace: curl
    rules:
    - kind: TCPRoute
        name: tcp-echo-route
    EOF
    ```

1. Confirm the `curl` client is able to send and receive a response from the `tcp-echo` service using SMI TCP route.
    ```console
    $ kubectl exec -n curl -ti "$(kubectl get pod -n curl -l app=curl -o jsonpath='{.items[0].metadata.name}')" -c curl -- sh -c 'echo hello | nc tcp-echo.tcp-demo 9000'
    echo response: hello
---
title: "Outbound Traffic IP Range Exclusions"
description: "Excluding IP address ranges of outbound traffic from sidecar interception"
type: docs
weight: 5
---

This guide demonstrates how outbound IP address ranges can be excluded from being intercepted by OSM's proxy sidecar, so as to not subject them to service mesh filtering and routing policies.

## Prerequisites

- Kubernetes cluster running Kubernetes v1.19.0 or greater.
- Have OSM installed.
- Have `kubectl` available to interact with the API server.
- Have `osm` CLI available for managing the service mesh.


## Demo

The following demo shows an HTTP `curl` client making HTTP requests to the `httpbin.org` website directly using its IP address. We will explicitly disable the egress functionality to ensure traffic to a non-mesh destination (`httpbin.org` in this demo) is not able to egress the pod.

1. Disable mesh-wide egress passthrough.
    ```bash
    export osm_namespace=osm-system # Replace osm-system with the namespace where OSM is installed
    kubectl patch meshconfig osm-mesh-config -n "$osm_namespace" -p '{"spec":{"traffic":{"enableEgress":false}}}'  --type=merge
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

1. Retrieve the public IP address for the `httpbin.org` website. For the purpose of this demo, we will test with a single IP range to be excluded from traffic interception. In this example, we will use the IP address `54.91.118.50` represented by the IP range `54.91.118.50/32`, to make HTTP requests with and without outbound IP range exclusions configured.
    ```console
    $ nslookup httpbin.org
    Server:		172.23.48.1
    Address:	172.23.48.1#53

    Non-authoritative answer:
    Name:	httpbin.org
    Address: 54.91.118.50
    Name:	httpbin.org
    Address: 54.166.163.67
    Name:	httpbin.org
    Address: 34.231.30.52
    Name:	httpbin.org
    Address: 34.199.75.4
    ```

    > Note: Replace `54.91.118.50` with a valid IP address returned by the above command in subsequent steps.

1. Confirm the `curl` client is unable to make successful HTTP requests to the `httpbin.org` website running on `http://54.91.118.50:80`.

    ```console
    $ kubectl exec -n curl -ti "$(kubectl get pod -n curl -l app=curl -o jsonpath='{.items[0].metadata.name}')" -c curl -- curl -I http://54.91.118.50:80
    curl: (7) Failed to connect to 54.91.118.50 port 80: Connection refused
    command terminated with exit code 7
    ```

    The failure above is expected because by default outbound traffic is redirected via the Envoy proxy sidecar running on the `curl` client's pod, and the proxy subjects this traffic to service mesh policies which does not allow this traffic.

1. Program OSM to exclude the IP range `54.91.118.50/32` IP range
    ```bash
    kubectl patch meshconfig osm-mesh-config -n "$osm_namespace" -p '{"spec":{"traffic":{"outboundIPRangeExclusionList":["54.91.118.50/32"]}}}'  --type=merge
    ```

1. Confirm the MeshConfig has been updated as expected
    ```console
    # 54.91.118.50 is one of the IP addresses of httpbin.org
    $ kubectl get meshconfig osm-mesh-config -n "$osm_namespace" -o jsonpath='{.spec.traffic.outboundIPRangeExclusionList}{"\n"}'
    ["54.91.118.50/32"]
    ```

1. Restart the `curl` client pod so the updated outbound IP range exclusions can be configured. It is important to note that existing pods must be restarted to pick up the updated configuration because the traffic interception rules are programmed by the init container only at the time of pod creation.
    ```bash
    kubectl rollout restart deployment curl -n curl
    ```

    Wait for the restarted pod to be up and running.

1. Confirm the `curl` client is able to make successful HTTP requests to the `httpbin.org` website running on `http://54.91.118.50:80`
    ```console
    # 54.91.118.50 is one of the IP addresses for httpbin.org
    $ kubectl exec -n curl -ti "$(kubectl get pod -n curl -l app=curl -o jsonpath='{.items[0].metadata.name}')" -c curl -- curl -I http://54.91.118.50:80
    HTTP/1.1 200 OK
    Date: Thu, 18 Mar 2021 23:17:44 GMT
    Content-Type: text/html; charset=utf-8
    Content-Length: 9593
    Connection: keep-alive
    Server: gunicorn/19.9.0
    Access-Control-Allow-Origin: *
    Access-Control-Allow-Credentials: true
    ```

1. Confirm that HTTP requests to other IP addresses of the `httpbin.org` website that are not excluded fail
    ```console
    # 34.199.75.4 is one of the IP addresses for httpbin.org
    $ kubectl exec -n curl -ti "$(kubectl get pod -n curl -l app=curl -o jsonpath='{.items[0].metadata.name}')" -c curl -- curl -I http://34.199.75.4:80
    curl: (7) Failed to connect to 34.199.75.4 port 80: Connection refused
    command terminated with exit code 7
    ```
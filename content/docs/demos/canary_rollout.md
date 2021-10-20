---
title: "Canary Rollouts using SMI Traffic Split"
description: "Managing Canary rollouts using SMI Taffic Split"
type: docs
weight: 21
---

This guide demonstrates how to perform Canary rollouts using the SMI Traffic Split configuration.


## Prerequisites

- Kubernetes cluster running Kubernetes v1.19.0 or greater.
- Have OSM installed.
- Have `kubectl` available to interact with the API server.
- Have `osm` CLI available for managing the service mesh.


## Demo

In this demo, we will deploy an HTTP application and perform a canary rollout where a new version of the application is deployed to serve a percentage of traffic directed to the service.

To split traffic to multiple service backends, the [SMI Traffic Split API](https://github.com/servicemeshinterface/smi-spec/blob/main/apis/traffic-split/v1alpha2/traffic-split.md) will be used. More about the usage of this API can be found in the [traffic split guide](/docs/guides/traffic_management/traffic_split). For client applications to transparently split traffic to multiple service backends, it is important to note that client applications must direct traffic to the FQDN of the root service referenced in a `TrafficSplit` resource. In this demo, the `curl` client will direct traffic to the `httpbin` root service, initially backed by version `v1` of the service, and then perform a canary rollout to direct a percentage of traffic to version `v2` of the service.

The following steps demonstrate the canary rollout deployment strategy.
> Note: [Permissive traffic policy mode](/docs/guides/traffic_management/permissive_mode) is enabled to avoid the need to create explicit access control policies.

1. Enable permissive mode

    ```bash
    osm_namespace=osm-system # Replace osm-system with the namespace where OSM is installed
    kubectl patch meshconfig osm-mesh-config -n "$osm_namespace" -p '{"spec":{"traffic":{"enablePermissiveTrafficPolicyMode":true}}}'  --type=merge
    ```

1. Deploy the `curl` client into the `curl` namespace after enrolling its namespace to the mesh.

    ```bash
    # Create the curl namespace
    kubectl create namespace curl

    # Add the namespace to the mesh
    osm namespace add curl

    # Deploy curl client in the curl namespace
    kubectl apply -f https://raw.githubusercontent.com/openservicemesh/osm/main/docs/example/manifests/samples/curl/curl.yaml -n curl
    ```

    Confirm the `curl` client pod is up and running.
    ```console
    $ kubectl get pods -n curl
    NAME                    READY   STATUS    RESTARTS   AGE
    curl-54ccc6954c-9rlvp   2/2     Running   0          20s
    ```

1. Create the root `httpbin` service that clients will direct traffic to. The service has the selector `app: httpbin`.

    ```bash
    # Create the httpbin namespace
    kubectl create namespace httpbin

    # Add the namespace to the mesh
    osm namespace add httpbin

    # Create the httpbin root service and service account
    kubectl apply -f https://raw.githubusercontent.com/openservicemesh/osm/main/docs/example/manifests/samples/canary/httpbin.yaml -n httpbin
    ```

1. Deploy version `v1` of the `httpbin` service. The service `httpbin-v1` has the selector `app: httpbin, version: v1`, and the deployment `httpbin-v1` has the labels `app: httpbin, version: v1` matching the selector of both the `httpbin` root service and `httpbin-v1` service.

    ```bash
    kubectl apply -f https://raw.githubusercontent.com/openservicemesh/osm/main/docs/example/manifests/samples/canary/httpbin-v1.yaml -n httpbin
    ```

1. Create an SMI TrafficSplit resource that directs all traffic to the `httpbin-v1` service.

    ```bash
    kubectl apply -f - <<EOF
    apiVersion: split.smi-spec.io/v1alpha2
    kind: TrafficSplit
    metadata:
      name: http-split
      namespace: httpbin
    spec:
      service: httpbin.httpbin.svc.cluster.local
      backends:
      - service: httpbin-v1
        weight: 100
    EOF
    ```

1. Confirm all traffic directed to the root service FQDN `httpbin.httpbin.svc.cluster.local` is routed to the `httpbin-v1` pod. This can be verified by inspecting the HTTP response headers and confirming that the request succeeds and the pod displayed corresponds to `httpbin-v1`.

    ```console
    for i in {1..10}; do kubectl exec -n curl -ti "$(kubectl get pod -n curl -l app=curl -o jsonpath='{.items[0].metadata.name}')" -c curl -- curl -sI http://httpbin.httpbin:14001/json | egrep 'HTTP|pod'; done
    HTTP/1.1 200 OK
    pod: httpbin-v1-77c99dccc9-q2gvt
    HTTP/1.1 200 OK
    pod: httpbin-v1-77c99dccc9-q2gvt
    HTTP/1.1 200 OK
    pod: httpbin-v1-77c99dccc9-q2gvt
    HTTP/1.1 200 OK
    pod: httpbin-v1-77c99dccc9-q2gvt
    HTTP/1.1 200 OK
    pod: httpbin-v1-77c99dccc9-q2gvt
    HTTP/1.1 200 OK
    pod: httpbin-v1-77c99dccc9-q2gvt
    HTTP/1.1 200 OK
    pod: httpbin-v1-77c99dccc9-q2gvt
    HTTP/1.1 200 OK
    pod: httpbin-v1-77c99dccc9-q2gvt
    HTTP/1.1 200 OK
    pod: httpbin-v1-77c99dccc9-q2gvt
    HTTP/1.1 200 OK
    pod: httpbin-v1-77c99dccc9-q2gvt
    ```

  The above output indicates all 10 requests returned HTTP 200 OK, and were responded by the `httpbin-v1` pod.

1. Prepare the canary rollout by deploying version `v2` of the `httpbin` service. The service `httpbin-v2` has the selector `app: httpbin, version: v2`, and the deployment `httpbin-v2` has the labels `app: httpbin, version: v2` matching the selector of both the `httpbin` root service and `httpbin-v2` service.

    ```bash
    kubectl apply -f https://raw.githubusercontent.com/openservicemesh/osm/main/docs/example/manifests/samples/canary/httpbin-v2.yaml -n httpbin
    ```

1. Perform the canary rollout by updating the SMI TrafficSplit resource to split traffic directed to the root service FQDN `httpbin.httpbin.svc.cluster.local` to both the `httpbin-v1` and `httpbin-v2` services, fronting the `v1` and `v2` versions of the `httpbin` service respectively. We will distribute the weight equally to demonstrate traffic splitting.

    ```bash
    kubectl apply -f - <<EOF
    apiVersion: split.smi-spec.io/v1alpha2
    kind: TrafficSplit
    metadata:
      name: http-split
      namespace: httpbin
    spec:
      service: httpbin.httpbin.svc.cluster.local
      backends:
      - service: httpbin-v1
        weight: 50
      - service: httpbin-v2
        weight: 50
    EOF
    ```

1. Confirm traffic is split proportional to the weights assigned to the backend services. Since we configured a weight of `50` for both `v1` and `v2`, requests should be load balanced to both the versions as seen below.

    ```console
    $ for i in {1..10}; do kubectl exec -n curl -ti "$(kubectl get pod -n curl -l app=curl -o jsonpath='{.items[0].metadata.name}')" -c curl -- curl -sI http://httpbin.httpbin:14001/json | egrep 'HTTP|pod'; done
    HTTP/1.1 200 OK
    pod: httpbin-v2-6b48697db-cdqld
    HTTP/1.1 200 OK
    pod: httpbin-v1-77c99dccc9-q2gvt
    HTTP/1.1 200 OK
    pod: httpbin-v1-77c99dccc9-q2gvt
    HTTP/1.1 200 OK
    pod: httpbin-v1-77c99dccc9-q2gvt
    HTTP/1.1 200 OK
    pod: httpbin-v2-6b48697db-cdqld
    HTTP/1.1 200 OK
    pod: httpbin-v2-6b48697db-cdqld
    HTTP/1.1 200 OK
    pod: httpbin-v1-77c99dccc9-q2gvt
    HTTP/1.1 200 OK
    pod: httpbin-v2-6b48697db-cdqld
    HTTP/1.1 200 OK
    pod: httpbin-v2-6b48697db-cdqld
    HTTP/1.1 200 OK
    pod: httpbin-v1-77c99dccc9-q2gvt
    ```

    The above output indicates all 10 requests returned an HTTP 200 OK, and both `httpbin-v1` and `httpbin-v2` pods responsed to 5 requests each based on the weight assigned to them in the `TrafficSplit` configuration.
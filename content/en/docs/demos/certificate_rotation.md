---
title: "Rotating the MeshRootCertificate"
description: "Using the OSM CLI to rotate the MeshRootCertificate"
type: docs
weight: 22
---

This guide demonstrates how to use the [MeshRootCertificate](../guides/certificate_management/certificate-rotation.md) preview feature with the OSM cli. 

>  WARNING:  This feature is currently in preview and requires the `EnableMeshRootCertificate` feature flag enabled in the MeshConfig.  

## Prerequisites

- Kubernetes cluster running Kubernetes {{< param min_k8s_version >}} or greater.
- Have `kubectl` available to interact with the API server.
- Have `osm` CLI available for installing and managing the service mesh.
- Have `openssl` avalaible to view certificate information

## Demo

The following demo shows how to use the osm cli to initiate a root certificate rotation.  Learn more about this process in the [Certificate Rotation documentation](../guides/certificate_management/certificate-rotation.md).

1. Install OSM with the `EnableMeshRootCertificate` feature flag set to true.
   ```console
   osm install --set="osm.featureFlags.enableMeshRootCertificate=true"
   ```

1. Check that the default `MeshRootCertificate` was created:
   ```console
   kubectl get mrc -n osm-system 
   NAME                          ROLE
   osm-mesh-root-certificate     active
   ```

   Confirm that a Secret was created with the Root certificate:
   ```
   kubectl get secrets -n osm-system
   NAME              TYPE     DATA   AGE
   osm-ca-bundle     Opaque   2      18h
   ```

   Take note of the modulus of the original certificate:
   ```
   kubectl get secret -n osm-system  osm-ca-bundle -o jsonpath='{.data.ca\.crt}' |
    base64 -d |
    openssl x509 -modulus -noout
    Modulus=BC79C8202E8EE11DC27335E7313BB9466DEC870367F57E9BA56F1E83296B9872FCDB924DC2A9B8C012EA72F7F76356E895D56A9D2DFAF8A62E768156A81F3787A526CAA0858FD1BC854E4BFE2D27C10F7DCC4586B7AB71572406A5A9648AB317C77DF56ED74F970A007D95FD11C1A51170A2579B3AE7CFE257714611A0C1FE11D2F8F3A6499CA94536B18E751D013B283A4331C1E6830B834CEC14A2A58B11537E2AFDB2A01BD5F606237F2DB25387F2954161AF0EC99D377D0CD3813B61BE39DF427C5B412D38207A0744E4B40869294B05171E1CE45253B8B4BEA15C009D8AF3DE7832E12D668636CF2D13D8F363A79F53CEECA4B3718FE557E136B6162E27
   ```

1. Enable permissive traffic policy mode to set up automatic application connectivity.
    > Note: this is not a requirement to use `MeshRootCertificate`'s but simplifies the demo by not requiring explicit traffic policies for application connectivity.

    ```bash
    kubectl patch meshconfig osm-mesh-config -n "$osm_namespace" -p '{"spec":{"traffic":{"enablePermissiveTrafficPolicyMode":true}}}'  --type=merge
    ```

1. Deploy the `httpbin` service into the `httpbin` namespace after enrolling its namespace to the mesh. The `httpbin` service runs on port `14001`.

    ```bash
    # Create the httpbin namespace
    kubectl create namespace httpbin

    # Add the namespace to the mesh
    osm namespace add httpbin

    # Deploy httpbin service in the httpbin namespace
    kubectl apply -f https://raw.githubusercontent.com/openservicemesh/osm-docs/{{< param osm_branch >}}/manifests/samples/httpbin/httpbin.yaml -n httpbin
    ```

    Confirm the `httpbin` service and pods are up and running.

    ```console
    $ kubectl get svc -n httpbin
    NAME      TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)     AGE
    httpbin   ClusterIP   10.96.198.23   <none>        14001/TCP   20s
    ```

    ```console
    $ kubectl get pods -n httpbin
    NAME                     READY   STATUS    RESTARTS   AGE
    httpbin-5b8b94b9-lt2vs   2/2     Running   0          20s
    ```

1. Deploy the `curl` client into the `curl` namespace after enrolling its namespace to the mesh.

    ```bash
    # Create the curl namespace
    kubectl create namespace curl

    # Add the namespace to the mesh
    osm namespace add curl

    # Deploy curl client in the curl namespace
    kubectl apply -f https://raw.githubusercontent.com/openservicemesh/osm-docs/{{< param osm_branch >}}/manifests/samples/curl/curl.yaml -n curl
    ```

    Confirm the `curl` client pod is up and running.

    ```console
    $ kubectl get pods -n curl
    NAME                    READY   STATUS    RESTARTS   AGE
    curl-54ccc6954c-9rlvp   2/2     Running   0          20s
    ```

1. Confirm the `curl` client is able to access the `httpbin` service on port `14001`.

    ```console
    $ kubectl exec -n curl -ti "$(kubectl get pod -n curl -l app=curl -o jsonpath='{.items[0].metadata.name}')" -c curl -- curl -I http://httpbin.httpbin:14001
    HTTP/1.1 200 OK
    server: envoy
    date: Mon, 15 Mar 2021 22:45:23 GMT
    content-type: text/html; charset=utf-8
    content-length: 9593
    access-control-allow-origin: *
    access-control-allow-credentials: true
    x-envoy-upstream-service-time: 2
    ```

    A `200 OK` response indicates the HTTP request from the `curl` client to the `httpbin` service was successful. The traffic between the application sidecar proxies is encrypted and authenticated using `Mutual TLS (mTLS)` by leverging the initial root certificate.

2. Rotate the root certificate `MeshRootCertificate` using the osm command: 
   ```console
   osm alpha certificate rotate -w 20s

    *** This command is in Preview.  Only run in dev/test environments ***
    Found Active MeshRootCertificate [osm-mesh-root-certificate] for mesh [osm] in namespace [osm-system]

    Created new MRC [osm-mesh-root-certificate-11546fc67c238b0de2d91ca8f8331173] -> Passive role
    waiting for propagation...
    Moving MRC [osm-mesh-root-certificate-11546fc67c238b0de2d91ca8f8331173] -> Active role
    waiting for propagation...
    Moving MRC [osm-mesh-root-certificate] -> Passive role
    waiting for propagation...
    Moving MRC [osm-mesh-root-certificate] -> Inactive role
    waiting for propagation...

    OSM successfully rotated root certificate for mesh [osm] in namespace [osm-system]
    ```

3. Confirm the only existing MRC is the and certificates are different.
   ```console
   kubectl get mrc -n osm-system 
   NAME                                                           ROLE
   osm-mesh-root-certificate                                      inactive
   osm-mesh-root-certificate-11546fc67c238b0de2d91ca8f8331173     active
   ```

   Confirm that a Secret was created with the Root certificate:
   ```console
   kubectl get secrets -n osm-system
   NAME              TYPE     DATA   AGE
   osm-ca-bundle-11546fc67c238b0de2d91ca8f8331173     Opaque   2      18h
   ```

   Take note of the modulus of the new certificate (it will be different from the original):
   ```console
   kubectl get secret -n osm-system  osm-mesh-root-certificate-11546fc67c238b0de2d91ca8f8331173 -o jsonpath='{.data.ca\.crt}' |
    base64 -d | openssl x509 -modulus -noout
   Modulus=D538E9513830028C19173CA653C7609AEC1D75E110F74C4C73BA6CD6C9C8F047C99DD3ECC3A66EA874510377B631C10C2E16521D7FF18725A525D87AC2072AFDA8DEDA8802558D2A93FC5DE9678F97BCFFBA854962E105C41DF64E2748AE31952833B3F8A37C06E1D1DBBA1967ADA0EFEE26634DE19A9174E106461DDE0F84B3311D204F6F0AC5C9A6A86572320A6B37C2CFF98DD8351FFBE13E47660E30F3EB703457E424DCEAF0CD9A92EE0CE3E47F578DB7399D001409712C9605D8897CD823690D51EE6FB1F2237291D19D3A546770350A1D6403D0466D3C5F2953DCC46056F320D53A5F34B4249F5E4BCD8F1B202D1374F5B9F01D39947DF5908E59FE49
   ```

   View the certificate modulus being used by the `httpbin` service (it will be the same as the new certificate):
   ```console
   osm proxy get config_dump httpbin-b8c7bc4d9-zpm8m -n httpbin | jq -r '.configs[] | select(."@type"=="type.googleapis.com/envoy.admin.v3.SecretsConfigDump") | .dynamic_active_secrets[] | select(.name == "root-cert-for-mtls-outbound:httpbin/httpbin").secret.validation_context.trusted_ca.inline_bytes' | base64 -d | openssl x509 -modulus -noout
   Modulus=D538E9513830028C19173CA653C7609AEC1D75E110F74C4C73BA6CD6C9C8F047C99DD3ECC3A66EA874510377B631C10C2E16521D7FF18725A525D87AC2072AFDA8DEDA8802558D2A93FC5DE9678F97BCFFBA854962E105C41DF64E2748AE31952833B3F8A37C06E1D1DBBA1967ADA0EFEE26634DE19A9174E106461DDE0F84B3311D204F6F0AC5C9A6A86572320A6B37C2CFF98DD8351FFBE13E47660E30F3EB703457E424DCEAF0CD9A92EE0CE3E47F578DB7399D001409712C9605D8897CD823690D51EE6FB1F2237291D19D3A546770350A1D6403D0466D3C5F2953DCC46056F320D53A5F34B4249F5E4BCD8F1B202D1374F5B9F01D39947DF5908E59FE49
   ```

4. Confirm the `curl` client is able to access the `httpbin` service on port `14001`.
    ```console
    $ kubectl exec -n curl -ti "$(kubectl get pod -n curl -l app=curl -o jsonpath='{.items[0].metadata.name}')" -c curl -- curl -I http://httpbin.httpbin:14001
    HTTP/1.1 200 OK
    server: envoy
    date: Mon, 15 Mar 2021 22:45:23 GMT
    content-type: text/html; charset=utf-8
    content-length: 9593
    access-control-allow-origin: *
    access-control-allow-credentials: true
    x-envoy-upstream-service-time: 2
    ```

    A `200 OK` response indicates the HTTP request from the `curl` client to the `httpbin` service was successful. The traffic between the application sidecar proxies is encrypted and authenticated using `Mutual TLS (mTLS)` by leverging the new root certificate.
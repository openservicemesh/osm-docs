---
title: "External Authorization"
description: "Configure external authorization through MeshConfig."
type: docs
---
# External Authorization

## Overview
External authorization allows offloading authorization of incoming HTTP requests to a remote endpoint not necessarily running inside the proxy's context.
Proxies supporting external authorization will, for every request, issue a permission check to the remote endpoint. Due to the potential high number of RPS a system can have, external authorization has a number of tuneable settings to allow more or less control at the expense of performance (send HTTP payload or headers only, timeout, default behaviour if authorization fails to reply, etc.).

OSM allows configuring envoy's [External Authorization extension](https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_filters/ext_authz_filter) through OSM's MeshConfig.

## Limitations
Currently, authorization filtering can't be dynamically configured to be installed in subsets or groups of proxies defined by the user, and is instead globally applied for all services in the mesh when enabled.

Similarly, the filtering direction is to be statically applied to  `inbound` and `ingress` connections within the mesh, affecting any and all HTTP request made towards any service or application in the mesh when enabled.


## OSM with OPA plugin external authorization walkthrough
The following section will document how to configure external authorization in conjunction with `opa-envoy-plugin`.

We strongly suggest to go through [OPA envoy plugin's](https://github.com/open-policy-agent/opa-envoy-plugin) documentation first to further understand their configuration options and deployment models.

The following example uses a single, remote (over the network) endpoint to validate all traffic. This configuration is not recommended for a production deployment.

- First, start by deploying OSM's Demo. We will use this sample deployment to test external authorization capabilities. Please refer to [OSM's Automated Demo](https://github.com/openservicemesh/osm/tree/main/demo#how-to-run-the-osm-automated-demo) and follow the instructions.

```
# Assuming OSM repo is available
cd <PATH_TO_OSM_REPO>
demo/run-osm-demo.sh  # wait for all services to come up
```

- When OSM's demo is up and running, proceed to deploy `opa-envoy-plugin`. OSM provides a [curated standalone opa-envoy-plugin deployment chart](https://github.com/openservicemesh/osm/blob/main/docs/example/manifests/opa/deploy-opa-envoy.yaml) which exposes `opa-envoy-plugin`'s gRPC port (default `9191`) through a service, over the network. This is the endpoint that OSM will configure the proxies with when enabling external authorization. The following snippet creates an `opa` namespace and deploys `opa-envoy-plugin` in it with minimal deny-all configuration:

```
kubectl create namespace opa
kubectl apply -f https://raw.githubusercontent.com/openservicemesh/osm/main/docs/example/manifests/opa/deploy-opa-envoy.yaml
```

- Once OSM's demo is up and running, proceed to edit OSM's MeshConfig to add external authorization to the mesh. For that, configure the `inboundExternalAuthorization` to point to the remote external authorization endpoint as follows:

```
kubectl edit meshconfig osm-mesh-config -n osm-system
## <scroll to the following section>
...
inboundExternalAuthorization:
      enable: true
      address: opa.opa.svc.cluster.local
      port: 9191
      failureModeAllow: false
      statPrefix: inboundExtAuthz
      timeout: 1s
...
```

- After this step, OSM should configure all proxies to rely on the external authorization service for authorization decisions. By default, the configuration provided with `opa-envoy-plugin` will deny all requests to any sercices in the mesh. This can be checked on the logs for any of the services on the network, `403 Forbidden` whould be expected:
```
kubectl logs <bookbuyer_pod> -n bookbuyer bookbuyer
```
```
...
--- bookbuyer:[ 8 ] -----------------------------------------

Fetching http://bookstore.bookstore:14001/books-bought
Request Headers: map[Client-App:[bookbuyer] User-Agent:[Go-http-client/1.1]]
Identity: n/a
Booksbought: n/a
Server: envoy
Date: Tue, 04 May 2021 01:20:39 GMT
Status: 403 Forbidden
ERROR: response code for "http://bookstore.bookstore:14001/books-bought" is 403;  expected 200
...
```

- To further verify that external authorization is at work, verify the logs of the `opa-envoy-plugin` instance. Logs of the instance should contain json blobs which log the evaluation of every request received - key-value `result` should be present on the authorization decision taken by the configuration fed to the `opa-envoy-plugin` instance:
```
kubectl logs <opa_pod_name> -n opa
...
{"decision_id":"1df154b5-658a-47bf-ac18-be52998605da"
...
"result":false,   // resulting decision
...
"time":"2021-05-04T01:21:18Z","timestamp":"2021-05-04T01:21:18.1971808Z","type":"openpolicyagent.org/decision_logs"
}
```

- Once the previous step is verified, proceed to change the OPA policy configuration to authorize all traffic by default:

```
kubectl edit configmap opa-policy -n opa
...
...
default allow = false
```
Change the previous line value to `true`:
```
default allow = true
```

- Finally, restart `opa-envoy-plugin`. This is a necessary step as the configuration for this deployment is pushed as config, as opposed to the application fetching it.
```
kubectl rollout restart deployment opa -n opa
```

- Verify that requests to `bookbuyer` service are now being authorized:
```
--- bookbuyer:[ 2663 ] -----------------------------------------

Fetching http://bookstore.bookstore:14001/books-bought
Request Headers: map[Client-App:[bookbuyer] User-Agent:[Go-http-client/1.1]]
Identity: bookstore-v1
Booksbought: 1087
Server: envoy
Date: Tue, 04 May 2021 02:00:46 GMT
Status: 200 OK
MAESTRO! THIS TEST SUCCEEDED!

Fetching http://bookstore.bookstore:14001/buy-a-book/new
Request Headers: map[]
Identity: bookstore-v1
Booksbought: 1088
Server: envoy
Date: Tue, 04 May 2021 02:00:47 GMT
Status: 200 OK
ESC[90m2:00AMESC[0m ESC[32mINFESC[0m BooksCountV1=21490056 ESC[36mcomponent=ESC[0mdemo ESC[36mfile=ESC[0mbooks.go:167
MAESTRO! THIS TEST SUCCEEDED!
```

The output of OPA plugin can be re-verified to make sure policy decisions are being taken for each request:
```
{"decision_id":"3f29d449-7f71-4721-b93c-ad7d375e0f80",
...
"result":true,
...
"time":"2021-05-04T02:01:35Z","timestamp":"2021-05-04T02:01:35.816768454Z","type":"openpolicyagent.org/decision_logs"}
```

This example rule shows how you can use the OPA plugin to enforce authorization policies on network traffic and is not intended for production deployments. For example, enforcing authorization on every HTTP call in this manner adds significant overhead, which can be reduced using the OPA injector. For more information on using OPA, see [Open Policy Agent official documentation](https://www.openpolicyagent.org/docs/latest/).

# External Authorization

## Overview and limitations
External authorization allows offloading authorization of incoming HTTP requests to a remote endpoint not necessarily running inside the proxy's context.
Proxies supporting external authorization will, for every request, issue a permission check to the remote endpoint. Due to the potential high number of RPS a system can have, external authorization has a number of tuneable settings to allow more or less control at the expense of performance (send HTTP payload or headers only, timeout, default behaviour if authorization fails to reply, etc.).

OSM allows configuring envoy's [External Authorization extension](https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_filters/ext_authz_filter) through OSM's MeshConfig.
Authorization filter is currently applied to inmesh `inbound` and `ingress` connections affecting HTTP request made towards any service in the mesh.

The following example will show how to configure external authorization with `opa-envoy-plugin`.
Note this example will use a single, remote (over the network) endpoint to validate all traffic. This is not a suggested production deployment.

Example OSM's MeshConfig:

```
...
inboundExternalAuthorization:
  enable: true
  address: opa.opa.svc.cluster.local
  port: 9191
  statPrefix: authz_opa
  timeout: 1s
  failureModeAllow: false
```

## Example Walkthrough

- Deploy an [opa-envoy-plugin](https://github.com/open-policy-agent/opa-envoy-plugin), you will need to back the service with network connectivity for this walkthrough - we highly suggest to read carefully `ola-envoy-plugin`'s guide first.
```
kubectl apply -f https://raw.githubusercontent.com/open-policy-agent/opa-envoy-plugin/main/quick_start.yaml
```
Once `opa-envoy-plugin` is running and reachable through the network, you can proceed to deploy OSM's Demo, follow `demo/run-osm-demo.sh`
```
demo/run-osm-demo.sh  # wait for all services to come up
```

Followingly, edit OSM's meshconfig to add external authorization to the mesh pods:
```
kubectl edit meshconfig osm-mesh-config -n osm-system
...
inboundExternalAuthorization:
      enable: true
      address: opa.opa.svc.cluster.local  ## or reachable FQDN for opa plugin running
      port: 9191
      failureModeAllow: false
      statPrefix: inboundExtAuthz
      timeout: 1s

```

Traffic should fail right out of the bat:
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

You shoud also be able to see the logs in `opa-envoy-plugin` for rejected authorization calls:
```
kubectl logs opa -n opa
...
{"decision_id":"1df154b5-658a-47bf-ac18-be52998605da"
...
"result":false,   // resulting decision
...
"time":"2021-05-04T01:21:18Z","timestamp":"2021-05-04T01:21:18.1971808Z","type":"openpolicyagent.org/decision_logs"
}
```

- Now edit OPA's policy:
```
kubectl edit configmap opa-policy -n opa
```
For the sake of testing end to end, look for a default rule such as:
```
default allow = false
```
and change it to:
```
default allow = true
```


Finally, restart `opa-envoy-plugin`:
```
kubectl rollout restart deployment opa -n opa
```

- Observe that bookbuyer calls are now being allowed:
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

```
{"decision_id":"3f29d449-7f71-4721-b93c-ad7d375e0f80",
...
"result":true,
...
"time":"2021-05-04T02:01:35Z","timestamp":"2021-05-04T02:01:35.816768454Z","type":"openpolicyagent.org/decision_logs"}
```

More complex authorization code can now be added on top of OPA's policy. Also important to note that with an external entity  operating through the network, a lot of latency is introduced to authorize every HTTP call. To mitigate this overhead, OPA has it's own injector that can inject policy agents on the same pod next to envoy, reducing the stack's latency to localhost communication between envoy and OPA plugin. For additional notes on their deployment model, check their [Open Policy Agent official documentation](https://www.openpolicyagent.org/docs/latest/).
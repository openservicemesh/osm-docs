---
title: "Troubleshoot Tracing/Jaeger"
description: "How to fix common issues with OSM's tracing integration"
type: docs
---

# When tracing is not working as expected

## 1. Verify that tracing is enabled
Ensure the `enable` key in the `tracing` configuration is set to `true`:
```bash
kubectl get meshconfig osm-mesh-config -n osm-system -o jsonpath='{.spec.observability.tracing.enable}{"\n"}'
true
```

## 2. Verify the tracing values being set are as expected
If tracing is enabled, you can verify the specific `address`, `port` and `endpoint` being used for tracing in the `osm-mesh-config` resource:
```bash
kubectl get meshconfig osm-mesh-config -n osm-system -o jsonpath='{.spec.observability.tracing}{"\n"}'
```
To verify that the Envoys point to the FQDN you intend to use, check the value for the `address` key.

## 3. Verify the tracing values being used are as expected
To dig one level deeper, you may also check whether the values set by the MeshConfig are being correctly used. Use the command below to get the config dump of the pod in question and save the output in a file.
```bash
osm proxy get config_dump -n <pod-namespace> <pod-name> > <file-name>
```
Open the file in your favorite text editor and search for `envoy-tracing-cluster`. You should be able to see the tracing values in use. Example output for the bookbuyer pod:
```console
"name": "envoy-tracing-cluster",
      "type": "LOGICAL_DNS",
      "connect_timeout": "1s",
      "alt_stat_name": "envoy-tracing-cluster",
      "load_assignment": {
       "cluster_name": "envoy-tracing-cluster",
       "endpoints": [
        {
         "lb_endpoints": [
          {
           "endpoint": {
            "address": {
             "socket_address": {
              "address": "jaeger.osm-system.svc.cluster.local",
              "port_value": 9411
        [...]
```

## 4. Verify that the OSM Controller was installed with Jaeger automatically deployed [optional]
If you used automatic bring-up, you can additionally check for the Jaeger service and Jaeger deployment:
```bash
# Assuming OSM is installed in the osm-system namespace:
kubectl get services -n osm-system -l app=jaeger

NAME     TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)    AGE
jaeger   ClusterIP   10.99.2.87   <none>        9411/TCP   27m
```

```bash
# Assuming OSM is installed in the osm-system namespace:
kubectl get deployments -n osm-system -l app=jaeger

NAME     READY   UP-TO-DATE   AVAILABLE   AGE
jaeger   1/1     1            1           27m
```

## 5. Verify Jaeger pod readiness, responsiveness and health
Check if the Jaeger pod is running in the namespace you have deployed it in:
> The commands below are specific to OSM's automatic deployment of Jaeger; substitute namespace and label values for your own tracing instance as applicable:
```bash
kubectl get pods -n osm-system -l app=jaeger

NAME                     READY   STATUS    RESTARTS   AGE
jaeger-8ddcc47d9-q7tgg   1/1     Running   5          27m
```

To get information about the Jaeger instance, use `kubectl describe pod` and check the `Events` in the output.
```bash
kubectl describe pod -n osm-system -l app=jaeger
```

## External Resources
* [Jaeger Troubleshooting docs](https://www.jaegertracing.io/docs/1.22/troubleshooting/)

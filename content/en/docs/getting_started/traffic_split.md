---
title: "Configure Traffic Split"
description: "Balance traffic between services with the SMI Traffic Split API"
type: docs
weight: 4
---

# Configure Traffic Split between Two Services

We will now demonstrate how to balance traffic between two Kubernetes services, commonly known as a traffic split. We will be splitting the traffic directed to the *root* `bookstore` service between the backends `bookstore-v1` service and `bookstore-v2` service.  The `bookstore-v1` and `bookstore-v2` services are also known as leaf services.  Learn more on how to configure services for Traffic Splitting in the [traffic Splitting how-to guide](/docs/guides/traffic_management/traffic_split)

### Deploy bookstore v2 application

To demonstrate usage of SMI traffic access and split policies, we will now deploy version v2 of the bookstore application (`bookstore-v2`) - remember that if you are using openshift, you must add the security context constraint to the bookstore-v2 service account as specified in the [installation guide](/docs/guides/install/#openshift).

```bash
# Contains the bookstore-v2 Kubernetes Service, Service Account, Deployment and SMI Traffic Target resource to allow
# `bookbuyer` to communicate with `bookstore-v2` pods
kubectl apply -f https://raw.githubusercontent.com/openservicemesh/osm-docs/{{< param osm_branch >}}/manifests/apps/bookstore-v2.yaml
```

Wait for the `bookstore-v2` pod to be running in the `bookstore` namespace. Next, exit and restart the port-forward scripts in order to access v2 of bookstore:

```bash
bash <<EOF
./scripts/port-forward-bookbuyer-ui.sh &
./scripts/port-forward-bookstore-ui.sh &
./scripts/port-forward-bookstore-ui-v2.sh &
./scripts/port-forward-bookthief-ui.sh &
wait
EOF
```

- [http://localhost:8082](http://localhost:8082) - **bookstore-v2**

The `bookstore-v2` counter should be incrementing because traffic to the root `bookstore` service is configured with a label selector which selects endpoints that include both the `bookstore-v1` and `bookstore-v2` backend pods.

### Create SMI Traffic Split

Deploy the SMI traffic split policy to direct 100 percent of the traffic sent to the root `bookstore` service to the `bookstore-v1` service backend. This is necessary to ensure traffic directed to the `bookstore` service is only directed to `version v1` of the bookstore app, which includes the pods backing the `bookstore-v1` service. The `TrafficSplit` configuration will be subsequently updated to direct a percentage of traffic to `version v2` of the bookstore service using the `bookstore-v2` leaf service. 

For this reason, it is important to ensure client applications always communicate with the root service if a traffic split is desired. Otherwise the client application will need to be updated to communicate with the *root* service when a traffic split is desired.

```bash
kubectl apply -f https://raw.githubusercontent.com/openservicemesh/osm-docs/{{< param osm_branch >}}/manifests/split/traffic-split-v1.yaml
```

_Note: The root service is a Kubernetes service whose selector needs to match the pods backing the leaf services. In this demo, the root service `bookstore` has the selector `app:bookstore`, which matches both the labels `app:bookstore,version:v1` and `app:bookstore,version=v2` on the `bookstore (v1)` and `bookstore-v2` deployments respectively. The root service can be referred to in the SMI Traffic Split resource as the name of the service with or without the `.<namespace>.svc.cluster.local` suffix._

The count for the books sold from the `bookstore-v2` browser window should stop incrementing. This is because the current traffic split policy is weighted 100 for `bookstore-v1` which exludes pods backing the `bookstore-v2` service. You can verify the traffic split policy by running the following and viewing the **Backends** properties:

```bash
kubectl describe trafficsplit bookstore-split -n bookstore
```

### Split Traffic to Bookstore v2

Update the SMI Traffic Split policy to direct 50 percent of the traffic sent to the root `bookstore` service to the `bookstore` service and 50 perfect to `bookstore-v2` service by adding the `bookstore-v2` backend to the spec and modifying the weight fields.

```bash
kubectl apply -f https://raw.githubusercontent.com/openservicemesh/osm-docs/{{< param osm_branch >}}/manifests/split/traffic-split-50-50.yaml
```

Wait for the changes to propagate and observe the counters increment for `bookstore` and `bookstore-v2` in your browser windows. Both
counters should be incrementing:

- [http://localhost:8084](http://localhost:8084) - **bookstore**
- [http://localhost:8082](http://localhost:8082) - **bookstore-v2**

### Split All Traffic to Bookstore v2

Update the `bookstore-split` TrafficSplit to configure all traffic to go to `bookstore-v2`:

```bash
kubectl apply -f https://raw.githubusercontent.com/openservicemesh/osm-docs/{{< param osm_branch >}}/manifests/split/traffic-split-v2.yaml
```

Wait for the changes to propagate and observe the counters increment for `bookstore-v2` and freeze for `bookstore` in your
browser windows:

- [http://localhost:8082](http://localhost:8082) - **bookstore-v2**
- [http://localhost:8083](http://localhost:8084) - **bookstore**

Now, all traffic directed to the `bookstore` service is flowing to `bookstore-v2`.

## Next Steps

- [Configure observability with Prometheus and Grafana](/docs/getting_started/observability/)
- [Cleanup sample applications and uninstall OSM](/docs/getting_started/cleanup/)
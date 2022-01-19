---
title: "Configure Traffic Split"
description: "Balance traffic between services with the SMI Traffic Split API"
type: docs
weight: 4
---

# Configure Traffic Split between Two Services

We will now demonstrate how to balance traffic between two Kubernetes services, commonly known as a traffic split. We will be splitting the traffic directed to the root `bookstore` service between the backends `bookstore` service and `bookstore-v2` service.

### Deploy bookstore v2 application

To demonstrate usage of SMI traffic access and split policies, we will now deploy version v2 of the bookstore application (`bookstore-v2`) - remember that if you are using openshift, you must add the security context constraint to the bookstore-v2 service account as specified in the [installation guide](/docs/install/#openshift).

```bash
# Contains the bookstore-v2 Kubernetes Service, Service Account, Deployment and SMI Traffic Target resource to allow
# `bookbuyer` to communicate with `bookstore-v2` pods
kubectl apply -f https://raw.githubusercontent.com/openservicemesh/osm-docs/{{< param osm_branch >}}/manifests/apps/bookstore-v2.yaml
```

Wait for the `bookstore-v2` pod to be running in the `bookstore` namespace. Next, exit and restart the `./scripts/port-forward-all.sh` script in order to access v2 of bookstore.

- [http://localhost:8082](http://localhost:8082) - **bookstore-v2**

The counter should _not_ be incrementing because no traffic is flowing yet to the `bookstore-v2` service.

### Create SMI Traffic Split

Deploy the SMI traffic split policy to direct 100 percent of the traffic sent to the root `bookstore` service to the `bookstore` service backend:

```bash
kubectl apply -f https://raw.githubusercontent.com/openservicemesh/osm-docs/{{< param osm_branch >}}/manifests/split/traffic-split-v1.yaml
```

_Note: The root service can be any Kubernetes service. It does not have any label selectors. It also doesn't need to overlap with any of the Backend services specified in the Traffic Split resource. The root service can be referred to in the SMI Traffic Split resource as the name of the service with or without the `.<namespace>` suffix._

The count for the books sold from the `bookstore-v2` browser window should remain at 0. This is because the current traffic split policy is currently weighted 100 for `bookstore` in addition to the fact that `bookbuyer` is sending traffic to the `bookstore` service and no application is sending requests to the `bookstore-v2` service. You can verify the traffic split policy by running the following and viewing the **Backends** properties:

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

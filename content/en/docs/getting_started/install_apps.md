---
title: "Deploy Sample Applications"
description: "Deploy the sample bookstore applications"
type: docs
weight: 2
aliases: = ["/docs/install/manual_demo/"]
---

# Deploy Applications

In this section we will deploy 5 different Pods, and we will apply policies to control the traffic between them.

- `bookbuyer` is an HTTP client making requests to `bookstore`. This traffic is **permitted**.
- `bookthief` is an HTTP client and much like `bookbuyer` also makes HTTP requests to `bookstore`. This traffic should be **blocked**.
- `bookstore` is a server, which responds to HTTP requests. It is also a client making requests to the `bookwarehouse` service. This traffic is **permitted**.
- `bookwarehouse` is a server and should respond only to `bookstore`. Both `bookbuyer` and `bookthief` should be blocked.
- `mysql` is a MySQL database only reachable by `bookwarehouse`.


We are going to define and deploy traffic access policies using [SMI](https://smi-spec.io/), which will bring us to this final desired
state of allowed and blocked traffic between pods:

| from  /   to: | bookbuyer | bookthief | bookstore | bookwarehouse | mysql |
| ------------- | --------- | --------- | --------- | ------------- | ----- |
| bookbuyer     | n/a       | ❌         | ✔         | ❌             | ❌     |
| bookthief     | ❌         | n/a       | ❌         | ❌             | ❌     |
| bookstore     | ❌         | ❌         | n/a       | ✔             | ❌     |
| bookwarehouse | ❌         | ❌         | ❌         | n/a           | ✔     |
| mysql         | ❌         | ❌         | ❌         | ❌             | n/a   |


To show how to split traffic using SMI Traffic Split, we will deploy an additional application:

- `bookstore-v2` - this is the same container as the first `bookstore` we deployed, but for this demo we will assume that it is a new version of the app we need to upgrade to.

The `bookbuyer`, `bookthief`, `bookstore`, and `bookwarehouse` Pods will be in separate Kubernetes Namespaces with
the same names. `mysql` will be in the `bookwarehouse` namespace. Each new Pod in the service mesh will be injected with an Envoy sidecar container.

### Create the Namespaces

```bash
kubectl create namespace bookstore
kubectl create namespace bookbuyer
kubectl create namespace bookthief
kubectl create namespace bookwarehouse
```

### Add the new namespaces to the OSM control plane

```bash
osm namespace add bookstore bookbuyer bookthief bookwarehouse
```

Now each one of the four namespaces is labelled with `openservicemesh.io/monitored-by: osm` and also
annotated with `openservicemesh.io/sidecar-injection: enabled`. The OSM Controller, noticing the label and annotation
on these namespaces, will start injecting all **new** pods with Envoy sidecars.

### Create Pods, Services, ServiceAccounts

Create the `bookbuyer` service account and deployment:

```bash
kubectl apply -f https://raw.githubusercontent.com/openservicemesh/osm-docs/{{< param osm_branch >}}/manifests/apps/bookbuyer.yaml
```

Create the `bookthief` service account and deployment:

```bash
kubectl apply -f https://raw.githubusercontent.com/openservicemesh/osm-docs/{{< param osm_branch >}}/manifests/apps/bookthief.yaml
```

Create the `bookstore` service account, service, and deployment:

```bash
kubectl apply -f https://raw.githubusercontent.com/openservicemesh/osm-docs/{{< param osm_branch >}}/manifests/apps/bookstore.yaml
```

Create the `bookwarehouse` service account, service, and deployment:

```bash
kubectl apply -f https://raw.githubusercontent.com/openservicemesh/osm-docs/{{< param osm_branch >}}/manifests/apps/bookwarehouse.yaml
```

Create the `mysql` service account, service, and stateful set:

```bash
kubectl apply -f https://raw.githubusercontent.com/openservicemesh/osm-docs/{{< param osm_branch >}}/manifests/apps/mysql.yaml
```

### Checkpoint: What Got Installed?

A Kubernetes Deployment and Pods for each of `bookbuyer`, `bookthief`, `bookstore` and `bookwarehouse`, and a StatefulSet for `mysql`. Also, Kubernetes Services and Endpoints for `bookstore`, `bookwarehouse`, and `mysql`.

To view these resources on your cluster, run the following commands:

```bash
kubectl get pods,deployments,serviceaccounts -n bookbuyer
kubectl get pods,deployments,serviceaccounts -n bookthief

kubectl get pods,deployments,serviceaccounts,services,endpoints -n bookstore
kubectl get pods,deployments,serviceaccounts,services,endpoints -n bookwarehouse
```

In addition, a [Kubernetes Service Account](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/) was also created for each application. The Service Account serves as the application's identity which will be used later in the demo to create service-to-service access control policies.

### View the Application UIs

Set up client port forwarding with the following steps to access the applications in the Kubernetes cluster. It is best to start a new terminal session for running the port forwarding scripts to maintain the port forwarding session, while using the original terminal to continue to issue commands. The port-forward scripts will look for a `.env` file for environment variables needed to run the script. The `.env` creates the necessary variables that target the previously created namespaces. We will use the reference `.env.example` file and then run the port forwarding scripts.

In a new terminal session, run the following commands to enable port forwarding into the Kubernetes cluster from the root of the project directory (your local clone of [upstream OSM](https://github.com/openservicemesh/osm)).

```bash
cp .env.example .env
bash <<EOF
./scripts/port-forward-bookbuyer-ui.sh &
./scripts/port-forward-bookstore-ui.sh &
./scripts/port-forward-bookthief-ui.sh &
wait
EOF
```

_Note: To override the default ports, prefix the `BOOKBUYER_LOCAL_PORT`, `BOOKSTORE_LOCAL_PORT`, and/or `BOOKTHIEF_LOCAL_PORT` variable assignments to the `port-forward` scripts. For example:_

```bash
export BOOKBUYER_LOCAL_PORT=7070 BOOKTHIEF_LOCAL_PORT=7073 BOOKSTORE_LOCAL_PORT=7074
bash <<EOF
./scripts/port-forward-bookbuyer-ui.sh &
./scripts/port-forward-bookstore-ui.sh &
./scripts/port-forward-bookthief-ui.sh &
wait
EOF
```

In a browser, open up the following urls:

- [http://localhost:8080](http://localhost:8080) - **bookbuyer**
- [http://localhost:8083](http://localhost:8083) - **bookthief**
- [http://localhost:8084](http://localhost:8084) - **bookstore**

Position the windows so that you can see all of them at the same time. The header at the top of the webpage indicates the application and version.

## Next Steps

Now that the sample applications are running, [configure traffic policies](/docs/getting_started/traffic_policies/) between the applications.

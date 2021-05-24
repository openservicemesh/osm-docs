---
title: "OSM Manual Demo Guide"
description: "This manual demo shows you a step-by-step walkthrough of the automated demo."
type: docs
weight: 2
---


This OSM Manual Install Demo Guide gives step-by-step instructions to quickly demo OSM's key features.


## Prerequisites
This demo of OSM v0.8.4 requires:
  - a cluster running Kubernetes v1.18 or greater
  - a workstation capable of executing [Bash](https://en.wikipedia.org/wiki/Bash_(Unix_shell)) scripts
  - [The Kubernetes command-line tool](https://kubernetes.io/docs/tasks/tools/#kubectl) - `kubectl`



## Download and install the OSM command-line tool

The `osm` command-line tool has everything needed to install and configure Open Service Mesh.
You can find the binary on the [OSM GitHub releases page](https://github.com/openservicemesh/osm/releases/).

### For GNU/Linux and macOS

Download the 64-bit GNU/Linux or macOS binary of OSM v0.8.4:
```bash
system=$(uname -s)
release=v0.8.4
curl -L https://github.com/openservicemesh/osm/releases/download/${release}/osm-${release}-${system}-amd64.tar.gz | tar -vxzf -
./${system}-amd64/osm version
```

### For Windows

Download the 64-bit Windows OSM v0.8.4 binary via Powershell:
```powershell
wget  https://github.com/openservicemesh/osm/releases/download/v0.8.4/osm-v0.8.4-windows-amd64.zip -o osm.zip
unzip osm.zip
.\windows-amd64\osm.exe version
```

You can compile the `osm` CLI from source using [this guide](/docs/install/).



## Installing OSM on Kubernetes

After you download and unsip the `osm` binary, you are ready to install Open Service Mesh on a Kubernetes cluster:


The Install command enables:
[Prometheus](https://github.com/prometheus/prometheus),
[Grafana](https://github.com/grafana/grafana), and
[Jaeger](https://github.com/jaegertracing/jaeger) integrations.
The `OpenServiceMesh.enablePermissiveTrafficPolicy` chart value instructs OSM to ignore any policies and
let traffic flow freely between the pods. With Permissive Traffic Policy mode enabled, new pods
will be injected with Envoy, but traffic will flow through the proxy and will not be blocked.

> Note: Permissive Traffic Policy mode is an important feature for brownfield deployments (where it may take some time to craft SMI policies). Operators can design the SMI policies while existing services continue to operate as they have been before you installed OSM.

```bash
osm install \
    --set=OpenServiceMesh.enablePermissiveTrafficPolicy=true
    --set=OpenServiceMesh.deployPrometheus=true \
    --set=OpenServiceMesh.deployGrafana=true \
    --set=OpenServiceMesh.deployJaeger=true
```

> Note: This document assumes you have already installed credentials for a Kubernetes cluster in ~/.kube/config and `kubectl cluster-info` executes successfully.

This process also installed the OSM Controller in the `osm-system` namespace.


Read more on OSM's integrations with Prometheus, Grafana, and Jaeger in the [observability documentation](/docs/tasks_usage/observability/).

### OpenShift
For details on how to install OSM on OpenShift, refer to the [installation guide](/docs/install/#openshift)



## Deploy Applications

In this section you will deploy 4 different Pods. You'll also apply policies to control the traffic between these pods.

- `bookbuyer` is an HTTP client makeing requests to `bookstore`. This traffic is **permitted**.
- `bookthief` is an HTTP client and much like `bookbuyer` also makes HTTP requests to `bookstore`. This traffic should be **blocked**.
- `bookstore` is a server, which responds to HTTP requests. It is also a client making requests to the `bookwarehouse` service.
- `bookwarehouse` is a server and should respond only to `bookstore`. Both `bookbuyer` and `bookthief` should be blocked.


You will craft SMI policies, which will bring you to this final desired
state of allowed and blocked traffic between pods:

| from  /   to: | bookbuyer | bookthief | bookstore | bookwarehouse |
|---------------|-----------|-----------|-----------|---------------|
| bookbuyer     |     \     |     ❌     |     ✔    |       ❌       |
| bookthief     |     ❌     |     \     |     ❌     |       ❌       |
| bookstore     |     ❌     |     ❌     |     \     |       ✔      |
| bookwarehouse |     ❌     |     ❌     |     ❌     |      \        |


To show the SMI Traffic Split, you need to deploy an additional application:

- `bookstore-v2` - This is the same container as the first `bookstore` you deployed, but for this demo you can assume that it is a new version of the app you need to upgrade to.

The `bookbuyer`, `bookthief`, `bookstore`, and `bookwarehouse` Pods will be in separate Kubernetes Namespaces with
the same names. Each new Pod in the service mesh will be injected with an Envoy sidecar container.

> Note: The following commands *must* be run in Bash and *not* in Powershell.

### Create the Namespaces

```bash
kubectl create namespace bookstore
kubectl create namespace bookbuyer
kubectl create namespace bookthief
kubectl create namespace bookwarehouse
```

### Add the new namespaces to the OSM control plane

```bash
osm namespace add bookstore
osm namespace add bookbuyer
osm namespace add bookthief
osm namespace add bookwarehouse
```

Each one of the four namespaces is now labelled with `openservicemesh.io/monitored-by: osm`. Each is also
annotated with `openservicemesh.io/sidecar-injection: enabled`. The OSM Controller, noticing the label and annotation
on these namespaces, will start injecting all **new** pods with Envoy sidecars.

### Create Pods, Services, ServiceAccounts

Create the service accounts:
```bash
kubectl apply -f - <<EOF
---

apiVersion: v1
kind: ServiceAccount
metadata:
  name: bookbuyer
  namespace: bookbuyer

---

apiVersion: v1
kind: ServiceAccount
metadata:
  name: bookthief
  namespace: bookthief

---

apiVersion: v1
kind: ServiceAccount
metadata:
  name: bookstore
  namespace: bookstore

---

apiVersion: v1
kind: ServiceAccount
metadata:
  name: bookwarehouse
  namespace: bookwarehouse

EOF
```


Create the `bookbuyer` deployment:
```bash
kubectl apply -f - <<EOF
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: bookbuyer
  namespace: bookbuyer
spec:
  replicas: 1
  selector:
    matchLabels:
      app: bookbuyer
      version: v1
  template:
    metadata:
      labels:
        app: bookbuyer
        version: v1
    spec:
      serviceAccountName: bookbuyer
      containers:
      - name: bookbuyer
        image: openservicemesh/bookbuyer:v0.8.2
        imagePullPolicy: Always
        command: ["/bookbuyer"]
        env:
        - name: "BOOKSTORE_NAMESPACE"
          value: bookstore
EOF
```

Create the `bookthief` deployment:
```bash
kubectl apply -f - <<EOF
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: bookthief
  namespace: bookthief
spec:
  replicas: 1
  selector:
    matchLabels:
      app: bookthief
  template:
    metadata:
      labels:
        app: bookthief
        version: v1
    spec:
      serviceAccountName: bookthief
      containers:
      - name: bookthief
        image: openservicemesh/bookthief:v0.8.2
        imagePullPolicy: Always
        command: ["/bookthief"]
        env:
        - name: "BOOKSTORE_NAMESPACE"
          value: bookstore
EOF
```


Create bookstore service:
```bash
kubectl apply -f - <<EOF
---
apiVersion: v1
kind: Service
metadata:
  name: bookstore
  namespace: bookstore
  labels:
    app: bookstore
spec:
  selector:
    app: bookstore
  ports:
  - port: 14001
    name: bookstore-port
EOF
```

Create bookstore deployment:
```bash
kubectl apply -f - <<EOF
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: bookstore
  namespace: bookstore
spec:
  replicas: 1
  selector:
    matchLabels:
      app: bookstore
  template:
    metadata:
      labels:
        app: bookstore
    spec:
      serviceAccountName: bookstore
      containers:
      - name: bookstore
        image: openservicemesh/bookstore:v0.8.2
        imagePullPolicy: Always
        ports:
          - containerPort: 14001
        command: ["/bookstore"]
        args: ["--path", "./", "--port", "14001"]
        env:
        - name: BOOKWAREHOUSE_NAMESPACE
          value: bookwarehouse
        - name: IDENTITY
          value: bookstore-v1
EOF
```

Create the `bookwarehouse` service:
```bash
kubectl apply -f - <<EOF
---
apiVersion: v1
kind: Service
metadata:
  name: bookwarehouse
  namespace: bookwarehouse
  labels:
    app: bookwarehouse
spec:
  selector:
    app: bookwarehouse
  ports:
  - port: 14001
EOF
```

Create the `bookwarehouse` deployment:
```bash
kubectl apply -f - <<EOF
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: bookwarehouse
  namespace: bookwarehouse
spec:
  replicas: 1
  selector:
    matchLabels:
      app: bookwarehouse
  template:
    metadata:
      labels:
        app: bookwarehouse
        version: v1
    spec:
      serviceAccountName: bookwarehouse
      containers:
      - name: bookwarehouse
        image: openservicemesh/bookwarehouse:v0.8.2
        imagePullPolicy: Always
        command: ["/bookwarehouse"]
EOF
```


### Checkpoint: What Got Installed?

A Kubernetes Service, Deployment, and Service Account for applications `bookbuyer`, `bookthief`, `bookstore` and `bookwarehouse`.

To view these resources on your cluster, run these commands:

```bash
kubectl get deployments -n bookbuyer
kubectl get deployments -n bookthief
kubectl get deployments -n bookstore
kubectl get deployments -n bookwarehouse

kubectl get pods -n bookbuyer
kubectl get pods -n bookthief
kubectl get pods -n bookstore
kubectl get pods -n bookwarehouse

kubectl get services -n bookbuyer
kubectl get services -n bookthief
kubectl get services -n bookstore
kubectl get services -n bookwarehouse

kubectl get endpoints -n bookbuyer
kubectl get endpoints -n bookthief
kubectl get endpoints -n bookstore
kubectl get endpoints -n bookwarehouse
```

In addition to Kubernetes Services and Deployments, these commands also created a [Kubernetes Service Account](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/) for each Deployment. The Service Account serves as the application's identity which you will use later in the demo to create service-to-service access control policies.

### View the Application UIs

Use the following steps to set up client port forwarding to access the applications in the Kubernetes cluster. 

> Note: It is best to start a new terminal session to run the port forwarding script and maintain the port forwarding session. Use your original terminal to continue to issue commands. 
 
The port-forward-all.sh script looks for a `.env` file for the environment variables needed to run the script. The `.env` creates the necessary variables that target the previously created namespaces. These steps use the reference `.env.example` file and then run the port forwarding script.

In the new terminal session, run these commands to enable port forwarding into the Kubernetes cluster from the root of the project directory:

```bash
cp .env.example .env
./scripts/port-forward-all.sh
```

_Note: To override the default ports, prefix the `BOOKBUYER_LOCAL_PORT`, `BOOKSTORE_LOCAL_PORT`, `BOOKSTOREv1_LOCAL_PORT`, `BOOKSTOREv2_LOCAL_PORT`, and/or `BOOKTHIEF_LOCAL_PORT` variable assignments to the `port-forward` scripts. For example:_

```bash
BOOKBUYER_LOCAL_PORT=7070 BOOKSTOREv1_LOCAL_PORT=7071 BOOKSTOREv2_LOCAL_PORT=7072 BOOKTHIEF_LOCAL_PORT=7073 BOOKSTORE_LOCAL_PORT=7074 ./scripts/port-forward-all.sh
```

In a browser, open up the following urls:

- [http://localhost:8080](http://localhost:8080) - **bookbuyer**
- [http://localhost:8083](http://localhost:8083) - **bookthief**
- [http://localhost:8084](http://localhost:8084) - **bookstore**
- [http://localhost:8082](http://localhost:8082) - **bookstore-v2**
  - _Note: This page will not be available at this time in the demo. This will become available during the SMI Traffic Split configuration set up_

Position the browser windows so that you can see all four at the same time. Note that the header at the top of each webpage shows the application and version.

## Traffic Encryption

OSM encrypts all traffic via mTLS regardless of whether you're using access control policies or have enabled permissive traffic policy mode.

## Traffic Policy Modes

Once the applications are up and running, they can interact with each other using [permissive traffic policy mode](#permissive-traffic-policy-mode) or [SMI traffic policy mode](#smi-traffic-policy-mode). In permissive traffic policy mode, `osm-controller` automatically configures traffic between application services and does not enforce SMI policies. In the SMI policy mode, all traffic is denied by default unless explicitly allowed using a combination of SMI access and routing policies.

### How to Check Traffic Policy Mode

Check whether you have permissive traffic policy mode enabled or not. To do this, retrieve the value for the `enablePermissiveTrafficPolicyMode` key in the `osm-mesh-config` `MeshConfig` resource:

```bash
# Replace osm-system with osm-controller's namespace if using a non default namespace
kubectl get meshconfig osm-mesh-config -n osm-system -o jsonpath='{.spec.traffic.enablePermissiveTrafficPolicyMode}{"\n"}'
# Output:
# false: permissive traffic policy mode is disabled, SMI policy mode is enabled
# true: permissive traffic policy mode is enabled, SMI policy mode is disabled
```

The following sections demonstrate how to use OSM with [permissive traffic policy mode](#permissive-traffic-policy-mode) and [SMI Traffic Policy Mode](#smi-traffic-policy-mode).

## Permissive Traffic Policy Mode

The `osm-controller` can automatically configure application connectivity within the mesh in permissive traffic policy mode. To enable this:

1. During install using `osm` CLI:
  ```bash
  osm install --set=OpenServiceMesh.enablePermissiveTrafficPolicy=true
  ```

1. Post-install by patching the `osm-mesh-config` custom resource in the control plane's namespace (`osm-system` by default)
  ```bash
  kubectl patch meshconfig osm-mesh-config -n osm-system -p '{"spec":{"traffic":{"enablePermissiveTrafficPolicyMode":true}}}'  --type=merge
  ```

### Verify OSM is in permissive traffic policy mode

Before proceeding, [verify the traffic policy mode](#verify-the-traffic-policy-mode) and ensure that the `enablePermissiveTrafficPolicyMode` key is set to `true` in the `osm-mesh-config` `MeshConfig` resource. (Refer to the section above if you need to enable permissive traffic policy mode.)

In step [Deploy the Bookstore Application](#deploy-the-bookstore-application), you already deployed the applications needed to verify traffic flow in permissive traffic policy mode. The `bookstore` service you previously deployed is encoded with an identity of `bookstore-v1` for demo purposes, as can be seen in the [Deployment's manifest](https://raw.githubusercontent.com/openservicemesh/osm/release-v0.8/docs/example/manifests/apps/bookstore.yaml). The identity reflects which counter increments in the `bookbuyer` and `bookthief` UI, and the identity displayed in the `bookstore` UI.

The counter in the `bookbuyer` and in the `bookthief` UI is for the books bought or stolen respectively from `bookstore v1`. These counters will now increment:

- [http://localhost:8080](http://localhost:8080) - **bookbuyer**
- [http://localhost:8083](http://localhost:8083) - **bookthief**

The counter in the `bookstore` UI is for the books sold and should also increment:

- [http://localhost:8084](http://localhost:8084) - **bookstore**

The `bookbuyer` and `bookthief` applications can buy and steal books respectively from the newly deployed `bookstore` application. They can do this because permissive traffic policy mode is enabled. This mode allows connectivity between applications without the need for SMI traffic access policies.

You can demonstrate this operation further by disabling permissive traffic policy mode. To verify the mode is disabled, check that the counter for books bought from `bookstore` no longer increments:

```bash
kubectl patch meshconfig osm-mesh-config -n osm-system -p '{"spec":{"traffic":{"enablePermissiveTrafficPolicyMode":false}}}'  --type=merge
```

_Note: When you disable permissive traffic policy mode, SMI traffic access mode is implicitly enabled. If counters for the books increment, this could be because some SMI Traffic Access policies have been applied previously to allow such traffic._

## SMI Traffic Policy Mode

You can use SMI traffic policies for:

1. SMI access control policies to authorize traffic access between service identities
2. SMI traffic specs policies to define routing rules to associate with access control policies
3. SMI traffic split policies to direct client traffic to multiple backends based on weights

The following sections describe how to leverage each of these policies to enforce fine-grained control over traffic flow within the service mesh. Before proceeding, [verify the traffic policy mode](#verify-the-traffic-policy-mode) and ensure that the `enablePermissiveTrafficPolicyMode` key is set to `false` in the `osm-mesh-config` `MeshConfig` resource.

You can enable SMI traffic policy mode by disabling permissive traffic policy mode:

```bash
kubectl patch meshconfig osm-mesh-config -n osm-system -p '{"spec":{"traffic":{"enablePermissiveTrafficPolicyMode":false}}}'  --type=merge
```

### Deploy SMI Access Control Policies

At this point, applications do not have access to each other because you have not applied any access control policies. To confirm this, verify that none of the counters in the `bookbuyer`, `bookthief`, `bookstore`, and `bookstore-v2` UI are incrementing.

Apply the [SMI Traffic Target][1] and [SMI Traffic Specs][2] resources to define access control and routing policies for the applications to communicate:

 Deploy SMI TrafficTarget
```bash
kubectl apply -f - <<EOF
---
kind: TrafficTarget
apiVersion: access.smi-spec.io/v1alpha3
metadata:
  name: bookstore
  namespace: bookstore
spec:
  destination:
    kind: ServiceAccount
    name: bookstore
    namespace: bookstore
  rules:
  - kind: HTTPRouteGroup
    name: bookstore-service-routes
    matches:
    - buy-a-book
    - books-bought
  sources:
  - kind: ServiceAccount
    name: bookbuyer
    namespace: bookbuyer
# Bookthief is NOT allowed to talk to Bookstore
#  - kind: ServiceAccount
#    name: bookthief
#    namespace: bookthief
EOF
```

Deploy HTTPRouteGroup policy
```bash
kubectl apply -f - <<EOF
---
apiVersion: specs.smi-spec.io/v1alpha4
kind: HTTPRouteGroup
metadata:
  name: bookstore-service-routes
  namespace: bookstore
spec:
  matches:
  - name: books-bought
    pathRegex: /books-bought
    methods:
    - GET
    headers:
    - host: "bookstore.bookstore"
    - "user-agent": ".*-http-client/*.*"
    - "client-app": "bookbuyer"
  - name: buy-a-book
    pathRegex: ".*a-book.*new"
    methods:
    - GET
    headers:
    - host: "bookstore.bookstore"
EOF
```

The counters should now be incrementing for the `bookbuyer`, and `bookstore` applications:

- [http://localhost:8080](http://localhost:8080) - **bookbuyer**
- [http://localhost:8084](http://localhost:8084) - **bookstore**

Note that the counter is _not_ incrementing for the `bookthief` application:

- [http://localhost:8083](http://localhost:8083) - **bookthief**

Why is the counter not incrementing? This is because the SMI Traffic Target SMI HTTPRouteGroup resources deployed only allow `bookbuyer` to communicate with the `bookstore` and not `bookthief`.

#### Allowing the Bookthief Application to access the Mesh

Currently, the Bookthief application is not authorized to participate in the service mesh communication. To authorize this application, uncomment the lines in the [docs/example/manifests/access/traffic-access-v1.yaml](https://raw.githubusercontent.com/openservicemesh/osm/release-v0.8/docs/example/manifests/access/traffic-access-v1.yaml) to allow `bookthief` to communicate with `bookstore`. Then, re-apply the manifest and watch the change in policy propagate.

Current TrafficTarget spec with commented `bookthief` kind:

```yaml
kind: TrafficTarget
apiVersion: access.smi-spec.io/v1alpha3
metadata:
  name: bookstore-v1
  namespace: bookstore
spec:
  destination:
    kind: ServiceAccount
    name: bookstore
    namespace: bookstore
  rules:
  - kind: HTTPRouteGroup
    name: bookstore-service-routes
    matches:
    - buy-a-book
    - books-bought
  sources:
  - kind: ServiceAccount
    name: bookbuyer
    namespace: bookbuyer
  #- kind: ServiceAccount
    #name: bookthief
    #namespace: bookthief
```

Updated TrafficTarget spec with uncommented `bookthief` kind:

```yaml
kind: TrafficTarget
apiVersion: access.smi-spec.io/v1alpha3
metadata:
 name: bookstore-v1
 namespace: bookstore
spec:
 destination:
   kind: ServiceAccount
   name: bookstore
   namespace: bookstore
 rules:
 - kind: HTTPRouteGroup
   name: bookstore-service-routes
   matches:
   - buy-a-book
   - books-bought
 sources:
 - kind: ServiceAccount
   name: bookbuyer
   namespace: bookbuyer
 - kind: ServiceAccount
   name: bookthief
   namespace: bookthief
```

Re-apply the access manifest with the updates.

```bash
kubectl apply -f https://raw.githubusercontent.com/openservicemesh/osm/release-v0.8/docs/example/manifests/access/traffic-access-v1-allow-bookthief.yaml
```

The counter in the `bookthief` window starts incrementing.

- [http://localhost:8083](http://localhost:8083) - **bookthief**

Comment out the bookthief source lines in the Traffic Target object and re-apply the Traffic Access policies:

```bash
# Re-apply original SMI TrafficTarget and HTTPRouteGroup resources
kubectl apply -f https://raw.githubusercontent.com/openservicemesh/osm/release-v0.8/docs/example/manifests/access/traffic-access-v1.yaml
```

The counter in the `bookthief` window now stops incrementing.

- [http://localhost:8083](http://localhost:8083) - **bookthief**

### Configure Traffic Split between Two Services

This section demonstrates how to balance traffic between two Kubernetes services, commonly known as a traffic split. You will split the traffic directed to the root `bookstore` service between the backend's `bookstore` service and `bookstore-v2` service.

### Deploy bookstore v2 application

To demonstrate usage of SMI traffic access and split policies, deploy version v2 of the bookstore application (`bookstore-v2`):

```bash
# Contains the bookstore-v2 Kubernetes Service, Service Account, Deployment and SMI Traffic Target resource to allow
# bookbuyer to communicate with `bookstore-v2` pods
kubectl apply -f https://raw.githubusercontent.com/openservicemesh/osm/release-v0.8/docs/example/manifests/apps/bookstore-v2.yaml
```

Wait for the `bookstore-v2` pod to run in the `bookstore` namespace. Next, exit and restart the `./scripts/port-forward-all.sh` script in order to access v2 of bookstore.

- [http://localhost:8082](http://localhost:8082) - **bookstore-v2**

The counter will _not_ be incrementing because traffic is not flowing yet to the `bookstore-v2` service.

#### Create SMI Traffic Split

Deploy the SMI traffic split policy. This directs 100 percent of the traffic sent to the root `bookstore` service to the `bookstore` service backend:

```bash
kubectl apply -f - <<EOF
---
apiVersion: split.smi-spec.io/v1alpha2
kind: TrafficSplit
metadata:
  name: bookstore-split
  namespace: bookstore
spec:
  service: bookstore.bookstore # <root-service>.<namespace>
  backends:
  - service: bookstore
    weight: 100
EOF
```

_Note: The root service can be any Kubernetes service. It does not have any label selectors. It also doesn't need to overlap with any of the backend services specified in the Traffic Split resource. You can refer to the root service in the SMI Traffic Split resource as the name of the service with or without the `.<namespace>` suffix._

The count for the books sold from the `bookstore-v2` browser window should remain at 0. This is because the current traffic split policy is weighted 100 for `bookstore`. Also,   `bookbuyer` is sending traffic to the `bookstore` service and no application is sending requests to the `bookstore-v2` service. You can verify the traffic split policy by running the following to view the **Backends** properties:

```bash
kubectl describe trafficsplit bookstore-split -n bookstore
```

#### Split Traffic to Bookstore v2

Update the SMI Traffic Split policy to direct 50 percent of the traffic sent to the root `bookstore` service to the `bookstore` service and 50 percent to `bookstore-v2` service. To do this, add the `bookstore-v2` backend to the spec and modify the weight fields.

```bash
kubectl apply -f - <<EOF
---
apiVersion: split.smi-spec.io/v1alpha2
kind: TrafficSplit
metadata:
  name: bookstore-split
  namespace: bookstore
spec:
  service: bookstore.bookstore # <root-service>.<namespace>
  backends:
  - service: bookstore
    weight: 50
  - service: bookstore-v2
    weight: 50
EOF
```

Wait for the changes to propagate and to observe the counters increment for `bookstore` and `bookstore-v2` in your browser windows. Both
counters should increment:

- [http://localhost:8084](http://localhost:8084) - **bookstore**
- [http://localhost:8082](http://localhost:8082) - **bookstore-v2**

#### Split All Traffic to Bookstore v2

Update the SMI TrafficSplit policy for `bookstore` Service to configure all traffic to go to `bookstore-v2`:

```bash
kubectl apply -f - <<EOF
---
apiVersion: split.smi-spec.io/v1alpha2
kind: TrafficSplit
metadata:
  name: bookstore-split
  namespace: bookstore
spec:
  service: bookstore.bookstore # <root-service>.<namespace>
  backends:
  - service: bookstore
    weight: 0
  - service: bookstore-v2
    weight: 100
EOF
```

Wait for the changes to propagate and to observe the counters increment for `bookstore-v2` and to freeze for `bookstore` in your
browser windows:

- [http://localhost:8082](http://localhost:8082) - **bookstore-v2**
- [http://localhost:8083](http://localhost:8084) - **bookstore**

Now, all traffic directed to the `bookstore` service flows to `bookstore-v2`.

## Inspect Dashboards

You can configure OSM to deploy Grafana dashboards using the `--deploy-grafana` flag in `osm install`. **NOTE** If you still have the additional terminal session running the `./scripts/port-forward-all.sh` script, use `CTRL+C` to terminate the port forwarding. The `osm dashboard` port redirection will not work simultaneously with the port forwarding script still running. You can view the `osm dashboard` using this command:

```bash
osm dashboard
```

Navigate to http://localhost:3000 to access the Grafana dashboards. The default user name is `admin` and the default password is also `admin`. On the Grafana homepage, click on the **Home** icon. You will see a folder containing dashboards for both OSM Control Plan and OSM Data Plane.


## Cleanup

To cleanup after the demo, delete all the resources created for the demo, the OSM control plane, SMI resources, and the sample applications. To do this:

Uninstall the sample applications and SMI resources and delete their namespaces with the following command:
```bash
kubectl delete ns bookbuyer bookthief bookstore bookwarehouse
```

To uninstall OSM, run:
```bash
osm uninstall
```

For more details about uninstalling OSM, see the [uninstallation guide](../uninstallation_guide/).

[1]: https://github.com/servicemeshinterface/smi-spec/blob/v0.6.0/apis/traffic-access/v1alpha2/traffic-access.md
[2]: https://github.com/servicemeshinterface/smi-spec/blob/v0.6.0/apis/traffic-specs/v1alpha4/traffic-specs.md
[3]: https://github.com/servicemeshinterface/smi-spec/blob/v0.6.0/apis/traffic-split/v1alpha2/traffic-split.md

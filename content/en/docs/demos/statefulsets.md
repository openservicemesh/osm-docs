---
title: "Configure Stateful Applications"
description: "Secure your Kubernetes Kafka cluster with OSM"
type: docs
weight: 21
---

This guide will illustrate how to configure stateful applications with OSM and Statefulsets in Kubernetes. For this demo, we will be installing [Apache Kafka](https://kafka.apache.org/) and its backing store [Apache Zookeeper](https://zookeeper.apache.org/) and set traffic policies allowing them to talk to one another. Finally, we'll test that we're able to produce and consume messages to/from a Kafka topic with all communication encrypted via mTLS.

## Prerequisites

- Kubernetes cluster running Kubernetes {{< param min_k8s_version >}} or greater with a default [StorageClass](https://kubernetes.io/docs/concepts/storage/storage-classes/) configured.
- Have `kubectl` available to interact with the API server.
- Have OSM version >= v1.2.0 installed.
- Have `osm` CLI available for managing the service mesh.
- Have [`localProxyMode`](/docs/api_reference/config/v1alpha2/#config.openservicemesh.io/v1alpha2.LocalProxyMode) set to `PodIP` in the OSM `MeshConfig`.
  - Most applications that run in a statefulset (Apache Kafka included) require all incoming network traffic to come via the pod IP. By default, OSM configures sends traffic over localhost, so it is important to modify that behavior via this MeshConfig setting. **The default behavior will be switched in a later version of OSM**

## Demo

### Install Zookeeper

First, we need to install Apache Zookeeper, the backing metadata store for Kafka. We're going to start off by creating a namespace for our zookeeper pods and adding that namespace to our OSM mesh:

```shell
# Create a namespace for Zookeeper and add it to OSM
kubectl create ns zookeeper
osm namespace add zookeeper
```

Next, we need to configure traffic policies that will allow the Zookeepers to talk to each other once they're installed. These policies will also allow our eventual Kafka deployment to talk to Zookeeper:

```shell
kubectl apply -f - <<EOF
apiVersion: specs.smi-spec.io/v1alpha4
kind: TCPRoute
metadata:
  name: zookeeper
  namespace: zookeeper
spec:
  matches:
    ports:
    - 2181
    - 3181
---
apiVersion: specs.smi-spec.io/v1alpha4
kind: TCPRoute
metadata:
  name: zookeeper-internal
  namespace: zookeeper
spec:
  matches:
    ports:
    - 2181
    - 3181
    - 2888
    - 3888
---
kind: TrafficTarget
apiVersion: access.smi-spec.io/v1alpha3
metadata:
  name: zookeeper
  namespace: zookeeper
spec:
  destination:
    kind: ServiceAccount
    name: zookeeper
    namespace: zookeeper
  rules:
  - kind: TCPRoute
    name: zookeeper
  sources:
  - kind: ServiceAccount
    name: kafka
    namespace: kafka
---
kind: TrafficTarget
apiVersion: access.smi-spec.io/v1alpha3
metadata:
  name: zookeeper-internal
  namespace: zookeeper
spec:
  destination:
    kind: ServiceAccount
    name: zookeeper
    namespace: zookeeper
  rules:
  - kind: TCPRoute
    name: zookeeper-internal
  sources:
  - kind: ServiceAccount
    name: zookeeper
    namespace: zookeeper
EOF
```

Notice that there are 2 different `TCPRoute`s being created: one for communication between Zookeepers (4 ports allowed) and another for clients external to the zookeeper instances (only 2 ports allowed). Then, in turn, we create 2 different traffic targets. Again, one is for intra-zookeeper traffic and the other is for external clients (e.g. the "kafka" ServiceAccount in the "kafka" namespace).

Now that we've prepared our traffic policies, we're ready to install Zookeeper. For this demo, we're going to leverage the Helm chart published by Bitnami, performing a Helm install in our new `zookeeper` namespace:

```shell
# Install the Zookeeper helm chart
helm repo add bitnami https://charts.bitnami.com/bitnami
helm install zookeeper bitnami/zookeeper --set replicaCount=3 --set serviceAccount.create=true --set serviceAccount.name=zookeeper --namespace zookeeper
```

Confirm that the pods are ready in the `zookeeper` namespace:

```shell
kubectl get pod -n zookeeper

NAME                    READY   STATUS    RESTARTS   AGE
zookeeper-zookeeper-0   2/2     Running   0          4m30s
zookeeper-zookeeper-1   2/2     Running   0          4m30s
zookeeper-zookeeper-2   2/2     Running   0          4m29s
```

Let's confirm that the Zookeepers have established consensus with one another with the following command:

```shell
kubectl exec zookeeper-zookeeper-1 -c zookeeper -n zookeeper -- /opt/bitnami/zookeeper/bin/zkServer.sh status

/opt/bitnami/java/bin/java
ZooKeeper JMX enabled by default
Using config: /opt/bitnami/zookeeper/bin/../conf/zoo.cfg
Client port found: 2181. Client address: localhost. Client SSL: false.
Mode: follower
```

Zookeeper is up and running!

### Install Kafka

Now it's time to install our Kafka brokers. For the sake of this demo, we're going to install Kafka in a different namespace than our Zookeepers (similar to a multi-tenant Zookeeper deployment). First, we create a new kafka namespace and add it to our mesh:

```shell
# Create a namespace for Kafka and add it to OSM
kubectl create ns kafka
osm namespace add kafka
```

Just like Zookeeper, we need to create the appropriate traffic policies to allow the Kafka pods to talk to one another. We also allow the default service account to talk to the client-facing Kafka ports *for purposes of this demo only*. **This configuration is NOT appropriate for production**.

```shell
kubectl apply -f - <<EOF
apiVersion: specs.smi-spec.io/v1alpha4
kind: TCPRoute
metadata:
  name: kafka
  namespace: kafka
spec:
  matches:
    ports:
    - 9092
---
apiVersion: specs.smi-spec.io/v1alpha4
kind: TCPRoute
metadata:
  name: kafka-internal
  namespace: kafka
spec:
  matches:
    ports:
    - 9092
    - 9093
---
kind: TrafficTarget
apiVersion: access.smi-spec.io/v1alpha3
metadata:
  name: kafka
  namespace: kafka
spec:
  destination:
    kind: ServiceAccount
    name: kafka
    namespace: kafka
  rules:
  - kind: TCPRoute
    name: kafka
  sources:
  - kind: ServiceAccount
    name: default
    namespace: kafka
---
kind: TrafficTarget
apiVersion: access.smi-spec.io/v1alpha3
metadata:
  name: kafka-internal
  namespace: kafka
spec:
  destination:
    kind: ServiceAccount
    name: kafka
    namespace: kafka
  rules:
  - kind: TCPRoute
    name: kafka-internal
  sources:
  - kind: ServiceAccount
    name: kafka
    namespace: kafka
EOF
```

With our traffic policies configured, we're ready to install the Bitnami Kafka Helm chart in our kafka namespace:

```shell
helm install kafka bitnami/kafka --set replicaCount=3 --set zookeeper.enabled=false --set zookeeperChrootPath='/kafka-root' --set serviceAccount.create=true --set serviceAccount.name=kafka --namespace kafka --set "externalZookeeper.servers={zookeeper-zookeeper-0.zookeeper-zookeeper-headless.zookeeper.svc.cluster.local,zookeeper-zookeeper-1.zookeeper-zookeeper-headless.zookeeper.svc.cluster.local,zookeeper-zookeeper-2.zookeeper-zookeeper-headless.zookeeper.svc.cluster.local}"
```

There are a couple of important details to note here. For one, we disable the zookeeper nodes that come pre-installed in the Kafka Helm chart since we've already installed our own. Additionally, we set a specific Zookeeper chroot path for isolating our Kafka metadata from other potential clients of the Zookeeper instances. Finally, we pass in a set of service FQDNs so that Kafka can connect to our Zookeepers. We technically don't need all 3; only 1 service has to be reachable and Kafka will be able to find the other nodes on its own. Now, we just have to wait for the Kafka pods to come up:

```shell
kubectl get pod -nkafka

NAME      READY   STATUS    RESTARTS   AGE
kafka-0   3/3     Running   1          3m57s
kafka-1   3/3     Running   1          3m56s
kafka-2   3/3     Running   1          3m54s
```

Excellent! All the compute is in place

### Putting it all together

Now let's confirm that all the communication between our different components is working as it should (zookeeper->zookeeper, kakfa->kafka, and kafka->zookeeper). To do this, we're going to spin up a test pod running a kafka image so that we can produce and consume to a topic:

```shell
kubectl run --rm -it kafka-client --image docker.io/bitnami/kafka:3.1.0-debian-10-r60 --namespace kafka -- bash
```

Once you run this command, you should have a bash shell open for you to send commands to the `kafka-client` pod. In that console, run the following command:

```shell
kafka-console-producer.sh --broker-list kafka-0.kafka-headless.kafka.svc.cluster.local:9092 --topic test
```

Here, we only pass in 1 kafka broker because, just like zookeeper, the client only needs to talk to a single host in order to retrieve metadata about the other nodes in the quorum. After running this command, you should see another prompt; this is the Kafka shell and any text typed here will be serialized and sent as a Kafka message once you hit enter. Let's give it a try!

(Note: the `>` is the prompt you'll see when typing in the shell. Don't copy and paste it!)

```shell
> hello
> world
```

Great - you've sent two Kafka messages to your in-cluster Kafka brokers: "hello" and "world". Now, hit Ctrl-C a couple of times to exit the Kafka prompt and return to you bash shell. Now let's fire up a Kafka consumer to read the messages we've just written. From that bash shell, run the following command:

```shell
kafka-console-consumer.sh --bootstrap-server kafka.kafka.svc.cluster.local:9092 --topic test --from-beginning
```

You may see what appear to be error messages when you run this command, but those are most-likely symptoms of a [Kakfa broker rebalance](https://stackoverflow.com/questions/30988002/what-does-rebalancing-mean-in-apache-kafka-context), a process that's completely normal. One way or another, you should soon see your "hello" and "world" messages appear in your shell, separated by a newline. Congratulations! You've just run 2 stateful applications in Kubernetes, transparently securing the communication between all of their components using mTLS with OSM.

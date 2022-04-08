<h1> Example Architecture Overview </h1>

![Screenshot_2022-01-26_at_12.23.37](uploads/a03862096397feb36f74acd2a0b70e0d/Screenshot_2022-01-26_at_12.23.37.png)

<h1> Terminology </h1>

<h3> Knative terminologies </h3>

1. **Event Source:** An event source is a Kubernetes custom resource (CR), created by a developer or cluster administrator, that acts as a link between an event producer and an event sink. 

2. **Event Sink:** A sink can be a k8s service, including Knative Services, a Channel, or a Broker that receives events from an event source.

3. **Channel:** Channels are Kubernetes custom resources that define a single event forwarding and persistence layer.

A channel provides an event delivery mechanism that can fan-out received events, through subscriptions, to multiple destinations, or sinks. Examples of sinks include brokers and Knative services.

4. **Broker:** Brokers are Kubernetes custom resources that define an event mesh for collecting a pool of CloudEvents. Brokers provide a discoverable endpoint, status.address, for event ingress, and triggers for event delivery. Event producers can send events to a broker by POSTing the event to the status.address.url of the broker.

5. **Trigger:** A trigger represents a desire to subscribe to events from a specific broker.

Brokers and Triggers provide an "event mesh" model, which allows an event producer to deliver events to a Broker, which then distributes them uniformly to consumers by using Triggers.

<h3> Kafka Terminologies </h3>

1. **Producer:** A producer is a client that sends messages to the Kafka server to the specified topic.

2. **Consumer:** Consumers are the recipients who receive messages from the Kafka server.

3. **Broker:** Brokers can create a Kafka cluster by sharing information using Zookeeper. A broker receives messages from producers and consumers fetch messages from the broker by topic, partition, and offset.

4. **Cluster:** Kafka is a distributed system. A Kafka cluster contains multiple brokers sharing the workload.

5. **Topic:** A topic is a category name to which messages are published and from which consumers can receive messages.

6. **Partition:** Messages published to a topic are spread across a Kafka cluster into several partitions. Each partition can be associated with a broker to allow consumers to read from a topic in parallel.

<h1> Examples Resources </h1>

<h3> Knative Springboot Eventing Examples </h3>

- (Blog) Getting Started with Knative Eventing: https://salaboy.com/2020/02/20/getting-started-with-knative-2020/
- (Blog) Knative Eventing with Kafka and Spring Cloud: https://piotrminkowski.com/2021/03/12/knative-eventing-with-kafka-and-spring-cloud/
- (Video) Processing CloudEvents with Spring and Knative: https://www.youtube.com/watch?v=ok6FoZWte4U
- (Step by Step guide) Processing CloudEvents with Spring and Knative: https://github.com/trisberg/spring-knative-cloudevents-2020/blob/master/spring-knative-cloudevents.adoc

<h3> Springboot Kafka Examples </h3>

- Springboot and Kafka (reflectoring.io) :  https://reflectoring.io/spring-boot-kafka/
- Springboot and Kafka (baeldung) : https://www.baeldung.com/spring-kafka

<h1> 1. Install Knative Eventing </h1>

1. Install the required eventing custom resource definitions (CRDs) by running the command:

```
kubectl apply -f https://github.com/knative/eventing/releases/download/knative-v1.1.0/eventing-crds.yaml

```

2. Install the core components of Eventing by running the command:

```
kubectl apply -f https://github.com/knative/eventing/releases/download/knative-v1.1.0/eventing-core.yaml

```



<h1> 2. Install Kafka Channel (messaging) Layer </h1>

1. let’s first create our kafka namespace:

```
kubectl create namespace kafka

```

2. Next we apply the Strimzi install files:

```
kubectl create -f 'https://strimzi.io/install/latest?namespace=kafka' -n kafka

```
This includes ClusterRoles, ClusterRoleBindings and some Custom Resource Definitions (CRDs). The CRDs define the schemas used for declarative management of the Kafka cluster, Kafka topics and users.

3. Create the Apache Kafka cluster

```
kubectl apply -f https://strimzi.io/examples/latest/kafka/kafka-persistent-single.yaml -n kafka 

```
This will give us a small persistent Apache Kafka Cluster with one node for each - Apache Zookeeper and Apache Kafka.


<h1> 3. Install Kafka Broker Layer </h1>

1. Install the Kafka controller by running the following command:

```
kubectl apply -f https://github.com/knative-sandbox/eventing-kafka-broker/releases/download/knative-v1.1.0/eventing-kafka-controller.yaml

```

2. Install the Kafka Broker data plane by running the following command:

```
kubectl apply -f https://github.com/knative-sandbox/eventing-kafka-broker/releases/download/knative-v1.1.0/eventing-kafka-broker.yaml

```

<h1> 4. Install Kafka Source </h1>

1. Install the Apache Kafka Source by running the command:

```
kubectl apply -f https://github.com/knative-sandbox/eventing-kafka/releases/download/knative-v1.1.0/source.yaml

```





# Event driven applications with Serverless Knative Eventing - Instructions

1. Create Knative Service persist-reviews
2. Create Knative KafkaSink
3. Create Sink Binding
4. Create Event Mesh <br>
4.1 Create Knative Broker <br>
4.2 Create Knative source <br>
4.3 Create Knative Triggers <br>

## 1. Create Knative Service `persist-reviews`: 

```yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: persist-reviews
  namespace: globex-serverless-user1
spec:
  template:
    metadata:
      annotations:
        autoscaling.knative.dev/min-scale: "1"
    spec:
      securityContext:
        seccompProfile:
          type: RuntimeDefault
      containers:
        - image: quay.io/globex-sentiment-analysis/persist-reviews:latest
          volumeMounts:
            - mountPath: /deployments/config
              name: config
              readOnly: true
      volumes:
        - name: config
          secret:
            secretName: persist-reviews            
```

```sh
oc apply -f instructions-md/persist-reviews.yaml
```


## 2. Create Knative KafkaSink

In this section we will connect the Knative Services (refer to previous section) to Kafka using *Knative Sink* and *SinkBinding*. 

KafkaSink persists the incoming message (CloudEvent) to a configurable Apache Kafka Topic.

![](imagesink.png)


```yaml
apiVersion: eventing.knative.dev/v1alpha1
kind: KafkaSink
metadata:
  name: reviews-sink
  namespace: globex-serverless-user1
spec:
  bootstrapServers:
    - kafka-kafka-bootstrap.globex-mw-user1.svc.cluster.local:9092
  topic: globex.reviews
  numPartitions: 1
  contentMode: binary
  auth:
     secret:
       ref:
         name: kafka-secret

```

```sh
oc apply -f instructions-md/kafka-sink.yaml
```


## 3. Create a *Sink Binding*

SinkBinding supports decoupling the services producing the events from the Sink.

![](imagesinkbinding.png)

* Use the *Import YAML* form to create a *Sink Binding* from the `product-reviews` Quarkus Service to the KafkaSink `reviews-sink` that you created in the previous step.

```yaml
apiVersion: sources.knative.dev/v1
kind: SinkBinding
metadata:
  name: product-reviews-to-reviews-sink
  namespace: globex-serverless-user1
spec:
  sink:
    ref:
      apiVersion: eventing.knative.dev/v1alpha1
      kind: KafkaSink
      name: reviews-sink
      namespace: globex-serverless-user1
  subject:
    apiVersion: apps/v1
    kind: Deployment
    name: product-reviews
    namespace: globex-serverless-user1
````

```sh
oc apply -f instructions-md/sink-binding.yaml
```

## 4 Build the Event Mesh 

![](broker-source-filter.png)

KafkaSource reads messages in existing Apache Kafka topics, and sends them to a Knative Broker for Kafka.

Brokers use Triggers for event delivery.

A Trigger subscribes to events from a specific broker, filters them based on CloudEvents headers, and delivers them to a Knative service’s HTTP endpoint.

## 4.1 Create Knative Broker

```yaml
apiVersion: eventing.knative.dev/v1
kind: Broker
metadata:
  name: globex-broker
  namespace: globex-serverless-user1
```

```sh
oc apply -f instructions-md/knative-broker.yaml
```



## 4.2 Create Knative source
* KafkaSource reads messages in existing Apache Kafka topics, and sends those messages (CloudEvents format) to a Knative Kafka Broker.

```yaml
apiVersion: sources.knative.dev/v1beta1
kind: KafkaSource
metadata:
  name: kafka-source
  namespace: globex-serverless-user1
spec:
  bootstrapServers:
    - 'kafka-kafka-bootstrap.globex-mw-user1.svc.cluster.local:9092'
  topics:
    - globex.reviews
    - reviews.moderated
    - reviews.sentiment
  net:
    sasl:
      enable: true
      password:
        secretKeyRef:
          key: password
          name: kafka-secret
      type:
        secretKeyRef:
          key: sasl.mechanism
          name: kafka-secret
      user:
        secretKeyRef:
          key: user
          name: kafka-secret
    tls:
      caCert: {}
      cert: {}
      key: {}
  sink:
    ref:
      apiVersion: eventing.knative.dev/v1
      kind: Broker
      name: globex-broker
      namespace: globex-serverless-user1      
```

```sh
oc apply -f instructions-md/knative-source.yaml
```



## 4.3 Create Knative Triggers

You will now create triggers which will invoke the HTTP endpoint of Knative services depending on the CloudEvents headers. +
Each CloudEvents created will be tagged with specific values in the headers `ce-type` and `ce-source` which is then used by the Trigger to route them to the correct service HTTP endpoint


```yaml
apiVersion: eventing.knative.dev/v1
kind: Trigger
metadata:
  name: persist-reviews-trigger
  namespace: globex-serverless-user1
spec:
  broker: globex-broker
  filter:
    attributes:
      source: review-moderated
      type: review-moderated-event
  subscriber:
    ref:
      apiVersion: serving.knative.dev/v1
      kind: Service
      name: persist-reviews
    uri: /review/submit

---
apiVersion: eventing.knative.dev/v1
kind: Trigger
metadata:
  name: moderate-reviews-trigger
  namespace: globex-serverless-user1
spec:
  broker: globex-broker
  filter:
    attributes:
      source: submit-review
      type: submit-review-event
  subscriber:
    ref:
      apiVersion: serving.knative.dev/v1
      kind: Service
      name: aiml-moderate-reviews
    uri: /analyze
---
apiVersion: eventing.knative.dev/v1
kind: Trigger
metadata:
  name: sentiment-reviews-trigger
  namespace: globex-serverless-user1
spec:
  broker: globex-broker
  filter:
    attributes:
      source: submit-review
      type: submit-review-event
  subscriber:
    ref:
      apiVersion: serving.knative.dev/v1
      kind: Service
      name: aiml-sentiment-reviews
    uri: /analyze
```

```sh
oc apply -f  instructions-md/knative-triggers.yaml
```

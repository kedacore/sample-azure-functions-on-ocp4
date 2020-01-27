# Using Keda and Azure Functions on Openshift 4

The below will walk you through creating a Kafka instance, publishing your function to a cluster, and then publishing an agent to pull data from Twitter and publish it to Kafka. As the events land in Kafka, the function will automatically trigger and scale. Feel free to skip portions if they already exist in your cluster.

## Prerequisites

- A running OCP4 cluster
- `oc` and `kubectl` pointing to that cluster (can be achieved via `oc login` or setting `KUBECONFIG` accordingly)
- A Dockerhub account and `docker` being logged into that account to be able to push images.

## Install Keda

Install keda using the `func` CLI:

```console
$ func kubernetes install --keda-only --namespace keda
namespace/keda created
customresourcedefinition.apiextensions.k8s.io/scaledobjects.keda.k8s.io created
secret/keda-docker-auth created
serviceaccount/keda created
clusterrolebinding.rbac.authorization.k8s.io/keda created
clusterrolebinding.rbac.authorization.k8s.io/keda-hpa-role-binding created
service/keda created
deployment.apps/keda created
apiservice.apiregistration.k8s.io/v1beta1.custom.metrics.k8s.io created
apiservice.apiregistration.k8s.io/v1beta1.external.metrics.k8s.io created
```

`oc -n keda get pods` should now show the keda controller being deployed.

```console
$ oc -n keda get pods
NAME                    READY     STATUS    RESTARTS   AGE
keda-5f5674b499-n6snr   1/1       Running   0          93s
```

## Install Kafka using the Strimzi operator

### Install the operator itself

Follow the [install instructions for the Strimzi Kafka operator](https://operatorhub.io/operator/stable/strimzi-cluster-operator.v0.11.1) on OperatorHub.io. Alternatively, in your OCP4 UI, head to **Catalog**/**OperatorHub** and search for the **Strimzi Kafka** operator. Install it.

### Create a Kafka instance

Create your Kafka instance by applying the following YAML. These are the default settings of creating a Kafka instance via the UI. You can create it there as well via **Catalog**/**Installed Operators**/**Strimzi Kafka** and navigating to the *Create new* button on the Kafka resource.

```yaml
apiVersion: kafka.strimzi.io/v1alpha1
kind: Kafka
metadata:
  name: my-cluster
  namespace: openshift-operators
spec:
  kafka:
    version: 2.1.0
    replicas: 3
    listeners:
      plain: {}
      tls: {}
    config:
      offsets.topic.replication.factor: 3
      transaction.state.log.replication.factor: 3
      transaction.state.log.min.isr: 2
    storage:
      type: ephemeral
  zookeeper:
    replicas: 3
    storage:
      type: ephemeral
  entityOperator:
    topicOperator: {}
    userOperator: {}
```

You should now see Kafka pods appearing in the `openshift-operators` namespace.

```console
$ oc -n openshift-operators get pods
NAME                                          READY     STATUS    RESTARTS   AGE
my-cluster-entity-operator-55c4676687-5qrvt   3/3       Running   0          5m27s
my-cluster-kafka-0                            2/2       Running   0          6m8s
my-cluster-kafka-1                            2/2       Running   0          6m8s
my-cluster-kafka-2                            2/2       Running   0          6m8s
my-cluster-zookeeper-0                        2/2       Running   0          6m59s
my-cluster-zookeeper-1                        2/2       Running   0          6m59s
my-cluster-zookeeper-2                        2/2       Running   0          6m59s
strimzi-cluster-operator-66d7bd49f8-qlmbx     1/1       Running   0          11m
```

### Create a Kafka topic

Now create a topic `twitter` by applying the following YAML. Note the connections being made to the cluster we created above via the `strimzi.io/cluster` label. These are the default settings of creating a Kafka topic via the UI. You can create it there as well via **Catalog**/**Installed Operators**/**Strimzi Kafka** and navigating to the *Create new* button on the Kafka Topic resource.

```yaml
apiVersion: kafka.strimzi.io/v1alpha1
kind: KafkaTopic
metadata:
  name: twitter
  labels:
    strimzi.io/cluster: my-cluster
  namespace: openshift-operators
spec:
  partitions: 10
  replicas: 3
  config:
    retention.ms: 604800000
    segment.bytes: 1073741824
```

### Deploy a Kafka client

To be able to play with this Kafka instance later, install a Kafka client into the cluster by applying `client.yaml`.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: kafka-client
  namespace: default
spec:
  containers:
  - name: kafka-client
    image: confluentinc/cp-kafka:5.2.1
    command:
      - sh
      - -c
      - "exec tail -f /dev/null"
```

## Using the sample

### Clone the TypeScript based Kafka example and navigate to it

```
git clone https://github.com/kedacore/sample-typescript-kafka-azure-function
cd sample-typescript-kafka-azure-function
```

Edit the `function.json` file to point to the Kafka cluster created above.

```diff
diff --git a/KafkaTwitterTrigger/function.json b/KafkaTwitterTrigger/function.json
index 8b1da0c..3c96501 100644
--- a/KafkaTwitterTrigger/function.json
+++ b/KafkaTwitterTrigger/function.json
@@ -5,7 +5,7 @@
       "direction": "in",
       "name": "event",
       "topic": "twitter",
-      "brokerList": "kafka-cp-kafka-headless.default.svc.cluster.local:9092",
+      "brokerList": "my-cluster-kafka-bootstrap.openshift-operators:9092",
       "consumerGroup": "functions",
       "dataType": "binary"
     }
```

### Deploy the function

```
func kubernetes deploy --name twitter-function --registry $DOCKER_HUB_USERNAME
```

Alternatively, you can build and publish the image on your own and provide the --image-name instead of the --registry

### Validate the function is deployed

```
oc get deploy
```

You should see the twitter-function is deployed, but since there are no Twitter events it has 0 replicas.

### Generate test twitter data

As our function is listening for arbitrary data on the `twitter` topic and only analyzes the `text` field of the incoming JSON decoded data you can easily create a test workload to test the function works as intended. Using the the Kafka client installed earlier you can create messages on the topic that look like tweets to the function:

```
oc exec kafka-client -- sh -c 'echo "{\"text\": \"this is not something nice to say\"}" | kafka-console-producer --broker-list my-cluster-kafka-brokers.openshift-operators:9092 --topic twitter'
```

After doing that, you should see a pod appear to handle the event.

```console
$ oc get pods
NAME                                READY     STATUS    RESTARTS   AGE
twitter-function-5d684fd7b7-8bqwn   1/1       Running   0          10s
```

Now checking the logs of that pod you'll see the event coming in and being analyzed.

```console
$ oc logs twitter-function-5d684fd7b7-8bqwn
...
info: Function.KafkaTwitterTrigger.User[0]
      Kafka trigger fired!
info: Function.KafkaTwitterTrigger.User[0]
      Tweet analyzed
      Tweet text: this is not something nice to say
      Sentiment: 0.42857142857142855
info: Function.KafkaTwitterTrigger[0]
      Executed 'Functions.KafkaTwitterTrigger' (Succeeded, Id=19c8a1c7-ad66-4d35-b5b6-1cc90327995e)
...
```

### Using actual twitter data

Follow the rest of the sample from the [**Feed twitter data**](https://github.com/kedacore/sample-typescript-kafka-azure-function#feed-twitter-data) step on.

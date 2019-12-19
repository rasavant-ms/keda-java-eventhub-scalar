# Deploy Schema Registry to work with Event Hubs Kafka

## Create Event Hub

Documentation for [creating Event Hub can be found here](https://docs.microsoft.com/en-us/azure/event-hubs/event-hubs-quickstart-cli). Make sure that Standard Tier is selected and Kafka is enabled.

```bash
# Create EventHub namespace
az eventhubs namespace create --name myEventHubNamespace \
                              --resource-group myResourceGroup \
                              --capacity 10 \
                              --enable-auto-inflate true \
                              --enable-kafka true \
                              --location EastUS \
                              --maximum-throughput-units 20 \
                              --sku Standard

# Create EventHub
az eventhubs eventhub create --name myEventHub \ 
                             --namespace-name myEventHubNamespace \
                             --resource-group myResourceGroup \
                             --partition-count 10
```

## Get EventHub ConnectionString

This can be obtained from the Azure portal or via commandline as shown below

```
az eventhubs namespace authorization-rule keys list --resource-group myResourceGroup --namespace-name myEventHubNamespace --name RootManageSharedAccessKey
```
Note value of `ConnectionString`, Event Hub namespace, Event Hub name

## Deploy AKS Cluster for Confluent Schema Registry

The Azure portal was used to deploy basic AKS service using instructions in the [Azure docs](https://docs.microsoft.com/en-us/azure/aks/tutorial-kubernetes-deploy-cluster).

```bash
# Create AKS cluster
az aks create \
    --resource-group myResourceGroup \
    --name myAKSCluster \
    --node-count 1 \
    --generate-ssh-keys

# Install Kubernetes CLI
az aks install-cli

# Connect to cluster using kubectl
az aks get-credentials --resource-group myResourceGroup --name myAKSCluster

# Verify connection to cluster
$ kubectl get nodes

NAME                       STATUS   ROLES   AGE   VERSION
aks-nodepool1-12345678-0   Ready    agent   32m   v1.13.10
```

The resulting ARM template of this deployment is available as [AKS-deployment-KEDA-ARMtemplate.zip](../EventProcessorSample/AKS-deployment-KEDA-ARMtemplate.zip) for download.

## Deploy Confluent Schema Registry with Public Endpoint

```bash
# Add Helm repo
helm repo add incubator http://storage.googleapis.com/kubernetes-charts-incubator
>"incubator" has been added to your repositories

# Install the chart with self-generated release name
helm install --generate-name incubator/schema-registry --set external.enabled=true,external.servicePort=8081

# Verify Schema Registry installation by looking at pods
kubectl get pods
NAME                                          READY   STATUS    RESTARTS   AGE
schema-registry-1576616160-57bc4b7d47-rgzzq   1/1     Running   3          34h
schema-registry-1576616160-kafka-0            1/1     Running   0          34h
schema-registry-1576616160-zookeeper-0        1/1     Running   0          34h

# Verify Schema Registry installation by looking at services
kubectl get service
NAME                                            TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)                      AGE
kubernetes                                      ClusterIP      10.0.0.1       <none>           443/TCP                      6d8h
schema-registry-1576616160                      ClusterIP      10.0.15.208    <none>           8081/TCP                     34h
schema-registry-1576616160-external             LoadBalancer   10.0.120.245   52.148.149.211   8081:31565/TCP               34h
schema-registry-1576616160-kafka                ClusterIP      10.0.76.182    <none>           9092/TCP                     34h
schema-registry-1576616160-kafka-headless       ClusterIP      None           <none>           9092/TCP                     34h
schema-registry-1576616160-zookeeper            ClusterIP      10.0.2.72      <none>           2181/TCP                     34h
schema-registry-1576616160-zookeeper-headless   ClusterIP      None           <none>           2181/TCP,3888/TCP,2888/TCP   34h
```

Note `EXTERNAL-IP` value for service type `Loadbalancer` which will be used as the public endpoint for Schema Registry.

More details of the [Confluent Schema Registry Incubator can be found here](https://github.com/helm/charts/tree/master/incubator/schema-registry).

## Run test sample for Kafka Consumer and Producer using AVRO schema

Modify the `ConsumerExample.java` and `ProducerExample.java` by updating the values from above sections in the lines of code shown below:

```java
private static final String TOPIC = "<EVENTHUB_NAME>";
...
..
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "<EVENTHUB_NAMESPACE>.servicebus.windows.net:9093");
        ...
        props.put(AbstractKafkaAvroSerDeConfig.SCHEMA_REGISTRY_URL_CONFIG, "http://<SCHEMA_REGISTRY_URL>:8081");

        ...
        props.put("sasl.jaas.config", "org.apache.kafka.common.security.plain.PlainLoginModule required username=\"$ConnectionString\" password=\"<EVENTHUB_CONNECTIONSTRING>\";");

```

Build the Java application

```bash
mvn clean package
```

Run the Java `ProducerExample` application

```bash
mvn exec:java -Dexec.mainClass="ProducerExample"
```

You will see multiple messages being sent to the EventHub using AVRO schema for Payment defined in [`Payment.avsc`](./src/main/resources/avro/io/confluent/examples/clients/basicavro/Payment.avsc)

Now run the Java `ConsumerExample` application

```bash
mvn exec:java -Dexec.mainClass="ConsumerExample"
```

You will see the same number of messages being received by the EventHub using AVRO schema for Payment.

More details of these changes can be found in the [Event Hubs with Kafka tutorials here](https://github.com/Azure/azure-event-hubs-for-kafka/tree/master/tutorials/schema-registry).


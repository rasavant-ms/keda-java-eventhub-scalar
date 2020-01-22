# Deploying Java Kafka Event Consumer to AKS with KEDA autoscalar for nodes

# Deploy AKS cluster

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

The resulting ARM template of this deployment is available as [AKS-deployment-KEDA-ARMtemplate.zip](./AKS-deployment-KEDA-ARMtemplate.zip) for download.

# Deploy KEDA on the AKS cluster

KEDA was deployed on the AKS cluster using Helm.

Detailed instructions for this are available [here](https://dev.azure.com/banco-itau/b1-ted-openshift-poc/_git/prep?path=%2Fprep-aks-keda%2FREADME.md&_a=preview).

```bash
# Add KEDA Helm repo
helm repo add kedacore https://kedacore.github.io/charts

# Update KEDA Helm repo
helm repo update

# Install KEDA Helm chart for Helm 3
kubectl create namespace keda
helm install keda kedacore/keda --namespace keda

# Verify KEDA operator was deployed correctly
kubectl -n keda get pods

NAME                            READY     STATUS    RESTARTS   AGE
keda-operator-576c5dfcdf-bjmtd   2/2       Running   0          42s
```

# Create Event Hub to act as Scalar

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

# Create Storage account to store checkpoints

Details to [create Storage account and blob container can be found here](https://docs.microsoft.com/en-us/azure/storage/blobs/storage-quickstart-blobs-cli).

```bash
# Create storage account
az storage account create \
    --name myStorageAccount \
    --resource-group myResourceGroup \
    --location westus \
    --sku Standard_LRS \
    --kind StorageV2

# Create container
az storage container create --name myContainer
```

# Store secrets in `deploy-secret.yaml`

## Get EventHub primary sasKey and ConnectionString

This can be obtained from the Azure portal or via commandline as shown below

```
az eventhubs namespace authorization-rule keys list --resource-group myResourceGroup --namespace-name myEventHubNamespace --name RootManageSharedAccessKey
```
Copy value of `primarykey` and `ConnectionString`

## Get Storage account ConnectionString

This can be obtained from the Azure portal or via commandline as shown below

```bash
az storage account keys list -g MyResourceGroup -n MyStorageAccount
```

Build the storage account key as follows

```
DefaultEndpointsProtocol=https;AccountName=<storage_account_name>;AccountKey=<storage_account_key>;EndpointSuffix=core.windows.net
```

## Base64 encoding of secrets

All the secrets need to be base64 encoded as shown

```bash
echo "<eventhub_connectionstring>" | base64
```

Enter the base64 encoded secrets in the `deploy-secret.yaml` file for corresponding values

```bash
STORAGE_CONNECTIONSTRING: 
EVENTHUB_CONNECTIONSTRING: 
BLOB_CONTAINER: 
CONSUMER_GROUP: 
EVENTHUB_NAMESPACE: 
EVENTHUB_NAME: 
EVENTHUB_SAS_KEY:
```

# Build and Dockerize JAVA application

The Java application can be built independently with

```bash
mvn clean package
```

Verify that the `Dockerfile` has the right path for target

```bash
# Create docker image for Java producer application
docker build -t <tag_name> .

# Run docker image to verify it is working
docker run -t <tag_name>
>Registering host named eph-05d6a558-9d49-4ad8-9a00-3bc89ea35fe4
>Press enter to stop.
>SAMPLE: Partition 1 is opening
>SAMPLE: Partition 0 is opening
```

Create ACR to store docker image

```bash
# Create ACR
az acr create --resource-group MyResourceGroup --name myacr --sku Basic

# Merge AKS cluster to current context
az aks get-credentials --name myAKSCluster --resource-group MyResourceGroup

# Integrate ACR to existing AKS cluster
az aks update -n myAKSCluster -g MyResourceGroup --attach-acr myacr
```

Upload docker image to ACR

```bash
# Login to ACR
az acr login --name myacr

# Get ACR server name
az acr list --resource-group MyResourceGroup --query "[].{acrLoginServer:loginServer}" --output table
>AcrLoginServer
------------------
myacr.azurecr.io

# Tag docker image to created ACR
docker tag <image_tag> myacr.azurecr.io/<image_tag>

# Publish docker image to created ACR
docker push myacr.azurecr.io/<image_tag>
```

# Deploy secrets and application to AKS

```bash
# Deploy secrets
kubectl apply -f deploy-secret.yaml
>secret/consumer-secrets created
```

Make sure that the image name and tag in the `deploy-eventhub-processor.yaml` file matches the docker image pushed to ACR in the previous section

```bash
# Deploy Java application as Deployment and ScaledObject details
kubectl apply -f deploy-eventhub-processor.yaml
>deployment.apps/java-consumer created
>scaledobject.keda.k8s.io/java-consumer-scalar created
```
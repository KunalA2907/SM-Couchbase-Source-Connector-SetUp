## Deploy SM Couchbase Source Connector with Self Managed Kafka Connect Pod on Kubernetes & connect it to Confluent Cloud


### 📌 Pre-Requisites:

1. Aws EKS cluster & EBS CSI Driver for K8 which is used to deploy Confluent for Kubernetes have been setup before: https://docs.aws.amazon.com/eks/latest/userguide/create-cluster.html
2. Install and Set Up kubectl on macOS: https://kubernetes.io/docs/tasks/tools/install-kubectl-macos/
3. Install Helm3 Charts: https://helm.sh/docs/intro/install/
4. Couchbase Cluster - Bucket - Scope - Collections & input Document (Json or Blob Data) is already created before


### 📌 Set-Up:

1. Create a Namespace
```
 - kubectl create ns <namespace name>
 - kubectl config set-context --current --namespace=<namespace name>
```
2. Deploy Confluent for Kubernetes

```
- helm repo add confluentinc https://packages.confluent.io/helm
- helm repo update 

- helm upgrade --install operator confluentinc/confluent-for-kubernetes -n <namespace name>
``` 
3. Create file ccloud-credentials.txt containing your cluster api key and secret

```
username=kafka-api-key
password=kafka-api-secret
```
4. Create file ccloud-sr-credentials.txt containing api key and secret for Schema Registry of your Environment

```
username=schema-registry-api-key
password=schema-registry-api-secret
```
5. Create Kubernetes Secrets for Confluent Cloud API Key and Confluent Cloud Schema Registry API Key(Replace ./ccloud-credentials.txt and ./ccloud-sr-credentials.txt to your path)

```
- kubectl create secret generic ccloud-credentials --from-file=plain.txt=./ccloud-credentials.txt 

- kubectl create secret generic ccloud-sr-credentials --from-file=basic.txt=./ccloud-sr-credentials.txt 
```
6. Create Yaml file for self-managed Kafka Connect connecting to Confluent Cloud(Refer this link) & also for Control Center

<ul> 
 <li>Example Yaml file for Connect Cluster with overriding connector Properties enabled is mentioned below:
 <li>Change the cluster bootstrap url and schema registry url below</li>
</ul>

#### 📌 Connect.yaml

```
apiVersion: platform.confluent.io/v1beta1
kind: Connect
metadata:
  name: **connect-demo**
  namespace: confluent 
spec:
  replicas: 1
  configOverrides:
    server:
      - connector.client.config.override.policy=All
  image:
    application: confluentinc/cp-server-connect:7.7.0
    init: confluentinc/confluent-init-container:2.9.0
  build:
    type: onDemand
    onDemand:
      plugins:
        locationType: confluentHub
        confluentHub:
          - name: kafka-connect-datagen
            owner: confluentinc
            version: 0.6.0
          - name: kafka-connect-couchbase
            owner: couchbase
            version: 4.2.3
  dependencies:
    kafka:
      bootstrapEndpoint: **pkc-xxxxx.us-east-2.aws.confluent.cloud:9092**  ➔ update the Bootsrap Server Endpoint
      authentication:
        type: plain
        jaasConfig:
          secretRef: cluster-demo-credentials
      tls:
        enabled: true
        ignoreTrustStoreConfig: true
    schemaRegistry:
      url: **https://psrc-xxxxx.us-east-2.aws.confluent.cloud**  ➔ update the SR Endpoint
      authentication:
        type: basic
        basic:
          secretRef: ccloud-sr-credentials
```
#### 📌 Control-Center.yaml

```
---
apiVersion: platform.confluent.io/v1beta1
kind: ControlCenter
metadata:
  name: controlcenter 
  namespace: confluent
spec:
  replicas: 1
  image:
    application: confluentinc/cp-enterprise-control-center:7.7.0
    init: confluentinc/confluent-init-container:2.9.0
  dataVolumeCapacity: 100Gi
  configOverrides:
    server:
      - confluent.metrics.topic.max.message.bytes=8388608  
      - confluent.controlcenter.mode.enable=management
  dependencies:
    kafka:
      bootstrapEndpoint: **pkc-xxxxx.us-east-2.aws.confluent.cloud:9092**  ➔ update the Bootsrap Server Endpoint
      authentication:
        type: plain
        jaasConfig:
          secretRef: cluster-demo-credentials
      tls:
        enabled: true
        ignoreTrustStoreConfig: true
    schemaRegistry:
      url: **https://pxxx-l6oz3.us-east-2.aws.confluent.cloud**  ➔ update the SR Endpoint
      authentication:
        type: basic
        basic:
          secretRef: ccloud-sr-credentials
    connect:
      - name: connect-demo
        url: http://connect-demo.confluent.svc.cluster.local:8083
```
7. Deploy self-managed Kafka Connect connecting to Confluent Cloud & Control Center

```
- kubectl apply -f connect.yaml -n confluent

- kubectl apply -f Control-center.yaml -n confluent
```

8. Once it is deployed check the status of the connect cluster using the below command to see whether the connect cluster is running

```
kubectl get pods -n <namespace name>
```
9. Now for deploying the connector create a file named postgres.yaml and add your Couchbase configurations. I have shared the sample below. Please refer to the link for all the  configuration parameters .
```
---
apiVersion: platform.confluent.io/v1beta1
kind: Connector
metadata:
  name: test-couchbase-source
  namespace: confluent
spec:
  class: "com.couchbase.connect.kafka.CouchbaseSourceConnector"
  taskMax: 1
  connectClusterRef:
    name: connect-demo
  configs:
    couchbase.topic: "test-default"
    couchbase.seed.nodes: **"Node Hostname"**
    couchbase.bootstrap.timeout": "10s"
    couchbase.bucket: **"travel-sample"** ➔ update your bucket name
    couchbase.username: **"Cluster Usr"** ➔ update your Cluster Username
    couchbase.password: **"Cluster Pss"** ➔ update your Cluster Password
    key.converter: "org.apache.kafka.connect.storage.StringConverter"
    couchbase.source.handler: "com.couchbase.connect.kafka.handler.source.RawJsonSourceHandler"
    value.converter: "org.apache.kafka.connect.converters.ByteArrayConverter"
    couchbase.event.filter: "com.couchbase.connect.kafka.filter.AllPassFilter"
    couchbase.stream.from: "SAVED_OFFSET_OR_BEGINNING"
    couchbase.compression: "ENABLED"
    couchbase.flow.control.buffer: "16m"
    couchbase.persistence.polling.interval: "100ms"
    couchbase.collections: "inventory.airline"
    couchbase.enable.tls: "true"
    value.converter.schema.registry.url: **"https://psrc-xxxx.us-east-2.aws.confluent.cloud"** ➔ update the SR endpoint
    value.converter.basic.auth.credentials.source: "USER_INFO"
    value.converter.schema.registry.basic.auth.user.info: **"ZO54HFZPNKAV26AB:l6N4kk2ScdtXa6x6us7bUvwVAYK6GH+hUhvWDPaGU5jMaLiI+D0QTo6q+HLgrdRU" ** ➔ update as this SR api key: SR password
```
10. You can deploy the connecter using following command:
```
kubectl apply -f couchbase_source_connector.yaml 
```
11. You can verify the connecter status on Control Center - by Accessing Control Center web UI using port-forwarding:

<ul>
 <li>Use the URL to access the control center http://localhost:9021</li>
</ul>

```
kubectl port-forward controlcenter-0 -n <namespace name> 9021:9021
```
12. At the end, you can visit your Confluent Cloud Cluster and observe that the topic named "test-default" has been automatically created. It is receiving data from the Scope.Collections ~ inventory.airlines.

13. Tear Down
    
<ul>
<li>Shut down Confluent Platform and the data:</li>
</ul>

```
kubectl delete -f connect.yaml
kubectl delete -f Control-Center.yaml
helm delete operator

```

 

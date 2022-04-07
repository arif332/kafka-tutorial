

- [1. Kafka Strimzi Install on Kubernetes (K3s)](#1-kafka-strimzi-install-on-kubernetes-k3s)
- [2. K3s Kubernetes Cluster using K3d](#2-k3s-kubernetes-cluster-using-k3d)
- [3. Kafa Installation procedure using strimzi](#3-kafa-installation-procedure-using-strimzi)
- [4. Send and receive messages](#4-send-and-receive-messages)
- [5. Single Node Kafka Cluster Definition](#5-single-node-kafka-cluster-definition)
- [6. References](#6-references)
- [7. Appendix](#7-appendix)
  - [7.1. Logs from Kubernetes cluster](#71-logs-from-kubernetes-cluster)


# 1. Kafka Strimzi Install on Kubernetes (K3s)

Installed K3s cluster using K3d on Apple Silicon laptop. After that followed kafka installation procedure using strimzi operator. 


# 2. K3s Kubernetes Cluster using K3d 
A K3s cluster can be initiated using below command. Make sure that K3d already installed.
```bash
# one control node and two worker node
k3d cluster create kafkaCluster1 --servers 1  --agents 2
```


# 3. Kafa Installation procedure using strimzi
Create a namespace for kafka
```bash
kubectl create namespace kafka
```

Create operator in kafka namespace.
```bash
kubectl create -f 'https://strimzi.io/install/latest?namespace=kafka' -n kafka

kubectl get pod -n kafka --watch
kubectl logs deployment/strimzi-cluster-operator -n kafka -f
```

Provision the Apache Kafka cluster
```bash
# Apply the `Kafka` Cluster CR file, you might modify the parameter value which can be done by downloading the config yaml file locally and apply local config file
# kubectl apply -f https://strimzi.io/examples/latest/kafka/kafka-persistent-single.yaml -n kafka 
# kubectl apply -f install/single-node-kafka-using-strimz/kafka-persistent-single.yaml -n kafka 
kubectl apply -f kafka-persistent-single.yaml -n kafka 
kubectl wait kafka/kafka-single-node-cluster --for=condition=Ready --timeout=300s -n kafka 

# check status 
% kubectl wait kafka/kafka-single-node-cluster --for=condition=Ready --timeout=300s -n kafka 
kafka.kafka.strimzi.io/kafka-single-node-cluster condition met
% 
```


# 4. Send and receive messages

Send message using a producer:
```bash
kubectl -n kafka run kafka-producer -ti --image=quay.io/strimzi/kafka:0.28.0-kafka-3.1.0 --rm=true --restart=Never -- bin/kafka-console-producer.sh\
     --broker-list kafka-single-node-cluster-kafka-bootstrap:9092 --topic my-topic
```


Receve message in another terminal:
```bash
kubectl -n kafka run kafka-consumer -ti --image=quay.io/strimzi/kafka:0.28.0-kafka-3.1.0 --rm=true --restart=Never -- bin/kafka-console-consumer.sh\
     --bootstrap-server kafka-single-node-cluster-kafka-bootstrap:9092 --topic my-topic --from-beginning
```


# 5. Single Node Kafka Cluster Definition

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: kafka-single-node-cluster
spec:
  kafka:
    version: 3.1.0
    replicas: 1
    listeners:
      - name: plain
        port: 9092
        type: internal
        tls: false
      - name: tls
        port: 9093
        type: internal
        tls: true
    config:
      offsets.topic.replication.factor: 1
      transaction.state.log.replication.factor: 1
      transaction.state.log.min.isr: 1
      default.replication.factor: 1
      min.insync.replicas: 1
      inter.broker.protocol.version: "3.1"
    storage:
      type: jbod
      volumes:
      - id: 0
        type: persistent-claim
        size: 20Gi
        deleteClaim: false
  zookeeper:
    replicas: 1
    storage:
      type: persistent-claim
      size: 20Gi
      deleteClaim: false
  entityOperator:
    topicOperator: {}
    userOperator: {}
```    


# 6. References
- https://strimzi.io/quickstarts/
- https://github.com/strimzi/strimzi-kafka-operator/tree/0.28.0/examples/kafka
- https://github.com/strimzi/strimzi-kafka-operator/tree/main/examples/mirror-maker
- https://strimzi.io/docs/operators/latest/overview.html


# 7. Appendix

## 7.1. Logs from Kubernetes cluster
```bash
% k get pod -n kafka
NAME                                                         READY   STATUS    RESTARTS   AGE
strimzi-cluster-operator-587cb79468-hplrr                    1/1     Running   0          21m
kafka-single-node-cluster-zookeeper-0                        1/1     Running   0          12m
kafka-single-node-cluster-kafka-0                            1/1     Running   0          11m
kafka-single-node-cluster-entity-operator-75bcb454c4-ltbc8   3/3     Running   0          10m
kafka-producer                                               1/1     Running   0          2m38s


% k get kafkatopics.kafka.strimzi.io -n kafka
NAME                                                                                               CLUSTER                     PARTITIONS   REPLICATION FACTOR   READY
strimzi-store-topic---effb8e3e057afce1ecf67c3f5d8e4e3ff177fc55                                     kafka-single-node-cluster   1            1                    True
strimzi-topic-operator-kstreams-topic-store-changelog---b75e702040b99be8a9263134de3507fc0cc4017b   kafka-single-node-cluster   1            1                    True
consumer-offsets---84e7a678d08f4bd226872e5cdd4eb527fadc1c6a                                        kafka-single-node-cluster   50           1                    True
my-topic                                                                                           kafka-single-node-cluster   1            1                    True
% 

```
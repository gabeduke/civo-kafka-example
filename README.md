# Kafka CIVO Cloud

This example deploys a Kafka cluster using [Bonsai Cloud Kafka Operator](https://github.com/banzaicloud/kafka-operator)

## Setup

_Prerequisites_:

- Civo CLI and access to the Beta program
- Helm3
- Kubectl
- Bonsai Helm Repository: `helm repo add banzaicloud-stable https://kubernetes-charts.banzaicloud.com/`

### Provision Kubernetes

_Civo_:

```bash
civo k8s create \
  --nodes 3 \
  --save --switch --wait \
  kafka
```

### Provision Dependencies

_Cert-Manager_:

```bash
# create cert-manager deployment
kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v0.11.0/cert-manager.yaml
```

_Prometheus Operator_:

```bash
# create the prometheus operator
kubectl apply -n default -f https://raw.githubusercontent.com/coreos/prometheus-operator/master/bundle.yaml
```

_Zookeeper Operator_:

```bash
# create zookeeper namespace
kubectl create namespace zookeeper

# create zk operator
helm upgrade --install \
  --namespace zookeeper \
  --wait \
  zookeeper-operator banzaicloud-stable/zookeeper-operator
```

_Zookeeper_:

```bash
# create zookeeper cluster
kubectl create --namespace zookeeper -f zookeeper.yaml
```

_Kafka Operator_:

```bash
# create the kafka namespace
kubectl create namespace kafka

# install kafka operator with prometheus alerts
helm upgrade --install \
  --namespace kafka \
  --values kafka-prometheus-alerts.yaml \
  kafka-operator banzaicloud-stable/kafka-operator
```

_Kafka_:

```bash
# create the kafka cluster
kubectl create -n kafka -f kafka.yaml

# create the service monitor
kubectl create -n kafka -f kafka-prometheus.yaml
```

### Validate

_Dashboard_:

```bash
# proxy to the cruise-control dashboard for kafka maintenance (may take a couple minutes)
kubectl port-forward -n kafka svc/kafka-cruisecontrol-svc 8090:8090 &
echo http://localhost:8090
```

Kafka cluster must be finished provisioning before the topic can be applied:

```bash
# create a topic to which we can produce/consume
kubectl apply -f civo-topic.yaml
```

Run the following two commands in separate terminals:

_Produce_:

```bash
# run a producer in a pod
kubectl run kafka-producer \
  -n kafka -it --rm=true --restart=Never \
  --image=wurstmeister/kafka:2.12-2.3.0 \
    -- /opt/kafka/bin/kafka-console-producer.sh \
    --broker-list kafka-headless:29092 \
    --topic civo-topic
```

_Consume_:

```bash
# run a consumer in a pod
kubectl run kafka-consumer \
  -n kafka -it --rm=true --restart=Never \
  --image=wurstmeister/kafka:2.12-2.3.0 \
    -- /opt/kafka/bin/kafka-console-consumer.sh \
    --bootstrap-server kafka-headless:29092 \
    --from-beginning \
    --topic civo-topic
```

## Clean

```bash
# kill any dangling proxies
killall kubectl

# clean up the kubernetes cluster
civo k8s delete kafka
```

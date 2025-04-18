
## Deploy Confluent for Kubernetes

This workflow scenario assumes you are using the namespace `confluent`.

1. Set up the Helm Chart:

```
helm repo add confluentinc https://packages.confluent.io/helm
```

2. Install Confluent For Kubernetes using Helm:

```
helm upgrade --install operator confluentinc/confluent-for-kubernetes -n confluent 
```

3. Check that the Confluent For Kubernetes pod comes up and is running:

```
kubectl get pods -n confluent
```

## Set up cluster

### Deploy KRaft broker and controller

    kubectl apply -f $TUTORIAL_HOME/kraftbroker_controller.yaml

### Produce and consume from the topics
```
kubectl -n confluent exec -it kafka-0 -- bash

seq 5 | kafka-console-producer --topic demotopic --broker-list kafka.confluent.svc.cluster.local:9092

kafka-console-consumer --from-beginning --topic demotopic --bootstrap-server  kafka.confluent.svc.cluster.local:9092
1
2
3
4
5
```

## Tear down Cluster
    kubectl delete -f $TUTORIAL_HOME/kraftbroker_controller.yaml
    helm uninstall operator -n confluent
    kubectl delete namespace confluent

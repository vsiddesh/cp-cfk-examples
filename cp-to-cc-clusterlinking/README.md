# Setup Topic and Clusterlink in CCLOUD

In this example, CFK cluster is in SASL plain mode and the CCLOUD cluster is configured by default with SASL-SSL

## Table of Contents
- [Basic setup](#basic-setup)
- [CCLOUD setup](#ccloud-setup)
- [CFK setup](#cfk-setup)
    * [Create required secrets](#create-required-secrets)
    * [Deploy](#deploy)
- [Create source initiated clusterlink on CCLOUD](#create-source-initiated-clusterlink-on-ccloud)
    * [Run cluster link test](#run-cluster-link-test-1)
        + [Exec into source kafka pod](#exec-into-source-kafka-pod-1)
        + [Create kafka.properties](#create-kafkaproperties-1)
        + [Produce in source kafka cluster](#produce-in-source-kafka-cluster-1)
        + [Login to ccloud with confluent CLI.](#login-to-ccloud-with-confluent-cli-2)
        + [Consume message](#consume-message-1)

## Basic setup
- Set the tutorial directory for this tutorial under the directory you downloaded
  the tutorial files:
```
export TUTORIAL_HOME=<Tutorial directory>/hybrid/clusterlink/ccloud-as-destination-cluster
```
- Create a namespace `kubectl create ns confluent`

```
- Deploy Confluent for Kubernetes (CFK)
```
    helm repo add confluentinc https://packages.confluent.io/helm
    helm upgrade --install confluent confluentinc/confluent-for-kubernetes
```
- Check that the Confluent For Kubernetes pod comes up and is running:
```
  kubectl get pods
```

## CCLOUD setup
- Create a dedicated a cluster in the Confluent Cloud. Dedicated cluster is required for cluster linking. Standard cluster should be good for creating topics
- Create an API Key with `Global Access`. You can create API key in `Cluster Overview -> Data Integration -> API Keys`
- Create a file `basic.txt` with API Key and API secret in this format
```
username=<API-KEY>
password=<API-SECRET>
```
- Create  a secret with this API key and secret
```
kubectl -n confluent create secret generic restclass-ccloud --from-file=basic.txt=basic.txt
```

- Create a file `jaas.text` with API Key and API secret in this format
```
sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required username="<API-KEY>" password="<API-SECRET>";
```
- Create  a secret with this API key and secret
```
kubectl -n confluent create secret generic jaasconfig-ccloud --from-file=plain-jaas.conf=jaas.txt
```

Note: If you need the server certificates for the ccloud cluster, use the following command. This will return the server cert and the ca cert in that order
Save these certs in `fullchain.pem` and `cacert.pem`
```
openssl s_client -showcerts -servername pkc-3wkro.us-west4.gcp.confluent.cloud \
-connect pkc-3wkro.us-west4.gcp.confluent.cloud:443  < /dev/null
```
- We can create a secret with ccloud certs if we want to use them.
```
kubectl -n confluent create secret generic ccloud-tls-certs \
--from-file=fullchain.pem=fullchain.pem --from-file=cacerts.pem=cacert.pem
```

## CFK setup

### Create required secrets
    kubectl -n confluent create secret generic credential \
    --from-file=plain-users.json=$TUTORIAL_HOME/creds-kafka-sasl-users.json \
    --from-file=plain.txt=$TUTORIAL_HOME/creds-client-kafka-sasl-user.txt \
    --from-file=basic.txt=$TUTORIAL_HOME/creds-basic-users.txt

    kubectl -n confluent create secret generic rest-credential \
    --from-file=basic.txt=$TUTORIAL_HOME/rest-credential.txt

    kubectl -n confluent create secret generic password-encoder-secret \
    --from-file=password-encoder.txt=$TUTORIAL_HOME/passwordencoder.txt

### Deploy
Deploy source zookeeper, kafka cluster and topic `demo-topic`

    kubectl apply -f $TUTORIAL_HOME/confluent-platform.yaml

Create KafkaRestClass for ccloud

    kubectl apply -f $TUTORIAL_HOME/kafkarestclass-ccloud.yaml

After the Kafka cluster is in running state, create DNS entries

## Create source initiated clusterlink on CCLOUD
Once DNS entries are set, create cluster link between CFK cluster(source) and ccloud cluster(destination).
Cluster link will be created in the ccloud cluster.

    kubectl apply -f $TUTORIAL_HOME/source-initiated-link/clusterlink-ccloud-dst.yaml
    kubectl apply -f $TUTORIAL_HOME/source-initiated-link/clusterlink-ccloud-src.yaml

### Run cluster link test

#### Exec into source kafka pod
    kubectl -n confluent exec kafka-0 -it -- bash

#### Create kafka.properties
    cat <<EOF > /tmp/kafka.properties
    bootstrap.servers=kafka.confluent.svc.cluster.local:9071
    sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required username=kafka password=kafka-secret;
    security.protocol=SASL_PLAINTEXT
    sasl.mechanism=PLAIN
    EOF

#### Produce in source kafka cluster
    seq 20 | kafka-console-producer --topic demo-topic --bootstrap-server kafka.confluent.svc.cluster.local:9071 --producer.config /tmp/kafka.properties

#### Login to ccloud with confluent CLI.
This is described in `Setup CLI` section in ccloud

    confluent login --save
    confluent environment use <env name>
    confluent kafka cluster use <clusterId>

#### Consume message
Consume messages from `demo-topic`. This is the mirrored topic and this should have the message produced above
in the CFK cluster.

    confluent kafka topic consume -b demo-topic
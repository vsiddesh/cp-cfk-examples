# Confluent Platform Deployment on Kubernetes with External Kerberos Authentication Mode

This repository contains scenario workflows to deploy and manage Confluent on Kubernetes for Kerberos Authentication.

## Prerequisites

* Kerberos , principals, and keytab file.
* Helm 3 is installed on your local machine
* Kubectl is installed on your local machine
* A namespace created in the Kubernetes cluster - `confluent` 

## Installation Steps

### Step 1: Create Confluent Namespaces

```bash
kubectl create namespace confluent
```

### Step 2: Set the Default Namespace

```bash
kubectl config set-context --current --namespace confluent
```

### Step 3: Add the Confluent Helm Repository

```bash
helm repo add confluentinc https://packages.confluent.io/helm
helm repo update
```

### Step 4: Install Confluent for Kubernetes Operator

```bash
helm upgrade --install confluent-operator confluentinc/confluent-for-kubernetes \
  --namespace confluent
```

### Step 6: Verify Operator Installation

```bash
kubectl get pods -n confluent
```

### Step 7: Configure DNS for External Access

Once the LoadBalancer services are provisioned, map your DNS entries to the external IPs assigned to each Kafka broker and Control Center instance.

Example DNS mappings (replace `$DOMAIN` with your actual domain):

Assuming the following externalAccess configuration in the CR:

```bash
externalAccess:
  type: loadBalancer
  loadBalancer:
    domain: <your-domain>
```

Your DNS mappings should be:

```bash
kafka.<your-domain> : The EXTERNAL-IP value of kafka-bootstrap-lb service
broker-0.<your-domain> : The EXTERNAL-IP value of kafka-0-lb service
broker-1.<your-domain> : The EXTERNAL-IP value of kafka-1-lb service
broker-2.<your-domain> : The EXTERNAL-IP value of kafka-2-lb service
controlcenter.<your-domain> : The EXTERNAL-IP value of controlcenter-bootstrap-lb service
```

Ensure these DNS entries are resolvable by external clients

### Step 8: Create a Reverse DNS Zone 
Create PTR records
( This is essential for Client Authentication via Kerberos )
```bash
The EXTERNAL-IP value of kafka-bootstrap-lb service : kafka.<your-domain>
The EXTERNAL-IP value of kafka-0-lb service : broker-0.<your-domain> 
The EXTERNAL-IP value of kafka-1-lb service : broker-1.<your-domain>
The EXTERNAL-IP value of kafka-2-lb service : broker-2.<your-domain>
```
### Step 9: Validate External Connectivity

Verify DNS resolution:
```bash
nslookup kafka.<your-domain>
```
---
### Step 10:  Create Keytab secret

**Keytab for Kafka Brokers**
Note: Refer to kerberos-setup.readme for generating the keytab

```bash
kubectl create secret generic kafka-keytab-secret \
  --from-file=kafka-keytab=kafka.keytab \
  -n confluent
```

**Keytab for Control Center**
```bash
kubectl create secret generic c3-keytab-secret \
  --from-file=c3-keytab=c3.keytab \
  -n confluent
```

### Step 11: Create Configmaps 

**Create a shared configmap for kafka-jaas-configs**
```bash
kubectl create configmap kafka-jaas-configs \
  --from-file=broker-0-jaas.conf=./jaas/broker-0-jaas.conf \
  --from-file=broker-1-jaas.conf=./jaas/broker-1-jaas.conf \
  --from-file=broker-2-jaas.conf=./jaas/broker-2-jaas.conf \
  -n confluent
```

** Update kerberos_conf.yaml and apply the yaml to provision krb5.conf configmap.**

```bash
kubectl apply -f kerberos_conf.yaml
```

### Step 12: kubectl create configmap for pod overlays \

```bash
kubectl create configmap kafka-pod-overlay \
  --from-file=pod-template.yaml=extra-init-container.yaml \
  -n confluent
```

### Step 13: Deploy Confluent Platform Components

**Update all the placeholders in kraft.yaml**
- \<your-domain>

Apply your platform configuration for KRaft mode:
```bash
kubectl apply -f kraft.yaml
```

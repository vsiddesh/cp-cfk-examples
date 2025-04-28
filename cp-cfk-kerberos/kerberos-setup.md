# Setup Kerberos, Configure Principals and Keytab files

Kerberos Authentication Flow 
![Kerberos auth](https://github.com/user-attachments/assets/65fa4faf-6644-48e0-8381-e1580c773fd4)


## Prerequisites

* Virtual machine to install Kerberos KDC server.
* DNS service (AWS Hosted Zones/Azure DNS etc)

## Kerberos Installation Steps

### Step 1: Create Confluent Namespaces

```bash
Realm: EXAMPLE.COM

Primary KDC: kdc01.example.com

Secondary KDC: kdc02.example.com

User principal: ubuntu

Admin principal: ubuntu/admin
```

### Step 2: Install the Kerberos packages

You will be asked at the end of the install to supply the hostname for the Kerberos and Admin servers for the realm, which may or may not be the same server. 
Since we are going to create the realm, and thus these servers, type in the full hostname of this server.
```bash
sudo apt install krb5-kdc krb5-admin-server
```

### Step 3: Create the new realm with the kdb5_newrealm utility:

It will ask you for a database master password, which is used to encrypt the local database. Chose a secure password: its strength is not verified for you.

```bash
sudo krb5_newrealm
```

### Step 4: Configure the Kerberos server:

```bash
sudo dpkg-reconfigure krb5-kdc
```

### Step 6: Create Kerberos Principals & Generate Keytab Files: 

Create admin principal:
```bash
sudo kadmin.local
Authenticating as principal root/admin@EXAMPLE.COM with password.
kadmin.local: addprinc ubuntu
WARNING: no policy specified for ubuntu@EXAMPLE.COM; defaulting to no policy
Enter password for principal "ubuntu@EXAMPLE.COM": 
Re-enter password for principal "ubuntu@EXAMPLE.COM": 
Principal "ubuntu@EXAMPLE.COM" created.
kadmin.local: quit
```
Create component level principals 

broker-0:
```bash
kadmin.local -q "addprinc -randkey kafka/broker-0.example.com@example.com"
kadmin.local -q "ktadd -k /root/krb/kafka-broker-0.keytab -e aes256-cts-hmac-sha1-96:normal kafka/broker-0.example.com@example.com"
```
broker-1
```bash
kadmin.local -q "addprinc -randkey kafka/broker-1.example.com@example.com"
kadmin.local -q "ktadd -k /root/krb/kafka-broker-1.keytab -e aes256-cts-hmac-sha1-96:normal kafka/broker-1.example.com@example.com"
```
broker 2:
```bash
kadmin.local -q "addprinc -randkey kafka/broker-2.example.com@example.com"
kadmin.local -q "ktadd -k /root/krb/kafka-broker-2.keytab -e aes256-cts-hmac-sha1-96:normal kafka/broker-2.example.com@example.com"
```
c3:
```bash
kadmin.local -q "addprinc -randkey c3@example.com"
kadmin.local -q "ktadd -k /root/krb/producer.keytab -e aes256-cts-hmac-sha1-96:normal c3@example.com"
```
client:
```bash
kadmin.local -q "addprinc -randkey producer@example.com"
kadmin.local -q "ktadd -k /root/krb/producer.keytab -e aes256-cts-hmac-sha1-96:normal producer@example.com"
```
kadmin.local -q "addprinc -randkey kafka/broker-0.example.com@example.com"

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

# Setup Kerberos, Configure Principals and Keytab files

Kerberos Authentication Flow 
![Kerberos auth](https://github.com/user-attachments/assets/65fa4faf-6644-48e0-8381-e1580c773fd4)


## Prerequisites

* Virtual machine to install Kerberos KDC server.
* DNS service (AWS Hosted Zones/Azure DNS etc)

## Kerberos Installation Steps:

### Step 1: Add DNS A record for the KDC servers:

```bash
Realm: EXAMPLE.COM

Primary KDC: kdc01.example.com

Secondary KDC: kdc02.example.com

User principal: ubuntu

Admin principal: ubuntu/admin

kdc01.example.com 
```

### Step 2: Install the Kerberos packages

You will be asked at the end of the install to supply the hostname for the Kerberos and Admin servers for the realm, which may or may not be the same server. 
Since we are going to create the realm and these servers, type in the full hostname of this server.
```bash
sudo apt install krb5-kdc krb5-admin-server
```

### Step 3: Create the new realm with the kdb5_newrealm utility:

It will ask you for a database master password, which is used to encrypt the local database. Choose a secure password: its strength is not verified for you.

```bash
sudo krb5_newrealm
```

### Step 4: Configure the Kerberos server:

```bash
sudo dpkg-reconfigure krb5-kdc
```

### Step 6: Create Kerberos Principals & Generate Keytab Files: 

Create an admin principal:
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
Create component-level principals: 

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

### Step 7: Configure ACLS for all the principals:

The new admin principal needs to have the appropriate ACL permissions. 
The permissions are configured in the /etc/krb5kdc/kadm5.acl file:

```bash
kafka/broker-0.example.com@example.com *
kafka/broker-1.example.com@example.com *
kafka/broker-2.example.com@example.com *
c3@example.com *
producer@example.com *
```

### Step 8: Restart the krb5-admin-server for the new ACL to take effect:

```bash
sudo systemctl restart krb5-admin-server.service
```


### Step 9: Test created principals

The new user principal can be tested using the kinit utility:

```bash
kinit c3@example.com -kt /root/krb/c3.keytab 
```
After entering the password, use the klist utility to view information about the Ticket Granting Ticket (TGT):

```bash
klist
Ticket cache: FILE:/tmp/krb5cc_0
Default principal: c3@example.com

Valid starting     Expires            Service principal
04/28/25 03:48:47  04/28/25 13:48:47  krbtgt/example.com@example.com
	renew until 04/29/25 03:48:47
```


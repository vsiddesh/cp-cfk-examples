Deploy Confluent Platform + Flink on Azure AKS
======================================

To complete this scenario, you'll follow these steps:

#. Create an Azure resource group.

#. Deploy Azure AKS with system node pool.

#. Deploy all node pools for Confluent Platform components.

#. Deploy Confluent For Kubernetes Operator.

#. Deploy Confluent Platform.

#. Run the Producer & Consumer.

#. Deploy Flink Operators.

#. Deploy Flink Applications.

#. Tear down Confluent Platform.

==================================
Create an Azure resource group.
==================================

Create a new resource group in the JioAzureWest region and create an AKS cluster with the system Nodepool.

::
   
  az login
  az group create --name rg-cp-poc-aks --location jioindiawest

==================================
Deploy Azure AKS with System Nodepool.
==================================

Create an AKS cluster with the system Nodepool.

   ::
   
     az aks create \
     --resource-group rg-cp-poc-aks \
     --name aks-cp-poc  \
     --location jioindiawest \
     --node-count 2 \
     --node-vm-size Standard_D8ds_v5 \
     --generate-ssh-keys \
     --nodepool-name agentpool \
     --enable-cluster-autoscaler \
     --min-count 2 \
     --max-count 5 \
     --kubernetes-version 1.31.6 \
     --nodepool-taints CriticalAddonsOnly=true:NoSchedule \
     --no-wait \ 
     --enable-private-cluster 

==================================
Deploy all nodepools for Confluent Platform components.
==================================

A. Create CFK Operator node pool:

   ::
   
     az aks nodepool add \
     --resource-group rg-cp-poc-aks \
     --cluster-name aks-cp-poc  \
     --name cfkoperator \
     --node-count 1 \
     --node-vm-size Standard_D4as_v5 \
     --enable-cluster-autoscaler \
     --min-count 1 \
     --max-count 2 \
     --labels app-confluent=cfkoperator \
     --no-wait

B. Create Kraft node pool:

   ::
   
     az aks nodepool add \
     --resource-group rg-cp-poc-aks \
     --cluster-name aks-cp-poc  \
     --name kraft \
     --node-count 3 \
     --node-vm-size Standard_D8as_v5 \
     --enable-cluster-autoscaler \
     --min-count 3 \
     --max-count 4 \
     --labels app-confluent=kraft \
     --no-wait

C. Create Kafka Broker node pool::

   ::
   
     az aks nodepool add \
     --resource-group rg-cp-poc-aks \
     --cluster-name aks-cp-poc  \
     --name kafka \
     --node-count 3 \
     --node-vm-size Standard_E32bds_v5 \
     --enable-cluster-autoscaler \
     --min-count 3 \
     --max-count 4 \
     --labels app-confluent=kafka-broker \
     --no-wait

D. Create Schema Registry node pool:

   ::
   
     az aks nodepool add \
     --resource-group rg-cp-poc-aks \
     --cluster-name aks-cp-poc  \
     --name sr \
     --node-count 2 \
     --node-vm-size Standard_D4as_v5 \
     --enable-cluster-autoscaler \
     --min-count 2 \
     --max-count 3 \
     --labels app-confluent=sr \
     --no-wait

E. Create Connect node pool:

   ::
   
     az aks nodepool add \
     --resource-group rg-cp-poc-aks \
     --cluster-name aks-cp-poc  \
     --name connect \
     --node-count 2 \
     --node-vm-size Standard_D8as_v5 \
     --enable-cluster-autoscaler \
     --min-count 2 \
     --max-count 3 \
     --labels app-confluent=connect \
     --no-wait

F. Create Control Center node pool:

   ::
   
     az aks nodepool add \
     --resource-group rg-cp-poc-aks \
     --cluster-name aks-cp-poc  \
     --name c3 \
     --node-count 1 \
     --node-vm-size Standard_E16as_v5 \
     --enable-cluster-autoscaler \
     --min-count 1 \
     --max-count 2 \
     --labels app-confluent=c3 \
     --no-wait

G. Flink Kubernetes Operator node pool:

   ::
   
     az aks nodepool add \
     --resource-group rg-cp-poc-aks \
     --cluster-name aks-cp-poc  \
     --name flinkop \
     --node-count 1 \
     --node-vm-size Standard_D4as_v5 \
     --enable-cluster-autoscaler \
     --min-count 1 \
     --max-count 2 \
     --labels app-confluent=flinkoperator \
     --no-wait

H. Confluent Manager for Apache Flink Operator node pool:

   ::
   
     az aks nodepool add \
     --resource-group rg-cp-poc-aks \
     --cluster-name aks-cp-poc  \
     --name cmfoperator \
     --node-count 1 \
     --node-vm-size Standard_D4as_v5 \
     --enable-cluster-autoscaler \
     --min-count 1 \
     --max-count 2 \
     --labels app-confluent=cmfoperator \
     --no-wait

9. Flink Task manager node pool: 

   ::
   
     az aks nodepool add \
     --resource-group rg-cp-poc-aks \
     --cluster-name aks-cp-poc  \
     --name taskmanager \
     --node-count 4 \
     --node-vm-size Standard_E32bds_v5 \
     --node-osdisk-type Ephemeral \
     --enable-cluster-autoscaler \
     --min-count 4 \
     --max-count 5 \
     --labels app-confluent=taskmanager \
     --no-wait

J. Create Flink Job Manager node pool:

   ::
   NOTE: NOT REQUIRED, DELETE THIS NODEPOOL.
     az aks nodepool add \
     --resource-group rg-cp-poc-aks \
     --cluster-name aks-cp-poc  \
     --name jobmanager \
     --node-count 2 \
     --node-vm-size Standard_E16bds_v5 \
     --node-osdisk-type Ephemeral \
     --enable-cluster-autoscaler \
     --min-count 2 \
     --max-count 3 \
     --labels app-confluent=jobmanager \
     --no-wait

========================================
Deploy Confluent for Kubernetes Operator
========================================

#. Create the namespace to use.

   :: 
   
      kubectl create namespace confluent

#. Set this namespace to default for your Kubernetes context.

   :: 
   
      kubectl config set-context --current --namespace confluent

#. Set up the Helm Chart:

   ::

     helm repo add confluentinc https://packages.confluent.io/helm


#. Install Confluent For Kubernetes using Helm:

   ::

     helm upgrade --install confluent-operator confluentinc/confluent-for-kubernetes --namespace confluent -f values.yaml --set enableCMFDay2Ops=true
     
     NOTE: values.yaml file contains labels for operator pod for scheduling pods to particular nodes.
  
#. Check that the Confluent For Kubernetes pod comes up and is running:

   ::
     
     kubectl get pods

========================================
Review Confluent Platform configurations
========================================

You install Confluent Platform components as custom resources (CRs). 

You can configure all Confluent Platform components as custom resources. In this
tutorial, you will configure all components in a single file and deploy all
components with one ``kubectl apply`` command.

The entire Confluent Platform is configured in one configuration file:
``confluent-platform.yaml``

In this configuration file, there is a custom Resource configuration spec for
each Confluent Platform component - replicas, image to use, resource
allocations.
  
=========================
Deploy Confluent Platform
=========================

#. Replace Kubernetes Node host/ domain in Confluent Platform 
   ::

     Replace "<NODEIP/HOST>" with the node's k8s host domain / ip address.

#. Deploy Storage Class for automatically storage provisioning:

   ::

     kubectl apply -f storage-class.yaml

#. Deploy Confluent Platform with the above configuration:

   ::

     kubectl apply -f confluent-platform.yaml

#. Check that all Confluent Platform resources are deployed:

   ::
   
     kubectl get pods

========
Validate
========
Create an Azure VM to validate producers and consumers.

Run producer application CLI on Azure VM
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Now that we've got the infrastructure set up, let's deploy the producer client.

#. Install Confluent CLI utility to create topics and test the production & consumption of messages. 

   ::

     curl -O https://packages.confluent.io/archive/7.9/confluent-7.9.0.tar.gz

     tar xzf confluent-7.9.0.tar.gz

     export CONFLUENT_HOME=~/confluent-7.9.0

     export PATH=$PATH:$CONFLUENT_HOME/bin

     sudo apt install default-jre

      
#. Create topics using CLI:

   ::
   
      kafka-topics --create --bootstrap-server <NODEIP/HOST>:30000  --topic test-topic

#. Produce using kafka-console-cli:

   ::
   
      kafka-console-producer --bootstrap-server <NODEIP/HOST>:30000 --topic test-topic

#. Consume using kafka-console-cli:

   ::
   
      kafka-console-consumer --bootstrap-server <NODEIP/HOST>:30000 --topic test-topic --from-beginning

Validate in Control Center
^^^^^^^^^^^^^^^^^^^^^^^^^^

Use Control Center to monitor the Confluent Platform, and see the created topic and data.

#. Set up port forwarding to Control Center web UI from local machine:

   ::

     kubectl port-forward controlcenter-0 9021:9021

#. Browse to Control Center:

   ::
   
     http://localhost:9021

#. Check that the ``test-topic`` topic was created and that messages are being produced to the topic.

=======================
Deploy Flink Operators.
=======================

#. Create a namespace or use an existing namespace:

   ::

     kubectl create namespace flink

#. Install the cert-manager

   ::

     kubectl create -f https://github.com/jetstack/cert-manager/releases/download/v1.8.2/cert-manager.yaml
  
#. Install Flink K8s operator:

   ::

     helm upgrade --install cp-flink-kubernetes-operator confluentinc/flink-kubernetes-operator -n confluent -f flink-operator-values.yaml
  
#. Install Confluent Manager for Apache Flink K8s operator:

   ::

     helm upgrade --install cmf confluentinc/confluent-manager-for-apache-flink --namespace confluent

#. Run kubectl command to check all pods are running - 3 cert-manager pods and two pods for CMF and operator:

   ::

     kubectl get pods -n confluent

=========================
Deploy Flink Applications
=========================

#. Deploy confluent manager for apache flink rest class:

   ::

     kubectl apply -f cmfrestclass.yaml

#. Create the environment:

   ::

     kubectl apply -f flink-env.yaml

#. Create a new Azure storage account and provision a container on it. Update below blob properties in flink-app.yaml:
   
   ::

     state.checkpoints.dir: wasbs://<container>@<storage-account>.blob.core.windows.net/checkpoint/
     fs.azure.account.key.<storage-account>.blob.core.windows.net: <azure-access-key>

#. Deploy the Flink application:

   ::

     kubectl apply -f flink-app.yaml

#. Flink Application Port forwarding for accessing the Flink application UI:

   ::
     
     kubectl port-forward svc/<service_name> 8081:8081 -n flink
     kubectl port-forward svc/flink-app-rest 8081:8081 -n flink


=========
Tear Down
=========

#. Delete Flink components:

   ::

     kubectl delete -f flink-app.yaml
     kubectl delete -f flink-env.yaml
     kubectl delete -f cmfrestclass.yaml
  
     helm delete  cmf -n confluent
     helm delete cp-flink-kubernetes-operator -n confluent
  

#. Delete Confluent Platform components:

   ::

     kubectl delete -f confluent-platform.yaml
     kubectl delete -f storage-class.yaml

     helm uninstall confluent-operator -n confluent
  

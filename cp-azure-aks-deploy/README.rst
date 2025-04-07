Deploy Confluent Platform
=========================

To complete this scenario, you'll follow these steps:

#. Create an Azure resource group.

#. Deploy Azure AKS with System Nodepool.

#. Deploy all nodepools for Confluent Platform components.

#. Deploy the Producer application.

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
  --resource-group rg-cp-poc-aks\
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
  --no-wait

==================================
Deploy all nodepools for Confluent Platform components.
==================================

A. Create CFK Operator node pool:

::
   
  az aks nodepool add \
  --resource-group rg-jio-analytics-poc-sid\
  --cluster-name cli-aks-jio-v2  \
  --name cfkoperator \
  --node-count 1 \
  --node-vm-size Standard_D4as_v5 \
  --enable-cluster-autoscaler \
  --min-count 1 \
  --max-count 2 \
  --labels app-confluent=cfkoperator \
  --no-wait

A. Create Kraft node pool:

::
   
  az aks nodepool add \
  --resource-group rg-jio-analytics-poc-sid\
  --cluster-name cli-aks-jio-v2  \
  --name kraft \
  --node-count 3 \
  --node-vm-size Standard_D8as_v5 \
  --enable-cluster-autoscaler \
  --min-count 3 \
  --max-count 4 \
  --labels app-confluent=kraft \
  --no-wait



==================================
Create an Azure resource group.
==================================

Create a new resource group in the JioAzureWest region and create an AKS cluster with the system Nodepool.

::
   
  az login
  az group create --name rg-cp-poc-aks --location jioindiawest

==================================
Create an Azure resource group.
==================================

Create a new resource group in the JioAzureWest region and create an AKS cluster with the system Nodepool.

::
   
  az login
  az group create --name rg-cp-poc-aks --location jioindiawest

===============================
Deploy Confluent for Kubernetes
===============================

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

     helm upgrade --install confluent-operator confluentinc/confluent-for-kubernetes --namespace confluent
  
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
``$TUTORIAL_HOME/confluent-platform.yaml``

In this configuration file, there is a custom Resource configuration spec for
each Confluent Platform component - replicas, image to use, resource
allocations.

For example, the Kafka section of the file is as follows:

::
  
  ---
  apiVersion: platform.confluent.io/v1beta1
  kind: Kafka
  metadata:
    name: kafka
    namespace: confluent
  spec:
    replicas: 3
    image:
      application: confluentinc/cp-server:7.9.0
      init: confluentinc/confluent-init-container:2.11.0
    dataVolumeCapacity: 10Gi
    metricReporter:
      enabled: true
    dependencies:
      zookeeper:
        endpoint: zookeeper.confluent.svc.cluster.local:2181
  ---
  
=========================
Deploy Confluent Platform
=========================

#. Deploy Confluent Platform with the above configuration:

   ::

     kubectl apply -f $TUTORIAL_HOME/confluent-platform.yaml

   Note: If you are deploying a single node dev cluster, then use this yaml file:

   ::

     kubectl apply -f $TUTORIAL_HOME/confluent-platform-singlenode.yaml
     

#. Check that all Confluent Platform resources are deployed:

   ::
   
     kubectl get confluent

#. Get the status of any component. For example, to check Kafka:

   ::
   
     kubectl describe kafka

========
Validate
========

Deploy producer application
^^^^^^^^^^^^^^^^^^^^^^^^^^^

Now that we've got the infrastructure set up, let's deploy the producer client
app.

The producer app is packaged and deployed as a pod on Kubernetes. The required
topic is defined as a KafkaTopic custom resource in
``$TUTORIAL_HOME/producer-app-data.yaml``.

The ``$TUTORIAL_HOME/producer-app-data.yaml`` defines the ``elastic-0``
topic as follows:

::

  apiVersion: platform.confluent.io/v1beta1
  kind: KafkaTopic
  metadata:
    name: elastic-0
    namespace: confluent
  spec:
    replicas: 3 # change to 1 if using single node
    partitionCount: 1
    configs:
      cleanup.policy: "delete"
      
Deploy the producer app:

::
   
   kubectl apply -f $TUTORIAL_HOME/producer-app-data.yaml

Note: If you are deploying a single node dev cluster, then use this yaml file:

::
  
  kubectl apply -f $TUTORIAL_HOME/producer-app-data-singlenode.yaml

Validate in Control Center
^^^^^^^^^^^^^^^^^^^^^^^^^^

Use Control Center to monitor the Confluent Platform, and see the created topic and data.

#. Set up port forwarding to Control Center web UI from local machine:

   ::

     kubectl port-forward controlcenter-0 9021:9021

#. Browse to Control Center:

   ::
   
     http://localhost:9021

#. Check that the ``elastic-0`` topic was created and that messages are being produced to the topic.

=========
Tear Down
=========

Shut down Confluent Platform and the data:

::

  kubectl delete -f $TUTORIAL_HOME/producer-app-data.yaml

::

  kubectl delete -f $TUTORIAL_HOME/confluent-platform.yaml

::

  helm uninstall confluent-operator
  

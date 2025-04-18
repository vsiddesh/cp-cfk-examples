Deploy Confluent Platform
=========================

* Quickly set up the complete Confluent Platform on the Kubernetes.
* Configure a producer to generate sample data.

To complete this scenario, you'll follow these steps:

#. Deploy Confluent For Kubernetes.

#. Deploy Confluent Platform.

#. Deploy connectors.

#. Validate deployment.

#. Tear down Confluent Platform.

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
``cp-kraft.yaml``

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

     kubectl apply -f cp-kraft.yaml
     

#. Check that all Confluent Platform resources are deployed:

   ::
   
     kubectl get pods

#. Deploy required connectors:

   ::
   
     kubectl apply -f connector-pg.yaml

========
Validate
========

Validate in Control Center
^^^^^^^^^^^^^^^^^^^^^^^^^^

Use Control Center to monitor the Confluent Platform, and see the created topic and data.

#. Set up port forwarding to Control Center web UI from local machine:

   ::

     kubectl port-forward controlcenter-0 9021:9021

#. Browse to Control Center:

   ::
   
     http://localhost:9021

=========
Tear Down
=========

Shut down Confluent Platform and the data:

::

  kubectl delete -f connector-pg.yaml

::

  kubectl delete -f cp-kraft.yaml

::

  helm uninstall confluent-operator
  

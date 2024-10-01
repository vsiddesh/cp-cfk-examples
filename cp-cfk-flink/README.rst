Deploy Confluent Platform Flink 
===============================


==============================================
Deploy Confluent for Kubernetes Flink Operator
==============================================

#. Create the namespace to use.

   :: 
   
      kubectl create namespace flink

#. Set this namespace to default for your Kubernetes context.

   :: 
   
      kubectl config set-context --current --namespace flink

#. Set up the Helm Chart:

   ::

     helm repo add confluentinc https://packages.confluent.io/helm

#. Install the certificate manager.

   :: 
   
      kubectl create -f https://github.com/jetstack/cert-manager/releases/download/v1.8.2/cert-manager.yaml

#. Install Confluent For Apache Flink using Helm:

   ::

     helm upgrade --install cp-flink-kubernetes-operator confluentinc/flink-kubernetes-operator

#. Install Confluent For Apache Flink using Helm:

   ::

     helm upgrade --install cp-flink-kubernetes-operator confluentinc/flink-kubernetes-operator

   Note: If you are deploying a privately hosted flink operator docker image use imagePullSecret params. First create the secret in the namespace.

   ::
    
     kubectl create secret docker-registry imgsecret \
      --docker-server=https://index.docker.io/v1/ --docker-username=<uname> \
      --docker-password=<pass> --docker-email=<email> \
      --namespace flink

     helm upgrade --install cp-flink-kubernetes-operator confluentinc/flink-kubernetes-operator -n flink \
     --set "image.repository=siddeshvcflt/cp-flink2" --set "image.tag=1.8.0-cp1-operator" --set "imagePullSecrets={imgsecret}"
  
#. Check that the Confluent For Apache Flink pod comes up and is running:

   ::
     
     kubectl get pods

========================================
Deploy Flink Jobs
========================================

You install  Confluent Platform for Apache Flink component as custom resource (CRs). 

You can configure all Confluent Platform Apache Flink components as custom resource. In this
tutorial, you will configure all components in a single file and deploy all
components with one ``kubectl apply`` command.

For example, the FlinkDeployment section of the file is as follows:

::
  
  ---
  apiVersion: flink.apache.org/v1beta1
  kind: FlinkDeployment
  metadata:
    name: basic-example
    namespace: flink
  spec:
    image: confluentinc/cp-flink:1.19.1-cp1
    flinkVersion: v1_19
    flinkConfiguration:
      taskmanager.numberOfTaskSlots: "2"
    serviceAccount: flink
    jobManager:
      resource:
        memory: "2048m"
        cpu: 1
    taskManager:
      resource:
        memory: "2048m"
        cpu: 1
    job:
      jarURI: local:///opt/flink/examples/streaming/StateMachineExample.jar
      parallelism: 2
      upgradeMode: stateless
  ---
  
============================================
Deploy Apache Flink Jobs
============================================

#. Deploy Confluent Platform Apache Flink with the above configuration:

   ::

     kubectl apply -f flink-basic.yaml

#. Get the status of any component. For example, to check Kafka:

   ::
   
     kubectl describe flinkdeployment

==================================================================
Deploy Apache Flink Jobs for Privately Hosted Flink docker Images
==================================================================

#. Create docker-regsitry secret for authentication to private docker repository.
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Note: Skip this step if secret is already created while installing flink operator.
::

  
  kubectl create secret docker-registry imgsecret \
   --docker-server=https://index.docker.io/v1/ --docker-username=<uname> \
   --docker-password=<pass> --docker-email=<email> \
   --namespace flink

#. Deploy Confluent Platform Apache Flink with the above configuration:

   ::

     kubectl apply -f private-image-flink-basic.yaml

#. Get the status of any component. For example, to check Kafka:

   ::
   
     kubectl describe flinkdeployment

========
Validate
========


Validate in Flink Console
^^^^^^^^^^^^^^^^^^^^^^^^^^

Use Flink Console to monitor the Flink jobs.

#. Get the endpoints with the following command:

   ::

     kubectl get endpoints
     
     NAME                             ENDPOINTS                                 AGE
     
     basic-example                    192.168.228.91:6124,192.168.228.91:6123   8m22s
     basic-example-rest               192.168.228.91:8081                       8m22s
     flink-operator-webhook-service   192.168.179.202:9443                      12m
     kubernetes                       192.168.238.57:443,192.168.29.233:443     16d


#. Find the REST endpoint, and set up port forwarding with a command like the following:

   ::

     kubectl port-forward service/basic-example-rest -n flink 8081:8081

#. Browse to Flink console.

   ::
   
     http://localhost:8081


=========
Tear Down
=========

Shut down Confluent Platform Apache Flink jobs, cert-manager and the data:

::

  kubectl delete -f flink-basic.yaml

::

  kubectl delete -f https://github.com/jetstack/cert-manager/releases/download/v1.8.2/cert-manager.yaml

::

  helm uninstall cp-flink-kubernetes-operator
  

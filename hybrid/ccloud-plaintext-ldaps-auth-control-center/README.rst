Deploy Confluent Platform
=========================

In this workflow scenario, you'll set up Control Center with LDAPs (ldap over TLS) based authentication to monitor and connect to a Confluent Cloud Platform and local Connect CR.  
The goal for this scenario is for you to:

* Configure LDAP with TLS server for Control Center authentication (no RBAC)
* Configure Connect worker with Confluent Cloud
* Configure Control Center with ldaps to monitor Confluent Cloud and local Connect worker


To complete this scenario, you'll follow these steps:

#. Set the current tutorial directory.

#. Deploy Confluent For Kubernetes.

#. Deploy LDAPS server

#. Deploy Connect and Control Center.

#. Tear down Confluent Platform.

==================================
Set the current tutorial directory
==================================

Set the tutorial directory for this tutorial under the directory you downloaded
the tutorial files:

::
   
  export TUTORIAL_HOME=<Tutorial directory>/hybrid/ccloud-plaintext-ldaps-auth-control-center

===============================
Deploy Confluent for Kubernetes
===============================

#. Set up the Helm Chart:

   ::

     helm repo add confluentinc https://packages.confluent.io/helm


#. Install Confluent For Kubernetes using Helm:

   ::

     helm upgrade --install operator confluentinc/confluent-for-kubernetes --namespace=confluent
  
#. Check that the Confluent For Kubernetes pod comes up and is running:

   ::
     
     kubectl get pods --namespace=confluent

===============
Deploy OpenLDAP
===============

This repo includes a Helm chart for `OpenLdap
<https://github.com/osixia/docker-openldap>`__. The chart ``values.yaml``
includes the set of principal definitions that Confluent Platform needs for
RBAC.

#. Deploy OpenLdap

   ::

     helm upgrade --install -f $TUTORIAL_HOME/ldapswithccloudcerts/ldaps.yaml test-ldap $TUTORIAL_HOME/../../assets/openldap --namespace confluent

Note that it is assumed that your Kubernetes cluster has a ``confluent`` namespace available, otherwise you can create it by running ``kubectl create namespace confluent``. 

#. Validate that OpenLDAP is running:  
   
   ::

     kubectl get pods --namespace=confluent

#. Log in to the LDAP pod:

   ::

     kubectl --namespace=confluent exec -it ldap-0 -- bash

#. Run the LDAP search command:

   ::

     ldapsearch -LLL -x -H ldaps://ldap.confluent.svc.cluster.local:636 -b 'dc=test,dc=com' -D "cn=mds,dc=test,dc=com" -w 'Developer!'

#. Exit out of the LDAP pod:

   ::
   
     exit 

#. Create secret for Control Center to use 

   ::
   
    kubectl create secret generic ldaps-tls-ccloud \
    --from-file=truststore.jks=$TUTORIAL_HOME/ldapswithccloudcerts/truststore.jks \
    --from-file=jksPassword.txt=$TUTORIAL_HOME/ldapswithccloudcerts/jksPassword.txt \
    --namespace confluent

#. Create Control Center `bindDn` and `bindPassword`

  ::

    kubectl create secret generic ldaps-user \
    --from-file=ldap.txt=$TUTORIAL_HOME/ldapswithccloudcerts/ldapbinds.txt \
    --namespace confluent

Provide authentication credentials
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Confluent Cloud provides you an API key for both Kafka.
Configure Confluent For Kubernetes to use the API.

Create a Kubernetes secret object for Confluent Cloud Kafka access.
This secret object contains file based properties. These files are in the format that each respective Confluent component requires for authentication credentials. 

Edit the file `creds-client-kafka-sasl-user.txt` to hold the <api-key> / <api-secret>.  

::

  kubectl create secret generic cloud-plain \
  --from-file=plain.txt=$TUTORIAL_HOME/creds-client-kafka-sasl-user.txt

=========================
Deploy Confluent Platform
=========================

Replace `<cloudKafka_url>:9092` with your cluster bootstrap in the `confluent-platform-ccc-ldaps.yaml` file.  

#. Deploy Confluent Platform with the above configuration:

::

  kubectl apply -f $TUTORIAL_HOME/confluent-platform-ccc-ldaps.yaml --namespace=confluent

#. Check that all Confluent Platform resources are deployed:

   ::
   
     kubectl get confluent --namespace=confluent

#. Get the status of any component. For example, to check Kafka:

   ::
   
     kubectl describe kafka --namespace=confluent

========
Validate
========

Validate in Control Center
^^^^^^^^^^^^^^^^^^^^^^^^^^

Use Control Center to monitor the Confluent Platform, and see the created topic and data.

#. Set up port forwarding to Control Center web UI from local machine:

   ::

     kubectl port-forward controlcenter-0 9021:9021 --namespace=confluent

#. Browse to Control Center:

   ::
   
     http://localhost:9021


#. Users: 

    Full Control: Username:james Password:james-secret  

    Restricted Control: Username:alice Password:alice-secret
    
    

=========
Tear Down
=========

Shut down Confluent Platform and the data:

::

  kubectl delete -f $TUTORIAL_HOME/confluent-platform-ccc-ldaps.yaml --namespace=confluent

::

  helm delete operator --namespace=confluent

::

  helm delete test-ldap --namespace=confluent

::

  kubectl delete pvc ldap-config-ldap-0 --namespace=confluent

::

  kubectl delete pvc ldap-data-ldap-0 --namespace=confluent

::

  kubectl delete secret ldaps-tls-ccloud --namespace=confluent

::

  kubectl delete secret ldaps-user --namespace=confluent


::

  kubectl delete secret cloud-plain --namespace=confluent

===============
Troubleshooting
===============

:: 

  openssl s_client -connect ldap.confluent.svc.cluster.local:636
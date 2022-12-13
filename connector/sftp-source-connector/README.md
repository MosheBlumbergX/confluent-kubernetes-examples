```
Deploy Confluent Platform
=========================

In this workflow scenario, you'll set up a simple non-secure (no authn, authz or
encryption) Confluent Platform, consisting of all components.

The goal for this scenario is for you to:

* Quickly set up the complete Confluent Platform on the Kubernetes.
* Configure a producer to generate sample data.

Watch the walkthrough: `Quickstart Demonstration <https://youtu.be/qepFNPhrL08>`_

Before continuing with the scenario, ensure that you have set up the
`prerequisites </README.md#prerequisites>`_.

To complete this scenario, you'll follow these steps:

#. Set the current tutorial directory.

#. Deploy Confluent For Kubernetes.

#. Deploy Confluent Platform.

#. Deploy the Producer application.

#. Tear down Confluent Platform.

==================================
Set the current tutorial directory
==================================

Set the tutorial directory for this tutorial under the directory you downloaded
the tutorial files:

::
   
  export TUTORIAL_HOME=<Tutorial directory>/connector/sftp-source-connector

===============================
Deploy Confluent for Kubernetes
===============================

#. Set up the Helm Chart:

   ::

     helm repo add confluentinc https://packages.confluent.io/helm


#. Install Confluent For Kubernetes using Helm:

   ::

     helm upgrade --install confluent-operator confluentinc/confluent-for-kubernetes --namespace confluent
  
#. Check that the Confluent For Kubernetes pod comes up and is running:

   ::
     
     kubectl get pods


kubectl apply -f $TUTORIAL_HOME/sftp-server-deployment.yaml

% g pods
NAME                                  READY   STATUS     RESTARTS   AGE
confluent-operator-7ccdd57888-k9jcv   1/1     Running    0          59m
connect-0                             0/1     Init:0/1   0          19s
kafka-0                               1/1     Running    0          54m
kafka-1                               1/1     Running    0          54m
kafka-2                               0/1     Running    0          8s
sftp-server-79cf99d8f5-6fz9g          1/1     Running    0          3m15s
zookeeper-0                           1/1     Running    0          56m
zookeeper-1                           1/1     Running    0          56m
zookeeper-2                           1/1     Running    0          56m
%  kubectl --namespace=confluent exec -it sftp-server-79cf99d8f5-6fz9g  -- mkdir -p /chroot/home/foo/upload/input                
%  kubectl --namespace=confluent exec -it sftp-server-79cf99d8f5-6fz9g  -- mkdir -p /chroot/home/foo/upload/error
%  kubectl --namespace=confluent exec -it sftp-server-79cf99d8f5-6fz9g  -- mkdir -p /chroot/home/foo/upload/finished
%  kubectl --namespace=confluent exec -it sftp-server-79cf99d8f5-6fz9g  -- chown -R foo /chroot/home/foo/upload     


mosheblumberg@C02F4360MD6T sftp %      kubectl --namespace=confluent exec -it sftp-server-79cf99d8f5-6fz9g   -- bash

root@sftp-server:/# 
root@sftp-server:/# 
root@sftp-server:/# 
root@sftp-server:/# echo $'id,first_name,last_name,email,gender,ip_address,last_login,account_balance,country,favorite_color\n1,Salmon,Baitman,sbaitman0@feedburner.com,Male,120.181.75.98,2015-03-01T06:01:15Z,17462.66,IT,#f09bc0\n2,Debby,Brea,dbrea1@icio.us,Female,153.239.187.49,2018-10-21T12:27:12Z,14693.49,CZ,#73893a' > /chroot/home/foo/upload/input/csv-sftp-source.csv
root@sftp-server:/# cat /chroot/home/foo/upload/input/csv-sftp-source.csv
id,first_name,last_name,email,gender,ip_address,last_login,account_balance,country,favorite_color
1,Salmon,Baitman,sbaitman0@feedburner.com,Male,120.181.75.98,2015-03-01T06:01:15Z,17462.66,IT,#f09bc0
2,Debby,Brea,dbrea1@icio.us,Female,153.239.187.49,2018-10-21T12:27:12Z,14693.49,CZ,#73893a

% kubectl --namespace=confluent exec -it kafka-0 -- bash                     
Defaulted container "kafka" out of: kafka, config-init-container (init)
bash-4.4$ kafka-topics --bootstrap-server localhost:9071 --create --partitions 3 --replication-factor 3  --topic sftp-testing-topic 
exit

% kubectl --namespace=confluent exec -it connect-0   -- bash   

 curl -X PUT \
     -H "Content-Type: application/json" \
     --data '{
        "topics": "test_sftp_sink",
               "tasks.max": "1",
               "connector.class": "io.confluent.connect.sftp.SftpCsvSourceConnector",
               "cleanup.policy":"MOVE",
               "behavior.on.error":"IGNORE",
               "input.path": "/home/foo/upload/input",
               "error.path": "/home/foo/upload/error",
               "finished.path": "/home/foo/upload/finished",
               "input.file.pattern": ".*\\.csv",
               "sftp.username":"foo",
               "sftp.password":"pass",
               "sftp.host":"sftp-server",
               "sftp.port":"2222",
               "kafka.topic": "sftp-testing-topic",
               "csv.first.row.as.header": "true",
               "schema.generation.enabled": "true"
          }' \
     http://localhost:8083/connectors/sftp-source-csv/config 


$ curl http://localhost:8083/connectors

$ curl http://localhost:8083/connectors/sftp-source-csv/status
{"name":"sftp-source-csv","connector":{"state":"RUNNING","worker_id":"connect-0.connect.confluent.svc.cluster.local:8083"},"tasks":[{"id":0,"state":"RUNNING","worker_id":"connect-0.connect.confluent.svc.cluster.local:8083"}],"type":"source"}bash-4.4$ 

#curl -X DELETE http://localhost:8083/connectors/sftp-source-csv/


% kubectl --namespace=confluent exec -it kafka-0 -- bash                     
 kafka-console-consumer --from-beginning -bootstrap-server localhost:9071 --topic sftp-testing-topic
```


To add more data if needed:  

in the ftp pod:  
```
root@sftp-server:/# for i in {1..4505}; do echo $i,Debby,Brea,dbrea1@icio.us,Female,153.239.187.49,2018-10-21T12:27:12Z,14693.49,CZ,#73893a >> /chroot/home/foo/upload/input/csv-sftp-source.csv; done
```
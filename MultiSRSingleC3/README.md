# Multiple Kafka cluster and Schema with Single C3

In this setup we will set 5 clusters with a single C3 view 

## Set up Pre-requisites

Set the tutorial directory for this tutorial under the directory you downloaded
the tutorial files:

```
export TUTORIAL_HOME=<Tutorial directory>/MultiSRSingleC3
```

Create 5 namespaces, where the destination cluster is the C3 backing cluster:

```
kubectl create ns destination     
kubectl create ns dev              
kubectl create ns stage           
kubectl create ns test            
kubectl create ns xyz
```

Deploy Confluent for Kubernetes (CFK) in cluster mode, so that the one CFK instance can manage Confluent deployments in multiple namespaces. Here, CFk is deployed to the `default` namespace.

```
helm upgrade --install confluent-operator \
  confluentinc/confluent-for-kubernetes \
  --namespace default \
  --set namespaced=false
```

Confluent For Kubernetes provides auto-generated certificates for Confluent Platform components to use for inter-component TLS. You'll need to generate and provide a Root Certificate Authority (CA).

Generate a CA pair to use in this tutorial:

```
openssl genrsa -out $TUTORIAL_HOME/ca-key.pem 2048
openssl req -new -key $TUTORIAL_HOME/ca-key.pem -x509 \
  -days 1000 \
  -out $TUTORIAL_HOME/ca.pem \
  -subj "/C=US/ST=CA/L=MountainView/O=Confluent/OU=Operator/CN=TestCA"
```

Then, provide the certificate authority as a Kubernetes secret `ca-pair-sslcerts` to be used to 
generate the auto-generated certs, in both the source and destination namespaces:

```
nslist=(destination     
dev              
stage           
test            
xyz )

for i in $nslist; do kubectl create secret tls ca-pair-sslcerts \
  --cert=$TUTORIAL_HOME/ca.pem \
  --key=$TUTORIAL_HOME/ca-key.pem \
  -n $i; done
```

## Create credentials secrets  

In this step you will be creating secrets to be used to authenticate the clusters.  

```
nslist=(destination     
dev              
stage           
test            
xyz )

for i in $nslist; do kubectl create secret generic credential -n $i \
--from-file=plain-users.json=$TUTORIAL_HOME/creds-kafka-sasl-users.json \
--from-file=plain.txt=$TUTORIAL_HOME/creds-client-kafka-sasl-user.txt \
--from-file=basic.txt=$TUTORIAL_HOME/creds-client-kafka-sasl-user.txt ; done
```

## Deploy the clusters 

Deploy the source and destination cluster.  
```
kubectl apply -f $TUTORIAL_HOME/components-dev.yaml
kubectl apply -f $TUTORIAL_HOME/components-stage.yaml
kubectl apply -f $TUTORIAL_HOME/components-test.yaml
kubectl apply -f $TUTORIAL_HOME/components-xyz.yaml


kubectl apply -f $TUTORIAL_HOME/components-destination.yaml
kubectl apply -f $TUTORIAL_HOME/controlcenter.yaml
```

Check pods: 

```
kubectl get  pods --all-namespaces -l confluent-platform=true 
```

If for some reason the `schemaregistry-0` pod in any of the namespaces is in a `CrashLoopBackOff` check the log, it could be that internal topic was created before all brokers were ready, the log will indicate this:  

```
k logs -n destination  schemaregistry-0
```

```
Caused by: java.util.concurrent.ExecutionException: org.apache.kafka.common.errors.NotEnoughReplicasException: Messages are rejected since there are fewer in-sync replicas than required.
```

You can fix it by opening a bash to the kafka pod, delete the schema topic and then delete the schema pod, this will force it to create the topic again. 

```
kubectl -n destination  exec -it  kafka-0  -- bash
bash-4.4$ cat <<EOF > /tmp/kafka.properties                                                                             
bootstrap.servers=kafka.destination.svc.cluster.local:9071
sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required username=kafka password=kafka-secret;
sasl.mechanism=PLAIN
security.protocol=SASL_SSL
ssl.truststore.location=/mnt/sslcerts/truststore.jks
ssl.truststore.password=mystorepassword
EOF

bash-4.4$ kafka-topics --bootstrap-server kafka.destination.svc.cluster.local:9071 --command-config /tmp/kafka.properties --delete --topic _schemas_schemaregistry_destination 

exit

kubectl -n destination delete pod schemaregistry-0
```

## Validate that it works

### If you want to produce data to topic in on od the clusters cluster

Create the kafka.properties file in any of the brokers:
Replace `$cluster` with the namespace (destination,dev,stage,test,xyz): 

```
cat <<EOF > /tmp/kafka.properties
bootstrap.servers=kafka.$cluster.svc.cluster.local:9071
sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required username=kafka password=kafka-secret;
sasl.mechanism=PLAIN
security.protocol=SASL_SSL
ssl.truststore.location=/mnt/sslcerts/truststore.jks
ssl.truststore.password=mystorepassword
EOF
```


For example:   
```
kubectl -n xyz exec -it  kafka-0   -- bash

Defaulted container "kafka" out of: kafka, config-init-container (init)
bash-4.4$ cat <<EOF > /tmp/kafka.properties
> bootstrap.servers=kafka.xyz.svc.cluster.local:9071
> sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required username=kafka password=kafka-secret;
> sasl.mechanism=PLAIN
> security.protocol=SASL_SSL
> ssl.truststore.location=/mnt/sslcerts/truststore.jks
> ssl.truststore.password=mystorepassword
> EOF

kafka-topics --bootstrap-server kafka.xyz.svc.cluster.local:9071 --command-config /tmp/kafka.properties --list
```

### View in Control Center


Create a `port-forward`:  
```
kubectl --namespace destination port-forward controlcenter-0 9021:9021 
```

Open Confluent Control Center:  
https://localhost:9021/  

## Check Schemas 

You can create Schemas via Control Center:
* Select topic with schema or create one
* Click schema tab
* Set schema

After that you can check where it was created: 

```
nslist=(destination     
dev              
stage           
test            
xyz )

for i in $nslist;do echo "curl url https://schemaregistry.$i.svc.cluster.local:8081/subjects" \
&& echo "\n"  \
&& k -n dev exec -it  kafka-0 -- curl -k  https://schemaregistry.$i.svc.cluster.local:8081/subjects \
&& echo "\n" \
; done
```

The above will output for example: 

```
curl url https://schemaregistry.destination.svc.cluster.local:8081/subjects


Defaulted container "kafka" out of: kafka, config-init-container (init)
[]

curl url https://schemaregistry.dev.svc.cluster.local:8081/subjects


Defaulted container "kafka" out of: kafka, config-init-container (init)
["dev-sr-value"]

curl url https://schemaregistry.stage.svc.cluster.local:8081/subjects


Defaulted container "kafka" out of: kafka, config-init-container (init)
[]

curl url https://schemaregistry.test.svc.cluster.local:8081/subjects


Defaulted container "kafka" out of: kafka, config-init-container (init)
[]

curl url https://schemaregistry.xyz.svc.cluster.local:8081/subjects


Defaulted container "kafka" out of: kafka, config-init-container (init)
[]
```

## Tear down

```
kubectl delete -f $TUTORIAL_HOME/components-dev.yaml
kubectl delete -f $TUTORIAL_HOME/components-stage.yaml
kubectl delete -f $TUTORIAL_HOME/components-test.yaml
kubectl delete -f $TUTORIAL_HOME/components-xyz.yaml
kubectl delete -f $TUTORIAL_HOME/components-destination.yaml
kubectl delete -f $TUTORIAL_HOME/controlcenter.yaml

nslist=(destination     
dev              
stage           
test            
xyz )

for i in $nslist; do kubectl delete -n $i secret credential ca-pair-sslcerts; done
for i in $nslist; do kubectl delete ns $i; done
helm delete confluent-operator
```
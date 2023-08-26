cat <<EOF > /tmp/james
sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required username=james password=james-secret;
sasl.mechanism=PLAIN
security.protocol=SASL_PLAINTEXT
EOF



cat <<EOF > /tmp/mds
sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required username=mds password="Developer!";
sasl.mechanism=PLAIN
security.protocol=SASL_PLAINTEXT
EOF


cat <<-EOF > /opt/confluentinc/kafka.properties
bootstrap.servers=kafka.confluent.svc.cluster.local:9071
security.protocol=SSL
ssl.keystore.location=/mnt/sslcerts/keystore.p12
ssl.keystore.password=mystorepassword
ssl.truststore.location=/mnt/sslcerts/truststore.p12
ssl.truststore.password=mystorepassword
EOF



kafka-topics --bootstrap-server kafka.confluent.svc.cluster.local:9071 \
--list \
--command-config /opt/confluentinc/kafka.properties

kafka-acls --bootstrap-server kafka.confluent.svc.cluster.local:9071 \
--list \
--command-config /opt/confluentinc/kafka.properties


kafka-topics --bootstrap-server kafka.confluent.svc.cluster.local:9095 \
--list \
--command-config /tmp/james

kafka-acls --bootstrap-server kafka.confluent.svc.cluster.local:9095 \
--list \
--command-config /tmp/james

 /bin/kafka-acls --bootstrap-server kafka.confluent.svc.cluster.local:9071 \
 --command-config /opt/confluentinc/kafka.properties \
 --add \
 --allow-principal "User:james" \
 --operation all \
 --cluster kafka-cluster

 /bin/kafka-acls --bootstrap-server kafka.confluent.svc.cluster.local:9071 \
 --command-config /opt/confluentinc/kafka.properties \
 --add \
 --allow-principal "User:james" \
 --operation all \
 --topic *



 /bin/kafka-acls --bootstrap-server kafka.confluent.svc.cluster.local:9071 \
 --command-config /opt/confluentinc/kafka.properties \
 --remove \
 --allow-principal "User:james" \
 --operation all \
 --cluster kafka-cluster

 /bin/kafka-acls --bootstrap-server kafka.confluent.svc.cluster.local:9071 \
 --command-config /opt/confluentinc/kafka.properties \
 --remove \
 --allow-principal "User:james" \
 --operation all \
 --topic *



 /bin/kafka-acls --bootstrap-server kafka.confluent.svc.cluster.local:9071 \
 --command-config /opt/confluentinc/kafka.properties \
 --add \
 --allow-principal "Group:c3users" \
 --operation all \
 --cluster kafka-cluster

 /bin/kafka-acls --bootstrap-server kafka.confluent.svc.cluster.local:9071 \
 --command-config /opt/confluentinc/kafka.properties \
 --add \
 --allow-principal "Group:c3users" \
 --operation all \
 --topic *


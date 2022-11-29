cat /etc/os-release 
NAME="Red Hat Enterprise Linux"
VERSION="8.6 (Ootpa)"
ID="rhel"
ID_LIKE="fedora"
VERSION_ID="8.6"
PLATFORM_ID="platform:el8"
PRETTY_NAME="Red Hat Enterprise Linux 8.6 (Ootpa)"
ANSI_COLOR="0;31"
CPE_NAME="cpe:/o:redhat:enterprise_linux:8::baseos"
HOME_URL="https://www.redhat.com/"
DOCUMENTATION_URL="https://access.redhat.com/documentation/red_hat_enterprise_linux/8/"
BUG_REPORT_URL="https://bugzilla.redhat.com/"

REDHAT_BUGZILLA_PRODUCT="Red Hat Enterprise Linux 8"
REDHAT_BUGZILLA_PRODUCT_VERSION=8.6
REDHAT_SUPPORT_PRODUCT="Red Hat Enterprise Linux"
REDHAT_SUPPORT_PRODUCT_VERSION="8.6"


1001           1       0 37 11:51 ?        00:03:03 java -cp /usr/share/java/acl/*:/usr/share/java/confluent-control-center/*:/usr/share/java/monitoring-interceptors/*:/usr/share/java/rest-utils/*:/usr/share/java/confluent-telemetry/*:/usr/share/java/confluent-common/*: -Dcom.sun.management.jmxremote.port= -Dconfluent.controlcenter.log.dir=/usr/logs -Dlog4j.configuration=file:/opt/confluentinc/etc/controlcenter/log4j.properties -Djava.rmi.server.hostname=moshevtwocontrolcenter-0.moshevtwocontrolcenter.confluent.svc.cluster.local -Dcom.sun.management.jmxremote=true -Dcom.sun.management.jmxremote.authenticate=false -Dcom.sun.management.jmxremote.local.only=false -Dcom.sun.management.jmxremote.port=7203 -Dcom.sun.management.jmxremote.rmi.port=7203 -Dcom.sun.management.jmxremote.ssl=false -Djava.awt.headless=true -Djava.security.auth.login.config=/mnt/config/shared/jaas-config.file -Djdk.tls.ephemeralDHKeySize=2048 -Djdk.tls.server.enableSessionTicketExtension=false -XX:+ExplicitGCInvokesConcurrent -XX:+PrintFlagsFinal -XX:+UnlockDiagnosticVMOptions -XX:+UseG1GC -XX:ConcGCThreads=1 -XX:G1HeapRegionSize=16 -XX:InitiatingHeapOccupancyPercent=35 -XX:MaxGCPauseMillis=20 -XX:MaxMetaspaceFreeRatio=80 -XX:MetaspaceSize=96m -XX:MinMetaspaceFreeRatio=50 -XX:ParallelGCThreads=1 -server -javaagent:/usr/share/java/cp-base-new/disk-usage-agent-7.3.0.jar=/opt/confluentinc/etc/controlcenter/disk-usage-agent.properties -javaagent:/usr/share/java/cp-base-new/jolokia-jvm-1.7.1.jar=port=7777,host=0.0.0.0 -javaagent:/usr/share/java/cp-base-new/jmx_prometheus_javaagent-0.14.0.jar=7778:/mnt/config/shared/jmx-exporter.yaml io.confluent.controlcenter.ControlCenter /opt/confluentinc/etc/controlcenter/controlcenter.properties

bash-4.4$ cat /opt/confluentinc/etc/controlcenter/controlcenter.properties
bootstrap.servers=pkc-e8mp5.eu-west-1.aws.confluent.cloud:9092
config.providers=file
config.providers.file.class=org.apache.kafka.common.config.provider.FileConfigProvider
confluent.controlcenter.auth.restricted.roles=readonlyusers
confluent.controlcenter.auth.session.expiration.ms=600000
confluent.controlcenter.connect.connect.cluster=http://connect.confluent.svc.cluster.local:8083
confluent.controlcenter.data.dir=/mnt/data/data0
confluent.controlcenter.id=0
confluent.controlcenter.name=_confluent-controlcenter
confluent.controlcenter.rest.authentication.method=BASIC
confluent.controlcenter.rest.authentication.realm=ldap
confluent.controlcenter.rest.authentication.roles=c3users,readonlyusers
confluent.controlcenter.rest.authentication.skip.paths=/2.0/status/app_info
confluent.controlcenter.rest.listeners=http://0.0.0.0:9021
confluent.controlcenter.streams.sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required username="${file:/mnt/secrets/cloud-plain/plain.txt:username}" password="${file:/mnt/secrets/cloud-plain/plain.txt:password}";
confluent.controlcenter.streams.sasl.mechanism=PLAIN
confluent.controlcenter.streams.security.protocol=SASL_SSL
confluent.controlcenter.topic.inspection.enable=true
confluent.license=${file:/mnt/secrets/internal-confluent-operator-licensing/license.txt:license}


https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/security_hardening/using-shared-system-certificates_security-hardening



# get certificates: 
wget https://letsencrypt.org/certs/isrgrootx1.der
wget https://letsencrypt.org/certs/isrg-root-x2.der

wget https://letsencrypt.org/certs/lets-encrypt-r3-cross-signed.der
wget https://letsencrypt.org/certs/lets-encrypt-r4-cross-signed.der

wget https://letsencrypt.org/certs/lets-encrypt-e1.der
wget https://letsencrypt.org/certs/lets-encrypt-e2.der

# import to the jvm truststore 
keytool -trustcacerts -keystore truststore.jks -storepass mystorepassword -noprompt -importcert -alias isrgrootx1 -file isrgrootx1.der
keytool -trustcacerts -keystore truststore.jks -storepass mystorepassword -noprompt -importcert -alias isrgrootx2 -file isrg-root-x2.der
keytool -trustcacerts -keystore truststore.jks -storepass mystorepassword -noprompt -importcert -alias letsencryptauthorityr3 -file lets-encrypt-r3-cross-signed.der
keytool -trustcacerts -keystore truststore.jks -storepass mystorepassword -noprompt -importcert -alias letsencryptauthorityr4 -file lets-encrypt-r4-cross-signed.der
keytool -trustcacerts -keystore truststore.jks -storepass mystorepassword -noprompt -importcert -alias letsencryptauthoritye1 -file lets-encrypt-e1.der
keytool -trustcacerts -keystore truststore.jks -storepass mystorepassword -noprompt -importcert -alias letsencryptauthoritye2 -file lets-encrypt-e2.der


# create new secret holding the new truststore 

    kubectl create secret generic ldaps-tls-ccloud \
    --from-file=truststore.jks=$TUTORIAL_HOME/ldapswithccloudcerts/truststore.jks \
    --from-file=jksPassword.txt=$TUTORIAL_HOME/ldapswithccloudcerts/jksPassword.txt \
    --namespace confluent

Update the CR 
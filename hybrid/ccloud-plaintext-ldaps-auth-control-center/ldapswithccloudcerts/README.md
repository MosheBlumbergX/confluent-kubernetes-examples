# These steps were already done and there is no need to do it again 

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
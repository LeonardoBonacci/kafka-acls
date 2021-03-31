openssl req -new -x509 -keyout ca-key -out ca-cert -days 365
keytool -keystore kafka.truststore.jks -alias CARoot -importcert -file ca-cert

OU=broker
keytool -keystore kafka.server0.keystore.jks -alias localhost -keyalg RSA -validity 365 -genkey
keytool -keystore kafka.server0.keystore.jks -alias localhost -certreq -file cert-file
openssl x509 -req -CA ca-cert -CAkey ca-key -in cert-file -out cert-signed -days 365 -CAcreateserial -passin pass:xxx
keytool -keystore kafka.server0.keystore.jks -alias CARoot -importcert -file ca-cert
keytool -keystore kafka.server0.keystore.jks -alias localhost -importcert -file cert-signed

OU=broker
keytool -keystore kafka.server1.keystore.jks -alias localhost -keyalg RSA -validity 365 -genkey
keytool -keystore kafka.server1.keystore.jks -alias localhost -certreq -file cert-file
keytool -keystore kafka.server1.keystore.jks -alias CARoot -importcert -file ca-cert
openssl x509 -req -CA ca-cert -CAkey ca-key -in cert-file -out cert-signed -days 365 -CAcreateserial -passin pass:xxx
keytool -keystore kafka.server1.keystore.jks -alias localhost -importcert -file cert-signed

OU=admin
keytool -keystore kafka.admin.client.keystore.jks -alias localhost -keyalg RSA -validity 365 -genkey
keytool -keystore kafka.admin.client.keystore.jks -alias localhost -certreq -file client-cert-file
openssl x509 -req -CA ca-cert -CAkey ca-key -in client-cert-file -out client-cert-signed -days 365 -CAcreateserial -passin pass:xxx
keytool -keystore kafka.admin.client.keystore.jks -alias CARoot -importcert -file ca-cert
keytool -keystore kafka.admin.client.keystore.jks -alias localhost -importcert -file client-cert-signed

OU=dummy
keytool -keystore kafka.dummy.client.keystore.jks -alias localhost -keyalg RSA -validity 365 -genkey
keytool -keystore kafka.dummy.client.keystore.jks -alias localhost -certreq -file client-cert-file
openssl x509 -req -CA ca-cert -CAkey ca-key -in client-cert-file -out client-cert-signed -days 365 -CAcreateserial -passin pass:xxx
keytool -keystore kafka.dummy.client.keystore.jks -alias CARoot -importcert -file ca-cert
keytool -keystore kafka.dummy.client.keystore.jks -alias localhost -importcert -file client-cert-signed

server0.properties
-----------
broker.id=0
listeners=SSL://localhost:9093
advertised.listeners=SSL://localhost:9093

ssl.keystore.location=/.../kafka.server0.keystore.jks
ssl.keystore.password=xxx
ssl.key.password=xxx
ssl.truststore.location=/.../kafka.truststore.jks
ssl.truststore.password=xxx

ssl.client.auth=required
security.inter.broker.protocol=SSL

authorizer.class.name=kafka.security.auth.SimpleAclAuthorizer
super.users=User:broker;User:admin;User:kafka-broker-metric-reporter
#principal is organizatinoal unit OU
ssl.principal.mapping.rules=RULE:^CN=(.*?),OU=(.*?),O=(.*?),L=(.*?),ST=(.*?),C=(.*?)$/$2/L,DEFAULT
-----------

server1.properties
-----------
broker.id=1
listeners=SSL://localhost:19093
advertised.listeners=SSL://localhost:19093

ssl.keystore.location=/.../kafka.server1.keystore.jks
ssl.keystore.password=xxx
ssl.key.password=xxx
ssl.truststore.location=/.../kafka.truststore.jks
ssl.truststore.password=xxx

ssl.client.auth=required
security.inter.broker.protocol=SSL

authorizer.class.name=kafka.security.auth.SimpleAclAuthorizer
super.users=User:broker;User:admin;User:kafka-broker-metric-reporter
#principal is organizatinoal unit
ssl.principal.mapping.rules=RULE:^CN=(.*?),OU=(.*?),O=(.*?),L=(.*?),ST=(.*?),C=(.*?)$/$2/L,DEFAULT
-----------

export KAFKA_OPTS=-Djavax.net.debug=all
log4j.logger.kafka.authorizer.logger=DEBUG, authorizerAppender

./kafka-topics --create --bootstrap-server localhost:9093 -topic test-topic --command-config /.../local-admin-ssl.properties --replication-factor 2
./kafka-topics --create --bootstrap-server localhost:9093 -topic test-topic2 --command-config /.../local-dummy-ssl.properties --replication-factor 2 

./kafka-acls --bootstrap-server localhost:9093 --command-config /.../local-admin-ssl.properties \
 --add --allow-principal User:dummy  \
 --operation write --topic test-topic

./kafka-acls --bootstrap-server localhost:19093 --list --command-config /.../local-admin-ssl.properties

./kafka-console-producer --broker-list localhost:19093 -topic test-topic --producer.config /.../local-dummy-ssl.properties

./kafka-acls --bootstrap-server localhost:9093 --command-config /.../local-admin-ssl.properties \
 --add --allow-principal User:dummy  \
 --operation read --topic test-topic --group '*'

./kafka-console-consumer --bootstrap-server localhost:19093 --topic test-topic --consumer.config /.../local-dummy-ssl.properties --from-beginning

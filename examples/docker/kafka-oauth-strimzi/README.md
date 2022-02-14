Kafka using OAuth2 for authentication
=====================================

This projects build and runs a docker container running a single-broker Kafka cluster configured with OAuth2 support.


Building
--------

Copy resources to prepare the docker-compose project by running:

    mvn clean package
    

Preparing
---------

This demo comes with several configurations using three different OAuth2 authorization servers - Keycloak, Hydra, and Spring Authorization Server. 
It is important that all clients access the OAuth2 endpoints using the same url schema, host and port.
 
First, determine your machine's local network IP address, and add keycloak / hydra / spring entry to your `/etc/host`.

You can use `ifconfig` utility. On macOS for example you can run:

    ifconfig en0 | grep 'inet ' | awk '{print $2}'

Then, add keycloak / hydra / spring entry to your `/etc/hosts` file 

    <YOUR_IP_ADDRESS>    hydra
    <YOUR_IP_ADDRESS>    kafka
    <YOUR_IP_ADDRESS>    opa


Usually you can simply use `localhost` instead of <YOUR_IP_ADDRESS>.

Before each run you may want to delete any previous instances by using:

    docker rm -f kafka zookeeper


Running using Hydra with SSL and JWT tokens
----------------------------------------------

From `docker` directory run:

    docker-compose -f compose.yml -f kafka-oauth-strimzi/compose-hydra-jwt.yml -f hydra/compose-with-jwt.yml -f hydra-import/compose.yml up --build

Kafka broker should be available on localhost:9092. It connects to Hydra using `https://hydra:4444`

To change OPA policies, you can find them within `example/docker/kafka-oauth-strimzi/bundles. 
In case you want to change ACLs: 
1. Install OPA client
    brew install opa
2. Build ACLs
    opa build --bundle policies/ --output bundles/bundle.tar.gz

Add self signed certs within your local java truststore: 
    sudo keytool -import -alias hydra -file ca.crt -keystore /Library/Java/JavaVirtualMachines/jdk1.8.0_144.jdk/Contents/Home/jre/lib/security/cacerts

`ca.crt`location: /Users/<user>/Documents/Kafka/strimzi-kafka-oauth-0.10.0/examples/docker/hydra-import/ca.crt
PWD of `cacerts` is usually `changeit`


Running Kafka Client
---------------------------------------
Download library which can use  `kafka-console-producer.sh` and within directory create file:

    producer-oauth.config
    oauth.ssl.truststore.location=/Users/<user>/Documents/Kafka/strimzi-kafka-oauth-0.10.0/examples/docker/hydra-import/ca.crt
    oauth.ssl.truststore.type=PEM
    ssl.truststore.location=/Users/<user>/Documents/Kafka/strimzi-kafka-oauth-0.10.0/examples/docker/certificates/ca-truststore.p12
    ssl.truststore.password=changeit
    security.protocol=SASL_PLAINTEXT
    sasl.mechanism=OAUTHBEARER
    sasl.jaas.config=org.apache.kafka.common.security.oauthbearer.OAuthBearerLoginModule required;
    sasl.login.callback.handler.class=io.strimzi.kafka.oauth.client.JaasClientOauthLoginCallbackHandler

Export following vars for Kafka Producer Client: 
    export OAUTH_CLIENT_SECRET=kafka-producer-client-secret && export OAUTH_CLIENT_ID=kafka-producer-client
    export OAUTH_TOKEN_ENDPOINT_URI=https://hydra:4444/oauth2/token

Export following vars for Kafka Consumer Client: 
    export OAUTH_CLIENT_SECRET=kafka-consumer-client-secret && export OAUTH_CLIENT_ID=kafka-consumer-client
    export OAUTH_TOKEN_ENDPOINT_URI=https://hydra:4444/oauth2/token

Export following vars for Kafka Broker Client: 
    export OAUTH_CLIENT_SECRET=kafka-broker-secret && export OAUTH_CLIENT_ID=kafka-broker
    export OAUTH_TOKEN_ENDPOINT_URI=https://hydra:4444/oauth2/token


Run Kafka Client: 
    bin/kafka-console-producer --bootstrap-server kafka:9092 --topic credit-scores --producer.config producer-oauth.config


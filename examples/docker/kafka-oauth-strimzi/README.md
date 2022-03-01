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


Running using custom made plugin JWT decoding + OPA
----------------------------------------------

From `docker` directory run:

    docker-compose -f compose.yml -f kafka-oauth-strimzi/compose-plain-jwt.yml up --build


Running using custom made plugin JWT decoding and SSL as second listener + OPA
----------------------------------------------

From `docker` directory run:

    docker-compose -f compose.yml -f kafka-oauth-strimzi/compose-ssl-jwt.yml up --build

In case you are running Hydra then - Kafka broker should be available on localhost:9092. It connects to Hydra using `https://hydra:4444`

In case you are running JWT + SSL then: 
1. port 9092 is for SSL
2. port 9093 is for OAUTHBEARER

To change OPA policies, you can find them within `example/docker/kafka-oauth-strimzi/bundles. 
In case you want to change ACLs: 
1. Install OPA client

    brew install opa
2. Build ACLs

    opa build --bundle policies/ --output bundles/bundle.tar.gz


[For Hydra Deployments] 

Add self signed certs within your local java truststore: 

    sudo keytool -import -alias hydra -file ca.crt -keystore /Library/Java/JavaVirtualMachines/jdk1.8.0_144.jdk/Contents/Home/jre/lib/security/cacerts

`ca.crt`location:   /Users/<user>/Documents/Kafka/strimzi-kafka-oauth-0.10.0/examples/docker/hydra-import/ca.crt

    PWD of `cacerts` is usually `changeit`


Running Kafka Client with Hydra
---------------------------------------
Download library which can use  `kafka-console-producer.sh` and within directory create file `producer-oauth.config`:

    
    oauth.ssl.truststore.location=/Users/<user>/Documents/Kafka/strimzi-kafka-oauth-0.10.0/examples/docker/hydra-import/ca.crt
    oauth.ssl.truststore.type=PEM
    ssl.truststore.location=/Users/<user>/Documents/Kafka/strimzi-kafka-oauth-0.10.0/examples/docker/certificates/ca-truststore.p12
    ssl.truststore.password=changeit
    security.protocol=SASL_PLAINTEXT
    sasl.mechanism=OAUTHBEARER
    sasl.jaas.config=org.apache.kafka.common.security.oauthbearer.OAuthBearerLoginModule required;
    sasl.login.callback.handler.class=io.strimzi.kafka.oauth.client.JaasClientOauthLoginCallbackHandler

Export following vars for Kafka Producer Client (based on policy this user can write to any topic): 
    
    export OAUTH_CLIENT_SECRET=kafka-producer-client-secret && export OAUTH_CLIENT_ID=kafka-producer-client
    export OAUTH_TOKEN_ENDPOINT_URI=https://hydra:4444/oauth2/token

Export following vars for Kafka Consumer Client (based on policy this user cannot read any topics): 
    
    export OAUTH_CLIENT_SECRET=kafka-consumer-client-secret && export OAUTH_CLIENT_ID=kafka-consumer-client
    export OAUTH_TOKEN_ENDPOINT_URI=https://hydra:4444/oauth2/token

Export following vars for Kafka Broker Client: 
    
    export OAUTH_CLIENT_SECRET=kafka-broker-secret && export OAUTH_CLIENT_ID=kafka-broker
    export OAUTH_TOKEN_ENDPOINT_URI=https://hydra:4444/oauth2/token

If passing `OAUTH_CLIENT_SECRET` and `OAUTH_CLIENT_ID` is not what you want you can obtain JWT in front and pass it to Kafka as follows: 
    
    # Hydra install
    brew install ory/tap/hydra
    
    # Obtain token calling hydra auth api
    hydra token client  --endpoint https://hydra:4444/ --client-id kafka-producer-client --client-secret kafka-producer-client-secret --skip-tls-verify
    
    Result: 
    <JWT>
    
    # export VAR
    export OAUTH_ACCESS_TOKEN=<jwt>
 
Be aware that `OAUTH_ACCESS_TOKEN` always overrides `OAUTH_CLIENT_SECRET` + `OAUTH_CLIENT_ID`
        

Running Kafka Client with OAUTHBEARER with custom plugin
---------------------------------------
Download library which can use  `kafka-console-producer.sh` and within directory create file `producer-oauth.config`:

    
    security.protocol=SASL_PLAINTEXT
    sasl.mechanism=OAUTHBEARER
    sasl.jaas.config=org.apache.kafka.common.security.oauthbearer.OAuthBearerLoginModule required;
    sasl.login.callback.handler.class=io.strimzi.kafka.oauth.client.JaasClientOauthLoginCallbackHandler

Export following vars for Kafka Producer Client (based on policy this user can write to any topic): 
    
    export OAUTH_ACCESS_TOKEN=<JWT>

Export following vars for Kafka Consumer Client (based on policy this user cannot read any topics): 
    
    export OAUTH_ACCESS_TOKEN=<JWT>



Run Kafka Client: 
    
    bin/kafka-console-producer --bootstrap-server kafka:9092 --topic credit-scores --producer.config producer-oauth.config

    bin/kafka-console-consumer --bootstrap-server kafka:9092 --topic credit-scores --consumer.config producer-oauth.config --group test-group

# kafka-jmx-grafana-docker

Docker-compose file for Confluent Kafka with configuration mounted as properties files. Brings up Kafka and components with JMX metrics exposed and visualized using Prometheus and Grafana

## Start

```
docker-compose up -d
```

## Usage

The docker-compose file brings up 3 node kafka cluster with security enabled. Each service in the compose file has its properties/configurations mounted as a volume from a directory with the same name as the service.

Check the kafka server.properties for more details about the Kafka setup.

### Health

Check if all components are up and running using

```bash
docker-compose ps -a
# Ensure there are no Exited services and all containers have the status `Up`
```


### Client

To use a kafka client, exec into the `kfkclient` container which contains the Kafka CLI and other tools necessary for troubleshooting Kafka. THe `kfkclient` container also contains a properties file mounted to `/opt/client`, which can be used to define the client properties for communicating with Kafka.

```
docker exec -it kfkclient bash
```

### Logs

Check the logs of the respective service by its container name.

```bash
docker logs <container_name> # docker logs kafka1
```

### Restarting services

To restart a particular service - 

```bash
docker-compose restart <service_name> # docker-compose restart kafka1
# OR
docker-compose up -d --force-recreate <service_name> # docker-compose up -d --force-recreate kafka1
```

# Scenario 3

> **Before starting ensure that there are no other versions of the sandbox running**
> Run `docker-compose down -v` before starting

1. Start the scenario with `docker-compose up -d`
2. Wait for all services to be up and healthy `docker-compose ps`

## Problem Statement

The client is unable to interact with the cluster using `kafka-topics`, `kafka-console-producer` or `kafka-console-consumer` using the super user `bob`(password - `bob-secret`) with some of the kakfa brokers as the bootstrap server. Only one of the kafka broker works as the bootstrap server.

The error message when using kafka2 or kafka3 as the bootstrap-server

```
[2023-07-26 13:20:58,720] ERROR [AdminClient clientId=adminclient-1] Connection to node 3 (kafka3/192.168.112.8:39092) failed authentication due to: Authentication failed: Invalid username or password (org.apache.kafka.clients.NetworkClient)
[2023-07-26 13:20:58,722] WARN [AdminClient clientId=adminclient-1] Metadata update failed due to authentication error (org.apache.kafka.clients.admin.internals.AdminMetadataManager)
org.apache.kafka.common.errors.SaslAuthenticationException: Authentication failed: Invalid username or password
Error while executing topic command : Authentication failed: Invalid username or password
[2023-07-26 13:20:58,907] ERROR org.apache.kafka.common.errors.SaslAuthenticationException: Authentication failed: Invalid username or password
```

Commands ran on the `kfkclient` host

```bash
kafka-topics --bootstrap-server <broker>:<port> --command-config /opt/client/client.properties --list
```

##**Solution**



**1. Regenerate certificates: **

```
Steps to Set Up Kafka with New Certificates 

Clean Up Old Files 

Clean Up Old Files in ~/scenario_3/cp-sandbox/certs: 

cd cp-sandbox/certs 
rm -f ca-key ca-cert cert-signed-1 cert-signed-2 cert-signed-3 cert-file-1 cert-file-2 cert-file-3 ca-cert.srl 
 

Clean Up Old Files in Each Broker Directory: 

Assuming that the Kafka broker directories (kafka1, kafka2, kafka3) are inside ~/scenario_3/cp-sandbox/, execute the following: 

# For kafka1 
cd ../kafka1 
rm -f kafka.server.truststore.jks kafka.server.keystore.jks 
 
# For kafka2 
cd ../kafka2 
rm -f kafka.server.truststore.jks kafka.server.keystore.jks 
 
# For kafka3 
cd ../kafka3 
rm -f kafka.server.truststore.jks kafka.server.keystore.jks 
 

 

Create New Certificates for the CA 

Run the following commands in the ~/scenario_3/cp-sandbox/certs directory: 

cd ../certs 
 
# Step 1: Create a new Root CA 
openssl req -new -x509 -days 365 -keyout ca-key -out ca-cert -subj "/C=DE/ST=NRW/L=MS/O=juplo/OU=kafka/CN=Root-CA" -passout pass:kafka-broker 
 

 

Create Truststore for Each Kafka Broker 

Now, import the CA certificate into the truststore for each broker: 

# For kafka1 
keytool -keystore ../kafka1/kafka.server.truststore.jks -storepass kafka-broker -import -alias ca-root -file ca-cert -noprompt 
 
# For kafka2 
keytool -keystore ../kafka2/kafka.server.truststore.jks -storepass kafka-broker -import -alias ca-root -file ca-cert -noprompt 
 
# For kafka3 
keytool -keystore ../kafka3/kafka.server.truststore.jks -storepass kafka-broker -import -alias ca-root -file ca-cert -noprompt 
 

 

Generate Keystores and Certificates for Each Broker 

For kafka1: 

# Step 3: Generate a Key Pair for kafka1 
keytool -keystore ../kafka1/kafka.server.keystore.jks -storepass kafka-broker -alias kafka1 -validity 365 -keyalg RSA -genkeypair -keypass kafka-broker -dname "CN=kafka1,OU=kafka,O=juplo,L=MS,ST=NRW,C=DE" 
 
# Step 4: Create a CSR for kafka1 
keytool -alias kafka1 -keystore ../kafka1/kafka.server.keystore.jks -certreq -file cert-file-1 -storepass kafka-broker -keypass kafka-broker 
 
# Step 5: Sign the CSR with the CA 
openssl x509 -req -CA ca-cert -CAkey ca-key -in cert-file-1 -out cert-signed-1 -days 365 -CAcreateserial -passin pass:kafka-broker -extensions SAN -extfile <(printf "\n[SAN]\nsubjectAltName=DNS:kafka1,DNS:localhost") 
 
# Step 6: Import the CA Certificate into the Keystore 
keytool -importcert -keystore ../kafka1/kafka.server.keystore.jks -alias ca-root -file ca-cert -storepass kafka-broker -keypass kafka-broker -noprompt 
 
# Step 7: Import the Signed Certificate into the Keystore 
keytool -keystore ../kafka1/kafka.server.keystore.jks -alias kafka1 -import -file cert-signed-1 -storepass kafka-broker -keypass kafka-broker -noprompt 
 

For kafka2: 

# Step 3: Generate a Key Pair for kafka2 
keytool -keystore ../kafka2/kafka.server.keystore.jks -storepass kafka-broker -alias kafka2 -validity 365 -keyalg RSA -genkeypair -keypass kafka-broker -dname "CN=kafka2,OU=kafka,O=juplo,L=MS,ST=NRW,C=DE" 
 
# Step 4: Create a CSR for kafka2 
keytool -alias kafka2 -keystore ../kafka2/kafka.server.keystore.jks -certreq -file cert-file-2 -storepass kafka-broker -keypass kafka-broker 
 
# Step 5: Sign the CSR with the CA 
openssl x509 -req -CA ca-cert -CAkey ca-key -in cert-file-2 -out cert-signed-2 -days 365 -CAcreateserial -passin pass:kafka-broker -extensions SAN -extfile <(printf "\n[SAN]\nsubjectAltName=DNS:kafka2,DNS:localhost") 
 
# Step 6: Import the CA Certificate into the Keystore 
keytool -importcert -keystore ../kafka2/kafka.server.keystore.jks -alias ca-root -file ca-cert -storepass kafka-broker -keypass kafka-broker -noprompt 
 
# Step 7: Import the Signed Certificate into the Keystore 
keytool -keystore ../kafka2/kafka.server.keystore.jks -alias kafka2 -import -file cert-signed-2 -storepass kafka-broker -keypass kafka-broker -noprompt 
 

For kafka3: 

# Step 3: Generate a Key Pair for kafka3 
keytool -keystore ../kafka3/kafka.server.keystore.jks -storepass kafka-broker -alias kafka3 -validity 365 -keyalg RSA -genkeypair -keypass kafka-broker -dname "CN=kafka3,OU=kafka,O=juplo,L=MS,ST=NRW,C=DE" 
 
# Step 4: Create a CSR for kafka3 
keytool -alias kafka3 -keystore ../kafka3/kafka.server.keystore.jks -certreq -file cert-file-3 -storepass kafka-broker -keypass kafka-broker 
 
# Step 5: Sign the CSR with the CA 
openssl x509 -req -CA ca-cert -CAkey ca-key -in cert-file-3 -out cert-signed-3 -days 365 -CAcreateserial -passin pass:kafka-broker -extensions SAN -extfile <(printf "\n[SAN]\nsubjectAltName=DNS:kafka3,DNS:localhost") 
 
# Step 6: Import the CA Certificate into the Keystore 
keytool -importcert -keystore ../kafka3/kafka.server.keystore.jks -alias ca-root -file ca-cert -storepass kafka-broker -keypass kafka-broker -noprompt 
 
# Step 7: Import the Signed Certificate into the Keystore 
keytool -keystore ../kafka3/kafka.server.keystore.jks -alias kafka3 -import -file cert-signed-3 -storepass kafka-broker -keypass kafka-broker -noprompt 
 

 
```
![image](https://github.com/user-attachments/assets/1e99bc39-4289-4074-b85a-f4b8cd6d03df)


Password corrected to bob-secret 

 
```
Incorrect (old) line: 
 
    user_bob="b0b-secret" 
 
Correct (new) line: 
 
    user_bob="bob-secret" 

 ```
Restart docker service:

![image](https://github.com/user-attachments/assets/a84c4afb-2772-475e-83a4-5c60ce1b019a)

![image](https://github.com/user-attachments/assets/755c67fe-366f-4330-bc5f-31490b1312b0)



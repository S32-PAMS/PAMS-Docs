---
{"dg-publish":true,"permalink":"/for-security-personnel/","tags":["security"]}
---

> [!abstract] Security manual
> This manual provides comprehensive documentation of the security measures implemented in the Personnel and Asset Monitoring System (PAMS). It serves as a guide for setting up and maintaining the system's security features.

PAMS employs a multi-layered architecture designed to track and visualize the location data of tagged assets. The system integrates hardware components like ESP32 microcontrollers ([[Tags\|Tags]], [[Anchors\|Anchors]]) and software components including [[Server/MQTT Broker\|MQTT Broker]], [[Apache Kafka\|Apache Kafka]], and a [[Web server\|Web server]], among others.

# Local PKI setup

In the instance of a local server, PAMS will have a private CA for its Public Key Infrastructure (PKI) needs.

> [!note]
> In project directory, make a directory called `CA`, and go into it.

## Scripting 1 : Initial scripts

> [!note]
> PAMS prototype currently only uses 1 private root CA.
> 
> For prototyping purposes, this code will be made to keep the CA container running. For best practice, you should implement a offline root CA, and intermediate CAs to handle day-to-day issuance and revocation of certificates.
> 
>This manual will be a documentation of how we set our prototype up.

> [!important]
> Ensure that `chmod +x` is run for every shell script created here before the next section on running the scripts.

**Note**: This prototype is using some unsafe permissions in the scripts, assuming that everyone with access to the local server can have permissions to run all scripts. Adjust accordingly to your system.

### Docker files

Prepare a Docker container for Certificate Authority (CA) operations. Use the provided `Dockerfile`, `docker-compose.yml`, and initialisation scripts.

> [!attention]
> Change relevant fields to your instance's requirements.

`Dockerfile`

```dockerfile
# Containerised PKI
# (c)2019-2021, Karneliuk.com
# Take the latest Alpine image.
FROM alpine:latest

LABEL maintainer="capstonepams@gmail.com"
LABEL updated="2024-02-21"

# Everything below will be run inside the container when docker compose build is ran

# Update the packages and install openssl on it.
RUN apk update; apk add openssl

# Create the following directories:
#     /opt/rootca/local to store the key and certificate of the root CA.
#     /opt/rootca/input to store the CSRs.
#     /opt/rootca/output, which will store the certificates created based on CSRs.
RUN mkdir -p /opt/rootca; mkdir /opt/rootca/local; mkdir /opt/rootca/output; mkdir /opt/rootca/input
WORKDIR /opt/rootca

# Copy the script run.sh, which will keep the container busy all the time (see below).
COPY run.sh /opt/run.sh
# Copy the script init_ca.sh, which will initiaalise the creation of a root CA in the container
COPY init_ca.sh /opt/init_ca.sh
```

`docker-compose.yml`

```yaml
version: '3.8'
services:
  rust-log-processing:
    image: mcr.microsoft.com/devcontainers/rust:0-1-bullseye
    volumes:
      - ..:/workspaces:cached
      - /home/klass/Desktop/PAMS-Middleware/CA/mqtt_broker:/certs/mqtt_broker
      - /home/klass/Desktop/PAMS-Middleware/CA/kafka1:/certs/kafka1
      - /home/klass/Desktop/PAMS-Middleware/CA/rootCA:/certs/rootCA
    cap_add:
      - SYS_PTRACE
    security_opt:
      - seccomp:unconfined
    command: /bin/sh -c "while sleep 1000; do :; done"
  
  zookeeper:
    image: confluentinc/cp-zookeeper:7.3.0
    container_name: zookeeper
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000
  
  broker:
    image: confluentinc/cp-kafka:7.3.0
    volumes:
      - /home/klass/Desktop/PAMS-Middleware/CA/kafka1/kafka1.pem:/etc/certs/kafka1.pem
      - /home/klass/Desktop/PAMS-Middleware/CA/kafka1/kafka1.key:/etc/certs/kafka1.key
    container_name: broker
    ports:
    # To learn about configuring Kafka for access across networks see
    # https://www.confluent.io/blog/kafka-client-cannot-connect-to-broker-on-aws-on-docker-etc/
      # - "9092:9092"
      - "10920:10920"
      - "9093:9093"
    depends_on:
      - zookeeper
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: 'zookeeper:2181'
      KAFKA_AUTO_CREATE_TOPICS_ENABLE: "true"
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_INTERNAL:PLAINTEXT, LOCALHOST:PLAINTEXT, LOCALHOST2:PLAINTEXT
      # KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://172.21.0.5:9092,PLAINTEXT_INTERNAL://broker:29092, LOCALHOST://broker:10920, LOCALHOST2://broker:9093
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://172.21.0.5:9092,PLAINTEXT_INTERNAL://broker:29092, LOCALHOST://localhost:10920, LOCALHOST2://localhost:9093
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT_INTERNAL
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1
    # Define custom topics using environment variables
    # command: bash -c "/etc/confluent/docker/run & while ! kafka-topics --create --topic packet_data --if-not-exists --bootstrap-server localhost:10920; do sleep 5; done; while ! kafka-topics --create --topic packet_alive --if-not-exists --bootstrap-server localhost:10920; do sleep 5; done; while true; do sleep 1000; done"

    #kafka-configs --alter --entity-name packet_data --bootstrap-server localhost:9093 --add-config retention.ms=100 --entity-type topics


  nanomq:
    image: emqx/nanomq:0.14.1-full
    container_name: nanomq
    ports:
      - "1883:1883"
      - "8883:8883"
    environment:
      - NANOMQ_BROKER_URL="nmq-tcp://0.0.0.0:1883"
    volumes:
      - ../nanomq/nanomq.conf:/etc/nanomq.conf
      - /home/klass/Desktop/PAMS-Middleware/CA/rootCA/rootCA.pem:/certs/rootCA.pem
      - /home/klass/Desktop/PAMS-Middleware/CA/mqtt_broker/mqtt_broker.pem:/certs/mqtt_broker.pem
      - /home/klass/Desktop/PAMS-Middleware/CA/mqtt_broker/mqtt_broker.key:/certs/mqtt_broker.key

  # mqtt-kafka-bridge:
  #   build:
  #     context: ../mqtt-kafka-bridge
  #   depends_on:
  #     - broker
  #     - nanomq


  # jobmanager:
  #   # build: .
  #   image: apache/flink:1.18.0
  #   volumes:
  #     - ./:/home
  #   hostname: "jobmanager"
  #   expose:
  #     - "6123"
  #   ports:
  #     - "8081:8081"
  #   environment:
  #     - JOB_MANAGER_RPC_ADDRESS=jobmanager
  #   command: jobmanager 
    
    # NOT USED (but might be)
    # bash -c "run-application \
    #     --target kubernetes-application
    #     -Dkubernetes.cluster-id={your-custom-cluster-id} \
    #     -Dkubernetes.container.image = {your-custom-image-name} \
    #     -pyfs {the python file path with the container image} \
    #     -pym {the-entry-module-name} \
    #     -- pyFiles file:///../Flink/message_pb2.py\
    #     -- pyclientexec file:////../Flink/venv\
    #     -- python ../Flink/runStream.py"
    # NOT USED

  # taskmanager:
  #   image: apache/flink:1.18.0
  #   volumes:
  #   - ./:/home
  #   expose:
  #     - "6121"
  #     - "
```

### Scripting : Generate Root CA's Key and Certificate

> [!attention]
> Change the fields accordingly. We have coded the `alt_names` to be the IP address of the network we are using for this prototype. You can add more domain names or IP addresses depending on your system.
> > [!attention]
> > Ensure all path names match your system in all the scripts.

> [!note]
> Inside the `CA` directory, a subdirectory called `rootCA` is created after `setup_ca.sh` is run.

`init_ca.sh` : Initializes the CA by generating a private key and certificate.

```shell
#!/bin/sh

# init_ca.sh: This script initializes the CA by generating a private key and a certificate.

# Create an OpenSSL config file for our CA
cat > /opt/rootca/openssl.cnf <<EOF
[ req ]
default_bits = 4096
prompt = no
default_md = sha512
distinguished_name = dn
req_extensions = req_ext

[ dn ]
C = SG
ST = Singapore
L = Singapore
O = PAMS
OU = S32
CN = capstonepams@gmail.com

[ req_ext ]
subjectAltName = @alt_names

[ alt_names ]
IP.1 = 192.168.252.121
IP.2 = 192.168.223.121
EOF

# Generate the Root CA private key
openssl genrsa -aes256 -out /opt/rootca/local/rootCA.key 4096


# Enter PEM pass phrase
#SUTDcapstoneS32PAMS
# Verifying - Enter PEM pass phrase
#SUTDcapstoneS32PAMS

# Generate the Root CA certificate
openssl req -x509 -new -nodes -config /opt/rootca/openssl.cnf -key /opt/rootca/local/rootCA.key -sha512 -days 365 -out /opt/rootca/local/rootCA.pem

# Convert Certificates to DER Format
openssl x509 -outform der -in /opt/rootca/local/rootCA.pem -out /opt/rootca/local/rootCA_cert.der


# Convert DER Files to C Byte Arrays
xxd -i /opt/rootca/local/rootCA_cert.der > /opt/rootca/local/rootCA_cert.c

# Enter pass phrase for  /opt/rootca/local/rootCA.key
#SUTDcapstoneS32PAMS
```

`run.sh` : Keeps the Docker container running.

```shell
#!/bin/sh
while :; do
    sleep 1
done
```

`setup_ca.sh` : Script to automate the CA setup.

```shell
#!/bin/sh
# setup_ca.sh
# To run this automation script of creating ca in the docker container after the docker container is created

# Create /etc/docker/certs to store all the .pem and .key files securely if this folder has not yet been created.
# Because sudo, will need to enter fill in password for current user.
sudo rm -rf /etc/docker/certs
sudo mkdir -p /etc/docker/certs
sudo chmod 700 /etc/docker/certs

# Build the Docker image
docker compose build

# Start the Docker container
docker compose up -d

# Wait for a few seconds to ensure the container is running
sleep 10

# Run the CA initialization script inside the container
container_name=$(docker compose ps -q rootca)
docker exec -it $container_name /bin/sh /opt/init_ca.sh

# Retrieve the .pem file (public key) of root ca into host /etc/docker/certs directory
docker cp "$container_name":"/opt/rootca/local/rootCA.pem" "./rootCA.pem"
sudo cp ./rootCA.pem /etc/docker/certs/rootCA.pem 

# Set secure permissions for the rootCA.pem file
sudo chmod 600 /etc/docker/certs/rootCA.pem 

# Retrieve the .der file (generated from rootCA certificate) of root ca into host /etc/docker/certs directory
docker cp "$container_name":"/opt/rootca/local/rootCA_cert.der" "./rootCA_cert.der"
sudo cp ./rootCA_cert.der /etc/docker/certs/rootCA_cert.der

# Set secure permissions for the rootCA_cert.der file
sudo chmod 600 /etc/docker/certs/rootCA_cert.der 

# Retrieve the .c file (generated from rootca_cert.der) of root ca into host /etc/docker/certs directory
docker cp "$container_name":"/opt/rootca/local/rootCA_cert.c" "./rootCA_cert.c"
sudo cp ./rootCA_cert.c /etc/docker/certs/rootCA_cert.c

# Set secure permissions for the rootCA.pem file
sudo chmod 600 /etc/docker/certs/rootCA_cert.c

echo "production code"
echo =============
# for production, put all certs into CA folder inside PAMS-Middleware:
sudo rm -rf /home/klass/Desktop/PAMS-Middleware/CA/rootCA
sudo mkdir -p /home/klass/Desktop/PAMS-Middleware/CA/rootCA

echo "rootCA folder created in CA"
echo ===============

# docker cp "$container_name":"/opt/rootca/local/rootCA.pem" "/home/klass/Desktop/PAMS-Middleware/CA/rootCA/rootCA.pem"
sudo mv rootCA.pem /home/klass/Desktop/PAMS-Middleware/CA/rootCA/rootCA.pem
sudo mv rootCA_cert.der /home/klass/Desktop/PAMS-Middleware/CA/rootCA/rootCA_cert.der
sudo mv rootCA_cert.c /home/klass/Desktop/PAMS-Middleware/CA/rootCA/rootCA_cert.c

echo "rootCA.pem moved into rootCA folder"
# confirm is in CA folder:
ls /home/klass/Desktop/PAMS-Middleware/CA/rootCA -l
echo ===============
sudo chmod 444 /home/klass/Desktop/PAMS-Middleware/CA/rootCA/rootCA.pem
sudo chmod 444 /home/klass/Desktop/PAMS-Middleware/CA/rootCA/rootCA_cert.der
sudo chmod 444 /home/klass/Desktop/PAMS-Middleware/CA/rootCA/rootCA_cert.c
sudo chmod 755 /home/klass/Desktop/PAMS-Middleware/CA/rootCA

# Confirm that it has been successfully moved with the right permissions
#ls /etc/docker/certs -l

echo "CA setup completed. The container name is $container_name"

# docker container stop $container_name
```


> [!attention]
> The script needs root privileges, so provide the `sudo` password when prompted by the script.

For this prototype, we are keeping the root CA up. Please don't do this in production, and use intermediate CAs instead.

> [!tip]
> The container name is echo-ed upon creation of the CA. Take note of this for when you need to `docker container start` or `docker container stop`.

### Scripting : Generating Keys and CSRs for each component

> [!note]
> Create a subdirectory inside the `CA` directory, called `CSR`.
> ```
> - CA/
>	- rootCA/
>	- CSR/
> ```
> This section should be done inside the `CSR` subdirectory.

> [!attention]
> Fill in the fields accordingly to your system's instance.

- This code will see a directory called with the component name inside `CA` directory, containing the component's `.key` file.
- Inside the `CSR` directory, you will see the component's `.csr` (Certificate Signing Request)

#### For [[Server/MQTT Broker\|MQTT Broker]] : `mqtt_broker_csr.sh`

```shell
#!/bin/sh

# mqtt_broker_csr.sh: Generate Private Key and CSR for the MQTT Broker:

# Replace these variables according to your component and environment
COMPONENT_NAME="mqtt_broker"
DOMAIN="capstonepams@gmail.com"
IP_ADDRESS="192.168.252.121"  # Use the actual IP address
OUTPUT_DIR="./"   # output in the same working directory as this script

# Create an OpenSSL config file for CSR generation with SANs
cat > ${OUTPUT_DIR}${COMPONENT_NAME}_openssl.cnf <<EOF
[ req ]
default_bits = 2048
prompt = no
default_md = sha256
req_extensions = req_ext
distinguished_name = dn

[ dn ]
C=SG
ST=Singapore
L=Singapore
O=PAMS
OU=MQTTBroker
CN=${DOMAIN}

[ req_ext ]
subjectAltName = @alt_names

[ alt_names ]
DNS.1 = ${DOMAIN}
IP.1 = ${IP_ADDRESS}
IP.2 = 192.168.223.121
IP.3 = 192.168.35.121
EOF

# Generate a Private Key
openssl genrsa -out ${OUTPUT_DIR}${COMPONENT_NAME}.key 2048

# Copy the mqtt_broker.key (private key) to /etc/docker/certs
# Because sudo, will need to enter fill in password for current user.
echo "Copying key into /etc/docker/certs"
sudo cp ${OUTPUT_DIR}${COMPONENT_NAME}.key /etc/docker/certs/${COMPONENT_NAME}.key
sudo chmod 600 /etc/docker/certs/${COMPONENT_NAME}.key
echo ===========

# Set permissions for the generated file
sudo chmod 444 /home/klass/Desktop/PAMS-Middleware/CA/${COMPONENT_NAME}/${COMPONENT_NAME}.key

# Generate a CSR using the OpenSSL config file
openssl req -new -key ${OUTPUT_DIR}${COMPONENT_NAME}.key -out ${OUTPUT_DIR}${COMPONENT_NAME}.csr -config ${OUTPUT_DIR}${COMPONENT_NAME}_openssl.cnf -extensions req_ext

# move private key generated into mqtt_broker folder
sudo rm -rf /home/klass/Desktop/PAMS-Middleware/CA/${COMPONENT_NAME}
sudo mkdir -p /home/klass/Desktop/PAMS-Middleware/CA/${COMPONENT_NAME}
sudo mv ${OUTPUT_DIR}${COMPONENT_NAME}.key /home/klass/Desktop/PAMS-Middleware/CA/${COMPONENT_NAME}

echo "Generated ${COMPONENT_NAME}.key and ${COMPONENT_NAME}.csr in CA/${COMPONENT_NAME}"

# Confirm that it has been successfully moved with the right permissions
#Sls -l /etc/docker/certs

echo "Moved private ${COMPONENT_NAME}.key into /etc/docker/certs"
```

#### For [[Server/Rust Bridge\|Rust Bridge]] : `rust_bridge_csr.sh`

```shell
#!/bin/sh

# rust_bridge_csr.sh: Generate Private Key and CSR for the Kafka queue after middleware:

# Replace these variables according to your component and environment
COMPONENT_NAME="rust_bridge"
DOMAIN="capstonepams@gmail.com"
IP_ADDRESS="192.168.252.121"  # Use the actual IP address
OUTPUT_DIR="./"   # output in the same working directory as this script

# Create an OpenSSL config file for CSR generation with SANs
cat > ${OUTPUT_DIR}${COMPONENT_NAME}_openssl.cnf <<EOF
[ req ]
default_bits = 2048
prompt = no
default_md = sha256
req_extensions = req_ext
distinguished_name = dn

[ dn ]
C=SG
ST=Singapore
L=Singapore
O=PAMS
OU=MQTTBroker
CN=${DOMAIN}

[ req_ext ]
subjectAltName = @alt_names

[ alt_names ]
DNS.1 = ${DOMAIN}
IP.1 = ${IP_ADDRESS}
EOF

# Generate a Private Key
openssl genrsa -out ${OUTPUT_DIR}${COMPONENT_NAME}.key 2048

# Move the rust_bridge.key (private key) to /etc/docker/certs
# Because sudo, will need to enter fill in password for current user.
echo "Copying private ${COMPONENT_NAME}.key into /etc/docker/certs"
sudo cp ${OUTPUT_DIR}${COMPONENT_NAME}.key /etc/docker/certs/${COMPONENT_NAME}.key
sudo chmod 600 /etc/docker/certs/${COMPONENT_NAME}.key
echo ===========

# Generate a CSR using the OpenSSL config file
openssl req -new -key ${OUTPUT_DIR}${COMPONENT_NAME}.key -out ${OUTPUT_DIR}${COMPONENT_NAME}.csr -config ${OUTPUT_DIR}${COMPONENT_NAME}_openssl.cnf -extensions req_ext

# move private key generated into mqtt_broker folder
sudo rm -rf /home/klass/Desktop/PAMS-Middleware/CA/${COMPONENT_NAME}
sudo mkdir -p /home/klass/Desktop/PAMS-Middleware/CA/${COMPONENT_NAME}
sudo mv ${OUTPUT_DIR}${COMPONENT_NAME}.key /home/klass/Desktop/PAMS-Middleware/CA/${COMPONENT_NAME}

# Set permissions for the generated file
sudo chmod 444 /home/klass/Desktop/PAMS-Middleware/CA/${COMPONENT_NAME}/${COMPONENT_NAME}.key

echo "Generated ${COMPONENT_NAME}.key and ${COMPONENT_NAME}.csr in ${OUTPUT_DIR}"

# Confirm that it has been successfully moved with the right permissions
#ls -l /etc/docker/certs
```

#### For [[Apache Kafka\|Apache Kafka]] : `kafka1_csr.sh`

```shell
#!/bin/sh

# kafka1_csr.sh: Generate Private Key and CSR for the Kafka queue after middleware:

# Replace these variables according to your component and environment
COMPONENT_NAME="kafka1"
DOMAIN="capstonepams@gmail.com"
IP_ADDRESS="192.168.252.121"  # Use the actual IP address
OUTPUT_DIR="./"   # output in the same working directory as this script

# Create an OpenSSL config file for CSR generation with SANs
cat > ${OUTPUT_DIR}${COMPONENT_NAME}_openssl.cnf <<EOF
[ req ]
default_bits = 2048
prompt = no
default_md = sha256
req_extensions = req_ext
distinguished_name = dn

[ dn ]
C=SG
ST=Singapore
L=Singapore
O=PAMS
OU=MQTTBroker
CN=${DOMAIN}

[ req_ext ]
subjectAltName = @alt_names

[ alt_names ]
DNS.1 = ${DOMAIN}
IP.1 = ${IP_ADDRESS}
EOF

# Generate a Private Key
openssl genrsa -out ${OUTPUT_DIR}${COMPONENT_NAME}.key 2048

# Move the kafka1.key (private key) to /etc/docker/certs
# Because sudo, will need to enter fill in password for current user.
echo "Copying private ${COMPONENT_NAME}.key into /etc/docker/certs"
sudo cp ${OUTPUT_DIR}${COMPONENT_NAME}.key /etc/docker/certs/${COMPONENT_NAME}.key
sudo chmod 600 /etc/docker/certs/${COMPONENT_NAME}.key
echo ===========

# Generate a CSR using the OpenSSL config file
openssl req -new -key ${OUTPUT_DIR}${COMPONENT_NAME}.key -out ${OUTPUT_DIR}${COMPONENT_NAME}.csr -config ${OUTPUT_DIR}${COMPONENT_NAME}_openssl.cnf -extensions req_ext

# move private key generated into mqtt_broker folder
sudo rm -rf /home/klass/Desktop/PAMS-Middleware/CA/${COMPONENT_NAME}
sudo mkdir -p /home/klass/Desktop/PAMS-Middleware/CA/${COMPONENT_NAME}
sudo mv ${OUTPUT_DIR}${COMPONENT_NAME}.key /home/klass/Desktop/PAMS-Middleware/CA/${COMPONENT_NAME}

# Set permissions for the generated file
sudo chmod 444 /home/klass/Desktop/PAMS-Middleware/CA/${COMPONENT_NAME}/${COMPONENT_NAME}.key

echo "Generated ${COMPONENT_NAME}.key and ${COMPONENT_NAME}.csr in ${OUTPUT_DIR}"

# Confirm that it has been successfully moved with the right permissions
#ls -l /etc/docker/certs
```

### Scripting : Signing Certificate Signing Request (CSR) and Transfer Certificate (CA Side)

> [!attention]
> Fill in the fields accordingly to your system's instance.

> [!note]
> After running these scripts, the signed certificate (`.pem`) files will be present in the created subdirectories for the respective components.

- `sign_<component>_csr.sh` : To sign the CSR and generate the signed certificate, and put it into the component's folder inside the `CA` directory.
- `sign_<component>.sh` : Executes the signing process inside the `rootCA` 's container

#### For [[Server/MQTT Broker\|MQTT Broker]]

`sign_mqtt_broker_csr.sh`

```shell
#!/bin/sh

# Configuration variables
COMPONENT_NAME="mqtt_broker"
SIGN_SCRIPT="./sign_mqtt_broker.sh" # Path of local script that will run the sign commands inside rootCA container
CA_SIGN_SCRIPT="/opt/rootca/sign_mqtt_broker.sh" # Temporary path inside rootCA container to script that runs the sign commands 
CA_CONTAINER_NAME="ca-rootca-1" # Name of your CA container
HOST_CSR_PATH="./mqtt_broker.csr" # Path to the CSR on the host
HOST_CERT_PATH="./mqtt_broker.pem" # Path to save the signed certificate on the host
CONTAINER_CSR_PATH="/opt/rootca/input/mqtt_broker.csr" # Temporary path in the container for the CSR
CONTAINER_CERT_PATH="/opt/rootca/output/mqtt_broker.pem" # Temporary path in the container for the signed certificate


# Copy CSR to the CA container
docker cp "$HOST_CSR_PATH" "$CA_CONTAINER_NAME":"$CONTAINER_CSR_PATH"

# load sign.sh script into rootca
docker cp "$SIGN_SCRIPT" "$CA_CONTAINER_NAME":"$CA_SIGN_SCRIPT"

# run sign.sh in the container to sign the CSR inside the rootCA container
docker exec -it $CA_CONTAINER_NAME /bin/sh "$CA_SIGN_SCRIPT"

# Copy the signed certificate back to the host
docker cp "$CA_CONTAINER_NAME":"$CONTAINER_CERT_PATH" "$HOST_CERT_PATH"

# Clean up CSR and signed certificate inside the container
docker exec "$CA_CONTAINER_NAME" rm "$CONTAINER_CSR_PATH" "$CONTAINER_CERT_PATH"

# Clean up sign.sh script inside the container
docker exec "$CA_CONTAINER_NAME" rm "$CA_SIGN_SCRIPT"

echo "Signed certificate has been generated and saved to: $HOST_CERT_PATH"

# Move the signed certificate mqtt_broker.pem to /etc/docker/certs
# Because sudo, will need to enter fill in password for current user.
sudo cp ${COMPONENT_NAME}.pem /etc/docker/certs
sudo chmod 600 /etc/docker/certs/${COMPONENT_NAME}.pem

# move the ${COMPOENENT_NAME}.pem file inti CA/{COMPONENT_NAME} folder
sudo mv ${COMPONENT_NAME}.pem /home/klass/Desktop/PAMS-Middleware/CA/${COMPONENT_NAME}

# Confirm that it has been successfully moved with the right permissions
#ls -l /etc/docker/certs

echo "Moved signed certificate ${COMPONENT_NAME}.pem into /etc/docker/certs"
```

`sign_mqtt_broker.sh`

```shell
#!/bin/sh

# Configuration variables
CONTAINER_CSR_PATH="./input/mqtt_broker.csr" # Temporary path in the container for the CSR
CONTAINER_CERT_PATH="./output/mqtt_broker.pem" # Temporary path in the container for the signed certificate
CONTAINER_CA_CERT_PATH="./local/rootCA.pem" # Path to the CA certificate in the container
CONTAINER_CA_KEY_PATH="./local/rootCA.key" # Path to the CA private key in the container
DAYS_VALID=365 # Number of days the certificate should be valid

echo "========================"

echo "You are now signing for $CONTAINER_CSR_PATH"

# Sign the CSR inside the CA container
openssl x509 -req -in "$CONTAINER_CSR_PATH" -CA "$CONTAINER_CA_CERT_PATH" -CAkey "$CONTAINER_CA_KEY_PATH" -CAcreateserial -out "$CONTAINER_CERT_PATH" -days "$DAYS_VALID" -sha256
```

#### For [[Server/Rust Bridge\|Rust Bridge]]

`sign_rust_bridge_csr.sh`

```shell
#!/bin/sh

# Configuration variables
COMPONENT_NAME="rust_bridge"
SIGN_SCRIPT="./sign_rust_bridge.sh" # Path of local script that will run the sign commands inside rootCA container
CA_SIGN_SCRIPT="/opt/rootca/sign_rust_bridge.sh" # Temporary path inside rootCA container to script that runs the sign commands 
CA_CONTAINER_NAME="ca-rootca-1" # Name of your CA container
HOST_CSR_PATH="./rust_bridge.csr" # Path to the CSR on the host
HOST_CERT_PATH="./rust_bridge.pem" # Path to save the signed certificate on the host
CONTAINER_CSR_PATH="/opt/rootca/input/rust_bridge.csr" # Temporary path in the container for the CSR
CONTAINER_CERT_PATH="/opt/rootca/output/rust_bridge.pem" # Temporary path in the container for the signed certificate

# Copy CSR to the CA container
docker cp "$HOST_CSR_PATH" "$CA_CONTAINER_NAME":"$CONTAINER_CSR_PATH"

# load sign.sh script into rootca
docker cp "$SIGN_SCRIPT" "$CA_CONTAINER_NAME":"$CA_SIGN_SCRIPT"

# run sign.sh in the container to sign the CSR inside the rootCA container
docker exec -it $CA_CONTAINER_NAME /bin/sh "$CA_SIGN_SCRIPT"

# Copy the signed certificate back to the host
docker cp "$CA_CONTAINER_NAME":"$CONTAINER_CERT_PATH" "$HOST_CERT_PATH"

# Clean up CSR and signed certificate inside the container
docker exec "$CA_CONTAINER_NAME" rm "$CONTAINER_CSR_PATH" "$CONTAINER_CERT_PATH"

# Clean up sign.sh script inside the container
docker exec "$CA_CONTAINER_NAME" rm "$CA_SIGN_SCRIPT"

echo "Signed certificate has been generated and saved to: $HOST_CERT_PATH"

# Move the signed certificate rust_bridge.pem to /etc/docker/certs
# Because sudo, will need to enter fill in password for current user.
sudo cp ${COMPONENT_NAME}.pem /etc/docker/certs
sudo chmod 600 /etc/docker/certs/${COMPONENT_NAME}.pem

# move the ${COMPOENENT_NAME}.pem file inti CA/{COMPONENT_NAME} folder
sudo mv ${COMPONENT_NAME}.pem /home/klass/Desktop/PAMS-Middleware/CA/${COMPONENT_NAME}

# Confirm that it has been successfully moved with the right permissions
#ls -l /etc/docker/certs

echo "Moved signed certificate ${COMPONENT_NAME}.pem into /etc/docker/certs"
```

`sign_rust_bridge.sh`

```shell
#!/bin/sh

# Configuration variables
CONTAINER_CSR_PATH="./input/rust_bridge.csr" # Temporary path in the container for the CSR
CONTAINER_CERT_PATH="./output/rust_bridge.pem" # Temporary path in the container for the signed certificate
CONTAINER_CA_CERT_PATH="./local/rootCA.pem" # Path to the CA certificate in the container
CONTAINER_CA_KEY_PATH="./local/rootCA.key" # Path to the CA private key in the container
DAYS_VALID=365 # Number of days the certificate should be valid

echo "========================"

echo "You are now signing for $CONTAINER_CSR_PATH"

# Sign the CSR inside the CA container
openssl x509 -req -in "$CONTAINER_CSR_PATH" -CA "$CONTAINER_CA_CERT_PATH" -CAkey "$CONTAINER_CA_KEY_PATH" -CAcreateserial -out "$CONTAINER_CERT_PATH" -days "$DAYS_VALID" -sha256
```

#### For [[Apache Kafka\|Apache Kafka]]

`sign_kafka1_csr.sh`

```shell
#!/bin/sh

# Configuration variables
COMPONENT_NAME="kafka1"
SIGN_SCRIPT="./sign_kafka1.sh" # Path of local script that will run the sign commands inside rootCA container
CA_SIGN_SCRIPT="/opt/rootca/sign_kafka1.sh" # Temporary path inside rootCA container to script that runs the sign commands 
CA_CONTAINER_NAME="ca-rootca-1" # Name of your CA container
HOST_CSR_PATH="./kafka1.csr" # Path to the CSR on the host
HOST_CERT_PATH="./kafka1.pem" # Path to save the signed certificate on the host
CONTAINER_CSR_PATH="/opt/rootca/input/kafka1.csr" # Temporary path in the container for the CSR
CONTAINER_CERT_PATH="/opt/rootca/output/kafka1.pem" # Temporary path in the container for the signed certificate

# Copy CSR to the CA container
docker cp "$HOST_CSR_PATH" "$CA_CONTAINER_NAME":"$CONTAINER_CSR_PATH"

# load sign.sh script into rootca
docker cp "$SIGN_SCRIPT" "$CA_CONTAINER_NAME":"$CA_SIGN_SCRIPT"

# run sign.sh in the container to sign the CSR inside the rootCA container
docker exec -it $CA_CONTAINER_NAME /bin/sh "$CA_SIGN_SCRIPT"

# Copy the signed certificate back to the host
docker cp "$CA_CONTAINER_NAME":"$CONTAINER_CERT_PATH" "$HOST_CERT_PATH"

# Clean up CSR and signed certificate inside the container
docker exec "$CA_CONTAINER_NAME" rm "$CONTAINER_CSR_PATH" "$CONTAINER_CERT_PATH"

# Clean up sign.sh script inside the container
docker exec "$CA_CONTAINER_NAME" rm "$CA_SIGN_SCRIPT"

echo "Signed certificate has been generated and saved to: $HOST_CERT_PATH"

# Move the signed certificate kafka1.pem to /etc/docker/certs
# Because sudo, will need to enter fill in password for current user.
sudo cp ${COMPONENT_NAME}.pem /etc/docker/certs
sudo chmod 600 /etc/docker/certs/${COMPONENT_NAME}.pem

# move the ${COMPOENENT_NAME}.pem file inti CA/{COMPONENT_NAME} folder
sudo mv ${COMPONENT_NAME}.pem /home/klass/Desktop/PAMS-Middleware/CA/${COMPONENT_NAME}

# Confirm that it has been successfully moved with the right permissions
#ls -l /etc/docker/certs

echo "Moved signed certificate ${COMPONENT_NAME}.pem into /etc/docker/certs"
```

`sign_kafka1.sh`

```shell
#!/bin/sh

# Configuration variables
CONTAINER_CSR_PATH="./input/kafka1.csr" # Temporary path in the container for the CSR
CONTAINER_CERT_PATH="./output/kafka1.pem" # Temporary path in the container for the signed certificate
CONTAINER_CA_CERT_PATH="./local/rootCA.pem" # Path to the CA certificate in the container
CONTAINER_CA_KEY_PATH="./local/rootCA.key" # Path to the CA private key in the container
DAYS_VALID=365 # Number of days the certificate should be valid

echo "========================"

echo "You are now signing for $CONTAINER_CSR_PATH"

# Sign the CSR inside the CA container
openssl x509 -req -in "$CONTAINER_CSR_PATH" -CA "$CONTAINER_CA_CERT_PATH" -CAkey "$CONTAINER_CA_KEY_PATH" -CAcreateserial -out "$CONTAINER_CERT_PATH" -days "$DAYS_VALID" -sha256
```





## Scripting 2 : Compilation scripts

> [!note]
> This script runs all the above scripts. Make sure you have `sudo` privileges.

### Scripting : Overarching script to create security components

`ca.sh`

```shell
#!/bin/sh

# run this script to create root CA, create CSR, keys for mqtt, kafka and sign the csr to get the .pem files

# Before we run each section, we make the relevant scripts for that section executable.

# start at CA directory
chmod +x run.sh
chmod +x init_ca.sh
chmod +x steup_ca.sh

./setup_ca.sh

# Go into CSR directory and run CSR generation and signing
cd CSR

# files that generate private key for each component & their CSR 
chmod +x mqtt_broker_csr.sh
chmod +x rust_bridge_csr.sh
chmod +x kafka1_csr.sh

./mqtt_broker_csr.sh
./kafka1_csr.sh
./rust_bridge_csr.sh

# files that sign csr
chmod +x sign_mqtt_broker_csr.sh
chmod +x sign_mqtt_broker.sh
chmod +x sign_rust_bridge_csr.sh
chmod +x sign_rust_bridge.sh
chmod +x sign_kafka1_csr.sh
chmod +x sign_kafka1.sh

./sign_mqtt_broker_csr.sh
./sign_kafka1_csr.sh
./sign_rust_bridge_csr.sh
```

### RUN SCRIPT

After checking that all scripts above are in place, and that all fields required are changed according to your instance, you can run:

```shell
chmod +x ca.sh
./ca.sh
```


## Component Configurations

> [!note]
> Now that each component has been given the required keys and certificates, it is time to configure the components to to use them.

Each component has to update:
- Docker compose file (but the one above should contain all things we managed to do)
- Its own configuration file, if any
- Its code, if relevant

### For [[Anchors\|Anchors]]

 

### For [[Server/MQTT Broker\|MQTT Broker]]

#### Configuration

The configuration of our MQTT broker, NanoMQ, is in a configuration file `nanomq.conf`. After running the above scripts, you need to configure the broker to be able to use the key and certificates given to it.

```conf
listeners.ssl {
 enable = true
 bind = "0.0.0.0:8883"
 keyfile = "/certs/mqtt_broker.key"
 certfile = "/certs/mqtt_broker.pem"
 cacertfile = "/certs/rootCA.pem"
 verify_peer = true
 fail_if_no_peer_cert = true
}
```

```conf
bridges.mqtt{
 nodes = [
  {
   name = emqx
   enable = false
   connector {
    server = "mqtt-tcp://localhost:8883"
    proto_ver = 4
    username = user
    password = pwd
    clean_start = true
    keepalive = 60s
   }
   forwards = ["topic1/#", "topic2/#"]
   quic_keepalive = 120s
   quic_idle_timeout = 120s
   quic_discon_timeout = 20s
   quic_handshake_timeout = 60s
   hybrid_bridging = false
   congestion_control = cubic
   subscription = [
    {
     topic = "cmd/topic1"
     qos = 1
    },
    {
     topic = "cmd/topic2"
     qos = 2
    }
   ]
   parallel = 2
   ssl {
    enable = true
    keyfile = "/certs/mqtt_broker.key"
    certfile = "/certs/mqtt_broker.pem"
    cacertfile = "/certs/rootCA.pem"
   }
  }
```

Just make sure the paths to the key, certificate, and root CA certificate is indicated in the configuration file correctly.

### For [[Server/Rust Bridge\|Rust Bridge]]

[Code is in our GitHub repository](https://github.com/S32-PAMS/PAMS-Middleware/blob/security/Kafka/producer/src/main.rs), in `security` branch: `Kafka/producer/src/main.rs`. This is different from the `main.rs` that does not use TLS/SSL, so take note of it.

Ensure the `Cargo.toml` file in `Kafka/producer/Cargo.toml` is as follows:

```toml
[package]
name = "producer"
version = "0.1.0"
edition = "2021"

# See more keys and their definitions at https://doc.rust-lang.org/cargo/reference/manifest.html

[dependencies]
kafka = "0.10.0"
reqwest = { version = "0.11.22", features = ["json"] }
serde = { version = "1.0.190", features = ["derive"] }
serde_json = "1.0.108"
urlencoding = "2.1.3"
rumqttc = "0.10"
rdkafka = { version = "0.26", features = ["tokio", "ssl"] }
tokio = { version = "1", features = ["full"] }
protobuf-codegen = "3.3.0"
protobuf = "3.3.0"
bytes = "1.5.0"
```

### For [[Apache Kafka\|Apache Kafka]]



## Anchor scripts

> [!note]
> These are the scripts to run in order to add a recognised anchor to your existing system. These are in `CA/CSR/` directory.

### Scripting : Generate key and CSR

`anchor_csr.sh`

```shell
#!/bin/sh

# anchor_csr.sh: Generate Private Key and CSR for each anchor to be added into the system:
# and also sign the CSR in here so that we can pass in the name argument

# take in the anchor's unique name/id
read -p "Enter the unique name or id for this anchor: " name

# Replace these variables according to your component and environment
COMPONENT_NAME="anchor_$name"
DOMAIN="capstonepams@gmail.com"
IP_ADDRESS="192.168.223.121"  # Use the actual IP address
OUTPUT_DIR="./"   # output in the same working directory as this script


# Create an OpenSSL config file for CSR generation with SANs
cat > ${OUTPUT_DIR}${COMPONENT_NAME}_openssl.cnf <<EOF
[ req ]
default_bits = 2048
prompt = no
default_md = sha256
req_extensions = req_ext
distinguished_name = dn

[ dn ]
C=SG
ST=Singapore
L=Singapore
O=PAMS
OU=${COMPONENT_NAME}
CN=${DOMAIN}

[ req_ext ]
subjectAltName = @alt_names

[ alt_names ]
DNS.1 = ${DOMAIN}
IP.1 = ${IP_ADDRESS}
IP.2 = 192.168.252.121
IP.3 = 192.168.35.121
EOF


# Generate a Private Key
openssl genrsa -out ${OUTPUT_DIR}${COMPONENT_NAME}.key 2048
# convert key into .der
openssl rsa -outform der -in ${COMPONENT_NAME}.key -out ${COMPONENT_NAME}_key.der
# cconvert key.der into key.c
xxd -i ${COMPONENT_NAME}_key.der > ${COMPONENT_NAME}_key.c

# Move the anchor.key (private key) to /etc/docker/certs
# Because sudo, will need to enter fill in password for current user.
echo "Copying private ${COMPONENT_NAME}.key into /etc/docker/certs"
sudo cp ${OUTPUT_DIR}${COMPONENT_NAME}.key /etc/docker/certs/${COMPONENT_NAME}.key
sudo chmod 600 /etc/docker/certs/${COMPONENT_NAME}.key
echo ===========


# Generate a CSR using the OpenSSL config file
openssl req -new -key ${OUTPUT_DIR}${COMPONENT_NAME}.key -out ${OUTPUT_DIR}${COMPONENT_NAME}.csr -config ${OUTPUT_DIR}${COMPONENT_NAME}_openssl.cnf -extensions req_ext



echo "Generated ${COMPONENT_NAME}.key and ${COMPONENT_NAME}.csr in ${OUTPUT_DIR}"

# Confirm that it has been successfully moved with the right permissions
#ls -l /etc/docker/certs

# sign_anchor_csr part below here, put in same bash script so that it can use name argument
# Configuration variables
SIGN_SCRIPT="./sign_anchor.sh" # Path of local script that will run the sign commands inside rootCA container
CA_SIGN_SCRIPT="/opt/rootca/sign_anchor.sh" # Temporary path inside rootCA container to script that runs the sign commands 
CA_CONTAINER_NAME="ca-rootca-1" # Name of your CA container
HOST_CSR_PATH="./${COMPONENT_NAME}.csr" # Path to the CSR on the host
HOST_CERT_PATH="./${COMPONENT_NAME}.pem" # Path to save the signed certificate on the host
CONTAINER_CSR_PATH="/opt/rootca/input/${COMPONENT_NAME}.csr" # Temporary path in the container for the CSR
CONTAINER_CERT_PATH="/opt/rootca/output/${COMPONENT_NAME}.pem" # Temporary path in the container for the signed certificate


# Copy CSR to the CA container
docker cp "$HOST_CSR_PATH" "$CA_CONTAINER_NAME":"$CONTAINER_CSR_PATH"

# load sign.sh script into rootca
docker cp "$SIGN_SCRIPT" "$CA_CONTAINER_NAME":"$CA_SIGN_SCRIPT"

# run sign.sh in the container to sign the CSR inside the rootCA container
docker exec -it $CA_CONTAINER_NAME /bin/sh "$CA_SIGN_SCRIPT" "$COMPONENT_NAME"

# Copy the signed certificate back to the host
docker cp "$CA_CONTAINER_NAME":"$CONTAINER_CERT_PATH" "$HOST_CERT_PATH"

# Clean up CSR and signed certificate inside the container
docker exec "$CA_CONTAINER_NAME" rm "$CONTAINER_CSR_PATH" "$CONTAINER_CERT_PATH"

# Clean up sign.sh script inside the container
docker exec "$CA_CONTAINER_NAME" rm "$CA_SIGN_SCRIPT"


echo "Signed certificate has been generated and saved to: $HOST_CERT_PATH"

# Convert Certificates to DER Format
openssl x509 -outform der -in ${COMPONENT_NAME}.pem -out ${COMPONENT_NAME}_cert.der


# Convert DER Files to C Byte Arrays
xxd -i ${COMPONENT_NAME}_cert.der > ${COMPONENT_NAME}_cert.c
# move private key generated into anchor folder
sudo rm -rf /home/klass/Desktop/PAMS-Middleware/CA/${COMPONENT_NAME}
sudo mkdir -p /home/klass/Desktop/PAMS-Middleware/CA/${COMPONENT_NAME}
sudo mv ${OUTPUT_DIR}${COMPONENT_NAME}.key /home/klass/Desktop/PAMS-Middleware/CA/${COMPONENT_NAME}

# Set permissions for the generated file
sudo chmod 444 /home/klass/Desktop/PAMS-Middleware/CA/${COMPONENT_NAME}/${COMPONENT_NAME}.key
echo "Moved signed certificate ${COMPONENT_NAME}.key into home/klass/Desktop/PAMS-Middleware/CA/${COMPONENT_NAME}"

# Move the signed certificate anchor.pem to /etc/docker/certs
# Because sudo, will need to enter fill in password for current user.
sudo cp ${COMPONENT_NAME}.pem /etc/docker/certs
sudo chmod 600 /etc/docker/certs/${COMPONENT_NAME}.pem
echo "Moved signed certificate ${COMPONENT_NAME}.pem into /etc/docker/certs"

# move the ${COMPOENENT_NAME}_key and ${COMPOENENT_NAME}_cert .pem, .der and .c files into CA/{COMPONENT_NAME} folder
sudo mv ${COMPONENT_NAME}.pem /home/klass/Desktop/PAMS-Middleware/CA/${COMPONENT_NAME}
echo "Moved signed certificate ${COMPONENT_NAME}.pem into home/klass/Desktop/PAMS-Middleware/CA/${COMPONENT_NAME}"

sudo mv ${COMPONENT_NAME}_key.der /home/klass/Desktop/PAMS-Middleware/CA/${COMPONENT_NAME}
echo "Moved signed certificate ${COMPONENT_NAME}_key.der into home/klass/Desktop/PAMS-Middleware/CA/${COMPONENT_NAME}"

sudo mv ${COMPONENT_NAME}_cert.der /home/klass/Desktop/PAMS-Middleware/CA/${COMPONENT_NAME}
echo "Moved signed certificate ${COMPONENT_NAME}_cert.der into home/klass/Desktop/PAMS-Middleware/CA/${COMPONENT_NAME}"

sudo mv ${COMPONENT_NAME}_key.c /home/klass/Desktop/PAMS-Middleware/CA/${COMPONENT_NAME}
echo "Moved signed certificate ${COMPONENT_NAME}_key.c into home/klass/Desktop/PAMS-Middleware/CA/${COMPONENT_NAME}"

sudo mv ${COMPONENT_NAME}_cert.c /home/klass/Desktop/PAMS-Middleware/CA/${COMPONENT_NAME}
echo "Moved signed certificate ${COMPONENT_NAME}_cert.c into home/klass/Desktop/PAMS-Middleware/CA/${COMPONENT_NAME}"
```

### Scripting : Signing CSR

`sign_anchor.sh`

```shell
#!/bin/sh

# Take in the whole anchor component name
COMPONENT_NAME="$1"

# Configuration variables
CONTAINER_CSR_PATH="./input/${COMPONENT_NAME}.csr" # Temporary path in the container for the CSR
CONTAINER_CERT_PATH="./output/${COMPONENT_NAME}.pem" # Temporary path in the container for the signed certificate
CONTAINER_CA_CERT_PATH="./local/rootCA.pem" # Path to the CA certificate in the container
CONTAINER_CA_KEY_PATH="./local/rootCA.key" # Path to the CA private key in the container
DAYS_VALID=365 # Number of days the certificate should be valid

echo "========================"

echo "You are now signing for $CONTAINER_CSR_PATH"

# Sign the CSR inside the CA container
openssl x509 -req -in "$CONTAINER_CSR_PATH" -CA "$CONTAINER_CA_CERT_PATH" -CAkey "$CONTAINER_CA_KEY_PATH" -CAcreateserial -out "$CONTAINER_CERT_PATH" -days "$DAYS_VALID" -sha256
```

### RUNNING

This is only run when you are **adding a new anchor to the system**.

```shell
chmod +x anchor_csr.sh
chmod +x sign_anchor.sh
./anchor_csr.sh
```

# Remote (AWS) PKI setup
---
{"dg-publish":true,"permalink":"/guides/for-running-server/","tags":["run","middleware","software"],"noteIcon":""}
---

> [!abstract] How to set up the backend for the prototype
> This page contains instructions on how to get our prototype to run.

# MQTT, Rust and Kafka broker setup

```bash 
cd PAMS-Middleware/Rust
docker compose up
```

After running the docker containers, check the logs to make sure all the the containers are running properly if broker and broker2 logs are not slowing down, refer to  [[Guides/For Running Server#Troubleshooting\|#Troubleshooting]] under section 1.1

## Set up the Kafka brokers

Create topics to be linked to every other component.

Ensure the CLI returns "Create topic <>" for each topic created. If this fails, refer to  [[Guides/For Running Server#Troubleshooting\|#Troubleshooting]]  under section 1.2

```bash
docker exec -ti broker bash
>> kafka-topics --create --topic packet_data --if-not-exists --bootstrap-server broker:10920
>> kafka-topics --create --topic packet_alive --if-not-exists --bootstrap-server broker:10920
>> kafka-topics --create --topic anchor_cood --if-not-exists --bootstrap-server broker:10920

docker exec -ti broker2 bash
>> kafka-topics --create --topic out_data --if-not-exists --bootstrap-server broker2:9095
>> kafka-topics --create --topic out_alive --if-not-exists --bootstrap-server broker2:9095
```

## Connect Kafka brokers to rest of system

Create a bridge that connects to other components

```bash
docker create network sysbridge
docker network connect sysbridge broker
docker network connect sysbridge broker2
```

## Start Rust bridge

[[Server/Rust Bridge\|Rust Bridge]] sends information to [[Server/Apache Kafka\|Apache Kafka]] from [[Server/MQTT Broker\|MQTT Broker]]. If no messages are appearing, refer to [[Guides/For Running Server#Troubleshooting\|#Troubleshooting]] under 1.3

First, add your network's IP address to `PAMS-Middleware/Rust/producer/src/main.rs` as the value for the variable `mqtt_broker`.

Then start up the producer code.

```bash
docker exec -ti rustbridge bash
>> cd workspaces/Rust
>> cargo run -p producer
```

# Setup Flink, Node bridge and Frontend
### Create the Flink bridge

#### Install Helm, Kubectl and Kind on your device.

```bash
cd PAMS-Software
```

#### Create the Kubectl cluster

```bash
{ cat | kind create cluster --name pyflink-test --config -; } << EOF
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
  - role: worker
EOF
```

#### Install the flink-kubernetes operator

1. deploy the certificate manager (for the operator's webhook)
2. add the helm repository for the operator
3. install the operator using the provided helm chart

```bash
> kubectl create -f https://github.com/cert-manager/cert-manager/releases/download/v1.8.2/cert-manager.yaml
> `helm repo add flink-operator-repo https://downloads.apache.org/flink/flink-kubernetes-operator-1.7.0/
> helm install flink-kubernetes-operator flink-operator-repo/flink-kubernetes-operator
```

Run `kubectl get pods` and wait for 2/2 containers to run

#### Build the existing docker file and upload image to kind

```bash
> docker build -f Dockerfile . -t s32pams/pyflink-runstream:latest
> kind load docker-image s32pams/pyflink-runstream:latest --name pyflink-test
> kubectl create -f pyflink-runstream.yaml
```

#### Run the PSQL_DB connection

```bash
cd PSQL_DB
docker compose build --no-cache
docker compse up
```

#### Run the Frontend

If the frontend cannot connect to the posgres db, refer to [[Guides/For Running Server#Troubleshooting\|#Troubleshooting]] under 1.4

```bash
cd PAMS-FrontEnd/frontend
npm i
npm run dev
```

#### Run the nodejs bridge

```bash
cd PAMS-Software/nodeserver
npm i
node app.js
```

#### Run the Camera Servers

``` bash
cd Cam_Server
npm i
node server.js
```

---
# Troubleshooting

## 1.1 broker/broker2 infinite logs
- shut down just one broker. then start the broker back up. 
- If the issue persists, shut down zookeeper with the 2 brokers before starting all 3 services back up.
- The issue occurs since broker and broker2 are both fighting for similar resources in zookeeper

## 1.2 Kafka topics are not created
- find the broker the topic was wrongly created on and run a delete kafka topic command. retry the create.

```
>> kafka-topics --delete --topic topic_name --bootstrap-server brooker_name
```

- If the situation persists, or you are unable to find the broker the topic was initially created on, delete zookeeper and both brokers and start the process again.

## 1.3 No messages at Rust bridge
- Check your anchor's IP is allocated correctly
- Check your main.rs file's IP is allocated correctly
- Conduct a sanity check to ensure your network is ok and information can be sent to the MQTT broker.
- If all else fails, reset the Rust bridge with a docker compose down, deleting any relevant volumes.

## 1.4 Posgres, frontend cannot connect
- This usually occurs when posgres is started again after an improper shutdown. To resolve this, you will need to restart the posgres instance on the device.
```bash
cd PSQL_DB
docker compose down
# remove psql volume
docker compose up
```

- If this issue persists, delete the larger posgres server on your machine. 

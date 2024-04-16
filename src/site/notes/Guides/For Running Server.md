---
{"dg-publish":true,"permalink":"/Guides/For Running Server/","tags":["run","middleware","software"],"noteIcon":""}
---

> [!abstract] How to set up the backend for the prototype
> This page contains instructions on how to get our prototype to run.

# MQTT, Rust and Kafka broker setup

These are [[Architecture#Middleware components\|Architecture#Middleware components]].

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

## Sanity Check for Rust bridge data in kafka

This section assumes KafkaCat has been downloaded as per [[Server/Server initial setup#KafkaCat installation\|Server initial setup#KafkaCat installation]]. This section can be run in any shell on the machine.

To ensure the Kafka topics have been successfully created.

```bash
kafkacat -b localhost:9096 -L
```

The shell should print a list of all the topics created. 

To check if data is being sent to the correct Kafka topics.

```bash
kafkacat -b localhost:9096 -t packet_data
kafkacat -b localhost:9096 -t packet_alive
```

The shell will print all data consumed from `packet_data` and `packet_alive`.

# Setup Flink, Node bridge and Frontend
### Create the Flink bridge

#### Install Helm, Kubectl and Kind on your device.

See [[Server/Server initial setup#Kubectl\|Server initial setup#Kubectl]] for more.

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

1. Deploy the certificate manager (for the operator's webhook)
2. Add the helm repository for the operator
3. Install the operator using the provided helm chart

```bash
> kubectl create -f https://github.com/cert-manager/cert-manager/releases/download/v1.8.2/cert-manager.yaml
> `helm repo add flink-operator-repo https://downloads.apache.org/flink/flink-kubernetes-operator-1.7.0/
> helm install flink-kubernetes-operator flink-operator-repo/flink-kubernetes-operator
```

Run `kubectl get pods` and wait for 2/2 containers to run.

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
docker compose up
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
cd Cam_Server/cam_files
npm i
node server.js
```

in a separate terminal, run

```bash
# Run FFmpeg to convert RTSP stream to HLS format.
# Replace "192.168.195.105/stream" with your actual RTSP stream URL if different.
ffmpeg -i rtsp://CameraAccountName:CameraAccountPassword@192.168.195.105/stream1 -c:v copy -c:a copy -f hls -hls_time 2 -hls_playlist_type event stream.m3u8 
```

## Sanity Check for PyFlink data in kafka

This section assumes KafkaCat has been downloaded as per Server initial setup. This section can be run in any shell on the machine.

To check if data is being sent to the correct Kafka topics.

```bash
kafkacat -b localhost:9097 -t out_data
kafkacat -b localhost:9097 -t out_alive
```

The shell will print all data consumed from `packet_data` and `packet_alive`.

# Using the Frontend

Assume you are running the web app locally at port 3000, visit `localhost:3000` on your browser.   

You should see a login screen, click on "Sign Up Here" to create an account and start using our web app. ("Sign in with Google" option is provided as a template for future handler to implement if there's a need for it as OAuth was built-in to our web app.)

![signInScreen.png](/img/user/Attachments/frontend-users/signInScreen.png)

After registering, you will be redirected to `localhost:3000/`. 

## Initiate tags

Visit `localhost:3000/map` to initiate the tags. This is where you will monitor the location of the tags.    
  
The image below shows when the tag is detected to be out of its belonged room/area, the area on the map will be turned into red, notifying users. It will also prompt the user to click and view the camera footage.

![outOfRoom.png](/img/user/Attachments/frontend-users/outOfRoom.png)

The image below shows when the tag is detected to be detached, the area on the map will be turned into blue, notifying users. It will also prompt the user to click and view the camera footage.

![detached.png](/img/user/Attachments/frontend-users/detached.png)

## Initiate anchors

Anchors first need to be added via the frontend. visit localhost:3000/anchorsboard and click the + button at the top left corner. Use the popup available to add anchor data. For a more visual explanation, see [[UserManual#Anchorboard\|UserManual#Anchorboard]].

Additionally, the anchor ids and coordinates must be added into the static dictionary in the Flink file for Flink to retrieve information from, specifically into the variable anchorDict in runStream.py. When new anchors are added, the Flink instance should be redeployed to reflect updated changes. More information can be found in [[Server/Apache Flink#Add Anchors\|Apache Flink#Add Anchors]]

## Tagsboard

Visit `localhost:3000/tagsboard`.

The image below shows the page to monitor all the tags. Highlighted box 1 shows an active tag on low battery and is attached to an object. Highighted box 2 shows an active tag which has been detached, notifying the user.

![tagsboard.png](/img/user/Attachments/frontend-users/tagsboard.png)

The user will also be able to view the footage of camera associated to each tags.

![cam_gif.gif](/img/user/Attachments/frontend-users/cam_gif.gif)


> [!note]
> The system admin will need to manually associate each tag to each camera depending on the set up of the environment.

## Anchorboard

The image below shows how a user can add the anchors from the web app. This is crucial because the anchors will serve as beacons, drawing polygons in the physical room and keeping track of the information of the tags. Hence, the system admin will need to manually set known coordinates of the anchors when setting up the PAMS system in a room.

![addanchor.png](/img/user/Attachments/frontend-users/addanchor.png)

The anchor board also serves to alert users when an anchor has been tampered with and is no longer sending or receiving liveness.

![anchordead.png](/img/user/Attachments/frontend-users/anchordead.png)

---
# Troubleshooting

## 1.1 broker/broker2 infinite logs

- Shut down just one broker. then start the broker back up. 
- If the issue persists, shut down zookeeper with the 2 brokers before starting all 3 services back up.
- The issue occurs since broker and broker2 are both fighting for similar resources in zookeeper

## 1.2 Kafka topics are not created

- Find the broker the topic was wrongly created on and run a delete kafka topic command. Fetry the create.

```
>> kafka-topics --delete --topic topic_name --bootstrap-server broker_name
```

- If the situation persists, or you are unable to find the broker the topic was initially created on, delete zookeeper and both brokers and start the process again.

## 1.3 No messages at Rust bridge

- Check your anchor's IP is allocated correctly
- Check your `main.rs` file's IP is allocated correctly
- Conduct a sanity check to ensure your network is ok and information can be sent to the [[Server/MQTT Broker\|MQTT Broker]].
- If all else fails, reset the Rust bridge with a `docker compose down`, deleting any relevant volumes.

## 1.4 Postgres, frontend cannot connect

- This usually occurs when postgres is started again after an improper shutdown. To resolve this, you will need to restart the postgres instance on the device.

```bash
cd PSQL_DB
docker compose down
# remove psql volume
docker compose up
```

- If this issue persists, delete the larger postgres server on your machine. 


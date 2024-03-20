---
{"dg-publish":true,"permalink":"/server/server-initial-setup/","tags":["archi","middleware","software"],"noteIcon":""}
---

> [!abstract] Server setup
> This manual guides the initial setup of the software server for PAMS (Personnel and Asset Monitoring System), which can run both *locally* and *remotely* on a cloud instance. The server hosts various components of the PAMS [[Architecture\|Architecture]], including middleware and software components necessary for tracking and visualising location data of tagged objects.

The Dockerisation of our middleware and software components is what makes this application portable to be served locally or remotely, depending on the choice of the system administrator, and as such this is a necessary step before the set up of any other components.

### Operating system

Start with a Linux-based OS (e.g., Ubuntu, CentOS). Update your system and install the necessary packages.

```bash
sudo apt-get update && sudo apt-get upgrade -y
sudo apt-get install -y curl gcc make build-essential cmake git software-properties-common net-tools
```

#### (Optional) Firewall

Install and configure UFW (Uncomplicated Firewall) to manage network traffic. We did not do this in our prototype, but it is for your consideration.

```bash
sudo apt-get install -y ufw
sudo ufw enable
sudo ufw allow ssh
sudo ufw allow 80
sudo ufw allow 1883
sudo ufw allow 8883
... take note to allow more ports that you setup later
```

These additional steps will improve the initial security posture. These are basic examples, and the system can be complemented by more sophisticated firewall options as you deem fit.

### Rust installation

For the creation of our [[Server/Rust Bridge\|Rust Bridge]], install Rust on your development environment.

```shell
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

This assumes you are following our prototype. You can use any other language for your bridge between the [[Server/MQTT Broker\|MQTT Broker]] and [[Server/Apache Kafka\|Apache Kafka]].

### Protobuf compilation

In our initial message passing phase, messages will be serialised in [Google Protocol Buffers](https://protobuf.dev), which is a language-neutral, platform-neutral and extensible method of serialisation of structured data. To compile `.proto` files, we require that the `protoc` Protobuf compiler is installed in the system. As such, do the following:

```shell
sudo apt install protobuf-compiler -y
```

This is crucial for the following components in our prototype:
- [[Anchors\|Anchors]]
- [[Apache Flink\|Apache Flink]]

#### Creation of standard `.proto` file for messages

> [!note]
> This is placed here as this file, once created, will be the file used in the whole system throughout.

This contains the 2 designed messages which our [[Anchors\|Anchors]] will send, and our [[Apache Flink\|Apache Flink]] will use to deserialise and do calculations from.

`message.proto`

```c
syntax = "proto3";

import "google/protobuf/timestamp.proto";

package message;

message AnchorAlive {
    string anchor_id = 1; // anchor id
    google.protobuf.Timestamp timestamp = 2; // timestamp
}

message AnchorData {
    string anchor_id = 1; // anchor id
    google.protobuf.Timestamp timestamp = 2; // timestamp
    map<string, TagData> tags = 3; // tag data
}

message TagData {
    bool detached = 1; // is tag detached
    bool battery_low = 2; // is battery low
    float distance = 3; // distance
}
```

`timestamp.proto`, extracted from [official protobuf](https://github.com/protocolbuffers/protobuf/blob/main/src/google/protobuf/timestamp.proto)

```c
// Protocol Buffers - Google's data interchange format
// Copyright 2008 Google Inc.  All rights reserved.
// https://developers.google.com/protocol-buffers/
//

syntax = "proto3";

package google.protobuf;

option cc_enable_arenas = true;
option go_package = "google.golang.org/protobuf/types/known/timestamppb";
option java_package = "com.google.protobuf";
option java_outer_classname = "TimestampProto";
option java_multiple_files = true;
option objc_class_prefix = "GPB";
option csharp_namespace = "Google.Protobuf.WellKnownTypes";

message Timestamp {
  int64 seconds = 1;
  int32 nanos = 2;
}
```

#### Compilation for [[Anchors\|Anchors]]

As anchors are ESP32, it should require the [nanopb](https://jpa.kapsi.fi/nanopb/) plugin. First clone the repository:

```shell
 git clone https://github.com/nanopb/nanopb.git
```

Then compile the `.proto` files (in the same directory) with these commands:

```shell
protoc --plugin=protoc-gen-nanopb="$(pwd)/nanopb/generator/protoc-gen-nanopb" --nanopb_out=. message.proto
protoc --plugin=protoc-gen-nanopb="$(pwd)/nanopb/generator/protoc-gen-nanopb" --nanopb_out=. timestamp.proto
```

This generates `message.pb.c`, `message.pb.h`, `timestamp.pb.c` and `timestamp.pb.h` files. These files will be used in the setup of [[Anchors\|Anchors]], so have them ready.

#### Compilation for [[Apache Flink\|Apache Flink]]

```shell
protoc --python_out . message.proto
protoc --python_out . timestamp.proto
```

This generates `message_pb2.py` and `timestamp_pb2.py` files. These files will be used in the setup of [[Apache Flink\|Apache Flink]], so have them ready.

### Docker

> Required for running containerised applications such as the [[Server/MQTT Broker\|MQTT Broker]], [[Server/Rust Bridge\|Rust Bridge]], [[Server/Apache Kafka\|Apache Kafka]], and [[MongoDB\|MongoDB]].
> 
> This section provides step-by-step instructions for installing Docker on an Ubuntu-based system and setting up Docker authentication with GPG credentials. It ensures a secure and optimised Docker environment for PAMS deployment.

#### Installation

This installation requires administrative access (`sudo`) to the system.

To install docker, refer to the [documentation](https://docs.docker.com/engine/install/ubuntu/), but TLDR:

Remove old versions that might be installed on the system:

```bash
for pkg in docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc; do sudo apt-get remove $pkg; done
```

Install packages necessary for adding a new repository over HTTPS:

```bash
sudo apt-get install ca-certificates curl gnupg
```

Add Docker's official GPG Key, to ensure that the software you are installing is authenticated and trustworthy:

```bash
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
```

To install Docker from its official repository, add it to your system's (`apt`) sources list:

```bash
echo \
  "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

Update your package index and install Docker Engine, CLI, and other necessary plugins:

```bash
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

#### Docker authentication setup

For a secure Docker operation, generate a GPG key.

Create a file named `gen_key` with the GPG Key configuration:

```bash
cat >gen_key <<EOF
  %echo Generating a basic OpenPGP key
  Key-Type: DSA
  Key-Length: 1024
  Subkey-Type: ELG-E
  Subkey-Length: 1024
  Name-Real: PAMS
  Name-Email: capstonepams@gmail.com
  Expire-Date: 0
  Passphrase: capstonepams
  %commit
  %echo done
EOF
```

> [!note]
> The values in the fields should be changed to more secure values, according to your application instance. These are the default values we have used for our prototype.

Generate the GPG Key using the configuration:

```bash
gpg --batch --generate-key gen_key
```


Retrieve and note down the generated key ID for later use:

```bash
filename=$(ls -t ~/.gnupg/openpgp-revocs.d | head -1)
f=$(echo $filename | cut -d'.' -f1)
echo $f
```

Initialise the password store using the generated key ID:

```bash
pass init $f
```

After generating the key, remove the `gen_key` file for security:

```bash
rm gen_key
```

You have successfully installed Docker and configured Docker authentication with GPG credentials on your system. This setup ensures a secure environment for deploying and managing Dockerised applications for the PAMS project. Now, we can move on to setting up Docker Compose.

#### Docker Compose Setup

Installation should have been completed above. If not, refer to Docker's documentation. Verify installation by checking its version:

```bash
docker-compose --version
```

##### Create Docker Compose file

Navigate to your project directory and create a file named `docker-compose.yml`. This file will define all the necessary services, networks and volumes for the PAMS architecture.

> [!note]
> For our prototype, we created our `docker-compose.yml` inside our project directory's `Rust` folder. Path is `Rust/docker-compose.yml`.

```yml
version: '3.8'
services:
```

Services will be populated as the other components of the [[Architecture\|Architecture]] are set up.

##### Running Docker Compose

With your `docker-compose.yml` file ready, start your services by running:

```bash
docker-compose up -d
```


- Back to [[README\|README]]
- Go set up [[Server/MQTT Broker\|MQTT Broker]]
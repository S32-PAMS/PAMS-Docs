---
{"dg-publish":true,"permalink":"/Server/Rust Bridge/","tags":["middleware","archi"],"noteIcon":""}
---

> [!abstract] Rust bridge
> MQTT messages pass from the [[Server/MQTT Broker\|MQTT Broker]], but [[Server/Apache Kafka\|Apache Kafka]] cannot consume directly from the [[Server/MQTT Broker\|MQTT Broker]]. As such, we made this bridge to pass the message from the MQTT topics to the Kafka topics. You can implement this in any other development language, but we chose a Rust development image for this purpose.

This assumes [[Server/Server initial setup\|Server initial setup]] is complete.

# Code and configuration

## 1. Setup Rust Project

Create project directory in server.

```shell
mkdir Rust
cd Rust
```

Setup Rust workspace

```shell
cat > Cargo.toml
[workspace]
members = [
    "producer"
]
```

Create producer application. This refers to something that 'produces' inputs for [[Server/Apache Kafka\|Apache Kafka]], hence the name.

```shell
cargo new producer
cd producer
```

To setup the application, make another `Cargo.toml` inside the `producer` application you just created.

### For base implementation, use this `Cargo.toml`:

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
rdkafka = { version = "0.26", features = ["tokio"] }
tokio = { version = "1", features = ["full"] }
protobuf-codegen = "3.3.0"
protobuf = "3.3.0"
bytes = "1.5.0"
```

### For secure implementation, use this `Cargo.toml` instead:

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
bytes = "1.5.0"
```

## 2. Setup a Dev Container

- [Rust image Docker hub](https://hub.docker.com/_/microsoft-devcontainers-rust)

```yml
services:
  rust-log-processing:
    image: mcr.microsoft.com/devcontainers/rust:0-1-bullseye
    volumes:
      - ..:/workspaces:cached
    cap_add:
      - SYS_PTRACE
    security_opt:
      - seccomp:unconfined
    command: /bin/sh -c "while sleep 1000; do :; done"
```

Now open the project in container with your code editor.

```shell
code .
```

## 3. Add Producer Code

Inside `Rust/producer/src` which should be created after creating the `producer` application, create `main.rs`.

> [!note]
> - Ensure the MQTT topic name that you are subscribing to is consistent to what your [[Hardware/Anchor Setup\|anchors]] are sending, and what the [[Server/MQTT Broker\|MQTT Broker]] is sending.
> - Ensure the names of the [[Server/Apache Kafka\|Apache Kafka]] topics are also consistent with what [[Server/Apache Kafka\|Apache Kafka]] will later subscribe to.
> - Change your [[Server/MQTT Broker\|MQTT Broker]]'s domain or IP address accordingly.

### Base implementation (without TLS)

```rust
use rumqttc::{AsyncClient, Event, MqttOptions, Packet, Publish, QoS};
use rdkafka::config::ClientConfig;
use rdkafka::producer::{FutureProducer, FutureRecord};
use bytes::{Buf, Bytes};
use std::time::Duration;
use tokio::select;
use tokio::time::interval;

#[tokio::main]

async fn main() {
    // MQTT setup
    // let mqtt_broker = "192.168.35.121";
    let mqtt_broker = "192.168.255.121";
    
    let mqtt_port = 1883;
    const mqtt_topic1:&str = "IamALIVE";
    const mqtt_topic2:&str = "TisMyDATA";

    // Kafka setup
    let kafka_brokers = "broker:29092";
    let kafka_topic1 = "packet_alive";
    let kafka_topic2 = "packet_data";

    // Create MQTT client
    let mqtt_options = MqttOptions::new("rust_mqtt_client", mqtt_broker, mqtt_port);
    let (mut mqtt_client, mut mqtt_event_loop) = AsyncClient::new(mqtt_options, 10);

    // Subscribe to MQTT topics
    mqtt_client.subscribe(mqtt_topic1, QoS::AtLeastOnce).await.unwrap();
    mqtt_client.subscribe(mqtt_topic2, QoS::AtLeastOnce).await.unwrap();

    // Create Kafka producers
    let kafka_producer: FutureProducer = ClientConfig::new()
        .set("bootstrap.servers", kafka_brokers)
        .set("message.timeout.ms", "5000")
        .set("group.id", "software_dev")
        .create()
        .expect("Producer creation error");

    let mut interval_timer = interval(Duration::from_millis(1000));

    loop {
        select! {
            notification = mqtt_event_loop.poll() => match notification {
                Ok(event) => match event {
                    Event::Incoming(Packet::Publish(p)) => {
                        println!("Received MQTT message, forwarding to Kafka.");
                        match p.topic.as_str() {
                            mqtt_topic1 => {
                                publish_to_kafka(&kafka_producer, kafka_topic1, p.payload.clone()).await;
                            },
                            mqtt_topic2 => {
                                publish_to_kafka(&kafka_producer, kafka_topic2, p.payload.clone()).await;
                            },
                            _ => {
                                eprintln!("Received message from unexpected MQTT topic: {}", p.topic);
                            }
                        }
                    },
                    _ => (),
                },
                Err(e) => eprintln!("MQTT event loop error: {:?}", e),
            },
            _ = interval_timer.tick() => {
                // Nothing to do in the timer tick, but it ensures that the loop yields regularly
            }
        }
    }
}
async fn publish_to_kafka(producer: &FutureProducer, topic: &str, payload: Bytes) {
    // Convert Bytes to Vec<u8> for compatibility with ToBytes trait
    let payload_vec: Vec<u8> = payload.to_vec();
    let key_str = "protobuf".to_string(); // Generic key for all messages.

    let record: FutureRecord<String, Vec<u8>> = FutureRecord::to(topic)
        .payload(&payload_vec)
        .key(&key_str);

    // Send the message to Kafka
    match producer.send(record, Duration::from_secs(0)).await {
        Ok(delivery) => println!("Sent to Kafka: {:?}", delivery),
        Err((e, _)) => eprintln!("Error sending to Kafka: {}", e),
    }
}
```

### Secure implementation (with TLS)

```rust
use rumqttc::{AsyncClient, Event, EventLoop, MqttOptions, Packet, Publish, QoS, TlsConfiguration, Transport};
use std::fs::read;
use rdkafka::config::ClientConfig;
use rdkafka::producer::{FutureProducer, FutureRecord};
use bytes::Bytes;
use std::time::Duration;
use tokio::select;
use tokio::time::interval;

#[tokio::main]

async fn main() {
    // MQTT setup
    // let mqtt_broker = "192.168.35.121";
    let mqtt_broker = "192.168.255.121";
    

    // let mqtt_port = 1883;
    let mqtt_port = 8883; // TLS typically uses port 8883
    const mqtt_topic1:&str = "IamALIVE";
    const mqtt_topic2:&str = "TisMyDATA";

    // Kafka setup
    //let kafka_brokers = "broker:29092";
    let kafka_brokers = "broker:9093"; // Adjust as necessary for TLS, 
    let kafka_topic1 = "packet_alive";
    let kafka_topic2 = "packet_data";

    // Create MQTT client
    // let mqtt_options = MqttOptions::new("rust_mqtt_client", mqtt_broker, mqtt_port);
    let (mut mqtt_client, mut mqtt_event_loop) = setup_tls_mqtt_client(mqtt_broker, mqtt_port, "/certs/rootCA/rootCA.pem", Some("/certs/mqtt_broker/mqtt_broker.pem"), Some("/certs/mqtt_broker/mqtt_broker.key")).await;

    // Subscribe to MQTT topics
    mqtt_client.subscribe(mqtt_topic1, QoS::AtLeastOnce).await.unwrap();
    mqtt_client.subscribe(mqtt_topic2, QoS::AtLeastOnce).await.unwrap();

    // Create Kafka producers
    // let kafka_producer: FutureProducer = ClientConfig::new()
    //     .set("bootstrap.servers", kafka_brokers)
    //     .set("message.timeout.ms", "5000")
    //     .set("group.id", "software_dev")
    //     .create()
    //     .expect("Producer creation error");
    let kafka_producer = create_tls_kafka_producer(kafka_brokers);

    let mut interval_timer = interval(Duration::from_millis(1000));


    loop {
        select! {
            notification = mqtt_event_loop.poll() => match notification {
                Ok(event) => match event {
                    Event::Incoming(Packet::Publish(p)) => {
                        println!("Received MQTT message, forwarding to Kafka.");
                        match p.topic.as_str() {
                            mqtt_topic1 => {
                                publish_to_kafka(&kafka_producer, kafka_topic1, p.payload.clone()).await;
                            },
                            mqtt_topic2 => {
                                publish_to_kafka(&kafka_producer, kafka_topic2, p.payload.clone()).await;
                            },
                            _ => {
                                eprintln!("Received message from unexpected MQTT topic: {}", p.topic);
                            }
                        }
                    },
                    _ => (),
                },
                Err(e) => eprintln!("MQTT event loop error: {:?}", e),
            },
            _ = interval_timer.tick() => {
                // Nothing to do in the timer tick, but it ensures that the loop yields regularly
            }
        }
    }
}


async fn setup_tls_mqtt_client(broker_address: &str, broker_port: u16, ca_cert_path: &str, client_cert_path: Option<&str>, client_key_path: Option<&str>) -> (AsyncClient, EventLoop) {
    // Load the CA certificate
    let ca = read(ca_cert_path).expect("Failed to read CA certificate");

    // Optionally load client certificate and key, if provided
    let client_auth = if let (Some(cert_path), Some(key_path)) = (client_cert_path, client_key_path) {
        let cert = read(cert_path).expect("Failed to read client certificate");
        let key = read(key_path).expect("Failed to read client key");
        // Assume Key is a type that represents the private key, adjust as necessary
        Some((cert, rumqttc::Key::RSA(key))) // Adjust this according to the actual key type expected
    } else {
        None
    };

    let tls_config = TlsConfiguration::Simple {
        ca,
        alpn: None, // Adjust as necessary if ALPN is used
        client_auth,
    };

    // Initialize MqttOptions and configure it
    let mut mqtt_options = MqttOptions::new("rust_mqtt_client", broker_address, broker_port);
    
    mqtt_options.set_transport(Transport::Tls(tls_config)); // Chain other configurations as needed

    AsyncClient::new(mqtt_options, 10)
}



async fn publish_to_kafka(producer: &FutureProducer, topic: &str, payload: Bytes) {
    // Convert Bytes to Vec<u8> for compatibility with ToBytes trait
    let payload_vec: Vec<u8> = payload.to_vec();
    let key_str = "protobuf".to_string(); // Generic key for all messages.

    let record: FutureRecord<String, Vec<u8>> = FutureRecord::to(topic)
        .payload(&payload_vec)
        .key(&key_str);

    // Send the message to Kafka
    match producer.send(record, Duration::from_secs(0)).await {
        Ok(delivery) => println!("Sent to Kafka: {:?}", delivery),
        Err((e, _)) => eprintln!("Error sending to Kafka: {}", e),
    }
}

fn create_tls_kafka_producer(brokers: &str) -> FutureProducer {
    ClientConfig::new()
        .set("bootstrap.servers", brokers)
        .set("security.protocol", "SSL")
        .set("ssl.ca.location", "/certs/rootCA/rootCA.pem")
        // If using client authentication
        .set("ssl.certificate.location", "/certs/kafka1/kafka1.pem")
        .set("ssl.key.location", "/certs/kafka1/kafka1.key")
        // Additional SSL settings as needed
        .create()
        .expect("Kafka producer creation error")
}
```

With this, move on to set up [[Server/Apache Kafka\|Apache Kafka]].
---
{"dg-publish":true,"permalink":"/server/rust-bridge/","tags":["middleware","archi"]}
---

> [!abstract] Rust bridge
> MQTT messages pass from the [[Server/MQTT Broker\|MQTT Broker]], but [[Apache Kafka\|Apache Kafka]] cannot consume directly from the [[Server/MQTT Broker\|MQTT Broker]]. As such, we made this bridge to pass the message from the MQTT topics to the Kafka topics. You can implement this in any other development language, but we chose a Rust development image for this purpose.

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

Create producer application. This refers to something that 'produces' inputs for [[Apache Kafka\|Apache Kafka]], hence the name.

```shell
cargo new producer
cd producer
```

To setup the application, make another `Cargo.toml` inside the `producer` application you just created.

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



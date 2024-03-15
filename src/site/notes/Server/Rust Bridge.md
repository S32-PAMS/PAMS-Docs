---
{"dg-publish":true,"permalink":"/server/rust-bridge/","tags":["middleware","archi"]}
---

> [!abstract] Rust bridge
> MQTT messages pass from the [[Server/MQTT Broker\|MQTT Broker]], but [[Apache Kafka\|Apache Kafka]] cannot consume directly from the [[Server/MQTT Broker\|MQTT Broker]]. As such, we made this bridge to pass the message from the MQTT topics to the Kafka topics. You can implement this in any other development language, but we chose a Rust development image for this purpose.

This assumes [[Server/Server initial setup\|Server initial setup]] is complete.

# Inclusion to Docker compose

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


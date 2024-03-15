---
{"dg-publish":true,"permalink":"/server/mqtt-broker/","tags":["middleware","archi"]}
---

> [!abstract] MQTT Broker
> This is responsible for the **message-passing** functionality of our message-oriented middleware [[Architecture\|Architecture]]. As long as this function is satisfied, it does not matter the choice of the broker.
> Common options for MQTT Broker:
> - [Eclipse Mosquitto](https://mosquitto.org): preferable for single-threaded instances
> - [EMQX NanoMQ](https://nanomq.io): allows multi-threading
> 
> In PAMS prototype: bare prototype uses NanoMQ, and the implementation with security uses Mosquitto, as Mosquitto has support for TLS, while TLS is still a work in progess for NanoMQ.

This setup assumes that the setup in [[Server/Server initial setup\|Server initial setup]] has been completed.
You also need a text editor to create and edit the configuration and Docker Compose files.

> [!note]
> - Receives input from [[Anchors\|Anchors]]
> - Passes output to [[Server/Rust Bridge\|Rust Bridge]]
> - [MQTT Protocol](https://mqtt.org) uses port `1883` for default use, and port `8883` for MQTTS (MQTT with TLS)

# Inclusion of Docker image into the server

Choose one among:
- [NanoMQ Docker hub](https://hub.docker.com/r/emqx/nanomq)
- [Mosquitto Docker hub](https://hub.docker.com/_/eclipse-mosquitto)

Our prototype is on a `docker-compose.yml` file. Include the image of the MQTT you wish to use in your `docker-compose.yml`.

## Mosquitto

```yml
services:
  mosquitto:
    image: eclipse-mosquitto:2.0-openssl
    container_name: mosquitto
    ports:
      - "1883:1883"
      - "8883:8883"
    volumes:
	  - <path to mosquitto.conf>
	  - <path to CA certificate>
	  - <path to MQTT broker certificate>
	  - <path to MQTT broker private key>
```

This assumes that [[For security personnel\|the secure implementation]] is set up. Use the `openssl` version of the image.

## NanoMQ

```yml
services:
  nanomq:
    image: emqx/nanomq:0.14.1-full
    container_name: nanomq
    ports:
      - "1883:1883"
    environment:
      - NANOMQ_BROKER_URL="nmq-tcp://0.0.0.0:1883"
    volumes:
      - ../nanomq/nanomq.conf:/etc/nanomq.conf
```

We used the full image for full functionality of the MQTT broker, but it is not necessary in most cases. The default installation will also work.

Now you can proceed to [[Server/Rust Bridge\|Rust Bridge]]
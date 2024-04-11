---
dg-publish: true
dg-home: 
tags:
  - middleware
  - archi
  - message-queueing
---
> [!abstract] Apache Kafka
> This component is used for 2 purposes:
> 1. Satisfies the **message-queuing** capability of our Message-Oriented Middleware [[Architecture\|Architecture]]. Theoretically, other message queues can also be used, but our prototype used Apache Kafka.
> 2. Messages after [[Server/Apache Flink\|Apache Flink]] will be put into this component again, into another topic, to be consumed by the [[Server/Node bridge\|Node bridge]].

This setup assumes that [[Server/Server initial setup\|Server initial setup]] has been completed.

> [!note]
> - Receives input from [[Server/Rust Bridge\|Rust Bridge]]
> - Port usage:
> 	- Defaults: `9092` without TLS, `9093` with TLS
> 	- `29092` is used to communicate between [[Server/Rust Bridge\|Rust Bridge]] and Kafka
> 	- `10920` is used to communicate from Kafka to [[Server/Apache Flink\|Apache Flink]]
> 	- `9094` is used to communicate between [[Server/Apache Flink\|Apache Flink]] to Kafka to [[Server/Node bridge\|Node bridge]]

Apache Kafka has a Zookeeper dependency, so you need to first have that set up.

# Setup Docker compose for Zookeeper

Apache ZooKeeper is a centralised service for maintaining configuration information, and to do distributed synchronisation.

- [zookeeper Docker hub](https://hub.docker.com/_/zookeeper)

To include this dependency, add this to your `docker-compose.yml`:

```yml
services:
  zookeeper:
    image: confluentinc/cp-zookeeper:7.3.0
    container_name: zookeeper
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000
```

# Setup Docker compose for Kafka

We chose [Confluent](https://developer.confluent.io)'s Kafka implementation.

- [Kafka Docker hub](https://hub.docker.com/r/confluentinc/cp-kafka)

Add this to your `docker-compose.yml`:

```yml
services:
broker:
    image: confluentinc/cp-kafka:7.3.0
    hostname: broker
    container_name: broker
    ports:
    # To learn about configuring Kafka for access across networks see
    # https://www.confluent.io/blog/kafka-client-cannot-connect-to-broker-on-aws-on-docker-etc/
      - "10920:10920"
      - "9096:9096"
    depends_on:
      - zookeeper
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: 'zookeeper:2181'
      KAFKA_AUTO_CREATE_TOPICS_ENABLE: "false"
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: INTERNAL:PLAINTEXT, EXTERNAL:PLAINTEXT, EXTERNAL2:PLAINTEXT
      KAFKA_ADVERTISED_LISTENERS: INTERNAL://broker:29092, EXTERNAL://broker:10920, EXTERNAL2://localhost:9096
      KAFKA_INTER_BROKER_LISTENER_NAME: INTERNAL
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 2
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 2
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 2
```

The confluent image has several components but it's most prominent parts lie in it's environment. To set it up, it is important to have all these attributes. The broker_id helps differentiate it from other brokers that are set up for the solution. Zookeeper connect tells the broker where the zookeeper component is located. 

## Connecting broker ports

`KAFKA_ADVERTISED_LISTENERS` and `KAFKA_LISTENER_SECURITY_PROTOCOL_MAP` are 2 attributes that must be present to set up broker logic. Advertised listeners advertises it's ports to consumers and producers that want to consume and produce from the broker respectively. The format of the listener is as follows

```
< NAME >://< bootstrap-server >:< port >
```

an example of this would be `INTERNAL://broker:29092`. Ensure that the ports chosen are exposed and can be seen from outside the docker container.

```
ports:
  - "10920:10920"
  - "9096:9096"
```

It is important to note that if 2 different brokers are to be set up, their KAFKA_ADVERTISED_LISTENERS must have the same names/ Configurations. This can be seen in our docker compose file. 

```
# broker 1:
KAFKA_ADVERTISED_LISTENERS: INTERNAL://broker:29092, EXTERNAL://broker:10920, EXTERNAL2://localhost:9096

# broker 2: 
KAFKA_ADVERTISED_LISTENERS: INTERNAL://broker2:9094, EXTERNAL://broker2:9095, EXTERNAL2://localhost:9097
```

localhost is currently exposed as many of our frontend components run locally. If the frontend components are ever dockerized, it is ideal to remove the external2 ports.

Now you can proceed to [[Server/Apache Flink\|Apache Flink]]
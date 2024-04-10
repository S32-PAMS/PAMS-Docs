---
{"dg-publish":true,"permalink":"/server/apache-kafka/","tags":["middleware","archi","message-queueing"],"noteIcon":""}
---

> [!abstract] Apache Kafka
> This component is used for 2 purposes:
> 1. Satisfies the **message-queuing** capability of our Message-Oriented Middleware [[Architecture\|Architecture]]. Theoretically, other message queues can also be used, but our prototype used Apache Kafka.
> 2. Messages after [[Server/Apache Flink\|Apache Flink]] will be put into this component again, into another topic, to be consumed by the [[Web server\|Web server]].

This setup assumes that [[Server/Server initial setup\|Server initial setup]] has been completed.

> [!note]
> - Receives input from [[Server/Rust Bridge\|Rust Bridge]]
> - Port usage:
> 	- Defaults: `9092` without TLS, `9093` with TLS
> 	- `29092` is used to communicate between [[Server/Rust Bridge\|Rust Bridge]] and Kafka
> 	- `10920` is used to communicate from Kafka to [[Server/Apache Flink\|Apache Flink]]
> 	- `9094` is used to communicate between [[Server/Apache Flink\|Apache Flink]] to Kafka to [[Web server\|Web server]]

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

```


---
{"dg-publish":true,"permalink":"/readme/","tags":["home","gardenEntry"],"noteIcon":""}
---

# PAMS

> [!abstract] Personnel and Artefact Monitoring System (PAMS)
> **Personnel and Artefact Monitoring System (PAMS)** is an *anti-theft solution* that utilises reliable, robust technology for *precise, real-time localisation* of personnel and sensitive objects within specified environments.
> 
> PAMS is *designed* to provide *security* and enhance operational *efficiency*. It is both *scalable* and *cost-efficient*. It also integrates with existing networking infrastructures to provide *detailed analytics*, enabling effective resource monitoring and management across various sectors.

Find out more about our design:

- [[Architecture\|Architecture]]
- [[Security\|Security]]

Let PAMS resolve your asset tracking needs.

## User Manuals

**Get started** here

- [[Hardware initial setup\|Hardware initial setup]]
- [[Server/Server initial setup\|Server initial setup]]

| Manual | Description |
| ---- | ---- |
| [[Guides/For platform users\|For platform users]] | Guide on how to set up and run an instance of PAMS as a user |
| [[Guides/For security personnel\|For security personnel]] | Guide on how to create and administer the [[Security\|Security]] architecture of PAMS for your instance |
| [[For hardware engineers\|For hardware engineers]] | Guide on how to create your own PAMS Tags and Anchors, as well as link it up to PAMS middleware |
| [[For software engineers\|For software engineers]] | Guide on how to create the middleware, backend, databases and frontend for the PAMS |

Individual components

- #hardware: [[Tags\|Tags]], [[Anchors\|Anchors]]
- #middleware : [[Server/MQTT Broker\|MQTT Broker]], [[Server/Rust Bridge\|Rust Bridge]], [[Server/Apache Kafka\|Apache Kafka]]
- #software : [[Server/Apache Flink\|Apache Flink]], [[Server/Apache Kafka\|Apache Kafka]], [[Web server\|Web server]], [[Frontend/Database\|Database]], [[Frontend/Frontend\|Frontend]], [[Frontend/Camera Server\|Camera Server]]


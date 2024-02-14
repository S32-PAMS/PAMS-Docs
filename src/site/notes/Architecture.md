---
{"dg-publish":true,"permalink":"/architecture/","tags":["archi"]}
---


## System Overview of the PAMS Architecture


![systemarchi.png](/img/user/images/systemarchi.png)

> [!note]
> This is an advanced tracking system that employs Ultra-Wideband (UWB) technology for localisation within a defined space, such as a room. 

The system architecture is **designed to collect, process, and visualise location data** for personnel or assets attached to our PAMS tags. It integrates various technologies, including ESP32 microcontrollers, MQTT for messaging, Apache Kafka for message queuing, Apache Flink for stream processing, and a ExpressJS web server for data management and frontend interaction.

## Hardware components

Design is minimal to reduce weight and bulk.

| Component | Description and role                                                                                                                   |
| --------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| [[Tags\|Tags]]      | Equipped with ESP32 microcontrollers and DW1000 UWB modules<br><br>Broadcast unique data packets that include<br>  - Detachment status<br>  - Battery level<br>  - Distance from anchor<br><br>These tags are attached to objects which should be protected. |
| [[Anchors\|Anchors]]          | Also equipped with ESP32 microcontrollers and DW1000 UWB modules<br><br>Stationed at fixed positions in the room<br>Receive data from tags<br>Responsible for forwarding this information to the system's backend via WiFi, using MQTT protocol                                                                                                                                       |

## Middleware components

Adheres to Message-Oriented Middleware design, where components should have mechanisms for:
- Message passing
- Message queuing

| Component | Description and role |
| ---- | ---- |
| [[Server/MQTT Broker\|MQTT Broker]] | `nanomq` instance running in a Docker container<br>Acts as the nerve centre with hardware-facing communication<br><br>Receives data from anchors and categorises them into 2 topics:<br>- `alive` for heartbeat signals from anchors<br>- `data` for location information from tags |
| [[Rust Bridge\|Rust Bridge]] | A Rust programming environment container<br>Serves as the conduit between the MQTT Broker and Apache Kafka.<br><br>Consumes messages (in raw bytes) received by the MQTT Broker, and passes them into the Apache Kafka message queue |
| [[Apache Kafka\|Apache Kafka]] | Functions as the central message queue<br><br>Organises messages into `alive` and `data` topics<br>This separation facilitates efficient data processing downstream by Apache Flink |

## Software components

Designed for stream processing of real-time messages, to be aggregated in a user-friendly dashboard.

| Component | Description and role |
| ---- | ---- |
| [[Apache Flink\|Apache Flink]] | The stream processing engine<br><br>Consumes data from Kafka<br>Performs operations to transform raw data received into meaningful location data for each tag<br>This processed data is then translated from Protobuf bytes to JSON for web consumption |
| [[Apache Kafka 2\|Apache Kafka 2]] | Second Kafka instance<br><br>Acts as a buffer for processed data<br>Queues the data for storage in a MongoDB database |
| [[MongoDB\|MongoDB]] | Persistent storage backbone of the system<br><br>Archives processed location data along with configuration details for tags and anchors<br>Ensures that the frontend reflects the most current information, providing end-users with accurate and timely insights into the location of tagged objects within their monitored environment |
| [[Web server\|Web server]] | NodeJS web server, enhanced with ExpressJS web framework and Socket.io<br><br>Orchestrates the interaction between the database and the frontend<br>Responsible for the continuous synchronisation of the database and seamless updating of the frontend |
| [[Frontend\|Frontend]] | Interactive and informative dashboard based on Material UI design framework and CreateJS module for dynamic graphical representations<br><br>Ensures a user-friendly experience with an aesthetically pleasing layout and intuitive navigation<br>Enhanced by real-time updates via Socket.IO, the interface displays live location data of tagged objects on a map<br>This caters to users of varied technical proficiencies, presenting complex tracking information in an easily digestible and actionable format |

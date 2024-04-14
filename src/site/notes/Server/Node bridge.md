---
dg-publish: true
dg-home: 
tags:
  - software
  - parallel-processing
  - data-processing
  - socketio
  - websocket
  - backend
  - real-time-processing
---
> [!abstract] Node Bridge
> This component is responsible for consuming data from Apache Kafka and sending it into the relevant frontend components. Socket.IO was chosen for this as it supports real time processing and has broadcasting functionalities, allowing all of our components to receive relevant information at once.

This setup assumes that [[Frontend/Frontend\|Frontend]] has been completed and that the brokers are up running and connected to this container. It will run an error if there is no [[Server/Apache Kafka\|Apache Kafka]] broker to read from or [[Frontend/Frontend\|Frontend]] to connect to.

> [!note]
> - Receives input from [[Server/Apache Kafka\|Apache Kafka]] broker2
> - Sends input to [[Frontend/Frontend\|Frontend]]
> - Port usage:
> 	- `9095` is used to communicate from [[Server/Apache Kafka\|Apache Kafka]] broker2 to [[Server/Node bridge\|Node bridge]]
> 	- `8080` is used to communicate between [[Server/Node bridge\|Node bridge]] to [[Frontend/Frontend\|Frontend]]

This component uses NodeJS to send information to the frontend.

## Import necessary configurations for the bridge

Add the below configurations to the node app.js file

```javascript
const express = require('express');
const { Kafka,logLevel } = require('kafkajs');
const http = require('http');

const app = express();
const server = http.createServer(app);
```

## Read From Kafka

The Kafka consumer we are using here is KafkaJS. KafkaJS was chosen as it is a dependency free client of Apache Kafka, keeping this component lightweight and as fast as possible.

The Kafka client requires a configuration with broker ids. Add the below code to create the Kafka client consumer object
```javascript
const kafka = new Kafka({
  clientId: 'nodejs-app',
  brokers: ['localhost:9097','broker2:9095'],
});

const consumer = kafka.consumer({ groupId: 'test-group' });
```

The consumer runs an `eachMessage` function that processes each message read from the Kafka client. `partitionsConsumedConcurrently` indicate the number of parallel messages processed. Increasing this number can take advantage of parallelisation in your machine.

```javascript
consumer.run({
  partitionsConsumedConcurrently: 100,
  eachMessage: async ({ topic, partition, message }) => {
```

Now find the topic this data is sent from to process the data. 

```javascript
switch(topic){
	case 'out_data':
		...
	case 'out_alive':
		...
```

## Send to Socket.io

Set up socket io configurations, ensuring there are no CORS errors if data is passed locally.

```javascript
const socketIO = require('socket.io');
const io = socketIO(server, {
  cors: {
    Access_Control_Allow_Origin:["http://nodejs:8080", "http://localhost:8080"],
    methods: ["GET", "POST"]
  }
});
```

Initiate connection to the frontend web app.

```javascript
io.on('connection', (socket) => {
  console.log('\n Client connected');
  ...})
  
const port = process.env.PORT || 8080;
server.listen(port, () => {
  console.log(`Server is running on port ${port}`);
});
```

Send data out to frontend after processing the data, where message is the message retrieved from the Kafka client. Here, the WebSocket `id` will be the `tagid`. This allows each tag to be processed as a separate thread for processing at the frontend.

```javascript
var full_msg = JSON.parse(message.value.toString())
var tagid = full_msg["tagid"]
io.emit(tagid, {data:message.value.toString()})
```

### Combine both functions

Combining both functions lead to a` runKafkaConsumerCood` function run asynchronously. This component first seeks connections to both the Kafka broker and the Frontend. When both are found, data is then processed and handled.

```javascript
function runKafkaConsumerCood() {
    await consumer.connect();
    await consumer.subscribe({ topics: ['out_data','out_alive']});

    consumer.run({
      partitionsConsumedConcurrently: 100,
      eachMessage: async ({ topic, partition, message }) => {
        switch(topic){
        
            case 'out_data':
              var full_msg = JSON.parse(message.value.toString())
              var tagid = full_msg["tagid"]
              console.log("anchordata values", message.value.toString());
              var toSend = message.value.toString();
              io.emit(tagid, {data:toSend});

            case 'out_alive': // only sent if anchor is detected dead
              var isdead = message.value.toJSON();
              var anchorid = isdead["anchorid"];
              console.log("anchoralive values", message.value.toString());
              io.emit(anchorid, {data:message.value.toString()})
        }
      }
    });
  }

runKafkaConsumerCood().catch((error) => {
  console.error('Error starting Kafka consumer:', error);
});

io.on('connection', (socket) => {
  console.log('\n Client connected');
  socket.on('disconnect', () => {
	  console.log('Client disconnected');
  });
});
```
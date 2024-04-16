---
{"dg-publish":true,"permalink":"/Guides/For Testers/","tags":["run","middleware","software"],"noteIcon":""}
---

> [!abstract] How to run the test code
> This page contains instructions on how to run loadtesting.

# Running the Tests

## Setting the load to be tested

Simply change the maxcount to the desired number of tags in the system. 

```python
maxcount = 100
```

## Run the test

Install python and this file's dependencies. Before running the test, ensure the system is already working up to the frontend. 

```bash
python loadtesting.py
```

# Creating Load Testing Code

It is not sensible to create 100 tags to find the upper limit of our system. Instead for load testing, we shall be intercepting the system at the kafka broker, sending scripted messages in. 

## Creating mock data

The test code presumes a position in 6 different rooms, and 3 known anchor coordinates. generating a random room for each run.

```python
roomcoods = [[1.5, 0.5], [1.5, 1.5], [1.5, 2.5], [0.5, 0.5], [0.5, 1.5], [0.5, 2.5]]
anchorcoods = [[2.0,0.0], [0.0,2.0], [2.0,3.0]]
```

For each run, a distance sent is mocked as data consumed by each anchor.

```python
td = message_pb2.TagData()
td.detached = False
td.battery_low = False
td.distance = getDist(roomcoods[room], anchorcoods[anchorid])
```

To ensure the data had sufficient buffer while processing in [[Server/Apache Flink\|Apache Flink]], each anchor would send 3 sets of the same data. 

```python
for anchorid in range(3):
        for i in range(3):
```

## Sending data with a Python Kafka Client

The client `confluent_kafka` is used. The configurations are as follows.

```python
from confluent_kafka import Producer
import socket
import message_pb2
import random
import time

from google.protobuf.timestamp_pb2 import Timestamp
import math

conf = {'bootstrap.servers': 'localhost:9096',
        'security.protocol': 'PLAINTEXT',
        'client.id': socket.gethostname()}

producer = Producer(conf)
```

The data is then sent out one at a time to the kafka topic `out_data`

```python
maxcount = 100
topic1 = "packet_data"
topic2 = "packet_alive"

for i in range(len(taglist)):
    producer.produce(topic1, value=taglist[i], callback=acked)
    producer.poll(1)
```
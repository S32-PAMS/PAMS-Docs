---
{"dg-publish":true,"permalink":"/Server/Apache Flink/","tags":["software","parallel-processing","data-processing"],"noteIcon":""}
---

> [!abstract] Apache Flink
> This component is responsible for processing data to generate localization. Apache flink is chosen for it's ability to parallelize data processes, allowing us to scale the number of tags on assets.


This setup assumes that [[Server/Server initial setup\|Server initial setup]] has been completed.

> [!note]
> - Receives input from [[Server/Apache Kafka\|Apache Kafka]] (broker1)
> - Port usage:
> 	- `10920` is used to communicate from Kafka to [[Server/Apache Flink\|Apache Flink]]
> 	- `9094` is used to communicate between [[Server/Apache Flink\|Apache Flink]] to Kafka to [[Server/Node bridge\|Node bridge]]

Deploying Apache Flink in python requires the additional installation of helm, kubectl and kind. 

# Flink Processor and Components

Flink's datastream requires a source, a processor and a sink.

## Kafka data source

Data is read from the first kafka broker on the topics packet_alive and packet_data for information on anchor liveness data and tag data respectively. 

```python    
	ds = env.from_source(

        KafkaSource.builder()\
		.set_bootstrap_servers("broker:10920") \
		.set_topics("packet_data")\
		.set_value_only_deserializer(SimpleStringSchema(charset=enc0))\
		.set_starting_offsets(KafkaOffsetsInitializer.latest()).build(),

        WatermarkStrategy \
        .for_bounded_out_of_orderness(Duration.of_seconds(10)) \
        .with_idleness(Duration.of_days(1)),
        "brokerdata",
        type_info=Types.STRING()
    )
```

The open port on the first kafka broker uses the port 10920. Since kafka serializes data that is passed into it, it had to be deserialized. Here we use a simple string schema despite our data being serialized in Protobuf.

This is possible since protobuf data is compressed and sent as a string. Instead of deserializing at this step, data is deserialized en-mass later. 

The watermark strategy in a flink source has functions dictating the allowed lateness of a data packet. Here our maximum bounded out of orderness allowed is 10sec. Past 10sec we assume the information is now irrelavant. Additionally, the watermark strategy allows a maximum idleness of 1 day before the system is triggered for a restart. 
## Processor

### 1. Deserializing data from Protobuf

To deserialize data from Protobuf, one first needs to know the data structure the message has. With the .proto file, we are able to create a `python metaclass`. The python metaclass helps us parse objects into the given data structure. To create this python metaclass, a line of code has to be ran on the command line interface, generating a `._pb2.py` file. 

```bash
protoc --python_out=. messsage.proto
```

In our flink object, we take advantage of this metaclass to create the Protobuf object and deserialize the necessary variables. Below we see an AnchorData object created, with it's values Parsed from String. Here `Line` is the string extracted from the kafka source. Below is a sample of the data processor for anchor data. 

```python
import message_pb2 as makepb2
def getAnchor(line):
        msgAnchor = makepb2.AnchorData()
        msgAnchor.ParseFromString(bytes(line, encoding=enc))
        return msgAnchor
```

The information is then split into sections, to retrieve the information packed into other data structures

```python
def getviaTag(msgAnchor):
        out = [Row(tag_id=tagid, anchor_id=msgAnchor.anchor_id, tag=msgAnchor.tags[tagid]) for tagid in msgAnchor.tags]
        yield from out
        
def getTagInfo(msgAnchor):
        out = Row(msgAnchor.tag_id, msgAnchor.anchor_id, msgAnchor.tag.distance, msgAnchor.tag.detached, msgAnchor.tag.battery_low)
        return out
```

A key_by and window functions from datastream are then used to group the packets into tags and only take 3 packets at once. 

```python
.key_by(lambda i: i[0]) \
.count_window(3) \
```

Lastly, the trilateration algorithm is executed. 
```python
class GetCood(ProcessWindowFunction):
        def process(self, key, context:ProcessWindowFunction.Context, args): 
            msg1, msg2, msg3 = args # windowed arguments
            if msg1[1] == msg2[1] or msg1[1] == msg3[1] or msg3[1] == msg2[1]: # filters away sets of 3s with 2 same anchors.
                yield Row("err", -1.0, -1.0, False, False)
			# gets known coordinates
            x1, y1 = getAnchorCood(msg1[1])
            x2, y2 = getAnchorCood(msg2[1])
            x3, y3 = getAnchorCood(msg3[1])
            r1, r2, r3 = msg1[2], msg2[2], msg3[2]

            A = 2*x2 - 2*x1
            B = 2*y2 - 2*y1
            C = r1**2 - r2**2 - x1**2 + x2**2 - y1**2 + y2**2
            D = 2*x3 - 2*x2
            E = 2*y3 - 2*y2
            F = r2**2 - r3**2 - x2**2 + x3**2 - y2**2 + y3**2

            try:
                x = (C*E - F*B) / (E*A - B*D)
                y = (C*D - A*F) / (B*D - A*E)

            except: # catches when the float distance creates inaccurate values
                x = 0.1
                y = 0.1
                
            yield Row(msg1[0], float(x), float(y), msg1[3], msg1[4])
```
## Data sink

### Set the json schema for the output
```python
jsonSchema_data = JsonRowSerializationSchema.builder()\

        .with_type_info(Types.ROW_NAMED(['tagid', 'x','y', "detached", "battery_low"],[Types.STRING(), Types.FLOAT(), Types.FLOAT(), Types.BOOLEAN(), Types.BOOLEAN()])).build()
```

### Set properties for the sink as a kafka producer
```python
flinkKafkaProdSink_data = KafkaSink.builder() \
    .set_bootstrap_servers("broker2:9094") \
    .set_property("group.id", "test-group")\
    .set_record_serializer(
        KafkaRecordSerializationSchema.builder()
            .set_topic("out_data")
            .set_value_serialization_schema(jsonSchema_data)
            .build()
    ) \
    .set_delivery_guarantee(DeliveryGuarantee.AT_LEAST_ONCE) \
    .build()
```
### Sink the data
```python
ds.sink_to(flinkKafkaProdSink_data)
```
## Miscellaneous Environment Set Up
Environment set up gives us some opportunities. 
```python
env = StreamExecutionEnvironment.get_execution_environment()
# write all the data to one file
env.set_parallelism(2)
env.enable_checkpointing(10000, CheckpointingMode.EXACTLY_ONCE)
env.set_stream_time_characteristic(TimeCharacteristic.EventTime)
```
- Parallelism gives us the amount of data being ran simultaneuously. In this case since we are commbining data packets, we do not want to spread our resources too thin, opting for a parallelism of 2.
- Checkpointing gives us a reset mechanism, checking if a reset is necessary every 10 seconds and saving it's current state if a restart is not necessary.
# Changing features
## Add Anchors
To add anchors to the system, add to the anchorDict dictionary at the top of the file. Add the AnchorId as the key and the known coordinates, a list with x and y in the format [x, y] as the value.

```python
anchorDict = {"A0": [2,0], "A1": [0,2], "A2": [2,3]}
```

After adding to the dictionary, delete the existing kubectl container, build the existing docker file and upload image to kind. 

```python
> kubectl delete -f pyflink-runstream.yaml
> docker build -f Dockerfile . -t s32pams/pyflink-runstream:latest
> kind load docker-image s32pams/pyflink-runstream:latest --name pyflink-test
> kubectl create -f pyflink-runstream.yaml
```

This will then give you an updated version of the Flink Processor that retrieves and calculates information from the new anchor. 
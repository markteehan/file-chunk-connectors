## Quick Start
The following steps show the File Chunk Source & Sink connectors to stream a file to a Kafka topic named file-chunk-events. Create this topic in advance of running the example.

### Prerequisites
	•	Confluent Platform
	•	Confluent CLI (requires separate installation)
Install the connector the steps above for _Install the Connector Manually_
Start Confluent Platform using the Confluent CLI confluent local commands. confluent local services start

Create the source directories (queued, finished, error) and the sink directory (download)
#### Linux:
```
cd /tmp
mkdir queued finished error download
cp /var/log/install.log ./queued/install.log   (or any file as a test file to send)
```

#### Windows:
```

MKDIR queued finished error download
COPY  somefile.JPG .\queued\somefile.JPG (choose any file as a test file to send)
```

Create the topic - test operation with a single partition topic before setting up multi partition(/task) pipelines.

```
kafka-topics --create --topic file-chunk-events --partitions 1 --bootstrap-server localhost:9092
```

### Source Connector
Create a file chunk-source.properties with the following contents to split files into chunks of 50k bytes. The "converter" properties are needed to ensure that the default Connect worker serializer (generall Avro) is overwritten with the byteSerializer for this connector.
 
```
name=file-chunk-source
connector.class=com.github.markteehan.file.chunk.source.ChunkSourceConnector
binary.chunk.size.bytes=50000
input.file.pattern=.*
topic=file-chunk-events
input.path=/tmp/queued
error.path=/tmp/error
finished.path=/tmp/finished
#
task.count=1
halt.on.error=false
cleanup.policy.maintain.relative.path=true
input.path.walk.recursively=true
#
value.converter=org.apache.kafka.connect.converters.ByteArrayConverter
key.converter=org.apache.kafka.connect.converters.ByteArrayConverter
key.converter.schemas.enable=false
value.converter.schemas.enable=false

```

_Note: if you see this error then check for tabs or unnecessary spaces in the properties file_

```
"message": "Connector config {name=XX} contains no connector type"
```



Load the File Chunk Source connector:

```

 confluent local services connect connector load file-chunk-source --config file-chunk-source.properties
 ```

Confirm that the connector is in a RUNNING state.
```

confluent local services connect connector status file-chunk-source
```
		 

### Sink Connector

	Create a file file-chunk-sink.properties with the following contents. Note that “topics” should always contain the same (single) topic name specified for the source connector. This must be a single-partition topic. Set tasks.max=1.
The "converter" properties are needed to ensure that the default Connect worker serializer (generall Avro) is overwritten with the byteSerializer for this connector. 

```

connector.class=com.github.markteehan.file.chunk.sink.ChunkSinkConnector
name=file-chunk-sink
output.dir=/tmp/download
topics=file-chunk-events
#
tasks.max=1
#
auto.register.schemas=false
schema.ignore=true
schema.generation.enabled=false
key.converter.schemas.enable=false
value.converter.schemas.enable=false
schema.compatibility=none
#
value.converter=org.apache.kafka.connect.converters.ByteArrayConverter
key.converter=org.apache.kafka.connect.converters.ByteArrayConverter
key.converter.schemas.enable=false
value.converter.schemas.enable=false

```
	
Load the File Chunk Sink connector:

```

confluent local services connect connector load file-chunk-sink --config file-chunk-sink.properties
```

Confirm that the connector is in a RUNNING state. 
```

confluent local services connect connector status file-chunk-sink
```

Confirm that the messages are being sent to Kafka - note that the console output matches the content of the file: it may be binary.


Confirm that the files are being written into the Download directory. 
Note that with both connectors running on the same machine, the finished and download directories will contain the same contents.
Copy larger files to the queued directory and observe that they are reconstructed in the download directory. 
This Single-machine demo shows common operation: a common deployment pattern is to have many source connectors sending to one (or multiple) sink connectors. 

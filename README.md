# File Chunk Connector (Source & Sink)

<img width="905" alt="image" src="https://github.com/markteehan/file-chunk-connectors/assets/16135308/4b3b4c25-a9c7-4b09-9c0e-339c7e696b84">

_TL;DR: the Kafka Connect File Chunk Source & Sink connectors are a fancy way to send a file somewhere else._

The connectors are packaged either as Kafka Connect [plugins]([url](https://github.com/markteehan/file-chunk-connectors)) or as complete [tarballs]([url](https://github.com/markteehan/file-chunk-tarballs)) (that include Kafka, Java, the connectors and setup scripts) enabling low-friction deployment on windows or linux. Some users of these connectors desire to use Kafka client features (retries, partitions, encryption, authentication) to stream files from edge devices (Windows laptops) with poor connectivity. These connectors do not requires a license for use (they are not yet source-available).


The Kafka Connect File Chunk Source & Sink connectors watch a directory for files and read the data as new files are written to the input directory. Once a file has been read, it will be placed into the configured ```finished.path``` directory.  Each file is produced to a topic as a stream of messages after splitting the file into chunks of ```binary.chunk.size.bytes```. Messages are serialised as bytes: files can contain any content: text, logs, image, video, any binary encoding. The chunk size must be <= message.max.bytes for the Kafka cluster. 

The matching Sink connector consumes from the topic, using header metadata to reconstruct the file on a filesystem local to the Sink connector. The MD5 signature of the reconstructed target files must match the signature of the source file, otherwise an error is returned.  The connectors should be paired to form a complete data pipeline. Multipartition/multitask operation is supported.

Subdirectories at source are recreated at target. For example consider 100 source connectors sending binary files from a local directory called queued/`hostname`.  The Sink connector reconstructs the files inside 100 subdirectories on the target machine, enabling subdirectory names (to multiple levels) to carry metadata.

These connectors are based on the excellent [Spooldir]([url](https://github.com/jcustenborder/kafka-connect-spooldir)) connectors (created by Jeremy Custenborder) and the File-Sink connectors.

## Features
The File Chunk Source & Sink connectors include the following features:
	•	At least once delivery
	•	Multiple tasks

### Exactly once delivery
The File Chunk source and sink connector guarantee an exactly-once pipeline when they are run together - a combination of an at-least delivery guarantee for the source connector and duplicate handling by the sink connector. Consuming File Chunk messages from the topic using a client other than the File Chunk sink connector is not possible.

### Multiple tasks
The File Chunk connectors support running one or more tasks. You can specify the number of tasks in the tasks.max configuration parameter. Multiple tasks may improve performance when moving a large amount of data. The Sink connector handles chunk arrival in any order: the file is reconstructed when all chunks are available.


## Installation
### Install the File Chunk Source & Sink Connector Packages
You can install this connector by manually downloading the ZIP file. These connectors are not available on Confluent Hub.
#### Prerequisites
Note You must install the connector on every machine where Connect will run.  
_Install the connector manually_
Download the jarfiles for the Source & Sink connectors and then follow the manual connector installation instructions.
#### Manually
Copy the jarfiles for the source and sink connectors to your kafka connect plugins directory and restart Kafka Connect.

1. Create a directory under the `plugin.path` on your Connect worker.
2. On Linux/Mac this is generally under share/confluent-hub-components, on Windows create  new directory kafka\plugins
3. Copy these two jarfiles the newly created subdirectory:
```
curl -O -L https://raw.githubusercontent.com/markteehan/file-chunk-connectors/main/plugins/file-chunk-sink-1.0-jar-with-dependencies.jar
curl -O -L https://raw.githubusercontent.com/markteehan/file-chunk-connectors/main/plugins/file-chunk-source-1.0-jar-with-dependencies.jar
```
3. Restart the Connect worker. Kafka Connect will discover and unpack each jarfile. 


#### Deploy the Source/Sink connectors using the tarball
The file-chunk-tarballs repo is a self-contained tarball (=zipfile) containing the stack components run Kafka connect with these connectors. The tarball contains an uploader and a downloader: the streaming service starts on windows or on linux. Elevated privileges (admin or root account) are not required: the service runs in a CMD window. See the repo for deployment instructions.


## License
The File Chunk connectors do not require a License for use.

## Configuration Properties
For a complete list of configuration properties, see the specific connector documentation.
For an example of how to get Kafka Connect connected to Confluent Cloud, see Connect Self-Managed Kafka Connect to Confluent Cloud.

## Quick Start
The following steps show the File Chunk Source & Sink connectors to stream a file to a Kafka topic named file-chunk-events. Create this topic in advance of running the example.

### Prerequisites
	•	Confluent Platform
	•	Confluent CLI (requires separate installation)
Install the connector the steps above for _Install the Connector Manually_
Start Confluent Platform using the Confluent CLI confluent local commands. confluent local services start

Create a data directory and generate test data. 
#### Linux:
```

mkdir queued finished error download
cp /var/log/install.log ./queued/install.log   (or any file as a test file to send)
```

#### Windows:
```

MKDIR queued finished error download
COPY  somefile.JPG .\queued\somefile.JPG (choose any file as a test file to send)
```


 Create chunk-source.json file with the following contents to split files into chunks of 50k bytes.
 
```

{
                                   "name": "file-chunk-source"
, "config":{
                                  "topic": "file-chunk-events"
,                       "connector.class": "com.github.markteehan.file.chunk.source.SpoolDirBinaryFileSourceConnector"
,                            "input.path": “/path/to/queued"
,                            "error.path": "/path/to/error"
,                         "finished.path": "/path/to/finished"
,                    "input.file.pattern": ".*"
,                            "task.count": "1"
,                        "halt.on.error" : "FALSE"
,              "binary.chunk.size.bytes" : "50000"
, "cleanup.policy.maintain.relative.path": "true"
,          "input.path.walk.recursively" : "true"
}}
```

Load the File Chunk Source connector:

_Caution You must include a double dash (--) between the topic name and your flag. For more information, see [this]([url](https://unix.stackexchange.com/questions/11376/what-does-double-dash-mean-also-known-as-bare-double-dash)) post._

```

 confluent local services connect connector load file-chunk-source --config file-chunk-source.json
 ```
_Important Don’t use the confluent local commands in production environments. _

Confirm that the connector is in a RUNNING state.
```

confluent local services connect connector status file-chunk-source
```
		 
Confirm that the messages are being sent to Kafka - note that the console output matches the content of the file: it may be binary.
```

kafka-console-consumer \
		    --bootstrap-server localhost:9092 \
		    --topic file-chunk-events \
		    --from-beginning | jq '.'
 ```
	Start the Sink Connector:

	Create a chunk-sink.json file with the following contents. Note that “topics” should always contain the same (single) topic name specified for the source connector. If the topic has multiple partitions then set tasks.max to the same number.
```
		
		{
		                           "name": "file-chunk-sink"
		, "config":{
		                         "topics": "file-chunk-events"
		,               "connector.class": "io.confluent.developer.connect.ChunkSinkConnector"
		,                     "tasks.max": "1"
		,                    "output.dir": "/path/to/download"
		,         "auto.register.schemas": "false"
		,                 "schema.ignore": "true"
		,     "schema.generation.enabled": "false"
		,  "key.converter.schemas.enable": "false"
		,"value.converter.schemas.enable": "false"
		,          "schema.compatibility": "none"
		,              "merge.iterations": "3"
		,"merge.iterations.interval.secs": "30"
		}}
```
	
 Load the File Chunk Sink connector:

```

confluent local services connect connector load file-chunk-sink --config file-chunk-sink.json
```

Confirm that the connector is in a RUNNING state. 
```

confluent local services connect connector status file-chunk-sink
```

Confirm that the files are being written into the Download directory. 
Note that with both connectors running on the same machine, the finished and download directories will contain the same contents.
Copy larger files to the queued directory and observe that they are reconstructed in the download directory. 
This Single-machine demo shows common operation: a common deployment pattern is to have many source connectors sending to one (or multiple) sink connectors. 


## Logging
Logging output to stream two JPG files (using a binary.chunk.size.bytes = 512000).

**File Chunk Source Connector**
```
Checking to ensure input.path '/tmp/queued' is writable
Checking to ensure error.path '/tmp/error' is writable
Checking to ensure finished.path '/tmp/finished' is writable
WorkerSourceTask{id=uploader-nn-0} Source task finished initialization and start

Found 5 potential files 
someLargeFile.JPG: (size 16500000 bytes) producing 40 chunks 
someLargeFile.JPG: Finished processing all chunks (sent 40 file chunk(s) to Kafka). MD5=c31f4dbb504d805c389a26d3680d0f7b

```

**File Chunk Sink Connector**

```
someLargeFile.JPG: (size 16500000) - merge from 40 chunks completed MD5=c31f4dbb504d805c389a26d3680d0f7b

OR
ERROR someLargeFile.JPG: (size 16500000) - merge from 40 chunks failed - MD5 mismatch. MD5 (source)=c31f4dbb504d805c389a26d3680d0f7b MD5 (target)=889af9ce64b6395960aaf2a5e307cd6d

```

## Deployment Uses 
This connector can be used to stream binary files such as .JPEG, .AVI, encrypted or compressed content, ranging in size from megabytes to gigabytes.
This connector borrows heavily from the spooldir source connectors written by Jeremy Custenborder. To stream and schemify text or avro content, use the spooldir source connectors - to stream binary files; use this connector.

There are many options available to send files between two endpoints: rsync, sftp, scp, curl, wget: sending files using streaming via Kafka offers a number of benefits that are built into the kafka client, including resume/send-retry, TLS encryption, authentication, access control, compression, replay and parallelism. These are multiple tradeoffs to consider.

This connector enables any Kafka cluster (including Confluent Cloud, Confluent Platform or Apache Kafka) to be used to stream files of any size.
The [tarballs repo]([url](https://github.com/markteehan/file-chunk-tarballs)) contains a readymade stack (including the plugins, kafka connect, java, configuration properties and scripts) to start the source or sink on windows or linux using a "config" and then a "start". This uses connect-standalone (not connect-distributed). 
If you are unfamiliar with Kafka connect, then I recommend starting with the tarballs stack.



## Scenarios
It is suitable for data upload scenarios that include 
- file-generating edge devices (including windows clients) where a kafka connect client is preferable to custom-code uploader deployment
- sync filesystems contents to a remote server: this is only suitable for static (completed) files. It is unsuitable for mirroring open files
- client endpoints with unreliable networking: Kafka client infinite-retries ensures eventual data delivery with no-touch intervention
- a desire for sophisticated encryption (such as cipher selection) which can be challenging using other data-sender utilities
- a desire for sophisticated SaaS based authentiation models (including OAUTH2) enabling credential-less client deployments
- automatic compression & decompression of file content (if uncompressed)
- Kafka consumer features such as low-cost fanout of data to multiple services simultaneously, parallelism of consumer threads, delivery guarantees


### Operation
Similar to  "spooldir", this connector monitors an input directory for new files that match an input patterns. Eligible files are split into fixed-size chunks of 
_binary.chunk.size.bytes_ which are produced to a kafka topic before moving the file to the "finished" directory (or the "error" directory if any failure occurred). 
The input directory on the sending device requires sufficient headroom to duplicate the largest file, since file chunks are written to the filesystem temporarily during streaming. Files are processed one at a time: the first queued file is chunked, sent and finished; before the second file is processed; and so on. 
The connector observes & recreates subdirectories (to multiple levels): if an eligible file is created in a subdirectory "field-team-056", then the sink connector will reassemble the file in a subdirectory of the same name.


## Payloads
Message payloads are encoded as bytestream: there is no use of message schemas. Any Kafka client can be used to consume events created by the source connector. The accompanying file-chunk-sink connector reassembles chunks as files to a local filesystem using metadata in the message headers. This borrows from the open-source file-sink connector. 

## Ordering
With single partition operation, chunks are produced and consumed in order. With multi-partition operation, chunks arrive out of order but assembly of the final file starts when all chunks are available. File assembly is always in files order: since chunks are numbered monotonically, they are always assembled in order. At present sleep/iterate cycles for the sink connector are not configurable; this will be changed soon.

  
## Limitations
These limitations are in place for the current release:
- maximum chunk count for any single file is 100,000
- the maximum file size must fit inside the JVM of the source (and sink) machine (this is needed for MD5 verification)

## License
This repo contains compiled jarfiles only, so there is no license restriction for usage.
A [possible] future source-code release is likely to be restricted by a GPL v2 license.
Please feel free to raise issues and I will endeavour to address them.

# Troubleshooting
```org.apache.kafka.common.errors.RecordTooLargeException: The request included a message larger than the max message size the server will accept.```
The value specified for binary.chunk.size.bytes exceeds the maximum message size (message.max.bytes) for this Kafka cluster. Specify a smaller value.




## Connect Worker Properties
Aside from common defaults specify the following:
```
acks = ALL
max.in.flight.requests.per.connection = 1
retries = 2147483647
retry.backoff.ms = 500
producer.compression.type=none   # if the source files are already compressed (JPEG, AVI, etc)
value.converter=org.apache.kafka.connect.converters.ByteArrayConverter
value.converter.schemas.enable=false
key.converter=org.apache.kafka.connect.converters.ByteArrayConverter
key.converter.schemas.enable=false
value.converter.schema.registry.url=http://localhost:8081
key.converter.schema.registry.url=http://localhost:8081
group.id=file-chunk-group
```



## Source Connector configuration properties

#### File System
##### `error.path`

The directory to place files in which have error(s). This directory must exist and be writable by the user running Kafka Connect.

*Importance:* HIGH

*Type:* STRING


##### `input.file.pattern`

Regular expression to check input file names against. This expression must match the entire filename. The equivalent of Matcher.matches().

*Importance:* HIGH

*Type:* STRING


##### `input.path`

The directory to read files that will be processed. This directory must exist and be writable by the user running Kafka Connect.

*Importance:* HIGH

*Type:* STRING

*Validator:* Absolute path to a directory that exists and is writable.



##### `finished.path`

The directory to place files that have been successfully processed. This directory must exist and be writable by the user running Kafka Connect.

*Importance:* HIGH

*Type:* STRING



##### `halt.on.error`

Should the task halt when it encounters an error or continue to the next file.

*Importance:* HIGH

*Type:* BOOLEAN

*Default Value:* true



##### `binary.chunk.size.bytes`

The size of each data chunk that will be produced to the topic. The size must be < Kafka cluster message.max.bytes. 
Matching files in input.directory exceeding this size will be split (chunked) into multiple smaller files of this size. 
File chunks are renamed with incremental chunk numbering. Default is 1048588 bytes. 
See https://kafka.apache.org/documentation/#brokerconfigs_message.max.bytes";

*Importance:* HIGH

*Type:* INT

*Default Value:* 1048588

*Validator:* [1,...]




##### `file.minimum.age.ms`

The amount of time in milliseconds after the file was last written to before the file can be processed.

*Importance:* LOW

*Type:* LONG

*Default Value:* 0

*Validator:* [0,...]




##### `processing.file.extension`

Before a file is processed, a flag is created in its directory to indicate the file is being handled. The flag file has the same name as the file, but with this property appended as a suffix.

*Importance:* LOW

*Type:* STRING

*Default Value:* .PROCESSING

*Validator:* Matches regex( ^.*\..+$ )


#### General


##### `topic`

The Kafka topic to write the data to.

*Importance:* HIGH

*Type:* STRING


##### `empty.poll.wait.ms`

The amount of time to wait if a poll returns an empty list of records.

*Importance:* LOW

*Type:* LONG

*Default Value:* 500

*Validator:* [1,...,9223372036854775807]


## Sink Connector configuration properties

##### `merge.iterations`

The total number of times that the connect will attempt to reconstruct a file from its chunks. If any chunks are missing then a further attempt will be required. In some sceanrios, network conditions may require a higher value - for example if a high percentage of chunks arrive out of order or if a source connector is experiencing a high retry count due to poor network connectivity.

*Importance:* HIGH

*Type:* INTEGER

*Default Value:* 3

##### `merge.iterations.interval.secs`

The interval (in seconds) between reattempts to reconstruct a file from its chunks. Increasing this property increases the time that the connector will wait for all chunks for a file to be consumed.

*Importance:* HIGH

*Type:* INTEGER

*Default Value:* 30


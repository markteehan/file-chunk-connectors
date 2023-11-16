# File Chunk Connector (Source & Sink)

<img width="905" alt="image" src="https://github.com/markteehan/file-chunk-connectors/assets/16135308/4b3b4c25-a9c7-4b09-9c0e-339c7e696b84">


## Introduction
The **File Chunk Source Connector** streams files though a Kafka topic by breaking each file into fixed-size "chunks" that fit inside a kafka message. A matching **File Chunk Sink Connector** consumes file chunks and re-assembles the Kafka messages into the original file.
For example a 45.07MB .JPG image file using a chunk size of 512KB creates 89 Kafka messages: 88 fixed-size chunks of 512KB and a final 89th chunk of 14KB. The chunk size must be less than the **message.max.bytes** for the Kafka cluster. The maximum count of chunks for any file is 100,000 chunks.

This connector can be used to stream binary files such as .JPEG, .AVI, encrypted or compressed content, ranging in size from megabytes to gigabytes.
This connector borrows heavily from the spooldir source connectors written by Jeremy Custenborder. To stream and schemify text or avro content, use the spooldir source connectors - to stream binary files; use this connector.

There are many options available to send files between two endpoints: rsync, sftp, scp, curl, wget: sending files using streaming via Kafka offers a number of benefits that are built into the kafka client, including resume/send-retry, TLS encryption, authentication, access control, compression, replay and parallelism. These are multiple tradeoffs to consider.

This connector enables any Kafka cluster (including Confluent Cloud, Confluent Platform or Apache Kafka) to be used to stream files of any size.

## Scenarios
It is suitable for data upload scenarios that include 
- file-generating edge devices (including windows clients) where a kafka connect client is preferable to custom-code uploader deployment
- sync filesystems contents to a remote server
- client endpoints with unreliable networking: Kafka client infinite-retries ensures eventual data delivery with no-touch intervention
- a desire for sophisticated encryption (such as cipher selection) which can be challenging using other data-sender utilities
- a desire for sophisticated SaaS based authentiation models (including OAUTH2) enabling credential-less client deployments
- automatic compression & decompression of file content (if uncompressed)
- Kafka consumer features such as low-cost fanout of data to multiple services simultaneously, parallelism of consumer threads, delivery guarantees

### Operation
Similar to  "spooldir", this connector monitors an input directory for new files that match an input patterns. Eligible files are split into fixed-size chunks of 
_binary.chunk.size.bytes_ which are produced to a kafka topic before moving the file to the "finished" directory (or the "error" directory if any failure occurred). 
The input directory on the sending device requires sufficient headroom to duplicate the largest file, since file chunks are written to the filesystem temporarily during streaming. Files are processed one at a time: the first queued file is chunked, sent and finished; before the second file is processed; and so on. 
The connector observes & recreates subdirectories (to one level): if an eligible file is created in a subdirectory "field-team-056", then the sink connector will reassemble the file in a subdirectory of the same name.


## Payloads
Message payloads are encoded as bytestream: there is no use of message schemas. Any Kafka client can be used to consume events created by the source connector. The accompanying file-chunk-sink connector reassembles chunks as files to a local filesystem using metadata in the message headers. This borrows from the open-source file-sink connector. 


## Delivery guarantees
Similar to spooldir, data can be replayed by moving files from finished.path to input.path. 
The sink connector writes to a single directory, with a ".PROCESSING" extension while a file is being assembled.
The sink connector writes a new file when the first chunk is consumed; subsequent chunks are appended in sequence, and the ".PROCESSING" extension is removed by renaming the file after writing the final chunk. Message metadata is used to check the chunk count for each file. 


## Limitations
These limitations are in place for the current release:
- tasks.count = 1 - queued files are processed by a single uploader task
- partitions = 1 - single-partition operation to ensure out of the box ordering of chunks
- maximum chunk count for any single file is 100,000
- further testing is need to determine the maximum file size that can be sent
- file integrity is determined by chunk count, other techniques (chunk size verification, MD5 UUID) will be added.


## License
This repo contains compiled jarfiles only, so there is no license restriction for usage.
A [possible] future source-code release is likely to be restricted by a GPL v2 license.
Please feel free to raise issues and I will endeavour to address them.


# Troubleshooting
```org.apache.kafka.common.errors.RecordTooLargeException: The request included a message larger than the max message size the server will accept.```
The value specified for binary.chunk.size.bytes exceeds the maximum message size (message.max.bytes) for this Kafka cluster. Specify a smaller value.



# Installation
## Confluent Hub
These connectors are unavailable on Confluent Hub.


## Manually
Copy the jarfiles for the source and sink connectors to your kafka connect plugins directory and restart Kafka Connect.


1. Create a directory under the `plugin.path` on your Connect worker.
2. On Linux/Mac this is generally under share/confluent-hub-components, on Windows create  new directory kafka\plugins
3. Copy these two jarfiles the newly created subdirectory.
```
curl -O -L https://raw.githubusercontent.com/markteehan/file-chunk-connectors/main/plugins/file-chunk-sink-0.0.1-SNAPSHOT-jar-with-dependencies.jar
curl -O -L https://raw.githubusercontent.com/markteehan/file-chunk-connectors/main/plugins/file-chunk-source-0.1-SNAPSHOT-jar-with-dependencies.jar
```
3. Restart the Connect worker. Kafka Connect will discover and unpack each jarfile. 


## Connect Worker Properties
Aside from common defaults specify the following:
```
producer.compression.type=none   # if the source files are already compressed (JPEG, AVI, etc)
value.converter=org.apache.kafka.connect.converters.ByteArrayConverter
value.converter.schemas.enable=false
key.converter=io.confluent.connect.json.JsonSchemaConverter
key.converter.schemas.enable=false
value.converter.schema.registry.url=http://localhost:8081
key.converter.schema.registry.url=http://localhost:8081
group.id=source-775
```

## Source Connector Job Submit example
Aside from common defaults specify the following:
```
{
                           "name": "file-chunk-source-job"
,"config":{
                                  "topic": "file-chunk-events"
,                       "connector.class": "com.github.markteehan.file.chunk.source.SpoolDirBinaryFileSourceConnector"
,                            "input.path": "/tmp/queued"
,                            "error.path": "/tmp/error"
,                         "finished.path": "/tmp/finished"
,                    "input.file.pattern": ".*JPG.*"
,                            "task.count": "1"
,                        "halt.on.error" : "FALSE"
,               "file.buffer.size.bytes" : "3300000"
,                  "file.minimum.age.ms" : "1000"
,              "binary.chunk.size.bytes" : "510240"
, "cleanup.policy.maintain.relative.path": "true"
,          "input.path.walk.recursively" : "true"
}}
```

### Compartison with Spooldir
Some differences exist with the spooldir connector:
```
        cleanup.policy is MOVE: files are always moved from input.path to finished.path/error.path.   
      task.partitioner is unavailable as this version of the connector sets task.count=1
file.buffer.size.bytes is set to the same value as `binary.chunk.size.bytes`
 files.sort.attributes is preset to NAMEASC
          file.charset is preset to UTF-8
            batch.size is always 1 
```

### Logging
Logging output for a two JPG file splits using a binary.chunk.size.bytes = 512000. 

**Source Connector**
```
INFO Checking to ensure input.path '/tmp/queued' is writable
INFO Checking to ensure error.path '/tmp/error' is writable
INFO Checking to ensure finished.path '/tmp/finished' is writable
INFO WorkerSourceTask{id=uploader-nn-0} Source task finished initialization and start

INFO Found 2 potential files
INFO ImageFile-001.JPG: (size 26193701 bytes) producing 52 chunks of 512000 bytes
INFO ImageFile-001.JPG-00052-of-52.CHUNK: Finished. Produced 52 file chunks to Kafka.

INFO Found 1 potential files
INFO ImageFile-002.JPG: (size 15438513 bytes) producing 31 chunks of 512000 bytes
INFO ImageFile-002.JPG-00031-of-31.CHUNK: Finished. Produced 31 file chunks to Kafka.

```

**Sink Connector**

```
INFO [task-0] ImageFile-001.JPG:  (size 26193701 bytes) - merge from 52 chunks completed. (io.confluent.developer.connect.ChunkSinkTask:209)
INFO [task-0] ImageFile-002.JPG:  (size 15438513 bytes) - merge from 31 chunks completed. (io.confluent.developer.connect.ChunkSinkTask:209)

```



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


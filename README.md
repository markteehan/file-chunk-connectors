# File Chunk Connector (Source & Sink)

# Introduction
The File Chunk Source Kafka Connector streams files though a Kafka topic by breaking each file into "chunks" that fit inside a kafka message. A matching File Chunk Sink connector consumes file chunks and re-assembles the Kafka messages into the original file.
For example a 45.07MB .JPG image file using a chunk size of 512KB creates 89 Kafka messages: 88 chunks of 512KB and a final 89th chunk of 14KB. The chunk size must be less than the max.message.size for the nominated topic. The maximum chunk count for a file is 100,000 chunks.

This connector can be used to stream binary files such as .JPEG, .AVI, encrypted or compressed content, ranging in size from megabytes to gigabytes.
This connector borrows heavily from the spooldir source connectors written by Jeremy Custenborder. To stream and schemify text or avro content, use the spooldir source connectors - to stream binary files; use this connector.

There are many options available to send files between two endpoints: sftp, scp, curl, wget: sending files using streaming via Kafka offers a number of benefits that are built into the kafka client, including send-retry, TLS encryption, authentication, access control, compression, replay and parallelism. Sending files using Kafka Connect adds framework support that facilitates field deployment: including release control, access control, logging, configuration etc.

It is suitable for data upload scenarios that include 
- file-generating edge devices (including windows clients) where a kafka connect client is preferable to custom-code uploader deployment
- client endpoints with unreliable networking: Kafka client infinite-retries ensures eventual data delivery with no-touch intervention
- sophisticated encryption (such as cipher selection) which can be challenging using other data-sender utilities
- sophisticated SaaS based authentiation models (including OAUTH2) enabling credential-less client deployments
- automatic compression & decompression of file content (if uncompressed)
- Kafka consumer features such as low-cost fanout of data to multiple services simultaneously, parallelism of consumer threads, delivery guarantees

This connector enables any Kafka cluster (including Confluent Cloud, Confluent Platform or Apache Kafka) to be used to stream files of any size.

Similar to  "spooldir", this connector monitors an input directory for new files that match an input patterns. Eligible files are split into chunks of chunk size (binary.chunk.size.bytes) which are produced to a kafka topic before moving the file to the "finished" directory (or the "error" directory if any failure occurred). 
The input directory on the sending device requires sufficient headroom to duplicate the largest file, since file chunks are written to the filesystem temporarily during streaming. Files are processed one at a time: the first queued file is chunked, sent and finished; before the second file is processed; and so on. 

Message payloads are encoded as bytestream: there is no use of message schemas - any Kafka client can be used to consume. The accompanying file-chunk-sink connector reassembles chunks as files to a local filesystem - this borrows from the open-source file-sink connector. 



These limitations are in place for the current release:
- tasks.max = 1 - queued files are processed by a single uploader task
- partitions = 1 - single-partition operation to ensure out of the box ordering of chunks
- maximum chunk count for any single file is 100,000
- further testing is need to determine the maximum file size that can be sent   


# License
GPL v2. The source code is not yet available; pending some refactoring. Meanwhile the assembled jarfiles are available for download.
Please feel free to raise issues and I will endeavour to address them.

# Installation
## Confluent Hub
These connectors are unavailable on Confluent Hub.


## Errors
```org.apache.kafka.common.errors.RecordTooLargeException: The request included a message larger than the max message size the server will accept.```
The value specified for binary.chunk.size.bytes exceeds the maximum message size for this Kafka cluster. SPecify a smaller value.



## Manually
Copy the jarfiles for the source and sink connectors to your kafka connect plugins directory and restart Kafka Connect:

1. Create a directory under the `plugin.path` on your Connect worker.
2. Copy all of the dependencies under the newly created subdirectory (curl; below)
3. Restart the Connect worker.

curl -O -L https://raw.githubusercontent.com/markteehan/file-chunk-connectors/main/plugins/kafka-connect-spooldir-2.0-SNAPSHOT.tar.gz
curl -O -L https://raw.githubusercontent.com/markteehan/file-chunk-connectors/main/plugins/file-chunk-source-2.0-SNAPSHOT-jar-with-dependencies.jar

## Connect Worker Properties
Aside from common defaults specify the following:
producer.compression.type=none   # if the source files are already compressed (JPEG, AVI, etc)
value.converter=org.apache.kafka.connect.converters.ByteArrayConverter
value.converter.schemas.enable=false
...check
key.converter=io.confluent.connect.json.JsonSchemaConverter
key.converter.schemas.enable=true
topic=file-chunk-events  <==========
value.converter.schema.registry.url=http://localhost:8081
key.converter.schema.registry.url=http://localhost:8081
schema.registry.url=http://localhost:8081


## Source Connector Job Submit example
Aside from common defaults specify the following:
```
{
                           "name": "file-chunk-source-job"
,"config":{
                                  "topic": "file-chunk-connector-events"
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

#### File System


##### `error.path`

The directory to place files in which have error(s). This directory must exist and be writable by the user running Kafka Connect.

*Importance:* HIGH

*Type:* STRING

*Validator:* Absolute path to a directory that exists and is writable.



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



##### `cleanup.policy`

Determines how the connector should cleanup the files that have been successfully processed. NONE leaves the files in place which could cause them to be reprocessed if the connector is restarted. DELETE removes the file from the filesystem. MOVE will move the file to a finished directory. MOVEBYDATE will move the file to a finished directory with subdirectories by date

*Importance:* MEDIUM

*Type:* STRING

*Default Value:* MOVE

*Validator:* Matches: ``NONE``, ``DELETE``, ``MOVE``, ``MOVEBYDATE``



##### `task.partitioner`

The task partitioner implementation is used when the connector is configured to use more than one task. This is used by each task to identify which files will be processed by that task. This ensures that each file is only assigned to one task.

*Importance:* MEDIUM

*Type:* STRING

*Default Value:* ByName

*Validator:* Matches: ``ByName``


##### `binary.chunk.size.bytes`

The size of each data chunk that will be produced to the topic. The size must be < Kafka cluster max.message.size. 

*Importance:* HIGH

*Type:* INT

*Default Value:* 131072

*Validator:* [1,...]




##### `file.buffer.size.bytes`  DELETEME

The size of buffer for the BufferedInputStream that will be used to interact with the file system.

*Importance:* LOW

*Type:* INT

*Default Value:* 131072

*Validator:* [1,...]



##### `file.minimum.age.ms`

The amount of time in milliseconds after the file was last written to before the file can be processed.

*Importance:* LOW

*Type:* LONG

*Default Value:* 0

*Validator:* [0,...]



##### `files.sort.attributes`

The attributes each file will use to determine the sort order. `Name` is name of the file. `Length` is the length of the file preferring larger files first. `LastModified` is the LastModified attribute of the file preferring older files first.

*Importance:* LOW

*Type:* LIST

*Default Value:* [NameAsc]

*Validator:* Matches: ``NameAsc``, ``NameDesc``, ``LengthAsc``, ``LengthDesc``, ``LastModifiedAsc``, ``LastModifiedDesc``



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



##### `batch.size`

The number of records that should be returned with each batch.

*Importance:* LOW

*Type:* INT

*Default Value:* 1000



##### `empty.poll.wait.ms`

The amount of time to wait if a poll returns an empty list of records.

*Importance:* LOW

*Type:* LONG

*Default Value:* 500

*Validator:* [1,...,9223372036854775807]



##### `file.charset`  DELETEME

Character set to read wth file with.

*Importance:* LOW

*Type:* STRING

*Default Value:* UTF-8

*Validator:* Big5,Big5-HKSCS,CESU-8,EUC-JP,EUC-KR,GB18030,GB2312,GBK,IBM-Thai,IBM00858,IBM01140,IBM01141,IBM01142,IBM01143,IBM01144,IBM01145,IBM01146,IBM01147,IBM01148,IBM01149,IBM037,IBM1026,IBM1047,IBM273,IBM277,IBM278,IBM280,IBM284,IBM285,IBM290,IBM297,IBM420,IBM424,IBM437,IBM500,IBM775,IBM850,IBM852,IBM855,IBM857,IBM860,IBM861,IBM862,IBM863,IBM864,IBM865,IBM866,IBM868,IBM869,IBM870,IBM871,IBM918,ISO-2022-CN,ISO-2022-JP,ISO-2022-JP-2,ISO-2022-KR,ISO-8859-1,ISO-8859-13,ISO-8859-15,ISO-8859-16,ISO-8859-2,ISO-8859-3,ISO-8859-4,ISO-8859-5,ISO-8859-6,ISO-8859-7,ISO-8859-8,ISO-8859-9,JIS_X0201,JIS_X0212-1990,KOI8-R,KOI8-U,Shift_JIS,TIS-620,US-ASCII,UTF-16,UTF-16BE,UTF-16LE,UTF-32,UTF-32BE,UTF-32LE,UTF-8,windows-1250,windows-1251,windows-1252,windows-1253,windows-1254,windows-1255,windows-1256,windows-1257,windows-1258,windows-31j,x-Big5-HKSCS-2001,x-Big5-Solaris,x-euc-jp-linux,x-EUC-TW,x-eucJP-Open,x-IBM1006,x-IBM1025,x-IBM1046,x-IBM1097,x-IBM1098,x-IBM1112,x-IBM1122,x-IBM1123,x-IBM1124,x-IBM1129,x-IBM1166,x-IBM1364,x-IBM1381,x-IBM1383,x-IBM29626C,x-IBM300,x-IBM33722,x-IBM737,x-IBM833,x-IBM834,x-IBM856,x-IBM874,x-IBM875,x-IBM921,x-IBM922,x-IBM930,x-IBM933,x-IBM935,x-IBM937,x-IBM939,x-IBM942,x-IBM942C,x-IBM943,x-IBM943C,x-IBM948,x-IBM949,x-IBM949C,x-IBM950,x-IBM964,x-IBM970,x-ISCII91,x-ISO-2022-CN-CNS,x-ISO-2022-CN-GB,x-iso-8859-11,x-JIS0208,x-JISAutoDetect,x-Johab,x-MacArabic,x-MacCentralEurope,x-MacCroatian,x-MacCyrillic,x-MacDingbat,x-MacGreek,x-MacHebrew,x-MacIceland,x-MacRoman,x-MacRomania,x-MacSymbol,x-MacThai,x-MacTurkish,x-MacUkraine,x-MS932_0213,x-MS950-HKSCS,x-MS950-HKSCS-XP,x-mswin-936,x-PCK,x-SJIS_0213,x-UTF-16LE-BOM,X-UTF-32BE-BOM,X-UTF-32LE-BOM,x-windows-50220,x-windows-50221,x-windows-874,x-windows-949,x-windows-950,x-windows-iso2022jp



##### `task.count`

Internal setting to the connector used to instruct a task on which files to select. The connector will override this setting.

*Importance:* LOW

*Type:* INT

*Default Value:* 1

*Validator:* [1,...]



##### `task.index`

Internal setting to the connector used to instruct a task on which files to select. The connector will override this setting.

*Importance:* LOW

*Type:* INT

*Default Value:* 0

*Validator:* [0,...]


#### Timestamps


##### `timestamp.mode`  DELETEME

Determines how the connector will set the timestamp for the [ConnectRecord](https://kafka.apache.org/0102/javadoc/org/apache/kafka/connect/connector/ConnectRecord.html#timestamp()). If set to `Field` then the timestamp will be read from a field in the value. This field cannot be optional and must be a [Timestamp](https://kafka.apache.org/0102/javadoc/org/apache/kafka/connect/data/Schema.html). Specify the field  in `timestamp.field`. If set to `FILE_TIME` then the last modified time of the file will be used. If set to `PROCESS_TIME` the time the record is read will be used.

*Importance:* MEDIUM

*Type:* STRING

*Default Value:* PROCESS_TIME

*Validator:* Matches: ``FIELD``, ``FILE_TIME``, ``PROCESS_TIME``



## [Json Source Connector](https://jcustenborder.github.io/kafka-connect-documentation/projects/kafka-connect-spooldir/sources/SpoolDirJsonSourceConnector.html)  DELETEME

```
com.github.markteehan.file.chunk.source.SpoolDirJsonSourceConnector
```

This connector is used to `stream <https://en.wikipedia.org/wiki/JSON_Streaming>` JSON files from a directory while converting the data based on the schema supplied in the configuration.
### Important

There are some caveats to running this connector with `schema.generation.enabled = true`. If schema generation is enabled the connector will start by reading one of the files that match `input.file.pattern` in the path specified by `input.path`. If there are no files when the connector starts or is restarted the connector will fail to start. If there are different fields in other files they will not be detected. The recommended path is to specify a schema that the files will be parsed with. This will ensure that data written by this connector to Kafka will be consistent across files that have inconsistent columns. For example if some files have an optional column that is not always included, create a schema that includes the column marked as optional.
### Note

If you want to import JSON node by node in the file and do not care about schemas, do not use this connector with Schema Generation enabled. Take a look at the Schema Less Json Source Connector.
### Tip

To get a starting point for a schema you can use the following command to generate an all String schema. This will give you the basic structure of a schema. From there you can changes the types to match what you expect.
.. code-block:: bash

   mvn clean package
   export CLASSPATH="$(find target/kafka-connect-target/usr/share/kafka-connect/kafka-connect-spooldir -type f -name '*.jar' | tr '\n' ':')"
   kafka-run-class com.github.markteehan.file.chunk.source.AbstractSchemaGenerator -t json -f src/test/resources/com/github/jcustenborder/kafka/connect/spooldir/json/FieldsMatch.data -c config/JsonExample.properties -i id

Z
##### `timestamp.mode`

Determines how the connector will set the timestamp for the [ConnectRecord](https://kafka.apache.org/0102/javadoc/org/apache/kafka/connect/connector/ConnectRecord.html#timestamp()). If set to `Field` then the timestamp will be read from a field in the value. This field cannot be optional and must be a [Timestamp](https://kafka.apache.org/0102/javadoc/org/apache/kafka/connect/data/Schema.html). Specify the field  in `timestamp.field`. If set to `FILE_TIME` then the last modified time of the file will be used. If set to `PROCESS_TIME` the time the record is read will be used.

*Importance:* MEDIUM

*Type:* STRING

*Default Value:* PROCESS_TIME

*Validator:* Matches: ``FIELD``, ``FILE_TIME``, ``PROCESS_TIME``



##### `parser.timestamp.date.formats`

The date formats that are expected in the file. This is a list of strings that will be used to parse the date fields in order. The most accurate date format should be the first in the list. Take a look at the Java documentation for more info. https://docs.oracle.com/javase/6/docs/api/java/text/SimpleDateFormat.html

*Importance:* LOW

*Type:* LIST

*Default Value:* [yyyy-MM-dd'T'HH:mm:ss, yyyy-MM-dd' 'HH:mm:ss]



##### `parser.timestamp.timezone`

The timezone that all of the dates will be parsed with.

*Importance:* LOW

*Type:* STRING

*Default Value:* UTC



## [Binary File Source Connector](https://jcustenborder.github.io/kafka-connect-documentation/projects/kafka-connect-spooldir/sources/SpoolDirBinaryFileSourceConnector.html)

```
com.github.markteehan.file.chunk.source.SpoolDirBinaryFileSourceConnector
```

This connector is used to read an entire file as a byte array write the data to Kafka.
### Warning

Large files will be read as a single byte array. This means that the process could run out of memory or try to send a message to Kafka that is greater than the max message size. If this happens an exception will be thrown.
### Important

The recommended converter to use is the ByteArrayConverter. Example: `value.converter=org.apache.kafka.connect.storage.ByteArrayConverter`
### Configuration





# Development

## Building the source

```bash
mvn clean package
```

## Contributions

Contributions are always welcomed! Before you start any development please create an issue and
start a discussion. Create a pull request against your newly created issue and we're happy to see
if we can merge your pull request. First and foremost any time you're adding code to the code base
you need to include test coverage. Make sure that you run `mvn clean package` before submitting your
pull to ensure that all of the tests, checkstyle rules, and the package can be successfully built.

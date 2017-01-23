vaquar khan

Compile the code using  

mvn compile assembly:single

https://github.com/vaquarkhan/vk-notes--Apache-Spark-and-Flink/wiki/Kafka-POC-on-Ubanu

Read about Kafka:

https://sookocheff.com/post/kafka/kafka-in-a-nutshell/

http://www.slideshare.net/JiangjieQin/handle-large-messages-in-apache-kafka-58692297

--------------------------------------------------------------------------------------

How do I get exactly-once messaging from Kafka?


Exactly once semantics has two parts: avoiding duplication during data production and avoiding duplicates during data consumption.
There are two approaches to getting exactly once semantics during data production:

Use a single-writer per partition and every time you get a network error check the last message in that partition to see if your last write succeeded

Include a primary key (UUID or something) in the message and deduplicate on the consumer.
If you do one of these things, the log that Kafka hosts will be duplicate-free. However, reading without duplicates depends on some co-operation from the consumer too. If the consumer is periodically checkpointing its position then if it fails and restarts it will restart from the checkpointed position. Thus if the data output and the checkpoint are not written atomically it will be possible to get duplicates here as well. This problem is particular to your storage system. For example, if you are using a database you could commit these together in a transaction. The HDFS loader Camus that LinkedIn wrote does something like this for Hadoop loads. The other alternative that doesn't require a transaction is to store the offset with the data loaded and deduplicate using the topic/partition/offset combination.

I think there are two improvements that would make this a lot easier:

Producer idempotence could be done automatically and much more cheaply by optionally integrating support for this on the server.
The existing high-level consumer doesn't expose a lot of the more fine grained control of offsets (e.g. to reset your position). We will be working on that soon


---------------------------------------------------------------------------------------------
Read more about message delivery in kafka:

https://kafka.apache.org/08/design.html#semantics

So effectively Kafka guarantees at-least-once delivery by default and allows the user to implement at most once delivery by disabling retries on the producer and committing its offset prior to processing a batch of messages. Exactly-once delivery requires co-operation with the destination storage system but Kafka provides the offset which makes implementing this straight-forward.
Probably you are looking for "exactly once delivery" like in jms

https://cwiki.apache.org/confluence/display/KAFKA/FAQ#FAQ-HowdoIgetexactly-oncemessagingfromKafka?



There are two approaches to getting exactly once semantics during data production: 1. Use a single-writer per partition and every time you get a network error check the last message in that partition to see if your last write succeeded 2. Include a primary key (UUID or something) in the message and deduplicate on the consumer.


-------------------------------------------------------------------------------------------
http://ben.kirw.in/2014/11/28/kafka-patterns/

-------------------------------------------------------------------------------------------
Performance :
https://www.cloudera.com/documentation/kafka/latest/topics/kafka_performance.html

-------------------------------------------------------------------------------------------
HANDLING LARGE MESSAGES IN KAFKA

Kafka is well optimized for small messages. Around 10KB seem to be the sweet spot for getting the best throughput (as seen in LinkedIn’s famous benchmark). But sometimes, we need to send larger messages – XML documents or even JSON can get a bit bulky and we end up with messages in the 10M-100M range. What do we do in this case?

Here are few suggestions:

The best way to send large messages is not to send them at all. If shared storage is available (NAS, HDFS, S3), placing large files on the shared storage and using Kafka just to send a message about the file’s location is a much better use of Kafka.
The second best way to send large messages is to slice and dice them. Use the producing client to split the message into small 10K portions, use partition key to make sure all the portions will be sent to the same Kafka partition (so the order will be preserved) and have the consuming client sew them back up into a large message.
Kafka producer can be used to compress messages. If the original message is XML, there’s a good chance that the compressed message will not be very large at all. Use compression.codec and compressed.topics configuration parameters in the producer to enable compression. GZip and Snappy are both supported.
But what if you really need to use Kafka with large messages? Well, in this case, there are few configuration parameters you need to set:

broker configs:

message.max.bytes (default:1000000) – Maximum size of a message the broker will accept. This has to be smaller than the consumer fetch.message.max.bytes, or the broker will have messages that can’t be consumed, causing consumers to hang.
log.segment.bytes (default: 1GB) – size of a Kafka data file. Make sure its larger than 1 message. Default should be fine (i.e. large messages probably shouldn’t exceed 1GB in any case. Its a messaging system, not a file system)
replica.fetch.max.bytes (default: 1MB) – Maximum size of data that a broker can replicate. This has to be larger than message.max.bytes, or
a broker will accept messages and fail to replicate them. Leading to potential data loss.
Consumer configs:

fetch.message.max.bytes (default 1MB) – Maximum size of message a consumer can read. This should be the size of message.max.bytes or larger.
If you choose to configure Kafka for large messages, there are few things you need to take into consideration. Large messages are not an afterthought, they have impact on overall design of the cluster and the topics:

Performance: As we’ve mentioned, benchmarks indicate that Kafka reaches maximum throughput with message size of around 10K. Larger messages will show decreased throughput. Its important to remember this when doing capacity planning for a cluster.
Available memory and number of partitions: Brokers will need to allocate a buffer the size of replica.fetch.max.bytes for each partition they replicate. So if replica.fetch.max.bytes = 1MB and you have 1000 partitions, that will take around 1GB of RAM. Do the math and make sure the number of partitions * the size of the largest message does not exceed available memory, or you’ll see OOM errors. Same for consumers and fetch.message.max.bytes – make sure there’s enough memory for the largest message for each partition the consumer replicates. This may mean that you end up with fewer partitions if you have large messages, or you may need servers with more RAM.
Garbage collection - I did not personally see this issue, but I think its a reasonable concern. Large messages may cause longer garbage collection pauses (as brokers need to allocate large chunks), keep an eye on the GC log and on the server log. If long GC pauses cause Kafka to lose the zookeeper session, you may need to configure longer timeout values for zookeeper.session.timeout.ms.

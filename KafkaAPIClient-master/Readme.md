
Compile the code using  -vaquar khan

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

http://ben.kirw.in/2014/11/28/kafka-patterns/



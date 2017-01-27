Introduction to Kafka
=====================

Kafka acts as a kind of write-ahead log (WAL) that records messages to a persistent store (disk) and allows subscribers to read and apply these changes to their own stores in a system appropriate time-frame.

**Terminology**:

 * Producers send messages to brokers
 * Consumers read messages from brokers
 * Messages are sent to a topic
 * Each topic is broken into one or more ordered partitions of messages

**Internals**:

Kafka a distributed message publishing/subscribing system of one or more brokers, each one with a set of zero or more partitions for each existing topic. Kafka persists periodically messages to disk, so in case of failure last ones might get loss. This speeds up the publishing operation, as publishers dont need to wait until that data gets written to disk.

![kafka-cluster-overview](https://dl.dropbox.com/s/d3fyep6xp1lnrkb/kafka-cluster-overview.png)

When a publisher connects to a Kafka cluster, it queries which partitions exist for the topic and which nodes are responsible for each partition. Publishers assign messages to each partition using an hashing algorithm and deliver them to the broker responsible for that partition.

![kafka-topics-replication](https://dl.dropbox.com/s/m2wcy2kkmxykudv/kafka-topics-partitions-replicas.png)

For each partition, a broker stores the incoming messages with monoticaly increasing order identifiers (offsets) and persists the “deck” to disk using a data structure with access complexity of O(1).

A subscriber is a set of co-operating processes that belong to a consumer-group. Each consumer in the group get assigned a set of partitions to consume from. One key difference to other message queue systems is that each partition is consumed by the same consumer and this allows each consumer to track their progress on consumption on the thread and update it asynchronously.

Consumers keep track on what they consume and store asynchronously in Zookeeper. This is the key point that allows high-throughput. In case of consumer failure, a new process can start from the last saved point, eventually processing the last messages twice.

The broker subscription API requires the identifier of the last message a consumer had from a given partition and starts to stream from that point on. The constant access time data structures on disk play an important role here to reduce disk seeks.

Both consumer groups and brokers are dynamic, so if the amount of incoming messages increase, you can just add new broker nodes to the list and each of them will contain a defined number of partitions for each topic. According to the number of partitions you have, you can also spawn more subscriber processes if the ones you have can’t handle the new partition’s messages in a reasonable time.

The subscriber process can then have a consumer stream per thread and can co-operate with other instances running on other machines by defining each consumer to belong to the same consumer group.  There's no gain on having the total number of threads in the consumer cluster higher than the number of partitions, as each partition will comunicate at most through one consumer stream.

![kafka-subscriber-model](https://dl.dropbox.com/s/zm5f4hgt59uqqzo/kafka_subscriber_cluster.png)

Each consumer thread receives a message from a partition. The message is parsed into a POJO that will be a key to a counting hash map.


Custom Producer:
----------------
**Producer code:**

```java
import java.util.*;

import kafka.javaapi.producer.Producer;
import kafka.producer.KeyedMessage;
import kafka.producer.ProducerConfig;

public class TestProducer {
    public static void main(String[] args) {
        long events = Long.parseLong(args[0]);
        Random rnd = new Random();
        
        //Define properties for how the Producer finds the cluster, serializes 
        //the messages and if appropriate directs the message to a specific 
        //partition.
        Properties props = new Properties();
        props.put("metadata.broker.list", "broker1:9092,broker2:9092 ");
        props.put("serializer.class", "kafka.serializer.StringEncoder");
        props.put("partitioner.class", "example.producer.SimplePartitioner");
        props.put("request.required.acks", "1");
        
        ProducerConfig config = new ProducerConfig(props);

        //Define producer object, its a java generic and takes 2 params; first
        //type of partition key, second type of the message
        Producer<String, String> producer = new Producer<String, String>(config);

        for (long nEvents = 0; nEvents < events; nEvents++) { 
                //build message
               long runtime = new Date().getTime();  
               String ip = “192.168.2.” + rnd.nextInt(255); 
               String msg = runtime + “,www.example.com,” + ip; 
               
               //Finally write the message to broker (here, page_visits is topic
               //name to write to, ip is the partition key and msg is the actual
               //message)
               KeyedMessage<String, String> data = new KeyedMessage<String, String>("page_visits", ip, msg);
               producer.send(data);
        }
        producer.close();
    }
}
```

**Partitioning Code:**

Takes a key, in this case its IP address, finds the last octect and does a modulo on number of partitions defined for the topic. The benefit of this partitioning logic is all web visits from the same source IP end up in the same Partition. Of course so do other IPs, but your consumer logic will need to know how to handle that.

```java
import kafka.producer.Partitioner;
import kafka.utils.VerifiableProperties;

public class SimplePartitioner implements Partitioner<String> {
    public SimplePartitioner (VerifiableProperties props) {

    }

    public int partition(String key, int a_numPartitions) {
        int partition = 0;
        int offset = key.lastIndexOf('.');
        if (offset > 0) {
           partition = Integer.parseInt( key.substring(offset+1)) % a_numPartitions;
        }
       return partition;
  }

}
```

**Running Code:**

Before running this, make sure you have created the Topic page_visits. From the command line:

```shell
bin/kafka-create-topic.sh --topic page_visits --replica 3 --zookeeper localhost:2181 --partition 5
```

Run the code with below dependency jars compiled into final jar:

*Compile Time Required JARs*

```
kafka_2.8.0-0.8.0-beta1.jar
log4j-1.2.17.jar
metrics-core-2.1.1.jar
scala-library.jar
slf4j-api-1.7.5.jar
```

To confirm you have data, use the command line tool to see what was written:

```shell
bin/kafka-console-consumer.sh --zookeeper localhost:2181 --topic page_visits --from-beginning
```

Consumers
---------
Consumers are of two types `consumer groups` and `simple consumer`

###Consumer Groups:
Consumer Groups or High Level Consumer will abstract most of the details of consuming events from kafka. High Level Consumer stores the last offset read from a specific partition in ZooKeeper. This offset is stored based on the name provided to Kafka when the process starts. This name is referred to as the Consumer Group. The Consumer Group name is global across a Kafka cluster, so you should be careful that any 'old' logic Consumers be shutdown before starting new code.

**Desigining High Level Consumer:**

High Level Consumer should be a multi-threaded application.

Very simple consumer:

```java
package com.test.groups;

import kafka.consumer.ConsumerIterator;
import kafka.consumer.KafkaStream;

public class ConsumerTest implements Runnable {
    private KafkaStream m_stream;
    private int m_threadNumber;

    public ConsumerTest(KafkaStream a_stream, int a_threadNumber) {
        m_threadNumber = a_threadNumber;
        m_stream = a_stream;
    }

    public void run() {
        ConsumerIterator<byte[], byte[]> it = m_stream.iterator();
        while (it.hasNext())
            System.out.println("Thread " + m_threadNumber + ": " + new String(it.next().message()));
        System.out.println("Shutting down Thread: " + m_threadNumber);
    }
}
```

Consumer Group Implementation:

```java
package com.test.groups;

import kafka.consumer.ConsumerConfig;
import kafka.consumer.KafkaStream;
import kafka.javaapi.consumer.ConsumerConnector;

import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.Properties;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class ConsumerGroupExample {
    private final ConsumerConnector consumer;
    private final String topic;
    private  ExecutorService executor;

    public ConsumerGroupExample(String a_zookeeper, String a_groupId, String a_topic) {
        consumer = kafka.consumer.Consumer.createJavaConsumerConnector(
                createConsumerConfig(a_zookeeper, a_groupId));
        this.topic = a_topic;
    }

    public void shutdown() {
        if (consumer != null) consumer.shutdown();
        if (executor != null) executor.shutdown();
    }

    public void run(int a_numThreads) {
        Map<String, Integer> topicCountMap = new HashMap<String, Integer>();
        topicCountMap.put(topic, new Integer(a_numThreads));
        Map<String, List<KafkaStream<byte[], byte[]>>> consumerMap = consumer.createMessageStreams(topicCountMap);
        List<KafkaStream<byte[], byte[]>> streams = consumerMap.get(topic);

        // now launch all the threads
        //
        executor = Executors.newFixedThreadPool(a_numThreads);

        // now create an object to consume the messages
        //
        int threadNumber = 0;
        for (final KafkaStream stream : streams) {
            executor.submit(new ConsumerTest(stream, threadNumber));
            threadNumber++;
        }
    }

    private static ConsumerConfig createConsumerConfig(String a_zookeeper, String a_groupId) {
        Properties props = new Properties();
        props.put("zookeeper.connect", a_zookeeper);
        props.put("group.id", a_groupId);
        props.put("zookeeper.session.timeout.ms", "400");
        props.put("zookeeper.sync.time.ms", "200");
        props.put("auto.commit.interval.ms", "1000");

        return new ConsumerConfig(props);
    }

    public static void main(String[] args) {
        String zooKeeper = args[0];
        String groupId = args[1];
        String topic = args[2];
        int threads = Integer.parseInt(args[3]);

        ConsumerGroupExample example = new ConsumerGroupExample(zooKeeper, groupId, topic);
        example.run(threads);

        try {
            Thread.sleep(10000);
        } catch (InterruptedException ie) {

        }
        example.shutdown();
    }
}
```
*Compile Time Required JARs*

```
kafka_2.8.0-0.8.0-beta1.jar
log4j-1.2.17.jar
metrics-core-2.1.1.jar
scala-library.jar
slf4j-api-1.7.5.jar
zookeeper.jar
zkclient.jar
```

###Simple Consumer:
The main reason to use a SimpleConsumer implementation is you want greater control over partition consumption than Consumer Groups give you.

For example you want to:

1. Read a message multiple times
2. Consume only a subset of the partitions in a topic in a process
3. Manage transactions to make sure a message is processed once and only once

Downsides of using SimpleConsumer:

The SimpleConsumer does require a significant amount of work not needed in the Consumer Groups:

1. You must keep track of the offsets in your application to know where you left off consuming.
2. You must figure out which Broker is the lead Broker for a topic and partition
3. You must handle Broker leader changes

Steps for using a SimpleConsumer:

1. Find an active Broker and find out which Broker is the leader for your topic and partition
2. Determine who the replica Brokers are for your topic and partition
3. Build the request defining what data you are interested in
4. Fetch the data
5. Identify and recover from leader changes

Implementation:

```java
package com.kafka.simpleconsumer.example;

import kafka.api.FetchRequest;
import kafka.api.FetchRequestBuilder;
import kafka.api.PartitionOffsetRequestInfo;
import kafka.common.ErrorMapping;
import kafka.common.TopicAndPartition;
import kafka.javaapi.*;
import kafka.javaapi.consumer.SimpleConsumer;
import kafka.message.MessageAndOffset;

import java.nio.ByteBuffer;
import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

public class SimpleExample {
    public static void main(String args[]) {
        //Usage: com.kafka.simpleconsumer.example.SimpleExample max_num_of_reads topic_name partition_to_read_from broker_hostname broker_port
        SimpleExample example = new SimpleExample();
        long maxReads = Long.parseLong(args[0]);
        String topic = args[1];
        int partition = Integer.parseInt(args[2]);
        List<String> seeds = new ArrayList<String>();
        seeds.add(args[3]);
        int port = Integer.parseInt(args[4]);
        try {
            example.run(maxReads, topic, partition, seeds, port);
        } catch (Exception e) {
            System.out.println("Oops:" + e);
            e.printStackTrace();
        }
    }

    private List<String> m_replicaBrokers = new ArrayList<String>();

    public SimpleExample() {
        m_replicaBrokers = new ArrayList<String>();
    }

    public void run(long a_maxReads, String a_topic, int a_partition, List<String> a_seedBrokers, int a_port) throws Exception {
        // find the meta data about the topic and partition we are interested in
        //
        PartitionMetadata metadata = findLeader(a_seedBrokers, a_port, a_topic, a_partition);
        if (metadata == null) {
            System.out.println("Can't find metadata for Topic and Partition. Exiting");
            return;
        }
        if (metadata.leader() == null) {
            System.out.println("Can't find Leader for Topic and Partition. Exiting");
            return;
        }
        String leadBroker = metadata.leader().host();
        String clientName = "Client_" + a_topic + "_" + a_partition;

        SimpleConsumer consumer = new SimpleConsumer(leadBroker, a_port, 100000, 64 * 1024, clientName);
        long readOffset = getLastOffset(consumer,a_topic, a_partition, kafka.api.OffsetRequest.EarliestTime(), clientName);

        int numErrors = 0;
        while (a_maxReads > 0) {
            if (consumer == null) {
                consumer = new SimpleConsumer(leadBroker, a_port, 100000, 64 * 1024, clientName);
            }
            /**
             * Reading data
             */
            FetchRequest req = new FetchRequestBuilder()
                    .clientId(clientName)
                    .addFetch(a_topic, a_partition, readOffset, 100000)
                    .build();
            FetchResponse fetchResponse = consumer.fetch(req);


            /**
             * SimpleConsumer does not handle lead broker failures, you have to handle it
             *  once the fetch returns an error, we log the reason, close the consumer then try to figure
             *  out who the new leader is
             */
            if (fetchResponse.hasError()) {
                numErrors++;
                // Something went wrong!
                short code = fetchResponse.errorCode(a_topic, a_partition);
                System.out.println("Error fetching data from the Broker:" + leadBroker + " Reason: " + code);
                if (numErrors > 5) break;
                if (code == ErrorMapping.OffsetOutOfRangeCode())  {
                    // We asked for an invalid offset. For simple case ask for the last element to reset
                    readOffset = getLastOffset(consumer,a_topic, a_partition, kafka.api.OffsetRequest.LatestTime(), clientName);
                    continue;
                }
                consumer.close();
                consumer = null;
                leadBroker = findNewLeader(leadBroker, a_topic, a_partition, a_port);
                continue;
            }
            //End Error handling

            /**
             * Reading data cont.
             */
            numErrors = 0;

            long numRead = 0;
            for (MessageAndOffset messageAndOffset : fetchResponse.messageSet(a_topic, a_partition)) {
                long currentOffset = messageAndOffset.offset();
                if (currentOffset < readOffset) {
                    System.out.println("Found an old offset: " + currentOffset + " Expecting: " + readOffset);
                    continue;
                }
                readOffset = messageAndOffset.nextOffset();
                ByteBuffer payload = messageAndOffset.message().payload();

                byte[] bytes = new byte[payload.limit()];
                payload.get(bytes);
                System.out.println(String.valueOf(messageAndOffset.offset()) + ": " + new String(bytes, "UTF-8"));
                numRead++;
                a_maxReads--;
            }

            if (numRead == 0) {
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException ie) {
                }
            }
        }
        if (consumer != null) consumer.close();
    }

    /**
     * Defines where to start reading data from
     *  Helpers Available:
     *      kafka.api.OffsetRequest.EarliestTime() => finds the beginning of the data in the logs and starts streaming
     *                                                from there
     *      kafka.api.OffsetRequest.LatestTime()   => will only stream new messages
     * @param consumer
     * @param topic
     * @param partition
     * @param whichTime
     * @param clientName
     * @return
     */
    public static long getLastOffset(SimpleConsumer consumer, String topic, int partition, long whichTime, String clientName) {
        TopicAndPartition topicAndPartition = new TopicAndPartition(topic, partition);
        Map<TopicAndPartition, PartitionOffsetRequestInfo> requestInfo = new HashMap<TopicAndPartition, PartitionOffsetRequestInfo>();
        requestInfo.put(topicAndPartition, new PartitionOffsetRequestInfo(whichTime, 1));
        kafka.javaapi.OffsetRequest request = new kafka.javaapi.OffsetRequest(
                requestInfo, kafka.api.OffsetRequest.CurrentVersion(), clientName);
        OffsetResponse response = consumer.getOffsetsBefore(request);

        if (response.hasError()) {
            System.out.println("Error fetching data Offset Data the Broker. Reason: " + response.errorCode(topic, partition) );
            return 0;
        }
        long[] offsets = response.offsets(topic, partition);
        return offsets[0];
    }

    /**
     * Uses the findLeader() logic we defined to find the new leader, except here we only try to connect to one of the
     * replicas for the topic/partition. This way if we can’t reach any of the Brokers with the data we are interested
     * in we give up and exit hard.
     * @param a_oldLeader
     * @param a_topic
     * @param a_partition
     * @param a_port
     * @return
     * @throws Exception
     */
    private String findNewLeader(String a_oldLeader, String a_topic, int a_partition, int a_port) throws Exception {
        for (int i = 0; i < 3; i++) {
            boolean goToSleep = false;
            PartitionMetadata metadata = findLeader(m_replicaBrokers, a_port, a_topic, a_partition);
            if (metadata == null) {
                goToSleep = true;
            } else if (metadata.leader() == null) {
                goToSleep = true;
            } else if (a_oldLeader.equalsIgnoreCase(metadata.leader().host()) && i == 0) {
                // first time through if the leader hasn't changed give ZooKeeper a second to recover
                // second time, assume the broker did recover before failover, or it was a non-Broker issue
                //
                goToSleep = true;
            } else {
                return metadata.leader().host();
            }
            if (goToSleep) {
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException ie) {
                }
            }
        }
        System.out.println("Unable to find new leader after Broker failure. Exiting");
        throw new Exception("Unable to find new leader after Broker failure. Exiting");
    }

    /**
     * Query a live broker to find out leader information and replica information for a given topic and partition
     * @param a_seedBrokers
     * @param a_port
     * @param a_topic
     * @param a_partition
     * @return
     */
    private PartitionMetadata findLeader(List<String> a_seedBrokers, int a_port, String a_topic, int a_partition) {
        PartitionMetadata returnMetaData = null;
        for (String seed : a_seedBrokers) {
            SimpleConsumer consumer = null;
            try {
                consumer = new SimpleConsumer(seed, a_port, 100000, 64 * 1024, "leaderLookup"); //broker_host, broker_port, timeout, buffer_size, client_id
                List<String> topics = new ArrayList<String>();
                topics.add(a_topic);
                TopicMetadataRequest req = new TopicMetadataRequest(topics);
                kafka.javaapi.TopicMetadataResponse resp = consumer.send(req);

                //call to topicsMetadata() asks the Broker you are connected to for all the details about the topic we are interested in
                List<TopicMetadata> metaData = resp.topicsMetadata();
                //loop on partitionsMetadata iterates through all the partitions until we find the one we want.
                for (TopicMetadata item : metaData) {
                    for (PartitionMetadata part : item.partitionsMetadata()) {
                        if (part.partitionId() == a_partition) {
                            returnMetaData = part;
                            break;
                        }
                    }
                }
            } catch (Exception e) {
                System.out.println("Error communicating with Broker [" + seed + "] to find Leader for [" + a_topic
                        + ", " + a_partition + "] Reason: " + e);
            } finally {
                if (consumer != null) consumer.close();
            }
        }
        // add replica broker info to m_replicaBrokers
        if (returnMetaData != null) {
            m_replicaBrokers.clear();
            for (kafka.cluster.Broker replica : returnMetaData.replicas()) {
                m_replicaBrokers.add(replica.host());
            }
        }
        return returnMetaData;
    }
}
```
Scala Producers & Consumers:
---------------------------

**Producer Example**

```scala
import kafka.producer.{ProducerConfig,Producer,ProducerData}
import java.util.properties

object BasicProducer extends App {
    val options = Array("1. Send single message", "2. Send multiple messages")
    var continue: Boolean = true
    while(continue) {
      options.foreach(o => println(o))
      val option = readLine()
      continue = option != null
      if(continue) {
        option match {
          case "1" =>
            sendSingleMessage
          case "2" =>
            sendMultipleMessages
          case _ =>
            println("ERROR. Invalid option. Exiting...")
            continue = false
        }
    }

    def sendSingleMessage {
        val props = new Properties
        props.put("zk.connect", "127.0.0.1:2181")
        props.put("serializer.class", "kafka.serializer.StringEncoder")
        val config = new ProducerConfig(props)
        val producer = new Producer[String, String](config)

        // send a single message
        val producerData = new ProducerData[String, String]("test-topic", "test-message")
        producer.send(producerData)
        producer.close
        println("\nSent message -> test-message\n")
    }

    def sendMultipleMessages {
        val props = new Properties
        props.put("zk.connect", "127.0.0.1:2181")
        props.put("serializer.class", "kafka.serializer.StringEncoder")
        val config = new ProducerConfig(props)
        val producer = new Producer[String, String](config)

        // send a single message
        val messages = Array("test-message1", "test-message2")
        val producerData = new ProducerData[String, String]("test-topic", messages)

        producer.send(producerData)
        producer.close
        println("\nSent messages -> test-message1, test-message2\n")
    }    
}
```

**Producer with partitioner**

```scala
import kafka.producer.{ProducerConfig,Producer,ProducerData,Partitioner}
import java.util.Properties

object ProducerPartitionerExample extends App {
    val props = new Properties
    props.put("zk.connect", "127.0.0.1:2181");
    props.put("serializer.class", "scalathon.kafka.producer.MemberRecordEncoder");
    props.put("partitioner.class", "scalathon.kafka.producer.MemberLocationPartitioner")

    val config = new ProducerConfig(props);
    val producer = new Producer[String, MemberRecord](config);

    // send a single message
    val recordsUS = Array[MemberRecord](new MemberRecord(1, "John", "US"), new MemberRecord(2, "Joe", "US"))
    val dataUS = new ProducerData[String, MemberRecord]("member-records", "US", recordsUS)

    producer.send(dataUS)
    println("\nSent messages with key = US to topic member-records\n")

    val recordsEU = Array[MemberRecord](new MemberRecord(3, "John", "Europe"), new MemberRecord(4, "Joe", "Europe"))
    val dataEU = new ProducerData[String, MemberRecord]("member-records", "EUR", recordsEU)

    producer.send(dataEU)
    println("\nSent messages with key = EUR to topic member-records\n")
    producer.close
}

class MemberLocationPartitioner extends Partitioner[String] {
    def partition(location: String, numPartitions: Int): Int = {
        val ret = location.hashCode % numPartitions
        ret
    }
}
```

**Basic Consumer**

```scala
import java.util.Properties
import kafka.consumer.{Consumer, ConsumerConnector, ConsumerConfig}
import java.util.concurrent.Executors
import org.I0Itec.zkclient._
import kafka.utils.StringSerializer

object ConsumerExample extends App {
    val props = new Properties
    props.put("zk.connect", "127.0.0.1:2181")
    props.put("groupid", "test-group")

    // set the topic
    val topic = "test-topic"
    // set the number of consumer streams for this topic
    val partitions = 1

    val consumerConfig = new ConsumerConfig(props)
    val consumerConnector: ConsumerConnector = Consumer.create(consumerConfig)
    // create the consumer streams
    val topicMessageStreams = consumerConnector.createMessageStreams(Predef.Map(topic -> partitions))

    // get the streams for the "test-topic" topic
    val testTopicStreams = topicMessageStreams.get(topic).get

    val executor = Executors.newFixedThreadPool(partitions)

    for(stream <- testTopicStreams) {
      executor.execute(new Runnable() {
        override def run() {
          for(message <- stream) {
            // process message
            println("\nConsumed " + kafka.utils.Utils.toString(message.payload, "UTF-8"))
          }
        }
      });
    }

    // attach shutdown handler to catch control-c
    Runtime.getRuntime().addShutdownHook(new Thread() {
      override def run() = {
        consumerConnector.shutdown
        executor.shutdownNow()
        println("\nConsumer threads shutted down")
      }
    })
}
```

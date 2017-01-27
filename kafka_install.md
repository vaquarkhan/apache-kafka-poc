#Kafka - Messaging Basics
This assumes you are starting fresh and have no existing Kafka or ZooKeeper data. See http://kafka.apache.org/documentation.html#quickstart for more details.

##Install Java 

```
sudo apt-get install python-software-properties
sudo add-apt-repository ppa:webupd8team/java
sudo apt-get update
sudo apt-get install oracle-java8-installer
```

##Download Kafka
Download version 0.8.1.

```
cd /usr/src/
wget http://ftp.heanet.ie/mirrors/www.apache.org/dist/kafka/0.8.1/kafka_2.9.2-0.8.1.tgz
tar -xzf kafka_2.9.2-0.8.1.tgz
cd kafka_2.9.2-0.8.1
```

##Running Kafka
Kafka uses <a href="http://zookeeper.apache.org/">Zookeeper</a> to provide a centralized service for maintaining configuration information, naming, providing distributed synchronization, and providing group services.

###To Start Zookeeper

    bin/zookeeper-server-start.sh config/zookeeper.properties

###To Start Kafka

    bin/kafka-server-start.sh config/server.properties

###Create a Topic
This will create a topic called Test

    bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 --partitions 1 --topic test

This topic can then be view using:

    bin/kafka-topics.sh --list --zookeeper localhost:2181

Alternatively, instead of manually creating topics you can also configure your brokers to auto-create topics when a non-existent topic is published to.

###Create Messages - Producer
Kafka comes with a command line client that will take input from a file or from standard input and send it out as messages to the Kafka cluster. By default each line will be sent as a separate message.

Run the producer and then type a few messages into the console to send to the server. 

    bin/kafka-console-producer.sh --broker-list localhost:9092 --topic test
    This is a message
    This is another message
    And so is this

###Read Messages - Consumer
Kafka also has a command line consumer that will dump out messages to standard output.

    bin/kafka-console-consumer.sh --zookeeper localhost:2181 --topic test --from-beginning
    This is a message
    This is another message
    And so is this

##Setting up a multi-broker cluster
Kafka can run with multiple brokers.

Lets make it a three broker cluster

    cp config/server.properties config/server-1.properties 
    cp config/server.properties config/server-2.properties

Now edit these new files and set the following properties: 

config/server-1.properties:
```
    broker.id=1
    port=9093
    log.dir=/tmp/kafka-logs-1
```
 
config/server-2.properties:
```
    broker.id=2
    port=9094
    log.dir=/tmp/kafka-logs-2
```
Each kafka instance has a separate port (default is 9092).

The broker.id property is the unique and permanent name of each node in the cluster. We have to override the port and log directory only because we are running these all on the same machine and we want to keep the brokers from all trying to register on the same port or overwrite each others data.

We already have Zookeeper and our single node started, so we just need to start the two new nodes: 

    bin/kafka-server-start.sh config/server-1.properties &
    bin/kafka-server-start.sh config/server-2.properties &

Now create a new topic with a replication factor of three

    bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 3 --partitions 1 --topic my-replicated-topic

###How to see what the broker is doing

    bin/kafka-topics.sh --describe --zookeeper localhost:2181 --topic my-replicated-topic

This returns:

    Topic:my-replicated-topic	PartitionCount:1	ReplicationFactor:3	Configs:
    	    Topic: my-replicated-topic	Partition: 0	Leader: 2	Replicas: 2,1,0	Isr: 2,1,0


Here is an explanation of output. The first line gives a summary of all the partitions, each additional line gives information about one partition. Since we have only two partitions for this topic there are only two lines.

    *<strong>leader</strong> is the node responsible for all reads and writes for the given partition. Each node will be the leader for a randomly selected portion of the partitions.
    *<strong>replicas</strong> is the list of nodes that replicate the log for this partition regardless of whether they are the leader or even if they are currently alive.
    *<strong>isr</strong> is the set of "in-sync" replicas. This is the subset of the replicas list that is currently alive and caught-up to the leader. 

We can run the same command on the original topic we created to see where it is: 

    bin/kafka-topics.sh --describe --zookeeper localhost:2181 --topic test

This returns:

    Topic:test	PartitionCount:1	ReplicationFactor:1	Configs:
	Topic: test	Partition: 0	Leader: 0	Replicas: 0	Isr: 0

The original topic has no replicas and is on server 0, which was the only server in our cluster when we created it. 


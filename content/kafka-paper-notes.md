+++
title = "Kafka Paper Notes"
description = "My notes on the Kafka Paper"
draft = false
date = "2022-07-22"
template = "page.html"
menu = "sandesh"
+++

> [Paper Link](http://notes.stephenholiday.com/Kafka.pdf)

### What is kafka?

Kafka is a distributed messaging system for Log Processing.

### Notes:
#### The issues with traditional enterprise messaging systems: 
- Delivery guarantees are stringent (ironically, Kafka supports all kinds of delivery semantics - `At least once`, `At most once` and `Exactly once`)
- Guarantees complicate the API and throughput is a second class citizen in the hierarchy of non functional requirements
- Lack of distributed support across multiple machines
- They require that messages be consumed parallely with creation; persistent buffers degrades performance of the entire system
- They use a "push" system to push data. Instead a "pull" system is more suitable since no implementation of backpressure is required and message rewinding is possible this way.

#### Kafka vocabulary: 
1. **Topic** - The grouping parameter by which incoming logs are grouped. For e.g. `Sales`, `Inventory`, etc.
2. **Partition** - A single sharded instance of a Topic. One topic can have multiple partitions. 
3. **Broker** - A server / node that holds kafka data
4. **Consumer** - A server that consumes data from brokers
5. **Producer** - A server that pushes data to Brokers
6. **Message** - A collection of bytes that represents a log
7. **MessageSet** - A set of messages


#### The effectiveness of kafka:
This section poses architectural design decisions that kafka took: 
##### Log Storage:
- Kafka stores all logs into segment files of a predefined length. When one length is exceeded, another file is created. This way of sequential write makes kafka very fast.
- All messages are first written to segment files and only _then_ can they be accessed by consumers. 
- Each message in a segment file is represented by its offset. There are no message ids, just offsets. This reduces the pain of maintaining a separate index process for pointing messages to their offsets. 
- A partition is built up of multiple segment files. The files are retained until a predefined interval if no reads happen. 
- Requesting an offset within a segment means the consumer has already received all messages before that offset. 
- Since streaming is real time, consumer and producer lag with a very small offset. Utilizing this important feature, Kafka does not maintain any caches of messages. Instead, it relies on the Operating system's page cache to make operations more efficient. For e.g. if messages [a, b, c, d] are sent, and [a, b, c] are received, then [ d ] is already in the page cache!
- Kafka uses `sendfile` API of linux to send data to consumers from segment files.
- The broker does not maintain any information about how much data has been read by the consumer. The consumer maintains this data on a zookeeper directory (more on this later). This frees up the Broker from deleting the messages, as they are already covered by the SLA under which the retention is defined. 
- Consumers can replay messages

##### Distributed Coordination: 
- Consumers are grouped in consumer groups. Each message is delivered to one and only one consumer within the group.
- Messages are stored in segment files and segment files are sharded into partitions. Each partition is "owned" by a single consumer. This frees up any file-coordination requirements between consumers.
- More partitions than consumers is beneficial to the overall architecture to balance load effectively. This is achieved by "over partitioning" a topic. 

**Zookeeper semantics:**
- Zookeeper contains the following registries: broker, consumer, ownership, offset. The first three are ephemeral and the last one is persistent. 
- Broker registry contains Broker's hostname, port and topics and partitions contained inside it. 
- Consumer registry contains the consumer groups and the set of topics subscribed to it and link to ownership registry to maintain last read offsets. 
- Ownership registry is associated with consumers and tells us which partition is owned by which consumer. 
- Offset registry maintains the partitions' last-read offset. This is useful in case a consumer fails, we can spawn another one and it can continue to read information from there.
- Watchers are implemented on Broker, Consumer registry by each consumer for the above purpose. 
- When a new consumer group is spawned, they can read data from the largest or the smallest offset in each partition. This is defined in configuration. 

##### Delivery Guarantees:
- No de-duplication supported by default
- Messages from a single partition is delivered in-order to a consumer. Messages from multiple partitions and single topic have no such guarantees. 
- Kafka stores a CRC for each message in the log to avoid data corruption. 
- If a broker goes down **and** its storage fails, then the messages are lost forever. 

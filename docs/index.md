# Distributed systems

Build a kafka like publish/subscription application in Rust, to study some aspects of distributed system added to a more low level system programming.

## Topics

### Single node

 - [The first lines of code - a tcp service](initial_tcp_server.md)
 - multiple consumers
 - write ahead log
 - set a offset to each message - finding messages quickly
 - keep offset of consumers
 - improving throughput through partitions
 - message ordering and mutiple paritions - how to deal with that
 - read offset strategies - ranges and commit offset
 - partition replication - single leader strategy
 - write strategies - single send, one cofirm, two confirms

### Multiple nodes

 - leader node
 - ping alive
 - zookeeper
 - change leader


### Event driven architecture

## References

  - Designing Data-Intensive Applications: The Big Ideas Behind Reliable, Scalable, and Maintainable Systems, from Martin Kleppmann
  - Kafka: The Definitive Guide: Real-Time Data and Stream Processing at Scale, from Gwen Shapira, Neha Narkhede, and Todd Palino

# Allowing multiple consumers groups

Following Kafka design, we can introduce the ability to have multiple consumers through the concept of consumer groups. There, each partition can have a single consumer per consumer group. A consumer can take care of more than a partition, but, if you want to have multiple consumers to improve throughput, you must have more partitions.

We are going there, but not now. First, we must be able to allow more than one consumer group to connect to our service and receive all messages, without competign for then.

The solution we will use here is to add a offset to each content and allow consumers to choose what offset range they are going to get. To achieve that, we will need to change ou data structure to something that can deal with ordered keys associated with the produced content.
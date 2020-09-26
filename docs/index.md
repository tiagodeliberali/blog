# Distributed systems

Build a kafka like publish/subscription application in Rust, to study some aspects of distributed system added to a more low level system programming.

## Topics

### Single node

 - [The first lines of code - a tcp service](initial_tcp_server.md)
 - [Allowing multiple consumers](multiple_consumers.md)
 - reliability - adding checksum to messages
 - write ahead log
 - read offset strategies - ranges and commit offset
 - write strategies - single send, one cofirm, two confirms

### Multiple nodes

 - [Electing the controller](electing_controller.md)
 - partition replication
 - leader node
 - ping alive
 - zookeeper
 - change leader

### Event driven architecture

 - how to use this stuff?


## References

  - Designing Data-Intensive Applications: The Big Ideas Behind Reliable, Scalable, and Maintainable Systems, from Martin Kleppmann
  - Kafka: The Definitive Guide: Real-Time Data and Stream Processing at Scale, from Gwen Shapira, Neha Narkhede, and Todd Palino




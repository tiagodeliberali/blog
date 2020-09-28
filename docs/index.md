# Distributed systems

Build a Kafka like publish/subscription application in Rust to study aspects of the distributed system added to a more low-level system programming.

## Topics

### Single node

 - [The first lines of code - a TCP service](initial_tcp_server.md)
 - [Allowing multiple consumers](multiple_consumers.md)
 - reliability - adding a checksum to messages
 - write-ahead log
 - read offset strategies - ranges and commit offset
 - write strategies - single send, one confirms, two confirms

### Multiple nodes

 - [Electing the controller](electing_controller.md)
 - partition replication

### Event driven architecture

 - how to use this stuff?


## References

  - Designing Data-Intensive Applications: The Big Ideas Behind Reliable, Scalable, and Maintainable Systems, from Martin Kleppmann
  - Kafka: The Definitive Guide: Real-Time Data and Stream Processing at Scale, from Gwen Shapira, Neha Narkhede, and Todd Palino

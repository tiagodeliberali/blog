## Distributed systems

Build a kafka like in Rust, to study some aspects of distributed system added to a more low level system programming.

## Topics

### Single node

 - storing messages in a write ahead log
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

 ## References

  - Designing Data-Intensive Applications: The Big Ideas Behind Reliable, Scalable, and Maintainable Systems, from Martin Kleppmann
  - Kafka: The Definitive Guide: Real-Time Data and Stream Processing at Scale, from Gwen Shapira, Neha Narkhede, and Todd Palino
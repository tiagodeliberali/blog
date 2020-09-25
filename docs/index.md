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

 - Electing the controller
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



# Take me to your leader

We are going to go deep inside distributed system, but in baby steps. The first concept to understand is the controller. In a cluster, we have a set of brokers, but one of them has a special role: the controller. This includes things like:

- monitor brokers liveness
- elect new partition leaders when necessary
- communicate other brokers about changes
- manage metadata about where each partition leader is stored and which replicas are in sync


## Controller election

But how it is elected? Today, Kafka rely strongly on Zookeeper to controller election. So, it is important to understand two concepts of zookeeper that controls this election.

First, we have the concept of session. When a clinet connects to zookeper, it also start to send a heartbeat to inform zookeeper that it is alive. Also, we have the ephemeral node. This node keeps information added from a client while its session is alive. So, once a brokers crash, close connection or even blocks in garbage collection (something done by not so loved languages), all ephemeral nodes are deleted.

So, here is the election process: all brocker just to create a `/controller` ephemeral node to zookeeper. The first one wins. The other will receive an error of `node already exists"`. As easy as that. All brockers also subscribes to changes in the controler node. So, if something bad happens to the current controller, zookeper will delete the controler node, all other brokers will be nitified about this removal and will try to create the controller node again.

Be aware that things are changing and [Zookeeper is not going to be used by kafka soon](https://www.confluent.io/blog/removing-zookeeper-dependency-in-kafka/).

## How are we going to do the elections?

...
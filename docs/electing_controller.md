# Take me to your leader

We are going to go deep inside the distributed system but in baby steps. The first concept to understand is the controller. In a cluster, we have a set of brokers, but one of them has a special role: the controller. This [includes things](https://www.youtube.com/watch?v=li2aowPnezA) like:

- Monitor brokers liveness
- Elect new partition leaders when necessary
- Communicate other brokers about changes
- Manage metadata about where each partition leader is stored and which replicas are in sync


## Controller election

But how it is elected? Today, Kafka relies strongly on Apache ZooKeeper to controller election. So, it is important to understand two concepts of ZooKeeper that control this election.

First, we have the concept of a session. When a client connects to ZooKeeper, it also starts to send a heartbeat to inform ZooKeeper that it is alive. Also, we have the ephemeral node. This node keeps information added from a client while its session is alive. So, once a brokers crash, close a connection, or even blocks in garbage collection (something done by not so loved languages), all ephemeral nodes are deleted.

So, here is the election process: all broker try to create a `/controller` ephemeral node to ZooKeeper and the first one wins. The other will receive an error of `node already exists"`. As easy as that. All brokers also subscribe to changes in the controller node. So, if something bad happens to the current controller, ZooKeeper will delete the controller node, all other brokers will be notified about this removal and will try to create the controller node again.

Be aware that things are changing and [Kafka will stop using ZooKeeper soon](https://www.confluent.io/blog/removing-ZooKeeper-dependency-in-kafka/).


## Once upon a time...

Let's tell the story of a set of brokers with an elected controller and its relations with clients. Each broker will have a set of topics/partitions where some of them are replicas and others are leaders. Only the leader will accept requests to save or return content. Each partition has only one leader and zero or more replicas. So, when a producer wants to send content to our service, it must know which brokers have the partition leader it wants to send the content. Let's leave the consumer story for the future since it is a little bit more complicated.

How the producer can have the list of partition leaders and brokers? Well, it must ask. So, it will ask one of the known brokers passing a list of topics and will receive as response a list of partition leaders and brokers. Now, it can store those values and use them to produce content.

To make this beautiful story true, we must have few mechanics:

- The controller is responsible for creating topic/partition overall brokers and distribute leader and replicas in a reasonable way
- The controller also needs to keep one eye on each broker and be aware when one of them fails
- In a failure situation, it must assign a new leader for each partition that had a leader on the dead broker
- We must have a trustful source of partition leader and brokers

Sure it does not cover everything, but it is a coherent start. From now, we are going to imagine things and try by ourselves ways to implement these mechanics, not necessarily copying Kafka implementation, but consulting it whenever it looks a good option.

## How are we going to do the elections?

...
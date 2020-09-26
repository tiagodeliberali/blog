# Take me to your leader

We are going to go deep inside distributed system, but in baby steps. The first concept to understand is the controller. In a cluster, we have a set of brokers, but one of them has a special role: the controller. This includes things like:

- Monitor brokers liveness
- Elect new partition leaders when necessary
- Communicate other brokers about changes
- Manage metadata about where each partition leader is stored and which replicas are in sync


## Controller election

But how it is elected? Today, Kafka rely strongly on Zookeeper to controller election. So, it is important to understand two concepts of zookeeper that controls this election.

First, we have the concept of session. When a clinet connects to zookeper, it also start to send a heartbeat to inform zookeeper that it is alive. Also, we have the ephemeral node. This node keeps information added from a client while its session is alive. So, once a brokers crash, close connection or even blocks in garbage collection (something done by not so loved languages), all ephemeral nodes are deleted.

So, here is the election process: all brocker try to create a `/controller` ephemeral node to zookeeper and the first one wins. The other will receive an error of `node already exists"`. As easy as that. All brockers also subscribes to changes in the controler node. So, if something bad happens to the current controller, zookeper will delete the controler node, all other brokers will be nitified about this removal and will try to create the controller node again.

Be aware that things are changing and [Kafka will stop using Zookeeper soon](https://www.confluent.io/blog/removing-zookeeper-dependency-in-kafka/).


## Once upon a time...

Let's tell the story of a set of brokers with an elected controller and its relations with clients. Each broker will have a set of topic/partitions where some of them are replicas and others are leaders. Only the leader will accept requests to save or return content. Each partition has only one leader and zero or more replicas. So, when a producer wants to send content to our service, it must know which brokers has the partition leader it wants to send the content. Let's leave the consumer story for the future, since it is a little bit more complicated.

How the producer can have the list of partition leaders and brokers? Well, it must ask. So, it will ask one of the known brokers passing a list of topics and will receive as response a list of partition leaders and brokers. Now, it can store those values and use to produce content.

To make this beatiful story true, we must have few mechanics:

- The controller is responsible for creating topic/partition over all brokers and distribute leader and replicas in a reasonable way
- The controller also needs to keep one eye on each broker and be aware when one of they fail
- In failure situation, it must assign a new leader for each partititon that had a leader on the dead broker
- We must have a trustfull source of partition leader and brokers


## How are we going to do the elections?

...
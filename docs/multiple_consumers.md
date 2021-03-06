# Toward multiple consumers!

Sure we should be proud of our initial service! It allows producers and consumers to flow messages, almost like a queue. But, we must be honest: it is far from a queue and far from Kafka. So, let's take our road toward Kafka! Let's allow multiple consumers!

One aspect of Kafka design we will pay attention to now is its ability to have multiple consumers to get the same set of messages. We will achieve that replacing the data structure adopted in the [last session](https://tiagodeliberali.github.io/blog/initial_tcp_server.html) by a `Vec<Content>`. Also, we will create the topic/partition schema and take care of mutability and sync between threads.

So, our next challenges are:

- Change the data structure to stop dropping a message from memory on consumption
- Create a topic/partition structure
- Deal with multithread issues

## Housekeeping first

Since our system is growing (they grow up so fast!), we need to introduce some concepts to make things manageable. To have a place to put global structs, we introduced the `core` module and added a few basic types:

```rust
pub struct OffsetValue(pub u32);

pub struct TopicAddress {
    pub name: String,
    pub partition: u32,
}

#[derive(Clone)]
pub struct Content {
    pub value: String,
}
```


## Improved communication

An area that needed more attention is the `communication` module. Here, we have a place to deal with byte streams, define the actions the system can deal with, and the responses it can return. We defined our binary protocol in a way that it will be easy to maintain and expand. Also, we added some tests to preserve functionality since it is an area that is better defined now.

```rust
pub enum Action {
    Produce(TopicAddress, Vec<Content>),
    Consume(TopicAddress, OffsetValue, u32),
    CreateTopic(String, u32),
    Quit,
    Invalid,
}

pub struct ActionMessage {
    pub action: Action,
    pub consumer_id: String,
}

pub enum Response {
    Empty,
    Offset(OffsetValue),
    Content(OffsetValue, Content),
    Error,
}

pub struct ResponseMessage {
    pub response: Response,
}
```

A deep dive into our communication module can show us how organized things can get. To give you some details about that ([you can check our repo](https://github.com/tiagodeliberali/logstreamer/releases/tag/0.2.0) to see everything), here are our `parse` and `to_vec` associated with the `ActionMessage`:

```rust
    pub fn parse(buffer: &[u8]) -> ActionMessage {
        let mut data = Buffer::new(buffer);

        let action = match data.read_u8() {
            1 => {
                let topic = TopicAddress::new(data.read_string(), data.read_u32());
                let content_length = data.read_u32();
                let mut content_list = Vec::new();
                for _ in 0..content_length {
                    content_list.push(Content::new(data.read_string()))
                }
                Action::Produce(topic, content_list)
            }
            2 => {
                let topic = TopicAddress::new(data.read_string(), data.read_u32());
                let offset = OffsetValue(data.read_u32());
                let limit = data.read_u32();
                Action::Consume(topic, offset, limit)
            }
            3 => {
                let topic = data.read_string();
                let partition = data.read_u32();
                Action::CreateTopic(topic, partition)
            }
            4 => Action::Quit,
            _ => Action::Invalid,
        };

        let consumer_id = data.read_string();

        ActionMessage {
            action,
            consumer_id,
        }
    }

    pub fn as_vec(&self) -> Vec<u8> {
        let mut content_vec: Vec<u8> = Vec::new();

        match &self.action {
            Action::Produce(topic, content_list) => {
                content_vec.push(1);
                write_string(&mut content_vec, topic.name.clone());
                write_u32(&mut content_vec, topic.partition);
                write_u32(&mut content_vec, content_list.len() as u32);
                for content in content_list {
                    write_string(&mut content_vec, content.value.clone());
                }
            }
            Action::Consume(topic, offset, limit) => {
                content_vec.push(2);
                write_string(&mut content_vec, topic.name.clone());
                write_u32(&mut content_vec, topic.partition);
                write_u32(&mut content_vec, offset.0);
                write_u32(&mut content_vec, *limit);
            }
            Action::CreateTopic(topic, partition) => {
                content_vec.push(3);
                write_string(&mut content_vec, topic.clone());
                write_u32(&mut content_vec, *partition);
            }
            Action::Quit => content_vec.push(4),
            Action::Invalid => content_vec.push(0),
        }

        write_string(&mut content_vec, self.consumer_id.clone());

        content_vec
    }
```

The magic happens inside `Buffer`, a small helper struct we created to maintain the cursor position we consumed from our `u8 array`. In this way, each call to `read_string`, `read_u32`, or `read_u8` gives us a value while allows us to navigate inside the buffer. The other two functions, `write_string` and `write_u32`, are helper functions that respect the schema proposed by our binary communication protocol.


## Multithread storage sync

We created another important module to handle `storage` entities. In this way, we can split the logic of storing data, currently associated with a single partition, and introduce the code associated with the topic/partition organization. In our system, `Cluster` will keep a `HashMap` with topic names and `Vec's of partitions. Each `Partition` is responsible for a `Vec` of contents and its own interior mutability.

```rust
pub struct Cluster {
    topics: RwLock<HashMap<String, Vec<Arc<Partition>>>>,
}

pub struct Partition {
    pub queue: Mutex<Vec<Content>>,
}
```

Rust brings us the concept of [fearless concurrency](https://doc.rust-lang.org/book/ch16-00-concurrency.html) and some alternatives to model this kind of systems. Our initial design will be based on [Shared-State Concurrency](https://doc.rust-lang.org/book/ch16-03-shared-state.html) and the use of locks and multiple ownership types. To try to make the best from locks and wait time, we can adopt different strategies. 

In our case, we put out a collection of topics inside a `RwLock`. This sync structure has split `read` and a `write` lock, allowing multiple threads to read simultaneously or a single thread to write. Our system will write to this list to add a new topic, something very rare compared to other operations. So, it is perfect here.

Next, each `Partition` is wrapped by `Arc`. `Arc` allows us to share [multiple references to a value allocated in a heap](https://doc.rust-lang.org/beta/std/sync/struct.Arc.html). Since `Partition` has its own internal mutability mechanics, we can clone references to it, which is a cheap operation.

Inside partition, we choose to use a `Mutex`. Since a partition is a place where reads and writes occur in about the same frequency, we should not prioritize read or write as we are dealing with data streaming. At first, we can see a `RwLock` as a better option, because it could allow multiple reads at the same time, but it depends on OS specifics and, in Linux, [it looks like it prioritizes reads, making the write access to the lock scarce](https://stackoverflow.com/questions/56924866/why-do-rust-mutexes-not-seem-to-give-the-lock-to-the-thread-that-wanted-to-lock).

How bad can it be to ignore all this stuff and use a Mutex on `handle_connection`? Well, we can try it out and see by ourselves. I did a test with the following scenario:

- 10 producers producing 200k msgs each
- 10 consumers
- Batches of 30 messages to consume and produce
- Everything working in parallel

With a simple mutex, this scenario took about `380s to produce` and `600s to consume`. When we changed to our implementation, it took about `1 to 2s to produce and consume`. So, yes, it worth spending some time thinking about this stuff.

## A more realistic cluster

In the end, we have something that looks a little bit more like Kafka. Since we introduced the topic/partition structure, we added a `CreateTopic` action, and now we have something more serious here. Now we can:

- Create a topic with a defined number of partitions
- Produce and consume to specific partitions inside the topic
- Send and receive batches of content
- Keep consumers and producers connected working in parallel

Here is the new version of `handle_connection`:

```rust
fn handle_connection(mut stream: TcpStream, cluster: Arc<Cluster>) {
    loop {
        let mut buffer = [0; 1024];
        let _ = match stream.read(&mut buffer) {
            Ok(value) => value,
            Err(err) => {
                println!("Failed to read stream\r\n{}", err);
                return;
            }
        };

        let message = ActionMessage::parse(&buffer);

        let response_list = match message.action {
            Action::Produce(topic, content) => store_data(topic, content, cluster.clone()),
            Action::Consume(topic, offset, limit) => {
                read_data(topic, offset, limit, cluster.clone())
            }
            Action::CreateTopic(topic, partition_number) => {
                add_topic(topic, partition_number, cluster.clone())
            }
            Action::Quit => return,
            Action::Invalid => vec![ResponseMessage::new_empty()],
        };

        let mut response_content: Vec<u8> = Vec::new();
        for response in response_list {
            response_content.extend(response.as_vec());
        }

        if response_content.is_empty() {
            response_content.extend(ResponseMessage::new_empty().as_vec());
        }

        stream.write_all(&response_content[..]).unwrap();
        stream.flush().unwrap();
    }
}
```

### Faster than ever

After so many changes and the introduction of many structs and concepts, how fast can it be? Well, again, we made some changes to the `client_test` to get some numbers. For our new test scenario, we have:

- 1 topic with 10 partitions
- 10 producers creating 2M messages each to different partitions
- 10 consumers reading from different partitions
- Batches of 30 messages on producers and consumers

So, to `produce and consume 20 million messages`, we took '16s on average`! Well, it is about `1.25 million messages per second`! A great number, for sure!

The interesting part is that we could expand our system's capacity, allowing the creation of multiple partitions to a single topic. It helps because we can introduce a real parallelization of the work. Since each partition has its own lock system, we reduce the competition to own the Vec to read and write content.

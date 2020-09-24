# Toward multiple consumers!

Sure we should be proud about our initial service! It allow producers and consumer to flow messages, almost like a queue. But, we must be honest: it is far from a queue and far from Kafka as well. So, let's take our road toward Kafka! Let's allow multiple consumers!

One aspect of Kafka design we wiil pay attention now is its ability to have multiple consumers to get the same set of messages. We will achieve that adding an offset, that will be a sequential id, to each message. Each consumer will be responsible for managing its own offset, that will be sent to the server togheter with the number of messages to be received.

So, our next challeges are:

- Set an offset to each message produced
- Change the data structure to stop dropping message from memory on consumption

## House keeping first

To improve of our first version, we can start by moving the code related to communication to its own modules. Also, we can specify requests and responses, to make our tcp server easier to maintain. Also, we will introduce more types to make things easier to reason about in the future. 

Our set of types in a `core` module:

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

And our new `communication` module with actions and responses to make it simple to deal with buffers:

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

By this way, will be easier to add more request and response types. For each message, we specified a `parse` and a `to_vec` method, encapsulating the ability to build messages based on byte arrays.

To give you some details about that (you can check our repo to see everything), here are our `parse` and `to_vec` associated to the `ActionMessage`:

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

The magic happens inside `Buffer`, a small helper struct we use to maintain the cursor position we consumed from our u8 array. By this way, each call to `read_string`, `read_u32` or `read_u8` garantee that we are going to navigate inside the buffer while extract information. The other two funcions, `write_string` and `write_u32`, are helper functions that respect the schema proposed by our binary communication protocol.

We can create another important module to handle `storage` entities. By this way, we can split the logic of storing data, currently associated to a single parititon, and the logic associated with the topic/partition organization.

```rust
pub struct Cluster {
    topics: RwLock<HashMap<String, Vec<Arc<Partition>>>>,
}

pub struct Partition {
    pub queue: Mutex<Vec<Content>>,
}
```

When dealing with multithread, we must be careful with mutability of strucutures. To try to reduce locks, we can adopt a set of different strategies. First, we put out collection of topics inside a `RwLock`. This sync strucuture have a `read` and a `write` lock, allowing multiple threads to read in parallel or a single thread to write. Our system will just write to this list in caso of adding a channel, something very rare compared to other operations. So, it is perfect here.

Next, we have a `HashMap` associating topics to a collection of `Partition`, evolved by `Arc`. `Arc` allow us to share [multiple references to a value allocated in heap](https://doc.rust-lang.org/beta/std/sync/struct.Arc.html). Since `Partition` has its own internal mutability mechanics, we can just clone references to it, wich is an inespensive operation.

Inside partition, we choose to use a `Mutex`. Since a partition is a place where reads and writes occur in about the same frequency, as we are dealing with data streaming, we should not prioritize read nor write. At first, we can see a `RwLock` as a better option, because it could allow multiple reads at the same time, but it depends on OS specifics and, in Linux, it looks like it prioritizes reads making to write much slow.

How bad can be to ignore all this stuff and just use a Mutex on `handle_connection`? Well, a lot! I did a test with the following scenario:

- 10 producers producing 200k msgs
- 10 consumers
- batchs of 30 messages to consume and produce

With a simple mutex, this scenario took about `380s to produce` and `600s to consume` while using the proposed strategy it took about `1s to produce and consume`. So, yes, it worth to spend some time thinking about this stuff.


```rust
fn handle_connection(mut stream: TcpStream, storage: Arc<Storage>) {
    loop {
        let mut buffer = [0; 512];
        let _ = match stream.read(&mut buffer) {
            Ok(value) => value,
            Err(err) => {
                println!("Failed to read stream\r\n{}", err);
                return;
            }
        };

        let message = ActionMessage::parse(&buffer);

        let response_list = match message.action {
            Action::Produce(content) => store_data(content, storage.clone()),
            Action::Consume(offset, limit) => read_data(offset, limit, storage.clone()),
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
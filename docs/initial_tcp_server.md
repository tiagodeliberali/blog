# Let's rebuild Kafka in Rust!

So, here is a nice challenge. Let's build an incredible distributed event streaming system copying the current most famous one! :)

Why should we spend time doing that? Well, first of all, it looks really fun! But, more than that, the reason is to learn a lot in the process, challenge some decisions, see how hard and complex some areas are, and, finally, try to connect ideas, theory, and practice.

How far we can go with this project? No idea, but I have some ideas on how to start it. And our start could be very, very, humble. The initial idea, the simplest I can figure out, is a TCP server with a simple queue behind it. By this way, we will be able to:

- Publish content to our TCP server
- Store data in a queue
- Consume content in the same order that they are produced

To allow us to have a consumer and a producer connected to the server at the same time, we can use multithread and arc/mutex around our queue. In this way, we will be able to have multiple producers, but, to deliver ordered content, we will allow just a single consumer. More consumers will compete for the messages.

## Our TCP server

To start our server, lets bind a TCP listener to localhot at some port:

```rust
    let listener = match TcpListener::bind("127.0.0.1:8080") {
        Ok(listener) => listener,
        Err(err) => panic!("Failed to bind address\r\n{}", err),
    };
```

Then, we will create a loop to process each connection to our server:

```rust
    for stream in listener.incoming() {
        match stream {
            Ok(valid_stream) => {
                // here we process each connection serialized
            }
            Err(err) => println!("Failed to process current stream\n{}", err),
        };
    }
```

At this moment, we could deal with a single connection per time. But, thinking about how Kafka consumers or producers work, I believe we should be able to keep client and server connected exchanging information. If we try to keep the connection open with the above code, since it is a single thread process, the first to reach the server will be able to exchange data while the others will need to wait until it finishes.

To fix that, we can add more threads to our server:

```rust
    for stream in listener.incoming() {
        match stream {
            Ok(valid_stream) => {
                thread::spawn(move || {
                    handle_connection(valid_stream);
                });
            }
            Err(err) => println!("Failed to process current stream\n{}", err),
        };
    }
```

Considering our basic scenario, we must define our protocol to produce, consume, and close a connection with our service. Our initial version could be as simple as checking the first character coming in the TCP stream. 

- If we receive a `p`, we will add the content to our queue
- A `c` will be understood as a request to return the next content
- To close the connection, we will wait for a `q`

 Here is a implementation for our `handle_connection`:

```rust
const C_KEY: u8 = 99;
const P_KEY: u8 = 112;
const Q_KEY: u8 = 113;

fn handle_connection(mut stream: TcpStream) {
    loop {
        let mut buffer = [0; 512];
        let size = match stream.read(&mut buffer) {
            Ok(value) => value,
            Err(err) => {
                println!("Failed to read stream\r\n{}", err);
                return;
            }
        };

        match &buffer[0] {
            &P_KEY => store_data(&stream, &buffer, size),
            &C_KEY => read_data(&stream),
            &Q_KEY => return,
            _ => continue,
        }
    }
}
```

## Adding a queue

To have a working publish/subscribe, we can add a queue. Since we spawn some threads, we can use [Arc/Mutex](https://doc.rust-lang.org/book/ch16-03-shared-state.html). The `Mutex` will allow us to mutate the queue safely and `Arc` allows us to share the reference to the `Mutex` between threads.

```rust
let queue: VecDeque<String> = VecDeque::new();
let queue = Arc::new(Mutex::new(queue));
```

Now, the last changes we must introduce to the code is pass the new variable to `handle_connection` and then consume it in our `store_data` and `read_data`. Here is the final version of them:

```rust
fn store_data(
    mut stream: &TcpStream,
    buffer: &[u8; 512],
    size: usize,
    queue: Arc<Mutex<VecDeque<String>>>,
) {
    let content = String::from_utf8_lossy(&buffer[1..size]).to_string();
    println!("[APPEND] {}", content);
    queue.lock().unwrap().push_front(content);
    stream.write_all(b"ok\r\n").unwrap();
}

fn read_data(mut stream: &TcpStream, queue: Arc<Mutex<VecDeque<String>>>) {
    if let Some(content) = queue.lock().unwrap().pop_back() {
        stream.write_all(content.as_bytes()).unwrap();
    } else {
        stream.write_all("empty\r\n".as_bytes()).unwrap();
    }
}
```

Finally, we just clone ou Arc reference to send to the thread and change the signature of handle_connection to include it:

```rust
fn handle_connection(mut stream: TcpStream, queue: Arc<Mutex<VecDeque<String>>>) { ... }
```

```rust
    for stream in listener.incoming() {
        let cloned_queue = queue.clone();
        match stream {
            Ok(valid_stream) => {
                thread::spawn(move || {
                    handle_connection(valid_stream, cloned_queue);
                });
            }
            Err(err) => println!("Failed to process current stream\n{}", err),
        };
    }
```

The full source code will be available at [Github](https://github.com/tiagodeliberali/logstreamer). This code is tagged with [v0.1.0](https://github.com/tiagodeliberali/logstreamer/releases/tag/0.1.0).

## See our service working

Now, the fun part! To test ou service, you can run it and use [nc](https://en.wikipedia.org/wiki/Netcat) or [telnet](https://en.wikipedia.org/wiki/Telnet) to connect to it and use our protocol to publish and consume messages from it.

<pre>
nc localhost 8080
pXXX -&gt; publish content XXX
c -&gt; consume new content
q -&gt; close stream
pHere we go! 
ok
pSecond message
ok
pThird one
ok
</pre>

<pre>
nc localhost 8080
pXXX -&gt; publish content XXX
c -&gt; consume new content
q -&gt; close stream
c
Here we go!
c
Second message
c
Third one
c
empty
c
empty
</pre>

## How fast it can be?

Let's build a small test. We can create two threads, on for producing and one for consuming messages through our log stream. Here is a simple 2M messages test:

```rust
use std::io::prelude::*;
use std::net::TcpStream;
use std::thread;
use std::time::Instant;

fn main() {
    thread::spawn(move || {
        let last_value = b"nice message 1999999";
        let start = Instant::now();
        let mut stream = TcpStream::connect("127.0.0.1:8080").unwrap();
        let mut i = 0;

        loop {
            stream.write_all("c\r\n".as_bytes()).unwrap();
            stream.flush().unwrap();
            let mut buffer = [0; 512];
            let size = match stream.read(&mut buffer) {
                Ok(value) => value,
                Err(err) => {
                    println!("Failed to read stream\n{}", err);
                    continue;
                }
            };

            if buffer.starts_with(last_value) {
                println!(
                    "CONSUMER MESSAGE FOUND {}",
                    String::from_utf8_lossy(&buffer[..size])
                );
                break;
            }

            if i % 50_000 == 0 {
                print!(
                    "CONSUMED MESSAGE: {} WITH VALUE VALUE: {}",
                    i,
                    String::from_utf8_lossy(&buffer[..size])
                );
            }
            i += 1;
        }

        let duration = start.elapsed();

        println!("DURATION CONSUMER: {:?}", duration);
    });

    thread::spawn(move || {
        let start = Instant::now();
        let mut stream = TcpStream::connect("127.0.0.1:8080").unwrap();

        for i in 0..2_000_000 {
            stream
                .write_all(format!("pnice message {}\r\n", i).as_bytes())
                .unwrap();
            stream.flush().unwrap();
            let mut buffer = [0; 512];
            let _ = match stream.read(&mut buffer) {
                Ok(value) => value,
                Err(err) => {
                    println!("Failed to read stream\n{}", err);
                    continue;
                }
            };

            if i % 50_000 == 0 {
                println!("PRODUCED MESSAGE: {}", i);
            }
        }
        let duration = start.elapsed();

        println!("DURATION PRODUCER: {:?}", duration);
    });

    loop {}
}
```

At my machine, we can process those `2M messages` in about `52 seconds`. It gives us an average of `26 μs per message`.

Is this number relevant? Sure this system is pretty useless compared to Kafka, but it can serve us as a baseline to understand how each decision is going to take things slow.

## What if...

What if we do not establish a TCP connection and we just handle each request independently? We could remove the loop from our `handle_connection` and move the stream connection to inside the loop. How bad this can be for our performance?

Well, after running this change a few times, the average is `292 seconds` to process the same workload. This means `146 μs per message`. That's a lot! So, we can keep the original code as-is for now.

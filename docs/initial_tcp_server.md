# Starting our kafka like publish/subscribe system

To create this system, lets start humble and iterate over new features. The initial idea, the simplest I can figure out, is a TCP server with a simple queue behind it. By this way, we will be able to:

- Publish content to our TCP server
- Store data in a queue
- Consume content in the same order that they are produced

To allow us to have a consumer and a producer connected to the server at the same time, we need to deal with multithread and arc/mutex of our queue. By this way, we will be able to have multiple producers, but, to deliver ordered content to a consumer, we will be able to have a single consumer. More consumers will compete for the messages.

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

At this moment, we could deal with connections that would execute a single batch (one or more commands grouped toghether) each time. But, thinking about a kafka consumer or producer, I believe we should be able to keep the connection open and allow client and server exchange information. If we try to keep the connection open with the above code, since it is a single thread process, we will br waiting the first client to close the connection before to start processing the second connection.

Lets add more threads to our server:

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

Considering our basic scenario, we must define our protocol to produce, consume and close a connection with our service. Our initial version could be as simple as checking the fisrt character caming in the TCP request. 

- If we receive a `p`, we will add the content to our queue
- A `c` will be understood a request to return the next content
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
                println!("Failed to read stream\n{}", err);
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

To have a working publish/subscribe, we can add a queue. Since we spawn some threads, we can use [Arc/Mutex](https://doc.rust-lang.org/book/ch16-03-shared-state.html). The `Mutex` will allow us to mutate the queue safely and `Arc` allow us to share the `Mutex` between threads.

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

The full source code will be available at https://github.com/tiagodeliberali/logstreamer.

## See our service working

Now, the fun part! To test ou service, you can run it and use [nc](https://en.wikipedia.org/wiki/Netcat) or event [telnet](https://en.wikipedia.org/wiki/Telnet) to connect to it and use our protocol to publish and consume messages from it.

<pre>nc localhost 8080
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

<pre>nc localhost 8080
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
c
</pre>
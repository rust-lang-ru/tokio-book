# Example: An Echo Server

We're going to use what has been covered so far to build an echo server. This is a
Tokio application that incorporates everything we've learned so far. The server will
simply receive messages from the connected client and send back the same message it
received to the client.

We'll be able to test this echo server using the basic Tcp client we created in the
[hello world] section.

The full code can be found [here][full-code].

# Setup

First, generate a new crate.

```shell
$ cargo new --bin echo-server
cd echo-server
```

Next, add the necessary dependencies:

```toml
[dependencies]
tokio = "0.2.0-alpha"
futures-preview = "0.3.0-alpha"
```

and the crates and types into scope in `main.rs`:

```rust, ignore
use tokio::io;
use tokio::net::TcpListener;
use tokio::prelude::*;

# fn main() {}
```

Now, we setup the necessary structure for a server:

* Bind a `TcpListener` to a local port.
* Define a task that accepts inbound connections and processes them.
* Spawn the server task.
* Start the Tokio runtime

Again, no work actually happens until the server task is spawned on the
executor.

```rust
# use futures::prelude::*;
# use tokio::net::TcpListener;
# use tokio::prelude::*;
#[tokio::main]
async fn main() {
    let addr = "127.0.0.1:6142";
    let listener = TcpListener::bind(addr).await.unwrap();

    // Here we convert the `TcpListener` to a stream of incoming connections
    // with the `incoming` method. We then define how to process each element in
    // the stream with the `for_each` combinator method
    let server = async move {
        let mut incoming = listener.incoming();
        while let Some(socket_res) = incoming.next().await {
            match socket_res {
                Ok(socket) => {
                    println!("Accepted connection from {:?}", socket.peer_addr());
                    // TODO: Process socket
                }
                Err(err) => {
                    // Handle error by printing to STDOUT.
                    println!("accept error = {:?}", err);
                }
            }
        }
    };

    println!("Server running on localhost:6142");
#    // `select` completes when the first of the two futures completes. Since
#    // future::ready() completes immediately, the server won't hang waiting for
#    // more connections. This is just so the doc test doesn't hang.
#    let server = future::select(Box::pin(server), future::ready(Ok::<_, ()>(())));

    // Start the server and block this async fn until `server` spins down.
    server.await;
}
```

Here we've created a TcpListener that can listen for incoming TCP connections. On the
listener we call `incoming` which turns the listener into a `Stream` of inbound client
connections. We then call `next` in a while loop, which will yield each inbound client connection.
For now we're not doing anything with this inbound connection - that's our next step.

Once we have our server, we `.await` on it. Up until this point our
server feature has done nothing. It's up to the Tokio runtime to drive our future to
completion.

## Handling the connections

Now that we have incoming client connections, we should handle them.

We just want to copy all data read from the socket back onto the socket itself
(e.g. "echo"). We can use the standard [`AsyncReadExt::copy`] trait method to do precisely this.

The `copy` method is called on the source where to read from and takes one argument, where to write to.
We only have one argument, though, with `socket`. Luckily there's a method, [`split`]
, which will split a readable and writeable stream into its two halves. This
operation allows us to work with each stream independently, such as pass them as two
arguments to the `copy` function.

The `copy` method then returns a future, and this future will be resolved when the
copying operation is complete, resolving to the amount of data that was copied.

Let's take a look at the connection accept code again.

```rust, no_run
# use std::env;
# use tokio::prelude::*;
# use tokio::net::TcpListener;
# #[tokio::main]
# async fn main() {
# let addr = env::args().nth(1).unwrap_or("127.0.0.1:8080".to_string());
# // Bind the server's socket.
# let mut incoming = TcpListener::bind(&addr)
#     .await
#     .expect("unable to bind TCP listener")
#     .incoming();
let server = {
  async move {
    while let Some(conn) = incoming.next().await {
      match conn {
        Err(e) => eprintln!("accept failed = {:?}", e),
        Ok(mut sock) => {
          // Spawn the future that echos the data and returns how
          // many bytes were copied as a concurrent task.
          tokio::spawn(async move {
            // Split up the reading and writing parts of the
            // socket.
            let (mut reader, mut writer) = sock.split();

            match reader.copy(&mut writer).await {
              Ok(amt) => {
                println!("wrote {} bytes", amt);
              }
              Err(err) => {
                eprintln!("IO error {:?}", err);
              }
            }
          });
        }
      }
    }
  }
};
# server.await
# }
```

As you can see we've split the `socket` stream into readable and writable parts. We
then used `AsyncReadExt::copy` to read from `reader` and write into `writer`. We `await` on the result and inspect it printing some diagnostics.

The call to [`tokio::spawn`] is the key here. We crucially want all clients to make
progress concurrently, rather than blocking one on completion of another. To achieve
this we use the `tokio::spawn` function to execute the work in the background.

If we did not do this then each invocation of the block in the loop would be
resolved at a time meaning we could never have two client connections processed
concurrently!

The full code can be found [here][full-code].

[full-code]: https://github.com/tokio-rs/tokio/blob/master/examples/echo.rs
[hello world]: ../hello-world
[`AsyncReadExt::copy`]: https://docs.rs/tokio/*/tokio/io/trait.AsyncReadExt.html#method.copy
[`split`]: https://docs.rs/tokio/*/tokio/io/trait.AsyncRead.html#method.split
[`tokio::spawn`]: https://docs.rs/tokio/*/tokio/fn.spawn.html

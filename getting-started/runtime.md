# Runtime

In the previous section we explored Futures and Streams which allow us to represent
a value (in the case of a `Future`) or a series of values (in the case of `Stream`)
that will be available "at some point in the future". We talked about `poll` on
`Future` and `Stream` which the runtime will call to figure out if the `Future` or
the `Stream` are ready to yield a value.

Lastly, we said that the runtime is needed to poll Futures and Streams driving them
to completion. We'll take a closer look at the runtime now.

## Tokio runtime

In order for a `Future` to make progress, something has to call `poll`. This is the
job of the runtime.

The runtime is responsible for repeatedly calling `poll` on a `Future` until its
value is returned. There are many different ways to do this and thus many types of
runtime configurations. For example, the [`CurrentThread`] runtime configuration
will block the current thread and loop through all spawned Futures, calling poll on
them. The thread pool configuration schedules Futures across a thread pool. This
is also the default configuration used by the Tokio [runtime][rt].

It's important to remember that all futures **must** be spawned on the runtime or no
work will be performed.

## Spawning Tasks

One of the unique aspects of Tokio is that futures can be spawned on the runtime from
within other futures or streams. When we use futures in this way, we usually refer to
them as tasks. Tasks are the application’s “unit of logic”. They are similar to [Go’s
goroutine] and [Erlang’s process], but asynchronous. In other words, tasks are
asynchronous green threads.

Given that a task runs an asynchronous bit of logic, they are represented by the
Future trait. The task’s future implementation completes with a `()` value once the
task is done processing.

Tasks are passed to the runtime, which handle scheduling the task. The runtime is
usually scheduling many tasks across a single or small set of threads. Tasks must not
perform computation-heavy logic or they will prevent other tasks from executing. So
don’t try to compute the fibonacci sequence as a task!

Tasks are implemented by either building up a future using the various combinator
functions available in the futures and tokio crates or by implementing the Future
trait directly.

We can spawn tasks using `tokio::spawn`. For example:

```rust
# use tokio::prelude::*;

# fn main() {
# let mut my_outer_stream = tokio::stream::iter(Some(1));
// Create some kind of future that we want our runtime to execute
let program = async move {
  while let Some(my_outer_value) = my_outer_stream.next().await {
    println!("Got value {:?} from the stream", my_outer_value);

    let my_inner_future = tokio::future::ready(1);

    tokio::spawn(async move {
      let my_inner_value = my_inner_future.await;
      println!("Got a value {:?} from second future", my_inner_value);
    });
  }
};

tokio::runtime::Runtime::new().unwrap().block_on(program);
# }
```

Again spawning tasks can happen within other futures or streams allowing multiple
things to happen concurrently. In the above example we're spawning the inner future
from within the outer stream. Each time we get a value from the stream we'll simply
run inner future.

In the next section, we'll take a look at a more involved example than our hello-
world example that takes everything we've learned so far into account.

[`CurrentThread`]: https://docs.rs/tokio/*/tokio/runtime/current_thread
[rt]: {{< api-url "tokio" >}}/runtime/index.html
[Go’s goroutine]: https://www.golang-book.com/books/intro/10
[Erlang’s process]: http://erlang.org/doc/reference_manual/processes.html

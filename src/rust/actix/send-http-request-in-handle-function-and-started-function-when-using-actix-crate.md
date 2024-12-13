# Send Http Request in Handle function or in started function When Using Actix crate

- [Send Http Request in Handle function](#send-http-request-in-handle-function)
  - [Return type `Result`](#return-type-result)
  - [Return type `Result`](#return-type-result)
  - [Return type `Result`](#return-type-result)
  - [Return type `Result`](#return-type-result)
- [Send Http Request in started function](#send-http-request-in-started-function)
- [Full source code](#full-source-code)
  - [Send http request in `handle` function](#send-http-request-in-handle-function)
  - [Send http request in `started` function](#send-http-request-in-started-function)
- [Appendix](#appendix)
  - [Actor trait](#actor-trait)

# Send Http Request in Handle function

When using actors to develop concurrent applications, you may need to run asynchronous functions, such as sending HTTP requests, when an actor is started or when handling specific messages.

We know there's a method called `started` when implementing `Actor` trait. The `Actor` trait is defined as follows:

```rust
pub trait Actor: Sized + Unpin + 'static {
    /// Actor execution context type
    type Context: ActorContext;

    /// Called when an actor gets polled the first time.
    fn started(&mut self, ctx: &mut Self::Context) {}

    /// Called after an actor is in `Actor::Stopping` state.
    ///
    /// There can be several reasons for stopping:
    ///
    /// - `Context::stop` gets called by the actor itself.
    /// - All addresses to the current actor get dropped and no more
    ///   evented objects are left in the context.
    ///
    /// An actor can return from the stopping state to the running
    /// state by returning `Running::Continue`.
    fn stopping(&mut self, ctx: &mut Self::Context) -> Running {
        Running::Stop
    }

    /// Called after an actor is stopped.
    ///
    /// This method can be used to perform any needed cleanup work or
    /// to spawn more actors. This is the final state, after this
    /// method got called, the actor will be dropped.
    fn stopped(&mut self, ctx: &mut Self::Context) {}

    /// Start a new asynchronous actor, returning its address.
    fn start(self) -> Addr<Self>
    where
        Self: Actor<Context = Context<Self>>,
    {
        Context::new().run(self)
    }

    /// Construct and start a new asynchronous actor, returning its
    /// address.
    ///
    /// This is constructs a new actor using the `Default` trait, and
    /// invokes its `start` method.
    fn start_default() -> Addr<Self>
    where
        Self: Actor<Context = Context<Self>> + Default,
    {
        Self::default().start()
    }

    /// Start new actor in arbiter's thread.
    fn start_in_arbiter<F>(wrk: &ArbiterHandle, f: F) -> Addr<Self>
    where
        Self: Actor<Context = Context<Self>>,
        F: FnOnce(&mut Context<Self>) -> Self + Send + 'static,
    {
        let (tx, rx) = channel::channel(DEFAULT_CAPACITY);

        // create actor
        wrk.spawn_fn(move || {
            let mut ctx = Context::with_receiver(rx);
            let act = f(&mut ctx);
            let fut = ctx.into_future(act);

            actix_rt::spawn(fut);
        });

        Addr::new(tx)
    }

    /// Start a new asynchronous actor given a `Context`.
    ///
    /// Use this method if you need the `Context` object during actor
    /// initialization.
    fn create<F>(f: F) -> Addr<Self>
    where
        Self: Actor<Context = Context<Self>>,
        F: FnOnce(&mut Context<Self>) -> Self,
    {
        let mut ctx = Context::new();
        let act = f(&mut ctx);
        ctx.run(act)
    }
}
```

The `started` function will be called when actor is started, but if we call async function in `started` function(e.g. sending http request), we'll get an error:

```rust
error[E0728]: `await` is only allowed inside `async` functions and blocks
  --> src/bin/call-async-in-non-async-function.rs:25:57
   |
22 | /     fn handle(&mut self, _: Msg, _: &mut Context<Self>) -> Self::Result {
23 | |         // async move { Ok(()) }
24 | |
25 | |         let response = reqwest::get("https://hyper.rs").await.unwrap();
   | |                                                         ^^^^^ only allowed inside `async` functions and blocks
...  |
35 | |         // })
36 | |     }
   | |_____- this is not `async`

For more information about this error, try `rustc --explain E0728`.
warning: `actix_example` (bin "call-async-in-non-async-function") generated 6 warnings
error: could not compile `actix_example` (bin "call-async-in-non-async-function") due to previous error; 6 warnings emitted
```

In Rust, `await` can only be used _within_ an async function or an async block. You can refer to [Async book](https://rust-lang.github.io/async-book/03_async_await/01_chapter.html) for more details.

The solution is easy, I'll explain it step by step.

## Return type `Result<(), ()>`

Let's start with calling async function or async block in `handle` method.

We can specify the result to be a `ResponseFuture<Result<(), ()>>` and wrapper async block with `Box::pin`.

```rust
#[derive(Message)]
#[rtype(result = "Result<(), ()>")]
struct Msg;

struct MyActor2;

impl Actor for MyActor2 {
    type Context = Context<Self>;
}

impl Handler<Msg> for MyActor2 {
    type Result = ResponseFuture<Result<(), ()>>;

    fn handle(&mut self, _: Msg, _: &mut Context<Self>) -> Self::Result {
        Box::pin(async move {
            // Some async computation
            println!("Box::pin called");
            Ok(())
        })
    }
}
```

As we use `ResponseFuture<Result<(), ()>>` type in `Handler` trait's associated type `Result`, we can return a Box Future using `Box::pin` function in `handle` method.

## Return type `Result<usize, ()>`

Now, let's change return type from `Result<(), ()>` to `Result<usize, ()>`, which will return a `usize` from async block.

```rust
#[derive(Message)]
#[rtype(result = "Result<usize, ()>")]
struct Msg3;

struct MyActor3;

impl Actor for MyActor3 {
    type Context = Context<Self>;
}

impl Handler<Msg3> for MyActor3 {
    type Result = ResponseActFuture<Self, Result<usize, ()>>;

    fn handle(&mut self, _: Msg3, _: &mut Context<Self>) -> Self::Result {
        Box::pin(
            async {
                println!("will return 42");
                // Some async computation
                42
            }
            .into_actor(self) // converts future to ActorFuture
            .map(|res, _act, _ctx| {
                println!("map");
                // Do some computation with actor's state or context
                Ok(res)
            }),
        )
    }
}
```

We need to change in 3 places:

- Using `#[rtype(result = "Result<usize, ()>")]` macro in `struct Msg3`
- Change associated type from `ResponseActFuture<Self, Result<(), ()>>;` to `ResponseActFuture<Self, Result<usize, ()>>;`
- Change async block to return a value of `usize`

## Return type `Result<u16, ()>`

If we care about the status code from http response, what should we do? Obviousely, we can declare a `Result<u16, ()>` type. Here `u16` represents the status code from http response.

```rust
#[derive(Message)]
#[rtype(result = "Result<u16, ()>")]
// return http status code
struct Msg4;

struct MyActor4;

impl Actor for MyActor4 {
    type Context = Context<Self>;
}

impl Handler<Msg4> for MyActor4 {
    // type Result = ResponseActFuture<Self, Result<usize, ()>>;
    type Result = ResponseActFuture<Self, Result<u16, ()>>;

    fn handle(&mut self, _: Msg4, _: &mut Context<Self>) -> Self::Result {
        // let res = reqwest::get("https://hyper.rs").await?;
        // println!("Status: {}", res.status());
        // let body = res.text().await?;

        Box::pin(
            async {
                println!("will return 42");
                let status_code = match reqwest::get("https://hyper.rs").await {
                    Ok(response) => {
                        println!("Got status from hyper.rs {}", response.status());
                        response.status().as_u16()
                    },
                    Err(err) => {
                        println!("get response error : {err}");
                        42 as u16
                    },
                };
                status_code
            }
            .into_actor(self) // converts future to ActorFuture
            .map(|res, _act, _ctx| {
                println!("result in map process : {res}");
                // Do some computation with actor's state or context
                Ok(res)
            }),
        )
    }
}
```

In async block, we return status code using `response.status().as_u16()`.

## Return type `Result<String, ()>`

What if we want to use the response body, what should we do? It's quite easy to change from `u16` to `String`. The code looks like this:

```rust
#[derive(Message)]
#[rtype(result = "Result<String, ()>")]
// return http reponse body
struct Msg5;

struct MyActor5;

impl Actor for MyActor5 {
    type Context = Context<Self>;
}

impl Handler<Msg5> for MyActor5 {
    // type Result = ResponseActFuture<Self, Result<usize, ()>>;
    type Result = ResponseActFuture<Self, Result<String, ()>>;

    fn handle(&mut self, _: Msg5, _: &mut Context<Self>) -> Self::Result {
        // let res = reqwest::get("https://hyper.rs").await?;
        // println!("Status: {}", res.status());
        // let body = res.text().await?;

        Box::pin(
            async {
                let status_code = match reqwest::get("https://hyper.rs").await {
                    Ok(response) => {
                        println!("Reponse Ok from hyper.rs {}", response.status());
                        match response.text().await {
                            Ok(body) => body,
                            Err(err) => {
                                format!("Convert Reposne to string error : {err}")
                            }
                        }
                    },
                    Err(err) => {
                        format!("Reposne error from hyper.rs, error : {err}")
                    },
                };
                status_code
            }
            .into_actor(self) // converts future to ActorFuture
            .map(|res, _act, _ctx| {
                println!("result in map process : {res}");
                // Do some computation with actor's state or context
                Ok(res)
            }),
        )
    }
}
```

Now, we use `response.text().await` to convert reponse to string and return the response body for later use.

# Send Http Request in started function

If we want to store some state in actor and initialize it when actor is started, we can use `context.wait` to wait an async block, turn it into an actor through `into_actor` and store the return value of async block in `then` method.

```rust
#[derive(Clone)]
struct MyActor {
    status_code: Option<u16>,
}

impl MyActor {
    fn print_status_code(&mut self, context: &mut Context<Self>) {
        println!("status code: {:?}", self.status_code);
    }
}

impl Actor for MyActor {
    type Context = Context<Self>;

    fn started(&mut self, context: &mut Context<Self>) {
        println!("In started");
        // ✅NOTE: This will run
        context.wait(
            async move {
                // send http reqwest
                let status_code = match reqwest::get("https://hyper.rs").await {
                    Ok(response) => {
                        println!(
                            "Got status from hyper.rs {}",
                            response.status()
                        );
                        response.status().as_u16()
                    }
                    Err(err) => {
                        println!("get response error : {err}");
                        42 as u16
                    }
                };
                println!("status code: {status_code}");

                status_code
            }
            .into_actor(self)
            .then(|output, s, ctx| {
                s.status_code = Some(output);
                fut::ready(())
            }),
        );

        IntervalFunc::new(Duration::from_millis(5000), Self::print_status_code)
            .finish()
            .spawn(context);

        context.run_later(Duration::from_millis(20000), |_, _| {
            System::current().stop()
        });
    }
}
```

In this example, we store status code as `Option<u16>` in `MyActor` and save it then method from `ActorFutureExt` trait:

```rust
fn started(&mut self, context: &mut Context<Self>) {
    context.wait(
        async move {
            // send http reqwest
            let status_code = match reqwest::get("https://hyper.rs").await {
                Ok(response) => {
                    response.status().as_u16()
                }
                Err(err) => {
                    42 as u16
                }
            };
            status_code
        }
        .into_actor(self)
        .then(|output, s, ctx| {
            s.status_code = Some(output);
            fut::ready(())
        }),
    );
}
```

Here is the definition of `ActorFutureExt` trait.

```rust
pub trait ActorFutureExt<A: Actor>: ActorFuture<A> {
    /// Map this future's result to a different type, returning a new future of
    /// the resulting type.
    fn map<F, U>(self, f: F) -> Map<Self, F>
    where
        F: FnOnce(Self::Output, &mut A, &mut A::Context) -> U,
        Self: Sized,
    {
        Map::new(self, f)
    }

    /// Chain on a computation for when a future finished, passing the result of
    /// the future to the provided closure `f`.
    fn then<F, Fut>(self, f: F) -> Then<Self, Fut, F>
    where
        F: FnOnce(Self::Output, &mut A, &mut A::Context) -> Fut,
        Fut: ActorFuture<A>,
        Self: Sized,
    {
        then::new(self, f)
    }

    /// Add timeout to futures chain.
    ///
    /// `Err(())` returned as a timeout error.
    fn timeout(self, timeout: Duration) -> Timeout<Self>
    where
        Self: Sized,
    {
        Timeout::new(self, timeout)
    }

    /// Wrap the future in a Box, pinning it.
    ///
    /// A shortcut for wrapping in [`Box::pin`].
    fn boxed_local(self) -> LocalBoxActorFuture<A, Self::Output>
    where
        Self: Sized + 'static,
    {
        Box::pin(self)
    }
}
```

# Full source code

## Send http request in `handle` function

```rust
use actix::prelude::*;
use anyhow::Result;
use futures::prelude::*;
use tokio::time::{sleep, Duration};

#[derive(Message)]
#[rtype(result = "Result<(), ()>")]
struct Msg;

struct MyActor2;

impl Actor for MyActor2 {
    type Context = Context<Self>;
}

impl Handler<Msg> for MyActor2 {
    type Result = ResponseFuture<Result<(), ()>>;

    fn handle(&mut self, _: Msg, _: &mut Context<Self>) -> Self::Result {
        Box::pin(async move {
            // Some async computation
            println!("Box::pin called");
            Ok(())
        })
    }
}

#[derive(Message)]
#[rtype(result = "Result<usize, ()>")]
struct Msg3;

struct MyActor3;

impl Actor for MyActor3 {
    type Context = Context<Self>;
}

impl Handler<Msg3> for MyActor3 {
    type Result = ResponseActFuture<Self, Result<usize, ()>>;

    fn handle(&mut self, _: Msg3, _: &mut Context<Self>) -> Self::Result {
        Box::pin(
            async {
                println!("will return 42");
                // Some async computation
                42
            }
            .into_actor(self) // converts future to ActorFuture
            .map(|res, _act, _ctx| {
                println!("map");
                // Do some computation with actor's state or context
                Ok(res)
            }),
        )
    }
}

#[derive(Message)]
#[rtype(result = "Result<u16, ()>")]
// return http status code
struct Msg4;

struct MyActor4;

impl Actor for MyActor4 {
    type Context = Context<Self>;
}

impl Handler<Msg4> for MyActor4 {
    // type Result = ResponseActFuture<Self, Result<usize, ()>>;
    type Result = ResponseActFuture<Self, Result<u16, ()>>;

    fn handle(&mut self, _: Msg4, _: &mut Context<Self>) -> Self::Result {
        // let res = reqwest::get("https://hyper.rs").await?;
        // println!("Status: {}", res.status());
        // let body = res.text().await?;

        Box::pin(
            async {
                println!("will return 42");
                let status_code = match reqwest::get("https://hyper.rs").await {
                    Ok(response) => {
                        println!("Got status from hyper.rs {}", response.status());
                        response.status().as_u16()
                    },
                    Err(err) => {
                        println!("get response error : {err}");
                        42 as u16
                    },
                };
                status_code
            }
            .into_actor(self) // converts future to ActorFuture
            .map(|res, _act, _ctx| {
                println!("result in map process : {res}");
                // Do some computation with actor's state or context
                Ok(res)
            }),
        )
    }
}

#[derive(Message)]
#[rtype(result = "Result<String, ()>")]
// return http reponse body
struct Msg5;

struct MyActor5;

impl Actor for MyActor5 {
    type Context = Context<Self>;
}

impl Handler<Msg5> for MyActor5 {
    // type Result = ResponseActFuture<Self, Result<usize, ()>>;
    type Result = ResponseActFuture<Self, Result<String, ()>>;

    fn handle(&mut self, _: Msg5, _: &mut Context<Self>) -> Self::Result {
        // let res = reqwest::get("https://hyper.rs").await?;
        // println!("Status: {}", res.status());
        // let body = res.text().await?;

        Box::pin(
            async {
                let status_code = match reqwest::get("https://hyper.rs").await {
                    Ok(response) => {
                        println!("Reponse Ok from hyper.rs {}", response.status());
                        match response.text().await {
                            Ok(body) => body,
                            Err(err) => {
                                format!("Convert Reposne to string error : {err}")
                            }
                        }
                    },
                    Err(err) => {
                        format!("Reposne error from hyper.rs, error : {err}")
                    },
                };
                status_code
            }
            .into_actor(self) // converts future to ActorFuture
            .map(|res, _act, _ctx| {
                println!("result in map process : {res}");
                // Do some computation with actor's state or context
                Ok(res)
            }),
        )
    }
}

fn main() -> Result<()> {
    let mut sys = actix::System::new();

    sys.block_on(async {
        // let _addr = MyActor {}.start();
        // let _addr = MyActor2 {}.start();
        // let addr = MyActor3 {}.start();
        // addr.do_send(Msg3 {})
        // OK
        // let addr = MyActor4 {}.start();
        // addr.do_send(Msg4 {})
        // OK
        let addr = MyActor5 {}.start();
        addr.do_send(Msg5 {})
    });
    sys.run()?;

    Ok(())
}
```

## Send http request in `started` function

```rust
use actix::prelude::*;
use actix::utils::IntervalFunc;
use std::sync::Arc;
use std::time::Duration;
use tokio::sync::oneshot::channel;
use tokio::sync::Mutex;

#[derive(Clone)]
struct MyActor {
    status_code: Option<u16>,
}

impl MyActor {
    fn tick(&mut self, context: &mut Context<Self>) {
        println!("tick");
    }

    fn print_status_code(&mut self, context: &mut Context<Self>) {
        println!("status code: {:?}", self.status_code);
    }
}

impl Actor for MyActor {
    type Context = Context<Self>;

    fn started(&mut self, context: &mut Context<Self>) {
        println!("In started");
        // ✅NOTE: This will run
        context.wait(
            async move {
                // send http reqwest
                let status_code = match reqwest::get("https://hyper.rs").await {
                    Ok(response) => {
                        println!(
                            "Got status from hyper.rs {}",
                            response.status()
                        );
                        response.status().as_u16()
                    }
                    Err(err) => {
                        println!("get response error : {err}");
                        42 as u16
                    }
                };
                println!("status code: {status_code}");

                status_code
            }
            .into_actor(self)
            .then(|output, s, ctx| {
                s.status_code = Some(output);
                fut::ready(())
            }),
        );

        IntervalFunc::new(Duration::from_millis(5000), Self::print_status_code)
            .finish()
            .spawn(context);

        context.run_later(Duration::from_millis(20000), |_, _| {
            System::current().stop()
        });
    }
}

fn main() {
    let mut sys = System::new();
    let addr = sys.block_on(async { MyActor { status_code: None }.start() });
    sys.run();
}
```

# Appendix

## Actor trait

````rust
/// Actors are objects which encapsulate state and behavior.
///
/// Actors run within a specific execution context
/// [`Context<A>`](struct.Context.html). The context object is available
/// only during execution. Each actor has a separate execution
/// context. The execution context also controls the lifecycle of an
/// actor.
///
/// Actors communicate exclusively by exchanging messages. The sender
/// actor can wait for a response. Actors are not referenced directly,
/// but by address [`Addr`](struct.Addr.html) To be able to handle a
/// specific message actor has to provide a
/// [`Handler<M>`](trait.Handler.html) implementation for this
/// message. All messages are statically typed. A message can be
/// handled in asynchronous fashion. An actor can spawn other actors
/// or add futures or streams to the execution context. The actor
/// trait provides several methods that allow controlling the actor
/// lifecycle.
///
/// # Actor lifecycle
///
/// ## Started
///
/// An actor starts in the `Started` state, during this state the
/// `started` method gets called.
///
/// ## Running
///
/// After an actor's `started` method got called, the actor
/// transitions to the `Running` state. An actor can stay in the
/// `running` state for an indefinite amount of time.
///
/// ## Stopping
///
/// The actor's execution state changes to `stopping` in the following
/// situations:
///
/// * `Context::stop` gets called by actor itself
/// * all addresses to the actor get dropped
/// * no evented objects are registered in its context.
///
/// An actor can return from the `stopping` state to the `running`
/// state by creating a new address or adding an evented object, like
/// a future or stream, in its `Actor::stopping` method.
///
/// If an actor changed to a `stopping` state because
/// `Context::stop()` got called, the context then immediately stops
/// processing incoming messages and calls the `Actor::stopping()`
/// method. If an actor does not return back to a `running` state,
/// all unprocessed messages get dropped.
///
/// ## Stopped
///
/// If an actor does not modify execution context while in stopping
/// state, the actor state changes to `Stopped`. This state is
/// considered final and at this point the actor gets dropped.
#[allow(unused_variables)]
pub trait Actor: Sized + Unpin + 'static {
    /// Actor execution context type
    type Context: ActorContext;

    /// Called when an actor gets polled the first time.
    fn started(&mut self, ctx: &mut Self::Context) {}

    /// Called after an actor is in `Actor::Stopping` state.
    ///
    /// There can be several reasons for stopping:
    ///
    /// - `Context::stop` gets called by the actor itself.
    /// - All addresses to the current actor get dropped and no more
    ///   evented objects are left in the context.
    ///
    /// An actor can return from the stopping state to the running
    /// state by returning `Running::Continue`.
    fn stopping(&mut self, ctx: &mut Self::Context) -> Running {
        Running::Stop
    }

    /// Called after an actor is stopped.
    ///
    /// This method can be used to perform any needed cleanup work or
    /// to spawn more actors. This is the final state, after this
    /// method got called, the actor will be dropped.
    fn stopped(&mut self, ctx: &mut Self::Context) {}

    /// Start a new asynchronous actor, returning its address.
    ///
    /// # Examples
    ///
    /// ```
    /// use actix::prelude::*;
    ///
    /// struct MyActor;
    /// impl Actor for MyActor {
    ///     type Context = Context<Self>;
    /// }
    ///
    /// #[actix::main]
    /// async fn main() {
    ///     // start actor and get its address
    ///     let addr = MyActor.start();
    ///     # System::current().stop();
    /// }
    /// ```
    fn start(self) -> Addr<Self>
    where
        Self: Actor<Context = Context<Self>>,
    {
        Context::new().run(self)
    }

    /// Construct and start a new asynchronous actor, returning its
    /// address.
    ///
    /// This is constructs a new actor using the `Default` trait, and
    /// invokes its `start` method.
    fn start_default() -> Addr<Self>
    where
        Self: Actor<Context = Context<Self>> + Default,
    {
        Self::default().start()
    }

    /// Start new actor in arbiter's thread.
    fn start_in_arbiter<F>(wrk: &ArbiterHandle, f: F) -> Addr<Self>
    where
        Self: Actor<Context = Context<Self>>,
        F: FnOnce(&mut Context<Self>) -> Self + Send + 'static,
    {
        let (tx, rx) = channel::channel(DEFAULT_CAPACITY);

        // create actor
        wrk.spawn_fn(move || {
            let mut ctx = Context::with_receiver(rx);
            let act = f(&mut ctx);
            let fut = ctx.into_future(act);

            actix_rt::spawn(fut);
        });

        Addr::new(tx)
    }

    /// Start a new asynchronous actor given a `Context`.
    ///
    /// Use this method if you need the `Context` object during actor
    /// initialization.
    ///
    /// # Examples
    ///
    /// ```
    /// use actix::prelude::*;
    ///
    /// struct MyActor {
    ///     val: usize,
    /// }
    /// impl Actor for MyActor {
    ///     type Context = Context<Self>;
    /// }
    ///
    /// #[actix::main]
    /// async fn main() {
    ///     let addr = MyActor::create(|ctx: &mut Context<MyActor>| MyActor { val: 10 });
    ///     # System::current().stop();
    /// }
    /// ```
    fn create<F>(f: F) -> Addr<Self>
    where
        Self: Actor<Context = Context<Self>>,
        F: FnOnce(&mut Context<Self>) -> Self,
    {
        let mut ctx = Context::new();
        let act = f(&mut ctx);
        ctx.run(act)
    }
}
````

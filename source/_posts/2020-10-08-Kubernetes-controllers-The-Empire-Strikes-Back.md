---
title: Kubernetes controllers - The Empire Strikes Back
description: In this blog post I want to introduce you to some important steps forward we made in our Extending Kubernetes API In-Process project. We implemented an high level watch ABI, we made the host asynchronous, we invoke the controllers only on-demand without wasting resources and we finally use the full-fledged kube-runtime to create the controllers.
date: 2020-10-08 19:42:09
tags: cloud, kubernetes, rust, webassembly
---

In this blog post I want to introduce you to some important steps forward we made in our [Extending Kubernetes API In-Process project](https://github.com/slinkydeveloper/extending-kubernetes-api-in-process-poc). If you don't know what I'm talking about, check out the previous post: [Kubernetes Controllers - A new hope](../Kubernetes-controllers-A-New-Hope).

We implemented the high-level ABI to start watchers on Kubernetes resources, as argued in [Next steps](../Kubernetes-controllers-A-New-Hope#Next-Steps) paragraph. This allows us to identify watch requests sent by controllers and to deduplicate them.

Then we worked on modifying our ABI to make it **asynchronous**, in order to invoke the controllers only **on-demand**. Now, we don't spin up a thread for each module, but we wake controllers up asynchronously when there is a new task to process.

Thanks to these changes, we're now able to use Rust's `async`/`await` inside the module. This allowed us to realign our fork of `kube-rs`, bringing back all the original interfaces, and run `kube-runtime` inside the modules.

## The `kube-watch-abi` ABI

The `kube-watch-abi` ABI was originally composed by an import and an export:

```rust
#[link(wasm_import_module = "kube-watch-abi")]
extern "C" {
    // Returns the watch identifier
    fn watch(watch_req_ptr: *const u8, watch_req_len: usize) -> u64;
}

#[no_mangle]
pub extern "C" fn on_event(watch_id: u64, ev_ptr: *const u8, ev_len: usize) {
    // handle event
}
```

The `watch_req` is a description of the watch to register:

```rust
pub(crate) struct WatchRequest {
    pub(crate) resource: Resource,
    pub(crate) watch_params: WatchParams
}
```

### Controller

When the module invokes the `watch` import, it registers a new watch and returns a watch identifier. This identifier is stored together with a reference to the `Stream<WatchEvent>` in a global map.
In similar fashion to the `request` ABI discussed in the [previous post](../Kubernetes-controllers-A-New-Hope#The-ABI), we serialize the `WatchRequest` data structure in order to pass it to the host.

Then, every time the host has a new `WatchEvent` that the controller needs to handle, it will invoke `on_event` with the serialized `WatchEvent`. Using the watch identifier, the controller will get the associated `Stream` from the global map and append the deserialized `WatchEvent` to it.

### The host

When the controller invokes `watch`, the host checks if there is a registered watch for that resource. If there is, then it just registers the invoker as interested to that watch, otherwise it starts a new watch.
Every time a watch receives a new event from Kubernetes, the host resolves all the modules interested to that particular event. For each module, it allocates some module memory to pass the event and it finally wakes the controller up by invoking `on_event`.

## Rust module with async-await

### Callbacks are bad!

The `kube-watch-abi` is the de-facto an asynchronous API: `watch` starts the asynchronous operation, `on_event` notifies on the completion of the operation. 

In the initial implementation of the `watch`, on module side, I just modified the kube client `watch` to provide a callback:

```rust
pub fn watch<F: 'static + Fn(WatchEvent<K>) + Send>(&self, lp: &ListParams, version: &str, callback: F) {
    // Invoke watch
}
```

Every time `on_event` was invoked, the module resolved the callback from the watch identifier and invoked it.
This was fine as an initial approach, but we soon hit a problem: Porting all of the existing Kubernetes client and runtime code from `async`/`await` to the more _primitive_ callbacks could have caused a lot of issues, making the fork diverge too much from the upstream code.

### How `async`/`await` works

Here's a little refresh on `async`/`await` from [Asynchronous Programming in Rust book](https://rust-lang.github.io/async-book/01_getting_started/04_async_await_primer.html):

{% cq %}
async/.await is Rust's built-in tool for writing asynchronous functions that look like synchronous code. async transforms a block of code into a state machine that implements a trait called Future. Whereas calling a blocking function in a synchronous method would block the whole thread, blocked Futures will yield control of the thread, allowing other Futures to run._
{% endcq %}

Although there are a lot of details about how Rust implements the `async`/`await` feature, these are the relevant concepts for this post:

* `Future` is the type that represents an asynchronous result, e.g. `async fn` returns a `Future`. You can wait for the result of a `Future` using `.await`.
* `Stream` is the same as `Future`, but it returns several elements before `None`, which notifies the end of the stream.
* In order to use `.await`, you **must** be in an `async` code block.
* In order to run an `async` code block, you need to use a task executor.

For example:

```rust
use futures::executor::LocalPool;

fn main() {
    // Create the executor
    let mut pool = LocalPool::new();

    // Spawn an async task
    poll.spawner().spawn(async {
        // execute `my_async_fn()` and await for the result
        my_async_fn().await
    });    

    // Block the main thread until all the tasks are done
    pool.run();
}
```

In our `async`/`await` implementation we implemented the `Future` and `Stream` traits, we reused [LocalPool](https://docs.rs/futures/0.3.6/futures/executor/struct.LocalPool.html) from the `futures` crate as a task executor, and we generalized `on_event` to notify the asynchronous operation completion.

### The controller lifecycle

We first analyzed the controller lifecycle:

1. When host invokes `run`, the controller starts a bunch of watchers and then it waits for new events
2. Every time a new event comes in, the host wakes the controller up by calling `on_event` 

This means that during the `run` phase, the controller starts a bunch of asynchronous tasks, one or more of them waiting for asynchronous events. After the `run` phase completes, **there is no need to keep the module running**. When we wake the controller up again, we need to check for any tasks that can continue and run them up to the point where we have all the tasks waiting for an external event.

To implement this lifecycle, we need to execute both on `run` and on `on_event` the executor method [`run_until_stalled()`](https://docs.rs/futures/0.3.6/futures/executor/struct.LocalPool.html#method.run_until_stalled), which will run all tasks in the executor pool and returns if no more progress can be made on any task. This allows us to implement `run` as follows:

```rust
#[no_mangle]
pub extern "C" fn run() {
    // Get the global executor
    let exec = kube::abi::get_mut_executor();
    // Start the main task
    exec.deref().borrow_mut().spawner().spawn(main()).unwrap();
    // Give a little push to the executor.
    // This runs until all tasks up to when they're all waiting for async results completion.
    exec.deref().borrow_mut().run_until_stalled();
}
```

### Our custom `Future`/`Stream`

In order to encapsulate the pending asynchronous operation, we implemented our `Future`. The implementations are straightforward and pretty much the same [as explained in the Rust async book](https://rust-lang.github.io/async-book/02_execution/03_wakeups.html):

```rust
/// Shared state between the future and the waiting thread
struct AbiFutureState {
    // The result value
    value: Option<Vec<u8>>,
    // Was the future completed?
    completed: bool,
    // Object that notifies the completion to the executor
    waker: Option<Waker>,
}

impl Future for AbiFuture {
    type Output = Option<Vec<u8>>;

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        // Look at the shared state to see if the timer has already completed.
        let mut shared_state = self.shared_state.lock().unwrap();
        if shared_state.completed {
            Poll::Ready(shared_state.value.take())
        } else {
            shared_state.waker = Some(cx.waker().clone());
            Poll::Pending
        }
    }
}
```

A similar implementation exists for the `Stream` trait. To create a `Future`, we use this method:

```rust
pub fn start_future(async_operation_id: u64) -> AbiFuture {
    // Create the future state
    let state = Arc::new(Mutex::new(
        AbiFutureState {
            value: None,
            completed: false,
            waker: None
        }
    ));
    // Insert in the global map of pending future states this future state
    get_pending_futures().deref().borrow_mut().insert(async_operation_id, state.clone());

    // Return the new future
    AbiFuture {
        shared_state: state
    }
}
```

### Generalizing `on_event` to `wakeup_future`/`wakeup_stream`

At this point, we took the concept of `on_event` and generalized to `async`/`await`, introducing `wakeup_future`/`wakeup_stream` ABI exports.
Every time the controller invokes an asynchronous ABI import (like `watch`), it gets the identifier that we use to instantiate our `Future`/`Stream` implementation.
When the host completed the asynchronous operation, it invokes `wakeup_future`/`wakeup_stream`. The controller marks the `Future`/`Stream` as completed, including the result value, and invokes `LocalPool::run_until_stalled()` to wake up tasks waiting for that future/stream to complete.

```rust
#[no_mangle]
pub extern "C" fn wakeup_future(future_id: u64, ptr: *const u8, len: usize) {
    // Retrieve the global pending future states map
    let fut_state = get_pending_futures();
    let waker = {
        let state_arc = fut_state.deref().borrow_mut().remove(&future_id).unwrap();
        let mut state = state_arc.lock().unwrap();

        // If the pointer is not null, fill the Future result value 
        if !ptr.is_null() {
            state.value = Some(unsafe {
                Vec::from_raw_parts(
                    ptr as *mut u8,
                    len as usize,
                    len as usize,
                )
            });
        }

        state.completed = true;
        state.waker.take()
    };
    // Use the waker to notify the executor this future is completed
    if let Some(waker) = waker {
        waker.wake()
    }

    // Let's try to execute stuff up to the point where there isn't anything else to execute
    get_mut_executor().deref().borrow_mut().run_until_stalled();
}
```

### The complete flow

This is a complete flow of an asynchronous ABI method:

{% mermaid sequenceDiagram %}
participant C as Controller module
participant H as Host

activate C
C ->> H: do_async()
activate H
H ->> C: Returns async operation identifier
C ->> C: run_until_stalled()
deactivate C
Note over H: Waiting for the async result
H ->> C: wakeup_future()
deactivate H
activate C
C ->> C: run_until_stalled()
deactivate C
{% endmermaid %}

You can find the complete code regarding `async`/`await` support here: [`executor.rs`](https://github.com/slinkydeveloper/extending-kubernetes-api-in-process-poc/blob/master/kube-rs/src/abi/executor.rs)

### More async ABI methods!

After we implemented `async`/`await` in our WASM modules, we refactored the `request` ABI discussed in [our previous post](../Kubernetes-controllers-A-New-Hope/#The-ABI) as an asynchronous ABI method:

```rust
#[link(wasm_import_module = "http-proxy-abi")]
extern "C" {
    fn request(ptr: *const u8, len: usize) -> u64;
}
```

Now the returned value is the asynchronous operation identifier and, to signal the completion of the request, the host invokes `wakeup_future`.

We also included a new ABI method to sleep the execution of the module:

```rust
#[link(wasm_import_module = "delay-abi")]
extern "C" {
    fn delay(millis: u64) -> u64;
}
```

This is necessary to run the kube-runtime, which performs some sleeps before synchronizing the internal cache again.

## Redesigning the host as an event-driven application

The first implementation of the new `kube-watch-abi` ABI was a little rough: a lot of blocking threads, shared memory across threads, some unsafe sprinkled here and there to make the code compiling.
Because of that, we redesigned the host to transform it in a full asynchronous application made of channels and message handlers. For every asynchronous ABI method there is a channel that delivers the _request_ to a message handler, which processes the request, computes one or more responses and sends them back to another channel. This last channel delivers messages to the `AsyncResultDispatcher`, owner of the module instances, that invokes the `wakeup_future`/`wakeup_stream` of the interested controller.

Today we have 3 different message handlers, one for each async ABI method:

* `kube_watch::Watchers` that controls the watch operation. This message handler is also able to deduplicate the watch operations
* `http::start_request_executor` to execute HTTP requests
* `delay::start_delay_executor` to execute delay requests

When the host loads all the modules, it executes the ABI method `run` for each module, then it transfers the ownership of module instances to `AsyncResultDispatcher` that will start listening for new `AsyncResult` messages on its ingress channel.

Because all the message handlers and channels are `async`/`await` based, if all the handlers are in idle, virtually no resource is wasted with threads waiting.

Since `AsyncResultDispatcher` controls all the different module instances, it avoids invoking the same controller in parallel: `LocalLoop` is a single threaded async task executor, hence a module cannot process multiple async results in parallel.

## Compiling kube-runtime to WASM

Thanks to all the async changes, we managed to realign most of the APIs of `kube-rs` to the original ones. This allowed us to port `kube-runtime` to our WASM controllers.

{% cq %}
The kube_runtime crate contains sets of higher level abstractions on top of the Api and Resource types so that you don't have to do all the watch/resourceVersion/storage book-keeping yourself.
{% endcq %}

The problem we experienced with compiling `kube-runtime` to WASM is that it depends on `tokio::time::DelayQueue`, a queue that yields components up to a specified deadline. `DelayQueue` uses the `Future` type called `Delay` to effectively implement delays. The problem with this `Delay` is that it's implemented using the internal ticker of the Tokio async task executor `Runtime`, which we don't use inside WASM modules.

In order to fix this issue, we had to fork the implementation of `tokio::time::DelayQueue` and reimplement the `Delay` type using the `delay` ABI shown previously:

```rust
pub struct Delay {
    fut: Pin<Box<dyn Future<Output=()> + Send>>,
    deadline: Instant
}

impl Delay {
    pub(crate) fn new_timeout(deadline: Instant) -> Delay {
        let now = Instant::now();
        match deadline.checked_duration_since(now) {
            Some(dur) => Delay {
                fut: Box::pin(kube::abi::register_delay(dur)),
                deadline
            },
            None => Delay {
                fut: Box::pin(futures::future::ready(())),
                deadline: now
            }
        }
    }

    // [...]
}

impl Future for Delay {
    type Output = ();

    fn poll(mut self: Pin<&mut Self>, cx: &mut task::Context<'_>) -> Poll<Self::Output> {
        self.fut.as_mut().poll(cx)
    }
}
```

With the custom implementation of `DelayQueue`, the `rust-runtime` compiled successfully to WASM, and we managed to port our controllers to use it!

```rust
async fn main() {
    let client = Client::default();

    let simple_pods: Api<SimplePod> = Api::namespaced(client.clone(), "default");
    let pods: Api<Pod> = Api::namespaced(client.clone(), "default");

    Controller::new(simple_pods, ListParams::default())
        .owns(pods, ListParams::default())
        .run(reconcile, error_policy, Context::new(Data { client }))
        .for_each(|res| async move { match res {
            Ok((obj, _)) => println!("Reconciled {:?}", obj),
            Err(e) => println!("Reconcile error: {:?}", e),
        }}).await;
}
```

## Show me the code!

You can check out the complete code of controllers today here: [simple-pod controller](https://github.com/slinkydeveloper/extending-kubernetes-api-in-process-poc/blob/master/ext-simple-pod/src/lib.rs)

If you want to look at all the different changes the project went through, look at these PRs:

1. [Event system and implementation of `watch` ABI](https://github.com/slinkydeveloper/extending-kubernetes-api-in-process-poc/pull/6)
1. [Refactor of the host as event-driven application](https://github.com/slinkydeveloper/extending-kubernetes-api-in-process-poc/pull/7)
1. [`async`/`await` + `watch` returns `Stream`](https://github.com/slinkydeveloper/extending-kubernetes-api-in-process-poc/pull/8)
1. [Non blocking HTTP proxy ABI](https://github.com/slinkydeveloper/extending-kubernetes-api-in-process-poc/pull/9)
1. [Non blocking `delay` ABI](https://github.com/slinkydeveloper/extending-kubernetes-api-in-process-poc/pull/10)
1. [Kube runtime port](https://github.com/slinkydeveloper/extending-kubernetes-api-in-process-poc/pull/11)

## Next steps

Now the controller module looks like a real Kubernetes controller: the difference is minimal to a controller targeting the usual deployment style. We also opened up a door for important optimizations, thanks to the `watch` ABI method. The host refactoring and the async ABI methods should also simplify the future interaction with Golang WASM controllers, because our ABI now resembles the asynchronous semantics of their WASM ABI. 

Our next goals are:

* Hack the Golang compiler to fit our ABI (or a similar one). For more info check out the [previous blog post](../Kubernetes-controllers-A-New-Hope#Wait-where-is-Golang)
* Perform a comparison in terms of resource utilization between this deployment style using WASM modules and the usual one of 1 container per controller.
* Figure out how to handle the different `ServiceAccount`s per controller

Stay tuned!


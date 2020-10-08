---
title: Kubernetes controllers - The Empire Strikes Back
description: In this blog post I want to introduce you some important steps forward we made in our Extending Kubernetes API In-Process project. We implemented an high level watch ABI method, we made the host asynchronous, we invoke the controllers only on-demand without wasting resources and we finally use the full-fledged kube-runtime to create the controllers.
date: 2020-10-08 19:42:09
tags: cloud, kubernetes, rust, webassembly
---

In this blog post I want to introduce you some important steps forward we made in our [Extending Kubernetes API In-Process project](https://github.com/slinkydeveloper/extending-kubernetes-api-in-process-poc). If you don't know what I'm talking about, check out the previous post: [Kubernetes Controllers - A new hope](./Kubernetes-controllers-A-New-Hope).

We implemented a high-level ABI method to start watchers on Kubernetes resources, as argued in [Next steps](./Kubernetes-controllers-A-New-Hope#Next-Steps) paragraph. This allows us to identify, _at a high level_, the watchers configured by controllers and deduplicate watch requests.

Then we worked on modifying our ABI to make it **asynchronous**, in order to invoke the controllers only **on-demand**. Now, we don't spin a thread up for each module, but we wakeup controllers asynchronously when there is a new task to process.

Thanks to these changes, we're now able to use Rust `async`/`await` inside the module. This allowed us to realign our fork of `kube-rs`, bringing back all the original interfaces, and run `kube-runtime` inside the modules.

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
In a similar fashion to `request` ABI discussed in the [previous post](./Kubernetes-controllers-A-New-Hope#The-ABI), we serialize `WatchRequest` data structure in order to pass it to the host.

Then, every time the host has a new `WatchEvent` that controller needs to handle, it will invoke `on_event` with the serialized `WatchEvent`. Using the watch identifier, the controller will get back from the global map the associated `Stream` and append to it the deserialized `WatchEvent`.

### The host

When the controller invokes `watch`, the host checks if there is a registered watch for that resource. If there is, then it just registers the invoker as interested to that watch, otherwise it starts a new watch.
Every time a watch receives a new event from Kubernetes, the host resolves all the modules interested to that particular event. For each module, it allocates some module memory to pass the event and it finally wake up again the controller with `on_event`.

## Rust module with async-await

### Callbacks are bad!

The `kube-watch-abi` is de-facto an asynchronous API: `watch` starts the asynchronous operation, `on_event` notifies the completion of the operation. 

In the initial implementation of the `watch`, on module side, I just modified the kube client `watch` to provide a callback:

```rust
pub fn watch<F: 'static + Fn(WatchEvent<K>) + Send>(&self, lp: &ListParams, version: &str, callback: F) {
    // Invoke watch
}
```

Every time `on_event` was invoked, the module resolved the callback from the watch identifier and invoked it.
This is fine as initial approach, but we soon hit a problem: porting all the existing Kubernetes client and runtime code from `async`/`await` to the more _primitive_ callbacks could have caused a lot of issues, making the fork too much divergent from the upstream code.

### How `async`/`await` works

Here's for you a little refresh on `async`/`await`:

{% note %}
From [Asynchronous Programming in Rust book](https://rust-lang.github.io/async-book/01_getting_started/04_async_await_primer.html): _async/.await is Rust's built-in tool for writing asynchronous functions that look like synchronous code. async transforms a block of code into a state machine that implements a trait called Future. Whereas calling a blocking function in a synchronous method would block the whole thread, blocked Futures will yield control of the thread, allowing other Futures to run._
{% endnote %}

Although there are a lot of details about how Rust implements the `async`/`await` feature, I'm gonna try to summarize the concepts we need for this post:

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

If you want to know more about it, I encourage you to look at the [Asynchronous Programming in Rust book](https://rust-lang.github.io/async-book/01_getting_started/01_chapter.html).

In our `async`/`await` implementation we implemented the `Future` and `Stream` traits, we reused [LocalPool](https://docs.rs/futures/0.3.6/futures/executor/struct.LocalPool.html) from `futures` crate as task executor, and we generalized `on_event` to notify the asynchronous operation completion.

### The controller lifecycle

We first analyzed the controller lifecycle:

1. When host invokes `run`, the controller starts a bunch of watchers and then it waits for new events
2. Every time a new event comes in, the host wakeup again the controller with `on_event` 

This means that during the `run` phase, the controller starts a bunch of asynchronous tasks, one or more of them waiting for asynchronous events. After the `run` phase completes, **there is no need to keep the module running**. When we wakeup again the controller, we need to check for any tasks that can continue and run them up to the point where we have all the tasks waiting for an external event.

To implement this lifecycle, we just need to execute both on `run` and `on_event` the executor method [`run_until_stalled()`](https://docs.rs/futures/0.3.6/futures/executor/struct.LocalPool.html#method.run_until_stalled), which will run all tasks in the executor pool and returns if no more progress can be made on any task. This allows us to implement the `run` as following:

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
When the host completed the asynchronous operation, it invokes `wakeup_future`/`wakeup_stream`. The controller marks the `Future`/`Stream` as completed, including the result value, and invokes again `LocalPool::run_until_stalled()` to wakeup tasks waiting for that future/stream to complete.

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

### The whole flow



You can find the complete code regarding `async`/`await` support here: [`executor.rs`](https://github.com/slinkydeveloper/extending-kubernetes-api-in-process-poc/blob/master/kube-rs/src/abi/executor.rs)

## Redesigning the host as event-driven application

The first implementation of the new `kube-watch-abi` ABI was a little rough: a lot of blocking threads, shared memory across threads, some unsafe sprinkled here and there to make the code compiling.
Because of that, we redesigned the host to transform it in a full asynchronous application made of channels and message handlers:

* aaa

## Compiling kube-runtime to WASM

## Show me the code!

If you want to check out all the different changes, look at these PRs:

* [Event system and implementation of `watch` ABI](https://github.com/slinkydeveloper/extending-kubernetes-api-in-process-poc/pull/6)
* [Refactor of the host as event-driven application](https://github.com/slinkydeveloper/extending-kubernetes-api-in-process-poc/pull/7)
* [`async`/`await` + `watch` returns `Stream`](https://github.com/slinkydeveloper/extending-kubernetes-api-in-process-poc/pull/8)
* [Non blocking HTTP proxy ABI](https://github.com/slinkydeveloper/extending-kubernetes-api-in-process-poc/pull/9)
* [Non blocking `delay` ABI](https://github.com/slinkydeveloper/extending-kubernetes-api-in-process-poc/pull/10)
* [Kube runtime port](https://github.com/slinkydeveloper/extending-kubernetes-api-in-process-poc/pull/11)

## Next steps

Now the controller module looks like a real Kubernetes controller: the difference is minimal to a controller targeting the usual deployment style. We also opened up a door for important optimizations, thanks to the `watch` ABI method. 

Our next goals are:

* Hack the Golang compiler to fit our ABI (or a similar one). For more info check out the [previous blog post](./Kubernetes-controllers-A-New-Hope#Wait-where-is-Golang)
* Perform a comparison in terms of resource utilization between this deployment style using WASM modules and the usual one of 1 container per controller.
* Figure out how to handle the different `ServiceAccount`s per controller

Stay tuned!


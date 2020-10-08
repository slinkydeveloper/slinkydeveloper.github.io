---
title: Kubernetes controllers - The Empire Strikes Back
date: 2020-10-08 19:42:09
tags: cloud, kubernetes, rust, webassembly
---

In this blog post I want to introduce you some important steps forward we made in our [Extending Kubernetes API In-Process prototype](https://github.com/slinkydeveloper/extending-kubernetes-api-in-process-poc). If you don't know what I'm talking about, check out the previous post: [Kubernetes Controllers - A new hope](./Kubernetes-controllers-A-New-Hope).

We implemented a high-level ABI method to start watchers on Kubernetes resources, as argued in [Next steps](./Kubernetes-controllers-A-New-Hope#Next-Steps) paragraph. This allows us to identify, _at a high level_, the watchers configured by controllers and deduplicate watch requests.

Then we worked on modifying our ABI to make it **asynchronous**, in order to invoke the controllers only **on-demand**. Now, we don't spin a thread up for each module, but we wakeup controllers asynchronously when there is a new task to process.

Thanks to these changes, we're now able to use Rust `async`/`await` inside the module. This allowed us to realign our fork of `kube-rs`, bringing back all the original interfaces, and run `kube-runtime` inside the modules.

## The `watch` ABI method



## Redesigning the host as event-driven application

## Rust module with async-await

In the journey of implement

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

* Figure out how to handle different `ServiceAccount`
* Hack the Golang compiler to fit our ABI (or a similar one). For more info check out the [previous blog post](./Kubernetes-controllers-A-New-Hope#Next-Steps)
* Perform a comparison in terms of resource utilization between this deployment style using WASM modules and the usual one

Stay tuned!


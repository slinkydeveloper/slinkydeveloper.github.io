---
title: Routing with Eclipse Collections
description: "Eclipse Collections are so fast!"
date: 2017-10-03 10:10:30
tags: [web development, vertx, vertx web, routing, data structures, tree, skiplist, eclipse collections, fast list]
---

I found days ago the [Eclipse Collections](https://www.eclipse.org/collections/) library (ex Goldman Sachs Collections). Yes, that **Goldman Sachs**, one of the biggest investing banking group of the world, that for hobby creates super fast collection library for Java.

This article follows the previous, when I explored how to improve the routing of [Vert.x Web](http://vertx.io/docs/vertx-web/java/), so please check it out before reading this one: [Tree vs SkipList routing](https://slinkydeveloper.github.io/articles/Routing-Tree-vs-SkipList/)

## What's new
Three days ago, looking at my twitter wall, I've found a tweet about Eclipse Collections. I've found really interesting the performances of EC, so I've decided to put it into my benchmark and test it on our use case. I obviously choose the tree as data structure to rewrite the routing process of Vert.x Web, so I have write two variants of my original `TreeRouter`:

* `ECTreeRouter`: A tree that internally uses List implementations of Eclipse Collections.
* `ImmutableECTreeRouter`: A tree that internally uses **immutable** List implementations of Eclipse Collections. In this case user **can't change** the routing tree after routing has started.

The second option was a pure experiment: user creates the router and its internal tree doesn't change during the application execution. In this case you have a _simpler implementation_ and an immutable list (in some cases faster than a mutable one). I've also refactored only the `SocialNetworkBenchmark`, because we have similar results on `ECommerceBenchmark`.

## Results
And, as in the previous articles, here comes the graphs:

<figure>
  <a href="{{ site.url }}/images/tree-vs-router-2/basic_social.png" class="image-popup"><img src="{{ site.url }}/images/tree-vs-router-2/basic_social.png" alt="image"></a>
  <figcaption>Benchmark results for SocialNetworkBenchmark based on requested URLs</figcaption>
</figure>
<figure>
  <a href="{{ site.url }}/images/tree-vs-router-2/with_load_social.png" class="image-popup"><img src="{{ site.url }}/images/tree-vs-router-2/with_load_social.png" alt="image"></a>
  <figcaption>Benchmark results for SocialNetworkBenchmark based on requested URLs with 10 random requests</figcaption>
</figure>
<figure>
  <a href="{{ site.url }}/images/tree-vs-router-2/social_complete.png" class="image-popup"><img src="{{ site.url }}/images/tree-vs-router-2/social_complete.png" alt="image"></a>
  <figcaption>Final benchmark results for SocialNetworkBenchmark ("with load" values properly scaled)</figcaption>
</figure>

And in the end the _"final test"_ graph (now it does only random requests, not sequentially):

<figure>
  <a href="{{ site.url }}/images/tree-vs-router-2/social_average.png" class="image-popup"><img src="{{ site.url }}/images/tree-vs-router-2/social_average.png" alt="image"></a>
  <figcaption>Final benchmark results for SocialNetworkBenchmark</figcaption>
</figure>

I have some considerations about these results:

* Eclipse Collections are fast to iterate, faster than JDK's collections, so probably they are good for our use case. In particular, pay attention to particular events generated from benchmark data:
  * `ECTreeRouter` is faster than skip list in `/feed` request in "without load" tests! In particular it creates interesting deltas from `TreeRouter` when we request constant paths...
  * But going deeper doesn't help the `ECTreeRouter`, in particular in "without load" benchmarks. We have too few datas to assert that at deeper levels `ECTreeRouter` drops its performances or aligns it with `TreeRouter`
  * `ECTreeRouter`, without any doubt, in the random requests test is faster than `TreeRouter`
* Eclipse Collections contains a lot of optimized iteration patterns optimized, maybe useful for us
* It seems that EC thread-safe lists are faster than immutable lists, so the `ImmutableECTreeRouter` is a failed experiment :disappointed_relieved:

Unlike the previous post I'm little hesitant to give a verdict, but we have promising results with Eclipse Collections, so I want to start with it. In case we experience "not so good" performances after the implementation, migrate back to JDK's collections doesn't appear a complicated task 

Stay tuned for other updates!


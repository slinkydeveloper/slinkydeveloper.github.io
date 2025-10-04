---
title: Assertions with Vert.x Futures and JUnit5
description: "Write beautiful async assertions with Vert.x Futures and Vert.x JUnit 5"
date: 2018-10-29
tags: [development, testing, junit5, vertx]
---

During development of [Vert.x event manager library](https://github.com/slinkydeveloper/vertx-event-manager) (a blog post about it is coming soon) I wanted to play with new [`vertx-junit5`](https://vertx.io/docs/vertx-junit5/java/) library. I like the new async assertion APIs of `vertx-junit5`, but I feel very unconfortable using [`VertxTestContext.succeding(Handler)`](https://vertx.io/docs/apidocs/io/vertx/junit5/VertxTestContext.html#succeeding-io.vertx.core.Handler-) when I need to run sequentially different async tasks. With this method, your code rapidly grows in a big callback hell! Plus the interfaces I wanted to test are all in `Future`s style more than callback style.

In this post I'm going to explain you two methods I've added with a [PR](https://github.com/vert-x3/vertx-junit5/pull/53) that simplify tests with `Future`s

## `assertComplete()` and `assertFailure()`

The PR adds methods:

* `Future<T> assertComplete(Future<T> fut)`
* `Future<T> assertFailure(Future<T> fut)`

These methods take a future as parameter and register to it the handler that asserts the completion/failure of it. They return a **copy** of the future you passed as parameter

For example this callback style assertion:

```java
methodThatReturnsAFuture().setHandler(testContext.succeding(result -> {
    // Some assertions
    // Call testContext.complete() or flag a checkpoint
}));
```

Turns into:

```java
testContext.assertComplete(methodThatReturnsAFuture()).setHandler(asyncResult-> {
    // Some assertions. Note that result is in asyncResult.result()
    // Call testContext.complete() or flag a checkpoint
});
```

Nothing revolutionary, right? To appreciate it let's look at a more real use case

## Testing a Future chain

Let's say that we want to test an update method of a class that manage some entities in a database. A common flow for this kind of tests is:

1. Use the raw db client to add some data
2. Use the class instance you want to test to update data on db
3. Retrieve data from db to test if update is successfull

Assuming that both raw db client and entity manager has _futurized_ APIs, without these methods, this test translates in 3 nested callbacks. Now you can simplify it like this:

```java
testContext.assertComplete(
  rawClient.create(someData)
    .compose(addedData -> myEntityManager.update(addedData.getId(), stuffToUpdate))
    .compose(updatedData -> rawClient.get(updatedData.getId()))
).setHandler(resultAr -> {
    // assertComplete guarantees that resultAr is completed
    // Do the assertions you want
    testContext.complete();
});
```

With just one `assertComplete()` we assert that all chain of async operations completes without errors. Then I set an handler that does the final assertions before completing the test

Now, let's assume that you want to do the same test as before but testing a failure of your method. To do it you need to check every single step of future chain:

```java
testContext.assertComplete(rawClient.create(someData))
    .compose(addedData -> testContext.assertFailure(myEntityManager.update(addedData.getId(), stuffToUpdate)))
    .recover(failedAr -> {
        // Do some assertions on failedAr.cause()
        return testContext.assertComplete(rawClient.get(failedAr.cause().getEntityId()));
    })
    .setHandler(resultAr -> {
        // Do the assertions you want
        testContext.complete();
    });
```

## Tricks and tips

The bad thing of future chains is passing values through the chain. Let's say that in previous example the exception throwed by `update()` method doesn't return an exception that contains a super handy method like `getEntityId()`. But to get the data from db you need the `id` of your data instance, so how you can solve it?

You have two ways that really depend on your code style:

* If you are a bit more _functional_, use [`CompositeFuture.join()`](https://vertx.io/docs/apidocs/io/vertx/core/CompositeFuture.html#join-io.vertx.core.Future-io.vertx.core.Future-) to transform a tuple of Futures (one of them already completed with the value you want to pass through the chain) to a single Future that encapsulates both the previous async operation result and the new result. This method works only when you are in a chain of completed handlers because when a future inside `CompositeFuture.join()` fails, the "join future" is not an instance of `CompositeFuture` and doesn't return any information about other joined futures. I prefer to avoid this method, but keep it in mind because you can find it useful sometimes.

* If you don't care about functional stuff, just use old but gold `AtomicReference`s:

```java
AtomicReference<String> entityId = new AtomicReference<>();
testContext.assertComplete(rawClient.create(someData))
    .compose(addedData -> {
        entityId.set(addedData.getId());
        return testContext.assertFailure(myEntityManager.update(addedData.getId(), stuffToUpdate))
     })
    .recover(failedAr -> {
        // Do some assertions on failedAr.cause()
        return testContext.assertComplete(rawClient.get(entityId.get()));
    })
    .setHandler(resultAr -> {
        // Do the assertions you want
        testContext.complete();
    });
```

If you have any good tips don't hesitate to contact me! Happy testing!

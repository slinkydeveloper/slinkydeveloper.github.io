---
title: 'JUnit 5, Parallel tests, Extensions and ThreadLocal'
description: "In this blog post I'll talk about my story about enabling the Apache Flink project to use the new JUnit 5 features, including parallel execution, parametrized tests and extensions, and how they're going to help us improve our test codebase."
date: 2022-02-19 12:25:15
tags: [java, testing, junit5, flink]
---

In the [Apache Flink](https://flink.apache.org/) community we're in the process of [porting our huge test codebase to JUnit 5](https://lists.apache.org/thread/jsjvc2cqb91pyh47d4p6olk3c1vxqm3w). In order to leverage as much as we can the new JUnit 5 features, in the past days I've spent some time playing around with it.

In this blog post I'll talk about my story about enabling the project to use the new JUnit 5 features, including parallel execution, parametrized tests and extensions, and how they're going to help us improve our test codebase.

## Our test codebase

In Flink we've all kind of tests you can think of:

* Simple unit tests with a certain degree of mocking
* Integration tests
* End-to-end tests
* Various other tests for utilities, such as tests for our bash scripts

In most of the codebase, we refer to integration tests as tests that define and run a streaming job, but use a mocked sink and source. While end-to-end tests are like integration tests, but they use real external systems, such as Kafka, deployed with [TestContainers](https://www.testcontainers.org/test_framework_integration/junit_5/). 

In particular in Flink SQL, _unsurprisingly_, we have a lot of integration tests, because each single feature requires to be "understood" by all the different moving parts of our stack. For example, take the built-in function `COALESCE`: it has a runtime implementation, a Table API expression [^1], a custom logic for arguments type inference and return type inference, an optimizer rule that removes the call when possible. Each of these single pieces need to work in harmony, and integration tests usually gives us the guarantee that everything fits together.

Another aspect of JUnit 5 tests is that we have a lot of test bases and of parametrized test bases, à la Junit 4. This is due to the organic growth of the project and the effort to try to standardize certain test aspects, like starting and stopping a Flink `MiniCluster`, the embedded Flink cluster to run test jobs.

## Goals

In porting to JUnit 5, we want to:

* Have less test bases, but more extensions, hence composition over inheritance. This simplifies contributing new tests, as adding a new "capability" to the test suite won't require new ad-hoc test bases.
* Improve error reporting and test cases separation, in order to make the contributor experience nicer both when running tests from the IDE and with Maven
* Speedup the test suite as much as possible

In particular the last point is a hot topic, as today a CI run usually takes between 1 and a half and 2 hours, having a significant impact on the development loop of the project.

## Parallel tests

Parallel tests are, in my opinion, the killer feature of JUnit 5 and the real incentive to port JUnit 4 tests to JUnit 5.

With JUnit 4 you can parallelize test execution using the build tool, for example using [`maven-surefire-plugin` fork JVM feature](https://maven.apache.org/surefire/maven-surefire-plugin/examples/fork-options-and-parallel-execution.html). It runs tests in parallel by spawning several JVM processes, where each of them gets assigned a split of the overall list of tests to run. JUnit 5 on the other hand runs all the tests within the same JVM: the test runner manages a thread pool and takes care of assigning test cases to threads.

I think the JUnit 5 approach fits best in our use case, as spawning several JVMs is very resource intensive on constrained machines such as CI runners, so it's a constant source of issues. This statement is also valid for contributor's machines, as these days just running the browser with 20+ tabs open, Spotify, Slack and the IDE can easily eat up to 16Gb of RAM. Plus they work in any IDE without additional configuration, they can help you find out thread safety bugs and the granularity of the execution is easy and flexible to configure.

To start using JUnit 5 parallel tests, we just had to create a file called `junit-platform.properties` in our test resources and add the following:

```
junit.jupiter.execution.parallel.enabled = true
junit.jupiter.execution.parallel.config.strategy = dynamic
junit.jupiter.execution.parallel.mode.default = same_thread
```

This configuration enables to opt-in specific tests/classes to run in parallel. To flag a test class to run in parallel, we need to annotate it with `@Execution(CONCURRENT)`. Thanks to this configuration we can gradually enable parallel execution only for tests we know are safe.

JUnit 5 offers the ability to configure the granularity of the parallel test execution, e.g. run all test cases from all classes in parallel, run all test cases from a class sequentially but run the test classes in parallel, etc. Check out all the available options in the [JUnit 5 documentation](https://junit.org/junit5/docs/current/user-guide/#writing-tests-parallel-execution).

So I've decided I wanted to try this new feature with some complex parametrized test base. I ended up choosing [`CastRulesTest`](https://github.com/apache/flink/blob/271c59332c794d86d221b4fe10c4435cc37d652f/flink-table/flink-table-planner/src/test/java/org/apache/flink/table/planner/functions/casting/CastRulesTest.java), a parametrized unit test base checking the runtime implementation of the `CAST` logic. Because there is no shared state whatsoever, this test suite is embarrassingly parallelizable. Just adding the annotation `@Execution(CONCURRENT)` gave me a 3x times faster execution time for the whole suite.

## Integration tests in parallel

That 3x totally got my attention, so I wanted to try to apply the same annotation to integration tests as well. As my next target, I've chosen [`BuiltInFunctionTestBase`](https://github.com/apache/flink/blob/3cf0393d8946df3aa7a8836f5b2291791c13f215/flink-table/flink-table-planner/src/test/java/org/apache/flink/table/planner/functions/BuiltInFunctionTestBase.java). As the name implies, this is a parametrized integration test base we use to test the correct behaviour of our built-in functions. We have around 10 classes where we use this test base, for a total of 1200+ integration test cases.

### Porting the test suite to JUnit 5

The first thing I had to do was to port to JUnit 5 the base class and its inheritors. My initial thought was to use [`@TestFactory` feature](https://junit.org/junit5/docs/current/user-guide/#writing-tests-dynamic-tests), a new JUnit 5 feature to spawn dynamic tests, allowing you to group test cases. Think to it as a more powerful `@ParametrizedTest`.

This would have allowed me to have a nice nested view of the tests in the reports. For example, look at [`MathFunctionsITCase`](https://github.com/apache/flink/blob/3cf0393d8946df3aa7a8836f5b2291791c13f215/flink-table/flink-table-planner/src/test/java/org/apache/flink/table/planner/functions/MathFunctionsITCase.java): by mixing `DynamicTest` and `DynamicContainer` I could have achieved a report like:

* `MathFunctionsITCase`:
  * `PLUS`:
    * `f0 + 6 = 20`: Success
    * `f0 + 6 = 10`: Failure
  * `MINUS`: [...]

Because the test base itself already had some models to define test cases, including input/output data, query expressions and configuration (see [`TestSpec`](https://github.com/apache/flink/blob/3cf0393d8946df3aa7a8836f5b2291791c13f215/flink-table/flink-table-planner/src/test/java/org/apache/flink/table/planner/functions/BuiltInFunctionTestBase.java#L183)), what I had to do was simply to convert these models to `DynamicTest`/`DynamicContainer`. A little refactor of the [`testFunction` method](https://github.com/apache/flink/blob/3cf0393d8946df3aa7a8836f5b2291791c13f215/flink-table/flink-table-planner/src/test/java/org/apache/flink/table/planner/functions/BuiltInFunctionTestBase.java#L87) to wrap the logic into `DynamicTest` did it. Now every concrete test class just had to implement the abstract method `getTestSpecs` to return the test cases defined with my `TestSpec` class, so the final implementation of the `@TestFactory` just looked like:

```java
@TestFactory
Stream<DynamicContainer> tests() {
    return getTestSpecs().map(TestSpec::toDynamicContainer);
}
```

Every implementation class provided a `DynamicContainer`, containing a set of tests for a specific built-in function, like `PLUS`, and each container had a set of `DynamicTest` with the specific test cases for that built-in function, like `f0 + 6 = 20`.

Last but not least, to let the query run, I needed the extension to set up `MiniCluster` once per class. This is already available in our `flink-test-utils`, so with some copy-paste I enabled it:

```java
public static final MiniClusterWithClientExtension MINI_CLUSTER_RESOURCE =
        new MiniClusterWithClientExtension(
                new MiniClusterResourceConfiguration.Builder()
                        .setNumberTaskManagers(1)
                        .build());

@RegisterExtension
public static final AllCallbackWrapper<MiniClusterWithClientExtension> ALL_WRAPPER =
        new AllCallbackWrapper<>(MINI_CLUSTER_RESOURCE);
```

Tried a first run without parallel tests, and everything ran fine. Tried to add `@Execution(CONCURRENT)`, and Intellij IDEA welcomed my idea with a long report full of red crosses.

### Fixing the `MiniCluster` extension first

Looking at the logs, it became evident how the problem was `MiniClusterWithClientExtension`, given several tests were trying to push jobs to a `MiniCluster` already shut down.

The `MiniClusterWithClientExtension` was developed by wrapping the JUnit 4 `MiniClusterWithClientResource` rule in a custom interface defining `before` and `after`. Then, to register it, you could either use `AllCallbackWrapper` or `EachCallbackWrapper` to define whether to have one `MiniCluster` per test class, or per single test.

In JUnit 4 rules have a `before` and `after` extension point, and then when you register them, depending on whether you use `@Rule` or `@ClassRule`, the rule is executed for each test or once per class. In JUnit 5 the user cannot pick whether the extension is used globally or per method: it's the extension itself that defines where it can hook in the lifecycle.

An example of this shift of concept between JUnit 4 and 5 is provided by the JUnit 5 `TestContainers` integration, which depending on whether the container field is static or not, decides to share it between test methods or not[^3].

Our `AllCallbackWrapper` or `EachCallbackWrapper` were circumventing the new JUnit 5 Extension paradigm, bringing back the same semantics of `@Rule` or `@ClassRule`. And this worked fine, until I tried to use parallel execution.

The [`MiniClusterWithClientExtension#before`](https://github.com/apache/flink/blob/78b231f60aed59061f0f609e0cfd659d78e6fdd5/flink-test-utils-parent/flink-test-utils/src/main/java/org/apache/flink/test/util/MiniClusterWithClientExtension.java#L68) method was starting `MiniCluster`, creating some `ClusterClient` and then setting up a thread local for the environment configuration lookup.

The interaction between our `ThreadLocal` configuration when using `AllCallbackWrapper` didn't sound right, so I tried to do a little experiment. Take this simple extension:

```java
public class TestExtension
        implements BeforeEachCallback, BeforeAllCallback, AfterEachCallback, AfterAllCallback {
    @Override
    public void beforeAll(ExtensionContext context) throws Exception {
        System.out.println("before all: " + Thread.currentThread().getName());
    }

    @Override
    public void beforeEach(ExtensionContext context) throws Exception {
        System.out.println(
                "before " + context.getDisplayName() + ": " + Thread.currentThread().getName());
    }

    @Override
    public void afterEach(ExtensionContext context) throws Exception {
        System.out.println(
                "after " + context.getDisplayName() + ": " + Thread.currentThread().getName());
    }

    @Override
    public void afterAll(ExtensionContext context) throws Exception {
        System.out.println("after all: " + Thread.currentThread().getName());
    }
}
```

This is simply going to print the thread where the various hooks are executed, and it also prints the name of single test in case in `beforeEach`/`afterEach`. Then I tried to run this test:

```java
@ExtendWith(TestExtension.class)
@Execution(ExecutionMode.CONCURRENT)
public class MyTest {

    @ParameterizedTest(name = "{0}")
    @ValueSource(ints = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10})
    void name(int i) {
        System.out.println("test: " + i + ", thread: " + Thread.currentThread().getName());
    }
}
```

And here is the result:

```
before all: ForkJoinPool-1-worker-1
before 2: ForkJoinPool-1-worker-3
before 3: ForkJoinPool-1-worker-4
before 5: ForkJoinPool-1-worker-6
before 9: ForkJoinPool-1-worker-1
before 4: ForkJoinPool-1-worker-5
before 7: ForkJoinPool-1-worker-0
before 6: ForkJoinPool-1-worker-7
before 1: ForkJoinPool-1-worker-2
test: 7, thread: ForkJoinPool-1-worker-0
test: 9, thread: ForkJoinPool-1-worker-1
test: 4, thread: ForkJoinPool-1-worker-5
test: 1, thread: ForkJoinPool-1-worker-2
test: 6, thread: ForkJoinPool-1-worker-7
test: 2, thread: ForkJoinPool-1-worker-3
test: 3, thread: ForkJoinPool-1-worker-4
test: 5, thread: ForkJoinPool-1-worker-6
after 4: ForkJoinPool-1-worker-5
after 7: ForkJoinPool-1-worker-0
after 6: ForkJoinPool-1-worker-7
after 1: ForkJoinPool-1-worker-2
after 5: ForkJoinPool-1-worker-6
after 9: ForkJoinPool-1-worker-1
after 3: ForkJoinPool-1-worker-4
after 2: ForkJoinPool-1-worker-3
before 8: ForkJoinPool-1-worker-3
test: 8, thread: ForkJoinPool-1-worker-3
before 10: ForkJoinPool-1-worker-6
after 8: ForkJoinPool-1-worker-3
test: 10, thread: ForkJoinPool-1-worker-6
after 10: ForkJoinPool-1-worker-6
after all: ForkJoinPool-1-worker-1
```

As I was expecting, `beforeEach` and `afterEach` runs in the same thread of the test, as effectively they just "wrap" the test method, as described [here](https://junit.org/junit5/docs/current/api/org.junit.jupiter.api/org/junit/jupiter/api/extension/BeforeEachCallback.html).

In other words:

* There is no guarantee about which thread is used to execute `beforeAll` and `afterAll`, but
* `beforeEach` and `afterEach` are guaranteed to be executed within the same thread of the test

So here was the solution: we just had to set/unset that `ThreadLocal` **everytime** in `beforeEach`/`afterEach`, no matter whether the `MiniCluster` instance was meant to be per test class or per test method.

I was ready to declare victory, so I did some refactoring of `MiniClusterWithClientExtension` to always create one `MiniCluster` per test class, I removed the wrapping around `AllCallbackWrapper` and then I tried to run again my tests: still everything was red.

### Back on `@ParametrizedTest`

After some investigation, I found out that `DynamicTest` lifecycle doesn't work per `DynamicTest` instance, while parallelization does. For example, for this test class:

```java
@ExtendWith(TestExtension.class)
@Execution(ExecutionMode.CONCURRENT)
public class MyTest {

    @TestFactory
    public Stream<DynamicTest> tests() {
        return IntStream.rangeClosed(1, 10)
                .mapToObj(i ->
                  DynamicTest.dynamicTest(
                    String.valueOf(i),
                    () -> System.out.println("test: " + i + ", thread: " + Thread.currentThread().getName())
                  ));
    }
}
```

You get this output:

```
before all: ForkJoinPool-1-worker-1
before tests(): ForkJoinPool-1-worker-1
after tests(): ForkJoinPool-1-worker-1
test: 2, thread: ForkJoinPool-1-worker-3
test: 6, thread: ForkJoinPool-1-worker-7
test: 3, thread: ForkJoinPool-1-worker-4
test: 9, thread: ForkJoinPool-1-worker-1
test: 7, thread: ForkJoinPool-1-worker-0
test: 1, thread: ForkJoinPool-1-worker-2
test: 4, thread: ForkJoinPool-1-worker-5
test: 5, thread: ForkJoinPool-1-worker-6
test: 8, thread: ForkJoinPool-1-worker-3
test: 10, thread: ForkJoinPool-1-worker-2
after all: ForkJoinPool-1-worker-1
```

As you see, each `DynamicTest` is executed in parallel, as you would expect, but the `beforeEach` and `afterEach` is executed only once per `@TestFactory`. Because of this behaviour, the tests were not picking the correct `ThreadLocal`, hence failing the test because they could not find the `MiniCluster`.

It seems like there is no solution to this problem, and there is an issue open in the [JUnit 5 issue tracker](https://github.com/junit-team/junit5/issues/378) about whether this should be supported or not.

So I had to fall back to the good old `@ParametrizedTest`: I just removed the nested `DynamicContainer`, and I created my own `DynamicTest` like data structure to wrap a test `Executable` and its display name:

```java
/** Single test case. */
static class TestCase {

    private final String name;
    private final Executable executable;

    TestCase(String name, Executable executable) {
        this.name = name;
        this.executable = executable;
    }

    @Override
    public String toString() {
        return name;
    }
}
```

And then I kept the same code as before in the test base, but I had to add a method to flatten the previously used `DynamicContainer`s:

```java
abstract Stream<TestSpec> getTestSpecs();

final Stream<TestCase> getTestCases() {
    return this.getTestSpecs()
            .flatMap(testSpec -> testSpec.getTestCases(this.getConfiguration()));
}

@ParameterizedTest
@MethodSource("getTestCases")
final void test(TestCase testCase) throws Throwable {
    testCase.executable.execute();
}
```

The report doesn't look as nice as with `@TestFactory`, but it finally works fine with parallel execution.

## Results and conclusion

Before the parallelization, this test suite ran in around 50 seconds from Intellij IDEA on my i7 11th Gen packed with 16Gb of RAM. After the parallelization the suite runs in around 20 seconds[^2]. Not bad.

I think JUnit 5 is a very powerful tool, but you need to be careful when developing extensions that are supposed to work with parallel tests. These are my gotchas from this experience:

* Make sure thread locals are set in `beforeEach`, so they're set in the right thread, and **cleaned up** in `afterEach`, to avoid next tests reusing that thread to pick up the old `ThreadLocal` values
* Be aware that the method `beforeEach` could be invoked in parallel on the same instance of the extension, so you need to make sure the access to extension fields is thread safe, or...
* Use the `ExtensionContext.Store`, as it's thread safe, it's hierarchically organized per hook context, and it's particularly useful to auto cleanup objects built in the `before[..]` methods
* Rather than providing accessors in the extension class, use parameter injection implementing `ParameterResolver` together with the `ExtensionContext.Store`. This simplifies the implementation when storing your extension state in `ExtensionContext.Store`.

And last but not least, don't use `@TestFactory` in conjunction with parallel execution if your test requires `ThreadLocal` or other thread dependant thing correctly configured.

[^1]: The DSL provided by Flink SQL to define relational queries without SQL
[^2]: Please take these numbers with a grain of salt, my measurements were very empirical, and not proper benchmarks
[^3]: [TestContainers - JUnit 5 integration](https://www.testcontainers.org/test_framework_integration/junit_5/)
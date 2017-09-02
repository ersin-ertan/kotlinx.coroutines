<!--- INCLUDE .*/example-([a-z]+)-([0-9a-z]+)\.kt 
/*
 * Copyright 2016-2017 JetBrains s.r.o.
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 * http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */

// This file was automatically generated from coroutines-guide.md by Knit tool. Do not edit.
package guide.$$1.example$$2

import kotlinx.coroutines.experimental.*
-->
<!--- KNIT     kotlinx-coroutines-core/src/test/kotlin/guide/.*\.kt -->
<!--- TEST_OUT kotlinx-coroutines-core/src/test/kotlin/guide/test/GuideTest.kt
// This file was automatically generated from coroutines-guide.md by Knit tool. Do not edit.
package guide.test

import org.junit.Test

class GuideTest {
--> 

# kotlinx.coroutines by example tl;dr

* Kotlin's standard library for low-level coroutine APIs mostly for libraries.
* Common language keywords like `async` and `await` are not included.
* `kotlinx.coroutines` is high-level coroutine-enabled primitives(`async` and `await`). 
* See [`kotlinx-coroutines-core`](README.md#using-in-your-projects)

## Table of contents

<!--- TOC -->

* [Coroutine basics](#coroutine-basics)
  * [Your first coroutine](#your-first-coroutine)
  * [Bridging blocking and non-blocking worlds](#bridging-blocking-and-non-blocking-worlds)
  * [Waiting for a job](#waiting-for-a-job)
  * [Extract function refactoring](#extract-function-refactoring)
  * [Coroutines ARE light-weight](#coroutines-are-light-weight)
  * [Coroutines are like daemon threads](#coroutines-are-like-daemon-threads)
* [Cancellation and timeouts](#cancellation-and-timeouts)
  * [Cancelling coroutine execution](#cancelling-coroutine-execution)
  * [Cancellation is cooperative](#cancellation-is-cooperative)
  * [Making computation code cancellable](#making-computation-code-cancellable)
  * [Closing resources with finally](#closing-resources-with-finally)
  * [Run non-cancellable block](#run-non-cancellable-block)
  * [Timeout](#timeout)
* [Composing suspending functions](#composing-suspending-functions)
  * [Sequential by default](#sequential-by-default)
  * [Concurrent using async](#concurrent-using-async)
  * [Lazily started async](#lazily-started-async)
  * [Async-style functions](#async-style-functions)
* [Coroutine context and dispatchers](#coroutine-context-and-dispatchers)
  * [Dispatchers and threads](#dispatchers-and-threads)
  * [Unconfined vs confined dispatcher](#unconfined-vs-confined-dispatcher)
  * [Debugging coroutines and threads](#debugging-coroutines-and-threads)
  * [Jumping between threads](#jumping-between-threads)
  * [Job in the context](#job-in-the-context)
  * [Children of a coroutine](#children-of-a-coroutine)
  * [Combining contexts](#combining-contexts)
  * [Naming coroutines for debugging](#naming-coroutines-for-debugging)
  * [Cancellation via explicit job](#cancellation-via-explicit-job)
* [Channels](#channels)
  * [Channel basics](#channel-basics)
  * [Closing and iteration over channels](#closing-and-iteration-over-channels)
  * [Building channel producers](#building-channel-producers)
  * [Pipelines](#pipelines)
  * [Prime numbers with pipeline](#prime-numbers-with-pipeline)
  * [Fan-out](#fan-out)
  * [Fan-in](#fan-in)
  * [Buffered channels](#buffered-channels)
  * [Channels are fair](#channels-are-fair)
* [Shared mutable state and concurrency](#shared-mutable-state-and-concurrency)
  * [The problem](#the-problem)
  * [Volatiles are of no help](#volatiles-are-of-no-help)
  * [Thread-safe data structures](#thread-safe-data-structures)
  * [Thread confinement fine-grained](#thread-confinement-fine-grained)
  * [Thread confinement coarse-grained](#thread-confinement-coarse-grained)
  * [Mutual exclusion](#mutual-exclusion)
  * [Actors](#actors)
* [Select expression](#select-expression)
  * [Selecting from channels](#selecting-from-channels)
  * [Selecting on close](#selecting-on-close)
  * [Selecting to send](#selecting-to-send)
  * [Selecting deferred values](#selecting-deferred-values)
  * [Switch over a channel of deferred values](#switch-over-a-channel-of-deferred-values)
* [Further reading](#further-reading)

<!--- END_TOC -->

## Coroutine basics

### Your first coroutine

```kotlin
fun main(args: Array<String>) {
    launch(CommonPool) { // create new coroutine in common thread pool
        delay(1000L) // non-blocking delay for 1 second (default time unit is ms)
        println("World!") // print after delay
    }
    println("Hello,") // main function continues while coroutine is delayed
    Thread.sleep(2000L) // block main thread for 2 seconds to keep JVM alive
}
```

> [Full code](kotlinx-coroutines-core/src/test/kotlin/guide/example-basic-01.kt)

```text
Hello,
World!
```

<!--- TEST -->

Coroutines are like light-weight threads launched with [launch] (a _coroutine builder_).
Replace
`launch(CommonPool) { ... }` with `thread { ... }` and `delay(...)` with `Thread.sleep(...)` to see a compiler error:

```
Error: Kotlin: Suspend functions are only allowed to be called from a coroutine or another suspend function
```

[delay] is a _suspending function_, thread non blocking, _suspends_ the coroutine, only called from other coroutines.

### Bridging blocking and non-blocking worlds

Mixing _non-blocking_ `delay(...)` and _blocking_ `Thread.sleep(...)` is hard to read. Refactor with [runBlocking]:

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> { // start main coroutine
    launch(CommonPool) { // create new coroutine in common thread pool
        delay(1000L)
        println("World!")
    }
    println("Hello,") // main coroutine continues while child is delayed
    delay(2000L) // non-blocking delay for 2 seconds to keep JVM alive
}
```

> [Full code](kotlinx-coroutines-core/src/test/kotlin/guide/example-basic-02.kt)

<!--- TEST
Hello,
World!
-->

Same result, but with a non-blocking [delay]. 

`runBlocking { ... }` is an adaptor, starting the top-level main coroutine. 
Code outside of `runBlocking` _blocks_, until the coroutine inside `runBlocking` is active. 

Suspending function unit-test example:
 
```kotlin
class MyTest {
    @Test
    fun testMySuspendingFunction() = runBlocking<Unit> {
        // here we can use suspending functions using any assertion style that we like
    }
}
```

<!--- CLEAR -->
 
### Waiting for a job

While a coroutine is working, let's explicitly wait (non-blocking way) until the background [Job] is complete:

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    val job = launch(CommonPool) { // create new coroutine and keep a reference to its Job
        delay(1000L)
        println("World!")
    }
    println("Hello,")
    job.join() // wait until child coroutine completes
}
```

> [Full code](kotlinx-coroutines-core/src/test/kotlin/guide/example-basic-03.kt)

<!--- TEST
Hello,
World!
-->

Same result, but the main coroutine is not tied to the background job duration.

### Extract function refactoring

Refactor `launch(CommonPool) { ... }` into a separate function. "Extract function" refactoring provides a new function with the  `suspend` modifier (a _suspending function_). They are called from within coroutines like regular functions, but can also call other suspending functions, like `delay`, to _suspend_ execution of the coroutine.

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    val job = launch(CommonPool) { doWorld() }
    println("Hello,")
    job.join()
}

// this is your first suspending function
suspend fun doWorld() {
    delay(1000L)
    println("World!")
}
```

> [Full code](kotlinx-coroutines-core/src/test/kotlin/guide/example-basic-04.kt)

<!--- TEST
Hello,
World!
-->

### Coroutines ARE light-weight

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    val jobs = List(100_000) { // create a lot of coroutines and list their jobs
        launch(CommonPool) {
            delay(1000L)
            print(".")
        }
    }
    jobs.forEach { it.join() } // wait for all jobs to complete
}
```

> [Full code](kotlinx-coroutines-core/src/test/kotlin/guide/example-basic-05.kt)

<!--- TEST lines.size == 1 && lines[0] == ".".repeat(100_000) -->

After one second 100K coroutines start and print a dot. Threads would produce an out of memory error.

### Coroutines are like daemon threads

A long-running coroutine that prints "I'm sleeping" twice a second, then returns from the main function after a delay:

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    launch(CommonPool) {
        repeat(1000) { i ->
            println("I'm sleeping $i ...")
            delay(500L)
        }
    }
    delay(1300L) // just quit after delay
}
```

> [Full code](kotlinx-coroutines-core/src/test/kotlin/guide/example-basic-06.kt)

Outputs three lines and terminates:

```text
I'm sleeping 0 ...
I'm sleeping 1 ...
I'm sleeping 2 ...
```

<!--- TEST -->

Like daemon threads, active coroutines do not keep the process alive.

## Cancellation and timeouts

### Cancelling coroutine execution

Implicitly terminating coroutines from "main" is no good for larger, long-running applications. Finer-grained control like the [launch] function returns a [Job], to cancel running coroutines:
 
```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    val job = launch(CommonPool) {
        repeat(1000) { i ->
            println("I'm sleeping $i ...")
            delay(500L)
        }
    }
    delay(1300L) // delay a bit
    println("main: I'm tired of waiting!")
    job.cancel() // cancels the job
    delay(1300L) // delay a bit to ensure it was cancelled indeed
    println("main: Now I can quit.")
}
``` 

> [Full code](kotlinx-coroutines-core/src/test/kotlin/guide/example-cancel-01.kt)

Output:

```text
I'm sleeping 0 ...
I'm sleeping 1 ...
I'm sleeping 2 ...
main: I'm tired of waiting!
main: Now I can quit.
```

<!--- TEST -->

Main invokes `job.cancel`, the other coroutine is cancelled, thus no output. 

### Cancellation is cooperative

* Coroutine cancellation is _cooperative_. 
* `kotlinx.coroutines` spending functions are _cancellable_, checking for cancellation, throwing [CancellationException] when cancelled. 
* A working coroutine that does not check for cancellation, cannot be cancelled:

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    val startTime = System.currentTimeMillis()
    val job = launch(CommonPool) {
        var nextPrintTime = startTime
        var i = 0
        while (i < 10) { // computation loop, just wastes CPU
            // print a message twice a second
            if (System.currentTimeMillis() >= nextPrintTime) {
                println("I'm sleeping ${i++} ...")
                nextPrintTime += 500L
            }
        }
    }
    delay(1300L) // delay a bit
    println("main: I'm tired of waiting!")
    job.cancel() // cancels the job
    delay(1300L) // delay a bit to see if it was cancelled....
    println("main: Now I can quit.")
}
```

> [Full code](kotlinx-coroutines-core/src/test/kotlin/guide/example-cancel-02.kt)

Outputs "I'm sleeping" after cancellation:

<!--- TEST 
I'm sleeping 0 ...
I'm sleeping 1 ...
I'm sleeping 2 ...
main: I'm tired of waiting!
I'm sleeping 3 ...
I'm sleeping 4 ...
I'm sleeping 5 ...
main: Now I can quit.
-->

### Making computation code cancellable

* Periodically invoke the suspending function [yield]
* Explicitly check the cancellation status. 

Replace `while (i < 10)` in the previous example with `while (isActive)`. 

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    val startTime = System.currentTimeMillis()
    val job = launch(CommonPool) {
        var nextPrintTime = startTime
        var i = 0
        while (isActive) { // cancellable computation loop
            // print a message twice a second
            if (System.currentTimeMillis() >= nextPrintTime) {
                println("I'm sleeping ${i++} ...")
                nextPrintTime += 500L
            }
        }
    }
    delay(1300L) // delay a bit
    println("main: I'm tired of waiting!")
    job.cancel() // cancels the job
    delay(1300L) // delay a bit to see if it was cancelled....
    println("main: Now I can quit.")
}
```

> [Full code](kotlinx-coroutines-core/src/test/kotlin/guide/example-cancel-03.kt)

The loop can be cancelled because [isActive][CoroutineScope.isActive] is a property inside coroutines via [CoroutineScope].

<!--- TEST
I'm sleeping 0 ...
I'm sleeping 1 ...
I'm sleeping 2 ...
main: I'm tired of waiting!
main: Now I can quit.
-->

### Closing resources with finally

Cancellable suspending functions throw [CancellationException] on cancellation, to be handled with `try {...} finally {...}`.

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    val job = launch(CommonPool) {
        try {
            repeat(1000) { i ->
                println("I'm sleeping $i ...")
                delay(500L)
            }
        } finally {
            println("I'm running finally")
        }
    }
    delay(1300L) // delay a bit
    println("main: I'm tired of waiting!")
    job.cancel() // cancels the job
    delay(1300L) // delay a bit to ensure it was cancelled indeed
    println("main: Now I can quit.")
}
``` 

> [Full code](kotlinx-coroutines-core/src/test/kotlin/guide/example-cancel-04.kt)

Output:

```text
I'm sleeping 0 ...
I'm sleeping 1 ...
I'm sleeping 2 ...
main: I'm tired of waiting!
I'm running finally
main: Now I can quit.
```

<!--- TEST -->

### Run non-cancellable block

Rarely, a suspending function in the `finally` block  is need, but doing so causes a [CancellationException], the coroutine is already cancelled. Use `run(NonCancellable) {...}`.
 
```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    val job = launch(CommonPool) {
        try {
            repeat(1000) { i ->
                println("I'm sleeping $i ...")
                delay(500L)
            }
        } finally {
            run(NonCancellable) {
                println("I'm running finally")
                delay(1000L)
                println("And I've just delayed for 1 sec because I'm non-cancellable")
            }
        }
    }
    delay(1300L) // delay a bit
    println("main: I'm tired of waiting!")
    job.cancel() // cancels the job
    delay(1300L) // delay a bit to ensure it was cancelled indeed
    println("main: Now I can quit.")
}
``` 

> [Full code](kotlinx-coroutines-core/src/test/kotlin/guide/example-cancel-05.kt)

<!--- TEST
I'm sleeping 0 ...
I'm sleeping 1 ...
I'm sleeping 2 ...
main: I'm tired of waiting!
I'm running finally
And I've just delayed for 1 sec because I'm non-cancellable
main: Now I can quit.
-->

### Timeout

Cancel after a timeout using [withTimeout].

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    withTimeout(1300L) {
        repeat(1000) { i ->
            println("I'm sleeping $i ...")
            delay(500L)
        }
    }
}
```

> [Full code](kotlinx-coroutines-core/src/test/kotlin/guide/example-cancel-06.kt)

Output:

```text
I'm sleeping 0 ...
I'm sleeping 1 ...
I'm sleeping 2 ...
Exception in thread "main" kotlinx.coroutines.experimental.TimeoutException: Timed out waiting for 1300 MILLISECONDS
```

<!--- TEST STARTS_WITH -->

The `TimeoutException` thrown by [withTimeout] is a private subclass of [CancellationException].
A cancelled coroutine throwing `CancellationException` is standard when using  `withTimeout` inside `main`. 

On cancellation exception, all resources will be closed. Wrap the timeout in a `try {...} catch (e: CancellationException) {...}` block, to perform actions on timeout.

## Composing suspending functions

### Sequential by default

To invoke suspending functions defined elsewhere _sequentially_:

<!--- INCLUDE .*/example-compose-([0-9]+).kt
import kotlin.system.measureTimeMillis
-->

```kotlin
suspend fun doSomethingUsefulOne(): Int {
    delay(1000L) // pretend we are doing something useful here
    return 13
}

suspend fun doSomethingUsefulTwo(): Int {
    delay(1000L) // pretend we are doing something useful here, too
    return 29
}
```

<!--- INCLUDE .*/example-compose-([0-9]+).kt -->

-- first `doSomethingUsefulOne` _and then_ `doSomethingUsefulTwo` and compute the sum of their results. Useful if we use the first results to decide on second function invocation.

Sequential invocation is _sequential_ by default:

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    val time = measureTimeMillis {
        val one = doSomethingUsefulOne()
        val two = doSomethingUsefulTwo()
        println("The answer is ${one + two}")
    }
    println("Completed in $time ms")
}
```

> [Full code](kotlinx-coroutines-core/src/test/kotlin/guide/example-compose-01.kt)

Output:

```text
The answer is 42
Completed in 2017 ms
```

<!--- TEST ARBITRARY_TIME -->

### Concurrent using async

If both `doSomethingUsefulOne` and `doSomethingUsefulTwo` are independent we do both _concurrently_ with [async]
 
[async] is just like [launch], starting a separate coroutine but `launch` returns a [Job] with a resulting value, `async` returns a [Deferred] -- a light-weight non-blocking future, a promise to provide a result later. Use `.await()` to get the pending result, but like a `Job`, you can cancel it.
 
```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    val time = measureTimeMillis {
        val one = async(CommonPool) { doSomethingUsefulOne() }
        val two = async(CommonPool) { doSomethingUsefulTwo() }
        println("The answer is ${one.await() + two.await()}")
    }
    println("Completed in $time ms")
}
```

> [Full code](kotlinx-coroutines-core/src/test/kotlin/guide/example-compose-02.kt)

Output:

```text
The answer is 42
Completed in 1017 ms
```

<!--- TEST ARBITRARY_TIME -->

2x fast, concurrency with coroutines is always explicit.

### Lazily started async

With [async] use the [CoroutineStart.LAZY] parameter to start the coroutine when the result is needed by [await][Deferred.await] or if [start][Job.start] is invoked:

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    val time = measureTimeMillis {
        val one = async(CommonPool, CoroutineStart.LAZY) { doSomethingUsefulOne() }
        val two = async(CommonPool, CoroutineStart.LAZY) { doSomethingUsefulTwo() }
        println("The answer is ${one.await() + two.await()}")
    }
    println("Completed in $time ms")
}
```

> [Full code](kotlinx-coroutines-core/src/test/kotlin/guide/example-compose-03.kt)

Output:

```text
The answer is 42
Completed in 2017 ms
```

<!--- TEST ARBITRARY_TIME -->

Execution is sequential,  we _first_ start and await for `one`, _and then_ start and await
for `two`. It is not the intended use-case for laziness. It is designed as a replacement for
the standard `lazy` function for computating values involving suspending functions.

### Async-style functions

Use the [async] coroutine builder to call `doSomethingUsefulOne` and `doSomethingUsefulTwo`
_asynchronously_ . Prefix these functions with "async" or "Async" suffix because they start asynchronous computation when you need the deferred value to get the result.

```kotlin
// The result type of asyncSomethingUsefulOne is Deferred<Int>
fun asyncSomethingUsefulOne() = async(CommonPool) {
    doSomethingUsefulOne()
}

// The result type of asyncSomethingUsefulTwo is Deferred<Int>
fun asyncSomethingUsefulTwo() = async(CommonPool)  {
    doSomethingUsefulTwo()
}
```

`asyncXXX` functions are **not** _suspending_ functions, they can be used from anywhere. Invocation implies _concurrent_ execution:

```kotlin
// note, that we don't have `runBlocking` to the right of `main` in this example
fun main(args: Array<String>) {
    val time = measureTimeMillis {
        // we can initiate async actions outside of a coroutine
        val one = asyncSomethingUsefulOne()
        val two = asyncSomethingUsefulTwo()
        // but waiting for a result must involve either suspending or blocking.
        // here we use `runBlocking { ... }` to block the main thread while waiting for the result
        runBlocking {
            println("The answer is ${one.await() + two.await()}")
        }
    }
    println("Completed in $time ms")
}
```

> [Full code](kotlinx-coroutines-core/src/test/kotlin/guide/example-compose-04.kt)

<!--- TEST ARBITRARY_TIME
The answer is 42
Completed in 1085 ms
-->

## Coroutine context and dispatchers

We've already seen `launch(CommonPool) {...}`, `async(CommonPool) {...}`, `run(NonCancellable) {...}`, etc.
In these code snippets [CommonPool] and [NonCancellable] are _coroutine contexts_. Let's see more.

### Dispatchers and threads

Coroutine context includes a [_coroutine dispatcher_][CoroutineDispatcher] determining what thread/s the corresponding coroutine uses for execution. 
Coroutine dispatcher can:
* Confine coroutine execution to a specific thread
* Dispatch it to a thread pool
* Run unconfined

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    val jobs = arrayListOf<Job>()
    jobs += launch(Unconfined) { // not confined -- will work with main thread
        println("      'Unconfined': I'm working in thread ${Thread.currentThread().name}")
    }
    jobs += launch(coroutineContext) { // context of the parent, runBlocking coroutine
        println("'coroutineContext': I'm working in thread ${Thread.currentThread().name}")
    }
    jobs += launch(CommonPool) { // will get dispatched to ForkJoinPool.commonPool (or equivalent)
        println("      'CommonPool': I'm working in thread ${Thread.currentThread().name}")
    }
    jobs += launch(newSingleThreadContext("MyOwnThread")) { // will get its own new thread
        println("          'newSTC': I'm working in thread ${Thread.currentThread().name}")
    }
    jobs.forEach { it.join() }
}
```

> [Full code](kotlinx-coroutines-core/src/test/kotlin/guide/example-context-01.kt)

Output order may vary:

```text
      'Unconfined': I'm working in thread main
      'CommonPool': I'm working in thread ForkJoinPool.commonPool-worker-1
          'newSTC': I'm working in thread MyOwnThread
'coroutineContext': I'm working in thread main
```

<!--- TEST LINES_START_UNORDERED -->

### Unconfined vs confined dispatcher
 
[Unconfined] coroutine dispatcher starts coroutines in the caller thread, until the first suspension point. After suspension it resumes in the thread of its next invokation. Unconfined dispatcher is used when the coroutine does not consume CPU time nor updates shared data (like UI) that is confined to a specific thread. 

[coroutineContext][CoroutineScope.coroutineContext] is a property inside a coroutine block refrencing the context via [CoroutineScope] interface.
A parent context can be inherited. [runBlocking] default context is confined to be invoker thread, inheriting it is the same as confining execution to this thread with predictable FIFO scheduling.

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    val jobs = arrayListOf<Job>()
    jobs += launch(Unconfined) { // not confined -- will work with main thread
        println("      'Unconfined': I'm working in thread ${Thread.currentThread().name}")
        delay(500)
        println("      'Unconfined': After delay in thread ${Thread.currentThread().name}")
    }
    jobs += launch(coroutineContext) { // context of the parent, runBlocking coroutine
        println("'coroutineContext': I'm working in thread ${Thread.currentThread().name}")
        delay(1000)
        println("'coroutineContext': After delay in thread ${Thread.currentThread().name}")
    }
    jobs.forEach { it.join() }
}
```

> [Full code](kotlinx-coroutines-core/src/test/kotlin/guide/example-context-02.kt)

Output: 
 
```text
      'Unconfined': I'm working in thread main
'coroutineContext': I'm working in thread main
      'Unconfined': After delay in thread kotlinx.coroutines.DefaultExecutor
'coroutineContext': After delay in thread main
```

<!--- TEST LINES_START -->
 
The coroutine inheriting `coroutineContext` of `runBlocking {...}` continues to execute
in the `main` thread, while the unconfined one resumed in the default executor thread that [delay]
function is using.

### Debugging coroutines and threads

Coroutines can suspend on one thread and resume on another thread with [Unconfined] dispatcher or 
with a multi-threaded dispatcher like [CommonPool]. Even with a single-threaded dispatcher it might be hard to
figure out what coroutine was doing what, where, and when. Debugging threaded applications by logging thread name(universal logging framework feature), does not give context for coroutines. `kotlinx.coroutines` includes debugging facilities to make it easier.

Run with `-Dkotlinx.coroutines.debug` JVM option:

```kotlin
fun log(msg: String) = println("[${Thread.currentThread().name}] $msg")

fun main(args: Array<String>) = runBlocking<Unit> {
    val a = async(coroutineContext) {
        log("I'm computing a piece of the answer")
        6
    }
    val b = async(coroutineContext) {
        log("I'm computing another piece of the answer")
        7
    }
    log("The answer is ${a.await() * b.await()}")
}
```

> [Full code](kotlinx-coroutines-core/src/test/kotlin/guide/example-context-03.kt)

Three coroutines:
* main (#1) -- `runBlocking`
* two computing deferred values `a` (#2) and `b` (#3).

All execute in the `runBlocking` context, confined to the main thread.
Output:

```text
[main @coroutine#2] I'm computing a piece of the answer
[main @coroutine#3] I'm computing another piece of the answer
[main @coroutine#1] The answer is 42
```

<!--- TEST -->

`log` function prints the name of the thread in square brackets like `main` thread, with the currently executing coroutine's identifier appended. Debugging mode consecutively assigns the identifier to created coroutines.

See doccumentation for debugging facilities for [newCoroutineContext] function.

### Jumping between threads

Run `-Dkotlinx.coroutines.debug` JVM option:

```kotlin
fun log(msg: String) = println("[${Thread.currentThread().name}] $msg")

fun main(args: Array<String>) {
    val ctx1 = newSingleThreadContext("Ctx1")
    val ctx2 = newSingleThreadContext("Ctx2")
    runBlocking(ctx1) {
        log("Started in ctx1")
        run(ctx2) {
            log("Working in ctx2")
        }
        log("Back to ctx1")
    }
}
```

> [Full code](kotlinx-coroutines-core/src/test/kotlin/guide/example-context-04.kt)

Two new techniques:
* [runBlocking] with an explicitly specified context
* [run] function changing the context while staying in the same coroutine

Output:

```text
[Ctx1 @coroutine#1] Started in ctx1
[Ctx2 @coroutine#1] Working in ctx2
[Ctx1 @coroutine#1] Back to ctx1
```

<!--- TEST -->

### Job in the context

Use `coroutineContext[Job]` to retrieve [Job] it from its context:

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    println("My job is ${coroutineContext[Job]}")
}
```

> [Full code](kotlinx-coroutines-core/src/test/kotlin/guide/example-context-05.kt)

Output:
```
My job is BlockingCoroutine{Active}@65ae6ba4
```

<!--- TEST lines.size == 1 && lines[0].startsWith("My job is BlockingCoroutine{Active}@") -->

[isActive][CoroutineScope.isActive] in [CoroutineScope] is a shortcut for `coroutineContext[Job]!!.isActive`.

### Children of a coroutine

* Launching [Job]'s from [coroutineContext][CoroutineScope.coroutineContext] have a parent-_child_ relationship.
* Cancelling the parent recursively cancels its children.
  
```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    // start a coroutine to process some kind of incoming request
    val request = launch(CommonPool) {
        // it spawns two other jobs, one with its separate context
        val job1 = launch(CommonPool) {
            println("job1: I have my own context and execute independently!")
            delay(1000)
            println("job1: I am not affected by cancellation of the request")
        }
        // and the other inherits the parent context
        val job2 = launch(coroutineContext) {
            println("job2: I am a child of the request coroutine")
            delay(1000)
            println("job2: I will not execute this line if my parent request is cancelled")
        }
        // request completes when both its sub-jobs complete:
        job1.join()
        job2.join()
    }
    delay(500)
    request.cancel() // cancel processing of the request
    delay(1000) // delay a second to see what happens
    println("main: Who has survived request cancellation?")
}
```

> [Full code](kotlinx-coroutines-core/src/test/kotlin/guide/example-context-06.kt)

Output:

```text
job1: I have my own context and execute independently!
job2: I am a child of the request coroutine
job1: I am not affected by cancellation of the request
main: Who has survived request cancellation?
```

<!--- TEST -->

### Combining contexts

Use `+` operator to combine contexts, the right hand context replaces relevant entries for the left hand context. A [Job] of the parent coroutine can be inherited, while its dispatcher replaced:

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    // start a coroutine to process some kind of incoming request
    val request = launch(coroutineContext) { // use the context of `runBlocking`
        // spawns CPU-intensive child job in CommonPool !!! 
        val job = launch(coroutineContext + CommonPool) {
            println("job: I am a child of the request coroutine, but with a different dispatcher")
            delay(1000)
            println("job: I will not execute this line if my parent request is cancelled")
        }
        job.join() // request completes when its sub-job completes
    }
    delay(500)
    request.cancel() // cancel processing of the request
    delay(1000) // delay a second to see what happens
    println("main: Who has survived request cancellation?")
}
```

> [Full code](kotlinx-coroutines-core/src/test/kotlin/guide/example-context-07.kt)

Output:

```text
job: I am a child of the request coroutine, but with a different dispatcher
main: Who has survived request cancellation?
```

<!--- TEST -->

### Naming coroutines for debugging

During debugging, explicitly name coroutines tied to specific processing requests, or ones in background task.
[CoroutineName] is displayed with the thread name during coroutine execution when debugging mode is turned on.

```kotlin
fun log(msg: String) = println("[${Thread.currentThread().name}] $msg")

fun main(args: Array<String>) = runBlocking(CoroutineName("main")) {
    log("Started main coroutine")
    // run two background value computations
    val v1 = async(CommonPool + CoroutineName("v1coroutine")) {
        log("Computing v1")
        delay(500)
        252
    }
    val v2 = async(CommonPool + CoroutineName("v2coroutine")) {
        log("Computing v2")
        delay(1000)
        6
    }
    log("The answer for v1 / v2 = ${v1.await() / v2.await()}")
}
```

> [Full code](kotlinx-coroutines-core/src/test/kotlin/guide/example-context-08.kt)

The output it produces with `-Dkotlinx.coroutines.debug` JVM option is similar to:
 
```text
[main @main#1] Started main coroutine
[ForkJoinPool.commonPool-worker-1 @v1coroutine#2] Computing v1
[ForkJoinPool.commonPool-worker-2 @v2coroutine#3] Computing v2
[main @main#1] The answer for v1 / v2 = 42
```

<!--- TEST FLEXIBLE_THREAD -->

### Cancellation via explicit job

In applications, objects with lifecycles launch coroutines performing asynch operations. The coroutines must be cancelled on object destruction, else memory leaks. Create a parent [Job] tied to the objects lifecycle, and create new jobs with the parents context using [`Job()`][Job] factory function. Call the parent's [Job.cancel] to terminate both it, and it's children.

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    val job = Job() // create a job object to manage our lifecycle
    // now launch ten coroutines for a demo, each working for a different time
    val coroutines = List(10) { i ->
        // they are all children of our job object
        launch(coroutineContext + job) { // we use the context of main runBlocking thread, but with our own job object
            delay(i * 200L) // variable delay 0ms, 200ms, 400ms, ... etc
            println("Coroutine $i is done")
        }
    }
    println("Launched ${coroutines.size} coroutines")
    delay(500L) // delay for half a second
    println("Cancelling job!")
    job.cancel() // cancel our job.. !!!
    delay(1000L) // delay for more to see if our coroutines are still working
}
```

> [Full code](kotlinx-coroutines-core/src/test/kotlin/guide/example-context-09.kt)

Output:

```text
Launched 10 coroutines
Coroutine 0 is done
Coroutine 1 is done
Coroutine 2 is done
Cancelling job!
```

<!--- TEST -->

The first three coroutines printed a message. The others were cancelled  via `job.cancel()`. In a hypothetical Android 
application, create a parent job object when the activity is created, use it to create child coroutines,
and cancel it when activity is destroyed.

## Channels

Deferred values provide a convenient way to transfer a single value between coroutines.
Channels provide a way to transfer a stream of values.

<!--- INCLUDE .*/example-channel-([0-9]+).kt
import kotlinx.coroutines.experimental.channels.*
-->

### Channel basics

A [Channel] is conceptually very similar to `BlockingQueue`. One key difference is that
instead of a blocking `put` operation it has a suspending [send][SendChannel.send], and instead of 
a blocking `take` operation it has a suspending [receive][ReceiveChannel.receive].

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    val channel = Channel<Int>()
    launch(CommonPool) {
        // this might be heavy CPU-consuming computation or async logic, we'll just send five squares
        for (x in 1..5) channel.send(x * x)
    }
    // here we print five received integers:
    repeat(5) { println(channel.receive()) }
    println("Done!")
}
```

> You can get full code [here](kotlinx-coroutines-core/src/test/kotlin/guide/example-channel-01.kt)

The output of this code is:

```text
1
4
9
16
25
Done!
```

<!--- TEST -->

### Closing and iteration over channels 

Unlike a queue, a channel can be closed to indicate that no more elements are coming. 
On the receiver side it is convenient to use a regular `for` loop to receive elements 
from the channel. 
 
Conceptually, a [close][SendChannel.close] is like sending a special close token to the channel. 
The iteration stops as soon as this close token is received, so there is a guarantee 
that all previously sent elements before the close are received:

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    val channel = Channel<Int>()
    launch(CommonPool) {
        for (x in 1..5) channel.send(x * x)
        channel.close() // we're done sending
    }
    // here we print received values using `for` loop (until the channel is closed)
    for (y in channel) println(y)
    println("Done!")
}
```

> You can get full code [here](kotlinx-coroutines-core/src/test/kotlin/guide/example-channel-02.kt)

<!--- TEST 
1
4
9
16
25
Done!
-->

### Building channel producers

The pattern where a coroutine is producing a sequence of elements is quite common. 
This is a part of _producer-consumer_ pattern that is often found in concurrent code. 
You could abstract such a producer into a function that takes channel as its parameter, but this goes contrary
to common sense that results must be returned from functions. 

There is a convenience coroutine builder named [produce] that makes it easy to do it right on producer side,
and an extension function [consumeEach], that can replace a `for` loop on the consumer side:

```kotlin
fun produceSquares() = produce<Int>(CommonPool) {
    for (x in 1..5) send(x * x)
}

fun main(args: Array<String>) = runBlocking<Unit> {
    val squares = produceSquares()
    squares.consumeEach { println(it) }
    println("Done!")
}
```

> You can get full code [here](kotlinx-coroutines-core/src/test/kotlin/guide/example-channel-03.kt)

<!--- TEST 
1
4
9
16
25
Done!
-->

### Pipelines

Pipeline is a pattern where one coroutine is producing, possibly infinite, stream of values:

```kotlin
fun produceNumbers() = produce<Int>(CommonPool) {
    var x = 1
    while (true) send(x++) // infinite stream of integers starting from 1
}
```

And another coroutine or coroutines are consuming that stream, doing some processing, and producing some other results.
In the below example the numbers are just squared:

```kotlin
fun square(numbers: ReceiveChannel<Int>) = produce<Int>(CommonPool) {
    for (x in numbers) send(x * x)
}
```

The main code starts and connects the whole pipeline:

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    val numbers = produceNumbers() // produces integers from 1 and on
    val squares = square(numbers) // squares integers
    for (i in 1..5) println(squares.receive()) // print first five
    println("Done!") // we are done
    squares.cancel() // need to cancel these coroutines in a larger app
    numbers.cancel()
}
```

> You can get full code [here](kotlinx-coroutines-core/src/test/kotlin/guide/example-channel-04.kt)

<!--- TEST 
1
4
9
16
25
Done!
-->

We don't have to cancel these coroutines in this example app, because
[coroutines are like daemon threads](#coroutines-are-like-daemon-threads), 
but in a larger app we'll need to stop our pipeline if we don't need it anymore.
Alternatively, we could have run pipeline coroutines as 
[children of a coroutine](#children-of-a-coroutine).

### Prime numbers with pipeline

Let's take pipelines to the extreme with an example that generates prime numbers using a pipeline 
of coroutines. We start with an infinite sequence of numbers. This time we introduce an 
explicit context parameter, so that caller can control where our coroutines run:
 
<!--- INCLUDE kotlinx-coroutines-core/src/test/kotlin/guide/example-channel-05.kt  
import kotlin.coroutines.experimental.CoroutineContext
-->
 
```kotlin
fun numbersFrom(context: CoroutineContext, start: Int) = produce<Int>(context) {
    var x = start
    while (true) send(x++) // infinite stream of integers from start
}
```

The following pipeline stage filters an incoming stream of numbers, removing all the numbers 
that are divisible by the given prime number:

```kotlin
fun filter(context: CoroutineContext, numbers: ReceiveChannel<Int>, prime: Int) = produce<Int>(context) {
    for (x in numbers) if (x % prime != 0) send(x)
}
```

Now we build our pipeline by starting a stream of numbers from 2, taking a prime number from the current channel, 
and launching new pipeline stage for each prime number found:
 
```
numbersFrom(2) -> filter(2) -> filter(3) -> filter(5) -> filter(7) ... 
``` 
 
The following example prints the first ten prime numbers, 
running the whole pipeline in the context of the main thread:

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    var cur = numbersFrom(coroutineContext, 2)
    for (i in 1..10) {
        val prime = cur.receive()
        println(prime)
        cur = filter(coroutineContext, cur, prime)
    }
}
```

> You can get full code [here](kotlinx-coroutines-core/src/test/kotlin/guide/example-channel-05.kt)

The output of this code is:

```text
2
3
5
7
11
13
17
19
23
29
```

<!--- TEST -->

Note, that you can build the same pipeline using `buildIterator` coroutine builder from the standard library. 
Replace `produce` with `buildIterator`, `send` with `yield`, `receive` with `next`, 
`ReceiveChannel` with `Iterator`, and get rid of the context. You will not need `runBlocking` either.
However, the benefit of a pipeline that uses channels as shown above is that it can actually use 
multiple CPU cores if you run it in [CommonPool] context.

Anyway, this is an extremely impractical way to find prime numbers. In practice, pipelines do involve some
other suspending invocations (like asynchronous calls to remote services) and these pipelines cannot be
built using `buildSeqeunce`/`buildIterator`, because they do not allow arbitrary suspension, unlike
`produce` which is fully asynchronous.
 
### Fan-out

Multiple coroutines may receive from the same channel, distributing work between themselves.
Let us start with a producer coroutine that is periodically producing integers 
(ten numbers per second):

```kotlin
fun produceNumbers() = produce<Int>(CommonPool) {
    var x = 1 // start from 1
    while (true) {
        send(x++) // produce next
        delay(100) // wait 0.1s
    }
}
```

Then we can have several processor coroutines. In this example, they just print their id and
received number:

```kotlin
fun launchProcessor(id: Int, channel: ReceiveChannel<Int>) = launch(CommonPool) {
    channel.consumeEach {
        println("Processor #$id received $it")
    }    
}
```

Now let us launch five processors and let them work for almost a second. See what happens:

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    val producer = produceNumbers()
    repeat(5) { launchProcessor(it, producer) }
    delay(950)
    producer.cancel() // cancel producer coroutine and thus kill them all
}
```

> You can get full code [here](kotlinx-coroutines-core/src/test/kotlin/guide/example-channel-06.kt)

The output will be similar to the the following one, albeit the processor ids that receive
each specific integer may be different:

```
Processor #2 received 1
Processor #4 received 2
Processor #0 received 3
Processor #1 received 4
Processor #3 received 5
Processor #2 received 6
Processor #4 received 7
Processor #0 received 8
Processor #1 received 9
Processor #3 received 10
```

<!--- TEST lines.size == 10 && lines.withIndex().all { (i, line) -> line.startsWith("Processor #") && line.endsWith(" received ${i + 1}") } -->

Note, that cancelling a producer coroutine closes its channel, thus eventually terminating iteration
over the channel that processor coroutines are doing.

### Fan-in

Multiple coroutines may send to the same channel.
For example, let us have a channel of strings, and a suspending function that 
repeatedly sends a specified string to this channel with a specified delay:

```kotlin
suspend fun sendString(channel: SendChannel<String>, s: String, time: Long) {
    while (true) {
        delay(time)
        channel.send(s)
    }
}
```

Now, let us see what happens if we launch a couple of coroutines sending strings 
(in this example we launch them in the context of the main thread):

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    val channel = Channel<String>()
    launch(coroutineContext) { sendString(channel, "foo", 200L) }
    launch(coroutineContext) { sendString(channel, "BAR!", 500L) }
    repeat(6) { // receive first six
        println(channel.receive())
    }
}
```

> You can get full code [here](kotlinx-coroutines-core/src/test/kotlin/guide/example-channel-07.kt)

The output is:

```text
foo
foo
BAR!
foo
foo
BAR!
```

<!--- TEST -->

### Buffered channels

The channels shown so far had no buffer. Unbuffered channels transfer elements when sender and receiver 
meet each other (aka rendezvous). If send is invoked first, then it is suspended until receive is invoked, 
if receive is invoked first, it is suspended until send is invoked.

Both [`Channel()`][Channel] factory function and [produce] builder take an optional `capacity` parameter to
specify _buffer size_. Buffer allows senders to send multiple elements before suspending, 
similar to the `BlockingQueue` with a specified capacity, which blocks when buffer is full.

Take a look at the behavior of the following code:

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    val channel = Channel<Int>(4) // create buffered channel
    launch(coroutineContext) { // launch sender coroutine
        repeat(10) {
            println("Sending $it") // print before sending each element
            channel.send(it) // will suspend when buffer is full
        }
    }
    // don't receive anything... just wait....
    delay(1000)
}
```

> You can get full code [here](kotlinx-coroutines-core/src/test/kotlin/guide/example-channel-08.kt)

Outputs "sending" _five_ times using a buffered channel with capacity of _four_:

```text
Sending 0
Sending 1
Sending 2
Sending 3
Sending 4
```

<!--- TEST -->

The first four elements are added to the buffer and the sender suspends when trying to send the fifth one.


### Channels are fair

Send and receive operations to channels are _fair_ with respect to the order of their invocation from 
multiple coroutines. They are served in first-in first-out order, e.g. the first coroutine to invoke `receive` 
gets the element. In the following example two coroutines "ping" and "pong" are 
receiving the "ball" object from the shared "table" channel. 

```kotlin
data class Ball(var hits: Int)

fun main(args: Array<String>) = runBlocking<Unit> {
    val table = Channel<Ball>() // a shared table
    launch(coroutineContext) { player("ping", table) }
    launch(coroutineContext) { player("pong", table) }
    table.send(Ball(0)) // serve the ball
    delay(1000) // delay 1 second
    table.receive() // game over, grab the ball
}

suspend fun player(name: String, table: Channel<Ball>) {
    for (ball in table) { // receive the ball in a loop
        ball.hits++
        println("$name $ball")
        delay(300) // wait a bit
        table.send(ball) // send the ball back
    }
}
```

> You can get full code [here](kotlinx-coroutines-core/src/test/kotlin/guide/example-channel-09.kt)

The "ping" coroutine is started first, so it is the first one to receive the ball. Even though "ping"
coroutine immediately starts receiving the ball again after sending it back to the table, the ball gets
received by the "pong" coroutine, because it was already waiting for it:

```text
ping Ball(hits=1)
pong Ball(hits=2)
ping Ball(hits=3)
pong Ball(hits=4)
ping Ball(hits=5)
```

<!--- TEST -->

## Shared mutable state and concurrency

Coroutines can be executed concurrently using a multi-threaded dispatcher like [CommonPool]. It presents
all the usual concurrency problems. The main problem being synchronization of access to **shared mutable state**. 
Some solutions to this problem in the land of coroutines are similar to the solutions in the multi-threaded world, 
but others are unique.

### The problem

Let us launch a thousand coroutines all doing the same action thousand times (for a total of a million executions). 
We'll also measure their completion time for further comparisons:

<!--- INCLUDE .*/example-sync-([0-9a-z]+).kt
import kotlin.coroutines.experimental.CoroutineContext
import kotlin.system.measureTimeMillis
-->

<!--- INCLUDE .*/example-sync-03.kt
import java.util.concurrent.atomic.AtomicInteger
-->

<!--- INCLUDE .*/example-sync-06.kt
import kotlinx.coroutines.experimental.sync.Mutex
-->

<!--- INCLUDE .*/example-sync-07.kt
import kotlinx.coroutines.experimental.channels.*
-->

```kotlin
suspend fun massiveRun(context: CoroutineContext, action: suspend () -> Unit) {
    val n = 1000 // number of coroutines to launch
    val k = 1000 // times an action is repeated by each coroutine
    val time = measureTimeMillis {
        val jobs = List(n) {
            launch(context) {
                repeat(k) { action() }
            }
        }
        jobs.forEach { it.join() }
    }
    println("Completed ${n * k} actions in $time ms")    
}
```

<!--- INCLUDE .*/example-sync-([0-9a-z]+).kt -->

We start with a very simple action that increments a shared mutable variable using 
multi-threaded [CommonPool] context. 

```kotlin
var counter = 0

fun main(args: Array<String>) = runBlocking<Unit> {
    massiveRun(CommonPool) {
        counter++
    }
    println("Counter = $counter")
}
```

> You can get full code [here](kotlinx-coroutines-core/src/test/kotlin/guide/example-sync-01.kt)

<!--- TEST LINES_START
Completed 1000000 actions in
Counter =
-->

What does it print at the end? It is highly unlikely to ever print "Counter = 1000000", because a thousand coroutines 
increment the `counter` concurrently from multiple threads without any synchronization.

> Note: if you have an old system with 2 or fewer CPUs, then you _will_ consistently see 1000000, because
`CommonPool` is running in only one thread in this case. To reproduce the problem you'll need to make the
following change:

```kotlin
val mtContext = newFixedThreadPoolContext(2, "mtPool") // explicitly define context with two threads
var counter = 0

fun main(args: Array<String>) = runBlocking<Unit> {
    massiveRun(mtContext) { // use it instead of CommonPool in this sample and below 
        counter++
    }
    println("Counter = $counter")
}
```

> You can get full code [here](kotlinx-coroutines-core/src/test/kotlin/guide/example-sync-01b.kt)

<!--- TEST LINES_START
Completed 1000000 actions in
Counter =
-->

### Volatiles are of no help

There is common misconception that making a variable `volatile` solves concurrency problem. Let us try it:

```kotlin
@Volatile // in Kotlin `volatile` is an annotation 
var counter = 0

fun main(args: Array<String>) = runBlocking<Unit> {
    massiveRun(CommonPool) {
        counter++
    }
    println("Counter = $counter")
}
```

> You can get full code [here](kotlinx-coroutines-core/src/test/kotlin/guide/example-sync-02.kt)

<!--- TEST LINES_START
Completed 1000000 actions in
Counter =
-->

This code works slower, but we still don't get "Counter = 1000000" at the end, because volatile variables guarantee
linearizable (this is a technical term for "atomic") reads and writes to the corresponding variable, but
do not provide atomicity of larger actions (increment in our case).

### Thread-safe data structures

The general solution that works both for threads and for coroutines is to use a thread-safe (aka synchronized,
linearizable, or atomic) data structure that provides all the necessarily synchronization for the corresponding 
operations that needs to be performed on a shared state. 
In the case of a simple counter we can use `AtomicInteger` class which has atomic `incrementAndGet` operations:

```kotlin
var counter = AtomicInteger()

fun main(args: Array<String>) = runBlocking<Unit> {
    massiveRun(CommonPool) {
        counter.incrementAndGet()
    }
    println("Counter = ${counter.get()}")
}
```

> You can get full code [here](kotlinx-coroutines-core/src/test/kotlin/guide/example-sync-03.kt)

<!--- TEST ARBITRARY_TIME
Completed 1000000 actions in xxx ms
Counter = 1000000
-->

This is the fastest solution for this particular problem. It works for plain counters, collections, queues and other
standard data structures and basic operations on them. However, it does not easily scale to complex
state or to complex operations that do not have ready-to-use thread-safe implementations. 

### Thread confinement fine-grained

_Thread confinement_ is an approach to the problem of shared mutable state where all access to the particular shared
state is confined to a single thread. It is typically used in UI applications, where all UI state is confined to 
the single event-dispatch/application thread. It is easy to apply with coroutines by using a  
single-threaded context:

```kotlin
val counterContext = newSingleThreadContext("CounterContext")
var counter = 0

fun main(args: Array<String>) = runBlocking<Unit> {
    massiveRun(CommonPool) { // run each coroutine in CommonPool
        run(counterContext) { // but confine each increment to the single-threaded context
            counter++
        }
    }
    println("Counter = $counter")
}
```

> You can get full code [here](kotlinx-coroutines-core/src/test/kotlin/guide/example-sync-04.kt)

<!--- TEST ARBITRARY_TIME
Completed 1000000 actions in xxx ms
Counter = 1000000
-->

This code works very slowly, because it does _fine-grained_ thread-confinement. Each individual increment switches 
from multi-threaded `CommonPool` context to the single-threaded context using [run] block. 

### Thread confinement coarse-grained

In practice, thread confinement is performed in large chunks, e.g. big pieces of state-updating business logic
are confined to the single thread. The following example does it like that, running each coroutine in 
the single-threaded context to start with.

```kotlin
val counterContext = newSingleThreadContext("CounterContext")
var counter = 0

fun main(args: Array<String>) = runBlocking<Unit> {
    massiveRun(counterContext) { // run each coroutine in the single-threaded context
        counter++
    }
    println("Counter = $counter")
}
```

> You can get full code [here](kotlinx-coroutines-core/src/test/kotlin/guide/example-sync-05.kt)

<!--- TEST ARBITRARY_TIME
Completed 1000000 actions in xxx ms
Counter = 1000000
-->

This now works much faster and produces correct result.

### Mutual exclusion

Mutual exclusion solution to the problem is to protect all modifications of the shared state with a _critical section_
that is never executed concurrently. In a blocking world you'd typically use `synchronized` or `ReentrantLock` for that.
Coroutine's alternative is called [Mutex]. It has [lock][Mutex.lock] and [unlock][Mutex.unlock] functions to 
delimit a critical section. The key difference is that `Mutex.lock` is a suspending function. It does not block a thread.

```kotlin
val mutex = Mutex()
var counter = 0

fun main(args: Array<String>) = runBlocking<Unit> {
    massiveRun(CommonPool) {
        mutex.lock()
        try { counter++ }
        finally { mutex.unlock() }
    }
    println("Counter = $counter")
}
```

> You can get full code [here](kotlinx-coroutines-core/src/test/kotlin/guide/example-sync-06.kt)

<!--- TEST ARBITRARY_TIME
Completed 1000000 actions in xxx ms
Counter = 1000000
-->

The locking in this example is fine-grained, so it pays the price. However, it is a good choice for some situations
where you absolutely must modify some shared state periodically, but there is no natural thread that this state
is confined to.

### Actors

An actor is a combination of a coroutine, the state that is confined and is encapsulated into this coroutine,
and a channel to communicate with other coroutines. A simple actor can be written as a function, 
but an actor with a complex state is better suited for a class. 

There is an [actor] coroutine builder that conveniently combines actor's mailbox channel into its 
scope to receive messages from and combines the send channel into the resulting job object, so that a 
single reference to the actor can be carried around as its handle.

The first step of using an actor is to define a class of messages that an actor is going to process.
Kotlin's [sealed classes](https://kotlinlang.org/docs/reference/sealed-classes.html) are well suited for that purpose.
We define `CounterMsg` sealed class with `IncCounter` message to increment a counter and `GetCounter` message
to get its value. The later needs to send a response. A [CompletableDeferred] communication
primitive, that represents a single value that will be known (communicated) in the future,
is used here for that purpose.

```kotlin
// Message types for counterActor
sealed class CounterMsg
object IncCounter : CounterMsg() // one-way message to increment counter
class GetCounter(val response: CompletableDeferred<Int>) : CounterMsg() // a request with reply
```

Then we define a function that launches an actor using an [actor] coroutine builder:

```kotlin
// This function launches a new counter actor
fun counterActor() = actor<CounterMsg>(CommonPool) {
    var counter = 0 // actor state
    for (msg in channel) { // iterate over incoming messages
        when (msg) {
            is IncCounter -> counter++
            is GetCounter -> msg.response.complete(counter)
        }
    }
}
```

The main code is straightforward:

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    val counter = counterActor() // create the actor
    massiveRun(CommonPool) {
        counter.send(IncCounter)
    }
    // send a message to get a counter value from an actor
    val response = CompletableDeferred<Int>()
    counter.send(GetCounter(response))
    println("Counter = ${response.await()}")
    counter.close() // shutdown the actor
}
```

> You can get full code [here](kotlinx-coroutines-core/src/test/kotlin/guide/example-sync-07.kt)

<!--- TEST ARBITRARY_TIME
Completed 1000000 actions in xxx ms
Counter = 1000000
-->

It does not matter (for correctness) what context the actor itself is executed in. An actor is
a coroutine and a coroutine is executed sequentially, so confinement of the state to the specific coroutine
works as a solution to the problem of shared mutable state.

Actor is more efficient than locking under load, because in this case it always has work to do and it does not 
have to switch to a different context at all.

> Note, that an [actor] coroutine builder is a dual of [produce] coroutine builder. An actor is associated 
  with the channel that it receives messages from, while a producer is associated with the channel that it 
  sends elements to.

## Select expression

Select expression makes it possible to await multiple suspending functions simultaneously and _select_
the first one that becomes available.

<!--- INCLUDE .*/example-select-([0-9]+).kt
import kotlinx.coroutines.experimental.channels.*
import kotlinx.coroutines.experimental.selects.*
-->

### Selecting from channels

Let us have two producers of strings: `fizz` and `buzz`. The `fizz` produces "Fizz" string every 300 ms:

<!--- INCLUDE .*/example-select-01.kt
import kotlin.coroutines.experimental.CoroutineContext
-->

```kotlin
fun fizz(context: CoroutineContext) = produce<String>(context) {
    while (true) { // sends "Fizz" every 300 ms
        delay(300)
        send("Fizz")
    }
}
```

And the `buzz` produces "Buzz!" string every 500 ms:

```kotlin
fun buzz(context: CoroutineContext) = produce<String>(context) {
    while (true) { // sends "Buzz!" every 500 ms
        delay(500)
        send("Buzz!")
    }
}
```

Using [receive][ReceiveChannel.receive] suspending function we can receive _either_ from one channel or the
other. But [select] expression allows us to receive from _both_ simultaneously using its
[onReceive][SelectBuilder.onReceive] clauses:

```kotlin
suspend fun selectFizzBuzz(fizz: ReceiveChannel<String>, buzz: ReceiveChannel<String>) {
    select<Unit> { // <Unit> means that this select expression does not produce any result 
        fizz.onReceive { value ->  // this is the first select clause
            println("fizz -> '$value'")
        }
        buzz.onReceive { value ->  // this is the second select clause
            println("buzz -> '$value'")
        }
    }
}
```

Let us run it all seven times:

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    val fizz = fizz(coroutineContext)
    val buzz = buzz(coroutineContext)
    repeat(7) {
        selectFizzBuzz(fizz, buzz)
    }
}
```

> You can get full code [here](kotlinx-coroutines-core/src/test/kotlin/guide/example-select-01.kt)

The result of this code is: 

```text
fizz -> 'Fizz'
buzz -> 'Buzz!'
fizz -> 'Fizz'
fizz -> 'Fizz'
buzz -> 'Buzz!'
fizz -> 'Fizz'
buzz -> 'Buzz!'
```

<!--- TEST -->

### Selecting on close

The [onReceive][SelectBuilder.onReceive] clause in `select` fails when the channel is closed and the corresponding
`select` throws an exception. We can use [onReceiveOrNull][SelectBuilder.onReceiveOrNull] clause to perform a
specific action when the channel is closed. The following example also shows that `select` is an expression that returns 
the result of its selected clause:

```kotlin
suspend fun selectAorB(a: ReceiveChannel<String>, b: ReceiveChannel<String>): String =
    select<String> {
        a.onReceiveOrNull { value -> 
            if (value == null) 
                "Channel 'a' is closed" 
            else 
                "a -> '$value'"
        }
        b.onReceiveOrNull { value -> 
            if (value == null) 
                "Channel 'b' is closed"
            else    
                "b -> '$value'"
        }
    }
```

Let's use it with channel `a` that produces "Hello" string four times and 
channel `b` that produces "World" four times:

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    // we are using the context of the main thread in this example for predictability ... 
    val a = produce<String>(coroutineContext) {
        repeat(4) { send("Hello $it") }
    }
    val b = produce<String>(coroutineContext) {
        repeat(4) { send("World $it") }
    }
    repeat(8) { // print first eight results
        println(selectAorB(a, b))
    }
}
```

> You can get full code [here](kotlinx-coroutines-core/src/test/kotlin/guide/example-select-02.kt)

The result of this code is quite interesting, so we'll analyze it in mode detail:

```text
a -> 'Hello 0'
a -> 'Hello 1'
b -> 'World 0'
a -> 'Hello 2'
a -> 'Hello 3'
b -> 'World 1'
Channel 'a' is closed
Channel 'a' is closed
```

<!--- TEST -->

There are couple of observations to make out of it. 

First of all, `select` is _biased_ to the first clause. When several clauses are selectable at the same time, 
the first one among them gets selected. Here, both channels are constantly producing strings, so `a` channel,
being the first clause in select, wins. However, because we are using unbuffered channel, the `a` gets suspended from
time to time on its [send][SendChannel.send] invocation and gives a chance for `b` to send, too.

The second observation, is that [onReceiveOrNull][SelectBuilder.onReceiveOrNull] gets immediately selected when the 
channel is already closed.

### Selecting to send

Select expression has [onSend][SelectBuilder.onSend] clause that can be used for a great good in combination 
with a biased nature of selection.

Let us write an example of producer of integers that sends its values to a `side` channel when 
the consumers on its primary channel cannot keep up with it:

```kotlin
fun produceNumbers(side: SendChannel<Int>) = produce<Int>(CommonPool) {
    for (num in 1..10) { // produce 10 numbers from 1 to 10
        delay(100) // every 100 ms
        select<Unit> {
            onSend(num) {} // Send to the primary channel
            side.onSend(num) {} // or to the side channel     
        }
    }
}
```

Consumer is going to be quite slow, taking 250 ms to process each number:
 
```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    val side = Channel<Int>() // allocate side channel
    launch(coroutineContext) { // this is a very fast consumer for the side channel
        side.consumeEach { println("Side channel has $it") }
    }
    produceNumbers(side).consumeEach { 
        println("Consuming $it")
        delay(250) // let us digest the consumed number properly, do not hurry
    }
    println("Done consuming")
}
``` 
 
> You can get full code [here](kotlinx-coroutines-core/src/test/kotlin/guide/example-select-03.kt)
  
So let us see what happens:
 
```text
Consuming 1
Side channel has 2
Side channel has 3
Consuming 4
Side channel has 5
Side channel has 6
Consuming 7
Side channel has 8
Side channel has 9
Consuming 10
Done consuming
```

<!--- TEST -->

### Selecting deferred values

Deferred values can be selected using [onAwait][SelectBuilder.onAwait] clause. 
Let us start with an async function that returns a deferred string value after 
a random delay:

<!--- INCLUDE .*/example-select-04.kt
import java.util.*
-->

```kotlin
fun asyncString(time: Int) = async(CommonPool) {
    delay(time.toLong())
    "Waited for $time ms"
}
```

Let us start a dozen of them with a random delay.

```kotlin
fun asyncStringsList(): List<Deferred<String>> {
    val random = Random(3)
    return List(12) { asyncString(random.nextInt(1000)) }
}
```

Now the main function awaits for the first of them to complete and counts the number of deferred values
that are still active. Note, that we've used here the fact that `select` expression is a Kotlin DSL, 
so we can provide clauses for it using an arbitrary code. In this case we iterate over a list
of deferred values to provide `onAwait` clause for each deferred value.

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    val list = asyncStringsList()
    val result = select<String> {
        list.withIndex().forEach { (index, deferred) ->
            deferred.onAwait { answer ->
                "Deferred $index produced answer '$answer'"
            }
        }
    }
    println(result)
    val countActive = list.count { it.isActive }
    println("$countActive coroutines are still active")
}
```

> You can get full code [here](kotlinx-coroutines-core/src/test/kotlin/guide/example-select-04.kt)

The output is:

```text
Deferred 4 produced answer 'Waited for 128 ms'
11 coroutines are still active
```

<!--- TEST -->

### Switch over a channel of deferred values

Let us write a channel producer function that consumes a channel of deferred string values, waits for each received
deferred value, but only until the next deferred value comes over or the channel is closed. This example puts together 
[onReceiveOrNull][SelectBuilder.onReceiveOrNull] and [onAwait][SelectBuilder.onAwait] clauses in the same `select`:

```kotlin
fun switchMapDeferreds(input: ReceiveChannel<Deferred<String>>) = produce<String>(CommonPool) {
    var current = input.receive() // start with first received deferred value
    while (isActive) { // loop while not cancelled/closed
        val next = select<Deferred<String>?> { // return next deferred value from this select or null
            input.onReceiveOrNull { update ->
                update // replaces next value to wait
            }
            current.onAwait { value ->  
                send(value) // send value that current deferred has produced
                input.receiveOrNull() // and use the next deferred from the input channel
            }
        }
        if (next == null) {
            println("Channel was closed")
            break // out of loop
        } else {
            current = next
        }
    }
}
```

To test it, we'll use a simple async function that resolves to a specified string after a specified time:

```kotlin
fun asyncString(str: String, time: Long) = async(CommonPool) {
    delay(time)
    str
}
```

The main function just launches a coroutine to print results of `switchMapDeferreds` and sends some test
data to it:

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    val chan = Channel<Deferred<String>>() // the channel for test
    launch(coroutineContext) { // launch printing coroutine
        for (s in switchMapDeferreds(chan)) 
            println(s) // print each received string
    }
    chan.send(asyncString("BEGIN", 100))
    delay(200) // enough time for "BEGIN" to be produced
    chan.send(asyncString("Slow", 500))
    delay(100) // not enough time to produce slow
    chan.send(asyncString("Replace", 100))
    delay(500) // give it time before the last one
    chan.send(asyncString("END", 500))
    delay(1000) // give it time to process
    chan.close() // close the channel ... 
    delay(500) // and wait some time to let it finish
}
```

> You can get full code [here](kotlinx-coroutines-core/src/test/kotlin/guide/example-select-05.kt)

The result of this code:

```text
BEGIN
Replace
END
Channel was closed
```

<!--- TEST -->

## Further reading

* [Guide to UI programming with coroutines](ui/coroutines-guide-ui.md)
* [Guide to reactive streams with coroutines](reactive/coroutines-guide-reactive.md)
* [Coroutines design document (KEEP)](https://github.com/Kotlin/kotlin-coroutines/blob/master/kotlin-coroutines-informal.md)
* [Full kotlinx.coroutines API reference](http://kotlin.github.io/kotlinx.coroutines)

<!--- MODULE kotlinx-coroutines-core -->
<!--- INDEX kotlinx.coroutines.experimental -->
[launch]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/launch.html
[delay]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/delay.html
[runBlocking]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/run-blocking.html
[Job]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/-job/index.html
[CancellationException]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/-cancellation-exception.html
[yield]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/yield.html
[CoroutineScope.isActive]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/-coroutine-scope/is-active.html
[CoroutineScope]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/-coroutine-scope/index.html
[run]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/run.html
[NonCancellable]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/-non-cancellable/index.html
[withTimeout]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/with-timeout.html
[async]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/async.html
[Deferred]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/-deferred/index.html
[CoroutineStart.LAZY]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/-coroutine-start/-l-a-z-y.html
[Deferred.await]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/-deferred/await.html
[Job.start]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/-job/start.html
[CommonPool]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/-common-pool/index.html
[CoroutineDispatcher]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/-coroutine-dispatcher/index.html
[CoroutineScope.coroutineContext]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/-coroutine-scope/coroutine-context.html
[Unconfined]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/-unconfined/index.html
[newCoroutineContext]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/new-coroutine-context.html
[CoroutineName]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/-coroutine-name/index.html
[Job.cancel]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/-job/cancel.html
[CompletableDeferred]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/-completable-deferred/index.html
<!--- INDEX kotlinx.coroutines.experimental.sync -->
[Mutex]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental.sync/-mutex/index.html
[Mutex.lock]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental.sync/-mutex/lock.html
[Mutex.unlock]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental.sync/-mutex/unlock.html
<!--- INDEX kotlinx.coroutines.experimental.channels -->
[Channel]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental.channels/-channel/index.html
[SendChannel.send]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental.channels/-send-channel/send.html
[ReceiveChannel.receive]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental.channels/-receive-channel/receive.html
[SendChannel.close]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental.channels/-send-channel/close.html
[produce]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental.channels/produce.html
[consumeEach]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental.channels/consume-each.html
[actor]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental.channels/actor.html
<!--- INDEX kotlinx.coroutines.experimental.selects -->
[select]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental.selects/select.html
[SelectBuilder.onReceive]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental.selects/-select-builder/on-receive.html
[SelectBuilder.onReceiveOrNull]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental.selects/-select-builder/on-receive-or-null.html
[SelectBuilder.onSend]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental.selects/-select-builder/on-send.html
[SelectBuilder.onAwait]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental.selects/-select-builder/on-await.html
<!--- END -->

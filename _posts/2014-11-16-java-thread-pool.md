---
title: "Using Java ExecutorService (a.k.a. ThreadPool)"
description: "How to use Java thread pools."
tags: [java]
---

In my intern project this summer, I extensively used `ExecutorService`, which is an implementation of thread pool in JDK. It's well designed and easy to use but also has several caveats. In this article, I'll walk you through  the usages of `ExecutorService` and discuss how to properly handle the caveats as well.

## Why thread pool?

(Feel free to skip this part if you already knows the answer.)

To get an idea of why thread pool is useful, let's first suppose we are to build a web server. A web server is a software, as you know, that serves HTTP requests and returns HTTP responses. So how do we build it?

A naive solution is to process requests in a <abbr title="First Come First Serve">FCFS</abbr> fashion: when our server becomes idle, check if there's any pending request. If there is, process it and return response; otherwise sleep for a while. This implementation would work quite well if the rate of requests is slower than the rate of processing.

But our web server should be general enough to handle different loads of traffic, right? How do we handle this? Multi-threading should be an immediate response. And here's a naive design of a web server: every time a request comes, spawn a new worker thread to handle it. This time, we should be able to handle as many requests concurrently as the system resource permits.

Wait! The system resource is limited. If our system can handle a maximum of 10K connections per second, what if there are 100K connections? The system resource would be exhausted and we'll face unrecoverable disaster.

"Okay," you say, "let's enforce some limitation on number of worker threads so that system resource is not exhausted."

No problem. Let's change the design to: every time a request comes, spawn a new worker thread to handle it *only if the number of concurrent worker threads hasn't reached limit*; if the limit is reached, wait until at least one running worker thread finishes. This is a big improvement! We've reached a balance between handling as many requests as possible and not exhausting system resources.

What's left? If you know how threads are implemented, you know that a thread has its own set of context information, e.g. stack, registers, etc. As a result of this fact, creating and destroying a thread is not free of cost. In fact, on some platforms, creating a thread is as expensive as creating a process. This indicates that creating a worker thread for every request will have a non-negligible impact on performance.

This problem hints us to the point of reusing worker threads: when a thread has finished its request pick up a pending one if there is. And a thread is not destroyed unless there's an absolute need, e.g. some irrecoverable error happened. In this way, the cost of thread creation is bounded by the number of worker threads, instead of by the number of requests.

And this is the basic idea of why [thread pool](http://en.wikipedia.org/wiki/Thread_pool_pattern) is useful.

## Basic Usage with ExecutorService

In Java, an `ExecutorService` is what you use to execute asynchronous jobs, and `FixedSizeThreadPool` is the default implementation of it. The `Executors` class provides different factory methods for creating different `ExecutorService`s. What we are going to use right now is `Executors.newFixedThreadPool(int nThreads)`.

The `ExecutorService` interface is mostly straight-forward to use. I'll highlight the following methods:

- `submit(Runnable task)`: submit a task to run
- `submit(Callable<T> task)`: the same as above, but the task returns a value
- `shutdown()`: stop accepting new task submissions
- `shutdownNow()`: stop all tasks, including those under execution, returns a list of unexecuted tasks
- `awaitTermination(long timeout, TimeUnit unit)`: blocked waiting for termination with a time out

Putting the pieces together, here's a simple demo of how `ExecutorService` is typically used:

<script src="https://gist.github.com/kavinyao/2de4a72f01964e9ba2ac.js"></script>

Here a thread pool of 2 worker threads is created and 10 jobs are submitted. If you run this program, you'll see output like this:

```
Thread pool-1-thread-2 is printing 1.
Thread pool-1-thread-1 is printing 0.
Thread pool-1-thread-2 is printing 2.
Thread pool-1-thread-2 is printing 4.
Thread pool-1-thread-1 is printing 3.
Thread pool-1-thread-2 is printing 5.
Thread pool-1-thread-1 is printing 6.
Thread pool-1-thread-2 is printing 7.
Thread pool-1-thread-1 is printing 8.
Thread pool-1-thread-2 is printing 9.
```

We can see that it is exactly two worker threads running. Note that although jobs are enqueued to a blocking queue, there's no guarantee that they will be executed in the same order – this is what you'll expect from multi-threading.

## Caveat 1: shutdown and subsequent job submission

It is worth noting that calling `shutdown` does **not** terminate the `ExecutorService` immediately (maybe you already noticed this if you read carefully enough ;)). Rather, `shutdown` only prevents new tasks from being submitted. The pending tasks will get a change to be executed unless `shutdownNow` is called. Another case of bad naming in JDK.

As a result, if you uncomment the last statement in `ExecutorServiceDemo`, an exception will be thrown, telling you that this job submissions rejected, because `pool.shutdown()` is already called in `shutdownAndAwaitTermination`.

## Caveat 2: unhandled runtime exception

As you would always care about when using multi-threading, what if a `RuntimeException` occurs but gets uncaught? Let's try and see:

<script src="https://gist.github.com/kavinyao/b255109e75bf910d7a94.js"></script>

A possible output is:

```
Thread pool-1-thread-2 is printing 1.
Thread pool-1-thread-1 is printing 0.
Thread pool-1-thread-2 is printing 4.
Thread pool-1-thread-1 is printing 6.
Thread pool-1-thread-2 is printing 7.
```

Wait! Exception happened but we got nothing in stdout?! That's right. Only in the main thread, the stack trace of an uncaught exception is printed.

Letting go an exception silently is absolutely bad. What should we do? It turns out `ExecutorService.submit` returns a `Future` object and any uncaught exception will be thrown if we call `Future.get` on that object. So let's collect the `Future` objects and test them after job execution:

```java
public static void main(String[] args) {
    ExecutorService threadPool = Executors.newFixedThreadPool(2);

    List<Future<?>> futures = new ArrayList<Future<?>>();
    for (int i = 0; i < 10; ++i) {
        futures.add(threadPool.submit(new ExceptionalNumberPrintingJob(i)));
    }

    shutdownAndAwaitTermination(threadPool);

    for (Future<?> future : futures) {
        try {
            future.get();
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (ExecutionException e) {
            // here it is!
            // the uncaught exception is wrapped in Execution Exception
            e.printStackTrace();
        }
    }
}
```

Boom! Now we can guarantee that no exception shall pass silently. But can we do better? The hard requirement of collecting the `Future`s manually and calling `Future.get` on every object is not only cumbersome but also sometimes infeasible. Suppose you are creating a generic job executor which lets clients submit jobs as they wish and will run as long as possible, explicitly collecting `Future` objects would impose unnecessary requirement on clients and it would be difficult to decide when to call `Future.get` as we don't know when a job execution is finished. Yes `Future.get` has a timeout version but this only adds to the complexity.

Fortunately, there's a way that the `ExecutorService` automatically handles (or at least reports) uncaught exceptions by [overriding the `afterExecute` method](http://stackoverflow.com/q/2248131/1240620):

<script src="https://gist.github.com/kavinyao/2408edcb9cf15d6acc30.js"></script>

The `ErrorReportingThreadPoolExecutor` is a drop-in replacement of `Executors.newFixedThreadPool` but handles all possible exception happened in job execution. You can modify it based on your needs. For example, you can log the exceptions instead of printing out them.

*Side Note: you might think setting an `UncaughtExceptionHandler` to the worker threads might do the same trick. But the answer is [NO](http://stackoverflow.com/q/1838923/1240620). It is because the exception is actually handled by the `ExecutorService`: it is wrapped into a `ExecutionException` object and will be thrown if `Future.get` is called.*

## Job Scheduling with ScheduledExecutorService

`ScheduledExecutorService` an enhanced version of `ExecutorService` which can explicitly run jobs at scheduled time and periodically run jobs. Here's an example:

<script src="https://gist.github.com/kavinyao/2ebdd14003d954b4798a.js"></script>

So here we're scheduling a job to run after 10 seconds with `schedule` method and scheduling another job to run periodically every 3 seconds with `scheduleAtFixedRate` method. After 20 seconds, we terminate the executor service.

As what you would expect, the output is:

```
Periodic job is executed after 0 seconds.
Periodic job is executed after 3 seconds.
Periodic job is executed after 6 seconds.
Periodic job is executed after 9 seconds.
Delayed job is executed after 10 seconds.
Periodic job is executed after 12 seconds.
Periodic job is executed after 15 seconds.
Periodic job is executed after 18 seconds.
```

The ability of explicitly setting when to execute a job makes `ScheduledExecutorService` extremely flexible. It's very common to do things periodically, e.g. sending out a heartbeat. The internal implementation of `ScheduledExecutorService` only creates a job object when necessary so it's very space-efficient.

What you need to keep in mind is that `ScheduledExecutorService` provides no strong guarantee about the actual execution time. As a result, if you do not have enough worker threads, future job execution might get delayed. We'll discuss how to properly choose number of worker threads later.

## Caveat 3: uncaught exception in repeated jobs

As always, using `ScheduledExecutorService` is not without caveats. If you read the [JavaDocs](https://docs.oracle.com/javase/7/docs/api/java/util/concurrent/ScheduledExecutorService.html#scheduleAtFixedRate(java.lang.Runnable,%20long,%20long,%20java.util.concurrent.TimeUnit)) carefully, you'd notice this:

>  If any execution of the task encounters an exception, subsequent executions are suppressed.

This means that if you have a periodic job and an exception occurred during the execution and is unhandled, this job will not be executed any more. Clearly, this behavior is rather counter-intuitive as you might think for periodic jobs, a different instance is used for each execution. Sadly, it's not the case and someone has already [become crazy about this](http://code.nomad-labs.com/2011/12/09/mother-fk-the-scheduledexecutorservice/).

The solution, as suggested in [this answer](http://stackoverflow.com/a/1660946/1240620), is to use `Future.get` to test if something wrong happened. So the solution to Caveat 2 also works! :)

## Conclusion

In this article we introduced how to use `ExecutorService` and `ScheduledExecutorService` in Java with examples. By highlighting the caveats, hopefully your journey with thread pools in Java will be smoother and more enjoyable.

There are other topics that are not covered in this article. For example, we didn't explain the difference between `Runnable` and `Callable`, the difference between `scheduleAtFixedRate` and `scheduleWithFixedDelay`, how many threads to use, how to use `ThreadFactory` to systematically name threads, etc. As this article is already long enough, these topics may deserve another one.
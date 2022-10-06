## Vert.x Async/Await

Use virtual threads to write Vert.x code that looks like it is synchronous.

- await Vert.x futures
- more meaningful stack traces

You still write the traditional Vert.x code with events, but you have the opportunity to write synchronous code for complex
workflows and use thread locals in such workflows.

This PoC is based on the [Async/Await support by August Nagro](https://github.com/AugustNagro/vertx-async-await).

### Examples

```java
Async async = new Async(vertx);
async.run(v -> {
  // Run on a Vert.x a virtual thread
  HttpClient client = vertx.createHttpClient();
  HttpClientRequest req = await(client.request(HttpMethod.GET, 8080, "localhost", "/"));
  HttpClientResponse resp = await(req.send());
  int status = resp.statusCode();
  Buffer body = await(resp.body());
});
```

You can view all [examples](vertx-async-await-incubator/src/main/java/examples/AsyncExamples.java).

### What this is about

Async/Await for Vert.x

### What this is not about

Blocking on other JDK blocking constructs such as latches, locks, sleep, etc... (since it would imply to have a multi-threaded application).

NOTE: it remains possible to block on these constructs using `await`

### What you get

Vert.x default application model dispatches events on the event-loop.

```java
request.send(response -> {
  // Set the buffer for handlers
  response.handler(buffers -> {

  });
});
```

Using virtual threads with Vert.x requires to run application tasks on a virtual threads

```java
Thread.startVirtualThread(() -> {
  CompletableFuture<HttpClientResponse> fut = new CompletableFuture<>();
  request.send(response -> {
    fut.complete(response);
  });
  HttpClientResponse response = fut.get();
  // As we get the response the virtual thread, there is a window of time where the event-loop thread
  // as already sent buffers and we lost these events
  response.handler(buffer -> {

  });
});
```

This project implements a Vert.x Context with virtual threads in order to provide a race free model. All events are
dispatched to the same virtual thread, so when a virtual thread waits for an asynchronous result, the next events
remain in the context execution queue until the virtual thread is resumed.

```java
Future<HttpClientResponse> fut = request.send();
HttpClientResponse response = await(fut);
// Buffer events might be in the queue and if they are, they will be dispatched next
response.handler(buffer -> {

});
```

When a virtual thread awaits a future, a new virtual thread can be started to handle new events and avoid potential
self dead-locks, e.g in the following example, awaiting the response does not prevent the timer to fire

```java
Promise<HttpClientResponse> promise = context.promise();
vertx.setTimer(timeout, id -> promise.tryFail("Too late"));
request.send().onComplete(promise);
try {
  HttpClientResponse response = await(promise.future());
} catch (Exception timeout) {
  // Too late
}
```

### Supported primitives

#### Futures

You can await a Vert.x `Future`

```java
Buffer body = await(response.body());
```

or a JDK `CompletionStage`

```java
Buffer body = await(response.body().toCompletionStage());
```

#### Locks

You can lock a `java.util.concurrent.locks`

```java
lock(theLock);
  try {
  //
} finally {
  theLock.unlock();
}
```

### Thread local support

Thread locals are only reliable within the execution of a context task.

```java
ThreadLocal<String> local = new ThreadLocal();
local.set(userId);
HttpClientRequest req = Async.await(client.request(HttpMethod.GET, 8080, "localhost", "/"));
HttpClientResponse resp = Async.await(req.send());
// Thread local remains the same since it's the same virtual thread
```

### Virtual thread scheduling

#### Default scheduler

The implementation schedules virtual threads on the built-in scheduler.

#### Event loop scheduler

An implementation (using internal java API) schedules virtual threads on vertx event-loops.

```java
Async async = new Async(vertx, true);
```

It removes the need for Vert.x to be aware of thread blocking, e.g

```java
lock.lock();
try {
  //
} finally {
  theLock.unlock();
}
```

It requires to open the `java.base` and `java.lang` modules:

```xml
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-surefire-plugin</artifactId>
  <version>3.0.0-M4</version>
  <configuration>
    <argLine>--enable-preview  --add-opens=java.base/java.lang=ALL-UNNAMED</argLine>
  </configuration>
</plugin>
```

### How it works

`VirtualThreadContext` implements `io.vertx.core.Context` and runs Vert.x task on virtual threads.

Like other context implementations `VirtualThreadContext` serializes tasks, so that events are serialized on the virtual thread.

When the virtual thread awaits a future, the virtual thread is parked and a new virtual thread can be started to continue handling tasks

When a future is completed, the virtual thread is unparked and is executed next (the future might be completed by a context task like a timer) and preempts all the context pending tasks.

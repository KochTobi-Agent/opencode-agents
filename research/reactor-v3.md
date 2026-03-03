# Project Reactor (v3)

> **Note**: As of the latest stable release information, Reactor 3.7.x is the current stable version (with 3.7.4 being a stable release). The BOM version is `2024.0.14`. This document covers Reactor 3.7.

## Overview

Project Reactor is a fully non-blocking reactive programming foundation for the JVM, with efficient demand management (in the form of managing "backpressure"). It integrates directly with Java 8 functional APIs, notably `CompletableFuture`, `Stream`, and `Duration`. 

Reactor offers composable asynchronous sequence APIs:
- **`Flux`**: For sequences of 0 to N elements (like a reactive Stream)
- **`Mono`**: For asynchronous 0 or 1 results (like a reactive Optional)

Reactor extensively implements the **Reactive Streams** specification and is the foundation for **Spring WebFlux**. It supports non-blocking inter-process communication with the `reactor-netty` project.

### When to Use Reactor

- Building non-blocking, event-driven microservices
- High-concurrency systems that need efficient resource utilization
- Streaming data processing
- Building reactive APIs with Spring WebFlux

### Reactive Streams Overview

Reactive Streams is a specification for asynchronous stream processing with non-blocking backpressure. It defines four interfaces:
- `Publisher<T>`: Emits elements to subscribers
- `Subscriber<T>`: Consumes elements from publishers
- `Subscription`: Controls the demand (backpressure)
- `Processor<T,R>`: Both a publisher and subscriber

## Key Concepts

### Flux (0..N elements)

`Flux` is a Reactive Streams `Publisher` that emits 0 to N elements, then completes (successfully or with an error).

```java
Flux<String> flux = Flux.just("apple", "banana", "cherry");
Flux<Integer> numbers = Flux.range(1, 10);
```

### Mono (0..1 element)

`Mono` is a Reactive Streams `Publisher` that emits at most one item via the `onNext` signal, then terminates with `onComplete`, or emits a single `onError` signal.

```java
Mono<String> mono = Mono.just("hello");
Mono<Void> empty = Mono.empty();
```

### Backpressure

Backpressure is the ability to control the rate at which a publisher emits elements. When a subscriber cannot keep up with the publisher's rate, backpressure allows the subscriber to signal how many elements it can handle.

Reactor handles backpressure through:
- **Request-based**: The subscriber requests a specific number of elements
- **Operators**: Transform and control the demand

### Hot vs Cold Publishers

**Cold Publishers**:
- Generate data **anew** for each subscriber
- Each subscription gets the full sequence from the beginning
- Examples: `Flux.just()`, `Flux.fromIterable()`, `Flux.generate()`

```java
// Cold example - each subscriber gets full sequence
Flux<Integer> cold = Flux.range(1, 5);
cold.subscribe(i -> System.out.println("A: " + i));
cold.subscribe(i -> System.out.println("B: " + i));
// Both A and B see 1,2,3,4,5
```

**Hot Publishers**:
- Do not depend on subscribers
- Emit data regardless of who's listening
- Late subscribers may miss earlier emissions
- Examples: `Flux.interval()`, event streams, WebSocket feeds

```java
// Hot example - share() converts cold to hot
Flux<Long> hot = Flux.interval(Duration.ofSeconds(1)).share();
hot.subscribe(i -> System.out.println("A: " + i));
Thread.sleep(2500);
// Subscriber B joins late
hot.subscribe(i -> System.out.println("B: " + i));
```

## Setup

### Maven Dependencies

```xml
<!-- In dependencyManagement section first -->
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>io.projectreactor</groupId>
            <artifactId>reactor-bom</artifactId>
            <version>2024.0.14</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>

<!-- Then in dependencies -->
<dependencies>
    <dependency>
        <groupId>io.projectreactor</groupId>
        <artifactId>reactor-core</artifactId>
    </dependency>
    <!-- For testing -->
    <dependency>
        <groupId>io.projectreactor</groupId>
        <artifactId>reactor-test</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

### Gradle Dependencies

```groovy
implementation 'io.projectreactor:reactor-core'

// For testing
testImplementation 'io.projectreactor:reactor-test'
```

## Common Operators

### Transformation Operators

#### map
Transforms each element emitted by applying a function.

```java
Flux.just(1, 2, 3)
    .map(i -> i * 2)
    .subscribe(System.out::println); // 2, 4, 6
```

#### flatMap / flatMapMany
Transforms each element into a publisher, then flattens the emissions. Use `flatMap` for Mono results and `flatMapMany` for Flux results.

```java
Flux.just(1, 2, 3)
    .flatMap(i -> Flux.just(i, i * 10))
    .subscribe(System.out::println); // 1, 10, 2, 20, 3, 30
```

#### filter
Filters elements based on a predicate.

```java
Flux.range(1, 10)
    .filter(i -> i % 2 == 0)
    .subscribe(System.out::println); // 2, 4, 6, 8, 10
```

#### reduce / scan
- `reduce`: Aggregates all elements into a single value
- `scan`: Emits each intermediate result

```java
Flux.range(1, 5)
    .reduce(0, (acc, i) -> acc + i)
    .subscribe(System.out::println); // 15

Flux.range(1, 5)
    .scan(0, (acc, i) -> acc + i)
    .subscribe(System.out::println); // 1, 3, 6, 10, 15
```

#### switchIfEmpty / defaultIfEmpty
- `switchIfEmpty`: Switch to an alternative publisher if empty
- `defaultIfEmpty`: Emit a default value if empty

```java
Mono.empty()
    .switchIfEmpty(Mono.just("fallback"))
    .subscribe(System.out::println); // "fallback"

Mono.empty()
    .defaultIfEmpty("default")
    .subscribe(System.out::println); // "default"
```

### Combination Operators

#### merge
Interleaves emissions from multiple publishers.

```java
Flux.merge(
    Flux.just("A", "B"),
    Flux.just("1", "2")
).subscribe(System.out::println); // A, 1, B, 2 (interleaved)
```

#### concat
Concatenates publishers sequentially (no interleaving).

```java
Flux.concat(
    Flux.just("A", "B"),
    Flux.just("1", "2")
).subscribe(System.out::println); // A, B, 1, 2
```

#### zip
Combines elements from multiple publishers into tuples.

```java
Flux.zip(
    Flux.just("A", "B", "C"),
    Flux.just(1, 2, 3)
).subscribe(t -> System.out.println(t.getT1() + t.getT2()));
// A1, B2, C3
```

#### combineLatest
Combines the latest elements from each publisher.

```java
Flux.combineLatest(
    Flux.just("A", "B", "C"),
    Flux.just(1, 2),
    (letter, num) -> letter + num
).subscribe(System.out::println);
// B1, B2, C2
```

### Error Handling

#### onErrorReturn
Returns a static fallback value when an error occurs.

```java
Flux.just(1, 2, 0)
    .map(i -> 10 / i)
    .onErrorReturn(-1)
    .subscribe(
        v -> System.out.println(v),
        e -> System.out.println("Error: " + e)
    ); // 10, 5, -1
```

#### onErrorResume
Provides a fallback publisher dynamically based on the error.

```java
remoteService.getData()
    .onErrorResume(error -> {
        if (error instanceof TimeoutException) {
            return fallbackService.getCachedData();
        }
        return Mono.error(error); // re-throw other errors
    });
```

#### doOnError
Side-effect only - logs or handles error without changing the sequence.

```java
remoteService.getData()
    .doOnError(e -> log.error("Failed to get data", e))
    .subscribe();
```

#### retry / retryWhen
- `retry`: Re-subscribes on error (be careful - infinite loops!)
- `retryWhen`: Advanced retry with backoff

```java
// Simple retry
remoteService.getData()
    .retry(3)
    .subscribe();

// Exponential backoff
remoteService.getData()
    .retryWhen(Retry.backoff(3, Duration.ofMillis(100))
        .filter(e -> e instanceof IOException))
    .subscribe();
```

### Scheduling Operators

#### subscribeOn
Specifies the `Scheduler` on which the **source** will be subscribed. Placement in the chain doesn't matter - it always affects the source.

```java
Flux.just(1, 2, 3)
    .subscribeOn(Schedulers.boundedElastic())
    .map(i -> i * 2)
    .subscribe();
```

#### publishOn
Specifies the `Scheduler` for operators **after** it in the chain. Placement matters - affects downstream operators.

```java
Flux.just(1, 2, 3)
    .publishOn(Schedulers.parallel())
    .map(i -> i * 2)  // runs on parallel scheduler
    .subscribe();
```

### Schedulers

Reactor provides several built-in schedulers:

| Scheduler | Description |
|-----------|-------------|
| ` Schedulers.immediate()` | Executes on the current thread |
| `Schedulers.single()` | Single reusable thread |
| `Schedulers.parallel()` | Fixed pool of N threads (CPU cores) |
| `Schedulers.boundedElastic()` | Elastic pool for blocking tasks (default 10x CPU cores) |
| `Schedulers.fromExecutorService()` | Custom executor service |

```java
// Create custom scheduler
Scheduler custom = Schedulers.newParallel("my-scheduler", 4);

// Schedulers.boundedElastic() - for blocking operations
// Don't use in non-blocking code!
```

### Filtering/Backpressure

#### limitRate
Controls the number of elements requested from upstream in batches.

```java
Flux.range(1, 100)
    .limitRate(10)  // Request 10 at a time
    .subscribe();
```

#### buffer
Collects elements into lists/batches.

```java
Flux.range(1, 10)
    .buffer(3)
    .subscribe(list -> System.out.println(list));
// [1,2,3], [4,5,6], [7,8,9], [10]
```

#### window
Similar to buffer but emits windows as Flux.

```java
Flux.range(1, 5)
    .window(2)
    .flatMap(flux -> flux.collectList())
    .subscribe(list -> System.out.println(list));
// [1,2], [3,4], [5]
```

### Debugging

#### log
Logs all signals in the reactive chain.

```java
Flux.just(1, 2, 3)
    .log("my flux")
    .subscribe();
```

#### tap (Reactor 3.5+)
A powerful operator for debugging and observability. Allows you to intercept signals without modifying the stream.

```java
Flux.just(1, 2, 3)
    .tap(SignalLogger.of("debug"))
    .subscribe();
    
// With Micrometer metrics
Flux.just(1, 2, 3)
    .tap(Micrometer.metrics(registry))
    .subscribe();
```

#### doOnSuccess / doOnCancel / doOnTerminate
Side-effect operators for debugging:

```java
mono
    .doOnSuccess(value -> System.out.println("Success: " + value))
    .doOnError(error -> System.out.println("Error: " + error))
    .doOnCancel(() -> System.out.println("Cancelled"))
    .doOnTerminate(() -> System.out.println("Terminated"))
    .subscribe();
```

#### checkpoint
Creates a user-defined checkpoint for debugging stack traces.

```java
Flux.just(1, 2, 0)
    .map(i -> 10 / i)
    .checkpoint("division operation")
    .subscribe();
```

## Context API

The Context API provides a way to pass data through the reactive chain, similar to HTTP headers. It's immutable and propagates from subscriber up the chain.

### Writing to Context

```java
// Using contextWrite
Mono.just("data")
    .contextWrite(Context.of("user", "alice"))
    .subscribe();

// Using deferContextual
Mono.deferContextual(ctx -> 
    Mono.just("User: " + ctx.get("user"))
)
.contextWrite(Context.of("user", "alice"))
.subscribe(); // "User: alice"
```

### Reading from Context

```java
Mono.just("hello")
    .flatMap(msg -> Mono.deferContextual(ctx -> 
        Mono.just(msg + ", " + ctx.get("user"))
    ))
    .contextWrite(Context.of("user", "bob"))
    .subscribe(); // "hello, bob"
```

### Important Notes

1. Context flows from **bottom to top** (subscriber to source)
2. `contextWrite` should come **last** in the chain to cover all operators
3. Inner sequences in `flatMap` have their own isolated Context

```java
Flux.range(1, 3)
    .flatMap(i -> 
        Mono.deferContextual(ctx -> 
            Mono.just("item=" + i + ", user=" + ctx.get("user"))
        )
    )
    .contextWrite(Context.of("user", "alice"))
    .subscribe();
// item=1, user=alice
// item=2, user=alice
// item=3, user=alice
```

## Threading Model

### How subscribeOn Works

`subscribeOn` affects the thread where the **subscription** happens and typically where the source emits. It doesn't matter where you place it in the chain - it always affects the source.

```java
// Both are equivalent
Flux.just(1, 2)
    .subscribeOn(Schedulers.boundedElastic())
    .map(...);

Flux.just(1, 2)
    .map(...)
    .subscribeOn(Schedulers.boundedElastic());
```

### How publishOn Works

`publishOn` affects the thread for all operators **after** it in the chain. Placement matters!

```java
Flux.just(1, 2, 3)
    .map(i -> { 
        // runs on main thread
        return i * 2; 
    })
    .publishOn(Schedulers.parallel())
    .map(i -> { 
        // runs on parallel scheduler thread
        return "value " + i; 
    })
    .subscribe();
```

### Execution Flow Summary

1. Nothing happens until you **subscribe**
2. When subscribed, a chain of Subscriptions is created
3. `subscribeOn` controls where the **source** runs
4. `publishOn` switches threads for downstream operators
5. Operators default to running on the thread where the previous operator completed

## Code Examples

### Creating Flux/Mono

```java
// From values
Flux.just("a", "b", "c");
Mono.just("value");

// From collections
Flux.fromIterable(list);
Flux.fromArray(array);

// From streams
Flux.fromStream(stream);

// Generate sequences
Flux.generate(sink -> {
    sink.next("value");
    sink.complete();
});

// Interval (hot!)
Flux.interval(Duration.ofSeconds(1));

// Empty
Flux.empty();
Mono.empty();

// Error
Flux.error(new RuntimeException("oops"));
Mono.error(new RuntimeException("oops"));

// Defer - lazy creation
Mono.defer(() -> expensiveOperation());
```

### Transforming Data

```java
// Map - synchronous transformation
Flux.just("a", "b", "c")
    .map(String::toUpperCase)
    .subscribe(); // A, B, C

// FlatMap - async transformation
Flux.just(1, 2, 3)
    .flatMap(i -> fetchAsync(i))  // Returns Mono or Flux
    .subscribe();

// Filter
Flux.range(1, 10)
    .filter(i -> i > 5)
    .subscribe(); // 6, 7, 8, 9, 10
```

### Error Handling

```java
// Return fallback on error
remoteCall()
    .onErrorReturn("fallback value")
    .subscribe();

// Switch to fallback publisher
remoteCall()
    .onErrorResume(error -> {
        if (error instanceof TimeoutException) {
            return fallbackCache.get();
        }
        return Mono.error(error);
    })
    .subscribe();

// Retry with backoff
remoteCall()
    .retryWhen(Retry.backoff(3, Duration.ofMillis(100))
        .filter(e -> e instanceof IOException)
        .doBeforeRetry(s -> log.info("Retrying...")))
    .subscribe();
```

### Parallel Execution with flatMap

```java
// Parallel execution using flatMap with parallel scheduler
List<String> ids = Arrays.asList("1", "2", "3", "4");

Flux.fromIterable(ids)
    .flatMap(id -> 
        Mono.fromCallable(() -> processItem(id))
            .subscribeOn(Schedulers.boundedElastic())
    )
    .collectList()
    .subscribe();
```

### Using Context

```java
// Propagating user info through chain
Mono<User> getUser(String userId) {
    return webClient.get()
        .uri("/users/" + userId)
        .retrieve()
        .bodyToMono(User.class)
        .contextWrite(Context.of("correlationId", UUID.randomUUID().toString()));
}

// Reading context in a service
Mono<String> processData(String input) {
    return Mono.deferContextual(ctx -> {
        String correlationId = ctx.get("correlationId");
        log.info("Processing with correlation: {}", correlationId);
        return doProcess(input);
    });
}

// Context with flatMap - inner sequences are isolated
Flux.just(1, 2, 3)
    .flatMap(i -> 
        Mono.deferContextual(ctx -> 
            Mono.just(i * 2)
                .map(v -> "user=" + ctx.get("user") + ", val=" + v)
        )
        .contextWrite(Context.of("user", "user-" + i))  // Only affects inner sequence
    )
    .contextWrite(Context.of("user", "main"))
    .subscribe();
// Each flatMap inner sequence has its own context
```

## Notes

### Best Practices

1. **Avoid blocking in reactive chains**: Use `subscribeOn(Schedulers.boundedElastic())` for blocking operations, never block on the main thread.

2. **Use appropriate Mono/Flux**: Use `Mono` when you expect 0 or 1 element, `Flux` for multiple elements.

3. **Handle errors**: Always have error handling - errors are terminal events in reactive streams.

4. **Be careful with flatMap**: Each inner subscription runs concurrently. Use `concatMap` for sequential processing.

5. **Context placement**: Write context last, read context from within the chain.

6. **Avoid shared mutable state**: Lambdas in operators may be shared between subscribers.

### Common Pitfalls

1. **Forgetting to subscribe**: Nothing happens until you subscribe!
   ```java
   // Nothing happens!
   Flux.just(1, 2, 3).map(i -> i * 2);
   
   // Now it runs
   Flux.just(1, 2, 3).map(i -> i * 2).subscribe();
   ```

2. **Blocking in reactive chains**: Causes performance issues and potential deadlocks.

3. **Infinite retry loops**: Always set limits on retry attempts.

4. **Not understanding hot vs cold**: Cold publishers replay for each subscriber; hot publishers don't.

5. **Thread confusion**: Remember `subscribeOn` affects the source, `publishOn` affects downstream.

6. **Context immutability**: You can't modify context, only create new ones with `put`.

### Additional Resources

- [Project Reactor Reference Guide](https://projectreactor.io/docs/core/release/reference/)
- [Reactive Streams Specification](https://www.reactive-streams.org/)
- [Spring WebFlux Documentation](https://docs.spring.io/spring-framework/docs/current/reference/html/web-reactive.html)

# 21.5 — Reactive Libraries

> **Libraries covered:** Reactive Streams Specification · Project Reactor (Mono & Flux) · RxJava · Mutiny (Quarkus)
>
> Reactive programming is one of the most conceptually challenging shifts in Java development — not because the syntax is difficult, but because it requires a fundamentally different mental model of how your program executes. This file builds from the foundations upward: first the specification that all reactive libraries share, then Project Reactor (the most important reactive library in the Spring ecosystem), then RxJava, then Mutiny.

---

## 🧠 Why Reactive Programming Exists

To understand reactive programming, you first need to feel the pain it addresses.

Imagine a REST controller that handles a request by querying a database, calling an external API, and combining the results. In traditional blocking Java, each I/O operation occupies a thread while waiting for a response. The thread isn't doing useful work — it's just sitting there, blocked, consuming memory (typically 256KB–1MB of stack space) and an OS thread handle.

Under moderate load this is fine. Under high load, all available threads are blocked waiting for I/O, new requests queue up, latency climbs, and eventually the server stops accepting requests. You can add more threads, but threads are expensive — a JVM with 1,000 threads is already under significant memory pressure.

Reactive programming solves this by making I/O operations non-blocking at the programming model level. Instead of "do this, then wait for the result, then continue," you describe a **pipeline of transformations**: "when data arrives, transform it like this, then do that, then produce this output." No thread is held waiting. When data is ready, a callback advances the pipeline. One thread can interleave the progress of thousands of concurrent operations. This is the same model used by Node.js, but brought to Java with full type safety and a rich operator vocabulary.

> **Important context:** Java 21 Virtual Threads (Phase 20.3) significantly reduce the need for reactive programming for pure I/O-bound throughput. Virtual Threads give you high concurrency with blocking code. However, reactive libraries provide something Virtual Threads do not: **backpressure**, **stream composition operators**, and **lazy pipelines** that only pull data when consumers are ready. For complex data streaming, transformation pipelines, and systems that integrate with databases via R2DBC, reactive programming remains valuable.

---

## 📜 The Reactive Streams Specification

The Reactive Streams specification (defined at `reactivestreams.org`, now part of Java 9 as `java.util.concurrent.Flow`) is the common contract that all reactive libraries implement. It defines four interfaces and one critical mechanism: **backpressure**.

Backpressure is the ability of a consumer to signal to a producer how much data it can handle. Without backpressure, a fast producer overwhelming a slow consumer causes buffers to fill up and memory to run out. With backpressure, the consumer controls the flow: "give me 10 items," processes them, then requests more when ready.

```java
// java.util.concurrent.Flow — Java 9 standard interfaces for reactive streams

// Publisher<T>: produces items of type T
public interface Publisher<T> {
    void subscribe(Subscriber<? super T> subscriber);
}

// Subscriber<T>: consumes items of type T
public interface Subscriber<T> {
    void onSubscribe(Subscription subscription); // Called first — gives the subscriber a Subscription
    void onNext(T item);                         // Called for each item
    void onError(Throwable throwable);           // Called on terminal error (no more items)
    void onComplete();                           // Called on successful completion (no more items)
}

// Subscription: represents the link between Publisher and Subscriber
public interface Subscription {
    void request(long n);  // Ask for n more items (backpressure mechanism)
    void cancel();         // Cancel the subscription
}

// Processor<T,R>: both a Subscriber and a Publisher — transforms a stream
public interface Processor<T, R> extends Subscriber<T>, Publisher<R> {}
```

The lifecycle of a reactive stream always follows this contract:
1. A `Subscriber` calls `publisher.subscribe(this)`.
2. The `Publisher` calls `subscriber.onSubscribe(subscription)`.
3. The `Subscriber` calls `subscription.request(n)` to signal it wants items.
4. The `Publisher` calls `subscriber.onNext(item)` up to `n` times.
5. Steps 3 and 4 repeat until completion.
6. The `Publisher` calls either `subscriber.onComplete()` or `subscriber.onError(throwable)`.

You will almost never implement these interfaces directly — libraries like Project Reactor do that for you. But knowing the spec helps you understand what "subscribing", "requesting", and "backpressure" mean when you encounter them in Reactor or RxJava APIs.

---

## ⚛️ Project Reactor — Spring's Reactive Foundation

Project Reactor (`io.projectreactor:reactor-core`) is the reactive library at the heart of Spring WebFlux. It implements the Reactive Streams spec and provides two primary types: `Mono<T>` and `Flux<T>`. Think of them as the reactive equivalents of `Optional<T>` and `Stream<T>` respectively — but with asynchronous, non-blocking execution and the ability to represent an ongoing stream of data.

**`Mono<T>`** represents a publisher that emits zero or one element, then completes (or emits an error). It is the reactive equivalent of a `Future<T>` or `Optional<T>` — use it when an operation produces at most one result, such as finding a user by ID or creating an entity.

**`Flux<T>`** represents a publisher that emits zero to N elements, then completes. It is the reactive equivalent of a `Stream<T>` — use it when an operation produces multiple results, such as listing all users or streaming records from a database.

### 21.5.1 Creating Monos and Fluxes

```java
import reactor.core.publisher.Mono;
import reactor.core.publisher.Flux;
import java.time.Duration;

// ── Creating Mono ──────────────────────────────────────────────────────────────

Mono<String> just       = Mono.just("hello");           // Emit "hello" then complete
Mono<String> empty      = Mono.empty();                 // Complete immediately with no value
Mono<String> error      = Mono.error(new RuntimeException("oops")); // Error immediately
Mono<String> never      = Mono.never();                 // Never emits anything (for testing)

// Defer: create a new Mono for each subscriber — important for mutable state
Mono<LocalDateTime> deferred = Mono.defer(() -> Mono.just(LocalDateTime.now()));
// Without defer, the time would be captured once at Mono creation.
// With defer, each subscriber gets the time at the moment they subscribe.

// From a CompletableFuture (bridge from traditional async code)
CompletableFuture<User> future = userRepository.findByIdAsync(42L);
Mono<User> fromFuture = Mono.fromFuture(future);

// From a Callable (executed lazily on subscription)
Mono<String> fromCallable = Mono.fromCallable(() -> expensiveOperation());

// fromSupplier: same as fromCallable but for Supplier (no checked exceptions)
Mono<Config> config = Mono.fromSupplier(() -> loadConfiguration());

// ── Creating Flux ─────────────────────────────────────────────────────────────

Flux<Integer> ofItems     = Flux.just(1, 2, 3, 4, 5);
Flux<Integer> fromList    = Flux.fromIterable(List.of(1, 2, 3, 4, 5));
Flux<Integer> fromArray   = Flux.fromArray(new Integer[]{1, 2, 3});
Flux<Integer> fromStream  = Flux.fromStream(Stream.of(1, 2, 3)); // Careful: stream is consumed once

// Range: generate a sequence of integers (like IntStream.range)
Flux<Integer> range = Flux.range(1, 10);  // 1, 2, 3, ..., 10

// Interval: emit a long (0, 1, 2, ...) on a time schedule
Flux<Long> ticker = Flux.interval(Duration.ofSeconds(1));  // Emits 0,1,2,... every second

// Generate: programmatic, synchronous generation with state
Flux<String> generated = Flux.generate(
    () -> 0,                               // Initial state
    (state, sink) -> {
        sink.next("Item " + state);        // Emit one value per call
        if (state == 4) sink.complete();   // Signal completion
        return state + 1;                  // Return next state
    }
);
// Emits: "Item 0", "Item 1", "Item 2", "Item 3", "Item 4", then completes

// Create: asynchronous, multi-threaded emission (for bridging callbacks)
Flux<Event> fromCallback = Flux.create(sink -> {
    eventBus.register(event -> sink.next(event));       // Emit on each event
    eventBus.onShutdown(sink::complete);               // Complete on shutdown
    eventBus.onError(sink::error);                     // Error on failure
    sink.onCancel(() -> eventBus.unregister());        // Clean up on cancellation
});
```

### 21.5.2 Transforming Operators

The richness of Project Reactor lies in its operators — methods you chain on a `Mono` or `Flux` to build a pipeline. Think of them like Java Streams operators but with async, error handling, and concurrency built in.

```java
// ── map: synchronous 1-to-1 transformation ────────────────────────────────────
Flux<String> names = Flux.just(1L, 2L, 3L)
    .map(id -> userRepository.findByIdSync(id).getName());  // Blocking call — avoid in reactive!

// ── flatMap: async 1-to-1 (or 1-to-many) transformation ─────────────────────
// Returns a Mono/Flux for each input. The inner publishers are subscribed to concurrently
// and their results are merged (order may not match input order).
Flux<UserDto> users = Flux.just(1L, 2L, 3L)
    .flatMap(id -> userRepository.findById(id));  // Concurrent, unordered

// concatMap: like flatMap but sequential — waits for each inner Mono to complete
// before subscribing to the next. Preserves order. Slower than flatMap for I/O.
Flux<UserDto> orderedUsers = Flux.just(1L, 2L, 3L)
    .concatMap(id -> userRepository.findById(id));  // Sequential, ordered

// flatMapSequential: subscribes concurrently (like flatMap) but reorders results
// to match input order. Best of both worlds for order-sensitive concurrent ops.
Flux<UserDto> concurrentOrdered = Flux.just(1L, 2L, 3L)
    .flatMapSequential(id -> userRepository.findById(id));

// ── filter: keep only matching elements ──────────────────────────────────────
Flux<User> activeUsers = userFlux.filter(User::isActive);

// ── distinct and distinctUntilChanged ────────────────────────────────────────
Flux<String> unique     = nameFlux.distinct();                    // Remove ALL duplicates
Flux<String> noConsec   = nameFlux.distinctUntilChanged();        // Remove consecutive duplicates only

// ── take, skip, takeLast, skipLast ───────────────────────────────────────────
Flux<Integer> firstFive = Flux.range(1, 100).take(5);     // 1, 2, 3, 4, 5
Flux<Integer> afterFive = Flux.range(1, 10).skip(5);      // 6, 7, 8, 9, 10
Flux<Integer> takeUntil = Flux.range(1, 100)
    .takeWhile(n -> n < 6);                                // 1, 2, 3, 4, 5

// ── reduce and scan ──────────────────────────────────────────────────────────
Mono<Integer> sum = Flux.range(1, 5).reduce(0, Integer::sum);   // Mono<15>

Flux<Integer> runningSum = Flux.range(1, 5).scan(0, Integer::sum);
// Emits: 0, 1, 3, 6, 10, 15  (initial value + each partial sum)

// ── collect into a container ─────────────────────────────────────────────────
Mono<List<User>>          asList  = userFlux.collectList();
Mono<Map<Long, User>>     asMap   = userFlux.collectMap(User::getId);
Mono<List<User>>          sorted  = userFlux.sort(Comparator.comparing(User::getName)).collectList();

// ── zipWith and zip — combining multiple streams ──────────────────────────────
// zip waits for one item from each stream and combines them as a pair/tuple
Flux<Tuple2<User, Order>> userOrders = Flux.zip(userFlux, orderFlux);
// Each emitted tuple contains one User and one Order — stops when either stream ends

Mono<UserWithProfile> combined = Mono.zip(
    userRepository.findById(userId),      // Executes concurrently with the next line
    profileRepository.findByUserId(userId)
).map(tuple -> new UserWithProfile(tuple.getT1(), tuple.getT2()));
// Both findById calls are issued simultaneously — total time is max(t1, t2), not t1+t2
```

### 21.5.3 Error Handling

Error handling in reactive pipelines is a first-class operation. Unlike try/catch blocks (which can't span async boundaries), Reactor provides operators for handling, transforming, and recovering from errors within the pipeline itself.

```java
// ── onErrorReturn: replace the error with a fallback value ───────────────────
Mono<User> withFallback = userRepository.findById(userId)
    .onErrorReturn(UserNotFoundException.class, User.anonymous()); // Default for 404
    // Other exceptions are NOT caught — they propagate normally

// ── onErrorResume: replace the error with a fallback Mono/Flux ───────────────
Mono<User> withFallbackMono = userRepository.findById(userId)
    .onErrorResume(NotFoundException.class, ex ->
        cacheService.getUser(userId)   // Try cache on primary source failure
            .onErrorResume(e -> Mono.just(User.anonymous()))  // Final fallback
    );

// ── onErrorContinue (Flux only): skip error-producing elements and continue ───
// Use carefully — it can hide bugs. Better for data processing pipelines
// where one bad record shouldn't stop the whole batch.
Flux<ProcessedRecord> processed = rawRecordFlux
    .map(record -> processRecord(record))
    .onErrorContinue(ParseException.class, (ex, record) ->
        log.warn("Skipped unparseable record: {}", record, ex));

// ── retry and retryWhen: retry on transient failure ──────────────────────────
Mono<Response> withRetry = httpClient.get("/api/data")
    .retryWhen(Retry.backoff(3, Duration.ofMillis(500))
        .maxBackoff(Duration.ofSeconds(10))
        .filter(ex -> ex instanceof ConnectException || ex instanceof TimeoutException)
        .onRetryExhaustedThrow((spec, signal) ->
            new ServiceUnavailableException("Failed after 3 retries", signal.failure()))
    );

// ── doOnError: side effect on error (logging) without changing the stream ─────
Mono<User> withLogging = userRepository.findById(userId)
    .doOnError(ex -> log.error("Failed to load user {}", userId, ex));

// ── Timeout: fail the stream if no item arrives within a duration ─────────────
Mono<User> withTimeout = userRepository.findById(userId)
    .timeout(Duration.ofSeconds(5))
    .onErrorMap(TimeoutException.class, ex ->
        new ServiceTimeoutException("User service timed out after 5s"));
```

### 21.5.4 Scheduling and Threading

A key aspect of reactive programming that confuses newcomers is understanding which thread executes which part of the pipeline. By default, Reactor executes on whichever thread called `subscribe()`. Schedulers let you explicitly move work to different thread pools.

```java
import reactor.core.scheduler.Schedulers;

// ── publishOn: change the thread for DOWNSTREAM operators ─────────────────────
Flux.just(1, 2, 3, 4, 5)
    .map(n -> n * 2)                        // Runs on subscribing thread
    .publishOn(Schedulers.boundedElastic()) // Switch thread for everything below
    .map(n -> expensiveOp(n))              // Now runs on boundedElastic thread
    .subscribe(System.out::println);        // Also on boundedElastic

// ── subscribeOn: change the thread for the SOURCE and upstream ────────────────
// Unlike publishOn (which affects downstream), subscribeOn affects where the
// source starts emitting. For a Mono.fromCallable(), this is where the callable runs.
Mono.fromCallable(() -> blockingDatabaseCall())  // Blocking call
    .subscribeOn(Schedulers.boundedElastic())     // Run on a thread pool designed for blocking I/O
    .map(result -> transform(result))             // Runs on same boundedElastic thread
    .subscribe(System.out::println);

// ── Built-in Schedulers ───────────────────────────────────────────────────────
// Schedulers.immediate()        — current thread (no scheduler, default)
// Schedulers.single()           — one reusable thread (good for time-based operators)
// Schedulers.boundedElastic()   — elastic thread pool capped at 10x CPU cores
//                                  DESIGNED for blocking I/O: use when calling
//                                  blocking code (JDBC, blocking HTTP) from reactive code
// Schedulers.parallel()         — fixed-size pool = CPU core count
//                                  DESIGNED for CPU-bound tasks
// Schedulers.fromExecutor(exec) — use an existing ExecutorService

// ── Bridging reactive and blocking code ──────────────────────────────────────
// The fundamental rule: NEVER block inside a reactive pipeline without using
// subscribeOn(Schedulers.boundedElastic()). Blocking freezes the scheduler thread.

// ❌ WRONG: blocking inside reactive pipeline without scheduler change
Flux<User> bad = userIdFlux
    .map(id -> userRepository.findByIdBlocking(id)); // Blocks the reactor thread!

// ✅ CORRECT: wrap the blocking call in fromCallable + subscribeOn
Flux<User> good = userIdFlux
    .flatMap(id -> Mono.fromCallable(() -> userRepository.findByIdBlocking(id))
        .subscribeOn(Schedulers.boundedElastic()));
```

### 21.5.5 Hot vs. Cold Publishers

This distinction is one of the most important conceptual points in reactive programming.

A **cold publisher** doesn't start producing items until someone subscribes, and it produces a fresh sequence for each subscriber. A `Mono.fromCallable(...)` is cold — the callable doesn't run until subscribe() is called. Every subscriber gets its own independent execution. A database query wrapped in Reactor is cold — each subscriber triggers a fresh query.

A **hot publisher** is active regardless of subscribers and shares the same sequence among all subscribers. A Kafka topic consumer, a mouse click stream, or a live price feed is hot — the events happen whether or not anyone is listening, and late subscribers miss earlier events.

```java
// Cold publisher — each subscriber gets the ENTIRE sequence from the start
Flux<Integer> cold = Flux.range(1, 5);

cold.subscribe(n -> System.out.println("Sub1: " + n));
cold.subscribe(n -> System.out.println("Sub2: " + n));
// Output:
// Sub1: 1, Sub1: 2, Sub1: 3, Sub1: 4, Sub1: 5
// Sub2: 1, Sub2: 2, Sub2: 3, Sub2: 4, Sub2: 5 (independent sequence)

// Converting cold to hot: share() makes a Flux hot via a ConnectableFlux
Flux<Long> hotTicker = Flux.interval(Duration.ofSeconds(1)).share();
// Now all subscribers see the SAME interval events, not independent ones.
// A subscriber joining after t=3 misses the first 3 ticks.

// publish().autoConnect(n): start emitting when n subscribers have connected
Flux<Long> multicast = Flux.interval(Duration.ofSeconds(1))
    .publish()
    .autoConnect(2);  // Start ticking only when 2 subscribers are ready
```

### 21.5.6 Spring WebFlux Integration

```java
// Spring WebFlux reactive controller — returns Mono/Flux instead of objects
@RestController
@RequestMapping("/api/v1/users")
@RequiredArgsConstructor
public class UserController {

    private final UserRepository userRepository;   // ReactiveMongoRepository or R2DBC

    @GetMapping("/{id}")
    public Mono<ResponseEntity<UserDto>> getUser(@PathVariable Long id) {
        return userRepository.findById(id)
            .map(user -> ResponseEntity.ok(toDto(user)))
            .defaultIfEmpty(ResponseEntity.notFound().build());
    }

    // Flux return type streams results to the client as Server-Sent Events
    @GetMapping(produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<UserDto> streamAllUsers() {
        return userRepository.findAll().map(this::toDto);
    }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public Mono<UserDto> createUser(@RequestBody @Valid Mono<CreateUserRequest> requestMono) {
        return requestMono
            .map(this::toEntity)
            .flatMap(userRepository::save)
            .map(this::toDto);
    }
}
```

---

## 🔄 RxJava — ReactiveX for Java

RxJava is the Java implementation of the ReactiveX pattern (also found in RxJS, RxSwift, etc.). It predates Project Reactor and has a very similar operator vocabulary. RxJava 3 is the current version and it is also Reactive Streams compliant, making it interoperable with Reactor.

The key types in RxJava are `Observable<T>` (0-to-N items, no backpressure), `Flowable<T>` (0-to-N items with backpressure — the equivalent of Reactor's `Flux`), `Single<T>` (exactly 1 item or error — similar to Reactor's `Mono`), `Maybe<T>` (0 or 1 item — similar to `Mono<Optional<T>>`), and `Completable` (completes with no value — similar to `Mono<Void>`).

```java
import io.reactivex.rxjava3.core.*;
import io.reactivex.rxjava3.schedulers.Schedulers;

// ── Observable (no backpressure — for UI events, small collections) ───────────
Observable<String> obs = Observable.just("Alice", "Bob", "Carol");

obs.filter(name -> name.startsWith("A"))
   .map(String::toUpperCase)
   .subscribe(
       name  -> System.out.println("Got: " + name),  // onNext
       err   -> System.err.println("Error: " + err), // onError
       ()    -> System.out.println("Done")            // onComplete
   );

// ── Flowable (with backpressure — for database queries, file reading) ──────────
Flowable<User> users = Flowable.fromIterable(userRepository.findAll())
    .observeOn(Schedulers.computation())   // Process on computation thread pool
    .filter(User::isActive)
    .map(user -> enrichUser(user));

// ── Single (exactly one result) ───────────────────────────────────────────────
Single<User> findUser = Single.fromCallable(() -> userRepository.findById(userId)
    .orElseThrow(() -> new UserNotFoundException(userId)));

findUser
    .subscribeOn(Schedulers.io())
    .observeOn(Schedulers.single())
    .subscribe(
        user -> processUser(user),
        err  -> handleError(err)
    );

// ── Maybe (zero or one result) ────────────────────────────────────────────────
Maybe<User> maybeUser = Maybe.fromOptional(userRepository.findById(userId));
maybeUser.ifEmpty(Maybe.just(User.anonymous()))
         .subscribe(user -> System.out.println(user.getName()));

// ── Completable (no result — fire and forget) ─────────────────────────────────
Completable deleteUser = Completable.fromAction(() -> userRepository.deleteById(userId));
deleteUser.subscribeOn(Schedulers.io())
          .subscribe(() -> log.info("User deleted"), err -> log.error("Delete failed", err));

// ── Combining streams ─────────────────────────────────────────────────────────
// zip: combine one element from each stream
Single<UserWithOrders> userWithOrders = Single.zip(
    findUserSingle,
    findOrdersSingle,
    (user, orders) -> new UserWithOrders(user, orders)
);

// merge: interleave emissions from multiple Observables
Observable<String> merged = Observable.merge(obs1, obs2, obs3); // Unordered
Observable<String> concatenated = Observable.concat(obs1, obs2, obs3); // obs1 fully, then obs2, then obs3
```

> **RxJava vs. Project Reactor:** If you are using Spring (WebFlux, Spring Data R2DBC, Spring Cloud Gateway), use Project Reactor — it is the native reactive library of the Spring ecosystem and `Mono`/`Flux` are directly returned and consumed by Spring components. If you are using a non-Spring framework, doing Android development, or working with a codebase that already uses RxJava, use RxJava. The operator vocabulary is 90% the same, so learning one makes the other straightforward.

---

## 🔥 Mutiny — Quarkus's Reactive Library

Mutiny (`io.smallrye.reactive:mutiny`) is the reactive library used in Quarkus and Vert.x. It was designed with simplicity in mind — Mutiny's API surface is intentionally smaller than Reactor's or RxJava's, and it uses a verb-first, guided API that makes the intent of each operator clearer to newcomers.

Mutiny has two types: `Uni<T>` (zero or one item, equivalent to `Mono<T>`) and `Multi<T>` (zero to N items, equivalent to `Flux<T>`).

```java
import io.smallrye.mutiny.*;

// ── Uni<T> — zero or one item ──────────────────────────────────────────────────
Uni<User> findUser = Uni.createFrom().item(userId)
    .onItem().transformToUni(id -> userRepository.findById(id)); // Async, returns Uni

// The verb-first API makes intent explicit:
findUser
    .onItem().transform(user -> enrichUser(user))     // Transform the item
    .onItem().invoke(user -> audit(user))              // Side effect, doesn't change the item
    .onFailure().recoverWithItem(User.anonymous())     // Recover from error
    .onFailure().retry().atMost(3)                    // Retry on failure
    .subscribe().with(
        user -> processUser(user),
        err  -> handleError(err)
    );

// ── Multi<T> — multiple items ──────────────────────────────────────────────────
Multi<User> allUsers = Multi.createFrom().iterable(userRepository.findAll());

allUsers
    .select().where(User::isActive)                    // filter
    .onItem().transform(this::toDto)                   // map
    .group().by(UserDto::getDepartment)               // groupBy
    .onItem().transformToMultiAndMerge(group ->
        group.collect().asList()
             .onItem().transform(list -> new DeptGroup(group.key(), list))
             .toMulti())
    .subscribe().with(
        group -> System.out.println(group),
        err   -> log.error("Error", err),
        ()    -> log.info("All groups processed")
    );

// ── Combining ─────────────────────────────────────────────────────────────────
Uni<UserWithProfile> combined = Uni.combine().all()
    .unis(userRepository.findById(userId), profileRepository.findByUserId(userId))
    .asTuple()
    .onItem().transform(tuple -> new UserWithProfile(tuple.getItem1(), tuple.getItem2()));

// ── Quarkus reactive REST endpoint ────────────────────────────────────────────
@Path("/api/users")
public class UserResource {

    @Inject UserRepository repository;

    @GET
    @Path("/{id}")
    public Uni<Response> getUser(@PathParam("id") Long id) {
        return repository.findById(id)
            .onItem().transform(user -> Response.ok(user).build())
            .onItem().ifNull().continueWith(Response.status(404).build());
    }

    @GET
    @Produces(MediaType.SERVER_SENT_EVENTS)
    public Multi<User> streamUsers() {
        return repository.streamAll();
    }
}
```

---

## 💡 Phase 21.5 — Key Takeaways

Reactive programming changes the way you think about asynchronous code. Rather than writing imperative sequences with callbacks or futures, you declare a pipeline of transformations and let the reactive library manage scheduling, error propagation, and backpressure. The Reactive Streams specification gives all libraries a common contract, which means they can interoperate.

Project Reactor is the most important reactive library for Java developers today because it is the foundation of Spring WebFlux — every modern Spring application that uses non-blocking I/O, WebFlux controllers, or R2DBC uses Reactor's `Mono` and `Flux`. The key concepts to master are the difference between `map` and `flatMap` (synchronous vs. async transformation), how to handle errors within the pipeline, how `subscribeOn` and `publishOn` control threading, and when you must move to `boundedElastic` when bridging blocking code. RxJava is equally powerful and valuable outside the Spring ecosystem. Mutiny brings a cleaner, more guided API to the Quarkus world. All three implement the same specification — mastering one makes the others straightforward to pick up.

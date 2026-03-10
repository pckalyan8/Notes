# 📗 11.8 — Spring WebFlux: Reactive Programming, Mono, Flux & WebClient

Spring WebFlux is the reactive alternative to Spring MVC. Before diving into the API, it is essential to understand *why* it exists — because reactive programming solves a specific scalability problem that the traditional thread-per-request model cannot address efficiently.

---

## 🧠 The Problem: Thread-Per-Request and Blocking I/O

In Spring MVC, every incoming HTTP request occupies a thread from Tomcat's thread pool for the entire duration of its processing. Most of that time, the thread is waiting — waiting for a database query to return, waiting for an external API to respond, waiting for a file to be read. The thread is blocked: it cannot do any other work while it waits.

A typical Tomcat setup has 200 threads. Under heavy load with slow I/O, 200 threads are each blocked waiting, and incoming request 201 queues up even though your CPU is sitting at 5% utilization. You are not CPU-bound — you are I/O-bound. Adding more threads (vertical scaling) only pushes the problem further: OS context-switching overhead increases, and memory consumption grows (each thread uses ~1MB of stack space).

**Reactive programming** addresses this with a different mental model: instead of blocking a thread while waiting for I/O, you say "when the data is ready, call me back." A small number of threads can serve a very large number of concurrent requests because they never block — they are always doing productive work. This is the same principle behind Node.js's event loop, and Spring WebFlux brings it to the Java ecosystem.

```
Traditional (Blocking):           Reactive (Non-Blocking):
Thread → DB query → WAIT          Thread → DB query → continue other work
Thread → WAIT                     Thread → API call → continue other work
Thread → WAIT                     ... data arrives later → callback fires
Thread → WAITING (wasted)         Same 4 threads handle 10,000 requests
```

When to use WebFlux: high-concurrency, I/O-bound workloads where many requests are in flight simultaneously. When NOT to use it: CPU-intensive computation, blocking legacy APIs, or teams unfamiliar with reactive patterns — the complexity cost is real.

---

## 🌊 Project Reactor — The Foundation

Spring WebFlux is built on **Project Reactor**, which provides the core reactive types you work with. These two types represent the entire Reactor API surface that you need to master:

`Mono<T>` represents **zero or one item** that will be available asynchronously in the future — analogous to a `CompletableFuture<Optional<T>>`. Use it for operations that return one result: finding by ID, saving an entity, or calling an external API.

`Flux<T>` represents **zero or more items** that arrive asynchronously over time — analogous to a reactive `Stream<T>`. Use it for operations that return collections: finding all students, reading a file line-by-line, or subscribing to a Kafka topic.

Neither `Mono` nor `Flux` does any work when you create them. They are **lazy** — nothing happens until something subscribes to them. This is one of the most important and initially confusing aspects of reactive programming: you are building a **pipeline** (called an assembly), and the pipeline executes only when subscribed.

```java
// This creates the pipeline — NO database call yet
Mono<Student> studentMono = studentRepository.findById(1L);

// This also creates a pipeline — NO work done yet
Flux<Student> studentFlux = studentRepository.findAll();

// The pipeline executes only when someone subscribes
studentMono.subscribe(student -> System.out.println(student)); // Now it executes

// Or when the HTTP server subscribes (in WebFlux, the framework subscribes for you)
```

---

## 📦 Maven Setup

```xml
<!-- WebFlux replaces spring-boot-starter-web (Tomcat) with Netty as the server -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-webflux</artifactId>
</dependency>

<!-- Reactive MongoDB (non-blocking) -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-mongodb-reactive</artifactId>
</dependency>

<!-- R2DBC for reactive SQL databases (non-blocking JDBC alternative) -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-r2dbc</artifactId>
</dependency>
<dependency>
    <groupId>io.r2dbc</groupId>
    <artifactId>r2dbc-postgresql</artifactId>
    <scope>runtime</scope>
</dependency>
```

---

## 🏗️ Creating Mono and Flux

```java
public class ReactorCreationExamples {

    // ── Creating Mono ──

    // From a value you already have
    Mono<String> greeting = Mono.just("Hello, World!");

    // From a value that may be null (use instead of Mono.just(nullableValue))
    Mono<String> nullable = Mono.justOrEmpty(possiblyNullValue);

    // An empty Mono — represents "no value" (like an empty Optional)
    Mono<String> empty = Mono.empty();

    // From a future computation (deferred: computed when subscribed, not now)
    Mono<Student> fromSupplier = Mono.fromSupplier(() -> findStudentFromDatabase());

    // From a CompletableFuture (bridge to non-reactive async code)
    Mono<String> fromFuture = Mono.fromFuture(someCompletableFuture);

    // That immediately throws an error
    Mono<Student> error = Mono.error(new RuntimeException("Student not found"));

    // ── Creating Flux ──

    // From known values
    Flux<Integer> numbers = Flux.just(1, 2, 3, 4, 5);

    // From any Iterable (List, Set, etc.)
    Flux<Student> fromList = Flux.fromIterable(List.of(alice, bob, carol));

    // From an array
    Flux<String> fromArray = Flux.fromArray(new String[]{"a", "b", "c"});

    // Infinite counter (0, 1, 2, ...) — always use limit/take to stop it
    Flux<Long> counter = Flux.range(0, Integer.MAX_VALUE).map(Long::valueOf);

    // Emit one item per specified interval — useful for polling or streaming updates
    Flux<Long> ticker = Flux.interval(Duration.ofSeconds(1));

    // Generate dynamically — custom push-based emission
    Flux<Integer> generated = Flux.generate(
        () -> 0,           // Initial state
        (state, sink) -> {
            sink.next(state * state); // Emit state²
            if (state >= 5) sink.complete();
            return state + 1;
        }
    );
}
```

---

## 🔗 Operators — Building Pipelines

Operators transform, filter, and combine Mono and Flux values without blocking. They are the functional equivalent of Java Stream's `map`, `filter`, `flatMap` etc., but they work asynchronously.

```java
public class OperatorExamples {

    // ── Transformation operators ──

    // map: transform each item synchronously (1-to-1)
    Flux<String> names = studentFlux.map(student -> student.getName().toUpperCase());

    // flatMap: transform each item to another async publisher; results may interleave
    // (most important operator — used whenever one async call depends on another)
    Flux<EnrollmentDTO> enrollments = studentFlux
        .flatMap(student -> enrollmentService.findByStudent(student)); // Returns Flux per student
    // Result: merges all inner Fluxes — items from different students may interleave

    // concatMap: same as flatMap but preserves order (no interleaving); less concurrent
    Flux<EnrollmentDTO> orderedEnrollments = studentFlux
        .concatMap(student -> enrollmentService.findByStudent(student));

    // flatMapSequential: flatMap that reorders results to maintain sequence
    Flux<EnrollmentDTO> sequentialEnrollments = studentFlux
        .flatMapSequential(student -> enrollmentService.findByStudent(student));

    // switchMap: cancels the previous inner publisher when a new item arrives
    // Classic use case: autocomplete — cancel the previous search when the user types more
    Flux<List<Course>> searchResults = keyPressFlux
        .switchMap(query -> courseService.search(query));

    // ── Filtering operators ──
    Flux<Student> highScorers = studentFlux
        .filter(s -> s.getScore() > 85.0)
        .take(10)           // Take only the first 10 matching students
        .skip(0)            // Skip the first N items
        .distinct()         // Remove duplicates (uses equals/hashCode)
        .takeWhile(s -> s.getScore() > 80.0)  // Take while condition is true
        .skipWhile(s -> s.getScore() < 50.0); // Skip while condition is true

    // ── Aggregation operators ──
    Mono<Long>   count   = studentFlux.count();
    Mono<Double> max     = studentFlux.map(Student::getScore).reduce(Double::max);
    Mono<List<Student>> collected = studentFlux.collectList(); // Collect all into a Mono<List>

    // ── Combination operators ──

    // zip: combine two publishers item-by-item; emits when BOTH have an item
    // Stops when the shorter source is exhausted
    Flux<String> zipped = Flux.zip(
        nameFlux,
        scoreFlux,
        (name, score) -> name + ": " + score
    );

    // merge: combine multiple Fluxes; items arrive as they come (no ordering guarantee)
    Flux<Event> allEvents = Flux.merge(loginEvents, purchaseEvents, logoutEvents);

    // concat: append one Flux after another (sequential, not parallel)
    Flux<Student> allStudents = Flux.concat(enrolledStudents, graduatedStudents);

    // combineLatest: whenever any source emits, combine it with the latest from all other sources
    // Useful for combining user input with filter state
    Flux<SearchResult> liveSearch = Flux.combineLatest(
        nameFilterFlux, categoryFilterFlux, priceRangeFlux,
        (name, category, price) -> searchService.search(name, category, price)
    ).flatMap(resultsMono -> resultsMono);

    // ── Error handling operators ──

    // onErrorReturn: replace an error with a fallback value
    Mono<Student> withFallback = studentService.findById(id)
        .onErrorReturn(new Student("default", "fallback@example.com"));

    // onErrorResume: replace an error with a fallback publisher
    Mono<Student> withFallbackPublisher = studentService.findById(id)
        .onErrorResume(error -> cacheService.findById(id)); // Try cache if DB fails

    // retry: retry a failed publisher N times
    Flux<Price> withRetry = priceService.fetchPrices()
        .retry(3)                // Retry up to 3 times immediately
        .retryWhen(Retry.fixedDelay(3, Duration.ofSeconds(2))); // Retry with delay

    // doOnError: side effect on error (logging) without handling it
    Mono<Student> withLogging = studentService.findById(id)
        .doOnError(error -> log.error("Failed to find student {}: {}", id, error.getMessage()));

    // ── Side effect operators (do not change the stream) ──
    Flux<Student> withSideEffects = studentFlux
        .doOnNext(student -> log.debug("Processing: {}", student.getName()))   // For each item
        .doOnComplete(() -> log.info("All students processed"))                  // When done
        .doOnError(ex -> log.error("Error in pipeline: {}", ex.getMessage()))   // On error
        .doFirst(() -> log.info("Subscription started"))                         // On subscribe
        .doFinally(signal -> log.info("Pipeline terminated with: {}", signal));  // Always
}
```

---

## 🎮 WebFlux Controllers — Annotated Style

The annotated controller style in WebFlux looks almost identical to Spring MVC — but methods return `Mono<T>` or `Flux<T>` instead of plain objects.

```java
@RestController
@RequestMapping("/api/students")
public class StudentReactiveController {

    private final ReactiveStudentService studentService;

    public StudentReactiveController(ReactiveStudentService studentService) {
        this.studentService = studentService;
    }

    // Returns all students — Flux<StudentDTO> streams items as they are produced
    @GetMapping(produces = MediaType.APPLICATION_JSON_VALUE)
    public Flux<StudentDTO> findAll() {
        return studentService.findAll();
    }

    // Returns one student — Mono<StudentDTO> with automatic 404 on empty
    @GetMapping("/{id}")
    public Mono<ResponseEntity<StudentDTO>> findById(@PathVariable Long id) {
        return studentService.findById(id)
            .map(ResponseEntity::ok)                        // Found: wrap in 200 OK
            .defaultIfEmpty(ResponseEntity.notFound().build()); // Not found: 404
    }

    // Create a student — Mono<CreateStudentRequest> for the request body
    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public Mono<StudentDTO> create(@RequestBody @Valid Mono<CreateStudentRequest> request) {
        // Mono<CreateStudentRequest> is lazily bound — Spring deserializes when subscribed
        return request.flatMap(studentService::create);
    }

    // Update a student
    @PutMapping("/{id}")
    public Mono<ResponseEntity<StudentDTO>> update(@PathVariable Long id,
                                                    @RequestBody @Valid Mono<UpdateStudentRequest> request) {
        return request
            .flatMap(req -> studentService.update(id, req))
            .map(ResponseEntity::ok)
            .defaultIfEmpty(ResponseEntity.notFound().build());
    }

    // Delete a student
    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public Mono<Void> delete(@PathVariable Long id) {
        return studentService.delete(id);
    }

    // Server-Sent Events (SSE) — stream data to the client continuously
    // The client keeps the connection open and receives events as they arrive
    @GetMapping(value = "/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<StudentDTO> streamStudents() {
        return studentService.findAll()
            .delayElements(Duration.ofMillis(100)); // Simulate a stream arriving over time
    }

    // Streaming real-time updates using SSE
    @GetMapping(value = "/live-scores", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<ServerSentEvent<StudentScoreUpdate>> streamScoreUpdates() {
        return scoreUpdateService.getScoreUpdateStream()
            .map(update -> ServerSentEvent.<StudentScoreUpdate>builder()
                .event("score-update")
                .data(update)
                .build());
    }
}
```

---

## 🔄 Functional Routing Style

WebFlux also offers an alternative to annotations: functional routing. You define routes as beans using `RouterFunction` and `HandlerFunction`. This style is more explicit and composable, and avoids reflection-based proxy magic.

```java
// The handler — pure functions that process requests and return responses
@Component
public class StudentHandler {

    private final ReactiveStudentService studentService;

    public StudentHandler(ReactiveStudentService studentService) {
        this.studentService = studentService;
    }

    public Mono<ServerResponse> findAll(ServerRequest request) {
        return ServerResponse.ok()
            .contentType(MediaType.APPLICATION_JSON)
            .body(studentService.findAll(), StudentDTO.class);
    }

    public Mono<ServerResponse> findById(ServerRequest request) {
        Long id = Long.parseLong(request.pathVariable("id"));
        return studentService.findById(id)
            .flatMap(student -> ServerResponse.ok().bodyValue(student))
            .switchIfEmpty(ServerResponse.notFound().build()); // 404 if Mono is empty
    }

    public Mono<ServerResponse> create(ServerRequest request) {
        return request.bodyToMono(CreateStudentRequest.class) // Deserialize request body
            .flatMap(studentService::create)
            .flatMap(created -> ServerResponse.created(
                URI.create("/api/students/" + created.id())
            ).bodyValue(created));
    }
}

// The router — maps URL patterns to handler functions
@Configuration
public class StudentRouter {

    @Bean
    public RouterFunction<ServerResponse> studentRoutes(StudentHandler handler) {
        return RouterFunctions.route()
            .GET("/api/students",      handler::findAll)
            .GET("/api/students/{id}", handler::findById)
            .POST("/api/students",     handler::create)
            .build();
    }
}
```

---

## 📡 WebClient — Non-Blocking HTTP Client

`WebClient` is the reactive replacement for `RestTemplate`. It makes non-blocking HTTP calls — while waiting for the response, the thread is free to do other work.

```java
@Service
public class ExternalApiService {

    private final WebClient webClient;

    public ExternalApiService(WebClient.Builder webClientBuilder) {
        this.webClient = webClientBuilder
            .baseUrl("https://api.external-grades.example.com")
            .defaultHeader(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE)
            .defaultHeader(HttpHeaders.ACCEPT, MediaType.APPLICATION_JSON_VALUE)
            .filter(ExchangeFilterFunctions.basicAuthentication("user", "secret"))
            .build();
    }

    // Simple GET request
    public Mono<ExternalGrade> fetchGrade(Long studentId) {
        return webClient.get()
            .uri("/grades/{studentId}", studentId) // URI template
            .retrieve()                             // Initiate the request
            .bodyToMono(ExternalGrade.class)        // Deserialize response body
            .timeout(Duration.ofSeconds(5))         // Fail if no response in 5s
            .retryWhen(Retry.backoff(3, Duration.ofMillis(500))); // 3 retries with backoff
    }

    // POST request with a body
    public Mono<GradeReport> submitGrades(GradeSubmission submission) {
        return webClient.post()
            .uri("/grades/submit")
            .bodyValue(submission)                 // Serialize submission to JSON
            .retrieve()
            .onStatus(HttpStatusCode::is4xxClientError, response ->
                response.bodyToMono(String.class)
                    .flatMap(body -> Mono.error(new BadRequestException(body)))
            )
            .onStatus(HttpStatusCode::is5xxServerError, response ->
                Mono.error(new ExternalServiceException("External API server error"))
            )
            .bodyToMono(GradeReport.class);
    }

    // Fetch multiple resources in parallel — reactive version of parallel processing
    public Flux<ExternalGrade> fetchGradesForAll(List<Long> studentIds) {
        return Flux.fromIterable(studentIds)
            .flatMap(id -> fetchGrade(id)   // flatMap starts all calls concurrently
                .onErrorResume(ex -> {
                    log.warn("Failed to fetch grade for student {}: {}", id, ex.getMessage());
                    return Mono.empty(); // Skip failures instead of failing the whole stream
                }),
                10 // Concurrency limit: max 10 parallel requests at a time
            );
    }

    // Stream a large response body
    public Flux<CourseUpdate> streamCourseUpdates() {
        return webClient.get()
            .uri("/courses/updates/stream")
            .accept(MediaType.TEXT_EVENT_STREAM)
            .retrieve()
            .bodyToFlux(CourseUpdate.class); // Each SSE event becomes one Flux item
    }
}
```

---

## ⏱️ Backpressure — Managing Fast Producers

Backpressure is a mechanism for a slow consumer to signal to a fast producer to slow down or stop. Without backpressure, a fast data source can overflow a slow consumer's buffer and cause memory issues or data loss.

Reactor's publishers are backpressure-aware by default. When you use operators like `limitRate()` or `onBackpressureBuffer()`, you are controlling how backpressure is handled:

```java
Flux.range(1, 1_000_000)        // Producer: generates 1 million numbers fast
    .onBackpressureBuffer(1000)  // Buffer up to 1000 items if consumer is slow; drop on overflow
    .publishOn(Schedulers.boundedElastic()) // Move to a different thread for processing
    .map(i -> heavyComputation(i))  // Consumer: slow processing
    .subscribe();

// Other backpressure strategies:
Flux<Integer> withDrop     = sourceFlux.onBackpressureDrop();    // Drop items consumer can't keep up with
Flux<Integer> withLatest   = sourceFlux.onBackpressureLatest();  // Keep only the most recent
Flux<Integer> rateLimited  = sourceFlux.limitRate(100);          // Request upstream in batches of 100
```

---

## 🧵 Schedulers — Controlling Which Thread Does What

In reactive pipelines, you control thread execution with schedulers. By default, work runs on the thread that subscribed. For I/O or CPU-intensive work, you redirect to the appropriate scheduler.

```java
Flux<Student> pipeline = studentRepository.findAll()    // Runs on DB driver thread
    .publishOn(Schedulers.parallel())                   // Switch to parallel threads for CPU work
    .map(student -> heavyCpuProcessing(student))
    .publishOn(Schedulers.boundedElastic())              // Switch to boundedElastic for blocking I/O
    .flatMap(student -> Mono.fromCallable(() -> blockingFileWrite(student))); // Wrap blocking call

// Scheduler types:
// Schedulers.immediate()        → Current thread (no switch)
// Schedulers.single()           → Single reusable thread
// Schedulers.parallel()         → CPU cores × 2 threads; for CPU work
// Schedulers.boundedElastic()   → Elastic thread pool for blocking I/O; bounded to prevent OOM
// Schedulers.fromExecutor(exec) → Wrap your own ExecutorService

// CRITICAL: If you must call a blocking API (e.g., legacy JDBC) from a reactive pipeline:
Mono<Student> withBlockingCall = Mono.fromCallable(() -> blockingRepository.findById(id))
    .subscribeOn(Schedulers.boundedElastic()); // Never block a Netty or parallel scheduler thread!
```

---

## ✅ Best Practices & Important Points

**Never block in a reactive pipeline.** Calling `mono.block()`, `flux.blockFirst()`, or any blocking JDBC/legacy API directly in a reactive pipeline defeats the entire purpose and can deadlock Netty's event loop threads. Wrap blocking calls in `Mono.fromCallable()` and offload them to `Schedulers.boundedElastic()`.

**Use `flatMap` for dependent async calls, not nested `subscribe`.** Nested `subscribe` calls are the reactive anti-pattern equivalent of callback hell. When the result of one async call is needed to start another, chain them with `flatMap`.

**Don't use WebFlux for simple CRUD applications with blocking databases.** If you use Spring WebFlux but connect to a relational database through blocking JDBC (not R2DBC), you've gained all the complexity of reactive programming with none of the benefits. The entire chain must be non-blocking to benefit. Choose WebFlux only when your stack is fully reactive or when you have a compelling high-concurrency use case.

**Start with `Mono` and `Flux` patterns before exploring advanced operators.** The core operators `map`, `flatMap`, `filter`, `zip`, `merge`, and the error-handling operators cover the vast majority of real-world use cases. Master these before exploring edge cases like `switchMap`, `expand`, or custom operators.

---

## 🔑 Key Summary

```
Mono<T>           → 0 or 1 asynchronous item; like CompletableFuture<Optional<T>>
Flux<T>           → 0 or N asynchronous items; like a reactive Stream<T>
Lazy execution    → Nothing runs until subscribed; you are building a pipeline, not executing one
map               → Synchronous 1-to-1 transformation within the pipeline
flatMap           → Async transformation; each item becomes a new publisher; results may interleave
filter            → Keep only matching items
zip               → Combine two publishers item-by-item
onErrorReturn     → Fallback value when error occurs
onErrorResume     → Fallback publisher when error occurs
retry/retryWhen   → Re-subscribe on failure
publishOn         → Switch execution to a different scheduler for downstream operations
subscribeOn       → Determine which scheduler the source emits on
Schedulers.boundedElastic() → Use for wrapping blocking I/O calls
WebClient         → Non-blocking HTTP client; reactive replacement for RestTemplate
RouterFunction    → Functional routing alternative to @RequestMapping annotations
SSE               → TEXT_EVENT_STREAM_VALUE media type for streaming Flux to browser/client
Backpressure      → Consumer signals producer to slow down; managed via onBackpressureBuffer/Drop
```

# 21.3 — HTTP Clients

> **Libraries covered:** Java 11 HttpClient · OkHttp · Apache HttpClient 5 · Feign · RestTemplate · WebClient
>
> Making HTTP calls — to REST APIs, third-party services, microservice endpoints — is one of the most common tasks in Java backend development. Each client library in this section solves the same fundamental problem but with different design philosophies: some are imperative and low-level (Apache HttpClient), some are high-level and declarative (Feign), some are reactive and non-blocking (WebClient). Understanding when and why to choose each one is as important as knowing the API itself.

---

## ☕ Java 11 Built-in HttpClient

The Java 11 `HttpClient` (in `java.net.http`) was covered in Phase 20.1 as a language feature. We revisit it here in the context of comparing it to the ecosystem alternatives.

The built-in client is the right default choice for new projects that don't already have a dependency on a specific HTTP client library. It supports HTTP/1.1 and HTTP/2, has a clean builder API, supports both synchronous and asynchronous (CompletableFuture-based) requests, and requires zero additional dependencies.

```java
import java.net.http.*;
import java.net.URI;
import java.time.Duration;

// Create once, share everywhere — the client maintains a connection pool
HttpClient client = HttpClient.newBuilder()
    .version(HttpClient.Version.HTTP_2)
    .connectTimeout(Duration.ofSeconds(10))
    .followRedirects(HttpClient.Redirect.NORMAL)
    .build();

// GET with headers
HttpRequest request = HttpRequest.newBuilder()
    .uri(URI.create("https://api.github.com/users/octocat"))
    .header("Accept", "application/vnd.github+json")
    .header("User-Agent", "MyApp/1.0")
    .timeout(Duration.ofSeconds(30))
    .GET()
    .build();

// Synchronous — blocks the calling thread
HttpResponse<String> response = client.send(request, HttpResponse.BodyHandlers.ofString());

// Asynchronous — returns immediately, processing happens in ForkJoinPool.commonPool()
CompletableFuture<String> body = client
    .sendAsync(request, HttpResponse.BodyHandlers.ofString())
    .thenApply(HttpResponse::body);
```

The main limitation of the Java 11 client is that it has no built-in support for retries, circuit breakers, or interceptors. For production-grade service-to-service communication, you'll typically use Feign or WebClient instead.

---

## 🔷 OkHttp — Square's HTTP Client

OkHttp is an open-source HTTP client from Square (the company behind Retrofit and many Android libraries). It is widely used in both Android and server-side Java. Its strengths are transparent connection pooling and keep-alive, GZIP compression, a powerful interceptor mechanism, and excellent handling of connection recovery.

### 21.3.1 OkHttp — Core API

```java
import okhttp3.*;
import java.io.IOException;

// OkHttpClient should be a singleton — it manages thread pools and connection pools
OkHttpClient client = new OkHttpClient.Builder()
    .connectTimeout(10, TimeUnit.SECONDS)
    .readTimeout(30, TimeUnit.SECONDS)
    .writeTimeout(30, TimeUnit.SECONDS)
    .retryOnConnectionFailure(true)  // Retry on transient network failures
    .build();

// ── GET Request ────────────────────────────────────────────────────────────────
Request getRequest = new Request.Builder()
    .url("https://api.example.com/users/42")
    .header("Authorization", "Bearer " + token)
    .header("Accept", "application/json")
    .build();

// Synchronous call
try (Response response = client.newCall(getRequest).execute()) {
    if (!response.isSuccessful()) throw new IOException("HTTP " + response.code());
    String body = response.body().string();  // Read once — body can only be consumed once
    System.out.println(body);
}

// Asynchronous call — callback is invoked on OkHttp's internal thread pool
client.newCall(getRequest).enqueue(new Callback() {
    @Override
    public void onFailure(Call call, IOException e) {
        log.error("Request failed", e);
    }

    @Override
    public void onResponse(Call call, Response response) throws IOException {
        try (response) {
            if (response.isSuccessful()) {
                processResponse(response.body().string());
            }
        }
    }
});

// ── POST with JSON body ────────────────────────────────────────────────────────
MediaType JSON = MediaType.get("application/json; charset=utf-8");

String requestJson = """
    { "name": "Alice", "email": "alice@example.com" }
    """;

Request postRequest = new Request.Builder()
    .url("https://api.example.com/users")
    .post(RequestBody.create(requestJson, JSON))
    .build();

// ── POST with form parameters ─────────────────────────────────────────────────
RequestBody formBody = new FormBody.Builder()
    .add("username", "alice")
    .add("password", "secret")
    .build();

Request loginRequest = new Request.Builder()
    .url("https://api.example.com/auth/login")
    .post(formBody)
    .build();

// ── Multipart file upload ─────────────────────────────────────────────────────
RequestBody multipart = new MultipartBody.Builder()
    .setType(MultipartBody.FORM)
    .addFormDataPart("description", "My document")
    .addFormDataPart("file", "report.pdf",
        RequestBody.create(new File("report.pdf"), MediaType.get("application/pdf")))
    .build();
```

### 21.3.2 OkHttp — Interceptors

Interceptors are OkHttp's most powerful feature. They let you attach cross-cutting behaviour — logging, authentication header injection, retry logic, metrics — to every request without modifying each call site. They work like a middleware chain: each interceptor calls `chain.proceed(request)` to pass the request along and can modify both the outgoing request and the incoming response.

```java
// Application interceptor: runs once per call, sees final request/response
// Good for logging, auth header injection, rate limiting
class AuthInterceptor implements Interceptor {
    private final TokenProvider tokenProvider;

    @Override
    public Response intercept(Chain chain) throws IOException {
        Request original = chain.request();

        // Inject authentication header on every request
        Request authenticated = original.newBuilder()
            .header("Authorization", "Bearer " + tokenProvider.getCurrentToken())
            .build();

        Response response = chain.proceed(authenticated);

        // Check if token expired (401) and retry with a refreshed token
        if (response.code() == 401) {
            response.close();
            tokenProvider.refreshToken();
            Request retried = original.newBuilder()
                .header("Authorization", "Bearer " + tokenProvider.getCurrentToken())
                .build();
            return chain.proceed(retried);
        }

        return response;
    }
}

// Logging interceptor — logs request/response details at DEBUG level
class LoggingInterceptor implements Interceptor {
    @Override
    public Response intercept(Chain chain) throws IOException {
        Request request = chain.request();
        long startTime = System.nanoTime();

        log.debug("→ {} {}", request.method(), request.url());

        Response response = chain.proceed(request);
        long elapsed = (System.nanoTime() - startTime) / 1_000_000;

        log.debug("← {} {} ({}ms)", response.code(), request.url(), elapsed);
        return response;
    }
}

// Network interceptor: runs on each network call, including redirects
// Good for monitoring actual bytes sent/received
OkHttpClient clientWithInterceptors = new OkHttpClient.Builder()
    .addInterceptor(new AuthInterceptor(tokenProvider))  // Application interceptor
    .addInterceptor(new LoggingInterceptor())
    .addNetworkInterceptor(new MetricsInterceptor())     // Network interceptor
    .build();
```

---

## 🔶 Apache HttpClient 5

Apache HttpClient 5 is the mature, enterprise-grade HTTP client library with over 20 years of history. It provides the most fine-grained control of any HTTP client in the Java ecosystem — custom SSL/TLS configuration, proxy authentication, cookie management, connection pool tuning, and request retries. It is the right choice when you need deep control over connection lifecycle and advanced HTTP features.

### 21.3.3 Apache HttpClient 5 — Core Usage

```java
import org.apache.hc.client5.http.classic.methods.*;
import org.apache.hc.client5.http.impl.classic.*;
import org.apache.hc.client5.http.impl.io.*;
import org.apache.hc.core5.http.*;
import org.apache.hc.core5.http.io.entity.*;

// ── Creating a CloseableHttpClient ────────────────────────────────────────────
// CloseableHttpClient is thread-safe — create once and share
PoolingHttpClientConnectionManager connectionManager =
    new PoolingHttpClientConnectionManager();
connectionManager.setMaxTotal(200);          // Max total connections
connectionManager.setDefaultMaxPerRoute(20); // Max connections to any single host

CloseableHttpClient httpClient = HttpClients.custom()
    .setConnectionManager(connectionManager)
    .setDefaultRequestConfig(RequestConfig.custom()
        .setConnectTimeout(Timeout.ofSeconds(10))
        .setResponseTimeout(Timeout.ofSeconds(30))
        .setConnectionRequestTimeout(Timeout.ofSeconds(5)) // Timeout waiting for connection from pool
        .build())
    .setRetryStrategy(new DefaultHttpRequestRetryStrategy(3, TimeInterval.ofSeconds(1)))
    .build();

// ── GET Request ────────────────────────────────────────────────────────────────
HttpGet getRequest = new HttpGet("https://api.example.com/users/42");
getRequest.addHeader("Accept", "application/json");
getRequest.addHeader("Authorization", "Bearer " + token);

String responseBody = httpClient.execute(getRequest, response -> {
    int status = response.getCode();
    if (status < 200 || status >= 300) {
        throw new HttpException("Unexpected status: " + status);
    }
    return EntityUtils.toString(response.getEntity(), StandardCharsets.UTF_8);
});

// ── POST Request with JSON body ───────────────────────────────────────────────
HttpPost postRequest = new HttpPost("https://api.example.com/users");
postRequest.setHeader("Content-Type", "application/json");
postRequest.setEntity(new StringEntity(
    """{ "name": "Alice", "email": "alice@example.com" }""",
    ContentType.APPLICATION_JSON
));

try (CloseableHttpResponse response = httpClient.execute(postRequest)) {
    System.out.println("Status: " + response.getCode());
    System.out.println("Body:   " + EntityUtils.toString(response.getEntity()));
}

// ── Custom SSL/TLS configuration — a key strength of Apache HttpClient ─────────
// Accept self-signed certificates (for internal services / testing)
SSLContext sslContext = SSLContexts.custom()
    .loadTrustMaterial(null, new TrustAllStrategy())  // Trust all certs — dev only!
    .build();

// Or load a specific truststore for mutual TLS (mTLS)
SSLContext mtlsContext = SSLContexts.custom()
    .loadKeyMaterial(keyStore, keyPassword)         // Client certificate
    .loadTrustMaterial(trustStore, null)            // Server certificate trust
    .build();

CloseableHttpClient secureClient = HttpClients.custom()
    .setSSLContext(mtlsContext)
    .setSSLHostnameVerifier(NoopHostnameVerifier.INSTANCE) // Disable hostname verification (testing only)
    .build();
```

---

## 🎯 Feign — Declarative HTTP Client

Feign (part of OpenFeign) is a game-changer for service-to-service communication in microservices. Instead of writing HTTP client code, you **declare** the HTTP interface as a Java interface with annotations, and Feign generates the implementation. The result is that calling a remote service feels exactly like calling a local method.

Feign integrates naturally with Spring Cloud, supports load balancing via Spring Cloud LoadBalancer, and can be combined with Resilience4j for circuit breaking and retries. It is the standard approach for service-to-service HTTP in Spring Boot microservices.

### 21.3.4 Feign — Spring Cloud OpenFeign Integration

```java
// pom.xml dependency: spring-cloud-starter-openfeign

// ── Step 1: Enable Feign clients in your Spring Boot application ──────────────
@SpringBootApplication
@EnableFeignClients   // Scan for @FeignClient interfaces
public class OrderServiceApplication { ... }

// ── Step 2: Declare the remote API as an interface ────────────────────────────
@FeignClient(
    name = "user-service",             // Must match the registered service name in discovery
    url = "${services.user-service.url}", // Or hardcoded URL for non-discovery setups
    configuration = UserServiceFeignConfig.class,
    fallback = UserServiceFallback.class  // Fallback implementation (requires Resilience4j)
)
public interface UserServiceClient {

    @GetMapping("/api/v1/users/{id}")
    UserDto getUserById(@PathVariable("id") Long id);

    @GetMapping("/api/v1/users")
    Page<UserDto> searchUsers(
        @RequestParam("name")   String name,
        @RequestParam("page")   int page,
        @RequestParam("size")   int size
    );

    @PostMapping("/api/v1/users")
    @ResponseStatus(HttpStatus.CREATED)
    UserDto createUser(@RequestBody CreateUserRequest request);

    @PutMapping("/api/v1/users/{id}")
    UserDto updateUser(@PathVariable("id") Long id, @RequestBody UpdateUserRequest request);

    @DeleteMapping("/api/v1/users/{id}")
    void deleteUser(@PathVariable("id") Long id);
}

// ── Step 3: Configure timeout, logging, and error handling ────────────────────
@Configuration
public class UserServiceFeignConfig {

    @Bean
    public Request.Options requestOptions() {
        return new Request.Options(
            5, TimeUnit.SECONDS,   // Connect timeout
            30, TimeUnit.SECONDS,  // Read timeout
            true                   // Follow redirects
        );
    }

    @Bean
    public Logger.Level feignLoggerLevel() {
        return Logger.Level.FULL;  // Log headers, body, and metadata (for debugging)
        // NONE = no logging, BASIC = method+URL+status, HEADERS = + request/response headers
    }

    @Bean
    public ErrorDecoder errorDecoder() {
        return (methodKey, response) -> {
            // Map HTTP error codes to specific Java exceptions
            return switch (response.status()) {
                case 400 -> new BadRequestException("Invalid request to " + methodKey);
                case 401 -> new UnauthorizedException("Not authorised for " + methodKey);
                case 404 -> new ResourceNotFoundException("Resource not found via " + methodKey);
                case 429 -> new RateLimitException("Rate limited by " + methodKey);
                default  -> new ServiceException("Service error " + response.status() + " from " + methodKey);
            };
        };
    }
}

// ── Step 4: Fallback — what to return when the remote service fails ───────────
@Component
public class UserServiceFallback implements UserServiceClient {

    @Override
    public UserDto getUserById(Long id) {
        // Return a safe default — perhaps a cached value or an "anonymous" user
        return UserDto.builder().id(id).name("Unknown User").build();
    }

    @Override
    public Page<UserDto> searchUsers(String name, int page, int size) {
        return Page.empty();  // Return empty page rather than throwing
    }

    @Override
    public UserDto createUser(CreateUserRequest request) {
        throw new ServiceUnavailableException("User service is currently unavailable");
    }

    // ...other methods
}

// ── Step 5: Use it — exactly like a local service ─────────────────────────────
@Service
@RequiredArgsConstructor
public class OrderService {
    private final UserServiceClient userServiceClient;  // Injected by Spring

    public OrderSummaryDto createOrderSummary(Long userId, Long orderId) {
        // This looks like a local method call but makes an HTTP request to user-service
        UserDto user = userServiceClient.getUserById(userId);
        // ...
    }
}
```

> **Why Feign is preferred over RestTemplate for new code:** Feign client interfaces are self-documenting — the method signature tells you exactly what endpoint is called, what parameters it takes, and what it returns. RestTemplate calls are scattered imperative code that requires reading the full implementation to understand what it does. Feign also integrates naturally with Spring Cloud's load balancing, service discovery, and circuit breakers.

---

## 🏛️ RestTemplate — Spring's Legacy HTTP Client

RestTemplate has been the standard HTTP client in the Spring ecosystem for over a decade. It is still widely used in existing codebases and is important to understand. However, Spring has officially marked it as being in maintenance mode since Spring 5 and recommends WebClient for new code.

### 21.3.5 RestTemplate — Core API

```java
import org.springframework.web.client.RestTemplate;
import org.springframework.http.*;

@Configuration
public class RestTemplateConfig {

    @Bean
    public RestTemplate restTemplate(RestTemplateBuilder builder) {
        return builder
            .connectTimeout(Duration.ofSeconds(10))
            .readTimeout(Duration.ofSeconds(30))
            .build();
    }
}

@Service
@RequiredArgsConstructor
public class UserApiClient {
    private final RestTemplate restTemplate;
    private final String baseUrl = "https://api.example.com";

    // ── GET — returns response body directly ──────────────────────────────────
    public UserDto getUser(Long id) {
        return restTemplate.getForObject(baseUrl + "/users/{id}", UserDto.class, id);
    }

    // ── GET — returns ResponseEntity for access to status code and headers ────
    public ResponseEntity<UserDto> getUserWithStatus(Long id) {
        return restTemplate.getForEntity(baseUrl + "/users/{id}", UserDto.class, id);
    }

    // ── Generic type deserialization with ParameterizedTypeReference ──────────
    public List<UserDto> getAllUsers() {
        return restTemplate.exchange(
            baseUrl + "/users",
            HttpMethod.GET,
            null,
            new ParameterizedTypeReference<List<UserDto>>() {}  // Handles generic types
        ).getBody();
    }

    // ── POST ──────────────────────────────────────────────────────────────────
    public UserDto createUser(CreateUserRequest request) {
        HttpHeaders headers = new HttpHeaders();
        headers.setContentType(MediaType.APPLICATION_JSON);
        headers.setBearerAuth(getCurrentToken());

        HttpEntity<CreateUserRequest> entity = new HttpEntity<>(request, headers);
        return restTemplate.postForObject(baseUrl + "/users", entity, UserDto.class);
    }

    // ── PUT ───────────────────────────────────────────────────────────────────
    public void updateUser(Long id, UpdateUserRequest request) {
        restTemplate.put(baseUrl + "/users/{id}", request, id);
    }

    // ── DELETE ────────────────────────────────────────────────────────────────
    public void deleteUser(Long id) {
        restTemplate.delete(baseUrl + "/users/{id}", id);
    }

    // ── Error handling — add an interceptor to handle HTTP errors ─────────────
    // By default, RestTemplate throws HttpClientErrorException (4xx) and
    // HttpServerErrorException (5xx). You can customise this with an ErrorHandler:
    @Bean
    public RestTemplate restTemplateWithErrorHandling(RestTemplateBuilder builder) {
        return builder
            .errorHandler(new ResponseErrorHandler() {
                @Override
                public boolean hasError(ClientHttpResponse response) throws IOException {
                    return response.getStatusCode().isError();
                }
                @Override
                public void handleError(ClientHttpResponse response) throws IOException {
                    if (response.getStatusCode() == HttpStatus.NOT_FOUND) {
                        throw new ResourceNotFoundException("Resource not found");
                    }
                    throw new ServiceException("Service error: " + response.getStatusCode());
                }
            })
            .build();
    }
}
```

---

## 🌊 WebClient — Spring's Non-Blocking HTTP Client

WebClient is part of Spring WebFlux and is Spring's recommended HTTP client for all new development. Unlike RestTemplate (blocking) and even Feign (which can be blocking), WebClient is **fully non-blocking** — it integrates with Project Reactor (`Mono` and `Flux`) and never holds a thread while waiting for a response. This makes it ideal for high-concurrency applications.

Crucially, you can use WebClient in a traditional blocking (Spring MVC) application too — it works alongside RestTemplate. WebClient is not only for reactive applications.

### 21.3.6 WebClient — Core API

```java
import org.springframework.web.reactive.function.client.*;
import reactor.core.publisher.Mono;
import reactor.core.publisher.Flux;

// ── Create a WebClient with a base URL and default configuration ───────────────
WebClient webClient = WebClient.builder()
    .baseUrl("https://api.example.com")
    .defaultHeader(HttpHeaders.ACCEPT, MediaType.APPLICATION_JSON_VALUE)
    .defaultHeader(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE)
    .filter(ExchangeFilterFunctions.basicAuthentication("user", "pass")) // Basic auth filter
    .filter(logRequest())   // Custom logging filter
    .codecs(configurer -> configurer
        .defaultCodecs()
        .maxInMemorySize(10 * 1024 * 1024))  // 10MB buffer for large responses
    .build();

// ── GET — returns Mono<T> (0 or 1 result) ─────────────────────────────────────
Mono<UserDto> userMono = webClient
    .get()
    .uri("/users/{id}", userId)
    .header("X-Request-Id", requestId)
    .retrieve()
    .bodyToMono(UserDto.class);

// Block to get the result synchronously (only appropriate in non-reactive contexts)
UserDto user = userMono.block();

// Or subscribe asynchronously (in a reactive pipeline)
userMono.subscribe(
    u    -> processUser(u),
    err  -> log.error("Failed to get user", err),
    ()   -> log.debug("User fetch complete")
);

// ── GET — returns Flux<T> (0 or many results) ─────────────────────────────────
Flux<UserDto> allUsers = webClient
    .get()
    .uri("/users")
    .retrieve()
    .bodyToFlux(UserDto.class);

// Process as a stream — memory-efficient for large responses
allUsers
    .filter(u -> u.isActive())
    .map(UserDto::getEmail)
    .collectList()  // Collect all into a List<String>
    .subscribe(emails -> sendNewsletter(emails));

// ── POST ──────────────────────────────────────────────────────────────────────
Mono<UserDto> created = webClient
    .post()
    .uri("/users")
    .bodyValue(new CreateUserRequest("Alice", "alice@example.com"))
    .retrieve()
    .onStatus(HttpStatusCode::is4xxClientError, response ->
        response.bodyToMono(ErrorResponse.class)
            .flatMap(err -> Mono.error(new BadRequestException(err.getMessage())))
    )
    .onStatus(HttpStatusCode::is5xxServerError, response ->
        Mono.error(new ServiceUnavailableException("User service unavailable"))
    )
    .bodyToMono(UserDto.class);

// ── PUT ───────────────────────────────────────────────────────────────────────
Mono<UserDto> updated = webClient
    .put()
    .uri("/users/{id}", userId)
    .bodyValue(new UpdateUserRequest("Bob", "bob@example.com"))
    .retrieve()
    .bodyToMono(UserDto.class);

// ── Timeout and retry ─────────────────────────────────────────────────────────
Mono<UserDto> withRetry = webClient
    .get()
    .uri("/users/{id}", userId)
    .retrieve()
    .bodyToMono(UserDto.class)
    .timeout(Duration.ofSeconds(5))            // Cancel if no response in 5 seconds
    .retryWhen(Retry.backoff(3, Duration.ofMillis(500))  // Retry up to 3 times with backoff
        .filter(ex -> ex instanceof WebClientResponseException.ServiceUnavailable)
    )
    .onErrorReturn(UserDto.anonymous());       // Fallback value on persistent failure

// ── ExchangeFilterFunction — cross-cutting concerns like OkHttp interceptors ───
private ExchangeFilterFunction logRequest() {
    return ExchangeFilterFunction.ofRequestProcessor(request -> {
        log.debug("→ {} {}", request.method(), request.url());
        return Mono.just(request);
    });
}

private ExchangeFilterFunction addAuthHeader(TokenProvider tokenProvider) {
    return ExchangeFilterFunction.ofRequestProcessor(request ->
        Mono.just(ClientRequest.from(request)
            .header("Authorization", "Bearer " + tokenProvider.getToken())
            .build())
    );
}
```

---

## ⚖️ HTTP Client Comparison and Decision Guide

The correct choice of HTTP client depends on your context. Here is how to think about it clearly.

If you are building a new Spring Boot microservice and need to call other services, use **Feign** for internal service-to-service calls — the declarative interface is clean, self-documenting, and integrates perfectly with Spring Cloud service discovery and load balancing. Use **WebClient** when you need non-blocking calls, reactive backpressure, or when building with Spring WebFlux.

If you need advanced SSL/TLS configuration (mutual TLS, custom truststore), fine-grained connection pool tuning, or are working in a non-Spring environment, use **Apache HttpClient 5** — it gives you the most control.

If you want a lightweight, interceptor-based client with excellent Android support and need zero Spring dependencies, use **OkHttp**. It is especially natural when combined with Retrofit for declarative APIs in non-Spring contexts.

If you are adding HTTP calls to an existing Spring Boot application that already uses RestTemplate extensively, you can continue with **RestTemplate** — it is stable and fully functional. Just plan to migrate to Feign or WebClient as you touch that code.

The **Java 11 built-in HttpClient** is the right choice for libraries, command-line tools, or any application where you want to avoid adding dependencies. It lacks retry, circuit-breaking, and interceptor support but covers the majority of use cases cleanly.

---

## 💡 Phase 21.3 — Key Takeaways

The HTTP client landscape in Java has matured significantly. The old world of `HttpURLConnection` is gone. For modern Java, you have a rich set of choices with clear differentiation: Feign for declarative microservice calls, WebClient for reactive non-blocking IO, OkHttp for versatile client-side HTTP with interceptors, Apache HttpClient for maximum control, and the built-in Java 11 client for dependency-free use. Understanding this landscape means you can immediately evaluate the HTTP client in any codebase you join and make informed decisions about which to reach for in new code.

# Phase 18 — Part 3: System Design Patterns (Resilience Patterns)

> When you build distributed systems, failures are not exceptional events — they are guaranteed to happen. Networks drop packets, services restart, databases become unavailable for seconds, and downstream APIs have bad days. The patterns in this section are the battle-tested tools that make distributed systems robust in the face of these inevitable failures. Mastering them separates engineers who know microservices in theory from those who can actually run them in production.

---

## 1. Circuit Breaker

The **Circuit Breaker** pattern prevents a system from repeatedly trying an operation that is likely to fail, giving the failing component time to recover while preventing the caller from being blocked or exhausted.

**The core analogy:** Just like an electrical circuit breaker that trips to protect the circuit from overload, a software circuit breaker "trips" when it detects too many failures and stops sending traffic to the struggling service. Once the downstream service has had time to recover, the circuit breaker cautiously allows traffic through again.

**Theory:** Without a circuit breaker, if Service B is slow or down, every call from Service A to Service B will time out after, say, 30 seconds. Meanwhile, threads in Service A are blocked waiting for those responses. New requests keep coming in, new threads keep getting blocked, and very quickly Service A's thread pool is exhausted — even though Service A itself had nothing wrong with it. This is called **cascading failure**. A circuit breaker breaks the cascade by failing fast (immediately returning an error) rather than blocking indefinitely.

### The Three States

A circuit breaker moves through three states:

**CLOSED (normal operation):** Requests flow through normally. The circuit breaker counts failures. If the failure rate or count exceeds a threshold within a time window, the circuit trips open.

**OPEN (failing fast):** No requests flow through to the downstream service. The circuit breaker immediately returns a fallback response (cached data, a default value, an error). After a configured wait period (typically 30-60 seconds), it transitions to HALF-OPEN to test if the downstream service has recovered.

**HALF-OPEN (testing recovery):** A limited number of probe requests are allowed through. If they succeed, the circuit transitions back to CLOSED (normal operation). If they fail, the circuit trips OPEN again and the wait period resets.

```
         failures >= threshold
  ┌──────────────────────────────────────────────────────┐
  │                                                      ▼
CLOSED ──────────────────────────────────────────► OPEN
  ▲                                                      │
  │  probe requests succeed                              │  wait period expires
  │                                                      ▼
  └──────────────────────────────────────────── HALF-OPEN
                                               (probe a few requests)
```

```java
// Resilience4j is the standard Circuit Breaker library in the modern Java/Spring ecosystem
// (Hystrix by Netflix is deprecated; Resilience4j is its successor)
// Add dependency: io.github.resilience4j:resilience4j-spring-boot3

// Configuration in application.yml — every parameter is meaningful
/*
resilience4j:
  circuitbreaker:
    instances:
      inventory-service:
        sliding-window-type: COUNT_BASED      # Count last N calls (not time-based)
        sliding-window-size: 10               # Look at the last 10 calls
        failure-rate-threshold: 50            # Trip OPEN if >= 50% of calls fail
        wait-duration-in-open-state: 30s      # Stay OPEN for 30s before trying HALF-OPEN
        permitted-number-of-calls-in-half-open-state: 3  # Allow 3 probe requests
        minimum-number-of-calls: 5            # Need at least 5 calls before evaluating
*/

@Service
public class OrderService {
    private final InventoryServiceClient inventoryClient;

    // @CircuitBreaker annotation: wraps the method with circuit breaker logic
    // 'name' must match the configuration key in application.yml
    // 'fallbackMethod' is called when the circuit is OPEN or when the method throws
    @CircuitBreaker(name = "inventory-service", fallbackMethod = "reserveInventoryFallback")
    public InventoryReservation reserveInventory(List<OrderItem> items) {
        // This call goes to an external service — it can fail!
        return inventoryClient.reserve(items);
        // If this throws an exception too many times (>= 50% of last 10 calls),
        // Resilience4j will OPEN the circuit and stop calling this method entirely.
    }

    // Fallback method: called automatically when the circuit is OPEN or when the method throws
    // Must have the SAME return type and parameters + a Throwable at the end
    public InventoryReservation reserveInventoryFallback(List<OrderItem> items, Throwable ex) {
        // Log the error so we can investigate
        log.warn("Inventory service unavailable. Using degraded mode. Reason: {}", ex.getMessage());

        // Degraded behavior: instead of failing the entire order, create a "pending" reservation
        // The business may accept this: "We'll process your order once inventory is confirmed"
        return InventoryReservation.pending(items);
        // Alternative fallbacks: return cached data, return a default response,
        // queue the request for later processing, or return an error to the user
    }
}

// Monitoring circuit breaker state — crucial in production
// Resilience4j publishes metrics to Micrometer (Prometheus/Grafana)
@Component
public class CircuitBreakerMonitor {
    private final CircuitBreakerRegistry registry;

    @EventListener
    public void onCircuitBreakerStateChange(CircuitBreakerOnStateTransitionEvent event) {
        // Alert when circuit transitions to OPEN — this means a downstream service is failing
        if (event.getStateTransition().getToState() == CircuitBreaker.State.OPEN) {
            alertingService.critical("Circuit breaker OPENED for: " + event.getCircuitBreakerName()
                + " — downstream service may be down!");
        }
        log.info("Circuit breaker '{}' transitioned from {} to {}",
            event.getCircuitBreakerName(),
            event.getStateTransition().getFromState(),
            event.getStateTransition().getToState());
    }
}
```

---

## 2. Retry with Exponential Backoff

The **Retry** pattern automatically retries a failed operation, based on the premise that many failures in distributed systems are transient — a momentary network hiccup, a brief server overload, or a GC pause. Simply trying again a moment later will often succeed.

**The critical addition — Exponential Backoff:** Retrying immediately and repeatedly can make things worse. If a service is struggling under load and hundreds of clients all retry every second, you've made the overload problem worse. Exponential backoff increases the wait between retries: wait 1 second, then 2, then 4, then 8 — giving the downstream service progressively more time to recover.

**Jitter:** Even with exponential backoff, if all clients started their requests at the same moment (e.g., after a deployment), they'll all retry at the same moments too — creating "thundering herd" spikes. Adding a small random delay (jitter) spreads the retry traffic out, smoothing the load curve.

```java
// Resilience4j Retry configuration — all in application.yml
/*
resilience4j:
  retry:
    instances:
      payment-service:
        max-attempts: 3                     # Try at most 3 times (1 original + 2 retries)
        wait-duration: 500ms                # Initial wait: 500ms
        enable-exponential-backoff: true    # Each retry waits exponentially longer
        exponential-backoff-multiplier: 2   # 500ms → 1000ms → 2000ms
        randomized-wait-factor: 0.3         # ±30% jitter to prevent thundering herd
        retry-exceptions:                   # Only retry on these exception types
          - java.net.ConnectException       # Network issues — likely transient
          - java.net.SocketTimeoutException # Timeout — might succeed on retry
          - feign.RetryableException
        ignore-exceptions:                  # Do NOT retry on these — retrying won't help
          - com.example.PaymentDeclinedException  # Card declined — not a transient error
          - com.example.InsufficientFundsException
*/

@Service
public class PaymentService {

    // @Retry wraps the method: on failure, waits and tries again up to max-attempts
    // Combine with @CircuitBreaker: circuit breaker OPENS when retry exhaustion happens too often
    @Retry(name = "payment-service", fallbackMethod = "chargeCardFallback")
    @CircuitBreaker(name = "payment-service", fallbackMethod = "chargeCardFallback")
    public PaymentResult chargeCard(PaymentRequest request) {
        // This might fail transiently: network blip, payment processor momentarily busy
        return paymentProviderClient.charge(request);
        // Resilience4j will automatically retry with exponential backoff + jitter
    }

    public PaymentResult chargeCardFallback(PaymentRequest request, Throwable ex) {
        // After all retries are exhausted, route to a backup payment processor
        // or queue the payment for asynchronous processing
        log.error("Primary payment processor failed after retries. Trying backup.", ex);
        return backupPaymentClient.charge(request);
    }
}

// Manual retry with exponential backoff — understanding what's happening under the hood
public class ManualRetryDemo {
    private static final int MAX_RETRIES = 3;
    private static final long BASE_DELAY_MS = 500;
    private static final Random random = new Random();

    public PaymentResult chargeWithRetry(PaymentRequest request) throws InterruptedException {
        Exception lastException = null;

        for (int attempt = 1; attempt <= MAX_RETRIES; attempt++) {
            try {
                return paymentClient.charge(request); // Attempt the call
            } catch (TransientException e) {
                lastException = e;
                if (attempt == MAX_RETRIES) break; // Don't sleep on the last attempt

                // Exponential backoff: 500ms, 1000ms, 2000ms...
                long delay = (long) (BASE_DELAY_MS * Math.pow(2, attempt - 1));
                // Jitter: randomize by ±30% to spread out retries
                long jitter = (long) (delay * 0.3 * (random.nextDouble() * 2 - 1));
                long waitMs = delay + jitter;

                log.warn("Attempt {} failed. Retrying in {}ms. Error: {}",
                    attempt, waitMs, e.getMessage());
                Thread.sleep(waitMs);
            } catch (NonTransientException e) {
                // These errors won't get better with retrying — fail immediately
                throw e;
            }
        }
        throw new ServiceUnavailableException("All retries exhausted", lastException);
    }
}
```

---

## 3. Bulkhead

The **Bulkhead** pattern isolates failures in one part of a system from cascading into other parts by partitioning resources into pools, just like a ship's bulkhead compartments prevent water from flooding the entire vessel when one section is breached.

**Theory:** Without bulkheads, a slow downstream service can monopolize all your application's threads. If you have 200 threads and the Inventory Service is slow, those 200 threads might all be blocked waiting for Inventory responses. Now Orders, Users, Products — every feature of your application is broken, even though only Inventory is struggling.

Bulkheads solve this by giving each downstream dependency its own isolated thread pool or connection limit. The Inventory Service gets 20 threads. The Payment Service gets 20 threads. Even if all 20 Inventory threads are stuck, the 180 remaining threads can still serve Orders, Users, and Products normally.

```java
// Thread Pool Bulkhead: each service gets its own isolated thread pool
// If inventory-service's thread pool is full, it rejects ONLY inventory calls
/*
resilience4j:
  thread-pool-bulkhead:
    instances:
      inventory-service:
        max-thread-pool-size: 20        # Maximum concurrent calls to inventory service
        core-thread-pool-size: 10       # Minimum threads kept warm
        queue-capacity: 5               # Queue up to 5 calls before rejecting
        # If 20 threads are busy and queue has 5 more, next call is rejected immediately
        # This protects the rest of the system from inventory service slowness

      payment-service:
        max-thread-pool-size: 10        # Payment is more critical — give it its own pool
        core-thread-pool-size: 5
        queue-capacity: 2
*/

@Service
public class OrderOrchestrationService {

    // @Bulkhead(type = THREADPOOL): runs in the configured thread pool
    // If the thread pool is full, throws BulkheadFullException immediately (no blocking)
    @Bulkhead(name = "inventory-service", type = Bulkhead.Type.THREADPOOL,
              fallbackMethod = "reserveFallback")
    public CompletableFuture<InventoryResult> reserveInventoryAsync(List<OrderItem> items) {
        // This runs in inventory-service's isolated thread pool (max 20 threads)
        // If all 20 threads are busy, the next call fails fast rather than blocking
        return CompletableFuture.supplyAsync(() -> inventoryClient.reserve(items));
    }

    // Semaphore Bulkhead: limits CONCURRENT executions (no separate thread pool)
    // Cheaper than thread pool bulkhead; the call still runs in the caller's thread
    // Use when: the downstream call is non-blocking (reactive/async) or cheap
    @Bulkhead(name = "catalog-service", type = Bulkhead.Type.SEMAPHORE,
              fallbackMethod = "getCatalogFallback")
    public ProductCatalog getProductCatalog(String category) {
        // At most N concurrent calls allowed (configured by max-concurrent-calls)
        return catalogClient.getByCategory(category);
    }

    public CompletableFuture<InventoryResult> reserveFallback(List<OrderItem> items, Throwable ex) {
        // Bulkhead is full — inventory calls are being throttled for system protection
        log.warn("Inventory bulkhead full. Queuing order for async processing.");
        return CompletableFuture.completedFuture(InventoryResult.pendingAsync());
    }
}
```

---

## 4. Timeout

The **Timeout** pattern sets a maximum duration for an operation. If the operation doesn't complete within the allowed time, it is abandoned and an error is returned. Without timeouts, a single slow dependency can block a thread indefinitely — which is how thread pool exhaustion begins.

```java
// Resilience4j TimeLimiter — applies a timeout to any CompletableFuture
/*
resilience4j:
  timelimiter:
    instances:
      external-api:
        timeout-duration: 3s      # Maximum time to wait for a response
        cancel-running-future: true  # Cancel the underlying thread on timeout
*/

@Service
public class ExternalDataService {

    @TimeLimiter(name = "external-api", fallbackMethod = "getExternalDataFallback")
    @CircuitBreaker(name = "external-api")
    // It's very common to stack multiple resilience annotations
    // Timeout protects against slow calls; circuit breaker protects against repeated failures
    public CompletableFuture<ExternalData> getExternalData(String resourceId) {
        return CompletableFuture.supplyAsync(() ->
            externalApiClient.getData(resourceId) // Must complete within 3 seconds
        );
    }

    public CompletableFuture<ExternalData> getExternalDataFallback(
            String resourceId, TimeoutException ex) {
        log.warn("External API timed out for resource {}. Returning cached data.", resourceId);
        return CompletableFuture.completedFuture(
            cacheService.getStaleDataOrEmpty(resourceId)
        );
    }
}

// Timeout in Spring WebClient (reactive HTTP client)
WebClient client = WebClient.builder()
    .baseUrl("https://api.external.com")
    .build();

Mono<ExternalData> data = client.get()
    .uri("/resource/{id}", resourceId)
    .retrieve()
    .bodyToMono(ExternalData.class)
    .timeout(Duration.ofSeconds(3)) // Fail with TimeoutException if no response in 3s
    .onErrorReturn(TimeoutException.class, ExternalData.defaultValue()); // Fallback

// Timeout in standard HTTP client (RestTemplate)
HttpComponentsClientHttpRequestFactory factory = new HttpComponentsClientHttpRequestFactory();
factory.setConnectTimeout(2000);  // Maximum time to establish a connection: 2s
factory.setReadTimeout(5000);     // Maximum time to wait for the response: 5s
RestTemplate restTemplate = new RestTemplate(factory);
```

---

## 5. Fallback

The **Fallback** pattern defines an alternative action when the primary operation fails. Rather than propagating the failure up the call stack where it might crash the user's request, you return something — stale cached data, a default response, a partial response, or a meaningful error message.

```java
// Fallback strategies — choose based on what "acceptable degraded behavior" means for your feature
@Service
public class ProductRecommendationService {

    @CircuitBreaker(name = "recommendation-engine", fallbackMethod = "getRecommendationsFallback")
    public List<Product> getPersonalizedRecommendations(Long userId) {
        return recommendationEngineClient.getRecommendations(userId);
    }

    // Strategy 1: Return stale cached data
    // "Somewhat outdated recommendations are better than no recommendations"
    public List<Product> getRecommendationsFallback(Long userId, Throwable ex) {
        log.warn("Recommendation engine down. Returning cached recommendations for user {}", userId);
        List<Product> cached = cacheService.getLastKnownRecommendations(userId);
        if (!cached.isEmpty()) return cached;
        // If no cache, fall through to Strategy 2
        return getGenericRecommendationsFallback(userId, ex);
    }

    // Strategy 2: Return generic/popular items
    // "Popular items are better than an empty list"
    public List<Product> getGenericRecommendationsFallback(Long userId, Throwable ex) {
        log.warn("No cached recommendations for user {}. Returning bestsellers.", userId);
        return productRepository.findTop10ByOrderBySalesCountDesc(); // Generic popular items
    }

    // Strategy 3: Return empty list with a flag (UI shows "recommendations unavailable")
    // Appropriate when showing nothing is better than showing irrelevant data
    public List<Product> getEmptyFallback(Long userId, Throwable ex) {
        return Collections.emptyList();
    }
}
```

The art of the fallback is choosing the right degradation level. A product recommendation service falling back to "popular items" is a great user experience. A banking balance endpoint falling back to "your balance is $0" would be catastrophic. **Not every operation should have a fallback** — sometimes the correct answer is "this operation failed and the user must be told."

---

## 6. Sidecar Pattern

The **Sidecar** pattern deploys a secondary container (the sidecar) alongside each service container in the same pod (in Kubernetes), sharing the same lifecycle, network namespace, and storage. The sidecar handles cross-cutting concerns — logging, monitoring, TLS termination, service discovery, circuit breaking — independently of the main service.

**Why it's powerful:** Cross-cutting concerns don't need to be implemented in every service. The sidecar handles them uniformly, regardless of what language or framework each service uses. A Java service, a Python service, and a Go service can all get the same observability and security capabilities through their sidecars.

```
┌─────────────────────────────────────────────┐
│                  Kubernetes Pod              │
│                                              │
│  ┌──────────────────┐  ┌───────────────────┐ │
│  │   Order Service   │  │ Envoy Sidecar      │ │
│  │   (your code)     │  │ • TLS termination  │ │
│  │                   │  │ • Circuit breaking │ │
│  │   Port 8080       │  │ • Metrics          │ │
│  │                   │  │ • Distributed tracing│ │
│  └──────────────────┘  │ • Retry logic      │ │
│                         └───────────────────┘ │
│  Shared: network namespace, localhost, volume  │
└─────────────────────────────────────────────┘
```

```java
// The Order Service itself is unaware of the sidecar's presence
// It just makes HTTP calls to other services; the sidecar intercepts them transparently
@RestController
@RequestMapping("/api/orders")
public class OrderController {

    // The app just makes a plain HTTP call to "inventory-service"
    // The Envoy sidecar intercepts this outbound call and transparently adds:
    //   - Mutual TLS (mTLS) for authentication between services
    //   - Circuit breaking based on configured thresholds
    //   - Retry logic on transient failures
    //   - Distributed trace headers (correlation IDs)
    //   - Metrics (request count, latency, error rate) reported to Prometheus
    // The Order Service code has ZERO knowledge of any of this — pure separation of concerns
    private final WebClient inventoryClient = WebClient.builder()
        .baseUrl("http://inventory-service:8080") // Sidecar intercepts all outbound traffic
        .build();

    @PostMapping
    public ResponseEntity<OrderResponse> placeOrder(@RequestBody OrderRequest request) {
        // Just business logic — all resilience and observability handled by sidecar
        InventoryResult inventory = inventoryClient.post()
            .uri("/api/inventory/check")
            .bodyValue(request.getItems())
            .retrieve()
            .bodyToMono(InventoryResult.class)
            .block();
        // ...
    }
}

// Spring Boot actuator endpoints that the sidecar (or monitoring system) reads:
// GET /actuator/health     → Is the service healthy?
// GET /actuator/metrics    → Prometheus-format metrics
// GET /actuator/info       → Service version, build info
```

**The most important sidecar in practice today is Envoy Proxy**, which forms the data plane of service meshes like Istio and Linkerd. When you use Istio, every pod in your cluster gets an Envoy sidecar automatically injected — you get mTLS, circuit breaking, retries, canary routing, and distributed tracing with zero application code changes.

---

## 7. Strangler Fig Pattern

The **Strangler Fig** (named after a vine that slowly grows around and replaces a host tree) is a pattern for migrating from a legacy monolith to a new system incrementally, without a "big bang" rewrite. You build the new system piece by piece alongside the old one, routing increasing amounts of traffic to it over time, until eventually the old system is no longer needed and can be retired.

**Why this approach:** The alternative — stopping feature development for 12-18 months to rewrite everything from scratch — has a catastrophic failure rate. The new system almost never ships on time, and by the time it does, requirements have changed, the team has forgotten why the old system was designed the way it was, and bugs from the original system are faithfully reproduced in the new one. Strangler Fig avoids this by keeping the system running while you migrate.

```java
// The core of Strangler Fig is a Facade/Router that directs traffic to either the old or new system
// This router is the most critical piece — it must be correct even as you change what's behind it

@Component
public class StranglerFigRouter {
    private final LegacyOrderService legacyOrderService;
    private final NewOrderService newOrderService;
    private final FeatureFlagService featureFlags;

    // The router decides which implementation handles each request
    // You control migration via feature flags — no code changes needed to shift traffic
    public OrderResult processOrder(OrderRequest request) {

        if (featureFlags.isEnabled("new-order-service", request.getCustomerId())) {
            // New implementation: initially enabled for 1% of customers, then 10%, then 100%
            // This is canary deployment at the code level — catch bugs before full rollout
            try {
                OrderResult result = newOrderService.process(request);
                // Optionally: compare results with legacy system to validate correctness
                validateAgainstLegacy(request, result);
                return result;
            } catch (Exception e) {
                // If new system fails, automatically fall back to legacy
                // This is the safety net during migration
                log.error("New order service failed, falling back to legacy", e);
                return legacyOrderService.process(request);
            }
        }

        // Default: route to legacy system — safe and known good
        return legacyOrderService.process(request);
    }

    // Shadow mode: new system processes all requests but results are DISCARDED
    // Used to validate the new system without any production impact
    public OrderResult processOrderWithShadowing(OrderRequest request) {
        OrderResult legacyResult = legacyOrderService.process(request); // Always use legacy result

        // Fire-and-forget: run new system in background for comparison, don't affect response
        CompletableFuture.runAsync(() -> {
            try {
                OrderResult newResult = newOrderService.process(request);
                if (!legacyResult.equals(newResult)) {
                    // Log discrepancies for investigation — find all the edge cases before cutover
                    log.warn("Shadow mode discrepancy. Legacy: {}, New: {}", legacyResult, newResult);
                    metricsService.incrementDiscrepancyCounter();
                }
            } catch (Exception e) {
                log.error("Shadow mode failure in new system (not affecting production)", e);
            }
        });

        return legacyResult; // Always return legacy result — new system is invisible to users
    }
}

// Migration percentage increases over time (via feature flags):
// Week 1:  1% of users → new system.  Monitor for errors, compare responses.
// Week 2:  10% of users → new system.  If metrics look good, continue.
// Week 4:  50% of users → new system.  Both systems are equally used.
// Week 6:  100% of users → new system. Legacy is standby only.
// Week 8:  Decommission legacy system.  Migration complete.
```

---

## 8. Anti-Corruption Layer (ACL)

The **Anti-Corruption Layer** is a translation layer between two systems (typically a new system and a legacy system, or two external systems with different domain models) that prevents the concepts and terminology of one system from "leaking" into the other.

**Theory:** When your Order Service needs to integrate with a 20-year-old legacy billing system, that legacy system has its own model of the world: `InvoiceDocument`, `ClientCode`, `ChargeUnit`. If your code starts using these concepts directly, you've imported the legacy system's worldview into your clean domain model. Over time, every reference to "ClientCode" in your codebase is a piece of technical debt — if you ever replace the billing system, you'll need to change every reference. The ACL translates between domains, keeping each side clean.

```java
// The legacy billing system has its own, very different domain model
// It was built in 2003 and uses concepts from 2003 accounting software
class LegacyBillingSystem {
    public InvoiceDocument createInvoice(
        String clientCode,       // Their term for customer ID (format: "CUST-00042")
        List<ChargeUnit> charges, // Their term for line items
        String billingPeriodCode  // "2024-Q1", "2024-Q2", etc.
    ) { /* ... */ }

    public static class ChargeUnit {
        public String sku;
        public int quantity;
        public double unitRate; // Always in USD, despite the field being named "rate"
    }
}

// YOUR domain model — clean, modern, uses YOUR vocabulary
record OrderSummary(
    Long customerId,
    List<OrderLine> lines,
    LocalDate orderDate,
    Currency currency
) {}
record OrderLine(String productId, int quantity, Money unitPrice) {}

// The Anti-Corruption Layer: sits between YOUR domain and the legacy system
// It speaks BOTH languages and translates between them
// Your code never directly touches LegacyBillingSystem — only the ACL does
@Component
public class BillingAntiCorruptionLayer {
    private final LegacyBillingSystem legacyBilling;
    private final CustomerCodeMapper customerMapper; // Maps Long IDs to legacy "CUST-XXXXX" codes

    // YOUR domain calls this method using YOUR vocabulary
    // The ACL translates internally — your domain is completely insulated from legacy concepts
    public BillingResult recordBilling(OrderSummary order) {
        // Translate YOUR domain model → legacy model
        String legacyClientCode = customerMapper.toLegacyCode(order.customerId());
        // "CUST-00042"

        List<LegacyBillingSystem.ChargeUnit> legacyCharges = order.lines().stream()
            .map(line -> {
                LegacyBillingSystem.ChargeUnit charge = new LegacyBillingSystem.ChargeUnit();
                charge.sku = "SKU-" + line.productId(); // Legacy system requires "SKU-" prefix
                charge.quantity = line.quantity();
                // Legacy system only understands USD — convert if needed
                charge.unitRate = currencyConverter.toUsd(line.unitPrice(), order.currency());
                return charge;
            })
            .collect(Collectors.toList());

        // Map your LocalDate to their "YYYY-QX" billing period code
        String legacyBillingPeriod = toBillingPeriodCode(order.orderDate());

        // Call the legacy system using translated parameters
        InvoiceDocument invoice = legacyBilling.createInvoice(
            legacyClientCode, legacyCharges, legacyBillingPeriod);

        // Translate legacy result → YOUR domain model
        // Your domain never sees InvoiceDocument — only BillingResult
        return BillingResult.success(
            invoice.getInvoiceNumber(),    // Their term
            toLocalDate(invoice.getDueDate()), // Their date format → java.time.LocalDate
            invoice.getTotalAmount()
        );
    }

    private String toBillingPeriodCode(LocalDate date) {
        int quarter = (date.getMonthValue() - 1) / 3 + 1;
        return date.getYear() + "-Q" + quarter;
    }
}

// Now in your Order Service — completely clean from legacy concepts:
@Service
public class OrderService {
    private final BillingAntiCorruptionLayer billingAcl; // Only talk through the ACL

    public Order completeOrder(Order order) {
        // Your code is pure domain language — no trace of "ClientCode" or "ChargeUnit"
        BillingResult billing = billingAcl.recordBilling(order.toSummary());
        order.markBilled(billing.invoiceNumber(), billing.dueDate());
        return orderRepository.save(order);
    }
}
```

---

## 9. Ambassador Pattern

The **Ambassador** pattern deploys a helper process (the ambassador) in the same network namespace as the application to handle all outbound network traffic on behalf of the application. Think of it as a local proxy — it handles retry logic, circuit breaking, connection pooling, TLS, and protocol translation, so the application code stays simple.

```java
// The Ambassador pattern at the application level — an SDK or client library acts as the ambassador
// The application calls a simple interface; the ambassador handles all the network complexity

// Simple interface your code calls — no awareness of retries, timeouts, circuit breaking:
public interface PaymentGateway {
    PaymentResult charge(PaymentRequest request);
}

// The Ambassador implementation handles ALL network concerns transparently
@Component
public class PaymentGatewayAmbassador implements PaymentGateway {
    private final PaymentGatewayHttpClient httpClient;   // Raw HTTP client
    private final CircuitBreaker circuitBreaker;
    private final Retry retry;
    private final TimeLimiter timeLimiter;

    @Override
    public PaymentResult charge(PaymentRequest request) {
        // The ambassador chains resilience policies transparently:
        // TimeLimiter wraps Retry wraps CircuitBreaker wraps the actual HTTP call
        Supplier<CompletableFuture<PaymentResult>> decorated =
            TimeLimiter.decorateFutureSupplier(timeLimiter,
                Retry.decorateSupplier(retry,
                    CircuitBreaker.decorateSupplier(circuitBreaker,
                        () -> CompletableFuture.supplyAsync(
                            () -> httpClient.post("/charge", request, PaymentResult.class)
                        )
                    )
                )
            );
        // Your application code calls `charge(request)` and gets a PaymentResult back.
        // Everything else — timeouts, retries, circuit breaking — is completely hidden.
        try {
            return decorated.get().get();
        } catch (Exception e) {
            throw new PaymentException("Payment processing failed", e);
        }
    }
}
```

---

## 10. Important Points & Best Practices

**Layer your resilience patterns thoughtfully.** A common production configuration is: Timeout (fail fast on slow calls) → Retry (handle transient failures) → Circuit Breaker (stop calling a broken service) → Fallback (serve degraded response). The timeout should be shorter than the retry's total time budget. The circuit breaker opens when retries are consistently failing. Order matters.

**Configure timeouts at every level of your stack.** Resilience4j TimeLimiter, HTTP client timeouts (connect timeout AND read timeout), database connection timeouts, and Spring `@Transactional` timeouts are all separate and all necessary. A missing timeout at any level is a thread leak waiting to happen.

**Fallbacks must be carefully designed with the business.** Don't design fallbacks unilaterally. "What should happen when the inventory service is down?" is a business question: do we reject the order? Accept it and handle inventory discrepancies manually? Show an "out of stock" message? The technical implementation of the fallback is easy; getting the right answer from stakeholders is the real work.

**Make your circuit breakers visible.** A circuit breaker that opens silently in production is a nightmare to diagnose. Every state transition (CLOSED → OPEN, OPEN → HALF-OPEN, HALF-OPEN → CLOSED) should generate a log entry, a metric, and ideally an alert. When the circuit opens, your on-call engineer needs to know within seconds, not minutes.

**The Strangler Fig requires patience and discipline.** The hardest part of a Strangler Fig migration is keeping both systems synchronized during the transition. Every schema change, every feature addition must be applied to both the legacy and the new system. Define a clear migration completion criteria and timeline — otherwise "migration" becomes a permanent state where you maintain two systems forever.

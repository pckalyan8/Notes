# 19.2 — Technical Documentation in Java

> **Goal:** Master the art of writing documentation that engineers actually read — from Javadoc to Architecture Decision Records to operational runbooks.

---

## 🧠 Why Documentation Is a Lead Skill

Experienced engineers often resist documentation, seeing it as overhead that slows them down. But consider this: every time a colleague has to interrupt you to ask "what does this method do?" or "why did we choose this library?", that's documentation debt being paid in real time, at high cost.

As a software lead, your job is to make the team scale beyond your own bandwidth. Good documentation allows knowledge to flow without you being the bottleneck. It's not overhead — it's **engineering leverage**.

---

## 📖 Javadoc — Writing Effective API Documentation

Javadoc is Java's built-in documentation system. It processes comments in your source code to generate HTML API documentation (like the official Java SE documentation you use every day). 

The key principle of Javadoc is to document the **what and why**, not the **how**. The code already shows how — your job is to explain intent, contracts, and constraints that aren't obvious from reading the code itself.

### Basic Javadoc Anatomy

```java
/**
 * Calculates the compound interest for a given principal over a time period.
 *
 * <p>This method uses the standard compound interest formula:
 * A = P(1 + r/n)^(nt), where P is principal, r is annual interest rate,
 * n is compounding periods per year, and t is time in years.
 *
 * <p>For monetary calculations where precision is critical, callers should
 * use {@link #calculateCompoundInterestExact(BigDecimal, double, int, int)}
 * instead, as this method uses double arithmetic which can introduce
 * floating-point rounding errors.
 *
 * @param principal     the initial investment amount; must be positive
 * @param annualRate    the annual interest rate as a decimal (e.g., 0.05 for 5%);
 *                      must be between 0.0 and 1.0 inclusive
 * @param periodsPerYear the number of times interest compounds per year (e.g., 12 for monthly)
 * @param years         the investment duration in years; must be positive
 * @return the total accumulated amount (principal + interest) after the given period
 * @throws IllegalArgumentException if principal or years is not positive,
 *                                   or if annualRate is outside [0.0, 1.0]
 * @see #calculateCompoundInterestExact(BigDecimal, double, int, int)
 * @since 2.3.0
 */
public double calculateCompoundInterest(double principal, double annualRate,
                                         int periodsPerYear, int years) {
    if (principal <= 0) throw new IllegalArgumentException("Principal must be positive, got: " + principal);
    if (years <= 0) throw new IllegalArgumentException("Years must be positive, got: " + years);
    if (annualRate < 0.0 || annualRate > 1.0) {
        throw new IllegalArgumentException("Annual rate must be in [0.0, 1.0], got: " + annualRate);
    }
    return principal * Math.pow(1 + annualRate / periodsPerYear, periodsPerYear * years);
}
```

The most important Javadoc tags and their purposes are as follows. The `@param` tag documents each parameter, explaining not just the type but the valid range and meaning. The `@return` tag explains what comes back, including what "empty" or edge-case returns mean. The `@throws` tag documents every exception that can be thrown — including unchecked exceptions if callers need to be aware. The `@see` tag links to related methods or classes. The `@since` tag indicates when this API was introduced, which helps maintainers track compatibility.

### Documenting Interfaces — The Contract Definition

Interfaces are the most important things to document in Java because they represent contracts. The implementation classes are already constrained by the interface; the documentation should explain what every implementor must guarantee.

```java
/**
 * Defines the contract for persisting and retrieving {@link Customer} entities.
 *
 * <p>Implementations of this interface must be thread-safe. Multiple threads
 * may call any method concurrently without external synchronization.
 *
 * <p>All methods in this interface treat {@code null} customer IDs as invalid
 * and will throw {@link IllegalArgumentException}. Null customer objects passed
 * to write methods will throw {@link NullPointerException}.
 */
public interface CustomerRepository {

    /**
     * Persists a new customer or updates an existing one.
     *
     * <p>If the customer's ID is null, a new record is created and the returned
     * customer will have a system-generated ID populated. If the ID is non-null,
     * the existing record is updated if it exists.
     *
     * @param customer the customer to save; must not be null
     * @return the saved customer with all system-generated fields (ID, timestamps) populated
     * @throws DuplicateEmailException if another customer already exists with the same email
     */
    Customer save(Customer customer);

    /**
     * Finds a customer by their unique identifier.
     *
     * @param id the customer's unique identifier; must not be null
     * @return an {@link Optional} containing the customer if found, or empty if not found
     * @throws IllegalArgumentException if id is null
     */
    Optional<Customer> findById(UUID id);
}
```

### What NOT to Document

A common mistake is documenting the obvious — this adds noise without adding value. If someone has to read a comment to understand something that the code already says clearly, the comment is useless and creates maintenance burden.

```java
// ❌ BAD: Documents the obvious — "i is incremented" tells us nothing new
// Increment the counter by 1
i++;

// ❌ BAD: Restates the method name — useless
/**
 * Gets the user's name.
 * @return the user's name
 */
public String getUserName() { return userName; }

// ✅ GOOD: Documents what isn't obvious — the contract and behavior
/**
 * Returns the display name for this user.
 *
 * <p>Returns the user's first and last name separated by a space if both are set.
 * If only a first name is set, returns that alone. If neither is set, returns
 * "Anonymous User".
 *
 * @return the display name; never null, never empty
 */
public String getDisplayName() { ... }
```

---

## 📝 Architecture Decision Records (ADRs)

An ADR is a short document that records a significant architectural decision: what was decided, why, and what alternatives were considered. Without ADRs, institutional knowledge lives only in people's heads — and when those people leave, the reasoning behind critical decisions disappears.

ADRs answer the most frustrating question in software development: **"Why on earth did we do it this way?"**

### ADR Format

ADRs are intentionally lightweight. Store them as Markdown files in your repository, typically in a directory called `docs/adr/` or `docs/decisions/`. Each ADR is numbered sequentially and never deleted — if a decision is superseded, you create a new ADR and mark the old one as superseded.

```markdown
# ADR-0004: Use Redis for Session Storage

**Date:** 2024-11-15  
**Status:** Accepted  
**Deciders:** Alice Chen (Tech Lead), Bob Martinez (Backend), Carol Wong (Platform)

---

## Context

Our monolithic application currently stores HTTP sessions in server memory 
(using Spring Session with in-memory store). As we scale to multiple application 
instances behind a load balancer, sessions are lost when requests are routed to 
a different instance than the one where the session was created. This causes users 
to be unexpectedly logged out, which is impacting our NPS score.

We need a shared, distributed session store.

## Decision

We will use **Redis** (via Spring Session Data Redis) as our distributed session store.

## Consequences

**Positive:**
- Sessions survive instance restarts and scale across any number of app instances
- Redis's in-memory nature keeps session reads sub-millisecond
- Spring Session's Redis integration is mature and well-documented
- We already use Redis for caching, so no new infrastructure to manage

**Negative:**
- Redis becomes a single point of failure for authentication. Mitigated by 
  Redis Sentinel/Cluster setup (see ADR-0005).
- Session data must be serializable. All objects stored in the session must 
  implement Serializable or use JSON serialization.
- Operational complexity increases slightly — Redis must be monitored and backed up.

## Alternatives Considered

**Database-backed sessions (Spring Session JDBC):**  
Simpler operationally, but database becomes a bottleneck for session reads on 
every authenticated request. Not appropriate for our expected traffic volume.

**Sticky sessions (load balancer affinity):**  
Avoids shared store entirely, but sessions are still lost on instance restarts 
and doesn't work well with auto-scaling. This was rejected as a band-aid, not a fix.

**Hazelcast:**  
A viable distributed data grid alternative to Redis. Rejected primarily because 
we already have Redis operational expertise on the team and the additional learning 
curve wasn't justified.
```

The ADR above demonstrates the key qualities of a good ADR: it explains the **problem** clearly (sessions lost during scaling), the **decision** concisely, the **consequences** honestly (both positive and negative), and the **alternatives** that were genuinely considered. The "Alternatives Considered" section is the most valuable part — it prevents future engineers from re-evaluating the same options you already ruled out.

---

## 📄 README and Contributing Guides

The README is the front door to your project. It should answer one question above all others: **"How do I get this running on my laptop in the next 10 minutes?"**

```markdown
# Payments Service

Handles payment processing, refunds, and payment method management for the platform.

## Prerequisites

- Java 21 (install via [SDKMAN](https://sdkman.io/): `sdk install java 21.0.2-tem`)
- Docker & Docker Compose (for local database and Redis)
- Maven 3.9+

## Quick Start

```bash
# 1. Clone and enter the repo
git clone git@github.com:company/payments-service.git && cd payments-service

# 2. Start dependencies (PostgreSQL, Redis, WireMock for external payment gateway)
docker-compose up -d

# 3. Run database migrations
mvn flyway:migrate

# 4. Start the application
mvn spring-boot:run

# 5. Verify it's running
curl http://localhost:8080/actuator/health
```

The app starts on port **8080**. API documentation is available at 
`http://localhost:8080/swagger-ui.html`.

## Running Tests

```bash
mvn test                    # Unit tests only (fast, ~30 seconds)
mvn verify                  # All tests including integration tests (~3 minutes)
mvn verify -Pload-test      # Run load tests (requires full environment)
```

## Configuration

Configuration is loaded from `application.yml`. For local development, create 
`application-local.yml` (gitignored) to override settings. See 
`application-local.yml.example` for a template.

Sensitive values (API keys, DB passwords) should never be committed. Use 
environment variables or Vault. See [Secrets Management](docs/secrets.md).

## Architecture

See [Architecture Overview](docs/architecture.md) and the [ADR index](docs/adr/).

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md).
```

---

## 🔧 API Documentation with OpenAPI / Springdoc

Springdoc-OpenAPI integrates with Spring Boot to automatically generate API documentation from your controller annotations. This means your documentation stays in sync with your code — unlike hand-maintained wiki pages.

```java
// Add springdoc dependency in pom.xml:
// <dependency>
//   <groupId>org.springdoc</groupId>
//   <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
//   <version>2.3.0</version>
// </dependency>

@RestController
@RequestMapping("/api/v1/orders")
@Tag(name = "Orders", description = "Order management operations")
public class OrderController {

    /**
     * Creates a new order for the authenticated customer.
     */
    @Operation(
        summary = "Create a new order",
        description = "Creates a new order with the provided items. Inventory is reserved " +
                      "immediately upon creation. Payment is processed asynchronously."
    )
    @ApiResponses({
        @ApiResponse(responseCode = "201", description = "Order created successfully",
            content = @Content(schema = @Schema(implementation = OrderResponse.class))),
        @ApiResponse(responseCode = "400", description = "Invalid request — missing items or invalid quantities",
            content = @Content(schema = @Schema(implementation = ErrorResponse.class))),
        @ApiResponse(responseCode = "409", description = "One or more items are out of stock",
            content = @Content(schema = @Schema(implementation = ErrorResponse.class)))
    })
    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public OrderResponse createOrder(
            @RequestBody @Valid @io.swagger.v3.oas.annotations.parameters.RequestBody(
                description = "Order details including items and shipping address",
                required = true
            ) CreateOrderRequest request) {
        return orderService.createOrder(request);
    }
}

// Document your request/response models too
@Schema(description = "Request body for creating a new order")
public record CreateOrderRequest(

    @Schema(description = "List of items to order", minItems = 1)
    @NotEmpty @Valid
    List<OrderItemRequest> items,

    @Schema(description = "Shipping address for physical delivery; null for digital items")
    @Valid
    ShippingAddressRequest shippingAddress
) {}
```

With this setup, navigating to `http://localhost:8080/swagger-ui.html` gives you an interactive API explorer that stays automatically in sync with your code changes.

---

## 📚 Runbooks — Operational Documentation

A runbook is a document that tells an on-call engineer exactly what to do when an alert fires at 2am. Its audience is a tired, stressed person who needs to act quickly. Write accordingly: be direct, be complete, and be concrete.

```markdown
# Runbook: Payments Service — High Error Rate Alert

**Alert:** `payments.error_rate > 5% for 5 minutes`  
**Severity:** P1 (customer-facing impact)  
**On-call channel:** #payments-oncall

---

## Immediate Triage (First 5 Minutes)

**Step 1:** Check the Grafana dashboard → [Payments Error Rate Dashboard](https://grafana.internal/d/payments)

**Step 2:** Identify which error type is spiking:
- Go to Kibana: `service:payments AND level:ERROR`
- Look for the most frequent `exception.class` value in the last 15 minutes

**Step 3:** Based on the error class, follow the relevant section below.

---

## Scenario A: `PaymentGatewayTimeoutException` 

This means Stripe is not responding within our timeout (currently 10s).

1. Check Stripe's status page: https://status.stripe.com  
2. Check our Stripe latency metric in Grafana: `stripe.response_time_p99`  
3. If Stripe is degraded: payments will auto-retry for up to 30 minutes. 
   **No immediate action needed unless retry queue is growing** 
   (check `payments.retry_queue_depth` — alert if > 1000).  
4. If Stripe appears healthy: check our outbound connectivity — 
   run `curl -I https://api.stripe.com` from a pod:
   ```bash
   kubectl exec -n payments deployment/payments-service -- curl -I https://api.stripe.com
   ```

## Scenario B: `DataAccessException` (database errors)

1. Check PostgreSQL health: [DB Dashboard](https://grafana.internal/d/postgres)  
2. Check active connection count — alert if > 80% of connection pool  
3. If connection pool exhausted: restart the service to reset connections:
   ```bash
   kubectl rollout restart deployment/payments-service -n payments
   ```
4. Check for long-running queries blocking others:
   ```sql
   SELECT pid, now() - pg_stat_activity.query_start AS duration, query 
   FROM pg_stat_activity 
   WHERE state = 'active' AND now() - query_start > interval '30 seconds';
   ```

---

## Escalation

If not resolved within 30 minutes, escalate to:
- **Primary:** Alice Chen (alice@company.com, +1-555-0100)  
- **Secondary:** Bob Martinez (bob@company.com, +1-555-0101)

Post a status update to #incidents every 30 minutes.
```

The hallmark of a great runbook is that someone with no context can follow it successfully. Include actual commands, actual URLs to dashboards, and actual escalation contacts — not placeholders.

---

## 🗺️ Onboarding Documentation

Think about the last time you joined a new team. What questions did you spend the first week trying to answer? Onboarding documentation pre-answers those questions so new engineers can be productive quickly.

A good onboarding doc covers: environment setup step by step, an architecture overview (a diagram goes a long way), a glossary of domain terms and internal jargon, a guide to the development workflow (how to branch, how to PR, how to deploy to staging), and answers to the five most common "gotcha" questions that every new person asks.

---

## 💡 Key Takeaways

Documentation is a force multiplier. Javadoc makes your APIs self-explaining. ADRs preserve institutional knowledge across team changes. READMEs onboard new engineers without requiring anyone's time. Runbooks keep incidents from becoming disasters. Writing documentation well is a professional discipline — one that separates individual contributors from engineers who make entire teams more effective.

**The best time to write documentation is immediately after you figure something out, while the context is still fresh.** Don't let it become a separate task that gets pushed to the bottom of the backlog.

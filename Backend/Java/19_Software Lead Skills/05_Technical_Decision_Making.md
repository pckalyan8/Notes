# 19.5 — Technical Decision Making for Java Leads

> **Goal:** Build the judgment to evaluate technology choices, plan and execute migrations, manage risk, and lead the team from where the codebase is to where it needs to be.

---

## 🧠 Why Decision-Making Is a Distinct Skill

Every day as a technical lead involves decisions: which library to adopt, whether to refactor or rewrite, how to break a monolith into services, whether a proposed architecture will hold up under load. These decisions are rarely clear-cut, and the consequences of getting them wrong can ripple through the codebase for years.

The difference between an experienced lead and an inexperienced one is rarely technical knowledge — it's **judgment**: the ability to weigh tradeoffs clearly, consider second-order effects, acknowledge uncertainty, and make a decision that the team can align behind. This section is about how to develop and apply that judgment.

---

## ⚖️ Evaluating Technology Choices (Build vs. Buy)

One of the most common technical decisions is whether to build something yourself or adopt an existing library, framework, or service. The instinct to build is strong in engineers — we're problem-solvers who enjoy creating things. But the instinct is often wrong.

The total cost of building includes not just the initial implementation but the ongoing costs: every bug you have to fix, every feature you have to add, every integration you have to maintain, and every new engineer who has to learn your custom solution instead of reading documentation that already exists. Libraries like Spring, Hibernate, and Caffeine represent years of development and battle-testing across thousands of production systems. You're unlikely to outperform them with weeks of internal effort.

The case for building rather than buying is strongest when the available options don't fit your specific constraints in important ways, when you have a deep competitive advantage in this particular area, when the external solution brings significant risk (a dependency with poor maintenance, a license incompatible with your business, an API so complex it creates more problems than it solves), or when the thing you need is so small and simple that a dependency introduces more complexity than the code itself would.

A practical evaluation framework for choosing between external options looks like this:

```
Technology Evaluation: Choosing a Java HTTP Client Library

Decision: Select an HTTP client for service-to-service communication in our
          Spring Boot 3.x microservices.

Candidates: Java 11 HttpClient (built-in), OkHttp, Apache HttpClient 5, Feign

Criteria:
  1. Connection pooling and performance under concurrent load
  2. Timeout configuration granularity (connect, read, write separately)
  3. Retry and circuit breaker integration
  4. Spring Boot auto-configuration support
  5. Maintenance health (active development, responsive to CVEs)
  6. Team familiarity

Decision: Feign (declarative client) backed by OkHttp
Reasoning: Feign's declarative style eliminates boilerplate for calling 
  well-defined APIs, aligns with our existing Spring Cloud infrastructure, 
  and OkHttp provides excellent connection pooling under high concurrency. 
  Resilience4j integrates naturally for retry/circuit breaker behavior.
  
Risk: Feign adds a layer of abstraction — debugging requires understanding 
  both Feign and OkHttp. Mitigated by ensuring team training and good logging.

Decision Date: 2024-11-20
Owner: [Lead name]
Review: Revisit if we exceed 10k RPS per service (OkHttp's thread model may 
  not suit high-concurrency reactive workloads at that scale).
```

Note the "Review" entry. Good technical decisions include an expiration condition — a signal that tells you when to revisit the decision, rather than blindly continuing indefinitely.

---

## 🔬 Proof of Concept (POC) Planning

A Proof of Concept is a time-boxed, focused experiment to answer a specific technical question before committing to a direction. The key word is **specific**: a POC that sets out to "evaluate Kafka" is too vague and will run forever. A POC that sets out to "determine whether Kafka can process our event volume of 50k events/second with p99 latency under 100ms on our current infrastructure" has a clear question and a clear success criterion.

A well-structured POC looks like this:

**Question:** Can we achieve sub-100ms p99 latency on Kafka-based event delivery with 50k events/second?

**Scope:** Implement a minimal producer/consumer using Spring Kafka on our current cluster configuration. No UI, no persistence, no business logic — just the messaging layer.

**Timebox:** 3 days

**Success criteria:** p99 latency < 100ms measured by a JMH benchmark with 50k events/second throughput sustained for 5 minutes. Consumer lag < 10k messages.

**What we will NOT explore:** Kafka Streams, Schema Registry, consumer group strategies, multi-region replication. These are follow-on questions if the POC succeeds.

**Outcome document:** Written decision with benchmark results, configuration used, caveats, and go/no-go recommendation.

A POC that doesn't answer its question in the allotted time produces a partial answer: "We found these two things, but couldn't get to X — to complete the evaluation we'd need Y more days." This is still valuable — it prevents indefinite exploration by forcing an explicit decision about whether to invest more time.

```java
// Example: POC benchmark setup using JMH
// This is the kind of concrete artifact a good POC produces

@BenchmarkMode(Mode.Throughput)
@OutputTimeUnit(TimeUnit.SECONDS)
@State(Scope.Thread)
public class KafkaProducerBenchmark {

    private KafkaProducer<String, String> producer;
    private static final String TOPIC = "poc-events";

    @Setup
    public void setup() {
        Properties props = new Properties();
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        // Tune for throughput: larger batches, slight delay to fill them
        props.put(ProducerConfig.BATCH_SIZE_CONFIG, 65536);
        props.put(ProducerConfig.LINGER_MS_CONFIG, 5);
        props.put(ProducerConfig.COMPRESSION_TYPE_CONFIG, "lz4");
        this.producer = new KafkaProducer<>(props);
    }

    @Benchmark
    public Future<RecordMetadata> sendEvent() {
        // Each benchmark iteration sends one event and returns the future
        // JMH measures how many we can send per second
        return producer.send(
            new ProducerRecord<>(TOPIC, "key", generatePayload())
        );
    }

    @TearDown
    public void teardown() {
        producer.close();
    }
}
```

---

## ⚠️ Risk Assessment

Every significant technical decision carries risk, and part of a lead's job is to surface and quantify risk clearly — not to avoid all risk (that leads to stagnation) but to take **informed** risks and have mitigation plans ready.

A simple but effective risk framework is to score each identified risk on two dimensions: **probability** (how likely is this to happen?) and **impact** (how bad would it be if it did?). The product of these two scores tells you where to focus your mitigation efforts.

```
Migration Risk Assessment: Upgrading Payment Service from Java 11 to Java 21

Risk 1: Third-party PDF library is not Java 21 compatible
  Probability: Medium (library is actively maintained but last release was 8 months ago)
  Impact: High (payments require invoice PDF generation)
  Mitigation: Test PDF generation in a Java 21 environment FIRST, before upgrading 
  anything else. Identify a replacement library (Apache PDFBox) as a fallback.

Risk 2: Virtual threads break code that relies on synchronized blocks for thread-local state
  Probability: Low (we use ReentrantLock, not synchronized, in most places)
  Impact: Medium (could cause subtle deadlocks in specific code paths)
  Mitigation: Search codebase for synchronized usage in I/O-related code. 
  Use JFR virtual thread pinning events to detect issues in staging.

Risk 3: Deployment pipeline doesn't support Java 21 Docker image yet
  Probability: High (DevOps team hasn't added the image)  
  Impact: Low (easy fix, just needs a ticket)
  Mitigation: File ticket with DevOps before starting migration. 
  Block-risk: 0 — can be parallelized.

Go/No-Go: Proceed. Risk 1 can be mitigated with a pre-flight test. 
Risk 2 is low probability. Risk 3 is logistical, not technical.
```

---

## 🚀 Migration Planning

Technology migrations — library upgrades, framework version bumps, database changes, service splits — are among the highest-risk activities in software engineering because they combine technical complexity with operational risk. The key to safe migrations is the **strangler fig pattern**: you grow the new thing alongside the old thing, gradually routing traffic from old to new, until the old thing can be safely removed.

The strangler fig pattern applied to a Spring Boot 2.x → 3.x migration works like this. You don't upgrade the entire application at once. Instead, you identify the highest-risk changes (Security configuration, Hibernate 6 breaking changes, removed deprecated APIs, Jakarta EE namespace changes) and address them one by one in separate, reviewable PRs. Each PR leaves the application in a working, deployable state.

```
Spring Boot 2.7 → 3.x Migration Plan

Phase 1: Prepare (no runtime change)
  - Remove all deprecated API usage flagged by Spring Boot 2.7 deprecation warnings
  - Replace javax.* imports with jakarta.* across the codebase
  - Add spring-boot-properties-migrator to catch configuration renames automatically
  - Ensure all tests pass with deprecation warnings treated as errors

Phase 2: Spring Security migration
  - Replace WebSecurityConfigurerAdapter with SecurityFilterChain bean
  - Update Security test configuration
  - Deploy to staging, run full regression test suite

Phase 3: Hibernate 6 migration
  - Update any HQL queries using syntax that changed in Hibernate 6
  - Verify all entity mappings still work (GenerationType.AUTO changed behavior)
  - Run all JPA integration tests against production-like data volume

Phase 4: Upgrade spring-boot-parent to 3.x
  - All transitive dependency versions change — run full test suite
  - Deploy to staging with production traffic shadow (20% of real traffic)
  - Monitor for 48 hours before full rollout
  - Keep ability to roll back for 1 week post-deployment

Phase 5: Remove migration helpers and update remaining dependencies
```

The critical principle here is **never be in a state where you can't deploy**. If Phase 2 introduces a regression, you should be able to revert just Phase 2 without losing Phase 1's work. This is why each phase ends in a deployable, tested state.

---

## 🏛️ Legacy System Modernization

Working with legacy systems is an unavoidable part of professional Java development. A "legacy system" in Java context typically means one or more of: Java 8 or earlier with outdated dependencies, no test coverage (making changes terrifying), a monolithic architecture under microservices pressure, undocumented business logic embedded in SQL stored procedures or 10,000-line service classes, and absence of the original developers who could explain the decisions.

The key insight about legacy modernization is that it is never about rewriting everything — it's about **incrementally improving** while delivering value. The "big rewrite" fantasy is almost always a disaster: it takes far longer than expected, the new system doesn't replicate subtle behaviors of the old one that turned out to be important, and the business stops receiving new features for 6–18 months.

The process that works is: start by adding tests to whatever you're about to change (not to the whole codebase — just the parts you're touching), then make your change with confidence that tests will catch regressions, then refactor the code you just touched to be clearer. Repeat this cycle thousands of times over months and years and the codebase gradually improves without ever requiring a freeze.

```java
// Before: Legacy code — untested, hard to change
// This service class has grown to 3000 lines with no tests.
// You need to change how discounts are calculated.

public class OrderService {
    // ... hundreds of lines of unrelated concerns ...
    
    public double calculateTotal(Order order) {
        double total = 0;
        for (OrderItem item : order.getItems()) {
            total += item.getPrice() * item.getQuantity();
            // Discount logic buried in the middle of price calculation
            if (item.getCategory().equals("ELECTRONICS") && order.getCustomer().isPremium()) {
                total -= item.getPrice() * item.getQuantity() * 0.1;
            }
        }
        return total;
    }
    // ... hundreds more lines ...
}

// Modernization step 1: Write characterization tests BEFORE changing anything.
// Characterization tests document what the code CURRENTLY does — even if wrong.
@Test
void calculateTotal_premiumCustomerWithElectronics_applies10PercentDiscount() {
    // Arrange: Set up the exact scenario the production code handles
    Customer premiumCustomer = CustomerBuilder.premium().build();
    OrderItem laptop = OrderItemBuilder.electronics().price(1000).quantity(1).build();
    Order order = OrderBuilder.withCustomer(premiumCustomer).withItem(laptop).build();
    
    // Act
    double total = orderService.calculateTotal(order);
    
    // Assert: We observed this is what the current code returns
    assertEquals(900.0, total, 0.001);  // 10% discount applied
}

// Modernization step 2: Extract the discount logic to a testable, named method
// This improves clarity without changing behavior — tests still pass.
private double applyDiscounts(OrderItem item, Customer customer) {
    double itemTotal = item.getPrice() * item.getQuantity();
    if (item.getCategory().equals("ELECTRONICS") && customer.isPremium()) {
        return itemTotal * 0.9;  // 10% discount for premium customers on electronics
    }
    return itemTotal;
}

// Modernization step 3: In a future PR, extract to a DiscountService with 
// its own tests — now it's truly isolated and can evolve independently.
```

---

## 🗺️ Technical Roadmapping

A technical roadmap is a forward-looking plan for how the engineering system will evolve — aligned with business goals but driven by engineering judgment. Unlike a product roadmap (which focuses on user-facing features), a technical roadmap covers infrastructure, platform, architecture evolution, and capability building.

A good technical roadmap entry answers four questions: what is the current state (and what pain does it cause), what is the desired target state, what is the approximate timeline and effort, and what business value does achieving this unlock?

```markdown
## Technical Roadmap Item: Decompose Order Service from Monolith

Current State: Order management, inventory, shipping, and invoicing logic all 
live in the monolith. A change to shipping calculation requires a full monolith 
deploy. The shipping team cannot release independently.

Target State: A standalone Shipping Service with its own deployment pipeline, 
database, and API contract. The monolith calls the Shipping Service via HTTP.

Timeline: Q3 2025 (estimated 6 weeks of work)

Business Value: 
  - Enables the shipping team to release new carrier integrations in hours 
    instead of days (currently blocked by monolith release process)
  - Reduces blast radius of shipping-related bugs — a shipping bug no longer 
    risks taking down order processing
  - Unlocks independent scaling (shipping calculation is CPU-intensive during 
    peak seasons — it can now scale separately)

Dependencies: 
  - ADR-0007 (data migration strategy) must be completed first
  - DevOps: new CI/CD pipeline for shipping-service
```

Communicating the technical roadmap to product and business stakeholders is a key lead skill. The framing that works best is to tie engineering investments directly to business outcomes: not "we need to split the monolith" but "splitting the shipping service will let us integrate new carrier APIs in hours instead of weeks, which directly impacts our shipping cost optimization initiative."

---

## 💡 Key Takeaways

Technical decision-making improves with deliberate practice and reflection. Keep a decision log — not just the decisions you made, but what you expected to happen and what actually happened. The gap between expectations and outcomes is where the learning lives. Over time, you build calibration: an intuition for how long things actually take, which risks tend to materialize, and which technology choices age well. That calibration, more than any specific technical knowledge, is what makes a lead invaluable.

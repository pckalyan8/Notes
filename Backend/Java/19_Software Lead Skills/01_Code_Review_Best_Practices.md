# 19.1 — Code Review Best Practices in Java

> **Goal:** Understand what makes a great code review, how to give and receive feedback effectively, and what to specifically look for when reviewing Java code.

---

## 🧠 Why Code Reviews Matter

Code reviews are one of the highest-leverage activities a software lead can engage in. A single hour of review can prevent hours — or days — of debugging in production. But beyond catching bugs, reviews serve as a **knowledge distribution mechanism**: the reviewer learns the codebase, the author gets a second set of eyes, and the team builds shared standards over time.

Think of code review not as gatekeeping, but as **collaborative improvement**. The goal is not to find fault — it's to ship better software together.

---

## 📋 What to Look For in a Java Code Review

### 1. Correctness

Correctness is the most fundamental concern. The code must do what it claims to do under all expected (and unexpected) inputs.

**Things to check:**
- Does the logic match the requirements or ticket description?
- Are edge cases handled — null inputs, empty collections, boundary values?
- Are exceptions caught at the right level, or swallowed silently?

```java
// ❌ BAD: Exception is swallowed — failure is invisible
public User findUser(long id) {
    try {
        return userRepository.findById(id);
    } catch (Exception e) {
        // Nothing here — caller has no idea it failed
        return null;
    }
}

// ✅ GOOD: Meaningful exception handling with context
public User findUser(long id) {
    return userRepository.findById(id)
        .orElseThrow(() -> new UserNotFoundException("No user found with id: " + id));
}
```

**Key question to ask yourself during review:** "What happens if this method receives null? An empty list? A negative number? A duplicate entry?"

---

### 2. Performance

You don't need to micro-optimize everything, but obvious performance problems should be caught early — fixing them in production is far more expensive.

**Common Java performance issues to watch for:**

```java
// ❌ BAD: String concatenation inside a loop creates many temporary objects
public String buildReport(List<String> lines) {
    String result = "";
    for (String line : lines) {
        result += line + "\n";  // Each iteration creates a new String object
    }
    return result;
}

// ✅ GOOD: StringBuilder avoids repeated allocation
public String buildReport(List<String> lines) {
    StringBuilder sb = new StringBuilder();
    for (String line : lines) {
        sb.append(line).append('\n');
    }
    return sb.toString();
}
```

```java
// ❌ BAD: N+1 query problem — database hit inside a loop
public List<OrderDTO> getOrders(List<Long> orderIds) {
    return orderIds.stream()
        .map(id -> orderRepository.findById(id))  // 1 query per ID
        .collect(Collectors.toList());
}

// ✅ GOOD: Batch fetch — single query for all IDs
public List<OrderDTO> getOrders(List<Long> orderIds) {
    return orderRepository.findAllById(orderIds);  // 1 query total
}
```

Also look for:
- Unnecessary object creation inside hot loops
- Missing database indexes (often caught by reviewing queries alongside schema)
- Unbounded queries (no `LIMIT` clause — can return millions of rows)
- Eager loading where lazy would suffice in JPA (`FetchType.EAGER` on large collections)

---

### 3. Security

Security issues are among the most costly to fix after deployment. A Java code reviewer should always have a security mindset.

```java
// ❌ BAD: SQL injection vulnerability — user input in query string
public List<User> searchUsers(String name) {
    String query = "SELECT * FROM users WHERE name = '" + name + "'";
    return jdbcTemplate.query(query, userRowMapper);
    // Attacker input: "' OR '1'='1" — returns all users
}

// ✅ GOOD: Parameterized query prevents injection
public List<User> searchUsers(String name) {
    String query = "SELECT * FROM users WHERE name = ?";
    return jdbcTemplate.query(query, userRowMapper, name);
}
```

```java
// ❌ BAD: Sensitive data logged — appears in log aggregation systems
public void processPayment(PaymentRequest request) {
    log.info("Processing payment for card: {}", request.getCardNumber()); // PCI violation!
    paymentGateway.charge(request);
}

// ✅ GOOD: Mask sensitive data before logging
public void processPayment(PaymentRequest request) {
    String maskedCard = "****-****-****-" + request.getCardNumber().substring(12);
    log.info("Processing payment for card: {}", maskedCard);
    paymentGateway.charge(request);
}
```

Other security items to check:
- Are passwords stored as plain text anywhere?
- Are JWT tokens validated properly (expiry, signature, claims)?
- Is user-supplied data validated before use (`@Valid`, `@NotNull`, etc.)?
- Are file paths constructed from user input (path traversal risk)?

---

### 4. Maintainability

Code is read far more often than it is written. Maintainability is about making future changes safe and easy.

```java
// ❌ BAD: Magic numbers — what does 86400 mean?
public boolean isSessionExpired(long sessionStartTime) {
    return System.currentTimeMillis() - sessionStartTime > 86400000;
}

// ✅ GOOD: Named constant explains intent
private static final long SESSION_TIMEOUT_MS = TimeUnit.DAYS.toMillis(1);

public boolean isSessionExpired(long sessionStartTime) {
    return System.currentTimeMillis() - sessionStartTime > SESSION_TIMEOUT_MS;
}
```

```java
// ❌ BAD: Method doing too many things (violates Single Responsibility)
public void processOrder(Order order) {
    // Validates the order
    if (order.getItems().isEmpty()) throw new IllegalArgumentException("Empty order");
    
    // Calculates price
    double total = order.getItems().stream().mapToDouble(Item::getPrice).sum();
    order.setTotal(total);
    
    // Saves to database
    orderRepository.save(order);
    
    // Sends confirmation email
    emailService.sendConfirmation(order.getCustomerEmail(), total);
    
    // Updates inventory
    order.getItems().forEach(item -> inventoryService.decrement(item.getId()));
}

// ✅ GOOD: Each concern in its own method, orchestrated by the main method
public void processOrder(Order order) {
    validateOrder(order);
    calculateTotal(order);
    orderRepository.save(order);
    notifyCustomer(order);
    updateInventory(order);
}
```

**Ask yourself:** "If this method broke at 3am, could a new team member understand it quickly enough to fix it?"

---

### 5. Tests

Tests are first-class citizens in a pull request. Missing or weak tests are a valid reason to request changes.

```java
// ❌ BAD: Test only covers the happy path
@Test
void testCalculateDiscount() {
    assertEquals(10.0, discountService.calculate(100.0, "VIP"));
}

// ✅ GOOD: Tests cover happy path, edge cases, and error conditions
@Test
void calculateDiscount_vipCustomer_returns10Percent() {
    assertEquals(10.0, discountService.calculate(100.0, "VIP"));
}

@Test
void calculateDiscount_unknownTier_returnsZero() {
    assertEquals(0.0, discountService.calculate(100.0, "UNKNOWN"));
}

@Test
void calculateDiscount_negativeAmount_throwsIllegalArgumentException() {
    assertThrows(IllegalArgumentException.class, 
        () -> discountService.calculate(-50.0, "VIP"));
}

@Test
void calculateDiscount_nullTier_throwsNullPointerException() {
    assertThrows(NullPointerException.class, 
        () -> discountService.calculate(100.0, null));
}
```

Things to check in test reviews:
- Are tests independent (no shared mutable state between tests)?
- Are mocks being verified, or just set up and ignored?
- Is the test name descriptive enough to understand the failure without reading the body?
- Are assertions meaningful, or just checking that nothing threw an exception?

---

## 🗣️ How to Give Constructive Feedback

The tone and framing of your review matters as much as its technical content. A technically correct review that's harshly worded can damage team trust and discourage open collaboration.

### The Three-Level Framework

Think of comments as falling into three levels:

**Level 1 — Blocking issues (must fix before merge):** Bugs, security vulnerabilities, missing critical tests, or violations of core design principles.

**Level 2 — Suggestions (worth addressing but negotiable):** Performance improvements, better naming, slightly better patterns. Prefix these with "Suggestion:" or "Consider:".

**Level 3 — Discussion / curiosity (non-blocking):** Questions about design, alternatives worth noting, or learning opportunities. Prefix with "Nit:" or "Question:".

```
// ❌ BAD feedback — accusatory, vague
"This is wrong and will cause bugs."

// ✅ GOOD feedback — specific, explains why, offers a solution
"This approach will cause a ConcurrentModificationException if the list is 
modified during iteration. Consider using an Iterator explicitly, or collecting 
results to a new list first. See: https://docs.oracle.com/en/java/api/java.base/java/util/ConcurrentModificationException.html"
```

```
// ❌ BAD feedback — personal attack
"You should know not to use raw types. This is Java 101."

// ✅ GOOD feedback — teaches without condescending
"Suggestion: Using a raw type here (List instead of List<User>) bypasses 
generic type checking and can cause ClassCastException at runtime. 
Specifying the type parameter makes the compiler catch issues earlier."
```

### Praise What's Done Well

Don't only comment on problems. If you see elegant code, a clever use of streams, or a well-thought-out test, say so. This builds the relationship and makes critical feedback easier to receive.

```
// Example positive comment
"Nice use of Optional.ifPresentOrElse() here — much cleaner than the 
nested null checks we had before."
```

---

## 🤝 How to Receive Feedback Graciously

As an author, receiving a review that criticizes your code can sting — especially early in your career. The mindset shift that matters most: **the review is about the code, not about you.**

Practical tips for receiving feedback well:

**Respond to every comment**, even if just to say "Fixed" or "Good point, updated." Silence leaves the reviewer wondering if you saw it.

**Don't defend code emotionally.** If a reviewer misunderstood your intent, explain it calmly. If they're right, acknowledge it and update.

**Ask clarifying questions** rather than arguing: "I did it this way because X — does that concern you? Is there a better approach?"

**Thank reviewers** for thorough reviews. A detailed review is a gift — it represents someone investing time in your growth.

---

## ✅ Review Checklist for Java PRs

Use this as a mental checklist before approving or requesting changes:

**Correctness:** Does the code do what the ticket says? Are edge cases handled (null, empty, boundary values)? Are exceptions meaningful and not swallowed?

**Design:** Does this follow SOLID principles? Is there a simpler way? Does it fit the existing architecture?

**Performance:** Any N+1 query patterns? String concatenation in loops? Unbounded queries or collections?

**Security:** User input validated? No hardcoded secrets? Sensitive data masked in logs?

**Tests:** Are the happy path and failure cases covered? Are tests isolated? Are assertions meaningful?

**Code style:** Consistent naming conventions? No magic numbers? Methods short and focused?

**Documentation:** Are public APIs documented with Javadoc? Is complex logic commented?

---

## 📏 PR Size Guidelines

Smaller PRs lead to better reviews. When a PR exceeds 400–500 lines of changed code, reviewers lose focus, miss bugs, and rubber-stamp changes.

As a lead, encourage your team to:
- Keep PRs under 400 lines of meaningful change (excluding generated code)
- Split features into infrastructure PR + feature PR when possible
- Use feature flags to merge incomplete features safely

---

## 🤖 Automated Checks Before Human Review

Human attention is precious. Automate everything you can before the review reaches a human:

```xml
<!-- pom.xml: Configure Checkstyle for code style enforcement -->
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-checkstyle-plugin</artifactId>
    <version>3.3.0</version>
    <configuration>
        <configLocation>google_checks.xml</configLocation>
        <failsOnError>true</failsOnError>
    </configuration>
</plugin>
```

```yaml
# .github/workflows/pr-checks.yml: Run automated gates on every PR
name: PR Quality Checks
on: [pull_request]
jobs:
  quality:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Set up JDK 21
        uses: actions/setup-java@v3
        with: { java-version: '21', distribution: 'temurin' }
      - name: Run tests and coverage
        run: mvn verify jacoco:report
      - name: SonarCloud analysis
        run: mvn sonar:sonar
```

Automated checks should include: compilation, unit tests, code coverage thresholds (e.g., fail if below 80%), static analysis (SpotBugs, PMD), and code style (Checkstyle). By the time a human reviewer looks at the PR, all of these should already be green.

---

## 💡 Key Takeaways

Code review is a skill that improves with deliberate practice. The best reviewers are thorough but not pedantic, firm about important issues but flexible on style, and always respectful. As a lead, the culture of your team's code reviews will directly reflect your own behavior — review the way you want others to review.

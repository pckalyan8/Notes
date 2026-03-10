# 📘 11.2 — Spring AOP: Aspect-Oriented Programming

AOP solves a specific category of problem that OOP struggles with: behaviour that needs to happen in many places but isn't part of any single object's core responsibility. Understanding AOP will also demystify how Spring's `@Transactional`, `@Cacheable`, `@Secured`, and many other annotations actually work under the hood — they are all implemented as AOP aspects.

---

## 🧠 The Problem AOP Solves — Cross-Cutting Concerns

Imagine you need to add logging around every service method in your application. With OOP, you might add a log statement at the beginning and end of every method. You now have logging code in hundreds of methods — and it's all entangled with your business logic. If the logging format changes, you update hundreds of files. If you forget to add logging to a new method, you silently miss coverage.

This is a **cross-cutting concern**: behaviour that cuts across many unrelated classes and methods. Other examples include performance timing, security checks, transaction management, input validation, auditing, and rate limiting. These concerns don't belong to any one module — they crosscut the entire application.

AOP lets you define this behaviour **once**, in one place (called an **Aspect**), and declaratively apply it to many join points across your code without modifying the target classes at all.

---

## 📐 AOP Core Concepts

Before writing any code, the terminology must be clear — Spring's AOP API uses these terms precisely.

A **Join Point** is any point in the execution of your program where additional behaviour could be inserted. In Spring AOP, join points are always **method executions** — Spring's AOP works at the method call level, not at field access or constructor execution level.

A **Pointcut** is a predicate (expression) that matches a set of join points. You use a pointcut to specify *which* methods the advice should apply to. For example, "all methods in classes annotated with `@Service`" or "all methods named `find*` in the repository package".

An **Advice** is the action the aspect takes at a matched join point. It is the actual code that runs. There are five types of advice (covered below).

An **Aspect** is a class that combines one or more pointcuts with one or more pieces of advice. It is the module that encapsulates a cross-cutting concern.

A **Weaving** is the process of linking aspects to their target objects. Spring uses **proxy-based weaving at runtime**: it creates a proxy object wrapping your bean, and the proxy intercepts method calls to apply the advice before delegating to the real method.

```
                   Proxy (created by Spring)
                  ┌─────────────────────────────┐
Caller ──call()──▶│  Before Advice               │
                  │  ↓                           │
                  │  Target Method (your code)    │
                  │  ↓                           │
                  │  After/AfterReturning Advice  │
                  └─────────────────────────────┘
```

---

## 📦 Maven Setup

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-aop</artifactId>
</dependency>
<!-- Includes spring-aop and aspectjweaver -->
```

You also need to enable AOP in your configuration, though Spring Boot does this automatically when the starter is on the classpath:

```java
@Configuration
@EnableAspectJAutoProxy // Activates @Aspect processing; already active in Spring Boot
public class AopConfig { }
```

---

## ✍️ Writing Aspects — Pointcut Expressions

A pointcut expression tells Spring which methods to intercept. Spring uses the AspectJ pointcut expression language. The most common designator is `execution`, which matches method signatures using a pattern.

The full `execution` syntax is:
```
execution([modifiers] return-type declaring-type.method-name(params) [throws])
```

Every `*` matches any single segment, and `..` matches zero or more packages or parameters.

```java
// Match any public method in any class in any package
execution(public * *(..))

// Match any method starting with "find" in any class
execution(* find*(..))

// Match any method in the UserService class
execution(* com.example.service.UserService.*(..))

// Match any method in any class inside the service package (not sub-packages)
execution(* com.example.service.*.*(..))

// Match any method in the service package AND all sub-packages (..)
execution(* com.example.service..*.*(..))

// Match a specific method with specific parameters
execution(* com.example.service.OrderService.placeOrder(com.example.model.Order))

// Match methods with any single parameter
execution(* com.example.service.*.*(*)  )

// Match methods with no parameters
execution(* com.example.service.*.*())

// Other useful designators:
// within(com.example.service.*) — all methods in the service package
// @annotation(com.example.annotation.Audited) — methods annotated with @Audited
// @within(org.springframework.stereotype.Service) — methods in @Service classes
// bean(orderService) — methods on the bean named "orderService"
```

---

## 💡 The Five Advice Types

Each advice type defines *when* relative to the target method the advice code runs.

### @Before — Runs before the method executes

```java
@Aspect
@Component
public class LoggingAspect {

    private static final Logger log = LoggerFactory.getLogger(LoggingAspect.class);

    // JoinPoint gives you metadata about the intercepted method call
    @Before("execution(* com.example.service..*(..))")
    public void logMethodEntry(JoinPoint joinPoint) {
        String className  = joinPoint.getTarget().getClass().getSimpleName();
        String methodName = joinPoint.getSignature().getName();
        Object[] args     = joinPoint.getArgs();

        log.info("→ Entering {}.{}() with args: {}", className, methodName, Arrays.toString(args));
    }
}
```

### @AfterReturning — Runs after the method returns successfully (not on exception)

```java
@Aspect
@Component
public class AuditAspect {

    private final AuditLogRepository auditRepo;

    public AuditAspect(AuditLogRepository auditRepo) {
        this.auditRepo = auditRepo;
    }

    // "returning" binds the return value to the "result" parameter
    @AfterReturning(
        pointcut  = "execution(* com.example.service.OrderService.placeOrder(..))",
        returning = "result"
    )
    public void auditOrderPlaced(JoinPoint joinPoint, Object result) {
        // "result" holds what the method returned (the saved Order object, etc.)
        log.info("Order placed successfully. Result: {}", result);
        auditRepo.save(new AuditLog("ORDER_PLACED", result.toString()));
    }
}
```

### @AfterThrowing — Runs only if the method throws an exception

```java
@Aspect
@Component
public class ExceptionMonitoringAspect {

    // "throwing" binds the thrown exception to the "ex" parameter
    @AfterThrowing(
        pointcut  = "execution(* com.example.service..*(..))",
        throwing  = "ex"
    )
    public void handleException(JoinPoint joinPoint, Exception ex) {
        String method = joinPoint.getSignature().toShortString();
        log.error("Exception in {}: {}", method, ex.getMessage());
        // Could send to an alerting system, increment a counter metric, etc.
    }
}
```

### @After — Runs after the method, regardless of outcome (like finally)

```java
@Aspect
@Component
public class ResourceCleanupAspect {

    @After("execution(* com.example.service.FileService.*(..))")
    public void releaseResources(JoinPoint joinPoint) {
        // This runs whether the method returned normally or threw an exception
        log.debug("Ensuring resources are released after: {}", joinPoint.getSignature().getName());
    }
}
```

### @Around — Wraps the method; most powerful, full control over execution

`@Around` is the most powerful advice type because you control whether the target method runs at all, can modify the arguments before calling it, can modify the return value, and can handle exceptions however you like.

```java
@Aspect
@Component
public class PerformanceAspect {

    private static final Logger log = LoggerFactory.getLogger(PerformanceAspect.class);

    // ProceedingJoinPoint is a JoinPoint with a proceed() method to actually invoke the target
    @Around("execution(* com.example.service..*(..)) || execution(* com.example.repository..*(..))")
    public Object measureExecutionTime(ProceedingJoinPoint pjp) throws Throwable {
        String method    = pjp.getSignature().toShortString();
        long   startTime = System.currentTimeMillis();

        try {
            // Actually call the target method — MUST call this or the method never runs!
            Object result = pjp.proceed();

            long elapsed = System.currentTimeMillis() - startTime;
            log.info("⏱ {}: {}ms", method, elapsed);

            if (elapsed > 1000) {
                log.warn("⚠ Slow method detected: {} took {}ms", method, elapsed);
            }

            return result; // Must return the result, or the caller gets null

        } catch (Throwable ex) {
            long elapsed = System.currentTimeMillis() - startTime;
            log.error("✗ {}: {}ms — threw {}", method, elapsed, ex.getMessage());
            throw ex; // Re-throw the exception unless you want to swallow it
        }
    }
}
```

---

## 🎯 Reusable Pointcut Definitions

Rather than duplicating pointcut expressions in every advice, define them once in a dedicated class:

```java
// A class that only contains reusable pointcut definitions
@Aspect
@Component
public class AppPointcuts {

    // Define the pointcut once — reference it by its method name elsewhere
    @Pointcut("execution(* com.example.service..*(..))")
    public void serviceLayer() {}

    @Pointcut("execution(* com.example.repository..*(..))")
    public void repositoryLayer() {}

    @Pointcut("@annotation(com.example.annotation.Audited)")
    public void auditedMethods() {}

    // Combine pointcuts with AND (&&), OR (||), NOT (!)
    @Pointcut("serviceLayer() || repositoryLayer()")
    public void applicationLayer() {}
}

// Reference the pointcut by its fully-qualified method name
@Aspect
@Component
public class LoggingAspect {

    @Before("com.example.aop.AppPointcuts.serviceLayer()")
    public void logServiceCall(JoinPoint jp) {
        log.info("Service call: {}", jp.getSignature().getName());
    }

    @Around("com.example.aop.AppPointcuts.applicationLayer()")
    public Object timeAllLayers(ProceedingJoinPoint pjp) throws Throwable {
        // ...
        return pjp.proceed();
    }
}
```

---

## 🏷️ Custom Annotation-Based AOP

One of the most elegant patterns is creating your own annotation and writing an aspect that fires when that annotation is present. This is exactly how `@Transactional`, `@Cacheable`, and `@Secured` are implemented inside Spring itself.

```java
// Step 1: Define the annotation
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME) // Must be RUNTIME so Spring can see it via reflection
@Documented
public @interface RateLimit {
    int requestsPerMinute() default 60;
    String key() default "";
}

// Step 2: Write an aspect that responds to the annotation
@Aspect
@Component
public class RateLimitAspect {

    private final Map<String, AtomicInteger> counters = new ConcurrentHashMap<>();

    // Match methods annotated with @RateLimit
    // "rateLimit" parameter binds the annotation instance so we can read its attributes
    @Around("@annotation(rateLimit)")
    public Object enforceRateLimit(ProceedingJoinPoint pjp, RateLimit rateLimit) throws Throwable {
        String key = buildKey(pjp, rateLimit);
        AtomicInteger counter = counters.computeIfAbsent(key, k -> new AtomicInteger(0));
        int current = counter.incrementAndGet();

        if (current > rateLimit.requestsPerMinute()) {
            throw new RateLimitExceededException(
                "Rate limit of " + rateLimit.requestsPerMinute() + " req/min exceeded for: " + key
            );
        }
        return pjp.proceed();
    }

    private String buildKey(ProceedingJoinPoint pjp, RateLimit rateLimit) {
        if (!rateLimit.key().isEmpty()) return rateLimit.key();
        return pjp.getTarget().getClass().getSimpleName() + "." + pjp.getSignature().getName();
    }
}

// Step 3: Annotate methods that need rate limiting — no AOP knowledge required by the caller
@RestController
public class SearchController {

    @GetMapping("/api/search")
    @RateLimit(requestsPerMinute = 20, key = "search-api")
    public List<Result> search(@RequestParam String query) {
        return searchService.search(query);
    }
}
```

---

## 🔧 JDK Dynamic Proxy vs CGLIB Proxy

Spring AOP uses two proxy mechanisms. When the target bean implements an interface, Spring uses a **JDK dynamic proxy** — an implementation of the interface that intercepts calls and delegates to the real object. When the target is a concrete class with no interfaces, Spring uses a **CGLIB proxy** — a dynamically generated subclass of the target class that overrides methods to inject advice.

```java
@Service
public class OrderService { // No interface

    @Transactional // Works because Spring generates a CGLIB subclass
    public void placeOrder(Order order) { ... }

    public void helper() {
        // ⚠️ CRITICAL AOP PITFALL: Calling this.placeOrder() from within the same class
        // BYPASSES the proxy — the transaction annotation is completely ignored!
        this.placeOrder(new Order()); // "this" refers to the real object, not the proxy
    }
}

// The fix: inject the bean itself and call through the proxy
@Service
public class OrderService {

    @Autowired
    private ApplicationContext context; // Or @Lazy self-injection

    public void helper() {
        // Gets the Spring-managed proxy — @Transactional works correctly
        context.getBean(OrderService.class).placeOrder(new Order());
    }
}
```

This self-invocation problem is the most common AOP-related bug. Any Spring annotation that uses AOP (`@Transactional`, `@Cacheable`, `@Async`, `@Secured`) will be silently bypassed if called from a method within the same class without going through the proxy. The solution is to either extract the method to a separate Spring bean or (carefully) inject the bean into itself.

---

## 🔢 Advice Ordering — When Multiple Aspects Apply

When multiple aspects apply to the same method, their order matters. You control it with `@Order`:

```java
@Aspect
@Component
@Order(1) // Lower number = higher priority = outermost wrapper
public class SecurityAspect {
    @Around("com.example.aop.AppPointcuts.serviceLayer()")
    public Object checkSecurity(ProceedingJoinPoint pjp) throws Throwable {
        // Check auth first...
        return pjp.proceed();
    }
}

@Aspect
@Component
@Order(2) // Runs inside the security aspect
public class TransactionAspect {
    @Around("com.example.aop.AppPointcuts.serviceLayer()")
    public Object manageTransaction(ProceedingJoinPoint pjp) throws Throwable {
        // Begin transaction...
        Object result = pjp.proceed();
        // Commit transaction...
        return result;
    }
}

@Aspect
@Component
@Order(3) // Innermost — runs closest to the actual method
public class LoggingAspect {
    @Around("com.example.aop.AppPointcuts.serviceLayer()")
    public Object log(ProceedingJoinPoint pjp) throws Throwable {
        log.info("Before: " + pjp.getSignature().getName());
        Object r = pjp.proceed();
        log.info("After: "  + pjp.getSignature().getName());
        return r;
    }
}
// Execution order: Security → Transaction → Logging → [METHOD] → Logging → Transaction → Security
```

---

## ✅ Best Practices & Important Points

**Keep pointcut expressions precise.** An overly broad pointcut like `execution(* *(..))` intercepts every single method call in your application — including framework internals. This causes serious performance issues and unexpected behaviour. Always scope your pointcuts to your own packages.

**Use `@Around` only when you truly need the power it provides.** For simple pre/post actions, `@Before` and `@AfterReturning` express intent more clearly and are easier to reason about. Reserve `@Around` for cases where you need to control whether the target method runs, modify arguments, or handle the return value.

**Never forget to call `pjp.proceed()` in `@Around` advice.** If you forget this call, the target method will never execute — your application will silently return `null` from any intercepted method. This is one of the most common AOP bugs.

**Understand the self-invocation limitation.** Any call from within a class to another method in the same class bypasses the proxy and therefore bypasses all AOP advice (including `@Transactional`). If you encounter a `@Transactional` method that doesn't seem to roll back, or a `@Cacheable` method that never caches, this is often the cause.

**Use custom annotations to make AOP usage explicit.** When you write `@RateLimit` or `@Audited` on a method, the developer reading the code knows exactly what behavior to expect. This is far more readable than having a broad pointcut silently intercept methods based on package names.

---

## 🔑 Key Summary

```
Cross-cutting concern  → Behavior needed everywhere but belonging to no single class
Join Point             → Any method execution where advice could be applied
Pointcut               → Expression selecting which join points to intercept
Advice                 → The code that runs at a matched join point
Aspect                 → Class combining pointcuts + advice; annotated with @Aspect
@Before                → Runs before method; cannot prevent execution (throw to prevent)
@AfterReturning        → Runs after successful return; can access return value
@AfterThrowing         → Runs only when method throws; can access exception
@After                 → Runs always (like finally); cannot access return value or exception
@Around                → Full control; must call pjp.proceed(); most powerful and most risky
@Pointcut              → Reusable named expression; reference by method name
Self-invocation        → Calling a proxied method from same class bypasses AOP — critical pitfall
@Order                 → Controls which aspect wraps which when multiple aspects apply
```

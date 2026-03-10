# ☕ Java Phase 11 — Spring Framework Ecosystem
### Complete Guide with Theory, Examples & Best Practices

---

## 📁 Files in This Series

| File | Topic |
|------|-------|
| `01_Spring_Core.md` | IoC Container, Dependency Injection, Bean Lifecycle |
| `02_Spring_AOP.md` | Aspect-Oriented Programming |
| `03_Spring_Boot.md` | Auto-configuration, Starters, Actuator, DevTools |
| `04_Spring_MVC.md` | DispatcherServlet, REST Controllers, Exception Handling |
| `05_Spring_Security.md` | Authentication, Authorization, JWT, OAuth2 |
| `06_Spring_Transactions.md` | @Transactional, Propagation, Isolation |
| `07_Spring_Testing.md` | JUnit 5, MockMvc, Testcontainers, Test Slices |
| `08_Spring_WebFlux.md` | Reactive Programming, Mono, Flux, WebClient |
| `09_Spring_Messaging.md` | Kafka, RabbitMQ, WebSocket, Spring Integration |

---

## 🗺️ Learning Order & Why It Matters

The Spring ecosystem is large, but every part builds on a small number of foundational ideas. Learn them in this order and the rest will feel logical rather than magical.

```
Spring Core (IoC, DI, Beans)
  ↓  Everything else is built on top of the container
Spring AOP
  ↓  Used internally by Transactions, Security, Caching
Spring Boot
  ↓  Packages everything into a runnable app with zero XML
Spring MVC (Web layer)
  ↓  How HTTP requests enter your application
Spring Security
  ↓  Authentication and authorization layer around MVC
Spring Transactions
  ↓  How database operations stay consistent
Spring Testing
  ↓  How to verify everything above actually works
Spring WebFlux
  ↓  The reactive alternative to Spring MVC
Spring Messaging
  ↓  Async communication with Kafka, RabbitMQ, WebSocket
```

## 🔑 The One Idea That Unifies Spring

Every component in the Spring ecosystem is a **Spring bean** managed by the **IoC container**. Once you understand what the container does (create, wire, manage lifecycle), every other module — MVC, Security, JPA, AOP, Messaging — is just a collection of pre-built beans that you configure and extend. This is the single most important concept to internalize before studying anything else.

---

> 💡 **Estimated Time for Phase 11:** 6–8 weeks of dedicated study.
> Build at least one full-stack Spring Boot application (REST API + database + security + tests) to cement every concept.

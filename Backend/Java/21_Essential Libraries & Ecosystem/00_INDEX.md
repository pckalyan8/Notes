# 📚 Phase 21 — Essential Libraries & Ecosystem
### The Java Ecosystem Beyond the Standard Library

---

## 📂 Files in This Guide

| File | Topic | Key Libraries |
|------|-------|--------------|
| `01_Utility_Libraries.md` | General-purpose helpers | Apache Commons, Guava, Lombok, MapStruct, ModelMapper |
| `02_JSON_Serialization.md` | Data serialization formats | Jackson, Gson, JSON-B, Avro, Protobuf, Kryo |
| `03_HTTP_Clients.md` | Making HTTP calls | Java HttpClient, OkHttp, Apache HttpClient 5, Feign, RestTemplate, WebClient |
| `04_Caching_Libraries.md` | In-memory & distributed caching | Caffeine, Ehcache, Redis (Lettuce/Jedis), Hazelcast, Spring Cache |
| `05_Reactive_Libraries.md` | Asynchronous reactive programming | Reactive Streams spec, Project Reactor, RxJava, Mutiny |
| `06_Alternative_Frameworks.md` | Non-Spring application frameworks | Quarkus, Micronaut, Vert.x, Helidon, Dropwizard, Play |

---

## 🧭 How to Use This Guide

Phase 21 is about the broader Java ecosystem — the libraries and frameworks that
surround the standard library and Spring. No production Java application uses
only the JDK; understanding this ecosystem makes you effective immediately when
joining any Java project.

Each file follows the same structure: **why this library exists** (the problem
it solves), **core concepts and API**, **annotated code examples**, and
**best practices** including when to use each option and common pitfalls.

---

## ⚖️ Maven Dependency Reference

All examples in this guide assume Maven. Add these to your `pom.xml`:

```xml
<!-- Utility -->
<dependency>
  <groupId>org.apache.commons</groupId>
  <artifactId>commons-lang3</artifactId>
  <version>3.14.0</version>
</dependency>
<dependency>
  <groupId>com.google.guava</groupId>
  <artifactId>guava</artifactId>
  <version>33.1.0-jre</version>
</dependency>
<dependency>
  <groupId>org.projectlombok</groupId>
  <artifactId>lombok</artifactId>
  <version>1.18.32</version>
  <scope>provided</scope>
</dependency>

<!-- JSON -->
<dependency>
  <groupId>com.fasterxml.jackson.core</groupId>
  <artifactId>jackson-databind</artifactId>
  <version>2.17.0</version>
</dependency>

<!-- Reactive -->
<dependency>
  <groupId>io.projectreactor</groupId>
  <artifactId>reactor-core</artifactId>
  <version>3.6.5</version>
</dependency>
```

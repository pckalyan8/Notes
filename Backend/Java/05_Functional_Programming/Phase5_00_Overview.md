# ⚡ Phase 5 — Functional Programming in Java
### Complete Study Guide Index

---

Java 8 was arguably the most transformative release in Java's history. It introduced a whole new
programming paradigm — **functional programming** — directly into an object-oriented language.
Understanding this phase deeply is what separates junior developers from mid-to-senior Java engineers.

---

## 📂 Files in This Phase

| File | Topic | Key Concepts |
|------|-------|--------------|
| `Phase5_01_Lambda_Expressions.md` | Lambda Expressions | Syntax, method references, effectively final |
| `Phase5_02_Functional_Interfaces.md` | Functional Interfaces | `Function`, `Consumer`, `Supplier`, `Predicate`, composition |
| `Phase5_03_Streams_API.md` | Streams API | Pipelines, intermediate/terminal ops, Collectors, parallel streams |
| `Phase5_04_Comparator_Advanced.md` | Advanced Comparator | Chaining, null-safety, `Comparable` vs `Comparator` |

---

## 🧭 Why Functional Programming in Java?

Before Java 8, if you wanted to pass **behavior** to a method (e.g., "sort these objects by name"),
you had to create an anonymous inner class — verbose and noisy. Functional programming solves this
by allowing you to treat **functions as first-class citizens**: pass them as arguments, return them
from methods, and compose them together.

### The Core Idea

```java
// Before Java 8 — Anonymous inner class (verbose)
List<String> names = Arrays.asList("Charlie", "Alice", "Bob");
Collections.sort(names, new Comparator<String>() {
    @Override
    public int compare(String a, String b) {
        return a.compareTo(b);
    }
});

// Java 8+ — Lambda expression (concise, readable)
Collections.sort(names, (a, b) -> a.compareTo(b));

// Even shorter — method reference
names.sort(String::compareTo);
```

---

## 🔗 How the Topics Connect

```
Functional Interfaces  ──►  Lambda Expressions
        │                          │
        │                          ▼
        └──────────────►  Streams API
                                   │
                                   ▼
                          Comparator (used in sort/min/max)
```

Functional interfaces define the **shape** (signature) of a function.
Lambdas are the **implementation** of that shape.
Streams use both to build **data processing pipelines**.
Comparator is a specific functional interface used for **ordering**.

---

## ⚠️ Study Tips

- Don't skip Functional Interfaces — every lambda relies on one.
- Practice Streams daily; they appear in almost every real Java codebase.
- Type inference can feel magical at first — learn to read the types mentally.
- The best way to learn this phase is to **rewrite loops as stream pipelines** until it feels natural.

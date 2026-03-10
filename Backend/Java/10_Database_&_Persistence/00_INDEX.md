# ☕ Java Phase 10 — Database & Persistence
### Complete Guide with Theory, Examples & Best Practices

---

## 📁 Files in This Series

| File | Topic |
|------|-------|
| `01_JDBC.md` | JDBC — Java Database Connectivity |
| `02_JPA.md` | JPA / Jakarta Persistence API |
| `03_Hibernate.md` | Hibernate ORM (Deep Dive) |
| `04_Spring_Data_JPA.md` | Spring Data JPA |
| `05_NoSQL.md` | NoSQL with Java (MongoDB, Redis, Cassandra, Elasticsearch) |

---

## 🗺️ Learning Order

Follow the files in order — each builds on the previous:

```
JDBC (raw SQL)
  ↓
JPA (abstraction over JDBC)
  ↓
Hibernate (most popular JPA implementation)
  ↓
Spring Data JPA (simplifies Hibernate/JPA further)
  ↓
NoSQL (alternative non-relational databases)
```

## 🔑 Key Insight Before You Start

Understanding **why** each layer exists is more important than memorising APIs:

- **JDBC** is the lowest level — you write raw SQL, manage connections, and parse result sets manually. Powerful but verbose.
- **JPA** is a *specification* (not an implementation) that defines a standard way to map Java objects to database tables.
- **Hibernate** is the most popular *implementation* of JPA. It does the heavy lifting of translating object operations to SQL.
- **Spring Data JPA** is a *convenience layer on top of Hibernate/JPA* that eliminates even more boilerplate.
- **NoSQL** is a completely different paradigm — document, key-value, column, or graph databases — used when relational tables don't fit the data model.

---

> 💡 **Estimated Time for Phase 10:** 4–5 weeks of dedicated study
> Build a small project (e.g., a library management system or an e-commerce backend) that uses all five layers to truly cement the concepts.

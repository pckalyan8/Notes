# Phase 13 — APIs & Web Services (Java)
### Complete Learning Guide

---

## What This Phase Covers

Phase 13 is about how Java applications **communicate with the outside world** — how they expose their functionality to clients and how they consume services from others. This is a cornerstone of modern backend development, and every Java developer working on server-side applications will touch this material daily.

---

## Sections

| File | Topic | Key Skills |
|------|-------|------------|
| `01_REST_API_Design.md` | REST API Design | HTTP verbs, status codes, URI design, versioning, pagination |
| `02_OpenAPI_Swagger.md` | OpenAPI / Swagger | API documentation, code generation, Springdoc |
| `03_GraphQL_Java.md` | GraphQL with Java | Schema, queries, mutations, Spring for GraphQL |
| `04_gRPC_Java.md` | gRPC with Java | Protobuf, service types, stub generation |
| `05_SOAP_WebServices.md` | SOAP Web Services | WSDL, JAX-WS, Apache CXF |

---

## How These Technologies Relate

```
Client Request
      │
      ▼
  ┌─────────────────────────────────────────┐
  │         API / Communication Layer        │
  │                                          │
  │  REST ──── Most common, JSON/HTTP        │
  │  GraphQL ── Flexible querying, one URL   │
  │  gRPC ───── High-performance, binary     │
  │  SOAP ───── Enterprise/legacy, XML       │
  └─────────────────────────────────────────┘
      │
      ▼
  Java Backend (Spring Boot, etc.)
```

---

## Prerequisites

Before diving in, you should be comfortable with:
- HTTP fundamentals (requests, responses, headers)
- Spring Boot basics (Phase 11)
- Jackson / JSON serialization (Phase 21.2)
- Basic understanding of interfaces and annotations

---

## Recommended Study Order

Work through the files **in order**. REST is the foundation everything else is compared against. OpenAPI documents your REST APIs. GraphQL and gRPC are modern alternatives to REST, each solving specific problems REST doesn't handle elegantly. SOAP is important for legacy/enterprise systems.

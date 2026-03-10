# 🧪 Java Phase 12 — Testing
### Complete Guide: From Unit Tests to Code Quality Automation

---

## 📁 Files in This Series

| File | Topic |
|------|-------|
| `01_JUnit5.md` | JUnit 5 (Jupiter) — The Testing Framework |
| `02_Mockito.md` | Mockito — Mocking, Stubbing, Verification |
| `03_Test_Strategy.md` | Test Pyramid, AAA/BDD, Naming, Coverage |
| `04_Testing_Tools.md` | AssertJ, Hamcrest, WireMock, Testcontainers, H2, Awaitility, RestAssured |
| `05_Code_Quality_Tools.md` | JaCoCo, SonarQube, Checkstyle, PMD, SpotBugs, Pitest |

---

## 🧠 The Core Idea Behind Phase 12

The roadmap says "Professional developers write tests" — but that statement only becomes meaningful once you understand *why*. Tests are not bureaucracy or box-ticking. They are the mechanism that lets you change code with confidence. Without them, every refactoring is a gamble and every new feature carries the hidden risk of breaking something you didn't anticipate.

Phase 12 teaches you the entire testing stack: the framework you write tests in (JUnit 5), the library that replaces real collaborators with controllable fakes (Mockito), the philosophy that tells you how many of each kind of test to write (the Test Pyramid), the specialized tools for different testing scenarios (AssertJ, WireMock, Testcontainers, RestAssured), and the automation that measures and enforces quality without human effort (JaCoCo, SonarQube, Pitest).

---

## 🗺️ How These Concepts Connect

```
Test Pyramid (Strategy)
  ↓  tells you what to build and in what proportion
JUnit 5 (Framework)
  ↓  gives you the structure to write those tests
Mockito (Isolation)
  ↓  lets unit tests stay pure by replacing real dependencies
AssertJ / Hamcrest (Readability)
  ↓  makes assertions expressive and error messages meaningful
WireMock / Testcontainers / H2 / Awaitility / RestAssured (Specialized Scenarios)
  ↓  handles edge cases that basic unit tests cannot cover
JaCoCo / SonarQube / Pitest / Checkstyle / PMD / SpotBugs (Quality Gates)
  ↓  automates measurement and enforcement so quality doesn't slip
```

---

> 💡 **Estimated time:** 2–3 weeks.
> The most valuable way to learn this phase is to retrofit tests onto a real project you already own. Finding a bug through a test you just wrote is the experience that makes testing intuition click permanently.

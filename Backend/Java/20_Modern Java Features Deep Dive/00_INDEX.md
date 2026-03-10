# ☕ Phase 20 — Modern Java Features Deep Dive
### Stay Current with the Latest Java (Java 9 → Java 24)

---

## 📂 Files in This Guide

| File | Versions Covered | Key Topics |
|------|-----------------|------------|
| `01_Java9_to_11.md` | Java 9, 10, 11 | JPMS, JShell, `var`, Collection factories, HTTP Client, String APIs |
| `02_Java12_to_16.md` | Java 12–16 | Switch expressions, Text Blocks, Records, Pattern Matching instanceof, Sealed Classes (intro) |
| `03_Java17_to_21.md` | Java 17–21 (LTS) | Sealed Classes stable, Pattern Matching switch, Virtual Threads, Structured Concurrency, Sequenced Collections, Record Patterns |
| `04_Java22_to_24.md` | Java 22–24 | Stream Gatherers, Unnamed Variables, Unnamed Classes, Flexible Constructors, FFM API, Class-file API, Vector API |

---

## 🧭 How to Read This Guide

Java moves on a **6-month release cadence**. Features go through a preview
lifecycle before becoming stable:

```
Incubating  →  Preview (1st)  →  Preview (2nd)  →  Standard (stable)
```

Each file in this guide marks the version where a feature became **stable** and
also notes preview history so you understand why some APIs changed between versions.

LTS (Long-Term Support) releases — Java 11, 17, 21 — are the versions most
production systems run. Everything in between is worth understanding because
those previews became the features you use in production today.

---

## 🗺️ Quick Version Map

| LTS Release | Year | Most Important New Features |
|-------------|------|-----------------------------|
| Java 11 LTS | 2018 | `var`, HTTP Client, improved String/Files APIs |
| Java 17 LTS | 2021 | Records, Sealed Classes, Text Blocks, Pattern instanceof |
| Java 21 LTS | 2023 | Virtual Threads, Pattern switch, Sequenced Collections, Record Patterns |

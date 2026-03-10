# 19.3 — Agile & Project Management for Java Leads

> **Goal:** Understand how Scrum and Kanban work in Java teams, how to estimate and manage technical work, and how to handle technical debt without stalling product delivery.

---

## 🧠 Why Agile Matters for a Technical Lead

Many engineers believe project management is "someone else's job" — the Scrum Master's or Product Manager's domain. But a software lead who understands Agile principles deeply becomes far more effective: they can translate vague requirements into clear technical work, protect the team from unrealistic commitments, advocate for engineering health, and help the whole squad move faster with less friction.

You don't need to manage the process — you need to understand it well enough to participate intelligently.

---

## 🔄 Scrum — The Framework

Scrum is the most widely adopted Agile framework in Java engineering teams. It organizes work into short, fixed-length cycles called **sprints** (typically 1–2 weeks), with explicit events that keep the team aligned and improving.

### The Five Scrum Events

**Sprint Planning** is the meeting at the start of each sprint where the team decides what work to commit to. The product owner presents prioritized backlog items, and the team collectively estimates and selects what's achievable in the sprint. As a tech lead, your job here is twofold: ensure the team doesn't over-commit (velocity is a guide, not a target to maximize), and ensure technical considerations are surfaced — if a story requires a database migration before it can be built, that dependency must be visible.

**Daily Standup** (Daily Scrum) is a 15-minute synchronization where each engineer answers three questions: what did I complete yesterday, what will I do today, and is there anything blocking me? As a lead, watch the standup for signals — if someone says "I'm working on the same thing as yesterday" for two days running, that's a flag to check in privately.

**Sprint Review** is a demo at the end of the sprint where the team shows working software to stakeholders. This is important because it forces the team to actually finish things (not just "90% done") and creates a regular feedback loop with the business.

**Sprint Retrospective** is the team's opportunity to inspect its own process. Three questions drive it: what went well, what could be improved, and what will we commit to changing next sprint? As a lead, bring specific observations — "deployments took 45 minutes twice this sprint and blocked two people" is more actionable than "our deployment process is slow."

**Backlog Refinement** (Grooming) is an ongoing activity where the team breaks down upcoming stories, clarifies requirements, and estimates effort before they land in a sprint. Well-refined stories make sprint planning fast and reduce mid-sprint surprises.

### Sprint Metrics

The metrics that matter most in Scrum are velocity (story points completed per sprint, used for forecasting — not for comparing teams), sprint goal completion rate (did the team achieve the overall goal for the sprint?), and cycle time (how long does a story take from "in progress" to "done"?).

---

## 📋 Kanban — The Alternative

Kanban is a flow-based system that suits teams doing continuous delivery rather than fixed sprints. Instead of committing to a batch of work every two weeks, work flows continuously through stages: **To Do → In Progress → Review → Done**. The key mechanism is the **WIP limit** (Work In Progress limit) — each column has a maximum number of items allowed in it simultaneously.

```
┌─────────────┬─────────────────────┬──────────────────────┬──────────────┐
│   TO DO     │    IN PROGRESS      │      IN REVIEW       │    DONE      │
│  (no limit) │      (WIP: 3)       │       (WIP: 2)       │              │
├─────────────┼─────────────────────┼──────────────────────┼──────────────┤
│ Feature A   │ Feature C  [Alice]  │ Feature D  [Bob]     │ Feature B    │
│ Bug fix 7   │ Bug fix 8  [Carol]  │ Bug fix 6  [Dave]    │ Feature E    │
│ Refactor X  │ Feature F  [Alice]  │                      │              │
│ Feature G   │                     │                      │              │
└─────────────┴─────────────────────┴──────────────────────┴──────────────┘
```

The WIP limit of 3 in "In Progress" means that if all three slots are occupied, no new work can start — the team must first complete something and move it forward. This is counterintuitive but powerful: it prevents the "everyone is busy but nothing ships" failure mode. When the board gets blocked, the whole team focuses on unblocking the bottleneck rather than starting new work.

Kanban teams track **throughput** (items completed per week) and **lead time** (total time from item created to delivered) rather than story points.

---

## 🔢 Story Pointing and Estimation

Story pointing is a way of estimating the relative effort of work items using abstract units (story points) rather than time (hours/days). The most common scale is Fibonacci-like: 1, 2, 3, 5, 8, 13, 21. The key insight is that humans are much better at comparing relative sizes ("this is about twice as big as that") than absolute time estimates ("this will take exactly 6 hours").

### Planning Poker in Practice

Planning Poker is the estimation technique most teams use. Everyone estimates independently and simultaneously reveals their guess — if estimates diverge significantly, the high and low estimators explain their reasoning, and the team re-estimates. Divergence often reveals hidden complexity or assumptions that nobody had surfaced.

```
// A real example from a story refinement session:

Story: "Add pagination to the orders API endpoint"

Dave estimates: 3 (it's just adding a Pageable parameter to the repository)

Alice estimates: 8 (we need to add pagination to the API, update the OpenAPI 
                    docs, handle cursor-based pagination for the mobile clients 
                    who can't use offset pagination at scale, add tests, and 
                    update the frontend integration — this is more than it looks)

After discussion, the team aligns on 8. Without Dave's low estimate, 
Alice's concerns might never have been surfaced.
```

### Common Pitfalls in Estimation

The most dangerous estimation mistake is confusing story points with hours. Points measure complexity and risk, not time. A 5-point story might take 2 hours for an experienced engineer or 8 hours for a junior — that's expected and fine. What points capture is that this story involves moderate complexity and some unknowns.

Another pitfall is "point inflation" — teams that are pressured to hit velocity targets start inflating their estimates to look productive. This makes velocity meaningless as a forecasting tool. Protect your team from this by ensuring stakeholders understand that velocity is a planning tool, not a performance measure.

---

## 💳 Technical Debt Management

Technical debt is the accumulated cost of shortcuts, quick fixes, and architectural compromises made during delivery. Like financial debt, a small amount of it is healthy (sometimes you genuinely need to move fast and clean up later), but when it accumulates, the interest payments — in the form of slow development, frequent bugs, and painful onboarding — become crushing.

### Classifying Technical Debt

Not all technical debt is equal. Before deciding how to address it, classify it:

**Deliberate-reckless debt** is when you knowingly took a shortcut and didn't plan to pay it back ("we'll fix this after launch" — and then never did). This is the most dangerous kind because it often grows uncontrolled.

**Deliberate-prudent debt** is a conscious tradeoff: "we know this architecture won't scale beyond 10x our current load, but we need to ship in two weeks and can revisit this in Q3." This is legitimate technical debt if the tradeoff is explicit and tracked.

**Inadvertent-reckless debt** is code written without enough knowledge or care — a junior developer who didn't know about connection pooling and created a new database connection on every request. This kind is discovered during code review or when something breaks.

**Inadvertent-prudent debt** is when you learn better practices after the code is already written — "Now that I understand Hibernate's N+1 problem, I can see we have this issue in a dozen places." This is normal and should be tracked and addressed systematically.

### How to Track and Communicate Technical Debt

As a lead, your job is to make technical debt visible to the product team without drowning them in detail. Create debt items in your backlog the same way you create features — with a clear description of the current state, the desired state, and the risk of not addressing it.

A useful format is the "Technical Debt Brief":

```markdown
## Tech Debt: Order Service Uses Deprecated Spring Security Configuration

**Current State:** The OrderService uses the deprecated `WebSecurityConfigurerAdapter`
which was removed in Spring Boot 3.x. We're currently on Spring Boot 2.7, which 
still supports it, but our upgrade path is blocked until this is resolved.

**Desired State:** Migrate to the Spring Security 6 `SecurityFilterChain` bean approach.

**Risk of Deferring:** 
- Blocks our Spring Boot 3.x upgrade (high value for virtual threads, native image support)
- The old API receives no security patches from Spring Security 6+
- Each sprint we delay, the migration gets more complex as we add new security rules

**Estimated Effort:** 3 story points  
**Recommended Priority:** Next sprint (P1 — blocking upgrade path)
```

This framing — current state, desired state, risk of deferring — makes technical debt legible to product managers and stakeholders who need to make tradeoff decisions. Saying "we have tech debt" doesn't help anyone decide what to do; saying "this is blocking our upgrade, which we need for the performance improvements we committed to in Q4" does.

### The 20% Rule

Many engineering teams reserve approximately 20% of each sprint's capacity for technical debt and internal improvements. This isn't a magic number, but the principle matters: if every sprint is 100% feature work, the codebase degrades continuously until delivery grinds to a halt. By making debt reduction a regular, planned activity, you avoid the "big rewrite" that emerges when debt becomes unbearable.

---

## 📖 Breaking Down Epics to User Stories to Tasks

One of the most practical skills a tech lead develops is the ability to decompose large, vague features into concrete, shippable stories.

An **epic** is a large body of work too big for a single sprint: "Add real-time notifications to the platform."

A **user story** is a deliverable piece of value from a user's perspective, small enough for one sprint: "As a buyer, I want to receive a browser notification when my order ships, so I don't have to keep checking the status page."

A **task** is a technical step inside a story: "Implement WebSocket endpoint for notification delivery."

```
Epic: Real-time notifications

Story 1: As a buyer, I want a browser notification when my order ships (5 points)
  ├── Task: Design WebSocket message schema for shipping events
  ├── Task: Implement WebSocket handler in Spring
  ├── Task: Connect order service to publish shipping events via Kafka
  ├── Task: Frontend: subscribe to WebSocket and display notification
  └── Task: Integration test: verify end-to-end notification flow

Story 2: As a seller, I want a notification when I receive a new order (3 points)
  ├── Task: Add "new order" event type to Kafka topic
  ├── Task: Implement seller WebSocket subscription filtering by seller ID
  └── Task: Frontend: show notification badge on seller dashboard

Story 3: As a user, I can control which notifications I receive in settings (5 points)
  ├── Task: Design notification preferences schema in DB (migration)
  ├── Task: Build CRUD API for notification preferences
  └── Task: Apply preferences as filter before WebSocket delivery
```

The key principle in decomposition is that every story should be independently valuable and demonstrable. Stories 1 and 2 above could each be shipped without the other — they're not artificially split. Story 3 depends on 1 and 2 but adds a meaningful layer of control. This vertical slice approach (thin end-to-end functionality rather than horizontal layer-by-layer development) means the team always has something to show.

---

## ✅ Definition of Done

The "Definition of Done" (DoD) is a shared agreement about what it means for a story to be complete. Without a clear DoD, "done" means different things to different people — and unfinished work accumulates invisibly.

A strong DoD for a Java backend team typically includes: the code is implemented and peer-reviewed, all existing tests still pass, new unit and integration tests are written for the new behavior, code coverage hasn't decreased, the feature is documented (Javadoc for new public APIs, README updated if necessary), the change has been deployed to a staging environment and tested, and no critical Sonar findings were introduced.

Enforce the DoD as a team standard, not a bureaucratic checklist. When the team consistently meets its DoD, confidence in each release grows and the rate of production incidents falls.

---

## 💡 Key Takeaways

Agile is not a silver bullet — it's a set of principles and practices that help teams deliver value incrementally while adapting to change. Your role as a tech lead is to be a bridge between the engineering realities (how hard something actually is, what risks exist, what debt is accumulating) and the product and business needs (what value must ship, when, and to what quality standard). Understanding Scrum and Kanban deeply lets you play that role effectively and advocate for your team with credibility.

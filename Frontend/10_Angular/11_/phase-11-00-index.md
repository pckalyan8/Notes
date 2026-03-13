# Phase 11 — Angular Intermediate: Complete Study Guide
### From the Frontend Angular Mastery Roadmap

> **How to use these files:** Read each file in order — they are sequenced so that concepts in earlier files support later ones. Each file contains full theory, annotated code examples, and a best practices section at the end.

---

## 📂 Files in This Phase

| File | Topic | Core Concepts Covered |
|---|---|---|
| `phase-11-01-advanced-routing.md` | Advanced Routing (Angular 21) | `loadComponent`, `loadChildren`, guards, resolvers, typed params, nested routes, preloading, view transitions, signal router, `TitleStrategy` |
| `phase-11-02-template-driven-forms.md` | Template-Driven Forms | `FormsModule`, `ngModel`, `NgForm`, built-in validators, CSS state classes, select/radio/checkbox binding, form reset |
| `phase-11-03-reactive-forms.md` | Reactive Forms | `FormControl`, `FormGroup`, `FormArray`, `FormBuilder`, typed forms, custom validators, async validators, `FormRecord`, `ControlValueAccessor` |
| `phase-11-04-signal-based-forms.md` | Signal-Based Forms (Experimental) | `signalForm()`, `signalFormControl()`, reactive validators with `computed()`, async status as signals, comparison with reactive forms |
| `phase-11-05-http-httpclient.md` | HTTP & HttpClient (Angular 21) | `provideHttpClient()`, `withFetch()`, typed requests, `HttpParams`, functional interceptors, `HttpContext`, `httpResource()`, Transfer State |
| `phase-11-06-change-detection.md` | Change Detection | Zone.js, Default vs OnPush strategy, `ChangeDetectorRef`, `NgZone`, async pipe, signals for CD, zoneless Angular |
| `phase-11-07-content-projection.md` | Content Projection | `<ng-content>`, multi-slot `select`, `ng-container`, `ng-template`, `NgTemplateOutlet`, `TemplateRef`, `ViewContainerRef`, compound components |
| `phase-11-08-angular-animations.md` | Angular Animations | `trigger`, `state`, `transition`, `animate()`, `:enter`/`:leave`, `keyframes()`, `group()`, `sequence()`, `stagger()`, `AnimationBuilder`, route animations |
| `phase-11-09-custom-directives.md` | Custom Directives | `@Directive`, `ElementRef`, `Renderer2`, `@HostListener`, `@HostBinding`, `host` property, signal inputs, structural directives, Directive Composition API |
| `phase-11-10-view-transitions.md` | View Transitions API | `withViewTransitions()`, CSS pseudo-elements, `view-transition-name`, shared element animations, `@starting-style`, cross-document transitions, `prefers-reduced-motion` |

---

## 🗺️ How These Topics Relate

A helpful way to understand Phase 11 is to see how its topics cluster into three conceptual groups.

**Data layer:** HTTP & HttpClient (11.5) is how your application fetches and sends data. Reactive Forms (11.3) and Template-Driven Forms (11.2) are how users input data. Signal-Based Forms (11.4) represents the future direction for tying both together with the signal system.

**Rendering and reactivity layer:** Change Detection (11.6) is the foundation that explains how Angular knows when to update the DOM. Content Projection (11.7) explains how components can be flexible containers for external content. Custom Directives (11.9) lets you encapsulate reusable DOM behaviour outside of components.

**Navigation and motion layer:** Advanced Routing (11.1) manages how users move between application states. View Transitions (11.10) provides smooth visual continuity between those states. Angular Animations (11.8) handles in-component choreography.

---

## 🧠 Key Mental Models to Take Away

Understanding Angular at an intermediate level means internalising a few mental models that cut across all ten topics.

**Everything is a reactive graph.** Whether it's signals flowing through computed values, RxJS Observables piped through operators, or form `valueChanges` feeding into validation logic — Angular's intermediate layer is fundamentally about expressing relationships between data over time. Understanding this helps you choose between the different tools (signals vs Observables vs direct callbacks) based on what the relationship looks like.

**The component tree is also a dependency injection tree.** This is why `provideHttpClient()` at the root makes `HttpClient` available everywhere, why guards use `inject()` to access services, and why `ChangeDetectorRef` is scoped to the specific component that injects it.

**Angular separates ownership of content from ownership of rendering.** This is the core insight of content projection — the parent component owns and provides the content, but the child component controls where and how it appears in the layout. Directives extend this by separating DOM behaviour from the component responsible for that DOM.

**Performance is opt-in at intermediate level, default at advanced level.** `OnPush` change detection, `trackBy` (or `track`), `takeUntilDestroyed()`, and avoiding memory leaks are all things you must consciously choose in Phase 11. By Phase 12 (Angular Advanced), these become the automatic defaults of a signals-first, zoneless architecture.

---

## ⚡ Quick Reference: When to Use What

**Template-Driven Forms** — use for login/signup forms, contact forms, and any form where the validation is simple and you want minimal TypeScript code. If the form could be described entirely in HTML, template-driven is often cleaner.

**Reactive Forms** — use when you need to add or remove controls dynamically, when validation involves comparing multiple fields, when you need async validation (API calls), or when you want to write unit tests for your form logic without rendering a component.

**Signal-Based Forms** — evaluate for new projects on Angular 20+ when everything else is signals-based. Do not use in production yet if stability is required.

**`@HostBinding` / `@HostListener` vs `host` property** — prefer the `host` property in the `@Directive` or `@Component` decorator for new code; it collocates all host-related concerns and is more readable. Use decorators only when you need them inside a class method.

**Content projection vs `@Input()`** — use `@Input()` for data (strings, objects, arrays). Use content projection for structure (HTML elements, other components). A card component that displays a title passed via `@Input()` is correct. A card component that renders an arbitrary header passed via `<ng-content select="[card-header]">` is even more flexible and reusable.

**Angular Animations vs View Transitions** — use View Transitions for route-level page transitions and shared element animations between routes. Use Angular Animations for complex in-component animation sequences (staggered lists, multi-state UI elements, animated feedback).

---

## 🔄 Recommended Study Order

If you're studying these topics for the first time, the following order minimises forward dependencies.

Start with **Change Detection (11.6)** because understanding how Angular knows when to update the DOM is foundational to understanding why `OnPush` exists and why the async pipe is so important. Then move to **Content Projection (11.7)**, which teaches you how Angular separates template ownership from rendering. Follow that with **Custom Directives (11.9)**, which builds on the same concepts.

Then study **Template-Driven Forms (11.2)** followed by **Reactive Forms (11.3)** in sequence — the contrast between the two approaches is educational. After understanding stable reactive forms, read **Signal-Based Forms (11.4)** to see the future direction.

Next, study **HTTP & HttpClient (11.5)**, which is the bridge between your Angular forms and your backend API. Then move to **Advanced Routing (11.1)**, which ties navigation guards, resolvers, and typed parameters together with what you now know about services and HTTP.

Finish with **Angular Animations (11.8)** and **View Transitions (11.10)**, which build on your routing knowledge and introduce the visual polish layer of your application.

---

*Phase 11 is part of the Complete Frontend Angular Mastery Roadmap | Angular 21 · TypeScript 5.9 · 2026*

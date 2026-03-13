# Phase 12.2 — Zoneless Angular (Angular 18 Preview → 20 Stable → 21 Recommended)

> **Prerequisites:** Phase 11.5 (Change Detection), Phase 12.1 (Angular Signals)

---

## Table of Contents
1. [What Is "Zoneless" and Why It Matters](#1-what-is-zoneless-and-why-it-matters)
2. [Zone.js — The Full Picture](#2-zonejs--the-full-picture)
3. [Enabling Zoneless Change Detection](#3-enabling-zoneless-change-detection)
4. [How Zoneless Change Detection Works](#4-how-zoneless-change-detection-works)
5. [Removing Zone.js from the Bundle](#5-removing-zonejs-from-the-bundle)
6. [OnPush as the New Default](#6-onpush-as-the-new-default)
7. [Migrating Zone.js-Dependent Code](#7-migrating-zonejs-dependent-code)
8. [Testing Zoneless Components](#8-testing-zoneless-components)
9. [Angular 21: zoneless: true in angular.json](#9-angular-21-zoneless-true-in-angularjson)
10. [Performance Benchmarks](#10-performance-benchmarks)
11. [Third-Party Library Compatibility](#11-third-party-library-compatibility)
12. [Migration Strategy](#12-migration-strategy)
13. [Best Practices](#13-best-practices)

---

## 1. What Is "Zoneless" and Why It Matters

### Zone.js — Necessary but Costly

Zone.js has powered Angular's change detection since 2016. It works by **monkey-patching** JavaScript's async primitives:

```javascript
// What Zone.js does under the hood:
const originalSetTimeout = window.setTimeout;
window.setTimeout = function patchedSetTimeout(fn, delay) {
  return originalSetTimeout(() => {
    fn();                         // Run original callback
    ngZone.checkStable();        // Notify Angular: "maybe something changed"
  }, delay);
};
// Same for: setInterval, Promise, fetch, addEventListener, MutationObserver, ...
```

Every time any async operation completes → Angular runs change detection on the **entire component tree**.

### The Problems with Zone.js

| Problem | Impact |
|---|---|
| **~13kB bundle size** | Larger initial download |
| **Non-standard behavior** | Async/await stack traces are modified |
| **Over-triggering** | CD runs even when nothing Angular-related changed |
| **Hard to debug** | Errors point into zone.js internals |
| **SSR/Edge incompatibility** | Zone.js doesn't work in all environments |
| **Micro-benchmark overhead** | Every async operation has Zone.js overhead |

### Zoneless — The Solution

Zoneless Angular removes Zone.js entirely. Change detection is triggered **only** when:
1. A **signal** value changes (fine-grained)
2. `markForCheck()` is explicitly called
3. An `async` pipe receives a new value

Result: **Precise, zero-overhead change detection**.

---

## 2. Zone.js — The Full Picture

### What Zone.js Patches

Zone.js patches these APIs when imported:
```
Browser APIs: setTimeout, setInterval, clearTimeout, clearInterval
Promises: Promise.then, Promise.catch, Promise.finally
DOM events: addEventListener, removeEventListener
XHR: XMLHttpRequest.prototype.send
Fetch: window.fetch
MutationObserver
IntersectionObserver
ResizeObserver
...and more
```

### The Cost in Practice

```typescript
// This single WebSocket message triggers a full CD cycle for all 500 components:
this.ws.onmessage = (msg) => {
  this.messages.push(msg.data); // Zone.js sees this setState in async context
};

// With zoneless + signals: only components that read messages() re-render
```

---

## 3. Enabling Zoneless Change Detection

### Angular 20+ (Stable API)

```typescript
// main.ts
import { bootstrapApplication } from '@angular/platform-browser';
import { provideZonelessChangeDetection } from '@angular/core';

bootstrapApplication(AppComponent, {
  providers: [
    provideZonelessChangeDetection(),
    // ... other providers
  ],
});
```

> **Note:** Angular 18 had `provideExperimentalZonelessChangeDetection()`. Angular 20 stabilized it to `provideZonelessChangeDetection()`. Use the stable API.

### NgModule-Based (Legacy)

```typescript
@NgModule({
  providers: [
    provideZonelessChangeDetection(),
  ],
})
export class AppModule {}
```

---

## 4. How Zoneless Change Detection Works

Without Zone.js, Angular uses a **signal-driven scheduler**:

```
Signal value changes
    ↓
Angular marks affected components as dirty (via reactive graph)
    ↓
Angular schedules a render microtask (queueMicrotask / scheduler.postTask)
    ↓
Only dirty components are re-rendered
```

### Triggering Mechanisms in Zoneless

```typescript
// 1. Signal mutation (most common)
this.count.set(5);       // → components reading count() re-render

// 2. Explicit mark
inject(ChangeDetectorRef).markForCheck(); // → this component re-renders

// 3. async pipe emitting
// observable$ | async  → marks component for check when observable emits

// 4. output() / EventEmitter — still works, but doesn't trigger CD by itself
//    The parent must use signal or markForCheck to respond
```

### What NO LONGER Triggers CD in Zoneless

```typescript
// These DO NOT trigger CD in zoneless mode:
setTimeout(() => { /* this.value changed */ }, 1000);
fetch('/api/data').then(r => { /* this.data changed */ });
this.someValue = 'new'; // Direct property assignment without signal
```

> **This is the key insight:** In zoneless mode, you MUST use signals (or `markForCheck()`) to notify Angular of changes. Plain property assignments are invisible.

---

## 5. Removing Zone.js from the Bundle

### Step 1: Remove polyfill from angular.json

```json
// angular.json — BEFORE
{
  "projects": {
    "my-app": {
      "architect": {
        "build": {
          "options": {
            "polyfills": ["zone.js"]
          }
        },
        "test": {
          "options": {
            "polyfills": ["zone.js", "zone.js/testing"]
          }
        }
      }
    }
  }
}

// angular.json — AFTER (zoneless)
{
  "projects": {
    "my-app": {
      "architect": {
        "build": {
          "options": {
            "polyfills": []  // Zone.js removed
          }
        },
        "test": {
          "options": {
            "polyfills": []  // Zone.js removed from tests too
          }
        }
      }
    }
  }
}
```

### Step 2: Remove zone.js from package.json (Optional)

```bash
npm uninstall zone.js
```

Or keep it for gradual migration (just don't import it).

### Bundle Size Impact

```
Before: main.js includes zone.js (~13kB gzipped)
After:  zone.js completely absent from bundle

Savings: ~13kB per page load
Additional savings from smaller CD runtime code
```

---

## 6. OnPush as the New Default

In zoneless mode, **all components effectively behave like OnPush**. Angular only re-renders a component when explicitly notified.

### Best Practice: Mark All Components as OnPush

```typescript
@Component({
  changeDetection: ChangeDetectionStrategy.OnPush, // Always add this
  ...
})
export class MyComponent { }
```

### Angular 21: OnPush Becomes Default for Signal Components

Components using `input()` (signal inputs) automatically get OnPush behavior:

```typescript
@Component({
  selector: 'app-product-card',
  // No need to explicitly set ChangeDetectionStrategy.OnPush
  // when using signal inputs — Angular 21 handles this automatically
  template: `<div>{{ product().name }}</div>`,
})
export class ProductCardComponent {
  product = input.required<Product>(); // Signal input → implicit OnPush
}
```

---

## 7. Migrating Zone.js-Dependent Code

### Pattern 1: setTimeout / setInterval → signal + effect

```typescript
// ❌ Zone.js-dependent pattern
@Component({ ... })
export class TimerComponent implements OnInit {
  seconds = 0;

  ngOnInit() {
    setInterval(() => {
      this.seconds++; // Zone.js made this trigger CD
    }, 1000);
  }
}

// ✅ Zoneless pattern
@Component({ ... })
export class TimerComponent {
  seconds = signal(0);

  constructor() {
    effect((onCleanup) => {
      const id = setInterval(() => {
        this.seconds.update(s => s + 1); // Signal mutation triggers CD
      }, 1000);
      onCleanup(() => clearInterval(id));
    });
  }
}
```

### Pattern 2: Direct Property Assignment → Signal

```typescript
// ❌ Zone.js-dependent
@Component({ template: `{{ data }}` })
export class DataComponent {
  data: any;

  loadData() {
    fetch('/api/data')
      .then(r => r.json())
      .then(json => {
        this.data = json; // Zone.js patched fetch → triggers CD
      });
  }
}

// ✅ Zoneless
@Component({ template: `{{ data() | json }}` })
export class DataComponent {
  data = signal<any>(null);

  loadData() {
    fetch('/api/data')
      .then(r => r.json())
      .then(json => this.data.set(json)); // Signal mutation → triggers CD
  }
}

// ✅ Even better — httpResource
productsResource = httpResource('/api/products');
```

### Pattern 3: Async Pipe Still Works

```typescript
// ✅ async pipe works in zoneless — it calls markForCheck() internally
@Component({
  template: `
    @if (user$ | async; as user) {
      <p>{{ user.name }}</p>
    }
  `,
})
export class UserComponent {
  user$ = this.userService.getCurrentUser(); // Observable
}
```

### Pattern 4: NgZone.run() → Signal

```typescript
// ❌ Pattern needed when callbacks were outside zone
ngZone.run(() => {
  this.status = 'connected';
});

// ✅ Zoneless — just use signal
this.status.set('connected');
```

### Pattern 5: Third-Party Callbacks

When a third-party library calls your code in a non-Angular context:

```typescript
// ❌ Zone.js handled this automatically
initGoogleMaps() {
  google.maps.event.addListener(this.marker, 'click', () => {
    this.selectedMarker = marker; // Zone.js triggered CD
  });
}

// ✅ Zoneless — use signal
initGoogleMaps() {
  google.maps.event.addListener(this.marker, 'click', () => {
    this.selectedMarker.set(marker); // Signal triggers CD
  });
}
```

---

## 8. Testing Zoneless Components

### TestBed Setup

```typescript
import { TestBed } from '@angular/core/testing';
import { provideZonelessChangeDetection } from '@angular/core';

describe('CounterComponent', () => {
  beforeEach(async () => {
    await TestBed.configureTestingModule({
      imports: [CounterComponent],
      providers: [
        provideZonelessChangeDetection(), // Required for zoneless component tests
      ],
    }).compileComponents();
  });

  it('should increment count on button click', async () => {
    const fixture = TestBed.createComponent(CounterComponent);
    fixture.detectChanges(); // Initial render

    const button = fixture.nativeElement.querySelector('button');
    button.click();

    // In zoneless, must wait for the scheduler
    await fixture.whenStable();
    fixture.detectChanges();

    expect(fixture.nativeElement.querySelector('p').textContent).toBe('1');
  });
});
```

### Flushing Effects in Tests

```typescript
import { TestBed } from '@angular/core/testing';

it('should sync to localStorage when theme changes', () => {
  const fixture = TestBed.createComponent(ThemeComponent);
  fixture.detectChanges();

  fixture.componentInstance.theme.set('dark');

  // Flush pending effects synchronously
  TestBed.flushEffects();

  expect(localStorage.getItem('theme')).toBe('dark');
});
```

---

## 9. Angular 21: `zoneless: true` in angular.json

Angular 21 supports opting into zoneless at project creation level:

```json
// angular.json
{
  "projects": {
    "my-app": {
      "architect": {
        "build": {
          "options": {
            "zoneless": true
          }
        }
      }
    }
  }
}
```

Or at project creation:
```bash
ng new my-app --zoneless
```

This automatically:
1. Adds `provideZonelessChangeDetection()` to `app.config.ts`
2. Removes `zone.js` from polyfills
3. Configures test setup accordingly

---

## 10. Performance Benchmarks

Real-world performance improvements with Zoneless + Signals:

| Metric | Zone.js | Zoneless + Signals | Improvement |
|---|---|---|---|
| **Bundle size** | +13kB (zone.js) | 0kB | -13kB |
| **Initial CD cycles** | Full tree (500 components) | Only changed (5-10) | ~50× fewer checks |
| **TTI (Time to Interactive)** | Baseline | 30-50% faster | Significant |
| **Runtime memory** | Higher (zone.js overhead) | Lower | 10-20% |
| **Event handler overhead** | Every event patched | Native | Near-zero |

### Angular Team Benchmarks (TodoMVC-style app)

```
Zone.js + Default CD:    ~8ms per interaction
Zone.js + OnPush:        ~3ms per interaction
Zoneless + Signals:      ~0.8ms per interaction
```

---

## 11. Third-Party Library Compatibility

When going zoneless, audit every third-party library:

### ✅ Compatible Libraries
```
@angular/material     ✅ Fully zoneless-compatible (Angular 17+)
@angular/cdk          ✅ Fully compatible
@ngrx/store           ✅ Works (uses markForCheck internally)
@ngrx/signals         ✅ Native signals, ideal for zoneless
rxjs                  ✅ Observable + async pipe works
@angular/forms        ✅ Works with markForCheck
@angular/router       ✅ Works
transloco             ✅ Works
```

### ⚠️ Libraries Requiring Wrapping

```typescript
// Libraries that trigger CD via direct DOM manipulation
// Need NgZone.run() wrapper or signal bridge

// Example: Socket.io client
import { inject, NgZone } from '@angular/core'; // Keep NgZone for bridging

@Injectable({ providedIn: 'root' })
export class SocketService {
  private ngZone = inject(NgZone);
  messages = signal<Message[]>([]);

  connect() {
    const socket = io('http://localhost:3000');

    socket.on('message', (data: Message) => {
      // NgZone.run ensures signal mutation is "seen" by Angular scheduler
      this.ngZone.run(() => {
        this.messages.update(msgs => [...msgs, data]);
      });
    });
  }
}
```

### Checklist for Library Compatibility

```
□ Does the library use direct property assignment?  → Wrap in signal
□ Does the library use callbacks outside Angular?   → Use NgZone.run() or signals
□ Does the library patch async APIs?                → Redundant but harmless
□ Does the library use ChangeDetectorRef?           → Should work (markForCheck is zoneless-safe)
□ Does the library use ApplicationRef.tick()?       → Works in zoneless
```

---

## 12. Migration Strategy

### Recommended Migration Path

```
Step 1: Add OnPush to all components
        (Makes them behave more like zoneless)

Step 2: Replace @Input/@Output with input()/output()
        (Start using signal-based component API)

Step 3: Replace component state with signals
        (this.value = x → this.value.set(x))

Step 4: Replace BehaviorSubject services with signal services
        (Gradually)

Step 5: Add provideZonelessChangeDetection() alongside zone.js
        (Test both modes simultaneously — possible!)

Step 6: Remove zone.js from polyfills
        (After validating all third-party libs)
```

### Running Both Zone.js and Zoneless Simultaneously

```typescript
// During migration, you can have both:
providers: [
  provideZonelessChangeDetection(), // Zoneless scheduler
  // zone.js still in polyfills — acts as a safety net
]

// As you migrate components, zone.js becomes less relevant
// Final step: remove zone.js from polyfills
```

---

## 13. Best Practices

1. **All new Angular 21 projects should use zoneless** — it's the recommended default.
2. **Always pair zoneless with OnPush** on all components.
3. **Use signal-based APIs exclusively** in zoneless apps — `input()`, `output()`, `signal()`, `computed()`, `resource()`.
4. **Test with `provideZonelessChangeDetection()`** in all component tests.
5. **Use `NgZone.run()`** as a bridge for third-party library callbacks that can't be refactored.
6. **Audit third-party libraries** before going fully zoneless — incompatible libs can silently break.
7. **Never use `ApplicationRef.tick()`** to force CD — use signals or `markForCheck()`.
8. **`TestBed.flushEffects()`** is essential in zoneless unit tests for verifying effect-based behavior.

---

> **Summary:** Zoneless Angular is the culmination of years of work to make Angular's change detection as efficient as possible. By removing Zone.js and relying entirely on Signals for state propagation, Angular applications gain ~13kB bundle savings, dramatically fewer CD cycles, and superior runtime performance. Angular 21 makes it the recommended default for new projects — `ng new my-app --zoneless` is the modern starting point.

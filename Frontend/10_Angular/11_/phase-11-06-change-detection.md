# Phase 11.6 — Change Detection

## What Is Change Detection and Why Does It Exist?

Angular's change detection system is the mechanism that keeps your component templates in sync with your TypeScript data. When your data changes — a user types in a field, an HTTP response arrives, a timer fires — Angular needs to know about it and update the DOM accordingly. Change detection is the process of Angular examining your component tree, comparing current data to previously rendered data, and updating the DOM where differences are found.

Understanding change detection is critical because it directly determines how fast your application runs, why some templates feel sluggish, and what the difference is between `Default` and `OnPush` strategies. It is also the foundation for understanding Angular Signals and zoneless Angular.

---

## How Zone.js Patches the World

The traditional Angular change detection relies on **Zone.js**, a library that monkey-patches dozens of browser APIs — `setTimeout`, `setInterval`, `Promise.then()`, `XMLHttpRequest`, DOM event listeners, and more. Every time one of these async operations completes, Zone.js intercepts the completion and signals Angular to run change detection.

Think of Zone.js as a "spy" that watches everything that could possibly change your data. When any async operation completes, Angular reasons: "something might have changed, so let me check the entire component tree."

```typescript
// This code works automatically in Angular with Zone.js:
@Component({
  template: `<p>{{ message }}</p>`,
})
export class ExampleComponent {
  message = 'Hello';

  constructor() {
    // Zone.js intercepts setTimeout, then tells Angular to run CD after it fires
    setTimeout(() => {
      this.message = 'World'; // Angular automatically updates the template
    }, 1000);
  }
}
```

Without Zone.js, you would need to manually tell Angular "I just changed something, please re-render."

---

## The Default Change Detection Strategy

By default, Angular uses `ChangeDetectionStrategy.Default`. This means Angular checks **every component** in the component tree, from root to leaves, on every change detection cycle. Even if a deeply nested component has data that hasn't changed, Angular still checks it.

For small apps, this is fine. For large apps with hundreds of components, this becomes a performance bottleneck. A single button click could trigger Angular to check 200 components even though only 3 of them have changed data.

---

## `ChangeDetectionStrategy.OnPush`

`OnPush` is an optimisation strategy that tells Angular: "Only check this component if something interesting has happened." Specifically, Angular only runs change detection on an `OnPush` component when one of these conditions is met:

**1. An `@Input()` reference changes.** Angular checks if the object reference of an input has changed, not whether its properties changed. This is why immutability matters for OnPush.

**2. An event originates from this component or its children.** A button click, input change, or any DOM event triggers a check of that component and its ancestors.

**3. An Observable subscribed with the `async` pipe emits a new value.** The `async` pipe automatically marks the component for check when a new value arrives.

**4. `ChangeDetectorRef.markForCheck()` is called explicitly.** You can manually tell Angular "this component needs a check."

```typescript
import { Component, ChangeDetectionStrategy, Input } from '@angular/core';

@Component({
  selector: 'app-user-card',
  template: `
    <div>{{ user.name }}</div>
    <div>{{ user.email }}</div>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush,  // The magic line
})
export class UserCardComponent {
  @Input() user!: { name: string; email: string };
  // Angular only re-renders this if the 'user' object reference changes
}
```

```typescript
// In the parent — CORRECT: creates a new object reference → triggers OnPush
this.user = { ...this.user, name: 'Jane' };  // New object

// INCORRECT: mutates existing object → OnPush won't detect the change
this.user.name = 'Jane';  // Same reference → Angular skips the check
```

This is why **immutable data patterns** are fundamental to OnPush performance. When you work with OnPush components, you should never mutate objects or arrays in place; always create new ones.

---

## The Change Detector Reference (`ChangeDetectorRef`)

Sometimes you need to explicitly control change detection. `ChangeDetectorRef` (CDR) gives you that power.

```typescript
import { Component, ChangeDetectionStrategy, ChangeDetectorRef, inject } from '@angular/core';

@Component({
  selector: 'app-live-feed',
  template: `<p>{{ latestMessage }}</p>`,
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class LiveFeedComponent {
  private cdr = inject(ChangeDetectorRef);
  latestMessage = '';

  constructor(private websocketService: WebsocketService) {
    // WebSocket messages don't come through Zone.js events
    // An OnPush component won't detect them automatically
    this.websocketService.messages$.subscribe(msg => {
      this.latestMessage = msg;
      this.cdr.markForCheck();  // Tell Angular "please check this component next cycle"
    });
  }
}
```

The four key `ChangeDetectorRef` methods are these. `markForCheck()` marks the component and all its ancestors to be checked in the next CD cycle — it schedules a check without triggering one immediately. `detectChanges()` immediately runs change detection for this component and its children — synchronous and immediate, useful for critical UI updates. `detach()` completely removes the component from the CD tree — no automatic checks will occur; you must call `detectChanges()` manually. `reattach()` puts the component back into the CD tree.

```typescript
// Detaching and manually controlling a high-frequency component
@Component({
  selector: 'app-live-chart',
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class LiveChartComponent implements OnInit, OnDestroy {
  private cdr = inject(ChangeDetectorRef);
  dataPoints: number[] = [];

  ngOnInit() {
    // Completely detach from automatic CD
    this.cdr.detach();

    // Only update the DOM at 60fps using requestAnimationFrame
    const interval = setInterval(() => {
      this.dataPoints = [...this.dataPoints, Math.random()];
      this.cdr.detectChanges();  // Manually trigger CD
    }, 1000 / 60);
  }
}
```

---

## `NgZone` — Controlling Zone.js Boundaries

`NgZone` gives you the ability to run code either inside or outside Angular's zone. Running code **outside the zone** means Zone.js won't intercept it, so no change detection cycle fires. This is critical for performance when you have high-frequency operations like `requestAnimationFrame` loops, WebSocket streams, or D3 chart updates.

```typescript
import { Component, inject, NgZone } from '@angular/core';

@Component({
  selector: 'app-animation',
  template: `<canvas #canvas></canvas>`,
})
export class AnimationComponent implements OnInit {
  private ngZone = inject(NgZone);

  ngOnInit() {
    // Running this OUTSIDE the Angular zone prevents CD from firing 60 times/second
    // This is a massive performance win for animation loops
    this.ngZone.runOutsideAngular(() => {
      const render = () => {
        this.updateCanvas();  // DOM manipulation that doesn't need Angular's CD
        requestAnimationFrame(render);
      };
      requestAnimationFrame(render);
    });
  }

  // When you DO need Angular to update the template, explicitly run inside the zone
  updateUserScore(newScore: number) {
    this.ngZone.run(() => {
      this.userScore = newScore;  // This WILL trigger change detection
    });
  }
}
```

---

## The `async` Pipe and OnPush

The `async` pipe is the most elegant way to use Observables in templates. It subscribes to an Observable, renders its values, and automatically marks an `OnPush` component for checking whenever a new value arrives. It also automatically unsubscribes when the component is destroyed.

```typescript
@Component({
  selector: 'app-user-list',
  template: `
    <!-- async pipe handles subscribe, unsubscribe, AND markForCheck() automatically -->
    @if (users$ | async; as users) {
      @for (user of users; track user.id) {
        <app-user-card [user]="user" />
      }
    }

    <!-- Loading and error states -->
    @if (!(users$ | async)) {
      <p>Loading users...</p>
    }
  `,
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class UserListComponent {
  users$ = inject(UsersService).getUsers();
  // No subscribe(), no unsubscribe(), no markForCheck() needed!
}
```

---

## Change Detection and Angular Signals

Signals represent the future of Angular change detection and are designed to make fine-grained updates possible without Zone.js. When a component reads a signal in its template, Angular creates a dependency on that signal. When the signal's value changes, **only the components that read that specific signal** are re-rendered — not the entire component tree.

```typescript
import { Component, signal, computed } from '@angular/core';

@Component({
  selector: 'app-counter',
  template: `
    <!-- Angular tracks that this template reads 'count' and 'doubled' -->
    <p>Count: {{ count() }}</p>
    <p>Doubled: {{ doubled() }}</p>
    <button (click)="increment()">+</button>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush,  // Signals work best with OnPush
})
export class CounterComponent {
  count = signal(0);
  doubled = computed(() => this.count() * 2);

  increment() {
    this.count.update(c => c + 1);
    // Angular knows 'count' changed.
    // It automatically marks only this component (and components that read count) for check.
    // No Zone.js, no markForCheck() needed.
  }
}
```

Signals and `OnPush` together form the recommended pattern for high-performance Angular applications. Signal changes are tracked with surgical precision — Angular updates exactly the parts of the DOM that depend on the changed signal, nothing more.

---

## Zoneless Angular (`provideZonelessChangeDetection`)

Angular 20 stabilised **zoneless change detection**. When you opt into zoneless mode, Zone.js is completely removed from your application — saving ~13KB of bundle size and eliminating the performance overhead of Zone.js's async interception.

In zoneless mode, Angular relies entirely on explicit signals to know when to check components. This means your entire application must use signals (or the `async` pipe with `markForCheck`) to trigger re-renders.

```typescript
// app.config.ts — enables zoneless change detection
import { provideZonelessChangeDetection } from '@angular/core';

export const appConfig: ApplicationConfig = {
  providers: [
    provideZonelessChangeDetection(),
    // Remove zone.js from angular.json polyfills too
    provideRouter(routes),
  ],
};
```

```json
// angular.json — remove Zone.js from polyfills
{
  "polyfills": []  // Previously: ["zone.js"]
}
```

In zoneless mode, `setTimeout` and `setInterval` no longer automatically trigger change detection. You must explicitly update a signal or call `markForCheck()` whenever you want the template to update after an async operation.

---

## Change Detection in Practice — Full Example

Here is how these concepts come together in a production-grade component that uses `OnPush`, signals, and the async pipe efficiently.

```typescript
import { Component, ChangeDetectionStrategy, signal, computed, inject } from '@angular/core';
import { AsyncPipe } from '@angular/common';
import { UsersService } from './users.service';

@Component({
  selector: 'app-user-dashboard',
  standalone: true,
  imports: [AsyncPipe],
  template: `
    <div class="filters">
      <input
        type="text"
        (input)="searchTerm.set($event.target.value)"
        placeholder="Search users..."
      />
      <p>Showing {{ filteredCount() }} of {{ totalCount() }} users</p>
    </div>

    @if (users$ | async; as allUsers) {
      @for (user of filtered(allUsers); track user.id) {
        <app-user-row [user]="user" />
      }
    }
  `,
  changeDetection: ChangeDetectionStrategy.OnPush,  // Only re-check when signals or async pipe fire
})
export class UserDashboardComponent {
  private usersService = inject(UsersService);

  users$ = this.usersService.getUsers();  // Observable — handled by async pipe

  searchTerm = signal('');     // A signal for the search input

  // computed() that derives a filtered count — always in sync with searchTerm
  filteredCount = computed(() => /* filtered list length based on searchTerm */ 0);
  totalCount = computed(() => /* total list length */ 0);

  filtered(users: User[]) {
    // Filter based on the current signal value
    const term = this.searchTerm().toLowerCase();
    return term ? users.filter(u => u.name.toLowerCase().includes(term)) : users;
  }
}
```

---

## Key Best Practices

Apply `ChangeDetectionStrategy.OnPush` to every component in your application as a default — this is the single most impactful performance decision you can make in an Angular app. When using `OnPush`, never mutate objects or arrays directly; always create new references so Angular can detect the change. Use the `async` pipe in templates instead of subscribing manually in the component class — it correctly handles `markForCheck()` for `OnPush` components and eliminates memory leaks. Move high-frequency operations (animation frames, WebSocket streams, D3 updates) outside the Angular zone with `runOutsideAngular()` and only bring them back inside when you need the template to update. If you're building a greenfield application in Angular 20/21, strongly consider going fully zoneless with signals — the performance gains are significant and the code model is cleaner. Think of change detection as a tree traversal algorithm; the goal is to minimise how many branches of the tree get traversed on each cycle.

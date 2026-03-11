# Phase 10 — Angular Core Fundamentals
## Part 3: Components & Component Lifecycle Hooks
> Topics: 10.6 · 10.7

---

## 10.6 Components

### What is a Component?

A **component** is the most fundamental building block of an Angular application. Every visible piece of your UI is a component: a navigation bar, a user card, a modal dialog, a login form, and even the root `AppComponent` that wraps everything.

Each component is a TypeScript class decorated with `@Component` that ties together three things:
- **Logic** — the TypeScript class with properties and methods
- **Template** — the HTML that defines what the component renders
- **Styles** — the CSS/SCSS scoped to this component

Think of each component as a custom HTML element with its own behavior. When Angular encounters your selector in a template, it stamps out that component's template and wires up its logic.

### The `@Component` Decorator

```typescript
import { Component, Input, Output, EventEmitter, OnInit } from '@angular/core';
import { CommonModule } from '@angular/common';

@Component({
  // 'selector' defines the HTML tag used to embed this component.
  // Element selectors (app-*) are the most common and recommended.
  selector: 'app-user-card',

  standalone: true,

  // 'templateUrl' points to a separate HTML file.
  // Use 'template' (inline) for very small templates only.
  templateUrl: './user-card.component.html',

  // 'styleUrls' points to separate style files.
  // Use 'styles' (inline) for tiny style snippets only.
  styleUrls: ['./user-card.component.scss'],

  // Imports for standalone component — other components, directives, pipes it needs.
  imports: [CommonModule]
})
export class UserCardComponent implements OnInit {
  // Component logic goes here
}
```

### The Three Types of Component Selectors

Angular supports three selector patterns, though **element selectors are strongly preferred** for components.

**Element selector** (recommended for components):
```html
<!-- Selector: 'app-user-card' -->
<app-user-card [user]="currentUser" />
```

**Attribute selector** (best for directives and structural wrappers):
```typescript
// Selector: '[app-highlight-card]'
// Used when you want to apply a component's template to an existing element.
@Component({ selector: '[app-highlight-card]', ... })
```
```html
<div app-highlight-card>...</div>
```

**Class selector** (rarely used):
```typescript
// Selector: '.app-user-card'
@Component({ selector: '.app-user-card', ... })
```
```html
<div class="app-user-card"></div>
```

### Component Inputs: `@Input()` and `input()`

Inputs are the way a parent component passes data down to a child component. Angular 16+ introduced **signal-based inputs** using `input()`, which is the modern preferred approach.

**Classic `@Input()` decorator approach:**
```typescript
import { Component, Input } from '@angular/core';

@Component({ selector: 'app-user-card', standalone: true, template: `
  <div>{{ user.name }}</div>
` })
export class UserCardComponent {
  // The @Input() decorator marks this property as an input.
  // The parent binds to it with [user]="someUser"
  @Input() user!: { id: number; name: string; email: string };

  // You can alias inputs so the property name differs from the binding name
  @Input('userId') id!: number;

  // Required inputs (Angular 16+) — Angular will error if parent doesn't provide it
  @Input({ required: true }) title!: string;
}
```

**Modern signal-based `input()` approach (Angular 17+, recommended):**
```typescript
import { Component, input } from '@angular/core';

@Component({ selector: 'app-user-card', standalone: true, template: `
  <!-- Access signal inputs by calling them as functions: user() -->
  <div>{{ user().name }}</div>
  <div>{{ title() }}</div>
` })
export class UserCardComponent {
  // input() returns a Signal<T | undefined> — value might not be provided
  user = input<{ id: number; name: string }>();

  // input.required<T>() — Angular errors if parent doesn't provide this
  title = input.required<string>();

  // Default values
  isActive = input<boolean>(false);
}
```

Signal inputs are preferred because they integrate with Angular's fine-grained change detection — a component only re-renders when the signal's value actually changes, without needing Zone.js.

**Parent component binding syntax (same for both approaches):**
```html
<app-user-card
  [user]="selectedUser"
  [title]="'Profile Details'"
  [isActive]="true"
/>
```

### Component Outputs: `@Output()` and `output()`

Outputs let a child component communicate events back up to its parent.

**Classic `@Output()` with EventEmitter:**
```typescript
import { Component, Output, EventEmitter } from '@angular/core';

@Component({ selector: 'app-user-card', standalone: true, template: `
  <button (click)="onDeleteClick()">Delete</button>
` })
export class UserCardComponent {
  // The output() emits a value of type number upward to the parent
  @Output() userDeleted = new EventEmitter<number>();

  onDeleteClick() {
    this.userDeleted.emit(this.user.id);
  }
}
```

**Modern signal-based `output()` approach (Angular 17+, recommended):**
```typescript
import { Component, output, input } from '@angular/core';

@Component({ selector: 'app-user-card', standalone: true, template: `
  <button (click)="onDeleteClick()">Delete</button>
` })
export class UserCardComponent {
  userId = input.required<number>();

  // output() returns an OutputEmitterRef — emit values with .emit()
  userDeleted = output<number>();

  onDeleteClick() {
    this.userDeleted.emit(this.userId());
  }
}
```

**Parent template listens with event binding:**
```html
<app-user-card
  [userId]="user.id"
  (userDeleted)="handleUserDeleted($event)"
/>
```

### The `model()` Signal — Two-Way Binding

Angular 17 introduced `model()` for two-way binding — the signal equivalent of combining `@Input` and `@Output` with matching names:

```typescript
// child.component.ts
import { Component, model } from '@angular/core';

@Component({ selector: 'app-counter', standalone: true, template: `
  <button (click)="decrement()">-</button>
  <span>{{ count() }}</span>
  <button (click)="increment()">+</button>
` })
export class CounterComponent {
  // model() creates a writable signal AND an output named 'countChange'
  // Angular uses these together for [(count)] two-way binding in the parent
  count = model<number>(0);

  increment() { this.count.update(v => v + 1); }
  decrement() { this.count.update(v => v - 1); }
}
```

```html
<!-- Parent can use two-way binding with the banana-in-a-box syntax -->
<app-counter [(count)]="myValue" />
```

### `ViewChild` and `ViewChildren`

`ViewChild` gives you a reference to a child component or DOM element from within the parent's TypeScript class. The modern signal-based `viewChild()` function is preferred.

```typescript
import { Component, viewChild, ElementRef, afterNextRender } from '@angular/core';

@Component({ standalone: true, selector: 'app-form', template: `
  <input #emailInput type="email" placeholder="Enter email">
  <app-user-card #cardRef [title]="'Details'" />
` })
export class FormComponent {
  // viewChild() returns a Signal<ElementRef | undefined>
  // The signal becomes defined after the view is initialized
  emailInput = viewChild<ElementRef>('emailInput');

  // Query by component type instead of template variable
  userCard = viewChild(UserCardComponent);

  constructor() {
    // afterNextRender is the safe hook for DOM manipulation
    afterNextRender(() => {
      // Now we can safely read DOM properties
      this.emailInput()?.nativeElement.focus();
    });
  }
}
```

`viewChildren()` returns a signal with an array of all matching elements/components:

```typescript
import { viewChildren, QueryList } from '@angular/core';

listItems = viewChildren<ElementRef>('item');
// This gives a Signal<readonly ElementRef[]>
```

### `ContentChild` and `ContentChildren`

While `ViewChild` queries a component's own template, `ContentChild` queries content that was **projected into it from the outside** (via `<ng-content>`):

```typescript
@Component({ selector: 'app-card', standalone: true, template: `
  <div class="card">
    <ng-content select="[card-header]"></ng-content>
    <ng-content></ng-content>
  </div>
` })
export class CardComponent {
  // Signal-based content queries (Angular 17+)
  header = contentChild<ElementRef>('[card-header]');
  allChildren = contentChildren(SomeComponent);
}
```

### The Host Element and `host` Property

The "host element" is the actual DOM element that Angular creates for your component — the `<app-user-card>` element in the DOM. You can bind properties and events to it:

```typescript
@Component({
  selector: 'app-user-card',
  standalone: true,
  // The host property lets you bind to the component's own element
  host: {
    // Add CSS class to the host element
    '[class.active]': 'isActive',
    // Bind an attribute
    '[attr.role]': '"article"',
    // Listen for a click event on the host element
    '(click)': 'onClick()'
  },
  template: `<div>{{ title }}</div>`
})
export class UserCardComponent {
  isActive = input<boolean>(false);
  onClick() { console.log('Card clicked!'); }
}
```

---

## 10.7 Component Lifecycle Hooks

Angular manages every component's **lifecycle** — from creation to destruction. At each key moment, Angular calls specific methods on your component if you implement them. These are the **lifecycle hooks**.

Understanding the lifecycle is essential because it determines *when* it's safe to access DOM elements, set up subscriptions, perform side effects, and clean up resources.

### The Complete Lifecycle Sequence

Here is every lifecycle hook in the exact order Angular calls them:

```
Component instantiation
        ↓
1. ngOnChanges()        ← Called when @Input properties change (BEFORE ngOnInit)
        ↓
2. ngOnInit()           ← Component initialized; all inputs set for the first time
        ↓
3. ngDoCheck()          ← Every change detection cycle (careful: expensive!)
        ↓
4. ngAfterContentInit() ← After <ng-content> projected content is initialized
        ↓
5. ngAfterContentChecked() ← After projected content is checked (every cycle)
        ↓
6. ngAfterViewInit()    ← After component's own view (and children) are initialized
        ↓
7. ngAfterViewChecked() ← After view is checked (every cycle; expensive!)
        ↓
       ... (change detection cycles repeat hooks 3, 5, 7) ...
        ↓
8. ngOnDestroy()        ← Just before component is destroyed; do cleanup here!
```

### `ngOnChanges` — Reacting to Input Changes

`ngOnChanges` is called before `ngOnInit` and every time a bound input property changes. It receives a `SimpleChanges` object containing the old and new values.

```typescript
import { Component, Input, OnChanges, SimpleChanges } from '@angular/core';

@Component({ selector: 'app-user-card', standalone: true, template: `<div>{{ user?.name }}</div>` })
export class UserCardComponent implements OnChanges {
  @Input() user!: { name: string; id: number };

  ngOnChanges(changes: SimpleChanges): void {
    // 'changes' is a dictionary keyed by input property name
    if (changes['user']) {
      const current = changes['user'].currentValue;
      const previous = changes['user'].previousValue;
      const isFirstChange = changes['user'].firstChange;

      console.log(`User changed from`, previous, `to`, current);

      if (!isFirstChange) {
        // React to user changing — e.g., re-fetch related data
        this.loadUserDetails(current.id);
      }
    }
  }

  private loadUserDetails(id: number) { /* ... */ }
}
```

**Important note about signal inputs:** If you use `input()` signals instead of `@Input()`, `ngOnChanges` is **not called** — signals have their own reactivity model. You react to signal changes using `effect()` or `computed()` instead.

### `ngOnInit` — Component Initialization

`ngOnInit` is called **once**, after Angular sets all input properties for the first time. This is the right place to:
- Fetch initial data from a service
- Initialize complex computed values based on inputs
- Set up subscriptions (but also see `ngOnDestroy` for cleanup)

```typescript
import { Component, OnInit, Input } from '@angular/core';
import { UserService } from '../services/user.service';
import { User } from '../models/user.model';

@Component({ selector: 'app-user-profile', standalone: true, template: `
  @if (user) {
    <h2>{{ user.name }}</h2>
    <p>{{ user.bio }}</p>
  } @else {
    <p>Loading...</p>
  }
` })
export class UserProfileComponent implements OnInit {
  @Input() userId!: number;

  user: User | null = null;
  isLoading = true;

  constructor(private userService: UserService) {
    // The constructor is for dependency injection ONLY.
    // Do NOT do anything that could fail or require inputs here,
    // because inputs are NOT yet set when the constructor runs.
  }

  ngOnInit(): void {
    // Inputs are guaranteed to be set now. Safe to use this.userId.
    this.userService.getUser(this.userId).subscribe(user => {
      this.user = user;
      this.isLoading = false;
    });
  }
}
```

**A critical rule:** Use the **constructor only for dependency injection**. Never call services, access inputs, or touch the DOM in the constructor — inputs haven't been set, and DOM doesn't exist yet. `ngOnInit` is where initialization logic belongs.

### `ngDoCheck` — Custom Change Detection

`ngDoCheck` runs on **every single change detection cycle** — potentially hundreds of times per second in active applications. It's designed for components that need to detect changes Angular cannot detect automatically (like mutations deep inside an object).

```typescript
import { Component, Input, DoCheck, IterableDiffer, IterableDiffers } from '@angular/core';

@Component({ selector: 'app-list', standalone: true, template: `<ul>
  @for (item of items; track item) { <li>{{ item }}</li> }
</ul>` })
export class ListComponent implements DoCheck {
  @Input() items: string[] = [];

  // IterableDiffer can detect array mutations (push, pop, splice) that Angular's
  // default change detection misses since the array reference doesn't change.
  private differ: any;

  constructor(private differs: IterableDiffers) {
    this.differ = this.differs.find([]).create();
  }

  ngDoCheck(): void {
    const changes = this.differ.diff(this.items);
    if (changes) {
      console.log('Array was mutated:', changes);
    }
  }
}
```

**Best practice: Avoid `ngDoCheck` unless absolutely necessary.** It runs every cycle and is expensive. Most cases are better handled with immutable data patterns or signals.

### `ngAfterContentInit` and `ngAfterContentChecked`

These hooks are called after Angular initializes content projected via `<ng-content>`. `ngAfterContentInit` fires once; `ngAfterContentChecked` fires on every change detection cycle after that.

```typescript
import { Component, ContentChild, AfterContentInit, ElementRef } from '@angular/core';

@Component({ selector: 'app-card', standalone: true, template: `
  <div class="card-wrapper">
    <ng-content select="[card-icon]"></ng-content>
    <ng-content></ng-content>
  </div>
` })
export class CardComponent implements AfterContentInit {
  @ContentChild('cardIcon') icon!: ElementRef;

  ngAfterContentInit(): void {
    // Safe to access ContentChild/ContentChildren references HERE.
    // They are not available in ngOnInit.
    if (this.icon) {
      console.log('Card has an icon:', this.icon.nativeElement);
    }
  }
}
```

### `ngAfterViewInit` — After View Is Ready

`ngAfterViewInit` fires **once** after Angular fully renders the component's own view AND all child component views. This is the first moment you can safely access `ViewChild` and `ViewChildren` references.

```typescript
import { Component, ViewChild, AfterViewInit, ElementRef } from '@angular/core';

@Component({ selector: 'app-chart', standalone: true, template: `
  <canvas #chartCanvas></canvas>
` })
export class ChartComponent implements AfterViewInit {
  @ViewChild('chartCanvas') canvas!: ElementRef<HTMLCanvasElement>;

  ngAfterViewInit(): void {
    // 'canvas' is guaranteed to be defined here.
    // This is the right place to integrate third-party libraries that need DOM elements.
    const ctx = this.canvas.nativeElement.getContext('2d');
    this.initializeChart(ctx!);
  }

  private initializeChart(ctx: CanvasRenderingContext2D) { /* ... */ }
}
```

**Important:** Do not modify component data in `ngAfterViewInit` or `ngAfterViewChecked` — doing so triggers a second change detection pass, which will cause the dreaded `ExpressionChangedAfterItHasBeenCheckedError` in development mode.

### `ngOnDestroy` — Cleanup

`ngOnDestroy` is called just before Angular removes a component from the DOM. This is where you **must** clean up to avoid memory leaks:
- Unsubscribe from Observables
- Clear timers and intervals
- Unregister event listeners
- Cancel pending HTTP requests

```typescript
import { Component, OnInit, OnDestroy } from '@angular/core';
import { Subscription, interval } from 'rxjs';
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';

@Component({ selector: 'app-timer', standalone: true, template: `<p>Count: {{ count }}</p>` })
export class TimerComponent implements OnInit, OnDestroy {
  count = 0;
  private timerSubscription?: Subscription;

  // --- Option 1: Manual cleanup with ngOnDestroy ---
  ngOnInit(): void {
    this.timerSubscription = interval(1000).subscribe(() => {
      this.count++;
    });
  }

  ngOnDestroy(): void {
    // Unsubscribing is CRITICAL. If you don't, the subscription continues
    // running even after the component is destroyed — a classic memory leak.
    this.timerSubscription?.unsubscribe();
  }
}

// --- Option 2: Modern approach with takeUntilDestroyed (Angular 16+) ---
// This is cleaner because you don't need ngOnDestroy at all.
@Component({ selector: 'app-timer-modern', standalone: true, template: `<p>Count: {{ count }}</p>` })
export class TimerModernComponent {
  count = 0;

  constructor() {
    // takeUntilDestroyed() automatically completes the Observable when the
    // component's DestroyRef is triggered — no manual unsubscribe needed.
    interval(1000)
      .pipe(takeUntilDestroyed())
      .subscribe(() => { this.count++; });
  }
}
```

### Lifecycle Hooks with Signal-Based Components

When using signal inputs (`input()`, `model()`) and signal-based queries, the lifecycle shifts slightly. Signals react to value changes automatically — you use `effect()` to run side effects when signals change, rather than `ngOnChanges`:

```typescript
import { Component, input, effect, OnInit } from '@angular/core';

@Component({ selector: 'app-signal-demo', standalone: true, template: `
  <p>User: {{ userId() }}</p>
` })
export class SignalDemoComponent implements OnInit {
  userId = input.required<number>();

  constructor() {
    // effect() runs whenever the signals it reads change.
    // This replaces ngOnChanges for signal inputs.
    effect(() => {
      const id = this.userId();  // Reading the signal inside effect() creates a dependency
      console.log('userId changed to:', id);
      // Fetch new data, update derived state, etc.
    });
  }

  ngOnInit(): void {
    // ngOnInit still works fine alongside signals for initialization logic
    // that doesn't depend on signal values.
  }
}
```

---

## Best Practices for 10.6–10.7

**Always implement the interface.** If you implement `ngOnInit`, add `implements OnInit` to the class declaration. TypeScript will then catch typos (like `ngOnlnit` — lowercase L) that would otherwise silently not be called by Angular.

**Prefer `takeUntilDestroyed()` over manual `ngOnDestroy` subscriptions.** The `takeUntilDestroyed()` operator from `@angular/core/rxjs-interop` (Angular 16+) auto-unsubscribes when the component is destroyed. It results in cleaner, less error-prone code.

**Never use `ngAfterViewChecked` or `ngAfterContentChecked` to update data.** These hooks run on every change detection cycle. Any state mutation inside them triggers another cycle, creating an infinite loop or the `ExpressionChangedAfterItHasBeenCheckedError`.

**Use `afterNextRender` for DOM manipulation.** The newer `afterNextRender()` function (Angular 17+) is preferred over `ngAfterViewInit` for DOM access because it is SSR-safe — it only runs in the browser, never on the server.

**Use `DestroyRef` for imperative cleanup.** If you need to clean up resources that aren't subscriptions (like third-party library instances), Angular 16+ provides `DestroyRef` via injection:

```typescript
constructor(private destroyRef: DestroyRef) {
  destroyRef.onDestroy(() => {
    // Clean up anything here — runs just like ngOnDestroy
    thirdPartyLibInstance.cleanup();
  });
}
```

**Keep components small and focused.** A component should do one thing well. If a component has more than 200 lines of template or more than 5–6 inputs, consider breaking it into smaller components. The "smart vs presentational" pattern — where "smart" components handle data fetching and state, and "presentational" (dumb) components receive data via inputs — makes testing and reuse far easier.

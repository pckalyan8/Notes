# Phase 12.1 — Angular Signals (Angular 16 → 21)

> **Prerequisites:** Angular components, RxJS basics, Change Detection (Phase 11.5)

---

## Table of Contents
1. [What Are Signals and Why They Exist](#1-what-are-signals-and-why-they-exist)
2. [signal() — Writable Signal](#2-signal--writable-signal)
3. [computed() — Derived Signal](#3-computed--derived-signal)
4. [effect() — Reactive Side Effects](#4-effect--reactive-side-effects)
5. [Signal-Based Inputs: input() and input.required()](#5-signal-based-inputs-input-and-inputrequired)
6. [Signal-Based Outputs: output()](#6-signal-based-outputs-output)
7. [model() — Two-Way Binding Signal](#7-model--two-way-binding-signal)
8. [toSignal() and toObservable() — RxJS Interop](#8-tosignal-and-toobservable--rxjs-interop)
9. [Signal-Based Queries](#9-signal-based-queries)
10. [linkedSignal() — Writable Derived Signal (Angular 19+)](#10-linkedsignal--writable-derived-signal-angular-19)
11. [Resource API: resource() and rxResource() (Angular 19+)](#11-resource-api-resource-and-rxresource-angular-19)
12. [httpResource() (Angular 21)](#12-httpresource-angular-21)
13. [untracked() — Read Without Dependency](#13-untracked--read-without-dependency)
14. [afterRender and afterNextRender](#14-afterrender-and-afternextrender)
15. [Signal Change Detection — How It Works](#15-signal-change-detection--how-it-works)
16. [Angular DevTools — Signal Graph Visualization](#16-angular-devtools--signal-graph-visualization)
17. [Signals vs RxJS — Decision Guide](#17-signals-vs-rxjs--decision-guide)
18. [Best Practices](#18-best-practices)

---

## 1. What Are Signals and Why They Exist

### The Problem Zone.js Solves — and the Cost

Before Angular Signals, Angular relied on **Zone.js** for change detection. Zone.js monkey-patches every async API (setTimeout, Promise, fetch, addEventListener). When any async operation completes anywhere in the app, Angular triggers a full change detection pass — checking **every component in the tree**.

```
A button click in ComponentA:
Zone.js intercepts event → triggers Angular CD:
  → checks ComponentA template
  → checks ComponentB template  (even though it has no relation to A)
  → checks ComponentC template
  → checks all 300 components...
```

**Cost:** Even unchanged components are re-evaluated. This is wasteful.

### The Problem with RxJS for Local State

`BehaviorSubject` requires:
- Manual subscription management
- Manual unsubscription in `ngOnDestroy` (memory leaks if forgotten)
- `async` pipe just to use in templates
- Verbose for simple counter/flag state

### What Signals Provide

A **Signal** is a reactive value container that:
1. **Tracks consumers** — knows exactly which templates, computed values, and effects read it
2. **Notifies precisely** — only re-renders components that consumed the changed signal
3. **No subscription** — Angular handles tracking automatically
4. **Works without Zone.js** — enables zoneless apps (Phase 12.2)

```
signal(count = 0) changed:
  → re-render CounterComponent   (reads count())
  → re-render SummaryComponent   (reads count() in computed)
  → skip all other 298 components
```

### Signal Mental Model: Reactive Spreadsheet

Think of a spreadsheet. Cell A1 = 5. Cell B1 = `=A1 * 2`. Cell C1 = `=B1 + 10`.

- Change A1 → Excel recalculates only B1 and C1
- Other cells are untouched

Angular Signals work identically — a precise dependency graph propagates changes only where needed.

---

## 2. `signal()` — Writable Signal

### Creating Signals

```typescript
import { signal } from '@angular/core';

// TypeScript infers the type from the initial value
const count = signal(0);               // Signal<number>
const name  = signal('Angular');       // Signal<string>
const flag  = signal(false);           // Signal<boolean>

// Explicit generic type
const user    = signal<User | null>(null);
const items   = signal<string[]>([]);
const config  = signal<AppConfig>({ theme: 'light', lang: 'en' });
```

### Reading a Signal

A signal is a **zero-argument function** — calling it returns the current value:

```typescript
console.log(count());    // 0
console.log(name());     // 'Angular'
console.log(user()?.id); // null
```

In templates, call signals the same way:
```html
<p>Count: {{ count() }}</p>
<p>User: {{ user()?.name ?? 'Guest' }}</p>
```

### Updating Signals

```typescript
// .set() — replace the entire value
count.set(5);
name.set('Angular 21');
user.set({ id: '1', name: 'Alice', role: 'admin' });

// .update() — compute new value from old value (ALWAYS use this for objects/arrays)
count.update(n => n + 1);
items.update(list => [...list, 'new item']);        // ✅ New reference
config.update(cfg => ({ ...cfg, theme: 'dark' })); // ✅ New reference

// ❌ WRONG — mutating in place doesn't notify consumers
items().push('new item');    // Does NOT trigger reactivity
config().theme = 'dark';     // Does NOT trigger reactivity
```

> **Important:** Signals use `Object.is` equality by default. Mutating an object in-place produces the same reference → no change detected → no re-render. Always create new references with `update()`.

### `.asReadonly()` — Encapsulation in Services

```typescript
@Injectable({ providedIn: 'root' })
export class CartService {
  // Private writable signal — only service can mutate it
  private _items = signal<CartItem[]>([]);
  private _total = computed(() =>
    this._items().reduce((sum, i) => sum + i.price * i.qty, 0)
  );

  // Public read-only signal — components can only read
  readonly items = this._items.asReadonly();
  readonly total = this._total;

  addItem(item: CartItem) {
    this._items.update(list => [...list, item]);
  }

  removeItem(id: string) {
    this._items.update(list => list.filter(i => i.id !== id));
  }
}
```

### Custom Equality Function

```typescript
// Avoid re-renders when content is the same even if reference changed
const user = signal<User>(
  { id: '1', name: 'Alice' },
  {
    equal: (prev, next) =>
      prev.id === next.id &&
      prev.name === next.name &&
      prev.role === next.role,
  }
);
```

---

## 3. `computed()` — Derived Signal

`computed()` creates a **read-only signal** whose value derives from other signals.

### Key Properties
- **Lazy** — only calculated when first read, not when dependencies change
- **Memoized** — cached; only recalculated when a dependency actually changed
- **Glitch-free** — never reads a "half-updated" state; evaluation is consistent

```typescript
import { signal, computed } from '@angular/core';

const firstName = signal('John');
const lastName  = signal('Doe');

const fullName = computed(() => `${firstName()} ${lastName()}`);

console.log(fullName()); // 'John Doe'

firstName.set('Jane');
// computed is NOT recalculated yet — it's lazy

console.log(fullName()); // 'Jane Doe' — recalculated now because it was read
```

### Complex Computed with Multiple Dependencies

```typescript
const products  = signal<Product[]>([]);
const search    = signal('');
const sortKey   = signal<'name' | 'price'>('name');
const sortOrder = signal<'asc' | 'desc'>('asc');

const filteredProducts = computed(() => {
  const term  = search().toLowerCase();
  const key   = sortKey();
  const order = sortOrder();

  const filtered = products().filter(p =>
    p.name.toLowerCase().includes(term) ||
    p.category.toLowerCase().includes(term)
  );

  return filtered.sort((a, b) => {
    const diff = key === 'name'
      ? a.name.localeCompare(b.name)
      : a.price - b.price;
    return order === 'asc' ? diff : -diff;
  });
});

const totalCount    = computed(() => filteredProducts().length);
const hasResults    = computed(() => totalCount() > 0);
const pageProducts  = computed(() => filteredProducts().slice(0, 20));
```

### Computed Chains (Derived from Derived)

```typescript
const cartItems = signal<CartItem[]>([]);

const subtotal  = computed(() => cartItems().reduce((s, i) => s + i.price * i.qty, 0));
const tax       = computed(() => Math.round(subtotal() * 0.1 * 100) / 100);
const shipping  = computed(() => subtotal() >= 100 ? 0 : 9.99);
const total     = computed(() => subtotal() + tax() + shipping());
const savings   = computed(() => cartItems().reduce((s, i) => s + (i.originalPrice - i.price) * i.qty, 0));
```

### Rules for computed()
1. **Pure** — no side effects (no HTTP calls, no DOM manipulation, no `.set()`)
2. **No writing to signals** inside computed — use `linkedSignal()` for that
3. **Tracks any signal read inside** — even conditionally read signals

```typescript
// ⚠️ CONDITIONAL TRACKING — be careful
const showDiscount = signal(true);
const price = signal(100);
const discount = signal(20);

// If showDiscount() is false, discount() is never read → not tracked
// Changing discount.set(30) won't recalculate this computed
const displayPrice = computed(() => {
  if (showDiscount()) {
    return price() - discount(); // discount tracked only when showDiscount is true
  }
  return price();
});
```

---

## 4. `effect()` — Reactive Side Effects

`effect()` runs code whenever any signal it reads changes. Unlike `computed()`, effects are for **side effects** — operations that interact with the outside world.

### When to Use effect()
- Syncing signal values to `localStorage`
- Logging/analytics
- Calling third-party imperative library APIs (charts, maps)
- DOM manipulation that must happen outside templates

```typescript
import { signal, effect } from '@angular/core';

@Component({ ... })
export class ThemeComponent {
  theme = signal<'light' | 'dark'>('light');
  fontSize = signal(14);

  constructor() {
    // Runs immediately on creation, then whenever theme or fontSize changes
    effect(() => {
      document.documentElement.setAttribute('data-theme', this.theme());
      document.documentElement.style.fontSize = `${this.fontSize()}px`;
    });
  }
}
```

### Effect with Cleanup (`onCleanup`)

When the effect re-runs, clean up the previous execution first:

```typescript
@Component({ ... })
export class PollingComponent {
  endpoint = signal('/api/status');

  constructor() {
    effect((onCleanup) => {
      const url = this.endpoint();
      const intervalId = setInterval(async () => {
        const data = await fetch(url).then(r => r.json());
        this.status.set(data);
      }, 5000);

      // Runs before next effect execution OR when component destroys
      onCleanup(() => clearInterval(intervalId));
    });
  }
}
```

### Effect Scheduling — Asynchronous

Effects do NOT run synchronously when you call `.set()`. They are scheduled asynchronously:

```typescript
const count = signal(0);

effect(() => console.log('Effect ran, count =', count()));
// Logs: 'Effect ran, count = 0' (initial run)

count.set(1);
count.set(2);
count.set(3);
// Effect has NOT run yet

// After Angular's scheduling (next microtask):
// Logs: 'Effect ran, count = 3'  (batched — only final value)
```

### Injection Context Requirement

`effect()` must be called in an **injection context**:

```typescript
@Component({ ... })
export class MyComponent {
  private destroyRef = inject(DestroyRef);

  // ✅ Field initializer — injection context
  private logEffect = effect(() => console.log(this.value()));

  // ✅ Constructor — injection context
  constructor() {
    effect(() => this.syncStorage());
  }

  someMethod() {
    // ❌ ERROR — not in injection context
    effect(() => ...);

    // ✅ Workaround — use runInInjectionContext
    runInInjectionContext(this.injector, () => {
      effect(() => ...);
    });
  }
}
```

### `allowSignalWrites` — Writing Inside Effects

By default writing to signals inside effects is disallowed (prevents infinite loops). When truly needed:

```typescript
effect(() => {
  const raw = this.rawInput();
  const processed = heavyProcessing(raw);
  this.processedOutput.set(processed); // Writing to a signal in effect
}, { allowSignalWrites: true });
```

> **Best Practice:** If you need to write a signal based on another signal, use `computed()` or `linkedSignal()` instead. `allowSignalWrites` should be a last resort.

---

## 5. Signal-Based Inputs: `input()` and `input.required()`

`input()` replaces `@Input()`. The value is a **read-only signal** — reactive by default.

### Basic Usage

```typescript
import { Component, input, computed } from '@angular/core';

@Component({
  selector: 'app-user-card',
  standalone: true,
  template: `
    <div class="card" [class]="cardClass()">
      <img [src]="user().avatar" [alt]="user().name" />
      <h3>{{ user().name }}</h3>
      <span class="badge">{{ user().role }}</span>
    </div>
  `,
})
export class UserCardComponent {
  // Optional with default value
  user = input<User>({ id: '', name: 'Anonymous', avatar: '/default.jpg', role: 'guest' });

  // Required — Angular errors at compile time if parent doesn't provide it
  theme = input.required<'light' | 'dark'>();

  // Computed from inputs — replaces complex ngOnChanges logic
  cardClass = computed(() => `card card--${this.theme()} card--${this.user().role}`);
}
```

Parent template:
```html
<app-user-card [user]="currentUser" theme="dark" />
```

### Input with Transform

```typescript
import { input, numberAttribute, booleanAttribute } from '@angular/core';

@Component({ selector: 'app-progress', ... })
export class ProgressComponent {
  // Automatically converts string "75" attribute to number 75
  value = input(0, { transform: numberAttribute });

  // Automatically converts presence of "disabled" to boolean true
  disabled = input(false, { transform: booleanAttribute });

  // Custom transform
  label = input('', {
    transform: (v: string) => v.trim().toUpperCase()
  });
}
```

```html
<app-progress value="75" disabled label="  loading... "></app-progress>
<!-- value() = 75, disabled() = true, label() = 'LOADING...' -->
```

### Replacing ngOnChanges with computed()

```typescript
// ❌ Old pattern — verbose ngOnChanges
@Component({ ... })
export class OldComponent {
  @Input() items: Product[] = [];
  @Input() filter = '';
  filteredItems: Product[] = [];

  ngOnChanges(changes: SimpleChanges) {
    if (changes['items'] || changes['filter']) {
      this.filteredItems = this.items.filter(p =>
        p.name.includes(this.filter)
      );
    }
  }
}

// ✅ New pattern — computed() is cleaner and reactive
@Component({ ... })
export class NewComponent {
  items  = input<Product[]>([]);
  filter = input('');

  filteredItems = computed(() =>
    this.items().filter(p =>
      p.name.toLowerCase().includes(this.filter().toLowerCase())
    )
  );
}
```

---

## 6. Signal-Based Outputs: `output()`

`output()` replaces `@Output() + EventEmitter`. Returns an `OutputEmitterRef`.

```typescript
import { Component, output, input, signal } from '@angular/core';

@Component({
  selector: 'app-paginator',
  standalone: true,
  template: `
    <button [disabled]="page() <= 1" (click)="prev()">Previous</button>
    <span>{{ page() }} of {{ totalPages() }}</span>
    <button [disabled]="page() >= totalPages()" (click)="next()">Next</button>
  `,
})
export class PaginatorComponent {
  page      = input(1);
  pageSize  = input(10);
  total     = input.required<number>();

  // Outputs
  pageChange = output<number>();

  totalPages = computed(() => Math.ceil(this.total() / this.pageSize()));

  prev() { this.pageChange.emit(this.page() - 1); }
  next() { this.pageChange.emit(this.page() + 1); }
}
```

Parent:
```html
<app-paginator
  [page]="currentPage"
  [pageSize]="20"
  [total]="totalItems"
  (pageChange)="onPageChange($event)"
/>
```

### `outputFromObservable()` and `outputToObservable()`

```typescript
import { outputFromObservable, outputToObservable } from '@angular/core/rxjs-interop';

// Observable → output
searched = outputFromObservable(
  this.searchSubject.pipe(debounceTime(300), distinctUntilChanged())
);

// output → Observable (in parent)
const pageChanges$ = outputToObservable(this.paginatorRef.pageChange);
pageChanges$.pipe(takeUntilDestroyed()).subscribe(page => this.loadPage(page));
```

---

## 7. `model()` — Two-Way Binding Signal

`model()` is a writable signal that's both an input AND an output. It powers `[(binding)]` syntax.

### How It Works

```typescript
@Component({
  selector: 'app-color-picker',
  standalone: true,
  template: `
    <div class="palette">
      @for (c of colors; track c) {
        <div
          [class.selected]="color() === c"
          [style.background]="c"
          (click)="color.set(c)"
        ></div>
      }
    </div>
  `,
})
export class ColorPickerComponent {
  color = model<string>('#ffffff');  // Implicitly creates 'color' input + 'colorChange' output
  colors = ['#ff0000', '#00ff00', '#0000ff', '#ffffff', '#000000'];
}
```

Parent:
```html
<!-- Two-way binding with [()] banana-in-a-box syntax -->
<app-color-picker [(color)]="selectedColor" />

<!-- Equivalent to: -->
<app-color-picker [color]="selectedColor" (colorChange)="selectedColor = $event" />
```

### `model.required()`

```typescript
@Component({ selector: 'app-rating', ... })
export class RatingComponent {
  value = model.required<number>(); // Parent MUST provide initial value

  rate(n: number) { this.value.set(n); }
}
```

---

## 8. `toSignal()` and `toObservable()` — RxJS Interop

### `toSignal()` — Convert Observable to Signal

```typescript
import { toSignal } from '@angular/core/rxjs-interop';

@Component({ ... })
export class UserListComponent {
  private userService = inject(UserService);

  // Wraps Observable in a signal — handles subscription/unsubscription automatically
  users = toSignal(
    this.userService.getAll(),
    { initialValue: [] as User[] }  // Required if Observable doesn't emit synchronously
  );

  // Without initialValue, type is Signal<User[] | undefined>
  // Angular will be undefined until the Observable emits
}
```

Template — no `async` pipe needed:
```html
@for (user of users(); track user.id) {
  <app-user-card [user]="user" />
}
```

### toSignal() Options

```typescript
// initialValue — prevents undefined on first render
const users = toSignal(users$, { initialValue: [] as User[] });

// rejectErrors — unhandled Observable errors throw in signal reads
const data = toSignal(data$, { rejectErrors: true });

// manualCleanup — don't unsubscribe on component destroy (for shared signals)
const globalClock = toSignal(interval(1000), { manualCleanup: true });

// injector — use a specific injector (for non-injection-context usage)
const data = toSignal(data$, { injector: this.injector });
```

### `toObservable()` — Convert Signal to Observable

```typescript
import { toObservable } from '@angular/core/rxjs-interop';

@Component({ ... })
export class SearchComponent {
  searchTerm = signal('');

  // Signal → Observable → pipe RxJS operators → back to signal
  results = toSignal(
    toObservable(this.searchTerm).pipe(
      debounceTime(400),
      distinctUntilChanged(),
      filter(term => term.length >= 2),
      switchMap(term =>
        this.searchService.search(term).pipe(
          catchError(() => of([]))
        )
      )
    ),
    { initialValue: [] as SearchResult[] }
  );
}
```

### `takeUntilDestroyed()` — Clean Subscriptions

```typescript
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';

@Component({ ... })
export class MyComponent {
  private destroyRef = inject(DestroyRef);

  ngOnInit() {
    this.someService.stream$.pipe(
      takeUntilDestroyed(this.destroyRef) // Auto-unsubscribes on destroy
    ).subscribe(data => this.data.set(data));
  }
}
```

---

## 9. Signal-Based Queries

Signal equivalents of `@ViewChild`, `@ViewChildren`, `@ContentChild`, `@ContentChildren`:

```typescript
import {
  viewChild, viewChildren,
  contentChild, contentChildren,
  ElementRef, Component
} from '@angular/core';

@Component({
  selector: 'app-chart',
  template: `
    <canvas #canvas></canvas>
    <div class="controls">
      <button #btn>Reset</button>
      <button #btn>Export</button>
    </div>
    <ng-content></ng-content>
  `,
})
export class ChartComponent {
  // Signal<ElementRef | undefined> — may not exist
  canvas = viewChild<ElementRef<HTMLCanvasElement>>('canvas');

  // Signal<ElementRef> — throws if not found (preferred when always present)
  canvasRequired = viewChild.required<ElementRef<HTMLCanvasElement>>('canvas');

  // Signal<readonly ElementRef[]> — all matching elements
  buttons = viewChildren<ElementRef<HTMLButtonElement>>('btn');

  // Query projected content
  legend = contentChild(ChartLegendComponent);
  tooltips = contentChildren(TooltipDirective);

  ngAfterViewInit() {
    const ctx = this.canvasRequired().nativeElement.getContext('2d');
    this.initChart(ctx!);
  }

  // Use in computed — reactive to DOM changes
  buttonCount = computed(() => this.buttons().length);
}
```

### Why Signal Queries Are Better

```typescript
// ❌ Old: @ViewChild is not reactive — can't use in computed()
@ViewChild('canvas') canvas!: ElementRef;
someComputed = computed(() => this.canvas.width); // ERROR: canvas not a signal

// ✅ New: viewChild() is a signal — fully reactive
canvas = viewChild.required<ElementRef>('canvas');
canvasWidth = computed(() => this.canvas().nativeElement.offsetWidth);
```

---

## 10. `linkedSignal()` — Writable Derived Signal (Angular 19+)

`linkedSignal()` fills the gap between `computed()` (read-only) and independent `signal()` (no dependency). It's a **writable signal that resets when its source changes**.

```typescript
import { signal, linkedSignal } from '@angular/core';

@Component({ ... })
export class DataTableComponent {
  pageSize = signal(10);

  // Resets to 1 whenever pageSize changes, but user can also manually set it
  currentPage = linkedSignal(() => {
    this.pageSize(); // dependency
    return 1;        // reset value when dependency changes
  });

  goToPage(n: number) { this.currentPage.set(n); }
}
```

### Advanced Form with linkedSignal

```typescript
const selectedCountry = signal<string>('US');
const countryCities: Record<string, string[]> = {
  US: ['New York', 'Los Angeles', 'Chicago'],
  UK: ['London', 'Manchester', 'Birmingham'],
  DE: ['Berlin', 'Munich', 'Hamburg'],
};

const cities = computed(() => countryCities[selectedCountry()] ?? []);

// City resets to first city when country changes, but user can pick another
const selectedCity = linkedSignal({
  source: selectedCountry,
  computation: (country, previous) => {
    return countryCities[country]?.[0] ?? '';
    // previous?.value was the previously selected city
    // previous?.source was the previous country
  },
});

// User can override:
selectedCity.set('Chicago');

// Change country → city resets to default
selectedCountry.set('UK'); // selectedCity() = 'London'
```

---

## 11. Resource API: `resource()` and `rxResource()` (Angular 19+)

Resources are signal-based wrappers for **async operations** with built-in state management.

### `resource()` — Promise-Based

```typescript
import { resource, signal } from '@angular/core';

@Component({ ... })
export class ProductDetailComponent {
  productId = signal<string>('');

  productResource = resource({
    // Reactive request — re-runs loader when productId changes
    request: () => ({ id: this.productId() }),

    // Loader receives the request value + AbortSignal
    loader: async ({ request, abortSignal }) => {
      if (!request.id) return null;

      const response = await fetch(`/api/products/${request.id}`, {
        signal: abortSignal,  // Aborts in-flight request if request changes
      });

      if (!response.ok) throw new Error(`HTTP ${response.status}`);
      return response.json() as Promise<Product>;
    },
  });
}
```

Template:
```html
@switch (productResource.status()) {
  @case ('loading') { <app-skeleton /> }
  @case ('refreshing') { <app-skeleton /> }
  @case ('error') {
    <app-error [message]="productResource.error()?.message" />
  }
  @case ('resolved') {
    <app-product-detail [product]="productResource.value()!" />
  }
}
```

### Resource API — All Properties

```typescript
const r = resource({ request: ..., loader: ... });

r.value()       // T | undefined
r.error()       // unknown | undefined
r.isLoading()   // boolean (status is 'loading' or 'refreshing')
r.status()      // 'idle' | 'loading' | 'refreshing' | 'resolved' | 'error' | 'local'

r.reload()      // Force re-fetch with same request
r.set(value)    // Optimistically set local value (status → 'local')
r.update(fn)    // Optimistically update local value

// Status meanings:
// 'idle'       — request is null/undefined, no fetch attempted
// 'loading'    — first load in progress
// 'refreshing' — reload in progress (previous value still accessible)
// 'resolved'   — data successfully loaded
// 'error'      — fetch threw or rejected
// 'local'      — manually set via r.set() or r.update()
```

### `rxResource()` — Observable-Based

```typescript
import { rxResource } from '@angular/core/rxjs-interop';

@Component({ ... })
export class ProductListComponent {
  category = signal('electronics');

  productsResource = rxResource({
    request: () => this.category(),
    loader: ({ request: cat }) =>
      this.productService.getByCategory(cat) // Returns Observable<Product[]>
  });
}
```

---

## 12. `httpResource()` (Angular 21)

`httpResource()` is a first-class signal-based HTTP resource built on top of Angular's `HttpClient`:

```typescript
import { httpResource } from '@angular/core';

@Component({ ... })
export class ProductListComponent {
  filter   = signal<'all' | 'active' | 'inactive'>('all');
  page     = signal(1);
  pageSize = signal(20);

  // Simplest form — reactive URL string
  productsResource = httpResource<ProductsResponse>(
    () => `/api/products?filter=${this.filter()}&page=${this.page()}&limit=${this.pageSize()}`
  );

  // Full config form
  productsResource = httpResource<ProductsResponse>({
    url: () => `/api/products`,
    method: 'GET',
    params: () => ({
      filter:  this.filter(),
      page:    String(this.page()),
      limit:   String(this.pageSize()),
    }),
    headers: () => ({
      'Accept': 'application/json',
      'X-API-Version': '2',
    }),
  });

  // When filter changes → automatically re-fetches
  total = computed(() => this.productsResource.value()?.total ?? 0);
  products = computed(() => this.productsResource.value()?.items ?? []);
}
```

### httpResource POST

```typescript
saveResource = httpResource<SaveResponse>({
  url: '/api/products',
  method: 'POST',
  body: () => this.form.value,
});
```

### httpResource vs Manual HttpClient

```typescript
// ❌ Manual approach — verbose
export class ProductComponent {
  products = signal<Product[]>([]);
  isLoading = signal(false);
  error = signal<string | null>(null);

  ngOnInit() {
    this.isLoading.set(true);
    this.http.get<Product[]>('/api/products').subscribe({
      next: p => { this.products.set(p); this.isLoading.set(false); },
      error: e => { this.error.set(e.message); this.isLoading.set(false); },
    });
  }
}

// ✅ httpResource — all state built-in
export class ProductComponent {
  productsResource = httpResource<Product[]>('/api/products');
  // products, isLoading, error — all available via resource API
}
```

---

## 13. `untracked()` — Read Without Dependency

Reading a signal normally inside `computed()` or `effect()` creates a reactive dependency. `untracked()` reads a signal **without registering a dependency**:

```typescript
import { signal, computed, effect, untracked } from '@angular/core';

const userId   = signal('user-1');
const logLevel = signal<'debug' | 'info' | 'warn'>('info');
const userData = signal<User | null>(null);

// Effect that re-runs only when userId changes
// logLevel is read but NOT tracked
effect(() => {
  const id  = userId();               // Tracked — effect re-runs when userId changes
  const lvl = untracked(logLevel);    // Untracked — effect does NOT re-run when logLevel changes

  if (lvl !== 'warn') {
    console.log(`Loading user ${id}`);
  }
  loadUser(id);
});

// Computed that only tracks priceSignal, not the format settings
const formattedPrice = computed(() => {
  const price    = priceSignal();            // Tracked
  const currency = untracked(currencyPref);  // Untracked

  return new Intl.NumberFormat('en', { style: 'currency', currency }).format(price);
});
```

---

## 14. `afterRender` and `afterNextRender`

Safe hooks for DOM interactions in a signals-based world:

```typescript
import { afterRender, afterNextRender, AfterRenderPhase } from '@angular/core';

@Component({ ... })
export class ChartComponent {
  private canvas = viewChild.required<ElementRef<HTMLCanvasElement>>('canvas');
  private chart?: Chart;

  constructor() {
    // Runs ONCE after the next render (like ngAfterViewInit)
    afterNextRender(() => {
      const ctx = this.canvas().nativeElement.getContext('2d')!;
      this.chart = new Chart(ctx, { type: 'line', data: this.getChartData() });
    });

    // Runs after EVERY render (like ngAfterViewChecked)
    afterRender(() => {
      this.chart?.update(); // Update chart whenever component re-renders
    });
  }
}
```

### Read and Write Phases — Avoid Layout Thrashing

```typescript
import { AfterRenderPhase } from '@angular/core';

let measuredHeight = 0;

// Phase 1: Read DOM measurements (don't write here)
afterRender(() => {
  measuredHeight = element.offsetHeight; // READ
}, { phase: AfterRenderPhase.Read });

// Phase 2: Apply DOM writes (don't read here)
afterRender(() => {
  siblingElement.style.height = `${measuredHeight}px`; // WRITE
}, { phase: AfterRenderPhase.Write });
```

> Angular batches all Read-phase callbacks before Write-phase, preventing the "read → layout invalidation → write → read → layout" thrash cycle.

---

## 15. Signal Change Detection — How It Works

### The Dependency Graph

Angular builds a reactive dependency graph at runtime:

```
signal(count)  ←── computed(double = count * 2)  ←── ComponentA template
                                                  ←── ComponentB template

signal(name)   ←── ComponentC template

signal(theme)  ←── effect (updates DOM attribute)
```

When `count.set(5)`:
1. Angular marks `computed(double)` as stale
2. Angular marks `ComponentA` and `ComponentB` as dirty
3. `ComponentC` and the effect for theme are **completely skipped**
4. On next frame, Angular re-renders only dirty components

### Glitch-Free Evaluation

The reactive graph ensures no "glitches" — scenarios where a component reads an inconsistent state:

```typescript
const a = signal(1);
const b = computed(() => a() * 2);
const c = computed(() => a() + b());
// c should always be a + a*2 = 3a

a.set(2);
// Naive system might: c reads a=2, b=old(2) → c = 4 (wrong!)
// Angular's system: c waits until b recomputes → c = 2 + 4 = 6 (correct)
```

---

## 16. Angular DevTools — Signal Graph Visualization

**Angular DevTools** (Chrome extension) provides visual insight into signals:

- **Component Explorer**: shows signal values alongside component inputs/outputs
- **Signal Profiler**: visualizes the dependency graph
- **Effect tracker**: shows when effects run and what triggered them

To use:
1. Install **Angular DevTools** from Chrome Web Store
2. Open Chrome DevTools → "Angular" tab
3. Select a component → see all signals and their current values
4. Click a signal to see what depends on it

---

## 17. Signals vs RxJS — Decision Guide

```
┌─────────────────────────────────┬──────────────┬────────────────┐
│ Use Case                        │ Signals      │ RxJS           │
├─────────────────────────────────┼──────────────┼────────────────┤
│ Component local state           │ ✅ signal()  │ ❌ Too heavy   │
│ Derived/computed values         │ ✅ computed()│ combineLatest  │
│ One-shot async (HTTP, resource) │ ✅ resource()│ Observable     │
│ Stream of events over time      │ ❌           │ ✅ Subject      │
│ debounce / throttle / retry     │ ❌           │ ✅ operators   │
│ WebSocket / SSE streams         │ ❌           │ ✅             │
│ Search-as-you-type              │ toObservable │ ✅ switchMap   │
│ Service shared state            │ ✅ signal()  │ BehaviorSubject│
│ HTTP data in component          │ ✅ httpResource│ HttpClient   │
│ Complex async orchestration     │ rxResource() │ ✅ RxJS        │
│ Time-based sequences            │ ❌           │ ✅ interval    │
└─────────────────────────────────┴──────────────┴────────────────┘
```

**Rule of thumb:**
- **Signals = state** (what is the current value of something?)
- **RxJS = streams** (what happened over time?)

---

## 18. Best Practices

### Signal Naming
```typescript
// No $ suffix for signals ($ is RxJS Observable convention)
count   = signal(0);      // ✅
count$  = signal(0);      // ❌ Confusing with Observables
```

### Always Update Objects/Arrays with New References
```typescript
// ❌ Mutating in-place — no reactivity
this.users().push(newUser);

// ✅ New reference — triggers reactivity
this.users.update(list => [...list, newUser]);
```

### Prefer computed() over Methods in Templates
```typescript
// ❌ Called on every CD cycle
<p>{{ formatDate(user().createdAt) }}</p>

// ✅ Cached, only recalculated when user() changes
formattedDate = computed(() => formatDate(this.user().createdAt));
<p>{{ formattedDate() }}</p>
```

### Use input.required() for Mandatory Inputs
```typescript
// ✅ TypeScript + Angular enforce at compile time
userId = input.required<string>();

// ❌ Runtime-only check
@Input({ required: true }) userId!: string;
```

### Combine Signals + Zoneless for Maximum Performance
```typescript
// ✅ The 2026 gold standard
@Component({
  changeDetection: ChangeDetectionStrategy.OnPush, // Always with signals
  ...
})
export class ProductCardComponent {
  product = input.required<Product>();
  addedToCart = output<Product>();
  discountedPrice = computed(() => this.product().price * 0.9);
}
```

---

> **Summary:** Angular Signals are the most significant architectural shift in Angular's history. They provide fine-grained reactivity (`signal`, `computed`, `effect`), a modern component API (`input`, `output`, `model`), first-class async handling (`resource`, `httpResource`), and natural RxJS interop (`toSignal`, `toObservable`). Mastering Signals is the foundation of all modern Angular development in 2026.

# Phase 14.3 — Service-Based State
## BehaviorSubject Pattern, Signal Stores, Facade & Repository Patterns

---

## Why Service-Based State?

Before NgRx existed and before Angular Signals were introduced, Angular developers discovered that the simplest way to share state between components was through **injectable services**. A service in Angular is a singleton (when provided at the root level), which means every component that injects it receives the same instance. If that service holds mutable state, every component that injects it sees the same state — and if that state is reactive (via RxJS or signals), every component updates automatically when it changes.

This is not a workaround or a hack. Service-based state management is a first-class, fully legitimate approach for managing state in Angular applications, particularly when the overhead of NgRx would be disproportionate. Understanding it deeply is important both for working in real-world codebases (where many mature Angular apps use this pattern) and as the foundational layer that more sophisticated patterns build upon.

---

## 14.3.1 — BehaviorSubject and ReplaySubject Pattern

### BehaviorSubject — The Classic Approach

`BehaviorSubject` is an RxJS Subject that holds a *current value* and emits it immediately to any new subscriber. This makes it ideal for state — any component that subscribes gets the current value right away, without waiting for the next emission.

The classic service-based state pattern wraps `BehaviorSubject` in a service with a public Observable exposed via `asObservable()` and mutation methods that hide the Subject from consumers. This enforces unidirectional flow: consumers can only read state through the observable, and they can only change state by calling defined service methods.

```typescript
import { Injectable } from '@angular/core';
import { BehaviorSubject, combineLatest, Observable } from 'rxjs';
import { map, distinctUntilChanged } from 'rxjs/operators';

export interface Todo {
  id: string;
  text: string;
  completed: boolean;
  priority: 'low' | 'medium' | 'high';
}

type FilterMode = 'all' | 'active' | 'completed';

@Injectable({ providedIn: 'root' })
export class TodoStore {
  // Private BehaviorSubjects — only this service writes to them
  // The initial value is set in the constructor
  private _todos = new BehaviorSubject<Todo[]>([]);
  private _filter = new BehaviorSubject<FilterMode>('all');
  private _loading = new BehaviorSubject<boolean>(false);

  // Public Observables — components can only read, never write directly
  // asObservable() hides the Subject's .next() method from consumers
  readonly todos$ = this._todos.asObservable();
  readonly filter$ = this._filter.asObservable();
  readonly loading$ = this._loading.asObservable();

  // Derived observable using combineLatest — automatically recalculates
  // when either todos$ or filter$ emits a new value
  readonly filteredTodos$: Observable<Todo[]> = combineLatest([
    this._todos,
    this._filter,
  ]).pipe(
    map(([todos, filter]) => {
      switch (filter) {
        case 'active':    return todos.filter(t => !t.completed);
        case 'completed': return todos.filter(t => t.completed);
        default:          return todos;
      }
    }),
    // distinctUntilChanged prevents emissions when the derived value is the same
    // as the previous one — an important performance optimization
    distinctUntilChanged((a, b) => JSON.stringify(a) === JSON.stringify(b))
  );

  // Synchronous selectors using .getValue() — useful when you need the current
  // value inside a method without subscribing
  get currentFilter(): FilterMode {
    return this._filter.getValue();
  }

  // Mutation methods — the only way to change state
  add(text: string) {
    const newTodo: Todo = {
      id: crypto.randomUUID(),
      text,
      completed: false,
      priority: 'medium',
    };
    // Always create a new array — never push to the existing one
    // This ensures reference equality works for OnPush change detection
    this._todos.next([...this._todos.getValue(), newTodo]);
  }

  toggle(id: string) {
    this._todos.next(
      this._todos.getValue().map(t =>
        t.id === id ? { ...t, completed: !t.completed } : t
      )
    );
  }

  remove(id: string) {
    this._todos.next(this._todos.getValue().filter(t => t.id !== id));
  }

  setFilter(filter: FilterMode) {
    this._filter.next(filter);
  }
}
```

```typescript
// Using the BehaviorSubject service in a component
@Component({
  imports: [AsyncPipe],
  template: `
    @if (todoStore.loading$ | async) {
      <mat-spinner />
    }

    @for (todo of todoStore.filteredTodos$ | async; track todo.id) {
      <div [class.completed]="todo.completed">
        <input type="checkbox"
               [checked]="todo.completed"
               (change)="todoStore.toggle(todo.id)">
        {{ todo.text }}
        <button (click)="todoStore.remove(todo.id)">✕</button>
      </div>
    }

    <div class="filters">
      <button (click)="todoStore.setFilter('all')">All</button>
      <button (click)="todoStore.setFilter('active')">Active</button>
      <button (click)="todoStore.setFilter('completed')">Completed</button>
    </div>
  `
})
export class TodoListComponent {
  todoStore = inject(TodoStore);
  newTodoText = '';

  addTodo() {
    if (this.newTodoText.trim()) {
      this.todoStore.add(this.newTodoText.trim());
      this.newTodoText = '';
    }
  }
}
```

### ReplaySubject — For Late Subscribers

`BehaviorSubject` always requires an initial value, which may not always make sense. If you are loading data from an API, the "initial value" before the first fetch completes is genuinely absent — not an empty array, but *nothing yet*. `ReplaySubject` with a buffer size of 1 solves this by replaying only the most recent value to new subscribers, without needing an initial value.

```typescript
import { ReplaySubject } from 'rxjs';

@Injectable({ providedIn: 'root' })
export class UserProfileService {
  // ReplaySubject(1) replays only the last emitted value to new subscribers
  // Unlike BehaviorSubject, it has no "initial" value — it emits nothing until
  // the first .next() call. This is semantically correct for "data not yet loaded"
  private _profile = new ReplaySubject<UserProfile>(1);

  readonly profile$ = this._profile.asObservable();

  loadProfile(userId: string) {
    this.http.get<UserProfile>(`/api/users/${userId}`)
      .subscribe(profile => this._profile.next(profile));
  }
}
```

A critical difference: `BehaviorSubject` emits its initial value immediately to all subscribers, even before any real data arrives. If your initial value is `null` or `[]`, you may render briefly with that empty value before real data arrives. `ReplaySubject(1)` does not emit until real data arrives, which is often semantically cleaner but requires you to handle the "not yet emitted" case in your template (the `async` pipe returns `null` until the first emission).

---

## 14.3.2 — Signal-Based Service Store (The Modern Approach)

Since Angular 16 and especially with Angular's push toward zoneless in Angular 20, **signals are the preferred mechanism for service-based state** in new code. They are simpler to reason about than RxJS subjects, integrate naturally with computed values, and perform better in zoneless applications.

The signal-based service store is essentially the same pattern as BehaviorSubject, but implemented with Angular's built-in signal primitives:

```typescript
import { Injectable, computed, signal } from '@angular/core';

export interface CartItem {
  productId: string;
  name: string;
  price: number;
  quantity: number;
  imageUrl: string;
}

@Injectable({ providedIn: 'root' })
export class CartStore {
  // Private writable signals — the internal state
  private _items = signal<CartItem[]>([]);
  private _couponCode = signal<string | null>(null);
  private _discountPercent = signal(0);

  // Public read-only signals — this is what components consume
  // asReadonly() creates a ReadonlySignal<T> that has no .set() or .update() methods
  readonly items = this._items.asReadonly();
  readonly couponCode = this._couponCode.asReadonly();

  // computed() automatically derives values from signals — memoized and lazy
  // These only recalculate when their dependencies change
  readonly itemCount = computed(() =>
    this._items().reduce((sum, item) => sum + item.quantity, 0)
  );

  readonly subtotal = computed(() =>
    this._items().reduce((sum, item) => sum + item.price * item.quantity, 0)
  );

  readonly discount = computed(() =>
    this.subtotal() * (this._discountPercent() / 100)
  );

  readonly total = computed(() =>
    this.subtotal() - this.discount()
  );

  readonly isEmpty = computed(() => this._items().length === 0);

  readonly isCartValidForCheckout = computed(() =>
    !this.isEmpty() && this.total() > 0
  );

  // Mutation methods — explicit, named channels for state changes
  addItem(product: { id: string; name: string; price: number; imageUrl: string }) {
    this._items.update(items => {
      const existing = items.find(i => i.productId === product.id);
      if (existing) {
        // If already in cart, increase quantity
        return items.map(i =>
          i.productId === product.id
            ? { ...i, quantity: i.quantity + 1 }
            : i
        );
      }
      // Otherwise add as a new item
      return [...items, {
        productId: product.id,
        name: product.name,
        price: product.price,
        imageUrl: product.imageUrl,
        quantity: 1,
      }];
    });
  }

  removeItem(productId: string) {
    this._items.update(items => items.filter(i => i.productId !== productId));
  }

  updateQuantity(productId: string, quantity: number) {
    if (quantity <= 0) {
      this.removeItem(productId);
      return;
    }
    this._items.update(items =>
      items.map(i => i.productId === productId ? { ...i, quantity } : i)
    );
  }

  applyCoupon(code: string, discountPercent: number) {
    this._couponCode.set(code);
    this._discountPercent.set(discountPercent);
  }

  removeCoupon() {
    this._couponCode.set(null);
    this._discountPercent.set(0);
  }

  clearCart() {
    this._items.set([]);
    this._couponCode.set(null);
    this._discountPercent.set(0);
  }
}
```

```typescript
// Using in a component — completely clean, no async pipe, no subscriptions
@Component({
  template: `
    @if (cart.isEmpty()) {
      <div class="empty-cart">
        <p>Your cart is empty.</p>
        <a routerLink="/products">Start shopping</a>
      </div>
    } @else {
      @for (item of cart.items(); track item.productId) {
        <div class="cart-item">
          <img [src]="item.imageUrl" [alt]="item.name">
          <span>{{ item.name }}</span>
          <input type="number"
                 [value]="item.quantity"
                 (change)="cart.updateQuantity(item.productId, +$any($event.target).value)"
                 min="1">
          <span>{{ item.price * item.quantity | currency }}</span>
          <button (click)="cart.removeItem(item.productId)">Remove</button>
        </div>
      }

      <div class="cart-summary">
        <p>Subtotal: {{ cart.subtotal() | currency }}</p>
        @if (cart.discount() > 0) {
          <p class="discount">Discount: -{{ cart.discount() | currency }}</p>
        }
        <p class="total"><strong>Total: {{ cart.total() | currency }}</strong></p>
      </div>

      <button [disabled]="!cart.isCartValidForCheckout()"
              (click)="checkout()">
        Proceed to Checkout ({{ cart.itemCount() }} items)
      </button>
    }
  `
})
export class CartComponent {
  cart = inject(CartStore);

  checkout() {
    // Navigate to checkout or open checkout flow
  }
}
```

Notice how clean the template is. No `async` pipe, no subscription management, no `ngOnDestroy`. Signals handle all of the reactivity automatically. The component reads signal values by calling them as functions — `cart.items()`, `cart.isEmpty()`, `cart.total()` — and Angular's template engine automatically re-renders only the parts that depend on changed signals.

---

## 14.3.3 — The Facade Pattern

The **Facade pattern** is a layer of abstraction placed between your components and your state management implementation. A Facade service exposes a clean, semantic, component-friendly API and internally delegates to whatever state management mechanism is in use — NgRx, signals, BehaviorSubject, or any combination.

The Facade pattern solves a real problem: components that import directly from NgRx selectors and dispatch actions directly are tightly coupled to the NgRx implementation. If you ever want to replace NgRx with a signal store, or add caching, or change the action structure, you have to update every component that touches that state. With a Facade, you only update the Facade service itself — components remain completely unaware of the change.

```typescript
// Without a Facade — the component is tightly coupled to NgRx implementation details
@Component({ /* ... */ })
export class ProductsComponent {
  private store = inject(Store);

  // Component must know about the selector structure
  products$ = this.store.select(selectAllProducts);
  loading$ = this.store.select(selectProductsLoading);

  // Component must know the action creators and their parameters
  load() {
    this.store.dispatch(ProductsApiActions.loadProducts());
  }

  delete(id: string) {
    this.store.dispatch(ProductsPageActions.deleteProduct({ id }));
  }
}
```

```typescript
// The Facade — a thin semantic layer hiding the NgRx details
@Injectable({ providedIn: 'root' })
export class ProductsFacade {
  private store = inject(Store);

  // Expose semantically meaningful state — components don't care about selectors
  readonly products = toSignal(this.store.select(selectAllProducts), { initialValue: [] });
  readonly loading = toSignal(this.store.select(selectProductsLoading), { initialValue: false });
  readonly error = toSignal(this.store.select(selectProductsError), { initialValue: null });
  readonly selectedProduct = toSignal(this.store.select(selectSelectedProduct), { initialValue: null });

  // Expose semantically meaningful operations — components don't care about actions
  loadProducts() {
    this.store.dispatch(ProductsApiActions.loadProducts());
  }

  selectProduct(id: string) {
    this.store.dispatch(ProductsPageActions.selectProduct({ id }));
  }

  deleteProduct(id: string) {
    this.store.dispatch(ProductsPageActions.deleteProduct({ id }));
  }

  createProduct(product: CreateProductDto) {
    this.store.dispatch(ProductsPageActions.createProduct({ product }));
  }
}
```

```typescript
// With a Facade — the component has zero NgRx knowledge
// It reads and writes through a semantic API it can understand
@Component({ /* ... */ })
export class ProductsComponent {
  // Only knows about the Facade — no NgRx imports at all
  protected facade = inject(ProductsFacade);

  constructor() {
    this.facade.loadProducts();
  }

  onDelete(id: string) {
    this.facade.deleteProduct(id);
  }
}
```

The Facade can also combine data from multiple state sources — for example, combining products from NgRx with user permissions from a separate auth service to create a single `canDeleteProducts` signal that the component simply consumes:

```typescript
@Injectable({ providedIn: 'root' })
export class ProductsFacade {
  private store = inject(Store);
  private authService = inject(AuthService);

  // Combine multiple state sources into one coherent signal
  readonly canDeleteProducts = computed(() =>
    this.authService.currentUser()?.role === 'admin'
  );

  // The component doesn't need to know this combines two sources
  readonly viewModel = computed(() => ({
    products: this.products(),
    loading: this.loading(),
    canDelete: this.canDeleteProducts(),
  }));
}
```

---

## 14.3.4 — The Repository Pattern for Data Access

The **Repository pattern** provides an abstraction over your data access logic — HTTP calls, IndexedDB operations, localStorage reads, etc. — behind a consistent interface. A Repository component doesn't know or care whether data comes from an API, a local cache, or localStorage. It just calls the Repository and receives the data.

This pattern is especially valuable when you need to change your data source (migrate from REST to GraphQL, add offline caching, switch API endpoints between environments) without touching any component code.

```typescript
// The Repository Interface — defines the contract
export abstract class ProductRepository {
  abstract getAll(): Observable<Product[]>;
  abstract getById(id: string): Observable<Product>;
  abstract create(product: CreateProductDto): Observable<Product>;
  abstract update(id: string, changes: Partial<Product>): Observable<Product>;
  abstract delete(id: string): Observable<void>;
}
```

```typescript
// HTTP Implementation — talks to the real API
@Injectable()
export class HttpProductRepository extends ProductRepository {
  private http = inject(HttpClient);
  private baseUrl = '/api/products';

  getAll(): Observable<Product[]> {
    return this.http.get<Product[]>(this.baseUrl);
  }

  getById(id: string): Observable<Product> {
    return this.http.get<Product>(`${this.baseUrl}/${id}`);
  }

  create(product: CreateProductDto): Observable<Product> {
    return this.http.post<Product>(this.baseUrl, product);
  }

  update(id: string, changes: Partial<Product>): Observable<Product> {
    return this.http.patch<Product>(`${this.baseUrl}/${id}`, changes);
  }

  delete(id: string): Observable<void> {
    return this.http.delete<void>(`${this.baseUrl}/${id}`);
  }
}
```

```typescript
// In-Memory Implementation — for tests or offline scenarios
@Injectable()
export class InMemoryProductRepository extends ProductRepository {
  private products = signal<Product[]>([
    { id: '1', name: 'Test Product', price: 29.99, category: 'test' },
  ]);

  getAll(): Observable<Product[]> {
    return of(this.products()).pipe(delay(50)); // simulate network delay
  }

  getById(id: string): Observable<Product> {
    const product = this.products().find(p => p.id === id);
    if (!product) return throwError(() => new Error(`Product ${id} not found`));
    return of(product);
  }

  create(dto: CreateProductDto): Observable<Product> {
    const newProduct: Product = { ...dto, id: crypto.randomUUID() };
    this.products.update(current => [...current, newProduct]);
    return of(newProduct);
  }

  update(id: string, changes: Partial<Product>): Observable<Product> {
    let updated!: Product;
    this.products.update(current =>
      current.map(p => {
        if (p.id === id) { updated = { ...p, ...changes }; return updated; }
        return p;
      })
    );
    return of(updated);
  }

  delete(id: string): Observable<void> {
    this.products.update(current => current.filter(p => p.id !== id));
    return of(undefined);
  }
}
```

```typescript
// Registering the implementation — swap this to change the data source
// The service and all components remain unchanged
bootstrapApplication(AppComponent, {
  providers: [
    // Use HTTP implementation in production
    { provide: ProductRepository, useClass: HttpProductRepository },

    // Use in-memory implementation for testing or development offline
    // { provide: ProductRepository, useClass: InMemoryProductRepository },
  ]
});
```

```typescript
// A ProductsService uses the Repository — it doesn't know if data is from HTTP or memory
@Injectable({ providedIn: 'root' })
export class ProductsService {
  // Inject the abstract class — the concrete implementation is swapped at the DI level
  private repo = inject(ProductRepository);

  getAll(): Observable<Product[]> {
    return this.repo.getAll();
  }

  async createProduct(dto: CreateProductDto): Promise<Product> {
    return firstValueFrom(this.repo.create(dto));
  }
}
```

The repository pattern makes testing dramatically easier. In unit tests, you provide `InMemoryProductRepository` and test your service and component behavior without making real HTTP calls. In integration tests or development without a backend, you can run the entire application with in-memory data.

---

## Important Points and Best Practices

The BehaviorSubject pattern is still widely used in production Angular codebases and you will encounter it frequently. Understanding it is not just historical knowledge — it is necessary for reading and maintaining existing code. However, for new services, prefer the signal-based approach because it eliminates the need for `async` pipe in templates, integrates naturally with `computed()` for derived values, and performs better in zoneless Angular applications.

Never expose a `Subject` or `BehaviorSubject` directly from a service. Always expose it as an Observable via `.asObservable()` (for RxJS) or as a signal via `.asReadonly()` (for signals). Exposing the raw Subject allows consumers to call `.next()` directly, bypassing the service's mutation methods and breaking unidirectional flow. This is one of the most common and damaging mistakes in service-based state management.

The Facade pattern has a real cost: it adds a layer of indirection, which means more files and more indirection to trace when debugging. Use it when it provides genuine value — specifically, when you have components that should be decoupled from your state management implementation. Do not use it as a ritual applied uniformly to every service regardless of need.

Combine the Repository and Facade patterns thoughtfully. The Repository handles *data access* (how to read and write data). The Facade handles *state coordination* (what state to expose and what operations to offer). They are complementary but serve different purposes. A service can use a Repository internally and be exposed via a Facade externally without any conceptual conflict.

The signal-based service store is the spiritual successor to the BehaviorSubject pattern. If you are starting a new Angular project today, use signals for service-based state. The signal primitives are simpler to learn, have fewer pitfalls (no subscription management, no timing issues with initial values), and align with where Angular's reactivity model is heading.

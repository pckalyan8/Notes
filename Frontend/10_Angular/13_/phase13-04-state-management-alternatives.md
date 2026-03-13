# Phase 13.4 — Other State Management Options
## Complete Guide: NGXS, Elf, Signal Patterns & TanStack Store

---

## The State Management Landscape in Angular (2026)

NgRx isn't the only game in town. The Angular ecosystem has several state management solutions, each making different trade-offs between power, boilerplate, learning curve, and philosophy. Understanding them helps you choose the right tool for each project rather than defaulting to NgRx's verbosity when something simpler would serve better.

Here's a mental model for when to evaluate alternatives: if you're on a **large enterprise team** with many engineers maintaining shared state across dozens of feature modules, NgRx's strict conventions and excellent DevTools are hard to beat. If you're on a **small-to-medium product team** building a focused application, the boilerplate may slow you down more than it helps you. That's where these alternatives shine.

---

## 1. Signals-Based State (No Library Required)

Before reaching for any state management library, consider whether Angular's built-in signals can handle your needs. For many applications — especially in the 2025+ era of Angular — a signal-based service is sufficient and dramatically simpler.

### Signal Store Service Pattern

This pattern gives you reactive state, derived computations, and clean methods, all with zero extra dependencies:

```typescript
import { Injectable, computed, signal } from '@angular/core';
import { rxResource } from '@angular/core/rxjs-interop';
import { HttpClient } from '@angular/common/http';
import { inject } from '@angular/core';

export interface CartItem {
  id: string;
  name: string;
  price: number;
  quantity: number;
}

@Injectable({ providedIn: 'root' })
export class CartStore {
  // Private writable signals — external code can only read, not write
  private _items = signal<CartItem[]>([]);
  private _isCheckingOut = signal(false);

  // Public read-only signals exposed to consumers
  readonly items = this._items.asReadonly();
  readonly isCheckingOut = this._isCheckingOut.asReadonly();

  // Computed signals — automatically recalculate when dependencies change
  readonly itemCount = computed(() =>
    this._items().reduce((total, item) => total + item.quantity, 0)
  );

  readonly subtotal = computed(() =>
    this._items().reduce((sum, item) => sum + item.price * item.quantity, 0)
  );

  readonly isEmpty = computed(() => this._items().length === 0);

  // Methods that mutate state — these are your "actions" + "reducers" combined
  addItem(item: Omit<CartItem, 'quantity'>) {
    this._items.update(current => {
      const existing = current.find(i => i.id === item.id);
      if (existing) {
        // Update quantity if item already exists
        return current.map(i =>
          i.id === item.id ? { ...i, quantity: i.quantity + 1 } : i
        );
      }
      return [...current, { ...item, quantity: 1 }];
    });
  }

  removeItem(id: string) {
    this._items.update(current => current.filter(i => i.id !== id));
  }

  updateQuantity(id: string, quantity: number) {
    if (quantity <= 0) {
      this.removeItem(id);
      return;
    }
    this._items.update(current =>
      current.map(i => i.id === id ? { ...i, quantity } : i)
    );
  }

  clearCart() {
    this._items.set([]);
  }
}
```

```typescript
// Using in a component — completely clean, no subscriptions, no boilerplate
@Component({
  template: `
    <p>{{ cartStore.itemCount() }} items — ${{ cartStore.subtotal() | number:'1.2-2' }}</p>

    @for (item of cartStore.items(); track item.id) {
      <div class="cart-item">
        <span>{{ item.name }}</span>
        <button (click)="cartStore.updateQuantity(item.id, item.quantity - 1)">-</button>
        <span>{{ item.quantity }}</span>
        <button (click)="cartStore.updateQuantity(item.id, item.quantity + 1)">+</button>
        <button (click)="cartStore.removeItem(item.id)">Remove</button>
      </div>
    }

    @if (cartStore.isEmpty()) {
      <p>Your cart is empty.</p>
    }
  `
})
export class CartComponent {
  cartStore = inject(CartStore);
}
```

> **Key insight:** This pattern is surprisingly powerful. `computed()` memoizes derived values. `signal.update()` makes state updates pure (takes current state, returns new state). The `asReadonly()` call exposes a read-only view of the signal, preventing external mutation — this enforces the same discipline as NgRx without the boilerplate. Use this for **local to medium complexity** state.

### linkedSignal — Resettable Derived Signals (Angular 19+)

`linkedSignal` is useful when you have a signal that should reset to a new value whenever a source signal changes, but can also be independently written to:

```typescript
import { signal, linkedSignal } from '@angular/core';

const selectedUserId = signal<string | null>(null);

// This signal resets to null whenever selectedUserId changes,
// but can also be set independently
const selectedTab = linkedSignal({
  source: selectedUserId,
  computation: () => 'overview',  // reset to 'overview' when user changes
});

// Elsewhere, user can switch tabs within the same user view:
selectedTab.set('settings');  // works like a normal signal
// But when the user changes, selectedTab resets to 'overview'
selectedUserId.set('user-456'); // selectedTab() is now 'overview' again
```

---

## 2. NGXS — Decorator-Based State Management

NGXS is a state management library that borrows from Redux's ideas but uses Angular decorators and classes to reduce boilerplate. Instead of separate files for actions, reducers, and selectors, you write everything in a single state class.

```bash
npm install @ngxs/store
```

### Core Concepts in NGXS

In NGXS, a **State** is a class decorated with `@State`. **Actions** are simple classes. The state class handles actions in methods decorated with `@Action`. You read state using `@Select` decorators or the `Store.select()` method.

```typescript
// books/books.actions.ts
export class LoadBooks {
  static readonly type = '[Books] Load Books';
}

export class AddBook {
  static readonly type = '[Books] Add Book';
  constructor(public book: Book) {}  // payload passed as constructor args
}

export class DeleteBook {
  static readonly type = '[Books] Delete Book';
  constructor(public id: string) {}
}
```

```typescript
// books/books.state.ts
import { State, Action, Selector, StateContext } from '@ngxs/store';
import { Injectable } from '@angular/core';
import { tap } from 'rxjs/operators';

interface BooksStateModel {
  books: Book[];
  loading: boolean;
}

@State<BooksStateModel>({
  name: 'books',
  defaults: { books: [], loading: false }
})
@Injectable()
export class BooksState {
  constructor(private booksService: BooksService) {}

  // Selectors — static methods that project state
  @Selector()
  static allBooks(state: BooksStateModel) {
    return state.books;
  }

  @Selector()
  static loading(state: BooksStateModel) {
    return state.loading;
  }

  // Derived selector from another selector
  @Selector([BooksState.allBooks])
  static topRatedBooks(books: Book[]) {
    return books.filter(b => b.rating >= 4.5);
  }

  // Action handlers — patch state using the context
  @Action(LoadBooks)
  loadBooks(ctx: StateContext<BooksStateModel>) {
    ctx.patchState({ loading: true });

    return this.booksService.getAll().pipe(
      tap(books => ctx.patchState({ books, loading: false }))
    );
  }

  @Action(AddBook)
  addBook(ctx: StateContext<BooksStateModel>, { book }: AddBook) {
    const currentBooks = ctx.getState().books;
    ctx.patchState({ books: [...currentBooks, book] });
  }

  @Action(DeleteBook)
  deleteBook(ctx: StateContext<BooksStateModel>, { id }: DeleteBook) {
    const filtered = ctx.getState().books.filter(b => b.id !== id);
    ctx.patchState({ books: filtered });
  }
}
```

```typescript
// Component usage
@Component({ /* ... */ })
export class BooksComponent {
  private store = inject(Store);

  // Selects return Observables — convert to signals
  books = toSignal(this.store.select(BooksState.allBooks), { initialValue: [] });
  loading = toSignal(this.store.select(BooksState.loading), { initialValue: false });

  ngOnInit() {
    this.store.dispatch(new LoadBooks());
  }

  delete(id: string) {
    this.store.dispatch(new DeleteBook(id));
  }
}
```

### NGXS vs NgRx Comparison

NGXS has significantly less boilerplate than classic NgRx. You don't need separate action files, reducer files, effects files, and selector files — it's all in one state class. This is a genuine advantage for teams that find NgRx's file explosion overwhelming.

However, NGXS is less popular and less actively maintained than NgRx, and it lacks the ecosystem depth (particularly the Signals Store in NgRx) that NgRx offers. If you're starting fresh, NgRx Signals Store is usually the better choice in 2026.

---

## 3. Elf — Lightweight Reactive Store with RxJS

Elf is a lightweight, reactive state management library built on top of RxJS. It's framework-agnostic but works very well with Angular. It's conceptually similar to NgRx's `@ngrx/component-store` but with a more functional API and less opinionation.

```bash
npm install @ngneat/elf
```

```typescript
import { createStore, withProps, select, setProps } from '@ngneat/elf';
import { withEntities, setAllEntities, deleteEntity, upsertEntities } from '@ngneat/elf-entities';
import { createRequestsStatusOperator, withRequestsStatus } from '@ngneat/elf-requests';

interface Book {
  id: string;
  title: string;
  author: string;
}

// Create the store — compose multiple "features"
const store = createStore(
  { name: 'books' },
  withEntities<Book>(),          // normalized entity management
  withProps<{ selectedId: string | null }>({ selectedId: null }),
  withRequestsStatus(),          // track loading/error state per request
);

// Create a request status operator for type-safe request tracking
const trackRequestsStatus = createRequestsStatusOperator(store);

// Elf stores are just RxJS observables under the hood
export const books$ = store.pipe(selectAllEntities());
export const selectedId$ = store.pipe(select(s => s.selectedId));

// Mutations — plain functions that update the store
export function setBooks(books: Book[]) {
  store.update(setAllEntities(books));
}

export function selectBook(id: string) {
  store.update(setProps({ selectedId: id }));
}

export function removeBook(id: string) {
  store.update(deleteEntity(id));
}
```

Elf's strength is its composability — you build stores by composing small "features" (withEntities, withProps, withRequestsStatus, withPagination, etc.) rather than writing monolithic state classes. It's an excellent choice for **medium-complexity apps** where NgRx feels like overkill but you still need structure.

---

## 4. TanStack Store — Framework-Agnostic Reactive Store

TanStack Store is the state management part of the TanStack ecosystem (which also includes TanStack Query, Table, Router). It's framework-agnostic, meaning the same store code could theoretically work in Angular, React, Vue, or plain JavaScript. Angular has an adapter via `@tanstack/angular-store`.

```bash
npm install @tanstack/store @tanstack/angular-store
```

```typescript
import { Store } from '@tanstack/store';

// Define the store with plain TypeScript
const counterStore = new Store({
  state: { count: 0 },
});

// Update state — always immutable
counterStore.setState(prev => ({ count: prev.count + 1 }));
```

```typescript
// In an Angular component — use the injectStore adapter
import { injectStore } from '@tanstack/angular-store';

@Component({
  template: `<p>Count: {{ count() }}</p><button (click)="increment()">+</button>`
})
export class CounterComponent {
  // injectStore returns a signal that updates when the store changes
  count = injectStore(counterStore, state => state.count);

  increment() {
    counterStore.setState(prev => ({ count: prev.count + 1 }));
  }
}
```

TanStack Store is most useful when you want to **share state between Angular and another framework** (rare) or when you're already deeply invested in the TanStack ecosystem and want consistent patterns.

---

## 5. Component Store (`@ngrx/component-store`) — Local Component State

The NgRx Component Store is worth special attention because it's a middle ground — it uses NgRx's patterns and DevTools, but is scoped to a single component and its children rather than the global store. It's perfect for complex, stateful components that shouldn't pollute the global state.

```typescript
import { ComponentStore } from '@ngrx/component-store';

interface PaginationState {
  currentPage: number;
  pageSize: number;
  totalItems: number;
  items: Product[];
  loading: boolean;
}

@Injectable()
export class PaginationStore extends ComponentStore<PaginationState> {
  constructor(private productService: ProductService) {
    super({ currentPage: 1, pageSize: 10, totalItems: 0, items: [], loading: false });
  }

  // Selectors
  readonly items$ = this.select(state => state.items);
  readonly loading$ = this.select(state => state.loading);
  readonly totalPages$ = this.select(state =>
    Math.ceil(state.totalItems / state.pageSize)
  );

  // Updaters — synchronous state updates
  readonly setPage = this.updater((state, page: number) => ({
    ...state, currentPage: page
  }));

  // Effects — async operations
  readonly loadPage = this.effect<number>(page$ =>
    page$.pipe(
      tap(() => this.patchState({ loading: true })),
      switchMap(page =>
        this.productService.getPage(page).pipe(
          tapResponse(
            ({ items, total }) => this.patchState({ items, totalItems: total, loading: false }),
            () => this.patchState({ loading: false })
          )
        )
      )
    )
  );
}
```

```typescript
// Provide at the component level — destroyed with the component
@Component({
  providers: [PaginationStore],  // local scope
  template: `...`
})
export class ProductListComponent {
  private paginationStore = inject(PaginationStore);

  items = toSignal(this.paginationStore.items$, { initialValue: [] });

  changePage(page: number) {
    this.paginationStore.setPage(page);
    this.paginationStore.loadPage(page);
  }
}
```

---

## Choosing the Right State Management Solution

The decision depends on three factors: **app complexity**, **team size**, and **existing codebase**.

For **simple apps or isolated features** with one or two developers, plain Angular signals in a service (`@Injectable({ providedIn: 'root' })`) are entirely sufficient. The signal-based service pattern shown earlier can handle most state management needs without any library.

For **medium complexity apps** (5–15 feature areas, 2–5 developers), NgRx Signals Store or Elf strike the right balance — enough structure to keep things organized, without the full ceremony of classic NgRx.

For **large enterprise applications** (15+ feature areas, multiple teams), classic NgRx provides the strictest conventions and the best DevTools experience. The additional boilerplate is worthwhile because it enforces consistency across many engineers.

For **component-level complex state** — like a multi-step wizard, a data table with complex filtering, or a draggable canvas — NgRx ComponentStore or a local signal service scoped to that component tree is the right choice, regardless of what you use globally.

A common and effective architecture is using **NgRx (classic or signals) for global app-level state** (authentication, user preferences, global notifications) while using **local signal stores for feature-level state** (the state of the current page's form, a table's filters, etc.). This keeps the global store lean and prevents it from becoming a dumping ground for every piece of state in the app.

---

## Important Points & Best Practices

**Don't use global state for everything.** Server response data that's only displayed on one screen doesn't belong in a global store — it belongs in a component, resolved by a route resolver, or fetched via TanStack Query. Global state is for data that is shared across multiple unrelated parts of the app.

**Colocate state with the feature that owns it.** Rather than one giant global store, use feature stores registered when a feature module or lazy route loads. This keeps state modular and eliminates risk of naming conflicts.

**Avoid storing derived data.** If `totalPrice` can be computed from `items` with a `computed()` or selector, don't store it separately. Storing derived data introduces bugs when the source data changes but the derived data doesn't update properly.

**State persistence (localStorage sync) should be done in an effect or a store plugin**, not inside a reducer or signal update directly. NgRx has `ngrx-store-localstorage`. NGXS has `@ngxs/storage-plugin`. For signals, you can write a `localStorage` sync effect using `effect()` that watches relevant signals and saves them.

**Devtools matter more than you think.** One of the biggest practical advantages of NgRx over plain signals is the Redux DevTools. Being able to replay the sequence of actions that led to a bug cuts debugging time dramatically in complex apps. If you're using pure signals, add some form of logging to compensate.

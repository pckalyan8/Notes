# Phase 14.5 — NgRx Signals Store (Modern Angular)
## signalStore, withState, withComputed, withMethods, withEntities & DevTools

---

## Why the NgRx Signals Store Exists

The classic NgRx store is powerful, but it carries a reputation that is partially deserved: it requires a significant amount of boilerplate. For a typical feature, you create at minimum four separate files — actions, reducer, selectors, and effects — before writing a single line of component code. On large teams with strict conventions, this ceremony is an acceptable trade-off for the clarity and traceability it provides. On smaller teams or for simpler features, it can feel like moving furniture before you can sit down.

The NgRx team recognized this tension and introduced the **NgRx Signals Store** (`@ngrx/signals`) in NgRx 17. It is not a replacement for the classic store — it is a complementary option that occupies the space between "plain signal service" and "full Redux NgRx store." It gives you NgRx's DevTools integration and established patterns while replacing the Actions/Reducers/Selectors/Effects quartet with a unified, declarative API that feels much more like writing a regular Angular service.

Think of the Signals Store as NgRx adapting to Angular's signal era. It is built entirely on Angular signals, composes cleanly with `computed()` and `effect()`, and requires no Zone.js — making it naturally compatible with Angular's zoneless change detection introduced in Angular 20.

---

## Core Concept: The `signalStore()` Function

The entire Signals Store API is built around one function: `signalStore()`. You pass it a series of "features" — functions that each add a specific capability to the store — and it returns an injectable service class. The store is just a service, which means it participates in Angular's DI system exactly like any other service. You can provide it at root level, at the route level, at the component level, or anywhere the DI tree supports.

```typescript
import {
  signalStore,
  withState,
  withComputed,
  withMethods,
  withHooks
} from '@ngrx/signals';
import { computed, inject } from '@angular/core';

// The simplest possible Signals Store — a counter
export const CounterStore = signalStore(
  { providedIn: 'root' },  // optional: makes it a root singleton

  // withState() defines the initial shape and values of the store's state
  withState({ count: 0, step: 1 }),

  // withComputed() adds derived signals — automatically memoized
  withComputed(({ count, step }) => ({
    doubled: computed(() => count() * 2),
    canDecrement: computed(() => count() > 0),
    nextValue: computed(() => count() + step()),
  })),

  // withMethods() adds the "actions" — functions that update state or perform effects
  withMethods((store) => ({
    increment() {
      patchState(store, { count: store.count() + store.step() });
    },
    decrement() {
      if (store.canDecrement()) {
        patchState(store, { count: store.count() - store.step() });
      }
    },
    reset() {
      patchState(store, { count: 0 });
    },
    setStep(step: number) {
      patchState(store, { step });
    }
  }))
);
```

Notice what is absent: no actions file, no reducer file, no selectors file. Everything lives in one coherent definition. The state is declared with `withState()`, derived values with `withComputed()`, and mutations with `withMethods()`. This is the key architectural difference from classic NgRx.

---

## Understanding `withState()` in Depth

`withState()` accepts an initial state object and turns every top-level property into a **DeepSignal** — a signal that also makes its nested properties accessible as signals. This is an important detail that makes working with nested state feel natural.

```typescript
import { signalStore, withState, patchState } from '@ngrx/signals';
import { computed } from '@angular/core';

interface UserProfileState {
  user: {
    id: string;
    name: string;
    email: string;
    preferences: {
      theme: 'light' | 'dark';
      language: string;
      notifications: boolean;
    };
  } | null;
  loading: boolean;
  lastUpdated: number | null;
}

const initialState: UserProfileState = {
  user: null,
  loading: false,
  lastUpdated: null,
};

export const UserProfileStore = signalStore(
  withState<UserProfileState>(initialState),

  withComputed(({ user, loading }) => ({
    // user() returns the whole user object or null
    // user.name() drills into the nested 'name' property directly
    // This only works because withState creates DeepSignals
    isLoggedIn: computed(() => user() !== null),
    displayName: computed(() => user()?.name ?? 'Guest'),
    // Note: accessing nested properties of a potentially null DeepSignal
    // requires the optional chaining pattern
    currentTheme: computed(() => user()?.preferences?.theme ?? 'light'),
    isLoading: computed(() => loading()),
  })),

  withMethods((store) => ({
    setUser(user: UserProfileState['user']) {
      patchState(store, { user, lastUpdated: Date.now() });
    },
    updatePreferences(preferences: Partial<UserProfileState['user']['preferences']>) {
      const currentUser = store.user();
      if (!currentUser) return;
      patchState(store, {
        user: {
          ...currentUser,
          preferences: { ...currentUser.preferences, ...preferences }
        }
      });
    },
    setLoading(loading: boolean) {
      patchState(store, { loading });
    },
    clearUser() {
      patchState(store, { user: null, lastUpdated: null });
    }
  }))
);
```

The `patchState()` function is the primary mutation tool in the Signals Store. It takes the store reference and a partial state object (or a function that receives current state and returns a partial state), and immutably applies the changes — conceptually similar to spreading state in a reducer, but more concise.

```typescript
// patchState with a direct partial object — for simple field updates
patchState(store, { loading: false });

// patchState with an updater function — for updates that depend on current state
patchState(store, (state) => ({
  items: [...state.items, newItem]
}));
```

---

## Understanding `withComputed()` in Depth

`withComputed()` receives an object where each property is a computed signal. The callback receives the store's current signals as its argument, which gives you access to both state signals and other computed signals (if you define them correctly — Angular computes them lazily and memoizes each one).

An important rule: computed signals in `withComputed()` are **lazy** — they are not calculated until something reads them, and they are **memoized** — they only recalculate when one of their signal dependencies has changed. This makes them extremely efficient even for expensive derivations.

```typescript
import { signalStore, withState, withComputed, withMethods, patchState } from '@ngrx/signals';
import { computed } from '@angular/core';

interface OrdersState {
  orders: Order[];
  statusFilter: OrderStatus | 'all';
  searchQuery: string;
  sortBy: 'date' | 'total' | 'status';
  sortDirection: 'asc' | 'desc';
}

export const OrdersStore = signalStore(
  withState<OrdersState>({
    orders: [],
    statusFilter: 'all',
    searchQuery: '',
    sortBy: 'date',
    sortDirection: 'desc',
  }),

  withComputed(({ orders, statusFilter, searchQuery, sortBy, sortDirection }) => {
    // Step 1: Filter by status — this recomputes only when orders() or statusFilter() changes
    const statusFiltered = computed(() => {
      const filter = statusFilter();
      return filter === 'all'
        ? orders()
        : orders().filter(o => o.status === filter);
    });

    // Step 2: Filter by search — depends on statusFiltered and searchQuery
    const searchFiltered = computed(() => {
      const q = searchQuery().toLowerCase().trim();
      if (!q) return statusFiltered();
      return statusFiltered().filter(o =>
        o.id.toLowerCase().includes(q) ||
        o.customerName.toLowerCase().includes(q)
      );
    });

    // Step 3: Sort — depends on searchFiltered and sort settings
    const sorted = computed(() => {
      const field = sortBy();
      const dir = sortDirection();
      return [...searchFiltered()].sort((a, b) => {
        const aVal = a[field];
        const bVal = b[field];
        const comparison = aVal < bVal ? -1 : aVal > bVal ? 1 : 0;
        return dir === 'asc' ? comparison : -comparison;
      });
    });

    // Statistics — computed from the full orders list regardless of filters
    const totalRevenue = computed(() =>
      orders().reduce((sum, o) => sum + o.total, 0)
    );

    const pendingCount = computed(() =>
      orders().filter(o => o.status === 'pending').length
    );

    return {
      // Expose the final derived list and statistics
      displayedOrders: sorted,
      totalRevenue,
      pendingCount,
      isEmpty: computed(() => sorted().length === 0),
    };
  }),

  withMethods((store) => ({
    setFilter(statusFilter: OrderStatus | 'all') {
      patchState(store, { statusFilter });
    },
    setSearch(searchQuery: string) {
      patchState(store, { searchQuery });
    },
    setSort(sortBy: OrdersState['sortBy']) {
      // Toggle direction if clicking the same column; otherwise reset to desc
      const newDir = store.sortBy() === sortBy && store.sortDirection() === 'desc'
        ? 'asc'
        : 'desc';
      patchState(store, { sortBy, sortDirection: newDir });
    },
    addOrder(order: Order) {
      patchState(store, (state) => ({ orders: [order, ...state.orders] }));
    },
  }))
);
```

---

## Understanding `withMethods()` and Async Operations

`withMethods()` is where you define the store's mutation API and any async operations. For async operations involving RxJS (like HTTP calls), NgRx provides the `rxMethod()` helper that creates an RxJS-based method that integrates with the signal system.

```typescript
import { signalStore, withState, withComputed, withMethods, patchState, withHooks } from '@ngrx/signals';
import { rxMethod } from '@ngrx/signals/rxjs-interop';
import { tapResponse } from '@ngrx/operators';
import { inject } from '@angular/core';
import { pipe, switchMap, tap } from 'rxjs';
import { HttpClient } from '@angular/common/http';

export interface Book {
  id: string;
  title: string;
  author: string;
  rating: number;
}

export interface BooksState {
  books: Book[];
  loading: boolean;
  error: string | null;
  searchQuery: string;
}

export const BooksStore = signalStore(
  { providedIn: 'root' },

  withState<BooksState>({
    books: [],
    loading: false,
    error: null,
    searchQuery: '',
  }),

  withComputed(({ books, searchQuery }) => ({
    filteredBooks: computed(() => {
      const q = searchQuery().toLowerCase();
      return q ? books().filter(b => b.title.toLowerCase().includes(q)) : books();
    }),
    totalBooks: computed(() => books().length),
    topRated: computed(() =>
      [...books()].sort((a, b) => b.rating - a.rating).slice(0, 5)
    ),
  })),

  withMethods((store, http = inject(HttpClient)) => ({

    // Synchronous method — simple state update
    setSearchQuery(query: string) {
      patchState(store, { searchQuery: query });
    },

    // Async method using rxMethod — for HTTP calls and other RxJS streams
    // rxMethod creates a method that accepts either a static value, a signal, or an observable
    loadBooks: rxMethod<void>(
      pipe(
        // tap runs before the HTTP call — show the loading state
        tap(() => patchState(store, { loading: true, error: null })),

        // switchMap: cancels any previous in-flight request when called again
        switchMap(() =>
          http.get<Book[]>('/api/books').pipe(
            // tapResponse handles both success and error cases cleanly
            tapResponse({
              next: (books) => patchState(store, { books, loading: false }),
              error: (err: Error) =>
                patchState(store, { loading: false, error: err.message }),
            })
          )
        )
      )
    ),

    // rxMethod with a parameter — the parameter type is inferred from the generic
    deleteBook: rxMethod<string>(
      pipe(
        switchMap((id) =>
          http.delete(`/api/books/${id}`).pipe(
            tapResponse({
              next: () =>
                patchState(store, (state) => ({
                  books: state.books.filter(b => b.id !== id),
                })),
              error: (err: Error) =>
                patchState(store, { error: err.message }),
            })
          )
        )
      )
    ),

    // Regular async method using firstValueFrom (simpler for one-shot operations)
    async createBook(dto: Omit<Book, 'id'>) {
      patchState(store, { loading: true });
      try {
        const newBook = await firstValueFrom(
          http.post<Book>('/api/books', dto)
        );
        patchState(store, (state) => ({
          books: [newBook, ...state.books],
          loading: false,
        }));
        return newBook;
      } catch (err) {
        patchState(store, { loading: false, error: (err as Error).message });
        throw err;
      }
    },
  })),

  // withHooks provides lifecycle callbacks for the store
  withHooks({
    onInit(store) {
      // Automatically load books when the store is first created
      store.loadBooks();
    },
    onDestroy(store) {
      // Cleanup when the store is destroyed (only relevant if scoped to a component)
      console.log('BooksStore destroyed');
    }
  })
);
```

---

## 14.5.1 — Entities with `withEntities()`

The `withEntities()` feature in NgRx Signals provides the same normalized entity management that `@ngrx/entity`'s `EntityAdapter` provides in classic NgRx — but integrated naturally with the signal API.

```typescript
import { signalStore, withState, withComputed, withMethods, withHooks, patchState } from '@ngrx/signals';
import {
  withEntities,
  setAllEntities,
  addEntity,
  updateEntity,
  removeEntity,
  setEntity,
  upsertEntities,
} from '@ngrx/signals/entities';
import { rxMethod } from '@ngrx/signals/rxjs-interop';
import { tapResponse } from '@ngrx/operators';
import { computed, inject } from '@angular/core';
import { pipe, switchMap, tap } from 'rxjs';

export interface Product {
  id: string;
  name: string;
  price: number;
  category: string;
  inStock: boolean;
}

export const ProductsSignalStore = signalStore(
  { providedIn: 'root' },

  // withEntities manages { ids: string[], entityMap: { [id]: Product } }
  // It accepts a type argument for the entity type
  // You can also specify a custom id field: withEntities({ entity: type<Product>(), idKey: 'productId' })
  withEntities<Product>(),

  // Add any extra non-entity state alongside the entity collection
  withState({
    loading: boolean = false,
    error: string | null = null,
    selectedId: string | null = null,
    categoryFilter: string | null = null,
  }),

  withComputed(({ entities, selectedId, categoryFilter }) => ({
    // entities() returns all entities as an array, in insertion order
    filteredProducts: computed(() => {
      const filter = categoryFilter();
      return filter
        ? entities().filter(p => p.category === filter)
        : entities();
    }),

    selectedProduct: computed(() =>
      // Entity lookup is O(1) using the entityMap internally
      entities().find(p => p.id === selectedId()) ?? null
    ),

    inStockCount: computed(() =>
      entities().filter(p => p.inStock).length
    ),

    categories: computed(() =>
      [...new Set(entities().map(p => p.category))].sort()
    ),
  })),

  withMethods((store, http = inject(HttpClient)) => ({

    loadProducts: rxMethod<void>(
      pipe(
        tap(() => patchState(store, { loading: true })),
        switchMap(() =>
          http.get<Product[]>('/api/products').pipe(
            tapResponse({
              // setAllEntities replaces the entire collection atomically
              next: (products) => patchState(store, setAllEntities(products), { loading: false }),
              error: (err: Error) => patchState(store, { loading: false, error: err.message }),
            })
          )
        )
      )
    ),

    addProduct(product: Product) {
      // addEntity adds a single entity — throws if ID already exists
      patchState(store, addEntity(product));
    },

    updateProduct(id: string, changes: Partial<Product>) {
      // updateEntity merges partial changes into the existing entity
      patchState(store, updateEntity({ id, changes }));
    },

    removeProduct(id: string) {
      // removeEntity removes the entity by ID
      patchState(store, removeEntity(id));
    },

    upsertProducts(products: Product[]) {
      // upsertEntities adds new entities and updates existing ones
      patchState(store, upsertEntities(products));
    },

    selectProduct(id: string | null) {
      patchState(store, { selectedId: id });
    },

    setFilter(category: string | null) {
      patchState(store, { categoryFilter: category });
    },
  })),

  withHooks({ onInit: (store) => store.loadProducts() })
);
```

---

## 14.5.2 — Custom Store Features

One of the most powerful aspects of the Signals Store is the ability to create **custom store features** — reusable composable pieces that can be mixed into any store. This is the Signals Store's answer to higher-order reducers and reusable selectors in classic NgRx.

```typescript
import { signalStoreFeature, withState, withMethods, withComputed, patchState } from '@ngrx/signals';
import { computed } from '@angular/core';

// A reusable "pagination" feature that any store can incorporate
export function withPagination(initialPageSize = 10) {
  return signalStoreFeature(
    // This feature adds its own state slice
    withState({
      currentPage: 1,
      pageSize: initialPageSize,
      totalItems: 0,
    }),

    // And its own computed values
    withComputed(({ currentPage, pageSize, totalItems }) => ({
      totalPages: computed(() => Math.ceil(totalItems() / pageSize())),
      hasNextPage: computed(() => currentPage() < Math.ceil(totalItems() / pageSize())),
      hasPreviousPage: computed(() => currentPage() > 1),
      // Offset for API calls: "skip this many items"
      pageOffset: computed(() => (currentPage() - 1) * pageSize()),
    })),

    // And its own methods
    withMethods((store) => ({
      goToPage(page: number) {
        const totalPages = Math.ceil(store.totalItems() / store.pageSize());
        const clampedPage = Math.max(1, Math.min(page, totalPages));
        patchState(store, { currentPage: clampedPage });
      },
      nextPage() {
        if (store.hasNextPage()) {
          patchState(store, { currentPage: store.currentPage() + 1 });
        }
      },
      previousPage() {
        if (store.hasPreviousPage()) {
          patchState(store, { currentPage: store.currentPage() - 1 });
        }
      },
      setTotalItems(totalItems: number) {
        patchState(store, { totalItems, currentPage: 1 });
      },
      setPageSize(pageSize: number) {
        patchState(store, { pageSize, currentPage: 1 });
      },
    }))
  );
}

// A reusable "loading with error" feature
export function withLoadingState() {
  return signalStoreFeature(
    withState({ loading: false, error: null as string | null }),
    withComputed(({ loading, error }) => ({
      isReady: computed(() => !loading() && !error()),
    })),
    withMethods((store) => ({
      startLoading() { patchState(store, { loading: true, error: null }); },
      finishLoading() { patchState(store, { loading: false }); },
      setError(error: string) { patchState(store, { loading: false, error }); },
    }))
  );
}
```

```typescript
// Now compose these features into a store — this is the real power
export const BooksStore = signalStore(
  { providedIn: 'root' },
  withEntities<Book>(),

  // Drop in the reusable features — they bring their own state, computed, and methods
  withPagination(20),
  withLoadingState(),

  withMethods((store, http = inject(HttpClient)) => ({
    loadPage: rxMethod<void>(
      pipe(
        tap(() => store.startLoading()),  // method from withLoadingState
        switchMap(() =>
          http.get<{ books: Book[]; total: number }>(
            `/api/books?page=${store.currentPage()}&limit=${store.pageSize()}`
          ).pipe(
            tapResponse({
              next: ({ books, total }) => {
                patchState(store, setAllEntities(books));
                store.setTotalItems(total);  // method from withPagination
                store.finishLoading();        // method from withLoadingState
              },
              error: (err: Error) => store.setError(err.message),
            })
          )
        )
      )
    ),
  }))
);
```

---

## 14.5.3 — DevTools Integration

The NgRx Signals Store integrates with the Redux DevTools browser extension using the `withDevtools()` feature from `@ngrx/signals/devtools`. This gives you the same time-travel debugging capabilities that classic NgRx provides — you can see the current state, inspect state at any point in time, and (with some limitations) replay state changes.

```typescript
import { withDevtools } from '@ngrx/signals/devtools';

export const BooksStore = signalStore(
  withEntities<Book>(),
  withState({ loading: false }),
  withMethods((store) => ({ /* ... */ })),

  // Add this feature to enable DevTools — each store gets its own name in the DevTools panel
  withDevtools('books-store')
);
```

Because the Signals Store uses a different update model than classic NgRx (there are no discrete "action" events), the DevTools integration shows state snapshots rather than an action log. You can still see what the current state is and inspect how it changed over time.

---

## 14.5.4 — Classic NgRx vs NgRx Signals Store — When to Use Each

Both tools solve the same fundamental problem — shared, reactive application state — but they make different trade-offs. Understanding these trade-offs helps you make the right choice for your specific situation.

The **classic NgRx store** excels when you have a large team where the strict conventions (every state change must be an explicitly defined action) prevent architectural drift. Its action log in Redux DevTools is unmatched for debugging complex, multi-step state interactions. It enforces a clear audit trail: you always know exactly what action caused a particular state change and in what order. For applications where state management is the core complexity — financial systems, complex workflows, collaborative tools — this traceability is invaluable. The cost is boilerplate: four files per feature, each with its own conventions to learn.

The **NgRx Signals Store** excels when you want NgRx-level organization without the ceremony. A single store file replaces the four-file classic pattern. The API feels more like writing a service than writing Redux, which reduces the learning curve for developers new to NgRx. The signal-native architecture integrates seamlessly with Angular's broader signal ecosystem (`computed()`, `effect()`, `toSignal()`). For teams adopting NgRx for the first time, or for features that need shared state but don't have the complexity to justify the full Redux pattern, the Signals Store is often the better starting point.

A practical 2026 recommendation: use the NgRx Signals Store for all new features in new projects. Use classic NgRx when migrating an existing NgRx codebase feature-by-feature, or when a specific feature requires the granular action-level audit trail that classic NgRx provides. Both can coexist in the same application — you can have global authentication state in classic NgRx and a product search store in NgRx Signals Store without any conflict.

---

## Important Points and Best Practices

Every method in `withMethods()` runs synchronously unless you explicitly use `rxMethod()` or `async/await`. This means that state updates via `patchState()` inside a non-async method are immediate and synchronous — there is no delay or scheduling. This is different from classic NgRx effects, which are always asynchronous (they work through the action stream).

The `patchState()` function is always immutable under the hood. Even if you pass it an object that looks like a partial update (`{ loading: false }`), NgRx Signals Store creates a new state snapshot internally. You never need to manually spread state as you do in NgRx reducers.

Scope stores at the right level. Providing a store in `{ providedIn: 'root' }` makes it a global singleton, which is appropriate for truly application-wide state (authentication, notifications, shopping cart). Providing it in a component's `providers` array makes it scoped to that component tree — it is created when the component is created and destroyed when the component is destroyed. Use component-level scoping for feature state that should be cleaned up when the user navigates away.

The `withHooks()` feature is the best place for store initialization logic (calling `loadData()` when the store is created) and cleanup logic (cancelling subscriptions, clearing timers when the store is destroyed). Do not put initialization logic in component `ngOnInit` if the initialization is about loading the store's data — that couples the component to the store's initialization details in a way that makes testing harder.

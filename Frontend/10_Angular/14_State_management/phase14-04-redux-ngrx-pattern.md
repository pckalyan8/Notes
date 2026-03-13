# Phase 14.4 — Redux Pattern / NgRx
## The Cycle, Immutability, Normalized State, Memoization, Effects & Updates

---

## Understanding the Redux Mental Model Before Touching NgRx

One of the most common mistakes Angular developers make is jumping straight into NgRx code without first understanding the Redux *mental model* that NgRx implements. NgRx has a lot of API surface — `createAction`, `createReducer`, `createSelector`, `createEffect`, and more — and it's easy to get lost in the mechanics. But the mechanics are just implementations of a very simple idea.

The Redux idea can be stated in one sentence: **your entire application's mutable state lives in a single JavaScript object (the Store), and the only way to change that object is by dispatching a plain Action that describes what happened, which a pure Reducer function processes to produce a new version of the state.**

Every other concept — selectors, effects, entity adapters, DevTools — is built on top of this single idea. When you truly internalize the idea, the API becomes straightforward. Let's build up from first principles.

---

## 14.4.1 — The Complete Action → Reducer → Store → Selector → Component Cycle

The Redux cycle is a closed loop. Nothing breaks out of it; everything flows in one direction. Understanding each step and why it exists is the foundation of good NgRx architecture.

### Step 1: Something Happens (an Event)

A user clicks a button. An HTTP response arrives. A timer fires. A WebSocket message is received. Something changes in the world that your application cares about. This is the moment you dispatch an **Action**.

An action is the simplest possible data structure: a plain JavaScript object with a mandatory `type` string and an optional payload. The `type` uniquely identifies *what happened*. The payload carries the data associated with the event.

```typescript
// actions.ts
import { createAction, createActionGroup, emptyProps, props } from '@ngrx/store';

// createActionGroup is the modern, recommended way to define related actions
// It namespaces all actions with the source string, preventing type collisions
export const ProductsPageActions = createActionGroup({
  source: 'Products Page',  // becomes '[Products Page]' prefix on every action type
  events: {
    // emptyProps() means the action carries no data — just signals that something happened
    'Open Products Page': emptyProps(),

    // props<T>() defines the shape of the optional payload
    'Select Product': props<{ productId: string }>(),
    'Delete Product': props<{ productId: string }>(),
    'Search Products': props<{ query: string; category?: string }>(),
  }
});

export const ProductsApiActions = createActionGroup({
  source: 'Products API',
  events: {
    'Load Products Success': props<{ products: Product[] }>(),
    'Load Products Failure': props<{ error: string }>(),
    'Create Product Success': props<{ product: Product }>(),
    'Delete Product Success': props<{ productId: string }>(),
  }
});

// Usage: ProductsPageActions.selectProduct({ productId: '42' })
// This creates: { type: '[Products Page] Select Product', productId: '42' }
```

> **Naming Actions as Events, Not Commands:** Actions should describe *what happened*, not *what to do*. Compare `deleteProduct({ id })` (a command — anti-pattern) with `deleteProductClicked({ id })` (an event — what the user did). The distinction matters because the same action might trigger multiple downstream reactions — a reducer update, an HTTP effect, an analytics logging effect. A command-style name implies a single, specific consequence; an event-style name makes no such assumption.

### Step 2: The Reducer Computes New State

When an action is dispatched, NgRx passes it through every registered Reducer. A Reducer is a pure function: it takes the current state and the action, and returns a *new* state object without modifying the old one.

The word "pure" here is technically precise: the function has no side effects (no HTTP calls, no console logs, no `Date.now()` calls), and given the same inputs it always returns the same output. This purity is what makes time-travel debugging and state replay possible.

```typescript
// reducer.ts
import { createReducer, on } from '@ngrx/store';
import { ProductsPageActions, ProductsApiActions } from './products.actions';
import { Product } from '../models/product.model';

// The shape of this feature's slice of state — always define this as an interface
export interface ProductsState {
  products: Product[];
  selectedProductId: string | null;
  searchQuery: string;
  loading: boolean;
  error: string | null;
}

// The starting state before any actions have been dispatched
const initialState: ProductsState = {
  products: [],
  selectedProductId: null,
  searchQuery: '',
  loading: false,
  error: null,
};

export const productsReducer = createReducer(
  initialState,

  // Handle the "Open Products Page" action — signal loading should start
  // The effect will then make the HTTP call and dispatch success/failure
  on(ProductsPageActions.openProductsPage, (state) => ({
    ...state,            // ALWAYS spread existing state to preserve unrelated fields
    loading: true,
    error: null,         // Clear any previous error when retrying
  })),

  // Handle successful data load — replace all products with fresh data
  on(ProductsApiActions.loadProductsSuccess, (state, action) => ({
    ...state,
    // action.products comes from props<{ products: Product[] }>() in the action creator
    products: action.products,
    loading: false,
  })),

  // Handle load failure — preserve existing products, record the error
  on(ProductsApiActions.loadProductsFailure, (state, action) => ({
    ...state,
    loading: false,
    error: action.error,  // action.error comes from props<{ error: string }>()
  })),

  // Handle product selection — simple ID change
  on(ProductsPageActions.selectProduct, (state, { productId }) => ({
    ...state,
    selectedProductId: productId,
  })),

  // Handle search — update the query (the filtering happens in a selector, not here)
  on(ProductsPageActions.searchProducts, (state, { query }) => ({
    ...state,
    searchQuery: query,
  })),

  // Handle successful deletion — filter the product out of the array
  // CRITICAL: filter() creates a new array — it never mutates the original
  on(ProductsApiActions.deleteProductSuccess, (state, { productId }) => ({
    ...state,
    products: state.products.filter(p => p.id !== productId),
    // Clear selection if the deleted product was selected
    selectedProductId: state.selectedProductId === productId
      ? null
      : state.selectedProductId,
  })),

  // Handle successful creation — add the new product to the front of the list
  on(ProductsApiActions.createProductSuccess, (state, { product }) => ({
    ...state,
    // Create a new array — NEVER use push() on the existing one
    products: [product, ...state.products],
  })),
);
```

### Step 3: The Store Holds and Broadcasts State

The Store is NgRx's central singleton that holds the current state tree and makes it observable. When a Reducer produces new state, the Store emits the new state to all active subscriptions. Components and Selectors that have subscribed to the Store automatically receive the update.

You rarely interact with the Store directly in application code — Selectors and `dispatch()` are the primary API surface.

### Step 4: Selectors Derive and Memoize Slices

Components should not subscribe to the entire Store and then filter the data they need — that would cause every component to re-render on every state change anywhere in the application. Selectors solve this by creating **memoized projections** of the state.

A Selector is a pure function that takes the entire state tree and returns a specific slice or derived value. NgRx memoizes selectors — a selector only recalculates if the specific parts of state it depends on have changed. If `productsState.products` hasn't changed since the last render cycle, `selectAllProducts` returns the same object reference it returned last time. Angular's `OnPush` change detection sees an identical reference and skips re-rendering.

```typescript
// selectors.ts
import { createFeatureSelector, createSelector } from '@ngrx/store';
import { ProductsState } from './products.reducer';

// Step 1: Create a "feature selector" — the entry point to this feature's state slice
// 'products' must match the key used when registering the reducer (provideState)
const selectProductsFeature = createFeatureSelector<ProductsState>('products');

// Step 2: Create "child selectors" that project specific fields
// These are memoized — they only recalculate when their inputs change
export const selectAllProducts = createSelector(
  selectProductsFeature,
  (state) => state.products
);

export const selectProductsLoading = createSelector(
  selectProductsFeature,
  (state) => state.loading
);

export const selectProductsError = createSelector(
  selectProductsFeature,
  (state) => state.error
);

export const selectSearchQuery = createSelector(
  selectProductsFeature,
  (state) => state.searchQuery
);

export const selectSelectedProductId = createSelector(
  selectProductsFeature,
  (state) => state.selectedProductId
);

// Step 3: Compose selectors to derive computed data
// This selector depends on TWO other selectors — it re-evaluates only when
// either selectAllProducts OR selectSearchQuery changes
export const selectFilteredProducts = createSelector(
  selectAllProducts,
  selectSearchQuery,
  (products, query) => {
    if (!query.trim()) return products;
    const lowerQuery = query.toLowerCase();
    return products.filter(p =>
      p.name.toLowerCase().includes(lowerQuery) ||
      p.description.toLowerCase().includes(lowerQuery)
    );
  }
);

// Combining three selectors — the projector only runs when any of them change
export const selectSelectedProduct = createSelector(
  selectAllProducts,
  selectSelectedProductId,
  (products, selectedId) => products.find(p => p.id === selectedId) ?? null
);

// A "view model" selector that combines everything a component needs in one call
// This prevents the component from having to call multiple individual selectors
export const selectProductsPageViewModel = createSelector(
  selectFilteredProducts,
  selectProductsLoading,
  selectProductsError,
  selectSelectedProduct,
  (products, loading, error, selectedProduct) => ({
    products,
    loading,
    error,
    selectedProduct,
  })
);
```

### Step 5: The Component Reads and Dispatches

The Component is at the end of the data flow (reading from selectors) and at the start of the next cycle (dispatching actions):

```typescript
// products-page.component.ts
@Component({
  standalone: true,
  template: `
    @let vm = viewModel();
    <!-- @let creates a template variable without needing ngIf/as -->

    @if (vm.loading) { <mat-progress-bar mode="indeterminate" /> }
    @if (vm.error) { <app-error-banner [message]="vm.error" /> }

    <app-search-bar
      [value]="vm.selectedProduct?.id"
      (search)="onSearch($event)"
    />

    <app-products-grid
      [products]="vm.products"
      [selectedId]="vm.selectedProduct?.id"
      (select)="onSelect($event)"
      (delete)="onDelete($event)"
    />

    @if (vm.selectedProduct) {
      <app-product-detail [product]="vm.selectedProduct" />
    }
  `
})
export class ProductsPageComponent {
  private store = inject(Store);

  // selectSignal() returns a signal — preferred in Angular 17+ for template use
  // It integrates with Angular's signal-based change detection
  viewModel = this.store.selectSignal(selectProductsPageViewModel);

  constructor() {
    // Dispatch the initialization action in the constructor
    // (or in ngOnInit for class-based lifecycle hooks)
    this.store.dispatch(ProductsPageActions.openProductsPage());
  }

  onSearch(query: string) {
    this.store.dispatch(ProductsPageActions.searchProducts({ query }));
  }

  onSelect(productId: string) {
    this.store.dispatch(ProductsPageActions.selectProduct({ productId }));
  }

  onDelete(productId: string) {
    this.store.dispatch(ProductsPageActions.deleteProduct({ productId }));
  }
}
```

---

## 14.4.2 — Immutable State Updates in Depth

Immutability means that you never modify an existing state object. Instead, you always create a new object that incorporates the changes. This sounds restrictive, but it enables the performance optimization that makes large Angular applications fast: **reference equality checks**.

Angular's `OnPush` change detection and signal-based reactivity work by comparing object references — if the reference is the same, nothing changed and the component does not re-render. If you mutate state in place, the reference stays the same even though the values changed, and the UI never updates. This is why immutability is not optional in Redux-based architectures.

### The Spread Operator Pattern

The spread operator (`...`) is the standard tool for immutable state updates:

```typescript
// WRONG — mutating state directly. This breaks change detection.
on(SomeAction, (state) => {
  state.count += 1;  // ❌ Mutates the existing state object
  return state;      // The reference is the same — Angular won't detect this
}),

// CORRECT — creating a new object with the updated field
on(SomeAction, (state) => ({
  ...state,      // ✅ Copies all existing fields
  count: state.count + 1,  // Overwrites just the 'count' field in the new object
})),
```

```typescript
// WRONG — mutating an array with push()
on(AddItemAction, (state, { item }) => {
  state.items.push(item);  // ❌ Mutates the array — same reference
  return state;
}),

// CORRECT — creating a new array with the spread operator
on(AddItemAction, (state, { item }) => ({
  ...state,
  items: [...state.items, item],  // ✅ Creates a new array containing all old items plus the new one
})),
```

```typescript
// WRONG — mutating a nested object
on(UpdateUserAction, (state, { userId, changes }) => {
  state.users[userId] = { ...state.users[userId], ...changes };  // ❌ Mutates the nested object
  return state;
}),

// CORRECT — immutably updating a nested value
on(UpdateUserAction, (state, { userId, changes }) => ({
  ...state,
  users: {
    ...state.users,                        // ✅ New outer object
    [userId]: { ...state.users[userId], ...changes }  // ✅ New inner object
  }
})),
```

For deeply nested state updates, the spread operator can become verbose. The Immer library (used internally by NgRx Entity) allows you to write mutations as if they were direct (using a special "draft" proxy), while Immer handles creating the new immutable state under the hood.

---

## 14.4.3 — Normalized State with @ngrx/entity

When your state contains collections of entities (products, users, orders), storing them as flat arrays creates performance problems. Finding a product by ID requires iterating the entire array — O(n) lookup. Removing a product requires `filter()` — another full iteration.

Normalized state stores entities in a dictionary (`{ [id: string]: Entity }`) with a separate array of IDs for ordering. Lookups become O(1), and the `@ngrx/entity` package provides an adapter that manages this structure automatically.

```typescript
import { EntityAdapter, EntityState, createEntityAdapter } from '@ngrx/entity';

// EntityState<T> provides the base shape: { ids: string[], entities: { [id]: T } }
export interface ProductsState extends EntityState<Product> {
  // Add any non-entity state fields here
  selectedProductId: string | null;
  loading: boolean;
  error: string | null;
  searchQuery: string;
}

// The adapter manages the normalized data structure for you
const adapter: EntityAdapter<Product> = createEntityAdapter<Product>({
  selectId: (product) => product.id,  // which field is the unique identifier?
  sortComparer: (a, b) => a.name.localeCompare(b.name),  // optional default sort
});

const initialState: ProductsState = adapter.getInitialState({
  // getInitialState sets up { ids: [], entities: {} } and merges your extra fields
  selectedProductId: null,
  loading: false,
  error: null,
  searchQuery: '',
});

export const productsReducer = createReducer(
  initialState,

  on(ProductsApiActions.loadProductsSuccess, (state, { products }) =>
    // setAll replaces the entire collection atomically
    adapter.setAll(products, { ...state, loading: false })
  ),

  on(ProductsApiActions.createProductSuccess, (state, { product }) =>
    // addOne inserts a single entity — IDs and entities are both updated
    adapter.addOne(product, state)
  ),

  on(ProductsApiActions.updateProductSuccess, (state, { product }) =>
    // updateOne merges partial changes into the existing entity
    adapter.updateOne({ id: product.id, changes: product }, state)
  ),

  on(ProductsApiActions.deleteProductSuccess, (state, { productId }) =>
    // removeOne removes the entity and its ID from both arrays
    adapter.removeOne(productId, state)
  ),

  on(ProductsApiActions.loadManyProductsSuccess, (state, { products }) =>
    // upsertMany — add or update many entities at once
    adapter.upsertMany(products, state)
  ),
);

// The adapter generates ready-made selectors for the normalized collection
const { selectAll, selectEntities, selectIds, selectTotal } =
  adapter.getSelectors();

const selectProductsFeature = createFeatureSelector<ProductsState>('products');

// Compose with the feature selector to make them usable
export const selectAllProducts = createSelector(selectProductsFeature, selectAll);
export const selectProductEntities = createSelector(selectProductsFeature, selectEntities);
export const selectTotalProducts = createSelector(selectProductsFeature, selectTotal);

// With normalized state, looking up by ID is trivially fast
export const selectProductById = (id: string) => createSelector(
  selectProductEntities,
  (entities) => entities[id]  // O(1) dictionary lookup
);
```

---

## 14.4.4 — Effects for Async Operations and Side Effect Management

Reducers are pure functions. They cannot make HTTP calls, access localStorage, navigate to a different route, or do anything asynchronous. All of those operations are **side effects**, and they belong in **Effects**.

An Effect listens to the stream of dispatched actions, filters for the actions it cares about, performs a side effect (usually an HTTP call), and then dispatches a new action reflecting the outcome.

```typescript
// products.effects.ts
import { inject, Injectable } from '@angular/core';
import { Actions, createEffect, ofType } from '@ngrx/effects';
import { ProductsPageActions, ProductsApiActions } from './products.actions';
import { ProductsService } from '../products.service';
import { Router } from '@angular/router';
import { catchError, concatMap, map, switchMap, tap } from 'rxjs/operators';
import { of } from 'rxjs';

@Injectable()
export class ProductsEffects {
  // inject() is the modern way to access DI in effects
  private actions$ = inject(Actions);
  private productsService = inject(ProductsService);
  private router = inject(Router);

  // Load products when the page opens
  // switchMap: if "openProductsPage" is dispatched again while loading,
  // the previous HTTP request is cancelled and a new one starts
  loadProducts$ = createEffect(() =>
    this.actions$.pipe(
      ofType(ProductsPageActions.openProductsPage),
      switchMap(() =>
        this.productsService.getAll().pipe(
          map(products => ProductsApiActions.loadProductsSuccess({ products })),
          catchError(error =>
            of(ProductsApiActions.loadProductsFailure({ error: error.message }))
          )
        )
      )
    )
  );

  // Delete a product
  // concatMap: if multiple deletes are dispatched rapidly,
  // they are queued and executed one after another (preserving order)
  deleteProduct$ = createEffect(() =>
    this.actions$.pipe(
      ofType(ProductsPageActions.deleteProduct),
      concatMap(({ productId }) =>
        this.productsService.delete(productId).pipe(
          map(() => ProductsApiActions.deleteProductSuccess({ productId })),
          catchError(error =>
            of(ProductsApiActions.loadProductsFailure({ error: error.message }))
          )
        )
      )
    )
  );

  // Navigate after successful creation
  // { dispatch: false } means this effect does not dispatch another action
  // It just performs a navigation as a side effect
  navigateAfterCreate$ = createEffect(
    () =>
      this.actions$.pipe(
        ofType(ProductsApiActions.createProductSuccess),
        tap(({ product }) => this.router.navigate(['/products', product.id]))
      ),
    { dispatch: false }
  );

  // Log all actions to an analytics service (non-dispatching)
  logAnalytics$ = createEffect(
    () =>
      this.actions$.pipe(
        ofType(
          ProductsPageActions.selectProduct,
          ProductsPageActions.deleteProduct,
        ),
        tap(action => this.analyticsService.track(action.type, action))
      ),
    { dispatch: false }
  );
}
```

**Choosing the right flattening operator in effects** is critical. `switchMap` is for operations where you want to cancel a previous in-flight request when a new action arrives — ideal for search, page loads, and any "fetch the latest" scenario. `concatMap` is for operations that must complete in sequence — useful for ordered writes like queue processing and delete operations. `mergeMap` is for operations that are truly independent and can run in parallel — rare in practice, useful for analytics events. `exhaustMap` is for operations where you want to ignore new actions until the current one completes — useful for form submissions where clicking twice should be a no-op.

---

## 14.4.5 — Optimistic Updates vs Pessimistic Updates

Both patterns deal with the question of *when* to update the UI in response to an action that requires an async operation (like an HTTP call to delete a record).

### Pessimistic Updates — Wait for Confirmation

In a pessimistic update, you wait for the server to confirm the operation before updating the UI. The UI stays in a loading/pending state while the request is in flight, then updates when success arrives.

This is the default approach and appropriate for most situations — it is safe, simple, and avoids the complexity of rollbacks. The downside is that the user perceives latency: clicking "Delete" doesn't remove the item immediately; it shows a spinner until the server responds.

The reducer shown earlier (handling `deleteProductSuccess`) is a pessimistic update — the product is only removed from the list after the API confirms deletion.

### Optimistic Updates — Assume Success, Rollback on Failure

In an optimistic update, you update the UI *immediately* when the user takes an action, then make the API call in the background. If the API call succeeds, nothing changes — the UI is already correct. If the API call fails, you *roll back* the UI to its previous state and show an error.

This approach feels significantly more responsive — the UI reacts instantly to user interactions — but requires careful rollback logic.

```typescript
// Optimistic delete — we immediately remove the product from the list,
// then make the API call. If it fails, we restore it.

// In the actions file:
export const ProductsPageActions = createActionGroup({
  source: 'Products Page',
  events: {
    // Dispatch with the full product so we can restore it on failure
    'Delete Product Optimistic': props<{ product: Product }>(),
    'Delete Product Rollback': props<{ product: Product }>(),
  }
});

// In the reducer:
on(ProductsPageActions.deleteProductOptimistic, (state, { product }) =>
  // Immediately remove the product — no loading state needed
  adapter.removeOne(product.id, state)
),

on(ProductsPageActions.deleteProductRollback, (state, { product }) =>
  // Restore the product if the API call failed
  adapter.addOne(product, state)
),

// In the effect:
deleteProductOptimistic$ = createEffect(() =>
  this.actions$.pipe(
    ofType(ProductsPageActions.deleteProductOptimistic),
    // mergeMap here because deletes are independent and can be parallel
    mergeMap(({ product }) =>
      this.productsService.delete(product.id).pipe(
        // Success — no UI change needed, already done
        map(() => ProductsApiActions.deleteProductSuccess({ productId: product.id })),
        catchError(() =>
          // Failure — dispatch rollback action to restore the product
          of(ProductsPageActions.deleteProductRollback({ product }))
        )
      )
    )
  )
);
```

Optimistic updates are most appropriate for operations that very rarely fail — toggle actions (mark as read, like a post), low-stakes list reorderings, and similar. They are less appropriate for critical operations (payment processing, account deletion) where failure must be visibly communicated before the user believes the operation completed.

---

## Important Points and Best Practices

The single biggest NgRx mistake is treating actions as commands rather than events. An action that says `{ type: '[Products API] Load Products' }` is both a command ("go fetch products") AND the signal for reducers to update loading state AND the trigger for analytics to log a "products viewed" event. If you name it as a command with a single implied recipient, you lose this flexibility.

Keep reducers absolutely pure. The moment you put a `console.log`, a `new Date()`, a `Math.random()`, or any HTTP call inside a reducer, you have broken the predictability that makes Redux debugging possible. All impure operations belong in effects. Reducers are only allowed to read the state, read the action payload, and compute a new state.

Selector composition is the key to performance. Build simple, single-purpose selectors and compose them into more complex selectors rather than writing large selectors that do many things. The memoization happens at each level — a composed selector that depends on two other selectors only re-evaluates when one of those two selectors produces a new value. This granular memoization prevents unnecessary re-renders even in large, frequently-updating state trees.

Registering reducers lazily (via `provideState` in route providers) instead of eagerly (in the root `provideStore`) keeps the initial store small and ensures that feature state is only loaded when that feature is actually used. This is essential for large applications with many features — you should not be initializing the checkout state when the user is on the product listing page.

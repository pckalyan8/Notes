# Phase 13.3 — NgRx: Redux Pattern for Angular
## Complete Guide: Store, Actions, Reducers, Effects, Selectors & NgRx Signals

---

## Why NgRx Exists: The Problem It Solves

Before NgRx, Angular applications managed shared state using services with `BehaviorSubject`. This works fine for small apps, but as an application grows, you face problems: data flows in multiple directions, components mutate state in unpredictable ways, and debugging becomes extremely hard because you don't know *who* changed *what* and *when*.

NgRx solves this by enforcing **unidirectional data flow** and **immutable state**, borrowed from the Redux pattern created by Dan Abramov. The core idea is:

- There is **one single source of truth** (the Store) for your entire application's state
- State is **read-only** — you can never mutate it directly
- State changes only happen by dispatching **Actions** (plain objects describing what happened)
- **Reducers** (pure functions) compute the new state based on the action
- **Selectors** provide optimized, memoized slices of state to components
- **Effects** handle side effects (HTTP calls, routing, etc.) in isolation

This makes every state change **predictable**, **traceable** (via Redux DevTools), and **testable**.

---

## Installation

```bash
npm install @ngrx/store @ngrx/effects @ngrx/entity @ngrx/router-store
npm install @ngrx/store-devtools --save-dev
```

---

## The Data Flow — The NgRx Cycle

Understanding the cycle is everything. Here it is from the user's perspective:

```
User clicks "Delete Item"
  → Component dispatches Action: deleteItem({ id: 42 })
  → Reducer handles action → returns new state without item 42
  → Store emits the new state
  → Selectors compute the affected slices (e.g., itemList)
  → Component re-renders with updated list
  → (Side effect) Effect handles HTTP call to API to persist the deletion
```

This flow **never reverses**. Components never write to state directly. They only read (via selectors) and write by dispatching actions.

---

## 1. Setting Up the Store

```typescript
// main.ts — standalone Angular app
import { provideStore } from '@ngrx/store';
import { provideEffects } from '@ngrx/effects';
import { provideStoreDevtools } from '@ngrx/store-devtools';
import { provideRouterStore } from '@ngrx/router-store';
import { booksReducer } from './books/store/books.reducer';
import { BookEffects } from './books/store/books.effects';

bootstrapApplication(AppComponent, {
  providers: [
    provideStore({ books: booksReducer }),  // Register feature reducers
    provideEffects([BookEffects]),           // Register effects
    provideRouterStore(),                    // Sync router state into store
    provideStoreDevtools({
      maxAge: 25,             // Keep last 25 state snapshots
      logOnly: !isDevMode(), // Only log in dev mode, disable in production
    }),
  ]
});
```

---

## 2. Actions — Describing What Happened

Actions are **plain objects** with a required `type` property and an optional `props` payload. The `createAction` helper ensures type safety.

There is an important naming convention: actions should be named from the **user's or system's perspective**, not a command. Think of them as **events** that happened, not commands to execute:

```typescript
// books/store/books.actions.ts
import { createAction, createActionGroup, emptyProps, props } from '@ngrx/store';
import { Book } from '../models/book.model';
import { HttpErrorResponse } from '@angular/common/http';

// The modern approach: createActionGroup groups related actions
// and automatically namespaces them with [Books API], [Books Page], etc.
export const BooksApiActions = createActionGroup({
  source: 'Books API',
  events: {
    // emptyProps() means the action carries no data
    'Load Books': emptyProps(),

    // props<T>() defines the shape of the action's payload
    'Load Books Success': props<{ books: Book[] }>(),
    'Load Books Failure': props<{ error: HttpErrorResponse }>(),

    'Create Book Success': props<{ book: Book }>(),
    'Delete Book Success': props<{ id: string }>(),
  }
});

export const BooksPageActions = createActionGroup({
  source: 'Books Page',
  events: {
    'Open Books Page': emptyProps(),
    'Select Book': props<{ id: string }>(),
    'Create Book': props<{ title: string; author: string }>(),
    'Delete Book': props<{ id: string }>(),
  }
});
```

`createActionGroup` generates action creators like `BooksApiActions.loadBooks()`, `BooksApiActions.loadBooksSuccess({ books })`, etc. The type string becomes `'[Books API] Load Books'`.

> **Best Practice — Action Hygiene:** Define *many, specific* actions rather than a few generic ones. An action like `updateState({ data })` is an anti-pattern. Instead use `loadBooksSuccess({ books })`, `selectBook({ id })`, `removeBook({ id })`. This makes the Redux DevTools log readable and meaningful.

---

## 3. State Shape & Reducers

A reducer is a **pure function** that takes the current state and an action, and returns the **new state** — without ever modifying the original state.

```typescript
// books/store/books.reducer.ts
import { createReducer, on } from '@ngrx/store';
import { BooksApiActions, BooksPageActions } from './books.actions';
import { Book } from '../models/book.model';

// Define the shape of this feature's state
export interface BooksState {
  books: Book[];
  selectedId: string | null;
  loading: boolean;
  error: string | null;
}

// Initial state — what state looks like before any actions are dispatched
const initialState: BooksState = {
  books: [],
  selectedId: null,
  loading: false,
  error: null,
};

export const booksReducer = createReducer(
  initialState,

  // When Load Books is dispatched, mark loading as true
  on(BooksApiActions.loadBooks, (state) => ({
    ...state,        // spread existing state — never mutate!
    loading: true,
    error: null,
  })),

  // When books arrive from the API, replace the books array
  on(BooksApiActions.loadBooksSuccess, (state, { books }) => ({
    ...state,
    books,           // ES shorthand for books: books
    loading: false,
  })),

  // When the API call fails, store the error
  on(BooksApiActions.loadBooksFailure, (state, { error }) => ({
    ...state,
    loading: false,
    error: error.message,
  })),

  // When a book is selected, update the selectedId
  on(BooksPageActions.selectBook, (state, { id }) => ({
    ...state,
    selectedId: id,
  })),

  // When a book is deleted successfully, filter it out
  on(BooksApiActions.deleteBookSuccess, (state, { id }) => ({
    ...state,
    books: state.books.filter(book => book.id !== id),
  })),

  // When a new book is created, prepend it to the list
  on(BooksApiActions.createBookSuccess, (state, { book }) => ({
    ...state,
    books: [book, ...state.books],
  })),
);
```

> **Critical rule:** Never mutate the state object. Always use the spread operator (`...state`) or array methods like `filter`, `map` that return *new* arrays. If you mutate state directly, NgRx and Angular's change detection won't detect the change, causing bugs that are incredibly hard to debug.

### Registering Feature State (Lazy Loading)

For lazy-loaded feature modules or standalone routes, you register the reducer with the feature:

```typescript
// books.routes.ts
import { provideState } from '@ngrx/store';

export const BOOKS_ROUTES: Routes = [{
  path: 'books',
  loadComponent: () => import('./books.component').then(m => m.BooksComponent),
  providers: [
    provideState('books', booksReducer),  // only loaded when route is visited
    provideEffects([BookEffects]),
  ]
}];
```

---

## 4. Selectors — Reading State Efficiently

Selectors are pure functions that compute derived data from the store. NgRx memoizes them — a selector only recomputes when its input changes. This prevents unnecessary re-renders.

```typescript
// books/store/books.selectors.ts
import { createFeatureSelector, createSelector } from '@ngrx/store';
import { BooksState } from './books.reducer';

// Step 1: Create a feature selector to get the feature slice
const selectBooksFeature = createFeatureSelector<BooksState>('books');

// Step 2: Create derived selectors using createSelector
// These are memoized — they only recompute when their inputs change
export const selectAllBooks = createSelector(
  selectBooksFeature,
  (state) => state.books  // projector function
);

export const selectBooksLoading = createSelector(
  selectBooksFeature,
  (state) => state.loading
);

export const selectSelectedBookId = createSelector(
  selectBooksFeature,
  (state) => state.selectedId
);

// Combining multiple selectors to compute derived data
// This only recomputes when either selectAllBooks or selectSelectedBookId changes
export const selectSelectedBook = createSelector(
  selectAllBooks,
  selectSelectedBookId,
  (books, selectedId) => books.find(b => b.id === selectedId) ?? null
);

// Selector with a parameter (selector factory)
export const selectBookById = (id: string) => createSelector(
  selectAllBooks,
  (books) => books.find(b => b.id === id)
);

// Complex derived computation
export const selectBooksStats = createSelector(
  selectAllBooks,
  (books) => ({
    total: books.length,
    averageRating: books.reduce((sum, b) => sum + b.rating, 0) / books.length || 0,
    topRated: books.filter(b => b.rating >= 4.5),
  })
);
```

```typescript
// Using selectors in a component
@Component({
  template: `
    @if (loading()) {
      <mat-progress-bar mode="indeterminate" />
    }

    @for (book of books(); track book.id) {
      <app-book-card [book]="book" />
    }
  `
})
export class BooksListComponent {
  private store = inject(Store);

  // select() returns an Observable — toSignal converts to a signal
  books = toSignal(this.store.select(selectAllBooks), { initialValue: [] });
  loading = toSignal(this.store.select(selectBooksLoading), { initialValue: false });

  // Or use selectSignal() for a slightly cleaner syntax
  stats = this.store.selectSignal(selectBooksStats);

  ngOnInit() {
    this.store.dispatch(BooksApiActions.loadBooks());
  }
}
```

---

## 5. Effects — Handling Side Effects

Effects are where async operations happen — API calls, navigation, localStorage, etc. An Effect listens to the action stream and, when a matching action is dispatched, performs a side effect and optionally dispatches a new action.

```typescript
// books/store/books.effects.ts
import { inject, Injectable } from '@angular/core';
import { Actions, createEffect, ofType } from '@ngrx/effects';
import { catchError, map, switchMap, tap } from 'rxjs/operators';
import { of } from 'rxjs';
import { BooksApiActions, BooksPageActions } from './books.actions';
import { BooksService } from '../books.service';
import { Router } from '@angular/router';

@Injectable()
export class BookEffects {
  private actions$ = inject(Actions);
  private booksService = inject(BooksService);
  private router = inject(Router);

  // Effect 1: Load books from API when the page opens
  loadBooks$ = createEffect(() =>
    this.actions$.pipe(
      ofType(BooksApiActions.loadBooks),  // only react to this action
      switchMap(() =>                     // switchMap cancels previous request if action fires again
        this.booksService.getAll().pipe(
          map(books => BooksApiActions.loadBooksSuccess({ books })),
          catchError(error => of(BooksApiActions.loadBooksFailure({ error })))
        )
      )
    )
  );

  // Effect 2: Delete a book — uses concatMap to preserve order
  deleteBook$ = createEffect(() =>
    this.actions$.pipe(
      ofType(BooksPageActions.deleteBook),
      concatMap(({ id }) =>
        this.booksService.delete(id).pipe(
          map(() => BooksApiActions.deleteBookSuccess({ id })),
          catchError(error => of(BooksApiActions.loadBooksFailure({ error })))
        )
      )
    )
  );

  // Effect 3: Navigate after creation — { dispatch: false } means no action is dispatched
  navigateAfterCreate$ = createEffect(() =>
    this.actions$.pipe(
      ofType(BooksApiActions.createBookSuccess),
      tap(({ book }) => this.router.navigate(['/books', book.id]))
    ),
    { dispatch: false }  // This effect doesn't dispatch another action
  );
}
```

> **switchMap vs concatMap vs mergeMap in effects:** Use `switchMap` when you want to cancel previous requests (search, page load). Use `concatMap` when order matters and requests must complete sequentially (delete, update). Use `mergeMap` when requests are independent and can run in parallel.

---

## 6. @ngrx/entity — Normalized Collection Management

When you're managing a collection of items (users, books, products), normalizing them by ID is far more efficient than using arrays. `@ngrx/entity` provides an `EntityAdapter` that handles this automatically.

```typescript
// books/store/books.reducer.ts — with Entity
import { EntityAdapter, EntityState, createEntityAdapter } from '@ngrx/entity';

export interface BooksState extends EntityState<Book> {
  // EntityState adds: ids: string[], entities: { [id: string]: Book }
  loading: boolean;
  error: string | null;
}

// Create adapter — tells NgRx which field is the unique identifier
const adapter: EntityAdapter<Book> = createEntityAdapter<Book>({
  selectId: (book) => book.id,
  sortComparer: (a, b) => a.title.localeCompare(b.title), // optional default sort
});

const initialState: BooksState = adapter.getInitialState({
  loading: false,
  error: null,
});

export const booksReducer = createReducer(
  initialState,

  on(BooksApiActions.loadBooksSuccess, (state, { books }) =>
    // setAll replaces all entities with the new list
    adapter.setAll(books, { ...state, loading: false })
  ),

  on(BooksApiActions.createBookSuccess, (state, { book }) =>
    // addOne inserts a single entity
    adapter.addOne(book, state)
  ),

  on(BooksApiActions.updateBookSuccess, (state, { book }) =>
    // updateOne merges changes into an existing entity
    adapter.updateOne({ id: book.id, changes: book }, state)
  ),

  on(BooksApiActions.deleteBookSuccess, (state, { id }) =>
    // removeOne removes by ID
    adapter.removeOne(id, state)
  ),
);

// The adapter also generates selectors for you
const { selectAll, selectEntities, selectIds, selectTotal } =
  adapter.getSelectors();

const selectBooksFeature = createFeatureSelector<BooksState>('books');

export const selectAllBooks = createSelector(selectBooksFeature, selectAll);
export const selectBookEntities = createSelector(selectBooksFeature, selectEntities);
export const selectTotalBooks = createSelector(selectBooksFeature, selectTotal);
```

---

## 7. @ngrx/router-store — Router State in the Store

Syncing Angular Router state into the NgRx Store means you can write selectors that depend on the current URL, and effects that react to navigation changes.

```typescript
// Selecting route params
import { getRouterSelectors } from '@ngrx/router-store';

export const {
  selectCurrentRoute,
  selectQueryParams,
  selectQueryParam,
  selectRouteParams,
  selectRouteParam,
  selectRouteData,
  selectUrl,
} = getRouterSelectors();

// Usage in a component or effect
const selectedBookId$ = store.select(selectRouteParam('id'));
```

---

## 8. NgRx Signals Store (`@ngrx/signals`) — The Modern Approach

The NgRx Signals Store is the **recommended approach for new Angular projects in 2025/2026**. It replaces the classic Store/Actions/Reducers/Selectors pattern with a simpler, signal-based API that still integrates with NgRx DevTools.

```bash
npm install @ngrx/signals
```

```typescript
// books/store/books.store.ts
import { signalStore, withState, withComputed, withMethods, withHooks, patchState } from '@ngrx/signals';
import { withEntities, setAllEntities, removeEntity, addEntity } from '@ngrx/signals/entities';
import { inject, computed } from '@angular/core';
import { rxMethod } from '@ngrx/signals/rxjs-interop';
import { tapResponse } from '@ngrx/operators';
import { pipe, switchMap } from 'rxjs';
import { BooksService } from './books.service';
import { Book } from './book.model';

// Define extra state beyond entities
interface BooksExtraState {
  selectedId: string | null;
  loading: boolean;
  error: string | null;
}

export const BooksStore = signalStore(
  { providedIn: 'root' },  // or provide at component level

  // Manage a collection of Books normalized by ID
  withEntities<Book>(),

  // Extra state signals
  withState<BooksExtraState>({
    selectedId: null,
    loading: false,
    error: null,
  }),

  // Computed signals derived from state — automatically memoized
  withComputed(({ entities, selectedId }) => ({
    selectedBook: computed(() =>
      entities().find(b => b.id === selectedId()) ?? null
    ),
    topRated: computed(() =>
      entities().filter(b => b.rating >= 4.5)
    ),
    totalBooks: computed(() => entities().length),
  })),

  // Methods — the "actions + reducers" merged into one
  withMethods((store, booksService = inject(BooksService)) => ({
    // Simple synchronous method
    selectBook(id: string) {
      patchState(store, { selectedId: id });
    },

    // Async method using rxMethod (handles RxJS integration)
    loadBooks: rxMethod<void>(
      pipe(
        tap(() => patchState(store, { loading: true, error: null })),
        switchMap(() =>
          booksService.getAll().pipe(
            tapResponse({
              next: (books) => patchState(store, setAllEntities(books), { loading: false }),
              error: (error: HttpErrorResponse) =>
                patchState(store, { loading: false, error: error.message }),
            })
          )
        )
      )
    ),

    deleteBook: rxMethod<string>(
      pipe(
        switchMap((id) =>
          booksService.delete(id).pipe(
            tapResponse({
              next: () => patchState(store, removeEntity(id)),
              error: (error: HttpErrorResponse) => patchState(store, { error: error.message }),
            })
          )
        )
      )
    ),
  })),

  // Lifecycle hooks — called when the store is initialized/destroyed
  withHooks({
    onInit(store) {
      store.loadBooks();  // auto-load on initialization
    }
  })
);
```

```typescript
// Using the Signal Store in a component
@Component({
  providers: [BooksStore],  // or use providedIn: 'root' in the store
  template: `
    @if (store.loading()) {
      <mat-spinner />
    }

    @for (book of store.entities(); track book.id) {
      <app-book-card
        [book]="book"
        [selected]="book.id === store.selectedId()"
        (select)="store.selectBook(book.id)"
        (delete)="store.deleteBook(book.id)"
      />
    }

    <p>Total: {{ store.totalBooks() }} books</p>
  `
})
export class BooksPageComponent {
  store = inject(BooksStore);
}
```

---

## Classic NgRx vs NgRx Signals — When To Use Each

The classic NgRx pattern (Actions/Reducers/Selectors/Effects) has been the standard for years and has powerful DevTools integration. The NgRx Signals Store is newer but dramatically reduces boilerplate and integrates naturally with Angular's signal system.

Use **classic NgRx** when you have a large, existing NgRx application, need very granular DevTools action logging (each action visible separately), or when working in a team already familiar with the Redux pattern. Its strict separation makes large-team code organization easier.

Use **NgRx Signals Store** for new projects, smaller-to-medium apps, or when your team finds the classic pattern too verbose. It has less boilerplate, feels more like writing a service, and works naturally with Angular signals throughout your app.

---

## Redux DevTools Integration

Install the Redux DevTools browser extension for Chrome/Firefox. When `provideStoreDevtools()` is configured, you get:

- A time-travel debugger — jump back to any past state and replay actions
- A log of every dispatched action with its payload
- A tree view of the current store state
- Diffing to see exactly what changed between states

This alone makes NgRx invaluable for debugging complex state bugs in production.

---

## Important Points & Best Practices

**Keep your state as flat as possible.** Deeply nested state objects (`state.user.profile.settings.theme`) are hard to update immutably and hard to select efficiently. Prefer flat structures and use `@ngrx/entity` for collections.

**Never put non-serializable data in the store.** Avoid storing `Date` objects, class instances, functions, or `Observable` references. The store should contain plain JSON-compatible data. NgRx DevTools will warn you if you do this.

**Selector memoization is not magic.** A selector memoizes based on reference equality. If you return a new object literal inside a selector (like `{ total: books.length }`), it will always be considered "changed" even if the values are the same. Pull stable computations to their own selectors and combine them.

**Effects should only handle side effects.** Never put business logic that transforms data inside an effect. Data transformation belongs in the reducer or selector. Effects should be thin: listen, call API, dispatch success/failure.

**Use `createActionGroup` instead of individual `createAction` calls.** It enforces consistent naming, reduces typos, and groups related actions logically.

**The `dispatch: false` option on an effect is important.** If an effect doesn't dispatch a new action (e.g., it only navigates or logs), you must set `{ dispatch: false }`. Without it, NgRx will try to dispatch the effect's observable emissions as actions, causing an infinite loop.

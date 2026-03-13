# Phase 14.1 — State Management Fundamentals
## What Is State, Why It Matters, and How to Think About It

---

## What Is Application State?

Before you can manage state well, you need an extremely clear mental model of what state *is*. State is any data that, when it changes, should cause some part of your user interface to update. That's the working definition, and it's more profound than it sounds.

Think about a simple online bookstore. At any given moment your application holds: a list of books fetched from the API, the user's shopping cart contents, whether the cart panel is currently open or closed, the search term the user just typed, whether a loading spinner should be visible, what page of results is currently shown, and the currently logged-in user's profile. All of that is state. Every piece of data your application remembers — even for a fraction of a second — is state.

State is what makes your application feel *alive*. Without it, every interaction would produce a blank page. With poorly managed state, your application becomes a bug-breeding ground where the cart says "3 items" but the checkout page shows 2, or a loading spinner stays on screen forever because the "loaded" state was set in one place but checked in another.

The core challenge of frontend development in 2026 is not rendering — Angular, React, and Vue all render HTML perfectly well. The challenge is **coordinating state changes across a complex, asynchronous, event-driven UI** in a way that remains understandable and maintainable as the application grows.

---

## Types of State — The Four Categories

The most useful mental framework for thinking about state is to categorize it by *who owns it* and *how it changes*. There are four primary categories, and each one deserves different treatment.

### 1. UI State

UI state describes how the interface looks and behaves, independent of the underlying data. A modal being open or closed, which tab is currently selected, whether a sidebar is expanded or collapsed, whether a row in a table is in edit mode — these are all UI state.

UI state tends to be **ephemeral** (it resets when the component is destroyed), **local** (only relevant to one part of the UI), and **synchronous** (it changes instantly in response to user actions, with no async operations involved). Because of these characteristics, UI state almost never belongs in a global state store. Keeping it in the component that owns it is almost always the right decision.

```typescript
// Perfect example of UI state — belongs in the component, nowhere else
@Component({ /* ... */ })
export class UserCardComponent {
  // isExpanded is pure UI state — no API call, no global relevance
  isExpanded = signal(false);
  activeTab = signal<'overview' | 'activity' | 'settings'>('overview');

  toggle() {
    this.isExpanded.update(v => !v);
  }
}
```

### 2. Server State (Remote State)

Server state is data that *lives on the server* and is temporarily *copied* into the browser. The definitive version always lives in your database. Book lists, user profiles, order histories, product inventories — all of this is server state.

Server state has characteristics that make it fundamentally different from other state categories. It is **asynchronous** (you must wait for a network request to get it), **stale** (the moment you fetch it, it may already be outdated on the server), **shared** (multiple users may be reading and modifying the same data), and **persistently owned elsewhere** (if your Angular app crashes, the data still exists on the server).

These characteristics mean server state needs special treatment: caching, background refetching, synchronization, loading and error states. Tools like TanStack Query and Angular's `httpResource()` are built specifically to handle server state. Treating server state the same way you treat local UI state is the source of many subtle bugs.

### 3. Form State

Form state is the current value and validation status of your forms. It includes what the user has typed, which fields have been touched, which fields are valid or invalid, and whether the form as a whole is ready to submit.

Angular's Reactive Forms system manages form state for you. `FormControl`, `FormGroup`, and `FormArray` are purpose-built containers for form state, with their own reactivity model (via `valueChanges` observables and, experimentally, via signal forms in Angular 20+). Form state should almost never leave the form component that owns it — it's ephemeral by nature, meant to be validated and then submitted or discarded.

### 4. URL State (Navigation State)

URL state is the data encoded in the current URL: the path, route parameters, query parameters, and hash fragment. It is unique among state categories because it is **visible and shareable** — a user can copy the URL, share it with a colleague, and that colleague's browser should render the same view.

Because of this, URL state is the most powerful form of UI state for anything that represents a *navigational position* in your application. The currently selected product ID, the current search term, the active filters, the current page number — all of these should live in the URL if they represent something a user would want to bookmark or share. Using Angular Router's query parameters and route parameters for this purpose is a fundamental best practice.

---

## Local vs Global State — The Most Misunderstood Distinction

This is the decision that causes the most architectural errors in Angular applications. Developers discover NgRx or a signal-based store and immediately move everything into it. The result is a bloated global store containing the `isMenuOpen` flag, the `activeFormStep` variable, and the `selectedTabIndex` — none of which any other part of the app cares about.

**Local state** is state that is only relevant to a single component or a small subtree of related components. It should live as close as possible to the component that uses it.

**Global state** is state that is genuinely shared across multiple, unrelated parts of the application. The currently logged-in user's profile, the application's notification queue, the items in the shopping cart — these are legitimately global because many unrelated components need to read or react to them.

The test for deciding which category a piece of state belongs to is simple: if you destroyed every component that currently reads or modifies this state, would any *other* component in the application break or need to know about the change? If yes, it's global. If no, it's local.

Here is an important nuance: a piece of state can be global in scope but still scoped narrowly in implementation. An NgRx feature store registered lazily for a specific route is "global" in the sense that multiple components within that route can access it, but it's not polluting the application-wide root store. This intermediate level — **feature-level state** — is where most of your application's shared state should live.

---

## When NOT to Use Global State

The gravitational pull toward global state management is strong, especially after a team has invested in setting up NgRx. Every new piece of data feels like it should go into the store. Resisting this pull is a sign of architectural maturity.

You should *not* use global state for data that is only ever read and written by one component. A `UserEditComponent` manages an in-progress edit of a user's profile — that draft data belongs in the component, not the store. When the component is destroyed (user navigates away), the draft should disappear.

You should not use global state for ephemeral UI interactions. Whether a dropdown is open, whether a row is in edit mode, which tooltip is visible — these should be managed locally, not stored in a shared state container.

You should not use global state as a cache for every API response. If you fetch a list of countries to populate a dropdown in a settings form, that data does not need to be in the NgRx store. If you fetch it once and the form is the only consumer, a component-local signal or even a simple variable is appropriate. Server state caching tools like TanStack Query handle this far more elegantly than putting raw API data into NgRx.

You should not use global state for data that is already captured in the URL. If the current page number is in the query params (`?page=3`), there is no need to also store `currentPage` in a signal or NgRx store. The URL is the state — read it from there.

---

## The State Colocation Principle

State colocation means: **keep your state as close as possible to the code that uses it**. This is the most important principle in state management architecture.

The principle comes from React's co-developer Kent C. Dodds, but it applies universally to any component-based framework including Angular. It can be expressed as a decision hierarchy. If only one component needs a piece of state, keep it in that component. If two sibling components need it, lift it up to their common parent. If a subtree of related components needs it, put it in a shared service scoped to that subtree. Only if truly unrelated, distant parts of the application need it should you put it in a global store.

This hierarchy prevents two failure modes: state scattered in a global store that nobody can trace, and state duplicated in multiple components that gets out of sync. When state lives as close as possible to where it is used, you always know exactly where to look for it and exactly who can change it.

In Angular's signal era, this principle has a new implementation form: a signal-based service provided at the component level via the component's `providers` array. This makes the service — and its state — scoped to exactly that component tree, automatically destroyed when the component is destroyed, and invisible to the rest of the application.

```typescript
// Scoped state — provided at the component level, not 'root'
@Injectable()
export class ProductSearchStore {
  private _query = signal('');
  private _filters = signal<Filters>({});
  private _results = signal<Product[]>([]);

  readonly query = this._query.asReadonly();
  readonly results = this._results.asReadonly();
  readonly hasResults = computed(() => this._results().length > 0);

  setQuery(q: string) { this._query.set(q); }
  setFilters(f: Filters) { this._filters.set(f); }
  setResults(r: Product[]) { this._results.set(r); }
}

@Component({
  // This store is created fresh for this component and destroyed with it
  providers: [ProductSearchStore],
  template: `
    <app-search-bar (search)="store.setQuery($event)" />
    <app-filter-panel (change)="store.setFilters($event)" />
    <app-results-list [products]="store.results()" />
  `
})
export class ProductSearchPage {
  store = inject(ProductSearchStore);
}
```

The child components (`SearchBar`, `FilterPanel`, `ResultsList`) can also inject `ProductSearchStore` and they'll all get the same instance — but that instance is invisible to any component outside this tree. That is perfect colocation.

---

## Unidirectional Data Flow — The Architecture That Makes Debugging Possible

Unidirectional data flow is the architectural constraint that every major modern frontend framework enforces in some form. It means that data travels in only one direction through your application: from a single source of truth (the state), down through components, into the UI. User interactions trigger events that flow back up to update the state, and the cycle repeats.

The opposite of unidirectional flow is bidirectional binding (the approach AngularJS 1.x used), where data can be modified by any component in any direction at any time. Bidirectional binding is easy to get started with but catastrophically hard to debug at scale, because when something goes wrong, you have no idea which of the dozens of components that touch a piece of data caused the corruption.

With unidirectional flow, there is always a clear answer to "where did this state come from?" It came from the state container. And there is always a clear answer to "who changed this?" — whoever dispatched the action or called the update method. The entire state history becomes traceable.

Angular Signal's model enforces unidirectional flow elegantly. A writable signal has an `asReadonly()` method that exposes a read-only view to consumers. Components can read the signal but cannot write to it directly — they must call a method on the store service that owns the signal. This is Angular's equivalent of the Redux principle that state can only be updated through defined channels.

```typescript
// Unidirectional flow implemented with signals
@Injectable({ providedIn: 'root' })
export class NotificationStore {
  // Private — only this service can write
  private _notifications = signal<Notification[]>([]);

  // Public — anyone can read, but cannot write
  readonly notifications = this._notifications.asReadonly();
  readonly unreadCount = computed(() =>
    this._notifications().filter(n => !n.read).length
  );

  // Defined mutation channels — this is how state changes
  add(notification: Omit<Notification, 'id' | 'read'>) {
    const newNotification = { ...notification, id: crypto.randomUUID(), read: false };
    this._notifications.update(current => [newNotification, ...current]);
  }

  markAsRead(id: string) {
    this._notifications.update(current =>
      current.map(n => n.id === id ? { ...n, read: true } : n)
    );
  }

  dismiss(id: string) {
    this._notifications.update(current => current.filter(n => n.id !== id));
  }
}
```

Every component that reads `notifications` and `unreadCount` receives the same data from the same source. Every component that needs to change notifications must go through `add()`, `markAsRead()`, or `dismiss()`. There is one direction of data flow: store → component → template. There is one channel for change: component → store method. Debugging becomes straightforward.

---

## Important Points and Best Practices

The most important decision in state management is not which library to use — it is *where* to put each piece of state. Get the colocation decision right first; the library choice is secondary.

Treat state categories differently. UI state belongs in components. Server state belongs in a caching layer (TanStack Query, `httpResource()`). Form state belongs in Angular Forms. URL state belongs in the Router. Only genuinely shared, cross-cutting application state belongs in a global store.

Never store derived data. If `totalPrice` can be computed from `cartItems` using a `computed()` signal or an NgRx selector, do not store `totalPrice` separately. Storing derived data introduces synchronization bugs — the source updates but the derived value doesn't, and now your UI shows contradictory information.

State should always be the single source of truth. If you find yourself maintaining the same data in two places (for example, a user list in NgRx and also in a component's local variable), you have duplicated state and you *will* encounter consistency bugs. Pick one source and derive everything else from it.

Immutability is not optional in unidirectional architectures. When you update state, always return a *new* object or array rather than mutating the existing one. Angular's change detection — especially with `OnPush` and signals — relies on reference equality to know when something has changed. If you mutate a state object in place, the reference stays the same, the change detection cycle sees no change, and your UI does not update. This is one of the most common sources of hard-to-debug bugs in Angular applications.

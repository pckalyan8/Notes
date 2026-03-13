# Phase 14.8 — Advanced State Patterns
## Event Sourcing, CQRS, Undo/Redo, Time-Travel Debugging, localStorage Sync & Cross-Tab Communication

---

## Why Advanced Patterns Matter

The patterns in this section represent the frontier of frontend state management thinking. They are not techniques you will use in every project — in fact, reaching for them when simpler approaches suffice is a form of over-engineering. But understanding them deeply serves two purposes. First, some applications genuinely need them: collaborative editors, financial trading UIs, complex multi-step workflows, and offline-first applications all benefit significantly from these patterns. Second, studying them sharpens your intuition for all state management, because they reveal the fundamental properties of state — identity, time, causality, and ownership — in ways that simpler patterns obscure.

---

## 14.8.1 — Event Sourcing in the Frontend

### The Core Idea

Traditional state management stores the *current state* — the latest snapshot of what is true right now. Event sourcing stores the *history of events* that caused the current state — the sequence of things that happened. The current state is always *derived* from the event log by replaying it from the beginning.

Imagine a bank account. Traditional approach: store the current balance (`$1,247.50`). Event sourcing approach: store the history of transactions (`[deposit $500, withdrawal $100, deposit $847.50, ...]`). The balance is derived by summing the transactions. If you want to know the balance at any point in history, you replay only the events up to that moment.

This sounds expensive, but the benefits are remarkable. You have a complete audit trail — you can answer "how did we end up in this state?" with perfect accuracy. You can replay the history to reconstruct state at any point in time. You can introduce new derived views by replaying history through a different projection function. And because events are immutable (something that happened cannot be un-happened), the history is an append-only log that is very safe to work with.

### Implementing Event Sourcing in Angular

A pure event-sourced store in Angular keeps an append-only array of events and derives the current state by reducing over them:

```typescript
import { Injectable, signal, computed } from '@angular/core';

// Events are discriminated union types — each event has a unique 'type' and its payload
type DrawingEvent =
  | { type: 'CANVAS_CLEARED'; timestamp: number }
  | { type: 'SHAPE_ADDED'; timestamp: number; shape: Shape }
  | { type: 'SHAPE_MOVED'; timestamp: number; shapeId: string; dx: number; dy: number }
  | { type: 'SHAPE_RESIZED'; timestamp: number; shapeId: string; width: number; height: number }
  | { type: 'SHAPE_DELETED'; timestamp: number; shapeId: string }
  | { type: 'COLOR_CHANGED'; timestamp: number; shapeId: string; color: string };

interface Shape {
  id: string;
  type: 'rect' | 'circle' | 'line';
  x: number;
  y: number;
  width: number;
  height: number;
  color: string;
}

interface DrawingState {
  shapes: Map<string, Shape>;
}

@Injectable({ providedIn: 'root' })
export class DrawingEventStore {
  // The append-only event log — this is the source of truth
  // The current state is DERIVED from this log, not stored separately
  private _events = signal<DrawingEvent[]>([]);

  // Public read-only access to the event log
  readonly events = this._events.asReadonly();

  // The current state is a computed projection of the event log
  // It only recomputes when the events array changes (i.e., when a new event is appended)
  readonly state = computed<DrawingState>(() => {
    return this._events().reduce<DrawingState>(
      (state, event) => this.applyEvent(state, event),
      { shapes: new Map() }
    );
  });

  // Derived views from the current state
  readonly shapes = computed(() => Array.from(this.state().shapes.values()));
  readonly shapeCount = computed(() => this.state().shapes.size);

  // Append an event — the ONLY way to change state
  dispatch(event: Omit<DrawingEvent, 'timestamp'>) {
    const stampedEvent = { ...event, timestamp: Date.now() } as DrawingEvent;
    this._events.update(events => [...events, stampedEvent]);
  }

  // The reducer — a pure function that computes the state after one event
  private applyEvent(state: DrawingState, event: DrawingEvent): DrawingState {
    const shapes = new Map(state.shapes); // Immutable copy

    switch (event.type) {
      case 'CANVAS_CLEARED':
        return { shapes: new Map() };

      case 'SHAPE_ADDED':
        shapes.set(event.shape.id, event.shape);
        return { shapes };

      case 'SHAPE_MOVED': {
        const shape = shapes.get(event.shapeId);
        if (!shape) return state;
        shapes.set(event.shapeId, {
          ...shape,
          x: shape.x + event.dx,
          y: shape.y + event.dy,
        });
        return { shapes };
      }

      case 'SHAPE_DELETED':
        shapes.delete(event.shapeId);
        return { shapes };

      case 'COLOR_CHANGED': {
        const shape = shapes.get(event.shapeId);
        if (!shape) return state;
        shapes.set(event.shapeId, { ...shape, color: event.color });
        return { shapes };
      }

      default:
        return state;
    }
  }

  // Project the event log at a specific point in time
  // This is one of the most powerful capabilities of event sourcing
  getStateAtTime(timestamp: number): DrawingState {
    const eventsUpToTime = this._events().filter(e => e.timestamp <= timestamp);
    return eventsUpToTime.reduce<DrawingState>(
      (state, event) => this.applyEvent(state, event),
      { shapes: new Map() }
    );
  }

  // Replay from a specific event index onward (for time-travel)
  replayFrom(eventIndex: number) {
    // Trim the history to only include events up to the chosen point
    this._events.update(events => events.slice(0, eventIndex + 1));
  }
}
```

```typescript
// Using the event-sourced store in a component
@Component({ /* ... */ })
export class DrawingCanvasComponent {
  protected store = inject(DrawingEventStore);

  addRect(x: number, y: number) {
    // Dispatch an event — not a mutation command
    this.store.dispatch({
      type: 'SHAPE_ADDED',
      shape: {
        id: crypto.randomUUID(),
        type: 'rect',
        x, y,
        width: 100, height: 60,
        color: '#6200ea',
      }
    });
  }

  moveShape(shapeId: string, dx: number, dy: number) {
    this.store.dispatch({ type: 'SHAPE_MOVED', shapeId, dx, dy });
  }
}
```

The power becomes obvious when you realize that the shape history is always available (`store.events()`), you can reconstruct the canvas at any past moment (`getStateAtTime()`), and you can implement undo simply by truncating the event log.

---

## 14.8.2 — CQRS (Command Query Responsibility Segregation) on the Frontend

### Understanding CQRS

CQRS is the principle that **reading data (Queries) and writing data (Commands) should be handled by completely separate code paths**. The key insight is that read operations and write operations have fundamentally different requirements. Reads are frequent, need to be fast, and often need many different shapes of the same data. Writes are less frequent, need to be reliable, and must maintain consistency.

In a traditional Angular architecture, the same service might be responsible for both fetching a product (`getProduct(id)`) and updating it (`updateProduct(id, changes)`). CQRS separates these: a Query service handles reads (optimized for the view's specific shape), and a Command service handles writes (focused on validation and consistency).

On the frontend, a useful way to apply CQRS is to have separate APIs for "give me data for this view" and "apply this change":

```typescript
// ---- QUERY SIDE — reading data optimized for specific views ----
// Query services focus on returning exactly the data shape the UI needs.
// They can aggregate data from multiple sources, apply transformations,
// and cache aggressively.

@Injectable({ providedIn: 'root' })
export class OrderDashboardQueryService {
  private http = inject(HttpClient);
  private store = inject(Store);

  // This query returns a "view model" tailored to the dashboard view
  // It combines data from multiple sources — the API and the NgRx store
  readonly dashboardViewModel = computed(() => {
    const user = this.store.selectSignal(selectCurrentUser)();
    const ordersResource = httpResource<Order[]>(() => `/api/orders?userId=${user?.id}`);

    return {
      orders: ordersResource.value() ?? [],
      loading: ordersResource.isLoading(),
      // Derived data computed from the raw API response
      pendingCount: (ordersResource.value() ?? []).filter(o => o.status === 'pending').length,
      totalRevenue: (ordersResource.value() ?? []).reduce((sum, o) => sum + o.total, 0),
      // Personalization from the NgRx store
      userName: user?.name ?? 'Guest',
    };
  });

  // Different query for a different view — same underlying data, different shape
  getOrderHistory(page: number, size: number): Observable<PagedResult<Order>> {
    return this.http.get<PagedResult<Order>>(`/api/orders?page=${page}&size=${size}`);
  }
}
```

```typescript
// ---- COMMAND SIDE — writing data with validation and consistency ----
// Command services focus on expressing clear intent and maintaining invariants.
// They validate input, communicate to the API, and update state on success.

@Injectable({ providedIn: 'root' })
export class OrderCommandService {
  private http = inject(HttpClient);
  private store = inject(Store);
  private router = inject(Router);

  // Each method is a "command" — it expresses clear intent
  placeOrder(cart: CartItem[]): Observable<Order> {
    if (cart.length === 0) {
      return throwError(() => new Error('Cannot place an empty order'));
    }
    return this.http.post<Order>('/api/orders', { items: cart }).pipe(
      tap(order => {
        // Side effect: update NgRx state and navigate on success
        this.store.dispatch(OrderApiActions.placeOrderSuccess({ order }));
        this.router.navigate(['/orders', order.id, 'confirmation']);
      })
    );
  }

  cancelOrder(orderId: string, reason: string): Observable<void> {
    return this.http.patch<void>(`/api/orders/${orderId}/cancel`, { reason }).pipe(
      tap(() => this.store.dispatch(OrderApiActions.cancelOrderSuccess({ orderId })))
    );
  }

  markAsShipped(orderId: string, trackingNumber: string): Observable<Order> {
    return this.http.patch<Order>(`/api/orders/${orderId}/ship`, { trackingNumber });
  }
}
```

CQRS on the frontend is often implemented implicitly — the fact that TanStack Query handles reads and HTTP client calls handle writes is essentially CQRS. Making it explicit by naming your services this way improves readability and makes it clear which code path is being invoked at any point.

---

## 14.8.3 — Undo/Redo Implementation

### The Two Approaches

There are two classic ways to implement undo/redo: the **state snapshot** approach and the **command history** approach.

The state snapshot approach stores a stack of past complete state snapshots. Undo pops the current state off the stack and restores the previous one. Redo pushes a state back onto the stack. This is simple to implement but can be memory-intensive if states are large.

The command history approach stores a log of reversible operations (commands) instead of snapshots. Each command knows how to `execute()` itself (apply the change) and `undo()` itself (reverse the change). Undo calls the current command's `undo()` method. This is more memory efficient but requires every command to implement a reverse operation, which can be complex for some operations.

### The State Snapshot Approach with Signals

This approach is natural for small-to-medium state objects:

```typescript
@Injectable()
export class UndoableTextEditorStore {
  // The history stack contains previous state snapshots
  private _past = signal<string[]>([]);
  // The current state
  private _content = signal<string>('');
  // The future stack contains states that were undone (for redo)
  private _future = signal<string[]>([]);

  readonly content = this._content.asReadonly();
  readonly canUndo = computed(() => this._past().length > 0);
  readonly canRedo = computed(() => this._future().length > 0);

  setContent(newContent: string) {
    // Before changing, push the current state onto the past stack
    this._past.update(past => [...past, this._content()]);
    // Clear the future stack — once you make a new change,
    // the "redo" history becomes invalid (same behavior as most text editors)
    this._future.set([]);
    // Apply the new content
    this._content.set(newContent);
  }

  undo() {
    if (this._past().length === 0) return;

    const past = this._past();
    const previous = past[past.length - 1];

    // Push current state to the future (for redo)
    this._future.update(future => [this._content(), ...future]);
    // Restore the previous state
    this._content.set(previous);
    // Remove the last entry from the past stack
    this._past.update(p => p.slice(0, -1));
  }

  redo() {
    if (this._future().length === 0) return;

    const [next, ...remainingFuture] = this._future();

    // Push current state to past
    this._past.update(past => [...past, this._content()]);
    // Restore the next future state
    this._content.set(next);
    this._future.set(remainingFuture);
  }
}
```

### Undo/Redo in NgRx

When using NgRx, implementing undo/redo involves wrapping the standard store with a history mechanism. The `ngrx-history` pattern stores snapshots of the NgRx state and a pointer to the current position:

```typescript
// A reducer "meta-reducer" that wraps any reducer with undo/redo capability
// A meta-reducer is a higher-order function: it takes a reducer and returns a new reducer
export function undoRedoMetaReducer(reducer: ActionReducer<any>): ActionReducer<any> {
  interface UndoRedoState {
    past: any[];
    present: any;
    future: any[];
  }

  let initialState: UndoRedoState;

  return (state, action) => {
    if (!initialState) {
      // Initialize with the reducer's initial state
      const presentState = reducer(undefined, action);
      initialState = { past: [], present: presentState, future: [] };
      return initialState;
    }

    const { past, present, future } = state as UndoRedoState;

    if (action.type === 'UNDO') {
      if (past.length === 0) return state;
      const previous = past[past.length - 1];
      return {
        past: past.slice(0, -1),
        present: previous,
        future: [present, ...future],
      };
    }

    if (action.type === 'REDO') {
      if (future.length === 0) return state;
      const [next, ...remainingFuture] = future;
      return {
        past: [...past, present],
        present: next,
        future: remainingFuture,
      };
    }

    // For all other actions, compute the new present and add current to past
    const newPresent = reducer(present, action);
    if (newPresent === present) return state; // State unchanged — don't pollute history
    return {
      past: [...past, present].slice(-50), // Limit history to 50 entries
      present: newPresent,
      future: [],
    };
  };
}
```

---

## 14.8.4 — Time-Travel Debugging with Redux DevTools

If you are using NgRx (classic or Signals Store with DevTools integration), the Redux DevTools browser extension gives you time-travel debugging for free. You can jump to any point in the action history and see exactly what the state looked like at that moment, without writing any additional code.

What is happening under the hood is exactly the event sourcing pattern: the DevTools records every dispatched action, and "time-traveling" means replaying those actions from the beginning up to the chosen point. This is why Redux's architecture — with its immutable state and pure reducers — is specifically designed to enable time-travel debugging. A mutable state container cannot be time-traveled because you lose previous states permanently when you mutate.

```typescript
// Enable DevTools when bootstrapping your app
import { provideStoreDevtools } from '@ngrx/store-devtools';

bootstrapApplication(AppComponent, {
  providers: [
    provideStore({ products: productsReducer }),
    provideStoreDevtools({
      maxAge: 50,              // Keep last 50 actions in history
      logOnly: !isDevMode(),   // Only log in development — disable time-travel in production
      // actionsBlocklist: ['[Router] Navigation']  // Optionally filter noisy actions
    }),
  ]
});
```

With DevTools enabled, you get these capabilities: you can pause dispatching (freeze the application in its current state), you can jump to any past state, you can "commit" (reset the baseline to the current state), and you can "revert" (go back to the last committed state). For debugging complex state sequences — for example, "how did I end up with an empty cart after completing checkout?" — this is invaluable.

---

## 14.8.5 — State Persistence with localStorage

State persistence means saving a portion of your application state to `localStorage` (or `sessionStorage`) so that it survives a page refresh. This is useful for preferences (dark mode, selected language), partially completed workflows (a multi-step form the user started but didn't finish), and cache preloading (showing cached data immediately on page load while fresh data arrives in the background).

### Persistence with Signal Services

```typescript
@Injectable({ providedIn: 'root' })
export class ThemePreferencesStore {
  // Key constants prevent typos and make it easy to audit what's stored
  private static readonly STORAGE_KEY = 'user-theme-preferences';

  // Initialize state from localStorage if available, otherwise use defaults
  private _preferences = signal<ThemePreferences>(this.loadFromStorage());

  readonly preferences = this._preferences.asReadonly();
  readonly isDark = computed(() => this._preferences().mode === 'dark');

  constructor() {
    // Automatically persist to localStorage whenever preferences change
    // effect() runs once initially and again whenever this.preferences() changes
    effect(() => {
      const prefs = this._preferences();
      try {
        localStorage.setItem(ThemePreferencesStore.STORAGE_KEY, JSON.stringify(prefs));
      } catch {
        // localStorage can throw in some browsers (incognito mode, storage full)
        // Fail gracefully — the app still works, preferences just won't persist
        console.warn('Could not save preferences to localStorage');
      }
    });
  }

  setMode(mode: 'light' | 'dark' | 'system') {
    this._preferences.update(p => ({ ...p, mode }));
  }

  setLanguage(language: string) {
    this._preferences.update(p => ({ ...p, language }));
  }

  private loadFromStorage(): ThemePreferences {
    try {
      const stored = localStorage.getItem(ThemePreferencesStore.STORAGE_KEY);
      if (stored) {
        const parsed = JSON.parse(stored);
        // Validate the shape before trusting it — localStorage data can be corrupted
        // or from an older version of your app with a different schema
        if (this.isValidPreferences(parsed)) return parsed;
      }
    } catch {
      // JSON.parse can throw if the stored data is malformed
    }
    // Return default preferences if nothing valid was found
    return { mode: 'system', language: 'en', fontSize: 'medium' };
  }

  private isValidPreferences(value: unknown): value is ThemePreferences {
    if (typeof value !== 'object' || value === null) return false;
    const v = value as Record<string, unknown>;
    return (
      ['light', 'dark', 'system'].includes(v['mode'] as string) &&
      typeof v['language'] === 'string'
    );
  }
}
```

### Persistence with NgRx: ngrx-store-localstorage

For NgRx applications, the `ngrx-store-localstorage` library provides a meta-reducer that automatically persists specified state slices to localStorage and rehydrates them on startup:

```bash
npm install ngrx-store-localstorage
```

```typescript
import { localStorageSync } from 'ngrx-store-localstorage';

// The meta-reducer wraps your normal reducers and adds persistence
export function localStorageSyncReducer(reducer: ActionReducer<AppState>): ActionReducer<AppState> {
  return localStorageSync({
    // List the feature state slices to persist
    keys: [
      'cart',                                // Persist the entire 'cart' slice
      { preferences: ['theme', 'language'] }, // Persist only specific fields of 'preferences'
    ],
    rehydrate: true,  // On startup, load persisted state back into the store
    storage: localStorage,
  })(reducer);
}

bootstrapApplication(AppComponent, {
  providers: [
    provideStore(
      { cart: cartReducer, preferences: preferencesReducer },
      { metaReducers: [localStorageSyncReducer] }  // Apply the persistence meta-reducer
    ),
  ]
});
```

An important consideration with any persistence mechanism: your state schema will change over time as your application evolves. A user who hasn't visited your site in six months may have an old schema version in their localStorage. You need a **migration strategy** — either clearing all persisted state when the schema changes (simplest but disruptive), versioning your storage keys, or writing explicit migration functions that transform old schemas to new ones.

---

## 14.8.6 — Cross-Tab State Synchronization with Broadcast Channel

When a user has your application open in multiple browser tabs, state changes in one tab need to propagate to others. If the user adds an item to their cart in Tab 1, Tab 2 should reflect that change. The `BroadcastChannel` API provides exactly this capability — a pub/sub channel for messages between browser tabs (and windows and iframes) with the same origin.

```typescript
import { Injectable, signal, OnDestroy, effect, NgZone, inject } from '@angular/core';

interface CartSyncMessage {
  type: 'CART_UPDATED';
  cart: CartItem[];
  timestamp: number;
  tabId: string;  // Include the tab ID to avoid echo (processing your own messages)
}

@Injectable({ providedIn: 'root' })
export class CrossTabCartSync implements OnDestroy {
  private ngZone = inject(NgZone);

  // Unique identifier for this specific tab — prevents processing our own messages
  private readonly tabId = crypto.randomUUID();

  // BroadcastChannel is scoped to the channel name — all tabs using 'cart-sync' can communicate
  private channel = new BroadcastChannel('cart-sync');

  // The local cart state — this tab's source of truth
  private _cart = signal<CartItem[]>([]);
  readonly cart = this._cart.asReadonly();

  constructor() {
    // Listen for cart updates from other tabs
    this.channel.onmessage = (event: MessageEvent<CartSyncMessage>) => {
      const message = event.data;

      // Ignore messages that originated from this same tab
      if (message.tabId === this.tabId) return;

      if (message.type === 'CART_UPDATED') {
        // BroadcastChannel callbacks run outside Angular's zone
        // Use ngZone.run() to ensure the signal update triggers change detection
        this.ngZone.run(() => {
          this._cart.set(message.cart);
        });
      }
    };

    // Whenever this tab's cart changes, broadcast the update to other tabs
    effect(() => {
      const cart = this._cart(); // reactive dependency
      this.broadcastCartUpdate(cart);
    });
  }

  addItem(item: CartItem) {
    this._cart.update(current => {
      const exists = current.find(i => i.id === item.id);
      return exists
        ? current.map(i => i.id === item.id ? { ...i, quantity: i.quantity + 1 } : i)
        : [...current, item];
    });
    // The effect() above will automatically broadcast the change
  }

  removeItem(id: string) {
    this._cart.update(current => current.filter(i => i.id !== id));
  }

  private broadcastCartUpdate(cart: CartItem[]) {
    const message: CartSyncMessage = {
      type: 'CART_UPDATED',
      cart,
      timestamp: Date.now(),
      tabId: this.tabId,
    };
    try {
      this.channel.postMessage(message);
    } catch (err) {
      // postMessage can fail if the data is not serializable
      console.warn('Could not broadcast cart update:', err);
    }
  }

  ngOnDestroy() {
    // Always close the channel when the service is destroyed — it releases the listener
    this.channel.close();
  }
}
```

The `BroadcastChannel` API supports any serializable data — plain objects, arrays, numbers, strings. It does not support functions, class instances, or DOM nodes. The data is serialized using the Structured Clone algorithm (similar to `JSON.stringify` but supporting more types like `Date`, `Map`, and `Set`).

A practical consideration for cross-tab synchronization: it is possible for two tabs to make conflicting changes simultaneously (the user adds an item in Tab 1 at the same time as removing an item in Tab 2). The simple "last write wins" approach of just setting the cart to whatever the latest broadcast message says can produce unexpected behavior. For collaborative applications where this matters, you need a more sophisticated conflict resolution strategy — often the same strategies used in distributed systems (operational transforms, CRDTs, or server-side conflict resolution via websockets).

---

## Important Points and Best Practices

Event sourcing is powerful but comes with real costs. The `reduce()` over the entire event log runs on every state change — for thousands of events this can become slow. The standard solution is **snapshotting**: periodically saving the current state as a "checkpoint", then only replaying events *after* the checkpoint. Start simple and add snapshotting only when performance becomes a measurable issue.

Undo/redo must decide what is "undoable". In most applications, not every state change should be undoable — only intentional, user-initiated modifications. Automatic state changes (data fetched from the server, timers firing) should not pollute the undo history. Design your undo/redo system to only record changes that make semantic sense to reverse.

CQRS is a pattern, not a library, and it exists on a spectrum. You do not have to fully separate commands and queries into different database schemas (as some backend CQRS implementations do). Simply naming your services and functions to reflect whether they read or write, and keeping those concerns from mixing in the same function, is already applying the principle beneficially.

For localStorage persistence, always validate what you read back. localStorage is mutable by anything — browser extensions, other scripts, even the developer manually — and its contents can be from an older version of your app with a different data shape. A Zod schema validation on the deserialized data before using it protects against these edge cases and prevents cryptic runtime errors.

Cross-tab synchronization via BroadcastChannel is best reserved for high-value, low-frequency state changes — things like "user signed out in another tab" (which should sign out all tabs), cart updates, and settings changes. Do not try to synchronize rapidly-changing state (like cursor positions or real-time collaborative editing) via BroadcastChannel — for that, you need a server-side WebSocket-based synchronization system.

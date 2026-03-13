# Phase 15.10 — RxJS Advanced Patterns
## State Machine with scan, WebSocket with RxJS, Polling, Search-as-you-type, Infinite Scroll, Undo/Redo, Optimistic Updates, Complex Async Orchestration

---

## Why Advanced Patterns Matter

RxJS operators are the vocabulary; patterns are the sentences. Individual operators answer the question "what does this function do?" Advanced patterns answer the real engineering question: "how do I compose multiple operators to solve this class of problem elegantly?" Each pattern in this section represents a canonical solution to a frequently-occurring async coordination challenge in modern Angular applications. Learning them is learning a set of repeatable solutions you can apply with confidence.

---

## 15.10.1 — State Machine with `scan()`

A state machine is a computation that moves through a finite set of named states, transitioning between them in response to events. State machines are excellent for complex UI flows because they make impossible states impossible — if "loading" and "error" are distinct states, you can never accidentally be in both at once (a common bug with boolean flags).

`scan()` is the perfect operator for implementing state machines reactively because it maintains running state and emits the new state after every event.

```typescript
import { Subject, merge, of } from 'rxjs';
import { scan, switchMap, map, startWith, catchError } from 'rxjs/operators';

// Define all possible states as a discriminated union — no impossible combinations
type FetchState<T> =
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'success'; data: T; lastFetched: Date }
  | { status: 'error'; error: Error; retryCount: number };

// Define all events that can cause transitions
type FetchEvent<T> =
  | { type: 'FETCH' }
  | { type: 'SUCCESS'; data: T }
  | { type: 'FAILURE'; error: Error }
  | { type: 'RESET' };

// The pure transition function — given the current state and an event, return the next state
function fetchReducer<T>(state: FetchState<T>, event: FetchEvent<T>): FetchState<T> {
  switch (event.type) {
    case 'FETCH':
      return { status: 'loading' };

    case 'SUCCESS':
      return { status: 'success', data: event.data, lastFetched: new Date() };

    case 'FAILURE':
      const retryCount = state.status === 'error' ? state.retryCount + 1 : 1;
      return { status: 'error', error: event.error, retryCount };

    case 'RESET':
      return { status: 'idle' };

    default:
      return state; // Unknown event — stay in current state
  }
}

@Component({ selector: 'app-product-list', standalone: true, /* ... */ })
export class ProductListComponent {
  // The event bus — we push events into this Subject
  private events$ = new Subject<FetchEvent<Product[]>>();

  // The state stream — scan() drives the state machine
  state$ = this.events$.pipe(
    scan(fetchReducer, { status: 'idle' } as FetchState<Product[]>),
    startWith({ status: 'idle' } as FetchState<Product[]>)
  );

  constructor(private productService: ProductService) {
    // Wire up the fetch trigger: when a FETCH event arrives, make the HTTP call
    // and dispatch SUCCESS or FAILURE based on the result
    this.events$.pipe(
      filter(e => e.type === 'FETCH'),
      switchMap(() =>
        this.productService.getAll().pipe(
          map(data => ({ type: 'SUCCESS' as const, data })),
          catchError(error => of({ type: 'FAILURE' as const, error }))
        )
      ),
      takeUntilDestroyed()
    ).subscribe(event => this.events$.next(event));
  }

  fetch() { this.events$.next({ type: 'FETCH' }); }
  reset() { this.events$.next({ type: 'RESET' }); }
}
```

```html
@switch ((state$ | async)?.status) {
  @case ('idle') { <button (click)="fetch()">Load Products</button> }
  @case ('loading') { <app-spinner /> }
  @case ('success') {
    @for (p of (state$ | async)?.data; track p.id) {
      <app-product-card [product]="p" />
    }
  }
  @case ('error') {
    <p>Error: {{ (state$ | async)?.error?.message }}</p>
    <button (click)="fetch()">Retry</button>
  }
}
```

The advantage of this pattern over `boolean` flags (`isLoading`, `hasError`, `hasData`) is that the state machine only allows legal transitions. You cannot be `loading: true` and `error: true` simultaneously — they are different objects in the discriminated union, not different properties on the same object.

---

## 15.10.2 — WebSocket with RxJS

WebSockets are a natural fit for RxJS because they represent a long-lived bidirectional stream of messages. RxJS provides the `webSocket()` creation function from `rxjs/webSocket` that wraps the browser's native WebSocket API in an Observable/Subject hybrid, giving you both sending and receiving capabilities.

```typescript
import { webSocket, WebSocketSubject } from 'rxjs/webSocket';
import { retryWhen, delay, tap, share, filter, map } from 'rxjs/operators';
import { timer, Subject } from 'rxjs';

// Define your message types for type safety
type ServerMessage =
  | { type: 'CHAT_MESSAGE'; payload: { userId: string; text: string; timestamp: number } }
  | { type: 'USER_JOINED'; payload: { userId: string; username: string } }
  | { type: 'USER_LEFT'; payload: { userId: string } }
  | { type: 'PING' };

type ClientMessage =
  | { type: 'CHAT_MESSAGE'; payload: { text: string } }
  | { type: 'PONG' };

@Injectable({ providedIn: 'root' })
export class ChatWebSocketService {
  private socket$!: WebSocketSubject<ServerMessage | ClientMessage>;
  private connected$ = new Subject<boolean>();

  // Public streams for components to subscribe to
  readonly chatMessages$: Observable<ServerMessage & { type: 'CHAT_MESSAGE' }>;
  readonly userEvents$: Observable<ServerMessage & { type: 'USER_JOINED' | 'USER_LEFT' }>;
  readonly isConnected$ = this.connected$.asObservable().pipe(
    startWith(false),
    distinctUntilChanged()
  );

  constructor() {
    this.socket$ = webSocket<ServerMessage | ClientMessage>({
      url: 'wss://api.example.com/chat',
      openObserver: {
        next: () => {
          console.log('[WebSocket] Connected');
          this.connected$.next(true);
        }
      },
      closeObserver: {
        next: () => {
          console.log('[WebSocket] Disconnected');
          this.connected$.next(false);
        }
      }
    });

    // The messages stream — shared, with automatic reconnection on error
    const messages$ = this.socket$.pipe(
      // Automatically reconnect with exponential backoff on connection drop
      retry({
        count: 10,
        delay: (_, retryCount) => {
          const waitMs = Math.min(Math.pow(2, retryCount) * 1000, 30_000);
          console.log(`[WebSocket] Reconnecting in ${waitMs}ms...`);
          return timer(waitMs);
        }
      }),
      // Filter out PING messages and respond with PONG automatically
      tap(message => {
        if ((message as ServerMessage).type === 'PING') {
          this.socket$.next({ type: 'PONG' });
        }
      }),
      filter((msg): msg is ServerMessage => (msg as ServerMessage).type !== 'PING'),
      share() // Share one WebSocket connection across all subscribers
    );

    // Partition the message stream by type
    this.chatMessages$ = messages$.pipe(
      filter((msg): msg is ServerMessage & { type: 'CHAT_MESSAGE' } =>
        msg.type === 'CHAT_MESSAGE'
      )
    );

    this.userEvents$ = messages$.pipe(
      filter((msg): msg is ServerMessage & { type: 'USER_JOINED' | 'USER_LEFT' } =>
        msg.type === 'USER_JOINED' || msg.type === 'USER_LEFT'
      )
    );
  }

  sendMessage(text: string): void {
    this.socket$.next({ type: 'CHAT_MESSAGE', payload: { text } });
  }

  disconnect(): void {
    this.socket$.complete();
  }
}

// In a component
@Component({ /* ... */ })
export class ChatRoomComponent {
  messages = toSignal(
    this.chatService.chatMessages$.pipe(
      scan((history, msg) => [...history, msg.payload], [] as ChatMessage[])
    ),
    { initialValue: [] }
  );

  isConnected = toSignal(this.chatService.isConnected$, { initialValue: false });
}
```

---

## 15.10.3 — Polling with `interval` + `switchMap`

Polling is the practice of repeatedly fetching a resource at a fixed interval to check for updates. The reactive pattern for polling is combining `timer()` or `interval()` with `switchMap()` to trigger a fresh HTTP request on each tick.

```typescript
import { timer, Subject, merge } from 'rxjs';
import { switchMap, share, takeUntilDestroyed } from 'rxjs/operators';

@Injectable({ providedIn: 'root' })
export class NotificationPollingService {
  // Manual refresh trigger — lets components force an immediate poll
  private manualRefresh$ = new Subject<void>();

  // The polling stream — combines automatic interval with manual triggers
  readonly notifications$ = merge(
    timer(0, 30_000),    // Start immediately (0ms delay), then every 30 seconds
    this.manualRefresh$  // Also trigger on manual refresh
  ).pipe(
    switchMap(() =>
      this.http.get<Notification[]>('/api/notifications').pipe(
        catchError(err => {
          console.error('Polling failed:', err.message);
          return of([] as Notification[]);  // Return empty on error — don't kill the poll
        })
      )
    ),
    distinctUntilChanged((prev, curr) =>
      JSON.stringify(prev) === JSON.stringify(curr) // Only emit if notifications changed
    ),
    share() // All subscribers share one polling loop
  );

  refresh(): void {
    this.manualRefresh$.next();
  }
}
```

A few important design points in this pattern. Using `timer(0, 30_000)` instead of `interval(30_000)` makes the first poll happen immediately at subscription time (0ms initial delay), which is almost always what you want — the user should see data right away, not after the first interval. Using `switchMap` (rather than `mergeMap`) ensures that if a previous request is still in flight when the next poll fires (the server is slow), the old request is cancelled and a new one starts — preventing response ordering bugs. Using `catchError` inside the `switchMap` projection (not outside it) means an individual poll failure returns empty data but keeps the timer alive — the next tick will try again. Putting `catchError` outside `switchMap` would terminate the entire polling loop on the first failure.

---

## 15.10.4 — Search-as-you-type with `debounceTime` + `switchMap`

This is the most iconic RxJS pattern and the one most Angular developers encounter first. It requires careful composition of four operators working in concert: `debounceTime` (to avoid hammering the server with every keystroke), `distinctUntilChanged` (to avoid re-fetching when the value has not actually changed), `filter` (to avoid fetching for empty or trivial queries), and `switchMap` (to cancel pending requests when a newer query is available).

```typescript
import { Component, ViewChild, ElementRef, AfterViewInit } from '@angular/core';
import { fromEvent, Subject, Observable, of, EMPTY } from 'rxjs';
import {
  debounceTime, distinctUntilChanged, filter, switchMap,
  map, catchError, startWith, tap
} from 'rxjs/operators';

interface SearchState {
  results: SearchResult[];
  isLoading: boolean;
  query: string;
  error: string | null;
}

@Component({
  selector: 'app-search',
  standalone: true,
  template: `
    <input #searchInput type="text" placeholder="Search..." />

    @if ((state$ | async)?.isLoading) { <app-spinner /> }
    @if ((state$ | async)?.error; as error) { <p class="error">{{ error }}</p> }

    @for (result of (state$ | async)?.results; track result.id) {
      <app-search-result [result]="result" />
    }
  `
})
export class SearchComponent implements AfterViewInit {
  @ViewChild('searchInput') searchInput!: ElementRef<HTMLInputElement>;

  state$!: Observable<SearchState>;

  constructor(private searchService: SearchService) {}

  ngAfterViewInit() {
    const query$ = fromEvent<Event>(this.searchInput.nativeElement, 'input').pipe(
      map(event => (event.target as HTMLInputElement).value.trim()),
    );

    this.state$ = query$.pipe(
      debounceTime(350),         // Wait 350ms after the last keystroke
      distinctUntilChanged(),    // Only proceed if the value actually changed
      tap(query => {
        // Clear results immediately when query is empty — fast UX feedback
        if (!query) this.searchResultsCache = [];
      }),
      filter(query => query.length >= 2),  // Minimum 2 characters before searching

      // switchMap: cancel old requests, start fresh for newest query
      switchMap(query =>
        this.searchService.search(query).pipe(
          map(results => ({
            results,
            isLoading: false,
            query,
            error: null,
          })),
          catchError(err => of({
            results: [],
            isLoading: false,
            query,
            error: 'Search failed. Please try again.',
          })),
          startWith({  // Emit a loading state immediately before the HTTP call resolves
            results: [],
            isLoading: true,
            query,
            error: null,
          })
        )
      ),
      startWith({ results: [], isLoading: false, query: '', error: null })
    );
  }
}
```

The `startWith` inside the `switchMap` projection is a subtle but important detail. When a new query arrives, `switchMap` cancels the previous request and starts the new one. During the brief moment between the old request being cancelled and the new request resolving, you want to show a loading state. By emitting `{ isLoading: true }` as the first thing from the new inner Observable (via `startWith`), the template immediately reflects the loading state before the HTTP response arrives.

---

## 15.10.5 — Infinite Scroll with `concatMap`

Infinite scroll loads the next page of data when the user scrolls to the bottom of the list. The reactive pattern uses `IntersectionObserver` to detect when the "load more" sentinel element is visible, and `concatMap` to sequentially load each page (so page 3 is never requested before page 2 completes).

```typescript
import { Component, ElementRef, ViewChild, AfterViewInit } from '@angular/core';
import { Subject, BehaviorSubject } from 'rxjs';
import { concatMap, scan, takeUntil, switchMap, startWith, filter } from 'rxjs/operators';

@Component({
  selector: 'app-infinite-list',
  standalone: true,
  template: `
    @for (item of items(); track item.id) {
      <app-item [item]="item" />
    }

    <!-- Sentinel element: when this becomes visible, load the next page -->
    <div #loadMoreSentinel style="height: 1px;"></div>

    @if (isLoading()) { <app-loading-spinner /> }
    @if (hasReachedEnd()) { <p>All items loaded</p> }
  `
})
export class InfiniteListComponent implements AfterViewInit {
  @ViewChild('loadMoreSentinel') sentinel!: ElementRef;

  private nextPage$ = new Subject<void>();
  private allItems = signal<Item[]>([]);
  private isLoadingSignal = signal(false);
  private hasReachedEndSignal = signal(false);

  protected items = this.allItems.asReadonly();
  protected isLoading = this.isLoadingSignal.asReadonly();
  protected hasReachedEnd = this.hasReachedEndSignal.asReadonly();

  constructor(private itemService: ItemService) {}

  ngAfterViewInit() {
    // Create an IntersectionObserver to detect when the sentinel is visible
    const observer = new IntersectionObserver(entries => {
      const isVisible = entries[0]?.isIntersecting;
      if (isVisible && !this.isLoadingSignal() && !this.hasReachedEndSignal()) {
        this.nextPage$.next();
      }
    }, { rootMargin: '100px' }); // Start loading 100px before the sentinel is visible

    observer.observe(this.sentinel.nativeElement);

    // Use concatMap to load pages sequentially — page 2 never starts until page 1 finishes
    // Use scan to accumulate all loaded pages into a single growing array
    this.nextPage$.pipe(
      startWith(undefined),  // Trigger the first page immediately
      concatMap((_, index) => {
        const pageNumber = index + 1;
        this.isLoadingSignal.set(true);

        return this.itemService.getPage(pageNumber).pipe(
          catchError(err => {
            console.error('Failed to load page:', err);
            return of({ items: [], hasMore: false });
          })
        );
      }),
      scan((allItems, response) => {
        if (!response.hasMore) {
          this.hasReachedEndSignal.set(true);
        }
        return [...allItems, ...response.items];
      }, [] as Item[]),
      tap(() => this.isLoadingSignal.set(false)),
      takeUntilDestroyed()
    ).subscribe(items => this.allItems.set(items));
  }
}
```

The critical choices here are `concatMap` (not `mergeMap`) to ensure pages load in order, and `scan` to accumulate all pages. The alternative — a counter that increments on each load-more event and `switchMap` to the appropriate page — also works, but `concatMap` with `scan` is more elegant because the page number is implicit (the index in `concatMap`'s callback) rather than being a separate piece of state.

---

## 15.10.6 — Undo/Redo with `scan()`

An undo/redo system tracks a history of states and allows navigating backwards and forwards through that history. `scan()` is ideal for this because it maintains the history as its accumulated state, updating it in response to three types of events: a new state change, an undo request, and a redo request.

```typescript
type HistoryEvent<T> =
  | { type: 'CHANGE'; newState: T }
  | { type: 'UNDO' }
  | { type: 'REDO' };

interface HistoryState<T> {
  past: T[];      // Previous states (oldest first)
  present: T;     // Current state
  future: T[];    // States ahead of current (for redo)
}

function historyReducer<T>(
  history: HistoryState<T>,
  event: HistoryEvent<T>
): HistoryState<T> {
  switch (event.type) {
    case 'CHANGE':
      return {
        past: [...history.past, history.present],
        present: event.newState,
        future: [],  // Clear redo history when a new change is made
      };

    case 'UNDO': {
      if (history.past.length === 0) return history; // Nothing to undo
      const previous = history.past[history.past.length - 1];
      return {
        past: history.past.slice(0, -1),
        present: previous,
        future: [history.present, ...history.future],
      };
    }

    case 'REDO': {
      if (history.future.length === 0) return history; // Nothing to redo
      const next = history.future[0];
      return {
        past: [...history.past, history.present],
        present: next,
        future: history.future.slice(1),
      };
    }
  }
}

@Component({ selector: 'app-canvas-editor', standalone: true, /* ... */ })
export class CanvasEditorComponent {
  private events$ = new Subject<HistoryEvent<CanvasState>>();

  history$ = this.events$.pipe(
    scan(historyReducer, {
      past: [],
      present: initialCanvasState,
      future: [],
    }),
    startWith({ past: [], present: initialCanvasState, future: [] }),
    shareReplay(1)
  );

  canUndo$ = this.history$.pipe(map(h => h.past.length > 0));
  canRedo$ = this.history$.pipe(map(h => h.future.length > 0));
  currentState$ = this.history$.pipe(map(h => h.present));

  applyChange(newState: CanvasState) {
    this.events$.next({ type: 'CHANGE', newState });
  }
  undo() { this.events$.next({ type: 'UNDO' }); }
  redo() { this.events$.next({ type: 'REDO' }); }
}
```

This pattern is pure functional state management — no mutations, no class instances, no imperative loops. Every historical state is preserved as an immutable snapshot in the `past` and `future` arrays. The state machine transitions are pure functions (`historyReducer`) that are trivially testable.

---

## 15.10.7 — Optimistic Updates Pattern

Optimistic updates apply a change to the local UI state *immediately*, before the server confirms the change, giving the user instant feedback. If the server call fails, the change is rolled back. This pattern dramatically improves perceived performance for operations like toggling a like button, marking a todo as complete, or reordering a list.

```typescript
@Component({ selector: 'app-todo-list', standalone: true, /* ... */ })
export class TodoListComponent {
  private todoActions$ = new Subject<
    | { type: 'TOGGLE'; id: string; currentDone: boolean }
    | { type: 'TOGGLE_SUCCESS'; id: string }
    | { type: 'TOGGLE_ROLLBACK'; id: string; revertTo: boolean }
  >();

  // The todo list state, driven by optimistic actions
  todos$ = this.todoService.getAll().pipe(
    switchMap(initialTodos =>
      this.todoActions$.pipe(
        scan((todos, action) => {
          switch (action.type) {
            case 'TOGGLE':
              // Optimistic: flip the state immediately
              return todos.map(t =>
                t.id === action.id ? { ...t, done: !t.done } : t
              );
            case 'TOGGLE_SUCCESS':
              // Server confirmed — nothing to do, state is already correct
              return todos;
            case 'TOGGLE_ROLLBACK':
              // Server rejected — revert to original value
              return todos.map(t =>
                t.id === action.id ? { ...t, done: action.revertTo } : t
              );
          }
        }, initialTodos)
      )
    ),
    startWith([] as Todo[])
  );

  constructor(private todoService: TodoService) {
    // Wire up the side effects: when a TOGGLE action arrives, call the API
    this.todoActions$.pipe(
      filter(a => a.type === 'TOGGLE'),
      mergeMap(action => { // mergeMap: multiple toggles can be in-flight simultaneously
        const originalDone = action.currentDone;
        return this.todoService.toggle(action.id).pipe(
          map(() => ({ type: 'TOGGLE_SUCCESS' as const, id: action.id })),
          catchError(() => of({
            type: 'TOGGLE_ROLLBACK' as const,
            id: action.id,
            revertTo: originalDone  // Revert to the original state before the optimistic update
          }))
        );
      }),
      takeUntilDestroyed()
    ).subscribe(result => this.todoActions$.next(result as any));
  }

  toggleTodo(todo: Todo) {
    this.todoActions$.next({ type: 'TOGGLE', id: todo.id, currentDone: todo.done });
  }
}
```

---

## 15.10.8 — Complex Async Orchestration

Real applications often need to coordinate several async operations with complex ordering rules: "do A, then in parallel do B and C, then do D only if B succeeded, otherwise do E." The combination of `switchMap`, `forkJoin`, `concatMap`, `catchError`, and `combineLatest` lets you express these requirements as clean, readable declarative pipelines.

```typescript
// Scenario: User checkout flow
// 1. Validate cart (parallel with check user credit)
// 2. If both pass → reserve inventory → charge payment
// 3. If payment fails → release inventory → show payment error
// 4. If payment succeeds → send confirmation email (fire-and-forget)
// 5. Return final order regardless of email success

const checkoutResult$ = checkoutTrigger$.pipe(
  exhaustMap(({ cart, userId }) =>
    // Step 1: Run validation and credit check in parallel
    forkJoin({
      validation: cartService.validate(cart),
      credit: userService.checkCredit(userId, cart.total),
    }).pipe(
      // Step 2: Both passed — reserve inventory
      switchMap(({ validation, credit }) => {
        if (!validation.isValid) return throwError(() => new Error(validation.reason));
        if (!credit.approved) return throwError(() => new Error('Insufficient credit'));
        return inventoryService.reserve(cart.items);
      }),
      // Step 3: Reserve succeeded — charge payment
      switchMap(reservation =>
        paymentService.charge(userId, cart.total).pipe(
          // Payment failed — release the reservation
          catchError(paymentError =>
            inventoryService.release(reservation.id).pipe(
              switchMap(() => throwError(() => paymentError))
            )
          ),
          // Map to an object that carries both reservation and payment result
          map(payment => ({ reservation, payment }))
        )
      ),
      // Step 4: Payment succeeded — send email (fire and forget — never block the result)
      tap(({ payment }) => {
        emailService.sendConfirmation(userId, payment.orderId).subscribe({
          error: err => console.warn('Email failed (non-critical):', err.message)
        });
      }),
      // Step 5: Return the final order regardless of email outcome
      map(({ payment }) => ({
        success: true,
        orderId: payment.orderId,
        message: 'Order placed successfully!',
      })),
      catchError(err => of({
        success: false,
        orderId: null,
        message: err.message,
      }))
    )
  )
);
```

The `exhaustMap` at the top ensures that if the user double-clicks the checkout button, the second click is ignored while the first checkout is in progress. Each step is a separate `switchMap` or `forkJoin` that clearly expresses its role. Error handling is co-located with the operation that can fail, and the final `catchError` at the end provides a unified error format regardless of which step failed.

---

## Important Points and Best Practices

The state machine with `scan()` pattern is the most important advanced pattern to internalize. Once you see UI state as "a current state driven by a stream of events," many complex problems become mechanical. Define your states as a discriminated union, define your events, write a pure reducer function, and wire it with `scan()`. The resulting code is testable in complete isolation from Angular, because the reducer is just a pure function.

For WebSocket reconnection, always put the `retry()` on the `socket$` itself, not on individual message subscriptions. The socket connection is the resource that needs to be re-established — retrying individual subscriptions to an already-dead socket will not help. The pattern shown in 15.10.2, where `retry()` is applied immediately after `this.socket$` in the shared messages stream, is the correct approach.

With optimistic updates, always save the original state *before* dispatching the optimistic action, so you have a concrete rollback target if the server call fails. Trying to "reverse" the optimistic change by applying the opposite action is fragile if the operation is not perfectly invertible. Storing the original and restoring it on failure is always reliable.

The search-as-you-type pattern's `startWith({ isLoading: true })` inside `switchMap` is a subtlety that many developers miss. Without it, there is a flash between the old results being cleared (when `switchMap` cancels the old request) and the loading indicator appearing (only after the new request emits). The `startWith` inside the projection ensures the loading state is emitted synchronously the moment the new query fires, eliminating that flash.

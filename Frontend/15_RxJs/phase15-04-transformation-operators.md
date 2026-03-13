# Phase 15.4 — Transformation Operators
## map, mergeMap, concatMap, switchMap, exhaustMap, scan, reduce, groupBy, buffer, window, toArray, pairwise, startWith

---

## What Are Transformation Operators?

Transformation operators take values from a source Observable, do something with them, and emit transformed values on a new Observable. They are the workhorses of every reactive pipeline — the functional tools that shape, reshape, flatten, and accumulate streams of data. Understanding these operators — especially the four "flattening" operators (`mergeMap`, `concatMap`, `switchMap`, `exhaustMap`) — is arguably the single most important skill in RxJS mastery.

---

## 15.4.1 — `map()` — Transform Each Value

`map()` is the fundamental transformation operator. It applies a projection function to each emitted value and emits the result. One input value always produces exactly one output value. The marble diagram makes this explicit: every emission is transformed and re-emitted at the same time frame.

```
Source:  --1--2--3--|
map(x => x * 10)
Result:  --10-20-30-|
```

```typescript
import { of } from 'rxjs';
import { map } from 'rxjs/operators';

// Transform numbers
of(1, 2, 3).pipe(
  map(n => n * n)
).subscribe(v => console.log(v)); // 1, 4, 9

// Transform objects
this.http.get<ApiResponse<User[]>>('/api/users').pipe(
  map(response => response.data),          // Extract the nested data
  map(users => users.filter(u => u.active)) // Filter active users
).subscribe(users => this.activeUsers = users);

// Transform event objects into useful data
fromEvent<MouseEvent>(document, 'click').pipe(
  map(event => ({ x: event.clientX, y: event.clientY })),
  map(coords => `Clicked at ${coords.x}, ${coords.y}`)
).subscribe(message => console.log(message));
```

The key mental model for `map()` is the mathematical function: one input, one output, same timing, no side effects. Every other transformation operator is built on top of this concept but adds complexity in the form of time, flattening, or accumulation.

---

## 15.4.2 — The Four Flattening Operators: The Heart of RxJS

This is the most important section in Phase 15. The four operators — `mergeMap`, `concatMap`, `switchMap`, `exhaustMap` — all do the same fundamental thing: they take each value from a source Observable, pass it to a function that returns a new "inner" Observable, and then *flatten* the emissions from that inner Observable into the outer stream. The critical difference between them is **what they do when a new value arrives from the source while an inner Observable is still active**.

Before diving in, let's establish the core mental model. You have an outer stream emitting "triggers," and for each trigger you want to start some async operation (like an HTTP request) that itself returns a stream. How do you manage multiple overlapping async operations? That is precisely what each flattening operator answers differently.

---

## 15.4.3 — `mergeMap()` — Parallel: All Inner Observables Run Concurrently

`mergeMap()` subscribes to every inner Observable as it is created, without cancelling previous ones. All inner subscriptions run concurrently, and all their emissions are merged into the output stream. The order of output values depends purely on timing — whichever inner Observable emits first, that value comes out first.

```
Source:      --a---------b----------|
Inner for a:   --1--2--3|
Inner for b:             --4--5--6|
Result:      ----1--2--3----4--5--6|

(No overlap in this diagram, but if they overlapped, outputs would interleave)
```

```typescript
import { fromEvent, mergeMap, interval } from 'rxjs';
import { take } from 'rxjs/operators';

// Each click starts its OWN independent timer — all run concurrently
// Click fast enough and you will have multiple timers running at once
fromEvent(document, 'click').pipe(
  mergeMap(() => interval(500).pipe(take(3)))
).subscribe(v => console.log(v));
// Click 1: starts timer → 0, 1, 2
// Click 2 (before timer 1 finishes): starts ANOTHER timer → 0, 1, 2
// Output interleaves: 0, 0, 1, 1, 2, 2

// Real-world use: parallel downloads (order doesn't matter, all should complete)
const fileUrls = ['url1', 'url2', 'url3'];
from(fileUrls).pipe(
  mergeMap(url => this.http.get(url)) // All 3 requests fire simultaneously
).subscribe(data => processFile(data)); // Process each as it arrives

// mergeMap with concurrency limit — at most 2 at a time
from(fileUrls).pipe(
  mergeMap(url => this.http.get(url), 2) // Third request waits until one of the first two finishes
).subscribe(data => processFile(data));
```

**When to use `mergeMap`:** When each inner Observable is independent, the order of results does not matter, and you want maximum throughput. Use it for analytics event logging, parallel downloads, fire-and-forget operations, and any scenario where you want true concurrent execution.

**The danger:** `mergeMap` makes no attempt to coordinate or limit concurrent subscriptions. If a very fast source produces many values and each triggers a slow inner Observable, you can accumulate a large number of simultaneous subscriptions, potentially overwhelming a server or causing memory issues.

---

## 15.4.4 — `concatMap()` — Sequential: Inner Observables Queue Up

`concatMap()` subscribes to inner Observables *one at a time*, in the order they are created. It waits for each inner Observable to **complete** before subscribing to the next one. New values from the source are buffered — they are not lost, they wait in a queue. This guarantees ordering: the output values from the first inner Observable always appear before the output values from the second.

```
Source:      --a-----b-----c--|
Inner for a:   --1--2|
Inner for b:         --3--4|   (b was buffered while a's inner ran)
Inner for c:               --5--6|
Result:      ----1--2--3--4--5--6|

All in order, no overlap, none dropped
```

```typescript
import { from, concatMap, of } from 'rxjs';
import { delay } from 'rxjs/operators';

// Sequential HTTP requests — each must complete before the next starts
const userIds = [1, 2, 3];
from(userIds).pipe(
  concatMap(id => this.http.get<User>(`/api/users/${id}`))
).subscribe(user => console.log(user));
// User 1 is fetched, logged, THEN user 2 is fetched, etc.
// Order is always maintained even if network timing varies

// Processing a queue of tasks in order
const tasks$ = new Subject<Task>();
tasks$.pipe(
  concatMap(task => processTask(task)) // Each task completes before the next starts
).subscribe(result => console.log('Task done:', result));

tasks$.next(taskA);
tasks$.next(taskB);
tasks$.next(taskC);
// A completes, then B starts, then B completes, then C starts...
```

**When to use `concatMap`:** When order matters and you cannot run inner Observables in parallel. Use it for sequential database migrations, ordered API calls (create → update → notify), processing a queue where each item must complete before the next begins, and any operation where race conditions between parallel requests would cause data corruption.

**The danger:** If the source emits faster than inner Observables complete, the buffer grows unboundedly. If the source never stops and inner Observables are slow, you have a growing backlog. For high-frequency sources, consider whether you actually need to process every single value.

---

## 15.4.5 — `switchMap()` — Cancel Previous: Only the Latest Matters

`switchMap()` is the most commonly used flattening operator in Angular applications. When a new value arrives from the source, it **cancels the currently active inner Observable** (unsubscribes from it, aborting any pending work) and starts a new inner Observable for the new source value. At any given moment, only ONE inner Observable is active.

```
Source:       --a-----b----------c--|
Inner for a:    --1--2...cancelled
Inner for b:          --3--4--5|
Inner for c:                     --6--7|
Result:       --------3--4--5------6--7|

Note: 1 and 2 are never emitted because 'b' arrived before they completed
```

```typescript
import { fromEvent, switchMap, debounceTime, map } from 'rxjs';

// CLASSIC ANGULAR PATTERN: search-as-you-type
// Every new keystroke cancels the previous HTTP request
const searchInput = document.getElementById('search') as HTMLInputElement;
fromEvent(searchInput, 'input').pipe(
  map(e => (e.target as HTMLInputElement).value),
  debounceTime(300),
  // OLD request cancelled, NEW request starts — prevents stale responses
  switchMap(query => this.http.get<SearchResult[]>(`/api/search?q=${query}`))
).subscribe(results => this.results = results);

// Route-based data loading — navigate to new route cancels previous load
this.route.paramMap.pipe(
  map(params => params.get('id')!),
  switchMap(id => this.http.get<Product>(`/api/products/${id}`))
).subscribe(product => this.product = product);

// User preference change triggers new data fetch — only latest fetch matters
this.selectedCategory$.pipe(
  switchMap(category => this.http.get<Item[]>(`/api/items?category=${category}`))
).subscribe(items => this.items = items);
```

**When to use `switchMap`:** When you only care about the result of the *most recent* trigger and want previous in-flight operations to be cancelled. This is the default choice for user-driven searches, navigation-triggered data loads, real-time filter updates, and any "show me the latest result for the latest input" pattern.

**The critical gotcha:** If you use `switchMap` for operations that cause side effects (like POST/DELETE requests), cancelling the Observable subscription does NOT undo the server-side operation. The HTTP request was sent — the server may have already processed it. If you use `switchMap` with a "save user" operation and the user saves twice rapidly, the first save request may still complete on the server even though you unsubscribed from its Observable. For write operations, use `concatMap` (queue saves) or `exhaustMap` (ignore second save while first is in flight) instead.

---

## 15.4.6 — `exhaustMap()` — Ignore New While Busy: Current Takes Priority

`exhaustMap()` is the opposite strategy from `switchMap`. When a new value arrives from the source while an inner Observable is still active, `exhaustMap` **ignores the new value completely** — it does not buffer it, does not cancel the current inner Observable, it simply drops it. When the current inner Observable completes, `exhaustMap` goes back to accepting new values from the source.

```
Source:       --a--b--c---------d--|
Inner for a:    ----1--2--3|      (a's inner runs uninterrupted)
Inner for b:       IGNORED        (b arrives while a's inner is running)
Inner for c:          IGNORED     (c arrives while a's inner is running)
Inner for d:                 --4--5|  (d accepted after a's inner completed)
Result:       ------1--2--3------4--5|
```

```typescript
import { fromEvent, exhaustMap } from 'rxjs';

// Form submission — ignore repeated clicks while submitting
// This prevents double-submit bugs
fromEvent(submitButton, 'click').pipe(
  exhaustMap(() => this.http.post('/api/orders', this.form.value))
).subscribe(order => {
  console.log('Order created:', order);
});
// User clicks submit → request starts
// User clicks again impatiently → IGNORED, original request continues
// Original request completes → next click is now accepted

// Login form — prevent multiple simultaneous login requests
fromEvent(loginButton, 'click').pipe(
  exhaustMap(() => this.authService.login(this.credentials))
).subscribe({
  next: () => this.router.navigate(['/dashboard']),
  error: err => this.loginError = err.message,
});
```

**When to use `exhaustMap`:** When triggering an operation that should run to completion without interruption, and new triggers during that time should be discarded. Classic use cases include form submissions, payment processing, login buttons, and any "idempotent-only-once-at-a-time" operations.

### The Flattening Operator Decision Summary

To consolidate the four operators in one mental picture: ask yourself what should happen when a NEW source value arrives while a PREVIOUS inner Observable is still running. If you want BOTH to run in parallel, use `mergeMap`. If you want to QUEUE the new one until the previous finishes, use `concatMap`. If you want to CANCEL the previous and start fresh with the new, use `switchMap`. If you want to IGNORE the new one entirely until the previous completes, use `exhaustMap`.

---

## 15.4.7 — `scan()` — Running Accumulation

`scan()` applies an accumulator function to each emitted value, maintaining a running state and emitting the accumulated result after each value. It is the streaming version of `Array.reduce()` — the critical difference being that `scan()` emits *intermediate results* after each step, while `reduce()` only emits the final result when the stream completes.

```typescript
import { fromEvent, scan, map } from 'rxjs';
import { Subject } from 'rxjs';

// Running total of click counts
const counter$ = fromEvent(document, 'click').pipe(
  scan((total, _) => total + 1, 0) // acc, current value, seed
);
counter$.subscribe(count => console.log(`Clicks so far: ${count}`)); // 1, 2, 3...

// Accumulate a shopping cart from item events
interface CartAction {
  type: 'ADD' | 'REMOVE';
  item: CartItem;
}

const cartActions$ = new Subject<CartAction>();
const cart$ = cartActions$.pipe(
  scan((cart: CartItem[], action: CartAction) => {
    if (action.type === 'ADD') {
      return [...cart, action.item];
    } else {
      return cart.filter(i => i.id !== action.item.id);
    }
  }, [] as CartItem[]) // Initial accumulator value: empty cart
);

cart$.subscribe(cart => console.log('Cart:', cart));
cartActions$.next({ type: 'ADD', item: { id: '1', name: 'Widget', price: 9.99 } });
cartActions$.next({ type: 'ADD', item: { id: '2', name: 'Gadget', price: 19.99 } });
cartActions$.next({ type: 'REMOVE', item: { id: '1', name: 'Widget', price: 9.99 } });
// Cart: [{id:'1',...}]
// Cart: [{id:'1',...}, {id:'2',...}]
// Cart: [{id:'2',...}]

// State machine using scan
type UIState = 'idle' | 'loading' | 'success' | 'error';
type UIEvent = 'FETCH' | 'SUCCESS' | 'FAILURE' | 'RESET';

const stateTransitions: Record<UIState, Partial<Record<UIEvent, UIState>>> = {
  idle:    { FETCH: 'loading' },
  loading: { SUCCESS: 'success', FAILURE: 'error' },
  success: { FETCH: 'loading', RESET: 'idle' },
  error:   { FETCH: 'loading', RESET: 'idle' },
};

const events$ = new Subject<UIEvent>();
const state$ = events$.pipe(
  scan((state: UIState, event: UIEvent) => {
    return stateTransitions[state][event] ?? state;
  }, 'idle' as UIState)
);
```

`scan()` is the foundational tool for implementing state machines, real-time aggregations, running statistics, and any pattern where you need to maintain state across a stream of events.

---

## 15.4.8 — `reduce()` — Final Accumulation

`reduce()` works exactly like `scan()` but emits only the *final accumulated value* after the source stream completes. It does not emit intermediate results. If the source never completes, `reduce()` never emits.

```typescript
import { from, reduce } from 'rxjs';

// Sum all numbers — only emits the final total
from([1, 2, 3, 4, 5]).pipe(
  reduce((acc, val) => acc + val, 0)
).subscribe(total => console.log('Total:', total)); // Total: 15

// Collect all items into a single array — equivalent of toArray()
from([1, 2, 3]).pipe(
  reduce((acc, val) => [...acc, val], [] as number[])
).subscribe(all => console.log(all)); // [1, 2, 3]
```

Use `scan()` when you need each intermediate state (streaming UI updates, real-time dashboards). Use `reduce()` when you only care about the final result after all processing is complete (batch operations, computing final statistics from a finite data set).

---

## 15.4.9 — `groupBy()` — Group Emissions by Key

`groupBy()` splits a single Observable into multiple Observables, each grouping emissions that share the same key. Each group is itself an Observable that you typically need to process with another operator (usually `mergeMap`).

```typescript
import { from, groupBy, mergeMap, toArray, reduce } from 'rxjs';

const orders = [
  { id: 1, category: 'electronics', total: 299 },
  { id: 2, category: 'books', total: 29 },
  { id: 3, category: 'electronics', total: 599 },
  { id: 4, category: 'books', total: 19 },
  { id: 5, category: 'clothing', total: 89 },
];

// Group by category and sum totals
from(orders).pipe(
  groupBy(order => order.category),      // Returns Observable<GroupedObservable>
  mergeMap(group$ =>
    group$.pipe(
      reduce((acc, order) => acc + order.total, 0),
      map(total => ({ category: group$.key, total }))
    )
  )
).subscribe(result => console.log(result));
// { category: 'electronics', total: 898 }
// { category: 'books', total: 48 }
// { category: 'clothing', total: 89 }
```

`groupBy()` is most useful for real-time data classification — grouping incoming WebSocket messages by type, categorizing log entries by severity, or partitioning stream events by user ID.

---

## 15.4.10 — `bufferTime()` and `bufferCount()` — Collect Values into Arrays

Buffer operators collect multiple emissions from the source and group them into arrays, emitting those arrays as single values. This is useful for batching — when you want to process groups of events together rather than each one individually.

`bufferTime(milliseconds)` collects all emissions within a time window and emits them as an array.  
`bufferCount(count)` collects a fixed number of emissions and emits them as an array.

```typescript
import { fromEvent, bufferTime, bufferCount, filter } from 'rxjs';

// Collect all clicks within 1-second windows
fromEvent(document, 'click').pipe(
  bufferTime(1000),
  filter(clicks => clicks.length > 0) // Don't emit empty arrays
).subscribe(clicks => {
  console.log(`${clicks.length} clicks in the last second`);
});

// Batch analytics events — send in groups of 10 to reduce HTTP requests
const analyticsEvent$ = new Subject<AnalyticsEvent>();
analyticsEvent$.pipe(
  bufferCount(10) // Wait for 10 events, then emit the array
).subscribe(batch => {
  this.http.post('/api/analytics', { events: batch }).subscribe();
});

// bufferTime with start boundary — sliding window
fromEvent(document, 'mousemove').pipe(
  bufferTime(500, 100) // 500ms window, new window every 100ms
).subscribe(events => updateHeatmap(events));
```

---

## 15.4.11 — `windowTime()` and `windowCount()` — Split into Sub-Observables

Window operators are similar to buffer operators but instead of emitting arrays, they emit **Observables** — each "window" is a stream that you can process with further operators. This makes them more flexible for complex windowing scenarios.

```typescript
import { interval, windowCount, mergeMap, toArray } from 'rxjs';

// Every 5 emissions from the source, create a new window
interval(100).pipe(
  windowCount(5),
  mergeMap(window$ => window$.pipe(toArray())) // Collect each window into an array
).subscribe(window => console.log('Window:', window));
// Window: [0, 1, 2, 3, 4]
// Window: [5, 6, 7, 8, 9]
// Window: [10, 11, 12, 13, 14]...
```

Use buffer operators when you want arrays. Use window operators when you need to apply Observable operators to each group of values independently.

---

## 15.4.12 — `toArray()` — Collect All Values, Emit Once

`toArray()` collects all values emitted by the source into a single array and emits it once when the source completes. It is essentially `reduce((acc, val) => [...acc, val], [])`.

```typescript
import { from, toArray, map } from 'rxjs';

from([1, 2, 3, 4, 5]).pipe(
  map(n => n * 2),
  toArray()
).subscribe(arr => console.log(arr)); // [2, 4, 6, 8, 10]

// Useful when processing paginated data and wanting all results at once
const allPages$ = from([1, 2, 3]).pipe(
  concatMap(page => this.http.get<Item[]>(`/api/items?page=${page}`)),
  toArray(),
  map(pages => pages.flat()) // [[...], [...], [...]] → [...]
);
```

---

## 15.4.13 — `pairwise()` — Previous and Current Value

`pairwise()` groups consecutive emissions into pairs: `[previous, current]`. It waits for at least two emissions before emitting anything — the first emission is held until the second arrives.

```typescript
import { fromEvent, pairwise, map } from 'rxjs';

// Calculate mouse movement delta
fromEvent<MouseEvent>(document, 'mousemove').pipe(
  map(e => ({ x: e.clientX, y: e.clientY })),
  pairwise(), // Emits [previous position, current position]
  map(([prev, curr]) => ({
    dx: curr.x - prev.x,
    dy: curr.y - prev.y,
    speed: Math.sqrt((curr.x - prev.x) ** 2 + (curr.y - prev.y) ** 2)
  }))
).subscribe(movement => console.log('Speed:', movement.speed.toFixed(1)));

// Detect direction changes in a number stream
import { from } from 'rxjs';
from([1, 3, 7, 5, 8, 2]).pipe(
  pairwise(),
  map(([prev, curr]) => ({ value: curr, trend: curr > prev ? '↑' : '↓' }))
).subscribe(v => console.log(v));
// { value: 3, trend: '↑' }, { value: 7, trend: '↑' }, { value: 5, trend: '↓' }...
```

---

## 15.4.14 — `startWith()` — Prepend an Initial Value

`startWith()` emits one or more values synchronously *before* the source begins emitting. It is the reactive equivalent of prepending an element to a sequence.

```typescript
import { fromEvent, startWith, scan } from 'rxjs';

// Ensure there's always a value in the stream before the first event
fromEvent(window, 'resize').pipe(
  startWith(null),             // Emit null immediately so subscription fires on init
  map(() => window.innerWidth)
).subscribe(width => updateLayout(width));
// Fires immediately with current width, then on every resize

// Provide an initial loading state before data arrives
this.http.get<Product[]>('/api/products').pipe(
  map(products => ({ status: 'loaded' as const, products })),
  startWith({ status: 'loading' as const, products: [] })
).subscribe(state => {
  if (state.status === 'loading') showSpinner();
  else { hideSpinner(); renderProducts(state.products); }
});

// For NgRx actions stream — ensure store has initial value
actions$.pipe(
  ofType(loadProducts),
  startWith(loadProducts())  // Trigger initial load immediately on subscription
);
```

`startWith()` is particularly powerful in Angular when combined with `combineLatest` — if one stream has not yet emitted, `combineLatest` will not emit. Providing `startWith(initialValue)` on each stream ensures `combineLatest` emits immediately upon subscription.

---

## Important Points and Best Practices

The flattening operators (`mergeMap`, `concatMap`, `switchMap`, `exhaustMap`) are the hardest concept in RxJS and also the most important. Memorize the one-sentence decision rule: **switch for latest, exhaust for busy, concat for order, merge for parallel**. Print it out and stick it next to your monitor until it is automatic.

Never use `mergeMap` for form submissions, payment operations, or any mutation where running two copies simultaneously would cause data corruption or duplicate entries in a database. In those cases, use `exhaustMap` (ignore the second tap) or `concatMap` (queue and process in order).

Never use `switchMap` for write operations (POST, PUT, DELETE). If the user triggers a "delete" and then immediately triggers another "delete," `switchMap` will cancel the Observable for the first delete request — but the HTTP request was already sent to the server. The server may process both deletions. Use `exhaustMap` for idempotent writes or `concatMap` for ordered writes.

`scan()` is dramatically underused. Many developers reach for a `BehaviorSubject` + imperative mutation when `scan()` with a pure accumulator function would express the same pattern more clearly and testably. If you find yourself subscribing to a stream and calling `.next()` on a Subject inside the subscription, consider whether `scan()` could replace the whole pattern.

Chain `map()` calls rather than combining all transformations into one complex projection. Separate `map()` calls are easier to read, test individually, and understand in isolation. A pipeline of five simple `map()` calls is far more maintainable than one `map()` with a 15-line arrow function.

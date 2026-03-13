# Phase 15.7 & 15.8 — Error Handling and Utility Operators
## catchError, retry, throwIfEmpty, onErrorResumeNext, tap, delay, timeout, finalize, share, shareReplay, publish, observeOn, subscribeOn

---

# PART 1: Phase 15.7 — Error Handling Operators

## Why Error Handling in Streams Is Different

Error handling in RxJS requires a different mental model than try/catch in synchronous code. In synchronous JavaScript, an uncaught error propagates up the call stack until a try/catch intercepts it. In reactive programming, an error in a stream terminates that stream — no more `next` emissions will arrive after an `error` notification. The stream is dead. This means you must plan your error handling *inside the pipeline*, not outside it, and you must decide whether to recover the stream (return a replacement Observable), re-throw a different error, or accept the termination.

The RxJS error handling operators are your tools for building resilient, fault-tolerant data pipelines.

---

## 15.7.1 — `catchError()` — Intercept and Recover

`catchError()` intercepts an error in the pipeline and gives you the chance to return a replacement Observable. Whatever Observable you return from `catchError`'s handler becomes the new stream — values from it are emitted as if nothing went wrong. The original Observable that errored is discarded.

```typescript
import { catchError, of, throwError, EMPTY } from 'rxjs';

// Strategy 1: Return a fallback value — the stream continues normally
this.http.get<Product[]>('/api/products').pipe(
  catchError(err => {
    console.error('Failed to load products:', err.message);
    return of([]); // Return empty array — the subscriber sees [] instead of an error
  })
).subscribe(products => this.products = products);
// Subscriber never sees the error — it gets an empty array instead

// Strategy 2: Return EMPTY — the stream completes silently with no values
this.http.get<Analytics>('/api/optional-analytics').pipe(
  catchError(() => EMPTY) // Fail silently — no value, no error, just completes
).subscribe(data => this.analyticsData = data);
// subscriber.complete() fires; subscriber.next() never fires

// Strategy 3: Re-throw a transformed error — change type or add context
this.http.get<User>('/api/user').pipe(
  catchError(httpError => {
    const appError = new AppError(
      httpError.status === 404 ? 'USER_NOT_FOUND' : 'NETWORK_ERROR',
      httpError.message
    );
    return throwError(() => appError);
  })
).subscribe({
  next: user => this.user = user,
  error: (err: AppError) => this.showErrorMessage(err.code),
});

// Strategy 4: Conditional recovery based on error type
this.http.get<Data>('/api/data').pipe(
  catchError(err => {
    if (err.status === 401) {
      // Token expired — refresh and retry
      return this.authService.refreshToken().pipe(
        switchMap(() => this.http.get<Data>('/api/data'))
      );
    }
    if (err.status === 503) {
      // Service unavailable — return cached data
      return this.cacheService.get<Data>('last-data');
    }
    // All other errors — rethrow
    return throwError(() => err);
  })
);
```

A critically important detail: `catchError` receives the error AND the source Observable as arguments. The second argument (the caught source Observable) lets you re-subscribe to the original Observable, effectively implementing a retry. However, this pattern can cause infinite loops — always guard it carefully. The explicit `retry()` operator (covered next) is safer for this use case.

```typescript
import { catchError } from 'rxjs/operators';

// DANGEROUS — can cause infinite loop if the error is persistent
source$.pipe(
  catchError((err, caught$) => caught$) // Re-subscribe forever on any error
);

// BETTER — use retry() instead (it has built-in limits and config)
```

---

## 15.7.2 — `retry()` — Automatically Re-subscribe on Error

`retry()` re-subscribes to the source Observable when it errors, giving the source another chance to succeed. Because most creation operators (like `HttpClient.get()`) produce cold Observables, re-subscribing effectively retries the operation from scratch — a new HTTP request is made, a new timer starts, etc.

Without any arguments, `retry()` retries indefinitely — avoid this as it can cause infinite request loops. Always configure a maximum retry count and ideally a delay strategy.

```typescript
import { retry } from 'rxjs/operators';
import { timer } from 'rxjs';

// Basic retry: try up to 3 times total (1 original + 2 retries)
this.http.get<Data>('/api/data').pipe(
  retry(2) // Short form — retry 2 times before erroring
).subscribe({
  next: data => this.data = data,
  error: err => this.handleFinalFailure(err),
});

// Advanced retry with exponential backoff
// This is the production-ready retry pattern
this.http.get<Data>('/api/data').pipe(
  retry({
    count: 3,              // Maximum 3 retry attempts
    delay: (error, retryCount) => {
      // Exponential backoff: wait 1s, then 2s, then 4s
      const waitMs = Math.pow(2, retryCount - 1) * 1000;
      console.log(`Retry ${retryCount} in ${waitMs}ms after error:`, error.message);
      return timer(waitMs);
    },
    resetOnSuccess: true,  // Reset retry counter if a later emission succeeds
  })
).subscribe({
  next: data => this.data = data,
  error: err => console.error('All retries exhausted:', err.message),
});

// Only retry on specific error types
this.http.get<Data>('/api/data').pipe(
  retry({
    count: 3,
    delay: (error, retryCount) => {
      // Only retry on network errors (5xx), not client errors (4xx)
      if (error.status >= 400 && error.status < 500) {
        return throwError(() => error); // Don't retry — re-throw immediately
      }
      return timer(retryCount * 1000); // Retry with linear backoff for server errors
    }
  })
);
```

The `resetOnSuccess: true` option is important for long-lived streams (like WebSocket reconnection) where a successful period should reset the retry count, allowing the full retry budget to be available for the next failure.

---

## 15.7.3 — `throwIfEmpty()` — Error If Stream Completes Without Emitting

`throwIfEmpty()` throws an error if the source Observable completes without emitting any values. It is the guard operator for "this stream must produce at least one value — if it doesn't, that's an error."

```typescript
import { throwIfEmpty, EMPTY } from 'rxjs';

// Ensure the user lookup produces a result
this.http.get<User[]>('/api/users').pipe(
  mergeMap(users => from(users)),
  filter(user => user.role === 'admin'),
  throwIfEmpty(() => new Error('No admin users found in the system')),
  first() // Take the first admin user
).subscribe({
  next: admin => this.adminUser = admin,
  error: err => this.showCriticalError(err.message),
});

// Without throwIfEmpty, the observable would just complete silently
// if no admin users exist — throwIfEmpty converts that silent completion into an error
```

---

## 15.7.4 — `onErrorResumeNext()` — Continue with Next Sources on Error

`onErrorResumeNext()` is a combination-and-error-handling operator. It concatenates multiple Observables but skips to the next source whenever the current source errors (rather than propagating the error). The combined stream never errors — it just moves on.

```typescript
import { onErrorResumeNext, of, throwError } from 'rxjs';

// Try primary, then fallback1, then fallback2 — errors are silently skipped
onErrorResumeNext(
  this.http.get<Data>('/api/primary-endpoint'),      // Might fail
  this.http.get<Data>('/api/fallback-endpoint'),      // Might also fail
  of({ data: null, source: 'default' })               // Always succeeds
).subscribe(data => this.data = data);

// The subscriber receives the FIRST source that produces a value
// Without needing to write multiple catchError handlers
```

`onErrorResumeNext` is relatively rare in practice — `catchError` with conditional fallback logic is usually more explicit and easier to understand. But it is concise for the "try these in order, use whichever succeeds" pattern.

---

---

# PART 2: Phase 15.8 — Utility Operators

---

## 15.8.1 — `tap()` — Side Effects Without Modifying the Stream

`tap()` (formerly `do()`) lets you perform side effects for each emission *without modifying the stream in any way*. The value passes through tap unchanged. Use it for logging, debugging, updating external state, triggering analytics, or any operation that should happen in response to an emission but must not alter what downstream receives.

```typescript
import { tap } from 'rxjs/operators';

// Logging for debugging — the canonical use of tap()
this.http.get<Product[]>('/api/products').pipe(
  tap(products => console.log('[API Response]', products)),  // Log the raw data
  map(products => products.filter(p => p.active)),
  tap(active => console.log('[After filter]', active.length, 'active products')),
).subscribe(products => this.products = products);

// Update loading state on the way in
this.http.get<User>('/api/user').pipe(
  tap(() => this.isLoading.set(true)),  // Side effect: set loading spinner
  finalize(() => this.isLoading.set(false)),  // Clear it when done (see finalize below)
).subscribe(user => this.user = user);

// Analytics — track when certain values flow through
searchQuery$.pipe(
  debounceTime(300),
  filter(query => query.length > 2),
  tap(query => this.analytics.track('search', { query })), // Log to analytics
  switchMap(query => this.http.get(`/api/search?q=${query}`))
).subscribe(results => this.results = results);

// tap also accepts an Observer object — useful for handling error and complete
source$.pipe(
  tap({
    next: v => console.log('Value:', v),
    error: e => console.error('Error in tap:', e),
    complete: () => console.log('Stream completed'),
  })
)
```

A key principle: `tap()` should only contain *side effects* — operations with no return value that affect the world outside the stream. Never compute a new value inside `tap()` and return it — that is what `map()` is for. The split of responsibilities is: `map()` transforms values, `tap()` reacts to values.

---

## 15.8.2 — `delay()` and `delayWhen()` — Postpone Emissions

`delay(ms)` adds a fixed time offset to every emission. Every value that would have been emitted at time T is instead emitted at time T + `ms`. The complete and error notifications are also delayed.

```typescript
import { delay, delayWhen, of, timer } from 'rxjs';

// Delay the entire response by 1 second (useful for testing loading states)
this.http.get<Data>('/api/data').pipe(
  delay(1000) // ONLY for development — simulates slow network
).subscribe(data => this.data = data);

// Show a notification, then auto-dismiss after 3 seconds
const notification$ = of({ message: 'Saved!', type: 'success' });
notification$.subscribe(n => this.showNotification(n));
notification$.pipe(delay(3000)).subscribe(() => this.hideNotification());

// Stagger a list of animations
from(itemList).pipe(
  concatMap((item, index) =>
    of(item).pipe(delay(index * 100)) // Each item delayed 100ms more than previous
  )
).subscribe(item => animateIn(item));
```

`delayWhen(durationSelector)` is the dynamic version — the delay duration is determined by an Observable returned from the selector function. The value is emitted when that duration Observable emits.

```typescript
import { delayWhen, interval } from 'rxjs';

// Delay each value by a variable amount
from([1, 2, 3]).pipe(
  delayWhen(value => timer(value * 500)) // 1 → 500ms, 2 → 1000ms, 3 → 1500ms
).subscribe(v => console.log(v));
// 1 appears at 500ms, 2 at 1000ms, 3 at 1500ms
```

---

## 15.8.3 — `timeout()` — Error If No Value Arrives in Time

`timeout()` sets a deadline for the source Observable. If no value is emitted within the specified time (from the last emission, or from subscription time), it terminates the stream with a `TimeoutError`. This is essential for preventing forever-pending requests from silently blocking your UI.

```typescript
import { timeout, catchError, of } from 'rxjs';
import { TimeoutError } from 'rxjs';

// Error if the HTTP request takes more than 5 seconds
this.http.get<Data>('/api/slow-endpoint').pipe(
  timeout(5000),
  catchError(err => {
    if (err instanceof TimeoutError) {
      return of({ data: null, timedOut: true });
    }
    return throwError(() => err);
  })
).subscribe(result => {
  if (result.timedOut) {
    this.showTimeoutMessage();
  } else {
    this.data = result.data;
  }
});

// Advanced timeout config — different deadlines for first vs subsequent values
interval(500).pipe(
  timeout({
    first: 3000,  // Error if no first value within 3 seconds
    each: 1000,   // Error if any subsequent value takes more than 1 second after the previous
    with: () => of(-1), // Instead of erroring, emit -1 as a fallback
  })
).subscribe(v => console.log(v));
```

---

## 15.8.4 — `finalize()` — Run Cleanup Code on Termination

`finalize()` registers a callback that runs when the source Observable terminates for ANY reason — completion, error, or unsubscription. It is the stream equivalent of `finally` in a try/catch/finally block. The finalize callback receives no arguments and its return value is ignored.

```typescript
import { finalize } from 'rxjs/operators';

// Classic loading state management — guaranteed to clear regardless of outcome
this.isLoading.set(true);
this.http.get<Data>('/api/data').pipe(
  finalize(() => this.isLoading.set(false)) // Runs whether success, error, or unsubscribe
).subscribe({
  next: data => this.data = data,
  error: err => this.handleError(err),
  // NOT here — because if you set isLoading = false in both next/error/complete,
  // you miss the unsubscription case (e.g., component destroyed before response arrives)
});

// Cleanup resources — close a database connection, release a lock, etc.
openDatabaseConnection().pipe(
  switchMap(connection => processData(connection)),
  finalize(() => closeConnection()) // Always close, even if an error occurred
).subscribe(result => displayResult(result));

// Track subscription lifetime for debugging
source$.pipe(
  tap(() => console.log('Subscription active — receiving values')),
  finalize(() => console.log('Subscription ended — resources released')),
).subscribe(v => console.log(v));
```

The key advantage of `finalize()` over putting cleanup code in `complete()` and `error()` callbacks separately is that it also handles unsubscription — when a component is destroyed mid-request, the `complete()` and `error()` callbacks are never called, but `finalize()` still runs. Always use `finalize()` for resource cleanup, never rely on `complete()` alone.

---

## 15.8.5 — `share()` and `shareReplay()` — Multicasting

These operators convert a cold Observable into a hot one that **multicasts** — sharing a single execution among all subscribers rather than creating a new execution for each subscriber. This is the solution to the "multiple subscriptions trigger multiple HTTP requests" problem.

### `share()` — Multicast, No Replay

`share()` turns a cold Observable into a hot multicast Observable. All current subscribers share the same source execution. Crucially, `share()` does NOT replay past values — new subscribers only receive future emissions.

`share()` also ref-counts — when the subscriber count drops to zero, the source is unsubscribed. When a new subscriber arrives, the source is re-subscribed from scratch.

```typescript
import { share, tap, interval } from 'rxjs';

// Without share: two subscriptions → two HTTP requests
const products$ = this.http.get<Product[]>('/api/products');
products$.subscribe(p => renderSidebar(p));
products$.subscribe(p => renderTable(p));
// TWO HTTP requests are sent!

// With share: two subscriptions → one HTTP request
const sharedProducts$ = this.http.get<Product[]>('/api/products').pipe(
  tap(() => console.log('HTTP request made')), // Only logged once
  share()
);
sharedProducts$.subscribe(p => renderSidebar(p));
sharedProducts$.subscribe(p => renderTable(p));
// ONE HTTP request; both subscribers receive the same response

// share() with timing: late subscribers miss past values
const shared$ = interval(1000).pipe(share());
shared$.subscribe(v => console.log('Sub A:', v)); // Gets 0, 1, 2, 3...

setTimeout(() => {
  shared$.subscribe(v => console.log('Sub B:', v)); // Joins at 3 → gets 3, 4, 5...
  // Sub B doesn't get 0, 1, 2 — those are gone
}, 3500);
```

### `shareReplay()` — Multicast with Replay Buffer

`shareReplay(config)` is like `share()` but with a replay buffer. New subscribers receive the specified number of most recently emitted values immediately upon subscribing. This makes late subscribers "catch up" to the current stream state.

```typescript
import { shareReplay } from 'rxjs/operators';

// The canonical pattern for sharing HTTP responses across multiple consumers
@Injectable({ providedIn: 'root' })
export class UserService {
  // shareReplay(1): share the request + replay the last response to late subscribers
  // This means: only ONE HTTP request ever goes out, and any component that subscribes
  // later (even after the response arrived) gets the cached response immediately
  readonly currentUser$ = this.http.get<User>('/api/me').pipe(
    shareReplay({ bufferSize: 1, refCount: true }),
    // refCount: true → unsubscribe from HTTP when all consumers unsubscribe
    // (prevents keeping a stale subscription alive forever)
  );
}

// In multiple components — all share the single HTTP request + cached response
// Component A
this.userService.currentUser$.subscribe(user => this.user = user);
// Component B — subscribes later, gets cached value immediately
this.userService.currentUser$.subscribe(user => this.displayName = user.name);
// Component C — subscribes much later, still gets the same cached value
this.userService.currentUser$.subscribe(user => this.permissions = user.roles);
```

**The critical difference between `share()` and `shareReplay()`:** `share()` is for "subscribe now and receive future emissions." `shareReplay(1)` is for "subscribe at any time and get the most recent value." Use `share()` for event-like streams (WebSockets, mouse events, real-time feeds). Use `shareReplay(1)` for request-like streams where late subscribers need the current state.

**`refCount: true` vs `refCount: false` in `shareReplay`:** With `refCount: true`, the source is unsubscribed when all consumers unsubscribe — if a new consumer subscribes later, the source is re-subscribed (a new HTTP request is made). With `refCount: false` (the old default behavior), the source stays active forever once started. For HTTP requests, `refCount: true` is almost always what you want — you want the response cached while it is being used, but you want a fresh request if the data becomes stale and new consumers arrive.

---

## 15.8.6 — `publish()` and `refCount()` — Lower-Level Multicasting

`publish()` and `refCount()` are the lower-level building blocks that `share()` is composed from. Understanding them gives insight into how multicasting works, but in practice, `share()` and `shareReplay()` are simpler and cover most needs.

`publish()` wraps the source in a `ConnectableObservable` — an Observable that only starts when you explicitly call `.connect()` on it. Until `.connect()` is called, no values flow regardless of how many subscribers there are. This gives you precise control over when multicasting begins.

`refCount()` adds automatic connection management — it calls `.connect()` automatically when the first subscriber arrives and calls `.unsubscribe()` when the last subscriber leaves.

```typescript
import { interval, publish, refCount } from 'rxjs';

const source$ = interval(1000).pipe(
  publish(),   // Creates a ConnectableObservable backed by a Subject
  refCount(),  // Auto-connect on first subscriber, auto-disconnect on last
);

// This combination is equivalent to share()
// In modern RxJS, just use share() instead
```

A practical use case for `publish()` directly is when you need to subscribe to multiple derived streams that all share the same source, while ensuring the source only runs once and all derivations start simultaneously:

```typescript
const hot$ = source$.pipe(publish());

// Set up multiple independent derived streams
const evens$ = hot$.pipe(filter(v => v % 2 === 0));
const odds$ = hot$.pipe(filter(v => v % 2 !== 0));
const total$ = hot$.pipe(scan((acc, v) => acc + v, 0));

evens$.subscribe(v => console.log('Even:', v));
odds$.subscribe(v => console.log('Odd:', v));
total$.subscribe(v => console.log('Total:', v));

// NOW start the source — all three subscriptions receive values from the same source
hot$.connect();
```

---

## 15.8.7 — `observeOn()` and `subscribeOn()` — Controlling Schedulers

These operators give explicit control over which scheduler (execution context) the Observable uses for delivering notifications and for subscribing.

`observeOn(scheduler)` controls the scheduler used for delivering `next`, `error`, and `complete` notifications to subscribers. It shifts where in the event loop the notification arrives — without changing when the source produces values.

`subscribeOn(scheduler)` controls where the subscription to the source Observable itself happens — useful in rare cases where you need the subscription to occur asynchronously.

```typescript
import { observeOn, subscribeOn, asyncScheduler, animationFrameScheduler } from 'rxjs';

// Run heavy computation on asyncScheduler (macrotask), then deliver UI update synchronously
heavyComputation$.pipe(
  observeOn(asyncScheduler) // Deliver updates in the next macrotask
).subscribe(result => this.result = result);

// Schedule DOM-related updates before the next animation frame for smooth rendering
dataStream$.pipe(
  observeOn(animationFrameScheduler) // Perfect for canvas drawing, CSS updates
).subscribe(data => drawToCanvas(data));
```

In practice, `observeOn` and `subscribeOn` are rarely needed in Angular applications because Zone.js (or Angular's signal scheduler in zoneless mode) handles the delivery of notifications into the correct execution context. They become relevant when building high-performance custom rendering pipelines, when working with Web Workers, or when writing library code that must control its own scheduling.

---

## Important Points and Best Practices

Always use `finalize()` for cleanup rather than the `complete()` callback alone. The `complete()` callback is only called on successful stream termination — it is not called when the subscriber unsubscribes (for example, when a component is destroyed mid-request). `finalize()` runs in all termination scenarios, making it the reliable choice for resource cleanup, hiding loading spinners, and releasing locks.

The `retry()` with exponential backoff pattern is the production-standard approach for HTTP resilience. Never use bare `retry(n)` without a delay in network code — hammering a failing server with immediate retries can make outages worse and get your application's IP rate-limited. Always add a timer-based delay that grows with each retry attempt.

`shareReplay({ bufferSize: 1, refCount: true })` is the standard caching pattern for shared HTTP requests in Angular services. Every long-lived HTTP Observable that multiple components might subscribe to should go through this combination. Without it, you will make redundant network requests as components mount and unmount.

`tap()` is for side effects, not for transformations. If you find yourself putting logic inside `tap()` that changes a value and then trying to use that changed value downstream, you are misusing it. Move the transformation into `map()`. Reserve `tap()` for logging, analytics, state updates, and other side effects where the return value is irrelevant. This separation makes pipelines more readable and testable because `tap()` is a visible signal: "there is a side effect here, but the data stream is unchanged."

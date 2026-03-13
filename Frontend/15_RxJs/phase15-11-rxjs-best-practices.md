# Phase 15.11 — RxJS Best Practices
## Naming Conventions, Preferring Operators, Flattening Selection Guide, Avoiding Nested Subscriptions, Memory Leak Prevention, Marble Testing

---

## Why Best Practices Matter in RxJS

RxJS is one of the most powerful libraries in the JavaScript ecosystem, and also one of the most misused. Its flexibility means there are often many ways to accomplish the same goal, but not all of them are equal in readability, safety, or testability. The best practices in this section are not arbitrary stylistic preferences — each one exists because a specific category of bug, performance problem, or maintainability issue arises when the practice is ignored. Internalizing these rules transforms RxJS code from something that is hard to review and debug into something that is elegant and declarative.

---

## 15.11.1 — Naming Observables with the `$` Suffix

The `$` suffix convention marks a variable as an Observable (or stream). This simple visual marker carries substantial information when reading code: it signals that the variable is not an immediately-readable value but a *description of a stream* — something lazy, potentially asynchronous, that needs to be subscribed to.

```typescript
// Without $ suffix — ambiguous types at a glance
const user = this.userService.getCurrentUser();       // Is this a User? A Promise? An Observable?
const products = this.store.select(selectProducts);   // Same ambiguity
const clickEvent = fromEvent(button, 'click');        // You have to read the right side

// With $ suffix — intent is clear at a glance
const user$ = this.userService.getCurrentUser();      // Observable<User>
const products$ = this.store.select(selectProducts);  // Observable<Product[]>
const click$ = fromEvent(button, 'click');            // Observable<Event>

// Makes code much easier to scan in templates and services
this.viewModel$ = combineLatest({
  user: this.user$,                    // Immediately obvious: streams feeding a combined stream
  products: this.products$,
  selectedId: this.selectedId$,
});
```

The convention extends naturally to Subjects (`private destroy$`, `private refresh$`) and to streams returned from methods. Some teams use `$$` for Subjects specifically, to distinguish the fact that a Subject is both subscribable AND directly callable with `.next()`, but the single `$` is the more widely adopted standard.

When you consistently apply the `$` suffix, code review becomes dramatically easier. A reviewer can immediately see which variables are streams (and therefore need subscription management attention) and which are plain values. The `$` suffix is especially powerful in Angular services where a class may mix Observable properties, signal properties, and plain data fields.

---

## 15.11.2 — Preferring Operators Over Manual Subscriptions

The biggest architectural mistake in RxJS codebases is subscribing to an Observable inside another subscription. This is the reactive equivalent of deeply nested callbacks — it produces the same "callback hell" problem that reactive programming was designed to solve. The root cause is thinking imperatively ("when this Observable gives me a value, I will then go and subscribe to another Observable for that value") rather than declaratively ("this stream depends on that stream").

The fix is always to use a flattening operator. Here is a concrete example of the anti-pattern and its correct form:

```typescript
// WRONG — nested subscriptions: manual, hard to cancel, leaks easily
this.route.paramMap.subscribe(params => {
  const id = params.get('id')!;
  this.productService.getProduct(id).subscribe(product => {
    this.product = product;
    this.categoryService.getCategory(product.categoryId).subscribe(category => {
      this.category = category;
    });
  });
});
// Problems: three separate subscriptions to manage, cancellation requires tracking
// three separate Subscription objects, errors in inner streams are not coordinated,
// and if the route param changes, old subscriptions may still be running.

// CORRECT — flat pipeline with operators
this.product$ = this.route.paramMap.pipe(
  map(params => params.get('id')!),
  switchMap(id => this.productService.getProduct(id)),
  shareReplay(1),
  takeUntilDestroyed()
);

this.category$ = this.product$.pipe(
  switchMap(product => this.categoryService.getCategory(product.categoryId))
);
// One pipeline per "data shape", one place to add error handling, one place to cancel.
// Cancellation is automatic on navigation via takeUntilDestroyed.
```

The general principle is that any time you find yourself writing `.subscribe()` inside a `.subscribe()` callback, you need a flattening operator. The correct operator depends on the concurrency semantics you need (see section 15.11.3 below).

Beyond nested subscriptions, prefer operators over manual imperative logic wherever possible. If you have a stream and you want to transform its values, use `map()` rather than subscribing, transforming, and pushing to a Subject. If you want to combine two streams, use `combineLatest` rather than subscribing to each and manually merging the results into shared state.

```typescript
// WRONG — subscribing just to pipe data into another Subject (unnecessary manual pipe)
private results$ = new Subject<SearchResult[]>();

this.query$.subscribe(query => {
  this.http.get<SearchResult[]>(`/api/search?q=${query}`).subscribe(results => {
    this.results$.next(results);
  });
});

// CORRECT — let operators do the composition
readonly results$ = this.query$.pipe(
  switchMap(query => this.http.get<SearchResult[]>(`/api/search?q=${query}`))
);
```

---

## 15.11.3 — The Flattening Strategy Selection Guide

Choosing the wrong flattening operator (`mergeMap` vs `concatMap` vs `switchMap` vs `exhaustMap`) is one of the most consequential mistakes in RxJS code. It often produces bugs that only appear under specific timing conditions (fast clicks, slow networks, concurrent users) making them hard to reproduce and diagnose. The following guide encodes the decision as a series of questions, building the mental muscle to make this choice automatically.

The decision starts with a single question: **"What should happen when a new source value arrives while a previous inner Observable is still running?"**

If the answer is **"the old one should be cancelled and the new one should start"**, the operator is `switchMap`. This is the right choice whenever the new input makes the in-progress work irrelevant. The canonical examples are search-as-you-type (a new keystroke makes the previous search obsolete), navigation-triggered data loading (navigating to a new route makes the previous route's data load irrelevant), and filter/sort changes that refresh a list (new filter criteria make the previous HTTP request's results stale). In all these cases, you only care about the result of the *most recent* input.

If the answer is **"the new one should wait until the old one finishes"**, the operator is `concatMap`. This is the right choice when order matters and operations must not overlap. Sequential data migrations, ordered API calls where each result feeds the next, and processing a queue of tasks where the order of completion must match the order of input all demand `concatMap`. The cost is that if the inner Observable is slow and the source is fast, a queue builds up — which may be acceptable or may indicate a design problem.

If the answer is **"both the old and new should run simultaneously"**, the operator is `mergeMap`. This is correct for operations that are fully independent and where throughput matters more than order. Parallel file uploads, broadcasting analytics events, and any "fire and forget" pattern with multiple concurrent operations suit `mergeMap`. Be aware that with an unbounded source, `mergeMap` can create unbounded concurrency — use the third argument (e.g., `mergeMap(fn, 3)`) to cap the number of simultaneous subscriptions.

If the answer is **"the new one should be silently dropped until the old one finishes"**, the operator is `exhaustMap`. This is correct when a single instance of the operation must run to completion before a new one can start, and triggering it again mid-operation is a user error to be ignored. Form submissions, login buttons, payment processing, and any "one at a time, no queuing" operation call for `exhaustMap`.

A helpful summary table for quick reference:

The question to ask yourself is: does the trigger relate to a read operation (searching, loading, filtering) or a write operation (saving, submitting, deleting)? Read operations almost always call for `switchMap` because you want the latest result, not all results. Write operations demand careful thought: `exhaustMap` if double-triggering should be ignored, `concatMap` if writes should be queued and preserved in order, and `mergeMap` only if writes are truly independent (like logging to an analytics endpoint where duplicates are acceptable).

---

## 15.11.4 — Avoiding Nested Subscriptions

This deserves its own section beyond the general "prefer operators" guidance because nested subscriptions are so common and so harmful. A nested subscription is created whenever you call `.subscribe()` inside a `.subscribe()` callback. Here are the specific harms this causes and the operator that fixes each case.

The first harm is resource leaks. Each inner subscription holds resources (event listeners, HTTP connections, timer handles). If the outer subscription is cancelled (component destroyed, user navigates away), the inner subscriptions continue running because nothing is tracking them. They will try to update the component's state even after the component no longer exists.

The second harm is broken cancellation logic. With nested subscriptions, there is no single unsubscription point. You would need to track every inner subscription object separately and call `.unsubscribe()` on each individually in `ngOnDestroy`. This is error-prone and easy to forget.

The third harm is incorrect ordering and race conditions. When the outer stream emits a new value, you get a new subscription to the inner stream — but the old inner subscription is still running. If your inner Observable is an HTTP request, you may now have two concurrent requests in flight, and whichever responds last will overwrite the first's result, even if the first response came from the more recent request. This is the classic "stale response" bug.

```typescript
// COMMON ANTI-PATTERN: form value changes that trigger HTTP calls
this.searchForm.get('query')!.valueChanges.subscribe(query => {
  // New subscription on every keystroke — old subscriptions never cancelled!
  this.http.get<Result[]>(`/api/search?q=${query}`).subscribe(results => {
    this.results = results; // Race condition: whichever request finishes last "wins"
  });
});

// CORRECT: One pipeline, cancellation handled by switchMap
this.results$ = this.searchForm.get('query')!.valueChanges.pipe(
  debounceTime(300),
  distinctUntilChanged(),
  // switchMap automatically unsubscribes from the previous HTTP call when a new value arrives
  switchMap(query => this.http.get<Result[]>(`/api/search?q=${query}`))
);
```

The practical rule is this: if you find yourself writing `someObservable$.subscribe(value => { anotherObservable$.subscribe(...) })`, stop immediately. Identify which flattening operator fits the concurrency semantics of the problem and restructure the code as a flat pipeline.

---

## 15.11.5 — Memory Leak Prevention Patterns

A memory leak in Angular RxJS code occurs when a subscription is not cleaned up when a component is destroyed. The subscription holds a reference to the subscriber's callbacks (which close over the component instance and its injected services), preventing the entire component tree from being garbage collected. Multiply this across many navigations in a long-running SPA and the application's memory footprint grows without bound.

There are five concrete patterns for preventing leaks, ordered from most preferred to least preferred.

The first and most modern approach is `toSignal()`. When you consume an Observable in a component's template or as a reactive value in the class, converting it with `toSignal()` from `@angular/core/rxjs-interop` gives you automatic subscription management through the injection context. No cleanup code required — the framework handles it. This is the preferred approach for Angular 16+ applications using the signals architecture.

The second approach is the `async` pipe. Any Observable used directly in a template with `| async` has its subscription managed entirely by Angular — subscribing on component creation and unsubscribing on destruction. This is zero-maintenance and is the preferred approach for template-bound streams when you are not using signals.

The third approach is `takeUntilDestroyed()` from `@angular/core/rxjs-interop`. For imperative subscriptions where you must call `.subscribe()` explicitly (because the subscription drives a side effect like analytics, DOM manipulation outside the template, or WebSocket message handling), pipe through `takeUntilDestroyed()`. It ties the subscription's lifetime to the component's `DestroyRef` automatically.

The fourth approach is the classic `takeUntil(this.destroy$)` pattern, which requires a `Subject<void>` field and a `ngOnDestroy` method that calls `this.destroy$.next()`. This is the pre-Angular 16 standard and still valid, but `takeUntilDestroyed()` is strictly cleaner for Angular 16+ projects.

The fifth approach, to be used as a last resort or for explicit control, is storing the `Subscription` object returned by `.subscribe()` and calling `.unsubscribe()` in `ngOnDestroy`. This is valid but verbose for multiple subscriptions, which is why the above patterns were developed to simplify it.

```typescript
// The five approaches compared — all prevent leaks, prefer from top to bottom

// 1. toSignal() — best for reactive class properties and template use
readonly user = toSignal(this.userService.user$, { initialValue: null });

// 2. async pipe — best for pure template consumption
// template: @if (orders$ | async; as orders) { ... }

// 3. takeUntilDestroyed() — best for imperative side effects (Angular 16+)
ngOnInit() {
  this.websocketService.messages$.pipe(
    takeUntilDestroyed(this.destroyRef)
  ).subscribe(msg => this.processMessage(msg));
}

// 4. takeUntil(destroy$) — valid for pre-16 or for non-injection contexts
private destroy$ = new Subject<void>();
ngOnInit() {
  interval(1000).pipe(takeUntil(this.destroy$)).subscribe(/* ... */);
}
ngOnDestroy() { this.destroy$.next(); this.destroy$.complete(); }

// 5. Manual Subscription — last resort
private sub: Subscription = Subscription.EMPTY;
ngOnInit() { this.sub = observable$.subscribe(/* ... */); }
ngOnDestroy() { this.sub.unsubscribe(); }
```

There is also a category of "soft leaks" — streams that are technically cleaned up but that have unnecessary complexity because they subscribe before they need to. A common example is subscribing to route parameters in `ngOnInit` when the component does not actually need the data until a button is clicked. Prefer lazy subscription patterns (like `fromEvent` on a button) over eagerly subscribing in `ngOnInit` when the subscription is not immediately needed.

---

## 15.11.6 — Testing RxJS with Marble Testing and TestScheduler

RxJS operators are time-dependent — their behavior changes based on when values arrive relative to each other. This makes them difficult to test with conventional async testing approaches (actual `setTimeout` delays, `fakeAsync` with manually-advanced time). The solution is RxJS's built-in `TestScheduler`, which provides a "virtual time" environment where time is a number you control, and marble diagrams are the language for describing expected behavior.

### Setting Up TestScheduler

```typescript
import { TestScheduler } from 'rxjs/testing';

describe('debounceTime + switchMap pipeline', () => {
  let testScheduler: TestScheduler;

  beforeEach(() => {
    // The comparator function receives actual vs expected marble frame data
    // and should throw if they do not match — here using Jasmine/Jest expect
    testScheduler = new TestScheduler((actual, expected) => {
      expect(actual).toEqual(expected);
    });
  });

  it('should debounce a fast-typing search and cancel old requests', () => {
    testScheduler.run(({ cold, hot, expectObservable }) => {
      // hot() creates an Observable that has already started by the time test begins
      // The '^' marker in a hot marble denotes the subscription point
      const queryInput$ = hot('--a-b-----c---d|', {
        a: 'a',
        b: 'ab',
        c: 'abc',
        d: 'abcd',
      });

      // cold() creates an Observable that starts fresh for each subscription
      // We use a factory to produce different result values for each query
      const fetchResults = (query: string) =>
        cold('---r|', { r: `results for ${query}` });

      // Apply our real production operators
      const result$ = queryInput$.pipe(
        debounceTime(3), // 3 frames in virtual time (represents 300ms in real time)
        switchMap(query => fetchResults(query)),
      );

      // Assert the expected behavior with a marble string
      // 'b' arrives at frame 4 (after 3-frame debounce from frame 1)
      // 'c' arrives at frame 9 (after 3-frame debounce from frame 6)
      // 'd' arrives at frame 12 (too close to c — c is debounced away)
      // 'd' debounce fires at frame 15, result arrives 3 frames later
      expectObservable(result$).toBe('--------r-------r|', {
        r: 'results for abc', // 'c' survived the debounce
        // Wait... only one 'r' for abc and one 'r' for abcd in the actual calculation
      });
    });
  });
});
```

### Writing Effective Marble Tests for Angular Services

The marble testing approach shines when testing RxJS-heavy Angular services. By injecting mock HTTP responses as cold Observables with marble strings, you can test timing-dependent behavior deterministically without any real time passing.

```typescript
import { TestScheduler } from 'rxjs/testing';
import { TestBed } from '@angular/core/testing';
import { provideHttpClientTesting, HttpTestingController } from '@angular/common/http/testing';

describe('SearchService', () => {
  let testScheduler: TestScheduler;
  let service: SearchService;

  beforeEach(() => {
    testScheduler = new TestScheduler((actual, expected) => {
      expect(actual).toEqual(expected);
    });
    TestBed.configureTestingModule({ providers: [SearchService, provideHttpClientTesting()] });
    service = TestBed.inject(SearchService);
  });

  it('should only emit the result of the LAST query when typed quickly', () => {
    testScheduler.run(({ cold, hot, expectObservable }) => {
      // Simulate fast typing: queries arrive at frames 1, 3, 5
      const queries$ = hot('  -a-b-c|', { a: 'he', b: 'hel', c: 'hell' });

      // Each HTTP response takes 10 frames, but the debounce is 5 frames
      // So 'he' and 'hel' are debounced away; only 'hell' makes it through
      spyOn(service, 'fetchResults').and.callFake((query: string) =>
        cold('----------r|', { r: { query, results: [`result for ${query}`] } })
      );

      const result$ = service.search$(queries$);

      // Only 'hell' makes it through after the 5-frame debounce
      // That request completes 10 frames later → frame 5 + 5(debounce) + 10(http) = 20
      expectObservable(result$).toBe('--------------------r|', {
        r: { query: 'hell', results: ['result for hell'] }
      });
    });
  });
});
```

### Using `cold()` and `hot()` Effectively

The key to writing good marble tests is understanding when to use `cold()` versus `hot()`. Use `cold()` for any Observable that represents an operation that starts fresh when subscribed to — HTTP requests, computed derivations, timers created for the test. Use `hot()` for any Observable that represents an ongoing stream that exists before the subscription point — user input streams, router events, Subject-based event buses.

The `^` character in a `hot()` marble string marks the subscription point — the moment when your test Subject subscribes to the stream. Values before `^` are emitted "in the past" before subscription (and are missed by the subscriber, since hot Observables do not replay). Values after `^` are emitted during the subscription.

```typescript
// hot() — values before ^ are emitted before subscription (missed)
const earlyEvents$ = hot('---a^---b---c|');
// Subscriber sees: ---b---c| (misses 'a' which happened before ^)

// cold() — all values start at the subscription point
const freshRequest$ = cold('---r|', { r: response });
// Every subscriber gets: ---r|  starting from their subscription moment
```

### Testing Error Scenarios

Marble testing makes error scenarios easy to test — the `#` character represents an error in the stream:

```typescript
it('should retry twice and then error', () => {
  testScheduler.run(({ cold, expectObservable }) => {
    // The source errors on its first and second subscriptions, succeeds on the third
    let subscriptionCount = 0;
    const intermittentSource$ = cold('#', undefined, new Error('Network error'));
    const eventuallySucceeds$ = cold('--r|', { r: 'data' });

    // Test that retry(2) retries twice then errors
    const failingSource$ = cold('#').pipe(retry(2));

    // After 2 retries (each is immediate since no delay), errors on the 3rd attempt
    // --# means error arrives 2 frames in (1 attempt = 1 frame, 3 attempts = 3 frames)
    expectObservable(failingSource$).toBe('---#', undefined, new Error('Network error'));
  });
});
```

---

## A Final Summary: The RxJS Mindset

Mastering RxJS is ultimately about making a mental shift — from "what do I do with this value?" (imperative) to "what is the relationship between these streams?" (declarative). Concrete rules to carry forward are outlined here.

Think in pipelines, not subscriptions. Every transformation, combination, and side effect should live inside a `.pipe()` chain. The number of explicit `.subscribe()` calls in your codebase should be minimal — ideally zero in components using `async` pipe and `toSignal()`.

Design Observables at the service level, not the component level. An Observable that represents "the current user" or "the list of products" belongs in a service. Components consume these streams but do not create them. This creates a clean separation between data orchestration (services) and data display (components).

Let the operator type communicate intent. When you use `switchMap`, you are communicating "only the latest matters, cancel old ones." When you use `concatMap`, you are communicating "order matters, serialize these operations." This semantic communication is built into the code itself — future maintainers see the operator and understand the design decision.

Never let a subscription outlive the context that created it. Every long-lived subscription (anything that does not emit and complete immediately) must be managed through `toSignal()`, `async` pipe, `takeUntilDestroyed()`, `takeUntil(destroy$)`, or explicit `Subscription.unsubscribe()` in `ngOnDestroy`. Leaving this to chance is leaving memory leaks to chance.

Write marble tests for timing-dependent logic. If your code uses `debounceTime`, `switchMap`, `retry`, `throttleTime`, or any other time-sensitive operator, the behavior under specific timing conditions is part of the specification and should be tested. Marble tests make this specification exact, machine-verifiable, and immune to flakiness from real time.

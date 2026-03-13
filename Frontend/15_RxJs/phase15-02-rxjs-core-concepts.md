# Phase 15.2 — RxJS Core Concepts
## Cold vs Hot Observables, Observers, Subscriptions, Subjects, Schedulers, and Marble Diagrams

---

## 15.2.1 — Cold vs Hot Observables

This is one of the most confusing concepts in RxJS, and also one of the most important. Getting it wrong leads to subtle bugs that are extremely hard to track down. The "temperature" of an Observable describes whether the *data producer* is created inside or outside the Observable.

### Cold Observables — Each Subscriber Gets Their Own Stream

A cold Observable creates its data producer *inside* the Observable's subscriber function. This means every new subscriber gets a completely independent execution of the producer. The data starts fresh for each subscriber, and subscribers receive values from the very beginning of the stream.

Think of a cold Observable like a movie on Netflix. Every viewer who presses play starts from the beginning and gets their own private copy of the movie playing for them. Two viewers watching the same movie are not watching the same thing at the same time — each has their own independent playback.

```typescript
import { Observable, interval } from 'rxjs';
import { take } from 'rxjs/operators';

// This Observable is COLD because it creates a new interval timer
// inside the subscriber function for each subscriber
const coldTimer$ = new Observable<number>(subscriber => {
  let count = 0;
  // This timer is created fresh for EACH subscriber
  const intervalId = setInterval(() => {
    subscriber.next(count++);
  }, 1000);

  // Teardown clears the timer when unsubscribed
  return () => clearInterval(intervalId);
});

// Subscribe first time
const sub1 = coldTimer$.subscribe(v => console.log(`Sub 1: ${v}`));

// Subscribe 3 seconds later
setTimeout(() => {
  const sub2 = coldTimer$.subscribe(v => console.log(`Sub 2: ${v}`));
  // Sub 1 and Sub 2 are now running their OWN independent timers
  // Sub 1 is at ~3, Sub 2 starts at 0 — they are completely independent
}, 3000);

// Most RxJS creation functions produce cold Observables by default
// HttpClient requests are cold — each subscription triggers a new HTTP call
const users$ = this.http.get<User[]>('/api/users');
const sub1 = users$.subscribe(u => /* display in sidebar */);
const sub2 = users$.subscribe(u => /* display in table */);
// This sends TWO HTTP requests! Both subscriptions get their own execution.
```

The implication for Angular is critical: if you subscribe to an `HttpClient` observable in multiple places, you make multiple HTTP requests. This is the behavior that `shareReplay()` is designed to prevent.

### Hot Observables — All Subscribers Share One Stream

A hot Observable's data producer exists *outside* the Observable. The producer is already running before anyone subscribes, and subscribers tune in to receive values that are happening right now. New subscribers do not receive past values — they only get values emitted after they subscribed.

Think of a hot Observable like a live radio broadcast. If you tune in at 3pm, you hear what is playing at 3pm. The music that played at 2pm is gone — you cannot rewind. All listeners at 3pm hear the same song at the same moment.

```typescript
import { fromEvent } from 'rxjs';

// Mouse clicks are HOT — the click events exist independently
// of whether anyone is subscribed. When you subscribe, you catch
// clicks happening from that moment forward — not past clicks.
const clicks$ = fromEvent(document, 'click');

// Sub 1 subscribes now and catches all future clicks
const sub1 = clicks$.subscribe(e => console.log('Sub 1 caught click'));

setTimeout(() => {
  // Sub 2 subscribes 2 seconds later — both sub1 and sub2 will
  // receive the SAME click events from now on (shared producer)
  // But sub2 missed any clicks that happened in those 2 seconds
  const sub2 = clicks$.subscribe(e => console.log('Sub 2 caught click'));
}, 2000);
```

### The Temperature Chart

Understanding where different RxJS sources fall on the cold/hot spectrum helps you predict behavior and avoid bugs.

**Cold sources** include `of()`, `from()`, `interval()`, `timer()`, `ajax()`, `HttpClient` observables, and any Observable you create with `new Observable(subscriber => {...})`. All of these create a fresh execution for each subscriber.

**Hot sources** include `fromEvent()` (the DOM events exist independently), `Subject` (covered below — it is explicitly hot), `BehaviorSubject`, and Observables that have been converted to hot via `share()` or `shareReplay()`.

---

## 15.2.2 — The Observer: next, error, complete

An Observer is the consumer side of the Observable/Observer contract. It is an object with three optional callbacks that together define what to do with the stream's output at every possible moment in its lifecycle.

### The `next` Callback

`next` is called once for each value emitted by the Observable. It is the main handler — the one that processes the actual data. It can be called zero times (if the stream emits nothing before completing), once (like an HTTP request), or many times (like a timer or WebSocket stream). After `complete` or `error` is called, `next` is never called again.

### The `error` Callback

`error` is called at most once, when the stream encounters an unrecoverable error. It receives the error object as its argument. After `error` is called, the stream is considered terminated — no more `next` or `complete` callbacks will fire, and the subscription's teardown function runs. If you do not provide an `error` callback and the stream errors, RxJS will throw the error to the global error handler.

### The `complete` Callback

`complete` is called at most once, when the stream finishes successfully with no more values to emit. It receives no arguments — its only role is to signal the end. After `complete`, no more `next` or `error` callbacks fire, and the subscription's teardown function runs.

```typescript
import { Observable } from 'rxjs';

const demoStream$ = new Observable<number>(subscriber => {
  subscriber.next(1);
  subscriber.next(2);
  subscriber.next(3);
  subscriber.complete(); // Signal: no more values, all is well
  // This next() call is IGNORED — complete() was already called
  subscriber.next(4);
});

demoStream$.subscribe({
  next: value => console.log(`Value: ${value}`),      // 1, 2, 3 (not 4)
  error: err => console.error(`Error: ${err}`),        // never called here
  complete: () => console.log('Done!'),                // called once after 3
});
// Output: Value: 1, Value: 2, Value: 3, Done!
```

### Partial Observers and Shorthand

You are not required to provide all three callbacks. RxJS uses default no-op handlers for any callbacks you omit. You can also pass a single function as the `next` handler as shorthand:

```typescript
// Only handle next values, ignore errors and completion
stream$.subscribe(value => console.log(value));

// Handle next and errors but not completion
stream$.subscribe({
  next: value => console.log(value),
  error: err => console.error(err),
});
```

---

## 15.2.3 — Subscription and Unsubscribing

A Subscription represents an active connection between an Observable and an Observer. It is the object that holds the resources being used to produce values — timers, event listeners, HTTP requests, WebSocket connections. When you unsubscribe, all of those resources are released.

```typescript
import { interval } from 'rxjs';

// Start the subscription — resources are allocated
const subscription = interval(1000).subscribe(v => console.log(v));

// The subscription is active — values are flowing
// 0, 1, 2, 3...

// Release all resources — timer is cleared, no more values
subscription.unsubscribe();

// Check the status
console.log(subscription.closed); // true — unsubscribed
```

### Composing Subscriptions with `add()`

When you have multiple subscriptions that should all be cancelled together, you can group them by adding child subscriptions to a parent:

```typescript
import { interval, fromEvent } from 'rxjs';

const parent = interval(1000).subscribe(v => console.log('timer', v));
const child1 = fromEvent(document, 'click').subscribe(() => console.log('click'));
const child2 = fromEvent(document, 'keyup').subscribe(() => console.log('keyup'));

// Add child subscriptions to the parent
parent.add(child1);
parent.add(child2);

// Unsubscribing the parent automatically unsubscribes all children
parent.unsubscribe(); // Cancels timer, click listener, AND keyup listener
```

This pattern is useful in Angular components when you want a single `ngOnDestroy` cleanup call to cancel all subscriptions. However, in modern Angular, `takeUntilDestroyed()` (covered in section 15.9) is the preferred approach.

---

## 15.2.4 — Subjects: The Bridge Between Imperative and Reactive

A Subject is simultaneously an Observable and an Observer. This dual nature makes it the bridge between the imperative world (where your code explicitly calls a method to emit a value) and the reactive world (where code subscribes to observe values).

As an Observable, a Subject has `.subscribe()`. As an Observer, it has `.next()`, `.error()`, and `.complete()`. When you call `.next(value)` on a Subject, every current subscriber receives that value. This makes Subjects inherently hot — the values flow to whoever is subscribed at the moment `.next()` is called.

### `Subject` — The Basic Multicaster

A plain `Subject` has no memory. It emits values to current subscribers and forgets them. New subscribers receive nothing from before their subscription.

```typescript
import { Subject } from 'rxjs';

const subject$ = new Subject<string>();

// Subscribe BEFORE any values are emitted
subject$.subscribe(v => console.log('Sub 1:', v));

subject$.next('first');  // Sub 1: first
subject$.next('second'); // Sub 1: second

// Subscribe AFTER some values were emitted
subject$.subscribe(v => console.log('Sub 2:', v));
// Sub 2 gets nothing for 'first' or 'second' — they are gone

subject$.next('third');  // Sub 1: third, Sub 2: third (both receive)

subject$.complete();
// All subscribers receive complete notification
```

Subjects are most useful as event buses and as the mechanism inside Angular services that bridge imperative code (something happens, call `.next()`) with reactive consumers (components that subscribe and render based on the values).

### `BehaviorSubject` — "Current Value" Semantics

`BehaviorSubject` remembers the most recently emitted value and immediately delivers it to any new subscriber. It requires an initial value, which represents the state before any explicit `.next()` calls.

This makes `BehaviorSubject` ideal for representing **state** — a value that always has some current version that new subscribers should immediately know about.

```typescript
import { BehaviorSubject } from 'rxjs';

// Initial state: user is not logged in
const isLoggedIn$ = new BehaviorSubject<boolean>(false);

// Component A subscribes immediately
isLoggedIn$.subscribe(v => console.log('Component A:', v)); // Immediately: false

// User logs in — emit new value
isLoggedIn$.next(true);
// Component A receives: true

// Component B loads later (after login)
isLoggedIn$.subscribe(v => console.log('Component B:', v));
// Component B IMMEDIATELY receives: true (the current value)
// It does not have to wait for the next emission

// Synchronous access to current value — useful in imperative code
console.log(isLoggedIn$.getValue()); // true
```

The critical property of `BehaviorSubject` is that it always has a value. `.getValue()` always returns the most recently emitted value. This is why it is perfect for representing state in Angular services — any component that subscribes always gets the current state immediately, without waiting for the next update.

### `ReplaySubject` — "N Last Values" Semantics

`ReplaySubject` records the last N emissions and replays them to any new subscriber. A `ReplaySubject(1)` behaves like `BehaviorSubject` but without requiring an initial value — it emits nothing to new subscribers until at least one value has been emitted.

```typescript
import { ReplaySubject } from 'rxjs';

// Remember the last 3 emissions
const events$ = new ReplaySubject<string>(3);

events$.next('event-1');
events$.next('event-2');
events$.next('event-3');
events$.next('event-4'); // Now only 2, 3, 4 are remembered (buffer is 3)

// Late subscriber receives the last 3 values immediately upon subscribing
events$.subscribe(v => console.log(v));
// Output (immediately): event-2, event-3, event-4

// ReplaySubject(1) is often preferred over BehaviorSubject when
// there is no meaningful "initial" value before the first emission
const currentUser$ = new ReplaySubject<User>(1);
// Before any .next() call: new subscribers wait (no initial value)
// After first .next(user): new subscribers immediately get the current user
```

### `AsyncSubject` — Only the Last Value, Only on Complete

`AsyncSubject` is the most specialized Subject. It holds the last emitted value and only delivers it when `complete()` is called. It is useful when you want to represent a computation whose final result you care about, but intermediate values are irrelevant.

```typescript
import { AsyncSubject } from 'rxjs';

const result$ = new AsyncSubject<number>();

result$.subscribe(v => console.log('Subscriber 1:', v));

result$.next(1); // Not emitted yet
result$.next(2); // Not emitted yet
result$.next(3); // Not emitted yet

result$.subscribe(v => console.log('Subscriber 2:', v));

result$.complete();
// NOW both subscribers receive: 3 (the last value)
// Subscriber 1: 3
// Subscriber 2: 3
```

`AsyncSubject` is rarely used directly, but it is how `lastValueFrom()` is implemented internally.

---

## 15.2.5 — Schedulers: Controlling When Work Happens (Conceptual)

A Scheduler in RxJS is an abstraction that controls *when* and *in which execution context* a piece of work runs. Most developers rarely interact with Schedulers directly, but understanding them conceptually helps you understand why some Observables are synchronous and others are asynchronous, and how to control the order of execution in testing.

RxJS provides several built-in Schedulers. `asyncScheduler` schedules work asynchronously using `setTimeout(fn, 0)` — placing it in the macrotask queue. `queueScheduler` schedules work synchronously within the current event loop tick, using a queue to prevent stack overflow for recursive operations. `animationFrameScheduler` schedules work before the next browser repaint — ideal for smooth animations. `asapScheduler` schedules work as a microtask, executing before macrotasks but after the current synchronous code.

```typescript
import { of, asyncScheduler, asapScheduler } from 'rxjs';
import { observeOn } from 'rxjs/operators';

// By default, of() is synchronous
of(1, 2, 3).subscribe(v => console.log('sync:', v));
// Output immediately: 1, 2, 3

// observeOn() makes notifications happen on the specified scheduler
of(1, 2, 3).pipe(
  observeOn(asyncScheduler)
).subscribe(v => console.log('async:', v));
// Output in next macrotask (after setTimeout delay)

console.log('after subscriptions');
// Output order: 1, 2, 3 (sync), then 'after subscriptions', then 1, 2, 3 (async)
```

The most important practical use of schedulers is in **testing** — specifically with the `TestScheduler` that lets you control virtual time in marble tests, making time-dependent Observables testable without waiting for real time to pass. This is covered in detail in section 15.11.

---

## 15.2.6 — Marble Diagrams: Reading and Writing

Marble diagrams are the visual language of RxJS. They represent the behavior of Observables and operators in a time-based notation that is easier to read than code alone. Once you can read and write marble diagrams fluently, understanding and communicating about any RxJS operator becomes much easier.

### The Anatomy of a Marble Diagram

A marble diagram consists of a timeline (the horizontal dashes and the arrow), marbles (the values emitted), and terminal symbols. Here is the full notation:

```
----- represents time passing with no emission
a     represents a value "a" being emitted at this moment
|     represents the stream completing successfully (no more values)
#     represents the stream erroring (terminates the stream)
^     represents the moment a subscriber subscribes (starting point)
!     represents the moment an unsubscription occurs

Example timeline:
--a--b--c--|
  ↑  ↑  ↑  ↑
  |  |  |  └── Stream completes
  |  |  └── Value 'c' emitted
  |  └── Value 'b' emitted
  └── Value 'a' emitted after two time units
```

Each character represents one frame of virtual time. The spacing matters — it represents the relative timing of emissions.

### Reading Common Operator Diagrams

The best way to build marble diagram fluency is to trace through operator diagrams step by step. Let's look at several key operators:

**`map(x => x * 10)`:**
```
Source:    --1--2--3--|
Result:    --10-20-30-|

Reading: Every value from the source is multiplied by 10
and emitted at the exact same time as the source value.
The completion passes through unchanged.
```

**`filter(x => x % 2 === 0)`:**
```
Source:    --1--2--3--4--|
Result:    -----2-----4--|

Reading: Odd values (1, 3) are dropped. Even values (2, 4)
pass through at the same time. Completion passes through.
```

**`take(2)`:**
```
Source:    --a--b--c--d--|
Result:    --a--b|

Reading: After 2 values pass through, the stream completes
immediately — even though the source had more values.
```

**`switchMap(x => interval(10).pipe(take(3)))`:**
```
Source:          --a--------b---------|
Inner for 'a':   --0--1--2| (created when a arrives)
Inner for 'b':              --0--1--2| (created when b arrives; previous inner cancelled)
Result:          --0--1--2----0--1--2|

Reading: When 'a' arrives, start an inner stream. When 'b' arrives,
cancel the inner stream for 'a' and start a new inner stream for 'b'.
Emit values from whichever inner stream is currently active.
```

### Writing Marble Strings for Testing

RxJS provides a `TestScheduler` that interprets marble strings programmatically. You use this in tests to verify that operators and Observables behave as expected without waiting for real time:

```typescript
import { TestScheduler } from 'rxjs/testing';
import { map, filter } from 'rxjs/operators';

describe('map and filter', () => {
  let testScheduler: TestScheduler;

  beforeEach(() => {
    testScheduler = new TestScheduler((actual, expected) => {
      expect(actual).toEqual(expected); // Your test assertion library
    });
  });

  it('should double even numbers only', () => {
    testScheduler.run(({ cold, expectObservable }) => {
      // Define the source stream using marble notation
      const source$ = cold('--1--2--3--4--|', {
        '1': 1, '2': 2, '3': 3, '4': 4  // Map marble chars to actual values
      });

      const result$ = source$.pipe(
        filter(x => x % 2 === 0),
        map(x => x * 2),
      );

      // Assert the result matches this marble string
      expectObservable(result$).toBe('-----4-----8--|', {
        '4': 4, '8': 8
      });
    });
  });
});
```

The `cold()` helper creates a cold Observable from a marble string. `hot()` creates a hot Observable (useful for testing Subjects and shared streams). `expectObservable()` asserts that an Observable produces the values described by the marble string.

### The Value Mapping Object

When your emitted values are not single characters, you use a second argument to `cold()`, `hot()`, and `toBe()` — a mapping object that translates the single-character marble keys to actual TypeScript values:

```typescript
const source$ = cold('--a--b--c--|', {
  a: { id: 1, name: 'Alice' },
  b: { id: 2, name: 'Bob' },
  c: { id: 3, name: 'Carol' },
});
// The characters 'a', 'b', 'c' in the marble string represent
// the corresponding objects in the mapping
```

---

## Important Points and Best Practices

The cold/hot distinction is not just academic — it has direct, practical consequences. The most important consequence in Angular is that `HttpClient` observables are cold. If you have two components or two parts of a template that both subscribe to the same HTTP observable (via `async` pipe or direct subscriptions), you will make two HTTP requests. Use `shareReplay(1)` or Angular's `httpResource()` when you need to share a single HTTP response across multiple consumers.

`BehaviorSubject` vs `ReplaySubject(1)` is a genuine design choice. Use `BehaviorSubject` when your state always has a meaningful initial value (for example, "user is not logged in" is a meaningful initial state for authentication). Use `ReplaySubject(1)` when there is no meaningful initial value — the state only exists after the first real emission. Using `BehaviorSubject` with `null` as an initial value and then constantly checking for null is often a code smell suggesting `ReplaySubject(1)` would be more appropriate.

Never call `.next()` on a Subject from inside a subscriber of that same Subject. This creates a synchronous recursive cycle that can cause infinite loops or missed emissions depending on the scheduler. If you need to derive a new stream from a Subject and feed values back in, use operators in a pipeline rather than calling `.next()` imperatively inside a subscription.

Learn to read marble diagrams before you try to write them. The RxJS documentation at rxjs.dev includes marble diagrams for every operator. When you encounter an unfamiliar operator, the first thing to do is look at its marble diagram — it conveys the operator's behavior more clearly than a paragraph of prose in almost every case. After reading dozens of them, writing them becomes natural.

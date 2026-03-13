# Phase 15.5 — Filtering Operators
## filter, take, takeUntil, takeWhile, takeLast, skip, first, last, distinctUntilChanged, debounceTime, throttleTime, auditTime, sampleTime

---

## What Are Filtering Operators?

Filtering operators decide which values from a source Observable are allowed to pass through to the next stage of the pipeline. They never transform values — a value either passes through unchanged or is dropped. But "filtering" in RxJS is a broader concept than it sounds: some operators filter based on value content, some filter based on position in the sequence, some filter based on timing, and some filter based on whether the value changed from the last emission. Together they give you precise control over the shape and rate of your stream.

---

## 15.5.1 — `filter()` — Pass Values That Satisfy a Predicate

`filter()` is the most direct filtering operator. It evaluates a predicate function for each emitted value and only lets values through where the predicate returns `true`. It is the stream equivalent of `Array.prototype.filter()`.

```typescript
import { from, fromEvent, filter } from 'rxjs';
import { map } from 'rxjs/operators';

// Simple numeric filter
from([1, 2, 3, 4, 5, 6]).pipe(
  filter(n => n % 2 === 0)
).subscribe(v => console.log(v)); // 2, 4, 6

// Filter objects by property
this.http.get<Order[]>('/api/orders').pipe(
  // Assuming the response is a single array, map it to individual emissions
  mergeMap(orders => from(orders)),
  filter(order => order.status === 'pending' && order.total > 100)
).subscribe(order => processHighValuePending(order));

// Filter keyboard events by key
fromEvent<KeyboardEvent>(document, 'keydown').pipe(
  filter(event => event.key === 'Enter' && !event.shiftKey)
).subscribe(event => {
  event.preventDefault();
  submitForm();
});

// TypeScript type narrowing with filter
// filter() can narrow types using a type predicate function
function isNotNull<T>(value: T | null): value is T {
  return value !== null;
}

this.store.select(selectSelectedProduct).pipe(
  filter(isNotNull) // TypeScript now knows the stream emits Product, not Product | null
).subscribe(product => {
  // product is typed as Product here, not Product | null
  console.log(product.name);
});
```

The type predicate form of `filter()` is particularly useful in Angular with NgRx selectors that may return `null`. By applying `filter(isNotNull)`, you narrow the type for all downstream operators, avoiding `!` non-null assertions or `?.` optional chaining in every subsequent operator.

---

## 15.5.2 — `take()`, `takeUntil()`, `takeWhile()`, `takeLast()` — Limit How Many Values You Receive

These four operators control the "how long" dimension of a stream — when should the subscription end based on value count, an external signal, a condition, or position from the end.

### `take(count)` — First N Values

`take(n)` emits the first `n` values from the source and then completes the stream. After `n` values, the source is unsubscribed from, releasing any resources.

```typescript
import { interval, take } from 'rxjs';

// Only take the first 5 values from an infinite timer
interval(1000).pipe(take(5))
  .subscribe({
    next: v => console.log(v),      // 0, 1, 2, 3, 4
    complete: () => console.log('done'), // fires after 4
  });

// Take the first emission from a stream — very common pattern
this.router.events.pipe(
  filter(e => e instanceof NavigationEnd),
  take(1) // Only care about the FIRST navigation completing
).subscribe(() => {
  // Run some initialization that should happen once, after first navigation
  this.initializeAnalytics();
});
```

### `takeUntil(notifier$)` — Take Until Another Observable Emits

`takeUntil(notifier$)` keeps the source active until the `notifier$` Observable emits any value, at which point the source is completed and unsubscribed. This is the most important mechanism for managing subscription lifetime in Angular — connecting a stream's life to a component's life.

```typescript
import { takeUntil, Subject, interval, fromEvent } from 'rxjs';

// The classic Angular component unsubscription pattern
@Component({ /* ... */ })
export class DataComponent implements OnInit, OnDestroy {
  private destroy$ = new Subject<void>();

  ngOnInit() {
    // This subscription stays alive until destroy$ emits
    interval(5000).pipe(
      switchMap(() => this.http.get('/api/notifications')),
      takeUntil(this.destroy$) // Auto-unsubscribes when component is destroyed
    ).subscribe(notifications => this.notifications = notifications);

    // You can reuse the same destroy$ for multiple subscriptions
    fromEvent(window, 'resize').pipe(
      takeUntil(this.destroy$)
    ).subscribe(() => this.recalculateLayout());
  }

  ngOnDestroy() {
    this.destroy$.next(); // Signal: all subscriptions using destroy$ should stop
    this.destroy$.complete(); // Clean up the Subject itself
  }
}
```

Note: In Angular 16+, `takeUntilDestroyed()` from `@angular/core/rxjs-interop` replaces this pattern more elegantly (covered in section 15.9). But `takeUntil` remains important for understanding the underlying mechanism and for non-Angular contexts.

### `takeWhile(predicate, inclusive?)` — Take While Condition Is True

`takeWhile(predicate)` emits values while the predicate returns `true` and stops (completing the stream) when the predicate first returns `false`. The value that "breaks" the condition is, by default, NOT emitted. Passing `true` as the second argument (inclusive mode) emits that breaking value and then completes.

```typescript
import { fromEvent, takeWhile, map } from 'rxjs';

// Take mouse position updates while mouse stays in the left half of the screen
fromEvent<MouseEvent>(document, 'mousemove').pipe(
  map(e => e.clientX),
  takeWhile(x => x < window.innerWidth / 2) // Stop when cursor moves to right half
).subscribe(x => console.log('x:', x));

// Count down from 10, stopping at 0 (inclusive — includes 0 in output)
import { interval } from 'rxjs';
interval(1000).pipe(
  map(v => 10 - v),
  takeWhile(v => v >= 0, true) // inclusive: emits the 0 that breaks the condition
).subscribe({
  next: v => console.log(v),      // 10, 9, 8...1, 0
  complete: () => console.log('Countdown finished!')
});

// Stop reading user input when they submit a valid form
formValues$.pipe(
  takeWhile(value => !isValidSubmission(value), true) // Stop + include the valid submission
).subscribe(value => {
  if (isValidSubmission(value)) {
    submitForm(value);
  } else {
    updateValidationUI(value);
  }
});
```

### `takeLast(count)` — Last N Values After Completion

`takeLast(n)` waits for the source to complete and then emits only the last `n` values. It requires the source to complete — it cannot work with infinite streams.

```typescript
import { from, takeLast } from 'rxjs';

from([1, 2, 3, 4, 5]).pipe(
  takeLast(2)
).subscribe(v => console.log(v)); // 4, 5

// Get the last 3 audit log entries
this.http.get<AuditEntry[]>('/api/audit-log').pipe(
  mergeMap(entries => from(entries)),
  takeLast(3)
).subscribe(entry => console.log('Recent:', entry));
```

---

## 15.5.3 — `skip()`, `skipUntil()`, `skipWhile()` — Ignore the Beginning

The skip operators are the mirror image of take operators — they ignore a certain number or range of values and then let everything through.

### `skip(count)` — Skip the First N Values

```typescript
import { from, skip } from 'rxjs';

// Skip the first 2 values
from([1, 2, 3, 4, 5]).pipe(
  skip(2)
).subscribe(v => console.log(v)); // 3, 4, 5

// Skip the initial emission from BehaviorSubject (the synthetic "initial" value)
// and only react to explicit updates
this.searchStore.filter$.pipe(
  skip(1) // Skip the BehaviorSubject's initial value; only process user-driven changes
).subscribe(filter => this.logFilterChange(filter));
```

### `skipUntil(notifier$)` — Skip Until Another Observable Emits

`skipUntil(notifier$)` ignores all values until the notifier emits at least once, then lets everything through.

```typescript
import { interval, timer, skipUntil } from 'rxjs';

// Start accepting values only after a 3-second warm-up period
interval(100).pipe(
  skipUntil(timer(3000)) // Ignore all values emitted in the first 3 seconds
).subscribe(v => console.log(v)); // Values only from second 3 onward

// Skip API results until authentication is confirmed
this.apiData$.pipe(
  skipUntil(this.authConfirmed$) // Don't process data until auth is established
).subscribe(data => processData(data));
```

### `skipWhile(predicate)` — Skip While Condition Is True

`skipWhile(predicate)` ignores values while the predicate returns `true` and starts letting values through once the predicate first returns `false`. Unlike `filter()`, once the predicate becomes false and values start flowing, `skipWhile` lets ALL subsequent values through — even if the predicate would return true again for later values.

```typescript
import { from, skipWhile } from 'rxjs';

// Skip loading states, start processing when first real value arrives
from([null, null, { data: 'ready' }, null, { data: 'more' }]).pipe(
  skipWhile(v => v === null) // Skip initial nulls — BUT once non-null arrives, null is allowed through
).subscribe(v => console.log(v));
// { data: 'ready' }, null, { data: 'more' }  ← the null after 'ready' passes through!

// This is different from filter(v => v !== null) which would skip ALL nulls
```

Understanding the difference between `skipWhile` and `filter` for this use case is important. `skipWhile` skips *until the condition breaks once*, then stops checking — making it a one-way gate. `filter` checks every value forever.

---

## 15.5.4 — `first()` and `last()` — Single Value Shortcuts

`first(predicate?)` emits the first value that satisfies an optional predicate and then completes. It throws an error if no value satisfies the predicate before the stream completes (unless you provide a default value).

`last(predicate?)` waits for the stream to complete and then emits the last value that satisfied an optional predicate. Like `takeLast(1)`, it requires the stream to complete.

```typescript
import { from, first, last } from 'rxjs';

// First value
from([1, 2, 3]).pipe(first()).subscribe(v => console.log(v)); // 1

// First value satisfying a predicate
from([1, 2, 3, 4, 5]).pipe(
  first(v => v > 3)
).subscribe(v => console.log(v)); // 4

// With a default value — prevents error if no value matches
from([1, 2, 3]).pipe(
  first(v => v > 10, 0) // Default: 0 if nothing > 10
).subscribe(v => console.log(v)); // 0 (default)

// Common Angular pattern: get the current value from a BehaviorSubject's observable
// without staying subscribed
const currentUser = await firstValueFrom(this.authService.user$);

// last() — needs a completing stream
from([1, 2, 3, 4, 5]).pipe(
  last(v => v % 2 === 0)
).subscribe(v => console.log(v)); // 4 (last even number)
```

---

## 15.5.5 — `distinctUntilChanged()` and `distinct()` — Deduplicate Emissions

These operators prevent duplicate values from passing through.

### `distinctUntilChanged(comparator?)` — Skip Consecutive Duplicates

`distinctUntilChanged()` compares each new value with the previously emitted value. If they are equal (using reference equality by default), the new value is dropped. Only when the value *changes* does it pass through. This is the operator behind Angular's `OnPush` change detection philosophy — only process changes, not repetitions.

```typescript
import { BehaviorSubject, distinctUntilChanged, map } from 'rxjs';

// Prevent re-renders when the value hasn't actually changed
const windowWidth$ = fromEvent(window, 'resize').pipe(
  map(() => window.innerWidth),
  distinctUntilChanged() // Only emit when width actually changes
);

// Custom comparator for objects — compare by a specific property
const productUpdates$ = new BehaviorSubject<Product>({ id: '1', name: 'Widget', stock: 5 });
productUpdates$.pipe(
  distinctUntilChanged((prev, curr) => prev.stock === curr.stock) // Only emit when stock changes
).subscribe(product => updateStockIndicator(product));

// Angular routing — react only to actual route changes
this.router.events.pipe(
  filter(e => e instanceof NavigationEnd),
  map(e => (e as NavigationEnd).urlAfterRedirects),
  distinctUntilChanged() // Don't fire for same-URL navigations
).subscribe(url => this.analyticsService.pageView(url));
```

### `distinct(keySelector?)` — Skip ALL Previously Seen Values

`distinct()` tracks ALL previously emitted values (not just the most recent) and only lets through values that have never been emitted before in the entire stream's lifetime. Optionally takes a key selector function for object comparison.

```typescript
import { from, distinct } from 'rxjs';

from([1, 2, 1, 3, 2, 4]).pipe(
  distinct()
).subscribe(v => console.log(v)); // 1, 2, 3, 4

// Distinct by property
from([
  { id: 1, category: 'books' },
  { id: 2, category: 'books' },  // Different object but same category
  { id: 3, category: 'electronics' },
]).pipe(
  distinct(item => item.category) // Key selector: compare categories
).subscribe(item => console.log(item.category)); // books, electronics
```

**Memory consideration:** `distinct()` keeps an internal `Set` of all values ever seen. For long-running streams with many unique values, this Set grows unboundedly. For infinite streams, prefer `distinctUntilChanged()` which only keeps the last value. `distinct()` is most appropriate for finite streams like processing a batch of records.

---

## 15.5.6 — `debounceTime()` and `debounce()` — Wait for a Pause in Activity

Debounce operators address a specific and very common problem: a high-frequency event source that should only trigger an action after activity has *paused* for a specified duration. The classic example is search-as-you-type — you don't want to fire an API call for every keystroke; you want to wait until the user has stopped typing for 300ms.

### `debounceTime(milliseconds)` — Simplest Debounce

`debounceTime(ms)` waits `ms` milliseconds after the most recent emission. If no new value arrives during that wait, the most recently received value is emitted. If a new value arrives before the wait ends, the timer resets and the previous value is discarded.

```
Source:  --a-b----c-d-e--------f--|
debounceTime(100ms)
Result:  ---------b-----------e---f--|

'a' is discarded (b arrives before timeout)
'b' is emitted (pause after b)
'c', 'd' are discarded
'e' is emitted (pause after e)
'f' is emitted (then stream completes)
```

```typescript
import { fromEvent, debounceTime, map, switchMap } from 'rxjs';

// Search-as-you-type with debounce
const searchInput = document.getElementById('search') as HTMLInputElement;
fromEvent(searchInput, 'input').pipe(
  map(e => (e.target as HTMLInputElement).value.trim()),
  debounceTime(300),       // Wait 300ms after last keystroke
  distinctUntilChanged(),  // Don't search if value hasn't changed
  switchMap(query => query.length > 1
    ? this.http.get<Result[]>(`/api/search?q=${query}`)
    : of([])
  )
).subscribe(results => this.searchResults = results);

// Window resize debounce — only recalculate layout after user stops resizing
fromEvent(window, 'resize').pipe(
  debounceTime(200) // Wait 200ms after last resize event
).subscribe(() => this.recalculateLayout());

// Form value debounce — auto-save drafts
this.form.valueChanges.pipe(
  debounceTime(2000) // Wait 2 seconds after user stops editing
).subscribe(value => this.autoSaveDraft(value));
```

### `debounce(durationSelector)` — Dynamic Debounce Duration

`debounce()` is the dynamic version where the debounce duration is itself an Observable. The duration Observable is subscribed to after each source emission, and the source value is emitted when the duration Observable emits.

```typescript
import { fromEvent, debounce, interval } from 'rxjs';

// Variable debounce: longer debounce for short queries (likely mid-typing),
// shorter for longer queries (likely complete)
fromEvent(searchInput, 'input').pipe(
  map(e => (e.target as HTMLInputElement).value),
  debounce(query =>
    query.length < 3
      ? interval(500) // Wait longer for very short queries
      : interval(200)   // Shorter wait for longer queries
  )
).subscribe(query => this.search(query));
```

---

## 15.5.7 — `throttleTime()` and `throttle()` — Limit Rate of Output

Throttle operators are the complement of debounce. While debounce emits *after activity stops*, throttle emits at most once per time window — it limits the maximum rate of output regardless of how fast the source emits.

Throttle is appropriate when you want to react to frequent events at a controlled rate, rather than only after a pause. The difference in behavior: if you type continuously for 10 seconds, `debounceTime(300)` emits once (300ms after you stop). `throttleTime(300)` emits approximately every 300ms throughout the entire 10 seconds.

```typescript
import { fromEvent, throttleTime } from 'rxjs';

// Scroll events — only process at most once per 100ms regardless of scroll speed
fromEvent(window, 'scroll').pipe(
  throttleTime(100)
).subscribe(() => updateScrollIndicator());

// Button click protection — ignore clicks for 1 second after each valid click
fromEvent(submitButton, 'click').pipe(
  throttleTime(1000) // At most one submission per second
).subscribe(() => this.handleSubmit());

// Mouse tracking — limit position updates to 30fps
fromEvent<MouseEvent>(document, 'mousemove').pipe(
  throttleTime(1000 / 30) // ~33ms per frame
).subscribe(e => updateCursorPosition(e.clientX, e.clientY));
```

`throttleTime` has an optional `config` parameter controlling the "leading" (emit at the start of the window) and "trailing" (emit the last value at the end of the window) edge behavior, similar to Lodash's throttle function.

---

## 15.5.8 — `auditTime()` and `audit()` — Emit the Last Value After a Fixed Window

`auditTime(ms)` ignores all source values for `ms` milliseconds, then emits the *most recent* value from the source at the end of that window. Unlike debounce (which resets the timer on each new value), `auditTime` runs a fixed duration window and emits whatever is the latest value when the window closes.

```
Source:   --a-b-c-d-------e-f--g--|
auditTime(100ms)
           [---100ms---]  [---100ms---]
Result:   ----------d-----------g-|

Emits the LAST value ('d') at the end of each time window
```

```typescript
import { fromEvent, auditTime } from 'rxjs';

// Emit the most recent input value every 500ms, not on every keystroke
fromEvent(input, 'input').pipe(
  auditTime(500) // Every 500ms, emit whatever is currently in the input
).subscribe(e => this.updatePreview((e.target as HTMLInputElement).value));
```

The key distinction between `debounceTime`, `throttleTime`, and `auditTime` comes down to which value you emit and when. `debounceTime` emits the last value *after a gap in activity*. `throttleTime` emits the first value of each time window (leading edge by default). `auditTime` emits the last value of each time window, on a fixed schedule.

---

## 15.5.9 — `sampleTime()` and `sample()` — Periodic Snapshots

`sampleTime(ms)` emits the most recent value from the source at regular intervals, but only if the source has emitted since the last sample. It is like taking a periodic "snapshot" of whatever the latest value was.

```typescript
import { fromEvent, sampleTime, map } from 'rxjs';

// Sample mouse position every 200ms — useful for smooth analytics without per-pixel overhead
fromEvent<MouseEvent>(document, 'mousemove').pipe(
  map(e => ({ x: e.clientX, y: e.clientY })),
  sampleTime(200) // Snapshot position every 200ms
).subscribe(pos => sendPositionToAnalytics(pos));
```

`sample(notifier$)` is the same idea but uses an external Observable as the sampling trigger instead of a fixed interval.

---

## Important Points and Best Practices

The debounce vs throttle distinction trips up many developers. Use the right mental model: debounce answers the question "has activity *stopped*?" and throttle answers "how often should I react to *continuous* activity?" For search inputs, debounce. For scroll events driving UI updates, throttle or auditTime.

`distinctUntilChanged()` should be in your reflexive toolkit for any stream derived from state. If you are selecting from an NgRx store, a BehaviorSubject service, or Angular signals via `toObservable()`, adding `distinctUntilChanged()` prevents downstream reactions to state updates that did not actually change the value you care about. This is a free performance optimization that is almost always correct.

Always pair `takeUntil(destroy$)` with long-lived subscriptions in Angular components. Any subscription to `interval`, `fromEvent`, WebSocket Observables, or polling patterns will continue running after the component is destroyed if you do not manage the subscription lifetime. This is the most common source of memory leaks in Angular applications. Modern Angular makes `takeUntilDestroyed()` available as a better alternative (see section 15.9).

Be careful with `first()` when the predicate might never match. If you write `source$.pipe(first(v => v.status === 'approved'))` and the status is never 'approved', `first()` will emit an `EmptyError` when the source completes. Always provide a default value as the second argument when the match is not guaranteed: `first(predicate, defaultValue)`.

# Phase 15.6 — Combination Operators
## merge, concat, forkJoin, combineLatest, zip, withLatestFrom, race, and pipeable variants

---

## What Are Combination Operators?

Combination operators take **multiple Observable inputs** and weave them together into a single output Observable. This is where reactive programming truly shines — coordinating multiple asynchronous sources that were previously independent into coherent, combined streams. Whether you need to load data from three APIs simultaneously, react to two user inputs together, or coordinate multiple streams with complex timing requirements, combination operators are the tools you reach for.

Each combination operator has a fundamentally different answer to the question: "when should the output Observable emit, and with what value?" Understanding that question for each operator is the key to choosing the right one.

---

## 15.6.1 — `merge()` — Concurrent Passthrough

`merge()` subscribes to all source Observables simultaneously and emits every value from every source as they arrive — first-come, first-served. The output Observable completes only when ALL source Observables have completed.

```
Source A:  --a1----a2--------a3--|
Source B:  ----b1------b2--------|
merge(A, B)
Result:    --a1-b1-a2--b2----a3--|

Every emission from every source passes through immediately.
Order in the result depends purely on timing.
```

```typescript
import { merge, fromEvent, interval } from 'rxjs';
import { map } from 'rxjs/operators';

// Listen to multiple event types as a unified stream
const userActivity$ = merge(
  fromEvent(document, 'click').pipe(map(() => 'click')),
  fromEvent(document, 'keydown').pipe(map(() => 'keydown')),
  fromEvent(document, 'scroll').pipe(map(() => 'scroll')),
);

userActivity$.subscribe(activityType => {
  resetIdleTimer();
  console.log('User activity:', activityType);
});

// Merge multiple data sources — process whichever arrives first
const notifications$ = merge(
  this.http.get<Notification[]>('/api/notifications/urgent'),
  this.http.get<Notification[]>('/api/notifications/regular'),
  this.websocket.messages$.pipe(filter(m => m.type === 'notification'))
);

notifications$.subscribe(notifications => this.addToQueue(notifications));

// Merge with concurrency limit (third argument)
// Process at most 2 inner streams at a time
const tasks$ = from(taskList);
merge(tasks$, 2).subscribe(result => console.log(result));
```

Use `merge()` when your sources are independent and you want to process all of them in real time, with no preference for any particular source's values over another's. The critical property to remember is that `merge()` does NOT wait — it subscribes to everything at once and emits as fast as sources produce.

---

## 15.6.2 — `concat()` — Sequential Subscription

`concat()` subscribes to source Observables **one at a time, in order**. It only subscribes to the next source after the current one completes. The output is the concatenated sequence of all sources in order, regardless of timing.

```
Source A:  --a1--a2--|
Source B:             --b1--b2--|
concat(A, B)
Result:    --a1--a2----b1--b2--|

B doesn't start until A is completely done.
If A never completes, B is never subscribed to.
```

```typescript
import { concat, of, timer } from 'rxjs';
import { map, delay } from 'rxjs/operators';

// Show intro animation, then the main content
const intro$ = timer(0, 100).pipe(take(5), map(v => ({ screen: 'intro', frame: v })));
const mainContent$ = of({ screen: 'main' });

concat(intro$, mainContent$).subscribe(screen => render(screen));
// Renders 5 intro frames, then renders main content once intro is done

// Load data in sequence — each request starts only after the previous finishes
const step1$ = this.http.get('/api/setup/step1');
const step2$ = this.http.get('/api/setup/step2');
const step3$ = this.http.get('/api/setup/step3');

concat(step1$, step2$, step3$).subscribe({
  next: result => console.log('Step result:', result),
  complete: () => console.log('All setup steps complete'),
});

// Provide immediate feedback while loading real data
const immediateDefault$ = of({ items: [], loading: true });
const actualData$ = this.http.get<ItemsResponse>('/api/items').pipe(
  map(data => ({ items: data.items, loading: false }))
);

concat(immediateDefault$, actualData$).subscribe(state => this.state = state);
// First emission: { items: [], loading: true }  ← immediate, shows spinner
// Second emission: { items: [...real data], loading: false } ← when HTTP completes
```

`concat()` is perfect when order and completeness matter — when you need step A done before step B starts, when you need to prepend static values before a dynamic stream, or when you want to sequence animations.

---

## 15.6.3 — `forkJoin()` — Wait for All to Complete (Like Promise.all)

`forkJoin()` subscribes to all source Observables simultaneously and waits until they ALL complete. When the last one completes, it emits a single array (or object) containing the *last value* from each source, in the same order (or under the same key). If any source errors before completing, the whole `forkJoin` errors. If any source completes with NO values, `forkJoin` completes with no values either.

```
Source A:  --a1--a2--|
Source B:  ----b1--------b2--|
forkJoin([A, B])
Result:    -----------------[a2, b2]|

Waits for both to complete, then emits one combined value.
```

```typescript
import { forkJoin } from 'rxjs';

// Load multiple resources simultaneously — all required for the page to render
forkJoin({
  user: this.http.get<User>('/api/user'),
  permissions: this.http.get<Permission[]>('/api/permissions'),
  settings: this.http.get<Settings>('/api/settings'),
}).subscribe({
  next: ({ user, permissions, settings }) => {
    // All three are available together — guaranteed to be the final values
    this.initializePage(user, permissions, settings);
  },
  error: err => {
    // If ANY of the three requests fails, this error handler fires
    this.showErrorPage(err.message);
  }
});

// Array syntax for positional results
forkJoin([
  this.productsService.getAll(),
  this.categoriesService.getAll(),
]).subscribe(([products, categories]) => {
  // Destructure by position — products and categories are both fully loaded
  this.initializeCatalog(products, categories);
});

// Important: forkJoin only makes sense with HTTP requests or other finite Observables.
// If you pass an infinite Observable (like interval()), forkJoin will never emit
// because it waits for ALL sources to complete.
```

**The critical distinction from `combineLatest`:** `forkJoin` only emits ONCE (the final combined value when all complete). `combineLatest` emits every time ANY source emits (continuously reactive). Use `forkJoin` for "load everything and show once." Use `combineLatest` for "react continuously to changing state from multiple sources."

**Handling partial failure:** If you want `forkJoin` to succeed even when some requests fail, pipe each individual source with `catchError(err => of(null))` before passing to `forkJoin`:

```typescript
forkJoin({
  required: this.http.get('/api/required-data'),  // This must succeed
  optional: this.http.get('/api/optional-data').pipe(
    catchError(() => of(null)) // This can fail gracefully
  ),
}).subscribe(({ required, optional }) => {
  // required has data, optional may be null
});
```

---

## 15.6.4 — `combineLatest()` — Reactive Combination of Latest Values

`combineLatest()` is the most powerful and commonly used combination operator. It subscribes to all sources simultaneously and emits a new combined array (or object) **every time any source emits a new value**, using the most recent value from every other source.

The critical detail: `combineLatest` does not emit until **every source has emitted at least once**. After that threshold, every new emission from any source triggers a new combined output.

```
Source A:  --a1--------a2--------|
Source B:  ------b1-------b2-----|
combineLatest([A, B])
Result:    ------[a1,b1]-[a2,b1]-[a2,b2]|

First emission: when BOTH a1 and b1 have arrived.
Subsequent: whenever either changes.
```

```typescript
import { combineLatest, BehaviorSubject } from 'rxjs';
import { map } from 'rxjs/operators';

// A view model that reacts to multiple pieces of state
const searchQuery$ = new BehaviorSubject<string>('');
const selectedCategory$ = new BehaviorSubject<string>('all');
const sortOrder$ = new BehaviorSubject<'asc' | 'desc'>('asc');

// Whenever ANY of the three change, the view model updates automatically
const viewModel$ = combineLatest({
  query: searchQuery$,
  category: selectedCategory$,
  sort: sortOrder$,
}).pipe(
  // Derive the filtered+sorted product list whenever any input changes
  switchMap(({ query, category, sort }) =>
    this.http.get<Product[]>('/api/products', {
      params: { query, category, sort }
    })
  )
);

viewModel$.subscribe(products => this.products = products);

// Update any of the inputs — the viewModel$ automatically re-emits
searchQuery$.next('angular');          // viewModel$ updates with new query
selectedCategory$.next('frameworks');  // viewModel$ updates with new category

// Angular template equivalent using the async pipe:
// viewModel$ | async — subscribes and unsubscribes automatically
```

```typescript
// Form validation that depends on multiple fields
combineLatest({
  email: this.emailControl.valueChanges.pipe(startWith(this.emailControl.value)),
  password: this.passwordControl.valueChanges.pipe(startWith(this.passwordControl.value)),
  confirmPassword: this.confirmControl.valueChanges.pipe(startWith(this.confirmControl.value)),
}).pipe(
  map(({ email, password, confirmPassword }) => ({
    emailValid: /^[^@]+@[^@]+\.[^@]+$/.test(email ?? ''),
    passwordValid: (password?.length ?? 0) >= 8,
    passwordsMatch: password === confirmPassword,
    allValid: true, // computed below
  })),
  map(v => ({ ...v, allValid: v.emailValid && v.passwordValid && v.passwordsMatch }))
).subscribe(validation => this.validation = validation);
```

**The `startWith` trick for `combineLatest`:** Because `combineLatest` requires every source to have emitted at least once, combining sources where some are slow to emit can cause the combined stream to be delayed. Adding `.pipe(startWith(initialValue))` to each source ensures the combined stream emits immediately with the initial values for all sources.

---

## 15.6.5 — `zip()` — Pair Values by Index

`zip()` pairs values from multiple sources **by index position**. The first value from source A is paired with the first value from source B. The second value from A is paired with the second value from B. And so on. It waits for all sources to have emitted their N-th value before emitting the N-th combined value.

```
Source A:  --a1----a2------a3--|
Source B:  --------b1--b2--b3--|
zip([A, B])
Result:    --------[a1,b1]-[a2,b2]-[a3,b3]|

a1 is paired with b1 (both are "first" values)
Even though a2 arrived much earlier, it waits for b2
```

```typescript
import { zip, of, interval } from 'rxjs';
import { map } from 'rxjs/operators';

// Pair questions with answers by position
const questions$ = of('What is 1+1?', 'What color is the sky?', 'How many days in a week?');
const answers$ = of('2', 'Blue', '7');

zip(questions$, answers$).subscribe(([question, answer]) => {
  console.log(`Q: ${question} → A: ${answer}`);
});
// Q: What is 1+1? → A: 2
// Q: What color is the sky? → A: Blue
// Q: How many days in a week? → A: 7

// Animate items with a stagger delay
const items = ['item1', 'item2', 'item3', 'item4'];
zip(
  from(items),
  interval(150), // Each item is "released" 150ms after the previous
).pipe(
  map(([item]) => item)
).subscribe(item => animateIn(item));
// item1 at 0ms, item2 at 150ms, item3 at 300ms, item4 at 450ms

// Combine two streams where you need strict positional pairing
zip(
  this.http.get<string[]>('/api/image-urls'),
  this.http.get<string[]>('/api/image-captions'),
).pipe(
  map(([urls, captions]) => urls.map((url, i) => ({ url, caption: captions[i] })))
).subscribe(images => this.images = images);
```

`zip` is the right choice when sources represent parallel indexed sequences and you need to combine them element-by-element. It is rarely the right tool for event streams (since events do not have meaningful index-based pairing), but perfect for combining positionally-related finite data sources.

---

## 15.6.6 — `withLatestFrom()` — Combine Trigger with Latest Snapshot

`withLatestFrom(other$)` combines the source Observable with another Observable, but unlike `combineLatest`, it only emits when the **primary source** emits — the second Observable provides only a snapshot of its latest value at that moment.

The second Observable is effectively "sampled" each time the primary source emits. If the second Observable has not yet emitted, the primary emission is silently dropped.

```
Primary:   --a1-------a2-----a3--|
Other:     -----b1--b2-----------|
withLatestFrom(Other)
Result:    ----------[a2,b2]-[a3,b2]|

a1 is dropped (Other hasn't emitted yet when a1 arrives)
a2 is paired with b2 (latest from Other at that moment)
a3 is paired with b2 (b2 is still the latest since Other hasn't changed)
```

```typescript
import { fromEvent, withLatestFrom } from 'rxjs';

// Submit button reads the current form value, but doesn't subscribe to every change
const submitClicks$ = fromEvent(submitButton, 'click');
const formValue$ = this.form.valueChanges.pipe(startWith(this.form.value));

submitClicks$.pipe(
  withLatestFrom(formValue$)  // Read current form value at the moment of click
).subscribe(([_, formValue]) => {
  // formValue is a snapshot of the form at the moment of click
  // We're NOT reacting to every form change — only to clicks
  this.saveForm(formValue);
});

// NgRx pattern: combine user action with current store state
this.actions$.pipe(
  ofType(UserActions.saveProfile),
  withLatestFrom(this.store.select(selectCurrentUser)), // Snapshot of store
).subscribe(([action, currentUser]) => {
  // currentUser is the latest user from the store at the moment the action was dispatched
  this.apiService.updateProfile(currentUser.id, action.changes);
});

// Key insight: withLatestFrom vs combineLatest
// withLatestFrom: only the LEFT (primary) source drives emissions
// combineLatest: EITHER source drives emissions
```

`withLatestFrom` is the solution to a very common problem: you have a "trigger" stream (button clicks, actions, route changes) and a "context" stream (current state, user info, form data). You want to react to the trigger but need access to the current context at that moment, without reacting every time the context changes. `withLatestFrom` reads the context as a snapshot when the trigger fires.

---

## 15.6.7 — `race()` — First to Emit Wins

`race()` subscribes to all sources simultaneously and emits all values from whichever source emits its **first** value first. All other sources are immediately unsubscribed once the "winner" is determined.

```
Source A:  ------a1--a2--|
Source B:  --b1-------b2--|
race([A, B])
Result:    --b1-------b2--|   (B won the race, A is unsubscribed)
```

```typescript
import { race, timer, fromEvent } from 'rxjs';
import { mapTo } from 'rxjs/operators';

// Show "press a button or we'll navigate automatically" pattern
const userAction$ = fromEvent(document, 'click').pipe(mapTo('user'));
const timeout$ = timer(10_000).pipe(mapTo('timeout'));

race([userAction$, timeout$]).pipe(
  take(1)
).subscribe(winner => {
  if (winner === 'timeout') this.router.navigate(['/home']);
  // If user clicks, the timeout is cancelled — user stays on the page
});

// Fastest cache vs network — use whichever responds first
const cached$ = this.cacheService.get(key);
const network$ = this.http.get(`/api/data/${key}`);

race([cached$, network$]).pipe(
  take(1) // Only take the first one that responds
).subscribe(data => this.data = data);
```

`race()` is most useful for timeout patterns, "first available source wins" architectures, and A/B testing where you want whichever experiment responds first.

---

## 15.6.8 — Pipeable Combination Variants

RxJS also provides **pipeable instance operator versions** of several combination operators. These allow you to combine a source with additional streams inside a `.pipe()` chain, rather than as a static function that takes all sources as arguments.

```typescript
import { combineLatestWith, mergeWith, concatWith, zipWith, raceWith } from 'rxjs/operators';

// combineLatestWith — use inside pipe when you have a primary source
this.searchQuery$.pipe(
  combineLatestWith(this.categoryFilter$, this.sortOrder$),
  // Equivalent to: combineLatest([this.searchQuery$, this.categoryFilter$, this.sortOrder$])
  // but primary source is explicit
  map(([query, category, sort]) => ({ query, category, sort })),
).subscribe(params => this.fetchResults(params));

// mergeWith — merge additional sources inside a pipeline
primaryEvents$.pipe(
  mergeWith(additionalEvents$, backupEvents$)
).subscribe(event => handleEvent(event));

// concatWith — append additional sources sequentially
initialData$.pipe(
  concatWith(updatedData$)
).subscribe(data => renderData(data));
```

These pipeable variants are often cleaner when one source is clearly "primary" and others are supplementary. They read more naturally in a pipeline because the primary source is visually at the top of the chain.

---

## A Mental Model for Choosing Combination Operators

When you need to combine streams, ask yourself three questions in order. First: do I need to process sources one at a time (sequentially), or all at once (concurrently)? If sequentially, use `concat`. If concurrently, continue. Second: do I need all sources to complete before I emit, or should I emit reactively as values arrive? If "wait for all," use `forkJoin`. If "emit reactively," continue. Third: should the output emit when ANY source changes (both drive the output), or only when the PRIMARY source changes (one drives, others are snapshots)? If any-drives, use `combineLatest`. If primary-drives, use `withLatestFrom`. Special cases: `zip` for strict index pairing, `race` for first-response wins, `merge` for unstructured concurrent passthrough.

---

## Important Points and Best Practices

The `forkJoin` vs `combineLatest` confusion is the most common mistake when learning combination operators. Remember: `forkJoin` is for "load everything once and continue," like a Promise.all. `combineLatest` is for "react every time any input changes," like a reactive spreadsheet formula. If you catch yourself using `combineLatest` to load data once and never update, you probably want `forkJoin`. If you catch yourself using `forkJoin` but wondering why it doesn't update when state changes, you probably want `combineLatest`.

Never use `combineLatest` without adding `startWith` to streams that might not have emitted yet. If you combine a `BehaviorSubject` (which has an initial value) with an HTTP Observable (which hasn't emitted yet), `combineLatest` will not emit until the HTTP call completes, even though the BehaviorSubject is ready immediately. Add `startWith(null)` or a meaningful initial value to the HTTP Observable to ensure immediate combination.

`withLatestFrom` silently drops primary emissions if the secondary source has not emitted yet. This can cause confusing behavior where button clicks appear to do nothing. If you need the combination to wait rather than drop, use `combineLatest` with `take(1)` on the primary source, or ensure the secondary Observable emits immediately by converting a BehaviorSubject or using `startWith`.

Be cautious of `race()` with sources that have side effects. The "loser" sources are unsubscribed, but if subscribing to them caused side effects (like HTTP requests being sent), those requests still go to the server — they are just no longer listened to. This is the same cancel-without-undo problem as `switchMap` for write operations.

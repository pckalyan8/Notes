# Phase 14.2 — Component State Patterns
## Local Signals, Smart/Dumb Components, Lifting State, and State Machines

---

## The Philosophy of Component State

Before reaching for any shared state solution, you should ask whether the state you are about to manage truly needs to leave the component at all. Component state is state that lives *inside* a single component and has no reason to be visible to anything outside it. It is the simplest possible form of state management, and simplicity is a virtue. The discipline of keeping state local whenever possible is what prevents your application from becoming an unmaintainable tangle of shared dependencies.

In Angular 2026, component state is managed primarily through **signals** — Angular's built-in reactive primitive that replaced the older approach of maintaining local variables and relying on Zone.js to detect changes. Signals make component state explicit, reactive, and performant in a way that plain class properties never were.

---

## 14.2.1 — Local State with Signals

A signal is the simplest possible reactive container. It holds a value, and anything that reads it during a reactive context (a template, a `computed()`, or an `effect()`) is automatically notified when the value changes. For component-local state, signals replace what were previously just class properties or `BehaviorSubject` instances.

### The Basic Pattern

The most straightforward use of signal-based local state is for UI toggles, counters, selections, and visibility flags. These are pieces of data that only this component cares about and that reset when the component is destroyed.

```typescript
import { Component, signal, computed } from '@angular/core';

@Component({
  selector: 'app-expandable-card',
  standalone: true,
  template: `
    <div class="card" [class.expanded]="isExpanded()">
      <div class="card-header" (click)="toggleExpand()">
        <h3>{{ title }}</h3>
        <!-- The button label reacts to signal changes automatically -->
        <button>{{ isExpanded() ? 'Collapse' : 'Expand' }}</button>
      </div>

      @if (isExpanded()) {
        <div class="card-body">
          <ng-content />
        </div>
      }
    </div>
  `
})
export class ExpandableCardComponent {
  // signal() creates a writable reactive container with an initial value
  // This state is completely private to this component instance
  isExpanded = signal(false);

  title = 'Details';

  toggleExpand() {
    // .update() takes the current value and returns the new value
    // This is the immutable update pattern — no direct mutation
    this.isExpanded.update(current => !current);
  }
}
```

### Managing Multiple Related Local State Values

When a component has several pieces of state that relate to the same concept, it is often better to group them into a single signal containing an object than to scatter many individual signals. This keeps related state together and makes updates more atomic — you update them all at once rather than in separate operations that could leave state briefly inconsistent.

```typescript
@Component({
  selector: 'app-image-gallery',
  standalone: true,
  template: `
    <div class="gallery">
      <img [src]="gallery().images[gallery().activeIndex]" [alt]="gallery().caption" />

      <div class="controls">
        <button (click)="prev()" [disabled]="gallery().activeIndex === 0">Previous</button>
        <span>{{ gallery().activeIndex + 1 }} of {{ gallery().images.length }}</span>
        <button (click)="next()" [disabled]="isLastImage()">Next</button>
      </div>

      @if (gallery().isFullscreen) {
        <div class="fullscreen-overlay" (click)="exitFullscreen()">
          <img [src]="gallery().images[gallery().activeIndex]" />
        </div>
      }
    </div>
  `
})
export class ImageGalleryComponent {
  // Group related state into one signal — the "gallery state" is one coherent concept
  gallery = signal({
    images: ['photo1.jpg', 'photo2.jpg', 'photo3.jpg'],
    activeIndex: 0,
    caption: 'Photo 1',
    isFullscreen: false,
  });

  // computed() derives a value from other signals — it automatically recalculates
  // only when gallery() changes, and is memoized (cached until it needs to update)
  isLastImage = computed(() =>
    this.gallery().activeIndex === this.gallery().images.length - 1
  );

  next() {
    this.gallery.update(state => {
      const nextIndex = state.activeIndex + 1;
      return {
        ...state,           // spread existing state — preserve all other fields
        activeIndex: nextIndex,
        caption: `Photo ${nextIndex + 1}`,
      };
    });
  }

  prev() {
    this.gallery.update(state => {
      const prevIndex = state.activeIndex - 1;
      return { ...state, activeIndex: prevIndex, caption: `Photo ${prevIndex + 1}` };
    });
  }

  exitFullscreen() {
    // When updating only one field, you can use a targeted spread like this
    this.gallery.update(state => ({ ...state, isFullscreen: false }));
  }
}
```

### Using `effect()` for Local Side Effects

Sometimes a piece of local state needs to trigger a side effect — starting an animation, updating a canvas element, or scrolling to a position. The `effect()` function lets you run code reactively whenever its signal dependencies change, without subscribing to observables.

```typescript
import { Component, signal, effect, ElementRef, ViewChild, inject, NgZone } from '@angular/core';

@Component({
  selector: 'app-progress-bar',
  template: `
    <canvas #progressCanvas width="300" height="20"></canvas>
    <button (click)="advance()">Advance Progress</button>
  `
})
export class ProgressBarComponent {
  @ViewChild('progressCanvas') canvas!: ElementRef<HTMLCanvasElement>;
  private ngZone = inject(NgZone);

  progress = signal(0); // 0 to 100

  constructor() {
    // effect() runs whenever this.progress() changes
    // Run outside Angular zone because canvas drawing should not trigger change detection
    effect(() => {
      const value = this.progress(); // reading the signal registers this dependency
      // All canvas drawing happens as a side effect of the signal change
      this.ngZone.runOutsideAngular(() => this.drawProgress(value));
    });
  }

  advance() {
    this.progress.update(v => Math.min(v + 10, 100));
  }

  private drawProgress(value: number) {
    const ctx = this.canvas?.nativeElement?.getContext('2d');
    if (!ctx) return;
    ctx.clearRect(0, 0, 300, 20);
    ctx.fillStyle = '#6200ea';
    ctx.fillRect(0, 0, value * 3, 20);
  }
}
```

---

## 14.2.2 — Smart/Dumb Component Pattern (Container/Presenter)

The Smart/Dumb component pattern — also called the Container/Presenter pattern — is one of the most important architectural patterns in Angular development. It is a specific application of the Single Responsibility Principle to UI components.

The central idea is to divide components into two distinct roles. A **Smart component** (Container) knows about the application's state and services. It fetches data, dispatches actions, subscribes to state, and passes data down to its children. It should contain very little template markup. A **Dumb component** (Presenter) knows nothing about state, services, or where its data comes from. It receives data via `@Input()` signals, emits events via `@Output()` signals, and focuses purely on rendering and user interaction. It is a pure function of its inputs.

### Why This Separation Matters

When a component is both responsible for fetching data *and* rendering it, it becomes tightly coupled to the specific data source. You cannot reuse it in a context with different data, you cannot test the rendering logic without setting up the data-fetching machinery, and you cannot reason about either concern independently.

When you separate the concerns, the Presenter component becomes a reusable, easily testable, and easily documented UI piece. The Container component becomes a thin coordination layer that can be replaced, reconfigured, or mocked without touching the Presenter. This is the architectural foundation that makes large-scale Angular applications maintainable.

### A Concrete Example

Imagine a user management section of an admin panel. A poorly structured approach puts everything in one component. A well-structured approach separates the data concern from the display concern.

```typescript
// ---- THE SMART COMPONENT (Container) ----
// Its job: connect the UI to state and services
// It knows about NgRx, HTTP services, routing
// It should have minimal HTML in its template
@Component({
  selector: 'app-users-container',
  standalone: true,
  imports: [UsersTableComponent, UserSearchComponent],
  template: `
    <!-- The container just passes data down and listens for events -->
    <app-user-search
      [initialQuery]="currentQuery()"
      (queryChange)="onQueryChange($event)"
    />

    <app-users-table
      [users]="filteredUsers()"
      [loading]="loading()"
      [error]="error()"
      (deleteUser)="onDeleteUser($event)"
      (editUser)="onNavigateToEdit($event)"
    />
  `
})
export class UsersContainerComponent {
  private userStore = inject(UserStore);         // knows about the store
  private router = inject(Router);               // knows about routing

  // Container reads from the store and passes data to presenters
  users = this.userStore.users;
  loading = this.userStore.loading;
  error = this.userStore.error;
  currentQuery = signal('');

  filteredUsers = computed(() => {
    const q = this.currentQuery().toLowerCase();
    return this.users().filter(u =>
      u.name.toLowerCase().includes(q) || u.email.toLowerCase().includes(q)
    );
  });

  constructor() {
    // Container is responsible for triggering data loads
    this.userStore.loadUsers();
  }

  // Container handles business-level events
  onDeleteUser(userId: string) {
    this.userStore.deleteUser(userId);
  }

  onQueryChange(query: string) {
    this.currentQuery.set(query);
  }

  onNavigateToEdit(userId: string) {
    this.router.navigate(['/admin/users', userId, 'edit']);
  }
}
```

```typescript
// ---- THE DUMB COMPONENT (Presenter) ----
// Its job: display data and report user interactions
// It knows NOTHING about NgRx, services, or routing
// It is a pure function of its inputs
@Component({
  selector: 'app-users-table',
  standalone: true,
  imports: [MatTableModule, MatButtonModule, MatProgressBarModule],
  template: `
    @if (loading()) {
      <mat-progress-bar mode="indeterminate" />
    }

    @if (error()) {
      <div class="error-banner">{{ error() }}</div>
    }

    <mat-table [dataSource]="users()">
      <ng-container matColumnDef="name">
        <th mat-header-cell *matHeaderCellDef>Name</th>
        <td mat-cell *matCellDef="let user">{{ user.name }}</td>
      </ng-container>

      <ng-container matColumnDef="actions">
        <th mat-header-cell *matHeaderCellDef>Actions</th>
        <td mat-cell *matCellDef="let user">
          <!-- Dumb component does NOT navigate directly — it emits an event -->
          <!-- The smart parent decides what "edit" means in this context -->
          <button mat-button (click)="editUser.emit(user.id)">Edit</button>
          <button mat-button color="warn" (click)="deleteUser.emit(user.id)">Delete</button>
        </td>
      </ng-container>

      <tr mat-header-row *matHeaderRowDef="columns"></tr>
      <tr mat-row *matRowDef="let row; columns: columns;"></tr>
    </mat-table>
  `
})
export class UsersTableComponent {
  // signal-based inputs — the modern replacement for @Input() decorator
  users = input.required<User[]>();
  loading = input(false);
  error = input<string | null>(null);

  // signal-based outputs — the modern replacement for @Output() + EventEmitter
  deleteUser = output<string>();
  editUser = output<string>();

  columns = ['name', 'email', 'role', 'actions'];
}
```

The `UsersTableComponent` can now be used in any context — an admin panel, a customer-facing page, a dialog, a reporting dashboard — because it has no opinions about where its data comes from or what should happen when buttons are clicked. It only knows how to display users and report interactions. This reusability is the most tangible benefit of the smart/dumb pattern.

---

## 14.2.3 — Lifting State Up

Lifting state up is the process of moving state from a child component to a parent component so that it can be shared between siblings that don't have a direct communication channel. It is the first and most important technique to apply when two components need to share state, and it should be tried before reaching for a shared service or global store.

The principle is simple: find the **lowest common ancestor** of all the components that need a piece of state, and place the state there. The ancestor passes the state down as inputs and receives updates back as outputs.

```typescript
// Before lifting: each sibling manages its own selected state
// Problem: clicking "Select All" in one sibling doesn't affect the other

// ---- After lifting: the parent owns the selection state ----
@Component({
  selector: 'app-order-page',
  template: `
    <!-- The parent owns selectedItems — it passes it to both children -->
    <app-item-selector
      [items]="availableItems"
      [selectedIds]="selectedItemIds()"
      (selectionChange)="selectedItemIds.set($event)"
    />

    <!-- The summary receives the same selectedIds to compute its display -->
    <app-order-summary
      [allItems]="availableItems"
      [selectedIds]="selectedItemIds()"
      (removeItem)="removeFromSelection($event)"
    />
  `
})
export class OrderPageComponent {
  availableItems: Item[] = this.itemsService.getAll();

  // State lifted to the parent — both children can now read and "write" it
  // (by emitting events that the parent handles)
  selectedItemIds = signal<Set<string>>(new Set());

  removeFromSelection(id: string) {
    this.selectedItemIds.update(set => {
      const newSet = new Set(set);
      newSet.delete(id);
      return newSet;
    });
  }
}
```

The key insight about lifting state is that it should only go as high as necessary. If `ItemSelector` and `OrderSummary` both live inside `OrderPageComponent` and nowhere else, the state belongs in `OrderPageComponent`. It should not go into a root-level service just because two components need it. That would be over-lifting, and it pollutes a scope that is much broader than needed.

---

## 14.2.4 — Component State Machines

A **state machine** is a formal model of computation that describes a system as being in one of a finite number of discrete states at any time, transitioning between states in response to events. Applying this model to component state is one of the most powerful techniques for eliminating an enormous class of UI bugs.

### The Problem with Boolean Flags

Consider a component that loads data from an API. A naive approach uses multiple boolean flags:

```typescript
// The problematic "boolean flag" approach
@Component({ /* ... */ })
export class DataLoaderComponent {
  isLoading = signal(false);
  hasError = signal(false);
  isEmpty = signal(false);
  isSuccess = signal(false);

  // This is already creating an impossible combination problem:
  // What happens if isLoading and isSuccess are both true simultaneously?
  // What if hasError and isEmpty are both true?
  // With 4 booleans, there are 16 possible combinations,
  // but only 4 are valid. The other 12 are bugs waiting to happen.
}
```

### The State Machine Solution

A state machine solves this by defining an explicit, finite set of states and making invalid states *impossible to represent* in your type system.

```typescript
// Define all possible states as a discriminated union type
type LoadState =
  | { status: 'idle' }                             // before any fetch
  | { status: 'loading' }                          // fetch in progress
  | { status: 'success'; data: Product[] }         // fetch succeeded
  | { status: 'error'; message: string }           // fetch failed
  | { status: 'empty' };                           // fetch succeeded but no results

@Component({
  selector: 'app-products',
  standalone: true,
  template: `
    <!-- Exhaustive state-based rendering — every state is handled explicitly -->
    @switch (state().status) {
      @case ('idle') {
        <button (click)="load()">Load Products</button>
      }
      @case ('loading') {
        <mat-progress-bar mode="indeterminate" />
        <p>Loading products...</p>
      }
      @case ('success') {
        <!-- TypeScript knows state().data exists here — it's narrowed by @switch -->
        @for (product of state().data; track product.id) {
          <app-product-card [product]="product" />
        }
      }
      @case ('empty') {
        <app-empty-state message="No products found." />
      }
      @case ('error') {
        <app-error-state [message]="state().message" (retry)="load()" />
      }
    }
  `
})
export class ProductsComponent {
  private productService = inject(ProductService);

  // Only ONE signal needed — it either is loading, has data, has an error, etc.
  // It is IMPOSSIBLE to be in "loading and success" at the same time
  state = signal<LoadState>({ status: 'idle' });

  load() {
    // Transition to loading state
    this.state.set({ status: 'loading' });

    this.productService.getAll().subscribe({
      next: (products) => {
        // Transition to success or empty based on result
        if (products.length === 0) {
          this.state.set({ status: 'empty' });
        } else {
          this.state.set({ status: 'success', data: products });
        }
      },
      error: (err) => {
        // Transition to error state with the message
        this.state.set({ status: 'error', message: err.message });
      }
    });
  }
}
```

### Using XState for Complex State Machines

For more complex state machines with guarded transitions, parallel states, or history states, the `XState` library provides a full state machine and statechart implementation that integrates with Angular:

```bash
npm install xstate @xstate/angular
```

```typescript
import { setup, assign, fromPromise } from 'xstate';
import { useMachine } from '@xstate/angular';

// Define the machine declaratively outside the component
const checkoutMachine = setup({
  types: {} as {
    context: { cartItems: CartItem[]; orderId: string | null; error: string | null };
    events:
      | { type: 'PROCEED_TO_PAYMENT' }
      | { type: 'PAYMENT_SUCCESS'; orderId: string }
      | { type: 'PAYMENT_FAILURE'; error: string }
      | { type: 'RETRY' }
      | { type: 'CANCEL' };
  },
}).createMachine({
  id: 'checkout',
  initial: 'reviewingCart',
  context: { cartItems: [], orderId: null, error: null },
  states: {
    reviewingCart: {
      on: {
        PROCEED_TO_PAYMENT: 'processingPayment'
      }
    },
    processingPayment: {
      // Invoke async operation — machine transitions based on outcome
      invoke: {
        src: fromPromise(({ input }) => paymentService.charge(input.cartItems)),
        onDone: {
          target: 'orderConfirmed',
          actions: assign({ orderId: ({ event }) => event.output })
        },
        onError: {
          target: 'paymentFailed',
          actions: assign({ error: ({ event }) => event.error.message })
        }
      }
    },
    paymentFailed: {
      on: {
        RETRY: 'processingPayment',
        CANCEL: 'reviewingCart'
      }
    },
    orderConfirmed: {
      type: 'final'
    }
  }
});

@Component({ /* ... */ })
export class CheckoutComponent {
  // useMachine provides the current state and a send function
  machine = useMachine(checkoutMachine);

  proceedToPayment() {
    this.machine.send({ type: 'PROCEED_TO_PAYMENT' });
  }
}
```

State machines are most valuable when a component has complex, conditional behavior with many possible states and transitions — wizard-style flows, checkout processes, multi-step forms, upload processes, real-time connection management, and similar scenarios. For simple on/off toggles or basic loading states, the discriminated union approach shown earlier is sufficient without the overhead of XState.

---

## Important Points and Best Practices

The smart/dumb component pattern is not about whether a component is "complex" or "simple" — it is about what a component is *responsible for*. A simple component that knows about NgRx is still a smart component. A complex component that only receives data via inputs and emits events is still a dumb component. The distinction is purely about dependencies on external state, not about size or template complexity.

When lifting state up, lift only as far as needed. The most common mistake is lifting too high — placing state in a root component or root service when only two sibling components inside one feature module need it. Over-lifting pollutes a broader scope than necessary and makes it harder to understand which parts of the app actually depend on a piece of state.

Prefer discriminated union types for state machines over boolean flags, even without a library like XState. A `type StatusState = 'idle' | 'loading' | 'success' | 'error'` union type with a single `status` signal eliminates impossible state combinations and makes TypeScript's type narrowing work naturally in your templates with `@switch` blocks.

Signal-based inputs (`input()` and `input.required()`) are the modern replacement for `@Input()` decorators in Angular 17+. They return a signal, so child components can use `computed()` to derive values from them without extra subscriptions or change detection complexity. Prefer them over `@Input()` for all new components, especially in conjunction with the smart/dumb pattern.

Use `effect()` for side effects that are driven by local state changes, but be cautious. Effects are powerful but can create hidden dependencies and ordering issues if overused. If you find yourself writing complex logic inside an `effect()`, consider whether the logic belongs in a service method or a signal update function instead. The cleanest Angular code uses `effect()` primarily for DOM manipulation, logging, and similar truly side-effectful operations — not for deriving state from state.

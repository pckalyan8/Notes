# Phase 10 — Angular Core Fundamentals
## Part 5: Built-in Directives & Control Flow
> Topic: 10.9

---

## 10.9 Built-in Directives & Control Flow

### What Are Directives?

A **directive** is a class that adds behavior to or changes the appearance of a DOM element. All Angular components are technically directives (they have a template), but Angular also has **attribute directives** that modify elements without their own template, and **structural directives** that shape or reshape the DOM by adding and removing elements.

Understanding Angular 17's new **built-in control flow syntax** is essential — it replaces the old structural directives (`*ngIf`, `*ngFor`, `*ngSwitch`) with a native template syntax that is significantly more powerful, more ergonomic, and better optimized by the Angular compiler.

---

## Modern Control Flow (Angular 17+, Strongly Preferred)

The new control flow blocks — `@if`, `@for`, `@switch`, and `@defer` — are **not directives**. They are built directly into the Angular template compiler, which gives them significant advantages: better type narrowing, the required `track` expression for performance, optional branches like `@empty` and `@loading`, and zero import overhead.

New code should always use these blocks. The old structural directives (`*ngIf`, `*ngFor`) still work but are considered legacy.

---

### `@if` — Conditional Rendering

`@if` conditionally includes or excludes a block of the template. Critically, Angular's TypeScript type checker **narrows the type** inside an `@if` block, just like TypeScript does in `.ts` files.

```typescript
@Component({ standalone: true, selector: 'app-user-display', template: `
  <!-- Basic @if: 'user' is narrowed to non-null inside the block -->
  @if (user) {
    <h2>Welcome, {{ user.name }}!</h2>
    <p>Email: {{ user.email }}</p>
  }

  <!-- @if with @else -->
  @if (isLoggedIn) {
    <button (click)="logout()">Logout</button>
  } @else {
    <button (click)="login()">Login</button>
  }

  <!-- @if with @else if chain for multiple conditions -->
  @if (status === 'loading') {
    <app-spinner />
  } @else if (status === 'error') {
    <app-error-message [message]="errorMsg" />
  } @else if (status === 'empty') {
    <p class="empty-state">No data found.</p>
  } @else {
    <app-data-table [rows]="data" />
  }
` })
export class UserDisplayComponent {
  user: { name: string; email: string } | null = null;
  isLoggedIn = false;
  status: 'loading' | 'error' | 'empty' | 'success' = 'loading';
  errorMsg = '';
  data: any[] = [];

  login() { this.isLoggedIn = true; }
  logout() { this.isLoggedIn = false; }
}
```

**Type narrowing benefit (very important):** With the old `*ngIf="user"`, the TypeScript type inside the template was still `User | null`, so you still needed `user?.name`. With `@if (user) { }`, TypeScript knows `user` is not null inside the block — you can write `user.name` directly and get full type checking.

---

### `@for` — Iteration

`@for` iterates over a collection and renders a block for each item. The `track` expression is **required** (not optional like `trackBy` was with `*ngFor`) — it tells Angular how to identify items for efficient DOM updates.

```typescript
@Component({ standalone: true, selector: 'app-user-list', template: `
  <!-- Basic @for with required 'track' expression -->
  <ul>
    @for (user of users; track user.id) {
      <li>{{ user.name }} — {{ user.email }}</li>
    }
  </ul>

  <!-- @for with the @empty block: shown when the collection is empty -->
  <div class="user-grid">
    @for (user of filteredUsers; track user.id) {
      <app-user-card [user]="user" />
    } @empty {
      <p class="no-results">No users match your search.</p>
    }
  </div>

  <!-- @for with implicit context variables -->
  <table>
    @for (item of items; track item.id; let i = $index, let isLast = $last) {
      <tr [class.last-row]="isLast">
        <td>{{ i + 1 }}</td>
        <td>{{ item.name }}</td>
      </tr>
    }
  </table>
` })
export class UserListComponent {
  users = [
    { id: 1, name: 'Alice', email: 'alice@example.com' },
    { id: 2, name: 'Bob', email: 'bob@example.com' },
    { id: 3, name: 'Carol', email: 'carol@example.com' },
  ];
  filteredUsers = [...this.users];
  items = this.users;
}
```

**Context variables available inside `@for`:**

Inside a `@for` block, you can access these built-in context variables using the `let` syntax:

- `$index` — zero-based index of the current item
- `$count` — total number of items in the collection
- `$first` — `true` for the first item
- `$last` — `true` for the last item
- `$even` — `true` for even-indexed items (0, 2, 4...)
- `$odd` — `true` for odd-indexed items (1, 3, 5...)

**The `track` expression is critical for performance.** Angular uses the track value to determine *which* DOM nodes need to be updated, created, or removed when the list changes. If `track` is set to the item's unique ID (like `user.id`), Angular can reuse DOM nodes for unchanged items even if the array is re-fetched from a server. If you track by the whole object (`track user`), Angular compares object references — useful if your objects are immutable but less helpful with re-fetched data.

---

### `@switch` — Multi-Branch Conditional

`@switch` is Angular's equivalent of a JavaScript `switch` statement for templates.

```typescript
@Component({ standalone: true, selector: 'app-status-badge', template: `
  @switch (status) {
    @case ('active') {
      <span class="badge badge-success">Active</span>
    }
    @case ('inactive') {
      <span class="badge badge-warning">Inactive</span>
    }
    @case ('banned') {
      <span class="badge badge-danger">Banned</span>
    }
    @default {
      <span class="badge badge-secondary">Unknown</span>
    }
  }
` })
export class StatusBadgeComponent {
  status: 'active' | 'inactive' | 'banned' | 'unknown' = 'active';
}
```

Unlike JavaScript's `switch`, Angular's `@switch` does **not fall through** — each `@case` is completely independent. You don't need `break` statements.

---

### `@defer` — Deferrable Views (Angular 17+)

`@defer` is one of the most powerful Angular features for performance. It **lazily loads** the components, directives, and pipes used inside the block — they are not downloaded until the trigger condition is met. This directly reduces your initial bundle size.

```typescript
@Component({ standalone: true, selector: 'app-dashboard', template: `
  <!-- Always shown immediately (this is the main content) -->
  <app-header />
  <app-hero-section />

  <!-- Deferred: the HeavyChart component is only downloaded when needed -->
  @defer (on viewport) {
    <!-- This block is only rendered when it scrolls into view -->
    <app-heavy-chart [data]="chartData" />
  } @placeholder {
    <!-- Shown immediately while the deferred content is not yet loaded -->
    <!-- Note: @placeholder is eagerly loaded (keep it lightweight!) -->
    <div class="chart-placeholder" style="height: 300px; background: #f0f0f0;">
      Chart loading area
    </div>
  } @loading (minimum 300ms) {
    <!-- Shown while the deferred content is being loaded -->
    <!-- 'minimum 300ms' prevents a flash for fast connections -->
    <app-spinner size="large" />
  } @error {
    <!-- Shown if the deferred block fails to load -->
    <p class="error">Could not load the chart. Please refresh.</p>
  }

  <!-- Defer a comments section until user interacts -->
  @defer (on interaction) {
    <app-comments-section [postId]="postId" />
  } @placeholder {
    <button>Load Comments</button>
  }

  <!-- Defer until idle (browser has free time) -->
  @defer (on idle) {
    <app-recommendations />
  }
` })
export class DashboardComponent {
  chartData = [/* ... */];
  postId = 42;
}
```

**Available `@defer` triggers:**

- `on idle` — when the browser's main thread is idle (using `requestIdleCallback`)
- `on viewport` — when the placeholder scrolls into the browser's viewport (uses `IntersectionObserver`)
- `on interaction` — when the user clicks or focuses the placeholder element
- `on hover` — when the user hovers over the placeholder element
- `on immediate` — immediately after rendering (useful for slightly delaying non-critical UI without a scroll trigger)
- `on timer(2s)` — after a specified delay
- `when condition` — when a boolean expression becomes `true`

Multiple triggers can be combined: `@defer (on idle; on viewport)` fires whichever comes first.

---

## Legacy Structural Directives

These directives still work perfectly, and you'll encounter them extensively in existing codebases. Knowing them is essential for maintenance. New code should prefer the `@if`/`@for`/`@switch` blocks above.

### `*ngIf` (Legacy)

```typescript
import { NgIf } from '@angular/common'; // or CommonModule in NgModule-based apps

@Component({ standalone: true, imports: [NgIf], selector: 'app-demo', template: `
  <!-- Show when isLoggedIn is true -->
  <div *ngIf="isLoggedIn">Welcome back!</div>

  <!-- With else clause using a template reference variable -->
  <div *ngIf="isLoggedIn; else loginPrompt">Welcome back!</div>
  <ng-template #loginPrompt>
    <p>Please log in to continue.</p>
  </ng-template>

  <!-- With 'then' and 'else' clauses -->
  <ng-container *ngIf="user; then userBlock; else loadingBlock"></ng-container>
  <ng-template #userBlock><p>{{ user?.name }}</p></ng-template>
  <ng-template #loadingBlock><app-spinner /></ng-template>
` })
export class DemoComponent {
  isLoggedIn = false;
  user: { name: string } | null = null;
}
```

### `*ngFor` (Legacy)

```typescript
import { NgFor } from '@angular/common'; // or CommonModule

@Component({ standalone: true, imports: [NgFor], selector: 'app-list-legacy', template: `
  <!-- Basic *ngFor with trackBy for performance -->
  <ul>
    <li *ngFor="let user of users; trackBy: trackByUserId; let i = index">
      {{ i + 1 }}. {{ user.name }}
    </li>
  </ul>
` })
export class ListLegacyComponent {
  users = [
    { id: 1, name: 'Alice' },
    { id: 2, name: 'Bob' }
  ];

  // trackBy function — the old equivalent of @for's track expression
  trackByUserId(index: number, user: { id: number }): number {
    return user.id;
  }
}
```

### `*ngSwitch` (Legacy)

```typescript
import { NgSwitch, NgSwitchCase, NgSwitchDefault } from '@angular/common';

@Component({ standalone: true, imports: [NgSwitch, NgSwitchCase, NgSwitchDefault], template: `
  <div [ngSwitch]="color">
    <p *ngSwitchCase="'red'">Red like fire</p>
    <p *ngSwitchCase="'blue'">Blue like the ocean</p>
    <p *ngSwitchCase="'green'">Green like nature</p>
    <p *ngSwitchDefault>Unknown color</p>
  </div>
` })
export class SwitchDemoComponent {
  color = 'blue';
}
```

---

## Attribute Directives: `NgClass` and `NgStyle`

These are still widely used because they offer convenient object-based binding for classes and styles. They are **not being deprecated** — they work alongside the new control flow.

### `NgClass`

```typescript
import { NgClass } from '@angular/common';

@Component({ standalone: true, imports: [NgClass], selector: 'app-alert', template: `
  <!-- Object syntax: { 'class-name': boolean expression } -->
  <div [ngClass]="{ 'alert': true, 'alert-success': isSuccess, 'alert-danger': isError }">
    {{ message }}
  </div>

  <!-- Array syntax: apply multiple classes -->
  <div [ngClass]="['card', 'rounded', isActive ? 'card-active' : 'card-inactive']">
    Card Content
  </div>

  <!-- String syntax (rarely needed — prefer [class]="...") -->
  <div [ngClass]="alertClass">{{ message }}</div>
` })
export class AlertComponent {
  isSuccess = true;
  isError = false;
  isActive = false;
  message = 'Operation successful!';
  alertClass = 'alert alert-success';
}
```

### `NgStyle`

```typescript
import { NgStyle } from '@angular/common';

@Component({ standalone: true, imports: [NgStyle], selector: 'app-colored-box', template: `
  <!-- Object syntax with camelCase property names -->
  <div [ngStyle]="{ 'background-color': bgColor, 'border-radius': borderRadius + 'px' }">
    Styled Box
  </div>

  <!-- Computed from a getter -->
  <div [ngStyle]="computedStyles">Dynamic Styles</div>
` })
export class ColoredBoxComponent {
  bgColor = '#e3f2fd';
  borderRadius = 8;

  get computedStyles() {
    return {
      'font-size': '14px',
      'color': this.isHighlighted ? '#d32f2f' : '#333',
    };
  }

  isHighlighted = false;
}
```

---

## `ng-container` and `ng-template`

These two elements are crucial for grouping or defining template content **without adding extra DOM elements**.

### `ng-container`

`ng-container` is a logical grouping element that is **not rendered in the DOM**. Use it when you need to apply a structural directive (or the new control flow blocks) to a group of elements without a wrapper `<div>`.

```html
<!-- Problem: wrapping in a div adds unwanted markup (e.g., breaks a table layout) -->
<!-- <div *ngIf="showFooter"> -->
<!--   <td>...</td><td>...</td> -->
<!-- </div> -->

<!-- Solution: use ng-container which renders nothing in the DOM -->
<ng-container *ngIf="showFooter">
  <td>{{ footerLabel }}</td>
  <td>{{ footerTotal | currency }}</td>
</ng-container>

<!-- Modern equivalent with @if — no need for ng-container in new code -->
@if (showFooter) {
  <td>{{ footerLabel }}</td>
  <td>{{ footerTotal | currency }}</td>
}
```

### `ng-template`

`ng-template` defines a reusable piece of template that is **not rendered by default**. It can be stamped out programmatically or referenced by directives like `*ngIf`'s `else` clause.

```typescript
@Component({ standalone: true, selector: 'app-card', template: `
  <!-- This template is only rendered when explicitly used -->
  <ng-template #loadingTpl>
    <div class="skeleton">Loading...</div>
  </ng-template>

  <ng-template #errorTpl let-message="message">
    <!-- let-message binds the 'message' context variable from the calling directive -->
    <div class="error-box">{{ message }}</div>
  </ng-template>

  <!-- Use it with ngTemplateOutlet for conditional rendering -->
  <ng-container *ngIf="isLoading; else content">
    <ng-container *ngTemplateOutlet="loadingTpl"></ng-container>
  </ng-container>

  <ng-template #content>
    <p>Actual content here</p>
  </ng-template>
` })
export class CardComponent {
  isLoading = true;
}
```

---

## Best Practices for 10.9

**Always use the new control flow blocks for new code.** The `@if`, `@for`, and `@switch` blocks are the future of Angular templating. They offer better performance through compiler optimization, required `track` expressions, type narrowing, and clearer syntax. The Angular CLI's migration command `ng g @angular/core:control-flow` can automatically convert old `*ngIf`/`*ngFor` directives to the new syntax.

**Always set a meaningful `track` expression in `@for`.** Never use `track $index` as your default — it's only correct when the list order never changes and items are never added/removed. Track by a unique, stable identifier like `item.id` or `item.uuid`. Incorrect tracking causes bugs where Angular updates the wrong DOM node, leading to stale UI.

**Use `@defer` aggressively for below-fold and non-critical content.** Any component that users don't need immediately when the page loads is a candidate for `@defer`. Features like comments sections, analytics dashboards, heavy charts, and recommendations are perfect candidates. This directly improves your Largest Contentful Paint (LCP) score.

**Keep `@placeholder` blocks lightweight.** The content inside `@placeholder` is eagerly loaded — it's what users see before the deferred content loads. Make it a simple div, skeleton, or text. Don't place heavy components there.

**Avoid `*ngFor` without `trackBy` in legacy code.** If you're maintaining code that uses `*ngFor`, always add a `trackBy` function. Without it, Angular destroys and recreates every DOM element in the list on every change — even if only one item changed.

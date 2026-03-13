# Phase 11.1 — Advanced Routing (Angular 21)

## What Is the Angular Router and Why Does It Matter?

The Angular Router is the official navigation library built into the framework. In a Single Page Application (SPA), the browser never fully reloads between pages — the Router intercepts URL changes and renders the correct component without a full page refresh. As applications grow, routing moves well beyond basic `path → component` mappings. You need lazy loading, guards, resolvers, typed parameters, and seamless view transitions. This section covers all of that for Angular 21.

---

## Lazy Loading Routes

Lazy loading is one of the most impactful performance optimizations available in Angular. Instead of bundling every component into the initial JavaScript payload, lazy loading splits your app into separate chunks that are only downloaded when the user navigates to that feature.

### Using `loadComponent` (Standalone — Preferred)

```typescript
// app.routes.ts
import { Routes } from '@angular/router';

export const routes: Routes = [
  {
    path: 'dashboard',
    // Angular fetches this chunk only when the user visits /dashboard
    loadComponent: () =>
      import('./dashboard/dashboard.component').then(m => m.DashboardComponent),
  },
  {
    path: 'settings',
    loadComponent: () =>
      import('./settings/settings.component').then(m => m.SettingsComponent),
  },
  {
    path: '',
    redirectTo: 'dashboard',
    pathMatch: 'full',
  },
];
```

### Using `loadChildren` (For Feature Route Groups)

When a feature has its own set of child routes, use `loadChildren` to lazy load the entire route subtree.

```typescript
// app.routes.ts
export const routes: Routes = [
  {
    path: 'admin',
    // The entire admin feature — its routes, components, services — is one lazy chunk
    loadChildren: () =>
      import('./admin/admin.routes').then(m => m.ADMIN_ROUTES),
  },
];

// admin/admin.routes.ts
import { Routes } from '@angular/router';
export const ADMIN_ROUTES: Routes = [
  { path: '', loadComponent: () => import('./admin-dashboard.component').then(m => m.AdminDashboardComponent) },
  { path: 'users', loadComponent: () => import('./users/users.component').then(m => m.UsersComponent) },
];
```

> **Important:** `loadComponent` is for a single standalone component. `loadChildren` is for an array of routes (a feature module's route config). Prefer `loadComponent` for standalone architectures and `loadChildren` when grouping a feature's internal navigation.

---

## Route Guards

Guards are functions (or classes) that decide whether a navigation can proceed. Angular 15+ introduced **functional guards**, which are plain functions rather than classes implementing an interface. Functional guards are preferred in Angular 21 because they are simpler and tree-shakeable.

### `canActivate` — Protecting Route Access

```typescript
// auth.guard.ts
import { inject } from '@angular/core';
import { CanActivateFn, Router } from '@angular/router';
import { AuthService } from './auth.service';

export const authGuard: CanActivateFn = (route, state) => {
  const auth = inject(AuthService);   // inject() works in functional guards
  const router = inject(Router);

  if (auth.isLoggedIn()) {
    return true;  // Allow navigation
  }

  // Redirect to login, preserving the intended URL
  return router.createUrlTree(['/login'], {
    queryParams: { returnUrl: state.url },
  });
};

// Using the guard in routes:
export const routes: Routes = [
  {
    path: 'profile',
    loadComponent: () => import('./profile/profile.component').then(m => m.ProfileComponent),
    canActivate: [authGuard],  // Array of guard functions
  },
];
```

### `canDeactivate` — Preventing Unsaved Data Loss

```typescript
// unsaved-changes.guard.ts
import { CanDeactivateFn } from '@angular/router';

// Generic interface for components that have unsaved state
export interface HasUnsavedChanges {
  hasUnsavedChanges(): boolean;
}

export const unsavedChangesGuard: CanDeactivateFn<HasUnsavedChanges> = (component) => {
  if (component.hasUnsavedChanges()) {
    return window.confirm('You have unsaved changes. Leave anyway?');
  }
  return true;
};

// In your component:
@Component({ selector: 'app-edit-form', ... })
export class EditFormComponent implements HasUnsavedChanges {
  private formDirty = false;

  hasUnsavedChanges(): boolean {
    return this.formDirty;
  }
}
```

### `canMatch` — Conditionally Matching Routes

`canMatch` is different from `canActivate`. While `canActivate` blocks navigation and shows a 403/redirect, `canMatch` prevents the route from being matched at all — Angular continues looking at subsequent route definitions. This is useful for feature flags or role-based routing.

```typescript
export const adminGuard: CanMatchFn = () => {
  const auth = inject(AuthService);
  return auth.hasRole('admin');
  // If false, Angular skips this route and tries the next one
};

export const routes: Routes = [
  {
    path: 'dashboard',
    canMatch: [adminGuard],
    loadComponent: () => import('./admin-dashboard.component').then(m => m.AdminDashboardComponent),
  },
  {
    path: 'dashboard',  // Same path, different component for non-admins
    loadComponent: () => import('./user-dashboard.component').then(m => m.UserDashboardComponent),
  },
];
```

---

## Route Resolvers

Resolvers pre-fetch data before the component activates, so the component can render immediately with data available rather than showing loading spinners.

```typescript
// product.resolver.ts
import { inject } from '@angular/core';
import { ResolveFn } from '@angular/router';
import { ProductService } from './product.service';
import { Product } from './product.model';

export const productResolver: ResolveFn<Product> = (route, state) => {
  const productService = inject(ProductService);
  const id = route.paramMap.get('id')!;
  return productService.getProduct(id);  // Returns Observable<Product> or Promise<Product>
};

// In routes:
export const routes: Routes = [
  {
    path: 'products/:id',
    loadComponent: () => import('./product-detail.component').then(m => m.ProductDetailComponent),
    resolve: { product: productResolver },  // 'product' is the key
  },
];

// In the component, access resolved data:
@Component({ ... })
export class ProductDetailComponent implements OnInit {
  private route = inject(ActivatedRoute);
  product!: Product;

  ngOnInit() {
    // The resolved data is available synchronously here
    this.product = this.route.snapshot.data['product'];
  }
}
```

> **Best Practice:** Resolvers are great for critical data, but avoid them for non-essential or slow data. If data fetching might take seconds, users see a frozen URL bar instead of a loading indicator — which feels broken. For slow data, render the component with a skeleton and fetch inside `ngOnInit`.

---

## Route Parameters — Typed (Angular 20+)

Angular 20 introduced typed route parameters through `withComponentInputBinding()`. This automatically maps route parameters, query parameters, and resolved data to component `input()` signals — no `ActivatedRoute` injection needed.

### Setting Up Input Binding

```typescript
// main.ts or app.config.ts
import { provideRouter, withComponentInputBinding } from '@angular/router';

export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(routes, withComponentInputBinding()), // Enable automatic binding
  ],
};
```

### Using Input Binding in Components

```typescript
// product-detail.component.ts
import { Component, input, OnInit } from '@angular/core';

@Component({
  selector: 'app-product-detail',
  template: `<h1>Product: {{ productId() }}</h1>`,
})
export class ProductDetailComponent {
  // Angular automatically binds the :id route param to this input signal
  productId = input<string>('');

  // Query params (?page=2) also get bound automatically
  page = input<string>('1');

  // Resolved data also binds by key name
  product = input<Product | null>(null);
}

// Route definition:
export const routes: Routes = [
  {
    path: 'products/:productId',  // Must match the input() signal name
    resolve: { product: productResolver },
    loadComponent: () => import('./product-detail.component').then(m => m.ProductDetailComponent),
  },
];
```

This approach eliminates the boilerplate of `ActivatedRoute` subscriptions and keeps your component clean and testable.

---

## Nested (Child) Routes

Child routes render inside a parent component's `<router-outlet>`, creating hierarchical navigation.

```typescript
export const routes: Routes = [
  {
    path: 'users',
    loadComponent: () => import('./users/users-layout.component').then(m => m.UsersLayoutComponent),
    children: [
      { path: '', loadComponent: () => import('./users/users-list.component').then(m => m.UsersListComponent) },
      { path: ':id', loadComponent: () => import('./users/user-detail.component').then(m => m.UserDetailComponent) },
      { path: ':id/edit', loadComponent: () => import('./users/user-edit.component').then(m => m.UserEditComponent) },
    ],
  },
];

// users-layout.component.ts — the parent renders child routes here
@Component({
  template: `
    <nav>
      <a routerLink="/users">All Users</a>
    </nav>
    <router-outlet></router-outlet>  <!-- Child components render here -->
  `,
})
export class UsersLayoutComponent {}
```

---

## Preloading Strategies

By default, lazy chunks are only loaded on demand. Preloading strategies allow Angular to load lazy chunks in the background after the initial page loads, so subsequent navigations feel instant.

```typescript
import { PreloadAllModules, provideRouter, withPreloading } from '@angular/router';

// Strategy 1: Preload everything immediately after initial load
provideRouter(routes, withPreloading(PreloadAllModules))

// Strategy 2: Custom preloading — only preload routes with data: { preload: true }
import { PreloadingStrategy, Route } from '@angular/router';
import { Observable, of } from 'rxjs';
import { Injectable } from '@angular/core';

@Injectable({ providedIn: 'root' })
export class SelectivePreloadingStrategy implements PreloadingStrategy {
  preload(route: Route, load: () => Observable<any>): Observable<any> {
    return route.data?.['preload'] ? load() : of(null);
  }
}

// Usage:
export const routes: Routes = [
  {
    path: 'products',
    data: { preload: true },  // This route will be preloaded
    loadComponent: () => import('./products.component').then(m => m.ProductsComponent),
  },
];
```

---

## `withViewTransitions()` — Animated Route Navigation

Angular integrates with the browser's native View Transitions API to animate route changes. All you need is one configuration flag.

```typescript
import { provideRouter, withViewTransitions } from '@angular/router';

export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(routes, withViewTransitions()),
  ],
};
```

Angular wraps each navigation inside `document.startViewTransition()`, and the browser applies a default cross-fade. You can customise it with CSS:

```css
/* Define a named view transition for a specific element */
.product-card {
  view-transition-name: product-card;
}

/* The browser provides these pseudo-elements automatically */
::view-transition-old(product-card) {
  animation: slide-out 0.3s ease;
}

::view-transition-new(product-card) {
  animation: slide-in 0.3s ease;
}
```

Always respect user preference for reduced motion:

```css
@media (prefers-reduced-motion: reduce) {
  ::view-transition-old(*),
  ::view-transition-new(*) {
    animation: none;
  }
}
```

---

## Signal-Based Router (Angular 21 Preview)

Angular 21 introduces `injectRouterState()` which exposes router state as signals, enabling reactive route-driven UIs without subscribing to observables.

```typescript
import { injectRouterState } from '@angular/router';

@Component({ ... })
export class NavComponent {
  private routerState = injectRouterState();

  // These are computed signals that update automatically on navigation
  currentUrl = this.routerState.url;
  isNavigating = this.routerState.isNavigating;
}
```

---

## Dynamic Redirects with `redirectTo` as Function

Angular 21 allows `redirectTo` to be a function, enabling conditional redirects based on the current route or application state.

```typescript
export const routes: Routes = [
  {
    path: '',
    redirectTo: ({ queryParams }) => {
      // Redirect based on query parameters
      return queryParams['preview'] === 'true' ? '/preview' : '/home';
    },
    pathMatch: 'full',
  },
];
```

---

## `TitleStrategy` — Dynamic Page Titles

Setting meaningful page titles improves both SEO and accessibility (screen readers announce the title on navigation).

```typescript
import { Injectable } from '@angular/core';
import { RouterStateSnapshot, TitleStrategy } from '@angular/router';
import { Title } from '@angular/platform-browser';

@Injectable({ providedIn: 'root' })
export class AppTitleStrategy extends TitleStrategy {
  constructor(private title: Title) {
    super();
  }

  override updateTitle(snapshot: RouterStateSnapshot): void {
    const routeTitle = this.buildTitle(snapshot);
    this.title.setTitle(routeTitle ? `${routeTitle} | MyApp` : 'MyApp');
  }
}

// In routes, add a title:
export const routes: Routes = [
  {
    path: 'dashboard',
    title: 'Dashboard',    // Static title
    loadComponent: () => import('./dashboard.component').then(m => m.DashboardComponent),
  },
];

// Register in app config:
export const appConfig: ApplicationConfig = {
  providers: [
    { provide: TitleStrategy, useClass: AppTitleStrategy },
    provideRouter(routes),
  ],
};
```

---

## `withNavigationErrorHandler()` — Centralised Error Handling

Rather than handling navigation errors in each guard or resolver, Angular 21 provides a central hook.

```typescript
import { provideRouter, withNavigationErrorHandler } from '@angular/router';

provideRouter(
  routes,
  withNavigationErrorHandler((error) => {
    console.error('Navigation failed:', error);
    // You can inject services here using inject()
    const router = inject(Router);
    router.navigate(['/error'], { state: { message: error.message } });
  })
)
```

---

## `RouterLink` and Active Styling

```html
<!-- Basic navigation -->
<a routerLink="/dashboard">Dashboard</a>

<!-- With parameters -->
<a [routerLink]="['/users', user.id]">View User</a>

<!-- Active class — added when URL matches exactly -->
<a routerLink="/settings" routerLinkActive="active-link" [routerLinkActiveOptions]="{ exact: true }">
  Settings
</a>

<!-- Programmatic navigation -->
```

```typescript
@Component({ ... })
export class NavComponent {
  private router = inject(Router);

  goToProfile(userId: string) {
    this.router.navigate(['/users', userId]);
  }

  goWithQuery() {
    this.router.navigate(['/products'], { queryParams: { page: 2, sort: 'price' } });
  }
}
```

---

## Key Best Practices

Always use `loadComponent` for every route in a standalone Angular application — never eagerly load route components unless they are part of the initial critical path. Apply guards at the route level rather than inside components; guards are declarative, reusable, and testable in isolation. When you have multiple guards on one route, they run in order and the first `false` short-circuits the rest. Keep resolvers fast — pre-fetching is valuable only if the data arrives quickly. Enable `withComponentInputBinding()` on all new projects to simplify component code and improve testability. Always implement `prefers-reduced-motion` when using view transitions to avoid causing distress to motion-sensitive users.

```typescript
// Minimal, production-ready router setup for Angular 21
export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(
      routes,
      withComponentInputBinding(),   // Route params → signal inputs
      withViewTransitions(),          // Native animated transitions
      withPreloading(SelectivePreloadingStrategy), // Smart preloading
      withNavigationErrorHandler(err => inject(ErrorService).handle(err))
    ),
  ],
};
```

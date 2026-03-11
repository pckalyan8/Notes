# Phase 10 — Angular Core Fundamentals
## Part 8: Routing (Angular Router)
> Topic: 10.12

---

## 10.12 Routing (Angular Router)

### What Is Client-Side Routing?

In a traditional multi-page application, every link click sends a request to the server, which responds with a whole new HTML document. The browser navigates away from the current page, re-downloads assets, and re-renders everything from scratch.

Angular applications are **Single Page Applications (SPAs)**. There is exactly one HTML file delivered by the server (`index.html`). After the initial load, **Angular's Router intercepts all navigation** — it prevents the browser from making a full-page request, updates the URL in the address bar, and swaps the displayed component entirely in JavaScript. The user gets the feel of navigating between pages without ever leaving the single HTML document.

This delivers a much faster, app-like experience. The tradeoff is that you need a router to manage which component to show based on the current URL.

---

### Setting Up the Router

In modern standalone Angular, you configure routing in `app.config.ts` using `provideRouter()`:

```typescript
// app.routes.ts — Define the route configuration
import { Routes } from '@angular/router';
import { HomeComponent } from './home/home.component';
import { AboutComponent } from './about/about.component';
import { NotFoundComponent } from './not-found/not-found.component';

export const routes: Routes = [
  // The simplest route: a path maps to a component.
  { path: '', component: HomeComponent },
  { path: 'about', component: AboutComponent },

  // Wildcard route: matches any URL that no other route matches.
  // Must always be the LAST route — Angular matches routes in order.
  { path: '**', component: NotFoundComponent }
];
```

```typescript
// app.config.ts — Register the routes with the application
import { ApplicationConfig } from '@angular/core';
import { provideRouter } from '@angular/router';
import { routes } from './app.routes';

export const appConfig: ApplicationConfig = {
  providers: [
    // provideRouter() sets up the router with your route configuration.
    // Additional features (like withViewTransitions) are passed here too.
    provideRouter(routes)
  ]
};
```

For NgModule-based apps (legacy), you'd use `RouterModule.forRoot(routes)` in `AppModule`'s imports array instead.

---

### `RouterOutlet` — Where Components Are Rendered

The `RouterOutlet` directive is a **placeholder in the template** that tells Angular where to render the component associated with the active route. You place it in your root `AppComponent`:

```typescript
// app.component.ts
import { Component } from '@angular/core';
import { RouterOutlet } from '@angular/router';
import { NavbarComponent } from './shared/navbar.component';

@Component({
  selector: 'app-root',
  standalone: true,
  imports: [RouterOutlet, NavbarComponent],
  template: `
    <!-- The NavbarComponent is always visible -->
    <app-navbar />

    <!-- RouterOutlet is where Angular renders the active route's component -->
    <!-- As you navigate, the component here swaps out while the navbar stays -->
    <main>
      <router-outlet />
    </main>
  `
})
export class AppComponent { }
```

When the user navigates to `/about`, Angular replaces whatever is in the `<router-outlet>` with the `AboutComponent`. When they go to `/`, it shows `HomeComponent`. The `AppComponent`, `NavbarComponent`, and the rest of the shell stay untouched.

---

### `RouterLink` — Navigating Between Routes

Never use plain `<a href="/about">` in Angular. A regular `href` triggers a full browser navigation, defeating the purpose of the SPA. Instead, use `RouterLink`:

```typescript
import { RouterLink, RouterLinkActive } from '@angular/router';

@Component({
  selector: 'app-navbar',
  standalone: true,
  imports: [RouterLink, RouterLinkActive],
  template: `
    <nav>
      <!-- RouterLink prevents a full page reload and lets Angular handle navigation -->
      <a routerLink="/">Home</a>
      <a routerLink="/about">About</a>
      <a routerLink="/contact">Contact</a>

      <!-- RouterLinkActive automatically adds the specified CSS class(es)
           when the link's route is the currently active route.
           Perfect for highlighting the current page in navigation. -->
      <a routerLink="/dashboard"
         routerLinkActive="active-nav-link">
        Dashboard
      </a>

      <!-- routerLinkActiveOptions: exact=true ensures the class is only added
           when the path is an EXACT match.
           Without this, routerLink="/" would be "active" on ALL routes
           because every URL starts with "/". -->
      <a routerLink="/"
         routerLinkActive="active-nav-link"
         [routerLinkActiveOptions]="{ exact: true }">
        Home
      </a>
    </nav>
  `
})
export class NavbarComponent { }
```

**The array syntax for dynamic routes.** When navigating to a route with parameters, use an array:

```html
<!-- Navigates to /users/42 -->
<a [routerLink]="['/users', user.id]">View {{ user.name }}</a>

<!-- Navigates to /users/42/posts/7 (nested params) -->
<a [routerLink]="['/users', user.id, 'posts', post.id]">View Post</a>

<!-- With query parameters and fragment -->
<a
  [routerLink]="['/search']"
  [queryParams]="{ q: searchTerm, page: 1 }"
  fragment="results">
  Search
</a>
<!-- Navigates to /search?q=angular&page=1#results -->
```

---

### Route Parameters — Dynamic Segments

Route parameters let you embed dynamic values directly in the URL path, making each URL unique:

```typescript
// app.routes.ts — Define a route with a parameter
export const routes: Routes = [
  { path: 'users', component: UserListComponent },
  // ':id' is a route parameter — it matches any value in that URL segment.
  // Angular captures the value and makes it available via ActivatedRoute.
  { path: 'users/:id', component: UserDetailComponent },
  // Multiple parameters in one route
  { path: 'teams/:teamId/members/:memberId', component: MemberDetailComponent }
];
```

```typescript
// user-detail.component.ts — Reading route parameters
import { Component, OnInit, inject } from '@angular/core';
import { ActivatedRoute } from '@angular/router';
import { UserService } from '../services/user.service';
import { User } from '../models/user.model';
import { Observable } from 'rxjs';
import { switchMap } from 'rxjs/operators';

@Component({
  selector: 'app-user-detail',
  standalone: true,
  template: `
    @if (user$ | async; as user) {
      <h1>{{ user.name }}</h1>
      <p>{{ user.email }}</p>
    }
  `
})
export class UserDetailComponent implements OnInit {
  private route = inject(ActivatedRoute);
  private userService = inject(UserService);

  // Observable approach — handles navigation from one user directly to another
  // (e.g., /users/1 → /users/2) without destroying/recreating the component
  user$!: Observable<User>;

  ngOnInit(): void {
    this.user$ = this.route.paramMap.pipe(
      // switchMap cancels the previous HTTP request if the param changes
      // before the previous request completes — prevents race conditions
      switchMap(params => {
        const id = Number(params.get('id'));
        return this.userService.getUserById(id);
      })
    );
  }
}
```

**The snapshot approach** is simpler when you know the component will always be destroyed and re-created on navigation (route params won't change *while the component is mounted*):

```typescript
ngOnInit(): void {
  // snapshot gives the current state of the route at the moment of initialization.
  // This works fine when users can only reach this page fresh each time,
  // but will NOT update if the user navigates from /users/1 to /users/2
  // without the component being destroyed (e.g., if using the same route).
  const id = Number(this.route.snapshot.paramMap.get('id'));
  this.userService.getUserById(id).subscribe(user => this.user = user);
}
```

---

### Query Parameters — Optional Filters and State

While route parameters are part of the path (they change the URL segment), query parameters are optional key-value pairs appended after `?`. They are typically used for filters, search terms, sorting, and pagination — state that doesn't change which component is shown, just what it displays.

```typescript
// Reading query parameters
@Component({
  selector: 'app-product-list',
  standalone: true,
  template: `...`
})
export class ProductListComponent implements OnInit {
  private route = inject(ActivatedRoute);

  ngOnInit(): void {
    // Subscribe to query params as an Observable
    this.route.queryParamMap.subscribe(params => {
      const searchTerm = params.get('search') ?? '';
      const page = Number(params.get('page') ?? 1);
      const sortBy = params.get('sort') ?? 'name';

      // Use these to filter/sort your data
      this.loadProducts({ searchTerm, page, sortBy });
    });
  }

  loadProducts(filters: any) { /* ... */ }
}
```

```typescript
// Navigating with query params programmatically
import { Router } from '@angular/router';

@Component({ /* ... */ })
export class SearchComponent {
  private router = inject(Router);

  onSearch(term: string): void {
    // Navigate to the same route with updated query params.
    // This updates the URL without changing the component — perfect for filters.
    this.router.navigate(['/products'], {
      queryParams: { search: term, page: 1 },
      // 'merge' keeps other existing query params and only changes specified ones.
      // 'preserve' keeps all existing params unchanged.
      queryParamsHandling: 'merge'
    });
  }
}
```

---

### Programmatic Navigation with `Router`

Beyond `RouterLink`, you can navigate imperatively from TypeScript code — after a form submission, upon receiving a server response, or based on business logic:

```typescript
import { Component, inject } from '@angular/core';
import { Router } from '@angular/router';
import { AuthService } from '../services/auth.service';

@Component({
  selector: 'app-login',
  standalone: true,
  template: `...`
})
export class LoginComponent {
  private router = inject(Router);
  private authService = inject(AuthService);

  onLogin(credentials: { email: string; password: string }): void {
    this.authService.login(credentials).subscribe({
      next: () => {
        // After successful login, navigate to the dashboard
        this.router.navigate(['/dashboard']);
      },
      error: () => {
        // Handle login error
      }
    });
  }

  onLoginWithExtras(): void {
    this.router.navigate(['/dashboard'], {
      queryParams: { welcome: 'true' },   // Add query params
      fragment: 'getting-started',         // Add URL fragment (#getting-started)
      replaceUrl: true                     // Replace current history entry instead of adding
    });
  }

  goBack(): void {
    // navigateByUrl accepts a complete URL string including query params
    this.router.navigateByUrl('/home');
  }
}
```

---

### Lazy Loading — Critical for Performance

Loading all routes eagerly (at application startup) means users download the JavaScript for every page even if they never visit most of them. **Lazy loading** defers the download of a route's code until the user actually navigates to it.

```typescript
// app.routes.ts
export const routes: Routes = [
  { path: '', component: HomeComponent },  // Eagerly loaded — always needed

  {
    path: 'dashboard',
    // loadComponent: lazily loads a standalone component.
    // The import() is a dynamic import — it creates a separate JavaScript chunk.
    // The code for DashboardComponent is NOT included in the main bundle.
    loadComponent: () =>
      import('./dashboard/dashboard.component').then(m => m.DashboardComponent)
  },

  {
    path: 'admin',
    // loadChildren: lazily loads a whole set of child routes.
    // Perfect for feature areas with multiple sub-pages.
    loadChildren: () =>
      import('./admin/admin.routes').then(m => m.adminRoutes)
  }
];
```

```typescript
// admin/admin.routes.ts — A lazily loaded route group
import { Routes } from '@angular/router';

export const adminRoutes: Routes = [
  { path: '', component: AdminDashboardComponent },
  { path: 'users', component: AdminUsersComponent },
  { path: 'settings', component: AdminSettingsComponent }
];
```

When you navigate to `/admin`, Angular downloads the admin bundle, registers the admin routes, and renders `AdminDashboardComponent`. The user never downloaded that code before clicking the admin link. This is one of the most impactful performance techniques available in Angular.

---

### Nested (Child) Routes

Child routes let you compose layouts — a parent component provides a layout shell (sidebar, tabs, header), and child routes swap out content within that shell via a nested `<router-outlet>`.

```typescript
// app.routes.ts
export const routes: Routes = [
  {
    path: 'settings',
    component: SettingsLayoutComponent,  // This component has its own <router-outlet>
    children: [
      // Navigating to /settings shows SettingsLayoutComponent + SettingsProfileComponent
      { path: '', redirectTo: 'profile', pathMatch: 'full' },
      { path: 'profile', component: SettingsProfileComponent },
      { path: 'security', component: SettingsSecurityComponent },
      { path: 'notifications', component: SettingsNotificationsComponent }
    ]
  }
];
```

```typescript
// settings-layout.component.ts — The parent component with a nested outlet
@Component({
  selector: 'app-settings-layout',
  standalone: true,
  imports: [RouterOutlet, RouterLink, RouterLinkActive],
  template: `
    <div class="settings-page">
      <nav class="settings-sidebar">
        <!-- These links navigate within the settings section -->
        <a routerLink="profile" routerLinkActive="active">Profile</a>
        <a routerLink="security" routerLinkActive="active">Security</a>
        <a routerLink="notifications" routerLinkActive="active">Notifications</a>
      </nav>
      <div class="settings-content">
        <!-- Child route components render here, not in the root outlet -->
        <router-outlet />
      </div>
    </div>
  `
})
export class SettingsLayoutComponent { }
```

---

### `redirectTo` and `pathMatch`

`redirectTo` is how you handle default routes and URL aliases:

```typescript
export const routes: Routes = [
  // Redirect the empty path to '/home'. 
  // pathMatch: 'full' means the ENTIRE URL must be empty, not just the prefix.
  // Always use pathMatch: 'full' with empty-path redirects.
  { path: '', redirectTo: '/home', pathMatch: 'full' },
  { path: 'home', component: HomeComponent },

  // Redirect an old URL to a new one (handles renamed routes)
  { path: 'old-products', redirectTo: '/catalog', pathMatch: 'full' },
  { path: 'catalog', component: CatalogComponent },
];
```

---

### Reading Route Data with `ActivatedRoute`

`ActivatedRoute` is the service that gives you access to everything about the current route: parameters, query params, static data, and more. It has two flavors of access: **snapshot** (current value once) and **Observable** (streams of changes over time):

```typescript
@Component({ /* ... */ })
export class ArticleComponent implements OnInit {
  private route = inject(ActivatedRoute);

  ngOnInit(): void {
    // --- Snapshot: read once at initialization ---
    const snapshot = this.route.snapshot;
    const articleId = snapshot.paramMap.get('id');       // Route param
    const queryPage = snapshot.queryParamMap.get('page'); // Query param
    const staticData = snapshot.data['title'];            // Static route data

    // --- Observable: react to changes over time ---
    // Use Observable when the component can be reused for different routes
    // (e.g., navigating from /articles/1 directly to /articles/2)
    this.route.paramMap.subscribe(params => {
      const id = params.get('id');
      this.loadArticle(id!);
    });

    // For query params
    this.route.queryParamMap.subscribe(params => {
      const searchQuery = params.get('q');
      // Re-filter results...
    });
  }

  loadArticle(id: string) { /* ... */ }
}
```

---

### Router Events — Listening to Navigation Lifecycle

The `Router` service emits a stream of events as navigation progresses through its lifecycle. This is useful for showing global loading indicators, tracking page views in analytics, or debugging:

```typescript
import { Component, inject, OnInit } from '@angular/core';
import { Router, NavigationStart, NavigationEnd, NavigationError } from '@angular/router';
import { filter } from 'rxjs/operators';

@Component({ selector: 'app-root', /* ... */ })
export class AppComponent implements OnInit {
  private router = inject(Router);
  isNavigating = false;

  ngOnInit(): void {
    this.router.events.subscribe(event => {
      // NavigationStart fires when Angular begins processing a navigation request
      if (event instanceof NavigationStart) {
        this.isNavigating = true;
      }

      // NavigationEnd fires when Angular has committed a navigation
      if (event instanceof NavigationEnd) {
        this.isNavigating = false;

        // Track page views for analytics — the correct URL is available here
        this.trackPageView(event.urlAfterRedirects);
      }

      // NavigationError fires if navigation fails (route guard rejected, etc.)
      if (event instanceof NavigationError) {
        this.isNavigating = false;
        console.error('Navigation failed:', event.error);
      }
    });
  }

  trackPageView(url: string) { /* Send to analytics */ }
}
```

---

## Best Practices for 10.12

**Lazy-load everything except the home page.** The initial bundle should contain only what's needed for the first paint. Every route beyond the landing page should use `loadComponent` or `loadChildren`. This is not premature optimization — it is the correct default architecture and directly improves your app's Largest Contentful Paint (LCP) score.

**Always provide a wildcard `**` route.** Users will type URLs directly, share links, and bookmark pages. A wildcard route ensures they see a meaningful 404 page instead of a blank screen when Angular can't match their URL.

**Use `pathMatch: 'full'` for empty-path redirects.** Omitting this means the redirect fires for ANY route (since all routes have an empty prefix), creating redirect loops.

**Subscribe to `ActivatedRoute` Observables, not just the snapshot, when components can be reused.** If a user can navigate from `/users/1` to `/users/2` without the component being destroyed (because the route is the same, only the parameter changes), the `snapshot` approach will show stale data. The Observable approach handles this correctly.

**Use `withViewTransitions()` in `provideRouter()` for smooth page transitions.** Angular 17+ integrates the browser's native View Transitions API. Adding `provideRouter(routes, withViewTransitions())` animates route changes with a cross-fade by default, with zero additional CSS required.

**Organize routes to mirror your folder structure.** If your files live in `src/app/features/admin/`, your routes should live under `/admin`. This makes navigation through the codebase intuitive for the entire team.

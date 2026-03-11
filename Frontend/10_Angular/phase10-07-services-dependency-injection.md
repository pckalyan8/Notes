# Phase 10 — Angular Core Fundamentals
## Part 7: Services & Dependency Injection (DI)
> Topic: 10.11

---

## 10.11 Services & Dependency Injection (DI)

### Why Do We Need Services?

Angular components handle the user interface — they render HTML, respond to events, and display data. But where should the *logic* live? Where do you put HTTP calls, data caching, authentication tokens, business rules, or state shared across multiple components?

The answer is **services**. A service is a TypeScript class focused on a specific responsibility — fetching users, managing the current session, logging errors, etc. Components stay thin (display + event handling), and services carry the weight of application logic.

This separation follows the **Single Responsibility Principle** and makes code dramatically easier to test, because you can test a service's logic without instantiating any component at all.

**Dependency Injection (DI)** is the mechanism that connects services to the components (and other services) that need them. Instead of a component creating its own service instances with `new UserService()`, Angular's DI system creates and provides those instances automatically. This means:
- You get the same singleton instance everywhere (shared state)
- Components don't need to know *how* to construct their dependencies
- Testing is easy — you can substitute mock services in tests

---

### The `@Injectable` Decorator

Every class that will be provided by Angular's DI system must be decorated with `@Injectable`. This decoration signals to the Angular compiler that the class participates in dependency injection.

```typescript
// services/user.service.ts
import { Injectable, inject } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable } from 'rxjs';
import { User } from '../models/user.model';

// 'providedIn: root' is the most important part of this decorator.
// It registers the service in the ROOT injector, making it:
//   1. A singleton — only one instance exists for the entire application
//   2. Tree-shakeable — if no component injects it, it's removed from the bundle
//   3. Available everywhere — no need to add it to any module's providers array
@Injectable({
  providedIn: 'root'
})
export class UserService {
  // Services can inject other services using inject() or constructor injection.
  // inject() is the modern preferred approach.
  private http = inject(HttpClient);
  private readonly API_URL = 'https://api.example.com/users';

  getUsers(): Observable<User[]> {
    return this.http.get<User[]>(this.API_URL);
  }

  getUserById(id: number): Observable<User> {
    return this.http.get<User>(`${this.API_URL}/${id}`);
  }

  createUser(user: Partial<User>): Observable<User> {
    return this.http.post<User>(this.API_URL, user);
  }

  updateUser(id: number, changes: Partial<User>): Observable<User> {
    return this.http.patch<User>(`${this.API_URL}/${id}`, changes);
  }

  deleteUser(id: number): Observable<void> {
    return this.http.delete<void>(`${this.API_URL}/${id}`);
  }
}
```

---

### The Two Ways to Inject: Constructor vs `inject()`

Angular supports two syntaxes for injecting dependencies. The modern `inject()` function is preferred for new code.

**Constructor injection (classic, still valid):**
```typescript
@Injectable({ providedIn: 'root' })
export class AuthService {
  // Angular reads the constructor parameter types via TypeScript metadata
  // and automatically provides the correct instances.
  constructor(
    private http: HttpClient,
    private router: Router,
    private userService: UserService
  ) {}
}
```

**`inject()` function (modern, preferred):**
```typescript
import { Injectable, inject } from '@angular/core';

@Injectable({ providedIn: 'root' })
export class AuthService {
  // inject() can be called at class field initialization time.
  // This works in class fields (properties), constructors, and factory functions.
  private http = inject(HttpClient);
  private router = inject(Router);
  private userService = inject(UserService);

  // inject() can also be used in non-class contexts like standalone functions,
  // which makes it ideal for functional guards and resolvers.
}
```

The `inject()` function approach is cleaner, avoids long constructor signatures, and works in **functional guards**, **functional interceptors**, and **factory providers** where constructor injection cannot be used.

---

### Understanding Angular's Hierarchical Injector Tree

This is one of the most important conceptual models in Angular. Angular doesn't have just one DI container — it has a **tree of injectors** that mirrors the component tree.

```
Platform Injector (singleton for the entire browser tab)
    ↓
Root Injector (created by bootstrapApplication / AppModule)
    ↓
Environment Injector (created for lazy-loaded routes)
    ↓
Component Injector (created per component with 'providers' array)
    ↓
Child Component Injector
```

When a component requests a dependency, Angular walks **up** this tree until it finds a provider. The first injector that has a provider for the requested token wins.

```typescript
// This service is in the ROOT injector — shared singleton everywhere
@Injectable({ providedIn: 'root' })
export class SharedDataService { /* ... */ }

// This component creates its OWN injector scope with a COMPONENT-LEVEL instance
@Component({
  selector: 'app-form',
  standalone: true,
  // 'providers' here creates a new instance of FormStateService
  // scoped ONLY to this component and its children.
  // Each <app-form> element in the DOM gets its OWN FormStateService instance.
  providers: [FormStateService]
})
export class FormComponent { }
```

This hierarchical model is powerful because you can choose the scope of your service:
- **Application-wide singleton:** use `providedIn: 'root'`
- **Per-lazy-loaded-route instance:** provide in the route's `providers` array
- **Per-component instance:** provide in the component's `providers` array

---

### The `providedIn` Options

`providedIn` on `@Injectable` controls which injector the service is registered with:

```typescript
// Registered in the ROOT injector. Singleton. Tree-shakeable. The standard choice.
@Injectable({ providedIn: 'root' })
export class GlobalService { }

// Registered in the PLATFORM injector. 
// Shared even across multiple Angular applications on the same page.
// Useful only in very specific multi-app scenarios.
@Injectable({ providedIn: 'platform' })
export class PlatformService { }

// Creates a NEW instance for EACH lazy-loaded module or feature that injects it.
// Each feature gets its own isolated copy.
@Injectable({ providedIn: 'any' })
export class FeatureIsolatedService { }
```

For services that should be scoped to a specific feature or component without the tree-shakeability of `providedIn`, you can provide them explicitly in a component or module:

```typescript
@Component({
  selector: 'app-checkout',
  providers: [CartService]  // CartService instance is scoped to this component subtree
})
export class CheckoutComponent { }
```

---

### `InjectionToken` — Providing Non-Class Values

What if you want to inject a configuration object, a string, a number, or a factory function rather than a class instance? You cannot use a class type as a DI token for primitive values. Instead, use `InjectionToken`:

```typescript
import { InjectionToken, inject } from '@angular/core';

// Define a typed injection token
export const API_URL = new InjectionToken<string>('API_URL');
export const APP_CONFIG = new InjectionToken<AppConfig>('APP Config');

// Provide the token value in app.config.ts
export const appConfig: ApplicationConfig = {
  providers: [
    { provide: API_URL, useValue: 'https://api.production.example.com' },
    {
      provide: APP_CONFIG,
      useValue: { maxRetries: 3, timeout: 5000, featureFlags: { darkMode: true } }
    }
  ]
};

// Inject the token value in a service or component
@Injectable({ providedIn: 'root' })
export class ApiService {
  private apiUrl = inject(API_URL);       // Injects the string
  private config = inject(APP_CONFIG);    // Injects the config object

  getData() {
    return `Calling ${this.apiUrl} with timeout ${this.config.timeout}ms`;
  }
}
```

---

### Provider Types: `useClass`, `useValue`, `useFactory`, `useExisting`

The `providers` array in modules, components, and `appConfig` accepts provider objects with different shapes:

**`useClass` — substitute a different implementation:**
```typescript
providers: [
  // When anything asks for UserService, give them MockUserService instead.
  // Perfect for testing or swapping implementations.
  { provide: UserService, useClass: MockUserService }
]
```

**`useValue` — provide a static value:**
```typescript
providers: [
  { provide: API_URL, useValue: 'https://api.example.com' },
  { provide: LOGGING_ENABLED, useValue: !environment.production }
]
```

**`useFactory` — provide a value computed by a function:**
```typescript
providers: [
  {
    provide: APP_CONFIG,
    // The factory function creates the value.
    // 'deps' specifies other tokens to inject into the factory.
    useFactory: (environment: EnvironmentService) => ({
      apiUrl: environment.getApiUrl(),
      isDev: !environment.isProduction()
    }),
    deps: [EnvironmentService]
  }
]
```

**`useExisting` — create an alias for an existing token:**
```typescript
providers: [
  // When something asks for LoggerService, give it the SAME instance as NgRxLoggerService.
  // Unlike useClass (which creates a new instance), useExisting returns the same object.
  { provide: LoggerService, useExisting: NgRxLoggerService }
]
```

---

### Multi-Providers — Collecting Multiple Implementations

Multi-providers let multiple values be registered for the same token. Angular collects all of them into an **array** when the token is injected. This is commonly used for interceptors, validators, and plugin systems.

```typescript
// Define the token
export const VALIDATORS = new InjectionToken<Validator[]>('App Validators');

// Register multiple providers for the same token
providers: [
  { provide: HTTP_INTERCEPTORS, useClass: AuthInterceptor, multi: true },
  { provide: HTTP_INTERCEPTORS, useClass: LoggingInterceptor, multi: true },
  { provide: HTTP_INTERCEPTORS, useClass: CachingInterceptor, multi: true }
  // When HTTP_INTERCEPTORS is injected, Angular provides [AuthInterceptor, LoggingInterceptor, CachingInterceptor]
]
```

---

### The `inject()` Function in Non-Class Contexts

The `inject()` function is not limited to class fields and constructors. It can also be called in **factory functions**, **functional guards**, and **functional interceptors** — contexts where constructor injection cannot work:

```typescript
import { inject } from '@angular/core';
import { CanActivateFn, Router } from '@angular/router';
import { AuthService } from './auth.service';

// A functional guard — no class, no constructor, but inject() works perfectly.
export const authGuard: CanActivateFn = (route, state) => {
  // inject() works here because Angular calls this function within an injection context.
  const authService = inject(AuthService);
  const router = inject(Router);

  if (authService.isAuthenticated()) {
    return true;
  }

  return router.createUrlTree(['/login']);
};
```

```typescript
import { HttpInterceptorFn } from '@angular/common/http';
import { inject } from '@angular/core';
import { AuthService } from './auth.service';

// A functional interceptor — no class needed.
export const authInterceptor: HttpInterceptorFn = (req, next) => {
  const authService = inject(AuthService);
  const token = authService.getToken();

  if (token) {
    const authReq = req.clone({
      headers: req.headers.set('Authorization', `Bearer ${token}`)
    });
    return next(authReq);
  }

  return next(req);
};
```

---

### Practical Example: Building a Complete Feature Service

Here is a realistic service that demonstrates the patterns working together:

```typescript
// services/product.service.ts
import { Injectable, inject, signal, computed } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { toSignal } from '@angular/core/rxjs-interop';
import { Observable, catchError, of, tap } from 'rxjs';
import { Product } from '../models/product.model';

@Injectable({ providedIn: 'root' })
export class ProductService {
  private http = inject(HttpClient);
  private readonly url = 'https://api.example.com/products';

  // Signal-based state management within a service
  private _products = signal<Product[]>([]);
  private _loading = signal<boolean>(false);
  private _error = signal<string | null>(null);

  // Expose as read-only signals to consumers
  readonly products = this._products.asReadonly();
  readonly loading = this._loading.asReadonly();
  readonly error = this._error.asReadonly();

  // Derived computed signal — only updates when _products changes
  readonly productCount = computed(() => this._products().length);
  readonly availableProducts = computed(() =>
    this._products().filter(p => p.inStock)
  );

  loadProducts(): void {
    this._loading.set(true);
    this._error.set(null);

    this.http.get<Product[]>(this.url)
      .pipe(
        tap(products => {
          this._products.set(products);  // Update the signal
          this._loading.set(false);
        }),
        catchError(err => {
          this._error.set('Failed to load products. Please try again.');
          this._loading.set(false);
          return of([]);
        })
      )
      .subscribe();
  }

  addProduct(product: Omit<Product, 'id'>): Observable<Product> {
    return this.http.post<Product>(this.url, product).pipe(
      tap(created => {
        // Immutably add to the signal's array — creates a new reference
        this._products.update(products => [...products, created]);
      })
    );
  }
}
```

And using this service in a component:

```typescript
// components/product-list.component.ts
import { Component, OnInit, inject } from '@angular/core';
import { ProductService } from '../services/product.service';

@Component({
  selector: 'app-product-list',
  standalone: true,
  template: `
    @if (productService.loading()) {
      <app-spinner />
    }

    @if (productService.error(); as error) {
      <div class="error">{{ error }}</div>
    }

    <p>Showing {{ productService.productCount() }} products
       ({{ productService.availableProducts().length }} in stock)</p>

    @for (product of productService.products(); track product.id) {
      <app-product-card [product]="product" />
    } @empty {
      <p>No products available.</p>
    }
  `
})
export class ProductListComponent implements OnInit {
  // inject() at the class level — clean, no constructor needed
  productService = inject(ProductService);

  ngOnInit(): void {
    this.productService.loadProducts();
  }
}
```

---

## Best Practices for 10.11

Use `providedIn: 'root'` for the vast majority of services. It is the simplest, most tree-shakeable, and most performant option. Only deviate when you genuinely need a component-scoped or route-scoped instance.

Prefer `inject()` over constructor injection for new code. It results in cleaner classes, eliminates long constructor parameter lists, and works naturally in functional contexts (guards, interceptors) that don't have constructors.

Keep services focused on a single domain. A `UserService` handles user CRUD operations. An `AuthService` handles authentication. A `CartService` handles shopping cart state. Mixing concerns into a single "super-service" makes code hard to test and maintain.

Never inject `ElementRef` into a service. Services should be environment-agnostic so they can run in tests, server-side rendering, and web workers. DOM references belong in components and directives, not services.

Use `InjectionToken` for configuration values rather than hardcoding them into services. Environment-specific URLs, feature flags, and API keys should be injectable so they can be swapped in tests and different deployment environments without touching the service code.

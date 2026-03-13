# Phase 11.5 — HTTP & HttpClient (Angular 21)

## Why Angular Has Its Own HTTP Layer

You might wonder why Angular wraps the browser's native fetch/XMLHttpRequest rather than using them directly. The Angular `HttpClient` gives you typed responses, automatic JSON parsing, RxJS Observable-based responses (so you can use all RxJS operators like `switchMap`, `retry`, `catchError`), interceptors for cross-cutting concerns (auth, logging, caching), and seamless integration with server-side rendering. It's a higher-level abstraction that handles the tedious parts of HTTP and fits naturally into Angular's reactive programming model.

---

## Setting Up `provideHttpClient()`

In modern Angular (standalone architecture), you configure `HttpClient` in your `ApplicationConfig` using `provideHttpClient()` rather than importing `HttpClientModule`.

```typescript
// app.config.ts
import { ApplicationConfig } from '@angular/core';
import { provideHttpClient, withFetch, withInterceptors } from '@angular/common/http';

export const appConfig: ApplicationConfig = {
  providers: [
    provideHttpClient(
      withFetch(),                           // Use native fetch() instead of XHR — recommended
      withInterceptors([authInterceptor, loggingInterceptor]),  // Functional interceptors
    ),
  ],
};
```

The `withFetch()` option is important for two reasons: it reduces bundle size by using the browser's native fetch API instead of XMLHttpRequest, and it is **required for Angular SSR** because the server environment has `fetch` but not `XMLHttpRequest`.

---

## Making HTTP Requests

`HttpClient` is injected into any service or component using `inject()`. All methods return `Observable<T>` — they don't send the request until something subscribes.

```typescript
// products.service.ts
import { Injectable, inject } from '@angular/core';
import { HttpClient, HttpHeaders, HttpParams } from '@angular/common/http';
import { Observable } from 'rxjs';
import { Product } from './product.model';

@Injectable({ providedIn: 'root' })
export class ProductsService {
  private http = inject(HttpClient);
  private apiUrl = 'https://api.example.com/products';

  // GET — fetch a collection
  getProducts(): Observable<Product[]> {
    return this.http.get<Product[]>(this.apiUrl);
  }

  // GET — fetch a single item by ID
  getProduct(id: string): Observable<Product> {
    return this.http.get<Product>(`${this.apiUrl}/${id}`);
  }

  // POST — create a new resource
  createProduct(product: Omit<Product, 'id'>): Observable<Product> {
    return this.http.post<Product>(this.apiUrl, product);
  }

  // PUT — replace an entire resource
  updateProduct(id: string, product: Product): Observable<Product> {
    return this.http.put<Product>(`${this.apiUrl}/${id}`, product);
  }

  // PATCH — partially update a resource
  patchProduct(id: string, changes: Partial<Product>): Observable<Product> {
    return this.http.patch<Product>(`${this.apiUrl}/${id}`, changes);
  }

  // DELETE — remove a resource
  deleteProduct(id: string): Observable<void> {
    return this.http.delete<void>(`${this.apiUrl}/${id}`);
  }
}
```

The generic type parameter (`get<Product[]>`) tells `HttpClient` what type to expect in the response. Angular automatically parses the JSON body and TypeScript enforces the correct type — no manual `JSON.parse()` needed.

---

## `HttpHeaders` and `HttpParams`

`HttpHeaders` and `HttpParams` are **immutable value objects**. Every method call (`.set()`, `.append()`) returns a new instance rather than mutating the original. This functional immutability prevents accidental side effects when reusing header objects across multiple requests.

```typescript
// Building headers
const headers = new HttpHeaders()
  .set('Content-Type', 'application/json')
  .set('X-Custom-Header', 'my-value');

// Or from an object literal (more concise)
const headers = new HttpHeaders({
  'Authorization': `Bearer ${this.authToken}`,
  'Accept-Language': 'en-US',
});

// Building query parameters (?page=2&limit=20&sort=name)
const params = new HttpParams()
  .set('page', 2)
  .set('limit', 20)
  .set('sort', 'name');

// Using them in a request
this.http.get<Product[]>('/api/products', { headers, params });
```

---

## Error Handling

Every HTTP request can fail — network errors, server errors (5xx), client errors (4xx). Angular wraps all these in an `HttpErrorResponse` object. You handle errors with RxJS's `catchError` operator.

```typescript
import { HttpErrorResponse } from '@angular/common/http';
import { catchError, throwError } from 'rxjs';

@Injectable({ providedIn: 'root' })
export class ProductsService {
  private http = inject(HttpClient);

  getProducts(): Observable<Product[]> {
    return this.http.get<Product[]>('/api/products').pipe(
      catchError((error: HttpErrorResponse) => {
        // Determine error type
        if (error.status === 0) {
          // Network error (no internet, CORS, etc.) — no status code
          console.error('Network error:', error.error);
        } else if (error.status === 404) {
          console.error('Products not found');
        } else if (error.status === 401) {
          console.error('Unauthorized — redirect to login');
        } else if (error.status >= 500) {
          console.error('Server error:', error.status);
        }
        // Re-throw as an Observable error for the component to handle
        return throwError(() => new Error(`HTTP ${error.status}: ${error.message}`));
      })
    );
  }
}
```

### Retry Patterns

The `retry` operator is invaluable for transient failures (network hiccups, temporarily unavailable services).

```typescript
import { retry } from 'rxjs';

// Simple retry — attempt 3 times before failing
this.http.get<Data>('/api/data').pipe(
  retry(3),
  catchError(handleError)
)

// Retry with increasing delay (exponential backoff) — production pattern
import { retry, timer } from 'rxjs';

this.http.get<Data>('/api/data').pipe(
  retry({
    count: 3,
    delay: (error, retryCount) =>
      // Wait 1s, then 2s, then 4s before each retry
      timer(1000 * Math.pow(2, retryCount - 1)),
  }),
  catchError(handleError)
)
```

---

## Functional Interceptors (Angular 15+ — Preferred)

Interceptors are middleware that sit between your service and the HTTP backend. Every outgoing request and incoming response passes through them. The functional interceptor pattern is preferred in Angular 21 because it's concise, tree-shakeable, and requires no class boilerplate.

A functional interceptor is a function with the signature `(req: HttpRequest<unknown>, next: HttpHandlerFn) => Observable<HttpEvent<unknown>>`.

### Authentication Interceptor

The most common use case: automatically attach a Bearer token to every request.

```typescript
// auth.interceptor.ts
import { HttpInterceptorFn } from '@angular/common/http';
import { inject } from '@angular/core';
import { AuthService } from './auth.service';

export const authInterceptor: HttpInterceptorFn = (req, next) => {
  const auth = inject(AuthService);   // inject() works in functional interceptors
  const token = auth.getToken();

  if (!token) {
    // No token — pass through unchanged
    return next(req);
  }

  // Clone the request and add the Authorization header
  // HttpRequest is immutable — you must clone() to modify it
  const authReq = req.clone({
    headers: req.headers.set('Authorization', `Bearer ${token}`),
  });

  return next(authReq);  // Forward the modified request
};
```

### Error Interceptor — Global HTTP Error Handling

```typescript
// error.interceptor.ts
import { HttpInterceptorFn, HttpErrorResponse } from '@angular/common/http';
import { inject } from '@angular/core';
import { catchError, throwError } from 'rxjs';
import { NotificationService } from './notification.service';
import { Router } from '@angular/router';

export const errorInterceptor: HttpInterceptorFn = (req, next) => {
  const notification = inject(NotificationService);
  const router = inject(Router);

  return next(req).pipe(
    catchError((error: HttpErrorResponse) => {
      if (error.status === 401) {
        // Session expired — redirect to login
        router.navigate(['/login']);
        notification.error('Your session has expired. Please log in again.');
      } else if (error.status === 403) {
        notification.error('You do not have permission to perform this action.');
      } else if (error.status >= 500) {
        notification.error('A server error occurred. Please try again later.');
      }

      // Always re-throw so services can also handle the error if needed
      return throwError(() => error);
    })
  );
};
```

### Logging / Timing Interceptor

```typescript
// logging.interceptor.ts
import { HttpInterceptorFn } from '@angular/common/http';
import { tap, finalize } from 'rxjs';

export const loggingInterceptor: HttpInterceptorFn = (req, next) => {
  const startTime = Date.now();
  console.log(`[HTTP] → ${req.method} ${req.url}`);

  return next(req).pipe(
    tap({
      next: () => {},
      error: (err) => console.error(`[HTTP] ✗ ${req.method} ${req.url}`, err),
    }),
    finalize(() => {
      const elapsed = Date.now() - startTime;
      console.log(`[HTTP] ← ${req.method} ${req.url} (${elapsed}ms)`);
    })
  );
};
```

### Registering Interceptors

```typescript
// app.config.ts
provideHttpClient(
  withFetch(),
  withInterceptors([
    loggingInterceptor,   // Runs first (outermost)
    authInterceptor,      // Runs second
    errorInterceptor,     // Runs last outgoing; first returning
  ])
)
```

Interceptors form a middleware chain. For outgoing requests, they execute in the order listed. For incoming responses, they execute in reverse order.

---

## `HttpContext` — Per-Request Metadata

`HttpContext` lets you pass metadata to an interceptor on a per-request basis without adding query params or custom headers. The token is a typed key.

```typescript
// Define a context token — this is a typed signal between components and interceptors
import { HttpContextToken } from '@angular/common/http';

export const SKIP_AUTH = new HttpContextToken<boolean>(() => false);
export const RETRY_COUNT = new HttpContextToken<number>(() => 3);

// In an interceptor — read the context
export const authInterceptor: HttpInterceptorFn = (req, next) => {
  if (req.context.get(SKIP_AUTH)) {
    return next(req);  // Skip authentication for this request
  }
  // ... normal auth logic
};

// At the call site — tell the interceptor to skip auth for this request
this.http.get('/api/public-data', {
  context: new HttpContext()
    .set(SKIP_AUTH, true)
    .set(RETRY_COUNT, 1),
});
```

---

## `httpResource()` — Signal-Based HTTP (Angular 21)

`httpResource()` is the newest addition in Angular 21 and represents the future of HTTP integration in Angular's signal ecosystem. It creates a signal-based resource that automatically manages loading, error, and data states — no subscriptions, no `BehaviorSubject`, no manual loading flags.

```typescript
import { httpResource } from '@angular/common/http';
import { Component } from '@angular/core';

@Component({
  selector: 'app-products',
  template: `
    @if (products.isLoading()) {
      <app-skeleton-list />
    }

    @if (products.error()) {
      <app-error-banner [message]="products.error().message" />
    }

    @if (products.value()) {
      @for (product of products.value(); track product.id) {
        <app-product-card [product]="product" />
      }
    }
  `,
})
export class ProductsComponent {
  // httpResource automatically fetches and re-fetches when signals change
  products = httpResource<Product[]>('/api/products');

  // products.value()     — Signal<Product[] | undefined>
  // products.isLoading() — Signal<boolean>
  // products.error()     — Signal<unknown>
  // products.status()    — Signal<'idle' | 'loading' | 'resolved' | 'error'>
}
```

### Dynamic `httpResource` with Signal Dependencies

When the URL or parameters depend on signals, `httpResource` automatically re-fetches when those signals change.

```typescript
@Component({ ... })
export class ProductDetailComponent {
  // Route param comes from input binding
  productId = input.required<string>();

  // This resource re-fetches automatically whenever productId() changes
  product = httpResource<Product>(
    () => `/api/products/${this.productId()}`  // A function — evaluated reactively
  );

  // With request configuration
  searchResults = httpResource<Product[]>(() => ({
    url: '/api/products/search',
    params: { q: this.searchQuery(), page: this.currentPage() },
    headers: { 'Accept': 'application/json' },
  }));
}
```

### Comparing `httpResource()` vs `HttpClient` + Observable

The Observable-based approach (`httpClient.get()`) gives you fine-grained RxJS control — perfect for complex flows like polling, retry with backoff, combining multiple streams, and caching. Use it in services.

`httpResource()` is the right choice when a component simply needs to display remote data and react to changing inputs. It eliminates the boilerplate of loading/error state management and integrates naturally with the signal ecosystem. Think of it as the Angular equivalent of TanStack Query's `useQuery`.

---

## Transfer State in SSR

When Angular SSR pre-renders a page on the server, it often makes HTTP requests to fetch data. Without transfer state, the browser would re-fetch that same data after hydration — wasting a network round trip.

Transfer state solves this by serialising the server's HTTP responses into the HTML, so the browser can pick them up without making duplicate requests.

```typescript
// With provideHttpClient() and withFetch(), Transfer State happens automatically
// for HttpClient requests made during SSR. You don't need to do anything manually.

// For custom transfer state needs:
import { TransferState, makeStateKey } from '@angular/core';

const PRODUCTS_KEY = makeStateKey<Product[]>('products');

@Injectable({ providedIn: 'root' })
export class ProductsService {
  private http = inject(HttpClient);
  private transferState = inject(TransferState);

  getProducts(): Observable<Product[]> {
    // If the server already fetched and stored this data, return it immediately
    if (this.transferState.hasKey(PRODUCTS_KEY)) {
      const products = this.transferState.get<Product[]>(PRODUCTS_KEY, []);
      this.transferState.remove(PRODUCTS_KEY);  // Clean up after reading
      return of(products);
    }

    return this.http.get<Product[]>('/api/products').pipe(
      tap(products => {
        // On the server, store the result in transfer state
        if (isPlatformServer(inject(PLATFORM_ID))) {
          this.transferState.set(PRODUCTS_KEY, products);
        }
      })
    );
  }
}
```

In practice, the built-in `withFetch()` combined with Angular's hydration handles this automatically for most cases. Manual transfer state management is needed only for custom data sources.

---

## Progress Events — Upload and Download Tracking

For large file uploads or downloads where you want to show a progress bar, you need to request the full HTTP response including progress events.

```typescript
import { HttpEventType, HttpRequest } from '@angular/common/http';

uploadFile(file: File): Observable<number> {
  const formData = new FormData();
  formData.append('file', file);

  const req = new HttpRequest('POST', '/api/upload', formData, {
    reportProgress: true,  // Enable progress events
  });

  return this.http.request(req).pipe(
    map(event => {
      switch (event.type) {
        case HttpEventType.UploadProgress:
          // event.loaded / event.total gives the percentage
          return event.total ? Math.round(100 * event.loaded / event.total) : 0;
        case HttpEventType.Response:
          return 100;  // Complete
        default:
          return 0;
      }
    }),
    filter(progress => progress >= 0)
  );
}
```

---

## A Complete Service Pattern

This is how a well-structured, production-ready HTTP service looks in Angular 21.

```typescript
// users.service.ts
import { Injectable, inject } from '@angular/core';
import { HttpClient, HttpParams } from '@angular/common/http';
import { Observable, throwError } from 'rxjs';
import { catchError, map, retry, timer } from 'rxjs';
import { User, UserCreateDto, UserListResponse } from './user.model';

@Injectable({ providedIn: 'root' })
export class UsersService {
  private http = inject(HttpClient);
  private baseUrl = '/api/users';

  getUsers(page = 1, pageSize = 20): Observable<User[]> {
    const params = new HttpParams()
      .set('page', page)
      .set('pageSize', pageSize);

    return this.http.get<UserListResponse>(this.baseUrl, { params }).pipe(
      map(response => response.data),  // Extract just the data array
      retry({                           // Retry transient failures
        count: 2,
        delay: (_, count) => timer(count * 500),
      }),
      catchError(err => throwError(() => new Error(`Failed to load users: ${err.message}`)))
    );
  }

  getUserById(id: string): Observable<User> {
    return this.http.get<User>(`${this.baseUrl}/${id}`).pipe(
      catchError(err => throwError(() => new Error(`User ${id} not found`)))
    );
  }

  createUser(dto: UserCreateDto): Observable<User> {
    return this.http.post<User>(this.baseUrl, dto);
  }

  updateUser(id: string, changes: Partial<User>): Observable<User> {
    return this.http.patch<User>(`${this.baseUrl}/${id}`, changes);
  }

  deleteUser(id: string): Observable<void> {
    return this.http.delete<void>(`${this.baseUrl}/${id}`);
  }
}
```

---

## Key Best Practices

Always use `withFetch()` in `provideHttpClient()` — it uses native fetch, reduces bundle size, and is required for SSR compatibility. Keep HTTP logic in services, not components. Components should call service methods and handle the resulting data; they should not construct HTTP requests directly. Use typed generics on all HttpClient methods — `this.http.get<User[]>(url)` is always better than `this.http.get(url)` as it eliminates `any` types. Always handle errors — an unhandled Observable error terminates the stream, which means a component subscribed to it will stop receiving future values (for hot observables) or the error propagates to the global unhandled error handler. Use the global error interceptor for cross-cutting concerns (401 redirects, 500 notifications) and local `catchError` for route or component-specific error messages. Prefer `httpResource()` in Angular 21 for simple data-display components; keep the Observable pattern in services for complex async orchestration.

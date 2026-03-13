# Phase 12.5 — Angular SSR (Server-Side Rendering — Angular 17–21)

> **Prerequisites:** Angular routing, HttpClient, basic Node.js/Express knowledge

---

## Table of Contents
1. [Why SSR? The Core Case](#1-why-ssr-the-core-case)
2. [SSR in Angular 17+ — Built Into CLI](#2-ssr-in-angular-17--built-into-cli)
3. [Server-Side Rendering with Express](#3-server-side-rendering-with-express)
4. [Transfer State — Avoiding Double Fetching](#4-transfer-state--avoiding-double-fetching)
5. [Non-Destructive Hydration (Angular 16+)](#5-non-destructive-hydration-angular-16)
6. [Incremental Hydration (Angular 19+)](#6-incremental-hydration-angular-19)
7. [Partial Hydration (Angular 21)](#7-partial-hydration-angular-21)
8. [Event Replay (Angular 19+)](#8-event-replay-angular-19)
9. [Static Site Generation (SSG / Prerendering)](#9-static-site-generation-ssg--prerendering)
10. [App Shell Pattern](#10-app-shell-pattern)
11. [SSR-Specific APIs and Guards](#11-ssr-specific-apis-and-guards)
12. [HTTP in SSR Context](#12-http-in-ssr-context)
13. [Edge SSR Deployment](#13-edge-ssr-deployment)
14. [renderApplication() and renderModule()](#14-renderapplication-and-rendermodule)
15. [SSR Performance Checklist](#15-ssr-performance-checklist)
16. [Best Practices](#16-best-practices)

---

## 1. Why SSR? The Core Case

### The SPA Problem

A client-side Angular SPA sends this to the browser initially:

```html
<!DOCTYPE html>
<html>
  <head><title>My App</title></head>
  <body>
    <app-root></app-root>
    <script src="main.js"></script>  <!-- 300kB+ of JavaScript -->
  </body>
</html>
```

The browser must:
1. Download `main.js` (300kB+)
2. Parse and execute it
3. Angular bootstraps
4. Components fetch data
5. DOM renders

**Time to Meaningful Content: 3–8 seconds on slow connections.**

### What SSR Provides

With SSR, the server renders the initial HTML and sends it fully populated:

```html
<!DOCTYPE html>
<html>
  <head><title>Products | My Shop</title>
    <meta property="og:title" content="Amazing Products" />
    <meta name="description" content="Browse 10,000+ products..." />
  </head>
  <body>
    <app-root>
      <!-- Pre-rendered HTML — visible immediately -->
      <header>My Shop</header>
      <main>
        <h1>Products</h1>
        <div class="product-grid">
          <div class="product-card">Product A - $29.99</div>
          <div class="product-card">Product B - $49.99</div>
          <!-- ... full product list -->
        </div>
      </main>
    </app-root>
    <script src="main.js"></script>
  </body>
</html>
```

| Benefit | Details |
|---|---|
| **SEO** | Search engines see real content, not empty `<app-root>` |
| **Social sharing** | Open Graph / Twitter Card meta tags populated |
| **LCP (Largest Contentful Paint)** | Content visible before JS loads |
| **Core Web Vitals** | Dramatically better scores |
| **Slow devices** | Usable before JavaScript executes |

---

## 2. SSR in Angular 17+ — Built Into CLI

Prior to Angular 17, SSR required `@nguniversal/express-engine` — a separate package. Angular 17+ builds SSR directly into the CLI.

### New Project with SSR

```bash
ng new my-app --ssr
```

### Add SSR to Existing Project

```bash
ng add @angular/ssr
```

This modifies:
- `angular.json` — adds SSR build configuration
- `app.config.ts` — adds `provideClientHydration()`
- `server.ts` — Express server entry point
- `app.config.server.ts` — server-specific providers

### Generated File Structure

```
src/
├── app/
│   ├── app.component.ts
│   ├── app.config.ts           ← Browser config
│   ├── app.config.server.ts    ← Server-specific config (merged with browser)
│   └── app.routes.ts
├── main.ts                     ← Browser entry point
└── main.server.ts              ← Server entry point
server.ts                       ← Express server
```

### app.config.ts (Browser)
```typescript
// app.config.ts
import { ApplicationConfig, provideZoneChangeDetection } from '@angular/core';
import { provideRouter } from '@angular/router';
import { provideClientHydration } from '@angular/platform-browser';
import { provideHttpClient, withFetch } from '@angular/common/http';

export const appConfig: ApplicationConfig = {
  providers: [
    provideZoneChangeDetection({ eventCoalescing: true }),
    provideRouter(routes),
    provideClientHydration(),  // ← Enables non-destructive hydration
    provideHttpClient(withFetch()),
  ],
};
```

### app.config.server.ts (Server)
```typescript
// app.config.server.ts
import { mergeApplicationConfig, ApplicationConfig } from '@angular/core';
import { provideServerRendering } from '@angular/platform-server';

const serverConfig: ApplicationConfig = {
  providers: [
    provideServerRendering(),  // ← Enables SSR
  ],
};

export const config = mergeApplicationConfig(appConfig, serverConfig);
```

---

## 3. Server-Side Rendering with Express

### server.ts — The Express Server

```typescript
// server.ts
import { APP_BASE_HREF } from '@angular/common';
import { CommonEngine } from '@angular/ssr';
import express from 'express';
import { fileURLToPath } from 'node:url';
import { dirname, join, resolve } from 'node:path';
import bootstrap from './src/main.server';

const serverDistFolder = dirname(fileURLToPath(import.meta.url));
const browserDistFolder = resolve(serverDistFolder, '../browser');
const indexHtml = join(serverDistFolder, 'index.server.html');

const app = express();
const commonEngine = new CommonEngine();

// Serve static files from the /browser folder
app.get('**', express.static(browserDistFolder, {
  maxAge: '1y',
  index: false,
}));

// All regular routes use the Angular Universal engine
app.get('**', (req, res, next) => {
  const { protocol, originalUrl, baseUrl, headers } = req;

  commonEngine
    .render({
      bootstrap,
      documentFilePath: indexHtml,
      url: `${protocol}://${headers.host}${originalUrl}`,
      publicPath: browserDistFolder,
      providers: [
        { provide: APP_BASE_HREF, useValue: baseUrl },
      ],
    })
    .then(html => res.send(html))
    .catch(err => next(err));
});

const port = process.env['PORT'] || 4000;
app.listen(port, () => {
  console.log(`Node server listening on http://localhost:${port}`);
});
```

### Building for SSR

```bash
ng build        # Builds both browser and server bundles

# dist/my-app/
# ├── browser/   ← Static files served to client
# └── server/    ← Node.js server bundle
```

### Running SSR Locally

```bash
ng serve        # Dev server with SSR support (Angular 17+)
node dist/my-app/server/server.mjs  # Production server
```

---

## 4. Transfer State — Avoiding Double Fetching

Without Transfer State, data is fetched **twice** — once on the server and once on the client after hydration.

### The Problem

```
Server: GET /api/products → renders HTML with product list
Browser receives HTML (visible immediately) ✅
Browser downloads main.js, Angular bootstraps
Angular re-fetches /api/products → flickers while re-rendering ❌
```

### Angular 17+ Automatic Transfer State

With `withFetch()` in `provideHttpClient()`, Angular **automatically** transfers HTTP responses from server to client. No manual configuration needed:

```typescript
// app.config.ts — withFetch() enables automatic transfer state
provideHttpClient(withFetch())
```

### Manual Transfer State (Advanced)

For custom data not going through HttpClient:

```typescript
import { TransferState, makeStateKey, inject } from '@angular/core';

const PRODUCTS_KEY = makeStateKey<Product[]>('products-data');

@Injectable({ providedIn: 'root' })
export class ProductService {
  private transferState = inject(TransferState);
  private http = inject(HttpClient);

  getProducts(): Observable<Product[]> {
    // Check if server already fetched this
    if (this.transferState.hasKey(PRODUCTS_KEY)) {
      const cached = this.transferState.get(PRODUCTS_KEY, []);
      this.transferState.remove(PRODUCTS_KEY); // Consume once
      return of(cached);
    }

    return this.http.get<Product[]>('/api/products').pipe(
      tap(products => {
        // On the server: save for client to pick up
        this.transferState.set(PRODUCTS_KEY, products);
      })
    );
  }
}
```

---

## 5. Non-Destructive Hydration (Angular 16+)

### Old Approach — Destructive Hydration

Before Angular 16, client-side bootstrap **destroyed the server-rendered DOM** and re-created everything from scratch:

```
Server renders: <h1>Products</h1> + full product list HTML
Browser: downloads JS, bootstraps...
Angular: DESTROYS the server HTML
Angular: RE-CREATES the same DOM from scratch → flicker! 💥
```

### New Approach — Non-Destructive Hydration

Angular 16+ **reuses the server-rendered DOM** and only attaches event listeners and data bindings:

```
Server renders: <h1>Products</h1> + full product list HTML
Browser: sees the HTML immediately (fast perceived load) ✅
Angular bootstraps...
Angular: WALKS the existing DOM and attaches bindings (no recreation) ✅
Result: No flicker, no layout shift ✅
```

### Enabling Hydration

```typescript
// app.config.ts
import { provideClientHydration } from '@angular/platform-browser';

export const appConfig: ApplicationConfig = {
  providers: [
    provideClientHydration(), // Enables non-destructive hydration
    // ...
  ],
};
```

### `ngSkipHydration` Attribute

For components that can't be hydrated (e.g., use browser-only APIs):

```html
<!-- Skip hydration for this component — re-rendered by client -->
<app-map ngSkipHydration></app-map>
```

---

## 6. Incremental Hydration (Angular 19+)

Incremental hydration defers hydrating `@defer` blocks until they're needed:

```typescript
// app.config.ts — enable incremental hydration
import { provideClientHydration, withIncrementalHydration } from '@angular/platform-browser';

providers: [
  provideClientHydration(withIncrementalHydration()),
]
```

Template:
```html
<main>
  <!-- Immediately hydrated — user interacts with this above fold -->
  <app-product-hero [product]="featured" />
  <app-add-to-cart [productId]="featured.id" />

  <!-- Only hydrated when scrolled into view -->
  @defer (hydrate on viewport) {
    <app-recommendations />
  }

  <!-- Only hydrated when user interacts -->
  @defer (hydrate on interaction) {
    <app-review-section [productId]="product.id" />
  }

  <!-- Never hydrated — static content -->
  @defer (hydrate never) {
    <app-footer-links />
  }
</main>
```

### Hydration Triggers

| Trigger | When it hydrates |
|---|---|
| `hydrate on viewport` | Component scrolls into view |
| `hydrate on interaction` | User clicks/touches the component |
| `hydrate on hover` | User hovers over the component |
| `hydrate on idle` | Browser is idle |
| `hydrate on immediate` | As soon as possible (eagerly) |
| `hydrate when condition` | When a condition signal becomes true |
| `hydrate never` | Never hydrated (pure static HTML) |

---

## 7. Partial Hydration (Angular 21)

Partial hydration is the full realization of the islands architecture — each component independently decides whether and when to hydrate:

```typescript
// Components can opt into hydration independently
@Component({
  selector: 'app-product-price',
  // This component is static — never needs hydration
  // Marked as non-interactive — pure server HTML
  template: `<span class="price">{{ product().price | currency }}</span>`,
})
export class ProductPriceComponent {
  product = input.required<Product>();
}

// Interactive component — gets hydrated
@Component({
  selector: 'app-add-to-cart',
  template: `<button (click)="addToCart()">Add to Cart</button>`,
})
export class AddToCartComponent {
  productId = input.required<string>();
  addToCart() { /* ... */ }
}
```

```html
<!-- Static components stay as inert HTML — zero JavaScript overhead -->
<app-product-price [product]="product" />

<!-- Interactive components hydrate when needed -->
@defer (hydrate on interaction) {
  <app-add-to-cart [productId]="product.id" />
}
```

This achieves the **Islands Architecture** pattern — where most of the page is static HTML and only interactive "islands" load JavaScript.

---

## 8. Event Replay (Angular 19+)

A common SSR problem: user clicks a button before Angular finishes hydrating → the click is lost.

**Event Replay** captures user interactions before hydration and **replays them** once Angular is ready:

```typescript
// app.config.ts — enable event replay
import { provideClientHydration, withEventReplay } from '@angular/platform-browser';

providers: [
  provideClientHydration(withEventReplay()),
]
```

```
User clicks "Add to Cart" at t=0.5s
Angular bootstraps at t=1.5s
→ Without event replay: click lost ❌
→ With event replay: click captured, replayed at t=1.5s ✅
```

---

## 9. Static Site Generation (SSG / Prerendering)

For static pages (blog posts, documentation, product pages), prerender at build time:

### Configuration in angular.json

```json
{
  "architect": {
    "build": {
      "options": {
        "prerender": true,           // Enable prerendering
        "server": "src/main.server.ts"
      }
    }
  }
}
```

### Prerender Specific Routes

```json
{
  "prerender": {
    "routesFile": "routes.txt",   // List of routes to prerender
    "discoverRoutes": true         // Auto-discover from router config
  }
}
```

```
# routes.txt
/
/about
/products
/products/1
/products/2
/blog
/blog/first-post
```

### Route Discovery

Angular 17+ can automatically discover routes from the router config:

```typescript
// Routes with params need data to generate all URLs
{
  path: 'products/:id',
  component: ProductDetailComponent,
  resolve: {
    // Provide all possible IDs for prerendering
    product: productResolver
  }
}
```

### Prerendering with `ng build`

```bash
ng build   # Generates dist/my-app/browser/ with pre-rendered HTML files

# dist/my-app/browser/
# ├── index.html              ← / route
# ├── about/index.html        ← /about route
# ├── products/index.html     ← /products route
# ├── products/1/index.html   ← /products/1 route
# └── ...
```

Serve with any static file host — no Node.js server needed for SSG!

---

## 10. App Shell Pattern

The App Shell pre-renders the **application shell** (header, navigation, loading skeleton) while keeping route content lazy-loaded:

```bash
ng generate app-shell
```

```typescript
// app-shell component
@Component({
  selector: 'app-shell',
  template: `
    <header>
      <nav>My App</nav>
    </header>
    <main>
      <div class="skeleton-loader">Loading...</div>
    </main>
  `,
})
export class AppShellComponent {}

// Route config
{
  path: 'shell',
  component: AppShellComponent,
}
```

The shell is pre-rendered → displayed instantly → full app bootstraps and replaces skeleton.

---

## 11. SSR-Specific APIs and Guards

### `isPlatformBrowser()` and `isPlatformServer()`

```typescript
import { PLATFORM_ID, inject, isPlatformBrowser, isPlatformServer } from '@angular/core';

@Component({ ... })
export class MapComponent implements OnInit {
  private platformId = inject(PLATFORM_ID);

  ngOnInit() {
    if (isPlatformBrowser(this.platformId)) {
      // Only runs in browser — safe to use window, document, localStorage
      this.initGoogleMaps();
    }
  }
}

@Injectable({ providedIn: 'root' })
export class AnalyticsService {
  private platformId = inject(PLATFORM_ID);

  track(event: string) {
    if (!isPlatformBrowser(this.platformId)) return; // Skip on server
    window.gtag('event', event);
  }
}
```

### `afterRender()` — Browser-Only Lifecycle

```typescript
import { afterRender } from '@angular/core';

@Component({ ... })
export class ChartComponent {
  constructor() {
    // afterRender only runs in the browser — safe for DOM access
    afterRender(() => {
      this.initChart(this.canvas().nativeElement);
    });
  }
}
```

### Browser-Only Providers

```typescript
// app.config.ts — provide different implementations per platform
import { isPlatformBrowser } from '@angular/common';
import { PLATFORM_ID } from '@angular/core';

export const STORAGE = new InjectionToken<Storage>('STORAGE', {
  factory: () => {
    const platformId = inject(PLATFORM_ID);
    return isPlatformBrowser(platformId)
      ? localStorage
      : { getItem: () => null, setItem: () => {}, removeItem: () => {} }; // SSR stub
  },
});
```

---

## 12. HTTP in SSR Context

### Absolute URLs Required on Server

HTTP requests on the server **must use absolute URLs** — relative URLs don't work in Node.js:

```typescript
// ❌ Relative URL — works in browser, fails on server
this.http.get('/api/products')

// ✅ Absolute URL — works everywhere
this.http.get('https://api.example.com/products')
```

### Environment-Based URL Strategy

```typescript
// app.config.server.ts — provide absolute base URL on server
import { APP_BASE_HREF } from '@angular/common';

const serverConfig: ApplicationConfig = {
  providers: [
    provideServerRendering(),
    { provide: APP_BASE_HREF, useValue: process.env['API_URL'] ?? 'http://localhost:4000' },
  ],
};
```

```typescript
// api.service.ts
@Injectable({ providedIn: 'root' })
export class ApiService {
  private baseHref = inject(APP_BASE_HREF, { optional: true }) ?? '';

  get<T>(path: string) {
    // Browser: path is '/api/products'
    // Server: baseHref + '/api/products' = 'http://localhost:4000/api/products'
    return this.http.get<T>(`${this.baseHref}${path}`);
  }
}
```

### `withFetch()` — Required for Edge/Deno SSR

```typescript
provideHttpClient(withFetch()) // Uses native fetch() — works on all platforms
```

---

## 13. Edge SSR Deployment

Angular 21 SSR works on edge runtimes (Cloudflare Workers, Vercel Edge, Deno Deploy):

### Cloudflare Workers

```typescript
// worker.ts
import { renderApplication } from '@angular/platform-server';
import bootstrap from './main.server';

export default {
  async fetch(request: Request): Promise<Response> {
    const url = new URL(request.url);
    const html = await renderApplication(bootstrap, {
      document: '<app-root></app-root>',
      url: url.pathname,
    });
    return new Response(html, { headers: { 'content-type': 'text/html' } });
  },
};
```

### Vercel Edge Functions

```typescript
// api/index.ts (Vercel edge function)
import { renderApplication } from '@angular/platform-server';

export const config = { runtime: 'edge' };

export default async function handler(req: Request) {
  const html = await renderApplication(bootstrap, {
    document: indexHtml,
    url: new URL(req.url).pathname,
  });
  return new Response(html, { headers: { 'content-type': 'text/html' } });
}
```

---

## 14. `renderApplication()` and `renderModule()`

Low-level APIs for custom SSR integrations:

```typescript
import { renderApplication } from '@angular/platform-server';

// Render a standalone application
const html = await renderApplication(bootstrap, {
  document: '<app-root></app-root>',
  url: '/products/123',
  platformProviders: [
    { provide: APP_BASE_HREF, useValue: 'https://api.example.com' },
  ],
});

// renderModule — for NgModule-based apps (legacy)
import { renderModule } from '@angular/platform-server';

const html = await renderModule(AppModule, {
  document: '<app-root></app-root>',
  url: '/products',
});
```

---

## 15. SSR Performance Checklist

```
□ provideClientHydration() enabled
□ withFetch() in provideHttpClient() — enables auto Transfer State
□ withIncrementalHydration() for @defer blocks
□ withEventReplay() for pre-hydration interactions
□ Absolute URLs for all server-side HTTP requests
□ isPlatformBrowser() guards for browser-only code
□ ngSkipHydration on problematic components
□ Static pages using prerendering (SSG), not SSR
□ App Shell for perceived performance
□ @defer with hydrate triggers for below-fold content
□ Edge deployment for global low-latency rendering
```

---

## 16. Best Practices

1. **Use `ng add @angular/ssr`** for existing projects — it configures everything correctly.
2. **Always use `withFetch()`** — enables Transfer State and edge compatibility.
3. **Use `isPlatformBrowser()`** to guard all browser-specific APIs (window, document, localStorage).
4. **Prefer prerendering (SSG)** over SSR for static content — scales infinitely, no server needed.
5. **Use `withIncrementalHydration()`** + `@defer` for below-fold content — massive LCP improvement.
6. **Absolute URLs** for all HTTP calls — relative URLs break in Node.js context.
7. **Deploy to edge** (Cloudflare Workers, Vercel Edge) for global low-latency SSR.
8. **Test SSR locally** with `ng serve` before deploying — SSR errors are often server-only.

---

> **Summary:** Angular SSR in 2026 is a fully integrated, powerful system. From simple `ng new --ssr` setup to incremental hydration, partial hydration, event replay, and edge deployment — Angular provides a complete toolkit for building production-ready SSR applications. The key evolution is from "full hydration overhead" to "islands-like partial hydration" where static content remains inert and only interactive components pay the JavaScript cost.

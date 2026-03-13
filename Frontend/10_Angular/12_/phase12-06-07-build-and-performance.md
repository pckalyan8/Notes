# Phase 12.6 — Angular Build System (Angular 17–21)

> **Prerequisites:** npm/package managers, basic bundling concepts (Phase 9)

---

## Table of Contents
1. [The New Application Builder](#1-the-new-application-builder)
2. [esbuild — The Production Bundler](#2-esbuild--the-production-bundler)
3. [Vite-Based Dev Server](#3-vite-based-dev-server)
4. [Build Configuration in angular.json](#4-build-configuration-in-angularjson)
5. [Build Cache](#5-build-cache)
6. [Bundle Budgets](#6-bundle-budgets)
7. [outputMode — Static, Server, SSR](#7-outputmode--static-server-ssr)
8. [define — Compile-Time Constants](#8-define--compile-time-constants)
9. [externalDependencies](#9-externaldependencies)
10. [Bundle Analysis](#10-bundle-analysis)
11. [Rolldown Integration (Angular 21 Experimental)](#11-rolldown-integration-angular-21-experimental)
12. [Best Practices](#12-best-practices)

---

## 1. The New Application Builder

Since Angular 17, the default builder is `@angular-devkit/build-angular:application` — replacing the old `@angular-devkit/build-angular:browser`.

```json
// angular.json
{
  "architect": {
    "build": {
      "builder": "@angular-devkit/build-angular:application",  // New builder
      "options": {
        "outputPath": "dist/my-app",
        "index": "src/index.html",
        "browser": "src/main.ts",
        "polyfills": [],
        "tsConfig": "tsconfig.app.json",
        "assets": ["src/favicon.ico", "src/assets"],
        "styles": ["src/styles.scss"],
        "scripts": []
      }
    }
  }
}
```

### Key Differences from Old Builder

| Feature | Old (browser builder) | New (application builder) |
|---|---|---|
| Bundler | Webpack | esbuild |
| Dev server | webpack-dev-server | Vite-based |
| Build speed | Slow (30s+) | Fast (3-8s) |
| SSR support | Separate builder | Built-in |
| HMR | Slow (full reload) | Instant |

---

## 2. esbuild — The Production Bundler

Angular 17+ uses **esbuild** (written in Go) for production builds — 3–5× faster than Webpack.

### Speed Comparison

```
Small app (50 components):
  Webpack: ~25s
  esbuild: ~5s

Large app (500 components):
  Webpack: ~120s
  esbuild: ~15s

Incremental build (one file changed):
  Webpack: ~8s
  esbuild: ~0.5s
```

### esbuild Capabilities in Angular

- TypeScript transpilation
- ES module bundling
- Tree-shaking (dead code elimination)
- CSS bundling + minification
- HTML processing
- Source map generation
- Code splitting (lazy routes → separate chunks)

### What Webpack Had that esbuild Handles Differently

```typescript
// Webpack custom loaders/plugins → use Angular CLI builders instead
// No webpack.config.js in new Angular projects

// If you need webpack customization:
npm install @angular-builders/custom-webpack
// (Rare edge case — avoid if possible)
```

---

## 3. Vite-Based Dev Server

The Angular development server now uses **Vite** — providing instant HMR (Hot Module Replacement):

### How It Works

In development, the app is **not bundled** — ES modules are served directly to the browser. Only changed modules are reloaded.

```bash
ng serve
# Vite starts in milliseconds
# Files served as native ES modules
# Changes: only the modified file is reprocessed → near-instant HMR
```

### Dev Server Configuration

```json
// angular.json
{
  "serve": {
    "options": {
      "host": "0.0.0.0",
      "port": 4200,
      "open": true,
      "liveReload": true,
      "hmr": true,                  // Hot Module Replacement
      "proxyConfig": "proxy.conf.json"  // API proxy
    }
  }
}
```

### API Proxy Configuration

```json
// proxy.conf.json — proxy /api to backend
{
  "/api": {
    "target": "http://localhost:3000",
    "secure": false,
    "changeOrigin": true,
    "pathRewrite": { "^/api": "" }
  },
  "/ws": {
    "target": "ws://localhost:3000",
    "ws": true
  }
}
```

---

## 4. Build Configuration in angular.json

### Production Configuration

```json
{
  "configurations": {
    "production": {
      "optimization": {
        "scripts": true,    // Minify JavaScript
        "styles": {
          "minify": true,
          "inlineCritical": true  // Inline critical CSS for fast FCP
        },
        "fonts": {
          "inline": true    // Inline font @font-face — avoids render-blocking
        }
      },
      "outputHashing": "all",   // Hash all output files for cache busting
      "sourceMap": false,
      "namedChunks": false,
      "extractLicenses": true,  // Extract LICENSE.txt
      "budgets": [
        { "type": "initial", "maximumWarning": "500kb", "maximumError": "1mb" },
        { "type": "anyComponentStyle", "maximumWarning": "4kb" }
      ]
    },
    "development": {
      "optimization": false,
      "sourceMap": true,
      "namedChunks": true       // Human-readable chunk names
    }
  }
}
```

### Source Maps

```json
{
  "sourceMap": {
    "scripts": true,    // Source maps for JavaScript
    "styles": true,     // Source maps for CSS
    "vendor": false,    // Exclude vendor libraries from source maps
    "hidden": true      // Don't reference source maps in output (upload to error tracker instead)
  }
}
```

---

## 5. Build Cache

Angular CLI caches build artifacts locally — rebuild only what changed.

```bash
# Check cache status
ng cache info

# Enable (default in Angular 17+)
ng cache enable

# Disable
ng cache disable

# Clear cache
ng cache clean
```

```json
// angular.json — cache configuration
{
  "cli": {
    "cache": {
      "enabled": true,
      "path": ".angular/cache",
      "environment": "local"  // 'local' | 'ci' | 'all'
    }
  }
}
```

### Cache in CI/CD

```yaml
# GitHub Actions — cache Angular build artifacts
- name: Cache Angular Build
  uses: actions/cache@v3
  with:
    path: .angular/cache
    key: angular-${{ hashFiles('package-lock.json') }}-${{ github.sha }}
    restore-keys: |
      angular-${{ hashFiles('package-lock.json') }}-
      angular-
```

---

## 6. Bundle Budgets

Define maximum bundle sizes — Angular warns or errors when exceeded:

```json
{
  "budgets": [
    {
      "type": "initial",
      "maximumWarning": "500kb",    // Warning at 500kB
      "maximumError": "1mb"         // Build error at 1MB
    },
    {
      "type": "anyComponentStyle",
      "maximumWarning": "4kb",       // Warn if any component stylesheet > 4kB
      "maximumError": "10kb"
    },
    {
      "type": "allScript",
      "maximumWarning": "2mb",       // All scripts combined
      "maximumError": "5mb"
    },
    {
      "type": "anyScript",
      "maximumWarning": "150kb"      // Any single script chunk
    }
  ]
}
```

### Budget Types

| Type | Monitors |
|---|---|
| `initial` | Initial load bundle (main.js + vendor) |
| `lazy` | Any lazy-loaded chunk |
| `allScript` | All JavaScript combined |
| `all` | Everything (JS + CSS) |
| `anyScript` | Any individual script file |
| `any` | Any individual file |
| `anyComponentStyle` | Any component's inline styles |

---

## 7. `outputMode` — Static, Server, SSR

```json
{
  "options": {
    "outputMode": "static",            // Pure SPA — index.html + assets only
    "outputMode": "server",            // SSR with Node.js server
    "outputMode": "server-side-rendering" // Full SSR (same as "server")
  }
}
```

| Mode | Output | Use For |
|---|---|---|
| `static` | `index.html` + JS/CSS | CDN deployment, simple SPAs |
| `server` | Browser bundle + server bundle | SSR with Node.js |

---

## 8. `define` — Compile-Time Constants

Replace the old `environment.ts` approach with `define` for compile-time constants:

```json
// angular.json
{
  "configurations": {
    "production": {
      "define": {
        "API_URL": "\"https://api.example.com\"",
        "APP_VERSION": "\"1.2.3\"",
        "FEATURE_FLAGS.NEW_DASHBOARD": "true"
      }
    },
    "development": {
      "define": {
        "API_URL": "\"http://localhost:3000\"",
        "APP_VERSION": "\"dev\"",
        "FEATURE_FLAGS.NEW_DASHBOARD": "false"
      }
    }
  }
}
```

```typescript
// global.d.ts — declare the constants for TypeScript
declare const API_URL: string;
declare const APP_VERSION: string;

// Usage
@Injectable({ providedIn: 'root' })
export class ApiService {
  private baseUrl = API_URL; // 'https://api.example.com' in production
}
```

> `define` performs **string replacement at compile time** — tree-shaking can eliminate dead code branches based on constant conditions.

---

## 9. `externalDependencies`

Exclude packages from the bundle (load them via CDN or peer dependency):

```json
{
  "options": {
    "externalDependencies": [
      "lodash",       // Don't bundle lodash — expect it as window._
      "chart.js"      // Don't bundle Chart.js
    ]
  }
}
```

Use case: Module Federation (micro frontends) where a library is shared across apps.

---

## 10. Bundle Analysis

### Using `--stats-json` + webpack-bundle-analyzer

```bash
ng build --stats-json
npx webpack-bundle-analyzer dist/my-app/browser/stats.json
```

Opens interactive treemap showing every module's contribution to bundle size.

### Using `source-map-explorer`

```bash
ng build --source-map
npx source-map-explorer 'dist/my-app/browser/*.js' --html report.html
```

### Key Things to Look For

1. **Large third-party libraries** — often import more than needed
2. **Duplicate modules** — same library appearing multiple times
3. **Lazy chunk analysis** — ensure lazy routes aren't accidentally eager
4. **Lodash/moment full imports** — should use `lodash-es` or tree-shakeable alternatives

```typescript
// ❌ Imports entire lodash (700kB+)
import _ from 'lodash';
const result = _.groupBy(items, 'category');

// ✅ Import only what you need (a few kB)
import groupBy from 'lodash-es/groupBy';
const result = groupBy(items, 'category');
```

---

## 11. Rolldown Integration (Angular 21 Experimental)

Angular 21 introduces **experimental Rolldown** support — a Rust-based Rollup-compatible bundler even faster than esbuild:

```json
// angular.json — enable experimental Rolldown
{
  "options": {
    "bundler": "rolldown"  // Experimental in Angular 21
  }
}
```

### Rolldown Performance

```
esbuild (current default):   5s for large app
Rolldown (Rust, v0.x):       2-3s for same app
Rolldown (target v1.0):      Sub-second expected
```

> Rolldown will become the default in a future Angular release.

---

## 12. Best Practices

1. **Never configure Webpack manually** — use Angular CLI builders and features instead.
2. **Enable build cache** — `.angular/cache` — especially valuable in CI.
3. **Set bundle budgets** from day one — prevents gradual bloat.
4. **Use `--source-map` + `source-map-explorer`** quarterly to audit bundle growth.
5. **Use `define`** instead of `environment.ts` — enables better tree-shaking.
6. **Analyze lazy chunks** — each route chunk should be < 100kB ideally.
7. **Use `namedChunks: true`** in development for readable chunk names.

---

---

# Phase 12.7 — Performance Optimization in Angular (2026 Best Practices)

---

## Table of Contents
13. [The Performance Hierarchy](#13-the-performance-hierarchy)
14. [Signals + Zoneless — The Foundation](#14-signals--zoneless--the-foundation)
15. [track in @for — Required Optimization](#15-track-in-for--required-optimization)
16. [Pure Pipes Over Methods](#16-pure-pipes-over-methods)
17. [@defer — Eliminate Initial Bundle Weight](#17-defer--eliminate-initial-bundle-weight)
18. [CDK Virtual Scrolling](#18-cdk-virtual-scrolling)
19. [NgOptimizedImage](#19-ngoptimizedimage)
20. [afterNextRender — Defer DOM Reads](#20-afternextrender--defer-dom-reads)
21. [Tree-Shaking Services](#21-tree-shaking-services)
22. [takeUntilDestroyed — No More Memory Leaks](#22-takeuntildestroyed--no-more-memory-leaks)
23. [withFetch() — Smaller Bundle, SSR Ready](#23-withfetch--smaller-bundle-ssr-ready)
24. [Font Optimization](#24-font-optimization)
25. [Hydration Deferral — Islands Pattern](#25-hydration-deferral--islands-pattern)
26. [Preloading Strategy](#26-preloading-strategy)
27. [Best Practices Checklist](#27-best-practices-checklist)

---

## 13. The Performance Hierarchy

Performance improvements in priority order:

```
1. Zoneless + Signals          → ~40% fewer CD cycles, -13kB bundle
2. @defer for heavy content    → Massive initial bundle reduction
3. Lazy loading routes         → Only load what's needed
4. OnPush everywhere           → Skip unchanged components
5. track in @for               → Efficient list updates
6. Virtual scrolling           → Handle 10,000+ items
7. NgOptimizedImage            → LCP optimization
8. Pure pipes                  → Avoid template method calls
9. takeUntilDestroyed          → No memory leaks
10. Bundle budgets             → Prevent gradual bloat
```

---

## 14. Signals + Zoneless — The Foundation

```typescript
// The 2026 gold standard component
@Component({
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    @if (productsResource.isLoading()) {
      <app-skeleton [count]="pageSize()" />
    }
    @for (product of products(); track product.id) {
      <app-product-card [product]="product" />
    }
  `,
})
export class ProductListComponent {
  pageSize = input(20);

  productsResource = httpResource<Product[]>(
    () => `/api/products?limit=${this.pageSize()}`
  );

  products = computed(() => this.productsResource.value() ?? []);
}
```

**Why it's fast:**
- Zoneless: no Zone.js overhead (~13kB saved)
- `httpResource`: built-in loading/error state, no manual subscriptions
- `OnPush`: only re-renders when signals change
- `computed`: memoized derived value, not recalculated on every render

---

## 15. `track` in `@for` — Required Optimization

The `track` expression in `@for` is **required** (unlike optional `trackBy` in `*ngFor`). It tells Angular which item identity to use when the list changes:

```html
<!-- ✅ CORRECT — track by unique identifier -->
@for (product of products(); track product.id) {
  <app-product-card [product]="product" />
}

<!-- ✅ For primitive arrays -->
@for (tag of tags(); track tag) {
  <span class="tag">{{ tag }}</span>
}

<!-- ✅ Track by index (use when items have no stable ID) -->
@for (item of items(); track $index) {
  <li>{{ item }}</li>
}
```

### Why track Matters

```
Without track: list of 100 items, 1 item inserted at top
→ Angular destroys and recreates 100 DOM nodes

With track product.id: list of 100 items, 1 item inserted at top
→ Angular creates 1 new DOM node, moves 100 existing nodes
→ ~100× faster for large lists
```

---

## 16. Pure Pipes Over Methods

Methods in templates are called **on every change detection cycle**:

```html
<!-- ❌ BAD — formatPrice() called on EVERY CD cycle -->
<span>{{ formatPrice(product.price) }}</span>

<!-- ❌ BAD — same problem with getters -->
<span>{{ formattedPrice }}</span>  <!-- if it's a computed getter -->
```

### Solution 1: Pure Pipe

```typescript
@Pipe({ name: 'formatPrice', pure: true, standalone: true })
export class FormatPricePipe implements PipeTransform {
  transform(price: number, currency = 'USD'): string {
    return new Intl.NumberFormat('en-US', {
      style: 'currency',
      currency,
    }).format(price);
  }
}
```

```html
<!-- ✅ GOOD — pipe only re-executes when price changes -->
<span>{{ product.price | formatPrice }}</span>
```

### Solution 2: computed() Signal

```typescript
formattedPrice = computed(() =>
  new Intl.NumberFormat('en-US', { style: 'currency', currency: 'USD' })
    .format(this.product().price)
);
```

```html
<span>{{ formattedPrice() }}</span>
```

---

## 17. `@defer` — Eliminate Initial Bundle Weight

`@defer` is the most impactful single optimization for initial load:

```html
<!-- Heavy dashboard with charts — loaded only when visible -->
@defer (on viewport) {
  <app-analytics-dashboard />
} @placeholder {
  <div class="chart-placeholder" style="height: 400px;"></div>
}

<!-- Comments section — loaded when user scrolls to it -->
@defer (on viewport; prefetch on idle) {
  <app-comments [postId]="postId()" />
} @loading (minimum 300ms) {
  <app-comments-skeleton />
} @error {
  <p>Failed to load comments</p>
}

<!-- Heavy markdown editor — loaded only on interaction -->
@defer (on interaction) {
  <app-rich-text-editor [(content)]="body" />
} @placeholder {
  <div class="editor-placeholder" (click)="void 0">
    Click to edit...
  </div>
}
```

### Defer Triggers Reference

```html
@defer (on idle)         { ... }  <!-- When browser is idle -->
@defer (on viewport)     { ... }  <!-- When element enters viewport -->
@defer (on interaction)  { ... }  <!-- When user clicks/focuses -->
@defer (on hover)        { ... }  <!-- When user hovers -->
@defer (on immediate)    { ... }  <!-- ASAP, after initial render -->
@defer (on timer(2000))  { ... }  <!-- After 2 seconds -->
@defer (when isLoggedIn()) { ... } <!-- When signal/condition is true -->

<!-- Prefetch separately from display -->
@defer (on viewport; prefetch on idle) { ... }
```

---

## 18. CDK Virtual Scrolling

For lists with 500+ items, render only visible items:

```typescript
import { ScrollingModule } from '@angular/cdk/scrolling';

@Component({
  standalone: true,
  imports: [ScrollingModule],
  template: `
    <cdk-virtual-scroll-viewport
      itemSize="72"              
      class="viewport"
      style="height: 600px;"
    >
      @for (item of items; track item.id) {
        <app-list-item *cdkVirtualFor="let item of items" [item]="item" />
      }
    </cdk-virtual-scroll-viewport>
  `,
})
export class VirtualListComponent {
  items = Array.from({ length: 10000 }, (_, i) => ({
    id: i,
    name: `Item ${i}`,
  }));
}
```

### Variable Size Items

```typescript
<cdk-virtual-scroll-viewport [itemSize]="50" [minBufferPx]="200" [maxBufferPx]="400">
  <div *cdkVirtualFor="let item of items; templateCacheSize: 0">
    {{ item.name }}
  </div>
</cdk-virtual-scroll-viewport>
```

---

## 19. NgOptimizedImage

`NgOptimizedImage` automatically:
- Generates `srcset` for responsive images
- Adds `loading="lazy"` (except LCP images)
- Adds `fetchpriority="high"` for LCP images
- Automatically generates `preconnect` hints
- Warns about missing `width`/`height` (prevent CLS)

```typescript
import { NgOptimizedImage } from '@angular/common';

@Component({
  standalone: true,
  imports: [NgOptimizedImage],
  template: `
    <!-- LCP image — loads with high priority -->
    <img ngSrc="/hero.webp" width="1200" height="630" priority />

    <!-- Lazy-loaded product images -->
    @for (product of products(); track product.id) {
      <img
        [ngSrc]="product.imageUrl"
        [width]="300"
        [height]="300"
        [alt]="product.name"
      />
    }

    <!-- With image CDN (auto-resizing) -->
    <img ngSrc="product-123.jpg" width="400" height="400"
         loaderParams='{ "format": "webp", "quality": 85 }' />
  `,
})
```

### Image CDN Loader

```typescript
// Cloudinary loader
import { provideImageKitLoader } from '@angular/common';

providers: [
  provideImageKitLoader('https://ik.imagekit.io/your-account/'),
]

// Imgix, Cloudflare, custom loader also available
```

---

## 20. `afterNextRender` — Defer DOM Reads

Never read from the DOM in constructors or `ngOnInit` — they may run on the server (SSR) or before the element is rendered:

```typescript
@Component({ ... })
export class ResponsiveComponent {
  private el = inject(ElementRef);
  containerWidth = signal(0);

  constructor() {
    // ✅ Reads DOM after render is complete — safe for SSR
    afterNextRender(() => {
      this.containerWidth.set(this.el.nativeElement.offsetWidth);
    });

    // ✅ Updates on every render
    afterRender(() => {
      // Batched with other renders — efficient
    }, { phase: AfterRenderPhase.Read });
  }
}
```

---

## 21. Tree-Shaking Services

```typescript
// ✅ Tree-shakeable — only included if injected somewhere
@Injectable({ providedIn: 'root' })
export class AnalyticsService {}

// ❌ Always bundled — included even if never used
@NgModule({ providers: [AnalyticsService] })
export class AppModule {}
```

**All standalone services should use `providedIn: 'root'`** — the bundler removes them if unused.

---

## 22. `takeUntilDestroyed` — No More Memory Leaks

```typescript
import { takeUntilDestroyed, DestroyRef } from '@angular/core/rxjs-interop';

@Component({ ... })
export class DataComponent {
  private destroyRef = inject(DestroyRef);

  ngOnInit() {
    // ✅ Auto-unsubscribes when component destroys
    this.dataService.stream$.pipe(
      takeUntilDestroyed(this.destroyRef)
    ).subscribe(data => this.data.set(data));

    // ✅ Even simpler — inject destroyRef implicitly
    this.dataService.stream$.pipe(
      takeUntilDestroyed() // Works when called in injection context
    ).subscribe(data => this.data.set(data));
  }
}
```

---

## 23. `withFetch()` — Smaller Bundle, SSR Ready

```typescript
provideHttpClient(withFetch())
```

- Removes `XMLHttpRequest`-based adapter (~5kB)
- Uses native `fetch()` — works in SSR environments
- Required for Angular 21's Transfer State automatic handling

---

## 24. Font Optimization

```json
// angular.json
{
  "optimization": {
    "fonts": {
      "inline": true   // Inlines font @font-face into CSS — avoids render-blocking request
    }
  }
}
```

Additionally in HTML:
```html
<!-- Preconnect to Google Fonts -->
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
```

---

## 25. Hydration Deferral — Islands Pattern

Combine SSR + `@defer` for zero JavaScript on static content:

```html
<!-- Server renders this immediately — no JS hydration needed -->
<app-product-hero [product]="featuredProduct" />
<app-product-description [product]="product" />
<app-product-specs [specs]="product.specs" />

<!-- Only interactive parts load JavaScript -->
@defer (hydrate on interaction) {
  <app-add-to-cart [product]="product" />
}

@defer (hydrate on viewport) {
  <app-reviews [productId]="product.id" />
}

@defer (hydrate never) {
  <app-footer />
}
```

---

## 26. Preloading Strategy

```typescript
// Custom selective preloader — only preload marked routes
@Injectable({ providedIn: 'root' })
export class SelectivePreloadStrategy implements PreloadingStrategy {
  preload(route: Route, load: () => Observable<any>): Observable<any> {
    return route.data?.['preload'] === true ? load() : of(null);
  }
}

provideRouter(routes, withPreloading(SelectivePreloadStrategy))

// Mark routes for preloading
{
  path: 'dashboard',
  loadComponent: () => import('./dashboard.component'),
  data: { preload: true },
}
```

---

## 27. Best Practices Checklist

```
□ provideZonelessChangeDetection() — remove Zone.js
□ ChangeDetectionStrategy.OnPush on ALL components
□ Signal inputs (input()) instead of @Input()
□ computed() instead of methods in templates
□ Pure pipes instead of template function calls
□ @defer for below-fold and heavy components
□ track expression on every @for loop
□ NgOptimizedImage for all images (ngSrc)
□ CDK virtual scrolling for lists > 200 items
□ Bundle budgets set in angular.json
□ takeUntilDestroyed for all Observable subscriptions
□ providedIn: 'root' for all services (tree-shakeable)
□ provideHttpClient(withFetch())
□ Lazy-loaded routes for all features
□ source-map-explorer monthly bundle analysis
□ afterNextRender for DOM reads (no ngAfterViewInit DOM access)
```

---

> **Summary (12.6):** Angular's application builder (esbuild + Vite) delivers 3–5× faster builds than Webpack. Key features: persistent build cache, bundle budgets, `define` for compile-time constants, and source map analysis. Angular 21 adds experimental Rolldown for even faster production builds.

> **Summary (12.7):** Angular performance in 2026 is dominated by Signals + Zoneless (the architectural foundation), `@defer` (the biggest single-optimization tool), and the combination of `OnPush`, `track`, `pure pipes`, and `NgOptimizedImage`. Layer in virtual scrolling, hydration deferral, and bundle analysis for world-class performance.

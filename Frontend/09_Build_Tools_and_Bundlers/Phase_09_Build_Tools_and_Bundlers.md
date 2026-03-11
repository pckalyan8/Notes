# Phase 9 — Build Tools & Bundlers
## Frontend Angular Mastery Roadmap
> **Duration:** 2–3 weeks | **Goal:** Understand the build pipeline deeply

---

## Table of Contents

1. [9.1 — Why Build Tools?](#91--why-build-tools)
2. [9.2 — Webpack](#92--webpack)
3. [9.3 — Vite (v5/v6)](#93--vite-v5v6)
4. [9.4 — esbuild](#94--esbuild)
5. [9.5 — Rolldown (Emerging — 2025+)](#95--rolldown-emerging--2025)
6. [9.6 — Rollup](#96--rollup)
7. [9.7 — Babel](#97--babel)
8. [9.8 — SWC (Speedy Web Compiler)](#98--swc-speedy-web-compiler)
9. [9.9 — Linting & Formatting (2026 Stack)](#99--linting--formatting-2026-stack)
10. [9.10 — Task Automation & Monorepo Tooling](#910--task-automation--monorepo-tooling)
11. [Summary & Key Takeaways](#summary--key-takeaways)

---

## 9.1 — Why Build Tools?

### The Problem Build Tools Solve

When you open a modern Angular project and run `ng serve`, a remarkable chain of transformations happens in milliseconds: TypeScript files become JavaScript, SCSS becomes CSS, component templates are compiled and type-checked, modules are bundled together, dead code is eliminated, assets are fingerprinted with hashes, and a development server with live reload is started. All of this happens automatically, invisibly, and reliably.

None of this would be possible without build tools. To understand why they exist, it helps to understand the raw problems they solve — because each feature of a build tool is a direct answer to a specific limitation of the browser or the development process.

---

### Transpilation — ES2015+ → ES5 (and TypeScript → JavaScript)

**The problem:** JavaScript evolves through the ECMAScript specification. New features (arrow functions, classes, async/await, optional chaining, decorators) are added each year. But browsers update at different speeds, and enterprise environments may still be running Internet Explorer 11 or an older Chrome. Your code may use ES2025 syntax that a specific browser simply does not understand.

**The solution — transpilation:** A transpiler reads your modern source code and outputs an older, equivalent version that runs everywhere you need to support. The most famous transpiler is Babel. TypeScript's compiler (`tsc`) also transpiles — it takes TypeScript and produces JavaScript. Modern tools like esbuild and SWC do the same thing but at dramatically higher speed.

```typescript
// Source: TypeScript with modern ES2022+ features
async function fetchUser(id?: number): Promise<User | null> {
  const user = await userService.getById(id ?? 0);
  return user?.active ? user : null;
}

// After transpilation to ES5 (what older browsers understand):
"use strict";
var __awaiter = (this && this.__awaiter) || function(thisArg, _arguments, P, generator) {
  // ... complex Promise polyfill...
};
function fetchUser(id) {
  return __awaiter(this, void 0, void 0, function() {
    var user;
    return __generator(this, function(_a) {
      switch (_a.label) {
        case 0:
          return [4, userService.getById(id !== null && id !== void 0 ? id : 0)];
        case 1:
          user = _a.sent();
          return [2, (user === null || user === void 0 ? void 0 : user.active) ? user : null];
      }
    });
  });
}
```

**In Angular today:** Angular targets ES2022+ for modern browsers. TypeScript handles the TypeScript-to-JavaScript transformation, and esbuild handles syntax lowering for browser targets. You rarely configure transpilation manually — Angular CLI manages it through `tsconfig.json`'s `target` field.

**`browserslist`** is the configuration that tells transpilers and autoprefixers which browsers to support. It lives in a `.browserslistrc` file or in `package.json`:

```
# .browserslistrc — Angular's default (modern browsers only)
last 2 Chrome versions
last 1 Firefox version
last 2 Edge versions
last 2 Safari versions
last 2 iOS versions
not dead
```

The `browserslist` query is evaluated against the Can I Use database. When Angular builds, it checks this list to determine what JavaScript syntax and CSS features are safe to use and what needs polyfilling or transpiling.

---

### Module Bundling

**The problem:** A real Angular application imports thousands of modules — your own components, Angular framework code, RxJS operators, third-party libraries. Each `import` statement is a separate file. If the browser had to make a separate HTTP request for every file, a typical application might require thousands of round trips just to start up — even with HTTP/2, this creates significant overhead from request setup, DNS lookups, and header processing.

**The solution — bundling:** A bundler traces the entire import graph starting from your entry point (`main.ts`), resolves every `import` statement to its source file, and combines everything into one (or a few) larger JavaScript files. Instead of thousands of requests, the browser downloads a handful of optimised bundles.

```
Entry point: main.ts
  imports: app.component.ts
    imports: auth.service.ts
      imports: @angular/core (from node_modules)
      imports: rxjs/operators (from node_modules)
    imports: user.component.ts
      imports: user.service.ts
        imports: @angular/common/http

→ Bundle output: main-abc123.js (all combined into one file with a hash)
```

The bundler also handles **circular dependencies**, resolves **path aliases** from `tsconfig.json`, processes **side effects** correctly, and ensures each module's code executes exactly once.

---

### Minification and Uglification

**The problem:** Raw source code is written for human readability — long variable names, generous whitespace, comments, and descriptive identifiers. This verbosity is wasted bytes over the network.

**The solution — minification:** Removes all unnecessary whitespace, comments, and line breaks. **Uglification** (also called mangling) goes further and renames variables to the shortest possible names. Together, they can reduce JavaScript file size by 60–80%.

```javascript
// Original source (readable, 180 bytes)
function calculateUserDiscount(user, basePrice) {
  const discountRate = user.isPremium ? 0.20 : 0.05;
  const discountAmount = basePrice * discountRate;
  return basePrice - discountAmount;
}

// After minification + uglification (~65 bytes)
function c(u,p){let r=u.isPremium?.2:.05;return p-p*r}

// After further compression with Terser (esbuild's default):
c=(u,p)=>p*(1-(u.isPremium?.2:.05))
```

Modern minifiers are also **semantically aware** — they understand JavaScript's behavior well enough to perform optimisations like constant folding (replacing `2 * 3` with `6`), dead code elimination within functions, and inlining single-use variables.

---

### Tree Shaking — Dead Code Elimination

**The problem:** When you `import { Component } from '@angular/core'`, you're importing from a package that contains hundreds of functions and classes. Most of them you never use. Without tree shaking, they'd all end up in your bundle, massively inflating its size.

**The solution — tree shaking:** The bundler statically analyses the import/export graph and identifies which exports are actually used anywhere in the application. Exports that are never imported anywhere are "dead code" and are removed (shaken out of the tree) from the final bundle.

Tree shaking is only possible with **ES Modules** (which have a static, analysable structure). CommonJS `require()` is dynamic — a bundler can't know at build time what a `require()` will load, so CommonJS modules cannot be tree-shaken.

```typescript
// Your code
import { signal, computed } from '@angular/core';
// You never import: Injectable, Component, Directive, Pipe, etc.

// After tree shaking, ONLY the code for signal() and computed()
// ends up in the bundle. Hundreds of other Angular exports are eliminated.

// BAD — prevents tree shaking for this import:
import * as _ from 'lodash';       // bundler must include all of lodash
_.sortBy(users, 'name');

// GOOD — enables tree shaking:
import { sortBy } from 'lodash-es'; // bundler includes only sortBy
sortBy(users, 'name');
```

**`sideEffects` field in `package.json`:** To enable aggressive tree shaking, library authors add a `sideEffects: false` field to their `package.json`, telling bundlers "none of the files in this package have global side effects — feel free to remove anything not explicitly imported."

```json
// Library's package.json
{
  "name": "my-pure-library",
  "sideEffects": false  // safe to tree-shake any unused export
}

// Or specify files that DO have side effects:
{
  "sideEffects": [
    "*.css",
    "./src/polyfills.js"
  ]
}
```

---

### Code Splitting

**The problem:** A single large JavaScript bundle means the browser must download, parse, and execute all of the application's code before anything can be displayed — even for features the user might never visit.

**The solution — code splitting:** The bundler creates multiple smaller bundles (chunks) and loads them on demand. The initial bundle contains only what is needed to display the first page. Other chunks are loaded lazily when needed.

Angular implements code splitting at the route level using dynamic `import()`:

```typescript
// routes.ts — route-level code splitting
export const routes: Routes = [
  {
    path: '',
    loadComponent: () => import('./home/home.component')
      .then(m => m.HomeComponent)
  },
  {
    path: 'admin',
    // AdminModule and ALL its dependencies are a separate chunk
    // Only downloaded when the user navigates to /admin
    loadChildren: () => import('./admin/admin.routes')
      .then(m => m.ADMIN_ROUTES)
  },
  {
    path: 'reports',
    loadComponent: () => import('./reports/reports.component')
      .then(m => m.ReportsComponent)
  }
];

// Angular also uses @defer for component-level code splitting:
@Component({
  template: `
    @defer (on viewport) {
      <!-- This entire component and its dependencies become a separate chunk -->
      <!-- Downloaded only when it scrolls into the viewport -->
      <app-heavy-analytics-dashboard />
    } @placeholder {
      <div class="skeleton"></div>
    }
  `
})
export class PageComponent {}
```

The build output for a code-split Angular app looks like:

```
dist/browser/
├── main-3f2a8b9c.js          ← initial bundle (framework + root component)
├── chunk-admin-a1b2c3d4.js   ← admin module (lazy loaded)
├── chunk-reports-e5f6g7h8.js ← reports component (lazy loaded)
├── chunk-dashboard-9i0j1k2l.js ← @defer deferred block
├── styles-7m8n9o0p.css       ← global styles
└── index.html
```

---

### Asset Handling

**The problem:** Applications use many types of assets beyond JavaScript — images, fonts, SVG files, JSON data, CSS, video. Each needs different treatment: images should be optimised and fingerprinted, fonts should be preloaded, SVGs can be inlined or referenced.

**The solution:** Build tools include asset processing pipelines:

```
Image files (.jpg, .png, .webp):
  → Optimise (reduce size)
  → Content hash added to filename: logo-a1b2c3d4.png
  → Copied to output directory

Font files (.woff, .woff2):
  → Content hash added to filename
  → Optionally inlined as base64 for small fonts

CSS/SCSS files:
  → Preprocessed (SCSS → CSS)
  → Autoprefixed (vendor prefixes added where needed)
  → Minified
  → Content hash added

SVG files:
  → Optionally optimised with SVGO
  → Can be inlined into HTML or referenced as external file

JSON files:
  → Can be imported directly in TypeScript and inlined into bundles
```

**Content hashing** is critical for cache busting. The filename `main-3f2a8b9c.js` contains a hash of the file's content. When you deploy a new version, the hash changes. The browser's cache sees it as a completely new file and downloads it. Meanwhile, unchanged files (like vendor libraries that haven't been updated) keep their old hash and are served from cache — zero download cost for the user.

---

### Source Maps

**The problem:** The minified, bundled code that runs in production looks nothing like the TypeScript source you wrote. When an error occurs, the stack trace points to line 1, column 48,293 of `main-3f2a8b9c.js` — completely useless for debugging.

**The solution — source maps:** A source map is a JSON file that maps positions in the compiled output back to positions in the original source files. Developer tools (and error monitoring services like Sentry) use source maps to show you the original TypeScript code in stack traces and during debugging.

```json
// main-3f2a8b9c.js.map — a source map file (simplified)
{
  "version": 3,
  "sources": [
    "src/app/auth/auth.service.ts",
    "src/app/core/http.service.ts"
  ],
  "mappings": "AAAA,IAAI,CAAC,GAAG...",  // VLQ-encoded position mappings
  "sourcesContent": [
    "// original TypeScript source...",
    "// original TypeScript source..."
  ]
}
```

**Source map types and their trade-offs:**

| Type | Build Speed | Debug Quality | Included in Bundle? |
|---|---|---|---|
| `false` | Fastest | None | No |
| `eval` | Fast | Approximate | Inline (dev only) |
| `source-map` | Slow | Full, original source | Separate `.map` file |
| `hidden-source-map` | Slow | Full (not exposed to browser) | Separate `.map` file |
| `inline-source-map` | Slow | Full | Inline base64 in bundle |

For **development**: use `eval-source-map` (fast rebuild, good quality) or `inline-source-map`.
For **production**: use `hidden-source-map` — the `.map` file is generated but not referenced in the bundle. Upload it to Sentry or your error monitoring service. Users cannot access your source code, but you get full stack traces in your monitoring dashboard.

> **Best Practice:** Never expose source maps publicly in production by using `source-map` mode. Use `hidden-source-map` and upload maps to your error monitoring service. Angular CLI configures this correctly by default — `sourceMap: true` in development, `sourceMap: { scripts: true, hidden: true }` in production when you explicitly enable it.

---

## 9.2 — Webpack

### What Is Webpack and Why Learn It?

Webpack is a JavaScript module bundler first released in 2012 and dominant from roughly 2015 to 2022. Even though Angular migrated away from Webpack to esbuild in Angular 17, understanding Webpack remains essential for three reasons:

1. **Millions of existing projects** (Angular apps built before v17, React, Vue) still use Webpack
2. **Conceptual foundation** — Webpack introduced concepts (loaders, plugins, entry/output, HMR) that all modern build tools are designed around or in reaction to
3. **Custom builds** — `@angular-builders/custom-webpack` still allows Webpack for edge cases in Angular, and you may need to customise it in corporate environments

---

### Core Concepts: Entry, Output, Mode

A Webpack configuration is a JavaScript file (`webpack.config.js`) that exports a configuration object. The four fundamental properties are `entry`, `output`, `mode`, and `module` (which contains loaders):

```javascript
// webpack.config.js — fundamental configuration
const path = require('path');
const HtmlWebpackPlugin = require('html-webpack-plugin');

module.exports = {
  // mode: tells Webpack which optimisations to apply
  // 'development' → readable output, source maps, fast rebuild
  // 'production'  → minification, tree shaking, optimised chunks
  // 'none'        → no built-in optimisations
  mode: 'production',

  // entry: the starting point of your application
  // Webpack traces all imports from here to build the dependency graph
  entry: './src/main.ts',

  // Multiple entry points (for multi-page applications):
  // entry: {
  //   main: './src/main.ts',
  //   admin: './src/admin/main.ts',
  // },

  // output: where and how to write the compiled files
  output: {
    path: path.resolve(__dirname, 'dist'),
    filename: '[name].[contenthash:8].js',  // content-hashed filename
    chunkFilename: '[name].[contenthash:8].chunk.js',  // lazy chunks
    publicPath: '/',                         // base URL for all assets
    clean: true,                             // clean dist/ before each build
  },

  // resolve: how Webpack resolves imports
  resolve: {
    extensions: ['.ts', '.tsx', '.js', '.json'],
    alias: {
      '@app': path.resolve(__dirname, 'src/app'),
      '@env': path.resolve(__dirname, 'src/environments'),
    },
  },
};
```

---

### Loaders — Transforming Files Before Bundling

Webpack natively understands only JavaScript and JSON. **Loaders** are transformations that Webpack applies to files before adding them to the dependency graph. They convert TypeScript, CSS, images, and other file types into valid JavaScript modules.

```javascript
module.exports = {
  module: {
    rules: [
      // TypeScript → JavaScript via ts-loader
      {
        test: /\.ts$/,           // regex: apply this loader to .ts files
        exclude: /node_modules/,
        use: [
          {
            loader: 'ts-loader',
            options: {
              transpileOnly: true,  // skip type checking (use tsc separately)
            },
          },
        ],
      },

      // CSS → JavaScript (with style injection)
      // Loaders are applied RIGHT TO LEFT:
      // 1. css-loader: resolves @import and url() in CSS, returns JS module
      // 2. style-loader: injects the CSS into a <style> tag at runtime
      {
        test: /\.css$/,
        use: ['style-loader', 'css-loader'],
      },

      // SCSS → CSS → JavaScript
      {
        test: /\.scss$/,
        use: [
          'style-loader',   // 3. inject into <style> tag
          'css-loader',     // 2. resolve imports in CSS
          'sass-loader',    // 1. compile SCSS to CSS first
        ],
      },

      // CSS with file extraction (for production — creates a separate .css file)
      {
        test: /\.css$/,
        use: [
          MiniCssExtractPlugin.loader,  // extracts CSS to separate file
          'css-loader',
        ],
      },

      // Images
      // url-loader: inlines small images as base64 data URLs,
      //             emits larger images as separate files
      {
        test: /\.(png|jpg|gif|svg)$/,
        use: [
          {
            loader: 'url-loader',
            options: {
              limit: 8192,         // files < 8KB → inline as base64
              name: '[name].[contenthash:8].[ext]',
              outputPath: 'assets/',
            },
          },
        ],
      },

      // Modern replacement: asset modules (Webpack 5 built-in, no loader needed)
      {
        test: /\.(png|jpg|gif|svg)$/,
        type: 'asset',             // 'asset/resource', 'asset/inline', 'asset'
        parser: {
          dataUrlCondition: {
            maxSize: 8 * 1024,     // < 8KB → inline, >= 8KB → file
          },
        },
      },

      // Fonts
      {
        test: /\.(woff|woff2|eot|ttf|otf)$/,
        type: 'asset/resource',    // always emit as separate file
        generator: {
          filename: 'fonts/[name].[contenthash:8][ext]',
        },
      },
    ],
  },
};
```

**The critical loader order rule:** Loaders in the `use` array are applied **right-to-left** (or bottom-to-top). For SCSS: first `sass-loader` runs (SCSS→CSS), then `css-loader` runs (processes CSS), then `style-loader` runs (injects into DOM). Getting this order wrong causes cryptic errors.

---

### Plugins — Extending Webpack's Capabilities

While loaders transform individual files, **plugins** hook into the entire compilation lifecycle and can perform broader operations — generating HTML, extracting CSS to files, injecting environment variables, and more.

```javascript
const HtmlWebpackPlugin = require('html-webpack-plugin');
const MiniCssExtractPlugin = require('mini-css-extract-plugin');
const CssMinimizerPlugin = require('css-minimizer-webpack-plugin');
const { DefinePlugin } = require('webpack');

module.exports = {
  plugins: [
    // HtmlWebpackPlugin: generates index.html and automatically
    // injects <script> and <link> tags for all output bundles
    new HtmlWebpackPlugin({
      template: './src/index.html',   // use your template
      filename: 'index.html',
      minify: {
        collapseWhitespace: true,
        removeComments: true,
      },
    }),

    // MiniCssExtractPlugin: extracts CSS into separate .css files
    // (instead of injecting them into JS via style-loader)
    // Essential for production — CSS files can be cached separately
    new MiniCssExtractPlugin({
      filename: '[name].[contenthash:8].css',
      chunkFilename: '[name].[contenthash:8].chunk.css',
    }),

    // DefinePlugin: replaces string tokens in your source code at build time
    // Useful for injecting environment variables and feature flags
    new DefinePlugin({
      'process.env.NODE_ENV': JSON.stringify(process.env.NODE_ENV),
      '__APP_VERSION__': JSON.stringify(process.env.npm_package_version),
      '__API_URL__': JSON.stringify(process.env.API_URL || 'https://api.example.com'),
    }),
  ],

  optimization: {
    minimizer: [
      '...', // extend default minimizers (includes TerserPlugin)
      new CssMinimizerPlugin(),  // minify extracted CSS
    ],
  },
};
```

---

### Code Splitting: SplitChunksPlugin and Dynamic Imports

Webpack implements code splitting in two ways: automatic chunk splitting via `SplitChunksPlugin`, and manual splitting via dynamic `import()`.

**SplitChunksPlugin — automatic vendor chunk extraction:**

```javascript
module.exports = {
  optimization: {
    // SplitChunksPlugin is built into Webpack 5 — configure it here
    splitChunks: {
      chunks: 'all',  // split both sync and async chunks

      cacheGroups: {
        // Create a separate vendor chunk for node_modules
        // This chunk rarely changes → long cache lifetime
        vendors: {
          test: /[\\/]node_modules[\\/]/,
          name: 'vendors',
          chunks: 'all',
          priority: -10,
        },

        // Create a separate chunk for Angular framework code
        angular: {
          test: /[\\/]node_modules[\\/]@angular[\\/]/,
          name: 'angular',
          chunks: 'all',
          priority: 10,
        },

        // Create a separate chunk for code shared between multiple entry points
        common: {
          name: 'common',
          minChunks: 2,        // included in at least 2 chunks
          chunks: 'all',
          priority: -20,
          reuseExistingChunk: true,
        },
      },
    },

    // Separate the webpack runtime into its own tiny file
    // (prevents vendor chunk hash from changing on every build)
    runtimeChunk: 'single',
  },
};
```

**Dynamic imports — manual code splitting:**

```javascript
// Without dynamic import — everything loads upfront
import { AdminDashboard } from './admin/dashboard';

// With dynamic import — AdminDashboard becomes a separate chunk
// Only downloaded when showAdmin() is called
async function showAdmin() {
  const { AdminDashboard } = await import(
    /* webpackChunkName: "admin-dashboard" */  // name the chunk
    /* webpackPrefetch: true */                // hint: prefetch in background
    './admin/dashboard'
  );
  render(AdminDashboard);
}

// webpackPrefetch vs webpackPreload:
// prefetch: low-priority background download (after page load)
// preload: high-priority parallel download (during page load)
```

---

### Webpack Dev Server and Hot Module Replacement (HMR)

`webpack-dev-server` provides a development server with automatic rebuilding and **Hot Module Replacement (HMR)** — the ability to update modules in the running application without a full page reload.

```javascript
// webpack.config.js — dev server configuration
module.exports = {
  devServer: {
    port: 4200,
    open: true,           // open browser automatically
    hot: true,            // enable HMR
    historyApiFallback: true,  // for SPA routing — serve index.html for 404s
    proxy: [
      {
        context: ['/api'],
        target: 'http://localhost:3000',  // proxy API calls to avoid CORS in dev
        changeOrigin: true,
      },
    ],
    static: {
      directory: path.join(__dirname, 'public'),  // serve static files
    },
  },
};
```

**How HMR works:** When you save a file, Webpack recompiles only the changed module and its direct dependants. The dev server sends the new module code to the browser via a WebSocket. The browser **replaces the old module with the new one in memory** — without reloading the page. You keep your application state (form values, scroll position, open dialogs) while seeing the code change take effect instantly.

HMR is not magic — it requires the module being updated to have HMR handler code (or for the framework to provide it). Angular, React (with react-refresh), and Vue all have HMR support built in.

---

### Bundle Analysis with webpack-bundle-analyzer

Understanding what is in your bundle is critical for performance optimisation. `webpack-bundle-analyzer` creates an interactive treemap visualisation of your bundle contents:

```bash
# Install
npm install --save-dev webpack-bundle-analyzer

# Generate stats file
npx webpack --profile --json > stats.json

# Visualise (opens in browser)
npx webpack-bundle-analyzer stats.json

# In Angular (uses esbuild now, but can still generate stats for analysis)
ng build --stats-json
npx webpack-bundle-analyzer dist/browser/stats.json
```

The visualisation shows each module as a rectangle sized proportionally to its compressed size. Common things to look for:

- Unexpectedly large third-party packages (moment.js being 65kB+ is a classic)
- Duplicate packages (lodash appearing twice from different transitive deps)
- Modules from `node_modules/` that should have been tree-shaken
- Your own modules that are larger than expected

---

## 9.3 — Vite (v5/v6)

### The Problem Vite Solves

As JavaScript applications grew larger, Webpack-based build processes became painfully slow. A cold start (first `npm start`) of a large Angular or React application could take 60–120 seconds. Every file save triggered a rebuild that took 3–10 seconds. Developer experience suffered badly.

The root cause: Webpack bundles everything together even in development mode. Before the dev server is ready, it must trace the entire dependency graph, apply all loaders to every file, bundle everything into chunks, and only then start serving. For an application with 3,000 modules, this is a lot of work.

**Evan You (creator of Vue.js) created Vite in 2020** based on a key insight: modern browsers natively support ES Modules. In development, you don't need to bundle at all — the browser can request individual module files directly.

---

### ESM-Based Dev Server — No Bundling in Development

Vite's development server works fundamentally differently from Webpack:

**Webpack approach:**
```
Save file → Webpack rebuilds entire affected subgraph → 
Updates bundle in memory → Browser reloads chunk
(seconds to tens of seconds)
```

**Vite approach:**
```
Save file → Vite transforms ONLY that single file → 
Browser requests the updated file via ESM → 
HMR updates just that module in memory
(milliseconds)
```

When you start `vite dev`, the server starts immediately — no upfront bundling. When the browser loads your application and encounters `import { signal } from '@angular/core'`, it sends an HTTP request to the Vite dev server for that specific module. Vite transforms the file on demand (TypeScript → JavaScript, SCSS → CSS) and serves it. The browser handles the module graph resolution natively.

```
Browser request: GET /src/app/app.component.ts
Vite: transforms TypeScript to JavaScript (esbuild, ~1ms)
Vite: rewrites imports to URLs the browser can resolve
Vite: serves the transformed file
Browser: makes further requests for each import found in the file
```

This "request-driven" approach means startup time is near-instant regardless of application size. The first meaningful render may take slightly longer than with a bundled approach (more HTTP requests), but developer iteration speed — the time from saving a file to seeing the change — is dramatically faster.

---

### esbuild for Pre-bundling Dependencies

There is one exception to Vite's "no bundling" rule: **dependencies** (code in `node_modules/`). These are pre-bundled with esbuild when the dev server first starts (or when dependencies change).

Why pre-bundle dependencies?

1. **CommonJS compatibility:** Many npm packages use CommonJS (`require()`), which browsers cannot import natively. esbuild converts them to ESM.
2. **Performance:** Some packages (like lodash) consist of hundreds of tiny files. Each would require a separate browser request. esbuild bundles them into a single file.
3. **Caching:** Pre-bundled deps are cached in `node_modules/.vite/deps/`. Subsequent dev server starts skip pre-bundling unless deps change.

```javascript
// vite.config.ts
import { defineConfig } from 'vite';

export default defineConfig({
  optimizeDeps: {
    // Force include a package that Vite's auto-detection misses
    include: ['some-commonjs-package'],

    // Exclude a package from pre-bundling (e.g., already pure ESM)
    exclude: ['pure-esm-package'],

    // Force re-bundle (useful when you've changed a linked local package)
    force: false,
  },
});
```

---

### Rollup for Production Builds

While the dev server uses native ESM, Vite uses **Rollup** for production builds. The reasons:

1. **Bundle optimisation:** Despite HTTP/2, sending thousands of tiny files in production creates overhead. A properly bundled output is more efficient.
2. **Tree shaking:** Rollup has excellent, granular tree shaking.
3. **Code splitting:** Rollup's chunk splitting is highly configurable.
4. **CSS handling:** Rollup (via Vite plugins) can extract and optimise CSS.

```javascript
// vite.config.ts — production build configuration
import { defineConfig } from 'vite';
import angular from '@analogjs/vite-plugin-angular';

export default defineConfig({
  plugins: [angular()],

  build: {
    outDir: 'dist',
    sourcemap: false,         // or 'hidden' for Sentry upload
    minify: 'esbuild',        // use esbuild for JS minification (fast)
    cssMinify: true,

    rollupOptions: {
      output: {
        // Manual chunk splitting
        manualChunks: {
          'vendor-angular': ['@angular/core', '@angular/common', '@angular/router'],
          'vendor-rxjs': ['rxjs'],
        },
        // Asset naming
        assetFileNames: 'assets/[name].[hash][extname]',
        chunkFileNames: 'chunks/[name].[hash].js',
        entryFileNames: '[name].[hash].js',
      },
    },
  },
});
```

---

### Hot Module Replacement in Vite

Vite's HMR is significantly faster than Webpack's because only the changed module needs to be re-transformed and sent to the browser. When you save a component file, Vite:

1. Re-transforms only that file (~1ms with esbuild)
2. Sends the new module via WebSocket
3. The HMR runtime replaces the old module in the browser's module registry
4. Framework-level HMR (via framework plugins) preserves component state where possible

```javascript
// Vite exposes HMR API for manual module hot replacement:
if (import.meta.hot) {
  import.meta.hot.accept('./some-module.ts', (newModule) => {
    // Called when ./some-module.ts is updated
    // newModule is the updated module
    replaceWithNewModule(newModule);
  });

  import.meta.hot.dispose((data) => {
    // Called before the module is replaced — clean up side effects
    clearInterval(data.intervalId);
  });
}
```

---

### Vite 5 and Vite 6 — Key Changes

**Vite 5 (released November 2023):**
- Dropped support for Rollup 3, migrated to Rollup 4 (faster)
- Improved SSR (Server-Side Rendering) with better environment separation
- `Environment API` introduced as experimental
- Better CSS handling and import optimisation
- Requires Node.js 18+

**Vite 6 (released November 2024):**
- **Environment API stable** — allows plugins to handle multiple environments (client, SSR, edge) with different configurations in a single Vite process
- **Experimental Rolldown integration** — Rolldown (Rust-based bundler) available as the production bundler instead of Rollup. Dramatically faster production builds.
- **CSS improvements** — better handling of CSS Modules, PostCSS
- Requires Node.js 18.0+

```javascript
// Vite 6 — Environment API example
export default defineConfig({
  environments: {
    client: {
      build: { outDir: 'dist/client' },
    },
    ssr: {
      build: { outDir: 'dist/server', ssr: true },
    },
    edge: {
      build: { outDir: 'dist/edge', target: 'chrome89' },
    },
  },
});
```

---

### Vite Plugin Ecosystem

Vite's plugin system is compatible with Rollup plugins (with some limitations), giving it a rich ecosystem:

```javascript
// vite.config.ts — common plugins
import { defineConfig } from 'vite';
import { visualizer } from 'rollup-plugin-visualizer'; // bundle analysis
import viteCompression from 'vite-plugin-compression';  // gzip/brotli output
import { createSvgIconsPlugin } from 'vite-plugin-svg-icons'; // SVG sprite
import checker from 'vite-plugin-checker'; // TypeScript type checking in dev

export default defineConfig({
  plugins: [
    // Bundle visualiser (generates treemap like webpack-bundle-analyzer)
    visualizer({
      open: true,
      filename: 'dist/stats.html',
      gzipSize: true,
    }),

    // Pre-compress output files (Gzip and Brotli)
    viteCompression({ algorithm: 'brotliCompress' }),

    // TypeScript type checking (runs in parallel, non-blocking)
    checker({ typescript: true }),
  ],
});
```

---

### Angular + Vite

Angular's build system architecture separates the **bundler** (esbuild) from the **dev server**. Since Angular 17:

- **Production builds:** Angular CLI uses esbuild directly (not Vite)
- **Development server:** Angular CLI uses a Vite-powered dev server with esbuild plugin

```bash
# Angular 17+ creates projects with the esbuild application builder by default
ng new my-app
# angular.json will have:
# "builder": "@angular-devkit/build-angular:application"
# which uses esbuild for bundling and Vite's dev server for development

# Analog.js — for full Vite integration with Angular
npm create analog@latest
# Uses @analogjs/vite-plugin-angular for native Vite Angular support
```

---

## 9.4 — esbuild

### What Is esbuild and Why Is It So Fast?

esbuild is a JavaScript and TypeScript bundler and minifier written in **Go** by Evan Wallace (co-founder of Figma). Released in 2020, it immediately benchmarked at 10–100× faster than any JavaScript-based bundler.

The speed comes from several engineering decisions:
1. **Go is compiled to native machine code** — no JIT warmup, no garbage collection pauses
2. **Parallelism** — uses all CPU cores simultaneously (JavaScript engines are single-threaded)
3. **Shared memory architecture** — Go routines share data without copying
4. **Written from scratch** — optimised for this specific use case, no legacy compatibility layers

```
Benchmark: Bundle 3-clause BSD license project (typical mid-size app)

esbuild:     0.33s
swc:         1.6s
Parcel 2:    15.8s
Rollup:      16.5s
Webpack:     38.2s

esbuild is ~115× faster than Webpack for the same task
```

---

### esbuild API — JavaScript and CLI

esbuild can be used as a CLI tool or as a programmatic JavaScript API:

```bash
# CLI usage
npx esbuild src/main.ts \
  --bundle \             # bundle all imports
  --outfile=dist/main.js \
  --minify \             # minify output
  --sourcemap \          # generate source map
  --target=chrome115 \   # target browser version
  --format=esm \         # output as ES module
  --splitting \          # enable code splitting
  --chunk-names=[name]-[hash] \
  --define:__DEV__=false    # replace constants
```

```javascript
// JavaScript API (used by Angular CLI, Vite, and other tools internally)
import * as esbuild from 'esbuild';

// Build API — full build
const result = await esbuild.build({
  entryPoints: ['src/main.ts'],
  bundle: true,
  outdir: 'dist',
  format: 'esm',
  splitting: true,            // code splitting (only works with format:'esm')
  minify: true,
  sourcemap: 'external',
  target: ['chrome115', 'firefox115', 'safari17'],
  define: {
    '__API_URL__': '"https://api.example.com"',
    'process.env.NODE_ENV': '"production"',
  },
  loader: {
    '.ts': 'ts',              // built-in TypeScript support
    '.tsx': 'tsx',
    '.css': 'css',
    '.svg': 'text',           // import SVG as string
    '.png': 'dataurl',        // import PNG as data URL
    '.woff2': 'file',         // copy font files to output
  },
  plugins: [
    sassPlugin(),             // third-party plugin for SCSS
  ],
  metafile: true,             // include bundle metadata for analysis
});

// Analyse the bundle (like webpack-bundle-analyzer but simpler)
console.log(await esbuild.analyzeMetafile(result.metafile));
```

```javascript
// Transform API — transform a single string (no file I/O)
// Used internally by Vite and other tools to transform files on demand
const result = await esbuild.transform(`
  const greeting: string = 'Hello, TypeScript!';
  console.log(greeting);
`, {
  loader: 'ts',
  target: 'es2022',
  minify: false,
});

console.log(result.code);
// Output: const greeting = "Hello, TypeScript!";
// console.log(greeting);
```

---

### Angular's Application Builder Uses esbuild (Angular 17+)

Angular 17 replaced the old Webpack-based builder with a new `application` builder that uses esbuild directly. The results were dramatic:

```
Build time comparison for a medium Angular application:

Angular 16 (Webpack):
  Cold production build:  45–90 seconds
  Incremental rebuild:    3–8 seconds

Angular 17+ (esbuild):
  Cold production build:  5–15 seconds
  Incremental rebuild:    200–500ms
```

The Angular CLI exposes esbuild through a thin configuration layer in `angular.json`:

```json
// angular.json — application builder configuration
{
  "projects": {
    "my-app": {
      "architect": {
        "build": {
          "builder": "@angular-devkit/build-angular:application",
          "options": {
            "outputPath": "dist/my-app",
            "index": "src/index.html",
            "browser": "src/main.ts",        // entry point
            "polyfills": ["zone.js"],
            "tsConfig": "tsconfig.app.json",
            "assets": ["src/favicon.ico", "src/assets"],
            "styles": ["src/styles.scss"],
            "scripts": [],
            "server": "src/main.server.ts",  // SSR entry point
            "prerender": true,               // static generation
            "ssr": { "entry": "server.ts" }  // SSR server
          },
          "configurations": {
            "production": {
              "budgets": [...],
              "outputHashing": "all",          // content hashing
              "optimization": true,            // minify, tree-shake
              "sourceMap": false,
              "namedChunks": false,
              "aot": true                      // Ahead-of-Time compilation
            },
            "development": {
              "optimization": false,
              "sourceMap": true,
              "namedChunks": true
            }
          }
        }
      }
    }
  }
}
```

---

### CSS Bundling with esbuild

esbuild supports CSS natively — it can bundle CSS files, resolve `@import` statements, and handle CSS Modules. However, it does not include a preprocessor for SCSS by default — that requires a plugin:

```javascript
// esbuild.config.mjs
import esbuild from 'esbuild';
import { sassPlugin } from 'esbuild-sass-plugin';  // third-party plugin

await esbuild.build({
  entryPoints: ['src/styles.scss'],
  outfile: 'dist/styles.css',
  bundle: true,
  minify: true,
  plugins: [sassPlugin()],
});
```

Angular CLI handles SCSS preprocessing separately (using the Dart Sass compiler) before passing CSS to esbuild.

---

### esbuild Limitations

esbuild is intentionally scoped to what it does best. Knowing its limitations prevents frustration:

1. **No TypeScript type checking** — esbuild strips types without checking them. Run `tsc --noEmit` separately for type checking (Angular CLI does this).
2. **Limited plugin ecosystem** compared to Webpack — fewer loaders/plugins available, but growing.
3. **No support for some Webpack features** — Webpack's `SplitChunksPlugin` with complex `cacheGroups` has no direct equivalent.
4. **CSS Modules are basic** — Webpack's `css-loader` with `modules: true` is more featureful.
5. **No legacy browser support** — does not generate IE11-compatible output. For IE11 support (rare in 2026), use Babel + Webpack.

---

## 9.5 — Rolldown (Emerging — 2025+)

### What Is Rolldown?

Rolldown is a Rust-based JavaScript bundler built to be a **drop-in replacement for Rollup**, the bundler currently used by Vite for production builds. It is being developed by the Vite team and Void(0) as the next-generation production bundler for the Vite ecosystem.

The goal is to achieve performance comparable to esbuild (Rust/Go native speed) while maintaining full compatibility with the Rollup plugin API — so existing Vite projects can migrate with zero configuration changes.

---

### Why Rolldown Matters

The current Vite architecture has a fundamental inconsistency: the dev server uses esbuild (extremely fast), but the production build uses Rollup (JavaScript-based, slower). This means dev and production use different bundlers with potentially different behaviour — a source of hard-to-reproduce bugs.

Rolldown resolves this by eventually serving as a single bundler for both dev and production:

```
Current Vite (v5):
  Dev server: esbuild (Go, fast)
  Production: Rollup (JavaScript, slower)
  → Two different bundlers = potential behaviour differences

Future Vite with Rolldown:
  Dev server: Rolldown (Rust, fast)
  Production: Rolldown (Rust, fast)
  → One bundler = consistent behaviour + maximum speed
```

---

### Rolldown's Rollup API Compatibility

Rolldown is designed to be compatible with Rollup's JavaScript API:

```javascript
// Rollup API (current)
import { rollup } from 'rollup';

const bundle = await rollup({
  input: 'src/main.ts',
  plugins: [typescript(), resolve(), commonjs()],
});

await bundle.write({
  file: 'dist/bundle.js',
  format: 'esm',
  sourcemap: true,
});

// Rolldown API (intended to be a drop-in replacement)
import { rolldown } from 'rolldown';  // same API, different package

const bundle = await rolldown({
  input: 'src/main.ts',
  plugins: [typescript(), resolve(), commonjs()],
});

await bundle.write({
  file: 'dist/bundle.js',
  format: 'esm',
  sourcemap: true,
});
```

Existing Rollup plugins (like `@rollup/plugin-commonjs`, `@rollup/plugin-node-resolve`, and most community plugins) work with Rolldown with little or no modification.

---

### Performance Benchmarks

Based on early 2025 benchmarks comparing Rolldown to Rollup on large codebases:

```
Bundling rome (large TypeScript codebase, ~1.3MB):
  Rollup 4:     7.7 seconds
  Rolldown:     0.8 seconds  (~9.6× faster)

Bundling three.js:
  Rollup 4:     2.8 seconds
  Rolldown:     0.4 seconds  (~7× faster)
```

---

### Integration Status in Vite 6

In Vite 6 (released November 2024), Rolldown is available as an **experimental opt-in**:

```javascript
// vite.config.ts — opt into Rolldown (Vite 6 experimental)
import { defineConfig } from 'vite';
// Import from the Rolldown-Vite package (separate from standard Vite)
// This is expected to merge into standard Vite in a future release

export default defineConfig({
  // Rolldown integration is transparent when enabled —
  // your configuration doesn't change
});
```

The expectation is that Rolldown will become the default production bundler in Vite 7 (2025–2026), with the current Rollup integration deprecated.

> **What this means for Angular developers:** Analog.js (the Angular meta-framework built on Vite) will benefit from Rolldown automatically. Standard Angular CLI (which uses esbuild, not Vite's Rollup) is unaffected — Angular's build speed is already esbuild-powered.

---

## 9.6 — Rollup

### Rollup's Place in the Ecosystem

Rollup is a JavaScript module bundler that pioneered **ES Module tree shaking** and remains the gold standard for **library bundling**. While Webpack dominates application bundling and esbuild dominates speed, Rollup is still the preferred choice for building distributable JavaScript libraries.

When you publish an npm package — an Angular component library, a utility library, a shared SDK — Rollup (or `ng-packagr` which wraps it) is typically what creates the optimised, tree-shakeable output.

---

### ES Module-First Philosophy

Rollup was built from the ground up around ES Modules. Unlike Webpack (which retrofitted ES Module support onto a CommonJS foundation), Rollup treats `import`/`export` as first-class citizens. This gives it superior tree-shaking — it analyses the static import graph at an extremely granular level.

```javascript
// math.js
export function add(a, b) { return a + b; }
export function multiply(a, b) { return a * b; }
export function divide(a, b) { return a / b; }
export function subtract(a, b) { return a - b; }

// app.js
import { add } from './math.js';  // only add() is used

// After Rollup tree-shaking — only add() is in the bundle:
function add(a, b) { return a + b; }
// multiply, divide, subtract → completely removed
```

For complex utility libraries with hundreds of exports, this produces dramatically smaller bundles than Webpack's approach.

---

### Output Formats

Rollup's killer feature for library authors is supporting multiple output formats from a single source:

```javascript
// rollup.config.mjs
import typescript from '@rollup/plugin-typescript';
import resolve from '@rollup/plugin-node-resolve';
import commonjs from '@rollup/plugin-commonjs';
import terser from '@rollup/plugin-terser';
import dts from 'rollup-plugin-dts';

export default [
  // ESM build — for modern bundlers (Vite, Webpack 5, esbuild)
  {
    input: 'src/index.ts',
    output: {
      file: 'dist/esm/index.mjs',
      format: 'esm',           // ES Module format
      sourcemap: true,
    },
    plugins: [
      resolve(),               // resolve node_modules imports
      commonjs(),              // convert CommonJS deps to ESM
      typescript({ tsconfig: './tsconfig.build.json' }),
      terser(),                // minify
    ],
    external: ['@angular/core', 'rxjs'],  // don't bundle peer deps
  },

  // CJS build — for Node.js and older bundlers
  {
    input: 'src/index.ts',
    output: {
      file: 'dist/cjs/index.cjs',
      format: 'cjs',           // CommonJS format
      exports: 'named',
      sourcemap: true,
    },
    plugins: [resolve(), commonjs(), typescript(), terser()],
    external: ['@angular/core', 'rxjs'],
  },

  // IIFE build — for direct <script> tag usage
  {
    input: 'src/index.ts',
    output: {
      file: 'dist/iife/index.js',
      format: 'iife',          // Self-executing function
      name: 'MyLibrary',       // global variable name: window.MyLibrary
      globals: {
        '@angular/core': 'ng.core',
      },
    },
    plugins: [resolve(), commonjs(), typescript(), terser()],
  },

  // UMD build — works everywhere (AMD, CJS, browser global)
  {
    input: 'src/index.ts',
    output: {
      file: 'dist/umd/index.js',
      format: 'umd',
      name: 'MyLibrary',
      globals: { '@angular/core': 'ng.core' },
    },
    plugins: [resolve(), commonjs(), typescript(), terser()],
  },

  // TypeScript declarations build
  {
    input: 'src/index.ts',
    output: { file: 'dist/index.d.ts', format: 'esm' },
    plugins: [dts()],           // generates .d.ts declaration files
    external: ['@angular/core', 'rxjs'],
  },
];
```

The `external` array is critical — it tells Rollup not to bundle `@angular/core` and `rxjs` into the library output. These are peer dependencies that the consuming application already has.

---

### ng-packagr — Angular Libraries + Rollup

`ng-packagr` is the tool used by Angular CLI to build Angular libraries. Under the hood, it orchestrates TypeScript compilation, the Angular compiler, and (historically) Rollup to produce output conforming to the **Angular Package Format (APF)**.

```bash
# Create an Angular library in a workspace
ng generate library my-ui-components

# Build the library
ng build my-ui-components

# ng-packagr produces output conforming to Angular Package Format:
# dist/my-ui-components/
# ├── esm2022/
# │   └── my-ui-components.mjs    ← ESM for modern bundlers
# ├── fesm2022/
# │   └── my-ui-components.mjs    ← Flattened ESM (better tree shaking)
# ├── my-ui-components.d.ts       ← TypeScript declarations
# └── package.json                ← with 'exports' field pointing to all formats
```

> **Note:** Since Angular 18, `ng-packagr` has been migrating from Rollup to esbuild for library builds, significantly reducing build times. The output format (APF) remains the same — only the underlying tool changes.

---

## 9.7 — Babel

### What Is Babel?

Babel is a JavaScript compiler (transpiler) that converts modern JavaScript syntax into older, widely compatible syntax. It was the dominant transpilation tool from 2015 to ~2022. While esbuild and SWC have superseded it for performance-critical use cases, Babel remains important because:

1. It has the richest ecosystem of plugins for experimental JavaScript features
2. Many existing projects and configurations still use it
3. It supports transformations that esbuild and SWC don't (custom plugins)

Babel is a **pure transpiler** — it understands JavaScript/TypeScript syntax but performs no type checking. Run TypeScript's `tsc` separately for type checking.

---

### Presets — Bundled Plugin Collections

Babel's transformation power comes from plugins, but configuring individual plugins is tedious. **Presets** are curated collections of plugins for common scenarios:

```bash
npm install --save-dev @babel/core @babel/preset-env @babel/preset-typescript
```

```javascript
// babel.config.js — full configuration
module.exports = {
  presets: [
    // @babel/preset-env: transpiles modern JS to your target browsers
    // Most important and commonly used preset
    [
      '@babel/preset-env',
      {
        // Which environments to target (reads from .browserslistrc by default)
        targets: {
          chrome: '115',
          firefox: '115',
          safari: '17',
          edge: '115',
        },

        // useBuiltIns: how to inject polyfills for missing browser features
        // 'usage': automatically inject only polyfills your code actually uses
        // 'entry': inject polyfills for all targets (add import 'core-js' in entry)
        // false: no polyfills (you handle them manually)
        useBuiltIns: 'usage',
        corejs: '3.38',        // use core-js 3 for polyfills

        // Keeps ES Modules syntax — lets bundlers tree-shake
        // (Babel transforms syntax but NOT module format when this is set)
        modules: false,

        // Debug: print which transforms and polyfills are applied
        debug: false,
      },
    ],

    // @babel/preset-typescript: strips TypeScript type annotations
    // Does NOT type-check — just removes types so JS can be produced
    [
      '@babel/preset-typescript',
      {
        allowDeclareFields: true,       // support TypeScript declare modifier
        optimizeConstEnums: true,       // inline const enum values
      },
    ],

    // @babel/preset-react (if using React alongside Angular in monorepo)
    // '@babel/preset-react',
  ],

  plugins: [
    // TC39 Stage 3 decorators (for Angular's @Component, @Injectable etc.)
    ['@babel/plugin-proposal-decorators', { version: '2023-05' }],

    // Class fields (public/private class properties)
    '@babel/plugin-transform-class-properties',

    // Transform async generators
    '@babel/plugin-transform-async-generator-functions',
  ],

  // Environment-specific configuration
  env: {
    test: {
      // For Jest: use CommonJS modules (Jest can't handle native ESM well)
      presets: [
        ['@babel/preset-env', { targets: { node: 'current' }, modules: 'commonjs' }],
        '@babel/preset-typescript',
      ],
    },
  },
};
```

---

### browserslist Configuration

`browserslist` is the shared configuration that tells Babel, Autoprefixer, ESLint, and other tools which browsers to target. It can be in `package.json` or a separate `.browserslistrc` file:

```
# .browserslistrc — Angular default
last 2 Chrome versions
last 1 Firefox version
last 2 Edge versions
last 2 Safari versions
last 2 iOS versions
not dead
not IE 11              # explicitly exclude IE 11
```

```json
// Or inline in package.json
{
  "browserslist": {
    "production": [
      ">0.2%",
      "not dead",
      "not op_mini all"
    ],
    "development": [
      "last 1 chrome version",
      "last 1 firefox version",
      "last 1 safari version"
    ]
  }
}
```

```bash
# See which browsers match your query
npx browserslist "last 2 Chrome versions, last 1 Firefox version"

# See the full list of browsers your current .browserslistrc targets
npx browserslist
```

---

### Babel vs TypeScript Compiler — Understanding Both Roles

A common source of confusion: both Babel (`@babel/preset-typescript`) and TypeScript (`tsc`) can take TypeScript as input and output JavaScript. They serve **different purposes**:

| Feature | TypeScript Compiler (`tsc`) | Babel (`@babel/preset-typescript`) |
|---|---|---|
| **Type checking** | ✅ Full type system | ❌ None — just strips types |
| **Transpilation speed** | Medium | Fast |
| **Custom transforms** | Limited | Extensive plugin ecosystem |
| **Declaration files (.d.ts)** | ✅ Generated | ❌ Not generated |
| **Decorator support** | ✅ (via tsconfig flags) | ✅ (via plugin) |
| **ES Module output** | ✅ | ✅ |

**In Angular projects:** Angular CLI runs the TypeScript compiler for type checking and AOT template compilation, then esbuild (not Babel) for the actual JavaScript transformation. Babel is not in the Angular CLI build pipeline at all.

**When would you use Babel in an Angular context?** Jest configuration — Jest historically required CommonJS modules and used Babel (via `babel-jest`) to transform TypeScript for testing. This is changing as Jest gains better ESM support and `jest-preset-angular` moves toward esbuild-based transforms, but many existing Angular Jest setups still use Babel.

---

### Babel's Future Role

As esbuild and SWC have become the dominant transpilers for build performance, Babel's role has narrowed. Its future lies in:

1. **Experimental plugin ecosystem** — Babel can implement Stage 1/2 TC39 proposals before they're supported by esbuild/SWC
2. **Custom transforms** — code mods, instrumentation, framework-specific transforms
3. **Jest testing** — still widely used for transforming test files
4. **Legacy projects** — millions of projects rely on Babel and won't be migrated soon

> **Best Practice for Angular:** Don't add Babel to a new Angular CLI project. The CLI uses TypeScript + esbuild, which is faster and simpler. Only add Babel if you have a specific plugin need (rare) or if your Jest setup requires it and you haven't migrated to Vitest/Jest-with-esbuild.

---

## 9.8 — SWC (Speedy Web Compiler)

### What Is SWC?

SWC (Speedy Web Compiler) is a Rust-based compiler for JavaScript and TypeScript. Created by Dongyoon Won and now maintained by Vercel, it is designed as a **drop-in replacement for Babel** — it reads Babel configuration and produces identical output, but runs 20–70× faster.

```
Benchmark: Transform 1000 TypeScript files

Babel:   23.0 seconds
SWC:      0.9 seconds   (~25× faster)
esbuild:  0.5 seconds   (only for reference — different scope)
```

---

### SWC vs esbuild — Understanding the Difference

Both SWC and esbuild are fast Rust/Go tools, but they have different scopes:

| Feature | SWC | esbuild |
|---|---|---|
| **Primary role** | TypeScript/JS compiler | Bundler + compiler |
| **Babel compatibility** | ✅ Drop-in replacement | ❌ Different API |
| **Module bundling** | ✅ (via `@swc/cli` with bundler) | ✅ Primary feature |
| **Custom Babel plugins** | Limited | None |
| **TypeScript type checking** | ❌ Strips types only | ❌ Strips types only |
| **Speed** | ~25× faster than Babel | ~100× faster than Webpack |
| **Used by** | Next.js, Parcel, Nx (Jest) | Angular CLI, Vite (pre-bundling) |

**Key distinction:** SWC is primarily a *compiler* (transforms individual files, like Babel). esbuild is primarily a *bundler* (resolves module graph, produces bundled output). Both can do some of what the other does, but each is optimised for its primary role.

---

### @swc/core — Programmatic API

```bash
npm install --save-dev @swc/core @swc/cli
```

```javascript
// swc.config.mjs — SWC configuration
// (SWC can also read .swcrc files)

// Programmatic API
import { transform, transformFile } from '@swc/core';

// Transform a string
const output = await transform(`
  const greet = (name: string): string => \`Hello, \${name}!\`;
  export default greet;
`, {
  filename: 'greet.ts',  // filename affects which transforms apply

  jsc: {
    parser: {
      syntax: 'typescript',    // 'typescript' or 'ecmascript'
      tsx: false,
      decorators: true,         // enable decorator syntax
    },
    transform: {
      decoratorMetadata: true,  // emit decorator metadata (for DI)
    },
    target: 'es2022',           // output target
    minify: {
      compress: true,
      mangle: true,
    },
    externalHelpers: true,      // use @swc/helpers instead of inlining
  },

  module: {
    type: 'es6',                // 'es6' | 'commonjs' | 'amd' | 'umd'
  },

  sourceMaps: true,
  inlineSourcesContent: true,
});

console.log(output.code);
console.log(output.map);

// Transform a file directly
const fileOutput = await transformFile('./src/app/app.component.ts', {
  // same options as above
});
```

**`.swcrc` file format:**

```json
// .swcrc
{
  "$schema": "https://json.schemastore.org/swcrc",
  "jsc": {
    "parser": {
      "syntax": "typescript",
      "decorators": true,
      "dynamicImport": true
    },
    "target": "es2022",
    "loose": false,
    "externalHelpers": true
  },
  "module": {
    "type": "es6"
  },
  "sourceMaps": true
}
```

---

### SWC in the Angular Ecosystem

SWC is not used in the Angular CLI build pipeline (which uses esbuild). However, it is used in the Angular ecosystem in two places:

**Nx + Jest with SWC:** When using Nx for Angular monorepos, Nx can configure Jest to use `@swc/jest` (the SWC Jest transformer) instead of `ts-jest` or `babel-jest`. This dramatically reduces test run startup time:

```bash
# Nx with SWC for Jest
nx generate @nx/angular:application my-app --unitTestRunner=jest
# Nx configures @swc/jest automatically

# jest.config.ts in an Nx project with SWC
```

```javascript
// jest.config.ts — Nx + SWC configuration
import type { Config } from 'jest';

const config: Config = {
  preset: '@nx/jest/presets/js-with-ts-swc', // uses SWC for transforms
  testEnvironment: 'jsdom',
  transform: {
    '^.+\\.(ts|mts|js|html)$': [
      '@swc/jest',
      {
        jsc: {
          parser: { syntax: 'typescript', decorators: true },
          transform: { decoratorMetadata: true },
        },
      },
    ],
  },
};

export default config;
```

**Parcel bundler:** Parcel uses SWC as its JavaScript/TypeScript transformer. If you encounter a Parcel-based project, SWC is running under the hood.

---

## 9.9 — Linting & Formatting (2026 Stack)

### Why Linting and Formatting Matter

Linting and formatting are not just about aesthetics — they are quality control systems that catch bugs before runtime, enforce architectural boundaries, and eliminate entire categories of code review discussion. A file that violates a linting rule might have a real bug (using `==` instead of `===`, calling an async function without `await`, referencing an undefined variable). Consistent formatting means code review discussions focus on logic, not whitespace.

---

### ESLint v9 — The Industry Standard Linter

ESLint is the most widely used JavaScript and TypeScript linter. Version 9 (released April 2024) introduced a breaking change: the configuration format changed from `.eslintrc.*` files to a **flat config** file named `eslint.config.mjs`.

**The old format (v8 and below, still common):**

```json
// .eslintrc.json — OLD format (deprecated)
{
  "root": true,
  "ignorePatterns": ["projects/**/*"],
  "overrides": [
    {
      "files": ["*.ts"],
      "parser": "@typescript-eslint/parser",
      "parserOptions": {
        "project": ["tsconfig.json"],
        "createDefaultProgram": true
      },
      "extends": [
        "eslint:recommended",
        "plugin:@typescript-eslint/recommended",
        "plugin:@angular-eslint/recommended"
      ],
      "rules": {
        "@angular-eslint/directive-selector": ["error", { "type": "attribute", "prefix": "app", "style": "camelCase" }],
        "@angular-eslint/component-selector": ["error", { "type": "element", "prefix": "app", "style": "kebab-case" }]
      }
    }
  ]
}
```

**The new flat config format (v9+):**

```javascript
// eslint.config.mjs — NEW flat config format (ESLint v9+)
import eslint from '@eslint/js';
import tseslint from 'typescript-eslint';
import angular from 'angular-eslint';

export default tseslint.config(
  {
    // Files to lint
    files: ['**/*.ts'],

    // Extend from multiple base configs (no more 'extends' array — it's all flat)
    extends: [
      eslint.configs.recommended,
      ...tseslint.configs.recommended,
      ...tseslint.configs.stylistic,
      ...angular.configs.tsRecommended,
    ],

    // TypeScript project reference (for type-aware rules)
    languageOptions: {
      parserOptions: {
        projectService: true,   // auto-detect tsconfig.json (new in v9)
      },
    },

    // Override and configure specific rules
    rules: {
      // Angular-specific rules
      '@angular-eslint/directive-selector': ['error', {
        type: 'attribute',
        prefix: 'app',
        style: 'camelCase',
      }],
      '@angular-eslint/component-selector': ['error', {
        type: 'element',
        prefix: 'app',
        style: 'kebab-case',
      }],
      '@angular-eslint/prefer-standalone': 'error',   // enforce standalone components
      '@angular-eslint/no-empty-lifecycle-method': 'warn',

      // TypeScript rules
      '@typescript-eslint/no-explicit-any': 'warn',
      '@typescript-eslint/explicit-function-return-type': 'off',
      '@typescript-eslint/no-unused-vars': ['error', { argsIgnorePattern: '^_' }],
      '@typescript-eslint/prefer-nullish-coalescing': 'error',
      '@typescript-eslint/prefer-optional-chain': 'error',

      // General JavaScript rules
      'no-console': ['warn', { allow: ['warn', 'error'] }],
      'prefer-const': 'error',
      'eqeqeq': ['error', 'always'],
    },

    // Files to completely ignore
    ignores: ['dist/', '.angular/', 'node_modules/', '*.config.js'],
  },

  // Separate config for HTML templates
  {
    files: ['**/*.html'],
    extends: [...angular.configs.templateRecommended, ...angular.configs.templateAccessibility],
    rules: {
      '@angular-eslint/template/prefer-control-flow': 'error',  // enforce @if/@for/@switch
      '@angular-eslint/template/no-negated-async': 'error',
    },
  }
);
```

---

### @angular-eslint — Angular-Specific Rules

`@angular-eslint` provides rules that understand Angular's specific patterns — component selectors, lifecycle hooks, template syntax, and more:

```bash
# Install @angular-eslint (Angular CLI does this automatically for new projects)
ng add @angular-eslint/schematics
```

**Important Angular-specific rules:**

```javascript
{
  rules: {
    // Component and directive naming
    '@angular-eslint/component-selector': ['error', { type: 'element', prefix: 'app', style: 'kebab-case' }],
    '@angular-eslint/directive-selector': ['error', { type: 'attribute', prefix: 'app', style: 'camelCase' }],

    // Enforce modern Angular patterns
    '@angular-eslint/prefer-standalone': 'error',      // standalone components only
    '@angular-eslint/prefer-output-emitter-ref': 'error', // use output() not @Output()
    '@angular-eslint/prefer-signals': 'warn',            // prefer signal inputs

    // Prevent common bugs
    '@angular-eslint/no-empty-lifecycle-method': 'error',
    '@angular-eslint/no-input-rename': 'error',          // don't alias @Input names
    '@angular-eslint/no-output-native': 'error',         // don't name outputs like DOM events
    '@angular-eslint/use-lifecycle-interface': 'error',  // implement interfaces explicitly

    // Template rules (in HTML config block)
    '@angular-eslint/template/no-any': 'error',
    '@angular-eslint/template/prefer-control-flow': 'error', // use @if not *ngIf
    '@angular-eslint/template/use-track-by-function': 'warn', // require track in @for
  }
}
```

---

### Prettier — Opinionated Formatting

Prettier is a code formatter that enforces a consistent code style by parsing and reprinting your code according to a fixed set of rules. It removes all formatting discussions from code review — Prettier makes the decisions, you don't argue.

```bash
npm install --save-dev prettier
```

```json
// .prettierrc — Prettier configuration (minimal by design)
{
  "semi": true,              // always use semicolons
  "singleQuote": true,       // single quotes for strings
  "quoteProps": "as-needed", // only quote object properties when necessary
  "trailingComma": "all",    // trailing commas in multi-line expressions
  "printWidth": 100,         // line length target
  "tabWidth": 2,             // 2 spaces for indentation
  "useTabs": false,
  "bracketSpacing": true,    // { foo: bar } not {foo: bar}
  "arrowParens": "always",   // (x) => x not x => x
  "endOfLine": "lf",         // Unix line endings
  "overrides": [
    {
      "files": "*.html",
      "options": { "printWidth": 120, "parser": "angular" }
    },
    {
      "files": "*.json",
      "options": { "printWidth": 80 }
    }
  ]
}
```

```bash
# Format all files
npx prettier --write "src/**/*.{ts,html,scss,json}"

# Check without modifying (for CI)
npx prettier --check "src/**/*.{ts,html,scss,json}"
```

**ESLint + Prettier together:** ESLint and Prettier can conflict — ESLint has formatting rules, Prettier has formatting rules. The standard solution is `eslint-config-prettier` which disables all ESLint rules that Prettier handles:

```bash
npm install --save-dev eslint-config-prettier
```

```javascript
// eslint.config.mjs — disable ESLint formatting rules
import prettier from 'eslint-config-prettier';

export default tseslint.config(
  // ... other configs ...
  prettier,  // MUST be last — disables ESLint formatting rules that conflict with Prettier
);
```

---

### Biome — The Unified Alternative

**Biome** is a Rust-based toolchain that combines a linter, formatter, and import organiser into a single tool. It is positioned as a replacement for ESLint + Prettier, with a single configuration file and dramatically faster execution.

```bash
npm install --save-dev @biomejs/biome
npx biome init  # creates biome.json
```

```json
// biome.json
{
  "$schema": "https://biomejs.dev/schemas/1.9.0/schema.json",
  "organizeImports": { "enabled": true },
  "linter": {
    "enabled": true,
    "rules": {
      "recommended": true,
      "complexity": {
        "noBannedTypes": "error",
        "noUselessTypeConstraint": "error"
      },
      "correctness": {
        "noUnusedVariables": "error",
        "useExhaustiveDependencies": "warn"
      },
      "style": {
        "noNonNullAssertion": "warn",
        "useConst": "error",
        "useTemplate": "error"
      }
    }
  },
  "formatter": {
    "enabled": true,
    "indentStyle": "space",
    "indentWidth": 2,
    "lineWidth": 100,
    "lineEnding": "lf"
  },
  "javascript": {
    "formatter": {
      "quoteStyle": "single",
      "trailingCommas": "all",
      "semicolons": "always"
    }
  },
  "files": {
    "ignore": ["dist/", ".angular/", "node_modules/"]
  }
}
```

```bash
# Check (lint + format check)
npx biome check src/

# Fix automatically
npx biome check --write src/

# Format only
npx biome format --write src/

# Lint only
npx biome lint src/
```

**Biome vs ESLint + Prettier:**

| Feature | ESLint + Prettier | Biome |
|---|---|---|
| Setup complexity | High (2 tools, many plugins) | Low (1 tool, 1 config) |
| Speed | Medium | Very fast (Rust) |
| Angular template linting | ✅ (@angular-eslint) | ❌ Not yet supported |
| Plugin ecosystem | Massive | Growing (smaller) |
| Config migration | Existing ecosystem | `biome migrate eslint` command |
| TypeScript-aware rules | ✅ (type-aware rules) | Limited |

> **For Angular projects in 2026:** ESLint with `@angular-eslint` remains the recommendation because Biome does not yet support Angular template linting. Use Biome for non-Angular TypeScript projects or when you need maximum simplicity.

---

### Husky — Git Hooks Automation

Husky makes it easy to run scripts as Git hooks, ensuring code quality gates run before commits and pushes:

```bash
npm install --save-dev husky
npx husky init
```

```bash
# .husky/pre-commit — runs before every git commit
#!/bin/sh
npx lint-staged   # lint and format only staged files
```

```bash
# .husky/commit-msg — validate commit message format
#!/bin/sh
npx --no -- commitlint --edit "$1"
```

```bash
# .husky/pre-push — run tests before pushing
#!/bin/sh
npm run test:ci && npm run build
```

---

### lint-staged — Lint Only Changed Files

Running ESLint on the entire codebase before every commit would be too slow. `lint-staged` runs linters only on the files that are currently staged in Git — much faster:

```json
// package.json
{
  "lint-staged": {
    "*.{ts,tsx}": [
      "eslint --fix --max-warnings=0",
      "prettier --write"
    ],
    "*.{html}": [
      "eslint --fix --max-warnings=0",
      "prettier --write --parser=angular"
    ],
    "*.{scss,css}": [
      "stylelint --fix",
      "prettier --write"
    ],
    "*.{json,md}": [
      "prettier --write"
    ]
  }
}
```

---

### Stylelint — CSS/SCSS Linting (v16+)

Stylelint catches errors and enforces conventions in your CSS and SCSS files:

```bash
npm install --save-dev stylelint stylelint-config-standard-scss
```

```json
// .stylelintrc.json
{
  "extends": [
    "stylelint-config-standard-scss"
  ],
  "rules": {
    "color-no-invalid-hex": true,
    "declaration-no-important": true,
    "no-duplicate-selectors": true,
    "scss/no-duplicate-dollar-variables": true,
    "selector-class-pattern": "^[a-z][a-z0-9-]*$",   // kebab-case classes
    "custom-property-pattern": "^[a-z][a-z0-9-]*$",  // kebab-case CSS vars
    "order/properties-alphabetical-order": null        // disable if not wanted
  },
  "ignoreFiles": ["dist/**", "node_modules/**"]
}
```

---

### EditorConfig — Baseline Editor Consistency

`.editorconfig` is a file format that configures fundamental editor settings consistently across all editors and IDEs:

```ini
# .editorconfig
root = true

[*]
charset = utf-8
end_of_line = lf
indent_style = space
indent_size = 2
insert_final_newline = true
trim_trailing_whitespace = true
max_line_length = 100

[*.md]
trim_trailing_whitespace = false
max_line_length = off

[*.json]
indent_size = 2

[Makefile]
indent_style = tab    # Makefiles require tabs
```

VS Code, IntelliJ, Vim, Emacs, and most other editors read `.editorconfig` automatically (sometimes requiring a plugin). This ensures consistent newlines (LF vs CRLF is a common Git conflict cause), consistent indentation, and trailing whitespace trimming across the entire team.

---

## 9.10 — Task Automation & Monorepo Tooling

### npm Scripts as Task Runner

As covered in Phase 8, `npm scripts` in `package.json` are the simplest form of task automation. For a single Angular project, they are often sufficient:

```json
{
  "scripts": {
    "start":          "ng serve",
    "build":          "ng build --configuration production",
    "test":           "ng test --watch=false",
    "lint":           "ng lint",
    "lint:fix":       "ng lint --fix",
    "format":         "prettier --write \"src/**/*.{ts,html,scss,json}\"",
    "format:check":   "prettier --check \"src/**/*.{ts,html,scss,json}\"",
    "analyze":        "ng build --stats-json && npx webpack-bundle-analyzer dist/browser/stats.json",
    "validate":       "npm run lint && npm run test && npm run build",
    "prepare":        "husky"
  }
}
```

The limitations appear in monorepos: there's no awareness of which projects are affected by a change, no caching of results, and no parallelisation across projects.

---

### Nx 20+ — The Gold Standard for Angular Monorepos

**Nx** is a smart build system and monorepo tool with first-class Angular support. It was originally created by ex-Google engineers (Jeff Cross and Victor Savkin) who built Angular. Nx understands Angular's project structure, generates code following Angular best practices, and provides capabilities that Angular CLI alone cannot match.

```bash
# Create a new Nx workspace with Angular
npx create-nx-workspace@latest my-org --preset=angular-monorepo

# Or add Nx to an existing Angular CLI workspace
npx nx@latest init
```

**The Nx project graph — understanding affected builds:**

Nx analyses import relationships between all projects and builds a **Project Graph** — a directed acyclic graph showing which projects depend on which other projects. When you change a file in `libs/shared-ui`, Nx knows which `apps/` and `libs/` depend on it and only rebuilds/retests those.

```bash
# Visualise the project graph in the browser
npx nx graph

# Only run lint for projects AFFECTED by changes since main branch
npx nx affected --target=lint --base=main

# Only run tests for affected projects (in parallel)
npx nx affected --target=test --base=main --parallel=3

# Build only affected apps
npx nx affected --target=build --base=main --configuration=production
```

**Nx workspace structure for Angular:**

```
my-org/
├── apps/
│   ├── web/                    ← main Angular app
│   │   ├── src/
│   │   ├── project.json        ← Nx project config (targets, tags)
│   │   └── tsconfig.app.json
│   └── admin/                  ← admin Angular app
├── libs/
│   ├── feature-auth/           ← feature library (smart, connects to state)
│   │   ├── src/
│   │   └── project.json
│   ├── ui-components/          ← UI library (dumb, presentational only)
│   │   ├── src/
│   │   └── project.json
│   ├── data-access-users/      ← data access library (services, API calls)
│   │   ├── src/
│   │   └── project.json
│   └── util-formatting/        ← utility library (pure functions)
│       ├── src/
│       └── project.json
├── nx.json                     ← Nx configuration
└── package.json
```

**Nx generators — scaffolding code:**

```bash
# Generate a new Angular application
npx nx generate @nx/angular:application admin --directory=apps/admin

# Generate an Angular library
npx nx generate @nx/angular:library ui-components --directory=libs/ui-components \
  --publishable --importPath=@myorg/ui-components

# Generate a component inside a specific project
npx nx generate @nx/angular:component button --project=ui-components

# Generate a service
npx nx generate @nx/angular:service auth --project=data-access-users
```

**Module boundary enforcement with ESLint:**

One of Nx's most valuable features is the ability to enforce architectural boundaries between library layers using ESLint rules:

```json
// project.json — tagging projects by layer and domain
{
  "name": "feature-auth",
  "tags": ["type:feature", "domain:auth"]
}
```

```javascript
// eslint.config.mjs — enforce module boundaries
{
  rules: {
    '@nx/enforce-module-boundaries': ['error', {
      enforceBuildableLibDependency: true,
      allow: [],
      depConstraints: [
        // feature libraries can only import ui, data-access, util
        { sourceTag: 'type:feature', onlyDependOnLibsWithTags: ['type:ui', 'type:data-access', 'type:util'] },
        // ui libraries can only import other ui or util
        { sourceTag: 'type:ui', onlyDependOnLibsWithTags: ['type:ui', 'type:util'] },
        // data-access can only import util
        { sourceTag: 'type:data-access', onlyDependOnLibsWithTags: ['type:util'] },
        // util cannot import anything from our workspace
        { sourceTag: 'type:util', onlyDependOnLibsWithTags: ['type:util'] },
        // auth domain cannot import from other domains
        { sourceTag: 'domain:auth', onlyDependOnLibsWithTags: ['domain:auth', 'domain:shared'] },
      ],
    }],
  }
}
```

**Nx build cache — local and remote:**

Nx caches the results of every task. If the inputs (source files, dependencies, configuration) haven't changed, Nx replays the cached output instead of re-running the task:

```bash
# First run: actually builds
npx nx build web    # 15 seconds

# Second run (nothing changed): replays cache
npx nx build web    # 0.1 seconds — "cache hit"

# nx.json — cache configuration
```

```json
// nx.json
{
  "tasksRunnerOptions": {
    "default": {
      "runner": "nx/tasks-runners/default",
      "options": {
        "cacheableOperations": ["build", "test", "lint", "e2e"]
      }
    }
  },
  "nxCloudId": "your-cloud-id"  // enables Nx Cloud (remote distributed cache)
}
```

**Nx Cloud** extends caching to a remote cache shared by all developers and CI machines. When a developer builds on their laptop, the result is uploaded to Nx Cloud. When the same task runs in CI (with the same inputs), it downloads the cached result instead of rebuilding — CI runs that previously took 20 minutes can complete in 2 minutes.

**Nx Agents — distributed task execution:**

Nx Cloud also enables distributing tasks across multiple CI machines:

```yaml
# .github/workflows/ci.yml — Nx with distributed agents
- name: Start Nx Agents
  run: npx nx-cloud start-ci-run --distribute-on="3 linux-medium-js" --stop-agents-after="build"

- name: Run affected targets (distributed across 3 machines)
  run: |
    npx nx affected --target=lint &
    npx nx affected --target=test &
    npx nx affected --target=build &
    wait
```

---

### Turborepo — The Alternative Monorepo Runner

Turborepo is a monorepo task runner created by Jared Palmer (now maintained by Vercel). It is framework-agnostic and focuses on task orchestration and caching without opinionated code generation.

```json
// turbo.json — Turborepo configuration
{
  "$schema": "https://turbo.build/schema.json",
  "tasks": {
    "build": {
      "dependsOn": ["^build"],  // build dependencies first
      "outputs": ["dist/**"],   // cache these outputs
      "env": ["NODE_ENV", "API_URL"]  // include env vars in cache key
    },
    "test": {
      "dependsOn": ["^build"],
      "outputs": ["coverage/**"]
    },
    "lint": {
      "outputs": []
    },
    "dev": {
      "cache": false,    // never cache dev server
      "persistent": true // long-running task
    }
  },
  "remoteCache": {
    "enabled": true
  }
}
```

```bash
# Run build across all packages (in correct dependency order, parallel where safe)
npx turbo build

# Run only for packages affected by changes
npx turbo build --filter=...[HEAD^1]

# Run multiple tasks
npx turbo lint test build
```

**Turborepo vs Nx:**

| Feature | Turborepo | Nx |
|---|---|---|
| Framework-agnostic | ✅ | ✅ |
| Angular code generation | ❌ | ✅ (excellent) |
| Project graph analysis | Basic | Advanced (import-based) |
| Remote caching | ✅ (Vercel) | ✅ (Nx Cloud) |
| Module boundary enforcement | ❌ | ✅ |
| Plugin ecosystem | Small | Large (Angular, React, Node) |
| Learning curve | Lower | Higher |

> **Recommendation:** Use **Nx** for Angular monorepos — its Angular-specific generators, module boundary enforcement, and deep Angular CLI integration make it the clear choice. Use **Turborepo** for polyglot monorepos (multiple languages/frameworks) where you need a simpler task runner without framework-specific opinions.

---

### Angular CLI as Build Orchestrator

Even without Nx or Turborepo, Angular CLI is a capable build orchestrator for single-app workspaces:

```bash
# Build the application
ng build --configuration production

# Serve with live reload
ng serve --host 0.0.0.0 --port 4200

# Run unit tests
ng test --watch=false --code-coverage

# Run e2e tests
ng e2e

# Generate code
ng generate component auth/login --standalone

# Run lint
ng lint

# Update Angular and schematics
ng update @angular/core @angular/cli
```

Angular CLI also provides **build caching** out of the box:

```bash
# Enable persistent build cache (stores cache in .angular/cache/)
ng cache enable

# Clean the cache
ng cache clean

# Show cache status
ng cache info
```

The Angular CLI cache can reduce cold build times by 40–60% on subsequent builds when files haven't changed.

---

### Bun — Fast JS Runtime and Package Manager

**Bun** is an all-in-one JavaScript runtime, package manager, bundler, and test runner written in Zig. Released in 2022, it targets developer tooling workflows where Node.js and npm feel slow.

```bash
# Install Bun
curl -fsSL https://bun.sh/install | bash

# Use as a package manager (npm-compatible)
bun install                     # 10-30× faster than npm install
bun add lodash                  # add dependency
bun remove lodash               # remove dependency
bun run build                   # run package.json scripts

# Use as a test runner
bun test                        # Jest-compatible API, very fast

# Use as a bundler
bun build src/main.ts --outdir dist --minify

# Use as a runtime (instead of node)
bun run src/server.ts
```

**Bun and Angular:** Bun cannot replace the Angular CLI build process (which uses esbuild and the Angular compiler). However, Bun can be used for:
- Running Angular CLI commands faster: `bun run ng serve` uses Bun as the runtime for the CLI itself
- Package installation: `bun install` instead of `npm install` (significant speed improvement)
- Non-Angular scripts: utility scripts, code generators, build helper scripts

```json
// package.json — use Bun for scripts
{
  "scripts": {
    "start": "bun run ng serve",
    "build": "bun run ng build --configuration production",
    "install:fast": "bun install"  // explicit Bun install option
  }
}
```

> **Caveat:** Bun's Node.js compatibility is excellent but not perfect. Test your Angular CLI workflow with Bun before adopting it in production CI pipelines. The Angular team officially supports npm, yarn, and pnpm.

---

### Make and Shell Scripts

For build automation beyond what npm scripts handle cleanly, `Makefile` and shell scripts remain valuable — especially in CI/CD environments where developers need clear, composable commands:

```makefile
# Makefile — cross-project automation commands
.PHONY: install build test lint deploy clean

install:
	npm ci

build: install
	npm run build:prod

test: install
	npm run test:ci

lint: install
	npm run lint
	npm run format:check

deploy: build
	aws s3 sync dist/browser/ s3://my-bucket/ --delete
	aws cloudfront create-invalidation --distribution-id ABC123 --paths "/*"

clean:
	rm -rf dist .angular/cache node_modules

# Run full CI pipeline locally
ci: install lint test build
	@echo "✅ All CI checks passed"
```

```bash
# Simple validation script used in git hooks or CI
#!/bin/bash
set -e  # exit immediately on any error

echo "🔍 Running lint..."
npm run lint

echo "🧪 Running tests..."
npm run test:ci

echo "🏗️ Building..."
npm run build:prod

echo "✅ All checks passed!"
```

---

### GitHub Actions for CI Builds

A complete Angular CI pipeline in GitHub Actions brings together everything from this phase:

```yaml
# .github/workflows/ci.yml
name: Angular CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  NODE_VERSION: '22'

jobs:
  ci:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0  # needed for Nx affected commands

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'     # caches node_modules between runs

      - name: Install dependencies
        run: npm ci         # uses package-lock.json, never updates it

      - name: Lint
        run: npm run lint

      - name: Format check
        run: npm run format:check

      - name: Security audit
        run: npm audit --audit-level=high

      - name: Unit tests
        run: npm run test:ci

      - name: Build (production)
        run: npm run build:prod
        env:
          API_URL: ${{ secrets.PRODUCTION_API_URL }}

      - name: Upload build artifact
        uses: actions/upload-artifact@v4
        if: github.ref == 'refs/heads/main'
        with:
          name: dist
          path: dist/browser/

  deploy:
    needs: ci
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
      - name: Download artifact
        uses: actions/download-artifact@v4
        with:
          name: dist
          path: dist/

      - name: Deploy to S3
        run: aws s3 sync dist/ s3://my-bucket/ --delete
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
```

---

## Summary & Key Takeaways

Build tools form the invisible infrastructure that transforms your source code into the optimised, secure, and performant applications users experience. Every concept in this phase has direct implications for Angular development — from the esbuild-powered Angular CLI builder to ESLint template rules to Nx's affected build detection.

### The Modern Angular Build Stack (2026)

```
Source Code (TypeScript + SCSS + HTML templates)
        ↓
Angular Compiler (AOT template compilation + type checking)
        ↓
esbuild (transpilation + bundling + minification)
        ↓
Output: Optimised JavaScript bundles (ES2022+, content-hashed)
        ↓
Deployed to CDN (brotli-compressed, cached by hash)
```

### Tool Selection Guide

| Need | Tool |
|---|---|
| Angular app build | Angular CLI application builder (esbuild) |
| Angular library build | ng-packagr (esbuild under the hood) |
| Development server | Angular CLI dev server (Vite-powered + esbuild) |
| TypeScript compilation + type checking | TypeScript compiler (tsc) — run separately |
| Code linting | ESLint v9 + @angular-eslint |
| Code formatting | Prettier (+ eslint-config-prettier) |
| CSS/SCSS linting | Stylelint |
| Git hooks | Husky + lint-staged |
| Monorepo (Angular) | Nx 20+ |
| Monorepo (polyglot) | Turborepo |
| CI/CD | GitHub Actions (or GitLab CI, Azure Pipelines) |
| Bundle analysis | source-map-explorer, webpack-bundle-analyzer |

### Build Tool Evolution Timeline

```
2015 — Webpack dominates (Grunt/Gulp die out)
2016 — Rollup introduces tree shaking (ES Module-first)
2018 — Parcel brings zero-config bundling
2020 — esbuild (10-100× speed) + Vite (ESM dev server) change everything
2021 — SWC (Rust Babel replacement) adopted by Next.js
2022 — Bun (all-in-one runtime) released
2023 — Angular migrates from Webpack to esbuild (v17)
2024 — Rolldown emerges (Rust Rollup replacement)
2026 — Rolldown integrates into Vite; esbuild the standard for Angular
```

> **The meta-lesson:** Build tools are infrastructure, not magic. Every feature of every build tool exists to solve a specific problem. When you encounter a build error or a performance issue, knowing what each tool does — and why — gives you the mental model to diagnose and fix it rather than guessing with Stack Overflow.

---

*Phase 9 Complete — Next: Phase 10 — Angular Core (Fundamentals)*

*Document version: March 2026 | Angular 21 · esbuild · Vite 6 · ESLint v9 · Nx 20*

# Phase 10 — Angular Core Fundamentals
## Part 1: Overview, Philosophy, CLI & Workspace Structure
> Topics: 10.1 · 10.2 · 10.3

---

## 10.1 Angular Overview & Philosophy

### What is Angular?

Angular is a **platform and framework** for building single-page client applications using HTML and TypeScript. Developed and maintained by Google, Angular provides a complete, opinionated solution out of the box — everything from routing, forms, HTTP client, animations, testing utilities, and internationalization is part of the official framework.

This is fundamentally different from React or Vue, which are *libraries* that solve specific UI problems and rely on the community for the rest. Angular gives you the *entire airplane*, not just the wings.

### Angular vs AngularJS — A Complete Rewrite

AngularJS (Angular 1.x, released 2010) and Angular (v2+, released 2016) share a name but are entirely different frameworks. AngularJS used JavaScript with a two-way data binding model and a controller/scope architecture. Angular 2+ was a **ground-up rewrite** in TypeScript with a completely different component-based model, a new dependency injection system, and a modern build pipeline. They are not compatible, and migration requires rewriting code.

When people say "Angular" today, they always mean Angular v2+.

### Angular's Core Design Philosophy

**1. Opinionated structure.** Angular enforces a consistent project structure across teams. There is a "right way" to do routing, forms, HTTP, state communication, and testing. This reduces decision fatigue and makes large codebases from different teams feel familiar.

**2. TypeScript-first.** Angular was designed from the start to use TypeScript. All APIs are typed, templates are type-checked in strict mode, and the DI system leverages TypeScript's type system heavily.

**3. Component-based architecture.** The entire UI is a tree of components. Each component encapsulates its template, styles, and logic — making the application modular, testable, and reusable.

**4. Dependency Injection (DI).** Angular has a powerful built-in DI system. Services are injected into components and other services, promoting loose coupling and testability.

**5. Reactive by design.** Angular deeply integrates RxJS for async operations, and from Angular 16 onwards, **Signals** provide fine-grained reactive state management at the component level.

**6. Scalability.** Angular's architecture is designed for *large teams and large codebases*. Its enforced patterns, strong typing, and CLI tooling mean it scales better than loosely structured approaches when hundreds of developers work on the same application.

### Angular Versioning & Release Cadence

Angular follows **semantic versioning** (semver) with a predictable release schedule:
- A **major version** is released every 6 months (typically March and September/October).
- Each major version has **18 months of active support** followed by **12 months of long-term support (LTS)**, for a total of 30 months.
- Patch and minor releases happen frequently within a major cycle.

This means Angular 21 (as of March 2026) is the current major version. The Angular team publishes detailed **migration guides and automated migrations** via `ng update`, so upgrades rarely require manual intervention.

**Important:** Angular's commitment to backward compatibility within a major version means that upgrading between major versions is almost always possible using the CLI's schematic migration tools.

### The Modern Angular Era (Signals + Zoneless)

Angular has undergone a major architectural evolution starting around Angular 16:
- **Angular 16–17:** Signals introduced as developer preview; new `@if`, `@for`, `@defer` control flow blocks
- **Angular 17–18:** Standalone components become the default; esbuild replaces webpack
- **Angular 19–20:** Signals stable; Zoneless change detection stable; Resource API for async signals
- **Angular 21:** Partial hydration, `httpResource`, Signal Router preview, Karma deprecated

The direction is clear: **Signals replace observables for local state, standalone components replace NgModules, and Zone.js is being eliminated** for better performance. New projects should embrace all three.

---

## 10.2 Angular CLI (Angular 21)

The Angular CLI is the **command-line interface** that scaffolds, builds, serves, tests, and manages Angular applications. It is one of Angular's greatest strengths — it abstracts away all build configuration so developers can focus on code.

### Installing the Angular CLI

```bash
# Install globally
npm install -g @angular/cli

# Verify installation
ng version
```

### Creating a New Project

```bash
# Basic project creation (standalone by default in Angular 17+)
ng new my-angular-app

# With specific options
ng new my-angular-app \
  --style=scss \          # Use SCSS for styles
  --routing=true \        # Generate app-routing module
  --ssr=false \           # No server-side rendering
  --standalone=true       # Standalone components (default in v17+)
```

Since Angular 17, new projects are **standalone by default** — no `AppModule` is generated. This is the modern, recommended approach.

### Development Server

```bash
# Start the dev server (uses esbuild + Vite dev server under the hood)
ng serve

# With specific port and open browser
ng serve --port 4200 --open

# With host binding (for network access)
ng serve --host 0.0.0.0
```

Under the hood, `ng serve` in Angular 17+ uses **esbuild** for compilation and a **Vite-based dev server** for fast HMR (Hot Module Replacement). Build times dropped dramatically compared to the old Webpack-based dev server.

### Building for Production

```bash
# Standard production build
ng build

# Production build with full optimization
ng build --configuration=production
```

The output goes to `dist/your-app-name/browser/` by default. Angular automatically:
- Minifies and uglifies JavaScript
- Performs tree-shaking (removes unused code)
- Generates hashed filenames for cache busting
- Inlines critical CSS
- Optimizes images and fonts

### Code Generation with `ng generate`

The `ng generate` (or `ng g`) command scaffolds Angular artifacts following the project's conventions:

```bash
# Generate a component
ng generate component features/user-profile
ng g c features/user-profile         # Shorthand

# Generate a service
ng generate service core/auth
ng g s core/auth

# Generate a directive
ng generate directive shared/directives/highlight
ng g d shared/directives/highlight

# Generate a pipe
ng generate pipe shared/pipes/format-date
ng g p shared/pipes/format-date

# Generate a guard
ng generate guard core/guards/auth
ng g guard core/guards/auth

# Generate an interceptor
ng generate interceptor core/interceptors/auth
ng g interceptor core/interceptors/auth

# Generate an interface
ng generate interface models/user
ng g interface models/user

# Generate an enum
ng generate enum models/user-role
ng g enum models/user-role

# Generate a class
ng generate class models/user
ng g class models/user
```

By default, `ng g c` will also generate a spec file for testing. Use `--skip-tests` to skip it.

**Important:** Generated files follow the project's configured style (inline templates vs separate files, SCSS vs CSS, etc.) as defined in `angular.json`.

### `ng add` — Schematics for Third-Party Integration

`ng add` installs a package *and* runs its Angular schematic to configure the integration automatically:

```bash
# Add Angular Material (installs + configures theme, animations, etc.)
ng add @angular/material

# Add PWA support
ng add @angular/pwa

# Add SSR
ng add @angular/ssr
```

This is far better than manually installing packages and configuring them, because schematics update `angular.json`, `app.config.ts`, and other configuration files correctly.

### `ng update` — Migration Tool

Angular's migration system is one of its most impressive features. When upgrading Angular, running `ng update` applies automated code migrations (called schematics) that update your code to use new APIs:

```bash
# Check what can be updated
ng update

# Update Angular core and CLI
ng update @angular/core @angular/cli

# Update a specific package
ng update @angular/material
```

Schematics can rename API calls, transform templates, add new configuration options, and refactor code patterns — saving hours of manual migration work.

### Other Useful CLI Commands

```bash
# Run unit tests (Jest or Vitest in modern setups)
ng test

# Run end-to-end tests
ng e2e

# Run linting
ng lint

# Analyze build output
ng build --stats-json
# Then open the stats file in webpack-bundle-analyzer or source-map-explorer

# Manage the build cache
ng cache enable
ng cache disable
ng cache clean
```

### The Application Builder (`@angular-devkit/build-angular:application`)

Since Angular 17, the default builder for new projects is the **application builder** backed by **esbuild**. This replaced the Webpack-based builder and offers:
- 3–5× faster production builds
- 10× faster watch mode rebuilds
- Native ESM output
- Vite-based dev server for sub-second HMR

The builder is configured in `angular.json` under `architect.build.builder`. You shouldn't need to change this manually — it's set correctly by the CLI.

---

## 10.3 Angular Workspace Structure

Understanding the project structure is essential before writing any code. Here is the structure generated by `ng new my-app`:

```
my-app/
├── angular.json                 ← Workspace configuration (the CLI's control panel)
├── package.json                 ← NPM dependencies and scripts
├── package-lock.json            ← Lockfile for deterministic installs
├── tsconfig.json                ← Base TypeScript configuration
├── tsconfig.app.json            ← TypeScript config for the app (extends base)
├── tsconfig.spec.json           ← TypeScript config for tests (extends base)
├── .editorconfig                ← Editor formatting rules (tab size, etc.)
├── .gitignore                   ← Files excluded from git
├── .angular/                    ← Angular's build cache (do not commit)
└── src/
    ├── main.ts                  ← Application entry point (bootstrap)
    ├── index.html               ← Root HTML file
    ├── styles.scss              ← Global styles
    └── app/
        ├── app.component.ts     ← Root component
        ├── app.component.html   ← Root component template
        ├── app.component.scss   ← Root component styles
        ├── app.component.spec.ts ← Root component tests
        └── app.config.ts        ← Application configuration (standalone bootstrap)
```

### `angular.json` — The Workspace Configuration

This is the most important configuration file in an Angular project. It controls build options, file locations, test runners, linting, and more:

```json
{
  "version": 1,
  "projects": {
    "my-app": {
      "architect": {
        "build": {
          "builder": "@angular-devkit/build-angular:application",
          "options": {
            "outputPath": "dist/my-app",
            "index": "src/index.html",
            "browser": "src/main.ts",
            "tsConfig": "tsconfig.app.json",
            "styles": ["src/styles.scss"],
            "scripts": [],
            "assets": [
              { "glob": "**/*", "input": "src/assets", "output": "assets" }
            ]
          },
          "configurations": {
            "production": {
              "optimization": true,
              "outputHashing": "all",     // Adds hash to filenames for cache busting
              "sourceMap": false,
              "budgets": [               // Warns/errors if bundle exceeds these sizes
                {
                  "type": "initial",
                  "maximumWarning": "500kB",
                  "maximumError": "1MB"
                }
              ]
            }
          }
        }
      }
    }
  }
}
```

### `src/main.ts` — The Bootstrap Entry Point

For standalone applications (Angular 17+), `main.ts` looks like this:

```typescript
import { bootstrapApplication } from '@angular/platform-browser';
import { appConfig } from './app/app.config';
import { AppComponent } from './app/app.component';

// bootstrapApplication is the modern entry point for standalone apps
// It replaces the old platformBrowserDynamic().bootstrapModule(AppModule)
bootstrapApplication(AppComponent, appConfig)
  .catch(err => console.error(err));
```

### `src/app/app.config.ts` — Application Configuration

This file replaces the old `AppModule` as the central place to configure application-wide providers:

```typescript
import { ApplicationConfig } from '@angular/core';
import { provideRouter } from '@angular/router';
import { provideHttpClient, withFetch } from '@angular/common/http';
import { provideAnimationsAsync } from '@angular/platform-browser/animations/async';
import { routes } from './app.routes';

export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(routes),                    // Configure routing
    provideHttpClient(withFetch()),            // HttpClient with native fetch
    provideAnimationsAsync(),                 // Async animations loading
  ]
};
```

### TypeScript Configuration Files

Three `tsconfig` files work together:

**`tsconfig.json`** — the base config shared by everything:
```json
{
  "compileOnSave": false,
  "compilerOptions": {
    "baseUrl": "./",
    "outDir": "./dist/out-tsc",
    "strict": true,                // Enable all strict checks — always use this
    "noImplicitOverride": true,
    "noPropertyAccessFromIndexSignature": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,
    "skipLibCheck": true,
    "esModuleInterop": true,
    "experimentalDecorators": true, // Required for Angular decorators
    "moduleResolution": "bundler",  // Angular 17+ recommended setting
    "importHelpers": true,
    "target": "ES2022",
    "module": "ES2022",
    "lib": ["ES2022", "dom"]
  }
}
```

**`tsconfig.app.json`** — extends the base, applies to application code:
```json
{
  "extends": "./tsconfig.json",
  "compilerOptions": {
    "outDir": "./out-tsc/app",
    "types": []
  },
  "files": ["src/main.ts"],
  "include": ["src/**/*.d.ts"]
}
```

**`tsconfig.spec.json`** — extends the base, applies to test files:
```json
{
  "extends": "./tsconfig.json",
  "compilerOptions": {
    "outDir": "./out-tsc/spec",
    "types": ["jasmine"]  // or "jest" / "vitest" depending on test runner
  },
  "include": [
    "src/**/*.spec.ts",
    "src/**/*.d.ts"
  ]
}
```

### The `src/assets` Folder

Static files placed here are copied verbatim to the build output. Reference them with paths relative to the domain root:

```html
<!-- Accessing a file at src/assets/images/logo.png -->
<img src="assets/images/logo.png" alt="Logo">
```

### The `src/environments` Pattern (Legacy) vs Modern Approach

Older Angular projects used `environment.ts` / `environment.production.ts` for config switching. The modern approach (Angular 15+) uses **file replacements** in `angular.json` or the `define` option to inject build-time constants. However, the environments pattern still works and is widely used.

---

## Best Practices for Phase 10.1–10.3

**Use strict TypeScript from day one.** The `"strict": true` option in `tsconfig.json` enables all strict checks. Do not turn this off — it will save you from countless runtime bugs.

**Commit `angular.json` carefully.** It is the control panel for your entire build pipeline. Small mistakes here (like wrong asset paths or missing style files) can silently break builds.

**Never commit the `.angular/` cache folder.** It should be in `.gitignore` by default. This folder holds build caches and can be several hundred MB.

**Prefer `ng generate` over creating files manually.** The CLI ensures consistent naming, correct file placement, correct imports, and automatic spec file creation. Manually created files often miss these details.

**Keep `ng update` discipline.** Upgrade Angular every major version using `ng update`. Skipping multiple versions makes migrations much harder. The Angular team provides excellent migration tooling — use it regularly.

**Use `ng add` for third-party libraries.** Libraries that support `ng add` (like Angular Material, NGX-Translate, etc.) will configure themselves correctly. Manual installation often leads to misconfiguration.

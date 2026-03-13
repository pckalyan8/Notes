# Phase 13.12 — Analog.js & Nx Monorepo
## Complete Guide: Angular Meta-Framework & Enterprise Monorepo Tooling

---

## Part 1: Analog.js — Angular's Answer to Next.js

### What Problem Does Analog Solve?

To understand Analog.js, you need to understand what's missing from a plain Angular application. When you run `ng new my-app`, you get a Single Page Application (SPA) that runs entirely in the browser. This is fine for many use cases, but SPAs have known limitations: poor initial load performance because the browser must download, parse, and execute JavaScript before rendering anything visible; poor SEO because search engine crawlers may not execute JavaScript at all; and no native support for server-side data fetching colocated with your routes.

Frameworks like Next.js (for React) and Nuxt (for Vue) solve these problems by providing a full-stack meta-framework layer on top of the UI framework. They add file-based routing, server-side rendering, static generation, and API routes — all integrated into one cohesive developer experience.

Analog.js is exactly this for Angular. It wraps Angular in a Vite-powered meta-framework that adds file-based routing, SSR/SSG support, API routes, and Markdown pages — while still being 100% Angular underneath. Everything you know about Angular still applies: components, services, signals, RxJS, Angular Material. Analog just adds the infrastructure layer.

### Installation and Project Creation

```bash
# Create a new Analog project
npm create analog@latest

# Or add Analog to an existing Angular workspace via Nx
npx nx g @analogjs/platform:app my-app
```

### File-Based Routing

This is one of Analog's most compelling features. Instead of defining routes in a `Routes` array, you create files in a specific directory structure and Analog generates the routing configuration automatically.

The routing convention maps file paths to URL paths. A file at `src/app/pages/index.page.ts` maps to `/`. A file at `src/app/pages/books.page.ts` maps to `/books`. A file at `src/app/pages/books/[id].page.ts` maps to `/books/:id` with a dynamic parameter.

```
src/app/pages/
├── index.page.ts            → /
├── about.page.ts            → /about
├── books/
│   ├── index.page.ts        → /books
│   ├── [id].page.ts         → /books/:id   (dynamic segment)
│   └── [id]/
│       └── reviews.page.ts  → /books/:id/reviews
└── (auth)/                  → route group (doesn't affect URL)
    ├── login.page.ts        → /login
    └── register.page.ts     → /register
```

Page components are regular Angular components with a `.page.ts` extension. Analog detects these automatically:

```typescript
// src/app/pages/books/[id].page.ts
import { Component, inject } from '@angular/core';
import { injectLoad, toObservable } from '@analogjs/router';
import { AsyncPipe } from '@angular/common';
import type { load } from './[id].server.ts';  // type-safe from the server loader

@Component({
  standalone: true,
  imports: [AsyncPipe],
  template: `
    @if (book()) {
      <article>
        <h1>{{ book()!.title }}</h1>
        <p>by {{ book()!.author }}</p>
        <p>{{ book()!.description }}</p>
      </article>
    }
  `,
})
export default class BookDetailPage {
  // injectLoad() provides access to data loaded by the server-side loader
  readonly data = injectLoad<typeof load>();
  readonly book = computed(() => this.data().book);
}
```

### Server Loaders — Fetching Data on the Server

Server loaders are the mechanism Analog uses to fetch data on the server before the page renders. This is similar to `loader` in Remix or `getServerSideProps` in Next.js. The loader runs server-side, the data is serialized and sent to the browser, and the component receives it before it renders.

```typescript
// src/app/pages/books/[id].server.ts
import { PageServerLoad } from '@analogjs/router';

export const load = async ({ params, fetch }: PageServerLoad) => {
  // This code runs on the server — you can securely access databases,
  // call internal APIs, or read environment variables
  const book = await fetch(`/api/books/${params['id']}`).then(r => r.json());

  if (!book) {
    // Throw a redirect or error response
    throw redirect('/books');
  }

  return { book };
};
```

The `params`, `request`, and `fetch` utilities are available in the loader context. Because this runs on the server, you can safely include authentication checks, database queries, or access to secrets that should never be exposed to the browser.

### API Routes

Analog lets you define backend API endpoints alongside your Angular pages, colocated in the same project. These are Node.js functions that respond to HTTP requests.

```typescript
// src/server/routes/api/books/[id].ts
import { defineEventHandler, getRouterParam, createError } from 'h3';

export default defineEventHandler(async (event) => {
  const id = getRouterParam(event, 'id');

  const book = await database.books.findById(id);

  if (!book) {
    throw createError({ statusCode: 404, message: 'Book not found' });
  }

  return book;  // automatically serialized to JSON
});
```

Analog uses the `h3` HTTP framework (the same one Nuxt and Nitro use) for API routes, which runs on Node.js by default but can also be deployed to Cloudflare Workers, Vercel Edge, or Deno Deploy.

### Single-File Components (`.analog` extension)

Analog introduces an optional single-file component format that combines template, script, and styles in one file — similar to Vue's `.vue` files:

```html
<!-- src/app/pages/books.analog -->
<script lang="ts">
  import { BooksService } from '~/services/books.service';
  import { inject } from '@angular/core';

  const booksService = inject(BooksService);
  const books = signal<Book[]>([]);

  onInit(() => {
    booksService.getAll().subscribe(data => books.set(data));
  });
</script>

<template>
  <h1>Books</h1>
  @for (book of books(); track book.id) {
    <app-book-card [book]="book" />
  }
</template>

<style>
  h1 { color: var(--primary-color); }
</style>
```

This format is optional — you can mix regular `.ts` component files with `.analog` files freely.

### Markdown Content Pages

Analog supports Markdown files as routes, making it trivial to build documentation sites or blogs where content is written in Markdown but rendered by Angular:

```markdown
<!-- src/app/pages/(blog)/angular-signals.md -->
---
title: Understanding Angular Signals
date: 2026-01-15
author: Jane Developer
---

# Understanding Angular Signals

Angular Signals are a reactive primitive that lets you declare...

<!-- You can even use Angular components inside Markdown -->
<app-code-demo [code]="signalExample" />
```

### Deployment Options

Because Analog uses Nitro as its server engine (the same as Nuxt 3), it can deploy to any of Nitro's supported targets: Node.js, Cloudflare Workers, Vercel, Netlify, AWS Lambda, Deno Deploy, and more. For static sites, you get a fully pre-rendered output that can be hosted on any CDN.

### Vitest Support — `@analogjs/vitest-angular`

Analog provides first-class Vitest support, which is the modern test runner recommended for Angular in 2025/2026 (Karma is being deprecated). The `@analogjs/vitest-angular` package configures Vitest to work with Angular's template compilation and dependency injection.

```typescript
// vitest.config.ts
import { defineConfig } from 'vitest/config';
import angular from '@analogjs/vite-plugin-angular';

export default defineConfig({
  plugins: [angular()],
  test: {
    globals: true,
    environment: 'jsdom',
    setupFiles: ['src/test-setup.ts'],
    include: ['**/*.spec.ts'],
    coverage: {
      reporter: ['text', 'lcov'],
    }
  }
});
```

### When To Choose Analog vs Angular CLI with SSR

Use Analog when you want Next.js-style developer experience with file-based routing, colocated API routes, and Markdown content. It's excellent for content-heavy sites, blogs, documentation, and marketing pages that share code with your Angular application. It's also the right choice when you want the Vite ecosystem (plugins, tooling) deeply integrated with Angular.

Use Angular CLI with SSR (`ng new --ssr`) when you're building a large enterprise SPA where SSR is needed for SEO and performance, but you don't need file-based routing or API routes. If your team already has Angular expertise and you're adding SSR to an existing app, the Angular CLI's built-in SSR support is simpler to adopt than migrating to Analog.

---

## Part 2: Nx — Monorepo Build System for Angular

### What Is a Monorepo and Why Should You Care?

A monorepo is a single repository that contains multiple related projects — multiple frontend applications, backend services, and shared libraries — all managed together. The opposite approach is a polyrepo, where each project lives in its own separate repository.

Monorepos sound deceptively simple but they introduce a challenge: as the codebase grows, build times and test times explode because running all builds and tests every time someone pushes a change becomes impractical. A monorepo without tooling is worse than separate repos.

This is exactly the problem Nx solves. Nx is an intelligent build system that understands the dependency graph between your projects. When you change a file in a shared utility library, Nx knows exactly which projects depend on that library and only rebuilds and retests those projects — everything else is either skipped or served from cache. This is what makes monorepos at scale practical.

### Nx Setup for Angular

```bash
# Create a new Nx workspace with Angular
npx create-nx-workspace@latest my-workspace --preset=angular-monorepo

# Or add Nx to an existing Angular project
npm install -g nx
nx init

# Add Angular support to an existing Nx workspace
npm install @nx/angular
```

The resulting workspace structure looks like this:

```
my-workspace/
├── apps/
│   ├── store/                    # Angular SPA
│   ├── admin/                    # Second Angular SPA
│   └── api/                      # Node.js/NestJS backend
├── libs/
│   ├── shared/
│   │   ├── ui/                   # UI components used by both apps
│   │   ├── data-access/          # HTTP services & state
│   │   └── util/                 # Pure utilities (pipes, validators)
│   ├── store/
│   │   ├── feature-checkout/     # Complete checkout feature
│   │   └── feature-catalog/      # Product catalog feature
│   └── admin/
│       └── feature-dashboard/    # Admin dashboard feature
├── nx.json                       # Nx workspace configuration
└── package.json
```

### The Nx Library Layering System

The most important architectural concept in Nx is the **library layering convention**. Projects are organized into layers, and strict rules govern which layers can import from which. Nx enforces these rules via ESLint.

The four standard layers, from most application-specific to most general, are as follows. The **feature** layer contains smart components with routing, business logic, and state management — it can import from all other layers. The **ui** layer contains presentational (dumb) components with inputs and outputs but no knowledge of routing or state — it can only import from `util`. The **data-access** layer contains services, NgRx stores, and HTTP calls — it can import from `util`. The **util** layer contains pure utility functions, models, constants, and validators — it has no dependencies on Angular or any other layer.

```typescript
// nx.json — enforcing module boundaries with tags
{
  "projects": {
    "shared-ui": {
      "tags": ["scope:shared", "type:ui"]
    },
    "store-feature-checkout": {
      "tags": ["scope:store", "type:feature"]
    }
  }
}
```

```json
// .eslintrc.json — the boundary rules
{
  "@nx/enforce-module-boundaries": [
    "error",
    {
      "depConstraints": [
        {
          "sourceTag": "type:feature",
          "onlyDependOnLibsWithTags": ["type:feature", "type:ui", "type:data-access", "type:util"]
        },
        {
          "sourceTag": "type:ui",
          "onlyDependOnLibsWithTags": ["type:ui", "type:util"]
        },
        {
          "sourceTag": "type:data-access",
          "onlyDependOnLibsWithTags": ["type:data-access", "type:util"]
        },
        {
          "sourceTag": "type:util",
          "onlyDependOnLibsWithTags": ["type:util"]
        },
        {
          "sourceTag": "scope:store",
          "onlyDependOnLibsWithTags": ["scope:store", "scope:shared"]
        },
        {
          "sourceTag": "scope:admin",
          "onlyDependOnLibsWithTags": ["scope:admin", "scope:shared"]
        }
      ]
    }
  ]
}
```

When a developer accidentally imports from a library that violates these constraints, ESLint reports an error immediately in the IDE and in CI. This **automatically prevents architectural drift** — the common problem where well-designed boundaries erode over time as teams take shortcuts.

### Nx Generators — Scaffolding Code Consistently

Nx provides generators (similar to Angular schematics) that create code with the correct structure, including the correct tags, the correct barrel exports, and the correct test setup.

```bash
# Generate a new Angular library
nx g @nx/angular:library shared-ui --directory=libs/shared/ui --tags="scope:shared,type:ui"

# Generate a component inside a library
nx g @nx/angular:component --project=shared-ui button --standalone

# Generate a feature library for the store app
nx g @nx/angular:library feature-checkout --directory=libs/store/feature-checkout --tags="scope:store,type:feature"

# Generate an NgRx feature store
nx g @ngrx/schematics:feature --project=store-feature-checkout checkout
```

Using generators ensures every library is created with the same structure, same ESLint configuration, same test setup, and same tagging. This is important for consistency across large teams.

### Nx Affected — Only Build What Changed

The `nx affected` command is one of Nx's most powerful features. It analyzes the dependency graph and your git changes to determine which projects are actually affected by your change.

```bash
# Show which projects are affected compared to main branch
nx show projects --affected --base=main --head=HEAD

# Only test projects affected by changes since main
nx affected --target=test --base=main --head=HEAD

# Only build affected projects
nx affected --target=build

# Only lint affected projects  
nx affected --target=lint
```

Imagine your workspace has 20 Angular libraries and 3 applications. You made a change to the `shared-ui` library. Nx knows that `store-feature-checkout` depends on `shared-ui`, `store-app` depends on `store-feature-checkout`, and `admin-app` does not depend on `shared-ui` at all. So `nx affected --target=test` only runs tests for `shared-ui`, `store-feature-checkout`, and `store-app` — saving the time it would take to test the admin app unnecessarily. In a large workspace this can reduce CI times from 30 minutes to 5 minutes.

### Nx Caching — Local and Remote

Nx caches the output of every task. When you run `nx build store-app`, Nx stores the build output and the hash of all inputs (source files, dependencies, configuration). The next time you run the same command with the same inputs, Nx returns the cached result instantly without rebuilding.

```bash
# Enable the local cache
nx cache  # it's enabled by default

# Clear the cache
nx reset
```

For teams, Nx Cloud provides a **remote distributed cache**. When a CI machine builds something, the result is stored in Nx Cloud. When another CI machine — or your local machine — runs the same command with the same inputs, it downloads the cached result rather than rebuilding from scratch. This can make CI pipelines nearly instantaneous for commands that already ran earlier in the day.

```bash
# Connect your workspace to Nx Cloud (one-time setup)
npx nx connect
```

### Task Distribution — Nx Agents

For very large workspaces, Nx can distribute tasks across multiple machines in parallel using **Nx Agents**. Instead of running all 500 tests on one CI machine, Nx can distribute them across 10 machines, each running 50 tests in parallel, reducing total test time by an order of magnitude.

```yaml
# .github/workflows/ci.yml
- name: Start Nx Agents
  run: npx nx-cloud start-ci-run --distribute-on="5 linux-medium-js"

- name: Run all affected tasks
  run: |
    nx affected --target=build &
    nx affected --target=test &
    nx affected --target=lint &
    wait
```

### Nx Console — VS Code and JetBrains Extension

The Nx Console extension for VS Code provides a GUI for everything Nx does from the command line. You can browse your project graph visually, run generators from a form UI, see which tasks are cacheable, and run affected commands with one click. It is highly recommended for teams new to Nx.

### Nx Powerpack

Nx Powerpack is a set of enterprise features available under a paid license. It includes `@nx/conformance` (enforce custom rules across all projects in the workspace), `@nx/sync` (synchronize workspace structure from a template), and enhanced security features. Most teams won't need Powerpack; it targets large enterprise organizations managing hundreds of projects.

### Full-Stack Monorepo — Angular + Node.js

One of Nx's most compelling use cases is co-locating your Angular frontend and NestJS (or Express) backend in the same repository, with shared TypeScript types and DTOs:

```
my-workspace/
├── apps/
│   ├── frontend/    # Angular app
│   └── api/         # NestJS API
└── libs/
    └── shared/
        └── types/   # Shared interfaces — used by BOTH apps!
            ├── user.interface.ts
            ├── book.dto.ts
            └── api-response.interface.ts
```

```typescript
// libs/shared/types/src/book.dto.ts
// This file is used by BOTH the Angular frontend and the NestJS backend!
export interface CreateBookDto {
  title: string;
  author: string;
  genre: 'fiction' | 'non-fiction' | 'biography';
  price: number;
}

export interface BookResponseDto extends CreateBookDto {
  id: string;
  createdAt: string;
}
```

When you change the `BookResponseDto`, TypeScript catches type errors in both the Angular templates and the NestJS controllers **at the same time** in the same IDE session. This eliminates an entire class of frontend-backend contract bugs.

---

## Important Points & Best Practices

**Analog is additive, not a replacement.** Everything you know about Angular applies inside an Analog project. Analog adds file-based routing and server capabilities on top. You don't need to relearn Angular; you just add the Analog layer.

**Server loaders in Analog are not Angular services.** They run in a Node.js environment without the Angular DI system. You can't inject an Angular service into a loader. Instead, use the `fetch` utility or import Node.js modules directly. Database clients, file system access, and server-side environment variables are all accessible from loaders.

**In Nx, always generate with the correct tags from the start.** Going back and adding tags to existing projects is tedious and error-prone. Establish your tagging strategy (`scope:*` and `type:*`) before creating your first library, then use it consistently in every generator command.

**Nx is not just for large teams.** Even solo developers and small teams benefit from Nx's caching and project graph analysis. If you have more than one Angular app or even two or three libraries, Nx adds value. The common misconception is that Nx only makes sense for enterprise-scale monorepos with 10+ developers.

**Don't put everything in the global store when using Nx libraries.** Feature libraries should manage their own local state (using signals or NgRx ComponentStore) and only promote data to a global store when other feature areas genuinely need it. The `data-access` layer is the right place for global NgRx state — not the `feature` layer.

**Nx's module boundary rules only work if ESLint is running.** Make sure your CI pipeline runs `nx affected --target=lint` before every merge to main. Without this check, the boundary rules exist in configuration but don't actually prevent violations, because developers can bypass ESLint in their IDEs or locally.

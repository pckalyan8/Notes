# Phase 10 — Angular Core Fundamentals
## Part 2: NgModule (Classic) & Standalone Components (Modern)
> Topics: 10.4 · 10.5

---

## 10.4 Modules (NgModule) — Classic Angular

### What is an NgModule?

An NgModule is a class decorated with `@NgModule` that acts as a **compilation context** for a set of Angular components, directives, and pipes. It tells the Angular compiler: "these pieces belong together, and here is what they need to function."

Before Angular 14, *everything* in an Angular application had to live inside an NgModule. Every component you created had to be declared in exactly one module. This was often a source of friction — especially the error "Component X is not a part of any NgModule" — but it enforced clear boundaries.

Understanding NgModules is still important because:
1. Millions of lines of Angular code use them and will for years to come.
2. Many third-party libraries still export NgModules that you need to import.
3. The mental model (what gets compiled together, what gets injected where) still applies even in the standalone world.

### The `@NgModule` Decorator

```typescript
import { NgModule } from '@angular/core';
import { BrowserModule } from '@angular/platform-browser';
import { RouterModule } from '@angular/router';
import { AppComponent } from './app.component';
import { UserProfileComponent } from './user-profile.component';
import { SharedModule } from './shared/shared.module';

@NgModule({
  // declarations: Components, Directives, and Pipes that BELONG to this module.
  // They are only usable within this module unless explicitly exported.
  declarations: [
    AppComponent,
    UserProfileComponent
  ],

  // imports: Other modules whose exported declarations this module needs.
  // Think of it as "I need the things that these other modules export."
  imports: [
    BrowserModule,      // Provides browser-specific services (only in root module)
    RouterModule,       // Enables <router-outlet> and routerLink
    SharedModule        // Our own shared module
  ],

  // exports: Declarations from this module that other modules can use.
  // If you don't export a component, it is private to this module.
  exports: [
    UserProfileComponent
  ],

  // providers: Services registered at the module's injector level.
  // Use providedIn: 'root' in the @Injectable instead — it's tree-shakeable.
  providers: [],

  // bootstrap: The root component(s) Angular should insert into index.html.
  // Only used in the root AppModule.
  bootstrap: [AppComponent]
})
export class AppModule { }
```

### BrowserModule vs CommonModule

This is one of the most common NgModule confusions. The rule is simple: **import `BrowserModule` in the root `AppModule` exactly once**. For every other feature module or shared module, import `CommonModule` instead.

`BrowserModule` re-exports `CommonModule` and additionally provides browser-specific services (like DOM rendering). Importing it in a feature module throws a runtime error.

```typescript
// ❌ Wrong: Feature module importing BrowserModule
@NgModule({
  imports: [BrowserModule]  // Error! Only root module gets BrowserModule
})
export class FeatureModule { }

// ✅ Correct: Feature module importing CommonModule
@NgModule({
  imports: [CommonModule]   // Gives access to *ngIf, *ngFor, async pipe, etc.
})
export class FeatureModule { }
```

### Feature Modules

As your application grows, you break functionality into **feature modules**. A feature module owns everything related to one domain area of your application:

```typescript
// features/dashboard/dashboard.module.ts
import { NgModule } from '@angular/core';
import { CommonModule } from '@angular/common';
import { RouterModule, Routes } from '@angular/router';
import { DashboardComponent } from './dashboard.component';
import { MetricsWidgetComponent } from './metrics-widget.component';

const routes: Routes = [
  { path: '', component: DashboardComponent }
];

@NgModule({
  declarations: [DashboardComponent, MetricsWidgetComponent],
  imports: [
    CommonModule,
    RouterModule.forChild(routes)  // forChild() for feature module routes
  ]
})
export class DashboardModule { }
```

### The Shared Module Pattern

A shared module collects commonly used components, directives, pipes, and Angular Material modules that many feature modules need. It exports them all so they are available wherever the shared module is imported:

```typescript
// shared/shared.module.ts
import { NgModule } from '@angular/core';
import { CommonModule } from '@angular/common';
import { ReactiveFormsModule } from '@angular/forms';
import { MatButtonModule } from '@angular/material/button';
import { LoadingSpinnerComponent } from './components/loading-spinner.component';
import { HighlightDirective } from './directives/highlight.directive';
import { FormatDatePipe } from './pipes/format-date.pipe';

@NgModule({
  declarations: [
    LoadingSpinnerComponent,
    HighlightDirective,
    FormatDatePipe
  ],
  imports: [
    CommonModule,
    ReactiveFormsModule,
    MatButtonModule
  ],
  // Export everything that other modules will need
  exports: [
    CommonModule,           // Re-export so importing modules get *ngIf, etc.
    ReactiveFormsModule,
    MatButtonModule,
    LoadingSpinnerComponent,
    HighlightDirective,
    FormatDatePipe
  ]
})
export class SharedModule { }
```

### The Core Module Pattern

A core module holds **application-wide singletons** — services, interceptors, and guards that should only be instantiated once. It's imported only in `AppModule`.

```typescript
// core/core.module.ts
import { NgModule, Optional, SkipSelf } from '@angular/core';
import { HTTP_INTERCEPTORS } from '@angular/common/http';
import { AuthInterceptor } from './interceptors/auth.interceptor';

@NgModule({
  providers: [
    { provide: HTTP_INTERCEPTORS, useClass: AuthInterceptor, multi: true }
  ]
})
export class CoreModule {
  // Guard against importing CoreModule more than once.
  // The constructor will throw if an instance already exists.
  constructor(@Optional() @SkipSelf() parentModule?: CoreModule) {
    if (parentModule) {
      throw new Error('CoreModule is already loaded. Import it only in AppModule.');
    }
  }
}
```

### Lazy Loading with NgModule

One of the key performance techniques is **lazy loading** feature modules — they are not downloaded until the user navigates to their route. This is done with `loadChildren`:

```typescript
// app-routing.module.ts
const routes: Routes = [
  { path: '', component: HomeComponent },
  {
    path: 'dashboard',
    // The module is loaded lazily. The string after # is the class name.
    // With newer syntax you can use a direct import()
    loadChildren: () =>
      import('./features/dashboard/dashboard.module')
        .then(m => m.DashboardModule)
  }
];
```

### `forRoot()` and `forChild()` Patterns

Some modules like `RouterModule` and `StoreModule` use a pattern where:
- `forRoot()` registers services at the **root injector** (called once in AppModule)
- `forChild()` registers only routing/features without re-registering root services

```typescript
// In AppModule
imports: [RouterModule.forRoot(routes)]   // Sets up the root router service

// In every feature module
imports: [RouterModule.forChild(featureRoutes)]  // Adds child routes only
```

---

## 10.5 Standalone Components (Modern Angular)

### The Problem NgModules Solved (and Created)

NgModules were Angular's solution to compilation contexts — the compiler needed to know exactly what was available in each template. But they introduced significant boilerplate and a common frustration: you had to declare every component in a module, import the module to use the component, and export it if other modules needed it. For small apps, this felt like a lot of ceremony.

### What Are Standalone Components?

A **standalone component** is a component (or directive or pipe) that manages its own dependencies directly, without needing to be declared in an NgModule. It uses an `imports` array right inside the `@Component` decorator to pull in what it needs.

```typescript
// user-card.component.ts — A standalone component
import { Component, Input } from '@angular/core';
import { CommonModule } from '@angular/common';  // For NgClass, NgIf, etc. (pre-v17)
import { RouterLink } from '@angular/router';

@Component({
  selector: 'app-user-card',
  standalone: true,                 // ← This is the magic flag
  imports: [                        // ← Dependencies declared right here
    CommonModule,
    RouterLink
  ],
  template: `
    <div class="card">
      <h2>{{ user.name }}</h2>
      <a [routerLink]="['/users', user.id]">View Profile</a>
    </div>
  `
})
export class UserCardComponent {
  @Input() user!: { name: string; id: number };
}
```

In Angular 17+, the `standalone: true` flag became the **default** for newly generated components, so you often don't even need to write it explicitly (the CLI sets it up automatically).

### Bootstrapping a Standalone Application

Without an `AppModule`, you use `bootstrapApplication()` in `main.ts`:

```typescript
// main.ts
import { bootstrapApplication } from '@angular/platform-browser';
import { appConfig } from './app/app.config';
import { AppComponent } from './app/app.component';

bootstrapApplication(AppComponent, appConfig).catch(console.error);
```

The `appConfig` object replaces the providers array from AppModule:

```typescript
// app.config.ts
import { ApplicationConfig } from '@angular/core';
import { provideRouter } from '@angular/router';
import { provideHttpClient, withFetch } from '@angular/common/http';
import { routes } from './app.routes';

export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(routes),
    provideHttpClient(withFetch())
  ]
};
```

### Using a Standalone Component in Another Standalone Component

Since standalone components manage their own imports, using one in another is simply adding it to the `imports` array:

```typescript
// dashboard.component.ts
import { Component } from '@angular/core';
import { UserCardComponent } from '../user-card/user-card.component';
import { LoadingSpinnerComponent } from '../shared/loading-spinner.component';

@Component({
  selector: 'app-dashboard',
  standalone: true,
  imports: [
    UserCardComponent,         // ← Directly import the component you need
    LoadingSpinnerComponent
  ],
  template: `
    <app-loading-spinner *ngIf="isLoading" />
    <app-user-card [user]="currentUser" />
  `
})
export class DashboardComponent { }
```

### Routing with Standalone Components

Standalone components make lazy loading even simpler with `loadComponent`:

```typescript
// app.routes.ts
import { Routes } from '@angular/router';

export const routes: Routes = [
  {
    path: '',
    loadComponent: () =>
      import('./home/home.component').then(m => m.HomeComponent)
  },
  {
    path: 'dashboard',
    loadComponent: () =>
      import('./dashboard/dashboard.component').then(m => m.DashboardComponent)
  },
  {
    path: 'admin',
    // You can also lazy-load a whole set of routes defined in a separate file
    loadChildren: () =>
      import('./admin/admin.routes').then(m => m.adminRoutes)
  }
];
```

### Standalone Directives and Pipes

Directives and pipes can also be standalone. This is now the default for all CLI-generated artifacts:

```typescript
// highlight.directive.ts
import { Directive, ElementRef, HostListener, Input } from '@angular/core';

@Directive({
  selector: '[appHighlight]',
  standalone: true   // Can be imported directly in any component
})
export class HighlightDirective {
  @Input() appHighlight = 'yellow';

  constructor(private el: ElementRef) {}

  @HostListener('mouseenter')
  onMouseEnter() {
    this.el.nativeElement.style.backgroundColor = this.appHighlight;
  }

  @HostListener('mouseleave')
  onMouseLeave() {
    this.el.nativeElement.style.backgroundColor = '';
  }
}
```

To use this directive in a standalone component, you just add it to imports:

```typescript
@Component({
  standalone: true,
  imports: [HighlightDirective],
  template: `<p appHighlight="lightblue">Hover over me!</p>`
})
export class MyComponent { }
```

### Migrating from NgModule to Standalone

The Angular CLI provides an automated migration schematic:

```bash
ng generate @angular/core:standalone
```

This migration runs in three passes:
1. Converts all components, directives, and pipes to standalone
2. Removes unnecessary NgModule declarations
3. Bootstraps the application using `bootstrapApplication`

For brownfield applications (existing NgModule-based codebases), you can mix standalone and NgModule-based components freely during migration. A standalone component can be declared in an NgModule for incremental adoption.

### Mixing Standalone with NgModules (Interop)

During a migration period, you can use standalone components inside NgModule-based templates by importing them into the module:

```typescript
@NgModule({
  declarations: [OldLegacyComponent],
  imports: [
    CommonModule,
    NewStandaloneComponent,  // ← Import standalone components directly into NgModule
    NewStandaloneDirective
  ]
})
export class LegacyFeatureModule { }
```

---

## NgModule vs Standalone: When to Use Which

In **new projects (2026)**, always use standalone components. The Angular team has made this the default and recommended approach. NgModules add ceremony without benefit for new code.

In **existing NgModule-based projects**, migrate incrementally. Start by making new components standalone, then migrate existing ones when you touch them. Full migration is the goal, but it doesn't need to happen all at once.

The standalone architecture is strictly simpler: fewer files, less boilerplate, clearer dependency ownership, and better tree-shaking since unused components are never accidentally included.

---

## Best Practices for 10.4–10.5

**One component, one file (mostly).** Each component should have its own `.ts`, `.html`, and `.scss` files. Very small components can use inline templates and styles. The CLI generates this structure by default.

**Prefer `providedIn: 'root'` for services.** Instead of adding services to `providers` arrays in modules, use `providedIn: 'root'` in the `@Injectable` decorator. This makes the service available everywhere and allows it to be tree-shaken if unused.

**Don't over-export from SharedModule.** Export only what other modules actually use. Over-exporting inflates every module that imports SharedModule.

**In standalone apps, keep `app.config.ts` clean.** Only application-wide providers (router, HTTP client, animations, error handlers, global interceptors) belong here. Feature-specific providers should be scoped to their components or routes.

**Avoid importing FormsModule and ReactiveFormsModule globally.** Import them only in the components that actually use forms. This keeps bundle sizes lean.

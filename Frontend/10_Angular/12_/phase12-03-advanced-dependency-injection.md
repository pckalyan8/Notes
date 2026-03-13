# Phase 12.3 — Dependency Injection — Advanced

> **Prerequisites:** Basic DI (Phase 10.11), services, `@Injectable`, `inject()` function

---

## Table of Contents
1. [The Angular DI System — Mental Model](#1-the-angular-di-system--mental-model)
2. [Hierarchical DI Tree](#2-hierarchical-di-tree)
3. [ElementInjector vs ModuleInjector](#3-elementinjector-vs-moduleinjector)
4. [Self, SkipSelf, Optional, Host Modifiers](#4-self-skipself-optional-host-modifiers)
5. [ViewProviders vs providers in Components](#5-viewproviders-vs-providers-in-components)
6. [Multi-Providers and Ordered Execution](#6-multi-providers-and-ordered-execution)
7. [Factory Providers with Dependencies](#7-factory-providers-with-dependencies)
8. [InjectionToken — Typed Non-Class Tokens](#8-injectiontoken--typed-non-class-tokens)
9. [Environment Injectors](#9-environment-injectors)
10. [runInInjectionContext()](#10-runInInjectionContext)
11. [Injection Contexts](#11-injection-contexts)
12. [Advanced Provider Recipes](#12-advanced-provider-recipes)
13. [Best Practices](#13-best-practices)

---

## 1. The Angular DI System — Mental Model

Angular's DI is a **hierarchical tree of injectors**. When you inject a token, Angular walks up the injector tree until it finds a provider for that token.

```
Request: inject(UserService)

ComponentInjector (no UserService) → UpwardSearch
  └─ ModuleInjector (no UserService) → UpwardSearch
       └─ RootInjector (has UserService!) → Return instance
```

This tree structure allows:
- **Scoped services** — different instances per route, feature, or component
- **Provider overriding** — child injectors shadow parent providers
- **Testing** — replace real services with mocks without touching the whole tree

---

## 2. Hierarchical DI Tree

### The Full Hierarchy

```
PlatformInjector           ← Created by platformBrowserDynamic()
  └─ RootInjector          ← Created by bootstrapApplication()
       ├─ EnvironmentInjector (route-level lazy modules)
       │    └─ ComponentInjector (for each component)
       │         └─ ElementInjector (directives on the element)
       └─ ComponentInjector
            └─ ...
```

### providedIn Options

```typescript
@Injectable({ providedIn: 'root' })          // Singleton in the root injector
class UserService {}

@Injectable({ providedIn: 'platform' })      // Singleton across multiple apps on the page
class SharedService {}

@Injectable({ providedIn: 'any' })           // New instance per lazy-loaded module
class FormStateService {}

// Component-scoped — new instance per component
@Component({
  providers: [CartService],   // New instance for this component + its children
})
class CheckoutComponent {}
```

### Why `providedIn: 'root'` is Best for Tree Shaking

When a service uses `providedIn: 'root'` and is **never injected**, the bundler tree-shakes it out of the final bundle. Class-decorated providers in `NgModule.providers` are always included even if unused.

```typescript
// ✅ Tree-shakeable — only included in bundle if injected somewhere
@Injectable({ providedIn: 'root' })
class AnalyticsService {}

// ❌ Always bundled, even if unused
@NgModule({ providers: [AnalyticsService] })
class FeatureModule {}
```

---

## 3. ElementInjector vs ModuleInjector

### Two Parallel Injector Hierarchies

Angular actually has **two separate hierarchies** that are checked sequentially:

1. **ElementInjector** — built from `providers` and `viewProviders` in `@Component` / `@Directive` decorators. Tied to the DOM tree.
2. **ModuleInjector** — built from `@NgModule.providers`, `@Injectable({ providedIn })`, and `bootstrapApplication` providers.

When you inject a token:
1. Angular checks the **ElementInjector chain** first (walking up the component tree)
2. If not found, checks the **ModuleInjector chain** (walking up the module tree)
3. If not found anywhere, throws `NullInjectorError`

```
DOM Tree:                       ElementInjector chain:
<app-root>                      RootElement → null
  <app-checkout>                CheckoutElement (provides CartService)
    <app-cart-item>             CartItemElement → null
      <app-price>               PriceElement → null
        inject(CartService)     Search: PriceElement → CartItemElement → CheckoutElement → FOUND!
```

---

## 4. Self, SkipSelf, Optional, Host Modifiers

These modifiers change **where Angular looks** for a provider:

### `@Self()` — Only This Injector

```typescript
import { Self, inject, Optional } from '@angular/core';

@Component({
  providers: [FormService],
  ...
})
export class NestedFormComponent {
  // Look ONLY in this component's injector — don't walk up
  private formService = inject(FormService, { self: true });

  // ✅ With @Optional to avoid error if not found locally
  private localService = inject(MyService, { self: true, optional: true });
}
```

Use case: A component wants only its own instance, not an ancestor's.

### `@SkipSelf()` — Skip This Injector, Look Up

```typescript
@Component({
  providers: [MyService], // provides its own instance
  ...
})
export class ChildComponent {
  // Skip ChildComponent's own providers, look in parent
  private parentService = inject(MyService, { skipSelf: true });
}
```

Use case: A component provides its own service but needs access to the parent's version.

### `@Optional()` — Allow Null (Don't Throw)

```typescript
// Without Optional: throws NullInjectorError if not provided
private service = inject(OptionalService);

// With Optional: returns null if not provided
private service = inject(OptionalService, { optional: true });

// Check before using
if (this.service) {
  this.service.doSomething();
}
```

### `@Host()` — Stop at the Host Element

```typescript
// Directives: stop searching at the component boundary, don't go to module
@Directive({ selector: '[appHighlight]' })
export class HighlightDirective {
  // Only looks in the host component and its element injector
  // Does NOT search parent components or root injector
  private config = inject(HighlightConfig, { host: true, optional: true });
}
```

Use case: Directives that should only be aware of their immediate host component's providers.

### Combining Modifiers

```typescript
// Skip self, but stop at host component boundary, allow null
private service = inject(MyService, { skipSelf: true, host: true, optional: true });
```

---

## 5. ViewProviders vs providers in Components

This is a subtle but important distinction:

### `providers` — Shared with All Children + Projected Content

```typescript
@Component({
  selector: 'app-parent',
  providers: [UserService], // Available to component children AND projected ng-content
  template: `
    <app-child></app-child>       <!-- ✅ Gets this UserService instance -->
    <ng-content></ng-content>     <!-- ✅ Projected content ALSO gets this instance -->
  `,
})
export class ParentComponent {}
```

### `viewProviders` — Only View Children, NOT Projected Content

```typescript
@Component({
  selector: 'app-parent',
  viewProviders: [UserService], // Only available to view children, NOT projected content
  template: `
    <app-child></app-child>       <!-- ✅ Gets this UserService instance -->
    <ng-content></ng-content>     <!-- ❌ Projected content does NOT get this instance -->
  `,
})
export class ParentComponent {}
```

### Practical Use Case

```typescript
@Component({
  selector: 'app-form',
  // Only components within THIS form's template get this FormContext
  // External components projected into this form do NOT get access
  viewProviders: [
    {
      provide: FORM_CONTEXT,
      useFactory: () => new FormContext(),
    }
  ],
  template: `
    <form>
      <app-form-field></app-form-field>  <!-- Gets FORM_CONTEXT -->
      <ng-content></ng-content>          <!-- External content — no access to FORM_CONTEXT -->
    </form>
  `,
})
export class FormComponent {}
```

---

## 6. Multi-Providers and Ordered Execution

`multi: true` allows **multiple values** for the same token — Angular collects them into an array:

### Built-in Multi-Provider Examples
- `HTTP_INTERCEPTORS` — multiple interceptors
- `VALIDATORS` — multiple form validators
- `APP_INITIALIZER` — multiple initialization functions
- `LOCALE_ID` — locale data

### Creating Custom Multi-Providers

```typescript
// Define the token
import { InjectionToken } from '@angular/core';

export interface AppPlugin {
  name: string;
  initialize(): void;
}

export const APP_PLUGINS = new InjectionToken<AppPlugin[]>('APP_PLUGINS');

// Provide multiple implementations
providers: [
  { provide: APP_PLUGINS, useClass: AnalyticsPlugin, multi: true },
  { provide: APP_PLUGINS, useClass: LoggingPlugin, multi: true },
  { provide: APP_PLUGINS, useClass: CachePlugin, multi: true },
]

// Inject — get ALL of them as array
@Injectable({ providedIn: 'root' })
export class PluginManager {
  private plugins = inject(APP_PLUGINS);

  initAll() {
    this.plugins.forEach(p => p.initialize()); // Runs in order of registration
  }
}
```

### APP_INITIALIZER — Run Code Before App Bootstrap

```typescript
import { APP_INITIALIZER } from '@angular/core';

function initializeApp(configService: ConfigService) {
  return () => configService.loadConfig(); // Returns Promise or Observable
}

bootstrapApplication(AppComponent, {
  providers: [
    {
      provide: APP_INITIALIZER,
      useFactory: initializeApp,
      deps: [ConfigService],
      multi: true,
    },
  ],
});
```

---

## 7. Factory Providers with Dependencies

Factory providers compute the provided value dynamically, with access to other injected services:

```typescript
import { InjectionToken, inject } from '@angular/core';

// Token
export const API_URL = new InjectionToken<string>('API_URL');
export const AUTH_HTTP_CLIENT = new InjectionToken<HttpClient>('AUTH_HTTP_CLIENT');

// Factory with dependencies
providers: [
  {
    provide: AUTH_HTTP_CLIENT,
    useFactory: () => {
      const httpClient = inject(HttpClient);
      const authService = inject(AuthService);

      // Return a pre-configured client (conceptual — actual interceptors work differently)
      return new HttpClient(/* ... */);
    },
  },
]
```

### Real Example: Database Connection Factory

```typescript
export const DB_CONNECTION = new InjectionToken<DbConnection>('DB_CONNECTION');

providers: [
  {
    provide: DB_CONNECTION,
    useFactory: () => {
      const config = inject(ConfigService);
      const env = inject(ENVIRONMENT);

      if (env.production) {
        return new ProductionDbConnection(config.dbUrl);
      } else {
        return new InMemoryDbConnection();
      }
    },
  },
]
```

### `useExisting` — Alias Provider

```typescript
// Make TokenA resolve to the same instance as TokenB
providers: [
  UserService,
  { provide: AuthService, useExisting: UserService }, // AuthService is an alias for UserService
]
```

---

## 8. `InjectionToken` — Typed Non-Class Tokens

Use `InjectionToken` when you need to inject **primitives, objects, or interfaces** (not classes):

```typescript
import { InjectionToken } from '@angular/core';

// Typed tokens for configuration
export const API_BASE_URL   = new InjectionToken<string>('API_BASE_URL');
export const FEATURE_FLAGS  = new InjectionToken<FeatureFlags>('FEATURE_FLAGS');
export const MAX_RETRIES    = new InjectionToken<number>('MAX_RETRIES');

// With factory default (tree-shakeable)
export const ENVIRONMENT = new InjectionToken<Environment>('ENVIRONMENT', {
  factory: () => ({ production: false, apiUrl: 'http://localhost:3000' }),
});

// Provide values
bootstrapApplication(AppComponent, {
  providers: [
    { provide: API_BASE_URL,  useValue: 'https://api.example.com' },
    { provide: MAX_RETRIES,   useValue: 3 },
    { provide: FEATURE_FLAGS, useValue: { darkMode: true, newDashboard: false } },
  ],
});

// Inject
@Injectable({ providedIn: 'root' })
export class ProductService {
  private baseUrl = inject(API_BASE_URL);
  private maxRetries = inject(MAX_RETRIES);

  getProducts() {
    return this.http.get<Product[]>(`${this.baseUrl}/products`).pipe(
      retry(this.maxRetries)
    );
  }
}
```

---

## 9. Environment Injectors

`EnvironmentInjector` (introduced Angular 14+) allows you to create injectors **outside the component tree**. Used for:
- Lazy-loaded route injectors
- Dynamic component creation
- Running code with DI context in tests

```typescript
import { EnvironmentInjector, createEnvironmentInjector, inject, runInInjectionContext } from '@angular/core';

// Create a custom environment injector
const customInjector = createEnvironmentInjector(
  [
    { provide: MyService, useClass: MockMyService },
    { provide: FEATURE_FLAGS, useValue: { newFeature: true } },
  ],
  inject(EnvironmentInjector) // Parent injector
);

// Use it to resolve dependencies
const service = customInjector.get(MyService);

// Run code in its context
runInInjectionContext(customInjector, () => {
  const flags = inject(FEATURE_FLAGS);
  console.log(flags.newFeature); // true
});
```

### Environment Injectors in Lazy Routes

When you lazy-load a route, Angular automatically creates a new `EnvironmentInjector` for it:

```typescript
{
  path: 'admin',
  loadChildren: () => import('./admin/admin.routes'),
  providers: [
    // These providers are scoped to the admin route's environment injector
    AdminService,
    { provide: ADMIN_CONFIG, useValue: { maxUsers: 100 } },
  ],
}
```

---

## 10. `runInInjectionContext()`

Runs a function **within an injection context** — enabling `inject()` calls outside the normal constructor/field context:

```typescript
import { runInInjectionContext, EnvironmentInjector, inject } from '@angular/core';

@Injectable({ providedIn: 'root' })
export class DynamicEffectService {
  private injector = inject(EnvironmentInjector);

  createEffect(fn: () => void) {
    // effect() requires injection context — runInInjectionContext provides it
    runInInjectionContext(this.injector, () => {
      effect(fn);
    });
  }
}
```

### Use Case: Creating Effects Dynamically

```typescript
@Component({ ... })
export class DashboardComponent {
  private injector = inject(EnvironmentInjector);

  setupWidgetEffect(widget: Widget) {
    runInInjectionContext(this.injector, () => {
      effect(() => {
        if (widget.dataSignal()) {
          this.renderWidget(widget);
        }
      });
    });
  }
}
```

---

## 11. Injection Contexts

An **injection context** is any scope where `inject()` is valid:

```typescript
// ✅ Valid injection contexts:
@Injectable({ providedIn: 'root' })
class MyService {
  private dep = inject(Dependency);     // Field initializer ✅
  constructor() {
    const dep = inject(Dependency);     // Constructor ✅
  }
}

@Component({ ... })
class MyComponent {
  private dep = inject(Dependency);     // Field initializer ✅
  constructor() {
    const dep = inject(Dependency);     // Constructor ✅
  }
}

// Route guard (functional) ✅
export const myGuard: CanActivateFn = () => {
  const auth = inject(AuthService);     // Guard function ✅
  return auth.isAuthenticated();
};

// Route resolver (functional) ✅
export const myResolver: ResolveFn<User> = (route) => {
  const userService = inject(UserService); // Resolver function ✅
  return userService.getUser(route.params['id']);
};

// ❌ Invalid — not in injection context
class RegularClass {
  method() {
    inject(MyService); // ERROR! Not in injection context
  }
}
```

### The `inject()` Function vs Constructor Injection

```typescript
// Traditional constructor injection
@Injectable({ providedIn: 'root' })
export class UserService {
  constructor(
    private http: HttpClient,
    private authService: AuthService,
    private router: Router
  ) {}
}

// Modern inject() function — cleaner, works in class fields
@Injectable({ providedIn: 'root' })
export class UserService {
  private http         = inject(HttpClient);
  private authService  = inject(AuthService);
  private router       = inject(Router);
  // No constructor needed unless doing initialization logic
}
```

---

## 12. Advanced Provider Recipes

### Providing a Service Based on Platform

```typescript
import { PLATFORM_ID, isPlatformBrowser } from '@angular/common';

export const STORAGE = new InjectionToken<Storage>('STORAGE');

providers: [
  {
    provide: STORAGE,
    useFactory: () => {
      const platformId = inject(PLATFORM_ID);
      return isPlatformBrowser(platformId)
        ? localStorage
        : new InMemoryStorage(); // Server-safe fallback
    },
  },
]
```

### Self-Providing Directive Pattern

```typescript
// A directive that provides itself via an interface token
export const TOOLTIP_DIRECTIVE = new InjectionToken<TooltipDirective>('TOOLTIP_DIRECTIVE');

@Directive({
  selector: '[appTooltip]',
  providers: [
    { provide: TOOLTIP_DIRECTIVE, useExisting: TooltipDirective }
  ],
})
export class TooltipDirective {
  // Other directives/components can inject TOOLTIP_DIRECTIVE
  // without knowing the concrete class
}
```

### Configuration Object Pattern (forRoot equivalent for standalone)

```typescript
export interface LoggingConfig {
  level: 'debug' | 'info' | 'warn' | 'error';
  remote: boolean;
  endpoint?: string;
}

export const LOGGING_CONFIG = new InjectionToken<LoggingConfig>('LOGGING_CONFIG');

export function provideLogging(config: LoggingConfig) {
  return [
    LoggingService,
    { provide: LOGGING_CONFIG, useValue: config },
  ];
}

// Usage in main.ts
bootstrapApplication(AppComponent, {
  providers: [
    provideLogging({ level: 'warn', remote: true, endpoint: '/api/logs' }),
  ],
});
```

---

## 13. Best Practices

1. **Use `inject()` function** over constructor injection in all new code — cleaner, supports inheritance.
2. **`providedIn: 'root'` for most services** — tree-shakeable and singleton by default.
3. **`InjectionToken` for all non-class values** — gives type safety and descriptive names.
4. **Use `viewProviders`** when a parent component's service should NOT leak to projected content.
5. **Use `@Optional()`** defensively for plugin-style services that may or may not be provided.
6. **Multi-providers for extensibility** — plugins, validators, app initializers.
7. **Create scoped providers in route config** (`providers: [...]` in route definition) for features that need isolated service instances.
8. **Never provide services in both `NgModule.providers` AND `providedIn: 'root'`** — creates duplicate instances.

---

> **Summary:** Angular's DI system is one of the most powerful aspects of the framework. The hierarchical injector tree, combined with `InjectionToken`, multi-providers, factory providers, and environment injectors, enables sophisticated patterns like plugin systems, platform-adaptive services, configuration injection, and component-scoped state. Mastering DI unlocks the ability to design truly flexible, testable Angular architectures.

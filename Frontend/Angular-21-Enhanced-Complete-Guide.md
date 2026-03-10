# Angular 21 Complete Interview Guide - Part 1: Fundamentals (Enhanced)

## Table of Contents
1. [Introduction to Angular](#introduction)
2. [Architecture & Core Concepts](#architecture)
3. [Components](#components)
4. [Templates & Data Binding](#templates)
5. [Directives](#directives)
6. [Pipes](#pipes)
7. [Services & Dependency Injection](#services)
8. [Routing](#routing)

---

[![](https://mermaid.ink/img/pako:eNqVVl1vmzAU_SuWq06ZlCAwoRAmTVrbh05ap22t9jDTBwdMY4XgzDhbsqr_ff4IJkRbSfPge699z7nXB8fwBHNeUJjCR0HWC3B5n9VA_c7PQeCBO7mrKLimJauZZLxu7GJekaZRs4DVkoqS5BSUrKrSMxqUUUnHjRR8SdMzP4ji2XwfTn6zQi5StN6Oc15x0S6_O-Ikc5VPcrmnLMtylk8dZTnPfVT8lzIuL3zfb5cL0iyIEGSXgghEx4VyXueCStd7UkZ05gohGhch6hcKu0LBPKLIV5ROLuSBb5xLcMOoICJf7OzKFa8qmmvxRrjzH96madqpN5lkkG4lrYsmg5PJe_BRrZB5RUe49foIy-1qhx74xBoJLgWp84Wd1hMjrMehYl1bFvlBS6aB2HkPisHpNfEyyFbriq5oLRWHpzh0kkV_V1RcYGtegbuTJF9iM_ZRx-1a5naT9ZIWptfOPbGoU2_qga8buqHgjTrqP5U9lNGsjLAxrxXSsI2wMUNYU8DCvgjGBZM7M4V70eDWDmjMwzO1cecOEpisY3FfynQyRuq-oP0zqOIRVsNrhbvjQtLColt3iEOlWPBn8os96r-MwR9GgxRtLUt0o64PFeC9HZTOAa1wLbwXvXy090mW5V5Q3TTe28Hyhzs9eCwXHrgl695jUfEIq-Ffd4rVwGQ4d0g3lXIkvcEfRqdJb4g68VWI93Zw966HTm4N70XD4jsWLbrG7-3p4jsGTSf1DHbeKZuAY_UmZgVMpdjQMVxRsSI6hE-aNoNyoQAZTJVbELHMYFY_K8ya1D84X7UwwTePC5iWpGpUtFkXRNJrRtQ7vktRO6fiim9qCVOEUGhIYPoEtzANEy-YTePpRRQkcTxLojHcwTQOvSj2Yz-ZohglCYqex_CPqep7Mz9EMx8FKIimYZJMx5AWTN3Tt_brwnxkPP8FmETD7g?type=png)](https://mermaid.live/edit#pako:eNqVVl1vmzAU_SuWq06ZlCAwoRAmTVrbh05ap22t9jDTBwdMY4XgzDhbsqr_ff4IJkRbSfPge699z7nXB8fwBHNeUJjCR0HWC3B5n9VA_c7PQeCBO7mrKLimJauZZLxu7GJekaZRs4DVkoqS5BSUrKrSMxqUUUnHjRR8SdMzP4ji2XwfTn6zQi5StN6Oc15x0S6_O-Ikc5VPcrmnLMtylk8dZTnPfVT8lzIuL3zfb5cL0iyIEGSXgghEx4VyXueCStd7UkZ05gohGhch6hcKu0LBPKLIV5ROLuSBb5xLcMOoICJf7OzKFa8qmmvxRrjzH96madqpN5lkkG4lrYsmg5PJe_BRrZB5RUe49foIy-1qhx74xBoJLgWp84Wd1hMjrMehYl1bFvlBS6aB2HkPisHpNfEyyFbriq5oLRWHpzh0kkV_V1RcYGtegbuTJF9iM_ZRx-1a5naT9ZIWptfOPbGoU2_qga8buqHgjTrqP5U9lNGsjLAxrxXSsI2wMUNYU8DCvgjGBZM7M4V70eDWDmjMwzO1cecOEpisY3FfynQyRuq-oP0zqOIRVsNrhbvjQtLColt3iEOlWPBn8os96r-MwR9GgxRtLUt0o64PFeC9HZTOAa1wLbwXvXy090mW5V5Q3TTe28Hyhzs9eCwXHrgl695jUfEIq-Ffd4rVwGQ4d0g3lXIkvcEfRqdJb4g68VWI93Zw966HTm4N70XD4jsWLbrG7-3p4jsGTSf1DHbeKZuAY_UmZgVMpdjQMVxRsSI6hE-aNoNyoQAZTJVbELHMYFY_K8ya1D84X7UwwTePC5iWpGpUtFkXRNJrRtQ7vktRO6fiim9qCVOEUGhIYPoEtzANEy-YTePpRRQkcTxLojHcwTQOvSj2Yz-ZohglCYqex_CPqep7Mz9EMx8FKIimYZJMx5AWTN3Tt_brwnxkPP8FmETD7g)


## 1. Introduction to Angular {#introduction}

### What is Angular?

**Angular** is a **platform and framework** for building single-page client applications using HTML, CSS, and TypeScript. It's developed and maintained by Google and is a complete rewrite of AngularJS (Angular 1.x).

**Historical Context:**
- **AngularJS (2010)**: Original framework, JavaScript-based, pioneered two-way data binding
- **Angular 2+ (2016-present)**: Complete rewrite in TypeScript, component-based architecture, significantly improved performance and tooling
- **Angular 21 (2024)**: Latest version with signals, standalone components, and modern reactive patterns

**Why TypeScript?**
TypeScript is a superset of JavaScript that adds static typing, interfaces, and advanced OOP features. Angular uses TypeScript because it provides:
- **Type Safety**: Catch errors at compile-time rather than runtime
- **Better Tooling**: Enhanced IDE support with autocomplete and refactoring
- **Class and Decorator Support**: Essential for Angular's architecture (decorators like @Component, @Injectable)
- **Modern JavaScript Features**: ES6+ features with transpilation to older browsers

### Core Philosophy

Angular follows several key design principles:

1. **Component-Based Architecture**: The entire application is built as a tree of components, where each component encapsulates its own template, styles, and logic. This promotes reusability and maintainability.

2. **Separation of Concerns**: Templates (view), TypeScript classes (logic), and CSS (styles) are separated, making code easier to understand and maintain.

3. **Dependency Injection**: A design pattern where components and services receive their dependencies from an external source rather than creating them. This makes code more modular, testable, and flexible.

4. **Reactive Programming**: Uses RxJS (Reactive Extensions for JavaScript) to handle asynchronous operations and data streams in a declarative way.

### Angular 21 New Features (2024)

Let me explain each major new feature in detail:

#### 1. Enhanced Signals API
**What are Signals?** Signals are a reactive primitive introduced in Angular 16 and enhanced in v21. They provide a new way to manage state that's more fine-grained than traditional zone-based change detection.

**Why Signals Matter:**
- **Fine-grained Reactivity**: Only components that depend on changed signals re-render, not the entire component tree
- **Better Performance**: Eliminates unnecessary change detection cycles
- **Simpler Mental Model**: No need to worry about when change detection runs
- **Interoperability**: Works seamlessly with RxJS observables

**Core Signal Concepts:**
```typescript
// signal() - Creates a writable reactive value
const count = signal(0);  // Initial value is 0

// To read the value, call it like a function
console.log(count());  // 0

// To update, use set() or update()
count.set(5);           // Set to specific value
count.update(v => v + 1); // Update based on current value

// computed() - Creates a derived reactive value
const doubleCount = computed(() => count() * 2);
// doubleCount automatically updates when count changes

// effect() - Runs side effects when signals change
effect(() => {
  console.log(`Count is now: ${count()}`);
  // This runs automatically whenever count changes
});
```

**When to Use Signals:**
- Component state that changes frequently
- Derived/computed values
- When you need fine-grained reactivity
- For better performance in large component trees

#### 2. Standalone Components by Default
**What Changed?** Previously, all components had to be declared in NgModules. Now, components can exist independently.

**Traditional NgModule Approach (Old Way):**
```typescript
// Every component needed a module
@NgModule({
  declarations: [MyComponent],  // Declare components
  imports: [CommonModule],      // Import other modules
  exports: [MyComponent]        // Export for use in other modules
})
export class MyModule { }
```

**Problems with NgModules:**
- **Boilerplate**: Lots of repetitive code
- **Cognitive Overhead**: Need to understand module system
- **Import Chains**: Complex dependency graphs
- **Bundle Size**: Entire modules loaded even if only one component needed

**Standalone Components (New Way):**
```typescript
@Component({
  selector: 'app-my-component',
  standalone: true,  // This is the key!
  imports: [CommonModule, FormsModule],  // Import dependencies directly
  template: `...`
})
export class MyComponent { }
```

**Benefits:**
- **Simpler Mental Model**: No need to think about modules
- **Better Tree-shaking**: Only import what you need
- **Easier Testing**: No complex module setup
- **Faster Development**: Less boilerplate

#### 3. Improved Server-Side Rendering (SSR)
**What is SSR?** Server-Side Rendering generates HTML on the server instead of in the browser.

**Why SSR?**
- **SEO**: Search engines can crawl fully-rendered HTML
- **Performance**: Faster first contentful paint (FCP)
- **Social Media Sharing**: Preview cards work properly

**Angular 21 SSR Improvements:**
- **Better Hydration**: Smoother transition from server-rendered to client-side app
- **Partial Hydration**: Only hydrate interactive parts of the page
- **Streaming**: Send HTML to browser progressively, don't wait for everything

**How Hydration Works:**
```
1. Server renders HTML with data
2. Browser displays static HTML (users see content immediately)
3. Angular JavaScript loads
4. Angular "hydrates" - attaches event listeners and makes it interactive
5. App is now fully functional
```

#### 4. New Control Flow Syntax
**The Problem with Directives:** `*ngIf`, `*ngFor`, `*ngSwitch` were "structural directives" - special syntax that was confusing for beginners.

**Old Way:**
```typescript
<div *ngIf="condition">Content</div>
<div *ngFor="let item of items">{{ item }}</div>
```

**New Built-in Control Flow (@-syntax):**
```typescript
@if (condition) {
  <div>Content</div>
} @else {
  <div>Alternative</div>
}

@for (item of items; track item.id) {
  <div>{{ item }}</div>
}

@switch (value) {
  @case ('A') { <div>A</div> }
  @case ('B') { <div>B</div> }
  @default { <div>Default</div> }
}
```

**Why This is Better:**
- **More Intuitive**: Looks like regular programming constructs
- **Better Performance**: Optimized by the compiler
- **Type Safety**: Better TypeScript integration
- **Less Magic**: No need to understand structural directive syntax

**Key Difference - track in @for:**
The `track` keyword is required in `@for` loops. It tells Angular how to identify items for efficient re-rendering.

```typescript
@for (item of items; track item.id) {
  // Angular uses item.id to track which items changed
  // Only re-renders items that actually changed
}

// Without track, Angular would re-render everything
// This is especially important for large lists
```

#### 5. Deferred Loading (@defer)
**The Problem:** Loading everything upfront makes initial page load slow.

**Solution:** `@defer` blocks allow you to lazy-load parts of your template.

```typescript
@defer (on viewport) {
  // This expensive component loads only when scrolled into view
  <app-heavy-chart [data]="chartData"></app-heavy-chart>
} @placeholder {
  <!-- Shown while not loaded -->
  <div class="skeleton">Loading chart...</div>
} @loading (minimum 500ms) {
  <!-- Shown during loading -->
  <app-spinner></app-spinner>
} @error {
  <!-- Shown if loading fails -->
  <div>Failed to load chart</div>
}
```

**Defer Triggers:**
- `on viewport` - When element enters viewport
- `on idle` - When browser is idle
- `on immediate` - Immediately
- `on timer(3s)` - After 3 seconds
- `on interaction` - On user interaction
- `on hover` - On mouse hover

**Benefits:**
- **Smaller Initial Bundles**: Don't load code until needed
- **Better Performance**: Faster page load
- **User-Centric**: Load based on user behavior

### Key Features Explained

#### Component-Based Architecture
**What It Means:** Your entire app is a tree of components. Each component is a self-contained unit with its own:
- **Template**: The HTML view
- **Class**: TypeScript code with properties and methods
- **Styles**: Component-specific CSS
- **Metadata**: Configuration via decorators

**Component Tree Example:**
```
AppComponent (root)
├── HeaderComponent
│   ├── LogoComponent
│   └── NavigationComponent
│       └── MenuItemComponent (multiple instances)
├── MainComponent
│   ├── SidebarComponent
│   └── ContentComponent
│       └── ProductListComponent
│           └── ProductCardComponent (multiple instances)
└── FooterComponent
```

**Benefits:**
- **Reusability**: Create once, use everywhere (ProductCardComponent used for each product)
- **Maintainability**: Changes to ProductCardComponent update all instances
- **Testing**: Test components in isolation
- **Encapsulation**: Component styles don't leak to other components

#### Two-Way Data Binding
**What It Is:** Synchronization between the model (TypeScript) and the view (template).

**How It Works:**
```
┌──────────────┐
│  Component   │
│  (TypeScript)│
│   name = ''  │
└──────────────┘
       ↕️  (Two-way binding)
┌──────────────┐
│   Template   │
│    (HTML)    │
│  <input>     │
└──────────────┘

When user types → Component updates
When component changes → View updates
```

**Example:**
```typescript
// Component
export class UserComponent {
  userName = 'John';
}

// Template
<input [(ngModel)]="userName">
<p>Hello, {{ userName }}!</p>

// When user types "Jane", userName becomes "Jane"
// Display updates to "Hello, Jane!"
```

**Under the Hood:**
Two-way binding is syntactic sugar for property binding + event binding:
```typescript
// This:
<input [(ngModel)]="userName">

// Is equivalent to:
<input 
  [ngModel]="userName"           // Property binding (data flows to view)
  (ngModelChange)="userName=$event">  // Event binding (data flows from view)
```

#### Dependency Injection (DI)
**What It Solves:** Without DI, components would need to create their own dependencies:

```typescript
// BAD: Component creates its own dependencies
export class UserComponent {
  private userService = new UserService();  // Tightly coupled!
  private http = new HttpClient();          // Hard to test!
  
  getUsers() {
    return this.userService.getUsers();
  }
}
```

**Problems:**
- **Tight Coupling**: Component is tied to specific implementations
- **Hard to Test**: Can't easily mock dependencies
- **Inflexible**: Can't swap implementations
- **Responsibility Violation**: Component shouldn't manage dependency lifecycle

**With DI:**
```typescript
// GOOD: Dependencies injected
export class UserComponent {
  constructor(private userService: UserService) {}
  // Angular provides UserService automatically
  
  getUsers() {
    return this.userService.getUsers();
  }
}
```

**How DI Works:**
1. **Provider Registration**: Tell Angular how to create a service
2. **Injection**: Angular creates and injects the service when needed
3. **Singleton Pattern**: Same instance shared across app (by default)

```typescript
// Step 1: Mark service as injectable
@Injectable({ providedIn: 'root' })  // Provided at root level (singleton)
export class UserService {
  getUsers() { /* ... */ }
}

// Step 2: Inject into component
export class UserComponent {
  constructor(private userService: UserService) {
    // Angular sees UserService in constructor
    // Looks in DI container
    // Creates instance (or reuses existing one)
    // Injects it
  }
}
```

**Benefits:**
- **Testability**: Easy to inject mock services
- **Flexibility**: Swap implementations (e.g., real vs. mock API)
- **Maintainability**: Centralized service management
- **Memory Efficiency**: Singleton services reused

#### RxJS for Reactive Programming
**What is Reactive Programming?** Programming with asynchronous data streams.

**What are Streams?** Sequences of events over time:
```
User clicks ──1──2────3──4──────5───>
Mouse moves ──────m──m─m─m─m────m──>
HTTP response ──────────────[data]──>
```

**Why RxJS?**
Traditional async handling (callbacks, promises) gets messy:
```typescript
// Callback hell
fetchUser(id, (user) => {
  fetchPosts(user.id, (posts) => {
    fetchComments(posts[0].id, (comments) => {
      // Finally have comments... but this is hard to read!
    });
  });
});
```

**With RxJS:**
```typescript
// Clean, declarative
this.userService.getUser(id).pipe(
  switchMap(user => this.postService.getPosts(user.id)),
  switchMap(posts => this.commentService.getComments(posts[0].id))
).subscribe(comments => {
  // Handle comments
});
```

**Key RxJS Concepts:**
- **Observable**: A stream that can emit values over time
- **Observer**: Something that listens to an Observable
- **Subscription**: The connection between Observable and Observer
- **Operators**: Functions to transform/filter/combine streams

**Common Use Cases:**
- HTTP requests
- User input (search, autocomplete)
- WebSocket messages
- Timer/interval events
- State management

#### CLI (Command Line Interface)
**What It Does:** Angular CLI is a command-line tool that helps you:

1. **Create Projects:**
```bash
ng new my-app
# Creates complete project structure
# Sets up TypeScript, testing, linting
# Configures build tools
```

2. **Generate Code:**
```bash
ng generate component user-profile
# Creates: .ts, .html, .css, .spec.ts files
# Updates necessary imports
# Follows best practices

ng g c user-profile  # Shorthand
ng g s user          # Generate service
ng g m feature       # Generate module
ng g directive highlight  # Generate directive
ng g pipe custom     # Generate pipe
```

3. **Serve Application:**
```bash
ng serve
# Compiles TypeScript
# Starts development server
# Enables live reload
# Opens browser at localhost:4200
```

4. **Build for Production:**
```bash
ng build --configuration production
# Optimizes code
# Minifies files
# Tree-shakes unused code
# Generates source maps
# Outputs to dist/ folder
```

5. **Run Tests:**
```bash
ng test         # Unit tests
ng e2e          # End-to-end tests
ng lint         # Code linting
```

**Why CLI is Important:**
- **Consistency**: Everyone uses same project structure
- **Best Practices**: Built-in conventions
- **Time Saving**: No manual configuration
- **Updates**: Easy to update Angular versions

#### AOT (Ahead-of-Time) Compilation
**What is Compilation?** Converting Angular's template syntax to JavaScript.

**Two Approaches:**

**1. JIT (Just-in-Time) - Old Default:**
```
Browser downloads:
1. Angular framework
2. Compiler
3. Your code (templates as strings)

Then in browser:
4. Compiler converts templates to JavaScript
5. App runs
```

**Problems:**
- Compiler adds ~1MB to bundle
- Slower startup (compilation happens in browser)
- Template errors found at runtime

**2. AOT (Ahead-of-Time) - Modern Default:**
```
At build time:
1. Compiler converts templates to JavaScript
2. Generates optimized code

Browser downloads:
3. Angular framework (no compiler)
4. Pre-compiled code
5. App runs immediately
```

**Benefits:**
- **Smaller Bundles**: No compiler in production
- **Faster Startup**: No compilation needed
- **Template Errors**: Caught at build time
- **Better Security**: No eval() or new Function()
- **Better Performance**: Optimized code

**Example of AOT Transformation:**
```typescript
// Your template:
<h1>{{ title }}</h1>
<button (click)="handleClick()">Click</button>

// AOT compiles to (simplified):
function render() {
  const h1 = document.createElement('h1');
  h1.textContent = this.title;
  
  const button = document.createElement('button');
  button.textContent = 'Click';
  button.addEventListener('click', () => this.handleClick());
  
  return [h1, button];
}
```

---

## 2. Architecture & Core Concepts {#architecture}

### Angular Application Architecture

Let me explain the layered architecture of Angular applications:

```
┌─────────────────────────────────────────────────────┐
│              APPLICATION LAYER                       │
│  User Interface + User Interactions                 │
│                                                      │
│  ┌──────────────────────────────────────────┐      │
│  │         PRESENTATION LAYER               │      │
│  │  Components (Smart + Presentational)     │      │
│  │  Templates, Directives, Pipes            │      │
│  └──────────────────────────────────────────┘      │
│              ↕️ (Data Flow)                          │
│  ┌──────────────────────────────────────────┐      │
│  │         BUSINESS LOGIC LAYER             │      │
│  │  Services (Data Access, State,           │      │
│  │  Business Rules, API Communication)      │      │
│  └──────────────────────────────────────────┘      │
│              ↕️ (Dependency Injection)               │
│  ┌──────────────────────────────────────────┐      │
│  │         INFRASTRUCTURE LAYER             │      │
│  │  HTTP Client, Router, Forms Module       │      │
│  │  Angular Core, RxJS, Zone.js             │      │
│  └──────────────────────────────────────────┘      │
└─────────────────────────────────────────────────────┘
```

**Layer Responsibilities:**

**1. Presentation Layer (Components):**
- **Smart Components (Containers)**: Manage state, communicate with services
- **Presentational Components (Dumb)**: Display data, emit events, no business logic
- **Purpose**: Render UI, handle user interactions, delegate to services

**2. Business Logic Layer (Services):**
- **Data Services**: Fetch/save data via HTTP
- **State Services**: Manage application state
- **Business Logic**: Validation, calculations, transformations
- **Purpose**: Encapsulate reusable logic, isolate from UI

**3. Infrastructure Layer:**
- **Angular Core**: Framework fundamentals
- **Router**: Navigation and routing
- **Forms**: Form handling and validation
- **HTTP**: Communication with backend
- **Purpose**: Provide framework capabilities

### Building Blocks Deep Dive

#### 1. Components
**What They Are:** The fundamental building blocks of Angular apps.

**Anatomy of a Component:**
```typescript
import { Component } from '@angular/core';

@Component({
  // HOW to use this component in templates
  selector: 'app-user-profile',  // <app-user-profile></app-user-profile>
  
  // WHAT to display
  templateUrl: './user-profile.component.html',
  // OR inline: template: `<h1>{{ title }}</h1>`
  
  // HOW it looks
  styleUrls: ['./user-profile.component.css'],
  // OR inline: styles: [`h1 { color: blue; }`]
  
  // METADATA for Angular
  standalone: true,  // New: no module needed
  imports: [CommonModule]  // Dependencies
})
export class UserProfileComponent {
  // PROPERTIES (data)
  userName = 'John Doe';
  age = 25;
  
  // CONSTRUCTOR (dependency injection)
  constructor(private userService: UserService) {}
  
  // LIFECYCLE HOOKS (timing)
  ngOnInit() {
    // Runs after component initializes
  }
  
  // METHODS (behavior)
  updateName(newName: string) {
    this.userName = newName;
  }
}
```

**Component Metadata Explained:**

- **selector**: CSS selector to use this component
  ```typescript
  selector: 'app-user'           // <app-user></app-user>
  selector: '[app-user]'         // <div app-user></div>
  selector: '.app-user'          // <div class="app-user"></div>
  ```

- **template vs templateUrl**:
  - Use `template` for simple, short templates (< 10 lines)
  - Use `templateUrl` for complex templates
  
- **styles vs styleUrls**:
  - Component styles are **encapsulated** by default
  - Styles only apply to this component's template
  - Uses Shadow DOM or emulated encapsulation

**Component Types:**

**Smart Components (Containers):**
```typescript
// Manages state, talks to services
@Component({
  selector: 'app-user-list-container',
  template: `
    <app-search-box (search)="onSearch($event)"></app-search-box>
    <app-user-grid 
      [users]="users$ | async"
      [loading]="loading$ | async"
      (select)="onUserSelect($event)">
    </app-user-grid>
  `
})
export class UserListContainerComponent {
  users$ = this.userService.getUsers();
  loading$ = this.loadingService.loading$;
  
  constructor(
    private userService: UserService,
    private loadingService: LoadingService
  ) {}
  
  onSearch(term: string) {
    this.users$ = this.userService.searchUsers(term);
  }
  
  onUserSelect(user: User) {
    this.router.navigate(['/users', user.id]);
  }
}
```

**Presentational Components (Dumb):**
```typescript
// Just displays data and emits events
@Component({
  selector: 'app-user-grid',
  template: `
    <div *ngIf="loading">Loading...</div>
    <div class="grid">
      <app-user-card 
        *ngFor="let user of users"
        [user]="user"
        (click)="select.emit(user)">
      </app-user-card>
    </div>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush  // Optimization!
})
export class UserGridComponent {
  @Input() users: User[] = [];
  @Input() loading = false;
  @Output() select = new EventEmitter<User>();
}
```

**Benefits of This Pattern:**
- **Testability**: Presentational components easy to test
- **Reusability**: Presentational components reusable
- **Performance**: OnPush change detection
- **Maintainability**: Clear separation of concerns

#### 2. Templates
**What They Are:** HTML with Angular-specific syntax.

**Template Syntax Categories:**

**a) Interpolation - Display Data:**
```typescript
{{ expression }}

// Examples:
{{ userName }}                    // Property
{{ user.firstName }}              // Object property
{{ 1 + 1 }}                      // Expression
{{ getFullName() }}              // Method call
{{ user.age > 18 ? 'Adult' : 'Minor' }}  // Ternary
```

**How Interpolation Works:**
1. Angular evaluates expression
2. Converts result to string
3. Updates DOM when value changes

**b) Property Binding - Set Element Properties:**
```typescript
[property]="expression"

// Examples:
<img [src]="imageUrl">                    // Element property
<button [disabled]="isProcessing">        // Element property
<app-user [data]="userData">              // Component property
<div [class.active]="isActive">           // CSS class
<div [style.color]="textColor">           // Inline style
```

**Property Binding vs Attribute:**
```typescript
// Property binding (preferred)
<img [src]="url">              // Sets DOM property

// Attribute binding (for HTML attributes without DOM properties)
<button [attr.aria-label]="description">  // Sets HTML attribute
<td [attr.colspan]="columnSpan">          // colspan has no property
```

**c) Event Binding - Handle User Actions:**
```typescript
(event)="handler()"

// Examples:
<button (click)="save()">                  // Click event
<input (input)="onInput($event)">          // Input event
<input (keyup.enter)="search()">           // Keyboard event
<div (mouseenter)="highlight()">           // Mouse event
```

**Event Object ($event):**
```typescript
<input (input)="onInput($event)">

onInput(event: Event) {
  const target = event.target as HTMLInputElement;
  console.log(target.value);  // Get input value
}
```

**d) Two-Way Binding - Sync Model and View:**
```typescript
[(ngModel)]="property"

// Example:
<input [(ngModel)]="userName">

// When user types → userName property updates
// When code changes userName → input updates
```

#### 3. Directives
**What They Are:** Classes that add behavior to elements.

**Three Types:**

**a) Component Directives:**
- Components ARE directives with templates
- Most common type

**b) Structural Directives:**
- Change DOM structure (add/remove elements)
- Marked with * or @ prefix
```typescript
*ngIf, *ngFor, *ngSwitch  // Old
@if, @for, @switch        // New
```

**c) Attribute Directives:**
- Change appearance or behavior
- Don't change DOM structure
```typescript
ngClass, ngStyle, ngModel
```

#### 4. Services
**What They Are:** Classes that encapsulate reusable logic.

**When to Use Services:**
- Data fetching/storage
- Business logic
- Calculations
- Logging
- Configuration
- State management

**Service Characteristics:**
```typescript
@Injectable({ providedIn: 'root' })
export class UserService {
  // Singleton: One instance for entire app
  
  private users: User[] = [];
  
  getUsers(): Observable<User[]> {
    return this.http.get<User[]>('/api/users');
  }
  
  addUser(user: User): Observable<User> {
    return this.http.post<User>('/api/users', user);
  }
}
```

**Why Services?**
- **Separation of Concerns**: Keep components lean
- **Reusability**: Share logic across components
- **Testability**: Easy to mock in tests
- **Maintainability**: Centralized business logic

#### 5. Dependency Injection (DI)
**The Problem DI Solves:**

**Without DI (Bad):**
```typescript
export class UserComponent {
  private http = new HttpClient();  // Component creates dependency
  
  // Problems:
  // 1. Hard-coded dependency
  // 2. Can't swap implementations
  // 3. Hard to test (can't mock HttpClient)
  // 4. Component manages lifecycle
}
```

**With DI (Good):**
```typescript
export class UserComponent {
  constructor(private http: HttpClient) {}
  // Angular provides HttpClient
  
  // Benefits:
  // 1. Flexible (can inject different implementation)
  // 2. Testable (can inject mock)
  // 3. Component doesn't manage lifecycle
}
```

**How DI Works - Step by Step:**

**Step 1: Register Provider**
```typescript
@Injectable({ providedIn: 'root' })  // Register in root injector
export class UserService { }
```

**Step 2: Request Dependency**
```typescript
export class UserComponent {
  constructor(private userService: UserService) {}
  // Angular sees UserService in constructor
}
```

**Step 3: Angular's Injector Resolution**
```
1. Check component's injector
2. Check parent component's injector
3. Check module injector
4. Check root injector
5. If not found → Error
6. If found → Create or reuse instance
7. Inject into constructor
```

**Provider Scopes:**
```typescript
// Root level - One instance for entire app (Singleton)
@Injectable({ providedIn: 'root' })
export class GlobalService { }

// Component level - New instance per component
@Component({
  providers: [LocalService]
})
export class MyComponent { }

// Module level - One instance per lazy-loaded module
@NgModule({
  providers: [ModuleService]
})
export class FeatureModule { }
```

#### 6. Modules vs Standalone
**Understanding the Evolution:**

**Why NgModules Were Created:**
In early Angular (v2-v13), modules organized code:
```typescript
@NgModule({
  declarations: [Component1, Component2],  // Own components
  imports: [CommonModule, FormsModule],    // External modules
  providers: [Service1, Service2],         // Services
  exports: [Component1]                    // Make available to others
})
export class FeatureModule { }
```

**Problems:**
- Boilerplate code
- Complex import chains
- Entire module loaded even if only one component needed
- Learning curve for beginners

**Standalone Components Solution:**
```typescript
@Component({
  selector: 'app-example',
  standalone: true,  // No module needed!
  imports: [CommonModule, MyOtherComponent],  // Direct imports
  template: `...`
})
export class ExampleComponent { }
```

**Migration Path:**
You can mix modules and standalone components during transition:
```typescript
// Old module-based component
@Component({ ... })
export class OldComponent { }

// New standalone component
@Component({ standalone: true, ... })
export class NewComponent { }

// Use standalone in module
@NgModule({
  imports: [NewComponent],  // Import standalone component
  declarations: [OldComponent]
})
export class MyModule { }
```

#### 7. Routing
**What It Does:** Navigate between views without page reload.

**How SPA Routing Works:**
```
Traditional Multi-Page App:
/home    → Server sends home.html
/about   → Server sends about.html (full page reload!)
/contact → Server sends contact.html (full page reload!)

Angular SPA:
/home    → Angular shows HomeComponent
/about   → Angular shows AboutComponent (no page reload!)
/contact → Angular shows ContactComponent (no page reload!)

All URLs handled by single index.html
JavaScript changes what's displayed
```

**Routing Components:**
```
┌─────────────────────────────────────┐
│       Browser URL Bar               │
│  http://app.com/products/123        │
└─────────────────────────────────────┘
            ↓
┌─────────────────────────────────────┐
│       Angular Router                │
│  - Matches URL to route             │
│  - Activates component              │
│  - Guards check permissions         │
└─────────────────────────────────────┘
            ↓
┌─────────────────────────────────────┐
│       Router Outlet                 │
│  <router-outlet></router-outlet>    │
│  (Component renders here)           │
└─────────────────────────────────────┘
```

#### 8. Pipes
**What They Are:** Transform data for display.

**Why Pipes?**
- Keep component logic clean
- Reusable transformations
- Declarative syntax in templates

**How Pipes Work:**
```typescript
// In template:
{{ birthday | date:'fullDate' }}

// Angular:
1. Evaluates 'birthday' (e.g., Date object)
2. Passes to 'date' pipe
3. Pipe transforms value
4. Returns formatted string
5. Displays in template
```

This completes the enhanced Part 1 with detailed descriptions. Shall I continue with Parts 2-4?
# Angular 21 Complete Interview Guide - Part 2: Advanced Topics (Enhanced)

## Table of Contents
1. [Forms](#forms)
2. [HTTP Client](#http)
3. [RxJS & Observables](#rxjs)
4. [State Management](#state-management)
5. [Change Detection](#change-detection)
6. [Performance Optimization](#performance)

---

## 1. Forms {#forms}

### Understanding Forms in Angular

Forms are fundamental to web applications - they're how users interact with and submit data. Angular provides two powerful approaches to handle forms, each with different philosophies and use cases.

**Why Forms Matter:**
- Data collection (registration, login, surveys)
- User input validation
- State tracking (pristine, dirty, valid, invalid)
- Error messaging
- Form submission

**Two Approaches:**

### Template-Driven Forms

**Philosophy:** Forms are defined primarily in the template (HTML) with minimal TypeScript code. Similar to AngularJS approach.

**When to Use:**
- Simple forms with basic validation
- Rapid prototyping
- Forms with mostly static structure
- When you prefer working in HTML

**Mental Model:**
```
Template (HTML) is the source of truth
        ↓
Angular automatically creates FormControls
        ↓
Two-way binding synchronizes data
```

**How It Works - Under the Hood:**

1. **You add directives to HTML:**
```html
<form #userForm="ngForm">
  <input name="email" ngModel>
</form>
```

2. **Angular automatically:**
   - Creates a FormGroup for the `<form>`
   - Creates FormControls for each `ngModel`
   - Tracks form state (valid, invalid, dirty, etc.)
   - Provides validation

3. **You access form state:**
```typescript
@ViewChild('userForm') form: NgForm;

onSubmit() {
  console.log(this.form.value);     // { email: 'user@example.com' }
  console.log(this.form.valid);     // true/false
}
```

**Complete Template-Driven Form Example:**

```typescript
import { Component } from '@angular/core';
import { FormsModule } from '@angular/forms';
import { CommonModule } from '@angular/common';

@Component({
  selector: 'app-template-form',
  standalone: true,
  imports: [FormsModule, CommonModule],
  template: `
    <form #userForm="ngForm" (ngSubmit)="onSubmit(userForm)">
      <!-- 
        #userForm="ngForm" creates template reference variable
        Gives access to form state in template
      -->
      
      <!-- Email Input with Validation -->
      <div class="form-group">
        <label>Email:</label>
        <input 
          type="email" 
          name="email"              <!-- name is REQUIRED -->
          [(ngModel)]="user.email"  <!-- Two-way binding -->
          #email="ngModel"          <!-- Reference to FormControl -->
          required                  <!-- Built-in validator -->
          email>                    <!-- Email format validator -->
        
        <!--
          Validation Error Display
          email.invalid: Form control has errors
          email.touched: User has interacted with field
        -->
        <div *ngIf="email.invalid && email.touched" class="error">
          <p *ngIf="email.errors?.['required']">Email is required</p>
          <p *ngIf="email.errors?.['email']">Invalid email format</p>
        </div>
      </div>
      
      <!-- Password with Custom Validation -->
      <div class="form-group">
        <label>Password:</label>
        <input 
          type="password" 
          name="password"
          [(ngModel)]="user.password"
          #password="ngModel"
          required
          minlength="8"
          pattern="^(?=.*[A-Z])(?=.*[0-9]).+$">  <!-- Regex: uppercase + number -->
        
        <div *ngIf="password.invalid && password.touched" class="error">
          <p *ngIf="password.errors?.['required']">Password is required</p>
          <p *ngIf="password.errors?.['minlength']">
            Password must be at least 8 characters
            (Current: {{ password.errors?.['minlength'].actualLength }})
          </p>
          <p *ngIf="password.errors?.['pattern']">
            Password must contain uppercase letter and number
          </p>
        </div>
      </div>
      
      <!-- Select Dropdown -->
      <div class="form-group">
        <label>Country:</label>
        <select name="country" [(ngModel)]="user.country" required>
          <option value="">Select a country</option>
          <option value="us">United States</option>
          <option value="uk">United Kingdom</option>
          <option value="in">India</option>
        </select>
      </div>
      
      <!-- Checkbox -->
      <div class="form-group">
        <label>
          <input 
            type="checkbox" 
            name="agree" 
            [(ngModel)]="user.agree"
            required>
          I agree to terms and conditions
        </label>
      </div>
      
      <!-- Submit Button -->
      <button 
        type="submit" 
        [disabled]="userForm.invalid">  <!-- Disable if form invalid -->
        Submit
      </button>
      
      <!-- Form State Debug Info -->
      <div class="debug">
        <h4>Form State:</h4>
        <p>Valid: {{ userForm.valid }}</p>
        <p>Invalid: {{ userForm.invalid }}</p>
        <p>Pristine: {{ userForm.pristine }}</p>
        <p>Dirty: {{ userForm.dirty }}</p>
        <p>Touched: {{ userForm.touched }}</p>
        <p>Value: {{ userForm.value | json }}</p>
      </div>
    </form>
  `
})
export class TemplateFormComponent {
  // Data model
  user = {
    email: '',
    password: '',
    country: '',
    agree: false
  };
  
  onSubmit(form: NgForm) {
    if (form.valid) {
      console.log('Form Data:', this.user);
      // API call here
      form.reset();  // Clear form after submission
    }
  }
}
```

**Form State Properties Explained:**

```typescript
// VALIDATION STATE
.valid      // All validations pass
.invalid    // At least one validation fails

// USER INTERACTION STATE
.pristine   // User hasn't changed value (opposite of dirty)
.dirty      // User has changed value
.touched    // User has focused and blurred the field
.untouched  // User hasn't interacted with field

// COMMON PATTERNS
// Show errors only after user has interacted:
*ngIf="field.invalid && field.touched"

// Show errors after form submission attempt:
*ngIf="field.invalid && (field.touched || formSubmitted)"

// Disable submit until form is valid:
[disabled]="form.invalid"
```

**Why "name" attribute is required:**
```html
<!-- ❌ WRONG - Angular can't track this field -->
<input [(ngModel)]="user.email">

<!-- ✅ CORRECT - Angular uses 'name' to create FormControl -->
<input name="email" [(ngModel)]="user.email">
```

### Reactive Forms

**Philosophy:** Forms are defined in TypeScript code. The component class is the source of truth.

**When to Use:**
- Complex forms with dynamic fields
- Custom validators
- Cross-field validation
- When you need more control
- Testing is priority
- Forms generated from data

**Mental Model:**
```
Component (TypeScript) is the source of truth
        ↓
You explicitly create FormControls
        ↓
Template binds to FormControls
        ↓
All logic in TypeScript (testable!)
```

**How It Works - Architecture:**

```typescript
// The Reactive Forms Hierarchy:

FormGroup                  // Container for form
├── FormControl            // Individual field
├── FormControl            // Individual field
└── FormGroup             // Nested group
    ├── FormControl
    └── FormControl
```

**Complete Reactive Form Example:**

```typescript
import { Component, OnInit } from '@angular/core';
import { 
  FormBuilder,           // Helper to create forms
  FormGroup,            // Group of controls
  FormControl,          // Individual field
  Validators,           // Built-in validators
  ReactiveFormsModule,  // Module to import
  AbstractControl       // For custom validators
} from '@angular/forms';
import { CommonModule } from '@angular/common';

@Component({
  selector: 'app-reactive-form',
  standalone: true,
  imports: [ReactiveFormsModule, CommonModule],
  template: `
    <!-- 
      [formGroup]="userForm" binds entire form to FormGroup
      formControlName="email" binds input to specific FormControl
    -->
    <form [formGroup]="userForm" (ngSubmit)="onSubmit()">
      
      <!-- Email Field -->
      <div class="form-group">
        <label>Email:</label>
        <input 
          type="email" 
          formControlName="email"   <!-- Binds to FormControl named 'email' -->
          placeholder="Enter email">
        
        <!-- 
          Access control via getter: this.email
          Check validation state and errors
        -->
        <div *ngIf="email.invalid && email.touched" class="error">
          <p *ngIf="email.hasError('required')">Email is required</p>
          <p *ngIf="email.hasError('email')">Invalid email format</p>
        </div>
      </div>
      
      <!-- Password Field -->
      <div class="form-group">
        <label>Password:</label>
        <input 
          type="password" 
          formControlName="password">
        
        <div *ngIf="password.invalid && password.touched" class="error">
          <p *ngIf="password.hasError('required')">Password is required</p>
          <p *ngIf="password.hasError('minlength')">
            Minimum {{ password.getError('minlength').requiredLength }} characters
          </p>
          <p *ngIf="password.hasError('pattern')">
            Must contain uppercase, lowercase, and number
          </p>
        </div>
      </div>
      
      <!-- Confirm Password -->
      <div class="form-group">
        <label>Confirm Password:</label>
        <input 
          type="password" 
          formControlName="confirmPassword">
        
        <!-- 
          Cross-field validation error
          Error is on the FormGroup, not individual field
        -->
        <div *ngIf="userForm.hasError('passwordMismatch') && confirmPassword.touched" 
             class="error">
          <p>Passwords do not match</p>
        </div>
      </div>
      
      <!-- Submit Button -->
      <button 
        type="submit" 
        [disabled]="userForm.invalid">
        Submit
      </button>
      
      <!-- Debug Info -->
      <pre>{{ userForm.value | json }}</pre>
      <pre>{{ userForm.status }}</pre>
    </form>
  `
})
export class ReactiveFormComponent implements OnInit {
  userForm!: FormGroup;  // ! tells TypeScript we'll initialize in ngOnInit
  
  constructor(private fb: FormBuilder) {}
  
  ngOnInit() {
    // METHOD 1: Using FormBuilder (Recommended)
    this.userForm = this.fb.group({
      // FormControl: [initial value, [sync validators], [async validators]]
      email: ['', [
        Validators.required,
        Validators.email
      ]],
      
      password: ['', [
        Validators.required,
        Validators.minLength(8),
        Validators.pattern(/^(?=.*[a-z])(?=.*[A-Z])(?=.*\d).+$/)
      ]],
      
      confirmPassword: ['', Validators.required]
    }, {
      // Form-level validators (cross-field validation)
      validators: this.passwordMatchValidator
    });
    
    // METHOD 2: Manual FormGroup creation (More verbose)
    /*
    this.userForm = new FormGroup({
      email: new FormControl('', [Validators.required, Validators.email]),
      password: new FormControl('', [Validators.required]),
      confirmPassword: new FormControl('', [Validators.required])
    });
    */
  }
  
  // Getters for easy access in template
  get email() {
    return this.userForm.get('email')!;  // ! asserts it's not null
  }
  
  get password() {
    return this.userForm.get('password')!;
  }
  
  get confirmPassword() {
    return this.userForm.get('confirmPassword')!;
  }
  
  /**
   * Custom Validator: Password Match
   * 
   * Validators are functions that return:
   * - null if valid
   * - { errorKey: errorValue } if invalid
   */
  passwordMatchValidator(control: AbstractControl): { [key: string]: boolean } | null {
    const password = control.get('password');
    const confirmPassword = control.get('confirmPassword');
    
    // If fields don't exist yet (during initialization)
    if (!password || !confirmPassword) {
      return null;
    }
    
    // If values don't match
    if (password.value !== confirmPassword.value) {
      return { passwordMismatch: true };  // Error object
    }
    
    return null;  // Valid
  }
  
  onSubmit() {
    if (this.userForm.valid) {
      console.log('Form Data:', this.userForm.value);
      
      // Get raw value (includes disabled fields)
      console.log('Raw Value:', this.userForm.getRawValue());
      
      // API call here
      this.userForm.reset();
    } else {
      // Mark all fields as touched to show errors
      this.userForm.markAllAsTouched();
    }
  }
}
```

**FormControl Methods Explained:**

```typescript
const control = this.userForm.get('email');

// SET VALUES
control.setValue('new@example.com');        // Set value
control.patchValue('partial@example.com');  // Same as setValue for single control
this.userForm.patchValue({ email: 'new@example.com' });  // Partial update

// GET VALUES
control.value                    // Current value
this.userForm.value             // All values (excludes disabled)
this.userForm.getRawValue()     // All values (includes disabled)

// VALIDATION
control.valid                   // Is valid?
control.invalid                 // Has errors?
control.errors                  // Error object { required: true }
control.hasError('required')    // Has specific error?
control.getError('minlength')   // Get error details

// STATE
control.pristine    // Not modified
control.dirty       // Modified
control.touched     // Focused and blurred
control.untouched   // Never focused

// ENABLE/DISABLE
control.disable()                // Disable field
control.enable()                 // Enable field
control.disabled                 // Is disabled?

// STATE CHANGES
control.markAsTouched()          // Mark as touched
control.markAsDirty()            // Mark as dirty
control.markAsPristine()         // Mark as pristine
control.updateValueAndValidity() // Recalculate validation

// RESET
control.reset()                  // Reset to initial state
control.reset('new value')       // Reset with new value
```

### FormArray - Dynamic Form Fields

**What is FormArray?** A collection of FormControls that can grow/shrink dynamically.

**Use Cases:**
- Adding multiple skills
- Multiple phone numbers
- Dynamic list of items
- Order items
- Survey questions

**How FormArray Works:**

```typescript
FormArray
├── FormControl (0)  ← Can add/remove
├── FormControl (1)  ← Can add/remove
├── FormControl (2)  ← Can add/remove
└── ... dynamic length
```

**Complete FormArray Example:**

```typescript
import { Component, OnInit } from '@angular/core';
import { FormBuilder, FormGroup, FormArray, Validators } from '@angular/forms';

@Component({
  selector: 'app-dynamic-form',
  template: `
    <form [formGroup]="profileForm" (ngSubmit)="onSubmit()">
      <h3>Skills</h3>
      
      <!-- 
        formArrayName="skills" binds to FormArray
        Loop through controls
      -->
      <div formArrayName="skills">
        <div *ngFor="let skill of skills.controls; let i = index" class="skill-item">
          <!-- 
            [formControlName]="i" binds to FormControl at index i
            Note: Use property binding [formControlName] for dynamic index
          -->
          <input 
            [formControlName]="i" 
            placeholder="Skill {{ i + 1 }}">
          
          <button 
            type="button" 
            (click)="removeSkill(i)"
            [disabled]="skills.length === 1">  <!-- Keep at least one -->
            Remove
          </button>
        </div>
      </div>
      
      <button type="button" (click)="addSkill()">
        Add Skill
      </button>
      
      <button type="submit" [disabled]="profileForm.invalid">
        Submit
      </button>
      
      <!-- Display current skills -->
      <div class="debug">
        <h4>Current Skills:</h4>
        <ul>
          <li *ngFor="let skill of skills.value">{{ skill }}</li>
        </ul>
      </div>
    </form>
  `
})
export class DynamicFormComponent implements OnInit {
  profileForm!: FormGroup;
  
  constructor(private fb: FormBuilder) {}
  
  ngOnInit() {
    this.profileForm = this.fb.group({
      skills: this.fb.array([
        this.fb.control('', Validators.required)  // Initial skill
      ])
    });
  }
  
  // Getter to access skills FormArray
  get skills(): FormArray {
    return this.profileForm.get('skills') as FormArray;
  }
  
  // Add new skill
  addSkill() {
    const newSkill = this.fb.control('', Validators.required);
    this.skills.push(newSkill);  // Add to end of array
  }
  
  // Remove skill at index
  removeSkill(index: number) {
    this.skills.removeAt(index);
  }
  
  onSubmit() {
    console.log('Skills:', this.skills.value);
    // Output: ['JavaScript', 'TypeScript', 'Angular']
  }
}
```

**FormArray with Complex Objects:**

```typescript
// Instead of simple FormControls, use FormGroups
interface Education {
  school: string;
  degree: string;
  year: number;
}

ngOnInit() {
  this.form = this.fb.group({
    education: this.fb.array([
      this.createEducationGroup()
    ])
  });
}

createEducationGroup(): FormGroup {
  return this.fb.group({
    school: ['', Validators.required],
    degree: ['', Validators.required],
    year: ['', [Validators.required, Validators.min(1900)]]
  });
}

get education(): FormArray {
  return this.form.get('education') as FormArray;
}

addEducation() {
  this.education.push(this.createEducationGroup());
}

// Template
<div formArrayName="education">
  <div *ngFor="let edu of education.controls; let i = index" 
       [formGroupName]="i">  <!-- Bind to FormGroup at index i -->
    <input formControlName="school">
    <input formControlName="degree">
    <input formControlName="year" type="number">
  </div>
</div>
```

### Custom Validators

**Why Custom Validators?** Built-in validators don't cover all cases:
- Username availability
- Password strength requirements
- Custom business rules
- API-based validation

**Synchronous Validator Example:**

```typescript
import { AbstractControl, ValidationErrors, ValidatorFn } from '@angular/forms';

/**
 * Validator: Forbidden Name
 * Prevents specific names/patterns
 * 
 * @param forbiddenPattern - Regex pattern to forbid
 * @returns ValidatorFn - Function that validates control
 */
export function forbiddenNameValidator(forbiddenPattern: RegExp): ValidatorFn {
  // This function returns another function (closure)
  // The returned function is the actual validator
  
  return (control: AbstractControl): ValidationErrors | null => {
    // If no value, don't validate (let 'required' handle that)
    if (!control.value) {
      return null;
    }
    
    // Test if value matches forbidden pattern
    const forbidden = forbiddenPattern.test(control.value);
    
    // If forbidden, return error object
    // Error key: 'forbiddenName'
    // Error value: object with details
    if (forbidden) {
      return {
        forbiddenName: {
          value: control.value,
          pattern: forbiddenPattern.toString()
        }
      };
    }
    
    // If valid, return null
    return null;
  };
}

// Usage in component:
this.form = this.fb.group({
  username: ['', [
    Validators.required,
    forbiddenNameValidator(/admin|root|system/i)  // Can't be admin, root, or system
  ]]
});

// In template:
<div *ngIf="username.hasError('forbiddenName')">
  Username "{{ username.getError('forbiddenName').value }}" is not allowed
</div>
```

**Password Strength Validator:**

```typescript
export function passwordStrengthValidator(): ValidatorFn {
  return (control: AbstractControl): ValidationErrors | null => {
    const value = control.value;
    
    if (!value) {
      return null;
    }
    
    // Requirements
    const hasUpperCase = /[A-Z]/.test(value);
    const hasLowerCase = /[a-z]/.test(value);
    const hasNumeric = /[0-9]/.test(value);
    const hasSpecial = /[!@#$%^&*(),.?":{}|<>]/.test(value);
    const isLongEnough = value.length >= 8;
    
    const passwordValid = 
      hasUpperCase && 
      hasLowerCase && 
      hasNumeric && 
      hasSpecial && 
      isLongEnough;
    
    if (!passwordValid) {
      return {
        passwordStrength: {
          hasUpperCase,
          hasLowerCase,
          hasNumeric,
          hasSpecial,
          isLongEnough
        }
      };
    }
    
    return null;
  };
}

// Usage:
this.form = this.fb.group({
  password: ['', [Validators.required, passwordStrengthValidator()]]
});

// Template - Show specific missing requirements:
<div *ngIf="password.hasError('passwordStrength')" class="error">
  <p>Password must contain:</p>
  <ul>
    <li [class.valid]="password.getError('passwordStrength').hasUpperCase">
      At least one uppercase letter
    </li>
    <li [class.valid]="password.getError('passwordStrength').hasLowerCase">
      At least one lowercase letter
    </li>
    <li [class.valid]="password.getError('passwordStrength').hasNumeric">
      At least one number
    </li>
    <li [class.valid]="password.getError('passwordStrength').hasSpecial">
      At least one special character
    </li>
    <li [class.valid]="password.getError('passwordStrength').isLongEnough">
      At least 8 characters
    </li>
  </ul>
</div>
```

**Asynchronous Validator (API Check):**

Async validators return an Observable or Promise instead of immediate result. Used for server-side validation.

```typescript
import { AsyncValidatorFn } from '@angular/forms';
import { Observable, of } from 'rxjs';
import { map, catchError, debounceTime, switchMap, first } from 'rxjs/operators';

/**
 * Async Validator: Username Availability
 * Checks if username exists via API call
 */
export class UsernameValidator {
  static createValidator(userService: UserService): AsyncValidatorFn {
    return (control: AbstractControl): Observable<ValidationErrors | null> => {
      // No value, don't validate
      if (!control.value) {
        return of(null);
      }
      
      // Return Observable that emits validation result
      return control.valueChanges.pipe(
        debounceTime(500),     // Wait 500ms after user stops typing
        switchMap(value =>      // Cancel previous API call, start new one
          userService.checkUsernameExists(value).pipe(
            map(exists => {
              // If username exists, return error
              // If available, return null
              return exists ? { usernameExists: true } : null;
            }),
            catchError(() => of(null))  // On error, assume valid
          )
        ),
        first()  // Complete after first emission
      );
    };
  }
}

// Usage:
constructor(private userService: UserService) {}

ngOnInit() {
  this.form = this.fb.group({
    username: ['',
      [Validators.required],  // Sync validators
      [UsernameValidator.createValidator(this.userService)]  // Async validator
    ]
  });
}

// Template:
<input formControlName="username">

<!-- Show loading while checking -->
<span *ngIf="username.pending">
  Checking availability...
</span>

<!-- Show error if taken -->
<span *ngIf="username.hasError('usernameExists') && username.touched">
  Username already taken
</span>
```

**How Async Validation Works:**

```
User types → Debounce → API Call → Result
    ↓
Form status = PENDING
    ↓
API responds
    ↓
Form status = VALID or INVALID
```

This completes the comprehensive Forms section. Shall I continue with HTTP Client, RxJS, and the remaining topics?
# Angular 21 Complete Interview Guide - Part 1: Fundamentals

## Table of Contents
1. [Introduction to Angular](#introduction)
2. [Architecture & Core Concepts](#architecture)
3. [Components](#components)
4. [Templates & Data Binding](#templates)
5. [Directives](#directives)
6. [Pipes](#pipes)
7. [Services & Dependency Injection](#services)
8. [Routing](#routing)

---

## 1. Introduction to Angular {#introduction}

### What is Angular?

Angular is a **TypeScript-based, open-source web application framework** led by Google. It's a complete rewrite from AngularJS (Angular 1.x) and uses a component-based architecture.

**Key Features:**
- Component-based architecture
- Two-way data binding
- Dependency Injection
- RxJS for reactive programming
- Built-in routing and HTTP client
- CLI for scaffolding and tooling
- AOT (Ahead-of-Time) compilation
- Signals (Angular 16+) for reactive state management

### Angular 21 New Features (2024)

- **Enhanced Signals API** - Improved reactivity with `signal()`, `computed()`, `effect()`
- **Standalone Components by Default** - No NgModule required
- **Improved Server-Side Rendering (SSR)** - Better hydration
- **New Control Flow Syntax** - `@if`, `@for`, `@switch` replacing `*ngIf`, `*ngFor`
- **Deferred Loading** - `@defer` for lazy loading
- **Built-in Form Validation Enhancements**
- **Improved DevTools**

---

## 2. Architecture & Core Concepts {#architecture}

### Angular Application Architecture

```
┌─────────────────────────────────────────┐
│           Angular Application            │
├─────────────────────────────────────────┤
│  ┌──────────────────────────────────┐   │
│  │        Components                 │   │
│  │  (UI + Logic + Templates)        │   │
│  └──────────────────────────────────┘   │
│              ↓↑                          │
│  ┌──────────────────────────────────┐   │
│  │         Services                  │   │
│  │  (Business Logic + Data)         │   │
│  └──────────────────────────────────┘   │
│              ↓↑                          │
│  ┌──────────────────────────────────┐   │
│  │      Dependency Injection        │   │
│  │      (DI Container)              │   │
│  └──────────────────────────────────┘   │
│              ↓↑                          │
│  ┌──────────────────────────────────┐   │
│  │         Modules/Standalone       │   │
│  │      (Organization Layer)        │   │
│  └──────────────────────────────────┘   │
└─────────────────────────────────────────┘
```

### Building Blocks

1. **Components** - UI building blocks
2. **Templates** - HTML views
3. **Directives** - DOM manipulation
4. **Services** - Reusable business logic
5. **Dependency Injection** - Service management
6. **Modules/Standalone** - Code organization
7. **Routing** - Navigation
8. **Pipes** - Data transformation

### Module vs Standalone Components

**Traditional NgModule (Legacy):**
```typescript
// app.module.ts
import { NgModule } from '@angular/core';
import { BrowserModule } from '@angular/platform-browser';
import { AppComponent } from './app.component';

@NgModule({
  declarations: [AppComponent],  // Components, directives, pipes
  imports: [BrowserModule],       // Other modules
  providers: [],                  // Services
  bootstrap: [AppComponent]       // Root component
})
export class AppModule { }
```

**Standalone Components (Angular 21 Default):**
```typescript
// main.ts
import { bootstrapApplication } from '@angular/platform-browser';
import { AppComponent } from './app/app.component';

bootstrapApplication(AppComponent, {
  providers: [
    // Global providers
  ]
});

// app.component.ts
import { Component } from '@angular/core';
import { CommonModule } from '@angular/common';

@Component({
  selector: 'app-root',
  standalone: true,
  imports: [CommonModule],  // Import dependencies directly
  template: `<h1>Hello Angular 21</h1>`
})
export class AppComponent { }
```

---

## 3. Components {#components}

### Component Basics

A component consists of:
1. **TypeScript class** - Logic
2. **HTML template** - View
3. **CSS styles** - Presentation
4. **Metadata** - Configuration via `@Component` decorator

**Component Creation:**
```bash
# Using Angular CLI
ng generate component user-profile
# or shorthand
ng g c user-profile
```

**Basic Component Structure:**
```typescript
import { Component, OnInit } from '@angular/core';

@Component({
  selector: 'app-user-profile',      // Custom HTML tag
  templateUrl: './user-profile.component.html',  // External template
  styleUrls: ['./user-profile.component.css'],   // Styles
  standalone: true                    // Standalone component
})
export class UserProfileComponent implements OnInit {
  // Properties
  userName: string = 'John Doe';
  age: number = 25;
  
  // Constructor
  constructor() {
    console.log('Constructor called');
  }
  
  // Lifecycle hook
  ngOnInit(): void {
    console.log('Component initialized');
  }
  
  // Methods
  updateName(newName: string): void {
    this.userName = newName;
  }
}
```

### Inline Templates & Styles

```typescript
@Component({
  selector: 'app-inline-example',
  standalone: true,
  template: `
    <div class="container">
      <h1>{{ title }}</h1>
      <p>Count: {{ count }}</p>
      <button (click)="increment()">Increment</button>
    </div>
  `,
  styles: [`
    .container {
      padding: 20px;
      background: #f0f0f0;
    }
    h1 { color: blue; }
  `]
})
export class InlineExampleComponent {
  title = 'Inline Component';
  count = 0;
  
  increment() {
    this.count++;
  }
}
```

### Component Lifecycle Hooks

Angular calls lifecycle hooks in this order:

```typescript
import { Component, OnInit, OnChanges, DoCheck, AfterContentInit,
         AfterContentChecked, AfterViewInit, AfterViewChecked, 
         OnDestroy, SimpleChanges } from '@angular/core';

@Component({
  selector: 'app-lifecycle',
  template: `<p>Lifecycle Demo</p>`
})
export class LifecycleComponent implements OnInit, OnChanges, DoCheck,
  AfterContentInit, AfterContentChecked, AfterViewInit, 
  AfterViewChecked, OnDestroy {
  
  // 1. Called when input properties change
  ngOnChanges(changes: SimpleChanges): void {
    console.log('1. ngOnChanges', changes);
  }
  
  // 2. Called once after first ngOnChanges
  ngOnInit(): void {
    console.log('2. ngOnInit - Initialize component');
  }
  
  // 3. Called during every change detection run
  ngDoCheck(): void {
    console.log('3. ngDoCheck - Custom change detection');
  }
  
  // 4. Called after content projection (ng-content)
  ngAfterContentInit(): void {
    console.log('4. ngAfterContentInit - Content initialized');
  }
  
  // 5. Called after content check
  ngAfterContentChecked(): void {
    console.log('5. ngAfterContentChecked');
  }
  
  // 6. Called after component's view initialized
  ngAfterViewInit(): void {
    console.log('6. ngAfterViewInit - View ready');
  }
  
  // 7. Called after view checked
  ngAfterViewChecked(): void {
    console.log('7. ngAfterViewChecked');
  }
  
  // 8. Called before component is destroyed
  ngOnDestroy(): void {
    console.log('8. ngOnDestroy - Cleanup');
  }
}
```

**Lifecycle Hook Use Cases:**

| Hook | Use Case |
|------|----------|
| `ngOnChanges` | React to input property changes |
| `ngOnInit` | Initialize component, fetch data |
| `ngDoCheck` | Custom change detection |
| `ngAfterContentInit` | Access projected content |
| `ngAfterViewInit` | Access child components/DOM |
| `ngOnDestroy` | Cleanup, unsubscribe observables |

### Component Communication

#### 1. Parent to Child: @Input()

**Parent Component:**
```typescript
@Component({
  selector: 'app-parent',
  template: `
    <app-child 
      [message]="parentMessage" 
      [user]="currentUser">
    </app-child>
  `
})
export class ParentComponent {
  parentMessage = 'Hello from Parent';
  currentUser = { name: 'John', age: 25 };
}
```

**Child Component:**
```typescript
import { Component, Input } from '@angular/core';

@Component({
  selector: 'app-child',
  template: `
    <p>{{ message }}</p>
    <p>User: {{ user.name }}, Age: {{ user.age }}</p>
  `
})
export class ChildComponent {
  @Input() message: string = '';
  @Input() user!: { name: string; age: number };
  
  // With alias
  @Input('userName') name: string = '';
  
  // Required input (Angular 16+)
  @Input({ required: true }) requiredProp!: string;
  
  // Transform input
  @Input({ transform: booleanAttribute }) isActive: boolean = false;
}
```

#### 2. Child to Parent: @Output() & EventEmitter

**Child Component:**
```typescript
import { Component, Output, EventEmitter } from '@angular/core';

@Component({
  selector: 'app-child',
  template: `
    <button (click)="sendMessage()">Send to Parent</button>
    <button (click)="sendData()">Send Data</button>
  `
})
export class ChildComponent {
  @Output() messageEvent = new EventEmitter<string>();
  @Output() dataEvent = new EventEmitter<{ id: number; name: string }>();
  
  sendMessage() {
    this.messageEvent.emit('Hello from Child!');
  }
  
  sendData() {
    this.dataEvent.emit({ id: 1, name: 'John' });
  }
}
```

**Parent Component:**
```typescript
@Component({
  selector: 'app-parent',
  template: `
    <app-child 
      (messageEvent)="receiveMessage($event)"
      (dataEvent)="receiveData($event)">
    </app-child>
    <p>{{ receivedMessage }}</p>
  `
})
export class ParentComponent {
  receivedMessage = '';
  
  receiveMessage(message: string) {
    this.receivedMessage = message;
  }
  
  receiveData(data: { id: number; name: string }) {
    console.log('Received:', data);
  }
}
```

#### 3. ViewChild & ContentChild

**@ViewChild - Access child component/element:**
```typescript
import { Component, ViewChild, ElementRef, AfterViewInit } from '@angular/core';

@Component({
  selector: 'app-parent',
  template: `
    <input #nameInput type="text">
    <app-child></app-child>
  `
})
export class ParentComponent implements AfterViewInit {
  @ViewChild('nameInput') nameInput!: ElementRef;
  @ViewChild(ChildComponent) childComponent!: ChildComponent;
  
  ngAfterViewInit() {
    // Access native element
    this.nameInput.nativeElement.focus();
    
    // Call child component method
    this.childComponent.someMethod();
  }
}
```

**@ContentChild - Access projected content:**
```typescript
@Component({
  selector: 'app-wrapper',
  template: `
    <div class="wrapper">
      <ng-content></ng-content>
    </div>
  `
})
export class WrapperComponent implements AfterContentInit {
  @ContentChild(ChildComponent) projectedChild!: ChildComponent;
  
  ngAfterContentInit() {
    console.log('Projected child:', this.projectedChild);
  }
}

// Usage
@Component({
  template: `
    <app-wrapper>
      <app-child></app-child>
    </app-wrapper>
  `
})
export class ParentComponent { }
```

#### 4. Service-based Communication

```typescript
// shared.service.ts
import { Injectable } from '@angular/core';
import { BehaviorSubject, Observable } from 'rxjs';

@Injectable({ providedIn: 'root' })
export class SharedService {
  private messageSource = new BehaviorSubject<string>('Initial message');
  currentMessage$ = this.messageSource.asObservable();
  
  changeMessage(message: string) {
    this.messageSource.next(message);
  }
}

// component-a.ts
export class ComponentA {
  constructor(private shared: SharedService) {}
  
  sendMessage() {
    this.shared.changeMessage('Message from A');
  }
}

// component-b.ts
export class ComponentB implements OnInit {
  message = '';
  
  constructor(private shared: SharedService) {}
  
  ngOnInit() {
    this.shared.currentMessage$.subscribe(msg => {
      this.message = msg;
    });
  }
}
```

### Signals (Angular 16+)

Signals provide a reactive way to manage state with better performance:

```typescript
import { Component, signal, computed, effect } from '@angular/core';

@Component({
  selector: 'app-signals-demo',
  template: `
    <p>Count: {{ count() }}</p>
    <p>Double: {{ doubleCount() }}</p>
    <button (click)="increment()">Increment</button>
    <button (click)="reset()">Reset</button>
  `
})
export class SignalsDemoComponent {
  // Signal - writable reactive value
  count = signal(0);
  
  // Computed - derived reactive value
  doubleCount = computed(() => this.count() * 2);
  
  // Effect - side effects when signals change
  constructor() {
    effect(() => {
      console.log(`Count changed to: ${this.count()}`);
    });
  }
  
  increment() {
    this.count.update(value => value + 1);
    // or
    // this.count.set(this.count() + 1);
  }
  
  reset() {
    this.count.set(0);
  }
}
```

**Signal Methods:**
- `signal(initialValue)` - Create writable signal
- `signal().set(value)` - Set new value
- `signal().update(fn)` - Update based on current value
- `computed(() => expression)` - Derived signal
- `effect(() => {})` - Side effects

---

## 4. Templates & Data Binding {#templates}

### Data Binding Types

Angular supports four types of data binding:

```
┌──────────────────────────────────────────────────┐
│         Component (TypeScript)                    │
│                     ↓↑                            │
│              ┌──────────────┐                     │
│              │   Template   │                     │
│              │    (HTML)    │                     │
│              └──────────────┘                     │
└──────────────────────────────────────────────────┘

1. Interpolation:        {{ value }}          [Component → View]
2. Property Binding:     [property]="value"   [Component → View]
3. Event Binding:        (event)="handler()"  [View → Component]
4. Two-way Binding:      [(ngModel)]="value"  [Component ↔ View]
```

### 1. Interpolation

```typescript
@Component({
  template: `
    <h1>{{ title }}</h1>
    <p>{{ 1 + 1 }}</p>
    <p>{{ getMessage() }}</p>
    <p>{{ user.name.toUpperCase() }}</p>
    <img src="{{ imageUrl }}" alt="Logo">
  `
})
export class InterpolationComponent {
  title = 'Angular Interpolation';
  user = { name: 'John' };
  imageUrl = 'assets/logo.png';
  
  getMessage() {
    return 'Hello World';
  }
}
```

### 2. Property Binding

```typescript
@Component({
  template: `
    <!-- Element properties -->
    <img [src]="imageUrl" [alt]="imageAlt">
    <button [disabled]="isDisabled">Click</button>
    <div [id]="dynamicId" [class.active]="isActive">Content</div>
    
    <!-- Component properties -->
    <app-child [userName]="currentUser"></app-child>
    
    <!-- Attribute binding (for non-DOM properties) -->
    <button [attr.aria-label]="ariaLabel">Accessible Button</button>
    <td [attr.colspan]="columnSpan">Cell</td>
    
    <!-- Class binding -->
    <div [class]="cssClasses"></div>
    <div [class.selected]="isSelected"></div>
    <div [class]="{'active': isActive, 'disabled': isDisabled}"></div>
    
    <!-- Style binding -->
    <div [style.color]="textColor"></div>
    <div [style.font-size.px]="fontSize"></div>
    <div [style]="{'color': 'red', 'font-size': '20px'}"></div>
  `
})
export class PropertyBindingComponent {
  imageUrl = 'assets/image.jpg';
  imageAlt = 'Description';
  isDisabled = false;
  isActive = true;
  dynamicId = 'unique-123';
  currentUser = 'John Doe';
  ariaLabel = 'Close dialog';
  columnSpan = 2;
  cssClasses = 'btn btn-primary';
  isSelected = true;
  textColor = 'blue';
  fontSize = 16;
}
```

### 3. Event Binding

```typescript
@Component({
  template: `
    <!-- Click event -->
    <button (click)="handleClick()">Click Me</button>
    <button (click)="handleClick($event)">With Event</button>
    
    <!-- Input events -->
    <input (input)="onInput($event)" (blur)="onBlur()">
    <input (keyup)="onKeyUp($event)" (keyup.enter)="onEnter()">
    
    <!-- Form events -->
    <form (submit)="onSubmit($event)">
      <input (change)="onChange($event)">
      <button type="submit">Submit</button>
    </form>
    
    <!-- Mouse events -->
    <div (mouseenter)="onMouseEnter()" (mouseleave)="onMouseLeave()">
      Hover me
    </div>
    
    <!-- Custom events from child -->
    <app-child (customEvent)="handleCustom($event)"></app-child>
  `
})
export class EventBindingComponent {
  handleClick() {
    console.log('Button clicked');
  }
  
  handleClick(event: Event) {
    console.log('Event:', event);
    event.stopPropagation();
  }
  
  onInput(event: Event) {
    const input = event.target as HTMLInputElement;
    console.log('Input value:', input.value);
  }
  
  onKeyUp(event: KeyboardEvent) {
    console.log('Key:', event.key);
  }
  
  onEnter() {
    console.log('Enter pressed');
  }
  
  onSubmit(event: Event) {
    event.preventDefault();
    console.log('Form submitted');
  }
  
  onChange(event: Event) {
    console.log('Input changed');
  }
  
  onMouseEnter() {
    console.log('Mouse entered');
  }
  
  onMouseLeave() {
    console.log('Mouse left');
  }
  
  handleCustom(data: any) {
    console.log('Custom event:', data);
  }
}
```

### 4. Two-way Binding

```typescript
import { Component } from '@angular/core';
import { FormsModule } from '@angular/forms';

@Component({
  selector: 'app-two-way-binding',
  standalone: true,
  imports: [FormsModule],
  template: `
    <!-- Two-way binding with ngModel -->
    <input [(ngModel)]="userName" placeholder="Enter name">
    <p>Hello, {{ userName }}!</p>
    
    <!-- Expanded form (property + event binding) -->
    <input [ngModel]="userName" (ngModelChange)="userName = $event">
    
    <!-- Custom two-way binding -->
    <app-custom [(value)]="customValue"></app-custom>
    
    <!-- Checkbox -->
    <input type="checkbox" [(ngModel)]="isChecked">
    <p>Checked: {{ isChecked }}</p>
    
    <!-- Radio buttons -->
    <input type="radio" [(ngModel)]="selectedOption" value="option1"> Option 1
    <input type="radio" [(ngModel)]="selectedOption" value="option2"> Option 2
    <p>Selected: {{ selectedOption }}</p>
    
    <!-- Select dropdown -->
    <select [(ngModel)]="selectedCountry">
      <option value="us">USA</option>
      <option value="uk">UK</option>
      <option value="in">India</option>
    </select>
    <p>Country: {{ selectedCountry }}</p>
  `
})
export class TwoWayBindingComponent {
  userName = '';
  customValue = '';
  isChecked = false;
  selectedOption = 'option1';
  selectedCountry = 'us';
}
```

**Custom Two-way Binding:**
```typescript
// custom-input.component.ts
import { Component, Input, Output, EventEmitter } from '@angular/core';

@Component({
  selector: 'app-custom-input',
  template: `
    <input [value]="value" (input)="onInput($event)">
  `
})
export class CustomInputComponent {
  @Input() value = '';
  @Output() valueChange = new EventEmitter<string>();
  
  onInput(event: Event) {
    const input = event.target as HTMLInputElement;
    this.valueChange.emit(input.value);
  }
}

// Usage: <app-custom-input [(value)]="myValue"></app-custom-input>
```

### Template Reference Variables

```typescript
@Component({
  template: `
    <!-- Reference to element -->
    <input #nameInput type="text">
    <button (click)="nameInput.focus()">Focus Input</button>
    <p>Value: {{ nameInput.value }}</p>
    
    <!-- Reference to directive -->
    <form #myForm="ngForm">
      <input name="email" ngModel required>
      <p>Form valid: {{ myForm.valid }}</p>
    </form>
    
    <!-- Reference to component -->
    <app-child #childRef></app-child>
    <button (click)="childRef.someMethod()">Call Child Method</button>
  `
})
export class TemplateRefComponent { }
```

---

## 5. Directives {#directives}

Directives are classes that add behavior to elements. Three types:
1. **Component Directives** - Components are directives with templates
2. **Structural Directives** - Change DOM structure (*ngIf, *ngFor)
3. **Attribute Directives** - Change appearance/behavior (ngClass, ngStyle)

### Structural Directives

#### New Control Flow Syntax (Angular 17+)

**@if - Conditional Rendering:**
```typescript
@Component({
  template: `
    <!-- New @if syntax -->
    @if (isLoggedIn) {
      <p>Welcome, {{ userName }}!</p>
    } @else if (isGuest) {
      <p>Welcome, Guest!</p>
    } @else {
      <p>Please log in</p>
    }
    
    <!-- With expression -->
    @if (users.length > 0) {
      <ul>
        @for (user of users; track user.id) {
          <li>{{ user.name }}</li>
        }
      </ul>
    } @else {
      <p>No users found</p>
    }
  `
})
export class NewSyntaxComponent {
  isLoggedIn = true;
  isGuest = false;
  userName = 'John';
  users = [
    { id: 1, name: 'Alice' },
    { id: 2, name: 'Bob' }
  ];
}
```

**@for - Looping:**
```typescript
@Component({
  template: `
    <!-- Basic @for with track -->
    @for (item of items; track item.id) {
      <div>{{ item.name }}</div>
    }
    
    <!-- With index and other variables -->
    @for (item of items; track item.id; let idx = $index; let isFirst = $first; let isLast = $last) {
      <div [class.first]="isFirst" [class.last]="isLast">
        {{ idx + 1 }}. {{ item.name }}
      </div>
    }
    
    <!-- With empty state -->
    @for (item of items; track item.id) {
      <div>{{ item.name }}</div>
    } @empty {
      <p>No items available</p>
    }
  `
})
export class ForLoopComponent {
  items = [
    { id: 1, name: 'Item 1' },
    { id: 2, name: 'Item 2' },
    { id: 3, name: 'Item 3' }
  ];
}
```

**@switch - Switch Case:**
```typescript
@Component({
  template: `
    @switch (userRole) {
      @case ('admin') {
        <p>Admin Dashboard</p>
      }
      @case ('user') {
        <p>User Dashboard</p>
      }
      @case ('guest') {
        <p>Guest View</p>
      }
      @default {
        <p>Unknown Role</p>
      }
    }
  `
})
export class SwitchComponent {
  userRole = 'admin';
}
```

#### Legacy Structural Directives (Still Supported)

**ngIf:**
```typescript
@Component({
  template: `
    <!-- Basic ngIf -->
    <div *ngIf="isVisible">Content</div>
    
    <!-- With else -->
    <div *ngIf="isLoggedIn; else loggedOut">
      Welcome back!
    </div>
    <ng-template #loggedOut>
      Please log in
    </ng-template>
    
    <!-- With then/else -->
    <div *ngIf="condition; then thenBlock else elseBlock"></div>
    <ng-template #thenBlock>Then content</ng-template>
    <ng-template #elseBlock>Else content</ng-template>
    
    <!-- Store value (ngIf as) -->
    <div *ngIf="user$ | async as user">
      {{ user.name }}
    </div>
  `
})
export class NgIfComponent {
  isVisible = true;
  isLoggedIn = false;
  condition = true;
  user$ = of({ name: 'John' });
}
```

**ngFor:**
```typescript
@Component({
  template: `
    <!-- Basic ngFor -->
    <div *ngFor="let item of items">{{ item }}</div>
    
    <!-- With index -->
    <div *ngFor="let item of items; let i = index">
      {{ i }}: {{ item }}
    </div>
    
    <!-- With trackBy for performance -->
    <div *ngFor="let item of items; trackBy: trackByFn">
      {{ item.name }}
    </div>
    
    <!-- All local variables -->
    <div *ngFor="let item of items; 
                 let i = index;
                 let first = first;
                 let last = last;
                 let even = even;
                 let odd = odd">
      <span [class.highlight]="odd">{{ i }}: {{ item }}</span>
    </div>
  `
})
export class NgForComponent {
  items = ['Apple', 'Banana', 'Cherry'];
  
  objects = [
    { id: 1, name: 'Item 1' },
    { id: 2, name: 'Item 2' }
  ];
  
  trackByFn(index: number, item: any) {
    return item.id; // Unique identifier
  }
}
```

**ngSwitch:**
```typescript
@Component({
  template: `
    <div [ngSwitch]="color">
      <p *ngSwitchCase="'red'">Red color</p>
      <p *ngSwitchCase="'blue'">Blue color</p>
      <p *ngSwitchCase="'green'">Green color</p>
      <p *ngSwitchDefault>Unknown color</p>
    </div>
  `
})
export class NgSwitchComponent {
  color = 'red';
}
```

### Attribute Directives

**ngClass:**
```typescript
@Component({
  template: `
    <!-- String -->
    <div [ngClass]="'class1 class2'">Content</div>
    
    <!-- Array -->
    <div [ngClass]="['class1', 'class2', condition ? 'active' : '']">
      Content
    </div>
    
    <!-- Object (most common) -->
    <div [ngClass]="{
      'active': isActive,
      'disabled': isDisabled,
      'highlight': score > 80
    }">Content</div>
    
    <!-- From method -->
    <div [ngClass]="getClasses()">Content</div>
  `
})
export class NgClassComponent {
  isActive = true;
  isDisabled = false;
  score = 90;
  condition = true;
  
  getClasses() {
    return {
      'active': this.isActive,
      'disabled': this.isDisabled
    };
  }
}
```

**ngStyle:**
```typescript
@Component({
  template: `
    <!-- Object syntax -->
    <div [ngStyle]="{
      'color': textColor,
      'font-size': fontSize + 'px',
      'background-color': isHighlight ? 'yellow' : 'white'
    }">Styled Content</div>
    
    <!-- From method -->
    <div [ngStyle]="getStyles()">Content</div>
  `
})
export class NgStyleComponent {
  textColor = 'blue';
  fontSize = 16;
  isHighlight = true;
  
  getStyles() {
    return {
      'color': this.textColor,
      'font-size': this.fontSize + 'px'
    };
  }
}
```

**ngModel (FormsModule required):**
```typescript
import { FormsModule } from '@angular/forms';

@Component({
  standalone: true,
  imports: [FormsModule],
  template: `
    <input [(ngModel)]="name">
    <p>{{ name }}</p>
  `
})
export class NgModelComponent {
  name = '';
}
```

### Custom Attribute Directive

```typescript
import { Directive, ElementRef, HostListener, Input } from '@angular/core';

@Directive({
  selector: '[appHighlight]',
  standalone: true
})
export class HighlightDirective {
  @Input() appHighlight = 'yellow';
  @Input() defaultColor = 'transparent';
  
  constructor(private el: ElementRef) {
    this.el.nativeElement.style.backgroundColor = this.defaultColor;
  }
  
  @HostListener('mouseenter') onMouseEnter() {
    this.highlight(this.appHighlight);
  }
  
  @HostListener('mouseleave') onMouseLeave() {
    this.highlight(this.defaultColor);
  }
  
  private highlight(color: string) {
    this.el.nativeElement.style.backgroundColor = color;
  }
}

// Usage
@Component({
  template: `
    <p appHighlight>Hover me (default yellow)</p>
    <p [appHighlight]="'lightblue'" defaultColor="white">
      Hover me (custom colors)
    </p>
  `,
  imports: [HighlightDirective]
})
export class AppComponent { }
```

### Custom Structural Directive

```typescript
import { Directive, Input, TemplateRef, ViewContainerRef } from '@angular/core';

@Directive({
  selector: '[appUnless]',
  standalone: true
})
export class UnlessDirective {
  private hasView = false;
  
  constructor(
    private templateRef: TemplateRef<any>,
    private viewContainer: ViewContainerRef
  ) {}
  
  @Input() set appUnless(condition: boolean) {
    if (!condition && !this.hasView) {
      this.viewContainer.createEmbeddedView(this.templateRef);
      this.hasView = true;
    } else if (condition && this.hasView) {
      this.viewContainer.clear();
      this.hasView = false;
    }
  }
}

// Usage: <p *appUnless="condition">Show when false</p>
```

---

## 6. Pipes {#pipes}

Pipes transform data in templates. Syntax: `{{ value | pipeName:param1:param2 }}`

### Built-in Pipes

```typescript
@Component({
  template: `
    <!-- Date Pipe -->
    <p>{{ today | date }}</p>
    <p>{{ today | date:'short' }}</p>
    <p>{{ today | date:'dd/MM/yyyy' }}</p>
    <p>{{ today | date:'fullDate' }}</p>
    
    <!-- Currency Pipe -->
    <p>{{ price | currency }}</p>
    <p>{{ price | currency:'EUR' }}</p>
    <p>{{ price | currency:'INR':'symbol':'1.0-0' }}</p>
    
    <!-- Decimal/Number Pipe -->
    <p>{{ pi | number }}</p>
    <p>{{ pi | number:'1.2-4' }}</p> <!-- min.minDecimal-maxDecimal -->
    
    <!-- Percent Pipe -->
    <p>{{ ratio | percent }}</p>
    <p>{{ ratio | percent:'1.2-2' }}</p>
    
    <!-- Uppercase/Lowercase Pipe -->
    <p>{{ name | uppercase }}</p>
    <p>{{ name | lowercase }}</p>
    <p>{{ name | titlecase }}</p>
    
    <!-- JSON Pipe (debugging) -->
    <pre>{{ user | json }}</pre>
    
    <!-- Slice Pipe -->
    <p>{{ text | slice:0:10 }}</p>
    <li *ngFor="let item of items | slice:0:3">{{ item }}</li>
    
    <!-- Async Pipe (for Observables/Promises) -->
    <p>{{ user$ | async | json }}</p>
    <div *ngIf="data$ | async as data">{{ data }}</div>
    
    <!-- Chaining Pipes -->
    <p>{{ today | date:'fullDate' | uppercase }}</p>
    <p>{{ price | currency:'USD' | lowercase }}</p>
  `
})
export class PipesComponent {
  today = new Date();
  price = 1234.56;
  pi = 3.14159265359;
  ratio = 0.75;
  name = 'john doe';
  user = { name: 'John', age: 25 };
  text = 'Hello World, this is a long text';
  items = ['A', 'B', 'C', 'D', 'E'];
  user$ = of({ name: 'John' });
  data$ = of('Async Data');
}
```

### Custom Pipe

**Pure Pipe (default - better performance):**
```typescript
import { Pipe, PipeTransform } from '@angular/core';

@Pipe({
  name: 'exponential',
  standalone: true
})
export class ExponentialPipe implements PipeTransform {
  transform(value: number, exponent: number = 1): number {
    return Math.pow(value, exponent);
  }
}

// Usage: {{ 2 | exponential:3 }} → 8
```

**Impure Pipe (recalculates on every change detection):**
```typescript
@Pipe({
  name: 'filter',
  standalone: true,
  pure: false  // Impure pipe
})
export class FilterPipe implements PipeTransform {
  transform(items: any[], searchText: string): any[] {
    if (!items || !searchText) {
      return items;
    }
    return items.filter(item => 
      item.name.toLowerCase().includes(searchText.toLowerCase())
    );
  }
}

// Usage: <li *ngFor="let item of items | filter:searchText">
```

**Real-world Custom Pipes:**

```typescript
// Truncate text
@Pipe({ name: 'truncate', standalone: true })
export class TruncatePipe implements PipeTransform {
  transform(value: string, limit: number = 50, trail: string = '...'): string {
    return value.length > limit ? value.substring(0, limit) + trail : value;
  }
}

// Time ago
@Pipe({ name: 'timeAgo', standalone: true })
export class TimeAgoPipe implements PipeTransform {
  transform(value: Date): string {
    const seconds = Math.floor((Date.now() - value.getTime()) / 1000);
    
    const intervals: { [key: string]: number } = {
      year: 31536000,
      month: 2592000,
      week: 604800,
      day: 86400,
      hour: 3600,
      minute: 60,
      second: 1
    };
    
    for (const [name, secondsInInterval] of Object.entries(intervals)) {
      const interval = Math.floor(seconds / secondsInInterval);
      if (interval >= 1) {
        return `${interval} ${name}${interval > 1 ? 's' : ''} ago`;
      }
    }
    return 'just now';
  }
}

// Safe HTML (bypass sanitization)
import { DomSanitizer, SafeHtml } from '@angular/platform-browser';

@Pipe({ name: 'safeHtml', standalone: true })
export class SafeHtmlPipe implements PipeTransform {
  constructor(private sanitizer: DomSanitizer) {}
  
  transform(value: string): SafeHtml {
    return this.sanitizer.bypassSecurityTrustHtml(value);
  }
}

// Usage: <div [innerHTML]="htmlContent | safeHtml"></div>
```

---

## 7. Services & Dependency Injection {#services}

### What is a Service?

A service is a class with a specific purpose - typically:
- Data fetching/storage
- Business logic
- Logging
- State management
- Utilities

### Creating a Service

```bash
ng generate service services/user
# or
ng g s services/user
```

**Basic Service:**
```typescript
import { Injectable } from '@angular/core';

@Injectable({
  providedIn: 'root'  // Singleton service (app-wide)
})
export class UserService {
  private users: string[] = ['Alice', 'Bob', 'Charlie'];
  
  getUsers(): string[] {
    return this.users;
  }
  
  addUser(name: string): void {
    this.users.push(name);
  }
  
  deleteUser(name: string): void {
    const index = this.users.indexOf(name);
    if (index > -1) {
      this.users.splice(index, 1);
    }
  }
}
```

### Dependency Injection

**Injecting Services:**
```typescript
import { Component, OnInit } from '@angular/core';
import { UserService } from './services/user.service';

@Component({
  selector: 'app-users',
  template: `
    <ul>
      <li *ngFor="let user of users">{{ user }}</li>
    </ul>
  `
})
export class UsersComponent implements OnInit {
  users: string[] = [];
  
  // Constructor injection (preferred)
  constructor(private userService: UserService) {}
  
  ngOnInit() {
    this.users = this.userService.getUsers();
  }
}
```

**Inject function (Angular 14+):**
```typescript
import { Component, OnInit, inject } from '@angular/core';
import { UserService } from './services/user.service';

@Component({
  selector: 'app-users',
  template: `...`
})
export class UsersComponent implements OnInit {
  // Using inject function
  private userService = inject(UserService);
  users: string[] = [];
  
  ngOnInit() {
    this.users = this.userService.getUsers();
  }
}
```

### Provider Scopes

```typescript
// 1. Root level (singleton across entire app)
@Injectable({
  providedIn: 'root'
})
export class GlobalService { }

// 2. Component level (new instance per component)
@Component({
  selector: 'app-example',
  providers: [LocalService],  // Component-specific instance
  template: `...`
})
export class ExampleComponent { }

// 3. Module level (legacy, for NgModule)
@NgModule({
  providers: [ModuleScopedService]
})
export class FeatureModule { }

// 4. Platform level (shared across multiple Angular apps)
@Injectable({
  providedIn: 'platform'
})
export class PlatformService { }
```

### Service with HTTP

```typescript
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable, catchError, of } from 'rxjs';

export interface User {
  id: number;
  name: string;
  email: string;
}

@Injectable({
  providedIn: 'root'
})
export class ApiService {
  private apiUrl = 'https://api.example.com';
  
  constructor(private http: HttpClient) {}
  
  getUsers(): Observable<User[]> {
    return this.http.get<User[]>(`${this.apiUrl}/users`)
      .pipe(
        catchError(error => {
          console.error('Error fetching users:', error);
          return of([]);
        })
      );
  }
  
  getUser(id: number): Observable<User> {
    return this.http.get<User>(`${this.apiUrl}/users/${id}`);
  }
  
  createUser(user: User): Observable<User> {
    return this.http.post<User>(`${this.apiUrl}/users`, user);
  }
  
  updateUser(id: number, user: User): Observable<User> {
    return this.http.put<User>(`${this.apiUrl}/users/${id}`, user);
  }
  
  deleteUser(id: number): Observable<void> {
    return this.http.delete<void>(`${this.apiUrl}/users/${id}`);
  }
}
```

### Value Providers & InjectionToken

```typescript
import { InjectionToken } from '@angular/core';

// Create injection token
export const API_URL = new InjectionToken<string>('api.url');

// Provide value
bootstrapApplication(AppComponent, {
  providers: [
    { provide: API_URL, useValue: 'https://api.example.com' }
  ]
});

// Inject token
@Injectable({ providedIn: 'root' })
export class ApiService {
  constructor(@Inject(API_URL) private apiUrl: string) {
    console.log('API URL:', this.apiUrl);
  }
}
```

### Factory Providers

```typescript
// Factory function
export function loggerFactory(isDevelopment: boolean) {
  return isDevelopment ? new ConsoleLogger() : new RemoteLogger();
}

// Provider configuration
bootstrapApplication(AppComponent, {
  providers: [
    {
      provide: Logger,
      useFactory: loggerFactory,
      deps: [IS_DEVELOPMENT]
    },
    { provide: IS_DEVELOPMENT, useValue: !environment.production }
  ]
});
```

---

## 8. Routing {#routing}

### Basic Routing Setup

**Standalone App Routing:**
```typescript
// app.routes.ts
import { Routes } from '@angular/router';
import { HomeComponent } from './pages/home.component';
import { AboutComponent } from './pages/about.component';
import { NotFoundComponent } from './pages/not-found.component';

export const routes: Routes = [
  { path: '', component: HomeComponent },
  { path: 'about', component: AboutComponent },
  { path: '**', component: NotFoundComponent }  // Wildcard route (404)
];

// main.ts
import { bootstrapApplication } from '@angular/platform-browser';
import { provideRouter } from '@angular/router';
import { AppComponent } from './app/app.component';
import { routes } from './app/app.routes';

bootstrapApplication(AppComponent, {
  providers: [
    provideRouter(routes)
  ]
});

// app.component.ts
import { Component } from '@angular/core';
import { RouterOutlet, RouterLink, RouterLinkActive } from '@angular/router';

@Component({
  selector: 'app-root',
  standalone: true,
  imports: [RouterOutlet, RouterLink, RouterLinkActive],
  template: `
    <nav>
      <a routerLink="/" routerLinkActive="active" [routerLinkActiveOptions]="{exact: true}">Home</a>
      <a routerLink="/about" routerLinkActive="active">About</a>
    </nav>
    <router-outlet></router-outlet>
  `
})
export class AppComponent { }
```

### Route Parameters

```typescript
// routes
export const routes: Routes = [
  { path: 'user/:id', component: UserDetailComponent },
  { path: 'product/:id/:slug', component: ProductComponent }
];

// Component
import { Component, OnInit } from '@angular/core';
import { ActivatedRoute } from '@angular/router';

@Component({
  selector: 'app-user-detail',
  template: `<p>User ID: {{ userId }}</p>`
})
export class UserDetailComponent implements OnInit {
  userId: string = '';
  
  constructor(private route: ActivatedRoute) {}
  
  ngOnInit() {
    // Snapshot (one-time read)
    this.userId = this.route.snapshot.paramMap.get('id') || '';
    
    // Observable (for param changes)
    this.route.paramMap.subscribe(params => {
      this.userId = params.get('id') || '';
    });
  }
}
```

### Query Parameters

```typescript
// Navigate with query params
import { Router } from '@angular/router';

constructor(private router: Router) {}

navigateWithQuery() {
  this.router.navigate(['/search'], {
    queryParams: { q: 'angular', page: 1 }
  });
  // URL: /search?q=angular&page=1
}

// Read query params
import { ActivatedRoute } from '@angular/router';

constructor(private route: ActivatedRoute) {}

ngOnInit() {
  this.route.queryParamMap.subscribe(params => {
    const query = params.get('q');
    const page = params.get('page');
    console.log('Query:', query, 'Page:', page);
  });
}
```

### Programmatic Navigation

```typescript
import { Router } from '@angular/router';

constructor(private router: Router) {}

// Navigate to route
goToAbout() {
  this.router.navigate(['/about']);
}

// Navigate with params
goToUser(id: number) {
  this.router.navigate(['/user', id]);
}

// Navigate with query params
search(term: string) {
  this.router.navigate(['/search'], {
    queryParams: { q: term }
  });
}

// Navigate relative to current route
import { ActivatedRoute } from '@angular/router';

constructor(
  private router: Router,
  private route: ActivatedRoute
) {}

goToSibling() {
  this.router.navigate(['../sibling'], { relativeTo: this.route });
}
```

### Child Routes

```typescript
export const routes: Routes = [
  {
    path: 'dashboard',
    component: DashboardComponent,
    children: [
      { path: '', redirectTo: 'overview', pathMatch: 'full' },
      { path: 'overview', component: OverviewComponent },
      { path: 'stats', component: StatsComponent },
      { path: 'settings', component: SettingsComponent }
    ]
  }
];

// DashboardComponent template
@Component({
  template: `
    <h1>Dashboard</h1>
    <nav>
      <a routerLink="overview">Overview</a>
      <a routerLink="stats">Stats</a>
      <a routerLink="settings">Settings</a>
    </nav>
    <router-outlet></router-outlet>  <!-- Child routes render here -->
  `
})
export class DashboardComponent { }
```

This completes Part 1 covering Angular fundamentals. The file continues with advanced topics in subsequent parts.
# Angular 21 Complete Interview Guide - Part 2: Advanced Topics

## Table of Contents
1. [Forms](#forms)
2. [HTTP Client](#http)
3. [RxJS & Observables](#rxjs)
4. [State Management](#state-management)
5. [Change Detection](#change-detection)
6. [Performance Optimization](#performance)

---

## 1. Forms {#forms}

### Template-Driven Forms

**Setup:**
```typescript
import { FormsModule } from '@angular/forms';

@Component({
  standalone: true,
  imports: [FormsModule],
  template: `...`
})
```

**Basic Template-Driven Form:**
```typescript
@Component({
  template: `
    <form #userForm="ngForm" (ngSubmit)="onSubmit(userForm)">
      <!-- Text Input -->
      <input 
        type="text" 
        name="username" 
        [(ngModel)]="user.username"
        #username="ngModel"
        required
        minlength="3">
      <div *ngIf="username.invalid && username.touched">
        <p *ngIf="username.errors?.['required']">Username is required</p>
        <p *ngIf="username.errors?.['minlength']">Minimum 3 characters</p>
      </div>
      
      <!-- Email -->
      <input 
        type="email" 
        name="email" 
        [(ngModel)]="user.email"
        #email="ngModel"
        required
        email>
      <div *ngIf="email.invalid && email.touched">
        <p *ngIf="email.errors?.['email']">Invalid email</p>
      </div>
      
      <!-- Select -->
      <select name="country" [(ngModel)]="user.country" required>
        <option value="">Select Country</option>
        <option value="us">USA</option>
        <option value="uk">UK</option>
        <option value="in">India</option>
      </select>
      
      <!-- Checkbox -->
      <input type="checkbox" name="agree" [(ngModel)]="user.agree" required>
      
      <!-- Submit Button -->
      <button type="submit" [disabled]="userForm.invalid">Submit</button>
      
      <!-- Form State -->
      <pre>{{ userForm.value | json }}</pre>
      <p>Valid: {{ userForm.valid }}</p>
      <p>Touched: {{ userForm.touched }}</p>
      <p>Dirty: {{ userForm.dirty }}</p>
    </form>
  `
})
export class TemplateDrivenFormComponent {
  user = {
    username: '',
    email: '',
    country: '',
    agree: false
  };
  
  onSubmit(form: NgForm) {
    if (form.valid) {
      console.log('Form submitted:', this.user);
      form.reset();
    }
  }
}
```

### Reactive Forms

**Setup:**
```typescript
import { ReactiveFormsModule } from '@angular/forms';

@Component({
  standalone: true,
  imports: [ReactiveFormsModule],
  template: `...`
})
```

**Basic Reactive Form:**
```typescript
import { Component, OnInit } from '@angular/core';
import { FormBuilder, FormGroup, FormControl, Validators } from '@angular/forms';

@Component({
  template: `
    <form [formGroup]="userForm" (ngSubmit)="onSubmit()">
      <!-- Text Input -->
      <input formControlName="username" type="text">
      <div *ngIf="username.invalid && username.touched">
        <p *ngIf="username.errors?.['required']">Username is required</p>
        <p *ngIf="username.errors?.['minlength']">
          Min {{ username.errors?.['minlength'].requiredLength }} chars
        </p>
      </div>
      
      <!-- Email -->
      <input formControlName="email" type="email">
      <div *ngIf="email.invalid && email.touched">
        <p *ngIf="email.errors?.['required']">Email is required</p>
        <p *ngIf="email.errors?.['email']">Invalid email</p>
      </div>
      
      <!-- Password with confirmation -->
      <input formControlName="password" type="password">
      <input formControlName="confirmPassword" type="password">
      <div *ngIf="userForm.errors?.['passwordMismatch'] && confirmPassword.touched">
        <p>Passwords do not match</p>
      </div>
      
      <button type="submit" [disabled]="userForm.invalid">Submit</button>
    </form>
  `
})
export class ReactiveFormComponent implements OnInit {
  userForm!: FormGroup;
  
  constructor(private fb: FormBuilder) {}
  
  ngOnInit() {
    // Method 1: Using FormBuilder (recommended)
    this.userForm = this.fb.group({
      username: ['', [Validators.required, Validators.minLength(3)]],
      email: ['', [Validators.required, Validators.email]],
      password: ['', [Validators.required, Validators.minLength(8)]],
      confirmPassword: ['', Validators.required]
    }, {
      validators: this.passwordMatchValidator
    });
    
    // Method 2: Manual FormGroup
    this.userForm = new FormGroup({
      username: new FormControl('', [Validators.required]),
      email: new FormControl('', [Validators.required, Validators.email])
    });
  }
  
  // Getter for easy access
  get username() {
    return this.userForm.get('username')!;
  }
  
  get email() {
    return this.userForm.get('email')!;
  }
  
  get confirmPassword() {
    return this.userForm.get('confirmPassword')!;
  }
  
  // Custom validator
  passwordMatchValidator(group: FormGroup) {
    const password = group.get('password')?.value;
    const confirm = group.get('confirmPassword')?.value;
    return password === confirm ? null : { passwordMismatch: true };
  }
  
  onSubmit() {
    if (this.userForm.valid) {
      console.log('Form data:', this.userForm.value);
      this.userForm.reset();
    }
  }
}
```

### FormArray - Dynamic Form Fields

```typescript
import { FormArray, FormControl } from '@angular/forms';

@Component({
  template: `
    <form [formGroup]="form" (ngSubmit)="onSubmit()">
      <div formArrayName="skills">
        <div *ngFor="let skill of skills.controls; let i = index">
          <input [formControlName]="i" placeholder="Skill {{ i + 1 }}">
          <button type="button" (click)="removeSkill(i)">Remove</button>
        </div>
      </div>
      <button type="button" (click)="addSkill()">Add Skill</button>
      <button type="submit">Submit</button>
    </form>
  `
})
export class FormArrayComponent implements OnInit {
  form!: FormGroup;
  
  constructor(private fb: FormBuilder) {}
  
  ngOnInit() {
    this.form = this.fb.group({
      skills: this.fb.array([
        this.fb.control('', Validators.required)
      ])
    });
  }
  
  get skills() {
    return this.form.get('skills') as FormArray;
  }
  
  addSkill() {
    this.skills.push(this.fb.control('', Validators.required));
  }
  
  removeSkill(index: number) {
    this.skills.removeAt(index);
  }
  
  onSubmit() {
    console.log('Skills:', this.skills.value);
  }
}
```

### Nested FormGroups

```typescript
@Component({
  template: `
    <form [formGroup]="userForm">
      <input formControlName="name">
      
      <div formGroupName="address">
        <input formControlName="street">
        <input formControlName="city">
        <input formControlName="zip">
      </div>
      
      <div formGroupName="contact">
        <input formControlName="email">
        <input formControlName="phone">
      </div>
    </form>
  `
})
export class NestedFormComponent implements OnInit {
  userForm!: FormGroup;
  
  ngOnInit() {
    this.userForm = this.fb.group({
      name: [''],
      address: this.fb.group({
        street: [''],
        city: [''],
        zip: ['']
      }),
      contact: this.fb.group({
        email: ['', Validators.email],
        phone: ['']
      })
    });
  }
  
  // Access nested controls
  get addressCity() {
    return this.userForm.get('address.city');
  }
}
```

### Custom Validators

**Synchronous Validator:**
```typescript
import { AbstractControl, ValidationErrors, ValidatorFn } from '@angular/forms';

// Validator function
export function forbiddenNameValidator(forbidden: RegExp): ValidatorFn {
  return (control: AbstractControl): ValidationErrors | null => {
    const isForbidden = forbidden.test(control.value);
    return isForbidden ? { forbiddenName: { value: control.value } } : null;
  };
}

// Usage
this.userForm = this.fb.group({
  username: ['', [
    Validators.required,
    forbiddenNameValidator(/admin/i)
  ]]
});
```

**Asynchronous Validator:**
```typescript
import { AsyncValidatorFn } from '@angular/forms';
import { map, catchError } from 'rxjs/operators';
import { of } from 'rxjs';

export class CustomValidators {
  static usernameExists(userService: UserService): AsyncValidatorFn {
    return (control: AbstractControl): Observable<ValidationErrors | null> => {
      if (!control.value) {
        return of(null);
      }
      
      return userService.checkUsername(control.value).pipe(
        map(exists => exists ? { usernameExists: true } : null),
        catchError(() => of(null))
      );
    };
  }
}

// Usage
this.userForm = this.fb.group({
  username: ['', 
    [Validators.required],
    [CustomValidators.usernameExists(this.userService)]
  ]
});
```

### Form Value Changes & Status

```typescript
ngOnInit() {
  this.userForm = this.fb.group({
    username: [''],
    email: ['']
  });
  
  // Subscribe to value changes
  this.userForm.valueChanges.subscribe(value => {
    console.log('Form value changed:', value);
  });
  
  // Subscribe to specific control
  this.userForm.get('username')?.valueChanges.subscribe(value => {
    console.log('Username changed:', value);
  });
  
  // Subscribe to status changes
  this.userForm.statusChanges.subscribe(status => {
    console.log('Form status:', status); // VALID, INVALID, PENDING, DISABLED
  });
  
  // Debounce value changes
  this.userForm.get('username')?.valueChanges.pipe(
    debounceTime(300),
    distinctUntilChanged()
  ).subscribe(value => {
    console.log('Debounced username:', value);
  });
}
```

### Form Methods

```typescript
// Set value (all controls required)
this.userForm.setValue({
  username: 'john',
  email: 'john@example.com'
});

// Patch value (partial update)
this.userForm.patchValue({
  username: 'john'
});

// Reset form
this.userForm.reset();
this.userForm.reset({ username: 'default' });

// Disable/Enable
this.userForm.disable();
this.userForm.enable();
this.userForm.get('email')?.disable();

// Mark as touched
this.userForm.markAsTouched();
this.userForm.markAllAsTouched();

// Mark as dirty
this.userForm.markAsDirty();

// Get raw value (includes disabled controls)
const rawValue = this.userForm.getRawValue();
```

---

## 2. HTTP Client {#http}

### Setup

```typescript
// main.ts
import { provideHttpClient, withInterceptors } from '@angular/common/http';

bootstrapApplication(AppComponent, {
  providers: [
    provideHttpClient(
      withInterceptors([authInterceptor, loggingInterceptor])
    )
  ]
});
```

### Basic HTTP Operations

```typescript
import { HttpClient, HttpHeaders, HttpParams } from '@angular/common/http';
import { Observable } from 'rxjs';

@Injectable({ providedIn: 'root' })
export class ApiService {
  private apiUrl = 'https://api.example.com';
  
  constructor(private http: HttpClient) {}
  
  // GET request
  getUsers(): Observable<User[]> {
    return this.http.get<User[]>(`${this.apiUrl}/users`);
  }
  
  // GET with params
  getUsersPaginated(page: number, limit: number): Observable<User[]> {
    const params = new HttpParams()
      .set('page', page.toString())
      .set('limit', limit.toString());
    
    return this.http.get<User[]>(`${this.apiUrl}/users`, { params });
  }
  
  // GET with headers
  getProtectedData(): Observable<any> {
    const headers = new HttpHeaders()
      .set('Authorization', `Bearer ${this.getToken()}`)
      .set('Content-Type', 'application/json');
    
    return this.http.get(`${this.apiUrl}/protected`, { headers });
  }
  
  // POST request
  createUser(user: User): Observable<User> {
    return this.http.post<User>(`${this.apiUrl}/users`, user);
  }
  
  // PUT request
  updateUser(id: number, user: User): Observable<User> {
    return this.http.put<User>(`${this.apiUrl}/users/${id}`, user);
  }
  
  // PATCH request
  patchUser(id: number, updates: Partial<User>): Observable<User> {
    return this.http.patch<User>(`${this.apiUrl}/users/${id}`, updates);
  }
  
  // DELETE request
  deleteUser(id: number): Observable<void> {
    return this.http.delete<void>(`${this.apiUrl}/users/${id}`);
  }
  
  private getToken(): string {
    return localStorage.getItem('token') || '';
  }
}
```

### Response Types

```typescript
// Get full response
getUsers(): Observable<HttpResponse<User[]>> {
  return this.http.get<User[]>(`${this.apiUrl}/users`, {
    observe: 'response'
  });
}

// Usage
this.apiService.getUsers().subscribe(response => {
  console.log('Status:', response.status);
  console.log('Headers:', response.headers);
  console.log('Body:', response.body);
});

// Get events (for progress tracking)
uploadFile(file: File): Observable<HttpEvent<any>> {
  const formData = new FormData();
  formData.append('file', file);
  
  return this.http.post(`${this.apiUrl}/upload`, formData, {
    reportProgress: true,
    observe: 'events'
  });
}

// Usage
this.apiService.uploadFile(file).subscribe(event => {
  if (event.type === HttpEventType.UploadProgress) {
    const progress = Math.round(100 * event.loaded / (event.total || 1));
    console.log(`Upload progress: ${progress}%`);
  } else if (event.type === HttpEventType.Response) {
    console.log('Upload complete:', event.body);
  }
});
```

### Error Handling

```typescript
import { catchError, retry, timeout } from 'rxjs/operators';
import { throwError } from 'rxjs';
import { HttpErrorResponse } from '@angular/common/http';

getUsers(): Observable<User[]> {
  return this.http.get<User[]>(`${this.apiUrl}/users`).pipe(
    timeout(5000),  // 5 second timeout
    retry(2),       // Retry 2 times on failure
    catchError(this.handleError)
  );
}

private handleError(error: HttpErrorResponse): Observable<never> {
  let errorMessage = 'An error occurred';
  
  if (error.error instanceof ErrorEvent) {
    // Client-side error
    errorMessage = `Error: ${error.error.message}`;
  } else {
    // Server-side error
    errorMessage = `Error Code: ${error.status}\nMessage: ${error.message}`;
    
    switch (error.status) {
      case 400:
        errorMessage = 'Bad Request';
        break;
      case 401:
        errorMessage = 'Unauthorized';
        break;
      case 403:
        errorMessage = 'Forbidden';
        break;
      case 404:
        errorMessage = 'Not Found';
        break;
      case 500:
        errorMessage = 'Internal Server Error';
        break;
    }
  }
  
  console.error(errorMessage);
  return throwError(() => new Error(errorMessage));
}
```

### Interceptors

**Functional Interceptor (Angular 15+):**
```typescript
import { HttpInterceptorFn } from '@angular/common/http';

// Auth Interceptor
export const authInterceptor: HttpInterceptorFn = (req, next) => {
  const token = localStorage.getItem('token');
  
  if (token) {
    const cloned = req.clone({
      headers: req.headers.set('Authorization', `Bearer ${token}`)
    });
    return next(cloned);
  }
  
  return next(req);
};

// Logging Interceptor
export const loggingInterceptor: HttpInterceptorFn = (req, next) => {
  console.log('HTTP Request:', req.method, req.url);
  const startTime = Date.now();
  
  return next(req).pipe(
    tap(event => {
      if (event.type === HttpEventType.Response) {
        const elapsed = Date.now() - startTime;
        console.log(`Response received in ${elapsed}ms`, event.status);
      }
    })
  );
};

// Error Interceptor
export const errorInterceptor: HttpInterceptorFn = (req, next) => {
  return next(req).pipe(
    catchError((error: HttpErrorResponse) => {
      if (error.status === 401) {
        // Redirect to login
        console.log('Unauthorized - redirecting to login');
      }
      return throwError(() => error);
    })
  );
};

// Caching Interceptor
const cache = new Map<string, HttpResponse<any>>();

export const cachingInterceptor: HttpInterceptorFn = (req, next) => {
  if (req.method !== 'GET') {
    return next(req);
  }
  
  const cachedResponse = cache.get(req.url);
  if (cachedResponse) {
    return of(cachedResponse);
  }
  
  return next(req).pipe(
    tap(event => {
      if (event instanceof HttpResponse) {
        cache.set(req.url, event);
      }
    })
  );
};
```

**Class-based Interceptor (Legacy):**
```typescript
import { Injectable } from '@angular/core';
import { HttpInterceptor, HttpRequest, HttpHandler, HttpEvent } from '@angular/common/http';
import { Observable } from 'rxjs';

@Injectable()
export class AuthInterceptor implements HttpInterceptor {
  intercept(req: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {
    const token = localStorage.getItem('token');
    
    if (token) {
      const cloned = req.clone({
        setHeaders: {
          Authorization: `Bearer ${token}`
        }
      });
      return next.handle(cloned);
    }
    
    return next.handle(req);
  }
}

// Provide in module
providers: [
  { provide: HTTP_INTERCEPTORS, useClass: AuthInterceptor, multi: true }
]
```

---

## 3. RxJS & Observables {#rxjs}

### Observable Basics

```typescript
import { Observable, Observer } from 'rxjs';

// Create custom observable
const observable = new Observable<number>(observer => {
  observer.next(1);
  observer.next(2);
  observer.next(3);
  observer.complete();
  
  // Cleanup
  return () => {
    console.log('Observable unsubscribed');
  };
});

// Subscribe
const subscription = observable.subscribe({
  next: (value) => console.log('Value:', value),
  error: (err) => console.error('Error:', err),
  complete: () => console.log('Complete')
});

// Unsubscribe
subscription.unsubscribe();
```

### Creation Operators

```typescript
import { of, from, interval, timer, fromEvent, ajax } from 'rxjs';

// of - emit values in sequence
of(1, 2, 3).subscribe(v => console.log(v));

// from - convert array/promise to observable
from([1, 2, 3]).subscribe(v => console.log(v));
from(fetch('/api/data')).subscribe(v => console.log(v));

// interval - emit every X milliseconds
interval(1000).subscribe(v => console.log('Tick:', v));

// timer - emit after delay, then interval
timer(2000, 1000).subscribe(v => console.log(v));

// fromEvent - convert DOM event to observable
const clicks$ = fromEvent(document, 'click');
clicks$.subscribe(event => console.log('Click:', event));

// ajax - HTTP request
ajax.getJSON('https://api.example.com/users').subscribe(data => {
  console.log('Data:', data);
});
```

### Operators

**Transformation Operators:**
```typescript
import { map, pluck, mapTo, scan } from 'rxjs/operators';

// map - transform each value
of(1, 2, 3).pipe(
  map(x => x * 10)
).subscribe(v => console.log(v)); // 10, 20, 30

// pluck - extract property (deprecated, use map instead)
of({ name: 'John' }, { name: 'Jane' }).pipe(
  map(user => user.name)
).subscribe(name => console.log(name));

// mapTo - map to constant value
fromEvent(button, 'click').pipe(
  mapTo(1)
).subscribe(v => console.log(v)); // Always emits 1

// scan - accumulator (like reduce)
of(1, 2, 3, 4).pipe(
  scan((acc, curr) => acc + curr, 0)
).subscribe(v => console.log(v)); // 1, 3, 6, 10
```

**Filtering Operators:**
```typescript
import { filter, take, takeUntil, takeWhile, skip, distinctUntilChanged, debounceTime } from 'rxjs/operators';

// filter - emit values that pass condition
of(1, 2, 3, 4, 5).pipe(
  filter(x => x % 2 === 0)
).subscribe(v => console.log(v)); // 2, 4

// take - take first N values
interval(1000).pipe(
  take(3)
).subscribe(v => console.log(v)); // 0, 1, 2 then complete

// takeUntil - emit until notifier emits
const stop$ = new Subject();
interval(1000).pipe(
  takeUntil(stop$)
).subscribe(v => console.log(v));
setTimeout(() => stop$.next(), 5000); // Stop after 5 seconds

// takeWhile - emit while condition is true
interval(1000).pipe(
  takeWhile(v => v < 5)
).subscribe(v => console.log(v)); // 0, 1, 2, 3, 4

// skip - skip first N values
of(1, 2, 3, 4, 5).pipe(
  skip(2)
).subscribe(v => console.log(v)); // 3, 4, 5

// distinctUntilChanged - skip consecutive duplicates
of(1, 1, 2, 2, 3, 1).pipe(
  distinctUntilChanged()
).subscribe(v => console.log(v)); // 1, 2, 3, 1

// debounceTime - emit after silence period
fromEvent(input, 'input').pipe(
  debounceTime(300)
).subscribe(event => console.log('Search:', event));
```

**Combination Operators:**
```typescript
import { merge, concat, combineLatest, forkJoin, zip, switchMap, mergeMap, concatMap } from 'rxjs';

// merge - emit from all observables as they occur
const obs1$ = interval(1000).pipe(map(v => `A${v}`));
const obs2$ = interval(1500).pipe(map(v => `B${v}`));
merge(obs1$, obs2$).subscribe(v => console.log(v));

// concat - emit from observables in sequence
concat(
  of(1, 2, 3),
  of(4, 5, 6)
).subscribe(v => console.log(v)); // 1, 2, 3, 4, 5, 6

// combineLatest - emit when any observable emits (latest from each)
combineLatest([obs1$, obs2$]).subscribe(([a, b]) => {
  console.log(`A: ${a}, B: ${b}`);
});

// forkJoin - wait for all to complete, emit last values
forkJoin([
  ajax.getJSON('/api/users'),
  ajax.getJSON('/api/posts')
]).subscribe(([users, posts]) => {
  console.log('Users:', users, 'Posts:', posts);
});

// zip - emit when all have emitted (pairwise)
zip(
  of(1, 2, 3),
  of('a', 'b', 'c')
).subscribe(([num, letter]) => {
  console.log(num, letter); // [1, 'a'], [2, 'b'], [3, 'c']
});

// switchMap - cancel previous inner observable
fromEvent(input, 'input').pipe(
  debounceTime(300),
  switchMap(event => ajax.getJSON(`/api/search?q=${event.target.value}`))
).subscribe(results => console.log(results));

// mergeMap - run all inner observables concurrently
of(1, 2, 3).pipe(
  mergeMap(id => ajax.getJSON(`/api/user/${id}`))
).subscribe(user => console.log(user));

// concatMap - run inner observables sequentially
of(1, 2, 3).pipe(
  concatMap(id => ajax.getJSON(`/api/user/${id}`))
).subscribe(user => console.log(user));
```

### Subjects

```typescript
import { Subject, BehaviorSubject, ReplaySubject, AsyncSubject } from 'rxjs';

// Subject - multicast, no initial value
const subject = new Subject<number>();
subject.subscribe(v => console.log('A:', v));
subject.subscribe(v => console.log('B:', v));
subject.next(1); // Both subscribers receive 1
subject.next(2); // Both subscribers receive 2

// BehaviorSubject - stores current value
const behaviorSubject = new BehaviorSubject<number>(0);
behaviorSubject.subscribe(v => console.log('A:', v)); // Immediately logs 0
behaviorSubject.next(1);
behaviorSubject.subscribe(v => console.log('B:', v)); // Immediately logs 1
console.log('Current value:', behaviorSubject.value);

// ReplaySubject - replays N previous values
const replaySubject = new ReplaySubject<number>(2); // Replay last 2
replaySubject.next(1);
replaySubject.next(2);
replaySubject.next(3);
replaySubject.subscribe(v => console.log('A:', v)); // Logs 2, 3

// AsyncSubject - emits last value on complete
const asyncSubject = new AsyncSubject<number>();
asyncSubject.subscribe(v => console.log('A:', v));
asyncSubject.next(1);
asyncSubject.next(2);
asyncSubject.next(3);
asyncSubject.complete(); // Now logs 3
```

### Common Patterns in Angular

**Unsubscribe Strategies:**
```typescript
import { Component, OnDestroy } from '@angular/core';
import { Subject, takeUntil } from 'rxjs';

@Component({
  selector: 'app-example'
})
export class ExampleComponent implements OnDestroy {
  private destroy$ = new Subject<void>();
  
  ngOnInit() {
    // Pattern 1: takeUntil (recommended)
    this.dataService.getData().pipe(
      takeUntil(this.destroy$)
    ).subscribe(data => {
      console.log(data);
    });
    
    // Multiple subscriptions
    this.service1.data$.pipe(takeUntil(this.destroy$)).subscribe();
    this.service2.data$.pipe(takeUntil(this.destroy$)).subscribe();
  }
  
  ngOnDestroy() {
    this.destroy$.next();
    this.destroy$.complete();
  }
}

// Pattern 2: Manual unsubscribe
export class ManualUnsubscribeComponent implements OnDestroy {
  private subscription = new Subscription();
  
  ngOnInit() {
    this.subscription.add(
      this.dataService.getData().subscribe()
    );
    this.subscription.add(
      this.otherService.getData().subscribe()
    );
  }
  
  ngOnDestroy() {
    this.subscription.unsubscribe();
  }
}

// Pattern 3: Async pipe (auto-unsubscribe)
@Component({
  template: `<div *ngIf="data$ | async as data">{{ data }}</div>`
})
export class AsyncPipeComponent {
  data$ = this.dataService.getData();
}
```

**HTTP + RxJS:**
```typescript
@Injectable({ providedIn: 'root' })
export class UserService {
  private apiUrl = 'https://api.example.com';
  
  constructor(private http: HttpClient) {}
  
  // Search with debounce
  searchUsers(term: Observable<string>): Observable<User[]> {
    return term.pipe(
      debounceTime(300),
      distinctUntilChanged(),
      switchMap(searchTerm => 
        this.http.get<User[]>(`${this.apiUrl}/users?q=${searchTerm}`)
      ),
      catchError(() => of([]))
    );
  }
  
  // Load multiple resources
  loadDashboard(): Observable<Dashboard> {
    return forkJoin({
      users: this.http.get<User[]>(`${this.apiUrl}/users`),
      posts: this.http.get<Post[]>(`${this.apiUrl}/posts`),
      stats: this.http.get<Stats>(`${this.apiUrl}/stats`)
    }).pipe(
      map(({ users, posts, stats }) => ({
        users,
        posts,
        stats
      }))
    );
  }
  
  // Retry with exponential backoff
  getDataWithRetry(): Observable<any> {
    return this.http.get(`${this.apiUrl}/data`).pipe(
      retryWhen(errors =>
        errors.pipe(
          scan((retryCount, err) => {
            if (retryCount >= 3) {
              throw err;
            }
            return retryCount + 1;
          }, 0),
          delay(1000)
        )
      )
    );
  }
}
```

---

## 4. State Management {#state-management}

### Services as State Management

**Simple State Service:**
```typescript
import { Injectable } from '@angular/core';
import { BehaviorSubject, Observable } from 'rxjs';

interface AppState {
  user: User | null;
  theme: 'light' | 'dark';
  notifications: Notification[];
}

@Injectable({ providedIn: 'root' })
export class StateService {
  private state = new BehaviorSubject<AppState>({
    user: null,
    theme: 'light',
    notifications: []
  });
  
  public state$ = this.state.asObservable();
  
  // Selectors
  get currentState(): AppState {
    return this.state.value;
  }
  
  get user$(): Observable<User | null> {
    return this.state$.pipe(map(state => state.user));
  }
  
  get theme$(): Observable<string> {
    return this.state$.pipe(map(state => state.theme));
  }
  
  // Actions
  setUser(user: User | null) {
    this.updateState({ user });
  }
  
  setTheme(theme: 'light' | 'dark') {
    this.updateState({ theme });
  }
  
  addNotification(notification: Notification) {
    const notifications = [...this.currentState.notifications, notification];
    this.updateState({ notifications });
  }
  
  private updateState(partial: Partial<AppState>) {
    this.state.next({ ...this.currentState, ...partial });
  }
}
```

### Signals for State (Angular 16+)

```typescript
import { Injectable, signal, computed } from '@angular/core';

@Injectable({ providedIn: 'root' })
export class SignalStateService {
  // State signals
  private userSignal = signal<User | null>(null);
  private themeSignal = signal<'light' | 'dark'>('light');
  private notificationsSignal = signal<Notification[]>([]);
  
  // Public readonly signals
  readonly user = this.userSignal.asReadonly();
  readonly theme = this.themeSignal.asReadonly();
  readonly notifications = this.notificationsSignal.asReadonly();
  
  // Computed signals
  readonly unreadCount = computed(() => 
    this.notificationsSignal().filter(n => !n.read).length
  );
  
  readonly isLoggedIn = computed(() => 
    this.userSignal() !== null
  );
  
  // Actions
  setUser(user: User | null) {
    this.userSignal.set(user);
  }
  
  setTheme(theme: 'light' | 'dark') {
    this.themeSignal.set(theme);
  }
  
  addNotification(notification: Notification) {
    this.notificationsSignal.update(current => [...current, notification]);
  }
  
  markNotificationRead(id: string) {
    this.notificationsSignal.update(current =>
      current.map(n => n.id === id ? { ...n, read: true } : n)
    );
  }
  
  clearNotifications() {
    this.notificationsSignal.set([]);
  }
}

// Usage in component
@Component({
  template: `
    <p>User: {{ state.user()?.name }}</p>
    <p>Theme: {{ state.theme() }}</p>
    <p>Unread: {{ state.unreadCount() }}</p>
    <button (click)="toggleTheme()">Toggle Theme</button>
  `
})
export class AppComponent {
  constructor(public state: SignalStateService) {}
  
  toggleTheme() {
    const newTheme = this.state.theme() === 'light' ? 'dark' : 'light';
    this.state.setTheme(newTheme);
  }
}
```

### NgRx (Redux Pattern)

**Installation:**
```bash
npm install @ngrx/store @ngrx/effects @ngrx/entity @ngrx/store-devtools
```

**Actions:**
```typescript
// user.actions.ts
import { createAction, props } from '@ngrx/store';

export const loadUsers = createAction('[User] Load Users');
export const loadUsersSuccess = createAction(
  '[User] Load Users Success',
  props<{ users: User[] }>()
);
export const loadUsersFailure = createAction(
  '[User] Load Users Failure',
  props<{ error: string }>()
);
export const addUser = createAction(
  '[User] Add User',
  props<{ user: User }>()
);
```

**Reducer:**
```typescript
// user.reducer.ts
import { createReducer, on } from '@ngrx/store';
import * as UserActions from './user.actions';

export interface UserState {
  users: User[];
  loading: boolean;
  error: string | null;
}

export const initialState: UserState = {
  users: [],
  loading: false,
  error: null
};

export const userReducer = createReducer(
  initialState,
  on(UserActions.loadUsers, state => ({
    ...state,
    loading: true,
    error: null
  })),
  on(UserActions.loadUsersSuccess, (state, { users }) => ({
    ...state,
    users,
    loading: false
  })),
  on(UserActions.loadUsersFailure, (state, { error }) => ({
    ...state,
    error,
    loading: false
  })),
  on(UserActions.addUser, (state, { user }) => ({
    ...state,
    users: [...state.users, user]
  }))
);
```

**Selectors:**
```typescript
// user.selectors.ts
import { createFeatureSelector, createSelector } from '@ngrx/store';
import { UserState } from './user.reducer';

export const selectUserState = createFeatureSelector<UserState>('user');

export const selectUsers = createSelector(
  selectUserState,
  state => state.users
);

export const selectLoading = createSelector(
  selectUserState,
  state => state.loading
);

export const selectUserById = (id: number) => createSelector(
  selectUsers,
  users => users.find(u => u.id === id)
);
```

**Effects:**
```typescript
// user.effects.ts
import { Injectable } from '@angular/core';
import { Actions, createEffect, ofType } from '@ngrx/effects';
import { of } from 'rxjs';
import { map, catchError, switchMap } from 'rxjs/operators';
import * as UserActions from './user.actions';

@Injectable()
export class UserEffects {
  loadUsers$ = createEffect(() =>
    this.actions$.pipe(
      ofType(UserActions.loadUsers),
      switchMap(() =>
        this.userService.getUsers().pipe(
          map(users => UserActions.loadUsersSuccess({ users })),
          catchError(error => of(UserActions.loadUsersFailure({ 
            error: error.message 
          })))
        )
      )
    )
  );
  
  constructor(
    private actions$: Actions,
    private userService: UserService
  ) {}
}
```

**Store Setup:**
```typescript
// main.ts
import { provideStore } from '@ngrx/store';
import { provideEffects } from '@ngrx/effects';
import { userReducer } from './store/user.reducer';
import { UserEffects } from './store/user.effects';

bootstrapApplication(AppComponent, {
  providers: [
    provideStore({ user: userReducer }),
    provideEffects([UserEffects])
  ]
});
```

**Component Usage:**
```typescript
import { Store } from '@ngrx/store';
import * as UserActions from './store/user.actions';
import { selectUsers, selectLoading } from './store/user.selectors';

@Component({
  template: `
    <div *ngIf="loading$ | async">Loading...</div>
    <ul>
      <li *ngFor="let user of users$ | async">{{ user.name }}</li>
    </ul>
  `
})
export class UsersComponent implements OnInit {
  users$ = this.store.select(selectUsers);
  loading$ = this.store.select(selectLoading);
  
  constructor(private store: Store) {}
  
  ngOnInit() {
    this.store.dispatch(UserActions.loadUsers());
  }
  
  addUser(user: User) {
    this.store.dispatch(UserActions.addUser({ user }));
  }
}
```

---

## 5. Change Detection {#change-detection}

### How Change Detection Works

Angular runs change detection to sync component state with the view. It checks:
1. DOM events (click, input, etc.)
2. XHR/HTTP requests
3. Timers (setTimeout, setInterval)

**Change Detection Strategies:**

```typescript
import { Component, ChangeDetectionStrategy } from '@angular/core';

// Default strategy - checks entire component tree
@Component({
  selector: 'app-default',
  changeDetection: ChangeDetectionStrategy.Default,
  template: `...`
})
export class DefaultComponent { }

// OnPush strategy - only checks when:
// - Input properties change (reference)
// - Events from component or children
// - Observable emits (with async pipe)
// - Manual trigger (ChangeDetectorRef.markForCheck())
@Component({
  selector: 'app-on-push',
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `...`
})
export class OnPushComponent { }
```

### OnPush Optimization

```typescript
@Component({
  selector: 'app-user-list',
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <div *ngFor="let user of users">{{ user.name }}</div>
    <div>{{ data$ | async }}</div>
  `
})
export class UserListComponent {
  @Input() users: User[] = [];  // Must be new reference to trigger
  data$ = this.dataService.getData();
  
  constructor(private dataService: DataService) {}
}

// Parent component
@Component({
  template: `<app-user-list [users]="users"></app-user-list>`
})
export class ParentComponent {
  users = [{ name: 'John' }];
  
  addUser() {
    // ✅ Creates new reference - triggers change detection
    this.users = [...this.users, { name: 'Jane' }];
    
    // ❌ Mutates array - won't trigger OnPush
    // this.users.push({ name: 'Jane' });
  }
}
```

### Manual Change Detection

```typescript
import { ChangeDetectorRef } from '@angular/core';

@Component({
  selector: 'app-manual',
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `<p>{{ count }}</p>`
})
export class ManualComponent {
  count = 0;
  
  constructor(private cdr: ChangeDetectorRef) {}
  
  increment() {
    this.count++;
    this.cdr.markForCheck();  // Schedule change detection
  }
  
  detectChanges() {
    this.cdr.detectChanges();  // Run immediately
  }
  
  detach() {
    this.cdr.detach();  // Detach from change detection
  }
  
  reattach() {
    this.cdr.reattach();  // Reattach to change detection
  }
}
```

### Zone.js & NgZone

```typescript
import { NgZone } from '@angular/core';

@Component({
  selector: 'app-zone',
  template: `<p>{{ value }}</p>`
})
export class ZoneComponent {
  value = 0;
  
  constructor(private ngZone: NgZone) {}
  
  // Run outside Angular zone (no change detection)
  runOutsideAngular() {
    this.ngZone.runOutsideAngular(() => {
      setInterval(() => {
        this.value++;  // Won't trigger change detection
      }, 1000);
    });
  }
  
  // Run inside Angular zone (triggers change detection)
  runInsideAngular() {
    this.ngZone.run(() => {
      this.value++;  // Triggers change detection
    });
  }
}
```

---

## 6. Performance Optimization {#performance}

### TrackBy in ngFor

```typescript
@Component({
  template: `
    <!-- Without trackBy - recreates all DOM nodes on change -->
    <div *ngFor="let item of items">{{ item.name }}</div>
    
    <!-- With trackBy - only updates changed items -->
    <div *ngFor="let item of items; trackBy: trackByFn">
      {{ item.name }}
    </div>
  `
})
export class TrackByComponent {
  items = [
    { id: 1, name: 'Item 1' },
    { id: 2, name: 'Item 2' }
  ];
  
  trackByFn(index: number, item: any): number {
    return item.id;  // Unique identifier
  }
}
```

### Lazy Loading Modules

```typescript
// app.routes.ts
export const routes: Routes = [
  { path: '', component: HomeComponent },
  {
    path: 'admin',
    loadChildren: () => import('./admin/admin.routes').then(m => m.adminRoutes)
  },
  {
    path: 'user',
    loadComponent: () => import('./user/user.component').then(m => m.UserComponent)
  }
];
```

### Preloading Strategies

```typescript
import { PreloadAllModules, provideRouter, withPreloading } from '@angular/router';

// Preload all lazy modules
bootstrapApplication(AppComponent, {
  providers: [
    provideRouter(routes, withPreloading(PreloadAllModules))
  ]
});

// Custom preloading strategy
import { Injectable } from '@angular/core';
import { PreloadingStrategy, Route } from '@angular/router';
import { Observable, of, timer } from 'rxjs';
import { switchMap } from 'rxjs/operators';

@Injectable({ providedIn: 'root' })
export class CustomPreloadStrategy implements PreloadingStrategy {
  preload(route: Route, load: () => Observable<any>): Observable<any> {
    if (route.data?.['preload']) {
      console.log('Preloading:', route.path);
      return timer(2000).pipe(switchMap(() => load()));
    }
    return of(null);
  }
}

// Usage in routes
{ 
  path: 'admin', 
  loadChildren: () => import('./admin/admin.routes'),
  data: { preload: true }
}
```

### Virtual Scrolling

```typescript
import { ScrollingModule } from '@angular/cdk/scrolling';

@Component({
  standalone: true,
  imports: [ScrollingModule],
  template: `
    <cdk-virtual-scroll-viewport itemSize="50" style="height: 400px;">
      <div *cdkVirtualFor="let item of items" class="item">
        {{ item }}
      </div>
    </cdk-virtual-scroll-viewport>
  `
})
export class VirtualScrollComponent {
  items = Array.from({ length: 10000 }, (_, i) => `Item ${i + 1}`);
}
```

### Pure Pipes

```typescript
// Impure pipe - recalculates on every change detection
@Pipe({ 
  name: 'filter',
  pure: false  // Expensive!
})
export class FilterPipe implements PipeTransform {
  transform(items: any[], filter: string): any[] {
    return items.filter(item => item.includes(filter));
  }
}

// Better: Use pure pipe with immutable data
@Pipe({ 
  name: 'filter',
  pure: true  // Only recalculates when input reference changes
})
export class FilterPipe implements PipeTransform {
  transform(items: any[], filter: string): any[] {
    return items.filter(item => item.includes(filter));
  }
}

// Component creates new filtered array
this.filteredItems = this.items.filter(item => item.includes(this.filter));
```

### Detaching Change Detector

```typescript
@Component({
  template: `
    <p>{{ counter }}</p>
    <button (click)="start()">Start</button>
  `
})
export class DetachedComponent implements OnInit, OnDestroy {
  counter = 0;
  private interval: any;
  
  constructor(private cdr: ChangeDetectorRef) {}
  
  ngOnInit() {
    // Detach from automatic change detection
    this.cdr.detach();
  }
  
  start() {
    this.interval = setInterval(() => {
      this.counter++;
      this.cdr.detectChanges();  // Manual trigger
    }, 1000);
  }
  
  ngOnDestroy() {
    if (this.interval) {
      clearInterval(this.interval);
    }
  }
}
```

### Memoization

```typescript
import { memoize } from 'lodash-es';

@Component({
  template: `<div>{{ getExpensiveValue() }}</div>`
})
export class MemoizedComponent {
  private data = [/* large dataset */];
  
  // Memoize expensive function
  getExpensiveValue = memoize((id: number) => {
    console.log('Expensive calculation for', id);
    return this.data.filter(/* complex logic */).reduce(/* more logic */);
  });
}
```

This completes Part 2. Part 3 will cover enterprise topics like authentication, interceptors, micro-frontends, testing, and more.
# Angular 21 Complete Interview Guide - Part 3: Enterprise Topics

## Table of Contents
1. [Authentication & Authorization](#auth)
2. [Route Guards](#guards)
3. [Interceptors (Advanced)](#interceptors)
4. [Micro-frontends](#micro-frontends)
5. [Server-Side Rendering (SSR)](#ssr)
6. [Testing](#testing)
7. [Security](#security)
8. [Build & Deployment](#build)

---

## 1. Authentication & Authorization {#auth}

### JWT Authentication Flow

```typescript
// auth.service.ts
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { BehaviorSubject, Observable, tap } from 'rxjs';
import { Router } from '@angular/router';

interface AuthResponse {
  token: string;
  refreshToken: string;
  user: User;
}

interface User {
  id: string;
  email: string;
  name: string;
  roles: string[];
}

@Injectable({ providedIn: 'root' })
export class AuthService {
  private apiUrl = 'https://api.example.com/auth';
  private currentUserSubject = new BehaviorSubject<User | null>(null);
  public currentUser$ = this.currentUserSubject.asObservable();
  
  constructor(
    private http: HttpClient,
    private router: Router
  ) {
    this.loadUserFromStorage();
  }
  
  get currentUser(): User | null {
    return this.currentUserSubject.value;
  }
  
  get isLoggedIn(): boolean {
    return !!this.getToken();
  }
  
  login(email: string, password: string): Observable<AuthResponse> {
    return this.http.post<AuthResponse>(`${this.apiUrl}/login`, {
      email,
      password
    }).pipe(
      tap(response => this.handleAuthResponse(response))
    );
  }
  
  register(userData: any): Observable<AuthResponse> {
    return this.http.post<AuthResponse>(`${this.apiUrl}/register`, userData)
      .pipe(
        tap(response => this.handleAuthResponse(response))
      );
  }
  
  logout(): void {
    localStorage.removeItem('token');
    localStorage.removeItem('refreshToken');
    localStorage.removeItem('user');
    this.currentUserSubject.next(null);
    this.router.navigate(['/login']);
  }
  
  refreshToken(): Observable<AuthResponse> {
    const refreshToken = this.getRefreshToken();
    return this.http.post<AuthResponse>(`${this.apiUrl}/refresh`, {
      refreshToken
    }).pipe(
      tap(response => this.handleAuthResponse(response))
    );
  }
  
  getToken(): string | null {
    return localStorage.getItem('token');
  }
  
  getRefreshToken(): string | null {
    return localStorage.getItem('refreshToken');
  }
  
  hasRole(role: string): boolean {
    return this.currentUser?.roles.includes(role) || false;
  }
  
  hasAnyRole(roles: string[]): boolean {
    return roles.some(role => this.hasRole(role));
  }
  
  private handleAuthResponse(response: AuthResponse): void {
    localStorage.setItem('token', response.token);
    localStorage.setItem('refreshToken', response.refreshToken);
    localStorage.setItem('user', JSON.stringify(response.user));
    this.currentUserSubject.next(response.user);
  }
  
  private loadUserFromStorage(): void {
    const userStr = localStorage.getItem('user');
    if (userStr) {
      try {
        const user = JSON.parse(userStr);
        this.currentUserSubject.next(user);
      } catch (e) {
        this.logout();
      }
    }
  }
}
```

### Login Component

```typescript
import { Component } from '@angular/core';
import { FormBuilder, FormGroup, Validators, ReactiveFormsModule } from '@angular/forms';
import { Router } from '@angular/router';
import { AuthService } from './auth.service';

@Component({
  selector: 'app-login',
  standalone: true,
  imports: [ReactiveFormsModule],
  template: `
    <form [formGroup]="loginForm" (ngSubmit)="onSubmit()">
      <h2>Login</h2>
      
      <div>
        <input formControlName="email" type="email" placeholder="Email">
        <div *ngIf="email.invalid && email.touched">
          <p *ngIf="email.errors?.['required']">Email required</p>
          <p *ngIf="email.errors?.['email']">Invalid email</p>
        </div>
      </div>
      
      <div>
        <input formControlName="password" type="password" placeholder="Password">
        <div *ngIf="password.invalid && password.touched">
          <p *ngIf="password.errors?.['required']">Password required</p>
        </div>
      </div>
      
      <div *ngIf="errorMessage" class="error">{{ errorMessage }}</div>
      
      <button type="submit" [disabled]="loginForm.invalid || loading">
        {{ loading ? 'Logging in...' : 'Login' }}
      </button>
    </form>
  `
})
export class LoginComponent {
  loginForm: FormGroup;
  errorMessage = '';
  loading = false;
  
  constructor(
    private fb: FormBuilder,
    private authService: AuthService,
    private router: Router
  ) {
    this.loginForm = this.fb.group({
      email: ['', [Validators.required, Validators.email]],
      password: ['', Validators.required]
    });
  }
  
  get email() {
    return this.loginForm.get('email')!;
  }
  
  get password() {
    return this.loginForm.get('password')!;
  }
  
  onSubmit(): void {
    if (this.loginForm.valid) {
      this.loading = true;
      this.errorMessage = '';
      
      const { email, password } = this.loginForm.value;
      
      this.authService.login(email, password).subscribe({
        next: () => {
          this.router.navigate(['/dashboard']);
        },
        error: (error) => {
          this.errorMessage = error.error?.message || 'Login failed';
          this.loading = false;
        },
        complete: () => {
          this.loading = false;
        }
      });
    }
  }
}
```

### OAuth2 / Social Login

```typescript
import { Injectable } from '@angular/core';

@Injectable({ providedIn: 'root' })
export class SocialAuthService {
  constructor(private http: HttpClient) {}
  
  // Google Login
  loginWithGoogle(): void {
    const googleAuthUrl = 'https://accounts.google.com/o/oauth2/v2/auth';
    const params = new URLSearchParams({
      client_id: 'YOUR_CLIENT_ID',
      redirect_uri: 'http://localhost:4200/auth/google/callback',
      response_type: 'code',
      scope: 'openid email profile',
      access_type: 'offline'
    });
    
    window.location.href = `${googleAuthUrl}?${params.toString()}`;
  }
  
  // Handle Google callback
  handleGoogleCallback(code: string): Observable<AuthResponse> {
    return this.http.post<AuthResponse>('/api/auth/google', { code });
  }
  
  // GitHub Login
  loginWithGitHub(): void {
    const githubAuthUrl = 'https://github.com/login/oauth/authorize';
    const params = new URLSearchParams({
      client_id: 'YOUR_CLIENT_ID',
      redirect_uri: 'http://localhost:4200/auth/github/callback',
      scope: 'read:user user:email'
    });
    
    window.location.href = `${githubAuthUrl}?${params.toString()}`;
  }
}
```

---

## 2. Route Guards {#guards}

### Functional Guards (Angular 15+)

**Auth Guard:**
```typescript
// auth.guard.ts
import { inject } from '@angular/core';
import { Router, CanActivateFn } from '@angular/router';
import { AuthService } from './auth.service';

export const authGuard: CanActivateFn = (route, state) => {
  const authService = inject(AuthService);
  const router = inject(Router);
  
  if (authService.isLoggedIn) {
    return true;
  }
  
  // Redirect to login with return URL
  return router.createUrlTree(['/login'], {
    queryParams: { returnUrl: state.url }
  });
};

// Usage in routes
export const routes: Routes = [
  { path: 'login', component: LoginComponent },
  { 
    path: 'dashboard', 
    component: DashboardComponent,
    canActivate: [authGuard]
  }
];
```

**Role Guard:**
```typescript
// role.guard.ts
export const roleGuard = (allowedRoles: string[]): CanActivateFn => {
  return (route, state) => {
    const authService = inject(AuthService);
    const router = inject(Router);
    
    if (!authService.isLoggedIn) {
      return router.createUrlTree(['/login']);
    }
    
    if (authService.hasAnyRole(allowedRoles)) {
      return true;
    }
    
    // Forbidden
    return router.createUrlTree(['/forbidden']);
  };
};

// Usage
{ 
  path: 'admin', 
  component: AdminComponent,
  canActivate: [authGuard, roleGuard(['admin', 'superadmin'])]
}
```

**Data-based Role Guard:**
```typescript
export const dataRoleGuard: CanActivateFn = (route, state) => {
  const authService = inject(AuthService);
  const router = inject(Router);
  const requiredRoles = route.data['roles'] as string[];
  
  if (!authService.isLoggedIn) {
    return router.createUrlTree(['/login']);
  }
  
  if (requiredRoles && !authService.hasAnyRole(requiredRoles)) {
    return router.createUrlTree(['/forbidden']);
  }
  
  return true;
};

// Usage
{ 
  path: 'admin', 
  component: AdminComponent,
  canActivate: [dataRoleGuard],
  data: { roles: ['admin', 'superadmin'] }
}
```

**Can Deactivate (Unsaved Changes):**
```typescript
export interface CanComponentDeactivate {
  canDeactivate: () => boolean | Observable<boolean>;
}

export const canDeactivateGuard: CanDeactivateFn<CanComponentDeactivate> = (component) => {
  return component.canDeactivate ? component.canDeactivate() : true;
};

// Component implementation
@Component({
  selector: 'app-edit-form'
})
export class EditFormComponent implements CanComponentDeactivate {
  formDirty = false;
  
  canDeactivate(): boolean {
    if (this.formDirty) {
      return confirm('You have unsaved changes. Do you really want to leave?');
    }
    return true;
  }
}

// Route
{ 
  path: 'edit/:id', 
  component: EditFormComponent,
  canDeactivate: [canDeactivateGuard]
}
```

**Can Load (Lazy Loading):**
```typescript
export const canLoadGuard: CanLoadFn = (route) => {
  const authService = inject(AuthService);
  const router = inject(Router);
  
  if (authService.hasRole('admin')) {
    return true;
  }
  
  return router.createUrlTree(['/forbidden']);
};

// Usage
{ 
  path: 'admin',
  canLoad: [canLoadGuard],
  loadChildren: () => import('./admin/admin.routes')
}
```

**Resolve Guard (Pre-fetch Data):**
```typescript
// user.resolver.ts
export const userResolver: ResolveFn<User> = (route, state) => {
  const userService = inject(UserService);
  const router = inject(Router);
  const userId = route.paramMap.get('id')!;
  
  return userService.getUser(userId).pipe(
    catchError(() => {
      router.navigate(['/not-found']);
      return EMPTY;
    })
  );
};

// Route
{ 
  path: 'user/:id',
  component: UserDetailComponent,
  resolve: { user: userResolver }
}

// Component
@Component({
  template: `<h1>{{ user.name }}</h1>`
})
export class UserDetailComponent implements OnInit {
  user!: User;
  
  constructor(private route: ActivatedRoute) {}
  
  ngOnInit() {
    this.user = this.route.snapshot.data['user'];
  }
}
```

### Class-based Guards (Legacy)

```typescript
import { Injectable } from '@angular/core';
import { CanActivate, ActivatedRouteSnapshot, RouterStateSnapshot } from '@angular/router';

@Injectable({ providedIn: 'root' })
export class AuthGuard implements CanActivate {
  constructor(
    private authService: AuthService,
    private router: Router
  ) {}
  
  canActivate(
    route: ActivatedRouteSnapshot,
    state: RouterStateSnapshot
  ): boolean {
    if (this.authService.isLoggedIn) {
      return true;
    }
    
    this.router.navigate(['/login'], {
      queryParams: { returnUrl: state.url }
    });
    return false;
  }
}
```

---

## 3. Interceptors (Advanced) {#interceptors}

### Token Interceptor with Retry

```typescript
import { HttpInterceptorFn, HttpErrorResponse } from '@angular/common/http';
import { inject } from '@angular/core';
import { catchError, switchMap, throwError } from 'rxjs';
import { AuthService } from './auth.service';

export const tokenInterceptor: HttpInterceptorFn = (req, next) => {
  const authService = inject(AuthService);
  const token = authService.getToken();
  
  // Clone request and add token
  const authReq = token ? req.clone({
    setHeaders: { Authorization: `Bearer ${token}` }
  }) : req;
  
  return next(authReq).pipe(
    catchError((error: HttpErrorResponse) => {
      if (error.status === 401 && !authReq.url.includes('/auth/')) {
        // Token expired - try to refresh
        return authService.refreshToken().pipe(
          switchMap(() => {
            // Retry with new token
            const newToken = authService.getToken();
            const retryReq = req.clone({
              setHeaders: { Authorization: `Bearer ${newToken}` }
            });
            return next(retryReq);
          }),
          catchError(refreshError => {
            // Refresh failed - logout
            authService.logout();
            return throwError(() => refreshError);
          })
        );
      }
      return throwError(() => error);
    })
  );
};
```

### Caching Interceptor

```typescript
import { HttpInterceptorFn, HttpResponse } from '@angular/common/http';
import { of } from 'rxjs';
import { tap } from 'rxjs/operators';

const cache = new Map<string, HttpResponse<any>>();
const CACHE_TIME = 5 * 60 * 1000; // 5 minutes

export const cacheInterceptor: HttpInterceptorFn = (req, next) => {
  // Only cache GET requests
  if (req.method !== 'GET') {
    return next(req);
  }
  
  // Check for cache-control header
  if (req.headers.get('x-no-cache')) {
    return next(req);
  }
  
  const cachedResponse = cache.get(req.url);
  if (cachedResponse) {
    const age = Date.now() - cachedResponse.headers.get('x-cache-time')!;
    if (age < CACHE_TIME) {
      console.log(`Cache hit: ${req.url}`);
      return of(cachedResponse.clone());
    } else {
      cache.delete(req.url);
    }
  }
  
  return next(req).pipe(
    tap(event => {
      if (event instanceof HttpResponse) {
        const response = event.clone({
          headers: event.headers.set('x-cache-time', Date.now().toString())
        });
        cache.set(req.url, response);
      }
    })
  );
};
```

### Loading Interceptor

```typescript
import { HttpInterceptorFn } from '@angular/common/http';
import { inject } from '@angular/core';
import { finalize } from 'rxjs';
import { LoadingService } from './loading.service';

export const loadingInterceptor: HttpInterceptorFn = (req, next) => {
  const loadingService = inject(LoadingService);
  
  // Ignore loading for specific requests
  if (req.headers.get('x-no-loading')) {
    return next(req);
  }
  
  loadingService.show();
  
  return next(req).pipe(
    finalize(() => loadingService.hide())
  );
};

// loading.service.ts
@Injectable({ providedIn: 'root' })
export class LoadingService {
  private loadingSubject = new BehaviorSubject<boolean>(false);
  public loading$ = this.loadingSubject.asObservable();
  private requests = 0;
  
  show(): void {
    this.requests++;
    this.loadingSubject.next(true);
  }
  
  hide(): void {
    this.requests = Math.max(0, this.requests - 1);
    if (this.requests === 0) {
      this.loadingSubject.next(false);
    }
  }
}
```

### Retry with Exponential Backoff

```typescript
import { HttpInterceptorFn, HttpErrorResponse } from '@angular/common/http';
import { retry, timer } from 'rxjs';

export const retryInterceptor: HttpInterceptorFn = (req, next) => {
  const maxRetries = 3;
  const initialDelay = 1000;
  
  return next(req).pipe(
    retry({
      count: maxRetries,
      delay: (error: HttpErrorResponse, retryCount) => {
        // Only retry on specific errors
        if (error.status >= 500 || error.status === 0) {
          const delay = initialDelay * Math.pow(2, retryCount - 1);
          console.log(`Retry ${retryCount}/${maxRetries} after ${delay}ms`);
          return timer(delay);
        }
        throw error;
      }
    })
  );
};
```

### Request/Response Logging

```typescript
import { HttpInterceptorFn, HttpResponse } from '@angular/common/http';
import { tap } from 'rxjs';

export const loggingInterceptor: HttpInterceptorFn = (req, next) => {
  const startTime = Date.now();
  
  console.group(`HTTP ${req.method} ${req.url}`);
  console.log('Request:', req);
  console.log('Headers:', req.headers);
  console.log('Body:', req.body);
  
  return next(req).pipe(
    tap({
      next: (event) => {
        if (event instanceof HttpResponse) {
          const elapsed = Date.now() - startTime;
          console.log(`Response (${elapsed}ms):`, event);
          console.log('Status:', event.status);
          console.log('Body:', event.body);
          console.groupEnd();
        }
      },
      error: (error) => {
        const elapsed = Date.now() - startTime;
        console.error(`Error (${elapsed}ms):`, error);
        console.groupEnd();
      }
    })
  );
};
```

---

## 4. Micro-frontends {#micro-frontends}

### Module Federation (Webpack 5)

**Installation:**
```bash
npm install @angular-architects/module-federation
ng add @angular-architects/module-federation
```

**Host Application (Shell):**

```typescript
// webpack.config.js
const { shareAll, withModuleFederationPlugin } = require('@angular-architects/module-federation/webpack');

module.exports = withModuleFederationPlugin({
  remotes: {
    "mfe1": "http://localhost:4201/remoteEntry.js",
    "mfe2": "http://localhost:4202/remoteEntry.js"
  },
  shared: shareAll({ 
    singleton: true, 
    strictVersion: true, 
    requiredVersion: 'auto' 
  })
});

// app.routes.ts
export const routes: Routes = [
  {
    path: 'mfe1',
    loadChildren: () => import('mfe1/Module').then(m => m.RemoteEntryModule)
  },
  {
    path: 'mfe2',
    loadChildren: () => import('mfe2/Module').then(m => m.RemoteEntryModule)
  }
];

// app.component.ts
@Component({
  selector: 'app-root',
  template: `
    <nav>
      <a routerLink="/">Home</a>
      <a routerLink="/mfe1">Micro-frontend 1</a>
      <a routerLink="/mfe2">Micro-frontend 2</a>
    </nav>
    <router-outlet></router-outlet>
  `
})
export class AppComponent { }
```

**Remote Application (Micro-frontend):**

```typescript
// webpack.config.js
const { shareAll, withModuleFederationPlugin } = require('@angular-architects/module-federation/webpack');

module.exports = withModuleFederationPlugin({
  name: 'mfe1',
  filename: 'remoteEntry.js',
  exposes: {
    './Module': './src/app/remote-entry/entry.module.ts'
  },
  shared: shareAll({ 
    singleton: true, 
    strictVersion: true, 
    requiredVersion: 'auto' 
  })
});

// entry.module.ts
@NgModule({
  imports: [
    RouterModule.forChild([
      { path: '', component: HomeComponent }
    ])
  ],
  declarations: [HomeComponent]
})
export class RemoteEntryModule { }
```

### Dynamic Module Loading

```typescript
import { Injectable } from '@angular/core';
import { loadRemoteModule } from '@angular-architects/module-federation';

@Injectable({ providedIn: 'root' })
export class MicroFrontendService {
  async loadRemote(remoteName: string, exposedModule: string) {
    const module = await loadRemoteModule({
      type: 'module',
      remoteEntry: `http://localhost:4201/remoteEntry.js`,
      exposedModule: `./${exposedModule}`
    });
    return module;
  }
}
```

### Communication Between Micro-frontends

**Shared State Service:**
```typescript
// shared-state.service.ts
@Injectable({ providedIn: 'root' })
export class SharedStateService {
  private stateSubject = new BehaviorSubject<any>({});
  public state$ = this.stateSubject.asObservable();
  
  setState(key: string, value: any): void {
    const currentState = this.stateSubject.value;
    this.stateSubject.next({ ...currentState, [key]: value });
  }
  
  getState(key: string): any {
    return this.stateSubject.value[key];
  }
}

// MFE 1 - Set state
constructor(private sharedState: SharedStateService) {}

sendData() {
  this.sharedState.setState('userData', { id: 1, name: 'John' });
}

// MFE 2 - Read state
ngOnInit() {
  this.sharedState.state$.subscribe(state => {
    console.log('Shared state:', state.userData);
  });
}
```

**Event Bus:**
```typescript
@Injectable({ providedIn: 'root' })
export class EventBusService {
  private eventSubject = new Subject<{ type: string; payload: any }>();
  public events$ = this.eventSubject.asObservable();
  
  emit(type: string, payload: any): void {
    this.eventSubject.next({ type, payload });
  }
  
  on(type: string): Observable<any> {
    return this.events$.pipe(
      filter(event => event.type === type),
      map(event => event.payload)
    );
  }
}

// MFE 1 - Emit event
this.eventBus.emit('USER_LOGGED_IN', { userId: 123 });

// MFE 2 - Listen to event
this.eventBus.on('USER_LOGGED_IN').subscribe(payload => {
  console.log('User logged in:', payload);
});
```

### Web Components Approach

```typescript
// Create standalone component as web component
import { createCustomElement } from '@angular/elements';
import { createApplication } from '@angular/platform-browser';

@Component({
  selector: 'app-widget',
  standalone: true,
  template: `<p>Standalone Widget</p>`
})
export class WidgetComponent { }

// Bootstrap as web component
(async () => {
  const app = await createApplication({
    providers: []
  });
  
  const widgetElement = createCustomElement(WidgetComponent, {
    injector: app.injector
  });
  
  customElements.define('app-widget', widgetElement);
})();

// Use in any HTML
<app-widget></app-widget>
```

---

## 5. Server-Side Rendering (SSR) {#ssr}

### Setup Angular Universal

```bash
ng add @angular/ssr
```

**Server Configuration:**

```typescript
// server.ts (auto-generated)
import 'zone.js/node';
import { APP_BASE_HREF } from '@angular/common';
import { CommonEngine } from '@angular/ssr';
import express from 'express';
import { fileURLToPath } from 'node:url';
import { dirname, join, resolve } from 'node:path';
import bootstrap from './src/main.server';

export function app(): express.Express {
  const server = express();
  const serverDistFolder = dirname(fileURLToPath(import.meta.url));
  const browserDistFolder = resolve(serverDistFolder, '../browser');
  const indexHtml = join(serverDistFolder, 'index.server.html');
  
  const commonEngine = new CommonEngine();
  
  server.set('view engine', 'html');
  server.set('views', browserDistFolder);
  
  // Serve static files
  server.get('*.*', express.static(browserDistFolder, {
    maxAge: '1y'
  }));
  
  // All regular routes
  server.get('*', (req, res, next) => {
    const { protocol, originalUrl, baseUrl, headers } = req;
    
    commonEngine
      .render({
        bootstrap,
        documentFilePath: indexHtml,
        url: `${protocol}://${headers.host}${originalUrl}`,
        publicPath: browserDistFolder,
        providers: [{ provide: APP_BASE_HREF, useValue: baseUrl }],
      })
      .then((html) => res.send(html))
      .catch((err) => next(err));
  });
  
  return server;
}

function run(): void {
  const port = process.env['PORT'] || 4000;
  const server = app();
  server.listen(port, () => {
    console.log(`Node Express server listening on http://localhost:${port}`);
  });
}

run();
```

### Transfer State (Avoid Duplicate HTTP Calls)

```typescript
import { TransferState, makeStateKey } from '@angular/platform-browser';
import { isPlatformBrowser, isPlatformServer } from '@angular/common';

const USERS_KEY = makeStateKey<User[]>('users');

@Injectable({ providedIn: 'root' })
export class UserService {
  constructor(
    private http: HttpClient,
    private transferState: TransferState,
    @Inject(PLATFORM_ID) private platformId: Object
  ) {}
  
  getUsers(): Observable<User[]> {
    // Check if data exists in transfer state (from SSR)
    const cachedUsers = this.transferState.get(USERS_KEY, null);
    
    if (cachedUsers) {
      // Remove from state to free memory
      this.transferState.remove(USERS_KEY);
      return of(cachedUsers);
    }
    
    // Fetch from API
    return this.http.get<User[]>('/api/users').pipe(
      tap(users => {
        // Store in transfer state if on server
        if (isPlatformServer(this.platformId)) {
          this.transferState.set(USERS_KEY, users);
        }
      })
    );
  }
}
```

### Platform-specific Code

```typescript
import { isPlatformBrowser, isPlatformServer } from '@angular/common';
import { PLATFORM_ID, Inject } from '@angular/core';

@Component({
  selector: 'app-example'
})
export class ExampleComponent implements OnInit {
  constructor(@Inject(PLATFORM_ID) private platformId: Object) {}
  
  ngOnInit() {
    if (isPlatformBrowser(this.platformId)) {
      // Browser-specific code
      console.log('Running in browser');
      localStorage.setItem('key', 'value');
      window.addEventListener('scroll', this.onScroll);
    }
    
    if (isPlatformServer(this.platformId)) {
      // Server-specific code
      console.log('Running on server');
    }
  }
  
  onScroll() {
    // Handle scroll
  }
}
```

### Prerendering (Static Site Generation)

```typescript
// angular.json
{
  "projects": {
    "my-app": {
      "architect": {
        "prerender": {
          "builder": "@angular/ssr:prerender",
          "options": {
            "routes": [
              "/",
              "/about",
              "/contact",
              "/blog/post-1",
              "/blog/post-2"
            ]
          }
        }
      }
    }
  }
}

// Build and prerender
npm run prerender
```

**Dynamic Route Discovery:**
```typescript
// routes-file.ts
export async function getRoutes(): Promise<string[]> {
  const response = await fetch('https://api.example.com/routes');
  const routes = await response.json();
  return routes.map(r => r.path);
}
```

---

## 6. Testing {#testing}

### Unit Testing (Jasmine + Karma)

**Component Testing:**
```typescript
import { ComponentFixture, TestBed } from '@angular/core/testing';
import { UserComponent } from './user.component';

describe('UserComponent', () => {
  let component: UserComponent;
  let fixture: ComponentFixture<UserComponent>;
  
  beforeEach(async () => {
    await TestBed.configureTestingModule({
      imports: [UserComponent]
    }).compileComponents();
    
    fixture = TestBed.createComponent(UserComponent);
    component = fixture.componentInstance;
    fixture.detectChanges();
  });
  
  it('should create', () => {
    expect(component).toBeTruthy();
  });
  
  it('should display user name', () => {
    component.userName = 'John Doe';
    fixture.detectChanges();
    
    const compiled = fixture.nativeElement;
    expect(compiled.querySelector('h1')?.textContent).toContain('John Doe');
  });
  
  it('should call method on button click', () => {
    spyOn(component, 'handleClick');
    
    const button = fixture.nativeElement.querySelector('button');
    button.click();
    
    expect(component.handleClick).toHaveBeenCalled();
  });
});
```

**Service Testing:**
```typescript
import { TestBed } from '@angular/core/testing';
import { HttpClientTestingModule, HttpTestingController } from '@angular/common/http/testing';
import { UserService } from './user.service';

describe('UserService', () => {
  let service: UserService;
  let httpMock: HttpTestingController;
  
  beforeEach(() => {
    TestBed.configureTestingModule({
      imports: [HttpClientTestingModule],
      providers: [UserService]
    });
    
    service = TestBed.inject(UserService);
    httpMock = TestBed.inject(HttpTestingController);
  });
  
  afterEach(() => {
    httpMock.verify();
  });
  
  it('should fetch users', () => {
    const mockUsers = [
      { id: 1, name: 'John' },
      { id: 2, name: 'Jane' }
    ];
    
    service.getUsers().subscribe(users => {
      expect(users).toEqual(mockUsers);
      expect(users.length).toBe(2);
    });
    
    const req = httpMock.expectOne('/api/users');
    expect(req.request.method).toBe('GET');
    req.flush(mockUsers);
  });
  
  it('should handle error', () => {
    service.getUsers().subscribe({
      next: () => fail('should have failed'),
      error: (error) => {
        expect(error.status).toBe(404);
      }
    });
    
    const req = httpMock.expectOne('/api/users');
    req.flush('Not found', { status: 404, statusText: 'Not Found' });
  });
});
```

**Testing with Mocks:**
```typescript
import { of } from 'rxjs';

class MockUserService {
  getUsers() {
    return of([{ id: 1, name: 'Mock User' }]);
  }
}

describe('UserListComponent', () => {
  beforeEach(async () => {
    await TestBed.configureTestingModule({
      imports: [UserListComponent],
      providers: [
        { provide: UserService, useClass: MockUserService }
      ]
    }).compileComponents();
  });
  
  it('should display users from service', () => {
    const fixture = TestBed.createComponent(UserListComponent);
    fixture.detectChanges();
    
    const compiled = fixture.nativeElement;
    expect(compiled.querySelector('li')?.textContent).toContain('Mock User');
  });
});
```

**Testing Forms:**
```typescript
describe('LoginComponent', () => {
  let component: LoginComponent;
  let fixture: ComponentFixture<LoginComponent>;
  
  beforeEach(async () => {
    await TestBed.configureTestingModule({
      imports: [LoginComponent, ReactiveFormsModule]
    }).compileComponents();
    
    fixture = TestBed.createComponent(LoginComponent);
    component = fixture.componentInstance;
    fixture.detectChanges();
  });
  
  it('should create form with controls', () => {
    expect(component.loginForm).toBeDefined();
    expect(component.loginForm.get('email')).toBeDefined();
    expect(component.loginForm.get('password')).toBeDefined();
  });
  
  it('should validate email', () => {
    const emailControl = component.loginForm.get('email')!;
    
    emailControl.setValue('');
    expect(emailControl.hasError('required')).toBeTruthy();
    
    emailControl.setValue('invalid');
    expect(emailControl.hasError('email')).toBeTruthy();
    
    emailControl.setValue('test@example.com');
    expect(emailControl.valid).toBeTruthy();
  });
  
  it('should disable submit when form invalid', () => {
    const button = fixture.nativeElement.querySelector('button[type="submit"]');
    expect(button.disabled).toBeTruthy();
    
    component.loginForm.patchValue({
      email: 'test@example.com',
      password: 'password123'
    });
    fixture.detectChanges();
    
    expect(button.disabled).toBeFalsy();
  });
});
```

### E2E Testing (Cypress)

**Installation:**
```bash
npm install cypress --save-dev
npx cypress open
```

**Cypress Test:**
```typescript
// cypress/e2e/login.cy.ts
describe('Login', () => {
  beforeEach(() => {
    cy.visit('/login');
  });
  
  it('should display login form', () => {
    cy.get('form').should('exist');
    cy.get('input[type="email"]').should('exist');
    cy.get('input[type="password"]').should('exist');
    cy.get('button[type="submit"]').should('exist');
  });
  
  it('should show validation errors', () => {
    cy.get('input[type="email"]').type('invalid');
    cy.get('input[type="email"]').blur();
    cy.contains('Invalid email').should('be.visible');
  });
  
  it('should login successfully', () => {
    cy.intercept('POST', '/api/auth/login', {
      statusCode: 200,
      body: {
        token: 'fake-token',
        user: { id: 1, email: 'test@example.com' }
      }
    }).as('loginRequest');
    
    cy.get('input[type="email"]').type('test@example.com');
    cy.get('input[type="password"]').type('password123');
    cy.get('button[type="submit"]').click();
    
    cy.wait('@loginRequest');
    cy.url().should('include', '/dashboard');
  });
});
```

---

## 7. Security {#security}

### XSS Protection

Angular automatically sanitizes values:

```typescript
import { DomSanitizer, SafeHtml } from '@angular/platform-browser';

@Component({
  selector: 'app-sanitize'
})
export class SanitizeComponent {
  userInput = '<script>alert("XSS")</script>';
  
  constructor(private sanitizer: DomSanitizer) {}
  
  // Angular automatically escapes in interpolation
  // {{ userInput }} → Safe
  
  // Bypass sanitization (use carefully!)
  getSafeHtml(): SafeHtml {
    return this.sanitizer.bypassSecurityTrustHtml(this.userInput);
  }
}

// Template
<div>{{ userInput }}</div>  <!-- Safe: escaped -->
<div [innerHTML]="userInput"></div>  <!-- Safe: sanitized -->
<div [innerHTML]="getSafeHtml()"></div>  <!-- Unsafe: bypassed -->
```

### CSRF Protection

```typescript
// Configure CSRF token
import { provideHttpClient, withXsrfConfiguration } from '@angular/common/http';

bootstrapApplication(AppComponent, {
  providers: [
    provideHttpClient(
      withXsrfConfiguration({
        cookieName: 'XSRF-TOKEN',
        headerName: 'X-XSRF-TOKEN'
      })
    )
  ]
});
```

### Content Security Policy

```html
<!-- index.html -->
<meta http-equiv="Content-Security-Policy" 
      content="default-src 'self'; 
               script-src 'self' 'unsafe-inline';
               style-src 'self' 'unsafe-inline';
               img-src 'self' data: https:;">
```

---

## 8. Build & Deployment {#build}

### Production Build

```bash
# Build for production
ng build --configuration production

# Build with AOT
ng build --aot

# Build with source maps
ng build --source-map

# Analyze bundle
ng build --stats-json
npx webpack-bundle-analyzer dist/my-app/stats.json
```

### Environment Configuration

```typescript
// environment.ts
export const environment = {
  production: false,
  apiUrl: 'http://localhost:3000'
};

// environment.prod.ts
export const environment = {
  production: true,
  apiUrl: 'https://api.production.com'
};

// Usage
import { environment } from '../environments/environment';

@Injectable()
export class ApiService {
  private apiUrl = environment.apiUrl;
}
```

### Bundle Optimization

```typescript
// angular.json
{
  "projects": {
    "my-app": {
      "architect": {
        "build": {
          "configurations": {
            "production": {
              "optimization": true,
              "outputHashing": "all",
              "sourceMap": false,
              "namedChunks": false,
              "extractLicenses": true,
              "vendorChunk": false,
              "buildOptimizer": true,
              "budgets": [
                {
                  "type": "initial",
                  "maximumWarning": "500kb",
                  "maximumError": "1mb"
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

### Docker Deployment

```dockerfile
# Dockerfile
FROM node:20 AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM nginx:alpine
COPY --from=build /app/dist/my-app/browser /usr/share/nginx/html
COPY nginx.conf /etc/nginx/nginx.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

```nginx
# nginx.conf
events { }

http {
  include /etc/nginx/mime.types;
  
  server {
    listen 80;
    server_name localhost;
    root /usr/share/nginx/html;
    index index.html;
    
    location / {
      try_files $uri $uri/ /index.html;
    }
    
    # Cache static assets
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg)$ {
      expires 1y;
      add_header Cache-Control "public, immutable";
    }
  }
}
```

This completes Part 3. Part 4 will include HLD/LLD problems with solutions.
# Angular 21 Complete Interview Guide - Part 4: HLD & LLD Problems

## Table of Contents
1. [High-Level Design Problems](#hld)
2. [Low-Level Design Problems](#lld)
3. [Architecture Patterns](#architecture)
4. [Real-world Scenarios](#scenarios)

---

## 1. High-Level Design Problems {#hld}

### HLD Problem 1: E-commerce Application

**Requirements:**
- Product catalog with search, filters, sorting
- Shopping cart
- User authentication
- Order management
- Payment integration
- Admin panel
- Real-time inventory updates
- Multi-language support
- Mobile responsive

**Solution:**

```
┌─────────────────────────────────────────────────────────────┐
│                    E-commerce Application                    │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                      Module Architecture                     │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │   Core       │  │   Shared     │  │  Features    │      │
│  │  Module      │  │   Module     │  │   Modules    │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
│       │                  │                  │               │
│  ┌────┴──────┐      ┌────┴──────┐      ┌────┴──────┐      │
│  │- Auth     │      │- UI Comps │      │- Products │      │
│  │- Guards   │      │- Pipes    │      │- Cart     │      │
│  │- Intercept│      │- Directive│      │- Orders   │      │
│  │- Services │      │- Utils    │      │- Admin    │      │
│  └───────────┘      └───────────┘      └───────────┘      │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

**Folder Structure:**
```
src/
├── app/
│   ├── core/                       # Singleton services
│   │   ├── auth/
│   │   │   ├── services/
│   │   │   │   ├── auth.service.ts
│   │   │   │   └── token.service.ts
│   │   │   ├── guards/
│   │   │   │   ├── auth.guard.ts
│   │   │   │   └── role.guard.ts
│   │   │   └── interceptors/
│   │   │       ├── auth.interceptor.ts
│   │   │       └── error.interceptor.ts
│   │   ├── services/
│   │   │   ├── api.service.ts
│   │   │   ├── storage.service.ts
│   │   │   └── notification.service.ts
│   │   └── models/
│   │       ├── user.model.ts
│   │       └── api-response.model.ts
│   │
│   ├── shared/                     # Shared components/utilities
│   │   ├── components/
│   │   │   ├── header/
│   │   │   ├── footer/
│   │   │   ├── loading-spinner/
│   │   │   └── confirmation-dialog/
│   │   ├── directives/
│   │   │   ├── click-outside.directive.ts
│   │   │   └── lazy-load.directive.ts
│   │   ├── pipes/
│   │   │   ├── currency-format.pipe.ts
│   │   │   └── truncate.pipe.ts
│   │   └── utils/
│   │       ├── validators.ts
│   │       └── helpers.ts
│   │
│   ├── features/                   # Feature modules
│   │   ├── products/
│   │   │   ├── components/
│   │   │   │   ├── product-list/
│   │   │   │   ├── product-detail/
│   │   │   │   ├── product-filters/
│   │   │   │   └── product-search/
│   │   │   ├── services/
│   │   │   │   └── product.service.ts
│   │   │   ├── state/
│   │   │   │   ├── product.actions.ts
│   │   │   │   ├── product.reducer.ts
│   │   │   │   ├── product.effects.ts
│   │   │   │   └── product.selectors.ts
│   │   │   └── products.routes.ts
│   │   │
│   │   ├── cart/
│   │   │   ├── components/
│   │   │   │   ├── cart-list/
│   │   │   │   ├── cart-item/
│   │   │   │   └── cart-summary/
│   │   │   ├── services/
│   │   │   │   └── cart.service.ts
│   │   │   └── cart.routes.ts
│   │   │
│   │   ├── checkout/
│   │   │   ├── components/
│   │   │   ├── services/
│   │   │   └── checkout.routes.ts
│   │   │
│   │   ├── orders/
│   │   │   ├── components/
│   │   │   ├── services/
│   │   │   └── orders.routes.ts
│   │   │
│   │   └── admin/
│   │       ├── components/
│   │       ├── services/
│   │       └── admin.routes.ts
│   │
│   ├── app.component.ts
│   ├── app.routes.ts
│   └── app.config.ts
│
└── assets/
    ├── i18n/
    │   ├── en.json
    │   └── es.json
    └── images/
```

**State Management (NgRx):**
```typescript
// Product State Architecture
interface AppState {
  products: ProductState;
  cart: CartState;
  auth: AuthState;
  ui: UIState;
}

interface ProductState {
  products: Product[];
  filters: ProductFilters;
  selectedProduct: Product | null;
  loading: boolean;
  error: string | null;
}

// Product Actions
export const loadProducts = createAction('[Products] Load Products');
export const loadProductsSuccess = createAction(
  '[Products] Load Products Success',
  props<{ products: Product[] }>()
);
export const filterProducts = createAction(
  '[Products] Filter',
  props<{ filters: ProductFilters }>()
);
export const addToCart = createAction(
  '[Cart] Add Item',
  props<{ product: Product; quantity: number }>()
);
```

**Service Layer:**
```typescript
@Injectable({ providedIn: 'root' })
export class ProductService {
  private apiUrl = environment.apiUrl;
  
  constructor(private http: HttpClient) {}
  
  getProducts(params?: ProductQueryParams): Observable<ProductResponse> {
    const queryParams = this.buildQueryParams(params);
    return this.http.get<ProductResponse>(`${this.apiUrl}/products`, { params: queryParams });
  }
  
  searchProducts(term: string): Observable<Product[]> {
    return this.http.get<Product[]>(`${this.apiUrl}/products/search`, {
      params: { q: term }
    });
  }
  
  private buildQueryParams(params?: ProductQueryParams): HttpParams {
    let httpParams = new HttpParams();
    if (params) {
      Object.keys(params).forEach(key => {
        if (params[key]) {
          httpParams = httpParams.set(key, params[key]);
        }
      });
    }
    return httpParams;
  }
}
```

**Component Communication:**
```typescript
// Smart Component (Container)
@Component({
  selector: 'app-product-list',
  template: `
    <app-product-filters 
      [filters]="filters$ | async"
      (filterChange)="onFilterChange($event)">
    </app-product-filters>
    
    <app-product-grid
      [products]="products$ | async"
      [loading]="loading$ | async"
      (addToCart)="onAddToCart($event)">
    </app-product-grid>
  `
})
export class ProductListComponent {
  products$ = this.store.select(selectProducts);
  filters$ = this.store.select(selectFilters);
  loading$ = this.store.select(selectLoading);
  
  constructor(private store: Store) {}
  
  onFilterChange(filters: ProductFilters) {
    this.store.dispatch(filterProducts({ filters }));
  }
  
  onAddToCart(event: { product: Product; quantity: number }) {
    this.store.dispatch(addToCart(event));
  }
}

// Presentational Component
@Component({
  selector: 'app-product-grid',
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `...`
})
export class ProductGridComponent {
  @Input() products: Product[] = [];
  @Input() loading = false;
  @Output() addToCart = new EventEmitter<{ product: Product; quantity: number }>();
}
```

**Real-time Updates (WebSocket):**
```typescript
@Injectable({ providedIn: 'root' })
export class InventoryWebSocketService {
  private socket: WebSocket;
  private inventoryUpdates$ = new Subject<InventoryUpdate>();
  
  constructor(private store: Store) {
    this.connect();
  }
  
  connect() {
    this.socket = new WebSocket('wss://api.example.com/inventory');
    
    this.socket.onmessage = (event) => {
      const update = JSON.parse(event.data);
      this.inventoryUpdates$.next(update);
      this.store.dispatch(updateInventory({ update }));
    };
  }
  
  getUpdates(): Observable<InventoryUpdate> {
    return this.inventoryUpdates$.asObservable();
  }
}
```

---

### HLD Problem 2: Social Media Dashboard

**Requirements:**
- User posts (create, read, update, delete)
- Real-time notifications
- Comments and likes
- User profiles
- Follow/unfollow users
- News feed with infinite scroll
- Image/video uploads
- Search functionality

**Architecture:**

```typescript
// Feature-based Architecture
src/app/
├── core/
│   ├── auth/
│   ├── api/
│   └── websocket/
├── features/
│   ├── feed/
│   │   ├── components/
│   │   │   ├── feed-container/
│   │   │   ├── post-card/
│   │   │   ├── post-create/
│   │   │   └── infinite-scroll/
│   │   ├── services/
│   │   │   └── feed.service.ts
│   │   └── state/
│   │       ├── feed.state.ts (Signal-based)
│   │       └── feed.facade.ts
│   │
│   ├── notifications/
│   │   ├── components/
│   │   │   ├── notification-bell/
│   │   │   └── notification-list/
│   │   └── services/
│   │       └── notification.service.ts
│   │
│   ├── profile/
│   └── search/
└── shared/
```

**Signal-based State Management:**
```typescript
// feed.state.ts
import { Injectable, signal, computed } from '@angular/core';

interface Post {
  id: string;
  author: User;
  content: string;
  images: string[];
  likes: number;
  comments: number;
  createdAt: Date;
  isLiked: boolean;
}

@Injectable({ providedIn: 'root' })
export class FeedState {
  // State signals
  private postsSignal = signal<Post[]>([]);
  private loadingSignal = signal(false);
  private hasMoreSignal = signal(true);
  private pageSignal = signal(1);
  
  // Public readonly signals
  readonly posts = this.postsSignal.asReadonly();
  readonly loading = this.loadingSignal.asReadonly();
  readonly hasMore = this.hasMoreSignal.asReadonly();
  
  // Computed signals
  readonly postCount = computed(() => this.postsSignal().length);
  readonly likedPosts = computed(() => 
    this.postsSignal().filter(p => p.isLiked)
  );
  
  // Actions
  addPosts(posts: Post[]) {
    this.postsSignal.update(current => [...current, ...posts]);
    this.pageSignal.update(p => p + 1);
  }
  
  addPost(post: Post) {
    this.postsSignal.update(current => [post, ...current]);
  }
  
  updatePost(id: string, updates: Partial<Post>) {
    this.postsSignal.update(current =>
      current.map(p => p.id === id ? { ...p, ...updates } : p)
    );
  }
  
  toggleLike(postId: string) {
    this.postsSignal.update(current =>
      current.map(p => p.id === postId 
        ? { ...p, isLiked: !p.isLiked, likes: p.likes + (p.isLiked ? -1 : 1) }
        : p
      )
    );
  }
  
  setLoading(loading: boolean) {
    this.loadingSignal.set(loading);
  }
  
  setHasMore(hasMore: boolean) {
    this.hasMoreSignal.set(hasMore);
  }
  
  reset() {
    this.postsSignal.set([]);
    this.pageSignal.set(1);
    this.hasMoreSignal.set(true);
  }
}
```

**Facade Pattern:**
```typescript
// feed.facade.ts
@Injectable({ providedIn: 'root' })
export class FeedFacade {
  // Expose state
  posts = this.feedState.posts;
  loading = this.feedState.loading;
  hasMore = this.feedState.hasMore;
  
  constructor(
    private feedState: FeedState,
    private feedService: FeedService,
    private notificationService: NotificationService
  ) {}
  
  loadFeed() {
    this.feedState.setLoading(true);
    
    this.feedService.getFeed(this.feedState.page()).subscribe({
      next: (response) => {
        this.feedState.addPosts(response.posts);
        this.feedState.setHasMore(response.hasMore);
        this.feedState.setLoading(false);
      },
      error: () => {
        this.feedState.setLoading(false);
        this.notificationService.showError('Failed to load feed');
      }
    });
  }
  
  createPost(content: string, images: File[]) {
    return this.feedService.createPost(content, images).pipe(
      tap(post => {
        this.feedState.addPost(post);
        this.notificationService.showSuccess('Post created');
      })
    );
  }
  
  likePost(postId: string) {
    this.feedState.toggleLike(postId);
    this.feedService.toggleLike(postId).subscribe({
      error: () => {
        // Rollback on error
        this.feedState.toggleLike(postId);
      }
    });
  }
  
  refreshFeed() {
    this.feedState.reset();
    this.loadFeed();
  }
}
```

**Infinite Scroll Component:**
```typescript
@Component({
  selector: 'app-feed-container',
  template: `
    <app-post-create (postCreated)="facade.createPost($event)"></app-post-create>
    
    <div class="feed">
      @for (post of facade.posts(); track post.id) {
        <app-post-card 
          [post]="post"
          (like)="facade.likePost(post.id)"
          (comment)="openComments(post.id)">
        </app-post-card>
      }
    </div>
    
    @if (facade.loading()) {
      <app-loading-spinner></app-loading-spinner>
    }
    
    <div #sentinel class="sentinel"></div>
  `
})
export class FeedContainerComponent implements OnInit, AfterViewInit {
  @ViewChild('sentinel') sentinel!: ElementRef;
  
  constructor(
    public facade: FeedFacade,
    private intersectionObserver: IntersectionObserverService
  ) {}
  
  ngOnInit() {
    this.facade.loadFeed();
  }
  
  ngAfterViewInit() {
    // Infinite scroll with Intersection Observer
    this.intersectionObserver.observe(this.sentinel.nativeElement, () => {
      if (!this.facade.loading() && this.facade.hasMore()) {
        this.facade.loadFeed();
      }
    });
  }
}
```

**WebSocket for Real-time Updates:**
```typescript
@Injectable({ providedIn: 'root' })
export class RealtimeService {
  private socket!: WebSocket;
  private reconnectAttempts = 0;
  private maxReconnectAttempts = 5;
  
  private updates$ = new Subject<RealtimeUpdate>();
  
  connect() {
    this.socket = new WebSocket('wss://api.example.com/realtime');
    
    this.socket.onopen = () => {
      console.log('WebSocket connected');
      this.reconnectAttempts = 0;
    };
    
    this.socket.onmessage = (event) => {
      const update = JSON.parse(event.data);
      this.updates$.next(update);
    };
    
    this.socket.onclose = () => {
      this.attemptReconnect();
    };
    
    this.socket.onerror = (error) => {
      console.error('WebSocket error:', error);
    };
  }
  
  getUpdates(): Observable<RealtimeUpdate> {
    return this.updates$.asObservable();
  }
  
  private attemptReconnect() {
    if (this.reconnectAttempts < this.maxReconnectAttempts) {
      this.reconnectAttempts++;
      const delay = Math.min(1000 * Math.pow(2, this.reconnectAttempts), 30000);
      setTimeout(() => this.connect(), delay);
    }
  }
}
```

---

### HLD Problem 3: Analytics Dashboard

**Requirements:**
- Multiple chart types (line, bar, pie, etc.)
- Real-time data updates
- Date range selection
- Export functionality (PDF, CSV)
- Customizable dashboards
- User preferences
- Lazy loading for heavy charts

**Architecture:**

```typescript
// Module structure
src/app/
├── core/
│   ├── services/
│   │   ├── analytics-api.service.ts
│   │   ├── export.service.ts
│   │   └── websocket.service.ts
│   └── models/
│       └── chart-config.model.ts
│
├── features/
│   ├── dashboard/
│   │   ├── components/
│   │   │   ├── dashboard-grid/
│   │   │   ├── widget-container/
│   │   │   ├── date-range-picker/
│   │   │   └── export-menu/
│   │   ├── services/
│   │   │   └── dashboard.service.ts
│   │   └── state/
│   │       └── dashboard.state.ts
│   │
│   └── widgets/
│       ├── line-chart/
│       ├── bar-chart/
│       ├── pie-chart/
│       └── kpi-card/
│
└── shared/
    ├── directives/
    │   └── lazy-render.directive.ts
    └── utils/
        └── chart-helpers.ts
```

**Dynamic Component Loading:**
```typescript
@Injectable({ providedIn: 'root' })
export class WidgetLoaderService {
  private componentMap = new Map<WidgetType, Type<any>>([
    ['line-chart', LineChartComponent],
    ['bar-chart', BarChartComponent],
    ['pie-chart', PieChartComponent],
    ['kpi-card', KpiCardComponent]
  ]);
  
  async loadWidget(type: WidgetType): Promise<Type<any>> {
    // Lazy load widget component
    const component = this.componentMap.get(type);
    if (!component) {
      throw new Error(`Unknown widget type: ${type}`);
    }
    return component;
  }
}

// Dashboard Grid Component
@Component({
  selector: 'app-dashboard-grid',
  template: `
    <div class="grid" cdkDropListGroup>
      @for (widget of widgets; track widget.id) {
        <div 
          cdkDrag
          [style.grid-column]="'span ' + widget.width"
          [style.grid-row]="'span ' + widget.height">
          
          <ng-container *ngComponentOutlet="
            getWidgetComponent(widget.type);
            inputs: { config: widget.config, data: widget.data$ | async }
          "></ng-container>
        </div>
      }
    </div>
  `
})
export class DashboardGridComponent implements OnInit {
  widgets: Widget[] = [];
  
  constructor(private widgetLoader: WidgetLoaderService) {}
  
  async ngOnInit() {
    this.widgets = await this.loadDashboardConfig();
  }
  
  getWidgetComponent(type: WidgetType) {
    return this.widgetLoader.loadWidget(type);
  }
}
```

**Performance Optimization with Virtual Scrolling:**
```typescript
@Component({
  selector: 'app-data-table',
  template: `
    <cdk-virtual-scroll-viewport itemSize="50" class="table-viewport">
      <table>
        <thead>
          <tr>
            <th *ngFor="let col of columns">{{ col.label }}</th>
          </tr>
        </thead>
        <tbody>
          <tr *cdkVirtualFor="let row of data; trackBy: trackByFn">
            <td *ngFor="let col of columns">{{ row[col.key] }}</td>
          </tr>
        </tbody>
      </table>
    </cdk-virtual-scroll-viewport>
  `
})
export class DataTableComponent {
  @Input() data: any[] = [];
  @Input() columns: Column[] = [];
  
  trackByFn(index: number, item: any) {
    return item.id;
  }
}
```

---

## 2. Low-Level Design Problems {#lld}

### LLD Problem 1: Autocomplete Component

**Requirements:**
- Debounced search
- Keyboard navigation (up/down/enter)
- Highlight matching text
- Loading state
- Empty state
- Custom templates

**Solution:**

```typescript
// autocomplete.component.ts
import { Component, Input, Output, EventEmitter, TemplateRef } from '@angular/core';
import { FormControl } from '@angular/forms';
import { debounceTime, distinctUntilChanged, switchMap, catchError } from 'rxjs/operators';
import { of } from 'rxjs';

interface AutocompleteItem {
  id: string | number;
  label: string;
  [key: string]: any;
}

@Component({
  selector: 'app-autocomplete',
  template: `
    <div class="autocomplete" (clickOutside)="closeDropdown()">
      <input
        [formControl]="searchControl"
        [placeholder]="placeholder"
        (keydown)="onKeyDown($event)"
        (focus)="onFocus()"
        autocomplete="off">
      
      @if (showDropdown) {
        <div class="dropdown">
          @if (loading) {
            <div class="loading">Loading...</div>
          } @else if (items.length === 0) {
            <div class="empty">{{ emptyMessage }}</div>
          } @else {
            <div 
              *ngFor="let item of items; let i = index"
              class="item"
              [class.highlighted]="i === highlightedIndex"
              (click)="selectItem(item)"
              (mouseenter)="highlightedIndex = i">
              
              <ng-container *ngIf="itemTemplate; else defaultTemplate">
                <ng-container *ngTemplateOutlet="itemTemplate; context: { $implicit: item }">
                </ng-container>
              </ng-container>
              
              <ng-template #defaultTemplate>
                <span [innerHTML]="highlightMatch(item.label, searchControl.value)"></span>
              </ng-template>
            </div>
          }
        </div>
      }
    </div>
  `,
  styles: [`
    .autocomplete {
      position: relative;
      width: 100%;
    }
    
    input {
      width: 100%;
      padding: 8px 12px;
      border: 1px solid #ccc;
      border-radius: 4px;
    }
    
    .dropdown {
      position: absolute;
      top: 100%;
      left: 0;
      right: 0;
      max-height: 300px;
      overflow-y: auto;
      background: white;
      border: 1px solid #ccc;
      border-top: none;
      border-radius: 0 0 4px 4px;
      box-shadow: 0 2px 4px rgba(0,0,0,0.1);
      z-index: 1000;
    }
    
    .item {
      padding: 8px 12px;
      cursor: pointer;
    }
    
    .item:hover,
    .item.highlighted {
      background-color: #f0f0f0;
    }
    
    .loading,
    .empty {
      padding: 8px 12px;
      color: #666;
    }
  `]
})
export class AutocompleteComponent implements OnInit {
  @Input() placeholder = 'Search...';
  @Input() emptyMessage = 'No results found';
  @Input() debounceTime = 300;
  @Input() minChars = 2;
  @Input() searchFn!: (term: string) => Observable<AutocompleteItem[]>;
  @Input() itemTemplate?: TemplateRef<any>;
  
  @Output() itemSelected = new EventEmitter<AutocompleteItem>();
  
  searchControl = new FormControl('');
  items: AutocompleteItem[] = [];
  loading = false;
  showDropdown = false;
  highlightedIndex = -1;
  
  ngOnInit() {
    this.searchControl.valueChanges.pipe(
      debounceTime(this.debounceTime),
      distinctUntilChanged(),
      switchMap(term => {
        if (!term || term.length < this.minChars) {
          this.items = [];
          this.showDropdown = false;
          return of([]);
        }
        
        this.loading = true;
        this.showDropdown = true;
        
        return this.searchFn(term).pipe(
          catchError(() => {
            this.loading = false;
            return of([]);
          })
        );
      })
    ).subscribe(items => {
      this.items = items;
      this.loading = false;
      this.highlightedIndex = -1;
    });
  }
  
  onKeyDown(event: KeyboardEvent) {
    if (!this.showDropdown || this.items.length === 0) return;
    
    switch (event.key) {
      case 'ArrowDown':
        event.preventDefault();
        this.highlightedIndex = Math.min(
          this.highlightedIndex + 1,
          this.items.length - 1
        );
        break;
        
      case 'ArrowUp':
        event.preventDefault();
        this.highlightedIndex = Math.max(this.highlightedIndex - 1, 0);
        break;
        
      case 'Enter':
        event.preventDefault();
        if (this.highlightedIndex >= 0) {
          this.selectItem(this.items[this.highlightedIndex]);
        }
        break;
        
      case 'Escape':
        this.closeDropdown();
        break;
    }
  }
  
  onFocus() {
    if (this.items.length > 0) {
      this.showDropdown = true;
    }
  }
  
  selectItem(item: AutocompleteItem) {
    this.searchControl.setValue(item.label, { emitEvent: false });
    this.itemSelected.emit(item);
    this.closeDropdown();
  }
  
  closeDropdown() {
    this.showDropdown = false;
    this.highlightedIndex = -1;
  }
  
  highlightMatch(text: string, term: string): string {
    if (!term) return text;
    
    const regex = new RegExp(`(${term})`, 'gi');
    return text.replace(regex, '<mark>$1</mark>');
  }
}

// Usage
@Component({
  template: `
    <app-autocomplete
      [searchFn]="searchUsers"
      (itemSelected)="onUserSelected($event)">
    </app-autocomplete>
    
    <!-- With custom template -->
    <app-autocomplete
      [searchFn]="searchUsers"
      [itemTemplate]="userTemplate"
      (itemSelected)="onUserSelected($event)">
    </app-autocomplete>
    
    <ng-template #userTemplate let-user>
      <div class="user-item">
        <img [src]="user.avatar" alt="">
        <div>
          <div class="name">{{ user.label }}</div>
          <div class="email">{{ user.email }}</div>
        </div>
      </div>
    </ng-template>
  `
})
export class AppComponent {
  searchUsers = (term: string) => {
    return this.userService.searchUsers(term);
  };
  
  onUserSelected(user: AutocompleteItem) {
    console.log('Selected:', user);
  }
}
```

---

### LLD Problem 2: Infinite Scroll Directive

**Requirements:**
- Trigger callback when scrolling near bottom
- Configurable threshold
- Loading state handling
- Cleanup on destroy

**Solution:**

```typescript
// infinite-scroll.directive.ts
import { Directive, Output, EventEmitter, Input, OnInit, OnDestroy, ElementRef } from '@angular/core';

@Directive({
  selector: '[appInfiniteScroll]',
  standalone: true
})
export class InfiniteScrollDirective implements OnInit, OnDestroy {
  @Input() scrollThreshold = 200; // pixels from bottom
  @Input() scrollContainer?: HTMLElement;
  @Input() disabled = false;
  
  @Output() scrolled = new EventEmitter<void>();
  
  private scrollListener?: () => void;
  
  constructor(private el: ElementRef) {}
  
  ngOnInit() {
    const container = this.scrollContainer || window;
    
    this.scrollListener = this.onScroll.bind(this);
    container.addEventListener('scroll', this.scrollListener, { passive: true });
  }
  
  ngOnDestroy() {
    if (this.scrollListener) {
      const container = this.scrollContainer || window;
      container.removeEventListener('scroll', this.scrollListener);
    }
  }
  
  private onScroll() {
    if (this.disabled) return;
    
    const element = this.el.nativeElement;
    const container = this.scrollContainer || document.documentElement;
    
    const scrollTop = container === document.documentElement 
      ? window.pageYOffset 
      : (container as HTMLElement).scrollTop;
      
    const scrollHeight = container === document.documentElement
      ? document.documentElement.scrollHeight
      : (container as HTMLElement).scrollHeight;
      
    const clientHeight = container === document.documentElement
      ? window.innerHeight
      : (container as HTMLElement).clientHeight;
    
    const distanceFromBottom = scrollHeight - (scrollTop + clientHeight);
    
    if (distanceFromBottom < this.scrollThreshold) {
      this.scrolled.emit();
    }
  }
}

// Usage
@Component({
  template: `
    <div 
      appInfiniteScroll
      [scrollThreshold]="300"
      [disabled]="loading || !hasMore"
      (scrolled)="loadMore()">
      
      @for (item of items; track item.id) {
        <div class="item">{{ item.name }}</div>
      }
      
      @if (loading) {
        <div class="loading">Loading...</div>
      }
      
      @if (!hasMore) {
        <div class="end">No more items</div>
      }
    </div>
  `
})
export class ListComponent {
  items: any[] = [];
  loading = false;
  hasMore = true;
  page = 1;
  
  loadMore() {
    if (this.loading || !this.hasMore) return;
    
    this.loading = true;
    this.dataService.getItems(this.page).subscribe({
      next: (response) => {
        this.items = [...this.items, ...response.items];
        this.hasMore = response.hasMore;
        this.page++;
        this.loading = false;
      },
      error: () => {
        this.loading = false;
      }
    });
  }
}
```

---

### LLD Problem 3: Toast Notification Service

**Requirements:**
- Multiple notification types (success, error, warning, info)
- Auto-dismiss with configurable duration
- Manual dismiss
- Stacking notifications
- Position configuration

**Solution:**

```typescript
// notification.model.ts
export interface Notification {
  id: string;
  type: 'success' | 'error' | 'warning' | 'info';
  message: string;
  duration?: number;
  dismissible?: boolean;
}

// notification.service.ts
@Injectable({ providedIn: 'root' })
export class NotificationService {
  private notificationsSubject = new BehaviorSubject<Notification[]>([]);
  public notifications$ = this.notificationsSubject.asObservable();
  
  private defaultDuration = 3000;
  
  show(notification: Omit<Notification, 'id'>) {
    const id = this.generateId();
    const newNotification: Notification = {
      id,
      dismissible: true,
      duration: this.defaultDuration,
      ...notification
    };
    
    const current = this.notificationsSubject.value;
    this.notificationsSubject.next([...current, newNotification]);
    
    if (newNotification.duration) {
      setTimeout(() => this.dismiss(id), newNotification.duration);
    }
  }
  
  success(message: string, duration?: number) {
    this.show({ type: 'success', message, duration });
  }
  
  error(message: string, duration?: number) {
    this.show({ type: 'error', message, duration });
  }
  
  warning(message: string, duration?: number) {
    this.show({ type: 'warning', message, duration });
  }
  
  info(message: string, duration?: number) {
    this.show({ type: 'info', message, duration });
  }
  
  dismiss(id: string) {
    const current = this.notificationsSubject.value;
    this.notificationsSubject.next(current.filter(n => n.id !== id));
  }
  
  clear() {
    this.notificationsSubject.next([]);
  }
  
  private generateId(): string {
    return `notification-${Date.now()}-${Math.random()}`;
  }
}

// notification-container.component.ts
@Component({
  selector: 'app-notification-container',
  standalone: true,
  template: `
    <div class="notification-container" [class]="position">
      @for (notification of notifications$ | async; track notification.id) {
        <div 
          class="notification"
          [class]="notification.type"
          @slideIn>
          
          <div class="icon">
            @switch (notification.type) {
              @case ('success') { ✓ }
              @case ('error') { ✕ }
              @case ('warning') { ⚠ }
              @case ('info') { ℹ }
            }
          </div>
          
          <div class="message">{{ notification.message }}</div>
          
          @if (notification.dismissible) {
            <button 
              class="close"
              (click)="dismiss(notification.id)">
              ✕
            </button>
          }
        </div>
      }
    </div>
  `,
  styles: [`
    .notification-container {
      position: fixed;
      z-index: 9999;
      display: flex;
      flex-direction: column;
      gap: 8px;
      padding: 16px;
    }
    
    .notification-container.top-right {
      top: 0;
      right: 0;
    }
    
    .notification {
      display: flex;
      align-items: center;
      gap: 12px;
      padding: 12px 16px;
      border-radius: 4px;
      background: white;
      box-shadow: 0 2px 8px rgba(0,0,0,0.15);
      min-width: 300px;
    }
    
    .notification.success {
      border-left: 4px solid #4caf50;
    }
    
    .notification.error {
      border-left: 4px solid #f44336;
    }
    
    .notification.warning {
      border-left: 4px solid #ff9800;
    }
    
    .notification.info {
      border-left: 4px solid #2196f3;
    }
    
    .close {
      margin-left: auto;
      background: none;
      border: none;
      cursor: pointer;
      font-size: 18px;
      opacity: 0.5;
    }
    
    .close:hover {
      opacity: 1;
    }
  `],
  animations: [
    trigger('slideIn', [
      transition(':enter', [
        style({ transform: 'translateX(100%)', opacity: 0 }),
        animate('200ms ease-out', style({ transform: 'translateX(0)', opacity: 1 }))
      ]),
      transition(':leave', [
        animate('200ms ease-in', style({ transform: 'translateX(100%)', opacity: 0 }))
      ])
    ])
  ]
})
export class NotificationContainerComponent {
  @Input() position: 'top-right' | 'top-left' | 'bottom-right' | 'bottom-left' = 'top-right';
  
  notifications$ = this.notificationService.notifications$;
  
  constructor(private notificationService: NotificationService) {}
  
  dismiss(id: string) {
    this.notificationService.dismiss(id);
  }
}

// Usage in app.component.ts
@Component({
  selector: 'app-root',
  template: `
    <router-outlet></router-outlet>
    <app-notification-container position="top-right"></app-notification-container>
  `
})
export class AppComponent {}

// Usage in any component
@Component({})
export class SomeComponent {
  constructor(private notification: NotificationService) {}
  
  save() {
    this.api.save().subscribe({
      next: () => {
        this.notification.success('Saved successfully!');
      },
      error: () => {
        this.notification.error('Failed to save');
      }
    });
  }
}
```

This completes Part 4. Let me now combine all parts into a single comprehensive file.

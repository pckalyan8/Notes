# Phase 10 — Angular Core Fundamentals
## Part 4: Templates & Data Binding
> Topic: 10.8

---

## 10.8 Templates & Data Binding

### What Is Data Binding?

Data binding is the mechanism that **synchronizes data between a component's TypeScript class and its HTML template**. Without data binding, you would manually manipulate the DOM to reflect state changes (like jQuery days) — a tedious, error-prone approach. Angular's binding syntax lets you declare relationships between your data and your UI, and Angular manages the updates.

There are four fundamental forms of data binding in Angular, each with distinct syntax and direction of data flow. Understanding which form to use in which situation is one of the most important skills in Angular development.

---

### 1. Interpolation: `{{ expression }}`

Interpolation is the simplest form of binding. Angular evaluates the expression between the double curly braces and renders its **string value** into the template. This is **one-way** — from the component class to the template.

```typescript
@Component({ standalone: true, selector: 'app-greeting', template: `
  <!-- Interpolation with a property -->
  <h1>Hello, {{ userName }}!</h1>

  <!-- Interpolation with a method call -->
  <p>Today is {{ getFormattedDate() }}</p>

  <!-- Interpolation with an expression -->
  <p>Total: {{ price * quantity }}</p>

  <!-- Interpolation with the ternary operator -->
  <p>Status: {{ isActive ? 'Active' : 'Inactive' }}</p>

  <!-- You can concatenate strings in interpolation -->
  <p>{{ firstName + ' ' + lastName }}</p>
` })
export class GreetingComponent {
  userName = 'Alice';
  price = 49.99;
  quantity = 3;
  isActive = true;
  firstName = 'John';
  lastName = 'Doe';

  getFormattedDate(): string {
    return new Date().toLocaleDateString();
  }
}
```

**What expressions are allowed?** Angular template expressions support most JavaScript expressions — arithmetic, ternary, string concatenation, method calls, and property access. But they **do not support** statements like assignments (`=`), `new`, `typeof`, and chaining operators that have side effects. Angular keeps expressions simple to prevent hidden complexity in templates.

**Important:** Interpolation always produces a string. Angular calls `.toString()` on the value. If you need to bind to a non-string property (like an image `src` that needs a URL, or `disabled` that needs a boolean), use **property binding** instead.

---

### 2. Property Binding: `[property]="expression"`

Property binding sets a **DOM property** (not an HTML attribute) to the value of a template expression. The square brackets tell Angular to evaluate the expression on the right-hand side and assign it to the property.

```typescript
@Component({ standalone: true, selector: 'app-image-display', template: `
  <!-- Property binding to DOM property 'src'. The src is dynamic. -->
  <img [src]="imageUrl" [alt]="imageAlt">

  <!-- Binding 'disabled' to a boolean. This would NOT work as interpolation. -->
  <button [disabled]="isSubmitting">Submit</button>

  <!-- Binding a class using NgClass (see 'class binding' below) -->
  <div [ngClass]="{ 'active': isActive, 'error': hasError }">Content</div>

  <!-- Binding a CSS property using NgStyle -->
  <div [ngStyle]="{ 'font-size': fontSize + 'px', 'color': textColor }">Text</div>

  <!-- Binding to a child component's @Input() -->
  <app-user-card [userId]="selectedUser.id" [title]="'Profile'" />
` })
export class ImageDisplayComponent {
  imageUrl = 'https://example.com/avatar.png';
  imageAlt = 'User avatar';
  isSubmitting = false;
  isActive = true;
  hasError = false;
  fontSize = 16;
  textColor = 'navy';
  selectedUser = { id: 42 };
}
```

#### The Critical Distinction: Attribute vs Property

This is a common point of confusion for developers coming from plain HTML. HTML **attributes** initialize DOM element state. DOM **properties** represent the *current* runtime state. After the page loads, the DOM works with properties, not attributes.

Angular's property binding targets **DOM properties**, not HTML attributes. Most of the time the names match (`src`, `href`, `disabled`), but some attributes have no direct matching property. For those cases, Angular provides **attribute binding**:

```html
<!-- When no corresponding DOM property exists, use attr. prefix -->
<!-- 'colspan' in a table cell is an attribute, not a DOM property -->
<td [attr.colspan]="dynamicColspan">Cell</td>

<!-- ARIA attributes are also attribute-only -->
<button [attr.aria-label]="buttonLabel">Click</button>

<!-- 'data-*' custom attributes use attr. binding -->
<div [attr.data-user-id]="userId">Content</div>
```

---

### 3. Class Binding

Class binding is a specialized form of property binding for adding and removing CSS classes based on component state.

```typescript
@Component({ standalone: true, selector: 'app-button', template: `
  <!-- Single class binding: adds 'active' class when isActive is truthy -->
  <button [class.active]="isActive">Click me</button>

  <!-- Add multiple classes conditionally using an object -->
  <!-- The key is the class name, the value is a boolean expression -->
  <div [class]="{ 'active': isActive, 'error': hasError, 'loading': isLoading }">
    Status Bar
  </div>

  <!-- Bind to a space-separated string (less common but valid) -->
  <div [class]="computedClasses">Content</div>
` })
export class ButtonComponent {
  isActive = true;
  hasError = false;
  isLoading = false;

  get computedClasses(): string {
    // Build class string programmatically
    const classes = ['base-class'];
    if (this.isActive) classes.push('active');
    if (this.hasError) classes.push('error');
    return classes.join(' ');
  }
}
```

---

### 4. Style Binding

Style binding applies inline styles based on component data, similarly to class binding.

```typescript
@Component({ standalone: true, selector: 'app-progress', template: `
  <!-- Single style binding. Value should include units where needed. -->
  <div [style.width.px]="progressWidth">Progress</div>

  <!-- Using string interpolation for the value with units included -->
  <div [style.color]="textColor">Colored Text</div>

  <!-- Binding an entire object of styles at once -->
  <div [style]="cardStyles">Card</div>
` })
export class ProgressComponent {
  progressWidth = 75;  // Applied as "75px" because of the .px suffix
  textColor = '#3498db';

  // Object-style binding applies all styles at once
  cardStyles = {
    'background-color': '#f0f0f0',
    'border-radius': '8px',
    'padding': '16px'
  };
}
```

---

### 5. Event Binding: `(event)="handler()"`

Event binding listens for DOM events and calls a method on the component when they fire. The parentheses indicate "something happening that Angular should listen to and react."

```typescript
@Component({ standalone: true, selector: 'app-counter', template: `
  <!-- Basic click event -->
  <button (click)="increment()">+</button>
  <span>{{ count }}</span>
  <button (click)="decrement()">-</button>

  <!-- Passing the event object with $event -->
  <input (input)="onInput($event)" placeholder="Type here">

  <!-- Keyboard events with key filtering -->
  <input (keyup.enter)="onEnterPressed()" placeholder="Press Enter">
  <input (keyup.escape)="onEscapePressed()">

  <!-- Mouse events -->
  <div (mouseenter)="onHoverStart()" (mouseleave)="onHoverEnd()">Hover Zone</div>

  <!-- Form events -->
  <form (submit)="onFormSubmit($event)">
    <button type="submit">Submit</button>
  </form>
` })
export class CounterComponent {
  count = 0;
  inputValue = '';

  increment(): void { this.count++; }
  decrement(): void { this.count--; }

  onInput(event: Event): void {
    // $event is the native DOM event object
    // Cast it to the specific event type for TypeScript safety
    this.inputValue = (event.target as HTMLInputElement).value;
  }

  onEnterPressed(): void { console.log('Enter pressed!'); }
  onEscapePressed(): void { console.log('Escape pressed!'); }

  onHoverStart(): void { console.log('Mouse entered'); }
  onHoverEnd(): void { console.log('Mouse left'); }

  onFormSubmit(event: Event): void {
    event.preventDefault();  // Stop the browser's native form submission
    console.log('Form submitted');
  }
}
```

**The `$event` variable** is a special Angular template variable that refers to the DOM event object fired by the event you're listening to. Its type depends on the event — a `click` event provides a `MouseEvent`, an `input` event provides an `InputEvent`, and a custom Angular output provides whatever type the `EventEmitter<T>` was typed with.

---

### 6. Two-Way Binding: `[(ngModel)]`

Two-way binding combines property binding and event binding into a single notation — nicknamed the "banana in a box" because `[()]` looks like a banana inside a box. It keeps the view and the model in sync automatically: when the user types in an input, the component property updates; when the component property changes, the input value updates.

```typescript
import { Component } from '@angular/core';
import { FormsModule } from '@angular/forms';  // Required for ngModel

@Component({ standalone: true, imports: [FormsModule], selector: 'app-name-form', template: `
  <!-- Two-way binding: typing in the input updates this.userName instantly -->
  <input [(ngModel)]="userName" placeholder="Enter your name">

  <!-- The value is immediately reflected below -->
  <p>Hello, {{ userName }}!</p>

  <!-- Two-way binding with a checkbox -->
  <label>
    <input type="checkbox" [(ngModel)]="isSubscribed">
    Subscribe to newsletter
  </label>
  <p>Subscribed: {{ isSubscribed }}</p>
` })
export class NameFormComponent {
  userName = '';
  isSubscribed = false;
}
```

**Under the hood**, `[(ngModel)]` expands to: `[ngModel]="userName" (ngModelChange)="userName = $event"`. It's syntactic sugar for the combination of a property binding (to display the value) and an event binding (to update the value when the user changes it).

**Note:** `[(ngModel)]` is part of Angular's **Template-Driven Forms** approach and requires `FormsModule`. For complex forms, Angular's **Reactive Forms** API is preferred (covered in Phase 11). Two-way binding with `model()` signals (covered in 10.6) is the modern approach for component-level two-way binding.

---

### 7. Template Reference Variables: `#ref`

A template reference variable gives you a **reference to a DOM element or component instance** directly within the template, so you can pass it to methods or access its properties.

```typescript
@Component({ standalone: true, selector: 'app-form-demo', template: `
  <!-- #emailInput is a reference to the native input element -->
  <input #emailInput type="email" placeholder="Your email">

  <!-- Pass the reference to a method — no TypeScript class property needed! -->
  <button (click)="logEmail(emailInput.value)">Log Email</button>

  <!-- Or use the reference directly in the template -->
  <p>You typed: {{ emailInput.value }}</p>

  <!-- Reference a component instance to call its methods -->
  <app-video-player #player></app-video-player>
  <button (click)="player.play()">Play</button>
  <button (click)="player.pause()">Pause</button>
` })
export class FormDemoComponent {
  logEmail(value: string): void {
    console.log('Email value:', value);
  }
}
```

Template reference variables are powerful because they let you wire up simple DOM interactions without writing TypeScript code at all. They exist only within the template scope — you cannot access them in the TypeScript class (use `ViewChild` for that).

---

### 8. The Safe Navigation Operator: `?.`

In templates, you frequently deal with data that might be `null` or `undefined` initially (while loading). The safe navigation operator (also called the "optional chaining" operator in Angular templates) prevents `Cannot read property of undefined` errors:

```typescript
@Component({ standalone: true, selector: 'app-user-detail', template: `
  <!-- Without safe navigation: throws error if user is null -->
  <!-- <p>{{ user.address.city }}</p> -->

  <!-- With safe navigation: renders nothing if user or address is null/undefined -->
  <p>{{ user?.address?.city }}</p>

  <!-- Safe navigation with method calls is also supported -->
  <p>{{ user?.getFullName() }}</p>

  <!-- Use with property bindings too -->
  <img [src]="user?.avatarUrl" [alt]="user?.name">
` })
export class UserDetailComponent {
  user: { address: { city: string }; avatarUrl: string; name: string } | null = null;

  ngOnInit() {
    // Simulate async data loading
    setTimeout(() => {
      this.user = { address: { city: 'Paris' }, avatarUrl: '...', name: 'Alice' };
    }, 1000);
  }
}
```

A modern, more readable alternative is to use Angular's `@if` control flow block to guard against null values, which also narrows the TypeScript type for the entire block:

```html
@if (user) {
  <!-- TypeScript knows 'user' is not null here -->
  <p>{{ user.address.city }}</p>
  <p>{{ user.name }}</p>
}
```

---

## Summary: Data Binding Quick Reference

The four forms of data binding correspond to four directions of communication, and it's worth having them fully memorized as a mental model:

```
One-way, component → template:
  Interpolation:      {{ expression }}
  Property binding:   [property]="expression"
  Attribute binding:  [attr.name]="expression"
  Class binding:      [class.name]="expression"
  Style binding:      [style.prop]="expression"

One-way, template → component:
  Event binding:      (event)="handler()"

Two-way:
  NgModel:            [(ngModel)]="property"
  Signal model:       [(model)]="property"  (child uses model())
```

---

## Best Practices for 10.8

**Choose the right binding form.** Interpolation is for displaying text content. Property binding is for dynamic HTML properties and inputs. Event binding is for reacting to user actions. Using the wrong form (like interpolation for `disabled`) leads to bugs that are hard to track down.

**Avoid calling methods in templates that perform heavy computation.** Angular's change detection calls template expressions repeatedly. A method like `getFilteredUsers()` in a template will be called on every change detection cycle. Use `computed()` signals or pure pipes for derived values instead — they are memoized.

**Keep template expressions simple.** Ideally, a template expression is a property access or a signal call. If you find yourself writing complex logic in a template, extract it into the component class (or a computed signal). Complex template expressions are hard to test, debug, and understand.

**Use `[(ngModel)]` only for template-driven forms.** For any form more complex than a single input field, use Reactive Forms with `FormControl` and `FormGroup`. They provide better type safety, testability, and validation control.

**Prefer `@if` over the safe navigation operator for null guarding.** `@if (user) { ... }` narrows the TypeScript type and produces cleaner templates than chaining `?.` everywhere.

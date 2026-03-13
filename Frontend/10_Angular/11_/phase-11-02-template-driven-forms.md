# Phase 11.2 — Forms: Template-Driven

## The Two Forms Approaches in Angular

Angular offers two distinct approaches to building forms: template-driven and reactive. Understanding both is important because real-world codebases contain both. Template-driven forms put most of the logic in the HTML template, relying on Angular directives to do the heavy lifting. Reactive forms put the logic in the TypeScript class, giving you full programmatic control. This file covers template-driven forms.

**When to choose template-driven forms:** Simple forms, data-entry forms with straightforward validation, scenarios where the template already drives the UI logic, and when you want minimal boilerplate. Think login forms, contact forms, newsletter signups.

---

## Setting Up `FormsModule`

Template-driven forms require `FormsModule` from `@angular/forms`. In standalone components, you import it directly.

```typescript
// login.component.ts
import { Component } from '@angular/core';
import { FormsModule } from '@angular/forms';

@Component({
  selector: 'app-login',
  standalone: true,
  imports: [FormsModule],  // Required for ngModel and form directives
  templateUrl: './login.component.html',
})
export class LoginComponent {
  email = '';
  password = '';

  onSubmit() {
    console.log('Login with:', this.email, this.password);
  }
}
```

---

## `ngModel` — Two-Way Data Binding

`ngModel` is the core directive of template-driven forms. It creates a **two-way binding** between a form control (input, select, textarea) and a TypeScript property. The `[( )]` banana-in-a-box syntax combines property binding `[ ]` (model → view) and event binding `( )` (view → model).

```html
<!-- login.component.html -->
<form (ngSubmit)="onSubmit()">

  <label for="email">Email</label>
  <!-- [(ngModel)] creates two-way binding with this.email -->
  <!-- name="email" is REQUIRED — it registers the control with the form -->
  <input
    id="email"
    type="email"
    name="email"
    [(ngModel)]="email"
    placeholder="Enter your email"
  />

  <label for="password">Password</label>
  <input
    id="password"
    type="password"
    name="password"
    [(ngModel)]="password"
  />

  <button type="submit">Login</button>
</form>
```

> **Critical point:** Every `ngModel` control **must** have a `name` attribute. This is how Angular registers the control with the parent form internally. Forgetting `name` causes a runtime error.

---

## Template Reference Variables and `ngForm`

Angular automatically creates an `NgForm` instance for every `<form>` element when `FormsModule` is imported. You can get a reference to it using a template reference variable.

```html
<!-- #loginForm="ngForm" gives us access to the form's state -->
<form #loginForm="ngForm" (ngSubmit)="onSubmit(loginForm)">

  <input
    id="email"
    type="email"
    name="email"
    [(ngModel)]="email"
    required
    email
  />

  <!-- Disable button until form is valid -->
  <button type="submit" [disabled]="loginForm.invalid">
    Login
  </button>

  <!-- Debugging: see the form's current state as JSON -->
  <pre>{{ loginForm.value | json }}</pre>
  <pre>Valid: {{ loginForm.valid }}</pre>
</form>
```

```typescript
import { NgForm } from '@angular/forms';

@Component({ ... })
export class LoginComponent {
  onSubmit(form: NgForm) {
    if (form.valid) {
      console.log('Form values:', form.value);
      // form.value = { email: '...', password: '...' }
    }
  }
}
```

---

## Built-In Validators

Angular provides a set of HTML5 validation attributes that automatically wire up form validation when `FormsModule` is active.

```html
<form #userForm="ngForm" (ngSubmit)="onSubmit()">

  <!-- required: field must not be empty -->
  <input name="username" [(ngModel)]="username" required />

  <!-- minlength / maxlength: string length bounds -->
  <input name="password" [(ngModel)]="password" required minlength="8" maxlength="32" />

  <!-- email: validates email format -->
  <input name="email" [(ngModel)]="email" required email />

  <!-- pattern: validates against a regex -->
  <input name="phone" [(ngModel)]="phone" pattern="[0-9]{10}" />

  <!-- min / max: for numeric inputs -->
  <input name="age" type="number" [(ngModel)]="age" min="18" max="99" />

</form>
```

---

## CSS State Classes — Visual Feedback

Angular automatically applies CSS classes to form controls based on their state. You can use these to style validation feedback.

| Class | Meaning |
|---|---|
| `ng-untouched` | User hasn't focused the field yet |
| `ng-touched` | User has focused and left the field |
| `ng-pristine` | Value hasn't changed since the form loaded |
| `ng-dirty` | Value has changed |
| `ng-valid` | All validators pass |
| `ng-invalid` | At least one validator fails |

```css
/* Show error state only after the user has interacted with the field */
input.ng-invalid.ng-touched {
  border-color: red;
  background-color: #fff0f0;
}

input.ng-valid.ng-dirty {
  border-color: green;
}
```

---

## Displaying Validation Errors

The best practice is to get a reference to the individual control using a template variable and then conditionally show error messages.

```html
<form #contactForm="ngForm" (ngSubmit)="onSubmit(contactForm)">

  <!-- Get a reference to this specific control with #nameCtrl="ngModel" -->
  <div class="field">
    <label for="name">Full Name</label>
    <input
      id="name"
      name="name"
      [(ngModel)]="name"
      required
      minlength="2"
      #nameCtrl="ngModel"
    />

    <!--
      Show errors only if:
      1. The field is invalid (nameCtrl.invalid)
      2. The user has touched it (nameCtrl.touched) — don't scold on page load
    -->
    <div class="errors" *ngIf="nameCtrl.invalid && nameCtrl.touched">
      <!-- Check specific error types -->
      <span *ngIf="nameCtrl.errors?.['required']">Name is required.</span>
      <span *ngIf="nameCtrl.errors?.['minlength']">
        Name must be at least {{ nameCtrl.errors?.['minlength'].requiredLength }} characters.
      </span>
    </div>
  </div>

  <div class="field">
    <label for="email">Email</label>
    <input id="email" name="email" [(ngModel)]="email" required email #emailCtrl="ngModel" />
    <div class="errors" *ngIf="emailCtrl.invalid && emailCtrl.touched">
      <span *ngIf="emailCtrl.errors?.['required']">Email is required.</span>
      <span *ngIf="emailCtrl.errors?.['email']">Please enter a valid email address.</span>
    </div>
  </div>

  <button type="submit" [disabled]="contactForm.invalid">Submit</button>

</form>
```

---

## Modern Control Flow Equivalent

If your component uses Angular 17+ modern control flow (`@if` instead of `*ngIf`), the pattern looks like this:

```html
<div class="field">
  <input name="email" [(ngModel)]="email" required email #emailCtrl="ngModel" />

  @if (emailCtrl.invalid && emailCtrl.touched) {
    <div class="errors">
      @if (emailCtrl.errors?.['required']) {
        <span>Email is required.</span>
      }
      @if (emailCtrl.errors?.['email']) {
        <span>Invalid email format.</span>
      }
    </div>
  }
</div>
```

---

## `datalist` and `autocomplete`

Template-driven forms work seamlessly with HTML5 `datalist` for autocomplete suggestions without a custom directive.

```html
<input
  name="country"
  [(ngModel)]="country"
  list="countries"
  autocomplete="off"
/>
<datalist id="countries">
  <option value="United States"></option>
  <option value="United Kingdom"></option>
  <option value="Canada"></option>
  <option value="Australia"></option>
</datalist>
```

---

## Select, Radio, and Checkbox Binding

### Select (Dropdown)

```html
<!-- Simple string binding -->
<select name="role" [(ngModel)]="selectedRole">
  <option value="">-- Select a role --</option>
  <option value="admin">Admin</option>
  <option value="editor">Editor</option>
  <option value="viewer">Viewer</option>
</select>

<!-- Dynamic options from an array -->
<select name="category" [(ngModel)]="selectedCategory">
  <option *ngFor="let cat of categories" [value]="cat.id">{{ cat.name }}</option>
</select>
```

### Radio Buttons

```html
<!-- Each radio shares the same name and binds to the same model property -->
<label>
  <input type="radio" name="plan" [(ngModel)]="selectedPlan" value="basic" /> Basic
</label>
<label>
  <input type="radio" name="plan" [(ngModel)]="selectedPlan" value="pro" /> Pro
</label>
<label>
  <input type="radio" name="plan" [(ngModel)]="selectedPlan" value="enterprise" /> Enterprise
</label>

<p>Selected plan: {{ selectedPlan }}</p>
```

### Checkbox (Boolean)

```html
<!-- Single boolean checkbox -->
<label>
  <input type="checkbox" name="agreed" [(ngModel)]="agreedToTerms" />
  I agree to the terms and conditions
</label>

<!-- Multiple checkboxes bound to an array requires manual handling -->
```

---

## Form Reset

```typescript
@Component({ ... })
export class RegistrationComponent {
  @ViewChild('regForm') regForm!: NgForm;

  onSubmit(form: NgForm) {
    if (form.valid) {
      this.userService.register(form.value).subscribe(() => {
        form.reset();  // Resets values AND form state (touched, dirty, etc.)
        // Or reset to specific values:
        // form.resetForm({ email: '', role: 'viewer' });
      });
    }
  }
}
```

---

## A Complete Registration Form Example

```typescript
// registration.component.ts
import { Component } from '@angular/core';
import { FormsModule, NgForm } from '@angular/forms';

interface RegistrationData {
  firstName: string;
  lastName: string;
  email: string;
  password: string;
  role: string;
  newsletter: boolean;
}

@Component({
  selector: 'app-registration',
  standalone: true,
  imports: [FormsModule],
  templateUrl: './registration.component.html',
})
export class RegistrationComponent {
  formData: RegistrationData = {
    firstName: '',
    lastName: '',
    email: '',
    password: '',
    role: 'viewer',
    newsletter: false,
  };

  submitted = false;

  onSubmit(form: NgForm) {
    if (form.valid) {
      this.submitted = true;
      console.log('Registering:', this.formData);
      // Call API here
    }
  }
}
```

```html
<!-- registration.component.html -->
<form #regForm="ngForm" (ngSubmit)="onSubmit(regForm)" novalidate>

  <div class="field">
    <label for="firstName">First Name *</label>
    <input id="firstName" name="firstName" [(ngModel)]="formData.firstName"
           required #firstNameCtrl="ngModel" />
    @if (firstNameCtrl.invalid && firstNameCtrl.touched) {
      <span class="error">First name is required.</span>
    }
  </div>

  <div class="field">
    <label for="email">Email *</label>
    <input id="email" type="email" name="email" [(ngModel)]="formData.email"
           required email #emailCtrl="ngModel" />
    @if (emailCtrl.invalid && emailCtrl.touched) {
      @if (emailCtrl.errors?.['required']) { <span class="error">Email is required.</span> }
      @if (emailCtrl.errors?.['email']) { <span class="error">Invalid email.</span> }
    }
  </div>

  <div class="field">
    <label for="role">Role</label>
    <select id="role" name="role" [(ngModel)]="formData.role">
      <option value="viewer">Viewer</option>
      <option value="editor">Editor</option>
      <option value="admin">Admin</option>
    </select>
  </div>

  <label>
    <input type="checkbox" name="newsletter" [(ngModel)]="formData.newsletter" />
    Subscribe to newsletter
  </label>

  <button type="submit" [disabled]="regForm.invalid">Register</button>
  <button type="button" (click)="regForm.reset()">Clear</button>

</form>

@if (submitted) {
  <p class="success">Registration successful!</p>
}
```

---

## Key Best Practices

Always use `novalidate` on your `<form>` element to disable browser-native validation bubbles and take full control with Angular's system. Show validation errors only after the user has touched a field — showing errors on an untouched form is poor UX. Always link `<label>` elements to inputs with `for` and `id` for accessibility. Use `ngModel` with object properties (`formData.email`) rather than individual component properties when the form has many fields — this makes serialising and resetting the form much easier. Template-driven forms are excellent for simple forms, but switch to reactive forms when you need dynamic form controls, cross-field validation, or complex async validation logic.

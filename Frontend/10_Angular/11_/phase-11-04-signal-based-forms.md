# Phase 11.4 — Forms: Signal-Based (Angular 20+ Experimental)

## Why Signal-Based Forms Were Created

To understand why Angular introduced yet another form approach, you need to understand what the existing approaches lack in a signals-first world.

Reactive forms are built on top of RxJS Observables. When your component is otherwise entirely signal-based (using `signal()`, `computed()`, `effect()`), reactive forms stick out like a foreign paradigm. You're forced to `.subscribe()` to `valueChanges`, manage subscriptions with `takeUntilDestroyed()`, and bridge the RxJS world to the signal world with `toSignal()`. It works, but the mental model is inconsistent.

Signal-based forms solve this by expressing the entire form's state — values, validation, errors, status — as signals. You get fine-grained reactivity, automatic change detection, and a uniform mental model that fits naturally into a zoneless, signal-driven component.

> **Important caveat:** As of Angular 20/21, signal-based forms are in **developer preview** (experimental). The API surface may change before stabilisation. Understand the patterns and architecture, but be cautious about adopting them in production without thorough evaluation. Reactive forms remain the stable, production-recommended choice for now.

---

## Core Primitives

### `signalFormControl()`

A `signalFormControl` is a single form field whose `.value` property is a readable signal. Instead of subscribing to `valueChanges`, you simply read the signal in a template or `computed()`.

```typescript
import { signalFormControl } from '@angular/forms';  // API is still stabilising

const emailControl = signalFormControl<string>('', {
  validators: [required(), email()],  // Signal-aware validator functions
});

// Reading the value — no subscription needed
emailControl.value();      // Signal: current string value
emailControl.valid();      // Signal: boolean
emailControl.errors();     // Signal: ValidationErrors | null
emailControl.touched();    // Signal: boolean
emailControl.dirty();      // Signal: boolean
```

The crucial difference is that `emailControl.value()` is a **signal read**, not a snapshot. Any `computed()` or template expression that calls it will automatically re-evaluate when the user changes the input.

### `signalFormGroup()`

A `signalFormGroup` is the signal equivalent of `FormGroup`. It groups controls together and exposes a composite value signal.

```typescript
import { signalFormGroup, signalFormControl } from '@angular/forms';

const loginForm = signalFormGroup({
  email: signalFormControl<string>('', { validators: [required(), email()] }),
  password: signalFormControl<string>('', { validators: [required(), minLength(8)] }),
});

// The value signal returns the full form object
loginForm.value();  // Signal: { email: string; password: string }
loginForm.valid();  // Signal: boolean — true only if ALL controls are valid
```

### `signalFormArray()`

`signalFormArray` manages a dynamic list of controls, mirroring `FormArray` but with signal semantics.

```typescript
const phoneNumbers = signalFormArray([
  signalFormControl<string>('+1-555-0001'),
  signalFormControl<string>('+1-555-0002'),
]);

phoneNumbers.value();   // Signal: string[]
phoneNumbers.length();  // Signal: number

// Dynamically add a control
phoneNumbers.push(signalFormControl<string>(''));

// Remove at index
phoneNumbers.removeAt(1);
```

### `signalForm()` — The Top-Level Entry Point

`signalForm()` is a convenience function that creates the root form container, similar to how `FormBuilder.group()` creates a `FormGroup`.

```typescript
import { signalForm } from '@angular/forms';

const registrationForm = signalForm({
  firstName: signalFormControl<string>('', { validators: [required()] }),
  lastName: signalFormControl<string>('', { validators: [required()] }),
  email: signalFormControl<string>('', { validators: [required(), email()] }),
  address: signalFormGroup({
    street: signalFormControl<string>(''),
    city: signalFormControl<string>('', { validators: [required()] }),
  }),
});
```

---

## Using Signal Forms in a Component

The component can treat the entire form as signal state. Templates read signals directly without async pipes or subscription management.

```typescript
import { Component } from '@angular/core';
import { signalForm, signalFormControl, required, email, minLength } from '@angular/forms';
import { ReactiveFormsModule } from '@angular/forms';

@Component({
  selector: 'app-signal-login',
  standalone: true,
  imports: [ReactiveFormsModule],  // Adapter layer still needed for template binding
  template: `
    <form (submit)="onSubmit($event)">

      <input [signalFormControl]="form.controls.email" type="email" />

      @if (form.controls.email.invalid() && form.controls.email.touched()) {
        <div class="error">
          @if (form.controls.email.errors()?.['required']) {
            <span>Email is required.</span>
          }
          @if (form.controls.email.errors()?.['email']) {
            <span>Enter a valid email.</span>
          }
        </div>
      }

      <input [signalFormControl]="form.controls.password" type="password" />

      <!-- The submit button reacts to the form's valid signal -->
      <button type="submit" [disabled]="!form.valid()">Login</button>

    </form>

    <!-- Computed values from signals work naturally -->
    <p>Form is {{ form.valid() ? 'valid' : 'invalid' }}</p>
    <p>Email value: {{ form.controls.email.value() }}</p>
  `,
})
export class SignalLoginComponent {
  form = signalForm({
    email: signalFormControl<string>('', { validators: [required(), email()] }),
    password: signalFormControl<string>('', { validators: [required(), minLength(8)] }),
  });

  onSubmit(event: Event) {
    event.preventDefault();
    if (this.form.valid()) {
      // All signal reads — no subscriptions
      const payload = this.form.value();
      console.log('Submitting:', payload);
    }
  }
}
```

The template reads `form.valid()`, `form.controls.email.invalid()`, and `form.controls.email.errors()` — all are signal reads that Angular tracks for fine-grained change detection.

---

## Reactive Validation — Computed Validators

One of the most powerful features of signal-based forms is that **validators can themselves read signals**. This means you can write validators whose logic changes in response to other signals in your component.

```typescript
@Component({ ... })
export class RegistrationComponent {
  // A signal that controls the validation rules
  isBusinessAccount = signal(false);

  form = signalForm({
    email: signalFormControl<string>('', {
      validators: [required(), email()],
    }),
    vatNumber: signalFormControl<string>('', {
      // This validator reads isBusinessAccount — it automatically re-runs
      // whenever isBusinessAccount changes!
      validators: computed(() =>
        this.isBusinessAccount() ? [required()] : []
      ),
    }),
  });
}
```

In reactive forms, implementing "required only if checkbox is checked" requires manual cross-field logic and re-triggering validation. In signal-based forms, `computed()` handles this naturally.

---

## Async Validators in Signal Forms

Async validators in signal forms work similarly to reactive forms, but their status is also expressed as a signal.

```typescript
const usernameControl = signalFormControl<string>('', {
  validators: [required()],
  asyncValidators: [uniqueUsernameAsyncValidator()],  // Returns Observable or Promise
  updateOn: 'blur',  // Only validate after focus loss
});

// In the template:
// usernameControl.status() — Signal: 'VALID' | 'INVALID' | 'PENDING' | 'DISABLED'
```

```html
@if (usernameControl.status() === 'PENDING') {
  <span>Checking availability...</span>
}
@if (usernameControl.errors()?.['usernameTaken']) {
  <span class="error">Username is already taken.</span>
}
```

---

## Two-Way Binding with `model()` Signals

Signal-based forms integrate with Angular's `model()` signal for parent-child synchronisation. When a parent component passes form state to a child via `model()`, changes propagate bidirectionally without event emitters.

```typescript
// parent.component.ts
@Component({
  template: `
    <app-address-fields [(address)]="form.controls.address.value" />
  `,
})
export class ParentComponent {
  form = signalForm({
    address: signalFormGroup({
      street: signalFormControl<string>(''),
      city: signalFormControl<string>(''),
    }),
  });
}

// child.component.ts
@Component({
  selector: 'app-address-fields',
  template: `...`,
})
export class AddressFieldsComponent {
  address = model.required<{ street: string; city: string }>();
}
```

---

## `ControlValueAccessor` Compatibility

Existing custom form controls built with `ControlValueAccessor` continue to work with signal-based forms — there is a compatibility adapter layer. This is a deliberate design decision to protect the existing ecosystem. You don't need to rewrite your custom controls to use signal-based forms.

```html
<!-- A custom ControlValueAccessor component works inside signal form binding -->
<app-star-rating [signalFormControl]="form.controls.rating"></app-star-rating>
```

---

## Comparing Signal Forms vs Reactive Forms

Understanding when to use each helps you make informed architectural decisions.

Signal-based forms excel when your component is entirely built on signals (zoneless, signal inputs/outputs), when you need computed validators that depend on other reactive state, and when you want to avoid the cognitive overhead of mixing RxJS and signals. The value reads are synchronous signal calls rather than Observable subscriptions, so the mental model stays consistent.

Reactive forms remain the right choice for all production code right now due to their stability and thorough documentation. They also have a richer ecosystem of third-party libraries and patterns built around them. When an existing codebase uses `FormGroup` and `FormArray` extensively, migrating everything to signal forms before the API stabilises carries unnecessary risk.

The pragmatic path is to understand signal-based forms as the future direction of Angular forms and monitor stabilisation. Build new projects with reactive forms today, while architecting them in a way that won't make migration painful — avoid leaking form internals across too many components.

---

## Migration Path: Reactive → Signal Forms

When signal forms stabilise, the migration is conceptual rather than mechanical. The key mappings are these:

`FormControl<T>` becomes `signalFormControl<T>()`. `FormGroup` becomes `signalFormGroup()`. `FormArray` becomes `signalFormArray()`. `control.valueChanges` subscriptions become `computed()` or `effect()` expressions that read `control.value()`. The template switches from `formControlName` to `[signalFormControl]`. Cross-field validators written as `FormGroup`-level validators become `computed()` validators that read other signals.

---

## Key Best Practices

Treat signal-based forms as a preview and learn the concepts thoroughly, but evaluate carefully before using them in any customer-facing application. If your team is experimenting with signal forms, isolate the experiment to a single feature or sub-form rather than the whole application. Always keep `ControlValueAccessor`-based custom controls compatible by not tightly coupling them to signal form internals. When writing computed validators, ensure the signals they read are scoped correctly to avoid memory leaks or unexpected re-evaluation cycles. Monitor Angular's official blog and the `CHANGELOG.md` of `@angular/forms` for stabilisation news before committing to this API in a long-running project.

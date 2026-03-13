# Phase 11.3 — Forms: Reactive

## Why Reactive Forms Exist

Reactive forms take a fundamentally different philosophical stance from template-driven forms. Where template-driven forms put form logic in the template and let Angular's directives manage state behind the scenes, reactive forms put the entire form model in your TypeScript class. This makes the form fully **explicit, synchronous, and testable** — you can inspect, manipulate, and validate the form structure in your component code without touching the DOM at all.

The key insight is that reactive forms give you a **reactive stream of form values**. Every time the user types something, the form emits a new value through an Observable, which means you can compose powerful async patterns (like debounced search, cross-field validation, and dependent field logic) using RxJS.

**When to choose reactive forms:** Complex forms with dynamic controls, forms requiring cross-field validation, forms with async validation (checking username availability against an API), wizard-style multi-step forms, and any scenario where you need to programmatically add or remove controls at runtime.

---

## Setting Up Reactive Forms

Reactive forms require `ReactiveFormsModule` from `@angular/forms`.

```typescript
import { Component } from '@angular/core';
import { ReactiveFormsModule, FormControl, FormGroup, Validators } from '@angular/forms';

@Component({
  selector: 'app-login',
  standalone: true,
  imports: [ReactiveFormsModule],
  templateUrl: './login.component.html',
})
export class LoginComponent {
  // Define the form model in the class — this is the key difference from template-driven
  loginForm = new FormGroup({
    email: new FormControl('', [Validators.required, Validators.email]),
    password: new FormControl('', [Validators.required, Validators.minLength(8)]),
  });

  onSubmit() {
    if (this.loginForm.valid) {
      console.log(this.loginForm.value); // { email: '...', password: '...' }
    }
  }
}
```

```html
<!-- Bind the FormGroup to the form element, FormControls to inputs -->
<form [formGroup]="loginForm" (ngSubmit)="onSubmit()">

  <input formControlName="email" type="email" placeholder="Email" />

  <input formControlName="password" type="password" placeholder="Password" />

  <button type="submit" [disabled]="loginForm.invalid">Login</button>

</form>
```

The connection between template and model is entirely explicit: `[formGroup]="loginForm"` connects the form, and `formControlName="email"` connects individual inputs by matching the key name in the `FormGroup`.

---

## The Core Building Blocks

### `FormControl`

`FormControl` represents a single form input. It tracks the input's value, validation state, and user interaction state.

```typescript
// A FormControl with an initial value and validators
const emailControl = new FormControl('', [
  Validators.required,
  Validators.email,
]);

// Reading state programmatically
emailControl.value;         // Current value (string | null)
emailControl.valid;         // true if all validators pass
emailControl.invalid;       // true if any validator fails
emailControl.touched;       // true if user focused and left the field
emailControl.dirty;         // true if the value has changed
emailControl.errors;        // null or { required: true, email: true }

// Updating programmatically
emailControl.setValue('new@example.com');   // Sets value, triggers validation
emailControl.patchValue('partial@...');     // Same for single controls
emailControl.reset();                       // Resets to initial value + pristine state
```

### `FormGroup`

`FormGroup` groups related `FormControl`s together into a single object. Its value is always an object matching the shape of its controls.

```typescript
const addressForm = new FormGroup({
  street: new FormControl('', Validators.required),
  city: new FormControl('', Validators.required),
  state: new FormControl(''),
  zip: new FormControl('', Validators.pattern(/^\d{5}$/)),
});

// addressForm.value = { street: '', city: '', state: '', zip: '' }
addressForm.get('city')?.setValue('New York');   // Access a child control
addressForm.get('city')?.value;                  // 'New York'
```

### `FormArray`

`FormArray` manages a dynamic, ordered list of controls. It's perfect for "add another phone number" or "add another team member" patterns.

```typescript
const skillsForm = new FormGroup({
  name: new FormControl(''),
  skills: new FormArray([
    new FormControl('JavaScript'),
    new FormControl('TypeScript'),
  ]),
});

// Access the FormArray
const skills = skillsForm.get('skills') as FormArray;

// Add a control dynamically
skills.push(new FormControl('Angular'));

// Remove a control
skills.removeAt(1);

// skills.value = ['JavaScript', 'Angular']
```

---

## Using `FormBuilder` — Less Boilerplate

`FormBuilder` is a service that provides shorthand methods for creating reactive form models. It produces the exact same objects as `new FormControl()` / `new FormGroup()`, just with less verbosity.

```typescript
import { Component, inject } from '@angular/core';
import { FormBuilder, ReactiveFormsModule, Validators } from '@angular/forms';

@Component({ standalone: true, imports: [ReactiveFormsModule], ... })
export class RegisterComponent {
  private fb = inject(FormBuilder);

  // Equivalent to manually creating FormGroup and FormControls
  registerForm = this.fb.group({
    firstName: ['', [Validators.required, Validators.minLength(2)]],
    lastName: ['', Validators.required],
    email: ['', [Validators.required, Validators.email]],
    address: this.fb.group({           // Nested FormGroup
      street: [''],
      city: ['', Validators.required],
    }),
    skills: this.fb.array([           // FormArray
      this.fb.control(''),
    ]),
  });

  // Getter for the skills FormArray (makes template access cleaner)
  get skills() {
    return this.registerForm.get('skills') as FormArray;
  }

  addSkill() {
    this.skills.push(this.fb.control(''));
  }

  removeSkill(index: number) {
    this.skills.removeAt(index);
  }
}
```

---

## Typed Reactive Forms (Angular 14+)

A major upgrade introduced in Angular 14 was **typed reactive forms**. Before Angular 14, `form.value` returned `any`, which eliminated all TypeScript benefits. Now, form controls are fully generic, and TypeScript enforces correct value types throughout.

```typescript
import { FormControl, FormGroup } from '@angular/forms';

// TypeScript knows exactly what type each control holds
const profileForm = new FormGroup({
  name: new FormControl<string>('', { nonNullable: true }),  // never null
  age: new FormControl<number | null>(null),
  bio: new FormControl<string | null>(null),
});

// profileForm.value type is inferred as:
// { name: string; age: number | null; bio: string | null }
profileForm.value.name;  // TypeScript knows this is 'string'
profileForm.value.age;   // TypeScript knows this is 'number | null'

// The nonNullable option ensures reset() goes back to the initial value
// rather than setting the control to null
const nonNullForm = new FormGroup({
  username: new FormControl('', { nonNullable: true }),
  // Without nonNullable, reset() would set username to null
  // With nonNullable, reset() sets username back to ''
});
```

---

## Built-In Validators

`Validators` from `@angular/forms` provides all the standard validation functions.

```typescript
import { Validators } from '@angular/forms';

// Most common validators:
Validators.required          // Field must not be empty
Validators.requiredTrue      // Checkbox must be checked (boolean true)
Validators.email             // Must match email pattern
Validators.minLength(8)      // String must be at least 8 characters
Validators.maxLength(100)    // String must be at most 100 characters
Validators.min(0)            // Numeric value must be >= 0
Validators.max(999)          // Numeric value must be <= 999
Validators.pattern(/^\d+$/) // Must match this regex
Validators.nullValidator     // Always passes (useful as placeholder)

// Pass multiple validators as an array — all must pass
new FormControl('', [Validators.required, Validators.email, Validators.maxLength(200)])
```

---

## Displaying Validation Errors in Templates

```html
<!-- profile.component.html -->
<form [formGroup]="registerForm" (ngSubmit)="onSubmit()">

  <div class="field">
    <label for="firstName">First Name *</label>
    <input id="firstName" formControlName="firstName" />

    <!--
      Access the control's state directly via the form group.
      Show errors only after the user has interacted (touched).
    -->
    @if (registerForm.get('firstName')?.invalid && registerForm.get('firstName')?.touched) {
      <div class="errors">
        @if (registerForm.get('firstName')?.errors?.['required']) {
          <span>First name is required.</span>
        }
        @if (registerForm.get('firstName')?.errors?.['minlength']) {
          <span>
            Minimum {{registerForm.get('firstName')?.errors?.['minlength'].requiredLength}} characters required.
          </span>
        }
      </div>
    }
  </div>

</form>
```

A cleaner pattern is to create getter properties for frequently-accessed controls, avoiding the verbose `.get('controlName')` in templates.

```typescript
// In the component class, expose getters:
get firstName() { return this.registerForm.get('firstName'); }
get email() { return this.registerForm.get('email'); }
```

```html
<!-- Now the template is much cleaner: -->
@if (firstName?.invalid && firstName?.touched) {
  <span>{{ firstName?.errors?.['required'] ? 'Required' : 'Invalid name' }}</span>
}
```

---

## Custom Validators

When built-in validators aren't enough, you write custom validator functions. A validator is simply a function that takes an `AbstractControl` and returns either `null` (valid) or a `ValidationErrors` object (invalid).

### Synchronous Custom Validator

```typescript
import { AbstractControl, ValidationErrors, ValidatorFn } from '@angular/forms';

// A validator function — optionally parameterised
export function noSpacesValidator(): ValidatorFn {
  return (control: AbstractControl): ValidationErrors | null => {
    const hasSpaces = (control.value as string)?.includes(' ');
    // Return null if valid, or an error object if invalid
    return hasSpaces ? { noSpaces: { value: control.value } } : null;
  };
}

// A validator for a specific domain (e.g., no numbers in a name field)
export function alphabeticOnly(control: AbstractControl): ValidationErrors | null {
  const isAlphabetic = /^[a-zA-Z\s]*$/.test(control.value);
  return isAlphabetic ? null : { alphabeticOnly: true };
}

// Usage:
const usernameControl = new FormControl('', [
  Validators.required,
  noSpacesValidator(),
  alphabeticOnly,
]);
```

---

## Cross-Field (Group-Level) Validation

Cross-field validators are applied to a `FormGroup` rather than an individual control. This lets you compare or correlate values across multiple controls.

```typescript
// passwords-match.validator.ts
export function passwordsMatchValidator(group: AbstractControl): ValidationErrors | null {
  const password = group.get('password')?.value;
  const confirmPassword = group.get('confirmPassword')?.value;

  if (password !== confirmPassword) {
    // Return error on the group itself
    return { passwordsMismatch: true };
  }
  return null;
}

// In the component:
this.registerForm = this.fb.group(
  {
    password: ['', [Validators.required, Validators.minLength(8)]],
    confirmPassword: ['', Validators.required],
  },
  { validators: passwordsMatchValidator }  // Applied to the GROUP
);
```

```html
<!-- Template: check error on the group, not a single control -->
@if (registerForm.errors?.['passwordsMismatch'] && registerForm.get('confirmPassword')?.dirty) {
  <span class="error">Passwords do not match.</span>
}
```

---

## Asynchronous Validators

Async validators are for validations that require a network call — like checking if a username is already taken. They return an `Observable` or `Promise` of `ValidationErrors | null`.

```typescript
import { AbstractControl, AsyncValidatorFn } from '@angular/forms';
import { Observable, of } from 'rxjs';
import { map, debounceTime, switchMap, first } from 'rxjs/operators';
import { inject } from '@angular/core';
import { UserService } from './user.service';

export function uniqueUsernameValidator(): AsyncValidatorFn {
  const userService = inject(UserService);

  return (control: AbstractControl): Observable<ValidationErrors | null> => {
    if (!control.value) return of(null);  // Skip empty values

    return of(control.value).pipe(
      debounceTime(400),              // Wait for the user to stop typing
      switchMap(username =>           // Cancel in-flight requests on new keystrokes
        userService.checkUsernameAvailable(username)
      ),
      map(isAvailable => isAvailable ? null : { usernameTaken: true }),
      first()                         // Complete the Observable for FormControl to resolve
    );
  };
}

// Usage: async validators go in the THIRD argument
const usernameControl = new FormControl(
  '',
  [Validators.required],            // Sync validators (2nd arg)
  [uniqueUsernameValidator()]        // Async validators (3rd arg)
);
```

```html
<!-- Show a "checking..." state while async validation is pending -->
@if (usernameControl.pending) {
  <span>Checking availability...</span>
}
@if (usernameControl.errors?.['usernameTaken']) {
  <span class="error">This username is already taken.</span>
}
```

---

## Dynamic Form Controls with `FormArray`

A `FormArray` lets you add and remove controls at runtime — perfect for forms where users can add multiple items.

```typescript
@Component({
  standalone: true,
  imports: [ReactiveFormsModule],
  template: `
    <form [formGroup]="projectForm">
      <input formControlName="projectName" placeholder="Project name" />

      <div formArrayName="members">
        @for (member of members.controls; track $index; let i = $index) {
          <div [formGroupName]="i">
            <input formControlName="name" placeholder="Member name" />
            <input formControlName="role" placeholder="Role" />
            <button type="button" (click)="removeMember(i)">Remove</button>
          </div>
        }
      </div>

      <button type="button" (click)="addMember()">+ Add Member</button>
    </form>
  `,
})
export class ProjectFormComponent {
  private fb = inject(FormBuilder);

  projectForm = this.fb.group({
    projectName: ['', Validators.required],
    members: this.fb.array([this.createMember()]),  // Start with one member
  });

  get members() {
    return this.projectForm.get('members') as FormArray;
  }

  createMember() {
    return this.fb.group({
      name: ['', Validators.required],
      role: ['', Validators.required],
    });
  }

  addMember() {
    this.members.push(this.createMember());
  }

  removeMember(index: number) {
    this.members.removeAt(index);
  }
}
```

---

## Reacting to Value Changes with RxJS

Because reactive form controls expose `valueChanges` as an Observable, you can build sophisticated reactive behaviours.

```typescript
@Component({ ... })
export class SearchComponent implements OnInit {
  private destroyRef = inject(DestroyRef);

  searchForm = this.fb.group({ query: [''], category: ['all'] });

  ngOnInit() {
    // Debounced search-as-you-type
    this.searchForm.get('query')!.valueChanges.pipe(
      debounceTime(300),                      // Wait 300ms after last keystroke
      distinctUntilChanged(),                 // Only emit if value actually changed
      switchMap(query => this.searchService.search(query)),  // Cancel previous request
      takeUntilDestroyed(this.destroyRef),    // Auto-unsubscribe on component destroy
    ).subscribe(results => {
      this.results = results;
    });

    // React to the entire form changing
    this.searchForm.valueChanges.pipe(
      takeUntilDestroyed(this.destroyRef)
    ).subscribe(value => {
      // Save form state to URL params, localStorage, etc.
      console.log('Form changed:', value);
    });
  }
}
```

---

## `updateOn` — Controlling When Validation Runs

By default, Angular validates after every keystroke (`change`). You can change this for performance or better UX.

```typescript
// Validate only when the user tabs out (blur) — better for expensive validators
const emailControl = new FormControl('', {
  validators: [Validators.required, Validators.email],
  updateOn: 'blur',  // 'change' | 'blur' | 'submit'
});

// Set at the FormGroup level — applies to all children
const form = new FormGroup({
  email: new FormControl(''),
  password: new FormControl(''),
}, { updateOn: 'submit' });  // Only validate on form submission
```

---

## Patching and Setting Values

```typescript
// setValue() — sets ALL controls; throws if a key is missing
this.userForm.setValue({
  firstName: 'John',
  lastName: 'Doe',
  email: 'john@example.com',
  // Must provide ALL keys or TypeScript/Angular throws an error
});

// patchValue() — sets ONLY the provided controls; safe for partial updates
this.userForm.patchValue({
  firstName: 'Jane',
  // Other controls remain unchanged
});

// Resetting the form — resets values AND clears touched/dirty state
this.userForm.reset();

// Reset to specific values (combine reset with new values)
this.userForm.reset({ firstName: '', email: '' });
```

---

## `ControlValueAccessor` — Custom Form Controls

`ControlValueAccessor` is the interface that lets you create custom form field components that integrate seamlessly with both reactive and template-driven forms. Think of a star rating input, a colour picker, or a rich-text editor.

```typescript
// star-rating.component.ts
import { Component, forwardRef } from '@angular/core';
import { ControlValueAccessor, NG_VALUE_ACCESSOR } from '@angular/forms';

@Component({
  selector: 'app-star-rating',
  standalone: true,
  template: `
    @for (star of [1, 2, 3, 4, 5]; track star) {
      <span
        [class.filled]="star <= value"
        (click)="onStarClick(star)"
      >★</span>
    }
  `,
  providers: [
    {
      provide: NG_VALUE_ACCESSOR,
      useExisting: forwardRef(() => StarRatingComponent),
      multi: true,
    },
  ],
})
export class StarRatingComponent implements ControlValueAccessor {
  value = 0;
  private onChange: (val: number) => void = () => {};
  private onTouched: () => void = () => {};

  // Called by Angular to write an external value INTO the component
  writeValue(value: number): void {
    this.value = value;
  }

  // Register Angular's callback for when our value changes
  registerOnChange(fn: (val: number) => void): void {
    this.onChange = fn;
  }

  // Register Angular's callback for when our component is touched
  registerOnTouched(fn: () => void): void {
    this.onTouched = fn;
  }

  onStarClick(star: number) {
    this.value = star;
    this.onChange(star);   // Notify Angular of the new value
    this.onTouched();      // Mark as touched
  }
}

// Usage in a reactive form — works exactly like a native input
// <app-star-rating formControlName="rating"></app-star-rating>
```

---

## `FormRecord` — Dynamic Key-Value Forms

When you need a form with a dynamic, string-keyed set of controls (like user preferences), `FormRecord` provides a type-safe alternative to a plain `FormGroup` with unknown keys.

```typescript
import { FormRecord, FormControl } from '@angular/forms';

// A form where keys are dynamic strings but all values are booleans
const permissionsForm = new FormRecord({
  read: new FormControl(true),
  write: new FormControl(false),
});

// Add/remove controls dynamically
permissionsForm.addControl('delete', new FormControl(false));
permissionsForm.removeControl('write');
```

---

## Key Best Practices

Define your form model in `ngOnInit` or as a class property if it doesn't depend on external data, keeping the constructor clean. Always use `nonNullable: true` for controls that should never become `null` — this prevents confusing type errors. Create getter properties for controls you reference multiple times in the template to avoid verbose `.get('controlName')` chains. Apply async validators only on critical fields and use `debounceTime` inside the validator to avoid hammering your API on every keystroke. When an entire form is submitted, validate all controls and mark them as touched so users see all errors at once. Use `this.form.markAllAsTouched()` before checking validity on submission. Prefer `FormBuilder` in most cases — it reduces boilerplate substantially while producing the exact same underlying objects.

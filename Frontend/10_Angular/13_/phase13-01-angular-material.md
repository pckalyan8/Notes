# Phase 13.1 — Angular Material
## Complete Guide: Components, Theming & Best Practices

---

## What Is Angular Material?

Angular Material is the **official component library for Angular**, built by the Angular team at Google. It implements Google's **Material Design** specification — a design language that defines how components should look, feel, and behave. Think of it as a professionally designed, accessible, and well-tested toolkit of UI components you can drop into any Angular application without building from scratch.

It is tightly coupled with the **Angular CDK (Component Dev Kit)**, which provides the unstyled, behavioral infrastructure that Material's components sit on top of. This means when you use a Material dialog, it's actually a CDK overlay with Material styling applied.

> **Key insight:** Angular Material is not just a CSS library. Each component comes with **accessibility (ARIA) support**, **keyboard navigation**, **animation**, and **theming** built in. You get far more than visual polish.

---

## Setup & Installation

```bash
ng add @angular/material
```

This command does several things automatically:
- Installs `@angular/material` and `@angular/cdk`
- Adds a pre-built theme to `angular.json`
- Imports `BrowserAnimationsModule` (or `provideAnimations()`)
- Optionally adds Material typography globally

For standalone applications (Angular 17+), the setup uses providers:

```typescript
// main.ts
import { bootstrapApplication } from '@angular/platform-browser';
import { provideAnimationsAsync } from '@angular/platform-browser/animations/async';
import { AppComponent } from './app/app.component';

bootstrapApplication(AppComponent, {
  providers: [
    provideAnimationsAsync(), // async animations — loads only when needed
  ]
});
```

---

## Material Design System: M2 vs M3

Angular Material supports two generations of Material Design. Understanding the difference is critical because they use **different theming APIs**.

### Material 2 (M2) — Legacy but stable
- The original Material Design spec used in Material 6–14
- Token-based via Sass variables (e.g., `$primary`, `$accent`, `$warn`)
- Uses palettes from Material Design color system
- Still fully supported but considered legacy

### Material 3 (M3) — Current and recommended
- Introduced in Angular Material 15+ and fully supported from Angular 17+
- Uses **design tokens** (CSS custom properties) for every color, shape, and typography decision
- Supports **dynamic color** (colors generated from a seed color)
- Better dark mode support out of the box
- More flexible theming compared to M2

> **Best Practice:** If you're starting a new project in 2025/2026, use M3. If you're maintaining an older project, you can migrate incrementally.

---

## Theming System In Depth

Theming is one of the most important — and most misunderstood — aspects of Angular Material. Let me walk through it carefully.

### How M3 Theming Works

Angular Material's M3 theme is defined using SCSS mixins. A theme is a collection of **color**, **typography**, and **density** configurations.

```scss
// styles.scss
@use '@angular/material' as mat;

// Step 1: Include core styles (resets, ripple, etc.) — call once globally
@include mat.core();

// Step 2: Define your theme
$my-theme: mat.define-theme((
  color: (
    theme-type: light,          // 'light' or 'dark'
    primary: mat.$azure-palette,  // M3 built-in palettes
    tertiary: mat.$blue-palette,
  ),
  typography: (
    brand-family: 'Roboto, sans-serif',
    bold-weight: 900,
  ),
  density: (
    scale: 0,  // 0 is default; negative values = more compact
  )
));

// Step 3: Apply the theme to your root element
html {
  @include mat.all-component-themes($my-theme);
}
```

### Multi-Theme Support (Dark Mode)

You can define multiple themes and switch between them:

```scss
// Define both themes
$light-theme: mat.define-theme((
  color: (theme-type: light, primary: mat.$violet-palette)
));

$dark-theme: mat.define-theme((
  color: (theme-type: dark, primary: mat.$violet-palette)
));

html {
  // Apply light theme by default
  @include mat.all-component-themes($light-theme);
}

// Apply dark theme when system prefers dark
@media (prefers-color-scheme: dark) {
  html {
    @include mat.all-component-colors($dark-theme);
    // Only re-apply colors — typography/density stay the same
  }
}

// OR: apply dark theme via a CSS class for user-toggle
.dark-mode {
  @include mat.all-component-colors($dark-theme);
}
```

### Component-Level Theme Override

You don't have to apply the same theme everywhere. You can scope a different theme to a specific component or section:

```scss
.admin-panel {
  // Apply a different, denser theme for the admin section
  $admin-theme: mat.define-theme((
    color: (theme-type: light, primary: mat.$red-palette),
    density: (scale: -2),
  ));
  @include mat.all-component-themes($admin-theme);
}
```

---

## Typography System

Material's typography system defines text styles used across components. M3 uses a role-based typography system with roles like `display-large`, `headline-medium`, `body-small`, `label-large`, etc.

```scss
$my-theme: mat.define-theme((
  typography: (
    brand-family: 'Inter, sans-serif',    // Used for headlines
    plain-family: 'Roboto, sans-serif',   // Used for body text
    bold-weight: 700,
    medium-weight: 500,
    regular-weight: 400,
  )
));
```

You can also apply typography styles directly to your own elements:

```scss
.page-title {
  @include mat.m2-typography-level($my-theme, 'headline-large');
}
```

---

## Core Components: A Detailed Tour

### Buttons

Material provides several button variants, each serving a different visual hierarchy:

```html
<!-- Filled button (highest emphasis) -->
<button mat-flat-button color="primary">Save</button>

<!-- Outlined button (medium emphasis) -->
<button mat-stroked-button color="primary">Cancel</button>

<!-- Text button (lowest emphasis) -->
<button mat-button color="primary">Learn More</button>

<!-- Icon button (no label) -->
<button mat-icon-button aria-label="Delete">
  <mat-icon>delete</mat-icon>
</button>

<!-- FAB (Floating Action Button) -->
<button mat-fab extended color="primary">
  <mat-icon>add</mat-icon>
  Add Item
</button>
```

> **Important:** Always import `MatButtonModule` (or just `MatButton` standalone) in your component's imports array.

```typescript
@Component({
  imports: [MatButton, MatIconButton, MatIcon],
  // ...
})
```

### Form Fields & Inputs

The `mat-form-field` is a container that applies consistent styling, labels, hints, and error messages around form controls:

```html
<mat-form-field appearance="outline">
  <mat-label>Email Address</mat-label>
  <input matInput type="email" [formControl]="emailControl" placeholder="you@example.com">
  <mat-hint>We'll never share your email</mat-hint>
  <mat-error *ngIf="emailControl.hasError('required')">Email is required</mat-error>
  <mat-error *ngIf="emailControl.hasError('email')">Enter a valid email</mat-error>
  <mat-icon matSuffix>email</mat-icon>
</mat-form-field>
```

The `appearance` attribute controls the visual style:
- `outline` — border all around (most common)
- `fill` — filled background with underline
- `legacy` — classic Material 1 style (deprecated)

### Select & Autocomplete

```html
<!-- Select -->
<mat-form-field appearance="outline">
  <mat-label>Country</mat-label>
  <mat-select [formControl]="countryControl">
    <mat-option value="us">United States</mat-option>
    <mat-option value="uk">United Kingdom</mat-option>
    @for (country of countries; track country.code) {
      <mat-option [value]="country.code">{{ country.name }}</mat-option>
    }
  </mat-select>
</mat-form-field>

<!-- Autocomplete -->
<mat-form-field appearance="outline">
  <mat-label>City</mat-label>
  <input matInput [formControl]="cityControl" [matAutocomplete]="cityAuto">
  <mat-autocomplete #cityAuto="matAutocomplete">
    @for (city of filteredCities(); track city) {
      <mat-option [value]="city">{{ city }}</mat-option>
    }
  </mat-autocomplete>
</mat-form-field>
```

### Dialog

Dialogs are opened programmatically via the `MatDialog` service:

```typescript
import { MatDialog } from '@angular/material/dialog';
import { inject } from '@angular/core';

@Component({ /* ... */ })
export class ParentComponent {
  private dialog = inject(MatDialog);

  openConfirmDialog() {
    const dialogRef = this.dialog.open(ConfirmDialogComponent, {
      width: '400px',
      data: { message: 'Are you sure you want to delete this item?' },
      disableClose: true,  // prevent closing on backdrop click
    });

    // Subscribe to the result when dialog closes
    dialogRef.afterClosed().subscribe(result => {
      if (result === 'confirm') {
        this.deleteItem();
      }
    });
  }
}
```

```typescript
// The dialog component itself
@Component({
  template: `
    <h2 mat-dialog-title>Confirm Action</h2>
    <mat-dialog-content>{{ data.message }}</mat-dialog-content>
    <mat-dialog-actions align="end">
      <button mat-button mat-dialog-close>Cancel</button>
      <button mat-flat-button color="warn" [mat-dialog-close]="'confirm'">Delete</button>
    </mat-dialog-actions>
  `,
  imports: [MatDialogModule, MatButton]
})
export class ConfirmDialogComponent {
  data = inject(MAT_DIALOG_DATA);
}
```

### Snackbar (Toast Notifications)

```typescript
import { MatSnackBar } from '@angular/material/snack-bar';

@Component({ /* ... */ })
export class AppComponent {
  private snackBar = inject(MatSnackBar);

  showSuccess() {
    this.snackBar.open('Item saved successfully!', 'Dismiss', {
      duration: 3000,       // auto-dismiss after 3 seconds
      horizontalPosition: 'end',
      verticalPosition: 'bottom',
      panelClass: ['success-snackbar'],  // custom CSS class
    });
  }

  showCustomSnackbar() {
    // Open a custom component as a snackbar
    this.snackBar.openFromComponent(CustomSnackbarComponent, {
      duration: 5000,
      data: { message: 'File uploaded', progress: 100 }
    });
  }
}
```

### Table (MatTable)

`MatTable` is a powerful, flexible table with support for sorting, pagination, and filtering. It uses a **DataSource** pattern — the table itself doesn't know how data is fetched; it just renders what the DataSource provides.

```typescript
@Component({
  template: `
    <mat-table [dataSource]="dataSource" matSort>
      <!-- Name Column -->
      <ng-container matColumnDef="name">
        <th mat-header-cell *matHeaderCellDef mat-sort-header>Name</th>
        <td mat-cell *matCellDef="let row">{{ row.name }}</td>
      </ng-container>

      <!-- Status Column -->
      <ng-container matColumnDef="status">
        <th mat-header-cell *matHeaderCellDef>Status</th>
        <td mat-cell *matCellDef="let row">
          <mat-chip [color]="row.active ? 'primary' : 'warn'">
            {{ row.active ? 'Active' : 'Inactive' }}
          </mat-chip>
        </td>
      </ng-container>

      <tr mat-header-row *matHeaderRowDef="displayedColumns"></tr>
      <tr mat-row *matRowDef="let row; columns: displayedColumns;"></tr>

      <!-- No data row -->
      <tr class="mat-row" *matNoDataRow>
        <td class="mat-cell" [attr.colspan]="displayedColumns.length">
          No data matching the filter.
        </td>
      </tr>
    </mat-table>

    <mat-paginator [pageSizeOptions]="[10, 25, 50]" showFirstLastButtons />
  `,
  imports: [MatTableModule, MatSortModule, MatPaginatorModule, MatChipsModule]
})
export class UsersTableComponent implements AfterViewInit {
  @ViewChild(MatPaginator) paginator!: MatPaginator;
  @ViewChild(MatSort) sort!: MatSort;

  displayedColumns = ['name', 'status'];
  dataSource = new MatTableDataSource<User>(this.usersService.getAll());

  ngAfterViewInit() {
    // Connect paginator and sort to the data source AFTER view is ready
    this.dataSource.paginator = this.paginator;
    this.dataSource.sort = this.sort;
  }

  applyFilter(event: Event) {
    const filterValue = (event.target as HTMLInputElement).value;
    this.dataSource.filter = filterValue.trim().toLowerCase();
  }
}
```

### Tabs

```html
<mat-tab-group [(selectedIndex)]="activeTab" animationDuration="200ms">
  <mat-tab label="Overview">
    <!-- lazy load tab content with ng-template -->
    <ng-template matTabContent>
      <app-overview />
    </ng-template>
  </mat-tab>
  <mat-tab label="Details">
    <ng-template matTabContent>
      <app-details />
    </ng-template>
  </mat-tab>
  <mat-tab label="Settings" [disabled]="!canAccessSettings">
    <ng-template matTabContent>
      <app-settings />
    </ng-template>
  </mat-tab>
</mat-tab-group>
```

> **Performance tip:** Use `matTabContent` with `ng-template` to lazily render tab content — the component is only created when the tab is first activated.

### Progress Indicators

```html
<!-- Determinate (known progress) -->
<mat-progress-bar mode="determinate" [value]="uploadProgress" />

<!-- Indeterminate (unknown duration) -->
<mat-progress-bar mode="indeterminate" *ngIf="isLoading" />

<!-- Spinner -->
<mat-progress-spinner mode="indeterminate" diameter="40" />
<mat-progress-spinner mode="determinate" [value]="75" diameter="80" />
```

### Date Picker

```html
<mat-form-field appearance="outline">
  <mat-label>Birth Date</mat-label>
  <input matInput [matDatepicker]="birthDatePicker" [formControl]="birthDateControl">
  <mat-datepicker-toggle matIconSuffix [for]="birthDatePicker" />
  <mat-datepicker #birthDatePicker
    [startAt]="minDate"
    [min]="minDate"
    [max]="maxDate">
  </mat-datepicker>
</mat-form-field>
```

You need to provide a date adapter. Angular Material works with native Date, Luxon, Moment (legacy), and date-fns:

```typescript
// For date-fns (recommended in 2026)
import { provideDateFnsAdapter } from '@angular/material-date-fns-adapter';
import { MAT_DATE_LOCALE } from '@angular/material/core';

bootstrapApplication(AppComponent, {
  providers: [
    provideDateFnsAdapter(),
    { provide: MAT_DATE_LOCALE, useValue: enUS }
  ]
});
```

### Stepper

Steppers guide users through multi-step processes:

```html
<mat-stepper [linear]="isLinear" orientation="horizontal" #stepper>
  <mat-step [stepControl]="personalFormGroup" label="Personal Info">
    <form [formGroup]="personalFormGroup">
      <mat-form-field appearance="outline">
        <mat-label>First Name</mat-label>
        <input matInput formControlName="firstName" required>
      </mat-form-field>
    </form>
    <div>
      <button mat-button matStepperNext [disabled]="personalFormGroup.invalid">Next</button>
    </div>
  </mat-step>

  <mat-step [stepControl]="addressFormGroup" label="Address">
    <form [formGroup]="addressFormGroup">
      <!-- address fields -->
    </form>
    <div>
      <button mat-button matStepperPrevious>Back</button>
      <button mat-flat-button color="primary" matStepperNext>Continue</button>
    </div>
  </mat-step>

  <mat-step label="Review & Submit">
    <p>Review your details before submitting.</p>
    <button mat-button matStepperPrevious>Back</button>
    <button mat-flat-button color="primary" (click)="onSubmit()">Submit</button>
  </mat-step>
</mat-stepper>
```

---

## Custom Theming with CSS Variables

Because M3 uses CSS custom properties, you can easily customize any Material component in your component's CSS without touching SCSS:

```css
/* component.css — override Material tokens locally */
:host {
  --mdc-outlined-button-outline-color: var(--my-brand-color);
  --mdc-filled-button-container-color: #ff5722;
  --mat-tab-header-active-label-text-color: purple;
}
```

> **Best Practice:** Use CSS variable overrides for **local, component-level** customizations. Use SCSS theming for **global or multi-component** changes.

---

## Standalone Component Imports

In modern Angular (17+), you import Material components individually as standalone components instead of importing large modules. This enables better tree-shaking:

```typescript
import { MatButton } from '@angular/material/button';
import { MatFormField, MatLabel, MatHint, MatError } from '@angular/material/form-field';
import { MatInput } from '@angular/material/input';
import { MatSelect, MatOption } from '@angular/material/select';
import { MatDialog } from '@angular/material/dialog';

@Component({
  standalone: true,
  imports: [
    MatButton,
    MatFormField, MatLabel, MatHint, MatError,
    MatInput,
    MatSelect, MatOption,
  ],
  // ...
})
```

---

## Important Points & Best Practices

**Always use `provideAnimationsAsync()` instead of `BrowserAnimationsModule`** in standalone apps. The async version loads animation code only when needed, improving initial bundle size.

**Prefer standalone component imports over module imports.** Instead of `import { MatInputModule }`, import `import { MatInput }` directly. This gives bundlers better tree-shaking opportunities.

**Never forget to call `@include mat.core()`** exactly once, at the top of your global styles. Calling it multiple times creates duplicate CSS.

**For form validation errors**, always wrap `mat-error` inside a `mat-form-field` and use Angular's reactive form validators. The error message is automatically shown only when the control is invalid AND touched.

**Accessibility comes free, but you still need to help it.** Material adds ARIA attributes automatically, but you must still:
- Provide `aria-label` on icon-only buttons (`<button mat-icon-button aria-label="Delete">`)
- Use proper `<mat-label>` inside every form field (not placeholder as a substitute)
- Ensure dialog titles use `mat-dialog-title` (it sets the correct ARIA role)

**The `matSort` directive requires `MatSort` as a `@ViewChild`** and must be connected to the `MatTableDataSource` inside `ngAfterViewInit`, not `ngOnInit`. The view is not available yet during `ngOnInit`.

**For large tables with thousands of rows**, combine `MatTable` with `CdkVirtualScrollViewport` from the CDK for virtual scrolling — only rendering visible rows instead of all rows.

---

## Complete Working Example

Here is a complete component demonstrating several Material patterns together:

```typescript
import { Component, inject, signal } from '@angular/core';
import { FormBuilder, ReactiveFormsModule, Validators } from '@angular/forms';
import { MatButton } from '@angular/material/button';
import { MatFormField, MatLabel, MatError, MatHint } from '@angular/material/form-field';
import { MatInput } from '@angular/material/input';
import { MatSelect, MatOption } from '@angular/material/select';
import { MatSnackBar } from '@angular/material/snack-bar';
import { MatProgressBar } from '@angular/material/progress-bar';

@Component({
  selector: 'app-contact-form',
  standalone: true,
  imports: [
    ReactiveFormsModule,
    MatButton, MatFormField, MatLabel, MatError, MatHint,
    MatInput, MatSelect, MatOption, MatProgressBar,
  ],
  template: `
    @if (isSubmitting()) {
      <mat-progress-bar mode="indeterminate" />
    }

    <form [formGroup]="form" (ngSubmit)="onSubmit()">
      <mat-form-field appearance="outline">
        <mat-label>Name</mat-label>
        <input matInput formControlName="name" />
        <mat-hint>Full name as on your ID</mat-hint>
        @if (form.get('name')?.hasError('required') && form.get('name')?.touched) {
          <mat-error>Name is required</mat-error>
        }
      </mat-form-field>

      <mat-form-field appearance="outline">
        <mat-label>Topic</mat-label>
        <mat-select formControlName="topic">
          <mat-option value="support">Support</mat-option>
          <mat-option value="billing">Billing</mat-option>
          <mat-option value="feedback">Feedback</mat-option>
        </mat-select>
      </mat-form-field>

      <button mat-flat-button color="primary" type="submit" [disabled]="form.invalid || isSubmitting()">
        Submit
      </button>
    </form>
  `
})
export class ContactFormComponent {
  private fb = inject(FormBuilder);
  private snackBar = inject(MatSnackBar);

  isSubmitting = signal(false);

  form = this.fb.group({
    name: ['', Validators.required],
    topic: ['', Validators.required],
  });

  async onSubmit() {
    if (this.form.invalid) return;
    this.isSubmitting.set(true);
    try {
      await this.submitToApi(this.form.value);
      this.snackBar.open('Message sent!', 'OK', { duration: 3000 });
      this.form.reset();
    } finally {
      this.isSubmitting.set(false);
    }
  }

  private submitToApi(data: unknown) {
    return new Promise(resolve => setTimeout(resolve, 1500));
  }
}
```

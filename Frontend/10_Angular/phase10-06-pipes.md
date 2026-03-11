# Phase 10 — Angular Core Fundamentals
## Part 6: Pipes
> Topic: 10.10

---

## 10.10 Pipes

### What Is a Pipe?

A **pipe** is a template operator that transforms a value before displaying it. The syntax uses the `|` character (the "pipe" character) between the value and the transformation name. Pipes let you format data — dates, currencies, strings, arrays — right in the template, without cluttering your component class with formatting logic.

Think of pipes as **display formatters**. Your component holds raw data (`price = 4999.99`), and a pipe handles how it appears to the user (`$4,999.99`). This separation keeps your component's business logic clean and makes your formatting reusable across many templates.

---

## Angular's Built-in Pipes

Angular ships with a rich set of built-in pipes. All of them are standalone and can be imported directly into standalone components from `@angular/common`.

### `DatePipe`

The `DatePipe` formats a `Date` object, a timestamp number, or a date string into a human-readable string using Angular's locale-aware formatting:

```typescript
import { Component } from '@angular/core';
import { DatePipe } from '@angular/common';

@Component({
  standalone: true,
  imports: [DatePipe],
  selector: 'app-event',
  template: `
    <!-- 'short' format: '6/15/25, 9:00 AM' -->
    <p>Short: {{ eventDate | date:'short' }}</p>

    <!-- 'longDate' format: 'June 15, 2025' -->
    <p>Long date: {{ eventDate | date:'longDate' }}</p>

    <!-- 'fullDate' format: 'Sunday, June 15, 2025' -->
    <p>Full date: {{ eventDate | date:'fullDate' }}</p>

    <!-- Custom format string -->
    <p>Custom: {{ eventDate | date:'MMMM d, y — h:mm a' }}</p>

    <!-- With timezone parameter -->
    <p>UTC time: {{ eventDate | date:'HH:mm' : 'UTC' }}</p>

    <!-- With locale parameter for French formatting -->
    <p>French: {{ eventDate | date:'fullDate' : '' : 'fr' }}</p>

    <!-- Handling null safely — the pipe returns '' for null/undefined -->
    <p>Unset: {{ nullableDate | date:'short' }}</p>
  `
})
export class EventComponent {
  eventDate = new Date(2025, 5, 15, 9, 0, 0);  // June 15, 2025, 9:00 AM
  nullableDate: Date | null = null;
}
```

Common date format shortcuts: `'short'`, `'medium'`, `'long'`, `'full'`, `'shortDate'`, `'mediumDate'`, `'longDate'`, `'fullDate'`, `'shortTime'`, `'mediumTime'`.

### `CurrencyPipe`

```typescript
import { CurrencyPipe } from '@angular/common';

@Component({ standalone: true, imports: [CurrencyPipe], template: `
  <!-- Basic usage: defaults to USD in en-US locale -->
  <p>{{ price | currency }}</p>
  <!-- Output: $1,234.56 -->

  <!-- Specify currency code -->
  <p>{{ price | currency:'EUR' }}</p>
  <!-- Output: €1,234.56 -->

  <!-- Change display format: 'code' | 'symbol' | 'symbol-narrow' -->
  <p>{{ price | currency:'GBP':'code' }}</p>
  <!-- Output: GBP1,234.56 -->

  <!-- Control decimal places: minimum.minimum-maximum -->
  <p>{{ price | currency:'USD':'symbol':'1.0-0' }}</p>
  <!-- Output: $1,235 (rounded, no decimals) -->
` })
export class PriceComponent {
  price = 1234.56;
}
```

### `DecimalPipe` and `PercentPipe`

```typescript
import { DecimalPipe, PercentPipe } from '@angular/common';

@Component({ standalone: true, imports: [DecimalPipe, PercentPipe], template: `
  <!-- DecimalPipe — format: 'minIntegerDigits.minFractionDigits-maxFractionDigits' -->
  <p>{{ pi | number:'1.2-4' }}</p>
  <!-- Output: 3.1416 (2 minimum fraction digits, 4 maximum) -->

  <p>{{ bigNumber | number:'1.0-0' }}</p>
  <!-- Output: 1,234,567 (no decimals, thousands separator) -->

  <!-- PercentPipe — multiplies value by 100 and adds % -->
  <p>{{ completionRatio | percent }}</p>
  <!-- Output: 78% -->

  <p>{{ completionRatio | percent:'1.1-2' }}</p>
  <!-- Output: 78.5% -->
` })
export class StatsComponent {
  pi = 3.14159265;
  bigNumber = 1234567;
  completionRatio = 0.785;
}
```

### String Case Pipes

```typescript
import { UpperCasePipe, LowerCasePipe, TitleCasePipe } from '@angular/common';

@Component({ standalone: true, imports: [UpperCasePipe, LowerCasePipe, TitleCasePipe], template: `
  <p>{{ 'hello world' | uppercase }}</p>   <!-- HELLO WORLD -->
  <p>{{ 'HELLO WORLD' | lowercase }}</p>   <!-- hello world -->
  <p>{{ 'hello world' | titlecase }}</p>   <!-- Hello World -->
  <p>{{ 'the quick brown fox' | titlecase }}</p>  <!-- The Quick Brown Fox -->
` })
export class TextComponent { }
```

### `SlicePipe`

Slices an array or string, extracting a subset. Works exactly like JavaScript's `.slice()`:

```typescript
import { SlicePipe } from '@angular/common';

@Component({ standalone: true, imports: [SlicePipe], template: `
  <!-- Show only the first 3 items of an array -->
  @for (item of items | slice:0:3; track item.id) {
    <li>{{ item.name }}</li>
  }

  <!-- Show the last 2 items (negative index) -->
  @for (item of items | slice:-2; track item.id) {
    <li>{{ item.name }}</li>
  }

  <!-- Truncate a string for display -->
  <p>{{ longText | slice:0:100 }}{{ longText.length > 100 ? '...' : '' }}</p>
` })
export class SliceExampleComponent {
  items = [
    { id: 1, name: 'Alpha' }, { id: 2, name: 'Beta' },
    { id: 3, name: 'Gamma' }, { id: 4, name: 'Delta' }
  ];
  longText = 'A very long article excerpt that goes on and on...';
}
```

### `AsyncPipe` — A Critical Angular Pipe

`AsyncPipe` subscribes to an Observable or Promise and returns the most recently emitted value. When the component is destroyed, it **automatically unsubscribes** — this is its most important feature, since it prevents memory leaks without any manual cleanup code.

```typescript
import { Component } from '@angular/core';
import { AsyncPipe } from '@angular/common';
import { Observable, of, interval } from 'rxjs';
import { map } from 'rxjs/operators';

@Component({ standalone: true, imports: [AsyncPipe], selector: 'app-live-clock', template: `
  <!-- The async pipe subscribes, displays the value, and auto-unsubscribes on destroy -->
  <p>Current time: {{ currentTime$ | async }}</p>

  <!-- Using async with @if to unwrap to a non-null value -->
  @if (user$ | async; as user) {
    <!-- 'user' here is the unwrapped, non-null value -->
    <h2>Hello, {{ user.name }}</h2>
    <p>Role: {{ user.role }}</p>
  } @else {
    <p>Loading user...</p>
  }

  <!-- Multiple async subscriptions in a template (each creates its own subscription) -->
  @if (products$ | async; as products) {
    @for (product of products; track product.id) {
      <app-product-card [product]="product" />
    }
  }
` })
export class LiveClockComponent {
  // Emits the formatted time every second
  currentTime$ = interval(1000).pipe(
    map(() => new Date().toLocaleTimeString())
  );

  // Observable of user data (would normally come from a service)
  user$: Observable<{ name: string; role: string }> = of({ name: 'Alice', role: 'Admin' });
  products$: Observable<any[]> = of([]);
}
```

The pattern `@if (observable$ | async; as value)` is extremely common and important in Angular. It unwraps the observable, provides a named local variable (`value`), and shows nothing while the observable hasn't emitted yet. This eliminates the need for manual subscriptions in most display scenarios.

### `JsonPipe`

Displays an object as a formatted JSON string. Invaluable during development and debugging:

```html
<!-- Debug component state directly in the template -->
<pre>{{ user | json }}</pre>
<!-- Output:
{
  "id": 1,
  "name": "Alice",
  "email": "alice@example.com"
}
-->
```

### `KeyValuePipe`

Converts an object or Map into an array of `{ key, value }` pairs, making objects iterable in `@for`:

```typescript
import { KeyValuePipe } from '@angular/common';

@Component({ standalone: true, imports: [KeyValuePipe], template: `
  <!-- Without KeyValuePipe, you can't iterate over an object in a template -->
  <dl>
    @for (entry of userSettings | keyvalue; track entry.key) {
      <dt>{{ entry.key }}</dt>
      <dd>{{ entry.value }}</dd>
    }
  </dl>
` })
export class SettingsComponent {
  userSettings = {
    theme: 'dark',
    language: 'en',
    timezone: 'UTC+0'
  };
}
```

---

## Pipe Chaining

Pipes can be chained with multiple `|` operators, applying transformations in sequence from left to right:

```html
<!-- First format as currency, then convert to uppercase (results in "$1,234.56" → "$1,234.56" — same since currency output is not all-alpha) -->
{{ price | currency:'EUR' | uppercase }}

<!-- A date formatted in French, then uppercased -->
{{ eventDate | date:'MMMM yyyy' : '' : 'fr' | uppercase }}
<!-- Output: JUIN 2025 -->
```

---

## Creating Custom Pipes

When the built-in pipes don't cover your needs, creating a custom pipe is straightforward. Custom pipes are a great place to put reusable display transformation logic.

### Basic Custom Pipe

```typescript
// pipes/truncate.pipe.ts
import { Pipe, PipeTransform } from '@angular/core';

// @Pipe marks this class as an Angular pipe
@Pipe({
  name: 'truncate',   // The name used in templates: {{ value | truncate }}
  standalone: true,   // Standalone pipe — can be imported directly into components
})
export class TruncatePipe implements PipeTransform {
  // 'transform' is called whenever Angular evaluates the pipe expression.
  // 'value' is the input (the left side of the |).
  // Additional parameters come from the template: {{ text | truncate:50:'...' }}
  transform(value: string, limit: number = 100, ellipsis: string = '...'): string {
    if (!value) return '';
    if (value.length <= limit) return value;
    return value.substring(0, limit).trimEnd() + ellipsis;
  }
}
```

```typescript
// Using the custom pipe in a component
import { TruncatePipe } from '../pipes/truncate.pipe';

@Component({
  standalone: true,
  imports: [TruncatePipe],
  template: `
    <!-- Use the pipe with default params (100 chars, '...') -->
    <p>{{ article.content | truncate }}</p>

    <!-- Pass a custom limit and ellipsis string -->
    <p>{{ article.content | truncate:50 }}</p>
    <p>{{ article.content | truncate:200:' [read more]' }}</p>
  `
})
export class ArticleCardComponent {
  article = { content: 'A very long piece of article text that should be truncated...' };
}
```

### A More Complex Example: A Search Filter Pipe

```typescript
// pipes/filter.pipe.ts
import { Pipe, PipeTransform } from '@angular/core';

interface Nameable {
  name: string;
  [key: string]: any;
}

@Pipe({
  name: 'filter',
  standalone: true,
  pure: false,  // Set to impure — see below for explanation
})
export class FilterPipe implements PipeTransform {
  transform<T extends Nameable>(items: T[], searchTerm: string): T[] {
    if (!items || !searchTerm) return items;
    const lowerTerm = searchTerm.toLowerCase();
    return items.filter(item =>
      item.name.toLowerCase().includes(lowerTerm)
    );
  }
}
```

---

## Pure vs Impure Pipes — A Critical Distinction

This is one of the most important concepts to understand about pipes, with direct performance implications.

**Pure pipes** (the default — `pure: true`) only re-execute when Angular detects that the input value itself has changed. For primitive types, this means the value changed. For objects and arrays, it means the **reference** changed. If you push an item into an array or modify a property on an object without creating a new reference, Angular will not re-run the pipe.

```typescript
@Pipe({ name: 'filterByStatus', standalone: true, pure: true })
export class FilterByStatusPipe implements PipeTransform {
  transform(users: User[], status: string): User[] {
    // This will NOT re-run if you mutate the array with users.push(newUser)
    // because the array reference (pointer) didn't change.
    // It WILL re-run if you do: this.users = [...this.users, newUser]
    return users.filter(u => u.status === status);
  }
}
```

**Impure pipes** (`pure: false`) re-execute on **every single change detection cycle**, regardless of whether the input changed. This solves the mutation detection problem but at a significant performance cost — the pipe runs potentially hundreds of times per second.

```typescript
@Pipe({ name: 'filter', standalone: true, pure: false })
export class FilterPipe implements PipeTransform {
  // This WILL re-run on every change detection cycle — expensive!
  // Only use impure pipes when absolutely necessary.
  transform(items: any[], term: string): any[] {
    return items.filter(i => i.name.includes(term));
  }
}
```

**Angular's built-in `AsyncPipe` is impure** — because it needs to detect whenever a new value is emitted by the Observable, regardless of whether the Observable reference changed.

The **recommended practice** is to keep pipes pure and use **immutable data patterns** — always produce a new array/object reference instead of mutating in place. This way you get the efficiency of pure pipes without the stale-data problem.

---

## The `PipeTransform` Interface

Every custom pipe must implement the `PipeTransform` interface, which requires a single `transform` method. TypeScript enforces this contract at compile time:

```typescript
import { Pipe, PipeTransform } from '@angular/core';

@Pipe({ name: 'myPipe', standalone: true })
export class MyPipe implements PipeTransform {
  // The first argument is always the value being piped.
  // Subsequent arguments are the pipe parameters passed in the template.
  // The return type should match what you're displaying.
  transform(value: number, multiplier: number): number {
    return value * multiplier;
  }
}
```

---

## i18n Pipes and Locale Configuration

Angular's `DatePipe`, `CurrencyPipe`, `DecimalPipe`, and `PercentPipe` are locale-aware. To use a non-English locale globally, register the locale data and set the `LOCALE_ID` token in your application config:

```typescript
// In app.config.ts or AppModule
import { registerLocaleData } from '@angular/common';
import localeFr from '@angular/common/locales/fr';
import { LOCALE_ID } from '@angular/core';

// Register the locale data for French
registerLocaleData(localeFr);

export const appConfig: ApplicationConfig = {
  providers: [
    // Tell Angular to use French locale for all locale-aware pipes
    { provide: LOCALE_ID, useValue: 'fr-FR' },
    // ... other providers
  ]
};
```

With `LOCALE_ID` set to `'fr-FR'`, `{{ price | currency:'EUR' }}` will format as `1 234,56 €` instead of `€1,234.56`.

---

## Best Practices for 10.10

**Prefer pure pipes over methods in templates.** A method call in a template like `{{ formatUser(user) }}` executes on every change detection cycle. A pure pipe only runs when the input reference changes — often orders of magnitude fewer times. For any transformation you use in templates, wrap it in a pure pipe or a `computed()` signal.

**Keep pipes focused and composable.** A pipe should do one transformation. Instead of a `formatUserWithCurrencyAndDate` pipe, have three separate pipes and chain them. This makes each one reusable and testable independently.

**Leverage `AsyncPipe` for all Observable subscriptions in templates.** Manual `subscribe()` calls in component classes require manual `unsubscribe()` in `ngOnDestroy`. `AsyncPipe` handles this automatically, and it also correctly handles the case where a new Observable is assigned to the property — it unsubscribes from the old one and subscribes to the new one.

**Test your custom pipes thoroughly.** Pipes are pure TypeScript functions — they are the easiest thing to test in Angular. A pipe test doesn't need `TestBed` at all; just instantiate the class and call `transform()` directly:

```typescript
describe('TruncatePipe', () => {
  const pipe = new TruncatePipe();

  it('should truncate to 50 chars', () => {
    const result = pipe.transform('A'.repeat(100), 50);
    expect(result).toBe('A'.repeat(50) + '...');
  });

  it('should not truncate short strings', () => {
    expect(pipe.transform('hello', 50)).toBe('hello');
  });
});
```

**Don't use impure pipes for filtering or sorting lists.** This is an especially common mistake. If you put an impure filter pipe on a list with 1000 items in a busy application, Angular will re-filter all 1000 items on every keystroke, mouse movement, and timer tick. The better pattern is to filter the array in the component class (using a computed signal or an explicit method), and bind the filtered array to the `@for` block directly.

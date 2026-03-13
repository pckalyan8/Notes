# Phase 11.9 — Custom Directives

## What Are Directives and Why Build Custom Ones?

Angular has two types of built-in directives: **structural** (which add or remove DOM elements — `*ngIf`, `*ngFor`, `@if`, `@for`) and **attribute** (which change the appearance or behaviour of existing elements — `NgClass`, `NgStyle`). Directives let you attach reusable behaviour to any DOM element without creating a full component.

The key mental model is this: a component is a directive with a template. A directive is behaviour without a template. When you find yourself writing the same event handlers, DOM manipulations, or state logic across multiple components — click-outside detection, auto-focus, tooltip positioning, character counting, drag-and-drop, lazy image loading — that's a strong signal that the behaviour should live in a directive instead.

Directives promote the **single responsibility principle** at the template level. Rather than burdening a component with peripheral DOM concerns, you describe the behaviour declaratively on the element itself.

---

## Creating an Attribute Directive

The `@Directive` decorator is what turns a TypeScript class into an Angular directive. You provide a `selector` — a CSS-style attribute selector is conventional, typically in the format `[appMyDirective]`.

```typescript
// highlight.directive.ts
import { Directive, ElementRef, inject } from '@angular/core';

@Directive({
  selector: '[appHighlight]',  // Matches any element with the appHighlight attribute
  standalone: true,
})
export class HighlightDirective {
  // ElementRef gives you direct access to the host DOM element
  private el = inject(ElementRef);

  constructor() {
    // Directly set a style on the element — works but isn't ideal (see Renderer2 below)
    this.el.nativeElement.style.backgroundColor = 'yellow';
  }
}
```

Usage in a template — any element can receive this directive:

```html
<!-- The directive applies to this paragraph immediately on render -->
<p appHighlight>This text will have a yellow background.</p>
<span appHighlight>So will this span.</span>
```

---

## `Renderer2` — Safe DOM Manipulation

Directly setting `nativeElement.style` works in a browser, but Angular applications also run during Server-Side Rendering (SSR), in Web Workers, and in testing environments — contexts where `nativeElement` might not exist or where direct DOM access has security implications. `Renderer2` is the correct, platform-agnostic way to manipulate the DOM from a directive.

```typescript
import { Directive, ElementRef, Renderer2, inject } from '@angular/core';

@Directive({ selector: '[appHighlight]', standalone: true })
export class HighlightDirective {
  private el = inject(ElementRef);
  private renderer = inject(Renderer2);

  constructor() {
    // Use Renderer2 — safe across all platforms (browser, SSR, web worker)
    this.renderer.setStyle(this.el.nativeElement, 'background-color', 'yellow');
    this.renderer.addClass(this.el.nativeElement, 'highlighted');
    this.renderer.setAttribute(this.el.nativeElement, 'aria-highlighted', 'true');
  }
}
```

`Renderer2` provides a full API: `setStyle()`, `removeStyle()`, `addClass()`, `removeClass()`, `setAttribute()`, `removeAttribute()`, `appendChild()`, `insertBefore()`, `removeChild()`, and more. Prefer it over direct `nativeElement` access in all cases.

---

## `@HostListener` — Reacting to Events on the Host Element

Directives frequently need to respond to events on the element they're attached to. `@HostListener` registers an event listener on the host element without requiring the directive to access `addEventListener` manually.

```typescript
import { Directive, ElementRef, Renderer2, HostListener, inject } from '@angular/core';

@Directive({ selector: '[appHighlight]', standalone: true })
export class HighlightDirective {
  private el = inject(ElementRef);
  private renderer = inject(Renderer2);

  // @HostListener registers an event listener on the HOST element
  @HostListener('mouseenter')
  onMouseEnter() {
    this.setHighlight('yellow');
  }

  @HostListener('mouseleave')
  onMouseLeave() {
    this.setHighlight('');
  }

  // You can also listen to window or document events
  @HostListener('window:scroll', ['$event'])
  onWindowScroll(event: Event) {
    console.log('Window scrolled', event);
  }

  private setHighlight(color: string) {
    this.renderer.setStyle(this.el.nativeElement, 'background-color', color);
  }
}
```

---

## `@HostBinding` — Binding to Host Element Properties

While `@HostListener` responds to events, `@HostBinding` binds a directive class property to a property or attribute of the host element. This is how a directive can dynamically control things like CSS classes, ARIA attributes, or element properties.

```typescript
import { Directive, HostBinding, HostListener, Input } from '@angular/core';

@Directive({ selector: '[appHighlight]', standalone: true })
export class HighlightDirective {
  @Input('appHighlight') highlightColor = 'yellow';  // The colour is an input

  // Binds this property to the host element's 'style.background-color'
  @HostBinding('style.background-color')
  backgroundColor = '';

  // Binds to the host element's 'class.highlighted' CSS class
  @HostBinding('class.highlighted')
  isHighlighted = false;

  // Binds to an ARIA attribute
  @HostBinding('attr.aria-selected')
  ariaSelected: boolean | null = null;

  @HostListener('mouseenter')
  onMouseEnter() {
    this.backgroundColor = this.highlightColor;
    this.isHighlighted = true;
    this.ariaSelected = true;
  }

  @HostListener('mouseleave')
  onMouseLeave() {
    this.backgroundColor = '';
    this.isHighlighted = false;
    this.ariaSelected = null;
  }
}
```

Usage with a custom highlight colour:

```html
<!-- Pass the colour as the directive's input value -->
<p [appHighlight]="'lightblue'">Custom highlight colour</p>
<p appHighlight>Default yellow highlight</p>
```

---

## The `host` Property — Modern Alternative to Decorators

In Angular 17+, the recommended pattern for host bindings and listeners is the `host` property in the `@Directive` decorator metadata. It's more explicit and collocates all host-related configuration in one place.

```typescript
@Directive({
  selector: '[appHighlight]',
  standalone: true,
  // The 'host' property defines host bindings and listeners declaratively
  host: {
    // Event listener — value is the method to call
    '(mouseenter)': 'onMouseEnter()',
    '(mouseleave)': 'onMouseLeave()',

    // Property binding — value is a class expression
    '[style.background-color]': 'backgroundColor',
    '[class.highlighted]': 'isHighlighted',
    '[attr.aria-selected]': 'ariaSelected',
  },
})
export class HighlightDirective {
  backgroundColor = '';
  isHighlighted = false;
  ariaSelected: boolean | null = null;

  onMouseEnter() { /* ... */ }
  onMouseLeave() { /* ... */ }
}
```

This pattern is preferred because it makes host interactions visible at the decorator level, keeping the class body focused on logic rather than DOM wiring.

---

## Inputs in Directives

Directives accept inputs just like components do. The input can be provided as the attribute value itself, or as additional attributes with different names.

```typescript
import { Directive, Input, OnInit, inject } from '@angular/core';

@Directive({ selector: '[appTooltip]', standalone: true })
export class TooltipDirective implements OnInit {
  // The directive's attribute value becomes the tooltip text
  @Input('appTooltip') tooltipText = '';

  // Additional inputs for configuration
  @Input() tooltipPosition: 'top' | 'bottom' | 'left' | 'right' = 'top';
  @Input() tooltipDelay = 300;

  ngOnInit() {
    console.log(`Tooltip: "${this.tooltipText}" at ${this.tooltipPosition}`);
  }
}
```

```html
<!-- The directive name is the input binding, additional inputs are separate attributes -->
<button appTooltip="Save your changes" tooltipPosition="bottom" [tooltipDelay]="500">
  Save
</button>
```

---

## Using Signal Inputs in Directives (Angular 17+)

The modern approach uses `input()` signal functions instead of `@Input()` decorators, giving you all the benefits of signals — computed values, effects, and fine-grained change detection.

```typescript
import { Directive, input, effect, inject, ElementRef } from '@angular/core';

@Directive({
  selector: '[appBadge]',
  standalone: true,
  host: {
    '[attr.data-count]': 'count()',
  },
})
export class BadgeDirective {
  count = input<number>(0);
  color = input<string>('red');

  private el = inject(ElementRef);

  constructor() {
    // effect() re-runs whenever count() or color() changes
    effect(() => {
      const currentCount = this.count();
      const currentColor = this.color();
      // Update the badge DOM based on signal values
      this.el.nativeElement.style.setProperty('--badge-color', currentColor);
      this.el.nativeElement.classList.toggle('badge-hidden', currentCount === 0);
    });
  }
}
```

---

## Creating a Structural Directive

Structural directives manipulate the DOM layout — they add or remove elements by controlling `TemplateRef` and `ViewContainerRef`. They conventionally use the `*` microsyntax prefix.

The `*` prefix is actually syntactic sugar. `*appIf="condition"` is desugared by Angular into `[appIf]="condition"` applied to an `<ng-template>` wrapping the element. Inside the directive, you receive the `TemplateRef` (what to render) and the `ViewContainerRef` (where to render it).

```typescript
// app-if.directive.ts — a simplified NgIf clone
import { Directive, Input, TemplateRef, ViewContainerRef, inject } from '@angular/core';

@Directive({
  selector: '[appIf]',
  standalone: true,
})
export class AppIfDirective {
  private templateRef = inject(TemplateRef<any>);
  private viewContainer = inject(ViewContainerRef);
  private hasView = false;

  @Input()
  set appIf(condition: boolean) {
    if (condition && !this.hasView) {
      // Create the view — this inserts the template into the DOM
      this.viewContainer.createEmbeddedView(this.templateRef);
      this.hasView = true;
    } else if (!condition && this.hasView) {
      // Clear removes all views (destroys the embedded view, removes from DOM)
      this.viewContainer.clear();
      this.hasView = false;
    }
  }
}
```

```html
<!-- Usage is identical to *ngIf -->
<div *appIf="isLoggedIn">Welcome back, {{username}}!</div>
<div *appIf="!isLoggedIn">Please log in to continue.</div>
```

---

## The Directive Composition API (Angular 15+)

The **Directive Composition API** allows you to apply directives to a component's host element from within the component's `@Component` metadata, without requiring the parent template to do so. This is how Angular Material components apply `CdkFocusTrap` or `A11yModule` directives automatically.

The `hostDirectives` property accepts a list of directives to compose onto the component. The composed directive's inputs and outputs can optionally be exposed through the component's own input/output API.

```typescript
// A reusable "resizable" behaviour directive
@Directive({ selector: '[appResizable]', standalone: true })
export class ResizableDirective {
  @Input() minWidth = 100;
  @Input() maxWidth = 800;
}

// A reusable "draggable" behaviour directive
@Directive({ selector: '[appDraggable]', standalone: true })
export class DraggableDirective {
  @Input() dragHandle = '';
}

// A component that ALWAYS has these behaviours — no parent template needed
@Component({
  selector: 'app-widget',
  standalone: true,
  template: `<h3>{{ title }}</h3><ng-content></ng-content>`,
  hostDirectives: [
    // Compose ResizableDirective onto the component's host element
    {
      directive: ResizableDirective,
      inputs: ['minWidth', 'maxWidth'],  // Expose to consumers
    },
    {
      directive: DraggableDirective,
      inputs: ['dragHandle'],
    },
  ],
})
export class WidgetComponent {
  @Input() title = '';
}
```

```html
<!-- The parent sees minWidth, maxWidth, and dragHandle as WidgetComponent inputs -->
<app-widget title="My Widget" [minWidth]="200" [maxWidth]="600">
  Content here
</app-widget>
```

This is much more ergonomic than requiring every template that uses `app-widget` to also apply `appResizable` and `appDraggable` separately.

---

## Real-World Directive Examples

### Auto-Focus Directive

```typescript
@Directive({
  selector: '[appAutoFocus]',
  standalone: true,
})
export class AutoFocusDirective implements AfterViewInit {
  private el = inject(ElementRef);

  ngAfterViewInit() {
    // Focus the element after the view is rendered
    // setTimeout(0) defers until after the current rendering cycle
    setTimeout(() => this.el.nativeElement.focus(), 0);
  }
}
```

```html
<!-- Focus the search input immediately on page load -->
<input type="search" appAutoFocus placeholder="Search..." />
```

### Click-Outside Directive

```typescript
@Directive({
  selector: '[appClickOutside]',
  standalone: true,
})
export class ClickOutsideDirective {
  private el = inject(ElementRef);
  clickOutside = output<void>();  // Signal output

  @HostListener('document:click', ['$event.target'])
  onDocumentClick(target: HTMLElement) {
    // If the clicked element is NOT inside our host element, emit the event
    if (!this.el.nativeElement.contains(target)) {
      this.clickOutside.emit();
    }
  }
}
```

```html
<div appClickOutside (clickOutside)="closeDropdown()">
  <button (click)="toggleDropdown()">Open Menu</button>
  @if (isOpen) {
    <ul class="dropdown">
      <li>Option 1</li>
      <li>Option 2</li>
    </ul>
  }
</div>
```

### Intersection Observer (Lazy Loading)

```typescript
@Directive({
  selector: '[appLazyLoad]',
  standalone: true,
})
export class LazyLoadDirective implements OnInit, OnDestroy {
  private el = inject(ElementRef);
  visible = output<boolean>();

  private observer!: IntersectionObserver;

  ngOnInit() {
    this.observer = new IntersectionObserver(
      ([entry]) => {
        this.visible.emit(entry.isIntersecting);
        if (entry.isIntersecting) {
          this.observer.disconnect();  // Stop observing once visible
        }
      },
      { threshold: 0.1 }  // Trigger when 10% of element is visible
    );
    this.observer.observe(this.el.nativeElement);
  }

  ngOnDestroy() {
    this.observer.disconnect();
  }
}
```

### Character Counter Directive (for textarea)

```typescript
@Directive({
  selector: 'textarea[appCharCounter]',
  standalone: true,
  host: {
    '(input)': 'onInput($event)',
    '[attr.maxlength]': 'maxLength()',
  },
})
export class CharCounterDirective {
  maxLength = input<number>(500);
  charCount = signal(0);
  remaining = computed(() => this.maxLength() - this.charCount());

  onInput(event: Event) {
    const value = (event.target as HTMLTextAreaElement).value;
    this.charCount.set(value.length);
  }
}
```

---

## Key Best Practices

Use the `[appMyDirective]` attribute selector convention for all custom directives, prefixing with your app's namespace to avoid naming collisions with future HTML attributes or third-party libraries. Always prefer `Renderer2` over direct `nativeElement` manipulation — this keeps your directives compatible with SSR and non-DOM rendering contexts. Use the `host` metadata property instead of `@HostBinding` and `@HostListener` decorators in Angular 17+ — it's more readable and collocates host-related concerns. Prefer signal `input()` over `@Input()` for new directives to maintain consistency with signal-based components. Clean up any manually created subscriptions, event listeners, or observers in `ngOnDestroy` — directives are just as susceptible to memory leaks as components. Consider the Directive Composition API before creating new compound components — if the behaviour you need is a good fit for composition, it keeps your component API surface smaller and your template expressions cleaner.

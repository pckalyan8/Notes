# Phase 11.10/11.11 — View Transitions API in Angular

## What Is the View Transitions API?

For most of the web's history, navigating between pages or changing significant UI state has been instantaneous and jarring — one moment you see the old content, the next moment you see new content, with no visual connection between them. Native apps, on the other hand, have always used smooth transitions: slides, fades, shared element animations. The **View Transitions API** brings this capability to the web platform natively, and Angular 17+ integrates it seamlessly.

The View Transitions API is a browser API (`document.startViewTransition()`) that creates a smooth animated transition between two DOM states. At the most basic level, it captures a screenshot of the current state, performs your DOM update, then animates between the old and new states using a cross-fade by default. But it goes much further — with `view-transition-name`, individual elements can animate across the transition independently, creating the "shared element" effect where a thumbnail smoothly expands into a full-size image, or a list item slides into a detail view.

---

## Enabling View Transitions in Angular Router

The simplest way to use view transitions in Angular is through the router configuration. Adding `withViewTransitions()` to `provideRouter()` wraps every Angular route navigation inside `document.startViewTransition()` automatically. No manual code needed in individual components.

```typescript
// app.config.ts
import { provideRouter, withViewTransitions } from '@angular/router';

export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(
      routes,
      withViewTransitions(),  // All route navigations become animated transitions
    ),
  ],
};
```

With just this one addition, all your route changes will cross-fade smoothly. The browser handles the default animation — a fade from the old content to the new. This single line turns a flat SPA into something that feels much more polished.

---

## How the API Works Internally

Understanding the browser mechanics helps you customise transitions effectively.

When `document.startViewTransition(callback)` is called, the browser pauses rendering and takes a screenshot of the current page state. It then calls your `callback` function — which performs the DOM update (the route navigation, in Angular's case). After the DOM has updated, the browser creates animated pseudo-elements representing the old state and the new state, and animates between them.

The browser generates these pseudo-elements automatically. The `::view-transition-old(root)` pseudo-element contains the old screenshot, `::view-transition-new(root)` contains the new live state, and they animate together inside a `::view-transition` overlay that sits on top of everything during the transition. You can target these pseudo-elements with CSS to customise the animation.

```css
/* Customise the default cross-fade */
@keyframes slide-in-from-right {
  from { transform: translateX(100%); }
  to   { transform: translateX(0); }
}

@keyframes slide-out-to-left {
  from { transform: translateX(0); }
  to   { transform: translateX(-100%); }
}

/* Override the default fade with a slide animation for all transitions */
::view-transition-old(root) {
  animation: slide-out-to-left 300ms ease-in forwards;
}

::view-transition-new(root) {
  animation: slide-in-from-right 300ms ease-out forwards;
}
```

---

## Named View Transitions — Shared Element Animations

This is where view transitions become magical. By assigning a `view-transition-name` CSS property to an element, you tell the browser: "This element in the old state corresponds to this element in the new state." The browser then animates that specific element smoothly between its old and new positions, sizes, and shapes — independently of the rest of the page transition.

Imagine a product card in a grid. When you click it, the card's image animates from its small grid position to its full-size position on the detail page. This is the shared element animation pattern.

```css
/* List page — mark the product image for transition */
.product-card .image {
  view-transition-name: product-image;
}

/* Detail page — the hero image receives the SAME name */
.product-hero-image {
  view-transition-name: product-image;
}
```

When you navigate from the list to the detail page, the browser sees that both pages have an element named `product-image` and smoothly interpolates between them — position, size, border-radius, everything. No JavaScript animation logic needed.

```css
/* Customise just the shared element's animation */
::view-transition-old(product-image),
::view-transition-new(product-image) {
  animation-duration: 400ms;
  animation-timing-function: cubic-bezier(0.4, 0, 0.2, 1);
}
```

---

## Dynamic `view-transition-name` in Angular

A common challenge: in a list of products, every card needs a unique `view-transition-name` (you can't give all cards the same name — that breaks the matching). In Angular, you can set this dynamically using style bindings.

```typescript
@Component({
  selector: 'app-product-card',
  standalone: true,
  template: `
    <div
      class="product-card"
      [style.view-transition-name]="'product-' + product().id"
    >
      <img
        [style.view-transition-name]="'product-image-' + product().id"
        [src]="product().imageUrl"
        [alt]="product().name"
      />
      <h3>{{ product().name }}</h3>
      <p>{{ product().price | currency }}</p>
    </div>
  `,
})
export class ProductCardComponent {
  product = input.required<Product>();
}
```

```typescript
// On the detail page — use the same ID-based naming scheme
@Component({
  selector: 'app-product-detail',
  template: `
    <div class="detail-page">
      <img
        [style.view-transition-name]="'product-image-' + product().id"
        [src]="product().imageUrl"
        class="hero-image"
      />
      <h1>{{ product().name }}</h1>
    </div>
  `,
})
export class ProductDetailComponent {
  product = input.required<Product>();
}
```

When the user navigates from the list to the detail page for product ID 42, the browser finds `view-transition-name: product-image-42` in both old and new states and creates a smooth shared element transition.

---

## `@starting-style` — Entry Animations for New Elements

One of the most useful CSS companions to view transitions is the `@starting-style` at-rule. This lets you define the initial style of an element *before* it's first rendered — enabling entry animations without JavaScript.

Normally, if you set `opacity: 0` on an element and transition it to `opacity: 1`, the browser has no "before" state on first render — the element appears at full opacity instantly. `@starting-style` provides that initial "before" state.

```css
/* The target state */
.modal {
  opacity: 1;
  transform: scale(1);
  transition: opacity 200ms ease-out, transform 200ms ease-out;
}

/* The initial state — only applied on first paint */
@starting-style {
  .modal {
    opacity: 0;
    transform: scale(0.9);
  }
}
```

Now when Angular's `@if` adds the modal to the DOM, it immediately transitions from the `@starting-style` state to the normal state — a smooth scale-in fade. This is a CSS-only alternative to Angular's `:enter` animation for simple cases.

---

## Manual View Transitions with `document.startViewTransition()`

While `withViewTransitions()` handles route navigations automatically, you may need to trigger view transitions manually for non-route state changes — like toggling a theme, switching tabs, or updating a component's internal view.

```typescript
import { Component, signal } from '@angular/core';

@Component({
  selector: 'app-image-gallery',
  template: `
    <img [src]="currentImage().url" [alt]="currentImage().alt"
         style="view-transition-name: gallery-image" />

    <div class="controls">
      <button (click)="previous()">← Prev</button>
      <button (click)="next()">Next →</button>
    </div>
  `,
})
export class ImageGalleryComponent {
  private images = signal([
    { url: '/img/photo1.jpg', alt: 'Photo 1' },
    { url: '/img/photo2.jpg', alt: 'Photo 2' },
    { url: '/img/photo3.jpg', alt: 'Photo 3' },
  ]);

  private currentIndex = signal(0);

  currentImage = computed(() => this.images()[this.currentIndex()]);

  next() {
    // Check if the API is supported (not in all browsers yet)
    if (!document.startViewTransition) {
      // Fallback: just update immediately
      this.currentIndex.update(i => (i + 1) % this.images().length);
      return;
    }

    document.startViewTransition(() => {
      // This callback runs inside the transition — DOM updates here are animated
      this.currentIndex.update(i => (i + 1) % this.images().length);
    });
  }

  previous() {
    if (!document.startViewTransition) {
      this.currentIndex.update(i => (i - 1 + this.images().length) % this.images().length);
      return;
    }

    document.startViewTransition(() => {
      this.currentIndex.update(i => (i - 1 + this.images().length) % this.images().length);
    });
  }
}
```

---

## `withViewTransitions()` Options

The Angular router's `withViewTransitions()` accepts an options object that gives you access to the `ViewTransitionInfo` object, which lets you customise the animation based on the transition direction or route metadata.

```typescript
provideRouter(
  routes,
  withViewTransitions({
    onViewTransitionCreated: (info) => {
      const { transition, from, to } = info;

      // Skip the animation for certain navigations (back/forward, specific routes)
      if (info.isBackNavigation) {
        // Different animation for back navigation
        document.documentElement.classList.add('back-navigation');
        transition.finished.finally(() => {
          document.documentElement.classList.remove('back-navigation');
        });
      }
    },
  }),
)
```

```css
/* Apply different animation direction for back navigation */
:root.back-navigation::view-transition-old(root) {
  animation: slide-out-to-right 300ms ease-in forwards;
}

:root.back-navigation::view-transition-new(root) {
  animation: slide-in-from-left 300ms ease-out forwards;
}
```

---

## Cross-Document View Transitions (MPA)

An exciting evolution of the API (available in Chrome 126+ with progressive adoption) extends view transitions to work between separate HTML pages — not just within a SPA. This is the **Multi-Page Application (MPA) transition** and it requires no JavaScript at all. You declare it in CSS:

```css
/* page1.html and page2.html — add this to both pages' CSS */
@view-transition {
  navigation: auto;  /* Enable cross-document view transitions */
}
```

Angular's SSR-rendered applications (which are technically MPAs on first load) can benefit from this for transitions between pre-rendered pages. Angular 21 supports cross-document view transitions in conjunction with incremental hydration.

---

## Accessibility: Always Handle `prefers-reduced-motion`

Motion sensitivity is a real accessibility concern. The W3C's WCAG 2.1 Success Criterion 2.3.3 (Level AAA) recommends giving users control over animations. The `prefers-reduced-motion` media query lets you detect users who have requested reduced motion at the system level (Windows Ease of Access, macOS Display Accessibility, iOS/Android Motion settings).

For view transitions specifically, the recommended approach is to completely disable them when the user prefers reduced motion, since view transitions are inherently motion-based.

```css
/* Disable all view transitions for users who prefer reduced motion */
@media (prefers-reduced-motion: reduce) {
  ::view-transition-group(*),
  ::view-transition-old(*),
  ::view-transition-new(*) {
    animation: none !important;
  }
}
```

In Angular, you can respect this at the router level:

```typescript
provideRouter(
  routes,
  withViewTransitions({
    onViewTransitionCreated: ({ transition }) => {
      if (window.matchMedia('(prefers-reduced-motion: reduce)').matches) {
        // Skip the animation entirely by calling skipTransition()
        transition.skipTransition();
      }
    },
  }),
)
```

---

## Combining View Transitions with Angular Animations

View Transitions and Angular Animations serve complementary roles. View Transitions handle smooth transitions between discrete DOM states and route changes — they're particularly powerful for shared element effects. Angular Animations handle complex choreography within a component — staggered lists, multi-step sequences, state-machine-driven transitions.

You can use both in the same application. A route navigation might use a View Transition (via `withViewTransitions()`) to slide the page in, while the content within the new route uses Angular Animations to stagger its list items into view. The key is not to fight both systems against each other on the same element — choose one per element.

---

## Browser Support and Progressive Enhancement

As of early 2026, View Transitions are supported in Chrome, Edge, and Safari (since Safari 18). Firefox support is underway. The pattern you saw in the gallery example — checking `document.startViewTransition` before calling it and falling back to an instant update — is the correct progressive enhancement approach.

Angular's `withViewTransitions()` already handles this gracefully. On browsers that don't support the API, navigations simply happen instantly without an error. You get enhanced UX on modern browsers and full functionality on older ones.

---

## A Complete View Transitions Implementation

This example puts everything together into a production-grade setup that handles both route transitions and shared element animations, with proper accessibility and progressive enhancement.

```typescript
// app.config.ts
export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(
      routes,
      withViewTransitions({
        onViewTransitionCreated: ({ transition }) => {
          if (window.matchMedia('(prefers-reduced-motion: reduce)').matches) {
            transition.skipTransition();
          }
        },
      }),
    ),
  ],
};
```

```css
/* global-transitions.css */

/* Default: smooth cross-fade for route changes */
::view-transition-old(root) {
  animation: fade-out 200ms ease-in forwards;
}

::view-transition-new(root) {
  animation: fade-in 200ms ease-out forwards;
}

@keyframes fade-out {
  to { opacity: 0; }
}

@keyframes fade-in {
  from { opacity: 0; }
}

/* Shared element: smooth morph between list and detail */
::view-transition-old(product-image),
::view-transition-new(product-image) {
  animation-duration: 400ms;
  animation-timing-function: cubic-bezier(0.4, 0, 0.2, 1);
  overflow: hidden;
}

/* Disable everything for reduced motion */
@media (prefers-reduced-motion: reduce) {
  ::view-transition-group(*),
  ::view-transition-old(*),
  ::view-transition-new(*) {
    animation: none !important;
  }
}
```

---

## Key Best Practices

Always check browser support and provide a no-animation fallback before calling `document.startViewTransition()` manually. The `withViewTransitions()` router integration handles this for you, but manual calls need the guard. Every `view-transition-name` value must be unique on the page at any given moment — having two elements with the same name during a transition confuses the browser and can cause visual artefacts. Use `view-transition-name` on a small number of semantically important elements (hero images, primary content containers) rather than blanketing every element with names — more named transitions means more work for the browser compositor. Always implement `prefers-reduced-motion` handling — this is both an accessibility best practice and increasingly a legal accessibility requirement in many jurisdictions. Use `@starting-style` for simple element entry animations (modals, dropdowns, toasts) before reaching for Angular's animation system — it's a CSS-only solution with no runtime cost. Think of view transitions as the bridge between web and native app experiences — used judiciously, they dramatically improve perceived quality without adding significant complexity.

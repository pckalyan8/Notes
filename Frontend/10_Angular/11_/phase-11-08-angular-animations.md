# Phase 11.8 — Angular Animations

## Why Angular Has a Built-In Animation System

CSS transitions and keyframes are powerful, but they have significant limitations in an Angular application. You can't trigger a CSS animation from a component's data model. You can't animate an element *before* it's removed from the DOM (Angular removes it immediately). You can't synchronise complex multi-step sequences tied to state changes. And you can't easily make animations configurable or reusable across components.

Angular's animation DSL solves all of these problems by integrating animations directly into component state management. When a signal or property changes, Angular can trigger a full choreographed sequence — transitioning old state out while the new state transitions in, even coordinating multiple elements simultaneously.

---

## Setting Up Animations

Angular animations require `BrowserAnimationsModule` or its standalone provider equivalent.

```typescript
// app.config.ts — standalone setup
import { provideAnimations } from '@angular/platform-browser/animations';
// Or for testing / accessibility concerns:
import { provideNoopAnimations } from '@angular/platform-browser/animations';

export const appConfig: ApplicationConfig = {
  providers: [
    provideAnimations(),  // Enables animation engine throughout the app
  ],
};
```

Then import `trigger` and related animation functions in your component:

```typescript
import {
  trigger, state, style, animate, transition,
  keyframes, group, sequence, query, stagger
} from '@angular/animations';
```

---

## Core Concepts: Trigger, State, Transition, Animate

The Angular animation API has four fundamental concepts that build on each other.

A **trigger** is the named animation attachment point. You bind a trigger to a template element and its value determines which animation state the element is in. Think of it as a state machine name.

A **state** defines the CSS styles that apply when an element is in a particular animation state. States are stable — they persist indefinitely until the trigger value changes.

A **transition** defines what animation plays *between* two states. It answers: "When the element moves from state A to state B, how does it animate?"

`**animate()**` defines the *how* of a transition — the timing, easing curve, and optionally the intermediate styles.

```typescript
@Component({
  selector: 'app-toggle-box',
  standalone: true,
  template: `
    <!-- The trigger name 'openClose' is bound to the component property 'isOpen' -->
    <!-- Angular maps the boolean to the string states 'open' and 'closed' -->
    <div [@openClose]="isOpen ? 'open' : 'closed'" class="box">
      <ng-content></ng-content>
    </div>
    <button (click)="toggle()">Toggle</button>
  `,
  animations: [
    trigger('openClose', [
      // State: what styles apply when the element is "open"
      state('open', style({
        height: '300px',
        opacity: 1,
        backgroundColor: '#ffffff',
      })),
      // State: what styles apply when the element is "closed"
      state('closed', style({
        height: '0px',
        opacity: 0,
        backgroundColor: '#e8e8e8',
      })),
      // Transition: animate when going from 'open' to 'closed'
      transition('open => closed', [
        animate('0.3s ease-in')  // Duration + easing only — styles from state definitions
      ]),
      // Transition: animate when going from 'closed' to 'open'
      transition('closed => open', [
        animate('0.4s ease-out')
      ]),
      // Shorthand: same animation in BOTH directions
      // transition('open <=> closed', animate('0.3s ease-in-out')),
    ]),
  ],
})
export class ToggleBoxComponent {
  isOpen = true;
  toggle() { this.isOpen = !this.isOpen; }
}
```

---

## `:enter` and `:leave` — Animating Elements in the DOM

The most powerful feature of Angular's animation system (compared to pure CSS) is the ability to animate elements as they are added to or removed from the DOM. The special aliases `:enter` and `:leave` map to Angular's `ngIf`, `@if`, and `@for` lifecycle.

When Angular is about to remove an element (`:leave`), it pauses the removal, runs the leave animation, and only removes the element from the DOM once the animation completes. This is impossible with pure CSS because the element is already gone before you can animate it.

```typescript
@Component({
  standalone: true,
  template: `
    @if (isVisible) {
      <!-- @fadeInOut trigger causes the div to animate in and out -->
      <div @fadeInOut class="notification">
        Saved successfully!
      </div>
    }
    <button (click)="isVisible = !isVisible">Toggle</button>
  `,
  animations: [
    trigger('fadeInOut', [
      // ':enter' is an alias for 'void => *' (element enters the DOM)
      transition(':enter', [
        style({ opacity: 0, transform: 'translateY(-10px)' }),  // Start state
        animate('200ms ease-out',
          style({ opacity: 1, transform: 'translateY(0)' })     // End state
        ),
      ]),
      // ':leave' is an alias for '* => void' (element leaves the DOM)
      transition(':leave', [
        animate('150ms ease-in',
          style({ opacity: 0, transform: 'translateY(-10px)' })
        ),
      ]),
    ]),
  ],
})
export class NotificationComponent {
  isVisible = false;
}
```

The `void` state represents the absence of the element in the DOM. So `:enter` (`void => *`) means "going from not existing to any state", and `:leave` (`* => void`) means "going from any state to not existing."

---

## `keyframes()` — Multi-Step Animations

For animations that need to pass through intermediate steps — like a shake effect or a bounce — use `keyframes()` inside `animate()`. Each `style()` inside `keyframes()` accepts an `offset` from 0 to 1 representing the percentage through the animation.

```typescript
trigger('shake', [
  transition('* => shaking', [
    animate('500ms', keyframes([
      style({ transform: 'translateX(0)',    offset: 0 }),
      style({ transform: 'translateX(-8px)', offset: 0.25 }),
      style({ transform: 'translateX(8px)',  offset: 0.5 }),
      style({ transform: 'translateX(-8px)', offset: 0.75 }),
      style({ transform: 'translateX(0)',    offset: 1 }),
    ])),
  ]),
])
```

---

## `group()` and `sequence()` — Choreographing Multiple Animations

`group()` runs multiple animations **in parallel** (simultaneously). `sequence()` runs multiple animations **in order** (one after another). You can nest them to create complex choreographies.

```typescript
trigger('cardExpand', [
  transition('collapsed => expanded', [
    // Run both animations at the same time
    group([
      animate('400ms ease-out', style({ height: '400px' })),    // Height expands
      animate('300ms 100ms ease-in', style({ opacity: 1 })),    // Fade in with 100ms delay
    ]),
  ]),

  transition('expanded => collapsed', [
    // Run animations one after the other
    sequence([
      animate('200ms', style({ opacity: 0 })),  // First: fade out
      animate('300ms', style({ height: '60px' })), // Then: collapse height
    ]),
  ]),
])
```

---

## `query()` and `stagger()` — Animating Lists

`query()` finds child elements inside the animated host and applies animations to them. `stagger()` introduces a delay between each child's animation, creating a cascading "waterfall" effect that makes lists feel alive and natural.

```typescript
@Component({
  standalone: true,
  template: `
    <ul @listAnimation>
      @for (item of items; track item.id) {
        <li>{{ item.label }}</li>
      }
    </ul>
  `,
  animations: [
    trigger('listAnimation', [
      transition('* => *', [  // Run on any state change (items added/removed)
        query(':enter', [
          style({ opacity: 0, transform: 'translateX(-20px)' }),
          // stagger: each child starts its animation 60ms after the previous one
          stagger(60, [
            animate('300ms ease-out',
              style({ opacity: 1, transform: 'translateX(0)' })
            ),
          ]),
        ], { optional: true }),  // optional: true prevents errors if no elements match

        query(':leave', [
          stagger(40, [
            animate('200ms ease-in',
              style({ opacity: 0, transform: 'translateX(20px)' })
            ),
          ]),
        ], { optional: true }),
      ]),
    ]),
  ],
})
export class AnimatedListComponent {
  items = signal<{ id: number; label: string }[]>([]);
}
```

---

## Animating Route Transitions

One of the most impactful uses of Angular animations is creating smooth transitions between routes — giving your SPA the feel of a native app.

```typescript
// Define the animation
export const routeAnimations = trigger('routeAnimations', [
  transition('* <=> *', [
    query(':enter, :leave', [
      style({
        position: 'absolute',
        width: '100%',
        opacity: 0,
      }),
    ], { optional: true }),

    query(':enter', [
      style({ transform: 'translateX(100%)', opacity: 0 }),
    ], { optional: true }),

    query(':leave', [
      animate('300ms ease-out',
        style({ transform: 'translateX(-100%)', opacity: 0 })
      ),
    ], { optional: true }),

    query(':enter', [
      animate('300ms ease-out',
        style({ transform: 'translateX(0)', opacity: 1 })
      ),
    ], { optional: true }),
  ]),
]);

// In the shell component — apply the trigger to the router-outlet host
@Component({
  template: `
    <div [@routeAnimations]="getRouteAnimationData()">
      <router-outlet></router-outlet>
    </div>
  `,
  animations: [routeAnimations],
})
export class ShellComponent {
  private router = inject(Router);

  getRouteAnimationData() {
    // Return a unique value per route to trigger the animation on navigation
    return this.router.url;
  }
}
```

---

## Reusable Animation Definitions

You can extract animation definitions into constants and import them across multiple components. This is the Angular equivalent of CSS keyframe libraries.

```typescript
// animations.ts — shared animation definitions
import { animate, style, transition, trigger } from '@angular/animations';

export const fadeIn = trigger('fadeIn', [
  transition(':enter', [
    style({ opacity: 0 }),
    animate('200ms ease-out', style({ opacity: 1 })),
  ]),
]);

export const slideDown = trigger('slideDown', [
  transition(':enter', [
    style({ height: 0, overflow: 'hidden' }),
    animate('300ms ease-out', style({ height: '*' })),  // '*' = computed auto height
  ]),
  transition(':leave', [
    animate('200ms ease-in', style({ height: 0, overflow: 'hidden' })),
  ]),
]);
```

```typescript
// In any component — just import and use
import { fadeIn, slideDown } from '../animations';

@Component({
  animations: [fadeIn, slideDown],
  template: `
    <div @fadeIn class="modal">...</div>
    <div @slideDown class="dropdown">...</div>
  `,
})
export class MyComponent {}
```

---

## `AnimationBuilder` — Programmatic Animations

Sometimes you need to trigger animations from code rather than template bindings — for example, in response to a scroll event, a WebSocket message, or after an imperative DOM operation. `AnimationBuilder` lets you create and play animations programmatically.

```typescript
import { Component, ElementRef, ViewChild, inject } from '@angular/core';
import { AnimationBuilder, animate, style } from '@angular/animations';

@Component({ ... })
export class HighlightComponent {
  @ViewChild('targetElement') targetEl!: ElementRef;
  private animBuilder = inject(AnimationBuilder);

  highlightRow() {
    // Build the animation factory
    const factory = this.animBuilder.build([
      style({ backgroundColor: '#fff3cd' }),
      animate('1000ms ease-out', style({ backgroundColor: 'transparent' })),
    ]);

    // Create a player for the specific DOM element
    const player = factory.create(this.targetEl.nativeElement);
    player.play();

    // Clean up after animation completes
    player.onDone(() => player.destroy());
  }
}
```

---

## Animation Callbacks — Reacting to Animation Events

Angular exposes `(@trigger.start)` and `(@trigger.done)` event bindings that let you react to animation lifecycle events.

```typescript
@Component({
  template: `
    <div
      [@expand]="isExpanded"
      (@expand.start)="onAnimationStart($event)"
      (@expand.done)="onAnimationDone($event)"
    >
      Content
    </div>
  `,
})
export class ExpandableComponent {
  isExpanded = false;

  onAnimationStart(event: AnimationEvent) {
    console.log(`Animation started: ${event.fromState} → ${event.toState}`);
  }

  onAnimationDone(event: AnimationEvent) {
    console.log(`Animation finished in ${event.totalTime}ms`);
    // Good place to emit events, toggle loading states, etc.
  }
}
```

---

## Disabling Animations

You can disable animations on an element and all its descendants with `[@.disabled]="true"`. This is useful for conditionally disabling animations during testing or when the user prefers reduced motion.

```typescript
@Component({
  template: `
    <!-- Respect the user's motion preference -->
    <div [@.disabled]="prefersReducedMotion">
      <div @fadeIn>Animated content</div>
      <app-animated-card />
    </div>
  `,
})
export class AppComponent {
  // Read the user's system preference
  prefersReducedMotion = window.matchMedia('(prefers-reduced-motion: reduce)').matches;
}
```

---

## Key Best Practices

Always define animations in the component's `animations` array rather than inline in the template, keeping templates clean and animation logic centralised. Extract commonly used animations (fade, slide, shake) into a shared `animations.ts` file and import them where needed — avoid duplicating animation definitions. Use `{ optional: true }` in `query()` calls when the queried elements might not exist — otherwise Angular throws a runtime error. Keep animation durations short for UI micro-interactions (150–300ms) and slightly longer for page-level transitions (300–500ms). Long animations feel sluggish and reduce perceived performance. Always handle `prefers-reduced-motion` by either disabling animations entirely or replacing motion with instant transitions for users who are motion-sensitive. This is both an accessibility requirement and increasingly a legal requirement. For simple enter/leave animations, prefer CSS View Transitions (covered in 11.11) when you're on Angular 17+ with `withViewTransitions()` — they're simpler to implement and leverage native browser APIs. Use Angular Animations for more complex, stateful, or coordinated sequences.

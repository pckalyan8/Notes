# Phase 11.7 — Content Projection

## What Is Content Projection?

Content projection is Angular's mechanism for building components that act as containers or wrappers — components that accept external content from their parent and render it in a designated slot. Think of HTML elements like `<button>` or `<dialog>`: you don't tell `<button>` what text to display via an attribute; you put the content *between* the tags. Content projection gives your Angular components that same expressive quality.

Without content projection, every piece of content your component displays must be either hardcoded in the template or passed as an `@Input()` string/object. Content projection goes further — it lets parent components inject arbitrary HTML, other components, or template blocks into specific slots in your component's template. This is what enables truly flexible, reusable components like cards, modals, tabs, and layout wrappers.

---

## Basic Content Projection with `<ng-content>`

The `<ng-content>` element in your component's template acts as a **placeholder** where Angular will insert whatever content the parent provides between your component's tags.

```typescript
// card.component.ts
import { Component } from '@angular/core';

@Component({
  selector: 'app-card',
  standalone: true,
  template: `
    <div class="card">
      <div class="card-body">
        <!-- Whatever the parent puts between <app-card>...</app-card> lands here -->
        <ng-content></ng-content>
      </div>
    </div>
  `,
  styles: [`
    .card { border: 1px solid #ddd; border-radius: 8px; box-shadow: 0 2px 4px rgba(0,0,0,0.1); }
    .card-body { padding: 16px; }
  `]
})
export class CardComponent {}
```

Now the parent can project any content into the card — text, other components, even complex nested templates:

```html
<!-- parent.component.html -->
<app-card>
  <!-- Everything between the tags is projected into <ng-content> -->
  <h2>User Profile</h2>
  <p>John Doe — Software Engineer</p>
  <app-avatar [user]="currentUser" />
</app-card>
```

The rendered DOM will have the card's wrapper with the projected content nested inside, exactly as if you had written it there yourself. Importantly, the projected content is **owned by the parent** — it uses the parent's styles, context, and change detection scope.

---

## Multi-Slot Content Projection with `select`

Basic `<ng-content>` creates a single slot. For more structured components — like a card with a distinct header, body, and footer — you need multiple named slots. The `select` attribute on `<ng-content>` acts as a CSS selector to route specific content to specific slots.

```typescript
// panel.component.ts
@Component({
  selector: 'app-panel',
  standalone: true,
  template: `
    <div class="panel">
      <!-- Selects elements with the CSS class 'panel-header' -->
      <div class="panel-header">
        <ng-content select=".panel-header"></ng-content>
      </div>

      <!-- Selects elements with an attribute [panel-body] -->
      <div class="panel-body">
        <ng-content select="[panel-body]"></ng-content>
      </div>

      <!-- A 'catch-all' slot for anything that doesn't match other selectors -->
      <div class="panel-footer">
        <ng-content select="app-panel-footer"></ng-content>
      </div>

      <!-- This catches all content that didn't match any select -->
      <ng-content></ng-content>
    </div>
  `,
})
export class PanelComponent {}
```

```html
<!-- parent.component.html — parent controls what goes where -->
<app-panel>

  <!-- Routed to select=".panel-header" -->
  <div class="panel-header">
    <h2>My Panel Title</h2>
  </div>

  <!-- Routed to select="[panel-body]" -->
  <div panel-body>
    <p>This is the main content of the panel.</p>
    <app-data-table [data]="tableData" />
  </div>

  <!-- Routed to select="app-panel-footer" -->
  <app-panel-footer>
    <button (click)="save()">Save</button>
    <button (click)="cancel()">Cancel</button>
  </app-panel-footer>

</app-panel>
```

The `select` attribute accepts CSS selector syntax: element names (`app-header`), class selectors (`.my-class`), attribute selectors (`[my-attribute]`), or combinations. Anything that doesn't match a `select` falls into the default `<ng-content>` (the one without `select`).

---

## `ng-container` — Projection Without Extra DOM Elements

Sometimes you want to project content but don't want an extra wrapper `<div>` or `<span>` in the rendered DOM. `<ng-container>` is a logical grouping element that Angular uses at compile time but never renders into the DOM.

```html
<!-- Using ng-container as a projection wrapper — no extra DOM node created -->
<app-panel>

  <!-- This ng-container groups the header content without adding a DOM element -->
  <ng-container ngProjectAs=".panel-header">
    <h2>Title</h2>
    <span class="badge">New</span>
  </ng-container>

  <div panel-body>
    <p>Body content here</p>
  </div>

</app-panel>
```

The `ngProjectAs` attribute tells Angular to treat this `ng-container` *as if* it were the specified selector for projection routing purposes. Without `ngProjectAs`, the `ng-container` would fall into the default catch-all slot since it doesn't have the `.panel-header` class.

---

## `ng-template` — Reusable Template Definitions

`<ng-template>` is a template definition element — it defines a block of HTML that Angular can render on demand, but it doesn't render anything immediately when the page loads. You can think of it as a template blueprint that gets rendered only when explicitly stamped out.

```html
<!-- Define a template — nothing renders here initially -->
<ng-template #loadingSpinner>
  <div class="spinner">
    <div class="spinner-circle"></div>
    <p>Loading, please wait...</p>
  </div>
</ng-template>

<!-- Render the template conditionally -->
@if (isLoading) {
  <ng-container *ngTemplateOutlet="loadingSpinner"></ng-container>
}
```

`ng-template` becomes powerful when you pass it into components as an `@Input()`, letting the parent customise how a child component renders certain pieces of its UI.

---

## `NgTemplateOutlet` — Rendering Templates Dynamically

`NgTemplateOutlet` is a directive that renders an `<ng-template>` reference. This is how you build components that accept custom templates from the parent — the classic "render prop" or "slot" pattern.

```typescript
// list.component.ts — accepts a custom item template from the parent
import { Component, Input, ContentChild, TemplateRef } from '@angular/core';
import { NgTemplateOutlet } from '@angular/common';

@Component({
  selector: 'app-list',
  standalone: true,
  imports: [NgTemplateOutlet],
  template: `
    <ul>
      @for (item of items; track item.id) {
        <li>
          <!--
            If the parent provided a custom template, use it.
            $implicit passes 'item' as the default template context variable.
          -->
          @if (itemTemplate) {
            <ng-container
              *ngTemplateOutlet="itemTemplate; context: { $implicit: item, index: $index }"
            ></ng-container>
          } @else {
            <!-- Default rendering if no custom template is provided -->
            <span>{{ item.label }}</span>
          }
        </li>
      }
    </ul>
  `,
})
export class ListComponent<T> {
  @Input() items: T[] = [];
  @ContentChild('itemTemplate') itemTemplate!: TemplateRef<any>;
}
```

```html
<!-- parent.component.html — customising how each list item renders -->
<app-list [items]="products">

  <!-- #itemTemplate references the ContentChild in the list component -->
  <ng-template #itemTemplate let-product let-index="index">
    <!-- 'let-product' binds to $implicit; 'let-index' binds to the 'index' context key -->
    <div class="product-row">
      <span class="index">{{ index + 1 }}.</span>
      <strong>{{ product.name }}</strong>
      <span class="price">${{ product.price }}</span>
      <app-product-badge [status]="product.status" />
    </div>
  </ng-template>

</app-list>
```

The `context` object passed to `ngTemplateOutlet` defines the template variables available inside the `<ng-template>`. The special `$implicit` key becomes the default variable when you use `let-varName` without an assignment (`let-product`). Named keys (`index: $index`) become named bindings (`let-index="index"`).

---

## `TemplateRef` and `ViewContainerRef` — Programmatic Rendering

`TemplateRef` is a reference to a compiled template. `ViewContainerRef` is a container in the DOM where you can programmatically create, insert, and destroy views. Together, they power structural directives like `*ngIf` and `*ngFor`.

```typescript
// tooltip.component.ts — dynamically creates and inserts a tooltip view
import { Component, ContentChild, ViewChild, TemplateRef, ViewContainerRef, inject } from '@angular/core';

@Component({
  selector: 'app-tooltip',
  template: `
    <div (mouseenter)="show()" (mouseleave)="hide()">
      <!-- The trigger content (whatever the parent passes) -->
      <ng-content select="[trigger]"></ng-content>

      <!-- The container where the tooltip will be inserted programmatically -->
      <div #tooltipContainer></div>
    </div>

    <!-- The tooltip content template — not rendered until show() is called -->
    <ng-template #tooltipContent>
      <div class="tooltip">
        <ng-content select="[tooltip]"></ng-content>
      </div>
    </ng-template>
  `,
})
export class TooltipComponent {
  @ViewChild('tooltipContainer', { read: ViewContainerRef })
  container!: ViewContainerRef;

  @ViewChild('tooltipContent')
  template!: TemplateRef<any>;

  private isVisible = false;

  show() {
    if (!this.isVisible) {
      // createEmbeddedView() stamps the template into the container
      this.container.createEmbeddedView(this.template);
      this.isVisible = true;
    }
  }

  hide() {
    this.container.clear();  // Remove all views from the container
    this.isVisible = false;
  }
}
```

```html
<!-- Usage -->
<app-tooltip>
  <button trigger>Hover for info</button>
  <span tooltip>This is the tooltip content that appears on hover.</span>
</app-tooltip>
```

---

## `ContentChild` and `ContentChildren`

Where `ViewChild` accesses elements in the component's **own template**, `ContentChild` and `ContentChildren` access elements that the parent **projected into** the component. This is how a component can reach into the content it receives and interact with it programmatically.

```typescript
// tabs.component.ts — manages projected tab items
import { Component, ContentChildren, QueryList, AfterContentInit } from '@angular/core';
import { TabItemComponent } from './tab-item.component';

@Component({
  selector: 'app-tabs',
  template: `
    <div class="tab-headers">
      @for (tab of tabs; track tab.id) {
        <button
          [class.active]="tab === activeTab"
          (click)="activate(tab)"
        >{{ tab.title }}</button>
      }
    </div>
    <div class="tab-content">
      <!-- Projected tab content renders here -->
      <ng-content></ng-content>
    </div>
  `,
})
export class TabsComponent implements AfterContentInit {
  // Queries ALL projected TabItemComponent instances
  @ContentChildren(TabItemComponent)
  tabs!: QueryList<TabItemComponent>;

  activeTab!: TabItemComponent;

  ngAfterContentInit() {
    // Projected content is available here (NOT in ngOnInit)
    this.activeTab = this.tabs.first;
    this.tabs.forEach((tab, i) => tab.isActive = (i === 0));
  }

  activate(tab: TabItemComponent) {
    this.activeTab = tab;
    this.tabs.forEach(t => t.isActive = t === tab);
  }
}
```

> **Critical timing fact:** Projected content (`<ng-content>`) is only fully available in `ngAfterContentInit`, not in `ngOnInit`. This is because content projection happens after the component's own view is initialised but before `ngAfterContentInit`. Reading `@ContentChild` or `@ContentChildren` in `ngOnInit` will give you `undefined`.

---

## Angular CDK Portal — Advanced Dynamic Content

The Angular CDK (Component Dev Kit) provides a `Portal` abstraction that takes content rendering to the next level. A portal lets you render a component or template in an *entirely different location in the DOM* — including outside the Angular component tree, like in an overlay or a modal outlet.

```typescript
import { Component, inject } from '@angular/core';
import { ComponentPortal, TemplatePortal, PortalModule } from '@angular/cdk/portal';
import { Overlay } from '@angular/cdk/overlay';
import { ConfirmDialogComponent } from './confirm-dialog.component';

@Component({ ... })
export class AppComponent {
  private overlay = inject(Overlay);

  openConfirmDialog() {
    // Create an overlay (a DOM container appended to <body>)
    const overlayRef = this.overlay.create({
      positionStrategy: this.overlay.position().global().centerHorizontally().centerVertically(),
      hasBackdrop: true,
    });

    // Create a portal wrapping the ConfirmDialogComponent
    const portal = new ComponentPortal(ConfirmDialogComponent);

    // Attach the portal to the overlay — this renders ConfirmDialogComponent in the overlay
    const componentRef = overlayRef.attach(portal);

    // Communicate with the rendered component
    componentRef.instance.confirmed.subscribe(() => {
      overlayRef.dispose();
    });

    overlayRef.backdropClick().subscribe(() => overlayRef.dispose());
  }
}
```

CDK Portals are the foundation of Angular Material's dialogs, menus, and overlays. Understanding this pattern helps you build these kinds of UI components from scratch or extend the existing ones.

---

## The Compound Component Pattern

Content projection is the key enabler of **compound components** — a pattern where multiple related components work together through shared state, without the parent needing to wire them together manually.

```typescript
// accordion.component.ts — shares state with projected AccordionItemComponents
import { Component, ContentChildren, QueryList, AfterContentInit } from '@angular/core';
import { AccordionItemComponent } from './accordion-item.component';

@Component({
  selector: 'app-accordion',
  template: `<ng-content></ng-content>`,  // Just project the children — no layout markup
})
export class AccordionComponent implements AfterContentInit {
  @ContentChildren(AccordionItemComponent)
  items!: QueryList<AccordionItemComponent>;

  ngAfterContentInit() {
    // Each item needs a reference to the accordion so it can close siblings
    this.items.forEach(item => item.setAccordion(this));
  }

  closeAll() {
    this.items.forEach(item => item.close());
  }
}

// accordion-item.component.ts
@Component({
  selector: 'app-accordion-item',
  template: `
    <div class="accordion-item">
      <button class="accordion-trigger" (click)="toggle()">
        <ng-content select="[title]"></ng-content>
      </button>
      @if (isOpen) {
        <div class="accordion-content">
          <ng-content></ng-content>
        </div>
      }
    </div>
  `,
})
export class AccordionItemComponent {
  isOpen = false;
  private accordion?: AccordionComponent;

  setAccordion(accordion: AccordionComponent) {
    this.accordion = accordion;
  }

  toggle() {
    this.accordion?.closeAll();  // Close all siblings first
    this.isOpen = !this.isOpen;
  }

  close() {
    this.isOpen = false;
  }
}
```

```html
<!-- Usage is clean and semantic — the parent declares structure, not state -->
<app-accordion>
  <app-accordion-item>
    <span title>Section One</span>
    <p>Content for section one.</p>
  </app-accordion-item>
  <app-accordion-item>
    <span title>Section Two</span>
    <p>Content for section two.</p>
  </app-accordion-item>
</app-accordion>
```

---

## Key Best Practices

Use `<ng-content>` for simple slot-based APIs where the parent provides arbitrary content. Use `select` attribute selectors for structured multi-slot components like cards, panels, and modals — keep selectors semantic (use attribute selectors like `[card-header]` rather than fragile class selectors). Use `NgTemplateOutlet` when a component needs a *customisable rendering strategy* — the component controls the data and iteration, while the parent controls the visual rendering. Reserve `ViewContainerRef` and `TemplateRef` for directives or components that need fully programmatic control over when and where content appears. Remember the content lifecycle timing: always access `@ContentChild` and `@ContentChildren` in `ngAfterContentInit`, never in `ngOnInit`. Think of content projection as a way to separate concerns — the container component manages layout and behaviour, while the parent component manages the content and its appearance.

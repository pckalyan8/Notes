# Phase 13.2 — Angular CDK (Component Dev Kit)
## Complete Guide: The Foundation Beneath Angular Material

---

## What Is the Angular CDK?

The **Angular CDK** (Component Dev Kit) is a library of primitive building blocks that solve **common UI behavior patterns** without imposing any visual styling. Think of it as the engine room that powers Angular Material — but it's entirely independent and can be used on its own, even in projects that don't use Angular Material at all.

While Angular Material gives you fully styled, opinionated components (a blue button, a card with shadows), the CDK gives you the **raw behaviors** — "how do I trap keyboard focus inside a modal?", "how do I detect if an element is visible in the viewport?", "how do I build a drag-and-drop list?" — without any CSS attached.

The CDK is organized into distinct feature packages that you can import individually, keeping your bundle lean.

```bash
npm install @angular/cdk
```

---

## 1. Accessibility (A11y) Module — `@angular/cdk/a11y`

This is one of the most underused but most important parts of the CDK. It provides utilities that make your custom components truly accessible.

### FocusTrap — Keeping Focus Inside Modals

When a dialog or drawer opens, keyboard users should not be able to tab out of it and interact with the page behind it. `FocusTrap` enforces this constraint automatically.

```typescript
import { FocusTrap, FocusTrapFactory } from '@angular/cdk/a11y';
import { ElementRef, OnInit, OnDestroy, inject } from '@angular/core';

@Component({
  selector: 'app-modal',
  template: `
    <div class="modal-backdrop" (click)="close()">
      <div class="modal-panel" (click)="$event.stopPropagation()">
        <h2>Edit Profile</h2>
        <input type="text" placeholder="Your name">
        <button (click)="close()">Close</button>
      </div>
    </div>
  `
})
export class ModalComponent implements OnInit, OnDestroy {
  private el = inject(ElementRef);
  private focusTrapFactory = inject(FocusTrapFactory);
  private focusTrap!: FocusTrap;

  ngOnInit() {
    // Creates an invisible boundary around this component's DOM
    // Tab key will cycle only through elements inside this component
    this.focusTrap = this.focusTrapFactory.create(this.el.nativeElement);
    this.focusTrap.focusInitialElementWhenReady(); // auto-focus the first focusable element
  }

  ngOnDestroy() {
    this.focusTrap.destroy(); // Always clean up!
  }

  close() { /* ... */ }
}
```

> **Why this matters:** Without focus trapping, a screen reader or keyboard user pressing Tab inside an open dialog will silently move focus to elements hidden behind the overlay. This is a major accessibility failure. `FocusTrap` prevents it automatically.

### FocusMonitor — Detecting How Focus Was Applied

`FocusMonitor` can tell you *how* an element received focus — via mouse click, keyboard Tab, touch, or programmatic focus call. This is incredibly useful because you may want to show a focus ring when using the keyboard but not when clicking with a mouse (which is common in modern design systems).

```typescript
import { FocusMonitor, FocusOrigin } from '@angular/cdk/a11y';

@Component({
  template: `
    <button #myBtn [class.keyboard-focused]="isFocusedByKeyboard">
      Click or Tab to me
    </button>
  `
})
export class SmartButtonComponent implements AfterViewInit, OnDestroy {
  @ViewChild('myBtn') myBtn!: ElementRef;
  private focusMonitor = inject(FocusMonitor);

  isFocusedByKeyboard = false;

  ngAfterViewInit() {
    this.focusMonitor.monitor(this.myBtn).subscribe((origin: FocusOrigin) => {
      // origin is 'mouse', 'keyboard', 'touch', 'program', or null (blur)
      this.isFocusedByKeyboard = origin === 'keyboard';
    });
  }

  ngOnDestroy() {
    // Stop monitoring — prevents memory leaks
    this.focusMonitor.stopMonitoring(this.myBtn);
  }
}
```

### LiveAnnouncer — Speaking to Screen Readers

`LiveAnnouncer` injects text into an ARIA live region so that screen readers announce it to the user. This is essential when content changes happen without a page reload — like toasts, status updates, or form validation summaries.

```typescript
import { LiveAnnouncer } from '@angular/cdk/a11y';

@Component({ /* ... */ })
export class DataTableComponent {
  private liveAnnouncer = inject(LiveAnnouncer);

  async loadData() {
    this.liveAnnouncer.announce('Loading data, please wait...');
    await this.fetchData();
    // 'polite' waits for the user to finish their current interaction
    // 'assertive' interrupts immediately — use sparingly
    this.liveAnnouncer.announce('Data loaded. 42 results found.', 'polite');
  }

  onSortChange(column: string, direction: string) {
    this.liveAnnouncer.announce(
      `Table sorted by ${column} in ${direction} order`
    );
  }
}
```

---

## 2. DragDrop Module — `@angular/cdk/drag-drop`

The CDK DragDrop module lets you build fully accessible, animated drag-and-drop interfaces without a third-party library.

### Basic Draggable Item

```html
<div cdkDrag class="drag-item">
  <mat-icon cdkDragHandle>drag_indicator</mat-icon>
  Drag me around!
</div>
```

`cdkDrag` makes the element draggable. `cdkDragHandle` (optional) restricts dragging to a specific handle element — if omitted, the whole element is the handle.

### Sortable List

The most common use case: reordering a list by dragging items.

```typescript
import { CdkDragDrop, moveItemInArray, CdkDropList, CdkDrag } from '@angular/cdk/drag-drop';

@Component({
  imports: [CdkDropList, CdkDrag],
  template: `
    <div cdkDropList class="task-list" (cdkDropListDropped)="onDrop($event)">
      @for (task of tasks; track task.id) {
        <div cdkDrag class="task-item">
          {{ task.title }}
          <!-- Placeholder shown while item is being dragged -->
          <div class="drag-placeholder" *cdkDragPlaceholder></div>
          <!-- Preview shown while dragging (follows cursor) -->
          <div *cdkDragPreview>
            <span>{{ task.title }}</span>
          </div>
        </div>
      }
    </div>
  `
})
export class TaskListComponent {
  tasks = [
    { id: 1, title: 'Design mockups' },
    { id: 2, title: 'Write tests' },
    { id: 3, title: 'Deploy to staging' },
  ];

  onDrop(event: CdkDragDrop<typeof this.tasks>) {
    // moveItemInArray mutates the array in-place
    // event.previousIndex and event.currentIndex tell you what moved where
    moveItemInArray(this.tasks, event.previousIndex, event.currentIndex);
    // Now persist the new order to your API
    this.saveOrder(this.tasks.map(t => t.id));
  }
}
```

### Transferring Items Between Lists

```typescript
import { transferArrayItem, CdkDragDrop } from '@angular/cdk/drag-drop';

@Component({
  template: `
    <!-- Connected lists can accept items from each other -->
    <div cdkDropList
         #todoList="cdkDropList"
         [cdkDropListData]="todo"
         [cdkDropListConnectedTo]="[doneList]"
         (cdkDropListDropped)="drop($event)">
      @for (item of todo; track item) {
        <div cdkDrag>{{ item }}</div>
      }
    </div>

    <div cdkDropList
         #doneList="cdkDropList"
         [cdkDropListData]="done"
         [cdkDropListConnectedTo]="[todoList]"
         (cdkDropListDropped)="drop($event)">
      @for (item of done; track item) {
        <div cdkDrag>{{ item }}</div>
      }
    </div>
  `
})
export class KanbanComponent {
  todo = ['Write feature spec', 'Design UI'];
  done = ['Set up project', 'Configure CI'];

  drop(event: CdkDragDrop<string[]>) {
    if (event.previousContainer === event.container) {
      // Same list — just reorder
      moveItemInArray(event.container.data, event.previousIndex, event.currentIndex);
    } else {
      // Different lists — transfer the item
      transferArrayItem(
        event.previousContainer.data,
        event.container.data,
        event.previousIndex,
        event.currentIndex
      );
    }
  }
}
```

---

## 3. Overlay Module — `@angular/cdk/overlay`

The Overlay module is how Angular Material builds tooltips, dialogs, dropdowns, and select menus. It gives you a way to render content **above** the normal document flow, positioned correctly relative to a trigger element.

### Understanding Overlay Concepts

An overlay consists of three key decisions:
1. **What to render** — a `TemplateRef` or a `ComponentRef`
2. **Where to position it** — using a position strategy (global or connected)
3. **When to close it** — using a scroll strategy

```typescript
import { Overlay, OverlayRef, OverlayConfig } from '@angular/cdk/overlay';
import { TemplatePortal } from '@angular/cdk/portal';

@Component({
  template: `
    <button (click)="openPanel()" #triggerEl>Open Panel</button>

    <ng-template #panelTemplate>
      <div class="panel">
        <p>This is the overlay content!</p>
        <button (click)="closePanel()">Close</button>
      </div>
    </ng-template>
  `
})
export class DropdownComponent implements OnDestroy {
  @ViewChild('panelTemplate') panelTemplate!: TemplateRef<unknown>;
  @ViewChild('triggerEl', { read: ElementRef }) trigger!: ElementRef;

  private overlay = inject(Overlay);
  private vcr = inject(ViewContainerRef);
  private overlayRef: OverlayRef | null = null;

  openPanel() {
    // Define the position: connect the overlay to the trigger element
    const positionStrategy = this.overlay
      .position()
      .flexibleConnectedTo(this.trigger)  // attach to the button
      .withPositions([
        {
          originX: 'start', originY: 'bottom',   // of the trigger
          overlayX: 'start', overlayY: 'top',    // of the overlay
          offsetY: 8,  // 8px gap between trigger and panel
        },
        {
          // Fallback: if no space below, try above
          originX: 'start', originY: 'top',
          overlayX: 'start', overlayY: 'bottom',
          offsetY: -8,
        }
      ]);

    this.overlayRef = this.overlay.create(new OverlayConfig({
      positionStrategy,
      scrollStrategy: this.overlay.scrollStrategies.reposition(), // reposition on scroll
      hasBackdrop: true,      // clicking outside closes the panel
      backdropClass: 'cdk-overlay-transparent-backdrop',
    }));

    // Attach the template to the overlay
    const portal = new TemplatePortal(this.panelTemplate, this.vcr);
    this.overlayRef.attach(portal);

    // Close when backdrop is clicked
    this.overlayRef.backdropClick().subscribe(() => this.closePanel());
  }

  closePanel() {
    this.overlayRef?.detach();
  }

  ngOnDestroy() {
    this.overlayRef?.dispose();
  }
}
```

> **When to use this directly:** Most of the time you'll use Angular Material components (Dialog, Menu, Tooltip) which are built on top of this. Use the raw Overlay API when building your own custom dropdown, contextual menu, or popover that doesn't fit Angular Material's existing components.

---

## 4. Portal Module — `@angular/cdk/portal`

Portals solve the problem of rendering a component or template **in a different location in the DOM** than where it's declared. This is useful for sidebars, teleporting content, and — importantly — it powers the Overlay system.

### Types of Portals

There are three portal types:

`TemplatePortal` renders an `ng-template` at a different location. `ComponentPortal` renders a component dynamically. `DomPortal` moves an existing DOM element to a new location.

```typescript
import { Portal, TemplatePortal, ComponentPortal } from '@angular/cdk/portal';
import { CdkPortalOutlet } from '@angular/cdk/portal';

@Component({
  imports: [CdkPortalOutlet],
  template: `
    <!-- The portal outlet defines WHERE content will be rendered -->
    <h3>Sidebar</h3>
    <ng-template [cdkPortalOutlet]="activeSidebarPortal"></ng-template>

    <!-- Define the source template somewhere else in the app -->
    <ng-template #myTemplate>
      <app-user-profile [userId]="currentUserId" />
    </ng-template>
  `
})
export class AppShellComponent {
  @ViewChild('myTemplate') myTemplate!: TemplateRef<unknown>;
  private vcr = inject(ViewContainerRef);

  activeSidebarPortal!: Portal<unknown>;

  openUserProfile() {
    // Create a portal from the template
    this.activeSidebarPortal = new TemplatePortal(this.myTemplate, this.vcr);
  }

  openSettingsPanel() {
    // Or create a portal from a component class (no template needed)
    this.activeSidebarPortal = new ComponentPortal(SettingsComponent);
  }
}
```

---

## 5. Virtual Scrolling — `@angular/cdk/scrolling`

Virtual scrolling is a performance technique where only the visible rows in a list are actually rendered in the DOM. A list with 10,000 items renders only ~20 items at any time, keeping memory and render time constant.

```typescript
import { ScrollingModule } from '@angular/cdk/scrolling';

@Component({
  imports: [ScrollingModule],
  template: `
    <!-- 
      itemSize: the fixed height of each item in pixels.
      This is REQUIRED — the virtual scroller needs to know the size
      to calculate how many items fit in the viewport.
    -->
    <cdk-virtual-scroll-viewport itemSize="72" class="user-list-viewport">
      <div *cdkVirtualFor="let user of users; trackBy: trackByUser"
           class="user-row">
        <img [src]="user.avatar" [alt]="user.name">
        <span>{{ user.name }}</span>
        <span>{{ user.email }}</span>
      </div>
    </cdk-virtual-scroll-viewport>
  `,
  styles: [`
    .user-list-viewport {
      height: 600px;   /* The visible height of the list */
      width: 100%;
    }
    .user-row {
      height: 72px;    /* Must match itemSize exactly */
      display: flex;
      align-items: center;
      gap: 12px;
    }
  `]
})
export class LargeUserListComponent {
  users: User[] = this.userService.getAllUsers(); // Could be 100,000 users!

  trackByUser(index: number, user: User) {
    return user.id; // Essential for performance — reuse DOM nodes
  }
}
```

> **Important limitation:** `CdkVirtualScrollViewport` with `cdkVirtualFor` requires all items to have the **same fixed height**. For variable-height items, you need `AutoSizeVirtualScrollStrategy` (more complex) or a third-party library like `@angular/cdk-experimental`.

---

## 6. Stepper — `@angular/cdk/stepper`

The CDK Stepper provides the navigation logic and state management for multi-step forms, without any styling. Angular Material's `mat-stepper` is built on top of this.

```typescript
import { CdkStepper } from '@angular/cdk/stepper';

// Build a completely custom stepper with CDK logic but your own styles
@Component({
  selector: 'app-custom-stepper',
  providers: [{ provide: CdkStepper, useExisting: CustomStepperComponent }],
  template: `
    <div class="step-indicators">
      @for (step of steps; track step; let i = $index) {
        <div class="step-dot"
             [class.active]="selectedIndex === i"
             [class.completed]="step.completed"
             (click)="selectedIndex = i">
          {{ i + 1 }}
        </div>
      }
    </div>

    <!-- Render the currently selected step's content -->
    @if (selected) {
      <div class="step-content">
        <ng-container [ngTemplateOutlet]="selected.content"></ng-container>
      </div>
    }

    <div class="step-actions">
      <button [disabled]="!hasPreviousStep" (click)="previous()">Back</button>
      <button [disabled]="!hasNextStep" (click)="next()">Next</button>
    </div>
  `
})
export class CustomStepperComponent extends CdkStepper { }
```

---

## 7. Layout — `@angular/cdk/layout`

The Layout module provides tools for responding to changes in viewport size — useful for adapting layouts based on screen width, similar to CSS media queries but in TypeScript.

```typescript
import { BreakpointObserver, Breakpoints } from '@angular/cdk/layout';

@Component({ /* ... */ })
export class ResponsiveLayoutComponent {
  private breakpointObserver = inject(BreakpointObserver);

  // Observe multiple breakpoints at once
  readonly isMobile$ = this.breakpointObserver.observe([
    Breakpoints.XSmall,  // '(max-width: 599.98px)'
    Breakpoints.Small,   // '(min-width: 600px) and (max-width: 959.98px)'
  ]).pipe(
    map(result => result.matches)
  );

  // You can also use custom breakpoints
  readonly isWideScreen$ = this.breakpointObserver.observe('(min-width: 1400px)').pipe(
    map(result => result.matches)
  );

  // Convert to signal using toSignal for template use
  isMobile = toSignal(this.isMobile$, { initialValue: false });
}
```

```html
@if (isMobile()) {
  <app-mobile-nav />
} @else {
  <app-desktop-nav />
}
```

The built-in `Breakpoints` constants follow Material Design's breakpoint system. Available breakpoints include `XSmall`, `Small`, `Medium`, `Large`, `XLarge`, `Handset`, `Tablet`, `Web`, `HandsetPortrait`, `HandsetLandscape`, etc.

---

## 8. Text Field — `@angular/cdk/text-field`

Provides utilities for textarea auto-sizing and input autofill detection.

```typescript
import { TextFieldModule } from '@angular/cdk/text-field';

@Component({
  imports: [TextFieldModule],
  template: `
    <!-- 
      cdkTextareaAutosize: automatically grows the textarea as the user types
      cdkAutosizeMinRows / cdkAutosizeMaxRows: control the min/max height
    -->
    <textarea cdkTextareaAutosize
              cdkAutosizeMinRows="3"
              cdkAutosizeMaxRows="10"
              placeholder="Write your message...">
    </textarea>
  `
})
```

---

## 9. Table — `@angular/cdk/table`

The CDK Table is the unstyled version of `mat-table`. If Angular Material's design doesn't fit your project, you can use the CDK Table to build your own data grid with full sorting, pagination, and filtering, applying your own CSS.

```typescript
import { CdkTable, CdkColumnDef, CdkHeaderCellDef, CdkCellDef,
         CdkHeaderRowDef, CdkRowDef } from '@angular/cdk/table';

@Component({
  imports: [CdkTable, CdkColumnDef, CdkHeaderCellDef, CdkCellDef,
            CdkHeaderRowDef, CdkRowDef],
  template: `
    <cdk-table [dataSource]="dataSource">
      <ng-container cdkColumnDef="name">
        <th cdk-header-cell *cdkHeaderCellDef>Name</th>
        <td cdk-cell *cdkCellDef="let row">{{ row.name }}</td>
      </ng-container>

      <tr cdk-header-row *cdkHeaderRowDef="columns"></tr>
      <tr cdk-row *cdkRowDef="let row; columns: columns;"></tr>
    </cdk-table>
  `
})
```

---

## 10. Clipboard — `@angular/cdk/clipboard`

```typescript
import { ClipboardModule } from '@angular/cdk/clipboard';

@Component({
  imports: [ClipboardModule],
  template: `
    <!-- Directive approach: copies text on click -->
    <button [cdkCopyToClipboard]="apiKey" (cdkCopyToClipboardCopied)="onCopied($event)">
      Copy API Key
    </button>
  `
})
export class ApiKeyComponent {
  apiKey = 'sk-1234567890abcdef';

  onCopied(success: boolean) {
    if (success) {
      console.log('Copied successfully!');
    }
  }
}
```

For programmatic use:

```typescript
import { Clipboard } from '@angular/cdk/clipboard';

@Component({ /* ... */ })
export class ShareComponent {
  private clipboard = inject(Clipboard);

  copyUrl() {
    const pending = this.clipboard.beginCopy(window.location.href);
    let remainingAttempts = 3;

    const attempt = () => {
      const result = pending.copy();
      if (!result && --remainingAttempts) {
        setTimeout(attempt, 100); // retry if clipboard was busy
      } else {
        pending.destroy();
      }
    };

    attempt();
  }
}
```

---

## Important Points & Best Practices

**The CDK is your alternative to jQuery and random npm packages.** Before adding a third-party drag-drop library, a virtual scroll library, or a focus management library, check if the CDK already has it. It integrates seamlessly with Angular's change detection and dependency injection.

**`FocusTrap` and `LiveAnnouncer` are non-negotiable for any custom modal or notification system.** Angular Material uses them automatically, but if you build custom modals or toasts, you must add these yourself to stay accessible.

**Virtual scrolling requires fixed item heights for simple usage.** Design your list items with a known, consistent height and set it in both the `itemSize` attribute and CSS. Mismatched heights cause visual glitches.

**When using `Overlay`, always call `overlayRef.dispose()` in `ngOnDestroy`.** Forgetting this causes memory leaks where overlay DOM elements accumulate in the document body, especially in SPAs where components are created and destroyed frequently.

**`BreakpointObserver` is preferred over window resize listeners** because it uses `matchMedia` internally, which is more efficient (no polling, no scroll handler) and automatically cleans up when the component is destroyed.

**The CDK is designed to work with Zone.js or zoneless equally well.** All CDK observables and events integrate correctly with both change detection strategies, making it future-proof for zoneless Angular apps.

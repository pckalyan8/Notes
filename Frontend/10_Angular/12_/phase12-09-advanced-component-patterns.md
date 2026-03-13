# Phase 12.9 — Advanced Component Patterns

> **Prerequisites:** Angular components, DI, content projection (Phase 11.6), directives (Phase 11.8)

---

## Table of Contents
1. [Slot-Based Design with ng-content (Advanced)](#1-slot-based-design-with-ng-content-advanced)
2. [Dynamic Component Loader Pattern](#2-dynamic-component-loader-pattern)
3. [Angular CDK Overview](#3-angular-cdk-overview)
4. [CDK Overlay — Positioning Engine](#4-cdk-overlay--positioning-engine)
5. [CDK DragDrop — Drag and Drop](#5-cdk-dragdrop--drag-and-drop)
6. [CDK Virtual Scroll (Advanced)](#6-cdk-virtual-scroll-advanced)
7. [CDK Table — Data Table Foundation](#7-cdk-table--data-table-foundation)
8. [CDK Stepper](#8-cdk-stepper)
9. [Portals and Overlays](#9-portals-and-overlays)
10. [Compound Component Pattern with InjectionToken](#10-compound-component-pattern-with-injectiontoken)
11. [Headless UI Components](#11-headless-ui-components)
12. [Best Practices](#12-best-practices)

---

## 1. Slot-Based Design with ng-content (Advanced)

### Fallback Content for Empty Slots

```typescript
@Component({
  selector: 'app-panel',
  standalone: true,
  template: `
    <div class="panel">
      <div class="panel__header">
        <ng-content select="[panel-header]">
          <!-- Default header if nothing projected -->
          <span>Panel</span>
        </ng-content>
      </div>
      <div class="panel__body">
        <ng-content></ng-content>
      </div>
      <div class="panel__footer" *ngIf="hasFooter">
        <ng-content select="[panel-footer]"></ng-content>
      </div>
    </div>
  `,
})
export class PanelComponent {
  // Check if footer content was projected
  @ContentChild('[panel-footer]') footerContent?: ElementRef;
  hasFooter = !!this.footerContent;
}
```

### `@ContentChildren` for Dynamic Slot Rendering

```typescript
@Component({
  selector: 'app-menu',
  standalone: true,
  template: `
    <nav role="navigation">
      <ul>
        @for (item of menuItems; track item) {
          <li role="menuitem">
            <ng-container [ngTemplateOutlet]="item.templateRef"></ng-container>
          </li>
        }
      </ul>
    </nav>
  `,
})
export class MenuComponent implements AfterContentInit {
  @ContentChildren(MenuItemDirective) menuItems!: QueryList<MenuItemDirective>;
}

@Directive({ selector: '[appMenuItem]', standalone: true })
export class MenuItemDirective {
  templateRef = inject(TemplateRef);
}
```

---

## 2. Dynamic Component Loader Pattern

### Loading Components at Runtime

```typescript
import { ComponentRef, ViewContainerRef, Type, inject } from '@angular/core';

@Component({
  selector: 'app-widget-host',
  standalone: true,
  template: `<div #container></div>`,
})
export class WidgetHostComponent implements OnDestroy {
  @ViewChild('container', { read: ViewContainerRef }) container!: ViewContainerRef;

  private componentRef?: ComponentRef<any>;

  async loadWidget(widgetType: string) {
    this.container.clear();
    this.componentRef?.destroy();

    // Dynamic import — lazy loads the widget chunk
    const component = await this.resolveWidget(widgetType);

    this.componentRef = this.container.createComponent(component);

    // Pass data via setInput (signal-compatible)
    this.componentRef.setInput('data', this.widgetData);
    this.componentRef.setInput('config', this.widgetConfig);
  }

  private async resolveWidget(type: string): Promise<Type<any>> {
    switch (type) {
      case 'chart':   return import('./widgets/chart.component').then(m => m.ChartComponent);
      case 'table':   return import('./widgets/table.component').then(m => m.TableComponent);
      case 'gauge':   return import('./widgets/gauge.component').then(m => m.GaugeComponent);
      default: throw new Error(`Unknown widget: ${type}`);
    }
  }

  ngOnDestroy() {
    this.componentRef?.destroy();
  }
}
```

### Component Factory with Providers

```typescript
// Inject custom providers into dynamically created component
const environmentInjector = inject(EnvironmentInjector);

const customInjector = createEnvironmentInjector(
  [{ provide: WIDGET_CONFIG, useValue: this.config }],
  environmentInjector
);

const ref = this.container.createComponent(WidgetComponent, {
  environmentInjector: customInjector,
});
```

### Type-Safe Dynamic Components

```typescript
export interface DynamicWidget {
  data: signal<any>;
  refresh(): void;
}

@Component({ ... })
export class ChartWidget implements DynamicWidget {
  data = input.required();
  refresh() { this.reloadChart(); }
}

// Type-safe reference
const ref = this.container.createComponent(ChartWidget);
const instance: DynamicWidget = ref.instance;
instance.refresh();
```

---

## 3. Angular CDK Overview

The **Component Dev Kit (CDK)** is Angular's toolkit for building UI primitives — the foundation that Angular Material is built on. Use it directly when you need low-level building blocks without Material's opinionated styling.

### CDK Packages

| Package | Purpose |
|---|---|
| `@angular/cdk/overlay` | Positioning floating elements |
| `@angular/cdk/drag-drop` | Drag and drop lists |
| `@angular/cdk/virtual-scroll` | Render large lists efficiently |
| `@angular/cdk/table` | Data table rendering engine |
| `@angular/cdk/stepper` | Multi-step wizard |
| `@angular/cdk/a11y` | Focus management, screen readers |
| `@angular/cdk/portal` | Render content outside the component tree |
| `@angular/cdk/clipboard` | Clipboard operations |
| `@angular/cdk/layout` | Breakpoint observation |
| `@angular/cdk/text-field` | Auto-sizing textareas |

---

## 4. CDK Overlay — Positioning Engine

`CdkOverlay` creates DOM elements floating above the page — used for tooltips, dropdowns, modals, and context menus:

```typescript
import { Overlay, OverlayRef, OverlayModule } from '@angular/cdk/overlay';
import { ComponentPortal } from '@angular/cdk/portal';

@Component({
  standalone: true,
  imports: [OverlayModule],
  template: `
    <button #trigger (click)="toggle()" cdkOverlayOrigin #origin="cdkOverlayOrigin">
      Open Menu
    </button>

    <!-- Angular CDK Connected Overlay — declarative approach -->
    <ng-template
      cdkConnectedOverlay
      [cdkConnectedOverlayOrigin]="origin"
      [cdkConnectedOverlayOpen]="isOpen()"
      [cdkConnectedOverlayPositions]="positions"
      (overlayOutsideClick)="isOpen.set(false)"
    >
      <div class="dropdown-menu">
        <button (click)="onAction('edit')">Edit</button>
        <button (click)="onAction('delete')">Delete</button>
      </div>
    </ng-template>
  `,
})
export class DropdownComponent {
  isOpen = signal(false);

  positions: ConnectedPosition[] = [
    { originX: 'start', originY: 'bottom', overlayX: 'start', overlayY: 'top' },
    { originX: 'start', originY: 'top', overlayX: 'start', overlayY: 'bottom' },
  ];

  toggle() { this.isOpen.update(v => !v); }
}
```

### Programmatic Overlay (for services)

```typescript
@Injectable({ providedIn: 'root' })
export class TooltipService {
  private overlay = inject(Overlay);

  show(text: string, origin: ElementRef): OverlayRef {
    const positionStrategy = this.overlay.position()
      .flexibleConnectedTo(origin)
      .withPositions([{
        originX: 'center', originY: 'top',
        overlayX: 'center', overlayY: 'bottom',
        offsetY: -8,
      }]);

    const overlayRef = this.overlay.create({
      positionStrategy,
      scrollStrategy: this.overlay.scrollStrategies.reposition(),
      hasBackdrop: false,
    });

    const tooltipPortal = new ComponentPortal(TooltipComponent);
    const tooltipRef = overlayRef.attach(tooltipPortal);
    tooltipRef.instance.text = text;

    return overlayRef; // Call overlayRef.dispose() to close
  }
}
```

---

## 5. CDK DragDrop — Drag and Drop

```typescript
import { DragDropModule, CdkDragDrop, moveItemInArray, transferArrayItem } from '@angular/cdk/drag-drop';

@Component({
  standalone: true,
  imports: [DragDropModule],
  template: `
    <!-- Simple sortable list -->
    <ul cdkDropList (cdkDropListDropped)="reorder($event)">
      @for (task of tasks(); track task.id) {
        <li cdkDrag>
          <span cdkDragHandle>⠿</span>
          {{ task.title }}
        </li>
      }
    </ul>

    <!-- Multi-list transfer (Kanban board) -->
    <div class="kanban">
      @for (column of columns(); track column.id) {
        <div
          cdkDropList
          [id]="column.id"
          [cdkDropListData]="column.tasks"
          [cdkDropListConnectedTo]="connectedLists()"
          (cdkDropListDropped)="onDrop($event)"
          class="kanban-column"
        >
          <h3>{{ column.title }}</h3>
          @for (task of column.tasks; track task.id) {
            <div cdkDrag class="task-card">{{ task.title }}</div>
          }
        </div>
      }
    </div>
  `,
})
export class KanbanBoardComponent {
  tasks = signal<Task[]>([...]);
  columns = signal<Column[]>([...]);

  // IDs of all connected lists (so items can be transferred between them)
  connectedLists = computed(() => this.columns().map(c => c.id));

  reorder(event: CdkDragDrop<Task[]>) {
    this.tasks.update(list => {
      const reordered = [...list];
      moveItemInArray(reordered, event.previousIndex, event.currentIndex);
      return reordered;
    });
  }

  onDrop(event: CdkDragDrop<Task[]>) {
    if (event.previousContainer === event.container) {
      moveItemInArray(event.container.data, event.previousIndex, event.currentIndex);
    } else {
      transferArrayItem(
        event.previousContainer.data,
        event.container.data,
        event.previousIndex,
        event.currentIndex,
      );
    }
    // Trigger signal update
    this.columns.update(cols => [...cols]);
  }
}
```

---

## 6. CDK Virtual Scroll (Advanced)

```typescript
import { ScrollingModule, CdkVirtualScrollViewport } from '@angular/cdk/scrolling';

@Component({
  standalone: true,
  imports: [ScrollingModule],
  template: `
    <cdk-virtual-scroll-viewport
      itemSize="60"
      class="list-viewport"
      (scrolledIndexChange)="onScrollIndexChange($event)"
    >
      <div *cdkVirtualFor="let item of dataSource; trackBy: trackItem">
        <app-list-item [item]="item" />
      </div>
    </cdk-virtual-scroll-viewport>
  `,
})
export class VirtualListComponent {
  @ViewChild(CdkVirtualScrollViewport) viewport!: CdkVirtualScrollViewport;

  items = signal<Item[]>(largeDataset);

  // Scroll to specific item programmatically
  scrollToItem(index: number) {
    this.viewport.scrollToIndex(index, 'smooth');
  }

  onScrollIndexChange(index: number) {
    // Load more data when near the bottom (infinite scroll)
    if (index > this.items().length - 20) {
      this.loadMoreItems();
    }
  }

  trackItem = (index: number, item: Item) => item.id;
}
```

### Custom Virtual Scroll Strategy

```typescript
import { VirtualScrollStrategy, VIRTUAL_SCROLL_STRATEGY } from '@angular/cdk/scrolling';

// Variable-height items
@Injectable()
export class VariableHeightScrollStrategy implements VirtualScrollStrategy {
  // Custom implementation for items with different heights
  // Useful for chat messages, feed items, etc.
}

@Component({
  providers: [
    { provide: VIRTUAL_SCROLL_STRATEGY, useClass: VariableHeightScrollStrategy }
  ],
  ...
})
export class VariableHeightListComponent {}
```

---

## 7. CDK Table — Data Table Foundation

The CDK Table provides flexible table rendering without opinionated styling:

```typescript
import { CdkTableModule } from '@angular/cdk/table';

interface User { id: string; name: string; email: string; role: string; }

@Component({
  standalone: true,
  imports: [CdkTableModule],
  template: `
    <table cdk-table [dataSource]="dataSource" class="data-table">
      <!-- Name Column -->
      <ng-container cdkColumnDef="name">
        <th cdk-header-cell *cdkHeaderCellDef (click)="sortBy('name')">Name ↕</th>
        <td cdk-cell *cdkCellDef="let user">{{ user.name }}</td>
      </ng-container>

      <!-- Email Column -->
      <ng-container cdkColumnDef="email">
        <th cdk-header-cell *cdkHeaderCellDef>Email</th>
        <td cdk-cell *cdkCellDef="let user">{{ user.email }}</td>
      </ng-container>

      <!-- Actions Column -->
      <ng-container cdkColumnDef="actions">
        <th cdk-header-cell *cdkHeaderCellDef>Actions</th>
        <td cdk-cell *cdkCellDef="let user">
          <button (click)="edit(user)">Edit</button>
          <button (click)="delete(user)">Delete</button>
        </td>
      </ng-container>

      <!-- Row Definitions -->
      <tr cdk-header-row *cdkHeaderRowDef="displayedColumns"></tr>
      <tr cdk-row *cdkRowDef="let row; columns: displayedColumns;"
          [class.selected]="isSelected(row)">
      </tr>

      <!-- Empty state -->
      <tr class="mat-row" *cdkNoDataRow>
        <td class="mat-cell" [attr.colspan]="displayedColumns.length">
          No data available
        </td>
      </tr>
    </table>
  `,
})
export class DataTableComponent {
  displayedColumns = ['name', 'email', 'actions'];
  dataSource = signal<User[]>([]);

  sortBy(column: string) { /* ... */ }
  edit(user: User) { /* ... */ }
  delete(user: User) { /* ... */ }
  isSelected(user: User) { return false; }
}
```

---

## 8. CDK Stepper

```typescript
import { CdkStepperModule, CdkStepper } from '@angular/cdk/stepper';

@Component({
  selector: 'app-custom-stepper',
  standalone: true,
  imports: [CdkStepperModule],
  template: `
    <cdk-stepper #stepper [linear]="isLinear" orientation="horizontal">

      <cdk-step [stepControl]="personalForm" label="Personal Info">
        <form [formGroup]="personalForm">
          <input formControlName="firstName" placeholder="First Name" />
          <input formControlName="lastName" placeholder="Last Name" />
          <button cdkStepperNext [disabled]="personalForm.invalid">Next</button>
        </form>
      </cdk-step>

      <cdk-step [stepControl]="addressForm" label="Address">
        <form [formGroup]="addressForm">
          <input formControlName="street" placeholder="Street" />
          <input formControlName="city" placeholder="City" />
          <button cdkStepperPrevious>Back</button>
          <button cdkStepperNext [disabled]="addressForm.invalid">Next</button>
        </form>
      </cdk-step>

      <cdk-step label="Review">
        <p>Name: {{ personalForm.value.firstName }} {{ personalForm.value.lastName }}</p>
        <p>Address: {{ addressForm.value.street }}, {{ addressForm.value.city }}</p>
        <button cdkStepperPrevious>Back</button>
        <button (click)="submit()">Submit</button>
      </cdk-step>

    </cdk-stepper>
  `,
})
export class RegistrationStepperComponent {
  isLinear = true;
  // Form groups here...
}
```

---

## 9. Portals and Overlays

**Portal** = content to be rendered somewhere else.
**PortalOutlet** = the location where content is rendered.

### ComponentPortal

```typescript
import { Portal, ComponentPortal, PortalModule } from '@angular/cdk/portal';

// The portal outlet (destination)
@Component({
  standalone: true,
  imports: [PortalModule],
  template: `
    <div class="sidebar">
      <ng-template [cdkPortalOutlet]="sidebarPortal()"></ng-template>
    </div>
  `,
})
export class AppComponent {
  private portalService = inject(PortalService);
  sidebarPortal = this.portalService.sidebarPortal;
}

// Service manages what's in the sidebar
@Injectable({ providedIn: 'root' })
export class PortalService {
  sidebarPortal = signal<Portal<any> | null>(null);

  showInSidebar(component: Type<any>) {
    this.sidebarPortal.set(new ComponentPortal(component));
  }

  clearSidebar() {
    this.sidebarPortal.set(null);
  }
}
```

### TemplatePortal

```typescript
@Component({
  standalone: true,
  imports: [PortalModule],
  template: `
    <!-- Template defined here... -->
    <ng-template #myTemplate>
      <p>This content will be teleported!</p>
    </ng-template>

    <!-- ...rendered in the outlet (which could be in another component) -->
    <ng-template [cdkPortalOutlet]="portal"></ng-template>
  `,
})
export class PortalDemoComponent implements AfterViewInit {
  @ViewChild('myTemplate') templateRef!: TemplateRef<any>;
  private viewContainerRef = inject(ViewContainerRef);
  portal!: TemplatePortal;

  ngAfterViewInit() {
    this.portal = new TemplatePortal(this.templateRef, this.viewContainerRef);
  }
}
```

---

## 10. Compound Component Pattern with InjectionToken

The compound component pattern uses DI to share state between parent and child components **without prop drilling**:

```typescript
// tabs.token.ts
export interface TabsContext {
  activeTab: Signal<string>;
  setActive(id: string): void;
  registerTab(id: string, label: string): void;
}

export const TABS_CONTEXT = new InjectionToken<TabsContext>('TABS_CONTEXT');

// tabs.component.ts — parent provides the context
@Component({
  selector: 'app-tabs',
  standalone: true,
  providers: [{
    provide: TABS_CONTEXT,
    useFactory: () => {
      const activeTab = signal('');
      const tabs = signal<{ id: string; label: string }[]>([]);
      return {
        activeTab,
        setActive: (id: string) => activeTab.set(id),
        registerTab: (id, label) => {
          tabs.update(t => [...t, { id, label }]);
          if (!activeTab()) activeTab.set(id); // First tab is active by default
        },
      };
    },
  }],
  template: `
    <div class="tabs">
      <div class="tab-list" role="tablist">
        @for (tab of tabs(); track tab.id) {
          <button
            role="tab"
            [attr.aria-selected]="ctx.activeTab() === tab.id"
            (click)="ctx.setActive(tab.id)"
          >
            {{ tab.label }}
          </button>
        }
      </div>
      <div class="tab-content">
        <ng-content></ng-content>
      </div>
    </div>
  `,
})
export class TabsComponent {
  ctx = inject(TABS_CONTEXT);
  tabs = computed(() => /* from context */);
}

// tab-panel.component.ts — child injects the context
@Component({
  selector: 'app-tab-panel',
  standalone: true,
  template: `
    @if (ctx.activeTab() === id()) {
      <div role="tabpanel">
        <ng-content></ng-content>
      </div>
    }
  `,
})
export class TabPanelComponent implements OnInit {
  id = input.required<string>();
  label = input.required<string>();
  ctx = inject(TABS_CONTEXT);

  ngOnInit() {
    this.ctx.registerTab(this.id(), this.label());
  }
}
```

Usage:
```html
<app-tabs>
  <app-tab-panel id="overview" label="Overview">
    <p>Product overview content...</p>
  </app-tab-panel>
  <app-tab-panel id="specs" label="Specifications">
    <p>Technical specs...</p>
  </app-tab-panel>
  <app-tab-panel id="reviews" label="Reviews">
    <app-reviews [productId]="product.id" />
  </app-tab-panel>
</app-tabs>
```

---

## 11. Headless UI Components

**Headless UI** = behavior and accessibility, zero styling. Consumer provides 100% of the visual design.

```typescript
@Component({
  selector: 'app-headless-select',
  standalone: true,
  template: `
    <!-- No styling at all — consumer styles via ::ng-deep or CSS custom properties -->
    <div
      class="select-container"
      [attr.aria-expanded]="isOpen()"
      [attr.aria-haspopup]="true"
      role="combobox"
    >
      <div
        class="select-trigger"
        tabindex="0"
        (click)="toggle()"
        (keydown.enter)="toggle()"
        (keydown.space)="toggle()"
        (keydown.escape)="close()"
      >
        <ng-content select="[trigger]"></ng-content>
      </div>

      @if (isOpen()) {
        <div class="select-dropdown" role="listbox" (keydown)="onKeydown($event)">
          <ng-content select="[options]"></ng-content>
        </div>
      }
    </div>
  `,
})
export class HeadlessSelectComponent {
  isOpen = signal(false);

  toggle() { this.isOpen.update(v => !v); }
  close() { this.isOpen.set(false); }

  onKeydown(event: KeyboardEvent) {
    if (event.key === 'Escape') this.close();
    if (event.key === 'ArrowDown') this.focusNext();
    if (event.key === 'ArrowUp') this.focusPrev();
  }
}
```

Consumer applies styling:
```html
<app-headless-select>
  <div trigger class="my-custom-trigger">
    Select a country...
  </div>
  <div options class="my-dropdown">
    <option value="us">United States</option>
    <option value="uk">United Kingdom</option>
  </div>
</app-headless-select>
```

---

## 12. Best Practices

1. **Prefer CDK over building from scratch** — CDK handles accessibility, keyboard navigation, and cross-browser quirks.
2. **Headless components for design systems** — separate behavior from styling for maximum flexibility.
3. **Compound components with InjectionToken** avoid prop drilling in complex component groups.
4. **Use `setInput()` for dynamic components** — signal-compatible, type-safe.
5. **CDK Overlay for floating elements** — it handles scroll containment, positioning, and backdrop.
6. **Always provide `trackBy`/`trackItem`** in CDK Virtual Scroll.
7. **CDK DragDrop works with signals** — just trigger signal updates after `moveItemInArray`.

---

> **Summary:** Advanced Angular component patterns leverage the CDK for robust primitives (overlay, drag-drop, virtual scroll, table, stepper), the compound component pattern for cohesive APIs, and headless components for design-system flexibility. `ViewContainerRef.createComponent()` with dynamic imports enables truly lazy-loaded, runtime-composed UIs.

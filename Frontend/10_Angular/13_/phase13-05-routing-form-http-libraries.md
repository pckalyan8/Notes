# Phase 13.5, 13.6, 13.7 — Routing, Form & HTTP Libraries
## Complete Guide: Angular Ecosystem Libraries for Navigation, Forms, and Data Fetching

---

## Phase 13.5 — Routing Libraries

### Angular Router (Primary)

The Angular Router is the built-in, officially supported routing solution and the default choice for all Angular applications. It is covered in depth in Phase 10 and Phase 11. What's worth noting in the context of the ecosystem is that the Angular Router is remarkably full-featured — it handles everything from simple URL-to-component mapping all the way to lazy loading, route guards, typed parameters, and deep integration with the NgRx Router Store. You should reach for the Angular Router first, every time. Third-party routing libraries for Angular are rare and generally unnecessary.

### @ngrx/router-store — Connecting Navigation to State

While not a standalone router, `@ngrx/router-store` is a critical companion to both the Angular Router and NgRx. It bridges the gap between URL state and application state, letting you treat navigation events as part of your Redux action stream.

The package provides pre-built selectors you can use anywhere in your app to read the current URL, route parameters, and query parameters without injecting `ActivatedRoute` everywhere:

```typescript
import { getRouterSelectors } from '@ngrx/router-store';

// These are ready-made selectors — no custom code needed
export const {
  selectCurrentRoute,         // the full ActivatedRouteSnapshot
  selectQueryParams,          // { search: 'angular', page: '2' }
  selectQueryParam,           // selectQueryParam('search') → 'angular'
  selectRouteParams,          // { id: '42' }
  selectRouteParam,           // selectRouteParam('id') → '42'
  selectRouteData,            // static data defined in route config
  selectUrl,                  // '/books/42?tab=reviews'
  selectTitle,                // page title from route config
} = getRouterSelectors();
```

This is particularly powerful in Effects, where you can combine router selectors with HTTP calls without injecting `ActivatedRoute` into your service:

```typescript
loadBookDetails$ = createEffect(() =>
  this.actions$.pipe(
    ofType(ROUTER_NAVIGATION),   // fires on every navigation
    withLatestFrom(this.store.select(selectRouteParam('id'))),
    filter(([, id]) => !!id),
    switchMap(([, id]) =>
      this.booksService.getById(id!).pipe(
        map(book => BookActions.loadDetailSuccess({ book })),
        catchError(error => of(BookActions.loadDetailFailure({ error })))
      )
    )
  )
);
```

---

## Phase 13.6 — Form Libraries

### Angular Reactive Forms (Built-In — Always Start Here)

Angular's built-in reactive forms module should be your default. As covered in Phase 11, reactive forms give you strongly typed form groups, validators, async validators, and programmatic control over every aspect of the form. Before considering a third-party form library, confirm that built-in reactive forms genuinely cannot solve your problem.

### ng-select — Advanced Select & Multi-Select Component

The native HTML `<select>` element is limited: no search/filtering, no multi-select with custom tags, no grouping, no async data loading. `ng-select` fills this gap with a fully-featured, accessible select component that feels native to Angular.

```bash
npm install @ng-select/ng-select
```

```typescript
// component.ts
import { NgSelectModule } from '@ng-select/ng-select';

@Component({
  imports: [NgSelectModule, ReactiveFormsModule],
  template: `
    <!-- Basic single select with search -->
    <ng-select
      [items]="countries"
      bindLabel="name"
      bindValue="code"
      placeholder="Select a country"
      [formControl]="countryControl"
      [searchable]="true"
      [clearable]="true">
    </ng-select>

    <!-- Multi-select with tags -->
    <ng-select
      [items]="skills"
      bindLabel="label"
      bindValue="value"
      [multiple]="true"
      [formControl]="skillsControl"
      placeholder="Select your skills">
    </ng-select>

    <!-- Async search — loads options from an API as the user types -->
    <ng-select
      [items]="users$ | async"
      bindLabel="name"
      bindValue="id"
      [typeahead]="userSearch$"
      placeholder="Search users..."
      [formControl]="assigneeControl"
      [loading]="usersLoading">

      <!-- Custom option template -->
      <ng-template ng-option-tmp let-item="item">
        <img [src]="item.avatar" style="width:20px"> {{ item.name }}
      </ng-template>
    </ng-select>
  `
})
export class AssignmentFormComponent {
  userSearch$ = new Subject<string>();
  users$: Observable<User[]>;
  usersLoading = false;

  constructor(private userService: UserService) {
    // Wire up the typeahead search to an API call
    this.users$ = this.userSearch$.pipe(
      debounceTime(300),
      distinctUntilChanged(),
      tap(() => this.usersLoading = true),
      switchMap(term => this.userService.search(term).pipe(
        finalize(() => this.usersLoading = false)
      ))
    );
  }
}
```

`ng-select` is one of the most widely used Angular libraries and is production-ready. It handles virtual scrolling for large option lists automatically, supports grouping options, and can be fully customized with templates.

### ngx-formly — Dynamic Form Generation

`ngx-formly` solves a different problem: building forms from **JSON configuration at runtime** rather than from HTML templates. This is essential when your forms are dynamically generated (from API responses, user configuration, CMS data, etc.) or when you need a large number of similar forms that differ only in structure.

```bash
npm install @ngx-formly/core @ngx-formly/material
```

```typescript
// main.ts — register the library
import { FormlyModule } from '@ngx-formly/core';
import { FormlyMaterialModule } from '@ngx-formly/material';

bootstrapApplication(AppComponent, {
  providers: [
    importProvidersFrom(
      FormlyModule.forRoot({
        validationMessages: [
          { name: 'required', message: 'This field is required' },
          { name: 'email', message: 'Enter a valid email address' },
        ]
      }),
      FormlyMaterialModule  // use Material components as form controls
    )
  ]
});
```

```typescript
@Component({
  imports: [ReactiveFormsModule, FormlyModule],
  template: `
    <form [formGroup]="form" (ngSubmit)="onSubmit()">
      <formly-form [form]="form" [fields]="fields" [model]="model" />
      <button mat-flat-button color="primary" type="submit">Submit</button>
    </form>
  `
})
export class DynamicFormComponent {
  form = new FormGroup({});
  model: Record<string, unknown> = {};

  // This entire form definition could come from an API!
  fields: FormlyFieldConfig[] = [
    {
      key: 'firstName',
      type: 'input',
      props: {
        label: 'First Name',
        required: true,
        placeholder: 'Enter your first name',
      }
    },
    {
      key: 'email',
      type: 'input',
      props: {
        label: 'Email',
        type: 'email',
        required: true,
      },
      validators: { validation: ['email'] }
    },
    {
      key: 'accountType',
      type: 'select',
      props: {
        label: 'Account Type',
        options: [
          { label: 'Personal', value: 'personal' },
          { label: 'Business', value: 'business' },
        ]
      }
    },
    {
      // Conditional field — only shown when accountType === 'business'
      key: 'companyName',
      type: 'input',
      props: { label: 'Company Name' },
      expressions: {
        'props.required': 'model.accountType === "business"',
        hide: 'model.accountType !== "business"',
      }
    }
  ];

  onSubmit() {
    if (this.form.valid) {
      console.log('Form data:', this.model);
    }
  }
}
```

The power of formly becomes clear when the `fields` array comes from an API. You can define custom field types, custom wrappers, and complex validation rules, all declaratively. It supports Angular Material, Bootstrap, PrimeNG, and more as UI layers.

### Signal-Based Forms (Angular 20+ Experimental)

Angular 20 introduced an experimental signal-based forms API that replaces `FormControl`/`FormGroup` with signal primitives. This is covered in Phase 11.4 of the roadmap. The key benefit is that form state — value, validity, dirty status — is all exposed as signals, integrating naturally with signal-based components and computed values without needing to subscribe to observables.

```typescript
import { signalForm, signalFormControl } from '@angular/forms'; // experimental

const form = signalForm({
  email: signalFormControl('', { validators: [Validators.email] }),
  name: signalFormControl('', { validators: [Validators.required] }),
});

// Access values and validity as signals — no .valueChanges observable needed
const isValid = computed(() => form.valid());
const emailValue = computed(() => form.controls.email.value());
```

This is still experimental and the API may change before it stabilizes. Use it for new exploratory features, but stick with reactive forms for production code until it graduates to stable.

---

## Phase 13.7 — HTTP & Data Libraries

### Angular HttpClient + httpResource (Built-In)

The built-in `HttpClient` is your foundation. From Angular 21, `httpResource()` provides a signal-based HTTP data-fetching primitive that handles loading, error, and data states automatically — without needing to write a whole service plus RxJS operators for each endpoint.

```typescript
import { httpResource } from '@angular/core'; // Angular 21+

@Component({ /* ... */ })
export class BookDetailComponent {
  // Route parameter as a signal (via withComponentInputBinding)
  bookId = input.required<string>();

  // httpResource automatically re-fetches when bookId changes
  bookResource = httpResource<Book>(() => `/api/books/${this.bookId()}`);

  // The resource exposes signals for all states
  book = this.bookResource.value;       // signal: Book | undefined
  loading = this.bookResource.isLoading; // signal: boolean
  error = this.bookResource.error;      // signal: unknown | undefined
}
```

```html
@if (bookResource.isLoading()) {
  <app-skeleton />
} @else if (bookResource.error()) {
  <app-error-state [error]="bookResource.error()" />
} @else if (bookResource.value()) {
  <app-book-detail [book]="bookResource.value()!" />
}
```

This is remarkably clean. For the common case of "load data when component mounts or a parameter changes, show loading and error states", `httpResource` is all you need. You no longer need to write a service method, inject it, subscribe, manage loading/error signals manually, and unsubscribe.

### TanStack Query for Angular — Server State Management

`@tanstack/angular-query-experimental` brings the TanStack Query paradigm (also known as React Query in the React ecosystem) to Angular. The fundamental idea is that **server data is not application state** — it's a cache. Data fetched from an API should be treated differently from local UI state: it needs caching, background refetching, stale-time management, and automatic invalidation.

```bash
npm install @tanstack/angular-query-experimental
```

```typescript
// main.ts
import { provideTanStackQuery, QueryClient } from '@tanstack/angular-query-experimental';

bootstrapApplication(AppComponent, {
  providers: [
    provideTanStackQuery(new QueryClient({
      defaultOptions: {
        queries: {
          staleTime: 1000 * 60 * 5,  // data is fresh for 5 minutes
          gcTime: 1000 * 60 * 10,    // cache removed after 10 minutes of inactivity
          retry: 2,                   // retry failed requests twice
        }
      }
    }))
  ]
});
```

```typescript
import { injectQuery, injectMutation, injectQueryClient } from '@tanstack/angular-query-experimental';

@Component({
  template: `
    @if (booksQuery.isPending()) {
      <app-loading-skeleton />
    } @else if (booksQuery.isError()) {
      <app-error [message]="booksQuery.error()?.message" (retry)="booksQuery.refetch()" />
    } @else {
      @for (book of booksQuery.data(); track book.id) {
        <app-book-card
          [book]="book"
          (delete)="deleteBook.mutate(book.id)"
        />
      }
    }
  `
})
export class BooksListComponent {
  private booksService = inject(BooksService);
  private queryClient = injectQueryClient();

  // Declares a query — TanStack handles caching, refetching, deduplication
  booksQuery = injectQuery(() => ({
    queryKey: ['books'],           // cache key — same key = same cache across components
    queryFn: () => this.booksService.getAll(),
  }));

  // A mutation — for data changes (POST, PUT, DELETE)
  deleteBook = injectMutation(() => ({
    mutationFn: (id: string) => this.booksService.delete(id),
    onSuccess: () => {
      // Invalidate the books cache so the list refetches
      this.queryClient.invalidateQueries({ queryKey: ['books'] });
    },
    onError: (error) => {
      console.error('Delete failed:', error);
    }
  }));
}
```

The key advantages TanStack Query provides over plain `HttpClient` are cache deduplication (if two components ask for the same data, only one request fires), background refetching (data automatically refreshes when the window regains focus or the network reconnects), and stale-time management (you control how long data is considered "fresh" before a background refresh is triggered).

**When should you use TanStack Query vs `httpResource`?** Use `httpResource` for simple, component-scoped data fetching where caching isn't needed. Use TanStack Query when the same data is needed in multiple places in your app, when you need sophisticated cache invalidation strategies, when you need optimistic updates with automatic rollback on failure, or when you need infinite pagination (loading more data as users scroll).

### Apollo Angular — GraphQL Client

If your backend uses GraphQL, Apollo Angular is the standard integration. It provides a full-featured GraphQL client with normalized caching, optimistic updates, and — in recent versions — signal support.

```bash
npm install @apollo/client apollo-angular graphql
```

```typescript
// main.ts
import { provideApollo } from 'apollo-angular';
import { InMemoryCache } from '@apollo/client/core';

bootstrapApplication(AppComponent, {
  providers: [
    provideApollo(() => ({
      cache: new InMemoryCache(),
      uri: 'https://api.example.com/graphql',
    }))
  ]
});
```

```typescript
import { Apollo, gql } from 'apollo-angular';
import { toSignal } from '@angular/core/rxjs-interop';

const GET_BOOKS = gql`
  query GetBooks($genre: String) {
    books(genre: $genre) {
      id
      title
      author {
        name
      }
      rating
    }
  }
`;

const DELETE_BOOK = gql`
  mutation DeleteBook($id: ID!) {
    deleteBook(id: $id) {
      success
    }
  }
`;

@Component({ /* ... */ })
export class BooksComponent {
  private apollo = inject(Apollo);

  // Executing a query
  booksResult = toSignal(
    this.apollo.watchQuery<{ books: Book[] }>({
      query: GET_BOOKS,
      variables: { genre: 'fiction' },
    }).valueChanges.pipe(
      map(result => result.data.books)
    )
  );

  // Executing a mutation
  deleteBook(id: string) {
    this.apollo.mutate({
      mutation: DELETE_BOOK,
      variables: { id },
      update: (cache) => {
        // Update the Apollo cache directly after mutation
        // so the UI updates without a refetch
        cache.evict({ id: `Book:${id}` });
        cache.gc();
      }
    }).subscribe();
  }
}
```

Apollo's normalized cache is its killer feature. When you fetch a book and it appears in multiple queries (a list query and a detail query), Apollo merges the data in a normalized cache by ID. Updating a book via a mutation automatically updates all queries that contain that book, without any manual cache manipulation.

### RxAngular — Template-Level Rendering Optimization

`RxAngular` is a collection of performance-focused Angular utilities centered around making reactive, RxJS-driven templates first-class citizens. Its most used parts are `@rx-angular/template` (for reactive rendering) and `@rx-angular/state` (for lightweight component-level state management).

```bash
npm install @rx-angular/template @rx-angular/state
```

The `rxFor` and `rxIf` directives replace `*ngFor` and `*ngIf` with versions that integrate directly with RxJS Observables without needing the `async` pipe and that use a "concurrent mode" rendering approach — they can defer expensive list renders across multiple frames to keep the UI responsive.

```html
<!-- Standard approach — blocks the main thread while rendering the full list -->
<div *ngFor="let item of items$ | async; trackBy: trackById">{{ item.name }}</div>

<!-- rxFor — renders progressively, keeping the UI responsive for large lists -->
<div *rxFor="let item of items$; trackBy: 'id'; renderCallback: renderDone$">
  {{ item.name }}
</div>
```

`@rx-angular/state` provides a component-level state management solution similar to NgRx ComponentStore but with a more RxJS-centric API:

```typescript
import { RxState } from '@rx-angular/state';

interface ComponentState {
  loading: boolean;
  items: Product[];
  searchTerm: string;
}

@Component({
  providers: [RxState],
  // ...
})
export class ProductListComponent {
  private state = inject<RxState<ComponentState>>(RxState);

  readonly items$ = this.state.select('items');
  readonly loading$ = this.state.select('loading');

  readonly filteredItems$ = this.state.select(
    combineLatest([
      this.state.select('items'),
      this.state.select('searchTerm')
    ]),
    ([items, term]) => items.filter(i => i.name.includes(term))
  );

  constructor() {
    // Initialize state
    this.state.set({ loading: false, items: [], searchTerm: '' });

    // Connect an observable to state — it automatically updates state when the observable emits
    this.state.connect('items', this.productService.getAll());
  }

  search(term: string) {
    this.state.set({ searchTerm: term });
  }
}
```

RxAngular is a good choice when you have **large, performant lists** driven by RxJS streams, or when you want ergonomic RxJS-native state management at the component level.

---

## Important Points & Best Practices

For routing, always use the Angular Router. Never implement client-side navigation manually or with a third-party router.

For forms, evaluate your actual complexity first. Most forms fit perfectly into Angular reactive forms with typed form groups and built-in validators. Reach for `ng-select` when you need advanced select functionality. Reach for `ngx-formly` when forms are generated from runtime configuration.

For HTTP and data fetching, follow this decision tree: if you're fetching data for a single component and don't need caching, use `httpResource`. If you're fetching data that needs to be shared across multiple components with caching and invalidation, use TanStack Query. If you're using GraphQL, use Apollo Angular. If you need real-time data, use `HttpClient` with SSE or WebSockets via RxJS.

Always configure `TanStack Query`'s `staleTime` appropriately for your data. Data that changes frequently (notifications, stock prices) should have a very short staleTime (seconds) or be driven by WebSockets instead. Data that rarely changes (user settings, reference lists) can have a long staleTime (minutes or hours), dramatically reducing API traffic.

When using Apollo, always think carefully about your cache update strategy after mutations. The two main approaches are cache invalidation (force a re-fetch by calling `refetchQueries`) and cache updates (directly modify the Apollo cache in the `update` callback). Cache updates are faster for the user but require more code. For anything that lists many items, invalidation is usually safer and simpler.

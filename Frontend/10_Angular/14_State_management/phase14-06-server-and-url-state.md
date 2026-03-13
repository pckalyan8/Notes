# Phase 14.6 & 14.7 — Server State & URL State Management
## Caching, TanStack Query, Resource API, and the Router as State Container

---

# PART 1: Phase 14.6 — Server State Management

## The Fundamental Insight: Server State Is Different

One of the most important conceptual shifts in modern frontend architecture is recognizing that **server state is not the same kind of thing as client state**. This distinction sounds philosophical, but it has extremely practical consequences for how you architect your Angular application.

Client state — things like "is this dropdown open", "which tab is active", "what items are in the shopping cart" — is data that exists *only in the browser*. Your application owns it completely. You create it, modify it, and destroy it on your schedule. It is perfectly predictable.

Server state is fundamentally different. It is data that **lives on a server**, that you are temporarily *copying* into the browser. The original always lives elsewhere. Other users may be modifying it right now. The copy in your browser may already be outdated seconds after you fetched it. You don't truly own it — you are just borrowing a snapshot.

This distinction matters because client state tools (like NgRx store or a signal service) are designed for data you own. When you put server state into NgRx, you implicitly take on responsibilities that are invisible but real: keeping the data fresh, handling stale data gracefully, coordinating multiple components that ask for the same data, invalidating cached data when mutations occur. NgRx gives you no help with any of these — you build it all yourself, and most teams build it incompletely.

The modern solution is to use tools designed *specifically* for server state: Angular's `httpResource()` for simple cases, and TanStack Query for sophisticated caching requirements.

---

## 14.6.1 — Distinguishing Client State from Server State in Practice

A useful mental exercise: for each piece of data in your application, ask "where does this ultimately live?" If the answer is "in the browser only, and nowhere else", it's client state. If the answer is "in a database, and the browser has a copy", it's server state.

Here are some examples to calibrate your thinking. The user's authentication token is server state — it's issued by the server and the server is the authority on whether it's valid. The shopping cart in a logged-in user's account is server state — it persists to a database and the user expects to see it across devices. A draft blog post that hasn't been saved yet is client state — it only exists in this browser tab. The selected color theme is client state if stored in localStorage, but server state if stored in the user's account preferences on the backend.

---

## 14.6.2 — Caching Server Responses

When a component mounts and needs data, the naive approach is to make an HTTP request every single time. This is wasteful when the data hasn't changed, frustrating for the user who sees a loading spinner on every navigation, and expensive for the server. The solution is **caching**.

Caching means storing a fetched result so that subsequent requests for the same data can be served from the cache without going to the network. But caching introduces a new problem: **stale data**. How long do you trust the cached version before re-fetching? This is the core challenge of server state management.

The answer depends on how frequently your data changes. A list of countries for a dropdown? It changes essentially never — you could cache it for hours. A user's unread notification count? It might change every few seconds — the cache duration should be very short. An order's shipping status? It changes occasionally — maybe cache for a minute or two, and always refetch when the user manually navigates to the order page.

---

## 14.6.3 — Background Refetching and Stale-While-Revalidate

**Stale-while-revalidate (SWR)** is a caching strategy where you immediately return cached (potentially stale) data to the user while simultaneously fetching a fresh version in the background. When the fresh version arrives, you update the display. The key benefit is that the user never stares at a loading spinner — they see data instantly, and if that data was stale, it updates smoothly.

This pattern is named after an HTTP cache-control directive (`stale-while-revalidate`) but is widely implemented in frontend state management. TanStack Query and Angular's `httpResource()` both implement this pattern.

```
User visits the page:
  1. Check cache → data found (from last visit, 30 seconds ago)
  2. Display cached data IMMEDIATELY → user sees content at once
  3. In background → fetch fresh data from server
  4. Fresh data arrives → update display (user may see brief UI update)

User visits again after 10 minutes:
  1. Cache might be expired → show loading state briefly
  2. Fresh data arrives → display it and store in cache for next time
```

**Background refetching** also happens on specific triggers beyond staleness. When the browser window regains focus (the user switched tabs and came back), when the network reconnects after going offline, or when the user explicitly requests a refresh — all of these are good moments to re-validate cached data in the background.

---

## 14.6.4 — Angular's Resource API and `httpResource()`

Angular 19 introduced the `resource()` function and Angular 21 stabilized `httpResource()` — a signal-based data-fetching primitive that handles loading, error, and success states automatically, with built-in reactivity.

`httpResource()` takes a function that returns a URL (or a request config), and returns a resource object with signals for the current value, loading state, and error state. Critically, if the URL function depends on other signals, `httpResource` automatically refetches when those signals change.

```typescript
import { Component, input, computed } from '@angular/core';
import { httpResource } from '@angular/core';  // Angular 21+

export interface BookDetail {
  id: string;
  title: string;
  author: string;
  description: string;
  reviews: Review[];
}

@Component({
  selector: 'app-book-detail',
  standalone: true,
  template: `
    <!-- Three states, handled cleanly with @switch -->
    @if (bookResource.isLoading()) {
      <app-skeleton-loader />
    } @else if (bookResource.error()) {
      <app-error-state
        [message]="bookResource.error()"
        (retry)="bookResource.reload()"
      />
    } @else if (bookResource.value()) {
      <article>
        <h1>{{ bookResource.value()!.title }}</h1>
        <p class="author">by {{ bookResource.value()!.author }}</p>
        <p>{{ bookResource.value()!.description }}</p>

        <section class="reviews">
          @for (review of bookResource.value()!.reviews; track review.id) {
            <app-review-card [review]="review" />
          }
        </section>
      </article>
    }
  `
})
export class BookDetailComponent {
  // This input could come from the router via withComponentInputBinding()
  bookId = input.required<string>();

  // httpResource automatically refetches when bookId() changes
  // The function is reactive — reading bookId() inside it registers a dependency
  bookResource = httpResource<BookDetail>(() => `/api/books/${this.bookId()}`);

  // You can also pass a full request config for POST requests with body, headers, etc.
  // bookResource = httpResource<BookDetail>(() => ({
  //   url: `/api/books/${this.bookId()}`,
  //   method: 'GET',
  //   headers: { 'Accept-Language': this.locale() },
  // }));
}
```

`httpResource()` is the right tool for component-scoped data fetching where you need reactivity to input signals. It handles the lifecycle concerns (cancelling in-flight requests when parameters change, resetting on re-fetch) automatically. Its limitation is that it is scoped to the component — it does not share a cache between two different components that request the same data.

---

## 14.6.5 — TanStack Query for Angular: Sophisticated Server State

When multiple components across the application need the same server data, when you need sophisticated cache invalidation after mutations, when you need infinite pagination, or when you need optimistic updates with automatic rollback — `@tanstack/angular-query-experimental` is the right tool.

TanStack Query (formerly React Query) is built around the observation that server state has specific needs: caching with configurable staleness, background updates, request deduplication (two components asking for the same data = one HTTP request), and mutation handling. It provides all of these out of the box.

```typescript
// main.ts — provide the query client globally
import { provideTanStackQuery, QueryClient } from '@tanstack/angular-query-experimental';

bootstrapApplication(AppComponent, {
  providers: [
    provideTanStackQuery(new QueryClient({
      defaultOptions: {
        queries: {
          // Data is considered fresh for 5 minutes — no background refetch during this time
          staleTime: 1000 * 60 * 5,

          // Data is garbage-collected from cache 10 minutes after all observers unmount
          gcTime: 1000 * 60 * 10,

          // Automatically retry failed requests twice before showing an error
          retry: 2,

          // Refetch when the browser window regains focus (user switches tabs and comes back)
          refetchOnWindowFocus: true,
        }
      }
    }))
  ]
});
```

```typescript
// A service that defines the query functions — keeps HTTP logic centralized
@Injectable({ providedIn: 'root' })
export class BooksQueryService {
  private http = inject(HttpClient);

  // These functions define HOW to fetch — TanStack Query manages WHEN to fetch
  getAll = () => firstValueFrom(this.http.get<Book[]>('/api/books'));

  getById = (id: string) =>
    firstValueFrom(this.http.get<Book>(`/api/books/${id}`));

  create = (dto: CreateBookDto) =>
    firstValueFrom(this.http.post<Book>('/api/books', dto));

  delete = (id: string) =>
    firstValueFrom(this.http.delete<void>(`/api/books/${id}`));

  update = (id: string, changes: Partial<Book>) =>
    firstValueFrom(this.http.patch<Book>(`/api/books/${id}`, changes));
}
```

```typescript
import {
  injectQuery,
  injectMutation,
  injectQueryClient,
  keepPreviousData,
} from '@tanstack/angular-query-experimental';

@Component({
  standalone: true,
  template: `
    <!-- Pagination controls -->
    <div class="pagination-controls">
      <button (click)="page.set(page() - 1)" [disabled]="page() <= 1">Previous</button>
      <span>Page {{ page() }}</span>
      <button (click)="page.set(page() + 1)">Next</button>
    </div>

    <!-- Query state handling -->
    @if (booksQuery.isPending()) {
      <app-skeleton-list [count]="10" />
    } @else if (booksQuery.isError()) {
      <app-error-state
        [message]="booksQuery.error()?.message"
        (retry)="booksQuery.refetch()"
      />
    } @else {
      <div [class.stale]="booksQuery.isStale()">
        <!-- isStale() is true when data is older than staleTime — useful for visual indicators -->
        @for (book of booksQuery.data()?.books; track book.id) {
          <app-book-card
            [book]="book"
            [isDeleting]="deleteMutation.isPending() && deleteMutation.variables() === book.id"
            (delete)="onDelete(book.id)"
          />
        }
      </div>
    }
  `
})
export class BooksListComponent {
  private queryService = inject(BooksQueryService);
  private queryClient = injectQueryClient();

  // A signal controlling which page we're on
  page = signal(1);

  // injectQuery creates a reactive query — it automatically re-runs when `page()` changes
  // because the queryKey and queryFn both read the page signal
  booksQuery = injectQuery(() => ({
    // queryKey is the cache key — same key = same cache entry
    // Include all parameters that affect the query in the key
    queryKey: ['books', { page: this.page() }] as const,

    queryFn: () => this.queryService.getAll(this.page()),

    // keepPreviousData: while fetching page 2, show page 1 data instead of a loading state
    // This prevents the jarring flash of empty content during pagination
    placeholderData: keepPreviousData,
  }));

  // injectMutation creates a mutation function for data changes
  deleteMutation = injectMutation(() => ({
    mutationFn: (id: string) => this.queryService.delete(id),

    onSuccess: (_, deletedId) => {
      // After a successful delete, invalidate the cache for all book queries
      // This triggers a background refetch so the list reflects the deletion
      this.queryClient.invalidateQueries({ queryKey: ['books'] });

      // Or: if you know the new state, update the cache directly without refetching
      this.queryClient.setQueryData(
        ['books', { page: this.page() }],
        (old: { books: Book[] } | undefined) =>
          old ? { ...old, books: old.books.filter(b => b.id !== deletedId) } : old
      );
    },

    onError: (error) => {
      // Error handling — can display a toast notification here
      console.error('Delete failed:', error);
    }
  }));

  onDelete(id: string) {
    this.deleteMutation.mutate(id);
  }
}
```

### Optimistic Updates with TanStack Query

TanStack Query provides built-in support for optimistic updates — updating the UI immediately before the server responds, then rolling back if the operation fails:

```typescript
updateBookMutation = injectMutation(() => ({
  mutationFn: ({ id, changes }: { id: string; changes: Partial<Book> }) =>
    this.queryService.update(id, changes),

  // onMutate runs BEFORE the mutation function — this is where we apply the optimistic update
  onMutate: async ({ id, changes }) => {
    // Cancel any outgoing refetches to avoid overwriting our optimistic update
    await this.queryClient.cancelQueries({ queryKey: ['books'] });

    // Snapshot the previous value in case we need to roll back
    const previousBooks = this.queryClient.getQueryData(['books', { page: this.page() }]);

    // Optimistically update the cache immediately
    this.queryClient.setQueryData(
      ['books', { page: this.page() }],
      (old: { books: Book[] } | undefined) =>
        old
          ? { ...old, books: old.books.map(b => b.id === id ? { ...b, ...changes } : b) }
          : old
    );

    // Return context with the snapshot — available in onError for rollback
    return { previousBooks };
  },

  // onError runs if the mutation fails — use the context to roll back
  onError: (err, variables, context) => {
    if (context?.previousBooks) {
      this.queryClient.setQueryData(['books', { page: this.page() }], context.previousBooks);
    }
  },

  // onSettled runs after success or failure — refetch to ensure consistency
  onSettled: () => {
    this.queryClient.invalidateQueries({ queryKey: ['books'] });
  },
}));
```

The key insight is the three-callback pattern: `onMutate` applies the optimistic update and saves a snapshot for rollback, `onError` restores the snapshot if something went wrong, and `onSettled` ensures the cache stays consistent by triggering a background refetch regardless of outcome.

---

---

# PART 2: Phase 14.7 — URL State Management

## The URL as the Most Underused State Container

The URL is probably the most powerful, most durable, and most underused state container in any web application. Unlike signals, NgRx, or services — which all reset to initial values when the page refreshes or when the user shares a link — the URL persists across refreshes and is inherently shareable. If a user filters a product list and shares the URL, the recipient should see the same filtered list. That is only possible if the filter state is encoded in the URL.

Every piece of state that represents a *navigational position* — something the user might want to bookmark, share, or return to — belongs in the URL. Concretely, this includes the current page number in paginated lists, active search queries, filter selections (category, price range, rating), sort order and direction, active tab or section, and selected item IDs in master-detail views.

---

## 14.7.1 — Router as Source of Truth for Navigation State

The Angular Router exposes the current URL state through the `ActivatedRoute` service and its observable properties. The key observables are `paramMap` (for route parameters like `/books/:id`), `queryParamMap` (for query parameters like `?search=angular&page=2`), and `fragment` (for the hash fragment like `#section-3`).

The critical mental shift is treating the router as the canonical source of truth for navigation-related state, rather than keeping a separate copy in a service or signal. When the URL changes (including when the user clicks Back in the browser), your application should derive its state from the URL, not the other way around.

```typescript
@Component({
  selector: 'app-products-list',
  standalone: true,
  template: `
    <app-search-bar [value]="searchQuery()" (search)="onSearch($event)" />
    <app-category-filter [selected]="category()" (change)="onCategoryChange($event)" />
    <app-sort-control [sortBy]="sortBy()" [direction]="sortDir()" (change)="onSortChange($event)" />

    @for (product of products(); track product.id) {
      <app-product-card [product]="product" />
    }

    <app-pagination
      [currentPage]="currentPage()"
      [totalPages]="totalPages()"
      (pageChange)="onPageChange($event)"
    />
  `
})
export class ProductsListComponent {
  private route = inject(ActivatedRoute);
  private router = inject(Router);
  private productsService = inject(ProductsService);

  // Read URL state as signals — these update automatically when the URL changes
  // toSignal() converts the observable to a signal, with an initial value from the current route
  private queryParams = toSignal(this.route.queryParamMap, {
    initialValue: this.route.snapshot.queryParamMap
  });

  // Derive specific values from the query params signal
  // These update automatically whenever queryParams() changes
  searchQuery = computed(() => this.queryParams().get('search') ?? '');
  category = computed(() => this.queryParams().get('category') ?? 'all');
  sortBy = computed(() => this.queryParams().get('sortBy') ?? 'name');
  sortDir = computed(() => this.queryParams().get('sortDir') ?? 'asc');
  currentPage = computed(() => Number(this.queryParams().get('page') ?? '1'));

  // Data loading — reacts to the computed URL params
  // When any URL param changes, these automatically update
  productsResource = httpResource<{ products: Product[]; total: number }>(() => ({
    url: '/api/products',
    params: {
      search: this.searchQuery(),
      category: this.category(),
      sortBy: this.sortBy(),
      sortDir: this.sortDir(),
      page: String(this.currentPage()),
      limit: '20',
    }
  }));

  products = computed(() => this.productsResource.value()?.products ?? []);
  totalPages = computed(() =>
    Math.ceil((this.productsResource.value()?.total ?? 0) / 20)
  );

  // When the user changes a filter, update the URL — the URL is the state
  // The URL change triggers queryParams() to update, which triggers data re-fetch
  private updateQueryParams(params: Record<string, string | null>) {
    this.router.navigate([], {
      relativeTo: this.route,
      queryParams: params,
      queryParamsHandling: 'merge', // preserve other params not being changed
      // replaceUrl: true prevents every filter change from creating a browser history entry
      // (optional — use it when you want filter changes to not create Back button entries)
    });
  }

  onSearch(search: string) {
    // Reset to page 1 when search changes — don't preserve old page number
    this.updateQueryParams({ search: search || null, page: '1' });
  }

  onCategoryChange(category: string) {
    this.updateQueryParams({ category: category === 'all' ? null : category, page: '1' });
  }

  onSortChange({ sortBy, direction }: { sortBy: string; direction: string }) {
    this.updateQueryParams({ sortBy, sortDir: direction });
  }

  onPageChange(page: number) {
    this.updateQueryParams({ page: String(page) });
  }
}
```

Notice the data flow: user interactions → update URL → URL triggers queryParams to update → computed signals update → `httpResource` refetches. Every single step is reactive and automatic. The user clicking the browser's Back button works correctly for free, because going back changes the URL, which changes the queryParams signal, which rerenders the filter state and re-fetches the appropriate data.

---

## 14.7.2 — Storing Filters and Pagination in Query Params

Here are concrete patterns and guidelines for what to put in query params and how to handle edge cases:

```typescript
// A reusable utility for working with URL filter state
@Injectable({ providedIn: 'root' })
export class UrlStateService {
  private router = inject(Router);

  // Navigate with updated query params, resetting pagination when filters change
  updateFilters(
    route: ActivatedRoute,
    params: Record<string, string | number | boolean | null>
  ) {
    const normalized: Record<string, string | null> = {};

    for (const [key, value] of Object.entries(params)) {
      if (value === null || value === undefined || value === '' || value === false) {
        // Remove empty/null/false values from URL to keep it clean
        normalized[key] = null;
      } else if (typeof value === 'boolean') {
        normalized[key] = value ? 'true' : null;
      } else {
        normalized[key] = String(value);
      }
    }

    this.router.navigate([], {
      relativeTo: route,
      queryParams: normalized,
      queryParamsHandling: 'merge',
    });
  }

  // Parse a numeric query param with a fallback default
  static getNumber(params: ParamMap, key: string, defaultValue: number): number {
    const raw = params.get(key);
    const parsed = raw !== null ? parseInt(raw, 10) : NaN;
    return isNaN(parsed) || parsed < 1 ? defaultValue : parsed;
  }

  // Parse a boolean query param ("true" string → true boolean)
  static getBoolean(params: ParamMap, key: string): boolean {
    return params.get(key) === 'true';
  }

  // Parse an enum query param with validation
  static getEnum<T extends string>(
    params: ParamMap,
    key: string,
    validValues: readonly T[],
    defaultValue: T
  ): T {
    const raw = params.get(key) as T;
    return validValues.includes(raw) ? raw : defaultValue;
  }
}
```

```typescript
// Using the utility in a component
@Component({ /* ... */ })
export class AdvancedSearchComponent {
  private route = inject(ActivatedRoute);
  private urlState = inject(UrlStateService);

  private queryParams = toSignal(this.route.queryParamMap, {
    initialValue: this.route.snapshot.queryParamMap
  });

  // Each param is parsed with proper type coercion and validation
  searchQuery = computed(() => this.queryParams().get('q') ?? '');
  currentPage = computed(() => UrlStateService.getNumber(this.queryParams(), 'page', 1));
  pageSize = computed(() => UrlStateService.getNumber(this.queryParams(), 'limit', 20));
  showOutOfStock = computed(() => UrlStateService.getBoolean(this.queryParams(), 'outOfStock'));
  sortBy = computed(() =>
    UrlStateService.getEnum(this.queryParams(), 'sort', ['name', 'price', 'rating'], 'name')
  );

  onPageChange(page: number) {
    this.urlState.updateFilters(this.route, { page });
  }

  onFilterChange(filters: Partial<SearchFilters>) {
    // Reset to page 1 when filters change
    this.urlState.updateFilters(this.route, { ...filters, page: 1 });
  }
}
```

---

## 14.7.3 — NgRx Router Store

When using NgRx, the `@ngrx/router-store` package synchronizes Angular Router state into the NgRx store. This enables using NgRx selectors to read the current URL, route params, and query params — and enables effects to react to navigation events.

```typescript
import { getRouterSelectors } from '@ngrx/router-store';

// These pre-built selectors are available once @ngrx/router-store is set up
export const {
  selectCurrentRoute,   // the full ActivatedRouteSnapshot
  selectQueryParams,    // the entire queryParams object: { search: 'angular', page: '2' }
  selectQueryParam,     // factory: selectQueryParam('search') → 'angular'
  selectRouteParams,    // the route params: { id: '42' }
  selectRouteParam,     // factory: selectRouteParam('id') → '42'
  selectUrl,            // the full URL string: '/products/42?tab=reviews'
} = getRouterSelectors();
```

```typescript
// In an effect — react to navigation and load data based on route params
@Injectable()
export class ProductDetailEffects {
  private actions$ = inject(Actions);
  private store = inject(Store);
  private productsService = inject(ProductsService);

  // This effect fires whenever the router navigates
  loadProductOnNavigation$ = createEffect(() =>
    this.actions$.pipe(
      ofType(ROUTER_NAVIGATION),

      // Get the current product ID from the router state in the store
      withLatestFrom(this.store.select(selectRouteParam('id'))),

      // Only proceed if there is an actual product ID in the URL
      filter(([, id]) => !!id),

      // Load the product with the current ID from the URL
      switchMap(([, id]) =>
        this.productsService.getById(id!).pipe(
          map(product => ProductsApiActions.loadProductDetailSuccess({ product })),
          catchError(error =>
            of(ProductsApiActions.loadProductDetailFailure({ error: error.message }))
          )
        )
      )
    )
  );
}
```

---

## Important Points and Best Practices

The most common mistake with URL state is storing state in *both* the URL and a signal service, then getting confused when they diverge. Choose one source of truth. If the state belongs in the URL (because it should survive page refresh and be shareable), read it from the URL and derive everything from there. Do not maintain a parallel copy in a signal or NgRx store.

Be conservative about what you put in query params. Not every piece of state belongs in the URL. User authentication tokens, form drafts, temporary UI selections — these do not belong in the URL because they are not shareable or meaningful in isolation. The URL should represent a meaningful, restorable view of the application — like a saved search — not a dump of every internal state value.

Use `queryParamsHandling: 'merge'` when updating filter params so that changing one filter does not wipe out others. The default behavior replaces all query params with the new set, so if you had `?search=angular&page=3` and you only update the `page` param, you'd end up with just `?page=1` if you don't merge.

For TanStack Query, always include all query parameters that affect the result in the `queryKey` array. The query key is the cache key — if you change the page number but keep the same query key, TanStack Query thinks it's the same query and returns cached data from page 1. The rule is: everything that goes into the query function should also go into the query key.

When using `httpResource()` for paginated data, be aware that it cancels the previous request and starts a new one each time the reactive function returns a different URL. This is usually the desired behavior (we don't want the old page's data arriving after the new page has loaded), but it means there's always a brief loading state during page transitions. Using TanStack Query with `placeholderData: keepPreviousData` prevents this flash by keeping the old page data visible while the new page loads.

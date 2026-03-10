# ⚡ Phase 4 — Part 4: Async JavaScript · Error Handling · Modules
> Sections 4.10 · 4.11 · 4.12 | Angular Frontend Mastery Roadmap

---

## 4.10 Asynchronous JavaScript

Asynchronous programming is the capability to start an operation and continue running other code while waiting for the result, rather than blocking. It is not optional knowledge for a frontend developer — virtually everything interesting on the web is async: network requests, timers, user input, file reading, animations.

Understanding async JavaScript deeply also makes Angular's RxJS (Phase 15) and the entire reactive model click into place.

### The Call Stack, Event Loop, and Task Queues

To understand async programming, you must first understand how JavaScript manages execution. JavaScript is single-threaded — it can only execute one piece of code at a time. The *call stack* tracks which functions are currently executing. The *event loop* is the mechanism that allows asynchronous operations to run without blocking the main thread.

Think of it like a restaurant. The chef (the call stack) can only cook one dish at a time. Orders (async operations) are placed with the kitchen staff (Web APIs). When a dish is ready, it goes to a waiting area (the task queue). The head waiter (the event loop) watches the chef — the moment the chef is free (the call stack is empty), the head waiter delivers the next waiting dish.

```
JavaScript Engine:
┌─────────────────┐         ┌──────────────────────┐
│   Call Stack    │         │   Web APIs           │
│                 │         │  (setTimeout, fetch, │
│  main()         │ ──────> │   DOM events, etc.)  │
│  greet()        │         │                      │
└─────────────────┘         └──────────┬───────────┘
                                        │
                              ┌─────────▼──────────┐
                              │  Microtask Queue   │ ← Promise callbacks
                              │  (high priority)   │
                              └─────────┬──────────┘
                              ┌─────────▼──────────┐
                              │  Task Queue        │ ← setTimeout, setInterval
                              │  (macrotasks)      │
                              └─────────┬──────────┘
                                        │
                              ┌─────────▼──────────┐
                              │   Event Loop       │
                              │ (moves tasks to    │
                              │  call stack when   │
                              │  stack is empty)   │
                              └────────────────────┘
```

The critical rule is: **microtasks always run before macrotasks.** After every macrotask, the event loop drains the *entire* microtask queue before picking up the next macrotask. This is why Promise callbacks (microtasks) always run before `setTimeout` callbacks (macrotasks), even if both have a delay of 0.

```js
console.log("1 — synchronous start");

setTimeout(() => console.log("2 — macrotask (setTimeout 0)"), 0);

Promise.resolve().then(() => console.log("3 — microtask"));

console.log("4 — synchronous end");

// Output order:
// 1 — synchronous start
// 4 — synchronous end
// 3 — microtask      ← microtask runs before macrotask!
// 2 — macrotask
```

### `setTimeout` and `setInterval`

`setTimeout` schedules a callback to run *at least* after a given delay. The actual execution time may be longer if the call stack is busy. `setInterval` schedules a callback to repeat at a given interval.

```js
// setTimeout — run once after delay
const timeoutId = setTimeout(() => {
  console.log("Runs after ~500ms");
}, 500);

// Cancel before it fires
clearTimeout(timeoutId);

// setInterval — repeat every interval
let count = 0;
const intervalId = setInterval(() => {
  count++;
  console.log(`Tick ${count}`);
  if (count === 5) clearInterval(intervalId); // stop after 5 ticks
}, 1000);
```

> **Important for Angular:** If you use `setTimeout` or `setInterval` inside an Angular component and do NOT use Angular's zone-aware versions, change detection may not trigger. Either use `NgZone.run()`, Angular's scheduler, or better yet — replace these patterns with RxJS `timer()` and `interval()` which work seamlessly with Angular.

### Callbacks and Callback Hell

The first asynchronous pattern JavaScript ever had was callbacks — passing a function as an argument to be called when an async operation completes. While simple, nesting multiple async callbacks creates deeply indented, hard-to-read code known as "callback hell" or the "pyramid of doom."

```js
// Callback hell — real-world example
getUser(userId, function(err, user) {
  if (err) return handleError(err);

  getProfile(user.profileId, function(err, profile) {
    if (err) return handleError(err);

    getPermissions(user.role, function(err, permissions) {
      if (err) return handleError(err);

      renderDashboard(user, profile, permissions, function(err) {
        if (err) return handleError(err);
        console.log("Done!");
      });
    });
  });
});
// Hard to read, hard to maintain, error handling duplicated everywhere
```

Promises and `async/await` were created specifically to solve this problem.

### Promises

A Promise is an object that represents the eventual result of an asynchronous operation. It is a placeholder for a value that will be available in the future. A Promise exists in one of three states: *pending* (still waiting), *fulfilled* (the operation succeeded), or *rejected* (the operation failed). Once settled, a Promise's state never changes.

```js
// Creating a Promise
const fetchUser = (id) => new Promise((resolve, reject) => {
  // resolve() moves the promise to "fulfilled"
  // reject() moves it to "rejected"
  fetch(`/api/users/${id}`)
    .then(response => {
      if (!response.ok) reject(new Error(`HTTP ${response.status}`));
      return response.json();
    })
    .then(data => resolve(data))
    .catch(err => reject(err));
});
```

### Promise Chaining with `.then()`, `.catch()`, `.finally()`

`.then()` receives the fulfilled value and can return a new value or a new Promise, creating a chain. `.catch()` handles rejections. `.finally()` runs regardless of success or failure — great for cleanup like hiding a loading spinner.

```js
fetchUser(42)
  .then(user => {
    console.log(user.name);
    return fetchProfile(user.profileId); // return a new Promise!
  })
  .then(profile => {
    console.log(profile.bio);
    return fetchPermissions(profile.role);
  })
  .then(permissions => {
    renderDashboard(permissions);
  })
  .catch(error => {
    // Any rejection anywhere in the chain lands here
    console.error("Something went wrong:", error.message);
  })
  .finally(() => {
    hideLoadingSpinner(); // runs regardless of success or failure
  });
```

### Promise Combinators

When you have multiple independent async operations, you often want to run them in parallel rather than sequentially. The Promise combinators enable this.

`Promise.all()` waits for ALL promises to fulfill. If *any* rejects, the whole thing rejects immediately (fail-fast). Returns an array of results in the same order as the input, regardless of which one finished first.

`Promise.allSettled()` waits for ALL promises to settle (either fulfill or reject). Never rejects itself. Returns an array of result objects with a `status` property (`"fulfilled"` or `"rejected"`). Use this when you want to handle partial failures gracefully.

`Promise.race()` settles as soon as the *first* promise settles, with that promise's result or rejection.

`Promise.any()` fulfills as soon as the *first* promise fulfills. Only rejects if *all* promises reject. Use this when you want the fastest successful result.

```js
// Load user, products, and settings in parallel
const [user, products, settings] = await Promise.all([
  fetchUser(id),
  fetchProducts(),
  fetchSettings()
]);
// All three run simultaneously — much faster than sequential!

// Handle partial failures — load what you can
const results = await Promise.allSettled([
  fetchUser(id),
  fetchProducts(),
  fetchSettings()
]);
results.forEach(result => {
  if (result.status === "fulfilled") {
    console.log("Success:", result.value);
  } else {
    console.error("Failed:", result.reason);
  }
});

// Implement a timeout pattern with race
const withTimeout = (promise, ms) => Promise.race([
  promise,
  new Promise((_, reject) =>
    setTimeout(() => reject(new Error(`Timed out after ${ms}ms`)), ms)
  )
]);

await withTimeout(fetchUser(id), 5000); // fails if takes more than 5 seconds
```

### `async` / `await`

`async/await` is syntactic sugar over Promises. An `async` function always returns a Promise. Inside it, `await` pauses execution until the awaited Promise settles, then unwraps the value. The resulting code reads like synchronous code, which is a massive readability improvement.

```js
// The Promise chain from before, rewritten with async/await
async function loadDashboard(userId) {
  try {
    const user = await fetchUser(userId);        // pauses here
    const profile = await fetchProfile(user.profileId); // then here
    const permissions = await fetchPermissions(profile.role); // then here
    renderDashboard(user, profile, permissions);
    return "success";
  } catch (error) {
    console.error("Failed:", error.message);
    throw error; // re-throw so callers know it failed
  } finally {
    hideLoadingSpinner();
  }
}

// Calling an async function — it returns a Promise
loadDashboard(42)
  .then(status => console.log(status))
  .catch(err => console.error(err));

// Or use await at the call site (if you're in an async context)
const status = await loadDashboard(42);
```

A common mistake is running `await` calls sequentially when they could be parallel. Watch for this pattern:

```js
// Sequential — slow! Each waits for the previous to finish
const user = await fetchUser(id);       // 100ms
const products = await fetchProducts(); // 200ms (starts after user)
// Total: ~300ms

// Parallel — fast! Both start at the same time
const [user, products] = await Promise.all([
  fetchUser(id),       // 100ms
  fetchProducts()      // 200ms (starts immediately)
]);
// Total: ~200ms (the longer one wins)
```

### AbortController and AbortSignal

`AbortController` allows you to cancel in-flight fetch requests or any async operation that supports the `AbortSignal` API. This is essential in Angular for canceling HTTP requests when a component is destroyed or when a search input changes before the previous result arrives.

```js
// Basic usage
const controller = new AbortController();
const signal = controller.signal;

fetch("/api/data", { signal })
  .then(res => res.json())
  .then(data => console.log(data))
  .catch(err => {
    if (err.name === "AbortError") {
      console.log("Request was cancelled");
    } else {
      throw err; // re-throw non-abort errors
    }
  });

// Cancel the request
controller.abort();

// Real-world pattern — cancel previous request on new input
class SearchService {
  #currentController = null;

  async search(query) {
    // Cancel the previous request if there is one
    this.#currentController?.abort();
    this.#currentController = new AbortController();

    const results = await fetch(
      `/api/search?q=${query}`,
      { signal: this.#currentController.signal }
    ).then(r => r.json());

    this.#currentController = null;
    return results;
  }
}
```

### Async Generators

An `async` function can also be a generator, enabling you to `yield` values asynchronously — perfect for streaming data.

```js
async function* streamPages(baseUrl) {
  let page = 1;
  while (true) {
    const data = await fetch(`${baseUrl}?page=${page}`).then(r => r.json());
    if (data.items.length === 0) break;
    yield data.items;
    page++;
  }
}

// Consume with for-await-of
for await (const pageItems of streamPages("/api/products")) {
  renderItems(pageItems);
}
```

---

## 4.11 Error Handling

Errors are inevitable. The question is not whether they will occur, but whether your application handles them gracefully or crashes with an unhelpful message.

### Error Types

JavaScript has a hierarchy of built-in error types, each intended for different situations.

`Error` is the base class. `TypeError` means a value is not of the expected type (calling something that is not a function, accessing a property of `null`, etc.). `RangeError` means a value is outside an acceptable range (e.g., creating an array with a negative length). `SyntaxError` is typically only thrown by the engine at parse time. `ReferenceError` means you referenced a variable that does not exist in scope. `URIError` is thrown by malformed URI encoding functions.

```js
typeof null.property; // TypeError: Cannot read properties of null
new Array(-1);        // RangeError: Invalid array length
undeclaredVar;        // ReferenceError: undeclaredVar is not defined
```

### `throw` Statement

You can throw any value, but best practice is to throw an Error object (or a subclass), because Error objects capture a stack trace that is invaluable for debugging.

```js
// Throwing primitives — poor practice, no stack trace
throw "Something went wrong";
throw 404;

// Throwing Error objects — best practice
throw new Error("Something went wrong");
throw new TypeError("Expected a string, got: " + typeof value);
```

### Custom Error Classes

In real applications, you will want to define your own error types to carry additional context and to allow `catch` blocks to differentiate between kinds of errors.

```js
class ApiError extends Error {
  constructor(message, statusCode, responseBody) {
    super(message);
    this.name = "ApiError";    // Override the generic "Error" name
    this.statusCode = statusCode;
    this.responseBody = responseBody;
  }
}

class ValidationError extends Error {
  constructor(field, message) {
    super(`Validation failed for '${field}': ${message}`);
    this.name = "ValidationError";
    this.field = field;
  }
}

// Now you can handle them differently
try {
  await submitForm(data);
} catch (error) {
  if (error instanceof ValidationError) {
    highlightField(error.field, error.message);
  } else if (error instanceof ApiError && error.statusCode === 401) {
    redirectToLogin();
  } else {
    showGenericErrorMessage();
    logErrorToMonitoring(error);
  }
}
```

### `try` / `catch` / `finally`

`try` contains the code that might throw. `catch` handles the error if one is thrown. `finally` runs unconditionally — whether an error occurred or not, whether the `try` block returned early or not.

```js
async function saveUser(userData) {
  let connection;
  try {
    connection = await openDatabaseConnection();
    const user = await connection.insert(userData);
    return user;
  } catch (error) {
    if (error instanceof ValidationError) {
      return { success: false, errors: error.errors };
    }
    // Unexpected error — re-throw for the caller to handle
    console.error("Unexpected error saving user:", error);
    throw error;
  } finally {
    // ALWAYS close the connection, even if we re-threw the error
    await connection?.close();
  }
}
```

### Global Error Handlers

Angular provides a clean way to handle errors globally via `ErrorHandler`, but it is useful to know the underlying browser APIs too.

```js
// Catches synchronous errors and errors in event handlers
window.onerror = function(message, source, lineno, colno, error) {
  logToMonitoring({ message, source, lineno, error });
  return true; // prevents default browser error handling
};

// Catches unhandled Promise rejections
window.addEventListener("unhandledrejection", event => {
  logToMonitoring(event.reason);
  event.preventDefault(); // prevents console error logging
});
```

---

## 4.12 JavaScript Modules

Modules are the mechanism for splitting code into separate files, each with its own scope. Before modules, every script shared the global scope — which caused naming collisions, order dependencies, and was a maintenance nightmare. Modules solve this by making each file's contents private by default and requiring explicit exports and imports.

### CommonJS — `require` / `module.exports`

CommonJS is Node.js's original module system. You still encounter it in Node.js tooling, config files, and older packages. It loads modules *synchronously* and *dynamically* — you can call `require()` conditionally or in the middle of a function.

```js
// utils.js — exporting with CommonJS
const PI = 3.14159;

function add(a, b) { return a + b; }
function subtract(a, b) { return a - b; }

module.exports = { add, subtract, PI };
// Or export a single value:
// module.exports = function createApp() { ... };

// main.js — importing with CommonJS
const { add, PI } = require("./utils");
const fs = require("fs"); // built-in Node module

add(2, 3); // 5
```

### ES Modules — `import` / `export`

ES Modules (ESM) are the standardized JavaScript module system, built into the language specification and supported natively in browsers and modern Node.js. They are *static* — imports are resolved at parse time, not at runtime. This enables tree-shaking (unused exports are eliminated at build time). Angular uses ES Modules exclusively.

```js
// math.ts / math.js — named exports
export const PI = 3.14159;
export function add(a, b) { return a + b; }
export function subtract(a, b) { return a - b; }

// Or export at the bottom (preferred for readability — shows the public API)
const PI = 3.14159;
function add(a, b) { return a + b; }
function subtract(a, b) { return a - b; }
export { PI, add, subtract };

// main.ts — named imports
import { add, subtract } from "./math";
import { add as plus } from "./math"; // rename an import
import * as MathUtils from "./math";  // import all into a namespace object
MathUtils.add(1, 2);                 // 3
```

**Default exports** — Each module can have one default export. Importing it does not require curly braces and can be named anything.

```js
// UserService.ts — default export
export default class UserService {
  getUser(id) { ... }
}

// Anywhere — import with any name
import UserService from "./UserService";
import MyUserService from "./UserService"; // Same module, different name
```

> **Best practice in Angular:** Prefer named exports over default exports. Named exports are self-documenting (the import must match the export name), work better with tooling autocomplete, and make refactoring easier. Angular CLI generates named exports for all components, services, and directives.

### Re-exporting — Building a Public API Barrel

A common pattern (especially in Angular with Nx workspaces) is to have an `index.ts` file that re-exports everything from a feature's public API. This creates a *barrel* that other modules import from, keeping internals private.

```js
// features/user/index.ts — the barrel file
export { UserComponent } from "./user.component";
export { UserService } from "./user.service";
export type { User, UserRole } from "./user.model";
// UserHelper is internal — NOT exported here

// Elsewhere in the app — clean, single import source
import { UserComponent, UserService, User } from "@app/features/user";
```

### Dynamic Imports — Code Splitting

Static imports are resolved at bundle time and included in the initial bundle. Dynamic imports use a Promise-based `import()` function call that loads a module *on demand* at runtime. This is the foundation of route-based lazy loading in Angular.

```js
// Load a heavy charting library only when needed
async function showChart() {
  const { ChartLibrary } = await import("./chart-library");
  // The library code only downloads when this function runs
  new ChartLibrary().render(document.getElementById("chart"), data);
}

// Angular uses dynamic imports for lazy-loaded routes
const routes = [
  {
    path: "dashboard",
    loadComponent: () =>
      import("./dashboard/dashboard.component").then(m => m.DashboardComponent)
  }
];
```

### `import.meta`

`import.meta` is an object that provides metadata about the current module. In Vite-based Angular projects, `import.meta.env` is the standard way to access environment variables.

```js
console.log(import.meta.url);   // the URL of the current module file

// In Vite-based Angular (replacing environments.ts in the future)
const apiUrl = import.meta.env["NG_APP_API_URL"];
const isDev = import.meta.env["DEV"]; // boolean
```

### Module Resolution Algorithm

When you write `import { x } from "something"`, JavaScript (and Node.js/bundlers) need to find the actual file. The resolution algorithm for Node.js-style resolution (used by Angular's build tools) works as follows. If the path starts with `.` or `/`, it is treated as a relative or absolute file path. If it has no path prefix, it is treated as a package name and looked up in `node_modules`. The bundler/TypeScript will try appending `.ts`, `.js`, `/index.ts`, etc. in that order.

In Angular projects with TypeScript path aliases (`paths` in `tsconfig.json`), the resolution is extended to support paths like `@app/shared`.

```json
// tsconfig.json
{
  "compilerOptions": {
    "paths": {
      "@app/*": ["src/app/*"],
      "@env/*": ["src/environments/*"]
    }
  }
}
```

```ts
// Now you can write clean imports instead of ../../../../shared
import { SharedModule } from "@app/shared";
import { environment } from "@env/environment";
```

### Circular Dependencies

A circular dependency occurs when module A imports from module B, which imports from module A. While JavaScript can technically handle this, it can lead to partially initialized modules and subtle bugs. Circular dependencies are usually a sign that your code needs to be restructured.

```
// Bad — circular dependency
// a.ts
import { b } from "./b";   // a depends on b
export const a = b + 1;

// b.ts
import { a } from "./a";   // b depends on a
export const b = a + 1;
// One of these will see `undefined` when first loaded!
```

The fix is usually to extract the shared logic into a third module that neither A nor B imports.

---

## 🔑 Key Takeaways — Part 4

The event loop is the most important mental model for JavaScript async programming. Memorise the rule: microtasks (Promises) always drain before macrotasks (setTimeout). Master `async/await` syntax — it is the dominant pattern in modern Angular HTTP services, guards, and resolvers. Always use `Promise.all()` for parallel operations; sequential awaits when the operations are truly dependent on each other. Build custom Error subclasses in your Angular applications — they enable precise error handling in interceptors and global error handlers. Use ES Module named exports everywhere in Angular; dynamic `import()` is the engine that powers lazy loading, which is a critical performance technique. Understanding module resolution and path aliases (`@app/*`) is practical knowledge you will use in every Angular project.

---

*Part 4 of 6 — Phase 4 JavaScript Mastery | Next: [DOM APIs, Events & Web APIs](./part5-dom-events-webapis.md)*

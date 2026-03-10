# ⚡ Phase 4 — Part 6: Design Patterns · Functional Programming · Performance · Modern JS
> Sections 4.16 · 4.17 · 4.18 · 4.19 | Angular Frontend Mastery Roadmap

---

## 4.16 JavaScript Design Patterns

Design patterns are reusable solutions to commonly occurring problems in software design. They are not algorithms or code templates — they are *conceptual blueprints* that describe how to structure code to solve a particular class of problem. Learning patterns gives you a shared vocabulary with other developers and sharpens your ability to recognize structure in code you read.

Patterns are traditionally grouped into three categories: **Creational** (how objects are created), **Structural** (how objects are composed and related), and **Behavioral** (how objects communicate and share responsibilities).

### Creational Patterns

**Singleton** ensures that a class has only one instance and provides a global access point to it. In JavaScript, simply exporting an instance from a module effectively creates a singleton, since ES modules are cached after the first import.

```js
// Module-level singleton — the most natural JS approach
class ConfigService {
  #config = {};

  set(key, value) { this.#config[key] = value; }
  get(key) { return this.#config[key]; }
}

// This single instance is exported — every importer gets the SAME object
export const configService = new ConfigService();

// Angular's providedIn: 'root' achieves the same thing:
// @Injectable({ providedIn: 'root' }) — one instance for the whole app
```

**Factory** provides a way to create objects without specifying the exact class of the object to be created. The factory decides which concrete class to instantiate based on input parameters.

```js
class Circle { area() { return Math.PI * this.radius ** 2; } }
class Rectangle { area() { return this.width * this.height; } }

// The factory hides the creation logic
function createShape(type, dimensions) {
  switch (type) {
    case "circle":
      const c = new Circle();
      c.radius = dimensions.radius;
      return c;
    case "rectangle":
      const r = new Rectangle();
      r.width = dimensions.width;
      r.height = dimensions.height;
      return r;
    default:
      throw new Error(`Unknown shape: ${type}`);
  }
}

// Callers don't need to know about Circle or Rectangle
const shape = createShape("circle", { radius: 5 });
shape.area(); // 78.53...
```

**Builder** constructs complex objects step by step, allowing you to produce different types and representations using the same construction process. The key benefit is that it prevents constructors with many optional parameters ("telescoping constructor" anti-pattern).

```js
// Without builder — hard to read, easy to mix up argument order
const req = new HttpRequest("POST", url, body, headers, timeout, retries, cache);

// With builder — readable and flexible
class HttpRequestBuilder {
  #method = "GET";
  #url = "";
  #headers = {};
  #timeout = 5000;
  #retries = 0;

  method(m) { this.#method = m; return this; } // return this for chaining
  url(u) { this.#url = u; return this; }
  header(key, value) { this.#headers[key] = value; return this; }
  timeout(ms) { this.#timeout = ms; return this; }
  retry(n) { this.#retries = n; return this; }

  build() {
    if (!this.#url) throw new Error("URL is required");
    return new HttpRequest(this.#method, this.#url, this.#headers, this.#timeout, this.#retries);
  }
}

const request = new HttpRequestBuilder()
  .method("POST")
  .url("/api/users")
  .header("Authorization", `Bearer ${token}`)
  .header("Content-Type", "application/json")
  .timeout(3000)
  .retry(2)
  .build();
```

### Structural Patterns

**Module** is arguably the most fundamental pattern in modern JavaScript. It uses closures and ES modules to encapsulate private state and expose a public API. Angular services, components, and the entire library system are built on this pattern.

```js
// IIFE module (pre-ES-modules era, still relevant for understanding)
const Cart = (() => {
  const items = []; // private

  return {
    add(item) { items.push(item); },
    remove(id) { const i = items.findIndex(x => x.id === id); items.splice(i, 1); },
    total() { return items.reduce((sum, item) => sum + item.price, 0); },
    count() { return items.length; }
  };
})();

Cart.add({ id: 1, name: "Widget", price: 9.99 });
Cart.count(); // 1
Cart.items;   // undefined — private, not accessible
```

**Decorator** adds new behaviors to objects by wrapping them in objects that implement the same interface. This is different from the TypeScript/Angular `@Decorator` annotation syntax — it is a structural pattern for composition.

```js
// Base logger
class ConsoleLogger {
  log(message) { console.log(message); }
}

// Decorator that adds timestamps
class TimestampLogger {
  constructor(logger) { this.logger = logger; } // wrap the original

  log(message) {
    this.logger.log(`[${new Date().toISOString()}] ${message}`);
  }
}

// Decorator that adds log levels
class LevelLogger {
  constructor(logger) { this.logger = logger; }

  info(message) { this.logger.log(`[INFO] ${message}`); }
  error(message) { this.logger.log(`[ERROR] ${message}`); }
}

// Compose decorators — stack them like layers
const logger = new LevelLogger(new TimestampLogger(new ConsoleLogger()));
logger.info("Application started");
// logs: [INFO] [2024-01-01T00:00:00.000Z] Application started
```

**Facade** provides a simplified, unified interface to a complex subsystem. In Angular, service classes are often facades — they hide the complexity of HTTP calls, caching, state management, and error handling behind clean methods.

```js
// Complex subsystem (multiple services, transformations, caching)
class UserService {
  #httpClient;
  #cache = new Map();
  #authToken;

  constructor(httpClient, authToken) {
    this.#httpClient = httpClient;
    this.#authToken = authToken;
  }

  // This facade method hides: auth, caching, error handling, transformation
  async getUser(id) {
    if (this.#cache.has(id)) return this.#cache.get(id);

    const raw = await this.#httpClient.get(`/api/users/${id}`, {
      headers: { Authorization: `Bearer ${this.#authToken}` }
    });

    const user = this.#transformUser(raw); // normalize API response
    this.#cache.set(id, user);
    return user;
  }

  #transformUser(raw) {
    return { id: raw.user_id, name: `${raw.first_name} ${raw.last_name}` };
  }
}

// The consumer sees a simple interface:
const user = await userService.getUser(42);
```

**Proxy** creates a wrapper that intercepts and controls access to another object. JavaScript has a built-in `Proxy` constructor for this purpose. Angular's change detection in older versions used similar interception techniques.

```js
function createValidatedObject(target, schema) {
  return new Proxy(target, {
    set(obj, prop, value) {
      if (schema[prop] && !schema[prop](value)) {
        throw new TypeError(`Invalid value for ${String(prop)}: ${value}`);
      }
      obj[prop] = value;
      return true; // must return true to indicate success
    }
  });
}

const user = createValidatedObject({}, {
  age: (v) => typeof v === "number" && v >= 0 && v <= 150,
  email: (v) => typeof v === "string" && v.includes("@")
});

user.age = 25;          // OK
user.age = -5;          // TypeError: Invalid value for age: -5
user.email = "x@y.com"; // OK
```

### Behavioral Patterns

**Observer / PubSub** is the most important pattern for frontend development and the conceptual foundation of Angular's event system, RxJS, and signals. An observable (subject) maintains a list of observers (subscribers) and notifies them when its state changes.

```js
class EventEmitter {
  #listeners = new Map();

  on(event, listener) {
    if (!this.#listeners.has(event)) this.#listeners.set(event, []);
    this.#listeners.get(event).push(listener);
    return () => this.off(event, listener); // return unsubscribe function
  }

  off(event, listener) {
    const listeners = this.#listeners.get(event) || [];
    this.#listeners.set(event, listeners.filter(l => l !== listener));
  }

  emit(event, data) {
    const listeners = this.#listeners.get(event) || [];
    listeners.forEach(listener => listener(data));
  }
}

const emitter = new EventEmitter();
const unsubscribe = emitter.on("userLoggedIn", user => {
  console.log(`Welcome, ${user.name}!`);
});

emitter.emit("userLoggedIn", { name: "Alice" }); // "Welcome, Alice!"
unsubscribe(); // clean up
emitter.emit("userLoggedIn", { name: "Bob" }); // no output — unsubscribed
```

**Strategy** defines a family of algorithms, encapsulates each one, and makes them interchangeable. The client can switch between algorithms without changing the code that uses them.

```js
// Different sorting strategies
const sortStrategies = {
  byName: (items) => [...items].sort((a, b) => a.name.localeCompare(b.name)),
  byPrice: (items) => [...items].sort((a, b) => a.price - b.price),
  byDate: (items) => [...items].sort((a, b) => a.date - b.date),
};

class ProductList {
  constructor(products) { this.products = products; }

  sort(strategy = "byName") {
    const sorter = sortStrategies[strategy];
    if (!sorter) throw new Error(`Unknown strategy: ${strategy}`);
    return sorter(this.products);
  }
}

const list = new ProductList(products);
list.sort("byPrice"); // easily switch strategies
```

---

## 4.17 Functional Programming in JavaScript

Functional Programming (FP) is a programming paradigm that treats computation as the evaluation of mathematical functions and avoids changing state or mutable data. You do not need to fully embrace FP to benefit from its principles — even applying a few of its ideas dramatically improves code quality. RxJS is heavily functional in design, which is why understanding FP is a prerequisite for mastering it.

### Immutability

In functional programming, data is never modified after it is created. Instead, you create new data structures with the desired changes. This makes your code predictable (a value never changes unexpectedly) and enables features like time-travel debugging in NgRx.

```js
// Mutable approach — modifying data in place
const state = { count: 0, user: { name: "Alice" } };
state.count++;  // mutation! Anyone holding a reference to state sees this
state.user.name = "Bob"; // deep mutation

// Immutable approach — creating new objects
const newState = { ...state, count: state.count + 1 }; // new object
const newState2 = {
  ...state,
  user: { ...state.user, name: "Bob" } // new nested object
};
// state is unchanged; newState and newState2 are new objects

// Immutable array operations
const items = [1, 2, 3, 4, 5];
const withSix = [...items, 6];          // add to end
const withoutFirst = items.slice(1);    // remove first
const updated = items.map((item, i) => i === 2 ? 99 : item); // update at index 2

// NgRx reducers must be immutable — this is why:
function reducer(state = initialState, action) {
  switch (action.type) {
    case "INCREMENT":
      return { ...state, count: state.count + 1 }; // new state object!
    default:
      return state; // return same reference if unchanged — enables === comparison
  }
}
```

### Pure Functions

A pure function is one that (1) given the same inputs, always returns the same output, and (2) has no side effects. Pure functions are inherently testable, composable, and safe to run in parallel.

```js
// Pure — depends only on inputs, no side effects
const formatCurrency = (amount, currency = "USD") =>
  new Intl.NumberFormat("en-US", { style: "currency", currency }).format(amount);

formatCurrency(42.5);          // "$42.50" — same every time
formatCurrency(42.5);          // "$42.50" — same every time

// Impure — depends on external state
let taxRate = 0.2;
const calculateTax = (amount) => amount * taxRate; // depends on outer taxRate

// Impure — side effect (modifies external state)
const addToCart = (item) => {
  cart.push(item); // modifies external `cart` variable
  return cart;
};

// Pure version of addToCart
const addToCart = (cart, item) => [...cart, item]; // takes cart as input, returns new cart
```

### Function Composition and Piping

Composition is the art of building complex transformations from simple, tested pieces. The `pipe` function (left-to-right composition) is the most readable pattern and is directly used in RxJS — `observable.pipe(map(...), filter(...), switchMap(...))`.

```js
// Individual pure transformations
const trim = str => str.trim();
const toLower = str => str.toLowerCase();
const removeSpaces = str => str.replace(/\s+/g, "-");
const addPrefix = prefix => str => `${prefix}-${str}`;

// pipe — applies functions left to right
const pipe = (...fns) => value => fns.reduce((v, fn) => fn(v), value);

const slugify = pipe(trim, toLower, removeSpaces, addPrefix("article"));
slugify("  Hello World  "); // "article-hello-world"

// This is exactly how RxJS pipe works:
// source$.pipe(
//   map(x => x * 2),      — pure transformation
//   filter(x => x > 5),   — pure filter
//   take(3)               — pure limiter
// )
```

### Map, Filter, Reduce as Core Abstractions

These three operations are the foundation of functional data transformation. Any data pipeline — from loading users to normalizing NgRx state — can be expressed as a combination of map, filter, and reduce.

```js
const orders = [
  { id: 1, status: "pending",  items: 3, total: 45.00, userId: 10 },
  { id: 2, status: "complete", items: 1, total: 12.50, userId: 12 },
  { id: 3, status: "pending",  items: 5, total: 89.99, userId: 10 },
  { id: 4, status: "complete", items: 2, total: 34.00, userId: 15 },
];

// Get the total revenue from completed orders over $20
const totalRevenue = orders
  .filter(order => order.status === "complete" && order.total > 20) // keep matching
  .map(order => order.total)                                         // extract totals
  .reduce((sum, total) => sum + total, 0);                          // sum them up
// Result: 34.00

// Group orders by userId
const ordersByUser = orders.reduce((acc, order) => {
  const key = order.userId;
  return { ...acc, [key]: [...(acc[key] || []), order] };
}, {});
// { 10: [order1, order3], 12: [order2], 15: [order4] }

// Transform into a summary per user
const userSummaries = Object.entries(ordersByUser).map(([userId, userOrders]) => ({
  userId: Number(userId),
  orderCount: userOrders.length,
  totalSpent: userOrders.reduce((sum, o) => sum + o.total, 0)
}));
```

### Point-Free Style

Point-free (or tacit) style is writing functions without explicitly mentioning the data they operate on. It makes pipelines more concise by using function references instead of wrapper lambdas.

```js
// Not point-free — explicitly mentions the data argument
const doubled = numbers.map(n => n * 2);

// Point-free — the function IS the operation, no intermediate variable
const double = n => n * 2;
const doubled = numbers.map(double);

// Even more point-free
const multiply = factor => n => n * factor;
const double = multiply(2);
const triple = multiply(3);

[1, 2, 3].map(double);  // [2, 4, 6]
[1, 2, 3].map(triple);  // [3, 6, 9]
```

### Avoiding Shared State

Shared mutable state is the root of many concurrency bugs and unexpected behaviors. Functions should communicate through their parameters and return values, not by reaching into shared variables.

```js
// Bad — functions communicate through shared mutable state
let total = 0;
function addItem(price) { total += price; } // reads and mutates shared `total`
function applyDiscount(pct) { total *= (1 - pct); }

// Good — functions take and return values (pure data pipeline)
const addItem = (cart, price) => ({ ...cart, total: cart.total + price });
const applyDiscount = (cart, pct) => ({ ...cart, total: cart.total * (1 - pct) });

const cart = { total: 0, items: [] };
const cart1 = addItem(cart, 10);
const cart2 = addItem(cart1, 20);
const cart3 = applyDiscount(cart2, 0.1); // 10% off
```

---

## 4.18 JavaScript Performance

Understanding how JavaScript affects performance at a fundamental level makes you a better Angular developer and helps you make informed decisions about which patterns to use.

### Memory Management and Garbage Collection

JavaScript manages memory automatically through *garbage collection*. The most common algorithm is *mark-and-sweep*: the engine periodically marks all objects reachable from root variables (global scope, active call stack) and then sweeps away — deallocates — everything that was not marked.

The implication for you: as long as a reference to an object exists, it will never be garbage collected, even if you think you are done with it. This is how *memory leaks* happen.

### Memory Leaks

A memory leak is when memory is allocated but never released because references are accidentally retained. Over time, leaked memory grows until the app slows down or crashes. The four most common sources of leaks in JavaScript are closures holding large objects, detached DOM nodes, event listeners that are never removed, and interval/timeout callbacks that hold references.

```js
// Leak 1 — event listener never removed
class Component {
  constructor() {
    this.#data = new Array(10000).fill("data"); // large object
    // This closure keeps `this` alive even after the component is destroyed
    window.addEventListener("resize", () => this.handleResize());
  }
  // If the component is destroyed without removing the listener, it leaks
}

// Fix — remove the listener on cleanup
class Component {
  #handleResize = () => this.handleResize(); // named for reference

  constructor() {
    window.addEventListener("resize", this.#handleResize);
  }

  destroy() {
    window.removeEventListener("resize", this.#handleResize);
    // Now the Component can be garbage collected
  }
}

// Leak 2 — setInterval keeps running
const intervalId = setInterval(() => {
  fetch("/api/ping").then(updateUI); // keeps running even after navigation
}, 5000);
// Fix: clearInterval(intervalId) in a cleanup/destroy lifecycle method
```

> **Angular note:** Angular's `ngOnDestroy` lifecycle hook is specifically for preventing memory leaks. The `takeUntilDestroyed()` operator (Angular 16+) and the `DestroyRef` injection are Angular's built-in mechanisms for automatically unsubscribing from RxJS observables when a component is destroyed.

### V8 Engine Optimizations

Understanding a few V8 (Chrome/Node.js JavaScript engine) optimization principles helps you write faster code. V8 uses JIT (Just-In-Time) compilation, which compiles hot (frequently executed) code paths to native machine code.

*Hidden classes* — V8 creates internal type representations (hidden classes) for objects. When all objects of the same "shape" (same properties in the same order) share a hidden class, V8 can optimize property access dramatically. Avoid adding or deleting properties after object creation, and always create objects in the same order.

```js
// Good — consistent shape, one hidden class
function createPoint(x, y) { return { x, y }; }
const p1 = createPoint(1, 2);
const p2 = createPoint(3, 4); // same hidden class as p1

// Bad — different property orders, different hidden classes
const p3 = { x: 1, y: 2 };
const p4 = { y: 2, x: 1 }; // V8 creates a DIFFERENT hidden class
```

### Debounce and Throttle

When an event fires very rapidly (e.g., window resize, search input, scroll), you often don't want to run an expensive handler on every single invocation.

*Debounce* waits until there has been a pause of at least `n` milliseconds before running the function. Perfect for search-as-you-type: don't fetch until the user stops typing.

*Throttle* ensures the function runs at most once per `n` milliseconds regardless of how many times it is called. Perfect for scroll handlers: execute at most once per 100ms even if scroll fires hundreds of times.

```js
// Debounce — run after the user stops for 300ms
function debounce(fn, delay) {
  let timeoutId;
  return function(...args) {
    clearTimeout(timeoutId);
    timeoutId = setTimeout(() => fn.apply(this, args), delay);
  };
}

const searchInput = document.querySelector("#search");
searchInput.addEventListener("input", debounce(event => {
  fetchSearchResults(event.target.value);
}, 300));

// Throttle — run at most once per 100ms
function throttle(fn, limit) {
  let lastCall = 0;
  return function(...args) {
    const now = Date.now();
    if (now - lastCall >= limit) {
      lastCall = now;
      fn.apply(this, args);
    }
  };
}

window.addEventListener("scroll", throttle(() => {
  updateScrollProgress();
}, 100));
```

> **In Angular:** RxJS provides `debounceTime()` and `throttleTime()` operators that do the same thing for Observable streams, and they integrate perfectly with Angular's form value changes and event streams. In most Angular code you will use these operators rather than the manual implementations above.

### Avoid Layout Thrashing

*Layout thrashing* (also called "forced synchronous layout") happens when you repeatedly read and write to the DOM in an interleaved way. Each write invalidates the layout, and each read forces the browser to recalculate layout before it can return a value. This can reduce a page that runs at 60fps to 5fps.

```js
// BAD — read/write interleaved (forces layout recalculation on each read)
elements.forEach(el => {
  const height = el.offsetHeight;  // READ — forces layout
  el.style.height = height + 10 + "px"; // WRITE — invalidates layout
  // Next iteration: READ again — forces layout again
});

// GOOD — batch reads first, then batch writes
const heights = elements.map(el => el.offsetHeight); // all READS first
elements.forEach((el, i) => {
  el.style.height = heights[i] + 10 + "px"; // all WRITES after
});
// Browser only calculates layout once between reads and writes
```

Angular's `afterNextRender()` hook is designed to help with this — it ensures your DOM reads happen in a stable, post-render phase rather than during an update cycle.

---

## 4.19 Modern JavaScript — ES2015 through ES2025

Each year, the TC39 committee releases new JavaScript features. Knowing the key additions from each year helps you read modern code, use the most expressive syntax available, and understand why certain patterns exist.

### ES2015 (ES6) — The Revolution

ES2015 was the largest single upgrade to JavaScript in history. It introduced: `let`/`const` (block scoping), arrow functions, template literals, destructuring (objects and arrays), default parameters, rest/spread operators, ES Modules (`import`/`export`), Promises (native), Symbol, Map and Set (proper key-value collections), WeakMap and WeakSet, iterators and generators, and the `class` keyword.

### ES2017 — Async Made Readable

The headline feature was `async`/`await`, which made asynchronous code look and behave like synchronous code. Also added were `Object.values()`, `Object.entries()`, and `String.padStart()`/`padEnd()`.

```js
// String padding — useful for formatting tables or IDs
"5".padStart(3, "0"); // "005"
"hi".padEnd(10, "."); // "hi........"
```

### ES2018 — Spread for Objects and Async Iteration

Object rest/spread was formalized (`{ ...obj }`), `Promise.finally()` was added, and async iteration (`for await...of`) arrived.

### ES2019 — Practical Utilities

`Array.flat()` and `Array.flatMap()` were added, along with `Object.fromEntries()` (convert an array of entries back to an object) and the optional catch binding (you can omit the error variable in `catch` when you don't need it).

```js
try { riskyOperation(); } catch { showError(); } // no need for `catch (e)` if unused
```

### ES2020 — Safety Operators and BigInt

`BigInt` for large integers, `Promise.allSettled()`, `globalThis` (a universal way to access the global object across environments), optional chaining (`?.`), and nullish coalescing (`??`) were the major additions.

### ES2021 — Convenience

Logical assignment operators (`&&=`, `||=`, `??=`), `Promise.any()` (first to fulfill wins), `String.replaceAll()`, and `WeakRef` (hold a weak reference that doesn't prevent garbage collection).

```js
const text = "foo bar foo baz foo";
text.replaceAll("foo", "qux"); // "qux bar qux baz qux"
// Previously required: text.replace(/foo/g, "qux")
```

### ES2022 — Class Fields and Array Improvements

Private class fields (`#field`) and methods (`#method()`) were standardized. `Object.hasOwn(obj, key)` was added as a more reliable alternative to `obj.hasOwnProperty(key)`. `Array.at()` allows negative indexing. `Error.cause` enables chaining errors with context. Top-level `await` in modules was finalized.

```js
// Array.at() — access from the end with negative index
const arr = [1, 2, 3, 4, 5];
arr.at(0);   // 1 (same as arr[0])
arr.at(-1);  // 5 — the last element!
arr.at(-2);  // 4 — second to last!
// Previously: arr[arr.length - 1] — much more verbose

// Error.cause — chain errors with context
try {
  await loadUserData();
} catch (err) {
  throw new Error("Failed to initialize dashboard", { cause: err });
  // The original error is preserved as `cause`
}
```

### ES2023 — Immutable Array Methods

`toSorted()`, `toReversed()`, `toSpliced()`, and `with()` — non-mutating counterparts to the historically mutating methods. Also added `findLast()` and `findLastIndex()`.

```js
const arr = [3, 1, 2];
const sorted = arr.toSorted(); // [1, 2, 3] — arr unchanged
const reversed = arr.toReversed(); // [2, 1, 3] — arr unchanged
const updated = arr.with(0, 99); // [99, 1, 2] — replace index 0
```

### ES2024 — Groups and Promise Improvements

`Object.groupBy()` and `Map.groupBy()` — native grouping of collections. `Promise.withResolvers()` — a cleaner way to create manually controlled promises. `ArrayBuffer.prototype.resize()` for resizable binary data.

```js
// Object.groupBy — replaces the common reduce-based grouping pattern
const products = [
  { name: "Widget", category: "tools" },
  { name: "Gadget", category: "electronics" },
  { name: "Hammer", category: "tools" }
];
const grouped = Object.groupBy(products, p => p.category);
// { tools: [Widget, Hammer], electronics: [Gadget] }

// Promise.withResolvers — cleaner than new Promise((resolve, reject) => ...)
const { promise, resolve, reject } = Promise.withResolvers();
// Useful when resolve/reject need to be called from outside the executor
button.addEventListener("click", resolve);
```

### ES2025 — Iterator Helpers and Set Methods

*Iterator helpers* add `map`, `filter`, `take`, `drop`, `flatMap`, `reduce`, and `toArray` directly onto Iterator objects, making it possible to lazily process any iterable without converting to an array first.

*Set methods* add mathematical set operations: `union()`, `intersection()`, `difference()`, `symmetricDifference()`, `isSubsetOf()`, and `isSupersetOf()`.

*Import attributes* (`with { type: "json" }`) allow you to import JSON files and other non-JS resources directly in ES module syntax.

```js
// Iterator helpers — lazy processing without creating intermediate arrays
const result = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
  .values()           // get iterator
  .filter(n => n % 2 === 0) // lazy — nothing runs yet
  .map(n => n * n)          // lazy
  .take(3)                  // lazy
  .toArray();               // NOW it runs: [4, 16, 36]

// Set operations — very useful for permissions and tag systems
const adminPerms = new Set(["read", "write", "delete", "admin"]);
const userPerms = new Set(["read", "write"]);

userPerms.isSubsetOf(adminPerms);                    // true
adminPerms.difference(userPerms);                    // Set {"delete", "admin"}
adminPerms.intersection(userPerms);                  // Set {"read", "write"}

// JSON import (with attributes)
import config from "./config.json" with { type: "json" };
```

### Upcoming Stage 3 Proposals

These proposals are stable enough that you may encounter them in TypeScript (which often implements Stage 3 proposals). **Explicit Resource Management** (`using`/`await using`) automatically calls a `dispose()` method when a variable goes out of scope — like C#'s `using` statement or Python's `with` statement. TypeScript 5.2+ already supports this.

```js
// Explicit Resource Management — auto-cleanup
function createConnection() {
  const conn = openDatabaseConnection();
  return {
    query: conn.query.bind(conn),
    [Symbol.dispose]() { conn.close(); } // cleanup method
  };
}

{
  using conn = createConnection();
  await conn.query("SELECT * FROM users");
  // conn[Symbol.dispose]() is called AUTOMATICALLY when the block exits
  // Even if an exception is thrown!
}
```

---

## 🔑 Key Takeaways — Part 6

Design patterns give you a vocabulary for discussing code structure and a toolkit for solving recurring problems. The Observer pattern is foundational to understanding Angular's event system, RxJS, and signals. The Facade pattern is how every good Angular service is structured. Functional programming principles — immutability, pure functions, and composition — directly inform how NgRx state management and RxJS operators work. Memory management is not just academic: memory leaks from unremoved event listeners and un-unsubscribed observables are real production issues in Angular apps, and `ngOnDestroy` with `takeUntilDestroyed()` is your solution. From ES2015 to ES2025, each version of JavaScript has added features that make code safer and more expressive. The most impactful for Angular development are optional chaining (`?.`), nullish coalescing (`??`), `async`/`await`, the immutable array methods from ES2023, and the Promise utilities from ES2024.

---

*Part 6 of 6 — Phase 4 JavaScript Mastery | Return to [Phase 4 Index](./phase4-index.md)*

---

*Phase 4 complete. You are now ready for Phase 5 — TypeScript Mastery.*

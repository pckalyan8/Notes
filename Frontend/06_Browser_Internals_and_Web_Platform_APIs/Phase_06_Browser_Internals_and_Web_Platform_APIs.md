# Phase 6 — Browser Internals & Web Platform APIs
## Frontend Angular Mastery Roadmap
> **Duration:** 2–3 weeks | **Goal:** Understand the browser deeply

---

## Table of Contents

1. [6.1 — Browser Rendering Pipeline](#61--browser-rendering-pipeline)
2. [6.2 — JavaScript Engine Internals](#62--javascript-engine-internals)
3. [6.3 — The Event Loop Deep Dive](#63--the-event-loop-deep-dive)
4. [6.4 — Web Storage APIs](#64--web-storage-apis)
5. [6.5 — Communication APIs](#65--communication-apis)
6. [6.6 — Security APIs](#66--security-apis)
7. [6.7 — Performance APIs](#67--performance-apis)
8. [6.8 — Intersection & Visibility](#68--intersection--visibility)
9. [6.9 — Offline & Background APIs](#69--offline--background-apis)
10. [6.10 — Device & Hardware APIs](#610--device--hardware-apis)
11. [6.11 — Modern Web Platform APIs (2024–2026)](#611--modern-web-platform-apis-20242026)

---

## 6.1 — Browser Rendering Pipeline

### What Is the Rendering Pipeline?

When a browser receives an HTML document from a server, it doesn't simply "display" it. It follows a precise, multi-step pipeline to convert raw bytes of HTML, CSS, and JavaScript into the pixels you see on screen. Understanding this pipeline is fundamental to writing performant frontend code — because every step has a cost, and knowing which steps are triggered by which operations tells you exactly where bottlenecks come from.

The pipeline can be summarised as:

```
Bytes → Characters → Tokens → Nodes → DOM
                                         \
Bytes → Characters → Tokens → Nodes → CSSOM
                                         /
                                   Render Tree
                                       |
                                    Layout
                                       |
                                     Paint
                                       |
                                   Compositing
```

---

### Step 1: Parsing HTML → DOM Construction

The browser receives raw bytes over the network. It decodes them into characters using the charset declared in the document (typically UTF-8). It then runs a tokeniser that converts characters into tokens — start tags, end tags, attribute values, text nodes, etc.

These tokens are fed into the **HTML parser**, which builds the **Document Object Model (DOM)** — a tree of nodes representing every element, attribute, and text node in your HTML.

```html
<html>
  <body>
    <h1>Hello</h1>
    <p>World</p>
  </body>
</html>
```

This becomes a tree:

```
Document
└── html
    └── body
        ├── h1 → "Hello"
        └── p  → "World"
```

**Important:** HTML parsing is *incremental and forgiving*. The parser doesn't wait for the full document — it starts building the DOM as bytes arrive. It also recovers gracefully from malformed HTML rather than throwing errors (unlike XML).

**Parser-blocking resources:** When the parser encounters a `<script>` tag without `async` or `defer`, it **stops parsing** and waits for the script to download and execute before continuing. This is why `<script>` tags have traditionally been placed at the bottom of `<body>`, and why modern practice uses `defer` or `type="module"`.

---

### Step 2: Parsing CSS → CSSOM Construction

While the DOM is being built, the browser also parses CSS stylesheets into the **CSS Object Model (CSSOM)** — another tree structure representing all the style rules and their relationships.

Unlike HTML parsing, **CSS parsing is not incremental** — the browser must parse the entire CSS file before it can use it, because later rules can override earlier ones. This means CSS is **render-blocking by default**: the browser won't render anything until all CSS has been parsed.

```css
body { font-size: 16px; }
p    { color: blue; }
```

This becomes a CSSOM node for `<p>` that knows its font-size is 16px (inherited from body) and color is blue.

**Important:** A large CSS file, or a CSS file on a slow connection, will delay the first render. This is why inlining critical CSS and loading non-critical CSS asynchronously is a significant performance optimisation.

---

### Step 3: Render Tree Construction

The browser combines the DOM and CSSOM into a **Render Tree**. This tree contains only the nodes that will actually be painted — nodes with `display: none` are excluded entirely (they exist in the DOM but not the render tree). Pseudo-elements like `::before` and `::after` are *included* in the render tree even though they don't exist in the DOM.

Each node in the render tree carries its computed style — the final resolved value of every CSS property after the cascade has been applied.

```
Render Tree Node: <p>
  - color: blue
  - font-size: 16px
  - display: block
  - ... (all computed styles)
```

---

### Step 4: Layout (Reflow)

With the render tree constructed, the browser now has to figure out where every element should go on the screen — its exact position and size. This process is called **Layout** (or **Reflow** when it happens after the initial layout).

The browser traverses the render tree from top to bottom, calculating the geometry of every box. This is a recursive process: the size of a child element can affect the size of its parent (think of how `height: auto` works).

Layout is **expensive**. Anything that changes an element's geometry — width, height, font-size, padding, margin, adding/removing elements — triggers a layout recalculation. On a page with many elements, this can be slow.

**What triggers layout?**
- Reading geometry properties: `offsetWidth`, `offsetHeight`, `getBoundingClientRect()`, `scrollTop`
- Writing to geometry properties: changing `width`, `height`, `padding`, etc.
- Adding or removing DOM nodes

**Layout Thrashing:** Alternating between reading and writing layout properties inside a loop is called layout thrashing, and it forces the browser to recalculate layout repeatedly:

```javascript
// BAD — causes layout thrashing (forces layout on every iteration)
elements.forEach(el => {
  const width = el.offsetWidth;     // READ  → forces layout
  el.style.width = width + 10 + 'px'; // WRITE → invalidates layout
});

// GOOD — batch reads first, then writes
const widths = elements.map(el => el.offsetWidth); // batch READ
elements.forEach((el, i) => {
  el.style.width = widths[i] + 10 + 'px';          // batch WRITE
});
```

---

### Step 5: Paint

After layout, the browser knows where every element is. Now it has to fill in the pixels — drawing text, colours, images, borders, shadows, etc. This is the **Paint** step.

Paint is done by dividing the page into layers and painting each layer separately. Elements are typically painted in the following order (the **painter's algorithm**): background colour → background image → borders → children → outlines.

**What triggers paint?**
- Colour changes (`background-color`, `color`, `border-color`)
- Visibility changes (`opacity`, `visibility`)
- Box shadows, text shadows

Note that changing `transform` or `opacity` alone does *not* trigger a paint — the browser can handle these entirely in the compositing step, which is why they are so cheap for animations.

---

### Step 6: Compositing

Modern browsers paint the page into multiple **layers** (also called compositor layers or paint layers). In the final step, these layers are assembled together — like layering transparencies — to produce the final image. This is done on the **GPU** (graphics processing unit), which is extremely fast at this operation.

The browser automatically promotes certain elements to their own compositor layer when it detects they will change frequently:

- Elements with `transform`, `opacity`, `will-change: transform`
- `<video>` and `<canvas>` elements
- Elements with 3D transforms

When an animation only affects composited properties (`transform`, `opacity`), the browser skips the Layout and Paint steps entirely and only does compositing on the GPU. This is why CSS animations of `transform` and `opacity` are so smooth — they run at 60fps even on mid-range hardware.

```css
/* GOOD — compositor-only, runs on GPU */
.animate-good {
  transition: transform 0.3s ease, opacity 0.3s ease;
}

/* BAD — triggers layout and paint on every frame */
.animate-bad {
  transition: left 0.3s ease, width 0.3s ease;
}
```

---

### Critical Rendering Path Optimisation

The **Critical Rendering Path (CRP)** is the sequence of steps the browser must complete before the first pixel appears on screen. Optimising it directly improves your Largest Contentful Paint (LCP) and First Contentful Paint (FCP) scores.

Key strategies:

**1. Minimise render-blocking resources.** Use `defer` or `async` on scripts; load non-critical CSS asynchronously.

**2. Inline critical CSS.** The CSS needed to style above-the-fold content can be inlined in `<style>` tags in `<head>` to eliminate the network round-trip.

**3. Reduce DOM size.** Large DOMs (over ~1,500 nodes) slow down style calculation and layout.

**4. Avoid forced synchronous layouts.** Don't read layout properties immediately after writing to them.

**5. Promote stable animations to compositor layers** using `transform` and `opacity` rather than `top/left/width/height`.

> **Best Practice:** Use Chrome DevTools' Performance panel to record a page load and look at the colour-coded waterfall — **blue** is HTML parsing, **yellow** is scripting, **purple** is layout, **green** is paint. Wherever you see large purple or green blocks, that's your optimisation target.

---

## 6.2 — JavaScript Engine Internals

### What Is a JavaScript Engine?

A JavaScript engine is the component inside a browser (or runtime like Node.js) that reads, parses, and executes JavaScript code. The dominant engine today is **V8**, used in Chrome, Edge, Node.js, and Deno. Firefox uses **SpiderMonkey**, and Safari uses **JavaScriptCore** (also called Nitro).

Understanding how V8 works tells you *why* certain JavaScript patterns are faster than others — it's not magic, it's engineering.

---

### V8 Architecture Overview

V8 processes JavaScript through several stages:

```
Source Code
    ↓
  Parser → AST (Abstract Syntax Tree)
    ↓
 Ignition (Interpreter) → Bytecode
    ↓
TurboFan (JIT Compiler) → Optimised Machine Code
    ↓
 Deoptimisation (when assumptions are violated)
```

---

### Parsing → AST

The first thing V8 does is **parse** your source code into an **Abstract Syntax Tree (AST)**. The AST is a tree representation of your code's grammatical structure.

```javascript
const x = 1 + 2;
```

The AST for this would have a `VariableDeclaration` node containing a `BinaryExpression` node (`+`) with two `Literal` children (`1` and `2`).

**Lazy parsing:** V8 uses a technique called lazy (or pre-) parsing to speed up startup. When it encounters a function, it does a quick pre-parse to check syntax validity but doesn't build the full AST. The full parse only happens when the function is actually called. This is why functions that are called immediately (like IIFEs) should be wrapped in parentheses — it hints to V8 to parse them eagerly.

```javascript
// Hint to V8: parse this eagerly (it'll be called immediately)
(function() { /* ... */ })();
```

---

### Ignition (Interpreter) → Bytecode

After building the AST, V8 sends it through **Ignition**, its interpreter. Ignition compiles the AST down to **bytecode** — a compact, lower-level representation that Ignition can execute efficiently without generating full machine code.

Bytecode is smaller than machine code and faster to generate, making it excellent for code that runs once or infrequently (startup code, rarely called functions). Ignition executes this bytecode while also collecting **profiling information** about how the code is used.

---

### TurboFan (JIT Compiler) → Optimised Machine Code

As Ignition executes bytecode, it notices which functions are called frequently — these are **hot functions**. It passes profiling data from these hot functions to **TurboFan**, V8's Just-In-Time (JIT) optimising compiler.

TurboFan makes **speculative assumptions** about the code. For example, if a function has always been called with two numbers as arguments, TurboFan generates highly optimised machine code that *assumes the arguments are always numbers*. This optimised code runs many times faster than interpreted bytecode.

---

### Hidden Classes and Inline Caching

V8's two most important micro-optimisation mechanisms are hidden classes and inline caching.

**Hidden Classes (Shapes/Maps):**

JavaScript objects are dynamic — you can add and remove properties at will. But V8 doesn't treat every object as a generic dictionary. Instead, it assigns a **hidden class** to each object that describes its current "shape" (what properties it has and in what order). When multiple objects share the same shape, V8 can use efficient, struct-like memory access instead of dictionary lookups.

```javascript
// GOOD — both objects get the same hidden class
function Point(x, y) {
  this.x = x; // property 'x' added first
  this.y = y; // property 'y' added second
}
const p1 = new Point(1, 2);
const p2 = new Point(3, 4);
// p1 and p2 share the same hidden class → fast

// BAD — objects get different hidden classes
const a = {};
a.x = 1;
a.y = 2;

const b = {};
b.y = 2; // 'y' added before 'x' — different order!
b.x = 1;
// a and b have DIFFERENT hidden classes → slower
```

**Inline Caching (IC):**

When V8 encounters a property access like `obj.x`, it caches the hidden class of `obj` and the memory offset of `x` at that call site. On subsequent calls, V8 checks if the object has the same hidden class — if yes, it reads `x` directly from the cached offset without any lookup. This is called a **monomorphic** inline cache and is extremely fast.

If the same property access site sees objects of *two* different hidden classes, the cache becomes **polymorphic** (slower). With three or more different classes, it becomes **megamorphic** (much slower, falls back to generic lookup).

> **Best Practice:** Always initialise all properties of an object in the constructor, in the same order, every time. Avoid adding properties to objects after creation. This keeps your objects monomorphic and V8 happy.

---

### Deoptimisation

If TurboFan's speculative assumptions are ever violated — for example, someone passes a string to a function that was always called with numbers — V8 **deoptimises** that function. It throws away the optimised machine code and falls back to interpreting bytecode, then re-profiles and potentially re-optimises later.

```javascript
function add(a, b) { return a + b; }

// V8 optimises assuming a and b are always numbers
add(1, 2);
add(3, 4);
add(5, 6);

// Deoptimisation! V8 must re-optimise for string case
add("hello", " world");
```

> **Best Practice:** Keep functions **monomorphic** — call them with the same types of arguments consistently. Don't mix types for the same function parameter. This is one reason TypeScript's type discipline naturally leads to better-optimised JavaScript.

---

### Garbage Collection

V8 manages memory automatically through **Garbage Collection (GC)**. When objects are no longer reachable from any root (global variables, the call stack, etc.), they become eligible for GC.

V8 uses a **generational GC** strategy based on the empirical observation that most objects die young:

**Young Generation (Nursery/Scavenger):** Newly created objects go here. GC happens frequently but quickly — only live objects are copied to the "from" space of the next cycle. Dead objects are simply abandoned.

**Old Generation (Major GC):** Objects that survive multiple young generation GCs are promoted here. GC happens less frequently but is more expensive. V8 uses a **mark-and-sweep** algorithm: it marks all reachable objects, then sweeps and frees the unmarked ones.

**Incremental and concurrent GC:** Modern V8 performs GC incrementally (in small slices interleaved with JavaScript execution) and concurrently (GC work happens on background threads) to avoid long pauses that would cause jank.

**Common memory leaks to avoid:**

```javascript
// LEAK 1: Forgotten event listeners
const button = document.querySelector('button');
button.addEventListener('click', handler); // add
// ... later, button is removed from DOM but handler still holds reference
// FIX: always removeEventListener when cleaning up

// LEAK 2: Closures holding onto large data
function createLeak() {
  const hugeArray = new Array(1000000);
  return function() { return hugeArray[0]; }; // hugeArray never freed
}

// LEAK 3: Detached DOM nodes stored in variables
let detached = document.querySelector('#my-element');
document.body.removeChild(detached);
// 'detached' variable still holds reference — element not GC'd
detached = null; // FIX: explicitly null the reference
```

---

## 6.3 — The Event Loop Deep Dive

### The Concurrency Model

JavaScript is a **single-threaded** language — it can only execute one piece of code at a time. Yet it handles concurrent operations like HTTP requests, timers, and user events without blocking. How? Through the **Event Loop**.

To understand the event loop, you need to understand five components working together: the **Call Stack**, **Web APIs**, the **Macrotask Queue** (callback queue), the **Microtask Queue**, and the **Event Loop** itself.

---

### The Call Stack

The call stack is a LIFO (Last In, First Out) data structure that tracks the execution context. Every time a function is called, a new **stack frame** is pushed onto the stack. When the function returns, its frame is popped off.

```javascript
function c() { console.log('c'); }
function b() { c(); }
function a() { b(); }

a();

// Call stack progression:
// [a]       → a calls b
// [a, b]    → b calls c
// [a, b, c] → c logs, returns
// [a, b]    → b returns
// [a]       → a returns
// []        → stack empty
```

When the stack is empty, the event loop can push the next item from the queues.

---

### Web APIs

Browsers provide APIs that handle async operations *outside* of the JavaScript thread: `setTimeout`, `setInterval`, `fetch`, `addEventListener`, `requestAnimationFrame`, etc.

When you call `setTimeout(fn, 1000)`, JavaScript hands the timer off to the browser's **Web API layer** and immediately returns. The timer runs in the browser (on a C++ thread, conceptually). After 1000ms, the Web API places `fn` into the macrotask queue.

---

### The Macrotask Queue (Task Queue)

The macrotask queue holds callbacks from:
- `setTimeout` / `setInterval`
- DOM events (click, keydown, etc.)
- `MessageChannel`
- `setImmediate` (Node.js)
- `requestAnimationFrame` (roughly — it's special, runs before paint)

The event loop processes **one macrotask per iteration**. After each macrotask, it fully drains the microtask queue before picking the next macrotask.

---

### The Microtask Queue

The microtask queue is a higher-priority queue that holds:
- Promise `.then()` / `.catch()` / `.finally()` callbacks
- `queueMicrotask()` calls
- `MutationObserver` callbacks
- `async/await` continuations (which are syntactic sugar for Promises)

**Critical rule:** After every macrotask, and after every individual microtask, the browser checks if the microtask queue is non-empty. If it is, it drains it **completely** before doing anything else — including rendering.

```
Event Loop Iteration:
1. Pick ONE macrotask from macrotask queue → execute it
2. Drain microtask queue COMPLETELY (including microtasks added by microtasks)
3. Perform rendering update (if needed, and if this is a "rendering opportunity")
4. Go back to step 1
```

---

### Visualising the Queues

```javascript
console.log('1 — script start');

setTimeout(() => console.log('5 — setTimeout'), 0);

Promise.resolve()
  .then(() => console.log('3 — promise 1'))
  .then(() => console.log('4 — promise 2'));

console.log('2 — script end');

// Output:
// 1 — script start
// 2 — script end
// 3 — promise 1
// 4 — promise 2
// 5 — setTimeout
```

**Why this order?**

The entire script runs as one macrotask. During execution, `setTimeout` is registered with Web APIs, and two Promise `.then()` callbacks are queued into the microtask queue. After the script finishes (macrotask complete), the event loop drains the microtask queue — printing "promise 1" and "promise 2". Only then does the `setTimeout` callback (a macrotask) get picked up.

---

### requestAnimationFrame Timing

`requestAnimationFrame` (rAF) sits between microtasks and the next rendering step. The browser calls rAF callbacks just before it paints the screen, approximately 60 times per second (every ~16.67ms).

```
Event Loop with rAF:
1. Macrotask
2. Microtasks (drain)
3. requestAnimationFrame callbacks (if rendering opportunity)
4. Style + Layout + Paint + Composite
5. Repeat
```

This is why rAF is the correct tool for smooth animations — your callback runs right before the browser paints, giving you the latest coordinates and a guarantee that any DOM changes you make will be reflected in the next frame.

```javascript
// Smooth animation using rAF
function animate(timestamp) {
  element.style.transform = `translateX(${timestamp / 10 % 300}px)`;
  requestAnimationFrame(animate); // schedule next frame
}
requestAnimationFrame(animate);
```

---

### Long Tasks and INP (Interaction to Next Paint)

A **Long Task** is any macrotask that takes longer than 50ms to execute. During a long task, the browser cannot respond to user interactions, paint new frames, or process other events. This causes **jank** — the UI feels frozen or sluggish.

**INP (Interaction to Next Paint)** is a Core Web Vital that measures the delay between a user interaction (click, tap, key press) and the next paint that reflects that interaction. Long Tasks are the primary cause of poor INP scores.

```javascript
// BAD — blocks the main thread for potentially 100ms+
function processLargeArray(data) {
  data.forEach(item => heavyComputation(item)); // long task
}

// GOOD — yield to the event loop periodically
async function processLargeArrayAsync(data) {
  for (let i = 0; i < data.length; i++) {
    heavyComputation(data[i]);
    if (i % 100 === 0) {
      // Yield to the event loop every 100 items
      await scheduler.yield(); // or: await new Promise(r => setTimeout(r, 0))
    }
  }
}
```

---

### scheduler.postTask()

The modern `Scheduler API` (`scheduler.postTask()`) gives you fine-grained control over task priority:

```javascript
// Schedule a background, low-priority task
scheduler.postTask(() => {
  prefetchNextPageData();
}, { priority: 'background' });

// Schedule a user-visible, high-priority task
scheduler.postTask(() => {
  updateSearchResults();
}, { priority: 'user-visible' });

// Yield to higher-priority tasks mid-computation
async function processData() {
  for (const chunk of chunks) {
    await scheduler.yield(); // yield to user interaction if pending
    processChunk(chunk);
  }
}
```

The three priorities are `'user-blocking'` (highest), `'user-visible'` (default), and `'background'` (lowest).

> **Best Practice:** Any operation that takes more than 50ms should be broken up. Use `scheduler.yield()` in loops, Web Workers for CPU-intensive work, and `requestIdleCallback` for truly low-priority background work.

---

## 6.4 — Web Storage APIs

The browser provides several mechanisms for persisting data on the client. Choosing the right one depends on the data size, structure, access pattern, and security requirements.

---

### Cookies

Cookies are the oldest web storage mechanism, originally designed for session management. They are automatically sent with every HTTP request to the server (for the same origin), which distinguishes them from all other storage APIs.

**Creating a cookie:**

```javascript
// JavaScript (document.cookie — not recommended for auth tokens)
document.cookie = "theme=dark; path=/; max-age=31536000; SameSite=Lax";
```

**Cookie flags and their security meaning:**

- `HttpOnly`: The cookie cannot be accessed by JavaScript at all (`document.cookie` returns nothing for it). This is the single most important flag for auth tokens — it prevents XSS attacks from stealing tokens.
- `Secure`: The cookie is only sent over HTTPS connections.
- `SameSite=Strict`: The cookie is never sent in cross-site requests — maximum CSRF protection, but can cause issues with OAuth redirects.
- `SameSite=Lax`: The cookie is sent in top-level navigations (clicking a link to your site) but not in sub-resource requests (images, iframes, API calls from other origins). Good default.
- `SameSite=None; Secure`: Cookie sent in all contexts (third-party cookies). Requires Secure flag.
- `Domain`: Which domain the cookie belongs to (and subdomains if set).
- `Path`: Cookie is only sent for requests matching this path.
- `Expires` / `max-age`: Session cookie (no expiry) vs persistent cookie.

**Storage limits:** Typically 4KB per cookie, ~20–50 cookies per domain.

> **Best Practice:** Never store sensitive data like JWTs in `localStorage`. Use `HttpOnly` cookies set by your server for authentication tokens. JavaScript cannot read `HttpOnly` cookies, so even if an attacker injects malicious scripts, they cannot steal the token.

---

### localStorage and sessionStorage

Both are simple synchronous key-value stores that hold string data only.

```javascript
// localStorage — persists until explicitly cleared
localStorage.setItem('user-preference', JSON.stringify({ theme: 'dark', lang: 'en' }));
const prefs = JSON.parse(localStorage.getItem('user-preference') ?? '{}');
localStorage.removeItem('user-preference');
localStorage.clear(); // removes ALL items for this origin

// sessionStorage — cleared when the tab/window is closed
sessionStorage.setItem('draft', formData);
const draft = sessionStorage.getItem('draft');
```

**Key differences:**

| Feature | localStorage | sessionStorage |
|---|---|---|
| Persistence | Until cleared | Tab session only |
| Scope | All tabs on same origin | One tab only |
| Storage limit | ~5–10MB | ~5–10MB |
| Accessible from JS | Yes | Yes |
| Sent to server | No | No |

**Limitations:**
- Synchronous API — large reads/writes block the main thread
- Strings only — must JSON.stringify/parse objects
- Not available in Web Workers or Service Workers
- Not suitable for sensitive data (readable by any JS on the page)
- Not suitable for structured or relational data

> **Best Practice:** Use `localStorage` for user preferences (theme, language, UI settings). Don't use it for security-sensitive data or large datasets. Always wrap in `try/catch` because `localStorage` can throw if storage quota is exceeded or if the user is in private browsing mode in some browsers.

```javascript
function safeLocalStorage() {
  try {
    return window.localStorage;
  } catch {
    return null; // private browsing or quota exceeded
  }
}
```

---

### IndexedDB

IndexedDB is a full-featured, **asynchronous**, transactional, key-value object store that can hold large amounts of structured JavaScript data (including blobs, files, typed arrays).

It works through **object stores** (like tables) and supports **indexes** for efficient queries. Every operation is asynchronous and returns a request object.

```javascript
// Opening a database
const request = indexedDB.open('MyAppDB', 1);

request.onupgradeneeded = (event) => {
  const db = event.target.result;
  // Create an object store with auto-incrementing key
  const store = db.createObjectStore('products', { keyPath: 'id', autoIncrement: true });
  // Create an index for querying by name
  store.createIndex('name', 'name', { unique: false });
};

request.onsuccess = (event) => {
  const db = event.target.result;

  // Adding data
  const tx = db.transaction('products', 'readwrite');
  const store = tx.objectStore('products');
  store.add({ name: 'Widget', price: 9.99 });

  // Reading data
  const readTx = db.transaction('products', 'readonly');
  const readStore = readTx.objectStore('products');
  const getReq = readStore.get(1);
  getReq.onsuccess = () => console.log(getReq.result);
};
```

Because the raw IndexedDB API is verbose, libraries like **Dexie.js** or **idb** (by Jake Archibald) provide cleaner, Promise-based wrappers:

```javascript
import { openDB } from 'idb';

const db = await openDB('MyAppDB', 1, {
  upgrade(db) {
    db.createObjectStore('products', { keyPath: 'id' });
  },
});

// Write
await db.add('products', { id: 1, name: 'Widget', price: 9.99 });

// Read
const product = await db.get('products', 1);
console.log(product); // { id: 1, name: 'Widget', price: 9.99 }

// Query all
const all = await db.getAll('products');
```

**When to use IndexedDB:**
- Offline-first apps (storing data when the network is unavailable)
- Caching large datasets (product catalogues, news articles)
- Applications that need client-side search
- Progressive Web Apps that need rich offline capability

**Storage limits:** Up to ~80% of disk space (browser-dependent), significantly more than localStorage.

---

### Cache API (for Service Workers)

The Cache API is a storage mechanism specifically designed for caching Request/Response pairs. It is primarily used within Service Workers to implement offline strategies. It is covered in detail in section 6.9.

```javascript
// From within a Service Worker
const cache = await caches.open('my-cache-v1');
await cache.addAll(['/index.html', '/styles.css', '/app.js']);

// Retrieving a cached response
const cachedResponse = await caches.match('/index.html');
```

---

### Origin Private File System (OPFS)

OPFS is a modern, performance-optimised virtual file system that is private to the origin. Unlike the File System Access API (which requires user permission dialogs), OPFS is always available without permission prompts. It is ideal for storing large amounts of binary data like SQLite databases, WASM modules, or file-based applications.

```javascript
// Access the OPFS root directory
const root = await navigator.storage.getDirectory();

// Create/open a file
const fileHandle = await root.getFileHandle('my-data.bin', { create: true });
const writable = await fileHandle.createWritable();
await writable.write(binaryData);
await writable.close();
```

---

### Storage Quotas and Eviction

Browsers allocate a storage quota per origin. You can query how much is used and available:

```javascript
const estimate = await navigator.storage.estimate();
console.log(`Used: ${estimate.usage} bytes`);
console.log(`Available: ${estimate.quota} bytes`);
```

By default, browser storage is **best-effort** — the browser may evict it under storage pressure. To request **persistent** storage (the browser will ask the user for permission):

```javascript
const persisted = await navigator.storage.persist();
if (persisted) {
  console.log('Storage will not be evicted');
}
```

---

## 6.5 — Communication APIs

### Fetch API — Deep Dive

The Fetch API is the modern replacement for `XMLHttpRequest`. It returns Promises and supports streaming, making it vastly more capable.

```javascript
// Basic GET request
const response = await fetch('/api/users');
if (!response.ok) {
  throw new Error(`HTTP error! status: ${response.status}`);
}
const users = await response.json();

// POST with JSON body and custom headers
const response = await fetch('/api/users', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'Authorization': `Bearer ${token}`,
  },
  body: JSON.stringify({ name: 'Alice', email: 'alice@example.com' }),
});
```

**Request object and options:**

```javascript
const request = new Request('/api/data', {
  method: 'GET',
  headers: new Headers({ 'Accept': 'application/json' }),
  cache: 'no-cache',          // 'default' | 'no-store' | 'reload' | 'no-cache' | 'force-cache'
  credentials: 'same-origin', // 'omit' | 'same-origin' | 'include'
  mode: 'cors',               // 'cors' | 'no-cors' | 'same-origin'
  redirect: 'follow',         // 'follow' | 'manual' | 'error'
  signal: abortController.signal,
});
```

**Reading response bodies:**

```javascript
response.json()        // parses body as JSON
response.text()        // reads body as string
response.blob()        // reads body as Blob (for files/images)
response.arrayBuffer() // reads body as ArrayBuffer (binary data)
response.formData()    // reads body as FormData
```

**Streaming a response:**

```javascript
const response = await fetch('/api/large-dataset');
const reader = response.body.getReader();
const decoder = new TextDecoder();

while (true) {
  const { done, value } = await reader.read();
  if (done) break;
  console.log(decoder.decode(value)); // process chunk
}
```

---

### AbortController and AbortSignal

Every async operation in the browser can be cancelled using `AbortController`:

```javascript
const controller = new AbortController();
const { signal } = controller;

// Cancel after 5 seconds
const timeoutId = setTimeout(() => controller.abort(), 5000);

try {
  const response = await fetch('/api/slow-endpoint', { signal });
  const data = await response.json();
  clearTimeout(timeoutId);
  return data;
} catch (err) {
  if (err.name === 'AbortError') {
    console.log('Request was cancelled');
  } else {
    throw err;
  }
}

// Cancel manually (e.g., when a component is destroyed)
controller.abort();
```

In Angular, this is particularly important to cancel HTTP requests when a component is destroyed or when the user navigates away:

```typescript
// Angular service with cancellation
class SearchService {
  private controller: AbortController | null = null;

  async search(query: string) {
    // Cancel any pending search
    this.controller?.abort();
    this.controller = new AbortController();

    const response = await fetch(`/api/search?q=${query}`, {
      signal: this.controller.signal,
    });
    return response.json();
  }
}
```

---

### WebSockets

WebSockets provide a persistent, full-duplex communication channel between the browser and server over a single TCP connection. Unlike HTTP, the server can push messages to the client at any time.

**Lifecycle:**

1. Client sends an HTTP upgrade request (`Upgrade: websocket` header)
2. Server responds with `101 Switching Protocols`
3. The connection is now a WebSocket — both sides can send and receive frames at any time
4. Either side can close the connection

```javascript
const ws = new WebSocket('wss://example.com/chat'); // always use wss:// in production

// Connection opened
ws.addEventListener('open', () => {
  ws.send(JSON.stringify({ type: 'join', room: 'general' }));
});

// Message received from server
ws.addEventListener('message', (event) => {
  const data = JSON.parse(event.data);
  console.log('Received:', data);
});

// Connection closed
ws.addEventListener('close', (event) => {
  console.log(`Closed: code=${event.code}, reason=${event.reason}`);
  // Reconnect logic here
});

// Error
ws.addEventListener('error', (error) => {
  console.error('WebSocket error:', error);
});

// Sending data
ws.send('Hello server!');
ws.send(JSON.stringify({ type: 'message', text: 'Hello' }));
ws.send(binaryData); // also supports ArrayBuffer and Blob

// Closing
ws.close(1000, 'Normal closure');
```

**WebSocket readyState values:**

| Value | Constant | Meaning |
|---|---|---|
| 0 | `CONNECTING` | Connection not yet established |
| 1 | `OPEN` | Connection open, can send |
| 2 | `CLOSING` | Connection closing |
| 3 | `CLOSED` | Connection closed |

> **Best Practice:** Always implement reconnection logic with exponential backoff. WebSocket connections can be dropped by proxies, CDNs, or network interruptions.

---

### Server-Sent Events (SSE)

SSE is a simpler alternative to WebSockets for scenarios where you only need the **server to push data to the client** (one-way streaming). A great use case is live notifications, real-time dashboards, or AI response streaming.

The server sends a `text/event-stream` response with a persistent connection. Data is sent as plain text lines prefixed with `data:`.

```javascript
const eventSource = new EventSource('/api/notifications');

// Unnamed events ('message' by default)
eventSource.addEventListener('message', (event) => {
  console.log('New notification:', event.data);
});

// Named events
eventSource.addEventListener('user-joined', (event) => {
  const user = JSON.parse(event.data);
  addUserToList(user);
});

// Handle errors and reconnection
eventSource.addEventListener('error', () => {
  if (eventSource.readyState === EventSource.CLOSED) {
    console.log('Connection was closed');
  }
});

// Close the connection
eventSource.close();
```

**SSE vs WebSockets:**

| Feature | SSE | WebSockets |
|---|---|---|
| Direction | Server → Client only | Bidirectional |
| Protocol | HTTP | WebSocket (ws://) |
| Auto-reconnect | Built-in | Must implement manually |
| Browser support | All modern browsers | All modern browsers |
| HTTP/2 compatible | Yes (multiplexed) | Separate connection |
| Use case | Notifications, feeds, AI streaming | Chat, games, real-time collaboration |

---

### Broadcast Channel API

The Broadcast Channel API lets different browsing contexts (tabs, windows, iframes, workers) from the same origin communicate with each other:

```javascript
// In tab 1
const channel = new BroadcastChannel('app-state');
channel.postMessage({ type: 'USER_LOGGED_OUT' });

// In tab 2 (same origin)
const channel = new BroadcastChannel('app-state');
channel.addEventListener('message', (event) => {
  if (event.data.type === 'USER_LOGGED_OUT') {
    redirectToLogin(); // sync logout across tabs
  }
});
```

A practical use case in Angular: when the user logs out in one tab, broadcast the event so all other open tabs also redirect to the login page.

---

### SharedWorker

A `SharedWorker` is a Web Worker that can be shared by multiple browser contexts (tabs, windows, iframes) from the same origin. It's useful for shared state or connections (like a shared WebSocket connection).

```javascript
// In your application code (multiple tabs can connect to the same worker)
const worker = new SharedWorker('/shared-worker.js');
worker.port.onmessage = (event) => console.log(event.data);
worker.port.postMessage('Hello from tab');
```

---

## 6.6 — Security APIs

### Content Security Policy (CSP)

CSP is an HTTP response header that tells the browser which sources of content (scripts, styles, images, etc.) are considered trustworthy. It's one of the most effective defences against XSS attacks.

```http
Content-Security-Policy:
  default-src 'self';
  script-src 'self' https://trusted-cdn.com;
  style-src 'self' 'unsafe-inline';
  img-src 'self' data: https:;
  connect-src 'self' https://api.example.com;
  font-src 'self' https://fonts.gstatic.com;
  frame-ancestors 'none';
  base-uri 'self';
  form-action 'self';
```

**Key directives explained:**

- `default-src 'self'` — by default, only load resources from the same origin
- `script-src` — controls which scripts can be executed; `'unsafe-inline'` allows inline scripts (avoid if possible); `'nonce-{random}'` allows specific inline scripts by nonce
- `style-src` — controls styles; Angular's inline styles typically require `'unsafe-inline'` or nonces
- `frame-ancestors 'none'` — prevents the page from being embedded in any iframe (clickjacking protection)
- `report-uri` / `report-to` — send CSP violation reports to a URL for monitoring
- `Content-Security-Policy-Report-Only` — test CSP without enforcing it

**CSP and Angular:** Angular 16+ supports nonce-based CSP for inline styles. You can provide a nonce via the `ngCspNonce` attribute on the `<app-root>` or `<body>` element and configure it in the server response.

---

### Subresource Integrity (SRI)

SRI ensures that files fetched from CDNs haven't been tampered with. You include a cryptographic hash of the expected file content; if the file's hash doesn't match, the browser refuses to execute it.

```html
<script
  src="https://cdn.example.com/library.min.js"
  integrity="sha384-oqVuAfXRKap7fdgcCY5uykM6+R9GqQ8K/uxy9rx7HNQlGYl1kPzQho1wx4JwY8wC"
  crossorigin="anonymous">
</script>
```

You can generate the hash with: `openssl dgst -sha384 -binary library.min.js | openssl base64 -A`

---

### Cross-Origin Resource Policy (CORP)

CORP is a response header that controls whether a response can be included in a cross-origin context:

```http
Cross-Origin-Resource-Policy: same-origin   # Only same origin can read
Cross-Origin-Resource-Policy: same-site     # Only same site can read
Cross-Origin-Resource-Policy: cross-origin  # Anyone can read
```

---

### Cross-Origin Embedder Policy (COEP) and Cross-Origin Opener Policy (COOP)

These headers are required to enable powerful features like `SharedArrayBuffer` and high-resolution timers (for performance measurements, and for Spectre mitigations):

```http
Cross-Origin-Embedder-Policy: require-corp
Cross-Origin-Opener-Policy: same-origin
```

Together, these put the page into a "cross-origin isolated" context. You can check if you're in this context with:

```javascript
console.log(crossOriginIsolated); // true if COEP + COOP are set
```

---

### Permissions API

The Permissions API lets you query and observe the state of browser permissions without triggering the permission prompt:

```javascript
// Query permission state
const status = await navigator.permissions.query({ name: 'geolocation' });
console.log(status.state); // 'granted' | 'denied' | 'prompt'

// Listen for changes
status.onchange = () => {
  console.log('Permission state changed to:', status.state);
};

// Common permission names:
// 'geolocation', 'notifications', 'push', 'camera', 'microphone',
// 'clipboard-read', 'clipboard-write', 'persistent-storage'
```

---

### Credential Management API

The Credential Management API allows websites to store and retrieve credentials from the browser's credential store, supporting password managers, federated identity (Google Sign-In, etc.), and WebAuthn (passwordless).

```javascript
// Store a password credential after successful login
const cred = new PasswordCredential({ id: email, password: password });
await navigator.credentials.store(cred);

// Retrieve credentials on next visit (may show browser UI)
const credential = await navigator.credentials.get({
  password: true,
  federated: { providers: ['https://accounts.google.com'] },
  mediation: 'optional', // 'silent' | 'optional' | 'required'
});

if (credential) {
  // Auto-sign in
  loginWithCredential(credential);
}
```

---

## 6.7 — Performance APIs

### Navigation Timing API

The Navigation Timing API provides timestamps for key milestones in the page load lifecycle. All data is available on `performance.getEntriesByType('navigation')[0]`.

```javascript
const [nav] = performance.getEntriesByType('navigation');

console.log(`DNS lookup: ${nav.domainLookupEnd - nav.domainLookupStart}ms`);
console.log(`TCP connect: ${nav.connectEnd - nav.connectStart}ms`);
console.log(`Request/response: ${nav.responseEnd - nav.requestStart}ms`);
console.log(`DOM interactive: ${nav.domInteractive - nav.startTime}ms`);
console.log(`DOM complete: ${nav.domComplete - nav.startTime}ms`);
console.log(`Load event end: ${nav.loadEventEnd - nav.startTime}ms`);
```

---

### Resource Timing API

Provides detailed timing for every resource (images, scripts, stylesheets, API calls) loaded by the page:

```javascript
const resources = performance.getEntriesByType('resource');

resources.forEach(resource => {
  console.log(`${resource.name}: ${resource.duration.toFixed(2)}ms`);
  // Properties: initiatorType, transferSize, encodedBodySize, decodedBodySize
});

// Filter by type
const scripts = resources.filter(r => r.initiatorType === 'script');
const apiCalls = resources.filter(r => r.initiatorType === 'fetch');
```

---

### User Timing API

The User Timing API lets you add custom performance measurements to your code, which then appear in DevTools:

```javascript
// Mark a point in time
performance.mark('data-fetch-start');

const data = await fetchData();

performance.mark('data-fetch-end');

// Measure the duration between two marks
performance.measure('data-fetch-duration', 'data-fetch-start', 'data-fetch-end');

// Read the measurement
const [measure] = performance.getEntriesByName('data-fetch-duration');
console.log(`Data fetch took: ${measure.duration.toFixed(2)}ms`);

// Clean up
performance.clearMarks();
performance.clearMeasures();
```

---

### Long Tasks API

Detects tasks that take longer than 50ms (long tasks that block the main thread):

```javascript
const observer = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    console.warn(`Long task: ${entry.duration.toFixed(2)}ms`);
    console.log('Attribution:', entry.attribution);
  }
});

observer.observe({ entryTypes: ['longtask'] });
```

---

### PerformanceObserver — The Universal Observer

`PerformanceObserver` is the modern, reactive way to observe all performance entries:

```javascript
const observer = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    // entry.entryType tells you what it is
    console.log(entry);
  }
});

// Available entry types:
// 'navigation', 'resource', 'paint', 'largest-contentful-paint',
// 'first-input', 'layout-shift', 'longtask', 'mark', 'measure', 'element'

observer.observe({
  entryTypes: ['largest-contentful-paint', 'layout-shift', 'first-input'],
  buffered: true, // also capture entries that happened before the observer was created
});
```

---

### Core Web Vitals Measurement

The `web-vitals` library (by Google) provides the simplest way to measure Core Web Vitals:

```javascript
import { onLCP, onINP, onCLS, onFCP, onTTFB } from 'web-vitals';

// Each function calls your callback with the metric result
onLCP(metric => sendToAnalytics('LCP', metric.value));
onINP(metric => sendToAnalytics('INP', metric.value));
onCLS(metric => sendToAnalytics('CLS', metric.value));
onFCP(metric => sendToAnalytics('FCP', metric.value));
onTTFB(metric => sendToAnalytics('TTFB', metric.value));

function sendToAnalytics(name, value) {
  navigator.sendBeacon('/analytics', JSON.stringify({ name, value }));
}
```

**Core Web Vitals thresholds:**

| Metric | Good | Needs Improvement | Poor |
|---|---|---|---|
| LCP (Largest Contentful Paint) | ≤ 2.5s | ≤ 4.0s | > 4.0s |
| INP (Interaction to Next Paint) | ≤ 200ms | ≤ 500ms | > 500ms |
| CLS (Cumulative Layout Shift) | ≤ 0.1 | ≤ 0.25 | > 0.25 |

---

## 6.8 — Intersection & Visibility

### Intersection Observer API

The Intersection Observer API lets you observe when an element enters or exits the viewport (or another element), without needing scroll event listeners. It's far more performant than the traditional scroll-based approach because it runs off the main thread.

**Core use cases:** Lazy loading images, triggering animations when elements scroll into view, infinite scroll, analytics (tracking which content was seen).

```javascript
// Basic setup
const observer = new IntersectionObserver((entries) => {
  entries.forEach(entry => {
    if (entry.isIntersecting) {
      // Element is now visible
      console.log(`${entry.target.id} is visible`);
      console.log(`Intersection ratio: ${entry.intersectionRatio}`); // 0 to 1
    }
  });
}, {
  root: null,           // null = viewport
  rootMargin: '0px',    // expand/contract the root's bounding box
  threshold: 0.5,       // callback fires when 50% of the element is visible
});

// Observe elements
observer.observe(document.querySelector('#section-1'));
observer.observe(document.querySelector('#section-2'));

// Stop observing
observer.unobserve(element);
observer.disconnect(); // stop all observations
```

**Lazy loading images:**

```html
<img data-src="heavy-image.jpg" alt="..." class="lazy">
```

```javascript
const imageObserver = new IntersectionObserver((entries) => {
  entries.forEach(entry => {
    if (entry.isIntersecting) {
      const img = entry.target;
      img.src = img.dataset.src; // swap data-src → src to trigger load
      img.classList.remove('lazy');
      imageObserver.unobserve(img); // stop observing once loaded
    }
  });
}, { rootMargin: '200px' }); // load when image is 200px away from viewport

document.querySelectorAll('img.lazy').forEach(img => imageObserver.observe(img));
```

**rootMargin** is particularly powerful — setting it to `'200px'` means the callback fires when the element is 200px *before* it enters the viewport, giving you time to start loading before the user sees the element.

**Multiple thresholds:**

```javascript
const observer = new IntersectionObserver((entries) => {
  entries.forEach(entry => {
    // entry.intersectionRatio tells you exactly how much is visible
    element.style.opacity = entry.intersectionRatio;
  });
}, {
  threshold: [0, 0.1, 0.2, 0.3, 0.4, 0.5, 0.6, 0.7, 0.8, 0.9, 1.0],
  // callback fires at each 10% increment
});
```

---

### Page Visibility API

The Page Visibility API tells you whether the user can currently see the page — the tab is in the foreground, or the user has minimised the window or switched to another tab.

```javascript
// Check current visibility
console.log(document.visibilityState); // 'visible' | 'hidden' | 'prerender'
console.log(document.hidden); // boolean shorthand

// Listen for changes
document.addEventListener('visibilitychange', () => {
  if (document.hidden) {
    // Page is hidden — pause video, stop animations, cancel polling
    videoElement.pause();
    clearInterval(pollingInterval);
    console.log('User left the page');
  } else {
    // Page is visible again — resume
    videoElement.play();
    pollingInterval = startPolling();
    console.log('User returned to the page');
  }
});
```

**Practical applications:**
- Pause video/audio when tab is hidden (saves CPU and battery)
- Stop expensive polling or WebSocket subscriptions
- Track engagement time (only count time when page is visible)
- Re-fetch data when user returns to the tab after a long absence

---

### Page Lifecycle API

The Page Lifecycle API extends Page Visibility with additional states for mobile-first browsers that may freeze or discard background tabs to save memory:

```javascript
// States: active → passive → hidden → frozen → discarded → terminated

document.addEventListener('freeze', () => {
  // Page is being frozen (background tab, low memory)
  // Save any important state — you may not get another chance
  saveStateToStorage();
});

document.addEventListener('resume', () => {
  // Page was frozen and is now resuming
  restoreStateFromStorage();
});

// The 'pagehide' event fires before the page might be discarded
window.addEventListener('pagehide', (event) => {
  if (event.persisted) {
    // Page is going into the Back/Forward Cache (bfcache)
    // Close WebSocket connections (bfcache doesn't allow active connections)
    wsConnection.close();
  }
});

window.addEventListener('pageshow', (event) => {
  if (event.persisted) {
    // Page was restored from bfcache
    wsConnection = reconnect();
  }
});
```

> **Best Practice:** Always handle `pagehide` to close WebSocket connections and flush analytics data. Pages with open WebSocket connections are ineligible for the Back/Forward Cache (bfcache), which would cause slower back-button navigation for your users.

---

## 6.9 — Offline & Background APIs

### Service Worker Lifecycle

A Service Worker is a JavaScript file that runs in a background thread, separate from your main page. It acts as a programmable proxy between your web app and the network. It is the foundation of PWA capabilities: offline support, background sync, push notifications, and more.

**The lifecycle has three phases:**

**1. Registration:**

```javascript
// main.js (your app code)
if ('serviceWorker' in navigator) {
  window.addEventListener('load', async () => {
    try {
      const registration = await navigator.serviceWorker.register('/sw.js', {
        scope: '/', // URL scope the SW controls
      });
      console.log('SW registered:', registration);
    } catch (error) {
      console.error('SW registration failed:', error);
    }
  });
}
```

**2. Installation:**

When the SW is first registered (or a new version is deployed), the browser installs it. This is where you precache assets:

```javascript
// sw.js
const CACHE_NAME = 'my-app-v1';
const PRECACHE_URLS = ['/index.html', '/styles.css', '/app.js', '/icon.png'];

self.addEventListener('install', (event) => {
  event.waitUntil(
    caches.open(CACHE_NAME)
      .then(cache => cache.addAll(PRECACHE_URLS))
      .then(() => self.skipWaiting()) // activate immediately
  );
});
```

**3. Activation:**

After installation, the SW waits until all pages using the old SW are closed. Then it activates. This is where you clean up old caches:

```javascript
self.addEventListener('activate', (event) => {
  event.waitUntil(
    caches.keys().then(cacheNames =>
      Promise.all(
        cacheNames
          .filter(name => name !== CACHE_NAME) // old caches
          .map(name => caches.delete(name))
      )
    ).then(() => self.clients.claim()) // take control immediately
  );
});
```

---

### Cache Strategies

The `fetch` event intercepts every network request made by pages the SW controls. You can implement different caching strategies:

**Cache First (good for static assets):**

```javascript
self.addEventListener('fetch', (event) => {
  event.respondWith(
    caches.match(event.request)
      .then(cached => cached ?? fetch(event.request))
  );
});
```

**Network First (good for API calls):**

```javascript
self.addEventListener('fetch', (event) => {
  event.respondWith(
    fetch(event.request)
      .then(response => {
        const clone = response.clone();
        caches.open(CACHE_NAME).then(cache => cache.put(event.request, clone));
        return response;
      })
      .catch(() => caches.match(event.request)) // fallback to cache on error
  );
});
```

**Stale-While-Revalidate (best of both worlds):**

```javascript
self.addEventListener('fetch', (event) => {
  event.respondWith(
    caches.open(CACHE_NAME).then(async cache => {
      const cached = await cache.match(event.request);

      // Fetch in background to update cache
      const fetchPromise = fetch(event.request).then(response => {
        cache.put(event.request, response.clone());
        return response;
      });

      // Return cached version immediately, update cache in background
      return cached ?? fetchPromise;
    })
  );
});
```

---

### Workbox (v7+)

Workbox is Google's library for Service Workers that provides production-ready implementations of all common caching strategies:

```javascript
// sw.js with Workbox
import { precacheAndRoute } from 'workbox-precaching';
import { registerRoute } from 'workbox-routing';
import { StaleWhileRevalidate, CacheFirst, NetworkFirst } from 'workbox-strategies';
import { ExpirationPlugin } from 'workbox-expiration';

// Precache files (list generated by build tool)
precacheAndRoute(self.__WB_MANIFEST);

// API calls — Network First, fall back to cache
registerRoute(
  ({ url }) => url.pathname.startsWith('/api/'),
  new NetworkFirst({ cacheName: 'api-cache' })
);

// Images — Cache First with expiration
registerRoute(
  ({ request }) => request.destination === 'image',
  new CacheFirst({
    cacheName: 'image-cache',
    plugins: [
      new ExpirationPlugin({ maxEntries: 100, maxAgeSeconds: 30 * 24 * 60 * 60 }),
    ],
  })
);

// Static assets — Stale-While-Revalidate
registerRoute(
  ({ request }) => ['style', 'script', 'worker'].includes(request.destination),
  new StaleWhileRevalidate({ cacheName: 'assets-cache' })
);
```

---

### Background Sync API

Background Sync lets you defer actions until the user has reliable connectivity. If the user submits a form while offline, Background Sync will retry the submission when the connection is restored — even if they've closed the app:

```javascript
// In your app
async function sendMessage(data) {
  try {
    await fetch('/api/messages', { method: 'POST', body: JSON.stringify(data) });
  } catch {
    // Save to IndexedDB and register a sync
    await saveToOutbox(data);
    const registration = await navigator.serviceWorker.ready;
    await registration.sync.register('send-messages');
  }
}

// In service worker
self.addEventListener('sync', (event) => {
  if (event.tag === 'send-messages') {
    event.waitUntil(
      getOutbox().then(messages =>
        Promise.all(messages.map(msg =>
          fetch('/api/messages', { method: 'POST', body: JSON.stringify(msg) })
            .then(() => removeFromOutbox(msg.id))
        ))
      )
    );
  }
});
```

---

### Push API and Web Notifications

Push notifications allow servers to send messages to users even when their browser isn't open (as long as the SW is registered and the user granted permission):

```javascript
// 1. Request notification permission
const permission = await Notification.requestPermission();

// 2. Subscribe to push (requires VAPID keys from your server)
const registration = await navigator.serviceWorker.ready;
const subscription = await registration.pushManager.subscribe({
  userVisibleOnly: true,
  applicationServerKey: urlBase64ToUint8Array(PUBLIC_VAPID_KEY),
});

// 3. Send subscription to your server
await fetch('/api/push-subscriptions', {
  method: 'POST',
  body: JSON.stringify(subscription),
  headers: { 'Content-Type': 'application/json' },
});

// In service worker — handle incoming push
self.addEventListener('push', (event) => {
  const data = event.data.json();
  event.waitUntil(
    self.registration.showNotification(data.title, {
      body: data.body,
      icon: '/icon-192.png',
      badge: '/badge-72.png',
      data: { url: data.url },
      actions: [
        { action: 'view', title: 'View' },
        { action: 'dismiss', title: 'Dismiss' },
      ],
    })
  );
});

// Handle notification click
self.addEventListener('notificationclick', (event) => {
  event.notification.close();
  if (event.action === 'view') {
    event.waitUntil(clients.openWindow(event.notification.data.url));
  }
});
```

---

### Periodic Background Sync

Periodic Background Sync (available in Chromium-based browsers for installed PWAs) lets you schedule periodic background data syncs, like a native app:

```javascript
const registration = await navigator.serviceWorker.ready;
await registration.periodicSync.register('news-update', {
  minInterval: 24 * 60 * 60 * 1000, // once per day at minimum
});

// In service worker
self.addEventListener('periodicsync', (event) => {
  if (event.tag === 'news-update') {
    event.waitUntil(updateNewsCache());
  }
});
```

---

## 6.10 — Device & Hardware APIs

### Geolocation API

```javascript
// One-time position
navigator.geolocation.getCurrentPosition(
  (position) => {
    console.log(`Lat: ${position.coords.latitude}`);
    console.log(`Lon: ${position.coords.longitude}`);
    console.log(`Accuracy: ${position.coords.accuracy}m`);
  },
  (error) => {
    // error.code: 1=PERMISSION_DENIED, 2=POSITION_UNAVAILABLE, 3=TIMEOUT
    console.error(error.message);
  },
  {
    enableHighAccuracy: true, // uses GPS (slower, higher battery)
    timeout: 5000,
    maximumAge: 0, // don't use cached position
  }
);

// Watch position (continuous updates)
const watchId = navigator.geolocation.watchPosition(onSuccess, onError, options);
navigator.geolocation.clearWatch(watchId); // stop watching
```

---

### Media Devices API (Camera & Microphone)

```javascript
// Request camera and microphone access
try {
  const stream = await navigator.mediaDevices.getUserMedia({
    video: { width: 1280, height: 720, facingMode: 'user' },
    audio: { echoCancellation: true, noiseSuppression: true },
  });

  // Attach to a <video> element
  const video = document.querySelector('video');
  video.srcObject = stream;
  await video.play();

  // Stop the stream when done (releases camera/mic)
  stream.getTracks().forEach(track => track.stop());
} catch (err) {
  if (err.name === 'NotAllowedError') {
    console.error('User denied permission');
  }
}

// Enumerate available devices
const devices = await navigator.mediaDevices.enumerateDevices();
const cameras = devices.filter(d => d.kind === 'videoinput');
```

---

### Screen Capture API

```javascript
// Capture the screen, a window, or a tab
try {
  const stream = await navigator.mediaDevices.getDisplayMedia({
    video: { cursor: 'always' },
    audio: false,
  });

  const video = document.querySelector('video');
  video.srcObject = stream;
} catch (err) {
  console.log('Screen capture cancelled or denied');
}
```

---

### Device Orientation and Motion

```javascript
// Orientation (compass/gyroscope data)
window.addEventListener('deviceorientation', (event) => {
  console.log(`Alpha: ${event.alpha}°`); // rotation around z-axis (compass heading)
  console.log(`Beta: ${event.beta}°`);   // rotation around x-axis (front-back tilt)
  console.log(`Gamma: ${event.gamma}°`); // rotation around y-axis (left-right tilt)
});

// Motion (accelerometer data)
window.addEventListener('devicemotion', (event) => {
  console.log(event.acceleration);          // without gravity
  console.log(event.accelerationIncludingGravity);
  console.log(event.rotationRate);
});
```

**Important:** On iOS 13+, accessing DeviceOrientation and DeviceMotion requires explicit user permission:

```javascript
if (typeof DeviceOrientationEvent.requestPermission === 'function') {
  const permission = await DeviceOrientationEvent.requestPermission();
  if (permission === 'granted') {
    window.addEventListener('deviceorientation', handler);
  }
}
```

---

### Vibration API

```javascript
navigator.vibrate(200);                    // vibrate for 200ms
navigator.vibrate([100, 50, 100, 50, 200]); // pattern: vibrate, pause, vibrate, pause, vibrate
navigator.vibrate(0);                      // stop vibrating
```

---

### Screen Orientation API

```javascript
// Read current orientation
console.log(screen.orientation.type); // 'landscape-primary' | 'portrait-primary' | etc.
console.log(screen.orientation.angle); // 0, 90, 180, 270

// Lock orientation (requires fullscreen in most browsers)
await screen.orientation.lock('landscape'); // or 'portrait', 'natural'
screen.orientation.unlock();

// Listen for changes
screen.orientation.addEventListener('change', () => {
  console.log('Orientation changed to:', screen.orientation.type);
});
```

---

## 6.11 — Modern Web Platform APIs (2024–2026)

### View Transitions API

The View Transitions API makes it possible to animate changes in the DOM — both single-page transitions (elements changing) and page-to-page transitions — using smooth, GPU-accelerated CSS animations. It requires no animation library.

**How it works:** You call `document.startViewTransition()` with a callback that performs the DOM change. The browser automatically captures a screenshot of the old state, performs the update, and then crossfades between the two states using CSS.

```javascript
// Without View Transitions: instant update
element.textContent = 'New Content';

// With View Transitions: smooth crossfade
document.startViewTransition(() => {
  element.textContent = 'New Content';
});
```

**Named element transitions** — animate specific elements between states by giving them a unique `view-transition-name`:

```css
.product-card.hero {
  view-transition-name: product-hero;
}
```

```javascript
document.startViewTransition(() => {
  // Move the element or change its state
  updateDOMForDetailView();
});
```

The browser will automatically animate the element with the `view-transition-name` from its old position/size to its new position/size — a "hero animation" like you see in native mobile apps, with zero manual animation code.

**In Angular Router** (with `withViewTransitions()`):

```typescript
// app.config.ts
export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(routes, withViewTransitions()),
  ],
};
```

Every route navigation now automatically crossfades. You can customise the animations with CSS:

```css
/* Customise the transition animation */
::view-transition-old(root) {
  animation: fade-out 0.3s ease;
}
::view-transition-new(root) {
  animation: fade-in 0.3s ease;
}
```

---

### Navigation API

The Navigation API is a modern replacement for the History API (`pushState`, `popstate`). It provides a single, consistent API for intercepting navigations, managing history, and tracking the current URL — including navigations triggered by the user clicking the back/forward buttons.

```javascript
// Intercept ALL navigations (links, back/forward, programmatic)
navigation.addEventListener('navigate', (event) => {
  // event.destination.url tells you where you're going
  if (!event.destination.url.startsWith(location.origin)) return; // let external links pass

  event.intercept({
    async handler() {
      // Perform the navigation yourself (e.g., update the DOM)
      const response = await fetch(event.destination.url);
      const html = await response.text();
      document.body.innerHTML = html; // simplified SPA routing
    },
  });
});

// Programmatic navigation
await navigation.navigate('/new-page');
await navigation.back();
await navigation.forward();
await navigation.reload();

// Access current history entry
console.log(navigation.currentEntry.url);
console.log(navigation.currentEntry.key); // unique identifier
```

---

### Popover API

The Popover API provides a native browser mechanism for creating popups, tooltips, dropdowns, and dialogs without any JavaScript library. Elements with the `popover` attribute gain special browser-managed behavior including top-layer rendering, light-dismiss (click outside to close), and focus management.

```html
<!-- Declare a popover -->
<button popovertarget="my-popup">Toggle Popup</button>
<div id="my-popup" popover>
  <p>I am a native popover!</p>
</div>
```

That's all you need for a fully functional popover that: opens on button click, closes on click outside (light dismiss), closes on Escape key, renders above all other content (top layer), and manages focus correctly.

```javascript
// Also controllable via JavaScript
const popover = document.getElementById('my-popup');
popover.showPopover();
popover.hidePopover();
popover.togglePopover();

// Types:
// popover="auto" (default) — light-dismiss, closes others
// popover="manual" — must close programmatically, no light-dismiss
```

---

### Speculation Rules API

The Speculation Rules API lets you declaratively tell the browser to prefetch or prerender pages the user is likely to navigate to next:

```html
<script type="speculationrules">
{
  "prerender": [
    { "urls": ["/product-detail", "/checkout"] }
  ],
  "prefetch": [
    { "where": { "href_matches": "/blog/*" }, "eagerness": "moderate" }
  ]
}
</script>
```

**Prerender** goes further than prefetch — the browser actually loads and *renders* the page in a hidden background tab. Navigation to a prerendered page is essentially instant. This is significantly more powerful than the old `<link rel="prerender">` hint.

**Eagerness levels:**
- `immediate` — speculate right away
- `eager` — speculate as soon as the resource is determined
- `moderate` — prefetch when hovered for ~200ms
- `conservative` — only on mousedown/touchstart

---

### Scheduler API

Covered in section 6.3, but worth reiterating here as a key modern API. `scheduler.postTask()` and `scheduler.yield()` give you priority-based task scheduling:

```javascript
// Post a task with a specific priority
await scheduler.postTask(doWork, { priority: 'user-blocking' });

// Yield control back to the browser (run pending user interactions, etc.)
async function processInChunks(data) {
  for (let i = 0; i < data.length; i += 100) {
    processChunk(data.slice(i, i + 100));
    await scheduler.yield(); // yield after every 100 items
  }
}
```

---

### Compression Streams API

Compress and decompress data natively in the browser with no libraries:

```javascript
// Compress data with gzip
async function compressData(data) {
  const stream = new CompressionStream('gzip');
  const writer = stream.writable.getWriter();
  writer.write(new TextEncoder().encode(data));
  writer.close();

  const chunks = [];
  const reader = stream.readable.getReader();
  while (true) {
    const { done, value } = await reader.read();
    if (done) break;
    chunks.push(value);
  }
  return new Blob(chunks);
}

// Decompress
async function decompressData(compressedBlob) {
  const stream = new DecompressionStream('gzip');
  const writer = stream.writable.getWriter();
  writer.write(await compressedBlob.arrayBuffer());
  writer.close();

  const output = await new Response(stream.readable).text();
  return output;
}
```

---

### File System Access API

The File System Access API lets users grant your web app permission to read and write files on their local file system — enabling true "save file" experiences without downloads:

```javascript
// Open a file
const [fileHandle] = await window.showOpenFilePicker({
  types: [{ description: 'Text files', accept: { 'text/plain': ['.txt'] } }],
});
const file = await fileHandle.getFile();
const contents = await file.text();

// Save a file
const fileHandle = await window.showSaveFilePicker({
  suggestedName: 'document.txt',
  types: [{ description: 'Text files', accept: { 'text/plain': ['.txt'] } }],
});
const writable = await fileHandle.createWritable();
await writable.write('Hello, world!');
await writable.close();

// Open a directory
const dirHandle = await window.showDirectoryPicker();
for await (const [name, handle] of dirHandle.entries()) {
  console.log(name, handle.kind); // 'file' or 'directory'
}
```

---

### Web Locks API

The Web Locks API coordinates access to shared resources across multiple tabs, windows, or workers — like a mutex for the web:

```javascript
// Acquire an exclusive lock (other code with this lock name must wait)
await navigator.locks.request('sync-operation', async (lock) => {
  // Only one tab can be here at a time
  await syncDataWithServer();
  // Lock is automatically released when the async function returns
});

// Shared lock (multiple readers, exclusive for writers)
await navigator.locks.request('data-cache', { mode: 'shared' }, async () => {
  return readFromCache(); // multiple tabs can read at the same time
});

// Query lock state
const state = await navigator.locks.query();
console.log(state.held);    // currently held locks
console.log(state.pending); // locks waiting to be acquired
```

---

### Import Maps

Import maps allow you to control how the browser resolves module specifiers, enabling you to use bare module names (like in Node.js) in the browser without a bundler:

```html
<script type="importmap">
{
  "imports": {
    "lodash": "https://cdn.skypack.dev/lodash",
    "lodash/": "https://cdn.skypack.dev/lodash/",
    "@utils/": "/src/utils/"
  }
}
</script>

<script type="module">
  import _ from 'lodash';            // resolves to cdn.skypack.dev
  import { formatDate } from '@utils/dates.js'; // resolves to /src/utils/dates.js
</script>
```

---

### Cookie Store API

The Cookie Store API provides an asynchronous, Promise-based replacement for the synchronous and awkward `document.cookie` string interface:

```javascript
// Get a specific cookie
const sessionCookie = await cookieStore.get('session');
console.log(sessionCookie?.value);

// Get all cookies
const allCookies = await cookieStore.getAll();

// Set a cookie
await cookieStore.set({
  name: 'theme',
  value: 'dark',
  maxAge: 365 * 24 * 60 * 60, // 1 year in seconds
  sameSite: 'lax',
});

// Delete a cookie
await cookieStore.delete('theme');

// Watch for changes
cookieStore.addEventListener('change', (event) => {
  console.log('Changed cookies:', event.changed);
  console.log('Deleted cookies:', event.deleted);
});
```

---

### Temporal API (Stage 3)

The `Temporal` API is the long-awaited replacement for the deeply problematic `Date` object. It provides immutable date/time objects with support for timezones, calendars, and arithmetic:

```javascript
// Current date and time (Temporal is not yet in all browsers; use polyfill)
const now = Temporal.Now.zonedDateTimeISO('America/New_York');
console.log(now.toString()); // 2026-03-10T14:30:00-05:00[America/New_York]

// Plain date (no time, no timezone)
const birthday = Temporal.PlainDate.from('1990-05-15');
const today = Temporal.Now.plainDateISO();
const age = today.since(birthday).years;

// Duration arithmetic
const meetingStart = Temporal.PlainDateTime.from('2026-03-10T14:00:00');
const duration = Temporal.Duration.from({ hours: 1, minutes: 30 });
const meetingEnd = meetingStart.add(duration);

// Timezone-aware comparison
const londonNoon = Temporal.ZonedDateTime.from('2026-03-10T12:00:00[Europe/London]');
const nyNoon = Temporal.ZonedDateTime.from('2026-03-10T12:00:00[America/New_York]');
const result = Temporal.ZonedDateTime.compare(londonNoon, nyNoon);
// -1 = London noon comes before NY noon (as it should — 5 hours earlier)
```

> **Best Practice:** Stop using `new Date()` for anything beyond the simplest cases. Adopt `Temporal` (via polyfill today, natively when broadly supported) for all date/time work in your Angular applications. Its correctness around timezones and DST alone makes it worth the switch.

---

## Summary & Key Takeaways

| Topic | The Single Most Important Thing to Remember |
|---|---|
| Rendering Pipeline | `transform` and `opacity` changes skip Layout and Paint — always prefer them for animations |
| JS Engine | Keep objects monomorphic (same shape, same property order) so V8 can apply inline caching |
| Event Loop | Microtasks drain completely after every macrotask — Promises resolve before the next `setTimeout` |
| Web Storage | Never store auth tokens in `localStorage`; use `HttpOnly` server-set cookies instead |
| Communication APIs | SSE for server→client streaming; WebSockets for bidirectional real-time; Broadcast Channel for cross-tab sync |
| Security APIs | CSP is your best XSS mitigation layer after Angular's built-in template sanitisation |
| Performance APIs | `PerformanceObserver` with `buffered: true` is the correct way to measure Web Vitals in production |
| Intersection Observer | Replaces all scroll-event-based visibility detection; use `rootMargin` to pre-load before viewport |
| Page Visibility | Always pause expensive work (polling, animations, video) when `document.hidden === true` |
| Service Workers | Understand the 3-phase lifecycle (install → activate → fetch) and choose caching strategies intentionally |
| Device APIs | `getUserMedia` requires a try/catch and graceful degradation when permission is denied |
| View Transitions | `withViewTransitions()` in Angular Router is the single easiest performance/UX win in modern Angular |
| Speculation Rules | Prerender critical next pages for near-instant navigation with no JavaScript required |

---

*Phase 6 Complete — Next: Phase 7 — Version Control & Developer Tooling*

*Document version: March 2026 | Covers Angular 21 · TypeScript 5.9 · Modern Web Platform APIs*

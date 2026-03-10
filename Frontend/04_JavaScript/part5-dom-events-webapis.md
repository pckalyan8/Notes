# ⚡ Phase 4 — Part 5: DOM APIs · Events · Web Platform APIs
> Sections 4.13 · 4.14 · 4.15 | Angular Frontend Mastery Roadmap

---

## 4.13 JavaScript DOM APIs

The Document Object Model (DOM) is the browser's in-memory, tree-shaped representation of an HTML document. Every HTML element becomes a *node* in this tree — and JavaScript has a rich API for reading and manipulating that tree. Angular abstracts away most direct DOM manipulation, but you must understand these fundamentals to debug Angular applications, write custom directives, and understand what Angular does on your behalf.

### Selecting Elements

Before you can do anything with an element, you need a reference to it.

`getElementById` is the fastest way to select a single element by its `id` attribute. It returns the element or `null` if not found.

`querySelector` accepts any valid CSS selector and returns the **first** matching element — or `null`. It is the most flexible single-element selector.

`querySelectorAll` returns a **static `NodeList`** of all matching elements. "Static" means it does not update if the DOM changes after the call. Note that `NodeList` is array-like but not a real array — use `Array.from()` or the spread operator to gain access to array methods.

```js
// Select by ID
const header = document.getElementById("main-header"); // returns Element or null

// Select by CSS selector
const firstButton = document.querySelector("button");
const submitBtn = document.querySelector("#form .submit-btn");
const activeItem = document.querySelector("[data-active='true']");

// Select all matching elements
const allButtons = document.querySelectorAll("button"); // NodeList
const listItems = document.querySelectorAll(".nav ul > li");

// Convert NodeList to real Array for array methods
const buttonArray = Array.from(allButtons);
const clickedButtons = buttonArray.filter(btn => btn.classList.contains("clicked"));
```

### Creating, Appending, and Removing Elements

You can dynamically build and modify the DOM tree. Angular does this on your behalf via its rendering engine, but custom directives and `ViewContainerRef` use the same underlying DOM operations.

```js
// Create a new element
const card = document.createElement("div");
card.className = "card";
card.id = "product-42";

// Create and append a text node
const title = document.createElement("h2");
title.textContent = "Product Name"; // safe — no XSS risk
card.appendChild(title);

// Insert HTML (carefully — XSS risk with user-supplied content)
card.innerHTML = `<p class="description">Safe static HTML content</p>`;
// NEVER do: card.innerHTML = userProvidedString; // XSS vulnerability!

// Append to the document
document.body.appendChild(card);
document.querySelector(".container").prepend(card); // add as first child
document.querySelector(".container").append(card, anotherCard); // append multiple

// Insert relative to an existing element
const reference = document.querySelector(".existing");
reference.insertAdjacentElement("beforebegin", card); // before it
reference.insertAdjacentElement("afterend", card);    // after it
reference.insertAdjacentElement("afterbegin", card);  // as first child
reference.insertAdjacentElement("beforeend", card);   // as last child

// Remove elements
card.remove();                   // modern — removes itself
parent.removeChild(card);       // legacy syntax
```

### `innerHTML`, `textContent`, and `innerText`

These three properties all deal with an element's content, but they behave very differently.

`textContent` gets or sets the raw text content of an element and all its descendants. It is the safest option — it treats everything as plain text, so it cannot cause XSS vulnerabilities. Use this when you just need to set text.

`innerHTML` gets or sets the HTML markup inside an element. When setting, the browser parses the string as HTML. This is powerful but dangerous when the string contains user-provided content, because any `<script>` or event handler attribute would be executed.

`innerText` is similar to `textContent` but is layout-aware — it respects CSS `display: none` and line breaks as they appear on screen. It triggers a layout reflow to compute the visible text, making it slower than `textContent`.

```js
const el = document.querySelector(".message");

// Reading
el.textContent; // all text, ignoring HTML tags
el.innerHTML;   // the full HTML markup including tags
el.innerText;   // visible text, respecting CSS visibility

// Writing
el.textContent = "Hello <b>World</b>"; // renders literally: Hello <b>World</b>
el.innerHTML = "Hello <b>World</b>";   // renders as: Hello World (with bold)
el.innerHTML = userInput;              // DANGER if userInput is untrusted!
```

> **Angular note:** Angular's template binding (`{{ expression }}` and `[textContent]`) always uses text-safe output — it never injects raw HTML. If you need HTML output, you must explicitly use `[innerHTML]` and sanitize the content through `DomSanitizer`. This is intentional protection against XSS.

### Attribute Manipulation

HTML attributes appear in the source markup. You can read and modify them with the Attribute API, but for boolean attributes and custom data attributes, there are better approaches.

```js
const input = document.querySelector("input");

// Generic attribute methods
input.getAttribute("placeholder");         // "Enter text..."
input.setAttribute("placeholder", "Type here...");
input.removeAttribute("disabled");
input.hasAttribute("required");           // true or false

// Direct property access (preferred for standard attributes)
input.value = "Hello";          // current value (not the attribute)
input.disabled = true;
input.type = "email";

// Custom data attributes — accessed via the dataset property
// HTML: <div data-user-id="42" data-role="admin">
const div = document.querySelector(".user-card");
div.dataset.userId; // "42" — always a string
div.dataset.role;   // "admin"
div.dataset.newProp = "value"; // sets data-new-prop attribute
```

### ClassList API

The `classList` property provides a clean API for working with an element's CSS classes, replacing the older and messier `className` string manipulation.

```js
const el = document.querySelector(".card");

el.classList.add("active");           // add a class
el.classList.remove("hidden");        // remove a class
el.classList.toggle("selected");      // add if absent, remove if present
el.classList.toggle("featured", true); // force add (second arg)
el.classList.contains("active");      // returns boolean
el.classList.replace("old", "new");   // replace one class with another

// Angular uses this API internally in class bindings:
// [class.active]="isActive" → el.classList.toggle("active", isActive)
```

### DOM Traversal

You can navigate between related nodes in the DOM tree.

```js
const item = document.querySelector(".list-item");

// Parent
item.parentElement;           // the immediate parent Element
item.parentNode;              // the parent Node (could be a document fragment)
item.closest(".container");   // nearest ancestor matching a CSS selector

// Children
item.children;                // HTMLCollection of child Elements
item.firstElementChild;       // first child Element
item.lastElementChild;        // last child Element
item.childElementCount;       // number of child elements

// Siblings
item.nextElementSibling;      // next sibling Element
item.previousElementSibling;  // previous sibling Element
```

### DocumentFragment for Batched DOM Updates

Every time you modify the DOM, the browser may need to recalculate layout and repaint. If you are creating many elements in a loop, inserting them one at a time causes many reflows. A `DocumentFragment` is an in-memory mini-document — you build your elements inside it, then insert the fragment into the live DOM in one operation, causing only a single reflow.

```js
const list = document.querySelector("ul");
const fragment = document.createDocumentFragment();

// Build all items in memory — no reflows yet
for (const item of itemsArray) {
  const li = document.createElement("li");
  li.textContent = item.name;
  fragment.appendChild(li); // appended to the fragment, not the DOM
}

// Single DOM mutation — only one reflow
list.appendChild(fragment);
```

---

## 4.14 JavaScript Events

Events are how the browser communicates that something has happened — a user click, a key press, a network response, a page load. JavaScript's event system is the foundation of all interactive web applications.

### `addEventListener` and `removeEventListener`

The modern way to attach events is via `addEventListener`. It allows multiple handlers on the same element for the same event type, unlike the older `onclick = fn` property approach which only allows one.

```js
const button = document.querySelector("button");

function handleClick(event) {
  console.log("Clicked!");
}

// Attach the handler
button.addEventListener("click", handleClick);

// Detach the handler — MUST reference the same function object
// This is why named functions are often needed instead of inline arrows
button.removeEventListener("click", handleClick);

// Inline arrow — can't be removed because there's no reference
button.addEventListener("click", () => console.log("can't remove me"));
```

### The Event Object

When an event fires, the event handler receives an *event object* containing information about what happened.

```js
document.addEventListener("click", function(event) {
  event.type;          // "click"
  event.target;        // the element that was actually clicked
  event.currentTarget; // the element the listener is attached to
  event.timeStamp;     // when the event occurred (milliseconds since page load)

  event.preventDefault();    // prevent default browser behavior (e.g., form submit)
  event.stopPropagation();   // stop the event from bubbling up the DOM tree
});

// Mouse events
document.addEventListener("mousemove", (e) => {
  e.clientX; e.clientY;   // position relative to viewport
  e.pageX; e.pageY;       // position relative to whole page
});

// Keyboard events
document.addEventListener("keydown", (e) => {
  e.key;     // "a", "Enter", "ArrowLeft", "Escape"
  e.code;    // "KeyA", "Enter" (physical key location)
  e.ctrlKey; e.shiftKey; e.altKey; e.metaKey; // modifier keys held
  if (e.key === "Enter" && e.ctrlKey) submitForm();
});
```

### Event Bubbling and Capturing

When you click a nested element, the click event does not just fire on that element. It travels through the DOM in two phases. The *capturing phase* travels down from the document root to the target element. The *bubbling phase* travels back up from the target to the document root. Most real-world event listeners use the bubbling phase (the default).

```js
// HTML: <div id="outer"><div id="inner"><button>Click</button></div></div>

document.getElementById("outer").addEventListener("click", () => console.log("outer"));
document.getElementById("inner").addEventListener("click", () => console.log("inner"));
document.querySelector("button").addEventListener("click", () => console.log("button"));

// Clicking the button outputs:
// "button" — target phase
// "inner" — bubbling up
// "outer" — bubbling up

// To use the capturing phase, pass `true` (or { capture: true }) as third argument
document.getElementById("outer").addEventListener("click", () => {
  console.log("outer (capturing)");
}, true); // fires BEFORE the target
```

`event.stopPropagation()` stops the event from continuing its journey through the tree. `event.stopImmediatePropagation()` also prevents other handlers on the same element from running.

### Event Delegation

Instead of attaching a listener to every item in a list (which is wasteful and breaks for dynamically added items), attach a single listener to the parent and use `event.target` to determine which item was interacted with. This is called event delegation.

```js
// Inefficient — a listener on every button
document.querySelectorAll(".product-list button").forEach(btn => {
  btn.addEventListener("click", handleAddToCart);
});
// Adding new products won't have a click handler!

// Efficient — single listener on the parent
document.querySelector(".product-list").addEventListener("click", (event) => {
  // Check if the click was on (or inside) an "add to cart" button
  const button = event.target.closest("[data-action='add-to-cart']");
  if (button) {
    const productId = button.dataset.productId;
    handleAddToCart(productId);
  }
});
// Handles all buttons, including ones added dynamically
```

> **Angular note:** Angular's event binding `(click)="handler($event)"` attaches event listeners behind the scenes. Angular's `@HostListener` decorator and CDK's event handling both use this underlying browser event system. Understanding event delegation helps you understand why Angular's event binding is efficient.

### Custom Events

You can create and dispatch your own custom events to communicate between loosely coupled parts of an application.

```js
// Create and dispatch a custom event
const event = new CustomEvent("productAdded", {
  detail: { id: 42, name: "Widget", price: 9.99 }, // payload
  bubbles: true,    // allow it to bubble up the DOM
  cancelable: true  // allow it to be cancelled with preventDefault()
});
document.querySelector(".cart").dispatchEvent(event);

// Listen for the custom event anywhere in the tree (because it bubbles)
document.addEventListener("productAdded", (event) => {
  console.log("Product added:", event.detail.name);
  updateCartCount(event.detail.id);
});
```

### Common Event Types Reference

Understanding which event to use for each scenario is a practical skill.

For form interactions, `input` fires every time the value changes (on every keystroke for text inputs). `change` fires when the value is committed (when the input loses focus, or immediately for checkboxes and selects). `submit` fires when a form is submitted — always call `event.preventDefault()` to handle it in JavaScript rather than triggering a page reload.

For keyboard events, `keydown` fires when a key is pressed and is the right event for shortcuts and intercepting Enter key presses. `keyup` fires when a key is released. `keypress` is deprecated — do not use it.

For mouse events, `click` fires on single click. `dblclick` on double click. `mousedown` and `mouseup` are the individual press and release phases. `mouseenter` and `mouseleave` do not bubble (useful for single-element hover). `mouseover` and `mouseout` do bubble (useful with delegation).

For document events, `DOMContentLoaded` fires when the HTML has been fully parsed and the DOM is ready, but before images and stylesheets finish loading. `load` fires when everything — including images — is fully loaded. For Angular, neither is typically needed because Angular bootstraps into a pre-existing `<app-root>` element.

---

## 4.15 JavaScript Web APIs

The browser exposes many powerful APIs beyond the DOM. Understanding the most important ones makes you effective in building real Angular applications and helps you understand what Angular libraries like `@angular/pwa` and the Angular CDK build on.

### Fetch API

`fetch` is the modern way to make HTTP requests from the browser. Angular's `HttpClient` is built on top of `XMLHttpRequest` (or `fetch` when you use `withFetch()`), but understanding the raw Fetch API helps you debug network issues.

```js
// Basic GET request
const response = await fetch("/api/users");
if (!response.ok) {
  throw new Error(`HTTP error! status: ${response.status}`);
}
const users = await response.json();

// POST request with JSON body
const response = await fetch("/api/users", {
  method: "POST",
  headers: {
    "Content-Type": "application/json",
    "Authorization": `Bearer ${token}`
  },
  body: JSON.stringify({ name: "Alice", email: "alice@example.com" })
});

// Streaming response (for AI-style text streaming)
const response = await fetch("/api/stream");
const reader = response.body.getReader();
const decoder = new TextDecoder();

while (true) {
  const { done, value } = await reader.read();
  if (done) break;
  const chunk = decoder.decode(value);
  appendToOutput(chunk);
}
```

### Web Storage — `localStorage` and `sessionStorage`

Web Storage provides a simple key-value store in the browser. `localStorage` persists across browser sessions (until explicitly cleared). `sessionStorage` is cleared when the browser tab is closed. Both are synchronous APIs and both store values as strings.

```js
// localStorage — persists across sessions
localStorage.setItem("theme", "dark");
const theme = localStorage.getItem("theme"); // "dark" or null
localStorage.removeItem("theme");
localStorage.clear(); // remove everything

// Storing objects — must serialize/deserialize manually
const user = { name: "Alice", role: "admin" };
localStorage.setItem("currentUser", JSON.stringify(user));
const savedUser = JSON.parse(localStorage.getItem("currentUser") || "null");

// A safe accessor to avoid JSON.parse errors
function getFromStorage(key, fallback = null) {
  try {
    const value = localStorage.getItem(key);
    return value ? JSON.parse(value) : fallback;
  } catch {
    return fallback;
  }
}
```

> **Security warning:** Never store authentication tokens, sensitive user data, or secrets in `localStorage`. It is accessible to any JavaScript running on the page, making it vulnerable to XSS attacks. Angular's security guidance recommends using `HttpOnly` cookies for auth tokens instead.

### Intersection Observer API

The Intersection Observer watches whether elements are visible in the viewport. This is the right tool for lazy loading images, triggering animations when elements scroll into view, and implementing infinite scroll.

```js
const observer = new IntersectionObserver(
  (entries) => {
    entries.forEach(entry => {
      if (entry.isIntersecting) {
        // The element is now visible in the viewport
        entry.target.classList.add("visible");
        entry.target.src = entry.target.dataset.src; // lazy load image

        // Stop observing once loaded
        observer.unobserve(entry.target);
      }
    });
  },
  {
    root: null,      // null = the viewport
    threshold: 0.1, // trigger when 10% of the element is visible
    rootMargin: "0px 0px 200px 0px" // trigger 200px before entering viewport
  }
);

// Observe all lazy images
document.querySelectorAll("img[data-src]").forEach(img => observer.observe(img));
```

Angular's `NgOptimizedImage` directive uses this API internally to implement lazy loading. The Angular CDK's `IntersectionObserverModule` wraps it in a declarative way.

### ResizeObserver API

The ResizeObserver fires a callback whenever an element's size changes. Use this instead of `window.resize` when you care about a specific element's size rather than the whole window.

```js
const observer = new ResizeObserver(entries => {
  for (const entry of entries) {
    const { width, height } = entry.contentRect;
    console.log(`Element is now ${width}px × ${height}px`);
    updateChartSize(width, height); // re-render a chart at new size
  }
});

observer.observe(document.querySelector(".chart-container"));
observer.disconnect(); // stop observing all elements
```

### MutationObserver API

The MutationObserver watches for changes to the DOM tree — added/removed nodes, attribute changes, or text content changes. Angular uses MutationObserver internally to detect when the DOM has been modified outside its control.

```js
const observer = new MutationObserver(mutations => {
  mutations.forEach(mutation => {
    if (mutation.type === "childList") {
      mutation.addedNodes.forEach(node => console.log("Added:", node));
    }
    if (mutation.type === "attributes") {
      console.log(`Attribute '${mutation.attributeName}' changed`);
    }
  });
});

observer.observe(document.querySelector(".dynamic-content"), {
  childList: true,  // watch for added/removed children
  subtree: true,    // watch the whole subtree
  attributes: true  // watch attribute changes
});
```

### `requestAnimationFrame`

`requestAnimationFrame` (rAF) schedules a callback to run just before the browser paints the next frame. Use it for smooth animations instead of `setInterval`, because rAF is synchronized to the display refresh rate and pauses automatically when the tab is inactive.

```js
function animate(timestamp) {
  // timestamp is the current time in milliseconds
  const progress = Math.min((timestamp - startTime) / duration, 1);
  element.style.transform = `translateX(${progress * 500}px)`;

  if (progress < 1) {
    requestAnimationFrame(animate); // schedule the next frame
  }
}

const startTime = performance.now();
requestAnimationFrame(animate); // kick off the animation

// Cancel an animation
const rafId = requestAnimationFrame(animate);
cancelAnimationFrame(rafId);
```

### Web Workers

JavaScript is single-threaded — long-running computations block the UI. Web Workers run JavaScript in a separate thread, keeping the main thread free for rendering and user interaction. The worker and the main thread communicate via message passing.

```js
// worker.js — runs in a separate thread
self.addEventListener("message", event => {
  const { data } = event;
  const result = heavyComputation(data); // doesn't block the UI!
  self.postMessage({ result });
});

// main.js — creates and communicates with the worker
const worker = new Worker(new URL("./worker.js", import.meta.url));

worker.postMessage({ input: largeDataset });

worker.addEventListener("message", event => {
  console.log("Result:", event.data.result);
});

// Clean up
worker.terminate();
```

### Performance API

The Performance API gives you fine-grained timing data for measuring how long operations take in your application.

```js
// High-resolution timestamp
performance.now(); // milliseconds since page load, e.g. 1523.45

// Mark significant moments
performance.mark("data-fetch-start");
const data = await fetchData();
performance.mark("data-fetch-end");

// Measure the duration between marks
performance.measure("data-fetch", "data-fetch-start", "data-fetch-end");

// Get the measurement
const measures = performance.getEntriesByName("data-fetch");
console.log(`Data fetch took: ${measures[0].duration.toFixed(2)}ms`);

// Core Web Vitals measurement with PerformanceObserver
const observer = new PerformanceObserver(list => {
  for (const entry of list.getEntries()) {
    if (entry.entryType === "largest-contentful-paint") {
      console.log("LCP:", entry.startTime);
    }
  }
});
observer.observe({ entryTypes: ["largest-contentful-paint"] });
```

### Clipboard API

```js
// Copy to clipboard (requires user gesture or permissions)
await navigator.clipboard.writeText("Copied text!");

// Read from clipboard (requires user permission)
const text = await navigator.clipboard.readText();
```

---

## 🔑 Key Takeaways — Part 5

Direct DOM manipulation is what Angular does for you, but you must understand it to write custom Angular directives, use `ViewContainerRef`, and debug rendering issues. `querySelector` is your go-to for element selection. `textContent` is always safe for setting text; `innerHTML` must be treated with caution. Event bubbling is why Angular event bindings on parent elements can respond to events from child elements. Event delegation — attaching one listener to a parent rather than many listeners to children — is a performance pattern that Angular's event binding system uses internally. The Observer APIs (Intersection, Resize, Mutation) are modern and efficient alternatives to polling or layout-triggered JavaScript, and Angular's CDK wraps them for convenient use in components.

---

*Part 5 of 6 — Phase 4 JavaScript Mastery | Next: [Design Patterns, FP, Performance & Modern JS](./part6-patterns-fp-performance-modern.md)*

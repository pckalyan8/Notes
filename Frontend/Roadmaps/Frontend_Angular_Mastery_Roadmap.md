# 🧭 Complete Frontend Mastery Roadmap — Angular Primary
### From Zero to Frontend Software Architect
#### Updated for Angular 21 · TypeScript 5.9 · 2026 Frontend Practices

> **How to use this roadmap:** Follow phases in order. Each phase builds on the previous. Topics within a phase can overlap. Time estimates are for dedicated full-time study. Adjust based on prior experience.

---

## 📌 TABLE OF CONTENTS

1. [Phase 0 — History & Evolution of the Web](#phase-0)
2. [Phase 1 — Internet & Web Fundamentals](#phase-1)
3. [Phase 2 — HTML Mastery](#phase-2)
4. [Phase 3 — CSS Mastery](#phase-3)
5. [Phase 4 — JavaScript Mastery (Core)](#phase-4)
6. [Phase 5 — TypeScript Mastery](#phase-5)
7. [Phase 6 — Browser Internals & Web Platform APIs](#phase-6)
8. [Phase 7 — Version Control & Developer Tooling](#phase-7)
9. [Phase 8 — Package Managers & Module Systems](#phase-8)
10. [Phase 9 — Build Tools & Bundlers](#phase-9)
11. [Phase 10 — Angular Core (Fundamentals)](#phase-10)
12. [Phase 11 — Angular Intermediate](#phase-11)
13. [Phase 12 — Angular Advanced](#phase-12)
14. [Phase 13 — Angular Ecosystem & Libraries](#phase-13)
15. [Phase 14 — State Management](#phase-14)
16. [Phase 15 — RxJS & Reactive Programming](#phase-15)
17. [Phase 16 — Testing (Frontend)](#phase-16)
18. [Phase 17 — Performance Optimization](#phase-17)
19. [Phase 18 — Accessibility (a11y)](#phase-18)
20. [Phase 19 — Security in Frontend](#phase-19)
21. [Phase 20 — Progressive Web Apps (PWA)](#phase-20)
22. [Phase 21 — Micro Frontends](#phase-21)
23. [Phase 22 — Design Systems & UI Architecture](#phase-22)
24. [Phase 23 — API Design & Integration](#phase-23)
25. [Phase 24 — DevOps for Frontend](#phase-24)
26. [Phase 25 — Monitoring, Observability & Analytics](#phase-25)
27. [Phase 26 — Cross-Platform & Mobile with Angular](#phase-26)
28. [Phase 27 — AI/ML Integration in Frontend](#phase-27)
29. [Phase 28 — Frontend Software Architecture](#phase-28)
30. [Phase 29 — Leadership, Communication & Soft Skills](#phase-29)
31. [Appendix — Reference Guides & Resources](#appendix)

---

<a name="phase-0"></a>
## 🕰️ PHASE 0 — History & Evolution of the Web
*Duration: 1–2 weeks | Goal: Understand why modern frontend exists*

### 0.1 The Birth of the Web
- Tim Berners-Lee and CERN (1989–1991)
- The first website and HTTP/HTML invention
- Early browsers: WorldWideWeb, Mosaic, Netscape Navigator
- The era of static HTML-only pages

### 0.2 The Browser Wars (1995–2001)
- Netscape vs Internet Explorer
- JavaScript invention by Brendan Eich (1995) — 10 days of creation
- JScript vs JavaScript incompatibilities
- CSS introduction (1996) and slow adoption
- The DOM as a concept

### 0.3 The DHTML & Flash Era (1998–2005)
- Dynamic HTML (DHTML) — combining HTML + CSS + JS
- Flash/ActionScript for rich media
- iframes and the first AJAX-like patterns
- XMLHttpRequest introduction (Microsoft, 1999)

### 0.4 The Web 2.0 Revolution (2004–2010)
- AJAX coined by Jesse James Garrett (2005)
- Google Maps — AJAX as proof of concept
- jQuery (2006) — DOM abstraction and cross-browser compatibility
- Rise of user-generated content and social platforms
- REST APIs and JSON emergence

### 0.5 The JavaScript Renaissance (2009–2015)
- Node.js (Ryan Dahl, 2009) — JavaScript on server
- V8 engine and JIT compilation
- npm (2010) — package ecosystem explosion
- Backbone.js, Knockout.js, Ember.js — MVC on the frontend
- AngularJS (Angular 1.x) by Google (2010)
- React by Facebook (2013)
- Grunt, Gulp — task runners

### 0.6 The Modern Framework Era (2015–2020)
- Angular 2 complete rewrite (2016) — TypeScript-first
- Vue.js rise (2014–2016)
- Webpack becoming the standard bundler
- ES6/ES2015 — the JavaScript modernization
- Babel — transpiling modern JS
- Rise of SPAs (Single Page Applications)
- Progressive Web Apps concept (Google, 2015)

### 0.7 The Current Era (2020–2026)
- Angular Ivy renderer → fully signal-based architecture
- Angular Standalone Components stable (v14+)
- Angular Signals stable and ecosystem-wide (v17+)
- Angular Zoneless (v20 stable) — Zone.js eliminated
- Angular 21: incremental hydration, signal router, full SSR renaissance
- Micro frontends via Module Federation and Native Federation
- Vite 6, esbuild, Rolldown — next-gen build tools
- Edge computing: Cloudflare Workers, Deno Deploy, Vercel Edge
- AI-assisted development mainstream (GitHub Copilot, Cursor, v0)
- Web Platform maturity: View Transitions API, Navigation API, CSS Anchor Positioning, Popover API, CSS Scroll-Driven Animations
- TypeScript 5.9 — inferred predicates, isolated modules improvements
- Islands Architecture and partial hydration becoming standard
- Bun as Node.js alternative for tooling workloads

---

<a name="phase-1"></a>
## 🌐 PHASE 1 — Internet & Web Fundamentals
*Duration: 1–2 weeks | Goal: Understand how the web actually works*

### 1.1 How the Internet Works
- IP addresses, DNS resolution
- TCP/IP protocol stack
- Packets, routers, and data transmission
- ISPs and the physical internet infrastructure

### 1.2 HTTP Protocol
- HTTP/1.0, HTTP/1.1, HTTP/2, HTTP/3
- Request/Response lifecycle
- HTTP methods: GET, POST, PUT, PATCH, DELETE, HEAD, OPTIONS
- HTTP status codes: 1xx, 2xx, 3xx, 4xx, 5xx
- HTTP headers: Content-Type, Authorization, Cache-Control, CORS, etc.
- Cookies and session management
- HTTP caching mechanisms: ETag, Last-Modified, Cache-Control

### 1.3 HTTPS & TLS/SSL
- Why HTTPS matters
- TLS handshake process
- Certificates, CA (Certificate Authority)
- HSTS (HTTP Strict Transport Security)
- Mixed content issues

### 1.4 URLs & URIs
- Structure: protocol, domain, port, path, query string, fragment
- URL encoding/decoding
- Absolute vs relative URLs

### 1.5 DNS Deep Dive
- DNS records: A, AAAA, CNAME, MX, TXT
- DNS propagation
- CDNs and DNS routing

### 1.6 Browsers & How They Work
- Browser components: UI, rendering engine, JS engine, networking
- Blink, WebKit, Gecko — rendering engines
- V8, SpiderMonkey — JS engines
- Browser security model
- Same-Origin Policy (SOP)
- CORS (Cross-Origin Resource Sharing)

### 1.7 Web Standards & Governance
- W3C, WHATWG, TC39 — who sets the standards
- ECMAScript specification
- RFC documents
- Caniuse.com and browser compatibility

---

<a name="phase-2"></a>
## 🏗️ PHASE 2 — HTML Mastery
*Duration: 2–3 weeks | Goal: Deep HTML expertise*

### 2.1 HTML Basics
- HTML document structure (DOCTYPE, html, head, body)
- Tags, elements, attributes
- Block vs inline elements
- Comments and whitespace handling

### 2.2 Semantic HTML
- Structural elements: header, nav, main, section, article, aside, footer
- Why semantics matter for SEO and accessibility
- Heading hierarchy (h1–h6)
- Lists: ul, ol, dl
- Figure, figcaption, blockquote, cite

### 2.3 HTML Forms
- Form element and action/method attributes
- Input types: text, email, password, number, date, file, checkbox, radio, range
- Select, textarea, fieldset, legend, label
- HTML5 validation: required, pattern, min, max, minlength, maxlength
- Datalist and autocomplete
- Form accessibility (for/id linking)

### 2.4 HTML Media
- Images: img, srcset, sizes, picture, WebP/AVIF formats
- Video: video, source, track (captions), controls
- Audio: audio, source
- iframe embedding and security (sandbox attribute)
- Canvas element (overview)
- SVG inline usage

### 2.5 HTML Tables
- When to use tables (tabular data only)
- thead, tbody, tfoot, tr, th, td
- colspan, rowspan
- Table accessibility (scope, caption)

### 2.6 HTML Metadata & Head Elements
- meta charset, viewport, description, OG tags
- Link: rel=stylesheet, rel=icon, rel=canonical, rel=preload, rel=prefetch
- Script: async, defer, type=module
- Open Graph and Twitter Card meta tags
- JSON-LD structured data basics

### 2.7 HTML APIs & Interfaces
- Custom data attributes (data-*)
- Draggable attribute and Drag and Drop API
- Contenteditable
- Spellcheck, translate attributes
- Template element and slot (Web Components basics)
- Dialog element

### 2.8 HTML5 Advanced
- Web Storage (localStorage, sessionStorage) — HTML5 origin
- Geolocation API
- Notifications API
- Web Workers (intro)
- History API (pushState, replaceState)
- Mutation Observer

---

<a name="phase-3"></a>
## 🎨 PHASE 3 — CSS Mastery
*Duration: 4–6 weeks | Goal: Complete CSS expertise*

### 3.1 CSS Fundamentals
- Syntax: selectors, properties, values
- How CSS is applied: inline, internal, external
- Cascade, specificity, inheritance
- CSS reset vs normalize
- The box model: content, padding, border, margin
- box-sizing: border-box

### 3.2 CSS Selectors & Combinators
- Type, class, ID, universal selectors
- Attribute selectors: [attr], [attr=val], [attr^=val], [attr$=val], [attr*=val]
- Pseudo-classes: :hover, :focus, :active, :nth-child, :nth-of-type, :not, :is, :where, :has
- Pseudo-elements: ::before, ::after, ::first-line, ::placeholder, ::selection
- Combinators: descendant, child (>), adjacent (+), general sibling (~)
- Specificity calculation rules

### 3.3 CSS Layout — Classic
- Display: block, inline, inline-block, none
- Position: static, relative, absolute, fixed, sticky
- Float and clear (legacy, understand for maintenance)
- z-index and stacking contexts
- Overflow: visible, hidden, scroll, auto, clip

### 3.4 CSS Flexbox
- Flex container vs flex items
- flex-direction, flex-wrap, flex-flow
- justify-content, align-items, align-content
- flex-grow, flex-shrink, flex-basis, flex shorthand
- align-self, order
- Common patterns: centering, navbars, card layouts
- Gap property

### 3.5 CSS Grid
- Grid container vs grid items
- grid-template-columns, grid-template-rows
- fr unit, repeat(), minmax()
- grid-area, grid-column, grid-row
- Named template areas
- Auto-placement algorithm
- Implicit vs explicit grid
- Subgrid
- Grid + Flexbox hybrid patterns

### 3.6 CSS Responsive Design
- Mobile-first approach
- Media queries: min-width, max-width, orientation, prefers-color-scheme, prefers-reduced-motion
- Viewport units: vw, vh, vmin, vmax, svh, dvh, lvh (dynamic viewport units)
- Fluid typography and fluid spacing
- Container queries (@container)
- clamp(), min(), max() functions

### 3.7 CSS Typography
- font-family, font-size, font-weight, font-style
- line-height, letter-spacing, word-spacing
- text-align, text-decoration, text-transform
- Web fonts: @font-face, Google Fonts, variable fonts
- font-display: swap, block, fallback, optional
- Typography scale and vertical rhythm
- Truncation: text-overflow, white-space, overflow

### 3.8 CSS Colors & Backgrounds
- Color formats: hex, rgb, rgba, hsl, hsla, oklch, color()
- CSS custom properties (CSS variables) — --var-name
- background-color, background-image, background-size, background-position
- Gradients: linear-gradient, radial-gradient, conic-gradient
- Multiple backgrounds
- background-blend-mode, mix-blend-mode
- filter and backdrop-filter

### 3.9 CSS Animations & Transitions
- transition: property, duration, timing-function, delay
- Timing functions: ease, ease-in, ease-out, ease-in-out, cubic-bezier, steps
- @keyframes
- animation: name, duration, timing-function, delay, iteration-count, direction, fill-mode, play-state
- will-change property and compositing
- CSS transforms: translate, rotate, scale, skew, matrix
- 3D transforms: rotateX, rotateY, perspective, transform-style: preserve-3d
- Animation performance (GPU compositing, avoid layout)

### 3.10 CSS Modern Features (2024–2026)
- Logical properties: margin-inline, padding-block
- CSS nesting (native — baseline 2024, all browsers)
- @layer (cascade layers) — specificity management at scale
- @scope — style scoping without BEM
- CSS Houdini overview: Paint API, Layout API
- :nth-child with selectors (CSS Selectors Level 4)
- color-mix(), relative colors (oklch-based design tokens)
- CSS scroll snap — carousels and paging without JS
- aspect-ratio property
- **CSS Anchor Positioning** — tooltip/popover positioning relative to any element
- **CSS @starting-style** — entry animations for newly-displayed elements
- **CSS Scroll-Driven Animations** — @keyframes driven by scroll progress
- **Popover API** (popover attribute) — native popovers without JS libraries
- **CSS if()** (experimental) — inline conditional values
- **View Transitions API (CSS layer)** — animate page/element transitions
- **@property** — typed CSS custom properties with inheritance control
- **light-dark()** — built-in dark mode value switching
- **field-sizing: content** — auto-sizing inputs and textareas
- **text-wrap: balance / pretty** — improved text wrapping algorithms

### 3.11 CSS Architecture & Methodologies
- BEM (Block Element Modifier)
- OOCSS (Object-Oriented CSS)
- SMACSS (Scalable and Modular Architecture)
- ITCSS (Inverted Triangle CSS)
- Utility-first CSS (Tailwind CSS concepts)
- Atomic CSS
- CSS Modules
- CSS-in-JS overview (Styled Components, Emotion)
- CSS specificity wars and how to prevent them
- Scalable naming conventions

### 3.12 CSS Preprocessors
- Sass/SCSS: variables, nesting, mixins, functions, extend, partials, @use, @forward
- Less overview
- PostCSS and plugin ecosystem
- Autoprefixer
- When to use preprocessors vs CSS native features

### 3.13 CSS in Angular Context
- Component styles and ViewEncapsulation
- ViewEncapsulation.Emulated (default), None, ShadowDom
- ::ng-deep (usage and deprecation)
- :host, :host-context selectors
- Angular Material theming with CSS custom properties
- Global styles vs component styles
- styleUrls vs styles in component decorator

---

<a name="phase-4"></a>
## ⚡ PHASE 4 — JavaScript Mastery (Core)
*Duration: 6–8 weeks | Goal: Deep JavaScript expertise*

### 4.1 JavaScript Fundamentals
- Variables: var, let, const — scoping differences
- Primitive types: string, number, bigint, boolean, undefined, null, symbol
- Reference types: objects, arrays, functions
- Type coercion and type conversion
- typeof, instanceof, Object.prototype.toString
- Truthy and falsy values
- Short-circuit evaluation

### 4.2 Operators & Expressions
- Arithmetic, assignment, comparison, logical operators
- Nullish coalescing (??) and optional chaining (?.)
- Spread operator (...) and rest parameters
- Comma operator
- Ternary operator
- Bitwise operators (awareness)

### 4.3 Control Flow
- if/else, switch
- for, for...in, for...of, while, do...while
- break, continue, labeled statements
- Iterators and the iteration protocol
- Generator functions (function*)

### 4.4 Functions Deep Dive
- Function declarations vs expressions vs arrow functions
- Hoisting behavior of each
- IIFE (Immediately Invoked Function Expressions)
- Default parameters
- Arguments object vs rest parameters
- First-class functions and higher-order functions
- Pure functions and side effects
- Function composition
- Currying and partial application
- Memoization
- Closures — lexical scope, closure over variables
- The call stack

### 4.5 Scope & Hoisting
- Global scope, function scope, block scope
- Lexical scope vs dynamic scope
- Hoisting: variable declarations (var) and function declarations
- Temporal Dead Zone (TDZ) for let/const
- Scope chain

### 4.6 The `this` Keyword
- `this` in global context
- `this` in object methods
- `this` in regular functions (strict mode vs sloppy mode)
- `this` in arrow functions (lexical this)
- `this` in class methods
- Explicit binding: call(), apply(), bind()
- `this` in event handlers

### 4.7 Objects & Prototypes
- Object literals, Object.create(), constructor functions, classes
- Property descriptors: value, writable, enumerable, configurable
- Object.defineProperty, Object.defineProperties
- Getters and setters (get/set)
- Prototype chain and __proto__
- Object.getPrototypeOf, Object.setPrototypeOf
- hasOwnProperty, Object.keys, Object.values, Object.entries
- Object.assign, Object.freeze, Object.seal
- Destructuring: objects and arrays
- Computed property names
- Property shorthand
- Prototype-based inheritance

### 4.8 JavaScript Classes
- Class declarations and expressions
- Constructor method
- Instance methods and static methods
- Private fields and methods (#field)
- Inheritance with extends and super
- Mixins pattern
- Abstract class pattern

### 4.9 Arrays & Array Methods
- Array creation: literals, Array.from(), Array.of()
- Mutating methods: push, pop, shift, unshift, splice, sort, reverse, fill, copyWithin
- Non-mutating methods: map, filter, reduce, reduceRight, find, findIndex, every, some, flat, flatMap, includes, indexOf, slice, join, concat
- Array destructuring and spread
- Iterating with forEach vs for...of
- Typed Arrays (overview)
- Array-like objects (NodeList, arguments)

### 4.10 Asynchronous JavaScript
- Call stack, event loop, task queue, microtask queue
- setTimeout, setInterval, clearTimeout, clearInterval
- Callbacks and callback hell
- Promises: states (pending, fulfilled, rejected)
- Promise chaining: .then(), .catch(), .finally()
- Promise combinators: Promise.all, Promise.allSettled, Promise.any, Promise.race
- async/await syntax
- Error handling in async code (try/catch)
- Async generators and for-await-of
- AbortController and AbortSignal

### 4.11 Error Handling
- Error types: Error, TypeError, RangeError, SyntaxError, ReferenceError
- throw statement
- try/catch/finally
- Custom error classes
- Error.prototype.stack
- Global error handlers: window.onerror, unhandledrejection

### 4.12 JavaScript Modules
- CommonJS (require/module.exports)
- ES Modules (import/export)
- Named vs default exports
- Re-exporting
- Dynamic import() — code splitting
- import.meta
- Module resolution algorithm
- Circular dependencies

### 4.13 JavaScript DOM APIs
- Selecting elements: getElementById, querySelector, querySelectorAll
- Creating, appending, removing elements
- innerHTML, textContent, innerText (differences)
- Attribute manipulation: getAttribute, setAttribute, dataset
- ClassList API: add, remove, toggle, contains, replace
- Style manipulation
- DOM traversal: parentNode, children, nextSibling, etc.
- DocumentFragment for batched DOM updates
- Shadow DOM overview

### 4.14 JavaScript Events
- addEventListener, removeEventListener
- Event object: target, currentTarget, type, preventDefault, stopPropagation
- Event bubbling and capturing (useCapture)
- Event delegation pattern
- Custom events (CustomEvent, dispatchEvent)
- Common event types: click, input, change, submit, keydown, keyup, focus, blur, resize, scroll, load, DOMContentLoaded
- Pointer events vs mouse events vs touch events

### 4.15 JavaScript Web APIs
- Fetch API and RequestInit options
- XMLHttpRequest (awareness for legacy)
- WebSockets
- localStorage, sessionStorage
- IndexedDB (basics)
- Web Workers
- Service Workers (overview)
- Geolocation API
- Clipboard API
- Intersection Observer API
- ResizeObserver API
- MutationObserver API
- requestAnimationFrame
- Performance API

### 4.16 JavaScript Design Patterns
- Creational: Singleton, Factory, Builder, Prototype
- Structural: Module, Decorator, Facade, Proxy, Adapter
- Behavioral: Observer/PubSub, Strategy, Command, Iterator, Mediator, State
- Functional patterns: Functor, Monad (conceptual)
- Anti-patterns to avoid

### 4.17 Functional Programming in JS
- Immutability
- Pure functions
- Function composition (compose, pipe)
- Map, filter, reduce as core abstractions
- Avoiding shared state
- Currying
- Point-free style
- Functors and monads (conceptual)

### 4.18 JavaScript Performance
- Memory management and garbage collection
- Memory leaks: closures, detached DOM nodes, event listeners
- V8 engine optimizations: hidden classes, inline caching
- Debounce and throttle
- Virtual scrolling concepts
- Avoid layout thrashing (read/write batching)

### 4.19 Modern JavaScript (ES2015–ES2025)
- ES2015 (ES6): arrow functions, classes, template literals, destructuring, modules, Promises, Symbol, Map, Set, WeakMap, WeakSet
- ES2017: async/await, Object.values/entries, String.padStart/padEnd
- ES2018: rest/spread for objects, Promise.finally, async iterators
- ES2019: Array.flat, Array.flatMap, Object.fromEntries, optional catch binding
- ES2020: BigInt, Promise.allSettled, globalThis, optional chaining (?.), nullish coalescing (??)
- ES2021: logical assignment (&&=, ||=, ??=), Promise.any, String.replaceAll, WeakRef
- ES2022: Array.at(), Object.hasOwn(), Error.cause, top-level await, class fields, private class methods
- ES2023: Array.findLast(), Array.findLastIndex(), Array.toSorted(), Array.toReversed(), Array.toSpliced(), Array.with(), Hashbang grammar
- ES2024: Promise.withResolvers(), Object.groupBy(), Map.groupBy(), ArrayBuffer.prototype.resize(), RegExp /v flag (Unicode sets), Atomics.waitAsync()
- ES2025: Iterator helpers (map, filter, take, drop, flatMap, reduce, toArray), import attributes (`with { type: "json" }`), duplicate named capture groups in RegExp, Float16Array, Set methods (union, intersection, difference, symmetricDifference, isSubsetOf, isSupersetOf)
- Upcoming Stage 3 proposals: Explicit Resource Management (using/await using), Decorator Metadata, Pipeline Operator (|>), Pattern Matching

---

<a name="phase-5"></a>
## 🔷 PHASE 5 — TypeScript Mastery
*Duration: 3–4 weeks | Goal: TypeScript for Angular development*

### 5.1 TypeScript Fundamentals
- Why TypeScript: static typing, tooling, scalability
- TypeScript compiler (tsc) and tsconfig.json
- Type annotations syntax
- Type inference
- Strict mode and compiler options

### 5.2 Basic Types
- string, number, boolean, null, undefined, symbol, bigint
- any, unknown, never, void
- Type assertions: as keyword and angle-bracket syntax
- Non-null assertion operator (!)
- Literal types: "admin" | "user"

### 5.3 Complex Types
- Arrays: number[], Array<number>
- Tuples: [string, number]
- Enums: numeric and string enums, const enums
- Objects and interfaces
- Type aliases (type keyword)
- Union types (|) and intersection types (&)

### 5.4 Interfaces & Type Aliases
- Interface declaration and usage
- Optional properties (?), readonly properties
- Index signatures: [key: string]: any
- Interface extension (extends)
- Interface merging (declaration merging)
- Interface vs type alias — when to use each

### 5.5 Functions in TypeScript
- Parameter types and return types
- Optional and default parameters
- Rest parameters
- Function overloads
- this parameter type
- Callback types

### 5.6 Classes in TypeScript
- Access modifiers: public, private, protected, readonly
- Parameter properties shorthand
- Abstract classes and methods
- Implementing interfaces (implements)
- Class decorators (experimental)
- Static members

### 5.7 Generics
- Generic functions, interfaces, classes
- Type constraints (extends)
- Default type parameters
- Generic utility types in practice
- Conditional types with generics
- infer keyword

### 5.8 Advanced Types
- Mapped types: { [K in keyof T]: ... }
- Conditional types: T extends U ? X : Y
- Template literal types
- Discriminated unions
- Recursive types
- Variadic tuple types
- **Inferred type predicates (TS 5.5)** — TypeScript infers `x is T` from return type analysis
- **const type parameters (TS 5.0)** — `<const T>` for literal inference in generics
- **using / await using** — Explicit Resource Management with Symbol.dispose

### 5.9 TypeScript Utility Types
- Partial<T>, Required<T>, Readonly<T>
- Pick<T, K>, Omit<T, K>
- Record<K, V>
- Exclude<T, U>, Extract<T, U>
- NonNullable<T>
- ReturnType<T>, Parameters<T>, ConstructorParameters<T>
- InstanceType<T>
- Awaited<T>
- NoInfer<T> (TS 5.4)
- **OmitThisParameter<T>**
- **ThisType<T>** for mixin patterns

### 5.10 TypeScript 5.x–5.9 Feature Highlights
- **TS 5.0:** const type parameters, decorators (TC39 Stage 3), verbatimModuleSyntax
- **TS 5.1:** Easier implicit return for undefined-returning functions; unrelated types for getters/setters
- **TS 5.2:** using/await using (Explicit Resource Management); decorator metadata
- **TS 5.3:** import attributes (`with { type: "json" }`); switch(true) narrowing improvements
- **TS 5.4:** NoInfer<T> utility type; preserved narrowing in closures after last assignment
- **TS 5.5:** Inferred type predicates; control flow for constant indexed access; RegExp /v flag
- **TS 5.6:** Disallowed nullish/truthy checks on non-nullable types; Iterator helpers typed
- **TS 5.7:** Checks for never-initialized variables; path rewriting for relative imports (--rewriteRelativeImportExtensions)
- **TS 5.8:** Granular return type checking in conditional branches; --erasableSyntaxOnly flag; require() of ESM modules in --module nodenext
- **TS 5.9:** Import deferral (`import defer`); improved --isolatedModules compatibility; enhanced tuple type labels

### 5.11 TypeScript Decorators
- What are decorators (metadata annotation pattern)
- Class decorators, method decorators, property decorators, accessor decorators, parameter decorators
- TC39 Stage 3 decorators (stable in TS 5.0+) vs legacy experimental decorators
- Decorator metadata (TS 5.2+) — built-in metadata without reflect-metadata
- Angular's use of decorators: @Component, @Injectable, @Input, @Output, etc.
- reflect-metadata and Angular's legacy DI system
- Angular 20/21: moving toward metadata-free DI via inject() function

### 5.12 TypeScript Configuration (tsconfig.json)
- compilerOptions: target, lib, module, strict, noImplicitAny, strictNullChecks
- **verbatimModuleSyntax** — replaces importsNotUsedAsValues/preserveValueImports
- **erasableSyntaxOnly** (TS 5.8) — type-only syntax safe for strip-type tools
- paths and baseUrl for module aliases
- include, exclude, files
- Project references (composite projects)
- Angular-specific tsconfig settings
- tsconfig.app.json vs tsconfig.spec.json
- **moduleResolution: bundler** — recommended for Angular + Vite/esbuild projects

### 5.13 TypeScript & Angular Integration
- Strongly typed component inputs and outputs (signal inputs — fully typed)
- Type-safe template forms (Typed Reactive Forms since Angular 14)
- Generic Angular services
- Type narrowing in templates (@if, @switch control flow)
- Typed reactive forms (FormGroup<T>)
- Strict templates (strictTemplates) — mandatory for production code
- **Template diagnostics** — strict signal type checks in Angular 20/21
- Type-safe route parameters with Angular Router typed params

---

<a name="phase-6"></a>
## 🌍 PHASE 6 — Browser Internals & Web Platform APIs
*Duration: 2–3 weeks | Goal: Understand the browser deeply*

### 6.1 Browser Rendering Pipeline
- Parsing HTML → DOM construction
- Parsing CSS → CSSOM construction
- Render tree construction
- Layout (Reflow)
- Paint
- Compositing
- How browsers optimize rendering (layers, GPU compositing)
- Critical rendering path optimization

### 6.2 JavaScript Engine Internals
- V8 architecture overview
- Parsing → AST → Bytecode → Machine code
- JIT compilation
- Hidden classes and inline caching
- Garbage collection: mark-and-sweep, generational GC
- Memory heap structure

### 6.3 The Event Loop Deep Dive
- Call stack
- Web APIs (setTimeout, fetch, DOM events)
- Callback/Task queue (macrotasks)
- Microtask queue (Promises, queueMicrotask, MutationObserver)
- Priority: microtasks before macrotasks
- requestAnimationFrame timing
- Long Tasks and INP (Interaction to Next Paint)
- Using scheduler.postTask()

### 6.4 Web Storage APIs
- Cookies: creation, expiry, SameSite, Secure, HttpOnly flags
- localStorage and sessionStorage: capabilities, limitations, sync API
- IndexedDB: async, transactional, structured data
- Cache API (for Service Workers)
- Origin Private File System (OPFS)
- Storage quotas and eviction policies

### 6.5 Communication APIs
- Fetch API deep dive: streams, AbortController
- WebSockets: handshake, frames, ping/pong
- Server-Sent Events (SSE): one-way streaming from server
- WebRTC: peer-to-peer, overview for frontend devs
- Broadcast Channel API: cross-tab communication
- SharedWorker

### 6.6 Security APIs
- Content Security Policy (CSP)
- Subresource Integrity (SRI)
- Cross-Origin Resource Policy (CORP)
- Cross-Origin Embedder Policy (COEP)
- Permissions API
- Credential Management API

### 6.7 Performance APIs
- Navigation Timing API
- Resource Timing API
- User Timing API (performance.mark, performance.measure)
- Long Tasks API
- PerformanceObserver
- Web Vitals: LCP, FID/INP, CLS, TTFB, FCP
- Core Web Vitals measurement

### 6.8 Intersection & Visibility
- Intersection Observer API — lazy loading, infinite scroll, analytics
- Visibility change event (Page Visibility API)
- Page Lifecycle API (freeze, discard states)

### 6.9 Offline & Background APIs
- Service Worker lifecycle: install, activate, fetch
- Workbox library (v7+)
- Background Sync API
- Background Fetch API
- Push API and Web Notifications
- Periodic Background Sync

### 6.10 Device & Hardware APIs
- Geolocation API
- Device Orientation and Motion
- Vibration API
- Bluetooth Web API (experimental)
- Web USB, Web Serial (experimental)
- Screen Orientation API
- Media Devices API (camera, microphone)
- Screen Capture API

### 6.11 Modern Web Platform APIs (2024–2026)
- **View Transitions API** — animate DOM changes and page navigations with `document.startViewTransition()`
- **Navigation API** — modern replacement for History API; intercept navigations, manage session history
- **Popover API** — native popovers with `popover` attribute, `popovertarget`, no JS library needed
- **Speculation Rules API** — prefetch/prerender declaratively for faster page loads
- **Scheduler API** — `scheduler.postTask()` for priority-based task scheduling
- **Compression Streams API** — native gzip/deflate in the browser
- **File System Access API** — read/write local files with user permission (OPFS)
- **Web Locks API** — coordinate async work across tabs/workers
- **EyeDropper API** — color picking from screen pixels
- **Shared Element Transitions** — cross-document view transitions
- **Cookie Store API** — async, Promise-based cookie management
- **Import Maps** — control how module specifiers resolve in the browser
- **Temporal API** (Stage 3) — modern date/time built-in to replace Date

---

<a name="phase-7"></a>
## 🔧 PHASE 7 — Version Control & Developer Tooling
*Duration: 1–2 weeks | Goal: Professional development workflow*

### 7.1 Git Fundamentals
- Initialize, clone, add, commit, push, pull
- Branching: branch, checkout, switch
- Merging: merge, fast-forward vs 3-way merge
- Rebasing: rebase, interactive rebase
- Stashing: stash, stash pop, stash list
- Tags: annotated vs lightweight tags
- Remotes: origin, upstream, fetch

### 7.2 Git Advanced
- Reflog — recovering lost commits
- Cherry-pick
- Bisect — binary search for bugs
- Blame and log — investigating history
- Hooks: pre-commit, pre-push, commit-msg
- Submodules
- Git worktrees

### 7.3 Branching Strategies
- GitFlow: feature, release, hotfix branches
- GitHub Flow: simple branch + PR model
- Trunk-Based Development
- Feature flags vs feature branches
- Monorepo branching strategies
- Release management

### 7.4 Code Review & Collaboration
- Pull Request (PR) / Merge Request (MR) best practices
- Writing good commit messages (Conventional Commits)
- CHANGELOG generation (conventional-changelog)
- Code review etiquette
- GitHub/GitLab features: Actions, Issues, Projects

### 7.5 Developer Tools (Browser DevTools)
- Elements panel: inspect, edit DOM/CSS, computed styles
- Console: logging, error tracking, REPL, console methods
- Network panel: request filtering, timing, headers, payload, throttling
- Performance panel: flame chart, long tasks, rendering
- Memory panel: heap snapshots, allocation profiler
- Application panel: storage, service workers, manifest
- Lighthouse audits
- Coverage panel: unused CSS/JS
- Sources panel: breakpoints, call stack, scope, watch expressions
- Mobile device simulation

### 7.6 IDE & Extensions (VS Code Focus)
- Workspace settings vs user settings
- Extensions for Angular: Angular Language Service, Angular Snippets
- Extensions: ESLint, Prettier, GitLens, Error Lens, Path Intellisense
- Debugging Angular in VS Code (launch.json)
- Multi-root workspaces
- Tasks and terminal integration
- Remote development (Dev Containers, Remote SSH)

---

<a name="phase-8"></a>
## 📦 PHASE 8 — Package Managers & Module Systems
*Duration: 1 week | Goal: Manage dependencies professionally*

### 8.1 npm Deep Dive
- package.json: name, version, scripts, dependencies, devDependencies, peerDependencies, optionalDependencies
- npm install, ci, update, audit, outdated
- Semantic versioning: major.minor.patch, ^, ~, ranges
- package-lock.json — deterministic installs
- npm scripts: lifecycle hooks (prebuild, postinstall)
- npx — execute packages without installing
- npm workspaces
- .npmrc configuration
- Publishing packages to npm registry
- Private registries (Verdaccio, Artifactory)

### 8.2 Yarn & pnpm
- Yarn Classic vs Yarn Berry (Plug'n'Play)
- yarn.lock and workspaces
- pnpm: symlink strategy, disk efficiency
- Lockfile differences
- Choosing between npm/yarn/pnpm

### 8.3 Module Systems
- CommonJS: require, module.exports, dynamic loading
- AMD (Asynchronous Module Definition) — historical context
- UMD (Universal Module Definition) — library compatibility
- ES Modules: static imports, tree-shakeable, browser-native
- Module resolution: Node.js algorithm
- Dual CJS/ESM packages

### 8.4 Dependency Management Best Practices
- Keeping dependencies updated
- Security audits: npm audit, Snyk, Dependabot
- License compliance
- Bundlesize and dependency cost awareness
- Avoiding dependency bloat

---

<a name="phase-9"></a>
## ⚙️ PHASE 9 — Build Tools & Bundlers
*Duration: 2–3 weeks | Goal: Understand the build pipeline*

### 9.1 Why Build Tools?
- Transpilation (ES2015+ → ES5)
- Module bundling
- Minification and uglification
- Tree-shaking (dead code elimination)
- Code splitting
- Asset handling (images, fonts, SVG)
- Source maps

### 9.2 Webpack
- Entry, output, loaders, plugins, mode
- Loaders: babel-loader, css-loader, style-loader, file-loader, url-loader
- Plugins: HtmlWebpackPlugin, MiniCssExtractPlugin, DefinePlugin
- Code splitting: SplitChunksPlugin, dynamic imports
- Webpack Dev Server and Hot Module Replacement (HMR)
- Bundle analysis: webpack-bundle-analyzer
- Angular uses Webpack internally (prior to v17 with esbuild)

### 9.3 Vite (v5/v6)
- ESM-based dev server (no bundling in dev)
- esbuild for pre-bundling dependencies
- Rollup for production builds
- Hot Module Replacement (HMR) — near-instant updates
- Vite 5: Environment API, improved SSR
- Vite 6: Environment API stable, experimental Rolldown bundler integration
- Vite plugins ecosystem
- Angular + Vite (Angular 17+ uses esbuild; Vite dev server available)

### 9.4 esbuild
- Go-based bundler — 10–100× faster than Webpack
- API: JavaScript and CLI
- Angular's application builder uses esbuild (Angular 17+)
- CSS bundling support
- Limitations: no code splitting complexity of Rollup

### 9.5 Rolldown (Emerging — 2025+)
- Rust-based Rollup-compatible bundler
- Being integrated into Vite 6 as the production bundler
- Dramatically faster than JavaScript bundlers
- Rollup API compatibility
- Expected to replace Rollup in most Vite-based stacks

### 9.6 Rollup
- ES module-first bundler
- Tree-shaking effectiveness
- Output formats: esm, cjs, umd, iife
- Used for library builds
- Angular library builds use ng-packagr + Rollup (migrating toward esbuild)

### 9.7 Babel
- Presets: @babel/preset-env, @babel/preset-typescript, @babel/preset-react
- Plugins: class properties, decorators
- browserslist configuration
- Babel vs TypeScript compiler — roles
- Babel becoming less critical as esbuild/SWC handle transpilation

### 9.8 SWC (Speedy Web Compiler)
- Rust-based TypeScript/JavaScript compiler
- Drop-in Babel replacement
- Used by Next.js, Parcel, Nx
- @swc/core and CLI usage
- SWC vs esbuild — use case comparison

### 9.9 Linting & Formatting (2026 Stack)
- **Biome** — unified Rust-based linter + formatter replacing ESLint + Prettier in many projects
- ESLint v9 (flat config `eslint.config.mjs` — breaking change from .eslintrc)
- @angular-eslint — Angular-specific rules (supports ESLint v9 flat config)
- Prettier: opinionated formatting (still widely used)
- Biome vs ESLint + Prettier trade-offs
- Husky: Git hooks automation
- lint-staged: lint only staged files
- Stylelint: CSS/SCSS linting (v16+)
- EditorConfig

### 9.10 Task Automation & Monorepo Tooling
- npm scripts as task runner
- **Nx 20+** — monorepo build system (primary for large Angular apps)
- Turborepo — alternative monorepo task runner
- Angular CLI as build orchestrator
- **Bun** — fast JS runtime + package manager + bundler, usable for Angular tooling scripts
- Make and shell scripts for automation
- GitHub Actions for CI builds

---

<a name="phase-10"></a>
## 🅰️ PHASE 10 — Angular Core (Fundamentals)
*Duration: 4–6 weeks | Goal: Solid Angular foundation*

### 10.1 Angular Overview & Philosophy
- Angular vs AngularJS (1.x) — complete rewrite
- Angular's opinionated nature — framework vs library
- Angular versioning and release cadence
- LTS support policy
- Angular team and community

### 10.2 Angular CLI (Angular 21)
- ng new — project creation options (standalone by default since Angular 17)
- ng serve — development server with HMR (esbuild-based, Vite dev server)
- ng build — production builds (esbuild application builder)
- ng generate (g): component, service, directive, pipe, guard, interceptor, interface, enum, class, resolver
- ng add — schematics for third-party integration
- ng update — migration tool (schematic-based automated migrations)
- ng test, ng lint, ng e2e
- **ng cache** — local build cache management
- workspace (angular.json) configuration
- **application builder** (`@angular-devkit/build-angular:application`) — default since Angular 17
- Project structure overview — flat standalone app (no app.module.ts)

### 10.3 Angular Workspace Structure
- angular.json — workspace configuration
- package.json — dependencies
- tsconfig.json, tsconfig.app.json, tsconfig.spec.json
- src/main.ts — bootstrap entry point
- src/app — application code
- src/assets — static files
- src/environments — environment configurations
- .editorconfig, .gitignore

### 10.4 Modules (NgModule) — Classic
- NgModule decorator: declarations, imports, exports, providers, bootstrap
- BrowserModule vs CommonModule
- Feature modules
- Shared modules pattern
- Core module pattern
- Lazy loading with NgModule (loadChildren)
- Root module (AppModule)
- forRoot() and forChild() patterns

### 10.5 Standalone Components (Modern Angular)
- Standalone component: standalone: true
- Importing dependencies directly in component
- Bootstrapping with standalone components (bootstrapApplication)
- ApplicationConfig
- Migrating from NgModule to Standalone
- Standalone directives and pipes
- provideRouter, provideHttpClient, etc.

### 10.6 Components
- @Component decorator: selector, template/templateUrl, styles/styleUrls
- Component lifecycle hooks (detailed below)
- Component interaction: @Input(), @Output(), EventEmitter
- Component selectors: element, attribute, class
- Template reference variables (#ref)
- ViewChild and ViewChildren
- ContentChild and ContentChildren
- Component host element: host property, @HostBinding, @HostListener

### 10.7 Component Lifecycle Hooks
- ngOnChanges: input changes, SimpleChanges object
- ngOnInit: initialization after inputs set
- ngDoCheck: custom change detection
- ngAfterContentInit: after content projection
- ngAfterContentChecked
- ngAfterViewInit: after view and children initialized
- ngAfterViewChecked
- ngOnDestroy: cleanup, unsubscribing

### 10.8 Templates & Data Binding
- Interpolation: {{ expression }}
- Property binding: [property]="expression"
- Event binding: (event)="handler()"
- Two-way binding: [(ngModel)] (FormsModule)
- Attribute binding: [attr.aria-label]
- Class binding: [class.active]="condition", [class]="object"
- Style binding: [style.color]="value", [style]="object"
- Template expressions — safe navigation operator (?.)

### 10.9 Built-in Directives & Control Flow
- **Modern control flow (Angular 17+, preferred):**
  - `@if` / `@else if` / `@else` — replaces *ngIf
  - `@for (item of items; track item.id)` with `@empty` — replaces *ngFor
  - `@switch` / `@case` / `@default` — replaces *ngSwitch
  - `@defer` / `@placeholder` / `@loading` / `@error` — deferrable views
  - `@defer` triggers: `on idle`, `on viewport`, `on interaction`, `on hover`, `on immediate`, `on timer`, `when condition`
- **Legacy structural directives (know for maintenance, migrate to control flow):**
  - *ngIf, *ngFor (with index, trackBy), *ngSwitch
- **Attribute directives:** NgClass, NgStyle (still used in templates)
- **Angular 20/21:** Control flow fully optimized for signal-based change detection; `@for` `track` expression required

### 10.10 Pipes
- Built-in pipes: DatePipe, CurrencyPipe, DecimalPipe, PercentPipe, UpperCasePipe, LowerCasePipe, TitleCasePipe, AsyncPipe, JsonPipe, SlicePipe, KeyValuePipe
- Creating custom pipes with @Pipe decorator
- Pure vs impure pipes
- Pipe chaining
- PipeTransform interface
- Locale configuration for i18n pipes

### 10.11 Services & Dependency Injection (DI)
- @Injectable decorator and providedIn
- Service singleton pattern
- Angular's hierarchical injector
- Root injector vs component injector
- Providing services: providedIn: 'root', 'platform', 'any', specific module/component
- Constructor injection
- inject() function (new functional DI)
- InjectionToken for non-class providers
- useClass, useValue, useFactory, useExisting providers
- Multi providers

### 10.12 Routing (Angular Router)
- RouterModule.forRoot() and RouterModule.forChild()
- provideRouter() for standalone
- Route configuration: path, component, redirectTo, pathMatch
- RouterOutlet
- RouterLink and RouterLinkActive
- Nested (child) routes
- Wildcard routes and 404 handling
- Router events
- ActivatedRoute: params, queryParams, data, snapshot vs Observable

---

<a name="phase-11"></a>
## 🅰️ PHASE 11 — Angular Intermediate
*Duration: 4–6 weeks | Goal: Real-world Angular application development*

### 11.1 Advanced Routing (Angular 21)
- Lazy loading routes: `loadComponent`, `loadChildren`
- Route guards: `CanActivate`, `CanDeactivate`, `CanMatch`, `CanActivateChild`
- **Functional guards (Angular 15+):** `canActivate` as plain function — preferred
- Route resolvers: `resolve` with functions (functional resolvers)
- Route parameters: `:id`, multiple params; **typed route params (Angular 20+)**
- Query parameters and fragment
- Named router outlets and auxiliary routes
- **`withComponentInputBinding()`** — automatically bind route params to component `input()` signals
- Preloading strategies: `NoPreloading`, `PreloadAllModules`, custom
- **`withViewTransitions()`** — integrate View Transitions API with Angular Router
- **`withNavigationErrorHandler()`** — centralized navigation error handling
- RouterLink active: `routerLinkActive`, `routerLinkActiveOptions`
- **`TitleStrategy`** — custom page titles per route
- **Signal-based Router (Angular 21 preview):** router state exposed as signals via `injectRouterState()`
- **`redirectTo` as function** — dynamic redirect based on route/state

### 11.2 Forms — Template-Driven
- FormsModule setup
- ngModel for two-way binding
- ngForm and template reference
- Form validation: required, minlength, maxlength, email, pattern
- ngModel class states: ng-valid, ng-invalid, ng-dirty, ng-touched, ng-pristine
- Displaying validation errors
- Form submission

### 11.3 Forms — Reactive
- ReactiveFormsModule setup
- FormControl, FormGroup, FormArray
- FormBuilder service
- Validators: built-in, custom synchronous, custom asynchronous
- Cross-field validation
- Dynamic form controls
- updateOn: 'change' | 'blur' | 'submit'
- FormRecord
- Typed Reactive Forms (Angular 14+): FormGroup<T>
- ControlValueAccessor — custom form controls
- Form arrays for dynamic fields
- Resetting forms

### 11.4 Forms — Signal-Based (Angular 20+ Experimental)
- `signalForm()` — top-level form primitive returning a signal-based form group
- `signalFormControl()` — individual control with `.value` as a signal
- `signalFormGroup()` — typed group of signal controls
- `signalFormArray()` — dynamic array of signal controls
- Reactive validation — validators re-run automatically when signal inputs change
- Async validators with signal-based status (`pending`, `valid`, `invalid`)
- `ControlValueAccessor` compatibility — reuse existing custom form controls
- Two-way binding with `model()` — signal forms + `[(model)]` for parent sync
- Comparing signal forms vs reactive forms: migration path and trade-offs
- Known limitations and stability caveats (experimental API surface)

### 11.5 HTTP & HttpClient (Angular 21)
- `provideHttpClient()` — standalone provider (module-free)
- `withFetch()` — use native fetch instead of XHR (recommended; required for SSR)
- `withInterceptors([...])` — functional interceptors array (preferred over class-based)
- `withInterceptorsFromDi()` — legacy class-based interceptors compatibility
- `withRequestsMadeViaParent()` — for child injector HTTP config
- `withNoXsrfProtection()` — disable default XSRF handling when not needed
- HttpClient methods: `get`, `post`, `put`, `patch`, `delete`, `head`, `request`
- `HttpHeaders`, `HttpParams` — immutable value objects
- Typed responses with generics
- Error handling with `catchError`, `retry`, `retryWhen`
- Progress events: upload/download progress tracking
- `HttpContext` — request-scoped metadata for interceptors
- **`httpResource()` (Angular 21)** — signal-based HTTP resource; auto-loading, error, value states
- Transfer State in SSR — `TransferState`, `makeStateKey()`
- Functional interceptor pattern: `(req, next) => next(req.clone({...}))`
- Authentication interceptor (bearer token injection)
- Error interceptor (global HTTP error handling)
- Caching interceptor pattern with Map-based cache

### 11.6 Change Detection
- How Angular's change detection works
- Zone.js and NgZone
- Default change detection strategy
- OnPush change detection strategy
- Triggering change detection: MarkForCheck, detectChanges
- runOutsideAngular for performance
- Change detection cycle order
- Detaching change detector

### 11.7 Content Projection
- Basic content projection: <ng-content>
- Multi-slot content projection: <ng-content select>
- NgTemplateOutlet — rendering templates dynamically
- TemplateRef and ViewContainerRef
- ng-container — grouping without DOM element
- ng-template — template definition

### 11.8 Angular Animations
- BrowserAnimationsModule / provideAnimations()
- trigger, state, style, animate, transition
- :enter and :leave transitions
- keyframes animation
- Animation sequences and groups
- Animating route transitions
- AnimationBuilder for programmatic animations
- Reusable animation definitions

### 11.9 Custom Directives
- Creating attribute directives with @Directive
- ElementRef and Renderer2 for safe DOM manipulation
- HostListener for events in directives
- HostBinding for property binding in directives
- Input in directives
- Structural directive creation (*appIf pattern)
- Directive composition API (Angular 15+)

### 11.11 View Transitions API in Angular
- `withViewTransitions()` in `provideRouter()` — automatic CSS-animated route transitions
- `document.startViewTransition()` for manual transitions
- `view-transition-name` CSS property for element-level transitions
- `@starting-style` CSS for entry animations
- Cross-document view transitions (MPA transitions)
- `ViewTransitionInfo` in router — querying transition state
- Combining with Angular animations for complex effects
- Accessibility: `prefers-reduced-motion` and view transitions
- Smart vs Presentational (dumb) components
- Container/Presenter pattern
- Compound components
- Service with Subject pattern (shared state)
- Component communication: parent-child, sibling via service
- Dynamic components: ViewContainerRef.createComponent()
- Angular CDK Portal

---

<a name="phase-12"></a>
## 🅰️ PHASE 12 — Angular Advanced
*Duration: 4–6 weeks | Goal: Expert Angular capabilities*

### 12.1 Angular Signals (Angular 16 → 21)
- What are Signals and why they exist (fine-grained reactivity, no Zone.js required)
- `signal()` — writable signal with `.set()`, `.update()`, `.mutate()` (deprecated), `.asReadonly()`
- `computed()` — derived signal, lazy and memoized, glitch-free evaluation
- `effect()` — side effects; cleanup with `onCleanup`; `effect()` scheduling semantics
- Signal-based inputs: `input()`, `input.required()` — replaces @Input()
- Signal-based outputs: `output()` — replaces @Output() + EventEmitter
- `model()` — two-way binding signal; writable signal with output semantic
- `toSignal()` and `toObservable()` — RxJS interop (from @angular/core/rxjs-interop)
- Signal-based queries: `viewChild()`, `viewChildren()`, `contentChild()`, `contentChildren()` — replaces @ViewChild/@ContentChild
- `linkedSignal()` (Angular 19+) — writable derived signal that resets on source change
- **Resource API (Angular 19+):** `resource()` and `rxResource()` — async signal-based data fetching with loading/error/value states
- **httpResource (Angular 21)** — `httpResource()` for HTTP requests returning a signal-based Resource
- Signal change detection — components re-render only when consumed signals change
- **Signal components (Angular 20+):** components that declare `changeDetection: ChangeDetectionStrategy.OnPush` by default when using signal inputs
- **Untracked reads:** `untracked()` — read signals without creating reactive dependencies
- **afterRender / afterNextRender** — lifecycle hooks safe to use in signals context
- Angular DevTools support for signal graph visualization

### 12.2 Zoneless Angular (Angular 18 preview → 20 stable → 21 recommended)
- `provideZonelessChangeDetection()` — stable since Angular 20 (replaces experimental API)
- Completely removing Zone.js from the bundle (saves ~13kB)
- How zoneless change detection works: signal-driven scheduling + `markForCheck` API
- `ChangeDetectionStrategy.OnPush` is the new default in zoneless context
- Signal-based components — naturally zoneless-compatible
- Migrating Zone.js-dependent code: async pipes, setTimeout patterns
- Testing zoneless components: `provideZonelessChangeDetection()` in TestBed
- **Angular 21:** `zoneless: true` in angular.json to opt new projects in by default
- Performance benchmarks: 30–50% improvement in TTI with zoneless + signals
- Third-party library compatibility checklist for zoneless

### 12.3 Dependency Injection — Advanced
- Hierarchical DI tree: platform → root → module → component
- ElementInjector vs ModuleInjector
- Self, SkipSelf, Optional, Host decorators
- ViewProviders vs providers in components
- Multi-providers and ordered execution
- Factory providers with dependencies
- Environment Injectors
- runInContext()
- Injection contexts

### 12.4 Angular Compiler
- Angular template compilation
- JIT (Just-In-Time) compilation
- AOT (Ahead-of-Time) compilation — production default
- Ivy renderer — what changed from ViewEngine
- Incremental DOM concept
- Template type checking (strictTemplates)
- View Engine vs Ivy differences
- Angular Language Service

### 12.5 Angular SSR (Server-Side Rendering — Angular 17–21)
- Why SSR: SEO, LCP, social sharing metadata
- **Angular 17+:** SSR built into Angular CLI — `ng new --ssr`; no separate `@nguniversal` package
- Server-side rendering with Express (built-in) or Node.js HTTP
- **Transfer State:** transferring server-fetched data to the browser to avoid double fetching
- **Non-destructive hydration (Angular 16+):** DOM reuse; no full re-render on bootstrap
- **Incremental hydration (Angular 19+):** `@defer` blocks hydrated lazily on interaction/viewport
- **Partial hydration (Angular 21):** components opt into hydration independently; static content stays inert
- Static Site Generation (SSG/prerendering): `prerender` option in angular.json with route discovery
- **App Shell pattern:** pre-render shell HTML, hydrate on load
- **Event replay (Angular 19+):** user events captured before hydration, replayed after
- SSR-specific guards: `isPlatformBrowser()`, `PLATFORM_ID`, `afterRender()`
- `HttpClient` with `withFetch()` — uses native fetch in server context
- Absolute URL requirement for server-side HTTP calls
- **Edge SSR:** deploying Angular SSR to Cloudflare Workers, Vercel Edge, Deno Deploy
- `renderApplication()` and `renderModule()` APIs

### 12.6 Angular Build System (Angular 17–21)
- **Application builder** (`@angular-devkit/build-angular:application`) — default since Angular 17
- **esbuild** for production bundling — 3–5× faster than Webpack
- **Vite-based dev server** for local development with native HMR
- Build options: optimization (scripts, styles, fonts), sourceMap, budgets, baseHref
- **Build cache** — persistent local cache; `ng cache enable/disable/clean`
- Bundle budgets — warning and error thresholds in angular.json
- **`outputMode: 'static' | 'server' | 'server-side-rendering'`** in angular.json
- Named chunks and lazy chunk naming
- `externalDependencies` for excluding packages from bundle
- `define` for compile-time constants (replacing environment.ts pattern)
- Source map explorer for production bundle analysis
- **Angular 21:** experimental Rolldown integration for even faster builds
- Webpack removed as default — only available via `@angular-builders/custom-webpack` for edge cases

### 12.7 Performance Optimization in Angular (2026 Best Practices)
- **Signals everywhere** — replace all Subject-based state with signals for fine-grained updates
- **Zoneless + Signals** — the gold standard; ~40% bundle reduction + faster CD
- `track` expression in `@for` (required — no trackBy function needed)
- Pure pipes over methods in templates (still applies)
- **`@defer` with viewport/interaction triggers** — eliminate initial bundle weight
- Preloading strategies: custom preloading for critical routes
- **CDK Virtual Scrolling** — `CdkVirtualScrollViewport` for long lists
- **NgOptimizedImage** (`ngSrc`) with automatic `srcset`, `preconnect`, LCP image priority
- **`afterNextRender`** — defer DOM reads to after paint, avoiding layout thrashing
- Bundle analysis: `ng build --stats-json` → `npx webpack-bundle-analyzer`; source-map-explorer
- Tree-shaking — `providedIn: 'root'` ensures tree-shakeable services
- **`withFetch()`** in `provideHttpClient()` — native fetch, smaller bundle
- Font optimization: `optimizeFonts: true` in angular.json
- **`@angular/core/rxjs-interop`** — `takeUntilDestroyed()`, avoid manual subscriptions
- Removing unused Angular modules / using standalone components exclusively
- **Hydration deferral** — combine `@defer` + incremental hydration for islands-like patterns

### 12.8 Angular Internationalization (i18n)
- Angular built-in i18n with @angular/localize
- i18n attribute and $localize tag
- Translation files: XLF, XLF2, JSON
- Running ng extract-i18n
- Building for multiple locales
- Runtime translation (transloco, ngx-translate)
- Transloco library deep dive
- Date, number, currency locale pipes

### 12.9 Advanced Component Patterns
- Slot-based design with ng-content
- Dynamic component loader pattern
- Angular CDK — Component Dev Kit
- CDK Overlay, CDK DragDrop, CDK Stepper, CDK Table, CDK Virtual Scroll
- Portals and overlays
- Compound component pattern with InjectionToken
- Headless UI components

### 12.10 Angular Testing — Advanced
- TestBed configuration and module setup
- Component testing with DOM interaction
- Mocking Angular dependencies
- Async testing: fakeAsync, tick, flushMicrotasks, waitForAsync
- HTTP testing with HttpClientTestingModule
- Router testing
- Form testing
- Custom matchers
- Harnesses (CDK component harnesses)

### 12.11 Angular Security
- Cross-Site Scripting (XSS) prevention — Angular's sanitization
- DomSanitizer: bypassSecurityTrustHtml, bypassSecurityTrustUrl
- CSRF protection with Angular HttpClient
- Content Security Policy (CSP) headers
- Angular Trusted Types support
- HttpOnly cookies for auth tokens
- JWT handling best practices in Angular

### 12.12 Angular Libraries & ng-packagr
- Creating Angular libraries with ng-library schematic
- ng-packagr: building libraries
- Secondary entry points
- Publishing to npm
- Peer dependencies configuration
- Testing libraries
- Angular Package Format (APF)

---

<a name="phase-13"></a>
## 🅰️ PHASE 13 — Angular Ecosystem & Libraries
*Duration: 2–3 weeks | Goal: Master the Angular ecosystem*

### 13.1 Angular Material
- MatModule imports and standalone components
- Theming system: M2 vs M3 (Material 3)
- Typography system
- Components: Button, Card, Form Field, Input, Select, Autocomplete, Checkbox, Radio, Slider, Slide Toggle, DatePicker, Dialog, Snackbar, Tooltip, Menu, Tabs, Table, Paginator, Progress indicators, Stepper, Tree, List, Chips, Badge
- CDK utilities underlying Material
- Custom theming with SCSS mixins
- Dark mode support

### 13.2 Angular CDK (Component Dev Kit)
- Accessibility (a11y): FocusTrap, LiveAnnouncer, AriaDescriber
- DragDrop: sortable lists, transferable lists
- Overlay: position strategies, scroll strategies
- Portal: dynamically rendering components/templates
- Virtual scrolling: FixedSizeVirtualScrollStrategy
- Stepper
- Table: CDK Table + DataSource
- Layout: BreakpointObserver, MediaMatcher
- Text Field: AutofillMonitor, CdkTextareaAutosize
- Clipboard: CdkCopyToClipboard
- Scrolling utilities

### 13.3 NgRx (Redux Pattern for Angular — v18+)
- Store: single source of truth
- Actions: type safety with `createAction`, `props<T>()`
- Reducers: pure functions, `on()` helper, `createReducer()`
- Selectors: `createSelector`, memoization, projector functions
- Effects: `createEffect`, `Actions` stream, `ofType()` operator
- Entity: `@ngrx/entity` — normalized collection management
- Router Store: `@ngrx/router-store` — syncing Angular Router with NgRx
- Component Store: `@ngrx/component-store` — local component-level state
- **NgRx Signals Store (`@ngrx/signals`)** — signal-based store (recommended for new projects)
  - `signalStore()`, `withState()`, `withComputed()`, `withMethods()`, `withHooks()`
  - `withEntities()` — entity management in signal store
  - Custom store features with `signalStoreFeature()`
  - `patchState()` — immer-like state updates
  - `selectSignal()` — signal-based selectors
- Redux DevTools: time-travel debugging
- NgRx best practices: action hygiene, selector composition, effect isolation

### 13.4 Other State Management Options
- **TanStack Store** — framework-agnostic reactive store (Angular adapter)
- NGXS — decorator-based state management
- Elf — lightweight reactive store with RxJS
- Signal Store patterns without NgRx (for simpler apps)
- Choosing the right solution: signals for local/medium state, NgRx Signals or NgRx for large-scale

### 13.5 Routing Libraries
- Angular Router (primary — covered above)
- @ngrx/router-store for state integration

### 13.6 Form Libraries
- Angular Reactive Forms (built-in)
- ng-select (advanced select)
- ngx-formly — dynamic forms
- Angular Material form integration

### 13.7 HTTP & Data Libraries
- Angular HttpClient + `httpResource()` (Angular 21 built-in)
- **TanStack Query for Angular (`@tanstack/angular-query-experimental`)** — server state management
- Apollo Angular (v7+) — GraphQL client with signal support
- **RxAngular** — template rendering and state management with RxJS
- swr-like patterns implemented with signals + Resource API

### 13.8 UI Component Libraries
- Angular Material (official)
- PrimeNG — feature-rich component library
- NG-ZORRO — Ant Design for Angular
- Nebular — theming-focused
- Angular Clarity (VMware)
- Taiga UI
- DevExtreme Angular
- AG Grid (data grids)

### 13.9 Animation Libraries
- Angular Animations (built-in)
- GreenSock (GSAP) with Angular
- Framer Motion (via Web Animations API)
- Three.js with Angular (3D graphics)

### 13.10 Utility Libraries
- Lodash / Lodash-es (tree-shakeable)
- date-fns / dayjs / Luxon (date handling)
- UUID generation
- class-validator and class-transformer (DTOs in Angular services)
- Zod (schema validation)

### 13.12 Analog.js — Meta-Framework for Angular
- What is Analog.js: Angular's answer to Next.js / Nuxt
- File-based routing (similar to Next.js `app/` directory)
- Single-file components (`.analog` extension)
- Server-side rendering and static generation built-in
- API routes (Node.js endpoints collocated with Angular app)
- Vite-powered build (esbuild + Rollup)
- Markdown-based content pages
- When to choose Analog vs Angular CLI with SSR
- `@analogjs/vitest-angular` — first-class Vitest support
- Deployment: Vercel, Netlify, Cloudflare, Node.js server
- Why Nx for Angular monorepos: task orchestration, affected builds, caching
- Nx workspace setup: `npx create-nx-workspace --preset=angular-monorepo`
- **Nx 20:** improved project crystal (inferred tasks from package.json/angular.json)
- Nx generators and executors (plugins)
- Project configuration (project.json / inferred from angular.json)
- **Nx library layers:** `feature` → `ui` → `data-access` → `util` — domain layering
- Implicit and explicit project dependencies (project graph)
- `nx affected` — only build/test/lint what changed
- **Caching:** local cache + Nx Cloud (remote distributed cache)
- **Task distribution (Nx Agents)** — distribute CI tasks across machines
- **Nx Console** (VS Code + JetBrains extension)
- ESLint module boundary rules: `@nx/enforce-module-boundaries`
- **Nx Powerpack** — additional enterprise features
- Angular + Node.js in same Nx workspace (full-stack monorepo)
- Migrating existing Angular workspaces to Nx

---

<a name="phase-14"></a>
## 🗄️ PHASE 14 — State Management
*Duration: 3–4 weeks | Goal: Master state architecture*

### 14.1 State Management Fundamentals
- What is application state?
- Types of state: UI state, server state, form state, URL state
- Local vs global state
- When NOT to use global state
- State colocation principle
- Unidirectional data flow

### 14.2 Component State Patterns
- Local state with signals
- Smart/dumb component pattern
- Lifting state up
- Component state machines

### 14.3 Service-Based State
- BehaviorSubject and ReplaySubject pattern
- Signal-based service store
- Facade pattern — abstracting state from components
- Repository pattern for data access

### 14.4 Redux Pattern / NgRx
- Action → Reducer → Store → Selector → Component → Action cycle
- Immutable state updates
- Normalized state (entities)
- Selector memoization with createSelector
- Effects for async operations
- Side effect management
- Optimistic updates
- Pessimistic updates

### 14.5 NgRx Signals Store (Modern Angular)
- signalStore() and withState(), withComputed(), withMethods()
- Entities with withEntities()
- Custom store features
- DevTools integration
- Comparing with classic NgRx

### 14.6 Server State Management
- Distinguishing client state from server state
- Caching server responses
- Background refetching
- Stale-while-revalidate pattern
- TanStack Query patterns
- Resource API (Angular 19+ signals)
- Optimistic UI patterns

### 14.7 URL State Management
- Router as source of truth for navigation state
- Storing filters/pagination in query params
- ActivatedRoute as state
- Router Store (NgRx)

### 14.8 Advanced State Patterns
- Event Sourcing in frontend
- CQRS on the frontend
- Undo/redo implementation
- Time-travel debugging with Redux DevTools
- State persistence (localStorage sync)
- Cross-tab state sync (Broadcast Channel)

---

<a name="phase-15"></a>
## 🔄 PHASE 15 — RxJS & Reactive Programming
*Duration: 4–5 weeks | Goal: Master reactive programming*

### 15.1 Reactive Programming Fundamentals
- Reactive paradigm vs imperative paradigm
- Streams and data over time
- Observable streams concept
- Push vs Pull model
- Observer, Observable, Subscription

### 15.2 RxJS Core Concepts
- Observable: cold vs hot
- Observer: next, error, complete
- Subscription and unsubscribing
- Subject: Subject, BehaviorSubject, ReplaySubject, AsyncSubject
- Scheduler (conceptual)
- Marble diagrams — reading and writing

### 15.3 Creation Operators
- of, from, fromEvent, fromPromise
- interval, timer
- range
- throwError, EMPTY, NEVER
- defer
- ajax (RxJS Ajax)
- generate

### 15.4 Transformation Operators
- map, mapTo (deprecated, use map)
- mergeMap (flatMap) — parallel inner subscriptions
- concatMap — sequential inner subscriptions
- switchMap — cancel previous, switch to new
- exhaustMap — ignore new while active
- scan — running accumulation
- reduce
- pluck (deprecated, use map)
- groupBy
- bufferTime, bufferCount, buffer
- windowTime, windowCount
- toArray
- pairwise, startWith

### 15.5 Filtering Operators
- filter
- take, takeUntil, takeWhile, takeLast
- skip, skipUntil, skipWhile
- first, last
- distinctUntilChanged, distinct
- debounceTime, debounce
- throttleTime, throttle
- auditTime, audit
- sampleTime, sample

### 15.6 Combination Operators
- merge — concurrent merge of streams
- concat — sequential merging
- forkJoin — wait for all to complete (like Promise.all)
- combineLatest — combine latest values from each
- zip — pair by index
- withLatestFrom — combine with latest from second
- race — first to emit wins
- combineLatestWith, mergeWith, concatWith

### 15.7 Error Handling Operators
- catchError — recover from error
- retry, retryWhen, retry with delay
- throwIfEmpty
- onErrorResumeNext

### 15.8 Utility Operators
- tap — side effects without modifying stream
- delay, delayWhen
- timeout, timeoutWith
- finalize — like finally
- share, shareReplay — multicasting
- publish, refCount (lower level multicasting)
- observeOn, subscribeOn

### 15.9 RxJS in Angular Context
- AsyncPipe — subscribing in templates
- takeUntilDestroyed() (Angular 16+) — auto unsubscribe
- toSignal() and toObservable() — signals interop
- HttpClient returns Observables
- Router events as Observable
- Form value changes as Observable
- Store selects as Observables (NgRx)
- Subject-based event bus pattern
- Avoiding memory leaks in Angular

### 15.10 RxJS Advanced Patterns
- State machine with scan
- WebSocket with RxJS
- Polling with interval + switchMap
- Search-as-you-type with debounceTime + switchMap
- Infinite scroll with concatMap
- Undo/redo with scan
- Optimistic updates pattern
- Complex orchestration of async flows

### 15.11 RxJS Best Practices
- Naming Observables with $ suffix
- Preferring operators over manual subscription
- Flattening strategy selection guide
- Avoiding nested subscriptions
- Memory leak prevention patterns
- Testing RxJS (marble testing, TestScheduler)

---

<a name="phase-16"></a>
## 🧪 PHASE 16 — Testing (Frontend)
*Duration: 3–4 weeks | Goal: Comprehensive testing capability*

### 16.1 Testing Fundamentals
- Why testing matters: confidence, refactoring, documentation
- Testing pyramid: unit, integration, end-to-end
- TDD (Test-Driven Development) approach
- BDD (Behavior-Driven Development) approach
- Test coverage: statement, branch, function, line
- Code coverage tools and pitfalls (coverage ≠ quality)

### 16.2 Unit Testing with Jest / Vitest (2026 Stack)
- **Vitest** — Vite-native test runner, Jest-compatible API; preferred for new projects
  - `vitest.config.ts` setup with Angular plugin
  - `@analogjs/vitest-angular` — Angular + Vitest integration
  - Watch mode, UI mode, coverage with V8 or Istanbul
- **Jest** — still widely used; `jest-preset-angular` for Angular projects
- Common API: `describe`, `it`/`test`, `beforeEach`, `afterEach`, `beforeAll`, `afterAll`
- `expect` and matchers: `toBe`, `toEqual`, `toStrictEqual`, `toBeTruthy`, `toBeFalsy`, `toContain`, `toMatchObject`, `toMatchSnapshot`
- Mocking: `vi.fn()` / `jest.fn()`, `vi.spyOn()` / `jest.spyOn()`, `vi.mock()` / `jest.mock()`
- Timer mocks: `vi.useFakeTimers`, `vi.runAllTimers`
- Inline snapshots and file snapshots
- Testing async code: Promises, async/await, `vi.waitFor()`

### 16.3 Unit Testing with Karma + Jasmine (Legacy — migrating away)
- Jasmine syntax: describe, it, beforeEach, afterEach
- Jasmine matchers: toEqual, toBe, toHaveBeenCalled, toHaveBeenCalledWith
- Spies: jasmine.createSpy, spyOn
- Async testing: fakeAsync/tick, waitForAsync
- **Angular 21:** Karma officially deprecated; migrate to Jest or Vitest with `ng generate @angular/core:karma-to-jest`
- Karma-to-Vitest migration path

### 16.4 Angular-Specific Testing (Angular 21)
- `TestBed` — Angular testing module setup
- `ComponentFixture` — wrapper for component testing
- `DebugElement` — querying the DOM
- `provideZonelessChangeDetection()` in TestBed for zoneless component testing
- **Testing signal-based components:**
  - `TestBed.flushEffects()` — flush pending effects
  - Signals update synchronously in test context
  - `fixture.detectChanges()` still needed for template updates
- Async component testing: `fakeAsync`, `tick`, `waitForAsync`
- Router testing: `RouterTestingModule` (legacy), `RouterTestingHarness` (modern)
- HTTP testing: `provideHttpClientTesting()`, `HttpTestingController`
- Service testing: `TestBed.inject()`
- Pipe testing, Guard testing, Interceptor testing
- **CDK Component Harnesses** — type-safe, resilient component interaction testing
- `NO_ERRORS_SCHEMA` vs proper mocking — prefer real dependency injection
- Shallow rendering strategies for performance

### 16.5 Component Testing with Angular Testing Library
- @testing-library/angular
- User-centric queries: getByRole, getByText, getByLabelText
- userEvent for realistic interactions
- render() utility
- Why prefer Testing Library over fixture-based tests
- Accessibility-driven test queries

### 16.6 Integration Testing
- Testing multiple components together
- Testing component + service integration
- Mocking HTTP calls in integration tests
- Module-level integration tests

### 16.7 End-to-End Testing with Cypress
- Cypress setup and architecture
- cy.visit, cy.get, cy.find, cy.contains
- Chaining commands
- Assertions: should, expect
- Custom commands
- Fixtures and data management
- Intercept and stub network requests
- Cypress Component Testing (isolated component testing)
- CI integration
- Visual regression testing

### 16.8 End-to-End Testing with Playwright
- Playwright setup and browser support
- Page Object Model (POM)
- locator API
- Assertions: expect(locator)
- Network interception
- Screenshots and trace viewer
- Parallel test execution
- Playwright Test vs Cypress comparison
- @playwright/test for Angular

### 16.9 Visual Regression Testing
- Storybook + Chromatic
- Percy visual testing
- Playwright visual comparisons (toHaveScreenshot)
- Applitools Eyes

### 16.10 Test Architecture & Best Practices
- AAA pattern: Arrange, Act, Assert
- Test isolation and independence
- Test data builders (object mother pattern)
- Mock vs Stub vs Spy vs Fake vs Dummy
- Testing what matters (user behavior)
- Avoiding implementation detail testing
- When to use TestBed vs lightweight testing
- Flaky test prevention
- CI pipeline for tests
- Code coverage thresholds

---

<a name="phase-17"></a>
## 🚀 PHASE 17 — Performance Optimization
*Duration: 3–4 weeks | Goal: Build blazing-fast Angular apps*

### 17.1 Web Performance Fundamentals
- Core Web Vitals: LCP, INP, CLS
- Supporting metrics: TTFB, FCP, TTI, TBT
- Lighthouse performance scoring
- Performance budget concept
- Lab data vs field data (CrUX)
- Real User Monitoring (RUM)

### 17.2 Network Performance
- HTTP/2 multiplexing
- Resource hints: preload, prefetch, preconnect, dns-prefetch
- Content Delivery Networks (CDN)
- Compression: Brotli and Gzip
- Response caching strategies
- Service Worker caching (Workbox)
- Lazy loading resources
- Critical resource prioritization

### 17.3 JavaScript Performance
- Bundle size reduction techniques
- Code splitting: route-based, component-based
- Tree shaking — ensuring it works
- Dynamic imports for on-demand loading
- Polyfill optimization (only for browsers that need it)
- Script loading: async, defer, type=module
- Long task optimization
- Worker threads for CPU-intensive tasks
- WASM for performance-critical computations

### 17.4 CSS Performance
- Critical CSS extraction and inlining
- Avoiding layout-triggering properties in animations
- CSS containment (contain property)
- content-visibility: auto
- font-display optimization
- Reducing unused CSS
- CSS layers for cascade performance

### 17.5 Rendering Performance
- requestAnimationFrame for smooth animations
- Avoid layout thrashing (Fastdom pattern)
- Virtual DOM vs incremental DOM
- Angular's rendering optimization (Ivy)
- GPU-accelerated properties: transform, opacity
- will-change: transform (use carefully)
- Skeleton screens and perceived performance

### 17.6 Angular-Specific Performance
- OnPush change detection everywhere
- Signals for fine-grained reactivity
- trackBy in ngFor / track in @for
- @defer for deferred views
- NgOptimizedImage for image optimization
- Lazy loading routes and standalone components
- Preloading strategies (custom)
- Removing Zone.js (zoneless)
- Tree-shakeable providers (providedIn: 'root')
- Pure pipes instead of methods in templates
- Avoiding complex template expressions
- Detaching change detector for static content

### 17.7 Image & Asset Optimization
- Modern formats: WebP, AVIF
- Responsive images: srcset, sizes, picture
- Lazy loading images (loading="lazy")
- Angular's NgOptimizedImage directive
- Image CDN integration (Cloudinary, Imgix)
- SVG optimization (SVGO)
- Font subsetting
- Icon strategies: SVG sprite, icon font, inline SVG

### 17.8 Performance Measurement & Tools
- Chrome DevTools Performance panel
- Lighthouse CI in pipelines
- WebPageTest.org
- Core Web Vitals measurement: web-vitals.js
- Bundle analysis: source-map-explorer
- Network waterfall analysis
- Performance budgets in angular.json
- Real user monitoring (Datadog RUM, SpeedCurve)

### 17.9 Server-Side Rendering Performance
- Initial page load performance with SSR
- Hydration — reducing re-rendering
- Partial hydration (Angular 19+)
- Static generation for content pages
- Edge rendering

---

<a name="phase-18"></a>
## ♿ PHASE 18 — Accessibility (a11y)
*Duration: 2–3 weeks | Goal: Build inclusive applications*

### 18.1 Accessibility Fundamentals
- WCAG 2.1 / 2.2 guidelines: A, AA, AAA levels
- Four principles: Perceivable, Operable, Understandable, Robust (POUR)
- Why accessibility matters: legal, moral, business
- Section 508 (US) and EN 301 549 (EU) compliance
- Accessibility audits and reports

### 18.2 Semantic HTML for Accessibility
- Correct use of heading hierarchy
- Landmark elements: main, nav, header, footer, aside
- Button vs div (clickable behavior)
- Links vs buttons — when to use which
- Table accessibility: scope, caption, headers
- Form labels and association

### 18.3 ARIA (Accessible Rich Internet Applications)
- ARIA roles: landmark, widget, structure, window
- ARIA states and properties: aria-label, aria-labelledby, aria-describedby, aria-hidden, aria-expanded, aria-checked, aria-disabled
- aria-live regions: polite, assertive
- ARIA patterns for custom widgets
- When NOT to use ARIA ("no ARIA is better than bad ARIA")
- ARIA authoring practices guide (APG)

### 18.4 Keyboard Navigation
- Tab order and tabindex
- Keyboard traps (dialog focus management)
- Skip navigation links
- Focus indicators (visible focus)
- Keyboard shortcuts
- Common keyboard interactions: Enter, Space, Arrow keys, Escape

### 18.5 Screen Reader Testing
- Screen readers: NVDA (Windows), JAWS, VoiceOver (Mac/iOS), TalkBack (Android)
- Testing with NVDA + Firefox, JAWS + Chrome
- VoiceOver basics on Mac
- Reading order and focus order
- Announcements for dynamic content

### 18.6 Color & Visual Accessibility
- Color contrast ratios: 4.5:1 (normal), 3:1 (large text)
- Not relying solely on color to convey information
- Focus visible indicators
- Animations and prefers-reduced-motion
- Dark mode support (prefers-color-scheme)
- Text sizing (em/rem over px)
- Text spacing adjustments

### 18.7 Angular Accessibility
- Angular CDK a11y module overview: `A11yModule` and its exports
- `FocusTrap` — trap keyboard focus within a modal or overlay
- `FocusMonitor` — detect focus origin (mouse, keyboard, touch, program) on any element
- `InteractivityChecker` — determine if an element is focusable or tabbable
- `LiveAnnouncer` — programmatically announce messages to screen readers via ARIA live regions
- `AriaDescriber` — manage `aria-describedby` relationships without ID collisions
- Angular Material built-in accessibility: roles, labels, and keyboard patterns per component
- Router focus management — `RouterLink` and automatic focus on navigation
- Custom `TitleStrategy` for meaningful page titles announced on route change
- `@angular/cdk/a11y` `isFocusable()` / `isTabbable()` utilities

### 18.8 Angular ARIA Patterns
- Binding ARIA attributes dynamically: `[attr.aria-expanded]`, `[attr.aria-controls]`
- `[attr.aria-hidden]` on decorative elements and icon-only buttons
- `[attr.aria-label]` vs `[attr.aria-labelledby]` — when to use each in templates
- `[attr.aria-describedby]` for form field error messages (Angular forms integration)
- `[attr.aria-live]` and `[attr.aria-atomic]` for dynamic content regions
- `[attr.aria-busy]` during loading states (skeleton screens, async data)
- `[attr.role]` — assigning landmark and widget roles to host elements
- `[attr.aria-selected]`, `[attr.aria-checked]`, `[attr.aria-pressed]` for interactive state
- `[attr.aria-disabled]` vs the `disabled` attribute — differences in Angular contexts
- `@HostBinding('attr.aria-*')` — exposing ARIA state from directive/component class
- `host: { '[attr.aria-*]': '...' }` — declarative host bindings in component metadata
- `LiveAnnouncer.announce()` — triggering screen reader announcements for route changes, toasts, alerts
- Focus management after dialog open/close using `FocusTrap` + `afterOpened` / `afterClosed`
- Skip-link pattern implementation in Angular shell components

### 18.9 Accessibility Testing Tools
- axe DevTools browser extension
- Lighthouse accessibility audit
- WAVE (Web Accessibility Evaluation Tool)
- Deque axe-core for automated tests
- @axe-core/playwright, cypress-axe
- Manual keyboard testing checklist
- Screen reader testing checklist

---

<a name="phase-19"></a>
## 🔒 PHASE 19 — Security in Frontend
*Duration: 2–3 weeks | Goal: Build secure applications*

### 19.1 Web Security Fundamentals
- OWASP Top 10 — web application security risks
- Defense in depth principle
- Security-by-design approach
- Threat modeling basics

### 19.2 Cross-Site Scripting (XSS)
- Reflected XSS
- Stored XSS
- DOM-based XSS
- Prevention: output encoding, Angular's automatic sanitization
- Angular's DomSanitizer
- Content Security Policy (CSP) as defense
- Trusted Types API

### 19.3 Cross-Site Request Forgery (CSRF)
- How CSRF attacks work
- SameSite cookie attribute: Strict, Lax, None
- CSRF tokens (double submit cookie, synchronizer token)
- Angular HttpClient's withXsrfConfiguration()
- Custom CSRF header

### 19.4 Authentication & Authorization
- Authentication patterns: JWT, session cookies, OAuth 2.0, OIDC
- JWT: structure (header.payload.signature), storage (memory vs localStorage vs HttpOnly cookie)
- Refresh token patterns and rotation
- Angular route guards for authorization
- Role-based access control (RBAC) on frontend
- Permission-based UI rendering
- OAuth 2.0 flows: Authorization Code + PKCE for SPAs
- OIDC (OpenID Connect)
- Angular OAuth OIDC libraries (angular-oauth2-oidc)

### 19.5 Content Security Policy (CSP)
- CSP header directives: default-src, script-src, style-src, img-src, connect-src
- CSP nonces and hashes
- Report-only mode
- CSP with Angular (nonce support in Angular 16+)
- Strict CSP

### 19.6 Secure Communication
- HTTPS everywhere
- HSTS
- Certificate pinning (awareness)
- CORS configuration
- Subresource Integrity (SRI)

### 19.7 Data & Input Validation
- Never trust user input
- Client-side validation (UX) + server-side validation (security)
- Angular reactive form validators
- Sanitizing data before display
- Avoiding eval() and similar dynamic code execution

### 19.8 Dependency Security
- npm audit
- Snyk integration
- Dependabot alerts
- License scanning
- Software composition analysis (SCA)
- SBOM (Software Bill of Materials)

### 19.9 Angular-Specific Security
- Angular's automatic XSS protection (template binding)
- Avoiding bypassSecurityTrust methods unless necessary
- HTTP interceptor for security headers
- Secure storage of sensitive data
- Environment variables — what NOT to put in Angular environments

---

<a name="phase-20"></a>
## 📱 PHASE 20 — Progressive Web Apps (PWA)
*Duration: 2–3 weeks | Goal: Build installable, offline-capable apps*

### 20.1 PWA Fundamentals
- What is a PWA?
- PWA checklist (Google Lighthouse)
- Progressive enhancement approach
- HTTPS requirement
- Web App Manifest
- Service Workers
- Installation and home screen

### 20.2 Web App Manifest
- manifest.json configuration
- name, short_name, icons, start_url, display, theme_color, background_color
- Screenshots and description
- Shortcuts
- Categories
- install prompt (beforeinstallprompt event)

### 20.3 Service Workers
- Service Worker lifecycle: installing, waiting, activating
- Fetch event interception
- Cache strategies: Cache First, Network First, Stale-While-Revalidate, Network Only, Cache Only
- Cache versioning and cleanup
- Background Sync
- Push Notifications
- Periodic Background Sync

### 20.4 Workbox
- Workbox strategies
- Workbox precaching
- Runtime caching
- Workbox CLI and webpack plugin
- Angular's integration with Workbox

### 20.5 Angular PWA (@angular/pwa)
- ng add @angular/pwa
- ngsw-config.json: assetGroups, dataGroups
- SwUpdate service: version updates, reloading
- SwPush service: push notifications
- App Shell with Angular Universal
- Offline strategies for Angular routes
- Handling cache for API calls

### 20.6 Push Notifications
- Push API and Notifications API
- VAPID keys
- Permission request UX patterns
- Service Worker push event handler
- Notification options: title, body, icon, actions, badge, vibrate

### 20.7 PWA Performance & UX
- App Shell model
- Skeleton screens
- Offline fallback pages
- Network status detection (navigator.onLine)
- Handling poor connectivity gracefully
- Background data sync

---

<a name="phase-21"></a>
## 🧩 PHASE 21 — Micro Frontends
*Duration: 3–4 weeks | Goal: Architect large-scale frontend systems*

### 21.1 Micro Frontend Fundamentals
- What are micro frontends and why they exist
- Monolith → micro frontend evolution
- Benefits: independent deployments, team autonomy, technology flexibility
- Challenges: shared dependencies, performance overhead, coordination
- Vertical vs horizontal splits
- Domain-driven design applied to frontend

### 21.2 Micro Frontend Integration Approaches
- Iframe-based integration (simplest, most isolated)
- Web Components / Custom Elements approach
- JavaScript integration at runtime (Module Federation)
- Server-Side Integration (SSI — Server Side Includes)
- Build-time integration (npm packages)
- Edge-side composition

### 21.3 Webpack Module Federation
- What is Module Federation
- Host and remote applications
- Shared dependencies configuration
- Eager vs lazy shared modules
- Dynamic remotes
- Federated types for TypeScript
- Module Federation in Angular with @angular-architects/module-federation

### 21.4 Native Federation (Angular 17+)
- @angular-architects/native-federation
- ES Module-based federation (no Webpack required)
- Setup and configuration
- Working with esbuild-based Angular builder
- Comparing Native Federation vs Webpack Module Federation

### 21.5 Angular Elements (Web Components)
- @angular/elements package
- createCustomElement()
- Packaging Angular components as Custom Elements
- Shadow DOM with Angular Elements
- Using Angular Elements in non-Angular hosts
- Communication between Angular Elements

### 21.6 Shared Infrastructure
- Design system / component library as shared MFE
- Authentication across micro frontends
- Shared state management strategies
- Event bus for cross-MFE communication
- Routing coordination

### 21.7 Micro Frontend Orchestration
- Shell application (orchestrator/host)
- Dynamic remote loading
- Error boundaries for failing remotes
- Fallback strategies
- Lazy loading remotes
- Routing in micro frontends

### 21.8 Deployment & Operations
- Independent deployment pipelines
- Versioning strategies
- Rollback strategies
- CDN configuration for remotes
- Environment configuration per MFE
- Monitoring per micro frontend

---

<a name="phase-22"></a>
## 🎨 PHASE 22 — Design Systems & UI Architecture
*Duration: 2–3 weeks | Goal: Scale UI across teams*

### 22.1 Design System Fundamentals
- What is a design system?
- Design tokens: colors, typography, spacing, shadows, radii
- Component library vs design system
- Atomic design: atoms, molecules, organisms, templates, pages
- Design-development collaboration

### 22.2 Design Tokens
- CSS custom properties as tokens
- Token naming conventions (semantic vs raw)
- Multi-brand theming
- Dark mode implementation
- Token management tools: Style Dictionary, Theo
- JSON token format (W3C Design Tokens spec)

### 22.3 Angular Component Libraries
- Building reusable Angular component libraries
- Storybook for Angular: stories, args, decorators
- Storybook controls, actions, docs
- Chromatic for visual regression
- Testing stories with Interaction tests
- Publishing with ng-packagr
- Versioning and changelog

### 22.4 Theming Architecture
- Angular Material theming system (M2 vs M3)
- Multi-theme setup
- CSS variables-based theming
- User-selectable themes (stored preference)
- System-level dark mode detection

### 22.5 Component API Design
- Inputs: sensible defaults, required vs optional
- Outputs: meaningful event names
- Content projection: flexibility vs complexity
- Compound component patterns
- Configuration objects vs individual inputs
- Accessibility in component API
- Versioning and backward compatibility

### 22.6 Documentation
- Storybook as living documentation
- Compodoc for Angular documentation generation
- README-driven development
- ADR (Architecture Decision Records)
- API documentation standards

---

<a name="phase-23"></a>
## 🔌 PHASE 23 — API Design & Integration
*Duration: 2–3 weeks | Goal: Master backend integration*

### 23.1 REST API Design Principles
- Resource-based URLs
- HTTP methods and their semantics
- HTTP status codes — correct usage
- Request/Response formats
- HATEOAS
- Versioning strategies: URL, header, query param
- Pagination: offset, cursor-based, keyset
- Filtering, sorting, field selection

### 23.2 REST Integration in Angular
- HttpClient service
- Environment-based API URLs
- API client service pattern
- Typed responses with generics
- Error handling patterns: global interceptor, service-level
- Loading state management
- Retry and circuit breaker patterns
- API caching strategies

### 23.3 GraphQL
- GraphQL vs REST
- Queries, mutations, subscriptions
- Schema Definition Language (SDL)
- Variables and fragments
- Error handling in GraphQL
- Apollo Client for Angular
- Code generation (graphql-code-generator)
- Normalized cache
- Optimistic UI with GraphQL mutations
- Pagination: cursor-based (Relay spec)

### 23.4 Real-Time Communication
- WebSockets in Angular
- RxJS WebSocket: webSocket() creation operator
- Socket.io client in Angular
- Server-Sent Events (SSE) with Angular HttpClient (streaming)
- Long polling fallback
- Reconnect strategies

### 23.5 gRPC & Protocol Buffers
- gRPC overview and protobuf format
- gRPC-Web for browser compatibility
- Code generation from .proto files
- Usage in Angular applications (awareness level)

### 23.6 OpenAPI & API Client Generation
- OpenAPI (Swagger) specification
- Generating Angular services from OpenAPI: openapi-generator, swagger-codegen, ng-openapi-gen
- Type-safe API clients
- Mocking with Mirage.js or msw (Mock Service Worker)

### 23.7 Mock Service Worker (MSW)
- MSW setup for browser and Node.js
- Request handlers
- Response resolvers
- Integration with Angular testing
- Development mocking without backend
- API contract testing

---

<a name="phase-24"></a>
## 🚢 PHASE 24 — DevOps for Frontend
*Duration: 2–3 weeks | Goal: Own the deployment pipeline*

### 24.1 CI/CD Fundamentals
- Continuous Integration concepts
- Continuous Delivery vs Continuous Deployment
- Pipeline stages: build → test → lint → security scan → deploy
- Trunk-based development and CI
- Feature flags for safe deployments

### 24.2 GitHub Actions
- Workflow YAML syntax
- Triggers: push, pull_request, workflow_dispatch, schedule
- Jobs and steps
- Using marketplace actions
- Caching node_modules and build artifacts
- Environment secrets
- Matrix builds (multiple Node.js versions)
- Deployment jobs with environments
- OIDC for cloud authentication

### 24.3 Other CI Platforms
- GitLab CI/CD: .gitlab-ci.yml, stages, pipelines
- Azure Pipelines: YAML pipelines for Angular
- CircleCI, TeamCity (awareness)
- Jenkins (legacy awareness)

### 24.4 Containerization
- Docker fundamentals: images, containers, Dockerfile
- Multi-stage Docker builds for Angular
- Serving Angular with Nginx in Docker
- docker-compose for local development
- Container registries: Docker Hub, GHCR, ECR, ACR

### 24.5 Deployment Targets
- Static hosting: Netlify, Vercel, GitHub Pages, Firebase Hosting, Cloudflare Pages
- Object storage: AWS S3 + CloudFront
- Nginx configuration for SPA routing
- Azure Static Web Apps
- Node.js server for Angular Universal
- Kubernetes for containerized Angular apps

### 24.6 Environment Configuration
- Angular environments (environment.ts)
- Runtime configuration (app-config.json loaded at startup)
- Environment variable injection at build time
- Runtime injection via window.__env__ pattern
- Configuration for containers (environment variables)
- Secrets management

### 24.7 CDN Strategy
- CDN concepts: edge nodes, PoPs
- Cache-Control headers for Angular apps
- Cache busting with hashed file names
- CDN invalidation strategies
- Multi-region deployments

### 24.8 Infrastructure as Code
- Terraform for cloud infrastructure
- CloudFormation (AWS) awareness
- Bicep (Azure) awareness
- Pulumi awareness
- Frontend infrastructure: S3, CloudFront, Route53, ACM

---

<a name="phase-25"></a>
## 📊 PHASE 25 — Monitoring, Observability & Analytics
*Duration: 1–2 weeks | Goal: Understand production behavior*

### 25.1 Error Monitoring
- Sentry for Angular: ErrorHandler integration
- Source maps upload for production debugging
- Error grouping and fingerprinting
- Breadcrumbs and context
- Performance monitoring with Sentry
- Custom error handlers in Angular

### 25.2 Real User Monitoring (RUM)
- Web Vitals measurement in production
- web-vitals library integration
- Sending metrics to analytics
- Datadog RUM
- New Relic Browser
- SpeedCurve

### 25.3 Logging
- Console logging (avoid in production)
- Log levels: debug, info, warn, error
- Structured logging
- Log aggregation: ELK Stack, Datadog Logs, Splunk
- Correlation IDs for distributed tracing

### 25.4 Analytics
- Google Analytics 4 integration with Angular Router
- Tracking page views in SPA routing
- Custom event tracking
- User journey analysis
- Heatmaps: Hotjar, Microsoft Clarity
- Session recording
- Feature flag analytics

### 25.5 Performance Monitoring
- Synthetic monitoring with Lighthouse CI
- Real-user performance data collection
- Core Web Vitals dashboard
- Performance regression alerts
- Canary deployments with performance comparison

---

<a name="phase-26"></a>
## 📱 PHASE 26 — Cross-Platform & Mobile with Angular
*Duration: 2–3 weeks | Goal: Angular beyond the browser*

### 26.1 Responsive Web Design (advanced)
- Mobile-first CSS architecture
- Fluid layouts and container queries
- Touch events and mobile UX
- Virtual keyboard and viewport issues
- iOS Safari quirks

### 26.2 Ionic Framework
- Ionic + Angular setup
- Ionic components: IonButton, IonList, IonCard, IonTabs, IonModal
- Ionic routing with Angular Router
- Adaptive UI (iOS vs Material design)
- Capacitor: bridge to native APIs
- Cordova (legacy) vs Capacitor
- Native plugins: Camera, Geolocation, Push, Filesystem

### 26.3 NativeScript with Angular
- NativeScript + Angular architecture
- NativeScript UI components
- Accessing native APIs
- Comparing to Ionic/Capacitor

### 26.4 Electron with Angular
- Electron architecture: main process vs renderer process
- Angular + Electron setup
- IPC (Inter-Process Communication)
- Native Node.js APIs in Electron
- Packaging and distributing desktop apps
- Auto-updater

### 26.5 Angular Universal for SSR/SSG
- Node.js Express server for SSR
- Prerendering for SSG
- Angular Universal Transfer State
- Hydration

---

<a name="phase-27"></a>
## 🤖 PHASE 27 — AI/ML Integration in Frontend
*Duration: 2–3 weeks | Goal: Build AI-powered Angular apps*

### 27.1 AI Integration Fundamentals
- LLM APIs: OpenAI (GPT-4o, o3), Anthropic (Claude 3.5/4), Google (Gemini 2.0), Mistral
- REST API integration from Angular (HttpClient or httpResource)
- **Streaming responses** — SSE with Angular HttpClient (`text/event-stream`), RxJS-based streaming
- Token limits, context windows, and multi-turn conversations
- Prompt engineering: system prompts, few-shot examples, chain-of-thought
- Model selection: capability vs cost vs latency trade-offs
- **Structured outputs** — JSON mode and function calling for typed AI responses

### 27.2 AI-Powered UI Patterns
- Chatbot / assistant integration with Angular
- Streaming text display: character-by-character with RxJS `scan`
- AI-powered search: semantic search + vector retrieval
- Content generation UX: loading skeletons, streaming placeholders
- AI form filling and autosuggest patterns
- Rate limiting and retry patterns with exponential backoff
- Error handling for AI APIs (hallucinations, content filters, timeouts)
- Cost estimation UI (token counters)

### 27.3 AI SDK Integration
- **Vercel AI SDK** — framework-agnostic, streaming-first, works with Angular
- **LangChain.js** for agentic workflows in the browser/Node
- **LlamaIndex.TS** for RAG pipelines
- OpenAI Node.js SDK in Angular (careful with bundling)
- Edge-compatible AI integrations (lighter SDKs)

### 27.4 On-Device & Local AI
- **TensorFlow.js** in Angular — run models in the browser
- **ONNX Runtime Web** — broad model format support
- **WebAssembly ML models** (WASM SIMD acceleration)
- **Transformers.js (Hugging Face)** — run BERT, Whisper, LLaMA in-browser
- **MediaPipe** — vision tasks (hand tracking, face detection, pose estimation)
- **WebGPU** for GPU-accelerated ML inference in the browser
- Web Workers for off-main-thread inference — keep UI responsive

### 27.5 Vector Search & RAG (Retrieval-Augmented Generation)
- Embeddings concept — text → vector representation
- Vector databases: Pinecone, Weaviate, pgvector, Chroma
- RAG pipeline architecture — retrieve → augment prompt → generate
- Building knowledge-base chat applications with Angular frontend
- Hybrid search: keyword + vector

### 27.6 AI-Assisted Angular Development (2026)
- **GitHub Copilot** — code completion and chat in VS Code
- **Cursor IDE** — AI-native IDE for Angular development
- **v0 (Vercel)** — AI component generation (adaptable to Angular)
- **Angular Copilot / AI schematics** — AI-assisted `ng generate`
- AI code review (CodeRabbit, Sourcery)
- AI-assisted test generation (Copilot unit tests, Diffblue)
- Prompt patterns for generating Angular components, services, tests

---

<a name="phase-28"></a>
## 🏛️ PHASE 28 — Frontend Software Architecture
*Duration: 4–6 weeks | Goal: Design scalable frontend systems*

### 28.1 Architecture Fundamentals
- What is software architecture?
- Architecture drivers: quality attributes, constraints, stakeholders
- Trade-off analysis (ATAM)
- Architecture views: logical, deployment, process
- C4 Model for frontend architecture documentation
- ADR (Architecture Decision Records)

### 28.2 Angular Application Architecture
- Feature-based folder structure vs type-based
- LIFT principle: Locate, Identify, Flat, Try DRY
- Core / Feature / Shared module pattern (and standalone equivalent)
- Domain-Driven Design in Angular
- Layered architecture: presentation → domain → data/infrastructure
- Bounded contexts in frontend

### 28.3 Large-Scale Angular Architecture
- Nx workspace layering: app → feature → ui → data-access → util
- Tag-based module boundaries
- Dependency rules and enforcement (ESLint nx/enforce-module-boundaries)
- Monorepo vs polyrepo trade-offs
- Team topology and codebase organization
- Strangler fig pattern for incremental migration

### 28.4 Frontend Architecture Patterns
- Single Page Application (SPA) architecture
- Server-Side Rendering (SSR) architecture
- Static Site Generation (SSG) architecture
- Islands Architecture
- Micro Frontend Architecture
- Edge rendering architecture
- Hybrid architectures

### 28.5 Scalability Patterns
- Code splitting strategy (route-level, feature-level, component-level)
- Progressive loading patterns
- Lazy loading everything possible
- Bundle splitting and shared chunks strategy
- Worker-based architecture (Web Workers for CPU tasks)

### 28.6 Resilience Patterns
- Error boundaries (Angular ErrorHandler)
- Retry patterns with RxJS
- Circuit breaker pattern
- Graceful degradation
- Fallback UI patterns
- Offline-first architecture
- Optimistic UI patterns

### 28.7 API Layer Architecture
- API client abstraction layer
- Repository pattern for data access
- Adapter pattern for backend data transformation
- Data Transfer Objects (DTOs)
- Domain models vs API models
- Caching layer design

### 28.8 Cross-Cutting Concerns
- Authentication & authorization architecture
- Logging and monitoring strategy
- Internationalization architecture
- Theme and design token system
- Feature flags system design
- A/B testing infrastructure

### 28.9 Evolutionary Architecture
- Planning for change
- Architectural fitness functions
- Automated architecture compliance (ESLint rules, nx boundaries)
- Incremental migration strategies
- API versioning for frontend
- Long-term dependency management

### 28.10 Architecture Documentation
- Living architecture documentation
- C4 model diagrams
- Sequence diagrams for complex flows
- Data flow diagrams
- ADRs for key decisions
- README-driven architecture

### 28.11 Architecture Evaluation
- Analyzing existing architecture quality
- Performance architecture review
- Security architecture review
- Accessibility architecture review
- Technical debt identification and prioritization
- Architecture risk assessment

### 28.12 Technology Evaluation
- Framework selection criteria
- Build tool selection
- Library evaluation: community, maintenance, bundle size, API quality
- Make-or-buy decisions for UI components
- Vendor lock-in assessment
- Migration risk assessment

---

<a name="phase-29"></a>
## 👥 PHASE 29 — Leadership, Communication & Soft Skills
*Duration: Ongoing | Goal: Architect-level influence and effectiveness*

### 29.1 Technical Leadership
- Difference between senior dev and architect
- Influencing without authority
- Making architectural decisions with incomplete information
- Communicating trade-offs to stakeholders
- Technical vision and roadmap
- Guiding teams through technical challenges

### 29.2 Team Collaboration
- Effective code review: giving and receiving feedback
- Pair programming and mob programming
- Technical mentorship and coaching
- Onboarding new team members
- Documentation as team force multiplier
- Remote collaboration best practices

### 29.3 Communication Skills
- Writing clear technical proposals (RFC / design docs)
- Presenting architecture to non-technical stakeholders
- Writing postmortems and incident reports
- Technical blog writing
- Conference talks and knowledge sharing

### 29.4 Project & Delivery
- Estimating frontend complexity
- Breaking down large features into deliverable increments
- Managing technical debt alongside feature work
- Definition of Done for frontend work
- Working with product managers and designers
- Agile ceremonies from a technical perspective

### 29.5 Hiring & Interviews
- Writing frontend job descriptions
- Technical interview question design
- Conducting architecture discussions
- Evaluating candidate portfolios
- Building fair, inclusive hiring processes

### 29.6 Continuous Learning
- Staying updated: TC39 proposals, Angular blog, RFC process
- Contributing to open source
- Reading and writing technical articles
- Building side projects for experimentation
- Community participation (Angular Discord, Stack Overflow, GitHub)

---

<a name="appendix"></a>
## 📚 APPENDIX — Reference Guides & Resources

### A.1 Key Official Documentation
| Resource | URL |
|---|---|
| Angular Docs | angular.dev |
| Angular Blog | blog.angular.dev |
| TypeScript Handbook | typescriptlang.org/docs |
| TypeScript Playground | typescriptlang.org/play |
| MDN Web Docs | developer.mozilla.org |
| RxJS Docs | rxjs.dev |
| NgRx Docs | ngrx.io |
| Nx Docs | nx.dev |
| Analog Docs | analogjs.org |
| Web.dev | web.dev |
| TC39 Proposals | github.com/tc39/proposals |
| Can I Use | caniuse.com |
| web.dev Vitals | web.dev/vitals |
| Workbox | developers.google.com/web/tools/workbox |

### A.2 Essential Books
- *Angular Development with TypeScript* — Yakov Fain & Anton Moiseev
- *RxJS in Action* — Paul Daniels & Luis Atencio
- *You Don't Know JS* series — Kyle Simpson
- *Clean Code* — Robert C. Martin
- *Designing Data-Intensive Applications* — Martin Kleppmann
- *Building Micro-Frontends* — Luca Mezzalira
- *A Philosophy of Software Design* — John Ousterhout
- *The Pragmatic Programmer* — Hunt & Thomas
- *Accelerate* — Nicole Forsgren (DevOps research)

### A.3 Angular Version History (Key Milestones)
| Version | Year | Key Feature |
|---|---|---|
| AngularJS (1.x) | 2010 | Two-way binding, directives |
| Angular 2 | 2016 | TypeScript rewrite, components |
| Angular 4 | 2017 | Angular Universal, Animations |
| Angular 6 | 2018 | Angular Elements, CLI workspaces |
| Angular 8 | 2019 | Ivy preview, Differential loading |
| Angular 9 | 2020 | Ivy default, AOT default |
| Angular 12 | 2021 | View Engine removed |
| Angular 14 | 2022 | Standalone components, Typed forms |
| Angular 15 | 2022 | Standalone stable, Directive composition API |
| Angular 16 | 2023 | Signals (developer preview), Required inputs, Non-destructive hydration |
| Angular 17 | 2023 | @if/@for/@switch, @defer, esbuild builder, Signal inputs preview |
| Angular 18 | 2024 | Zoneless (experimental), Material M3, Stable Signals |
| Angular 19 | 2024 | Incremental hydration, Resource API, LinkedSignal, Event replay, httpResource preview |
| Angular 20 | 2025 | Zoneless stable (`provideZonelessChangeDetection`), Signal-based forms preview, improved SSR/hydration, Vitest integration |
| Angular 21 | 2025 | Partial hydration stable, httpResource stable, Signal Router preview, Rolldown build integration, Karma deprecated |

### A.4 TypeScript Version History (5.x)
| Version | Release | Key Feature |
|---|---|---|
| TS 5.0 | Mar 2023 | Decorators (TC39), const type params, verbatimModuleSyntax |
| TS 5.1 | May 2023 | Easier undefined return, unrelated getter/setter types |
| TS 5.2 | Aug 2023 | using/await using (Explicit Resource Management), decorator metadata |
| TS 5.3 | Nov 2023 | import attributes, switch(true) narrowing |
| TS 5.4 | Mar 2024 | NoInfer<T>, preserved narrowing in closures |
| TS 5.5 | Jun 2024 | Inferred type predicates, control flow for const indexed access |
| TS 5.6 | Sep 2024 | Disallowed nullish/truthy checks on non-nullable, Iterator helpers |
| TS 5.7 | Nov 2024 | Never-initialized variable checks, --rewriteRelativeImportExtensions |
| TS 5.8 | Feb 2025 | Granular conditional return checking, --erasableSyntaxOnly, require() of ESM |
| TS 5.9 | May 2025 | import defer (deferred module loading), improved isolated modules, tuple label improvements |

### A.4 Frontend Architecture Decision Checklist
- [ ] State management strategy defined
- [ ] Module/standalone boundaries documented
- [ ] Error handling strategy established
- [ ] Authentication flow designed
- [ ] Performance budget set
- [ ] Accessibility baseline defined (WCAG AA)
- [ ] Testing strategy documented
- [ ] CI/CD pipeline configured
- [ ] Monitoring and alerting set up
- [ ] Design token system established
- [ ] API integration layer abstracted
- [ ] Internationalization approach decided
- [ ] SSR/SSG/CSR strategy evaluated
- [ ] Feature flag system designed
- [ ] Code review process established

### A.5 Frontend Performance Checklist (2026)
- [ ] Core Web Vitals targets: LCP < 2.5s, INP < 200ms, CLS < 0.1
- [ ] Lazy loading all routes (`loadComponent`)
- [ ] `@defer` blocks for below-fold components
- [ ] Images using WebP/AVIF with `NgOptimizedImage` (`ngSrc`)
- [ ] `provideZonelessChangeDetection()` — Zone.js removed
- [ ] Signal-based components throughout the app
- [ ] `@for` with `track` on all lists
- [ ] `withFetch()` in `provideHttpClient()` for SSR compatibility
- [ ] Bundle size budgets set in angular.json
- [ ] Third-party scripts deferred/async
- [ ] Service Worker caching with Workbox
- [ ] CDN with long-term caching (hashed filenames)
- [ ] Font loading optimized (`font-display: swap`, `preconnect`)
- [ ] `withViewTransitions()` for smooth navigation
- [ ] `httpResource()` for HTTP data with built-in loading states

### A.6 Learning Order Summary (Quick Reference)
```
1. HTML → 2. CSS → 3. JavaScript → 4. TypeScript
    ↓
5. Browser Internals → 6. Git & Tooling
    ↓
7. Angular Core → 8. Angular Intermediate
    ↓
9. RxJS → 10. Angular Advanced → 11. Angular Ecosystem
    ↓
12. State Management → 13. Testing → 14. Performance
    ↓
15. Security → 16. a11y → 17. PWA
    ↓
18. Micro Frontends → 19. Design Systems → 20. DevOps
    ↓
21. Architecture → 22. Leadership
```

---

*Last updated: March 2026 | Angular 21 | TypeScript 5.9 | RxJS 7.x | Zoneless + Signals era*

> **Remember:** Mastery is not about knowing everything at once. It's about building a strong foundation and systematically expanding your expertise. Focus on depth over breadth in the early phases, then expand horizontally as an architect.

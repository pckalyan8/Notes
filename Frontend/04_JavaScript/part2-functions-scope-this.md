# ⚡ Phase 4 — Part 2: Functions · Scope & Hoisting · `this`
> Sections 4.4 · 4.5 · 4.6 | Angular Frontend Mastery Roadmap

---

## 4.4 Functions Deep Dive

Functions are first-class citizens in JavaScript, meaning they can be assigned to variables, passed as arguments, and returned from other functions. This property unlocks the entire world of functional programming patterns.

### Function Declarations vs Expressions vs Arrow Functions

There are three primary syntaxes for creating functions, and each has different hoisting behavior and `this` semantics.

**Function declarations** are hoisted fully — both the name and the body are moved to the top of their scope, so you can call a function declaration before it appears in the code.

```js
greet("Alice"); // Works! Declaration is hoisted
function greet(name) {
  console.log(`Hello, ${name}!`);
}
```

**Function expressions** assign an anonymous (or named) function to a variable. The variable binding follows normal hoisting rules — `var` is hoisted as `undefined`, `let`/`const` are in the temporal dead zone. Either way, the function is not available before the assignment line.

```js
// greet(); // TypeError if var, ReferenceError if const
const greet = function(name) {
  console.log(`Hello, ${name}!`);
};
greet("Alice"); // Works after the assignment
```

**Arrow functions** (ES2015) are the most concise syntax and differ crucially in how they handle `this` — they do not create their own `this` context, instead inheriting it from the surrounding lexical scope. This makes them ideal for callbacks and class methods.

```js
// Single parameter, implicit return
const double = x => x * 2;

// Multiple parameters
const add = (a, b) => a + b;

// Multiple statements — need braces and explicit return
const compute = (a, b) => {
  const sum = a + b;
  return sum * 2;
};

// Returning an object literal — wrap in parentheses to avoid
// being confused with a function body
const makeUser = name => ({ name, active: true });
```

> **Important distinction:** Arrow functions cannot be used as constructors, do not have their own `arguments` object, and cannot be used as generator functions. Their primary advantages are concise syntax and lexical `this`.

### Hoisting Behavior Summary

```js
// Function declaration — fully hoisted
sayHi(); // "Hi!" — works before the declaration
function sayHi() { console.log("Hi!"); }

// Function expression with var — variable hoisted as undefined
sayHello(); // TypeError: sayHello is not a function
var sayHello = function() { console.log("Hello!"); };

// Function expression with const — not accessible before declaration
sayBye(); // ReferenceError
const sayBye = () => console.log("Bye!");
```

### IIFE — Immediately Invoked Function Expression

An IIFE is a function that is defined and called immediately. Before ES modules existed, IIFEs were used to create private scopes and avoid polluting the global namespace. You will still encounter them in older codebases and minified bundles.

```js
(function() {
  const privateData = "secret";
  console.log(privateData); // accessible inside
})();

// privateData is not accessible here

// IIFE with arguments
const result = (function(a, b) {
  return a + b;
})(3, 5); // result = 8
```

### Default Parameters

Parameters can have default values if the caller does not provide them or passes `undefined`.

```js
function createUser(name, role = "viewer", active = true) {
  return { name, role, active };
}

createUser("Alice");                    // { name: "Alice", role: "viewer", active: true }
createUser("Bob", "admin");             // { name: "Bob", role: "admin", active: true }
createUser("Carol", undefined, false);  // role uses default, active is false
createUser("Dave", null, false);        // role is null — null does NOT trigger default
// Only undefined triggers the default value

// Defaults can reference earlier parameters
function repeat(str, times = str.length) {
  return str.repeat(times);
}
```

### First-Class Functions and Higher-Order Functions

Because functions are values in JavaScript, you can pass them as arguments (*callbacks*) or return them from other functions. Functions that operate on other functions are called *higher-order functions*. This is the foundation of array methods like `map`, `filter`, and `reduce`.

```js
// Function stored in a variable
const multiply = (a, b) => a * b;

// Function passed as an argument (callback)
function applyOperation(a, b, operation) {
  return operation(a, b);
}
applyOperation(4, 5, multiply); // 20
applyOperation(4, 5, (a, b) => a + b); // 9

// Function returned from a function (higher-order)
function createMultiplier(factor) {
  return (number) => number * factor; // returns a new function
}

const triple = createMultiplier(3);
triple(7); // 21
triple(10); // 30

// Real-world: array methods are higher-order functions
const prices = [10, 20, 30, 40];
const discounted = prices.map(price => price * 0.9); // [9, 18, 27, 36]
```

### Pure Functions and Side Effects

A *pure function* always returns the same output for the same input and has no *side effects* — it does not modify anything outside its own scope. Pure functions are predictable, testable, and composable.

A *side effect* is anything that affects the outside world: modifying a variable, mutating an argument, logging to the console, making a network call, or writing to the DOM.

```js
// Pure function — same input, same output, no side effects
function add(a, b) { return a + b; }
function formatName(first, last) { return `${first} ${last}`; }

// Impure — modifies external state
let count = 0;
function increment() { count++; } // side effect: changes outer variable

// Impure — mutates the argument
function addItem(arr, item) {
  arr.push(item); // mutates the original array!
  return arr;
}

// Pure version — creates a new array
function addItem(arr, item) {
  return [...arr, item]; // returns a new array, original unchanged
}
```

> **Best practice:** Strive for pure functions in your business logic. Angular services and RxJS operators work best when data flows through pure transformations. Reserve side effects for explicit boundary points (component lifecycle hooks, effects, interceptors).

### Function Composition

Composition means combining two or more functions so the output of one becomes the input of the next. This is a powerful pattern for building complex transformations from simple, tested pieces.

```js
const double = x => x * 2;
const addTen = x => x + 10;
const square = x => x * x;

// Manual composition
const result = square(addTen(double(5))); // double(5)=10, addTen(10)=20, square(20)=400

// Reusable compose utility (right-to-left like mathematical composition)
const compose = (...fns) => x => fns.reduceRight((v, f) => f(v), x);

const transform = compose(square, addTen, double);
transform(5); // 400

// pipe (left-to-right, more readable)
const pipe = (...fns) => x => fns.reduce((v, f) => f(v), x);
const process = pipe(double, addTen, square);
process(5); // 400
```

### Currying and Partial Application

*Currying* is transforming a function that takes multiple arguments into a series of functions that each take one argument. *Partial application* is pre-filling some arguments and returning a new function that takes the rest.

```js
// Curried function
const add = a => b => a + b;
const addFive = add(5); // returns a function waiting for b
addFive(3); // 8
addFive(10); // 15

// Practical currying — Angular-style
const createValidator = pattern => message => value =>
  new RegExp(pattern).test(value) ? null : { message };

const emailValidator = createValidator("^[^@]+@[^@]+$")("Invalid email");
emailValidator("user@example.com"); // null (valid)
emailValidator("notanemail");       // { message: "Invalid email" }
```

### Memoization

Memoization is caching the results of expensive function calls so they are not repeated for the same inputs.

```js
function memoize(fn) {
  const cache = new Map();
  return function(...args) {
    const key = JSON.stringify(args);
    if (cache.has(key)) {
      console.log("cache hit");
      return cache.get(key);
    }
    const result = fn.apply(this, args);
    cache.set(key, result);
    return result;
  };
}

const slowSquare = x => { /* expensive computation */ return x * x; };
const fastSquare = memoize(slowSquare);
fastSquare(10); // computes
fastSquare(10); // cache hit — instant
```

### Closures

A closure is a function that "remembers" the variables from its outer (enclosing) scope even after that scope has finished executing. This is one of the most fundamental and powerful concepts in JavaScript.

```js
function makeCounter(initialValue = 0) {
  // `count` is in the enclosing scope
  let count = initialValue;

  // The returned function "closes over" the `count` variable
  return {
    increment() { count++; },
    decrement() { count--; },
    value() { return count; }
  };
}

const counter = makeCounter(10);
counter.increment();
counter.increment();
counter.increment();
counter.decrement();
console.log(counter.value()); // 12

// Each call to makeCounter creates its own independent `count`
const counter2 = makeCounter(0);
counter2.increment();
console.log(counter2.value()); // 1
console.log(counter.value());  // 12 — not affected
```

The most important thing to understand about closures is that they close over *variables*, not *values*. This is the source of the classic `var` in a loop bug:

```js
// Bug — all callbacks share the same `i` variable
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 100);
}
// Prints: 3, 3, 3 — not 0, 1, 2!
// Because `var i` is a single shared variable, and by the time
// the callbacks run, the loop has finished and i === 3

// Fix 1 — use `let` (creates a new binding per iteration)
for (let i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 100);
}
// Prints: 0, 1, 2 ✓

// Fix 2 — IIFE to capture the value (old approach)
for (var i = 0; i < 3; i++) {
  (function(j) {
    setTimeout(() => console.log(j), 100);
  })(i);
}
```

### The Call Stack

The call stack is the mechanism JavaScript uses to track what function is currently executing and what to return to after it finishes. Each function call adds a *frame* to the stack. When the function returns, the frame is removed.

```js
function third() { console.log("third"); }
function second() { third(); console.log("second"); }
function first() { second(); console.log("first"); }

first();
// Call stack at deepest point: [first → second → third]
// Output order: "third", "second", "first"

// Stack overflow — infinite recursion without base case
function infinite() { infinite(); }
infinite(); // Maximum call stack size exceeded
```

---

## 4.5 Scope & Hoisting

### Scope Types

*Scope* determines where a variable is accessible. JavaScript has three levels of scope.

**Global scope** — Variables declared outside any function or block are in the global scope and accessible everywhere. In browsers, global variables become properties of `window`. Polluting global scope is a major anti-pattern.

**Function scope** — Variables declared with `var` inside a function are accessible anywhere within that function, including nested functions, but not outside.

**Block scope** — Variables declared with `let` or `const` inside a `{}` block (an `if`, `for`, or `while` block, or even a standalone `{}`) are only accessible within that block.

```js
let globalVar = "global";         // global scope

function outer() {
  let outerVar = "outer";         // function scope

  if (true) {
    let blockVar = "block";       // block scope
    var functionScoped = "fn";    // function scope (not block!) — avoid
    console.log(globalVar);       // accessible
    console.log(outerVar);        // accessible
    console.log(blockVar);        // accessible
  }

  console.log(globalVar);         // accessible
  console.log(outerVar);          // accessible
  console.log(functionScoped);    // accessible (var ignores block)
  // console.log(blockVar);       // ReferenceError — out of scope
}
```

### Lexical Scope

JavaScript uses *lexical* (also called *static*) scope. This means a function's scope is determined by where the function is *defined* in the source code, not by where it is *called*. This is what makes closures possible.

```js
let x = "global";

function outer() {
  let x = "outer";

  function inner() {
    // inner() was defined inside outer(), so it sees outer's x
    console.log(x); // "outer" — not "global", not "local"
  }

  inner(); // calling from here doesn't change which x it sees
}

outer();
inner(); // ReferenceError — inner is not in scope here
```

### The Scope Chain

When JavaScript looks up a variable name, it starts in the current scope, then moves outward through each enclosing scope until it reaches the global scope. This chain of scopes is the *scope chain*.

```js
const a = 1; // global

function outer() {
  const b = 2;

  function inner() {
    const c = 3;
    console.log(a, b, c); // 1, 2, 3 — found via scope chain
    // c found in inner, b found in outer, a found in global
  }

  inner();
}

outer();
```

### Hoisting in Detail

*Hoisting* is JavaScript's behavior of moving variable and function declarations to the top of their containing scope during the compilation phase, before code executes.

**`var` hoisting** — The variable name is hoisted and initialized to `undefined`. The assignment stays in place.

```js
console.log(name); // undefined — NOT a ReferenceError
var name = "Alice";
console.log(name); // "Alice"

// What the engine actually sees:
var name = undefined; // hoisted declaration
console.log(name);   // undefined
name = "Alice";      // assignment stays here
console.log(name);   // "Alice"
```

**Function declaration hoisting** — Both the name and the function body are hoisted. This is why you can call function declarations before they appear.

```js
greet(); // "Hello!" — works!
function greet() { console.log("Hello!"); }
```

**`let` and `const` hoisting** — These are technically hoisted (the names are registered) but they are placed in the *Temporal Dead Zone (TDZ)* and cannot be accessed before the declaration line. Attempting to access them throws a ReferenceError, which is much more helpful than `var`'s silent `undefined`.

```js
console.log(name); // ReferenceError: Cannot access 'name' before initialization
let name = "Alice";
```

### Temporal Dead Zone (TDZ)

The TDZ is the period between the start of a block and the `let`/`const` declaration. Any access to the variable in this period throws a ReferenceError. The TDZ exists to prevent the confusing behavior that `var` exhibits.

```js
{
  // TDZ for `value` starts here
  console.log(value); // ReferenceError!
  let value = 42;     // TDZ ends here
  console.log(value); // 42
}
```

---

## 4.6 The `this` Keyword

`this` is one of the most confusing concepts in JavaScript because its value depends entirely on *how a function is called*, not where it is defined. Understanding `this` is essential for working with Angular components, services, and event handlers.

### `this` in Global Context

In non-strict mode in a browser, `this` in the global context refers to the `window` object. In strict mode or Node.js, it is `undefined` or the module object respectively.

```js
// Browser (non-strict mode)
console.log(this); // window

// A regular function called without an object context
function showThis() {
  console.log(this); // window (non-strict) or undefined (strict)
}
showThis();
```

### `this` in Object Methods

When a function is called as a method of an object, `this` refers to the object the method is called on.

```js
const user = {
  name: "Alice",
  greet() {
    console.log(`Hello, I'm ${this.name}`);
  }
};

user.greet(); // "Hello, I'm Alice" — this === user
```

However, this changes when the method is extracted from its object:

```js
const greet = user.greet;
greet(); // "Hello, I'm undefined" — this is no longer user!
// The function is called without an object context, so this === window
```

### `this` in Arrow Functions — Lexical `this`

Arrow functions do **not** have their own `this`. Instead, they inherit `this` from their *enclosing lexical scope* at the time they are defined. This is by far the most important characteristic of arrow functions.

```js
const timer = {
  seconds: 0,

  // Regular function — `this` depends on how it's called
  startWithRegular() {
    setInterval(function() {
      this.seconds++; // Bug! `this` is window, not timer
    }, 1000);
  },

  // Arrow function — `this` is lexically bound to timer
  startWithArrow() {
    setInterval(() => {
      this.seconds++; // Correct! `this` is timer
    }, 1000);
  }
};
```

This is why Angular component class methods that are passed as callbacks should be arrow functions or use `.bind()`.

### Explicit Binding — `call()`, `apply()`, `bind()`

You can explicitly set what `this` should be using these three methods.

**`call()`** invokes the function immediately with `this` set to the first argument, followed by individual arguments.

**`apply()`** is identical to `call()` but takes the function arguments as an array. Useful when you have arguments already in an array.

**`bind()`** returns a *new function* with `this` permanently bound to the provided value. It does not call the function immediately.

```js
function introduce(greeting, punctuation) {
  console.log(`${greeting}, I'm ${this.name}${punctuation}`);
}

const alice = { name: "Alice" };
const bob = { name: "Bob" };

introduce.call(alice, "Hello", "!");   // "Hello, I'm Alice!"
introduce.apply(bob, ["Hi", "."]);     // "Hi, I'm Bob."

const aliceIntro = introduce.bind(alice, "Hey");
aliceIntro("?");  // "Hey, I'm Alice?" — this is permanently alice
aliceIntro("!!");  // "Hey, I'm Alice!!"

// bind is commonly used in class constructors (pre-arrow-function era)
class Timer {
  constructor() {
    this.tick = this.tick.bind(this); // permanently bind tick to this instance
  }
  tick() {
    console.log(this.seconds);
  }
}
```

### `this` in Classes

Inside a class method, `this` refers to the instance of the class. However, if you pass the method as a callback (e.g., to an event listener), you lose the binding.

```js
class Button {
  constructor(label) {
    this.label = label;
    this.handleClick = this.handleClick.bind(this); // fix binding
    // OR use class field with arrow function:
    // handleClick = () => { console.log(this.label); };
  }

  handleClick() {
    console.log(this.label);
  }

  attachToDOM(buttonEl) {
    // Without bind or arrow, `this` would be the button element
    buttonEl.addEventListener("click", this.handleClick);
  }
}
```

> **Angular note:** Angular component methods used as template event handlers like `(click)="handleClick()"` call the method on the component instance, so `this` is always correct. But if you ever pass a method as a plain callback to an outside function, you need `bind()` or an arrow function.

### `this` in Event Handlers

In traditional DOM event listeners using `function`, `this` is set to the element that the event was attached to.

```js
const button = document.querySelector("button");

button.addEventListener("click", function() {
  console.log(this); // the <button> element
});

button.addEventListener("click", () => {
  console.log(this); // NOT the button! Arrow functions don't rebind this
  // this is whatever it was in the enclosing scope
});
```

### Summary — `this` Resolution Rules

The rules for determining `this`, in priority order:

1. **`new` binding** — When called with `new`, `this` is the newly created object.
2. **Explicit binding** — When using `.call()`, `.apply()`, or `.bind()`, `this` is the provided value.
3. **Implicit binding** — When called as an object method (`obj.fn()`), `this` is the object.
4. **Default binding** — In all other cases, `this` is `undefined` (strict mode) or `window` (non-strict).
5. **Arrow function exception** — Arrow functions ignore all of the above and use the lexical `this` from where they were defined.

```js
// 1. new binding
function Person(name) { this.name = name; }
const p = new Person("Alice"); // this === p

// 2. explicit
fn.call(obj); // this === obj

// 3. implicit
obj.fn(); // this === obj

// 4. default
fn(); // this === undefined (strict) or window

// 5. arrow (overrides all)
const arrow = () => console.log(this); // lexical this
```

---

## 🔑 Key Takeaways — Part 2

The difference between function declarations, expressions, and arrow functions is not just syntactic — it affects hoisting and `this` binding. Always use `const` arrow functions for callbacks and class fields to avoid `this` confusion. Closures are the mechanism by which Angular services encapsulate state and by which RxJS observables maintain their internal behavior. The `var` hoisting and TDZ behaviors explain why modern code always uses `let` and `const`. The five-rule hierarchy for `this` is worth memorising — it explains nearly every confusing `this` bug you will encounter in real Angular code.

---

*Part 2 of 6 — Phase 4 JavaScript Mastery | Next: [Objects, Classes & Arrays](./part3-objects-classes-arrays.md)*

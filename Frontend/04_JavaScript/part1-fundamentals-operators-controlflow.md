# ⚡ Phase 4 — Part 1: JS Fundamentals · Operators · Control Flow
> Sections 4.1 · 4.2 · 4.3 | Angular Frontend Mastery Roadmap

---

## 4.1 JavaScript Fundamentals

### Variables — `var`, `let`, `const`

Variables are named containers for values. JavaScript has three ways to declare them, and choosing the right one matters for predictable, bug-free code.

**`var`** is the legacy way. It is *function-scoped*, not block-scoped, which means a `var` declared inside an `if` block is visible outside that block. It is also *hoisted* (explained in section 4.5), which leads to confusing bugs. Avoid `var` in all modern code.

```js
function exampleVar() {
  if (true) {
    var message = "hello"; // declared inside a block...
  }
  console.log(message); // ...but accessible here! "hello"
  // This surprises most developers expecting block-scoping.
}
```

**`let`** is block-scoped. A `let` variable lives only within the `{}` block in which it is declared. Use `let` when the variable's value will change over time (e.g., a loop counter, an accumulator).

```js
function exampleLet() {
  if (true) {
    let message = "hello";
  }
  console.log(message); // ReferenceError: message is not defined
  // Correct! It's confined to the if-block.
}
```

**`const`** is also block-scoped but additionally prevents *reassignment* — the binding cannot point to a different value. Note that `const` does **not** make the value itself immutable. An object or array declared with `const` can still have its contents modified.

```js
const name = "Alice";
name = "Bob"; // TypeError: Assignment to constant variable

const user = { name: "Alice" };
user.name = "Bob"; // This is fine — you're mutating the object, not reassigning the variable
user = {};         // TypeError — this IS reassignment

const numbers = [1, 2, 3];
numbers.push(4);   // Fine — mutating the array
numbers = [];      // TypeError — reassignment forbidden
```

> **Best practice:** Default to `const` for everything. Only switch to `let` when you know the binding must change. Never use `var`.

---

### Primitive Types

JavaScript has seven primitive types. Primitives are immutable — operations on them always return new values rather than modifying the original.

**`string`** — Represents text. Can be created with single quotes, double quotes, or template literals (backticks). Template literals support interpolation and multiline content.

```js
const single = 'Hello';
const double = "World";
const template = `${single}, ${double}!`; // "Hello, World!"
const multiline = `Line one
Line two`;
```

**`number`** — JavaScript uses IEEE 754 double-precision floating-point for all numbers, both integers and decimals. This leads to the famous floating-point quirk:

```js
console.log(0.1 + 0.2); // 0.30000000000000004 — not 0.3!
// Use toFixed or libraries like decimal.js for financial calculations.

console.log(Number.MAX_SAFE_INTEGER); // 9007199254740991
console.log(Number.isInteger(4.0));   // true
console.log(Number.isNaN(NaN));       // true — always use isNaN() not ==
console.log(isFinite(Infinity));      // false
```

**`bigint`** — For integers larger than `Number.MAX_SAFE_INTEGER`. Append `n` to a number literal.

```js
const huge = 9007199254740991n + 1n; // 9007199254740992n — precise!
```

**`boolean`** — Either `true` or `false`. The backbone of all conditional logic.

**`undefined`** — The default value of a variable that has been declared but not assigned, or a function parameter that was not provided, or a property that does not exist on an object.

**`null`** — Explicitly represents "no value" or "empty." It must be intentionally assigned. The `typeof null` quirk is a famous JavaScript bug — it returns `"object"` instead of `"null"`.

```js
let x;
console.log(x);        // undefined — declared but not assigned
console.log(typeof x); // "undefined"

let y = null;
console.log(y);        // null — intentionally empty
console.log(typeof y); // "object" — this is a historical bug in JS
```

**`symbol`** — A unique, immutable primitive used as object property keys to avoid naming collisions, especially in libraries.

```js
const id1 = Symbol("id");
const id2 = Symbol("id");
console.log(id1 === id2); // false — every Symbol is unique
```

---

### Reference Types

Objects, arrays, and functions are *reference types*. When you assign a reference type to a variable, the variable holds a *reference* (a memory address), not the actual data. This is why two variables can point to the same object.

```js
// Primitives are copied by value
let a = 5;
let b = a; // b gets a copy of the value 5
b = 10;
console.log(a); // 5 — unchanged

// Objects are copied by reference
let obj1 = { name: "Alice" };
let obj2 = obj1; // obj2 holds the same reference as obj1
obj2.name = "Bob";
console.log(obj1.name); // "Bob" — BOTH variables see the change!
```

This distinction is critical when you pass objects to functions or compare them.

```js
// Equality compares references, not content
const a = { x: 1 };
const b = { x: 1 };
console.log(a === b); // false — different objects in memory
console.log(a === a); // true — same reference
```

---

### Type Coercion and Conversion

JavaScript will automatically convert types in certain contexts — this is called *implicit coercion*. It is a frequent source of bugs for developers who do not understand it.

```js
// == uses coercion; === does not
console.log(1 == "1");   // true — string "1" coerced to number
console.log(1 === "1");  // false — different types, no coercion
console.log(null == undefined);  // true — special case
console.log(null === undefined); // false

// Arithmetic coercion
console.log("5" - 2);  // 3 — string coerced to number
console.log("5" + 2);  // "52" — number coerced to string (concatenation!)
console.log(true + 1); // 2 — true coerces to 1
console.log(false + 1);// 1 — false coerces to 0
```

*Explicit conversion* is always safer and shows intent:

```js
Number("42");     // 42
Number("hello");  // NaN
parseInt("42px"); // 42 — stops at non-numeric
parseFloat("3.14rem"); // 3.14
String(42);       // "42"
Boolean(0);       // false
Boolean("hello"); // true
```

---

### `typeof`, `instanceof`, `Object.prototype.toString`

`typeof` returns a string indicating the type. It works well for primitives but has limitations for objects:

```js
typeof "hello"    // "string"
typeof 42         // "number"
typeof true       // "boolean"
typeof undefined  // "undefined"
typeof Symbol()   // "symbol"
typeof 42n        // "bigint"
typeof null       // "object" — the famous bug!
typeof {}         // "object"
typeof []         // "object" — arrays are objects!
typeof function(){} // "function"
```

`instanceof` checks if an object was created by a specific constructor (follows the prototype chain):

```js
[] instanceof Array    // true
[] instanceof Object   // true — arrays are objects
"hello" instanceof String // false — primitives are not instances
```

`Object.prototype.toString.call()` is the most reliable way to get a detailed type string:

```js
Object.prototype.toString.call([])        // "[object Array]"
Object.prototype.toString.call(null)      // "[object Null]"
Object.prototype.toString.call(new Date)  // "[object Date]"
Object.prototype.toString.call(/regex/)   // "[object RegExp]"
```

---

### Truthy and Falsy Values

Every value in JavaScript can be evaluated in a boolean context. Values that coerce to `false` are called *falsy*. Everything else is *truthy*.

The complete list of falsy values is: `false`, `0`, `-0`, `0n` (BigInt zero), `""` (empty string), `null`, `undefined`, and `NaN`.

```js
// These all take the "else" branch:
if (0) { ... }
if ("") { ... }
if (null) { ... }
if (undefined) { ... }
if (NaN) { ... }

// Common gotcha — these are all truthy:
if ("0") { ... }  // truthy! Non-empty string
if ([]) { ... }   // truthy! Empty array
if ({}) { ... }   // truthy! Empty object
```

---

### Short-Circuit Evaluation

Logical operators `&&` and `||` do not always return a boolean — they return one of the operands. This is called short-circuit evaluation.

`||` returns the first *truthy* value it finds, or the last value if none are truthy.
`&&` returns the first *falsy* value it finds, or the last value if all are truthy.

```js
// || is used for default values
const name = userInput || "Anonymous";
// If userInput is "" or null or undefined, "Anonymous" is used

// && is used for conditional execution
const user = getUser();
user && console.log(user.name); // only logs if user is not null/undefined

// Practical Angular example — conditional rendering logic
const displayName = user && user.profile && user.profile.displayName || "Guest";
```

The **nullish coalescing operator** `??` is a safer alternative to `||` for defaults, because it only falls back when the value is *specifically* `null` or `undefined`, not when it is `0` or `""`.

```js
const count = 0;
console.log(count || 10);  // 10 — wrong if 0 is a valid value
console.log(count ?? 10);  // 0 — correct! 0 is not null/undefined
```

---

## 4.2 Operators & Expressions

### Arithmetic Operators

Standard arithmetic operators work as expected, with a few JavaScript-specific notes.

```js
5 + 3   // 8
5 - 3   // 2
5 * 3   // 15
5 / 3   // 1.6666... (always floating point)
5 % 3   // 2 (modulo / remainder)
5 ** 3  // 125 (exponentiation — ES2016)
-5      // unary negation

// Division always returns float in JS
10 / 3  // 3.333... — use Math.floor(10/3) for integer division
```

### Assignment Operators

```js
let x = 10;
x += 5;  // x = x + 5 → 15
x -= 3;  // x = x - 3 → 12
x *= 2;  // x = x * 2 → 24
x /= 4;  // x = x / 4 → 6
x **= 2; // x = x ** 2 → 36
x %= 5;  // x = x % 5 → 1

// Logical assignment (ES2021)
let a = null;
a ??= "default"; // a = "default" (only assigns if null/undefined)

let b = false;
b ||= "fallback"; // b = "fallback" (assigns if falsy)

let c = "keep";
c &&= "update";  // c = "update" (assigns only if truthy)
```

### Comparison Operators

Always prefer strict equality (`===` / `!==`) over loose equality (`==` / `!=`). Loose equality applies type coercion rules that are hard to predict.

```js
5 === 5    // true
5 === "5"  // false — different types
5 == "5"   // true — string coerced to number (avoid!)

// Comparison operators
5 > 3     // true
5 >= 5    // true
3 < 5     // true
3 <= 3    // true

// NaN is never equal to anything, including itself
NaN === NaN  // false!
Number.isNaN(NaN) // true — correct way to check
```

### Logical Operators

```js
// AND — true if both operands are truthy
true && true   // true
true && false  // false

// OR — true if at least one operand is truthy
false || true  // true
false || false // false

// NOT — inverts truthiness
!true   // false
!false  // true
!!value // double negation — converts any value to a true boolean
```

### Nullish Coalescing (`??`) and Optional Chaining (`?.`)

These two operators work beautifully together for safely navigating data that may be partially undefined — extremely common when working with API responses in Angular.

```js
const user = {
  profile: {
    address: null
  }
};

// Without optional chaining — crashes if any part is null/undefined
const city = user.profile.address.city; // TypeError!

// With optional chaining — returns undefined safely
const city = user.profile?.address?.city; // undefined — no crash

// Combined with nullish coalescing for a default
const displayCity = user.profile?.address?.city ?? "Unknown";

// Optional chaining also works with methods and array indices
const first = arr?.[0];              // safe array access
const result = obj?.method?.();      // safe method call
```

### Spread Operator (`...`) and Rest Parameters

The spread operator *expands* an iterable into individual elements. Rest parameters *collect* individual elements into an array.

```js
// Spread — expanding
const arr1 = [1, 2, 3];
const arr2 = [4, 5, 6];
const combined = [...arr1, ...arr2]; // [1, 2, 3, 4, 5, 6]

// Spread — copying an array (shallow)
const copy = [...arr1]; // [1, 2, 3] — a new array

// Spread — copying an object (shallow)
const original = { a: 1, b: 2 };
const clone = { ...original, b: 99, c: 3 }; // { a: 1, b: 99, c: 3 }
// Properties listed later override earlier ones

// Spread with function arguments
function sum(x, y, z) { return x + y + z; }
const nums = [1, 2, 3];
sum(...nums); // 6

// Rest parameters — collecting
function logAll(first, second, ...rest) {
  console.log(first);  // "a"
  console.log(second); // "b"
  console.log(rest);   // ["c", "d", "e"]
}
logAll("a", "b", "c", "d", "e");
```

### Ternary Operator

A compact inline if-else. Best for simple conditional expressions, not for complex logic.

```js
const age = 20;
const status = age >= 18 ? "adult" : "minor"; // "adult"

// Good use — simple value selection
const label = isLoading ? "Loading..." : "Submit";

// Bad use — complex nested ternaries become unreadable
const result = a ? b ? "both" : "only a" : c ? "only c" : "neither";
// Prefer if/else for complex logic
```

### Comma Operator

Evaluates multiple expressions from left to right and returns the value of the last one. Rarely needed, but you'll see it in minified code and `for` loop initializations.

```js
let x = (1, 2, 3); // x = 3 — all evaluated, last value returned
for (let i = 0, j = 10; i < 5; i++, j--) { ... } // common valid use
```

---

## 4.3 Control Flow

### `if` / `else if` / `else`

The most fundamental branching construct. Evaluates a condition and executes different code paths.

```js
function classify(score) {
  if (score >= 90) {
    return "A";
  } else if (score >= 80) {
    return "B";
  } else if (score >= 70) {
    return "C";
  } else {
    return "F";
  }
}
```

> **Best practice:** Use early returns to reduce nesting. The "guard clause" pattern makes code easier to read by eliminating the `else` when the `if` block returns.

```js
// Deeply nested — hard to follow
function process(user) {
  if (user) {
    if (user.isActive) {
      if (user.hasPermission) {
        doWork(user);
      }
    }
  }
}

// Guard clauses — flat and clear
function process(user) {
  if (!user) return;
  if (!user.isActive) return;
  if (!user.hasPermission) return;
  doWork(user);
}
```

### `switch`

Switch is ideal when you have many discrete cases to check against a single value. Uses strict equality internally.

```js
function getDayName(day) {
  switch (day) {
    case 0: return "Sunday";
    case 1: return "Monday";
    case 2: return "Tuesday";
    case 3: return "Wednesday";
    case 4: return "Thursday";
    case 5: return "Friday";
    case 6: return "Saturday";
    default: return "Unknown";
  }
}

// Fallthrough — intentional when multiple cases share logic
function getCategory(status) {
  switch (status) {
    case "pending":
    case "processing":  // falls through to "active"
      return "active";
    case "complete":
    case "archived":
      return "inactive";
    default:
      return "unknown";
  }
}
```

> **Important:** Always include `break` or `return` in each case unless intentional fallthrough. Forgetting `break` is a common bug.

### Loops

**`for` loop** — ideal when you know the number of iterations upfront.

```js
for (let i = 0; i < 5; i++) {
  console.log(i); // 0, 1, 2, 3, 4
}

// Looping backwards
for (let i = arr.length - 1; i >= 0; i--) {
  console.log(arr[i]);
}
```

**`while` loop** — runs as long as a condition is true. Use when the number of iterations is not known in advance.

```js
let attempts = 0;
while (attempts < 3) {
  const success = tryOperation();
  if (success) break;
  attempts++;
}
```

**`do...while` loop** — always executes the body at least once before checking the condition.

```js
let input;
do {
  input = prompt("Enter a number greater than 0:");
} while (Number(input) <= 0);
```

**`for...of` loop** — iterates over the *values* of any iterable (arrays, strings, Maps, Sets, NodeLists, generators). This is the preferred way to iterate collections in modern JavaScript.

```js
const fruits = ["apple", "banana", "cherry"];
for (const fruit of fruits) {
  console.log(fruit); // "apple", "banana", "cherry"
}

// Works on strings
for (const char of "hello") {
  console.log(char); // "h", "e", "l", "l", "o"
}

// Works on Maps
const map = new Map([["a", 1], ["b", 2]]);
for (const [key, value] of map) {
  console.log(key, value);
}
```

**`for...in` loop** — iterates over the *keys* (property names) of an object. Generally avoid using this on arrays because it can iterate inherited properties and does not guarantee order.

```js
const person = { name: "Alice", age: 30, city: "London" };
for (const key in person) {
  console.log(key, person[key]);
}
// Prefer Object.keys(person).forEach() for objects
// Prefer for...of for arrays
```

### `break`, `continue`, and Labeled Statements

`break` exits the nearest enclosing loop or switch statement immediately.
`continue` skips the rest of the current loop iteration and moves to the next one.

```js
// break — exit early
for (let i = 0; i < 10; i++) {
  if (i === 5) break; // stops the loop at 5
  console.log(i); // 0, 1, 2, 3, 4
}

// continue — skip an iteration
for (let i = 0; i < 10; i++) {
  if (i % 2 === 0) continue; // skip even numbers
  console.log(i); // 1, 3, 5, 7, 9
}
```

Labeled statements allow `break` and `continue` to target outer loops. Rare in practice, but useful for nested loop escapes.

```js
outer: for (let i = 0; i < 3; i++) {
  for (let j = 0; j < 3; j++) {
    if (i === 1 && j === 1) break outer; // breaks the OUTER loop
    console.log(i, j);
  }
}
```

### Iterators and the Iteration Protocol

The iteration protocol is the contract that makes `for...of` work on any data structure. An object is *iterable* if it has a `Symbol.iterator` method that returns an *iterator* — an object with a `next()` method that returns `{ value, done }`.

```js
// Custom iterable — a range from start to end
function range(start, end) {
  return {
    // This method makes the object iterable
    [Symbol.iterator]() {
      let current = start;
      return {
        // The iterator's next() method
        next() {
          if (current <= end) {
            return { value: current++, done: false };
          }
          return { value: undefined, done: true };
        }
      };
    }
  };
}

for (const n of range(1, 5)) {
  console.log(n); // 1, 2, 3, 4, 5
}

// Spread and destructuring also use the iteration protocol
const arr = [...range(1, 3)]; // [1, 2, 3]
```

### Generator Functions (`function*`)

Generators are functions that can *pause* their execution and *resume* later. They return an iterator automatically. The `yield` keyword pauses the function and produces a value.

```js
function* countUp() {
  yield 1;
  yield 2;
  yield 3;
}

const gen = countUp();
console.log(gen.next()); // { value: 1, done: false }
console.log(gen.next()); // { value: 2, done: false }
console.log(gen.next()); // { value: 3, done: false }
console.log(gen.next()); // { value: undefined, done: true }

// Generators are iterable
for (const n of countUp()) {
  console.log(n); // 1, 2, 3
}

// Practical use — infinite sequence (taken lazily)
function* naturals() {
  let n = 1;
  while (true) {
    yield n++;
  }
}

function take(n, iterable) {
  const result = [];
  for (const val of iterable) {
    result.push(val);
    if (result.length === n) break;
  }
  return result;
}

take(5, naturals()); // [1, 2, 3, 4, 5]
```

Generators are the foundation that powers async generators and some RxJS internals. You will encounter them in the context of Redux Saga and advanced RxJS usage.

---

## 🔑 Key Takeaways — Part 1

Use `const` by default and `let` only when reassignment is needed — never `var`. Understanding the difference between primitive (value) types and reference types prevents a whole class of bugs, particularly when passing objects to functions or comparing them. Type coercion via `==` is unpredictable, so always use `===`. Master the truthy/falsy system and short-circuit evaluation because Angular templates and component logic rely on them constantly. The `?.` and `??` operators are essential for safely navigating nested API data. Choose `for...of` for collections — it is the modern default — and understand how the iteration protocol enables this pattern.

---

*Part 1 of 6 — Phase 4 JavaScript Mastery | Next: [Functions, Scope & `this`](./part2-functions-scope-this.md)*

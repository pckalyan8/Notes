# 🔷 Phase 5 — Part 1: TypeScript Fundamentals · Basic Types · Complex Types
> Sections 5.1 · 5.2 · 5.3 | Angular Frontend Mastery Roadmap

---

## 5.1 TypeScript Fundamentals

### Why TypeScript Exists — The Problem It Solves

JavaScript is dynamically typed, which means a variable can hold any kind of value at any time, and the language never complains about it. This freedom is wonderful for small scripts, but it becomes a serious liability in large applications where dozens of developers work across hundreds of files. Consider this perfectly valid JavaScript:

```js
// JavaScript — no errors, even though this is clearly broken
function calculateTotal(items) {
  return items.reduce((sum, item) => sum + item.price, 0);
}

calculateTotal("hello");        // No error — returns "hello" but then crashes
calculateTotal([{ cost: 10 }]); // No error at definition time — item.price is undefined
```

Every one of those calls is silently broken, and JavaScript will not tell you until the code actually runs — possibly in production, possibly days after you wrote it. TypeScript's solution is to add a **static type system**: a layer of analysis that runs before your code executes and checks that every value is being used in a way consistent with its declared type.

```ts
// TypeScript — bugs caught immediately, before running
interface CartItem {
  name: string;
  price: number;
}

function calculateTotal(items: CartItem[]): number {
  return items.reduce((sum, item) => sum + item.price, 0);
}

calculateTotal("hello");         // Error: Argument of type 'string' is not assignable to 'CartItem[]'
calculateTotal([{ cost: 10 }]);  // Error: Object literal may only specify known properties; 'cost' does not exist in type 'CartItem'
```

The two incorrect calls are caught instantly in the editor, with a red underline and a message explaining exactly what is wrong. This is the core value proposition of TypeScript: **moving error detection from runtime to compile time.**

Beyond catching bugs, TypeScript dramatically improves the developer experience. Because the compiler understands the shape of every value, your editor can offer highly accurate autocomplete suggestions, rename refactoring that works across every file, and jump-to-definition navigation that actually works. In a large Angular codebase, these capabilities are not a luxury — they are what make the codebase navigable.

---

### The TypeScript Compiler and `tsc`

TypeScript ships with a command-line compiler called `tsc`. Its job is to take your `.ts` source files, run type checks, and output `.js` files (and optionally `.d.ts` declaration files). In an Angular project, you never call `tsc` directly for building — Angular's CLI uses esbuild which strips types extremely fast — but `tsc` is still used for type-checking during development and in CI pipelines.

```bash
# Install TypeScript globally
npm install -g typescript

# Check the version
tsc --version

# Compile a single file
tsc app.ts

# Watch mode — recompile on every save
tsc --watch

# Type-check only (no output files) — useful in CI
tsc --noEmit
```

When TypeScript compiles, it **erases all type information** from the output. The resulting JavaScript is exactly what a JavaScript developer would write — no runtime overhead, no type-checking at runtime, just clean JavaScript.

---

### Type Annotation Syntax

A type annotation is the `: Type` syntax you write after a variable name, parameter name, or return type position to explicitly declare what type a value should be.

```ts
// Variable annotations
let userName: string = "Alice";
let userAge: number = 30;
let isActive: boolean = true;

// Function parameter and return type annotations
function greet(name: string, title: string): string {
  return `Hello, ${title} ${name}!`;
}

// Arrow function
const multiply = (a: number, b: number): number => a * b;

// When the annotation goes on the left side of the assignment (object type)
let user: { name: string; age: number } = { name: "Alice", age: 30 };
```

Type annotations are always placed **after** the thing being typed, separated by a colon. This is the opposite of languages like C or Java where the type comes before the name.

---

### Type Inference — TypeScript's Superpower

One of TypeScript's most important features is **type inference**: the ability to automatically determine the type of a value from its context, without you writing an explicit annotation. TypeScript is remarkably good at this, and it means you only need to write annotations in places where TypeScript cannot figure out the type on its own.

```ts
// TypeScript infers all of these — no annotations needed
let name = "Alice";           // inferred: string
let count = 42;               // inferred: number
let active = true;            // inferred: boolean

// Infers the return type from the function body
function add(a: number, b: number) {
  return a + b;               // return type inferred as: number
}

// Infers array element type from the literal
let scores = [95, 87, 73];    // inferred: number[]

// Infers object shape from the literal
let user = {
  name: "Alice",              // TypeScript sees this is { name: string; age: number }
  age: 30
};
user.name = 42;               // Error: Type 'number' is not assignable to type 'string'
```

> **Best practice:** Do not write annotations everywhere just because you can. TypeScript's inference is excellent for local variables, function return types, and simple expressions. Add explicit annotations when the type cannot be inferred (function parameters, class properties, complex return types), when you want to be more precise than inference allows, or when you want the annotation to serve as documentation for future readers.

---

### Strict Mode — Always Enable It

TypeScript's `strict` compiler option enables a collection of the most important safety checks. In Angular projects created with the CLI, strict mode is enabled by default, and you should never disable it.

The most impactful checks that `strict` enables are `noImplicitAny` (which prevents TypeScript from silently assuming a type of `any` when it cannot infer a type — this is the "escape hatch" that lets TypeScript give up on safety) and `strictNullChecks` (which makes `null` and `undefined` their own distinct types, requiring you to explicitly handle the possibility that a value may not exist before using it).

```ts
// Without strictNullChecks — dangerous
function getLength(value: string) {
  return value.length; // TypeScript won't warn if value could be null
}
getLength(null); // Compiles fine, crashes at runtime

// With strictNullChecks — safe
function getLength(value: string | null): number {
  if (value === null) return 0; // must handle null explicitly
  return value.length;          // TypeScript now knows value is string here
}
```

The `tsconfig.json` entry for strict mode is simply:

```json
{
  "compilerOptions": {
    "strict": true
  }
}
```

---

## 5.2 Basic Types

### Primitive Types

TypeScript's primitive types map directly to JavaScript's seven primitive types. You already know these from JavaScript — TypeScript simply adds the ability to declare and check them statically.

**`string`** represents any sequence of text characters. Use it for names, labels, URLs, messages — anything textual.

```ts
let firstName: string = "Alice";
let greeting: string = `Hello, ${firstName}!`;  // template literals work too
```

**`number`** represents all numeric values — integers and floating-point numbers alike, just as in JavaScript. TypeScript does not distinguish between `int` and `float`.

```ts
let age: number = 30;
let price: number = 19.99;
let hex: number = 0xFF;       // hexadecimal
let binary: number = 0b1010;  // binary
```

**`boolean`** represents the two logical values `true` and `false`.

```ts
let isLoggedIn: boolean = false;
let hasPermission: boolean = true;
```

**`null` and `undefined`** are their own distinct types in TypeScript when `strictNullChecks` is enabled. This is crucial: a variable typed as `string` cannot be `null` or `undefined` unless you explicitly allow it with a union type (`string | null`).

```ts
let maybeString: string | null = null;      // allowed — explicit union
let definiteString: string = "hello";
definiteString = null;                       // Error! Cannot assign null to string

// undefined typically appears as the return type of functions that don't return
let noValue: undefined = undefined;
```

**`symbol`** represents a unique, immutable primitive. It is less common in application code but shows up in library code and when defining unique object property keys.

```ts
const id: symbol = Symbol("userId");
```

**`bigint`** represents integers of arbitrary precision, for when you need numbers larger than `Number.MAX_SAFE_INTEGER`.

```ts
const largeNumber: bigint = 9007199254740991n;
```

---

### The Special Types: `any`, `unknown`, `never`, `void`

These four types are unique to TypeScript's type system and understanding them deeply is essential.

**`any`** is the ultimate escape hatch. A value typed as `any` can be assigned to anything and anything can be assigned to it. TypeScript completely abandons type checking for `any` values. Think of `any` as opting out of TypeScript for a specific value. It exists for backward compatibility and for situations where you genuinely cannot know the type (such as working with a JavaScript library that has no type definitions). However, overusing `any` defeats the entire purpose of TypeScript.

```ts
let flexible: any = "hello";
flexible = 42;            // no error
flexible = { x: 1 };     // no error
flexible.nonExistent();   // no error — TypeScript has given up checking this
```

> **Best practice:** Treat `any` as a code smell. If you find yourself writing `any`, ask whether `unknown` would be more appropriate (it almost always is), whether you can write a more specific type, or whether you need to improve the type definitions of a library.

**`unknown`** is `any`'s type-safe counterpart. A value typed as `unknown` can hold anything, but unlike `any`, you cannot use an `unknown` value in any way without first narrowing its type with a type check. Use `unknown` when you receive data from the outside world (API responses, JSON parsing, user input) and need to validate it before using it.

```ts
function processInput(input: unknown): string {
  // Cannot use `input` until we check its type
  if (typeof input === "string") {
    return input.toUpperCase();  // TypeScript knows it's a string here
  }
  if (typeof input === "number") {
    return input.toFixed(2);     // TypeScript knows it's a number here
  }
  return String(input);
}

// This is how you should type API responses
async function fetchUser(id: number): Promise<unknown> {
  const response = await fetch(`/api/users/${id}`);
  return response.json(); // json() returns any — cast to unknown for safety
}
```

**`never`** represents a value that can never exist. It is the return type of functions that never return (they throw unconditionally or run forever), and it appears as the result of type narrowing when TypeScript has exhausted all possibilities. The `never` type is immensely useful for exhaustiveness checking.

```ts
// A function that always throws — it never returns a value
function fail(message: string): never {
  throw new Error(message);
}

// Exhaustiveness checking — TypeScript ensures every case is handled
type Shape = "circle" | "square" | "triangle";

function getArea(shape: Shape): number {
  switch (shape) {
    case "circle": return Math.PI * 5 ** 2;
    case "square": return 5 * 5;
    case "triangle": return (5 * 5) / 2;
    default:
      // If you add "hexagon" to Shape without adding a case here,
      // TypeScript will error because shape has type "hexagon" here (not never)
      const exhaustiveCheck: never = shape;
      throw new Error(`Unhandled shape: ${exhaustiveCheck}`);
  }
}
```

**`void`** represents the absence of a meaningful return value. It is primarily used as the return type of functions that perform side effects but do not return a useful value — essentially, functions that would return `undefined` in JavaScript.

```ts
function logMessage(msg: string): void {
  console.log(msg);
  // No return statement needed
}

// Event handlers typically return void
button.addEventListener("click", (event: MouseEvent): void => {
  handleClick(event);
});
```

---

### Type Assertions

Sometimes you know more about the type of a value than TypeScript can infer. A type assertion is a way to tell the compiler "trust me, I know this is of type X." It is important to understand that type assertions do not change the runtime value in any way — they are purely a compile-time instruction to the type checker.

```ts
// Using `as` keyword (preferred syntax)
const inputEl = document.querySelector("#username") as HTMLInputElement;
inputEl.value = "Alice"; // TypeScript now knows .value exists

// Angle-bracket syntax (older style, avoid in .tsx files where it conflicts with JSX)
const inputEl = <HTMLInputElement>document.querySelector("#username");
```

The `as` assertion is a promise to TypeScript that you know what you are doing. If you are wrong, you will get a runtime error that TypeScript could not protect you from. Use assertions sparingly and only when you have genuine knowledge the type system cannot express.

**Double assertion** (also called an "assertion sandwich") is used in rare cases where TypeScript refuses a direct assertion because the types are too unrelated. You pass through `unknown` as an intermediate step.

```ts
// TypeScript won't allow this directly — the types are too different
const result = someValue as number; // might error if someValue is a complex type

// Force it via unknown as an intermediate step (use with extreme care)
const result = someValue as unknown as number;
```

---

### The Non-Null Assertion Operator `!`

The non-null assertion operator (`!`) is placed after a value to tell TypeScript "this value is definitely not `null` or `undefined` — proceed as if it has its non-nullable type." Like `as`, it is a promise to the compiler that you know more than it does.

```ts
// Without the assertion — TypeScript complains because querySelector might return null
const input = document.querySelector("#username");
input.value = "Alice"; // Error: Object is possibly 'null'

// With the assertion — you guarantee it exists
const input = document.querySelector("#username")!;
input.value = "Alice"; // Compiles fine

// This is very common in Angular when accessing ViewChild elements
@ViewChild("myInput") myInput!: ElementRef<HTMLInputElement>;
// The ! says "I know Angular will have set this by the time I use it"
```

> **Important:** The `!` operator is easy to overuse. Before reaching for it, ask whether you can restructure the code to handle the null case properly, or whether using `?` optional chaining would be safer.

---

### Literal Types

A literal type is a type that represents exactly one specific value, not just a category of values. Instead of the broad `string` type (which could be any text), a literal type like `"admin"` can only ever be the string "admin".

Literal types become powerful when combined with union types to create a precise set of allowed values — essentially a type-level enum.

```ts
// Literal types for a fixed set of valid strings
type UserRole = "admin" | "editor" | "viewer";
type Direction = "north" | "south" | "east" | "west";
type HttpMethod = "GET" | "POST" | "PUT" | "PATCH" | "DELETE";

function setRole(role: UserRole): void {
  // ...
}

setRole("admin");    // OK
setRole("editor");   // OK
setRole("hacker");   // Error: Argument of type '"hacker"' is not assignable to type 'UserRole'

// Numeric literal types
type DiceValue = 1 | 2 | 3 | 4 | 5 | 6;
let roll: DiceValue = 7; // Error: Type '7' is not assignable to type 'DiceValue'

// Boolean literal types (less common, but valid)
type AlwaysTrue = true;

// Literal types in function return types — forces specific values
function getStatus(code: number): "success" | "error" | "pending" {
  if (code === 200) return "success";
  if (code >= 500) return "error";
  return "pending";
}
```

**Literal widening** is a subtle behavior to be aware of. When you declare `let role = "admin"`, TypeScript infers the type as `string` (widened), not as the literal `"admin"`, because `let` allows reassignment. To keep the literal type, use `const` or add an explicit annotation.

```ts
let role1 = "admin";              // inferred: string (widened, can change)
const role2 = "admin";            // inferred: "admin" (literal, cannot change)
let role3: "admin" = "admin";     // explicit literal type annotation
```

---

## 5.3 Complex Types

### Arrays

TypeScript offers two equivalent syntaxes for typed arrays. Both are identical in behavior — choose one style and be consistent within a project.

```ts
// Syntax 1: type[] — more common, preferred for simple types
let names: string[] = ["Alice", "Bob", "Carol"];
let scores: number[] = [95, 87, 73];
let flags: boolean[] = [true, false, true];

// Syntax 2: Array<type> — generic syntax, required for complex types
let users: Array<{ id: number; name: string }> = [];
let callbacks: Array<() => void> = [];

// Read-only array — cannot be modified after creation
const ids: readonly number[] = [1, 2, 3];
ids.push(4);    // Error: Property 'push' does not exist on type 'readonly number[]'
ids[0] = 99;    // Error: Index signature in type 'readonly number[]' only permits reading

// Equivalent generic syntax
const ids: ReadonlyArray<number> = [1, 2, 3];
```

---

### Tuples

A tuple is a fixed-length array where each position has a known, potentially different type. Tuples are ideal for representing data where the position carries meaning — like the return value of `useState` in React, or the coordinates `[latitude, longitude]`.

```ts
// A tuple with a string at position 0 and a number at position 1
let coordinate: [number, number] = [40.7128, -74.0060];
let nameAndAge: [string, number] = ["Alice", 30];

// Accessing tuple elements preserves their individual types
let lat: number = coordinate[0];
let lng: number = coordinate[1];
let name: string = nameAndAge[0];
let age: number = nameAndAge[1];

// Destructuring tuples — natural and common
const [latitude, longitude] = coordinate;
const [userName, userAge] = nameAndAge;

// Labeled tuple elements (TypeScript 4.0+) — adds clarity without changing behavior
type Coordinate = [latitude: number, longitude: number];
type UserRecord = [name: string, age: number, active: boolean];

// Optional tuple elements
type FlexibleConfig = [host: string, port?: number];
let config1: FlexibleConfig = ["localhost"];          // OK — port is optional
let config2: FlexibleConfig = ["localhost", 3000];    // OK

// Rest elements in tuples (useful for typed variadic functions)
type AtLeastOneString = [string, ...string[]];
let single: AtLeastOneString = ["a"];           // OK
let multiple: AtLeastOneString = ["a", "b", "c"]; // OK
let empty: AtLeastOneString = [];               // Error — must have at least one
```

---

### Enums

Enums (enumerations) define a named set of constant values. They make intent clear and prevent magic strings and numbers from littering your codebase. TypeScript supports three kinds of enums.

**Numeric enums** assign auto-incrementing numeric values starting from 0 by default. You can override the starting value or any individual value.

```ts
enum Direction {
  Up,    // 0
  Down,  // 1
  Left,  // 2
  Right  // 3
}

// Usage
let move: Direction = Direction.Up;

// You can also go from number to name
Direction[0]; // "Up" — reverse mapping built in

// Override the start value
enum StatusCode {
  OK = 200,
  NotFound = 404,
  ServerError = 500
}

// Auto-increment from an overridden value
enum ErrorCodes {
  None = 0,
  MissingField,  // 1
  InvalidEmail,  // 2
  DuplicateUser  // 3
}
```

**String enums** assign explicit string values to each member. They have no reverse mapping, but they are far more readable in debugging, logging, and network calls.

```ts
enum UserRole {
  Admin = "ADMIN",
  Editor = "EDITOR",
  Viewer = "VIEWER"
}

enum ApiStatus {
  Loading = "loading",
  Success = "success",
  Error = "error"
}

// In Angular services — string enums are excellent for API status tracking
let status: ApiStatus = ApiStatus.Loading;
console.log(status); // "loading" — human-readable, not a magic number
```

**`const` enums** are erased entirely at compile time and replaced with their literal values inline. This produces smaller, faster JavaScript output because there is no runtime enum object. The tradeoff is that you cannot use `const` enum values dynamically (no reverse mapping at runtime).

```ts
const enum Alignment {
  Left = "left",
  Center = "center",
  Right = "right"
}

const align: Alignment = Alignment.Center;
// Compiled to: const align = "center"; — enum object doesn't exist at runtime
```

> **Best practice for Angular:** Prefer `const` enums for performance-critical code, or better yet, prefer **string union types** (`type Role = "admin" | "editor" | "viewer"`) over enums for most application-level constants. String union types are simpler, require no import, are erased at compile time automatically, and read naturally. Numeric enums are most useful when interfacing with external systems that use numeric codes (like HTTP status codes or bitmask flags).

---

### Object Types

An object type describes the shape of an object — the names of its properties and the type of each property. You can express object types inline or via interfaces and type aliases (covered in Section 5.4).

```ts
// Inline object type annotation
let user: { id: number; name: string; email: string } = {
  id: 1,
  name: "Alice",
  email: "alice@example.com"
};

// Optional properties with ?
let config: { host: string; port?: number } = { host: "localhost" }; // OK — port is optional

// Readonly properties — cannot be reassigned after initial creation
let point: { readonly x: number; readonly y: number } = { x: 10, y: 20 };
point.x = 99; // Error: Cannot assign to 'x' because it is a read-only property
```

---

### Type Aliases

A type alias gives a name to any type, making it reusable and self-documenting. You create type aliases with the `type` keyword. Unlike interfaces (which are covered in depth in Section 5.4), type aliases can describe any type — not just objects.

```ts
// Alias for a primitive type
type UserId = number;
type UserName = string;
type Callback = () => void;

// Alias for an object type
type Point = { x: number; y: number };
type User = { id: UserId; name: UserName; active: boolean };

// Alias for a union type
type StringOrNumber = string | number;
type Status = "pending" | "active" | "inactive";

// Alias for a function type
type EventHandler<T = Event> = (event: T) => void;
type ClickHandler = EventHandler<MouseEvent>;

// Use the aliases
let p: Point = { x: 10, y: 20 };
let handler: ClickHandler = (e) => console.log(e.clientX);
```

---

### Union Types (`|`)

A union type says that a value can be one of several types. It is written with the `|` (pipe) character between types. Union types are one of TypeScript's most frequently used features.

```ts
// A value that can be either a string or number
type StringOrNumber = string | number;
let id: StringOrNumber = "abc123";
id = 42; // also valid

// The real power: narrowing unions
function formatId(id: string | number): string {
  if (typeof id === "string") {
    // TypeScript knows id is string here
    return id.toUpperCase();
  } else {
    // TypeScript knows id is number here
    return id.toFixed(0);
  }
}

// Unions of object types (discriminated unions — see Section 5.8)
type ApiResult =
  | { status: "success"; data: User[] }
  | { status: "error"; message: string }
  | { status: "loading" };

function renderResult(result: ApiResult): void {
  if (result.status === "success") {
    displayUsers(result.data); // TypeScript knows result.data exists here
  } else if (result.status === "error") {
    showError(result.message); // TypeScript knows result.message exists here
  } else {
    showSpinner(); // TypeScript knows this is the "loading" case
  }
}
```

---

### Intersection Types (`&`)

An intersection type combines multiple types into one — a value of an intersection type must satisfy all of the combined types simultaneously. It is written with the `&` (ampersand) character. Think of intersection as "AND" — a value must have everything from type A AND everything from type B.

```ts
type Timestamped = { createdAt: Date; updatedAt: Date };
type Identifiable = { id: number };
type Named = { name: string };

// A DatabaseRecord must have an ID, a name, and timestamps
type DatabaseRecord = Identifiable & Named & Timestamped;

const record: DatabaseRecord = {
  id: 1,
  name: "Widget",
  createdAt: new Date(),
  updatedAt: new Date()
  // All four properties are required — any missing one is an error
};

// Intersection types for mixing in capabilities — a common Angular pattern
type WithLoading<T> = T & { isLoading: boolean; error: string | null };
type UserWithState = WithLoading<User>;

// The WithLoading mixin adds isLoading and error to any type
const userState: UserWithState = {
  id: 1,
  name: "Alice",
  isLoading: false,
  error: null
};
```

> **Union vs Intersection mental model:** A union type (`A | B`) is **either A or B** — broader, more permissive. An intersection type (`A & B`) is **both A and B simultaneously** — narrower, more restrictive. Union widens the set of allowed values; intersection narrows it.

---

## 🔑 Key Takeaways — Part 1

TypeScript's purpose is to catch errors at compile time rather than runtime, and its power comes from the combination of explicit type annotations and automatic type inference. You should write annotations where inference falls short (function parameters, class properties, API boundaries) and let inference do the work everywhere else. The four special types each serve a distinct purpose: `any` opts out of the type system entirely (avoid it), `unknown` accepts anything but forces you to check before using (prefer this over `any` for external data), `never` represents impossible values and is essential for exhaustiveness checking, and `void` signals an intentional absence of a return value. Literal types combined with union types give you the precision of an enum with the simplicity of a plain type. Understanding the difference between union (`|`) and intersection (`&`) — "or" versus "and" — is foundational to every complex type you will write in Angular.

---

*Part 1 of 5 — Phase 5 TypeScript Mastery | Next: [Interfaces, Functions & Classes](./part2-interfaces-functions-classes.md)*

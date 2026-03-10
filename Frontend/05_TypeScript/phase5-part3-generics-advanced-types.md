# 🔷 Phase 5 — Part 3: Generics · Advanced Types
> Sections 5.7 · 5.8 | Angular Frontend Mastery Roadmap

---

## 5.7 Generics

Generics are the feature that takes TypeScript from a "types for beginners" tool to a genuinely powerful type system capable of expressing complex relationships between values. The core idea is simple: instead of hardcoding a specific type, you write a **type variable** (a placeholder) that gets filled in when the generic is used. This lets you write code that is simultaneously **flexible** (works with many types) and **type-safe** (TypeScript still knows exactly what types are involved at every usage site).

The mental model is: generics are to types what function parameters are to values. A function that takes a parameter can work with many different values while remaining specific about what it does with them. A generic function or type that takes a type parameter can work with many different types while remaining specific about the relationships between them.

---

### Generic Functions

Consider writing a function that returns the first element of an array. Without generics, you have two bad options: write a separate function for every type (`firstNumber`, `firstString`, etc.), or use `any` (which throws away all type safety).

```ts
// The problem without generics
function firstAny(arr: any[]): any {  // loses all type information!
  return arr[0];
}
const n = firstAny([1, 2, 3]);  // TypeScript thinks n is `any` — dangerous!

// The solution with generics
function first<T>(arr: T[]): T | undefined {
  return arr[0];
}

// TypeScript INFERS the type argument from the argument you pass — usually no annotation needed
const n = first([1, 2, 3]);         // TypeScript infers T = number, result type: number | undefined
const s = first(["a", "b", "c"]);   // TypeScript infers T = string, result type: string | undefined
const u = first([{ id: 1 }]);       // TypeScript infers T = { id: number }, result type: ...

// You can also explicitly provide the type argument when inference isn't enough
const explicit = first<number>([1, 2, 3]);
```

The `<T>` syntax declares a type parameter named `T`. The name is a convention — `T` for "Type," `K` for "Key," `V` for "Value," `E` for "Element" — but you can use any identifier.

```ts
// A function with multiple type parameters expressing a relationship between input and output
function mapObject<TInput, TOutput>(
  obj: Record<string, TInput>,
  transformer: (value: TInput) => TOutput
): Record<string, TOutput> {
  const result: Record<string, TOutput> = {};
  for (const key in obj) {
    result[key] = transformer(obj[key]);
  }
  return result;
}

// TypeScript infers TInput = string, TOutput = number
const lengths = mapObject({ a: "hello", b: "hi" }, s => s.length);
// result type: Record<string, number>

// A paired function — swaps key and value
function swap<A, B>(pair: [A, B]): [B, A] {
  return [pair[1], pair[0]];
}

const swapped = swap(["Alice", 30]); // type: [number, string]
```

---

### Generic Interfaces and Type Aliases

Generics are not limited to functions — interfaces and type aliases can also be generic. This is how you define reusable shapes that work across different data types.

```ts
// Generic interface — the shape of an API response wrapper
interface ApiResponse<TData> {
  data: TData;
  status: number;
  message: string;
  timestamp: Date;
}

// Reuse with different data types
type UserResponse = ApiResponse<User>;
type ProductListResponse = ApiResponse<Product[]>;
type PaginatedUsersResponse = ApiResponse<{ items: User[]; total: number; page: number }>;

// Generic type alias for an async action
type AsyncOperation<TInput, TOutput> = (input: TInput) => Promise<TOutput>;
type UserLoader = AsyncOperation<number, User>;         // (id: number) => Promise<User>
type ProductSearch = AsyncOperation<string, Product[]>; // (query: string) => Promise<Product[]>

// Generic Result type — represents success or failure (like Rust's Result type)
type Result<TSuccess, TError = Error> =
  | { success: true; value: TSuccess }
  | { success: false; error: TError };

async function tryFetch<T>(url: string): Promise<Result<T>> {
  try {
    const res = await fetch(url);
    const data: T = await res.json();
    return { success: true, value: data };
  } catch (error) {
    return { success: false, error: error as Error };
  }
}

const result = await tryFetch<User[]>("/api/users");
if (result.success) {
  result.value.forEach(user => console.log(user.name)); // TypeScript knows it's User[]
} else {
  console.error(result.error.message); // TypeScript knows it's Error
}
```

---

### Generic Classes

Classes can be generic too. This is essential for building reusable data structures and service abstractions in Angular.

```ts
// A generic in-memory repository (common Angular service pattern)
class Repository<TEntity extends { id: number }> {
  private items: TEntity[] = [];

  findById(id: number): TEntity | undefined {
    return this.items.find(item => item.id === id);
  }

  findAll(): TEntity[] {
    return [...this.items]; // return a copy — immutability
  }

  save(entity: TEntity): TEntity {
    const existingIndex = this.items.findIndex(item => item.id === entity.id);
    if (existingIndex >= 0) {
      this.items[existingIndex] = entity;
    } else {
      this.items.push(entity);
    }
    return entity;
  }

  delete(id: number): boolean {
    const index = this.items.findIndex(item => item.id === id);
    if (index >= 0) {
      this.items.splice(index, 1);
      return true;
    }
    return false;
  }
}

// TypeScript ensures type safety for each specific repository
const userRepo = new Repository<User>();
const productRepo = new Repository<Product>();

userRepo.save({ id: 1, name: "Alice", email: "a@b.com", active: true });
productRepo.save({ id: 1, name: "Widget", price: 9.99 });

// Cross-contamination is impossible:
userRepo.save({ id: 1, name: "Widget", price: 9.99 }); // Error: Product is not assignable to User
```

---

### Type Constraints with `extends`

By default, a type parameter `T` can be absolutely anything — a string, a number, an object, a function. Sometimes you need to constrain the type parameter to only accept types that have certain properties. You do this with `extends` in the type parameter declaration.

```ts
// Without constraint — TypeScript doesn't know what properties T has
function getLength<T>(value: T): number {
  return value.length; // Error: Property 'length' does not exist on type 'T'
}

// With a constraint — T must have a .length property
function getLength<T extends { length: number }>(value: T): number {
  return value.length; // OK — TypeScript knows T has .length
}

getLength("hello");       // 5 — string has .length
getLength([1, 2, 3]);     // 3 — array has .length
getLength({ length: 10 }); // 10 — plain object with .length
getLength(42);            // Error: number doesn't have .length

// Constraint to only keyof another type — safe property access
function getProperty<TObject, TKey extends keyof TObject>(
  obj: TObject,
  key: TKey
): TObject[TKey] {
  return obj[key]; // TypeScript knows this is safe because TKey is a valid key of TObject
}

const user = { id: 1, name: "Alice", email: "a@b.com" };
getProperty(user, "name");    // "Alice" — return type: string
getProperty(user, "id");      // 1 — return type: number
getProperty(user, "missing"); // Error: "missing" is not a key of user
```

---

### Default Type Parameters

Just as function parameters can have default values, type parameters can have default types. If a type argument is not provided and cannot be inferred, the default is used.

```ts
// Default type parameter
interface Container<T = string> {
  value: T;
  label: string;
}

const textContainer: Container = { value: "hello", label: "Text" };   // T defaults to string
const numContainer: Container<number> = { value: 42, label: "Number" }; // explicit

// Useful in Angular-style service types where the default is the most common case
interface LoadingState<TData = unknown, TError = Error> {
  data: TData | null;
  loading: boolean;
  error: TError | null;
}

// Most components just use LoadingState — no type args needed
type UserState = LoadingState<User[]>;         // common — specify data type
type GenericState = LoadingState;              // uses both defaults
```

---

### Conditional Types with Generics and the `infer` Keyword

Conditional types allow TypeScript to choose between two types based on a condition. The syntax `T extends U ? X : Y` reads: "if T is assignable to U, the type is X; otherwise it is Y." This becomes especially powerful when combined with generics and the `infer` keyword.

The `infer` keyword, usable only inside the `extends` clause of a conditional type, lets TypeScript **extract** a type from another type and give it a name you can use in the result branches.

```ts
// A simple conditional type
type IsString<T> = T extends string ? true : false;

type A = IsString<string>;  // true
type B = IsString<number>;  // false
type C = IsString<"hello">; // true — "hello" extends string

// Flatten one level of array
type Flatten<T> = T extends (infer Item)[] ? Item : T;
// If T is an array, infer the element type as `Item` and return `Item`
// If T is not an array, return T unchanged

type X = Flatten<string[]>;   // string — extracted the element type
type Y = Flatten<number[][]>; // number[] — only one level flattened
type Z = Flatten<string>;     // string — T is not an array, returned as-is

// Extract the return type of a function (this is how ReturnType<T> is built)
type MyReturnType<T extends (...args: any[]) => any> =
  T extends (...args: any[]) => infer R ? R : never;

function fetchUser(): Promise<User> { /* ... */ return Promise.resolve({} as User); }
type FetchUserResult = MyReturnType<typeof fetchUser>; // Promise<User>

// Extract the type inside a Promise
type Awaited<T> = T extends Promise<infer U> ? Awaited<U> : T; // recursive!
type UserValue = Awaited<Promise<Promise<User>>>; // User — unwraps nested Promises

// Distributive conditional types — when T is a union, the condition applies to each member
type ToArray<T> = T extends any ? T[] : never;
type StringOrNumberArrays = ToArray<string | number>; // string[] | number[]
// NOT (string | number)[] — each union member is processed separately!
```

---

## 5.8 Advanced Types

Advanced types are where TypeScript becomes a genuinely expressive language for describing complex type relationships. These are the tools that let you write type-safe utility functions, derive types from existing types automatically, and create precise types that eliminate entire categories of bugs. Many of these features are used internally by TypeScript's built-in utility types (Section 5.9) and by Angular's own type system.

### Mapped Types

A mapped type creates a new type by transforming each property of an existing type. The syntax iterates over the keys of a type using `keyof` and applies a transformation to each one. Think of `map()` for arrays, but for types.

```ts
// The basic pattern: { [K in keyof T]: transformation }
type Stringify<T> = {
  [K in keyof T]: string; // every property becomes a string
};

interface User { id: number; name: string; active: boolean; }

type StringifiedUser = Stringify<User>;
// { id: string; name: string; active: string }

// Adding modifiers — make all properties optional (this is how Partial<T> is built)
type MyPartial<T> = {
  [K in keyof T]?: T[K]; // ? makes each property optional; T[K] preserves original type
};

// Adding modifiers — make all properties readonly (how Readonly<T> is built)
type MyReadonly<T> = {
  readonly [K in keyof T]: T[K];
};

// Removing modifiers — the - prefix removes optional and readonly modifiers
type Mutable<T> = {
  -readonly [K in keyof T]: T[K]; // remove readonly from every property
};

type Required<T> = {
  [K in keyof T]-?: T[K]; // remove optional (?) from every property
};

// Remapping keys — transforming the key names themselves (TS 4.1+)
type Getters<T> = {
  [K in keyof T as `get${Capitalize<string & K>}`]: () => T[K];
};

type UserGetters = Getters<User>;
// { getId: () => number; getName: () => string; getActive: () => boolean }

// Filtering keys — use `never` to exclude properties you don't want
type PickFunctions<T> = {
  [K in keyof T as T[K] extends (...args: any[]) => any ? K : never]: T[K];
};
```

---

### Conditional Types in Depth

Conditional types combined with mapped types unlock the ability to derive entirely new types from existing ones. This is the engine behind most of TypeScript's utility types and behind Angular's type-safe form and router APIs.

```ts
// NonNullable — removes null and undefined from a type
type MyNonNullable<T> = T extends null | undefined ? never : T;
type A = MyNonNullable<string | null | undefined>; // string

// Extract — keep only the types in T that are assignable to U
type MyExtract<T, U> = T extends U ? T : never;
type B = MyExtract<"a" | "b" | "c" | 1 | 2, string>; // "a" | "b" | "c"

// Exclude — remove the types in T that are assignable to U
type MyExclude<T, U> = T extends U ? never : T;
type C = MyExclude<"a" | "b" | "c" | 1 | 2, string>; // 1 | 2

// Deep partial — recursively make all nested properties optional
type DeepPartial<T> = {
  [K in keyof T]?: T[K] extends object ? DeepPartial<T[K]> : T[K];
};

interface Config {
  server: { host: string; port: number };
  database: { url: string; pool: number };
}

type PartialConfig = DeepPartial<Config>;
// server and database are optional, and their properties are also optional
const partial: PartialConfig = { server: { host: "localhost" } }; // valid
```

---

### Template Literal Types

Template literal types (TypeScript 4.1+) apply the same template literal syntax you know from JavaScript strings — but at the type level. They let you construct new string literal types by combining existing ones.

```ts
// Basic template literal type
type EventName = "click" | "focus" | "blur";
type EventHandler = `on${Capitalize<EventName>}`; // "onClick" | "onFocus" | "onBlur"

// Generating CSS property names — useful for type-safe style objects
type CssProperty = `--${string}`; // any CSS custom property
const token: CssProperty = "--primary-color"; // OK
const bad: CssProperty = "color"; // Error: must start with "--"

// Building typed API route patterns
type ApiVersion = "v1" | "v2";
type ApiResource = "users" | "products" | "orders";
type ApiRoute = `/${ApiVersion}/${ApiResource}`; // "/v1/users" | "/v1/products" | ...

// Extracting event names — Angular-style event binding types
type DomEventNames = keyof HTMLElementEventMap; // "click" | "focus" | "blur" | ...
type AngularEventBinding<T extends string> = `(${T})`; // "(click)" | "(focus)" | ...

// Very practical: get the type of a nested property using template literals
type PropType<T, K extends string> =
  K extends `${infer Parent}.${infer Rest}`
    ? Parent extends keyof T
      ? PropType<T[Parent], Rest>
      : never
    : K extends keyof T
      ? T[K]
      : never;

interface UserProfile { address: { city: string; zip: string }; name: string; }
type CityType = PropType<UserProfile, "address.city">; // string
type NameType = PropType<UserProfile, "name">;          // string
```

---

### Discriminated Unions

A discriminated union (also called a tagged union) is one of the most powerful patterns in TypeScript for modeling data that can be in different states. It is a union of object types where each member has a common **discriminant property** — a literal-typed property that uniquely identifies which member of the union you are dealing with.

When TypeScript narrows a discriminated union (via an `if` or `switch` on the discriminant property), it knows precisely which member you are working with in each branch, giving you full type safety for all the other properties.

```ts
// Each state of an API request is a separate type with a discriminant "status" property
type ApiState<T> =
  | { status: "idle" }
  | { status: "loading" }
  | { status: "success"; data: T }
  | { status: "error"; error: Error; retryCount: number };

function renderUserState(state: ApiState<User[]>): string {
  switch (state.status) {
    case "idle":
      return "Click to load users";
    case "loading":
      return "Loading...";
    case "success":
      // TypeScript knows `state.data` exists here — it's User[]
      return `Loaded ${state.data.length} users`;
    case "error":
      // TypeScript knows `state.error` and `state.retryCount` exist here
      return `Error: ${state.error.message} (retried ${state.retryCount} times)`;
  }
}

// Discriminated unions for shape modeling
type Shape =
  | { kind: "circle"; radius: number }
  | { kind: "rectangle"; width: number; height: number }
  | { kind: "triangle"; base: number; height: number };

function area(shape: Shape): number {
  switch (shape.kind) {
    case "circle":
      return Math.PI * shape.radius ** 2;     // only `radius` is available here
    case "rectangle":
      return shape.width * shape.height;       // only `width` and `height` here
    case "triangle":
      return (shape.base * shape.height) / 2; // only `base` and `height` here
  }
}

// Angular usage: typed action objects (NgRx-style)
type UserAction =
  | { type: "LOAD_USERS" }
  | { type: "LOAD_USERS_SUCCESS"; users: User[] }
  | { type: "LOAD_USERS_FAILURE"; error: string }
  | { type: "DELETE_USER"; userId: number };
```

---

### Recursive Types

TypeScript supports types that reference themselves, which is necessary for describing recursive data structures like trees, nested menus, and JSON data.

```ts
// A JSON value — can be any JSON-compatible type, including nested objects/arrays
type JsonValue =
  | string
  | number
  | boolean
  | null
  | JsonValue[]                     // an array of JsonValues
  | { [key: string]: JsonValue };   // an object of JsonValues

const valid: JsonValue = {
  name: "Alice",
  age: 30,
  address: { city: "London", zip: "EC1A" }, // nested object — still a JsonValue
  tags: ["admin", "active"],                // array of strings — still JsonValues
};

// A tree node structure
interface TreeNode<T> {
  value: T;
  children: TreeNode<T>[]; // recursive reference to itself
}

const menu: TreeNode<string> = {
  value: "Home",
  children: [
    { value: "Products", children: [
      { value: "Widgets", children: [] },
      { value: "Gadgets", children: [] }
    ]},
    { value: "About", children: [] }
  ]
};
```

---

### Variadic Tuple Types

Variadic tuple types (TypeScript 4.0+) allow tuple types to use spread syntax with generic type parameters. This enables precise typing for functions that take or return variable-length argument lists.

```ts
// A function that prepends an element to a tuple
function prepend<T, Tuple extends unknown[]>(
  value: T,
  tuple: [...Tuple]
): [T, ...Tuple] {
  return [value, ...tuple];
}

const result = prepend("hello", [1, true, "world"]);
// TypeScript infers the return type as [string, number, boolean, string]

// Typing a generic curry function (simplified)
type Head<T extends any[]> = T extends [infer H, ...any[]] ? H : never;
type Tail<T extends any[]> = T extends [any, ...infer R] ? R : never;

// A concat function with precise typing
function concat<T extends unknown[], U extends unknown[]>(
  a: [...T],
  b: [...U]
): [...T, ...U] {
  return [...a, ...b];
}

const merged = concat([1, 2], ["a", "b"]); // type: [number, number, string, string]
```

---

### `const` Type Parameters (TypeScript 5.0)

When you pass a value to a generic function, TypeScript usually infers a "widened" type. For example, passing `["a", "b"]` infers `string[]` rather than the literal tuple `["a", "b"]`. The `const` modifier on a type parameter forces TypeScript to infer the most specific (narrowest) type possible, as if you had written `as const` on the argument.

```ts
// Without const type parameter — widened inference
function makeArray<T>(values: T[]): T[] {
  return values;
}
const arr = makeArray(["a", "b", "c"]);
// arr type: string[] — widened, lost the specific values

// With const type parameter — narrow inference
function makeArrayConst<const T>(values: T): T {
  return values;
}
const arr2 = makeArrayConst(["a", "b", "c"]);
// arr2 type: readonly ["a", "b", "c"] — narrow, knows the exact values!

// Practical Angular use: type-safe route definitions
function defineRoutes<const T extends readonly string[]>(routes: T): T {
  return routes;
}
const routes = defineRoutes(["home", "users", "products"] as const);
type RouteNames = typeof routes[number]; // "home" | "users" | "products"
```

---

### Inferred Type Predicates (TypeScript 5.5)

A **type predicate** is a function return type of the form `value is Type`, which tells TypeScript that if the function returns `true`, then the argument has the narrowed type. Before TypeScript 5.5, you had to write these predicates manually. TypeScript 5.5 infers them automatically when the function body provides enough evidence.

```ts
// Manual type predicate — before TS 5.5 (still valid, now often unnecessary)
function isString(value: unknown): value is string {
  return typeof value === "string";
}

// Automatic inference — TypeScript 5.5 infers the predicate from the body
function isNonNull<T>(value: T | null | undefined): value is T {
  return value !== null && value !== undefined;
}
// TypeScript 5.5 automatically infers the return type as `value is T`

// Very useful for filtering arrays
const items: (User | null)[] = [user1, null, user2, null, user3];
const activeUsers = items.filter(isNonNull);
// activeUsers type: User[] — TypeScript knows null is filtered out
// Before TS 5.5, filter(Boolean) did NOT narrow the type automatically

// Automatically inferred from a typeof check
function isUser(value: unknown) {
  return (
    typeof value === "object" &&
    value !== null &&
    "id" in value &&
    "name" in value
  );
}
// TypeScript 5.5 infers: (value: unknown) => value is { id: unknown; name: unknown }
```

---

### Explicit Resource Management — `using` / `await using`

The `using` and `await using` declarations (TypeScript 5.2+, implementing the TC39 Explicit Resource Management proposal) ensure that resources are automatically cleaned up when a variable goes out of scope, even if an exception is thrown.

An object is a "disposable resource" if it has a `[Symbol.dispose]()` method (synchronous) or `[Symbol.asyncDispose]()` method (asynchronous). When you declare it with `using`, TypeScript guarantees the dispose method is called at the end of the block.

```ts
// Creating a disposable resource
function createDatabaseConnection(url: string) {
  const connection = openConnection(url);
  return {
    query: connection.query.bind(connection),
    [Symbol.dispose]() {
      connection.close();
      console.log("Connection closed");
    }
  };
}

// Usage — dispose is called AUTOMATICALLY when the block exits
function processData() {
  using db = createDatabaseConnection("postgres://localhost/mydb");
  // Even if this throws, db[Symbol.dispose]() is called
  const result = db.query("SELECT * FROM users");
  return result;
} // db is disposed here

// Async version
async function fetchAndProcess() {
  await using client = createHttpClient();
  const data = await client.get("/api/data");
  return transformData(data);
} // client is asynchronously disposed here

// Angular use case — managing temporary subscriptions
function createTemporarySubscription(observable: Observable<Event>) {
  const sub = observable.subscribe(handleEvent);
  return {
    [Symbol.dispose]() { sub.unsubscribe(); }
  };
}
```

---

## 🔑 Key Takeaways — Part 3

Generics are the single most important advanced TypeScript concept. Once you understand them, you can write utility types and service abstractions that are simultaneously flexible and fully type-safe. The mental model is: type parameters are to types what function parameters are to values. Always let TypeScript infer type arguments where possible — explicit type arguments are only needed when inference fails or when you want to be more restrictive.

Type constraints (`extends`) are what make generic functions useful in practice — they let you access properties on a type parameter without losing genericity. The `infer` keyword inside conditional types is what powers all of TypeScript's extraction utilities (`ReturnType`, `Parameters`, `Awaited`) and is worth studying carefully.

Discriminated unions with a shared discriminant property are the most robust way to model data that can be in multiple distinct states — this pattern appears throughout Angular applications in API state management, NgRx actions, and routing state. Template literal types let you construct precise string types, which is useful for type-safe event names, API routes, and CSS property names. The `const` type parameter modifier (TS 5.0) and inferred type predicates (TS 5.5) are recent additions that significantly reduce the boilerplate needed for narrowing and precise type inference.

---

*Part 3 of 5 — Phase 5 TypeScript Mastery | Next: [Utility Types & Modern TS Features](./part4-utility-types-modern-features.md)*

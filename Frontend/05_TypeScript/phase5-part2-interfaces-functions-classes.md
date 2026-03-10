# 🔷 Phase 5 — Part 2: Interfaces · Type Aliases · Functions · Classes
> Sections 5.4 · 5.5 · 5.6 | Angular Frontend Mastery Roadmap

---

## 5.4 Interfaces & Type Aliases

Interfaces are TypeScript's primary way to describe the shape of objects. They are one of the most written constructs in any Angular application — every model, DTO (Data Transfer Object), service contract, and component API is typically described as an interface. Understanding interfaces deeply, including their subtle differences from type aliases, makes your Angular code more robust and readable.

### Declaring and Using Interfaces

An interface is a named contract that describes the shape an object must have. It specifies property names and their types, but contains no implementation — just the declaration.

```ts
// A basic interface for a user entity
interface User {
  id: number;
  name: string;
  email: string;
  active: boolean;
}

// Any object used as a User must match this exact shape
const alice: User = {
  id: 1,
  name: "Alice",
  email: "alice@example.com",
  active: true
};

// TypeScript performs structural type checking — if the shape matches, it's compatible
// (More on structural typing below)
const bob = { id: 2, name: "Bob", email: "bob@example.com", active: false };
const users: User[] = [alice, bob]; // OK — bob's shape matches User
```

An important TypeScript concept at work here is **structural typing** (also called "duck typing"). TypeScript does not care about the nominal class or interface name — it only cares about the shape. If an object has all the required properties with the correct types, it satisfies the interface, regardless of how it was created.

---

### Optional Properties (`?`)

Not every property of an object is always required. Optional properties are marked with a `?` after the property name. When a property is optional, its type is automatically widened to `Type | undefined` — meaning TypeScript forces you to handle the case where the property may not be present.

```ts
interface UserProfile {
  id: number;
  name: string;
  bio?: string;         // may or may not exist
  avatarUrl?: string;   // may or may not exist
  birthYear?: number;
}

const minimalUser: UserProfile = { id: 1, name: "Alice" }; // valid — optional fields omitted
const fullUser: UserProfile = {
  id: 2,
  name: "Bob",
  bio: "Software engineer",
  avatarUrl: "https://example.com/bob.jpg",
  birthYear: 1990
};

// TypeScript forces you to handle optionality before using the value
function displayBio(profile: UserProfile): string {
  // profile.bio might be undefined — TypeScript prevents you from treating it as a string
  // without checking first
  if (profile.bio) {
    return profile.bio.trim(); // TypeScript knows bio is string here
  }
  return "No bio provided";
}

// Optional chaining works perfectly with optional properties
const bioLength = profile.bio?.length ?? 0;
```

---

### Readonly Properties

The `readonly` modifier on an interface property means the property can be set when the object is first created, but cannot be changed afterward. This is the TypeScript-level enforcement of immutability for object properties.

```ts
interface Config {
  readonly apiUrl: string;
  readonly maxRetries: number;
  timeout?: number; // this one can be changed
}

const config: Config = {
  apiUrl: "https://api.example.com",
  maxRetries: 3,
  timeout: 5000
};

config.timeout = 10000; // OK — not readonly
config.apiUrl = "https://other.com"; // Error: Cannot assign to 'apiUrl' because it is a read-only property
```

> **Important distinction:** `readonly` in TypeScript interfaces is a compile-time check only. It does not use JavaScript's `Object.freeze()` under the hood. At runtime, the property can technically be mutated if you bypass TypeScript's type system. For true runtime immutability, you still need `Object.freeze()`.

---

### Index Signatures

Sometimes you need an object whose keys are not known ahead of time — like a dictionary, a map, or a record where the keys come from user input or API data. An index signature describes the types of all keys and values in such an object.

```ts
// A dictionary where keys are strings and values are strings
interface StringMap {
  [key: string]: string;
}

const translations: StringMap = {
  greeting: "Hello",
  farewell: "Goodbye",
  thanks: "Thank you"
};
translations["newKey"] = "Some value"; // valid
translations["anotherKey"] = 42;       // Error: Type 'number' is not assignable to type 'string'

// You can combine index signatures with specific known properties,
// but the known properties must be compatible with the index signature's value type
interface FlexibleConfig {
  timeout: number;        // specific known property
  retries: number;        // specific known property
  [key: string]: number;  // all other keys must also be numbers
}

// A more common pattern in Angular — an object with string keys and any value
interface FormErrors {
  [controlName: string]: string | null;
}

const errors: FormErrors = {
  email: "Invalid email format",
  password: "Too short",
  username: null
};
```

---

### Interface Extension with `extends`

Interfaces can extend other interfaces, inheriting all of their properties and adding new ones. This models inheritance in data shapes — a `AdminUser` is a `User` with extra properties — without the complexity of class inheritance.

```ts
interface Animal {
  name: string;
  age: number;
}

interface Pet extends Animal {
  owner: string;
  isVaccinated: boolean;
}

interface ServiceDog extends Pet {
  certificationId: string;
  trainingLevel: "basic" | "advanced" | "expert";
}

// ServiceDog must have all properties from Animal, Pet, and its own
const rex: ServiceDog = {
  name: "Rex",
  age: 4,
  owner: "Alice",
  isVaccinated: true,
  certificationId: "SD-2024-001",
  trainingLevel: "expert"
};

// Multiple inheritance — an interface can extend multiple interfaces
interface Printable {
  print(): void;
}

interface Exportable {
  export(format: "pdf" | "csv"): Blob;
}

interface Document extends Printable, Exportable {
  content: string;
  title: string;
}
```

---

### Declaration Merging

One of the unique capabilities of TypeScript interfaces (not available with type aliases) is **declaration merging**: if you declare two interfaces with the same name in the same scope, TypeScript merges them into a single interface containing all properties from both declarations.

This feature is primarily used in two scenarios: extending third-party library types (augmenting the interfaces they expose), and augmenting global types like `Window` or `HTMLElement`.

```ts
// Two separate declarations of the same interface
interface User {
  id: number;
  name: string;
}

interface User {
  email: string;     // TypeScript merges this into the first User interface
  active: boolean;
}

// Now User is as if it were declared with all four properties
const user: User = { id: 1, name: "Alice", email: "a@b.com", active: true };

// Practical use: extending global types
// Augmenting the Window interface to add a custom property
interface Window {
  __APP_CONFIG__: { apiUrl: string; version: string };
}
window.__APP_CONFIG__; // TypeScript now knows this exists

// Augmenting Express Request (common in Angular Universal server code)
declare global {
  namespace Express {
    interface Request {
      user?: User;
    }
  }
}
```

---

### Interfaces vs Type Aliases — When to Use Each

This is one of the most frequently debated questions among TypeScript developers. Both can describe object shapes. Both can be extended. Here is the precise breakdown of when each is appropriate.

**Use `interface` when:**
- You are describing the shape of an object or class.
- You want declaration merging capabilities (for library augmentation).
- You are defining contracts for class implementations.
- The type is likely to be extended by other interfaces.

**Use `type` alias when:**
- You are creating a union or intersection type (`A | B`, `A & B`).
- You are aliasing a primitive, function signature, tuple, or mapped type.
- You need to use conditional types or `infer`.
- The type is not meant to be extended or merged.

```ts
// Interfaces — for object shapes and class contracts
interface Repository<T> {
  findById(id: number): Promise<T | null>;
  findAll(): Promise<T[]>;
  save(entity: T): Promise<T>;
  delete(id: number): Promise<void>;
}

interface UserRepository extends Repository<User> {
  findByEmail(email: string): Promise<User | null>;
}

// Type aliases — for unions, functions, and computed types
type Status = "pending" | "active" | "archived";
type EventHandler<T = Event> = (event: T) => void;
type Nullable<T> = T | null;
type ApiResponse<T> = { data: T; status: number; message: string };

// Both work for simple object shapes — choose interface for consistency
// across a team, then use type for everything else
```

> **Angular-specific best practice:** The Angular community convention is to use `interface` for data models, component input/output types, and service contracts, and `type` for union types, conditional types, and complex computed types. When in doubt, `interface` is the safe default for object shapes — it reads more naturally and provides better error messages in some cases.

---

## 5.5 Functions in TypeScript

TypeScript's function type system is rich and precise. You can type every aspect of a function: its parameter types, whether parameters are optional, the type of `this` inside the function, and its return type. Getting function types right is essential because functions are the unit of every Angular service method, lifecycle hook, event handler, and guard.

### Parameter Types and Return Types

The most fundamental TypeScript skill for functions is annotating parameters (after the parameter name with `: Type`) and the return type (after the closing parenthesis with `: ReturnType`).

```ts
// Named function with typed parameters and return type
function divide(numerator: number, denominator: number): number {
  if (denominator === 0) throw new Error("Cannot divide by zero");
  return numerator / denominator;
}

// Arrow function — same annotation positions
const multiply = (a: number, b: number): number => a * b;

// A function returning an object
function createUser(name: string, role: UserRole): User {
  return { id: Date.now(), name, role, active: true };
}

// A function returning a Promise (async functions)
async function fetchUser(id: number): Promise<User> {
  const response = await fetch(`/api/users/${id}`);
  if (!response.ok) throw new Error(`HTTP ${response.status}`);
  return response.json() as Promise<User>;
}

// Void return type — for functions whose return value is not meaningful
function logError(error: Error): void {
  console.error(`[${new Date().toISOString()}]`, error.message);
}

// Never return type — for functions that never complete
function panic(message: string): never {
  throw new Error(message);
}
```

---

### Optional and Default Parameters

Optional parameters use `?` and make TypeScript treat that parameter as `Type | undefined` at the call site and inside the function body. Default parameters provide a fallback value and make the parameter optional without the `?`.

```ts
// Optional parameter — caller may omit it
function greet(name: string, title?: string): string {
  if (title) {
    return `Hello, ${title} ${name}!`;
  }
  return `Hello, ${name}!`;
}

greet("Alice");          // "Hello, Alice!"
greet("Alice", "Dr.");   // "Hello, Dr. Alice!"

// Default parameter — implicit optional, cleaner code
function createProduct(name: string, price: number, inStock: boolean = true): Product {
  return { name, price, inStock };
}

createProduct("Widget", 9.99);        // inStock defaults to true
createProduct("Gadget", 19.99, false); // inStock explicitly set to false

// Important: optional parameters must come AFTER required ones
function bad(optional?: string, required: number): void { } // Error!
function good(required: number, optional?: string): void { } // OK
```

---

### Rest Parameters

Rest parameters collect any number of additional arguments into a typed array. TypeScript ensures every element in the collected array has the declared type.

```ts
// Rest parameters — collect remaining arguments into an array
function sum(...numbers: number[]): number {
  return numbers.reduce((total, n) => total + n, 0);
}

sum(1, 2, 3, 4, 5); // 15
sum();               // 0 — empty array

// You can combine normal parameters with rest
function createEvent(type: string, target: Element, ...args: unknown[]): CustomEvent {
  return new CustomEvent(type, { detail: { target, args } });
}

// Typed rest with specific tuple spread (TS 4.0+)
function printf(format: string, ...args: [number, string, boolean]): string {
  return format.replace(/%[dsc]/g, () => String(args.shift()));
}
```

---

### Function Overloads

Function overloads let you define multiple signatures for a function that behaves differently depending on the type or number of its arguments. TypeScript checks callers against the overload signatures, but the actual implementation uses a broader "umbrella" signature that is not part of the public API.

The overload signatures come first, the implementation signature last. The implementation must be compatible with all overload signatures.

```ts
// Overload signatures (these are the public API — TypeScript checks callers against these)
function process(input: string): string;
function process(input: number): number;
function process(input: string[]): string[];

// Implementation signature (not part of the public API — wider type to handle all cases)
function process(input: string | number | string[]): string | number | string[] {
  if (typeof input === "string") {
    return input.toUpperCase();
  }
  if (typeof input === "number") {
    return input * 2;
  }
  return input.map(s => s.toUpperCase());
}

// Callers see only the overload signatures — clean, precise autocomplete
const result1 = process("hello");   // TypeScript knows return type is string
const result2 = process(5);         // TypeScript knows return type is number
const result3 = process(["a","b"]); // TypeScript knows return type is string[]

// Overloads on a class method — very common in Angular services
class QueryBuilder {
  select(): QueryBuilder;
  select(fields: string[]): QueryBuilder;
  select(fields?: string[]): QueryBuilder {
    this.fields = fields || ["*"];
    return this;
  }
}
```

---

### The `this` Parameter Type

TypeScript allows you to explicitly type what `this` refers to inside a function, catching errors where a method is called with the wrong `this` context. The `this` parameter is a fake parameter — it only exists in TypeScript, not in the compiled JavaScript.

```ts
// Without this parameter — TypeScript can't catch incorrect usage
function getFullName() {
  return `${this.firstName} ${this.lastName}`; // Error in strict mode: 'this' implicitly has type 'any'
}

// With this parameter — TypeScript ensures correct usage
interface Person {
  firstName: string;
  lastName: string;
}

function getFullName(this: Person): string {
  return `${this.firstName} ${this.lastName}`; // TypeScript knows `this` is Person
}

const alice: Person = { firstName: "Alice", lastName: "Smith" };
getFullName.call(alice);            // OK
getFullName.call({ name: "Bob" }); // Error: Argument doesn't match Person
```

---

### Typing Callbacks and Function Signatures

When you accept a function as a parameter (a callback) or store a function in a variable, you need to type its signature precisely.

```ts
// Inline function type syntax
function forEach<T>(items: T[], callback: (item: T, index: number) => void): void {
  items.forEach(callback);
}

// Type alias for a reusable function type
type Predicate<T> = (item: T) => boolean;
type Transformer<TInput, TOutput> = (input: TInput) => TOutput;
type AsyncAction<T> = (data: T) => Promise<void>;

// Angular-style: typing event handler callbacks
type ClickHandler = (event: MouseEvent) => void;
type ChangeHandler = (event: Event) => void;

// Interface with a call signature (the object itself is callable)
interface Formatter {
  (value: number, locale?: string): string;   // call signature
  currency: string;                            // also has properties
}

// How to type a function that can be either sync or async (Angular guards)
type GuardResult = boolean | Promise<boolean>;
type CanActivateFn = (route: ActivatedRouteSnapshot) => GuardResult;
```

---

## 5.6 Classes in TypeScript

TypeScript adds a powerful type layer on top of JavaScript classes. Access modifiers, abstract classes, and interface implementation give you a full OOP toolkit. In Angular, every component, service, directive, and pipe is a class — so understanding TypeScript's class features deeply is non-negotiable.

### Access Modifiers

TypeScript provides four access modifiers that control the visibility of class members. Critically, these are **compile-time checks only** — like all TypeScript features, they are erased at runtime.

**`public`** is the default modifier. Any code can access a public member. You rarely need to write `public` explicitly.

**`private`** makes a member only accessible within the class that declares it. Not even subclasses can access it.

**`protected`** makes a member accessible within the class and all its subclasses, but not from outside.

**`readonly`** means the property can only be assigned in the declaration or in the constructor. It cannot be modified afterward.

Note: TypeScript's `private` is different from JavaScript's `#private`. TypeScript's `private` is erased at compile time and is not truly private at runtime. JavaScript's `#private` syntax is enforced by the runtime. In modern Angular code, you will see both.

```ts
class BankAccount {
  public owner: string;             // accessible from anywhere
  private balance: number;          // only accessible within BankAccount
  protected accountNumber: string;  // accessible in BankAccount and subclasses
  readonly currency: string;        // can only be set in constructor

  constructor(owner: string, initialBalance: number, currency = "USD") {
    this.owner = owner;
    this.balance = initialBalance;
    this.accountNumber = Math.random().toString(36).slice(2);
    this.currency = currency;
  }

  deposit(amount: number): void {
    this.balance += amount; // OK — inside the class
  }

  getBalance(): number {
    return this.balance; // OK — reading private within the class
  }
}

const account = new BankAccount("Alice", 1000);
account.owner;                // OK — public
account.getBalance();         // OK — public method
account.balance;              // Error: Property 'balance' is private
account.currency = "EUR";     // Error: Cannot assign to 'currency' (readonly)
```

---

### Parameter Properties Shorthand

TypeScript provides a very convenient shorthand for the extremely common pattern of declaring a class property and assigning a constructor parameter to it. By adding an access modifier to a constructor parameter, TypeScript automatically creates the property and assigns the value in one step.

This is the primary style used throughout Angular — virtually every `@Injectable` service with constructor-injected dependencies uses this pattern.

```ts
// Without parameter properties — verbose
class UserService {
  private httpClient: HttpClient;
  private authService: AuthService;

  constructor(httpClient: HttpClient, authService: AuthService) {
    this.httpClient = httpClient;
    this.authService = authService;
  }
}

// With parameter properties shorthand — concise and idiomatic Angular
class UserService {
  constructor(
    private httpClient: HttpClient,
    private authService: AuthService,
    public readonly config: AppConfig
  ) {
    // Nothing needed here! TypeScript creates and assigns the properties automatically.
  }
}

// Both compile to identical JavaScript — the shorthand is purely a TypeScript convenience
```

---

### Abstract Classes

Abstract classes serve as a base that cannot be instantiated directly — they exist only to be extended. They can contain both abstract methods (declared but not implemented — subclasses must provide the implementation) and concrete methods (fully implemented and inherited by subclasses).

Abstract classes are the right tool when you have a family of classes that share some behavior but require each subclass to provide specific implementations of certain operations.

```ts
// An abstract class that defines the contract for all data exporters
abstract class DataExporter {
  // Abstract method — subclasses MUST implement this
  abstract formatData(data: unknown[]): string;

  // Abstract property — subclasses MUST implement this
  abstract readonly fileExtension: string;

  // Concrete method — inherited and shared by all subclasses
  export(data: unknown[], filename: string): Blob {
    const formatted = this.formatData(data); // calls the subclass's implementation
    const blob = new Blob([formatted], { type: this.mimeType });
    this.downloadFile(blob, `${filename}.${this.fileExtension}`);
    return blob;
  }

  private downloadFile(blob: Blob, name: string): void {
    const a = document.createElement("a");
    a.href = URL.createObjectURL(blob);
    a.download = name;
    a.click();
  }

  protected get mimeType(): string {
    return "text/plain"; // default — subclasses can override
  }
}

// Concrete subclass — must implement all abstract members
class CsvExporter extends DataExporter {
  readonly fileExtension = "csv";

  formatData(data: unknown[]): string {
    if (data.length === 0) return "";
    const headers = Object.keys(data[0] as object).join(",");
    const rows = data.map(row => Object.values(row as object).join(","));
    return [headers, ...rows].join("\n");
  }

  protected override get mimeType(): string {
    return "text/csv";
  }
}

class JsonExporter extends DataExporter {
  readonly fileExtension = "json";

  formatData(data: unknown[]): string {
    return JSON.stringify(data, null, 2);
  }
}

// new DataExporter(); // Error: Cannot create an instance of an abstract class
const csvExporter = new CsvExporter(); // OK
csvExporter.export(users, "users");    // Calls the shared concrete method
```

---

### Implementing Interfaces (`implements`)

When a class declares that it `implements` an interface, TypeScript verifies that the class provides all the required properties and methods defined in that interface. This is how you formally bind a class to a contract — making it impossible to accidentally miss implementing a required method.

This pattern is central to Angular: `CanActivateFn` guards, lifecycle hook interfaces (`OnInit`, `OnDestroy`, `OnChanges`), and `ControlValueAccessor` for custom form controls all use this mechanism.

```ts
interface Serializable {
  serialize(): string;
  deserialize(data: string): void;
}

interface Validatable {
  validate(): boolean;
  readonly errors: string[];
}

// A class can implement multiple interfaces
class UserForm implements Serializable, Validatable {
  name: string = "";
  email: string = "";
  errors: string[] = [];

  validate(): boolean {
    this.errors = [];
    if (!this.name) this.errors.push("Name is required");
    if (!this.email.includes("@")) this.errors.push("Invalid email");
    return this.errors.length === 0;
  }

  serialize(): string {
    return JSON.stringify({ name: this.name, email: this.email });
  }

  deserialize(data: string): void {
    const parsed = JSON.parse(data);
    this.name = parsed.name;
    this.email = parsed.email;
  }
}

// Angular lifecycle hook — the interface ensures you spell the method correctly
class MyComponent implements OnInit, OnDestroy {
  private subscription!: Subscription;

  ngOnInit(): void {
    this.subscription = this.dataService.data$.subscribe(data => {
      this.data = data;
    });
  }

  ngOnDestroy(): void {
    this.subscription.unsubscribe();
  }
}
```

---

### Static Members

Static properties and methods belong to the **class itself**, not to instances. They are accessed on the class name, not on an object created with `new`. They are useful for factory methods, utility functions tied to a class, shared configuration, and counters.

```ts
class IdGenerator {
  private static nextId = 1;  // shared across all instances

  static generate(): number {
    return IdGenerator.nextId++;
  }

  static reset(): void {
    IdGenerator.nextId = 1;
  }
}

IdGenerator.generate(); // 1
IdGenerator.generate(); // 2
IdGenerator.generate(); // 3
IdGenerator.reset();
IdGenerator.generate(); // 1

// Static factory method pattern — a very common OOP pattern
class Color {
  private constructor(
    public readonly r: number,
    public readonly g: number,
    public readonly b: number
  ) {}

  // Static factory methods are the only way to create a Color
  static fromHex(hex: string): Color {
    const r = parseInt(hex.slice(1, 3), 16);
    const g = parseInt(hex.slice(3, 5), 16);
    const b = parseInt(hex.slice(5, 7), 16);
    return new Color(r, g, b);
  }

  static fromRgb(r: number, g: number, b: number): Color {
    return new Color(r, g, b);
  }

  toHex(): string {
    return `#${[this.r, this.g, this.b].map(v => v.toString(16).padStart(2, "0")).join("")}`;
  }
}

const red = Color.fromHex("#FF0000");
const blue = Color.fromRgb(0, 0, 255);
// new Color(255, 0, 0); // Error: Constructor is private
```

---

## 🔑 Key Takeaways — Part 2

Interfaces are the backbone of TypeScript's structural type system — they define the shape that data must have, and TypeScript verifies conformance everywhere the shape is used. The `?` optional modifier and `readonly` modifier on interface properties are the most frequently used tools for expressing real-world data shapes precisely. Declaration merging makes interfaces uniquely suited for extending third-party types, which is why library type definitions use interfaces heavily. For class contracts and object shapes, use `interface`; for union types, function signatures, and computed types, use `type`.

Function overloads let you express the relationship between different input and output types precisely — they are especially valuable in service methods that behave differently based on what you pass in. The `this` parameter type is TypeScript's mechanism for catching the most common JavaScript `this`-binding bugs at compile time.

In classes, access modifiers (`private`, `protected`, `readonly`) enforce encapsulation at the type level. The parameter properties shorthand (`constructor(private x: Service)`) is idiomatic Angular and should become second nature. Abstract classes define a contract plus shared implementation for a family of subclasses — use them when a base class should not be instantiated directly but when concrete subclasses share real behavior.

---

*Part 2 of 5 — Phase 5 TypeScript Mastery | Next: [Generics & Advanced Types](./part3-generics-advanced-types.md)*

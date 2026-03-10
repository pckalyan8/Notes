# ⚡ Phase 4 — Part 3: Objects · Prototypes · Classes · Arrays
> Sections 4.7 · 4.8 · 4.9 | Angular Frontend Mastery Roadmap

---

## 4.7 Objects & Prototypes

Objects are the most fundamental data structure in JavaScript. Nearly everything — arrays, functions, instances of classes — is an object under the hood. A deep understanding of how objects and prototypes work will make you a significantly better JavaScript developer.

### Creating Objects

There are three primary ways to create objects in JavaScript.

**Object literals** are the most common and most readable. They are how you will create most plain data objects day-to-day.

```js
const user = {
  name: "Alice",
  age: 30,
  active: true,
  greet() {        // method shorthand (ES2015)
    return `Hi, I'm ${this.name}`;
  }
};
```

**`Object.create()`** lets you create an object with a specific prototype. It gives you fine-grained control over the prototype chain and is used in some inheritance patterns.

```js
const animalProto = {
  speak() {
    return `${this.name} makes a sound`;
  }
};

const dog = Object.create(animalProto);
dog.name = "Rex";
dog.speak(); // "Rex makes a sound" — found via prototype

// Object.create(null) creates an object with NO prototype
// Useful for pure hash maps with no inherited properties
const pureMap = Object.create(null);
```

**Constructor functions** are how objects were created before the `class` keyword. Using `new` with a function creates a new object, sets its prototype, runs the function body, and returns the object.

```js
function Person(name, age) {
  this.name = name;
  this.age = age;
}
Person.prototype.greet = function() {
  return `Hi, I'm ${this.name}`;
};
const alice = new Person("Alice", 30);
alice.greet(); // "Hi, I'm Alice"
```

### Property Descriptors

Every property of an object has an associated *descriptor* — a set of attributes that control how the property behaves. You can inspect and define these with `Object.getOwnPropertyDescriptor` and `Object.defineProperty`.

A property descriptor has the following attributes. `value` is the property's value. `writable` controls whether `value` can be changed with an assignment. `enumerable` controls whether the property shows up in `for...in` loops and `Object.keys()`. `configurable` controls whether the property can be deleted or its descriptor modified.

```js
const config = {};

Object.defineProperty(config, "API_KEY", {
  value: "abc123",
  writable: false,      // cannot be changed
  enumerable: false,    // hidden from Object.keys(), for...in
  configurable: false   // cannot be deleted or redefined
});

config.API_KEY;       // "abc123"
config.API_KEY = "x"; // silently fails (or TypeError in strict mode)
Object.keys(config);  // [] — property is non-enumerable
```

### Getters and Setters

Getters and setters let you define computed properties — properties that look like values but execute logic when read or written.

```js
const temperature = {
  _celsius: 20, // convention: underscore means "internal"

  get fahrenheit() {
    return this._celsius * 9 / 5 + 32;
  },

  set fahrenheit(value) {
    this._celsius = (value - 32) * 5 / 9;
  }
};

console.log(temperature.fahrenheit); // 68 — runs the getter
temperature.fahrenheit = 77;          // runs the setter
console.log(temperature._celsius);   // 25

// Angular uses getters extensively in signals and component inputs
```

### The Prototype Chain

Every object in JavaScript has an internal link to another object called its *prototype*. When you access a property on an object and that property does not exist on the object itself, JavaScript follows the prototype link and looks there. This continues up the chain until it reaches `null` (the end of every chain).

```js
const animal = {
  breathe() { return "breathing"; }
};

const dog = Object.create(animal);
dog.bark = function() { return "woof"; };

const poodle = Object.create(dog);
poodle.name = "Fifi";

// Property lookup for poodle.breathe():
// 1. Check poodle itself — not found
// 2. Check dog (poodle's prototype) — not found
// 3. Check animal (dog's prototype) — found! Returns "breathing"
poodle.breathe(); // "breathing"
poodle.bark();    // "woof" — found on dog
poodle.name;      // "Fifi" — found on poodle itself
```

When you create a function and use `new` to create instances, those instances' prototype is set to `Function.prototype`. This is how the prototype chain for class-based objects works.

```js
function Dog(name) { this.name = name; }
Dog.prototype.bark = function() { return "woof"; };

const rex = new Dog("Rex");
rex.bark();  // found on Dog.prototype via the chain
rex.hasOwnProperty("name"); // true — name is on the instance
rex.hasOwnProperty("bark"); // false — bark is on the prototype
```

### Essential Object Methods

These are the Object static methods you will use most frequently in real-world Angular development.

```js
const user = { name: "Alice", age: 30, role: "admin" };

// Enumeration
Object.keys(user);           // ["name", "age", "role"]
Object.values(user);         // ["Alice", 30, "admin"]
Object.entries(user);        // [["name","Alice"], ["age",30], ["role","admin"]]

// Merging and copying (shallow)
const defaults = { role: "viewer", active: true };
const merged = Object.assign({}, defaults, user);
// same as spread: { ...defaults, ...user }

// Type-safe property check
Object.hasOwn(user, "name"); // true (modern — use over hasOwnProperty)
Object.hasOwn(user, "email"); // false

// Freezing — prevents all modifications (shallow)
const config = Object.freeze({ apiUrl: "https://api.example.com" });
config.apiUrl = "other"; // silently ignored (TypeError in strict mode)

// Sealing — allows modification but prevents adding/deleting properties
const sealed = Object.seal({ x: 1, y: 2 });
sealed.x = 99;  // allowed
sealed.z = 3;   // silently ignored
delete sealed.x; // silently ignored

// Creating from entries
const entries = [["a", 1], ["b", 2]];
Object.fromEntries(entries); // { a: 1, b: 2 }
Object.fromEntries(map);     // convert Map to plain object
```

### Destructuring Objects and Arrays

Destructuring is a concise syntax for extracting values from objects and arrays into named variables. You will use this constantly in Angular.

```js
// Object destructuring
const user = { name: "Alice", age: 30, role: "admin" };
const { name, age } = user;
console.log(name); // "Alice"

// Rename during destructuring
const { name: userName, role: userRole } = user;
console.log(userName); // "Alice"

// Default values
const { name, email = "no-email" } = user;
console.log(email); // "no-email" — user has no email property

// Nested destructuring
const config = { server: { host: "localhost", port: 3000 } };
const { server: { host, port } } = config;
console.log(host); // "localhost"

// Array destructuring
const [first, second, , fourth] = [1, 2, 3, 4];
// skip elements with empty commas

// Swap variables without a temp variable
let a = 1, b = 2;
[a, b] = [b, a]; // a = 2, b = 1

// Rest in destructuring
const { name: n, ...rest } = user;
console.log(rest); // { age: 30, role: "admin" }

// Function parameter destructuring
function displayUser({ name, role = "viewer" }) {
  console.log(`${name} (${role})`);
}
displayUser(user); // "Alice (admin)"
```

### Computed Property Names

You can use any expression as a property key by wrapping it in square brackets.

```js
const field = "name";
const prefix = "user";

const obj = {
  [field]: "Alice",                    // { name: "Alice" }
  [`${prefix}Age`]: 30,               // { userAge: 30 }
  [Symbol.iterator]() { ... }         // Symbol as a key
};

// Very useful for dynamic form building in Angular
const formUpdates = { [fieldName]: newValue };
this.form.patchValue(formUpdates);
```

---

## 4.8 JavaScript Classes

ES2015 classes are *syntactic sugar* over the prototype-based system. They do not introduce a new OOP model — they simply provide a cleaner, more familiar syntax for creating objects and setting up prototype chains. Understanding this is important so you are not surprised when class instances behave like prototype-based objects.

### Class Basics

```js
class Animal {
  // The constructor runs when you call `new Animal()`
  constructor(name, sound) {
    this.name = name;
    this.sound = sound;
  }

  // Instance method — added to Animal.prototype
  speak() {
    return `${this.name} says ${this.sound}`;
  }

  // Static method — belongs to the class itself, not instances
  static create(name, sound) {
    return new Animal(name, sound);
  }
}

const cat = new Animal("Whiskers", "meow");
cat.speak();             // "Whiskers says meow"
Animal.create("Rex", "woof"); // static call
```

### Private Fields and Methods

ES2022 introduced true private fields using the `#` prefix. These are not accessible outside the class, unlike the older convention of using `_` prefix (which was just a naming convention, not enforced).

```js
class BankAccount {
  #balance = 0;        // private field
  #transactionLog = []; // private field

  constructor(initialBalance) {
    this.#balance = initialBalance;
  }

  // Public method — the only way to interact with private data
  deposit(amount) {
    if (amount <= 0) throw new Error("Amount must be positive");
    this.#balance += amount;
    this.#logTransaction("deposit", amount);
    return this;  // allows method chaining
  }

  withdraw(amount) {
    if (amount > this.#balance) throw new Error("Insufficient funds");
    this.#balance -= amount;
    this.#logTransaction("withdrawal", amount);
    return this;
  }

  get balance() { return this.#balance; } // public getter for private field

  #logTransaction(type, amount) {  // private method
    this.#transactionLog.push({ type, amount, date: new Date() });
  }
}

const account = new BankAccount(100);
account.deposit(50).withdraw(30); // method chaining
account.balance; // 120
account.#balance; // SyntaxError — truly private
```

### Inheritance with `extends` and `super`

Classes can inherit from other classes using `extends`. The `super` keyword calls the parent class's constructor or methods.

```js
class Vehicle {
  constructor(make, model, year) {
    this.make = make;
    this.model = model;
    this.year = year;
  }

  describe() {
    return `${this.year} ${this.make} ${this.model}`;
  }

  start() {
    return "Vroom";
  }
}

class ElectricVehicle extends Vehicle {
  #batteryLevel = 100;

  constructor(make, model, year, range) {
    super(make, model, year); // Must call super() before using `this`
    this.range = range;
  }

  describe() {
    return `${super.describe()} (Electric, range: ${this.range}km)`;
    // super.describe() calls the parent's describe method
  }

  charge() {
    this.#batteryLevel = 100;
    return this;
  }
}

const tesla = new ElectricVehicle("Tesla", "Model S", 2024, 600);
tesla.describe(); // "2024 Tesla Model S (Electric, range: 600km)"
tesla.start();    // "Vroom" — inherited from Vehicle
```

### Abstract Class Pattern

JavaScript does not have built-in abstract classes, but you can simulate them by throwing errors in the base class constructor or methods that must be overridden.

```js
class Shape {
  constructor() {
    if (new.target === Shape) {
      throw new Error("Shape is abstract — use a subclass");
    }
  }

  // Abstract method — subclasses must override
  area() {
    throw new Error("area() must be implemented by subclass");
  }

  describe() {
    return `This shape has area: ${this.area()}`; // uses polymorphism
  }
}

class Circle extends Shape {
  constructor(radius) {
    super();
    this.radius = radius;
  }

  area() { return Math.PI * this.radius ** 2; }
}

class Rectangle extends Shape {
  constructor(w, h) {
    super();
    this.width = w;
    this.height = h;
  }

  area() { return this.width * this.height; }
}

// new Shape();       // Error — abstract
const c = new Circle(5);
c.describe(); // "This shape has area: 78.53..."
```

### Mixins Pattern

JavaScript only supports single inheritance (a class can only `extend` one class), but you can simulate multiple inheritance through mixins. A mixin is a function that takes a base class and returns a new class with added functionality.

```js
// Mixin — adds serialization capability to any class
const Serializable = (Base) => class extends Base {
  serialize() {
    return JSON.stringify(this);
  }
  static deserialize(json) {
    return Object.assign(new this(), JSON.parse(json));
  }
};

// Mixin — adds validation capability
const Validatable = (Base) => class extends Base {
  validate() {
    return Object.keys(this).every(key => this[key] !== null && this[key] !== undefined);
  }
};

class User {
  constructor(name, email) {
    this.name = name;
    this.email = email;
  }
}

// Apply multiple mixins
class EnhancedUser extends Serializable(Validatable(User)) {}

const user = new EnhancedUser("Alice", "alice@example.com");
user.validate();  // true
user.serialize(); // '{"name":"Alice","email":"alice@example.com"}'
```

---

## 4.9 Arrays & Array Methods

Arrays are ordered, indexed collections. In JavaScript they are objects with special behavior — you can mix types, they are dynamically sized, and they come with a rich set of built-in methods. Mastering array methods is essential because they are used throughout Angular templates, services, and RxJS pipelines.

### Creating Arrays

```js
// Literal syntax — most common
const fruits = ["apple", "banana", "cherry"];

// Array.from() — create from any iterable or array-like
Array.from("hello");         // ["h","e","l","l","o"]
Array.from({ length: 5 }, (_, i) => i * 2); // [0, 2, 4, 6, 8]
Array.from(new Set([1,2,2,3])); // [1, 2, 3] — deduplicate

// Array.of() — creates array from arguments (unlike new Array())
Array.of(7);       // [7] — single element
new Array(7);      // [empty × 7] — array of 7 empty slots!
```

### Mutating vs Non-Mutating Methods

This is the critical distinction. *Mutating* methods change the original array. *Non-mutating* methods return a new array and leave the original unchanged. In modern JavaScript and Angular, you should strongly prefer non-mutating methods to avoid unexpected side effects.

**Mutating methods:**

```js
const arr = [1, 2, 3, 4, 5];

// push / pop — add/remove from END
arr.push(6);    // arr = [1,2,3,4,5,6], returns new length (6)
arr.pop();      // arr = [1,2,3,4,5], returns removed element (6)

// unshift / shift — add/remove from BEGINNING
arr.unshift(0); // arr = [0,1,2,3,4,5], returns new length
arr.shift();    // arr = [1,2,3,4,5], returns removed element (0)

// splice — add/remove/replace at any position
arr.splice(2, 1);         // remove 1 element at index 2: arr = [1,2,4,5]
arr.splice(2, 0, 99, 100); // insert at index 2: arr = [1,2,99,100,4,5]
arr.splice(1, 2, 50);     // replace 2 elements with 50

// sort — sorts IN PLACE (also returns the array)
[3,1,2].sort();                  // [1,2,3] — lexicographic by default
[10,9,2,1].sort((a,b) => a - b); // [1,2,9,10] — numeric ascending
[10,9,2,1].sort((a,b) => b - a); // [10,9,2,1] — numeric descending

// reverse — reverses IN PLACE
[1,2,3].reverse(); // [3,2,1]

// fill — fills a range with a value
new Array(5).fill(0);    // [0,0,0,0,0]
[1,2,3,4].fill(0, 1, 3); // [1,0,0,4]
```

**Non-mutating methods — these are the heart of functional array operations:**

```js
const numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10];

// map — transform every element, returns same length
const doubled = numbers.map(n => n * 2); // [2,4,6,8,10,12,14,16,18,20]
const users = ids.map(id => ({ id, loaded: false }));

// filter — keep elements matching predicate, returns shorter array
const evens = numbers.filter(n => n % 2 === 0); // [2,4,6,8,10]
const activeUsers = users.filter(u => u.isActive);

// reduce — accumulate all elements into a single value
const sum = numbers.reduce((acc, n) => acc + n, 0); // 55
const max = numbers.reduce((acc, n) => n > acc ? n : acc, -Infinity);

// Group items into an object (common pattern)
const grouped = users.reduce((acc, user) => {
  const key = user.role;
  (acc[key] = acc[key] || []).push(user);
  return acc;
}, {});
// { admin: [...], viewer: [...] }

// find / findIndex — first match or -1/undefined
const first = numbers.find(n => n > 5);      // 6
const idx = numbers.findIndex(n => n > 5);   // 5

// every / some — boolean checks
numbers.every(n => n > 0);  // true — all positive
numbers.some(n => n > 9);   // true — at least one > 9

// includes — does it contain this value?
numbers.includes(5); // true

// slice — copy a portion (non-destructive)
numbers.slice(2, 5); // [3,4,5] (start inclusive, end exclusive)
numbers.slice(-3);   // [8,9,10] (last 3 elements)

// flat / flatMap — flatten nested arrays
[1,[2,3],[4,[5]]].flat();    // [1,2,3,4,[5]] (one level deep)
[1,[2,3],[4,[5]]].flat(Infinity); // [1,2,3,4,5] (fully flattened)

// flatMap = map + flat(1) — very useful for one-to-many transformations
const sentences = ["Hello world", "foo bar"];
sentences.flatMap(s => s.split(" ")); // ["Hello","world","foo","bar"]

// concat — join arrays non-destructively
[1,2].concat([3,4], [5,6]); // [1,2,3,4,5,6]

// join — convert to string
["a","b","c"].join(" - "); // "a - b - c"

// indexOf / lastIndexOf
[1,2,3,2,1].indexOf(2);     // 1 (first occurrence)
[1,2,3,2,1].lastIndexOf(2); // 3 (last occurrence)
```

### ES2023 Non-Mutating Counterparts

ES2023 introduced non-mutating alternatives to the historically mutating `sort`, `reverse`, and `splice` methods. Use these in modern code to avoid accidentally mutating shared arrays.

```js
const original = [3, 1, 2];

// toSorted — non-mutating sort
const sorted = original.toSorted((a, b) => a - b); // [1,2,3]
// original is still [3,1,2]

// toReversed — non-mutating reverse
const reversed = original.toReversed(); // [2,1,3]

// toSpliced — non-mutating splice
const spliced = original.toSpliced(1, 1, 99); // [3,99,2]

// with — replace element at index non-mutably
const updated = original.with(0, 100); // [100,1,2]
```

### Array Destructuring and Rest

```js
const [a, b, c] = [1, 2, 3];
const [first, , third] = [1, 2, 3];          // skip middle
const [head, ...tail] = [1, 2, 3, 4, 5];     // head=1, tail=[2,3,4,5]

// Swap variables
let x = 1, y = 2;
[x, y] = [y, x];

// Get specific items from a function return
function getCoords() { return [40.7128, -74.0060]; }
const [latitude, longitude] = getCoords();
```

### Iterating Arrays

Three ways to iterate. `forEach` for side effects. `for...of` for side effects with `break`/`continue` support. The functional methods (`map`, `filter`, etc.) for transformations.

```js
const items = ["a", "b", "c"];

// forEach — side effects, no break/continue, no return value
items.forEach((item, index) => console.log(index, item));

// for...of — can use break/continue, more flexible
for (const item of items) {
  if (item === "b") continue; // skip b
  console.log(item);
}

// Iterate with index using entries()
for (const [index, item] of items.entries()) {
  console.log(index, item);
}

// Never use for...in on arrays — it iterates keys (as strings!) and
// also inherits enumerable properties from the prototype
```

### Array-Like Objects

Some browser APIs return objects that *look* like arrays (they have a `length` property and numeric indices) but are not actual arrays. Common examples include `NodeList`, `HTMLCollection`, `arguments`, and `FileList`. Convert them to real arrays to use array methods.

```js
// NodeList from querySelectorAll is array-like but not an Array
const nodes = document.querySelectorAll(".item");
// nodes.map(...)   // TypeError: nodes.map is not a function

// Convert to array
const nodeArray = Array.from(nodes);
const nodeArray2 = [...nodes]; // spread works on iterables

nodeArray.map(node => node.textContent); // Now works!
```

> **Angular note:** You encounter array methods everywhere in Angular. Services transform data with `map` and `filter`. NgRx reducers use spread and non-mutating methods to return new state. `@for` in templates iterates arrays. `FormArray` contains an array of controls. Being completely fluent with these methods is non-negotiable.

---

## 🔑 Key Takeaways — Part 3

The prototype chain is not just an academic concept — it explains how class inheritance works, why `instanceof` behaves as it does, and how Angular's dependency injection resolves services. Private class fields (`#`) give you genuine encapsulation. For arrays, the most important skill is knowing which methods are mutating and which are not: always prefer non-mutating methods (`map`, `filter`, `reduce`, `toSorted`, `toReversed`) when working with shared state, which is the norm in Angular services and NgRx reducers. `reduce` is the most powerful array method — it can implement `map`, `filter`, grouping, and more. Master it thoroughly.

---

*Part 3 of 6 — Phase 4 JavaScript Mastery | Next: [Async JavaScript, Error Handling & Modules](./part4-async-errors-modules.md)*

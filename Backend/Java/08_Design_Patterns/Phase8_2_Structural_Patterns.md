# Phase 8.2 — Structural Design Patterns in Java

> **What are Structural Patterns?**
> Structural patterns are about **composing classes and objects into larger structures** while keeping those structures flexible and efficient. They describe how to assemble objects and classes into bigger systems — like Lego bricks — without coupling components tightly. The key question they answer is: *"How do these pieces fit together?"*

---

## 1. Adapter Pattern

### Theory

The Adapter pattern acts as a **bridge between two incompatible interfaces**. It wraps an existing class (the *adaptee*) with a new interface (the *target*) that the client expects, translating calls between the two without changing either the client or the adaptee.

Think of it like a travel plug adapter — your laptop's plug hasn't changed, the wall socket hasn't changed, but the adapter in between makes them work together.

**When to use it:**
- You want to use an existing class but its interface doesn't match what you need
- You're integrating a third-party library with an incompatible interface
- You're building a legacy integration layer

### Example — Adapting a Legacy Payment System

```java
// The TARGET interface — what the client code expects
interface ModernPaymentProcessor {
    boolean processPayment(String currency, double amount);
    String getTransactionId();
}

// The ADAPTEE — existing legacy class with a different interface
// Imagine this is a third-party library you cannot modify
class LegacyBankAPI {
    public void initiateTransfer(int amountInCents, String currencyCode) {
        System.out.println("Legacy API: Transferring "
            + amountInCents + " cents in " + currencyCode);
    }

    public String fetchLastTransferReference() {
        return "LEGACY-TXN-" + System.currentTimeMillis();
    }
}

// The ADAPTER — implements the target interface, wraps the adaptee
class LegacyBankAdapter implements ModernPaymentProcessor {

    // Adapter holds a reference to the adaptee (Object Adapter via composition)
    private final LegacyBankAPI legacyAPI;

    public LegacyBankAdapter(LegacyBankAPI legacyAPI) {
        this.legacyAPI = legacyAPI;
    }

    @Override
    public boolean processPayment(String currency, double amount) {
        // TRANSLATE: modern interface expects double dollars → legacy expects int cents
        int amountInCents = (int)(amount * 100);
        legacyAPI.initiateTransfer(amountInCents, currency.toUpperCase());
        return true; // assume success
    }

    @Override
    public String getTransactionId() {
        // TRANSLATE: modern interface expects getTransactionId → legacy has different method name
        return legacyAPI.fetchLastTransferReference();
    }
}

// CLIENT — only knows about ModernPaymentProcessor, unaware of legacy system
class OnlineStore {
    private final ModernPaymentProcessor processor;

    public OnlineStore(ModernPaymentProcessor processor) {
        this.processor = processor;
    }

    public void checkout(double amount) {
        boolean success = processor.processPayment("USD", amount);
        if (success) {
            System.out.println("Payment successful! Ref: " + processor.getTransactionId());
        }
    }
}

public class AdapterDemo {
    public static void main(String[] args) {
        // Wrap the legacy API in our adapter — client never knows
        LegacyBankAPI legacyAPI = new LegacyBankAPI();
        ModernPaymentProcessor adapter = new LegacyBankAdapter(legacyAPI);

        OnlineStore store = new OnlineStore(adapter);
        store.checkout(49.99);
    }
}
```

**Output:**
```
Legacy API: Transferring 4999 cents in USD
Payment successful! Ref: LEGACY-TXN-1718000000000
```

The `OnlineStore` client never changes. Tomorrow if you replace the legacy API with a modern one, you just swap the adapter — the store code is untouched.

---

### Two Flavors of Adapter

There are two implementation styles. The **Object Adapter** (used above) uses *composition* — the adapter holds a reference to the adaptee. The **Class Adapter** uses *multiple inheritance* (not possible in Java without interfaces) — the adapter extends the adaptee class. In Java, prefer the Object Adapter because it's more flexible (you can even adapt subclasses of the adaptee).

---

### ⚠️ Important Points & Best Practices

- Java's standard library uses this extensively. `Arrays.asList()` adapts an array to a `List`. `InputStreamReader` adapts a byte-based `InputStream` to a character-based `Reader`.
- Adapter is purely structural — it doesn't change behavior, only the interface.
- Don't confuse Adapter (makes incompatible interfaces work together) with Facade (simplifies a complex subsystem).

---

## 2. Bridge Pattern

### Theory

The Bridge pattern **decouples an abstraction from its implementation** so that the two can vary independently. Instead of having an explosion of subclasses for every combination of features, you split the class hierarchy into two separate hierarchies — an *abstraction* and an *implementation* — connected by a bridge (a reference/composition).

**The problem without Bridge:** Suppose you have `Shape` (Circle, Square) and `Color` (Red, Blue). Without Bridge you'd need `RedCircle`, `BlueCircle`, `RedSquare`, `BlueSquare` — 4 classes for 2×2. Adding a `Triangle` or `Green` multiplies the count further. With Bridge you have 2 + 2 = 4 classes total regardless of how many you add.

**When to use it:**
- You want to avoid a permanent binding between abstraction and implementation
- Both the abstraction and implementation should be extensible via subclassing
- Switching implementations at runtime

### Example — Shapes and Renderers

```java
// IMPLEMENTATION hierarchy — the "bridge" interface
interface Renderer {
    void renderCircle(double radius);
    void renderSquare(double side);
}

// Concrete Implementations
class VectorRenderer implements Renderer {
    @Override
    public void renderCircle(double radius) {
        System.out.printf("Drawing circle (radius=%.1f) as vector paths%n", radius);
    }
    @Override
    public void renderSquare(double side) {
        System.out.printf("Drawing square (side=%.1f) as vector paths%n", side);
    }
}

class RasterRenderer implements Renderer {
    @Override
    public void renderCircle(double radius) {
        System.out.printf("Drawing circle (radius=%.1f) as pixel grid%n", radius);
    }
    @Override
    public void renderSquare(double side) {
        System.out.printf("Drawing square (side=%.1f) as pixel grid%n", side);
    }
}

// ABSTRACTION hierarchy — holds a reference to Renderer (the bridge)
abstract class Shape {
    // The bridge — abstraction references implementation
    protected final Renderer renderer;

    public Shape(Renderer renderer) {
        this.renderer = renderer;
    }

    public abstract void draw();
    public abstract void resize(double factor);
}

// Refined Abstractions — extend Shape, delegate rendering to the bridge
class Circle extends Shape {
    private double radius;

    public Circle(Renderer renderer, double radius) {
        super(renderer);
        this.radius = radius;
    }

    @Override
    public void draw() {
        renderer.renderCircle(radius); // delegates to whichever renderer was injected
    }

    @Override
    public void resize(double factor) {
        radius *= factor;
    }
}

class Square extends Shape {
    private double side;

    public Square(Renderer renderer, double side) {
        super(renderer);
        this.side = side;
    }

    @Override
    public void draw() {
        renderer.renderSquare(side);
    }

    @Override
    public void resize(double factor) {
        side *= factor;
    }
}

public class BridgeDemo {
    public static void main(String[] args) {
        // Same Circle shape, two different renderers — no new subclasses needed
        Shape vectorCircle = new Circle(new VectorRenderer(), 5.0);
        Shape rasterCircle = new Circle(new RasterRenderer(), 5.0);

        vectorCircle.draw();
        rasterCircle.draw();

        // Switch renderers without touching Shape logic
        Shape square = new Square(new VectorRenderer(), 4.0);
        square.draw();
        square.resize(2.0);
        square.draw();
    }
}
```

---

### ⚠️ Important Points & Best Practices

- Bridge is about having two orthogonal dimensions of variation. If you only have one dimension, use simple inheritance instead.
- JDBC is a textbook Bridge example: `java.sql.DriverManager` (abstraction) is decoupled from the vendor-specific driver (implementation). Your SQL code doesn't change when you switch from MySQL to PostgreSQL.
- Don't confuse Bridge with Adapter: Adapter makes unrelated classes work together; Bridge is designed up-front to separate concerns.

---

## 3. Composite Pattern

### Theory

The Composite pattern lets you **compose objects into tree structures** and then work with these structures as if they were individual objects. Both single objects (leaves) and groups of objects (composites) implement the same interface, so clients treat them uniformly.

**Real-world analogy:** A company org chart. An employee is an individual. A department contains employees and other departments. You can call `getSalary()` on both an individual and a department — for a department it recursively sums all salaries.

**When to use it:**
- You need to represent part-whole hierarchies (trees)
- Clients should treat individual objects and compositions uniformly
- File systems, UI component trees, menu hierarchies, document elements

### Example — File System Tree

```java
import java.util.ArrayList;
import java.util.List;

// COMPONENT interface — shared by both files and directories
interface FileSystemComponent {
    String getName();
    long getSize();  // in bytes
    void display(String indent);
}

// LEAF — has no children
class File implements FileSystemComponent {
    private final String name;
    private final long size;

    public File(String name, long size) {
        this.name = name;
        this.size = size;
    }

    @Override public String getName() { return name; }
    @Override public long getSize()   { return size; }

    @Override
    public void display(String indent) {
        System.out.printf("%s📄 %s (%d bytes)%n", indent, name, size);
    }
}

// COMPOSITE — can hold children (both files and other directories)
class Directory implements FileSystemComponent {
    private final String name;
    private final List<FileSystemComponent> children = new ArrayList<>();

    public Directory(String name) {
        this.name = name;
    }

    public void add(FileSystemComponent component) {
        children.add(component);
    }

    public void remove(FileSystemComponent component) {
        children.remove(component);
    }

    @Override public String getName() { return name; }

    // Recursively sums the sizes of all children
    @Override
    public long getSize() {
        return children.stream()
                       .mapToLong(FileSystemComponent::getSize)
                       .sum();
    }

    @Override
    public void display(String indent) {
        System.out.printf("%s📁 %s/ (%d bytes total)%n", indent, name, getSize());
        // Recursively display children — uniform treatment of File and Directory
        for (FileSystemComponent child : children) {
            child.display(indent + "  ");
        }
    }
}

public class CompositeDemo {
    public static void main(String[] args) {
        // Build a tree
        Directory root = new Directory("project");

        Directory src = new Directory("src");
        src.add(new File("Main.java", 1200));
        src.add(new File("App.java", 800));

        Directory resources = new Directory("resources");
        resources.add(new File("config.yml", 300));
        resources.add(new File("logo.png", 45000));

        root.add(src);
        root.add(resources);
        root.add(new File("README.md", 500));

        // Client calls display() — works the same on root, subdirectory, or file
        root.display("");

        System.out.println("\nTotal project size: " + root.getSize() + " bytes");
        System.out.println("src folder size: " + src.getSize() + " bytes");
    }
}
```

**Output:**
```
📁 project/ (47800 bytes total)
  📁 src/ (2000 bytes total)
    📄 Main.java (1200 bytes)
    📄 App.java (800 bytes)
  📁 resources/ (45300 bytes total)
    📄 config.yml (300 bytes)
    📄 logo.png (45000 bytes)
  📄 README.md (500 bytes)
```

---

### ⚠️ Important Points & Best Practices

- The strength of Composite is **uniformity** — `getSize()` works on a single file or an entire directory tree through the same interface.
- Swing UI components use this pattern: `JPanel` is a composite that can hold other components including other `JPanel` objects.
- Be careful with operations that only make sense on leaves vs. composites (e.g., `add(child)` on a `File` makes no sense). One approach is to throw `UnsupportedOperationException` in the base interface; another is to only declare `add/remove` on the `Composite` type.

---

## 4. Decorator Pattern

### Theory

The Decorator pattern **attaches additional responsibilities to an object dynamically**. It provides a flexible alternative to subclassing for extending functionality. Decorators wrap the original object, adding behavior before or after delegating to it.

Think of it like assembling a coffee order: you start with plain coffee, then "decorate" it with milk, then sugar, then whipped cream — each wrapper adds something without changing the original coffee object.

**When to use it:**
- You want to add behavior to individual objects without affecting others of the same class
- Subclassing would lead to an explosion of classes
- You need to combine multiple independent enhancements

### Example — Text Formatter (and then the classic I/O example)

```java
// COMPONENT interface — the core contract
interface TextProcessor {
    String process(String text);
}

// CONCRETE COMPONENT — the base implementation
class PlainTextProcessor implements TextProcessor {
    @Override
    public String process(String text) {
        return text; // no modification
    }
}

// BASE DECORATOR — implements same interface, wraps another TextProcessor
abstract class TextDecorator implements TextProcessor {
    // The wrapped component — this is the key structural element
    protected final TextProcessor wrapped;

    public TextDecorator(TextProcessor wrapped) {
        this.wrapped = wrapped;
    }

    @Override
    public String process(String text) {
        return wrapped.process(text); // default: just delegate
    }
}

// CONCRETE DECORATORS — each adds one specific behavior
class UpperCaseDecorator extends TextDecorator {
    public UpperCaseDecorator(TextProcessor wrapped) {
        super(wrapped);
    }

    @Override
    public String process(String text) {
        return wrapped.process(text).toUpperCase(); // add behavior, then delegate
    }
}

class TrimDecorator extends TextDecorator {
    public TrimDecorator(TextProcessor wrapped) {
        super(wrapped);
    }

    @Override
    public String process(String text) {
        return wrapped.process(text.trim()); // trim before delegating
    }
}

class HtmlWrapDecorator extends TextDecorator {
    private final String tag;

    public HtmlWrapDecorator(TextProcessor wrapped, String tag) {
        super(wrapped);
        this.tag = tag;
    }

    @Override
    public String process(String text) {
        String processed = wrapped.process(text);
        return "<" + tag + ">" + processed + "</" + tag + ">";
    }
}

public class DecoratorDemo {
    public static void main(String[] args) {
        // Start with a plain processor, then layer decorators
        TextProcessor processor = new PlainTextProcessor();
        System.out.println(processor.process("  hello world  "));

        // Layer 1: trim whitespace
        processor = new TrimDecorator(new PlainTextProcessor());
        System.out.println(processor.process("  hello world  "));

        // Layer 2: trim + uppercase
        processor = new UpperCaseDecorator(
                        new TrimDecorator(
                            new PlainTextProcessor()));
        System.out.println(processor.process("  hello world  "));

        // Layer 3: trim + uppercase + wrap in <b>
        processor = new HtmlWrapDecorator(
                        new UpperCaseDecorator(
                            new TrimDecorator(
                                new PlainTextProcessor())), "b");
        System.out.println(processor.process("  hello world  "));
    }
}
```

**Output:**
```
  hello world  
hello world
HELLO WORLD
<b>HELLO WORLD</b>
```

Each decorator is independent — you can combine them in any order.

---

### The I/O Streams — Decorator in the JDK

Java's I/O library is the most famous real-world use of this pattern:

```java
// Layers of decoration, each adding one capability:
// FileInputStream        — reads raw bytes from a file
// BufferedInputStream    — adds buffering (fewer disk reads)
// GZIPInputStream        — adds decompression
// InputStreamReader      — adds character encoding translation
// BufferedReader         — adds readLine() convenience

BufferedReader reader = new BufferedReader(
    new InputStreamReader(
        new BufferedInputStream(
            new FileInputStream("data.gz"))));
// You've now built a line-reading, buffered, decoded file reader
// by composing simple decorators — no subclassing explosion
```

---

### ⚠️ Important Points & Best Practices

- Decorator and Composite both use recursive composition, but for different purposes: Composite builds trees of the same type; Decorator wraps to add behavior.
- Many decorators on an object create a "wrapper stack." Debugging can be harder because you need to trace through multiple layers.
- Order matters: `Trim(UpperCase(text))` may behave differently from `UpperCase(Trim(text))` depending on implementation.
- Java's `Collections.unmodifiableList()`, `Collections.synchronizedList()` are Decorator functions that add unmodifiability or thread-safety to any existing list.

---

## 5. Facade Pattern

### Theory

The Facade pattern provides a **simplified, unified interface to a complex subsystem**. It doesn't add new functionality — it hides complexity behind an easy-to-use interface, making the subsystem easier to use for common tasks.

**Analogy:** A TV remote is a facade. Behind the scenes, a TV has a tuner, amplifier, display controller, and input switcher. The remote provides 4 simple buttons. You don't need to understand the subsystem to change the channel.

**When to use it:**
- You want to provide a simple interface to a complex subsystem
- You want to layer a subsystem — high-level clients use the facade, advanced clients can still access internals directly
- Reducing dependencies of outside code on the inner workings of a subsystem

### Example — Home Theater System

```java
// Complex subsystem classes — each has many methods
class DVDPlayer {
    public void on()                { System.out.println("DVD Player: ON"); }
    public void play(String movie)  { System.out.println("DVD Player: Playing '" + movie + "'"); }
    public void stop()              { System.out.println("DVD Player: Stopped"); }
    public void off()               { System.out.println("DVD Player: OFF"); }
}

class Amplifier {
    public void on()                { System.out.println("Amplifier: ON"); }
    public void setVolume(int vol)  { System.out.println("Amplifier: Volume set to " + vol); }
    public void setSurroundSound()  { System.out.println("Amplifier: Surround Sound enabled"); }
    public void off()               { System.out.println("Amplifier: OFF"); }
}

class Projector {
    public void on()                { System.out.println("Projector: ON"); }
    public void wideScreenMode()    { System.out.println("Projector: Widescreen mode"); }
    public void off()               { System.out.println("Projector: OFF"); }
}

class Lights {
    public void dim(int level)      { System.out.println("Lights: Dimmed to " + level + "%"); }
    public void on()                { System.out.println("Lights: Full brightness"); }
}

// THE FACADE — one simple interface that orchestrates the entire subsystem
class HomeTheaterFacade {
    private final DVDPlayer dvd;
    private final Amplifier amp;
    private final Projector projector;
    private final Lights lights;

    public HomeTheaterFacade(DVDPlayer dvd, Amplifier amp,
                              Projector projector, Lights lights) {
        this.dvd       = dvd;
        this.amp       = amp;
        this.projector = projector;
        this.lights    = lights;
    }

    // ONE method replaces a sequence of 7+ subsystem calls
    public void watchMovie(String movie) {
        System.out.println("--- Getting ready to watch movie ---");
        lights.dim(20);
        projector.on();
        projector.wideScreenMode();
        amp.on();
        amp.setSurroundSound();
        amp.setVolume(70);
        dvd.on();
        dvd.play(movie);
    }

    public void endMovie() {
        System.out.println("\n--- Shutting down home theater ---");
        dvd.stop();
        dvd.off();
        amp.off();
        projector.off();
        lights.on();
    }
}

public class FacadeDemo {
    public static void main(String[] args) {
        // The client only uses the facade — clean and simple
        HomeTheaterFacade theater = new HomeTheaterFacade(
            new DVDPlayer(), new Amplifier(), new Projector(), new Lights()
        );

        theater.watchMovie("Inception");
        // ...movie plays...
        theater.endMovie();
    }
}
```

---

### ⚠️ Important Points & Best Practices

- Facade doesn't prevent advanced users from accessing the subsystem directly. It's an optional convenience layer, not a locked door.
- Spring's `JdbcTemplate` is a Facade over the verbose JDBC API — it hides connection management, statement creation, and resource closing.
- Facade vs Adapter: Adapter makes two incompatible interfaces work together. Facade simplifies one complex interface.
- Don't let your Facade grow into a "God Object" that knows too much. It should just orchestrate and delegate, not hold business logic.

---

## 6. Flyweight Pattern

### Theory

The Flyweight pattern **shares common state among many fine-grained objects** to use memory efficiently. Instead of each object storing all its data independently, shared (intrinsic) state is extracted and stored in shared flyweight objects, while unique (extrinsic) state is passed in from the context.

**Intrinsic state** = shared, immutable, stored in the flyweight (e.g., the letter 'A' in a text editor — its shape data is the same everywhere).
**Extrinsic state** = unique, context-dependent, passed in by caller (e.g., the position of that letter 'A' on screen).

**When to use it:**
- A large number of similar objects is consuming too much memory
- Most object state can be made extrinsic
- Game particles, characters in a text editor, trees in a forest renderer

### Example — Rendering a Forest of Trees

```java
import java.util.HashMap;
import java.util.Map;
import java.util.ArrayList;
import java.util.List;

// FLYWEIGHT — stores intrinsic (shared) state
// One TreeType object is shared by thousands of trees of the same type
class TreeType {
    private final String name;
    private final String color;
    private final String texture; // imagine this is a large texture object

    public TreeType(String name, String color, String texture) {
        this.name    = name;
        this.color   = color;
        this.texture = texture;
        System.out.println("Created TreeType: " + name + " (expensive!)");
    }

    // Extrinsic state (x, y position) is passed in — not stored here
    public void draw(int x, int y) {
        System.out.printf("Drawing %s tree [color=%s] at (%d, %d)%n",
            name, color, x, y);
    }
}

// FLYWEIGHT FACTORY — ensures flyweights are shared, not duplicated
class TreeTypeFactory {
    private static final Map<String, TreeType> cache = new HashMap<>();

    public static TreeType getTreeType(String name, String color, String texture) {
        String key = name + "_" + color;
        // Return existing flyweight if one exists for this combination
        return cache.computeIfAbsent(key, k -> new TreeType(name, color, texture));
    }

    public static int getCacheSize() { return cache.size(); }
}

// CONTEXT — each tree is lightweight; stores only its unique position
// and a reference to the shared flyweight
class Tree {
    private final int x;
    private final int y;
    private final TreeType type; // reference to shared flyweight — not a copy

    public Tree(int x, int y, TreeType type) {
        this.x    = x;
        this.y    = y;
        this.type = type;
    }

    public void draw() {
        type.draw(x, y); // passes its unique x,y as extrinsic state
    }
}

// Forest — would be enormous without flyweight
class Forest {
    private final List<Tree> trees = new ArrayList<>();

    public void plantTree(int x, int y, String name, String color, String texture) {
        TreeType type = TreeTypeFactory.getTreeType(name, color, texture);
        trees.add(new Tree(x, y, type)); // Tree is tiny — only stores x,y + reference
    }

    public void draw() {
        trees.forEach(Tree::draw);
    }
}

public class FlyweightDemo {
    public static void main(String[] args) {
        Forest forest = new Forest();

        // Plant 6 trees — but only 2 unique TreeType objects are created
        forest.plantTree(10, 20, "Oak",  "Green",  "oak_texture.png");
        forest.plantTree(30, 50, "Oak",  "Green",  "oak_texture.png"); // reused!
        forest.plantTree(15, 80, "Pine", "DarkGreen", "pine_texture.png");
        forest.plantTree(60, 10, "Pine", "DarkGreen", "pine_texture.png"); // reused!
        forest.plantTree(40, 40, "Oak",  "Green",  "oak_texture.png"); // reused!
        forest.plantTree(75, 90, "Pine", "DarkGreen", "pine_texture.png"); // reused!

        System.out.println("\n--- Drawing Forest ---");
        forest.draw();

        System.out.println("\nTreeType objects in memory: "
            + TreeTypeFactory.getCacheSize() + " (not 6!)");
    }
}
```

**Output:**
```
Created TreeType: Oak (expensive!)
Created TreeType: Pine (expensive!)

--- Drawing Forest ---
Drawing Oak tree [color=Green] at (10, 20)
Drawing Oak tree [color=Green] at (30, 50)
...

TreeType objects in memory: 2 (not 6!)
```

Six trees, but only two `TreeType` objects in memory — regardless of whether you plant 6 or 6 million trees.

---

### ⚠️ Important Points & Best Practices

- Java's `String` pool is the most famous Flyweight. String literals like `"hello"` are interned — only one `String` object exists per unique value.
- `Integer.valueOf()` caches integers from -128 to 127 — another built-in Flyweight.
- The tradeoff is **time for space**: you save memory but introduce the overhead of looking up flyweights and passing extrinsic state around.
- Flyweights must be **immutable** — since they're shared, any mutation would affect all users.

---

## 7. Proxy Pattern

### Theory

The Proxy pattern provides a **surrogate or placeholder object** that controls access to another object (the *real subject*). The proxy implements the same interface as the real subject, so clients can't tell the difference. The proxy can add logic before/after delegating to the real object.

**Types of proxies:**
- **Virtual Proxy** — delays creation of expensive objects until they're actually needed (lazy initialization)
- **Protection Proxy** — controls access based on permissions
- **Remote Proxy** — represents an object in a different address space (RMI)
- **Caching Proxy** — caches results of expensive operations
- **Logging Proxy** — logs calls to the real object

### Example — Virtual Proxy for Lazy Image Loading

```java
// SUBJECT interface — both real image and proxy implement this
interface Image {
    void display();
    String getFilename();
}

// REAL SUBJECT — expensive to create (simulates disk/network load)
class RealImage implements Image {
    private final String filename;

    public RealImage(String filename) {
        this.filename = filename;
        loadFromDisk(); // expensive operation in constructor
    }

    private void loadFromDisk() {
        System.out.println("Loading heavy image from disk: " + filename);
        // Imagine this takes 2 seconds
    }

    @Override
    public void display() {
        System.out.println("Displaying image: " + filename);
    }

    @Override
    public String getFilename() { return filename; }
}

// PROXY — same interface, defers real creation until display() is called
class LazyImageProxy implements Image {
    private final String filename;
    private RealImage realImage; // starts as null — real object not yet created

    public LazyImageProxy(String filename) {
        this.filename = filename;
        // No disk load here! Fast constructor.
        System.out.println("Proxy created for: " + filename + " (not loaded yet)");
    }

    @Override
    public void display() {
        // Only load the real image when we actually need to display it
        if (realImage == null) {
            realImage = new RealImage(filename); // lazy initialization
        }
        realImage.display();
    }

    @Override
    public String getFilename() { return filename; } // no real image needed for this
}

public class ProxyDemo {
    public static void main(String[] args) {
        System.out.println("=== Creating image objects ===");

        // Only proxy objects created — no disk loading yet
        Image img1 = new LazyImageProxy("photo1.jpg");
        Image img2 = new LazyImageProxy("photo2.jpg");
        Image img3 = new LazyImageProxy("photo3.jpg");

        System.out.println("\nFilenames available without loading:");
        System.out.println(img1.getFilename()); // proxy handles this, no load
        System.out.println(img2.getFilename());

        System.out.println("\n=== Now displaying images ===");
        img1.display(); // FIRST CALL: triggers actual load
        img1.display(); // SECOND CALL: already loaded, no reload
        img2.display(); // Load happens here
        // img3 is never displayed — its real image is never loaded (saves resources!)
    }
}
```

---

### JDK Dynamic Proxy — Reflection-Based Runtime Proxy

Java can create proxies at runtime without writing a proxy class. This is how Spring AOP, Hibernate lazy loading, and many frameworks work under the hood.

```java
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;

interface Service {
    String getData();
    void process();
}

class RealService implements Service {
    @Override public String getData()  { return "Real data"; }
    @Override public void process()    { System.out.println("Processing..."); }
}

// The InvocationHandler — intercepts all method calls on the proxy
class LoggingHandler implements InvocationHandler {
    private final Object realObject;

    public LoggingHandler(Object realObject) {
        this.realObject = realObject;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("[LOG] Calling: " + method.getName());
        long start = System.currentTimeMillis();

        Object result = method.invoke(realObject, args); // delegate to real object

        long elapsed = System.currentTimeMillis() - start;
        System.out.println("[LOG] " + method.getName() + " took " + elapsed + "ms");
        return result;
    }
}

public class DynamicProxyDemo {
    public static void main(String[] args) {
        Service realService = new RealService();

        // Create proxy at runtime — no need to write a LoggingProxy class
        Service proxy = (Service) Proxy.newProxyInstance(
            Service.class.getClassLoader(),
            new Class[]{Service.class},
            new LoggingHandler(realService)
        );

        System.out.println(proxy.getData());
        proxy.process();
    }
}
```

This is exactly how Spring's `@Transactional` and `@Cacheable` work — at startup Spring wraps your beans in dynamic proxies that add transactional or caching behavior before delegating to your real method.

---

### ⚠️ Important Points & Best Practices

- JDK dynamic proxies only work with **interfaces**. For proxying concrete classes, Spring uses **CGLIB** (a bytecode manipulation library) which subclasses the target class instead.
- Proxy vs Decorator: they look similar structurally, but the intent differs. Decorator *adds* behavior transparently; Proxy *controls access* to the real subject and may restrict, cache, or log it.
- Be aware of the "self-invocation" problem in Spring: if a method inside a Spring bean calls another method in the same bean, the proxy is bypassed — so `@Transactional` on the called method won't work.

---

## Summary Table — Structural Patterns

| Pattern | Intent | Key Mechanism | Java / Spring Example |
|---------|--------|---------------|-----------------------|
| Adapter | Make incompatible interfaces work together | Wraps adaptee, implements target interface | `Arrays.asList()`, `InputStreamReader` |
| Bridge | Decouple abstraction from implementation | Composition — abstraction holds implementation reference | JDBC Driver API |
| Composite | Uniform treatment of trees | Leaf and Composite share same interface | Swing UI, File system |
| Decorator | Add behavior dynamically | Wrapper chain — each adds one behavior | Java I/O streams, `Collections.unmodifiableList()` |
| Facade | Simplify complex subsystem | Single orchestrating class | `JdbcTemplate`, Spring's `RestTemplate` |
| Flyweight | Share common state to save memory | Factory returns shared, immutable objects | String pool, `Integer.valueOf()` cache |
| Proxy | Control access to an object | Same-interface wrapper with added logic | Spring AOP, Hibernate lazy loading |

---

*Next: Phase 8.3 — Behavioral Patterns →*

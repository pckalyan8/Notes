# Phase 8.3 — Behavioral Design Patterns in Java

> **What are Behavioral Patterns?**
> Behavioral patterns are concerned with **algorithms and the assignment of responsibilities between objects**. They describe not just patterns of objects or classes, but also the patterns of communication between them. While creational patterns answer "who creates objects?" and structural patterns answer "how do objects fit together?", behavioral patterns answer: *"How do objects collaborate and divide up work at runtime?"*

---

## 1. Chain of Responsibility Pattern

### Theory

The Chain of Responsibility pattern lets you **pass a request along a chain of handlers**. Each handler either processes the request or passes it to the next handler in the chain. The sender doesn't know which handler will ultimately process the request.

Think of it like a company's support ticket system: Level-1 support handles simple issues, escalates complex ones to Level-2, and Level-2 escalates to the engineering team. The ticket "travels" until someone handles it.

**When to use it:**
- More than one object may handle a request, and the handler isn't known a priori
- You want to issue a request without specifying the receiver explicitly
- The set of objects that can handle a request should be specified dynamically
- Logging systems, event handling, middleware pipelines, HTTP filter chains

### Example — Technical Support Escalation

```java
// Abstract handler — defines the chain structure
abstract class SupportHandler {
    // The next handler in the chain
    protected SupportHandler nextHandler;

    public SupportHandler setNext(SupportHandler handler) {
        this.nextHandler = handler;
        return handler; // enables fluent chaining: level1.setNext(level2).setNext(level3)
    }

    // Each subclass must decide: handle it OR pass it forward
    public abstract void handle(SupportTicket ticket);

    // Helper to forward to next handler (or report "unhandled")
    protected void passToNext(SupportTicket ticket) {
        if (nextHandler != null) {
            nextHandler.handle(ticket);
        } else {
            System.out.println("Ticket #" + ticket.getId()
                + " [" + ticket.getSeverity() + "] — No handler found! Escalating to management.");
        }
    }
}

// Request object — passed along the chain
class SupportTicket {
    private final int id;
    private final String description;
    private final int severity; // 1=low, 2=medium, 3=high, 4=critical

    public SupportTicket(int id, String description, int severity) {
        this.id          = id;
        this.description = description;
        this.severity    = severity;
    }

    public int getId()           { return id; }
    public String getSeverity()  { return switch(severity){ case 1->"LOW"; case 2->"MEDIUM"; case 3->"HIGH"; default->"CRITICAL"; }; }
    public int getSeverityLevel(){ return severity; }
    public String getDescription(){ return description; }
}

// Concrete handlers — each handles requests up to a certain severity
class Level1Support extends SupportHandler {
    @Override
    public void handle(SupportTicket ticket) {
        if (ticket.getSeverityLevel() == 1) {
            System.out.println("Level-1 resolved ticket #" + ticket.getId()
                + ": " + ticket.getDescription());
        } else {
            System.out.println("Level-1 escalating ticket #" + ticket.getId()
                + " (severity too high)");
            passToNext(ticket);
        }
    }
}

class Level2Support extends SupportHandler {
    @Override
    public void handle(SupportTicket ticket) {
        if (ticket.getSeverityLevel() <= 2) {
            System.out.println("Level-2 resolved ticket #" + ticket.getId()
                + ": " + ticket.getDescription());
        } else {
            System.out.println("Level-2 escalating ticket #" + ticket.getId()
                + " to engineering");
            passToNext(ticket);
        }
    }
}

class EngineeringTeam extends SupportHandler {
    @Override
    public void handle(SupportTicket ticket) {
        if (ticket.getSeverityLevel() <= 3) {
            System.out.println("Engineering resolved ticket #" + ticket.getId()
                + ": " + ticket.getDescription());
        } else {
            System.out.println("Engineering escalating critical ticket #"
                + ticket.getId() + " to CTO");
            passToNext(ticket); // chain continues or ends here
        }
    }
}

public class ChainOfResponsibilityDemo {
    public static void main(String[] args) {
        // Build the chain
        SupportHandler level1 = new Level1Support();
        SupportHandler level2 = new Level2Support();
        SupportHandler engineering = new EngineeringTeam();

        // Fluently link the chain
        level1.setNext(level2).setNext(engineering);

        // Send tickets — each travels the chain until handled
        level1.handle(new SupportTicket(101, "Reset password",        1));
        level1.handle(new SupportTicket(102, "App crashes on login",  2));
        level1.handle(new SupportTicket(103, "Data corruption bug",   3));
        level1.handle(new SupportTicket(104, "Production DB is down", 4));
    }
}
```

---

### ⚠️ Important Points & Best Practices

- Java Servlet Filters and Spring's `HandlerInterceptor` are real-world implementations of this pattern — each filter in the chain can process or skip a request.
- A handler isn't required to pass the request — it can absorb it completely (fire-and-forget).
- Chains can be assembled at runtime from config rather than being hardcoded — very powerful for middleware pipelines.
- Be careful of chains that are too long, which can hurt performance and make debugging hard.

---

## 2. Command Pattern

### Theory

The Command pattern **encapsulates a request as an object**, thereby letting you parameterize clients with different requests, queue or log requests, and support undoable operations. The request (what to do) is decoupled from the object that performs the action (the receiver), and the object that triggers the request (the invoker).

The four key roles are: **Command** (the encapsulated request), **Receiver** (the object that does the actual work), **Invoker** (triggers the command), and **Client** (creates and configures commands).

**When to use it:**
- Implementing undo/redo operations
- Queuing operations for later execution
- Remote execution, logging of operations
- GUI actions (every button click is a command in Swing/JavaFX)

### Example — Text Editor with Undo

```java
import java.util.ArrayDeque;
import java.util.Deque;

// COMMAND interface — declares execute() and undo()
interface Command {
    void execute();
    void undo();
}

// RECEIVER — the object that does the real work
class TextDocument {
    private final StringBuilder content = new StringBuilder();

    public void write(String text) {
        content.append(text);
    }

    public void delete(int length) {
        if (length > 0 && length <= content.length()) {
            content.delete(content.length() - length, content.length());
        }
    }

    public String getContent() { return content.toString(); }
}

// CONCRETE COMMANDS — each encapsulates one operation + how to undo it
class WriteCommand implements Command {
    private final TextDocument document; // reference to the receiver
    private final String text;

    public WriteCommand(TextDocument document, String text) {
        this.document = document;
        this.text     = text;
    }

    @Override
    public void execute() {
        document.write(text);    // tell the receiver to do the work
    }

    @Override
    public void undo() {
        document.delete(text.length()); // reverse the operation
    }
}

// INVOKER — maintains command history, triggers execute/undo
class TextEditor {
    private final Deque<Command> history = new ArrayDeque<>();

    public void executeCommand(Command command) {
        command.execute();
        history.push(command); // push to stack for undo
    }

    public void undo() {
        if (!history.isEmpty()) {
            Command last = history.pop();
            last.undo();
            System.out.println("Undone!");
        } else {
            System.out.println("Nothing to undo.");
        }
    }
}

public class CommandDemo {
    public static void main(String[] args) {
        TextDocument doc    = new TextDocument();
        TextEditor   editor = new TextEditor();

        editor.executeCommand(new WriteCommand(doc, "Hello"));
        System.out.println("After 'Hello': " + doc.getContent());

        editor.executeCommand(new WriteCommand(doc, ", World"));
        System.out.println("After ', World': " + doc.getContent());

        editor.executeCommand(new WriteCommand(doc, "!"));
        System.out.println("After '!': " + doc.getContent());

        editor.undo();
        System.out.println("After undo: " + doc.getContent());

        editor.undo();
        System.out.println("After undo: " + doc.getContent());
    }
}
```

---

### ⚠️ Important Points & Best Practices

- Each `Command` object carries everything needed to perform and reverse the operation — this is key to supporting undo/redo.
- In Java 8+, simple commands (without undo) can be expressed as lambdas: `Runnable command = () -> document.write("Hello");` — no need for a whole class.
- `java.lang.Runnable` and `java.util.concurrent.Callable` are the Command pattern baked into the JDK.
- For transactional systems, commands can be serialized to a log — if the system crashes, you can replay the log to recover.

---

## 3. Iterator Pattern

### Theory

The Iterator pattern provides a way to **sequentially access elements of a collection without exposing its underlying representation**. You can traverse a list, tree, graph, or any data structure in a unified way regardless of how it's internally stored.

You already use this pattern constantly in Java — the enhanced for-each loop and `java.util.Iterator` are this pattern built directly into the language.

**When to use it:**
- Traversing a collection without knowing how it's stored internally
- Providing multiple types of traversal for the same collection (forward, backward, filtered)
- Giving a uniform traversal interface over different types of collections

### Example — Custom Range Iterator

```java
import java.util.Iterator;
import java.util.NoSuchElementException;

// Custom iterable that generates a range of numbers lazily (no array needed)
class NumberRange implements Iterable<Integer> {
    private final int start;
    private final int end;
    private final int step;

    public NumberRange(int start, int end, int step) {
        if (step <= 0) throw new IllegalArgumentException("Step must be positive");
        this.start = start;
        this.end   = end;
        this.step  = step;
    }

    // Iterable contract — return an Iterator
    @Override
    public Iterator<Integer> iterator() {
        return new RangeIterator();
    }

    // Inner iterator class — holds the traversal state
    private class RangeIterator implements Iterator<Integer> {
        private int current = start;

        @Override
        public boolean hasNext() {
            return current <= end;
        }

        @Override
        public Integer next() {
            if (!hasNext()) {
                throw new NoSuchElementException();
            }
            int value = current;
            current += step; // advance by step
            return value;
        }
    }
}

public class IteratorDemo {
    public static void main(String[] args) {
        NumberRange range = new NumberRange(1, 10, 2); // 1, 3, 5, 7, 9

        // Works with enhanced for-each because we implement Iterable
        System.out.print("Odd numbers: ");
        for (int n : range) {
            System.out.print(n + " ");
        }

        System.out.println();

        // Explicit iterator use for fine-grained control
        Iterator<Integer> it = new NumberRange(0, 100, 10).iterator();
        System.out.print("Multiples of 10: ");
        while (it.hasNext()) {
            System.out.print(it.next() + " ");
        }
    }
}
```

---

### ⚠️ Important Points & Best Practices

- The Iterator pattern is so fundamental in Java that it's the basis of `Collections`, `Stream`, and the for-each loop. Implementing `Iterable<T>` on your custom collection makes it work with all these tools.
- Fail-fast iterators (like those in `ArrayList`, `HashMap`) throw `ConcurrentModificationException` if the collection is modified during iteration — use an explicit `Iterator` and call `iterator.remove()` to safely remove during traversal.
- For traversal without modifying, `Stream` is now more expressive than raw iterators.

---

## 4. Observer Pattern

### Theory

The Observer pattern defines a **one-to-many dependency** between objects so that when one object (the *subject/publisher*) changes state, all its dependents (*observers/subscribers*) are notified and updated automatically.

This is also known as the publish-subscribe pattern. The subject and observers are loosely coupled — the subject just notifies all registered observers; it doesn't know what they do with the notification.

**When to use it:**
- When a change in one object requires changing others, and you don't know how many others
- Event systems, GUI event listeners, messaging systems
- MVC architecture — model is the subject, views are observers

### Example — Stock Price Monitor

```java
import java.util.ArrayList;
import java.util.List;

// OBSERVER interface — all observers implement this
interface StockObserver {
    void onPriceChanged(String stockSymbol, double oldPrice, double newPrice);
}

// SUBJECT — maintains list of observers and notifies them on state change
class StockMarket {
    private final List<StockObserver> observers = new ArrayList<>();
    private final String symbol;
    private double currentPrice;

    public StockMarket(String symbol, double initialPrice) {
        this.symbol       = symbol;
        this.currentPrice = initialPrice;
    }

    public void subscribe(StockObserver observer) {
        observers.add(observer);
    }

    public void unsubscribe(StockObserver observer) {
        observers.remove(observer);
    }

    // When price changes, notify ALL registered observers
    public void updatePrice(double newPrice) {
        double oldPrice   = this.currentPrice;
        this.currentPrice = newPrice;
        System.out.printf("%nStock %s price changed: $%.2f → $%.2f%n",
            symbol, oldPrice, newPrice);
        notifyObservers(oldPrice, newPrice);
    }

    private void notifyObservers(double oldPrice, double newPrice) {
        for (StockObserver observer : observers) {
            observer.onPriceChanged(symbol, oldPrice, newPrice);
        }
    }
}

// CONCRETE OBSERVERS — each reacts differently to the same event
class AlertObserver implements StockObserver {
    private final double threshold;

    public AlertObserver(double threshold) {
        this.threshold = threshold;
    }

    @Override
    public void onPriceChanged(String symbol, double oldPrice, double newPrice) {
        double changePercent = Math.abs((newPrice - oldPrice) / oldPrice * 100);
        if (changePercent >= threshold) {
            System.out.printf("🔔 ALERT: %s moved %.1f%% — significant change!%n",
                symbol, changePercent);
        }
    }
}

class PortfolioObserver implements StockObserver {
    private final String investorName;
    private final int sharesHeld;

    public PortfolioObserver(String investorName, int sharesHeld) {
        this.investorName = investorName;
        this.sharesHeld   = sharesHeld;
    }

    @Override
    public void onPriceChanged(String symbol, double oldPrice, double newPrice) {
        double impact = (newPrice - oldPrice) * sharesHeld;
        String direction = impact >= 0 ? "gained" : "lost";
        System.out.printf("📊 %s's portfolio %s $%.2f on %s%n",
            investorName, direction, Math.abs(impact), symbol);
    }
}

public class ObserverDemo {
    public static void main(String[] args) {
        StockMarket apple = new StockMarket("AAPL", 180.00);

        // Register observers — they subscribe independently
        apple.subscribe(new AlertObserver(5.0)); // alert if >5% change
        apple.subscribe(new PortfolioObserver("Alice", 100));
        apple.subscribe(new PortfolioObserver("Bob", 50));

        // State changes — all observers notified automatically
        apple.updatePrice(190.00); // +5.56% — triggers alert
        apple.updatePrice(188.00); // small dip — no alert
        apple.updatePrice(170.00); // big drop — triggers alert
    }
}
```

---

### ⚠️ Important Points & Best Practices

- `java.util.Observable` (deprecated in Java 9) and `java.util.EventListener` are Java's built-in Observer infrastructure. For modern code, use your own interfaces or `java.beans.PropertyChangeSupport`.
- In a multithreaded environment, be careful about observers being called on the subject's thread — consider dispatching notifications asynchronously.
- Avoid circular dependencies where observers trigger subject state changes that re-notify observers.
- Spring's `ApplicationEvent` and `@EventListener` are the enterprise version of this pattern.

---

## 5. Strategy Pattern

### Theory

The Strategy pattern **defines a family of algorithms, encapsulates each one, and makes them interchangeable**. It lets the algorithm vary independently from clients that use it. Instead of if-else or switch statements to choose an algorithm, you inject the desired strategy.

**When to use it:**
- You have multiple variants of an algorithm and want to switch between them at runtime
- You want to eliminate conditional statements for algorithm selection
- Payment processing (credit card, PayPal, crypto), sorting strategies, compression algorithms, discount strategies

### Example — Pricing Strategy for an E-Commerce Cart

```java
// STRATEGY interface — defines the contract for all pricing strategies
interface PricingStrategy {
    double calculatePrice(double basePrice, int quantity);
    String getDescription();
}

// CONCRETE STRATEGIES — each encapsulates one pricing algorithm
class RegularPricing implements PricingStrategy {
    @Override
    public double calculatePrice(double basePrice, int quantity) {
        return basePrice * quantity; // no discount
    }

    @Override
    public String getDescription() { return "Regular pricing (no discount)"; }
}

class BulkPricing implements PricingStrategy {
    private final int bulkThreshold;
    private final double discountPercent;

    public BulkPricing(int bulkThreshold, double discountPercent) {
        this.bulkThreshold   = bulkThreshold;
        this.discountPercent = discountPercent;
    }

    @Override
    public double calculatePrice(double basePrice, int quantity) {
        double total = basePrice * quantity;
        if (quantity >= bulkThreshold) {
            total *= (1 - discountPercent / 100);
        }
        return total;
    }

    @Override
    public String getDescription() {
        return String.format("Bulk pricing (%.0f%% off for %d+ units)",
            discountPercent, bulkThreshold);
    }
}

class SeasonalPricing implements PricingStrategy {
    private final double surchargePercent;

    public SeasonalPricing(double surchargePercent) {
        this.surchargePercent = surchargePercent;
    }

    @Override
    public double calculatePrice(double basePrice, int quantity) {
        return basePrice * quantity * (1 + surchargePercent / 100);
    }

    @Override
    public String getDescription() {
        return String.format("Seasonal pricing (+%.0f%% holiday surcharge)",
            surchargePercent);
    }
}

// CONTEXT — uses a strategy, can swap it at runtime
class ShoppingCart {
    private PricingStrategy pricingStrategy;

    // Strategy is injected — not hardcoded
    public ShoppingCart(PricingStrategy strategy) {
        this.pricingStrategy = strategy;
    }

    // Strategy can be changed at any time
    public void setPricingStrategy(PricingStrategy strategy) {
        this.pricingStrategy = strategy;
    }

    public double checkout(double itemPrice, int quantity) {
        double total = pricingStrategy.calculatePrice(itemPrice, quantity);
        System.out.printf("[%s] %d × $%.2f = $%.2f%n",
            pricingStrategy.getDescription(), quantity, itemPrice, total);
        return total;
    }
}

public class StrategyDemo {
    public static void main(String[] args) {
        ShoppingCart cart = new ShoppingCart(new RegularPricing());
        cart.checkout(25.00, 3);

        // Swap strategy at runtime — same cart, different algorithm
        cart.setPricingStrategy(new BulkPricing(10, 20));
        cart.checkout(25.00, 15); // 15 units qualifies for bulk discount

        cart.setPricingStrategy(new SeasonalPricing(15));
        cart.checkout(25.00, 5);

        // In Java 8+, simple strategies can be lambdas
        PricingStrategy flatRate = (price, qty) -> 9.99 * qty; // $9.99 flat
        cart.setPricingStrategy(flatRate);
        cart.checkout(25.00, 3);
    }
}
```

---

### ⚠️ Important Points & Best Practices

- In Java 8+, if your strategy interface has only one method (it's a `@FunctionalInterface`), you can use lambdas instead of creating a full class per strategy — much less boilerplate.
- `java.util.Comparator` is the most prominent example of Strategy in the JDK — you pass different comparison algorithms to `Collections.sort()`.
- Strategy is often confused with State. The key difference: Strategy objects are usually stateless and interchangeable; State objects carry internal state and transition between themselves.

---

## 6. State Pattern

### Theory

The State pattern allows an object to **alter its behavior when its internal state changes**. The object will appear to change its class. Instead of a large conditional block (`if state == X ... else if state == Y ...`) in every method, each state is a separate class with its own behavior.

**When to use it:**
- An object's behavior depends on its state, and it must change behavior at runtime
- Operations have large, multipart conditional statements that depend on the object's state
- Vending machines, traffic lights, order processing pipelines, game character states

### Example — Vending Machine

```java
// STATE interface — all states implement the same actions
interface VendingMachineState {
    void insertCoin(VendingMachine machine);
    void selectProduct(VendingMachine machine);
    void dispenseProduct(VendingMachine machine);
}

// CONTEXT — the vending machine, delegates behavior to its current state
class VendingMachine {
    // State objects — created once and reused
    private final VendingMachineState idleState;
    private final VendingMachineState hasMoneyState;
    private final VendingMachineState outOfStockState;

    private VendingMachineState currentState;
    private int stockCount;

    public VendingMachine(int initialStock) {
        idleState      = new IdleState();
        hasMoneyState  = new HasMoneyState();
        outOfStockState = new OutOfStockState();

        this.stockCount = initialStock;
        this.currentState = (initialStock > 0) ? idleState : outOfStockState;
    }

    // Delegates all actions to the current state
    public void insertCoin()       { currentState.insertCoin(this); }
    public void selectProduct()    { currentState.selectProduct(this); }
    public void dispenseProduct()  { currentState.dispenseProduct(this); }

    public void setState(VendingMachineState state) { this.currentState = state; }
    public VendingMachineState getIdleState()        { return idleState; }
    public VendingMachineState getHasMoneyState()    { return hasMoneyState; }
    public VendingMachineState getOutOfStockState()  { return outOfStockState; }

    public int getStockCount() { return stockCount; }
    public void decrementStock() { stockCount--; }
}

// CONCRETE STATES — each encapsulates behavior for one state
class IdleState implements VendingMachineState {
    @Override
    public void insertCoin(VendingMachine machine) {
        System.out.println("Coin inserted. Select your product.");
        machine.setState(machine.getHasMoneyState()); // transition!
    }

    @Override
    public void selectProduct(VendingMachine machine) {
        System.out.println("Please insert a coin first.");
    }

    @Override
    public void dispenseProduct(VendingMachine machine) {
        System.out.println("Please insert a coin and select a product first.");
    }
}

class HasMoneyState implements VendingMachineState {
    private boolean productSelected = false;

    @Override
    public void insertCoin(VendingMachine machine) {
        System.out.println("Coin already inserted. Please select your product.");
    }

    @Override
    public void selectProduct(VendingMachine machine) {
        System.out.println("Product selected! Dispensing...");
        productSelected = true;
        machine.dispenseProduct(); // trigger next step in the same state
    }

    @Override
    public void dispenseProduct(VendingMachine machine) {
        if (productSelected) {
            machine.decrementStock();
            productSelected = false;
            System.out.println("Product dispensed! Thank you.");
            if (machine.getStockCount() == 0) {
                System.out.println("Machine is now out of stock.");
                machine.setState(machine.getOutOfStockState());
            } else {
                machine.setState(machine.getIdleState());
            }
        }
    }
}

class OutOfStockState implements VendingMachineState {
    @Override
    public void insertCoin(VendingMachine machine) {
        System.out.println("Sorry, machine is out of stock. Coin returned.");
    }

    @Override
    public void selectProduct(VendingMachine machine) {
        System.out.println("Sorry, out of stock.");
    }

    @Override
    public void dispenseProduct(VendingMachine machine) {
        System.out.println("Cannot dispense — out of stock.");
    }
}

public class StateDemo {
    public static void main(String[] args) {
        VendingMachine machine = new VendingMachine(2);

        // Normal flow — twice
        machine.insertCoin();
        machine.selectProduct();

        machine.insertCoin();
        machine.selectProduct();     // stock is now 0

        // Attempt to use when out of stock
        machine.insertCoin();        // coin returned
    }
}
```

---

### ⚠️ Important Points & Best Practices

- State eliminates large `if-else if` chains by distributing behavior into state classes — each state class is small, focused, and easy to test.
- The difference from Strategy: in Strategy, the client typically sets the strategy once; in State, the object automatically transitions between states based on its own logic.
- Spring's `StateMachine` (Spring State Machine library) is a full enterprise implementation of this pattern for complex workflow engines.

---

## 7. Template Method Pattern

### Theory

The Template Method pattern **defines the skeleton of an algorithm in a base class**, deferring some steps to subclasses. The template method calls abstract methods for the variable steps, and concrete methods for the invariant steps. Subclasses can override specific steps without changing the algorithm's structure.

**When to use it:**
- Multiple classes share the same algorithm structure but differ in specific steps
- You want to control which parts of an algorithm subclasses can override
- Eliminating duplicate code by moving common logic to the base class
- Data miners, report generators, game loops

### Example — Report Generation Pipeline

```java
// ABSTRACT CLASS — defines the template method (the algorithm skeleton)
abstract class ReportGenerator {

    // THE TEMPLATE METHOD — final, so no subclass can change the overall flow
    public final void generateReport() {
        fetchData();         // Step 1 — abstract, subclass decides how
        processData();       // Step 2 — abstract, subclass decides how
        formatReport();      // Step 3 — abstract, subclass decides how
        addHeader();         // Step 4 — concrete, same for everyone
        exportReport();      // Step 5 — hook method (optional override)
        System.out.println("--- Report generation complete ---\n");
    }

    // ABSTRACT STEPS — subclasses MUST implement these
    protected abstract void fetchData();
    protected abstract void processData();
    protected abstract void formatReport();

    // CONCRETE STEP — common for all reports, not overridable here
    private void addHeader() {
        System.out.println("[Header] Generated by ReportSystem v2.0 at " +
            java.time.LocalDate.now());
    }

    // HOOK METHOD — optional override point (has a default, benign implementation)
    protected void exportReport() {
        System.out.println("[Export] Report ready for viewing (no file export by default)");
    }
}

// CONCRETE SUBCLASS 1 — fills in the steps for CSV reports
class CsvReportGenerator extends ReportGenerator {

    @Override
    protected void fetchData() {
        System.out.println("[CSV] Fetching data from database...");
    }

    @Override
    protected void processData() {
        System.out.println("[CSV] Aggregating and sorting data...");
    }

    @Override
    protected void formatReport() {
        System.out.println("[CSV] Formatting data as CSV rows...");
    }

    @Override
    protected void exportReport() {
        System.out.println("[CSV] Exporting to report.csv");
    }
}

// CONCRETE SUBCLASS 2 — fills in the steps for HTML reports
class HtmlReportGenerator extends ReportGenerator {

    @Override
    protected void fetchData() {
        System.out.println("[HTML] Fetching data from REST API...");
    }

    @Override
    protected void processData() {
        System.out.println("[HTML] Filtering and grouping data...");
    }

    @Override
    protected void formatReport() {
        System.out.println("[HTML] Building HTML table with Bootstrap...");
    }
    // exportReport() not overridden — uses the default hook
}

public class TemplateMethodDemo {
    public static void main(String[] args) {
        ReportGenerator csvReport = new CsvReportGenerator();
        csvReport.generateReport();

        ReportGenerator htmlReport = new HtmlReportGenerator();
        htmlReport.generateReport();
    }
}
```

---

### ⚠️ Important Points & Best Practices

- The template method should be `final` in the base class — subclasses define *what* each step does, but they should not be able to reorder or remove steps.
- "Hook methods" are optional override points with default (often empty or no-op) implementations — they give subclasses fine-grained customization without forcing them to implement everything.
- Spring's `JdbcTemplate` uses this pattern heavily — it handles connection acquisition, exception handling, and resource cleanup (the invariant steps), and lets you provide the SQL and row mapping (the variable steps).
- The Inversion of Control principle applies here: the base class "calls" the subclass (a Hollywood Principle: "don't call us, we'll call you").

---

## 8. Mediator Pattern

### Theory

The Mediator pattern defines an object that **encapsulates how a set of objects interact**. It promotes loose coupling by keeping objects from referring to each other explicitly — instead they communicate through the mediator. It centralizes complex many-to-many communication into a single place.

**Without Mediator:** N components each know about all M others — O(N×M) connections, very tangled.
**With Mediator:** Each component only knows the mediator — O(N+M) connections, much cleaner.

**When to use it:**
- Many-to-many communications between objects are making the system hard to maintain
- Chat rooms (users communicate through the room), air traffic control, UI form components coordinating each other

### Example — Chat Room Mediator

```java
import java.util.ArrayList;
import java.util.List;

// MEDIATOR interface
interface ChatMediator {
    void sendMessage(String message, ChatUser sender);
    void addUser(ChatUser user);
}

// CONCRETE MEDIATOR — central communication hub
class ChatRoom implements ChatMediator {
    private final String roomName;
    private final List<ChatUser> users = new ArrayList<>();

    public ChatRoom(String roomName) {
        this.roomName = roomName;
    }

    @Override
    public void addUser(ChatUser user) {
        users.add(user);
        System.out.println(user.getName() + " joined " + roomName);
    }

    // Mediator routes messages — users never talk directly to each other
    @Override
    public void sendMessage(String message, ChatUser sender) {
        System.out.printf("[%s] %s: %s%n", roomName, sender.getName(), message);
        for (ChatUser user : users) {
            if (user != sender) {  // don't send back to sender
                user.receive(message, sender.getName());
            }
        }
    }
}

// COLLEAGUE — knows only about the mediator, not other users
class ChatUser {
    private final String name;
    private final ChatMediator mediator;

    public ChatUser(String name, ChatMediator mediator) {
        this.name     = name;
        this.mediator = mediator;
        mediator.addUser(this); // register with mediator
    }

    public String getName() { return name; }

    // Sends through the mediator — no direct user-to-user reference
    public void send(String message) {
        mediator.sendMessage(message, this);
    }

    public void receive(String message, String fromUser) {
        System.out.printf("  [%s received from %s]: %s%n", name, fromUser, message);
    }
}

public class MediatorDemo {
    public static void main(String[] args) {
        ChatMediator room = new ChatRoom("JavaDevs");

        ChatUser alice = new ChatUser("Alice", room);
        ChatUser bob   = new ChatUser("Bob", room);
        ChatUser carol = new ChatUser("Carol", room);

        System.out.println();
        alice.send("Has anyone used virtual threads yet?");
        System.out.println();
        bob.send("Yes! They're game-changing for IO-bound apps.");
    }
}
```

---

### ⚠️ Important Points & Best Practices

- The Mediator can become a "god object" if it accumulates too much logic. Keep it focused on routing/coordinating — not on business logic.
- Spring's `ApplicationEventPublisher` is a mediator — beans publish events and other beans receive them without knowing who published.

---

## 9. Memento Pattern

### Theory

The Memento pattern **captures and externalizes an object's internal state** so that the object can be restored to that state later — all without violating encapsulation. It's the foundation for undo/redo functionality.

Three roles: **Originator** (the object whose state you save), **Memento** (a snapshot of the state), **Caretaker** (stores mementos and requests restore).

### Example — Game Save System

```java
import java.util.ArrayDeque;
import java.util.Deque;

// MEMENTO — stores a snapshot of the game state
// It's an immutable value object — just data, no behavior
record GameMemento(int level, int score, String location, int health) {}

// ORIGINATOR — game character whose state we snapshot
class GameCharacter {
    private int level;
    private int score;
    private String location;
    private int health;

    public GameCharacter(int level, int score, String location, int health) {
        this.level    = level;
        this.score    = score;
        this.location = location;
        this.health   = health;
    }

    // Create a snapshot of current state
    public GameMemento save() {
        System.out.println("Game saved at: " + this);
        return new GameMemento(level, score, location, health);
    }

    // Restore from a snapshot
    public void restore(GameMemento memento) {
        this.level    = memento.level();
        this.score    = memento.score();
        this.location = memento.location();
        this.health   = memento.health();
        System.out.println("Game restored to: " + this);
    }

    public void progressLevel() {
        level++;
        score += 500;
        location = "World " + level + "-1";
        System.out.println("Level up! " + this);
    }

    public void takeDamage(int damage) {
        health = Math.max(0, health - damage);
        System.out.println("Took " + damage + " damage. " + this);
    }

    @Override
    public String toString() {
        return String.format("Character[level=%d, score=%d, loc=%s, hp=%d]",
            level, score, location, health);
    }
}

// CARETAKER — manages the save game history
class SaveGameManager {
    private final Deque<GameMemento> saveSlots = new ArrayDeque<>();

    public void save(GameCharacter character) {
        saveSlots.push(character.save());
    }

    public void load(GameCharacter character) {
        if (!saveSlots.isEmpty()) {
            character.restore(saveSlots.pop());
        } else {
            System.out.println("No save file found!");
        }
    }
}

public class MementoDemo {
    public static void main(String[] args) {
        GameCharacter player = new GameCharacter(1, 0, "World 1-1", 100);
        SaveGameManager saves = new SaveGameManager();

        saves.save(player);          // save before advancing
        player.progressLevel();
        player.progressLevel();
        saves.save(player);          // save at level 3

        player.takeDamage(80);       // oops, nearly dead

        System.out.println("\nLoading most recent save...");
        saves.load(player);          // back to level 3, full health

        System.out.println("\nLoading earlier save...");
        saves.load(player);          // back to level 1
    }
}
```

---

## 10. Visitor Pattern

### Theory

The Visitor pattern lets you **add further operations to objects without modifying them**. You define a `Visitor` interface with a `visit()` method for each type of element. Elements accept visitors by calling `visitor.visit(this)`, letting the visitor perform the appropriate operation.

It effectively separates algorithms from the objects on which they operate. The key mechanism is **double dispatch** — the call resolves at runtime based on both the visitor type and the element type.

**When to use it:**
- You need to perform many distinct and unrelated operations on an object structure without polluting the object classes
- The object structure classes rarely change, but you often need to add new operations
- Compilers (AST operations), tax calculators on different item types, document exporters

### Example — Tax Calculator for Different Order Items

```java
// ELEMENT interface — accepts a visitor
interface OrderItem {
    double getBasePrice();
    void accept(TaxVisitor visitor); // the visitor hook
}

// CONCRETE ELEMENTS — different item types with different tax rules
class Food implements OrderItem {
    private final String name;
    private final double price;

    public Food(String name, double price) {
        this.name  = name;
        this.price = price;
    }

    @Override public double getBasePrice() { return price; }
    @Override public String toString()     { return name; }

    @Override
    public void accept(TaxVisitor visitor) {
        visitor.visitFood(this); // calls the right method on visitor
    }
}

class Electronics implements OrderItem {
    private final String name;
    private final double price;

    public Electronics(String name, double price) {
        this.name  = name;
        this.price = price;
    }

    @Override public double getBasePrice() { return price; }
    @Override public String toString()     { return name; }

    @Override
    public void accept(TaxVisitor visitor) {
        visitor.visitElectronics(this);
    }
}

class Luxury implements OrderItem {
    private final String name;
    private final double price;

    public Luxury(String name, double price) {
        this.name  = name;
        this.price = price;
    }

    @Override public double getBasePrice() { return price; }
    @Override public String toString()     { return name; }

    @Override
    public void accept(TaxVisitor visitor) {
        visitor.visitLuxury(this);
    }
}

// VISITOR interface — one method per element type
interface TaxVisitor {
    void visitFood(Food food);
    void visitElectronics(Electronics electronics);
    void visitLuxury(Luxury luxury);
}

// CONCRETE VISITOR 1 — standard tax rates
class StandardTaxVisitor implements TaxVisitor {
    private double totalTax = 0;

    @Override
    public void visitFood(Food food) {
        double tax = food.getBasePrice() * 0.05; // 5% food tax
        System.out.printf("  Food '%s': $%.2f + tax $%.2f%n",
            food, food.getBasePrice(), tax);
        totalTax += tax;
    }

    @Override
    public void visitElectronics(Electronics e) {
        double tax = e.getBasePrice() * 0.15; // 15% electronics tax
        System.out.printf("  Electronics '%s': $%.2f + tax $%.2f%n",
            e, e.getBasePrice(), tax);
        totalTax += tax;
    }

    @Override
    public void visitLuxury(Luxury luxury) {
        double tax = luxury.getBasePrice() * 0.25; // 25% luxury tax
        System.out.printf("  Luxury '%s': $%.2f + tax $%.2f%n",
            luxury, luxury.getBasePrice(), tax);
        totalTax += tax;
    }

    public double getTotalTax() { return totalTax; }
}

public class VisitorDemo {
    public static void main(String[] args) {
        List<OrderItem> cart = List.of(
            new Food("Organic Apples", 5.00),
            new Electronics("Headphones", 120.00),
            new Luxury("Designer Watch", 800.00),
            new Food("Coffee Beans", 15.00)
        );

        StandardTaxVisitor taxCalc = new StandardTaxVisitor();

        System.out.println("Calculating tax:");
        for (OrderItem item : cart) {
            item.accept(taxCalc); // visitor determines correct method via double dispatch
        }

        System.out.printf("Total tax: $%.2f%n", taxCalc.getTotalTax());
    }
}
```

---

### ⚠️ Important Points & Best Practices

- The main limitation of Visitor is that adding a new element type requires changing the `Visitor` interface and all its concrete implementations — the opposite tradeoff from adding operations.
- Java 21's pattern matching in `switch` can replace some Visitor use cases more cleanly without the explicit `accept()` ceremony.
- Visitor gives you a powerful separation: the element classes stay simple (they just accept visitors), and all the complex operation logic lives in the visitor classes.

---

## 11. Interpreter Pattern

### Theory

The Interpreter pattern defines a **grammar for a language and provides an interpreter** to deal with that grammar. It's most useful for small, simple languages where you need to evaluate expressions repeatedly.

**When to use it:**
- Simple language or expression parsing (math expressions, boolean filters, SQL-like query builders)
- Interpreting regular expressions, scripting languages

### Example — Simple Boolean Expression Evaluator

```java
import java.util.Map;

// ABSTRACT EXPRESSION — all expressions implement this
interface BooleanExpression {
    boolean interpret(Map<String, Boolean> context);
}

// TERMINAL EXPRESSIONS — leaf nodes, no sub-expressions
class VariableExpression implements BooleanExpression {
    private final String variableName;

    public VariableExpression(String variableName) {
        this.variableName = variableName;
    }

    @Override
    public boolean interpret(Map<String, Boolean> context) {
        return context.getOrDefault(variableName, false);
    }
}

// NON-TERMINAL EXPRESSIONS — composed of sub-expressions
class AndExpression implements BooleanExpression {
    private final BooleanExpression left;
    private final BooleanExpression right;

    public AndExpression(BooleanExpression left, BooleanExpression right) {
        this.left  = left;
        this.right = right;
    }

    @Override
    public boolean interpret(Map<String, Boolean> context) {
        return left.interpret(context) && right.interpret(context);
    }
}

class OrExpression implements BooleanExpression {
    private final BooleanExpression left;
    private final BooleanExpression right;

    public OrExpression(BooleanExpression left, BooleanExpression right) {
        this.left  = left;
        this.right = right;
    }

    @Override
    public boolean interpret(Map<String, Boolean> context) {
        return left.interpret(context) || right.interpret(context);
    }
}

class NotExpression implements BooleanExpression {
    private final BooleanExpression operand;

    public NotExpression(BooleanExpression operand) {
        this.operand = operand;
    }

    @Override
    public boolean interpret(Map<String, Boolean> context) {
        return !operand.interpret(context);
    }
}

public class InterpreterDemo {
    public static void main(String[] args) {
        // Build the expression tree for: (isAdmin OR isManager) AND NOT isGuest
        BooleanExpression isAdmin   = new VariableExpression("isAdmin");
        BooleanExpression isManager = new VariableExpression("isManager");
        BooleanExpression isGuest   = new VariableExpression("isGuest");

        BooleanExpression hasRole  = new OrExpression(isAdmin, isManager);
        BooleanExpression notGuest = new NotExpression(isGuest);
        BooleanExpression canAccess = new AndExpression(hasRole, notGuest);

        // Test with different contexts
        Map<String, Boolean> userContext1 = Map.of("isAdmin", true, "isGuest", false);
        Map<String, Boolean> userContext2 = Map.of("isManager", true, "isGuest", true);
        Map<String, Boolean> userContext3 = Map.of("isGuest", true);

        System.out.println("User1 (admin, not guest) can access: " + canAccess.interpret(userContext1)); // true
        System.out.println("User2 (manager, guest) can access: "  + canAccess.interpret(userContext2)); // false
        System.out.println("User3 (just guest) can access: "      + canAccess.interpret(userContext3)); // false
    }
}
```

---

### ⚠️ Important Points & Best Practices

- Interpreter is elegant for small grammars but becomes unmanageable for large ones — for complex language parsing, use a dedicated parser library instead.
- Java's `java.util.regex.Pattern` is a sophisticated interpreter. Spring's SpEL (Spring Expression Language) is another example.

---

## Summary Table — Behavioral Patterns

| Pattern | Intent | Key Mechanism | Java / Spring Example |
|---------|--------|---------------|-----------------------|
| Chain of Responsibility | Pass request along handler chain | Linked handler references | Servlet filters, Spring interceptors |
| Command | Encapsulate request as object | execute()/undo() interface | `Runnable`, `Callable`, Swing actions |
| Iterator | Traverse collection uniformly | `hasNext()`/`next()` interface | `java.util.Iterator`, enhanced for-each |
| Observer | Notify dependents on state change | Subscribe/notify pattern | Spring `ApplicationEvent`, GUI listeners |
| Strategy | Swap interchangeable algorithms | Inject strategy via interface | `Comparator`, payment processors |
| State | Change behavior based on state | State objects replace conditionals | Spring State Machine, workflow engines |
| Template Method | Define algorithm skeleton | Abstract base class with hooks | `JdbcTemplate`, `HttpServlet` |
| Mediator | Centralize many-to-many communication | All colleagues talk through mediator | Chat rooms, Spring `ApplicationEventPublisher` |
| Memento | Snapshot and restore object state | Immutable memento object | Undo/redo, game saves |
| Visitor | Add operations without modifying objects | Double dispatch `accept(visitor)` | Compiler ASTs, tax calculators |
| Interpreter | Parse and evaluate a grammar | Composite of expression objects | SpEL, `java.util.regex.Pattern` |

---

*Next: Phase 8.4 — Architectural Patterns →*

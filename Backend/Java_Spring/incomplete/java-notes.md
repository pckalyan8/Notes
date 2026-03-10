# Complete Java Backend Notes with Code Snippets

## Table of Contents
1. [Object-Oriented Programming (OOP)](#1-object-oriented-programming-oop)
2. [Java Basics](#2-java-basics)
3. [Exception Handling](#3-exception-handling)
4. [Collections Framework](#4-collections-framework)
5. [Multithreading and Concurrency](#5-multithreading-and-concurrency)
6. [Java 8+ Features](#6-java-8-features)
7. [Memory Management](#7-memory-management)
8. [Input/Output (I/O)](#8-inputoutput-io)

---

## 1. Object-Oriented Programming (OOP)

### 1.1 Classes and Objects

**Class**: A blueprint or template for creating objects.
**Object**: An instance of a class.

```java
// Class definition
public class Car {
    // Instance variables (attributes)
    private String brand;
    private String model;
    private int year;
    
    // Constructor
    public Car(String brand, String model, int year) {
        this.brand = brand;
        this.model = model;
        this.year = year;
    }
    
    // Method
    public void displayInfo() {
        System.out.println(year + " " + brand + " " + model);
    }
    
    // Getters and Setters
    public String getBrand() {
        return brand;
    }
    
    public void setBrand(String brand) {
        this.brand = brand;
    }
}

// Creating objects
public class Main {
    public static void main(String[] args) {
        Car car1 = new Car("Toyota", "Camry", 2022);
        Car car2 = new Car("Honda", "Accord", 2023);
        
        car1.displayInfo(); // Output: 2022 Toyota Camry
        car2.displayInfo(); // Output: 2023 Honda Accord
    }
}
```

### 1.2 Encapsulation

Encapsulation is wrapping data (variables) and code (methods) together as a single unit and restricting access to some components.

```java
public class BankAccount {
    // Private variables - cannot be accessed directly
    private String accountNumber;
    private double balance;
    
    public BankAccount(String accountNumber, double initialBalance) {
        this.accountNumber = accountNumber;
        this.balance = initialBalance;
    }
    
    // Public methods to access and modify private data
    public void deposit(double amount) {
        if (amount > 0) {
            balance += amount;
            System.out.println("Deposited: $" + amount);
        }
    }
    
    public void withdraw(double amount) {
        if (amount > 0 && amount <= balance) {
            balance -= amount;
            System.out.println("Withdrawn: $" + amount);
        } else {
            System.out.println("Insufficient funds");
        }
    }
    
    public double getBalance() {
        return balance;
    }
}
```

### 1.3 Inheritance

Inheritance allows a class to inherit properties and methods from another class.

```java
// Parent class (Superclass)
public class Animal {
    protected String name;
    protected int age;
    
    public Animal(String name, int age) {
        this.name = name;
        this.age = age;
    }
    
    public void eat() {
        System.out.println(name + " is eating");
    }
    
    public void sleep() {
        System.out.println(name + " is sleeping");
    }
}

// Child class (Subclass)
public class Dog extends Animal {
    private String breed;
    
    public Dog(String name, int age, String breed) {
        super(name, age); // Call parent constructor
        this.breed = breed;
    }
    
    // Method overriding
    @Override
    public void eat() {
        System.out.println(name + " is eating dog food");
    }
    
    // New method specific to Dog
    public void bark() {
        System.out.println(name + " is barking");
    }
}

// Usage
public class Main {
    public static void main(String[] args) {
        Dog dog = new Dog("Buddy", 3, "Golden Retriever");
        dog.eat();   // Output: Buddy is eating dog food
        dog.bark();  // Output: Buddy is barking
        dog.sleep(); // Output: Buddy is sleeping
    }
}
```

**Types of Inheritance:**

```java
// Single Inheritance
class A { }
class B extends A { }

// Multilevel Inheritance
class A { }
class B extends A { }
class C extends B { }

// Hierarchical Inheritance
class A { }
class B extends A { }
class C extends A { }

// Note: Java does NOT support multiple inheritance with classes
// But supports it through interfaces
```

### 1.4 Polymorphism

**Compile-time Polymorphism (Method Overloading):**

```java
public class Calculator {
    // Method overloading - same method name, different parameters
    public int add(int a, int b) {
        return a + b;
    }
    
    public int add(int a, int b, int c) {
        return a + b + c;
    }
    
    public double add(double a, double b) {
        return a + b;
    }
    
    public static void main(String[] args) {
        Calculator calc = new Calculator();
        System.out.println(calc.add(5, 10));        // Output: 15
        System.out.println(calc.add(5, 10, 15));    // Output: 30
        System.out.println(calc.add(5.5, 10.5));    // Output: 16.0
    }
}
```

**Runtime Polymorphism (Method Overriding):**

```java
class Shape {
    public void draw() {
        System.out.println("Drawing a shape");
    }
    
    public double area() {
        return 0;
    }
}

class Circle extends Shape {
    private double radius;
    
    public Circle(double radius) {
        this.radius = radius;
    }
    
    @Override
    public void draw() {
        System.out.println("Drawing a circle");
    }
    
    @Override
    public double area() {
        return Math.PI * radius * radius;
    }
}

class Rectangle extends Shape {
    private double width, height;
    
    public Rectangle(double width, double height) {
        this.width = width;
        this.height = height;
    }
    
    @Override
    public void draw() {
        System.out.println("Drawing a rectangle");
    }
    
    @Override
    public double area() {
        return width * height;
    }
}

// Usage - Polymorphic behavior
public class Main {
    public static void main(String[] args) {
        Shape shape1 = new Circle(5);
        Shape shape2 = new Rectangle(4, 6);
        
        shape1.draw();  // Output: Drawing a circle
        shape2.draw();  // Output: Drawing a rectangle
        
        System.out.println("Circle area: " + shape1.area());
        System.out.println("Rectangle area: " + shape2.area());
    }
}
```

### 1.5 Abstraction

**Abstract Classes:**

```java
// Abstract class
public abstract class Vehicle {
    protected String brand;
    
    public Vehicle(String brand) {
        this.brand = brand;
    }
    
    // Abstract method - no implementation
    public abstract void start();
    
    // Concrete method
    public void stop() {
        System.out.println(brand + " is stopping");
    }
}

// Concrete class implementing abstract class
public class Car extends Vehicle {
    public Car(String brand) {
        super(brand);
    }
    
    @Override
    public void start() {
        System.out.println(brand + " car is starting with ignition");
    }
}

public class Motorcycle extends Vehicle {
    public Motorcycle(String brand) {
        super(brand);
    }
    
    @Override
    public void start() {
        System.out.println(brand + " motorcycle is starting with kick");
    }
}
```

**Interfaces:**

```java
// Interface
public interface Drawable {
    // All methods are public and abstract by default
    void draw();
    
    // Can have default methods (Java 8+)
    default void display() {
        System.out.println("Displaying drawable object");
    }
    
    // Can have static methods (Java 8+)
    static void info() {
        System.out.println("This is a drawable interface");
    }
}

public interface Resizable {
    void resize(int width, int height);
}

// Class implementing multiple interfaces
public class Image implements Drawable, Resizable {
    private String name;
    private int width;
    private int height;
    
    public Image(String name, int width, int height) {
        this.name = name;
        this.width = width;
        this.height = height;
    }
    
    @Override
    public void draw() {
        System.out.println("Drawing image: " + name);
    }
    
    @Override
    public void resize(int width, int height) {
        this.width = width;
        this.height = height;
        System.out.println("Image resized to " + width + "x" + height);
    }
}
```

**Abstract Class vs Interface:**

```java
// When to use Abstract Class:
// - When you want to share code among closely related classes
// - When you need non-final or non-static fields
// - When you need to declare non-public members

// When to use Interface:
// - When unrelated classes implement your interface
// - When you want to specify behavior but not how it's implemented
// - To achieve multiple inheritance
```

### 1.6 SOLID Principles

**S - Single Responsibility Principle:**

```java
// Bad - Multiple responsibilities
class User {
    public void createUser() { }
    public void sendEmail() { }
    public void saveToDatabase() { }
}

// Good - Single responsibility
class User {
    private String name;
    private String email;
    // User-related fields and methods only
}

class EmailService {
    public void sendEmail(User user) {
        // Email sending logic
    }
}

class UserRepository {
    public void save(User user) {
        // Database logic
    }
}
```

**O - Open/Closed Principle:**

```java
// Classes should be open for extension but closed for modification

// Bad approach
class AreaCalculator {
    public double calculateArea(Object shape) {
        if (shape instanceof Circle) {
            Circle circle = (Circle) shape;
            return Math.PI * circle.radius * circle.radius;
        } else if (shape instanceof Rectangle) {
            Rectangle rect = (Rectangle) shape;
            return rect.width * rect.height;
        }
        return 0;
    }
}

// Good approach - Using abstraction
interface Shape {
    double calculateArea();
}

class Circle implements Shape {
    private double radius;
    
    public Circle(double radius) {
        this.radius = radius;
    }
    
    @Override
    public double calculateArea() {
        return Math.PI * radius * radius;
    }
}

class Rectangle implements Shape {
    private double width;
    private double height;
    
    public Rectangle(double width, double height) {
        this.width = width;
        this.height = height;
    }
    
    @Override
    public double calculateArea() {
        return width * height;
    }
}

class AreaCalculator {
    public double calculateArea(Shape shape) {
        return shape.calculateArea();
    }
}
```

**L - Liskov Substitution Principle:**

```java
// Subtypes must be substitutable for their base types

class Bird {
    public void fly() {
        System.out.println("Bird is flying");
    }
}

// Bad - Penguin is a bird but can't fly
class Penguin extends Bird {
    @Override
    public void fly() {
        throw new UnsupportedOperationException("Penguins can't fly");
    }
}

// Good approach
abstract class Bird {
    public abstract void move();
}

class Sparrow extends Bird {
    @Override
    public void move() {
        System.out.println("Sparrow is flying");
    }
}

class Penguin extends Bird {
    @Override
    public void move() {
        System.out.println("Penguin is swimming");
    }
}
```

**I - Interface Segregation Principle:**

```java
// Clients should not be forced to depend on interfaces they don't use

// Bad - Fat interface
interface Worker {
    void work();
    void eat();
    void sleep();
}

// Good - Segregated interfaces
interface Workable {
    void work();
}

interface Eatable {
    void eat();
}

interface Sleepable {
    void sleep();
}

class Human implements Workable, Eatable, Sleepable {
    @Override
    public void work() {
        System.out.println("Human working");
    }
    
    @Override
    public void eat() {
        System.out.println("Human eating");
    }
    
    @Override
    public void sleep() {
        System.out.println("Human sleeping");
    }
}

class Robot implements Workable {
    @Override
    public void work() {
        System.out.println("Robot working");
    }
}
```

**D - Dependency Inversion Principle:**

```java
// Depend on abstractions, not concretions

// Bad - High-level module depends on low-level module
class MySQLDatabase {
    public void save(String data) {
        System.out.println("Saving to MySQL: " + data);
    }
}

class UserService {
    private MySQLDatabase database = new MySQLDatabase();
    
    public void saveUser(String user) {
        database.save(user);
    }
}

// Good - Both depend on abstraction
interface Database {
    void save(String data);
}

class MySQLDatabase implements Database {
    @Override
    public void save(String data) {
        System.out.println("Saving to MySQL: " + data);
    }
}

class MongoDatabase implements Database {
    @Override
    public void save(String data) {
        System.out.println("Saving to MongoDB: " + data);
    }
}

class UserService {
    private Database database;
    
    // Dependency injection
    public UserService(Database database) {
        this.database = database;
    }
    
    public void saveUser(String user) {
        database.save(user);
    }
}

// Usage
public class Main {
    public static void main(String[] args) {
        Database mysql = new MySQLDatabase();
        UserService service1 = new UserService(mysql);
        service1.saveUser("John");
        
        Database mongo = new MongoDatabase();
        UserService service2 = new UserService(mongo);
        service2.saveUser("Jane");
    }
}
```

### 1.7 Design Patterns

**Singleton Pattern:**

```java
// Ensures only one instance of a class exists

// Thread-safe singleton
public class DatabaseConnection {
    private static volatile DatabaseConnection instance;
    
    private DatabaseConnection() {
        // Private constructor prevents instantiation
    }
    
    public static DatabaseConnection getInstance() {
        if (instance == null) {
            synchronized (DatabaseConnection.class) {
                if (instance == null) {
                    instance = new DatabaseConnection();
                }
            }
        }
        return instance;
    }
    
    public void connect() {
        System.out.println("Connected to database");
    }
}

// Eager initialization (simpler, thread-safe)
public class Logger {
    private static final Logger instance = new Logger();
    
    private Logger() { }
    
    public static Logger getInstance() {
        return instance;
    }
    
    public void log(String message) {
        System.out.println("LOG: " + message);
    }
}
```

**Factory Pattern:**

```java
// Creates objects without exposing creation logic

interface Vehicle {
    void drive();
}

class Car implements Vehicle {
    @Override
    public void drive() {
        System.out.println("Driving a car");
    }
}

class Bike implements Vehicle {
    @Override
    public void drive() {
        System.out.println("Riding a bike");
    }
}

class Truck implements Vehicle {
    @Override
    public void drive() {
        System.out.println("Driving a truck");
    }
}

// Factory class
class VehicleFactory {
    public static Vehicle createVehicle(String type) {
        switch (type.toLowerCase()) {
            case "car":
                return new Car();
            case "bike":
                return new Bike();
            case "truck":
                return new Truck();
            default:
                throw new IllegalArgumentException("Unknown vehicle type");
        }
    }
}

// Usage
public class Main {
    public static void main(String[] args) {
        Vehicle car = VehicleFactory.createVehicle("car");
        car.drive();
        
        Vehicle bike = VehicleFactory.createVehicle("bike");
        bike.drive();
    }
}
```

**Builder Pattern:**

```java
// Constructs complex objects step by step

public class User {
    // Required parameters
    private final String firstName;
    private final String lastName;
    
    // Optional parameters
    private final int age;
    private final String phone;
    private final String address;
    
    private User(UserBuilder builder) {
        this.firstName = builder.firstName;
        this.lastName = builder.lastName;
        this.age = builder.age;
        this.phone = builder.phone;
        this.address = builder.address;
    }
    
    // Static nested Builder class
    public static class UserBuilder {
        private final String firstName;
        private final String lastName;
        private int age;
        private String phone;
        private String address;
        
        public UserBuilder(String firstName, String lastName) {
            this.firstName = firstName;
            this.lastName = lastName;
        }
        
        public UserBuilder age(int age) {
            this.age = age;
            return this;
        }
        
        public UserBuilder phone(String phone) {
            this.phone = phone;
            return this;
        }
        
        public UserBuilder address(String address) {
            this.address = address;
            return this;
        }
        
        public User build() {
            return new User(this);
        }
    }
    
    @Override
    public String toString() {
        return "User{" +
                "firstName='" + firstName + '\'' +
                ", lastName='" + lastName + '\'' +
                ", age=" + age +
                ", phone='" + phone + '\'' +
                ", address='" + address + '\'' +
                '}';
    }
}

// Usage
public class Main {
    public static void main(String[] args) {
        User user = new User.UserBuilder("John", "Doe")
                .age(30)
                .phone("123-456-7890")
                .address("123 Main St")
                .build();
        
        System.out.println(user);
    }
}
```

**Observer Pattern:**

```java
// Defines one-to-many dependency between objects

import java.util.ArrayList;
import java.util.List;

// Subject interface
interface Subject {
    void attach(Observer observer);
    void detach(Observer observer);
    void notifyObservers();
}

// Observer interface
interface Observer {
    void update(String message);
}

// Concrete Subject
class NewsAgency implements Subject {
    private List<Observer> observers = new ArrayList<>();
    private String news;
    
    @Override
    public void attach(Observer observer) {
        observers.add(observer);
    }
    
    @Override
    public void detach(Observer observer) {
        observers.remove(observer);
    }
    
    @Override
    public void notifyObservers() {
        for (Observer observer : observers) {
            observer.update(news);
        }
    }
    
    public void setNews(String news) {
        this.news = news;
        notifyObservers();
    }
}

// Concrete Observers
class NewsChannel implements Observer {
    private String name;
    
    public NewsChannel(String name) {
        this.name = name;
    }
    
    @Override
    public void update(String message) {
        System.out.println(name + " received news: " + message);
    }
}

// Usage
public class Main {
    public static void main(String[] args) {
        NewsAgency agency = new NewsAgency();
        
        NewsChannel cnn = new NewsChannel("CNN");
        NewsChannel bbc = new NewsChannel("BBC");
        
        agency.attach(cnn);
        agency.attach(bbc);
        
        agency.setNews("Breaking: New Java version released!");
    }
}
```

**Strategy Pattern:**

```java
// Defines a family of algorithms and makes them interchangeable

// Strategy interface
interface PaymentStrategy {
    void pay(int amount);
}

// Concrete strategies
class CreditCardPayment implements PaymentStrategy {
    private String cardNumber;
    
    public CreditCardPayment(String cardNumber) {
        this.cardNumber = cardNumber;
    }
    
    @Override
    public void pay(int amount) {
        System.out.println("Paid $" + amount + " using Credit Card: " + cardNumber);
    }
}

class PayPalPayment implements PaymentStrategy {
    private String email;
    
    public PayPalPayment(String email) {
        this.email = email;
    }
    
    @Override
    public void pay(int amount) {
        System.out.println("Paid $" + amount + " using PayPal: " + email);
    }
}

class BitcoinPayment implements PaymentStrategy {
    private String walletAddress;
    
    public BitcoinPayment(String walletAddress) {
        this.walletAddress = walletAddress;
    }
    
    @Override
    public void pay(int amount) {
        System.out.println("Paid $" + amount + " using Bitcoin: " + walletAddress);
    }
}

// Context
class ShoppingCart {
    private PaymentStrategy paymentStrategy;
    
    public void setPaymentStrategy(PaymentStrategy paymentStrategy) {
        this.paymentStrategy = paymentStrategy;
    }
    
    public void checkout(int amount) {
        paymentStrategy.pay(amount);
    }
}

// Usage
public class Main {
    public static void main(String[] args) {
        ShoppingCart cart = new ShoppingCart();
        
        cart.setPaymentStrategy(new CreditCardPayment("1234-5678-9012-3456"));
        cart.checkout(100);
        
        cart.setPaymentStrategy(new PayPalPayment("user@email.com"));
        cart.checkout(200);
    }
}
```

---

## 2. Java Basics

### 2.1 Data Types

**Primitive Data Types:**

```java
public class DataTypes {
    public static void main(String[] args) {
        // Integer types
        byte byteVar = 127;              // 8-bit: -128 to 127
        short shortVar = 32767;          // 16-bit: -32,768 to 32,767
        int intVar = 2147483647;         // 32-bit: -2^31 to 2^31-1
        long longVar = 9223372036854775807L; // 64-bit: -2^63 to 2^63-1
        
        // Floating-point types
        float floatVar = 3.14f;          // 32-bit: ~6-7 decimal digits
        double doubleVar = 3.14159265359; // 64-bit: ~15 decimal digits
        
        // Character type
        char charVar = 'A';              // 16-bit Unicode character
        
        // Boolean type
        boolean boolVar = true;          // true or false
        
        System.out.println("byte: " + byteVar);
        System.out.println("int: " + intVar);
        System.out.println("double: " + doubleVar);
        System.out.println("char: " + charVar);
        System.out.println("boolean: " + boolVar);
    }
}
```

**Reference Data Types:**

```java
public class ReferenceTypes {
    public static void main(String[] args) {
        // String
        String name = "John Doe";
        
        // Arrays
        int[] numbers = {1, 2, 3, 4, 5};
        String[] names = new String[3];
        
        // Objects
        Person person = new Person("Alice", 25);
        
        // Null reference
        String nullString = null;
    }
}

class Person {
    String name;
    int age;
    
    Person(String name, int age) {
        this.name = name;
        this.age = age;
    }
}
```

### 2.2 Type Casting

```java
public class TypeCasting {
    public static void main(String[] args) {
        // Widening (Implicit) - smaller to larger
        int intVal = 100;
        long longVal = intVal;        // int to long
        double doubleVal = intVal;    // int to double
        
        System.out.println("Widening:");
        System.out.println("int: " + intVal);
        System.out.println("long: " + longVal);
        System.out.println("double: " + doubleVal);
        
        // Narrowing (Explicit) - larger to smaller
        double d = 9.78;
        int i = (int) d;              // double to int (loses decimal)
        
        System.out.println("\nNarrowing:");
        System.out.println("double: " + d);
        System.out.println("int: " + i);
        
        // Object type casting
        Animal animal = new Dog();    // Upcasting (implicit)
        Dog dog = (Dog) animal;       // Downcasting (explicit)
    }
}

class Animal { }
class Dog extends Animal { }
```

### 2.3 Wrapper Classes

```java
public class WrapperClassDemo {
    public static void main(String[] args) {
        // Primitive to Wrapper (Boxing)
        int primitiveInt = 10;
        Integer wrapperInt = Integer.valueOf(primitiveInt);
        
        // Wrapper to Primitive (Unboxing)
        Integer wrapper = 20;
        int primitive = wrapper.intValue();
        
        // Autoboxing (automatic boxing)
        Integer autoBoxed = 30;
        
        // Auto-unboxing (automatic unboxing)
        int autoUnboxed = autoBoxed;
        
        // Wrapper class methods
        System.out.println("Parse string to int: " + Integer.parseInt("123"));
        System.out.println("Max value: " + Integer.MAX_VALUE);
        System.out.println("Min value: " + Integer.MIN_VALUE);
        System.out.println("Compare: " + Integer.compare(10, 20));
        
        // All primitive wrapper classes
        Byte b = 1;
        Short s = 2;
        Integer i = 3;
        Long l = 4L;
        Float f = 5.0f;
        Double d = 6.0;
        Character c = 'A';
        Boolean bool = true;
    }
}
```

### 2.4 Strings

```java
public class StringExamples {
    public static void main(String[] args) {
        // String creation
        String str1 = "Hello";              // String literal (in String pool)
        String str2 = new String("Hello");  // Using new keyword (in heap)
        
        // String methods
        String text = "  Java Programming  ";
        
        System.out.println("Length: " + text.length());
        System.out.println("Uppercase: " + text.toUpperCase());
        System.out.println("Lowercase: " + text.toLowerCase());
        System.out.println("Trim: '" + text.trim() + "'");
        System.out.println("Substring: " + text.substring(2, 6));
        System.out.println("Replace: " + text.replace("Java", "Python"));
        System.out.println("Contains: " + text.contains("Java"));
        System.out.println("Starts with: " + text.startsWith("  Java"));
        System.out.println("Ends with: " + text.endsWith("ing  "));
        
        // String comparison
        String s1 = "Hello";
        String s2 = "Hello";
        String s3 = new String("Hello");
        
        System.out.println("\nComparison:");
        System.out.println("s1 == s2: " + (s1 == s2));           // true (same reference)
        System.out.println("s1 == s3: " + (s1 == s3));           // false (different reference)
        System.out.println("s1.equals(s3): " + s1.equals(s3));   // true (same content)
        
        // String concatenation
        String first = "Hello";
        String last = "World";
        String full = first + " " + last;
        System.out.println("\nConcatenation: " + full);
        
        // StringBuilder (mutable, not thread-safe)
        StringBuilder sb = new StringBuilder("Hello");
        sb.append(" World");
        sb.insert(5, ",");
        sb.delete(5, 6);
        System.out.println("StringBuilder: " + sb.toString());
        
        // StringBuffer (mutable, thread-safe)
        StringBuffer sbf = new StringBuffer("Hello");
        sbf.append(" World");
        System.out.println("StringBuffer: " + sbf.toString());
        
        // String formatting
        String formatted = String.format("Name: %s, Age: %d, Score: %.2f", 
                                        "John", 25, 95.5678);
        System.out.println("\nFormatted: " + formatted);
    }
}
```

### 2.5 Arrays

```java
public class ArrayExamples {
    public static void main(String[] args) {
        // Array declaration and initialization
        int[] arr1 = new int[5];                    // Default values (0)
        int[] arr2 = {1, 2, 3, 4, 5};              // Direct initialization
        int[] arr3 = new int[]{10, 20, 30};        // Array creation
        
        // Accessing elements
        System.out.println("First element: " + arr2[0]);
        System.out.println("Last element: " + arr2[arr2.length - 1]);
        
        // Modifying elements
        arr2[0] = 100;
        System.out.println("Modified first element: " + arr2[0]);
        
        // Iterating through array
        System.out.println("\nFor loop:");
        for (int i = 0; i < arr2.length; i++) {
            System.out.print(arr2[i] + " ");
        }
        
        System.out.println("\n\nEnhanced for loop:");
        for (int num : arr2) {
            System.out.print(num + " ");
        }
        
        // Multi-dimensional arrays
        int[][] matrix = {
            {1, 2, 3},
            {4, 5, 6},
            {7, 8, 9}
        };
        
        System.out.println("\n\nMatrix:");
        for (int i = 0; i < matrix.length; i++) {
            for (int j = 0; j < matrix[i].length; j++) {
                System.out.print(matrix[i][j] + " ");
            }
            System.out.println();
        }
        
        // Array copying
        int[] original = {1, 2, 3, 4, 5};
        int[] copy = original.clone();
        int[] copy2 = new int[original.length];
        System.arraycopy(original, 0, copy2, 0, original.length);
        
        // Array sorting
        int[] unsorted = {5, 2, 8, 1, 9};
        java.util.Arrays.sort(unsorted);
        System.out.println("\nSorted: " + java.util.Arrays.toString(unsorted));
        
        // Array searching (must be sorted)
        int index = java.util.Arrays.binarySearch(unsorted, 8);
        System.out.println("Index of 8: " + index);
    }
}
```

### 2.6 Operators

```java
public class Operators {
    public static void main(String[] args) {
        int a = 10, b = 5;
        
        // Arithmetic operators
        System.out.println("Arithmetic Operators:");
        System.out.println("a + b = " + (a + b));
        System.out.println("a - b = " + (a - b));
        System.out.println("a * b = " + (a * b));
        System.out.println("a / b = " + (a / b));
        System.out.println("a % b = " + (a % b));
        
        // Unary operators
        System.out.println("\nUnary Operators:");
        int c = 10;
        System.out.println("c++ = " + (c++));  // Post-increment
        System.out.println("c = " + c);
        System.out.println("++c = " + (++c));  // Pre-increment
        System.out.println("c-- = " + (c--));  // Post-decrement
        System.out.println("--c = " + (--c));  // Pre-decrement
        
        // Relational operators
        System.out.println("\nRelational Operators:");
        System.out.println("a == b: " + (a == b));
        System.out.println("a != b: " + (a != b));
        System.out.println("a > b: " + (a > b));
        System.out.println("a < b: " + (a < b));
        System.out.println("a >= b: " + (a >= b));
        System.out.println("a <= b: " + (a <= b));
        
        // Logical operators
        System.out.println("\nLogical Operators:");
        boolean x = true, y = false;
        System.out.println("x && y: " + (x && y));  // AND
        System.out.println("x || y: " + (x || y));  // OR
        System.out.println("!x: " + (!x));          // NOT
        
        // Bitwise operators
        System.out.println("\nBitwise Operators:");
        int p = 5;  // 0101 in binary
        int q = 3;  // 0011 in binary
        System.out.println("p & q: " + (p & q));    // AND: 0001 = 1
        System.out.println("p | q: " + (p | q));    // OR:  0111 = 7
        System.out.println("p ^ q: " + (p ^ q));    // XOR: 0110 = 6
        System.out.println("~p: " + (~p));          // NOT
        System.out.println("p << 1: " + (p << 1));  // Left shift
        System.out.println("p >> 1: " + (p >> 1));  // Right shift
        
        // Ternary operator
        System.out.println("\nTernary Operator:");
        int max = (a > b) ? a : b;
        System.out.println("Max: " + max);
        
        // Assignment operators
        System.out.println("\nAssignment Operators:");
        int n = 10;
        n += 5;  // n = n + 5
        System.out.println("n += 5: " + n);
        n -= 3;  // n = n - 3
        System.out.println("n -= 3: " + n);
        n *= 2;  // n = n * 2
        System.out.println("n *= 2: " + n);
        n /= 4;  // n = n / 4
        System.out.println("n /= 4: " + n);
        n %= 3;  // n = n % 3
        System.out.println("n %= 3: " + n);
    }
}
```

### 2.7 Control Flow Statements

```java
public class ControlFlow {
    public static void main(String[] args) {
        // if-else statement
        int age = 20;
        if (age >= 18) {
            System.out.println("Adult");
        } else if (age >= 13) {
            System.out.println("Teenager");
        } else {
            System.out.println("Child");
        }
        
        // switch statement
        int day = 3;
        switch (day) {
            case 1:
                System.out.println("Monday");
                break;
            case 2:
                System.out.println("Tuesday");
                break;
            case 3:
                System.out.println("Wednesday");
                break;
            default:
                System.out.println("Other day");
        }
        
        // Enhanced switch (Java 14+)
        String dayName = switch (day) {
            case 1 -> "Monday";
            case 2 -> "Tuesday";
            case 3 -> "Wednesday";
            default -> "Other day";
        };
        System.out.println("Day: " + dayName);
        
        // for loop
        System.out.println("\nFor loop:");
        for (int i = 1; i <= 5; i++) {
            System.out.print(i + " ");
        }
        
        // while loop
        System.out.println("\n\nWhile loop:");
        int i = 1;
        while (i <= 5) {
            System.out.print(i + " ");
            i++;
        }
        
        // do-while loop
        System.out.println("\n\nDo-while loop:");
        int j = 1;
        do {
            System.out.print(j + " ");
            j++;
        } while (j <= 5);
        
        // Enhanced for loop
        System.out.println("\n\nEnhanced for loop:");
        int[] numbers = {10, 20, 30, 40, 50};
        for (int num : numbers) {
            System.out.print(num + " ");
        }
        
        // break statement
        System.out.println("\n\nBreak example:");
        for (int k = 1; k <= 10; k++) {
            if (k == 5) {
                break;  // Exit loop when k is 5
            }
            System.out.print(k + " ");
        }
        
        // continue statement
        System.out.println("\n\nContinue example:");
        for (int k = 1; k <= 10; k++) {
            if (k % 2 == 0) {
                continue;  // Skip even numbers
            }
            System.out.print(k + " ");
        }
        
        // labeled break
        System.out.println("\n\nLabeled break:");
        outer: for (int x = 1; x <= 3; x++) {
            for (int y = 1; y <= 3; y++) {
                if (x == 2 && y == 2) {
                    break outer;  // Break out of both loops
                }
                System.out.println("x=" + x + ", y=" + y);
            }
        }
    }
}
```

---

## 3. Exception Handling

### 3.1 Exception Hierarchy

```java
/*
Throwable
├── Error (Unchecked)
│   ├── OutOfMemoryError
│   ├── StackOverflowError
│   └── ...
└── Exception
    ├── RuntimeException (Unchecked)
    │   ├── NullPointerException
    │   ├── ArrayIndexOutOfBoundsException
    │   ├── ArithmeticException
    │   └── ...
    └── Checked Exceptions
        ├── IOException
        ├── SQLException
        ├── ClassNotFoundException
        └── ...
*/
```

### 3.2 Try-Catch-Finally

```java
public class ExceptionHandling {
    public static void main(String[] args) {
        // Basic try-catch
        try {
            int result = 10 / 0;  // ArithmeticException
        } catch (ArithmeticException e) {
            System.out.println("Cannot divide by zero");
        }
        
        // Multiple catch blocks
        try {
            int[] arr = new int[5];
            arr[10] = 50;  // ArrayIndexOutOfBoundsException
        } catch (ArrayIndexOutOfBoundsException e) {
            System.out.println("Array index out of bounds");
        } catch (Exception e) {
            System.out.println("General exception");
        }
        
        // Multi-catch (Java 7+)
        try {
            String str = null;
            str.length();
        } catch (NullPointerException | ArithmeticException e) {
            System.out.println("Exception: " + e.getMessage());
        }
        
        // Try-catch-finally
        try {
            int result = 10 / 2;
            System.out.println("Result: " + result);
        } catch (Exception e) {
            System.out.println("Exception occurred");
        } finally {
            System.out.println("Finally block always executes");
        }
        
        // Getting exception information
        try {
            int[] arr = {1, 2, 3};
            System.out.println(arr[5]);
        } catch (Exception e) {
            System.out.println("\nException details:");
            System.out.println("Message: " + e.getMessage());
            System.out.println("Class: " + e.getClass().getName());
            System.out.println("Stack trace:");
            e.printStackTrace();
        }
    }
}
```

### 3.3 Throw and Throws

```java
public class ThrowThrows {
    
    // throws - declares exceptions that might be thrown
    public static void checkAge(int age) throws IllegalArgumentException {
        if (age < 18) {
            // throw - actually throws an exception
            throw new IllegalArgumentException("Age must be 18 or above");
        }
        System.out.println("Age is valid");
    }
    
    // Checked exception - must be caught or declared
    public static void readFile(String filename) throws IOException {
        FileReader file = new FileReader(filename);
        // File operations
        file.close();
    }
    
    // Multiple exceptions
    public static void processData(String data) 
            throws IOException, SQLException {
        // Method implementation
    }
    
    public static void main(String[] args) {
        // Using method that throws unchecked exception
        try {
            checkAge(15);
        } catch (IllegalArgumentException e) {
            System.out.println("Error: " + e.getMessage());
        }
        
        // Using method that throws checked exception
        try {
            readFile("data.txt");
        } catch (IOException e) {
            System.out.println("File error: " + e.getMessage());
        }
        
        // Throwing an exception directly
        try {
            throw new RuntimeException("Custom error message");
        } catch (RuntimeException e) {
            System.out.println("Caught: " + e.getMessage());
        }
    }
}
```

### 3.4 Custom Exceptions

```java
// Custom checked exception
class InsufficientFundsException extends Exception {
    private double amount;
    
    public InsufficientFundsException(double amount) {
        super("Insufficient funds. Need $" + amount + " more.");
        this.amount = amount;
    }
    
    public double getAmount() {
        return amount;
    }
}

// Custom unchecked exception
class InvalidAccountException extends RuntimeException {
    public InvalidAccountException(String message) {
        super(message);
    }
}

class BankAccount {
    private String accountNumber;
    private double balance;
    
    public BankAccount(String accountNumber, double balance) {
        if (accountNumber == null || accountNumber.isEmpty()) {
            throw new InvalidAccountException("Account number cannot be empty");
        }
        this.accountNumber = accountNumber;
        this.balance = balance;
    }
    
    public void withdraw(double amount) throws InsufficientFundsException {
        if (amount > balance) {
            double shortage = amount - balance;
            throw new InsufficientFundsException(shortage);
        }
        balance -= amount;
        System.out.println("Withdrawal successful. New balance: $" + balance);
    }
    
    public double getBalance() {
        return balance;
    }
}

public class CustomExceptionDemo {
    public static void main(String[] args) {
        try {
            BankAccount account = new BankAccount("ACC001", 1000);
            account.withdraw(1500);
        } catch (InsufficientFundsException e) {
            System.out.println("Error: " + e.getMessage());
            System.out.println("Short by: $" + e.getAmount());
        }
        
        try {
            BankAccount invalidAccount = new BankAccount("", 1000);
        } catch (InvalidAccountException e) {
            System.out.println("Error: " + e.getMessage());
        }
    }
}
```

### 3.5 Try-with-Resources

```java
import java.io.*;

public class TryWithResources {
    
    // Before Java 7 - manual resource management
    public static void oldWay() {
        BufferedReader reader = null;
        try {
            reader = new BufferedReader(new FileReader("file.txt"));
            String line = reader.readLine();
            System.out.println(line);
        } catch (IOException e) {
            e.printStackTrace();
        } finally {
            try {
                if (reader != null) {
                    reader.close();
                }
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }
    
    // Java 7+ - try-with-resources (automatic resource management)
    public static void newWay() {
        try (BufferedReader reader = new BufferedReader(new FileReader("file.txt"))) {
            String line = reader.readLine();
            System.out.println(line);
        } catch (IOException e) {
            e.printStackTrace();
        }
        // reader is automatically closed
    }
    
    // Multiple resources
    public static void copyFile(String source, String destination) {
        try (
            BufferedReader reader = new BufferedReader(new FileReader(source));
            BufferedWriter writer = new BufferedWriter(new FileWriter(destination))
        ) {
            String line;
            while ((line = reader.readLine()) != null) {
                writer.write(line);
                writer.newLine();
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
    
    // Custom AutoCloseable resource
    static class DatabaseConnection implements AutoCloseable {
        public DatabaseConnection() {
            System.out.println("Connection opened");
        }
        
        public void executeQuery(String query) {
            System.out.println("Executing: " + query);
        }
        
        @Override
        public void close() {
            System.out.println("Connection closed");
        }
    }
    
    public static void main(String[] args) {
        // Using custom AutoCloseable
        try (DatabaseConnection conn = new DatabaseConnection()) {
            conn.executeQuery("SELECT * FROM users");
        }
        // Connection automatically closed
    }
}
```

---

## 4. Collections Framework

### 4.1 Collection Hierarchy

```java
/*
Collection (Interface)
├── List (Interface) - Ordered, allows duplicates
│   ├── ArrayList (Class)
│   ├── LinkedList (Class)
│   └── Vector (Class)
│       └── Stack (Class)
├── Set (Interface) - No duplicates
│   ├── HashSet (Class)
│   ├── LinkedHashSet (Class)
│   └── SortedSet (Interface)
│       └── TreeSet (Class)
└── Queue (Interface) - FIFO
    ├── PriorityQueue (Class)
    └── Deque (Interface)
        └── ArrayDeque (Class)

Map (Interface) - Key-value pairs
├── HashMap (Class)
├── LinkedHashMap (Class)
├── Hashtable (Class)
└── SortedMap (Interface)
    └── TreeMap (Class)
*/
```

### 4.2 ArrayList

```java
import java.util.*;

public class ArrayListExample {
    public static void main(String[] args) {
        // Creating ArrayList
        ArrayList<String> list = new ArrayList<>();
        
        // Adding elements
        list.add("Apple");
        list.add("Banana");
        list.add("Cherry");
        list.add(1, "Mango");  // Add at specific index
        
        System.out.println("ArrayList: " + list);
        
        // Accessing elements
        System.out.println("Element at index 0: " + list.get(0));
        System.out.println("First element: " + list.getFirst());  // Java 21+
        System.out.println("Last element: " + list.getLast());    // Java 21+
        
        // Size
        System.out.println("Size: " + list.size());
        
        // Checking if element exists
        System.out.println("Contains 'Banana': " + list.contains("Banana"));
        
        // Index of element
        System.out.println("Index of 'Cherry': " + list.indexOf("Cherry"));
        
        // Updating elements
        list.set(0, "Apricot");
        System.out.println("After update: " + list);
        
        // Removing elements
        list.remove("Banana");           // Remove by object
        list.remove(0);                  // Remove by index
        System.out.println("After removal: " + list);
        
        // Iterating
        System.out.println("\nIterating:");
        
        // For-each loop
        for (String fruit : list) {
            System.out.println(fruit);
        }
        
        // Iterator
        Iterator<String> iterator = list.iterator();
        while (iterator.hasNext()) {
            System.out.println(iterator.next());
        }
        
        // forEach with lambda (Java 8+)
        list.forEach(fruit -> System.out.println(fruit));
        
        // Sorting
        Collections.sort(list);
        System.out.println("\nSorted: " + list);
        
        // Reversing
        Collections.reverse(list);
        System.out.println("Reversed: " + list);
        
        // Converting to array
        String[] array = list.toArray(new String[0]);
        
        // Creating from array
        String[] fruits = {"Apple", "Banana", "Cherry"};
        List<String> list2 = Arrays.asList(fruits);
        
        // Clearing list
        list.clear();
        System.out.println("After clear: " + list);
        System.out.println("Is empty: " + list.isEmpty());
    }
}
```

### 4.3 LinkedList

```java
import java.util.*;

public class LinkedListExample {
    public static void main(String[] args) {
        LinkedList<String> linkedList = new LinkedList<>();
        
        // Adding elements
        linkedList.add("First");
        linkedList.add("Second");
        linkedList.add("Third");
        
        // Adding at beginning and end
        linkedList.addFirst("Start");
        linkedList.addLast("End");
        
        System.out.println("LinkedList: " + linkedList);
        
        // Getting first and last
        System.out.println("First: " + linkedList.getFirst());
        System.out.println("Last: " + linkedList.getLast());
        
        // Removing first and last
        linkedList.removeFirst();
        linkedList.removeLast();
        System.out.println("After removal: " + linkedList);
        
        // Using as Queue (FIFO)
        LinkedList<String> queue = new LinkedList<>();
        queue.offer("Task 1");  // Add to end
        queue.offer("Task 2");
        queue.offer("Task 3");
        
        System.out.println("\nQueue operations:");
        System.out.println("Poll (remove from front): " + queue.poll());
        System.out.println("Peek (view front): " + queue.peek());
        
        // Using as Stack (LIFO)
        LinkedList<String> stack = new LinkedList<>();
        stack.push("A");  // Add to front
        stack.push("B");
        stack.push("C");
        
        System.out.println("\nStack operations:");
        System.out.println("Pop: " + stack.pop());  // Remove from front
        System.out.println("Peek: " + stack.peek());
    }
}
```

### 4.4 HashSet

```java
import java.util.*;

public class HashSetExample {
    public static void main(String[] args) {
        HashSet<String> set = new HashSet<>();
        
        // Adding elements
        set.add("Apple");
        set.add("Banana");
        set.add("Cherry");
        set.add("Apple");  // Duplicate - will not be added
        
        System.out.println("HashSet: " + set);
        System.out.println("Size: " + set.size());
        
        // Checking if element exists
        System.out.println("Contains 'Banana': " + set.contains("Banana"));
        
        // Removing element
        set.remove("Banana");
        System.out.println("After removal: " + set);
        
        // Iterating
        for (String item : set) {
            System.out.println(item);
        }
        
        // Set operations
        HashSet<Integer> set1 = new HashSet<>(Arrays.asList(1, 2, 3, 4, 5));
        HashSet<Integer> set2 = new HashSet<>(Arrays.asList(4, 5, 6, 7, 8));
        
        // Union
        HashSet<Integer> union = new HashSet<>(set1);
        union.addAll(set2);
        System.out.println("\nUnion: " + union);
        
        // Intersection
        HashSet<Integer> intersection = new HashSet<>(set1);
        intersection.retainAll(set2);
        System.out.println("Intersection: " + intersection);
        
        // Difference
        HashSet<Integer> difference = new HashSet<>(set1);
        difference.removeAll(set2);
        System.out.println("Difference: " + difference);
    }
}
```

### 4.5 TreeSet

```java
import java.util.*;

public class TreeSetExample {
    public static void main(String[] args) {
        // TreeSet maintains sorted order
        TreeSet<Integer> treeSet = new TreeSet<>();
        
        treeSet.add(50);
        treeSet.add(20);
        treeSet.add(80);
        treeSet.add(10);
        treeSet.add(40);
        
        System.out.println("TreeSet (sorted): " + treeSet);
        
        // Navigation methods
        System.out.println("First: " + treeSet.first());
        System.out.println("Last: " + treeSet.last());
        System.out.println("Lower than 50: " + treeSet.lower(50));
        System.out.println("Higher than 50: " + treeSet.higher(50));
        System.out.println("Floor of 45: " + treeSet.floor(45));
        System.out.println("Ceiling of 45: " + treeSet.ceiling(45));
        
        // SubSet operations
        System.out.println("HeadSet (< 50): " + treeSet.headSet(50));
        System.out.println("TailSet (>= 50): " + treeSet.tailSet(50));
        System.out.println("SubSet [20, 80): " + treeSet.subSet(20, 80));
        
        // Descending order
        System.out.println("Descending: " + treeSet.descendingSet());
        
        // Custom comparator
        TreeSet<String> customSet = new TreeSet<>((a, b) -> b.compareTo(a));
        customSet.add("Apple");
        customSet.add("Banana");
        customSet.add("Cherry");
        System.out.println("\nReverse order: " + customSet);
    }
}
```

### 4.6 HashMap

```java
import java.util.*;

public class HashMapExample {
    public static void main(String[] args) {
        HashMap<String, Integer> map = new HashMap<>();
        
        // Adding key-value pairs
        map.put("Alice", 25);
        map.put("Bob", 30);
        map.put("Charlie", 35);
        map.put("Alice", 26);  // Updates existing value
        
        System.out.println("HashMap: " + map);
        
        // Getting values
        System.out.println("Alice's age: " + map.get("Alice"));
        System.out.println("David's age: " + map.get("David"));  // null
        
        // getOrDefault
        System.out.println("David's age: " + map.getOrDefault("David", 0));
        
        // Checking if key/value exists
        System.out.println("Contains key 'Bob': " + map.containsKey("Bob"));
        System.out.println("Contains value 30: " + map.containsValue(30));
        
        // Removing
        map.remove("Charlie");
        System.out.println("After removal: " + map);
        
        // Size
        System.out.println("Size: " + map.size());
        
        // Iterating over keys
        System.out.println("\nIterating over keys:");
        for (String key : map.keySet()) {
            System.out.println(key + ": " + map.get(key));
        }
        
        // Iterating over values
        System.out.println("\nIterating over values:");
        for (Integer value : map.values()) {
            System.out.println(value);
        }
        
        // Iterating over entries
        System.out.println("\nIterating over entries:");
        for (Map.Entry<String, Integer> entry : map.entrySet()) {
            System.out.println(entry.getKey() + ": " + entry.getValue());
        }
        
        // forEach (Java 8+)
        System.out.println("\nUsing forEach:");
        map.forEach((key, value) -> System.out.println(key + ": " + value));
        
        // putIfAbsent
        map.putIfAbsent("Alice", 40);  // Won't update
        map.putIfAbsent("David", 28);  // Will add
        System.out.println("\nAfter putIfAbsent: " + map);
        
        // compute methods
        map.compute("Alice", (key, value) -> value + 1);
        System.out.println("After compute: " + map);
        
        // merge
        map.merge("Bob", 5, (oldValue, newValue) -> oldValue + newValue);
        System.out.println("After merge: " + map);
    }
}
```

### 4.7 TreeMap

```java
import java.util.*;

public class TreeMapExample {
    public static void main(String[] args) {
        // TreeMap maintains sorted order by keys
        TreeMap<String, Integer> treeMap = new TreeMap<>();
        
        treeMap.put("Charlie", 35);
        treeMap.put("Alice", 25);
        treeMap.put("Bob", 30);
        treeMap.put("David", 28);
        
        System.out.println("TreeMap (sorted by keys): " + treeMap);
        
        // Navigation methods
        System.out.println("First key: " + treeMap.firstKey());
        System.out.println("Last key: " + treeMap.lastKey());
        System.out.println("Lower key than 'Charlie': " + treeMap.lowerKey("Charlie"));
        System.out.println("Higher key than 'Bob': " + treeMap.higherKey("Bob"));
        
        // Entry methods
        System.out.println("First entry: " + treeMap.firstEntry());
        System.out.println("Last entry: " + treeMap.lastEntry());
        
        // SubMap operations
        System.out.println("HeadMap (< 'Charlie'): " + treeMap.headMap("Charlie"));
        System.out.println("TailMap (>= 'Bob'): " + treeMap.tailMap("Bob"));
        System.out.println("SubMap ['Alice', 'Charlie'): " + 
                          treeMap.subMap("Alice", "Charlie"));
        
        // Descending map
        System.out.println("Descending: " + treeMap.descendingMap());
        
        // Custom comparator (reverse order)
        TreeMap<String, Integer> reverseMap = 
            new TreeMap<>((a, b) -> b.compareTo(a));
        reverseMap.putAll(treeMap);
        System.out.println("\nReverse order: " + reverseMap);
    }
}
```

### 4.8 Queue - PriorityQueue

```java
import java.util.*;

public class PriorityQueueExample {
    public static void main(String[] args) {
        // Min heap by default (natural ordering)
        PriorityQueue<Integer> pq = new PriorityQueue<>();
        
        pq.offer(30);
        pq.offer(10);
        pq.offer(50);
        pq.offer(20);
        
        System.out.println("PriorityQueue: " + pq);
        System.out.println("Peek (min element): " + pq.peek());
        
        System.out.println("\nPolling elements (in order):");
        while (!pq.isEmpty()) {
            System.out.println(pq.poll());
        }
        
        // Max heap (custom comparator)
        PriorityQueue<Integer> maxHeap = 
            new PriorityQueue<>((a, b) -> b - a);
        
        maxHeap.offer(30);
        maxHeap.offer(10);
        maxHeap.offer(50);
        maxHeap.offer(20);
        
        System.out.println("\nMax Heap - Polling:");
        while (!maxHeap.isEmpty()) {
            System.out.println(maxHeap.poll());
        }
        
        // Custom objects
        PriorityQueue<Task> taskQueue = new PriorityQueue<>();
        taskQueue.offer(new Task("Low priority task", 3));
        taskQueue.offer(new Task("High priority task", 1));
        taskQueue.offer(new Task("Medium priority task", 2));
        
        System.out.println("\nTasks by priority:");
        while (!taskQueue.isEmpty()) {
            System.out.println(taskQueue.poll());
        }
    }
}

class Task implements Comparable<Task> {
    String name;
    int priority;
    
    public Task(String name, int priority) {
        this.name = name;
        this.priority = priority;
    }
    
    @Override
    public int compareTo(Task other) {
        return this.priority - other.priority;
    }
    
    @Override
    public String toString() {
        return name + " (Priority: " + priority + ")";
    }
}
```

### 4.9 Comparable and Comparator

```java
import java.util.*;

// Using Comparable
class Student implements Comparable<Student> {
    String name;
    int age;
    double gpa;
    
    public Student(String name, int age, double gpa) {
        this.name = name;
        this.age = age;
        this.gpa = gpa;
    }
    
    // Natural ordering by name
    @Override
    public int compareTo(Student other) {
        return this.name.compareTo(other.name);
    }
    
    @Override
    public String toString() {
        return String.format("%s (Age: %d, GPA: %.2f)", name, age, gpa);
    }
}

public class ComparableComparatorExample {
    public static void main(String[] args) {
        List<Student> students = new ArrayList<>();
        students.add(new Student("Charlie", 20, 3.5));
        students.add(new Student("Alice", 22, 3.8));
        students.add(new Student("Bob", 21, 3.6));
        
        // Sort using Comparable (natural ordering)
        Collections.sort(students);
        System.out.println("Sorted by name (Comparable):");
        students.forEach(System.out::println);
        
        // Sort using Comparator - by age
        Comparator<Student> ageComparator = new Comparator<Student>() {
            @Override
            public int compare(Student s1, Student s2) {
                return s1.age - s2.age;
            }
        };
        Collections.sort(students, ageComparator);
        System.out.println("\nSorted by age (Comparator):");
        students.forEach(System.out::println);
        
        // Lambda expression (Java 8+)
        Collections.sort(students, (s1, s2) -> Double.compare(s1.gpa, s2.gpa));
        System.out.println("\nSorted by GPA (Lambda):");
        students.forEach(System.out::println);
        
        // Comparator.comparing (Java 8+)
        students.sort(Comparator.comparing(s -> s.name));
        System.out.println("\nSorted by name (Comparator.comparing):");
        students.forEach(System.out::println);
        
        // Multiple field sorting
        students.sort(Comparator
            .comparing((Student s) -> s.age)
            .thenComparing(s -> s.name));
        System.out.println("\nSorted by age, then name:");
        students.forEach(System.out::println);
        
        // Reverse order
        students.sort(Comparator.comparing((Student s) -> s.gpa).reversed());
        System.out.println("\nSorted by GPA (descending):");
        students.forEach(System.out::println);
    }
}
```

### 4.10 Collections Utility Class

```java
import java.util.*;

public class CollectionsUtility {
    public static void main(String[] args) {
        List<Integer> list = new ArrayList<>(Arrays.asList(5, 2, 8, 1, 9, 3));
        
        System.out.println("Original: " + list);
        
        // Sorting
        Collections.sort(list);
        System.out.println("Sorted: " + list);
        
        // Reverse
        Collections.reverse(list);
        System.out.println("Reversed: " + list);
        
        // Shuffle
        Collections.shuffle(list);
        System.out.println("Shuffled: " + list);
        
        // Min and Max
        System.out.println("Min: " + Collections.min(list));
        System.out.println("Max: " + Collections.max(list));
        
        // Binary search (list must be sorted)
        Collections.sort(list);
        int index = Collections.binarySearch(list, 5);
        System.out.println("Index of 5: " + index);
        
        // Frequency
        List<String> words = Arrays.asList("apple", "banana", "apple", "cherry", "apple");
        System.out.println("Frequency of 'apple': " + 
                          Collections.frequency(words, "apple"));
        
        // Fill
        Collections.fill(list, 0);
        System.out.println("After fill: " + list);
        
        // Copy
        List<Integer> dest = new ArrayList<>(Arrays.asList(0, 0, 0, 0, 0, 0));
        List<Integer> src = Arrays.asList(1, 2, 3, 4, 5, 6);
        Collections.copy(dest, src);
        System.out.println("After copy: " + dest);
        
        // Rotate
        List<Integer> nums = new ArrayList<>(Arrays.asList(1, 2, 3, 4, 5));
        Collections.rotate(nums, 2);
        System.out.println("After rotate by 2: " + nums);
        
        // Swap
        Collections.swap(nums, 0, 4);
        System.out.println("After swap: " + nums);
        
        // Unmodifiable collections
        List<String> modifiable = new ArrayList<>(Arrays.asList("A", "B", "C"));
        List<String> unmodifiable = Collections.unmodifiableList(modifiable);
        // unmodifiable.add("D");  // UnsupportedOperationException
        
        // Synchronized collections (thread-safe)
        List<String> syncList = Collections.synchronizedList(new ArrayList<>());
        
        // Empty collections
        List<String> emptyList = Collections.emptyList();
        Set<String> emptySet = Collections.emptySet();
        Map<String, String> emptyMap = Collections.emptyMap();
        
        // Singleton collections
        Set<String> singletonSet = Collections.singleton("OnlyElement");
    }
}
```

---

## 5. Multithreading and Concurrency

### 5.1 Creating Threads

```java
// Method 1: Extending Thread class
class MyThread extends Thread {
    @Override
    public void run() {
        for (int i = 1; i <= 5; i++) {
            System.out.println(Thread.currentThread().getName() + ": " + i);
            try {
                Thread.sleep(500);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}

// Method 2: Implementing Runnable interface
class MyRunnable implements Runnable {
    @Override
    public void run() {
        for (int i = 1; i <= 5; i++) {
            System.out.println(Thread.currentThread().getName() + ": " + i);
            try {
                Thread.sleep(500);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}

public class ThreadCreation {
    public static void main(String[] args) {
        // Using Thread class
        MyThread thread1 = new MyThread();
        thread1.setName("Thread-1");
        thread1.start();
        
        // Using Runnable interface
        Thread thread2 = new Thread(new MyRunnable());
        thread2.setName("Thread-2");
        thread2.start();
        
        // Using lambda expression (Java 8+)
        Thread thread3 = new Thread(() -> {
            for (int i = 1; i <= 5; i++) {
                System.out.println(Thread.currentThread().getName() + ": " + i);
                try {
                    Thread.sleep(500);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        });
        thread3.setName("Thread-3");
        thread3.start();
        
        System.out.println("Main thread: " + Thread.currentThread().getName());
    }
}
```

### 5.2 Thread Lifecycle and Methods

```java
public class ThreadLifecycle {
    public static void main(String[] args) throws InterruptedException {
        Thread thread = new Thread(() -> {
            System.out.println("Thread started");
            
            try {
                Thread.sleep(2000);
            } catch (InterruptedException e) {
                System.out.println("Thread interrupted");
            }
            
            System.out.println("Thread finishing");
        });
        
        // Thread states
        System.out.println("State (NEW): " + thread.getState());
        
        thread.start();
        System.out.println("State (RUNNABLE): " + thread.getState());
        
        Thread.sleep(100);
        System.out.println("State (TIMED_WAITING): " + thread.getState());
        
        thread.join();  // Wait for thread to complete
        System.out.println("State (TERMINATED): " + thread.getState());
        
        // Thread information
        Thread current = Thread.currentThread();
        System.out.println("\nCurrent thread info:");
        System.out.println("Name: " + current.getName());
        System.out.println("ID: " + current.getId());
        System.out.println("Priority: " + current.getPriority());
        System.out.println("Is alive: " + current.isAlive());
        System.out.println("Is daemon: " + current.isDaemon());
        
        // Priority example
        Thread highPriority = new Thread(() -> {
            System.out.println("High priority thread");
        });
        highPriority.setPriority(Thread.MAX_PRIORITY);
        
        Thread lowPriority = new Thread(() -> {
            System.out.println("Low priority thread");
        });
        lowPriority.setPriority(Thread.MIN_PRIORITY);
        
        // Daemon thread
        Thread daemon = new Thread(() -> {
            while (true) {
                System.out.println("Daemon running...");
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    break;
                }
            }
        });
        daemon.setDaemon(true);
        daemon.start();
        
        Thread.sleep(3000);
        System.out.println("Main thread ending (daemon will stop)");
    }
}
```

### 5.3 Synchronization

```java
// Problem: Race condition without synchronization
class Counter {
    private int count = 0;
    
    // Unsynchronized method
    public void incrementUnsafe() {
        count++;
    }
    
    // Synchronized method
    public synchronized void incrementSafe() {
        count++;
    }
    
    // Synchronized block
    public void incrementBlock() {
        synchronized (this) {
            count++;
        }
    }
    
    public int getCount() {
        return count;
    }
}

public class SynchronizationExample {
    public static void main(String[] args) throws InterruptedException {
        Counter counter = new Counter();
        
        // Creating multiple threads
        Thread[] threads = new Thread[1000];
        
        for (int i = 0; i < threads.length; i++) {
            threads[i] = new Thread(() -> {
                for (int j = 0; j < 1000; j++) {
                    counter.incrementSafe();
                }
            });
            threads[i].start();
        }
        
        // Wait for all threads to complete
        for (Thread thread : threads) {
            thread.join();
        }
        
        System.out.println("Final count: " + counter.getCount());
        // Should be 1,000,000 with synchronization
    }
}

// Static synchronization
class StaticCounter {
    private static int count = 0;
    
    // Synchronized on class object
    public static synchronized void increment() {
        count++;
    }
    
    public static int getCount() {
        return count;
    }
}

// Deadlock example
class Resource {
    private final String name;
    
    public Resource(String name) {
        this.name = name;
    }
    
    public String getName() {
        return name;
    }
}

public class DeadlockExample {
    public static void main(String[] args) {
        Resource resource1 = new Resource("Resource 1");
        Resource resource2 = new Resource("Resource 2");
        
        Thread thread1 = new Thread(() -> {
            synchronized (resource1) {
                System.out.println("Thread 1: Locked " + resource1.getName());
                
                try {
                    Thread.sleep(100);
                } catch (InterruptedException e) {}
                
                System.out.println("Thread 1: Waiting for " + resource2.getName());
                synchronized (resource2) {
                    System.out.println("Thread 1: Locked " + resource2.getName());
                }
            }
        });
        
        Thread thread2 = new Thread(() -> {
            synchronized (resource2) {
                System.out.println("Thread 2: Locked " + resource2.getName());
                
                try {
                    Thread.sleep(100);
                } catch (InterruptedException e) {}
                
                System.out.println("Thread 2: Waiting for " + resource1.getName());
                synchronized (resource1) {
                    System.out.println("Thread 2: Locked " + resource1.getName());
                }
            }
        });
        
        thread1.start();
        thread2.start();
        // This will cause a deadlock
    }
}
```

### 5.4 Volatile Keyword

```java
public class VolatileExample {
    // Without volatile, thread might cache the value
    private static volatile boolean running = true;
    
    public static void main(String[] args) throws InterruptedException {
        Thread thread = new Thread(() -> {
            int count = 0;
            while (running) {
                count++;
            }
            System.out.println("Thread stopped. Count: " + count);
        });
        
        thread.start();
        
        Thread.sleep(1000);
        System.out.println("Stopping thread...");
        running = false;  // volatile ensures visibility
        
        thread.join();
        System.out.println("Main thread finished");
    }
}

// Volatile vs Synchronized
class VolatileVsSynchronized {
    private volatile int volatileCount = 0;
    private int syncCount = 0;
    
    // Volatile: visibility guarantee, but not atomic
    public void incrementVolatile() {
        volatileCount++;  // NOT thread-safe (read-modify-write)
    }
    
    // Synchronized: both visibility and atomicity
    public synchronized void incrementSync() {
        syncCount++;  // Thread-safe
    }
    
    // Better: use AtomicInteger for atomic operations
    private java.util.concurrent.atomic.AtomicInteger atomicCount = 
        new java.util.concurrent.atomic.AtomicInteger(0);
    
    public void incrementAtomic() {
        atomicCount.incrementAndGet();  // Thread-safe and lock-free
    }
}
```

### 5.5 Executor Framework

```java
import java.util.concurrent.*;

public class ExecutorFrameworkExample {
    public static void main(String[] args) throws InterruptedException, ExecutionException {
        
        // Single thread executor
        ExecutorService singleExecutor = Executors.newSingleThreadExecutor();
        singleExecutor.execute(() -> System.out.println("Single thread task"));
        singleExecutor.shutdown();
        
        // Fixed thread pool
        ExecutorService fixedPool = Executors.newFixedThreadPool(3);
        
        for (int i = 1; i <= 10; i++) {
            final int taskId = i;
            fixedPool.execute(() -> {
                System.out.println("Task " + taskId + " executed by " + 
                                 Thread.currentThread().getName());
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            });
        }
        
        fixedPool.shutdown();
        fixedPool.awaitTermination(1, TimeUnit.MINUTES);
        
        // Cached thread pool (creates threads as needed)
        ExecutorService cachedPool = Executors.newCachedThreadPool();
        cachedPool.execute(() -> System.out.println("Cached pool task"));
        cachedPool.shutdown();
        
        // Scheduled executor
        ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(2);
        
        // Schedule with delay
        scheduler.schedule(() -> {
            System.out.println("Task executed after 2 seconds");
        }, 2, TimeUnit.SECONDS);
        
        // Schedule at fixed rate
        scheduler.scheduleAtFixedRate(() -> {
            System.out.println("Repeating task: " + System.currentTimeMillis());
        }, 0, 1, TimeUnit.SECONDS);
        
        Thread.sleep(5000);
        scheduler.shutdown();
        
        System.out.println("All executors shut down");
    }
}
```

### 5.6 Callable and Future

```java
import java.util.concurrent.*;
import java.util.*;

public class CallableFutureExample {
    public static void main(String[] args) throws InterruptedException, ExecutionException {
        
        // Callable - can return a value and throw checked exceptions
        Callable<Integer> task = () -> {
            System.out.println("Task started");
            Thread.sleep(2000);
            return 42;
        };
        
        ExecutorService executor = Executors.newSingleThreadExecutor();
        
        // Submit callable and get Future
        Future<Integer> future = executor.submit(task);
        
        System.out.println("Task submitted");
        
        // Do other work while task executes
        System.out.println("Doing other work...");
        
        // Check if task is done
        if (!future.isDone()) {
            System.out.println("Task is still running");
        }
        
        // Get result (blocks until ready)
        Integer result = future.get();
        System.out.println("Task result: " + result);
        
        // Timeout example
        Callable<String> longTask = () -> {
            Thread.sleep(5000);
            return "Done";
        };
        
        Future<String> longFuture = executor.submit(longTask);
        
        try {
            // Wait maximum 2 seconds
            String longResult = longFuture.get(2, TimeUnit.SECONDS);
            System.out.println(longResult);
        } catch (TimeoutException e) {
            System.out.println("Task timed out");
            longFuture.cancel(true);  // Cancel the task
        }
        
        // Multiple callables
        List<Callable<Integer>> tasks = new ArrayList<>();
        for (int i = 1; i <= 5; i++) {
            final int taskId = i;
            tasks.add(() -> {
                Thread.sleep(1000);
                return taskId * 10;
            });
        }
        
        ExecutorService multiExecutor = Executors.newFixedThreadPool(3);
        
        // invokeAll - executes all tasks
        List<Future<Integer>> futures = multiExecutor.invokeAll(tasks);
        
        System.out.println("\nResults from all tasks:");
        for (Future<Integer> f : futures) {
            System.out.println(f.get());
        }
        
        // invokeAny - returns result of first completed task
        Integer anyResult = multiExecutor.invokeAny(tasks);
        System.out.println("\nFirst completed result: " + anyResult);
        
        executor.shutdown();
        multiExecutor.shutdown();
    }
}
```

### 5.7 Concurrent Collections

```java
import java.util.concurrent.*;

public class ConcurrentCollectionsExample {
    public static void main(String[] args) throws InterruptedException {
        
        // ConcurrentHashMap - thread-safe HashMap
        ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();
        
        map.put("A", 1);
        map.put("B", 2);
        map.putIfAbsent("A", 10);  // Won't update
        
        System.out.println("Map: " + map);
        
        // Atomic operations
        map.compute("A", (key, value) -> value + 10);
        map.merge("C", 3, (oldVal, newVal) -> oldVal + newVal);
        
        System.out.println("After operations: " + map);
        
        // CopyOnWriteArrayList - thread-safe for read-heavy scenarios
        CopyOnWriteArrayList<String> list = new CopyOnWriteArrayList<>();
        list.add("Item1");
        list.add("Item2");
        list.add("Item3");
        
        // Iterator is immutable snapshot
        for (String item : list) {
            System.out.println(item);
            list.add("Item4");  // Safe during iteration
        }
        
        // BlockingQueue - thread-safe queue with blocking operations
        BlockingQueue<String> queue = new ArrayBlockingQueue<>(10);
        
        // Producer thread
        Thread producer = new Thread(() -> {
            try {
                for (int i = 1; i <= 5; i++) {
                    String item = "Item-" + i;
                    queue.put(item);  // Blocks if queue is full
                    System.out.println("Produced: " + item);
                    Thread.sleep(500);
                }
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        });
        
        // Consumer thread
        Thread consumer = new Thread(() -> {
            try {
                for (int i = 1; i <= 5; i++) {
                    String item = queue.take();  // Blocks if queue is empty
                    System.out.println("Consumed: " + item);
                    Thread.sleep(1000);
                }
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        });
        
        producer.start();
        consumer.start();
        
        producer.join();
        consumer.join();
        
        // ConcurrentLinkedQueue - non-blocking thread-safe queue
        ConcurrentLinkedQueue<String> concurrentQueue = new ConcurrentLinkedQueue<>();
        concurrentQueue.offer("A");
        concurrentQueue.offer("B");
        System.out.println("\nPolling from concurrent queue: " + concurrentQueue.poll());
    }
}
```

### 5.8 CountDownLatch, CyclicBarrier, and Semaphore

```java
import java.util.concurrent.*;

public class ConcurrencyUtilities {
    
    // CountDownLatch - wait for multiple threads to complete
    public static void countDownLatchExample() throws InterruptedException {
        int threadCount = 3;
        CountDownLatch latch = new CountDownLatch(threadCount);
        
        for (int i = 1; i <= threadCount; i++) {
            final int workerId = i;
            new Thread(() -> {
                System.out.println("Worker " + workerId + " is working");
                try {
                    Thread.sleep(2000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println("Worker " + workerId + " finished");
                latch.countDown();  // Decrement count
            }).start();
        }
        
        System.out.println("Main thread waiting for workers...");
        latch.await();  // Wait until count reaches 0
        System.out.println("All workers finished. Main thread continuing.");
    }
    
    // CyclicBarrier - synchronize threads at a common point
    public static void cyclicBarrierExample() {
        int parties = 3;
        CyclicBarrier barrier = new CyclicBarrier(parties, () -> {
            System.out.println("All threads reached barrier. Proceeding...");
        });
        
        for (int i = 1; i <= parties; i++) {
            final int threadId = i;
            new Thread(() -> {
                try {
                    System.out.println("Thread " + threadId + " doing work");
                    Thread.sleep(1000 * threadId);
                    System.out.println("Thread " + threadId + " reached barrier");
                    barrier.await();  // Wait for all threads
                    System.out.println("Thread " + threadId + " proceeding");
                } catch (InterruptedException | BrokenBarrierException e) {
                    e.printStackTrace();
                }
            }).start();
        }
    }
    
    // Semaphore - limit concurrent access to a resource
    public static void semaphoreExample() throws InterruptedException {
        Semaphore semaphore = new Semaphore(2);  // Allow 2 concurrent threads
        
        for (int i = 1; i <= 5; i++) {
            final int threadId = i;
            new Thread(() -> {
                try {
                    System.out.println("Thread " + threadId + " waiting for permit");
                    semaphore.acquire();  // Acquire permit
                    System.out.println("Thread " + threadId + " got permit");
                    
                    Thread.sleep(2000);  // Simulate work
                    
                    System.out.println("Thread " + threadId + " releasing permit");
                    semaphore.release();  // Release permit
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }).start();
        }
        
        Thread.sleep(12000);
    }
    
    public static void main(String[] args) throws InterruptedException {
        System.out.println("=== CountDownLatch Example ===");
        countDownLatchExample();
        
        Thread.sleep(1000);
        
        System.out.println("\n=== CyclicBarrier Example ===");
        cyclicBarrierExample();
        
        Thread.sleep(5000);
        
        System.out.println("\n=== Semaphore Example ===");
        semaphoreExample();
    }
}
```

---

## 6. Java 8+ Features

### 6.1 Lambda Expressions

```java
import java.util.*;
import java.util.function.*;

public class LambdaExpressions {
    public static void main(String[] args) {
        
        // Traditional anonymous class
        Runnable runnable1 = new Runnable() {
            @Override
            public void run() {
                System.out.println("Traditional way");
            }
        };
        
        // Lambda expression
        Runnable runnable2 = () -> System.out.println("Lambda way");
        
        runnable2.run();
        
        // Lambda with parameters
        Comparator<Integer> comparator = (a, b) -> a.compareTo(b);
        System.out.println("Compare 10 and 20: " + comparator.compare(10, 20));
        
        // Lambda with multiple statements
        Calculator add = (a, b) -> {
            int result = a + b;
            System.out.println("Adding " + a + " and " + b);
            return result;
        };
        System.out.println("Result: " + add.calculate(5, 3));
        
        // Lambda with collections
        List<String> names = Arrays.asList("Alice", "Bob", "Charlie", "David");
        
        // forEach
        names.forEach(name -> System.out.println("Hello, " + name));
        
        // removeIf
        List<Integer> numbers = new ArrayList<>(Arrays.asList(1, 2, 3, 4, 5, 6, 7, 8, 9, 10));
        numbers.removeIf(n -> n % 2 == 0);  // Remove even numbers
        System.out.println("Odd numbers: " + numbers);
        
        // sort
        List<String> fruits = new ArrayList<>(Arrays.asList("Banana", "Apple", "Cherry"));
        fruits.sort((s1, s2) -> s1.compareTo(s2));
        System.out.println("Sorted: " + fruits);
        
        // Predefined functional interfaces
        // Predicate<T> - takes T, returns boolean
        Predicate<Integer> isEven = n -> n % 2 == 0;
        System.out.println("Is 4 even? " + isEven.test(4));
        
        // Function<T, R> - takes T, returns R
        Function<String, Integer> stringLength = s -> s.length();
        System.out.println("Length of 'Hello': " + stringLength.apply("Hello"));
        
        // Consumer<T> - takes T, returns nothing
        Consumer<String> printer = s -> System.out.println("Printing: " + s);
        printer.accept("Test");
        
        // Supplier<T> - takes nothing, returns T
        Supplier<Double> randomSupplier = () -> Math.random();
        System.out.println("Random: " + randomSupplier.get());
        
        // BiFunction<T, U, R> - takes T and U, returns R
        BiFunction<Integer, Integer, Integer> multiply = (a, b) -> a * b;
        System.out.println("5 * 3 = " + multiply.apply(5, 3));
    }
}

@FunctionalInterface
interface Calculator {
    int calculate(int a, int b);
}
```

### 6.2 Functional Interfaces

```java
import java.util.function.*;

// Custom functional interface
@FunctionalInterface
interface StringProcessor {
    String process(String input);
    
    // Default methods allowed
    default String processWithPrefix(String input) {
        return "Prefix-" + process(input);
    }
    
    // Static methods allowed
    static String processUpperCase(String input) {
        return input.toUpperCase();
    }
}

public class FunctionalInterfaceExample {
    public static void main(String[] args) {
        
        // Using custom functional interface
        ```
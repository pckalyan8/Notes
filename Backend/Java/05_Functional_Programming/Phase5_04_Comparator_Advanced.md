# ⚡ Phase 5.4 — Advanced Comparator in Java
### Ordering, Chaining, and Null Safety Made Clean

---

## 🧠 The Foundation: `Comparable` vs `Comparator`

Before diving into the advanced Comparator API, it's essential to understand the two
approaches Java offers for defining ordering — because they solve *different* problems.

`Comparable<T>` defines the **natural ordering** of a class. When a class implements
`Comparable`, it bakes a single, default sort order directly into the class definition.
Think of it as saying: "This is how my objects are ordered by nature."

`Comparator<T>` defines an **external, custom ordering**. A Comparator is a separate
object that knows how to compare two objects of some type. Think of it as saying:
"For this specific purpose, I want to order these objects *this* way."

```java
// Comparable: natural order built into the class
public class Student implements Comparable<Student> {
    private String name;
    private int grade;

    @Override
    public int compareTo(Student other) {
        // Natural order: alphabetical by name
        return this.name.compareTo(other.name);
    }
}

// This works because Student "knows" its own natural order
List<Student> students = new ArrayList<>(List.of(
    new Student("Charlie", 88),
    new Student("Alice", 95),
    new Student("Bob", 72)
));
Collections.sort(students); // uses compareTo — sorts alphabetically by name
```

The limitation of `Comparable` is that you can only define *one* natural order.
What if sometimes you want to sort students by name, and other times by grade, and
other times by grade descending? That's where `Comparator` shines — you create as
many ordering strategies as you need, without touching the class itself.

The `compareTo()` (and `compare()` in Comparator) contract says: return a **negative
number** if the first argument comes before the second, **zero** if they are equal,
and a **positive number** if the first comes after the second.

---

## 🏗️ The Old Way vs. The Modern Way

Seeing the evolution helps you appreciate why the modern API exists.

```java
List<String> names = List.of("Charlie", "alice", "Bob", "diana");

// Old way — anonymous class (verbose, hard to chain)
Collections.sort(new ArrayList<>(names), new Comparator<String>() {
    @Override
    public int compare(String a, String b) {
        return a.compareToIgnoreCase(b);
    }
});

// Java 8+ — lambda (already better)
names.sort((a, b) -> a.compareToIgnoreCase(b));

// Java 8+ — method reference (even cleaner)
names.sort(String::compareToIgnoreCase);

// Java 8+ — Comparator factory method (most expressive)
names.sort(Comparator.comparing(String::toLowerCase));
```

---

## 🔧 `Comparator.comparing()` — The Entry Point

`Comparator.comparing()` is the primary factory method. It takes a **key extractor
function** — a function that extracts the value you want to compare by — and builds a
`Comparator` that compares objects by that extracted value.

```java
record Person(String firstName, String lastName, int age, double salary) {}

List<Person> people = List.of(
    new Person("Charlie", "Brown",  30, 75000),
    new Person("Alice",   "Smith",  25, 95000),
    new Person("Bob",     "Jones",  30, 65000),
    new Person("Diana",   "Brown",  25, 85000)
);

// Compare by last name (String's natural ordering)
Comparator<Person> byLastName = Comparator.comparing(Person::lastName);
people.stream().sorted(byLastName).forEach(System.out::println);
// Brown (Charlie), Brown (Diana), Jones (Bob), Smith (Alice)

// Compare by age (int — use comparingInt for efficiency, no boxing)
Comparator<Person> byAge = Comparator.comparingInt(Person::age);

// Compare by salary (double — use comparingDouble)
Comparator<Person> bySalary = Comparator.comparingDouble(Person::salary);
```

`comparingInt()`, `comparingLong()`, and `comparingDouble()` exist specifically for
primitive key types to avoid boxing overhead. Always prefer them over `comparing()`
when the sort key is a primitive.

---

## 🔗 Chaining with `thenComparing()` — Multi-Level Sort

This is one of the most powerful features of the modern Comparator API. When the
primary comparator considers two elements equal, `thenComparing()` provides a
tiebreaker. You can chain as many levels as you need.

```java
// Sort by age first, then by last name for people of the same age,
// then by first name for people with the same age and last name.
Comparator<Person> multiLevel = Comparator
    .comparingInt(Person::age)
    .thenComparing(Person::lastName)
    .thenComparing(Person::firstName);

List<Person> sorted = people.stream()
    .sorted(multiLevel)
    .collect(Collectors.toList());
// Output:
// Alice Smith (25)
// Diana Brown (25)  — same age as Alice, B < S alphabetically
// Bob Jones (30)
// Charlie Brown (30) — same age as Bob, B < J alphabetically
```

You can even chain with a custom comparator at any level:

```java
// Sort by age, then by salary descending (highest earner among same-age people first)
Comparator<Person> byAgeThenSalaryDesc = Comparator
    .comparingInt(Person::age)
    .thenComparingDouble(Person::salary).reversed();
// ⚠️ Careful: reversed() reverses the ENTIRE comparator, not just the last step.
// See the section below on reversing correctly.
```

---

## ↔️ `reversed()` — Reversing Order Correctly

`reversed()` flips the entire comparator. When you want to reverse only one
level of a chain, you need to reverse *that comparator* before chaining, not after.

```java
// ❌ Wrong: reversed() applies to the whole chain
Comparator<Person> wrong = Comparator
    .comparingInt(Person::age)          // ascending age
    .thenComparingDouble(Person::salary) // ascending salary
    .reversed();                         // NOW: reversed BOTH! descending age AND descending salary

// ✅ Correct: reverse only the salary part
Comparator<Person> correct = Comparator
    .comparingInt(Person::age)
    .thenComparing(
        Comparator.comparingDouble(Person::salary).reversed() // only salary is reversed
    );
// Result: ascending age, then descending salary within each age group
```

---

## 🔃 `naturalOrder()` and `reverseOrder()`

`Comparator.naturalOrder()` returns a comparator that uses the class's `Comparable`
implementation (its natural order). `Comparator.reverseOrder()` returns the inverse.

```java
List<Integer> numbers = List.of(3, 1, 4, 1, 5, 9, 2, 6);

// Natural order (ascending for integers)
numbers.stream()
       .sorted(Comparator.naturalOrder())
       .forEach(System.out::println); // 1 1 2 3 4 5 6 9

// Reverse of natural order (descending)
numbers.stream()
       .sorted(Comparator.reverseOrder())
       .forEach(System.out::println); // 9 6 5 4 3 2 1 1
```

These are most useful when you need to pass a comparator to a method but want the
standard ordering — for example, when building a multi-level sort where one level
happens to be a String that should sort naturally.

---

## 🛡️ `nullsFirst()` and `nullsLast()` — Handling Null Safely

In the real world, your data will have nulls. By default, a comparator that encounters
a null value will throw `NullPointerException`. The `nullsFirst()` and `nullsLast()`
wrappers handle this gracefully by defining where null elements should appear in the
sorted output.

```java
List<String> withNulls = Arrays.asList("banana", null, "apple", null, "cherry");

// Without null handling — throws NullPointerException!
// withNulls.sort(String::compareTo); // ❌

// Nulls go to the front
withNulls.sort(Comparator.nullsFirst(String::compareTo));
// [null, null, apple, banana, cherry]

// Nulls go to the back
withNulls.sort(Comparator.nullsLast(String::compareTo));
// [apple, banana, cherry, null, null]
```

This works seamlessly with `comparing()` chains too:

```java
// Sort people by last name, nulls last, then by first name, nulls first
Comparator<Person> nullSafe = Comparator
    .comparing(Person::lastName, Comparator.nullsLast(String::compareTo))
    .thenComparing(Person::firstName, Comparator.nullsFirst(String::compareTo));
```

---

## 🎯 `Comparator` with Streams

Comparators are deeply integrated into the Streams API. Here's a tour of where you'll
use them most.

### `sorted()` — In-stream sorting

```java
List<Person> sortedByAge = people.stream()
    .sorted(Comparator.comparingInt(Person::age))
    .collect(Collectors.toList());
```

### `min()` and `max()` — Finding Extremes

```java
// Youngest person
Optional<Person> youngest = people.stream()
    .min(Comparator.comparingInt(Person::age));

// Highest earner among people aged 30+
Optional<Person> highestEarnerOver30 = people.stream()
    .filter(p -> p.age() >= 30)
    .max(Comparator.comparingDouble(Person::salary));
```

### `Map.Entry.comparingByKey()` and `comparingByValue()`

When working with Map entries in a stream, these provide ready-made comparators.

```java
Map<String, Integer> scores = Map.of("Alice", 95, "Bob", 72, "Charlie", 88);

// Sort map entries by value (score), descending
scores.entrySet().stream()
      .sorted(Map.Entry.<String, Integer>comparingByValue().reversed())
      .forEach(e -> System.out.println(e.getKey() + ": " + e.getValue()));
// Alice: 95, Charlie: 88, Bob: 72
```

### `BinaryOperator.maxBy()` and `minBy()`

These are specifically for use with `reduce()`, giving you a clean way to find the
extreme element using a comparator.

```java
Optional<Person> mostExperienced = people.stream()
    .reduce(BinaryOperator.maxBy(Comparator.comparingInt(Person::age)));
```

---

## 🔑 Practical Patterns Cheat Sheet

```java
// Pattern 1: Sort by single field (ascending)
list.sort(Comparator.comparing(MyClass::getField));

// Pattern 2: Sort by single field (descending)
list.sort(Comparator.comparing(MyClass::getField).reversed());

// Pattern 3: Multi-field sort (ascending primary, ascending secondary)
list.sort(Comparator.comparing(MyClass::getField1)
                    .thenComparing(MyClass::getField2));

// Pattern 4: Multi-field sort with mixed directions
// ascending by field1, then DESCENDING by field2
list.sort(Comparator.comparing(MyClass::getField1)
                    .thenComparing(Comparator.comparing(MyClass::getField2).reversed()));

// Pattern 5: Numeric field (use comparingInt/Long/Double — avoids boxing)
list.sort(Comparator.comparingInt(MyClass::getIntField));

// Pattern 6: Case-insensitive string sort
list.sort(Comparator.comparing(MyClass::getName, String.CASE_INSENSITIVE_ORDER));

// Pattern 7: Null-safe sort (nulls at end)
list.sort(Comparator.comparing(MyClass::getNullableField, Comparator.nullsLast(Comparator.naturalOrder())));

// Pattern 8: Assign and reuse comparator
Comparator<MyClass> comp = Comparator.comparing(MyClass::getField1)
                                     .thenComparingInt(MyClass::getIntField);
list.sort(comp);
PriorityQueue<MyClass> pq = new PriorityQueue<>(comp);
```

---

## ⚙️ Custom Comparator Logic

Sometimes the comparison logic doesn't map to a simple key extraction. In that case,
write the comparison logic explicitly using a lambda.

```java
// Sort strings by their numeric suffix (e.g., "item10" before "item2" logically)
List<String> items = List.of("item2", "item10", "item1", "item20", "item3");

Comparator<String> numericSuffix = Comparator.comparingInt(
    s -> Integer.parseInt(s.replaceAll("[^0-9]", ""))
);
items.sort(numericSuffix);
// [item1, item2, item3, item10, item20]
// (Default string sort would give: item1, item10, item2, item20, item3 — wrong!)


// Sort objects by a computed value (e.g., distance from a point)
record Point(int x, int y) {
    double distanceTo(Point other) {
        int dx = this.x - other.x;
        int dy = this.y - other.y;
        return Math.sqrt(dx * dx + dy * dy);
    }
}
Point origin = new Point(0, 0);
List<Point> points = List.of(new Point(3, 4), new Point(1, 1), new Point(5, 0), new Point(2, 2));

List<Point> byDistance = points.stream()
    .sorted(Comparator.comparingDouble(p -> p.distanceTo(origin)))
    .collect(Collectors.toList());
// sorted by distance from origin: (1,1), (2,2), (3,4)/(5,0), then the farther ones
```

---

## ⚠️ Best Practices & Common Pitfalls

**Always use `comparingInt/Long/Double` for primitive key types.** The base
`comparing()` method works with any `Comparable`, but for `int`, `long`, and `double`
fields it requires boxing to `Integer`, `Long`, `Double`. The specialized versions
avoid that overhead and are always preferred.

**Be careful with `reversed()` placement in a chain.** It reverses everything
accumulated so far. If you want only the last level reversed, wrap that one comparator
in `.reversed()` before passing it to `thenComparing()`.

**Don't subtract integers for comparison — ever.** A classic trap is writing
`(a, b) -> a - b` as a numeric comparator. This seems to work but breaks with large
positive and negative numbers due to integer overflow. Always use
`Integer.compare(a, b)` or `Comparator.comparingInt(...)` instead.

```java
// ❌ Dangerous — overflow for large values like (Integer.MAX_VALUE - (-1))
Comparator<Integer> wrong = (a, b) -> a - b;

// ✅ Safe always
Comparator<Integer> correct = Integer::compare;
// Or:
Comparator<Integer> alsoCorrect = Comparator.comparingInt(Integer::intValue);
```

**Assign reusable comparators to named variables.** If you find yourself writing
the same comparator in multiple places, extract it to a named `static final` field
or a method. This also makes your intent clear and the code easier to test.

**Comparators are `Serializable` if the key extractors are.** If you need to
serialize a comparator (e.g., for distributed computing), ensure the functions you
pass are method references (which are serializable) rather than anonymous lambdas
(which are not guaranteed to be serializable).

---

## 🧪 Complete Working Example

```java
import java.util.*;
import java.util.stream.*;

public class ComparatorDemo {

    record Employee(String name, String department, int yearsOfExperience, double salary) {}

    public static void main(String[] args) {
        List<Employee> employees = new ArrayList<>(List.of(
            new Employee("Alice",   "Engineering", 8,  95000),
            new Employee("Bob",     "Marketing",   3,  72000),
            new Employee("Charlie", "Engineering", 5,  88000),
            new Employee("Diana",   "Engineering", 8,  102000),
            new Employee("Eve",     "Marketing",   8,  78000),
            new Employee("Frank",   "HR",          2,  65000)
        ));

        // 1. Sort by department (alphabetical), then by experience descending,
        //    then by name alphabetical as final tiebreaker
        Comparator<Employee> primary = Comparator
            .comparing(Employee::department)
            .thenComparing(Comparator.comparingInt(Employee::yearsOfExperience).reversed())
            .thenComparing(Employee::name);

        employees.sort(primary);
        System.out.println("=== By dept → exp desc → name ===");
        employees.forEach(e ->
            System.out.printf("  %-10s %-12s %d yrs%n", e.name(), e.department(), e.yearsOfExperience()));

        // 2. Find the highest-earning employee in Engineering
        Optional<Employee> topEngineer = employees.stream()
            .filter(e -> e.department().equals("Engineering"))
            .max(Comparator.comparingDouble(Employee::salary));
        topEngineer.ifPresent(e ->
            System.out.printf("Top engineer: %s ($%.0f)%n", e.name(), e.salary()));

        // 3. Find the most experienced employee overall (with tiebreak by salary)
        Optional<Employee> mostSenior = employees.stream()
            .max(Comparator.comparingInt(Employee::yearsOfExperience)
                           .thenComparingDouble(Employee::salary));
        mostSenior.ifPresent(e ->
            System.out.printf("Most senior: %s (%d yrs, $%.0f)%n",
                e.name(), e.yearsOfExperience(), e.salary()));

        // 4. Sort department names by the average salary of their members (highest first)
        employees.stream()
            .collect(Collectors.groupingBy(
                Employee::department,
                Collectors.averagingDouble(Employee::salary)
            ))
            .entrySet().stream()
            .sorted(Map.Entry.<String, Double>comparingByValue().reversed())
            .forEach(e ->
                System.out.printf("  %s avg salary: $%.0f%n", e.getKey(), e.getValue()));
    }
}
```

---

## 🔑 Key Takeaways

- `Comparable` defines **one natural order** built into the class. `Comparator` defines **any number of external orderings** without changing the class.
- `Comparator.comparing(keyExtractor)` is the modern entry point — prefer it over manual lambdas with comparison logic.
- Use `comparingInt`, `comparingLong`, `comparingDouble` for primitive fields to **avoid boxing overhead**.
- Chain secondary sorts with `thenComparing()`. Remember that `reversed()` reverses the **whole chain so far**, not just the last step.
- `nullsFirst()` and `nullsLast()` make your comparators **null-safe** — always use them when your data might contain nulls.
- **Never subtract integers** in a comparator — use `Integer.compare(a, b)` to avoid overflow bugs.
- Comparators are **functional interfaces** themselves — they can be stored in variables, composed, and passed around like any other function.

# Phase 17 — Part 1: Complexity Analysis

> Understanding complexity is the single most important theoretical tool in a programmer's arsenal. Before you can evaluate whether your solution is "good," you need a language to describe *how good* it is. That language is Big O notation.

---

## 1. Why Complexity Analysis Matters

Imagine you write a function that works perfectly on 100 elements. But what happens when your data grows to 1 million, 100 million, or a billion rows? A poorly chosen algorithm that felt "fast enough" locally can become a production disaster at scale.

Complexity analysis gives you a **mathematical model** to predict performance *before* you run the code. It answers two questions:
- **Time complexity:** How much longer does this take as input grows?
- **Space complexity:** How much more memory does this use as input grows?

---

## 2. Big O Notation — The Core Idea

Big O notation describes the **upper bound** of an algorithm's growth rate. You drop constants and lower-order terms because at large scales they become irrelevant.

For example, if an algorithm takes `3n² + 5n + 12` operations, we say it's **O(n²)** because for large n, the `n²` term overwhelms everything else.

### The Complexity Hierarchy (Fastest → Slowest)

| Notation | Name | Example |
|---|---|---|
| O(1) | Constant | Array index access |
| O(log n) | Logarithmic | Binary search |
| O(n) | Linear | Single loop |
| O(n log n) | Linearithmic | Merge sort |
| O(n²) | Quadratic | Nested loops |
| O(n³) | Cubic | Triple nested loops |
| O(2ⁿ) | Exponential | Recursive Fibonacci (naive) |
| O(n!) | Factorial | Brute-force permutations |

```java
// O(1) - Constant: no matter how big the array, this is one operation
public int getFirst(int[] arr) {
    return arr[0]; // Always exactly one step
}

// O(log n) - Logarithmic: input is halved each iteration
public int binarySearch(int[] sorted, int target) {
    int left = 0, right = sorted.length - 1;
    while (left <= right) {
        int mid = left + (right - left) / 2; // Avoids integer overflow
        if (sorted[mid] == target) return mid;
        else if (sorted[mid] < target) left = mid + 1;
        else right = mid - 1;
    }
    return -1;
}

// O(n) - Linear: one operation per element
public int sumArray(int[] arr) {
    int sum = 0;
    for (int x : arr) sum += x; // Visits each element once
    return sum;
}

// O(n log n) - Merge Sort (we'll implement this fully in the sorting section)
// Think of it as: divide n times (log n levels), process n elements at each level

// O(n²) - Quadratic: nested loops where both depend on n
public void printAllPairs(int[] arr) {
    for (int i = 0; i < arr.length; i++) {       // n iterations
        for (int j = 0; j < arr.length; j++) {   // n iterations each
            System.out.println(arr[i] + ", " + arr[j]);
        }
    }
}

// O(2^n) - Exponential: naive recursive Fibonacci
public long fib(int n) {
    if (n <= 1) return n;
    return fib(n - 1) + fib(n - 2); // Two recursive calls per call — tree of calls explodes
}
```

---

## 3. Rules for Calculating Big O

### Rule 1: Drop Constants
`O(2n)` → `O(n)`. The constant 2 is irrelevant as n grows toward infinity.

### Rule 2: Drop Non-Dominant Terms
`O(n² + n)` → `O(n²)`. The `n` term is swallowed by `n²`.

### Rule 3: Different Variables for Different Inputs
If you iterate over two separate arrays of sizes `n` and `m`, the complexity is `O(n + m)`, **not** `O(n)`.

```java
// This is O(a + b), NOT O(n)
public void twoLoops(int[] arrA, int[] arrB) {
    for (int x : arrA) System.out.println(x); // O(a)
    for (int x : arrB) System.out.println(x); // O(b)
}

// This is O(a * b) - nested loops over two different arrays
public void nestedTwoArrays(int[] arrA, int[] arrB) {
    for (int x : arrA) {       // O(a)
        for (int y : arrB) {   // O(b)
            System.out.println(x + ", " + y);
        }
    }
}
```

### Rule 4: Recursive Complexity — The Master Theorem (simplified)
For a recursive function that splits into `k` calls each of size `n/2`:
- One call per level (binary search) → O(log n)
- Two calls, each n/2 size, O(n) work per level (merge sort) → O(n log n)
- Two calls, each n-1 size (fibonacci naive) → O(2ⁿ)

---

## 4. Time Complexity vs. Space Complexity

Space complexity accounts for **extra memory** your algorithm allocates — not counting the input itself (unless specified as "total space").

```java
// O(1) space — uses only a few fixed variables regardless of input size
public int maxElement(int[] arr) {
    int max = arr[0];            // Single variable
    for (int x : arr) {
        if (x > max) max = x;   // No new memory allocated per iteration
    }
    return max;
}

// O(n) space — creates a new array proportional to input
public int[] doubleAll(int[] arr) {
    int[] result = new int[arr.length]; // New array of size n
    for (int i = 0; i < arr.length; i++) {
        result[i] = arr[i] * 2;
    }
    return result;
}

// O(n) space — recursion creates a call stack of depth n
public int factorial(int n) {
    if (n <= 1) return 1;
    return n * factorial(n - 1); // n stack frames at maximum depth
}

// O(log n) space — binary search recursion only goes log n deep
public int binarySearchRecursive(int[] arr, int target, int left, int right) {
    if (left > right) return -1;
    int mid = left + (right - left) / 2;
    if (arr[mid] == target) return mid;
    if (arr[mid] < target) return binarySearchRecursive(arr, target, mid + 1, right);
    return binarySearchRecursive(arr, target, left, mid - 1);
    // At most log n recursive calls on the stack at once
}
```

---

## 5. Best, Average, and Worst Case

Big O typically describes the **worst case**, but you need to understand all three:

| Case | Description | Example (QuickSort) |
|---|---|---|
| Best | Most favorable input | Already sorted array: O(n log n) with good pivot |
| Average | Expected over random inputs | O(n log n) |
| Worst | Most unfavorable input | Array sorted in reverse + bad pivot: O(n²) |

```java
// Linear search — best, average, worst illustrated
public int linearSearch(int[] arr, int target) {
    for (int i = 0; i < arr.length; i++) {
        if (arr[i] == target) return i; // Best case: O(1) — found at index 0
                                         // Average:   O(n/2) → O(n)
                                         // Worst:     O(n) — not found, or last element
    }
    return -1;
}
```

The Greek letters used formally:
- **O (Big O)** — upper bound (worst case ceiling)
- **Ω (Omega)** — lower bound (best case floor)
- **Θ (Theta)** — tight bound (both upper and lower — the "exact" growth rate)

In interviews, Big O is almost always what's asked.

---

## 6. Amortized Analysis

Some operations are occasionally expensive but **cheap on average** across many operations. The classic example is `ArrayList.add()`.

When an ArrayList is full, it doubles in size (copies all elements → O(n)). But most `add()` calls are O(1). The cost of that one expensive resize is **spread ("amortized") across** all the cheap inserts.

```java
import java.util.ArrayList;

// Demonstrating amortized O(1) for ArrayList add
ArrayList<Integer> list = new ArrayList<>();
// Internally starts with capacity 10
// When you add the 11th element: resizes to 20 (copies 10 elements — O(n) once)
// When you add the 21st element: resizes to 40 (copies 20 elements — O(n) once)
// Total cost of n insertions: n + n/2 + n/4 + ... = 2n → O(n) total → O(1) amortized per operation
for (int i = 0; i < 1000; i++) {
    list.add(i); // Each call is amortized O(1)
}
```

**Key amortized complexities to remember:**
- `ArrayList.add()` → amortized O(1)
- `ArrayDeque.push()`/`pop()` → amortized O(1)
- `HashMap.put()` → amortized O(1) (though worst case is O(n) with all collisions)

---

## 7. Complexity of Java Collections (Quick Reference)

| Structure | Get | Add | Remove | Contains | Notes |
|---|---|---|---|---|---|
| `ArrayList` | O(1) | O(1) amort. | O(n) | O(n) | Index-based |
| `LinkedList` | O(n) | O(1) at ends | O(1) at ends | O(n) | No random access |
| `HashSet` | — | O(1) avg | O(1) avg | O(1) avg | No order |
| `TreeSet` | — | O(log n) | O(log n) | O(log n) | Sorted |
| `HashMap` | O(1) avg | O(1) avg | O(1) avg | O(1) avg | No order |
| `TreeMap` | O(log n) | O(log n) | O(log n) | O(log n) | Sorted keys |
| `PriorityQueue` | O(1) peek | O(log n) | O(log n) | O(n) | Min by default |

---

## 8. Important Points & Best Practices

**Think before you code.** Before writing a single line, ask yourself: "What is the acceptable complexity for this problem given its constraints?" If n ≤ 10⁶, O(n log n) is fine; O(n²) is almost certainly not.

**Common constraint → complexity mapping used in competitive programming:**
- n ≤ 10 → O(n!) is fine
- n ≤ 25 → O(2ⁿ) is fine
- n ≤ 100 → O(n³) is fine
- n ≤ 1,000 → O(n²) is fine
- n ≤ 10⁵ → O(n log n) needed
- n ≤ 10⁶ → O(n) or O(n log n) at most
- n ≤ 10⁹ → O(log n) or O(1)

**Don't over-optimize prematurely.** A clean O(n log n) solution is vastly better than a convoluted O(n) solution with massive constants and poor readability. Always profile first.

**Space-time tradeoffs are real.** You can often trade memory for speed (e.g., memoization converts O(2ⁿ) time to O(n) time by using O(n) space). Know when this tradeoff is worth it.

**Hidden complexity lurks in Java APIs.** For example, `String.contains()` is O(n*m) not O(1). `Collections.sort()` is O(n log n). `HashMap.get()` is O(1) average but O(n) worst case if your `hashCode()` is terrible.

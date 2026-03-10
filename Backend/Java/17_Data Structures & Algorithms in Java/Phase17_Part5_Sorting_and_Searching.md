# Phase 17 — Part 5: Sorting and Searching Algorithms

> Sorting is arguably the most studied problem in computer science. Understanding each algorithm's mechanics, trade-offs, and best/worst cases gives you an intuition for algorithm design that transfers across all other problems.

---

## 1. Sorting Algorithms Overview

| Algorithm | Time (Best) | Time (Avg) | Time (Worst) | Space | Stable? | Notes |
|---|---|---|---|---|---|---|
| Bubble Sort | O(n) | O(n²) | O(n²) | O(1) | Yes | Good only for nearly-sorted data |
| Selection Sort | O(n²) | O(n²) | O(n²) | O(1) | No | Minimal swaps — good when writes are expensive |
| Insertion Sort | O(n) | O(n²) | O(n²) | O(1) | Yes | Excellent for small or nearly-sorted arrays |
| Merge Sort | O(n log n) | O(n log n) | O(n log n) | O(n) | Yes | Best choice when stability is required |
| Quick Sort | O(n log n) | O(n log n) | O(n²) | O(log n) | No | Fastest in practice for large random data |
| Heap Sort | O(n log n) | O(n log n) | O(n log n) | O(1) | No | Guaranteed O(n log n) with O(1) space |
| Counting Sort | O(n+k) | O(n+k) | O(n+k) | O(k) | Yes | Only works for integers in a known range |
| Radix Sort | O(nk) | O(nk) | O(nk) | O(n+k) | Yes | Excellent for fixed-length integers/strings |

> **What does "stable" mean?** A stable sort preserves the relative order of equal elements. If two records have the same sort key, a stable sort keeps them in their original order. This matters when sorting objects by multiple keys in sequence.

---

## 2. Bubble Sort — O(n²)

**Mechanism:** Repeatedly walk through the array, comparing adjacent pairs and swapping them if out of order. After each pass, the largest unsorted element "bubbles" to its correct position at the end. The algorithm is done when a full pass produces no swaps.

```java
// Bubble Sort with an early termination optimization
// If no swaps happened in a pass, the array is already sorted — stop early
public static void bubbleSort(int[] arr) {
    int n = arr.length;
    for (int i = 0; i < n - 1; i++) {
        boolean swapped = false; // Track whether any swap occurred in this pass
        // After pass i, the last i elements are guaranteed to be in their correct positions
        for (int j = 0; j < n - 1 - i; j++) {
            if (arr[j] > arr[j + 1]) {
                // Swap adjacent elements that are in the wrong order
                int temp = arr[j];
                arr[j] = arr[j + 1];
                arr[j + 1] = temp;
                swapped = true;
            }
        }
        if (!swapped) break; // No swaps = array is sorted; exit early (makes best case O(n))
    }
}
```

Think of sorting a hand of playing cards by repeatedly scanning left-to-right and swapping any card that's larger than its right neighbor. After enough passes, the largest cards will have floated all the way to the right.

---

## 3. Selection Sort — O(n²)

**Mechanism:** Find the minimum element in the unsorted portion and swap it to its correct position. Each pass grows the sorted portion by one. The key characteristic: it makes the minimum possible number of swaps (at most n-1), which is useful when write operations are expensive.

```java
public static void selectionSort(int[] arr) {
    int n = arr.length;
    for (int i = 0; i < n - 1; i++) {
        // Find the index of the minimum element in arr[i..n-1]
        int minIdx = i;
        for (int j = i + 1; j < n; j++) {
            if (arr[j] < arr[minIdx]) minIdx = j;
        }
        // Swap the found minimum into its correct position (the i-th position)
        int temp = arr[minIdx];
        arr[minIdx] = arr[i];
        arr[i] = temp;
    }
}
// Note: Selection sort is NOT stable. Swapping can move equal elements out of order.
// Example: [3a, 3b, 1] → after first pass: [1, 3b, 3a] — 3a and 3b swapped relative order.
```

---

## 4. Insertion Sort — O(n²) average, O(n) best

**Mechanism:** Build the sorted array one element at a time. For each new element, insert it into its correct position within the already-sorted portion by shifting larger elements right. Excellent for small arrays (n < ~20) and nearly-sorted arrays — Java's `Arrays.sort()` uses this for small subarrays.

```java
public static void insertionSort(int[] arr) {
    int n = arr.length;
    for (int i = 1; i < n; i++) {
        int key = arr[i]; // The element we're currently placing into the sorted portion
        int j = i - 1;
        // Shift all elements in the sorted portion that are greater than key one position right
        while (j >= 0 && arr[j] > key) {
            arr[j + 1] = arr[j]; // Shift right to make room
            j--;
        }
        arr[j + 1] = key; // Insert key at its correct position
    }
}
// Mental model: Like sorting playing cards in your hand.
// You pick up a new card and slide it left until it's in the right spot
// relative to the cards you already hold.
```

**When does insertion sort beat merge/quick sort?** When the array is already nearly sorted (most elements are already near their final positions), insertion sort only does a small number of shifts per element — approaching O(n) total operations. Quicksort/mergesort always pay O(n log n) even then.

---

## 5. Merge Sort — O(n log n), Stable

**Mechanism:** Divide the array into two halves, recursively sort each half, then merge the two sorted halves back together. This is a classic **divide-and-conquer** algorithm.

**Key insight — why O(n log n)?** The "divide" step splits the problem in half each time → log n levels of recursion. At each level, the "merge" step processes every element exactly once → O(n) per level. Total: O(n log n).

```java
public static void mergeSort(int[] arr, int left, int right) {
    if (left >= right) return; // Base case: a single element or empty range is already sorted

    int mid = left + (right - left) / 2; // Find midpoint (avoids integer overflow vs (left+right)/2)
    mergeSort(arr, left, mid);      // Recursively sort the left half
    mergeSort(arr, mid + 1, right); // Recursively sort the right half
    merge(arr, left, mid, right);   // Merge the two sorted halves
}

// Merge two sorted subarrays: arr[left..mid] and arr[mid+1..right]
private static void merge(int[] arr, int left, int mid, int right) {
    // Temporary array to hold the merged result
    int[] temp = new int[right - left + 1];
    int i = left;       // Pointer into left subarray
    int j = mid + 1;    // Pointer into right subarray
    int k = 0;          // Pointer into temp array

    // Compare elements from both halves and add the smaller to temp
    while (i <= mid && j <= right) {
        if (arr[i] <= arr[j]) { // <= makes the sort STABLE (left wins ties)
            temp[k++] = arr[i++];
        } else {
            temp[k++] = arr[j++];
        }
    }
    // Copy any remaining elements from the left half (right half is already in place)
    while (i <= mid) temp[k++] = arr[i++];
    // Copy any remaining elements from the right half
    while (j <= right) temp[k++] = arr[j++];

    // Copy temp back to the original array
    System.arraycopy(temp, 0, arr, left, temp.length);
}

// Usage:
int[] arr = {5, 2, 8, 1, 9, 3};
mergeSort(arr, 0, arr.length - 1);
// arr is now: [1, 2, 3, 5, 8, 9]
```

**Merge Sort's advantage:** It's the only common sort that's both O(n log n) worst-case AND stable. This is why Java uses a variant of merge sort (TimSort) for `Arrays.sort(Object[])` and `Collections.sort()` — when sorting objects, stability is usually important.

---

## 6. Quick Sort — O(n log n) average, O(n²) worst

**Mechanism:** Choose a "pivot" element. Partition the array so that all elements smaller than the pivot are to its left and all larger elements are to its right. Recursively sort both partitions. The pivot is now in its final sorted position.

**Why is it called "quick"?** Because the constant factors in practice are very small — cache-friendly in-place partitioning makes it faster than merge sort for most random data despite the same O(n log n) average.

```java
public static void quickSort(int[] arr, int low, int high) {
    if (low < high) {
        // Partition: place pivot in correct position, return its index
        int pivotIndex = partition(arr, low, high);
        // Recursively sort elements before and after partition
        quickSort(arr, low, pivotIndex - 1);
        quickSort(arr, pivotIndex + 1, high);
    }
}

// Lomuto partition scheme — simpler to understand
private static int partition(int[] arr, int low, int high) {
    int pivot = arr[high]; // Choose last element as pivot (simple, but can be bad for sorted arrays)
    int i = low - 1;       // i marks the boundary: everything at or before i is ≤ pivot

    for (int j = low; j < high; j++) {
        if (arr[j] <= pivot) {
            i++; // Expand the "small elements" region
            // Swap arr[j] into the small region
            int temp = arr[i]; arr[i] = arr[j]; arr[j] = temp;
        }
        // Elements > pivot stay in place; they're to the right of i
    }
    // Place pivot at its final sorted position (right after all elements ≤ pivot)
    int temp = arr[i + 1]; arr[i + 1] = arr[high]; arr[high] = temp;
    return i + 1; // Return pivot's final index
}

// Randomized pivot to avoid O(n²) worst case on sorted/reverse-sorted inputs
// This makes the average case O(n log n) with very high probability
private static int partitionRandom(int[] arr, int low, int high) {
    int randomIdx = low + (int)(Math.random() * (high - low + 1));
    // Swap random element to the end, then use Lomuto partition
    int temp = arr[randomIdx]; arr[randomIdx] = arr[high]; arr[high] = temp;
    return partition(arr, low, high);
}
```

**When does quicksort degrade to O(n²)?** When the pivot is always the minimum or maximum of the current subarray — typically on already-sorted or reverse-sorted input with the naive "last element as pivot" strategy. Randomizing the pivot choice almost entirely eliminates this.

---

## 7. Heap Sort — O(n log n), In-Place

**Mechanism:** Build a max-heap from the array, then repeatedly extract the maximum (swap it to the end) and restore the heap property. The result is a sorted array, all in place with O(1) extra space.

```java
public static void heapSort(int[] arr) {
    int n = arr.length;

    // Phase 1: Build a max-heap in O(n) using Floyd's algorithm
    // Start from the last non-leaf node and heapify down
    for (int i = n / 2 - 1; i >= 0; i--) {
        heapifyDown(arr, n, i);
    }

    // Phase 2: Extract max repeatedly and place it at the end
    for (int i = n - 1; i > 0; i--) {
        // The root (arr[0]) is always the current maximum — swap it to position i
        int temp = arr[0]; arr[0] = arr[i]; arr[i] = temp;
        // Restore heap property for the reduced heap (size i, ignoring sorted tail)
        heapifyDown(arr, i, 0);
    }
}

private static void heapifyDown(int[] arr, int heapSize, int i) {
    int largest = i;
    int left = 2 * i + 1, right = 2 * i + 2;

    if (left < heapSize && arr[left] > arr[largest]) largest = left;
    if (right < heapSize && arr[right] > arr[largest]) largest = right;

    if (largest != i) {
        int temp = arr[i]; arr[i] = arr[largest]; arr[largest] = temp;
        heapifyDown(arr, heapSize, largest);
    }
}
```

---

## 8. Counting Sort — O(n + k)

**Mechanism:** Count how many times each distinct value appears. Use those counts to directly place elements in their sorted positions. Only works for non-negative integers within a known, bounded range.

```java
// Counting Sort — O(n + k) where k is the range of input values
// Ideal when k is small relative to n (e.g., sort exam scores 0-100)
public static int[] countingSort(int[] arr) {
    if (arr.length == 0) return arr;

    // Find the range of values
    int max = Arrays.stream(arr).max().getAsInt();
    int min = Arrays.stream(arr).min().getAsInt();
    int range = max - min + 1;

    // Count occurrences of each value
    int[] count = new int[range];
    for (int x : arr) count[x - min]++;

    // Reconstruct the sorted array from the counts
    int[] sorted = new int[arr.length];
    int idx = 0;
    for (int i = 0; i < range; i++) {
        while (count[i]-- > 0) {
            sorted[idx++] = i + min;
        }
    }
    return sorted;
}
```

---

## 9. Radix Sort — O(nk)

**Mechanism:** Sort numbers digit by digit, from least significant to most significant. Uses a stable sort (counting sort) at each digit position. Works because sorting all numbers by their ones digit, then tens digit, etc., ultimately produces the globally sorted order.

```java
public static void radixSort(int[] arr) {
    int max = Arrays.stream(arr).max().getAsInt();
    // Process one digit place at a time: 1s, 10s, 100s, ...
    for (int exp = 1; max / exp > 0; exp *= 10) {
        countingSortByDigit(arr, exp);
    }
}

// Stable counting sort by the digit at position 'exp' (1 = ones, 10 = tens, etc.)
private static void countingSortByDigit(int[] arr, int exp) {
    int n = arr.length;
    int[] output = new int[n];
    int[] count = new int[10]; // Digits 0-9

    // Count occurrences of each digit
    for (int x : arr) count[(x / exp) % 10]++;

    // Cumulative count: count[i] now contains position AFTER the last element with digit i
    for (int i = 1; i < 10; i++) count[i] += count[i - 1];

    // Build output array (iterate backwards to maintain stability)
    for (int i = n - 1; i >= 0; i--) {
        int digit = (arr[i] / exp) % 10;
        output[count[digit] - 1] = arr[i];
        count[digit]--;
    }

    System.arraycopy(output, 0, arr, 0, n);
}
```

---

## 10. Java's Built-in Sorting (TimSort)

Java's `Arrays.sort()` and `Collections.sort()` use **TimSort** — a hybrid of merge sort and insertion sort designed by Tim Peters. It identifies already-sorted "runs" in the data and merges them efficiently. For primitive arrays, `Arrays.sort(int[])` uses dual-pivot quicksort.

```java
// Sorting primitives — uses dual-pivot quicksort — NOT stable
int[] nums = {5, 2, 8, 1};
Arrays.sort(nums); // [1, 2, 5, 8]

// Sorting objects — uses TimSort — STABLE
Integer[] boxed = {5, 2, 8, 1};
Arrays.sort(boxed); // [1, 2, 5, 8]

// Sorting with a custom Comparator
String[] words = {"banana", "apple", "cherry"};
Arrays.sort(words, Comparator.comparingInt(String::length)); // Sort by length

// Sorting a list
List<Integer> list = new ArrayList<>(Arrays.asList(5, 2, 8, 1));
Collections.sort(list);
// or equivalently:
list.sort(Comparator.naturalOrder());
list.sort(Comparator.reverseOrder());

// Sorting objects by multiple fields
record Student(String name, int grade) {}
List<Student> students = List.of(new Student("Alice", 90), new Student("Bob", 85), new Student("Charlie", 90));
students.stream()
    .sorted(Comparator.comparingInt(Student::grade).reversed()
        .thenComparing(Student::name))
    .forEach(System.out::println);
// Output: Alice 90, Charlie 90, Bob 85
```

---

## 11. Binary Search — O(log n)

Binary search only works on **sorted arrays**. The idea is elegant: compare the target to the middle element. If equal, done. If target is smaller, search the left half. If larger, search the right half. Each comparison halves the search space.

```java
// Iterative Binary Search — preferred (no stack overhead)
public static int binarySearch(int[] sorted, int target) {
    int left = 0, right = sorted.length - 1;
    while (left <= right) {
        int mid = left + (right - left) / 2; // CRITICAL: avoids integer overflow!
        // (left + right) / 2 can overflow when left and right are both large ints
        if (sorted[mid] == target) return mid;
        else if (sorted[mid] < target) left = mid + 1;  // Target is in right half
        else right = mid - 1;                           // Target is in left half
    }
    return -1; // Not found
}

// Finding the LEFTMOST index of a target (first occurrence in duplicates)
// This is the "lower bound" — smallest index where arr[i] >= target
public static int lowerBound(int[] arr, int target) {
    int left = 0, right = arr.length;
    while (left < right) { // Note: right = arr.length (one past end), not arr.length-1
        int mid = left + (right - left) / 2;
        if (arr[mid] < target) left = mid + 1;
        else right = mid; // Keep searching left even when we find target
    }
    return left; // First position where arr[i] >= target
}

// Finding the RIGHTMOST index of a target (last occurrence)
// This is the "upper bound" — smallest index where arr[i] > target
public static int upperBound(int[] arr, int target) {
    int left = 0, right = arr.length;
    while (left < right) {
        int mid = left + (right - left) / 2;
        if (arr[mid] <= target) left = mid + 1; // Note: <= instead of <
        else right = mid;
    }
    return left - 1; // Last position where arr[i] == target (or -1 if not found)
}

// Binary search on the ANSWER — a powerful pattern!
// "Find the minimum speed to finish all tasks in H hours" → binary search over possible speeds
// "Find the minimum capacity needed to ship within D days" → binary search over capacities
// The key insight: if a value X works, all values > X also work (monotonic condition)
public static int findMinSpeed(int[] piles, int h) {
    int left = 1, right = Arrays.stream(piles).max().getAsInt();
    while (left < right) {
        int mid = left + (right - left) / 2;
        if (canFinish(piles, h, mid)) right = mid; // Speed mid works, try lower
        else left = mid + 1;                        // Speed mid doesn't work, try higher
    }
    return left;
}

private static boolean canFinish(int[] piles, int h, int speed) {
    int hours = 0;
    for (int pile : piles) hours += (pile + speed - 1) / speed; // = Math.ceil(pile / speed)
    return hours <= h;
}
```

---

## 12. Important Points & Best Practices

**In Java, always prefer `Arrays.sort()` and `Collections.sort()`.** They use highly optimized, well-tested implementations (TimSort / dual-pivot quicksort) that are faster than anything you'd write from scratch. Only implement sorting algorithms yourself for learning or when the problem explicitly requires it.

**Stability determines when to use which algorithm.** If you need to preserve the relative order of equal elements (e.g., sorting a table by column A, then by column B while keeping column-A order within ties), you need a stable sort. Java's `List.sort()` and `Collections.sort()` are stable. `Arrays.sort(int[])` is NOT stable (it sorts primitives).

**The mid-point overflow bug is real.** `(left + right) / 2` can overflow when `left` and `right` are both close to `Integer.MAX_VALUE`. Always use `left + (right - left) / 2`. This is a famous bug that existed in Java's own `Arrays.binarySearch()` for nearly a decade.

**Binary search beyond "find element X".** The most powerful applications of binary search aren't "is this value in the array?" but rather "what's the minimum X such that condition(X) is true?" — binary search on the answer space. Any time you see a problem asking for a minimum/maximum value where feasibility is monotonic, think binary search.

**Counting sort and radix sort are often interview answers.** If someone asks "can you sort this in O(n)?", the answer is yes — but only under specific constraints (integer values in a bounded range). Understanding this distinction is what separates strong engineers from those who think O(n log n) is the universal lower bound for sorting.

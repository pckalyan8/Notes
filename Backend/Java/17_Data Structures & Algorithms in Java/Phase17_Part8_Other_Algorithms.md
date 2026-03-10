# Phase 17 — Part 8: Other Algorithms

> Beyond sorting and graph algorithms, there's a collection of powerful algorithmic techniques that appear constantly in technical interviews and real systems. These aren't individual algorithms so much as *patterns of thinking* — once you internalize each pattern, you'll start recognizing where it applies naturally.

---

## 1. Two Pointers

The **two pointers** technique uses two index variables that move through a data structure — often an array or string — in a coordinated way. The power is that you convert an O(n²) brute-force (check every pair) into O(n) by exploiting structure (sorted order, a sum constraint) to move both pointers intelligently.

There are two variations: pointers starting at opposite ends (converging), and pointers both starting at the left (the fast/slow pattern).

### Converging Pointers — Target Sum in a Sorted Array

```java
// Two Sum in sorted array: find two numbers that add up to target
// Brute force: O(n²) — check every pair
// Two pointers: O(n) — move pointers based on whether the current sum is too big or too small
public int[] twoSumSorted(int[] sorted, int target) {
    int left = 0, right = sorted.length - 1;
    while (left < right) {
        int sum = sorted[left] + sorted[right];
        if (sum == target) return new int[]{left, right}; // Found the pair
        else if (sum < target) left++;  // Sum is too small: advance left to get a bigger number
        else right--;                   // Sum is too big: retreat right to get a smaller number
    }
    return new int[]{-1, -1}; // No pair found
}

// Why does this work?
// The sorted order gives us the crucial insight:
//   if sum < target → left element is too small → only increasing left can help
//   if sum > target → right element is too big → only decreasing right can help
// Each step eliminates at least one candidate, so at most n steps are needed.
```

### Three Sum — Reduce to Two Sum with a Loop

```java
// Three Sum: find all unique triplets that sum to zero
// Sort first, then fix one element and use two pointers for the remaining two
// O(n²) total — much better than O(n³) brute force
public List<List<Integer>> threeSum(int[] nums) {
    Arrays.sort(nums);
    List<List<Integer>> result = new ArrayList<>();
    for (int i = 0; i < nums.length - 2; i++) {
        // Skip duplicate values for the fixed element to avoid duplicate triplets
        if (i > 0 && nums[i] == nums[i - 1]) continue;

        int left = i + 1, right = nums.length - 1;
        while (left < right) {
            int sum = nums[i] + nums[left] + nums[right];
            if (sum == 0) {
                result.add(List.of(nums[i], nums[left], nums[right]));
                // Skip duplicates for left and right pointers
                while (left < right && nums[left] == nums[left + 1]) left++;
                while (left < right && nums[right] == nums[right - 1]) right--;
                left++; right--;
            } else if (sum < 0) left++;
            else right--;
        }
    }
    return result;
}
```

### Container With Most Water

```java
// Given heights of vertical bars, find two bars that form the container holding the most water.
// Key insight: container area = min(height[left], height[right]) * (right - left)
// If we move the TALLER bar inward, the width decreases and the shorter bar (the bottleneck) stays.
// Only moving the SHORTER bar inward can possibly increase the area.
public int maxWater(int[] heights) {
    int left = 0, right = heights.length - 1;
    int maxArea = 0;
    while (left < right) {
        int area = Math.min(heights[left], heights[right]) * (right - left);
        maxArea = Math.max(maxArea, area);
        // Move the pointer with the smaller height (hoping to find a taller bar)
        if (heights[left] < heights[right]) left++;
        else right--;
    }
    return maxArea;
}
```

---

## 2. Sliding Window

The **sliding window** technique maintains a contiguous subarray (or substring) that satisfies some constraint, and slides it through the input. It avoids recomputing from scratch for each window by incrementally adding the new right element and removing the old left element.

### Fixed-Size Window — Maximum Sum Subarray of Size k

```java
// Maximum sum of any contiguous subarray of exactly size k — O(n)
// Brute force: O(n*k) — compute sum from scratch for each window
// Sliding window: O(n) — maintain a running sum, add one element and remove one
public int maxSumSubarray(int[] nums, int k) {
    int windowSum = 0;
    // Compute the sum of the first window
    for (int i = 0; i < k; i++) windowSum += nums[i];

    int maxSum = windowSum;
    // Slide the window: add the next element and remove the first element of the old window
    for (int i = k; i < nums.length; i++) {
        windowSum += nums[i];       // Add the new right element
        windowSum -= nums[i - k];  // Remove the element that just left the window
        maxSum = Math.max(maxSum, windowSum);
    }
    return maxSum;
}
```

### Variable-Size Window — Longest Substring Without Repeating Characters

When the window size is variable, we expand the right pointer freely and shrink the left pointer when the constraint is violated.

```java
// Longest substring without repeating characters — O(n)
// Window = contiguous substring; constraint = all characters unique
// Expand right: add char to the window
// Shrink left: if the new char is already in the window, remove chars from the left until it's not
public int lengthOfLongestSubstring(String s) {
    Map<Character, Integer> lastSeen = new HashMap<>(); // char → last seen index
    int left = 0, maxLen = 0;
    for (int right = 0; right < s.length(); right++) {
        char c = s.charAt(right);
        // If we've seen this character and it's within our current window:
        // Jump left pointer past its previous occurrence to eliminate the duplicate
        if (lastSeen.containsKey(c) && lastSeen.get(c) >= left) {
            left = lastSeen.get(c) + 1; // Shrink window from left
        }
        lastSeen.put(c, right); // Update last seen position
        maxLen = Math.max(maxLen, right - left + 1); // Current window size
    }
    return maxLen;
}
```

### Minimum Window Substring

This is the most powerful form of the sliding window: find the smallest window that contains all characters of a pattern string.

```java
// Minimum window in s that contains all characters of t — O(n + m)
public String minWindow(String s, String t) {
    if (s.isEmpty() || t.isEmpty()) return "";

    // Count required characters from t
    Map<Character, Integer> need = new HashMap<>();
    for (char c : t.toCharArray()) need.merge(c, 1, Integer::sum);

    int left = 0, formed = 0, required = need.size(); // 'required' = distinct chars needed
    int minLen = Integer.MAX_VALUE, minLeft = 0;
    Map<Character, Integer> have = new HashMap<>(); // Current window character counts

    for (int right = 0; right < s.length(); right++) {
        char c = s.charAt(right);
        have.merge(c, 1, Integer::sum);

        // Check if this character's count in window meets the requirement
        if (need.containsKey(c) && have.get(c).intValue() == need.get(c).intValue()) {
            formed++; // One more required character is fully satisfied
        }

        // Try to shrink the window from the left while all requirements are still met
        while (formed == required) {
            if (right - left + 1 < minLen) {
                minLen = right - left + 1;
                minLeft = left;
            }
            char leftChar = s.charAt(left++);
            have.merge(leftChar, -1, Integer::sum);
            if (need.containsKey(leftChar) && have.get(leftChar) < need.get(leftChar)) {
                formed--; // Window no longer satisfies this character's requirement
            }
        }
    }
    return minLen == Integer.MAX_VALUE ? "" : s.substring(minLeft, minLeft + minLen);
}
```

---

## 3. Divide and Conquer

**Divide and conquer** splits a problem into smaller subproblems, solves each subproblem independently, and combines the results. We already saw this with merge sort. Here are a few more powerful applications.

### Count Inversions in an Array

An inversion is a pair `(i, j)` where `i < j` but `arr[i] > arr[j]`. Counting inversions is a natural extension of merge sort — during the merge step, every time a right-half element is placed before one or more left-half elements, those are all inversions.

```java
// Count inversions using modified Merge Sort — O(n log n)
// An inversion pair [i,j] where i < j and arr[i] > arr[j]
public long countInversions(int[] arr) {
    return mergeCount(arr, 0, arr.length - 1);
}

private long mergeCount(int[] arr, int left, int right) {
    if (left >= right) return 0; // Base case: single element, no inversions
    int mid = left + (right - left) / 2;
    long count = 0;
    count += mergeCount(arr, left, mid);         // Count inversions in left half
    count += mergeCount(arr, mid + 1, right);    // Count inversions in right half
    count += mergeAndCount(arr, left, mid, right); // Count split inversions
    return count;
}

private long mergeAndCount(int[] arr, int left, int mid, int right) {
    int[] temp = Arrays.copyOfRange(arr, left, right + 1);
    int i = 0, j = mid - left + 1, k = left;
    long count = 0;
    int leftLen = mid - left + 1;

    while (i < leftLen && j < temp.length) {
        if (temp[i] <= temp[j]) {
            arr[k++] = temp[i++];
        } else {
            // temp[j] is being placed before temp[i..leftLen-1]
            // All remaining elements in the left half (leftLen - i of them) form inversions with temp[j]
            count += (leftLen - i);
            arr[k++] = temp[j++];
        }
    }
    while (i < leftLen) arr[k++] = temp[i++];
    while (j < temp.length) arr[k++] = temp[j++];
    return count;
}
```

---

## 4. Greedy Algorithms

A **greedy algorithm** makes the locally optimal choice at each step, hoping that a series of locally optimal choices leads to a globally optimal solution. Greedy algorithms are simple to implement and efficient, but they only work when the problem has the **greedy choice property** (a global optimum can be reached by making locally optimal choices) and **optimal substructure**.

The challenge is *proving* that greedy works for a given problem. When in doubt, use DP — it's safer. But when greedy does work, it's much simpler.

### Activity Selection — Maximum Non-Overlapping Intervals

```java
// Given intervals, select the maximum number of non-overlapping intervals.
// Greedy insight: always pick the interval that ends the EARLIEST.
// By finishing early, we leave the most room for future intervals to fit.
// This is a classic example where greedy is provably optimal.
public int maxNonOverlapping(int[][] intervals) {
    // Sort by end time — we always want to commit to the earliest-ending interval first
    Arrays.sort(intervals, Comparator.comparingInt(a -> a[1]));
    int count = 1;                   // At minimum, we can always take the first interval
    int lastEnd = intervals[0][1];   // End time of the last interval we selected

    for (int i = 1; i < intervals.length; i++) {
        if (intervals[i][0] >= lastEnd) { // This interval starts at or after the last one ends
            count++;
            lastEnd = intervals[i][1]; // Update the end time
        }
        // Otherwise, this interval overlaps with our last selection — skip it
    }
    return count;
}
```

### Jump Game — Can You Reach the End?

```java
// Each element represents the max jump length from that position.
// Greedy: track the farthest reachable index as you walk through the array.
public boolean canJump(int[] nums) {
    int farthest = 0; // The farthest index we can currently reach
    for (int i = 0; i < nums.length; i++) {
        if (i > farthest) return false; // We can't reach index i — stuck!
        farthest = Math.max(farthest, i + nums[i]); // Can we reach farther from here?
    }
    return true;
}

// Minimum number of jumps to reach the end
// Greedy: at each "jump boundary", pick the jump that reaches the farthest
public int minJumps(int[] nums) {
    int jumps = 0, currentEnd = 0, farthest = 0;
    for (int i = 0; i < nums.length - 1; i++) {
        farthest = Math.max(farthest, i + nums[i]); // Explore farthest from current range
        if (i == currentEnd) { // We've exhausted the current jump's range — must jump
            jumps++;
            currentEnd = farthest; // Jump to the farthest reachable position
        }
    }
    return jumps;
}
```

---

## 5. Backtracking

**Backtracking** is a refinement of brute-force search. You build a solution incrementally, and whenever you determine that the current partial solution cannot lead to a valid complete solution, you "backtrack" — undo the last choice and try the next option.

Think of it as navigating a decision tree: you go down one branch until you either find a solution or hit a dead end, then you climb back up and try the next branch.

The standard backtracking template is:
```
solve(state):
    if state is a complete valid solution: add to results; return
    for each possible choice:
        if choice is valid (not violating any constraint):
            apply choice
            solve(new state)    ← recurse deeper
            undo choice         ← backtrack
```

### Permutations

```java
// Generate all permutations of a list of distinct numbers
// Time: O(n * n!) — n! permutations, each takes O(n) to copy to results
public List<List<Integer>> permutations(int[] nums) {
    List<List<Integer>> result = new ArrayList<>();
    backtrackPerms(nums, new ArrayList<>(), new boolean[nums.length], result);
    return result;
}

private void backtrackPerms(int[] nums, List<Integer> current, boolean[] used, List<List<Integer>> result) {
    if (current.size() == nums.length) { // Complete permutation formed
        result.add(new ArrayList<>(current)); // Add a COPY (not the mutable reference!)
        return;
    }
    for (int i = 0; i < nums.length; i++) {
        if (used[i]) continue; // Skip elements already in the current permutation
        used[i] = true;
        current.add(nums[i]);           // ← Make choice
        backtrackPerms(nums, current, used, result); // ← Explore
        current.remove(current.size() - 1); // ← Undo choice (backtrack)
        used[i] = false;
    }
}
```

### Subsets

```java
// Generate all subsets (power set) of a list of distinct numbers
// Time: O(n * 2^n) — 2^n subsets, each takes O(n) to copy
public List<List<Integer>> subsets(int[] nums) {
    List<List<Integer>> result = new ArrayList<>();
    backtrackSubsets(nums, 0, new ArrayList<>(), result);
    return result;
}

private void backtrackSubsets(int[] nums, int start, List<Integer> current, List<List<Integer>> result) {
    result.add(new ArrayList<>(current)); // Add current subset (including empty set at start)
    for (int i = start; i < nums.length; i++) {
        current.add(nums[i]);                            // Include nums[i]
        backtrackSubsets(nums, i + 1, current, result); // Recurse for elements after i
        current.remove(current.size() - 1);             // Backtrack: exclude nums[i]
    }
}
```

### N-Queens

```java
// Place N queens on an N×N chessboard so no two queens threaten each other.
// A queen threatens any piece in the same row, column, or diagonal.
public List<List<String>> solveNQueens(int n) {
    List<List<String>> result = new ArrayList<>();
    int[] queens = new int[n]; // queens[row] = column where queen is placed in that row
    Arrays.fill(queens, -1);
    // Track which columns and diagonals are occupied (for O(1) conflict checking)
    Set<Integer> cols = new HashSet<>(), diag1 = new HashSet<>(), diag2 = new HashSet<>();
    backtrackQueens(n, 0, queens, cols, diag1, diag2, result);
    return result;
}

private void backtrackQueens(int n, int row, int[] queens, Set<Integer> cols,
                              Set<Integer> diag1, Set<Integer> diag2, List<List<String>> result) {
    if (row == n) { // All n queens placed successfully
        result.add(buildBoard(queens, n));
        return;
    }
    for (int col = 0; col < n; col++) {
        // Check if this cell is safe: no conflict with any previously placed queen
        // Two queens are on the same diagonal if (row - col) is equal (diag1)
        // or (row + col) is equal (diag2)
        if (cols.contains(col) || diag1.contains(row - col) || diag2.contains(row + col)) continue;

        queens[row] = col;   // Place queen
        cols.add(col); diag1.add(row - col); diag2.add(row + col);

        backtrackQueens(n, row + 1, queens, cols, diag1, diag2, result); // Next row

        queens[row] = -1;    // Remove queen (backtrack)
        cols.remove(col); diag1.remove(row - col); diag2.remove(row + col);
    }
}

private List<String> buildBoard(int[] queens, int n) {
    List<String> board = new ArrayList<>();
    for (int row = 0; row < n; row++) {
        char[] rowChars = new char[n];
        Arrays.fill(rowChars, '.');
        rowChars[queens[row]] = 'Q';
        board.add(new String(rowChars));
    }
    return board;
}
```

---

## 6. Bit Manipulation

Bit manipulation operates directly on the binary representations of numbers. These tricks are often O(1) and incredibly fast because they use single CPU instructions.

```java
// Core bit operations
int a = 0b1010; // Binary: 1010 = decimal 10
int b = 0b1100; // Binary: 1100 = decimal 12

int andOp  = a & b;  // AND:  1000 = 8  (both bits must be 1)
int orOp   = a | b;  // OR:   1110 = 14 (either bit must be 1)
int xorOp  = a ^ b;  // XOR:  0110 = 6  (bits must differ)
int notOp  = ~a;     // NOT:  ...11110101 (flips all bits)
int leftSh = a << 1; // Left shift:  10100 = 20 (multiply by 2)
int rightSh = a >> 1;// Right shift: 0101  = 5  (divide by 2)

// =====================================================
// Common Bit Tricks
// =====================================================

// Check if a number is even or odd — faster than % 2
boolean isOdd = (n & 1) == 1; // If the least significant bit is 1, it's odd

// Check if a number is a power of 2
// Powers of 2 in binary: 1, 10, 100, 1000...
// Powers of 2 minus 1:   0, 01, 011, 0111...
// AND them together: always 0 for powers of 2
boolean isPowerOf2 = n > 0 && (n & (n - 1)) == 0;
// Example: n=8 → 1000, n-1=7 → 0111, 1000 & 0111 = 0000 ✓
// Example: n=6 → 0110, n-1=5 → 0101, 0110 & 0101 = 0100 ≠ 0 ✗

// Get the k-th bit (0-indexed from right)
int getBit(int n, int k) { return (n >> k) & 1; } // Shift right to bring k-th bit to position 0, then AND with 1

// Set the k-th bit to 1
int setBit(int n, int k) { return n | (1 << k); } // OR with a mask that has only the k-th bit set

// Clear the k-th bit (set to 0)
int clearBit(int n, int k) { return n & ~(1 << k); } // AND with a mask that has all bits 1 except position k

// Count number of set bits (Hamming weight) — Brian Kernighan's algorithm
int countBits(int n) {
    int count = 0;
    while (n != 0) {
        n &= (n - 1); // Removes the lowest set bit (n & n-1 always clears the rightmost 1 bit)
        count++;
    }
    return count;
}
// Example: n=12 (1100) → n&(n-1) = 1100 & 1011 = 1000 (count=1)
//                        n&(n-1) = 1000 & 0111 = 0000 (count=2) → returns 2

// XOR trick: find the one number that appears only once (all others appear twice)
// XOR of a number with itself is 0; XOR of a number with 0 is the number itself
// So XOR-ing all elements cancels out pairs, leaving the unique element
int findUnique(int[] nums) {
    int result = 0;
    for (int n : nums) result ^= n; // All paired numbers cancel out; unique one remains
    return result;
}
// Example: [2, 3, 5, 3, 2] → 0 ^ 2 ^ 3 ^ 5 ^ 3 ^ 2 = 5

// Swap two numbers without a temporary variable using XOR
void swap(int[] arr, int i, int j) {
    if (i == j) return; // Must check: if i==j, XOR makes it 0!
    arr[i] ^= arr[j]; // a = a^b
    arr[j] ^= arr[i]; // b = b^(a^b) = a
    arr[i] ^= arr[j]; // a = (a^b)^a = b
}

// Generate all subsets using bit masking (elegant alternative to backtracking)
// For n elements, there are 2^n subsets, and each can be represented by an n-bit mask
// Bit k of the mask = 1 means "include element k in this subset"
List<List<Integer>> allSubsets(int[] nums) {
    int n = nums.length;
    List<List<Integer>> result = new ArrayList<>();
    for (int mask = 0; mask < (1 << n); mask++) { // Iterate all 2^n possible masks
        List<Integer> subset = new ArrayList<>();
        for (int k = 0; k < n; k++) {
            if ((mask & (1 << k)) != 0) { // If bit k is set, include nums[k]
                subset.add(nums[k]);
            }
        }
        result.add(subset);
    }
    return result;
}
```

---

## 7. Important Points & Best Practices

**Two pointers only work on sorted data (for the converging pattern).** When you're moving the left pointer right on a "sum too small" condition, you're relying on the fact that the array is sorted so moving right always increases the value. On an unsorted array, the pattern breaks completely. Always check or sort first.

**Sliding window is for subarray/substring problems with a "contiguous" constraint.** The moment a problem asks about "contiguous elements" with a condition on their aggregate (sum, count, distinct characters), think sliding window. The template is always the same: expand right freely, shrink left when the constraint is violated, track the best valid window.

**Greedy requires a proof.** You can't just guess that greedy works — you need an exchange argument: "assume the optimal solution differs from the greedy solution; swapping the greedy choice in for the non-greedy choice cannot worsen the result." If you can't articulate this argument, consider DP instead, because greedy often produces near-optimal but not guaranteed-optimal solutions.

**Backtracking's performance depends entirely on pruning.** A backtracking algorithm that never prunes (never rejects partial solutions early) is just brute-force enumeration. The power comes from recognizing constraints early — "if the current partial sum already exceeds the target, don't add more elements." Good constraint checking can reduce the effective search space from millions to hundreds.

**Bit manipulation is a knowledge game.** Unlike two pointers or DP (which require conceptual insight), bit tricks are largely memorized patterns. Learn the common ones: `n & (n-1)` clears the lowest set bit, `n & -n` isolates the lowest set bit, `x ^ x = 0`, `x ^ 0 = x`, and bitmask DP for subsets of up to ~20 elements. These appear in competitive programming and some high-level technical interviews.

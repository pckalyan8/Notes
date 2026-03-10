# Phase 17 — Part 7: Dynamic Programming

> Dynamic Programming (DP) is the art of solving complex problems by breaking them into overlapping subproblems and reusing solutions rather than recomputing them. Once you recognize the DP pattern, you can convert exponential-time solutions into polynomial ones.

---

## 1. What Is Dynamic Programming?

DP applies when a problem has two properties:

**Optimal Substructure:** The optimal solution to the problem can be built from optimal solutions to its subproblems. The best way to rob a row of houses depends on the best way to rob smaller prefixes of that row.

**Overlapping Subproblems:** The recursive solution solves the same subproblems many times. Fibonacci is the classic example — computing `fib(5)` requires `fib(4)` and `fib(3)`, computing `fib(4)` requires `fib(3)` again, and so on. The naive recursion recomputes `fib(3)` twice, `fib(2)` three times, etc.

The key idea: **compute each subproblem exactly once, store the result, and look it up instead of recomputing.**

There are two approaches to achieve this:

**Memoization (Top-Down):** Start with the recursive solution, add a cache (a map or array). Before computing a subproblem, check if it's already in the cache. This approach is intuitive because it follows the natural recursive structure of the problem.

**Tabulation (Bottom-Up):** Fill a table iteratively, starting from the smallest subproblems and building up to the final answer. This avoids recursion overhead and is often more space-efficient.

---

## 2. Fibonacci — The "Hello World" of DP

```java
// Naive recursion — O(2^n): recomputes the same values exponentially many times
// fib(5) = fib(4) + fib(3)
//        = (fib(3) + fib(2)) + (fib(2) + fib(1))
//        = ((fib(2) + fib(1)) + fib(2)) + ... fib(3) computed TWICE, fib(2) computed THREE times
public long fibNaive(int n) {
    if (n <= 1) return n;
    return fibNaive(n - 1) + fibNaive(n - 2); // Exponential time — completely impractical for n > 40
}

// Memoization (Top-Down DP) — O(n) time, O(n) space
// Modify the naive solution: add a cache, check before computing
public long fibMemo(int n, long[] memo) {
    if (n <= 1) return n;
    if (memo[n] != 0) return memo[n]; // Cache hit: return stored result instead of recomputing
    memo[n] = fibMemo(n - 1, memo) + fibMemo(n - 2, memo); // Store result before returning
    return memo[n];
}

// Tabulation (Bottom-Up DP) — O(n) time, O(n) space
// Fill the table from smallest subproblem (fib(0), fib(1)) up to fib(n)
public long fibTab(int n) {
    if (n <= 1) return n;
    long[] dp = new long[n + 1];
    dp[0] = 0; // Base case
    dp[1] = 1; // Base case
    for (int i = 2; i <= n; i++) {
        dp[i] = dp[i - 1] + dp[i - 2]; // Each value depends only on the two before it
    }
    return dp[n];
}

// Space-optimized tabulation — O(n) time, O(1) space
// Notice that fib(i) only depends on fib(i-1) and fib(i-2) — we don't need the whole table!
public long fibOptimal(int n) {
    if (n <= 1) return n;
    long prev2 = 0, prev1 = 1;
    for (int i = 2; i <= n; i++) {
        long curr = prev1 + prev2; // Compute current from the two previous
        prev2 = prev1;             // Slide the window
        prev1 = curr;
    }
    return prev1;
}
```

The space optimization from O(n) → O(1) is possible because fib(i) only looks back two steps. Many DP problems allow similar space optimization once you notice that you only need the "last row" or "last few values" of the DP table.

---

## 3. The General DP Framework

Most DP solutions follow this thought process:

1. **Define the state:** What does `dp[i]` or `dp[i][j]` represent? This is the hardest part.
2. **Write the recurrence:** How does `dp[i]` relate to smaller subproblems?
3. **Identify base cases:** What are the trivially known values?
4. **Determine the iteration order:** We must compute smaller subproblems first.
5. **Extract the answer:** Usually `dp[n]` or `max(dp[n][...])`.

---

## 4. Classic 1D DP Problems

### 0/1 Knapsack

You have a knapsack with capacity W and n items, each with a weight and value. Choose items (each item can be taken at most once) to maximize total value without exceeding capacity W.

```java
// 0/1 Knapsack — O(n * W) time and space
// State: dp[i][w] = max value using the first i items with capacity w
// Recurrence:
//   If we don't take item i: dp[i][w] = dp[i-1][w] (same as without item i)
//   If we take item i (only if weight[i] <= w): dp[i][w] = dp[i-1][w - weight[i]] + value[i]
//   We take the maximum of these two choices
public int knapsack(int[] weights, int[] values, int W) {
    int n = weights.length;
    int[][] dp = new int[n + 1][W + 1];
    // dp[0][*] = 0 (no items → no value)
    // dp[*][0] = 0 (no capacity → no value)

    for (int i = 1; i <= n; i++) {
        int wi = weights[i - 1]; // Weight of item i (1-indexed dp, 0-indexed arrays)
        int vi = values[i - 1];  // Value of item i
        for (int w = 0; w <= W; w++) {
            // Option 1: Don't take item i
            dp[i][w] = dp[i - 1][w];
            // Option 2: Take item i (only if it fits)
            if (wi <= w) {
                dp[i][w] = Math.max(dp[i][w], dp[i - 1][w - wi] + vi);
            }
        }
    }
    return dp[n][W]; // Max value using all n items with capacity W
}

// Space-optimized: O(W) space — iterate w in reverse so we don't use items twice
public int knapsackOptimized(int[] weights, int[] values, int W) {
    int[] dp = new int[W + 1]; // dp[w] = max value with capacity w
    for (int i = 0; i < weights.length; i++) {
        // Iterate right to left: ensures each item is counted at most once
        // If we went left to right, we might use item i multiple times in the same pass
        for (int w = W; w >= weights[i]; w--) {
            dp[w] = Math.max(dp[w], dp[w - weights[i]] + values[i]);
        }
    }
    return dp[W];
}
```

Think of the DP table as a grid where rows are items and columns are capacities. Each cell answers "what's the most I can carry if I've only considered the first i items and have capacity w?" You fill this grid row by row, and the answer sits in the bottom-right corner.

### Coin Change — Minimum Coins

Given coin denominations and a target amount, find the minimum number of coins to make that amount. Unlike the knapsack, each coin can be used unlimited times (unbounded knapsack).

```java
// Coin Change — O(amount * coins.length)
// State: dp[i] = minimum number of coins to make amount i
// Recurrence: dp[i] = min(dp[i - coin] + 1) for all coins where coin <= i
public int coinChange(int[] coins, int amount) {
    int[] dp = new int[amount + 1];
    Arrays.fill(dp, amount + 1); // Initialize to "infinity" (impossible value)
    dp[0] = 0; // Base case: 0 coins needed to make amount 0

    for (int i = 1; i <= amount; i++) {
        for (int coin : coins) {
            if (coin <= i) {
                // Can we reach amount i by using one of this coin on top of amount (i - coin)?
                dp[i] = Math.min(dp[i], dp[i - coin] + 1);
            }
        }
    }
    return dp[amount] > amount ? -1 : dp[amount]; // -1 if amount is unreachable
}
```

Notice we use `amount + 1` as the "infinity" sentinel because the maximum number of coins you'd ever need is `amount` (all 1-cent coins). So any value of `amount + 1` means "impossible."

---

## 5. Longest Common Subsequence (LCS)

Given two strings, find the length of their longest common subsequence — a subsequence that appears in both strings in the same relative order (but not necessarily contiguous).

For example, LCS("ABCBDAB", "BDCAB") = "BCAB" or "BDAB" — length 4.

```java
// LCS — O(m * n) time and space
// State: dp[i][j] = LCS length of s1[0..i-1] and s2[0..j-1]
// Recurrence:
//   If s1[i-1] == s2[j-1]: characters match! LCS = 1 + LCS of remaining strings
//   Otherwise: LCS = max(LCS without last char of s1, LCS without last char of s2)
public int lcs(String s1, String s2) {
    int m = s1.length(), n = s2.length();
    int[][] dp = new int[m + 1][n + 1];
    // dp[0][*] = 0 (empty s1 has no common subsequence with anything)
    // dp[*][0] = 0 (empty s2 has no common subsequence with anything)

    for (int i = 1; i <= m; i++) {
        for (int j = 1; j <= n; j++) {
            if (s1.charAt(i - 1) == s2.charAt(j - 1)) {
                // Characters match: extend the LCS from the previous state
                dp[i][j] = 1 + dp[i - 1][j - 1];
            } else {
                // No match: take the best LCS by skipping one character from either string
                dp[i][j] = Math.max(dp[i - 1][j], dp[i][j - 1]);
            }
        }
    }
    return dp[m][n];
}
```

---

## 6. Longest Increasing Subsequence (LIS)

Find the length of the longest subsequence of an array where elements are strictly increasing.

```java
// LIS — O(n²) DP approach
// State: dp[i] = length of LIS ending at index i
// Recurrence: dp[i] = max(dp[j] + 1) for all j < i where nums[j] < nums[i]
public int lis(int[] nums) {
    int n = nums.length;
    int[] dp = new int[n];
    Arrays.fill(dp, 1); // Every element is an LIS of length 1 by itself (base case)

    int maxLen = 1;
    for (int i = 1; i < n; i++) {
        for (int j = 0; j < i; j++) {
            if (nums[j] < nums[i]) { // nums[i] can extend the LIS ending at j
                dp[i] = Math.max(dp[i], dp[j] + 1);
            }
        }
        maxLen = Math.max(maxLen, dp[i]);
    }
    return maxLen;
}

// Optimal LIS — O(n log n) using patience sorting / binary search
// Maintain a list 'tails' where tails[i] = smallest tail element of all increasing
// subsequences of length i+1. This list is always sorted, so we can binary search.
public int lisOptimal(int[] nums) {
    List<Integer> tails = new ArrayList<>();
    for (int num : nums) {
        // Find the first element in tails that is >= num (using binary search)
        int pos = Collections.binarySearch(tails, num);
        if (pos < 0) pos = -(pos + 1); // Convert to insertion point if not found
        if (pos == tails.size()) tails.add(num); // num extends the longest subsequence
        else tails.set(pos, num); // Replace to keep the smallest possible tail value
    }
    return tails.size(); // The number of elements is the LIS length
}
```

---

## 7. Edit Distance (Levenshtein Distance)

Given two strings, find the minimum number of single-character edits (insertions, deletions, or substitutions) needed to transform one string into the other. Used in spell checkers, DNA sequence analysis, and diff tools.

```java
// Edit Distance — O(m * n)
// State: dp[i][j] = minimum edits to transform s1[0..i-1] into s2[0..j-1]
// Recurrence:
//   If s1[i-1] == s2[j-1]: no edit needed, carry forward dp[i-1][j-1]
//   Otherwise: 1 + min(
//     dp[i-1][j]   → delete from s1 (or insert into s2)
//     dp[i][j-1]   → insert into s1 (or delete from s2)
//     dp[i-1][j-1] → substitute (replace s1[i-1] with s2[j-1])
//   )
public int editDistance(String s1, String s2) {
    int m = s1.length(), n = s2.length();
    int[][] dp = new int[m + 1][n + 1];

    // Base cases:
    // dp[i][0] = i: transform s1[0..i-1] to empty string requires i deletions
    // dp[0][j] = j: transform empty string to s2[0..j-1] requires j insertions
    for (int i = 0; i <= m; i++) dp[i][0] = i;
    for (int j = 0; j <= n; j++) dp[0][j] = j;

    for (int i = 1; i <= m; i++) {
        for (int j = 1; j <= n; j++) {
            if (s1.charAt(i - 1) == s2.charAt(j - 1)) {
                dp[i][j] = dp[i - 1][j - 1]; // Characters match: no edit needed
            } else {
                dp[i][j] = 1 + Math.min(dp[i - 1][j - 1],      // Substitute
                               Math.min(dp[i - 1][j],           // Delete from s1
                                        dp[i][j - 1]));         // Insert into s1
            }
        }
    }
    return dp[m][n];
}
```

---

## 8. Matrix Chain Multiplication

Given a sequence of matrices, find the optimal parenthesization to minimize the number of scalar multiplications. This is a classic interval DP problem.

```java
// Matrix Chain Multiplication — O(n³)
// dp[i][j] = minimum cost to multiply matrices from index i to j
// We try every possible "split point" k between i and j
public int matrixChainOrder(int[] dims) {
    // dims[i] and dims[i+1] are the dimensions of matrix i
    // Matrix i has dimensions dims[i] × dims[i+1]
    int n = dims.length - 1; // Number of matrices
    int[][] dp = new int[n][n];
    // dp[i][i] = 0: a single matrix costs nothing to "multiply"

    // Fill for chains of increasing length
    for (int len = 2; len <= n; len++) {       // len = number of matrices in the chain
        for (int i = 0; i <= n - len; i++) {   // i = start of the chain
            int j = i + len - 1;               // j = end of the chain
            dp[i][j] = Integer.MAX_VALUE;
            // Try every possible split point k
            for (int k = i; k < j; k++) {
                // Cost = cost to multiply matrices [i..k] + cost to multiply [k+1..j]
                //       + cost of the final multiplication (rows of left × cols of right × shared dim)
                int cost = dp[i][k] + dp[k + 1][j] + dims[i] * dims[k + 1] * dims[j + 1];
                dp[i][j] = Math.min(dp[i][j], cost);
            }
        }
    }
    return dp[0][n - 1]; // Minimum cost to multiply all n matrices
}
```

---

## 9. DP on Trees

DP on trees means computing DP values for each node using the values of its children. A common pattern is to do post-order DFS (compute children first) and combine their results.

```java
// Example: Maximum path sum in a binary tree where the path can go through any node
// For each node, consider two choices:
// 1. The node is the "peak" of the path (uses both left and right subtrees)
// 2. The node extends a path from its parent (uses at most one subtree)
class MaxPathSumTree {
    int globalMax = Integer.MIN_VALUE;

    public int maxPathSum(TreeNode root) {
        maxGain(root);
        return globalMax;
    }

    // Returns the maximum gain if we EXTEND a path upward through this node
    // (meaning we can only "turn" in one direction)
    private int maxGain(TreeNode node) {
        if (node == null) return 0;

        // Recursively get max gain from left and right subtrees
        // Use max(..., 0) to ignore negative-sum subtrees (we can always "not take" them)
        int leftGain = Math.max(maxGain(node.left), 0);
        int rightGain = Math.max(maxGain(node.right), 0);

        // At this node, the best path THROUGH this node (using both sides) is:
        int pathThroughNode = node.val + leftGain + rightGain;
        globalMax = Math.max(globalMax, pathThroughNode); // Update global answer

        // Return the best path EXTENDING upward (can only use one side)
        return node.val + Math.max(leftGain, rightGain);
    }
}
```

---

## 10. Important Points & Best Practices

**The hardest part of DP is defining the state.** If your state definition is wrong, your recurrence will be wrong, and the table will give wrong answers. Spend more time thinking about "what does `dp[i]` represent?" than on anything else. A good state definition makes the recurrence fall naturally into place.

**Memoization vs. tabulation — when to use which.** Memoization is easier to write (you modify an existing recursive solution) and only computes states that are actually needed. Tabulation is often more space-efficient (you can throw away rows you don't need anymore) and avoids recursion stack overhead. For most interview problems, either works.

**Space optimization is often possible.** When `dp[i][j]` only depends on `dp[i-1][j]` and `dp[i][j-1]` (like LCS, edit distance), you can reduce the 2D table to just two 1D arrays (the current and previous row). When it only depends on the previous row going left-to-right or right-to-left, you might be able to use a single 1D array.

**Recognizing DP problems:** Look for "minimum/maximum/count of ways" combined with "choices at each step" and a constraint. Phrases like "optimal substructure," "minimum cost," "number of ways to reach," "longest/shortest subsequence," or "can we partition/fill" are strong DP signals. Also, if the brute-force recursion is exponential and you notice repeated subproblems, DP will almost certainly help.

**The recurrence encodes the choice.** Every DP recurrence reflects a decision: "at this step, what are my options, and which option gives the best result?" For knapsack it's "take this item or skip it." For LCS it's "these characters match (extend) or they don't (skip one string)." Once you see the choices clearly, the recurrence writes itself.

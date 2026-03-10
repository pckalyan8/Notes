# Phase 17 — Part 3: Trees and Heaps

> Trees are hierarchical data structures that model relationships where each item has at most one parent but potentially many children. They underpin databases (B-Trees), compilers (ASTs), file systems, and most of the efficient algorithms you'll use in practice.

---

## 1. Binary Tree — The Foundation

A **binary tree** is a tree where every node has at most two children, called `left` and `right`. Unlike a linked list (which is actually a degenerate tree), a well-balanced binary tree allows efficient search in O(log n) time.

**Terminology to internalize:**
- **Root:** The topmost node with no parent.
- **Leaf:** A node with no children.
- **Height:** The number of edges on the longest path from root to a leaf.
- **Depth:** The number of edges from the root to a given node.
- **Complete tree:** Every level is fully filled except possibly the last (which fills left to right). This is what a heap uses.
- **Perfect tree:** Every internal node has exactly two children and all leaves are at the same level.
- **Balanced tree:** Heights of left and right subtrees differ by at most 1 for every node (AVL/Red-Black guarantee this).

```java
public class BinaryTree {

    // Node is the building block — each node knows its value and its two children
    static class Node {
        int val;
        Node left, right;
        Node(int val) { this.val = val; }
    }

    Node root;

    // =====================================================
    // TREE TRAVERSALS — the most fundamental tree operations
    // =====================================================

    // In-Order: LEFT → ROOT → RIGHT
    // For a BST, this produces elements in sorted ascending order!
    public void inOrder(Node node) {
        if (node == null) return;
        inOrder(node.left);          // Recurse left subtree first
        System.out.print(node.val + " ");
        inOrder(node.right);         // Then recurse right subtree
    }

    // Pre-Order: ROOT → LEFT → RIGHT
    // Useful for: copying a tree, serialization, prefix expression evaluation
    public void preOrder(Node node) {
        if (node == null) return;
        System.out.print(node.val + " "); // Process root BEFORE children
        preOrder(node.left);
        preOrder(node.right);
    }

    // Post-Order: LEFT → RIGHT → ROOT
    // Useful for: deleting a tree (delete children before parent), postfix evaluation
    public void postOrder(Node node) {
        if (node == null) return;
        postOrder(node.left);
        postOrder(node.right);
        System.out.print(node.val + " "); // Process root AFTER children
    }

    // Level-Order (BFS): Process nodes level by level using a Queue
    // Useful for: finding shortest path, level-by-level processing
    public void levelOrder(Node root) {
        if (root == null) return;
        Queue<Node> queue = new LinkedList<>();
        queue.offer(root);              // Start with the root in the queue

        while (!queue.isEmpty()) {
            int levelSize = queue.size(); // How many nodes are on this level
            for (int i = 0; i < levelSize; i++) {
                Node node = queue.poll();
                System.out.print(node.val + " ");
                if (node.left != null) queue.offer(node.left);   // Enqueue children for next level
                if (node.right != null) queue.offer(node.right);
            }
            System.out.println(); // Newline between levels
        }
    }

    // Tree Height — O(n), must visit every node
    public int height(Node node) {
        if (node == null) return -1; // Height of empty tree is -1 (or 0, depending on convention)
        int leftHeight = height(node.left);
        int rightHeight = height(node.right);
        return 1 + Math.max(leftHeight, rightHeight); // Height = 1 + max child height
    }

    // Count nodes — O(n)
    public int countNodes(Node node) {
        if (node == null) return 0;
        return 1 + countNodes(node.left) + countNodes(node.right);
    }

    // Check if tree is balanced — O(n)
    // A tree is balanced if for every node, |height(left) - height(right)| <= 1
    public boolean isBalanced(Node node) {
        return checkHeight(node) != -1;
    }

    // Returns -1 if unbalanced, otherwise returns actual height
    private int checkHeight(Node node) {
        if (node == null) return 0;
        int left = checkHeight(node.left);
        if (left == -1) return -1;  // Left subtree is already unbalanced
        int right = checkHeight(node.right);
        if (right == -1) return -1; // Right subtree is already unbalanced
        if (Math.abs(left - right) > 1) return -1; // This node is unbalanced
        return 1 + Math.max(left, right);
    }
}
```

---

## 2. Binary Search Tree (BST)

A BST is a binary tree with one critical ordering property: **for every node, all values in its left subtree are smaller, and all values in its right subtree are larger.** This property is what enables O(log n) search in a balanced tree.

```java
public class BST {
    Node root;

    static class Node {
        int val;
        Node left, right;
        Node(int val) { this.val = val; }
    }

    // Insert — O(log n) average, O(n) worst case (degenerate/skewed tree)
    public void insert(int val) {
        root = insertRec(root, val);
    }

    private Node insertRec(Node node, int val) {
        if (node == null) return new Node(val); // Found the correct empty spot
        if (val < node.val) node.left = insertRec(node.left, val);   // Go left if smaller
        else if (val > node.val) node.right = insertRec(node.right, val); // Go right if larger
        // If val == node.val: duplicate — we ignore it (or handle as needed)
        return node;
    }

    // Search — O(log n) average
    public boolean contains(int val) {
        return searchRec(root, val);
    }

    private boolean searchRec(Node node, int val) {
        if (node == null) return false;  // Reached a null — not found
        if (val == node.val) return true;
        if (val < node.val) return searchRec(node.left, val);
        return searchRec(node.right, val);
    }

    // Delete — the trickiest BST operation: three cases
    public void delete(int val) {
        root = deleteRec(root, val);
    }

    private Node deleteRec(Node node, int val) {
        if (node == null) return null; // Value not found

        if (val < node.val) {
            node.left = deleteRec(node.left, val);   // Search in left subtree
        } else if (val > node.val) {
            node.right = deleteRec(node.right, val); // Search in right subtree
        } else {
            // Found the node to delete — three cases:

            // Case 1: No children (leaf node) — simply remove it
            if (node.left == null && node.right == null) return null;

            // Case 2: One child — replace the node with its only child
            if (node.left == null) return node.right;
            if (node.right == null) return node.left;

            // Case 3: Two children — find the in-order successor (smallest in right subtree)
            // Replace current node's value with successor's value, then delete the successor
            Node successor = findMin(node.right);
            node.val = successor.val; // Copy successor's value here
            node.right = deleteRec(node.right, successor.val); // Delete the successor
        }
        return node;
    }

    // Find minimum: keep going left until there's no more left child
    private Node findMin(Node node) {
        while (node.left != null) node = node.left;
        return node;
    }

    // Find maximum: keep going right
    public int findMax(Node node) {
        while (node.right != null) node = node.right;
        return node.val;
    }

    // Validate BST — check that every node respects the BST property
    // Common mistake: checking only direct children. Must check the RANGE constraint.
    public boolean isValidBST(Node node) {
        return validate(node, Long.MIN_VALUE, Long.MAX_VALUE);
    }

    private boolean validate(Node node, long min, long max) {
        if (node == null) return true;
        if (node.val <= min || node.val >= max) return false; // Violates BST property
        // Left subtree: values must be < node.val; right subtree: values must be > node.val
        return validate(node.left, min, node.val) && validate(node.right, node.val, max);
    }

    // In-order traversal on BST gives sorted output — the most useful BST property
    public void inOrderSorted(Node node) {
        if (node == null) return;
        inOrderSorted(node.left);
        System.out.print(node.val + " "); // Prints in ascending order for a valid BST
        inOrderSorted(node.right);
    }
}
```

**BST Weakness:** Without balancing, repeated insertions of sorted data create a degenerate tree (a linked list). Searching becomes O(n) instead of O(log n). That's why AVL and Red-Black trees exist.

---

## 3. AVL Tree (Self-Balancing BST)

An **AVL tree** (named after Adelson-Velsky and Landis, 1962) is a BST that automatically rebalances itself after every insertion or deletion. It maintains the invariant that for every node, the height difference between left and right subtrees (the **balance factor**) is at most 1.

When balance is violated, **rotations** restore it. There are four rotation types.

```java
public class AVLTree {

    static class Node {
        int val, height;
        Node left, right;
        Node(int val) {
            this.val = val;
            this.height = 1; // New nodes start at height 1
        }
    }

    private int height(Node n) {
        return (n == null) ? 0 : n.height;
    }

    // Balance factor: positive = left-heavy, negative = right-heavy
    private int getBalance(Node n) {
        return (n == null) ? 0 : height(n.left) - height(n.right);
    }

    private void updateHeight(Node n) {
        n.height = 1 + Math.max(height(n.left), height(n.right));
    }

    // Right Rotation — used when LEFT subtree is too heavy (balance > 1)
    //       y                x
    //      / \             /   \
    //     x   T3   →     T1    y
    //    / \                  / \
    //   T1  T2              T2   T3
    private Node rightRotate(Node y) {
        Node x = y.left;
        Node T2 = x.right;

        x.right = y;  // y becomes the right child of x
        y.left = T2;  // T2 fills the gap

        updateHeight(y); // Update y first (it's now lower)
        updateHeight(x); // Then update x (it's now higher)
        return x;        // x is the new subtree root
    }

    // Left Rotation — used when RIGHT subtree is too heavy (balance < -1)
    //     x                  y
    //    / \               /   \
    //   T1   y     →      x     T3
    //       / \          / \
    //     T2   T3       T1  T2
    private Node leftRotate(Node x) {
        Node y = x.right;
        Node T2 = y.left;

        y.left = x;
        x.right = T2;

        updateHeight(x);
        updateHeight(y);
        return y;
    }

    // Insert with auto-balancing
    public Node insert(Node node, int val) {
        // Step 1: Standard BST insert
        if (node == null) return new Node(val);
        if (val < node.val) node.left = insert(node.left, val);
        else if (val > node.val) node.right = insert(node.right, val);
        else return node; // Duplicate

        // Step 2: Update this node's height
        updateHeight(node);

        // Step 3: Check balance and perform rotations if needed
        int balance = getBalance(node);

        // Left-Left Case: left subtree too heavy, inserted into its left child
        if (balance > 1 && val < node.left.val) return rightRotate(node);

        // Right-Right Case: right subtree too heavy, inserted into its right child
        if (balance < -1 && val > node.right.val) return leftRotate(node);

        // Left-Right Case: left subtree too heavy, inserted into its right child
        // First rotate left child left, then rotate current node right
        if (balance > 1 && val > node.left.val) {
            node.left = leftRotate(node.left);
            return rightRotate(node);
        }

        // Right-Left Case: right subtree too heavy, inserted into its left child
        if (balance < -1 && val < node.right.val) {
            node.right = rightRotate(node.right);
            return leftRotate(node);
        }

        return node; // No rotation needed
    }
}
```

**AVL vs Red-Black Tree:**
- AVL trees are **more strictly balanced** → faster lookups (O(log n) with smaller constant).
- Red-Black trees require **fewer rotations on insert/delete** → faster writes.
- Java's `TreeMap` and `TreeSet` use a Red-Black tree. Most databases use B-Trees (generalization for disk storage).

---

## 4. Red-Black Tree (Conceptual Overview)

A Red-Black tree is a BST where every node is colored red or black, and four rules are enforced:
1. Every node is red or black.
2. The root is black.
3. Every null leaf is black.
4. If a node is red, both its children are black (no two adjacent reds).
5. All paths from any node to its descendant null leaves have the same number of black nodes.

These rules ensure the tree height is at most 2 log(n+1), giving O(log n) for all operations. Java's `TreeMap` is backed by a Red-Black tree — you use it every time you call `TreeMap.put()` or `TreeSet.add()`.

```java
// In Java you use TreeMap/TreeSet which are backed by a Red-Black tree
import java.util.TreeMap;
import java.util.TreeSet;

TreeMap<Integer, String> rbTree = new TreeMap<>();
rbTree.put(5, "five");   // O(log n) — Red-Black tree insert with potential rotations
rbTree.put(3, "three");
rbTree.put(7, "seven");
rbTree.put(1, "one");

System.out.println(rbTree.firstKey()); // 1 — Red-Black tree keeps keys sorted
System.out.println(rbTree.lastKey());  // 7
System.out.println(rbTree.floorKey(4)); // 3 — largest key ≤ 4
System.out.println(rbTree.ceilingKey(4)); // 5 — smallest key ≥ 4
```

---

## 5. Heap (Min-Heap and Max-Heap)

A **heap** is a **complete binary tree** that satisfies the heap property:
- **Min-heap:** Every parent is ≤ its children. The root is always the minimum.
- **Max-heap:** Every parent is ≥ its children. The root is always the maximum.

The elegant trick: a complete binary tree can be stored as an **array** with no wasted space. For a node at index `i`:
- Left child is at `2*i + 1`
- Right child is at `2*i + 2`
- Parent is at `(i - 1) / 2`

**Applications:** Priority queues, Dijkstra's algorithm, heap sort, "top K elements" problems.

```java
// Min-Heap implementation using an array
public class MinHeap {
    private int[] heap;
    private int size;
    private int capacity;

    public MinHeap(int capacity) {
        this.capacity = capacity;
        heap = new int[capacity];
        size = 0;
    }

    private int parent(int i) { return (i - 1) / 2; }
    private int leftChild(int i) { return 2 * i + 1; }
    private int rightChild(int i) { return 2 * i + 2; }
    private void swap(int i, int j) {
        int temp = heap[i];
        heap[i] = heap[j];
        heap[j] = temp;
    }

    // Insert — O(log n)
    // Add to end, then "bubble up" until heap property is restored
    public void insert(int val) {
        if (size == capacity) throw new RuntimeException("Heap is full");
        heap[size] = val;   // Add at the end of the array
        int i = size++;     // i is the index of the newly inserted element
        // Bubble up: while this node is smaller than its parent, swap them
        while (i > 0 && heap[i] < heap[parent(i)]) {
            swap(i, parent(i));
            i = parent(i); // Move up
        }
    }

    // Extract minimum — O(log n)
    // Remove root (minimum), replace with last element, then "heapify down"
    public int extractMin() {
        if (size == 0) throw new NoSuchElementException();
        int min = heap[0]; // Root is always the minimum
        heap[0] = heap[--size]; // Move last element to root
        heapifyDown(0);   // Restore heap property by sinking the new root down
        return min;
    }

    // Sink an element down to its correct position
    private void heapifyDown(int i) {
        int smallest = i;
        int left = leftChild(i);
        int right = rightChild(i);

        // Find the smallest among node, left child, right child
        if (left < size && heap[left] < heap[smallest]) smallest = left;
        if (right < size && heap[right] < heap[smallest]) smallest = right;

        if (smallest != i) { // If a child is smaller, swap and continue sinking
            swap(i, smallest);
            heapifyDown(smallest);
        }
    }

    // Peek minimum — O(1): just look at root
    public int peekMin() {
        if (size == 0) throw new NoSuchElementException();
        return heap[0];
    }

    // Build heap from an existing array — O(n) (faster than n insertions which is O(n log n)!)
    // This is Floyd's heap construction algorithm
    public static int[] buildHeap(int[] arr) {
        int n = arr.length;
        // Start from the last non-leaf node and heapify down each node
        // Last non-leaf is at index (n/2 - 1)
        for (int i = n / 2 - 1; i >= 0; i--) {
            heapifyDownStatic(arr, n, i);
        }
        return arr;
    }

    private static void heapifyDownStatic(int[] arr, int n, int i) {
        int smallest = i;
        int left = 2 * i + 1, right = 2 * i + 2;
        if (left < n && arr[left] < arr[smallest]) smallest = left;
        if (right < n && arr[right] < arr[smallest]) smallest = right;
        if (smallest != i) {
            int temp = arr[i]; arr[i] = arr[smallest]; arr[smallest] = temp;
            heapifyDownStatic(arr, n, smallest);
        }
    }
}

// =====================================================
// Using Java's PriorityQueue (backed by a min-heap)
// =====================================================

// Default: Min-Heap
PriorityQueue<Integer> minHeap = new PriorityQueue<>();
minHeap.offer(5);
minHeap.offer(1);
minHeap.offer(3);
System.out.println(minHeap.poll()); // 1 — always polls the minimum

// Max-Heap: use a reversed comparator
PriorityQueue<Integer> maxHeap = new PriorityQueue<>(Comparator.reverseOrder());
maxHeap.offer(5);
maxHeap.offer(1);
maxHeap.offer(3);
System.out.println(maxHeap.poll()); // 5 — always polls the maximum

// =====================================================
// Classic problem: Find the K Largest Elements — O(n log k)
// =====================================================
public static int[] kLargest(int[] nums, int k) {
    // Use a min-heap of size k: the root is always the k-th largest seen so far
    PriorityQueue<Integer> minHeap = new PriorityQueue<>();
    for (int num : nums) {
        minHeap.offer(num);
        if (minHeap.size() > k) {
            minHeap.poll(); // Evict the smallest — keeps only the k largest
        }
    }
    return minHeap.stream().mapToInt(Integer::intValue).toArray();
}

// =====================================================
// Classic problem: Merge K Sorted Arrays — O(n log k)
// =====================================================
public static int[] mergeKSortedArrays(int[][] arrays) {
    // Min-heap stores [value, arrayIndex, elementIndex]
    PriorityQueue<int[]> minHeap = new PriorityQueue<>(Comparator.comparingInt(a -> a[0]));
    int totalSize = 0;

    // Initialize heap with the first element of each array
    for (int i = 0; i < arrays.length; i++) {
        if (arrays[i].length > 0) {
            minHeap.offer(new int[]{arrays[i][0], i, 0});
            totalSize += arrays[i].length;
        }
    }

    int[] result = new int[totalSize];
    int idx = 0;
    while (!minHeap.isEmpty()) {
        int[] curr = minHeap.poll();     // Get current minimum
        result[idx++] = curr[0];
        int arrIdx = curr[1], elemIdx = curr[2];
        // Add the next element from the same array (if exists)
        if (elemIdx + 1 < arrays[arrIdx].length) {
            minHeap.offer(new int[]{arrays[arrIdx][elemIdx + 1], arrIdx, elemIdx + 1});
        }
    }
    return result;
}
```

---

## 6. Important Points & Best Practices

**BST worst case is a real concern.** If you insert [1, 2, 3, 4, 5] in order into a plain BST, you get a linked list with O(n) operations. Always use `TreeMap`/`TreeSet` (Red-Black tree) when you need a sorted, self-balancing structure in Java.

**Heap vs. sorted array for priority queues:** A sorted array gives O(1) min/max access but O(n) insertion. A heap gives O(log n) both ways. For dynamic data (insertions and extractions mixed), the heap wins.

**The heap is stored as an array — exploit this.** The index formula (`2*i+1`, `2*i+2`, `(i-1)/2`) is elegant and cache-friendly. This is why heap sort is O(1) space — it sorts in-place using the same array.

**Traversal order matters for different problems:**
- In-order on BST → sorted data.
- Pre-order → tree serialization (you can reconstruct the tree from pre-order + in-order).
- Post-order → safe deletion (children before parents).
- Level-order (BFS) → shortest path, level-by-level processing.

**`PriorityQueue` in Java does NOT support efficient removal of arbitrary elements.** `remove(element)` is O(n) because it searches linearly. If you need efficient arbitrary removal, consider using a `TreeSet` (which is O(log n) for all operations but requires unique elements).

# Phase 17 — Part 4: Hash Tables, Tries, and Graphs

> Hash tables give you O(1) average access by converting keys to array indices. Tries give you O(L) string operations where L is the string length. Graphs let you model any relationship between entities. Together, these three structures cover a huge percentage of real-world problems.

---

## 1. Hash Table

A **hash table** maps keys to values using a **hash function** that converts a key into an array index. The core idea: instead of searching through every element, we compute exactly *where* to look.

**Anatomy of a hash table:**
1. Compute `hash(key)` — convert key to an integer.
2. Map to an index: `index = hash(key) % capacity`.
3. Store the value at that index.

The challenge is **collisions**: two different keys can hash to the same index. There are two main strategies to handle this.

### Strategy 1: Separate Chaining

Each bucket holds a linked list (or other structure) of all entries that hash to that index.

```java
public class HashTableChaining<K, V> {

    // Each bucket holds a list of (key, value) pairs that share this bucket's hash
    private static class Entry<K, V> {
        K key;
        V value;
        Entry<K, V> next; // Chain: multiple entries per bucket if collision occurs

        Entry(K key, V value) {
            this.key = key;
            this.value = value;
        }
    }

    private Entry<K, V>[] buckets;
    private int capacity;
    private int size;
    private static final double LOAD_FACTOR_THRESHOLD = 0.75;

    @SuppressWarnings("unchecked")
    public HashTableChaining(int capacity) {
        this.capacity = capacity;
        buckets = new Entry[capacity];
    }

    // Hash function: spread keys evenly across buckets
    // Using Math.abs to avoid negative indices from integer overflow
    private int hash(K key) {
        return Math.abs(key.hashCode()) % capacity;
    }

    // Put — O(1) average, O(n) worst case (all keys collide to same bucket)
    public void put(K key, V value) {
        int index = hash(key);
        Entry<K, V> current = buckets[index];

        // Walk the chain to see if the key already exists (update if so)
        while (current != null) {
            if (current.key.equals(key)) {
                current.value = value; // Update existing
                return;
            }
            current = current.next;
        }

        // Key not found — insert at the front of the chain (O(1) for linked list prepend)
        Entry<K, V> newEntry = new Entry<>(key, value);
        newEntry.next = buckets[index];
        buckets[index] = newEntry;
        size++;

        // Resize when load factor exceeds threshold to keep chains short
        if ((double) size / capacity > LOAD_FACTOR_THRESHOLD) {
            resize();
        }
    }

    // Get — O(1) average: compute hash, then walk the chain (short on average)
    public V get(K key) {
        int index = hash(key);
        Entry<K, V> current = buckets[index];
        while (current != null) {
            if (current.key.equals(key)) return current.value;
            current = current.next;
        }
        return null; // Key not found
    }

    // Remove — O(1) average
    public void remove(K key) {
        int index = hash(key);
        Entry<K, V> current = buckets[index];
        Entry<K, V> prev = null;
        while (current != null) {
            if (current.key.equals(key)) {
                if (prev == null) buckets[index] = current.next; // Remove head of chain
                else prev.next = current.next;                    // Bypass current node
                size--;
                return;
            }
            prev = current;
            current = current.next;
        }
    }

    // Resize — double capacity and rehash all entries
    // O(n): must re-hash and re-insert every entry into the new array
    @SuppressWarnings("unchecked")
    private void resize() {
        int newCapacity = capacity * 2;
        Entry<K, V>[] newBuckets = new Entry[newCapacity];
        for (Entry<K, V> head : buckets) {
            Entry<K, V> current = head;
            while (current != null) {
                Entry<K, V> next = current.next;
                int newIndex = Math.abs(current.key.hashCode()) % newCapacity;
                current.next = newBuckets[newIndex]; // Prepend to new bucket
                newBuckets[newIndex] = current;
                current = next;
            }
        }
        buckets = newBuckets;
        capacity = newCapacity;
    }
}
```

### Strategy 2: Open Addressing (Linear Probing)

All entries live in the main array. On collision, probe for the next available slot.

```java
public class HashTableOpenAddressing<K, V> {

    private Object[] keys;
    private Object[] values;
    private boolean[] deleted; // Tombstone markers for deleted slots
    private int capacity;
    private int size;

    public HashTableOpenAddressing(int capacity) {
        this.capacity = capacity;
        keys = new Object[capacity];
        values = new Object[capacity];
        deleted = new boolean[capacity];
    }

    private int hash(K key) {
        return Math.abs(key.hashCode()) % capacity;
    }

    // Linear probing: if slot is taken, try next slot, and next, until empty
    @SuppressWarnings("unchecked")
    public void put(K key, V value) {
        int index = hash(key);
        // Probe until we find either the key (update) or an empty/deleted slot (insert)
        while (keys[index] != null && !deleted[index]) {
            if (((K) keys[index]).equals(key)) {
                values[index] = value; // Update existing key
                return;
            }
            index = (index + 1) % capacity; // Linear probe: try next slot
        }
        keys[index] = key;
        values[index] = value;
        deleted[index] = false;
        size++;
    }

    @SuppressWarnings("unchecked")
    public V get(K key) {
        int index = hash(key);
        while (keys[index] != null || deleted[index]) {
            if (!deleted[index] && ((K) keys[index]).equals(key)) {
                return (V) values[index];
            }
            index = (index + 1) % capacity;
        }
        return null;
    }
}
```

### HashMap Internals in Java (Java 8+)

Java's `HashMap` is a sophisticated hybrid: it uses separate chaining but switches from a linked list to a **red-black tree** for a bucket when the chain length exceeds 8. This prevents worst-case O(n) degradation to O(log n).

```java
// Java HashMap key usage and gotchas
import java.util.HashMap;

HashMap<String, Integer> map = new HashMap<>();

// Standard operations — all O(1) average
map.put("alice", 90);
map.put("bob", 85);
map.get("alice");               // 90
map.containsKey("alice");       // true
map.getOrDefault("charlie", 0); // 0 — safe default instead of null check

// computeIfAbsent — lazy initialization, very useful for grouping
map.computeIfAbsent("dave", k -> 0); // Creates entry with 0 if "dave" doesn't exist

// merge — update or create in one call
map.merge("alice", 5, Integer::sum); // alice's score becomes 90 + 5 = 95

// Frequency counting idiom — extremely common interview pattern
String[] words = {"apple", "banana", "apple", "cherry", "banana", "apple"};
Map<String, Integer> freq = new HashMap<>();
for (String word : words) {
    freq.merge(word, 1, Integer::sum); // Increment count, or initialize to 1
}
// Result: {apple=3, banana=2, cherry=1}

// Grouping by a property — e.g., group strings by their length
Map<Integer, List<String>> byLength = new HashMap<>();
for (String word : words) {
    byLength.computeIfAbsent(word.length(), k -> new ArrayList<>()).add(word);
}

// CRITICAL: hashCode() and equals() contract
// If you override equals(), you MUST override hashCode() too!
// Two objects that are equals() must have the same hashCode().
// Otherwise HashMap will fail to find your objects correctly.
```

---

## 2. Trie (Prefix Tree)

A **Trie** is a tree-shaped data structure designed specifically for **string operations**. Each node represents a character, and paths from root to marked nodes form complete words. The name comes from "re**trie**val."

**Why use a Trie instead of a HashMap?**
- `HashMap<String, Boolean>` for contains: O(L) where L is string length (hashing the string costs L steps).
- Trie for contains: also O(L), but a Trie additionally enables O(L) prefix search — "give me all words starting with 'pre'" — which a HashMap cannot do efficiently.

```java
public class Trie {

    private static class TrieNode {
        // Each node has up to 26 children (one per lowercase letter a-z)
        TrieNode[] children = new TrieNode[26];
        boolean isEndOfWord; // True if a complete word ends at this node
    }

    private final TrieNode root = new TrieNode();

    // Insert a word — O(L) where L is the word's length
    // Each character corresponds to one level in the trie
    public void insert(String word) {
        TrieNode node = root;
        for (char c : word.toCharArray()) {
            int index = c - 'a'; // Map 'a'→0, 'b'→1, ..., 'z'→25
            if (node.children[index] == null) {
                node.children[index] = new TrieNode(); // Create node if it doesn't exist
            }
            node = node.children[index]; // Move down to the child
        }
        node.isEndOfWord = true; // Mark the end of this word
    }

    // Search for exact word — O(L)
    public boolean search(String word) {
        TrieNode node = findNode(word);
        return node != null && node.isEndOfWord; // Node exists AND is marked as a word end
    }

    // Check if any word starts with the given prefix — O(L)
    // This is what makes Trie special compared to HashMap!
    public boolean startsWith(String prefix) {
        return findNode(prefix) != null; // Node exists for the prefix path
    }

    // Helper: navigate to the node corresponding to the last char of the string
    private TrieNode findNode(String s) {
        TrieNode node = root;
        for (char c : s.toCharArray()) {
            int index = c - 'a';
            if (node.children[index] == null) return null; // Path doesn't exist
            node = node.children[index];
        }
        return node;
    }

    // Get all words with a given prefix — autocomplete feature
    public List<String> autocomplete(String prefix) {
        List<String> results = new ArrayList<>();
        TrieNode node = findNode(prefix);
        if (node == null) return results; // No words with this prefix
        // DFS from the prefix node to find all complete words
        dfs(node, new StringBuilder(prefix), results);
        return results;
    }

    private void dfs(TrieNode node, StringBuilder current, List<String> results) {
        if (node.isEndOfWord) results.add(current.toString()); // Found a complete word
        for (int i = 0; i < 26; i++) {
            if (node.children[i] != null) {
                current.append((char) ('a' + i));  // Append this character
                dfs(node.children[i], current, results);
                current.deleteCharAt(current.length() - 1); // Backtrack
            }
        }
    }

    // Delete a word from the trie — only delete nodes not shared by other words
    public void delete(String word) {
        deleteRec(root, word, 0);
    }

    private boolean deleteRec(TrieNode node, String word, int depth) {
        if (depth == word.length()) {
            if (!node.isEndOfWord) return false; // Word doesn't exist
            node.isEndOfWord = false; // Unmark the word end
            return isNodeEmpty(node); // Return true if this node can be deleted
        }
        int index = word.charAt(depth) - 'a';
        if (node.children[index] == null) return false; // Word doesn't exist
        boolean shouldDelete = deleteRec(node.children[index], word, depth + 1);
        if (shouldDelete) {
            node.children[index] = null; // Delete the child node
            return isNodeEmpty(node) && !node.isEndOfWord; // Delete this node too?
        }
        return false;
    }

    private boolean isNodeEmpty(TrieNode node) {
        for (TrieNode child : node.children) if (child != null) return false;
        return true;
    }
}
```

---

## 3. Graph

A **graph** is a collection of **vertices** (nodes) connected by **edges**. Unlike a tree, a graph can have cycles, disconnected components, and edges in any direction.

**Types of graphs:**
- **Directed:** Edges have a direction (A→B does not imply B→A). Examples: web links, dependency graphs.
- **Undirected:** Edges go both ways. Examples: social network friendships, road maps.
- **Weighted:** Edges have costs/distances. Examples: flight distances, network latency.
- **Unweighted:** All edges are equal.
- **Cyclic / Acyclic (DAG):** Whether the graph has cycles. A DAG (Directed Acyclic Graph) is used for task scheduling, build dependencies.

### Representation 1: Adjacency List (preferred for sparse graphs)

```java
import java.util.*;

public class Graph {
    private int vertices;
    private Map<Integer, List<Integer>> adjList; // Vertex → list of neighbors
    private boolean directed;

    public Graph(int vertices, boolean directed) {
        this.vertices = vertices;
        this.directed = directed;
        adjList = new HashMap<>();
        for (int i = 0; i < vertices; i++) {
            adjList.put(i, new ArrayList<>());
        }
    }

    // Add an edge between u and v
    public void addEdge(int u, int v) {
        adjList.get(u).add(v);
        if (!directed) adjList.get(v).add(u); // Undirected: add both directions
    }

    // BFS — Breadth-First Search: explore level by level using a Queue
    // O(V + E): visits every vertex and every edge once
    // Use for: shortest path in unweighted graphs, level-order traversal
    public void bfs(int start) {
        boolean[] visited = new boolean[vertices];
        Queue<Integer> queue = new LinkedList<>();

        visited[start] = true;
        queue.offer(start);

        while (!queue.isEmpty()) {
            int vertex = queue.poll();
            System.out.print(vertex + " ");

            for (int neighbor : adjList.get(vertex)) {
                if (!visited[neighbor]) {
                    visited[neighbor] = true; // Mark BEFORE enqueuing to avoid duplicates
                    queue.offer(neighbor);
                }
            }
        }
    }

    // DFS — Depth-First Search: explore as deep as possible before backtracking
    // O(V + E): visits every vertex and every edge once
    // Use for: cycle detection, topological sort, connected components
    public void dfs(int start) {
        boolean[] visited = new boolean[vertices];
        dfsRec(start, visited);
    }

    private void dfsRec(int vertex, boolean[] visited) {
        visited[vertex] = true;
        System.out.print(vertex + " ");

        for (int neighbor : adjList.get(vertex)) {
            if (!visited[neighbor]) {
                dfsRec(neighbor, visited); // Recurse deep before exploring siblings
            }
        }
    }

    // DFS iterative — same logic but uses an explicit stack (avoids stack overflow for large graphs)
    public void dfsIterative(int start) {
        boolean[] visited = new boolean[vertices];
        Deque<Integer> stack = new ArrayDeque<>();
        stack.push(start);

        while (!stack.isEmpty()) {
            int vertex = stack.pop();
            if (!visited[vertex]) {
                visited[vertex] = true;
                System.out.print(vertex + " ");
                for (int neighbor : adjList.get(vertex)) {
                    if (!visited[neighbor]) stack.push(neighbor);
                }
            }
        }
    }

    // Detect cycle in a directed graph using DFS + recursion stack (color method)
    // "recursion stack" tracks vertices in the current DFS path
    public boolean hasCycleDirected() {
        boolean[] visited = new boolean[vertices];
        boolean[] recStack = new boolean[vertices]; // True = on current DFS path
        for (int i = 0; i < vertices; i++) {
            if (!visited[i] && hasCycleRec(i, visited, recStack)) return true;
        }
        return false;
    }

    private boolean hasCycleRec(int v, boolean[] visited, boolean[] recStack) {
        visited[v] = true;
        recStack[v] = true; // Add to current path

        for (int neighbor : adjList.get(v)) {
            if (!visited[neighbor] && hasCycleRec(neighbor, visited, recStack)) return true;
            if (recStack[neighbor]) return true; // Back edge: cycle detected!
        }

        recStack[v] = false; // Remove from current path (backtrack)
        return false;
    }

    // Topological Sort (Kahn's Algorithm using BFS) — only for DAGs
    // A topological ordering: if there's an edge A→B, then A comes before B
    // Use for: build systems, course prerequisites, task scheduling
    public List<Integer> topologicalSort() {
        int[] inDegree = new int[vertices]; // Count of incoming edges for each vertex
        for (int u = 0; u < vertices; u++) {
            for (int v : adjList.get(u)) inDegree[v]++;
        }

        Queue<Integer> queue = new LinkedList<>();
        for (int i = 0; i < vertices; i++) {
            if (inDegree[i] == 0) queue.offer(i); // Start with nodes that have no prerequisites
        }

        List<Integer> order = new ArrayList<>();
        while (!queue.isEmpty()) {
            int v = queue.poll();
            order.add(v);
            for (int neighbor : adjList.get(v)) {
                inDegree[neighbor]--; // Remove this edge
                if (inDegree[neighbor] == 0) queue.offer(neighbor); // Ready to process
            }
        }

        if (order.size() != vertices) throw new RuntimeException("Graph has a cycle!");
        return order;
    }

    // Connected components in an undirected graph
    public int countComponents() {
        boolean[] visited = new boolean[vertices];
        int components = 0;
        for (int i = 0; i < vertices; i++) {
            if (!visited[i]) {
                dfsForComponents(i, visited);
                components++; // Each unvisited starting vertex is a new component
            }
        }
        return components;
    }

    private void dfsForComponents(int v, boolean[] visited) {
        visited[v] = true;
        for (int neighbor : adjList.get(v)) {
            if (!visited[neighbor]) dfsForComponents(neighbor, visited);
        }
    }
}
```

### Representation 2: Adjacency Matrix (preferred for dense graphs)

```java
// Adjacency matrix: matrix[i][j] = 1 if edge from i to j exists
// Space: O(V²) — impractical for large sparse graphs
// Edge lookup: O(1) — great when you need to check "does edge (u,v) exist?" constantly

int[][] matrix = new int[n][n];
// matrix[u][v] = 1 means there's an edge from u to v
// For weighted graphs: matrix[u][v] = weight (0 or infinity means no edge)

// When to use adjacency matrix:
// - Dense graph (many edges close to V²)
// - Need O(1) edge existence check
// When to use adjacency list:
// - Sparse graph (few edges)
// - Need to enumerate all neighbors efficiently (BFS, DFS)
```

### Union-Find (Disjoint Set Union — DSU)

Union-Find is a data structure that efficiently answers the question: "Are these two nodes connected?" and "Connect these two nodes." It's essential for Kruskal's MST algorithm and detecting cycles in undirected graphs.

```java
public class UnionFind {
    private int[] parent; // parent[i] = the parent of node i
    private int[] rank;   // rank[i] = approximate height of the tree rooted at i
    private int components; // Count of connected components

    public UnionFind(int n) {
        parent = new int[n];
        rank = new int[n];
        components = n;
        for (int i = 0; i < n; i++) parent[i] = i; // Initially, each node is its own parent
    }

    // Find the root (representative) of i's component
    // Path compression: make every node point directly to root (flattens the tree)
    public int find(int i) {
        if (parent[i] != i) {
            parent[i] = find(parent[i]); // Path compression: flatten the tree
        }
        return parent[i];
    }

    // Union: merge the components of x and y
    // Union by rank: attach smaller tree under larger tree to keep tree shallow
    public boolean union(int x, int y) {
        int rootX = find(x), rootY = find(y);
        if (rootX == rootY) return false; // Already in the same component

        if (rank[rootX] < rank[rootY]) parent[rootX] = rootY;
        else if (rank[rootX] > rank[rootY]) parent[rootY] = rootX;
        else { parent[rootY] = rootX; rank[rootX]++; }

        components--;
        return true;
    }

    public boolean connected(int x, int y) { return find(x) == find(y); }
    public int getComponents() { return components; }
}

// With both path compression AND union by rank, operations are nearly O(1)
// Technically O(α(n)) where α is the inverse Ackermann function — effectively constant

// Common use: detect cycle in undirected graph
public boolean hasCycleUndirected(int[][] edges, int n) {
    UnionFind uf = new UnionFind(n);
    for (int[] edge : edges) {
        if (!uf.union(edge[0], edge[1])) {
            return true; // Merging two nodes that were already connected = cycle
        }
    }
    return false;
}
```

---

## 4. Important Points & Best Practices

**Always override both `hashCode()` and `equals()` together.** If you override `equals()` but not `hashCode()`, two "equal" objects may map to different buckets in a HashMap — causing `get()` to fail to find a key you just `put()`. The contract: if `a.equals(b)`, then `a.hashCode() == b.hashCode()` must hold.

**Use `LinkedHashMap` when insertion order matters.** Regular `HashMap` has no order. `LinkedHashMap` maintains insertion order with only a small memory overhead. `TreeMap` maintains sorted order at O(log n) cost per operation.

**Adjacency list vs. matrix.** For most interview and real-world problems, the graph is **sparse** (far fewer edges than V²) — use an adjacency list. The matrix is only worthwhile when you have a dense graph and constantly need O(1) "does edge (u,v) exist" queries.

**Union-Find is almost always the right tool for connectivity problems.** If a problem asks "are these two nodes connected?" or "how many connected components are there?", Union-Find will typically outperform DFS/BFS in practice due to its near-constant time operations.

**Tries vs. HashMaps for string prefixes.** A `HashMap<String, Boolean>` cannot efficiently answer "does any stored word start with this prefix?" A Trie answers this in O(L). If you're building autocomplete, spell-check, or IP routing, Trie is the right choice.

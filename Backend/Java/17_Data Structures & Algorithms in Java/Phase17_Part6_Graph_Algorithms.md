# Phase 17 — Part 6: Graph Algorithms

> Graph algorithms are where data structures meet real-world problem solving. Navigation apps, social networks, package managers, circuit routing, and compiler optimizations all reduce to graph problems at their core. Master these algorithms and you'll have a toolkit that applies to an enormous range of domains.

---

## 1. Shortest Path Algorithms

### BFS for Shortest Path in Unweighted Graphs

When all edges have equal cost, BFS naturally finds the shortest path because it explores nodes layer by layer — all distance-1 nodes before distance-2 nodes, and so on.

```java
// Shortest path in an unweighted graph using BFS
// Returns the shortest path from 'start' to 'end' as a list of vertices
public List<Integer> shortestPath(Map<Integer, List<Integer>> graph, int start, int end) {
    // 'prev' maps each visited node to the node we came from
    // This lets us reconstruct the path once we reach 'end'
    Map<Integer, Integer> prev = new HashMap<>();
    Queue<Integer> queue = new LinkedList<>();

    prev.put(start, -1); // Start has no predecessor
    queue.offer(start);

    while (!queue.isEmpty()) {
        int curr = queue.poll();
        if (curr == end) break; // Found the destination — stop early

        for (int neighbor : graph.getOrDefault(curr, List.of())) {
            if (!prev.containsKey(neighbor)) { // Not yet visited
                prev.put(neighbor, curr); // Remember we came from 'curr'
                queue.offer(neighbor);
            }
        }
    }

    // Reconstruct path by walking backward from 'end' to 'start' using 'prev'
    if (!prev.containsKey(end)) return List.of(); // No path exists

    List<Integer> path = new ArrayList<>();
    for (int at = end; at != -1; at = prev.get(at)) {
        path.add(at);
    }
    Collections.reverse(path); // We built it backward, so reverse
    return path;
}
```

---

### Dijkstra's Algorithm — Shortest Path in Weighted Graphs

Dijkstra solves the single-source shortest path problem for graphs with **non-negative** edge weights. It uses a greedy approach: always extend the shortest known path next.

**Core insight:** When you pop a node from the priority queue, you have found its definitively shortest path. Because all edge weights are non-negative, any future path to that node through other nodes will be at least as long (it can only get longer, never shorter). This is why Dijkstra fails with negative weights — a future shorter path could exist.

```java
// Dijkstra's Algorithm — O((V + E) log V) with a binary heap priority queue
public int[] dijkstra(int[][] graph, int src) {
    // graph[u] = [[v1, w1], [v2, w2], ...]: adjacency list with weights
    int n = graph.length; // Assume n×n adjacency matrix for simplicity here
    int[] dist = new int[n];
    Arrays.fill(dist, Integer.MAX_VALUE); // All distances start as infinity
    dist[src] = 0; // Distance from source to itself is 0

    // Min-heap: stores [distance, vertex]
    // We process the closest unvisited vertex first (greedy)
    PriorityQueue<int[]> pq = new PriorityQueue<>(Comparator.comparingInt(a -> a[0]));
    pq.offer(new int[]{0, src});

    boolean[] visited = new boolean[n]; // Once visited, distance is finalized

    while (!pq.isEmpty()) {
        int[] curr = pq.poll();
        int d = curr[0], u = curr[1];

        // Lazy deletion: if we already found a shorter path to u, skip this entry
        // (The priority queue may have stale entries from before we found a shorter path)
        if (visited[u]) continue;
        visited[u] = true;

        // Relax all edges from u
        for (int v = 0; v < n; v++) {
            if (graph[u][v] != 0 && !visited[v]) { // Edge exists and v not visited
                int newDist = d + graph[u][v];
                if (newDist < dist[v]) {
                    dist[v] = newDist;      // Found a shorter path to v!
                    pq.offer(new int[]{newDist, v}); // Add to queue with new distance
                }
            }
        }
    }
    return dist; // dist[i] = shortest distance from src to vertex i
}

// More practical version using adjacency list representation
public int[] dijkstraList(int n, List<int[]>[] adj, int src) {
    // adj[u] = list of [v, weight] pairs
    int[] dist = new int[n];
    Arrays.fill(dist, Integer.MAX_VALUE);
    dist[src] = 0;

    PriorityQueue<int[]> pq = new PriorityQueue<>(Comparator.comparingInt(a -> a[0]));
    pq.offer(new int[]{0, src});

    while (!pq.isEmpty()) {
        int[] curr = pq.poll();
        int d = curr[0], u = curr[1];
        if (d > dist[u]) continue; // Stale entry

        for (int[] edge : adj[u]) {
            int v = edge[0], w = edge[1];
            if (dist[u] + w < dist[v]) {
                dist[v] = dist[u] + w;
                pq.offer(new int[]{dist[v], v});
            }
        }
    }
    return dist;
}
```

**Real-world use:** Google Maps, GPS navigation — finding the shortest driving route between two points. Every call to `maps.getRoute()` is essentially Dijkstra (or A* which is Dijkstra with a heuristic).

---

### Bellman-Ford — Handles Negative Weights

Dijkstra fails when edges have negative weights because its greedy assumption breaks (a future path might be shorter). Bellman-Ford handles negative weights by relaxing all edges `V-1` times.

```java
// Bellman-Ford — O(V * E)
// Can detect negative cycles (a cycle whose total weight is negative — no shortest path exists)
public int[] bellmanFord(int n, int[][] edges, int src) {
    // edges[i] = [from, to, weight]
    int[] dist = new int[n];
    Arrays.fill(dist, Integer.MAX_VALUE);
    dist[src] = 0;

    // Relax all edges V-1 times
    // Why V-1? The shortest path between any two vertices in a graph with no negative cycles
    // has at most V-1 edges. So V-1 relaxation passes guarantee we've found all shortest paths.
    for (int i = 1; i < n; i++) {
        for (int[] edge : edges) {
            int u = edge[0], v = edge[1], w = edge[2];
            if (dist[u] != Integer.MAX_VALUE && dist[u] + w < dist[v]) {
                dist[v] = dist[u] + w;
            }
        }
    }

    // One more pass: if any distance still improves, there's a negative cycle
    for (int[] edge : edges) {
        int u = edge[0], v = edge[1], w = edge[2];
        if (dist[u] != Integer.MAX_VALUE && dist[u] + w < dist[v]) {
            throw new RuntimeException("Graph contains a negative cycle — no shortest path exists");
        }
    }

    return dist;
}
```

---

### Floyd-Warshall — All Pairs Shortest Path

Floyd-Warshall finds the shortest path between **every pair** of vertices. It works by asking: "Does routing through vertex k give a shorter path from i to j?"

```java
// Floyd-Warshall — O(V³) time, O(V²) space
// Returns a 2D array where dist[i][j] = shortest path from i to j
public int[][] floydWarshall(int[][] graph) {
    int n = graph.length;
    int[][] dist = new int[n][n];

    // Initialize: copy the direct edge weights; use "infinity" for no direct edge
    for (int i = 0; i < n; i++) {
        for (int j = 0; j < n; j++) {
            if (i == j) dist[i][j] = 0;
            else if (graph[i][j] != 0) dist[i][j] = graph[i][j];
            else dist[i][j] = Integer.MAX_VALUE / 2; // Use MAX/2 to avoid overflow during addition
        }
    }

    // For each intermediate vertex k, check if going through k is shorter
    // Think of it as: "Is the path i → k → j shorter than the current best i → j?"
    for (int k = 0; k < n; k++) {           // For each possible intermediate vertex
        for (int i = 0; i < n; i++) {        // For each source
            for (int j = 0; j < n; j++) {    // For each destination
                if (dist[i][k] + dist[k][j] < dist[i][j]) {
                    dist[i][j] = dist[i][k] + dist[k][j];
                }
            }
        }
    }
    return dist;
}
```

---

## 2. Minimum Spanning Tree (MST)

A **Minimum Spanning Tree** of a connected, weighted, undirected graph is a spanning tree (connects all vertices with no cycles) whose total edge weight is minimized. There can be multiple MSTs if weights are not unique.

**Applications:** Network design (cable TV, telephone, internet), cluster analysis, approximation algorithms.

### Kruskal's Algorithm

Sort all edges by weight, then greedily add the cheapest edge that doesn't form a cycle. Uses Union-Find to detect cycles in O(α(n)) ≈ O(1).

```java
// Kruskal's Algorithm — O(E log E) dominated by the edge sorting step
public int kruskalMST(int n, int[][] edges) {
    // edges[i] = [weight, u, v]
    Arrays.sort(edges, Comparator.comparingInt(e -> e[0])); // Sort by weight ascending

    UnionFind uf = new UnionFind(n);
    int mstWeight = 0;
    int edgesAdded = 0;

    for (int[] edge : edges) {
        int w = edge[0], u = edge[1], v = edge[2];
        // Add this edge if it connects two different components (no cycle)
        if (uf.union(u, v)) {
            mstWeight += w;
            edgesAdded++;
            if (edgesAdded == n - 1) break; // MST has exactly V-1 edges
        }
        // If union returns false, u and v are already connected — adding this edge would create a cycle
    }

    return mstWeight;
}

// Union-Find class (same as shown in Part 4)
static class UnionFind {
    int[] parent, rank;
    UnionFind(int n) {
        parent = new int[n]; rank = new int[n];
        for (int i = 0; i < n; i++) parent[i] = i;
    }
    int find(int x) { return parent[x] == x ? x : (parent[x] = find(parent[x])); }
    boolean union(int x, int y) {
        int px = find(x), py = find(y);
        if (px == py) return false;
        if (rank[px] < rank[py]) parent[px] = py;
        else if (rank[px] > rank[py]) parent[py] = px;
        else { parent[py] = px; rank[px]++; }
        return true;
    }
}
```

### Prim's Algorithm

Grows the MST one vertex at a time. Start with any vertex, then repeatedly add the cheapest edge that connects a vertex in the MST to one outside it. Essentially Dijkstra but for MST.

```java
// Prim's Algorithm — O(E log V) with a priority queue
public int primMST(int n, List<int[]>[] adj) {
    // adj[u] = list of [v, weight]
    boolean[] inMST = new boolean[n]; // Tracks which vertices are in the MST
    int[] minEdge = new int[n];       // Minimum edge weight to connect each vertex to the MST
    Arrays.fill(minEdge, Integer.MAX_VALUE);
    minEdge[0] = 0; // Start from vertex 0

    // Min-heap: [edge_weight, vertex]
    PriorityQueue<int[]> pq = new PriorityQueue<>(Comparator.comparingInt(a -> a[0]));
    pq.offer(new int[]{0, 0});

    int totalWeight = 0;
    while (!pq.isEmpty()) {
        int[] curr = pq.poll();
        int w = curr[0], u = curr[1];

        if (inMST[u]) continue; // Already included
        inMST[u] = true;
        totalWeight += w;

        // Update minimum edge weights for neighbors not yet in MST
        for (int[] edge : adj[u]) {
            int v = edge[0], edgeWeight = edge[1];
            if (!inMST[v] && edgeWeight < minEdge[v]) {
                minEdge[v] = edgeWeight;
                pq.offer(new int[]{edgeWeight, v});
            }
        }
    }
    return totalWeight;
}
```

**Kruskal vs. Prim:**
- Kruskal works better for **sparse graphs** (sorting E edges is cheap when E is small).
- Prim works better for **dense graphs** (the priority queue stays small because you only track vertices, not edges).

---

## 3. Topological Sort

Topological sorting orders the vertices of a **Directed Acyclic Graph (DAG)** such that for every directed edge `u → v`, vertex `u` appears before `v` in the ordering. If the graph has a cycle, no topological order exists.

**Applications:** Build systems (compile in dependency order), course prerequisites, task scheduling.

### Kahn's Algorithm (BFS-based)

```java
// Kahn's Algorithm — O(V + E)
public List<Integer> topologicalSortKahn(int n, int[][] edges) {
    int[] inDegree = new int[n]; // Count of incoming edges for each vertex
    List<List<Integer>> adj = new ArrayList<>();
    for (int i = 0; i < n; i++) adj.add(new ArrayList<>());

    for (int[] e : edges) {
        adj.get(e[0]).add(e[1]);
        inDegree[e[1]]++;
    }

    Queue<Integer> queue = new LinkedList<>();
    // All vertices with no prerequisites can be processed first
    for (int i = 0; i < n; i++) if (inDegree[i] == 0) queue.offer(i);

    List<Integer> order = new ArrayList<>();
    while (!queue.isEmpty()) {
        int v = queue.poll();
        order.add(v);
        // "Remove" v: reduce in-degree of all its neighbors
        for (int neighbor : adj.get(v)) {
            if (--inDegree[neighbor] == 0) {
                queue.offer(neighbor); // Now this neighbor has no unresolved prerequisites
            }
        }
    }

    if (order.size() != n) throw new RuntimeException("Graph has a cycle!");
    return order;
}
```

### DFS-based Topological Sort

```java
// DFS-based: finish order reversed gives topological order
// The key insight: a vertex should appear in the topological order AFTER all its descendants
public List<Integer> topologicalSortDFS(int n, List<List<Integer>> adj) {
    boolean[] visited = new boolean[n];
    Deque<Integer> stack = new ArrayDeque<>(); // Post-order stack

    for (int i = 0; i < n; i++) {
        if (!visited[i]) dfsTopoSort(i, adj, visited, stack);
    }

    List<Integer> order = new ArrayList<>();
    while (!stack.isEmpty()) order.add(stack.pop()); // Stack reverses the post-order
    return order;
}

private void dfsTopoSort(int v, List<List<Integer>> adj, boolean[] visited, Deque<Integer> stack) {
    visited[v] = true;
    for (int neighbor : adj.get(v)) {
        if (!visited[neighbor]) dfsTopoSort(neighbor, adj, visited, stack);
    }
    stack.push(v); // Push AFTER visiting all neighbors (post-order)
}
```

---

## 4. Important Points & Best Practices

**Choosing the right shortest-path algorithm:** If edges are unweighted → BFS. If edges are weighted and non-negative → Dijkstra. If edges have negative weights but no negative cycles → Bellman-Ford. If you need all-pairs shortest paths → Floyd-Warshall.

**Dijkstra's "lazy deletion" pattern is the standard.** Rather than updating entries in the priority queue (which Java's `PriorityQueue` doesn't efficiently support for arbitrary elements), we simply add a new entry with the updated distance and skip stale entries when they're popped (`if (d > dist[u]) continue`). This makes the implementation simple and correct.

**All graph algorithms share a core pattern.** They all involve: choosing a start node, marking nodes as visited to avoid reprocessing, and deciding which adjacent node to explore next. The only difference is the *order* of exploration — BFS uses a queue (FIFO, explores by distance), DFS uses a stack/recursion (LIFO, dives deep), Dijkstra uses a priority queue (explores by cost). Recognizing this unified pattern makes all graph algorithms easier to remember.

**MST vs. Shortest Path are fundamentally different.** MST minimizes the total weight to connect all vertices; it doesn't care about the path between any specific pair. Shortest path minimizes the distance between a specific source and target(s). Don't confuse them — the MST between A and B is not necessarily the shortest path from A to B.

**Topological sort detects cycles for free.** If `order.size() != n` in Kahn's algorithm, the graph has a cycle (some vertices have non-zero in-degree that was never reduced to zero). This makes it a clean way to validate that a dependency graph is a true DAG.

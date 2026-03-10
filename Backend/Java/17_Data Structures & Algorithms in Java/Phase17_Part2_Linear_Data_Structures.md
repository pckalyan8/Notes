# Phase 17 — Part 2: Linear Data Structures

> Arrays, Linked Lists, Stacks, Queues, and Deques are the foundational building blocks of virtually every data structure and algorithm. Master these and more complex structures become approachable.

---

## 1. Dynamic Array (ArrayList from scratch)

A **dynamic array** is a resizable array that starts with a fixed capacity and doubles when full. Understanding this helps you understand Java's `ArrayList`.

**Theory:** Elements are stored in contiguous memory, enabling O(1) indexed access. When capacity is exceeded, a new array of double the size is allocated and all elements are copied — this is the O(n) resize operation that gives `add()` its amortized O(1) complexity.

```java
public class DynamicArray<T> {
    private Object[] data;
    private int size;
    private int capacity;

    public DynamicArray() {
        capacity = 4;
        data = new Object[capacity];
        size = 0;
    }

    // O(1) amortized — most of the time no resize needed
    public void add(T item) {
        if (size == capacity) {
            resize(); // O(n) but happens rarely — doubles capacity
        }
        data[size++] = item;
    }

    private void resize() {
        capacity *= 2;
        Object[] newData = new Object[capacity];
        // Copy all existing elements to the new larger array
        System.arraycopy(data, 0, newData, 0, size);
        data = newData;
    }

    // O(1) — direct index access because array memory is contiguous
    @SuppressWarnings("unchecked")
    public T get(int index) {
        if (index < 0 || index >= size) throw new IndexOutOfBoundsException("Index: " + index);
        return (T) data[index];
    }

    // O(n) — must shift all elements after the removed index left by one
    public void remove(int index) {
        if (index < 0 || index >= size) throw new IndexOutOfBoundsException("Index: " + index);
        // Shift elements left: [0,1,2,3,4] remove index 2 → [0,1,3,4]
        for (int i = index; i < size - 1; i++) {
            data[i] = data[i + 1];
        }
        data[--size] = null; // Clear the last slot to allow GC
    }

    public int size() { return size; }
    public boolean isEmpty() { return size == 0; }

    @Override
    public String toString() {
        StringBuilder sb = new StringBuilder("[");
        for (int i = 0; i < size; i++) {
            sb.append(data[i]);
            if (i < size - 1) sb.append(", ");
        }
        return sb.append("]").toString();
    }
}
```

**Key insight:** The difference between an array (`int[]`) and a dynamic array (`ArrayList`) is that arrays have a fixed length set at creation time, while dynamic arrays grow automatically. Always prefer `ArrayList` in Java unless you know the exact size upfront and performance is critical.

---

## 2. Singly Linked List

A linked list stores elements as **nodes**, where each node holds a value and a reference (pointer) to the next node. Unlike arrays, elements are NOT in contiguous memory — you must follow the chain to find any element.

**Why use a linked list over an array?**
- O(1) insertions/deletions at the head (no shifting needed)
- No pre-declared size needed
- But: O(n) random access (no index), more memory per element (stores pointer too)

```java
public class SinglyLinkedList<T> {

    // Each node holds its value and a reference to the next node
    private static class Node<T> {
        T data;
        Node<T> next;

        Node(T data) {
            this.data = data;
            this.next = null;
        }
    }

    private Node<T> head; // Points to the first node
    private int size;

    // O(1) — just update the head pointer
    public void addFirst(T data) {
        Node<T> newNode = new Node<>(data);
        newNode.next = head; // New node points to old head
        head = newNode;       // Head now points to new node
        size++;
    }

    // O(n) — must traverse to the last node
    public void addLast(T data) {
        Node<T> newNode = new Node<>(data);
        if (head == null) {
            head = newNode;
        } else {
            Node<T> current = head;
            while (current.next != null) { // Walk to end
                current = current.next;
            }
            current.next = newNode; // Attach new node at end
        }
        size++;
    }

    // O(1) — just update the head pointer
    public T removeFirst() {
        if (head == null) throw new NoSuchElementException();
        T data = head.data;
        head = head.next; // Head now points to second node; old head is GC'd
        size--;
        return data;
    }

    // O(n) — need to find the second-to-last node to update its next pointer
    public T removeLast() {
        if (head == null) throw new NoSuchElementException();
        if (head.next == null) { // Only one element
            T data = head.data;
            head = null;
            size--;
            return data;
        }
        Node<T> current = head;
        while (current.next.next != null) { // Stop at second-to-last
            current = current.next;
        }
        T data = current.next.data;
        current.next = null; // Detach last node
        size--;
        return data;
    }

    // O(n) — must traverse to find the element
    public boolean contains(T data) {
        Node<T> current = head;
        while (current != null) {
            if (current.data.equals(data)) return true;
            current = current.next;
        }
        return false;
    }

    // Reverse a linked list — classic interview question
    // O(n) time, O(1) space — done in-place by re-wiring pointers
    public void reverse() {
        Node<T> prev = null;
        Node<T> current = head;
        while (current != null) {
            Node<T> nextTemp = current.next; // Save next before overwriting
            current.next = prev;              // Reverse the pointer
            prev = current;                   // Move prev forward
            current = nextTemp;               // Move current forward
        }
        head = prev; // prev is now the last node, which becomes the new head
    }

    // Detect cycle using Floyd's Tortoise and Hare algorithm
    // O(n) time, O(1) space
    public boolean hasCycle() {
        Node<T> slow = head; // Moves one step at a time
        Node<T> fast = head; // Moves two steps at a time
        while (fast != null && fast.next != null) {
            slow = slow.next;
            fast = fast.next.next;
            if (slow == fast) return true; // They met — cycle detected
        }
        return false; // fast reached null — no cycle
    }

    public int size() { return size; }

    @Override
    public String toString() {
        StringBuilder sb = new StringBuilder("HEAD → ");
        Node<T> current = head;
        while (current != null) {
            sb.append(current.data).append(" → ");
            current = current.next;
        }
        return sb.append("NULL").toString();
    }
}
```

---

## 3. Doubly Linked List

A doubly linked list adds a `prev` pointer to each node, allowing O(1) removal from any position *when you already have a reference to the node*. This is how Java's `LinkedList` works internally.

```java
public class DoublyLinkedList<T> {

    private static class Node<T> {
        T data;
        Node<T> prev; // Points backward
        Node<T> next; // Points forward

        Node(T data) { this.data = data; }
    }

    private Node<T> head; // First node
    private Node<T> tail; // Last node — key advantage: O(1) addLast
    private int size;

    // O(1) — update both head and tail if list was empty
    public void addFirst(T data) {
        Node<T> newNode = new Node<>(data);
        if (head == null) {
            head = tail = newNode;
        } else {
            newNode.next = head;
            head.prev = newNode; // Old head now points backward to new node
            head = newNode;
        }
        size++;
    }

    // O(1) — tail pointer makes this constant time (vs O(n) in singly linked list)
    public void addLast(T data) {
        Node<T> newNode = new Node<>(data);
        if (tail == null) {
            head = tail = newNode;
        } else {
            newNode.prev = tail;
            tail.next = newNode;
            tail = newNode;
        }
        size++;
    }

    // O(1) — given a node reference, detach it by rewiring prev and next neighbors
    public void removeNode(Node<T> node) {
        if (node.prev != null) node.prev.next = node.next;
        else head = node.next; // Removing head

        if (node.next != null) node.next.prev = node.prev;
        else tail = node.prev; // Removing tail

        node.prev = node.next = null; // Help GC
        size--;
    }

    public int size() { return size; }
}
```

**Real-world use:** Java's `LinkedList` implements both `List` and `Deque`. It's a doubly linked list with sentinel nodes internally. LRU Cache implementations famously use a doubly linked list + HashMap.

---

## 4. Circular Linked List

In a circular linked list, the last node's `next` pointer points back to the head instead of null. Useful for round-robin scheduling, circular buffers, and some game logic.

```java
public class CircularLinkedList<T> {

    private static class Node<T> {
        T data;
        Node<T> next;
        Node(T data) { this.data = data; }
    }

    private Node<T> tail; // We keep a reference to TAIL (not head)
    // Because tail.next IS the head — gives us O(1) add to both ends
    private int size;

    // O(1) — new node becomes head; tail.next points to it
    public void addFirst(T data) {
        Node<T> newNode = new Node<>(data);
        if (tail == null) {
            tail = newNode;
            tail.next = tail; // Points to itself
        } else {
            newNode.next = tail.next; // New node points to old head
            tail.next = newNode;      // Tail points to new head
        }
        size++;
    }

    // O(1) — new node becomes tail
    public void addLast(T data) {
        addFirst(data);  // Add as head first
        tail = tail.next; // Then advance tail to make the new node the tail
    }

    @Override
    public String toString() {
        if (tail == null) return "[]";
        StringBuilder sb = new StringBuilder("[");
        Node<T> current = tail.next; // Start from head
        for (int i = 0; i < size; i++) {
            sb.append(current.data);
            if (i < size - 1) sb.append(" → ");
            current = current.next;
        }
        return sb.append(" → (back to head)]").toString();
    }
}
```

---

## 5. Stack

A **Stack** is a Last-In-First-Out (LIFO) data structure. Think of a stack of plates: you always add to and remove from the top.

**Key operations:** `push()` (add to top), `pop()` (remove from top), `peek()` (view top without removing).

**Applications:** Function call stack, undo/redo, parsing expressions, DFS traversal, balanced parentheses checking.

```java
// Array-based Stack — O(1) push, pop, peek
public class ArrayStack<T> {
    private Object[] data;
    private int top;
    private int capacity;

    public ArrayStack(int capacity) {
        this.capacity = capacity;
        data = new Object[capacity];
        top = -1; // -1 means empty
    }

    public void push(T item) {
        if (top == capacity - 1) throw new RuntimeException("Stack overflow");
        data[++top] = item; // Increment top, then assign
    }

    @SuppressWarnings("unchecked")
    public T pop() {
        if (isEmpty()) throw new EmptyStackException();
        T item = (T) data[top];
        data[top--] = null; // Clear reference and decrement
        return item;
    }

    @SuppressWarnings("unchecked")
    public T peek() {
        if (isEmpty()) throw new EmptyStackException();
        return (T) data[top];
    }

    public boolean isEmpty() { return top == -1; }
    public int size() { return top + 1; }
}

// ============================================================
// Classic interview problem: Balanced Parentheses
// Input: "({[]})" → true
// Input: "({[}])" → false
// ============================================================
public static boolean isBalanced(String s) {
    Deque<Character> stack = new ArrayDeque<>(); // Java's recommended stack alternative
    for (char c : s.toCharArray()) {
        if (c == '(' || c == '{' || c == '[') {
            stack.push(c); // Push opening brackets
        } else if (c == ')' || c == '}' || c == ']') {
            if (stack.isEmpty()) return false; // No matching opener
            char top = stack.pop();
            // Check if the closing bracket matches the most recent opener
            if ((c == ')' && top != '(') ||
                (c == '}' && top != '{') ||
                (c == ']' && top != '[')) {
                return false;
            }
        }
    }
    return stack.isEmpty(); // All openers must have been matched
}

// ============================================================
// Classic problem: Evaluate Reverse Polish Notation
// Input: ["2","1","+","3","*"] → (2 + 1) * 3 = 9
// ============================================================
public static int evalRPN(String[] tokens) {
    Deque<Integer> stack = new ArrayDeque<>();
    for (String token : tokens) {
        switch (token) {
            case "+" -> { int b = stack.pop(); stack.push(stack.pop() + b); }
            case "-" -> { int b = stack.pop(); stack.push(stack.pop() - b); }
            case "*" -> { int b = stack.pop(); stack.push(stack.pop() * b); }
            case "/" -> { int b = stack.pop(); stack.push(stack.pop() / b); }
            default  -> stack.push(Integer.parseInt(token));
        }
    }
    return stack.pop();
}
```

> **Important Note:** In Java, avoid using the legacy `Stack` class (extends `Vector` which is synchronized — unnecessary overhead). Prefer `Deque<T> stack = new ArrayDeque<>()` as the official Javadoc recommends.

---

## 6. Queue

A **Queue** is a First-In-First-Out (FIFO) data structure. Think of a line at a coffee shop: first person in line is first to be served.

**Key operations:** `enqueue()` (add to back), `dequeue()` (remove from front), `peek()` (view front without removing).

**Applications:** BFS traversal, task scheduling, print queues, message buffers.

```java
// Array-based Circular Queue — avoids wasted space from shifting
public class CircularQueue<T> {
    private Object[] data;
    private int front, rear, size, capacity;

    public CircularQueue(int capacity) {
        this.capacity = capacity;
        data = new Object[capacity];
        front = 0;
        rear = -1;
        size = 0;
    }

    // O(1) — use modulo to wrap around the array circularly
    public void enqueue(T item) {
        if (isFull()) throw new RuntimeException("Queue is full");
        rear = (rear + 1) % capacity; // Wrap around: 9 → 0 in a size-10 array
        data[rear] = item;
        size++;
    }

    // O(1) — just advance the front pointer
    @SuppressWarnings("unchecked")
    public T dequeue() {
        if (isEmpty()) throw new NoSuchElementException("Queue is empty");
        T item = (T) data[front];
        data[front] = null;             // Help GC
        front = (front + 1) % capacity; // Wrap around
        size--;
        return item;
    }

    @SuppressWarnings("unchecked")
    public T peek() {
        if (isEmpty()) throw new NoSuchElementException();
        return (T) data[front];
    }

    public boolean isEmpty() { return size == 0; }
    public boolean isFull() { return size == capacity; }
    public int size() { return size; }
}

// In Java, always use ArrayDeque as a queue:
// Deque<Integer> queue = new ArrayDeque<>();
// queue.offer(1);   // enqueue — returns false if full (LinkedList variant)
// queue.poll();     // dequeue — returns null if empty
// queue.peek();     // view front
```

---

## 7. Deque (Double-Ended Queue)

A **Deque** (pronounced "deck") supports insertion and removal from both ends in O(1). It generalizes both Stack and Queue.

**Applications:** Sliding window maximum/minimum, palindrome checking, browser history, undo/redo.

```java
// Deque is built-in Java — ArrayDeque is the primary implementation
Deque<Integer> deque = new ArrayDeque<>();

// Adding to both ends
deque.addFirst(10);  // Front: [10]
deque.addLast(20);   // Rear:  [10, 20]
deque.addFirst(5);   //        [5, 10, 20]
deque.addLast(30);   //        [5, 10, 20, 30]

// Removing from both ends
deque.pollFirst(); // Removes 5  → [10, 20, 30]
deque.pollLast();  // Removes 30 → [10, 20]

// ============================================================
// Classic problem: Sliding Window Maximum
// Given array nums and window size k, find max in each window
// Input: [1,3,-1,-3,5,3,6,7], k=3 → [3,3,5,5,6,7]
// ============================================================
public static int[] maxSlidingWindow(int[] nums, int k) {
    int n = nums.length;
    int[] result = new int[n - k + 1];
    // Deque stores INDICES, front always holds index of current window's max
    Deque<Integer> dq = new ArrayDeque<>();

    for (int i = 0; i < n; i++) {
        // Remove indices that are out of the current window
        while (!dq.isEmpty() && dq.peekFirst() < i - k + 1) {
            dq.pollFirst();
        }
        // Remove from back all indices whose values are less than current
        // (they can never be the max while current element is in window)
        while (!dq.isEmpty() && nums[dq.peekLast()] < nums[i]) {
            dq.pollLast();
        }
        dq.offerLast(i);
        // Window is fully formed once we've processed k elements
        if (i >= k - 1) {
            result[i - k + 1] = nums[dq.peekFirst()]; // Front is always the max
        }
    }
    return result;
    // Time: O(n) — each element added and removed from deque at most once
    // Space: O(k) — deque holds at most k indices
}
```

---

## 8. Important Points & Best Practices

**Linked list vs. array: choose wisely.** If you need frequent insertions/deletions at arbitrary positions *and* you have a node reference, linked list wins. If you need random access by index, array wins. In practice, `ArrayList` beats `LinkedList` for most use cases because of CPU cache locality (contiguous memory is faster to traverse than pointer-chasing).

**Java's recommended stack and queue classes:**

```java
// Stack — use ArrayDeque, not the legacy Stack class
Deque<Integer> stack = new ArrayDeque<>();
stack.push(1);       // Adds to front
stack.pop();         // Removes from front
stack.peek();        // Views front

// Queue — use ArrayDeque or LinkedList
Queue<Integer> queue = new ArrayDeque<>();
queue.offer(1);      // Adds to back
queue.poll();        // Removes from front
queue.peek();        // Views front

// Deque — use ArrayDeque
Deque<Integer> deque = new ArrayDeque<>();
```

**Sentinel nodes simplify edge cases.** Many linked list implementations use dummy head/tail nodes so that `head.prev` and `tail.next` are never null — this eliminates a lot of null-checking in removal/insertion logic.

**The "fast/slow pointer" technique** (Floyd's cycle detection) is one of the most versatile linked list tricks: use it for cycle detection, finding the middle of a list, and the k-th node from the end. Remember it when you see any linked list + O(1) space constraint.

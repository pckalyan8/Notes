# 4.7 I/O and NIO in Java

## What & Why

Every real program eventually has to communicate with something outside its own memory: reading a configuration file, writing a log, processing a CSV upload, transferring data over a socket. Java's I/O story has evolved through three distinct eras, each addressing the limitations of the last.

`java.io` (classic I/O, since Java 1.0) models data as a **stream** — a one-directional flow of bytes or characters. It is simple but blocking: a thread that starts reading waits until data arrives.

`java.nio` (New I/O, Java 4) introduced **channels** and **buffers** — a two-way, buffer-oriented model that enabled non-blocking I/O on a single thread, critical for high-concurrency servers.

`java.nio.file` (NIO.2, Java 7) replaced the broken `java.io.File` class with the far more capable `Path` / `Files` API and added filesystem-watching, directory traversal, and symbolic link handling.

Understanding all three layers is essential because they appear in different contexts: legacy codebases use classic streams, modern file handling uses NIO.2, and high-performance networking uses channels.

---

## 1. Classic I/O — The Stream Model

Streams in classic I/O are **one-directional** and **sequential**. You get a byte (or character) at a time, or a buffer at a time, but you cannot skip backward arbitrarily (unless the stream supports `mark`/`reset`).

The class hierarchy divides along two axes: **direction** (input vs. output) and **unit** (bytes vs. characters):

```
InputStream   — reads raw bytes
OutputStream  — writes raw bytes
Reader        — reads decoded characters (Unicode)
Writer        — writes encoded characters (Unicode)
```

Every other stream class **wraps** one of these four abstractions via the **Decorator pattern** — adding buffering, compression, encryption, or object serialisation on top.

### Reading Files — Byte Streams

```java
import java.io.*;

// Reading bytes from a file, one byte at a time — correct but slow
try (FileInputStream fis = new FileInputStream("data.bin")) {
    int bite;
    while ((bite = fis.read()) != -1) { // read() returns -1 at end-of-stream
        process(bite);
    }
}

// Wrapping with BufferedInputStream — reads in chunks (8KB by default),
// dramatically reducing the number of OS system calls
try (BufferedInputStream bis = new BufferedInputStream(new FileInputStream("data.bin"))) {
    byte[] buffer = new byte[4096];
    int bytesRead;
    while ((bytesRead = bis.read(buffer)) != -1) {
        process(buffer, 0, bytesRead);
    }
}

// Writing bytes to a file
try (FileOutputStream fos    = new FileOutputStream("output.bin");
     BufferedOutputStream bos = new BufferedOutputStream(fos)) {
    bos.write("Hello".getBytes());
    bos.write(new byte[]{1, 2, 3, 4});
    // Flush happens automatically when bos is closed by try-with-resources
}
```

### Reading Text Files — Character Streams

`FileReader` wraps `FileInputStream` and decodes bytes into characters using the platform's default charset. Always specify the charset explicitly to avoid platform-dependent behaviour:

```java
import java.io.*;
import java.nio.charset.StandardCharsets;

// Read text line by line
try (BufferedReader reader = new BufferedReader(
        new InputStreamReader(new FileInputStream("notes.txt"), StandardCharsets.UTF_8))) {

    String line;
    while ((line = reader.readLine()) != null) {
        System.out.println(line);
    }
}

// Write text
try (BufferedWriter writer = new BufferedWriter(
        new OutputStreamWriter(new FileOutputStream("output.txt"), StandardCharsets.UTF_8))) {

    writer.write("Line one");
    writer.newLine();           // platform-independent line separator
    writer.write("Line two");
}

// PrintWriter — convenient for formatted text output
try (PrintWriter pw = new PrintWriter(new FileWriter("report.txt"))) {
    pw.printf("Name: %-20s Age: %d%n", "Alice", 30);
    pw.println("Done.");
}
```

### Object Serialization

Serialization converts an in-memory object graph into a byte stream so it can be persisted or sent over a network. A class must implement `Serializable` (a marker interface) to opt in. Fields marked `transient` are excluded.

```java
import java.io.*;

public class Employee implements Serializable {
    // serialVersionUID is used to verify class compatibility during deserialization.
    // If you don't declare it, Java generates one based on the class structure —
    // and if you later add a field, deserialization of old data will fail.
    private static final long serialVersionUID = 1L;

    private String name;
    private String department;
    private transient String password; // excluded from serialization

    public Employee(String name, String department, String password) {
        this.name = name;
        this.department = department;
        this.password = password;
    }

    @Override
    public String toString() {
        return "Employee{name=" + name + ", dept=" + department + ", password=" + password + "}";
    }
}

// Serialize
try (ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("emp.ser"))) {
    oos.writeObject(new Employee("Alice", "Engineering", "secret123"));
}

// Deserialize
try (ObjectInputStream ois = new ObjectInputStream(new FileInputStream("emp.ser"))) {
    Employee emp = (Employee) ois.readObject();
    System.out.println(emp);
    // Employee{name=Alice, dept=Engineering, password=null}
    // password is null because it was transient
}
```

**Important:** Java's built-in serialization has serious security vulnerabilities (arbitrary code execution via crafted byte streams) and poor versioning semantics. For new code, prefer JSON (Jackson), Protocol Buffers, or other explicit serialization frameworks over `Serializable`.

---

## 2. NIO.2 — The Modern File API (java.nio.file, Java 7+)

NIO.2 (`java.nio.file`) is the API you should reach for whenever you work with files in modern code. The `Path` interface represents a file or directory path (immutable), and the `Files` utility class provides static methods for every common file operation.

### Path — Representing File System Locations

```java
import java.nio.file.*;

// Creating Paths
Path absolute  = Path.of("/home/user/documents/report.txt");  // Java 11+
Path relative  = Path.of("src", "main", "java", "App.java"); // joins with OS separator
Path fromUri   = Path.of(new URI("file:///home/user/file.txt"));

// Navigating and resolving
Path parent    = absolute.getParent();        // /home/user/documents
Path filename  = absolute.getFileName();      // report.txt
Path root      = absolute.getRoot();          // / (on Unix)

// resolve() — append a relative path to an existing path
Path dir       = Path.of("/home/user");
Path file      = dir.resolve("documents/report.txt"); // /home/user/documents/report.txt

// relativize() — express one path relative to another
Path base      = Path.of("/home/user");
Path target    = Path.of("/home/user/documents/report.txt");
Path relative2 = base.relativize(target);     // documents/report.txt

// normalize() — collapse . and .. segments
Path messy     = Path.of("/home/user/../user/./documents");
System.out.println(messy.normalize());        // /home/user/documents

// Convert to absolute
Path abs       = relative.toAbsolutePath();
```

### Files — Doing Things with Paths

```java
import java.nio.file.*;
import java.nio.charset.StandardCharsets;
import java.util.List;

Path file = Path.of("data.txt");

// Read entire file at once — fine for small files, bad for gigabytes
String content       = Files.readString(file, StandardCharsets.UTF_8);       // Java 11+
List<String> lines   = Files.readAllLines(file, StandardCharsets.UTF_8);
byte[] bytes         = Files.readAllBytes(file);

// Write entire file at once
Files.writeString(file, "Hello, file!", StandardCharsets.UTF_8);             // Java 11+
Files.write(file, List.of("Line 1", "Line 2"), StandardCharsets.UTF_8);
Files.write(file, "more".getBytes(), StandardOpenOption.APPEND);             // append mode

// Copy and move
Path destination = Path.of("backup.txt");
Files.copy(file, destination);                                               // fails if exists
Files.copy(file, destination, StandardCopyOption.REPLACE_EXISTING);         // overwrite
Files.move(file, destination, StandardCopyOption.ATOMIC_MOVE);              // atomic rename

// Delete
Files.delete(file);                  // throws NoSuchFileException if missing
Files.deleteIfExists(file);          // returns boolean, no exception if missing

// Existence and attributes
System.out.println(Files.exists(file));
System.out.println(Files.isReadable(file));
System.out.println(Files.isDirectory(file));
System.out.println(Files.size(file));          // bytes
System.out.println(Files.getLastModifiedTime(file));

// Creating directories
Files.createDirectory(Path.of("logs"));              // fails if parent missing
Files.createDirectories(Path.of("a/b/c/d"));         // creates all intermediate dirs
Path temp = Files.createTempFile("prefix", ".tmp");  // temp file in system temp dir
```

### Lazy Line-by-Line Reading

`Files.lines()` returns a `Stream<String>` that is **lazily populated** — it reads lines as the stream is consumed, not all at once. This is memory-efficient for large files:

```java
// Count lines containing "ERROR" in a large log file
long errorCount;
try (Stream<String> lines = Files.lines(Path.of("server.log"))) {
    errorCount = lines
        .filter(line -> line.contains("ERROR"))
        .count();
}
// The stream MUST be closed — use try-with-resources (Stream implements AutoCloseable)
System.out.println("Errors found: " + errorCount);
```

### Walking Directory Trees

```java
// Files.walk() — returns a lazy stream of paths in the tree
try (Stream<Path> tree = Files.walk(Path.of("/home/user/projects"))) {
    tree.filter(Files::isRegularFile)
        .filter(p -> p.toString().endsWith(".java"))
        .forEach(System.out::println);
}

// Files.find() — same as walk but with an integrated predicate (more efficient)
try (Stream<Path> javaFiles = Files.find(
        Path.of("."),
        Integer.MAX_VALUE,  // max depth
        (path, attrs) -> attrs.isRegularFile() && path.toString().endsWith(".java"))) {
    javaFiles.forEach(System.out::println);
}

// List only direct children of a directory (non-recursive)
try (Stream<Path> children = Files.list(Path.of("."))) {
    children.forEach(System.out::println);
}
```

### Watching for File System Changes (WatchService)

```java
import java.nio.file.*;

// Register a directory for watching
WatchService watcher = FileSystems.getDefault().newWatchService();
Path dir = Path.of("/tmp/watched");

dir.register(watcher,
    StandardWatchEventKinds.ENTRY_CREATE,
    StandardWatchEventKinds.ENTRY_MODIFY,
    StandardWatchEventKinds.ENTRY_DELETE);

// Poll for events (usually runs in a dedicated thread)
while (true) {
    WatchKey key = watcher.take();  // blocks until an event occurs
    for (WatchEvent<?> event : key.pollEvents()) {
        WatchEvent.Kind<?> kind = event.kind();
        Path filename = (Path) event.context();
        System.out.printf("Event: %s on %s%n", kind.name(), filename);
    }
    boolean valid = key.reset();  // re-arm the key for future events
    if (!valid) break;            // directory was deleted
}
```

---

## 3. NIO Channels and Buffers (java.nio, Java 4)

Classic streams are one-directional; NIO channels are bidirectional. Classic streams read/write directly; NIO channels always read/write through a `Buffer`. This model enables non-blocking I/O — a selector can monitor many channels at once and be notified when any of them has data, all without creating a thread per connection.

### Buffers

A `Buffer` is a fixed-size array with three key pointers: `capacity` (total size), `limit` (how far you can read/write), and `position` (current read/write location):

```java
import java.nio.ByteBuffer;

// Allocate a buffer on the heap
ByteBuffer buf = ByteBuffer.allocate(1024);

// Allocate a direct buffer — memory managed outside the JVM heap,
// which can be more efficient for I/O because it avoids copying data to/from Java heap
ByteBuffer direct = ByteBuffer.allocateDirect(1024);

// Writing into the buffer (position advances with each write)
buf.put((byte) 65);       // 'A'
buf.putInt(42);
buf.put("Hello".getBytes());

// flip() switches from WRITE mode to READ mode:
// limit = current position, position = 0
buf.flip();

// Now read back (position advances with each read)
byte first = buf.get();   // 65 = 'A'
int  num   = buf.getInt(); // 42

// clear() resets for writing again: position = 0, limit = capacity
buf.clear();

// compact() keeps unread data, repositions for more writing
buf.compact();
```

### FileChannel — High-Performance File I/O

```java
import java.nio.channels.*;
import java.nio.file.*;
import java.nio.ByteBuffer;

// Reading via FileChannel
try (FileChannel channel = FileChannel.open(Path.of("data.bin"), StandardOpenOption.READ)) {
    ByteBuffer buffer = ByteBuffer.allocate(4096);
    while (channel.read(buffer) != -1) {
        buffer.flip();
        while (buffer.hasRemaining()) {
            byte b = buffer.get();
            // process b
        }
        buffer.clear();
    }
}

// Fast file copy using transferTo() — uses OS-level zero-copy where available
try (FileChannel src  = FileChannel.open(Path.of("source.bin"), StandardOpenOption.READ);
     FileChannel dest = FileChannel.open(Path.of("dest.bin"),
                            StandardOpenOption.WRITE,
                            StandardOpenOption.CREATE)) {
    src.transferTo(0, src.size(), dest);
}
```

### Memory-Mapped Files

Memory-mapped files let the OS map a file's contents directly into the process's address space. You can then read and write the file as if it were a byte array, with the OS handling the actual disk I/O lazily. This is extremely efficient for large files you need random access to:

```java
try (FileChannel channel = FileChannel.open(Path.of("large.bin"),
        StandardOpenOption.READ, StandardOpenOption.WRITE)) {

    // Map a 1GB section of the file into memory
    MappedByteBuffer mmap = channel.map(
        FileChannel.MapMode.READ_WRITE, 0, channel.size());

    // Read and write as if it's an in-memory byte array
    byte firstByte = mmap.get(0);
    mmap.put(100, (byte) 42);  // write at offset 100 — goes directly to file
    mmap.force(); // ensure all changes are flushed to disk
}
```

---

## ⚡ Best Practices & Important Points

**Always use try-with-resources for any `Closeable` resource.** Unclosed streams hold OS file handles. On some systems, leaking enough handles will prevent your process from opening any new files.

**Always specify the charset explicitly.** Never rely on the platform default charset — it varies across operating systems and can make your code silently produce corrupted files when running in a different environment. `StandardCharsets.UTF_8` is almost always the right choice.

**Use `Files.readString()` / `Files.readAllLines()` for small files**, and `Files.lines()` for large ones. Reading a 2GB log file into a `List<String>` will exhaust memory; streaming it line by line with `Files.lines()` is safe.

**`Files.lines()` returns an `AutoCloseable` stream** — you must close it. Forgetting to do so leaks an open file handle. Always wrap it in try-with-resources.

**Prefer NIO.2 (`Path`/`Files`) over classic `java.io.File` in new code.** The old `File` class's error reporting is terrible — `delete()` returns a boolean instead of throwing an exception when it fails. NIO.2 throws proper, typed exceptions (`IOException`, `NoSuchFileException`, `FileAlreadyExistsException`).

**`BufferedInputStream` / `BufferedReader` wrapping is critical for performance.** An unbuffered `FileInputStream.read()` makes one OS syscall per byte. With a `BufferedInputStream` wrapping it, the same read makes one syscall per 8KB chunk. For a 10MB file, that's the difference between 10 million syscalls and ~1,300.

**Use `FileChannel.transferTo()` for copying large files.** It can use OS-level zero-copy (like Linux's `sendfile`), which avoids the overhead of copying bytes through the Java heap.

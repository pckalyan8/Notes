# 4.8 Networking in Java

## What & Why

Java was designed from day one to be a networked language — the JVM runs everywhere, and programs often need to talk to remote services, exchange data over sockets, or make HTTP requests. Java's networking support lives in `java.net` and was substantially upgraded in Java 11 with a modern, non-blocking HTTP client in `java.net.http`.

The networking stack in Java maps closely to the underlying operating system networking model. Understanding the conceptual layers first makes the APIs far easier to reason about.

---

## 1. The Conceptual Layers

When two programs communicate over a network, they use a layered protocol stack. From a Java programmer's perspective, the relevant layers are:

**TCP (Transmission Control Protocol)** — A connection-oriented, reliable, ordered byte stream. Before data flows, a three-way handshake establishes a connection. The OS guarantees that every byte arrives in order and without corruption. The cost is the overhead of connection setup and acknowledgment packets. Use TCP for anything where data integrity matters: HTTP, database connections, file transfers.

**UDP (User Datagram Protocol)** — Connectionless, unreliable. Packets (datagrams) are sent individually, may arrive out of order, and may not arrive at all. No connection setup overhead. Use UDP when speed matters more than reliability and the application can tolerate or correct losses: DNS lookups, video streaming, gaming.

**URL / HTTP** — Built on top of TCP, HTTP is a request-response protocol. Java provides `URL`, `HttpURLConnection` (old), and the modern `HttpClient` (Java 11+).

---

## 2. InetAddress — Representing IP Addresses and Hostnames

`InetAddress` represents either an IPv4 or IPv6 address (or a hostname that can be resolved to one). It is the starting point for most networking operations.

```java
import java.net.InetAddress;
import java.net.UnknownHostException;

// Look up a hostname — this performs a DNS query
InetAddress google = InetAddress.getByName("www.google.com");
System.out.println(google.getHostName());         // www.google.com
System.out.println(google.getHostAddress());      // e.g., 142.250.74.36

// Get all IP addresses for a hostname (CDN services have many)
InetAddress[] all = InetAddress.getAllByName("www.google.com");
for (InetAddress addr : all) {
    System.out.println(addr.getHostAddress());
}

// The local machine
InetAddress localhost  = InetAddress.getLocalHost();
InetAddress loopback   = InetAddress.getLoopbackAddress(); // always 127.0.0.1 / ::1

// Check reachability (ping equivalent — may require OS privileges)
boolean reachable = google.isReachable(2000); // 2 second timeout

// Create from a raw IP address string
InetAddress specific = InetAddress.getByName("93.184.216.34");
```

---

## 3. TCP Sockets — ServerSocket and Socket

This is the raw building block for any TCP-based protocol. The model has two sides:

- The **server** calls `new ServerSocket(port)`, then blocks on `accept()` waiting for a client.
- The **client** calls `new Socket(host, port)`, which triggers the TCP handshake.
- After the handshake, both sides get a `Socket` object with an `InputStream` and `OutputStream` through which they exchange bytes.

### A Minimal TCP Echo Server

```java
import java.net.*;
import java.io.*;

public class EchoServer {

    public static void main(String[] args) throws IOException {
        // Bind to port 8080. The OS will refuse new connections
        // once the backlog (default 50) is reached.
        try (ServerSocket serverSocket = new ServerSocket(8080)) {
            System.out.println("Server listening on port 8080...");

            while (true) {
                // accept() blocks until a client connects
                // Returns a Socket representing the client connection
                Socket clientSocket = serverSocket.accept();
                System.out.println("Client connected: " + clientSocket.getInetAddress());

                // In a production server you'd handle each client in a new thread
                // or submit it to a thread pool. Here we handle it inline for simplicity.
                handleClient(clientSocket);
            }
        }
    }

    private static void handleClient(Socket socket) throws IOException {
        // try-with-resources closes the socket when done
        try (socket;
             BufferedReader in  = new BufferedReader(
                                    new InputStreamReader(socket.getInputStream()));
             PrintWriter    out = new PrintWriter(socket.getOutputStream(), true)) {

            String line;
            while ((line = in.readLine()) != null) {
                System.out.println("Received: " + line);
                out.println("ECHO: " + line); // true = auto-flush on println
            }
        }
    }
}
```

### A Minimal TCP Client

```java
import java.net.*;
import java.io.*;

public class EchoClient {

    public static void main(String[] args) throws IOException {
        // new Socket() initiates the TCP handshake with the server
        try (Socket socket       = new Socket("localhost", 8080);
             PrintWriter out     = new PrintWriter(socket.getOutputStream(), true);
             BufferedReader in   = new BufferedReader(
                                     new InputStreamReader(socket.getInputStream()));
             BufferedReader stdin = new BufferedReader(new InputStreamReader(System.in))) {

            System.out.print("Enter message: ");
            String userInput;
            while ((userInput = stdin.readLine()) != null) {
                out.println(userInput);           // send to server
                System.out.println(in.readLine()); // print server's response
                System.out.print("Enter message: ");
            }
        }
    }
}
```

### Multithreaded TCP Server (Thread-per-Connection)

In the server above, only one client can be served at a time. The standard solution is to hand each client socket to a new thread (or a thread pool) and loop back to `accept()` immediately:

```java
import java.net.*;
import java.io.*;
import java.util.concurrent.*;

public class ThreadedServer {

    public static void main(String[] args) throws IOException {
        // A fixed thread pool limits resource usage
        ExecutorService executor = Executors.newFixedThreadPool(50);

        try (ServerSocket serverSocket = new ServerSocket(8080)) {
            while (true) {
                Socket clientSocket = serverSocket.accept();
                // Submit the client handling to the thread pool immediately
                // so the main thread can loop back to accept() right away
                executor.submit(() -> handleClient(clientSocket));
            }
        }
    }

    private static void handleClient(Socket socket) {
        try (socket;
             BufferedReader in  = new BufferedReader(
                                    new InputStreamReader(socket.getInputStream()));
             PrintWriter    out = new PrintWriter(socket.getOutputStream(), true)) {
            String line;
            while ((line = in.readLine()) != null) {
                out.println("ECHO: " + line);
            }
        } catch (IOException e) {
            System.err.println("Client error: " + e.getMessage());
        }
    }
}
```

---

## 4. UDP Sockets — DatagramSocket and DatagramPacket

UDP is connectionless — you wrap your data in a `DatagramPacket` and fire it off without establishing a connection first.

```java
import java.net.*;

// --- UDP Sender ---
public class UdpSender {
    public static void main(String[] args) throws Exception {
        try (DatagramSocket socket = new DatagramSocket()) {
            String message  = "Hello, UDP!";
            byte[] data     = message.getBytes();

            // Packet carries: data, length, destination address, destination port
            InetAddress dest    = InetAddress.getByName("localhost");
            DatagramPacket packet = new DatagramPacket(data, data.length, dest, 9999);
            socket.send(packet);
            System.out.println("Sent: " + message);
        }
    }
}

// --- UDP Receiver ---
public class UdpReceiver {
    public static void main(String[] args) throws Exception {
        try (DatagramSocket socket = new DatagramSocket(9999)) {  // bind to port 9999
            byte[] buffer = new byte[1024];
            DatagramPacket packet = new DatagramPacket(buffer, buffer.length);

            System.out.println("Waiting for packets...");
            socket.receive(packet);  // blocks until a packet arrives

            String received = new String(packet.getData(), 0, packet.getLength());
            System.out.println("Received from " + packet.getAddress() + ": " + received);
        }
    }
}
```

---

## 5. URL and HttpURLConnection (Legacy)

`URL` represents a Uniform Resource Locator and can open a connection to the resource it points to. `HttpURLConnection` is the older API for making HTTP requests. While functional, it's verbose and lacks modern features like async support and automatic HTTP/2.

```java
import java.net.*;
import java.io.*;

public class LegacyHttpDemo {

    public static String get(String urlString) throws IOException {
        URL url = new URL(urlString);
        HttpURLConnection connection = (HttpURLConnection) url.openConnection();

        connection.setRequestMethod("GET");
        connection.setRequestProperty("Accept", "application/json");
        connection.setConnectTimeout(5000);  // 5 seconds to connect
        connection.setReadTimeout(10000);    // 10 seconds to read

        int responseCode = connection.getResponseCode();
        if (responseCode != 200) {
            throw new IOException("HTTP " + responseCode);
        }

        // Read the response body
        try (BufferedReader reader = new BufferedReader(
                new InputStreamReader(connection.getInputStream()))) {
            StringBuilder sb = new StringBuilder();
            String line;
            while ((line = reader.readLine()) != null) {
                sb.append(line).append('\n');
            }
            return sb.toString();
        } finally {
            connection.disconnect();
        }
    }
}
```

---

## 6. Java 11 HttpClient — The Modern HTTP API

Java 11 introduced `java.net.http.HttpClient`, which supports HTTP/1.1, HTTP/2, WebSockets, synchronous and asynchronous requests, and has a clean, builder-based API.

```java
import java.net.http.*;
import java.net.URI;
import java.time.Duration;
import java.util.concurrent.CompletableFuture;

public class HttpClientDemo {

    // HttpClient instances are expensive to create — make them singletons or class-level fields
    private static final HttpClient CLIENT = HttpClient.newBuilder()
        .version(HttpClient.Version.HTTP_2)       // prefer HTTP/2, fall back to HTTP/1.1
        .connectTimeout(Duration.ofSeconds(10))
        .followRedirects(HttpClient.Redirect.NORMAL)  // follow 3xx redirects
        .build();

    // --- Synchronous GET ---
    public static String get(String url) throws Exception {
        HttpRequest request = HttpRequest.newBuilder()
            .uri(URI.create(url))
            .header("Accept", "application/json")
            .timeout(Duration.ofSeconds(30))
            .GET()  // this is actually the default; .GET() is explicit
            .build();

        HttpResponse<String> response = CLIENT.send(request, HttpResponse.BodyHandlers.ofString());

        System.out.println("Status: " + response.statusCode());
        System.out.println("Headers: " + response.headers().map());
        return response.body();
    }

    // --- POST with a JSON body ---
    public static HttpResponse<String> post(String url, String json) throws Exception {
        HttpRequest request = HttpRequest.newBuilder()
            .uri(URI.create(url))
            .header("Content-Type", "application/json")
            .header("Accept", "application/json")
            .POST(HttpRequest.BodyPublishers.ofString(json))
            .build();

        return CLIENT.send(request, HttpResponse.BodyHandlers.ofString());
    }

    // --- Asynchronous GET — returns CompletableFuture ---
    public static CompletableFuture<String> getAsync(String url) {
        HttpRequest request = HttpRequest.newBuilder()
            .uri(URI.create(url))
            .build();

        return CLIENT.sendAsync(request, HttpResponse.BodyHandlers.ofString())
            .thenApply(HttpResponse::body);
    }

    // --- Downloading a file ---
    public static void download(String url, String filePath) throws Exception {
        HttpRequest request = HttpRequest.newBuilder()
            .uri(URI.create(url))
            .build();

        HttpResponse<Path> response = CLIENT.send(
            request,
            HttpResponse.BodyHandlers.ofFile(Path.of(filePath))  // streams directly to file
        );
        System.out.println("Downloaded to: " + response.body());
    }

    public static void main(String[] args) throws Exception {
        // Synchronous
        String body = get("https://api.github.com/zen");
        System.out.println(body);

        // Async — fire multiple requests concurrently
        CompletableFuture<String> f1 = getAsync("https://httpbin.org/get");
        CompletableFuture<String> f2 = getAsync("https://httpbin.org/ip");

        // Wait for both to complete
        CompletableFuture.allOf(f1, f2).join();
        System.out.println("Response 1: " + f1.get());
        System.out.println("Response 2: " + f2.get());
    }
}
```

### Body Handlers

`BodyHandlers` is a factory for specifying how to transform the response body. Common choices:

- `ofString()` — reads the body into a `String`
- `ofFile(path)` — streams the body directly to a file
- `ofByteArray()` — reads into a `byte[]`
- `ofLines()` — returns a `Stream<String>` of response lines
- `discarding()` — ignores the body (useful when you only need status codes)
- `ofInputStream()` — raw stream for custom processing

---

## 7. Setting Socket Options

Fine-tuning socket behaviour is important for production networking code:

```java
Socket socket = new Socket();

// SO_TIMEOUT — reading will throw SocketTimeoutException after this many milliseconds
// rather than blocking forever if the remote end goes silent
socket.setSoTimeout(30_000);  // 30 seconds

// TCP_NODELAY — disable Nagle's algorithm (which buffers small writes to improve
// throughput). Set to true for interactive/low-latency applications.
socket.setTcpNoDelay(true);

// SO_KEEPALIVE — have the OS send periodic keep-alive probes on idle connections
// so dead connections are detected even without application-level heartbeats
socket.setKeepAlive(true);

// SO_REUSEADDR — allow binding to a port that is in TIME_WAIT state.
// Essential for servers that restart frequently.
ServerSocket server = new ServerSocket();
server.setReuseAddress(true);
server.bind(new InetSocketAddress(8080));
```

---

## ⚡ Best Practices & Important Points

**Always set timeouts.** A socket with no timeout will block the calling thread forever if the remote end stops responding — this is a classic cause of thread pool exhaustion in production services. Set both `connectTimeout` and `readTimeout` (or `setSoTimeout`).

**Use the Java 11 `HttpClient` for all new HTTP work.** `HttpURLConnection` works but is verbose, lacks HTTP/2, and has no native async support. `HttpClient` is cleaner, more powerful, and reuses the underlying TCP connections automatically.

**Reuse `HttpClient` instances.** Creating a new one per request defeats connection pooling. Declare it as a `static final` constant or inject it as a singleton.

**Handle network exceptions gracefully.** Networks are unreliable. `SocketTimeoutException`, `ConnectException`, and `IOException` should all be expected and handled with retry logic, circuit breakers, or user-facing error messages — not just rethrown up to the root.

**Always close sockets.** Sockets are OS resources. Use try-with-resources. In a server loop, accept each client socket in try-with-resources or ensure it's closed in a `finally` block.

**Be aware of TIME_WAIT.** After a TCP connection closes, the port stays in TIME_WAIT for ~2 minutes. For servers that restart frequently, set `SO_REUSEADDR` before binding to avoid `Address already in use` errors on restart.

# 21.2 — JSON & Serialization

> **Libraries covered:** Jackson · Gson · JSON-B · Avro · Protobuf · Kryo
>
> Serialization is the act of converting an in-memory object into a format that can be stored or transmitted — and deserialization is the reverse. Every Java application that communicates over a network, stores data to disk, or integrates with another service needs serialization. Understanding the tradeoffs between formats (human-readable JSON vs. compact binary) and libraries (reflection-based vs. annotation-driven vs. schema-first) is one of the most practically important skills in the Java ecosystem.

---

## 🌐 Jackson — The Industry Standard for JSON

Jackson is the most widely used JSON library in the Java world. It is the default JSON processor in Spring Boot, Quarkus, Micronaut, and Dropwizard. Understanding Jackson deeply is non-negotiable for any Java backend developer.

Jackson's architecture has three layers. The **Streaming API** (`JsonParser`, `JsonGenerator`) is the lowest level — it processes JSON token by token with minimal memory use, suitable for very large documents. The **Tree Model** (`JsonNode`, `ObjectNode`, `ArrayNode`) represents JSON as a navigable in-memory tree, useful when the structure is unknown at compile time. The **Data Binding layer** (`ObjectMapper`) is what most developers use daily — it maps JSON to Java objects and back using reflection and annotations.

### 21.2.1 ObjectMapper — Core Operations

`ObjectMapper` is Jackson's main entry point. It is thread-safe and expensive to create, so create one instance and reuse it throughout your application (typically as a Spring `@Bean`).

```java
import com.fasterxml.jackson.databind.*;
import com.fasterxml.jackson.databind.node.*;
import com.fasterxml.jackson.core.type.TypeReference;

// ── Serialization (Object → JSON) ────────────────────────────────────────────
ObjectMapper mapper = new ObjectMapper();
mapper.findAndRegisterModules();   // Auto-detect modules on classpath (JavaTimeModule etc.)

User user = new User(1L, "Alice", "alice@example.com", LocalDateTime.now());

// writeValueAsString: Object → JSON String
String json = mapper.writeValueAsString(user);
// {"id":1,"name":"Alice","email":"alice@example.com","createdAt":"2024-01-15T10:30:00"}

// writeValueAsBytes: Object → byte[] (more memory-efficient for network transfer)
byte[] jsonBytes = mapper.writeValueAsBytes(user);

// writeValue(File, Object): Object → File (streams directly, no intermediate String)
mapper.writeValue(new File("user.json"), user);

// Pretty-printing for debugging or human-readable output
String pretty = mapper.writerWithDefaultPrettyPrinter().writeValueAsString(user);

// ── Deserialization (JSON → Object) ──────────────────────────────────────────
// readValue: JSON String → specific type
User deserialized = mapper.readValue(json, User.class);

// Deserializing from various sources
User fromFile    = mapper.readValue(new File("user.json"), User.class);
User fromStream  = mapper.readValue(new FileInputStream("user.json"), User.class);
User fromBytes   = mapper.readValue(jsonBytes, User.class);
User fromUrl     = mapper.readValue(new URL("https://api.example.com/users/1"), User.class);

// ── Deserializing generic types (TypeReference solves type erasure) ───────────
// WRONG — generics are erased at runtime, so this produces List<LinkedHashMap>
List<User> wrong = mapper.readValue(json, List.class);  // ❌

// CORRECT — TypeReference preserves the generic type at runtime
List<User> users = mapper.readValue(json, new TypeReference<List<User>>() {});  // ✅
Map<String, User> userMap = mapper.readValue(json, new TypeReference<Map<String, User>>() {});

// ── Tree Model — navigate JSON without a corresponding Java class ─────────────
String unknown = """
    {
        "config": { "host": "localhost", "port": 8080 },
        "flags":  ["feature-x", "feature-y"]
    }
    """;

JsonNode root    = mapper.readTree(unknown);
String host      = root.get("config").get("host").asText();    // "localhost"
int    port      = root.get("config").get("port").asInt();     // 8080
JsonNode flags   = root.get("flags");
boolean hasX     = flags.toString().contains("feature-x");

// Building JSON with the Tree Model
ObjectNode response = mapper.createObjectNode();
response.put("status", "success");
response.put("code", 200);
ArrayNode items = response.putArray("items");
items.add("item1").add("item2");
String built = mapper.writeValueAsString(response);
// {"status":"success","code":200,"items":["item1","item2"]}
```

### 21.2.2 Jackson Annotations

Jackson annotations are the primary way to control how your Java objects are serialized and deserialized. Understanding the most important annotations is essential because they appear throughout Spring Boot applications.

```java
import com.fasterxml.jackson.annotation.*;

// ── @JsonProperty — rename a field in JSON ────────────────────────────────────
public class ApiResponse {
    @JsonProperty("user_id")        // JSON field is "user_id", Java field is "userId"
    private Long userId;

    @JsonProperty("full_name")
    private String fullName;

    @JsonProperty(access = JsonProperty.Access.READ_ONLY)  // Included in output, ignored in input
    private LocalDateTime createdAt;

    @JsonProperty(access = JsonProperty.Access.WRITE_ONLY) // Accepted in input, never in output
    private String password;
}

// ── @JsonIgnore — exclude a field entirely from serialization ─────────────────
public class User {
    private Long id;
    private String name;

    @JsonIgnore                     // Never serialized or deserialized
    private String passwordHash;

    @JsonIgnoreProperties({"hibernateLazyInitializer", "handler"}) // Class-level: ignore these fields
    private Address address;
}

// ── @JsonIgnoreProperties — class-level field exclusion ───────────────────────
@JsonIgnoreProperties(ignoreUnknown = true)   // Don't fail if JSON has extra fields we don't know about
// This is CRITICAL for API evolution — new fields added to the API won't break old clients
public class OrderResponse {
    private Long id;
    private String status;
    // If the API adds a "estimatedDelivery" field later, we won't fail on deserialization
}

// ── @JsonInclude — control when null/empty fields are included ─────────────────
@JsonInclude(JsonInclude.Include.NON_NULL)      // Omit null fields from output
public class SearchResult {
    private String title;
    private String description;    // Only included in output if not null
    private List<String> tags;     // Included even if empty list — use NON_EMPTY for that
}

// ── @JsonFormat — control date/number formatting ──────────────────────────────
public class Event {
    @JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss", timezone = "UTC")
    private LocalDateTime startTime;

    @JsonFormat(shape = JsonFormat.Shape.STRING)  // Serialize long as String (avoids JS precision loss)
    private Long bigId;
}

// ── @JsonCreator — specify how to deserialize (for classes without default constructors)
public class Coordinate {
    private final double lat;
    private final double lon;

    @JsonCreator
    public Coordinate(
            @JsonProperty("latitude")  double lat,   // Map "latitude" JSON field → lat parameter
            @JsonProperty("longitude") double lon) {
        this.lat = lat;
        this.lon = lon;
    }
    // Getters named lat() and lon() — different from JSON field names "latitude"/"longitude"
    public double lat() { return lat; }
    public double lon() { return lon; }
}

// ── @JsonTypeInfo and @JsonSubTypes — polymorphic deserialization ──────────────
// When a JSON field can contain different subtypes, Jackson needs a type discriminator
@JsonTypeInfo(use = JsonTypeInfo.Id.NAME, property = "type")
@JsonSubTypes({
    @JsonSubTypes.Type(value = EmailNotification.class,  name = "EMAIL"),
    @JsonSubTypes.Type(value = SmsNotification.class,    name = "SMS"),
    @JsonSubTypes.Type(value = PushNotification.class,   name = "PUSH")
})
public abstract class Notification {
    private String recipientId;
    private String message;
}

public class EmailNotification extends Notification {
    private String subject;
    private boolean htmlEnabled;
}

// JSON: {"type":"EMAIL","recipientId":"u1","message":"Hello","subject":"Hi","htmlEnabled":true}
// Jackson uses "type":"EMAIL" to know to deserialize as EmailNotification
```

### 21.2.3 ObjectMapper Configuration and Modules

```java
// ── Configuring ObjectMapper as a Spring bean ─────────────────────────────────
@Configuration
public class JacksonConfig {

    @Bean
    public ObjectMapper objectMapper() {
        return JsonMapper.builder()
            // Serialization features
            .disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS)  // ISO-8601 dates, not epoch millis
            .disable(SerializationFeature.FAIL_ON_EMPTY_BEANS)        // Don't fail on empty objects
            .enable(SerializationFeature.INDENT_OUTPUT)               // Pretty print (only in dev)

            // Deserialization features
            .disable(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES)  // Critical for API evolution
            .enable(DeserializationFeature.READ_UNKNOWN_ENUM_VALUES_AS_NULL) // Graceful enum handling

            // Register modules
            .addModule(new JavaTimeModule())       // Support java.time (LocalDate, ZonedDateTime, etc.)
            .addModule(new Jdk8Module())           // Support Optional, OptionalInt, etc.
            // .addModule(new ParameterNamesModule()) // Support for records and constructor binding

            // Naming strategy: automatically convert camelCase Java to snake_case JSON
            .propertyNamingStrategy(PropertyNamingStrategies.SNAKE_CASE)
            .build();
    }
}

// ── Custom serializer — control exactly how a type is written to JSON ─────────
public class MoneySerializer extends JsonSerializer<Money> {
    @Override
    public void serialize(Money money, JsonGenerator gen, SerializerProvider provider)
            throws IOException {
        gen.writeStartObject();
        gen.writeStringField("amount",   money.getAmount().toPlainString());
        gen.writeStringField("currency", money.getCurrency().getCurrencyCode());
        gen.writeEndObject();
    }
}

// ── Custom deserializer — control exactly how JSON is read into a type ─────────
public class MoneyDeserializer extends JsonDeserializer<Money> {
    @Override
    public Money deserialize(JsonParser p, DeserializationContext ctxt)
            throws IOException {
        JsonNode node = p.getCodec().readTree(p);
        BigDecimal amount   = new BigDecimal(node.get("amount").asText());
        Currency currency   = Currency.getInstance(node.get("currency").asText());
        return new Money(amount, currency);
    }
}

// Register them either via annotations or directly on the ObjectMapper:
@JsonSerialize(using = MoneySerializer.class)
@JsonDeserialize(using = MoneyDeserializer.class)
public class Money { ... }
```

---

## 📦 Gson

Gson is Google's JSON library, simpler to configure than Jackson and widely used in Android development and smaller Java projects. It has no external dependencies and works well out of the box with minimal configuration.

### 21.2.4 Gson — Core API

```java
import com.google.gson.*;
import com.google.gson.reflect.TypeToken;

// ── Basic serialization/deserialization ───────────────────────────────────────
Gson gson = new Gson();

User user = new User(1L, "Alice", "alice@example.com");
String json = gson.toJson(user);
// {"id":1,"name":"Alice","email":"alice@example.com"}

User back = gson.fromJson(json, User.class);

// Generic types — use TypeToken the same way Jackson uses TypeReference
List<User> users = gson.fromJson(jsonArray, new TypeToken<List<User>>(){}.getType());
Map<String, User> map = gson.fromJson(json, new TypeToken<Map<String, User>>(){}.getType());

// ── GsonBuilder — customising Gson behaviour ──────────────────────────────────
Gson customGson = new GsonBuilder()
    .setPrettyPrinting()                        // Human-readable output
    .serializeNulls()                           // Include null fields (excluded by default)
    .disableHtmlEscaping()                      // Don't escape < > & in JSON strings
    .setDateFormat("yyyy-MM-dd'T'HH:mm:ssZ")   // Date format for java.util.Date
    .excludeFieldsWithoutExposeAnnotation()     // Only serialize @Expose-annotated fields
    .setFieldNamingPolicy(FieldNamingPolicy.LOWER_CASE_WITH_UNDERSCORES) // camelCase → snake_case
    .create();

// ── Gson annotations ──────────────────────────────────────────────────────────
public class UserProfile {
    @SerializedName("user_id")     // Maps Java field to JSON key with different name
    private Long id;

    @SerializedName(value = "full_name", alternate = {"name", "displayName"})
    // "alternate" allows multiple JSON key names to deserialize into this field
    private String fullName;

    @Expose(serialize = true, deserialize = false)  // Only included in output
    private LocalDateTime createdAt;

    // Fields without @Expose are excluded when excludeFieldsWithoutExposeAnnotation() is set
    private String internalNote;  // Would be excluded with that GsonBuilder setting
}

// ── Custom TypeAdapter — the Gson equivalent of Jackson's custom serializer ────
public class MoneyTypeAdapter extends TypeAdapter<Money> {
    @Override
    public void write(JsonWriter out, Money money) throws IOException {
        out.beginObject();
        out.name("amount").value(money.getAmount().toPlainString());
        out.name("currency").value(money.getCurrency().getCurrencyCode());
        out.endObject();
    }

    @Override
    public Money read(JsonReader in) throws IOException {
        in.beginObject();
        BigDecimal amount = null;
        String currency = null;
        while (in.hasNext()) {
            switch (in.nextName()) {
                case "amount"   -> amount   = new BigDecimal(in.nextString());
                case "currency" -> currency = in.nextString();
                default         -> in.skipValue();
            }
        }
        in.endObject();
        return new Money(amount, Currency.getInstance(currency));
    }
}

Gson gsonWithAdapter = new GsonBuilder()
    .registerTypeAdapter(Money.class, new MoneyTypeAdapter())
    .create();
```

> **Jackson vs. Gson:** For Spring Boot applications and complex enterprise systems, Jackson is the better choice — it has richer annotation support, better polymorphic handling, excellent module ecosystem (JavaTimeModule, Kotlin module, etc.), and is the Spring default. Gson is a great choice for simpler projects, libraries that need zero transitive dependencies, and Android apps. The APIs are similar enough that switching between them is straightforward.

---

## 📋 JSON-B (Jakarta JSON Binding)

JSON-B is the official Jakarta EE/MicroProfile standard for JSON binding — the spec-level equivalent of Jackson or Gson. It is used in Jakarta EE application servers (WildFly, GlassFish) and frameworks like Quarkus and Helidon that prefer standard APIs.

```java
import jakarta.json.bind.*;
import jakarta.json.bind.annotation.*;

// ── Basic serialization/deserialization ───────────────────────────────────────
Jsonb jsonb = JsonbBuilder.create();  // Create a Jsonb instance (thread-safe, reuse it)

User user = new User(1L, "Alice", "alice@example.com");
String json = jsonb.toJson(user);
User back   = jsonb.fromJson(json, User.class);

// Generic types using reflection token
List<User> users = jsonb.fromJson(jsonArray, new ArrayList<User>(){}.getClass().getGenericSuperclass());
// Or use the runtime type approach:
List<User> users2 = jsonb.fromJson(jsonArray, new ParameterizedCollectionType<>(List.class, User.class) {});

// ── JSON-B Annotations ────────────────────────────────────────────────────────
public class OrderDto {
    @JsonbProperty("order_id")    // Rename field
    private Long id;

    @JsonbDateFormat("yyyy-MM-dd") // Date format for java.time
    private LocalDate orderDate;

    @JsonbNumberFormat("#,##0.00") // Number format
    private BigDecimal total;

    @JsonbTransient               // Exclude from serialization (like @JsonIgnore)
    private String internalNote;

    @JsonbNillable                // Include this field even when null
    private String trackingNumber;

    @JsonbCreator                 // For constructor-based deserialization
    public OrderDto(@JsonbProperty("order_id") Long id, @JsonbProperty("order_date") LocalDate date) {
        this.id = id;
        this.orderDate = date;
    }
}

// ── JsonbConfig — customising JSON-B behaviour ────────────────────────────────
JsonbConfig config = new JsonbConfig()
    .withFormatting(true)                           // Pretty print
    .withNullValues(true)                           // Include null fields
    .withPropertyNamingStrategy(PropertyNamingStrategy.LOWER_CASE_WITH_UNDERSCORES)
    .withDateFormat("yyyy-MM-dd'T'HH:mm:ss", Locale.ENGLISH);

Jsonb configuredJsonb = JsonbBuilder.create(config);
```

---

## 🔷 Apache Avro — Schema-Based Binary Serialization

Avro is a data serialization framework from Apache that uses a **schema** (defined in JSON) to describe the structure of data. The schema is the contract — both the writer and reader must agree on it. Avro is the standard choice for Kafka-based event streaming because it produces compact binary messages and schema evolution (adding/removing optional fields) is built into the design.

### 21.2.5 Avro — Schema and Code Generation

```json
// user.avsc — Avro schema file (JSON format)
{
  "type": "record",
  "name": "User",
  "namespace": "com.example.avro",
  "fields": [
    { "name": "id",       "type": "long" },
    { "name": "name",     "type": "string" },
    { "name": "email",    "type": ["null", "string"], "default": null },
    { "name": "active",   "type": "boolean", "default": true }
  ]
}
```

```java
// Option 1: Generic API — works without code generation, useful for dynamic schemas
import org.apache.avro.*;
import org.apache.avro.generic.*;

Schema schema = new Schema.Parser().parse(new File("user.avsc"));

// Create a record
GenericRecord user = new GenericData.Record(schema);
user.put("id",     42L);
user.put("name",   "Alice");
user.put("email",  "alice@example.com");
user.put("active", true);

// Serialize to bytes
DatumWriter<GenericRecord> writer = new GenericDatumWriter<>(schema);
ByteArrayOutputStream out = new ByteArrayOutputStream();
Encoder encoder = EncoderFactory.get().binaryEncoder(out, null);
writer.write(user, encoder);
encoder.flush();
byte[] avroBytes = out.toByteArray();  // Compact binary — much smaller than JSON equivalent

// Deserialize from bytes
DatumReader<GenericRecord> reader = new GenericDatumReader<>(schema);
Decoder decoder = DecoderFactory.get().binaryDecoder(avroBytes, null);
GenericRecord deserialized = reader.read(null, decoder);
System.out.println(deserialized.get("name"));  // "Alice"

// Option 2: Specific API — generate Java classes from schema (recommended for production)
// Run: mvn generate-sources with avro-maven-plugin configured
// This generates com.example.avro.User with typed getters/setters

User user = User.newBuilder()
    .setId(42L)
    .setName("Alice")
    .setEmail("alice@example.com")
    .setActive(true)
    .build();

// Serialize with schema embedded (avro container file format — includes schema in output)
File file = new File("users.avro");
DatumWriter<User> specificWriter = new SpecificDatumWriter<>(User.class);
DataFileWriter<User> fileWriter = new DataFileWriter<>(specificWriter);
fileWriter.create(user.getSchema(), file);
fileWriter.append(user);
fileWriter.close();

// Deserialize from container file
DatumReader<User> specificReader = new SpecificDatumReader<>(User.class);
DataFileReader<User> fileReader = new DataFileReader<>(file, specificReader);
while (fileReader.hasNext()) {
    User read = fileReader.next();
    System.out.println(read.getName());
}
```

---

## 🔲 Protocol Buffers (Protobuf) — Google's Binary Format

Protocol Buffers (Protobuf) is Google's language-neutral, platform-neutral mechanism for serializing structured data. It is schema-first (like Avro) but uses a different schema language (`.proto` files) and generates more idiomatic Java code. Protobuf is the standard choice for gRPC communication.

### 21.2.6 Protobuf — `.proto` Schema and Java Usage

```protobuf
// user.proto — Protobuf schema
syntax = "proto3";
package com.example.proto;
option java_package = "com.example.proto";
option java_outer_classname = "UserProtos";

message User {
    int64  id      = 1;   // Field numbers are used in binary encoding, never change them
    string name    = 2;
    string email   = 3;
    bool   active  = 4;
    repeated string roles = 5;   // repeated = list
    Address address = 6;          // Nested message
}

message Address {
    string street  = 1;
    string city    = 2;
    string country = 3;
}
```

```java
// After running protoc or the protobuf-maven-plugin, generated classes are available:
import com.example.proto.UserProtos.*;

// Build a User message
User user = User.newBuilder()
    .setId(42L)
    .setName("Alice")
    .setEmail("alice@example.com")
    .setActive(true)
    .addRoles("ADMIN")
    .addRoles("USER")
    .setAddress(Address.newBuilder()
        .setStreet("123 Main St")
        .setCity("Springfield")
        .setCountry("US")
        .build())
    .build();

// Serialize to bytes — extremely compact
byte[] bytes = user.toByteArray();
System.out.println("JSON size:   ~150 bytes");
System.out.println("Protobuf:    " + bytes.length + " bytes");  // Typically 50-70% smaller

// Serialize to OutputStream
user.writeTo(new FileOutputStream("user.pb"));

// Deserialize from bytes
User deserialized = User.parseFrom(bytes);
System.out.println(deserialized.getName());    // "Alice"
System.out.println(deserialized.getRoles(0));  // "ADMIN"
System.out.println(deserialized.getAddress().getCity()); // "Springfield"

// Partial parsing — parse only specific fields (performance optimisation for large messages)
User partial = User.parseFrom(bytes, ExtensionRegistryLite.getEmptyRegistry());

// Converting between Protobuf and JSON (useful for debugging)
import com.google.protobuf.util.JsonFormat;

String jsonString = JsonFormat.printer()
    .includingDefaultValueFields()
    .print(user);

User.Builder builder = User.newBuilder();
JsonFormat.parser().ignoringUnknownFields().merge(jsonString, builder);
User fromJson = builder.build();
```

> **Avro vs. Protobuf:** Both are schema-first binary formats that produce compact messages. Avro integrates more naturally with the Kafka/Hadoop ecosystem (via Confluent Schema Registry) and handles schema evolution very gracefully with a reader/writer schema system. Protobuf generates more ergonomic Java code with a fluent builder API and is the mandatory choice for gRPC. Choose Avro when your primary transport is Kafka; choose Protobuf when your primary transport is gRPC.

---

## ⚡ Kryo — Fast Java Object Serialization

Kryo is a fast and efficient Java binary serialization library that works without schemas or annotations — it directly serializes Java objects using reflection. It is significantly faster than Java's built-in `Serializable` mechanism and produces much smaller output. Kryo is used internally by frameworks like Apache Spark for distributed computing and Hazelcast for distributed caching.

### 21.2.7 Kryo — Core Usage

```java
import com.esotericsoftware.kryo.*;
import com.esotericsoftware.kryo.io.*;

// IMPORTANT: Kryo instances are NOT thread-safe.
// Use a ThreadLocal pool or create one per thread.
ThreadLocal<Kryo> kryoThreadLocal = ThreadLocal.withInitial(() -> {
    Kryo kryo = new Kryo();
    kryo.setRegistrationRequired(false);  // Allow serializing unregistered classes
    // For production, register classes for better performance and smaller output:
    kryo.register(User.class,    1);  // Assign a numeric ID to avoid writing class name
    kryo.register(Address.class, 2);
    kryo.register(ArrayList.class, 3);
    return kryo;
});

// ── Serializing an object to bytes ────────────────────────────────────────────
Kryo kryo = kryoThreadLocal.get();
User user = new User(42L, "Alice", "alice@example.com");

Output output = new Output(1024, -1);   // Initial buffer 1KB, unbounded growth
kryo.writeObject(output, user);
byte[] bytes = output.toBytes();
output.close();

// ── Deserializing from bytes ───────────────────────────────────────────────────
Input input = new Input(bytes);
User deserialized = kryo.readObject(input, User.class);
input.close();

// ── Serializing when the type is unknown at compile time ──────────────────────
kryo.writeClassAndObject(output, someObject);  // Writes class info + data
Object obj = kryo.readClassAndObject(input);   // Reads class info and restores correct type

// ── Null handling ─────────────────────────────────────────────────────────────
kryo.writeObjectOrNull(output, maybeNull, User.class);  // Can handle null
User result = kryo.readObjectOrNull(input, User.class);  // Returns null if it was null

// ── Performance comparison: Kryo vs Java Serialization vs JSON ────────────────
// Typical results for serializing a moderately complex object:
// Java built-in Serializable: ~100-200 µs, ~500 bytes
// Jackson JSON:               ~20-50 µs,  ~250 bytes
// Kryo (registered):          ~3-5 µs,    ~80 bytes
// Kryo (unregistered):        ~10-15 µs,  ~180 bytes (class names are included)
// Protobuf:                   ~8-12 µs,   ~60 bytes  (requires schema)
```

> **When to use Kryo:** Kryo excels in scenarios where you're serializing large volumes of Java objects within a single JVM ecosystem — distributed processing (Spark jobs), distributed caches (Hazelcast), and session replication. It is not suitable for cross-language communication (use Protobuf or Avro) or public APIs (use JSON). The lack of a schema means there is no inherent protection against incompatibility — if you change a class, deserialization of old data may break.

---

## 📊 Serialization Format Comparison

Understanding the tradeoffs between formats is as important as knowing the APIs.

| Format | Human-readable | Schema required | Cross-language | Size | Speed | Best for |
|--------|---------------|-----------------|---------------|------|-------|---------|
| JSON (Jackson) | ✅ Yes | ❌ No | ✅ Yes | Medium | Fast | REST APIs, config, general purpose |
| JSON (Gson) | ✅ Yes | ❌ No | ✅ Yes | Medium | Fast | Simple apps, Android, libraries |
| JSON-B | ✅ Yes | ❌ No | ✅ Yes | Medium | Fast | Jakarta EE, MicroProfile |
| Avro | ❌ Binary | ✅ Yes | ✅ Yes | Small | Very Fast | Kafka streaming, Hadoop |
| Protobuf | ❌ Binary | ✅ Yes | ✅ Yes | Very Small | Very Fast | gRPC, cross-language APIs |
| Kryo | ❌ Binary | ❌ No | ❌ Java only | Very Small | Extremely Fast | Internal Java-to-Java, caching, Spark |

---

## 💡 Phase 21.2 — Key Takeaways

Jackson is the workhorse of Java JSON processing and the one library you absolutely must know well — its `ObjectMapper`, annotation model, `TypeReference` pattern, and module system appear in virtually every Spring Boot application. Gson is a clean, simpler alternative well-suited to lighter-weight projects. JSON-B matters if you work in Jakarta EE or MicroProfile environments. For high-performance, schema-validated data exchange — particularly in event streaming and microservice communication — Avro (for Kafka) and Protobuf (for gRPC) are the correct choices; they produce far smaller messages and provide compile-time type safety via code generation. Kryo fills the niche of ultra-fast Java-only serialization for distributed computing and caching. The right choice depends on your context: human-readable flexibility for APIs, compact schemas for services, raw speed for internal systems.

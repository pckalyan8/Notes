# 13.5 — SOAP Web Services
### Phase 13: APIs & Web Services

---

## Understanding SOAP — History and Context

SOAP stands for **Simple Object Access Protocol**. It was developed by Microsoft in the late 1990s and became a W3C standard in 2003. For much of the 2000s and early 2010s, SOAP was *the* way enterprises built web services. Banking systems, insurance platforms, government services, ERP integrations, payment gateways — the enterprise world ran on SOAP.

REST has since displaced SOAP for most new development, but SOAP remains critically important for two reasons. First, enormous amounts of enterprise software built on SOAP are still running in production and will be for decades. If you work in banking, insurance, healthcare, government, or any large enterprise, you will almost certainly encounter SOAP APIs. Second, SOAP offers something REST doesn't: a formal, machine-readable contract (WSDL), built-in security standards (WS-Security), and built-in reliability features (WS-ReliableMessaging). For some use cases — particularly regulated industries requiring formal contracts and guaranteed delivery — these aren't just nice-to-haves.

The name "Simple" is, notoriously, ironic. SOAP is not simple. It is verbose, XML-heavy, and can be complex to work with. But it is powerful and formally defined.

---

## How SOAP Works — The Core Concepts

### SOAP Message Structure

Every SOAP communication consists of a **SOAP message** — an XML document with a specific structure:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<soapenv:Envelope 
    xmlns:soapenv="http://schemas.xmlsoap.org/soap/envelope/"
    xmlns:usr="http://example.com/user-service">

    <!-- HEADER is optional. Used for cross-cutting concerns:
         authentication tokens, transaction IDs, security info -->
    <soapenv:Header>
        <usr:AuthToken>eyJhbGciOiJIUzI1NiJ9...</usr:AuthToken>
        <usr:RequestId>550e8400-e29b-41d4-a716-446655440000</usr:RequestId>
    </soapenv:Header>

    <!-- BODY is mandatory. Contains the actual request or response payload -->
    <soapenv:Body>
        <usr:GetUserRequest>
            <usr:userId>42</usr:userId>
        </usr:GetUserRequest>
    </soapenv:Body>

</soapenv:Envelope>
```

The response looks similar:

```xml
<soapenv:Envelope xmlns:soapenv="http://schemas.xmlsoap.org/soap/envelope/"
                  xmlns:usr="http://example.com/user-service">
    <soapenv:Body>
        <usr:GetUserResponse>
            <usr:user>
                <usr:id>42</usr:id>
                <usr:name>Alice Smith</usr:name>
                <usr:email>alice@example.com</usr:email>
                <usr:role>ADMIN</usr:role>
            </usr:user>
        </usr:GetUserResponse>
    </soapenv:Body>
</soapenv:Envelope>
```

When an error occurs, the response contains a **SOAP Fault** in the body:

```xml
<soapenv:Body>
    <soapenv:Fault>
        <faultcode>soapenv:Client</faultcode>
        <faultstring>User not found: 999</faultstring>
        <detail>
            <usr:UserNotFoundFault>
                <usr:userId>999</usr:userId>
                <usr:message>No user exists with ID 999</usr:message>
            </usr:UserNotFoundFault>
        </detail>
    </soapenv:Fault>
</soapenv:Body>
```

### WSDL — The Contract

**WSDL** (Web Services Description Language) is the formal contract for a SOAP service. It's an XML document that describes:

- What operations the service provides (like methods on a class)
- What data each operation accepts and returns (the message schemas)
- Where the service is located (its endpoint URL)
- How to communicate with it (which binding style to use)

A WSDL is analogous to an OpenAPI specification — it fully describes the service so that clients can generate code to call it. When you receive a WSDL from a partner, you can generate a Java client from it without knowing anything about their implementation.

A simplified WSDL structure:

```xml
<wsdl:definitions xmlns:wsdl="http://schemas.xmlsoap.org/wsdl/"
                  xmlns:soap="http://schemas.xmlsoap.org/wsdl/soap/"
                  xmlns:xsd="http://www.w3.org/2001/XMLSchema"
                  xmlns:tns="http://example.com/user-service"
                  targetNamespace="http://example.com/user-service">

    <!-- Types: define the XML schema for request and response messages -->
    <wsdl:types>
        <xsd:schema targetNamespace="http://example.com/user-service">
            <xsd:element name="GetUserRequest">
                <xsd:complexType>
                    <xsd:sequence>
                        <xsd:element name="userId" type="xsd:long"/>
                    </xsd:sequence>
                </xsd:complexType>
            </xsd:element>
            <xsd:element name="GetUserResponse">
                <xsd:complexType>
                    <xsd:sequence>
                        <xsd:element name="user" type="tns:User"/>
                    </xsd:sequence>
                </xsd:complexType>
            </xsd:element>
            <xsd:complexType name="User">
                <xsd:sequence>
                    <xsd:element name="id" type="xsd:long"/>
                    <xsd:element name="name" type="xsd:string"/>
                    <xsd:element name="email" type="xsd:string"/>
                    <xsd:element name="role" type="xsd:string"/>
                </xsd:sequence>
            </xsd:complexType>
        </xsd:schema>
    </wsdl:types>

    <!-- Messages: named collections of parts (inputs/outputs for operations) -->
    <wsdl:message name="GetUserRequestMessage">
        <wsdl:part name="parameters" element="tns:GetUserRequest"/>
    </wsdl:message>
    <wsdl:message name="GetUserResponseMessage">
        <wsdl:part name="parameters" element="tns:GetUserResponse"/>
    </wsdl:message>

    <!-- PortType: defines the operations (like a Java interface) -->
    <wsdl:portType name="UserServicePortType">
        <wsdl:operation name="GetUser">
            <wsdl:input message="tns:GetUserRequestMessage"/>
            <wsdl:output message="tns:GetUserResponseMessage"/>
        </wsdl:operation>
    </wsdl:portType>

    <!-- Binding: how the portType is bound to a specific protocol (SOAP over HTTP here) -->
    <wsdl:binding name="UserServiceSOAPBinding" type="tns:UserServicePortType">
        <soap:binding style="document" transport="http://schemas.xmlsoap.org/soap/http"/>
        <wsdl:operation name="GetUser">
            <soap:operation soapAction="getUser"/>
            <wsdl:input><soap:body use="literal"/></wsdl:input>
            <wsdl:output><soap:body use="literal"/></wsdl:output>
        </wsdl:operation>
    </wsdl:binding>

    <!-- Service: the actual endpoint address -->
    <wsdl:service name="UserService">
        <wsdl:port name="UserServicePort" binding="tns:UserServiceSOAPBinding">
            <soap:address location="http://localhost:8080/ws/users"/>
        </wsdl:port>
    </wsdl:service>

</wsdl:definitions>
```

---

## JAX-WS — Java's SOAP API

**JAX-WS** (Jakarta XML Web Services, formerly Java API for XML Web Services) is the Java standard for building and consuming SOAP web services. It uses annotations to describe your service, and it handles all the XML serialization/deserialization automatically.

### Building a SOAP Web Service (Code-First Approach)

The code-first approach means you write your Java class first, annotate it, and JAX-WS generates the WSDL automatically.

```java
import jakarta.jws.WebService;
import jakarta.jws.WebMethod;
import jakarta.jws.WebParam;
import jakarta.jws.WebResult;
import jakarta.jws.soap.SOAPBinding;
import jakarta.xml.ws.WebFault;

// @WebService marks this class as a SOAP web service endpoint
// serviceName: the name in the generated WSDL's <service> element
// portName: the name of the port in the WSDL
// targetNamespace: the XML namespace for the generated WSDL
@WebService(
    serviceName = "UserService",
    portName = "UserServicePort",
    targetNamespace = "http://example.com/user-service"
)
// Document/Literal is the recommended and most common SOAP binding style
@SOAPBinding(style = SOAPBinding.Style.DOCUMENT, use = SOAPBinding.Use.LITERAL)
public class UserWebService {

    private final UserRepository userRepository;

    public UserWebService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    // @WebMethod exposes this method as a SOAP operation
    // operationName: what the operation is called in the WSDL
    @WebMethod(operationName = "GetUser")
    // @WebResult names the return element in the response XML
    @WebResult(name = "user")
    public UserResponse getUser(
            // @WebParam names the parameter in the request XML
            @WebParam(name = "userId") Long userId) throws UserNotFoundException {

        return userRepository.findById(userId)
            .map(u -> new UserResponse(u.getId(), u.getName(), u.getEmail(), u.getRole()))
            .orElseThrow(() -> new UserNotFoundException("User not found: " + userId));
    }

    @WebMethod(operationName = "CreateUser")
    @WebResult(name = "createdUser")
    public UserResponse createUser(
            @WebParam(name = "name") String name,
            @WebParam(name = "email") String email,
            @WebParam(name = "password") String password) throws DuplicateEmailException {

        if (userRepository.existsByEmail(email)) {
            throw new DuplicateEmailException("Email already registered: " + email);
        }

        User user = userRepository.save(new User(name, email, password));
        return new UserResponse(user.getId(), user.getName(), user.getEmail(), user.getRole());
    }

    @WebMethod(operationName = "DeleteUser")
    public void deleteUser(@WebParam(name = "userId") Long userId) throws UserNotFoundException {
        if (!userRepository.existsById(userId)) {
            throw new UserNotFoundException("User not found: " + userId);
        }
        userRepository.deleteById(userId);
    }
}
```

### Response and Fault Classes

SOAP services use plain Java classes (POJOs) for request/response data, and checked exceptions for faults:

```java
// JAXB annotations control XML serialization
import jakarta.xml.bind.annotation.XmlElement;
import jakarta.xml.bind.annotation.XmlRootElement;
import jakarta.xml.bind.annotation.XmlType;

@XmlRootElement(name = "UserResponse", namespace = "http://example.com/user-service")
@XmlType(propOrder = {"id", "name", "email", "role"})  // Controls XML element ordering
public class UserResponse {

    private Long id;
    private String name;
    private String email;
    private String role;

    // No-arg constructor is REQUIRED by JAXB for marshalling/unmarshalling
    public UserResponse() {}

    public UserResponse(Long id, String name, String email, String role) {
        this.id = id;
        this.name = name;
        this.email = email;
        this.role = role;
    }

    // @XmlElement marks this as a required element in the XML
    @XmlElement(required = true)
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }

    @XmlElement(required = true)
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }

    @XmlElement(required = true)
    public String getEmail() { return email; }
    public void setEmail(String email) { this.email = email; }

    @XmlElement(required = true)
    public String getRole() { return role; }
    public void setRole(String role) { this.role = role; }
}

// Fault class — extends Exception and is annotated with @WebFault
// This controls how Java exceptions translate to SOAP Fault elements
@WebFault(name = "UserNotFoundFault", targetNamespace = "http://example.com/user-service")
public class UserNotFoundException extends Exception {

    // The 'faultInfo' object contains the detailed fault information in the SOAP Fault body
    private final UserNotFoundFaultBean faultInfo;

    public UserNotFoundException(String message) {
        super(message);
        this.faultInfo = new UserNotFoundFaultBean(message);
    }

    public UserNotFoundFaultBean getFaultInfo() {
        return faultInfo;
    }

    // Inner class representing the fault detail in the XML
    public static class UserNotFoundFaultBean {
        private String message;

        public UserNotFoundFaultBean() {}

        public UserNotFoundFaultBean(String message) {
            this.message = message;
        }

        public String getMessage() { return message; }
        public void setMessage(String message) { this.message = message; }
    }
}
```

### Registering the Service with Spring Boot

```java
@Configuration
public class WebServiceConfig extends WsConfigurerAdapter {

    // Expose the service as a Spring bean
    @Bean
    public UserWebService userWebService(UserRepository userRepository) {
        return new UserWebService(userRepository);
    }

    // MessageDispatcherServlet is the SOAP equivalent of DispatcherServlet
    @Bean
    public ServletRegistrationBean<MessageDispatcherServlet> messageDispatcherServlet(
            ApplicationContext applicationContext) {

        MessageDispatcherServlet servlet = new MessageDispatcherServlet();
        servlet.setApplicationContext(applicationContext);
        servlet.setTransformWsdlLocations(true);

        // All SOAP endpoints will be accessible under /ws/*
        return new ServletRegistrationBean<>(servlet, "/ws/*");
    }

    // Expose the WSDL at /ws/users.wsdl
    @Bean(name = "users")
    public DefaultWsdl11Definition defaultWsdl11Definition(XsdSchema usersSchema) {
        DefaultWsdl11Definition wsdl = new DefaultWsdl11Definition();
        wsdl.setPortTypeName("UserServicePortType");
        wsdl.setLocationUri("/ws");
        wsdl.setTargetNamespace("http://example.com/user-service");
        wsdl.setSchema(usersSchema);
        return wsdl;
    }

    @Bean
    public XsdSchema usersSchema() {
        return new SimpleXsdSchema(new ClassPathResource("schemas/users.xsd"));
    }
}
```

---

## Apache CXF — The Dominant SOAP Framework

While JAX-WS is the Java standard, **Apache CXF** is by far the most widely used SOAP framework in production Java systems. CXF builds on top of JAX-WS and adds features like a powerful client-side API, WS-Security support, REST support through JAX-RS, and Spring integration.

### CXF Setup

```xml
<dependency>
    <groupId>org.apache.cxf</groupId>
    <artifactId>cxf-spring-boot-starter-jaxws</artifactId>
    <version>4.0.3</version>
</dependency>
```

### Publishing a Service with CXF

```java
@Configuration
public class CxfConfig {

    @Autowired
    private Bus cxfBus; // CXF's main configuration object — auto-configured by the starter

    @Autowired
    private UserWebService userWebService;

    @Bean
    public Endpoint userServiceEndpoint() {
        EndpointImpl endpoint = new EndpointImpl(cxfBus, userWebService);
        endpoint.publish("/users"); // Service will be at /services/users

        // Enable WS-Addressing
        // endpoint.getFeatures().add(new WSAddressingFeature());

        // Enable logging (useful for debugging SOAP messages)
        endpoint.getFeatures().add(new LoggingFeature());

        return endpoint;
    }
}
```

```properties
# CXF exposes services under /services by default
cxf.path=/services
```

Your service is now accessible at `http://localhost:8080/services/users`, and the WSDL at `http://localhost:8080/services/users?wsdl`.

---

## Consuming an External SOAP Service (WSDL-First / Contract-First)

This is the more common real-world scenario: a bank, insurer, or government agency gives you a WSDL file, and you need to call their service.

### Step 1: Generate Java Client Code from the WSDL

Use the `wsimport` tool (bundled with the JDK) or Maven plugin to generate client code:

```xml
<plugin>
    <groupId>org.codehaus.mojo</groupId>
    <artifactId>jaxws-maven-plugin</artifactId>
    <version>3.0.0</version>
    <executions>
        <execution>
            <goals>
                <goal>wsimport</goal>
            </goals>
            <configuration>
                <!-- The WSDL can be a local file or a URL -->
                <wsdlUrls>
                    <wsdlUrl>http://partner.example.com/UserService?wsdl</wsdlUrl>
                </wsdlUrls>
                <!-- Or from a local file: -->
                <!-- <wsdlFiles>
                    <wsdlFile>${project.basedir}/src/main/resources/wsdl/user-service.wsdl</wsdlFile>
                </wsdlFiles> -->
                <packageName>com.example.generated.client</packageName>
                <sourceDestDir>${project.build.directory}/generated-sources/jaxws</sourceDestDir>
            </configuration>
        </execution>
    </executions>
</plugin>
```

Running `mvn generate-sources` creates Java classes for all the types in the WSDL, plus a service class and port interface. Do not manually edit these — they are regenerated.

### Step 2: Use the Generated Client

```java
@Service
public class ExternalUserServiceClient {

    // The generated class name comes from the WSDL's <service name="..."> element
    private final UserService_Service generatedService;

    public ExternalUserServiceClient() {
        // Point to the real WSDL — or a local copy for offline development
        this.generatedService = new UserService_Service(
            new URL("http://partner.example.com/UserService?wsdl")
        );
    }

    public UserResponse getUser(Long userId) throws UserNotFoundException {
        // Get the port — this creates the underlying HTTP client
        // The port name comes from the WSDL's <port name="..."> element
        UserServicePortType port = generatedService.getUserServicePort();

        try {
            // This method name comes from the generated interface — it matches the WSDL operation
            GetUserRequest request = new GetUserRequest();
            request.setUserId(userId);

            return port.getUser(request).getUser();

        } catch (com.example.generated.client.UserNotFoundException_Exception e) {
            // The generated exception wraps the SOAP Fault — unwrap it here
            throw new UserNotFoundException("User not found: " + userId);
        }
    }
}
```

### Configuring Timeouts and Custom HTTP Settings with CXF Client

```java
@Bean
public UserServicePortType userServicePort() {
    JaxWsProxyFactoryBean factory = new JaxWsProxyFactoryBean();
    factory.setServiceClass(UserServicePortType.class);
    factory.setAddress("http://partner.example.com/UserService");

    // Add logging for debugging SOAP envelope content
    factory.getFeatures().add(new LoggingFeature());

    UserServicePortType port = (UserServicePortType) factory.create();

    // Configure HTTP client settings (timeout, connection pool, etc.)
    Client client = ClientProxy.getClient(port);
    HTTPConduit httpConduit = (HTTPConduit) client.getConduit();

    HTTPClientPolicy policy = new HTTPClientPolicy();
    policy.setConnectionTimeout(5000);        // 5 seconds to establish connection
    policy.setReceiveTimeout(30000);           // 30 seconds to receive response
    policy.setAllowChunking(false);            // Some servers don't support chunked transfer
    httpConduit.setClient(policy);

    return port;
}
```

---

## WS-Security — Authentication in SOAP

SOAP has a formal security standard called **WS-Security** (WSS4J — Web Services Security for Java). Unlike REST which uses headers (like `Authorization: Bearer ...`), WS-Security embeds security information inside the SOAP Header.

A WS-Security SOAP message with a username/password token looks like this:

```xml
<soapenv:Header>
    <wsse:Security xmlns:wsse="http://docs.oasis-open.org/wss/2004/01/oasis-200401-wss-wssecurity-secext-1.0.xsd">
        <wsse:UsernameToken>
            <wsse:Username>myuser</wsse:Username>
            <wsse:Password Type="PasswordDigest">hashed_password_here</wsse:Password>
            <wsse:Nonce>base64_encoded_nonce</wsse:Nonce>
            <wsu:Created>2025-01-01T00:00:00Z</wsu:Created>
        </wsse:UsernameToken>
    </wsse:Security>
</soapenv:Header>
```

Adding WS-Security with CXF:

```java
@Bean
public UserServicePortType securedUserServicePort() {
    JaxWsProxyFactoryBean factory = new JaxWsProxyFactoryBean();
    factory.setServiceClass(UserServicePortType.class);
    factory.setAddress("http://partner.example.com/UserService");

    // Add WSS4J interceptor for outbound security
    Map<String, Object> outProps = new HashMap<>();
    outProps.put(WSHandlerConstants.ACTION, "UsernameToken");
    outProps.put(WSHandlerConstants.USER, "myuser");
    outProps.put(WSHandlerConstants.PASSWORD_TYPE, WSConstants.PW_DIGEST);
    outProps.put(WSHandlerConstants.PW_CALLBACK_CLASS, PasswordCallbackHandler.class.getName());

    WSS4JOutInterceptor outInterceptor = new WSS4JOutInterceptor(outProps);
    factory.getOutInterceptors().add(outInterceptor);

    return (UserServicePortType) factory.create();
}

// Provides the password — keep this out of source code (use environment variables)
public class PasswordCallbackHandler implements CallbackHandler {
    @Override
    public void handle(Callback[] callbacks) throws IOException {
        for (Callback callback : callbacks) {
            if (callback instanceof WSPasswordCallback pwCallback) {
                if ("myuser".equals(pwCallback.getIdentifier())) {
                    pwCallback.setPassword(System.getenv("WS_PASSWORD"));
                }
            }
        }
    }
}
```

---

## SOAP vs REST — A Clear-Eyed Comparison

| Aspect | SOAP | REST |
|--------|------|------|
| Protocol | HTTP, SMTP, TCP, JMS (any transport) | HTTP only |
| Message format | XML only | JSON, XML, any format |
| Contract | WSDL (formal, machine-readable) | OpenAPI (widely adopted but optional) |
| Verbosity | Very verbose (XML envelope, namespaces) | Lean (JSON by default) |
| Error handling | Formal SOAP Fault | HTTP status codes (informal) |
| Security | WS-Security standard | Various (JWT, OAuth, etc.) |
| Transactions | WS-AtomicTransaction | Not built-in |
| Reliable messaging | WS-ReliableMessaging | Not built-in |
| Tooling | Strong enterprise tooling | Universal |
| Performance | Slower (XML parsing) | Faster (JSON parsing) |
| Usability | Complex to implement | Simple and intuitive |
| Industry adoption | Legacy enterprise, banking, healthcare | Default for new APIs |

The fundamental difference in philosophy is that SOAP is a **messaging protocol** that can work over many transports and provides built-in enterprise features (transactions, security, reliable messaging), while REST is an **architectural style** built purely around HTTP that trades formal features for simplicity and ubiquity.

---

## ⚠️ Important Points & Best Practices

**1. Store WSDL files locally in your project.** When consuming an external SOAP service, download their WSDL and XSD files and store them in `src/main/resources/wsdl/`. Do not point your client at the live `?wsdl` URL in production — if their server is down, your entire application fails to start.

**2. Use contract-first development for services you publish.** Write the XSD schema and WSDL first, then use tools to generate your Java interface. This produces cleaner, more interoperable services than code-first, and forces you to think about the contract deliberately before implementation.

**3. Always enable logging during development, but disable it in production.** SOAP messages can contain sensitive data (credentials, PII). The CXF `LoggingFeature` is invaluable for debugging but must not log in production without careful consideration of what's in the messages.

**4. Test SOAP services with SoapUI.** SoapUI (or its open-source equivalent) is the standard tool for testing SOAP services. It can import a WSDL, auto-generate test requests, run assertions on responses, and even mock SOAP services for testing your consuming code.

**5. Be rigorous about XML namespaces.** Most integration problems with SOAP come from namespace mismatches. If a field name is the same but the XML namespace is different, SOAP considers them different fields. Pay close attention to `targetNamespace` values.

**6. Understand that SOAP faults are the equivalent of exceptions.** When calling a SOAP service, every operation that can fail should declare a fault in the WSDL. When you catch `SomeName_Exception` in JAX-WS generated code, that's a fault the server declared it could throw. Handle each declared fault specifically, and have a catch-all for `SOAPFaultException` for undeclared errors.

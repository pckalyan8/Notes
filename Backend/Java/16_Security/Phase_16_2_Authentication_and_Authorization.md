# Phase 16.2 — Authentication & Authorization

## The Fundamental Distinction

Before diving into any implementation, the most important conceptual clarity you can have is the difference between **authentication** and **authorization** — they are frequently confused, even by experienced developers, and mixing them up leads to real security holes.

**Authentication** answers the question: *"Who are you?"* It is the process of verifying that a principal (a user, service, or device) is who they claim to be. You present credentials (a password, a certificate, a biometric) and the system either confirms or denies your claimed identity.

**Authorization** answers the question: *"What are you allowed to do?"* It is the process of deciding whether an authenticated principal has permission to perform a specific action on a specific resource. Even after you are verified as "Alice," the system still needs to decide whether Alice can view *this* invoice, modify *that* record, or access the admin dashboard.

A common architectural mistake is performing authorization based purely on authentication — assuming that because someone is logged in, they can do anything. The correct model is: authenticate first, then for every action and resource, check authorization independently.

---

## Session-Based Authentication

Session-based authentication is the traditional model, dominant since the early days of web development, and still the right choice for server-rendered applications.

Here is how it works conceptually. When a user submits their username and password, the server verifies the credentials against the database (comparing the submitted password against the stored BCrypt hash). On success, the server creates a **session** — a small record of the user's identity and state — stored server-side (in memory, in a database like Redis, or in a clustered session store). The server sends back a **session cookie** containing only the session ID (a random, unpredictable string like `SESSION=a3f8b2c1...`). The browser stores this cookie and automatically sends it with every subsequent request. The server looks up the session by its ID, retrieves the user's data, and knows who is making the request.

```java
// Spring Security 6 — form-login with session-based authentication
@Configuration
@EnableWebSecurity
public class SessionSecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                // Public endpoints — anyone can access
                .requestMatchers("/", "/login", "/register", "/public/**").permitAll()
                // Admin section — requires ROLE_ADMIN
                .requestMatchers("/admin/**").hasRole("ADMIN")
                // Everything else requires any authenticated user
                .anyRequest().authenticated()
            )
            .formLogin(form -> form
                .loginPage("/login")                    // Custom login page URL
                .loginProcessingUrl("/login")           // POST endpoint that processes credentials
                .defaultSuccessUrl("/dashboard", true)  // Where to go after successful login
                .failureUrl("/login?error=true")        // Where to go after failed login
                .permitAll()
            )
            .logout(logout -> logout
                .logoutUrl("/logout")
                .logoutSuccessUrl("/login?logout=true")
                .invalidateHttpSession(true)            // Destroy the server-side session
                .deleteCookies("JSESSIONID")            // Remove the session cookie
            )
            .sessionManagement(session -> session
                // Only allow one active session per user at a time
                // If they log in from a second device, the first session is invalidated
                .maximumSessions(1)
                .maxSessionsPreventsLogin(false)        // false = newer login kicks out older one
            );

        return http.build();
    }

    // Spring Security needs this to load user details from your database
    @Bean
    public UserDetailsService userDetailsService(UserRepository userRepository) {
        return username -> userRepository.findByUsername(username)
            .map(user -> org.springframework.security.core.userdetails.User.builder()
                .username(user.getUsername())
                .password(user.getPasswordHash())      // Already BCrypt-hashed
                .roles(user.getRole().name())
                .build()
            )
            .orElseThrow(() -> new UsernameNotFoundException("User not found: " + username));
    }
}
```

**Session fixation** is a critical attack on session-based auth: an attacker gets a pre-authentication session ID, tricks the user into authenticating with it, and then uses that same ID to hijack the session. Spring Security prevents this by default — it creates a new session ID immediately after a successful login:

```java
// Explicitly configure session fixation protection (it's on by default, this is for illustration)
.sessionManagement(session -> session
    .sessionFixation().changeSessionId()   // Create a new session ID after login (default)
    // Never use .sessionFixation().none() — that leaves you vulnerable
)
```

---

## Token-Based Authentication — JWT

**JSON Web Tokens (JWT)** are the dominant authentication mechanism for REST APIs, especially those consumed by Single Page Applications or mobile clients. Unlike session-based auth, JWTs are **stateless** — the server does not store anything. The token itself carries all the information needed to identify the user and their permissions.

### JWT Structure — Header.Payload.Signature

A JWT is a Base64URL-encoded string with three dot-separated parts:

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9
.eyJzdWIiOiJ1c2VyXzEyMyIsInVzZXJuYW1lIjoiYWxpY2UiLCJyb2xlIjoiVVNFUiIsImlhdCI6MTcwOTU3NjAwMCwiZXhwIjoxNzA5NTc5NjAwfQ
.SfKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
```

**Header** — Specifies the type (`JWT`) and the signing algorithm (`HS256` for HMAC-SHA256, `RS256` for RSA-SHA256):
```json
{ "alg": "HS256", "typ": "JWT" }
```

**Payload (Claims)** — Contains the data. Standard registered claims include `sub` (subject — the user identifier), `iat` (issued-at timestamp), `exp` (expiration timestamp), and `iss` (issuer). You add your own custom claims (roles, user ID, email):
```json
{
  "sub": "user_123",
  "username": "alice",
  "role": "USER",
  "iat": 1709576000,
  "exp": 1709579600
}
```

**Signature** — The server uses a secret key to sign the header + payload with the chosen algorithm. Anyone can *read* a JWT (it is Base64-encoded, not encrypted), but no one can *forge* or *tamper with* it without knowing the secret key. If any part of the token is modified, signature verification fails and the server rejects the token.

This is a critical point to understand deeply: **the JWT payload is NOT encrypted by default**. It is only encoded and signed. Never put sensitive information (passwords, credit card numbers, full SSNs) in a JWT payload. Put only what you need for authorization decisions — user ID, roles, and expiration.

### Implementing JWT with JJWT

```xml
<!-- pom.xml -->
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-api</artifactId>
    <version>0.12.5</version>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-impl</artifactId>
    <version>0.12.5</version>
    <scope>runtime</scope>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-jackson</artifactId>
    <version>0.12.5</version>
    <scope>runtime</scope>
</dependency>
```

```java
@Service
public class JwtTokenService {

    // The secret key must be:
    // - At minimum 256 bits (32 bytes) for HS256
    // - Stored securely (environment variable or vault, never hardcoded)
    // - Different per environment (dev/staging/production have different keys)
    @Value("${jwt.secret}")
    private String jwtSecret;

    @Value("${jwt.access-token.expiry-minutes:15}")
    private int accessTokenExpiryMinutes;

    @Value("${jwt.refresh-token.expiry-days:7}")
    private int refreshTokenExpiryDays;

    private SecretKey getSigningKey() {
        // Convert the base64-encoded secret from config to a SecretKey
        byte[] keyBytes = Decoders.BASE64.decode(jwtSecret);
        return Keys.hmacShaKeyFor(keyBytes);
    }

    // Generate a short-lived access token (15 minutes is typical)
    // The short expiry limits the damage if a token is stolen — it expires quickly
    public String generateAccessToken(UserDetails userDetails) {
        Map<String, Object> claims = new HashMap<>();
        claims.put("role", userDetails.getAuthorities().stream()
            .map(GrantedAuthority::getAuthority)
            .collect(Collectors.toList()));

        return Jwts.builder()
            .claims(claims)
            .subject(userDetails.getUsername())
            .issuer("myapp.com")                          // Identify who issued the token
            .issuedAt(new Date())
            .expiration(new Date(System.currentTimeMillis()
                + accessTokenExpiryMinutes * 60 * 1000))
            .signWith(getSigningKey(), Jwts.SIG.HS256)    // Sign with HMAC-SHA256
            .compact();
    }

    // Generate a longer-lived refresh token (7 days)
    // Refresh tokens are used ONLY to obtain new access tokens — nothing else
    // They should be stored in an HttpOnly cookie (inaccessible to JavaScript)
    // to protect against XSS-based theft
    public String generateRefreshToken(UserDetails userDetails) {
        return Jwts.builder()
            .subject(userDetails.getUsername())
            .issuer("myapp.com")
            .issuedAt(new Date())
            .expiration(new Date(System.currentTimeMillis()
                + refreshTokenExpiryDays * 24 * 60 * 60 * 1000L))
            .claim("type", "refresh")                     // Mark as a refresh token
            .signWith(getSigningKey(), Jwts.SIG.HS256)
            .compact();
    }

    // Validate and extract claims from a token
    // Throws JwtException if the token is expired, tampered with, or invalid
    public Claims extractAllClaims(String token) {
        return Jwts.parser()
            .verifyWith(getSigningKey())          // Verifies the signature
            .requireIssuer("myapp.com")           // Verifies the issuer claim
            .build()
            .parseSignedClaims(token)             // Throws if expired or invalid
            .getPayload();
    }

    public String extractUsername(String token) {
        return extractAllClaims(token).getSubject();
    }

    public boolean isTokenValid(String token) {
        try {
            Claims claims = extractAllClaims(token);
            // Ensure this is an access token, not a refresh token
            return !"refresh".equals(claims.get("type"));
        } catch (JwtException ex) {
            return false;
        }
    }
}
```

### JWT Authentication Filter in Spring Security

```java
// This filter runs on every request, extracts the JWT from the Authorization header,
// validates it, and sets the authenticated principal in the SecurityContext
@Component
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    private final JwtTokenService jwtTokenService;
    private final UserDetailsService userDetailsService;

    @Override
    protected void doFilterInternal(HttpServletRequest request,
            HttpServletResponse response, FilterChain filterChain)
            throws ServletException, IOException {

        // JWT is sent in the Authorization header as: "Bearer <token>"
        String authHeader = request.getHeader("Authorization");

        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            // No JWT present — continue the filter chain without authentication
            // Spring Security will decide if the endpoint requires auth
            filterChain.doFilter(request, response);
            return;
        }

        String token = authHeader.substring(7);  // Remove "Bearer " prefix

        try {
            if (jwtTokenService.isTokenValid(token)) {
                String username = jwtTokenService.extractUsername(token);
                UserDetails userDetails = userDetailsService.loadUserByUsername(username);

                // Create an authentication token and put it in the SecurityContext
                // This tells Spring Security "this request is authenticated as this user"
                UsernamePasswordAuthenticationToken authToken =
                    new UsernamePasswordAuthenticationToken(
                        userDetails,
                        null,                        // No credentials needed (we have the token)
                        userDetails.getAuthorities()
                    );
                authToken.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));
                SecurityContextHolder.getContext().setAuthentication(authToken);
            }
        } catch (JwtException ex) {
            // Invalid token — do NOT authenticate, just log and continue
            // The SecurityContext remains empty, so Spring Security will return 401
            log.warn("Invalid JWT token: {}", ex.getMessage());
        }

        filterChain.doFilter(request, response);
    }
}
```

### Access Tokens and Refresh Tokens

A crucial design decision is token lifetime. If access tokens never expire, a stolen token is permanently useful to an attacker. But if they expire after 5 minutes, users are constantly forced to re-authenticate.

The solution is a two-token architecture: a **short-lived access token** (5–15 minutes) used for every API call, and a **long-lived refresh token** (7–30 days) used only to obtain a new access token when the old one expires. The refresh token is stored in an **HttpOnly cookie** (inaccessible to JavaScript, protecting against XSS), while the access token is kept in memory.

```java
// Endpoint for refreshing an access token
@RestController
@RequestMapping("/auth")
public class AuthController {

    @PostMapping("/refresh")
    public ResponseEntity<TokenResponse> refreshToken(
            @CookieValue(name = "refreshToken", required = false) String refreshToken) {

        if (refreshToken == null) {
            return ResponseEntity.status(HttpStatus.UNAUTHORIZED)
                .body(new TokenResponse(null, "No refresh token provided"));
        }

        try {
            Claims claims = jwtTokenService.extractAllClaims(refreshToken);

            // Verify this is actually a refresh token
            if (!"refresh".equals(claims.get("type"))) {
                return ResponseEntity.status(HttpStatus.UNAUTHORIZED).build();
            }

            // Optionally: look up the refresh token in the database to allow revocation
            // If it's been revoked (user logged out), reject it
            if (refreshTokenRepository.isRevoked(refreshToken)) {
                return ResponseEntity.status(HttpStatus.UNAUTHORIZED).build();
            }

            UserDetails userDetails = userDetailsService.loadUserByUsername(claims.getSubject());
            String newAccessToken = jwtTokenService.generateAccessToken(userDetails);

            return ResponseEntity.ok(new TokenResponse(newAccessToken, null));

        } catch (JwtException ex) {
            return ResponseEntity.status(HttpStatus.UNAUTHORIZED)
                .body(new TokenResponse(null, "Invalid or expired refresh token"));
        }
    }
}
```

---

## OAuth 2.0 — Delegated Authorization

**OAuth 2.0** is an *authorization framework* that allows a user to grant a third-party application limited access to their resources on another service — without sharing their password. When you click "Sign in with Google" or "Connect with GitHub," you are using OAuth 2.0.

The key innovation is the concept of scopes: the user grants only specific permissions (read email, view profile) rather than full account access. If the third-party app is compromised, the attacker only gets what the scope allowed, not the user's full credentials.

### The Four Main Flows

**Authorization Code Flow** is the secure, standard flow for web applications where a backend server is involved. It is the flow you should use for any server-side application:

```
User → Clicks "Login with Google"
Browser → Redirected to Google's authorization endpoint:
  https://accounts.google.com/o/oauth2/auth
    ?client_id=YOUR_CLIENT_ID
    &redirect_uri=https://yourapp.com/callback
    &response_type=code
    &scope=openid email profile
    &state=RANDOM_CSRF_TOKEN        ← Prevents CSRF on the OAuth flow itself

User → Approves permissions on Google's page

Google → Redirects back to your app with a short-lived authorization code:
  https://yourapp.com/callback?code=AUTH_CODE&state=CSRF_TOKEN

Your Server → Exchanges the code for tokens (server-to-server, not in the browser):
  POST https://oauth2.googleapis.com/token
  grant_type=authorization_code&code=AUTH_CODE&redirect_uri=...&client_secret=...

Google → Returns access_token, refresh_token, id_token

Your Server → Uses access_token to call Google APIs on behalf of the user
```

Notice the critical security property: the `client_secret` is only ever used in the server-to-server token exchange — it is never exposed to the browser. This is why **Implicit Flow** (which skips this step and returns tokens directly to the browser) is deprecated in OAuth 2.1 — it exposed tokens in browser history and referrer headers.

**Client Credentials Flow** is used for machine-to-machine communication where no user is involved. A microservice authenticates itself to another microservice:

```java
// Service A obtaining a token to call Service B
// This is used in microservice architectures where services authenticate each other
RestTemplate restTemplate = new RestTemplate();

MultiValueMap<String, String> params = new LinkedMultiValueMap<>();
params.add("grant_type", "client_credentials");
params.add("client_id", "service-a-client-id");
params.add("client_secret", "service-a-secret");
params.add("scope", "order:read order:write");

TokenResponse tokenResponse = restTemplate.postForObject(
    "https://auth-server.example.com/token",
    params,
    TokenResponse.class
);
// Use tokenResponse.getAccessToken() in requests to Service B
```

**Authorization Code with PKCE (Proof Key for Code Exchange)** is the secure flow for Single Page Applications and mobile apps that cannot safely store a `client_secret` (since their code is public). PKCE replaces the client secret with a dynamically generated code challenge.

```
SPA generates a random code_verifier (43-128 chars)
SPA computes: code_challenge = BASE64URL(SHA256(code_verifier))

SPA → Authorization request includes code_challenge (not the secret)
Auth Server → Returns authorization code
SPA → Token request includes code_verifier (the original, unhashed value)
Auth Server → Verifies SHA256(code_verifier) == code_challenge, then returns tokens
```

An attacker who intercepts the authorization code cannot exchange it for a token because they do not know the `code_verifier` — it never left the user's device.

### Spring Security OAuth 2.0 Client

```java
// Spring Security handles the OAuth 2.0 flow automatically with this configuration
@Configuration
@EnableWebSecurity
public class OAuth2Config {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/", "/error").permitAll()
                .anyRequest().authenticated()
            )
            .oauth2Login(oauth2 -> oauth2
                .loginPage("/login")
                // After successful login, Spring calls your custom handler
                .successHandler((request, response, authentication) -> {
                    OAuth2AuthenticationToken token = (OAuth2AuthenticationToken) authentication;
                    // Extract user info from the OAuth2 provider
                    String email = token.getPrincipal().getAttribute("email");
                    String name = token.getPrincipal().getAttribute("name");
                    // Register/update user in your database
                    userService.upsertOAuthUser(email, name, token.getAuthorizedClientRegistrationId());
                    response.sendRedirect("/dashboard");
                })
            );
        return http.build();
    }
}
```

```yaml
# application.yml — register OAuth2 providers
spring:
  security:
    oauth2:
      client:
        registration:
          google:
            client-id: ${GOOGLE_CLIENT_ID}           # From Google Cloud Console
            client-secret: ${GOOGLE_CLIENT_SECRET}
            scope: openid, email, profile
          github:
            client-id: ${GITHUB_CLIENT_ID}
            client-secret: ${GITHUB_CLIENT_SECRET}
            scope: read:user, user:email
```

---

## OpenID Connect (OIDC)

**OpenID Connect** is an authentication layer built on top of OAuth 2.0. While OAuth 2.0 is an *authorization* framework (it grants access to resources), OIDC adds a standardized way to authenticate users — to know *who* the user is, not just that they have permission to access something.

OIDC extends OAuth 2.0 with the **ID Token** — a JWT containing standardized claims about the authenticated user: `sub` (a stable, unique user identifier), `name`, `email`, `email_verified`, `picture`, and more. The ID Token is meant to be consumed by your application to identify the user; the Access Token is meant to be sent to resource servers (APIs) to authorize operations.

The key distinction to internalize is this: use the **ID Token** for authentication (signing the user into your app), and use the **Access Token** for authorization (calling APIs on behalf of the user). Never use an Access Token as proof of identity — its format is not standardized and its contents are opaque to your client.

---

## SAML 2.0

**SAML 2.0 (Security Assertion Markup Language)** is an XML-based open standard for authentication and authorization, predominantly used in enterprise SSO (Single Sign-On) scenarios. When a company uses Okta, Active Directory Federation Services (ADFS), or Ping Identity for employee authentication, you will likely encounter SAML.

The flow is conceptually similar to OIDC: the user is redirected to an **Identity Provider (IdP)** like Okta, authenticates there (using their corporate credentials, MFA, smart card, etc.), and is redirected back to your application (**Service Provider — SP**) with a signed XML **assertion** containing the user's identity and attributes.

SAML is older and more complex than OIDC, but it is still widely used in enterprise environments and is often a requirement for selling software to large corporations. Spring Security SAML 2.0 support is available through `spring-security-saml2-service-provider`.

---

## API Keys

**API keys** are long, random, opaque strings used to authenticate programmatic clients — developer tools, third-party integrations, server-to-server communication. Unlike JWTs, they carry no embedded claims; the server must look them up in a database on every request. This means they can be instantly revoked (delete the database record) and you can track usage per key.

```java
@Service
public class ApiKeyService {

    private final ApiKeyRepository apiKeyRepository;

    // Generate a new API key for a client application
    public String generateApiKey(Long applicationId) {
        // Use SecureRandom (via UUID or direct generation) — NEVER use Math.random()
        // A UUID is 128-bit random, sufficient for an API key
        String rawKey = "sk_live_" + UUID.randomUUID().toString().replace("-", "");
        // sk_live_ prefix helps users identify this as a production live key

        // Hash the key before storing — if your database is breached,
        // attackers get hashes, not the actual keys
        String hashedKey = hashApiKey(rawKey);

        apiKeyRepository.save(new ApiKey(applicationId, hashedKey,
            Instant.now(), null)); // null expiry = non-expiring

        // Return the RAW key to the client — they will never see it again
        // (similar to how GitHub shows a personal access token only once)
        return rawKey;
    }

    // Validate an incoming API key — look it up by its hash
    public Optional<ApiKeyPrincipal> validateApiKey(String rawKey) {
        String hashedKey = hashApiKey(rawKey);
        return apiKeyRepository.findByHashedKey(hashedKey)
            .filter(key -> !key.isRevoked())
            .filter(key -> key.getExpiresAt() == null ||
                           key.getExpiresAt().isAfter(Instant.now()))
            .map(key -> new ApiKeyPrincipal(key.getApplicationId(), key.getScopes()));
    }

    private String hashApiKey(String rawKey) {
        // SHA-256 is appropriate for hashing API keys (they are long and random,
        // so a slow hash like BCrypt is not needed — brute force is not feasible)
        try {
            MessageDigest digest = MessageDigest.getInstance("SHA-256");
            byte[] hash = digest.digest(rawKey.getBytes(StandardCharsets.UTF_8));
            return HexFormat.of().formatHex(hash);
        } catch (NoSuchAlgorithmException e) {
            throw new RuntimeException("SHA-256 not available", e);
        }
    }
}

// Spring Security filter to authenticate API key requests
@Component
public class ApiKeyAuthenticationFilter extends OncePerRequestFilter {

    private final ApiKeyService apiKeyService;

    @Override
    protected void doFilterInternal(HttpServletRequest request,
            HttpServletResponse response, FilterChain filterChain)
            throws ServletException, IOException {

        // API keys are passed in the X-API-Key header (standard convention)
        String apiKey = request.getHeader("X-API-Key");

        if (apiKey != null) {
            apiKeyService.validateApiKey(apiKey).ifPresent(principal -> {
                ApiKeyAuthenticationToken auth = new ApiKeyAuthenticationToken(
                    principal, principal.getAuthorities());
                SecurityContextHolder.getContext().setAuthentication(auth);
            });
        }

        filterChain.doFilter(request, response);
    }
}
```

---

## Best Practices Summary

The decision between session-based and JWT authentication depends on your architecture. Session-based authentication is simpler, supports instant revocation (delete the session), and is the right choice for traditional server-rendered web applications. JWT is stateless and works well for distributed systems and SPAs, but requires careful handling of revocation (you must maintain a token blacklist or use short expiry times).

For any public-facing API that will be used by third-party developers, use OAuth 2.0. Do not invent your own authentication protocol — the ecosystem of libraries, tooling, and security research around OAuth 2.0 and OIDC is vast and well-tested.

Store refresh tokens securely. Access tokens can live in memory (a JavaScript variable in a SPA), but refresh tokens should be in HttpOnly, Secure, SameSite=Strict cookies where JavaScript cannot read them and cross-site requests cannot send them.

Never trust the payload of a JWT without verifying the signature. The signature verification step is what makes JWTs trustworthy. A JWT decoder that skips signature verification is a critical vulnerability.

Always implement the `alg: none` attack defense — reject any JWT that specifies `"alg": "none"` in its header. JJWT does this correctly by default, but if you ever build your own JWT validation logic, this is an easy mistake to make.

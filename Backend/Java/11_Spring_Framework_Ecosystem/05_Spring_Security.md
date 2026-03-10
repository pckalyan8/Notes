# 🔐 11.5 — Spring Security: Authentication, Authorization, JWT & OAuth2

Spring Security is the most powerful and most complex module in the Spring ecosystem. It provides authentication (who are you?) and authorization (what are you allowed to do?) out of the box. When you add `spring-boot-starter-security` to your project, every HTTP endpoint is immediately secured — no configuration needed. Understanding *why* it works that way, and how to shape it to your needs, is what this chapter is about.

---

## 🧠 Core Concepts Before Any Code

Two terms are the absolute foundation of Spring Security and must be clear before going further.

**Authentication** is the process of verifying identity — "prove who you are." It answers the question "Are you really Alice?" The outcome is a populated `Authentication` object in Spring's `SecurityContextHolder`, containing the principal (the user) and their granted authorities.

**Authorization** is the process of verifying permission — "are you allowed to do this?" It answers the question "Alice is verified, but is Alice allowed to access the admin dashboard?" Authorization always happens *after* authentication succeeds.

The `SecurityContextHolder` is a thread-local container that holds the `Authentication` object for the currently executing request. When a request arrives and passes the security filter chain, its `Authentication` is placed here. When your code runs, you can always retrieve the current user from this holder.

---

## 🏗️ The Security Filter Chain

Spring Security is implemented as a chain of servlet filters. Every HTTP request passes through this chain before reaching your controllers. Each filter in the chain has a specific job: one reads the session cookie and restores the `Authentication` object, another checks for a JWT Bearer token in the Authorization header, another checks if the authenticated user has the right roles for the requested URL, and so on.

```
HTTP Request
    │
    ▼
SecurityFilterChain
    ├── SecurityContextPersistenceFilter   ← Restores SecurityContext from session/token
    ├── UsernamePasswordAuthenticationFilter ← Processes login form submissions
    ├── BearerTokenAuthenticationFilter    ← Reads JWT from Authorization header (if configured)
    ├── ExceptionTranslationFilter         ← Converts AuthenticationException to 401, AccessDeniedException to 403
    └── FilterSecurityInterceptor          ← Checks authorization rules for the request URL
    │
    ▼
Your Controller (only reached if all filters pass)
```

---

## ⚙️ Basic Configuration — SecurityFilterChain Bean

Since Spring Security 6 (Spring Boot 3+), you configure security by declaring a `SecurityFilterChain` bean. The older `WebSecurityConfigurerAdapter` (extend a class) pattern has been removed.

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity // Enables @PreAuthorize, @Secured, @PostAuthorize on methods
public class SecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            // ── CSRF ──
            // CSRF protection prevents malicious sites from submitting forms on behalf of users
            // For stateless REST APIs (JWT-based), CSRF is not needed — disable it
            .csrf(csrf -> csrf.disable())

            // ── Session Management ──
            // For REST APIs: STATELESS = no HttpSession; every request must authenticate itself
            // For web apps with login: use SessionCreationPolicy.IF_REQUIRED (the default)
            .sessionManagement(session ->
                session.sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            )

            // ── Authorization Rules ──
            // Rules are evaluated top-to-bottom; the FIRST matching rule wins
            .authorizeHttpRequests(auth -> auth
                // Fully public endpoints — no authentication required
                .requestMatchers("/api/auth/**").permitAll()
                .requestMatchers("/api/public/**").permitAll()
                .requestMatchers(HttpMethod.GET, "/api/courses/**").permitAll()
                .requestMatchers("/actuator/health").permitAll()

                // Requires ADMIN role (Spring adds the "ROLE_" prefix automatically)
                .requestMatchers("/api/admin/**").hasRole("ADMIN")

                // Requires either ADMIN or TEACHER role
                .requestMatchers("/api/grades/**").hasAnyRole("ADMIN", "TEACHER")

                // Requires a specific authority (not a role — used for fine-grained permissions)
                .requestMatchers(HttpMethod.DELETE, "/api/**").hasAuthority("DELETE_RECORDS")

                // Everything else requires the user to be authenticated
                .anyRequest().authenticated()
            )

            // ── JWT Filter — add BEFORE the standard auth filter ──
            .addFilterBefore(jwtAuthenticationFilter(), UsernamePasswordAuthenticationFilter.class);

        return http.build();
    }

    // The AuthenticationManager is needed to process login requests
    @Bean
    public AuthenticationManager authenticationManager(AuthenticationConfiguration config)
            throws Exception {
        return config.getAuthenticationManager();
    }

    // BCrypt is the standard password hashing algorithm — always use a proper encoder
    // Never store passwords as plain text or MD5/SHA-1
    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder(12); // Cost factor 12 — work factor; higher = slower
    }

    @Bean
    public JwtAuthenticationFilter jwtAuthenticationFilter() {
        return new JwtAuthenticationFilter(jwtService, userDetailsService);
    }
}
```

---

## 👤 UserDetailsService — Loading Users

Spring Security needs to know how to load a user by their username (usually their email in modern apps). You provide this by implementing `UserDetailsService`:

```java
@Service
public class CustomUserDetailsService implements UserDetailsService {

    private final UserRepository userRepository;

    public CustomUserDetailsService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    @Override
    public UserDetails loadUserByUsername(String email) throws UsernameNotFoundException {
        // Load the user from your database
        User user = userRepository.findByEmail(email)
            .orElseThrow(() -> new UsernameNotFoundException("User not found: " + email));

        // Convert to Spring Security's UserDetails
        // GrantedAuthority is what Spring Security checks against your authorization rules
        List<GrantedAuthority> authorities = user.getRoles().stream()
            .map(role -> new SimpleGrantedAuthority("ROLE_" + role.name()))
            .collect(Collectors.toList());

        return new org.springframework.security.core.userdetails.User(
            user.getEmail(),
            user.getPasswordHash(),
            user.isEnabled(),          // accountEnabled
            true,                      // accountNonExpired
            true,                      // credentialsNonExpired
            !user.isLocked(),          // accountNonLocked
            authorities
        );
    }
}
```

---

## 🔑 JWT Authentication — Full Implementation

JSON Web Tokens (JWT) are the standard authentication mechanism for stateless REST APIs. A JWT is a self-contained token that encodes the user's identity and claims, signed with a secret so the server can verify it without hitting the database on every request.

A JWT has three parts separated by dots: `Header.Payload.Signature`. The header specifies the algorithm (HS256 or RS256). The payload contains claims (user id, roles, expiry). The signature prevents tampering — it is the HMAC of the header and payload, signed with a secret key only the server knows.

```xml
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
public class JwtService {

    // ⚠️ Store the secret in application.properties or an env variable — NEVER hardcode it
    @Value("${app.jwt.secret}")
    private String jwtSecret;

    @Value("${app.jwt.expiration-ms:86400000}") // Default: 24 hours
    private long jwtExpirationMs;

    // Build the signing key from the Base64-encoded secret
    private SecretKey getSigningKey() {
        byte[] keyBytes = Decoders.BASE64.decode(jwtSecret);
        return Keys.hmacShaKeyFor(keyBytes);
    }

    // Generate a JWT access token for an authenticated user
    public String generateToken(UserDetails userDetails) {
        Map<String, Object> extraClaims = new HashMap<>();
        // Add roles as a claim in the token body
        List<String> roles = userDetails.getAuthorities().stream()
            .map(GrantedAuthority::getAuthority)
            .collect(Collectors.toList());
        extraClaims.put("roles", roles);

        return Jwts.builder()
            .claims(extraClaims)                           // Custom claims
            .subject(userDetails.getUsername())            // The "sub" claim — typically the email
            .issuedAt(new Date())                          // "iat" — when the token was issued
            .expiration(new Date(System.currentTimeMillis() + jwtExpirationMs)) // "exp" claim
            .signWith(getSigningKey())                     // Sign with HMAC-SHA256
            .compact();                                    // Build and serialize to a string
    }

    // Generate a longer-lived refresh token (used to get new access tokens)
    public String generateRefreshToken(UserDetails userDetails) {
        return Jwts.builder()
            .subject(userDetails.getUsername())
            .issuedAt(new Date())
            .expiration(new Date(System.currentTimeMillis() + 7 * 24 * 60 * 60 * 1000L)) // 7 days
            .signWith(getSigningKey())
            .compact();
    }

    // Extract the email (subject) from a JWT
    public String extractEmail(String token) {
        return extractClaim(token, Claims::getSubject);
    }

    // Check if a token is valid and not expired
    public boolean isTokenValid(String token, UserDetails userDetails) {
        final String email = extractEmail(token);
        return email.equals(userDetails.getUsername()) && !isTokenExpired(token);
    }

    private boolean isTokenExpired(String token) {
        return extractClaim(token, Claims::getExpiration).before(new Date());
    }

    // Generic helper to extract any claim from the token
    public <T> T extractClaim(String token, Function<Claims, T> claimsResolver) {
        final Claims claims = extractAllClaims(token);
        return claimsResolver.apply(claims);
    }

    private Claims extractAllClaims(String token) {
        return Jwts.parser()
            .verifyWith(getSigningKey())   // Verifies the signature
            .build()
            .parseSignedClaims(token)      // Throws if signature is invalid or token is expired
            .getPayload();
    }
}
```

```java
// The filter that intercepts every request and validates the JWT
@Component
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    private final JwtService jwtService;
    private final UserDetailsService userDetailsService;

    public JwtAuthenticationFilter(JwtService jwtService, UserDetailsService userDetailsService) {
        this.jwtService        = jwtService;
        this.userDetailsService = userDetailsService;
    }

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                     HttpServletResponse response,
                                     FilterChain filterChain) throws ServletException, IOException {
        // 1. Extract the Authorization header
        final String authHeader = request.getHeader(HttpHeaders.AUTHORIZATION);

        // 2. If there's no Bearer token, skip JWT processing and continue the filter chain
        // (The request will later be rejected by the authorization rules if auth is required)
        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            filterChain.doFilter(request, response);
            return;
        }

        // 3. Extract the token (remove the "Bearer " prefix)
        final String jwt   = authHeader.substring(7);
        final String email = jwtService.extractEmail(jwt);

        // 4. If the email was extracted and the user is not yet authenticated in this request
        if (email != null && SecurityContextHolder.getContext().getAuthentication() == null) {
            UserDetails userDetails = userDetailsService.loadUserByUsername(email);

            // 5. Validate the token
            if (jwtService.isTokenValid(jwt, userDetails)) {
                // 6. Create an Authentication object and store it in the SecurityContext
                UsernamePasswordAuthenticationToken authToken =
                    new UsernamePasswordAuthenticationToken(
                        userDetails,
                        null,                          // No credentials needed after validation
                        userDetails.getAuthorities()   // Roles/permissions from the UserDetails
                    );
                authToken.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));
                SecurityContextHolder.getContext().setAuthentication(authToken);
            }
        }

        // 7. Continue processing the request (now with the Authentication set in the context)
        filterChain.doFilter(request, response);
    }
}
```

```java
// The authentication controller — handles login and token refresh
@RestController
@RequestMapping("/api/auth")
public class AuthController {

    private final AuthenticationManager authManager;
    private final UserDetailsService    userDetailsService;
    private final JwtService            jwtService;
    private final UserRepository        userRepository;
    private final PasswordEncoder       passwordEncoder;

    public AuthController(AuthenticationManager authManager,
                          UserDetailsService userDetailsService,
                          JwtService jwtService,
                          UserRepository userRepository,
                          PasswordEncoder passwordEncoder) {
        this.authManager        = authManager;
        this.userDetailsService = userDetailsService;
        this.jwtService         = jwtService;
        this.userRepository     = userRepository;
        this.passwordEncoder    = passwordEncoder;
    }

    @PostMapping("/login")
    public ResponseEntity<AuthResponse> login(@RequestBody @Valid LoginRequest request) {
        // AuthenticationManager verifies the credentials against your UserDetailsService
        // Throws AuthenticationException if credentials are wrong — never return 200 on failure
        Authentication auth = authManager.authenticate(
            new UsernamePasswordAuthenticationToken(request.email(), request.password())
        );

        UserDetails userDetails = (UserDetails) auth.getPrincipal();
        String accessToken  = jwtService.generateToken(userDetails);
        String refreshToken = jwtService.generateRefreshToken(userDetails);

        return ResponseEntity.ok(new AuthResponse(accessToken, refreshToken,
            "Bearer", jwtService.getExpirationMs()));
    }

    @PostMapping("/register")
    public ResponseEntity<Void> register(@RequestBody @Valid RegisterRequest request) {
        if (userRepository.existsByEmail(request.email())) {
            throw new DuplicateEmailException("Email already registered: " + request.email());
        }
        User user = new User(
            request.name(),
            request.email(),
            passwordEncoder.encode(request.password()), // ALWAYS hash the password
            Set.of(Role.STUDENT)
        );
        userRepository.save(user);
        return ResponseEntity.status(HttpStatus.CREATED).build();
    }

    @PostMapping("/refresh")
    public ResponseEntity<AuthResponse> refresh(@RequestBody @Valid RefreshRequest request) {
        String email = jwtService.extractEmail(request.refreshToken());
        UserDetails userDetails = userDetailsService.loadUserByUsername(email);
        if (!jwtService.isTokenValid(request.refreshToken(), userDetails)) {
            return ResponseEntity.status(HttpStatus.UNAUTHORIZED).build();
        }
        String newAccessToken = jwtService.generateToken(userDetails);
        return ResponseEntity.ok(new AuthResponse(newAccessToken, request.refreshToken(),
            "Bearer", jwtService.getExpirationMs()));
    }
}

// DTOs
public record LoginRequest(
    @NotBlank @Email String email,
    @NotBlank String password) {}

public record RegisterRequest(
    @NotBlank String name,
    @NotBlank @Email String email,
    @Size(min = 8) String password) {}

public record AuthResponse(
    String accessToken,
    String refreshToken,
    String tokenType,
    long expiresInMs) {}
```

---

## 🛡️ Method-Level Security

With `@EnableMethodSecurity`, you can apply security rules directly on service methods using annotations. This is powerful because it enforces access control at the business logic layer, not just at the HTTP routing layer.

```java
@Service
public class CourseService {

    // @PreAuthorize: evaluated BEFORE the method executes
    // Uses SpEL (Spring Expression Language) to express conditions
    @PreAuthorize("hasRole('ADMIN') or hasRole('TEACHER')")
    public Course createCourse(CreateCourseRequest request) { ... }

    @PreAuthorize("hasRole('ADMIN')")
    public void deleteCourse(Long id) { ... }

    // Access the method's own arguments in the expression
    @PreAuthorize("hasRole('ADMIN') or #studentId == authentication.principal.id")
    public StudentProgress getProgress(Long studentId) { ... }
    // This means: allow access if you're an ADMIN, or if you're requesting your OWN progress

    // @PostAuthorize: evaluated AFTER the method executes; can access the return value
    // Useful when the authorization decision depends on the data being returned
    @PostAuthorize("returnObject.ownerEmail == authentication.principal.username")
    public Document getDocument(Long id) { ... }

    // @PreFilter / @PostFilter: filter a collection argument / return value
    @PostFilter("filterObject.ownerEmail == authentication.principal.username")
    public List<Document> findAllDocuments() { ... }
    // Only returns documents owned by the currently authenticated user

    // @Secured: simpler; only checks role names; less expressive than @PreAuthorize
    @Secured({"ROLE_ADMIN", "ROLE_TEACHER"})
    public List<Student> findEnrolledStudents(Long courseId) { ... }
}
```

---

## 📋 Accessing the Current User

Your service and controller methods often need to know who the currently logged-in user is:

```java
// Method 1: SecurityContextHolder — works anywhere in the call stack
public void someServiceMethod() {
    Authentication auth = SecurityContextHolder.getContext().getAuthentication();
    if (auth != null && auth.isAuthenticated()) {
        String email = auth.getName(); // Returns the username (principal's name)
        Collection<? extends GrantedAuthority> roles = auth.getAuthorities();
        UserDetails user = (UserDetails) auth.getPrincipal(); // Cast to your UserDetails type
    }
}

// Method 2: @AuthenticationPrincipal — inject the principal directly into a controller method
@RestController
@RequestMapping("/api/me")
public class ProfileController {

    @GetMapping
    public UserProfileDTO getMyProfile(@AuthenticationPrincipal UserDetails currentUser) {
        // Spring injects the principal from the SecurityContextHolder automatically
        return profileService.findByEmail(currentUser.getUsername());
    }

    // If you have a custom UserDetails implementation with extra fields:
    @GetMapping("/settings")
    public SettingsDTO getMySettings(@AuthenticationPrincipal CustomUserDetails currentUser) {
        return settingsService.findById(currentUser.getUserId()); // Access custom fields
    }
}

// Method 3: Custom annotation wrapping @AuthenticationPrincipal
@Target(ElementType.PARAMETER)
@Retention(RetentionPolicy.RUNTIME)
@AuthenticationPrincipal
public @interface CurrentUser {}

// Usage becomes more expressive:
@GetMapping("/dashboard")
public DashboardDTO getDashboard(@CurrentUser CustomUserDetails user) {
    return dashboardService.buildFor(user.getUserId());
}
```

---

## 🌐 OAuth2 / OpenID Connect (Preview)

OAuth2 allows users to log in using a third-party identity provider (Google, GitHub, Facebook). Your application never sees the user's password — the provider handles authentication and returns a token confirming the user's identity.

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-client</artifactId>
</dependency>
```

```properties
# Register your application with Google Cloud Console first
# Then configure the client credentials here
spring.security.oauth2.client.registration.google.client-id=YOUR_GOOGLE_CLIENT_ID
spring.security.oauth2.client.registration.google.client-secret=YOUR_GOOGLE_CLIENT_SECRET
spring.security.oauth2.client.registration.google.scope=email,profile

spring.security.oauth2.client.registration.github.client-id=YOUR_GITHUB_CLIENT_ID
spring.security.oauth2.client.registration.github.client-secret=YOUR_GITHUB_CLIENT_SECRET
```

```java
@Configuration
@EnableWebSecurity
public class OAuth2SecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/", "/login/**", "/error").permitAll()
                .anyRequest().authenticated()
            )
            .oauth2Login(oauth2 -> oauth2
                .loginPage("/login")
                .defaultSuccessUrl("/dashboard", true)
                .userInfoEndpoint(userInfo -> userInfo
                    // Customize how the OAuth2 user info is converted to your UserDetails
                    .userService(customOAuth2UserService)
                )
            )
            .logout(logout -> logout
                .logoutSuccessUrl("/")
                .invalidateHttpSession(true)
                .deleteCookies("JSESSIONID")
            );
        return http.build();
    }
}
```

---

## ✅ Best Practices & Important Points

**Never store passwords in plain text.** Always use `BCryptPasswordEncoder` or `Argon2PasswordEncoder`. BCrypt is purposefully slow (computational work factor) which makes brute-force attacks impractical. Never use MD5, SHA-1, or SHA-256 for passwords — they are too fast and therefore brute-forceable.

**Store the JWT secret securely.** The JWT secret should be a long (256-bit minimum), cryptographically random key stored in an environment variable or secrets manager — never hardcoded in source code or checked into version control. If the secret is leaked, every token issued with that secret can be forged.

**JWT access tokens should have a short expiry (15–60 minutes).** If a JWT is stolen, there is no way to invalidate it on the server side (stateless by design). A short expiry limits the damage window. Use refresh tokens (longer-lived) to issue new access tokens, and keep a server-side blacklist of revoked refresh tokens.

**Always disable CSRF protection for stateless REST APIs.** CSRF attacks exploit session cookies. Since JWT-based REST APIs don't use cookies for auth (they use the Authorization header), CSRF is not applicable. Leaving CSRF enabled in a JWT API will cause all state-changing requests to fail with a 403 because there's no CSRF token to send.

**Put fine-grained authorization in `@PreAuthorize` on service methods, not just URL patterns.** URL-pattern security is a coarse-grained first line of defense. Method-level security catches cases where data belonging to one user is accessed by another user through a different URL. Both layers together give defense in depth.

**Never catch and suppress `AuthenticationException`.** If login fails, let the exception propagate. Spring Security's `ExceptionTranslationFilter` will convert it to a 401 response. If you catch and ignore it, you may accidentally return sensitive information or allow unauthenticated access.

---

## 🔑 Key Summary

```
SecurityFilterChain        → Replaces WebSecurityConfigurerAdapter in Spring Security 6+; configures the filter chain
Authentication             → Represents a verified user; lives in SecurityContextHolder
Authorization              → Rules for what an authenticated user can access
UserDetailsService         → You implement this to tell Spring how to load a user from your database
PasswordEncoder            → BCryptPasswordEncoder; never store plain-text passwords
@EnableMethodSecurity      → Activates @PreAuthorize, @PostAuthorize, @Secured on service methods
@PreAuthorize              → SpEL expression evaluated BEFORE the method; access method args and principal
CSRF                       → Disable for stateless REST APIs; keep enabled for form-based web apps
SessionCreationPolicy      → STATELESS for JWT APIs; IF_REQUIRED for session-based web apps
JWT                        → Self-contained token: Header.Payload.Signature; signed with HMAC secret
JwtAuthenticationFilter    → OncePerRequestFilter that validates JWT and populates SecurityContext
OAuth2                     → Delegate authentication to Google/GitHub/etc.; starter-oauth2-client
@AuthenticationPrincipal   → Inject the current UserDetails into a controller method parameter
```

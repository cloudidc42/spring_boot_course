# Part 16: Spring Security
## ขั้นตอนที่ 396-430

> **ระดับ:** กลาง (Intermediate)  
> **เวลาเรียน:** 6-7 ชั่วโมง  
> **เป้าหมาย:** ตั้งค่า Spring Security พร้อม JWT Authentication

---

## ขั้นตอนที่ 396: Spring Security Dependency

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-api</artifactId>
    <version>0.12.3</version>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-impl</artifactId>
    <version>0.12.3</version>
    <scope>runtime</scope>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-jackson</artifactId>
    <version>0.12.3</version>
    <scope>runtime</scope>
</dependency>
```

---

## ขั้นตอนที่ 397: Security Architecture

```
HTTP Request
    ↓
[DelegatingFilterProxy]
    ↓
[SecurityFilterChain]
    ↓ (filters execute in order)
[JwtAuthenticationFilter]  ← extract JWT, set authentication
    ↓
[SecurityContext]  ← stores Authentication
    ↓
[AuthorizationFilter]  ← check permissions
    ↓
DispatcherServlet
    ↓
Controller
```

---

## ขั้นตอนที่ 398: Security Configuration

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity  // @PreAuthorize, @PostAuthorize
@RequiredArgsConstructor
public class SecurityConfig {
    
    private final JwtAuthenticationFilter jwtAuthFilter;
    private final UserDetailsService userDetailsService;
    private final AuthEntryPoint authEntryPoint;
    private final AccessDeniedHandlerImpl accessDeniedHandler;
    
    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        return http
            // Disable CSRF (REST API ใช้ JWT)
            .csrf(csrf -> csrf.disable())
            
            // CORS
            .cors(cors -> cors.configurationSource(corsConfigurationSource()))
            
            // Session management - STATELESS for JWT
            .sessionManagement(session -> 
                session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            
            // Exception handling
            .exceptionHandling(ex -> ex
                .authenticationEntryPoint(authEntryPoint)
                .accessDeniedHandler(accessDeniedHandler))
            
            // Authorization rules
            .authorizeHttpRequests(auth -> auth
                // Public endpoints
                .requestMatchers("/api/v1/auth/**").permitAll()
                .requestMatchers(HttpMethod.GET, "/api/v1/products/**").permitAll()
                .requestMatchers(HttpMethod.GET, "/api/v1/categories/**").permitAll()
                .requestMatchers("/actuator/health", "/actuator/info").permitAll()
                .requestMatchers("/swagger-ui/**", "/v3/api-docs/**").permitAll()
                
                // Admin only
                .requestMatchers("/api/v1/admin/**").hasRole("ADMIN")
                
                // Authenticated users
                .anyRequest().authenticated()
            )
            
            // Add JWT filter before UsernamePasswordAuthenticationFilter
            .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class)
            
            .build();
    }
    
    @Bean
    public CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration config = new CorsConfiguration();
        config.setAllowedOriginPatterns(List.of("*"));
        config.setAllowedMethods(List.of("GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS"));
        config.setAllowedHeaders(List.of("*"));
        config.setAllowCredentials(true);
        config.setMaxAge(3600L);
        
        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/**", config);
        return source;
    }
    
    @Bean
    public AuthenticationProvider authenticationProvider() {
        DaoAuthenticationProvider provider = new DaoAuthenticationProvider();
        provider.setUserDetailsService(userDetailsService);
        provider.setPasswordEncoder(passwordEncoder());
        return provider;
    }
    
    @Bean
    public AuthenticationManager authenticationManager(AuthenticationConfiguration config) throws Exception {
        return config.getAuthenticationManager();
    }
    
    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder(12);
    }
}
```

---

## ขั้นตอนที่ 399: UserDetails Implementation

```java
// UserDetailsService
@Service
@RequiredArgsConstructor
public class UserDetailsServiceImpl implements UserDetailsService {
    
    private final UserRepository userRepository;
    
    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        return userRepository.findByEmail(username)
            .orElseThrow(() -> new UsernameNotFoundException("User not found: " + username));
    }
}

// User implements UserDetails
@Entity
@Table(name = "users")
public class User extends BaseEntity implements UserDetails {
    
    // ... fields ...
    
    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
        return List.of(new SimpleGrantedAuthority("ROLE_" + role.name()));
    }
    
    @Override
    public String getPassword() {
        return password;
    }
    
    @Override
    public String getUsername() {
        return email;  // use email as username
    }
    
    @Override
    public boolean isAccountNonExpired() { return true; }
    
    @Override
    public boolean isAccountNonLocked() { return active; }
    
    @Override
    public boolean isCredentialsNonExpired() { return true; }
    
    @Override
    public boolean isEnabled() { return active; }
}
```

---

## ขั้นตอนที่ 400: JWT Service

```java
@Service
@Slf4j
public class JwtService {
    
    @Value("${security.jwt.secret-key}")
    private String secretKey;
    
    @Value("${security.jwt.access-token-expiration:86400000}")  // 24 hours
    private long accessTokenExpiration;
    
    @Value("${security.jwt.refresh-token-expiration:604800000}")  // 7 days
    private long refreshTokenExpiration;
    
    // ===== Generate Tokens =====
    
    public String generateAccessToken(UserDetails user) {
        Map<String, Object> claims = new HashMap<>();
        if (user instanceof User u) {
            claims.put("role", u.getRole().name());
            claims.put("userId", u.getId());
        }
        return buildToken(claims, user.getUsername(), accessTokenExpiration);
    }
    
    public String generateRefreshToken(UserDetails user) {
        return buildToken(new HashMap<>(), user.getUsername(), refreshTokenExpiration);
    }
    
    private String buildToken(Map<String, Object> claims, String subject, long expiration) {
        return Jwts.builder()
            .claims(claims)
            .subject(subject)
            .issuedAt(new Date())
            .expiration(new Date(System.currentTimeMillis() + expiration))
            .signWith(getSigningKey())
            .compact();
    }
    
    // ===== Validate Token =====
    
    public boolean isTokenValid(String token, UserDetails user) {
        try {
            String username = extractUsername(token);
            return username.equals(user.getUsername()) && !isTokenExpired(token);
        } catch (JwtException e) {
            log.warn("Invalid JWT token: {}", e.getMessage());
            return false;
        }
    }
    
    private boolean isTokenExpired(String token) {
        return extractExpiration(token).before(new Date());
    }
    
    // ===== Extract Claims =====
    
    public String extractUsername(String token) {
        return extractClaim(token, Claims::getSubject);
    }
    
    public Date extractExpiration(String token) {
        return extractClaim(token, Claims::getExpiration);
    }
    
    public <T> T extractClaim(String token, Function<Claims, T> resolver) {
        Claims claims = extractAllClaims(token);
        return resolver.apply(claims);
    }
    
    private Claims extractAllClaims(String token) {
        return Jwts.parser()
            .verifyWith(getSigningKey())
            .build()
            .parseSignedClaims(token)
            .getPayload();
    }
    
    private SecretKey getSigningKey() {
        byte[] keyBytes = Decoders.BASE64.decode(secretKey);
        return Keys.hmacShaKeyFor(keyBytes);
    }
}
```

---

## ขั้นตอนที่ 401: JWT Authentication Filter

```java
@Component
@RequiredArgsConstructor
@Slf4j
public class JwtAuthenticationFilter extends OncePerRequestFilter {
    
    private final JwtService jwtService;
    private final UserDetailsService userDetailsService;
    
    @Override
    protected void doFilterInternal(
        HttpServletRequest request,
        HttpServletResponse response,
        FilterChain chain
    ) throws ServletException, IOException {
        
        String authHeader = request.getHeader("Authorization");
        
        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            chain.doFilter(request, response);
            return;
        }
        
        String token = authHeader.substring(7);
        
        try {
            String username = jwtService.extractUsername(token);
            
            // Only authenticate if not already authenticated
            if (username != null && SecurityContextHolder.getContext().getAuthentication() == null) {
                UserDetails userDetails = userDetailsService.loadUserByUsername(username);
                
                if (jwtService.isTokenValid(token, userDetails)) {
                    UsernamePasswordAuthenticationToken authToken = 
                        new UsernamePasswordAuthenticationToken(
                            userDetails,
                            null,
                            userDetails.getAuthorities()
                        );
                    
                    authToken.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));
                    SecurityContextHolder.getContext().setAuthentication(authToken);
                    
                    // Add user info to MDC for logging
                    MDC.put("userId", String.valueOf(((User) userDetails).getId()));
                }
            }
        } catch (JwtException e) {
            log.warn("JWT error: {}", e.getMessage());
        }
        
        chain.doFilter(request, response);
    }
}
```

---

## ขั้นตอนที่ 402: Auth Controller

```java
@RestController
@RequestMapping("/api/v1/auth")
@RequiredArgsConstructor
@Tag(name = "Authentication")
public class AuthController {
    
    private final AuthService authService;
    
    @PostMapping("/register")
    @ResponseStatus(HttpStatus.CREATED)
    public ResponseEntity<ApiResponse<AuthResponse>> register(
        @Valid @RequestBody RegisterRequest request
    ) {
        AuthResponse response = authService.register(request);
        return ResponseEntity.status(HttpStatus.CREATED)
            .body(ApiResponse.success(response, "Registration successful"));
    }
    
    @PostMapping("/login")
    public ResponseEntity<ApiResponse<AuthResponse>> login(
        @Valid @RequestBody LoginRequest request
    ) {
        AuthResponse response = authService.login(request);
        return ResponseEntity.ok(ApiResponse.success(response, "Login successful"));
    }
    
    @PostMapping("/refresh")
    public ResponseEntity<ApiResponse<TokenResponse>> refresh(
        @RequestBody RefreshTokenRequest request
    ) {
        TokenResponse response = authService.refresh(request.refreshToken());
        return ResponseEntity.ok(ApiResponse.success(response));
    }
    
    @PostMapping("/logout")
    @PreAuthorize("isAuthenticated()")
    public ResponseEntity<ApiResponse<Void>> logout(
        @RequestHeader("Authorization") String authHeader
    ) {
        String token = authHeader.substring(7);
        authService.logout(token);
        return ResponseEntity.ok(ApiResponse.success(null, "Logout successful"));
    }
    
    @GetMapping("/me")
    @PreAuthorize("isAuthenticated()")
    public ResponseEntity<ApiResponse<UserResponse>> me(
        @AuthenticationPrincipal User user
    ) {
        return ResponseEntity.ok(ApiResponse.success(authService.getCurrentUser(user)));
    }
}
```

---

## ขั้นตอนที่ 403: Auth Service

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class AuthService {
    
    private final UserRepository userRepository;
    private final PasswordEncoder passwordEncoder;
    private final JwtService jwtService;
    private final AuthenticationManager authenticationManager;
    private final UserMapper userMapper;
    private final RefreshTokenRepository refreshTokenRepository;
    
    @Transactional
    public AuthResponse register(RegisterRequest request) {
        if (userRepository.existsByEmail(request.email())) {
            throw new DuplicateResourceException("User", "email", request.email());
        }
        if (userRepository.existsByUsername(request.username())) {
            throw new DuplicateResourceException("User", "username", request.username());
        }
        
        User user = User.builder()
            .username(request.username())
            .email(request.email())
            .password(passwordEncoder.encode(request.password()))
            .firstName(request.firstName())
            .lastName(request.lastName())
            .role(UserRole.USER)
            .active(true)
            .build();
        
        User saved = userRepository.save(user);
        log.info("New user registered: {}", saved.getEmail());
        
        return generateAuthResponse(saved);
    }
    
    public AuthResponse login(LoginRequest request) {
        // Authenticate - throws exception if invalid
        Authentication auth = authenticationManager.authenticate(
            new UsernamePasswordAuthenticationToken(request.email(), request.password())
        );
        
        User user = (User) auth.getPrincipal();
        
        // Update last login
        user.setLastLoginAt(LocalDateTime.now());
        userRepository.save(user);
        
        log.info("User logged in: {}", user.getEmail());
        
        return generateAuthResponse(user);
    }
    
    public TokenResponse refresh(String refreshToken) {
        RefreshToken token = refreshTokenRepository.findByToken(refreshToken)
            .orElseThrow(() -> new UnauthorizedException("Invalid refresh token"));
        
        if (token.isExpired()) {
            refreshTokenRepository.delete(token);
            throw new UnauthorizedException("Refresh token expired");
        }
        
        String newAccessToken = jwtService.generateAccessToken(token.getUser());
        
        return new TokenResponse(newAccessToken, refreshToken);
    }
    
    @Transactional
    public void logout(String accessToken) {
        // In production: blacklist the token in Redis
        String username = jwtService.extractUsername(accessToken);
        userRepository.findByEmail(username).ifPresent(user -> {
            refreshTokenRepository.deleteByUser(user);
        });
    }
    
    private AuthResponse generateAuthResponse(User user) {
        String accessToken = jwtService.generateAccessToken(user);
        String refreshToken = saveRefreshToken(user);
        
        return new AuthResponse(
            accessToken,
            refreshToken,
            "Bearer",
            jwtService.extractExpiration(accessToken).getTime(),
            userMapper.toResponse(user)
        );
    }
    
    private String saveRefreshToken(User user) {
        refreshTokenRepository.deleteByUser(user);  // delete old token
        
        String token = UUID.randomUUID().toString();
        RefreshToken refreshToken = RefreshToken.builder()
            .token(token)
            .user(user)
            .expiresAt(LocalDateTime.now().plusDays(7))
            .build();
        
        refreshTokenRepository.save(refreshToken);
        return token;
    }
}
```

---

## ขั้นตอนที่ 404: Method-Level Security

```java
@Service
public class PostService {
    
    // Anyone can read
    @PreAuthorize("permitAll()")
    public PostResponse findById(Long id) { ... }
    
    // Must be authenticated
    @PreAuthorize("isAuthenticated()")
    public PostResponse create(CreatePostRequest request) { ... }
    
    // Must be ADMIN
    @PreAuthorize("hasRole('ADMIN')")
    public void deleteAny(Long id) { ... }
    
    // Must own the resource or be ADMIN
    @PreAuthorize("@postSecurity.canEdit(#id, authentication)")
    public PostResponse update(Long id, UpdatePostRequest request) { ... }
    
    // Multiple conditions
    @PreAuthorize("hasRole('ADMIN') or (isAuthenticated() and #userId == authentication.principal.id)")
    public void deleteUser(Long userId) { ... }
}

// Security helper bean
@Component("postSecurity")
@RequiredArgsConstructor
public class PostSecurityEvaluator {
    
    private final PostRepository postRepository;
    
    public boolean canEdit(Long postId, Authentication auth) {
        if (auth == null) return false;
        
        // Admin can edit anything
        boolean isAdmin = auth.getAuthorities().stream()
            .anyMatch(a -> a.getAuthority().equals("ROLE_ADMIN"));
        if (isAdmin) return true;
        
        // Author can edit their own post
        User user = (User) auth.getPrincipal();
        return postRepository.existsByIdAndAuthorId(postId, user.getId());
    }
}
```

---

## ขั้นตอนที่ 405: Error Handlers

```java
// 401 Unauthorized
@Component
public class AuthEntryPoint implements AuthenticationEntryPoint {
    
    @Override
    public void commence(
        HttpServletRequest request, 
        HttpServletResponse response, 
        AuthenticationException ex
    ) throws IOException {
        response.setContentType(MediaType.APPLICATION_JSON_VALUE);
        response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
        
        Map<String, Object> body = Map.of(
            "success", false,
            "errorCode", "UNAUTHORIZED",
            "message", "Authentication required",
            "path", request.getRequestURI()
        );
        
        new ObjectMapper().writeValue(response.getOutputStream(), body);
    }
}

// 403 Forbidden
@Component
public class AccessDeniedHandlerImpl implements AccessDeniedHandler {
    
    @Override
    public void handle(
        HttpServletRequest request, 
        HttpServletResponse response, 
        AccessDeniedException ex
    ) throws IOException {
        response.setContentType(MediaType.APPLICATION_JSON_VALUE);
        response.setStatus(HttpServletResponse.SC_FORBIDDEN);
        
        Map<String, Object> body = Map.of(
            "success", false,
            "errorCode", "FORBIDDEN",
            "message", "Access denied",
            "path", request.getRequestURI()
        );
        
        new ObjectMapper().writeValue(response.getOutputStream(), body);
    }
}
```

---

## ขั้นตอนที่ 406-430: Security Configuration

```yaml
# application.yml
security:
  jwt:
    secret-key: ${JWT_SECRET:your-256-bit-secret-key-here-must-be-long}
    access-token-expiration: 86400000    # 24 hours in ms
    refresh-token-expiration: 604800000  # 7 days in ms
```

### DTOs

```java
public record RegisterRequest(
    @NotBlank @Size(min = 3, max = 50) String username,
    @NotBlank @Email String email,
    @NotBlank @Size(min = 8) String password,
    String firstName,
    String lastName
) {}

public record LoginRequest(
    @NotBlank @Email String email,
    @NotBlank String password
) {}

public record AuthResponse(
    String accessToken,
    String refreshToken,
    String tokenType,
    long expiresAt,
    UserResponse user
) {}

public record RefreshTokenRequest(
    @NotBlank String refreshToken
) {}

public record TokenResponse(
    String accessToken,
    String refreshToken
) {}
```

### RefreshToken Entity

```java
@Entity
@Table(name = "refresh_tokens")
@Getter @Setter @Builder @NoArgsConstructor @AllArgsConstructor
public class RefreshToken {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(unique = true, nullable = false)
    private String token;
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id")
    private User user;
    
    @Column(name = "expires_at")
    private LocalDateTime expiresAt;
    
    public boolean isExpired() {
        return LocalDateTime.now().isAfter(expiresAt);
    }
}

public interface RefreshTokenRepository extends JpaRepository<RefreshToken, Long> {
    Optional<RefreshToken> findByToken(String token);
    void deleteByUser(User user);
}
```

---

*[← Part 15: Entity Relationships](./part-15-entity-relationships.md) | [Part 17: JWT Advanced →](./part-17-jwt-advanced.md)*

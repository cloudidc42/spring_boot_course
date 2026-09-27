# Part 75: Advanced Spring Security
## ขั้นตอนที่ 2601-2640

> **ระดับ:** ระดับโลก (World-Class)  
> **เวลาเรียน:** 6-8 ชั่วโมง  
> **เป้าหมาย:** Implement advanced security patterns for production applications

---

## ขั้นตอนที่ 2601: Method Security

```java
// เปิดใช้ Method Security
@Configuration
@EnableMethodSecurity(prePostEnabled = true, securedEnabled = true)
public class MethodSecurityConfig { }

// ตัวอย่างการใช้งาน
@Service
@RequiredArgsConstructor
public class OrderService {

    private final OrderRepository orderRepository;

    // ตรวจสอบว่าเป็นเจ้าของ order หรือ ADMIN
    @PreAuthorize("hasRole('ADMIN') or @orderSecurity.isOwner(#orderId, authentication)")
    public Order getOrder(Long orderId) {
        return orderRepository.findById(orderId)
            .orElseThrow(() -> new OrderNotFoundException(orderId));
    }

    // อนุญาตเฉพาะ ADMIN สร้าง discount
    @PreAuthorize("hasRole('ADMIN')")
    public Order applyDiscount(Long orderId, BigDecimal discount) {
        Order order = getOrder(orderId);
        order.applyDiscount(discount);
        return orderRepository.save(order);
    }

    // กรอง result หลังจาก method ทำงาน
    @PostAuthorize("returnObject.customerId == authentication.principal.id or hasRole('ADMIN')")
    public Order getOrderDetails(Long orderId) {
        return orderRepository.findById(orderId)
            .orElseThrow(() -> new OrderNotFoundException(orderId));
    }

    // กรอง collection ตาม tenant
    @PostFilter("filterObject.tenantId == authentication.principal.tenantId")
    public List<Order> getAllOrders() {
        return orderRepository.findAll();
    }
}
```

---

## ขั้นตอนที่ 2602: Custom Security Expressions

```java
// Custom security expression component
@Component("orderSecurity")
@RequiredArgsConstructor
public class OrderSecurityExpression {

    private final OrderRepository orderRepository;

    public boolean isOwner(Long orderId, Authentication auth) {
        if (auth == null) return false;

        UserDetails userDetails = (UserDetails) auth.getPrincipal();
        Long userId = ((CustomUserDetails) userDetails).getId();

        return orderRepository.existsByIdAndCustomerId(orderId, userId);
    }

    public boolean canAccessProduct(Long productId, Authentication auth) {
        if (auth == null) return false;
        // Custom logic: check if product is in user's allowed categories
        return true;
    }
}

// การใช้งาน
@GetMapping("/orders/{id}")
@PreAuthorize("@orderSecurity.isOwner(#id, authentication) or hasRole('ADMIN')")
public ResponseEntity<OrderResponse> getOrder(@PathVariable Long id) {
    return ResponseEntity.ok(orderService.getOrder(id));
}
```

---

## ขั้นตอนที่ 2603: API Key Authentication

```java
// API Key model
@Entity
@Table(name = "api_keys")
@Getter @Setter @Builder
public class ApiKey {
    @Id @GeneratedValue
    private Long id;
    private String keyHash;   // bcrypt hash
    private String prefix;    // first 8 chars (for lookup)
    private Long userId;
    private String name;
    private boolean active;
    private LocalDateTime expiresAt;
    @ElementCollection
    private Set<String> scopes;  // read, write, admin
}

// API Key filter
@Component
@RequiredArgsConstructor
public class ApiKeyAuthFilter extends OncePerRequestFilter {

    private static final String API_KEY_HEADER = "X-API-Key";
    private final ApiKeyService apiKeyService;

    @Override
    protected void doFilterInternal(
        HttpServletRequest request,
        HttpServletResponse response,
        FilterChain filterChain
    ) throws ServletException, IOException {

        String apiKey = request.getHeader(API_KEY_HEADER);

        if (apiKey != null && SecurityContextHolder.getContext().getAuthentication() == null) {
            try {
                Authentication auth = apiKeyService.authenticate(apiKey);
                SecurityContextHolder.getContext().setAuthentication(auth);
            } catch (AuthenticationException e) {
                response.setStatus(HttpStatus.UNAUTHORIZED.value());
                response.getWriter().write("{\"error\":\"Invalid API key\"}");
                return;
            }
        }

        filterChain.doFilter(request, response);
    }
}

// API Key service
@Service
@RequiredArgsConstructor
public class ApiKeyService {

    private final ApiKeyRepository apiKeyRepository;
    private final PasswordEncoder passwordEncoder;

    public Authentication authenticate(String rawKey) {
        // Extract prefix (first 8 chars) for DB lookup
        if (rawKey.length() < 8) throw new BadCredentialsException("Invalid API key format");
        String prefix = rawKey.substring(0, 8);

        ApiKey apiKey = apiKeyRepository.findByPrefix(prefix)
            .orElseThrow(() -> new BadCredentialsException("API key not found"));

        if (!apiKey.isActive()) throw new DisabledException("API key is disabled");
        if (apiKey.getExpiresAt() != null && apiKey.getExpiresAt().isBefore(LocalDateTime.now())) {
            throw new CredentialsExpiredException("API key expired");
        }
        if (!passwordEncoder.matches(rawKey, apiKey.getKeyHash())) {
            throw new BadCredentialsException("Invalid API key");
        }

        List<GrantedAuthority> authorities = apiKey.getScopes().stream()
            .map(scope -> new SimpleGrantedAuthority("SCOPE_" + scope))
            .collect(Collectors.toList());

        return new ApiKeyAuthentication(apiKey, authorities);
    }

    public ApiKeyCreationResult createApiKey(Long userId, String name, Set<String> scopes) {
        String rawKey = generateSecureKey();
        String prefix = rawKey.substring(0, 8);
        String hash = passwordEncoder.encode(rawKey);

        ApiKey key = ApiKey.builder()
            .keyHash(hash)
            .prefix(prefix)
            .userId(userId)
            .name(name)
            .active(true)
            .scopes(scopes)
            .expiresAt(LocalDateTime.now().plusYears(1))
            .build();

        apiKeyRepository.save(key);

        // Return raw key ONCE - never stored in plain text
        return new ApiKeyCreationResult(key.getId(), rawKey, prefix);
    }

    private String generateSecureKey() {
        byte[] bytes = new byte[32];
        new SecureRandom().nextBytes(bytes);
        return Base64.getUrlEncoder().withoutPadding().encodeToString(bytes);
    }
}
```

---

## ขั้นตอนที่ 2604: Mutual TLS (mTLS)

```java
// Enable mTLS in Spring Boot
// application.yml
/*
server:
  ssl:
    enabled: true
    key-store: classpath:server.p12
    key-store-password: ${SSL_KEYSTORE_PASSWORD}
    key-store-type: PKCS12
    trust-store: classpath:truststore.p12
    trust-store-password: ${SSL_TRUSTSTORE_PASSWORD}
    trust-store-type: PKCS12
    client-auth: need  # require client certificate
*/

// Extract client certificate info in Spring Security
@Configuration
@RequiredArgsConstructor
public class MtlsSecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .x509(x509 -> x509
                .subjectPrincipalRegex("CN=(.*?)(?:,|$)")
                .userDetailsService(mtlsUserDetailsService())
            )
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/internal/**").authenticated()
                .anyRequest().permitAll()
            );
        return http.build();
    }

    @Bean
    public UserDetailsService mtlsUserDetailsService() {
        return username -> {
            // username is extracted from CN of client certificate
            return User.withUsername(username)
                .password("")
                .roles("SERVICE")
                .build();
        };
    }
}
```

---

## ขั้นตอนที่ 2605: OAuth2 Token Introspection

```java
// Resource server with token introspection
@Configuration
public class ResourceServerConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .oauth2ResourceServer(oauth2 -> oauth2
                .opaqueToken(opaque -> opaque
                    .introspectionUri("https://auth-server/oauth2/introspect")
                    .introspectionClientCredentials("client-id", "client-secret")
                )
            );
        return http.build();
    }
}

// Custom OpaqueTokenIntrospector to add extra claims
@Component
@RequiredArgsConstructor
public class CustomOpaqueTokenIntrospector implements OpaqueTokenIntrospector {

    private final SpringOpaqueTokenIntrospector delegate;
    private final UserRepository userRepository;

    @Override
    public OAuth2AuthenticatedPrincipal introspect(String token) {
        OAuth2AuthenticatedPrincipal principal = delegate.introspect(token);

        // Enrich with user info from our DB
        String username = principal.getAttribute("username");
        User user = userRepository.findByUsername(username)
            .orElseThrow(() -> new OAuth2IntrospectionException("User not found"));

        Map<String, Object> claims = new HashMap<>(principal.getAttributes());
        claims.put("userId", user.getId());
        claims.put("tenantId", user.getTenantId());
        claims.put("roles", user.getRoles());

        List<GrantedAuthority> authorities = user.getRoles().stream()
            .map(role -> new SimpleGrantedAuthority("ROLE_" + role))
            .collect(Collectors.toList());

        return new DefaultOAuth2AuthenticatedPrincipal(username, claims, authorities);
    }
}
```

---

## ขั้นตอนที่ 2606: Security Audit Logging

```java
// Security audit event listener
@Component
@Slf4j
public class SecurityAuditLogger implements ApplicationListener<AbstractAuthenticationEvent> {

    private final AuditLogRepository auditLogRepository;
    private final HttpServletRequest request;

    @Override
    public void onApplicationEvent(AbstractAuthenticationEvent event) {
        AuditLog log = AuditLog.builder()
            .timestamp(LocalDateTime.now())
            .eventType(event.getClass().getSimpleName())
            .ipAddress(getClientIp())
            .userAgent(request.getHeader("User-Agent"))
            .build();

        if (event instanceof AuthenticationSuccessEvent success) {
            log.setUsername(success.getAuthentication().getName());
            log.setSuccess(true);
        } else if (event instanceof AbstractAuthenticationFailureEvent failure) {
            log.setUsername(failure.getAuthentication().getName());
            log.setSuccess(false);
            log.setFailureReason(failure.getException().getMessage());
        }

        auditLogRepository.save(log);

        if (!log.isSuccess()) {
            log.error("SECURITY: Failed auth attempt for user={} ip={} reason={}",
                log.getUsername(), log.getIpAddress(), log.getFailureReason());
        }
    }

    private String getClientIp() {
        String xff = request.getHeader("X-Forwarded-For");
        return xff != null ? xff.split(",")[0].trim() : request.getRemoteAddr();
    }
}

// Custom audit annotation
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Audited {
    String action();
    String resource() default "";
}

// AOP for @Audited methods
@Aspect
@Component
@RequiredArgsConstructor
@Slf4j
public class AuditAspect {

    private final AuditLogRepository auditLogRepository;

    @Around("@annotation(audited)")
    public Object audit(ProceedingJoinPoint pjp, Audited audited) throws Throwable {
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        String username = auth != null ? auth.getName() : "anonymous";

        try {
            Object result = pjp.proceed();
            auditLogRepository.save(AuditLog.builder()
                .username(username)
                .action(audited.action())
                .resource(audited.resource())
                .success(true)
                .timestamp(LocalDateTime.now())
                .build());
            return result;
        } catch (Exception e) {
            auditLogRepository.save(AuditLog.builder()
                .username(username)
                .action(audited.action())
                .resource(audited.resource())
                .success(false)
                .failureReason(e.getMessage())
                .timestamp(LocalDateTime.now())
                .build());
            throw e;
        }
    }
}
```

---

## ขั้นตอนที่ 2607: CORS Configuration

```java
@Configuration
public class CorsConfig {

    @Bean
    public CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration config = new CorsConfiguration();

        // Production: specific origins only
        config.setAllowedOrigins(List.of(
            "https://myapp.com",
            "https://admin.myapp.com"
        ));

        config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE", "PATCH", "OPTIONS"));
        config.setAllowedHeaders(List.of(
            "Authorization",
            "Content-Type",
            "X-Request-ID",
            "X-Tenant-ID"
        ));
        config.setExposedHeaders(List.of(
            "X-Total-Count",
            "X-Request-ID",
            "X-RateLimit-Remaining"
        ));
        config.setAllowCredentials(true);
        config.setMaxAge(3600L);  // Preflight cache 1 hour

        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/api/**", config);

        // Public endpoints: allow all origins
        CorsConfiguration publicConfig = new CorsConfiguration();
        publicConfig.setAllowedOrigins(List.of("*"));
        publicConfig.setAllowedMethods(List.of("GET"));
        source.registerCorsConfiguration("/api/public/**", publicConfig);

        return source;
    }
}
```

---

## ขั้นตอนที่ 2608-2640: Complete Security Setup Summary

```java
// Complete SecurityFilterChain for production
@Configuration
@EnableWebSecurity
@EnableMethodSecurity
@RequiredArgsConstructor
public class SecurityConfig {

    private final JwtAuthFilter jwtAuthFilter;
    private final ApiKeyAuthFilter apiKeyAuthFilter;
    private final CorsConfigurationSource corsConfigurationSource;

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            // Disable CSRF (stateless API)
            .csrf(AbstractHttpConfigurer::disable)

            // CORS
            .cors(cors -> cors.configurationSource(corsConfigurationSource))

            // Session management: stateless
            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            )

            // Security headers
            .headers(headers -> headers
                .frameOptions(HeadersConfigurer.FrameOptionsConfig::deny)
                .contentTypeOptions(withDefaults())
                .httpStrictTransportSecurity(hsts -> hsts
                    .maxAgeInSeconds(31536000)
                    .includeSubDomains(true)
                )
                .contentSecurityPolicy(csp -> csp
                    .policyDirectives("default-src 'self'")
                )
            )

            // Authorization rules
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/v1/auth/**").permitAll()
                .requestMatchers("/api/v1/public/**").permitAll()
                .requestMatchers("/actuator/health").permitAll()
                .requestMatchers("/actuator/**").hasRole("ADMIN")
                .requestMatchers(HttpMethod.GET, "/api/v1/products/**").permitAll()
                .requestMatchers("/api/v1/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )

            // Exception handling
            .exceptionHandling(ex -> ex
                .authenticationEntryPoint((req, res, e) -> {
                    res.setStatus(HttpStatus.UNAUTHORIZED.value());
                    res.setContentType(MediaType.APPLICATION_JSON_VALUE);
                    res.getWriter().write("{\"error\":\"Unauthorized\",\"message\":\"" + e.getMessage() + "\"}");
                })
                .accessDeniedHandler((req, res, e) -> {
                    res.setStatus(HttpStatus.FORBIDDEN.value());
                    res.setContentType(MediaType.APPLICATION_JSON_VALUE);
                    res.getWriter().write("{\"error\":\"Forbidden\",\"message\":\"Access denied\"}");
                })
            )

            // Add JWT filter before username/password filter
            .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class)
            // Add API key filter (for machine-to-machine)
            .addFilterBefore(apiKeyAuthFilter, JwtAuthFilter.class);

        return http.build();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder(12);
    }

    @Bean
    public AuthenticationManager authenticationManager(AuthenticationConfiguration config) throws Exception {
        return config.getAuthenticationManager();
    }
}
```

---

*[← Part 74: Event-Driven Advanced](./part-74-event-driven-advanced.md) | [Part 76: Elasticsearch →](./part-76-search-elasticsearch.md)*

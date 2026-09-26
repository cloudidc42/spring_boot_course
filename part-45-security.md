# Part 45: Security Best Practices
## ขั้นตอนที่ 1401-1440

> **ระดับ:** ระดับโลก (World-Class)  
> **เวลาเรียน:** 6-7 ชั่วโมง  
> **เป้าหมาย:** OWASP Top 10 prevention และ security hardening

---

## ขั้นตอนที่ 1401: OWASP Top 10 สำหรับ Spring Boot

```
OWASP Top 10 (2021):
  A01 - Broken Access Control
  A02 - Cryptographic Failures
  A03 - Injection (SQL, NoSQL, Command)
  A04 - Insecure Design
  A05 - Security Misconfiguration
  A06 - Vulnerable Components
  A07 - Identity & Auth Failures
  A08 - Software & Data Integrity Failures
  A09 - Security Logging & Monitoring Failures
  A10 - Server-Side Request Forgery (SSRF)
```

---

## ขั้นตอนที่ 1402: A01 - Broken Access Control Prevention

```java
// ❌ Bad: Resource-based access without ownership check
@GetMapping("/api/v1/orders/{id}")
public OrderResponse getOrder(@PathVariable Long id) {
    return orderService.findById(id);  // Any user can get any order!
}

// ✅ Good: Check ownership
@GetMapping("/api/v1/orders/{id}")
@PreAuthorize("isAuthenticated()")
public OrderResponse getOrder(@PathVariable Long id, Authentication auth) {
    Order order = orderService.findById(id);
    
    if (!order.getUserId().equals(getUserId(auth)) && !hasRole(auth, "ADMIN")) {
        throw new ForbiddenException("Access denied");
    }
    
    return orderMapper.toResponse(order);
}

// ✅ Better: Repository-level filtering
@Query("SELECT o FROM Order o WHERE o.id = :id AND o.userId = :userId")
Optional<Order> findByIdAndUserId(@Param("id") Long id, @Param("userId") Long userId);

// ✅ Method-level security with SpEL
@PreAuthorize("@orderSecurity.canAccess(authentication, #id)")
public OrderResponse findById(Long id) { ... }

@Component("orderSecurity")
public class OrderSecurityExpression {
    public boolean canAccess(Authentication auth, Long orderId) {
        Long userId = ((UserPrincipal) auth.getPrincipal()).getId();
        return orderRepository.existsByIdAndUserId(orderId, userId);
    }
}

// ✅ Admin-only endpoints
@RestController
@RequestMapping("/api/v1/admin")
@PreAuthorize("hasRole('ADMIN')")
public class AdminController { ... }
```

---

## ขั้นตอนที่ 1403: A03 - SQL Injection Prevention

```java
// ❌ NEVER do this - SQL Injection vulnerability
@Query(value = "SELECT * FROM products WHERE name = '" + name + "'", nativeQuery = true)
List<Product> findByName(String name);

// ❌ String concatenation in JPQL
String jpql = "SELECT p FROM Product p WHERE p.name = '" + name + "'";
em.createQuery(jpql);

// ✅ Always use parameterized queries
@Query("SELECT p FROM Product p WHERE p.name = :name")
List<Product> findByName(@Param("name") String name);

// ✅ JPA method names (always safe)
List<Product> findByNameContainingIgnoreCase(String name);

// ✅ Criteria API (safe)
CriteriaBuilder cb = em.getCriteriaBuilder();
CriteriaQuery<Product> query = cb.createQuery(Product.class);
Root<Product> root = query.from(Product.class);
query.where(cb.like(cb.lower(root.get("name")), "%" + name.toLowerCase() + "%"));

// ✅ Native SQL with parameters
@Query(value = "SELECT * FROM products WHERE LOWER(name) LIKE LOWER(:keyword)", nativeQuery = true)
List<Product> searchByKeyword(@Param("keyword") String keyword);
```

---

## ขั้นตอนที่ 1404: A02 - Cryptographic Failures

```java
// ❌ MD5 or SHA1 for passwords (broken!)
MessageDigest.getInstance("MD5").digest(password.getBytes());

// ✅ BCrypt (recommended)
@Bean
public PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder(12);  // cost factor 12
}

// ✅ Sensitive data encryption at rest
@Entity
public class PaymentMethod {
    
    @Convert(converter = AesEncryptedStringConverter.class)
    private String cardNumber;  // Encrypted in DB
    
    @Convert(converter = AesEncryptedStringConverter.class)
    private String cvv;
}

@Converter
public class AesEncryptedStringConverter implements AttributeConverter<String, String> {
    
    private final AesEncryptionService aesService;
    
    @Override
    public String convertToDatabaseColumn(String attribute) {
        return attribute == null ? null : aesService.encrypt(attribute);
    }
    
    @Override
    public String convertToEntityAttribute(String dbData) {
        return dbData == null ? null : aesService.decrypt(dbData);
    }
}

// ✅ TLS for data in transit
server:
  ssl:
    enabled: true
    key-store: classpath:keystore.p12
    key-store-password: ${SSL_KEYSTORE_PASSWORD}
    key-store-type: PKCS12

// ✅ Secrets management
@Value("${app.jwt.secret}")  // From environment variable, not hardcoded
private String jwtSecret;
```

---

## ขั้นตอนที่ 1405: A05 - Security Misconfiguration

```java
// Spring Security hardening
@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    return http
        // Disable unnecessary features
        .csrf(csrf -> csrf.disable())  // Stateless API
        .formLogin(AbstractHttpConfigurer::disable)
        .httpBasic(AbstractHttpConfigurer::disable)
        
        // Security headers
        .headers(headers -> headers
            .frameOptions(frame -> frame.deny())
            .contentTypeOptions(Customizer.withDefaults())
            .httpStrictTransportSecurity(hsts -> hsts
                .maxAgeInSeconds(31536000)
                .includeSubDomains(true)
                .preload(true))
            .referrerPolicy(referrer -> referrer
                .policy(ReferrerPolicyHeaderWriter.ReferrerPolicy.STRICT_ORIGIN_WHEN_CROSS_ORIGIN))
            .contentSecurityPolicy(csp -> csp
                .policyDirectives("default-src 'self'; img-src 'self' data: https:"))
        )
        
        // Session management
        .sessionManagement(session -> session
            .sessionCreationPolicy(SessionCreationPolicy.STATELESS))
        
        .build();
}

// Disable Actuator in production or secure it
management:
  endpoints:
    web:
      exposure:
        include: health, info, prometheus  # Only expose needed endpoints
  endpoint:
    health:
      show-details: never  # Don't expose in production
  server:
    port: 8090  # Separate port for management
```

---

## ขั้นตอนที่ 1406: A07 - Identity & Auth Failures

```java
// ✅ Account lockout after failed attempts
@Service
@RequiredArgsConstructor
public class LoginAttemptService {
    
    private final RedisTemplate<String, Integer> redisTemplate;
    private static final int MAX_ATTEMPTS = 5;
    private static final Duration LOCKOUT_DURATION = Duration.ofMinutes(15);
    
    public void recordFailedLogin(String username) {
        String key = "login:failed:" + username;
        Integer attempts = redisTemplate.opsForValue().get(key);
        
        if (attempts == null) {
            redisTemplate.opsForValue().set(key, 1, LOCKOUT_DURATION);
        } else {
            redisTemplate.opsForValue().increment(key);
        }
    }
    
    public boolean isLocked(String username) {
        String key = "login:failed:" + username;
        Integer attempts = redisTemplate.opsForValue().get(key);
        return attempts != null && attempts >= MAX_ATTEMPTS;
    }
    
    public void clearFailedAttempts(String username) {
        redisTemplate.delete("login:failed:" + username);
    }
}

// ✅ Password strength validation
@Target({ElementType.FIELD, ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = PasswordStrengthValidator.class)
public @interface StrongPassword {
    String message() default "Password must be 8+ characters with upper, lower, digit, special char";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}

@Component
public class PasswordStrengthValidator implements ConstraintValidator<StrongPassword, String> {
    
    private static final Pattern PATTERN = Pattern.compile(
        "^(?=.*[a-z])(?=.*[A-Z])(?=.*\\d)(?=.*[@$!%*?&])[A-Za-z\\d@$!%*?&]{8,}$"
    );
    
    @Override
    public boolean isValid(String password, ConstraintValidatorContext context) {
        return password != null && PATTERN.matcher(password).matches();
    }
}
```

---

## ขั้นตอนที่ 1407: A10 - SSRF Prevention

```java
// ❌ SSRF: Never allow user to control URL
@GetMapping("/proxy")
public String proxy(@RequestParam String url) {
    return restTemplate.getForObject(url, String.class);  // SSRF vulnerability!
}

// ✅ Whitelist allowed domains
@Component
public class SafeUrlValidator {
    
    private static final Set<String> ALLOWED_HOSTS = Set.of(
        "api.trusted-service.com",
        "cdn.myapp.com",
        "images.example.com"
    );
    
    public void validate(String url) {
        try {
            URI uri = URI.create(url);
            String host = uri.getHost();
            
            if (!ALLOWED_HOSTS.contains(host)) {
                throw new IllegalArgumentException("URL host not allowed: " + host);
            }
            
            // Block internal IPs
            InetAddress address = InetAddress.getByName(host);
            if (address.isLoopbackAddress() || address.isSiteLocalAddress()) {
                throw new IllegalArgumentException("Internal URLs are not allowed");
            }
            
        } catch (UnknownHostException e) {
            throw new IllegalArgumentException("Invalid URL");
        }
    }
}
```

---

## ขั้นตอนที่ 1408: Security Scanning in CI/CD

```yaml
# .github/workflows/security.yml
name: Security Scan

on:
  push:
    branches: [main]
  schedule:
    - cron: '0 0 * * 1'  # Weekly

jobs:
  dependency-check:
    name: OWASP Dependency Check
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Run OWASP Dependency Check
        uses: dependency-check/Dependency-Check_Action@main
        with:
          project: 'myapp'
          path: '.'
          format: 'HTML'
          failBuildOnCVSS: 7  # Fail on High severity
      
      - name: Upload report
        uses: actions/upload-artifact@v4
        with:
          name: dependency-check-report
          path: reports/
  
  code-scan:
    name: CodeQL Analysis
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Initialize CodeQL
        uses: github/codeql-action/init@v3
        with:
          languages: java
          queries: security-extended
      
      - name: Build
        run: mvn compile -q
      
      - name: Perform CodeQL Analysis
        uses: github/codeql-action/analyze@v3
  
  secrets-scan:
    name: Secret Scanning
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      
      - name: Scan for secrets
        uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

---

## ขั้นตอนที่ 1409-1440: Security Checklist

```
Application Security Checklist:
  
Authentication:
  ✅ Strong password policy enforced
  ✅ Account lockout after N failed attempts
  ✅ Multi-factor authentication available
  ✅ JWT tokens with short expiry
  ✅ Refresh token rotation
  ✅ Token blacklisting on logout
  
Authorization:
  ✅ Principle of least privilege
  ✅ Resource ownership validation
  ✅ Role-based access control (RBAC)
  ✅ API endpoint protection
  ✅ Method-level security
  
Data Protection:
  ✅ Passwords hashed with bcrypt/argon2
  ✅ Sensitive data encrypted at rest
  ✅ TLS for all communications
  ✅ No sensitive data in logs
  ✅ PII data masked in responses
  
Input Validation:
  ✅ All inputs validated
  ✅ SQL injection prevention (parameterized)
  ✅ XSS prevention (output encoding)
  ✅ File upload validation (type, size)
  ✅ Path traversal prevention
  
Infrastructure:
  ✅ Security headers (HSTS, CSP, etc.)
  ✅ Rate limiting
  ✅ CORS properly configured
  ✅ Error messages don't reveal internals
  ✅ Secrets in environment variables
  
Monitoring:
  ✅ Security events logged
  ✅ Audit trail for sensitive operations
  ✅ Alerts for suspicious activity
  ✅ Dependency vulnerability scanning
  ✅ Container image scanning
```

---

*[← Part 44: Complete Project](./part-44-complete-project.md) | [Part 46: Multi-Tenancy →](./part-46-multitenancy.md)*

# Part 17: JWT Advanced - Token Blacklisting & Password Reset
## ขั้นตอนที่ 431-460

> **ระดับ:** กลาง (Intermediate)  
> **เวลาเรียน:** 4-5 ชั่วโมง  
> **เป้าหมาย:** JWT ขั้นสูง: Blacklisting, Password Reset, Token Rotation

---

## ขั้นตอนที่ 431: Token Blacklisting ด้วย Redis

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class TokenBlacklistService {
    
    private final RedisTemplate<String, String> redisTemplate;
    private final JwtService jwtService;
    
    private static final String BLACKLIST_PREFIX = "blacklist:token:";
    
    public void blacklist(String token) {
        try {
            Date expiration = jwtService.extractExpiration(token);
            long ttl = expiration.getTime() - System.currentTimeMillis();
            
            if (ttl > 0) {
                String key = BLACKLIST_PREFIX + token;
                redisTemplate.opsForValue().set(key, "1", ttl, TimeUnit.MILLISECONDS);
                log.debug("Token blacklisted, TTL: {}ms", ttl);
            }
        } catch (JwtException e) {
            log.warn("Cannot blacklist invalid token");
        }
    }
    
    public boolean isBlacklisted(String token) {
        return Boolean.TRUE.equals(redisTemplate.hasKey(BLACKLIST_PREFIX + token));
    }
}

// Update JwtAuthenticationFilter
@Component
@RequiredArgsConstructor
public class JwtAuthenticationFilter extends OncePerRequestFilter {
    
    private final JwtService jwtService;
    private final UserDetailsService userDetailsService;
    private final TokenBlacklistService tokenBlacklistService;
    
    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain chain)
        throws ServletException, IOException {
        
        String authHeader = request.getHeader("Authorization");
        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            chain.doFilter(request, response);
            return;
        }
        
        String token = authHeader.substring(7);
        
        // Check blacklist first
        if (tokenBlacklistService.isBlacklisted(token)) {
            chain.doFilter(request, response);
            return;
        }
        
        // Continue with validation...
        String username = jwtService.extractUsername(token);
        if (username != null && SecurityContextHolder.getContext().getAuthentication() == null) {
            UserDetails userDetails = userDetailsService.loadUserByUsername(username);
            if (jwtService.isTokenValid(token, userDetails)) {
                UsernamePasswordAuthenticationToken authToken =
                    new UsernamePasswordAuthenticationToken(userDetails, null, userDetails.getAuthorities());
                authToken.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));
                SecurityContextHolder.getContext().setAuthentication(authToken);
            }
        }
        
        chain.doFilter(request, response);
    }
}
```

---

## ขั้นตอนที่ 432: Password Reset Flow

```
1. User requests reset → POST /api/v1/auth/forgot-password
2. System generates token → saves in DB with expiry
3. System sends email with link → /reset-password?token=xxx
4. User submits new password → POST /api/v1/auth/reset-password
5. System validates token → updates password → invalidates token
```

```java
// PasswordResetToken Entity
@Entity
@Table(name = "password_reset_tokens")
@Getter @Setter @Builder @NoArgsConstructor @AllArgsConstructor
public class PasswordResetToken {
    
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
    
    private boolean used = false;
    
    @Column(name = "created_at")
    private LocalDateTime createdAt = LocalDateTime.now();
    
    public boolean isExpired() {
        return LocalDateTime.now().isAfter(expiresAt);
    }
}

// Auth Service - forgot password
@Transactional
public void forgotPassword(String email) {
    userRepository.findByEmail(email).ifPresent(user -> {
        // Delete old tokens
        passwordResetTokenRepository.deleteByUser(user);
        
        // Generate new token
        String token = UUID.randomUUID().toString();
        PasswordResetToken resetToken = PasswordResetToken.builder()
            .token(token)
            .user(user)
            .expiresAt(LocalDateTime.now().plusHours(1))
            .build();
        
        passwordResetTokenRepository.save(resetToken);
        
        // Send email
        emailService.sendPasswordResetEmail(user.getEmail(), token);
    });
    // Don't reveal if email exists
}

@Transactional
public void resetPassword(String token, String newPassword) {
    PasswordResetToken resetToken = passwordResetTokenRepository.findByToken(token)
        .orElseThrow(() -> new AppException("Invalid or expired token", HttpStatus.BAD_REQUEST, "INVALID_TOKEN"));
    
    if (resetToken.isExpired()) {
        throw new AppException("Token expired", HttpStatus.BAD_REQUEST, "TOKEN_EXPIRED");
    }
    if (resetToken.isUsed()) {
        throw new AppException("Token already used", HttpStatus.BAD_REQUEST, "TOKEN_USED");
    }
    
    User user = resetToken.getUser();
    user.setPassword(passwordEncoder.encode(newPassword));
    userRepository.save(user);
    
    resetToken.setUsed(true);
    passwordResetTokenRepository.save(resetToken);
    
    // Invalidate all refresh tokens
    refreshTokenRepository.deleteByUser(user);
    
    log.info("Password reset for user: {}", user.getEmail());
}
```

---

## ขั้นตอนที่ 433: Email Verification

```java
// EmailVerificationToken Entity
@Entity
@Table(name = "email_verification_tokens")
@Getter @Setter @Builder @NoArgsConstructor @AllArgsConstructor
public class EmailVerificationToken {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(unique = true)
    private String token;
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id")
    private User user;
    
    private LocalDateTime expiresAt;
    private boolean verified = false;
    private LocalDateTime createdAt = LocalDateTime.now();
}

// Add emailVerified field to User
@Column(name = "email_verified", nullable = false)
private boolean emailVerified = false;

// Service
@Transactional
public void sendVerificationEmail(Long userId) {
    User user = userRepository.findById(userId).orElseThrow(...);
    if (user.isEmailVerified()) throw new BusinessException("Email already verified");
    
    String token = UUID.randomUUID().toString();
    EmailVerificationToken verificationToken = EmailVerificationToken.builder()
        .token(token)
        .user(user)
        .expiresAt(LocalDateTime.now().plusDays(3))
        .build();
    
    verificationTokenRepository.save(verificationToken);
    emailService.sendVerificationEmail(user.getEmail(), token);
}

@Transactional
public void verifyEmail(String token) {
    EmailVerificationToken verificationToken = verificationTokenRepository.findByToken(token)
        .orElseThrow(() -> new AppException("Invalid token", HttpStatus.BAD_REQUEST, "INVALID_TOKEN"));
    
    if (verificationToken.getExpiresAt().isBefore(LocalDateTime.now())) {
        throw new AppException("Token expired", HttpStatus.BAD_REQUEST, "TOKEN_EXPIRED");
    }
    
    User user = verificationToken.getUser();
    user.setEmailVerified(true);
    userRepository.save(user);
    
    verificationToken.setVerified(true);
    verificationTokenRepository.save(verificationToken);
}
```

---

## ขั้นตอนที่ 434: Change Password

```java
@Transactional
public void changePassword(Long userId, ChangePasswordRequest request) {
    User user = userRepository.findById(userId)
        .orElseThrow(() -> new ResourceNotFoundException("User", "id", userId));
    
    if (!passwordEncoder.matches(request.currentPassword(), user.getPassword())) {
        throw new AppException("Current password is incorrect", HttpStatus.BAD_REQUEST, "WRONG_PASSWORD");
    }
    
    if (request.currentPassword().equals(request.newPassword())) {
        throw new AppException("New password must be different", HttpStatus.BAD_REQUEST, "SAME_PASSWORD");
    }
    
    user.setPassword(passwordEncoder.encode(request.newPassword()));
    userRepository.save(user);
    
    // Invalidate all sessions
    refreshTokenRepository.deleteByUser(user);
}

public record ChangePasswordRequest(
    @NotBlank String currentPassword,
    @NotBlank @Size(min = 8) String newPassword,
    @NotBlank String confirmPassword
) {}
```

---

## ขั้นตอนที่ 435: Rate Limiting for Auth Endpoints

```java
// Bucket4j rate limiting
@Service
@RequiredArgsConstructor
public class RateLimitService {
    
    private final Map<String, Bucket> buckets = new ConcurrentHashMap<>();
    
    public Bucket getLoginBucket(String ip) {
        return buckets.computeIfAbsent("login:" + ip, k -> 
            Bucket.builder()
                .addLimit(Bandwidth.classic(5, Refill.intervally(5, Duration.ofMinutes(1))))
                .build()
        );
    }
    
    public Bucket getForgotPasswordBucket(String email) {
        return buckets.computeIfAbsent("forgot:" + email, k ->
            Bucket.builder()
                .addLimit(Bandwidth.classic(3, Refill.intervally(3, Duration.ofHours(1))))
                .build()
        );
    }
}

// In Auth Controller
@PostMapping("/login")
public ResponseEntity<ApiResponse<AuthResponse>> login(
    @Valid @RequestBody LoginRequest request,
    HttpServletRequest httpRequest
) {
    String ip = httpRequest.getRemoteAddr();
    Bucket bucket = rateLimitService.getLoginBucket(ip);
    
    if (!bucket.tryConsume(1)) {
        return ResponseEntity.status(HttpStatus.TOO_MANY_REQUESTS)
            .body(ApiResponse.error("Too many login attempts. Please try again later."));
    }
    
    return ResponseEntity.ok(ApiResponse.success(authService.login(request)));
}
```

---

## ขั้นตอนที่ 436-460: JWT Security Best Practices

```java
// Security Configuration best practices
@Configuration
public class SecurityBestPractices {
    
    // 1. Always use HTTPS in production
    // 2. Short-lived access tokens (15 min - 1 hour)
    // 3. Longer refresh tokens (7-30 days) with rotation
    // 4. Store refresh tokens in DB (can be revoked)
    // 5. Blacklist compromised access tokens in Redis
    // 6. Use strong secret key (256-bit minimum)
    // 7. Include essential claims only (don't put sensitive data)
    // 8. Validate token on every request
    // 9. Rate limit auth endpoints
    // 10. Log authentication events
    
    // Security Headers
    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http.headers(headers -> headers
            .contentSecurityPolicy(csp -> csp
                .policyDirectives("default-src 'self'; script-src 'self'"))
            .frameOptions(frame -> frame.deny())
            .xssProtection(xss -> xss.disable())  // Modern browsers handle this
            .httpStrictTransportSecurity(hsts -> hsts
                .maxAgeInSeconds(31536000)
                .includeSubDomains(true))
        );
        // ...
        return http.build();
    }
}
```

### Auth Endpoints Summary

```
POST /api/v1/auth/register          - Register new user
POST /api/v1/auth/login             - Login, get JWT
POST /api/v1/auth/refresh           - Refresh access token
POST /api/v1/auth/logout            - Logout (blacklist token)
GET  /api/v1/auth/me                - Get current user
POST /api/v1/auth/forgot-password   - Request password reset
POST /api/v1/auth/reset-password    - Reset password with token
POST /api/v1/auth/change-password   - Change password (authenticated)
POST /api/v1/auth/verify-email      - Verify email address
POST /api/v1/auth/resend-verification - Resend verification email
```

---

*[← Part 16: Spring Security](./part-16-spring-security.md) | [Part 18: File Upload →](./part-18-file-upload.md)*

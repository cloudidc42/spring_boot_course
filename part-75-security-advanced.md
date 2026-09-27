# Part 75: Advanced Spring Security Patterns
## ขั้นตอนที่ 2601-2640

**ระดับ:** ระดับโลก (World-Class)
**เวลาเรียน:** 5-6 ชั่วโมง
**เป้าหมาย:** เรียนรู้ Advanced Security Patterns ด้วย Spring Security ตั้งแต่ Method Security, Custom Expressions, API Key Authentication, mTLS, OAuth2 Token Introspection และ Security Audit Logging

---

## สารบัญ

1. [Method Security พร้อม @PreAuthorize/@PostAuthorize](#method-security)
2. [Custom Security Expressions](#custom-expressions)
3. [API Key Authentication](#api-key)
4. [Mutual TLS (mTLS)](#mtls)
5. [OAuth2 Token Introspection](#token-introspection)
6. [Security Audit Logging](#audit-logging)
7. [CORS Configuration](#cors)

---

## ขั้นตอนที่ 2601: Method Security Setup {#method-security}

### Enable Method Security

Spring Boot 3 ใช้ `@EnableMethodSecurity` แทน `@EnableGlobalMethodSecurity` ที่ deprecated แล้ว

```java
package com.example.config;

import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.method.configuration.EnableMethodSecurity;

@Configuration
@EnableMethodSecurity(
    prePostEnabled = true,     // @PreAuthorize, @PostAuthorize
    securedEnabled = true,     // @Secured
    jsr250Enabled = true       // @RolesAllowed
)
public class MethodSecurityConfig {
    // configuration อยู่ที่ annotation แล้ว ไม่ต้องเพิ่มโค้ด
}
```

### @PreAuthorize ตัวอย่างจริง

```java
package com.example.service;

import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.security.access.prepost.PostAuthorize;
import org.springframework.stereotype.Service;

@Service
public class OrderService {

    // 1. ตรวจสอบ Role
    @PreAuthorize("hasRole('ADMIN')")
    public List<Order> getAllOrders() {
        return orderRepository.findAll();
    }

    // 2. ตรวจสอบ Authority (fine-grained permission)
    @PreAuthorize("hasAuthority('order:read')")
    public Order getOrder(Long orderId) {
        return orderRepository.findById(orderId)
            .orElseThrow(() -> new OrderNotFoundException(orderId));
    }

    // 3. ตรวจสอบ parameter - owner หรือ admin เท่านั้น
    @PreAuthorize("hasRole('ADMIN') or #customerId == authentication.name")
    public List<Order> getOrdersByCustomer(String customerId) {
        return orderRepository.findByCustomerId(customerId);
    }

    // 4. Multiple conditions ด้วย AND
    @PreAuthorize("hasRole('MANAGER') and hasAuthority('order:approve')")
    public Order approveOrder(Long orderId) {
        Order order = getOrder(orderId);
        order.setStatus(OrderStatus.APPROVED);
        return orderRepository.save(order);
    }

    // 5. @PostAuthorize - ตรวจสอบหลัง return
    // ป้องกัน user ดู order ของคนอื่นแม้รู้ orderId
    @PostAuthorize("returnObject.customerId == authentication.name or hasRole('ADMIN')")
    public Order getOrderSecure(Long orderId) {
        return orderRepository.findById(orderId)
            .orElseThrow(() -> new OrderNotFoundException(orderId));
    }

    // 6. ตรวจสอบ object ใน request body
    @PreAuthorize("hasAuthority('order:create') and #request.customerId == authentication.name")
    public Order createOrder(@P("request") CreateOrderRequest request) {
        return orderRepository.save(buildOrder(request));
    }

    // 7. @PostFilter - กรอง collection ที่ return
    @PostFilter("filterObject.customerId == authentication.name or hasRole('ADMIN')")
    public List<Order> getOrdersForDashboard() {
        return orderRepository.findAll();
    }

    // 8. @PreFilter - กรอง input collection
    @PreFilter("filterObject.customerId == authentication.name")
    public List<Order> bulkUpdate(List<Order> orders) {
        return orderRepository.saveAll(orders);
    }
}
```

### @Secured และ @RolesAllowed

```java
package com.example.service;

import jakarta.annotation.security.RolesAllowed;
import org.springframework.security.access.annotation.Secured;
import org.springframework.stereotype.Service;

@Service
public class AdminService {

    // @Secured - Spring-specific, ต้องใส่ ROLE_ prefix
    @Secured({"ROLE_ADMIN", "ROLE_SUPER_ADMIN"})
    public void deleteAllData() {
        // dangerous operation
    }

    // @RolesAllowed - JSR-250 standard, ไม่ต้องใส่ ROLE_ prefix
    @RolesAllowed({"ADMIN", "SUPER_ADMIN"})
    public void exportAllData() {
        // export operation
    }
}
```

---

## ขั้นตอนที่ 2605: Custom Security Expressions {#custom-expressions}

### สร้าง Custom Security Service

Custom Security Service คือ Spring Bean ที่เราสามารถเรียกใน SpEL expression ได้ผ่าน `@beanName.methodName()`

```java
package com.example.security;

import org.springframework.security.core.Authentication;
import org.springframework.stereotype.Component;

@Component("securityService")
public class CustomSecurityService {

    private final OrderRepository orderRepository;
    private final TeamMemberRepository teamMemberRepository;
    private final RateLimitService rateLimitService;

    // ตรวจสอบว่า user เป็นเจ้าของ order
    public boolean isOrderOwner(Authentication authentication, Long orderId) {
        String userId = authentication.getName();
        return orderRepository.findById(orderId)
            .map(order -> order.getCustomerId().equals(userId))
            .orElse(false);
    }

    // ตรวจสอบว่า user อยู่ใน team เดียวกันกับ resource
    public boolean isInSameTeam(Authentication authentication, Long orderId) {
        String userId = authentication.getName();
        return orderRepository.findById(orderId)
            .map(order -> teamMemberRepository.existsByUserIdAndTeamId(
                userId, order.getTeamId()))
            .orElse(false);
    }

    // ตรวจสอบ subscription tier
    public boolean hasSubscriptionLevel(Authentication authentication, String requiredLevel) {
        if (authentication.getPrincipal() instanceof CustomUserDetails user) {
            int required = levelToInt(requiredLevel);
            int current = levelToInt(user.getSubscriptionLevel());
            return current >= required;
        }
        return false;
    }

    // Rate limit check
    public boolean isWithinRateLimit(Authentication authentication) {
        String userId = authentication.getName();
        return rateLimitService.isAllowed(userId, "api-calls", 100, Duration.ofMinutes(1));
    }

    // ตรวจสอบ IP whitelist
    public boolean isFromAllowedIp(Authentication auth, HttpServletRequest request) {
        String ip = getClientIp(request);
        return ipWhitelistService.isAllowed(ip);
    }

    private int levelToInt(String level) {
        return switch (level) {
            case "FREE" -> 0;
            case "BASIC" -> 1;
            case "PRO" -> 2;
            case "ENTERPRISE" -> 3;
            default -> 0;
        };
    }

    private String getClientIp(HttpServletRequest request) {
        String xff = request.getHeader("X-Forwarded-For");
        return xff != null ? xff.split(",")[0].trim() : request.getRemoteAddr();
    }
}
```

### ใช้ Custom Expressions ใน Controller

```java
package com.example.controller;

import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/orders")
public class OrderController {

    // ใช้ custom security service
    @GetMapping("/{orderId}")
    @PreAuthorize("@securityService.isOrderOwner(authentication, #orderId) " +
                  "or hasRole('ADMIN')")
    public Order getOrder(@PathVariable Long orderId) {
        return orderService.findById(orderId);
    }

    // ตรวจสอบ subscription
    @PostMapping("/bulk")
    @PreAuthorize("@securityService.hasSubscriptionLevel(authentication, 'PRO')")
    public List<Order> bulkCreate(@RequestBody List<CreateOrderRequest> requests) {
        return orderService.bulkCreate(requests);
    }

    // ตรวจสอบ team membership
    @PutMapping("/{orderId}/assign")
    @PreAuthorize("@securityService.isInSameTeam(authentication, #orderId) " +
                  "or hasRole('MANAGER')")
    public Order assignOrder(@PathVariable Long orderId,
                              @RequestBody AssignRequest request) {
        return orderService.assign(orderId, request);
    }

    // ตรวจสอบ rate limit
    @GetMapping("/search")
    @PreAuthorize("@securityService.isWithinRateLimit(authentication)")
    public List<Order> searchOrders(@RequestParam String query) {
        return orderService.search(query);
    }
}
```

### PermissionEvaluator สำหรับ @PreAuthorize("hasPermission()")

```java
package com.example.security;

import org.springframework.security.access.PermissionEvaluator;
import org.springframework.security.core.Authentication;
import org.springframework.stereotype.Component;
import java.io.Serializable;

@Component
public class CustomPermissionEvaluator implements PermissionEvaluator {

    private final OrderRepository orderRepository;
    private final DocumentRepository documentRepository;

    @Override
    public boolean hasPermission(Authentication auth, Object targetDomainObject, Object permission) {
        if (auth == null || targetDomainObject == null) return false;

        String targetType = targetDomainObject.getClass().getSimpleName().toUpperCase();
        return hasPermission(auth, targetDomainObject, targetType, permission);
    }

    @Override
    public boolean hasPermission(Authentication auth, Serializable targetId,
                                   String targetType, Object permission) {
        String userId = auth.getName();
        String perm = permission.toString().toUpperCase();

        return switch (targetType.toUpperCase()) {
            case "ORDER" -> checkOrderPermission(userId, (Long) targetId, perm);
            case "DOCUMENT" -> checkDocumentPermission(userId, (Long) targetId, perm);
            default -> false;
        };
    }

    private boolean checkOrderPermission(String userId, Long orderId, String permission) {
        Order order = orderRepository.findById(orderId).orElse(null);
        if (order == null) return false;

        return switch (permission) {
            case "READ" -> order.getCustomerId().equals(userId)
                          || order.getAssigneeId().equals(userId);
            case "WRITE" -> order.getCustomerId().equals(userId)
                           && order.getStatus() == OrderStatus.DRAFT;
            case "DELETE" -> order.getCustomerId().equals(userId)
                            && order.getStatus() == OrderStatus.PENDING;
            default -> false;
        };
    }

    private boolean checkDocumentPermission(String userId, Long docId, String permission) {
        return documentRepository.hasPermission(docId, userId, permission);
    }
}
```

### ใช้ hasPermission()

```java
@GetMapping("/{orderId}/document")
@PreAuthorize("hasPermission(#orderId, 'Order', 'READ')")
public OrderDocument getDocument(@PathVariable Long orderId) {
    return documentService.getOrderDocument(orderId);
}

@DeleteMapping("/{orderId}")
@PreAuthorize("hasPermission(#orderId, 'Order', 'DELETE') or hasRole('ADMIN')")
public void deleteOrder(@PathVariable Long orderId) {
    orderService.delete(orderId);
}
```

---

## ขั้นตอนที่ 2610: API Key Authentication {#api-key}

### API Key Filter

```java
package com.example.security;

import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.web.filter.OncePerRequestFilter;
import java.io.IOException;

public class ApiKeyAuthenticationFilter extends OncePerRequestFilter {

    private static final String API_KEY_HEADER = "X-API-Key";
    private final ApiKeyService apiKeyService;

    public ApiKeyAuthenticationFilter(ApiKeyService apiKeyService) {
        this.apiKeyService = apiKeyService;
    }

    @Override
    protected void doFilterInternal(
            HttpServletRequest request,
            HttpServletResponse response,
            FilterChain filterChain) throws ServletException, IOException {

        String apiKey = request.getHeader(API_KEY_HEADER);

        if (apiKey == null || apiKey.isBlank()) {
            filterChain.doFilter(request, response);
            return;
        }

        ApiKeyDetails keyDetails = apiKeyService.validateApiKey(apiKey);

        if (keyDetails == null) {
            response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
            response.setContentType("application/json");
            response.getWriter().write("""
                {"error": "INVALID_API_KEY", "message": "Invalid or expired API key"}
                """);
            return;
        }

        ApiKeyAuthentication authentication = new ApiKeyAuthentication(
            keyDetails.getClientId(),
            keyDetails.getAuthorities(),
            apiKey
        );
        authentication.setAuthenticated(true);

        SecurityContextHolder.getContext().setAuthentication(authentication);
        apiKeyService.recordUsage(keyDetails.getKeyId(), request.getRequestURI());

        filterChain.doFilter(request, response);
    }
}
```

### API Key Authentication Token

```java
package com.example.security;

import org.springframework.security.authentication.AbstractAuthenticationToken;
import org.springframework.security.core.GrantedAuthority;
import java.util.Collection;

public class ApiKeyAuthentication extends AbstractAuthenticationToken {

    private final String clientId;
    private final String apiKey;

    public ApiKeyAuthentication(String clientId,
                                  Collection<? extends GrantedAuthority> authorities,
                                  String apiKey) {
        super(authorities);
        this.clientId = clientId;
        this.apiKey = apiKey;
    }

    @Override
    public Object getCredentials() { return apiKey; }

    @Override
    public Object getPrincipal() { return clientId; }
}
```

### API Key Service พร้อม Secure Hashing

```java
package com.example.security;

import org.springframework.cache.annotation.Cacheable;
import org.springframework.stereotype.Service;
import java.security.MessageDigest;
import java.util.Base64;

@Service
public class ApiKeyService {

    private final ApiKeyRepository apiKeyRepository;
    private final ApiKeyUsageRepository usageRepository;

    @Cacheable(value = "api-keys", key = "#apiKey", unless = "#result == null")
    public ApiKeyDetails validateApiKey(String apiKey) {
        // Hash ก่อน lookup - ไม่เก็บ plain text ใน database
        String hashedKey = hashApiKey(apiKey);

        return apiKeyRepository.findByKeyHash(hashedKey)
            .filter(key -> !key.isExpired())
            .filter(key -> !key.isRevoked())
            .map(key -> ApiKeyDetails.builder()
                .keyId(key.getId())
                .clientId(key.getClientId())
                .clientName(key.getClientName())
                .authorities(key.getPermissions().stream()
                    .map(p -> (GrantedAuthority) () -> p)
                    .collect(Collectors.toList()))
                .rateLimit(key.getRateLimit())
                .build())
            .orElse(null);
    }

    // สร้าง API key ใหม่ - return raw key ครั้งเดียวเท่านั้น
    public GeneratedApiKey generateApiKey(String clientId, List<String> permissions) {
        String rawKey = "sk_live_" + generateSecureRandom(32);
        String hashedKey = hashApiKey(rawKey);

        ApiKey apiKey = ApiKey.builder()
            .clientId(clientId)
            .keyHash(hashedKey)
            .permissions(permissions)
            .createdAt(Instant.now())
            .expiresAt(Instant.now().plus(Duration.ofDays(365)))
            .revoked(false)
            .build();

        apiKeyRepository.save(apiKey);

        return new GeneratedApiKey(rawKey, apiKey.getId(),
            "IMPORTANT: Save this key - it will not be shown again");
    }

    public void revokeApiKey(Long keyId) {
        apiKeyRepository.findById(keyId).ifPresent(key -> {
            key.setRevoked(true);
            key.setRevokedAt(Instant.now());
            apiKeyRepository.save(key);
        });
    }

    public void recordUsage(Long keyId, String endpoint) {
        usageRepository.save(ApiKeyUsage.builder()
            .keyId(keyId)
            .endpoint(endpoint)
            .timestamp(Instant.now())
            .build());
    }

    private String hashApiKey(String apiKey) {
        try {
            MessageDigest digest = MessageDigest.getInstance("SHA-256");
            byte[] hash = digest.digest(apiKey.getBytes(java.nio.charset.StandardCharsets.UTF_8));
            return Base64.getEncoder().encodeToString(hash);
        } catch (Exception e) {
            throw new RuntimeException("Failed to hash API key", e);
        }
    }

    private String generateSecureRandom(int length) {
        byte[] bytes = new byte[length];
        new java.security.SecureRandom().nextBytes(bytes);
        return Base64.getUrlEncoder().withoutPadding().encodeToString(bytes);
    }
}
```

### API Key Management Endpoints

```java
package com.example.controller;

@RestController
@RequestMapping("/api/admin/api-keys")
@PreAuthorize("hasRole('ADMIN')")
public class ApiKeyManagementController {

    private final ApiKeyService apiKeyService;

    @PostMapping
    public ResponseEntity<GeneratedApiKey> createApiKey(
            @RequestBody CreateApiKeyRequest request) {
        GeneratedApiKey generated = apiKeyService.generateApiKey(
            request.getClientId(), request.getPermissions());
        return ResponseEntity.status(HttpStatus.CREATED).body(generated);
    }

    @DeleteMapping("/{keyId}")
    public ResponseEntity<Void> revokeApiKey(@PathVariable Long keyId) {
        apiKeyService.revokeApiKey(keyId);
        return ResponseEntity.noContent().build();
    }

    @GetMapping
    public Page<ApiKeyInfo> listApiKeys(@RequestParam String clientId, Pageable pageable) {
        return apiKeyService.listKeys(clientId, pageable);
    }
}
```

---

## ขั้นตอนที่ 2615: Mutual TLS (mTLS) {#mtls}

### Application Properties สำหรับ mTLS

```yaml
# application.yml
server:
  port: 8443
  ssl:
    enabled: true
    key-store: classpath:server-keystore.p12
    key-store-password: ${SSL_KEYSTORE_PASSWORD}
    key-store-type: PKCS12
    key-alias: server
    # mTLS - ต้องการ client certificate
    client-auth: need  # need = required, want = optional
    trust-store: classpath:trusted-clients.p12
    trust-store-password: ${SSL_TRUSTSTORE_PASSWORD}
    trust-store-type: PKCS12
```

### mTLS Authentication Filter

```java
package com.example.security;

import jakarta.servlet.FilterChain;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.springframework.web.filter.OncePerRequestFilter;
import java.security.cert.X509Certificate;

public class MutualTlsAuthenticationFilter extends OncePerRequestFilter {

    private final ClientCertificateService certService;

    @Override
    protected void doFilterInternal(
            HttpServletRequest request,
            HttpServletResponse response,
            FilterChain filterChain) throws Exception {

        // ดึง client certificate จาก request
        X509Certificate[] certs = (X509Certificate[]) request.getAttribute(
            "jakarta.servlet.request.X509Certificate");

        if (certs == null || certs.length == 0) {
            filterChain.doFilter(request, response);
            return;
        }

        X509Certificate clientCert = certs[0];

        try {
            clientCert.checkValidity();

            String subjectDN = clientCert.getSubjectX500Principal().getName();
            String clientId = extractCN(subjectDN);

            ClientDetails client = certService.validateCertificate(
                clientCert.getSerialNumber().toString(), clientId);

            if (client == null) {
                response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
                response.getWriter().write("{\"error\": \"CERTIFICATE_NOT_TRUSTED\"}");
                return;
            }

            MtlsAuthentication auth = new MtlsAuthentication(
                clientId, client.getAuthorities(), clientCert);
            auth.setAuthenticated(true);
            SecurityContextHolder.getContext().setAuthentication(auth);

            filterChain.doFilter(request, response);

        } catch (java.security.cert.CertificateExpiredException e) {
            response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
            response.getWriter().write("{\"error\": \"CERTIFICATE_EXPIRED\"}");
        }
    }

    private String extractCN(String subjectDN) {
        for (String part : subjectDN.split(",")) {
            part = part.trim();
            if (part.startsWith("CN=")) return part.substring(3);
        }
        return subjectDN;
    }
}
```

### Certificate Generation Script

```bash
#!/bin/bash
# generate-mtls-certs.sh

# CA Certificate
openssl req -new -x509 -keyout ca-key.pem -out ca-cert.pem -days 3650 \
  -subj "/CN=Internal CA/O=Company/C=TH" -passout pass:capassword

# Server Certificate
openssl req -new -keyout server-key.pem -out server-req.pem \
  -subj "/CN=api.company.com/O=Company/C=TH"
openssl x509 -req -in server-req.pem -CA ca-cert.pem -CAkey ca-key.pem \
  -CAcreateserial -out server-cert.pem -days 365 -passin pass:capassword

# Client Certificate (สำหรับ service-a)
openssl req -new -keyout service-a-key.pem -out service-a-req.pem \
  -subj "/CN=service-a/O=Company/C=TH"
openssl x509 -req -in service-a-req.pem -CA ca-cert.pem -CAkey ca-key.pem \
  -CAcreateserial -out service-a-cert.pem -days 365 -passin pass:capassword

# สร้าง PKCS12 Keystores
openssl pkcs12 -export -in server-cert.pem -inkey server-key.pem \
  -out server-keystore.p12 -name server -passout pass:serverpass

# Trust Store บน Server (เก็บ CA cert เพื่อ trust client certs)
keytool -import -alias ca -file ca-cert.pem \
  -keystore trusted-clients.p12 -storetype PKCS12 \
  -storepass trustpass -noprompt

echo "Certificates generated successfully!"
```

---

## ขั้นตอนที่ 2620: OAuth2 Token Introspection {#token-introspection}

### Resource Server Configuration

```java
package com.example.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/actuator/health/**").permitAll()
                .requestMatchers("/api/public/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer(oauth2 -> oauth2
                .opaqueToken(opaque -> opaque
                    .introspectionUri("https://auth-server.com/oauth2/introspect")
                    .introspectionClientCredentials("resource-server", "secret")
                )
            );
        return http.build();
    }
}
```

```yaml
# application.yml
spring:
  security:
    oauth2:
      resourceserver:
        opaquetoken:
          introspection-uri: https://auth-server.com/oauth2/introspect
          client-id: resource-server-client
          client-secret: ${OAUTH2_CLIENT_SECRET}
```

### Caching Token Introspector

```java
package com.example.security;

import org.springframework.security.oauth2.core.OAuth2AuthenticatedPrincipal;
import org.springframework.security.oauth2.server.resource.introspection.OpaqueTokenIntrospector;
import org.springframework.security.oauth2.server.resource.introspection.SpringOpaqueTokenIntrospector;
import org.springframework.cache.annotation.Cacheable;
import org.springframework.stereotype.Component;

// Cache introspection results เพื่อลด calls ไปยัง auth server
@Component
public class CachingTokenIntrospector implements OpaqueTokenIntrospector {

    private final SpringOpaqueTokenIntrospector delegate;
    private final UserEnrichmentService enrichmentService;

    @Override
    @Cacheable(
        value = "token-introspection",
        key = "#token",
        unless = "#result == null"
    )
    public OAuth2AuthenticatedPrincipal introspect(String token) {
        // ตรวจสอบ token กับ authorization server
        OAuth2AuthenticatedPrincipal principal = delegate.introspect(token);

        // Enrich ด้วย user info จาก database
        String userId = principal.getName();
        UserProfile profile = enrichmentService.getProfile(userId);

        // สร้าง enriched principal
        Map<String, Object> attributes = new HashMap<>(principal.getAttributes());
        attributes.put("profile", profile);
        attributes.put("permissions", enrichmentService.getPermissions(userId));

        return new DefaultOAuth2AuthenticatedPrincipal(
            userId, attributes, principal.getAuthorities());
    }
}
```

### ดึง Claims จาก Token

```java
package com.example.security;

import org.springframework.security.core.Authentication;
import org.springframework.security.oauth2.core.OAuth2AuthenticatedPrincipal;
import org.springframework.stereotype.Component;

@Component
public class SecurityContextHelper {

    public String getCurrentUserId(Authentication auth) {
        if (auth.getPrincipal() instanceof OAuth2AuthenticatedPrincipal p) {
            return p.getName();
        }
        return auth.getName();
    }

    public String getTenantId(Authentication auth) {
        if (auth.getPrincipal() instanceof OAuth2AuthenticatedPrincipal p) {
            return p.getAttribute("tenant_id");
        }
        return null;
    }

    public List<String> getScopes(Authentication auth) {
        if (auth.getPrincipal() instanceof OAuth2AuthenticatedPrincipal p) {
            Object scopes = p.getAttribute("scope");
            if (scopes instanceof String s) return Arrays.asList(s.split(" "));
            if (scopes instanceof List<?> list) {
                return list.stream().map(Object::toString).collect(Collectors.toList());
            }
        }
        return Collections.emptyList();
    }

    public boolean hasScope(Authentication auth, String requiredScope) {
        return getScopes(auth).contains(requiredScope);
    }
}
```

---

## ขั้นตอนที่ 2625: Security Audit Logging {#audit-logging}

### Spring Security Event Listeners

```java
package com.example.audit;

import org.springframework.context.event.EventListener;
import org.springframework.security.authentication.event.*;
import org.springframework.stereotype.Component;

@Component
@Slf4j
public class SecurityEventListener {

    private final AuditService auditService;
    private final LoginAttemptService loginAttemptService;

    // Login สำเร็จ
    @EventListener
    public void onSuccess(AuthenticationSuccessEvent event) {
        String username = event.getAuthentication().getName();

        auditService.log(AuditEvent.builder()
            .eventType("AUTH_LOGIN_SUCCESS")
            .userId(username)
            .severity(AuditSeverity.INFO)
            .message("User authenticated successfully")
            .build());

        // Reset failed attempts
        loginAttemptService.loginSucceeded(username);
    }

    // Login ล้มเหลว
    @EventListener
    public void onFailure(AbstractAuthenticationFailureEvent event) {
        String username = event.getAuthentication().getName();

        auditService.log(AuditEvent.builder()
            .eventType("AUTH_LOGIN_FAILURE")
            .userId(username)
            .severity(AuditSeverity.WARNING)
            .message("Authentication failed: " + event.getException().getMessage())
            .build());

        loginAttemptService.loginFailed(username);
    }

    // Authorization denied
    @EventListener
    public void onAccessDenied(
            org.springframework.security.access.event.AuthorizationFailureEvent event) {
        String username = event.getAuthentication().getName();

        auditService.log(AuditEvent.builder()
            .eventType("AUTHZ_ACCESS_DENIED")
            .userId(username)
            .severity(AuditSeverity.WARNING)
            .message("Access denied: " + event.getAccessDeniedException().getMessage())
            .build());
    }
}
```

### @Audited Custom Annotation

```java
package com.example.audit;

import java.lang.annotation.*;

@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Audited {
    String action();
    String resource() default "";
    AuditSeverity severity() default AuditSeverity.INFO;
    boolean logResult() default false;
}
```

### Audit AOP Aspect

```java
package com.example.audit;

import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.stereotype.Component;

@Aspect
@Component
@Slf4j
public class AuditAspect {

    private final AuditService auditService;

    @Around("@annotation(audited)")
    public Object auditMethod(ProceedingJoinPoint pjp, Audited audited) throws Throwable {
        String userId = getCurrentUserId();
        long start = System.currentTimeMillis();

        try {
            Object result = pjp.proceed();

            auditService.log(AuditEvent.builder()
                .eventType(audited.action())
                .userId(userId)
                .resource(audited.resource())
                .severity(audited.severity())
                .status("SUCCESS")
                .durationMs(System.currentTimeMillis() - start)
                .build());

            return result;

        } catch (Exception e) {
            auditService.log(AuditEvent.builder()
                .eventType(audited.action() + "_FAILED")
                .userId(userId)
                .resource(audited.resource())
                .severity(AuditSeverity.ERROR)
                .status("FAILURE")
                .errorMessage(e.getMessage())
                .durationMs(System.currentTimeMillis() - start)
                .build());
            throw e;
        }
    }

    private String getCurrentUserId() {
        var auth = SecurityContextHolder.getContext().getAuthentication();
        return auth != null ? auth.getName() : "anonymous";
    }
}
```

### Audit Service

```java
package com.example.audit;

import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Service;

@Service
@Slf4j
public class AuditService {

    private final AuditLogRepository auditLogRepository;
    private final KafkaTemplate<String, AuditEvent> kafkaTemplate;

    public void log(AuditEvent event) {
        enrichWithRequestContext(event);

        AuditLog auditLog = AuditLog.builder()
            .eventType(event.getEventType())
            .userId(event.getUserId())
            .resource(event.getResource())
            .severity(event.getSeverity().name())
            .status(event.getStatus())
            .message(event.getMessage())
            .errorMessage(event.getErrorMessage())
            .ipAddress(event.getIpAddress())
            .userAgent(event.getUserAgent())
            .durationMs(event.getDurationMs())
            .timestamp(Instant.now())
            .build();

        auditLogRepository.save(auditLog);

        // ส่งไป Kafka สำหรับ SIEM
        kafkaTemplate.send("security-audit-events", event.getUserId(), event);

        // Alert สำหรับ critical events
        if (event.getSeverity() == AuditSeverity.CRITICAL) {
            log.error("CRITICAL SECURITY EVENT: {}", event);
        }
    }

    private void enrichWithRequestContext(AuditEvent event) {
        try {
            var attrs = (ServletRequestAttributes)
                RequestContextHolder.getRequestAttributes();
            if (attrs != null) {
                HttpServletRequest request = attrs.getRequest();
                String xff = request.getHeader("X-Forwarded-For");
                event.setIpAddress(xff != null ? xff.split(",")[0].trim()
                                               : request.getRemoteAddr());
                event.setUserAgent(request.getHeader("User-Agent"));
                event.setRequestPath(request.getRequestURI());
            }
        } catch (Exception ignored) {}
    }
}
```

### ใช้ @Audited Annotation

```java
package com.example.service;

import com.example.audit.Audited;
import com.example.audit.AuditSeverity;

@Service
public class UserManagementService {

    @Audited(action = "USER_ROLE_CHANGE", resource = "user",
             severity = AuditSeverity.WARNING)
    public void changeUserRole(String userId, String newRole) {
        userRepository.updateRole(userId, newRole);
    }

    @Audited(action = "USER_DELETE", resource = "user",
             severity = AuditSeverity.CRITICAL)
    public void deleteUser(String userId) {
        userRepository.deleteById(userId);
    }

    @Audited(action = "DATA_EXPORT", resource = "customer-data")
    public byte[] exportData(String customerId) {
        return dataExportService.export(customerId);
    }
}
```

---

## ขั้นตอนที่ 2630: CORS Configuration {#cors}

### Comprehensive CORS Setup

```java
package com.example.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.cors.CorsConfiguration;
import org.springframework.web.cors.CorsConfigurationSource;
import org.springframework.web.cors.UrlBasedCorsConfigurationSource;

@Configuration
public class CorsConfig {

    @Bean
    public CorsConfigurationSource corsConfigurationSource() {
        // Public API CORS
        CorsConfiguration publicConfig = new CorsConfiguration();
        publicConfig.setAllowedOrigins(List.of(
            "https://app.company.com",
            "https://admin.company.com"
        ));
        publicConfig.setAllowedOriginPatterns(List.of("https://*.company.com"));
        publicConfig.setAllowedMethods(Arrays.asList(
            "GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS"));
        publicConfig.setAllowedHeaders(Arrays.asList(
            "Authorization", "Content-Type", "X-Requested-With",
            "X-API-Key", "X-Request-ID", "Accept", "Origin"));
        publicConfig.setExposedHeaders(Arrays.asList(
            "X-Request-ID", "X-Total-Count", "X-Rate-Limit-Remaining"));
        publicConfig.setAllowCredentials(true);
        publicConfig.setMaxAge(3600L);

        // Internal API CORS (stricter)
        CorsConfiguration internalConfig = new CorsConfiguration();
        internalConfig.setAllowedOrigins(List.of("https://internal.company.com"));
        internalConfig.setAllowedMethods(Arrays.asList("GET", "POST", "PUT", "DELETE"));
        internalConfig.setAllowedHeaders(List.of("*"));
        internalConfig.setAllowCredentials(true);

        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/api/**", publicConfig);
        source.registerCorsConfiguration("/internal/**", internalConfig);

        return source;
    }
}
```

### Complete SecurityFilterChain

```java
package com.example.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.web.SecurityFilterChain;
import org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter;

@Configuration
@EnableWebSecurity
public class SecurityFilterChainConfig {

    private final ApiKeyAuthenticationFilter apiKeyFilter;
    private final CorsConfigurationSource corsSource;

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .cors(cors -> cors.configurationSource(corsSource))
            .csrf(csrf -> csrf.disable())
            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .headers(headers -> headers
                .contentTypeOptions(Customizer.withDefaults())
                .frameOptions(frame -> frame.deny())
                .httpStrictTransportSecurity(hsts -> hsts
                    .includeSubDomains(true)
                    .maxAgeInSeconds(31536000))
                .contentSecurityPolicy(csp -> csp
                    .policyDirectives("default-src 'self'"))
            )
            .authorizeHttpRequests(auth -> auth
                .requestMatchers(HttpMethod.OPTIONS, "/**").permitAll()
                .requestMatchers("/actuator/health/**", "/actuator/info").permitAll()
                .requestMatchers("/api/v1/auth/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .requestMatchers(HttpMethod.DELETE, "/api/**")
                    .hasAnyRole("ADMIN", "MANAGER")
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer(oauth2 -> oauth2
                .opaqueToken(opaque -> opaque
                    .introspectionUri("https://auth.company.com/introspect")
                    .introspectionClientCredentials("resource-server", "secret"))
            )
            .addFilterBefore(apiKeyFilter, UsernamePasswordAuthenticationFilter.class)
            .exceptionHandling(ex -> ex
                .authenticationEntryPoint((req, res, e) -> {
                    res.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
                    res.setContentType(MediaType.APPLICATION_JSON_VALUE);
                    res.getWriter().write(
                        "{\"error\":\"UNAUTHORIZED\",\"message\":\"Authentication required\"}");
                })
                .accessDeniedHandler((req, res, e) -> {
                    res.setStatus(HttpServletResponse.SC_FORBIDDEN);
                    res.setContentType(MediaType.APPLICATION_JSON_VALUE);
                    res.getWriter().write(
                        "{\"error\":\"FORBIDDEN\",\"message\":\"Insufficient permissions\"}");
                })
            );

        return http.build();
    }
}
```

---

## ขั้นตอนที่ 2635: Security Testing

### Integration Tests

```java
package com.example.security;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.security.test.context.support.WithMockUser;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.http.HttpMethod;

import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@SpringBootTest
@AutoConfigureMockMvc
class SecurityIntegrationTest {

    @Autowired
    private MockMvc mockMvc;

    @Test
    void shouldDenyAccessWithoutAuth() throws Exception {
        mockMvc.perform(get("/api/orders"))
            .andExpect(status().isUnauthorized());
    }

    @Test
    @WithMockUser(roles = "USER")
    void shouldAllowAuthenticatedUser() throws Exception {
        mockMvc.perform(get("/api/orders"))
            .andExpect(status().isOk());
    }

    @Test
    @WithMockUser(roles = "USER")
    void shouldDenyNonAdminToAdminEndpoint() throws Exception {
        mockMvc.perform(get("/api/admin/users"))
            .andExpect(status().isForbidden());
    }

    @Test
    @WithMockUser(roles = "ADMIN")
    void shouldAllowAdminAccess() throws Exception {
        mockMvc.perform(get("/api/admin/users"))
            .andExpect(status().isOk());
    }

    @Test
    void shouldAcceptValidApiKey() throws Exception {
        mockMvc.perform(get("/api/orders")
                .header("X-API-Key", "sk_live_validtestkey"))
            .andExpect(status().isOk());
    }

    @Test
    void shouldRejectInvalidApiKey() throws Exception {
        mockMvc.perform(get("/api/orders")
                .header("X-API-Key", "invalid-key"))
            .andExpect(status().isUnauthorized())
            .andExpect(jsonPath("$.error").value("INVALID_API_KEY"));
    }

    @Test
    @WithMockUser(username = "user1", roles = "USER")
    void shouldDenyAccessToOtherUsersOrder() throws Exception {
        // Order 999 belongs to user2
        mockMvc.perform(get("/api/orders/999"))
            .andExpect(status().isForbidden());
    }

    @Test
    void shouldHandleCorsPreflightRequest() throws Exception {
        mockMvc.perform(options("/api/orders")
                .header("Origin", "https://app.company.com")
                .header("Access-Control-Request-Method", "GET"))
            .andExpect(status().isOk())
            .andExpect(header().string(
                "Access-Control-Allow-Origin", "https://app.company.com"))
            .andExpect(header().exists("Access-Control-Allow-Methods"));
    }

    @Test
    void shouldRejectCorsFromUnknownOrigin() throws Exception {
        mockMvc.perform(options("/api/orders")
                .header("Origin", "https://evil-site.com")
                .header("Access-Control-Request-Method", "GET"))
            .andExpect(header().doesNotExist("Access-Control-Allow-Origin"));
    }
}
```

### Method Security Tests

```java
package com.example.security;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.security.access.AccessDeniedException;
import org.springframework.security.test.context.support.WithMockUser;

import static org.assertj.core.api.Assertions.assertThatThrownBy;

@SpringBootTest
class MethodSecurityTest {

    @Autowired
    private OrderService orderService;

    @Test
    @WithMockUser(roles = "USER")
    void shouldDenyNonAdminFromGettingAllOrders() {
        assertThatThrownBy(() -> orderService.getAllOrders())
            .isInstanceOf(AccessDeniedException.class);
    }

    @Test
    @WithMockUser(username = "customer1", roles = "USER")
    void shouldAllowUserToGetTheirOwnOrders() {
        List<Order> orders = orderService.getOrdersByCustomer("customer1");
        assertThat(orders).allMatch(o -> o.getCustomerId().equals("customer1"));
    }

    @Test
    @WithMockUser(username = "customer1", roles = "USER")
    void shouldDenyUserFromGettingOthersOrders() {
        assertThatThrownBy(() -> orderService.getOrdersByCustomer("customer2"))
            .isInstanceOf(AccessDeniedException.class);
    }
}
```

---

## ขั้นตอนที่ 2638: Security Headers และ Best Practices

### Security Headers Configuration

```java
package com.example.config;

@Configuration
public class SecurityHeadersConfig {

    @Bean
    public SecurityFilterChain headersChain(HttpSecurity http) throws Exception {
        http.headers(headers -> headers
            // X-Content-Type-Options
            .contentTypeOptions(Customizer.withDefaults())

            // X-Frame-Options
            .frameOptions(frame -> frame.deny())

            // Strict-Transport-Security
            .httpStrictTransportSecurity(hsts -> hsts
                .includeSubDomains(true)
                .maxAgeInSeconds(31536000)  // 1 year
                .preload(true))

            // Content-Security-Policy
            .contentSecurityPolicy(csp -> csp
                .policyDirectives(
                    "default-src 'self'; " +
                    "script-src 'self' 'unsafe-inline' https://cdn.company.com; " +
                    "style-src 'self' 'unsafe-inline'; " +
                    "img-src 'self' data: https:; " +
                    "font-src 'self' https://fonts.googleapis.com; " +
                    "connect-src 'self' https://api.company.com; " +
                    "frame-ancestors 'none'; " +
                    "form-action 'self'"
                ))

            // Permissions-Policy
            .permissionsPolicy(pp -> pp
                .policy("geolocation=(), microphone=(), camera=()"))

            // Referrer-Policy
            .referrerPolicy(rp -> rp
                .policy(ReferrerPolicyHeaderWriter.ReferrerPolicy.STRICT_ORIGIN_WHEN_CROSS_ORIGIN))
        );

        return http.build();
    }
}
```

### Login Attempt Tracking

```java
package com.example.security;

import org.springframework.cache.annotation.CacheEvict;
import org.springframework.cache.annotation.CachePut;
import org.springframework.cache.annotation.Cacheable;
import org.springframework.stereotype.Service;

@Service
public class LoginAttemptService {

    private static final int MAX_ATTEMPTS = 5;
    private static final Duration LOCKOUT_DURATION = Duration.ofMinutes(15);

    private final StringRedisTemplate redisTemplate;

    public void loginSucceeded(String username) {
        // Reset failed attempt counter
        redisTemplate.delete("login_attempts:" + username);
    }

    public void loginFailed(String username) {
        String key = "login_attempts:" + username;
        Long attempts = redisTemplate.opsForValue().increment(key);

        if (attempts != null && attempts == 1) {
            // ตั้ง expiry ครั้งแรกที่ fail
            redisTemplate.expire(key, LOCKOUT_DURATION);
        }

        if (attempts != null && attempts >= MAX_ATTEMPTS) {
            // Lock account
            redisTemplate.opsForValue().set(
                "account_locked:" + username, "true", LOCKOUT_DURATION);

            auditService.log(AuditEvent.builder()
                .eventType("ACCOUNT_LOCKED")
                .userId(username)
                .severity(AuditSeverity.WARNING)
                .message("Account locked after " + attempts + " failed attempts")
                .build());
        }
    }

    public boolean isLocked(String username) {
        return Boolean.TRUE.toString().equals(
            redisTemplate.opsForValue().get("account_locked:" + username));
    }

    public int getFailedAttempts(String username) {
        String val = redisTemplate.opsForValue().get("login_attempts:" + username);
        return val != null ? Integer.parseInt(val) : 0;
    }
}
```

---

## สรุปสิ่งที่เรียนรู้

ใน Part นี้เราได้เรียนรู้:

1. **Method Security** - `@PreAuthorize`, `@PostAuthorize`, `@Secured`, `@RolesAllowed` และ filtering annotations
2. **Custom Security Expressions** - สร้าง `@securityService` bean สำหรับ complex authorization logic
3. **PermissionEvaluator** - ใช้ `hasPermission()` สำหรับ domain object security
4. **API Key Authentication** - การ implement API key ที่ปลอดภัยด้วย SHA-256 hashing
5. **Mutual TLS (mTLS)** - ตั้งค่า client certificate authentication สำหรับ service-to-service
6. **OAuth2 Token Introspection** - ตรวจสอบ token กับ authorization server พร้อม caching
7. **Security Audit Logging** - บันทึก security events ด้วย AOP และส่งไป Kafka
8. **CORS Configuration** - ตั้งค่า CORS อย่างปลอดภัยสำหรับ multiple origins
9. **Security Headers** - CSP, HSTS, Referrer-Policy, Permissions-Policy
10. **Login Attempt Tracking** - ป้องกัน brute force ด้วย Redis

### Security Best Practices Checklist
- ✓ ใช้ `@PreAuthorize` แทน if-else ใน business logic
- ✓ Hash API keys ด้วย SHA-256 ก่อนบันทึกลง database
- ✓ Audit log ทุก sensitive operations
- ✓ กำหนด CORS origins แบบ explicit - ไม่ใช้ `*` ใน production
- ✓ เปิด HTTPS + HSTS สำหรับทุก production environment
- ✓ กำหนด Content-Security-Policy เพื่อป้องกัน XSS
- ✓ ใช้ Principle of Least Privilege - grant minimal permissions
- ✓ Test security ด้วย integration tests เสมอ
- ✓ Monitor failed login attempts และ lock account เมื่อ threshold ถึง

---

*[← Part 74: Event-Driven Advanced](./part-74-event-driven-advanced.md) | [Part 76: Elasticsearch →](./part-76-search-elasticsearch.md)*

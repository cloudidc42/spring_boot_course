# Part 87: Reactive Security
## ขั้นตอนที่ 3081-3120

**ระดับ:** ระดับโลก (World-Class)
**เวลาเรียน:** 6-8 ชั่วโมง
**เป้าหมาย:** เรียนรู้ Spring Security ร่วมกับ WebFlux (Reactive Stack) สำหรับการสร้าง secure reactive applications ที่ปรับขนาดได้

---

## ขั้นตอนที่ 3081: Spring Security WebFlux คืออะไร?

Spring Security ในโลก reactive ทำงานต่างจาก servlet-based security โดยใช้ `SecurityWebFilterChain` แทน `HttpSecurity` และรองรับ non-blocking authentication

### เพิ่ม Dependencies

```xml
<!-- pom.xml -->
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-webflux</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-security</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
    </dependency>
    <dependency>
        <groupId>io.jsonwebtoken</groupId>
        <artifactId>jjwt-api</artifactId>
        <version>0.11.5</version>
    </dependency>
    <dependency>
        <groupId>io.jsonwebtoken</groupId>
        <artifactId>jjwt-impl</artifactId>
        <version>0.11.5</version>
        <scope>runtime</scope>
    </dependency>
    <dependency>
        <groupId>io.jsonwebtoken</groupId>
        <artifactId>jjwt-jackson</artifactId>
        <version>0.11.5</version>
        <scope>runtime</scope>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-r2dbc</artifactId>
    </dependency>
</dependencies>
```

---

## ขั้นตอนที่ 3082: ReactiveUserDetailsService

`ReactiveUserDetailsService` เป็น reactive version ของ `UserDetailsService`

```java
// model/User.java
package com.example.reactivesecurity.model;

import lombok.Data;
import lombok.Builder;
import org.springframework.data.annotation.Id;
import org.springframework.data.relational.core.mapping.Table;

import java.util.Set;
import java.util.HashSet;

@Data
@Builder
@Table("users")
public class User {
    @Id
    private Long id;
    private String username;
    private String password;
    private String email;
    private boolean enabled;
    private boolean accountNonExpired;
    private boolean accountNonLocked;
    private boolean credentialsNonExpired;
}
```

```java
// model/UserRole.java
package com.example.reactivesecurity.model;

import lombok.Data;
import org.springframework.data.annotation.Id;
import org.springframework.data.relational.core.mapping.Table;

@Data
@Table("user_roles")
public class UserRole {
    @Id
    private Long id;
    private Long userId;
    private String role;
}
```

```java
// repository/UserRepository.java
package com.example.reactivesecurity.repository;

import com.example.reactivesecurity.model.User;
import org.springframework.data.r2dbc.repository.R2dbcRepository;
import reactor.core.publisher.Mono;

public interface UserRepository extends R2dbcRepository<User, Long> {
    Mono<User> findByUsername(String username);
    Mono<User> findByEmail(String email);
    Mono<Boolean> existsByUsername(String username);
}
```

```java
// repository/UserRoleRepository.java
package com.example.reactivesecurity.repository;

import com.example.reactivesecurity.model.UserRole;
import org.springframework.data.r2dbc.repository.R2dbcRepository;
import reactor.core.publisher.Flux;

public interface UserRoleRepository extends R2dbcRepository<UserRole, Long> {
    Flux<UserRole> findByUserId(Long userId);
}
```

```java
// service/ReactiveUserDetailsServiceImpl.java
package com.example.reactivesecurity.service;

import com.example.reactivesecurity.repository.UserRepository;
import com.example.reactivesecurity.repository.UserRoleRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.core.userdetails.ReactiveUserDetailsService;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.core.userdetails.UsernameNotFoundException;
import org.springframework.stereotype.Service;
import reactor.core.publisher.Mono;

import java.util.stream.Collectors;

@Slf4j
@Service
@RequiredArgsConstructor
public class ReactiveUserDetailsServiceImpl implements ReactiveUserDetailsService {

    private final UserRepository userRepository;
    private final UserRoleRepository userRoleRepository;

    @Override
    public Mono<UserDetails> findByUsername(String username) {
        return userRepository.findByUsername(username)
            .switchIfEmpty(Mono.error(
                new UsernameNotFoundException("User not found: " + username)))
            .flatMap(user -> userRoleRepository.findByUserId(user.getId())
                .collectList()
                .map(roles -> {
                    var authorities = roles.stream()
                        .map(role -> new SimpleGrantedAuthority("ROLE_" + role.getRole()))
                        .collect(Collectors.toList());
                    
                    return org.springframework.security.core.userdetails.User.builder()
                        .username(user.getUsername())
                        .password(user.getPassword())
                        .disabled(!user.isEnabled())
                        .accountExpired(!user.isAccountNonExpired())
                        .accountLocked(!user.isAccountNonLocked())
                        .credentialsExpired(!user.isCredentialsNonExpired())
                        .authorities(authorities)
                        .build();
                }))
            .doOnSuccess(u -> log.debug("Loaded user: {}", u.getUsername()))
            .doOnError(e -> log.error("Error loading user: {}", e.getMessage()));
    }
}
```

---

## ขั้นตอนที่ 3083: JWT Authentication ใน Reactive Stack

การสร้าง JWT-based authentication สำหรับ WebFlux

```java
// security/JwtTokenProvider.java
package com.example.reactivesecurity.security;

import io.jsonwebtoken.*;
import io.jsonwebtoken.security.Keys;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.security.core.Authentication;
import org.springframework.security.core.GrantedAuthority;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.stereotype.Component;

import javax.crypto.SecretKey;
import java.nio.charset.StandardCharsets;
import java.util.Collection;
import java.util.Date;
import java.util.List;
import java.util.stream.Collectors;

@Slf4j
@Component
public class JwtTokenProvider {

    @Value("${jwt.secret:mySecretKeyThatIsAtLeast256BitsLongForHS256Algorithm}")
    private String jwtSecret;

    @Value("${jwt.expiration:86400000}") // 24 ชั่วโมง
    private long jwtExpiration;

    @Value("${jwt.refresh-expiration:604800000}") // 7 วัน
    private long refreshExpiration;

    private SecretKey getSigningKey() {
        return Keys.hmacShaKeyFor(jwtSecret.getBytes(StandardCharsets.UTF_8));
    }

    // สร้าง access token
    public String generateAccessToken(Authentication authentication) {
        UserDetails userDetails = (UserDetails) authentication.getPrincipal();
        return generateAccessToken(userDetails.getUsername(),
            userDetails.getAuthorities());
    }

    public String generateAccessToken(String username,
            Collection<? extends GrantedAuthority> authorities) {
        List<String> roles = authorities.stream()
            .map(GrantedAuthority::getAuthority)
            .collect(Collectors.toList());

        return Jwts.builder()
            .setSubject(username)
            .claim("roles", roles)
            .claim("type", "ACCESS")
            .setIssuedAt(new Date())
            .setExpiration(new Date(System.currentTimeMillis() + jwtExpiration))
            .signWith(getSigningKey(), SignatureAlgorithm.HS256)
            .compact();
    }

    // สร้าง refresh token
    public String generateRefreshToken(String username) {
        return Jwts.builder()
            .setSubject(username)
            .claim("type", "REFRESH")
            .setIssuedAt(new Date())
            .setExpiration(new Date(System.currentTimeMillis() + refreshExpiration))
            .signWith(getSigningKey(), SignatureAlgorithm.HS256)
            .compact();
    }

    // อ่าน username จาก token
    public String getUsernameFromToken(String token) {
        return parseClaims(token).getSubject();
    }

    // อ่าน roles จาก token
    @SuppressWarnings("unchecked")
    public List<String> getRolesFromToken(String token) {
        return (List<String>) parseClaims(token).get("roles");
    }

    // ตรวจสอบ token type
    public String getTokenType(String token) {
        return (String) parseClaims(token).get("type");
    }

    // ตรวจสอบว่า token valid
    public boolean validateToken(String token) {
        try {
            parseClaims(token);
            return true;
        } catch (SecurityException e) {
            log.error("Invalid JWT signature: {}", e.getMessage());
        } catch (MalformedJwtException e) {
            log.error("Invalid JWT token: {}", e.getMessage());
        } catch (ExpiredJwtException e) {
            log.error("Expired JWT token: {}", e.getMessage());
        } catch (UnsupportedJwtException e) {
            log.error("Unsupported JWT token: {}", e.getMessage());
        } catch (IllegalArgumentException e) {
            log.error("JWT claims string is empty: {}", e.getMessage());
        }
        return false;
    }

    private Claims parseClaims(String token) {
        return Jwts.parserBuilder()
            .setSigningKey(getSigningKey())
            .build()
            .parseClaimsJws(token)
            .getBody();
    }
}
```

---

## ขั้นตอนที่ 3084: Reactive JWT Authentication Filter

สร้าง filter สำหรับตรวจสอบ JWT token ใน reactive chain

```java
// security/JwtAuthenticationWebFilter.java
package com.example.reactivesecurity.security;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.http.HttpHeaders;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.core.context.ReactiveSecurityContextHolder;
import org.springframework.stereotype.Component;
import org.springframework.util.StringUtils;
import org.springframework.web.server.ServerWebExchange;
import org.springframework.web.server.WebFilter;
import org.springframework.web.server.WebFilterChain;
import reactor.core.publisher.Mono;

import java.util.List;
import java.util.stream.Collectors;

@Slf4j
@Component
@RequiredArgsConstructor
public class JwtAuthenticationWebFilter implements WebFilter {

    private final JwtTokenProvider jwtTokenProvider;

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, WebFilterChain chain) {
        String token = extractTokenFromRequest(exchange);
        
        if (token != null && jwtTokenProvider.validateToken(token)) {
            // ตรวจสอบว่าเป็น access token
            if (!"ACCESS".equals(jwtTokenProvider.getTokenType(token))) {
                log.warn("Rejected non-access token");
                return chain.filter(exchange);
            }
            
            String username = jwtTokenProvider.getUsernameFromToken(token);
            List<String> roles = jwtTokenProvider.getRolesFromToken(token);
            
            var authorities = roles.stream()
                .map(SimpleGrantedAuthority::new)
                .collect(Collectors.toList());
            
            var authentication = new UsernamePasswordAuthenticationToken(
                username, null, authorities);
            
            log.debug("Authenticated user: {} with roles: {}", username, roles);
            
            // ใส่ authentication เข้า reactive security context
            return chain.filter(exchange)
                .contextWrite(ReactiveSecurityContextHolder
                    .withAuthentication(authentication));
        }
        
        return chain.filter(exchange);
    }

    private String extractTokenFromRequest(ServerWebExchange exchange) {
        String bearerToken = exchange.getRequest()
            .getHeaders()
            .getFirst(HttpHeaders.AUTHORIZATION);
        
        if (StringUtils.hasText(bearerToken) && bearerToken.startsWith("Bearer ")) {
            return bearerToken.substring(7);
        }
        return null;
    }
}
```

---

## ขั้นตอนที่ 3085: SecurityWebFilterChain Configuration

การ configure security rules สำหรับ WebFlux

```java
// config/ReactiveSecurityConfig.java
package com.example.reactivesecurity.config;

import com.example.reactivesecurity.security.JwtAuthenticationWebFilter;
import com.example.reactivesecurity.service.ReactiveUserDetailsServiceImpl;
import lombok.RequiredArgsConstructor;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.HttpMethod;
import org.springframework.security.authentication.ReactiveAuthenticationManager;
import org.springframework.security.authentication.UserDetailsRepositoryReactiveAuthenticationManager;
import org.springframework.security.config.annotation.method.configuration.EnableReactiveMethodSecurity;
import org.springframework.security.config.annotation.web.reactive.EnableWebFluxSecurity;
import org.springframework.security.config.web.server.SecurityWebFiltersOrder;
import org.springframework.security.config.web.server.ServerHttpSecurity;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.security.web.server.SecurityWebFilterChain;
import org.springframework.security.web.server.context.NoOpServerSecurityContextRepository;
import org.springframework.web.cors.CorsConfiguration;
import org.springframework.web.cors.reactive.CorsConfigurationSource;
import org.springframework.web.cors.reactive.UrlBasedCorsConfigurationSource;

import java.util.Arrays;
import java.util.List;

@Configuration
@EnableWebFluxSecurity
@EnableReactiveMethodSecurity  // เปิดใช้ @PreAuthorize, @PostAuthorize
@RequiredArgsConstructor
public class ReactiveSecurityConfig {

    private final JwtAuthenticationWebFilter jwtAuthenticationWebFilter;
    private final ReactiveUserDetailsServiceImpl userDetailsService;

    @Bean
    public SecurityWebFilterChain securityWebFilterChain(ServerHttpSecurity http) {
        return http
            // ปิด CSRF เพราะใช้ JWT (stateless)
            .csrf(ServerHttpSecurity.CsrfSpec::disable)
            // กำหนด CORS
            .cors(cors -> cors.configurationSource(corsConfigurationSource()))
            // Stateless session - ไม่เก็บ session
            .securityContextRepository(NoOpServerSecurityContextRepository.getInstance())
            // กำหนด authorization rules
            .authorizeExchange(exchanges -> exchanges
                // Public endpoints
                .pathMatchers(HttpMethod.POST, "/api/auth/login", "/api/auth/register").permitAll()
                .pathMatchers(HttpMethod.POST, "/api/auth/refresh").permitAll()
                .pathMatchers(HttpMethod.GET, "/api/public/**").permitAll()
                .pathMatchers("/actuator/health", "/actuator/info").permitAll()
                .pathMatchers("/v3/api-docs/**", "/swagger-ui/**").permitAll()
                // Admin only
                .pathMatchers("/api/admin/**").hasRole("ADMIN")
                // User management
                .pathMatchers(HttpMethod.DELETE, "/api/users/**").hasAnyRole("ADMIN", "MODERATOR")
                // Authenticated users
                .pathMatchers("/api/**").authenticated()
                // Default - deny
                .anyExchange().denyAll()
            )
            // เพิ่ม JWT filter ก่อน authentication
            .addFilterBefore(jwtAuthenticationWebFilter,
                SecurityWebFiltersOrder.AUTHENTICATION)
            // Exception handling
            .exceptionHandling(ex -> ex
                .authenticationEntryPoint((exchange, e) -> {
                    exchange.getResponse().setStatusCode(
                        org.springframework.http.HttpStatus.UNAUTHORIZED);
                    return exchange.getResponse().setComplete();
                })
                .accessDeniedHandler((exchange, e) -> {
                    exchange.getResponse().setStatusCode(
                        org.springframework.http.HttpStatus.FORBIDDEN);
                    return exchange.getResponse().setComplete();
                })
            )
            .build();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder(12);
    }

    @Bean
    public ReactiveAuthenticationManager authenticationManager() {
        var authManager = new UserDetailsRepositoryReactiveAuthenticationManager(
            userDetailsService);
        authManager.setPasswordEncoder(passwordEncoder());
        return authManager;
    }

    @Bean
    public CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration configuration = new CorsConfiguration();
        configuration.setAllowedOriginPatterns(List.of(
            "http://localhost:3000",
            "https://*.example.com"
        ));
        configuration.setAllowedMethods(Arrays.asList(
            "GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS"));
        configuration.setAllowedHeaders(Arrays.asList(
            "Authorization", "Content-Type", "X-Requested-With", "Accept"));
        configuration.setExposedHeaders(List.of("Authorization", "X-Total-Count"));
        configuration.setAllowCredentials(true);
        configuration.setMaxAge(3600L);

        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/api/**", configuration);
        return source;
    }
}
```

---

## ขั้นตอนที่ 3086: Auth Controller แบบ Reactive

```java
// dto/LoginRequest.java
package com.example.reactivesecurity.dto;

import lombok.Data;
import javax.validation.constraints.NotBlank;

@Data
public class LoginRequest {
    @NotBlank
    private String username;
    @NotBlank
    private String password;
}
```

```java
// dto/AuthResponse.java
package com.example.reactivesecurity.dto;

import lombok.Builder;
import lombok.Data;

@Data
@Builder
public class AuthResponse {
    private String accessToken;
    private String refreshToken;
    private String tokenType;
    private long expiresIn;
    private String username;
    private java.util.List<String> roles;
}
```

```java
// controller/AuthController.java
package com.example.reactivesecurity.controller;

import com.example.reactivesecurity.dto.AuthResponse;
import com.example.reactivesecurity.dto.LoginRequest;
import com.example.reactivesecurity.security.JwtTokenProvider;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.http.HttpStatus;
import org.springframework.security.authentication.ReactiveAuthenticationManager;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.Authentication;
import org.springframework.security.core.GrantedAuthority;
import org.springframework.web.bind.annotation.*;
import reactor.core.publisher.Mono;

import java.util.stream.Collectors;

@Slf4j
@RestController
@RequestMapping("/api/auth")
@RequiredArgsConstructor
public class AuthController {

    private final ReactiveAuthenticationManager authenticationManager;
    private final JwtTokenProvider jwtTokenProvider;

    @PostMapping("/login")
    public Mono<AuthResponse> login(@RequestBody LoginRequest request) {
        return authenticationManager
            .authenticate(new UsernamePasswordAuthenticationToken(
                request.getUsername(), request.getPassword()))
            .map(authentication -> {
                String accessToken = jwtTokenProvider.generateAccessToken(authentication);
                String refreshToken = jwtTokenProvider.generateRefreshToken(
                    authentication.getName());
                
                var roles = authentication.getAuthorities().stream()
                    .map(GrantedAuthority::getAuthority)
                    .collect(Collectors.toList());
                
                return AuthResponse.builder()
                    .accessToken(accessToken)
                    .refreshToken(refreshToken)
                    .tokenType("Bearer")
                    .expiresIn(86400)
                    .username(authentication.getName())
                    .roles(roles)
                    .build();
            })
            .doOnSuccess(r -> log.info("User logged in: {}", r.getUsername()))
            .doOnError(e -> log.error("Login failed: {}", e.getMessage()));
    }

    @PostMapping("/refresh")
    public Mono<AuthResponse> refresh(@RequestHeader("X-Refresh-Token") String refreshToken) {
        if (!jwtTokenProvider.validateToken(refreshToken)) {
            return Mono.error(new RuntimeException("Invalid refresh token"));
        }
        
        if (!"REFRESH".equals(jwtTokenProvider.getTokenType(refreshToken))) {
            return Mono.error(new RuntimeException("Not a refresh token"));
        }
        
        String username = jwtTokenProvider.getUsernameFromToken(refreshToken);
        List<String> roles = jwtTokenProvider.getRolesFromToken(refreshToken);
        
        var authorities = roles.stream()
            .map(org.springframework.security.core.authority.SimpleGrantedAuthority::new)
            .collect(Collectors.toList());
        
        String newAccessToken = jwtTokenProvider.generateAccessToken(username, authorities);
        
        return Mono.just(AuthResponse.builder()
            .accessToken(newAccessToken)
            .tokenType("Bearer")
            .expiresIn(86400)
            .username(username)
            .roles(roles)
            .build());
    }

    @GetMapping("/me")
    public Mono<AuthResponse> getCurrentUser(Authentication authentication) {
        if (authentication == null) {
            return Mono.error(new RuntimeException("Not authenticated"));
        }
        
        var roles = authentication.getAuthorities().stream()
            .map(GrantedAuthority::getAuthority)
            .collect(Collectors.toList());
        
        return Mono.just(AuthResponse.builder()
            .username(authentication.getName())
            .roles(roles)
            .build());
    }
}
```

---

## ขั้นตอนที่ 3087: @PreAuthorize กับ Reactive Methods

การใช้ method-level security ใน reactive methods

```java
// service/UserService.java
package com.example.reactivesecurity.service;

import com.example.reactivesecurity.model.User;
import com.example.reactivesecurity.repository.UserRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.security.access.prepost.PostAuthorize;
import org.springframework.security.core.context.ReactiveSecurityContextHolder;
import org.springframework.stereotype.Service;
import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;

@Slf4j
@Service
@RequiredArgsConstructor
public class UserService {

    private final UserRepository userRepository;

    // ต้องเป็น ADMIN จึงจะดูรายการ users ทั้งหมดได้
    @PreAuthorize("hasRole('ADMIN')")
    public Flux<User> getAllUsers() {
        return userRepository.findAll()
            .doOnNext(u -> log.debug("Returning user: {}", u.getUsername()));
    }

    // ดูข้อมูล user ตัวเองหรือเป็น ADMIN
    @PreAuthorize("hasRole('ADMIN') or #username == authentication.name")
    public Mono<User> getUserByUsername(String username) {
        return userRepository.findByUsername(username);
    }

    // ต้องเป็น ADMIN หรือ MODERATOR
    @PreAuthorize("hasAnyRole('ADMIN', 'MODERATOR')")
    public Mono<Void> deleteUser(Long userId) {
        return userRepository.deleteById(userId)
            .doOnSuccess(v -> log.info("Deleted user: {}", userId));
    }

    // Post authorize - ตรวจสอบหลังจากได้ผลลัพธ์
    @PostAuthorize("returnObject.map(u -> u.username == authentication.name or hasRole('ADMIN')).defaultIfEmpty(true)")
    public Mono<User> getUserById(Long id) {
        return userRepository.findById(id);
    }

    // ใช้ SpEL expressions ซับซ้อน
    @PreAuthorize("@securityService.canAccessUserData(#userId, authentication)")
    public Mono<User> getSecureUserData(Long userId) {
        return userRepository.findById(userId);
    }

    // ดึง current user จาก security context
    public Mono<User> getCurrentUser() {
        return ReactiveSecurityContextHolder.getContext()
            .map(ctx -> ctx.getAuthentication().getName())
            .flatMap(userRepository::findByUsername);
    }

    // อัปเดตข้อมูลตัวเองหรือ ADMIN อัปเดตให้
    @PreAuthorize("hasRole('ADMIN') or #userId == @userService.getCurrentUserId(authentication)")
    public Mono<User> updateUser(Long userId, User updatedUser) {
        return userRepository.findById(userId)
            .flatMap(user -> {
                user.setEmail(updatedUser.getEmail());
                return userRepository.save(user);
            });
    }

    public Mono<Long> getCurrentUserId(
            org.springframework.security.core.Authentication authentication) {
        return userRepository.findByUsername(authentication.getName())
            .map(User::getId);
    }
}
```

```java
// service/SecurityService.java
package com.example.reactivesecurity.service;

import lombok.RequiredArgsConstructor;
import org.springframework.security.core.Authentication;
import org.springframework.security.core.GrantedAuthority;
import org.springframework.stereotype.Service;
import reactor.core.publisher.Mono;

@Service
@RequiredArgsConstructor
public class SecurityService {

    private final UserRepository userRepository;

    // Custom security check ที่ใช้กับ @PreAuthorize
    public Mono<Boolean> canAccessUserData(Long userId,
            Authentication authentication) {
        // Admin สามารถเข้าถึงได้ทุก user
        boolean isAdmin = authentication.getAuthorities().stream()
            .map(GrantedAuthority::getAuthority)
            .anyMatch(a -> a.equals("ROLE_ADMIN"));
        
        if (isAdmin) return Mono.just(true);
        
        // User ดูข้อมูลตัวเองได้
        return userRepository.findById(userId)
            .map(user -> user.getUsername().equals(authentication.getName()))
            .defaultIfEmpty(false);
    }
}
```

---

## ขั้นตอนที่ 3088: Reactive OAuth2 Resource Server

การ configure Spring Security เป็น OAuth2 Resource Server ใน reactive stack

```java
// config/OAuth2ResourceServerConfig.java
package com.example.reactivesecurity.config;

import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Profile;
import org.springframework.security.config.annotation.web.reactive.EnableWebFluxSecurity;
import org.springframework.security.config.web.server.ServerHttpSecurity;
import org.springframework.security.oauth2.jwt.*;
import org.springframework.security.oauth2.server.resource.authentication.JwtGrantedAuthoritiesConverter;
import org.springframework.security.oauth2.server.resource.authentication.ReactiveJwtAuthenticationConverter;
import org.springframework.security.web.server.SecurityWebFilterChain;
import reactor.core.publisher.Flux;

import java.security.interfaces.RSAPublicKey;
import java.io.IOException;
import java.security.spec.X509EncodedKeySpec;
import java.util.Base64;

@Slf4j
@Configuration
@Profile("oauth2")
public class OAuth2ResourceServerConfig {

    @Value("${spring.security.oauth2.resourceserver.jwt.issuer-uri}")
    private String issuerUri;

    @Bean
    public SecurityWebFilterChain oauth2SecurityFilterChain(ServerHttpSecurity http) {
        return http
            .csrf(ServerHttpSecurity.CsrfSpec::disable)
            .authorizeExchange(exchanges -> exchanges
                .pathMatchers("/api/public/**").permitAll()
                .pathMatchers("/actuator/**").permitAll()
                .anyExchange().authenticated()
            )
            // configure OAuth2 Resource Server
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(jwt -> jwt
                    .jwtAuthenticationConverter(jwtAuthenticationConverter())
                )
            )
            .build();
    }

    @Bean
    public ReactiveJwtAuthenticationConverter jwtAuthenticationConverter() {
        ReactiveJwtAuthenticationConverter converter = new ReactiveJwtAuthenticationConverter();
        
        // กำหนด roles จาก JWT claims
        JwtGrantedAuthoritiesConverter authoritiesConverter =
            new JwtGrantedAuthoritiesConverter();
        authoritiesConverter.setAuthorityPrefix("ROLE_");
        authoritiesConverter.setAuthoritiesClaimName("roles");
        
        converter.setJwtGrantedAuthoritiesConverter(
            jwt -> Flux.fromIterable(authoritiesConverter.convert(jwt)));
        
        return converter;
    }

    @Bean
    public ReactiveJwtDecoder reactiveJwtDecoder() {
        // ใช้ JWKS URI จาก authorization server
        return ReactiveJwtDecoders.fromIssuerLocation(issuerUri);
    }
}
```

---

## ขั้นตอนที่ 3089: Security Context ใน Reactive Chains

การเข้าถึง security context ใน reactive chain

```java
// handler/SecureWebHandler.java
package com.example.reactivesecurity.handler;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.security.core.Authentication;
import org.springframework.security.core.context.ReactiveSecurityContextHolder;
import org.springframework.security.core.context.SecurityContext;
import org.springframework.stereotype.Component;
import org.springframework.web.reactive.function.server.ServerRequest;
import org.springframework.web.reactive.function.server.ServerResponse;
import reactor.core.publisher.Mono;

import java.security.Principal;
import java.util.Map;

@Slf4j
@Component
@RequiredArgsConstructor
public class SecureWebHandler {

    // ดึง principal จาก request
    public Mono<ServerResponse> handleWithPrincipal(ServerRequest request) {
        return request.principal()
            .cast(Authentication.class)
            .flatMap(auth -> {
                log.info("User {} is accessing resource", auth.getName());
                return ServerResponse.ok()
                    .bodyValue(Map.of(
                        "user", auth.getName(),
                        "roles", auth.getAuthorities()
                    ));
            });
    }

    // ดึง authentication จาก ReactiveSecurityContextHolder
    public Mono<ServerResponse> handleWithSecurityContext(ServerRequest request) {
        return ReactiveSecurityContextHolder.getContext()
            .map(SecurityContext::getAuthentication)
            .flatMap(auth -> {
                String username = auth.getName();
                log.info("Authenticated user: {}", username);
                return ServerResponse.ok()
                    .bodyValue(Map.of("username", username));
            })
            .switchIfEmpty(ServerResponse.status(401).build());
    }

    // ส่ง security context ไปยัง downstream operations
    public Mono<String> processWithUserContext(String data) {
        return ReactiveSecurityContextHolder.getContext()
            .map(ctx -> ctx.getAuthentication().getName())
            .flatMap(username -> {
                // ทำงานใน context ของ user
                log.info("Processing data for user: {}", username);
                return Mono.just("Processed: " + data + " by " + username);
            });
    }

    // Propagate security context ใน parallel operations
    public Mono<Map<String, Object>> parallelSecureOperations(ServerRequest request) {
        Mono<String> userMono = ReactiveSecurityContextHolder.getContext()
            .map(ctx -> ctx.getAuthentication().getName());
        
        Mono<String> dataA = Mono.just("DataA");
        Mono<String> dataB = Mono.just("DataB");
        
        return Mono.zip(userMono, dataA, dataB)
            .map(tuple -> Map.of(
                "user", tuple.getT1(),
                "resultA", tuple.getT2(),
                "resultB", tuple.getT3()
            ));
    }
}
```

---

## ขั้นตอนที่ 3090: Reactive Security Events

การจัดการ security events ใน reactive stack

```java
// listener/ReactiveSecurityEventListener.java
package com.example.reactivesecurity.listener;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.context.event.EventListener;
import org.springframework.security.authentication.event.*;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.stereotype.Component;
import reactor.core.publisher.Sinks;

import java.time.LocalDateTime;
import java.util.Map;

@Slf4j
@Component
@RequiredArgsConstructor
public class ReactiveSecurityEventListener {

    // Sink สำหรับ broadcast security events
    private final Sinks.Many<Map<String, Object>> securityEventSink =
        Sinks.many().multicast().onBackpressureBuffer();

    @EventListener
    public void handleAuthenticationSuccess(AuthenticationSuccessEvent event) {
        String username = event.getAuthentication().getName();
        log.info("Authentication success for user: {}", username);
        
        securityEventSink.tryEmitNext(Map.of(
            "type", "LOGIN_SUCCESS",
            "username", username,
            "timestamp", LocalDateTime.now().toString()
        ));
    }

    @EventListener
    public void handleAuthenticationFailure(AbstractAuthenticationFailureEvent event) {
        String username = event.getAuthentication().getName();
        String reason = event.getException().getMessage();
        
        log.warn("Authentication failure for user: {} - Reason: {}", username, reason);
        
        securityEventSink.tryEmitNext(Map.of(
            "type", "LOGIN_FAILURE",
            "username", username,
            "reason", reason,
            "timestamp", LocalDateTime.now().toString()
        ));
    }

    @EventListener
    public void handleLogoutSuccess(
            org.springframework.security.authentication.event.LogoutSuccessEvent event) {
        log.info("Logout for user: {}", event.getAuthentication().getName());
    }

    // ส่ง security events แบบ Server-Sent Events
    public reactor.core.publisher.Flux<Map<String, Object>> getSecurityEvents() {
        return securityEventSink.asFlux();
    }
}
```

---

## ขั้นตอนที่ 3091: Rate Limiting และ Brute Force Protection

```java
// security/RateLimitingWebFilter.java
package com.example.reactivesecurity.security;

import lombok.extern.slf4j.Slf4j;
import org.springframework.http.HttpStatus;
import org.springframework.stereotype.Component;
import org.springframework.web.server.ServerWebExchange;
import org.springframework.web.server.WebFilter;
import org.springframework.web.server.WebFilterChain;
import reactor.core.publisher.Mono;

import java.time.Duration;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.AtomicInteger;

@Slf4j
@Component
public class RateLimitingWebFilter implements WebFilter {

    // เก็บ attempt count ต่อ IP
    private final Map<String, AttemptRecord> attemptCache = new ConcurrentHashMap<>();
    
    private static final int MAX_ATTEMPTS = 5;
    private static final Duration LOCKOUT_DURATION = Duration.ofMinutes(15);

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, WebFilterChain chain) {
        String path = exchange.getRequest().getPath().value();
        
        // ตรวจสอบเฉพาะ login endpoint
        if (!path.equals("/api/auth/login")) {
            return chain.filter(exchange);
        }
        
        String clientIp = getClientIp(exchange);
        
        if (isBlocked(clientIp)) {
            log.warn("Blocked request from IP: {}", clientIp);
            exchange.getResponse().setStatusCode(HttpStatus.TOO_MANY_REQUESTS);
            exchange.getResponse().getHeaders()
                .add("Retry-After", String.valueOf(LOCKOUT_DURATION.getSeconds()));
            return exchange.getResponse().setComplete();
        }
        
        return chain.filter(exchange)
            .then(Mono.fromRunnable(() -> {
                int statusCode = exchange.getResponse().getStatusCode() != null
                    ? exchange.getResponse().getStatusCode().value() : 200;
                if (statusCode == 401) {
                    recordFailedAttempt(clientIp);
                } else if (statusCode == 200) {
                    resetAttempts(clientIp);
                }
            }));
    }

    private String getClientIp(ServerWebExchange exchange) {
        String xForwardedFor = exchange.getRequest().getHeaders()
            .getFirst("X-Forwarded-For");
        if (xForwardedFor != null && !xForwardedFor.isEmpty()) {
            return xForwardedFor.split(",")[0].trim();
        }
        var remoteAddress = exchange.getRequest().getRemoteAddress();
        return remoteAddress != null ? remoteAddress.getAddress().getHostAddress() : "unknown";
    }

    private boolean isBlocked(String ip) {
        AttemptRecord record = attemptCache.get(ip);
        if (record == null) return false;
        
        // ตรวจสอบว่า lockout หมดอายุหรือยัง
        if (record.isExpired()) {
            attemptCache.remove(ip);
            return false;
        }
        
        return record.getCount() >= MAX_ATTEMPTS;
    }

    private void recordFailedAttempt(String ip) {
        attemptCache.compute(ip, (k, v) -> {
            if (v == null || v.isExpired()) {
                return new AttemptRecord(1, System.currentTimeMillis() +
                    LOCKOUT_DURATION.toMillis());
            }
            v.increment();
            return v;
        });
        
        AttemptRecord record = attemptCache.get(ip);
        if (record.getCount() >= MAX_ATTEMPTS) {
            log.warn("IP {} has been blocked after {} failed attempts", ip, MAX_ATTEMPTS);
        }
    }

    private void resetAttempts(String ip) {
        attemptCache.remove(ip);
    }

    private static class AttemptRecord {
        private final AtomicInteger count;
        private final long expiresAt;

        AttemptRecord(int initialCount, long expiresAt) {
            this.count = new AtomicInteger(initialCount);
            this.expiresAt = expiresAt;
        }

        void increment() { count.incrementAndGet(); }
        int getCount() { return count.get(); }
        boolean isExpired() { return System.currentTimeMillis() > expiresAt; }
    }
}
```

---

## ขั้นตอนที่ 3092: Custom Reactive Authentication

```java
// security/ApiKeyAuthenticationWebFilter.java
package com.example.reactivesecurity.security;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.core.context.ReactiveSecurityContextHolder;
import org.springframework.security.web.authentication.preauth.PreAuthenticatedAuthenticationToken;
import org.springframework.stereotype.Component;
import org.springframework.web.server.ServerWebExchange;
import org.springframework.web.server.WebFilter;
import org.springframework.web.server.WebFilterChain;
import reactor.core.publisher.Mono;

import java.util.List;
import java.util.Map;

@Slf4j
@Component
@RequiredArgsConstructor
public class ApiKeyAuthenticationWebFilter implements WebFilter {

    // API keys mapping (ในระบบจริงควรเก็บใน database)
    private static final Map<String, List<String>> API_KEYS = Map.of(
        "service-key-abc123", List.of("ROLE_SERVICE", "ROLE_API"),
        "admin-key-xyz789", List.of("ROLE_ADMIN", "ROLE_SERVICE", "ROLE_API")
    );

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, WebFilterChain chain) {
        String apiKey = exchange.getRequest().getHeaders().getFirst("X-API-Key");
        
        if (apiKey != null && API_KEYS.containsKey(apiKey)) {
            List<String> roles = API_KEYS.get(apiKey);
            var authorities = roles.stream()
                .map(SimpleGrantedAuthority::new)
                .toList();
            
            var auth = new PreAuthenticatedAuthenticationToken(
                "api-client", apiKey, authorities);
            auth.setAuthenticated(true);
            
            log.debug("API Key authentication successful, roles: {}", roles);
            
            return chain.filter(exchange)
                .contextWrite(ReactiveSecurityContextHolder.withAuthentication(auth));
        }
        
        return chain.filter(exchange);
    }
}
```

---

## ขั้นตอนที่ 3093: Testing Reactive Security

```java
// test/ReactiveSecurityTest.java
package com.example.reactivesecurity;

import com.example.reactivesecurity.model.Order;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.reactive.AutoConfigureWebTestClient;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.security.test.context.support.WithMockUser;
import org.springframework.security.test.web.reactive.server.SecurityMockServerConfigurers;
import org.springframework.test.web.reactive.server.WebTestClient;

import static org.springframework.security.test.web.reactive.server.SecurityMockServerConfigurers.mockUser;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@AutoConfigureWebTestClient
class ReactiveSecurityTest {

    @Autowired
    private WebTestClient webTestClient;

    @Test
    void publicEndpointShouldBeAccessibleWithoutAuth() {
        webTestClient.get()
            .uri("/api/public/info")
            .exchange()
            .expectStatus().isOk();
    }

    @Test
    void protectedEndpointShouldReturn401WithoutAuth() {
        webTestClient.get()
            .uri("/api/users")
            .exchange()
            .expectStatus().isUnauthorized();
    }

    @Test
    @WithMockUser(roles = "ADMIN")
    void adminEndpointShouldBeAccessibleByAdmin() {
        webTestClient
            .mutateWith(mockUser().roles("ADMIN"))
            .get()
            .uri("/api/admin/users")
            .exchange()
            .expectStatus().isOk();
    }

    @Test
    void adminEndpointShouldReturn403ForNormalUser() {
        webTestClient
            .mutateWith(mockUser().roles("USER"))
            .get()
            .uri("/api/admin/users")
            .exchange()
            .expectStatus().isForbidden();
    }

    @Test
    void loginWithValidCredentialsShouldReturnToken() {
        String loginBody = """
            {
                "username": "testuser",
                "password": "password123"
            }
            """;
        
        webTestClient.post()
            .uri("/api/auth/login")
            .contentType(org.springframework.http.MediaType.APPLICATION_JSON)
            .bodyValue(loginBody)
            .exchange()
            .expectStatus().isOk()
            .expectBody()
            .jsonPath("$.accessToken").isNotEmpty()
            .jsonPath("$.refreshToken").isNotEmpty()
            .jsonPath("$.tokenType").isEqualTo("Bearer");
    }

    @Test
    void requestWithValidJwtShouldBeAuthenticated() {
        // สร้าง JWT token สำหรับ test
        String validJwt = "eyJhbGciOiJIUzI1NiJ9..."; // test token
        
        webTestClient.get()
            .uri("/api/users/me")
            .header("Authorization", "Bearer " + validJwt)
            .exchange()
            .expectStatus().isOk();
    }
}
```

---

## Application Properties

```yaml
# application.yml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://auth.example.com
          jwk-set-uri: https://auth.example.com/.well-known/jwks.json

  r2dbc:
    url: r2dbc:postgresql://localhost:5432/securitydb
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}

jwt:
  secret: ${JWT_SECRET:myDefaultSecretKeyForDevelopmentOnly256bits}
  expiration: 86400000     # 24 ชั่วโมง
  refresh-expiration: 604800000  # 7 วัน

security:
  rate-limiting:
    max-attempts: 5
    lockout-duration: PT15M
  cors:
    allowed-origins:
      - http://localhost:3000
      - https://app.example.com
```

---

## สรุป Part 87

ในส่วนนี้เราได้เรียนรู้:
- **Spring Security WebFlux** - SecurityWebFilterChain และ reactive security
- **ReactiveUserDetailsService** - User authentication แบบ reactive
- **JWT Authentication** - สร้างและตรวจสอบ JWT tokens ใน reactive stack
- **@PreAuthorize/@PostAuthorize** - Method-level security
- **OAuth2 Resource Server** - Configure reactive OAuth2
- **CORS** - การ configure CORS ใน reactive applications
- **Security Context** - เข้าถึง security context ใน reactive chains
- **Rate Limiting** - ป้องกัน brute force attacks
- **Security Events** - การจัดการ security events
- **Testing** - ทดสอบ reactive security

---

*[← Part 86: Integration Patterns](./part-86-integration-patterns.md) | [Part 88: Data Streaming →](./part-88-data-streaming.md)*

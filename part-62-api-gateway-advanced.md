# Part 62: API Gateway Advanced
## ขั้นตอนที่ 2081-2120

**ระดับ:** ระดับโลก (World-Class)
**เวลาเรียน:** 4-5 ชั่วโมง
**เป้าหมาย:** Master Spring Cloud Gateway สำหรับ Production-Grade API Gateway

---

## บทนำ

Spring Cloud Gateway เป็น API Gateway ที่สร้างบน Spring WebFlux (Reactive) รองรับ High Throughput และมี Features ครบครัน ในส่วนนี้เราจะเจาะลึก Advanced Features ที่ใช้จริงในระบบ Production

---

## ขั้นตอนที่ 2081: Project Setup

```xml
<!-- pom.xml -->
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-gateway</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-redis-reactive</artifactId>
    </dependency>
    <dependency>
        <groupId>io.github.resilience4j</groupId>
        <artifactId>resilience4j-spring-boot3</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-security</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
    </dependency>
</dependencies>
```

---

## ขั้นตอนที่ 2082-2085: Advanced Routing Configuration

### อธิบาย

Spring Cloud Gateway ใช้ Route ซึ่งประกอบด้วย Predicate (เงื่อนไข) และ Filter (การประมวลผล) ก่อนและหลัง Proxy Request

```yaml
# application.yml - Advanced Routing
spring:
  cloud:
    gateway:
      routes:
        # Route พื้นฐาน
        - id: order-service
          uri: lb://order-service
          predicates:
            - Path=/api/orders/**
          filters:
            - StripPrefix=1
        
        # Route ที่มี Header Predicate
        - id: order-service-v2
          uri: lb://order-service-v2
          predicates:
            - Path=/api/orders/**
            - Header=X-API-Version, v2
          filters:
            - StripPrefix=1
        
        # Route ด้วย Query Parameter
        - id: product-search
          uri: lb://product-service
          predicates:
            - Path=/api/products/**
            - Query=search
          filters:
            - StripPrefix=1
            - AddRequestHeader=X-Source, gateway
        
        # Route ด้วย Weight (Canary Deployment)
        - id: payment-v1
          uri: lb://payment-service-v1
          predicates:
            - Path=/api/payments/**
            - Weight=payment-group, 90
          filters:
            - StripPrefix=1
        
        - id: payment-v2
          uri: lb://payment-service-v2
          predicates:
            - Path=/api/payments/**
            - Weight=payment-group, 10
          filters:
            - StripPrefix=1
        
        # Route ด้วย Method และ Time
        - id: read-only-hours
          uri: lb://catalog-service
          predicates:
            - Path=/api/catalog/**
            - Method=GET
            - Between=2024-01-01T00:00:00+07:00[Asia/Bangkok], 2024-12-31T23:59:59+07:00[Asia/Bangkok]
```

### Java Configuration สำหรับ Dynamic Routing

```java
// GatewayConfig.java
@Configuration
@RequiredArgsConstructor
public class GatewayConfig {
    
    private final RouteLocatorBuilder builder;
    private final JwtAuthenticationFilter jwtFilter;
    private final RateLimitFilter rateLimitFilter;
    
    @Bean
    public RouteLocator customRouteLocator() {
        return builder.routes()
            // Order Service Route
            .route("order-service", r -> r
                .path("/api/orders/**")
                .filters(f -> f
                    .filter(jwtFilter)
                    .filter(rateLimitFilter)
                    .rewritePath("/api/orders/(?<segment>.*)", 
                                 "/orders/${segment}")
                    .addRequestHeader("X-Gateway", "spring-cloud-gateway")
                    .addResponseHeader("X-Response-Time", 
                                       String.valueOf(System.currentTimeMillis()))
                    .retry(config -> config
                        .setRetries(3)
                        .setMethods(HttpMethod.GET)
                        .setBackoff(Duration.ofMillis(100), 
                                    Duration.ofSeconds(2), 2, true))
                    .circuitBreaker(config -> config
                        .setName("orderService")
                        .setFallbackUri("forward:/fallback/orders"))
                )
                .uri("lb://order-service")
            )
            .build();
    }
}
```

---

## ขั้นตอนที่ 2086-2090: Custom Filters

### Pre-Filter: ทำงานก่อนส่ง Request ไปยัง Service

```java
// RequestLoggingFilter.java - Global Pre-Filter
@Component
@Slf4j
public class RequestLoggingFilter implements GlobalFilter, Ordered {
    
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        ServerHttpRequest request = exchange.getRequest();
        
        String requestId = UUID.randomUUID().toString();
        long startTime = System.currentTimeMillis();
        
        // เพิ่ม Request ID เข้าไปใน Header
        ServerWebExchange modifiedExchange = exchange.mutate()
            .request(request.mutate()
                .header("X-Request-Id", requestId)
                .header("X-Start-Time", String.valueOf(startTime))
                .build())
            .build();
        
        log.info("Gateway Request: {} {} | RequestId: {} | Headers: {}",
                 request.getMethod(),
                 request.getURI(),
                 requestId,
                 request.getHeaders().toSingleValueMap());
        
        return chain.filter(modifiedExchange)
            .doFinally(signalType -> {
                long duration = System.currentTimeMillis() - startTime;
                log.info("Gateway Response: {} {} | RequestId: {} | Duration: {}ms | Signal: {}",
                         request.getMethod(),
                         request.getURI(),
                         requestId,
                         duration,
                         signalType);
            });
    }
    
    @Override
    public int getOrder() {
        return Ordered.HIGHEST_PRECEDENCE;  // รันก่อน Filter อื่นทั้งหมด
    }
}
```

### Post-Filter: ทำงานหลังได้รับ Response จาก Service

```java
// ResponseTransformFilter.java - ปรับแต่ง Response
@Component
public class ResponseTransformFilter implements GlobalFilter, Ordered {
    
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        return chain.filter(exchange).then(Mono.fromRunnable(() -> {
            ServerHttpResponse response = exchange.getResponse();
            
            // เพิ่ม Security Headers ใน Response
            HttpHeaders headers = response.getHeaders();
            headers.add("X-Content-Type-Options", "nosniff");
            headers.add("X-Frame-Options", "DENY");
            headers.add("X-XSS-Protection", "1; mode=block");
            headers.add("Strict-Transport-Security", 
                        "max-age=31536000; includeSubDomains");
            
            // เพิ่ม Request ID กลับไปใน Response
            String requestId = exchange.getRequest().getHeaders()
                .getFirst("X-Request-Id");
            if (requestId != null) {
                headers.add("X-Request-Id", requestId);
            }
        }));
    }
    
    @Override
    public int getOrder() {
        return -1;
    }
}
```

### Custom GatewayFilter Factory

```java
// RequestValidationGatewayFilterFactory.java - Custom Filter Factory
@Component
public class RequestValidationGatewayFilterFactory 
    extends AbstractGatewayFilterFactory<RequestValidationGatewayFilterFactory.Config> {
    
    public RequestValidationGatewayFilterFactory() {
        super(Config.class);
    }
    
    @Override
    public GatewayFilter apply(Config config) {
        return (exchange, chain) -> {
            ServerHttpRequest request = exchange.getRequest();
            
            // ตรวจสอบ Required Headers
            for (String requiredHeader : config.getRequiredHeaders()) {
                if (!request.getHeaders().containsKey(requiredHeader)) {
                    exchange.getResponse().setStatusCode(HttpStatus.BAD_REQUEST);
                    return exchange.getResponse().writeWith(
                        Mono.just(exchange.getResponse().bufferFactory()
                            .wrap(("Missing required header: " + requiredHeader)
                                .getBytes()))
                    );
                }
            }
            
            // ตรวจสอบ Content-Type สำหรับ POST/PUT
            if (config.isRequireJsonContentType() && 
                (request.getMethod() == HttpMethod.POST || 
                 request.getMethod() == HttpMethod.PUT)) {
                
                MediaType contentType = request.getHeaders().getContentType();
                if (contentType == null || 
                    !contentType.isCompatibleWith(MediaType.APPLICATION_JSON)) {
                    exchange.getResponse().setStatusCode(
                        HttpStatus.UNSUPPORTED_MEDIA_TYPE);
                    return exchange.getResponse().setComplete();
                }
            }
            
            return chain.filter(exchange);
        };
    }
    
    @Data
    public static class Config {
        private List<String> requiredHeaders = new ArrayList<>();
        private boolean requireJsonContentType = true;
    }
}
```

```yaml
# ใช้ Custom Filter Factory
spring:
  cloud:
    gateway:
      routes:
        - id: order-service
          uri: lb://order-service
          predicates:
            - Path=/api/orders/**
          filters:
            - name: RequestValidation
              args:
                requiredHeaders:
                  - X-API-Key
                  - X-Client-Id
                requireJsonContentType: true
```

---

## ขั้นตอนที่ 2091-2095: Rate Limiting ด้วย Redis

### อธิบาย

Rate Limiting ป้องกัน API ถูกใช้งานเกินขีดจำกัด มี 2 แนวทางหลัก:
- **Fixed Window:** นับ Request ใน Time Window คงที่
- **Sliding Window / Token Bucket:** นับ Request แบบ Rolling Window (แม่นยำกว่า)

```java
// RedisRateLimiterConfig.java
@Configuration
public class RedisRateLimiterConfig {
    
    @Bean
    public KeyResolver userKeyResolver() {
        // Rate limit ตาม User ID ใน JWT
        return exchange -> {
            String authorization = exchange.getRequest().getHeaders()
                .getFirst("Authorization");
            if (authorization != null && authorization.startsWith("Bearer ")) {
                // Extract user from JWT
                String token = authorization.substring(7);
                String userId = extractUserIdFromToken(token);
                return Mono.just("user:" + userId);
            }
            // Fallback ใช้ IP
            return Mono.just("ip:" + exchange.getRequest().getRemoteAddress()
                .getAddress().getHostAddress());
        };
    }
    
    @Bean
    public KeyResolver apiKeyResolver() {
        // Rate limit ตาม API Key
        return exchange -> {
            String apiKey = exchange.getRequest().getHeaders()
                .getFirst("X-API-Key");
            return Mono.just(apiKey != null ? "apikey:" + apiKey : "anonymous");
        };
    }
    
    private String extractUserIdFromToken(String token) {
        // Simplified JWT parsing
        try {
            String payload = token.split("\\.")[1];
            String decodedPayload = new String(
                Base64.getDecoder().decode(payload));
            // Parse JSON and extract sub claim
            return "user123"; // Simplified
        } catch (Exception e) {
            return "unknown";
        }
    }
}
```

```yaml
# application.yml - Redis Rate Limiter
spring:
  cloud:
    gateway:
      routes:
        - id: order-service
          uri: lb://order-service
          predicates:
            - Path=/api/orders/**
          filters:
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 10    # 10 Request/วินาที
                redis-rate-limiter.burstCapacity: 20    # Burst สูงสุด 20
                redis-rate-limiter.requestedTokens: 1   # แต่ละ Request ใช้ 1 Token
                key-resolver: "#{@userKeyResolver}"

  data:
    redis:
      host: localhost
      port: 6379
```

```java
// CustomRedisRateLimiter.java - Rate Limiter ที่ Customize เองได้
@Component
@RequiredArgsConstructor
@Slf4j
public class CustomRedisRateLimiter extends RedisRateLimiter {
    
    private final ReactiveStringRedisTemplate redisTemplate;
    
    public Mono<Response> isAllowed(String routeId, String id) {
        // ใช้ Sliding Window Algorithm
        long now = System.currentTimeMillis();
        long windowStart = now - 60000; // 60 seconds window
        
        String key = "rate_limit:" + routeId + ":" + id;
        
        return redisTemplate.execute(connection -> {
            // ลบ Request เก่าออกจาก Window
            connection.zRemRangeByScore(key.getBytes(), 
                                         0, windowStart);
            
            // นับ Request ใน Window ปัจจุบัน
            return connection.zCard(key.getBytes())
                .flatMap(count -> {
                    int limit = getRateLimit(routeId, id);
                    
                    if (count < limit) {
                        // เพิ่ม Request ปัจจุบันเข้า Window
                        return connection.zAdd(key.getBytes(), 
                                              now, 
                                              String.valueOf(now).getBytes())
                            .then(connection.expire(key.getBytes(), 
                                                    Duration.ofMinutes(1)))
                            .thenReturn(new Response(true, 
                                                    Map.of(
                                                        "X-RateLimit-Remaining", 
                                                        String.valueOf(limit - count - 1),
                                                        "X-RateLimit-Limit", 
                                                        String.valueOf(limit)
                                                    )));
                    } else {
                        log.warn("Rate limit exceeded for: {} on route: {}", 
                                id, routeId);
                        return Mono.just(new Response(false, 
                                        Map.of(
                                            "X-RateLimit-Remaining", "0",
                                            "X-RateLimit-Limit", String.valueOf(limit),
                                            "Retry-After", "60"
                                        )));
                    }
                });
        }).next();
    }
    
    private int getRateLimit(String routeId, String id) {
        // Different limits for different routes/users
        if (id.startsWith("apikey:premium")) return 1000;
        if (id.startsWith("apikey:")) return 100;
        return 10; // Default
    }
}
```

---

## ขั้นตอนที่ 2096-2100: Authentication/Authorization ที่ Gateway

### JWT Validation ที่ Gateway Level

```java
// JwtAuthenticationFilter.java
@Component
@RequiredArgsConstructor
@Slf4j
public class JwtAuthenticationFilter implements GatewayFilter, Ordered {
    
    private final JwtTokenValidator tokenValidator;
    
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        ServerHttpRequest request = exchange.getRequest();
        
        // ข้ามการตรวจสอบสำหรับ Public Endpoints
        if (isPublicEndpoint(request.getPath().toString())) {
            return chain.filter(exchange);
        }
        
        String authHeader = request.getHeaders().getFirst("Authorization");
        
        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            return onUnauthorized(exchange, "Missing or invalid Authorization header");
        }
        
        String token = authHeader.substring(7);
        
        return tokenValidator.validateToken(token)
            .flatMap(claims -> {
                // เพิ่ม User Info เข้าไปใน Request Headers
                // เพื่อให้ Downstream Services ใช้ได้โดยไม่ต้องตรวจสอบ JWT อีกครั้ง
                ServerWebExchange modifiedExchange = exchange.mutate()
                    .request(request.mutate()
                        .header("X-User-Id", claims.getSubject())
                        .header("X-User-Roles", String.join(",", 
                                                            claims.getRoles()))
                        .header("X-User-Email", claims.getEmail())
                        .build())
                    .build();
                
                return chain.filter(modifiedExchange);
            })
            .onErrorResume(e -> {
                log.warn("JWT validation failed: {}", e.getMessage());
                return onUnauthorized(exchange, "Invalid token");
            });
    }
    
    private boolean isPublicEndpoint(String path) {
        return path.startsWith("/api/auth/") ||
               path.startsWith("/api/public/") ||
               path.equals("/actuator/health");
    }
    
    private Mono<Void> onUnauthorized(ServerWebExchange exchange, String message) {
        ServerHttpResponse response = exchange.getResponse();
        response.setStatusCode(HttpStatus.UNAUTHORIZED);
        response.getHeaders().add("Content-Type", "application/json");
        
        String body = "{\"error\":\"UNAUTHORIZED\",\"message\":\"" + message + "\"}";
        DataBuffer buffer = response.bufferFactory().wrap(body.getBytes());
        
        return response.writeWith(Mono.just(buffer));
    }
    
    @Override
    public int getOrder() {
        return -100;
    }
}
```

```java
// JwtTokenValidator.java
@Service
@Slf4j
public class JwtTokenValidator {
    
    @Value("${jwt.secret}")
    private String jwtSecret;
    
    @Value("${jwt.public-key-url:}")
    private String publicKeyUrl;
    
    private final WebClient webClient;
    private final Cache<String, JwtClaims> tokenCache;
    
    public JwtTokenValidator(WebClient.Builder webClientBuilder) {
        this.webClient = webClientBuilder.build();
        // Cache token validation results สำหรับ 1 นาที
        this.tokenCache = Caffeine.newBuilder()
            .maximumSize(10000)
            .expireAfterWrite(Duration.ofMinutes(1))
            .build();
    }
    
    public Mono<JwtClaims> validateToken(String token) {
        // Check cache ก่อน
        JwtClaims cached = tokenCache.getIfPresent(token);
        if (cached != null) {
            if (cached.isExpired()) {
                tokenCache.invalidate(token);
                return Mono.error(new UnauthorizedException("Token expired"));
            }
            return Mono.just(cached);
        }
        
        return Mono.fromCallable(() -> parseAndValidate(token))
            .doOnSuccess(claims -> tokenCache.put(token, claims));
    }
    
    private JwtClaims parseAndValidate(String token) {
        try {
            Jws<Claims> jws = Jwts.parserBuilder()
                .setSigningKey(getSigningKey())
                .build()
                .parseClaimsJws(token);
            
            Claims claims = jws.getBody();
            
            return JwtClaims.builder()
                .subject(claims.getSubject())
                .email(claims.get("email", String.class))
                .roles(claims.get("roles", List.class))
                .expiration(claims.getExpiration().toInstant())
                .build();
                
        } catch (ExpiredJwtException e) {
            throw new UnauthorizedException("Token expired");
        } catch (Exception e) {
            throw new UnauthorizedException("Invalid token: " + e.getMessage());
        }
    }
    
    private Key getSigningKey() {
        byte[] keyBytes = Decoders.BASE64.decode(jwtSecret);
        return Keys.hmacShaKeyFor(keyBytes);
    }
}
```

### Role-Based Authorization ที่ Gateway

```java
// RouteAuthorizationFilter.java - ตรวจสอบ Role สำหรับแต่ละ Route
@Component
@RequiredArgsConstructor
public class RouteAuthorizationFilter implements GlobalFilter, Ordered {
    
    // กำหนด Role ที่ต้องการสำหรับแต่ละ Route Pattern
    private static final Map<String, List<String>> ROUTE_PERMISSIONS = Map.of(
        "/api/admin/**", List.of("ADMIN"),
        "/api/orders/**", List.of("USER", "ADMIN"),
        "/api/reports/**", List.of("ANALYST", "ADMIN")
    );
    
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String path = exchange.getRequest().getPath().toString();
        List<String> requiredRoles = getRequiredRoles(path);
        
        if (requiredRoles.isEmpty()) {
            return chain.filter(exchange); // ไม่ต้องตรวจสอบ Role
        }
        
        String userRolesHeader = exchange.getRequest().getHeaders()
            .getFirst("X-User-Roles");
        
        if (userRolesHeader == null) {
            return onForbidden(exchange, "No roles assigned");
        }
        
        List<String> userRoles = Arrays.asList(userRolesHeader.split(","));
        
        boolean hasPermission = requiredRoles.stream()
            .anyMatch(userRoles::contains);
        
        if (!hasPermission) {
            return onForbidden(exchange, 
                "Insufficient permissions. Required: " + requiredRoles);
        }
        
        return chain.filter(exchange);
    }
    
    private List<String> getRequiredRoles(String path) {
        return ROUTE_PERMISSIONS.entrySet().stream()
            .filter(e -> pathMatches(path, e.getKey()))
            .flatMap(e -> e.getValue().stream())
            .collect(Collectors.toList());
    }
    
    private boolean pathMatches(String path, String pattern) {
        PathMatcher matcher = new AntPathMatcher();
        return matcher.match(pattern, path);
    }
    
    private Mono<Void> onForbidden(ServerWebExchange exchange, String message) {
        exchange.getResponse().setStatusCode(HttpStatus.FORBIDDEN);
        String body = "{\"error\":\"FORBIDDEN\",\"message\":\"" + message + "\"}";
        DataBuffer buffer = exchange.getResponse().bufferFactory().wrap(body.getBytes());
        return exchange.getResponse().writeWith(Mono.just(buffer));
    }
    
    @Override
    public int getOrder() {
        return -90; // หลัง JWT Filter แต่ก่อน Business Logic
    }
}
```

---

## ขั้นตอนที่ 2101-2105: Request/Response Transformation

### Header Transformation

```java
// HeaderTransformationFilter.java
@Component
public class HeaderTransformationFilter implements GlobalFilter, Ordered {
    
    // Headers ที่ต้องการส่งต่อไปยัง Downstream
    private static final Set<String> FORWARDED_HEADERS = Set.of(
        "X-User-Id", "X-User-Roles", "X-Request-Id", "X-Correlation-Id"
    );
    
    // Headers ที่ต้องลบก่อนส่งต่อ (Security Sensitive)
    private static final Set<String> REMOVED_HEADERS = Set.of(
        "Authorization", "X-Internal-Secret", "Cookie"
    );
    
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        ServerHttpRequest.Builder requestBuilder = exchange.getRequest().mutate();
        
        // ลบ Headers ที่ Sensitive
        REMOVED_HEADERS.forEach(requestBuilder::removeHeader);
        
        // เพิ่ม Internal Service Token แทน
        requestBuilder.header("X-Internal-Token", generateInternalToken());
        
        ServerWebExchange modifiedExchange = exchange.mutate()
            .request(requestBuilder.build())
            .build();
        
        return chain.filter(modifiedExchange);
    }
    
    private String generateInternalToken() {
        // สร้าง Short-lived Internal Token สำหรับ Service-to-Service Auth
        return "internal-" + UUID.randomUUID().toString();
    }
    
    @Override
    public int getOrder() {
        return -80;
    }
}
```

### Body Transformation

```java
// RequestBodyTransformFilter.java - แปลง Request Body
@Component
@Slf4j
public class RequestBodyTransformFilter implements GlobalFilter, Ordered {
    
    private final ObjectMapper objectMapper;
    
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        // ทำงานเฉพาะกับ POST/PUT ที่มี JSON body
        if (!isJsonRequest(exchange.getRequest())) {
            return chain.filter(exchange);
        }
        
        return exchange.getRequest().getBody()
            .collectList()
            .flatMap(dataBuffers -> {
                // Read body
                byte[] bodyBytes = new byte[dataBuffers.stream()
                    .mapToInt(DataBuffer::readableByteCount).sum()];
                int offset = 0;
                for (DataBuffer buffer : dataBuffers) {
                    int length = buffer.readableByteCount();
                    buffer.read(bodyBytes, offset, length);
                    offset += length;
                }
                
                try {
                    // Transform body
                    JsonNode originalBody = objectMapper.readTree(bodyBytes);
                    ObjectNode transformedBody = (ObjectNode) originalBody;
                    
                    // เพิ่ม Metadata ที่ต้องการ
                    transformedBody.put("_gateway_timestamp", 
                                       System.currentTimeMillis());
                    transformedBody.put("_request_id", 
                                       exchange.getRequest().getHeaders()
                                           .getFirst("X-Request-Id"));
                    
                    byte[] newBodyBytes = objectMapper.writeValueAsBytes(
                        transformedBody);
                    
                    // สร้าง Request ใหม่ด้วย Body ที่แปลงแล้ว
                    ServerHttpRequest newRequest = exchange.getRequest().mutate()
                        .header("Content-Length", 
                                String.valueOf(newBodyBytes.length))
                        .build();
                    
                    DataBuffer newBuffer = exchange.getResponse().bufferFactory()
                        .wrap(newBodyBytes);
                    
                    ServerWebExchange newExchange = exchange.mutate()
                        .request(new ServerHttpRequestDecorator(newRequest) {
                            @Override
                            public Flux<DataBuffer> getBody() {
                                return Flux.just(newBuffer);
                            }
                        })
                        .build();
                    
                    return chain.filter(newExchange);
                    
                } catch (Exception e) {
                    log.error("Error transforming request body: {}", e.getMessage());
                    return chain.filter(exchange); // ส่งต่อ Original
                }
            });
    }
    
    private boolean isJsonRequest(ServerHttpRequest request) {
        MediaType contentType = request.getHeaders().getContentType();
        return contentType != null && 
               contentType.isCompatibleWith(MediaType.APPLICATION_JSON) &&
               (request.getMethod() == HttpMethod.POST || 
                request.getMethod() == HttpMethod.PUT);
    }
    
    @Override
    public int getOrder() {
        return -70;
    }
}
```

---

## ขั้นตอนที่ 2106-2110: Circuit Breaker ที่ Gateway

```java
// GatewayCircuitBreakerConfig.java
@Configuration
public class GatewayCircuitBreakerConfig {
    
    @Bean
    public Customizer<ReactiveResilience4JCircuitBreakerFactory> circuitBreakerCustomizer() {
        return factory -> {
            factory.configureDefault(id -> new Resilience4JConfigBuilder(id)
                .circuitBreakerConfig(CircuitBreakerConfig.custom()
                    .slidingWindowSize(10)
                    .minimumNumberOfCalls(5)
                    .failureRateThreshold(50.0f)
                    .waitDurationInOpenState(Duration.ofSeconds(30))
                    .permittedNumberOfCallsInHalfOpenState(3)
                    .slowCallRateThreshold(100.0f)
                    .slowCallDurationThreshold(Duration.ofSeconds(5))
                    .build())
                .timeLimiterConfig(TimeLimiterConfig.custom()
                    .timeoutDuration(Duration.ofSeconds(10))
                    .build())
                .build()
            );
            
            // Custom config สำหรับ Payment Service
            factory.configure(builder -> builder
                .circuitBreakerConfig(CircuitBreakerConfig.custom()
                    .slidingWindowSize(5)
                    .failureRateThreshold(30.0f) // ไวต่อ Failure มากกว่า
                    .waitDurationInOpenState(Duration.ofMinutes(1))
                    .build())
                .timeLimiterConfig(TimeLimiterConfig.custom()
                    .timeoutDuration(Duration.ofSeconds(3)) // Timeout ต่ำกว่า
                    .build())
                .build(), 
                "paymentService"
            );
        };
    }
}
```

```java
// GatewayFallbackController.java
@RestController
@Slf4j
public class GatewayFallbackController {
    
    @RequestMapping("/fallback/orders")
    public ResponseEntity<Map<String, Object>> ordersFallback(
            @RequestHeader(value = "X-Request-Id", required = false) String requestId) {
        
        log.warn("Orders service circuit breaker open, returning fallback. RequestId: {}", 
                requestId);
        
        return ResponseEntity
            .status(HttpStatus.SERVICE_UNAVAILABLE)
            .body(Map.of(
                "error", "SERVICE_UNAVAILABLE",
                "message", "Order service is temporarily unavailable",
                "requestId", requestId != null ? requestId : "unknown",
                "timestamp", System.currentTimeMillis(),
                "retryAfter", 30
            ));
    }
    
    @RequestMapping("/fallback/payments")
    public ResponseEntity<Map<String, Object>> paymentsFallback() {
        return ResponseEntity
            .status(HttpStatus.SERVICE_UNAVAILABLE)
            .body(Map.of(
                "error", "SERVICE_UNAVAILABLE",
                "message", "Payment service is temporarily unavailable. Please try again later.",
                "timestamp", System.currentTimeMillis()
            ));
    }
    
    @RequestMapping("/fallback/default")
    public ResponseEntity<Map<String, Object>> defaultFallback(
            ServerWebExchange exchange) {
        
        String path = exchange.getRequest().getPath().toString();
        
        return ResponseEntity
            .status(HttpStatus.SERVICE_UNAVAILABLE)
            .body(Map.of(
                "error", "SERVICE_UNAVAILABLE",
                "message", "Service temporarily unavailable",
                "path", path,
                "timestamp", System.currentTimeMillis()
            ));
    }
}
```

---

## ขั้นตอนที่ 2111-2115: Distributed Tracing ที่ Gateway

```java
// TracingFilter.java - Propagate Trace Context
@Component
@RequiredArgsConstructor
@Slf4j
public class TracingFilter implements GlobalFilter, Ordered {
    
    private final Tracer tracer;
    
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        ServerHttpRequest request = exchange.getRequest();
        
        // สร้าง Span สำหรับ Gateway Request
        Span span = tracer.nextSpan()
            .name("gateway " + request.getMethod() + " " + 
                  request.getPath().toString())
            .tag("http.method", request.getMethod().name())
            .tag("http.url", request.getURI().toString())
            .tag("gateway.route", getRouteId(exchange))
            .start();
        
        // ดึง Trace Context เพื่อ Propagate
        String traceId = span.context().traceId();
        String spanId = span.context().spanId();
        
        ServerWebExchange modifiedExchange = exchange.mutate()
            .request(request.mutate()
                .header("X-B3-TraceId", traceId)
                .header("X-B3-SpanId", spanId)
                .header("X-B3-Sampled", "1")
                .build())
            .build();
        
        return chain.filter(modifiedExchange)
            .doOnSuccess(v -> {
                span.tag("http.status_code", 
                         String.valueOf(exchange.getResponse()
                             .getStatusCode().value()));
                span.end();
            })
            .doOnError(e -> {
                span.tag("error", e.getMessage());
                span.end();
            });
    }
    
    private String getRouteId(ServerWebExchange exchange) {
        Route route = exchange.getAttribute(
            ServerWebExchangeUtils.GATEWAY_ROUTE_ATTR);
        return route != null ? route.getId() : "unknown";
    }
    
    @Override
    public int getOrder() {
        return -200; // รันก่อน Filter อื่นทั้งหมด
    }
}
```

---

## ขั้นตอนที่ 2116-2120: API Composition Pattern

### อธิบาย

API Composition คือ Pattern ที่ Gateway รวม Response จากหลาย Services เป็น Response เดียว ลด Round-trips ของ Client

```java
// ApiCompositionController.java
@RestController
@RequestMapping("/api/composite")
@RequiredArgsConstructor
@Slf4j
public class ApiCompositionController {
    
    private final WebClient.Builder webClientBuilder;
    
    // รวมข้อมูล Order, Product, Customer ใน Request เดียว
    @GetMapping("/order-details/{orderId}")
    public Mono<OrderDetailsResponse> getOrderDetails(
            @PathVariable String orderId,
            @RequestHeader("X-User-Id") String userId) {
        
        WebClient client = webClientBuilder.build();
        
        // ดึงข้อมูลจาก 3 Services พร้อมกัน
        Mono<OrderDto> orderMono = client.get()
            .uri("http://order-service/api/orders/{id}", orderId)
            .retrieve()
            .bodyToMono(OrderDto.class)
            .timeout(Duration.ofSeconds(3));
        
        Mono<CustomerDto> customerMono = client.get()
            .uri("http://customer-service/api/customers/{id}", userId)
            .retrieve()
            .bodyToMono(CustomerDto.class)
            .timeout(Duration.ofSeconds(3));
        
        // รวมผล
        return Mono.zip(orderMono, customerMono)
            .flatMap(tuple -> {
                OrderDto order = tuple.getT1();
                CustomerDto customer = tuple.getT2();
                
                // ดึงข้อมูล Products สำหรับแต่ละ Item ใน Order
                List<Mono<ProductDto>> productMonos = order.getItems().stream()
                    .map(item -> client.get()
                        .uri("http://product-service/api/products/{id}", 
                             item.getProductId())
                        .retrieve()
                        .bodyToMono(ProductDto.class)
                        .timeout(Duration.ofSeconds(2))
                        .onErrorReturn(ProductDto.unknown(item.getProductId())))
                    .collect(Collectors.toList());
                
                return Mono.zip(productMonos, products -> {
                    List<ProductDto> productList = Arrays.stream(products)
                        .map(p -> (ProductDto) p)
                        .collect(Collectors.toList());
                    
                    return OrderDetailsResponse.builder()
                        .order(order)
                        .customer(customer)
                        .products(productList)
                        .build();
                });
            })
            .doOnError(e -> log.error(
                "Error composing order details for {}: {}", orderId, e.getMessage()));
    }
    
    // Dashboard Aggregation
    @GetMapping("/dashboard")
    public Mono<DashboardResponse> getDashboard(
            @RequestHeader("X-User-Id") String userId) {
        
        WebClient client = webClientBuilder.build();
        
        Mono<List<OrderDto>> recentOrders = client.get()
            .uri("http://order-service/api/orders/recent?userId={userId}&limit=5", 
                 userId)
            .retrieve()
            .bodyToFlux(OrderDto.class)
            .collectList()
            .onErrorReturn(Collections.emptyList());
        
        Mono<UserStatsDto> userStats = client.get()
            .uri("http://analytics-service/api/stats/user/{userId}", userId)
            .retrieve()
            .bodyToMono(UserStatsDto.class)
            .onErrorReturn(UserStatsDto.empty());
        
        Mono<List<NotificationDto>> notifications = client.get()
            .uri("http://notification-service/api/notifications/unread?userId={userId}", 
                 userId)
            .retrieve()
            .bodyToFlux(NotificationDto.class)
            .collectList()
            .onErrorReturn(Collections.emptyList());
        
        return Mono.zip(recentOrders, userStats, notifications)
            .map(tuple -> DashboardResponse.builder()
                .recentOrders(tuple.getT1())
                .stats(tuple.getT2())
                .notifications(tuple.getT3())
                .generatedAt(Instant.now())
                .build());
    }
}
```

### Integration Test สำหรับ API Gateway

```java
// GatewayIntegrationTest.java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@ActiveProfiles("test")
class GatewayIntegrationTest {
    
    @Autowired
    private WebTestClient webTestClient;
    
    @MockBean
    private JwtTokenValidator jwtTokenValidator;
    
    @Test
    void shouldRouteToOrderService() {
        given(jwtTokenValidator.validateToken(any()))
            .willReturn(Mono.just(createTestClaims()));
        
        webTestClient.get()
            .uri("/api/orders/123")
            .header("Authorization", "Bearer test-token")
            .exchange()
            .expectStatus().isOk();
    }
    
    @Test
    void shouldReturnUnauthorizedWithoutToken() {
        webTestClient.get()
            .uri("/api/orders/123")
            .exchange()
            .expectStatus().isUnauthorized()
            .expectBody()
            .jsonPath("$.error").isEqualTo("UNAUTHORIZED");
    }
    
    @Test
    void shouldApplyRateLimiting() {
        // ส่ง Request เกิน Rate Limit
        IntStream.range(0, 25).forEach(i -> {
            webTestClient.get()
                .uri("/api/orders/123")
                .header("Authorization", "Bearer test-token")
                .header("X-Forwarded-For", "192.168.1.100")
                .exchange();
        });
        
        // Request สุดท้ายควร Rate Limited
        webTestClient.get()
            .uri("/api/orders/123")
            .header("Authorization", "Bearer test-token")
            .header("X-Forwarded-For", "192.168.1.100")
            .exchange()
            .expectStatus().isEqualTo(HttpStatus.TOO_MANY_REQUESTS);
    }
}
```

---

## สรุป API Gateway Advanced Features

| Feature | ใช้เมื่อ | ประโยชน์ |
|---------|---------|---------|
| Custom Filters | Need cross-cutting concerns | Centralized logic |
| Rate Limiting | Protect services from overload | Fair usage, DoS protection |
| JWT Auth at Gateway | All routes need auth | Single auth point |
| Request Transformation | Header/Body normalization | Backend simplification |
| Circuit Breaker | Unstable backends | Resilience, Fast fail |
| API Composition | Multiple service aggregation | Reduce client round-trips |
| Distributed Tracing | Debugging distributed systems | Observability |

---

*[← Part 61: Microservices Patterns](./part-61-microservices-patterns.md) | [Part 63: Distributed Tracing →](./part-63-distributed-tracing.md)*

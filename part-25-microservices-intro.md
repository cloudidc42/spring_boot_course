# Part 25: Microservices Introduction
## ขั้นตอนที่ 666-700

> **ระดับ:** สูง (Advanced)  
> **เวลาเรียน:** 6-8 ชั่วโมง  
> **เป้าหมาย:** เข้าใจ Microservices Architecture และเริ่มต้นด้วย Spring Cloud

---

## ขั้นตอนที่ 666: Monolith vs Microservices

```
Monolith                        Microservices
─────────────                   ───────────────────────────
Single deployable unit          Multiple small services
Single database                 Each service has own DB
Easy to develop initially       Complex distributed system
Hard to scale parts             Scale each service independently
Deploy whole app for 1 change   Deploy individual service
Technology lock-in              Polyglot (different tech per service)

When to choose Monolith:
  ✅ Small team (< 10 engineers)
  ✅ Early-stage startup
  ✅ Simple domain
  ✅ Unknown scaling needs

When to choose Microservices:
  ✅ Large team (50+ engineers)
  ✅ Different scaling needs per feature
  ✅ Independent deployment needed
  ✅ Multiple technology stacks needed
```

---

## ขั้นตอนที่ 667: E-Commerce Microservices Design

```
┌─────────────────────────────────────────────────────────────────┐
│                         API Gateway                              │
│                     (Spring Cloud Gateway)                       │
└──────┬──────────┬──────────┬──────────┬──────────┬─────────────┘
       │          │          │          │          │
   ┌───▼───┐  ┌───▼───┐  ┌───▼────┐  ┌───▼───┐  ┌───▼────┐
   │ User  │  │Product│  │ Order  │  │Payment│  │Notif.  │
   │Service│  │Service│  │Service │  │Service│  │Service │
   └───┬───┘  └───┬───┘  └───┬────┘  └───┬───┘  └───┬────┘
       │          │          │           │          │
   ┌───▼───┐  ┌───▼───┐  ┌───▼────┐  ┌───▼────┐  ┌───▼────┐
   │Users  │  │Prod.  │  │Orders  │  │Payments│  │Kafka   │
   │DB     │  │DB     │  │DB      │  │DB      │  │Events  │
   └───────┘  └───────┘  └────────┘  └────────┘  └────────┘

Infrastructure:
  ┌──────────────┐  ┌───────────────┐  ┌───────────┐
  │Service       │  │Config         │  │Distributed│
  │Discovery     │  │Server         │  │Tracing    │
  │(Eureka)      │  │(Spring Cloud  │  │(Zipkin)   │
  └──────────────┘  │ Config)       │  └───────────┘
                    └───────────────┘
```

---

## ขั้นตอนที่ 668: Spring Cloud Dependencies

```xml
<!-- Parent pom for microservices -->
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>2023.0.0</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>

<!-- Service Discovery (Eureka Client) -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>

<!-- API Gateway -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-gateway</artifactId>
</dependency>

<!-- Config Client -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-config</artifactId>
</dependency>

<!-- Load Balancer -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-loadbalancer</artifactId>
</dependency>

<!-- Circuit Breaker (Resilience4j) -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-circuitbreaker-resilience4j</artifactId>
</dependency>

<!-- OpenFeign (HTTP client) -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```

---

## ขั้นตอนที่ 669: Service Discovery - Eureka Server

```java
// eureka-server/src/main/java/...
@SpringBootApplication
@EnableEurekaServer
public class EurekaServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(EurekaServerApplication.class, args);
    }
}
```

```yaml
# eureka-server/src/main/resources/application.yml
server:
  port: 8761

spring:
  application:
    name: eureka-server

eureka:
  instance:
    hostname: localhost
  client:
    register-with-eureka: false  # Eureka server doesn't register itself
    fetch-registry: false
  server:
    wait-time-in-ms-when-sync-empty: 0
```

---

## ขั้นตอนที่ 670: Register Service with Eureka

```java
// user-service/src/main/java/...
@SpringBootApplication
@EnableDiscoveryClient
public class UserServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(UserServiceApplication.class, args);
    }
}
```

```yaml
# user-service/application.yml
server:
  port: 8081

spring:
  application:
    name: user-service

eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
  instance:
    prefer-ip-address: true
    instance-id: ${spring.application.name}:${random.value}
    health-check-url-path: /actuator/health
```

---

## ขั้นตอนที่ 671: API Gateway

```java
@SpringBootApplication
@EnableDiscoveryClient
public class GatewayApplication {
    public static void main(String[] args) {
        SpringApplication.run(GatewayApplication.class, args);
    }
}
```

```yaml
# gateway/application.yml
server:
  port: 8080

spring:
  application:
    name: api-gateway
  
  cloud:
    gateway:
      discovery:
        locator:
          enabled: true  # Auto-discover routes from Eureka
          lower-case-service-id: true
      
      routes:
        - id: user-service
          uri: lb://user-service      # lb = load balanced via Eureka
          predicates:
            - Path=/api/v1/users/**
          filters:
            - StripPrefix=0
            - AddRequestHeader=X-Gateway-Timestamp, #{T(java.time.Instant).now()}
        
        - id: product-service
          uri: lb://product-service
          predicates:
            - Path=/api/v1/products/**
          filters:
            - StripPrefix=0
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 10
                redis-rate-limiter.burstCapacity: 20
        
        - id: order-service
          uri: lb://order-service
          predicates:
            - Path=/api/v1/orders/**
            - Header=Authorization, Bearer .+    # Only authenticated
      
      # Global filters
      default-filters:
        - AddResponseHeader=X-Gateway-Version, 1.0
```

---

## ขั้นตอนที่ 672: Gateway Authentication Filter

```java
@Component
@RequiredArgsConstructor
@Slf4j
public class JwtGatewayFilter implements GlobalFilter, Ordered {
    
    private final JwtService jwtService;
    
    private final List<String> publicPaths = List.of(
        "/api/v1/auth/login",
        "/api/v1/auth/register",
        "/api/v1/products",
        "/actuator"
    );
    
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        ServerHttpRequest request = exchange.getRequest();
        String path = request.getPath().toString();
        
        // Skip public paths
        if (publicPaths.stream().anyMatch(path::startsWith)) {
            return chain.filter(exchange);
        }
        
        // Check Authorization header
        String authHeader = request.getHeaders().getFirst("Authorization");
        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            return unauthorized(exchange);
        }
        
        String token = authHeader.substring(7);
        try {
            String userId = jwtService.extractUserId(token);
            String userEmail = jwtService.extractUsername(token);
            
            // Add user info to downstream headers
            ServerHttpRequest modifiedRequest = exchange.getRequest().mutate()
                .header("X-User-Id", userId)
                .header("X-User-Email", userEmail)
                .build();
            
            return chain.filter(exchange.mutate().request(modifiedRequest).build());
            
        } catch (JwtException e) {
            log.warn("Invalid JWT: {}", e.getMessage());
            return unauthorized(exchange);
        }
    }
    
    private Mono<Void> unauthorized(ServerWebExchange exchange) {
        exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
        return exchange.getResponse().setComplete();
    }
    
    @Override
    public int getOrder() { return -1; }
}
```

---

## ขั้นตอนที่ 673: Inter-Service Communication - Feign

```java
// order-service calls user-service via Feign
@FeignClient(
    name = "user-service",
    fallback = UserClientFallback.class,
    configuration = FeignConfig.class
)
public interface UserClient {
    
    @GetMapping("/api/v1/users/{id}")
    ApiResponse<UserResponse> getUserById(@PathVariable Long id);
    
    @GetMapping("/api/v1/users/{id}/exists")
    boolean userExists(@PathVariable Long id);
}

// Fallback when user-service is down
@Component
public class UserClientFallback implements UserClient {
    
    @Override
    public ApiResponse<UserResponse> getUserById(Long id) {
        return ApiResponse.error("User service unavailable");
    }
    
    @Override
    public boolean userExists(Long id) {
        return false;
    }
}

// Feign configuration
@Configuration
public class FeignConfig {
    
    @Bean
    public RequestInterceptor authInterceptor() {
        // Pass auth token to downstream services
        return template -> {
            ServletRequestAttributes attributes = (ServletRequestAttributes) 
                RequestContextHolder.getRequestAttributes();
            if (attributes != null) {
                String authHeader = attributes.getRequest().getHeader("Authorization");
                if (authHeader != null) {
                    template.header("Authorization", authHeader);
                }
            }
        };
    }
    
    @Bean
    public Retryer feignRetryer() {
        return new Retryer.Default(100, 1000, 3);  // 3 retries
    }
}

// Enable Feign in main class
@SpringBootApplication
@EnableFeignClients
@EnableDiscoveryClient
public class OrderServiceApplication { ... }

// Use in service
@Service
@RequiredArgsConstructor
public class OrderService {
    
    private final UserClient userClient;
    
    public OrderResponse create(Long userId, CreateOrderRequest request) {
        // Check user exists
        if (!userClient.userExists(userId)) {
            throw new ResourceNotFoundException("User", "id", userId);
        }
        // ... create order
    }
}
```

---

## ขั้นตอนที่ 674: Circuit Breaker - Resilience4j

```yaml
# application.yml
resilience4j:
  circuitbreaker:
    instances:
      user-service:
        registerHealthIndicator: true
        slidingWindowSize: 10
        permittedNumberOfCallsInHalfOpenState: 3
        slidingWindowType: COUNT_BASED
        minimumNumberOfCalls: 5
        waitDurationInOpenState: 5s
        failureRateThreshold: 50
        automaticTransitionFromOpenToHalfOpenEnabled: true
  
  retry:
    instances:
      user-service:
        maxAttempts: 3
        waitDuration: 1s
        retryExceptions:
          - feign.FeignException
  
  timelimiter:
    instances:
      user-service:
        timeoutDuration: 3s
        cancelRunningFuture: true
```

```java
@Service
@RequiredArgsConstructor
public class OrderService {
    
    private final UserClient userClient;
    
    @CircuitBreaker(name = "user-service", fallbackMethod = "getUserFallback")
    @Retry(name = "user-service")
    @TimeLimiter(name = "user-service")
    public CompletableFuture<UserResponse> getUser(Long userId) {
        return CompletableFuture.supplyAsync(() -> 
            userClient.getUserById(userId).getData());
    }
    
    private CompletableFuture<UserResponse> getUserFallback(Long userId, Throwable t) {
        log.warn("User service unavailable, using fallback: {}", t.getMessage());
        // Return cached or minimal user data
        return CompletableFuture.completedFuture(
            new UserResponse(userId, "Unknown", null, null, null, null, null, false, null)
        );
    }
}
```

---

## ขั้นตอนที่ 675: Config Server

```java
// config-server/
@SpringBootApplication
@EnableConfigServer
public class ConfigServerApplication { ... }
```

```yaml
# config-server/application.yml
server:
  port: 8888

spring:
  cloud:
    config:
      server:
        git:
          uri: https://github.com/myorg/config-repo
          default-label: main
          search-paths: '{application}'
          clone-on-start: true
```

```yaml
# Client service - bootstrap.yml
spring:
  application:
    name: user-service
  cloud:
    config:
      uri: http://localhost:8888
      fail-fast: true  # Fail if config server unavailable
      retry:
        max-attempts: 5
```

---

## ขั้นตอนที่ 676-700: docker-compose สำหรับ Microservices

```yaml
version: '3.8'
services:
  
  # Infrastructure
  eureka-server:
    image: myapp/eureka-server:latest
    ports:
      - "8761:8761"
  
  config-server:
    image: myapp/config-server:latest
    ports:
      - "8888:8888"
    depends_on:
      - eureka-server
  
  api-gateway:
    image: myapp/api-gateway:latest
    ports:
      - "8080:8080"
    depends_on:
      - eureka-server
      - config-server
  
  # Services
  user-service:
    image: myapp/user-service:latest
    depends_on:
      - eureka-server
      - user-db
    environment:
      - SPRING_DATASOURCE_URL=jdbc:postgresql://user-db:5432/userdb
  
  product-service:
    image: myapp/product-service:latest
    depends_on:
      - eureka-server
      - product-db
  
  order-service:
    image: myapp/order-service:latest
    depends_on:
      - eureka-server
      - order-db
      - user-service
      - product-service
  
  # Databases (each service has own DB)
  user-db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: userdb
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
  
  product-db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: productdb
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
  
  order-db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: orderdb
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
  
  # Monitoring
  zipkin:
    image: openzipkin/zipkin:latest
    ports:
      - "9411:9411"
  
  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
  
  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
```

---

*[← Part 24: Docker Deployment](./part-24-docker-deployment.md) | [Part 26: Message Queue →](./part-26-message-queue.md)*

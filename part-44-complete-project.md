# Part 44: Complete E-Commerce Project - Architecture
## ขั้นตอนที่ 1356-1400

> **ระดับ:** ระดับโลก (World-Class)  
> **เวลาเรียน:** 15-20 ชั่วโมง (โปรเจคจริง)  
> **เป้าหมาย:** Build complete production-grade e-commerce system

---

## ขั้นตอนที่ 1356: Project Overview

```
ShopKub - E-Commerce Platform
================================

Microservices:
  1. user-service      (Port 8081) - Auth, Profiles
  2. product-service   (Port 8082) - Catalog, Inventory
  3. order-service     (Port 8083) - Orders, Cart
  4. payment-service   (Port 8084) - Payments, Billing
  5. notification-service (Port 8085) - Email, SMS, Push
  6. search-service    (Port 8086) - Elasticsearch
  7. api-gateway       (Port 8080) - Entry point

Supporting Services:
  - Config Server (Port 8888)
  - Service Registry/Eureka (Port 8761)
  - Kafka + Zookeeper (Message broker)
  - PostgreSQL (databases)
  - Redis (cache + session)
  - Elasticsearch (search)
  - MinIO (file storage, S3-compatible)
  - Zipkin (tracing)
  - Prometheus + Grafana (monitoring)

Tech Stack:
  - Spring Boot 3.2
  - Java 21 (Virtual Threads)
  - Spring Cloud 2023
  - Docker + Docker Compose
  - Kubernetes (production)
```

---

## ขั้นตอนที่ 1357: Project Structure

```
shopkub/
├── services/
│   ├── user-service/
│   │   ├── src/
│   │   ├── Dockerfile
│   │   └── pom.xml
│   ├── product-service/
│   ├── order-service/
│   ├── payment-service/
│   ├── notification-service/
│   ├── search-service/
│   └── api-gateway/
├── infrastructure/
│   ├── config-server/
│   └── service-registry/
├── config/
│   ├── application.yml         (shared config)
│   ├── user-service.yml
│   ├── product-service.yml
│   └── ...
├── docker-compose.yml
├── docker-compose.prod.yml
├── k8s/
│   ├── namespace.yaml
│   ├── configmaps/
│   ├── secrets/
│   └── services/
└── .github/
    └── workflows/
        ├── ci.yml
        └── deploy.yml
```

---

## ขั้นตอนที่ 1358: Parent POM

```xml
<!-- pom.xml (root) -->
<project>
    <groupId>com.shopkub</groupId>
    <artifactId>shopkub-parent</artifactId>
    <version>1.0.0-SNAPSHOT</version>
    <packaging>pom</packaging>
    
    <modules>
        <module>services/user-service</module>
        <module>services/product-service</module>
        <module>services/order-service</module>
        <module>services/payment-service</module>
        <module>services/notification-service</module>
        <module>services/api-gateway</module>
        <module>infrastructure/config-server</module>
        <module>infrastructure/service-registry</module>
    </modules>
    
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.2.0</version>
    </parent>
    
    <properties>
        <java.version>21</java.version>
        <spring-cloud.version>2023.0.0</spring-cloud.version>
        <mapstruct.version>1.5.5.Final</mapstruct.version>
    </properties>
    
    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>${spring-cloud.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>
    
    <!-- Common dependencies for all services -->
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-config</artifactId>
        </dependency>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>
    </dependencies>
</project>
```

---

## ขั้นตอนที่ 1359: Docker Compose (Development)

```yaml
# docker-compose.yml
version: '3.8'

services:
  # ===== Infrastructure =====
  
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_PASSWORD: secret
    volumes:
      - postgres-data:/var/lib/postgresql/data
      - ./scripts/init-db.sql:/docker-entrypoint-initdb.d/init.sql
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      retries: 5
  
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data
  
  kafka:
    image: confluentinc/cp-kafka:7.5.0
    depends_on:
      - zookeeper
    ports:
      - "9092:9092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092,PLAINTEXT_INTERNAL://kafka:29092
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_INTERNAL:PLAINTEXT
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
  
  zookeeper:
    image: confluentinc/cp-zookeeper:7.5.0
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
  
  minio:
    image: minio/minio
    command: server /data --console-address ":9001"
    environment:
      MINIO_ROOT_USER: admin
      MINIO_ROOT_PASSWORD: password123
    ports:
      - "9000:9000"
      - "9001:9001"
    volumes:
      - minio-data:/data
  
  elasticsearch:
    image: elasticsearch:8.12.0
    environment:
      discovery.type: single-node
      xpack.security.enabled: "false"
      ES_JAVA_OPTS: "-Xms512m -Xmx512m"
    ports:
      - "9200:9200"
    volumes:
      - es-data:/usr/share/elasticsearch/data
  
  zipkin:
    image: openzipkin/zipkin:latest
    ports:
      - "9411:9411"
  
  # ===== Spring Cloud Infrastructure =====
  
  config-server:
    build: ./infrastructure/config-server
    ports:
      - "8888:8888"
    environment:
      GIT_URI: file:///config
    volumes:
      - ./config:/config
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8888/actuator/health"]
      interval: 30s
      retries: 5
  
  service-registry:
    build: ./infrastructure/service-registry
    ports:
      - "8761:8761"
    depends_on:
      config-server:
        condition: service_healthy
  
  # ===== Microservices =====
  
  user-service:
    build: ./services/user-service
    ports:
      - "8081:8081"
    environment:
      SPRING_PROFILES_ACTIVE: docker
    depends_on:
      - postgres
      - redis
      - config-server
      - service-registry
  
  product-service:
    build: ./services/product-service
    ports:
      - "8082:8082"
    depends_on:
      - postgres
      - redis
      - elasticsearch
      - kafka
  
  order-service:
    build: ./services/order-service
    ports:
      - "8083:8083"
    depends_on:
      - postgres
      - kafka
      - user-service
      - product-service
  
  payment-service:
    build: ./services/payment-service
    ports:
      - "8084:8084"
    depends_on:
      - postgres
      - kafka
  
  notification-service:
    build: ./services/notification-service
    ports:
      - "8085:8085"
    depends_on:
      - kafka
  
  api-gateway:
    build: ./services/api-gateway
    ports:
      - "8080:8080"
    depends_on:
      - service-registry
      - user-service
      - product-service
      - order-service

volumes:
  postgres-data:
  redis-data:
  minio-data:
  es-data:
```

---

## ขั้นตอนที่ 1360: API Gateway Configuration

```java
// api-gateway/src/main/java/com/shopkub/gateway/GatewayConfig.java
@Configuration
@EnableDiscoveryClient
public class GatewayConfig {
    
    @Bean
    public RouteLocator routeLocator(RouteLocatorBuilder builder) {
        return builder.routes()
            
            // User Service
            .route("user-service", r -> r
                .path("/api/v1/users/**", "/api/v1/auth/**", "/oauth2/**")
                .filters(f -> f
                    .circuitBreaker(c -> c.setName("user-service-cb").setFallbackUri("/fallback/users"))
                    .retry(retry -> retry.setRetries(3).setStatuses(HttpStatus.SERVICE_UNAVAILABLE))
                    .requestRateLimiter(rl -> rl
                        .setRateLimiter(redisRateLimiter())
                        .setKeyResolver(ipKeyResolver())))
                .uri("lb://user-service"))
            
            // Product Service
            .route("product-service", r -> r
                .path("/api/v1/products/**", "/api/v1/categories/**")
                .filters(f -> f
                    .circuitBreaker(c -> c.setName("product-service-cb")))
                .uri("lb://product-service"))
            
            // Order Service (auth required)
            .route("order-service", r -> r
                .path("/api/v1/orders/**", "/api/v1/cart/**")
                .filters(f -> f
                    .filter(jwtAuthFilter.apply(new JwtAuthFilter.Config()))
                    .circuitBreaker(c -> c.setName("order-service-cb")))
                .uri("lb://order-service"))
            
            // Payment Service (auth required)
            .route("payment-service", r -> r
                .path("/api/v1/payments/**")
                .filters(f -> f
                    .filter(jwtAuthFilter.apply(new JwtAuthFilter.Config())))
                .uri("lb://payment-service"))
            
            .build();
    }
    
    @Bean
    public KeyResolver ipKeyResolver() {
        return exchange -> Mono.just(
            exchange.getRequest().getRemoteAddress().getAddress().getHostAddress()
        );
    }
    
    @Bean
    public RedisRateLimiter redisRateLimiter() {
        return new RedisRateLimiter(100, 200);  // replenish rate, burst capacity
    }
}
```

---

## ขั้นตอนที่ 1361-1400: Inter-Service Communication

```java
// order-service calling user-service and product-service
@Service
@RequiredArgsConstructor
public class OrderService {
    
    private final UserServiceClient userClient;
    private final ProductServiceClient productClient;
    private final OrderRepository orderRepository;
    private final KafkaTemplate<String, Object> kafkaTemplate;
    
    @Transactional
    public OrderResponse createOrder(String authToken, CreateOrderRequest request) {
        // Get current user
        UserResponse user = userClient.getCurrentUser(authToken);
        
        // Validate and calculate items
        List<OrderItem> items = request.items().stream().map(item -> {
            ProductResponse product = productClient.findById(item.productId());
            
            if (product.stock() < item.quantity()) {
                throw new InsufficientStockException(product.name());
            }
            
            return OrderItem.builder()
                .productId(product.id())
                .productName(product.name())
                .unitPrice(product.price())
                .quantity(item.quantity())
                .build();
        }).toList();
        
        BigDecimal total = items.stream()
            .map(i -> i.getUnitPrice().multiply(BigDecimal.valueOf(i.getQuantity())))
            .reduce(BigDecimal.ZERO, BigDecimal::add);
        
        Order order = Order.builder()
            .userId(user.id())
            .items(items)
            .totalAmount(total)
            .status(OrderStatus.PENDING)
            .shippingAddress(request.shippingAddress())
            .build();
        
        Order saved = orderRepository.save(order);
        
        // Publish event for payment, inventory, notification
        kafkaTemplate.send("orders.created", String.valueOf(saved.getId()),
            new OrderCreatedEvent(saved.getId(), user.id(), user.email(), items, total));
        
        return orderMapper.toResponse(saved);
    }
}

// Feign client for user-service
@FeignClient(name = "user-service", fallback = UserServiceClientFallback.class)
public interface UserServiceClient {
    
    @GetMapping("/api/v1/users/me")
    UserResponse getCurrentUser(@RequestHeader("Authorization") String token);
    
    @GetMapping("/api/v1/users/{id}")
    UserResponse findById(@PathVariable Long id);
}

@Component
public class UserServiceClientFallback implements UserServiceClient {
    
    @Override
    public UserResponse getCurrentUser(String token) {
        throw new ServiceUnavailableException("User service unavailable");
    }
    
    @Override
    public UserResponse findById(Long id) {
        throw new ServiceUnavailableException("User service unavailable");
    }
}
```

---

*[← Part 43: Infrastructure as Code](./part-43-iac.md) | [Part 45: Security Best Practices →](./part-45-security.md)*

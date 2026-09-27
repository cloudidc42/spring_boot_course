# Part 91: Complete Microservices Project - ShopHub Platform
## ขั้นตอนที่ 3241-3280

**ระดับ:** World-Class (ระดับโลก)
**เวลาเรียน:** 8-10 ชั่วโมง
**เป้าหมาย:** สร้าง Microservices Platform ชื่อ ShopHub ที่สมบูรณ์แบบ ประกอบด้วย 5 services พร้อม database แยกกัน, inter-service communication ผ่าน Feign และ Kafka, API Gateway, Shared Libraries และ Docker Compose

---

## ขั้นตอนที่ 3241: ภาพรวมของ ShopHub Platform

ShopHub เป็น e-commerce platform ที่ออกแบบด้วย Microservices Architecture ประกอบด้วย:

- **User Service** - จัดการข้อมูลผู้ใช้และ authentication
- **Product Service** - จัดการสินค้าและหมวดหมู่
- **Order Service** - จัดการคำสั่งซื้อ
- **Inventory Service** - จัดการสต็อกสินค้า
- **Notification Service** - ส่งการแจ้งเตือนผ่าน email และ push notification

```
ShopHub Architecture:

Client → API Gateway (8080)
            ├── User Service (8081) → PostgreSQL (user_db)
            ├── Product Service (8082) → PostgreSQL (product_db)
            ├── Order Service (8083) → PostgreSQL (order_db)
            ├── Inventory Service (8084) → PostgreSQL (inventory_db)
            └── Notification Service (8085) → MongoDB (notification_db)

Services communicate via:
- Feign Client (synchronous)
- Kafka (asynchronous events)

Infrastructure:
- Eureka (Service Discovery)
- Zipkin (Distributed Tracing)
- Redis (Caching)
```

## ขั้นตอนที่ 3242: โครงสร้าง Project และ Parent POM

โครงสร้างของ Maven Multi-Module Project สำหรับ ShopHub:

```
shophub/
├── pom.xml (parent)
├── shophub-common/
│   ├── pom.xml
│   └── src/main/java/com/shophub/common/
│       ├── dto/
│       ├── exception/
│       └── event/
├── api-gateway/
├── service-registry/
├── user-service/
├── product-service/
├── order-service/
├── inventory-service/
├── notification-service/
└── docker-compose.yml
```

**Parent POM (`shophub/pom.xml`):**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 
         https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.2.0</version>
        <relativePath/>
    </parent>

    <groupId>com.shophub</groupId>
    <artifactId>shophub-parent</artifactId>
    <version>1.0.0</version>
    <packaging>pom</packaging>

    <properties>
        <java.version>21</java.version>
        <spring-cloud.version>2023.0.0</spring-cloud.version>
    </properties>

    <modules>
        <module>shophub-common</module>
        <module>service-registry</module>
        <module>api-gateway</module>
        <module>user-service</module>
        <module>product-service</module>
        <module>order-service</module>
        <module>inventory-service</module>
        <module>notification-service</module>
    </modules>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>${spring-cloud.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
            <dependency>
                <groupId>com.shophub</groupId>
                <artifactId>shophub-common</artifactId>
                <version>${project.version}</version>
            </dependency>
        </dependencies>
    </dependencyManagement>

    <dependencies>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>
    </dependencies>
</project>
```

## ขั้นตอนที่ 3243: Shared Library (shophub-common)

Shared library ช่วยให้ทุก service ใช้ DTOs, exceptions และ events ร่วมกัน ลดการเขียนโค้ดซ้ำ

**`shophub-common/pom.xml`:**

```xml
<project>
    <parent>
        <groupId>com.shophub</groupId>
        <artifactId>shophub-parent</artifactId>
        <version>1.0.0</version>
    </parent>

    <artifactId>shophub-common</artifactId>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.kafka</groupId>
            <artifactId>spring-kafka</artifactId>
        </dependency>
    </dependencies>
</project>
```

**Common DTOs:**

```java
// shophub-common/src/main/java/com/shophub/common/dto/ApiResponse.java
package com.shophub.common.dto;

import com.fasterxml.jackson.annotation.JsonInclude;
import lombok.Builder;
import lombok.Data;

import java.time.LocalDateTime;

@Data
@Builder
@JsonInclude(JsonInclude.Include.NON_NULL)
public class ApiResponse<T> {
    private boolean success;
    private String message;
    private T data;
    private String errorCode;
    private LocalDateTime timestamp;

    public static <T> ApiResponse<T> success(T data) {
        return ApiResponse.<T>builder()
                .success(true)
                .data(data)
                .timestamp(LocalDateTime.now())
                .build();
    }

    public static <T> ApiResponse<T> success(String message, T data) {
        return ApiResponse.<T>builder()
                .success(true)
                .message(message)
                .data(data)
                .timestamp(LocalDateTime.now())
                .build();
    }

    public static <T> ApiResponse<T> error(String message, String errorCode) {
        return ApiResponse.<T>builder()
                .success(false)
                .message(message)
                .errorCode(errorCode)
                .timestamp(LocalDateTime.now())
                .build();
    }
}
```

```java
// shophub-common/src/main/java/com/shophub/common/dto/UserDto.java
package com.shophub.common.dto;

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class UserDto {
    private Long id;
    private String username;
    private String email;
    private String firstName;
    private String lastName;
    private String phone;
    private boolean active;
}
```

```java
// shophub-common/src/main/java/com/shophub/common/dto/ProductDto.java
package com.shophub.common.dto;

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;

import java.math.BigDecimal;

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class ProductDto {
    private Long id;
    private String sku;
    private String name;
    private String description;
    private BigDecimal price;
    private String category;
    private String imageUrl;
    private boolean available;
}
```

```java
// shophub-common/src/main/java/com/shophub/common/dto/OrderDto.java
package com.shophub.common.dto;

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;

import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.List;

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class OrderDto {
    private Long id;
    private String orderNumber;
    private Long userId;
    private List<OrderItemDto> items;
    private BigDecimal totalAmount;
    private String status;
    private String shippingAddress;
    private LocalDateTime createdAt;
}
```

```java
// shophub-common/src/main/java/com/shophub/common/dto/OrderItemDto.java
package com.shophub.common.dto;

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;

import java.math.BigDecimal;

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class OrderItemDto {
    private Long productId;
    private String productName;
    private String sku;
    private int quantity;
    private BigDecimal unitPrice;
    private BigDecimal subtotal;
}
```

**Common Exceptions:**

```java
// shophub-common/src/main/java/com/shophub/common/exception/ShopHubException.java
package com.shophub.common.exception;

import lombok.Getter;

@Getter
public class ShopHubException extends RuntimeException {
    private final String errorCode;
    private final int httpStatus;

    public ShopHubException(String message, String errorCode, int httpStatus) {
        super(message);
        this.errorCode = errorCode;
        this.httpStatus = httpStatus;
    }
}
```

```java
// shophub-common/src/main/java/com/shophub/common/exception/ResourceNotFoundException.java
package com.shophub.common.exception;

public class ResourceNotFoundException extends ShopHubException {
    public ResourceNotFoundException(String resource, Long id) {
        super(resource + " not found with id: " + id, "RESOURCE_NOT_FOUND", 404);
    }

    public ResourceNotFoundException(String resource, String identifier) {
        super(resource + " not found: " + identifier, "RESOURCE_NOT_FOUND", 404);
    }
}
```

```java
// shophub-common/src/main/java/com/shophub/common/exception/InsufficientStockException.java
package com.shophub.common.exception;

public class InsufficientStockException extends ShopHubException {
    public InsufficientStockException(String sku, int requested, int available) {
        super(String.format("Insufficient stock for %s: requested %d, available %d",
                sku, requested, available),
                "INSUFFICIENT_STOCK", 400);
    }
}
```

**Kafka Events:**

```java
// shophub-common/src/main/java/com/shophub/common/event/OrderCreatedEvent.java
package com.shophub.common.event;

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;

import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.List;

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class OrderCreatedEvent {
    private String eventId;
    private String orderNumber;
    private Long userId;
    private String userEmail;
    private List<OrderItemEvent> items;
    private BigDecimal totalAmount;
    private String shippingAddress;
    private LocalDateTime createdAt;
}
```

```java
// shophub-common/src/main/java/com/shophub/common/event/OrderItemEvent.java
package com.shophub.common.event;

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class OrderItemEvent {
    private Long productId;
    private String sku;
    private int quantity;
}
```

```java
// shophub-common/src/main/java/com/shophub/common/event/InventoryReservedEvent.java
package com.shophub.common.event;

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;

import java.time.LocalDateTime;

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class InventoryReservedEvent {
    private String eventId;
    private String orderNumber;
    private boolean success;
    private String failureReason;
    private LocalDateTime processedAt;
}
```

```java
// shophub-common/src/main/java/com/shophub/common/event/KafkaTopics.java
package com.shophub.common.event;

public final class KafkaTopics {
    public static final String ORDER_CREATED = "order.created";
    public static final String ORDER_CONFIRMED = "order.confirmed";
    public static final String ORDER_CANCELLED = "order.cancelled";
    public static final String INVENTORY_RESERVED = "inventory.reserved";
    public static final String INVENTORY_RELEASED = "inventory.released";
    public static final String NOTIFICATION_EMAIL = "notification.email";
    public static final String NOTIFICATION_PUSH = "notification.push";

    private KafkaTopics() {}
}
```

## ขั้นตอนที่ 3244: Service Registry (Eureka Server)

Eureka ทำหน้าที่เป็น Service Discovery ให้ services ทุกตัวลงทะเบียนและค้นหากันได้

```xml
<!-- service-registry/pom.xml -->
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
</dependencies>
```

```java
// service-registry/src/main/java/com/shophub/registry/ServiceRegistryApplication.java
package com.shophub.registry;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.netflix.eureka.server.EnableEurekaServer;

@SpringBootApplication
@EnableEurekaServer
public class ServiceRegistryApplication {
    public static void main(String[] args) {
        SpringApplication.run(ServiceRegistryApplication.class, args);
    }
}
```

```yaml
# service-registry/src/main/resources/application.yml
server:
  port: 8761

spring:
  application:
    name: service-registry

eureka:
  instance:
    hostname: localhost
  client:
    registerWithEureka: false
    fetchRegistry: false
    serviceUrl:
      defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/
  server:
    wait-time-in-ms-when-sync-empty: 0
    enableSelfPreservation: false
```

## ขั้นตอนที่ 3245: API Gateway

API Gateway ทำหน้าที่เป็นจุดเข้าหลักสำหรับ client ทุกประเภท

```xml
<!-- api-gateway/pom.xml -->
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-gateway</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
    <dependency>
        <groupId>io.micrometer</groupId>
        <artifactId>micrometer-tracing-bridge-brave</artifactId>
    </dependency>
</dependencies>
```

```java
// api-gateway/src/main/java/com/shophub/gateway/ApiGatewayApplication.java
package com.shophub.gateway;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;

@SpringBootApplication
@EnableDiscoveryClient
public class ApiGatewayApplication {
    public static void main(String[] args) {
        SpringApplication.run(ApiGatewayApplication.class, args);
    }
}
```

```yaml
# api-gateway/src/main/resources/application.yml
server:
  port: 8080

spring:
  application:
    name: api-gateway
  cloud:
    gateway:
      discovery:
        locator:
          enabled: true
          lower-case-service-id: true
      routes:
        - id: user-service
          uri: lb://user-service
          predicates:
            - Path=/api/users/**
          filters:
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 100
                redis-rate-limiter.burstCapacity: 200
            - AddRequestHeader=X-Gateway-Source, shophub-gateway
            - name: CircuitBreaker
              args:
                name: userServiceCB
                fallbackUri: forward:/fallback/user

        - id: product-service
          uri: lb://product-service
          predicates:
            - Path=/api/products/**
          filters:
            - name: CircuitBreaker
              args:
                name: productServiceCB
                fallbackUri: forward:/fallback/product

        - id: order-service
          uri: lb://order-service
          predicates:
            - Path=/api/orders/**
          filters:
            - name: AuthenticationFilter
            - name: CircuitBreaker
              args:
                name: orderServiceCB
                fallbackUri: forward:/fallback/order

        - id: inventory-service
          uri: lb://inventory-service
          predicates:
            - Path=/api/inventory/**
          filters:
            - name: CircuitBreaker
              args:
                name: inventoryServiceCB
                fallbackUri: forward:/fallback/inventory

eureka:
  client:
    serviceUrl:
      defaultZone: http://localhost:8761/eureka/

management:
  endpoints:
    web:
      exposure:
        include: "*"
  tracing:
    sampling:
      probability: 1.0
```

```java
// api-gateway/src/main/java/com/shophub/gateway/filter/AuthenticationFilter.java
package com.shophub.gateway.filter;

import org.springframework.cloud.gateway.filter.GatewayFilter;
import org.springframework.cloud.gateway.filter.factory.AbstractGatewayFilterFactory;
import org.springframework.http.HttpHeaders;
import org.springframework.http.HttpStatus;
import org.springframework.stereotype.Component;
import org.springframework.web.server.ServerWebExchange;
import reactor.core.publisher.Mono;

@Component
public class AuthenticationFilter extends AbstractGatewayFilterFactory<AuthenticationFilter.Config> {

    private final JwtUtil jwtUtil;

    public AuthenticationFilter(JwtUtil jwtUtil) {
        super(Config.class);
        this.jwtUtil = jwtUtil;
    }

    @Override
    public GatewayFilter apply(Config config) {
        return (exchange, chain) -> {
            String authHeader = exchange.getRequest()
                    .getHeaders()
                    .getFirst(HttpHeaders.AUTHORIZATION);

            if (authHeader == null || !authHeader.startsWith("Bearer ")) {
                return onError(exchange, HttpStatus.UNAUTHORIZED);
            }

            String token = authHeader.substring(7);
            if (!jwtUtil.validateToken(token)) {
                return onError(exchange, HttpStatus.UNAUTHORIZED);
            }

            String userId = jwtUtil.extractUserId(token);
            ServerWebExchange modifiedExchange = exchange.mutate()
                    .request(r -> r.header("X-User-Id", userId))
                    .build();

            return chain.filter(modifiedExchange);
        };
    }

    private Mono<Void> onError(ServerWebExchange exchange, HttpStatus status) {
        exchange.getResponse().setStatusCode(status);
        return exchange.getResponse().setComplete();
    }

    public static class Config {}
}
```

```java
// api-gateway/src/main/java/com/shophub/gateway/controller/FallbackController.java
package com.shophub.gateway.controller;

import com.shophub.common.dto.ApiResponse;
import org.springframework.http.HttpStatus;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.ResponseStatus;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("/fallback")
public class FallbackController {

    @RequestMapping("/user")
    @ResponseStatus(HttpStatus.SERVICE_UNAVAILABLE)
    public ApiResponse<Void> userFallback() {
        return ApiResponse.error("User Service is temporarily unavailable", "SERVICE_UNAVAILABLE");
    }

    @RequestMapping("/product")
    @ResponseStatus(HttpStatus.SERVICE_UNAVAILABLE)
    public ApiResponse<Void> productFallback() {
        return ApiResponse.error("Product Service is temporarily unavailable", "SERVICE_UNAVAILABLE");
    }

    @RequestMapping("/order")
    @ResponseStatus(HttpStatus.SERVICE_UNAVAILABLE)
    public ApiResponse<Void> orderFallback() {
        return ApiResponse.error("Order Service is temporarily unavailable", "SERVICE_UNAVAILABLE");
    }

    @RequestMapping("/inventory")
    @ResponseStatus(HttpStatus.SERVICE_UNAVAILABLE)
    public ApiResponse<Void> inventoryFallback() {
        return ApiResponse.error("Inventory Service is temporarily unavailable", "SERVICE_UNAVAILABLE");
    }
}
```

## ขั้นตอนที่ 3246: User Service

User Service จัดการข้อมูลผู้ใช้, authentication และ authorization

```xml
<!-- user-service/pom.xml -->
<dependencies>
    <dependency>
        <groupId>com.shophub</groupId>
        <artifactId>shophub-common</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-security</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
    </dependency>
    <dependency>
        <groupId>org.postgresql</groupId>
        <artifactId>postgresql</artifactId>
    </dependency>
    <dependency>
        <groupId>io.jsonwebtoken</groupId>
        <artifactId>jjwt-api</artifactId>
        <version>0.12.3</version>
    </dependency>
</dependencies>
```

```java
// user-service/src/main/java/com/shophub/user/domain/User.java
package com.shophub.user.domain;

import jakarta.persistence.*;
import lombok.*;
import org.springframework.data.annotation.CreatedDate;
import org.springframework.data.annotation.LastModifiedDate;
import org.springframework.data.jpa.domain.support.AuditingEntityListener;

import java.time.LocalDateTime;
import java.util.Set;

@Entity
@Table(name = "users")
@EntityListeners(AuditingEntityListener.class)
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(unique = true, nullable = false)
    private String username;

    @Column(unique = true, nullable = false)
    private String email;

    @Column(nullable = false)
    private String password;

    private String firstName;
    private String lastName;
    private String phone;

    @Builder.Default
    private boolean active = true;

    @Builder.Default
    private boolean emailVerified = false;

    @ElementCollection(fetch = FetchType.EAGER)
    @CollectionTable(name = "user_roles", joinColumns = @JoinColumn(name = "user_id"))
    @Column(name = "role")
    private Set<String> roles;

    @CreatedDate
    private LocalDateTime createdAt;

    @LastModifiedDate
    private LocalDateTime updatedAt;
}
```

```java
// user-service/src/main/java/com/shophub/user/service/UserService.java
package com.shophub.user.service;

import com.shophub.common.dto.UserDto;
import com.shophub.common.exception.ResourceNotFoundException;
import com.shophub.user.domain.User;
import com.shophub.user.dto.RegisterRequest;
import com.shophub.user.dto.LoginRequest;
import com.shophub.user.dto.AuthResponse;
import com.shophub.user.repository.UserRepository;
import com.shophub.user.security.JwtService;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.security.authentication.AuthenticationManager;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.Set;

@Service
@RequiredArgsConstructor
@Slf4j
@Transactional
public class UserService {

    private final UserRepository userRepository;
    private final PasswordEncoder passwordEncoder;
    private final JwtService jwtService;
    private final AuthenticationManager authenticationManager;

    public AuthResponse register(RegisterRequest request) {
        if (userRepository.existsByEmail(request.getEmail())) {
            throw new IllegalArgumentException("Email already registered: " + request.getEmail());
        }

        User user = User.builder()
                .username(request.getUsername())
                .email(request.getEmail())
                .password(passwordEncoder.encode(request.getPassword()))
                .firstName(request.getFirstName())
                .lastName(request.getLastName())
                .phone(request.getPhone())
                .roles(Set.of("ROLE_USER"))
                .build();

        User savedUser = userRepository.save(user);
        log.info("New user registered: {}", savedUser.getEmail());

        String token = jwtService.generateToken(savedUser);
        return AuthResponse.builder()
                .token(token)
                .userId(savedUser.getId())
                .email(savedUser.getEmail())
                .build();
    }

    public AuthResponse login(LoginRequest request) {
        authenticationManager.authenticate(
                new UsernamePasswordAuthenticationToken(request.getEmail(), request.getPassword())
        );

        User user = userRepository.findByEmail(request.getEmail())
                .orElseThrow(() -> new ResourceNotFoundException("User", request.getEmail()));

        String token = jwtService.generateToken(user);
        return AuthResponse.builder()
                .token(token)
                .userId(user.getId())
                .email(user.getEmail())
                .build();
    }

    @Transactional(readOnly = true)
    public UserDto getUserById(Long id) {
        return userRepository.findById(id)
                .map(this::toDto)
                .orElseThrow(() -> new ResourceNotFoundException("User", id));
    }

    @Transactional(readOnly = true)
    public UserDto getUserByEmail(String email) {
        return userRepository.findByEmail(email)
                .map(this::toDto)
                .orElseThrow(() -> new ResourceNotFoundException("User", email));
    }

    private UserDto toDto(User user) {
        return UserDto.builder()
                .id(user.getId())
                .username(user.getUsername())
                .email(user.getEmail())
                .firstName(user.getFirstName())
                .lastName(user.getLastName())
                .phone(user.getPhone())
                .active(user.isActive())
                .build();
    }
}
```

## ขั้นตอนที่ 3247: Product Service

Product Service จัดการข้อมูลสินค้า, หมวดหมู่ และการค้นหา

```java
// product-service/src/main/java/com/shophub/product/domain/Product.java
package com.shophub.product.domain;

import jakarta.persistence.*;
import lombok.*;
import org.springframework.data.annotation.CreatedDate;
import org.springframework.data.annotation.LastModifiedDate;
import org.springframework.data.jpa.domain.support.AuditingEntityListener;

import java.math.BigDecimal;
import java.time.LocalDateTime;

@Entity
@Table(name = "products")
@EntityListeners(AuditingEntityListener.class)
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class Product {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(unique = true, nullable = false)
    private String sku;

    @Column(nullable = false)
    private String name;

    @Column(columnDefinition = "TEXT")
    private String description;

    @Column(nullable = false, precision = 10, scale = 2)
    private BigDecimal price;

    private String category;
    private String brand;
    private String imageUrl;

    @Builder.Default
    private boolean available = true;

    @CreatedDate
    private LocalDateTime createdAt;

    @LastModifiedDate
    private LocalDateTime updatedAt;
}
```

```java
// product-service/src/main/java/com/shophub/product/service/ProductService.java
package com.shophub.product.service;

import com.shophub.common.dto.ProductDto;
import com.shophub.common.exception.ResourceNotFoundException;
import com.shophub.product.domain.Product;
import com.shophub.product.repository.ProductRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.cache.annotation.Cacheable;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;
import java.util.stream.Collectors;

@Service
@RequiredArgsConstructor
@Slf4j
@Transactional
public class ProductService {

    private final ProductRepository productRepository;

    @Transactional(readOnly = true)
    @Cacheable(value = "products", key = "#id")
    public ProductDto getProductById(Long id) {
        return productRepository.findById(id)
                .map(this::toDto)
                .orElseThrow(() -> new ResourceNotFoundException("Product", id));
    }

    @Transactional(readOnly = true)
    @Cacheable(value = "products", key = "'sku:' + #sku")
    public ProductDto getProductBySku(String sku) {
        return productRepository.findBySku(sku)
                .map(this::toDto)
                .orElseThrow(() -> new ResourceNotFoundException("Product", sku));
    }

    @Transactional(readOnly = true)
    public Page<ProductDto> searchProducts(String keyword, String category, Pageable pageable) {
        if (category != null && !category.isEmpty()) {
            return productRepository.findByCategoryAndNameContainingIgnoreCase(
                    category, keyword, pageable).map(this::toDto);
        }
        return productRepository.findByNameContainingIgnoreCase(keyword, pageable)
                .map(this::toDto);
    }

    @Transactional(readOnly = true)
    public List<ProductDto> getProductsByIds(List<Long> ids) {
        return productRepository.findAllById(ids).stream()
                .map(this::toDto)
                .collect(Collectors.toList());
    }

    private ProductDto toDto(Product product) {
        return ProductDto.builder()
                .id(product.getId())
                .sku(product.getSku())
                .name(product.getName())
                .description(product.getDescription())
                .price(product.getPrice())
                .category(product.getCategory())
                .imageUrl(product.getImageUrl())
                .available(product.isAvailable())
                .build();
    }
}
```

## ขั้นตอนที่ 3248: Order Service พร้อม Feign Client

Order Service ใช้ Feign Client เพื่อสื่อสารกับ User Service และ Product Service แบบ synchronous

```xml
<!-- order-service/pom.xml เพิ่ม Feign dependencies -->
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-openfeign</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-circuitbreaker-resilience4j</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.kafka</groupId>
        <artifactId>spring-kafka</artifactId>
    </dependency>
</dependencies>
```

```java
// order-service/src/main/java/com/shophub/order/client/UserClient.java
package com.shophub.order.client;

import com.shophub.common.dto.ApiResponse;
import com.shophub.common.dto.UserDto;
import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;

@FeignClient(
    name = "user-service",
    fallback = UserClientFallback.class
)
public interface UserClient {

    @GetMapping("/api/users/{id}")
    ApiResponse<UserDto> getUserById(@PathVariable Long id);
}
```

```java
// order-service/src/main/java/com/shophub/order/client/UserClientFallback.java
package com.shophub.order.client;

import com.shophub.common.dto.ApiResponse;
import com.shophub.common.dto.UserDto;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Component;

@Component
@Slf4j
public class UserClientFallback implements UserClient {

    @Override
    public ApiResponse<UserDto> getUserById(Long id) {
        log.warn("User service unavailable, using fallback for user id: {}", id);
        return ApiResponse.error("User service unavailable", "SERVICE_UNAVAILABLE");
    }
}
```

```java
// order-service/src/main/java/com/shophub/order/client/ProductClient.java
package com.shophub.order.client;

import com.shophub.common.dto.ApiResponse;
import com.shophub.common.dto.ProductDto;
import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestParam;

import java.util.List;

@FeignClient(
    name = "product-service",
    fallback = ProductClientFallback.class
)
public interface ProductClient {

    @GetMapping("/api/products/{id}")
    ApiResponse<ProductDto> getProductById(@PathVariable Long id);

    @GetMapping("/api/products/batch")
    ApiResponse<List<ProductDto>> getProductsByIds(@RequestParam List<Long> ids);
}
```

```java
// order-service/src/main/java/com/shophub/order/client/InventoryClient.java
package com.shophub.order.client;

import com.shophub.common.dto.ApiResponse;
import com.shophub.order.dto.StockCheckRequest;
import com.shophub.order.dto.StockCheckResponse;
import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;

@FeignClient(
    name = "inventory-service",
    fallback = InventoryClientFallback.class
)
public interface InventoryClient {

    @PostMapping("/api/inventory/check-and-reserve")
    ApiResponse<StockCheckResponse> checkAndReserveStock(@RequestBody StockCheckRequest request);
}
```

```java
// order-service/src/main/java/com/shophub/order/domain/Order.java
package com.shophub.order.domain;

import jakarta.persistence.*;
import lombok.*;
import org.springframework.data.annotation.CreatedDate;
import org.springframework.data.jpa.domain.support.AuditingEntityListener;

import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.List;

@Entity
@Table(name = "orders")
@EntityListeners(AuditingEntityListener.class)
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class Order {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(unique = true, nullable = false)
    private String orderNumber;

    @Column(nullable = false)
    private Long userId;

    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<OrderItem> items;

    @Column(nullable = false, precision = 10, scale = 2)
    private BigDecimal totalAmount;

    @Enumerated(EnumType.STRING)
    @Builder.Default
    private OrderStatus status = OrderStatus.PENDING;

    private String shippingAddress;

    @CreatedDate
    private LocalDateTime createdAt;

    public enum OrderStatus {
        PENDING, CONFIRMED, PROCESSING, SHIPPED, DELIVERED, CANCELLED
    }
}
```

```java
// order-service/src/main/java/com/shophub/order/service/OrderService.java
package com.shophub.order.service;

import com.shophub.common.dto.ProductDto;
import com.shophub.common.dto.UserDto;
import com.shophub.common.event.KafkaTopics;
import com.shophub.common.event.OrderCreatedEvent;
import com.shophub.common.event.OrderItemEvent;
import com.shophub.common.exception.ResourceNotFoundException;
import com.shophub.order.client.InventoryClient;
import com.shophub.order.client.ProductClient;
import com.shophub.order.client.UserClient;
import com.shophub.order.domain.Order;
import com.shophub.order.domain.OrderItem;
import com.shophub.order.dto.CreateOrderRequest;
import com.shophub.order.dto.StockCheckRequest;
import com.shophub.order.repository.OrderRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.*;
import java.util.stream.Collectors;

@Service
@RequiredArgsConstructor
@Slf4j
@Transactional
public class OrderService {

    private final OrderRepository orderRepository;
    private final UserClient userClient;
    private final ProductClient productClient;
    private final InventoryClient inventoryClient;
    private final KafkaTemplate<String, Object> kafkaTemplate;

    public Order createOrder(CreateOrderRequest request, Long userId) {
        // 1. ตรวจสอบว่า user มีอยู่จริง
        var userResponse = userClient.getUserById(userId);
        if (!userResponse.isSuccess()) {
            throw new ResourceNotFoundException("User", userId);
        }
        UserDto user = userResponse.getData();

        // 2. ดึงข้อมูลสินค้า
        List<Long> productIds = request.getItems().stream()
                .map(item -> item.getProductId())
                .collect(Collectors.toList());

        var productsResponse = productClient.getProductsByIds(productIds);
        if (!productsResponse.isSuccess()) {
            throw new RuntimeException("Failed to fetch product details");
        }

        Map<Long, ProductDto> productMap = productsResponse.getData().stream()
                .collect(Collectors.toMap(ProductDto::getId, p -> p));

        // 3. ตรวจสอบ stock และจองไว้
        StockCheckRequest stockRequest = new StockCheckRequest();
        stockRequest.setItems(request.getItems().stream()
                .map(item -> new StockCheckRequest.Item(item.getProductId(), 
                        productMap.get(item.getProductId()).getSku(), 
                        item.getQuantity()))
                .collect(Collectors.toList()));

        var stockResponse = inventoryClient.checkAndReserveStock(stockRequest);
        if (!stockResponse.isSuccess() || !stockResponse.getData().isAvailable()) {
            throw new RuntimeException("Insufficient stock: " + stockResponse.getData().getMessage());
        }

        // 4. สร้าง order
        String orderNumber = "ORD-" + System.currentTimeMillis();
        List<OrderItem> orderItems = new ArrayList<>();
        BigDecimal total = BigDecimal.ZERO;

        for (var item : request.getItems()) {
            ProductDto product = productMap.get(item.getProductId());
            BigDecimal subtotal = product.getPrice().multiply(BigDecimal.valueOf(item.getQuantity()));
            total = total.add(subtotal);

            orderItems.add(OrderItem.builder()
                    .productId(product.getId())
                    .productName(product.getName())
                    .sku(product.getSku())
                    .quantity(item.getQuantity())
                    .unitPrice(product.getPrice())
                    .subtotal(subtotal)
                    .build());
        }

        Order order = Order.builder()
                .orderNumber(orderNumber)
                .userId(userId)
                .items(orderItems)
                .totalAmount(total)
                .shippingAddress(request.getShippingAddress())
                .build();

        orderItems.forEach(item -> item.setOrder(order));
        Order savedOrder = orderRepository.save(order);

        // 5. ส่ง event ไปยัง Kafka
        OrderCreatedEvent event = OrderCreatedEvent.builder()
                .eventId(UUID.randomUUID().toString())
                .orderNumber(savedOrder.getOrderNumber())
                .userId(userId)
                .userEmail(user.getEmail())
                .items(request.getItems().stream()
                        .map(item -> OrderItemEvent.builder()
                                .productId(item.getProductId())
                                .sku(productMap.get(item.getProductId()).getSku())
                                .quantity(item.getQuantity())
                                .build())
                        .collect(Collectors.toList()))
                .totalAmount(total)
                .shippingAddress(request.getShippingAddress())
                .createdAt(LocalDateTime.now())
                .build();

        kafkaTemplate.send(KafkaTopics.ORDER_CREATED, savedOrder.getOrderNumber(), event);
        log.info("Order created: {} for user: {}", savedOrder.getOrderNumber(), userId);

        return savedOrder;
    }
}
```

## ขั้นตอนที่ 3249: Inventory Service

Inventory Service จัดการสต็อกสินค้า รับ order events จาก Kafka และอัปเดตสต็อก

```java
// inventory-service/src/main/java/com/shophub/inventory/domain/Inventory.java
package com.shophub.inventory.domain;

import jakarta.persistence.*;
import lombok.*;

@Entity
@Table(name = "inventory")
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class Inventory {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(unique = true, nullable = false)
    private String sku;

    private Long productId;

    @Column(nullable = false)
    private int availableQuantity;

    @Column(nullable = false)
    private int reservedQuantity;

    @Column(nullable = false)
    private int reorderLevel;

    @Version
    private Long version; // สำหรับ optimistic locking

    public int getTotalQuantity() {
        return availableQuantity + reservedQuantity;
    }

    public boolean canReserve(int quantity) {
        return availableQuantity >= quantity;
    }

    public void reserve(int quantity) {
        if (!canReserve(quantity)) {
            throw new IllegalStateException("Cannot reserve " + quantity + " units for " + sku);
        }
        this.availableQuantity -= quantity;
        this.reservedQuantity += quantity;
    }

    public void release(int quantity) {
        this.reservedQuantity -= quantity;
        this.availableQuantity += quantity;
    }

    public void deduct(int quantity) {
        this.reservedQuantity -= quantity;
    }
}
```

```java
// inventory-service/src/main/java/com/shophub/inventory/service/InventoryService.java
package com.shophub.inventory.service;

import com.shophub.common.event.InventoryReservedEvent;
import com.shophub.common.event.KafkaTopics;
import com.shophub.common.event.OrderCreatedEvent;
import com.shophub.common.exception.InsufficientStockException;
import com.shophub.inventory.domain.Inventory;
import com.shophub.inventory.dto.StockCheckRequest;
import com.shophub.inventory.dto.StockCheckResponse;
import com.shophub.inventory.repository.InventoryRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.time.LocalDateTime;
import java.util.UUID;

@Service
@RequiredArgsConstructor
@Slf4j
public class InventoryService {

    private final InventoryRepository inventoryRepository;
    private final KafkaTemplate<String, Object> kafkaTemplate;

    @Transactional
    public StockCheckResponse checkAndReserveStock(StockCheckRequest request) {
        // ตรวจสอบ stock ทั้งหมดก่อน
        for (var item : request.getItems()) {
            Inventory inventory = inventoryRepository.findBySku(item.getSku())
                    .orElseThrow(() -> new RuntimeException("SKU not found: " + item.getSku()));

            if (!inventory.canReserve(item.getQuantity())) {
                return StockCheckResponse.builder()
                        .available(false)
                        .message("Insufficient stock for " + item.getSku())
                        .build();
            }
        }

        // จอง stock
        for (var item : request.getItems()) {
            Inventory inventory = inventoryRepository.findBySkuWithLock(item.getSku())
                    .orElseThrow();
            inventory.reserve(item.getQuantity());
            inventoryRepository.save(inventory);
        }

        return StockCheckResponse.builder()
                .available(true)
                .message("Stock reserved successfully")
                .build();
    }

    // รับ event จาก Kafka เมื่อ order ถูกยืนยัน
    @KafkaListener(topics = KafkaTopics.ORDER_CREATED, groupId = "inventory-service")
    @Transactional
    public void handleOrderCreated(OrderCreatedEvent event) {
        log.info("Processing inventory for order: {}", event.getOrderNumber());

        boolean success = true;
        String failureReason = null;

        try {
            for (var item : event.getItems()) {
                Inventory inventory = inventoryRepository.findBySku(item.getSku())
                        .orElseThrow(() -> new RuntimeException("SKU not found: " + item.getSku()));
                inventory.deduct(item.getQuantity());
                inventoryRepository.save(inventory);

                // ตรวจสอบว่าถึง reorder level หรือไม่
                if (inventory.getAvailableQuantity() <= inventory.getReorderLevel()) {
                    log.warn("Low stock alert for SKU: {}, available: {}",
                            inventory.getSku(), inventory.getAvailableQuantity());
                }
            }
        } catch (Exception e) {
            success = false;
            failureReason = e.getMessage();
            log.error("Failed to process inventory for order: {}", event.getOrderNumber(), e);
        }

        // ส่ง event กลับ
        InventoryReservedEvent responseEvent = InventoryReservedEvent.builder()
                .eventId(UUID.randomUUID().toString())
                .orderNumber(event.getOrderNumber())
                .success(success)
                .failureReason(failureReason)
                .processedAt(LocalDateTime.now())
                .build();

        kafkaTemplate.send(KafkaTopics.INVENTORY_RESERVED, event.getOrderNumber(), responseEvent);
    }
}
```

## ขั้นตอนที่ 3250: Notification Service

Notification Service รับ events จาก Kafka และส่งการแจ้งเตือนผ่านช่องทางต่าง ๆ

```java
// notification-service/src/main/java/com/shophub/notification/domain/Notification.java
package com.shophub.notification.domain;

import lombok.*;
import org.springframework.data.annotation.Id;
import org.springframework.data.mongodb.core.mapping.Document;

import java.time.LocalDateTime;

@Document(collection = "notifications")
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class Notification {

    @Id
    private String id;
    private Long userId;
    private String type;
    private String subject;
    private String content;
    private String status;
    private String channel; // EMAIL, PUSH, SMS
    private String referenceId; // order number หรือ reference อื่น ๆ
    private LocalDateTime sentAt;
    private LocalDateTime createdAt;
}
```

```java
// notification-service/src/main/java/com/shophub/notification/service/NotificationService.java
package com.shophub.notification.service;

import com.shophub.common.event.InventoryReservedEvent;
import com.shophub.common.event.KafkaTopics;
import com.shophub.common.event.OrderCreatedEvent;
import com.shophub.notification.domain.Notification;
import com.shophub.notification.repository.NotificationRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.mail.SimpleMailMessage;
import org.springframework.mail.javamail.JavaMailSender;
import org.springframework.stereotype.Service;

import java.time.LocalDateTime;

@Service
@RequiredArgsConstructor
@Slf4j
public class NotificationService {

    private final NotificationRepository notificationRepository;
    private final JavaMailSender mailSender;

    @KafkaListener(topics = KafkaTopics.ORDER_CREATED, groupId = "notification-service")
    public void handleOrderCreated(OrderCreatedEvent event) {
        log.info("Sending order confirmation for: {}", event.getOrderNumber());

        String subject = "Order Confirmation - " + event.getOrderNumber();
        String content = buildOrderConfirmationEmail(event);

        sendEmail(event.getUserEmail(), subject, content);

        Notification notification = Notification.builder()
                .userId(event.getUserId())
                .type("ORDER_CONFIRMATION")
                .subject(subject)
                .content(content)
                .status("SENT")
                .channel("EMAIL")
                .referenceId(event.getOrderNumber())
                .sentAt(LocalDateTime.now())
                .createdAt(LocalDateTime.now())
                .build();

        notificationRepository.save(notification);
    }

    @KafkaListener(topics = KafkaTopics.INVENTORY_RESERVED, groupId = "notification-service")
    public void handleInventoryReserved(InventoryReservedEvent event) {
        if (!event.isSuccess()) {
            log.warn("Inventory reservation failed for order: {}", event.getOrderNumber());
            // อาจส่ง notification แจ้งว่า order มีปัญหา
        }
    }

    private void sendEmail(String to, String subject, String content) {
        try {
            SimpleMailMessage message = new SimpleMailMessage();
            message.setTo(to);
            message.setSubject(subject);
            message.setText(content);
            mailSender.send(message);
        } catch (Exception e) {
            log.error("Failed to send email to {}: {}", to, e.getMessage());
        }
    }

    private String buildOrderConfirmationEmail(OrderCreatedEvent event) {
        StringBuilder sb = new StringBuilder();
        sb.append("Thank you for your order!\n\n");
        sb.append("Order Number: ").append(event.getOrderNumber()).append("\n");
        sb.append("Total Amount: $").append(event.getTotalAmount()).append("\n\n");
        sb.append("Items:\n");
        event.getItems().forEach(item ->
                sb.append("- Product ID: ").append(item.getProductId())
                  .append(", Quantity: ").append(item.getQuantity()).append("\n")
        );
        sb.append("\nShipping to: ").append(event.getShippingAddress());
        return sb.toString();
    }
}
```

## ขั้นตอนที่ 3251: Kafka Configuration

การตั้งค่า Kafka สำหรับทุก service

```yaml
# shared kafka config (ใส่ใน application.yml ของแต่ละ service)
spring:
  kafka:
    bootstrap-servers: localhost:9092
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
      acks: all
      retries: 3
      properties:
        spring.json.add.type.headers: false
    consumer:
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
      auto-offset-reset: earliest
      properties:
        spring.json.trusted.packages: "com.shophub.common.event"
    listener:
      ack-mode: manual
```

```java
// shared/KafkaConfig.java (ใส่ใน shophub-common)
package com.shophub.common.config;

import org.apache.kafka.clients.admin.AdminClientConfig;
import org.apache.kafka.clients.admin.NewTopic;
import com.shophub.common.event.KafkaTopics;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.kafka.config.TopicBuilder;
import org.springframework.kafka.core.KafkaAdmin;

import java.util.HashMap;
import java.util.Map;

@Configuration
public class KafkaConfig {

    @Value("${spring.kafka.bootstrap-servers}")
    private String bootstrapServers;

    @Bean
    public KafkaAdmin kafkaAdmin() {
        Map<String, Object> configs = new HashMap<>();
        configs.put(AdminClientConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
        return new KafkaAdmin(configs);
    }

    @Bean
    public NewTopic orderCreatedTopic() {
        return TopicBuilder.name(KafkaTopics.ORDER_CREATED)
                .partitions(3)
                .replicas(1)
                .build();
    }

    @Bean
    public NewTopic inventoryReservedTopic() {
        return TopicBuilder.name(KafkaTopics.INVENTORY_RESERVED)
                .partitions(3)
                .replicas(1)
                .build();
    }

    @Bean
    public NewTopic notificationEmailTopic() {
        return TopicBuilder.name(KafkaTopics.NOTIFICATION_EMAIL)
                .partitions(3)
                .replicas(1)
                .build();
    }
}
```

## ขั้นตอนที่ 3252: Docker Compose สำหรับ Infrastructure

```yaml
# docker-compose.yml
version: '3.8'

services:
  # Databases
  postgres-user:
    image: postgres:15
    environment:
      POSTGRES_DB: user_db
      POSTGRES_USER: shophub
      POSTGRES_PASSWORD: shophub123
    ports:
      - "5432:5432"
    volumes:
      - user_db_data:/var/lib/postgresql/data
    networks:
      - shophub-network

  postgres-product:
    image: postgres:15
    environment:
      POSTGRES_DB: product_db
      POSTGRES_USER: shophub
      POSTGRES_PASSWORD: shophub123
    ports:
      - "5433:5432"
    volumes:
      - product_db_data:/var/lib/postgresql/data
    networks:
      - shophub-network

  postgres-order:
    image: postgres:15
    environment:
      POSTGRES_DB: order_db
      POSTGRES_USER: shophub
      POSTGRES_PASSWORD: shophub123
    ports:
      - "5434:5432"
    volumes:
      - order_db_data:/var/lib/postgresql/data
    networks:
      - shophub-network

  postgres-inventory:
    image: postgres:15
    environment:
      POSTGRES_DB: inventory_db
      POSTGRES_USER: shophub
      POSTGRES_PASSWORD: shophub123
    ports:
      - "5435:5432"
    volumes:
      - inventory_db_data:/var/lib/postgresql/data
    networks:
      - shophub-network

  mongodb:
    image: mongo:7
    environment:
      MONGO_INITDB_DATABASE: notification_db
    ports:
      - "27017:27017"
    volumes:
      - mongodb_data:/data/db
    networks:
      - shophub-network

  # Message Queue
  zookeeper:
    image: confluentinc/cp-zookeeper:7.5.0
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
    networks:
      - shophub-network

  kafka:
    image: confluentinc/cp-kafka:7.5.0
    depends_on:
      - zookeeper
    ports:
      - "9092:9092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_AUTO_CREATE_TOPICS_ENABLE: 'true'
    networks:
      - shophub-network

  # Cache
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    networks:
      - shophub-network

  # Service Discovery
  service-registry:
    build: ./service-registry
    ports:
      - "8761:8761"
    networks:
      - shophub-network
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8761/actuator/health"]
      interval: 30s
      timeout: 10s
      retries: 5

  # API Gateway
  api-gateway:
    build: ./api-gateway
    ports:
      - "8080:8080"
    depends_on:
      service-registry:
        condition: service_healthy
    environment:
      EUREKA_CLIENT_SERVICEURL_DEFAULTZONE: http://service-registry:8761/eureka/
      SPRING_REDIS_HOST: redis
    networks:
      - shophub-network

  # Microservices
  user-service:
    build: ./user-service
    ports:
      - "8081:8081"
    depends_on:
      - postgres-user
      - service-registry
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres-user:5432/user_db
      SPRING_DATASOURCE_USERNAME: shophub
      SPRING_DATASOURCE_PASSWORD: shophub123
      EUREKA_CLIENT_SERVICEURL_DEFAULTZONE: http://service-registry:8761/eureka/
    networks:
      - shophub-network

  product-service:
    build: ./product-service
    ports:
      - "8082:8082"
    depends_on:
      - postgres-product
      - service-registry
      - redis
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres-product:5432/product_db
      SPRING_DATASOURCE_USERNAME: shophub
      SPRING_DATASOURCE_PASSWORD: shophub123
      SPRING_REDIS_HOST: redis
      EUREKA_CLIENT_SERVICEURL_DEFAULTZONE: http://service-registry:8761/eureka/
    networks:
      - shophub-network

  order-service:
    build: ./order-service
    ports:
      - "8083:8083"
    depends_on:
      - postgres-order
      - kafka
      - service-registry
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres-order:5432/order_db
      SPRING_DATASOURCE_USERNAME: shophub
      SPRING_DATASOURCE_PASSWORD: shophub123
      SPRING_KAFKA_BOOTSTRAP_SERVERS: kafka:9092
      EUREKA_CLIENT_SERVICEURL_DEFAULTZONE: http://service-registry:8761/eureka/
    networks:
      - shophub-network

  inventory-service:
    build: ./inventory-service
    ports:
      - "8084:8084"
    depends_on:
      - postgres-inventory
      - kafka
      - service-registry
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres-inventory:5432/inventory_db
      SPRING_DATASOURCE_USERNAME: shophub
      SPRING_DATASOURCE_PASSWORD: shophub123
      SPRING_KAFKA_BOOTSTRAP_SERVERS: kafka:9092
      EUREKA_CLIENT_SERVICEURL_DEFAULTZONE: http://service-registry:8761/eureka/
    networks:
      - shophub-network

  notification-service:
    build: ./notification-service
    ports:
      - "8085:8085"
    depends_on:
      - mongodb
      - kafka
      - service-registry
    environment:
      SPRING_DATA_MONGODB_URI: mongodb://mongodb:27017/notification_db
      SPRING_KAFKA_BOOTSTRAP_SERVERS: kafka:9092
      EUREKA_CLIENT_SERVICEURL_DEFAULTZONE: http://service-registry:8761/eureka/
    networks:
      - shophub-network

  # Monitoring
  zipkin:
    image: openzipkin/zipkin:3
    ports:
      - "9411:9411"
    networks:
      - shophub-network

  kafka-ui:
    image: provectuslabs/kafka-ui:latest
    ports:
      - "8090:8080"
    environment:
      KAFKA_CLUSTERS_0_NAME: local
      KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: kafka:9092
    depends_on:
      - kafka
    networks:
      - shophub-network

volumes:
  user_db_data:
  product_db_data:
  order_db_data:
  inventory_db_data:
  mongodb_data:

networks:
  shophub-network:
    driver: bridge
```

## ขั้นตอนที่ 3253: Distributed Tracing กับ Zipkin

การตั้งค่า distributed tracing เพื่อติดตาม request ข้าม services

```xml
<!-- เพิ่มใน pom.xml ของทุก service -->
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-brave</artifactId>
</dependency>
<dependency>
    <groupId>io.zipkin.reporter2</groupId>
    <artifactId>zipkin-reporter-brave</artifactId>
</dependency>
```

```yaml
# application.yml ของทุก service
management:
  tracing:
    sampling:
      probability: 1.0
  zipkin:
    tracing:
      endpoint: http://localhost:9411/api/v2/spans
```

```java
// การใช้ custom span ใน service
package com.shophub.order.service;

import io.micrometer.tracing.Tracer;
import io.micrometer.tracing.Span;

@Service
@RequiredArgsConstructor
public class OrderService {
    private final Tracer tracer;

    public Order createOrder(CreateOrderRequest request, Long userId) {
        Span span = tracer.nextSpan().name("create-order").start();
        try (Tracer.SpanInScope scope = tracer.withSpan(span)) {
            span.tag("userId", String.valueOf(userId));
            span.tag("itemCount", String.valueOf(request.getItems().size()));
            
            // business logic...
            
            span.event("order-saved");
            return savedOrder;
        } catch (Exception e) {
            span.error(e);
            throw e;
        } finally {
            span.end();
        }
    }
}
```

## ขั้นตอนที่ 3254: Global Exception Handler

```java
// shophub-common/src/main/java/com/shophub/common/exception/GlobalExceptionHandler.java
package com.shophub.common.exception;

import com.shophub.common.dto.ApiResponse;
import lombok.extern.slf4j.Slf4j;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.validation.FieldError;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;

import java.util.HashMap;
import java.util.Map;

@RestControllerAdvice
@Slf4j
public class GlobalExceptionHandler {

    @ExceptionHandler(ShopHubException.class)
    public ResponseEntity<ApiResponse<Void>> handleShopHubException(ShopHubException ex) {
        log.error("ShopHub exception: {} - {}", ex.getErrorCode(), ex.getMessage());
        return ResponseEntity
                .status(ex.getHttpStatus())
                .body(ApiResponse.error(ex.getMessage(), ex.getErrorCode()));
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ApiResponse<Map<String, String>>> handleValidationException(
            MethodArgumentNotValidException ex) {
        Map<String, String> errors = new HashMap<>();
        ex.getBindingResult().getAllErrors().forEach(error -> {
            String fieldName = ((FieldError) error).getField();
            String message = error.getDefaultMessage();
            errors.put(fieldName, message);
        });
        return ResponseEntity
                .status(HttpStatus.BAD_REQUEST)
                .body(ApiResponse.<Map<String, String>>builder()
                        .success(false)
                        .message("Validation failed")
                        .data(errors)
                        .errorCode("VALIDATION_ERROR")
                        .build());
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ApiResponse<Void>> handleGenericException(Exception ex) {
        log.error("Unexpected error: ", ex);
        return ResponseEntity
                .status(HttpStatus.INTERNAL_SERVER_ERROR)
                .body(ApiResponse.error("An unexpected error occurred", "INTERNAL_ERROR"));
    }
}
```

## ขั้นตอนที่ 3255: Health Check และ Actuator Configuration

```yaml
# application.yml สำหรับ health checks
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  endpoint:
    health:
      show-details: when-authorized
      probes:
        enabled: true
  health:
    db:
      enabled: true
    kafka:
      enabled: true
    redis:
      enabled: true
```

```java
// Custom Health Indicator
package com.shophub.inventory.health;

import com.shophub.inventory.repository.InventoryRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.boot.actuate.health.Health;
import org.springframework.boot.actuate.health.HealthIndicator;
import org.springframework.stereotype.Component;

@Component
@RequiredArgsConstructor
public class InventoryHealthIndicator implements HealthIndicator {

    private final InventoryRepository inventoryRepository;

    @Override
    public Health health() {
        try {
            long count = inventoryRepository.count();
            return Health.up()
                    .withDetail("inventoryCount", count)
                    .withDetail("status", "Inventory service is operational")
                    .build();
        } catch (Exception e) {
            return Health.down()
                    .withDetail("error", e.getMessage())
                    .build();
        }
    }
}
```

## ขั้นตอนที่ 3256: Service-level Integration Test

```java
// order-service/src/test/java/com/shophub/order/integration/OrderServiceIntegrationTest.java
package com.shophub.order.integration;

import com.shophub.common.dto.ApiResponse;
import com.shophub.common.dto.ProductDto;
import com.shophub.common.dto.UserDto;
import com.shophub.order.client.InventoryClient;
import com.shophub.order.client.ProductClient;
import com.shophub.order.client.UserClient;
import com.shophub.order.domain.Order;
import com.shophub.order.dto.CreateOrderRequest;
import com.shophub.order.service.OrderService;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.mock.mockito.MockBean;
import org.springframework.kafka.test.context.EmbeddedKafka;
import org.springframework.test.context.ActiveProfiles;

import java.math.BigDecimal;
import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.*;
import static org.mockito.Mockito.when;

@SpringBootTest
@ActiveProfiles("test")
@EmbeddedKafka(partitions = 1, topics = {"order.created"})
class OrderServiceIntegrationTest {

    @Autowired
    private OrderService orderService;

    @MockBean
    private UserClient userClient;

    @MockBean
    private ProductClient productClient;

    @MockBean
    private InventoryClient inventoryClient;

    @Test
    void shouldCreateOrderSuccessfully() {
        // Arrange
        UserDto mockUser = UserDto.builder()
                .id(1L).email("user@test.com").build();
        when(userClient.getUserById(1L))
                .thenReturn(ApiResponse.success(mockUser));

        ProductDto mockProduct = ProductDto.builder()
                .id(1L).sku("SKU-001").name("Test Product")
                .price(new BigDecimal("99.99")).build();
        when(productClient.getProductsByIds(anyList()))
                .thenReturn(ApiResponse.success(List.of(mockProduct)));

        when(inventoryClient.checkAndReserveStock(any()))
                .thenReturn(ApiResponse.success(new StockCheckResponse(true, "OK")));

        CreateOrderRequest request = new CreateOrderRequest();
        request.setItems(List.of(new CreateOrderRequest.Item(1L, 2)));
        request.setShippingAddress("123 Test St");

        // Act
        Order order = orderService.createOrder(request, 1L);

        // Assert
        assertThat(order).isNotNull();
        assertThat(order.getOrderNumber()).startsWith("ORD-");
        assertThat(order.getStatus()).isEqualTo(Order.OrderStatus.PENDING);
        assertThat(order.getTotalAmount()).isEqualByComparingTo("199.98");
    }
}
```

## ขั้นตอนที่ 3257-3280: Summary และ Best Practices

### สรุป ShopHub Architecture

ShopHub Platform ที่เราสร้างมีลักษณะเด่น:

1. **Database per Service** - แต่ละ service มี database เป็นของตัวเอง ป้องกัน coupling
2. **Synchronous via Feign** - ใช้สำหรับ operations ที่ต้องการผลลัพธ์ทันที เช่น check inventory
3. **Asynchronous via Kafka** - ใช้สำหรับ events ที่ไม่ต้องการรอ เช่น notifications
4. **Circuit Breaker** - ป้องกัน cascade failures ระหว่าง services
5. **Distributed Tracing** - ติดตาม request ข้าม services ด้วย Zipkin

### Performance Tuning Tips

```yaml
# JVM tuning สำหรับ production
JAVA_OPTS: >
  -server
  -XX:+UseG1GC
  -XX:MaxGCPauseMillis=200
  -XX:+HeapDumpOnOutOfMemoryError
  -Xmx512m
  -Xms256m
  -XX:+UseContainerSupport
  -XX:MaxRAMPercentage=75.0
```

```java
// Connection Pool Tuning
@Bean
public HikariDataSource dataSource(DataSourceProperties properties) {
    HikariDataSource ds = properties.initializeDataSourceBuilder()
            .type(HikariDataSource.class)
            .build();
    ds.setMaximumPoolSize(20);
    ds.setMinimumIdle(5);
    ds.setConnectionTimeout(30000);
    ds.setIdleTimeout(600000);
    ds.setMaxLifetime(1800000);
    return ds;
}
```

---

*[← Part 90: Security Advanced](./part-90-security-advanced.md) | [Part 92: DevOps Practices →](./part-92-devops-practices.md)*

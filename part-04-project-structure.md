# Part 04: โครงสร้างโปรเจค Spring Boot แบบมืออาชีพ
## ขั้นตอนที่ 61-80

> **ระดับ:** พื้นฐาน (Beginner)  
> **เวลาเรียน:** 2-3 ชั่วโมง  
> **เป้าหมาย:** เข้าใจและออกแบบโครงสร้างโปรเจคที่ดี พร้อม Naming Conventions

---

## ขั้นตอนที่ 61: Package Structure แบบต่างๆ

### Pattern 1: Layer-based (เรียงตาม Layer)

```
com.example.myapp/
├── controller/           ← HTTP layer
│   ├── UserController.java
│   ├── ProductController.java
│   └── OrderController.java
├── service/              ← Business logic
│   ├── UserService.java
│   ├── ProductService.java
│   └── OrderService.java
├── repository/           ← Data access
│   ├── UserRepository.java
│   ├── ProductRepository.java
│   └── OrderRepository.java
├── entity/               ← Database entities
│   ├── User.java
│   ├── Product.java
│   └── Order.java
├── dto/                  ← Data Transfer Objects
│   ├── UserDTO.java
│   ├── ProductDTO.java
│   └── OrderDTO.java
├── mapper/               ← Object mapping
│   └── UserMapper.java
├── exception/            ← Custom exceptions
│   ├── ResourceNotFoundException.java
│   └── GlobalExceptionHandler.java
└── config/               ← Configuration
    ├── SecurityConfig.java
    └── SwaggerConfig.java
```

### Pattern 2: Feature-based (เรียงตาม Feature) - แนะนำ!

```
com.example.myapp/
├── user/
│   ├── User.java              ← Entity
│   ├── UserRepository.java    ← Repository
│   ├── UserService.java       ← Service
│   ├── UserController.java    ← Controller
│   ├── UserDTO.java           ← DTO
│   └── UserMapper.java        ← Mapper
├── product/
│   ├── Product.java
│   ├── ProductRepository.java
│   ├── ProductService.java
│   ├── ProductController.java
│   └── dto/
│       ├── CreateProductRequest.java
│       └── ProductResponse.java
├── order/
│   ├── Order.java
│   ├── OrderRepository.java
│   ├── OrderService.java
│   └── OrderController.java
├── shared/                    ← Shared components
│   ├── exception/
│   │   ├── AppException.java
│   │   └── GlobalExceptionHandler.java
│   ├── dto/
│   │   └── ApiResponse.java
│   └── util/
│       ├── DateUtils.java
│       └── StringUtils.java
└── config/
    ├── SecurityConfig.java
    └── SwaggerConfig.java
```

### Pattern 3: Domain-Driven Design (DDD)

```
com.example.myapp/
├── domain/
│   ├── user/
│   │   ├── model/
│   │   │   ├── User.java
│   │   │   ├── UserId.java
│   │   │   └── UserEmail.java    ← Value Object
│   │   ├── repository/
│   │   │   └── UserRepository.java  ← Interface
│   │   └── service/
│   │       └── UserDomainService.java
│   └── product/
│       └── ...
├── application/                   ← Application layer (Use Cases)
│   ├── user/
│   │   ├── UserApplicationService.java
│   │   ├── command/
│   │   │   └── CreateUserCommand.java
│   │   └── query/
│   │       └── GetUserQuery.java
│   └── product/
│       └── ...
├── infrastructure/                ← Infrastructure layer
│   ├── persistence/
│   │   ├── user/
│   │   │   ├── UserEntity.java    ← JPA Entity
│   │   │   ├── UserJpaRepository.java
│   │   │   └── UserRepositoryImpl.java
│   │   └── product/
│   └── web/
│       ├── user/
│       │   └── UserController.java
│       └── product/
└── config/
```

---

## ขั้นตอนที่ 62: Naming Conventions

### Java Naming Conventions

```java
// ✅ Classes - PascalCase
public class UserService {}
public class ProductRepository {}
public class OrderController {}
public class CreateProductRequest {}

// ✅ Interfaces - PascalCase (ไม่ต้องใส่ I prefix)
public interface UserRepository {}        // ✅ ดี
public interface IUserRepository {}      // ❌ Java ไม่นิยม

// ✅ Methods - camelCase
public User findById(Long id) {}
public List<Product> findAllByCategory(String category) {}
public boolean isEmailExists(String email) {}
public void sendEmailNotification(String to) {}

// ✅ Variables - camelCase
String userName = "John";
int maxRetryCount = 3;
List<User> activeUsers = new ArrayList<>();

// ✅ Constants - UPPER_SNAKE_CASE
public static final int MAX_RETRY_COUNT = 3;
public static final String DEFAULT_TIMEZONE = "Asia/Bangkok";
public static final long JWT_EXPIRATION_MS = 86400000L;

// ✅ Packages - lowercase
com.example.myapp.controller
com.example.myapp.service
com.example.myapp.repository

// ✅ Enums - PascalCase, values เป็น UPPER_SNAKE_CASE
public enum UserRole {
    ADMIN,
    USER,
    MODERATOR
}

public enum OrderStatus {
    PENDING,
    CONFIRMED,
    SHIPPED,
    DELIVERED,
    CANCELLED
}
```

### REST API Naming Conventions

```
RESTful URL Conventions:

✅ ใช้ noun (ชื่อสิ่ง) ไม่ใช้ verb (การกระทำ)
  /api/users          ← GET (list), POST (create)
  /api/users/{id}     ← GET (one), PUT (update), DELETE (delete)
  /api/products
  /api/orders

✅ ใช้ plural nouns
  /api/users          ✅
  /api/user           ❌

✅ ใช้ lowercase และ hyphens สำหรับ multi-word
  /api/user-profiles  ✅
  /api/userProfiles   ❌
  /api/user_profiles  ❌

✅ Nested resources
  /api/users/{userId}/orders         ← orders ของ user
  /api/users/{userId}/orders/{id}    ← specific order ของ user

✅ Query parameters สำหรับ filtering, sorting, pagination
  /api/products?category=laptop&sort=price&order=asc&page=1&size=10

❌ ห้ามใช้ verbs ใน URL
  /api/getUsers       ❌
  /api/createUser     ❌
  /api/deleteUser/1   ❌
  /api/users          ✅ (GET = get, POST = create)
  /api/users/1        ✅ (DELETE = delete)
```

---

## ขั้นตอนที่ 63: Entity Design Best Practices

```java
// src/main/java/com/example/myapp/entity/BaseEntity.java
package com.example.myapp.entity;

import jakarta.persistence.*;
import lombok.Getter;
import lombok.Setter;
import org.springframework.data.annotation.CreatedBy;
import org.springframework.data.annotation.CreatedDate;
import org.springframework.data.annotation.LastModifiedBy;
import org.springframework.data.annotation.LastModifiedDate;
import org.springframework.data.jpa.domain.support.AuditingEntityListener;

import java.time.LocalDateTime;

@Getter
@Setter
@MappedSuperclass  // ← ไม่สร้าง table แยก แต่ fields นี้จะอยู่ใน child tables
@EntityListeners(AuditingEntityListener.class)  // ← Auto audit
public abstract class BaseEntity {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @CreatedDate
    @Column(name = "created_at", nullable = false, updatable = false)
    private LocalDateTime createdAt;
    
    @LastModifiedDate
    @Column(name = "updated_at")
    private LocalDateTime updatedAt;
    
    @CreatedBy
    @Column(name = "created_by", updatable = false)
    private String createdBy;
    
    @LastModifiedBy
    @Column(name = "updated_by")
    private String updatedBy;
    
    @Column(name = "is_deleted")
    private boolean deleted = false;  // Soft delete
}
```

```java
// src/main/java/com/example/myapp/entity/User.java
package com.example.myapp.entity;

import jakarta.persistence.*;
import lombok.*;

@Entity
@Table(name = "users",
    indexes = {
        @Index(name = "idx_users_email", columnList = "email", unique = true),
        @Index(name = "idx_users_username", columnList = "username")
    }
)
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
@ToString(exclude = {"password", "orders"})  // ไม่ print sensitive fields
@EqualsAndHashCode(of = "id", callSuper = false)  // เปรียบเทียบด้วย id เท่านั้น
public class User extends BaseEntity {
    
    @Column(name = "username", nullable = false, unique = true, length = 50)
    private String username;
    
    @Column(name = "email", nullable = false, unique = true, length = 100)
    private String email;
    
    @Column(name = "password", nullable = false)
    private String password;  // Store hashed password
    
    @Column(name = "first_name", length = 50)
    private String firstName;
    
    @Column(name = "last_name", length = 50)
    private String lastName;
    
    @Column(name = "phone", length = 20)
    private String phone;
    
    @Enumerated(EnumType.STRING)  // Store enum as String ไม่ใช่ ordinal
    @Column(name = "role", nullable = false, length = 20)
    @Builder.Default
    private UserRole role = UserRole.USER;
    
    @Column(name = "is_active")
    @Builder.Default
    private boolean active = true;
    
    // Derived field - ไม่ store ใน database
    @Transient
    public String getFullName() {
        return firstName + " " + lastName;
    }
}
```

---

## ขั้นตอนที่ 64: DTO Design Best Practices

```java
// ใช้ record สำหรับ immutable DTO (Java 16+)
// src/main/java/com/example/myapp/dto/request/CreateUserRequest.java
package com.example.myapp.dto.request;

import jakarta.validation.constraints.*;

public record CreateUserRequest(
    
    @NotBlank(message = "Username is required")
    @Size(min = 3, max = 50, message = "Username must be 3-50 characters")
    @Pattern(regexp = "^[a-zA-Z0-9_]+$", message = "Username can only contain letters, numbers and underscore")
    String username,
    
    @NotBlank(message = "Email is required")
    @Email(message = "Invalid email format")
    String email,
    
    @NotBlank(message = "Password is required")
    @Size(min = 8, max = 100, message = "Password must be at least 8 characters")
    @Pattern(regexp = "^(?=.*[a-z])(?=.*[A-Z])(?=.*\\d).*$",
             message = "Password must contain uppercase, lowercase and number")
    String password,
    
    @Size(max = 50, message = "First name must not exceed 50 characters")
    String firstName,
    
    @Size(max = 50, message = "Last name must not exceed 50 characters")
    String lastName,
    
    @Pattern(regexp = "^[0-9+\\-() ]*$", message = "Invalid phone format")
    String phone
) {}
```

```java
// Response DTO - อาจใช้ class ถ้าต้องการ flexibility มากกว่า
// src/main/java/com/example/myapp/dto/response/UserResponse.java
package com.example.myapp.dto.response;

import com.fasterxml.jackson.annotation.JsonInclude;
import com.fasterxml.jackson.annotation.JsonProperty;

import java.time.LocalDateTime;

@JsonInclude(JsonInclude.Include.NON_NULL)
public record UserResponse(
    Long id,
    String username,
    String email,
    
    @JsonProperty("first_name")
    String firstName,
    
    @JsonProperty("last_name")
    String lastName,
    
    String phone,
    String role,
    boolean active,
    
    @JsonProperty("created_at")
    LocalDateTime createdAt
) {}
```

---

## ขั้นตอนที่ 65: Repository Pattern

```java
// src/main/java/com/example/myapp/repository/UserRepository.java
package com.example.myapp.repository;

import com.example.myapp.entity.User;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.JpaSpecificationExecutor;
import org.springframework.data.jpa.repository.Modifying;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;
import org.springframework.stereotype.Repository;

import java.time.LocalDateTime;
import java.util.List;
import java.util.Optional;

@Repository
public interface UserRepository extends 
    JpaRepository<User, Long>,         // CRUD operations
    JpaSpecificationExecutor<User> {    // Dynamic queries
    
    // Spring Data JPA - Query by method name
    Optional<User> findByEmail(String email);
    Optional<User> findByUsername(String username);
    boolean existsByEmail(String email);
    boolean existsByUsername(String username);
    
    // Find all active users
    List<User> findByActiveTrue();
    List<User> findByActiveFalse();
    
    // Find by role
    List<User> findByRole(UserRole role);
    
    // Find with pagination
    Page<User> findByActive(boolean active, Pageable pageable);
    
    // Custom JPQL query
    @Query("SELECT u FROM User u WHERE u.email = :email AND u.active = true")
    Optional<User> findActiveUserByEmail(@Param("email") String email);
    
    // Native SQL query
    @Query(value = "SELECT * FROM users WHERE created_at >= :since", 
           nativeQuery = true)
    List<User> findUsersCreatedSince(@Param("since") LocalDateTime since);
    
    // Modifying query (UPDATE/DELETE)
    @Modifying
    @Query("UPDATE User u SET u.active = false WHERE u.id = :id")
    int deactivateUser(@Param("id") Long id);
    
    // Count queries
    long countByRole(UserRole role);
    long countByActiveTrue();
    
    // Projection (ดึงเฉพาะบาง fields)
    @Query("SELECT u.id as id, u.username as username, u.email as email FROM User u")
    List<UserSummary> findUserSummaries();
    
    // Interface-based projection
    interface UserSummary {
        Long getId();
        String getUsername();
        String getEmail();
    }
}
```

---

## ขั้นตอนที่ 66: Service Layer Pattern

```java
// src/main/java/com/example/myapp/service/UserService.java
package com.example.myapp.service;

import com.example.myapp.dto.request.CreateUserRequest;
import com.example.myapp.dto.response.UserResponse;
import com.example.myapp.entity.User;
import com.example.myapp.exception.DuplicateResourceException;
import com.example.myapp.exception.ResourceNotFoundException;
import com.example.myapp.mapper.UserMapper;
import com.example.myapp.repository.UserRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;

@Service
@RequiredArgsConstructor
@Slf4j
@Transactional(readOnly = true)  // Default: read-only transactions
public class UserService {
    
    private final UserRepository userRepository;
    private final UserMapper userMapper;
    private final PasswordEncoder passwordEncoder;
    
    // GET - read-only (ใช้ default readOnly = true)
    public Page<UserResponse> findAll(Pageable pageable) {
        log.debug("Finding all users with pageable: {}", pageable);
        return userRepository.findAll(pageable)
            .map(userMapper::toResponse);
    }
    
    public UserResponse findById(Long id) {
        log.debug("Finding user with id: {}", id);
        User user = userRepository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("User", "id", id));
        return userMapper.toResponse(user);
    }
    
    public UserResponse findByEmail(String email) {
        User user = userRepository.findByEmail(email)
            .orElseThrow(() -> new ResourceNotFoundException("User", "email", email));
        return userMapper.toResponse(user);
    }
    
    // POST - write operation (override readOnly = false)
    @Transactional
    public UserResponse create(CreateUserRequest request) {
        log.info("Creating user with email: {}", request.email());
        
        // ตรวจสอบ email ซ้ำ
        if (userRepository.existsByEmail(request.email())) {
            throw new DuplicateResourceException("User", "email", request.email());
        }
        
        // ตรวจสอบ username ซ้ำ
        if (userRepository.existsByUsername(request.username())) {
            throw new DuplicateResourceException("User", "username", request.username());
        }
        
        // Encode password
        User user = userMapper.toEntity(request);
        user.setPassword(passwordEncoder.encode(request.password()));
        
        User saved = userRepository.save(user);
        log.info("User created successfully with id: {}", saved.getId());
        
        return userMapper.toResponse(saved);
    }
    
    // PUT - write operation
    @Transactional
    public UserResponse update(Long id, UpdateUserRequest request) {
        User user = userRepository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("User", "id", id));
        
        userMapper.updateFromRequest(request, user);
        User updated = userRepository.save(user);
        
        return userMapper.toResponse(updated);
    }
    
    // DELETE - soft delete
    @Transactional
    public void delete(Long id) {
        User user = userRepository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("User", "id", id));
        
        user.setDeleted(true);
        user.setActive(false);
        userRepository.save(user);
        
        log.info("User soft-deleted with id: {}", id);
    }
}
```

---

## ขั้นตอนที่ 67: Custom Exceptions

```java
// src/main/java/com/example/myapp/exception/AppException.java
package com.example.myapp.exception;

import lombok.Getter;
import org.springframework.http.HttpStatus;

@Getter
public class AppException extends RuntimeException {
    
    private final HttpStatus status;
    private final String errorCode;
    
    public AppException(HttpStatus status, String errorCode, String message) {
        super(message);
        this.status = status;
        this.errorCode = errorCode;
    }
    
    public AppException(HttpStatus status, String errorCode, String message, Throwable cause) {
        super(message, cause);
        this.status = status;
        this.errorCode = errorCode;
    }
}
```

```java
// src/main/java/com/example/myapp/exception/ResourceNotFoundException.java
package com.example.myapp.exception;

import org.springframework.http.HttpStatus;

public class ResourceNotFoundException extends AppException {
    
    public ResourceNotFoundException(String resource, String field, Object value) {
        super(
            HttpStatus.NOT_FOUND,
            "RESOURCE_NOT_FOUND",
            String.format("%s not found with %s: %s", resource, field, value)
        );
    }
}
```

```java
// src/main/java/com/example/myapp/exception/DuplicateResourceException.java
package com.example.myapp.exception;

import org.springframework.http.HttpStatus;

public class DuplicateResourceException extends AppException {
    
    public DuplicateResourceException(String resource, String field, Object value) {
        super(
            HttpStatus.CONFLICT,
            "DUPLICATE_RESOURCE",
            String.format("%s already exists with %s: %s", resource, field, value)
        );
    }
}
```

```java
// src/main/java/com/example/myapp/exception/GlobalExceptionHandler.java
package com.example.myapp.exception;

import com.example.myapp.dto.ApiResponse;
import lombok.extern.slf4j.Slf4j;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.validation.FieldError;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;
import org.springframework.web.context.request.WebRequest;

import java.time.LocalDateTime;
import java.util.HashMap;
import java.util.Map;

@RestControllerAdvice
@Slf4j
public class GlobalExceptionHandler {
    
    @ExceptionHandler(AppException.class)
    public ResponseEntity<ApiResponse<Void>> handleAppException(
        AppException ex, WebRequest request
    ) {
        log.warn("Application exception: {}", ex.getMessage());
        return ResponseEntity
            .status(ex.getStatus())
            .body(ApiResponse.error(ex.getMessage()));
    }
    
    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ApiResponse<Void>> handleNotFoundException(
        ResourceNotFoundException ex
    ) {
        log.warn("Resource not found: {}", ex.getMessage());
        return ResponseEntity
            .status(HttpStatus.NOT_FOUND)
            .body(ApiResponse.error(ex.getMessage()));
    }
    
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ApiResponse<Map<String, String>>> handleValidationException(
        MethodArgumentNotValidException ex
    ) {
        Map<String, String> errors = new HashMap<>();
        ex.getBindingResult().getAllErrors().forEach(error -> {
            String field = ((FieldError) error).getField();
            String message = error.getDefaultMessage();
            errors.put(field, message);
        });
        
        return ResponseEntity
            .status(HttpStatus.BAD_REQUEST)
            .body(ApiResponse.error("Validation failed", errors));
    }
    
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ApiResponse<Void>> handleGenericException(
        Exception ex, WebRequest request
    ) {
        log.error("Unexpected error: {}", ex.getMessage(), ex);
        return ResponseEntity
            .status(HttpStatus.INTERNAL_SERVER_ERROR)
            .body(ApiResponse.error("Internal server error"));
    }
}
```

---

## ขั้นตอนที่ 68: Configuration Classes

```java
// src/main/java/com/example/myapp/config/AppConfig.java
package com.example.myapp.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.web.client.RestTemplate;

import java.time.Clock;

@Configuration
public class AppConfig {
    
    // Password encoder bean
    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder(12);  // strength 12 (แนะนำ)
    }
    
    // HTTP client
    @Bean
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }
    
    // Clock สำหรับ testability
    @Bean
    public Clock clock() {
        return Clock.systemDefaultZone();
    }
}
```

---

## ขั้นตอนที่ 69: Application Properties Organization

```yaml
# application.yml (หลัก - ค่า default)
spring:
  application:
    name: myapp
  profiles:
    active: ${SPRING_PROFILES_ACTIVE:dev}  # default = dev

---
# application-dev.yml
spring:
  config:
    activate:
      on-profile: dev
  
  datasource:
    url: jdbc:postgresql://localhost:5432/myapp_dev
    username: dev_user
    password: dev_password
  
  jpa:
    show-sql: true
    hibernate:
      ddl-auto: update
  
  devtools:
    restart:
      enabled: true

logging:
  level:
    com.example: DEBUG
    org.springframework.web: DEBUG
    org.hibernate.SQL: DEBUG

---
# application-test.yml
spring:
  config:
    activate:
      on-profile: test
  
  datasource:
    url: jdbc:h2:mem:testdb;DB_CLOSE_DELAY=-1
    username: sa
    password: ""
    driver-class-name: org.h2.Driver
  
  jpa:
    hibernate:
      ddl-auto: create-drop
    database-platform: org.hibernate.dialect.H2Dialect

---
# application-prod.yml
spring:
  config:
    activate:
      on-profile: prod
  
  datasource:
    url: ${DATABASE_URL}
    username: ${DATABASE_USERNAME}
    password: ${DATABASE_PASSWORD}
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
  
  jpa:
    show-sql: false
    hibernate:
      ddl-auto: validate  # ไม่ auto update ใน production!

logging:
  level:
    root: WARN
    com.example: INFO
```

---

## ขั้นตอนที่ 70: Mapper Pattern ด้วย MapStruct

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.mapstruct</groupId>
    <artifactId>mapstruct</artifactId>
    <version>1.5.5.Final</version>
</dependency>
<dependency>
    <groupId>org.mapstruct</groupId>
    <artifactId>mapstruct-processor</artifactId>
    <version>1.5.5.Final</version>
    <scope>provided</scope>
</dependency>
```

```java
// src/main/java/com/example/myapp/mapper/UserMapper.java
package com.example.myapp.mapper;

import com.example.myapp.dto.request.CreateUserRequest;
import com.example.myapp.dto.response.UserResponse;
import com.example.myapp.entity.User;
import org.mapstruct.*;

@Mapper(componentModel = "spring",  // สร้างเป็น Spring Bean
        unmappedTargetPolicy = ReportingPolicy.IGNORE)  // ไม่ error ถ้า field ไม่ตรง
public interface UserMapper {
    
    // Entity → Response DTO
    @Mapping(target = "role", expression = "java(user.getRole().name())")
    UserResponse toResponse(User user);
    
    // Request DTO → Entity
    @Mapping(target = "id", ignore = true)        // ไม่ set id
    @Mapping(target = "active", constant = "true") // default active = true
    @Mapping(target = "password", ignore = true)  // จัดการ password แยกต่างหาก
    User toEntity(CreateUserRequest request);
    
    // Update entity จาก request (partial update)
    @BeanMapping(nullValuePropertyMappingStrategy = NullValuePropertyMappingStrategy.IGNORE)
    void updateFromRequest(UpdateUserRequest request, @MappingTarget User user);
    
    // List mapping
    List<UserResponse> toResponseList(List<User> users);
}
```

---

## ขั้นตอนที่ 71: Util Classes

```java
// src/main/java/com/example/myapp/util/DateUtils.java
package com.example.myapp.util;

import java.time.*;
import java.time.format.DateTimeFormatter;

public final class DateUtils {
    
    private static final ZoneId BANGKOK = ZoneId.of("Asia/Bangkok");
    private static final DateTimeFormatter DATE_FORMAT = 
        DateTimeFormatter.ofPattern("dd/MM/yyyy");
    private static final DateTimeFormatter DATETIME_FORMAT = 
        DateTimeFormatter.ofPattern("dd/MM/yyyy HH:mm:ss");
    
    private DateUtils() {} // Prevent instantiation
    
    public static LocalDateTime nowBangkok() {
        return LocalDateTime.now(BANGKOK);
    }
    
    public static String formatDate(LocalDate date) {
        return date != null ? date.format(DATE_FORMAT) : null;
    }
    
    public static String formatDateTime(LocalDateTime dateTime) {
        return dateTime != null ? dateTime.format(DATETIME_FORMAT) : null;
    }
    
    public static boolean isExpired(LocalDateTime dateTime) {
        return dateTime != null && dateTime.isBefore(LocalDateTime.now());
    }
}
```

```java
// src/main/java/com/example/myapp/util/StringUtils.java
package com.example.myapp.util;

import java.util.UUID;

public final class StringUtils {
    
    private StringUtils() {}
    
    public static boolean isBlank(String str) {
        return str == null || str.isBlank();
    }
    
    public static String generateRandomCode(int length) {
        String chars = "ABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789";
        StringBuilder sb = new StringBuilder();
        for (int i = 0; i < length; i++) {
            sb.append(chars.charAt((int)(Math.random() * chars.length())));
        }
        return sb.toString();
    }
    
    public static String toSlug(String text) {
        return text.toLowerCase()
            .replaceAll("[^a-z0-9\\s-]", "")
            .replaceAll("\\s+", "-")
            .replaceAll("-+", "-")
            .replaceAll("^-|-$", "");
    }
    
    public static String maskEmail(String email) {
        int atIndex = email.indexOf('@');
        if (atIndex <= 2) return email;
        
        String username = email.substring(0, atIndex);
        String domain = email.substring(atIndex);
        String masked = username.charAt(0) + "*".repeat(username.length() - 2) + username.charAt(username.length() - 1);
        
        return masked + domain;
    }
}
```

---

## ขั้นตอนที่ 72: Constants ใน Application

```java
// src/main/java/com/example/myapp/constant/AppConstants.java
package com.example.myapp.constant;

public final class AppConstants {
    
    private AppConstants() {}
    
    // Pagination
    public static final int DEFAULT_PAGE = 0;
    public static final int DEFAULT_SIZE = 10;
    public static final int MAX_PAGE_SIZE = 100;
    
    // JWT
    public static final String JWT_PREFIX = "Bearer ";
    public static final String JWT_HEADER = "Authorization";
    
    // Cache names
    public static final String CACHE_USERS = "users";
    public static final String CACHE_PRODUCTS = "products";
    
    // Roles
    public static final String ROLE_ADMIN = "ROLE_ADMIN";
    public static final String ROLE_USER = "ROLE_USER";
    
    // API Paths
    public static final String API_PREFIX = "/api/v1";
    public static final String AUTH_WHITELIST = "/api/auth/**";
}
```

---

## ขั้นตอนที่ 73: Error Response Structure

```java
// src/main/java/com/example/myapp/dto/ErrorResponse.java
package com.example.myapp.dto;

import com.fasterxml.jackson.annotation.JsonInclude;
import lombok.Builder;

import java.time.LocalDateTime;
import java.util.Map;

@Builder
@JsonInclude(JsonInclude.Include.NON_NULL)
public record ErrorResponse(
    int status,
    String error,
    String message,
    String path,
    Map<String, String> fieldErrors,
    LocalDateTime timestamp
) {
    public static ErrorResponse of(int status, String error, String message, String path) {
        return ErrorResponse.builder()
            .status(status)
            .error(error)
            .message(message)
            .path(path)
            .timestamp(LocalDateTime.now())
            .build();
    }
}
```

---

## ขั้นตอนที่ 74: Audit Configuration

```java
// src/main/java/com/example/myapp/config/AuditConfig.java
package com.example.myapp.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.domain.AuditorAware;
import org.springframework.data.jpa.repository.config.EnableJpaAuditing;
import org.springframework.security.core.Authentication;
import org.springframework.security.core.context.SecurityContextHolder;

import java.util.Optional;

@Configuration
@EnableJpaAuditing(auditorAwareRef = "auditorProvider")  // เปิด JPA Auditing
public class AuditConfig {
    
    @Bean
    public AuditorAware<String> auditorProvider() {
        return () -> {
            Authentication auth = SecurityContextHolder.getContext().getAuthentication();
            
            if (auth == null || !auth.isAuthenticated() || 
                "anonymousUser".equals(auth.getPrincipal())) {
                return Optional.of("SYSTEM");
            }
            
            return Optional.of(auth.getName());
        };
    }
}
```

---

## ขั้นตอนที่ 75: Best Practices Summary

### Do's ✅

```java
// 1. Constructor Injection (ไม่ใช่ Field Injection)
@Service
@RequiredArgsConstructor
public class UserService {
    private final UserRepository repo;  // final + constructor injection
}

// 2. Use @Transactional(readOnly = true) สำหรับ read operations
@Transactional(readOnly = true)
public UserResponse findById(Long id) { ... }

// 3. Return DTO ไม่ใช่ Entity โดยตรง
public UserResponse getUser(Long id) { ... }  // ✅
public User getUser(Long id) { ... }           // ❌ expose entity

// 4. Custom exceptions ที่มีความหมาย
throw new ResourceNotFoundException("User", "id", id);

// 5. Validate input ที่ Controller layer
@PostMapping
public ResponseEntity<?> create(@Valid @RequestBody CreateRequest req) { ... }

// 6. Log สิ่งสำคัญ
log.info("Creating user: {}", email);  // ✅
log.info("Creating user: " + email);   // ❌ String concatenation ใน log
```

### Don'ts ❌

```java
// ❌ อย่าใช้ Field Injection
@Autowired
private UserRepository repo;

// ❌ อย่า return Entity โดยตรงจาก Controller
public User getUser(Long id) { return user; }

// ❌ อย่า hardcode ค่าต่างๆ
String url = "http://localhost:8080";  // ใช้ @Value แทน

// ❌ อย่า catch Exception แล้ว swallow มัน
try {
    doSomething();
} catch (Exception e) {
    // Do nothing - นี่แย่มาก!
}

// ❌ อย่าใช้ System.out.println()
System.out.println("User created");  // ใช้ log.info แทน

// ❌ อย่า put business logic ใน Controller
@PostMapping
public ResponseEntity<?> create(@RequestBody User user) {
    // ❌ business logic ไม่ควรอยู่ที่นี่
    if (userRepo.existsByEmail(user.getEmail())) {
        // duplicate check
    }
}
```

---

## ขั้นตอนที่ 76: โครงสร้างโปรเจคสุดท้าย

```
myapp/
├── src/
│   ├── main/
│   │   ├── java/com/example/myapp/
│   │   │   ├── MyappApplication.java
│   │   │   ├── config/
│   │   │   │   ├── AppConfig.java
│   │   │   │   ├── AuditConfig.java
│   │   │   │   ├── JacksonConfig.java
│   │   │   │   ├── SecurityConfig.java
│   │   │   │   └── WebConfig.java
│   │   │   ├── constant/
│   │   │   │   └── AppConstants.java
│   │   │   ├── dto/
│   │   │   │   ├── ApiResponse.java
│   │   │   │   ├── ErrorResponse.java
│   │   │   │   ├── request/
│   │   │   │   └── response/
│   │   │   ├── entity/
│   │   │   │   ├── BaseEntity.java
│   │   │   │   └── User.java
│   │   │   ├── exception/
│   │   │   │   ├── AppException.java
│   │   │   │   ├── DuplicateResourceException.java
│   │   │   │   ├── GlobalExceptionHandler.java
│   │   │   │   └── ResourceNotFoundException.java
│   │   │   ├── mapper/
│   │   │   │   └── UserMapper.java
│   │   │   ├── repository/
│   │   │   │   └── UserRepository.java
│   │   │   ├── service/
│   │   │   │   └── UserService.java
│   │   │   ├── controller/
│   │   │   │   └── UserController.java
│   │   │   └── util/
│   │   │       ├── DateUtils.java
│   │   │       └── StringUtils.java
│   │   └── resources/
│   │       ├── application.yml
│   │       ├── application-dev.yml
│   │       ├── application-test.yml
│   │       └── application-prod.yml
│   └── test/
│       └── java/com/example/myapp/
│           ├── controller/
│           │   └── UserControllerTest.java
│           ├── service/
│           │   └── UserServiceTest.java
│           └── repository/
│               └── UserRepositoryTest.java
├── .gitignore
├── docker-compose.yml
├── Dockerfile
├── mvnw
└── pom.xml
```

---

## ขั้นตอนที่ 77-80: สรุปและแบบฝึกหัด

### สิ่งที่เรียนรู้ใน Part 04

```
✅ Package structure แบบต่างๆ (Layer-based, Feature-based, DDD)
✅ Naming conventions สำหรับ Java และ REST API
✅ Entity design พร้อม BaseEntity
✅ DTO design ด้วย record
✅ Repository pattern
✅ Service layer pattern
✅ Custom exceptions
✅ Configuration classes
✅ Mapper ด้วย MapStruct
✅ Util classes
✅ Constants
✅ Audit configuration
✅ Best practices
```

### แบบฝึกหัด

```
1. สร้าง Package structure แบบ Feature-based สำหรับ E-commerce ที่มี:
   - Product Management
   - User Management
   - Order Management
   - Payment Integration

2. ออกแบบ Entity สำหรับ:
   - Product (id, name, price, stock, category)
   - Category (id, name, description, parentCategory)
   - Order (id, user, products, total, status, address)

3. สร้าง Custom exceptions สำหรับ:
   - ProductOutOfStockException
   - PaymentFailedException
   - OrderAlreadyCancelledException
```

---

*[← Part 03: สร้าง Application แรก](./part-03-first-application.md) | [Part 05: REST API พื้นฐาน →](./part-05-rest-api-basics.md)*

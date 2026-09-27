# Part 51: Real-World E-Commerce REST API Project
## ขั้นตอนที่ 1641-1680

**ระดับ:** Advanced  
**เวลาเรียน:** 5-6 ชั่วโมง  
**เป้าหมาย:** สร้าง E-Commerce REST API ที่สมบูรณ์ตั้งแต่ต้น พร้อม User, Product, Order, Cart, การค้นหาสินค้าด้วย Specification pattern และ Docker Compose

---

## ขั้นตอนที่ 1641: ภาพรวมโปรเจกต์ (Project Overview)

โปรเจกต์นี้จะสร้าง E-Commerce REST API ที่มีฟีเจอร์ครบครัน ประกอบด้วย:
- **User Management**: สมัครสมาชิก, เข้าสู่ระบบ, จัดการโปรไฟล์
- **Product Management**: CRUD สินค้า, ค้นหา, กรอง, จัดเรียง
- **Cart**: ตะกร้าสินค้า, เพิ่ม/ลบ/อัปเดต
- **Order**: สร้างคำสั่งซื้อจากตะกร้า, ติดตามสถานะ
- **Security**: JWT Authentication & Authorization

### โครงสร้างโปรเจกต์

```
ecommerce-api/
├── src/main/java/com/example/ecommerce/
│   ├── EcommerceApplication.java
│   ├── config/
│   │   ├── SecurityConfig.java
│   │   ├── JwtConfig.java
│   │   └── OpenApiConfig.java
│   ├── controller/
│   │   ├── AuthController.java
│   │   ├── UserController.java
│   │   ├── ProductController.java
│   │   ├── CartController.java
│   │   └── OrderController.java
│   ├── domain/
│   │   ├── entity/
│   │   │   ├── User.java
│   │   │   ├── Product.java
│   │   │   ├── Category.java
│   │   │   ├── Cart.java
│   │   │   ├── CartItem.java
│   │   │   ├── Order.java
│   │   │   └── OrderItem.java
│   │   ├── repository/
│   │   │   ├── UserRepository.java
│   │   │   ├── ProductRepository.java
│   │   │   ├── CartRepository.java
│   │   │   └── OrderRepository.java
│   │   └── enums/
│   │       ├── Role.java
│   │       └── OrderStatus.java
│   ├── dto/
│   │   ├── request/
│   │   └── response/
│   ├── service/
│   │   ├── UserService.java
│   │   ├── ProductService.java
│   │   ├── CartService.java
│   │   └── OrderService.java
│   ├── specification/
│   │   └── ProductSpecification.java
│   └── exception/
│       ├── GlobalExceptionHandler.java
│       └── BusinessException.java
├── src/main/resources/
│   ├── application.yml
│   └── db/migration/
├── docker-compose.yml
└── pom.xml
```

---

## ขั้นตอนที่ 1642: pom.xml และ Dependencies

ไฟล์ `pom.xml` ต้องมี dependencies ที่จำเป็นสำหรับโปรเจกต์ทั้งหมด:

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
    
    <groupId>com.example</groupId>
    <artifactId>ecommerce-api</artifactId>
    <version>1.0.0</version>
    <packaging>jar</packaging>
    
    <properties>
        <java.version>21</java.version>
        <jjwt.version>0.12.3</jjwt.version>
        <mapstruct.version>1.5.5.Final</mapstruct.version>
    </properties>
    
    <dependencies>
        <!-- Spring Boot Starters -->
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
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-validation</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
        
        <!-- Database -->
        <dependency>
            <groupId>org.postgresql</groupId>
            <artifactId>postgresql</artifactId>
            <scope>runtime</scope>
        </dependency>
        <dependency>
            <groupId>org.flywaydb</groupId>
            <artifactId>flyway-core</artifactId>
        </dependency>
        
        <!-- JWT -->
        <dependency>
            <groupId>io.jsonwebtoken</groupId>
            <artifactId>jjwt-api</artifactId>
            <version>${jjwt.version}</version>
        </dependency>
        <dependency>
            <groupId>io.jsonwebtoken</groupId>
            <artifactId>jjwt-impl</artifactId>
            <version>${jjwt.version}</version>
            <scope>runtime</scope>
        </dependency>
        <dependency>
            <groupId>io.jsonwebtoken</groupId>
            <artifactId>jjwt-jackson</artifactId>
            <version>${jjwt.version}</version>
            <scope>runtime</scope>
        </dependency>
        
        <!-- MapStruct สำหรับ DTO mapping -->
        <dependency>
            <groupId>org.mapstruct</groupId>
            <artifactId>mapstruct</artifactId>
            <version>${mapstruct.version}</version>
        </dependency>
        
        <!-- Lombok -->
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>
        
        <!-- OpenAPI/Swagger -->
        <dependency>
            <groupId>org.springdoc</groupId>
            <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
            <version>2.3.0</version>
        </dependency>
        
        <!-- Testing -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.springframework.security</groupId>
            <artifactId>spring-security-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>
    
    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <configuration>
                    <annotationProcessorPaths>
                        <path>
                            <groupId>org.mapstruct</groupId>
                            <artifactId>mapstruct-processor</artifactId>
                            <version>${mapstruct.version}</version>
                        </path>
                        <path>
                            <groupId>org.projectlombok</groupId>
                            <artifactId>lombok</artifactId>
                        </path>
                    </annotationProcessorPaths>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```

---

## ขั้นตอนที่ 1643: application.yml

ไฟล์ configuration หลักของแอปพลิเคชัน:

```yaml
# src/main/resources/application.yml
spring:
  application:
    name: ecommerce-api
  
  datasource:
    url: jdbc:postgresql://localhost:5432/ecommerce_db
    username: ${DB_USERNAME:ecommerce_user}
    password: ${DB_PASSWORD:ecommerce_pass}
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
      connection-timeout: 30000
  
  jpa:
    hibernate:
      ddl-auto: validate
    properties:
      hibernate:
        dialect: org.hibernate.dialect.PostgreSQLDialect
        format_sql: true
        default_batch_fetch_size: 100
    show-sql: false
  
  flyway:
    enabled: true
    locations: classpath:db/migration
    baseline-on-migrate: true

server:
  port: 8080
  servlet:
    context-path: /api

jwt:
  secret: ${JWT_SECRET:mySecretKey12345678901234567890123456789012345678}
  expiration: 86400000  # 24 ชั่วโมง (มิลลิวินาที)
  refresh-expiration: 604800000  # 7 วัน

logging:
  level:
    com.example.ecommerce: DEBUG
    org.springframework.security: INFO

management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics
```

---

## ขั้นตอนที่ 1644: Domain Entities - User

Entity สำหรับผู้ใช้งานระบบ:

```java
// src/main/java/com/example/ecommerce/domain/entity/User.java
package com.example.ecommerce.domain.entity;

import com.example.ecommerce.domain.enums.Role;
import jakarta.persistence.*;
import lombok.*;
import org.springframework.data.annotation.CreatedDate;
import org.springframework.data.annotation.LastModifiedDate;
import org.springframework.data.jpa.domain.support.AuditingEntityListener;

import java.time.LocalDateTime;
import java.util.HashSet;
import java.util.Set;

@Entity
@Table(name = "users", indexes = {
    @Index(name = "idx_users_email", columnList = "email", unique = true),
    @Index(name = "idx_users_username", columnList = "username", unique = true)
})
@EntityListeners(AuditingEntityListener.class)
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class User {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(nullable = false, unique = true, length = 50)
    private String username;
    
    @Column(nullable = false, unique = true, length = 100)
    private String email;
    
    @Column(nullable = false)
    private String password;
    
    @Column(name = "first_name", length = 50)
    private String firstName;
    
    @Column(name = "last_name", length = 50)
    private String lastName;
    
    @Column(name = "phone_number", length = 20)
    private String phoneNumber;
    
    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private Role role = Role.CUSTOMER;
    
    @Column(nullable = false)
    private boolean enabled = true;
    
    @OneToOne(mappedBy = "user", cascade = CascadeType.ALL, fetch = FetchType.LAZY)
    private Cart cart;
    
    @OneToMany(mappedBy = "user", cascade = CascadeType.ALL, fetch = FetchType.LAZY)
    @Builder.Default
    private Set<Order> orders = new HashSet<>();
    
    @CreatedDate
    @Column(name = "created_at", nullable = false, updatable = false)
    private LocalDateTime createdAt;
    
    @LastModifiedDate
    @Column(name = "updated_at")
    private LocalDateTime updatedAt;
}
```

---

## ขั้นตอนที่ 1645: Domain Entities - Product และ Category

```java
// src/main/java/com/example/ecommerce/domain/entity/Category.java
package com.example.ecommerce.domain.entity;

import jakarta.persistence.*;
import lombok.*;

import java.util.HashSet;
import java.util.Set;

@Entity
@Table(name = "categories")
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class Category {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(nullable = false, unique = true, length = 100)
    private String name;
    
    @Column(length = 500)
    private String description;
    
    @Column(name = "image_url")
    private String imageUrl;
    
    // Self-referencing สำหรับ subcategory
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "parent_id")
    private Category parent;
    
    @OneToMany(mappedBy = "parent", cascade = CascadeType.ALL)
    @Builder.Default
    private Set<Category> children = new HashSet<>();
    
    @OneToMany(mappedBy = "category")
    @Builder.Default
    private Set<Product> products = new HashSet<>();
}
```

```java
// src/main/java/com/example/ecommerce/domain/entity/Product.java
package com.example.ecommerce.domain.entity;

import jakarta.persistence.*;
import lombok.*;
import org.springframework.data.annotation.CreatedDate;
import org.springframework.data.annotation.LastModifiedDate;
import org.springframework.data.jpa.domain.support.AuditingEntityListener;

import java.math.BigDecimal;
import java.time.LocalDateTime;

@Entity
@Table(name = "products", indexes = {
    @Index(name = "idx_products_sku", columnList = "sku", unique = true),
    @Index(name = "idx_products_name", columnList = "name"),
    @Index(name = "idx_products_category", columnList = "category_id")
})
@EntityListeners(AuditingEntityListener.class)
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class Product {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(nullable = false, length = 200)
    private String name;
    
    @Column(columnDefinition = "TEXT")
    private String description;
    
    @Column(nullable = false, unique = true, length = 50)
    private String sku;
    
    @Column(nullable = false, precision = 10, scale = 2)
    private BigDecimal price;
    
    @Column(name = "sale_price", precision = 10, scale = 2)
    private BigDecimal salePrice;
    
    @Column(name = "stock_quantity", nullable = false)
    private Integer stockQuantity = 0;
    
    @Column(name = "image_url")
    private String imageUrl;
    
    @Column(nullable = false)
    private boolean active = true;
    
    @Column(name = "weight_kg", precision = 5, scale = 3)
    private BigDecimal weightKg;
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "category_id", nullable = false)
    private Category category;
    
    @CreatedDate
    @Column(name = "created_at", nullable = false, updatable = false)
    private LocalDateTime createdAt;
    
    @LastModifiedDate
    @Column(name = "updated_at")
    private LocalDateTime updatedAt;
    
    // คำนวณราคาจริงที่จะแสดง (ราคาโปรโมชันหรือราคาปกติ)
    @Transient
    public BigDecimal getEffectivePrice() {
        return salePrice != null ? salePrice : price;
    }
}
```

---

## ขั้นตอนที่ 1646: Domain Entities - Cart และ CartItem

```java
// src/main/java/com/example/ecommerce/domain/entity/Cart.java
package com.example.ecommerce.domain.entity;

import jakarta.persistence.*;
import lombok.*;
import org.springframework.data.annotation.LastModifiedDate;
import org.springframework.data.jpa.domain.support.AuditingEntityListener;

import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.ArrayList;
import java.util.List;

@Entity
@Table(name = "carts")
@EntityListeners(AuditingEntityListener.class)
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class Cart {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @OneToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id", nullable = false, unique = true)
    private User user;
    
    @OneToMany(mappedBy = "cart", cascade = CascadeType.ALL, orphanRemoval = true)
    @Builder.Default
    private List<CartItem> items = new ArrayList<>();
    
    @LastModifiedDate
    @Column(name = "updated_at")
    private LocalDateTime updatedAt;
    
    // คำนวณยอดรวมทั้งหมดในตะกร้า
    @Transient
    public BigDecimal getTotalAmount() {
        return items.stream()
            .map(item -> item.getProduct().getEffectivePrice()
                .multiply(BigDecimal.valueOf(item.getQuantity())))
            .reduce(BigDecimal.ZERO, BigDecimal::add);
    }
    
    @Transient
    public int getTotalItems() {
        return items.stream()
            .mapToInt(CartItem::getQuantity)
            .sum();
    }
    
    // เพิ่มสินค้าลงตะกร้า (หรืออัปเดตจำนวน)
    public void addItem(Product product, int quantity) {
        items.stream()
            .filter(item -> item.getProduct().getId().equals(product.getId()))
            .findFirst()
            .ifPresentOrElse(
                item -> item.setQuantity(item.getQuantity() + quantity),
                () -> items.add(CartItem.builder()
                    .cart(this)
                    .product(product)
                    .quantity(quantity)
                    .build())
            );
    }
    
    // ลบสินค้าออกจากตะกร้า
    public void removeItem(Long productId) {
        items.removeIf(item -> item.getProduct().getId().equals(productId));
    }
    
    // ล้างตะกร้า
    public void clear() {
        items.clear();
    }
}
```

```java
// src/main/java/com/example/ecommerce/domain/entity/CartItem.java
package com.example.ecommerce.domain.entity;

import jakarta.persistence.*;
import lombok.*;

import java.math.BigDecimal;

@Entity
@Table(name = "cart_items", uniqueConstraints = {
    @UniqueConstraint(columnNames = {"cart_id", "product_id"})
})
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class CartItem {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "cart_id", nullable = false)
    private Cart cart;
    
    @ManyToOne(fetch = FetchType.EAGER)
    @JoinColumn(name = "product_id", nullable = false)
    private Product product;
    
    @Column(nullable = false)
    private Integer quantity;
    
    @Transient
    public BigDecimal getSubtotal() {
        return product.getEffectivePrice()
            .multiply(BigDecimal.valueOf(quantity));
    }
}
```

---

## ขั้นตอนที่ 1647: Domain Entities - Order และ OrderItem

```java
// src/main/java/com/example/ecommerce/domain/enums/OrderStatus.java
package com.example.ecommerce.domain.enums;

public enum OrderStatus {
    PENDING,        // รอการยืนยัน
    CONFIRMED,      // ยืนยันแล้ว
    PROCESSING,     // กำลังเตรียมสินค้า
    SHIPPED,        // จัดส่งแล้ว
    DELIVERED,      // ส่งถึงแล้ว
    CANCELLED,      // ยกเลิก
    REFUNDED        // คืนเงินแล้ว
}
```

```java
// src/main/java/com/example/ecommerce/domain/entity/Order.java
package com.example.ecommerce.domain.entity;

import com.example.ecommerce.domain.enums.OrderStatus;
import jakarta.persistence.*;
import lombok.*;
import org.springframework.data.annotation.CreatedDate;
import org.springframework.data.annotation.LastModifiedDate;
import org.springframework.data.jpa.domain.support.AuditingEntityListener;

import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.ArrayList;
import java.util.List;
import java.util.UUID;

@Entity
@Table(name = "orders", indexes = {
    @Index(name = "idx_orders_user", columnList = "user_id"),
    @Index(name = "idx_orders_status", columnList = "status"),
    @Index(name = "idx_orders_order_number", columnList = "order_number", unique = true)
})
@EntityListeners(AuditingEntityListener.class)
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class Order {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(name = "order_number", nullable = false, unique = true, length = 50)
    private String orderNumber;
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id", nullable = false)
    private User user;
    
    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
    @Builder.Default
    private List<OrderItem> items = new ArrayList<>();
    
    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    @Builder.Default
    private OrderStatus status = OrderStatus.PENDING;
    
    @Column(name = "total_amount", nullable = false, precision = 12, scale = 2)
    private BigDecimal totalAmount;
    
    // ที่อยู่จัดส่ง (snapshot ณ เวลาสั่งซื้อ)
    @Column(name = "shipping_address_line1", nullable = false)
    private String shippingAddressLine1;
    
    @Column(name = "shipping_address_line2")
    private String shippingAddressLine2;
    
    @Column(name = "shipping_city", nullable = false, length = 100)
    private String shippingCity;
    
    @Column(name = "shipping_postal_code", nullable = false, length = 10)
    private String shippingPostalCode;
    
    @Column(name = "shipping_country", nullable = false, length = 100)
    private String shippingCountry;
    
    @Column(name = "tracking_number", length = 100)
    private String trackingNumber;
    
    @Column(name = "notes", columnDefinition = "TEXT")
    private String notes;
    
    @CreatedDate
    @Column(name = "created_at", nullable = false, updatable = false)
    private LocalDateTime createdAt;
    
    @LastModifiedDate
    @Column(name = "updated_at")
    private LocalDateTime updatedAt;
    
    @PrePersist
    protected void generateOrderNumber() {
        if (orderNumber == null) {
            orderNumber = "ORD-" + UUID.randomUUID().toString().substring(0, 8).toUpperCase();
        }
    }
}
```

```java
// src/main/java/com/example/ecommerce/domain/entity/OrderItem.java
package com.example.ecommerce.domain.entity;

import jakarta.persistence.*;
import lombok.*;

import java.math.BigDecimal;

@Entity
@Table(name = "order_items")
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class OrderItem {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "order_id", nullable = false)
    private Order order;
    
    @ManyToOne(fetch = FetchType.EAGER)
    @JoinColumn(name = "product_id", nullable = false)
    private Product product;
    
    @Column(nullable = false)
    private Integer quantity;
    
    // บันทึกราคา ณ เวลาที่สั่งซื้อ (ราคาอาจเปลี่ยนแปลงในภายหลัง)
    @Column(name = "unit_price", nullable = false, precision = 10, scale = 2)
    private BigDecimal unitPrice;
    
    @Column(name = "product_name", nullable = false, length = 200)
    private String productName;
    
    @Column(name = "product_sku", nullable = false, length = 50)
    private String productSku;
    
    @Transient
    public BigDecimal getSubtotal() {
        return unitPrice.multiply(BigDecimal.valueOf(quantity));
    }
}
```

---

## ขั้นตอนที่ 1648: Repositories

```java
// src/main/java/com/example/ecommerce/domain/repository/UserRepository.java
package com.example.ecommerce.domain.repository;

import com.example.ecommerce.domain.entity.User;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;

import java.util.Optional;

public interface UserRepository extends JpaRepository<User, Long> {
    
    Optional<User> findByUsername(String username);
    
    Optional<User> findByEmail(String email);
    
    boolean existsByUsername(String username);
    
    boolean existsByEmail(String email);
    
    @Query("SELECT u FROM User u WHERE u.email = :email AND u.enabled = true")
    Optional<User> findActiveByEmail(String email);
}
```

```java
// src/main/java/com/example/ecommerce/domain/repository/ProductRepository.java
package com.example.ecommerce.domain.repository;

import com.example.ecommerce.domain.entity.Product;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.JpaSpecificationExecutor;
import org.springframework.data.jpa.repository.Modifying;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;

import java.util.List;
import java.util.Optional;

public interface ProductRepository extends JpaRepository<Product, Long>, 
                                           JpaSpecificationExecutor<Product> {
    
    Optional<Product> findBySku(String sku);
    
    Page<Product> findByActiveTrue(Pageable pageable);
    
    Page<Product> findByCategoryId(Long categoryId, Pageable pageable);
    
    @Query("SELECT p FROM Product p WHERE p.active = true AND p.stockQuantity > 0 " +
           "AND LOWER(p.name) LIKE LOWER(CONCAT('%', :keyword, '%'))")
    Page<Product> searchByKeyword(@Param("keyword") String keyword, Pageable pageable);
    
    @Modifying
    @Query("UPDATE Product p SET p.stockQuantity = p.stockQuantity - :quantity " +
           "WHERE p.id = :productId AND p.stockQuantity >= :quantity")
    int decrementStock(@Param("productId") Long productId, 
                       @Param("quantity") int quantity);
    
    @Query("SELECT p FROM Product p WHERE p.stockQuantity <= :threshold AND p.active = true")
    List<Product> findLowStockProducts(@Param("threshold") int threshold);
}
```

```java
// src/main/java/com/example/ecommerce/domain/repository/CartRepository.java
package com.example.ecommerce.domain.repository;

import com.example.ecommerce.domain.entity.Cart;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;

import java.util.Optional;

public interface CartRepository extends JpaRepository<Cart, Long> {
    
    Optional<Cart> findByUserId(Long userId);
    
    @Query("SELECT c FROM Cart c LEFT JOIN FETCH c.items i " +
           "LEFT JOIN FETCH i.product WHERE c.user.id = :userId")
    Optional<Cart> findByUserIdWithItems(@Param("userId") Long userId);
}
```

```java
// src/main/java/com/example/ecommerce/domain/repository/OrderRepository.java
package com.example.ecommerce.domain.repository;

import com.example.ecommerce.domain.entity.Order;
import com.example.ecommerce.domain.enums.OrderStatus;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;

import java.time.LocalDateTime;
import java.util.Optional;

public interface OrderRepository extends JpaRepository<Order, Long> {
    
    Optional<Order> findByOrderNumber(String orderNumber);
    
    Page<Order> findByUserId(Long userId, Pageable pageable);
    
    Page<Order> findByStatus(OrderStatus status, Pageable pageable);
    
    @Query("SELECT o FROM Order o LEFT JOIN FETCH o.items i " +
           "LEFT JOIN FETCH i.product WHERE o.id = :orderId AND o.user.id = :userId")
    Optional<Order> findByIdAndUserIdWithItems(@Param("orderId") Long orderId,
                                               @Param("userId") Long userId);
    
    @Query("SELECT COUNT(o) FROM Order o WHERE o.createdAt >= :since")
    long countOrdersSince(@Param("since") LocalDateTime since);
}
```

---

## ขั้นตอนที่ 1649: Product Specification Pattern

Specification Pattern ช่วยให้สร้าง query เงื่อนไขซับซ้อนได้อย่างยืดหยุ่น:

```java
// src/main/java/com/example/ecommerce/specification/ProductSpecification.java
package com.example.ecommerce.specification;

import com.example.ecommerce.domain.entity.Product;
import jakarta.persistence.criteria.Predicate;
import org.springframework.data.jpa.domain.Specification;

import java.math.BigDecimal;
import java.util.ArrayList;
import java.util.List;

public class ProductSpecification {
    
    private ProductSpecification() {}
    
    // สร้าง Specification สำหรับค้นหาสินค้าตาม keyword
    public static Specification<Product> hasKeyword(String keyword) {
        return (root, query, cb) -> {
            if (keyword == null || keyword.isEmpty()) {
                return cb.conjunction();
            }
            String pattern = "%" + keyword.toLowerCase() + "%";
            return cb.or(
                cb.like(cb.lower(root.get("name")), pattern),
                cb.like(cb.lower(root.get("description")), pattern),
                cb.like(cb.lower(root.get("sku")), pattern)
            );
        };
    }
    
    // กรองตาม category
    public static Specification<Product> inCategory(Long categoryId) {
        return (root, query, cb) -> {
            if (categoryId == null) {
                return cb.conjunction();
            }
            return cb.equal(root.get("category").get("id"), categoryId);
        };
    }
    
    // กรองตามช่วงราคา
    public static Specification<Product> priceBetween(BigDecimal minPrice, BigDecimal maxPrice) {
        return (root, query, cb) -> {
            List<Predicate> predicates = new ArrayList<>();
            if (minPrice != null) {
                predicates.add(cb.greaterThanOrEqualTo(root.get("price"), minPrice));
            }
            if (maxPrice != null) {
                predicates.add(cb.lessThanOrEqualTo(root.get("price"), maxPrice));
            }
            return cb.and(predicates.toArray(new Predicate[0]));
        };
    }
    
    // เฉพาะสินค้าที่มีในสต็อก
    public static Specification<Product> inStock() {
        return (root, query, cb) ->
            cb.greaterThan(root.get("stockQuantity"), 0);
    }
    
    // เฉพาะสินค้า active
    public static Specification<Product> isActive() {
        return (root, query, cb) ->
            cb.isTrue(root.get("active"));
    }
    
    // กรองตามสถานะโปรโมชัน (มี salePrice)
    public static Specification<Product> onSale() {
        return (root, query, cb) ->
            cb.isNotNull(root.get("salePrice"));
    }
    
    // รวม Specification ทั้งหมดตาม filter criteria
    public static Specification<Product> buildFilter(ProductFilterCriteria criteria) {
        return Specification.where(isActive())
            .and(hasKeyword(criteria.getKeyword()))
            .and(inCategory(criteria.getCategoryId()))
            .and(priceBetween(criteria.getMinPrice(), criteria.getMaxPrice()))
            .and(criteria.isInStockOnly() ? inStock() : null)
            .and(criteria.isOnSaleOnly() ? onSale() : null);
    }
}
```

```java
// src/main/java/com/example/ecommerce/specification/ProductFilterCriteria.java
package com.example.ecommerce.specification;

import lombok.*;

import java.math.BigDecimal;

@Getter
@Setter
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class ProductFilterCriteria {
    private String keyword;
    private Long categoryId;
    private BigDecimal minPrice;
    private BigDecimal maxPrice;
    private boolean inStockOnly;
    private boolean onSaleOnly;
}
```

---

## ขั้นตอนที่ 1650: DTOs - Request และ Response

```java
// src/main/java/com/example/ecommerce/dto/request/RegisterRequest.java
package com.example.ecommerce.dto.request;

import jakarta.validation.constraints.*;
import lombok.Data;

@Data
public class RegisterRequest {
    
    @NotBlank(message = "ชื่อผู้ใช้ต้องไม่ว่างเปล่า")
    @Size(min = 3, max = 50, message = "ชื่อผู้ใช้ต้องมี 3-50 ตัวอักษร")
    @Pattern(regexp = "^[a-zA-Z0-9_]+$", message = "ชื่อผู้ใช้ต้องเป็นตัวอักษรและตัวเลขเท่านั้น")
    private String username;
    
    @NotBlank(message = "อีเมลต้องไม่ว่างเปล่า")
    @Email(message = "รูปแบบอีเมลไม่ถูกต้อง")
    private String email;
    
    @NotBlank(message = "รหัสผ่านต้องไม่ว่างเปล่า")
    @Size(min = 8, message = "รหัสผ่านต้องมีอย่างน้อย 8 ตัวอักษร")
    @Pattern(regexp = "^(?=.*[A-Z])(?=.*[0-9]).+$",
             message = "รหัสผ่านต้องมีตัวอักษรพิมพ์ใหญ่และตัวเลขอย่างน้อย 1 ตัว")
    private String password;
    
    @Size(max = 50)
    private String firstName;
    
    @Size(max = 50)
    private String lastName;
    
    @Pattern(regexp = "^[0-9+\\-\\s]{10,20}$", message = "เบอร์โทรศัพท์ไม่ถูกต้อง")
    private String phoneNumber;
}
```

```java
// src/main/java/com/example/ecommerce/dto/request/ProductRequest.java
package com.example.ecommerce.dto.request;

import jakarta.validation.constraints.*;
import lombok.Data;

import java.math.BigDecimal;

@Data
public class ProductRequest {
    
    @NotBlank(message = "ชื่อสินค้าต้องไม่ว่างเปล่า")
    @Size(max = 200)
    private String name;
    
    private String description;
    
    @NotBlank(message = "SKU ต้องไม่ว่างเปล่า")
    @Size(max = 50)
    private String sku;
    
    @NotNull(message = "ราคาต้องไม่ว่างเปล่า")
    @DecimalMin(value = "0.01", message = "ราคาต้องมากกว่า 0")
    @Digits(integer = 8, fraction = 2)
    private BigDecimal price;
    
    @DecimalMin(value = "0.01")
    @Digits(integer = 8, fraction = 2)
    private BigDecimal salePrice;
    
    @NotNull
    @Min(value = 0, message = "จำนวนสต็อกต้องไม่น้อยกว่า 0")
    private Integer stockQuantity;
    
    private String imageUrl;
    
    @NotNull(message = "ต้องระบุหมวดหมู่สินค้า")
    private Long categoryId;
    
    @Digits(integer = 3, fraction = 3)
    private BigDecimal weightKg;
}
```

```java
// src/main/java/com/example/ecommerce/dto/response/ProductResponse.java
package com.example.ecommerce.dto.response;

import lombok.Data;

import java.math.BigDecimal;
import java.time.LocalDateTime;

@Data
public class ProductResponse {
    private Long id;
    private String name;
    private String description;
    private String sku;
    private BigDecimal price;
    private BigDecimal salePrice;
    private BigDecimal effectivePrice;
    private Integer stockQuantity;
    private boolean inStock;
    private String imageUrl;
    private boolean active;
    private CategoryResponse category;
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
}
```

```java
// src/main/java/com/example/ecommerce/dto/response/CartResponse.java
package com.example.ecommerce.dto.response;

import lombok.Data;

import java.math.BigDecimal;
import java.util.List;

@Data
public class CartResponse {
    private Long id;
    private List<CartItemResponse> items;
    private BigDecimal totalAmount;
    private int totalItems;
    
    @Data
    public static class CartItemResponse {
        private Long id;
        private Long productId;
        private String productName;
        private String productSku;
        private String productImageUrl;
        private BigDecimal unitPrice;
        private Integer quantity;
        private BigDecimal subtotal;
    }
}
```

```java
// src/main/java/com/example/ecommerce/dto/response/OrderResponse.java
package com.example.ecommerce.dto.response;

import com.example.ecommerce.domain.enums.OrderStatus;
import lombok.Data;

import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.List;

@Data
public class OrderResponse {
    private Long id;
    private String orderNumber;
    private OrderStatus status;
    private BigDecimal totalAmount;
    private List<OrderItemResponse> items;
    private String shippingAddressLine1;
    private String shippingAddressLine2;
    private String shippingCity;
    private String shippingPostalCode;
    private String shippingCountry;
    private String trackingNumber;
    private LocalDateTime createdAt;
    
    @Data
    public static class OrderItemResponse {
        private Long id;
        private String productName;
        private String productSku;
        private Integer quantity;
        private BigDecimal unitPrice;
        private BigDecimal subtotal;
    }
}
```

---

## ขั้นตอนที่ 1651: Services - ProductService

```java
// src/main/java/com/example/ecommerce/service/ProductService.java
package com.example.ecommerce.service;

import com.example.ecommerce.domain.entity.Category;
import com.example.ecommerce.domain.entity.Product;
import com.example.ecommerce.domain.repository.ProductRepository;
import com.example.ecommerce.dto.request.ProductRequest;
import com.example.ecommerce.dto.response.ProductResponse;
import com.example.ecommerce.exception.BusinessException;
import com.example.ecommerce.specification.ProductFilterCriteria;
import com.example.ecommerce.specification.ProductSpecification;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.math.BigDecimal;

@Service
@RequiredArgsConstructor
@Slf4j
public class ProductService {
    
    private final ProductRepository productRepository;
    private final CategoryRepository categoryRepository;
    private final ProductMapper productMapper;
    
    @Transactional(readOnly = true)
    public Page<ProductResponse> searchProducts(ProductFilterCriteria criteria, Pageable pageable) {
        log.debug("Searching products with criteria: {}", criteria);
        var spec = ProductSpecification.buildFilter(criteria);
        return productRepository.findAll(spec, pageable)
            .map(productMapper::toResponse);
    }
    
    @Transactional(readOnly = true)
    public ProductResponse getProductById(Long id) {
        Product product = productRepository.findById(id)
            .orElseThrow(() -> new BusinessException("PRODUCT_NOT_FOUND", 
                "ไม่พบสินค้า ID: " + id));
        return productMapper.toResponse(product);
    }
    
    @Transactional(readOnly = true)
    public ProductResponse getProductBySku(String sku) {
        Product product = productRepository.findBySku(sku)
            .orElseThrow(() -> new BusinessException("PRODUCT_NOT_FOUND",
                "ไม่พบสินค้า SKU: " + sku));
        return productMapper.toResponse(product);
    }
    
    @Transactional
    public ProductResponse createProduct(ProductRequest request) {
        // ตรวจสอบ SKU ซ้ำ
        if (productRepository.findBySku(request.getSku()).isPresent()) {
            throw new BusinessException("SKU_ALREADY_EXISTS", 
                "SKU นี้มีอยู่แล้ว: " + request.getSku());
        }
        
        Category category = categoryRepository.findById(request.getCategoryId())
            .orElseThrow(() -> new BusinessException("CATEGORY_NOT_FOUND",
                "ไม่พบหมวดหมู่ ID: " + request.getCategoryId()));
        
        Product product = productMapper.toEntity(request);
        product.setCategory(category);
        
        Product saved = productRepository.save(product);
        log.info("Created product: {} with SKU: {}", saved.getId(), saved.getSku());
        return productMapper.toResponse(saved);
    }
    
    @Transactional
    public ProductResponse updateProduct(Long id, ProductRequest request) {
        Product product = productRepository.findById(id)
            .orElseThrow(() -> new BusinessException("PRODUCT_NOT_FOUND",
                "ไม่พบสินค้า ID: " + id));
        
        // ตรวจสอบ SKU ซ้ำ (ยกเว้นสินค้าตัวเอง)
        productRepository.findBySku(request.getSku())
            .filter(p -> !p.getId().equals(id))
            .ifPresent(p -> {
                throw new BusinessException("SKU_ALREADY_EXISTS",
                    "SKU นี้มีอยู่แล้ว: " + request.getSku());
            });
        
        Category category = categoryRepository.findById(request.getCategoryId())
            .orElseThrow(() -> new BusinessException("CATEGORY_NOT_FOUND",
                "ไม่พบหมวดหมู่ ID: " + request.getCategoryId()));
        
        productMapper.updateEntity(product, request);
        product.setCategory(category);
        
        return productMapper.toResponse(productRepository.save(product));
    }
    
    @Transactional
    public void deleteProduct(Long id) {
        Product product = productRepository.findById(id)
            .orElseThrow(() -> new BusinessException("PRODUCT_NOT_FOUND",
                "ไม่พบสินค้า ID: " + id));
        product.setActive(false);  // Soft delete
        productRepository.save(product);
        log.info("Deactivated product: {}", id);
    }
    
    @Transactional
    public void updateStock(Long id, int quantity) {
        int updated = productRepository.decrementStock(id, quantity);
        if (updated == 0) {
            throw new BusinessException("INSUFFICIENT_STOCK",
                "สินค้าไม่เพียงพอ");
        }
    }
}
```

---

## ขั้นตอนที่ 1652: Services - CartService

```java
// src/main/java/com/example/ecommerce/service/CartService.java
package com.example.ecommerce.service;

import com.example.ecommerce.domain.entity.Cart;
import com.example.ecommerce.domain.entity.Product;
import com.example.ecommerce.domain.entity.User;
import com.example.ecommerce.domain.repository.CartRepository;
import com.example.ecommerce.domain.repository.ProductRepository;
import com.example.ecommerce.dto.request.CartItemRequest;
import com.example.ecommerce.dto.response.CartResponse;
import com.example.ecommerce.exception.BusinessException;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
@RequiredArgsConstructor
@Slf4j
public class CartService {
    
    private final CartRepository cartRepository;
    private final ProductRepository productRepository;
    private final CartMapper cartMapper;
    
    @Transactional(readOnly = true)
    public CartResponse getCart(Long userId) {
        Cart cart = getOrCreateCart(userId);
        return cartMapper.toResponse(cart);
    }
    
    @Transactional
    public CartResponse addToCart(Long userId, CartItemRequest request) {
        Cart cart = getOrCreateCart(userId);
        Product product = getActiveProduct(request.getProductId());
        
        // ตรวจสอบสต็อก
        if (product.getStockQuantity() < request.getQuantity()) {
            throw new BusinessException("INSUFFICIENT_STOCK",
                String.format("สินค้า %s มีในสต็อก %d ชิ้น แต่ต้องการ %d ชิ้น",
                    product.getName(), product.getStockQuantity(), request.getQuantity()));
        }
        
        cart.addItem(product, request.getQuantity());
        Cart saved = cartRepository.save(cart);
        log.debug("Added {} x {} to cart for user {}", 
            request.getQuantity(), product.getSku(), userId);
        return cartMapper.toResponse(saved);
    }
    
    @Transactional
    public CartResponse updateCartItem(Long userId, Long productId, int quantity) {
        Cart cart = getOrCreateCart(userId);
        
        if (quantity <= 0) {
            // ถ้า quantity = 0 ให้ลบออก
            cart.removeItem(productId);
        } else {
            Product product = getActiveProduct(productId);
            if (product.getStockQuantity() < quantity) {
                throw new BusinessException("INSUFFICIENT_STOCK",
                    "สินค้าไม่เพียงพอ");
            }
            // อัปเดตจำนวน
            cart.getItems().stream()
                .filter(item -> item.getProduct().getId().equals(productId))
                .findFirst()
                .ifPresentOrElse(
                    item -> item.setQuantity(quantity),
                    () -> { throw new BusinessException("ITEM_NOT_IN_CART", 
                        "ไม่พบสินค้าในตะกร้า"); }
                );
        }
        
        return cartMapper.toResponse(cartRepository.save(cart));
    }
    
    @Transactional
    public CartResponse removeFromCart(Long userId, Long productId) {
        Cart cart = getOrCreateCart(userId);
        cart.removeItem(productId);
        return cartMapper.toResponse(cartRepository.save(cart));
    }
    
    @Transactional
    public void clearCart(Long userId) {
        Cart cart = getOrCreateCart(userId);
        cart.clear();
        cartRepository.save(cart);
        log.debug("Cleared cart for user {}", userId);
    }
    
    // ดึงตะกร้า หรือสร้างใหม่ถ้าไม่มี
    @Transactional
    public Cart getOrCreateCart(Long userId) {
        return cartRepository.findByUserIdWithItems(userId)
            .orElseGet(() -> {
                Cart newCart = Cart.builder()
                    .user(User.builder().id(userId).build())
                    .build();
                return cartRepository.save(newCart);
            });
    }
    
    private Product getActiveProduct(Long productId) {
        Product product = productRepository.findById(productId)
            .orElseThrow(() -> new BusinessException("PRODUCT_NOT_FOUND",
                "ไม่พบสินค้า ID: " + productId));
        if (!product.isActive()) {
            throw new BusinessException("PRODUCT_INACTIVE", "สินค้านี้ไม่พร้อมจำหน่าย");
        }
        return product;
    }
}
```

---

## ขั้นตอนที่ 1653: Services - OrderService

```java
// src/main/java/com/example/ecommerce/service/OrderService.java
package com.example.ecommerce.service;

import com.example.ecommerce.domain.entity.*;
import com.example.ecommerce.domain.enums.OrderStatus;
import com.example.ecommerce.domain.repository.OrderRepository;
import com.example.ecommerce.dto.request.CreateOrderRequest;
import com.example.ecommerce.dto.response.OrderResponse;
import com.example.ecommerce.exception.BusinessException;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.math.BigDecimal;
import java.util.List;
import java.util.stream.Collectors;

@Service
@RequiredArgsConstructor
@Slf4j
public class OrderService {
    
    private final OrderRepository orderRepository;
    private final CartService cartService;
    private final ProductService productService;
    private final OrderMapper orderMapper;
    
    /**
     * สร้างคำสั่งซื้อจากตะกร้าสินค้า
     * ขั้นตอน:
     * 1. ดึงสินค้าจากตะกร้า
     * 2. ตรวจสอบสต็อกสินค้าทุกรายการ
     * 3. สร้าง Order และ OrderItems
     * 4. หักสต็อกสินค้า
     * 5. ล้างตะกร้า
     */
    @Transactional
    public OrderResponse createOrder(Long userId, CreateOrderRequest request) {
        // 1. ดึงข้อมูลตะกร้า
        Cart cart = cartService.getOrCreateCart(userId);
        
        if (cart.getItems().isEmpty()) {
            throw new BusinessException("EMPTY_CART", "ตะกร้าสินค้าว่างเปล่า");
        }
        
        // 2. ตรวจสอบสต็อกทุกรายการ
        validateStock(cart.getItems());
        
        // 3. สร้าง Order
        Order order = Order.builder()
            .user(User.builder().id(userId).build())
            .status(OrderStatus.PENDING)
            .shippingAddressLine1(request.getShippingAddressLine1())
            .shippingAddressLine2(request.getShippingAddressLine2())
            .shippingCity(request.getShippingCity())
            .shippingPostalCode(request.getShippingPostalCode())
            .shippingCountry(request.getShippingCountry())
            .notes(request.getNotes())
            .build();
        
        // สร้าง OrderItems จาก CartItems
        List<OrderItem> orderItems = cart.getItems().stream()
            .map(cartItem -> OrderItem.builder()
                .order(order)
                .product(cartItem.getProduct())
                .quantity(cartItem.getQuantity())
                .unitPrice(cartItem.getProduct().getEffectivePrice())
                .productName(cartItem.getProduct().getName())
                .productSku(cartItem.getProduct().getSku())
                .build())
            .collect(Collectors.toList());
        
        order.setItems(orderItems);
        
        // คำนวณยอดรวม
        BigDecimal totalAmount = orderItems.stream()
            .map(OrderItem::getSubtotal)
            .reduce(BigDecimal.ZERO, BigDecimal::add);
        order.setTotalAmount(totalAmount);
        
        Order savedOrder = orderRepository.save(order);
        
        // 4. หักสต็อกสินค้า
        cart.getItems().forEach(cartItem -> 
            productService.updateStock(
                cartItem.getProduct().getId(), 
                cartItem.getQuantity()
            )
        );
        
        // 5. ล้างตะกร้า
        cartService.clearCart(userId);
        
        log.info("Created order {} for user {}, total: {}", 
            savedOrder.getOrderNumber(), userId, totalAmount);
        
        return orderMapper.toResponse(savedOrder);
    }
    
    @Transactional(readOnly = true)
    public Page<OrderResponse> getUserOrders(Long userId, Pageable pageable) {
        return orderRepository.findByUserId(userId, pageable)
            .map(orderMapper::toResponse);
    }
    
    @Transactional(readOnly = true)
    public OrderResponse getOrderDetails(Long userId, Long orderId) {
        return orderRepository.findByIdAndUserIdWithItems(orderId, userId)
            .map(orderMapper::toResponse)
            .orElseThrow(() -> new BusinessException("ORDER_NOT_FOUND",
                "ไม่พบคำสั่งซื้อ"));
    }
    
    @Transactional
    public OrderResponse updateOrderStatus(Long orderId, OrderStatus newStatus) {
        Order order = orderRepository.findById(orderId)
            .orElseThrow(() -> new BusinessException("ORDER_NOT_FOUND",
                "ไม่พบคำสั่งซื้อ"));
        
        validateStatusTransition(order.getStatus(), newStatus);
        order.setStatus(newStatus);
        
        log.info("Updated order {} status: {} -> {}", 
            order.getOrderNumber(), order.getStatus(), newStatus);
        
        return orderMapper.toResponse(orderRepository.save(order));
    }
    
    @Transactional
    public OrderResponse cancelOrder(Long userId, Long orderId) {
        Order order = orderRepository.findByIdAndUserIdWithItems(orderId, userId)
            .orElseThrow(() -> new BusinessException("ORDER_NOT_FOUND",
                "ไม่พบคำสั่งซื้อ"));
        
        if (!List.of(OrderStatus.PENDING, OrderStatus.CONFIRMED).contains(order.getStatus())) {
            throw new BusinessException("CANNOT_CANCEL",
                "ไม่สามารถยกเลิกคำสั่งซื้อที่มีสถานะ: " + order.getStatus());
        }
        
        order.setStatus(OrderStatus.CANCELLED);
        
        // คืนสต็อกสินค้า
        order.getItems().forEach(item ->
            productRepository.save(
                item.getProduct().toBuilder()
                    .stockQuantity(item.getProduct().getStockQuantity() + item.getQuantity())
                    .build()
            )
        );
        
        return orderMapper.toResponse(orderRepository.save(order));
    }
    
    private void validateStock(List<CartItem> items) {
        items.forEach(item -> {
            if (item.getProduct().getStockQuantity() < item.getQuantity()) {
                throw new BusinessException("INSUFFICIENT_STOCK",
                    String.format("สินค้า '%s' มีในสต็อก %d ชิ้น แต่ต้องการ %d ชิ้น",
                        item.getProduct().getName(),
                        item.getProduct().getStockQuantity(),
                        item.getQuantity()));
            }
        });
    }
    
    private void validateStatusTransition(OrderStatus current, OrderStatus next) {
        // กำหนด state machine สำหรับ order status
        boolean valid = switch (current) {
            case PENDING -> next == OrderStatus.CONFIRMED || next == OrderStatus.CANCELLED;
            case CONFIRMED -> next == OrderStatus.PROCESSING || next == OrderStatus.CANCELLED;
            case PROCESSING -> next == OrderStatus.SHIPPED;
            case SHIPPED -> next == OrderStatus.DELIVERED;
            case DELIVERED -> next == OrderStatus.REFUNDED;
            default -> false;
        };
        
        if (!valid) {
            throw new BusinessException("INVALID_STATUS_TRANSITION",
                String.format("ไม่สามารถเปลี่ยนสถานะจาก %s เป็น %s", current, next));
        }
    }
}
```

---

## ขั้นตอนที่ 1654: Controllers

```java
// src/main/java/com/example/ecommerce/controller/ProductController.java
package com.example.ecommerce.controller;

import com.example.ecommerce.dto.request.ProductRequest;
import com.example.ecommerce.dto.response.ProductResponse;
import com.example.ecommerce.service.ProductService;
import com.example.ecommerce.specification.ProductFilterCriteria;
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.security.SecurityRequirement;
import io.swagger.v3.oas.annotations.tags.Tag;
import jakarta.validation.Valid;
import lombok.RequiredArgsConstructor;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.web.PageableDefault;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.web.bind.annotation.*;

import java.math.BigDecimal;

@RestController
@RequestMapping("/v1/products")
@RequiredArgsConstructor
@Tag(name = "Products", description = "จัดการสินค้า")
public class ProductController {
    
    private final ProductService productService;
    
    @GetMapping
    @Operation(summary = "ค้นหาสินค้า", description = "ค้นหาสินค้าพร้อม filter และ pagination")
    public ResponseEntity<Page<ProductResponse>> searchProducts(
            @RequestParam(required = false) String keyword,
            @RequestParam(required = false) Long categoryId,
            @RequestParam(required = false) BigDecimal minPrice,
            @RequestParam(required = false) BigDecimal maxPrice,
            @RequestParam(defaultValue = "false") boolean inStockOnly,
            @RequestParam(defaultValue = "false") boolean onSaleOnly,
            @PageableDefault(size = 20, sort = "createdAt") Pageable pageable) {
        
        ProductFilterCriteria criteria = ProductFilterCriteria.builder()
            .keyword(keyword)
            .categoryId(categoryId)
            .minPrice(minPrice)
            .maxPrice(maxPrice)
            .inStockOnly(inStockOnly)
            .onSaleOnly(onSaleOnly)
            .build();
        
        return ResponseEntity.ok(productService.searchProducts(criteria, pageable));
    }
    
    @GetMapping("/{id}")
    public ResponseEntity<ProductResponse> getProduct(@PathVariable Long id) {
        return ResponseEntity.ok(productService.getProductById(id));
    }
    
    @PostMapping
    @PreAuthorize("hasRole('ADMIN')")
    @SecurityRequirement(name = "bearerAuth")
    @Operation(summary = "เพิ่มสินค้าใหม่")
    public ResponseEntity<ProductResponse> createProduct(
            @Valid @RequestBody ProductRequest request) {
        return ResponseEntity.status(HttpStatus.CREATED)
            .body(productService.createProduct(request));
    }
    
    @PutMapping("/{id}")
    @PreAuthorize("hasRole('ADMIN')")
    @SecurityRequirement(name = "bearerAuth")
    public ResponseEntity<ProductResponse> updateProduct(
            @PathVariable Long id,
            @Valid @RequestBody ProductRequest request) {
        return ResponseEntity.ok(productService.updateProduct(id, request));
    }
    
    @DeleteMapping("/{id}")
    @PreAuthorize("hasRole('ADMIN')")
    @SecurityRequirement(name = "bearerAuth")
    public ResponseEntity<Void> deleteProduct(@PathVariable Long id) {
        productService.deleteProduct(id);
        return ResponseEntity.noContent().build();
    }
}
```

```java
// src/main/java/com/example/ecommerce/controller/CartController.java
package com.example.ecommerce.controller;

import com.example.ecommerce.dto.request.CartItemRequest;
import com.example.ecommerce.dto.response.CartResponse;
import com.example.ecommerce.security.UserPrincipal;
import com.example.ecommerce.service.CartService;
import jakarta.validation.Valid;
import lombok.RequiredArgsConstructor;
import org.springframework.http.ResponseEntity;
import org.springframework.security.core.annotation.AuthenticationPrincipal;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/v1/cart")
@RequiredArgsConstructor
public class CartController {
    
    private final CartService cartService;
    
    @GetMapping
    public ResponseEntity<CartResponse> getCart(
            @AuthenticationPrincipal UserPrincipal principal) {
        return ResponseEntity.ok(cartService.getCart(principal.getId()));
    }
    
    @PostMapping("/items")
    public ResponseEntity<CartResponse> addToCart(
            @AuthenticationPrincipal UserPrincipal principal,
            @Valid @RequestBody CartItemRequest request) {
        return ResponseEntity.ok(cartService.addToCart(principal.getId(), request));
    }
    
    @PutMapping("/items/{productId}")
    public ResponseEntity<CartResponse> updateItem(
            @AuthenticationPrincipal UserPrincipal principal,
            @PathVariable Long productId,
            @RequestParam int quantity) {
        return ResponseEntity.ok(
            cartService.updateCartItem(principal.getId(), productId, quantity));
    }
    
    @DeleteMapping("/items/{productId}")
    public ResponseEntity<CartResponse> removeItem(
            @AuthenticationPrincipal UserPrincipal principal,
            @PathVariable Long productId) {
        return ResponseEntity.ok(cartService.removeFromCart(principal.getId(), productId));
    }
    
    @DeleteMapping
    public ResponseEntity<Void> clearCart(
            @AuthenticationPrincipal UserPrincipal principal) {
        cartService.clearCart(principal.getId());
        return ResponseEntity.noContent().build();
    }
}
```

---

## ขั้นตอนที่ 1655: Exception Handling

```java
// src/main/java/com/example/ecommerce/exception/BusinessException.java
package com.example.ecommerce.exception;

import lombok.Getter;

@Getter
public class BusinessException extends RuntimeException {
    private final String errorCode;
    
    public BusinessException(String errorCode, String message) {
        super(message);
        this.errorCode = errorCode;
    }
}
```

```java
// src/main/java/com/example/ecommerce/exception/GlobalExceptionHandler.java
package com.example.ecommerce.exception;

import lombok.extern.slf4j.Slf4j;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.security.access.AccessDeniedException;
import org.springframework.validation.FieldError;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;

import java.time.LocalDateTime;
import java.util.HashMap;
import java.util.Map;

@RestControllerAdvice
@Slf4j
public class GlobalExceptionHandler {
    
    @ExceptionHandler(BusinessException.class)
    public ResponseEntity<ErrorResponse> handleBusinessException(BusinessException ex) {
        log.warn("Business exception: {} - {}", ex.getErrorCode(), ex.getMessage());
        
        HttpStatus status = switch (ex.getErrorCode()) {
            case "PRODUCT_NOT_FOUND", "ORDER_NOT_FOUND", "CATEGORY_NOT_FOUND" 
                -> HttpStatus.NOT_FOUND;
            case "SKU_ALREADY_EXISTS", "USERNAME_ALREADY_EXISTS" 
                -> HttpStatus.CONFLICT;
            case "INSUFFICIENT_STOCK", "EMPTY_CART", "CANNOT_CANCEL",
                 "INVALID_STATUS_TRANSITION" 
                -> HttpStatus.BAD_REQUEST;
            default -> HttpStatus.INTERNAL_SERVER_ERROR;
        };
        
        return ResponseEntity.status(status).body(ErrorResponse.builder()
            .timestamp(LocalDateTime.now())
            .errorCode(ex.getErrorCode())
            .message(ex.getMessage())
            .status(status.value())
            .build());
    }
    
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidation(
            MethodArgumentNotValidException ex) {
        Map<String, String> errors = new HashMap<>();
        ex.getBindingResult().getAllErrors().forEach(error -> {
            String field = ((FieldError) error).getField();
            errors.put(field, error.getDefaultMessage());
        });
        
        return ResponseEntity.badRequest().body(ErrorResponse.builder()
            .timestamp(LocalDateTime.now())
            .errorCode("VALIDATION_ERROR")
            .message("ข้อมูลไม่ถูกต้อง")
            .status(HttpStatus.BAD_REQUEST.value())
            .validationErrors(errors)
            .build());
    }
    
    @ExceptionHandler(AccessDeniedException.class)
    public ResponseEntity<ErrorResponse> handleAccessDenied(AccessDeniedException ex) {
        return ResponseEntity.status(HttpStatus.FORBIDDEN).body(ErrorResponse.builder()
            .timestamp(LocalDateTime.now())
            .errorCode("ACCESS_DENIED")
            .message("ไม่มีสิทธิ์เข้าถึง")
            .status(HttpStatus.FORBIDDEN.value())
            .build());
    }
    
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleGeneral(Exception ex) {
        log.error("Unexpected error", ex);
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
            .body(ErrorResponse.builder()
                .timestamp(LocalDateTime.now())
                .errorCode("INTERNAL_ERROR")
                .message("เกิดข้อผิดพลาดภายในระบบ")
                .status(HttpStatus.INTERNAL_SERVER_ERROR.value())
                .build());
    }
}
```

---

## ขั้นตอนที่ 1656: Flyway Database Migrations

```sql
-- src/main/resources/db/migration/V1__initial_schema.sql
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    username VARCHAR(50) NOT NULL UNIQUE,
    email VARCHAR(100) NOT NULL UNIQUE,
    password VARCHAR(255) NOT NULL,
    first_name VARCHAR(50),
    last_name VARCHAR(50),
    phone_number VARCHAR(20),
    role VARCHAR(20) NOT NULL DEFAULT 'CUSTOMER',
    enabled BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP
);

CREATE TABLE categories (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL UNIQUE,
    description VARCHAR(500),
    image_url VARCHAR(255),
    parent_id BIGINT REFERENCES categories(id)
);

CREATE TABLE products (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    description TEXT,
    sku VARCHAR(50) NOT NULL UNIQUE,
    price DECIMAL(10,2) NOT NULL,
    sale_price DECIMAL(10,2),
    stock_quantity INT NOT NULL DEFAULT 0,
    image_url VARCHAR(255),
    active BOOLEAN NOT NULL DEFAULT TRUE,
    weight_kg DECIMAL(5,3),
    category_id BIGINT NOT NULL REFERENCES categories(id),
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP,
    CONSTRAINT chk_price_positive CHECK (price > 0),
    CONSTRAINT chk_stock_non_negative CHECK (stock_quantity >= 0)
);

CREATE TABLE carts (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL UNIQUE REFERENCES users(id),
    updated_at TIMESTAMP
);

CREATE TABLE cart_items (
    id BIGSERIAL PRIMARY KEY,
    cart_id BIGINT NOT NULL REFERENCES carts(id) ON DELETE CASCADE,
    product_id BIGINT NOT NULL REFERENCES products(id),
    quantity INT NOT NULL,
    UNIQUE(cart_id, product_id),
    CONSTRAINT chk_quantity_positive CHECK (quantity > 0)
);

CREATE TABLE orders (
    id BIGSERIAL PRIMARY KEY,
    order_number VARCHAR(50) NOT NULL UNIQUE,
    user_id BIGINT NOT NULL REFERENCES users(id),
    status VARCHAR(20) NOT NULL DEFAULT 'PENDING',
    total_amount DECIMAL(12,2) NOT NULL,
    shipping_address_line1 VARCHAR(255) NOT NULL,
    shipping_address_line2 VARCHAR(255),
    shipping_city VARCHAR(100) NOT NULL,
    shipping_postal_code VARCHAR(10) NOT NULL,
    shipping_country VARCHAR(100) NOT NULL,
    tracking_number VARCHAR(100),
    notes TEXT,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP
);

CREATE TABLE order_items (
    id BIGSERIAL PRIMARY KEY,
    order_id BIGINT NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
    product_id BIGINT NOT NULL REFERENCES products(id),
    quantity INT NOT NULL,
    unit_price DECIMAL(10,2) NOT NULL,
    product_name VARCHAR(200) NOT NULL,
    product_sku VARCHAR(50) NOT NULL
);

-- Indexes
CREATE INDEX idx_products_category ON products(category_id);
CREATE INDEX idx_products_name ON products(name);
CREATE INDEX idx_products_active ON products(active) WHERE active = TRUE;
CREATE INDEX idx_orders_user ON orders(user_id);
CREATE INDEX idx_orders_status ON orders(status);
```

```sql
-- src/main/resources/db/migration/V2__seed_data.sql
INSERT INTO categories (name, description) VALUES
    ('Electronics', 'อุปกรณ์อิเล็กทรอนิกส์'),
    ('Clothing', 'เสื้อผ้าแฟชั่น'),
    ('Books', 'หนังสือและสื่อการเรียนรู้'),
    ('Home & Garden', 'ของใช้ในบ้านและสวน'),
    ('Sports', 'อุปกรณ์กีฬา');

INSERT INTO products (name, sku, price, stock_quantity, category_id) VALUES
    ('Laptop Pro X1', 'LAPTOP-X1-001', 45000.00, 50, 1),
    ('Wireless Headphones', 'HEAD-WL-001', 3500.00, 100, 1),
    ('Spring Boot Book', 'BOOK-SB-001', 890.00, 200, 3),
    ('Running Shoes', 'SHOE-RUN-001', 2800.00, 75, 5);
```

---

## ขั้นตอนที่ 1657: Docker Compose

```yaml
# docker-compose.yml
version: '3.9'

services:
  postgres:
    image: postgres:16-alpine
    container_name: ecommerce-postgres
    environment:
      POSTGRES_DB: ecommerce_db
      POSTGRES_USER: ecommerce_user
      POSTGRES_PASSWORD: ecommerce_pass
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ecommerce_user -d ecommerce_db"]
      interval: 10s
      timeout: 5s
      retries: 5
  
  redis:
    image: redis:7-alpine
    container_name: ecommerce-redis
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
  
  app:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: ecommerce-app
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/ecommerce_db
      SPRING_DATASOURCE_USERNAME: ecommerce_user
      SPRING_DATASOURCE_PASSWORD: ecommerce_pass
      JWT_SECRET: mySecretKeyForJWTTokenGeneration12345678901234567890
    ports:
      - "8080:8080"
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/api/actuator/health"]
      interval: 30s
      timeout: 10s
      retries: 3

volumes:
  postgres_data:
  redis_data:
```

```dockerfile
# Dockerfile
FROM eclipse-temurin:21-jdk-alpine AS builder
WORKDIR /app
COPY pom.xml .
COPY src ./src
RUN ./mvnw package -DskipTests

FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
COPY --from=builder /app/target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

---

## ขั้นตอนที่ 1658: Integration Tests

```java
// src/test/java/com/example/ecommerce/integration/ProductControllerIntegrationTest.java
package com.example.ecommerce.integration;

import com.example.ecommerce.dto.request.ProductRequest;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.http.MediaType;
import org.springframework.security.test.context.support.WithMockUser;
import org.springframework.test.context.ActiveProfiles;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.transaction.annotation.Transactional;

import java.math.BigDecimal;

import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@SpringBootTest
@AutoConfigureMockMvc
@ActiveProfiles("test")
@Transactional
class ProductControllerIntegrationTest {
    
    @Autowired
    private MockMvc mockMvc;
    
    @Autowired
    private ObjectMapper objectMapper;
    
    @Test
    void shouldReturnProductListWithPagination() throws Exception {
        mockMvc.perform(get("/api/v1/products")
                .param("page", "0")
                .param("size", "10"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.content").isArray())
            .andExpect(jsonPath("$.pageable.pageSize").value(10));
    }
    
    @Test
    void shouldSearchProductsByKeyword() throws Exception {
        mockMvc.perform(get("/api/v1/products")
                .param("keyword", "laptop"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.content[*].name", 
                org.hamcrest.Matchers.everyItem(
                    org.hamcrest.Matchers.containsStringIgnoringCase("laptop"))));
    }
    
    @Test
    @WithMockUser(roles = "ADMIN")
    void shouldCreateProductAsAdmin() throws Exception {
        ProductRequest request = new ProductRequest();
        request.setName("Test Product");
        request.setSku("TEST-001");
        request.setPrice(new BigDecimal("999.99"));
        request.setStockQuantity(10);
        request.setCategoryId(1L);
        
        mockMvc.perform(post("/api/v1/products")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request)))
            .andExpect(status().isCreated())
            .andExpect(jsonPath("$.id").exists())
            .andExpect(jsonPath("$.sku").value("TEST-001"));
    }
    
    @Test
    void shouldReturn403WhenCreatingProductWithoutAuth() throws Exception {
        ProductRequest request = new ProductRequest();
        request.setName("Test");
        request.setSku("TEST-002");
        request.setPrice(new BigDecimal("100"));
        request.setStockQuantity(5);
        request.setCategoryId(1L);
        
        mockMvc.perform(post("/api/v1/products")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request)))
            .andExpect(status().isUnauthorized());
    }
}
```

---

## สรุปท้ายบท

ในส่วนนี้เราได้สร้าง E-Commerce REST API ที่สมบูรณ์ครอบคลุม:

1. **Domain Model** ที่ออกแบบมาอย่างรอบคอบ (User, Product, Cart, Order)
2. **Specification Pattern** สำหรับการค้นหาสินค้าแบบยืดหยุ่น
3. **Service Layer** พร้อม business logic และ validation
4. **Transaction management** ที่ถูกต้องสำหรับ Order workflow
5. **Docker Compose** สำหรับ run โปรเจกต์ทั้งหมด
6. **Integration Tests** ที่ครอบคลุม happy path และ error cases

---

*[← Part 50: Production Runbook](./part-50-production-runbook.md) | [Part 52: Database Migration →](./part-52-database-migration.md)*

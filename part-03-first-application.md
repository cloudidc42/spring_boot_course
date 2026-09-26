# Part 03: สร้าง Spring Boot Application แรก
## ขั้นตอนที่ 36-60

> **ระดับ:** พื้นฐาน (Beginner)  
> **เวลาเรียน:** 3-4 ชั่วโมง  
> **เป้าหมาย:** สร้าง REST API ที่ใช้งานได้จริง ตั้งแต่ต้นจนจบ

---

## ขั้นตอนที่ 36: สร้าง Project ด้วย Spring Initializr

ไปที่ https://start.spring.io และตั้งค่าดังนี้:

```
Project:     Maven
Language:    Java
Spring Boot: 3.3.4
Group:       com.example
Artifact:    hello-spring
Name:        hello-spring
Package:     com.example.hellospring
Packaging:   Jar
Java:        21

Dependencies:
✓ Spring Web
✓ Spring Boot DevTools
✓ Lombok
```

คลิก **Generate** และ extract zip ลงในโฟลเดอร์

### เปิดโปรเจคใน IntelliJ IDEA

```
File → Open → เลือกโฟลเดอร์ hello-spring
รอให้ IntelliJ download dependencies
```

---

## ขั้นตอนที่ 37: ดูโครงสร้างโปรเจค

```
hello-spring/
├── src/
│   ├── main/
│   │   ├── java/
│   │   │   └── com/example/hellospring/
│   │   │       └── HelloSpringApplication.java  ← Main class
│   │   └── resources/
│   │       ├── application.properties           ← Config
│   │       ├── static/                          ← Static files (CSS, JS)
│   │       └── templates/                       ← Thymeleaf templates
│   └── test/
│       └── java/
│           └── com/example/hellospring/
│               └── HelloSpringApplicationTests.java
├── .gitignore
├── mvnw                                         ← Maven Wrapper
├── mvnw.cmd
└── pom.xml
```

---

## ขั้นตอนที่ 38: ดู Main Class

```java
// HelloSpringApplication.java
package com.example.hellospring;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication  // ← Magic annotation!
public class HelloSpringApplication {
    
    public static void main(String[] args) {
        SpringApplication.run(HelloSpringApplication.class, args);
    }
}
```

### @SpringBootApplication คืออะไร?

```java
// @SpringBootApplication = รวม 3 annotations:

@SpringBootConfiguration  // = @Configuration
// → บอกว่า class นี้เป็น Spring Configuration class

@EnableAutoConfiguration
// → เปิดใช้ Auto-configuration
// → Spring Boot จะ config ทุกอย่างให้อัตโนมัติ

@ComponentScan(basePackages = "com.example.hellospring")
// → สแกนหา @Component, @Service, @Controller, etc.
// → ใน package เดียวกันและ sub-packages
```

---

## ขั้นตอนที่ 39: สร้าง Controller แรก

```java
// src/main/java/com/example/hellospring/controller/HelloController.java
package com.example.hellospring.controller;

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController  // ← บอกว่านี่คือ REST API controller
public class HelloController {
    
    @GetMapping("/hello")  // ← HTTP GET /hello
    public String hello() {
        return "Hello, Spring Boot! 🚀";
    }
}
```

### รัน Application

```bash
# วิธีที่ 1: Maven
mvn spring-boot:run

# วิธีที่ 2: IntelliJ
# คลิก Run button หรือ Shift+F10

# วิธีที่ 3: IDE เปิด HelloSpringApplication.java
# คลิก green arrow ข้าง main method
```

### ทดสอบ

```bash
# ใช้ curl
curl http://localhost:8080/hello

# ผลลัพธ์:
Hello, Spring Boot! 🚀

# หรือเปิด browser
# http://localhost:8080/hello
```

---

## ขั้นตอนที่ 40: เพิ่ม Endpoints ต่างๆ

```java
package com.example.hellospring.controller;

import org.springframework.web.bind.annotation.*;

import java.time.LocalDateTime;
import java.util.Map;

@RestController
@RequestMapping("/api")  // ← Prefix สำหรับทุก endpoints ใน class นี้
public class HelloController {
    
    // GET /api/hello
    @GetMapping("/hello")
    public String hello() {
        return "Hello, Spring Boot! 🚀";
    }
    
    // GET /api/time
    @GetMapping("/time")
    public Map<String, Object> currentTime() {
        return Map.of(
            "timestamp", LocalDateTime.now(),
            "timezone", "Asia/Bangkok",
            "message", "Current server time"
        );
    }
    
    // GET /api/greet/{name}
    @GetMapping("/greet/{name}")
    public Map<String, String> greet(@PathVariable String name) {
        return Map.of(
            "message", "สวัสดี " + name + "!",
            "from", "Spring Boot"
        );
    }
    
    // GET /api/sum?a=5&b=3
    @GetMapping("/sum")
    public Map<String, Integer> sum(
        @RequestParam int a,
        @RequestParam int b
    ) {
        return Map.of(
            "a", a,
            "b", b,
            "sum", a + b
        );
    }
    
    // POST /api/echo
    @PostMapping("/echo")
    public Map<String, Object> echo(@RequestBody Map<String, Object> body) {
        return Map.of(
            "received", body,
            "timestamp", LocalDateTime.now()
        );
    }
}
```

### ทดสอบ Endpoints

```bash
# GET /api/hello
curl http://localhost:8080/api/hello

# GET /api/time
curl http://localhost:8080/api/time

# GET /api/greet/{name}
curl http://localhost:8080/api/greet/สมชาย

# GET /api/sum?a=5&b=3
curl "http://localhost:8080/api/sum?a=5&b=3"

# POST /api/echo
curl -X POST http://localhost:8080/api/echo \
  -H "Content-Type: application/json" \
  -d '{"name": "John", "age": 25}'
```

---

## ขั้นตอนที่ 41: สร้าง Model (Entity/Domain Object)

```java
// src/main/java/com/example/hellospring/model/Product.java
package com.example.hellospring.model;

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;

import java.math.BigDecimal;
import java.time.LocalDateTime;

@Data                    // Getter, Setter, toString, equals, hashCode
@Builder                 // Builder pattern
@NoArgsConstructor       // No-args constructor
@AllArgsConstructor      // All-args constructor
public class Product {
    
    private Long id;
    private String name;
    private String description;
    private BigDecimal price;
    private Integer quantity;
    private String category;
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
}
```

---

## ขั้นตอนที่ 42: สร้าง Service Layer

```java
// src/main/java/com/example/hellospring/service/ProductService.java
package com.example.hellospring.service;

import com.example.hellospring.model.Product;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;

import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.*;
import java.util.concurrent.atomic.AtomicLong;

@Service  // ← บอกว่านี่คือ Spring Service Bean
@Slf4j    // ← สร้าง SLF4J logger
public class ProductService {
    
    // จำลอง Database ด้วย in-memory Map (ก่อนจะเรียน JPA)
    private final Map<Long, Product> products = new HashMap<>();
    private final AtomicLong idCounter = new AtomicLong(1);
    
    // Constructor - เพิ่มข้อมูลตัวอย่าง
    public ProductService() {
        initSampleData();
    }
    
    private void initSampleData() {
        save(Product.builder()
            .name("MacBook Pro M3")
            .description("Apple MacBook Pro 14\" with M3 chip")
            .price(new BigDecimal("69900"))
            .quantity(10)
            .category("Laptop")
            .build());
        
        save(Product.builder()
            .name("iPhone 15 Pro")
            .description("Apple iPhone 15 Pro 256GB")
            .price(new BigDecimal("44900"))
            .quantity(25)
            .category("Mobile")
            .build());
        
        save(Product.builder()
            .name("Samsung 4K TV")
            .description("Samsung 65\" 4K QLED TV")
            .price(new BigDecimal("35900"))
            .quantity(5)
            .category("TV")
            .build());
    }
    
    // ดึงสินค้าทั้งหมด
    public List<Product> findAll() {
        log.info("Finding all products, total: {}", products.size());
        return new ArrayList<>(products.values());
    }
    
    // ดึงสินค้าตาม ID
    public Optional<Product> findById(Long id) {
        log.info("Finding product with id: {}", id);
        return Optional.ofNullable(products.get(id));
    }
    
    // ค้นหาสินค้าตาม category
    public List<Product> findByCategory(String category) {
        log.info("Finding products in category: {}", category);
        return products.values().stream()
            .filter(p -> p.getCategory().equalsIgnoreCase(category))
            .toList();
    }
    
    // บันทึกสินค้า (สร้างใหม่หรือ update)
    public Product save(Product product) {
        if (product.getId() == null) {
            // สร้างใหม่
            Long id = idCounter.getAndIncrement();
            product.setId(id);
            product.setCreatedAt(LocalDateTime.now());
            product.setUpdatedAt(LocalDateTime.now());
            log.info("Creating new product: {}", product.getName());
        } else {
            // Update
            product.setUpdatedAt(LocalDateTime.now());
            log.info("Updating product id: {}", product.getId());
        }
        products.put(product.getId(), product);
        return product;
    }
    
    // ลบสินค้า
    public boolean deleteById(Long id) {
        if (products.containsKey(id)) {
            products.remove(id);
            log.info("Deleted product id: {}", id);
            return true;
        }
        log.warn("Product not found with id: {}", id);
        return false;
    }
    
    // ตรวจสอบว่ามีสินค้าหรือไม่
    public boolean existsById(Long id) {
        return products.containsKey(id);
    }
    
    // นับจำนวนสินค้า
    public long count() {
        return products.size();
    }
}
```

---

## ขั้นตอนที่ 43: สร้าง REST Controller

```java
// src/main/java/com/example/hellospring/controller/ProductController.java
package com.example.hellospring.controller;

import com.example.hellospring.model.Product;
import com.example.hellospring.service.ProductService;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.List;

@RestController
@RequestMapping("/api/products")
@RequiredArgsConstructor  // Constructor injection สำหรับ final fields
@Slf4j
public class ProductController {
    
    private final ProductService productService;  // final + @RequiredArgsConstructor
    
    // GET /api/products
    // GET /api/products?category=Laptop
    @GetMapping
    public ResponseEntity<List<Product>> getAllProducts(
        @RequestParam(required = false) String category
    ) {
        log.info("GET /api/products, category: {}", category);
        
        List<Product> products;
        if (category != null && !category.isEmpty()) {
            products = productService.findByCategory(category);
        } else {
            products = productService.findAll();
        }
        
        return ResponseEntity.ok(products);
    }
    
    // GET /api/products/{id}
    @GetMapping("/{id}")
    public ResponseEntity<Product> getProductById(@PathVariable Long id) {
        log.info("GET /api/products/{}", id);
        
        return productService.findById(id)
            .map(ResponseEntity::ok)          // 200 OK
            .orElse(ResponseEntity.notFound() // 404 Not Found
                .build());
    }
    
    // POST /api/products
    @PostMapping
    public ResponseEntity<Product> createProduct(@RequestBody Product product) {
        log.info("POST /api/products: {}", product.getName());
        
        Product saved = productService.save(product);
        return ResponseEntity
            .status(HttpStatus.CREATED)  // 201 Created
            .body(saved);
    }
    
    // PUT /api/products/{id}
    @PutMapping("/{id}")
    public ResponseEntity<Product> updateProduct(
        @PathVariable Long id,
        @RequestBody Product product
    ) {
        log.info("PUT /api/products/{}", id);
        
        if (!productService.existsById(id)) {
            return ResponseEntity.notFound().build();  // 404
        }
        
        product.setId(id);
        Product updated = productService.save(product);
        return ResponseEntity.ok(updated);  // 200
    }
    
    // DELETE /api/products/{id}
    @DeleteMapping("/{id}")
    public ResponseEntity<Void> deleteProduct(@PathVariable Long id) {
        log.info("DELETE /api/products/{}", id);
        
        boolean deleted = productService.deleteById(id);
        
        if (deleted) {
            return ResponseEntity.noContent().build();  // 204 No Content
        } else {
            return ResponseEntity.notFound().build();    // 404
        }
    }
    
    // GET /api/products/count
    @GetMapping("/count")
    public ResponseEntity<Long> countProducts() {
        return ResponseEntity.ok(productService.count());
    }
}
```

---

## ขั้นตอนที่ 44: ทดสอบ REST API

### ใช้ curl

```bash
# 1. ดูสินค้าทั้งหมด
curl -X GET http://localhost:8080/api/products \
  -H "Accept: application/json"

# 2. ดูสินค้าตาม ID
curl -X GET http://localhost:8080/api/products/1

# 3. ดูสินค้าตาม category
curl -X GET "http://localhost:8080/api/products?category=Laptop"

# 4. สร้างสินค้าใหม่
curl -X POST http://localhost:8080/api/products \
  -H "Content-Type: application/json" \
  -d '{
    "name": "AirPods Pro",
    "description": "Apple AirPods Pro 2nd Gen",
    "price": 9900,
    "quantity": 50,
    "category": "Audio"
  }'

# 5. Update สินค้า
curl -X PUT http://localhost:8080/api/products/1 \
  -H "Content-Type: application/json" \
  -d '{
    "name": "MacBook Pro M3 Pro",
    "description": "Updated description",
    "price": 79900,
    "quantity": 8,
    "category": "Laptop"
  }'

# 6. ลบสินค้า
curl -X DELETE http://localhost:8080/api/products/3

# 7. นับสินค้า
curl http://localhost:8080/api/products/count
```

### สร้างไฟล์ .http สำหรับ IntelliJ HTTP Client

```http
### Get all products
GET http://localhost:8080/api/products
Accept: application/json

### Get product by ID
GET http://localhost:8080/api/products/1

### Get products by category
GET http://localhost:8080/api/products?category=Laptop

### Create product
POST http://localhost:8080/api/products
Content-Type: application/json

{
  "name": "AirPods Pro",
  "description": "Apple AirPods Pro 2nd Gen",
  "price": 9900,
  "quantity": 50,
  "category": "Audio"
}

### Update product
PUT http://localhost:8080/api/products/1
Content-Type: application/json

{
  "name": "MacBook Pro M3 Pro",
  "description": "Updated description",
  "price": 79900,
  "quantity": 8,
  "category": "Laptop"
}

### Delete product
DELETE http://localhost:8080/api/products/3

### Count products
GET http://localhost:8080/api/products/count
```

---

## ขั้นตอนที่ 45: HTTP Response Codes

```
HTTP Status Codes ที่ใช้บ่อยใน REST API:

Success (2xx):
  200 OK          → สำเร็จ (GET, PUT, PATCH)
  201 Created     → สร้างข้อมูลสำเร็จ (POST)
  204 No Content  → สำเร็จแต่ไม่มี response body (DELETE)

Client Error (4xx):
  400 Bad Request       → Request ไม่ถูกต้อง (validation error)
  401 Unauthorized      → ไม่ได้ authenticate
  403 Forbidden         → authenticate แล้วแต่ไม่มีสิทธิ์
  404 Not Found         → ไม่พบข้อมูล
  409 Conflict          → ข้อมูลซ้ำ (email ซ้ำ)
  422 Unprocessable     → ข้อมูลถูกรูปแบบแต่ logic ผิด

Server Error (5xx):
  500 Internal Server Error → เกิด error ใน server
  503 Service Unavailable   → Server ไม่พร้อมให้บริการ
```

### ResponseEntity ใน Spring Boot

```java
@RestController
@RequestMapping("/api/demo")
public class DemoController {
    
    // 200 OK with body
    @GetMapping("/ok")
    public ResponseEntity<String> okResponse() {
        return ResponseEntity.ok("Success!");
        // หรือ
        return ResponseEntity.status(HttpStatus.OK).body("Success!");
    }
    
    // 201 Created
    @PostMapping("/create")
    public ResponseEntity<String> createResponse() {
        return ResponseEntity
            .status(HttpStatus.CREATED)
            .header("X-Custom-Header", "value")  // เพิ่ม custom header
            .body("Created!");
    }
    
    // 204 No Content
    @DeleteMapping("/delete")
    public ResponseEntity<Void> deleteResponse() {
        return ResponseEntity.noContent().build();
    }
    
    // 400 Bad Request
    @GetMapping("/bad-request")
    public ResponseEntity<String> badRequest() {
        return ResponseEntity.badRequest().body("Invalid input!");
    }
    
    // 404 Not Found
    @GetMapping("/not-found")
    public ResponseEntity<String> notFound() {
        return ResponseEntity.notFound().build();
        // ไม่มี body!
    }
    
    // Custom status code
    @GetMapping("/custom")
    public ResponseEntity<String> custom() {
        return ResponseEntity
            .status(418)  // I'm a teapot 🫖
            .body("I'm a teapot!");
    }
}
```

---

## ขั้นตอนที่ 46: เพิ่ม Request Validation พื้นฐาน

```xml
<!-- เพิ่มใน pom.xml -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-validation</artifactId>
</dependency>
```

```java
// สร้าง DTO สำหรับรับข้อมูลจาก request
package com.example.hellospring.dto;

import jakarta.validation.constraints.*;
import lombok.Data;

import java.math.BigDecimal;

@Data
public class CreateProductRequest {
    
    @NotBlank(message = "ชื่อสินค้าต้องไม่ว่าง")
    @Size(min = 2, max = 100, message = "ชื่อสินค้าต้องมี 2-100 ตัวอักษร")
    private String name;
    
    @Size(max = 500, message = "คำอธิบายต้องไม่เกิน 500 ตัวอักษร")
    private String description;
    
    @NotNull(message = "ราคาต้องไม่ว่าง")
    @DecimalMin(value = "0.01", message = "ราคาต้องมากกว่า 0")
    @DecimalMax(value = "9999999.99", message = "ราคาต้องไม่เกิน 9,999,999.99")
    private BigDecimal price;
    
    @NotNull(message = "จำนวนต้องไม่ว่าง")
    @Min(value = 0, message = "จำนวนต้องไม่น้อยกว่า 0")
    private Integer quantity;
    
    @NotBlank(message = "หมวดหมู่ต้องไม่ว่าง")
    private String category;
}
```

```java
// Update Controller
@PostMapping
public ResponseEntity<Product> createProduct(
    @Valid @RequestBody CreateProductRequest request  // ← เพิ่ม @Valid
) {
    Product product = Product.builder()
        .name(request.getName())
        .description(request.getDescription())
        .price(request.getPrice())
        .quantity(request.getQuantity())
        .category(request.getCategory())
        .build();
    
    Product saved = productService.save(product);
    return ResponseEntity.status(HttpStatus.CREATED).body(saved);
}
```

### Validation Error Handler

```java
// src/main/java/com/example/hellospring/exception/GlobalExceptionHandler.java
package com.example.hellospring.exception;

import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.validation.FieldError;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;

import java.time.LocalDateTime;
import java.util.HashMap;
import java.util.Map;

@RestControllerAdvice  // ← Handle exceptions ทุก Controller
public class GlobalExceptionHandler {
    
    // Handle Validation errors
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<Map<String, Object>> handleValidationErrors(
        MethodArgumentNotValidException ex
    ) {
        Map<String, String> errors = new HashMap<>();
        
        ex.getBindingResult().getAllErrors().forEach(error -> {
            String fieldName = ((FieldError) error).getField();
            String errorMessage = error.getDefaultMessage();
            errors.put(fieldName, errorMessage);
        });
        
        Map<String, Object> response = new HashMap<>();
        response.put("status", HttpStatus.BAD_REQUEST.value());
        response.put("error", "Validation Failed");
        response.put("errors", errors);
        response.put("timestamp", LocalDateTime.now());
        
        return ResponseEntity.badRequest().body(response);
    }
    
    // Handle generic exceptions
    @ExceptionHandler(Exception.class)
    public ResponseEntity<Map<String, Object>> handleGenericError(Exception ex) {
        Map<String, Object> response = new HashMap<>();
        response.put("status", HttpStatus.INTERNAL_SERVER_ERROR.value());
        response.put("error", "Internal Server Error");
        response.put("message", ex.getMessage());
        response.put("timestamp", LocalDateTime.now());
        
        return ResponseEntity
            .status(HttpStatus.INTERNAL_SERVER_ERROR)
            .body(response);
    }
}
```

---

## ขั้นตอนที่ 47: สร้าง API Response Wrapper

```java
// src/main/java/com/example/hellospring/dto/ApiResponse.java
package com.example.hellospring.dto;

import com.fasterxml.jackson.annotation.JsonInclude;
import lombok.Builder;
import lombok.Data;

import java.time.LocalDateTime;

@Data
@Builder
@JsonInclude(JsonInclude.Include.NON_NULL)  // ไม่แสดง null fields
public class ApiResponse<T> {
    
    private boolean success;
    private String message;
    private T data;
    private Object errors;
    
    @Builder.Default
    private LocalDateTime timestamp = LocalDateTime.now();
    
    // Factory methods
    public static <T> ApiResponse<T> success(T data) {
        return ApiResponse.<T>builder()
            .success(true)
            .data(data)
            .build();
    }
    
    public static <T> ApiResponse<T> success(String message, T data) {
        return ApiResponse.<T>builder()
            .success(true)
            .message(message)
            .data(data)
            .build();
    }
    
    public static <T> ApiResponse<T> error(String message) {
        return ApiResponse.<T>builder()
            .success(false)
            .message(message)
            .build();
    }
    
    public static <T> ApiResponse<T> error(String message, Object errors) {
        return ApiResponse.<T>builder()
            .success(false)
            .message(message)
            .errors(errors)
            .build();
    }
}
```

### Update Controller ให้ใช้ ApiResponse

```java
@GetMapping
public ResponseEntity<ApiResponse<List<Product>>> getAllProducts() {
    List<Product> products = productService.findAll();
    return ResponseEntity.ok(
        ApiResponse.success("ดึงข้อมูลสำเร็จ", products)
    );
}

@GetMapping("/{id}")
public ResponseEntity<ApiResponse<Product>> getProductById(@PathVariable Long id) {
    return productService.findById(id)
        .map(product -> ResponseEntity.ok(
            ApiResponse.success("พบสินค้า", product)
        ))
        .orElse(ResponseEntity.status(HttpStatus.NOT_FOUND).body(
            ApiResponse.error("ไม่พบสินค้า id: " + id)
        ));
}
```

---

## ขั้นตอนที่ 48: ทำความเข้าใจ JSON Serialization

```java
import com.fasterxml.jackson.annotation.*;

@Data
public class ProductResponse {
    
    private Long id;
    
    // เปลี่ยนชื่อ field ใน JSON
    @JsonProperty("product_name")
    private String name;
    
    // ไม่แสดง field นี้ใน JSON
    @JsonIgnore
    private String internalCode;
    
    // แสดงเฉพาะเมื่อ serialize (write to JSON)
    @JsonProperty(access = JsonProperty.Access.READ_ONLY)
    private LocalDateTime createdAt;
    
    // Date format
    @JsonFormat(pattern = "dd/MM/yyyy HH:mm:ss", timezone = "Asia/Bangkok")
    private LocalDateTime updatedAt;
    
    // Serialize เฉพาะเมื่อไม่ null
    @JsonInclude(JsonInclude.Include.NON_NULL)
    private String description;
    
    // Custom serialization
    @JsonSerialize(using = BigDecimalSerializer.class)
    private BigDecimal price;
}
```

### Jackson Configuration

```java
// src/main/java/com/example/hellospring/config/JacksonConfig.java
package com.example.hellospring.config;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.SerializationFeature;
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.text.SimpleDateFormat;

@Configuration
public class JacksonConfig {
    
    @Bean
    public ObjectMapper objectMapper() {
        ObjectMapper mapper = new ObjectMapper();
        
        // รองรับ Java 8 Date/Time
        mapper.registerModule(new JavaTimeModule());
        
        // ไม่แสดง dates เป็น timestamps
        mapper.disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS);
        
        // Date format
        mapper.setDateFormat(new SimpleDateFormat("yyyy-MM-dd HH:mm:ss"));
        
        return mapper;
    }
}
```

---

## ขั้นตอนที่ 49: เพิ่ม Health Check Endpoint

```java
// src/main/java/com/example/hellospring/controller/HealthController.java
package com.example.hellospring.controller;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import java.time.LocalDateTime;
import java.util.Map;

@RestController
@RequestMapping("/api")
public class HealthController {
    
    @Value("${spring.application.name}")
    private String appName;
    
    @Value("${server.port:8080}")
    private String serverPort;
    
    @GetMapping("/health")
    public Map<String, Object> health() {
        return Map.of(
            "status", "UP",
            "application", appName,
            "port", serverPort,
            "timestamp", LocalDateTime.now(),
            "java", System.getProperty("java.version"),
            "os", System.getProperty("os.name")
        );
    }
}
```

### ทดสอบ

```bash
curl http://localhost:8080/api/health

# Response:
{
  "status": "UP",
  "application": "hello-spring",
  "port": "8080",
  "timestamp": "2026-09-26T10:00:00",
  "java": "21.0.3",
  "os": "Mac OS X"
}
```

---

## ขั้นตอนที่ 50: เพิ่ม Actuator

Spring Boot Actuator ให้ production-ready endpoints ฟรี

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

```properties
# application.properties
# เปิด actuator endpoints
management.endpoints.web.exposure.include=health,info,metrics,env
management.endpoint.health.show-details=always
management.info.env.enabled=true

# App info ใน /actuator/info
info.app.name=Hello Spring Boot
info.app.version=1.0.0
info.app.description=Learning Spring Boot
```

### Actuator Endpoints

```bash
# Health
curl http://localhost:8080/actuator/health

# Info
curl http://localhost:8080/actuator/info

# Metrics
curl http://localhost:8080/actuator/metrics
curl http://localhost:8080/actuator/metrics/http.server.requests

# Environment
curl http://localhost:8080/actuator/env

# Beans ทั้งหมด
curl http://localhost:8080/actuator/beans

# Mappings
curl http://localhost:8080/actuator/mappings
```

---

## ขั้นตอนที่ 51: เข้าใจ Spring Boot Banner

```
เมื่อรัน Spring Boot Application จะเห็น banner:

  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::                (v3.3.4)
```

### สร้าง Custom Banner

```bash
# สร้างไฟล์ src/main/resources/banner.txt

# ไปที่ https://www.kammerl.de/ascii/AsciiSignature.php
# หรือ https://patorjk.com/software/taag/
# สร้าง ASCII art และ paste ลงในไฟล์
```

```
# src/main/resources/banner.txt
  __  __        _                 
 |  \/  |_   _ / \   _ __  _ __  
 | |\/| | | | / _ \ | '_ \| '_ \ 
 | |  | | |_| / ___ \| |_) | |_) |
 |_|  |_|\__, /_/   \_\ .__/| .__/ 
         |___/         |_|   |_|   
 
 :: Spring Boot ${spring-boot.version} ::
 :: Application: ${spring.application.name} ::
 :: Profile: ${spring.profiles.active} ::
```

---

## ขั้นตอนที่ 52: ทดสอบแบบอัตโนมัติ (Automated Testing)

```java
// src/test/java/com/example/hellospring/controller/ProductControllerTest.java
package com.example.hellospring.controller;

import com.example.hellospring.model.Product;
import com.example.hellospring.service.ProductService;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.boot.test.mock.mockito.MockBean;
import org.springframework.http.MediaType;
import org.springframework.test.web.servlet.MockMvc;

import java.math.BigDecimal;
import java.util.List;
import java.util.Optional;

import static org.mockito.Mockito.*;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;
import static org.hamcrest.Matchers.*;

@WebMvcTest(ProductController.class)  // ทดสอบเฉพาะ Controller layer
class ProductControllerTest {
    
    @Autowired
    private MockMvc mockMvc;  // จำลอง HTTP requests
    
    @MockBean
    private ProductService productService;  // Mock Service
    
    @Test
    void getAllProducts_ShouldReturn200_WithProductList() throws Exception {
        // Arrange - เตรียมข้อมูล
        List<Product> products = List.of(
            Product.builder().id(1L).name("MacBook Pro").price(new BigDecimal("69900")).build(),
            Product.builder().id(2L).name("iPhone 15").price(new BigDecimal("44900")).build()
        );
        when(productService.findAll()).thenReturn(products);
        
        // Act & Assert - ทดสอบ
        mockMvc.perform(get("/api/products")
                .contentType(MediaType.APPLICATION_JSON))
            .andExpect(status().isOk())                    // 200
            .andExpect(jsonPath("$", hasSize(2)))          // 2 products
            .andExpect(jsonPath("$[0].name", is("MacBook Pro")))
            .andExpect(jsonPath("$[1].name", is("iPhone 15")));
        
        verify(productService, times(1)).findAll();
    }
    
    @Test
    void getProductById_WhenExists_ShouldReturn200() throws Exception {
        // Arrange
        Product product = Product.builder()
            .id(1L)
            .name("MacBook Pro")
            .price(new BigDecimal("69900"))
            .build();
        when(productService.findById(1L)).thenReturn(Optional.of(product));
        
        // Act & Assert
        mockMvc.perform(get("/api/products/1"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.id", is(1)))
            .andExpect(jsonPath("$.name", is("MacBook Pro")));
    }
    
    @Test
    void getProductById_WhenNotExists_ShouldReturn404() throws Exception {
        // Arrange
        when(productService.findById(999L)).thenReturn(Optional.empty());
        
        // Act & Assert
        mockMvc.perform(get("/api/products/999"))
            .andExpect(status().isNotFound());
    }
}
```

### รัน Tests

```bash
# รัน ทุก tests
mvn test

# รัน specific test class
mvn test -Dtest=ProductControllerTest

# รัน specific test method
mvn test -Dtest=ProductControllerTest#getAllProducts_ShouldReturn200_WithProductList

# ดู test report
open target/surefire-reports/index.html
```

---

## ขั้นตอนที่ 53: เพิ่ม CORS Configuration

CORS (Cross-Origin Resource Sharing) จำเป็นเมื่อ Frontend และ Backend อยู่ต่าง domain

```java
// src/main/java/com/example/hellospring/config/WebConfig.java
package com.example.hellospring.config;

import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.CorsRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

@Configuration
public class WebConfig implements WebMvcConfigurer {
    
    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")
            .allowedOrigins(
                "http://localhost:3000",    // React dev server
                "http://localhost:4200",    // Angular dev server
                "https://myapp.com"         // Production
            )
            .allowedMethods("GET", "POST", "PUT", "DELETE", "PATCH", "OPTIONS")
            .allowedHeaders("*")
            .allowCredentials(true)
            .maxAge(3600);  // Pre-flight request cache 1 hour
    }
}
```

---

## ขั้นตอนที่ 54: Build และรัน JAR

```bash
# Build executable JAR
mvn clean package -DskipTests

# JAR จะอยู่ที่ target/hello-spring-0.0.1-SNAPSHOT.jar

# รัน JAR
java -jar target/hello-spring-0.0.1-SNAPSHOT.jar

# รัน JAR พร้อม profile
java -jar target/hello-spring-0.0.1-SNAPSHOT.jar --spring.profiles.active=prod

# รัน JAR พร้อม JVM options (แนะนำสำหรับ production)
java \
  -Xms256m \
  -Xmx512m \
  -XX:+UseG1GC \
  -XX:+UseContainerSupport \
  -jar target/hello-spring-0.0.1-SNAPSHOT.jar

# ดู JAR size
ls -lh target/hello-spring-0.0.1-SNAPSHOT.jar

# ดู contents ของ JAR
jar tf target/hello-spring-0.0.1-SNAPSHOT.jar | head -50
```

---

## ขั้นตอนที่ 55: สร้าง Dockerfile พื้นฐาน

```dockerfile
# Dockerfile
FROM eclipse-temurin:21-jre-alpine

WORKDIR /app

# Copy JAR file
COPY target/hello-spring-0.0.1-SNAPSHOT.jar app.jar

# เปิด port
EXPOSE 8080

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=30s --retries=3 \
  CMD curl -f http://localhost:8080/actuator/health || exit 1

# รัน application
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### Build และรัน Docker Image

```bash
# Build image
docker build -t hello-spring:latest .

# รัน container
docker run -p 8080:8080 hello-spring:latest

# รัน พร้อม environment variables
docker run -p 8080:8080 \
  -e SPRING_PROFILES_ACTIVE=prod \
  -e DATABASE_URL=jdbc:postgresql://host:5432/db \
  hello-spring:latest
```

---

## ขั้นตอนที่ 56: Logging Configuration

```properties
# application.properties

# Log levels
logging.level.root=INFO
logging.level.com.example=DEBUG
logging.level.org.springframework.web=DEBUG
logging.level.org.hibernate.SQL=DEBUG
logging.level.org.hibernate.type.descriptor.sql=TRACE

# Log format
logging.pattern.console=%d{yyyy-MM-dd HH:mm:ss} [%thread] %-5level %logger{36} - %msg%n
logging.pattern.file=%d{yyyy-MM-dd HH:mm:ss} [%thread] %-5level %logger{36} - %msg%n

# Log file
logging.file.name=logs/application.log
logging.file.max-size=10MB
logging.file.max-history=30
logging.file.total-size-cap=100MB
```

### ใช้ Logger ใน Code

```java
import lombok.extern.slf4j.Slf4j;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

// วิธีที่ 1: ใช้ Lombok @Slf4j (แนะนำ!)
@Slf4j
@Service
public class ProductService {
    
    public Product findById(Long id) {
        log.debug("Finding product id: {}", id);  // {} = placeholder
        log.info("Product found: {}", productName);
        log.warn("Product quantity is low: {}", id);
        log.error("Failed to find product: {}", id, exception);
        
        return product;
    }
}

// วิธีที่ 2: Manual Logger
@Service
public class OrderService {
    
    private static final Logger log = LoggerFactory.getLogger(OrderService.class);
    
    // ...
}
```

---

## ขั้นตอนที่ 57: เพิ่ม Application Info

```java
// src/main/java/com/example/hellospring/config/AppInfo.java
package com.example.hellospring.config;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.boot.CommandLineRunner;
import org.springframework.stereotype.Component;

import lombok.extern.slf4j.Slf4j;

@Component
@Slf4j
public class AppInfo implements CommandLineRunner {
    
    @Value("${spring.application.name}")
    private String appName;
    
    @Value("${server.port:8080}")
    private String port;
    
    @Value("${spring.profiles.active:default}")
    private String profile;
    
    @Override
    public void run(String... args) {
        log.info("===========================================");
        log.info("  Application: {}", appName);
        log.info("  Port: {}", port);
        log.info("  Profile: {}", profile);
        log.info("  API: http://localhost:{}/api", port);
        log.info("  Health: http://localhost:{}/actuator/health", port);
        log.info("===========================================");
    }
}
```

---

## ขั้นตอนที่ 58: CommandLineRunner vs ApplicationRunner

```java
// CommandLineRunner - args เป็น String[]
@Component
@Slf4j
public class DataInitializer implements CommandLineRunner {
    
    @Override
    public void run(String... args) {
        // รันหลัง Application Context พร้อมแล้ว
        log.info("Application started with args: {}", (Object) args);
        
        // ใช้สำหรับ:
        // - Initialize data
        // - Connect to external services
        // - Warm up caches
        // - Log startup info
    }
}

// ApplicationRunner - args เป็น ApplicationArguments (ดีกว่า)
@Component
@Order(1)  // ลำดับการรัน (1 = แรกสุด)
public class AppStartupTask implements ApplicationRunner {
    
    @Override
    public void run(ApplicationArguments args) {
        log.info("Option names: {}", args.getOptionNames());
        log.info("Source args: {}", (Object) args.getSourceArgs());
        
        if (args.containsOption("debug")) {
            log.debug("Running in debug mode");
        }
    }
}
```

---

## ขั้นตอนที่ 59: @ConfigurationProperties

```java
// src/main/java/com/example/hellospring/config/AppProperties.java
package com.example.hellospring.config;

import lombok.Data;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.stereotype.Component;

import java.util.List;

@Component
@ConfigurationProperties(prefix = "app")  // อ่านจาก app.* ใน properties
@Data
public class AppProperties {
    
    private String name;
    private String version;
    private String description;
    
    private Security security = new Security();
    private Upload upload = new Upload();
    
    @Data
    public static class Security {
        private String jwtSecret;
        private long jwtExpiration = 86400000;  // 24 hours
        private List<String> allowedOrigins;
    }
    
    @Data
    public static class Upload {
        private String directory;
        private long maxSize = 10485760;  // 10MB
        private List<String> allowedTypes;
    }
}
```

```properties
# application.properties
app.name=Hello Spring Boot
app.version=1.0.0
app.description=Learning Spring Boot Application

app.security.jwt-secret=mySecretKey123456789
app.security.jwt-expiration=86400000
app.security.allowed-origins=http://localhost:3000,https://myapp.com

app.upload.directory=/tmp/uploads
app.upload.max-size=10485760
app.upload.allowed-types=image/jpeg,image/png,application/pdf
```

```java
// ใช้งาน
@Service
@RequiredArgsConstructor
public class FileService {
    
    private final AppProperties appProperties;
    
    public void uploadFile(MultipartFile file) {
        long maxSize = appProperties.getUpload().getMaxSize();
        List<String> allowedTypes = appProperties.getUpload().getAllowedTypes();
        String uploadDir = appProperties.getUpload().getDirectory();
        
        // validate and upload
    }
}
```

---

## ขั้นตอนที่ 60: สรุป Part 03

### สิ่งที่สร้างได้แล้ว

```
✅ Spring Boot Application ที่ทำงานได้จริง
✅ REST API (GET, POST, PUT, DELETE)
✅ Controller, Service layers
✅ ResponseEntity สำหรับ HTTP responses
✅ Request Validation ด้วย @Valid
✅ Global Exception Handler
✅ ApiResponse wrapper
✅ Health Check endpoint
✅ Actuator endpoints
✅ CORS Configuration
✅ Logging
✅ Dockerfile พื้นฐาน
✅ CommandLineRunner สำหรับ startup tasks
✅ @ConfigurationProperties
```

### โปรเจคตัวอย่างที่สมบูรณ์

```
hello-spring/
├── src/main/java/com/example/hellospring/
│   ├── HelloSpringApplication.java
│   ├── config/
│   │   ├── AppInfo.java
│   │   ├── AppProperties.java
│   │   ├── JacksonConfig.java
│   │   └── WebConfig.java
│   ├── controller/
│   │   ├── HelloController.java
│   │   ├── HealthController.java
│   │   └── ProductController.java
│   ├── dto/
│   │   ├── ApiResponse.java
│   │   └── CreateProductRequest.java
│   ├── exception/
│   │   └── GlobalExceptionHandler.java
│   ├── model/
│   │   └── Product.java
│   └── service/
│       └── ProductService.java
├── src/main/resources/
│   ├── application.properties
│   ├── banner.txt
│   └── static/
├── src/test/
├── Dockerfile
├── docker-compose.yml
└── pom.xml
```

---

*[← Part 02: ติดตั้ง Environment](./part-02-setup-environment.md) | [Part 04: โครงสร้างโปรเจค →](./part-04-project-structure.md)*

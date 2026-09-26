# Part 05: REST API พื้นฐานด้วย Spring MVC
## ขั้นตอนที่ 81-110

> **ระดับ:** พื้นฐาน (Beginner)  
> **เวลาเรียน:** 4-5 ชั่วโมง  
> **เป้าหมาย:** สร้าง REST API ระดับ Production ด้วย Spring MVC

---

## ขั้นตอนที่ 81: REST คืออะไร?

REST (Representational State Transfer) เป็น architectural style สำหรับการออกแบบ Web Services

### 6 Constraints ของ REST

```
1. Client-Server
   - แยก UI (Client) กับ Data Storage (Server)
   - Client ไม่ต้องรู้ว่า Server เก็บข้อมูลอย่างไร
   
2. Stateless
   - Server ไม่เก็บ session state ของ client
   - ทุก request ต้องมีข้อมูลครบในตัวเอง
   - Authentication ต้องส่งมาทุก request (ผ่าน JWT)

3. Cacheable
   - Response ระบุได้ว่า cacheable หรือไม่
   - ช่วยลด load บน server

4. Uniform Interface
   - ใช้ HTTP methods ถูกต้อง (GET, POST, PUT, DELETE)
   - Resource ถูกระบุด้วย URI
   - Hypermedia (HATEOAS)

5. Layered System
   - Client ไม่รู้ว่ากำลัง communicate กับ server ตัวไหน
   - ผ่าน Load Balancer, CDN, API Gateway ได้

6. Code on Demand (Optional)
   - Server ส่ง executable code ให้ client ได้ (JavaScript)
```

---

## ขั้นตอนที่ 82: HTTP Methods และการใช้งาน

```
HTTP Methods → CRUD Operations:

GET    → Read    → ดึงข้อมูล (ไม่ควรเปลี่ยนแปลงข้อมูล)
POST   → Create  → สร้างข้อมูลใหม่
PUT    → Update  → อัปเดตข้อมูลทั้งหมด (replace)
PATCH  → Update  → อัปเดตข้อมูลบางส่วน (partial update)
DELETE → Delete  → ลบข้อมูล

Properties:
                    Safe    Idempotent
GET                 ✅       ✅
POST                ❌       ❌
PUT                 ❌       ✅
PATCH               ❌       ❌
DELETE              ❌       ✅

Safe = ไม่เปลี่ยนแปลง state ของ server
Idempotent = เรียกซ้ำกี่ครั้งก็ได้ผลเหมือนเดิม
```

---

## ขั้นตอนที่ 83: สร้าง Complete CRUD API

```java
// src/main/java/com/example/myapp/controller/UserController.java
package com.example.myapp.controller;

import com.example.myapp.dto.ApiResponse;
import com.example.myapp.dto.request.CreateUserRequest;
import com.example.myapp.dto.request.UpdateUserRequest;
import com.example.myapp.dto.response.UserResponse;
import com.example.myapp.service.UserService;
import jakarta.validation.Valid;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.PageRequest;
import org.springframework.data.domain.Pageable;
import org.springframework.data.domain.Sort;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.List;

@RestController
@RequestMapping("/api/v1/users")
@RequiredArgsConstructor
@Slf4j
public class UserController {
    
    private final UserService userService;
    
    // ========================================
    // GET Endpoints
    // ========================================
    
    /**
     * GET /api/v1/users
     * ดึงรายการ users ทั้งหมด พร้อม pagination
     */
    @GetMapping
    public ResponseEntity<ApiResponse<Page<UserResponse>>> getAllUsers(
        @RequestParam(defaultValue = "0") int page,
        @RequestParam(defaultValue = "10") int size,
        @RequestParam(defaultValue = "id") String sortBy,
        @RequestParam(defaultValue = "asc") String sortDir
    ) {
        Sort sort = sortDir.equalsIgnoreCase("asc") 
            ? Sort.by(sortBy).ascending() 
            : Sort.by(sortBy).descending();
        
        Pageable pageable = PageRequest.of(page, size, sort);
        Page<UserResponse> users = userService.findAll(pageable);
        
        return ResponseEntity.ok(ApiResponse.success(users));
    }
    
    /**
     * GET /api/v1/users/{id}
     * ดึง user ตาม ID
     */
    @GetMapping("/{id}")
    public ResponseEntity<ApiResponse<UserResponse>> getUserById(@PathVariable Long id) {
        UserResponse user = userService.findById(id);
        return ResponseEntity.ok(ApiResponse.success(user));
    }
    
    /**
     * GET /api/v1/users/search?q=john&role=ADMIN
     * ค้นหา users
     */
    @GetMapping("/search")
    public ResponseEntity<ApiResponse<List<UserResponse>>> searchUsers(
        @RequestParam(required = false) String q,
        @RequestParam(required = false) String role
    ) {
        List<UserResponse> users = userService.search(q, role);
        return ResponseEntity.ok(ApiResponse.success(users));
    }
    
    // ========================================
    // POST Endpoints
    // ========================================
    
    /**
     * POST /api/v1/users
     * สร้าง user ใหม่
     */
    @PostMapping
    public ResponseEntity<ApiResponse<UserResponse>> createUser(
        @Valid @RequestBody CreateUserRequest request
    ) {
        log.info("Creating user with email: {}", request.email());
        UserResponse created = userService.create(request);
        
        return ResponseEntity
            .status(HttpStatus.CREATED)
            .body(ApiResponse.success("User created successfully", created));
    }
    
    // ========================================
    // PUT Endpoints
    // ========================================
    
    /**
     * PUT /api/v1/users/{id}
     * อัปเดต user ทั้งหมด (full update)
     */
    @PutMapping("/{id}")
    public ResponseEntity<ApiResponse<UserResponse>> updateUser(
        @PathVariable Long id,
        @Valid @RequestBody UpdateUserRequest request
    ) {
        log.info("Updating user id: {}", id);
        UserResponse updated = userService.update(id, request);
        return ResponseEntity.ok(ApiResponse.success("User updated successfully", updated));
    }
    
    // ========================================
    // PATCH Endpoints
    // ========================================
    
    /**
     * PATCH /api/v1/users/{id}/deactivate
     * ปิดใช้งาน user
     */
    @PatchMapping("/{id}/deactivate")
    public ResponseEntity<ApiResponse<Void>> deactivateUser(@PathVariable Long id) {
        userService.deactivate(id);
        return ResponseEntity.ok(ApiResponse.success("User deactivated"));
    }
    
    /**
     * PATCH /api/v1/users/{id}/activate
     * เปิดใช้งาน user
     */
    @PatchMapping("/{id}/activate")
    public ResponseEntity<ApiResponse<Void>> activateUser(@PathVariable Long id) {
        userService.activate(id);
        return ResponseEntity.ok(ApiResponse.success("User activated"));
    }
    
    /**
     * PATCH /api/v1/users/{id}/password
     * เปลี่ยนรหัสผ่าน
     */
    @PatchMapping("/{id}/password")
    public ResponseEntity<ApiResponse<Void>> changePassword(
        @PathVariable Long id,
        @Valid @RequestBody ChangePasswordRequest request
    ) {
        userService.changePassword(id, request);
        return ResponseEntity.ok(ApiResponse.success("Password changed successfully"));
    }
    
    // ========================================
    // DELETE Endpoints
    // ========================================
    
    /**
     * DELETE /api/v1/users/{id}
     * ลบ user (soft delete)
     */
    @DeleteMapping("/{id}")
    public ResponseEntity<ApiResponse<Void>> deleteUser(@PathVariable Long id) {
        log.info("Deleting user id: {}", id);
        userService.delete(id);
        return ResponseEntity.ok(ApiResponse.success("User deleted successfully"));
    }
    
    /**
     * DELETE /api/v1/users (Batch delete)
     * ลบหลาย users พร้อมกัน
     */
    @DeleteMapping
    public ResponseEntity<ApiResponse<Void>> deleteMultipleUsers(
        @RequestParam List<Long> ids
    ) {
        userService.deleteMultiple(ids);
        return ResponseEntity.ok(
            ApiResponse.success(ids.size() + " users deleted successfully")
        );
    }
}
```

---

## ขั้นตอนที่ 84: Request Parameter Types

```java
@RestController
@RequestMapping("/api/v1/demo")
public class DemoController {
    
    // 1. @PathVariable - ค่าจาก URL path
    @GetMapping("/users/{id}")
    public String pathVariable(@PathVariable Long id) {
        return "User ID: " + id;
    }
    
    // @PathVariable พร้อม custom name
    @GetMapping("/products/{product-id}/reviews/{review-id}")
    public String multiplePathVars(
        @PathVariable("product-id") Long productId,
        @PathVariable("review-id") Long reviewId
    ) {
        return String.format("Product: %d, Review: %d", productId, reviewId);
    }
    
    // 2. @RequestParam - Query parameters
    @GetMapping("/search")
    public String requestParam(
        @RequestParam String q,                              // บังคับ
        @RequestParam(required = false) String category,    // optional
        @RequestParam(defaultValue = "0") int page,         // มีค่า default
        @RequestParam(defaultValue = "10") int size,
        @RequestParam(name = "sort-by", defaultValue = "id") String sortBy,
        @RequestParam List<String> tags                     // หลายค่า ?tags=java&tags=spring
    ) {
        return "Search: " + q;
    }
    
    // 3. @RequestBody - JSON body
    @PostMapping("/create")
    public String requestBody(@RequestBody CreateRequest request) {
        return "Created: " + request.getName();
    }
    
    // 4. @RequestHeader - HTTP headers
    @GetMapping("/protected")
    public String requestHeader(
        @RequestHeader("Authorization") String auth,
        @RequestHeader(value = "X-Custom-Header", required = false) String customHeader
    ) {
        return "Auth: " + auth;
    }
    
    // 5. @CookieValue - Cookies
    @GetMapping("/cookie")
    public String cookie(
        @CookieValue(value = "session-id", required = false) String sessionId
    ) {
        return "Session: " + sessionId;
    }
    
    // 6. HttpServletRequest - Raw request
    @GetMapping("/raw")
    public String rawRequest(HttpServletRequest request) {
        String clientIp = request.getRemoteAddr();
        String userAgent = request.getHeader("User-Agent");
        return "IP: " + clientIp + ", UA: " + userAgent;
    }
    
    // 7. MultipartFile - File upload
    @PostMapping("/upload")
    public String fileUpload(@RequestParam("file") MultipartFile file) {
        return "Uploaded: " + file.getOriginalFilename() + 
               ", Size: " + file.getSize();
    }
}
```

---

## ขั้นตอนที่ 85: Request Validation ขั้นสูง

```java
// Built-in constraints
public record CreateProductRequest(
    
    // String validations
    @NotNull String name,
    @NotBlank String description,        // NotBlank = NotNull + not whitespace
    @NotEmpty String category,           // NotEmpty = NotNull + not empty string
    @Size(min = 2, max = 100) String title,
    @Email String email,
    @Pattern(regexp = "^\\d{10}$") String phone,
    @URL String websiteUrl,
    
    // Number validations
    @Min(1) Integer quantity,
    @Max(9999) Integer maxQuantity,
    @DecimalMin("0.01") BigDecimal price,
    @DecimalMax("9999999.99") BigDecimal maxPrice,
    @Positive BigDecimal cost,           // > 0
    @PositiveOrZero BigDecimal discount, // >= 0
    @Negative BigDecimal refund,         // < 0
    @NegativeOrZero BigDecimal loss,     // <= 0
    @Digits(integer = 10, fraction = 2) BigDecimal amount,
    
    // Date validations
    @Past LocalDate birthDate,           // ต้องเป็นอดีต
    @PastOrPresent LocalDate enrollDate,
    @Future LocalDate expiryDate,        // ต้องเป็นอนาคต
    @FutureOrPresent LocalDate startDate,
    
    // Boolean
    @AssertTrue Boolean termsAccepted,
    @AssertFalse Boolean isDeleted,
    
    // Collection
    @NotEmpty List<String> tags,
    @Size(max = 5) List<String> categories,
    
    // Nested object validation
    @Valid AddressRequest address        // validate nested object ด้วย
) {}
```

### Custom Validator

```java
// 1. สร้าง Annotation
// src/main/java/com/example/myapp/validation/ValidPhoneNumber.java
package com.example.myapp.validation;

import jakarta.validation.Constraint;
import jakarta.validation.Payload;

import java.lang.annotation.*;

@Documented
@Constraint(validatedBy = PhoneNumberValidator.class)
@Target({ElementType.FIELD, ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
public @interface ValidPhoneNumber {
    String message() default "Invalid Thai phone number";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}
```

```java
// 2. สร้าง Validator
// src/main/java/com/example/myapp/validation/PhoneNumberValidator.java
package com.example.myapp.validation;

import jakarta.validation.ConstraintValidator;
import jakarta.validation.ConstraintValidatorContext;

public class PhoneNumberValidator implements ConstraintValidator<ValidPhoneNumber, String> {
    
    // Thai phone number: 0X-XXXX-XXXX หรือ 0XXXXXXXXX
    private static final String THAI_PHONE_PATTERN = "^0[0-9]{8,9}$";
    
    @Override
    public boolean isValid(String phoneNumber, ConstraintValidatorContext context) {
        if (phoneNumber == null || phoneNumber.isEmpty()) {
            return true;  // ปล่อยให้ @NotBlank จัดการ
        }
        
        String cleaned = phoneNumber.replaceAll("[\\s\\-()]", "");
        return cleaned.matches(THAI_PHONE_PATTERN);
    }
}
```

```java
// 3. ใช้งาน
public record UserRequest(
    @ValidPhoneNumber
    String phone
) {}
```

### Class-level Validator (Cross-field validation)

```java
// Annotation
@Documented
@Constraint(validatedBy = PasswordMatchValidator.class)
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface PasswordMatch {
    String message() default "Passwords do not match";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}

// Validator
public class PasswordMatchValidator 
    implements ConstraintValidator<PasswordMatch, ChangePasswordRequest> {
    
    @Override
    public boolean isValid(ChangePasswordRequest request, ConstraintValidatorContext context) {
        if (request.newPassword() == null || request.confirmPassword() == null) {
            return false;
        }
        return request.newPassword().equals(request.confirmPassword());
    }
}

// ใช้งาน - @Valid ที่ class level
@PasswordMatch
public record ChangePasswordRequest(
    @NotBlank String currentPassword,
    @NotBlank @Size(min = 8) String newPassword,
    @NotBlank String confirmPassword
) {}
```

---

## ขั้นตอนที่ 86: Pagination และ Sorting

```java
// Controller
@GetMapping
public ResponseEntity<Page<ProductResponse>> getProducts(
    @RequestParam(defaultValue = "0") int page,
    @RequestParam(defaultValue = "10") int size,
    @RequestParam(defaultValue = "createdAt") String sortBy,
    @RequestParam(defaultValue = "desc") String direction
) {
    // ป้องกัน page size ใหญ่เกินไป
    size = Math.min(size, 100);
    
    Sort.Direction sortDirection = direction.equalsIgnoreCase("asc") 
        ? Sort.Direction.ASC 
        : Sort.Direction.DESC;
    
    Pageable pageable = PageRequest.of(page, size, Sort.by(sortDirection, sortBy));
    
    return ResponseEntity.ok(productService.findAll(pageable));
}
```

### Pagination Response Structure

```json
{
  "content": [...],
  "pageable": {
    "pageNumber": 0,
    "pageSize": 10,
    "sort": {
      "sorted": true,
      "direction": "ASC",
      "property": "name"
    }
  },
  "totalElements": 100,
  "totalPages": 10,
  "first": true,
  "last": false,
  "numberOfElements": 10,
  "empty": false
}
```

### Custom Pagination Response

```java
// DTO ที่ format ดีกว่า
@Builder
public record PagedResponse<T>(
    List<T> items,
    int page,
    int size,
    long totalElements,
    int totalPages,
    boolean first,
    boolean last
) {
    public static <T> PagedResponse<T> from(Page<T> page) {
        return PagedResponse.<T>builder()
            .items(page.getContent())
            .page(page.getNumber())
            .size(page.getSize())
            .totalElements(page.getTotalElements())
            .totalPages(page.getTotalPages())
            .first(page.isFirst())
            .last(page.isLast())
            .build();
    }
}
```

---

## ขั้นตอนที่ 87: Content Negotiation

```java
// รองรับทั้ง JSON และ XML
@GetMapping(value = "/{id}", 
    produces = {MediaType.APPLICATION_JSON_VALUE, MediaType.APPLICATION_XML_VALUE})
public ResponseEntity<UserResponse> getUserById(@PathVariable Long id) {
    // Spring จะ serialize เป็น JSON หรือ XML ตาม Accept header
    return ResponseEntity.ok(userService.findById(id));
}

// Client ส่ง: Accept: application/json
// → Response เป็น JSON

// Client ส่ง: Accept: application/xml
// → Response เป็น XML (ต้องเพิ่ม jackson-dataformat-xml dependency)
```

---

## ขั้นตอนที่ 88: Filtering ด้วย Specification

```java
// src/main/java/com/example/myapp/specification/ProductSpecification.java
package com.example.myapp.specification;

import com.example.myapp.entity.Product;
import jakarta.persistence.criteria.*;
import org.springframework.data.jpa.domain.Specification;

import java.math.BigDecimal;

public class ProductSpecification {
    
    public static Specification<Product> hasCategory(String category) {
        return (root, query, builder) -> {
            if (category == null || category.isEmpty()) {
                return builder.conjunction();  // No filter
            }
            return builder.equal(
                builder.lower(root.get("category")),
                category.toLowerCase()
            );
        };
    }
    
    public static Specification<Product> hasPriceBetween(
        BigDecimal minPrice, BigDecimal maxPrice
    ) {
        return (root, query, builder) -> {
            if (minPrice == null && maxPrice == null) {
                return builder.conjunction();
            }
            if (minPrice == null) {
                return builder.lessThanOrEqualTo(root.get("price"), maxPrice);
            }
            if (maxPrice == null) {
                return builder.greaterThanOrEqualTo(root.get("price"), minPrice);
            }
            return builder.between(root.get("price"), minPrice, maxPrice);
        };
    }
    
    public static Specification<Product> nameContains(String keyword) {
        return (root, query, builder) -> {
            if (keyword == null || keyword.isEmpty()) {
                return builder.conjunction();
            }
            return builder.like(
                builder.lower(root.get("name")),
                "%" + keyword.toLowerCase() + "%"
            );
        };
    }
    
    public static Specification<Product> isActive() {
        return (root, query, builder) -> builder.isTrue(root.get("active"));
    }
}
```

```java
// Service
public Page<ProductResponse> searchProducts(
    String keyword, String category, 
    BigDecimal minPrice, BigDecimal maxPrice,
    Pageable pageable
) {
    Specification<Product> spec = Specification
        .where(ProductSpecification.isActive())
        .and(ProductSpecification.nameContains(keyword))
        .and(ProductSpecification.hasCategory(category))
        .and(ProductSpecification.hasPriceBetween(minPrice, maxPrice));
    
    return productRepository.findAll(spec, pageable)
        .map(productMapper::toResponse);
}
```

```java
// Controller
@GetMapping("/search")
public ResponseEntity<Page<ProductResponse>> searchProducts(
    @RequestParam(required = false) String keyword,
    @RequestParam(required = false) String category,
    @RequestParam(required = false) BigDecimal minPrice,
    @RequestParam(required = false) BigDecimal maxPrice,
    Pageable pageable
) {
    return ResponseEntity.ok(
        productService.searchProducts(keyword, category, minPrice, maxPrice, pageable)
    );
}
```

---

## ขั้นตอนที่ 89: File Upload API

```java
// src/main/java/com/example/myapp/controller/FileController.java
package com.example.myapp.controller;

import lombok.RequiredArgsConstructor;
import org.springframework.core.io.Resource;
import org.springframework.core.io.UrlResource;
import org.springframework.http.HttpHeaders;
import org.springframework.http.MediaType;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.multipart.MultipartFile;

import java.io.IOException;
import java.nio.file.*;
import java.util.UUID;

@RestController
@RequestMapping("/api/v1/files")
public class FileController {
    
    private static final Path UPLOAD_DIR = Paths.get("uploads");
    private static final long MAX_FILE_SIZE = 10 * 1024 * 1024;  // 10MB
    private static final List<String> ALLOWED_TYPES = List.of(
        "image/jpeg", "image/png", "image/gif", "application/pdf"
    );
    
    // Upload single file
    @PostMapping("/upload")
    public ResponseEntity<Map<String, String>> uploadFile(
        @RequestParam("file") MultipartFile file
    ) throws IOException {
        
        // Validate
        if (file.isEmpty()) {
            return ResponseEntity.badRequest()
                .body(Map.of("error", "File is empty"));
        }
        
        if (file.getSize() > MAX_FILE_SIZE) {
            return ResponseEntity.badRequest()
                .body(Map.of("error", "File size exceeds 10MB limit"));
        }
        
        if (!ALLOWED_TYPES.contains(file.getContentType())) {
            return ResponseEntity.badRequest()
                .body(Map.of("error", "File type not allowed"));
        }
        
        // Save file
        String filename = UUID.randomUUID() + "_" + 
            file.getOriginalFilename().replaceAll("[^a-zA-Z0-9._-]", "_");
        
        Files.createDirectories(UPLOAD_DIR);
        Path targetPath = UPLOAD_DIR.resolve(filename);
        Files.copy(file.getInputStream(), targetPath, StandardCopyOption.REPLACE_EXISTING);
        
        return ResponseEntity.ok(Map.of(
            "filename", filename,
            "originalName", file.getOriginalFilename(),
            "size", String.valueOf(file.getSize()),
            "url", "/api/v1/files/download/" + filename
        ));
    }
    
    // Upload multiple files
    @PostMapping("/upload-multiple")
    public ResponseEntity<List<Map<String, String>>> uploadMultipleFiles(
        @RequestParam("files") List<MultipartFile> files
    ) throws IOException {
        
        List<Map<String, String>> results = new ArrayList<>();
        
        for (MultipartFile file : files) {
            if (!file.isEmpty()) {
                String filename = UUID.randomUUID() + "_" + file.getOriginalFilename();
                Files.copy(
                    file.getInputStream(),
                    UPLOAD_DIR.resolve(filename),
                    StandardCopyOption.REPLACE_EXISTING
                );
                results.add(Map.of("filename", filename, "size", String.valueOf(file.getSize())));
            }
        }
        
        return ResponseEntity.ok(results);
    }
    
    // Download file
    @GetMapping("/download/{filename}")
    public ResponseEntity<Resource> downloadFile(@PathVariable String filename) 
        throws IOException {
        
        Path filePath = UPLOAD_DIR.resolve(filename).normalize();
        
        // Security check - ป้องกัน path traversal
        if (!filePath.startsWith(UPLOAD_DIR)) {
            return ResponseEntity.badRequest().build();
        }
        
        Resource resource = new UrlResource(filePath.toUri());
        
        if (!resource.exists() || !resource.isReadable()) {
            return ResponseEntity.notFound().build();
        }
        
        // Detect content type
        String contentType = Files.probeContentType(filePath);
        if (contentType == null) {
            contentType = "application/octet-stream";
        }
        
        return ResponseEntity.ok()
            .contentType(MediaType.parseMediaType(contentType))
            .header(HttpHeaders.CONTENT_DISPOSITION, 
                    "attachment; filename=\"" + resource.getFilename() + "\"")
            .body(resource);
    }
}
```

---

## ขั้นตอนที่ 90: API Versioning

```java
// Strategy 1: URL versioning (แนะนำสำหรับ major changes)
@RequestMapping("/api/v1/products")
// @RequestMapping("/api/v2/products")

// Strategy 2: Header versioning
@GetMapping(value = "/products", 
            headers = "API-Version=1")
public ResponseEntity<?> getProductsV1() { ... }

@GetMapping(value = "/products", 
            headers = "API-Version=2")
public ResponseEntity<?> getProductsV2() { ... }

// Strategy 3: Accept header (Content negotiation)
@GetMapping(value = "/products",
            produces = "application/vnd.myapp.v1+json")
public ResponseEntity<?> getProductsV1() { ... }

@GetMapping(value = "/products",
            produces = "application/vnd.myapp.v2+json")
public ResponseEntity<?> getProductsV2() { ... }

// Strategy 4: Query parameter
@GetMapping("/products")
public ResponseEntity<?> getProducts(
    @RequestParam(defaultValue = "1") int apiVersion
) {
    if (apiVersion == 2) {
        return ResponseEntity.ok(productServiceV2.findAll());
    }
    return ResponseEntity.ok(productService.findAll());
}
```

---

## ขั้นตอนที่ 91: HTTP Response Headers

```java
@GetMapping("/{id}")
public ResponseEntity<ProductResponse> getProduct(@PathVariable Long id) {
    ProductResponse product = productService.findById(id);
    
    return ResponseEntity.ok()
        // Cache control
        .header(HttpHeaders.CACHE_CONTROL, "max-age=3600")
        // ETag สำหรับ conditional requests
        .header(HttpHeaders.ETAG, "\"" + product.version() + "\"")
        // Custom headers
        .header("X-Request-ID", UUID.randomUUID().toString())
        .body(product);
}
```

### ResponseEntity Builder Pattern

```java
// ตัวอย่างต่างๆ
ResponseEntity.ok(body)                           // 200
ResponseEntity.created(uri).body(body)            // 201
ResponseEntity.accepted().build()                 // 202
ResponseEntity.noContent().build()                // 204
ResponseEntity.badRequest().body(error)           // 400
ResponseEntity.notFound().build()                 // 404
ResponseEntity.status(HttpStatus.CONFLICT).body(error) // 409

// Full builder
ResponseEntity.status(200)
    .header("X-Custom", "value")
    .contentType(MediaType.APPLICATION_JSON)
    .body(response);
```

---

## ขั้นตอนที่ 92: Exception Handling แบบ Global

```java
// src/main/java/com/example/myapp/exception/GlobalExceptionHandler.java
@RestControllerAdvice
@Slf4j
public class GlobalExceptionHandler {
    
    // Resource not found
    @ExceptionHandler(ResourceNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public ErrorResponse handleNotFound(ResourceNotFoundException ex, HttpServletRequest request) {
        log.warn("Resource not found: {}", ex.getMessage());
        return ErrorResponse.of(404, "NOT_FOUND", ex.getMessage(), request.getRequestURI());
    }
    
    // Duplicate resource
    @ExceptionHandler(DuplicateResourceException.class)
    @ResponseStatus(HttpStatus.CONFLICT)
    public ErrorResponse handleDuplicate(DuplicateResourceException ex, HttpServletRequest request) {
        return ErrorResponse.of(409, "CONFLICT", ex.getMessage(), request.getRequestURI());
    }
    
    // Validation errors
    @ExceptionHandler(MethodArgumentNotValidException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ErrorResponse handleValidation(
        MethodArgumentNotValidException ex, HttpServletRequest request
    ) {
        Map<String, String> errors = new LinkedHashMap<>();
        ex.getBindingResult().getFieldErrors().forEach(error ->
            errors.put(error.getField(), error.getDefaultMessage())
        );
        
        return ErrorResponse.builder()
            .status(400)
            .error("VALIDATION_FAILED")
            .message("Input validation failed")
            .path(request.getRequestURI())
            .fieldErrors(errors)
            .timestamp(LocalDateTime.now())
            .build();
    }
    
    // Type mismatch (ส่งตัวเลขผิดประเภท)
    @ExceptionHandler(MethodArgumentTypeMismatchException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ErrorResponse handleTypeMismatch(
        MethodArgumentTypeMismatchException ex, HttpServletRequest request
    ) {
        String message = String.format("Parameter '%s' should be of type %s",
            ex.getName(), ex.getRequiredType().getSimpleName());
        return ErrorResponse.of(400, "TYPE_MISMATCH", message, request.getRequestURI());
    }
    
    // Access denied (403)
    @ExceptionHandler(AccessDeniedException.class)
    @ResponseStatus(HttpStatus.FORBIDDEN)
    public ErrorResponse handleAccessDenied(AccessDeniedException ex, HttpServletRequest request) {
        return ErrorResponse.of(403, "FORBIDDEN", "Access denied", request.getRequestURI());
    }
    
    // Default handler
    @ExceptionHandler(Exception.class)
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    public ErrorResponse handleGeneral(Exception ex, HttpServletRequest request) {
        log.error("Unexpected error on {}: {}", request.getRequestURI(), ex.getMessage(), ex);
        return ErrorResponse.of(500, "INTERNAL_ERROR", "An unexpected error occurred", 
                               request.getRequestURI());
    }
}
```

---

## ขั้นตอนที่ 93: Request Logging Filter

```java
// src/main/java/com/example/myapp/filter/RequestLoggingFilter.java
package com.example.myapp.filter;

import jakarta.servlet.*;
import jakarta.servlet.http.*;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Component;
import org.springframework.web.filter.OncePerRequestFilter;

import java.io.IOException;
import java.util.UUID;

@Component
@Slf4j
public class RequestLoggingFilter extends OncePerRequestFilter {
    
    @Override
    protected void doFilterInternal(
        HttpServletRequest request, 
        HttpServletResponse response, 
        FilterChain filterChain
    ) throws ServletException, IOException {
        
        String requestId = UUID.randomUUID().toString().substring(0, 8);
        long startTime = System.currentTimeMillis();
        
        // Log incoming request
        log.info("[{}] --> {} {} from {}",
            requestId,
            request.getMethod(),
            request.getRequestURI(),
            request.getRemoteAddr()
        );
        
        // เพิ่ม X-Request-ID header
        response.addHeader("X-Request-ID", requestId);
        
        try {
            filterChain.doFilter(request, response);
        } finally {
            long duration = System.currentTimeMillis() - startTime;
            
            // Log response
            log.info("[{}] <-- {} {} {}ms",
                requestId,
                response.getStatus(),
                request.getRequestURI(),
                duration
            );
        }
    }
    
    @Override
    protected boolean shouldNotFilter(HttpServletRequest request) {
        // ไม่ log health check endpoints
        String path = request.getRequestURI();
        return path.startsWith("/actuator");
    }
}
```

---

## ขั้นตอนที่ 94: Rate Limiting พื้นฐาน

```java
// สร้าง Rate Limiter ง่ายๆ ด้วย Guava
// เพิ่ม dependency:
// <dependency>
//     <groupId>com.google.guava</groupId>
//     <artifactId>guava</artifactId>
//     <version>32.1.3-jre</version>
// </dependency>

@Component
public class RateLimitFilter extends OncePerRequestFilter {
    
    // เก็บ rate limiter แยกตาม IP
    private final Map<String, RateLimiter> limiters = new ConcurrentHashMap<>();
    
    // 10 requests per second per IP
    private static final double RATE = 10.0;
    
    @Override
    protected void doFilterInternal(
        HttpServletRequest request,
        HttpServletResponse response,
        FilterChain chain
    ) throws IOException, ServletException {
        
        String clientIp = getClientIp(request);
        
        RateLimiter limiter = limiters.computeIfAbsent(
            clientIp,
            k -> RateLimiter.create(RATE)
        );
        
        if (!limiter.tryAcquire()) {
            response.setStatus(429);  // Too Many Requests
            response.setContentType("application/json");
            response.getWriter().write("""
                {"error": "Too many requests", "message": "Rate limit exceeded"}
                """);
            return;
        }
        
        chain.doFilter(request, response);
    }
    
    private String getClientIp(HttpServletRequest request) {
        String xForwardedFor = request.getHeader("X-Forwarded-For");
        if (xForwardedFor != null && !xForwardedFor.isEmpty()) {
            return xForwardedFor.split(",")[0].trim();
        }
        return request.getRemoteAddr();
    }
}
```

---

## ขั้นตอนที่ 95: HATEOAS (Hypermedia)

```xml
<!-- Spring HATEOAS -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-hateoas</artifactId>
</dependency>
```

```java
import org.springframework.hateoas.EntityModel;
import org.springframework.hateoas.server.mvc.WebMvcLinkBuilder;
import static org.springframework.hateoas.server.mvc.WebMvcLinkBuilder.*;

@RestController
@RequestMapping("/api/v1/users")
public class UserController {
    
    @GetMapping("/{id}")
    public EntityModel<UserResponse> getUser(@PathVariable Long id) {
        UserResponse user = userService.findById(id);
        
        return EntityModel.of(user,
            linkTo(methodOn(UserController.class).getUser(id)).withSelfRel(),
            linkTo(methodOn(UserController.class).getAllUsers(0, 10, "id", "asc")).withRel("users"),
            linkTo(methodOn(OrderController.class).getUserOrders(id)).withRel("orders")
        );
    }
}
```

```json
// Response with HATEOAS links
{
  "id": 1,
  "username": "john",
  "email": "john@example.com",
  "_links": {
    "self": {
      "href": "http://localhost:8080/api/v1/users/1"
    },
    "users": {
      "href": "http://localhost:8080/api/v1/users?page=0&size=10&sortBy=id&sortDir=asc"
    },
    "orders": {
      "href": "http://localhost:8080/api/v1/orders/user/1"
    }
  }
}
```

---

## ขั้นตอนที่ 96: Request/Response Interceptor

```java
// src/main/java/com/example/myapp/interceptor/AuthInterceptor.java
@Component
@RequiredArgsConstructor
public class AuthInterceptor implements HandlerInterceptor {
    
    private final JwtService jwtService;
    
    @Override
    public boolean preHandle(
        HttpServletRequest request,
        HttpServletResponse response,
        Object handler
    ) throws Exception {
        
        String path = request.getRequestURI();
        
        // Skip auth for public endpoints
        if (path.startsWith("/api/auth") || path.startsWith("/actuator")) {
            return true;
        }
        
        String token = request.getHeader("Authorization");
        
        if (token == null || !token.startsWith("Bearer ")) {
            response.setStatus(401);
            response.getWriter().write("Unauthorized");
            return false;  // Block request
        }
        
        // Validate token
        try {
            String jwt = token.substring(7);
            jwtService.validateToken(jwt);
            return true;  // Allow request
        } catch (Exception e) {
            response.setStatus(401);
            response.getWriter().write("Invalid token");
            return false;
        }
    }
    
    @Override
    public void postHandle(
        HttpServletRequest request,
        HttpServletResponse response,
        Object handler,
        ModelAndView modelAndView
    ) {
        // ทำงานหลัง handler แต่ก่อน render
    }
    
    @Override
    public void afterCompletion(
        HttpServletRequest request,
        HttpServletResponse response,
        Object handler,
        Exception ex
    ) {
        // ทำงานหลังจาก response ส่งไปยัง client แล้ว
        // ใช้สำหรับ cleanup
    }
}
```

```java
// Register interceptor
@Configuration
public class WebConfig implements WebMvcConfigurer {
    
    @Autowired
    private AuthInterceptor authInterceptor;
    
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(authInterceptor)
            .addPathPatterns("/api/**")
            .excludePathPatterns("/api/auth/**", "/actuator/**");
    }
}
```

---

## ขั้นตอนที่ 97-100: ตัวอย่าง Complete REST API

### Product API สมบูรณ์

```java
// Entity
@Entity
@Table(name = "products")
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class Product extends BaseEntity {
    
    @Column(nullable = false, length = 200)
    private String name;
    
    @Column(columnDefinition = "TEXT")
    private String description;
    
    @Column(nullable = false, precision = 12, scale = 2)
    private BigDecimal price;
    
    @Column(nullable = false)
    @Builder.Default
    private Integer stock = 0;
    
    @Column(length = 100)
    private String category;
    
    @Column(name = "image_url")
    private String imageUrl;
    
    @Column(nullable = false)
    @Builder.Default
    private boolean active = true;
}

// Request DTOs
public record CreateProductRequest(
    @NotBlank @Size(max = 200) String name,
    String description,
    @NotNull @DecimalMin("0") BigDecimal price,
    @NotNull @Min(0) Integer stock,
    @NotBlank String category,
    String imageUrl
) {}

public record UpdateProductRequest(
    @Size(max = 200) String name,
    String description,
    @DecimalMin("0") BigDecimal price,
    @Min(0) Integer stock,
    String category,
    String imageUrl
) {}

// Response DTO
public record ProductResponse(
    Long id,
    String name,
    String description,
    BigDecimal price,
    Integer stock,
    String category,
    String imageUrl,
    boolean active,
    @JsonProperty("created_at") LocalDateTime createdAt,
    @JsonProperty("updated_at") LocalDateTime updatedAt
) {}

// Repository
@Repository
public interface ProductRepository extends JpaRepository<Product, Long>,
    JpaSpecificationExecutor<Product> {
    
    List<Product> findByCategoryAndActiveTrue(String category);
    Page<Product> findByActiveTrue(Pageable pageable);
    boolean existsByNameAndActiveTrue(String name);
    
    @Query("SELECT p FROM Product p WHERE p.stock <= :threshold AND p.active = true")
    List<Product> findLowStockProducts(@Param("threshold") int threshold);
}

// Service
@Service
@RequiredArgsConstructor
@Slf4j
@Transactional(readOnly = true)
public class ProductService {
    
    private final ProductRepository productRepository;
    private final ProductMapper productMapper;
    
    public Page<ProductResponse> findAll(Pageable pageable) {
        return productRepository.findByActiveTrue(pageable)
            .map(productMapper::toResponse);
    }
    
    public ProductResponse findById(Long id) {
        return productRepository.findById(id)
            .filter(Product::isActive)
            .map(productMapper::toResponse)
            .orElseThrow(() -> new ResourceNotFoundException("Product", "id", id));
    }
    
    @Transactional
    public ProductResponse create(CreateProductRequest request) {
        if (productRepository.existsByNameAndActiveTrue(request.name())) {
            throw new DuplicateResourceException("Product", "name", request.name());
        }
        
        Product product = productMapper.toEntity(request);
        Product saved = productRepository.save(product);
        
        log.info("Product created: id={}, name={}", saved.getId(), saved.getName());
        return productMapper.toResponse(saved);
    }
    
    @Transactional
    public ProductResponse update(Long id, UpdateProductRequest request) {
        Product product = productRepository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("Product", "id", id));
        
        productMapper.updateFromRequest(request, product);
        return productMapper.toResponse(productRepository.save(product));
    }
    
    @Transactional
    public void delete(Long id) {
        Product product = productRepository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("Product", "id", id));
        
        product.setActive(false);
        productRepository.save(product);
        log.info("Product soft-deleted: id={}", id);
    }
    
    @Transactional
    public ProductResponse updateStock(Long id, int quantity) {
        Product product = productRepository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("Product", "id", id));
        
        int newStock = product.getStock() + quantity;
        if (newStock < 0) {
            throw new AppException(HttpStatus.BAD_REQUEST, "INSUFFICIENT_STOCK",
                "Insufficient stock. Current: " + product.getStock());
        }
        
        product.setStock(newStock);
        return productMapper.toResponse(productRepository.save(product));
    }
}
```

---

## ขั้นตอนที่ 101-110: ทดสอบ API ด้วย Postman

### Postman Collection สำหรับ Product API

```json
{
  "info": {
    "name": "Spring Boot Course - Product API",
    "schema": "https://schema.getpostman.com/json/collection/v2.1.0/collection.json"
  },
  "variable": [
    {"key": "baseUrl", "value": "http://localhost:8080/api/v1"},
    {"key": "token", "value": ""}
  ],
  "item": [
    {
      "name": "Products",
      "item": [
        {
          "name": "Get All Products",
          "request": {
            "method": "GET",
            "url": "{{baseUrl}}/products?page=0&size=10&sortBy=createdAt&direction=desc"
          }
        },
        {
          "name": "Create Product",
          "request": {
            "method": "POST",
            "url": "{{baseUrl}}/products",
            "header": [{"key": "Content-Type", "value": "application/json"}],
            "body": {
              "raw": "{\n  \"name\": \"Test Product\",\n  \"description\": \"Test description\",\n  \"price\": 999.99,\n  \"stock\": 100,\n  \"category\": \"Electronics\"\n}"
            }
          }
        }
      ]
    }
  ]
}
```

### สรุป Part 05

```
✅ REST principles และ HTTP methods
✅ Complete CRUD Controller
✅ Request parameters (path, query, body, header)
✅ Validation (built-in + custom)
✅ Pagination และ Sorting
✅ Content negotiation
✅ Filtering ด้วย Specification
✅ File upload/download
✅ API versioning
✅ Exception handling
✅ Request logging filter
✅ Rate limiting
✅ HATEOAS
✅ Interceptors
✅ Complete Product API
```

---

*[← Part 04: โครงสร้างโปรเจค](./part-04-project-structure.md) | [Part 06: Dependency Injection →](./part-06-dependency-injection.md)*

# Part 55: API Versioning Strategies
## ขั้นตอนที่ 1801-1840

**ระดับ:** Advanced  
**เวลาเรียน:** 3-4 ชั่วโมง  
**เป้าหมาย:** เรียนรู้กลยุทธ์การทำ API Versioning ทุกแบบ, implement URL versioning, สร้าง @ApiVersion annotation, เขียน multi-version Swagger docs และทดสอบ backward compatibility

---

## ขั้นตอนที่ 1801: ทำไมต้องทำ API Versioning?

เมื่อ API เปลี่ยนแปลง client ที่ใช้ version เดิมอาจพัง API Versioning ช่วยให้:
- **ทำ breaking changes** โดยไม่กระทบ client เดิม
- **Client อัปเกรดได้ตามเวลาของตัวเอง**
- **Deprecate endpoint เก่า** อย่างเป็นระบบ
- **รองรับหลาย version** พร้อมกัน

### 4 กลยุทธ์หลัก

```
1. URL Path Versioning (แนะนำสำหรับ REST APIs)
   GET /api/v1/products
   GET /api/v2/products

2. Header Versioning
   GET /api/products
   X-API-Version: 2

3. Query Parameter Versioning
   GET /api/products?version=2

4. Content-Type Versioning (Media Type)
   Accept: application/vnd.myapp.v2+json
```

---

## ขั้นตอนที่ 1802: URL Path Versioning

วิธีที่ใช้บ่อยที่สุดและ clear ที่สุด:

```java
// V1 Controller
// src/main/java/com/example/controller/v1/ProductControllerV1.java
package com.example.controller.v1;

import com.example.dto.v1.ProductResponseV1;
import com.example.service.ProductService;
import io.swagger.v3.oas.annotations.tags.Tag;
import lombok.RequiredArgsConstructor;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/v1/products")
@RequiredArgsConstructor
@Tag(name = "Products V1", description = "Product API - Version 1")
public class ProductControllerV1 {
    
    private final ProductService productService;
    
    @GetMapping
    public ResponseEntity<Page<ProductResponseV1>> getProducts(Pageable pageable) {
        return ResponseEntity.ok(productService.getProductsV1(pageable));
    }
    
    @GetMapping("/{id}")
    public ResponseEntity<ProductResponseV1> getProduct(@PathVariable Long id) {
        return ResponseEntity.ok(productService.getProductByIdV1(id));
    }
}
```

```java
// V2 Controller - มีฟีเจอร์เพิ่มเติม
// src/main/java/com/example/controller/v2/ProductControllerV2.java
package com.example.controller.v2;

import com.example.dto.v2.ProductResponseV2;
import com.example.service.ProductService;
import io.swagger.v3.oas.annotations.tags.Tag;
import lombok.RequiredArgsConstructor;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.math.BigDecimal;

@RestController
@RequestMapping("/api/v2/products")
@RequiredArgsConstructor
@Tag(name = "Products V2", description = "Product API - Version 2 (ปรับปรุงจาก V1)")
public class ProductControllerV2 {
    
    private final ProductService productService;
    
    @GetMapping
    public ResponseEntity<Page<ProductResponseV2>> getProducts(
            @RequestParam(required = false) String keyword,
            @RequestParam(required = false) Long categoryId,
            @RequestParam(required = false) BigDecimal minPrice,
            @RequestParam(required = false) BigDecimal maxPrice,
            Pageable pageable) {
        // V2 รองรับ filter ที่ละเอียดกว่า
        return ResponseEntity.ok(
            productService.getProductsV2(keyword, categoryId, minPrice, maxPrice, pageable)
        );
    }
    
    @GetMapping("/{id}")
    public ResponseEntity<ProductResponseV2> getProduct(@PathVariable Long id) {
        // V2 return ข้อมูลเพิ่มเติม (reviews, recommendations)
        return ResponseEntity.ok(productService.getProductByIdV2(id));
    }
    
    @GetMapping("/{id}/similar")
    public ResponseEntity<Page<ProductResponseV2>> getSimilarProducts(
            @PathVariable Long id, Pageable pageable) {
        // endpoint ใหม่ใน V2
        return ResponseEntity.ok(productService.getSimilarProducts(id, pageable));
    }
}
```

---

## ขั้นตอนที่ 1803: DTO Evolution

```java
// V1 Response DTO - โครงสร้างแบบเดิม
// src/main/java/com/example/dto/v1/ProductResponseV1.java
package com.example.dto.v1;

import lombok.Data;
import java.math.BigDecimal;
import java.time.LocalDateTime;

@Data
public class ProductResponseV1 {
    private Long id;
    private String name;
    private String description;
    private BigDecimal price;
    private Integer stockQuantity;
    private String category;       // V1: แค่ชื่อ category เป็น String
    private LocalDateTime createdAt;
}
```

```java
// V2 Response DTO - ปรับปรุงและเพิ่มข้อมูล
// src/main/java/com/example/dto/v2/ProductResponseV2.java
package com.example.dto.v2;

import lombok.Data;
import java.math.BigDecimal;
import java.time.LocalDateTime;

@Data
public class ProductResponseV2 {
    private Long id;
    private String name;
    private String description;
    private String sku;            // V2: เพิ่ม SKU
    private BigDecimal price;
    private BigDecimal salePrice;  // V2: เพิ่ม sale price
    private BigDecimal effectivePrice; // V2: ราคาจริงที่ต้องจ่าย
    private Integer stockQuantity;
    private boolean inStock;       // V2: เปลี่ยนจาก Integer เป็น boolean flag
    private CategoryInfo category; // V2: เปลี่ยนเป็น object แทน string
    private String imageUrl;       // V2: เพิ่ม image
    private Double averageRating;  // V2: เพิ่ม rating
    private Integer reviewCount;   // V2: เพิ่ม review count
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt; // V2: เพิ่ม updatedAt
    
    @Data
    public static class CategoryInfo {
        private Long id;
        private String name;
        private String slug;
    }
}
```

---

## ขั้นตอนที่ 1804: Custom @ApiVersion Annotation

```java
// src/main/java/com/example/versioning/ApiVersion.java
package com.example.versioning;

import org.springframework.web.bind.annotation.RequestMapping;

import java.lang.annotation.*;

/**
 * Annotation สำหรับระบุ API Version
 * ใช้แทน hard-code version ใน @RequestMapping
 */
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@RequestMapping
public @interface ApiVersion {
    int[] value();  // version numbers ที่ endpoint นี้รองรับ
}
```

```java
// src/main/java/com/example/versioning/ApiVersionRequestMappingHandlerMapping.java
package com.example.versioning;

import org.springframework.core.annotation.AnnotationUtils;
import org.springframework.web.servlet.mvc.method.RequestMappingInfo;
import org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping;

import java.lang.reflect.Method;
import java.util.Arrays;
import java.util.stream.Collectors;

/**
 * Custom HandlerMapping ที่สร้าง path จาก @ApiVersion annotation
 */
public class ApiVersionRequestMappingHandlerMapping 
        extends RequestMappingHandlerMapping {
    
    private final String prefix;
    
    public ApiVersionRequestMappingHandlerMapping(String prefix) {
        this.prefix = prefix;
    }
    
    @Override
    protected RequestMappingInfo getMappingForMethod(Method method, Class<?> handlerType) {
        RequestMappingInfo info = super.getMappingForMethod(method, handlerType);
        if (info == null) return null;
        
        ApiVersion versionAnnotation = AnnotationUtils.findAnnotation(handlerType, ApiVersion.class);
        if (versionAnnotation == null) {
            versionAnnotation = AnnotationUtils.findAnnotation(method, ApiVersion.class);
        }
        
        if (versionAnnotation != null) {
            RequestMappingInfo versionedInfo = createVersionedMappingInfo(versionAnnotation);
            info = versionedInfo.combine(info);
        }
        
        return info;
    }
    
    private RequestMappingInfo createVersionedMappingInfo(ApiVersion version) {
        String[] versionPaths = Arrays.stream(version.value())
            .mapToObj(v -> prefix + "/v" + v)
            .toArray(String[]::new);
        
        return RequestMappingInfo.paths(versionPaths).build();
    }
}
```

```java
// src/main/java/com/example/config/VersioningWebMvcConfig.java
package com.example.config;

import com.example.versioning.ApiVersionRequestMappingHandlerMapping;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurationSupport;
import org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping;

@Configuration
public class VersioningWebMvcConfig extends WebMvcConfigurationSupport {
    
    @Override
    protected RequestMappingHandlerMapping createRequestMappingHandlerMapping() {
        return new ApiVersionRequestMappingHandlerMapping("/api");
    }
}
```

### ใช้งาน @ApiVersion

```java
// src/main/java/com/example/controller/UserController.java
package com.example.controller;

import com.example.versioning.ApiVersion;
import lombok.RequiredArgsConstructor;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

// รองรับทั้ง v1 และ v2
@RestController
@ApiVersion({1, 2})
@RequestMapping("/users")
@RequiredArgsConstructor
public class UserController {
    
    @GetMapping("/{id}")
    public ResponseEntity<UserResponse> getUser(@PathVariable Long id) {
        // Path จะเป็น: /api/v1/users/{id} และ /api/v2/users/{id}
        return ResponseEntity.ok(userService.getUser(id));
    }
}
```

```java
// เฉพาะ v3 ขึ้นไป (method-level annotation)
@RestController
@ApiVersion({1, 2, 3})
@RequestMapping("/products")
public class ProductController {
    
    @GetMapping("/{id}")
    public ResponseEntity<ProductResponseV2> getProduct(@PathVariable Long id) {
        return ResponseEntity.ok(productService.getProduct(id));
    }
    
    // Endpoint นี้มีเฉพาะใน v3
    @GetMapping("/{id}/analytics")
    @ApiVersion(3)
    public ResponseEntity<ProductAnalytics> getAnalytics(@PathVariable Long id) {
        return ResponseEntity.ok(analyticsService.getProductAnalytics(id));
    }
}
```

---

## ขั้นตอนที่ 1805: Header-Based Versioning

```java
// src/main/java/com/example/versioning/HeaderVersionController.java
package com.example.versioning;

import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

/**
 * Header-based versioning
 * Client ส่ง header: X-API-Version: 2
 */
@RestController
@RequestMapping("/api/products")
public class HeaderVersionController {
    
    // Default หรือ V1
    @GetMapping(headers = "!X-API-Version")
    public ResponseEntity<ProductResponseV1> getProductsDefault() {
        return ResponseEntity.ok(productService.getProductsV1());
    }
    
    @GetMapping(headers = "X-API-Version=1")
    public ResponseEntity<ProductResponseV1> getProductsV1() {
        return ResponseEntity.ok(productService.getProductsV1());
    }
    
    @GetMapping(headers = "X-API-Version=2")
    public ResponseEntity<ProductResponseV2> getProductsV2() {
        return ResponseEntity.ok(productService.getProductsV2());
    }
}
```

---

## ขั้นตอนที่ 1806: Content-Type Versioning

```java
// src/main/java/com/example/versioning/ContentTypeVersionController.java
package com.example.versioning;

import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

/**
 * Content-Type versioning
 * Client ส่ง header: Accept: application/vnd.myapp.v2+json
 */
@RestController
@RequestMapping("/api/products")
public class ContentTypeVersionController {
    
    @GetMapping(produces = "application/vnd.myapp.v1+json")
    public ResponseEntity<ProductResponseV1> getProductsV1() {
        return ResponseEntity.ok(productService.getProductsV1());
    }
    
    @GetMapping(produces = "application/vnd.myapp.v2+json")
    public ResponseEntity<ProductResponseV2> getProductsV2() {
        return ResponseEntity.ok(productService.getProductsV2());
    }
    
    // Fallback สำหรับ standard JSON
    @GetMapping(produces = "application/json")
    public ResponseEntity<ProductResponseV2> getProductsDefault() {
        // default ไปที่ version ล่าสุด
        return ResponseEntity.ok(productService.getProductsV2());
    }
}
```

---

## ขั้นตอนที่ 1807: API Evolution - Adding Fields

การเพิ่ม field ใหม่เป็น backward compatible (ถ้า client ignore unknown fields):

```java
// V1 DTO - OriginalResponse
@Data
public class ProductResponseV1 {
    private Long id;
    private String name;
    private BigDecimal price;
}

// V1.1 DTO - เพิ่ม field โดยมี default value (backward compatible)
@Data
@JsonInclude(JsonInclude.Include.NON_NULL)  // ไม่ส่ง null fields
public class ProductResponseV1 {
    private Long id;
    private String name;
    private BigDecimal price;
    
    // Field ใหม่ใน V1.1 - client เดิมจะ ignore ได้
    @JsonProperty("stock_quantity")
    private Integer stockQuantity;  // null สำหรับ legacy data
    
    @JsonProperty("image_url")
    private String imageUrl;  // optional
}
```

```java
// Jackson Config สำหรับ backward compatibility
@Configuration
public class JacksonConfig {
    
    @Bean
    public ObjectMapper objectMapper() {
        ObjectMapper mapper = new ObjectMapper();
        
        // ไม่ fail เมื่อ client ส่ง field ที่ server ไม่รู้จัก
        mapper.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);
        
        // ไม่ fail เมื่อ client ไม่ส่ง field ที่ server expect
        mapper.configure(DeserializationFeature.FAIL_ON_NULL_FOR_PRIMITIVES, false);
        
        // ไม่ส่ง null fields
        mapper.setSerializationInclusion(JsonInclude.Include.NON_NULL);
        
        // Format dates as ISO 8601
        mapper.configure(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS, false);
        mapper.registerModule(new JavaTimeModule());
        
        return mapper;
    }
}
```

---

## ขั้นตอนที่ 1808: Deprecating Endpoints

```java
// src/main/java/com/example/controller/v1/DeprecatedProductController.java
package com.example.controller.v1;

import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.extensions.Extension;
import io.swagger.v3.oas.annotations.extensions.ExtensionProperty;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

/**
 * V1 controller ที่ถูก deprecated
 * จะถูกลบในวันที่ 2025-06-01
 */
@RestController
@RequestMapping("/api/v1/products")
@RequiredArgsConstructor
@Slf4j
public class DeprecatedProductController {
    
    @GetMapping
    @Deprecated  // Java deprecation
    @Operation(
        deprecated = true,  // OpenAPI deprecation
        summary = "[DEPRECATED] Get products - ใช้ /api/v2/products แทน",
        description = "Endpoint นี้จะถูกลบใน 2025-06-01 กรุณาเปลี่ยนไปใช้ V2"
    )
    public ResponseEntity<Object> getProducts() {
        // Log warning เมื่อมีการใช้ deprecated endpoint
        log.warn("DEPRECATED endpoint /api/v1/products called. " +
                 "Please migrate to /api/v2/products. " +
                 "Removal date: 2025-06-01");
        
        // เพิ่ม deprecation headers
        return ResponseEntity.ok()
            .header("Deprecation", "true")
            .header("Sunset", "Sat, 01 Jun 2025 00:00:00 GMT")
            .header("Link", "</api/v2/products>; rel=\"successor-version\"")
            .body(productService.getProductsV1());
    }
}
```

```java
// src/main/java/com/example/filter/DeprecationFilter.java
package com.example.filter;

import jakarta.servlet.*;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Component;

import java.io.IOException;
import java.util.Set;

/**
 * Filter ที่เพิ่ม deprecation headers สำหรับ v1 endpoints ทุกตัว
 */
@Component
@Slf4j
public class DeprecationFilter implements Filter {
    
    private static final Set<String> DEPRECATED_PATHS = Set.of("/api/v1/");
    private static final String SUNSET_DATE = "Sat, 01 Jun 2025 00:00:00 GMT";
    
    @Override
    public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain)
            throws IOException, ServletException {
        
        var request = (HttpServletRequest) req;
        var response = (HttpServletResponse) res;
        
        String path = request.getRequestURI();
        
        if (DEPRECATED_PATHS.stream().anyMatch(path::startsWith)) {
            response.addHeader("Deprecation", "true");
            response.addHeader("Sunset", SUNSET_DATE);
            
            // ระบุ successor
            String successorPath = path.replace("/api/v1/", "/api/v2/");
            response.addHeader("Link", 
                String.format("<%s>; rel=\"successor-version\"", successorPath));
            
            log.info("Deprecated endpoint accessed: {} by {}", 
                path, request.getRemoteAddr());
        }
        
        chain.doFilter(req, res);
    }
}
```

---

## ขั้นตอนที่ 1809: Swagger/OpenAPI Multi-Version Documentation

```java
// src/main/java/com/example/config/OpenApiConfig.java
package com.example.config;

import io.swagger.v3.oas.models.OpenAPI;
import io.swagger.v3.oas.models.info.Contact;
import io.swagger.v3.oas.models.info.Info;
import io.swagger.v3.oas.models.info.License;
import io.swagger.v3.oas.models.security.SecurityRequirement;
import io.swagger.v3.oas.models.security.SecurityScheme;
import io.swagger.v3.oas.models.Components;
import io.swagger.v3.oas.models.servers.Server;
import org.springdoc.core.models.GroupedOpenApi;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.util.List;

@Configuration
public class OpenApiConfig {
    
    /**
     * Group สำหรับ API V1
     */
    @Bean
    public GroupedOpenApi apiV1() {
        return GroupedOpenApi.builder()
            .group("v1")
            .displayName("API Version 1 (Deprecated)")
            .pathsToMatch("/api/v1/**")
            .addOpenApiCustomizer(openApi -> {
                openApi.info(new Info()
                    .title("E-Commerce API")
                    .version("1.0")
                    .description("**⚠️ DEPRECATED:** API Version 1 จะถูกลบใน 2025-06-01 " +
                                "กรุณา migrate ไป V2")
                );
            })
            .build();
    }
    
    /**
     * Group สำหรับ API V2 (Current)
     */
    @Bean
    public GroupedOpenApi apiV2() {
        return GroupedOpenApi.builder()
            .group("v2")
            .displayName("API Version 2 (Current)")
            .pathsToMatch("/api/v2/**")
            .addOpenApiCustomizer(openApi -> {
                openApi.info(new Info()
                    .title("E-Commerce API")
                    .version("2.0")
                    .description("API Version 2 - Stable Release\n\n" +
                                "**New in V2:**\n" +
                                "- Product filtering by price range\n" +
                                "- Product images\n" +
                                "- Similar products recommendation\n" +
                                "- Enhanced category information")
                    .contact(new Contact()
                        .name("API Team")
                        .email("api@example.com"))
                    .license(new License()
                        .name("MIT")
                        .url("https://opensource.org/licenses/MIT"))
                );
            })
            .build();
    }
    
    /**
     * OpenAPI config หลัก
     */
    @Bean
    public OpenAPI mainOpenApi() {
        return new OpenAPI()
            .info(new Info()
                .title("E-Commerce REST API")
                .version("2.0.0")
                .description("Full E-Commerce REST API with versioning support"))
            .servers(List.of(
                new Server().url("https://api.example.com").description("Production"),
                new Server().url("https://staging-api.example.com").description("Staging"),
                new Server().url("http://localhost:8080").description("Local Development")
            ))
            .components(new Components()
                .addSecuritySchemes("bearerAuth", new SecurityScheme()
                    .type(SecurityScheme.Type.HTTP)
                    .scheme("bearer")
                    .bearerFormat("JWT")
                    .description("Enter JWT token")))
            .addSecurityItem(new SecurityRequirement().addList("bearerAuth"));
    }
}
```

### application.yml สำหรับ Springdoc

```yaml
springdoc:
  api-docs:
    enabled: true
    path: /api-docs
  swagger-ui:
    enabled: true
    path: /swagger-ui.html
    # แสดง groups ทั้งหมด
    tags-sorter: alpha
    operations-sorter: alpha
    display-request-duration: true
    # เริ่มที่ V2
    urls-primary-name: v2
  # Path สำหรับ API docs แต่ละ version
  group-configs:
    - group: v1
      paths-to-match: /api/v1/**
    - group: v2
      paths-to-match: /api/v2/**
```

---

## ขั้นตอนที่ 1810: Version Router - Delegate Pattern

```java
// src/main/java/com/example/versioning/VersionRouter.java
package com.example.versioning;

import com.example.dto.v1.ProductResponseV1;
import com.example.dto.v2.ProductResponseV2;
import com.example.service.ProductService;
import lombok.RequiredArgsConstructor;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

/**
 * Router pattern: Controller เดียว delegate ไปตาม version
 * เหมาะสำหรับ business logic ที่ไม่ต่างกันมาก
 */
@RestController
@RequestMapping("/api/{version}/products")
@RequiredArgsConstructor
public class VersionedProductController {
    
    private final ProductService productService;
    private final ProductResponseMapper mapper;
    
    @GetMapping("/{id}")
    public ResponseEntity<?> getProduct(
            @PathVariable String version,
            @PathVariable Long id) {
        
        var product = productService.getProductById(id);
        
        return switch (version) {
            case "v1" -> ResponseEntity.ok(mapper.toV1Response(product));
            case "v2" -> ResponseEntity.ok(mapper.toV2Response(product));
            default -> ResponseEntity.badRequest()
                .body(Map.of("error", "Unsupported API version: " + version));
        };
    }
}
```

---

## ขั้นตอนที่ 1811: Backward Compatibility Testing

```java
// src/test/java/com/example/versioning/BackwardCompatibilityTest.java
package com.example.versioning;

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.test.web.servlet.MvcResult;

import static org.assertj.core.api.Assertions.assertThat;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;

@SpringBootTest
@AutoConfigureMockMvc
class BackwardCompatibilityTest {
    
    @Autowired
    private MockMvc mockMvc;
    
    @Autowired
    private ObjectMapper objectMapper;
    
    /**
     * ตรวจสอบว่า V1 response ยังมี fields ที่ client เดิมคาดหวัง
     */
    @Test
    void v1ResponseShouldContainExpectedFields() throws Exception {
        MvcResult result = mockMvc.perform(get("/api/v1/products/1"))
            .andExpect(status().isOk())
            .andReturn();
        
        JsonNode response = objectMapper.readTree(result.getResponse().getContentAsString());
        
        // fields ที่ V1 client ต้องการ
        assertThat(response.has("id")).isTrue();
        assertThat(response.has("name")).isTrue();
        assertThat(response.has("price")).isTrue();
        assertThat(response.has("category")).isTrue();
        
        // category ใน V1 ต้องเป็น String
        assertThat(response.get("category").isTextual()).isTrue();
    }
    
    /**
     * V2 ต้องมี fields เพิ่มขึ้นจาก V1
     */
    @Test
    void v2ResponseShouldHaveMoreFieldsThanV1() throws Exception {
        MvcResult v1Result = mockMvc.perform(get("/api/v1/products/1"))
            .andExpect(status().isOk())
            .andReturn();
        
        MvcResult v2Result = mockMvc.perform(get("/api/v2/products/1"))
            .andExpect(status().isOk())
            .andReturn();
        
        JsonNode v1Response = objectMapper.readTree(v1Result.getResponse().getContentAsString());
        JsonNode v2Response = objectMapper.readTree(v2Result.getResponse().getContentAsString());
        
        // V2 ต้องมีทุก field ของ V1
        v1Response.fieldNames().forEachRemaining(field -> {
            // ยกเว้น category ที่เปลี่ยน type
            if (!field.equals("category")) {
                assertThat(v2Response.has(field))
                    .withFailMessage("V2 missing field from V1: " + field)
                    .isTrue();
            }
        });
        
        // V2 ต้องมี fields ใหม่
        assertThat(v2Response.has("sku")).isTrue();
        assertThat(v2Response.has("imageUrl")).isTrue();
        assertThat(v2Response.has("effectivePrice")).isTrue();
        
        // category ใน V2 ต้องเป็น object
        assertThat(v2Response.get("category").isObject()).isTrue();
        assertThat(v2Response.get("category").has("id")).isTrue();
        assertThat(v2Response.get("category").has("name")).isTrue();
    }
    
    /**
     * ทดสอบ unknown fields ไม่ทำให้ V1 client พัง
     */
    @Test
    void v1ClientShouldIgnoreUnknownFieldsFromServer() throws Exception {
        // ถ้า server เพิ่ม field ใหม่ใน V1 response
        // V1 client (ที่ configure FAIL_ON_UNKNOWN_PROPERTIES=false) ต้องไม่พัง
        
        // สร้าง JSON ที่มี field พิเศษ
        String jsonWithExtraField = """
            {
                "id": 1,
                "name": "Product",
                "price": 100.00,
                "category": "Electronics",
                "newFieldAddedInV1_1": "some value"
            }
            """;
        
        // Client ที่ดีควร deserialize ได้โดยไม่มี error
        ProductResponseV1 response = objectMapper.readValue(jsonWithExtraField, 
            ProductResponseV1.class);
        
        assertThat(response.getId()).isEqualTo(1L);
        assertThat(response.getName()).isEqualTo("Product");
        // newFieldAddedInV1_1 จะถูก ignore
    }
}
```

---

## ขั้นตอนที่ 1812: Version Deprecation Strategy

```java
// src/main/java/com/example/versioning/VersionDeprecationService.java
package com.example.versioning;

import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;

import java.time.LocalDate;
import java.util.Map;

/**
 * Service สำหรับจัดการ lifecycle ของ API versions
 */
@Service
@Slf4j
public class VersionDeprecationService {
    
    // กำหนด lifecycle ของแต่ละ version
    private static final Map<String, VersionInfo> VERSION_LIFECYCLE = Map.of(
        "v1", new VersionInfo("v1", VersionStatus.DEPRECATED, 
            LocalDate.of(2024, 1, 1),   // deprecated since
            LocalDate.of(2025, 6, 1)),   // sunset date
        "v2", new VersionInfo("v2", VersionStatus.STABLE, 
            LocalDate.of(2024, 1, 1), null),
        "v3", new VersionInfo("v3", VersionStatus.BETA,
            LocalDate.of(2024, 12, 1), null)
    );
    
    public VersionInfo getVersionInfo(String version) {
        return VERSION_LIFECYCLE.getOrDefault(version, 
            new VersionInfo(version, VersionStatus.UNKNOWN, null, null));
    }
    
    public boolean isDeprecated(String version) {
        var info = VERSION_LIFECYCLE.get(version);
        return info != null && info.status() == VersionStatus.DEPRECATED;
    }
    
    public boolean isSunset(String version) {
        var info = VERSION_LIFECYCLE.get(version);
        return info != null && 
               info.sunsetDate() != null && 
               LocalDate.now().isAfter(info.sunsetDate());
    }
    
    public record VersionInfo(
        String version,
        VersionStatus status,
        LocalDate deprecatedSince,
        LocalDate sunsetDate
    ) {}
    
    public enum VersionStatus {
        STABLE, DEPRECATED, BETA, UNKNOWN
    }
}
```

```java
// src/main/java/com/example/versioning/VersionCheckInterceptor.java
package com.example.versioning;

import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Component;
import org.springframework.web.servlet.HandlerInterceptor;

import java.util.regex.Matcher;
import java.util.regex.Pattern;

@Component
@RequiredArgsConstructor
@Slf4j
public class VersionCheckInterceptor implements HandlerInterceptor {
    
    private final VersionDeprecationService deprecationService;
    private static final Pattern VERSION_PATTERN = Pattern.compile("/api/(v\\d+)/");
    
    @Override
    public boolean preHandle(HttpServletRequest request, 
                             HttpServletResponse response,
                             Object handler) throws Exception {
        
        String path = request.getRequestURI();
        Matcher matcher = VERSION_PATTERN.matcher(path);
        
        if (matcher.find()) {
            String version = matcher.group(1);
            
            // ถ้าหลัง sunset date ให้ return 410 Gone
            if (deprecationService.isSunset(version)) {
                response.setStatus(HttpServletResponse.SC_GONE);
                response.getWriter().write(String.format(
                    "{\"error\":\"API %s has been sunset. Please use /api/v2/\", " +
                    "\"migrationGuide\":\"https://docs.example.com/api/migration/%s-to-v2\"}",
                    version, version
                ));
                return false;
            }
            
            // ถ้า deprecated ให้เพิ่ม warning headers
            if (deprecationService.isDeprecated(version)) {
                var info = deprecationService.getVersionInfo(version);
                response.addHeader("Deprecation", "true");
                if (info.sunsetDate() != null) {
                    response.addHeader("Sunset", info.sunsetDate().toString());
                }
                response.addHeader("Link", 
                    "</api/v2/>; rel=\"successor-version\", " +
                    "<https://docs.example.com/api/migration>; rel=\"deprecation\"");
                
                log.warn("Deprecated API version accessed: {} by {}", 
                    version, request.getRemoteAddr());
            }
        }
        
        return true;
    }
}
```

---

## ขั้นตอนที่ 1813: API Version Analytics

```java
// src/main/java/com/example/versioning/VersionUsageMetrics.java
package com.example.versioning;

import io.micrometer.core.instrument.Counter;
import io.micrometer.core.instrument.MeterRegistry;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Component;
import org.springframework.web.servlet.HandlerInterceptor;

import java.util.regex.Matcher;
import java.util.regex.Pattern;

/**
 * ติดตาม metrics การใช้งานแต่ละ API version
 * ช่วยวางแผนว่าเมื่อไรจะ remove version เก่า
 */
@Component
@RequiredArgsConstructor
public class VersionUsageMetrics implements HandlerInterceptor {
    
    private final MeterRegistry meterRegistry;
    private static final Pattern VERSION_PATTERN = Pattern.compile("/api/(v\\d+)/");
    
    @Override
    public void afterCompletion(HttpServletRequest request,
                                 HttpServletResponse response,
                                 Object handler,
                                 Exception ex) {
        
        Matcher matcher = VERSION_PATTERN.matcher(request.getRequestURI());
        if (matcher.find()) {
            String version = matcher.group(1);
            String method = request.getMethod();
            String statusGroup = response.getStatus() / 100 + "xx";
            
            Counter.builder("api.version.requests")
                .tag("version", version)
                .tag("method", method)
                .tag("status", statusGroup)
                .description("API requests per version")
                .register(meterRegistry)
                .increment();
        }
    }
}
```

---

## ขั้นตอนที่ 1814: Migration Guide สำหรับ Clients

```java
// src/main/java/com/example/controller/MigrationGuideController.java
package com.example.controller;

import io.swagger.v3.oas.annotations.Hidden;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import java.util.Map;

/**
 * Endpoint ที่ให้ migration guide เมื่อ client ใช้ version เก่า
 */
@RestController
@RequestMapping("/api/migration")
@Hidden  // ซ่อนจาก Swagger
public class MigrationGuideController {
    
    @GetMapping("/{fromVersion}-to-{toVersion}")
    public Map<String, Object> getMigrationGuide(
            @PathVariable String fromVersion,
            @PathVariable String toVersion) {
        
        if ("v1".equals(fromVersion) && "v2".equals(toVersion)) {
            return Map.of(
                "breaking_changes", java.util.List.of(
                    Map.of(
                        "endpoint", "GET /api/v1/products",
                        "change", "Response field 'category' changed from String to Object",
                        "v1_format", "{\"category\": \"Electronics\"}",
                        "v2_format", "{\"category\": {\"id\": 1, \"name\": \"Electronics\"}}"
                    )
                ),
                "new_features", java.util.List.of(
                    "Product SKU field",
                    "Product images",
                    "Sale price",
                    "Stock availability flag",
                    "GET /api/v2/products/{id}/similar endpoint"
                ),
                "migration_steps", java.util.List.of(
                    "1. Update base URL from /api/v1/ to /api/v2/",
                    "2. Update 'category' field handling (String -> Object)",
                    "3. Test all endpoints",
                    "4. Remove deprecated V1 code"
                ),
                "sunset_date", "2025-06-01",
                "support_contact", "api-support@example.com"
            );
        }
        
        return Map.of("message", "Migration guide not available for " + 
            fromVersion + " to " + toVersion);
    }
}
```

---

## ขั้นตอนที่ 1815: Contract Testing

```java
// src/test/java/com/example/versioning/ApiContractTest.java
package com.example.versioning;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.web.servlet.MockMvc;

import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@SpringBootTest
@AutoConfigureMockMvc
class ApiContractTest {
    
    @Autowired
    private MockMvc mockMvc;
    
    // V1 Contract - ต้อง stable ตลอดไปจนถึง sunset date
    
    @Test
    void v1_getProducts_shouldReturnExpectedStructure() throws Exception {
        mockMvc.perform(get("/api/v1/products"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.content").isArray())
            .andExpect(jsonPath("$.content[0].id").exists())
            .andExpect(jsonPath("$.content[0].name").exists())
            .andExpect(jsonPath("$.content[0].price").isNumber())
            .andExpect(jsonPath("$.content[0].category").isString())  // V1: String
            .andExpect(jsonPath("$.totalElements").isNumber())
            .andExpect(jsonPath("$.totalPages").isNumber());
    }
    
    @Test
    void v1_shouldReturnDeprecationHeaders() throws Exception {
        mockMvc.perform(get("/api/v1/products"))
            .andExpect(status().isOk())
            .andExpect(header().exists("Deprecation"))
            .andExpect(header().exists("Sunset"))
            .andExpect(header().exists("Link"));
    }
    
    // V2 Contract
    
    @Test
    void v2_getProducts_shouldReturnEnhancedStructure() throws Exception {
        mockMvc.perform(get("/api/v2/products"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.content").isArray())
            .andExpect(jsonPath("$.content[0].id").exists())
            .andExpect(jsonPath("$.content[0].name").exists())
            .andExpect(jsonPath("$.content[0].sku").exists())         // V2 field
            .andExpect(jsonPath("$.content[0].effectivePrice").exists()) // V2 field
            .andExpect(jsonPath("$.content[0].category.id").isNumber())  // V2: Object
            .andExpect(jsonPath("$.content[0].category.name").isString()); // V2: Object
    }
    
    @Test
    void v2_shouldNotReturnDeprecationHeaders() throws Exception {
        mockMvc.perform(get("/api/v2/products"))
            .andExpect(status().isOk())
            .andExpect(header().doesNotExist("Deprecation"))
            .andExpect(header().doesNotExist("Sunset"));
    }
    
    @Test
    void v2_filterByPriceRange_shouldWork() throws Exception {
        mockMvc.perform(get("/api/v2/products")
                .param("minPrice", "100")
                .param("maxPrice", "1000"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.content").isArray());
    }
    
    @Test
    void unknownVersion_shouldReturn404() throws Exception {
        mockMvc.perform(get("/api/v99/products"))
            .andExpect(status().isNotFound());
    }
}
```

---

## ขั้นตอนที่ 1816: Version Selection Strategy สรุป

```java
// src/main/java/com/example/versioning/VersioningStrategy.java
package com.example.versioning;

/**
 * สรุปข้อดีข้อเสียของแต่ละ versioning strategy
 */
public class VersioningStrategy {
    
    /*
     * URL PATH VERSIONING (/api/v1/products)
     * ✅ ข้อดี:
     *   - Clear และ explicit
     *   - Easy to cache (different URLs)
     *   - Easy to test (ใช้ browser ได้)
     *   - Easy to document (Swagger groups by path)
     *   - Proxy/load balancer route ตาม version ได้
     *
     * ❌ ข้อเสีย:
     *   - URL ไม่ pure REST (resource identifier เปลี่ยน)
     *   - URL อาจยาว
     *
     * ✅ แนะนำสำหรับ: Public APIs, Mobile apps, Third-party integrations
     *
     * ---
     *
     * HEADER VERSIONING (X-API-Version: 2)
     * ✅ ข้อดี:
     *   - URL ไม่เปลี่ยน (pure REST)
     *   - ง่ายต่อการ switch version โดย client
     *
     * ❌ ข้อเสีย:
     *   - ไม่ cacheable ง่าย (ต้อง Vary header)
     *   - ทดสอบด้วย browser ยาก
     *   - Documentation ยุ่งยากกว่า
     *   - ลืมส่ง header ง่าย
     *
     * ✅ แนะนำสำหรับ: Internal APIs, Microservices
     *
     * ---
     *
     * QUERY PARAMETER (/api/products?version=2)
     * ✅ ข้อดี:
     *   - Explicit และ visible
     *   - Optional (มี default version)
     *
     * ❌ ข้อเสีย:
     *   - URL อาจ confusing
     *   - Caching อาจมีปัญหา
     *   - ดูไม่ professional
     *
     * ✅ แนะนำสำหรับ: Internal APIs ที่ไม่ formal
     *
     * ---
     *
     * CONTENT TYPE (Accept: application/vnd.app.v2+json)
     * ✅ ข้อดี:
     *   - Most RESTful approach
     *   - เหมาะกับ versioned representations
     *
     * ❌ ข้อเสีย:
     *   - Complex ที่สุด
     *   - Client ต้อง set header ถูกต้อง
     *   - Documentation ยาก
     *   - ใช้ browser ไม่ได้โดยตรง
     *
     * ✅ แนะนำสำหรับ: Advanced REST APIs, HATEOAS
     */
}
```

---

## สรุปท้ายบท

ในส่วนนี้เราได้เรียนรู้:

1. **4 กลยุทธ์ API Versioning** พร้อมข้อดีข้อเสีย
2. **URL Path Versioning** การ implement ที่ practical
3. **@ApiVersion Annotation** แบบ custom สำหรับ elegant code
4. **DTO Evolution** เพิ่ม field อย่าง backward compatible
5. **Deprecation Strategy** พร้อม sunset headers
6. **Swagger Multi-Version Docs** แยก documentation ต่าม version
7. **Contract Testing** ทดสอบ backward compatibility
8. **Version Analytics** ติดตามการใช้งาน

API Versioning ที่ดีเป็นส่วนสำคัญของ API design ที่ทำให้ team สามารถ evolve API ได้โดยไม่กระทบ client ที่มีอยู่

---

*[← Part 54: Virtual Threads](./part-54-virtual-threads.md) | [Part 56: Next Topic →](./part-56-next.md)*

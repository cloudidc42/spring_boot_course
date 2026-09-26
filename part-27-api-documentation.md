# Part 27: API Documentation - OpenAPI / Swagger
## ขั้นตอนที่ 736-760

> **ระดับ:** กลาง (Intermediate)  
> **เวลาเรียน:** 3-4 ชั่วโมง  
> **เป้าหมาย:** สร้าง API Documentation ที่ครบถ้วนและ interactive

---

## ขั้นตอนที่ 736: OpenAPI Dependencies

```xml
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
    <version>2.3.0</version>
</dependency>
```

---

## ขั้นตอนที่ 737: OpenAPI Configuration

```java
@Configuration
@OpenAPIDefinition
public class OpenApiConfig {
    
    @Bean
    public OpenAPI openAPI() {
        return new OpenAPI()
            .info(new Info()
                .title("Spring Boot Course API")
                .description("Complete E-Commerce REST API built with Spring Boot 3")
                .version("v1.0.0")
                .contact(new Contact()
                    .name("API Support")
                    .email("support@example.com")
                    .url("https://example.com"))
                .license(new License()
                    .name("MIT License")
                    .url("https://opensource.org/licenses/MIT")))
            
            .externalDocs(new ExternalDocumentation()
                .description("Full Documentation")
                .url("https://docs.example.com"))
            
            // Security scheme
            .addSecurityItem(new SecurityRequirement().addList("bearerAuth"))
            .components(new Components()
                .addSecuritySchemes("bearerAuth", 
                    new SecurityScheme()
                        .type(SecurityScheme.Type.HTTP)
                        .scheme("bearer")
                        .bearerFormat("JWT")
                        .description("Enter JWT token")))
            
            // Servers
            .addServersItem(new Server().url("http://localhost:8080").description("Development"))
            .addServersItem(new Server().url("https://api.example.com").description("Production"));
    }
    
    // Custom OpenAPI group
    @Bean
    public GroupedOpenApi publicApi() {
        return GroupedOpenApi.builder()
            .group("public")
            .pathsToMatch("/api/v1/products/**", "/api/v1/categories/**", "/api/v1/auth/**")
            .build();
    }
    
    @Bean
    public GroupedOpenApi adminApi() {
        return GroupedOpenApi.builder()
            .group("admin")
            .pathsToMatch("/api/v1/admin/**")
            .build();
    }
}
```

```yaml
# application.yml
springdoc:
  api-docs:
    path: /v3/api-docs
  swagger-ui:
    path: /swagger-ui.html
    operationsSorter: method
    tagsSorter: alpha
    tryItOutEnabled: true
    filter: true  # Search filter
  show-actuator: false
```

---

## ขั้นตอนที่ 738: Annotate Controllers

```java
@RestController
@RequestMapping("/api/v1/products")
@RequiredArgsConstructor
@Tag(name = "Products", description = "Product management API")
@SecurityRequirement(name = "bearerAuth")
public class ProductController {
    
    @GetMapping
    @Operation(
        summary = "Search products",
        description = "Search and filter products with pagination",
        parameters = {
            @Parameter(name = "keyword", description = "Search keyword"),
            @Parameter(name = "status", description = "Product status filter"),
            @Parameter(name = "page", description = "Page number (0-based)", example = "0"),
            @Parameter(name = "size", description = "Page size (max 100)", example = "20")
        }
    )
    @ApiResponses({
        @ApiResponse(responseCode = "200", description = "Success",
            content = @Content(schema = @Schema(implementation = ProductPageResponse.class))),
        @ApiResponse(responseCode = "400", description = "Invalid parameters",
            content = @Content(schema = @Schema(implementation = ErrorResponse.class)))
    })
    public ResponseEntity<ApiResponse<PageResponse<ProductResponse>>> search(
        @RequestParam(required = false) String keyword,
        @RequestParam(required = false) ProductStatus status,
        @RequestParam(defaultValue = "0") int page,
        @RequestParam(defaultValue = "20") int size
    ) { ... }
    
    @GetMapping("/{id}")
    @Operation(summary = "Get product by ID")
    @ApiResponse(responseCode = "404", description = "Product not found",
        content = @Content(schema = @Schema(implementation = ErrorResponse.class)))
    public ResponseEntity<ApiResponse<ProductDetailResponse>> findById(
        @Parameter(description = "Product ID", required = true, example = "1")
        @PathVariable Long id
    ) { ... }
    
    @PostMapping
    @Operation(summary = "Create product", description = "Admin only")
    @ApiResponse(responseCode = "201", description = "Product created")
    @ApiResponse(responseCode = "401", description = "Unauthorized")
    @ApiResponse(responseCode = "403", description = "Forbidden - Admin required")
    public ResponseEntity<ApiResponse<ProductResponse>> create(
        @io.swagger.v3.oas.annotations.parameters.RequestBody(
            description = "Product data",
            required = true,
            content = @Content(
                schema = @Schema(implementation = CreateProductRequest.class),
                examples = @ExampleObject(value = """
                    {
                      "name": "iPhone 15 Pro",
                      "price": 45000,
                      "stock": 100,
                      "categoryId": 1
                    }
                    """)
            )
        )
        @Valid @RequestBody CreateProductRequest request
    ) { ... }
}
```

---

## ขั้นตอนที่ 739: Annotate DTOs

```java
@Schema(description = "Request body for creating a product")
public record CreateProductRequest(
    
    @Schema(description = "Product name", example = "iPhone 15 Pro", minLength = 1, maxLength = 255)
    @NotBlank
    String name,
    
    @Schema(description = "Product description", example = "Latest Apple smartphone")
    String description,
    
    @Schema(description = "Product price in THB", example = "45000.00", minimum = "0.01")
    @NotNull @DecimalMin("0.01")
    BigDecimal price,
    
    @Schema(description = "Stock quantity", example = "100", minimum = "0")
    @NotNull @Min(0)
    Integer stock,
    
    @Schema(description = "Product image URL", example = "https://example.com/image.jpg")
    String imageUrl,
    
    @Schema(description = "Category ID", example = "1")
    Long categoryId
) {}

@Schema(description = "Product response")
public record ProductResponse(
    @Schema(description = "Product ID", example = "1")
    Long id,
    
    @Schema(description = "Product name", example = "iPhone 15 Pro")
    String name,
    
    @Schema(description = "Price in THB", example = "45000.00")
    BigDecimal price,
    
    @Schema(description = "Stock quantity", example = "100")
    Integer stock,
    
    @Schema(description = "Product status")
    ProductStatus status,
    
    @Schema(description = "Created timestamp")
    LocalDateTime createdAt
) {}
```

---

## ขั้นตอนที่ 740: Auth Controller Documentation

```java
@RestController
@RequestMapping("/api/v1/auth")
@Tag(name = "Authentication", description = "Authentication and authorization")
public class AuthController {
    
    @PostMapping("/login")
    @Operation(
        summary = "Login",
        description = "Authenticate with email and password, returns JWT tokens"
    )
    @ApiResponses({
        @ApiResponse(
            responseCode = "200",
            description = "Login successful",
            content = @Content(
                mediaType = "application/json",
                schema = @Schema(implementation = AuthResponseWrapper.class),
                examples = @ExampleObject(
                    name = "Success",
                    value = """
                        {
                          "success": true,
                          "data": {
                            "accessToken": "eyJhbGciOiJIUzI1NiJ9...",
                            "refreshToken": "550e8400-e29b-41d4-a716-446655440000",
                            "tokenType": "Bearer",
                            "expiresAt": 1735689600000
                          }
                        }
                        """
                )
            )
        ),
        @ApiResponse(responseCode = "401", description = "Invalid credentials"),
        @ApiResponse(responseCode = "429", description = "Too many login attempts")
    })
    public ResponseEntity<ApiResponse<AuthResponse>> login(
        @Valid @RequestBody LoginRequest request
    ) { ... }
}
```

---

## ขั้นตอนที่ 741: Security in Swagger UI

```java
// Hide endpoints from Swagger
@Hidden
@GetMapping("/internal/health")
public String internalHealth() { return "ok"; }

// Mark as deprecated
@Deprecated
@GetMapping("/v1/old-endpoint")
@Operation(summary = "Old endpoint", deprecated = true)
public ResponseEntity<?> oldEndpoint() { ... }

// Custom response examples
@GetMapping("/{id}")
@Operation(
    responses = {
        @ApiResponse(
            responseCode = "200",
            content = @Content(
                examples = {
                    @ExampleObject(name = "Active product", value = """
                        {"id": 1, "name": "iPhone", "status": "ACTIVE"}
                        """),
                    @ExampleObject(name = "Out of stock", value = """
                        {"id": 2, "name": "Old Phone", "status": "OUT_OF_STOCK"}
                        """)
                }
            )
        )
    }
)
public ResponseEntity<?> findById(@PathVariable Long id) { ... }
```

---

## ขั้นตอนที่ 742-760: API Documentation Best Practices

```java
// Generate OpenAPI spec as static file
// GET http://localhost:8080/v3/api-docs
// GET http://localhost:8080/v3/api-docs.yaml

// Export spec with Maven
// mvn springdoc-openapi:generate

// application.yml - customize URLs
springdoc:
  api-docs:
    path: /openapi/docs
  swagger-ui:
    path: /openapi/swagger-ui.html
    
// Disable in production
@Profile("!prod")
@Configuration
public class OpenApiConfig { ... }
```

### Swagger UI URLs

```
http://localhost:8080/swagger-ui.html     ← Interactive UI
http://localhost:8080/v3/api-docs         ← JSON spec
http://localhost:8080/v3/api-docs.yaml    ← YAML spec
```

### Documentation Checklist

```
✅ All endpoints documented
✅ Request/Response schemas complete
✅ Example values provided
✅ Error responses documented
✅ Authentication requirements noted
✅ Deprecated endpoints marked
✅ Tags organized logically
✅ Descriptions are helpful and accurate
✅ API versioning documented
✅ Rate limits documented
```

---

*[← Part 26: Message Queue](./part-26-message-queue.md) | [Part 28: Performance Optimization →](./part-28-performance.md)*

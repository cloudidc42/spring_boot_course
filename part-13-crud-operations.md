# Part 13: CRUD Operations ครบถ้วน
## ขั้นตอนที่ 296-330

> **ระดับ:** กลาง (Intermediate)  
> **เวลาเรียน:** 5-6 ชั่วโมง  
> **เป้าหมาย:** สร้าง CRUD API ที่สมบูรณ์ production-ready

---

## ขั้นตอนที่ 296: Complete Product CRUD - Entity

```java
@Entity
@Table(name = "products")
@Getter @Setter @Builder @NoArgsConstructor @AllArgsConstructor
@EntityListeners(AuditingEntityListener.class)
public class Product extends BaseEntity {
    
    @Column(nullable = false, length = 255)
    @NotBlank
    private String name;
    
    @Column(columnDefinition = "TEXT")
    private String description;
    
    @Column(nullable = false, precision = 10, scale = 2)
    @DecimalMin("0.01")
    private BigDecimal price;
    
    @Column(nullable = false)
    @Min(0)
    private Integer stock = 0;
    
    @Column(name = "image_url")
    private String imageUrl;
    
    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private ProductStatus status = ProductStatus.ACTIVE;
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "category_id")
    private Category category;
    
    @Column(nullable = false)
    private Long viewCount = 0L;
}

public enum ProductStatus {
    ACTIVE, INACTIVE, OUT_OF_STOCK
}
```

---

## ขั้นตอนที่ 297: DTOs

```java
// Request DTOs
public record CreateProductRequest(
    @NotBlank(message = "Name is required")
    @Size(max = 255, message = "Name must be less than 255 characters")
    String name,
    
    String description,
    
    @NotNull(message = "Price is required")
    @DecimalMin(value = "0.01", message = "Price must be greater than 0")
    BigDecimal price,
    
    @NotNull(message = "Stock is required")
    @Min(value = 0, message = "Stock cannot be negative")
    Integer stock,
    
    String imageUrl,
    
    Long categoryId
) {}

public record UpdateProductRequest(
    @Size(max = 255)
    String name,
    
    String description,
    
    @DecimalMin("0.01")
    BigDecimal price,
    
    @Min(0)
    Integer stock,
    
    String imageUrl,
    
    ProductStatus status,
    
    Long categoryId
) {}

public record ProductSearchRequest(
    String keyword,
    ProductStatus status,
    BigDecimal minPrice,
    BigDecimal maxPrice,
    Long categoryId
) {}

// Response DTOs
public record ProductResponse(
    Long id,
    String name,
    String description,
    BigDecimal price,
    Integer stock,
    String imageUrl,
    ProductStatus status,
    Long categoryId,
    String categoryName,
    Long viewCount,
    LocalDateTime createdAt,
    LocalDateTime updatedAt
) {}

public record ProductDetailResponse(
    Long id,
    String name,
    String description,
    BigDecimal price,
    Integer stock,
    String imageUrl,
    ProductStatus status,
    CategoryResponse category,
    Long viewCount,
    String createdBy,
    LocalDateTime createdAt,
    LocalDateTime updatedAt
) {}
```

---

## ขั้นตอนที่ 298: MapStruct Mapper

```java
@Mapper(componentModel = "spring", unmappedTargetPolicy = ReportingPolicy.IGNORE)
public interface ProductMapper {
    
    @Mapping(target = "categoryId", source = "category.id")
    @Mapping(target = "categoryName", source = "category.name")
    ProductResponse toResponse(Product product);
    
    @Mapping(target = "category", ignore = true)  // set in service
    ProductDetailResponse toDetailResponse(Product product);
    
    @Mapping(target = "id", ignore = true)
    @Mapping(target = "category", ignore = true)
    @Mapping(target = "viewCount", constant = "0L")
    @Mapping(target = "status", constant = "ACTIVE")
    Product toEntity(CreateProductRequest request);
    
    @BeanMapping(nullValuePropertyMappingStrategy = NullValuePropertyMappingStrategy.IGNORE)
    @Mapping(target = "id", ignore = true)
    @Mapping(target = "category", ignore = true)
    void updateFromRequest(UpdateProductRequest request, @MappingTarget Product product);
}
```

---

## ขั้นตอนที่ 299: Service Layer

```java
@Service
@RequiredArgsConstructor
@Transactional(readOnly = true)
@Slf4j
public class ProductService {
    
    private final ProductRepository productRepository;
    private final CategoryRepository categoryRepository;
    private final ProductMapper productMapper;
    
    // ===== READ =====
    
    public Page<ProductResponse> findAll(ProductSearchRequest search, Pageable pageable) {
        return productRepository.search(
            search.keyword(),
            search.status(),
            search.minPrice(),
            search.maxPrice(),
            pageable
        ).map(productMapper::toResponse);
    }
    
    public ProductDetailResponse findById(Long id) {
        Product product = productRepository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("Product", "id", id));
        
        incrementViewCountAsync(id);
        
        return productMapper.toDetailResponse(product);
    }
    
    @Async
    @Transactional
    public void incrementViewCountAsync(Long id) {
        productRepository.incrementViewCount(id);
    }
    
    // ===== CREATE =====
    
    @Transactional
    public ProductResponse create(CreateProductRequest request) {
        Product product = productMapper.toEntity(request);
        
        if (request.categoryId() != null) {
            Category category = categoryRepository.findById(request.categoryId())
                .orElseThrow(() -> new ResourceNotFoundException("Category", "id", request.categoryId()));
            product.setCategory(category);
        }
        
        Product saved = productRepository.save(product);
        log.info("Product created: id={}, name={}", saved.getId(), saved.getName());
        
        return productMapper.toResponse(saved);
    }
    
    // ===== UPDATE =====
    
    @Transactional
    public ProductResponse update(Long id, UpdateProductRequest request) {
        Product product = productRepository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("Product", "id", id));
        
        productMapper.updateFromRequest(request, product);
        
        if (request.categoryId() != null) {
            Category category = categoryRepository.findById(request.categoryId())
                .orElseThrow(() -> new ResourceNotFoundException("Category", "id", request.categoryId()));
            product.setCategory(category);
        }
        
        return productMapper.toResponse(productRepository.save(product));
    }
    
    // ===== PATCH (partial update) =====
    
    @Transactional
    public ProductResponse patch(Long id, Map<String, Object> updates) {
        Product product = productRepository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("Product", "id", id));
        
        updates.forEach((key, value) -> {
            switch (key) {
                case "name" -> product.setName((String) value);
                case "price" -> product.setPrice(new BigDecimal(value.toString()));
                case "stock" -> product.setStock((Integer) value);
                case "status" -> product.setStatus(ProductStatus.valueOf((String) value));
                default -> log.warn("Unknown field: {}", key);
            }
        });
        
        return productMapper.toResponse(productRepository.save(product));
    }
    
    // ===== DELETE =====
    
    @Transactional
    public void delete(Long id) {
        if (!productRepository.existsById(id)) {
            throw new ResourceNotFoundException("Product", "id", id);
        }
        productRepository.deleteById(id);
        log.info("Product deleted: id={}", id);
    }
    
    // ===== BULK OPERATIONS =====
    
    @Transactional
    public List<ProductResponse> createBatch(List<CreateProductRequest> requests) {
        return requests.stream()
            .map(this::create)
            .toList();
    }
    
    @Transactional
    public void deleteBatch(List<Long> ids) {
        List<Product> products = productRepository.findAllById(ids);
        if (products.size() != ids.size()) {
            List<Long> found = products.stream().map(Product::getId).toList();
            List<Long> notFound = ids.stream().filter(id -> !found.contains(id)).toList();
            throw new ResourceNotFoundException("Products not found: " + notFound);
        }
        productRepository.deleteAllById(ids);
    }
    
    // ===== STOCK MANAGEMENT =====
    
    @Transactional
    public ProductResponse updateStock(Long id, int quantity) {
        Product product = productRepository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("Product", "id", id));
        
        int newStock = product.getStock() + quantity;
        if (newStock < 0) {
            throw new InsufficientStockException(product.getName(), Math.abs(quantity));
        }
        
        product.setStock(newStock);
        product.setStatus(newStock == 0 ? ProductStatus.OUT_OF_STOCK : ProductStatus.ACTIVE);
        
        return productMapper.toResponse(productRepository.save(product));
    }
}
```

---

## ขั้นตอนที่ 300: Controller Layer

```java
@RestController
@RequestMapping("/api/v1/products")
@RequiredArgsConstructor
@Tag(name = "Products", description = "Product management APIs")
public class ProductController {
    
    private final ProductService productService;
    
    // ===== GET ALL =====
    @GetMapping
    @Operation(summary = "Search products with pagination")
    public ResponseEntity<ApiResponse<PageResponse<ProductResponse>>> search(
        @RequestParam(required = false) String keyword,
        @RequestParam(required = false) ProductStatus status,
        @RequestParam(required = false) BigDecimal minPrice,
        @RequestParam(required = false) BigDecimal maxPrice,
        @RequestParam(required = false) Long categoryId,
        @RequestParam(defaultValue = "0") int page,
        @RequestParam(defaultValue = "20") @Max(100) int size,
        @RequestParam(defaultValue = "createdAt") String sortBy,
        @RequestParam(defaultValue = "desc") String sortDir
    ) {
        ProductSearchRequest searchRequest = new ProductSearchRequest(
            keyword, status, minPrice, maxPrice, categoryId
        );
        
        Sort sort = sortDir.equalsIgnoreCase("asc")
            ? Sort.by(sortBy).ascending()
            : Sort.by(sortBy).descending();
        Pageable pageable = PageRequest.of(page, size, sort);
        
        Page<ProductResponse> result = productService.findAll(searchRequest, pageable);
        
        return ResponseEntity.ok(ApiResponse.success(PageResponse.of(result)));
    }
    
    // ===== GET ONE =====
    @GetMapping("/{id}")
    @Operation(summary = "Get product by ID")
    public ResponseEntity<ApiResponse<ProductDetailResponse>> findById(@PathVariable Long id) {
        return ResponseEntity.ok(ApiResponse.success(productService.findById(id)));
    }
    
    // ===== CREATE =====
    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    @Operation(summary = "Create new product")
    public ResponseEntity<ApiResponse<ProductResponse>> create(
        @Valid @RequestBody CreateProductRequest request
    ) {
        ProductResponse product = productService.create(request);
        
        URI location = ServletUriComponentsBuilder
            .fromCurrentRequest()
            .path("/{id}")
            .buildAndExpand(product.id())
            .toUri();
        
        return ResponseEntity
            .created(location)
            .body(ApiResponse.success(product, "Product created successfully"));
    }
    
    // ===== UPDATE (PUT - full replace) =====
    @PutMapping("/{id}")
    @Operation(summary = "Update product (full replace)")
    public ResponseEntity<ApiResponse<ProductResponse>> update(
        @PathVariable Long id,
        @Valid @RequestBody UpdateProductRequest request
    ) {
        return ResponseEntity.ok(ApiResponse.success(productService.update(id, request)));
    }
    
    // ===== PATCH (partial update) =====
    @PatchMapping("/{id}")
    @Operation(summary = "Partially update product")
    public ResponseEntity<ApiResponse<ProductResponse>> patch(
        @PathVariable Long id,
        @RequestBody Map<String, Object> updates
    ) {
        return ResponseEntity.ok(ApiResponse.success(productService.patch(id, updates)));
    }
    
    // ===== DELETE =====
    @DeleteMapping("/{id}")
    @Operation(summary = "Delete product")
    public ResponseEntity<ApiResponse<Void>> delete(@PathVariable Long id) {
        productService.delete(id);
        return ResponseEntity.ok(ApiResponse.success(null, "Product deleted successfully"));
    }
    
    // ===== BATCH CREATE =====
    @PostMapping("/batch")
    public ResponseEntity<ApiResponse<List<ProductResponse>>> createBatch(
        @Valid @RequestBody List<@Valid CreateProductRequest> requests
    ) {
        List<ProductResponse> products = productService.createBatch(requests);
        return ResponseEntity.status(HttpStatus.CREATED)
            .body(ApiResponse.success(products, requests.size() + " products created"));
    }
    
    // ===== BATCH DELETE =====
    @DeleteMapping("/batch")
    public ResponseEntity<ApiResponse<Void>> deleteBatch(
        @RequestBody List<Long> ids
    ) {
        productService.deleteBatch(ids);
        return ResponseEntity.ok(ApiResponse.success(null, ids.size() + " products deleted"));
    }
    
    // ===== STOCK UPDATE =====
    @PatchMapping("/{id}/stock")
    public ResponseEntity<ApiResponse<ProductResponse>> updateStock(
        @PathVariable Long id,
        @RequestParam int quantity
    ) {
        return ResponseEntity.ok(ApiResponse.success(productService.updateStock(id, quantity)));
    }
}
```

---

## ขั้นตอนที่ 301: Global Exception Handler

```java
@RestControllerAdvice
@Slf4j
public class GlobalExceptionHandler {
    
    @ExceptionHandler(ResourceNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public ApiResponse<Void> handleNotFound(ResourceNotFoundException ex) {
        log.warn("Resource not found: {}", ex.getMessage());
        return ApiResponse.error(ex.getMessage());
    }
    
    @ExceptionHandler(DuplicateResourceException.class)
    @ResponseStatus(HttpStatus.CONFLICT)
    public ApiResponse<Void> handleDuplicate(DuplicateResourceException ex) {
        return ApiResponse.error(ex.getMessage());
    }
    
    @ExceptionHandler(InsufficientStockException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ApiResponse<Void> handleInsufficientStock(InsufficientStockException ex) {
        return ApiResponse.error(ex.getMessage());
    }
    
    @ExceptionHandler(MethodArgumentNotValidException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ApiResponse<Map<String, String>> handleValidation(MethodArgumentNotValidException ex) {
        Map<String, String> errors = ex.getBindingResult().getFieldErrors()
            .stream()
            .collect(Collectors.toMap(
                FieldError::getField,
                fe -> Optional.ofNullable(fe.getDefaultMessage()).orElse("Invalid"),
                (a, b) -> a
            ));
        
        return ApiResponse.<Map<String, String>>builder()
            .success(false)
            .message("Validation failed")
            .data(errors)
            .build();
    }
    
    @ExceptionHandler(OptimisticLockException.class)
    @ResponseStatus(HttpStatus.CONFLICT)
    public ApiResponse<Void> handleOptimisticLock(OptimisticLockException ex) {
        return ApiResponse.error("Resource was modified by another user. Please refresh and try again.");
    }
    
    @ExceptionHandler(Exception.class)
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    public ApiResponse<Void> handleGeneral(Exception ex) {
        log.error("Unhandled exception", ex);
        return ApiResponse.error("An unexpected error occurred");
    }
}
```

---

## ขั้นตอนที่ 302: ApiResponse Wrapper

```java
@Getter @Builder
@JsonInclude(JsonInclude.Include.NON_NULL)
public class ApiResponse<T> {
    
    private final boolean success;
    private final String message;
    private final T data;
    private final String timestamp = LocalDateTime.now().toString();
    
    public static <T> ApiResponse<T> success(T data) {
        return ApiResponse.<T>builder()
            .success(true)
            .data(data)
            .build();
    }
    
    public static <T> ApiResponse<T> success(T data, String message) {
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
}
```

---

## ขั้นตอนที่ 303-330: Testing CRUD

```java
// Product Controller Tests
@WebMvcTest(ProductController.class)
@WithMockUser
class ProductControllerTest {
    
    @Autowired MockMvc mockMvc;
    @Autowired ObjectMapper objectMapper;
    @MockBean ProductService productService;
    
    @Test
    void search_ShouldReturnPagedProducts() throws Exception {
        var products = new PageImpl<>(List.of(
            new ProductResponse(1L, "iPhone 15", null, 
                new BigDecimal("45000"), 10, null, ProductStatus.ACTIVE, null, "Electronics", 100L,
                LocalDateTime.now(), LocalDateTime.now())
        ));
        
        given(productService.findAll(any(), any())).willReturn(products);
        
        mockMvc.perform(get("/api/v1/products")
                .param("keyword", "iPhone")
                .param("page", "0")
                .param("size", "10"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.success").value(true))
            .andExpect(jsonPath("$.data.content").isArray())
            .andExpect(jsonPath("$.data.content[0].name").value("iPhone 15"))
            .andExpect(jsonPath("$.data.totalElements").value(1));
    }
    
    @Test
    void create_ValidRequest_ShouldReturn201() throws Exception {
        var request = new CreateProductRequest(
            "iPhone 15", "Latest iPhone", 
            new BigDecimal("45000"), 100, null, 1L
        );
        
        var response = new ProductResponse(1L, "iPhone 15", "Latest iPhone",
            new BigDecimal("45000"), 100, null, ProductStatus.ACTIVE, 1L, "Electronics", 0L,
            LocalDateTime.now(), LocalDateTime.now()
        );
        
        given(productService.create(any())).willReturn(response);
        
        mockMvc.perform(post("/api/v1/products")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request)))
            .andExpect(status().isCreated())
            .andExpect(jsonPath("$.data.id").value(1))
            .andExpect(header().exists("Location"));
    }
    
    @Test
    void create_InvalidPrice_ShouldReturn400() throws Exception {
        var request = new CreateProductRequest(
            "Bad Product", null, 
            new BigDecimal("-100"), 0, null, null
        );
        
        mockMvc.perform(post("/api/v1/products")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request)))
            .andExpect(status().isBadRequest());
    }
    
    @Test
    void delete_WhenExists_ShouldReturn200() throws Exception {
        doNothing().when(productService).delete(1L);
        
        mockMvc.perform(delete("/api/v1/products/1"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.success").value(true));
    }
    
    @Test
    void delete_WhenNotExists_ShouldReturn404() throws Exception {
        doThrow(new ResourceNotFoundException("Product", "id", 999L))
            .when(productService).delete(999L);
        
        mockMvc.perform(delete("/api/v1/products/999"))
            .andExpect(status().isNotFound());
    }
}
```

---

*[← Part 12: Database Setup](./part-12-database-setup.md) | [Part 14: Validation →](./part-14-validation.md)*

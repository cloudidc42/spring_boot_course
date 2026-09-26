# Part 32: Spring WebFlux - Reactive Programming
## ขั้นตอนที่ 891-930

> **ระดับ:** สูง (Advanced)  
> **เวลาเรียน:** 6-7 ชั่วโมง  
> **เป้าหมาย:** Reactive programming ด้วย Spring WebFlux

---

## ขั้นตอนที่ 891: Reactive Programming Concepts

```
Traditional (Blocking):
  Thread 1: Request → DB query (WAIT) → Response
  Thread 2: Request → External API (WAIT) → Response
  → Need 100 threads for 100 concurrent requests

Reactive (Non-blocking):
  Thread 1: Request → DB query (register callback) → other work
  Thread 1: DB responds → callback runs → Response
  → 4 threads can handle 1000+ concurrent requests

Reactive Streams:
  Publisher → Subscriber (with backpressure)
  
Project Reactor (Spring's library):
  Mono<T>  = 0 or 1 item
  Flux<T>  = 0 to N items
```

---

## ขั้นตอนที่ 892: Dependencies

```xml
<!-- WebFlux replaces spring-boot-starter-web -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-webflux</artifactId>
</dependency>

<!-- Reactive database -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-r2dbc</artifactId>
</dependency>
<dependency>
    <groupId>io.asyncer</groupId>
    <artifactId>r2dbc-mysql</artifactId>
    <scope>runtime</scope>
</dependency>
<!-- OR PostgreSQL -->
<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>r2dbc-postgresql</artifactId>
</dependency>
```

---

## ขั้นตอนที่ 893: R2DBC Configuration

```yaml
spring:
  r2dbc:
    url: r2dbc:postgresql://localhost:5432/mydb
    username: postgres
    password: secret
    pool:
      max-size: 20
      initial-size: 5
```

```java
@Configuration
@EnableR2dbcRepositories
public class R2dbcConfig extends AbstractR2dbcConfiguration {
    
    @Value("${spring.r2dbc.url}")
    private String r2dbcUrl;
    
    @Override
    public ConnectionFactory connectionFactory() {
        return ConnectionFactories.get(r2dbcUrl);
    }
}
```

---

## ขั้นตอนที่ 894: Reactive Entity and Repository

```java
@Table("products")
@Getter @Setter @Builder @NoArgsConstructor @AllArgsConstructor
public class Product {
    
    @Id
    private Long id;
    
    private String name;
    
    @Column("price")
    private BigDecimal price;
    
    private Integer stock;
    
    @Column("status")
    private String status;
    
    @Column("created_at")
    private LocalDateTime createdAt;
}

// Reactive Repository
public interface ProductRepository extends ReactiveCrudRepository<Product, Long> {
    
    Flux<Product> findByStatus(String status);
    
    Flux<Product> findByPriceBetween(BigDecimal min, BigDecimal max);
    
    Mono<Long> countByStatus(String status);
    
    @Query("SELECT * FROM products WHERE name ILIKE :keyword ORDER BY id LIMIT :limit OFFSET :offset")
    Flux<Product> searchByKeyword(String keyword, int limit, long offset);
}
```

---

## ขั้นตอนที่ 895: Reactive Service

```java
@Service
@RequiredArgsConstructor
public class ProductService {
    
    private final ProductRepository productRepository;
    
    // Mono = single product
    public Mono<Product> findById(Long id) {
        return productRepository.findById(id)
            .switchIfEmpty(Mono.error(new ResourceNotFoundException("Product not found: " + id)));
    }
    
    // Flux = list of products
    public Flux<Product> findAll() {
        return productRepository.findByStatus("ACTIVE");
    }
    
    // Create
    public Mono<Product> create(CreateProductRequest request) {
        Product product = Product.builder()
            .name(request.name())
            .price(request.price())
            .stock(request.stock())
            .status("ACTIVE")
            .createdAt(LocalDateTime.now())
            .build();
        
        return productRepository.save(product);
    }
    
    // Update
    public Mono<Product> update(Long id, UpdateProductRequest request) {
        return findById(id)
            .flatMap(product -> {
                if (request.name() != null) product.setName(request.name());
                if (request.price() != null) product.setPrice(request.price());
                return productRepository.save(product);
            });
    }
    
    // Delete
    public Mono<Void> delete(Long id) {
        return findById(id)
            .flatMap(productRepository::delete);
    }
    
    // Aggregate: combine multiple async calls
    public Mono<ProductDetailResponse> getProductDetail(Long id) {
        Mono<Product> productMono = findById(id);
        Mono<List<Review>> reviewsMono = reviewService.findByProductId(id).collectList();
        Mono<Long> countMono = reviewService.countByProductId(id);
        
        return Mono.zip(productMono, reviewsMono, countMono)
            .map(tuple -> new ProductDetailResponse(
                tuple.getT1(),
                tuple.getT2(),
                tuple.getT3()
            ));
    }
}
```

---

## ขั้นตอนที่ 896: Reactive Controller

```java
@RestController
@RequestMapping("/api/v1/products")
@RequiredArgsConstructor
public class ProductController {
    
    private final ProductService productService;
    
    @GetMapping
    public Flux<Product> findAll() {
        return productService.findAll();
    }
    
    @GetMapping("/{id}")
    public Mono<ResponseEntity<Product>> findById(@PathVariable Long id) {
        return productService.findById(id)
            .map(ResponseEntity::ok)
            .onErrorReturn(ResourceNotFoundException.class, ResponseEntity.notFound().build());
    }
    
    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public Mono<Product> create(@Valid @RequestBody CreateProductRequest request) {
        return productService.create(request);
    }
    
    @PutMapping("/{id}")
    public Mono<Product> update(@PathVariable Long id, 
                                @Valid @RequestBody UpdateProductRequest request) {
        return productService.update(id, request);
    }
    
    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public Mono<Void> delete(@PathVariable Long id) {
        return productService.delete(id);
    }
    
    // SSE - Server-Sent Events (stream data to browser)
    @GetMapping(value = "/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<Product> stream() {
        return productService.findAll()
            .delayElements(Duration.ofMillis(100));  // Throttle stream
    }
}
```

---

## ขั้นตอนที่ 897: Error Handling in Reactive

```java
@Component
public class GlobalErrorWebExceptionHandler extends AbstractErrorWebExceptionHandler {
    
    @Override
    protected RouterFunction<ServerResponse> getRoutingFunction(ErrorAttributes errorAttributes) {
        return RouterFunctions.route(RequestPredicates.all(), this::renderErrorResponse);
    }
    
    private Mono<ServerResponse> renderErrorResponse(ServerRequest request) {
        Map<String, Object> errorPropertiesMap = getErrorAttributes(request, 
            ErrorAttributeOptions.defaults());
        
        int status = (Integer) errorPropertiesMap.getOrDefault("status", 500);
        
        return ServerResponse.status(status)
            .contentType(MediaType.APPLICATION_JSON)
            .body(BodyInserters.fromValue(errorPropertiesMap));
    }
}

// In service - transform errors
public Mono<Product> findById(Long id) {
    return productRepository.findById(id)
        .switchIfEmpty(Mono.error(new ResourceNotFoundException("Product not found: " + id)))
        .onErrorMap(DataAccessException.class, ex -> 
            new ServiceException("Database error: " + ex.getMessage()));
}
```

---

## ขั้นตอนที่ 898: WebClient (Reactive HTTP Client)

```java
@Configuration
public class WebClientConfig {
    
    @Bean
    public WebClient webClient() {
        return WebClient.builder()
            .codecs(config -> config.defaultCodecs().maxInMemorySize(2 * 1024 * 1024))
            .defaultHeader(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE)
            .filter(logRequest())
            .build();
    }
    
    private ExchangeFilterFunction logRequest() {
        return (request, next) -> {
            log.debug("{} {}", request.method(), request.url());
            return next.exchange(request);
        };
    }
}

@Service
@RequiredArgsConstructor
public class ExternalApiClient {
    
    private final WebClient webClient;
    
    public Mono<UserResponse> getUser(Long userId) {
        return webClient.get()
            .uri("https://user-service/api/v1/users/{id}", userId)
            .retrieve()
            .onStatus(HttpStatusCode::is4xxClientError, 
                resp -> resp.bodyToMono(String.class)
                    .flatMap(body -> Mono.error(new ClientException(body))))
            .onStatus(HttpStatusCode::is5xxServerError,
                resp -> Mono.error(new ServiceUnavailableException("User service unavailable")))
            .bodyToMono(UserResponse.class)
            .timeout(Duration.ofSeconds(5))
            .retryWhen(Retry.backoff(3, Duration.ofMillis(500)));
    }
    
    // Parallel calls
    public Mono<OrderDetailResponse> getOrderWithUser(Long orderId) {
        return orderRepository.findById(orderId)
            .flatMap(order -> 
                Mono.zip(
                    Mono.just(order),
                    getUser(order.getUserId())
                )
                .map(tuple -> new OrderDetailResponse(tuple.getT1(), tuple.getT2()))
            );
    }
}
```

---

## ขั้นตอนที่ 899-930: Reactive Testing

```java
@SpringBootTest
class ProductServiceTest {
    
    @Autowired
    private ProductService productService;
    
    @Test
    void findById_WhenExists_ShouldReturnProduct() {
        // Use StepVerifier to test reactive
        StepVerifier.create(productService.findById(1L))
            .assertNext(product -> {
                assertThat(product.getId()).isEqualTo(1L);
                assertThat(product.getName()).isNotBlank();
            })
            .verifyComplete();
    }
    
    @Test
    void findById_WhenNotExists_ShouldError() {
        StepVerifier.create(productService.findById(999L))
            .expectError(ResourceNotFoundException.class)
            .verify();
    }
    
    @Test
    void findAll_ShouldReturnFlux() {
        StepVerifier.create(productService.findAll())
            .expectNextCount(3)
            .verifyComplete();
    }
    
    // Test with timeout
    @Test
    void create_ShouldSaveProduct() {
        CreateProductRequest request = new CreateProductRequest("Test", BigDecimal.TEN, 100, null, null);
        
        StepVerifier.create(productService.create(request))
            .assertNext(p -> assertThat(p.getId()).isNotNull())
            .verifyComplete();
    }
}
```

---

*[← Part 31: WebSocket](./part-31-websocket.md) | [Part 33: Domain-Driven Design →](./part-33-ddd.md)*

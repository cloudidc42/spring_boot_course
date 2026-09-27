# Part 59: Reactive Patterns
## ขั้นตอนที่ 1961-2000

**ระดับ:** ระดับโลก (World-Class)
**เวลาเรียน:** 6-7 ชั่วโมง
**เป้าหมาย:** เชี่ยวชาญ Advanced Spring WebFlux Patterns, Backpressure Strategies, และ Reactive Streams ด้วย Project Reactor

---

## ขั้นตอนที่ 1961-1965: Advanced Spring WebFlux Patterns

### ทำไมต้อง Reactive?

ระบบ traditional (Blocking I/O) จะ block thread รอ database หรือ HTTP call ทำให้ใช้ thread จำนวนมาก แต่ Reactive ใช้ non-blocking I/O ทำให้ thread จำนวนน้อยรับ request ได้มาก

```
Traditional:
Thread 1: Request → [WAITING FOR DB] → Response  (thread blocked)
Thread 2: Request → [WAITING FOR DB] → Response  (thread blocked)
... Need 100 threads for 100 concurrent requests

Reactive:
Thread 1: Request → [async callback] → other work
Thread 1: DB response → [continue] → Response
... 10 threads can handle 1000 concurrent requests
```

### Dependencies

```xml
<!-- pom.xml -->
<dependencies>
    <!-- Spring WebFlux -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-webflux</artifactId>
    </dependency>
    
    <!-- R2DBC (Reactive Database) -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-r2dbc</artifactId>
    </dependency>
    
    <!-- R2DBC PostgreSQL driver -->
    <dependency>
        <groupId>io.r2dbc</groupId>
        <artifactId>r2dbc-postgresql</artifactId>
    </dependency>
    
    <!-- Reactive Redis -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-redis-reactive</artifactId>
    </dependency>
    
    <!-- Project Reactor Test -->
    <dependency>
        <groupId>io.projectreactor</groupId>
        <artifactId>reactor-test</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

### Configuration

```yaml
# application.yml
spring:
  r2dbc:
    url: r2dbc:postgresql://localhost:5432/mydb
    username: postgres
    password: ${DB_PASSWORD}
    pool:
      initial-size: 5
      max-size: 20
      max-idle-time: 30m
      
  data:
    redis:
      host: localhost
      port: 6379
      
  webflux:
    base-path: /api

reactor:
  netty:
    io-thread-count: 4
```

---

## ขั้นตอนที่ 1966-1972: Backpressure Strategies

### ทำความเข้าใจ Backpressure

Backpressure คือกลไกที่ subscriber แจ้ง publisher ว่ารับข้อมูลได้เท่าไร ป้องกัน buffer overflow

```java
// BackpressureExampleService.java
@Service
@Slf4j
public class BackpressureExampleService {
    
    /**
     * Strategy 1: BUFFER - เก็บ elements ที่เกินใน buffer
     * ⚠️ อาจ OutOfMemoryError ถ้า producer เร็วกว่า consumer มาก
     */
    public Flux<Integer> withBufferStrategy() {
        return Flux.range(1, 1000)
            .log("source")
            .onBackpressureBuffer(100, // buffer size
                dropped -> log.warn("Dropped: {}", dropped),
                BufferOverflowStrategy.DROP_OLDEST
            )
            .delayElements(Duration.ofMillis(10))
            .log("consumer");
    }
    
    /**
     * Strategy 2: DROP - ทิ้ง elements ที่ consumer ยังรับไม่ทัน
     * ใช้เมื่อ latest data สำคัญกว่า old data
     */
    public Flux<Integer> withDropStrategy() {
        return Flux.range(1, 1000)
            .onBackpressureDrop(dropped -> 
                log.warn("Dropped element: {}", dropped))
            .delayElements(Duration.ofMillis(10));
    }
    
    /**
     * Strategy 3: LATEST - เก็บแค่ค่าล่าสุด
     * เหมาะสำหรับ real-time sensor data
     */
    public Flux<Integer> withLatestStrategy() {
        return Flux.range(1, 1000)
            .onBackpressureLatest()
            .delayElements(Duration.ofMillis(10));
    }
    
    /**
     * Strategy 4: ERROR - throw BackpressureException
     * ใช้เมื่อต้องการรู้ว่ามี backpressure เกิดขึ้น
     */
    public Flux<Integer> withErrorStrategy() {
        return Flux.range(1, 1000)
            .onBackpressureError()
            .delayElements(Duration.ofMillis(10))
            .onErrorResume(ex -> {
                log.error("Backpressure error: {}", ex.getMessage());
                return Flux.empty();
            });
    }
    
    /**
     * Controlled Backpressure ด้วย limitRate
     */
    public Flux<String> withLimitRate() {
        return Flux.fromStream(generateInfiniteStream())
            .limitRate(100)        // request 100 items at a time
            .map(item -> processItem(item))
            .publishOn(Schedulers.boundedElastic(), 50); // prefetch 50
    }
    
    /**
     * Custom Backpressure ด้วย create()
     */
    public Flux<Event> withCustomBackpressure() {
        return Flux.create(sink -> {
            // FluxSink ช่วยให้ control emission
            sink.onRequest(requested -> {
                log.debug("Consumer requested: {} elements", requested);
                // ส่งแค่จำนวนที่ consumer ขอ
                for (long i = 0; i < requested && !sink.isCancelled(); i++) {
                    sink.next(generateEvent());
                }
            });
            
            sink.onDispose(() -> log.debug("Subscription disposed"));
            sink.onCancel(() -> log.debug("Subscription cancelled"));
        }, FluxSink.OverflowStrategy.BUFFER);
    }
    
    private Stream<String> generateInfiniteStream() {
        return Stream.iterate(0, i -> i + 1)
            .map(i -> "item-" + i);
    }
    
    private String processItem(String item) {
        return "processed-" + item;
    }
    
    private Event generateEvent() {
        return new Event(UUID.randomUUID().toString(), Instant.now());
    }
}
```

---

## ขั้นตอนที่ 1973-1978: Reactive Streams กับ Project Reactor

### Mono และ Flux Patterns

```java
// ReactorPatternsService.java
@Service
@Slf4j
public class ReactorPatternsService {
    
    /**
     * Pattern: Timeout with Fallback
     */
    public Mono<UserProfile> getUserWithTimeout(String userId) {
        return fetchUserFromDatabase(userId)
            .timeout(Duration.ofSeconds(3))
            .onErrorReturn(TimeoutException.class, 
                UserProfile.defaultProfile(userId))
            .onErrorResume(Exception.class, ex -> {
                log.error("Failed to fetch user: {}", userId, ex);
                return Mono.just(UserProfile.anonymousProfile());
            });
    }
    
    /**
     * Pattern: Retry with Exponential Backoff
     */
    public Mono<OrderResult> processOrderWithRetry(Order order) {
        return callExternalOrderService(order)
            .retryWhen(
                Retry.backoff(3, Duration.ofSeconds(1))
                    .maxBackoff(Duration.ofSeconds(10))
                    .jitter(0.5)
                    .filter(ex -> ex instanceof TransientException)
                    .doBeforeRetry(retrySignal -> 
                        log.warn("Retrying order {} (attempt {})", 
                            order.getId(), retrySignal.totalRetries() + 1))
            )
            .onErrorResume(ex -> {
                log.error("Order processing failed after retries: {}", order.getId());
                return Mono.just(OrderResult.failed(order.getId(), ex.getMessage()));
            });
    }
    
    /**
     * Pattern: Cache with TTL
     */
    private final Map<String, Mono<Product>> productCache = new ConcurrentHashMap<>();
    
    public Mono<Product> getProductCached(String productId) {
        return productCache.computeIfAbsent(productId, id -> 
            fetchProduct(id)
                .cache(Duration.ofMinutes(5))
        );
    }
    
    /**
     * Pattern: Conditional Processing
     */
    public Mono<Response> processConditionally(Request request) {
        return Mono.just(request)
            .filter(r -> r.isValid())
            .switchIfEmpty(Mono.error(new InvalidRequestException("Invalid request")))
            .flatMap(r -> r.requiresAuthentication() 
                ? authenticateAndProcess(r)
                : processWithoutAuth(r))
            .map(result -> Response.success(result))
            .onErrorMap(InvalidRequestException.class, 
                ex -> new ResponseException(400, ex.getMessage()))
            .onErrorMap(AuthException.class, 
                ex -> new ResponseException(401, "Unauthorized"));
    }
    
    /**
     * Pattern: Accumulate and Process in Batches
     */
    public Flux<BatchResult> processBatches(Flux<Item> items) {
        return items
            .bufferTimeout(100, Duration.ofSeconds(1)) // batch ทุก 100 items หรือ 1 วินาที
            .flatMap(batch -> processBatch(batch), 4) // parallel แค่ 4 concurrent batches
            .doOnNext(result -> log.info("Batch processed: {}", result.getBatchId()));
    }
    
    private Mono<UserProfile> fetchUserFromDatabase(String userId) {
        return Mono.fromCallable(() -> {
            // blocking call wrapped in Mono
            Thread.sleep(100);
            return new UserProfile(userId, "Test User");
        }).subscribeOn(Schedulers.boundedElastic());
    }
    
    private Mono<OrderResult> callExternalOrderService(Order order) {
        return Mono.error(new TransientException("Temporary error"));
    }
    
    private Mono<Product> fetchProduct(String id) {
        return Mono.just(new Product(id, "Product " + id));
    }
    
    private Mono<Object> authenticateAndProcess(Request r) {
        return Mono.just("authenticated-result");
    }
    
    private Mono<Object> processWithoutAuth(Request r) {
        return Mono.just("unauthenticated-result");
    }
    
    private Mono<BatchResult> processBatch(List<Item> batch) {
        return Mono.fromCallable(() -> new BatchResult(batch.size()))
            .subscribeOn(Schedulers.boundedElastic());
    }
}
```

---

## ขั้นตอนที่ 1979-1984: Error Handling ใน Reactive Pipelines

```java
// ReactiveErrorHandlingService.java
@Service
@Slf4j
public class ReactiveErrorHandlingService {
    
    /**
     * Error Handling Strategies
     */
    public Flux<ProductDto> getProductsWithErrorHandling(List<String> productIds) {
        return Flux.fromIterable(productIds)
            // แปลง upstream error เป็น data
            .flatMap(id -> fetchProduct(id)
                .onErrorResume(ProductNotFoundException.class, ex -> {
                    log.warn("Product not found: {}", id);
                    return Mono.just(ProductDto.notFound(id));
                })
                .onErrorResume(ServiceUnavailableException.class, ex -> {
                    log.error("Service unavailable for: {}", id);
                    return Mono.just(ProductDto.unavailable(id));
                })
            )
            // Filter out error placeholders
            .filter(dto -> !dto.isError())
            // Handle any remaining errors
            .onErrorContinue((ex, element) -> {
                log.error("Unexpected error for element {}: {}", element, ex.getMessage());
            });
    }
    
    /**
     * Error Recovery with Circuit Breaker
     */
    public Mono<String> callWithCircuitBreaker(String serviceId) {
        return callService(serviceId)
            .transformDeferred(CircuitBreakerOperator.of(
                CircuitBreaker.ofDefaults("myService")
            ))
            .onErrorResume(CallNotPermittedException.class, ex -> {
                log.warn("Circuit breaker open, returning fallback");
                return Mono.just("FALLBACK_RESPONSE");
            });
    }
    
    /**
     * Structured Error Response
     */
    public Mono<ApiResponse<Order>> createOrderSafely(OrderRequest request) {
        return validateRequest(request)
            .flatMap(this::checkInventory)
            .flatMap(this::createOrder)
            .flatMap(this::sendConfirmation)
            .map(order -> ApiResponse.success(order))
            .onErrorResume(ValidationException.class, ex ->
                Mono.just(ApiResponse.error(400, ex.getMessage())))
            .onErrorResume(InsufficientInventoryException.class, ex ->
                Mono.just(ApiResponse.error(409, "Insufficient inventory: " + ex.getMessage())))
            .onErrorResume(Exception.class, ex -> {
                log.error("Unexpected error creating order", ex);
                return Mono.just(ApiResponse.error(500, "Internal server error"));
            })
            .doOnSuccess(response -> 
                log.info("Order creation result: status={}", response.getStatus()))
            .doOnError(ex -> 
                log.error("Unhandled error in order creation", ex));
    }
    
    /**
     * Error propagation กับ context
     */
    public Mono<String> processWithContext(String userId, String data) {
        return Mono.deferContextual(ctx -> {
            String correlationId = ctx.getOrDefault("correlationId", "N/A");
            
            return Mono.just(data)
                .flatMap(d -> processData(d, userId))
                .onErrorMap(ex -> new ProcessingException(
                    "Failed to process for user " + userId + 
                    " (correlationId: " + correlationId + "): " + ex.getMessage(), ex
                ));
        }).contextWrite(Context.of("correlationId", UUID.randomUUID().toString()));
    }
    
    private Mono<OrderRequest> validateRequest(OrderRequest request) {
        if (request.getQuantity() <= 0) {
            return Mono.error(new ValidationException("Quantity must be positive"));
        }
        return Mono.just(request);
    }
    
    private Mono<OrderRequest> checkInventory(OrderRequest request) {
        return Mono.just(request); // simplified
    }
    
    private Mono<Order> createOrder(OrderRequest request) {
        return Mono.just(new Order()); // simplified
    }
    
    private Mono<Order> sendConfirmation(Order order) {
        return Mono.just(order); // simplified
    }
    
    private Mono<String> callService(String id) {
        return Mono.just("response");
    }
    
    private Mono<String> processData(String data, String userId) {
        return Mono.just("processed-" + data);
    }
    
    private Mono<ProductDto> fetchProduct(String id) {
        return Mono.just(new ProductDto(id, "Product " + id));
    }
}
```

---

## ขั้นตอนที่ 1985-1990: Combining Reactive Sources

```java
// ReactiveCombiningService.java
@Service
@Slf4j
public class ReactiveCombiningService {
    
    private final UserService userService;
    private final OrderService orderService;
    private final InventoryService inventoryService;
    private final PricingService pricingService;
    
    public ReactiveCombiningService(UserService userService,
                                     OrderService orderService,
                                     InventoryService inventoryService,
                                     PricingService pricingService) {
        this.userService = userService;
        this.orderService = orderService;
        this.inventoryService = inventoryService;
        this.pricingService = pricingService;
    }
    
    /**
     * zip: รวม 2 Mono ที่ต้องรอทั้งคู่
     */
    public Mono<OrderSummary> getOrderSummary(String orderId) {
        Mono<Order> orderMono = orderService.findById(orderId);
        Mono<User> userMono = orderMono.flatMap(o -> userService.findById(o.getUserId()));
        
        return Mono.zip(orderMono, userMono)
            .map(tuple -> {
                Order order = tuple.getT1();
                User user = tuple.getT2();
                return new OrderSummary(order, user);
            });
    }
    
    /**
     * zipWith: combine ด้วย BiFunction
     */
    public Mono<EnrichedProduct> getEnrichedProduct(String productId) {
        return inventoryService.getProduct(productId)
            .zipWith(
                pricingService.getCurrentPrice(productId),
                (product, price) -> new EnrichedProduct(product, price)
            );
    }
    
    /**
     * merge: รวม streams แบบ interleaved (ใช้เมื่อต้องการ events จากหลาย sources)
     */
    public Flux<Event> getAllEvents() {
        Flux<Event> userEvents = userService.getUserEvents();
        Flux<Event> orderEvents = orderService.getOrderEvents();
        Flux<Event> inventoryEvents = inventoryService.getInventoryEvents();
        
        return Flux.merge(userEvents, orderEvents, inventoryEvents)
            .sort(Comparator.comparing(Event::getTimestamp));
    }
    
    /**
     * concat: รวม streams ตามลำดับ
     */
    public Flux<Product> getAllProductsSequentially() {
        Flux<Product> electronics = inventoryService.getByCategory("electronics");
        Flux<Product> clothing = inventoryService.getByCategory("clothing");
        Flux<Product> food = inventoryService.getByCategory("food");
        
        return Flux.concat(electronics, clothing, food);
    }
    
    /**
     * flatMap: parallel processing
     */
    public Flux<ProductDetail> enrichProducts(List<String> productIds) {
        return Flux.fromIterable(productIds)
            .flatMap(id -> 
                Mono.zip(
                    inventoryService.getProduct(id),
                    pricingService.getCurrentPrice(id),
                    inventoryService.getStock(id)
                ).map(tuple -> new ProductDetail(
                    tuple.getT1(),
                    tuple.getT2(),
                    tuple.getT3()
                )),
                10 // concurrency: ทำพร้อมกันได้ 10 ids
            );
    }
    
    /**
     * switchMap: ใช้ล่าสุดเท่านั้น (เหมาะสำหรับ search-as-you-type)
     */
    public Flux<List<Product>> liveSearch(Flux<String> searchTerms) {
        return searchTerms
            .debounce(Duration.ofMillis(300)) // รอให้ผู้ใช้หยุดพิมพ์
            .distinctUntilChanged()
            .switchMap(term -> inventoryService.search(term)
                .timeout(Duration.ofSeconds(5))
                .onErrorReturn(List.of())
            );
    }
    
    /**
     * combineLatest: รวม latest values จากหลาย streams
     */
    public Flux<DashboardData> liveDashboard() {
        Flux<Long> activeUsers = userService.getActiveUserCount();
        Flux<Long> pendingOrders = orderService.getPendingOrderCount();
        Flux<BigDecimal> totalRevenue = orderService.getTotalRevenue();
        
        return Flux.combineLatest(
            activeUsers, pendingOrders, totalRevenue,
            (users, orders, revenue) -> new DashboardData(users, orders, revenue)
        ).distinctUntilChanged();
    }
    
    /**
     * groupBy: แบ่ง stream ตาม key
     */
    public Mono<Map<String, List<Order>>> groupOrdersByStatus() {
        return orderService.getAllActiveOrders()
            .groupBy(Order::getStatus)
            .flatMap(group -> group.collectList()
                .map(orders -> Map.entry(group.key(), orders)))
            .collectMap(Map.Entry::getKey, Map.Entry::getValue);
    }
}
```

---

## ขั้นตอนที่ 1991-1994: Reactive Redis Caching

```java
// ReactiveRedisService.java
@Service
@Slf4j
public class ReactiveRedisService {
    
    private final ReactiveRedisTemplate<String, String> redisTemplate;
    private final ReactiveValueOperations<String, String> valueOps;
    private final ReactiveHashOperations<String, String, String> hashOps;
    private final ObjectMapper objectMapper;
    
    public ReactiveRedisService(ReactiveRedisTemplate<String, String> redisTemplate,
                                 ObjectMapper objectMapper) {
        this.redisTemplate = redisTemplate;
        this.valueOps = redisTemplate.opsForValue();
        this.hashOps = redisTemplate.opsForHash();
        this.objectMapper = objectMapper;
    }
    
    /**
     * Cache-Aside pattern แบบ Reactive
     */
    public <T> Mono<T> cacheAside(String cacheKey, Class<T> type,
                                   Mono<T> dataSource, Duration ttl) {
        return valueOps.get(cacheKey)
            .flatMap(cached -> deserialize(cached, type))
            .switchIfEmpty(
                dataSource
                    .flatMap(data -> serialize(data)
                        .flatMap(serialized -> valueOps.set(cacheKey, serialized, ttl))
                        .thenReturn(data))
            )
            .doOnNext(data -> log.debug("Cache hit/miss for key: {}", cacheKey));
    }
    
    /**
     * Cache multiple keys พร้อมกัน
     */
    public <T> Mono<Map<String, T>> multiGet(List<String> keys, Class<T> type,
                                              Function<List<String>, Mono<Map<String, T>>> loader) {
        return Flux.fromIterable(keys)
            .flatMap(key -> valueOps.get(key)
                .flatMap(v -> deserialize(v, type))
                .map(value -> Map.entry(key, value))
                .switchIfEmpty(Mono.empty())
            )
            .collectMap(Map.Entry::getKey, Map.Entry::getValue)
            .flatMap(cached -> {
                List<String> missedKeys = keys.stream()
                    .filter(k -> !cached.containsKey(k))
                    .collect(Collectors.toList());
                
                if (missedKeys.isEmpty()) {
                    return Mono.just(cached);
                }
                
                return loader.apply(missedKeys)
                    .flatMap(loaded -> {
                        // Cache missed values
                        return Flux.fromIterable(loaded.entrySet())
                            .flatMap(entry -> serialize(entry.getValue())
                                .flatMap(v -> valueOps.set(entry.getKey(), v, 
                                    Duration.ofMinutes(10))))
                            .then(Mono.just(loaded));
                    })
                    .map(loaded -> {
                        Map<String, T> combined = new HashMap<>(cached);
                        combined.putAll(loaded);
                        return combined;
                    });
            });
    }
    
    /**
     * Pub/Sub Reactive
     */
    public Flux<String> subscribe(String channel) {
        return redisTemplate.listenToChannel(channel)
            .map(message -> message.getMessage())
            .doOnNext(msg -> log.debug("Received from {}: {}", channel, msg))
            .doOnError(ex -> log.error("Error in channel: {}", channel, ex));
    }
    
    public Mono<Long> publish(String channel, String message) {
        return redisTemplate.convertAndSend(channel, message)
            .doOnSuccess(subscribers -> 
                log.debug("Published to {} subscribers on channel: {}", 
                    subscribers, channel));
    }
    
    /**
     * Rate Limiting ด้วย Redis
     */
    public Mono<Boolean> checkRateLimit(String key, int maxRequests, Duration window) {
        String redisKey = "rate_limit:" + key;
        
        return redisTemplate.execute(
            connection -> connection.numberCommands()
                .incr(redisKey.getBytes())
                .flatMap(count -> {
                    if (count == 1L) {
                        // First request - set expiry
                        return connection.keyCommands()
                            .expire(redisKey.getBytes(), window)
                            .thenReturn(count);
                    }
                    return Mono.just(count);
                })
        )
        .next()
        .map(count -> count <= maxRequests);
    }
    
    private <T> Mono<String> serialize(T obj) {
        return Mono.fromCallable(() -> objectMapper.writeValueAsString(obj));
    }
    
    private <T> Mono<T> deserialize(String json, Class<T> type) {
        return Mono.fromCallable(() -> objectMapper.readValue(json, type));
    }
}
```

### Reactive Redis Config

```java
// ReactiveRedisConfig.java
@Configuration
public class ReactiveRedisConfig {
    
    @Bean
    public ReactiveRedisTemplate<String, String> reactiveRedisTemplate(
            ReactiveRedisConnectionFactory factory) {
        
        RedisSerializationContext<String, String> context = 
            RedisSerializationContext.<String, String>newSerializationContext(
                new StringRedisSerializer()
            )
            .value(new StringRedisSerializer())
            .build();
        
        return new ReactiveRedisTemplate<>(factory, context);
    }
    
    @Bean
    public ReactiveRedisTemplate<String, Object> reactiveObjectRedisTemplate(
            ReactiveRedisConnectionFactory factory) {
        
        Jackson2JsonRedisSerializer<Object> serializer = 
            new Jackson2JsonRedisSerializer<>(Object.class);
        
        RedisSerializationContext<String, Object> context = 
            RedisSerializationContext.<String, Object>newSerializationContext(
                new StringRedisSerializer()
            )
            .value(serializer)
            .build();
        
        return new ReactiveRedisTemplate<>(factory, context);
    }
}
```

---

## ขั้นตอนที่ 1995-2000: WebFlux + R2DBC Complete Example

### Domain Entity

```java
// Product.java (R2DBC Entity)
@Table("products")
public class Product {
    
    @Id
    private Long id;
    
    @Column("name")
    private String name;
    
    @Column("price")
    private BigDecimal price;
    
    @Column("category")
    private String category;
    
    @Column("active")
    private boolean active;
    
    @CreatedDate
    @Column("created_at")
    private LocalDateTime createdAt;
    
    // Constructors, getters, setters
    public Product() {}
    
    public Product(String name, BigDecimal price, String category) {
        this.name = name;
        this.price = price;
        this.category = category;
        this.active = true;
    }
    
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    public BigDecimal getPrice() { return price; }
    public void setPrice(BigDecimal price) { this.price = price; }
    public String getCategory() { return category; }
    public void setCategory(String category) { this.category = category; }
    public boolean isActive() { return active; }
    public void setActive(boolean active) { this.active = active; }
    public LocalDateTime getCreatedAt() { return createdAt; }
}
```

### R2DBC Repository

```java
// ProductR2dbcRepository.java
@Repository
public interface ProductR2dbcRepository extends ReactiveCrudRepository<Product, Long> {
    
    Flux<Product> findByCategory(String category);
    
    Flux<Product> findByActiveTrue();
    
    @Query("SELECT * FROM products WHERE price BETWEEN :minPrice AND :maxPrice AND active = true")
    Flux<Product> findByPriceRange(
        @Param("minPrice") BigDecimal minPrice,
        @Param("maxPrice") BigDecimal maxPrice
    );
    
    @Query("SELECT * FROM products WHERE LOWER(name) LIKE LOWER(CONCAT('%', :keyword, '%'))")
    Flux<Product> searchByName(@Param("keyword") String keyword);
    
    Mono<Long> countByCategory(String category);
}
```

### Service Layer

```java
// ProductReactiveService.java
@Service
@Slf4j
public class ProductReactiveService {
    
    private final ProductR2dbcRepository repository;
    private final ReactiveRedisService redisService;
    private final ReactiveTransactionalOperator transactionalOperator;
    
    public ProductReactiveService(ProductR2dbcRepository repository,
                                   ReactiveRedisService redisService,
                                   ReactiveTransactionalOperator transactionalOperator) {
        this.repository = repository;
        this.redisService = redisService;
        this.transactionalOperator = transactionalOperator;
    }
    
    public Mono<Product> createProduct(Product product) {
        return repository.save(product)
            .flatMap(saved -> {
                String cacheKey = "product:" + saved.getId();
                return redisService.serialize(saved)
                    .flatMap(json -> redisService.set(cacheKey, json, Duration.ofMinutes(30)))
                    .thenReturn(saved);
            })
            .doOnSuccess(p -> log.info("Created product: {}", p.getId()))
            .doOnError(ex -> log.error("Failed to create product", ex));
    }
    
    public Mono<Product> getProduct(Long id) {
        String cacheKey = "product:" + id;
        
        return redisService.cacheAside(
            cacheKey,
            Product.class,
            repository.findById(id)
                .switchIfEmpty(Mono.error(new ProductNotFoundException(id.toString()))),
            Duration.ofMinutes(30)
        );
    }
    
    public Flux<Product> getProductsByCategory(String category) {
        return repository.findByCategory(category)
            .filter(Product::isActive)
            .sort(Comparator.comparing(Product::getName));
    }
    
    /**
     * Transactional operation
     */
    public Mono<Void> transferStock(Long fromProductId, Long toProductId, int quantity) {
        return transactionalOperator.transactional(
            repository.findById(fromProductId)
                .zipWith(repository.findById(toProductId))
                .flatMap(tuple -> {
                    Product from = tuple.getT1();
                    Product to = tuple.getT2();
                    
                    // Business logic
                    log.info("Transferring stock from {} to {}", fromProductId, toProductId);
                    
                    return repository.save(from)
                        .then(repository.save(to))
                        .then();
                })
        );
    }
    
    /**
     * Streaming large dataset
     */
    public Flux<Product> streamAllActive() {
        return repository.findByActiveTrue()
            .log("stream-all-active")
            .doOnNext(p -> log.debug("Streaming product: {}", p.getId()))
            .share(); // multicasting
    }
    
    /**
     * Real-time search
     */
    public Flux<Product> searchProducts(String keyword) {
        return repository.searchByName(keyword)
            .timeout(Duration.ofSeconds(5))
            .onErrorResume(TimeoutException.class, ex -> {
                log.warn("Search timeout for keyword: {}", keyword);
                return Flux.empty();
            })
            .take(50); // limit results
    }
}
```

### WebFlux Controller

```java
// ProductReactiveController.java
@RestController
@RequestMapping("/api/v1/products")
@Slf4j
public class ProductReactiveController {
    
    private final ProductReactiveService productService;
    
    public ProductReactiveController(ProductReactiveService productService) {
        this.productService = productService;
    }
    
    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public Mono<Product> createProduct(@RequestBody @Valid Product product) {
        return productService.createProduct(product);
    }
    
    @GetMapping("/{id}")
    public Mono<ResponseEntity<Product>> getProduct(@PathVariable Long id) {
        return productService.getProduct(id)
            .map(ResponseEntity::ok)
            .onErrorReturn(ProductNotFoundException.class, 
                ResponseEntity.notFound().build());
    }
    
    @GetMapping
    public Flux<Product> getProducts(
            @RequestParam(required = false) String category,
            @RequestParam(required = false) String search) {
        
        if (search != null) {
            return productService.searchProducts(search);
        }
        if (category != null) {
            return productService.getProductsByCategory(category);
        }
        return productService.streamAllActive();
    }
    
    /**
     * Server-Sent Events (SSE) สำหรับ real-time streaming
     */
    @GetMapping(value = "/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<ServerSentEvent<Product>> streamProducts() {
        return productService.streamAllActive()
            .map(product -> ServerSentEvent.<Product>builder()
                .id(product.getId().toString())
                .event("product")
                .data(product)
                .build())
            .delayElements(Duration.ofMillis(100)); // rate limiting
    }
    
    /**
     * WebSocket Handler สำหรับ bidirectional
     */
    @Bean
    public WebSocketHandler productWebSocketHandler() {
        return session -> {
            Flux<WebSocketMessage> outbound = productService.streamAllActive()
                .map(product -> {
                    try {
                        return session.textMessage(
                            new ObjectMapper().writeValueAsString(product)
                        );
                    } catch (Exception e) {
                        return session.textMessage("Error serializing product");
                    }
                });
            
            return session.send(outbound)
                .and(session.receive()
                    .map(WebSocketMessage::getPayloadAsText)
                    .doOnNext(msg -> log.info("Received: {}", msg))
                );
        };
    }
}
```

### Testing Reactive Code

```java
// ProductReactiveServiceTest.java
@SpringBootTest
@Testcontainers
class ProductReactiveServiceTest {
    
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15-alpine")
        .withDatabaseName("testdb")
        .withUsername("test")
        .withPassword("test");
    
    @Container
    static GenericContainer<?> redis = new GenericContainer<>("redis:7-alpine")
        .withExposedPorts(6379);
    
    @Autowired
    private ProductReactiveService productService;
    
    @Test
    void createProduct_savesAndCaches() {
        Product product = new Product("Test Product", new BigDecimal("100.00"), "Electronics");
        
        StepVerifier.create(productService.createProduct(product))
            .assertNext(saved -> {
                assertThat(saved.getId()).isNotNull();
                assertThat(saved.getName()).isEqualTo("Test Product");
            })
            .verifyComplete();
    }
    
    @Test
    void getProduct_notFound_throwsException() {
        StepVerifier.create(productService.getProduct(999L))
            .expectError(ProductNotFoundException.class)
            .verify();
    }
    
    @Test
    void streamAllActive_returnsFlux() {
        // Create test data
        Flux<Product> created = Flux.range(1, 5)
            .flatMap(i -> productService.createProduct(
                new Product("Product " + i, 
                           BigDecimal.valueOf(i * 100), 
                           "Category " + (i % 2 == 0 ? "A" : "B"))
            ));
        
        StepVerifier.create(
            created.then(Mono.empty())
                .thenMany(productService.streamAllActive())
        )
        .expectNextCount(5)
        .verifyComplete();
    }
    
    @Test
    void searchProducts_withTimeout_returnsEmpty() {
        StepVerifier.create(productService.searchProducts("nonexistent"))
            .verifyComplete();
    }
    
    @Test
    void backpressureTest() throws InterruptedException {
        AtomicInteger received = new AtomicInteger(0);
        CountDownLatch latch = new CountDownLatch(1);
        
        // Create 100 products
        Flux.range(1, 100)
            .flatMap(i -> productService.createProduct(
                new Product("Product " + i, BigDecimal.valueOf(i), "Test")))
            .blockLast();
        
        // Subscribe with backpressure
        productService.streamAllActive()
            .limitRate(10) // request 10 at a time
            .subscribe(
                product -> {
                    received.incrementAndGet();
                    try { Thread.sleep(10); } 
                    catch (InterruptedException e) { Thread.currentThread().interrupt(); }
                },
                ex -> latch.countDown(),
                latch::countDown
            );
        
        latch.await(30, TimeUnit.SECONDS);
        assertThat(received.get()).isEqualTo(100);
    }
}
```

---

## สรุป Part 59

ในส่วนนี้เราได้เรียนรู้:

1. **Spring WebFlux Patterns** - Advanced reactive patterns
2. **Backpressure Strategies** - BUFFER, DROP, LATEST, ERROR
3. **Project Reactor** - Retry, timeout, caching patterns
4. **Error Handling** - Reactive error handling strategies
5. **Combining Sources** - zip, merge, flatMap, switchMap, combineLatest
6. **Reactive Redis** - Cache-aside, pub/sub, rate limiting
7. **WebFlux + R2DBC** - Complete reactive application

---

*[← Part 58: Hexagonal Architecture](./part-58-hexagonal-architecture.md) | [Part 60: Performance Testing →](./part-60-performance-testing.md)*

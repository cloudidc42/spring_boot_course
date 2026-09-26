# Part 28: Performance Optimization
## ขั้นตอนที่ 761-795

> **ระดับ:** สูง (Advanced)  
> **เวลาเรียน:** 6-7 ชั่วโมง  
> **เป้าหมาย:** Optimize Spring Boot application สำหรับ production load

---

## ขั้นตอนที่ 761: Performance Analysis Tools

```bash
# 1. JVM Profiling
# VisualVM (free): jvisualvm
# JProfiler (paid)
# Async Profiler (free): https://github.com/async-profiler/async-profiler

# 2. Load Testing
# k6: https://k6.io
# Gatling
# JMeter

# 3. APM
# Datadog, New Relic, Dynatrace (paid)
# Elastic APM (free)
# Spring Boot Actuator + Micrometer (free)

# Simple k6 load test
cat > load-test.js << 'EOF'
import http from 'k6/http';
import { sleep, check } from 'k6';

export const options = {
    vus: 100,           // 100 concurrent users
    duration: '30s',
    thresholds: {
        http_req_duration: ['p(99)<2000'],  // 99% < 2s
        http_req_failed: ['rate<0.01'],     // < 1% errors
    },
};

export default function() {
    const res = http.get('http://localhost:8080/api/v1/products');
    check(res, { 'status 200': (r) => r.status === 200 });
    sleep(1);
}
EOF

k6 run load-test.js
```

---

## ขั้นตอนที่ 762: JPA/Database Optimizations

```java
// ❌ BAD: N+1 problem
List<Order> orders = orderRepository.findAll();
orders.forEach(o -> o.getItems().size());  // N+1 queries!

// ✅ GOOD: Fetch with JOIN
@Query("SELECT o FROM Order o JOIN FETCH o.items WHERE o.userId = :userId")
List<Order> findByUserIdWithItems(@Param("userId") Long userId);

// ✅ GOOD: EntityGraph
@EntityGraph(attributePaths = {"items", "items.product"})
List<Order> findByUser(User user);

// ✅ GOOD: Batch fetch
@OneToMany(mappedBy = "order")
@BatchSize(size = 100)
private List<OrderItem> items;

// ===== Projections instead of full entity =====
// ❌ Loading full entity when only 2 fields needed
public List<ProductResponse> findAllNames() {
    return productRepository.findAll().stream()
        .map(p -> new ProductResponse(p.getId(), p.getName(), ...))
        .toList();
}

// ✅ Use interface projection
public interface ProductSummary {
    Long getId();
    String getName();
    BigDecimal getPrice();
}

public List<ProductSummary> findAllSummaries() {
    return productRepository.findAllProjectedBy();
}

// ✅ Or DTO projection with @Query
@Query("SELECT new com.myapp.dto.ProductSummaryDto(p.id, p.name, p.price) FROM Product p WHERE p.status = 'ACTIVE'")
List<ProductSummaryDto> findActiveSummaries();
```

---

## ขั้นตอนที่ 763: Database Connection Pool Tuning

```yaml
spring:
  datasource:
    hikari:
      # Formula: (2 * CPU cores) + effective_spindle_count
      # For 4-core SSD: 2*4+1 = 9, round to 10
      maximum-pool-size: 10
      minimum-idle: 5
      
      # Timeout settings
      connection-timeout: 30000    # Wait max 30s for connection
      idle-timeout: 600000         # Remove idle connection after 10min
      max-lifetime: 1800000        # Replace connection after 30min
      keepalive-time: 300000       # Test every 5min
      
      # Validation
      connection-test-query: SELECT 1
      validation-timeout: 5000
```

---

## ขั้นตอนที่ 764: Query Optimization

```java
// ✅ Use native SQL for complex reports
@Query(value = """
    SELECT 
        DATE_TRUNC('day', o.created_at) as date,
        COUNT(*) as order_count,
        SUM(o.total_amount) as revenue,
        AVG(o.total_amount) as avg_order_value
    FROM orders o
    WHERE o.created_at BETWEEN :start AND :end
    GROUP BY DATE_TRUNC('day', o.created_at)
    ORDER BY date
    """, nativeQuery = true)
List<Object[]> getDailySalesReport(@Param("start") LocalDateTime start,
                                    @Param("end") LocalDateTime end);

// ✅ Use @QueryHints for read-only
@QueryHints(@QueryHint(name = HINT_READONLY, value = "true"))
@Query("SELECT p FROM Product p WHERE p.status = 'ACTIVE'")
List<Product> findActiveProducts();

// ✅ Stream large results
@Query("SELECT p FROM Product p WHERE p.status = 'ACTIVE'")
@QueryHints(value = {
    @QueryHint(name = HINT_FETCH_SIZE, value = "100"),
    @QueryHint(name = HINT_READONLY, value = "true")
})
Stream<Product> streamActiveProducts();

// Use stream in service
@Transactional(readOnly = true)
public void exportProducts(OutputStream output) {
    try (Stream<Product> products = productRepository.streamActiveProducts()) {
        products.forEach(p -> writeToOutput(p, output));
    }
}
```

---

## ขั้นตอนที่ 765: Caching Strategies

```java
// L1 Cache: In-memory (ConcurrentHashMap or Caffeine)
// L2 Cache: Redis (distributed)

// Caffeine for L1 (fast local cache)
@Bean
public CacheManager caffeineCacheManager() {
    CaffeineCacheManager manager = new CaffeineCacheManager();
    manager.setCaffeine(Caffeine.newBuilder()
        .maximumSize(1000)
        .expireAfterWrite(5, TimeUnit.MINUTES)
        .recordStats());  // Enable stats
    return manager;
}

// Cache warming on startup
@Component
@RequiredArgsConstructor
public class CacheWarmer implements ApplicationListener<ApplicationReadyEvent> {
    
    private final CategoryService categoryService;
    
    @Override
    public void onApplicationEvent(ApplicationReadyEvent event) {
        log.info("Warming up caches...");
        categoryService.findAll();  // Loads into cache
        log.info("Cache warm-up complete");
    }
}

// Write-through caching
@Service
public class ProductService {
    
    @Cacheable(value = "products", key = "#id")
    public ProductResponse findById(Long id) {
        return productRepository.findById(id).map(mapper::toResponse)
            .orElseThrow(() -> new ResourceNotFoundException("Product", "id", id));
    }
    
    @CachePut(value = "products", key = "#result.id")
    @Transactional
    public ProductResponse update(Long id, UpdateProductRequest request) {
        // Update DB, cache auto-updated
        Product product = productRepository.findById(id).orElseThrow(...);
        mapper.updateFromRequest(request, product);
        return mapper.toResponse(productRepository.save(product));
    }
    
    @CacheEvict(value = "products", key = "#id")
    @Transactional
    public void delete(Long id) {
        productRepository.deleteById(id);
    }
}
```

---

## ขั้นตอนที่ 766: HTTP Response Compression

```yaml
server:
  compression:
    enabled: true
    mime-types: application/json,application/javascript,text/html,text/css,text/plain
    min-response-size: 1024  # Only compress responses > 1KB
```

---

## ขั้นตอนที่ 767: Async for I/O Operations

```java
// ✅ Parallel I/O operations
@Service
public class ProductDetailService {
    
    private final ProductRepository productRepository;
    private final ReviewRepository reviewRepository;
    private final RecommendationService recommendationService;
    
    public ProductPageData getProductPageData(Long productId) {
        // Run all queries in parallel
        CompletableFuture<Product> productFuture = 
            CompletableFuture.supplyAsync(() -> productRepository.findById(productId).orElseThrow());
        
        CompletableFuture<List<Review>> reviewsFuture = 
            CompletableFuture.supplyAsync(() -> reviewRepository.findByProductId(productId));
        
        CompletableFuture<List<Product>> similarFuture = 
            CompletableFuture.supplyAsync(() -> recommendationService.getSimilar(productId));
        
        // Wait for all
        return CompletableFuture.allOf(productFuture, reviewsFuture, similarFuture)
            .thenApply(v -> new ProductPageData(
                productFuture.join(),
                reviewsFuture.join(),
                similarFuture.join()
            ))
            .join();
    }
}
```

---

## ขั้นตอนที่ 768: JVM Tuning

```bash
# Start script with JVM flags
java \
  -Xms256m \                          # Min heap
  -Xmx1g \                            # Max heap
  -XX:+UseG1GC \                      # G1 GC (good for most apps)
  -XX:MaxGCPauseMillis=200 \          # Target GC pause < 200ms
  -XX:+UseStringDeduplication \       # Deduplicate strings
  -XX:+UseContainerSupport \          # Docker-aware CPU/memory limits
  -XX:MaxRAMPercentage=75.0 \         # Use 75% of container RAM
  -Djava.security.egd=file:/dev/./urandom \  # Faster SecureRandom
  -jar app.jar

# Modern Java (21+) for high throughput APIs
# Virtual threads (already enabled with spring.threads.virtual.enabled=true)
```

---

## ขั้นตอนที่ 769: Response Caching (HTTP)

```java
@RestController
public class ProductController {
    
    // Cache in browser/CDN for 5 minutes
    @GetMapping("/api/v1/products/{id}")
    public ResponseEntity<ApiResponse<ProductDetailResponse>> findById(@PathVariable Long id) {
        ProductDetailResponse product = productService.findById(id);
        
        return ResponseEntity.ok()
            .cacheControl(CacheControl.maxAge(5, TimeUnit.MINUTES)
                .cachePublic()
                .mustRevalidate())
            .eTag(String.valueOf(product.hashCode()))  // ETag for conditional requests
            .body(ApiResponse.success(product));
    }
    
    // No cache for dynamic content
    @GetMapping("/api/v1/orders")
    public ResponseEntity<?> getOrders() {
        return ResponseEntity.ok()
            .cacheControl(CacheControl.noCache().noStore())
            .body(ApiResponse.success(orderService.findAll()));
    }
}
```

---

## ขั้นตอนที่ 770: Database Indexing Strategy

```sql
-- Analyze slow queries
EXPLAIN ANALYZE SELECT * FROM products WHERE category_id = 1 AND status = 'ACTIVE';

-- Compound index for common query patterns
CREATE INDEX idx_products_category_status ON products(category_id, status);
CREATE INDEX idx_products_status_price ON products(status, price);

-- Partial index for active products only
CREATE INDEX idx_products_active ON products(id) WHERE status = 'ACTIVE';

-- Text search index
CREATE INDEX idx_products_name_gin ON products USING gin(to_tsvector('english', name));

-- Monitor index usage
SELECT 
    schemaname,
    tablename,
    indexname,
    idx_scan,
    idx_tup_read,
    idx_tup_fetch
FROM pg_stat_user_indexes
ORDER BY idx_scan DESC;

-- Find missing indexes (tables with seq scans)
SELECT 
    relname,
    seq_scan,
    idx_scan,
    n_live_tup
FROM pg_stat_user_tables
WHERE seq_scan > idx_scan
ORDER BY n_live_tup DESC;
```

---

## ขั้นตอนที่ 771-795: Performance Benchmarks

```java
// Benchmark with JMH (Java Microbenchmark Harness)
@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.MILLISECONDS)
@State(Scope.Benchmark)
@Fork(1)
@Warmup(iterations = 5)
@Measurement(iterations = 10)
public class ServiceBenchmark {
    
    @Benchmark
    public void findAllProducts() {
        productService.findAll(PageRequest.of(0, 20));
    }
    
    @Benchmark
    public void findProductById() {
        productService.findById(1L);
    }
}

// Target Performance Numbers (production ready)
// GET /api/v1/products      → P99 < 100ms, 500 RPS
// GET /api/v1/products/:id  → P99 < 20ms, 2000 RPS (cached)
// POST /api/v1/orders       → P99 < 500ms, 100 RPS
// POST /api/v1/auth/login   → P99 < 200ms, 100 RPS
```

### Performance Checklist

```
Database:
  ✅ Proper indexes on all query columns
  ✅ N+1 queries eliminated
  ✅ Connection pool properly sized
  ✅ Slow query log enabled
  ✅ Explain/analyze reviewed for complex queries

Application:
  ✅ Caching for read-heavy data
  ✅ Async I/O for parallel operations
  ✅ Response compression enabled
  ✅ open-in-view: false
  ✅ Lazy loading properly configured

Infrastructure:
  ✅ CDN for static assets
  ✅ Redis for session/cache
  ✅ Horizontal scaling ready
  ✅ JVM heap properly sized
  ✅ Garbage collection tuned
```

---

*[← Part 27: API Documentation](./part-27-api-documentation.md) | [Part 29: Advanced Testing →](./part-29-advanced-testing.md)*

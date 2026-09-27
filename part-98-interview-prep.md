# Part 98: Spring Boot Interview Preparation
## ขั้นตอนที่ 3521-3560

**ระดับ: World-Class Professional**

---

## บทนำ: การเตรียมตัวสัมภาษณ์งาน

การสัมภาษณ์งานในตำแหน่ง Senior/Lead Spring Boot Developer จะครอบคลุมหลายด้าน ตั้งแต่ความรู้พื้นฐาน, System Design, Performance Tuning, จนถึง Leadership

คู่มือนี้จะรวบรวมคำถามที่พบบ่อยพร้อมคำตอบที่ครบถ้วนและลึกซึ้ง

---

## ขั้นตอนที่ 3521: Core Spring Boot Questions

### Q1: Spring Boot vs Spring Framework ต่างกันอย่างไร?

**คำตอบ:**

Spring Framework เป็น Core Framework ที่ต้องการ Configuration จำนวนมาก ทั้ง XML และ Java Config

Spring Boot เพิ่ม:
1. **Auto-configuration** - Config อัตโนมัติตาม Dependencies ที่มี
2. **Starter Dependencies** - Bundle ของ Dependencies ที่ทำงานร่วมกันได้
3. **Embedded Server** - ไม่ต้อง Deploy WAR ไปยัง External Tomcat
4. **Production-ready Features** - Actuator, Health Checks, Metrics

```java
// Spring Framework (ต้องกำหนด Config เอง)
@Configuration
@EnableWebMvc
@ComponentScan("com.example")
public class WebConfig implements WebMvcConfigurer {
    @Bean
    public ViewResolver viewResolver() {
        // Manual Config...
    }
    
    @Bean
    public DataSource dataSource() {
        // Manual DB Config...
    }
}

// Spring Boot (Auto-config ทำให้อัตโนมัติ)
@SpringBootApplication  // = @Configuration + @EnableAutoConfiguration + @ComponentScan
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
// application.properties เพียงพอสำหรับ Config ส่วนใหญ่
```

---

### Q2: @SpringBootApplication ทำงานอย่างไร?

**คำตอบ:**

```java
// @SpringBootApplication เป็น Meta-annotation ที่รวม 3 annotations:
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootConfiguration    // = @Configuration
@EnableAutoConfiguration    // เปิดใช้ Auto-configuration
@ComponentScan              // Scan @Component ใน Package เดียวกันและ Sub-packages
public @interface SpringBootApplication {
    // ...
}

// Auto-configuration ทำงานอย่างไร?
// 1. อ่าน META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
// 2. Load AutoConfiguration Classes ที่ตรงตามเงื่อนไข @ConditionalOn*
// 3. Create Beans ตาม Configuration

// ตัวอย่าง: DataSourceAutoConfiguration
@AutoConfiguration
@ConditionalOnClass({ DataSource.class, EmbeddedDatabaseType.class })
@ConditionalOnMissingBean(type = "io.r2dbc.spi.ConnectionFactory")
@EnableConfigurationProperties(DataSourceProperties.class)
public class DataSourceAutoConfiguration {
    
    @Bean
    @ConditionalOnMissingBean(DataSource.class)  // สร้างเฉพาะถ้าไม่มี Bean นี้อยู่แล้ว
    @ConditionalOnProperty(name = "spring.datasource.url")
    public DataSource dataSource(DataSourceProperties properties) {
        return properties.initializeDataSourceBuilder().build();
    }
}
```

---

### Q3: Dependency Injection Types ใน Spring

```java
// 1. Constructor Injection (RECOMMENDED - Best Practice)
@Service
public class UserService {
    private final UserRepository repository;
    private final PasswordEncoder encoder;
    
    // @Autowired ไม่จำเป็นถ้ามี Constructor เดียว (Spring 4.3+)
    public UserService(UserRepository repository, PasswordEncoder encoder) {
        this.repository = repository;
        this.encoder = encoder;
    }
    // ข้อดี: Immutable, Easy to test, No circular dependency issues at startup
}

// 2. Setter Injection (สำหรับ Optional Dependencies)
@Service
public class EmailService {
    private EmailValidator validator;
    
    @Autowired
    public void setValidator(EmailValidator validator) {
        this.validator = validator;
    }
}

// 3. Field Injection (AVOID - ยากต่อการ Test)
@Service
public class BadService {
    @Autowired  // ไม่แนะนำ
    private SomeRepository repository;
    // ปัญหา: ไม่สามารถทำ Unit Test ได้โดยตรง
}
```

---

## ขั้นตอนที่ 3522: Spring Data JPA Questions

### Q4: Difference between @Transactional on Class vs Method?

```java
@Service
@Transactional(readOnly = true)  // Class-level: Default สำหรับทุก Method
public class ProductService {

    // Inherit class-level: readOnly = true
    public Product findById(UUID id) {
        return productRepository.findById(id)
            .orElseThrow(() -> new ProductNotFoundException(id));
    }

    // Override class-level: readOnly = false (สำหรับ Write operations)
    @Transactional  // Override เป็น readOnly = false
    public Product createProduct(CreateProductCommand command) {
        Product product = Product.create(command);
        return productRepository.save(product);
    }

    // Rollback on specific exceptions
    @Transactional(rollbackFor = PaymentException.class)
    public void processPaymentAndUpdateInventory(UUID orderId) {
        // ถ้าเกิด PaymentException จะ Rollback ทุกอย่าง
    }
}
```

### Q5: N+1 Problem และวิธีแก้ไข

```java
// PROBLEM: N+1 Query
@Entity
public class Order {
    @OneToMany(fetch = FetchType.LAZY)  // Default: Lazy
    private List<OrderItem> items;
}

// Service Code ที่ทำให้เกิด N+1
public List<OrderDto> getAllOrders() {
    List<Order> orders = orderRepository.findAll();  // Query 1: SELECT * FROM orders
    
    return orders.stream()
        .map(order -> {
            // Query N: SELECT * FROM order_items WHERE order_id = ?
            // (เรียกเป็น N ครั้ง สำหรับแต่ละ Order!)
            int itemCount = order.getItems().size();
            return new OrderDto(order, itemCount);
        })
        .toList();
}

// SOLUTION 1: JOIN FETCH (แนะนำสำหรับ Small Dataset)
@Query("SELECT o FROM Order o JOIN FETCH o.items WHERE o.userId = :userId")
List<Order> findByUserIdWithItems(@Param("userId") UUID userId);

// SOLUTION 2: @EntityGraph (สะอาดกว่า JPQL)
@EntityGraph(attributePaths = {"items", "items.product"})
List<Order> findByUserId(UUID userId);

// SOLUTION 3: Batch Size (สำหรับ Collection)
@Entity
public class Order {
    @OneToMany(fetch = FetchType.LAZY)
    @BatchSize(size = 100)  // โหลด 100 orders' items ในครั้งเดียว
    private List<OrderItem> items;
}

// SOLUTION 4: DTO Projection (ดีที่สุดสำหรับ Read-only)
@Query("""
    SELECT new com.example.dto.OrderSummaryDto(
        o.id, o.orderNumber, o.status, COUNT(i)
    )
    FROM Order o 
    LEFT JOIN o.items i
    WHERE o.userId = :userId
    GROUP BY o.id, o.orderNumber, o.status
""")
List<OrderSummaryDto> findOrderSummariesByUserId(@Param("userId") UUID userId);
```

---

## ขั้นตอนที่ 3523: System Design Questions

### Q6: Design Twitter (ระบบ Social Media)

**ตอบโดยแบ่งเป็น 3 ส่วน:**

**1. Clarify Requirements**
```
Functional:
- Post Tweet (280 characters)
- Follow/Unfollow Users
- Timeline: Home (Following) + Personal
- Like, Retweet
- Search Tweets

Non-Functional:
- 100M Daily Active Users
- 500M Tweets/day
- Read:Write = 100:1 (Heavy Read)
- Latency: p99 < 200ms
- Availability: 99.99%
```

**2. Capacity Estimation**
```
Write:
- 500M tweets/day = 5,787 tweets/second
- Assume 280 bytes/tweet
- Storage: 500M × 280 bytes = 140GB/day

Read:
- 100M DAU × 50 reads/day = 5B reads/day = 57,870 reads/second
```

**3. Architecture Design**
```java
// High-level Components:
// 1. Tweet Service - Post and store tweets
// 2. Timeline Service - Generate and serve timelines
// 3. Fan-out Service - Push tweets to followers
// 4. Search Service - Full-text search

// Timeline Approach: Fan-out on Write vs Fan-out on Read

// Fan-out on Write (Push Model)
// - เมื่อ User โพสต์ Tweet, ส่งไปยัง Timeline ของทุก Follower
// - ข้อดี: Read เร็ว (ดึงจาก Pre-computed Timeline)
// - ข้อเสีย: Write ช้า, ปัญหากับ Celebrity Users (ล้าน Followers)

// Fan-out on Read (Pull Model)
// - อ่าน Tweets จาก Followings แบบ Real-time
// - ข้อดี: Write เร็ว, ไม่มีปัญหา Celebrity
// - ข้อเสีย: Read ช้า

// Hybrid Approach (Twitter ใช้จริง)
// - Regular users: Fan-out on Write
// - Celebrity users (>1M followers): Fan-out on Read
// - Merge ทั้งสองตอน Read

@Service
public class TimelineService {
    
    public List<Tweet> getHomeTimeline(UUID userId, int page) {
        // 1. Get pre-computed timeline from Redis
        List<String> tweetIds = redisTemplate.opsForList()
            .range("timeline:" + userId, 
                   (long) page * PAGE_SIZE, 
                   (long) (page + 1) * PAGE_SIZE - 1);
        
        // 2. Get celebrity tweets (fan-out on read)
        List<UUID> celebrities = followRepository.findCelebrity(userId);
        List<Tweet> celebrityTweets = fetchRecentTweets(celebrities);
        
        // 3. Merge and sort by time
        List<Tweet> regularTweets = fetchTweetsByIds(tweetIds);
        return mergeAndSort(regularTweets, celebrityTweets);
    }
}
```

---

### Q7: Design Amazon Order System

```
Requirements:
- Handle 1M orders/day
- ACID for payment
- Real-time inventory
- Order status tracking
- Notification on status change

Architecture:

┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   Order     │───▶│  Payment    │───▶│  Inventory  │
│   Service   │    │  Service    │    │  Service    │
└─────────────┘    └─────────────┘    └─────────────┘
       │                  │                  │
       └──────────────────┼──────────────────┘
                          ▼
                   ┌─────────────┐
                   │   Kafka     │
                   └─────────────┘
                          │
                   ┌─────────────┐
                   │Notification │
                   │   Service   │
                   └─────────────┘
```

```java
// SAGA Pattern for Distributed Transaction
@Component
public class CreateOrderSaga {
    
    // Step 1: Create Order (PENDING)
    // Step 2: Reserve Inventory
    // Step 3: Process Payment
    // Step 4: Confirm Order
    // 
    // Compensating Transactions (Rollback):
    // If Payment fails -> Release Inventory -> Cancel Order
    
    public void execute(CreateOrderCommand command) {
        OrderSagaState state = OrderSagaState.builder()
            .orderId(UUID.randomUUID())
            .command(command)
            .step(SagaStep.CREATE_ORDER)
            .build();
        
        sagaStateRepository.save(state);
        eventBus.publish(new OrderSagaStarted(state.getOrderId()));
    }
    
    @EventHandler
    public void onInventoryReserved(InventoryReservedEvent event) {
        // ต่อไปทำ Payment
        eventBus.publish(new ProcessPaymentCommand(event.getOrderId()));
    }
    
    @EventHandler
    public void onPaymentFailed(PaymentFailedEvent event) {
        // Compensate: ปล่อย Inventory ที่ Reserve ไว้
        eventBus.publish(new ReleaseInventoryCommand(event.getOrderId()));
        eventBus.publish(new CancelOrderCommand(event.getOrderId()));
    }
}
```

---

## ขั้นตอนที่ 3524: Performance Questions

### Q8: How to Handle 10x Traffic Spike?

```java
// Strategy 1: Horizontal Scaling
// - Kubernetes HPA ช่วย Auto-scale ตาม CPU/Memory/Custom Metrics
// - Stateless Services ทำ Scale ได้ง่าย

// Strategy 2: Caching
@Service
public class ProductService {
    
    @Cacheable(value = "products", key = "#id", unless = "#result == null")
    public Product findById(UUID id) {
        return productRepository.findById(id).orElse(null);
    }
    
    @CacheEvict(value = "products", key = "#product.id")
    @Transactional
    public Product update(Product product) {
        return productRepository.save(product);
    }
}

// Strategy 3: Database Connection Pool Tuning
spring:
  datasource:
    hikari:
      maximum-pool-size: 50     # เพิ่มจาก Default 10
      minimum-idle: 10
      connection-timeout: 3000  # 3s timeout

// Strategy 4: Async Processing
@Service
public class OrderService {
    
    @Async("orderProcessingExecutor")
    public CompletableFuture<Order> processOrderAsync(CreateOrderCommand command) {
        // Long-running processing ทำใน Background Thread
        return CompletableFuture.completedFuture(processOrder(command));
    }
}

@Bean("orderProcessingExecutor")
public Executor orderProcessingExecutor() {
    ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
    executor.setCorePoolSize(10);
    executor.setMaxPoolSize(50);
    executor.setQueueCapacity(500);
    executor.setThreadNamePrefix("order-async-");
    executor.initialize();
    return executor;
}

// Strategy 5: Circuit Breaker
@Component
public class ProductClient {
    
    @CircuitBreaker(name = "productService", fallbackMethod = "getProductFallback")
    @TimeLimiter(name = "productService")
    public Product getProduct(UUID productId) {
        return productServiceClient.getProduct(productId);
    }
    
    public Product getProductFallback(UUID productId, Exception e) {
        // Return cached version หรือ Default Product
        return cache.get(productId);
    }
}
```

---

## ขั้นตอนที่ 3525: Code Review Scenarios

### Q9: Review This Code - What's Wrong?

```java
// Code ที่ให้ Review:
@RestController
@RequestMapping("/api/users")
public class UserController {
    
    @Autowired
    private UserService userService;
    
    @GetMapping
    public List<User> getAllUsers() {
        return userService.findAll();  // โหลด User ทั้งหมด!
    }
    
    @PostMapping
    public User createUser(@RequestBody User user) {
        if (userService.findByEmail(user.getEmail()) != null) {
            throw new RuntimeException("Email exists");
        }
        return userService.save(user);
    }
    
    @DeleteMapping("/{id}")
    public void deleteUser(@PathVariable Long id) {
        userService.deleteById(id);
    }
}
```

**ปัญหาที่พบ:**

```java
// ปัญหา 1: Field Injection แทน Constructor Injection
// ปัญหา 2: getAllUsers() ไม่มี Pagination - Memory Overflow
// ปัญหา 3: User Entity Return โดยตรง - Expose Internal Structure
// ปัญหา 4: Race Condition ใน createUser (Check-then-Act without Lock)
// ปัญหา 5: RuntimeException ไม่เฉพาะเจาะจง
// ปัญหา 6: deleteUser ไม่ตรวจสอบว่า User มีอยู่จริงหรือไม่
// ปัญหา 7: ไม่มี HTTP Status Code ที่ถูกต้อง

// Code ที่ถูกต้อง:
@RestController
@RequestMapping("/api/v1/users")
@RequiredArgsConstructor  // Constructor Injection
@Slf4j
public class UserController {
    
    private final UserService userService;  // final + Constructor Injection
    
    @GetMapping
    public ResponseEntity<Page<UserResponse>> getAllUsers(
        @RequestParam(defaultValue = "0") int page,
        @RequestParam(defaultValue = "20") int size,
        @RequestParam(defaultValue = "createdAt") String sortBy
    ) {
        Pageable pageable = PageRequest.of(page, size, Sort.by(sortBy).descending());
        Page<UserResponse> users = userService.findAll(pageable);
        return ResponseEntity.ok(users);
    }
    
    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public ApiResponse<UserResponse> createUser(
        @Valid @RequestBody CreateUserRequest request
    ) {
        // Uniqueness check ทำภายใน @Transactional Service
        UserResponse created = userService.createUser(request);
        return ApiResponse.success(created, "User created successfully");
    }
    
    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void deleteUser(@PathVariable UUID id) {
        userService.deleteUser(id);  // Service จัดการ NotFound exception
    }
}
```

---

## ขั้นตอนที่ 3526: Architecture Decision Questions

### Q10: Microservices vs Monolith - When to choose?

```
เลือก Monolith เมื่อ:
1. Team เล็ก (< 10 developers)
2. Business Domain ยังไม่ชัดเจน
3. Startup Phase - ต้องการ Move Fast
4. Complexity ต่ำ

เลือก Microservices เมื่อ:
1. Team ใหญ่ (> 20 developers) แบ่งเป็น Feature Teams
2. Scale ต่างกันตาม Service (Product Service ต้องการ Scale มากกว่า Admin Service)
3. Technology Requirements ต่างกัน (Search = Elasticsearch, Payment = Special Security)
4. Deployment Frequency สูง - ต้องการ Deploy อิสระ

Modular Monolith (Middle Ground):
- เริ่มด้วย Well-structured Monolith
- แบ่ง Module ตาม Domain
- ง่ายต่อการ Extract เป็น Microservices ในอนาคต
```

```java
// Modular Monolith Structure
com.ecommerce/
├── user/
│   ├── domain/
│   ├── application/
│   └── infrastructure/
├── product/
│   ├── domain/
│   ├── application/
│   └── infrastructure/
└── order/
    ├── domain/
    ├── application/
    └── infrastructure/

// Module ต่างๆ communicate ผ่าน Interface
// ไม่ Direct Access ถึง Repository ของกัน
@ApplicationScoped
public interface UserFacade {
    UserInfo getUserInfo(UUID userId);
    boolean userExists(UUID userId);
}
```

---

## ขั้นตอนที่ 3527: Performance Debugging Scenarios

### Q11: Your Service Suddenly Slows Down - How to Debug?

**กระบวนการ Debug:**

```
Step 1: Check Metrics Dashboard
- CPU/Memory/Disk: Grafana/Datadog
- Response Time: อะไร Slow?
- Error Rate: มี Error หรือเปล่า?

Step 2: Check Logs
- ERROR/WARN Logs ช่วง Time ที่ Slow
- Slow Query Logs ใน Database

Step 3: Check Active Connections
- Database Connection Pool
- External Service Calls

Step 4: Thread Dump Analysis
- jstack <PID>
- ดู Thread State: BLOCKED, WAITING
```

```bash
# Check Thread State
jstack $(pgrep -f "spring-boot") | grep -A 5 "BLOCKED\|WAITING"

# Check DB Slow Queries (PostgreSQL)
SELECT query, calls, total_time, mean_time
FROM pg_stat_statements
ORDER BY mean_time DESC
LIMIT 10;

# Check Active Connections
SELECT state, count(*) 
FROM pg_stat_activity 
GROUP BY state;
```

```java
// Common Causes and Solutions:

// Cause 1: Missing Database Index
// Diagnosis: Slow queries, High DB CPU
// Solution:
@Entity
@Table(name = "orders", indexes = {
    @Index(name = "idx_orders_user_status", columnList = "user_id, status"),
    @Index(name = "idx_orders_created_at", columnList = "created_at DESC")
})
public class Order { ... }

// Cause 2: Connection Pool Exhausted  
// Diagnosis: Thread WAITING for connection, Timeout errors
// Solution: Increase pool size, find connection leak
@PostConstruct
public void logPoolStats() {
    // Monitor HikariCP stats
    HikariDataSource ds = (HikariDataSource) dataSource;
    log.info("Active: {}, Idle: {}, Waiting: {}",
        ds.getHikariPoolMXBean().getActiveConnections(),
        ds.getHikariPoolMXBean().getIdleConnections(),
        ds.getHikariPoolMXBean().getThreadsAwaitingConnection()
    );
}

// Cause 3: Memory Leak
// Diagnosis: OOM errors, Increasing memory over time
// Solution: Heap Dump Analysis
// java -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/tmp/heap.hprof
```

---

## ขั้นตอนที่ 3528: Security Questions

### Q12: How to Prevent Common Security Vulnerabilities?

```java
// 1. SQL Injection Prevention
// BAD:
public User findUser(String username) {
    String sql = "SELECT * FROM users WHERE username = '" + username + "'";
    // SQL Injection: username = "'; DROP TABLE users;--"
    return jdbcTemplate.queryForObject(sql, User.class);
}

// GOOD: Parameterized Queries
public User findUser(String username) {
    return userRepository.findByUsername(username).orElse(null);
    // JPA uses PreparedStatement automatically
}

// 2. XSS Prevention
@Bean
public FilterRegistrationBean<XssFilter> xssFilter() {
    FilterRegistrationBean<XssFilter> bean = new FilterRegistrationBean<>();
    bean.setFilter(new XssFilter());
    bean.addUrlPatterns("/*");
    return bean;
}

// 3. IDOR (Insecure Direct Object Reference)
// BAD:
@GetMapping("/orders/{orderId}")
public Order getOrder(@PathVariable UUID orderId) {
    return orderRepository.findById(orderId)  // ใครก็เข้าถึงได้!
        .orElseThrow();
}

// GOOD:
@GetMapping("/orders/{orderId}")
public Order getOrder(
    @PathVariable UUID orderId,
    @AuthenticationPrincipal UserDetails currentUser
) {
    Order order = orderRepository.findById(orderId)
        .orElseThrow(() -> new OrderNotFoundException(orderId));
    
    // ตรวจสอบว่า Order เป็นของ User นี้จริงๆ
    if (!order.getUserId().equals(UUID.fromString(currentUser.getUsername()))) {
        throw new AccessDeniedException("You don't have access to this order");
    }
    
    return order;
}

// 4. Rate Limiting
@Component
public class RateLimitingFilter extends OncePerRequestFilter {
    
    private final Map<String, Bucket> cache = new ConcurrentHashMap<>();
    
    @Override
    protected void doFilterInternal(HttpServletRequest request, 
                                     HttpServletResponse response,
                                     FilterChain chain) throws IOException, ServletException {
        String clientId = getClientId(request);
        Bucket bucket = cache.computeIfAbsent(clientId, this::createNewBucket);
        
        if (bucket.tryConsume(1)) {
            chain.doFilter(request, response);
        } else {
            response.setStatus(429);
            response.getWriter().write("{\"error\": \"Too Many Requests\"}");
        }
    }
    
    private Bucket createNewBucket(String clientId) {
        return Bucket.builder()
            .addLimit(Bandwidth.classic(100, Refill.greedy(100, Duration.ofMinutes(1))))
            .build();
    }
}
```

---

## ขั้นตอนที่ 3529: Practical Coding Challenges

### Challenge 1: Implement LRU Cache

```java
// ถามบ่อยมาก: Implement LRU Cache
public class LRUCache<K, V> {
    
    private final int capacity;
    private final LinkedHashMap<K, V> cache;
    
    public LRUCache(int capacity) {
        this.capacity = capacity;
        // accessOrder = true: เรียงตามการ Access ล่าสุด
        this.cache = new LinkedHashMap<>(capacity, 0.75f, true) {
            @Override
            protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
                return size() > capacity;
            }
        };
    }
    
    public synchronized V get(K key) {
        return cache.getOrDefault(key, null);
    }
    
    public synchronized void put(K key, V value) {
        cache.put(key, value);
    }
}

// Spring Boot Integration
@Configuration
public class CacheConfig {
    
    @Bean
    public CacheManager cacheManager() {
        CaffeineCacheManager manager = new CaffeineCacheManager();
        manager.setCaffeine(
            Caffeine.newBuilder()
                .maximumSize(10_000)
                .expireAfterWrite(Duration.ofMinutes(30))
                .recordStats()
        );
        return manager;
    }
}
```

### Challenge 2: Distributed Rate Limiter

```java
// Distributed Rate Limiter using Redis
@Component
public class DistributedRateLimiter {
    
    private final RedisTemplate<String, Long> redisTemplate;
    
    /**
     * Sliding Window Algorithm
     * แม่นยำกว่า Fixed Window แต่ใช้ Memory มากกว่า
     */
    public boolean isAllowed(String clientId, int maxRequests, Duration window) {
        String key = "rate:" + clientId;
        long now = System.currentTimeMillis();
        long windowStart = now - window.toMillis();
        
        String luaScript = """
            local key = KEYS[1]
            local now = tonumber(ARGV[1])
            local windowStart = tonumber(ARGV[2])
            local maxRequests = tonumber(ARGV[3])
            
            -- Remove old requests outside window
            redis.call('ZREMRANGEBYSCORE', key, '-inf', windowStart)
            
            -- Count current requests in window
            local count = redis.call('ZCARD', key)
            
            if count < maxRequests then
                -- Add current request
                redis.call('ZADD', key, now, now)
                redis.call('EXPIRE', key, math.ceil(tonumber(ARGV[4]) / 1000))
                return 1  -- Allowed
            else
                return 0  -- Rate limited
            end
        """;
        
        Long result = redisTemplate.execute(
            RedisScript.of(luaScript, Long.class),
            List.of(key),
            now, windowStart, maxRequests, window.toMillis()
        );
        
        return result != null && result == 1;
    }
}
```

---

## ขั้นตอนที่ 3530: Kafka Deep Dive Questions

### Q13: Kafka Partition Strategy

```java
// Custom Partitioner สำหรับ Guarantee Ordering ต่อ Order
public class OrderPartitioner implements Partitioner {
    
    @Override
    public int partition(String topic, Object key, byte[] keyBytes, 
                         Object value, byte[] valueBytes, Cluster cluster) {
        
        List<PartitionInfo> partitions = cluster.partitionsForTopic(topic);
        int numPartitions = partitions.size();
        
        if (key instanceof String orderId) {
            // ทุก Event ของ Order เดียวกัน จะไปที่ Partition เดียวกัน
            // Guarantee Ordering ต่อ Order
            return Math.abs(orderId.hashCode()) % numPartitions;
        }
        
        // Random partition สำหรับ key อื่น
        return (int) (System.currentTimeMillis() % numPartitions);
    }
}

// Consumer Configuration
@Configuration
public class KafkaConsumerConfig {
    
    @Bean
    public ConsumerFactory<String, Object> consumerFactory() {
        Map<String, Object> props = new HashMap<>();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
        props.put(ConsumerConfig.GROUP_ID_CONFIG, "order-service");
        
        // At-least-once delivery
        props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);
        props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
        
        // Performance tuning
        props.put(ConsumerConfig.MAX_POLL_RECORDS_CONFIG, 500);
        props.put(ConsumerConfig.FETCH_MIN_BYTES_CONFIG, 1024 * 1024); // 1MB
        props.put(ConsumerConfig.FETCH_MAX_WAIT_MS_CONFIG, 500);
        
        return new DefaultKafkaConsumerFactory<>(props);
    }
}
```

---

## ขั้นตอนที่ 3531: Behavioral Interview Questions

### Q14: Tell me about a challenging technical problem you solved

**Framework: STAR (Situation, Task, Action, Result)**

```
Situation: 
ระบบ Payment Processing มี Response Time เพิ่มขึ้นจาก 50ms เป็น 2000ms
ในช่วง Peak Hour (ทุกวันศุกร์ 18:00-20:00)

Task:
ต้องหาสาเหตุและแก้ไขให้ Response Time กลับมาที่ < 200ms
ภายใน 1 Sprint

Action:
1. ตรวจ Grafana Dashboard พบ Database Query Time สูงผิดปกติ
2. เปิด pg_stat_statements พบว่า Query หนึ่งใช้เวลา 1.8s
3. ใช้ EXPLAIN ANALYZE ดูว่า Sequential Scan แทน Index Scan
4. พบว่า Table "payments" มี 50M rows และ Query ขาด Index บน (user_id, created_at)
5. สร้าง Index ใหม่: CREATE INDEX CONCURRENTLY idx_payments_user_created ON payments(user_id, created_at DESC)
6. ใช้ CONCURRENTLY เพื่อไม่ Lock Table ระหว่างสร้าง Index

Result:
- Response Time ลดลงจาก 2000ms เป็น 45ms (96% improvement)
- ไม่มี Downtime
- Deploy ใน Production ไม่กระทบ User
```

---

## ขั้นตอนที่ 3532: Advanced Topics Questions

### Q15: Explain CQRS and Event Sourcing

```java
// CQRS = Command Query Responsibility Segregation
// แยก "Write Operations" (Commands) ออกจาก "Read Operations" (Queries)

// Command Side (Write) - มุ่งเน้น Consistency
@Service
public class OrderCommandHandler {
    
    @CommandHandler
    @Transactional
    public UUID handle(CreateOrderCommand command) {
        Order order = Order.create(command);
        orderRepository.save(order);
        
        // Publish Event for Read Side
        eventPublisher.publish(new OrderCreatedEvent(order));
        
        return order.getId();
    }
}

// Query Side (Read) - มุ่งเน้น Performance
@Service
public class OrderQueryService {
    
    // Read from denormalized "Read Model" (ปรับให้เหมาะกับ Query)
    public OrderDetailView getOrderDetail(UUID orderId) {
        // Read จาก table ที่ denormalize แล้ว
        // Join กันไว้แล้ว ดึงข้อมูลครั้งเดียวได้เลย
        return orderDetailViewRepository.findById(orderId)
            .orElseThrow();
    }
}

// Event Sourcing
// แทนที่จะเก็บ Current State ให้เก็บ Sequence ของ Events ทั้งหมด
// State = Replay ของ Events ทั้งหมด

@EventSourcingHandler
public class OrderAggregate {
    
    private UUID id;
    private OrderStatus status;
    private BigDecimal totalAmount;
    
    @CommandHandler
    public OrderAggregate(CreateOrderCommand command) {
        apply(new OrderCreatedEvent(command.getOrderId(), command));
    }
    
    @EventSourcingHandler
    public void on(OrderCreatedEvent event) {
        this.id = event.getOrderId();
        this.status = OrderStatus.PENDING;
        this.totalAmount = event.getTotalAmount();
    }
    
    @CommandHandler
    public void handle(ConfirmOrderCommand command) {
        if (status != OrderStatus.PENDING) {
            throw new InvalidOrderStatusException();
        }
        apply(new OrderConfirmedEvent(id));
    }
    
    @EventSourcingHandler
    public void on(OrderConfirmedEvent event) {
        this.status = OrderStatus.CONFIRMED;
    }
}
```

---

## สรุป Part 98

คำถาม Interview ที่ต้องเตรียม:

1. **Core Concepts** - Spring Boot, DI, Auto-configuration
2. **Spring Data JPA** - N+1, @Transactional, Pagination
3. **System Design** - Twitter, Amazon, Design Patterns
4. **Performance** - Caching, Scaling, Profiling
5. **Security** - OWASP Top 10, JWT, Rate Limiting
6. **Kafka** - Partitioning, Consumer Groups, Ordering
7. **Architecture** - CQRS, Event Sourcing, SAGA
8. **Coding Challenges** - LRU Cache, Rate Limiter
9. **Behavioral** - STAR Method, Technical Stories

---

*[← Part 97: Capstone Implementation](./part-97-capstone-implementation.md) | [Part 99: Career Growth →](./part-99-career-growth.md)*

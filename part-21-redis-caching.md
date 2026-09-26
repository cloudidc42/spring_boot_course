# Part 21: Redis & Caching
## ขั้นตอนที่ 546-575

> **ระดับ:** กลาง-สูง (Intermediate-Advanced)  
> **เวลาเรียน:** 5-6 ชั่วโมง  
> **เป้าหมาย:** ใช้ Redis สำหรับ Caching, Session, Rate Limiting

---

## ขั้นตอนที่ 546: Redis Setup

```yaml
# docker-compose.yml
services:
  redis:
    image: redis:7-alpine
    container_name: spring_redis
    ports:
      - "6379:6379"
    command: redis-server --maxmemory 256mb --maxmemory-policy allkeys-lru
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
```

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-cache</artifactId>
</dependency>
```

```yaml
# application.yml
spring:
  data:
    redis:
      host: ${REDIS_HOST:localhost}
      port: ${REDIS_PORT:6379}
      password: ${REDIS_PASSWORD:}
      timeout: 2000ms
      lettuce:
        pool:
          max-active: 10
          max-idle: 5
          min-idle: 2
  cache:
    type: redis
    redis:
      time-to-live: 600000  # 10 minutes default
      cache-null-values: false
      key-prefix: "myapp:"
      use-key-prefix: true
```

---

## ขั้นตอนที่ 547: Redis Configuration

```java
@Configuration
@EnableCaching
@RequiredArgsConstructor
public class RedisConfig {
    
    @Bean
    public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory factory) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(factory);
        
        // Key serializer
        template.setKeySerializer(new StringRedisSerializer());
        template.setHashKeySerializer(new StringRedisSerializer());
        
        // Value serializer (JSON)
        GenericJackson2JsonRedisSerializer jsonSerializer = new GenericJackson2JsonRedisSerializer();
        template.setValueSerializer(jsonSerializer);
        template.setHashValueSerializer(jsonSerializer);
        
        template.afterPropertiesSet();
        return template;
    }
    
    @Bean
    public CacheManager cacheManager(RedisConnectionFactory factory) {
        RedisCacheConfiguration defaultConfig = RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(10))
            .serializeKeysWith(RedisSerializationContext.SerializationPair.fromSerializer(new StringRedisSerializer()))
            .serializeValuesWith(RedisSerializationContext.SerializationPair.fromSerializer(new GenericJackson2JsonRedisSerializer()))
            .disableCachingNullValues()
            .prefixCacheNameWith("myapp:");
        
        Map<String, RedisCacheConfiguration> cacheConfigs = Map.of(
            "products", defaultConfig.entryTtl(Duration.ofMinutes(30)),
            "users", defaultConfig.entryTtl(Duration.ofMinutes(60)),
            "categories", defaultConfig.entryTtl(Duration.ofHours(24)),
            "homepage", defaultConfig.entryTtl(Duration.ofMinutes(5))
        );
        
        return RedisCacheManager.builder(factory)
            .cacheDefaults(defaultConfig)
            .withInitialCacheConfigurations(cacheConfigs)
            .build();
    }
}
```

---

## ขั้นตอนที่ 548: Spring Cache Annotations

```java
@Service
@RequiredArgsConstructor
@Transactional(readOnly = true)
@Slf4j
public class ProductService {
    
    private final ProductRepository productRepository;
    private final ProductMapper productMapper;
    
    // ===== @Cacheable - cache results =====
    
    @Cacheable(value = "products", key = "#id")
    public ProductDetailResponse findById(Long id) {
        log.debug("Cache miss for product: {}", id);
        Product product = productRepository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("Product", "id", id));
        return productMapper.toDetailResponse(product);
    }
    
    // Complex key
    @Cacheable(value = "products", key = "#search.keyword + '_' + #pageable.pageNumber + '_' + #pageable.pageSize")
    public Page<ProductResponse> search(ProductSearchRequest search, Pageable pageable) {
        // ...
    }
    
    // Conditional caching
    @Cacheable(value = "products", key = "#id", condition = "#id > 0")
    public ProductDetailResponse findByIdConditional(Long id) { ... }
    
    // Unless - don't cache if result is null or condition
    @Cacheable(value = "products", key = "#name", unless = "#result == null")
    public Optional<ProductResponse> findByName(String name) { ... }
    
    // ===== @CachePut - update cache =====
    
    @CachePut(value = "products", key = "#result.id")
    @Transactional
    public ProductDetailResponse update(Long id, UpdateProductRequest request) {
        // Updates cache with new value after method executes
        Product product = productRepository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("Product", "id", id));
        productMapper.updateFromRequest(request, product);
        return productMapper.toDetailResponse(productRepository.save(product));
    }
    
    // ===== @CacheEvict - remove from cache =====
    
    @CacheEvict(value = "products", key = "#id")
    @Transactional
    public void delete(Long id) {
        productRepository.deleteById(id);
    }
    
    // Evict all entries in cache
    @CacheEvict(value = "products", allEntries = true)
    public void clearProductCache() {
        log.info("Product cache cleared");
    }
    
    // ===== @Caching - multiple cache operations =====
    
    @Caching(evict = {
        @CacheEvict(value = "products", key = "#id"),
        @CacheEvict(value = "products", key = "'search_*'", allEntries = true)
    })
    @Transactional
    public ProductResponse updateWithEvict(Long id, UpdateProductRequest request) { ... }
}
```

---

## ขั้นตอนที่ 549: Manual Redis Operations

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class RedisService {
    
    private final RedisTemplate<String, Object> redisTemplate;
    private final StringRedisTemplate stringRedisTemplate;
    
    // ===== String Operations =====
    
    public void set(String key, Object value, Duration ttl) {
        redisTemplate.opsForValue().set(key, value, ttl);
    }
    
    public <T> Optional<T> get(String key, Class<T> type) {
        Object value = redisTemplate.opsForValue().get(key);
        return Optional.ofNullable(type.cast(value));
    }
    
    public boolean delete(String key) {
        return Boolean.TRUE.equals(redisTemplate.delete(key));
    }
    
    public boolean exists(String key) {
        return Boolean.TRUE.equals(redisTemplate.hasKey(key));
    }
    
    // ===== Counter =====
    
    public Long increment(String key) {
        return redisTemplate.opsForValue().increment(key);
    }
    
    public Long incrementBy(String key, long delta) {
        return redisTemplate.opsForValue().increment(key, delta);
    }
    
    // ===== Hash Operations =====
    
    public void hset(String key, String field, Object value) {
        redisTemplate.opsForHash().put(key, field, value);
    }
    
    public Object hget(String key, String field) {
        return redisTemplate.opsForHash().get(key, field);
    }
    
    public Map<Object, Object> hgetAll(String key) {
        return redisTemplate.opsForHash().entries(key);
    }
    
    // ===== List Operations =====
    
    public void lpush(String key, Object value) {
        redisTemplate.opsForList().leftPush(key, value);
    }
    
    public Object rpop(String key) {
        return redisTemplate.opsForList().rightPop(key);
    }
    
    // ===== Set Operations =====
    
    public void sadd(String key, Object... values) {
        redisTemplate.opsForSet().add(key, values);
    }
    
    public Set<Object> smembers(String key) {
        return redisTemplate.opsForSet().members(key);
    }
    
    // ===== Sorted Set (leaderboard) =====
    
    public void zadd(String key, double score, String member) {
        redisTemplate.opsForZSet().add(key, member, score);
    }
    
    public Set<ZSetOperations.TypedTuple<Object>> zrangeWithScores(String key, long start, long end) {
        return redisTemplate.opsForZSet().rangeWithScores(key, start, end);
    }
    
    // ===== Pub/Sub =====
    
    public void publish(String channel, Object message) {
        redisTemplate.convertAndSend(channel, message);
    }
}
```

---

## ขั้นตอนที่ 550: Cache-Aside Pattern

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class UserService {
    
    private final UserRepository userRepository;
    private final UserMapper userMapper;
    private final RedisTemplate<String, Object> redisTemplate;
    
    private static final String USER_CACHE_KEY = "user:";
    private static final Duration CACHE_TTL = Duration.ofMinutes(30);
    
    public UserResponse findById(Long id) {
        String key = USER_CACHE_KEY + id;
        
        // 1. Check cache
        Object cached = redisTemplate.opsForValue().get(key);
        if (cached instanceof UserResponse user) {
            log.debug("Cache hit: user:{}", id);
            return user;
        }
        
        // 2. Query DB
        log.debug("Cache miss: user:{}", id);
        User user = userRepository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("User", "id", id));
        UserResponse response = userMapper.toResponse(user);
        
        // 3. Store in cache
        redisTemplate.opsForValue().set(key, response, CACHE_TTL);
        
        return response;
    }
    
    public UserResponse update(Long id, UpdateUserRequest request) {
        // ... update logic ...
        
        // Invalidate cache
        redisTemplate.delete(USER_CACHE_KEY + id);
        
        return updatedUser;
    }
}
```

---

## ขั้นตอนที่ 551: Rate Limiting with Redis

```java
@Service
@RequiredArgsConstructor
public class RateLimiterService {
    
    private final RedisTemplate<String, String> redisTemplate;
    
    // Sliding window rate limiter
    public boolean isAllowed(String key, int maxRequests, Duration window) {
        long now = System.currentTimeMillis();
        long windowMs = window.toMillis();
        
        String redisKey = "rate_limit:" + key;
        
        // Use Redis sorted set for sliding window
        ZSetOperations<String, String> zset = redisTemplate.opsForZSet();
        
        // Remove expired entries
        zset.removeRangeByScore(redisKey, 0, now - windowMs);
        
        // Count requests in window
        Long count = zset.size(redisKey);
        
        if (count != null && count >= maxRequests) {
            return false;  // rate limited
        }
        
        // Add current request
        zset.add(redisKey, String.valueOf(now), now);
        redisTemplate.expire(redisKey, window);
        
        return true;
    }
    
    // Fixed window rate limiter (simpler)
    public boolean isAllowedFixed(String key, int maxRequests, Duration window) {
        String redisKey = "rate_fixed:" + key;
        
        Long count = redisTemplate.opsForValue().increment(redisKey);
        
        if (count == 1) {
            redisTemplate.expire(redisKey, window);
        }
        
        return count <= maxRequests;
    }
}

// Use in Controller
@PostMapping("/login")
public ResponseEntity<?> login(@RequestBody LoginRequest request, HttpServletRequest req) {
    String key = req.getRemoteAddr();
    
    if (!rateLimiterService.isAllowed(key, 5, Duration.ofMinutes(1))) {
        return ResponseEntity.status(HttpStatus.TOO_MANY_REQUESTS)
            .header("Retry-After", "60")
            .body(ApiResponse.error("Too many requests"));
    }
    
    return ResponseEntity.ok(ApiResponse.success(authService.login(request)));
}
```

---

## ขั้นตอนที่ 552: Redis Pub/Sub

```java
// Publisher
@Service
@RequiredArgsConstructor
public class EventPublisher {
    
    private final RedisTemplate<String, Object> redisTemplate;
    
    public void publishOrderCreated(OrderCreatedMessage message) {
        redisTemplate.convertAndSend("orders.created", message);
    }
    
    public void publishUserActivity(UserActivityMessage message) {
        redisTemplate.convertAndSend("user.activity", message);
    }
}

// Subscriber
@Component
@RequiredArgsConstructor
@Slf4j
public class OrderCreatedSubscriber {
    
    private final NotificationService notificationService;
    
    @RedisListener(channels = "orders.created")
    public void handleOrderCreated(OrderCreatedMessage message) {
        log.info("Order created event: {}", message.orderId());
        notificationService.sendOrderNotification(message);
    }
}

// Config
@Configuration
public class RedisMessagingConfig {
    
    @Bean
    public RedisMessageListenerContainer container(
        RedisConnectionFactory factory,
        OrderCreatedSubscriber subscriber,
        ObjectMapper objectMapper
    ) {
        RedisMessageListenerContainer container = new RedisMessageListenerContainer();
        container.setConnectionFactory(factory);
        
        container.addMessageListener(
            new MessageListenerAdapter(subscriber, objectMapper),
            new ChannelTopic("orders.created")
        );
        
        return container;
    }
}
```

---

## ขั้นตอนที่ 553: Session Storage in Redis

```java
// Add: spring-session-data-redis
@Configuration
@EnableRedisHttpSession(maxInactiveIntervalInSeconds = 1800)
public class SessionConfig {
    
    @Bean
    public CookieSerializer cookieSerializer() {
        DefaultCookieSerializer serializer = new DefaultCookieSerializer();
        serializer.setCookieName("SESSION");
        serializer.setUseHttpOnlyCookie(true);
        serializer.setUseSecureCookie(true);  // HTTPS only
        serializer.setSameSite("Strict");
        return serializer;
    }
}
```

---

## ขั้นตอนที่ 554-575: Leaderboard with Redis

```java
// Real-time leaderboard
@Service
@RequiredArgsConstructor
public class LeaderboardService {
    
    private final RedisTemplate<String, String> redisTemplate;
    private static final String LEADERBOARD_KEY = "leaderboard:products:views";
    
    public void incrementView(Long productId) {
        redisTemplate.opsForZSet().incrementScore(LEADERBOARD_KEY, String.valueOf(productId), 1);
    }
    
    public List<LeaderboardEntry> getTopProducts(int count) {
        Set<ZSetOperations.TypedTuple<String>> topItems = 
            redisTemplate.opsForZSet().reverseRangeWithScores(LEADERBOARD_KEY, 0, count - 1);
        
        if (topItems == null) return List.of();
        
        return topItems.stream()
            .map(item -> new LeaderboardEntry(
                Long.parseLong(item.getValue()),
                item.getScore().longValue()
            ))
            .toList();
    }
    
    public Long getRank(Long productId) {
        Long rank = redisTemplate.opsForZSet().reverseRank(LEADERBOARD_KEY, String.valueOf(productId));
        return rank != null ? rank + 1 : null;
    }
    
    public record LeaderboardEntry(Long productId, long views) {}
}
```

---

*[← Part 20: Pagination Advanced](./part-20-pagination-advanced.md) | [Part 22: Async & Scheduling →](./part-22-async-scheduling.md)*

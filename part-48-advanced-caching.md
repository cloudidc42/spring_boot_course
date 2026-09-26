# Part 48: Advanced Caching Strategies
## ขั้นตอนที่ 1521-1560

> **ระดับ:** ระดับโลก (World-Class)  
> **เวลาเรียน:** 5-6 ชั่วโมง  
> **เป้าหมาย:** Multi-level caching, cache patterns, cache invalidation

---

## ขั้นตอนที่ 1521: Caching Patterns Overview

```
Cache Patterns:
  1. Cache-Aside (Lazy Loading)
     App checks cache → miss → load DB → write to cache
     
  2. Read-Through
     App reads cache → cache loads from DB if miss
     
  3. Write-Through
     App writes to cache → cache writes to DB
     
  4. Write-Behind (Write-Back)
     App writes to cache → async write to DB (fast!)
     
  5. Refresh-Ahead
     Cache refreshes before expiry

Cache Levels:
  L1 = In-process (JVM heap) - fastest, per-instance
  L2 = Remote Redis - slower, shared across instances
  L3 = CDN - even slower, for public content

Cache Invalidation Strategies:
  TTL      = Time-based expiry
  LRU      = Least Recently Used eviction
  Event    = Invalidate on data change
  Version  = Include version in cache key
```

---

## ขั้นตอนที่ 1522: Two-Level Cache (L1 + L2)

```java
@Configuration
public class TwoLevelCacheConfig {
    
    // L1: Caffeine (in-memory, per-instance)
    @Bean
    public CacheManager caffeineCacheManager() {
        CaffeineCacheManager manager = new CaffeineCacheManager();
        manager.setCaffeine(Caffeine.newBuilder()
            .maximumSize(1000)
            .expireAfterWrite(5, TimeUnit.MINUTES)
            .expireAfterAccess(2, TimeUnit.MINUTES)
            .recordStats());
        return manager;
    }
    
    // L2: Redis (distributed, shared)
    @Bean
    public CacheManager redisCacheManager(RedisConnectionFactory factory) {
        RedisCacheConfiguration config = RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofHours(1))
            .disableCachingNullValues()
            .serializeValuesWith(
                RedisSerializationContext.SerializationPair.fromSerializer(
                    new GenericJackson2JsonRedisSerializer()
                )
            );
        
        Map<String, RedisCacheConfiguration> cacheConfigs = new HashMap<>();
        cacheConfigs.put("products", config.entryTtl(Duration.ofHours(2)));
        cacheConfigs.put("categories", config.entryTtl(Duration.ofDays(1)));
        cacheConfigs.put("users", config.entryTtl(Duration.ofMinutes(30)));
        
        return RedisCacheManager.builder(factory)
            .cacheDefaults(config)
            .withInitialCacheConfigurations(cacheConfigs)
            .build();
    }
}

// Two-level cache service
@Service
@RequiredArgsConstructor
public class TwoLevelCacheService<K, V> {
    
    private final Cache l1Cache;  // Caffeine
    private final Cache l2Cache;  // Redis
    private final Function<K, V> loader;
    
    public V get(K key) {
        // Check L1 first
        V cached = l1Cache.getIfPresent(key);
        if (cached != null) {
            return cached;
        }
        
        // Check L2
        cached = l2Cache.get(key, k -> null);
        if (cached != null) {
            l1Cache.put(key, cached);  // Populate L1
            return cached;
        }
        
        // Load from source
        V value = loader.apply(key);
        if (value != null) {
            l1Cache.put(key, value);
            l2Cache.put(key, value);
        }
        
        return value;
    }
    
    public void invalidate(K key) {
        l1Cache.invalidate(key);
        l2Cache.evict(key);
    }
}
```

---

## ขั้นตอนที่ 1523: Cache Invalidation with Events

```java
// Publish cache invalidation event via Redis Pub/Sub
@Service
@RequiredArgsConstructor
public class CacheInvalidationService {
    
    private final RedisTemplate<String, String> redisTemplate;
    private final CacheManager caffeineCacheManager;
    
    public void invalidateProduct(Long productId) {
        // Invalidate local cache
        Cache cache = caffeineCacheManager.getCache("products");
        if (cache != null) cache.evict(productId);
        
        // Broadcast to other instances
        redisTemplate.convertAndSend("cache:invalidate:products", productId.toString());
    }
    
    public void invalidateAll(String cacheName) {
        Cache cache = caffeineCacheManager.getCache(cacheName);
        if (cache != null) cache.invalidateAll();
        
        redisTemplate.convertAndSend("cache:invalidate:all", cacheName);
    }
}

// Listen for cache invalidation messages
@Component
@RequiredArgsConstructor
public class CacheInvalidationListener {
    
    private final CacheManager caffeineCacheManager;
    
    @Bean
    public MessageListenerAdapter listenerAdapter() {
        return new MessageListenerAdapter(this, "onMessage");
    }
    
    @Bean
    public RedisMessageListenerContainer container(RedisConnectionFactory factory) {
        RedisMessageListenerContainer container = new RedisMessageListenerContainer();
        container.setConnectionFactory(factory);
        container.addMessageListener(listenerAdapter(), new PatternTopic("cache:invalidate:*"));
        return container;
    }
    
    public void onMessage(String message, String channel) {
        if (channel.startsWith("cache:invalidate:all:")) {
            String cacheName = channel.replace("cache:invalidate:all:", "");
            Cache cache = caffeineCacheManager.getCache(cacheName);
            if (cache != null) cache.invalidateAll();
        } else if (channel.startsWith("cache:invalidate:products")) {
            Long id = Long.parseLong(message);
            Cache cache = caffeineCacheManager.getCache("products");
            if (cache != null) cache.evict(id);
        }
    }
}
```

---

## ขั้นตอนที่ 1524: Write-Behind Caching

```java
// Write to cache immediately, async write to DB
@Service
@RequiredArgsConstructor
@Slf4j
public class WriteBehindCacheService {
    
    private final RedisTemplate<String, Object> redisTemplate;
    private final ProductRepository productRepository;
    
    private final Queue<CacheWriteTask> pendingWrites = new ConcurrentLinkedQueue<>();
    
    public void updateProduct(Long id, ProductUpdateRequest request) {
        String cacheKey = "products:" + id;
        
        // Write to cache immediately (fast response)
        Product product = (Product) redisTemplate.opsForValue().get(cacheKey);
        if (product != null) {
            product.setName(request.name());
            product.setPrice(request.price());
            redisTemplate.opsForValue().set(cacheKey, product);
        }
        
        // Queue DB write
        pendingWrites.offer(new CacheWriteTask(id, request));
    }
    
    @Scheduled(fixedDelay = 5000)
    @Transactional
    public void flushPendingWrites() {
        int count = 0;
        CacheWriteTask task;
        
        while ((task = pendingWrites.poll()) != null && count < 100) {
            try {
                Product product = productRepository.findById(task.productId()).orElseThrow();
                product.setName(task.request().name());
                product.setPrice(task.request().price());
                productRepository.save(product);
                count++;
            } catch (Exception e) {
                log.error("Failed to write product {} to DB", task.productId(), e);
                // Re-queue or send to DLQ
            }
        }
        
        if (count > 0) {
            log.debug("Flushed {} writes to DB", count);
        }
    }
}
```

---

## ขั้นตอนที่ 1525: Cache Key Strategies

```java
@Service
public class ProductCacheKeyStrategy {
    
    // Simple key
    // @Cacheable(value = "products", key = "#id")
    
    // Composite key
    // @Cacheable(value = "products", key = "#keyword + ':' + #page + ':' + #size")
    
    // SpEL expression
    // @Cacheable(value = "products", key = "#request.categoryId + ':' + #request.status")
    
    // Custom key generator
    @Bean
    public KeyGenerator customKeyGenerator() {
        return (target, method, params) -> {
            StringBuilder key = new StringBuilder();
            key.append(target.getClass().getSimpleName()).append(":");
            key.append(method.getName()).append(":");
            for (Object p : params) {
                if (p != null) key.append(p.toString()).append(":");
            }
            return key.toString();
        };
    }
    
    // Version-based cache key (for cache busting)
    @Cacheable(value = "products", key = "#root.targetClass.simpleName + ':v' + @cacheVersion.get() + ':' + #id")
    public Product findById(Long id) { ... }
}

// Cache version management
@Component
public class CacheVersion {
    
    private final AtomicLong version = new AtomicLong(1);
    
    public long get() { return version.get(); }
    
    public void increment() { version.incrementAndGet(); }
    
    // Call this to invalidate ALL caches
    public void invalidateAll() { version.incrementAndGet(); }
}
```

---

## ขั้นตอนที่ 1526-1560: Cache Monitoring

```java
@Component
@RequiredArgsConstructor
public class CacheMetricsCollector {
    
    private final CacheManager caffeineCacheManager;
    private final MeterRegistry registry;
    
    @Scheduled(fixedDelay = 60000)
    public void collectMetrics() {
        caffeineCacheManager.getCacheNames().forEach(cacheName -> {
            Cache cache = caffeineCacheManager.getCache(cacheName);
            if (cache instanceof CaffeineCache caffeineCache) {
                com.github.benmanes.caffeine.cache.Cache<?, ?> nativeCache = 
                    caffeineCache.getNativeCache();
                
                CacheStats stats = nativeCache.stats();
                
                Gauge.builder("cache.hit.rate", stats, CacheStats::hitRate)
                    .tag("cache", cacheName)
                    .register(registry);
                
                Gauge.builder("cache.miss.rate", stats, CacheStats::missRate)
                    .tag("cache", cacheName)
                    .register(registry);
                
                Gauge.builder("cache.size", nativeCache, c -> c.estimatedSize())
                    .tag("cache", cacheName)
                    .register(registry);
                
                Gauge.builder("cache.eviction.count", stats, CacheStats::evictionCount)
                    .tag("cache", cacheName)
                    .register(registry);
                
                log.debug("Cache {}: hitRate={:.2f}%, size={}", cacheName,
                    stats.hitRate() * 100, nativeCache.estimatedSize());
            }
        });
    }
}

// Grafana Dashboard Query:
// cache_hit_rate{cache="products"}  → should be > 90%
// cache_miss_rate{cache="products"} → should be < 10%
// cache_size{cache="products"}      → should be < maxSize
```

---

*[← Part 47: Spring Batch](./part-47-batch.md) | [Part 49: Clean Architecture →](./part-49-clean-architecture.md)*

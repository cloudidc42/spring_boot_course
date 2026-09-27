# Part 90: Production Optimization for Spring Boot
## ขั้นตอนที่ 3201-3240

**ระดับ:** World-Class (ระดับโลก)
**เวลาเรียน:** 8-10 ชั่วโมง
**เป้าหมาย:** เรียนรู้การ optimize Spring Boot application สำหรับ production จริง ครอบคลุม JVM tuning, Connection pool optimization, Thread pool tuning, Cache warming, HTTP/2, Lazy initialization และ AOT compilation

---

## ขั้นตอนที่ 3201: JVM Tuning - G1GC vs ZGC vs Shenandoah

การเลือก Garbage Collector (GC) ที่เหมาะสมมีผลอย่างมากต่อ latency และ throughput ของ application

### เปรียบเทียบ Garbage Collectors

| GC | Pause Time | Throughput | Use Case | Java Version |
|---|---|---|---|---|
| G1GC | ~50-200ms | High | General purpose | 9+ (default) |
| ZGC | <10ms | Medium | Low latency | 15+ (production) |
| Shenandoah | <10ms | Medium | Low latency | 12+ (Red Hat) |
| ParallelGC | High | Very High | Batch jobs | All |

### G1GC Configuration (แนะนำสำหรับ most workloads)

```bash
# G1GC - สมดุลระหว่าง latency และ throughput
java -server \
  -XX:+UseG1GC \
  -XX:MaxGCPauseMillis=200 \
  -XX:G1HeapRegionSize=16m \
  -XX:G1NewSizePercent=20 \
  -XX:G1MaxNewSizePercent=40 \
  -XX:G1MixedGCCountTarget=8 \
  -XX:InitiatingHeapOccupancyPercent=45 \
  -Xms2g -Xmx4g \
  -jar app.jar
```

### ZGC Configuration (สำหรับ low-latency requirements)

```bash
# ZGC - pause time <10ms เหมาะกับ real-time applications
java -server \
  -XX:+UseZGC \
  -XX:+ZGenerational \
  -XX:MaxGCPauseMillis=10 \
  -XX:SoftMaxHeapSize=6g \
  -Xms4g -Xmx8g \
  -XX:ZUncommitDelay=300 \
  -jar app.jar
```

### Shenandoah Configuration

```bash
# Shenandoah - low pause, good for containers
java -server \
  -XX:+UseShenandoahGC \
  -XX:ShenandoahGCMode=adaptive \
  -XX:ShenandoahGCHeuristics=adaptive \
  -Xms2g -Xmx4g \
  -jar app.jar
```

### GC Monitoring

```java
// config/GCMonitoringConfig.java
@Configuration
@ConditionalOnProperty("app.gc-monitoring.enabled")
public class GCMonitoringConfig {
    
    @Bean
    public GCNotificationListener gcNotificationListener(MeterRegistry meterRegistry) {
        GCNotificationListener listener = new GCNotificationListener(meterRegistry);
        
        // Register listener สำหรับทุก GC bean
        for (GarbageCollectorMXBean gcBean : ManagementFactory.getGarbageCollectorMXBeans()) {
            if (gcBean instanceof NotificationEmitter notificationEmitter) {
                notificationEmitter.addNotificationListener(listener, null, null);
            }
        }
        
        return listener;
    }
}

// monitoring/GCNotificationListener.java
public class GCNotificationListener implements NotificationListener {
    
    private static final Logger log = LoggerFactory.getLogger(GCNotificationListener.class);
    private final MeterRegistry meterRegistry;
    
    @Override
    public void handleNotification(Notification notification, Object handback) {
        GarbageCollectionNotificationInfo info = GarbageCollectionNotificationInfo
            .from((CompositeData) notification.getUserData());
        
        GcInfo gcInfo = info.getGcInfo();
        long duration = gcInfo.getDuration();
        
        // บันทึก metrics
        meterRegistry.timer("jvm.gc.pause",
            "cause", info.getGcCause(),
            "action", info.getGcAction())
            .record(duration, TimeUnit.MILLISECONDS);
        
        // Alert ถ้า pause นานเกินไป
        if (duration > 500) {
            log.warn("GC pause ยาวนาน: {}ms, cause={}", duration, info.getGcCause());
        }
    }
}
```

---

## ขั้นตอนที่ 3202: Heap Sizing สำหรับ Containers

### Container-Aware Memory Configuration

```bash
# ใช้ MaxRAMPercentage แทน -Xmx เพื่อให้ adapt ตาม container limit
java -XX:InitialRAMPercentage=50.0 \
     -XX:MaxRAMPercentage=75.0 \
     -XX:MinRAMPercentage=25.0 \
     -jar app.jar

# สูตรการคำนวณ:
# Container Memory: 2GB
# MaxRAMPercentage=75 → JVM heap max = 1.5GB
# เหลือ 25% (512MB) สำหรับ OS, off-heap, etc.
```

### Dockerfile ที่ถูกต้อง

```dockerfile
# Dockerfile
FROM eclipse-temurin:21-jre-alpine

# สร้าง non-root user
RUN addgroup -S spring && adduser -S spring -G spring

WORKDIR /app

# Copy jar
COPY target/shophub-*.jar app.jar

# ให้สิทธิ์ไฟล์
RUN chown spring:spring app.jar

USER spring

# JVM flags สำหรับ container
ENV JAVA_OPTS="-XX:+UseZGC \
               -XX:+ZGenerational \
               -XX:InitialRAMPercentage=50.0 \
               -XX:MaxRAMPercentage=75.0 \
               -XX:+ExitOnOutOfMemoryError \
               -XX:+HeapDumpOnOutOfMemoryError \
               -XX:HeapDumpPath=/tmp/heapdump.hprof \
               -Djava.security.egd=file:/dev/./urandom"

ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar app.jar"]
```

### Kubernetes Resource Configuration

```yaml
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: shophub-api
spec:
  template:
    spec:
      containers:
      - name: shophub-api
        image: shophub/api:latest
        resources:
          requests:
            memory: "512Mi"
            cpu: "250m"
          limits:
            memory: "1Gi"    # Container limit = 1GB
            cpu: "1000m"
        env:
        - name: JAVA_OPTS
          value: >-
            -XX:+UseZGC
            -XX:MaxRAMPercentage=75.0
            -XX:InitialRAMPercentage=50.0
            -XX:+ExitOnOutOfMemoryError
        livenessProbe:
          httpGet:
            path: /actuator/health/liveness
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /actuator/health/readiness
            port: 8080
          initialDelaySeconds: 20
          periodSeconds: 5
```

---

## ขั้นตอนที่ 3203: HikariCP Connection Pool Optimization

HikariCP เป็น default connection pool ใน Spring Boot ซึ่งต้องการการ tune อย่างระมัดระวัง

### สูตรการคำนวณ Pool Size

```
สูตรของ HikariCP:
pool_size = Tn × (Cm - 1) + 1

โดยที่:
- Tn = จำนวน threads ที่ใช้ DB พร้อมกัน
- Cm = จำนวน queries ต่อ transaction

ตัวอย่าง:
- Tomcat threads = 200
- Queries per transaction = 2
- pool_size = 200 × (2-1) + 1 = 201

แต่ในทางปฏิบัติ ใช้: pool_size = (core_count × 2) + effective_spindle_count
สำหรับ 4-core CPU: pool_size = (4 × 2) + 1 = 9
```

### HikariCP Configuration

```yaml
# application.yml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/shophub
    username: ${DB_USER}
    password: ${DB_PASSWORD}
    driver-class-name: org.postgresql.Driver
    hikari:
      # Pool sizing
      minimum-idle: 5           # Connections ขั้นต่ำ
      maximum-pool-size: 20     # Connections สูงสุด
      
      # Timeout settings
      connection-timeout: 30000       # รอ connection สูงสุด 30s
      idle-timeout: 600000            # Connection ไม่ใช้ 10 นาที → close
      max-lifetime: 1800000           # Connection อายุสูงสุด 30 นาที
      keepalive-time: 300000          # Ping ทุก 5 นาที
      
      # Performance
      connection-test-query: SELECT 1
      pool-name: ShophubPool
      
      # ตรวจสอบ connection quality
      leak-detection-threshold: 60000  # แจ้งเตือนถ้า connection ไม่คืน 60s
      
      # Data source properties
      data-source-properties:
        cachePrepStmts: true
        prepStmtCacheSize: 250
        prepStmtCacheSqlLimit: 2048
        useServerPrepStmts: true
        useLocalSessionState: true
        rewriteBatchedStatements: true
        cacheResultSetMetadata: true
        cacheServerConfiguration: true
        elideSetAutoCommits: true
        maintainTimeStats: false
```

### Dynamic Pool Sizing Based on Load

```java
// config/DynamicPoolConfig.java
@Configuration
@EnableScheduling
public class DynamicPoolConfig {
    
    private final HikariDataSource dataSource;
    private final MeterRegistry meterRegistry;
    
    @Scheduled(fixedRate = 60000) // ตรวจสอบทุก 1 นาที
    public void adjustPoolSize() {
        int activeConnections = dataSource.getHikariPoolMXBean().getActiveConnections();
        int maxPoolSize = dataSource.getMaximumPoolSize();
        
        double utilization = (double) activeConnections / maxPoolSize;
        
        if (utilization > 0.8 && maxPoolSize < 50) {
            // เพิ่ม pool size เมื่อใช้งานสูง
            int newSize = Math.min(maxPoolSize + 5, 50);
            dataSource.setMaximumPoolSize(newSize);
            log.info("เพิ่ม pool size เป็น {}", newSize);
        } else if (utilization < 0.3 && maxPoolSize > 10) {
            // ลด pool size เมื่อใช้งานต่ำ
            int newSize = Math.max(maxPoolSize - 5, 10);
            dataSource.setMaximumPoolSize(newSize);
            log.info("ลด pool size เป็น {}", newSize);
        }
        
        // Record metrics
        meterRegistry.gauge("db.pool.utilization", utilization);
    }
}
```

### Multiple DataSources

```java
// config/MultiDataSourceConfig.java
@Configuration
public class MultiDataSourceConfig {
    
    // Primary datasource สำหรับ write operations
    @Bean
    @Primary
    @ConfigurationProperties("spring.datasource.primary")
    public DataSource primaryDataSource() {
        return DataSourceBuilder.create()
            .type(HikariDataSource.class)
            .build();
    }
    
    // Read replica สำหรับ read operations
    @Bean
    @ConfigurationProperties("spring.datasource.replica")
    public DataSource replicaDataSource() {
        return DataSourceBuilder.create()
            .type(HikariDataSource.class)
            .build();
    }
    
    // Routing datasource ที่เลือก primary/replica อัตโนมัติ
    @Bean
    public DataSource routingDataSource(
            @Qualifier("primaryDataSource") DataSource primary,
            @Qualifier("replicaDataSource") DataSource replica) {
        
        RoutingDataSource routing = new RoutingDataSource();
        routing.setTargetDataSources(Map.of(
            DataSourceType.PRIMARY, primary,
            DataSourceType.REPLICA, replica
        ));
        routing.setDefaultTargetDataSource(primary);
        return routing;
    }
}

// Read-only transactions จะ route ไป replica อัตโนมัติ
@Service
@Transactional(readOnly = true) // → replica
public class ProductQueryService {
    
    public List<Product> findAll(ProductFilter filter) {
        return productRepository.findWithFilter(filter);
    }
}

@Service
@Transactional // → primary (default)
public class ProductCommandService {
    
    public Product save(Product product) {
        return productRepository.save(product);
    }
}
```

---

## ขั้นตอนที่ 3204: Tomcat Thread Pool Tuning

```yaml
# application.yml
server:
  tomcat:
    threads:
      min-spare: 10           # Threads ขั้นต่ำที่ keep alive
      max: 200                # Threads สูงสุด
    max-connections: 8192     # Connections สูงสุด
    accept-count: 100         # Queue ก่อน reject
    connection-timeout: 20000 # Connection timeout
    keep-alive-timeout: 60000 # Keep-alive timeout
    
  # Compression
  compression:
    enabled: true
    mime-types: application/json,application/xml,text/html,text/xml,text/plain
    min-response-size: 1024   # Compress ถ้า response > 1KB
```

### Custom Thread Pool สำหรับ Async Tasks

```java
// config/ThreadPoolConfig.java
@Configuration
@EnableAsync
public class ThreadPoolConfig {
    
    // Thread pool สำหรับ general async tasks
    @Bean(name = "taskExecutor")
    public TaskExecutor taskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(Runtime.getRuntime().availableProcessors() * 2);
        executor.setMaxPoolSize(Runtime.getRuntime().availableProcessors() * 4);
        executor.setQueueCapacity(500);
        executor.setThreadNamePrefix("async-");
        executor.setKeepAliveSeconds(60);
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        executor.setWaitForTasksToCompleteOnShutdown(true);
        executor.setAwaitTerminationSeconds(30);
        executor.initialize();
        return executor;
    }
    
    // Thread pool เฉพาะสำหรับ I/O operations
    @Bean(name = "ioExecutor")
    public TaskExecutor ioExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(50);
        executor.setMaxPoolSize(100);
        executor.setQueueCapacity(200);
        executor.setThreadNamePrefix("io-");
        executor.initialize();
        return executor;
    }
    
    // Thread pool สำหรับ notification tasks
    @Bean(name = "notificationExecutor")
    public TaskExecutor notificationExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);
        executor.setMaxPoolSize(20);
        executor.setQueueCapacity(1000);
        executor.setThreadNamePrefix("notify-");
        executor.initialize();
        return executor;
    }
}
```

### Virtual Threads (Java 21+)

```java
// config/VirtualThreadConfig.java
@Configuration
@ConditionalOnJava(JavaVersion.TWENTY_ONE)
public class VirtualThreadConfig {
    
    @Bean
    public TomcatProtocolHandlerCustomizer<?> protocolHandlerVirtualThreadExecutorCustomizer() {
        return protocolHandler -> {
            // ใช้ Virtual Threads สำหรับ Tomcat
            protocolHandler.setExecutor(Executors.newVirtualThreadPerTaskExecutor());
        };
    }
    
    @Bean(name = "virtualThreadExecutor")
    public AsyncTaskExecutor applicationTaskExecutor() {
        return new TaskExecutorAdapter(Executors.newVirtualThreadPerTaskExecutor());
    }
}
```

---

## ขั้นตอนที่ 3205: Cache Warming on Startup

การ warm up cache ก่อน traffic จริง ช่วยป้องกัน cold start problem

```java
// startup/CacheWarmupService.java
@Service
@Slf4j
public class CacheWarmupService implements ApplicationListener<ApplicationReadyEvent> {
    
    private final ProductService productService;
    private final CategoryService categoryService;
    private final PromotionService promotionService;
    private final CacheManager cacheManager;
    
    @Override
    public void onApplicationEvent(ApplicationReadyEvent event) {
        log.info("เริ่ม cache warming...");
        long startTime = System.currentTimeMillis();
        
        try {
            warmProductCache();
            warmCategoryCache();
            warmPromotionCache();
            
            long duration = System.currentTimeMillis() - startTime;
            log.info("Cache warming เสร็จสิ้นใน {}ms", duration);
        } catch (Exception e) {
            log.warn("Cache warming ล้มเหลว แต่ application ยังทำงานได้", e);
        }
    }
    
    private void warmProductCache() {
        log.info("กำลัง warm product cache...");
        
        // Load top 1000 products ที่ถูก view บ่อยที่สุด
        List<String> popularProductIds = getPopularProductIds();
        
        popularProductIds.parallelStream()
            .forEach(productId -> {
                try {
                    productService.findById(productId); // triggers cache put
                } catch (Exception e) {
                    log.debug("ไม่สามารถ warm product {}", productId);
                }
            });
        
        log.info("Warm {} products เสร็จสิ้น", popularProductIds.size());
    }
    
    private void warmCategoryCache() {
        // Load all categories (usually small set)
        categoryService.findAll().forEach(category -> {
            // triggers cache
        });
    }
    
    private void warmPromotionCache() {
        // Load active promotions
        promotionService.findActive().forEach(promotion -> {
            // triggers cache
        });
    }
    
    @Cacheable("popularProductIds")
    private List<String> getPopularProductIds() {
        return analyticsRepository.findTopProductIds(1000);
    }
}
```

### Layered Cache Configuration

```java
// config/CacheConfig.java
@Configuration
@EnableCaching
public class CacheConfig {
    
    @Bean
    public CacheManager cacheManager(RedisConnectionFactory redisFactory) {
        // L1: Local in-memory cache (Caffeine)
        CaffeineCacheManager caffeineCacheManager = new CaffeineCacheManager();
        caffeineCacheManager.setCaffeine(Caffeine.newBuilder()
            .maximumSize(10_000)
            .expireAfterWrite(Duration.ofMinutes(5))
            .recordStats()
        );
        
        // L2: Distributed cache (Redis)
        RedisCacheManager redisCacheManager = RedisCacheManager.builder(redisFactory)
            .cacheDefaults(RedisCacheConfiguration.defaultCacheConfig()
                .entryTtl(Duration.ofHours(1))
                .serializeValuesWith(RedisSerializationContext.SerializationPair
                    .fromSerializer(new GenericJackson2JsonRedisSerializer())))
            .withCacheConfiguration("products", 
                RedisCacheConfiguration.defaultCacheConfig()
                    .entryTtl(Duration.ofMinutes(30)))
            .withCacheConfiguration("userSessions",
                RedisCacheConfiguration.defaultCacheConfig()
                    .entryTtl(Duration.ofHours(24)))
            .build();
        
        // Composite cache: check L1 first, then L2
        return new CompositeCacheManager(caffeineCacheManager, redisCacheManager);
    }
}
```

### Cache Metrics

```java
// monitoring/CacheMetricsConfig.java
@Configuration
public class CacheMetricsConfig {
    
    @Bean
    public CacheMetricsRegistrar cacheMetricsRegistrar(
            Collection<CacheManager> cacheManagers,
            MeterRegistry meterRegistry) {
        
        cacheManagers.forEach(cm -> {
            cm.getCacheNames().forEach(cacheName -> {
                Cache cache = cm.getCache(cacheName);
                if (cache instanceof CaffeineCache caffeineCache) {
                    // Register Caffeine cache metrics
                    CaffeineCacheMetrics.monitor(
                        meterRegistry,
                        caffeineCache.getNativeCache(),
                        cacheName
                    );
                }
            });
        });
        
        return new CacheMetricsRegistrar(cacheManagers, meterRegistry);
    }
}
```

---

## ขั้นตอนที่ 3206: HTTP/2 Configuration

HTTP/2 ช่วยลด latency ด้วย multiplexing, header compression และ server push

```yaml
# application.yml
server:
  port: 8443
  http2:
    enabled: true
  ssl:
    key-store: classpath:keystore.p12
    key-store-password: ${SSL_KEYSTORE_PASSWORD}
    key-store-type: PKCS12
    key-alias: shophub
    enabled: true
    protocol: TLS
    enabled-protocols: TLSv1.2,TLSv1.3
```

### HTTP/2 Push (Server Push)

```java
// controller/ProductController.java
@RestController
@RequestMapping("/api/v1")
public class ProductController {
    
    @GetMapping("/products/{id}")
    public ResponseEntity<Product> getProduct(
            @PathVariable String id,
            HttpServletRequest request) {
        
        Product product = productService.findById(id);
        
        // HTTP/2 Server Push - ส่ง related resources ล่วงหน้า
        if (request.getServletContext().getMajorVersion() >= 3) {
            PushBuilder pushBuilder = request.newPushBuilder();
            if (pushBuilder != null) {
                // Push product images
                pushBuilder.path("/api/v1/products/" + id + "/images")
                    .push();
                
                // Push related products
                pushBuilder.path("/api/v1/products/" + id + "/related")
                    .push();
            }
        }
        
        return ResponseEntity.ok()
            .cacheControl(CacheControl.maxAge(Duration.ofMinutes(5)))
            .body(product);
    }
}
```

### HTTP/2 Connector for Development (HTTP/2 without SSL)

```java
// config/Http2Config.java
@Configuration
@Profile("dev")
public class Http2Config {
    
    @Bean
    public TomcatServletWebServerFactory tomcatServletWebServerFactory() {
        TomcatServletWebServerFactory factory = new TomcatServletWebServerFactory();
        
        factory.addConnectorCustomizers(connector -> {
            connector.setScheme("http");
            Http11NioProtocol protocol = (Http11NioProtocol) connector.getProtocolHandler();
            protocol.setMaxHttpHeaderSize(65536);
        });
        
        // Enable h2c (HTTP/2 cleartext) for development
        factory.addConnectorCustomizers(connector -> {
            connector.addUpgradeProtocol(new Http2Protocol());
        });
        
        return factory;
    }
}
```

---

## ขั้นตอนที่ 3207: Lazy Initialization สำหรับ Faster Startup

```yaml
# application.yml
spring:
  main:
    lazy-initialization: true  # Enable global lazy init
    
# ปัญหา: lazy init อาจซ่อน configuration errors
# แก้ไข: เพิ่ม eager beans สำหรับ critical components
```

### Selective Eager Loading

```java
// config/EagerInitializationConfig.java
@Configuration
public class EagerInitializationConfig {
    
    // Force eager initialization ของ critical beans
    @Bean
    public static BeanFactoryPostProcessor eagerBeanInitializer() {
        return beanFactory -> {
            // Beans เหล่านี้จะถูก initialize ทันที ไม่ว่าจะ enable lazy init
            String[] criticalBeans = {
                "dataSource",
                "entityManagerFactory",
                "transactionManager",
                "securityFilterChain",
                "cacheManager"
            };
            
            for (String beanName : criticalBeans) {
                if (beanFactory instanceof DefaultListableBeanFactory dlbf) {
                    BeanDefinition bd = dlbf.getBeanDefinition(beanName);
                    bd.setLazyInit(false);
                }
            }
        };
    }
}

// ใช้ @Lazy explicitly สำหรับ heavy beans
@Service
@Lazy // จะถูกสร้างเมื่อมีการใช้งานครั้งแรก
public class ReportGenerationService {
    
    // Heavy initialization - PDF libraries, etc.
    private final PdfRenderer pdfRenderer;
    private final ExcelRenderer excelRenderer;
    
    public ReportGenerationService() {
        this.pdfRenderer = new PdfRenderer(); // slow initialization
        this.excelRenderer = new ExcelRenderer();
    }
}

// @Lazy injection
@Service
public class OrderService {
    
    @Lazy // Inject lazily - ReportGenerationService ไม่ถูกสร้างจนกว่าจะเรียกใช้
    private final ReportGenerationService reportService;
    
    // ...
}
```

### Spring Boot 3.x Startup Actuator

```yaml
# เปิด startup endpoint เพื่อวิเคราะห์ startup time
management:
  endpoints:
    web:
      exposure:
        include: startup,health,metrics,info
  endpoint:
    startup:
      enabled: true
```

```java
// monitoring/StartupMetricsListener.java
@Component
public class StartupMetricsListener implements ApplicationListener<ApplicationStartedEvent> {
    
    private final MeterRegistry meterRegistry;
    
    @Override
    public void onApplicationEvent(ApplicationStartedEvent event) {
        Duration startupTime = event.getTimeTaken();
        
        meterRegistry.gauge("app.startup.time.seconds", 
            startupTime.toSeconds());
        
        log.info("Application เริ่มต้นใน {} วินาที", startupTime.toSeconds());
    }
}
```

---

## ขั้นตอนที่ 3208: AOT Compilation Hints สำหรับ GraalVM Native Image

AOT (Ahead-of-Time) compilation ใน Spring Boot 3.x ช่วยสร้าง native binary ที่ startup เร็วมาก

### เพิ่ม AOT Hints

```java
// hints/ShophubRuntimeHints.java
@Component
@ImportRuntimeHints(ShophubRuntimeHints.class)
public class ShophubRuntimeHints implements RuntimeHintsRegistrar {
    
    @Override
    public void registerHints(RuntimeHints hints, ClassLoader classLoader) {
        // Register reflection hints สำหรับ domain classes
        hints.reflection().registerType(
            Order.class,
            MemberMode.INVOKE_PUBLIC_CONSTRUCTORS,
            MemberMode.INVOKE_PUBLIC_METHODS
        );
        
        hints.reflection().registerType(
            Product.class,
            MemberMode.INVOKE_PUBLIC_CONSTRUCTORS,
            MemberMode.INVOKE_PUBLIC_METHODS
        );
        
        // Register resource hints สำหรับ static resources
        hints.resources().registerPattern("templates/*.html");
        hints.resources().registerPattern("i18n/*.properties");
        hints.resources().registerPattern("data/*.json");
        
        // Register serialization hints
        hints.serialization().registerType(OrderEvent.class);
        hints.serialization().registerType(ProductDto.class);
        
        // Register proxy hints สำหรับ Spring proxies
        hints.proxies().registerJdkProxy(ProductService.class);
    }
}
```

### Build Native Image

```xml
<!-- pom.xml -->
<build>
    <plugins>
        <plugin>
            <groupId>org.graalvm.buildtools</groupId>
            <artifactId>native-maven-plugin</artifactId>
            <configuration>
                <imageName>shophub-api</imageName>
                <buildArgs>
                    <buildArg>--no-fallback</buildArg>
                    <buildArg>-H:+ReportExceptionStackTraces</buildArg>
                    <buildArg>--initialize-at-build-time=org.slf4j</buildArg>
                </buildArgs>
            </configuration>
        </plugin>
    </plugins>
</build>
```

```bash
# Build native image
./mvnw -Pnative native:compile

# Run native binary (startup ~100ms แทนที่จะเป็น 5-10s)
./target/shophub-api

# ตรวจสอบ startup time
time ./target/shophub-api --server.port=8080
```

### Conditional AOT Optimization

```java
// config/NativeImageConfig.java
@Configuration
@ConditionalOnNativeImage // เฉพาะตอน run เป็น native
public class NativeImageConfig {
    
    @Bean
    public ObjectMapper objectMapper() {
        // Configuration พิเศษสำหรับ native image
        return JsonMapper.builder()
            .addModule(new JavaTimeModule())
            .configure(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS, false)
            .configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false)
            // Avoid reflection-based type detection ใน native
            .configure(MapperFeature.USE_ANNOTATIONS, true)
            .build();
    }
}
```

---

## ขั้นตอนที่ 3209: Response Compression และ Optimization

```java
// config/WebMvcConfig.java
@Configuration
public class WebMvcConfig implements WebMvcConfigurer {
    
    @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry) {
        registry.addResourceHandler("/static/**")
            .addResourceLocations("classpath:/static/")
            .setCacheControl(CacheControl.maxAge(Duration.ofDays(365)))
            .resourceChain(true)
            .addResolver(new GzipResourceResolver()) // Serve pre-compressed files
            .addResolver(new VersionResourceResolver()
                .addContentVersionStrategy("/**")); // Content-based versioning
    }
    
    @Override
    public void configureContentNegotiation(ContentNegotiationConfigurer configurer) {
        configurer
            .favorParameter(false)
            .favorPathExtension(false)
            .ignoreAcceptHeader(false)
            .defaultContentType(MediaType.APPLICATION_JSON);
    }
}
```

### Response Caching Headers

```java
// controller/ProductController.java
@GetMapping("/products/{id}")
public ResponseEntity<ProductDto> getProduct(@PathVariable String id) {
    Product product = productService.findById(id);
    
    return ResponseEntity.ok()
        .cacheControl(CacheControl.maxAge(Duration.ofMinutes(5))
            .mustRevalidate()
            .cachePublic())
        .eTag(String.valueOf(product.getVersion())) // ETag สำหรับ conditional requests
        .lastModified(product.getUpdatedAt().toInstant(ZoneOffset.UTC))
        .body(productMapper.toDto(product));
}

@GetMapping("/products")
public ResponseEntity<List<ProductDto>> listProducts(ProductFilter filter) {
    List<ProductDto> products = productService.findAll(filter);
    
    return ResponseEntity.ok()
        .cacheControl(CacheControl.maxAge(Duration.ofMinutes(2)))
        .body(products);
}
```

---

## ขั้นตอนที่ 3210: Performance Profiling

```java
// config/ProfilingConfig.java
@Configuration
@Profile("profiling")
public class ProfilingConfig {
    
    // Async profiler integration
    @Bean
    public AsyncProfilerMetrics asyncProfilerMetrics(MeterRegistry meterRegistry) {
        return new AsyncProfilerMetrics(meterRegistry);
    }
}

// aspect/PerformanceMonitoringAspect.java
@Aspect
@Component
@ConditionalOnProperty("app.performance-monitoring.enabled")
public class PerformanceMonitoringAspect {
    
    private final MeterRegistry meterRegistry;
    
    @Around("@annotation(Monitored)")
    public Object measurePerformance(ProceedingJoinPoint joinPoint) throws Throwable {
        String methodName = joinPoint.getSignature().toShortString();
        
        Timer.Sample sample = Timer.start(meterRegistry);
        
        try {
            Object result = joinPoint.proceed();
            
            sample.stop(Timer.builder("method.execution")
                .tag("method", methodName)
                .tag("status", "success")
                .register(meterRegistry));
            
            return result;
        } catch (Exception e) {
            sample.stop(Timer.builder("method.execution")
                .tag("method", methodName)
                .tag("status", "error")
                .tag("exception", e.getClass().getSimpleName())
                .register(meterRegistry));
            throw e;
        }
    }
}

// annotation/Monitored.java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Monitored {}
```

---

## ขั้นตอนที่ 3211-3240: Production Checklist และ Best Practices

### Performance Optimization Checklist

```
JVM:
  ✅ เลือก GC ที่เหมาะสม (G1GC/ZGC)
  ✅ ใช้ MaxRAMPercentage แทน -Xmx ใน containers
  ✅ เปิด GC logging สำหรับ monitoring
  ✅ ตั้งค่า OOMKiller (-XX:+ExitOnOutOfMemoryError)

Database:
  ✅ HikariCP pool size ตามสูตร
  ✅ Prepared statement cache
  ✅ Read replica สำหรับ query-heavy workloads
  ✅ Connection leak detection

Caching:
  ✅ Layered cache (L1: Caffeine, L2: Redis)
  ✅ Cache warming on startup
  ✅ Appropriate TTL per cache
  ✅ Cache metrics monitoring

HTTP:
  ✅ HTTP/2 enabled
  ✅ Response compression (gzip/brotli)
  ✅ Cache-Control headers
  ✅ ETag support

Startup:
  ✅ Lazy initialization
  ✅ AOT hints สำหรับ reflection
  ✅ Startup time monitoring
```

### Load Testing ก่อน Production

```bash
# ใช้ k6 สำหรับ load testing
# k6/load-test.js
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
    stages: [
        { duration: '2m', target: 100 },   // Ramp up
        { duration: '5m', target: 100 },   // Sustained
        { duration: '2m', target: 200 },   // Peak
        { duration: '5m', target: 200 },   // Sustained peak
        { duration: '2m', target: 0 },     // Ramp down
    ],
    thresholds: {
        http_req_duration: ['p(95)<500'],   // 95% requests < 500ms
        http_req_failed: ['rate<0.01'],     // Error rate < 1%
    },
};

export default function() {
    const response = http.get('http://localhost:8080/api/v1/products');
    check(response, {
        'status 200': (r) => r.status === 200,
        'response time < 500ms': (r) => r.timings.duration < 500,
    });
    sleep(1);
}
```

---

## สรุป

Part 90 ครอบคลุม Production Optimization ที่สำคัญ:

1. **JVM Tuning** - G1GC สำหรับ balanced, ZGC สำหรับ ultra-low latency
2. **Heap Sizing** - MaxRAMPercentage สำหรับ containers
3. **HikariCP** - pool sizing ตามสูตรและ workload
4. **Thread Pools** - multiple pools ตาม task type + Virtual Threads
5. **Cache Warming** - ป้องกัน cold start
6. **HTTP/2** - multiplexing และ server push
7. **Lazy Init** - ลด startup time
8. **AOT Hints** - เตรียมสำหรับ native image

การ optimize ที่ดีต้องมาพร้อมกับ **monitoring** และ **load testing** เสมอ อย่า guess - ใช้ data จาก profiling

---

*[← Part 89: Advanced Patterns](./part-89-advanced-patterns.md) | [Part 91: Microservices Project →](./part-91-microservices-project.md)*

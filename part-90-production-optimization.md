# Part 90: Production Optimization
## ขั้นตอนที่ 3201-3240

**ระดับ:** ระดับโลก (World-Class)
**เวลาเรียน:** 7-9 ชั่วโมง
**เป้าหมาย:** เรียนรู้การ optimize Spring Boot applications สำหรับ production ครอบคลุม JVM tuning, GC selection, connection pools, thread pools, caching, และ HTTP/2

---

## ขั้นตอนที่ 3201: JVM Tuning พื้นฐาน

การ configure JVM อย่างถูกต้องส่งผลอย่างมากต่อ performance ของ Spring Boot applications

### JVM Flags พื้นฐาน

```bash
# run-production.sh

# เปิดใช้งาน application พร้อม JVM options ที่ optimize แล้ว
java \
  # ---- Memory Settings ----
  -Xms2g \                          # Initial heap size
  -Xmx4g \                          # Maximum heap size
  -XX:MetaspaceSize=256m \           # Initial metaspace
  -XX:MaxMetaspaceSize=512m \        # Max metaspace
  -XX:+UseCompressedOops \           # Compress object pointers (< 32GB heap)
  -XX:+UseCompressedClassPointers \  # Compress class pointers
  \
  # ---- GC Settings (G1GC) ----
  -XX:+UseG1GC \
  -XX:MaxGCPauseMillis=200 \         # Target max pause 200ms
  -XX:G1HeapRegionSize=16m \         # G1 region size
  -XX:G1NewSizePercent=30 \          # Min young generation %
  -XX:G1MaxNewSizePercent=40 \       # Max young generation %
  -XX:G1MixedGCCountTarget=8 \       # Mixed GC cycles
  -XX:InitiatingHeapOccupancyPercent=45 \  # Trigger concurrent GC at 45%
  \
  # ---- JIT Compiler ----
  -XX:+TieredCompilation \           # เปิด tiered compilation
  -XX:ReservedCodeCacheSize=256m \   # Code cache สำหรับ JIT
  -XX:+UseCodeCacheFlushing \        # Flush code cache เมื่อเต็ม
  \
  # ---- GC Logging ----
  -Xlog:gc*:file=/var/log/app/gc.log:time,uptime:filecount=5,filesize=50m \
  \
  # ---- JVM Diagnostics ----
  -XX:+HeapDumpOnOutOfMemoryError \  # Heap dump เมื่อ OOM
  -XX:HeapDumpPath=/var/log/app/ \
  -XX:+ExitOnOutOfMemoryError \      # ออกจากโปรแกรมเมื่อ OOM
  \
  # ---- Performance ----
  -server \                          # Server VM mode
  -XX:+OptimizeStringConcat \        # Optimize string operations
  -XX:+UseStringDeduplication \      # Deduplicate strings (G1GC only)
  \
  # ---- Spring Boot specific ----
  -Dspring.profiles.active=production \
  -Dserver.port=8080 \
  \
  -jar app.jar
```

---

## ขั้นตอนที่ 3202: G1GC vs ZGC vs Shenandoah

การเลือก Garbage Collector ที่เหมาะสมกับ workload

```bash
# G1GC - เหมาะสำหรับ: Balanced throughput/latency, heap < 32GB
# ลักษณะ: Generational, region-based, concurrent marking
JAVA_OPTS_G1="
  -XX:+UseG1GC
  -XX:MaxGCPauseMillis=200
  -XX:G1HeapRegionSize=16m
  -XX:ParallelGCThreads=8
  -XX:ConcGCThreads=4
"

# ZGC - เหมาะสำหรับ: Ultra-low latency, large heaps (TB scale)
# ลักษณะ: Non-generational (Java 11-20), concurrent, < 10ms pauses
JAVA_OPTS_ZGC="
  -XX:+UseZGC
  -XX:ZAllocationSpikeTolerance=2
  -XX:ZFragmentationLimit=25
  -Xlog:gc*:file=/var/log/gc-zgc.log
"

# Shenandoah - เหมาะสำหรับ: Low latency, concurrent compaction
# ลักษณะ: Region-based, concurrent evacuation, consistent low latency
JAVA_OPTS_SHENANDOAH="
  -XX:+UseShenandoahGC
  -XX:ShenandoahGCHeuristics=adaptive
  -XX:ShenandoahAllocationThreshold=10
  -XX:ShenandoahInitFreeThreshold=70
"

# Generational ZGC (Java 21+) - ดีที่สุดสำหรับ ultra-low latency
JAVA_OPTS_GEN_ZGC="
  -XX:+UseZGC
  -XX:+ZGenerational
  -XX:MaxGCPauseMillis=5
"
```

```java
// config/GcMetricsConfig.java
package com.example.optimization.config;

import io.micrometer.core.instrument.MeterRegistry;
import io.micrometer.core.instrument.binder.jvm.*;
import lombok.extern.slf4j.Slf4j;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.lang.management.ManagementFactory;
import java.lang.management.MemoryPoolMXBean;
import java.lang.management.GarbageCollectorMXBean;
import java.util.List;

@Slf4j
@Configuration
public class GcMetricsConfig {

    // เพิ่ม JVM metrics
    @Bean
    public JvmGcMetrics jvmGcMetrics() {
        return new JvmGcMetrics();
    }

    @Bean
    public JvmMemoryMetrics jvmMemoryMetrics() {
        return new JvmMemoryMetrics();
    }

    @Bean
    public JvmThreadMetrics jvmThreadMetrics() {
        return new JvmThreadMetrics();
    }

    @Bean
    public ClassLoaderMetrics classLoaderMetrics() {
        return new ClassLoaderMetrics();
    }

    @Bean
    public ProcessorMetrics processorMetrics() {
        return new ProcessorMetrics();
    }

    // Log GC activity ที่สำคัญ
    public void logGcInfo() {
        List<GarbageCollectorMXBean> gcBeans = ManagementFactory.getGarbageCollectorMXBeans();
        for (GarbageCollectorMXBean gc : gcBeans) {
            log.info("GC: {} - Count: {}, Time: {}ms",
                gc.getName(), gc.getCollectionCount(), gc.getCollectionTime());
        }
        
        List<MemoryPoolMXBean> memPools = ManagementFactory.getMemoryPoolMXBeans();
        for (MemoryPoolMXBean pool : memPools) {
            if (pool.getUsage() != null) {
                log.info("Memory Pool: {} - Used: {}MB, Max: {}MB",
                    pool.getName(),
                    pool.getUsage().getUsed() / 1024 / 1024,
                    pool.getUsage().getMax() / 1024 / 1024);
            }
        }
    }
}
```

---

## ขั้นตอนที่ 3203: Heap Sizing Strategies

การกำหนดขนาด heap ให้เหมาะสม

```java
// util/HeapAnalyzer.java
package com.example.optimization.util;

import lombok.extern.slf4j.Slf4j;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Component;

import java.lang.management.ManagementFactory;
import java.lang.management.MemoryMXBean;
import java.lang.management.MemoryUsage;

@Slf4j
@Component
public class HeapAnalyzer {

    private final MemoryMXBean memoryBean = ManagementFactory.getMemoryMXBean();

    // วิเคราะห์ heap ทุก 5 นาที
    @Scheduled(fixedRate = 300000)
    public void analyzeHeap() {
        MemoryUsage heapUsage = memoryBean.getHeapMemoryUsage();
        MemoryUsage nonHeapUsage = memoryBean.getNonHeapMemoryUsage();
        
        double heapUsedMB = heapUsage.getUsed() / 1024.0 / 1024.0;
        double heapMaxMB = heapUsage.getMax() / 1024.0 / 1024.0;
        double heapUtilization = heapUsedMB / heapMaxMB * 100;
        
        log.info("Heap: {:.1f}MB / {:.1f}MB ({:.1f}%)",
            heapUsedMB, heapMaxMB, heapUtilization);
        
        // แจ้งเตือนถ้าใช้ heap เกิน 85%
        if (heapUtilization > 85) {
            log.warn("HIGH HEAP UTILIZATION: {:.1f}% - Consider increasing -Xmx",
                heapUtilization);
        }
        
        // แจ้งเตือนถ้าใช้ heap น้อยเกิน 30% สม่ำเสมอ
        if (heapUtilization < 30) {
            log.info("LOW HEAP UTILIZATION: {:.1f}% - Consider decreasing -Xmx to save memory",
                heapUtilization);
        }
    }

    // คำนวณ recommended heap size
    public HeapRecommendation calculateRecommendation() {
        MemoryUsage heapUsage = memoryBean.getHeapMemoryUsage();
        long usedBytes = heapUsage.getUsed();
        long maxBytes = heapUsage.getMax();
        
        // Recommended: live data * 3 (สำหรับ G1GC)
        long recommendedMax = (long)(usedBytes * 3.5);
        
        // Minimum: ต้องมี 20% headroom
        long minimumMax = (long)(usedBytes * 1.25);
        
        return new HeapRecommendation(
            usedBytes / 1024 / 1024,
            maxBytes / 1024 / 1024,
            minimumMax / 1024 / 1024,
            recommendedMax / 1024 / 1024
        );
    }

    public record HeapRecommendation(
        long currentUsedMB,
        long currentMaxMB,
        long minimumMaxMB,
        long recommendedMaxMB
    ) {}
}
```

---

## ขั้นตอนที่ 3204: Connection Pool Optimization

การ optimize HikariCP (default connection pool ของ Spring Boot)

```yaml
# application.yml - HikariCP Configuration
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/mydb
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
    driver-class-name: org.postgresql.Driver
    
    hikari:
      # Pool size
      minimum-idle: 5          # connections ขั้นต่ำในช่วง idle
      maximum-pool-size: 20    # connections สูงสุด
      
      # Timeouts
      connection-timeout: 30000      # รอ connection สูงสุด 30 วินาที
      idle-timeout: 600000           # connection idle หมดอายุใน 10 นาที
      max-lifetime: 1800000          # connection อายุสูงสุด 30 นาที
      keepalive-time: 300000         # ping ทุก 5 นาทีเพื่อป้องกัน timeout
      
      # Connection validation
      connection-test-query: SELECT 1
      validation-timeout: 5000       # ตรวจสอบ connection ภายใน 5 วินาที
      
      # Leak detection
      leak-detection-threshold: 60000  # แจ้งเตือนถ้า connection ถูกยืมนาน > 1 นาที
      
      # Pool name for monitoring
      pool-name: MainHikariPool
      
      # Register MBeans for monitoring
      register-mbeans: true
      
      # Data source properties
      data-source-properties:
        cachePrepStmts: true
        prepStmtCacheSize: 250
        prepStmtCacheSqlLimit: 2048
        useServerPrepStmts: true
        rewriteBatchedStatements: true
        cacheResultSetMetadata: true
        cacheServerConfiguration: true
        elideSetAutoCommits: true
        maintainTimeStats: false
```

```java
// config/DataSourceConfig.java
package com.example.optimization.config;

import com.zaxxer.hikari.HikariConfig;
import com.zaxxer.hikari.HikariDataSource;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Primary;

import javax.sql.DataSource;

@Slf4j
@Configuration
public class DataSourceConfig {

    @Value("${spring.datasource.url}")
    private String jdbcUrl;

    @Value("${spring.datasource.username}")
    private String username;

    @Value("${spring.datasource.password}")
    private String password;

    // Primary datasource สำหรับ writes (read-write)
    @Bean
    @Primary
    public DataSource primaryDataSource() {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl(jdbcUrl);
        config.setUsername(username);
        config.setPassword(password);
        config.setPoolName("WriterPool");
        config.setMaximumPoolSize(20);
        config.setMinimumIdle(5);
        config.setConnectionTimeout(30000);
        config.setIdleTimeout(600000);
        config.setMaxLifetime(1800000);
        config.setLeakDetectionThreshold(60000);
        
        // PostgreSQL-specific optimizations
        config.addDataSourceProperty("prepareThreshold", "5");
        config.addDataSourceProperty("preparedStatementCacheQueries", "256");
        config.addDataSourceProperty("preparedStatementCacheSizeMiB", "5");
        
        log.info("Primary datasource configured: pool size={}", 20);
        return new HikariDataSource(config);
    }

    // Read-only datasource (replica)
    @Bean("readOnlyDataSource")
    public DataSource readOnlyDataSource(
            @Value("${spring.datasource.replica.url:${spring.datasource.url}}") String replicaUrl) {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl(replicaUrl);
        config.setUsername(username);
        config.setPassword(password);
        config.setPoolName("ReaderPool");
        config.setMaximumPoolSize(30); // Read pool ใหญ่กว่า
        config.setMinimumIdle(10);
        config.setReadOnly(true); // กำหนดเป็น read-only
        config.setConnectionTimeout(15000); // Timeout เร็วกว่า write
        
        log.info("Read-only datasource configured: pool size={}", 30);
        return new HikariDataSource(config);
    }
}
```

```java
// monitoring/ConnectionPoolMonitor.java
package com.example.optimization.monitoring;

import com.zaxxer.hikari.HikariDataSource;
import com.zaxxer.hikari.pool.HikariPool;
import io.micrometer.core.instrument.Gauge;
import io.micrometer.core.instrument.MeterRegistry;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Component;

import javax.sql.DataSource;

@Slf4j
@Component
@RequiredArgsConstructor
public class ConnectionPoolMonitor {

    private final DataSource primaryDataSource;
    private final MeterRegistry meterRegistry;

    // Monitor pool metrics ทุกนาที
    @Scheduled(fixedRate = 60000)
    public void monitorConnectionPool() {
        if (primaryDataSource instanceof HikariDataSource hikariDs) {
            var poolProxy = hikariDs.getHikariPoolMXBean();
            if (poolProxy != null) {
                int active = poolProxy.getActiveConnections();
                int idle = poolProxy.getIdleConnections();
                int waiting = poolProxy.getThreadsAwaitingConnection();
                int total = poolProxy.getTotalConnections();
                
                log.info("Connection Pool - Active: {}, Idle: {}, Waiting: {}, Total: {}",
                    active, idle, waiting, total);
                
                // แจ้งเตือนถ้า connection หมด pool
                if (waiting > 0) {
                    log.warn("Threads waiting for connection: {} - Consider increasing pool size",
                        waiting);
                }
                
                // แจ้งเตือนถ้า pool utilization สูง
                double utilization = (double) active / total * 100;
                if (utilization > 80) {
                    log.warn("High connection pool utilization: {:.1f}%", utilization);
                }
            }
        }
    }
}
```

---

## ขั้นตอนที่ 3205: Thread Pool Tuning

การ configure thread pools สำหรับ async operations

```java
// config/ThreadPoolConfig.java
package com.example.optimization.config;

import io.micrometer.core.instrument.MeterRegistry;
import io.micrometer.core.instrument.binder.jvm.ExecutorServiceMetrics;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.scheduling.annotation.EnableAsync;
import org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor;

import java.util.concurrent.*;

@Slf4j
@Configuration
@EnableAsync
public class ThreadPoolConfig {

    @Value("${app.thread-pool.core-size:10}")
    private int coreSize;

    @Value("${app.thread-pool.max-size:50}")
    private int maxSize;

    @Value("${app.thread-pool.queue-capacity:1000}")
    private int queueCapacity;

    // Main async executor
    @Bean("taskExecutor")
    public ThreadPoolTaskExecutor taskExecutor(MeterRegistry meterRegistry) {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        
        // Core = จำนวน threads ที่ active อยู่เสมอ
        executor.setCorePoolSize(coreSize);
        
        // Max = จำนวน threads สูงสุด (เพิ่มเมื่อ queue เต็ม)
        executor.setMaxPoolSize(maxSize);
        
        // Queue capacity = จำนวน tasks ที่รอในคิว
        executor.setQueueCapacity(queueCapacity);
        
        // Thread naming
        executor.setThreadNamePrefix("app-async-");
        
        // Rejection policy เมื่อ pool และ queue เต็ม
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        
        // Wait for tasks ก่อน shutdown
        executor.setWaitForTasksToCompleteOnShutdown(true);
        executor.setAwaitTerminationSeconds(60);
        
        executor.initialize();
        
        // เพิ่ม metrics monitoring
        ExecutorServiceMetrics.monitor(
            meterRegistry,
            executor.getThreadPoolExecutor(),
            "app-async-pool"
        );
        
        log.info("Task executor configured: core={}, max={}, queue={}",
            coreSize, maxSize, queueCapacity);
        return executor;
    }

    // I/O-intensive executor - threads มากกว่า CPU cores
    @Bean("ioExecutor")
    public ThreadPoolTaskExecutor ioExecutor(MeterRegistry meterRegistry) {
        int cpuCores = Runtime.getRuntime().availableProcessors();
        int ioThreads = cpuCores * 4; // I/O ใช้ threads มากกว่า
        
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(ioThreads);
        executor.setMaxPoolSize(ioThreads * 2);
        executor.setQueueCapacity(500);
        executor.setThreadNamePrefix("io-async-");
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        executor.initialize();
        
        ExecutorServiceMetrics.monitor(meterRegistry,
            executor.getThreadPoolExecutor(), "io-pool");
        
        log.info("I/O executor configured: threads={}", ioThreads);
        return executor;
    }

    // CPU-intensive executor - ใช้ threads เท่ากับ CPU cores
    @Bean("cpuExecutor")
    public ThreadPoolTaskExecutor cpuExecutor(MeterRegistry meterRegistry) {
        int cpuCores = Runtime.getRuntime().availableProcessors();
        
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(cpuCores);
        executor.setMaxPoolSize(cpuCores);
        executor.setQueueCapacity(200);
        executor.setThreadNamePrefix("cpu-async-");
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.AbortPolicy());
        executor.initialize();
        
        ExecutorServiceMetrics.monitor(meterRegistry,
            executor.getThreadPoolExecutor(), "cpu-pool");
        
        log.info("CPU executor configured: threads={}", cpuCores);
        return executor;
    }

    // Scheduled tasks executor
    @Bean("scheduledExecutor")
    public ScheduledExecutorService scheduledExecutor() {
        return Executors.newScheduledThreadPool(
            Runtime.getRuntime().availableProcessors(),
            r -> {
                Thread t = new Thread(r);
                t.setName("scheduled-" + t.getId());
                t.setDaemon(true);
                return t;
            }
        );
    }
}
```

---

## ขั้นตอนที่ 3206: Cache Warming Strategies

การ warm up caches ก่อนที่ traffic จะเข้ามา

```java
// cache/CacheWarmupService.java
package com.example.optimization.cache;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.boot.context.event.ApplicationReadyEvent;
import org.springframework.cache.CacheManager;
import org.springframework.context.event.EventListener;
import org.springframework.scheduling.annotation.Async;
import org.springframework.stereotype.Service;

import java.time.Duration;
import java.time.Instant;
import java.util.List;
import java.util.concurrent.CompletableFuture;

@Slf4j
@Service
@RequiredArgsConstructor
public class CacheWarmupService {

    private final CacheManager cacheManager;
    private final ProductRepository productRepository;
    private final CategoryRepository categoryRepository;
    private final ConfigRepository configRepository;

    // เริ่ม warmup หลัง application พร้อมใช้งาน
    @EventListener(ApplicationReadyEvent.class)
    @Async("taskExecutor")
    public void warmupCachesOnStartup() {
        Instant start = Instant.now();
        log.info("Starting cache warmup...");
        
        // Warmup caches แบบ parallel
        CompletableFuture<Void> productWarmup = CompletableFuture.runAsync(
            this::warmupProductCache);
        CompletableFuture<Void> categoryWarmup = CompletableFuture.runAsync(
            this::warmupCategoryCache);
        CompletableFuture<Void> configWarmup = CompletableFuture.runAsync(
            this::warmupConfigCache);
        
        CompletableFuture.allOf(productWarmup, categoryWarmup, configWarmup).join();
        
        Duration elapsed = Duration.between(start, Instant.now());
        log.info("Cache warmup completed in {}ms", elapsed.toMillis());
    }

    // Warmup product cache - top 1000 products
    private void warmupProductCache() {
        try {
            log.info("Warming up product cache...");
            List<Product> topProducts = productRepository.findTopActiveProducts(1000);
            
            var cache = cacheManager.getCache("products");
            if (cache != null) {
                topProducts.forEach(product -> {
                    cache.put(product.getId(), product);
                    cache.put("sku:" + product.getSku(), product);
                });
            }
            
            log.info("Product cache warmed up: {} products", topProducts.size());
        } catch (Exception e) {
            log.error("Failed to warmup product cache", e);
        }
    }

    // Warmup category cache - ทุก categories
    private void warmupCategoryCache() {
        try {
            log.info("Warming up category cache...");
            List<Category> categories = categoryRepository.findAllActive();
            
            var cache = cacheManager.getCache("categories");
            if (cache != null) {
                categories.forEach(cat -> cache.put(cat.getId(), cat));
                cache.put("all", categories);
            }
            
            log.info("Category cache warmed up: {} categories", categories.size());
        } catch (Exception e) {
            log.error("Failed to warmup category cache", e);
        }
    }

    // Warmup config cache
    private void warmupConfigCache() {
        try {
            log.info("Warming up config cache...");
            List<AppConfig> configs = configRepository.findAll();
            
            var cache = cacheManager.getCache("configs");
            if (cache != null) {
                configs.forEach(config -> cache.put(config.getKey(), config.getValue()));
            }
            
            log.info("Config cache warmed up: {} configs", configs.size());
        } catch (Exception e) {
            log.error("Failed to warmup config cache", e);
        }
    }

    // Scheduled re-warmup ทุกชั่วโมง
    @org.springframework.scheduling.annotation.Scheduled(cron = "0 0 * * * *")
    @Async("taskExecutor")
    public void scheduledCacheWarmup() {
        log.info("Running scheduled cache re-warmup...");
        warmupCachesOnStartup();
    }
}
```

---

## ขั้นตอนที่ 3207: Lazy Loading vs Eager Loading

การเลือกใช้ lazy/eager loading ให้เหมาะสม

```java
// entity/Product.java
package com.example.optimization.entity;

import lombok.*;
import javax.persistence.*;
import java.math.BigDecimal;
import java.util.ArrayList;
import java.util.List;
import java.util.Set;

@Entity
@Table(name = "products", indexes = {
    @Index(name = "idx_product_sku", columnList = "sku"),
    @Index(name = "idx_product_category", columnList = "category_id"),
    @Index(name = "idx_product_status", columnList = "status")
})
@Getter
@Setter
@NoArgsConstructor
public class Product {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(unique = true)
    private String sku;
    
    private String name;
    private BigDecimal price;
    private String status;
    
    // EAGER - โหลดพร้อมกัน (เหมาะเมื่อใช้เสมอ)
    @ManyToOne(fetch = FetchType.EAGER)
    @JoinColumn(name = "category_id")
    private Category category;
    
    // LAZY - โหลดเมื่อเรียกใช้ (เหมาะเมื่อไม่ได้ใช้บ่อย)
    @OneToMany(mappedBy = "product", fetch = FetchType.LAZY,
        cascade = CascadeType.ALL)
    private List<ProductImage> images = new ArrayList<>();
    
    @OneToMany(mappedBy = "product", fetch = FetchType.LAZY)
    private List<Review> reviews = new ArrayList<>();
    
    // LAZY for large collections
    @ManyToMany(fetch = FetchType.LAZY)
    @JoinTable(name = "product_tags",
        joinColumns = @JoinColumn(name = "product_id"),
        inverseJoinColumns = @JoinColumn(name = "tag_id"))
    private Set<Tag> tags;
}
```

```java
// repository/ProductRepository.java
package com.example.optimization.repository;

import com.example.optimization.entity.Product;
import org.springframework.data.jpa.repository.EntityGraph;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.springframework.stereotype.Repository;

import java.util.List;
import java.util.Optional;

@Repository
public interface ProductRepository extends JpaRepository<Product, Long> {

    // Eager load images ด้วย EntityGraph (แทน N+1 problem)
    @EntityGraph(attributePaths = {"images", "category"})
    Optional<Product> findWithImagesBySku(String sku);

    // Fetch join สำหรับ collection
    @Query("SELECT DISTINCT p FROM Product p " +
           "LEFT JOIN FETCH p.images " +
           "LEFT JOIN FETCH p.tags " +
           "WHERE p.status = 'ACTIVE'")
    List<Product> findActiveProductsWithDetails();

    // Projection สำหรับ list views (เร็วกว่า entity)
    @Query("SELECT new com.example.optimization.dto.ProductSummary(" +
           "p.id, p.sku, p.name, p.price, p.status) " +
           "FROM Product p WHERE p.status = 'ACTIVE' " +
           "ORDER BY p.name")
    List<ProductSummary> findProductSummaries();

    // Top products สำหรับ cache warmup
    @Query("SELECT p FROM Product p WHERE p.status = 'ACTIVE' " +
           "ORDER BY p.viewCount DESC")
    List<Product> findTopActiveProducts(int limit);
}
```

---

## ขั้นตอนที่ 3208: HTTP/2 และ Connection Multiplexing

การ configure HTTP/2 ใน Spring Boot

```yaml
# application.yml - HTTP/2 Configuration
server:
  port: 8443
  ssl:
    enabled: true
    key-store: classpath:keystore.p12
    key-store-password: ${SSL_KEYSTORE_PASSWORD}
    key-store-type: PKCS12
    key-alias: myapp
    protocol: TLS
    enabled-protocols: TLSv1.2,TLSv1.3
    ciphers:
      - TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
      - TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
  
  http2:
    enabled: true    # เปิดใช้ HTTP/2
  
  tomcat:
    threads:
      min-spare: 10
      max: 200       # Max threads
    max-connections: 10000   # Max connections
    accept-count: 100        # Queue size เมื่อ threads เต็ม
    connection-timeout: 20000
    keep-alive-timeout: 60000
    max-keep-alive-requests: 100
    
    # Compression
    compression:
      enabled: true
      mime-types: application/json,text/html,text/css,application/javascript
      min-response-size: 1024   # Compress เมื่อ response > 1KB
```

```java
// config/TomcatConfig.java
package com.example.optimization.config;

import lombok.extern.slf4j.Slf4j;
import org.apache.coyote.http2.Http2Protocol;
import org.springframework.boot.web.embedded.tomcat.TomcatServletWebServerFactory;
import org.springframework.boot.web.server.WebServerFactoryCustomizer;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Slf4j
@Configuration
public class TomcatConfig {

    @Bean
    public WebServerFactoryCustomizer<TomcatServletWebServerFactory> tomcatCustomizer() {
        return factory -> {
            factory.addConnectorCustomizers(connector -> {
                // กำหนด HTTP/2 protocol
                connector.addUpgradeProtocol(new Http2Protocol());
                
                // Tune connector settings
                connector.setProperty("maxConnections", "10000");
                connector.setProperty("acceptCount", "100");
                connector.setProperty("connectionTimeout", "20000");
                connector.setProperty("maxKeepAliveRequests", "100");
                connector.setProperty("keepAliveTimeout", "60000");
                
                // NIO connector settings
                connector.setProperty("socket.soKeepAlive", "true");
                connector.setProperty("socket.performanceBandwidth", "2");
                connector.setProperty("socket.performanceConnectionTime", "2");
                connector.setProperty("socket.performanceLatency", "2");
                
                log.info("Tomcat connector customized with HTTP/2 support");
            });
        };
    }
}
```

---

## ขั้นตอนที่ 3209: Application Performance Monitoring

การวัดและตรวจสอบ performance ของ application

```java
// monitoring/PerformanceMonitor.java
package com.example.optimization.monitoring;

import io.micrometer.core.annotation.Timed;
import io.micrometer.core.instrument.*;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.springframework.stereotype.Component;

import java.util.concurrent.TimeUnit;

@Slf4j
@Aspect
@Component
@RequiredArgsConstructor
public class PerformanceMonitor {

    private final MeterRegistry meterRegistry;

    // วัด execution time ของ service methods
    @Around("@annotation(io.micrometer.core.annotation.Timed)")
    public Object measureMethodTime(ProceedingJoinPoint joinPoint) throws Throwable {
        String className = joinPoint.getTarget().getClass().getSimpleName();
        String methodName = joinPoint.getSignature().getName();
        String metricName = "method.execution.time";
        
        long start = System.nanoTime();
        Object result = null;
        boolean success = true;
        
        try {
            result = joinPoint.proceed();
            return result;
        } catch (Exception e) {
            success = false;
            throw e;
        } finally {
            long duration = System.nanoTime() - start;
            
            // บันทึก metrics
            Timer.builder(metricName)
                .tag("class", className)
                .tag("method", methodName)
                .tag("success", String.valueOf(success))
                .register(meterRegistry)
                .record(duration, TimeUnit.NANOSECONDS);
            
            // Log ถ้าช้าเกิน 1 วินาที
            if (duration > 1_000_000_000L) {
                log.warn("Slow method detected: {}.{} took {}ms",
                    className, methodName, duration / 1_000_000);
            }
        }
    }
}
```

```java
// service/ProductService.java
package com.example.optimization.service;

import io.micrometer.core.annotation.Timed;
import io.micrometer.core.instrument.Counter;
import io.micrometer.core.instrument.MeterRegistry;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.cache.annotation.Cacheable;
import org.springframework.stereotype.Service;

import java.util.List;

@Slf4j
@Service
@RequiredArgsConstructor
public class ProductService {

    private final ProductRepository productRepository;
    private final MeterRegistry meterRegistry;

    @Timed(value = "product.lookup", description = "Time to lookup product")
    @Cacheable(value = "products", key = "#id")
    public Product getProduct(Long id) {
        meterRegistry.counter("product.cache.miss", "operation", "getById").increment();
        return productRepository.findById(id)
            .orElseThrow(() -> new ProductNotFoundException(id));
    }

    @Timed(value = "product.search", description = "Time to search products")
    public List<ProductSummary> searchProducts(SearchCriteria criteria) {
        return productRepository.findProductSummaries();
    }

    // Batch loading ลด N+1 problem
    @Timed(value = "product.batch.load")
    public List<Product> getProductsBatch(List<Long> ids) {
        return productRepository.findAllById(ids);
    }
}
```

---

## ขั้นตอนที่ 3210: Production-Ready Configuration

การ configure Spring Boot สำหรับ production environment

```yaml
# application-production.yml
spring:
  # JPA/Hibernate ใน production
  jpa:
    hibernate:
      ddl-auto: validate     # ตรวจสอบ schema แต่ไม่แก้ไข
    show-sql: false           # ปิด SQL logging
    open-in-view: false       # ปิด OSIV (ลด connection holding)
    properties:
      hibernate:
        # Query optimization
        jdbc.batch_size: 50
        order_inserts: true
        order_updates: true
        batch_versioned_data: true
        
        # Cache (L2 cache)
        cache.use_second_level_cache: true
        cache.use_query_cache: true
        cache.region.factory_class: org.hibernate.cache.jcache.JCacheRegionFactory
        javax.cache.provider: org.ehcache.jsr107.EhcacheCachingProvider
        
        # Statistics
        generate_statistics: false  # ปิดใน production เพื่อ performance
        
        # Fetch size
        jdbc.fetch_size: 50

  # Redis cache ใน production
  redis:
    host: ${REDIS_HOST:localhost}
    port: ${REDIS_PORT:6379}
    password: ${REDIS_PASSWORD}
    timeout: 2000
    lettuce:
      pool:
        max-active: 20
        max-idle: 10
        min-idle: 5
        max-wait: 1000

  # Jackson
  jackson:
    default-property-inclusion: non_null
    serialization:
      write-dates-as-timestamps: false
      fail-on-empty-beans: false
    deserialization:
      fail-on-unknown-properties: false

# Management
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus,threaddump,heapdump,loggers
      base-path: /actuator
  endpoint:
    health:
      show-details: when-authorized
      show-components: when-authorized
      probes:
        enabled: true  # Kubernetes liveness/readiness probes
  health:
    circuitbreakers:
      enabled: true
    ratelimiters:
      enabled: true
  metrics:
    distribution:
      percentiles-histogram:
        http.server.requests: true
      percentiles:
        http.server.requests: 0.5,0.75,0.95,0.99
      sla:
        http.server.requests: 100ms,200ms,500ms,1s
    export:
      prometheus:
        enabled: true
  tracing:
    sampling:
      probability: 0.1   # Sample 10% ของ requests

# Logging
logging:
  level:
    root: WARN
    com.example: INFO
    org.springframework: WARN
    org.hibernate: WARN
    com.zaxxer.hikari: INFO
  pattern:
    console: "%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{36} [%X{traceId}] - %msg%n"
  file:
    name: /var/log/app/application.log
    max-size: 100MB
    max-history: 30
    total-size-cap: 1GB
```

---

## ขั้นตอนที่ 3211: Graceful Shutdown

การ shutdown application อย่างปลอดภัย

```java
// config/GracefulShutdownConfig.java
package com.example.optimization.config;

import lombok.extern.slf4j.Slf4j;
import org.springframework.boot.web.embedded.tomcat.TomcatServletWebServerFactory;
import org.springframework.boot.web.server.WebServerFactoryCustomizer;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.time.Duration;

@Slf4j
@Configuration
public class GracefulShutdownConfig {

    @Bean
    public WebServerFactoryCustomizer<TomcatServletWebServerFactory> gracefulShutdown() {
        return factory -> factory.addContextCustomizers(context -> {
            log.info("Configuring graceful shutdown...");
        });
    }
}
```

```yaml
# application.yml - Graceful Shutdown
server:
  shutdown: graceful        # รอให้ requests ปัจจุบันเสร็จก่อน shutdown

spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s   # รอสูงสุด 30 วินาที
```

```java
// lifecycle/ApplicationShutdownHandler.java
package com.example.optimization.lifecycle;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.context.ApplicationListener;
import org.springframework.context.event.ContextClosedEvent;
import org.springframework.stereotype.Component;

@Slf4j
@Component
@RequiredArgsConstructor
public class ApplicationShutdownHandler
        implements ApplicationListener<ContextClosedEvent> {

    private final CacheManager cacheManager;
    private final MetricsPublisher metricsPublisher;

    @Override
    public void onApplicationEvent(ContextClosedEvent event) {
        log.info("Application shutting down...");
        
        try {
            // บันทึก metrics สุดท้าย
            metricsPublisher.publishFinalMetrics();
            log.info("Final metrics published");
        } catch (Exception e) {
            log.error("Error publishing final metrics", e);
        }
        
        try {
            // Clear caches
            cacheManager.getCacheNames()
                .forEach(name -> {
                    var cache = cacheManager.getCache(name);
                    if (cache != null) {
                        cache.clear();
                    }
                });
            log.info("Caches cleared");
        } catch (Exception e) {
            log.error("Error clearing caches", e);
        }
        
        log.info("Shutdown complete");
    }
}
```

---

## ขั้นตอนที่ 3212: Performance Testing

การทดสอบ performance ของ application

```java
// test/PerformanceTest.java
package com.example.optimization;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.web.client.TestRestTemplate;
import org.springframework.boot.test.web.server.LocalServerPort;
import org.springframework.http.ResponseEntity;

import java.time.Duration;
import java.time.Instant;
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicInteger;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class PerformanceTest {

    @LocalServerPort
    private int port;

    @Autowired
    private TestRestTemplate restTemplate;

    @Test
    void productLookup_shouldRespondFastEnough() throws InterruptedException {
        // Warmup
        for (int i = 0; i < 10; i++) {
            restTemplate.getForEntity("/api/products/1", String.class);
        }

        // Measure
        List<Long> responseTimes = new ArrayList<>();
        for (int i = 0; i < 100; i++) {
            Instant start = Instant.now();
            restTemplate.getForEntity("/api/products/1", String.class);
            responseTimes.add(Duration.between(start, Instant.now()).toMillis());
        }

        double avgMs = responseTimes.stream()
            .mapToLong(Long::longValue).average().orElse(0);
        
        long p95Ms = responseTimes.stream()
            .sorted()
            .skip((long)(responseTimes.size() * 0.95))
            .findFirst()
            .orElse(0L);

        assertThat(avgMs).isLessThan(100); // avg < 100ms
        assertThat(p95Ms).isLessThan(200); // p95 < 200ms
    }

    @Test
    void concurrentRequests_shouldHandleLoad() throws InterruptedException {
        int threads = 50;
        int requestsPerThread = 20;
        
        ExecutorService executor = Executors.newFixedThreadPool(threads);
        CountDownLatch latch = new CountDownLatch(threads);
        AtomicInteger successCount = new AtomicInteger(0);
        AtomicInteger errorCount = new AtomicInteger(0);

        for (int i = 0; i < threads; i++) {
            executor.submit(() -> {
                for (int j = 0; j < requestsPerThread; j++) {
                    try {
                        ResponseEntity<String> response = restTemplate
                            .getForEntity("/api/products", String.class);
                        if (response.getStatusCode().is2xxSuccessful()) {
                            successCount.incrementAndGet();
                        } else {
                            errorCount.incrementAndGet();
                        }
                    } catch (Exception e) {
                        errorCount.incrementAndGet();
                    }
                }
                latch.countDown();
            });
        }

        latch.await(60, TimeUnit.SECONDS);
        executor.shutdown();

        int total = threads * requestsPerThread;
        double successRate = (double) successCount.get() / total * 100;
        
        assertThat(successRate).isGreaterThan(99.0); // > 99% success rate
    }
}
```

---

## ขั้นตอนที่ 3213: Kubernetes Deployment Optimization

```yaml
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spring-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: spring-app
  template:
    metadata:
      labels:
        app: spring-app
    spec:
      containers:
        - name: spring-app
          image: myapp:latest
          
          # Resource limits สำคัญมาก
          resources:
            requests:
              memory: "512Mi"
              cpu: "250m"
            limits:
              memory: "2Gi"
              cpu: "2"
          
          # JVM options ที่ optimize สำหรับ container
          env:
            - name: JAVA_OPTS
              value: >-
                -XX:+UseContainerSupport
                -XX:MaxRAMPercentage=75.0
                -XX:InitialRAMPercentage=50.0
                -XX:+UseG1GC
                -XX:MaxGCPauseMillis=200
                -XX:+ExitOnOutOfMemoryError
            - name: SPRING_PROFILES_ACTIVE
              value: production
          
          # Health probes
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
            failureThreshold: 3
          
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            initialDelaySeconds: 60
            periodSeconds: 30
            failureThreshold: 5
          
          # Graceful shutdown
          lifecycle:
            preStop:
              exec:
                command: ["/bin/sh", "-c", "sleep 10"]
          terminationGracePeriodSeconds: 60

---
# HorizontalPodAutoscaler
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: spring-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: spring-app
  minReplicas: 2
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
```

---

## สรุป Part 90

ในส่วนนี้เราได้เรียนรู้:
- **JVM Tuning** - JVM flags, heap sizing, metaspace configuration
- **GC Selection** - G1GC, ZGC, Shenandoah เลือกใช้อย่างไร
- **Heap Sizing** - วิเคราะห์และกำหนดขนาด heap อย่างเหมาะสม
- **Connection Pool** - HikariCP optimization สำหรับ production
- **Thread Pool** - Task, I/O, CPU executors
- **Cache Warming** - Warm up caches ก่อน traffic เข้ามา
- **Lazy vs Eager Loading** - เลือกใช้ให้เหมาะกับ use case
- **HTTP/2** - Connection multiplexing สำหรับ performance
- **Graceful Shutdown** - Shutdown อย่างปลอดภัย
- **Performance Testing** - วัด response time และ concurrency
- **Kubernetes** - Deploy และ scale อย่างเหมาะสม

---

## บทสรุปรวม Parts 86-90

เราได้เรียนรู้ Spring Boot ในระดับ World-Class ครอบคลุม:

| Part | หัวข้อ | ขั้นตอน |
|------|--------|---------|
| 86 | Integration Patterns | 3041-3080 |
| 87 | Reactive Security | 3081-3120 |
| 88 | Data Streaming | 3121-3160 |
| 89 | Advanced Patterns | 3161-3200 |
| 90 | Production Optimization | 3201-3240 |

---

*[← Part 89: Advanced Patterns](./part-89-advanced-patterns.md) | [Part 91: Cloud Native →](./part-91-cloud-native.md)*

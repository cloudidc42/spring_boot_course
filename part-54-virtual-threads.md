# Part 54: Java 21 Virtual Threads (Project Loom)
## ขั้นตอนที่ 1761-1800

**ระดับ:** Advanced  
**เวลาเรียน:** 3-4 ชั่วโมง  
**เป้าหมาย:** เข้าใจ Virtual Threads ของ Java 21, เปิดใช้งานใน Spring Boot 3.2, รู้ว่าควรใช้เมื่อไร, ทำ Structured Concurrency และวัดผลลัพธ์

---

## ขั้นตอนที่ 1761: Virtual Threads คืออะไร?

**Virtual Threads** (Project Loom) เป็นหนึ่งในฟีเจอร์ที่ใหญ่ที่สุดของ Java 21 ซึ่งช่วยแก้ปัญหา **Thread-per-Request** model แบบดั้งเดิม

### ปัญหาของ Platform Threads

```
Traditional Web Server (Platform Threads):
┌─────────────────────────────────────────┐
│  Thread Pool: 200 threads max           │
│                                         │
│  Request 1 → Thread 1 → [BLOCKED: DB]  │
│  Request 2 → Thread 2 → [BLOCKED: DB]  │
│  Request 3 → Thread 3 → [BLOCKED: DB]  │
│  ...                                    │
│  Request 200 → Thread 200 → [BLOCKED]  │
│                                         │
│  Request 201 → QUEUE (รอ thread ว่าง)   │
│  Request 202 → QUEUE ...               │
└─────────────────────────────────────────┘

ปัญหา: Thread ส่วนใหญ่รอ I/O อยู่เฉยๆ แต่กิน memory 1-2 MB ต่อ thread
```

### Virtual Threads แก้ปัญหาอย่างไร

```
Virtual Threads Model:
┌─────────────────────────────────────────┐
│  Carrier Threads: N threads (= CPU cores) │
│  Virtual Threads: Millions!              │
│                                         │
│  VT-1 → Carrier-1 → [BLOCKED: DB]      │
│         ↓ (park VT-1, mount VT-2)       │
│         Carrier-1 → [VT-2 working]      │
│                                         │
│  เมื่อ VT-1 ได้รับ response:            │
│  VT-1 → Carrier-? (available carrier)   │
└─────────────────────────────────────────┘

Virtual Thread ใช้ memory แค่ ~1KB (เทียบกับ 1MB ของ Platform Thread)
สามารถมี Virtual Thread ได้นับล้านตัวพร้อมกัน!
```

---

## ขั้นตอนที่ 1762: เปิดใช้ Virtual Threads ใน Spring Boot 3.2

### วิธีที่ 1: Properties (ง่ายที่สุด)

```yaml
# application.yml
spring:
  threads:
    virtual:
      enabled: true  # เปิด Virtual Threads ทั้ง Tomcat และ @Async
```

### วิธีที่ 2: Java Config (ควบคุมได้มากกว่า)

```java
// src/main/java/com/example/config/VirtualThreadConfig.java
package com.example.config;

import org.springframework.boot.web.embedded.tomcat.TomcatProtocolHandlerCustomizer;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.scheduling.annotation.EnableAsync;
import org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor;

import java.util.concurrent.Executor;
import java.util.concurrent.Executors;

@Configuration
@EnableAsync
public class VirtualThreadConfig {
    
    /**
     * ให้ Tomcat ใช้ Virtual Threads แทน Platform Threads
     * ทำให้รองรับ concurrent requests ได้มากขึ้นมาก
     */
    @Bean
    public TomcatProtocolHandlerCustomizer<?> tomcatVirtualThreads() {
        return protocolHandler ->
            protocolHandler.setExecutor(Executors.newVirtualThreadPerTaskExecutor());
    }
    
    /**
     * Executor สำหรับ @Async methods ที่ใช้ Virtual Threads
     */
    @Bean(name = "virtualThreadExecutor")
    public Executor virtualThreadExecutor() {
        return Executors.newVirtualThreadPerTaskExecutor();
    }
    
    /**
     * ถ้าต้องการ Platform Thread pool สำหรับงานที่ CPU-intensive
     */
    @Bean(name = "cpuBoundExecutor")
    public Executor cpuBoundExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(Runtime.getRuntime().availableProcessors());
        executor.setMaxPoolSize(Runtime.getRuntime().availableProcessors() * 2);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("cpu-");
        executor.initialize();
        return executor;
    }
}
```

---

## ขั้นตอนที่ 1763: Virtual Threads vs Platform Threads

```java
// src/main/java/com/example/demo/ThreadComparisonDemo.java
package com.example.demo;

import java.time.Duration;
import java.time.Instant;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.Executors;
import java.util.concurrent.atomic.AtomicInteger;

/**
 * Demo เปรียบเทียบ Virtual Threads vs Platform Threads
 * สำหรับ I/O-bound workload
 */
public class ThreadComparisonDemo {
    
    // จำลอง I/O operation (เช่น database query)
    static void simulateIoOperation() throws InterruptedException {
        Thread.sleep(100); // 100ms I/O wait
    }
    
    public static void main(String[] args) throws InterruptedException {
        int taskCount = 10_000;
        
        System.out.println("Testing with " + taskCount + " concurrent I/O tasks\n");
        
        // Test 1: Platform Threads (ผ่าน fixed thread pool)
        Instant start1 = Instant.now();
        try (var executor = Executors.newFixedThreadPool(200)) {
            CountDownLatch latch1 = new CountDownLatch(taskCount);
            for (int i = 0; i < taskCount; i++) {
                executor.submit(() -> {
                    try {
                        simulateIoOperation();
                    } catch (InterruptedException e) {
                        Thread.currentThread().interrupt();
                    } finally {
                        latch1.countDown();
                    }
                });
            }
            latch1.await();
        }
        Duration time1 = Duration.between(start1, Instant.now());
        System.out.printf("Platform Threads (200 pool): %d ms%n", time1.toMillis());
        
        // Test 2: Virtual Threads
        Instant start2 = Instant.now();
        try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
            CountDownLatch latch2 = new CountDownLatch(taskCount);
            for (int i = 0; i < taskCount; i++) {
                executor.submit(() -> {
                    try {
                        simulateIoOperation();
                    } catch (InterruptedException e) {
                        Thread.currentThread().interrupt();
                    } finally {
                        latch2.countDown();
                    }
                });
            }
            latch2.await();
        }
        Duration time2 = Duration.between(start2, Instant.now());
        System.out.printf("Virtual Threads: %d ms%n", time2.toMillis());
        
        System.out.printf("\nVirtual Threads %.1fx faster!%n",
            (double) time1.toMillis() / time2.toMillis());
    }
}
```

### ผลลัพธ์ที่คาดหวัง

```
Testing with 10,000 concurrent I/O tasks

Platform Threads (200 pool): 5,234 ms
Virtual Threads: 112 ms

Virtual Threads 46.7x faster!
```

---

## ขั้นตอนที่ 1764: เมื่อไรควรใช้ Virtual Threads

### ใช้ Virtual Threads เมื่อ (I/O-bound tasks):

```java
// 1. HTTP calls ไปยัง external services
@Service
public class PaymentGatewayService {
    
    private final RestClient restClient;
    
    // Virtual Thread จะ park ขณะรอ HTTP response
    // ไม่บล็อค carrier thread ทำให้ตัวอื่นทำงานได้
    public PaymentResult chargeCard(PaymentRequest request) {
        return restClient.post()
            .uri("https://payment-gateway.com/charge")
            .body(request)
            .retrieve()
            .body(PaymentResult.class);
    }
}
```

```java
// 2. Database queries
@Service
@RequiredArgsConstructor
public class OrderReportService {
    
    private final OrderRepository orderRepository;
    private final ProductRepository productRepository;
    
    // แต่ละ DB call จะ park virtual thread ขณะรอ
    public ReportData generateReport(LocalDate date) {
        var orders = orderRepository.findByDate(date);         // รอ DB
        var products = productRepository.findTopSelling(date); // รอ DB
        var revenue = orderRepository.calculateRevenue(date);  // รอ DB
        return new ReportData(orders, products, revenue);
    }
}
```

```java
// 3. File I/O operations
@Service
public class FileProcessingService {
    
    public void processUploadedFile(MultipartFile file) throws IOException {
        // Virtual Thread park ขณะ read/write file
        byte[] content = file.getBytes();
        processContent(content);
        saveToStorage(content); // ใช้เวลาเขียน disk
    }
}
```

### อย่าใช้ Virtual Threads เมื่อ (CPU-bound tasks):

```java
// CPU-bound: ใช้ Platform Thread pool ขนาดเหมาะสม
@Service
public class ImageProcessingService {
    
    @Async("cpuBoundExecutor")  // ใช้ Platform Thread pool
    public CompletableFuture<byte[]> resizeImage(byte[] imageData, int width, int height) {
        // การประมวลผลภาพ = CPU intensive
        // Virtual Thread จะ pin carrier thread ตลอดเวลา (ไม่มีประโยชน์)
        return CompletableFuture.completedFuture(
            imageProcessor.resize(imageData, width, height)
        );
    }
}
```

---

## ขั้นตอนที่ 1765: Thread Local Variables กับ Virtual Threads

Virtual Threads รองรับ `ThreadLocal` แต่มีข้อระวัง:

```java
// ปัญหา: ThreadLocal ถูก pool ใน Platform Threads
// แต่ใน Virtual Threads จะมี thread ใหม่ต่อ task
// ทำให้ object ใน ThreadLocal ไม่ถูก reuse = garbage ถูกสร้างเยอะ

// ❌ ไม่ดีสำหรับ Virtual Threads (ถ้า ThreadLocal expensive to create)
ThreadLocal<ExpensiveObject> threadLocal = ThreadLocal.withInitial(ExpensiveObject::new);

// ✅ ดีกว่า: ใช้ Scoped Values (Java 21 Preview)
ScopedValue<RequestContext> REQUEST_CONTEXT = ScopedValue.newInstance();

// ใน handler:
ScopedValue.where(REQUEST_CONTEXT, new RequestContext(userId, traceId))
    .run(() -> {
        // code ใน scope สามารถอ่าน REQUEST_CONTEXT ได้
        processRequest();
    });
```

```java
// src/main/java/com/example/context/RequestContextHolder.java
package com.example.context;

/**
 * Scoped Value สำหรับ propagate request context ใน Virtual Threads
 * ดีกว่า ThreadLocal เพราะ immutable และ thread-safe
 */
public class RequestContextHolder {
    
    // Java 21: ScopedValue เป็น replacement ที่ดีกว่า ThreadLocal
    public static final ScopedValue<RequestContext> CURRENT = ScopedValue.newInstance();
    
    public static RequestContext current() {
        return CURRENT.orElseThrow(() -> 
            new IllegalStateException("No request context available"));
    }
    
    public static <T> T withContext(RequestContext context, 
                                     java.util.concurrent.Callable<T> task) throws Exception {
        return ScopedValue.where(CURRENT, context).call(task);
    }
}
```

```java
// src/main/java/com/example/filter/RequestContextFilter.java
package com.example.filter;

import com.example.context.RequestContextHolder;
import jakarta.servlet.*;
import jakarta.servlet.http.HttpServletRequest;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Component;

import java.io.IOException;

@Component
@Slf4j
public class RequestContextFilter implements Filter {
    
    @Override
    public void doFilter(ServletRequest request, ServletResponse response, 
                         FilterChain chain) throws IOException, ServletException {
        
        var httpRequest = (HttpServletRequest) request;
        var context = new RequestContext(
            httpRequest.getHeader("X-Trace-Id"),
            httpRequest.getHeader("X-User-Id"),
            httpRequest.getRemoteAddr()
        );
        
        try {
            ScopedValue.where(RequestContextHolder.CURRENT, context)
                .run(() -> {
                    try {
                        chain.doFilter(request, response);
                    } catch (Exception e) {
                        throw new RuntimeException(e);
                    }
                });
        } catch (RuntimeException e) {
            if (e.getCause() instanceof IOException ioe) throw ioe;
            if (e.getCause() instanceof ServletException se) throw se;
            throw e;
        }
    }
}
```

---

## ขั้นตอนที่ 1766: Structured Concurrency (Java 21)

**Structured Concurrency** ช่วยจัดการ concurrent tasks ให้ง่ายและ safe กว่าเดิม:

```java
// src/main/java/com/example/service/ProductAggregationService.java
package com.example.service;

import jdk.incubator.concurrent.StructuredTaskScope;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;

import java.util.concurrent.ExecutionException;

/**
 * ใช้ Structured Concurrency เพื่อ run หลาย tasks พร้อมกัน
 * และ cancel ทั้งหมดถ้าตัวใดตัวหนึ่งล้มเหลว
 */
@Service
@RequiredArgsConstructor
@Slf4j
public class ProductAggregationService {
    
    private final ProductRepository productRepository;
    private final ReviewService reviewService;
    private final InventoryService inventoryService;
    private final RecommendationService recommendationService;
    
    /**
     * ดึงข้อมูลสินค้าทั้งหมดพร้อมกัน (parallel)
     * ถ้าอย่างใดอย่างหนึ่งล้มเหลว ทั้งหมดจะถูก cancel
     */
    public ProductDetailResponse getProductDetail(Long productId) 
            throws InterruptedException, ExecutionException {
        
        try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
            
            // Fork tasks ทั้งหมดพร้อมกัน
            var productTask = scope.fork(() -> 
                productRepository.findById(productId)
                    .orElseThrow(() -> new RuntimeException("Product not found"))
            );
            
            var reviewsTask = scope.fork(() -> 
                reviewService.getReviews(productId)
            );
            
            var inventoryTask = scope.fork(() -> 
                inventoryService.getInventoryStatus(productId)
            );
            
            var recommendationsTask = scope.fork(() ->
                recommendationService.getSimilarProducts(productId)
            );
            
            // รอทุก task เสร็จ (หรือล้มเหลว)
            scope.join()           // รอ
                 .throwIfFailed(); // throw exception ถ้ามี task ล้มเหลว
            
            // ดึงผลลัพธ์ (guaranteed successful at this point)
            return ProductDetailResponse.builder()
                .product(productTask.get())
                .reviews(reviewsTask.get())
                .inventoryStatus(inventoryTask.get())
                .recommendations(recommendationsTask.get())
                .build();
        }
    }
    
    /**
     * Race: ใช้ผลลัพธ์จาก service ที่ตอบกลับเร็วที่สุด
     * เหมาะสำหรับ multi-region / failover scenarios
     */
    public PriceInfo getBestPrice(Long productId) 
            throws InterruptedException, ExecutionException {
        
        try (var scope = new StructuredTaskScope.ShutdownOnSuccess<PriceInfo>()) {
            
            // ถามราคาจาก 3 แหล่ง พร้อมกัน
            scope.fork(() -> pricingService.getPriceFromSupplierA(productId));
            scope.fork(() -> pricingService.getPriceFromSupplierB(productId));
            scope.fork(() -> pricingService.getPriceFromCache(productId));
            
            // รอจนมีตัวแรกสำเร็จ (ShutdownOnSuccess จะ cancel ที่เหลือ)
            scope.join();
            
            return scope.result(); // ราคาจาก supplier ที่เร็วที่สุด
        }
    }
}
```

---

## ขั้นตอนที่ 1767: Virtual Threads กับ Spring @Async

```java
// src/main/java/com/example/service/NotificationService.java
package com.example.service;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.scheduling.annotation.Async;
import org.springframework.stereotype.Service;

import java.util.concurrent.CompletableFuture;

@Service
@RequiredArgsConstructor
@Slf4j
public class NotificationService {
    
    private final EmailService emailService;
    private final SmsService smsService;
    private final PushNotificationService pushService;
    
    /**
     * ส่ง notification แบบ async ด้วย Virtual Thread
     * ไม่บล็อค request thread หลัก
     */
    @Async("virtualThreadExecutor")
    public CompletableFuture<Void> sendOrderConfirmation(Order order) {
        log.info("Sending order confirmation for: {}", order.getOrderNumber());
        
        // ส่งทุก channel พร้อมกัน
        var emailFuture = CompletableFuture.runAsync(() ->
            emailService.sendOrderConfirmation(order), 
            Executors.newVirtualThreadPerTaskExecutor()
        );
        
        var smsFuture = CompletableFuture.runAsync(() ->
            smsService.sendOrderSms(order),
            Executors.newVirtualThreadPerTaskExecutor()
        );
        
        var pushFuture = CompletableFuture.runAsync(() ->
            pushService.sendOrderPush(order),
            Executors.newVirtualThreadPerTaskExecutor()
        );
        
        return CompletableFuture.allOf(emailFuture, smsFuture, pushFuture)
            .whenComplete((result, ex) -> {
                if (ex != null) {
                    log.error("Failed to send some notifications", ex);
                } else {
                    log.info("All notifications sent for order: {}", order.getOrderNumber());
                }
            });
    }
    
    /**
     * Bulk notification ด้วย Virtual Threads
     * ส่ง email ไปยังผู้ใช้ทุกคนพร้อมกัน
     */
    @Async("virtualThreadExecutor")
    public CompletableFuture<BulkNotificationResult> sendBulkPromotion(
            List<Long> userIds, String message) {
        
        var successCount = new java.util.concurrent.atomic.AtomicInteger(0);
        var failCount = new java.util.concurrent.atomic.AtomicInteger(0);
        
        try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
            var futures = userIds.stream()
                .map(userId -> CompletableFuture.runAsync(() -> {
                    try {
                        emailService.sendPromotion(userId, message);
                        successCount.incrementAndGet();
                    } catch (Exception e) {
                        failCount.incrementAndGet();
                        log.warn("Failed to send promotion to user {}: {}", 
                            userId, e.getMessage());
                    }
                }, executor))
                .toList();
            
            CompletableFuture.allOf(futures.toArray(new CompletableFuture[0])).join();
        }
        
        return CompletableFuture.completedFuture(
            new BulkNotificationResult(successCount.get(), failCount.get())
        );
    }
}
```

---

## ขั้นตอนที่ 1768: Pinning - สิ่งที่ต้องระวัง

**Thread Pinning** เกิดเมื่อ Virtual Thread ถูกผูกกับ Carrier Thread ตลอดเวลา ทำให้สูญเสียประโยชน์ของ Virtual Thread:

```java
// ❌ สาเหตุของ Pinning: synchronized block
public class BadExample {
    
    private final Object lock = new Object();
    
    public void criticalSection() {
        synchronized (lock) {
            // Virtual Thread จะถูก pin กับ carrier thread ตลอด synchronized block
            doSomeIoOperation(); // ❌ ทำให้ carrier thread blocked
        }
    }
}

// ✅ แก้ไขด้วย ReentrantLock แทน synchronized
import java.util.concurrent.locks.ReentrantLock;

public class GoodExample {
    
    private final ReentrantLock lock = new ReentrantLock();
    
    public void criticalSection() {
        lock.lock();
        try {
            // Virtual Thread สามารถ unmount ได้ระหว่าง I/O
            doSomeIoOperation(); // ✅ ไม่ pin carrier thread
        } finally {
            lock.unlock();
        }
    }
}
```

### ตรวจหา Pinning ด้วย JVM flags

```bash
# เปิด pinning detection
java -Djdk.tracePinnedThreads=full -jar app.jar

# Output เมื่อเกิด pinning:
# Thread[#xx,ForkJoinPool-1-worker-1,5,CarrierThreads]
#     com.example.service.PaymentService.process(PaymentService.java:45)
#     <== จุดที่เกิด pinning
```

---

## ขั้นตอนที่ 1769: Database Connection Pool กับ Virtual Threads

Virtual Threads ต้องการ connection pool ที่เหมาะสม:

```yaml
# application.yml
spring:
  threads:
    virtual:
      enabled: true
  
  datasource:
    hikari:
      # Virtual Threads: ลด pool size เพราะไม่จำเป็นต้องมี thread เยอะ
      # โดยทั่วไป = จำนวน DB connections สูงสุดที่ DB รองรับ
      maximum-pool-size: 50    # ลดลงจาก 200+
      minimum-idle: 10
      
      # เพิ่ม timeout เพราะ virtual thread อาจรอ connection นานกว่า
      connection-timeout: 30000
      keepalive-time: 30000
      
      # สำคัญ: ปิด connection pool thread management
      # เพราะ Virtual Threads จัดการเองได้
```

```java
// src/main/java/com/example/config/DataSourceConfig.java
package com.example.config;

import com.zaxxer.hikari.HikariConfig;
import com.zaxxer.hikari.HikariDataSource;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class DataSourceConfig {
    
    @Bean
    public HikariDataSource dataSource(HikariConfig config) {
        // HikariCP 5.1.0+ รองรับ Virtual Threads แล้ว
        // ไม่ต้องทำอะไรพิเศษ
        return new HikariDataSource(config);
    }
    
    @Bean
    public HikariConfig hikariConfig() {
        HikariConfig config = new HikariConfig();
        
        // สำหรับ Virtual Threads: pool size = DB max connections / services
        // ถ้า PostgreSQL รองรับ 100 connections และมี 2 instances
        // แต่ละ instance ควรมี pool size ไม่เกิน 50
        config.setMaximumPoolSize(50);
        config.setMinimumIdle(10);
        config.setConnectionTimeout(30_000);
        
        return config;
    }
}
```

---

## ขั้นตอนที่ 1770: Performance Benchmark ด้วย JMH

```java
// src/jmh/java/com/example/benchmark/VirtualThreadBenchmark.java
package com.example.benchmark;

import org.openjdk.jmh.annotations.*;
import org.openjdk.jmh.runner.Runner;
import org.openjdk.jmh.runner.options.OptionsBuilder;

import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicInteger;

@BenchmarkMode(Mode.Throughput)
@OutputTimeUnit(TimeUnit.SECONDS)
@State(Scope.Benchmark)
@Fork(value = 1, warmups = 1)
@Warmup(iterations = 3, time = 5)
@Measurement(iterations = 5, time = 10)
public class VirtualThreadBenchmark {
    
    private static final int CONCURRENT_TASKS = 1000;
    private static final int IO_DELAY_MS = 50;
    
    // Platform Thread Pool
    private ExecutorService platformThreadPool;
    
    // Virtual Thread Executor
    private ExecutorService virtualThreadExecutor;
    
    @Setup
    public void setup() {
        platformThreadPool = Executors.newFixedThreadPool(200);
        virtualThreadExecutor = Executors.newVirtualThreadPerTaskExecutor();
    }
    
    @TearDown
    public void teardown() {
        platformThreadPool.shutdown();
        virtualThreadExecutor.shutdown();
    }
    
    @Benchmark
    public int platformThreads() throws InterruptedException {
        return runTasks(platformThreadPool);
    }
    
    @Benchmark
    public int virtualThreads() throws InterruptedException {
        return runTasks(virtualThreadExecutor);
    }
    
    private int runTasks(ExecutorService executor) throws InterruptedException {
        CountDownLatch latch = new CountDownLatch(CONCURRENT_TASKS);
        AtomicInteger completed = new AtomicInteger(0);
        
        for (int i = 0; i < CONCURRENT_TASKS; i++) {
            executor.submit(() -> {
                try {
                    Thread.sleep(IO_DELAY_MS); // simulate I/O
                    completed.incrementAndGet();
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                } finally {
                    latch.countDown();
                }
            });
        }
        
        latch.await(30, TimeUnit.SECONDS);
        return completed.get();
    }
    
    public static void main(String[] args) throws Exception {
        var options = new OptionsBuilder()
            .include(VirtualThreadBenchmark.class.getSimpleName())
            .build();
        new Runner(options).run();
    }
}
```

---

## ขั้นตอนที่ 1771: Real-World Example - HTTP Client

```java
// src/main/java/com/example/service/AggregatorService.java
package com.example.service;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;
import org.springframework.web.client.RestClient;

import java.util.List;
import java.util.concurrent.Executors;
import java.util.concurrent.StructuredTaskScope;

/**
 * Aggregator service ที่ call หลาย external APIs พร้อมกัน
 * ด้วย Virtual Threads + Structured Concurrency
 */
@Service
@RequiredArgsConstructor
@Slf4j
public class AggregatorService {
    
    private final RestClient restClient;
    
    /**
     * ดึงข้อมูลจาก 3 services พร้อมกัน
     * รวม response time ≈ max(service1, service2, service3) ไม่ใช่ sum
     */
    public DashboardData getDashboard(String userId) 
            throws InterruptedException, Exception {
        
        try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
            
            // Fork all 3 calls พร้อมกัน
            var userTask = scope.fork(() -> 
                fetchUser(userId)
            );
            
            var ordersTask = scope.fork(() ->
                fetchRecentOrders(userId)
            );
            
            var recommendationsTask = scope.fork(() ->
                fetchRecommendations(userId)
            );
            
            // รอทั้งหมด
            scope.join().throwIfFailed();
            
            // สร้าง response
            return DashboardData.builder()
                .user(userTask.get())
                .recentOrders(ordersTask.get())
                .recommendations(recommendationsTask.get())
                .build();
        }
    }
    
    private UserInfo fetchUser(String userId) {
        log.debug("Fetching user {} on thread: {}", userId, Thread.currentThread());
        return restClient.get()
            .uri("https://user-service/users/{id}", userId)
            .retrieve()
            .body(UserInfo.class);
    }
    
    private List<Order> fetchRecentOrders(String userId) {
        log.debug("Fetching orders for {} on thread: {}", userId, Thread.currentThread());
        return restClient.get()
            .uri("https://order-service/orders?userId={id}&limit=5", userId)
            .retrieve()
            .body(new ParameterizedTypeReference<>() {});
    }
    
    private List<Product> fetchRecommendations(String userId) {
        log.debug("Fetching recommendations for {} on thread: {}", userId, Thread.currentThread());
        return restClient.get()
            .uri("https://recommendation-service/recommend/{id}", userId)
            .retrieve()
            .body(new ParameterizedTypeReference<>() {});
    }
}
```

---

## ขั้นตอนที่ 1772: Monitoring Virtual Threads

```java
// src/main/java/com/example/monitoring/VirtualThreadMetrics.java
package com.example.monitoring;

import io.micrometer.core.instrument.MeterRegistry;
import io.micrometer.core.instrument.binder.MeterBinder;
import lombok.extern.slf4j.Slf4j;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Component;

import java.lang.management.ManagementFactory;
import java.lang.management.ThreadMXBean;

@Component
@Slf4j
public class VirtualThreadMetrics implements MeterBinder {
    
    private final ThreadMXBean threadMXBean = ManagementFactory.getThreadMXBean();
    private MeterRegistry registry;
    
    @Override
    public void bindTo(MeterRegistry registry) {
        this.registry = registry;
    }
    
    @Scheduled(fixedRate = 30_000)
    public void reportThreadMetrics() {
        long platformThreadCount = Thread.getAllStackTraces().keySet().stream()
            .filter(t -> !t.isVirtual())
            .count();
        
        long virtualThreadCount = Thread.getAllStackTraces().keySet().stream()
            .filter(Thread::isVirtual)
            .count();
        
        log.info("Thread stats - Platform: {}, Virtual: {}", 
            platformThreadCount, virtualThreadCount);
        
        if (registry != null) {
            registry.gauge("threads.platform.count", platformThreadCount);
            registry.gauge("threads.virtual.count", virtualThreadCount);
        }
    }
    
    // ตรวจสอบว่า Virtual Thread support ถูกเปิดไว้
    public static boolean isVirtualThreadEnabled() {
        try {
            Thread.ofVirtual().name("test").start(() -> {}).join();
            return true;
        } catch (Exception e) {
            return false;
        }
    }
}
```

---

## ขั้นตอนที่ 1773: Testing Virtual Threads

```java
// src/test/java/com/example/virtualthread/VirtualThreadIntegrationTest.java
package com.example.virtualthread;

import org.junit.jupiter.api.Test;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.web.client.TestRestTemplate;
import org.springframework.boot.test.web.server.LocalServerPort;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.ResponseEntity;

import java.time.Duration;
import java.time.Instant;
import java.util.List;
import java.util.concurrent.CompletableFuture;
import java.util.stream.IntStream;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class VirtualThreadIntegrationTest {
    
    @LocalServerPort
    private int port;
    
    @Autowired
    private TestRestTemplate restTemplate;
    
    @Test
    void shouldHandle1000ConcurrentRequests() {
        int concurrentRequests = 1000;
        
        Instant start = Instant.now();
        
        // ส่ง 1000 requests พร้อมกัน
        List<CompletableFuture<ResponseEntity<String>>> futures = 
            IntStream.range(0, concurrentRequests)
                .mapToObj(i -> CompletableFuture.supplyAsync(() ->
                    restTemplate.getForEntity(
                        "http://localhost:" + port + "/api/v1/products",
                        String.class
                    )
                ))
                .toList();
        
        // รอทั้งหมดเสร็จ
        var results = futures.stream()
            .map(CompletableFuture::join)
            .toList();
        
        Duration elapsed = Duration.between(start, Instant.now());
        
        // ตรวจสอบผลลัพธ์
        long successCount = results.stream()
            .filter(r -> r.getStatusCode().is2xxSuccessful())
            .count();
        
        System.out.printf("Processed %d/%d requests in %d ms%n",
            successCount, concurrentRequests, elapsed.toMillis());
        
        assertThat(successCount).isEqualTo(concurrentRequests);
        assertThat(elapsed.toSeconds()).isLessThan(10); // ควรเสร็จใน 10 วินาที
    }
    
    @Test
    void shouldUseVirtualThreadsForRequests() throws Exception {
        // ตรวจสอบว่า request thread เป็น virtual thread
        CompletableFuture<Boolean> isVirtualThread = new CompletableFuture<>();
        
        // Endpoint ที่ return thread info
        ResponseEntity<String> response = restTemplate.getForEntity(
            "http://localhost:" + port + "/api/v1/debug/thread-info",
            String.class
        );
        
        assertThat(response.getBody()).contains("isVirtual=true");
    }
}
```

---

## ขั้นตอนที่ 1774: Debug Endpoint สำหรับ Thread Info

```java
// src/main/java/com/example/controller/DebugController.java
package com.example.controller;

import org.springframework.context.annotation.Profile;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import java.util.Map;

@RestController
@RequestMapping("/v1/debug")
@Profile("!prod")  // ปิดใน production
public class DebugController {
    
    @GetMapping("/thread-info")
    public Map<String, Object> getThreadInfo() {
        Thread current = Thread.currentThread();
        return Map.of(
            "threadName", current.getName(),
            "threadId", current.threadId(),
            "isVirtual", current.isVirtual(),
            "isDaemon", current.isDaemon(),
            "priority", current.getPriority(),
            "state", current.getState().name()
        );
    }
    
    @GetMapping("/all-threads")
    public Map<String, Long> getAllThreadStats() {
        var allThreads = Thread.getAllStackTraces().keySet();
        long virtualCount = allThreads.stream().filter(Thread::isVirtual).count();
        long platformCount = allThreads.size() - virtualCount;
        
        return Map.of(
            "total", (long) allThreads.size(),
            "virtual", virtualCount,
            "platform", platformCount
        );
    }
}
```

---

## สรุปท้ายบท

ในส่วนนี้เราได้เรียนรู้:

1. **Virtual Threads** คืออะไรและแก้ปัญหา scalability อย่างไร
2. **เปิดใช้งาน** ใน Spring Boot 3.2 ด้วย property หรือ Java config
3. **เปรียบเทียบ** Platform vs Virtual Threads ด้วย code จริง
4. **เมื่อไรควรใช้** (I/O-bound) และเมื่อไรไม่ควรใช้ (CPU-bound)
5. **Thread Local vs Scoped Values** สำหรับ context propagation
6. **Structured Concurrency** สำหรับ parallel tasks ที่ปลอดภัย
7. **Thread Pinning** และวิธีหลีกเลี่ยง
8. **Performance Benchmark** วัดผลจริง

Virtual Threads เป็นการเปลี่ยนแปลงที่ยิ่งใหญ่ใน Java ecosystem โดยไม่ต้องเขียน reactive code ที่ยุ่งยาก แค่เปิดใช้งานก็ได้ประโยชน์ทันที

---

*[← Part 53: GraalVM Native](./part-53-graalvm-native.md) | [Part 55: API Versioning →](./part-55-api-versioning.md)*

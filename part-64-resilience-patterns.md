# Part 64: Resilience Patterns
## ขั้นตอนที่ 2161-2200

**ระดับ:** ระดับโลก (World-Class)
**เวลาเรียน:** 4-5 ชั่วโมง
**เป้าหมาย:** Master Resilience4j สำหรับ Production-Grade Fault Tolerance

---

## บทนำ

ในระบบ Distributed System ความล้มเหลวเป็นสิ่งที่หลีกเลี่ยงไม่ได้ Resilience Patterns ช่วยให้ระบบของเรา "ล้มแบบสง่างาม" (Fail Gracefully) แทนที่จะ Cascade Failure ไปทั้งระบบ

Resilience4j มี 6 Core Modules:
1. **Circuit Breaker** - ป้องกัน Call Service ที่ล้มเหลวซ้ำๆ
2. **Retry** - ลอง Request ใหม่เมื่อล้มเหลว
3. **Rate Limiter** - จำกัด Request Rate
4. **Bulkhead** - แยก Thread Pool
5. **Time Limiter** - กำหนด Timeout
6. **Cache** - Cache ผลลัพธ์

---

## ขั้นตอนที่ 2161: Setup

```xml
<!-- pom.xml -->
<dependencies>
    <dependency>
        <groupId>io.github.resilience4j</groupId>
        <artifactId>resilience4j-spring-boot3</artifactId>
    </dependency>
    <dependency>
        <groupId>io.github.resilience4j</groupId>
        <artifactId>resilience4j-reactor</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-aop</artifactId>
    </dependency>
    <dependency>
        <groupId>io.micrometer</groupId>
        <artifactId>micrometer-registry-prometheus</artifactId>
    </dependency>
</dependencies>
```

```yaml
# application.yml - Resilience4j Configuration
resilience4j:
  circuit-breaker:
    instances:
      paymentService:
        slidingWindowType: COUNT_BASED
        slidingWindowSize: 10
        minimumNumberOfCalls: 5
        failureRateThreshold: 50
        waitDurationInOpenState: 30s
        permittedNumberOfCallsInHalfOpenState: 3
        automaticTransitionFromOpenToHalfOpenEnabled: true
        slowCallRateThreshold: 100
        slowCallDurationThreshold: 5s
        recordExceptions:
          - java.io.IOException
          - java.net.SocketTimeoutException
          - com.example.exception.ServiceUnavailableException
        ignoreExceptions:
          - com.example.exception.BusinessValidationException
  
  retry:
    instances:
      paymentService:
        maxAttempts: 3
        waitDuration: 500ms
        enableExponentialBackoff: true
        exponentialBackoffMultiplier: 2
        exponentialMaxWaitDuration: 10s
        enableRandomizedWait: true
        randomizedWaitFactor: 0.5
        retryExceptions:
          - java.io.IOException
          - java.net.SocketTimeoutException
  
  rate-limiter:
    instances:
      paymentService:
        limitForPeriod: 100
        limitRefreshPeriod: 1s
        timeoutDuration: 500ms
  
  bulkhead:
    instances:
      paymentService:
        maxConcurrentCalls: 20
        maxWaitDuration: 100ms
  
  time-limiter:
    instances:
      paymentService:
        timeoutDuration: 3s
        cancelRunningFuture: true
```

---

## ขั้นตอนที่ 2162-2165: Circuit Breaker States

### อธิบาย States

```
CLOSED (ปกติ) -> เมื่อ failure rate > threshold -> OPEN (บล็อก calls)
OPEN -> เมื่อครบ waitDuration -> HALF-OPEN (ทดสอบ)
HALF-OPEN -> เมื่อ success -> CLOSED
HALF-OPEN -> เมื่อ fail -> OPEN
```

```java
// CircuitBreakerService.java - ใช้งาน Circuit Breaker
@Service
@RequiredArgsConstructor
@Slf4j
public class PaymentService {
    
    private final PaymentClient paymentClient;
    private final CircuitBreakerRegistry circuitBreakerRegistry;
    
    @CircuitBreaker(name = "paymentService", fallbackMethod = "paymentFallback")
    public PaymentResult processPayment(PaymentRequest request) {
        log.info("Processing payment: {}", request.getOrderId());
        return paymentClient.charge(request);
    }
    
    public PaymentResult paymentFallback(PaymentRequest request, 
                                          CallNotPermittedException e) {
        log.warn("Circuit OPEN for payment: {} - {}", 
                 request.getOrderId(), e.getMessage());
        // ส่ง Request ไปที่ Queue เพื่อ Process ในภายหลัง
        return PaymentResult.queued(request.getOrderId());
    }
    
    public PaymentResult paymentFallback(PaymentRequest request, 
                                          Exception e) {
        log.error("Payment failed: {} - {}", request.getOrderId(), e.getMessage());
        return PaymentResult.failed(request.getOrderId(), e.getMessage());
    }
    
    // Monitor Circuit Breaker State
    @EventListener
    public void onCircuitBreakerStateChange(CircuitBreakerOnStateTransitionEvent event) {
        log.info("Circuit Breaker '{}' changed: {} -> {}",
                 event.getCircuitBreakerName(),
                 event.getStateTransition().getFromState(),
                 event.getStateTransition().getToState());
        
        if (event.getStateTransition().getToState() == CircuitBreaker.State.OPEN) {
            // Alert ทีม
            alertService.sendAlert(
                "Circuit Breaker OPEN: " + event.getCircuitBreakerName());
        }
    }
}
```

```java
// CircuitBreakerMonitor.java - Monitor Circuit Breaker แบบ Programmatic
@Component
@RequiredArgsConstructor
@Slf4j
public class CircuitBreakerMonitor {
    
    private final CircuitBreakerRegistry registry;
    private final MeterRegistry meterRegistry;
    
    @PostConstruct
    public void init() {
        // Register Metrics สำหรับทุก Circuit Breaker
        registry.getAllCircuitBreakers().forEach(cb -> {
            TaggedCircuitBreakerMetrics.ofCircuitBreakerRegistry(registry)
                .bindTo(meterRegistry);
            
            // Listen State Changes
            cb.getEventPublisher()
                .onStateTransition(event -> {
                    log.info("CB State Change: {} {} -> {}",
                             event.getCircuitBreakerName(),
                             event.getStateTransition().getFromState(),
                             event.getStateTransition().getToState());
                });
            
            // Listen Errors
            cb.getEventPublisher()
                .onError(event -> {
                    log.warn("CB Error: {} - {}",
                             event.getCircuitBreakerName(),
                             event.getThrowable().getMessage());
                });
        });
    }
    
    // API ดูสถานะ Circuit Breakers ทั้งหมด
    public Map<String, CircuitBreakerStatus> getAllStatuses() {
        Map<String, CircuitBreakerStatus> result = new HashMap<>();
        
        registry.getAllCircuitBreakers().forEach(cb -> {
            CircuitBreaker.Metrics metrics = cb.getMetrics();
            result.put(cb.getName(), CircuitBreakerStatus.builder()
                .state(cb.getState().toString())
                .failureRate(metrics.getFailureRate())
                .slowCallRate(metrics.getSlowCallRate())
                .bufferedCalls(metrics.getNumberOfBufferedCalls())
                .failedCalls(metrics.getNumberOfFailedCalls())
                .successfulCalls(metrics.getNumberOfSuccessfulCalls())
                .notPermittedCalls(metrics.getNumberOfNotPermittedCalls())
                .build());
        });
        
        return result;
    }
    
    // Force Open Circuit (สำหรับ Emergency)
    public void forceOpen(String circuitBreakerName) {
        CircuitBreaker cb = registry.circuitBreaker(circuitBreakerName);
        cb.transitionToForcedOpenState();
        log.warn("Circuit Breaker '{}' FORCE OPENED", circuitBreakerName);
    }
    
    // Force Close Circuit
    public void forceClose(String circuitBreakerName) {
        CircuitBreaker cb = registry.circuitBreaker(circuitBreakerName);
        cb.transitionToClosedState();
        log.info("Circuit Breaker '{}' FORCE CLOSED", circuitBreakerName);
    }
}
```

---

## ขั้นตอนที่ 2166-2170: Retry ด้วย Exponential Backoff และ Jitter

### อธิบาย

- **Simple Retry:** ลองใหม่ N ครั้ง ทุก X วินาที (ปัญหา: Thundering Herd)
- **Exponential Backoff:** รอนานขึ้นเรื่อยๆ (100ms, 200ms, 400ms, ...)
- **Jitter:** เพิ่ม Random ใน Wait Time เพื่อป้องกัน Thundering Herd

```java
// RetryService.java - ใช้งาน Retry
@Service
@RequiredArgsConstructor
@Slf4j
public class InventoryService {
    
    private final InventoryClient inventoryClient;
    
    @Retry(name = "inventoryService", fallbackMethod = "checkFallback")
    public InventoryStatus checkInventory(String productId) {
        log.info("Checking inventory for product: {}", productId);
        return inventoryClient.check(productId);
    }
    
    public InventoryStatus checkFallback(String productId, Exception e) {
        log.warn("Inventory check failed after retries: {} - {}", 
                 productId, e.getMessage());
        // Return cached/default value
        return InventoryStatus.unknown(productId);
    }
    
    // Retry พร้อม Event Listening
    @PostConstruct
    public void setupRetryEvents() {
        RetryRegistry registry = RetryRegistry.ofDefaults();
        Retry retry = registry.retry("inventoryService");
        
        retry.getEventPublisher()
            .onRetry(event -> log.warn("Retry attempt {} for: {}",
                event.getNumberOfRetryAttempts(),
                event.getLastThrowable().getMessage()))
            .onSuccess(event -> log.info("Retry succeeded after {} attempts",
                event.getNumberOfRetryAttempts()))
            .onError(event -> log.error("All retries exhausted: {}",
                event.getLastThrowable().getMessage()));
    }
}
```

```java
// CustomRetryConfig.java - Custom Retry Configuration
@Configuration
public class CustomRetryConfig {
    
    @Bean
    public RetryRegistry retryRegistry() {
        // Config 1: สำหรับ Network Errors (Exponential Backoff + Jitter)
        io.github.resilience4j.retry.RetryConfig networkRetryConfig = 
            io.github.resilience4j.retry.RetryConfig.custom()
                .maxAttempts(3)
                .intervalFunction(
                    IntervalFunction.ofExponentialRandomBackoff(
                        100,    // Initial interval: 100ms
                        2.0,    // Multiplier: x2 ทุกครั้ง
                        0.5,    // Jitter: ±50%
                        10000   // Max interval: 10s
                    )
                )
                .retryOnException(e -> e instanceof IOException ||
                                       e instanceof SocketTimeoutException)
                .build();
        
        // Config 2: สำหรับ Business Logic Retries (Fixed Delay)
        io.github.resilience4j.retry.RetryConfig businessRetryConfig = 
            io.github.resilience4j.retry.RetryConfig.custom()
                .maxAttempts(5)
                .waitDuration(Duration.ofMillis(200))
                .retryOnResult(result -> {
                    if (result instanceof ApiResponse) {
                        return ((ApiResponse<?>) result).isRetriable();
                    }
                    return false;
                })
                .build();
        
        return RetryRegistry.of(Map.of(
            "networkService", networkRetryConfig,
            "businessService", businessRetryConfig
        ));
    }
}
```

```java
// RetryWithContext.java - Retry ที่ส่ง Context ระหว่าง Attempts
@Service
@Slf4j
public class SmartRetryService {
    
    public <T> T executeWithRetry(Supplier<T> operation, 
                                    RetryContext context) {
        int attempt = 0;
        Exception lastException = null;
        
        while (attempt < context.getMaxAttempts()) {
            try {
                T result = operation.get();
                
                if (attempt > 0) {
                    log.info("Operation succeeded after {} retries", attempt);
                }
                
                return result;
                
            } catch (Exception e) {
                lastException = e;
                attempt++;
                
                if (attempt >= context.getMaxAttempts()) {
                    log.error("Operation failed after {} attempts", attempt);
                    break;
                }
                
                // ตรวจสอบว่า Exception นี้ควร Retry หรือไม่
                if (!context.shouldRetry(e)) {
                    log.info("Non-retriable exception, stopping retry: {}", 
                             e.getClass().getSimpleName());
                    break;
                }
                
                long waitMs = calculateWaitTime(attempt, context);
                log.warn("Attempt {} failed: {}. Retrying in {}ms...",
                         attempt, e.getMessage(), waitMs);
                
                try {
                    Thread.sleep(waitMs);
                } catch (InterruptedException ie) {
                    Thread.currentThread().interrupt();
                    throw new RuntimeException("Retry interrupted", ie);
                }
            }
        }
        
        throw new RetryExhaustedException(
            "Failed after " + attempt + " attempts", lastException);
    }
    
    private long calculateWaitTime(int attempt, RetryContext context) {
        // Exponential Backoff with Jitter
        long baseWait = (long) (context.getInitialWaitMs() * 
                                 Math.pow(context.getMultiplier(), attempt - 1));
        long maxWait = Math.min(baseWait, context.getMaxWaitMs());
        
        // Add Jitter: ±jitterPercent%
        double jitter = (Math.random() * 2 - 1) * context.getJitterPercent() * maxWait;
        return Math.max(0, (long) (maxWait + jitter));
    }
    
    @Data
    @Builder
    public static class RetryContext {
        @Builder.Default
        private int maxAttempts = 3;
        @Builder.Default
        private long initialWaitMs = 100;
        @Builder.Default
        private double multiplier = 2.0;
        @Builder.Default
        private long maxWaitMs = 10000;
        @Builder.Default
        private double jitterPercent = 0.5;
        @Builder.Default
        private Predicate<Exception> shouldRetry = e -> true;
    }
}
```

---

## ขั้นตอนที่ 2171-2175: Rate Limiter

### SemaphoreBased vs AtomicBased

```
SemaphoreBased:
- ใช้ Java Semaphore ควบคุมจำนวน Concurrent Calls
- ดีสำหรับ: Traditional Thread-based Applications
- ไม่รองรับ: Distributed Rate Limiting ข้าม Instances

AtomicBased:
- ใช้ AtomicInteger เร็วกว่า SemaphoreBased เล็กน้อย
- ดีสำหรับ: High-performance single-instance
```

```java
// RateLimiterService.java
@Service
@RequiredArgsConstructor
@Slf4j
public class ApiService {
    
    private final ExternalApiClient apiClient;
    
    // Rate Limit: 100 calls/second สำหรับ External API
    @RateLimiter(name = "externalApi", fallbackMethod = "rateLimitFallback")
    public ApiResponse callExternalApi(String requestData) {
        return apiClient.call(requestData);
    }
    
    public ApiResponse rateLimitFallback(String requestData, 
                                          RequestNotPermitted e) {
        log.warn("Rate limit exceeded for external API: {}", e.getMessage());
        // Return cached response หรือ Queue ไว้ Process ทีหลัง
        return ApiResponse.rateLimited();
    }
    
    // Dynamic Rate Limiting ตาม User Tier
    public ApiResponse callWithDynamicRateLimit(String requestData, 
                                                  String userTier) {
        RateLimiterConfig config = getRateLimitConfig(userTier);
        RateLimiter rateLimiter = RateLimiter.of("user-" + userTier, config);
        
        return rateLimiter.executeSupplier(() -> apiClient.call(requestData));
    }
    
    private RateLimiterConfig getRateLimitConfig(String userTier) {
        return switch (userTier) {
            case "premium" -> RateLimiterConfig.custom()
                .limitForPeriod(1000)
                .limitRefreshPeriod(Duration.ofSeconds(1))
                .timeoutDuration(Duration.ofMillis(100))
                .build();
            case "standard" -> RateLimiterConfig.custom()
                .limitForPeriod(100)
                .limitRefreshPeriod(Duration.ofSeconds(1))
                .timeoutDuration(Duration.ofMillis(500))
                .build();
            default -> RateLimiterConfig.custom()
                .limitForPeriod(10)
                .limitRefreshPeriod(Duration.ofSeconds(1))
                .timeoutDuration(Duration.ofMillis(100))
                .build();
        };
    }
}
```

---

## ขั้นตอนที่ 2176-2180: TimeLimiter

```java
// TimeLimiterService.java
@Service
@RequiredArgsConstructor
@Slf4j
public class SlowOperationService {
    
    private final SlowDataSource slowDataSource;
    
    // ใช้ @TimeLimiter กับ CompletableFuture
    @TimeLimiter(name = "slowOperation", fallbackMethod = "timeoutFallback")
    public CompletableFuture<DataResult> performSlowOperation(String query) {
        return CompletableFuture.supplyAsync(() -> {
            log.info("Starting slow operation: {}", query);
            return slowDataSource.executeQuery(query); // อาจใช้เวลานาน
        });
    }
    
    public CompletableFuture<DataResult> timeoutFallback(
            String query, TimeoutException e) {
        log.warn("Operation timed out for query: {}", query);
        // Return cached/partial result
        return CompletableFuture.completedFuture(DataResult.partial());
    }
    
    // TimeLimiter ด้วย Programmatic API
    public DataResult executeWithTimeout(String query, Duration timeout) {
        TimeLimiter timeLimiter = TimeLimiter.of(
            TimeLimiterConfig.custom()
                .timeoutDuration(timeout)
                .cancelRunningFuture(true)
                .build()
        );
        
        try {
            return timeLimiter.executeFutureSupplier(
                () -> CompletableFuture.supplyAsync(
                    () -> slowDataSource.executeQuery(query)
                )
            );
        } catch (TimeoutException e) {
            log.warn("Query timed out after {}: {}", timeout, query);
            return DataResult.partial();
        } catch (Exception e) {
            throw new RuntimeException("Query failed", e);
        }
    }
}
```

---

## ขั้นตอนที่ 2181-2185: Combining Patterns

### Order Service ที่ใช้ทุก Pattern รวมกัน

```java
// ResilientOrderService.java - ใช้ทุก Pattern รวมกัน
@Service
@RequiredArgsConstructor
@Slf4j
public class ResilientOrderService {
    
    private final PaymentClient paymentClient;
    private final InventoryClient inventoryClient;
    
    // Retry -> Circuit Breaker -> Rate Limiter -> Bulkhead -> Time Limiter
    // ลำดับนี้สำคัญ: Retry อยู่นอกสุด Circuit Breaker อยู่ด้านใน
    @Retry(name = "paymentService")
    @CircuitBreaker(name = "paymentService")
    @RateLimiter(name = "paymentService")
    @Bulkhead(name = "paymentService")
    @TimeLimiter(name = "paymentService")
    public CompletableFuture<PaymentResult> processPaymentResilient(
            PaymentRequest request) {
        
        return CompletableFuture.supplyAsync(() -> {
            log.info("Processing payment: {}", request.getOrderId());
            return paymentClient.charge(request);
        });
    }
    
    // ทำแบบ Programmatic (ชัดเจนกว่า)
    public PaymentResult processPaymentProgrammatic(PaymentRequest request) {
        CircuitBreaker cb = circuitBreakerRegistry.circuitBreaker("paymentService");
        RateLimiter rl = rateLimiterRegistry.rateLimiter("paymentService");
        Bulkhead bh = bulkheadRegistry.bulkhead("paymentService");
        Retry retry = retryRegistry.retry("paymentService");
        
        // Chain Pattern: Bulkhead -> Rate Limiter -> Circuit Breaker -> Retry
        Supplier<PaymentResult> supplier = () -> paymentClient.charge(request);
        supplier = CircuitBreaker.decorateSupplier(cb, supplier);
        supplier = RateLimiter.decorateSupplier(rl, supplier);
        supplier = Bulkhead.decorateSupplier(bh, supplier);
        supplier = Retry.decorateSupplier(retry, supplier);
        
        try {
            return supplier.get();
        } catch (CallNotPermittedException e) {
            log.warn("Circuit OPEN, using fallback");
            return PaymentResult.queued(request.getOrderId());
        } catch (RequestNotPermitted e) {
            log.warn("Rate limit exceeded, using fallback");
            return PaymentResult.rateLimited(request.getOrderId());
        } catch (BulkheadFullException e) {
            log.warn("Bulkhead full, using fallback");
            return PaymentResult.rejected(request.getOrderId());
        }
    }
}
```

---

## ขั้นตอนที่ 2186-2190: Resilience Testing

### Fault Injection Testing

```java
// FaultInjectionService.java - Inject Failures สำหรับ Testing
@Service
@Profile("fault-injection")
@Slf4j
public class FaultInjectionService {
    
    private final Map<String, FaultConfig> faultConfigs = new ConcurrentHashMap<>();
    
    @Data
    @Builder
    public static class FaultConfig {
        private double failureProbability;
        private Duration latency;
        private Class<? extends Exception> exceptionType;
        private String exceptionMessage;
        private boolean enabled;
    }
    
    public void configureFault(String serviceName, FaultConfig config) {
        faultConfigs.put(serviceName, config);
        log.warn("Fault injection configured for {}: {}", serviceName, config);
    }
    
    public void clearFault(String serviceName) {
        faultConfigs.remove(serviceName);
        log.info("Fault injection cleared for {}", serviceName);
    }
    
    public void maybeInjectFault(String serviceName) {
        FaultConfig config = faultConfigs.get(serviceName);
        if (config == null || !config.isEnabled()) return;
        
        // Inject Latency
        if (config.getLatency() != null) {
            try {
                log.debug("Injecting {}ms latency for {}", 
                         config.getLatency().toMillis(), serviceName);
                Thread.sleep(config.getLatency().toMillis());
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }
        
        // Inject Failure
        if (Math.random() < config.getFailureProbability()) {
            log.warn("Injecting failure for {}", serviceName);
            try {
                throw config.getExceptionType()
                    .getConstructor(String.class)
                    .newInstance(config.getExceptionMessage());
            } catch (ReflectiveOperationException e) {
                throw new RuntimeException("Failed to inject fault", e);
            }
        }
    }
}
```

```java
// ResilienceIntegrationTest.java - Test ทุก Pattern
@SpringBootTest
@ActiveProfiles({"test", "fault-injection"})
class ResilienceIntegrationTest {
    
    @Autowired
    private ResilientOrderService orderService;
    
    @Autowired
    private FaultInjectionService faultInjection;
    
    @Autowired
    private CircuitBreakerRegistry circuitBreakerRegistry;
    
    @Test
    void circuitBreakerOpensAfterConsecutiveFailures() {
        // Inject 100% failure rate
        faultInjection.configureFault("paymentService", 
            FaultInjectionService.FaultConfig.builder()
                .failureProbability(1.0)
                .exceptionType(ServiceUnavailableException.class)
                .exceptionMessage("Service Down")
                .enabled(true)
                .build());
        
        CircuitBreaker cb = circuitBreakerRegistry.circuitBreaker("paymentService");
        
        // สร้าง 10 Failed Calls เพื่อเปิด Circuit
        for (int i = 0; i < 10; i++) {
            try {
                orderService.processPaymentProgrammatic(createTestRequest());
            } catch (Exception e) {
                // Expected
            }
        }
        
        // Circuit ควรจะ Open แล้ว
        assertThat(cb.getState()).isEqualTo(CircuitBreaker.State.OPEN);
        
        // Call ต่อไปควร Fail Fast ด้วย CallNotPermittedException
        assertThatThrownBy(() -> 
            orderService.processPaymentProgrammatic(createTestRequest()))
            .isInstanceOf(CallNotPermittedException.class);
        
        // Cleanup
        faultInjection.clearFault("paymentService");
    }
    
    @Test
    void retryWithExponentialBackoff() throws Exception {
        AtomicInteger callCount = new AtomicInteger(0);
        List<Long> callTimes = Collections.synchronizedList(new ArrayList<>());
        
        faultInjection.configureFault("inventoryService",
            FaultInjectionService.FaultConfig.builder()
                .failureProbability(0.7) // 70% failure
                .exceptionType(IOException.class)
                .exceptionMessage("Network Error")
                .enabled(true)
                .build());
        
        // Track timing
        long start = System.currentTimeMillis();
        
        try {
            orderService.checkInventory("PROD-001");
        } catch (Exception e) {
            // May fail after retries
        }
        
        long elapsed = System.currentTimeMillis() - start;
        
        // ควรใช้เวลา > 100ms เพราะมี Backoff
        assertThat(elapsed).isGreaterThan(100L);
        
        faultInjection.clearFault("inventoryService");
    }
    
    @Test
    void rateLimiterPreventsTooManyCalls() throws InterruptedException {
        // ส่ง 200 Requests ใน 1 วินาที (Limit คือ 100/sec)
        int successCount = 0;
        int rateLimitedCount = 0;
        
        for (int i = 0; i < 200; i++) {
            try {
                orderService.processPaymentProgrammatic(createTestRequest());
                successCount++;
            } catch (RequestNotPermitted e) {
                rateLimitedCount++;
            }
        }
        
        log.info("Success: {}, RateLimited: {}", successCount, rateLimitedCount);
        
        // ควรมี ~100 Success และ ~100 RateLimited
        assertThat(successCount).isLessThanOrEqualTo(100);
        assertThat(rateLimitedCount).isGreaterThan(0);
    }
    
    @Test
    void bulkheadLimitsConcurrency() throws InterruptedException {
        int threadCount = 50;
        CountDownLatch startLatch = new CountDownLatch(1);
        CountDownLatch doneLatch = new CountDownLatch(threadCount);
        AtomicInteger bulkheadRejectCount = new AtomicInteger(0);
        
        // Inject Latency เพื่อให้ Concurrent Calls เกิดขึ้น
        faultInjection.configureFault("paymentService",
            FaultInjectionService.FaultConfig.builder()
                .latency(Duration.ofMillis(500))
                .failureProbability(0.0)
                .enabled(true)
                .build());
        
        for (int i = 0; i < threadCount; i++) {
            new Thread(() -> {
                try {
                    startLatch.await();
                    orderService.processPaymentProgrammatic(createTestRequest());
                } catch (BulkheadFullException e) {
                    bulkheadRejectCount.incrementAndGet();
                } catch (Exception e) {
                    // Other exceptions
                } finally {
                    doneLatch.countDown();
                }
            }).start();
        }
        
        startLatch.countDown(); // เริ่มพร้อมกันทุก Thread
        doneLatch.await(10, TimeUnit.SECONDS);
        
        // ควรมี Requests ถูก Reject เพราะ Bulkhead เต็ม
        assertThat(bulkheadRejectCount.get()).isGreaterThan(0);
        log.info("Bulkhead rejected {} out of {} requests", 
                bulkheadRejectCount.get(), threadCount);
        
        faultInjection.clearFault("paymentService");
    }
}
```

---

## ขั้นตอนที่ 2191-2200: Resilience Metrics Dashboard

```java
// ResilienceMetricsController.java
@RestController
@RequestMapping("/admin/resilience")
@RequiredArgsConstructor
public class ResilienceMetricsController {
    
    private final CircuitBreakerRegistry cbRegistry;
    private final RetryRegistry retryRegistry;
    private final RateLimiterRegistry rlRegistry;
    private final BulkheadRegistry bhRegistry;
    
    @GetMapping("/circuit-breakers")
    public List<Map<String, Object>> getCircuitBreakerStats() {
        return cbRegistry.getAllCircuitBreakers().stream()
            .map(cb -> {
                CircuitBreaker.Metrics m = cb.getMetrics();
                return Map.<String, Object>of(
                    "name", cb.getName(),
                    "state", cb.getState().toString(),
                    "failureRate", m.getFailureRate(),
                    "slowCallRate", m.getSlowCallRate(),
                    "totalCalls", m.getNumberOfBufferedCalls(),
                    "failedCalls", m.getNumberOfFailedCalls(),
                    "successfulCalls", m.getNumberOfSuccessfulCalls(),
                    "notPermittedCalls", m.getNumberOfNotPermittedCalls()
                );
            })
            .collect(Collectors.toList());
    }
    
    @GetMapping("/rate-limiters")
    public List<Map<String, Object>> getRateLimiterStats() {
        return rlRegistry.getAllRateLimiters().stream()
            .map(rl -> {
                RateLimiter.Metrics m = rl.getMetrics();
                return Map.<String, Object>of(
                    "name", rl.getName(),
                    "availablePermissions", m.getAvailablePermissions(),
                    "waitingThreads", m.getNumberOfWaitingThreads()
                );
            })
            .collect(Collectors.toList());
    }
    
    @PostMapping("/circuit-breakers/{name}/force-open")
    @PreAuthorize("hasRole('ADMIN')")
    public ResponseEntity<String> forceOpen(@PathVariable String name) {
        try {
            CircuitBreaker cb = cbRegistry.circuitBreaker(name);
            cb.transitionToForcedOpenState();
            return ResponseEntity.ok("Circuit Breaker '" + name + "' FORCE OPENED");
        } catch (Exception e) {
            return ResponseEntity.badRequest().body("Error: " + e.getMessage());
        }
    }
    
    @PostMapping("/circuit-breakers/{name}/force-close")
    @PreAuthorize("hasRole('ADMIN')")
    public ResponseEntity<String> forceClose(@PathVariable String name) {
        try {
            CircuitBreaker cb = cbRegistry.circuitBreaker(name);
            cb.transitionToClosedState();
            return ResponseEntity.ok("Circuit Breaker '" + name + "' FORCE CLOSED");
        } catch (Exception e) {
            return ResponseEntity.badRequest().body("Error: " + e.getMessage());
        }
    }
}
```

---

## Resilience4j Pattern Selection Guide

| สถานการณ์ | Pattern ที่แนะนำ | เหตุผล |
|-----------|----------------|-------|
| External API ช้า/ล้ม | Circuit Breaker + Retry | ป้องกัน Cascade Failure |
| Rate-limited External API | Rate Limiter | ป้องกัน 429 Too Many Requests |
| Database Slow | Bulkhead + Time Limiter | จำกัด Resource Usage |
| Network Flaky | Retry + Exponential Backoff | Handle Transient Failures |
| Critical Service | All Patterns Combined | Maximum Resilience |
| High-load Service | Bulkhead + Rate Limiter | Resource Protection |

---

## Best Practices

1. **Retry ก่อน Circuit Breaker:** Retry อยู่นอกสุด เพื่อให้ Circuit Breaker เห็น Retry เป็น Failed Calls
2. **อย่า Retry ทุก Exception:** Retry แค่ Transient Errors (Network, Timeout) ไม่ใช่ Business Errors
3. **ตั้ง Timeout ทุกที่:** ทุก External Call ต้องมี Timeout
4. **Monitor ด้วย Metrics:** Export Metrics ไปยัง Prometheus/Grafana
5. **Test ด้วย Fault Injection:** อย่าเชื่อ Code โดยไม่ทดสอบ

---

*[← Part 63: Distributed Tracing](./part-63-distributed-tracing.md) | [Part 65: Data Consistency →](./part-65-data-consistency.md)*

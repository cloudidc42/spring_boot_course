# Part 57: Chaos Engineering
## ขั้นตอนที่ 1881-1920

**ระดับ:** ระดับโลก (World-Class)
**เวลาเรียน:** 5-6 ชั่วโมง
**เป้าหมาย:** เข้าใจหลักการ Chaos Engineering และสามารถออกแบบ chaos experiments เพื่อทดสอบความแข็งแกร่งของระบบ

---

## ขั้นตอนที่ 1881-1885: หลักการ Chaos Engineering

### Chaos Engineering คืออะไร?

Chaos Engineering คือการจงใจทำให้ระบบล้มเหลวในสภาพแวดล้อม production (หรือ staging) เพื่อค้นหาจุดอ่อนก่อนที่ลูกค้าจะพบ

**หลักการ Chaos Engineering ของ Netflix:**
1. **Define steady state** - นิยามสภาวะปกติของระบบ
2. **Hypothesize** - คาดว่าระบบจะยังทำงานได้แม้มีความผิดพลาด
3. **Introduce variables** - ทำให้ระบบล้มเหลว (network latency, pod crashes, etc.)
4. **Disprove hypothesis** - ถ้าระบบล้มเหลว แสดงว่าพบจุดอ่อน
5. **Fix and repeat** - แก้ไขและทำซ้ำ

### ทำไมต้อง Chaos Engineering?

```
ระบบปกติ:
Service A → Service B → Database → OK ✅

สภาวะ Chaos:
Service A → Service B (SLOW 5s) → Database → Timeout ❌
Service A → Service B (DOWN) → Service C (fallback) → OK? 
Service A → Network Partition → Service B → ❌❌
```

### Dependencies

```xml
<!-- pom.xml -->
<dependencies>
    <!-- Chaos Monkey for Spring Boot -->
    <dependency>
        <groupId>de.codecentric</groupId>
        <artifactId>chaos-monkey-spring-boot</artifactId>
        <version>3.1.0</version>
    </dependency>
    
    <!-- Resilience4j for Circuit Breaker testing -->
    <dependency>
        <groupId>io.github.resilience4j</groupId>
        <artifactId>resilience4j-spring-boot3</artifactId>
        <version>2.1.0</version>
    </dependency>
    
    <!-- Micrometer for metrics -->
    <dependency>
        <groupId>io.micrometer</groupId>
        <artifactId>micrometer-registry-prometheus</artifactId>
    </dependency>
    
    <!-- Spring Boot Actuator -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
</dependencies>
```

---

## ขั้นตอนที่ 1886-1892: Chaos Monkey สำหรับ Spring Boot

### Configuration

```yaml
# application-chaos.yml
chaos:
  monkey:
    enabled: true
    watcher:
      # เปิดใช้ watcher สำหรับ Spring components
      service: true
      repository: false
      component: false
      rest-controller: true
      bean-classes: 
        - com.example.service.OrderService
        - com.example.service.PaymentService
    assaults:
      # ค่า default สำหรับทุก watcher
      level: 5              # 1-10: ความถี่ (1=หนึ่งครั้งในทุก 1 request, 10=ทุก request)
      latency-active: false
      exceptions-active: false
      kill-application-active: false
      memory-active: false

spring:
  profiles:
    active: chaos

management:
  endpoints:
    web:
      exposure:
        include: chaosmonkey,health,metrics,prometheus
```

### Enabling Chaos Monkey

```java
// ChaosMonkeyConfig.java
@Configuration
@Profile("chaos")
@Slf4j
public class ChaosMonkeyConfig {
    
    @Bean
    public ChaosMonkeySettings chaosMonkeySettings() {
        ChaosMonkeySettings settings = new ChaosMonkeySettings();
        settings.setEnabled(false); // เริ่มต้นปิดไว้ก่อน
        return settings;
    }
    
    @Bean
    public ChaosMonkeyRequestScope chaosMonkeyRequestScope(
            ChaosMonkeySettings settings,
            AssaultPropertiesUpdate assaultPropertiesUpdate) {
        return new ChaosMonkeyRequestScope(settings, assaultPropertiesUpdate, 
                                            new MetricEventPublisher());
    }
}
```

### ควบคุม Chaos ผ่าน Actuator API

```bash
# ดู status
curl http://localhost:8080/actuator/chaosmonkey

# เปิด Chaos Monkey
curl -X POST http://localhost:8080/actuator/chaosmonkey/enable

# ตั้งค่า Latency Assault
curl -X POST http://localhost:8080/actuator/chaosmonkey/assaults \
  -H "Content-Type: application/json" \
  -d '{
    "level": 5,
    "latencyActive": true,
    "latencyRangeStart": 1000,
    "latencyRangeEnd": 5000
  }'

# ตั้งค่า Exception Assault
curl -X POST http://localhost:8080/actuator/chaosmonkey/assaults \
  -H "Content-Type: application/json" \
  -d '{
    "level": 5,
    "exceptionsActive": true,
    "exception": {
      "type": "java.lang.RuntimeException",
      "arguments": [{"className": "java.lang.String", "value": "Chaos Exception"}]
    }
  }'

# ปิด Chaos Monkey
curl -X POST http://localhost:8080/actuator/chaosmonkey/disable
```

### Custom Chaos Assault

```java
// CustomChaosAssault.java
@Component
@Profile("chaos")
@Slf4j
public class NetworkPartitionAssault implements ChaosMonkeyAssault {
    
    @Autowired
    private Environment environment;
    
    private volatile boolean active = false;
    
    @Override
    public boolean isActive() {
        return active;
    }
    
    @Override
    public void attack() {
        // Simulate network partition โดย throw specific exception
        log.warn("CHAOS: Simulating network partition!");
        throw new NetworkPartitionException("Simulated network partition");
    }
    
    public void setActive(boolean active) {
        this.active = active;
    }
}

// NetworkPartitionException.java
public class NetworkPartitionException extends RuntimeException {
    public NetworkPartitionException(String message) {
        super(message);
    }
}
```

---

## ขั้นตอนที่ 1893-1898: Resilience Testing

### Testing Circuit Breaker Behavior

```java
// CircuitBreakerChaosTest.java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@ActiveProfiles("chaos")
@Slf4j
class CircuitBreakerChaosTest {
    
    @LocalServerPort
    private int port;
    
    @Autowired
    private CircuitBreakerRegistry circuitBreakerRegistry;
    
    @Autowired
    private TestRestTemplate restTemplate;
    
    @MockBean
    private ExternalPaymentService externalPaymentService;
    
    @Test
    void testCircuitBreakerOpensAfterFailures() {
        // Setup: mock service ให้ fail
        when(externalPaymentService.processPayment(any()))
            .thenThrow(new RuntimeException("Service unavailable"));
        
        // Steady state metrics
        SteadyStateMetrics before = collectMetrics();
        
        // ส่ง requests เพื่อ trigger circuit breaker
        int requestCount = 20;
        List<ResponseEntity<String>> responses = IntStream.range(0, requestCount)
            .mapToObj(i -> restTemplate.postForEntity(
                "/api/payments",
                new PaymentRequest("100.00", "THB"),
                String.class
            ))
            .collect(Collectors.toList());
        
        // ตรวจสอบว่า circuit breaker เปิด
        CircuitBreaker cb = circuitBreakerRegistry.circuitBreaker("paymentService");
        
        // รอให้ circuit breaker เปลี่ยนสถานะ
        await().atMost(10, TimeUnit.SECONDS)
            .until(() -> cb.getState() == CircuitBreaker.State.OPEN);
        
        log.info("Circuit breaker state: {}", cb.getState());
        
        // หลังจาก circuit เปิด: ต้อง fallback
        ResponseEntity<String> fallbackResponse = restTemplate.postForEntity(
            "/api/payments",
            new PaymentRequest("100.00", "THB"),
            String.class
        );
        
        assertThat(fallbackResponse.getStatusCode()).isEqualTo(HttpStatus.SERVICE_UNAVAILABLE);
        
        // Collect metrics หลัง chaos
        SteadyStateMetrics after = collectMetrics();
        
        // System ควรยังทำงานได้ แค่ degraded
        assertThat(after.getAvailability()).isGreaterThan(0.9);
        
        log.info("Before: {}, After: {}", before, after);
    }
    
    @Test
    void testCircuitBreakerHalfOpenRecovery() throws InterruptedException {
        CircuitBreaker cb = circuitBreakerRegistry.circuitBreaker("paymentService");
        
        // Force circuit open
        cb.transitionToOpenState();
        assertThat(cb.getState()).isEqualTo(CircuitBreaker.State.OPEN);
        
        // รอ wait duration ผ่าน
        Thread.sleep(6000); // assume waitDuration = 5s
        
        // Circuit ควร half-open แล้ว
        assertThat(cb.getState()).isEqualTo(CircuitBreaker.State.HALF_OPEN);
        
        // Setup mock ให้ success
        when(externalPaymentService.processPayment(any()))
            .thenReturn(new PaymentResult("SUCCESS", "TXN-001"));
        
        // ส่ง probe requests
        restTemplate.postForEntity("/api/payments",
            new PaymentRequest("100.00", "THB"), String.class);
        
        // Circuit ควร close
        await().atMost(5, TimeUnit.SECONDS)
            .until(() -> cb.getState() == CircuitBreaker.State.CLOSED);
        
        assertThat(cb.getState()).isEqualTo(CircuitBreaker.State.CLOSED);
        log.info("Circuit breaker recovered to: {}", cb.getState());
    }
    
    private SteadyStateMetrics collectMetrics() {
        // เก็บ metrics จาก Actuator
        ResponseEntity<Map> response = restTemplate.getForEntity(
            "/actuator/metrics/http.server.requests", Map.class);
        return new SteadyStateMetrics(response.getBody());
    }
}
```

### Latency Injection Test

```java
// LatencyInjectionTest.java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Slf4j
class LatencyInjectionTest {
    
    @LocalServerPort
    private int port;
    
    @Autowired
    private ChaosMonkeySettings chaosMonkeySettings;
    
    @Autowired
    private AssaultProperties assaultProperties;
    
    @Test
    void testSystemUnderLatency() throws Exception {
        // ตั้งค่า latency injection
        assaultProperties.setLatencyActive(true);
        assaultProperties.setLatencyRangeStart(500);
        assaultProperties.setLatencyRangeEnd(2000);
        assaultProperties.setLevel(3);
        chaosMonkeySettings.setEnabled(true);
        
        long startTime = System.currentTimeMillis();
        
        // ส่ง requests ขณะที่ chaos active
        List<CompletableFuture<Long>> futures = IntStream.range(0, 50)
            .mapToObj(i -> CompletableFuture.supplyAsync(() -> {
                long reqStart = System.currentTimeMillis();
                // ส่ง request
                return System.currentTimeMillis() - reqStart;
            }))
            .collect(Collectors.toList());
        
        List<Long> responseTimes = futures.stream()
            .map(CompletableFuture::join)
            .collect(Collectors.toList());
        
        // ปิด chaos
        chaosMonkeySettings.setEnabled(false);
        assaultProperties.setLatencyActive(false);
        
        // วิเคราะห์ผล
        LongSummaryStatistics stats = responseTimes.stream()
            .mapToLong(Long::longValue)
            .summaryStatistics();
        
        log.info("Response time stats - Min: {}ms, Max: {}ms, Avg: {}ms",
            stats.getMin(), stats.getMax(), (long) stats.getAverage());
        
        // ระบบต้องยังตอบสนองได้ (ไม่ timeout ทั้งหมด)
        long successfulRequests = responseTimes.stream()
            .filter(t -> t < 10000) // < 10 seconds
            .count();
        
        double successRate = (double) successfulRequests / responseTimes.size();
        assertThat(successRate).isGreaterThan(0.95); // 95% success rate
        
        log.info("Success rate under latency: {:.2f}%", successRate * 100);
    }
}
```

---

## ขั้นตอนที่ 1899-1905: Chaos Experiments กับ Kubernetes/Istio

### Chaos Experiment Definition (YAML)

```yaml
# chaos-experiment-pod-failure.yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: PodChaos
metadata:
  name: pod-failure-experiment
  namespace: production
spec:
  action: pod-failure
  mode: one
  selector:
    namespaces:
      - production
    labelSelectors:
      app: order-service
  duration: "60s"
  scheduler:
    cron: "@every 1h"  # รันทุกชั่วโมง
---
# chaos-experiment-network-latency.yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: NetworkChaos
metadata:
  name: network-latency-experiment
  namespace: production
spec:
  action: delay
  mode: one
  selector:
    namespaces:
      - production
    labelSelectors:
      app: payment-service
  delay:
    latency: "500ms"
    jitter: "100ms"
    correlation: "25"
  duration: "120s"
  direction: to
  target:
    selector:
      namespaces:
        - production
      labelSelectors:
        app: order-service
    mode: one
```

### Litmus Chaos Experiments

```yaml
# litmus-chaos-experiment.yaml
apiVersion: litmuschaos.io/v1alpha1
kind: ChaosEngine
metadata:
  name: order-service-chaos
  namespace: default
spec:
  appinfo:
    appns: production
    applabel: "app=order-service"
    appkind: deployment
  chaosServiceAccount: litmus-admin
  experiments:
    - name: pod-delete
      spec:
        components:
          env:
            - name: TOTAL_CHAOS_DURATION
              value: "60"
            - name: CHAOS_INTERVAL
              value: "10"
            - name: FORCE
              value: "false"
    
    - name: container-kill
      spec:
        components:
          env:
            - name: TOTAL_CHAOS_DURATION
              value: "120"
            - name: CHAOS_INTERVAL
              value: "20"
            - name: TARGET_CONTAINER
              value: order-service
```

### Chaos Testing กับ Istio

```yaml
# istio-fault-injection.yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: payment-service-chaos
spec:
  hosts:
    - payment-service
  http:
    - fault:
        delay:
          percentage:
            value: 10.0  # 10% ของ requests จะ delay
          fixedDelay: 5s
        abort:
          percentage:
            value: 5.0   # 5% ของ requests จะ fail
          httpStatus: 503
      route:
        - destination:
            host: payment-service
            port:
              number: 8080
```

---

## ขั้นตอนที่ 1906-1910: Steady State Hypothesis

### Defining Steady State

```java
// SteadyStateService.java
@Service
@Slf4j
public class SteadyStateService {
    
    private final MeterRegistry meterRegistry;
    private final CircuitBreakerRegistry circuitBreakerRegistry;
    
    public SteadyStateService(MeterRegistry meterRegistry,
                               CircuitBreakerRegistry circuitBreakerRegistry) {
        this.meterRegistry = meterRegistry;
        this.circuitBreakerRegistry = circuitBreakerRegistry;
    }
    
    /**
     * วัด steady state ของระบบ
     */
    public SteadyStateSnapshot captureSnapshot() {
        return SteadyStateSnapshot.builder()
            .timestamp(Instant.now())
            .orderSuccessRate(calculateOrderSuccessRate())
            .paymentSuccessRate(calculatePaymentSuccessRate())
            .p99Latency(calculateP99Latency("order"))
            .p99Latency99th(calculateP99Latency("payment"))
            .errorRate(calculateErrorRate())
            .activeCircuitBreakers(countOpenCircuitBreakers())
            .build();
    }
    
    /**
     * เปรียบเทียบ steady state
     */
    public SteadyStateComparison compare(SteadyStateSnapshot before, 
                                          SteadyStateSnapshot after) {
        double orderSuccessRateDelta = 
            after.getOrderSuccessRate() - before.getOrderSuccessRate();
        double paymentSuccessRateDelta = 
            after.getPaymentSuccessRate() - before.getPaymentSuccessRate();
        double latencyIncrease = 
            after.getP99Latency() - before.getP99Latency();
        
        boolean withinBounds = 
            orderSuccessRateDelta > -0.05 &&  // ลดลงได้ไม่เกิน 5%
            paymentSuccessRateDelta > -0.02 && // ลดลงได้ไม่เกิน 2%
            latencyIncrease < 500 &&            // เพิ่มได้ไม่เกิน 500ms
            after.getActiveCircuitBreakers() <= before.getActiveCircuitBreakers() + 1;
        
        return SteadyStateComparison.builder()
            .withinBounds(withinBounds)
            .orderSuccessRateDelta(orderSuccessRateDelta)
            .paymentSuccessRateDelta(paymentSuccessRateDelta)
            .latencyIncrease(latencyIncrease)
            .hypothesis(withinBounds ? "CONFIRMED" : "DISPROVED")
            .build();
    }
    
    private double calculateOrderSuccessRate() {
        Counter successCounter = meterRegistry.find("order.processed")
            .tag("status", "success").counter();
        Counter failCounter = meterRegistry.find("order.processed")
            .tag("status", "failure").counter();
        
        if (successCounter == null) return 1.0;
        
        double total = successCounter.count() + 
            (failCounter != null ? failCounter.count() : 0);
        
        return total == 0 ? 1.0 : successCounter.count() / total;
    }
    
    private double calculateP99Latency(String service) {
        Timer timer = meterRegistry.find("http.server.requests")
            .tag("uri", "/api/" + service)
            .timer();
        
        if (timer == null) return 0.0;
        
        return timer.percentile(0.99, TimeUnit.MILLISECONDS);
    }
    
    private double calculateErrorRate() {
        Counter errorCounter = meterRegistry.find("http.server.requests")
            .tag("status", "500").counter();
        
        if (errorCounter == null) return 0.0;
        return errorCounter.count();
    }
    
    private long countOpenCircuitBreakers() {
        return circuitBreakerRegistry.getAllCircuitBreakers().stream()
            .filter(cb -> cb.getState() == CircuitBreaker.State.OPEN)
            .count();
    }
}
```

### Chaos Experiment Runner

```java
// ChaosExperimentRunner.java
@Component
@Slf4j
public class ChaosExperimentRunner {
    
    private final SteadyStateService steadyStateService;
    private final ChaosMonkeySettings chaosMonkeySettings;
    private final AssaultProperties assaultProperties;
    private final ApplicationEventPublisher eventPublisher;
    
    public ChaosExperimentRunner(SteadyStateService steadyStateService,
                                  ChaosMonkeySettings chaosMonkeySettings,
                                  AssaultProperties assaultProperties,
                                  ApplicationEventPublisher eventPublisher) {
        this.steadyStateService = steadyStateService;
        this.chaosMonkeySettings = chaosMonkeySettings;
        this.assaultProperties = assaultProperties;
        this.eventPublisher = eventPublisher;
    }
    
    /**
     * รัน chaos experiment ตามขั้นตอน
     */
    public ChaosExperimentResult runExperiment(ChaosExperiment experiment) 
            throws InterruptedException {
        log.info("Starting chaos experiment: {}", experiment.getName());
        
        // Step 1: Baseline measurement
        log.info("Step 1: Measuring baseline steady state...");
        Thread.sleep(30000); // รอ 30 วินาที
        SteadyStateSnapshot baseline = steadyStateService.captureSnapshot();
        log.info("Baseline captured: {}", baseline);
        
        // Step 2: Inject chaos
        log.info("Step 2: Injecting chaos - {}", experiment.getChaosType());
        injectChaos(experiment);
        
        // Step 3: Monitor during chaos
        log.info("Step 3: Monitoring system during chaos for {}s...", 
            experiment.getDurationSeconds());
        
        List<SteadyStateSnapshot> duringChaos = new ArrayList<>();
        long endTime = System.currentTimeMillis() + 
            (experiment.getDurationSeconds() * 1000L);
        
        while (System.currentTimeMillis() < endTime) {
            Thread.sleep(10000); // เก็บ metrics ทุก 10 วินาที
            duringChaos.add(steadyStateService.captureSnapshot());
        }
        
        // Step 4: Stop chaos
        log.info("Step 4: Stopping chaos...");
        stopChaos();
        
        // Step 5: Measure recovery
        log.info("Step 5: Measuring recovery...");
        Thread.sleep(60000); // รอ 1 นาทีให้ระบบ recover
        SteadyStateSnapshot afterChaos = steadyStateService.captureSnapshot();
        
        // Step 6: Analyze results
        SteadyStateComparison comparison = 
            steadyStateService.compare(baseline, afterChaos);
        
        ChaosExperimentResult result = ChaosExperimentResult.builder()
            .experimentName(experiment.getName())
            .baseline(baseline)
            .duringChaosSnapshots(duringChaos)
            .afterChaos(afterChaos)
            .comparison(comparison)
            .hypothesis(experiment.getHypothesis())
            .hypothesisConfirmed(comparison.isWithinBounds())
            .build();
        
        log.info("Experiment '{}' complete. Hypothesis: {}",
            experiment.getName(), result.isHypothesisConfirmed() ? "CONFIRMED" : "DISPROVED");
        
        eventPublisher.publishEvent(new ChaosExperimentCompletedEvent(result));
        
        return result;
    }
    
    private void injectChaos(ChaosExperiment experiment) {
        switch (experiment.getChaosType()) {
            case LATENCY:
                assaultProperties.setLatencyActive(true);
                assaultProperties.setLatencyRangeStart(
                    experiment.getLatencyMs() - 200);
                assaultProperties.setLatencyRangeEnd(
                    experiment.getLatencyMs() + 200);
                assaultProperties.setLevel(experiment.getLevel());
                chaosMonkeySettings.setEnabled(true);
                break;
                
            case EXCEPTION:
                assaultProperties.setExceptionsActive(true);
                assaultProperties.setLevel(experiment.getLevel());
                chaosMonkeySettings.setEnabled(true);
                break;
                
            case KILL_APP:
                assaultProperties.setKillApplicationActive(true);
                chaosMonkeySettings.setEnabled(true);
                break;
        }
    }
    
    private void stopChaos() {
        chaosMonkeySettings.setEnabled(false);
        assaultProperties.setLatencyActive(false);
        assaultProperties.setExceptionsActive(false);
        assaultProperties.setKillApplicationActive(false);
    }
}
```

---

## ขั้นตอนที่ 1911-1916: Measuring System Resilience

### Resilience Metrics Dashboard

```java
// ResilienceMetricsCollector.java
@Component
@Slf4j
public class ResilienceMetricsCollector {
    
    private final MeterRegistry meterRegistry;
    private final CircuitBreakerRegistry circuitBreakerRegistry;
    private final RetryRegistry retryRegistry;
    
    public ResilienceMetricsCollector(MeterRegistry meterRegistry,
                                       CircuitBreakerRegistry circuitBreakerRegistry,
                                       RetryRegistry retryRegistry) {
        this.meterRegistry = meterRegistry;
        this.circuitBreakerRegistry = circuitBreakerRegistry;
        this.retryRegistry = retryRegistry;
    }
    
    @Scheduled(fixedRate = 60000) // ทุก 1 นาที
    public void collectResilienceMetrics() {
        // Circuit Breaker metrics
        circuitBreakerRegistry.getAllCircuitBreakers().forEach(cb -> {
            CircuitBreaker.Metrics metrics = cb.getMetrics();
            
            Gauge.builder("resilience.circuit_breaker.success_rate")
                .tag("name", cb.getName())
                .register(meterRegistry)
                .set(metrics.getSuccessRate());
            
            Gauge.builder("resilience.circuit_breaker.failure_rate")
                .tag("name", cb.getName())
                .register(meterRegistry)
                .set(metrics.getFailureRate());
            
            Gauge.builder("resilience.circuit_breaker.state")
                .tag("name", cb.getName())
                .register(meterRegistry)
                .set(cb.getState() == CircuitBreaker.State.OPEN ? 1.0 : 0.0);
            
            log.debug("Circuit Breaker '{}': state={}, failure_rate={}%",
                cb.getName(), cb.getState(), metrics.getFailureRate());
        });
        
        // Retry metrics
        retryRegistry.getAllRetries().forEach(retry -> {
            Retry.Metrics metrics = retry.getMetrics();
            
            Counter.builder("resilience.retry.calls")
                .tag("name", retry.getName())
                .tag("type", "success_without_retry")
                .register(meterRegistry)
                .increment(metrics.getNumberOfSuccessfulCallsWithoutRetryAttempt());
            
            Counter.builder("resilience.retry.calls")
                .tag("name", retry.getName())
                .tag("type", "success_with_retry")
                .register(meterRegistry)
                .increment(metrics.getNumberOfSuccessfulCallsWithRetryAttempt());
            
            Counter.builder("resilience.retry.calls")
                .tag("name", retry.getName())
                .tag("type", "failed")
                .register(meterRegistry)
                .increment(metrics.getNumberOfFailedCallsWithoutRetryAttempt());
        });
    }
    
    /**
     * คำนวณ MTTR (Mean Time To Recovery)
     */
    public Duration calculateMTTR(String serviceName, 
                                   List<ServiceIncident> incidents) {
        if (incidents.isEmpty()) return Duration.ZERO;
        
        long totalRecoveryMs = incidents.stream()
            .filter(i -> i.getRecoveredAt() != null)
            .mapToLong(i -> Duration.between(
                i.getDetectedAt(), 
                i.getRecoveredAt()
            ).toMillis())
            .sum();
        
        long resolvedCount = incidents.stream()
            .filter(i -> i.getRecoveredAt() != null)
            .count();
        
        if (resolvedCount == 0) return Duration.ZERO;
        
        return Duration.ofMillis(totalRecoveryMs / resolvedCount);
    }
    
    /**
     * คำนวณ Availability percentage
     */
    public double calculateAvailability(String serviceName, 
                                         Duration totalTime,
                                         List<ServiceIncident> incidents) {
        long totalDowntimeMs = incidents.stream()
            .mapToLong(i -> {
                Instant end = i.getRecoveredAt() != null 
                    ? i.getRecoveredAt() 
                    : Instant.now();
                return Duration.between(i.getDetectedAt(), end).toMillis();
            })
            .sum();
        
        long totalTimeMs = totalTime.toMillis();
        double availability = 1.0 - ((double) totalDowntimeMs / totalTimeMs);
        
        log.info("Service '{}' availability: {:.4f}% (downtime: {}ms / total: {}ms)",
            serviceName, availability * 100, totalDowntimeMs, totalTimeMs);
        
        return availability;
    }
}
```

### Chaos Report Generator

```java
// ChaosReportService.java
@Service
@Slf4j
public class ChaosReportService {
    
    private final List<ChaosExperimentResult> experimentHistory = 
        new CopyOnWriteArrayList<>();
    
    @EventListener
    public void onExperimentCompleted(ChaosExperimentCompletedEvent event) {
        experimentHistory.add(event.getResult());
    }
    
    public ChaosReport generateReport(String period) {
        List<ChaosExperimentResult> results = experimentHistory.stream()
            .filter(r -> isWithinPeriod(r.getRunAt(), period))
            .collect(Collectors.toList());
        
        long confirmedCount = results.stream()
            .filter(ChaosExperimentResult::isHypothesisConfirmed)
            .count();
        
        long disprovedCount = results.size() - confirmedCount;
        
        Map<String, Long> weaknessByType = results.stream()
            .filter(r -> !r.isHypothesisConfirmed())
            .collect(Collectors.groupingBy(
                r -> r.getComparison().getWeaknessType(),
                Collectors.counting()
            ));
        
        double systemResilienceScore = results.isEmpty() ? 100.0 
            : (double) confirmedCount / results.size() * 100;
        
        return ChaosReport.builder()
            .period(period)
            .totalExperiments(results.size())
            .confirmedHypotheses(confirmedCount)
            .disprovedHypotheses(disprovedCount)
            .systemResilienceScore(systemResilienceScore)
            .weaknessesByType(weaknessByType)
            .experiments(results)
            .generatedAt(Instant.now())
            .build();
    }
}
```

---

## ขั้นตอนที่ 1917-1920: GameDay Planning

### GameDay Runbook

```java
// GameDayRunbook.java
@Component
@Slf4j
public class GameDayRunbook {
    
    private final ChaosExperimentRunner runner;
    private final ResilienceMetricsCollector metricsCollector;
    private final NotificationService notificationService;
    
    public GameDayRunbook(ChaosExperimentRunner runner,
                          ResilienceMetricsCollector metricsCollector,
                          NotificationService notificationService) {
        this.runner = runner;
        this.metricsCollector = metricsCollector;
        this.notificationService = notificationService;
    }
    
    /**
     * รัน Game Day exercises
     */
    @Async
    public CompletableFuture<GameDayReport> runGameDay(GameDayConfig config) {
        log.info("🎮 Game Day Started: {}", config.getName());
        notificationService.notifyTeam("Game Day Started: " + config.getName());
        
        List<ChaosExperimentResult> results = new ArrayList<>();
        
        try {
            for (ChaosExperiment experiment : config.getExperiments()) {
                log.info("Running experiment: {}", experiment.getName());
                
                ChaosExperimentResult result = runner.runExperiment(experiment);
                results.add(result);
                
                // หยุดถ้าพบปัญหาร้ายแรง
                if (result.isCriticalFailure()) {
                    log.error("Critical failure detected! Stopping Game Day.");
                    notificationService.alertTeam(
                        "CRITICAL: Game Day stopped due to: " + 
                        result.getCriticalFailureReason()
                    );
                    break;
                }
                
                // รอระหว่าง experiments
                Thread.sleep(config.getCooldownSeconds() * 1000L);
            }
        } catch (Exception e) {
            log.error("Game Day failed with exception", e);
            notificationService.alertTeam("Game Day FAILED: " + e.getMessage());
        }
        
        GameDayReport report = GameDayReport.builder()
            .name(config.getName())
            .experiments(results)
            .overallResilienceScore(calculateScore(results))
            .recommendations(generateRecommendations(results))
            .completedAt(Instant.now())
            .build();
        
        log.info("🎮 Game Day Complete. Score: {}", report.getOverallResilienceScore());
        notificationService.notifyTeam("Game Day Complete. Report: " + report.getSummary());
        
        return CompletableFuture.completedFuture(report);
    }
    
    private double calculateScore(List<ChaosExperimentResult> results) {
        if (results.isEmpty()) return 100.0;
        
        long passed = results.stream()
            .filter(ChaosExperimentResult::isHypothesisConfirmed)
            .count();
        
        return (double) passed / results.size() * 100;
    }
    
    private List<String> generateRecommendations(List<ChaosExperimentResult> results) {
        List<String> recommendations = new ArrayList<>();
        
        results.stream()
            .filter(r -> !r.isHypothesisConfirmed())
            .forEach(r -> {
                switch (r.getComparison().getWeaknessType()) {
                    case "HIGH_LATENCY":
                        recommendations.add(
                            "เพิ่ม timeout configuration สำหรับ " + r.getExperimentName()
                        );
                        break;
                    case "LOW_AVAILABILITY":
                        recommendations.add(
                            "เพิ่ม replica count หรือ horizontal scaling สำหรับ " + 
                            r.getExperimentName()
                        );
                        break;
                    case "CIRCUIT_BREAKER_OPEN":
                        recommendations.add(
                            "ปรับ circuit breaker threshold สำหรับ " + r.getExperimentName()
                        );
                        break;
                }
            });
        
        return recommendations;
    }
}
```

### ตัวอย่าง GameDay Config

```yaml
# gameday-config.yaml
gameday:
  name: "Order Service GameDay - Q4 2024"
  cooldown-seconds: 300
  experiments:
    - name: "Payment Service Latency"
      hypothesis: "System maintains 99% order success rate with 1s payment latency"
      chaos-type: LATENCY
      target-service: payment-service
      latency-ms: 1000
      duration-seconds: 120
      level: 5
      
    - name: "Database Connection Pool Exhaustion"
      hypothesis: "System degrades gracefully when DB pool is exhausted"
      chaos-type: EXCEPTION
      target-service: order-service
      exception-class: "org.springframework.dao.DataAccessException"
      duration-seconds: 60
      level: 3
      
    - name: "Inventory Service Pod Kill"
      hypothesis: "System auto-recovers within 30s after pod failure"
      chaos-type: POD_KILL
      target-service: inventory-service
      duration-seconds: 180
      recovery-sla-seconds: 30
```

---

## สรุป Part 57

ในส่วนนี้เราได้เรียนรู้:

1. **Chaos Engineering Principles** - GameDay, fault injection, steady state hypothesis
2. **Chaos Monkey** - เครื่องมือสำหรับ Spring Boot chaos testing
3. **Resilience Testing** - ทดสอบ circuit breaker, latency injection
4. **Kubernetes/Istio Chaos** - ใช้ Chaos Mesh และ Istio fault injection
5. **Steady State Hypothesis** - นิยามและวัด baseline
6. **Measuring Resilience** - MTTR, Availability, circuit breaker metrics
7. **GameDay Planning** - วางแผนและรัน game day exercises

---

*[← Part 56: Distributed Locks](./part-56-distributed-locks.md) | [Part 58: Hexagonal Architecture →](./part-58-hexagonal-architecture.md)*

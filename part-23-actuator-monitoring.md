# Part 23: Actuator & Monitoring
## ขั้นตอนที่ 606-635

> **ระดับ:** กลาง-สูง (Intermediate-Advanced)  
> **เวลาเรียน:** 4-5 ชั่วโมง  
> **เป้าหมาย:** Monitor Spring Boot application ด้วย Actuator และ Micrometer

---

## ขั้นตอนที่ 606: Actuator Setup

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<!-- Prometheus metrics -->
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus,loggers,env,beans,mappings,scheduledtasks,threaddump,heapdump,httptrace
      base-path: /actuator
  endpoint:
    health:
      show-details: when_authorized
      show-components: when_authorized
    env:
      show-values: when_authorized
  info:
    env:
      enabled: true
    git:
      enabled: true
      mode: full
    build:
      enabled: true
  health:
    redis:
      enabled: true
    db:
      enabled: true
    diskspace:
      enabled: true
```

---

## ขั้นตอนที่ 607: Health Indicators

```java
// Custom Health Indicator
@Component
public class ExternalApiHealthIndicator implements HealthIndicator {
    
    private final RestTemplate restTemplate;
    
    public ExternalApiHealthIndicator(RestTemplateBuilder builder) {
        this.restTemplate = builder.build();
    }
    
    @Override
    public Health health() {
        try {
            ResponseEntity<String> response = restTemplate.getForEntity(
                "https://api.payment-gateway.com/health",
                String.class
            );
            
            if (response.getStatusCode().is2xxSuccessful()) {
                return Health.up()
                    .withDetail("url", "https://api.payment-gateway.com")
                    .withDetail("status", response.getStatusCode())
                    .build();
            }
            
            return Health.down()
                .withDetail("status", response.getStatusCode())
                .build();
                
        } catch (Exception ex) {
            return Health.down()
                .withDetail("error", ex.getMessage())
                .build();
        }
    }
}

// Composite Health Indicator
@Component("applicationHealth")
public class ApplicationHealthIndicator extends AbstractHealthIndicator {
    
    private final ProductRepository productRepository;
    private final OrderRepository orderRepository;
    
    @Override
    protected void doHealthCheck(Health.Builder builder) {
        long productCount = productRepository.count();
        long pendingOrders = orderRepository.countByStatus(OrderStatus.PENDING);
        
        if (pendingOrders > 1000) {
            builder.down()
                .withDetail("reason", "Too many pending orders: " + pendingOrders);
        } else {
            builder.up()
                .withDetail("products", productCount)
                .withDetail("pendingOrders", pendingOrders);
        }
    }
}
```

---

## ขั้นตอนที่ 608: Info Endpoint

```java
// Custom Info Contributor
@Component
public class AppInfoContributor implements InfoContributor {
    
    @Override
    public void contribute(Info.Builder builder) {
        builder.withDetail("app", Map.of(
            "name", "Spring Boot Course App",
            "version", "1.0.0",
            "description", "E-Commerce REST API",
            "contact", "support@example.com"
        ));
        
        builder.withDetail("runtime", Map.of(
            "java.version", System.getProperty("java.version"),
            "spring.version", SpringVersion.getVersion(),
            "os", System.getProperty("os.name")
        ));
    }
}
```

```yaml
# application.yml
info:
  app:
    name: My Spring Boot App
    version: "@project.version@"
    description: "@project.description@"
  build:
    artifact: "@project.artifactId@"
    group: "@project.groupId@"
  git:
    commit: "@git.commit.id.abbrev@"
    branch: "@git.branch@"
```

---

## ขั้นตอนที่ 609: Custom Metrics with Micrometer

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class OrderService {
    
    private final OrderRepository orderRepository;
    private final MeterRegistry meterRegistry;
    
    // Counter - counts occurrences
    private final Counter orderCreatedCounter;
    private final Counter orderFailedCounter;
    
    // Gauge - current value
    private final AtomicInteger pendingOrdersCount = new AtomicInteger(0);
    
    // Timer - measures duration
    private final Timer orderProcessingTimer;
    
    // Distribution Summary - records distribution
    private final DistributionSummary orderAmountSummary;
    
    @Autowired
    public OrderService(OrderRepository orderRepository, MeterRegistry meterRegistry) {
        this.orderRepository = orderRepository;
        this.meterRegistry = meterRegistry;
        
        this.orderCreatedCounter = Counter.builder("orders.created.total")
            .description("Total number of orders created")
            .tag("env", "production")
            .register(meterRegistry);
        
        this.orderFailedCounter = Counter.builder("orders.failed.total")
            .description("Total number of failed orders")
            .register(meterRegistry);
        
        Gauge.builder("orders.pending.count", pendingOrdersCount, AtomicInteger::get)
            .description("Number of pending orders")
            .register(meterRegistry);
        
        this.orderProcessingTimer = Timer.builder("orders.processing.time")
            .description("Order processing duration")
            .register(meterRegistry);
        
        this.orderAmountSummary = DistributionSummary.builder("orders.amount")
            .description("Order amount distribution")
            .baseUnit("THB")
            .register(meterRegistry);
    }
    
    @Transactional
    public OrderResponse create(Long userId, CreateOrderRequest request) {
        return orderProcessingTimer.record(() -> {
            try {
                OrderResponse order = doCreateOrder(userId, request);
                
                orderCreatedCounter.increment();
                orderAmountSummary.record(order.totalAmount().doubleValue());
                pendingOrdersCount.incrementAndGet();
                
                // Tag-based metrics
                meterRegistry.counter("orders.by.status", "status", "CREATED").increment();
                
                return order;
            } catch (Exception e) {
                orderFailedCounter.increment();
                throw e;
            }
        });
    }
    
    // Custom metrics with tags
    public void recordCheckoutStep(String step, boolean success) {
        meterRegistry.counter("checkout.steps",
            "step", step,
            "success", String.valueOf(success)
        ).increment();
    }
}
```

---

## ขั้นตอนที่ 610: Prometheus + Grafana Setup

```yaml
# docker-compose.yml - add monitoring stack
services:
  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./config/prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.retention.time=30d'

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    environment:
      GF_SECURITY_ADMIN_PASSWORD: admin123
    volumes:
      - grafana_data:/var/lib/grafana
    depends_on:
      - prometheus

volumes:
  prometheus_data:
  grafana_data:
```

```yaml
# config/prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'spring-boot-app'
    metrics_path: '/actuator/prometheus'
    scrape_interval: 5s
    static_configs:
      - targets: ['host.docker.internal:8080']
        labels:
          app: 'spring-boot-course'
          env: 'development'
```

---

## ขั้นตอนที่ 611: AOP-Based Metrics

```java
@Aspect
@Component
@RequiredArgsConstructor
@Slf4j
public class MetricsAspect {
    
    private final MeterRegistry meterRegistry;
    
    // Count method calls
    @Around("@annotation(Monitored)")
    public Object measureMethodMetrics(ProceedingJoinPoint pjp) throws Throwable {
        String methodName = pjp.getSignature().getName();
        String className = pjp.getTarget().getClass().getSimpleName();
        
        Timer.Sample sample = Timer.start(meterRegistry);
        
        try {
            Object result = pjp.proceed();
            
            sample.stop(Timer.builder("method.execution")
                .tag("class", className)
                .tag("method", methodName)
                .tag("success", "true")
                .register(meterRegistry));
            
            return result;
            
        } catch (Exception e) {
            sample.stop(Timer.builder("method.execution")
                .tag("class", className)
                .tag("method", methodName)
                .tag("success", "false")
                .tag("exception", e.getClass().getSimpleName())
                .register(meterRegistry));
            
            throw e;
        }
    }
    
    // Monitor slow requests
    @Around("within(@org.springframework.web.bind.annotation.RestController *)")
    public Object monitorControllers(ProceedingJoinPoint pjp) throws Throwable {
        long start = System.currentTimeMillis();
        
        try {
            return pjp.proceed();
        } finally {
            long duration = System.currentTimeMillis() - start;
            if (duration > 1000) {
                log.warn("Slow endpoint: {}.{} took {}ms",
                    pjp.getTarget().getClass().getSimpleName(),
                    pjp.getSignature().getName(),
                    duration);
                
                meterRegistry.counter("slow.endpoints",
                    "class", pjp.getTarget().getClass().getSimpleName(),
                    "method", pjp.getSignature().getName()
                ).increment();
            }
        }
    }
}

@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Monitored {}
```

---

## ขั้นตอนที่ 612: Request Tracing

```java
// Add Micrometer Tracing + Zipkin
// spring-boot-starter-actuator includes micrometer-tracing
// Add: io.micrometer:micrometer-tracing-bridge-brave
//      io.zipkin.reporter2:zipkin-reporter-brave

// application.yml
management:
  tracing:
    sampling:
      probability: 1.0  # 100% in dev, 10% (0.1) in production
  zipkin:
    tracing:
      endpoint: http://localhost:9411/api/v2/spans
logging:
  pattern:
    level: "%5p [${spring.application.name:},%X{traceId:-},%X{spanId:-}]"
```

```yaml
# docker-compose.yml - Zipkin
services:
  zipkin:
    image: openzipkin/zipkin:latest
    ports:
      - "9411:9411"
```

---

## ขั้นตอนที่ 613-635: Grafana Dashboard Queries

```promql
# Request rate
sum(rate(http_server_requests_seconds_count[5m])) by (method, uri)

# Error rate
sum(rate(http_server_requests_seconds_count{status=~"5.."}[5m]))
/
sum(rate(http_server_requests_seconds_count[5m]))

# P99 latency
histogram_quantile(0.99, sum(rate(http_server_requests_seconds_bucket[5m])) by (le, uri))

# JVM heap usage
jvm_memory_used_bytes{area="heap"} / jvm_memory_max_bytes{area="heap"} * 100

# Active DB connections
hikaricp_connections_active

# Order creation rate
rate(orders_created_total[5m])

# Custom: business metrics
sum(orders_amount_sum) / sum(orders_amount_count)  # Average order value
```

### Actuator Endpoints Summary

```
GET /actuator/health           - Application health
GET /actuator/info             - App info, git info
GET /actuator/metrics          - Available metrics
GET /actuator/metrics/{name}   - Specific metric
GET /actuator/prometheus       - Prometheus format
GET /actuator/env              - Environment variables
GET /actuator/loggers          - Logger configuration
POST /actuator/loggers/{name}  - Change log level
GET /actuator/beans            - Spring beans
GET /actuator/mappings         - URL mappings
GET /actuator/scheduledtasks   - Scheduled tasks
POST /actuator/shutdown        - Shutdown app (disabled by default)
```

---

*[← Part 22: Async & Scheduling](./part-22-async-scheduling.md) | [Part 24: Docker Deployment →](./part-24-docker-deployment.md)*

# Part 37: OpenTelemetry & Distributed Tracing
## ขั้นตอนที่ 1086-1125

> **ระดับ:** ระดับโลก (World-Class)  
> **เวลาเรียน:** 5-6 ชั่วโมง  
> **เป้าหมาย:** Observability ครบ 3 เสาหลัก: Metrics, Traces, Logs

---

## ขั้นตอนที่ 1086: Observability Pillars

```
The Three Pillars of Observability:

1. Metrics = What is happening? (Prometheus/Grafana)
   - CPU, Memory, Request rate, Error rate, Latency
   
2. Traces = Where does time go? (Zipkin/Jaeger)
   - End-to-end request flow across services
   - Span tree showing bottlenecks
   
3. Logs = What happened? (ELK Stack)
   - Detailed events with context

OpenTelemetry = Standard for all three
  - Vendor-neutral
  - Spring Boot 3.x has built-in support
  - One SDK for all signals
```

---

## ขั้นตอนที่ 1087: Dependencies

```xml
<!-- Spring Boot Actuator (metrics + health) -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>

<!-- Micrometer for metrics -->
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>

<!-- OpenTelemetry tracing -->
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-otel</artifactId>
</dependency>

<dependency>
    <groupId>io.opentelemetry</groupId>
    <artifactId>opentelemetry-exporter-otlp</artifactId>
</dependency>

<!-- Zipkin (alternative, simpler) -->
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-brave</artifactId>
</dependency>
<dependency>
    <groupId>io.zipkin.reporter2</groupId>
    <artifactId>zipkin-reporter-brave</artifactId>
</dependency>
```

---

## ขั้นตอนที่ 1088: Configuration

```yaml
spring:
  application:
    name: product-service  # Service name in traces
  
  # Tracing
  zipkin:
    base-url: http://zipkin:9411
  sleuth:
    sampler:
      probability: 1.0  # Trace 100% of requests (use 0.1 in production)

management:
  endpoints:
    web:
      exposure:
        include: health, metrics, prometheus, info, trace
  
  metrics:
    export:
      prometheus:
        enabled: true
    tags:
      application: ${spring.application.name}
      environment: ${spring.profiles.active:default}
  
  tracing:
    sampling:
      probability: 1.0
    
  # OpenTelemetry endpoint (if using OTEL Collector)
  otlp:
    tracing:
      endpoint: http://otel-collector:4318/v1/traces
    metrics:
      endpoint: http://otel-collector:4318/v1/metrics
```

---

## ขั้นตอนที่ 1089: Custom Metrics

```java
@Component
@RequiredArgsConstructor
public class BusinessMetrics {
    
    private final MeterRegistry registry;
    
    // Counter: total orders created
    private Counter orderCreatedCounter;
    
    // Gauge: current active sessions
    private AtomicInteger activeSessions = new AtomicInteger(0);
    
    // Timer: payment processing duration
    private Timer paymentTimer;
    
    // Distribution summary: order amounts
    private DistributionSummary orderAmountSummary;
    
    @PostConstruct
    void initMetrics() {
        orderCreatedCounter = Counter.builder("orders.created.total")
            .description("Total orders created")
            .tag("service", "order-service")
            .register(registry);
        
        Gauge.builder("sessions.active", activeSessions, AtomicInteger::get)
            .description("Number of active sessions")
            .register(registry);
        
        paymentTimer = Timer.builder("payment.processing.duration")
            .description("Payment processing time")
            .publishPercentiles(0.5, 0.95, 0.99)
            .register(registry);
        
        orderAmountSummary = DistributionSummary.builder("order.amount")
            .description("Order amount distribution")
            .baseUnit("THB")
            .publishPercentiles(0.5, 0.75, 0.90, 0.95, 0.99)
            .register(registry);
    }
    
    public void recordOrderCreated(BigDecimal amount) {
        orderCreatedCounter.increment();
        orderAmountSummary.record(amount.doubleValue());
    }
    
    public Timer.Sample startPayment() {
        return Timer.start(registry);
    }
    
    public void stopPayment(Timer.Sample sample, boolean success) {
        sample.stop(Timer.builder("payment.processing.duration")
            .tag("status", success ? "success" : "failed")
            .register(registry));
    }
    
    public void sessionStarted() { activeSessions.incrementAndGet(); }
    public void sessionEnded() { activeSessions.decrementAndGet(); }
}

// Usage in service
@Service
@RequiredArgsConstructor
public class OrderService {
    
    private final BusinessMetrics metrics;
    
    public OrderResponse create(CreateOrderRequest request) {
        Order order = // ... create order
        
        metrics.recordOrderCreated(order.getTotalAmount());
        
        return orderMapper.toResponse(order);
    }
    
    public PaymentResponse processPayment(Long orderId) {
        Timer.Sample sample = metrics.startPayment();
        boolean success = false;
        
        try {
            PaymentResponse result = // process payment
            success = true;
            return result;
        } finally {
            metrics.stopPayment(sample, success);
        }
    }
}
```

---

## ขั้นตอนที่ 1090: Custom Spans (Distributed Tracing)

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class ProductService {
    
    private final Tracer tracer;
    private final ProductRepository productRepository;
    
    public ProductResponse findById(Long id) {
        // Create custom span
        Span span = tracer.nextSpan().name("find-product-by-id");
        
        try (Tracer.SpanInScope ignored = tracer.withSpan(span.start())) {
            span.tag("product.id", String.valueOf(id));
            
            Product product = productRepository.findById(id)
                .orElseThrow(() -> new ResourceNotFoundException("Product not found: " + id));
            
            span.tag("product.name", product.getName());
            span.event("product.found");
            
            return productMapper.toResponse(product);
            
        } catch (Exception e) {
            span.tag("error", e.getMessage());
            throw e;
        } finally {
            span.end();
        }
    }
    
    // Automatic span creation with @NewSpan
    @NewSpan("search-products")
    public Page<ProductResponse> search(SearchRequest request) {
        // Spring auto creates span
        return // ... search logic
    }
}
```

---

## ขั้นตอนที่ 1091: Structured Logging with Trace Context

```java
// MDC (Mapped Diagnostic Context) with trace IDs
@Component
public class TraceIdFilter implements Filter {
    
    @Override
    public void doFilter(ServletRequest req, ServletResponse resp, FilterChain chain)
        throws IOException, ServletException {
        
        // Spring Sleuth automatically adds traceId, spanId to MDC
        // We can add custom fields
        String correlationId = ((HttpServletRequest) req).getHeader("X-Correlation-ID");
        if (correlationId != null) {
            MDC.put("correlationId", correlationId);
        }
        
        try {
            chain.doFilter(req, resp);
        } finally {
            MDC.clear();
        }
    }
}

// logback-spring.xml
<configuration>
    <appender name="JSON" class="ch.qos.logback.core.ConsoleAppender">
        <encoder class="net.logstash.logback.encoder.LogstashEncoder">
            <includeMdcKeyNames>traceId</includeMdcKeyNames>
            <includeMdcKeyNames>spanId</includeMdcKeyNames>
            <includeMdcKeyNames>correlationId</includeMdcKeyNames>
        </encoder>
    </appender>
    
    <root level="INFO">
        <appender-ref ref="JSON"/>
    </root>
</configuration>
```

---

## ขั้นตอนที่ 1092: Grafana Dashboard

```yaml
# docker-compose.monitoring.yml
services:
  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
      - "9090:9090"

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    environment:
      GF_SECURITY_ADMIN_PASSWORD: admin
    volumes:
      - grafana-data:/var/lib/grafana
      - ./grafana/dashboards:/etc/grafana/dashboards
      - ./grafana/provisioning:/etc/grafana/provisioning

  zipkin:
    image: openzipkin/zipkin:latest
    ports:
      - "9411:9411"

  # OpenTelemetry Collector (optional)
  otel-collector:
    image: otel/opentelemetry-collector-contrib:latest
    volumes:
      - ./otel-config.yml:/etc/otel/config.yml
    command: ["--config=/etc/otel/config.yml"]
    ports:
      - "4318:4318"
      - "8888:8888"

volumes:
  grafana-data:
```

```yaml
# prometheus.yml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'spring-boot-services'
    metrics_path: /actuator/prometheus
    static_configs:
      - targets: 
          - 'product-service:8080'
          - 'order-service:8081'
          - 'user-service:8082'
    
    # In K8s - use kubernetes_sd_configs instead
  
  - job_name: 'spring-boot-k8s'
    kubernetes_sd_configs:
      - role: pod
        namespaces:
          names: ['myapp']
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
```

---

## ขั้นตอนที่ 1093-1125: Alerting

```yaml
# alerting rules
groups:
  - name: spring-boot
    rules:
      - alert: HighErrorRate
        expr: rate(http_server_requests_seconds_count{status=~"5.."}[5m]) > 0.1
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "High error rate on {{ $labels.application }}"
          description: "Error rate is {{ $value }}"
      
      - alert: SlowRequests
        expr: histogram_quantile(0.99, rate(http_server_requests_seconds_bucket[5m])) > 2
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Slow P99 latency"
          description: "P99 latency is {{ $value }}s"
      
      - alert: HighMemory
        expr: jvm_memory_used_bytes{area="heap"} / jvm_memory_max_bytes{area="heap"} > 0.9
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High JVM heap usage"
          description: "Heap usage is {{ $value | humanizePercentage }}"
      
      - alert: ServiceDown
        expr: up{job="spring-boot-services"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Service {{ $labels.instance }} is down"
```

---

*[← Part 36: Kubernetes](./part-36-kubernetes.md) | [Part 38: Event Sourcing →](./part-38-event-sourcing.md)*

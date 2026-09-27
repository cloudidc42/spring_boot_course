# Part 63: Distributed Tracing
## ขั้นตอนที่ 2121-2160

**ระดับ:** ระดับโลก (World-Class)
**เวลาเรียน:** 4-5 ชั่วโมง
**เป้าหมาย:** Implement Distributed Tracing ครบวงจรด้วย Micrometer Tracing, Zipkin, Jaeger และ OpenTelemetry

---

## บทนำ

Distributed Tracing คือ Observability Technique ที่ช่วยให้เราติดตาม Request ที่เดินทางผ่าน Services หลายตัวได้ ตอบคำถามสำคัญเช่น "Request นี้ช้าเพราะ Service ไหน?" และ "Error เกิดที่จุดใด?"

```
Client -> API Gateway -> Order Service -> Inventory Service
                     -> Payment Service -> External Payment Gateway
                     -> Notification Service
```

ทุก Request จะมี TraceId เดียวกันตลอด และแต่ละ Service จะสร้าง SpanId ของตัวเอง

---

## ขั้นตอนที่ 2121: Setup Dependencies

```xml
<!-- pom.xml -->
<dependencies>
    <!-- Micrometer Tracing Core -->
    <dependency>
        <groupId>io.micrometer</groupId>
        <artifactId>micrometer-tracing-bridge-brave</artifactId>
    </dependency>
    
    <!-- Zipkin Reporter -->
    <dependency>
        <groupId>io.zipkin.reporter2</groupId>
        <artifactId>zipkin-reporter-brave</artifactId>
    </dependency>
    
    <!-- OpenTelemetry (alternative to Brave) -->
    <dependency>
        <groupId>io.micrometer</groupId>
        <artifactId>micrometer-tracing-bridge-otel</artifactId>
    </dependency>
    
    <!-- OTLP Exporter สำหรับ Jaeger/OpenTelemetry Collector -->
    <dependency>
        <groupId>io.opentelemetry</groupId>
        <artifactId>opentelemetry-exporter-otlp</artifactId>
    </dependency>
    
    <!-- Prometheus Metrics -->
    <dependency>
        <groupId>io.micrometer</groupId>
        <artifactId>micrometer-registry-prometheus</artifactId>
    </dependency>
    
    <!-- Actuator -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
</dependencies>
```

```yaml
# application.yml - Tracing Configuration
spring:
  application:
    name: order-service

management:
  tracing:
    sampling:
      probability: 1.0  # Sample 100% ใน Dev, ลดเหลือ 0.1 ใน Prod
  zipkin:
    tracing:
      endpoint: http://localhost:9411/api/v2/spans
  otlp:
    tracing:
      endpoint: http://localhost:4318/v1/traces
  
  endpoints:
    web:
      exposure:
        include: health, info, metrics, prometheus, traces
```

---

## ขั้นตอนที่ 2122-2125: Basic Tracing Setup

```java
// TracingConfig.java
@Configuration
public class TracingConfig {
    
    @Bean
    public Sampler defaultSampler() {
        // Sample ทุก Request ใน Development
        return Sampler.ALWAYS_SAMPLE;
    }
    
    @Bean
    public SpanCustomizer spanCustomizer(Tracer tracer) {
        return tracer.currentSpanCustomizer();
    }
    
    // HTTP Client tracing สำหรับ RestTemplate
    @Bean
    @LoadBalanced
    public RestTemplate tracedRestTemplate(
            HttpTracing httpTracing) {
        return new RestTemplateBuilder()
            .additionalInterceptors(TracingClientHttpRequestInterceptor
                .create(httpTracing))
            .build();
    }
    
    // WebClient tracing
    @Bean
    public WebClient tracedWebClient(
            WebClient.Builder builder,
            HttpTracing httpTracing) {
        return builder
            .filter(new TracingExchangeFilterFunction(httpTracing))
            .build();
    }
}
```

```java
// OrderService.java - Service ที่ใช้ Tracing
@Service
@RequiredArgsConstructor
@Slf4j
public class OrderService {
    
    private final Tracer tracer;
    private final OrderRepository orderRepository;
    private final InventoryClient inventoryClient;
    private final PaymentClient paymentClient;
    
    public Order createOrder(CreateOrderRequest request) {
        // Span ถูกสร้างอัตโนมัติโดย Spring AOP
        // แต่เราสามารถเพิ่ม Custom Spans ได้
        
        Span orderValidationSpan = tracer.nextSpan()
            .name("validate-order")
            .start();
        
        try (Tracer.SpanInScope scope = tracer.withSpan(orderValidationSpan)) {
            // เพิ่ม Tags เพื่อ Context
            orderValidationSpan.tag("order.customerId", request.getCustomerId());
            orderValidationSpan.tag("order.itemCount", 
                                    String.valueOf(request.getItems().size()));
            
            validateOrder(request);
            
        } finally {
            orderValidationSpan.end();
        }
        
        // ตรวจสอบ Inventory
        InventoryCheckResult inventoryResult = inventoryClient
            .checkAvailability(request.getItems());
        
        // Process Payment
        PaymentResult payment = paymentClient.charge(request.getPayment());
        
        // สร้าง Order
        Order order = Order.from(request, payment.getTransactionId());
        Order saved = orderRepository.save(order);
        
        // Log ที่มี Trace Context
        log.info("Order created: {} for customer: {}", 
                 saved.getId(), saved.getCustomerId());
        
        return saved;
    }
    
    @NewSpan("validate-order-items")  // สร้าง Child Span อัตโนมัติ
    public void validateOrderItems(@SpanTag("itemCount") List<OrderItem> items) {
        // Business validation logic
        if (items == null || items.isEmpty()) {
            throw new ValidationException("Order must have at least one item");
        }
        
        items.forEach(item -> {
            if (item.getQuantity() <= 0) {
                throw new ValidationException(
                    "Invalid quantity for product: " + item.getProductId());
            }
        });
    }
}
```

---

## ขั้นตอนที่ 2126-2130: Custom Spans และ Events

```java
// CustomTracingService.java - การใช้ Custom Spans
@Service
@RequiredArgsConstructor
@Slf4j
public class CustomTracingService {
    
    private final Tracer tracer;
    
    // วิธีที่ 1: Manual Span Creation
    public ProcessingResult processWithTrace(String data) {
        Span span = tracer.nextSpan()
            .name("data-processing")
            .tag("data.size", String.valueOf(data.length()))
            .tag("data.type", "user-input")
            .start();
        
        try (Tracer.SpanInScope scope = tracer.withSpan(span)) {
            log.info("Processing started for data: {}", 
                     data.substring(0, Math.min(50, data.length())));
            
            // Step 1: Parse
            Span parseSpan = tracer.nextSpan().name("parse-data").start();
            try (Tracer.SpanInScope parseScope = tracer.withSpan(parseSpan)) {
                ParsedData parsed = parseData(data);
                parseSpan.tag("parsed.records", String.valueOf(parsed.getCount()));
                
                // Step 2: Validate
                Span validateSpan = tracer.nextSpan().name("validate-data").start();
                try (Tracer.SpanInScope validateScope = tracer.withSpan(validateSpan)) {
                    validateData(parsed);
                } finally {
                    validateSpan.end();
                }
                
                // Step 3: Transform
                return transformData(parsed);
                
            } catch (Exception e) {
                span.tag("error", e.getMessage());
                throw e;
            } finally {
                parseSpan.end();
            }
            
        } finally {
            span.end();
        }
    }
    
    // วิธีที่ 2: Annotation-based Spans
    @NewSpan("external-api-call")
    public ExternalData callExternalApi(
            @SpanTag("api.endpoint") String endpoint,
            @SpanTag("api.version") String version) {
        
        // ดึงข้อมูล Span ปัจจุบัน
        Span currentSpan = tracer.currentSpan();
        if (currentSpan != null) {
            currentSpan.event("external-api-started");
        }
        
        try {
            ExternalData result = externalApiClient.call(endpoint, version);
            
            if (currentSpan != null) {
                currentSpan.tag("api.response.size", 
                               String.valueOf(result.getSize()));
                currentSpan.event("external-api-completed");
            }
            
            return result;
        } catch (Exception e) {
            if (currentSpan != null) {
                currentSpan.tag("error.type", e.getClass().getSimpleName());
                currentSpan.tag("error.message", e.getMessage());
                currentSpan.event("external-api-failed");
            }
            throw e;
        }
    }
    
    // วิธีที่ 3: Async Tracing
    @Async
    @NewSpan("async-processing")
    public CompletableFuture<Void> processAsync(
            @SpanTag("job.id") String jobId) {
        
        return CompletableFuture.runAsync(() -> {
            // Trace Context ถูก Propagate ไปใน Async Thread อัตโนมัติ
            log.info("Processing async job: {}", jobId);
            doHeavyWork(jobId);
        });
    }
}
```

---

## ขั้นตอนที่ 2131-2135: Baggage Propagation

### อธิบาย

Baggage คือข้อมูลที่ส่งผ่านตลอด Trace Chain ใช้สำหรับ Context ที่ต้องการใน Services ทั้งหมด เช่น Tenant ID, Feature Flag, Debug Mode

```java
// BaggageConfig.java
@Configuration
public class BaggageConfig {
    
    @Bean
    public BaggageField tenantIdField() {
        return BaggageField.create("tenant-id");
    }
    
    @Bean
    public BaggageField correlationIdField() {
        return BaggageField.create("correlation-id");
    }
    
    @Bean
    public BaggageField featureFlagField() {
        return BaggageField.create("feature-flag");
    }
    
    @Bean
    public CorrelationFields correlationFields(
            BaggageField tenantIdField,
            BaggageField correlationIdField) {
        // กำหนดว่า Baggage Field ไหนจะแสดงใน Logs (MDC)
        return CorrelationFields.create(tenantIdField, correlationIdField);
    }
}
```

```java
// BaggageService.java - การใช้ Baggage
@Service
@RequiredArgsConstructor
@Slf4j
public class BaggageService {
    
    private final BaggageField tenantIdField;
    private final BaggageField correlationIdField;
    private final Tracer tracer;
    
    // กำหนด Baggage ใน Entry Point (Gateway/Controller)
    public void setRequestContext(String tenantId, String correlationId) {
        // Baggage จะถูก Propagate ไปทุก Service ที่ Request นี้ผ่าน
        tenantIdField.updateValue(tenantId);
        correlationIdField.updateValue(correlationId);
        
        log.info("Set baggage - tenantId: {}, correlationId: {}", 
                 tenantId, correlationId);
    }
    
    // อ่าน Baggage ใน Service ใดก็ได้
    public String getCurrentTenantId() {
        return tenantIdField.getValue();
    }
    
    public String getCurrentCorrelationId() {
        return correlationIdField.getValue();
    }
}
```

```java
// TenantAwareFilter.java - ดึง Tenant จาก JWT แล้วใส่ใน Baggage
@Component
@RequiredArgsConstructor
@Order(1)
public class TenantAwareFilter implements Filter {
    
    private final BaggageField tenantIdField;
    
    @Override
    public void doFilter(ServletRequest request, ServletResponse response, 
                         FilterChain chain) throws IOException, ServletException {
        
        HttpServletRequest httpRequest = (HttpServletRequest) request;
        String tenantId = extractTenantFromJwt(httpRequest);
        
        if (tenantId != null) {
            tenantIdField.updateValue(tenantId);
        }
        
        chain.doFilter(request, response);
    }
    
    private String extractTenantFromJwt(HttpServletRequest request) {
        String auth = request.getHeader("Authorization");
        if (auth != null && auth.startsWith("Bearer ")) {
            // Extract tenant from JWT claims
            return "tenant-123"; // Simplified
        }
        return null;
    }
}
```

---

## ขั้นตอนที่ 2136-2140: Trace Context ใน Logs (MDC)

### อธิบาย

MDC (Mapped Diagnostic Context) ช่วยให้เราเพิ่ม Trace ID เข้าไปใน Log Records ทุกบรรทัดโดยอัตโนมัติ ทำให้ Search Logs โดย Trace ID ได้

```xml
<!-- logback-spring.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    
    <springProperty scope="context" name="APP_NAME" source="spring.application.name"/>
    
    <!-- Console Appender แบบ JSON -->
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder class="net.logstash.logback.encoder.LogstashEncoder">
            <includeMdcKeyName>traceId</includeMdcKeyName>
            <includeMdcKeyName>spanId</includeMdcKeyName>
            <includeMdcKeyName>tenant-id</includeMdcKeyName>
            <includeMdcKeyName>correlation-id</includeMdcKeyName>
            <customFields>{"service":"${APP_NAME}"}</customFields>
        </encoder>
    </appender>
    
    <!-- File Appender สำหรับ Production -->
    <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>/var/log/app/${APP_NAME}.log</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>/var/log/app/${APP_NAME}.%d{yyyy-MM-dd}.log.gz</fileNamePattern>
            <maxHistory>30</maxHistory>
        </rollingPolicy>
        <encoder class="net.logstash.logback.encoder.LogstashEncoder">
            <includeMdcKeyName>traceId</includeMdcKeyName>
            <includeMdcKeyName>spanId</includeMdcKeyName>
            <includeMdcKeyName>tenant-id</includeMdcKeyName>
        </encoder>
    </appender>
    
    <root level="INFO">
        <appender-ref ref="CONSOLE"/>
        <appender-ref ref="FILE"/>
    </root>
    
</configuration>
```

```java
// TracedController.java - Controller ที่มี Trace ใน Log อัตโนมัติ
@RestController
@RequestMapping("/api/orders")
@RequiredArgsConstructor
@Slf4j
public class TracedController {
    
    private final OrderService orderService;
    private final Tracer tracer;
    
    @PostMapping
    public ResponseEntity<OrderDto> createOrder(
            @RequestBody @Valid CreateOrderRequest request) {
        
        // Micrometer จะเพิ่ม traceId และ spanId ใน MDC อัตโนมัติ
        // ทุก log.info() ที่เรียกจากนี้จะมี traceId แนบมาด้วย
        
        log.info("Creating order for customer: {}", request.getCustomerId());
        
        // ดึง Trace Info สำหรับ Response Header
        Span currentSpan = tracer.currentSpan();
        String traceId = currentSpan != null ? 
            currentSpan.context().traceId() : "unknown";
        
        Order order = orderService.createOrder(request);
        
        log.info("Order created successfully: {}", order.getId());
        
        return ResponseEntity.ok()
            .header("X-Trace-Id", traceId)  // ส่ง Trace ID กลับไปให้ Client
            .body(OrderDto.from(order));
    }
    
    @GetMapping("/{orderId}")
    public ResponseEntity<OrderDto> getOrder(@PathVariable String orderId) {
        log.info("Fetching order: {}", orderId);  // Log นี้มี traceId อัตโนมัติ
        
        return orderService.findById(orderId)
            .map(order -> {
                log.info("Order found: {}", orderId);
                return ResponseEntity.ok(OrderDto.from(order));
            })
            .orElseGet(() -> {
                log.warn("Order not found: {}", orderId);
                return ResponseEntity.notFound().build();
            });
    }
}
```

---

## ขั้นตอนที่ 2141-2145: Sampling Strategies

### อธิบาย

Sampling กำหนดว่า % ของ Request ที่จะถูก Trace ใน Production เราไม่ต้อง Trace ทุก Request เพราะจะใช้ Resource มาก

```java
// SamplingConfig.java
@Configuration
public class SamplingConfig {
    
    // 1. Percentage Sampler (ง่ายสุด)
    @Bean
    @Profile("production")
    public Sampler productionSampler() {
        return Sampler.create(0.1f);  // Sample 10% ใน Production
    }
    
    @Bean
    @Profile("development")
    public Sampler devSampler() {
        return Sampler.ALWAYS_SAMPLE;  // Sample 100% ใน Dev
    }
    
    // 2. Rate-Limited Sampler (ดีกว่า Percentage)
    // Sample N requests/second โดยไม่สนใจ Traffic
    @Bean
    @Profile("staging")
    public Sampler rateLimitedSampler() {
        return RateLimitingSampler.create(100); // 100 traces/second
    }
}
```

```java
// PrioritySamplingDecider.java - Sampling ตาม Priority
@Component
@Slf4j
public class PrioritySamplingDecider {
    
    private final Set<String> alwaysTraceEndpoints = Set.of(
        "/api/payments/",
        "/api/orders/",
        "/api/auth/"
    );
    
    private final Set<String> neverTraceEndpoints = Set.of(
        "/actuator/",
        "/api/health"
    );
    
    // กำหนด Sampling Decision ตาม Endpoint และ Context
    public SamplingDecision decide(String path, Map<String, String> headers) {
        // ไม่ trace endpoints ที่ไม่สำคัญ
        if (neverTraceEndpoints.stream().anyMatch(path::startsWith)) {
            return SamplingDecision.NOT_SAMPLE;
        }
        
        // trace เสมอสำหรับ Critical Endpoints
        if (alwaysTraceEndpoints.stream().anyMatch(path::startsWith)) {
            return SamplingDecision.SAMPLE;
        }
        
        // Force trace จาก Debug Header
        if ("true".equals(headers.get("X-Debug-Trace"))) {
            return SamplingDecision.SAMPLE;
        }
        
        // Force trace จาก Error Header  
        if (headers.containsKey("X-Error-Trace")) {
            return SamplingDecision.SAMPLE;
        }
        
        // Default: ใช้ Probability Sampler
        return SamplingDecision.DEFAULT;
    }
    
    public enum SamplingDecision {
        SAMPLE, NOT_SAMPLE, DEFAULT
    }
}
```

---

## ขั้นตอนที่ 2146-2150: Zipkin Integration

### Setup Zipkin

```yaml
# docker-compose.yml
version: '3.8'
services:
  zipkin:
    image: openzipkin/zipkin:latest
    ports:
      - "9411:9411"
    environment:
      - STORAGE_TYPE=elasticsearch
      - ES_HOSTS=http://elasticsearch:9200
    depends_on:
      - elasticsearch
  
  elasticsearch:
    image: elasticsearch:8.11.0
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
    ports:
      - "9200:9200"
    volumes:
      - es-data:/usr/share/elasticsearch/data

volumes:
  es-data:
```

```yaml
# application.yml สำหรับ Zipkin
management:
  zipkin:
    tracing:
      endpoint: http://zipkin:9411/api/v2/spans
      connect-timeout: 1s
      read-timeout: 10s
  tracing:
    sampling:
      probability: 1.0
```

```java
// ZipkinConfig.java - Custom Zipkin Reporter
@Configuration
public class ZipkinConfig {
    
    @Bean
    public Reporter<Span> zipkinReporter(Sender sender) {
        return AsyncReporter.builder(sender)
            .messageTimeout(1, TimeUnit.SECONDS)
            .queuedMaxBytes(10 * 1024 * 1024) // 10MB buffer
            .build(SpanBytesEncoder.JSON_V2);
    }
    
    @Bean
    public Sender zipkinSender(
            @Value("${management.zipkin.tracing.endpoint}") String endpoint) {
        return OkHttpSender.create(endpoint);
    }
}
```

---

## ขั้นตอนที่ 2151-2155: Jaeger Integration

```yaml
# docker-compose-jaeger.yml
version: '3.8'
services:
  jaeger:
    image: jaegertracing/all-in-one:latest
    ports:
      - "16686:16686"   # Jaeger UI
      - "4317:4317"     # OTLP gRPC
      - "4318:4318"     # OTLP HTTP
      - "14268:14268"   # Jaeger HTTP Thrift
    environment:
      - COLLECTOR_OTLP_ENABLED=true
      - SPAN_STORAGE_TYPE=memory
```

```yaml
# application.yml สำหรับ Jaeger ด้วย OTLP
management:
  otlp:
    tracing:
      endpoint: http://jaeger:4318/v1/traces
      headers:
        - "x-service-name=order-service"
  tracing:
    sampling:
      probability: 1.0
```

```java
// JaegerOtelConfig.java - OpenTelemetry + Jaeger
@Configuration
public class JaegerOtelConfig {
    
    @Value("${spring.application.name}")
    private String serviceName;
    
    @Value("${management.otlp.tracing.endpoint}")
    private String jaegerEndpoint;
    
    @Bean
    public OpenTelemetry openTelemetry() {
        Resource resource = Resource.getDefault().toBuilder()
            .put(ResourceAttributes.SERVICE_NAME, serviceName)
            .put(ResourceAttributes.SERVICE_VERSION, "1.0.0")
            .put(ResourceAttributes.DEPLOYMENT_ENVIRONMENT, "production")
            .build();
        
        OtlpHttpSpanExporter exporter = OtlpHttpSpanExporter.builder()
            .setEndpoint(jaegerEndpoint)
            .setTimeout(Duration.ofSeconds(10))
            .addHeader("x-service-name", serviceName)
            .build();
        
        SdkTracerProvider tracerProvider = SdkTracerProvider.builder()
            .addSpanProcessor(BatchSpanProcessor.builder(exporter)
                .setScheduleDelay(Duration.ofMillis(100))
                .setMaxExportBatchSize(512)
                .build())
            .setResource(resource)
            .setSampler(Sampler.traceIdRatioBased(0.1)) // 10% sampling
            .build();
        
        return OpenTelemetrySdk.builder()
            .setTracerProvider(tracerProvider)
            .buildAndRegisterGlobal();
    }
}
```

---

## ขั้นตอนที่ 2156-2160: OpenTelemetry Collector Setup

### อธิบาย

OpenTelemetry Collector เป็น Middleware ที่รับ Telemetry Data (Traces, Metrics, Logs) จาก Applications แล้วส่งต่อไปยัง Backend หลายตัวพร้อมกัน

```yaml
# otel-collector-config.yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  batch:
    timeout: 1s
    send_batch_size: 1024
  
  # Filter ไม่เอา Health Check Spans
  filter:
    error_mode: ignore
    traces:
      span:
        - 'attributes["http.target"] == "/actuator/health"'
        - 'attributes["http.target"] == "/actuator/prometheus"'
  
  # เพิ่ม Resource Attributes
  resource:
    attributes:
      - key: environment
        value: production
        action: upsert
  
  # Memory Limiter ป้องกัน OOM
  memory_limiter:
    check_interval: 1s
    limit_mib: 1000
    spike_limit_mib: 200

exporters:
  # ส่งไป Zipkin
  zipkin:
    endpoint: http://zipkin:9411/api/v2/spans
  
  # ส่งไป Jaeger
  otlp/jaeger:
    endpoint: jaeger:4317
    tls:
      insecure: true
  
  # ส่งไป Grafana Tempo
  otlp/tempo:
    endpoint: tempo:4317
    tls:
      insecure: true
  
  # Debug output (ปิดใน Production)
  logging:
    verbosity: detailed

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, filter, resource, batch]
      exporters: [zipkin, otlp/jaeger]
    
    metrics:
      receivers: [otlp]
      processors: [memory_limiter, resource, batch]
      exporters: [otlp/tempo]
```

```yaml
# docker-compose-otel.yml
version: '3.8'
services:
  otel-collector:
    image: otel/opentelemetry-collector-contrib:latest
    command: ["--config=/etc/otel-config.yaml"]
    volumes:
      - ./otel-collector-config.yaml:/etc/otel-config.yaml
    ports:
      - "4317:4317"   # OTLP gRPC
      - "4318:4318"   # OTLP HTTP
      - "8888:8888"   # Prometheus metrics สำหรับ Collector เอง
    depends_on:
      - zipkin
      - jaeger
```

```java
// TraceAnalysisService.java - Programmatic Trace Analysis
@Service
@RequiredArgsConstructor
@Slf4j
public class TraceAnalysisService {
    
    private final ZipkinClient zipkinClient;
    private final MeterRegistry meterRegistry;
    
    // ดึง Traces ที่ Slow เพื่อ Performance Analysis
    public List<TraceDto> findSlowTraces(
            String serviceName, 
            Duration threshold, 
            Instant from,
            Instant to) {
        
        return zipkinClient.getTraces(
            serviceName,
            from.toEpochMilli() * 1000,
            to.toEpochMilli() * 1000,
            100
        ).stream()
            .filter(trace -> {
                long maxDuration = trace.getSpans().stream()
                    .mapToLong(Span::getDuration)
                    .max()
                    .orElse(0);
                return maxDuration > threshold.toMicros();
            })
            .collect(Collectors.toList());
    }
    
    // หา Bottleneck ใน Request Chain
    public BottleneckAnalysis analyzeBottleneck(String traceId) {
        List<Span> spans = zipkinClient.getTrace(traceId);
        
        Map<String, Long> serviceDurations = spans.stream()
            .collect(Collectors.groupingBy(
                Span::getServiceName,
                Collectors.summingLong(Span::getDuration)
            ));
        
        String bottleneckService = serviceDurations.entrySet().stream()
            .max(Map.Entry.comparingByValue())
            .map(Map.Entry::getKey)
            .orElse("unknown");
        
        long totalDuration = spans.stream()
            .filter(s -> s.getParentId() == null) // Root span
            .mapToLong(Span::getDuration)
            .sum();
        
        return BottleneckAnalysis.builder()
            .traceId(traceId)
            .bottleneckService(bottleneckService)
            .serviceDurations(serviceDurations)
            .totalDurationMs(totalDuration / 1000)
            .build();
    }
    
    // Record Trace Metrics สำหรับ SLO Monitoring
    @Scheduled(fixedRate = 60000)  // ทุก 1 นาที
    public void recordTraceMetrics() {
        Instant now = Instant.now();
        Instant oneMinuteAgo = now.minus(Duration.ofMinutes(1));
        
        List<TraceDto> recentTraces = zipkinClient.getTraces(
            "order-service",
            oneMinuteAgo.toEpochMilli() * 1000,
            now.toEpochMilli() * 1000,
            1000
        );
        
        long errorCount = recentTraces.stream()
            .filter(t -> t.getSpans().stream()
                .anyMatch(s -> s.getTags().containsKey("error")))
            .count();
        
        double avgDurationMs = recentTraces.stream()
            .mapToLong(t -> t.getSpans().get(0).getDuration())
            .average()
            .orElse(0) / 1000;
        
        meterRegistry.gauge("trace.error.count", errorCount);
        meterRegistry.gauge("trace.avg.duration.ms", avgDurationMs);
        
        if ((double) errorCount / recentTraces.size() > 0.05) {
            log.error("High error rate detected: {}/{} traces have errors",
                     errorCount, recentTraces.size());
        }
    }
}
```

---

## Performance Debugging ด้วย Traces

```java
// PerformanceTracingAspect.java - AOP สำหรับ Performance Monitoring
@Aspect
@Component
@Slf4j
public class PerformanceTracingAspect {
    
    private final Tracer tracer;
    private final MeterRegistry meterRegistry;
    
    // Trace ทุก Repository call อัตโนมัติ
    @Around("execution(* com.example.*.repository.*Repository.*(..))")
    public Object traceRepositoryCall(ProceedingJoinPoint joinPoint) throws Throwable {
        String methodName = joinPoint.getSignature().getName();
        String className = joinPoint.getTarget().getClass().getSimpleName();
        
        Span span = tracer.nextSpan()
            .name("db." + className + "." + methodName)
            .tag("db.operation", methodName)
            .tag("db.repository", className)
            .start();
        
        long start = System.nanoTime();
        
        try (Tracer.SpanInScope scope = tracer.withSpan(span)) {
            Object result = joinPoint.proceed();
            
            long duration = System.nanoTime() - start;
            span.tag("db.duration_ms", String.valueOf(duration / 1_000_000));
            
            // Record Metric
            meterRegistry.timer("db.operation.duration",
                "repository", className,
                "operation", methodName)
                .record(Duration.ofNanos(duration));
            
            // Warning ถ้า Query ช้า
            if (duration > 1_000_000_000L) { // > 1 second
                log.warn("Slow DB operation: {}.{} took {}ms",
                         className, methodName, duration / 1_000_000);
                span.tag("db.slow", "true");
            }
            
            return result;
        } catch (Exception e) {
            span.tag("error", e.getMessage());
            throw e;
        } finally {
            span.end();
        }
    }
    
    // Trace ทุก External API call
    @Around("@annotation(com.example.tracing.TraceExternalCall)")
    public Object traceExternalCall(ProceedingJoinPoint joinPoint) throws Throwable {
        TraceExternalCall annotation = ((MethodSignature) joinPoint.getSignature())
            .getMethod()
            .getAnnotation(TraceExternalCall.class);
        
        Span span = tracer.nextSpan()
            .name("external." + annotation.name())
            .tag("external.service", annotation.service())
            .start();
        
        try (Tracer.SpanInScope scope = tracer.withSpan(span)) {
            return joinPoint.proceed();
        } catch (Exception e) {
            span.tag("error", e.getMessage());
            throw e;
        } finally {
            span.end();
        }
    }
}
```

```java
// @TraceExternalCall Annotation
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface TraceExternalCall {
    String name();
    String service() default "unknown";
}
```

---

## Grafana Dashboard Integration

```yaml
# grafana-datasource.yaml
apiVersion: 1
datasources:
  - name: Tempo
    type: tempo
    url: http://tempo:3200
    jsonData:
      httpMethod: GET
      tracesToLogsV2:
        datasourceUid: loki
        spanStartTimeShift: '-1h'
        spanEndTimeShift: '1h'
        filterByTraceID: true
        filterBySpanID: false
        customQuery: true
        query: '{service_name="${__span.tags.service.name}"} |= "${__trace.traceId}"'
      serviceMap:
        datasourceUid: prometheus
  
  - name: Loki
    type: loki
    url: http://loki:3100
    jsonData:
      derivedFields:
        - matcherRegex: '"traceId":"(\w+)"'
          name: TraceID
          url: '$${__value.raw}'
          datasourceUid: tempo
```

---

## สรุป Distributed Tracing

| Component | เมื่อใช้ | ประโยชน์ |
|-----------|---------|---------|
| Zipkin | Simple setup, แสดงผลง่าย | Lightweight, UI ดี |
| Jaeger | Enterprise, Kubernetes | Scale ได้, Kubernetes-native |
| OTEL Collector | หลาย Backend | Vendor-neutral, Flexible |
| Custom Spans | Complex Business Logic | Deep insight |
| Baggage | Cross-service Context | Request context propagation |
| MDC | Log correlation | Search ด้วย TraceId |

---

*[← Part 62: API Gateway Advanced](./part-62-api-gateway-advanced.md) | [Part 64: Resilience Patterns →](./part-64-resilience-patterns.md)*

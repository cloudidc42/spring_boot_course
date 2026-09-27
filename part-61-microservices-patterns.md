# Part 61: Advanced Microservices Patterns
## ขั้นตอนที่ 2041-2080

**ระดับ:** ระดับโลก (World-Class)
**เวลาเรียน:** 4-5 ชั่วโมง
**เป้าหมาย:** เข้าใจและนำ Advanced Microservices Patterns มาใช้งานจริงในระบบ Production

---

## บทนำ

ในส่วนนี้เราจะเรียนรู้ Pattern ขั้นสูงสำหรับ Microservices ที่ใช้จริงในระบบขนาดใหญ่ระดับ Enterprise ทุก Pattern มีเหตุผลในการใช้งานและ Trade-off ที่ต้องพิจารณาอย่างรอบคอบ

---

## ขั้นตอนที่ 2041: Service Discovery Patterns - Client-Side vs Server-Side

### อธิบาย

Service Discovery คือกลไกที่ให้ Service หนึ่งค้นหาและเชื่อมต่อกับ Service อื่นได้โดยอัตโนมัติ มี 2 แนวทางหลัก:

**Client-Side Discovery:** Client ถามตรงไปที่ Service Registry แล้วทำ Load Balancing เอง
**Server-Side Discovery:** Client ส่ง Request ไปที่ Load Balancer แล้ว Load Balancer ถาม Registry เอง

### Client-Side Discovery ด้วย Spring Cloud Netflix Eureka

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
</dependency>
```

```java
// EurekaServerApplication.java
@SpringBootApplication
@EnableEurekaServer
public class EurekaServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(EurekaServerApplication.class, args);
    }
}
```

```yaml
# application.yml สำหรับ Eureka Server
server:
  port: 8761

eureka:
  instance:
    hostname: localhost
  client:
    registerWithEureka: false
    fetchRegistry: false
    serviceUrl:
      defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/
  server:
    waitTimeInMsWhenSyncEmpty: 0
    enableSelfPreservation: false
```

```java
// OrderService.java - ตัวอย่าง Client ที่ใช้ Eureka
@SpringBootApplication
@EnableDiscoveryClient
public class OrderServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(OrderServiceApplication.class, args);
    }
    
    @Bean
    @LoadBalanced  // ทำให้ RestTemplate ทำ Client-Side Load Balancing
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }
}
```

```yaml
# application.yml สำหรับ Order Service
spring:
  application:
    name: order-service

eureka:
  client:
    serviceUrl:
      defaultZone: http://localhost:8761/eureka/
  instance:
    preferIpAddress: true
    instanceId: ${spring.application.name}:${random.value}
```

```java
// OrderController.java
@RestController
@RequestMapping("/api/orders")
@RequiredArgsConstructor
public class OrderController {
    
    private final RestTemplate restTemplate;
    private final DiscoveryClient discoveryClient;
    
    @GetMapping("/product/{productId}")
    public ResponseEntity<ProductDto> getProduct(@PathVariable String productId) {
        // Client-Side Discovery: ใช้ชื่อ Service แทน URL จริง
        String url = "http://product-service/api/products/" + productId;
        return restTemplate.getForEntity(url, ProductDto.class);
    }
    
    @GetMapping("/services")
    public List<String> getRegisteredServices() {
        // ดูรายการ Service ที่ Register ไว้ใน Eureka
        return discoveryClient.getServices();
    }
    
    @GetMapping("/instances/{serviceName}")
    public List<ServiceInstance> getInstances(@PathVariable String serviceName) {
        // ดูรายละเอียด Instance ของ Service
        return discoveryClient.getInstances(serviceName);
    }
}
```

### Server-Side Discovery ด้วย AWS ALB / Kubernetes Service

```yaml
# kubernetes-service.yaml - Server-Side Discovery ด้วย Kubernetes
apiVersion: v1
kind: Service
metadata:
  name: product-service
  labels:
    app: product-service
spec:
  selector:
    app: product-service
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
  type: ClusterIP  # Kubernetes ทำหน้าที่เป็น Load Balancer ให้
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: product-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: product-service
  template:
    metadata:
      labels:
        app: product-service
    spec:
      containers:
        - name: product-service
          image: product-service:latest
          ports:
            - containerPort: 8080
```

```java
// ProductServiceClient.java - ใช้ชื่อ Kubernetes Service DNS
@Service
@RequiredArgsConstructor
public class ProductServiceClient {
    
    private final WebClient.Builder webClientBuilder;
    
    // ใน Kubernetes ไม่ต้องทำ Client-Side Discovery
    // DNS ของ Kubernetes จะ Resolve "product-service" ให้อัตโนมัติ
    @Value("${services.product.url:http://product-service}")
    private String productServiceUrl;
    
    public Mono<ProductDto> getProduct(String productId) {
        return webClientBuilder
            .baseUrl(productServiceUrl)
            .build()
            .get()
            .uri("/api/products/{id}", productId)
            .retrieve()
            .bodyToMono(ProductDto.class);
    }
}
```

---

## ขั้นตอนที่ 2042-2045: Bulkhead Pattern ด้วย Thread Pools

### อธิบาย

Bulkhead Pattern มาจากการออกแบบเรือที่แบ่ง Compartment เพื่อป้องกันน้ำท่วมทั้งเรือ ในซอฟต์แวร์คือการแยก Resources เพื่อป้องกันการล้มเหลวของ Service หนึ่งไม่ให้กระทบ Service อื่น

### การใช้ Resilience4j Bulkhead

```xml
<dependency>
    <groupId>io.github.resilience4j</groupId>
    <artifactId>resilience4j-spring-boot3</artifactId>
</dependency>
<dependency>
    <groupId>io.github.resilience4j</groupId>
    <artifactId>resilience4j-bulkhead</artifactId>
</dependency>
```

```yaml
# application.yml
resilience4j:
  bulkhead:
    instances:
      paymentService:
        maxConcurrentCalls: 10        # จำนวน Concurrent Call สูงสุด
        maxWaitDuration: 500ms        # รอนานสุด 500ms ถ้า Bulkhead เต็ม
      inventoryService:
        maxConcurrentCalls: 25
        maxWaitDuration: 100ms
  
  thread-pool-bulkhead:
    instances:
      externalApiService:
        maxThreadPoolSize: 10         # Thread Pool ขนาด 10
        coreThreadPoolSize: 5         # Core Thread 5
        queueCapacity: 100            # Queue รอ 100
        keepAliveDuration: 20ms
```

```java
// BulkheadDemoService.java
@Service
@RequiredArgsConstructor
@Slf4j
public class BulkheadDemoService {
    
    private final PaymentClient paymentClient;
    private final InventoryClient inventoryClient;
    
    // Semaphore-based Bulkhead สำหรับ Synchronous calls
    @Bulkhead(name = "paymentService", fallbackMethod = "paymentFallback")
    public PaymentResponse processPayment(PaymentRequest request) {
        log.info("Processing payment for order: {}", request.getOrderId());
        return paymentClient.process(request);
    }
    
    // Thread Pool Bulkhead สำหรับ Asynchronous calls
    @Bulkhead(name = "externalApiService", 
              type = Bulkhead.Type.THREADPOOL,
              fallbackMethod = "externalApiFallback")
    public CompletableFuture<ExternalData> callExternalApi(String requestId) {
        return CompletableFuture.supplyAsync(() -> {
            log.info("Calling external API for: {}", requestId);
            return externalApiClient.getData(requestId);
        });
    }
    
    public PaymentResponse paymentFallback(PaymentRequest request, 
                                            BulkheadFullException e) {
        log.warn("Payment Bulkhead full, returning cached response for: {}", 
                 request.getOrderId());
        return PaymentResponse.pending(request.getOrderId());
    }
    
    public CompletableFuture<ExternalData> externalApiFallback(
            String requestId, BulkheadFullException e) {
        return CompletableFuture.completedFuture(ExternalData.empty());
    }
}
```

```java
// BulkheadConfig.java - Custom Configuration
@Configuration
public class BulkheadConfig {
    
    @Bean
    public BulkheadRegistry bulkheadRegistry() {
        BulkheadConfig paymentConfig = BulkheadConfig.custom()
            .maxConcurrentCalls(10)
            .maxWaitDuration(Duration.ofMillis(500))
            .build();
        
        BulkheadConfig criticalConfig = BulkheadConfig.custom()
            .maxConcurrentCalls(5)
            .maxWaitDuration(Duration.ofMillis(100))
            .build();
        
        return BulkheadRegistry.of(Map.of(
            "paymentService", paymentConfig,
            "criticalService", criticalConfig
        ));
    }
    
    @Bean
    public ThreadPoolBulkheadRegistry threadPoolBulkheadRegistry() {
        ThreadPoolBulkheadConfig config = ThreadPoolBulkheadConfig.custom()
            .maxThreadPoolSize(10)
            .coreThreadPoolSize(5)
            .queueCapacity(100)
            .keepAliveDuration(Duration.ofMillis(20))
            .build();
        
        return ThreadPoolBulkheadRegistry.of(config);
    }
}
```

```java
// BulkheadMetricsController.java - Monitor Bulkhead stats
@RestController
@RequestMapping("/admin/bulkhead")
@RequiredArgsConstructor
public class BulkheadMetricsController {
    
    private final BulkheadRegistry bulkheadRegistry;
    
    @GetMapping("/stats")
    public Map<String, Object> getBulkheadStats() {
        Map<String, Object> stats = new HashMap<>();
        
        bulkheadRegistry.getAllBulkheads().forEach(bulkhead -> {
            Bulkhead.Metrics metrics = bulkhead.getMetrics();
            stats.put(bulkhead.getName(), Map.of(
                "availableConcurrentCalls", metrics.getAvailableConcurrentCalls(),
                "maxAllowedConcurrentCalls", metrics.getMaxAllowedConcurrentCalls()
            ));
        });
        
        return stats;
    }
}
```

---

## ขั้นตอนที่ 2046-2050: Sidecar Pattern

### อธิบาย

Sidecar Pattern คือการเพิ่ม Container/Process เล็กๆ ที่ทำงานคู่กับ Service หลัก โดยรับผิดชอบงาน Cross-Cutting Concerns เช่น Logging, Monitoring, Security, Configuration

```
[Application Container] <--> [Sidecar Container]
        |                           |
        v                           v
   Business Logic            Logging/Tracing/Config
```

### ตัวอย่าง Sidecar ด้วย Envoy Proxy

```yaml
# kubernetes-with-sidecar.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  replicas: 2
  selector:
    matchLabels:
      app: order-service
  template:
    metadata:
      labels:
        app: order-service
      annotations:
        # Istio inject sidecar อัตโนมัติ
        sidecar.istio.io/inject: "true"
    spec:
      containers:
        # Container หลัก
        - name: order-service
          image: order-service:latest
          ports:
            - containerPort: 8080
          env:
            - name: SPRING_PROFILES_ACTIVE
              value: "kubernetes"
        
        # Sidecar Container สำหรับ Log Collection
        - name: fluentd-sidecar
          image: fluent/fluentd:v1.16
          volumeMounts:
            - name: app-logs
              mountPath: /var/log/app
          resources:
            limits:
              memory: "128Mi"
              cpu: "100m"
      
      volumes:
        - name: app-logs
          emptyDir: {}
```

```java
// LoggingConfig.java - ส่ง Log ไปยัง File ที่ Sidecar อ่าน
@Configuration
public class LoggingConfig {
    
    @Bean
    public LoggingEventCompositeJsonEncoder jsonEncoder() {
        LoggingEventCompositeJsonEncoder encoder = 
            new LoggingEventCompositeJsonEncoder();
        
        // ตั้งค่า JSON format สำหรับ Structured Logging
        List<JsonProvider<ILoggingEvent>> providers = new ArrayList<>();
        providers.add(new TimestampJsonProvider());
        providers.add(new LogLevelJsonProvider());
        providers.add(new MessageJsonProvider());
        providers.add(new LoggerNameJsonProvider());
        providers.add(new ThreadNameJsonProvider());
        providers.add(new MdcJsonProvider());
        
        encoder.setProviders(new JsonProviders<>(providers));
        return encoder;
    }
}
```

```yaml
# fluent.conf - Config สำหรับ Fluentd Sidecar
<source>
  @type tail
  path /var/log/app/*.log
  pos_file /var/log/fluentd/app.log.pos
  tag app.logs
  <parse>
    @type json
    time_format %Y-%m-%dT%H:%M:%S.%NZ
  </parse>
</source>

<match app.logs>
  @type elasticsearch
  host elasticsearch.logging.svc.cluster.local
  port 9200
  index_name order-service-logs
  <buffer>
    flush_interval 5s
  </buffer>
</match>
```

---

## ขั้นตอนที่ 2051-2055: Ambassador Pattern

### อธิบาย

Ambassador Pattern คือ Sidecar พิเศษที่ทำหน้าที่เป็น Proxy สำหรับการสื่อสารออกไปข้างนอก (Outbound) รับผิดชอบ: Retry, Circuit Breaking, Load Balancing, Authentication

```java
// AmbassadorService.java - Spring Service ที่เป็น Ambassador
@Service
@RequiredArgsConstructor
@Slf4j
public class AmbassadorService {
    
    private final WebClient.Builder webClientBuilder;
    private final CircuitBreakerFactory circuitBreakerFactory;
    private final RetryRegistry retryRegistry;
    
    // Ambassador จัดการ Retry, Circuit Breaker, Timeout ทั้งหมด
    public <T> Mono<T> callService(String serviceUrl, Class<T> responseType) {
        CircuitBreaker cb = circuitBreakerFactory.create("ambassador");
        Retry retry = retryRegistry.retry("ambassador");
        
        return webClientBuilder
            .build()
            .get()
            .uri(serviceUrl)
            .retrieve()
            .bodyToMono(responseType)
            .retryWhen(Retry.backoff(3, Duration.ofMillis(100))
                .maxBackoff(Duration.ofSeconds(2))
                .jitter(0.5))
            .timeout(Duration.ofSeconds(5))
            .transformDeferred(CircuitBreakerOperator.of(cb))
            .doOnError(e -> log.error("Ambassador: Error calling {}: {}", 
                                       serviceUrl, e.getMessage()));
    }
}
```

```java
// ExternalServiceAmbassador.java - Ambassador สำหรับ External Services
@Component
@Slf4j
public class ExternalServiceAmbassador {
    
    private final HttpClient httpClient;
    private final RetryTemplate retryTemplate;
    
    public ExternalServiceAmbassador() {
        // สร้าง HTTP Client ที่มี Connection Pool และ Timeout
        this.httpClient = HttpClient.create()
            .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 5000)
            .responseTimeout(Duration.ofSeconds(10))
            .doOnConnected(conn -> conn
                .addHandlerLast(new ReadTimeoutHandler(10))
                .addHandlerLast(new WriteTimeoutHandler(10)));
        
        // ตั้งค่า Retry
        this.retryTemplate = RetryTemplate.builder()
            .maxAttempts(3)
            .exponentialBackoff(100, 2, 2000, true)
            .retryOn(ResourceAccessException.class)
            .build();
    }
    
    public <T> T get(String url, Class<T> responseType) {
        return retryTemplate.execute(context -> {
            log.debug("Ambassador: GET {} (attempt {})", 
                     url, context.getRetryCount() + 1);
            
            ResponseEntity<T> response = new RestTemplate(
                new HttpComponentsClientHttpRequestFactory()
            ).getForEntity(url, responseType);
            
            return response.getBody();
        });
    }
}
```

---

## ขั้นตอนที่ 2056-2060: Anti-Corruption Layer (ACL)

### อธิบาย

ACL คือ Pattern ที่ใช้เมื่อต้องการ Integrate กับ System ภายนอกที่มี Model แตกต่างจาก Domain ของเรา เพื่อป้องกันไม่ให้ Concept ของ External System "ปนเปื้อน" Domain ของเรา

```
[Our Domain] <--> [ACL Layer] <--> [External/Legacy System]
   Order              Translator        OrderLegacy
   Product            Mapper            ProductRecord
   Customer           Adapter           ClientRecord
```

```java
// ExternalOrderResponse.java - Model จาก External System (Legacy)
@Data
public class ExternalOrderResponse {
    // Legacy System ใช้ชื่อ Field แตกต่าง
    private String ord_num;          // เราเรียก orderId
    private String cli_code;         // เราเรียก customerId
    private String prod_sku;         // เราเรียก productId
    private BigDecimal unit_price;   // เราเรียก price
    private Integer qty;             // เราเรียก quantity
    private String stat_code;        // เราเรียก status (ใช้ code ต่างกัน)
    private String cre_dt;           // เราเรียก createdAt (format ต่างกัน)
}
```

```java
// Order.java - Domain Model ของเรา
@Data
@Builder
public class Order {
    private String orderId;
    private String customerId;
    private String productId;
    private BigDecimal price;
    private Integer quantity;
    private OrderStatus status;
    private LocalDateTime createdAt;
    
    public enum OrderStatus {
        PENDING, CONFIRMED, SHIPPED, DELIVERED, CANCELLED
    }
}
```

```java
// LegacyOrderTranslator.java - ACL Translator
@Component
@Slf4j
public class LegacyOrderTranslator {
    
    private static final DateTimeFormatter LEGACY_DATE_FORMAT = 
        DateTimeFormatter.ofPattern("yyyyMMddHHmmss");
    
    // Legacy status codes -> Domain status mapping
    private static final Map<String, Order.OrderStatus> STATUS_MAP = Map.of(
        "P", Order.OrderStatus.PENDING,
        "C", Order.OrderStatus.CONFIRMED,
        "S", Order.OrderStatus.SHIPPED,
        "D", Order.OrderStatus.DELIVERED,
        "X", Order.OrderStatus.CANCELLED
    );
    
    // แปลง External Model เป็น Domain Model
    public Order toDomain(ExternalOrderResponse external) {
        if (external == null) {
            return null;
        }
        
        return Order.builder()
            .orderId(translateOrderId(external.getOrd_num()))
            .customerId(translateCustomerId(external.getCli_code()))
            .productId(external.getProd_sku())
            .price(external.getUnit_price())
            .quantity(external.getQty())
            .status(translateStatus(external.getStat_code()))
            .createdAt(translateDate(external.getCre_dt()))
            .build();
    }
    
    // แปลง Domain Model เป็น External Model
    public ExternalOrderResponse toExternal(Order order) {
        ExternalOrderResponse external = new ExternalOrderResponse();
        external.setOrd_num(reverseTranslateOrderId(order.getOrderId()));
        external.setCli_code(reverseTranslateCustomerId(order.getCustomerId()));
        external.setProd_sku(order.getProductId());
        external.setUnit_price(order.getPrice());
        external.setQty(order.getQuantity());
        external.setStat_code(reverseTranslateStatus(order.getStatus()));
        external.setCre_dt(reverseTranslateDate(order.getCreatedAt()));
        return external;
    }
    
    private String translateOrderId(String legacyId) {
        // Legacy ใช้ format "ORD-YYYYMMDD-NNNN"
        // เราใช้ UUID format
        if (legacyId == null) return null;
        return "ORD-" + legacyId.replaceAll("[^0-9]", "");
    }
    
    private String translateCustomerId(String clientCode) {
        // แปลง Client Code จาก Legacy
        return clientCode != null ? "CUST-" + clientCode : null;
    }
    
    private Order.OrderStatus translateStatus(String statusCode) {
        if (statusCode == null) {
            return Order.OrderStatus.PENDING;
        }
        return STATUS_MAP.getOrDefault(statusCode, Order.OrderStatus.PENDING);
    }
    
    private LocalDateTime translateDate(String dateStr) {
        if (dateStr == null || dateStr.isEmpty()) {
            return null;
        }
        try {
            return LocalDateTime.parse(dateStr, LEGACY_DATE_FORMAT);
        } catch (DateTimeParseException e) {
            log.warn("Cannot parse legacy date: {}", dateStr);
            return null;
        }
    }
    
    // Reverse translations...
    private String reverseTranslateOrderId(String orderId) {
        return orderId != null ? orderId.replace("ORD-", "") : null;
    }
    
    private String reverseTranslateCustomerId(String customerId) {
        return customerId != null ? customerId.replace("CUST-", "") : null;
    }
    
    private String reverseTranslateStatus(Order.OrderStatus status) {
        return STATUS_MAP.entrySet().stream()
            .filter(e -> e.getValue() == status)
            .map(Map.Entry::getKey)
            .findFirst()
            .orElse("P");
    }
    
    private String reverseTranslateDate(LocalDateTime dateTime) {
        return dateTime != null ? dateTime.format(LEGACY_DATE_FORMAT) : null;
    }
}
```

```java
// LegacyOrderAdapter.java - ACL Adapter
@Service
@RequiredArgsConstructor
@Slf4j
public class LegacyOrderAdapter implements OrderRepository {
    
    private final LegacyOrderClient legacyClient;
    private final LegacyOrderTranslator translator;
    
    @Override
    public Optional<Order> findById(String orderId) {
        try {
            String legacyId = translator.toLegacyOrderId(orderId);
            ExternalOrderResponse response = legacyClient.getOrder(legacyId);
            return Optional.ofNullable(translator.toDomain(response));
        } catch (LegacySystemException e) {
            log.error("ACL: Error fetching order from legacy: {}", e.getMessage());
            return Optional.empty();
        }
    }
    
    @Override
    public Order save(Order order) {
        ExternalOrderResponse external = translator.toExternal(order);
        ExternalOrderResponse saved = legacyClient.saveOrder(external);
        return translator.toDomain(saved);
    }
}
```

---

## ขั้นตอนที่ 2061-2065: Strangler Fig Pattern

### อธิบาย

Strangler Fig Pattern มาจากต้นไม้ที่ค่อยๆ ปกคลุมและแทนที่ต้นไม้เก่า เป็น Pattern สำหรับการ Migrate จาก Legacy System ไปยัง Modern System แบบ Incremental โดยไม่ต้อง Rewrite ทั้งหมดในครั้งเดียว

```
Phase 1: Legacy handles everything
[Client] -> [Legacy System]

Phase 2: Route some paths to new service
[Client] -> [Facade/Router] -> [Legacy System]
                            -> [New Service] (new features)

Phase 3: Gradually strangle legacy
[Client] -> [Facade/Router] -> [Legacy System] (remaining)
                            -> [New Service] (most features)

Phase 4: Legacy is gone
[Client] -> [New System]
```

```java
// StranglerFacade.java - Routing Facade
@RestController
@RequiredArgsConstructor
@Slf4j
public class StranglerFacade {
    
    private final FeatureToggleService featureToggle;
    private final LegacyOrderAdapter legacyAdapter;
    private final NewOrderService newOrderService;
    
    // Route ตาม Feature Toggle
    @GetMapping("/api/orders/{orderId}")
    public ResponseEntity<OrderDto> getOrder(@PathVariable String orderId) {
        if (featureToggle.isEnabled("use-new-order-service", orderId)) {
            log.info("Routing to NEW service for orderId: {}", orderId);
            return newOrderService.getOrder(orderId);
        } else {
            log.info("Routing to LEGACY system for orderId: {}", orderId);
            return legacyAdapter.getOrder(orderId);
        }
    }
    
    // Feature ใหม่ไปที่ New Service เสมอ
    @PostMapping("/api/orders")
    public ResponseEntity<OrderDto> createOrder(@RequestBody CreateOrderRequest request) {
        return newOrderService.createOrder(request);
    }
}
```

```java
// FeatureToggleService.java - Gradual Migration Control
@Service
@RequiredArgsConstructor
public class FeatureToggleService {
    
    private final FeatureToggleRepository repository;
    
    // ควบคุม % ของ Traffic ที่ไปยัง New Service
    public boolean isEnabled(String feature, String entityId) {
        FeatureToggle toggle = repository.findByName(feature)
            .orElse(FeatureToggle.disabled(feature));
        
        if (!toggle.isEnabled()) {
            return false;
        }
        
        // Percentage rollout ตาม entityId hash
        if (toggle.getRolloutPercentage() < 100) {
            int hash = Math.abs(entityId.hashCode() % 100);
            return hash < toggle.getRolloutPercentage();
        }
        
        return true;
    }
    
    // เพิ่ม Rollout percentage ทีละน้อย
    public void increaseRollout(String feature, int percentage) {
        FeatureToggle toggle = repository.findByName(feature)
            .orElseThrow(() -> new NotFoundException(feature));
        
        int newPercentage = Math.min(100, 
            toggle.getRolloutPercentage() + percentage);
        toggle.setRolloutPercentage(newPercentage);
        repository.save(toggle);
        
        log.info("Feature '{}' rollout increased to {}%", feature, newPercentage);
    }
}
```

```java
// MigrationVerifier.java - Verify ว่า New Service ให้ผลเหมือน Legacy
@Service
@RequiredArgsConstructor
@Slf4j
public class MigrationVerifier {
    
    private final LegacyOrderAdapter legacyAdapter;
    private final NewOrderService newOrderService;
    private final MigrationMetrics metrics;
    
    // Shadow Testing: Call ทั้ง 2 แล้ว Compare ผล
    @Async
    public CompletableFuture<Void> shadowTest(String orderId) {
        CompletableFuture<OrderDto> legacyFuture = CompletableFuture
            .supplyAsync(() -> legacyAdapter.getOrder(orderId).getBody());
        
        CompletableFuture<OrderDto> newFuture = CompletableFuture
            .supplyAsync(() -> newOrderService.getOrder(orderId).getBody());
        
        return CompletableFuture.allOf(legacyFuture, newFuture)
            .thenAccept(v -> {
                OrderDto legacyResult = legacyFuture.join();
                OrderDto newResult = newFuture.join();
                
                if (!isEquivalent(legacyResult, newResult)) {
                    log.warn("Shadow test MISMATCH for orderId: {} | Legacy: {} | New: {}",
                             orderId, legacyResult, newResult);
                    metrics.recordMismatch(orderId);
                } else {
                    metrics.recordMatch(orderId);
                }
            });
    }
    
    private boolean isEquivalent(OrderDto legacy, OrderDto newDto) {
        if (legacy == null && newDto == null) return true;
        if (legacy == null || newDto == null) return false;
        
        return Objects.equals(legacy.getOrderId(), newDto.getOrderId()) &&
               Objects.equals(legacy.getStatus(), newDto.getStatus()) &&
               legacy.getTotal().compareTo(newDto.getTotal()) == 0;
    }
}
```

---

## ขั้นตอนที่ 2066-2070: Service Mesh Concepts

### อธิบาย

Service Mesh เป็น Infrastructure Layer ที่จัดการ Service-to-Service Communication โดยใช้ Sidecar Proxies (เช่น Envoy) ที่ Inject เข้าไปในแต่ละ Pod

### ความสามารถหลักของ Service Mesh

```
Without Service Mesh:          With Service Mesh (Istio):
Service A ---HTTP---> Service B    Service A ---> [Envoy] --mTLS--> [Envoy] ---> Service B
                                      |                                  |
                                   Metrics                           Metrics
                                   Traces                            Traces
                                   Auth                              Auth
```

```yaml
# istio-virtual-service.yaml - Traffic Management
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: product-service
spec:
  hosts:
    - product-service
  http:
    # Canary Deployment: 90% ไป v1, 10% ไป v2
    - match:
        - headers:
            x-canary:
              exact: "true"
      route:
        - destination:
            host: product-service
            subset: v2
    - route:
        - destination:
            host: product-service
            subset: v1
          weight: 90
        - destination:
            host: product-service
            subset: v2
          weight: 10
---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: product-service
spec:
  host: product-service
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        h2UpgradePolicy: UPGRADE
        http1MaxPendingRequests: 100
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 10s
      baseEjectionTime: 30s
  subsets:
    - name: v1
      labels:
        version: v1
    - name: v2
      labels:
        version: v2
```

```yaml
# istio-peer-authentication.yaml - mTLS
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: production
spec:
  mtls:
    mode: STRICT  # บังคับให้ใช้ mTLS ทุก Service
---
# Authorization Policy
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: allow-order-to-payment
  namespace: production
spec:
  selector:
    matchLabels:
      app: payment-service
  rules:
    - from:
        - source:
            principals:
              - "cluster.local/ns/production/sa/order-service"
      to:
        - operation:
            methods: ["POST"]
            paths: ["/api/payments/*"]
```

```java
// ServiceMeshAwareController.java - Spring Boot + Service Mesh
@RestController
@RequiredArgsConstructor
public class ServiceMeshAwareController {
    
    @GetMapping("/health")
    public ResponseEntity<Map<String, String>> health() {
        // Istio ใช้ Endpoint นี้สำหรับ Health Check
        return ResponseEntity.ok(Map.of(
            "status", "UP",
            "service", "order-service"
        ));
    }
    
    @GetMapping("/metrics")
    public String metrics() {
        // Prometheus Metrics สำหรับ Istio
        // Istio จะ scrape metrics จาก /metrics
        return "# Handled by Micrometer + Prometheus";
    }
}
```

---

## ขั้นตอนที่ 2071-2080: Pattern Combination - Real World Example

### สถานการณ์จริง: E-Commerce Platform Migration

ลองดูตัวอย่างการใช้หลาย Pattern ร่วมกันในการ Migrate E-Commerce Platform จาก Monolith ไป Microservices

```java
// OrderOrchestrator.java - รวม Patterns หลายตัว
@Service
@RequiredArgsConstructor
@Slf4j
public class OrderOrchestrator {
    
    private final InventoryService inventoryService;
    private final PaymentService paymentService;
    private final NotificationService notificationService;
    private final BulkheadRegistry bulkheadRegistry;
    private final CircuitBreakerRegistry circuitBreakerRegistry;
    
    public OrderResult processOrder(OrderRequest request) {
        // 1. Bulkhead: แยก Thread Pool สำหรับ Order Processing
        Bulkhead bulkhead = bulkheadRegistry.bulkhead("orderProcessing");
        
        return Bulkhead.decorateSupplier(bulkhead, () -> {
            // 2. Circuit Breaker สำหรับแต่ละ Service
            CircuitBreaker inventoryCB = circuitBreakerRegistry
                .circuitBreaker("inventoryService");
            CircuitBreaker paymentCB = circuitBreakerRegistry
                .circuitBreaker("paymentService");
            
            // 3. Check Inventory ผ่าน ACL
            InventoryResult inventory = CircuitBreaker
                .decorateSupplier(inventoryCB, 
                    () -> inventoryService.checkAndReserve(request))
                .get();
            
            if (!inventory.isAvailable()) {
                return OrderResult.failed("INSUFFICIENT_INVENTORY");
            }
            
            // 4. Process Payment ผ่าน Ambassador Pattern
            PaymentResult payment = CircuitBreaker
                .decorateSupplier(paymentCB,
                    () -> paymentService.charge(request.getPayment()))
                .get();
            
            if (!payment.isSuccessful()) {
                // Compensate: คืน Inventory
                inventoryService.release(inventory.getReservationId());
                return OrderResult.failed("PAYMENT_FAILED");
            }
            
            // 5. Create Order (Domain Logic)
            Order order = Order.builder()
                .orderId(generateOrderId())
                .customerId(request.getCustomerId())
                .status(OrderStatus.CONFIRMED)
                .build();
            
            // 6. Send Notification ผ่าน Sidecar
            notificationService.sendAsync(OrderConfirmation.from(order));
            
            return OrderResult.success(order);
            
        }).get();
    }
}
```

```java
// IntegrationTestForPatterns.java - Test ทุก Pattern รวมกัน
@SpringBootTest
@ActiveProfiles("test")
class OrderOrchestratorIntegrationTest {
    
    @Autowired
    private OrderOrchestrator orchestrator;
    
    @MockBean
    private InventoryService inventoryService;
    
    @MockBean
    private PaymentService paymentService;
    
    @Test
    void shouldProcessOrderSuccessfully() {
        // Arrange
        given(inventoryService.checkAndReserve(any()))
            .willReturn(InventoryResult.available("RES-001"));
        given(paymentService.charge(any()))
            .willReturn(PaymentResult.success("PAY-001"));
        
        OrderRequest request = createTestRequest();
        
        // Act
        OrderResult result = orchestrator.processOrder(request);
        
        // Assert
        assertThat(result.isSuccess()).isTrue();
        assertThat(result.getOrder().getStatus()).isEqualTo(OrderStatus.CONFIRMED);
    }
    
    @Test
    void shouldHandleBulkheadFull() throws InterruptedException {
        // ทดสอบว่า Bulkhead ทำงานเมื่อมี Concurrent Calls มากเกินไป
        given(inventoryService.checkAndReserve(any()))
            .willAnswer(inv -> {
                Thread.sleep(1000); // Simulate slow service
                return InventoryResult.available("RES-001");
            });
        
        int numConcurrentRequests = 20;
        CountDownLatch latch = new CountDownLatch(numConcurrentRequests);
        List<OrderResult> results = Collections.synchronizedList(new ArrayList<>());
        
        for (int i = 0; i < numConcurrentRequests; i++) {
            new Thread(() -> {
                try {
                    results.add(orchestrator.processOrder(createTestRequest()));
                } catch (BulkheadFullException e) {
                    results.add(OrderResult.failed("BULKHEAD_FULL"));
                } finally {
                    latch.countDown();
                }
            }).start();
        }
        
        latch.await(10, TimeUnit.SECONDS);
        
        long failedDueToBulkhead = results.stream()
            .filter(r -> "BULKHEAD_FULL".equals(r.getFailureReason()))
            .count();
        
        // ควรมีบาง Request ล้มเหลวเพราะ Bulkhead เต็ม
        assertThat(failedDueToBulkhead).isGreaterThan(0);
    }
}
```

---

## สรุป Advanced Microservices Patterns

| Pattern | เมื่อใช้ | ประโยชน์ | ข้อควรระวัง |
|---------|---------|---------|------------|
| Service Discovery | ทุก Microservices deployment | Dynamic routing | Registry เป็น SPOF |
| Bulkhead | Service มี slow/unstable dependencies | Fault isolation | เพิ่ม complexity |
| Sidecar | Cross-cutting concerns | Separation of concerns | เพิ่ม resource usage |
| Ambassador | External service integration | Resilience for outbound | Network hop เพิ่ม |
| ACL | Legacy/external system integration | Domain protection | Maintenance overhead |
| Strangler Fig | Legacy migration | Safe incremental migration | ต้องมี routing layer |
| Service Mesh | Large scale microservices | Observability + Security | High complexity |

---

*[← Part 60: Performance Testing](./part-60-performance-testing.md) | [Part 62: API Gateway Advanced →](./part-62-api-gateway-advanced.md)*

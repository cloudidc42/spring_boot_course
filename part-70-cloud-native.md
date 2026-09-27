# Part 70: Cloud Native
## ขั้นตอนที่ 2401-2440

**ระดับ:** ระดับโลก (World-Class)
**เวลาเรียน:** 7-8 ชั่วโมง
**เป้าหมาย:** เชี่ยวชาญการออกแบบ Spring Boot Applications ตาม Cloud Native principles ด้วย 12-Factor App, Kubernetes integration, Health checks, Service Discovery และ Horizontal Pod Autoscaling

---

## ขั้นตอนที่ 2401-2406: 12-Factor App สำหรับ Spring Boot

### 12 หลักการสำหรับ Cloud Native Applications

**Factor I: Codebase** - หนึ่ง codebase, หลาย deployments
```
Git Repository:
├── develop branch → DEV environment
├── staging branch → STAGING environment
└── main branch    → PRODUCTION environment

// ❌ BAD: แยก codebase ตาม environment
order-service-dev/
order-service-prod/

// ✅ GOOD: หนึ่ง codebase, ใช้ environment variables แยก
order-service/
├── src/
└── Dockerfile
```

**Factor II: Dependencies** - ประกาศ dependencies อย่างชัดเจน

```xml
<!-- pom.xml - ประกาศทุก dependency ด้วย Maven -->
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
        <!-- ไม่มี version: ใช้ parent BOM -->
    </dependency>
</dependencies>

<!-- ✅ Lock versions ด้วย dependency management -->
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>${spring-cloud.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

**Factor III: Config** - Config อยู่ใน Environment ไม่ใช่ Code

```java
// ❌ BAD: Hardcode config ใน code
@Service
public class EmailService {
    private final String smtpHost = "smtp.gmail.com"; // Hardcoded!
    private final String apiKey = "sk_live_xxxx";     // Hardcoded!
}

// ✅ GOOD: ดึง config จาก environment
@Service
public class EmailService {

    @Value("${mail.smtp.host}")
    private String smtpHost;

    @Value("${mail.api-key}")
    private String apiKey;

    // หรือใช้ @ConfigurationProperties
}
```

```yaml
# application.yml - ใช้ environment variables
spring:
  datasource:
    url: ${DATABASE_URL:jdbc:postgresql://localhost:5432/orders}
    username: ${DB_USERNAME:local_user}
    password: ${DB_PASSWORD:local_pass}

mail:
  smtp:
    host: ${SMTP_HOST:localhost}
    port: ${SMTP_PORT:587}
  api-key: ${MAIL_API_KEY}

app:
  jwt:
    secret: ${JWT_SECRET}
    expiration: ${JWT_EXPIRATION:86400}
```

**Factor IV: Backing Services** - treat external services เป็น attached resources

```java
// ✅ GOOD: Services ต้องสามารถเปลี่ยน backing service ได้โดยไม่แก้ code
@Configuration
public class CacheConfig {

    @Bean
    public CacheManager cacheManager(RedisConnectionFactory connectionFactory) {
        // เปลี่ยน Redis host ได้โดยแค่เปลี่ยน env variable REDIS_HOST
        RedisCacheManager manager = RedisCacheManager.builder(connectionFactory)
            .cacheDefaults(RedisCacheConfiguration.defaultCacheConfig()
                .entryTtl(Duration.ofMinutes(10)))
            .build();
        return manager;
    }
}
```

**Factor V: Build, Release, Run** - แยก Build, Release, Run ชัดเจน

```dockerfile
# Dockerfile - Multi-stage build (Build stage)
FROM eclipse-temurin:21-jdk-alpine AS builder
WORKDIR /app
COPY . .
RUN ./mvnw package -DskipTests

# Release stage - รวม binary กับ config
FROM eclipse-temurin:21-jre-alpine AS release
WORKDIR /app
COPY --from=builder /app/target/*.jar app.jar

# Run stage
ENTRYPOINT ["java", \
    "-XX:+UseContainerSupport", \
    "-XX:MaxRAMPercentage=75.0", \
    "-jar", "app.jar"]
```

**Factor VI: Processes** - application ต้องเป็น stateless

```java
// ❌ BAD: เก็บ state ใน memory (ปัญหาเมื่อมีหลาย instances)
@Service
public class SessionService {
    private final Map<String, UserSession> sessions = new ConcurrentHashMap<>(); // State ใน memory!

    public void createSession(String userId, UserSession session) {
        sessions.put(userId, session); // ถ้า request ถัดไปไปอีก instance จะหาไม่เจอ!
    }
}

// ✅ GOOD: เก็บ state ใน external store (Redis)
@Service
public class SessionService {

    private final RedisTemplate<String, UserSession> redisTemplate;

    public void createSession(String userId, UserSession session) {
        redisTemplate.opsForValue().set(
            "session:" + userId,
            session,
            Duration.ofHours(24)
        );
    }

    public Optional<UserSession> getSession(String userId) {
        UserSession session = redisTemplate.opsForValue().get("session:" + userId);
        return Optional.ofNullable(session);
    }
}
```

**Factor VII: Port Binding** - Export services ผ่าน port

```yaml
# application.yml
server:
  port: ${SERVER_PORT:8080}
  # Application เป็น self-contained ไม่ต้องการ external server
```

**Factor VIII: Concurrency** - Scale ออกด้วย Process model

```yaml
# Kubernetes - scale horizontally
apiVersion: apps/v1
kind: Deployment
spec:
  replicas: 3  # เพิ่ม replicas เพื่อ scale
  # ไม่ใช่ vertical scaling (เพิ่ม CPU/RAM)
```

**Factor IX: Disposability** - Fast startup, Graceful shutdown

```java
// Application.java
@SpringBootApplication
public class OrderServiceApplication {
    public static void main(String[] args) {
        SpringApplication app = new SpringApplication(OrderServiceApplication.class);
        // Fast startup: lazy initialization
        app.setLazyInitialization(false); // true สำหรับ dev, false สำหรับ prod
        app.run(args);
    }
}
```

```yaml
# application.yml - Graceful Shutdown
server:
  shutdown: graceful  # รอให้ request ที่ค้างอยู่เสร็จก่อน shutdown

spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s  # รอสูงสุด 30 วินาที
```

**Factor X: Dev/prod parity** - Dev, Staging, Prod ใกล้เคียงกันมากที่สุด

```yaml
# docker-compose.yml - Dev environment ใช้ infrastructure เดียวกันกับ Prod
services:
  app:
    build: .
    depends_on:
      - postgres
      - redis
      - kafka

  postgres:
    image: postgres:15  # Same version กับ prod
    ports:
      - "5432:5432"

  redis:
    image: redis:7-alpine  # Same version กับ prod
    ports:
      - "6379:6379"
```

---

## ขั้นตอนที่ 2407-2413: Health Checks - Liveness vs Readiness vs Startup

### ความแตกต่างระหว่าง Health Check Types

```
Startup Probe:   "Application เริ่มต้นสำเร็จแล้วหรือยัง?"
                 → ถ้า fail: Kubernetes restart container
                 → ใช้เวลาเริ่มต้น Spring Boot อาจนาน

Liveness Probe:  "Application ยังทำงานอยู่ไหม (ไม่ deadlock/crash)?"
                 → ถ้า fail: Kubernetes restart container
                 → ควร check เฉพาะ basic health

Readiness Probe: "Application พร้อมรับ traffic แล้วหรือยัง?"
                 → ถ้า fail: Kubernetes หยุดส่ง traffic ไปยัง pod นี้
                 → ควร check dependencies ด้วย (DB, Cache)
```

### Custom Health Indicators

```java
// DatabaseHealthIndicator.java
@Component
public class DatabaseHealthIndicator implements HealthIndicator {

    private final DataSource dataSource;

    public DatabaseHealthIndicator(DataSource dataSource) {
        this.dataSource = dataSource;
    }

    @Override
    public Health health() {
        try (Connection conn = dataSource.getConnection()) {
            if (conn.isValid(3)) {  // timeout 3 seconds
                return Health.up()
                    .withDetail("database", "PostgreSQL")
                    .withDetail("status", "Connected")
                    .build();
            } else {
                return Health.down()
                    .withDetail("database", "PostgreSQL")
                    .withDetail("error", "Connection invalid")
                    .build();
            }
        } catch (Exception e) {
            return Health.down()
                .withDetail("database", "PostgreSQL")
                .withException(e)
                .build();
        }
    }
}

// KafkaHealthIndicator.java
@Component
public class KafkaHealthIndicator implements HealthIndicator {

    private final KafkaAdmin kafkaAdmin;

    @Override
    public Health health() {
        try {
            Map<String, Object> description = kafkaAdmin.describeTopics("order-events");
            return Health.up()
                .withDetail("kafka", "Connected")
                .withDetail("topics", description.keySet())
                .build();
        } catch (Exception e) {
            return Health.down()
                .withDetail("kafka", "Disconnected")
                .withException(e)
                .build();
        }
    }
}

// ReadinessHealthIndicator.java - ตรวจสอบ dependencies ทั้งหมด
@Component
public class ApplicationReadinessIndicator implements HealthIndicator {

    private final OrderRepository orderRepository;
    private final RedisTemplate<String, Object> redisTemplate;
    private volatile boolean applicationReady = false;

    @EventListener(ApplicationReadyEvent.class)
    public void onApplicationReady() {
        applicationReady = true;
    }

    @Override
    public Health health() {
        if (!applicationReady) {
            return Health.outOfService()
                .withDetail("reason", "Application not ready yet")
                .build();
        }

        Map<String, Object> details = new LinkedHashMap<>();
        boolean allHealthy = true;

        // ตรวจสอบ database
        try {
            orderRepository.count(); // Simple query
            details.put("database", "UP");
        } catch (Exception e) {
            details.put("database", "DOWN: " + e.getMessage());
            allHealthy = false;
        }

        // ตรวจสอบ Redis
        try {
            redisTemplate.opsForValue().get("health-check");
            details.put("redis", "UP");
        } catch (Exception e) {
            details.put("redis", "DOWN: " + e.getMessage());
            allHealthy = false;
        }

        return allHealthy
            ? Health.up().withDetails(details).build()
            : Health.down().withDetails(details).build();
    }
}
```

### Actuator Configuration

```yaml
# application.yml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus,readiness,liveness
      base-path: /actuator

  endpoint:
    health:
      show-details: when_authorized  # prod: never หรือ when_authorized
      show-components: always
      probes:
        enabled: true  # เปิด /actuator/health/liveness และ /actuator/health/readiness

  health:
    livenessstate:
      enabled: true
    readinessstate:
      enabled: true
    # เพิ่ม health indicators ที่ต้องการ
    db:
      enabled: true
    redis:
      enabled: true
    kafka:
      enabled: true
```

---

## ขั้นตอนที่ 2414-2420: Kubernetes Deployment

### Complete Kubernetes Manifests

```yaml
# k8s/namespace.yml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    app: order-service
    environment: production

---
# k8s/configmap.yml
apiVersion: v1
kind: ConfigMap
metadata:
  name: order-service-config
  namespace: production
data:
  SPRING_PROFILES_ACTIVE: "prod"
  SPRING_JPA_HIBERNATE_DDL_AUTO: "validate"
  SERVER_PORT: "8080"
  MANAGEMENT_SERVER_PORT: "8081"

---
# k8s/secret.yml
apiVersion: v1
kind: Secret
metadata:
  name: order-service-secrets
  namespace: production
type: Opaque
stringData:
  DATABASE_URL: "jdbc:postgresql://postgres-service:5432/orders"
  DB_USERNAME: "order_service"
  DB_PASSWORD: "your-secret-password"
  REDIS_HOST: "redis-service"
  JWT_SECRET: "your-jwt-secret-key"
  KAFKA_BOOTSTRAP_SERVERS: "kafka-service:9092"

---
# k8s/deployment.yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  namespace: production
  labels:
    app: order-service
    version: "1.0.0"
spec:
  replicas: 3
  selector:
    matchLabels:
      app: order-service
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1          # เพิ่มได้สูงสุด 1 pod ระหว่าง update
      maxUnavailable: 0    # ไม่ให้มี pod ที่ไม่พร้อมเลย
  template:
    metadata:
      labels:
        app: order-service
        version: "1.0.0"
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8081"
        prometheus.io/path: "/actuator/prometheus"
    spec:
      serviceAccountName: order-service-sa
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 2000

      # Grace period สำหรับ shutdown
      terminationGracePeriodSeconds: 60

      containers:
        - name: order-service
          image: your-registry/order-service:1.0.0
          imagePullPolicy: Always
          ports:
            - name: http
              containerPort: 8080
            - name: management
              containerPort: 8081

          # Environment variables จาก ConfigMap และ Secret
          envFrom:
            - configMapRef:
                name: order-service-config
            - secretRef:
                name: order-service-secrets

          # Resource limits
          resources:
            requests:
              memory: "512Mi"
              cpu: "250m"
            limits:
              memory: "1Gi"
              cpu: "1000m"

          # Startup Probe - รอให้ application start สำเร็จ
          startupProbe:
            httpGet:
              path: /actuator/health/liveness
              port: management
            initialDelaySeconds: 30    # รอ 30 วินาทีก่อน probe
            periodSeconds: 10
            failureThreshold: 12       # รอสูงสุด 30 + 12*10 = 150 วินาที
            successThreshold: 1

          # Liveness Probe - ตรวจสอบว่า app ยังทำงานอยู่
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: management
            initialDelaySeconds: 0    # Startup probe จัดการแล้ว
            periodSeconds: 20
            failureThreshold: 3
            successThreshold: 1
            timeoutSeconds: 5

          # Readiness Probe - ตรวจสอบว่า app พร้อมรับ traffic
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: management
            initialDelaySeconds: 0
            periodSeconds: 10
            failureThreshold: 3
            successThreshold: 1
            timeoutSeconds: 5

          # Lifecycle hooks
          lifecycle:
            preStop:
              exec:
                # รอ 15 วินาทีเพื่อให้ load balancer หยุดส่ง traffic มา
                command: ["/bin/sh", "-c", "sleep 15"]

      # Anti-affinity: กระจาย pods ไปยัง nodes ต่างกัน
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchExpressions:
                    - key: app
                      operator: In
                      values:
                        - order-service
                topologyKey: kubernetes.io/hostname

---
# k8s/service.yml
apiVersion: v1
kind: Service
metadata:
  name: order-service
  namespace: production
spec:
  selector:
    app: order-service
  ports:
    - name: http
      port: 80
      targetPort: 8080
      protocol: TCP
  type: ClusterIP
```

---

## ขั้นตอนที่ 2421-2427: Service Discovery กับ Kubernetes DNS

### Kubernetes DNS-based Service Discovery

```java
// Kubernetes DNS ทำงานอย่างไร:
// service-name.namespace.svc.cluster.local
//
// ตัวอย่าง:
// order-service.production.svc.cluster.local:80
// payment-service.production.svc.cluster.local:80
// redis.production.svc.cluster.local:6379
```

```yaml
# application-prod.yml - ใช้ Kubernetes DNS
spring:
  datasource:
    url: jdbc:postgresql://postgres-service.production.svc.cluster.local:5432/orders

  redis:
    host: redis-service.production.svc.cluster.local
    port: 6379

  kafka:
    bootstrap-servers: kafka-service.production.svc.cluster.local:9092

# Service ใน namespace เดียวกัน ใช้แค่ service name
payment:
  service:
    url: http://payment-service/api
    # Kubernetes resolves เป็น payment-service.production.svc.cluster.local
```

### RestClient กับ Service Discovery

```java
// OrderServiceClient.java
@Service
public class OrderServiceClient {

    private final RestClient restClient;

    public OrderServiceClient(RestClient.Builder builder) {
        this.restClient = builder
            .baseUrl("http://payment-service")  // Kubernetes DNS
            .defaultHeader("Content-Type", MediaType.APPLICATION_JSON_VALUE)
            .build();
    }

    public PaymentResponse processPayment(PaymentRequest request) {
        return restClient.post()
            .uri("/api/payments")
            .body(request)
            .retrieve()
            .onStatus(HttpStatusCode::is4xxClientError, (req, res) -> {
                throw new PaymentClientException("Payment failed: " + res.getStatusCode());
            })
            .onStatus(HttpStatusCode::is5xxServerError, (req, res) -> {
                throw new PaymentServiceException("Payment service error: " + res.getStatusCode());
            })
            .body(PaymentResponse.class);
    }
}

// LoadBalancer Configuration (Spring Cloud LoadBalancer)
@Configuration
public class LoadBalancerConfig {

    @Bean
    @LoadBalanced  // เพิ่ม client-side load balancing
    public RestTemplate restTemplate() {
        RestTemplate template = new RestTemplate();
        template.setInterceptors(List.of(new CorrelationIdInterceptor()));
        return template;
    }
}
```

---

## ขั้นตอนที่ 2428-2434: Horizontal Pod Autoscaler (HPA)

### HPA กับ Standard Metrics

```yaml
# k8s/hpa.yml - Horizontal Pod Autoscaler
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: order-service-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-service
  minReplicas: 3
  maxReplicas: 20

  metrics:
    # Scale ตาม CPU usage
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70  # scale เมื่อ CPU > 70%

    # Scale ตาม Memory usage
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80

  behavior:
    # Scale Up: เร็ว (traffic spike)
    scaleUp:
      stabilizationWindowSeconds: 60  # รอ 60 วินาทีก่อน scale up
      policies:
        - type: Pods
          value: 4          # เพิ่มได้สูงสุด 4 pods ต่อครั้ง
          periodSeconds: 60
        - type: Percent
          value: 100        # หรือเพิ่ม 100% ต่อครั้ง
          periodSeconds: 60
      selectPolicy: Max     # ใช้ policy ที่ scale ได้มากกว่า

    # Scale Down: ช้า (ป้องกัน flapping)
    scaleDown:
      stabilizationWindowSeconds: 300  # รอ 5 นาทีก่อน scale down
      policies:
        - type: Pods
          value: 2          # ลดได้สูงสุด 2 pods ต่อครั้ง
          periodSeconds: 60
```

### HPA กับ Custom Metrics (Prometheus)

```yaml
# ต้องติดตั้ง Prometheus Adapter ก่อน
# k8s/hpa-custom-metrics.yml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: order-service-hpa-custom
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-service

  minReplicas: 3
  maxReplicas: 30

  metrics:
    # Custom metric: Requests per second
    - type: Pods
      pods:
        metric:
          name: http_requests_per_second
        target:
          type: AverageValue
          averageValue: "100"  # Scale เมื่อ RPS > 100 ต่อ pod

    # External metric: Queue length
    - type: External
      external:
        metric:
          name: kafka_consumer_lag
          selector:
            matchLabels:
              topic: order-events
        target:
          type: Value
          value: "1000"  # Scale เมื่อ lag > 1000 messages
```

### Expose Custom Metrics จาก Spring Boot

```java
// OrderMetrics.java - Expose metrics สำหรับ HPA
@Component
public class OrderMetrics {

    private final MeterRegistry meterRegistry;
    private final AtomicInteger pendingOrders = new AtomicInteger(0);

    public OrderMetrics(MeterRegistry meterRegistry, OrderRepository orderRepository) {
        this.meterRegistry = meterRegistry;

        // Gauge: จำนวน pending orders (สำหรับ HPA decision)
        Gauge.builder("order.pending.count", pendingOrders, AtomicInteger::get)
            .description("Number of pending orders")
            .register(meterRegistry);

        // Schedule: update gauge ทุก 30 วินาที
        Executors.newScheduledThreadPool(1)
            .scheduleAtFixedRate(() -> {
                long count = orderRepository.countByStatus(OrderStatus.PENDING);
                pendingOrders.set((int) count);
            }, 0, 30, TimeUnit.SECONDS);
    }

    // Counter: นับ orders ที่สร้าง
    public void recordOrderCreated(Order order) {
        meterRegistry.counter("order.created.total",
            "currency", order.getCurrency(),
            "status", order.getStatus().name()
        ).increment();
    }

    // Timer: วัดเวลา processing
    public void recordOrderProcessingTime(long durationMs) {
        meterRegistry.timer("order.processing.duration")
            .record(durationMs, TimeUnit.MILLISECONDS);
    }

    // Histogram: กระจาย order amounts
    public void recordOrderAmount(BigDecimal amount) {
        meterRegistry.summary("order.amount")
            .record(amount.doubleValue());
    }
}
```

### Prometheus Rules สำหรับ Custom Metrics

```yaml
# prometheus-adapter-config.yml
rules:
  - seriesQuery: 'http_server_requests_seconds_count{namespace!="",pod!=""}'
    resources:
      overrides:
        namespace: {resource: "namespace"}
        pod: {resource: "pod"}
    name:
      matches: "^(.*)_total$"
      as: "http_requests_per_second"
    metricsQuery: 'rate(http_server_requests_seconds_count{<<.LabelMatchers>>}[2m])'

  - seriesQuery: 'order_pending_count{namespace!="",pod!=""}'
    resources:
      overrides:
        namespace: {resource: "namespace"}
        pod: {resource: "pod"}
    name:
      as: "order_pending_count"
    metricsQuery: 'order_pending_count{<<.LabelMatchers>>}'
```

---

## ขั้นตอนที่ 2435-2440: Cloud Native Design Patterns

### Circuit Breaker Pattern

```java
// PaymentServiceClient.java - Circuit Breaker กับ Resilience4j
@Service
public class PaymentServiceClient {

    private final RestTemplate restTemplate;
    private final CircuitBreakerRegistry circuitBreakerRegistry;

    @CircuitBreaker(
        name = "paymentService",
        fallbackMethod = "processPaymentFallback"
    )
    @Retry(name = "paymentService")
    @TimeLimiter(name = "paymentService")
    @Bulkhead(name = "paymentService")
    public CompletableFuture<PaymentResponse> processPayment(PaymentRequest request) {
        return CompletableFuture.supplyAsync(() ->
            restTemplate.postForObject(
                "http://payment-service/api/payments",
                request,
                PaymentResponse.class
            )
        );
    }

    // Fallback เมื่อ Circuit Breaker เปิด
    public CompletableFuture<PaymentResponse> processPaymentFallback(
        PaymentRequest request, Exception e
    ) {
        log.error("Payment service unavailable, using fallback. Cause: {}", e.getMessage());

        // Queue ไว้สำหรับ retry ทีหลัง
        paymentQueueService.enqueueForRetry(request);

        return CompletableFuture.completedFuture(
            PaymentResponse.pending(request.getOrderId(), "Payment queued for processing")
        );
    }
}
```

```yaml
# application.yml - Resilience4j Configuration
resilience4j:
  circuitbreaker:
    instances:
      paymentService:
        slidingWindowSize: 10
        permittedNumberOfCallsInHalfOpenState: 3
        slidingWindowType: COUNT_BASED
        minimumNumberOfCalls: 5
        waitDurationInOpenState: 10s
        failureRateThreshold: 50
        eventConsumerBufferSize: 10

  retry:
    instances:
      paymentService:
        maxAttempts: 3
        waitDuration: 500ms
        exponentialBackoffMultiplier: 2
        retryExceptions:
          - org.springframework.web.client.HttpServerErrorException
          - java.io.IOException

  timelimiter:
    instances:
      paymentService:
        timeoutDuration: 3s
        cancelRunningFuture: true

  bulkhead:
    instances:
      paymentService:
        maxConcurrentCalls: 10
        maxWaitDuration: 500ms
```

### CQRS Pattern กับ Spring

```java
// Command side
@Component
public class CreateOrderCommandHandler {

    private final OrderRepository orderRepository;
    private final ApplicationEventPublisher eventPublisher;

    @Transactional
    public OrderId handle(CreateOrderCommand command) {
        Order order = Order.create(
            command.getCustomerId(),
            command.getItems()
        );

        orderRepository.save(order);
        eventPublisher.publishEvent(new OrderCreatedEvent(order));

        return order.getId();
    }
}

// Query side - Read model
@Component
public class GetOrderQueryHandler {

    private final OrderReadModelRepository readRepository;

    @Transactional(readOnly = true)
    public OrderReadModel handle(GetOrderQuery query) {
        return readRepository.findById(query.getOrderId())
            .orElseThrow(() -> new OrderNotFoundException(query.getOrderId()));
    }
}

// Event Handler สำหรับ update Read Model
@Component
public class OrderEventHandler {

    private final OrderReadModelRepository readRepository;

    @EventListener
    @Async
    public void onOrderCreated(OrderCreatedEvent event) {
        Order order = event.getOrder();

        OrderReadModel readModel = OrderReadModel.builder()
            .id(order.getId())
            .customerId(order.getCustomerId())
            .status(order.getStatus().name())
            .totalAmount(order.getTotalAmount())
            .itemCount(order.getItems().size())
            .createdAt(order.getCreatedAt())
            .build();

        readRepository.save(readModel);
    }
}
```

### Sidecar Pattern กับ Kubernetes

```yaml
# k8s/deployment-with-sidecar.yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  template:
    spec:
      containers:
        # Main application container
        - name: order-service
          image: order-service:latest
          ports:
            - containerPort: 8080

        # Sidecar: Log shipper (Filebeat)
        - name: filebeat
          image: docker.elastic.co/beats/filebeat:8.11.0
          volumeMounts:
            - name: logs
              mountPath: /app/logs
            - name: filebeat-config
              mountPath: /usr/share/filebeat/filebeat.yml
              subPath: filebeat.yml

        # Sidecar: Metrics scraper (cAdvisor)
        - name: jaeger-agent
          image: jaegertracing/jaeger-agent:latest
          args:
            - "--reporter.grpc.host-port=jaeger-collector:14250"
          ports:
            - containerPort: 6831
              protocol: UDP

      volumes:
        - name: logs
          emptyDir: {}
        - name: filebeat-config
          configMap:
            name: filebeat-config
```

### Graceful Shutdown Implementation

```java
// GracefulShutdownConfig.java
@Configuration
public class GracefulShutdownConfig {

    @Bean
    public GracefulShutdown gracefulShutdown() {
        return new GracefulShutdown();
    }

    @Bean
    public WebServerFactoryCustomizer<TomcatServletWebServerFactory> webServerFactoryCustomizer(
        GracefulShutdown gracefulShutdown
    ) {
        return factory -> factory.addConnectorCustomizers(gracefulShutdown);
    }
}

// GracefulShutdown.java
public class GracefulShutdown implements TomcatConnectorCustomizer, ApplicationListener<ContextClosedEvent> {

    private static final Logger log = LoggerFactory.getLogger(GracefulShutdown.class);
    private volatile Connector connector;

    @Override
    public void customize(Connector connector) {
        this.connector = connector;
    }

    @Override
    public void onApplicationEvent(ContextClosedEvent event) {
        this.connector.pause();

        Executor executor = this.connector.getProtocolHandler().getExecutor();
        if (executor instanceof ThreadPoolExecutor) {
            ThreadPoolExecutor threadPoolExecutor = (ThreadPoolExecutor) executor;
            threadPoolExecutor.shutdown();

            try {
                if (!threadPoolExecutor.awaitTermination(30, TimeUnit.SECONDS)) {
                    log.warn("Tomcat thread pool did not shut down gracefully within 30 seconds");
                    threadPoolExecutor.shutdownNow();
                }
            } catch (InterruptedException ex) {
                Thread.currentThread().interrupt();
                threadPoolExecutor.shutdownNow();
            }
        }
    }
}
```

---

## สรุป Part 70: Cloud Native

### Cloud Native Checklist

- [ ] Application ตาม 12-Factor App principles
- [ ] Stateless design (state อยู่ใน external store)
- [ ] Config ผ่าน environment variables
- [ ] Structured logging พร้อม correlation IDs
- [ ] Health checks ครบทั้ง 3 ประเภท (Startup, Liveness, Readiness)
- [ ] Graceful shutdown สำหรับ zero-downtime deployment
- [ ] Resource limits กำหนดใน Kubernetes
- [ ] HPA สำหรับ auto-scaling
- [ ] Circuit Breaker สำหรับ external dependencies
- [ ] Container image ใช้ non-root user

### Cloud Native Architecture Diagram

```
┌─────────────────────────────────────────────────────┐
│                   Kubernetes Cluster                │
│                                                     │
│  ┌──────────────┐    ┌──────────────────────────┐  │
│  │   Ingress    │───▶│    order-service pods    │  │
│  │  Controller  │    │  ┌────┐ ┌────┐ ┌────┐   │  │
│  └──────────────┘    │  │Pod1│ │Pod2│ │Pod3│   │  │
│                      │  └────┘ └────┘ └────┘   │  │
│  ┌──────────────┐    │        HPA (3-20)        │  │
│  │    Vault     │    └──────────────────────────┘  │
│  │  (Secrets)   │               │                  │
│  └──────────────┘    ┌──────────▼─────────┐        │
│                      │  Backing Services   │        │
│  ┌──────────────┐    │  ┌─────┐ ┌───────┐ │        │
│  │   Config     │    │  │ DB  │ │ Redis │ │        │
│  │   Server     │    │  └─────┘ └───────┘ │        │
│  └──────────────┘    │  ┌─────┐ ┌───────┐ │        │
│                      │  │Kafka│ │  ELK  │ │        │
│                      │  └─────┘ └───────┘ │        │
│                      └────────────────────┘        │
└─────────────────────────────────────────────────────┘
```

---

*[← Part 69: Database Advanced](./part-69-database-advanced.md) | [Part 71: Advanced Observability →](./part-71-advanced-observability.md)*

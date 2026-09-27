# Part 71: Observability - Three Pillars of Production Monitoring
## ขั้นตอนที่ 2441-2480

**ระดับ:** ระดับโลก (World-Class)
**เวลาเรียน:** 5-6 ชั่วโมง
**เป้าหมาย:** เรียนรู้การสร้างระบบ Observability ครบวงจรด้วย Metrics, Tracing และ Logging พร้อม Prometheus, Grafana และ Alert Rules สำหรับระบบ Production

---

## สารบัญ

1. [Three Pillars of Observability คืออะไร](#three-pillars)
2. [Micrometer Observation API ใน Spring Boot 3](#micrometer-observation)
3. [Custom @Observed Annotation](#custom-observed)
4. [Prometheus Metrics Configuration](#prometheus)
5. [Grafana Dashboard Setup](#grafana)
6. [Alert Rules](#alert-rules)
7. [On-Call Runbooks](#runbooks)

---

## ขั้นตอนที่ 2441: Three Pillars of Observability {#three-pillars}

### ทำไม Observability ถึงสำคัญ?

ในระบบ Production จริง เราไม่สามารถรู้ล่วงหน้าได้ว่าปัญหาจะเกิดขึ้นที่ไหน Observability ช่วยให้เราสามารถ "มองเห็น" สิ่งที่เกิดขึ้นภายในระบบโดยไม่ต้องแก้ไขโค้ดใหม่

**3 เสาหลักของ Observability:**

1. **Metrics (ตัวชี้วัด)** - ตัวเลขที่วัดค่าต่างๆ ของระบบในช่วงเวลาหนึ่ง เช่น จำนวน request per second, latency, error rate
2. **Tracing (การติดตาม)** - การติดตาม request ข้ามหลาย service เพื่อเห็นว่า request ผ่านที่ไหนบ้าง ใช้เวลาเท่าไหร่
3. **Logging (การบันทึก)** - การบันทึก event ที่เกิดขึ้น พร้อม context สำหรับการ debug

```
Request → [Service A] → [Service B] → [Database]
              ↓              ↓              ↓
           Metrics         Trace         Logs
        (latency: 50ms)  (span: B→DB)  (SQL query)
```

### Maven Dependencies

```xml
<!-- pom.xml -->
<dependencies>
    <!-- Spring Boot Actuator - สำหรับ metrics endpoint -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
    
    <!-- Micrometer Prometheus Registry -->
    <dependency>
        <groupId>io.micrometer</groupId>
        <artifactId>micrometer-registry-prometheus</artifactId>
    </dependency>
    
    <!-- Micrometer Tracing Bridge สำหรับ OpenTelemetry -->
    <dependency>
        <groupId>io.micrometer</groupId>
        <artifactId>micrometer-tracing-bridge-otel</artifactId>
    </dependency>
    
    <!-- OpenTelemetry Exporter -->
    <dependency>
        <groupId>io.opentelemetry</groupId>
        <artifactId>opentelemetry-exporter-otlp</artifactId>
    </dependency>
    
    <!-- Loki Logback Appender สำหรับ log shipping -->
    <dependency>
        <groupId>com.github.loki4j</groupId>
        <artifactId>loki-logback-appender</artifactId>
        <version>1.4.2</version>
    </dependency>
    
    <!-- AOP สำหรับ @Observed annotation -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-aop</artifactId>
    </dependency>
</dependencies>
```

---

## ขั้นตอนที่ 2442: การตั้งค่า Application Properties

### application.yml

```yaml
# application.yml
spring:
  application:
    name: my-spring-app

# Actuator configuration
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus,loggers,env
  endpoint:
    health:
      show-details: always
      probes:
        enabled: true
    prometheus:
      enabled: true
  metrics:
    tags:
      # Labels ที่จะติดไปกับทุก metric
      application: ${spring.application.name}
      environment: ${ENVIRONMENT:local}
      region: ${REGION:us-east-1}
    distribution:
      percentiles-histogram:
        # เปิด histogram สำหรับ HTTP requests
        http.server.requests: true
      percentiles:
        http.server.requests: 0.5, 0.75, 0.95, 0.99
      slo:
        # กำหนด SLO buckets สำหรับ latency
        http.server.requests: 10ms, 50ms, 100ms, 200ms, 500ms, 1s, 5s
  tracing:
    sampling:
      probability: 1.0  # sample ทุก request ใน dev, ใช้ 0.1 ใน prod
    enabled: true

# Logging configuration
logging:
  pattern:
    level: "%5p [${spring.application.name:},%X{traceId:-},%X{spanId:-}]"
```

---

## ขั้นตอนที่ 2443: Micrometer Observation API {#micrometer-observation}

### ความแตกต่างจาก Micrometer เดิม

Micrometer Observation API เป็น API ใหม่ใน Spring Boot 3 ที่รวม Metrics และ Tracing เข้าด้วยกันในที่เดียว แทนที่จะต้องเรียกทั้ง `Timer` และ `Tracer` แยกกัน

```java
// แบบเก่า - ต้องทำแยก
Timer timer = meterRegistry.timer("my.operation");
Span span = tracer.nextSpan().name("my.operation").start();
try {
    timer.record(() -> doSomething());
} finally {
    span.end();
}

// แบบใหม่ - Observation API รวมทุกอย่างไว้ที่เดียว
Observation.createNotStarted("my.operation", observationRegistry)
    .observe(() -> doSomething());
```

### Basic Observation Usage

```java
package com.example.observability;

import io.micrometer.observation.Observation;
import io.micrometer.observation.ObservationRegistry;
import org.springframework.stereotype.Service;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;

@Service
@RequiredArgsConstructor
@Slf4j
public class OrderService {

    private final ObservationRegistry observationRegistry;
    private final OrderRepository orderRepository;

    public Order createOrder(CreateOrderRequest request) {
        // สร้าง Observation พร้อม context
        return Observation.createNotStarted("order.create", observationRegistry)
            .contextualName("Create Order")  // ชื่อที่แสดงใน trace
            .lowCardinalityKeyValue("order.type", request.getType())
            .highCardinalityKeyValue("customer.id", request.getCustomerId())
            .observe(() -> {
                log.info("Creating order for customer: {}", request.getCustomerId());
                
                Order order = Order.builder()
                    .customerId(request.getCustomerId())
                    .items(request.getItems())
                    .status(OrderStatus.PENDING)
                    .build();
                
                return orderRepository.save(order);
            });
    }

    public Order getOrder(Long orderId) {
        // Observation พร้อม error handling
        Observation observation = Observation.createNotStarted("order.get", observationRegistry)
            .lowCardinalityKeyValue("operation", "read");
        
        observation.start();
        try {
            Order order = orderRepository.findById(orderId)
                .orElseThrow(() -> new OrderNotFoundException("Order not found: " + orderId));
            observation.stop();
            return order;
        } catch (Exception e) {
            observation.error(e);
            observation.stop();
            throw e;
        }
    }
}
```

### Observation Context สำหรับ Custom Metadata

```java
package com.example.observability;

import io.micrometer.observation.Observation;

// Custom context เพื่อเก็บข้อมูลเพิ่มเติม
public class OrderObservationContext extends Observation.Context {
    
    private String orderId;
    private String customerId;
    private String orderType;
    private int itemCount;
    
    public OrderObservationContext(String orderId, String customerId, 
                                    String orderType, int itemCount) {
        this.orderId = orderId;
        this.customerId = customerId;
        this.orderType = orderType;
        this.itemCount = itemCount;
    }
    
    // Getters
    public String getOrderId() { return orderId; }
    public String getCustomerId() { return customerId; }
    public String getOrderType() { return orderType; }
    public int getItemCount() { return itemCount; }
}
```

### Observation Convention สำหรับ Standardize Naming

```java
package com.example.observability;

import io.micrometer.common.KeyValues;
import io.micrometer.observation.Observation;
import io.micrometer.observation.ObservationConvention;

// กำหนด convention สำหรับ naming และ key-values ที่ consistent
public class OrderObservationConvention 
    implements ObservationConvention<OrderObservationContext> {

    @Override
    public boolean supportsContext(Observation.Context context) {
        return context instanceof OrderObservationContext;
    }

    @Override
    public String getName() {
        return "order.operation";
    }

    @Override
    public String getContextualName(OrderObservationContext context) {
        return "Order " + context.getOrderType();
    }

    @Override
    public KeyValues getLowCardinalityKeyValues(OrderObservationContext context) {
        return KeyValues.of(
            "order.type", context.getOrderType(),
            "item.count.bucket", getItemCountBucket(context.getItemCount())
        );
    }

    @Override
    public KeyValues getHighCardinalityKeyValues(OrderObservationContext context) {
        return KeyValues.of(
            "order.id", context.getOrderId(),
            "customer.id", context.getCustomerId()
        );
    }

    private String getItemCountBucket(int count) {
        if (count == 1) return "single";
        if (count <= 5) return "small";
        if (count <= 20) return "medium";
        return "large";
    }
}
```

---

## ขั้นตอนที่ 2444: Custom @Observed Annotation {#custom-observed}

### ใช้ @Observed ที่มาพร้อม Spring Boot 3

Spring Boot 3 มี `@Observed` annotation ที่ใช้งานได้ทันที โดยต้องเพิ่ม AOP dependency และ Bean configuration:

```java
package com.example.config;

import io.micrometer.observation.ObservationRegistry;
import io.micrometer.observation.aop.ObservedAspect;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class ObservabilityConfig {

    // Bean นี้จำเป็นสำหรับการใช้ @Observed annotation
    @Bean
    public ObservedAspect observedAspect(ObservationRegistry observationRegistry) {
        return new ObservedAspect(observationRegistry);
    }
}
```

### การใช้ @Observed

```java
package com.example.service;

import io.micrometer.observation.annotation.Observed;
import org.springframework.stereotype.Service;

@Service
public class PaymentService {

    // @Observed จะสร้าง metric และ trace โดยอัตโนมัติ
    @Observed(
        name = "payment.process",
        contextualName = "Process Payment",
        lowCardinalityKeyValues = {"payment.method", "credit_card"}
    )
    public PaymentResult processPayment(PaymentRequest request) {
        // logic ในนี้จะถูก observe โดยอัตโนมัติ
        validatePayment(request);
        return chargeCard(request);
    }

    @Observed(name = "payment.refund")
    public RefundResult refundPayment(String paymentId, BigDecimal amount) {
        return processRefund(paymentId, amount);
    }
    
    private void validatePayment(PaymentRequest request) {
        // validation logic
    }
    
    private PaymentResult chargeCard(PaymentRequest request) {
        // charge logic
        return new PaymentResult("SUCCESS", request.getAmount());
    }
    
    private RefundResult processRefund(String paymentId, BigDecimal amount) {
        return new RefundResult("REFUNDED", amount);
    }
}
```

### Custom @Observed Annotation พร้อม Business Context

```java
package com.example.observability;

import io.micrometer.observation.annotation.Observed;
import java.lang.annotation.*;

// สร้าง meta-annotation ที่ครอบ @Observed อีกชั้น
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Observed
public @interface BusinessOperation {
    String value() default "";
    String domain() default "general";
    String[] tags() default {};
}
```

```java
package com.example.service;

import com.example.observability.BusinessOperation;
import org.springframework.stereotype.Service;

@Service
public class InventoryService {

    @BusinessOperation(value = "inventory.check", domain = "warehouse")
    public StockLevel checkStock(String productId) {
        return getStockFromWarehouse(productId);
    }

    @BusinessOperation(value = "inventory.reserve", domain = "warehouse", 
                       tags = {"critical:true"})
    public boolean reserveStock(String productId, int quantity) {
        return performReservation(productId, quantity);
    }
    
    private StockLevel getStockFromWarehouse(String productId) {
        return new StockLevel(productId, 100);
    }
    
    private boolean performReservation(String productId, int quantity) {
        return true;
    }
}
```

---

## ขั้นตอนที่ 2445: Custom Metrics

### Counter, Gauge, Timer, DistributionSummary

```java
package com.example.metrics;

import io.micrometer.core.instrument.*;
import org.springframework.stereotype.Component;
import jakarta.annotation.PostConstruct;
import java.util.concurrent.atomic.AtomicInteger;

@Component
public class BusinessMetrics {

    private final MeterRegistry meterRegistry;
    
    // Counter - นับจำนวนที่เพิ่มขึ้นเรื่อยๆ
    private Counter ordersCreatedCounter;
    private Counter ordersFailedCounter;
    
    // Gauge - ค่าที่เปลี่ยนขึ้นลง
    private AtomicInteger activeConnections = new AtomicInteger(0);
    
    // Timer - วัดเวลา
    private Timer orderProcessingTimer;
    
    // DistributionSummary - วัดการกระจายของค่า
    private DistributionSummary orderValueSummary;

    public BusinessMetrics(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
    }

    @PostConstruct
    public void initMetrics() {
        // Counter
        ordersCreatedCounter = Counter.builder("orders.created.total")
            .description("Total number of orders created")
            .tag("version", "v2")
            .register(meterRegistry);

        ordersFailedCounter = Counter.builder("orders.failed.total")
            .description("Total number of failed orders")
            .register(meterRegistry);

        // Gauge
        Gauge.builder("connections.active", activeConnections, AtomicInteger::get)
            .description("Number of active connections")
            .register(meterRegistry);

        // Timer
        orderProcessingTimer = Timer.builder("order.processing.duration")
            .description("Time taken to process an order")
            .publishPercentiles(0.5, 0.95, 0.99)
            .publishPercentileHistogram()
            .register(meterRegistry);

        // DistributionSummary
        orderValueSummary = DistributionSummary.builder("order.value")
            .description("Distribution of order values in THB")
            .baseUnit("THB")
            .publishPercentiles(0.5, 0.75, 0.95)
            .scale(1.0)
            .register(meterRegistry);
    }

    public void recordOrderCreated(double orderValue) {
        ordersCreatedCounter.increment();
        orderValueSummary.record(orderValue);
    }

    public void recordOrderFailed() {
        ordersFailedCounter.increment();
    }

    public <T> T timeOrderProcessing(java.util.concurrent.Callable<T> operation) throws Exception {
        return orderProcessingTimer.recordCallable(operation);
    }

    public void connectionOpened() {
        activeConnections.incrementAndGet();
    }

    public void connectionClosed() {
        activeConnections.decrementAndGet();
    }
}
```

### Custom Health Indicators

```java
package com.example.health;

import org.springframework.boot.actuate.health.Health;
import org.springframework.boot.actuate.health.HealthIndicator;
import org.springframework.stereotype.Component;
import lombok.RequiredArgsConstructor;

@Component("paymentGateway")
@RequiredArgsConstructor
public class PaymentGatewayHealthIndicator implements HealthIndicator {

    private final PaymentGatewayClient gatewayClient;

    @Override
    public Health health() {
        try {
            GatewayStatus status = gatewayClient.checkStatus();
            
            if (status.isHealthy()) {
                return Health.up()
                    .withDetail("gateway", "Available")
                    .withDetail("latency_ms", status.getLatency())
                    .withDetail("version", status.getVersion())
                    .build();
            } else {
                return Health.down()
                    .withDetail("gateway", "Degraded")
                    .withDetail("error", status.getErrorMessage())
                    .build();
            }
        } catch (Exception e) {
            return Health.down()
                .withDetail("gateway", "Unreachable")
                .withException(e)
                .build();
        }
    }
}
```

---

## ขั้นตอนที่ 2450: Distributed Tracing Setup

### OpenTelemetry Configuration

```java
package com.example.config;

import io.opentelemetry.api.OpenTelemetry;
import io.opentelemetry.api.common.Attributes;
import io.opentelemetry.api.trace.propagation.W3CTraceContextPropagator;
import io.opentelemetry.context.propagation.ContextPropagators;
import io.opentelemetry.exporter.otlp.http.trace.OtlpHttpSpanExporter;
import io.opentelemetry.sdk.OpenTelemetrySdk;
import io.opentelemetry.sdk.resources.Resource;
import io.opentelemetry.sdk.trace.SdkTracerProvider;
import io.opentelemetry.sdk.trace.export.BatchSpanProcessor;
import io.opentelemetry.semconv.resource.attributes.ResourceAttributes;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class TracingConfig {

    @Value("${spring.application.name}")
    private String applicationName;

    @Value("${OTLP_ENDPOINT:http://localhost:4318}")
    private String otlpEndpoint;

    @Bean
    public OpenTelemetry openTelemetry() {
        // Resource คือ metadata ของ service เรา
        Resource resource = Resource.getDefault()
            .merge(Resource.create(Attributes.of(
                ResourceAttributes.SERVICE_NAME, applicationName,
                ResourceAttributes.DEPLOYMENT_ENVIRONMENT, 
                    System.getenv().getOrDefault("ENVIRONMENT", "local"),
                ResourceAttributes.SERVICE_VERSION, "1.0.0"
            )));

        // Exporter ส่ง trace ไปที่ Jaeger/Tempo
        OtlpHttpSpanExporter spanExporter = OtlpHttpSpanExporter.builder()
            .setEndpoint(otlpEndpoint + "/v1/traces")
            .build();

        // Tracer Provider
        SdkTracerProvider tracerProvider = SdkTracerProvider.builder()
            .addSpanProcessor(BatchSpanProcessor.builder(spanExporter).build())
            .setResource(resource)
            .build();

        return OpenTelemetrySdk.builder()
            .setTracerProvider(tracerProvider)
            .setPropagators(ContextPropagators.create(
                W3CTraceContextPropagator.getInstance()
            ))
            .build();
    }
}
```

### Logback Configuration พร้อม Trace Context

```xml
<!-- src/main/resources/logback-spring.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <include resource="org/springframework/boot/logging/logback/defaults.xml"/>

    <!-- Console Appender พร้อม trace ID -->
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>
                %d{yyyy-MM-dd HH:mm:ss.SSS} %highlight(%-5level) [%thread] 
                %cyan([traceId=%X{traceId:-none} spanId=%X{spanId:-none}])
                %logger{36} - %msg%n
            </pattern>
        </encoder>
    </appender>

    <!-- JSON Appender สำหรับ structured logging -->
    <appender name="JSON" class="ch.qos.logback.core.ConsoleAppender">
        <encoder class="net.logstash.logback.encoder.LogstashEncoder">
            <includedMdcKeys>traceId,spanId,userId,orderId</includedMdcKeys>
        </encoder>
    </appender>

    <!-- Loki Appender สำหรับส่ง log ไป Grafana Loki -->
    <appender name="LOKI" class="com.github.loki4j.logback.Loki4jAppender">
        <http>
            <url>http://loki:3100/loki/api/v1/push</url>
        </http>
        <format>
            <label>
                <pattern>
                    app=${spring.application.name},env=${ENVIRONMENT:local},
                    level=%level,traceId=%X{traceId:-none}
                </pattern>
            </label>
            <message>
                <pattern>
                    level=%level,logger=%logger,message=%msg,
                    traceId=%X{traceId:-none},spanId=%X{spanId:-none}
                </pattern>
            </message>
        </format>
    </appender>

    <root level="INFO">
        <appender-ref ref="CONSOLE"/>
        <appender-ref ref="LOKI"/>
    </root>

    <logger name="com.example" level="DEBUG"/>
</configuration>
```

---

## ขั้นตอนที่ 2455: Prometheus Configuration {#prometheus}

### Prometheus Scrape Config

```yaml
# prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s
  external_labels:
    monitor: 'spring-boot-monitor'

# Alertmanager configuration
alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - alertmanager:9093

# Load alert rules
rule_files:
  - "alert_rules/*.yml"

scrape_configs:
  # Prometheus scrape ตัวเอง
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  # Spring Boot application
  - job_name: 'spring-boot-app'
    metrics_path: '/actuator/prometheus'
    scrape_interval: 15s
    static_configs:
      - targets: 
          - 'app:8080'
    # Labels เพิ่มเติม
    relabel_configs:
      - source_labels: [__address__]
        target_label: instance
      - target_label: environment
        replacement: 'production'

  # Kubernetes service discovery
  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      # เอาเฉพาะ pod ที่มี annotation prometheus.io/scrape: "true"
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)
      - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
        action: replace
        regex: ([^:]+)(?::\d+)?;(\d+)
        replacement: $1:$2
        target_label: __address__
```

### Kubernetes Annotations สำหรับ Service Discovery

```yaml
# k8s-deployment.yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spring-boot-app
spec:
  template:
    metadata:
      annotations:
        # บอก Prometheus ให้ scrape pod นี้
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
        prometheus.io/path: "/actuator/prometheus"
    spec:
      containers:
        - name: app
          image: myapp:latest
          ports:
            - containerPort: 8080
          env:
            - name: ENVIRONMENT
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
```

---

## ขั้นตอนที่ 2460: Grafana Dashboard Setup {#grafana}

### Dashboard as Code (JSON)

```json
{
  "dashboard": {
    "id": null,
    "title": "Spring Boot Application Dashboard",
    "tags": ["spring-boot", "java"],
    "timezone": "browser",
    "schemaVersion": 38,
    "refresh": "30s",
    "panels": [
      {
        "id": 1,
        "title": "Request Rate (req/s)",
        "type": "stat",
        "gridPos": {"h": 4, "w": 6, "x": 0, "y": 0},
        "targets": [
          {
            "expr": "sum(rate(http_server_requests_seconds_count{application=\"$application\"}[1m]))",
            "legendFormat": "RPS"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "unit": "reqps",
            "thresholds": {
              "steps": [
                {"color": "green", "value": null},
                {"color": "yellow", "value": 100},
                {"color": "red", "value": 500}
              ]
            }
          }
        }
      },
      {
        "id": 2,
        "title": "Error Rate (%)",
        "type": "stat",
        "gridPos": {"h": 4, "w": 6, "x": 6, "y": 0},
        "targets": [
          {
            "expr": "sum(rate(http_server_requests_seconds_count{application=\"$application\",status=~\"5..\"}[1m])) / sum(rate(http_server_requests_seconds_count{application=\"$application\"}[1m])) * 100",
            "legendFormat": "Error %"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "unit": "percent",
            "thresholds": {
              "steps": [
                {"color": "green", "value": null},
                {"color": "yellow", "value": 1},
                {"color": "red", "value": 5}
              ]
            }
          }
        }
      },
      {
        "id": 3,
        "title": "P99 Latency (ms)",
        "type": "stat",
        "gridPos": {"h": 4, "w": 6, "x": 12, "y": 0},
        "targets": [
          {
            "expr": "histogram_quantile(0.99, sum(rate(http_server_requests_seconds_bucket{application=\"$application\"}[1m])) by (le)) * 1000",
            "legendFormat": "P99"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "unit": "ms",
            "thresholds": {
              "steps": [
                {"color": "green", "value": null},
                {"color": "yellow", "value": 200},
                {"color": "red", "value": 1000}
              ]
            }
          }
        }
      },
      {
        "id": 4,
        "title": "Request Latency Distribution",
        "type": "graph",
        "gridPos": {"h": 8, "w": 12, "x": 0, "y": 4},
        "targets": [
          {
            "expr": "histogram_quantile(0.50, sum(rate(http_server_requests_seconds_bucket{application=\"$application\"}[5m])) by (le)) * 1000",
            "legendFormat": "P50"
          },
          {
            "expr": "histogram_quantile(0.95, sum(rate(http_server_requests_seconds_bucket{application=\"$application\"}[5m])) by (le)) * 1000",
            "legendFormat": "P95"
          },
          {
            "expr": "histogram_quantile(0.99, sum(rate(http_server_requests_seconds_bucket{application=\"$application\"}[5m])) by (le)) * 1000",
            "legendFormat": "P99"
          }
        ],
        "yaxes": [{"format": "ms"}]
      }
    ],
    "templating": {
      "list": [
        {
          "name": "application",
          "type": "query",
          "query": "label_values(http_server_requests_seconds_count, application)",
          "refresh": 1
        },
        {
          "name": "environment",
          "type": "query",
          "query": "label_values(http_server_requests_seconds_count, environment)",
          "refresh": 1
        }
      ]
    }
  }
}
```

### Grafana Provisioning Configuration

```yaml
# grafana/provisioning/datasources/prometheus.yml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
    jsonData:
      httpMethod: POST
      exemplarTraceIdDestinations:
        - name: traceID
          datasourceUid: tempo

  - name: Loki
    type: loki
    access: proxy
    url: http://loki:3100
    jsonData:
      derivedFields:
        - datasourceUid: tempo
          matcherRegex: "traceId=(\\w+)"
          name: TraceID
          url: '$${__value.raw}'

  - name: Tempo
    uid: tempo
    type: tempo
    access: proxy
    url: http://tempo:3200
    jsonData:
      tracesToLogs:
        datasourceUid: loki
        tags: ['app', 'traceId']
```

---

## ขั้นตอนที่ 2465: Alert Rules {#alert-rules}

### Prometheus Alert Rules

```yaml
# alert_rules/spring-boot-alerts.yml
groups:
  - name: spring-boot-availability
    rules:
      # Application ไม่ตอบสนอง
      - alert: ApplicationDown
        expr: up{job="spring-boot-app"} == 0
        for: 1m
        labels:
          severity: critical
          team: platform
        annotations:
          summary: "Application {{ $labels.instance }} is down"
          description: |
            Application {{ $labels.instance }} has been down for more than 1 minute.
            Please check the application logs and container status.
          runbook: "https://runbook.company.com/app-down"
          dashboard: "https://grafana.company.com/d/abc123"

      # High Error Rate
      - alert: HighErrorRate
        expr: |
          sum(rate(http_server_requests_seconds_count{status=~"5.."}[5m]))
            by (application, instance)
          /
          sum(rate(http_server_requests_seconds_count[5m]))
            by (application, instance)
          > 0.05
        for: 2m
        labels:
          severity: warning
          team: backend
        annotations:
          summary: "High error rate for {{ $labels.application }}"
          description: |
            Error rate is {{ $value | humanizePercentage }} for {{ $labels.application }}.
            This exceeds the 5% threshold.
            Current value: {{ $value }}
          runbook: "https://runbook.company.com/high-error-rate"

      # Critical Error Rate
      - alert: CriticalErrorRate
        expr: |
          sum(rate(http_server_requests_seconds_count{status=~"5.."}[5m]))
            by (application)
          /
          sum(rate(http_server_requests_seconds_count[5m]))
            by (application)
          > 0.20
        for: 1m
        labels:
          severity: critical
          team: backend
        annotations:
          summary: "CRITICAL: Error rate > 20% for {{ $labels.application }}"
          description: |
            ERROR RATE IS {{ $value | humanizePercentage }}!
            Immediate action required for {{ $labels.application }}.
          runbook: "https://runbook.company.com/critical-error-rate"

  - name: spring-boot-latency
    rules:
      # High P95 Latency
      - alert: HighP95Latency
        expr: |
          histogram_quantile(0.95,
            sum(rate(http_server_requests_seconds_bucket[5m]))
            by (application, le)
          ) > 1.0
        for: 5m
        labels:
          severity: warning
          team: backend
        annotations:
          summary: "High P95 latency for {{ $labels.application }}"
          description: |
            P95 latency is {{ $value | humanizeDuration }} for {{ $labels.application }}.
            SLO threshold is 1 second.
          runbook: "https://runbook.company.com/high-latency"

      # Very High P99 Latency
      - alert: VeryHighP99Latency
        expr: |
          histogram_quantile(0.99,
            sum(rate(http_server_requests_seconds_bucket[5m]))
            by (application, le)
          ) > 5.0
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "P99 latency > 5s for {{ $labels.application }}"
          description: |
            P99 latency is {{ $value | humanizeDuration }}.
            Users are experiencing severe slowdowns.

  - name: spring-boot-jvm
    rules:
      # High Memory Usage
      - alert: HighMemoryUsage
        expr: |
          jvm_memory_used_bytes{area="heap"}
          /
          jvm_memory_max_bytes{area="heap"}
          > 0.85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High JVM heap usage for {{ $labels.application }}"
          description: |
            JVM heap usage is {{ $value | humanizePercentage }}.
            Consider increasing heap size or investigating memory leaks.

      # GC Pressure
      - alert: HighGCPressure
        expr: |
          rate(jvm_gc_pause_seconds_sum[5m]) > 0.1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High GC pressure in {{ $labels.application }}"
          description: |
            GC is consuming {{ $value | humanizePercentage }} of CPU time.
            This may impact application performance.

      # Thread Pool Saturation
      - alert: ThreadPoolSaturation
        expr: |
          hikaricp_connections_active / hikaricp_connections_max > 0.9
        for: 3m
        labels:
          severity: warning
        annotations:
          summary: "Database connection pool near saturation"
          description: |
            Connection pool utilization is {{ $value | humanizePercentage }}.
            Consider increasing pool size or optimizing queries.

  - name: spring-boot-slo
    rules:
      # SLO: Availability < 99.9%
      - alert: SLOAvailabilityBreach
        expr: |
          (
            1 - (
              sum(rate(http_server_requests_seconds_count{status=~"5.."}[1h]))
              /
              sum(rate(http_server_requests_seconds_count[1h]))
            )
          ) < 0.999
        labels:
          severity: critical
          slo: availability
        annotations:
          summary: "SLO breach: Availability below 99.9%"
          description: |
            Current availability: {{ $value | humanizePercentage }}.
            SLO target is 99.9%.
            Error budget is being consumed rapidly.
```

---

## ขั้นตอนที่ 2470: Alertmanager Configuration

### Alertmanager Routing

```yaml
# alertmanager.yml
global:
  resolve_timeout: 5m
  slack_api_url: 'https://hooks.slack.com/services/XXX/YYY/ZZZ'
  pagerduty_url: 'https://events.pagerduty.com/v2/enqueue'

# Templates สำหรับ notification messages
templates:
  - '/etc/alertmanager/templates/*.tmpl'

route:
  # Default route
  receiver: 'slack-notifications'
  group_by: ['alertname', 'cluster', 'service']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 12h

  routes:
    # Critical alerts -> PagerDuty + Slack
    - match:
        severity: critical
      receiver: 'pagerduty-critical'
      continue: true

    - match:
        severity: critical
      receiver: 'slack-critical'

    # Backend team alerts
    - match:
        team: backend
      receiver: 'slack-backend-team'

    # SLO breaches -> separate channel
    - match:
        slo: availability
      receiver: 'slack-slo-channel'

receivers:
  - name: 'slack-notifications'
    slack_configs:
      - channel: '#alerts-general'
        title: '{{ template "slack.title" . }}'
        text: '{{ template "slack.text" . }}'
        send_resolved: true

  - name: 'slack-critical'
    slack_configs:
      - channel: '#alerts-critical'
        title: ':rotating_light: CRITICAL: {{ .GroupLabels.alertname }}'
        text: |
          *Severity:* {{ .CommonLabels.severity }}
          *Application:* {{ .CommonLabels.application }}
          {{ range .Alerts }}
          *Alert:* {{ .Annotations.summary }}
          *Description:* {{ .Annotations.description }}
          *Runbook:* {{ .Annotations.runbook }}
          {{ end }}
        send_resolved: true

  - name: 'pagerduty-critical'
    pagerduty_configs:
      - service_key: '<PAGERDUTY_SERVICE_KEY>'
        description: '{{ .CommonAnnotations.summary }}'
        details:
          description: '{{ .CommonAnnotations.description }}'
          runbook: '{{ .CommonAnnotations.runbook }}'

inhibit_rules:
  # ถ้า app down แล้ว ไม่ต้อง alert เรื่อง latency/error rate
  - source_match:
      alertname: ApplicationDown
    target_match_re:
      alertname: 'HighErrorRate|HighP95Latency|ThreadPoolSaturation'
    equal: ['application']
```

---

## ขั้นตอนที่ 2475: On-Call Runbooks {#runbooks}

### Application Down Runbook

```markdown
# Runbook: ApplicationDown

## อาการ (Symptoms)
Application instance ไม่ตอบสนองต่อ health check

## ผลกระทบ (Impact)
- Users ไม่สามารถ access application ได้
- Traffic ถูก route ไปยัง instance อื่น (ถ้ามี)
- SLO อาจถูก breach

## ขั้นตอนการ investigate

### Step 1: ตรวจสอบสถานะ Pod
```bash
kubectl get pods -n production -l app=spring-boot-app
kubectl describe pod <pod-name> -n production
```

### Step 2: ดู Recent Logs
```bash
kubectl logs <pod-name> -n production --previous --tail=100
# ดู error ล่าสุด
kubectl logs <pod-name> -n production | grep -i "error\|exception\|fatal"
```

### Step 3: ตรวจสอบ Resources
```bash
kubectl top pods -n production
kubectl top nodes
```

### Step 4: ตรวจสอบ Events
```bash
kubectl get events -n production --sort-by='.lastTimestamp'
```

## การแก้ไข

### ถ้า Pod ใน CrashLoopBackOff
1. ดู logs จาก previous container
2. ตรวจสอบ memory/CPU limits
3. ตรวจสอบ startup probes configuration

### ถ้า OOMKilled
1. เพิ่ม memory limit
2. ตรวจสอบ memory leak ใน Grafana

### ถ้า Pod ไม่ start
1. ตรวจสอบ image pull errors
2. ตรวจสอบ configmap/secret references

## Escalation
- L1 (5 นาที): ทดลอง restart pod
- L2 (15 นาที): ติดต่อ backend team lead
- L3 (30 นาที): ติดต่อ CTO
```

### High Error Rate Runbook

```java
package com.example.observability;

// Runbook ที่ embed ใน code สำหรับ developers
/**
 * HIGH ERROR RATE RUNBOOK
 *
 * Metric: http_server_requests_seconds_count{status=~"5.."}
 * Threshold: > 5% of total requests
 *
 * STEP 1: ดูว่า error เกิดจาก endpoint ไหน
 * PromQL: sum by(uri) (rate(http_server_requests_seconds_count{status=~"5.."}[5m]))
 *
 * STEP 2: ดู logs สำหรับ errors นั้น
 * LogQL: {app="spring-boot-app"} | json | level="ERROR"
 *
 * STEP 3: ตรวจสอบ downstream dependencies
 * - Database: hikaricp_connections_active / hikaricp_connections_max
 * - External APIs: check circuit breaker metrics
 *
 * STEP 4: ตรวจสอบ recent deployments
 * - kubectl rollout history deployment/spring-boot-app
 * - อาจต้อง rollback: kubectl rollout undo deployment/spring-boot-app
 */
@RestController
@RequestMapping("/api/orders")
@Slf4j
public class OrderController {

    // Error handling ที่ดี - บันทึก context สำหรับ debugging
    @PostMapping
    public ResponseEntity<OrderResponse> createOrder(
            @Valid @RequestBody CreateOrderRequest request) {
        
        MDC.put("customerId", request.getCustomerId());
        MDC.put("operation", "createOrder");
        
        try {
            Order order = orderService.createOrder(request);
            log.info("Order created successfully: orderId={}", order.getId());
            return ResponseEntity.ok(new OrderResponse(order));
        } catch (InsufficientStockException e) {
            log.warn("Insufficient stock for order: customerId={}, items={}",
                request.getCustomerId(), request.getItems(), e);
            return ResponseEntity.status(HttpStatus.CONFLICT)
                .body(OrderResponse.error("INSUFFICIENT_STOCK", e.getMessage()));
        } catch (PaymentException e) {
            log.error("Payment failed for order: customerId={}", 
                request.getCustomerId(), e);
            return ResponseEntity.status(HttpStatus.PAYMENT_REQUIRED)
                .body(OrderResponse.error("PAYMENT_FAILED", e.getMessage()));
        } catch (Exception e) {
            log.error("Unexpected error creating order: customerId={}", 
                request.getCustomerId(), e);
            return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
                .body(OrderResponse.error("INTERNAL_ERROR", "Please try again"));
        } finally {
            MDC.clear();
        }
    }
}
```

---

## ขั้นตอนที่ 2478: Docker Compose สำหรับ Full Observability Stack

```yaml
# docker-compose-observability.yml
version: '3.8'

services:
  app:
    build: .
    ports:
      - "8080:8080"
    environment:
      - ENVIRONMENT=local
      - OTLP_ENDPOINT=http://otel-collector:4318
    depends_on:
      - prometheus
      - loki
      - tempo

  # Metrics
  prometheus:
    image: prom/prometheus:v2.47.0
    ports:
      - "9090:9090"
    volumes:
      - ./observability/prometheus.yml:/etc/prometheus/prometheus.yml
      - ./observability/alert_rules:/etc/prometheus/alert_rules
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--web.enable-lifecycle'
      - '--web.enable-remote-write-receiver'

  # Tracing
  tempo:
    image: grafana/tempo:2.2.3
    ports:
      - "3200:3200"
      - "4317:4317"  # OTLP gRPC
      - "4318:4318"  # OTLP HTTP
    volumes:
      - ./observability/tempo.yml:/etc/tempo.yml
    command: ["-config.file=/etc/tempo.yml"]

  # Logging
  loki:
    image: grafana/loki:2.9.0
    ports:
      - "3100:3100"
    volumes:
      - ./observability/loki.yml:/etc/loki/local-config.yaml
    command: -config.file=/etc/loki/local-config.yaml

  # Visualization
  grafana:
    image: grafana/grafana:10.1.0
    ports:
      - "3000:3000"
    volumes:
      - ./observability/grafana/provisioning:/etc/grafana/provisioning
      - ./observability/grafana/dashboards:/var/lib/grafana/dashboards
    environment:
      - GF_AUTH_ANONYMOUS_ENABLED=true
      - GF_AUTH_ANONYMOUS_ORG_ROLE=Admin
    depends_on:
      - prometheus
      - loki
      - tempo

  # Alerting
  alertmanager:
    image: prom/alertmanager:v0.26.0
    ports:
      - "9093:9093"
    volumes:
      - ./observability/alertmanager.yml:/etc/alertmanager/alertmanager.yml

  # OTel Collector
  otel-collector:
    image: otel/opentelemetry-collector-contrib:0.88.0
    ports:
      - "4317:4317"
      - "4318:4318"
    volumes:
      - ./observability/otel-collector.yml:/etc/otelcol-contrib/config.yaml
```

---

## ขั้นตอนที่ 2480: Integration Test สำหรับ Observability

```java
package com.example.observability;

import io.micrometer.core.instrument.MeterRegistry;
import io.micrometer.core.instrument.search.Search;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.web.servlet.MockMvc;

import static org.assertj.core.api.Assertions.assertThat;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@SpringBootTest
@AutoConfigureMockMvc
class ObservabilityIntegrationTest {

    @Autowired
    private MockMvc mockMvc;

    @Autowired
    private MeterRegistry meterRegistry;

    @Test
    void shouldRecordMetricsForHttpRequests() throws Exception {
        // Act
        mockMvc.perform(get("/api/orders/1"))
            .andExpect(status().isOk());

        // Assert - ตรวจสอบว่า metric ถูกบันทึก
        Search search = meterRegistry.find("http.server.requests");
        assertThat(search.timer()).isNotNull();
        assertThat(search.timer().count()).isGreaterThan(0);
    }

    @Test
    void shouldExposePrometheusEndpoint() throws Exception {
        mockMvc.perform(get("/actuator/prometheus"))
            .andExpect(status().isOk())
            .andExpect(content().string(org.hamcrest.Matchers.containsString("jvm_memory")))
            .andExpect(content().string(org.hamcrest.Matchers.containsString("http_server_requests")));
    }

    @Test
    void shouldExposeHealthEndpoint() throws Exception {
        mockMvc.perform(get("/actuator/health"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.status").value("UP"));
    }

    @Test
    void shouldIncrementCustomCounter() {
        // Arrange
        BusinessMetrics metrics = new BusinessMetrics(meterRegistry);

        // Act
        metrics.recordOrderCreated(1500.0);
        metrics.recordOrderCreated(2500.0);

        // Assert
        assertThat(
            meterRegistry.find("orders.created.total").counter().count()
        ).isEqualTo(2.0);
    }
}
```

---

## สรุปสิ่งที่เรียนรู้

ใน Part นี้เราได้เรียนรู้:

1. **Three Pillars** - Metrics, Tracing, Logging ทำงานร่วมกันอย่างไร
2. **Micrometer Observation API** - API ใหม่ที่รวมทั้ง metrics และ tracing
3. **@Observed Annotation** - วิธีง่ายในการ observe method calls
4. **Prometheus** - การ scrape metrics และ define alert rules
5. **Grafana** - การสร้าง dashboard และ visualization
6. **Alert Rules** - การตั้ง threshold และ routing alerts
7. **Runbooks** - การเตรียม documentation สำหรับ on-call

### Key Takeaways
- ใช้ low cardinality labels สำหรับ metric labels (หลีกเลี่ยง userId, orderId)
- ใช้ high cardinality values สำหรับ trace attributes เท่านั้น
- กำหนด SLO ก่อน แล้วค่อย set alert threshold
- Runbook ควรมี step-by-step และ escalation path ที่ชัดเจน

---

*[← Part 70: Advanced Patterns](./part-70-advanced-patterns.md) | [Part 72: Contract Testing →](./part-72-contract-testing.md)*

# Part 85: Health Monitoring
## ขั้นตอนที่ 3001-3040

**ระดับ:** ระดับโลก (World-Class)
**เวลาเรียน:** 6-8 ชั่วโมง
**เป้าหมาย:** เรียนรู้การสร้างระบบ Health Monitoring ที่ครบวงจร ตั้งแต่ Custom HealthIndicator, Health Groups สำหรับ Kubernetes Probes, การตรวจสอบ dependencies ภายนอก ไปจนถึงการ alert เมื่อระบบมีปัญหา

---

## 3001-3006: Spring Boot Actuator Health Endpoint

Spring Boot Actuator มี `/actuator/health` endpoint ที่รายงานสถานะของแอปพลิเคชัน ซึ่งใช้ใน production เพื่อ monitoring และ orchestration (Kubernetes)

### Dependencies

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

### Actuator Configuration

```yaml
# application.yml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus,env,beans,loggers
      base-path: /actuator
  endpoint:
    health:
      show-details: always           # แสดง details เสมอ
      show-components: always        # แสดง components
      probes:
        enabled: true                # เปิด liveness/readiness probes
  health:
    livenessstate:
      enabled: true                  # เปิด liveness state
    readinessstate:
      enabled: true                  # เปิด readiness state
    defaults:
      enabled: true
    diskspace:
      enabled: true
      threshold: 10MB                # เตือนเมื่อ disk เหลือน้อยกว่า 10MB
  info:
    env:
      enabled: true
    git:
      enabled: true
      mode: full
    build:
      enabled: true
```

### ตัวอย่าง Health Response

```json
{
  "status": "UP",
  "components": {
    "db": {
      "status": "UP",
      "details": {
        "database": "PostgreSQL",
        "validationQuery": "isValid()"
      }
    },
    "redis": {
      "status": "UP",
      "details": {
        "version": "7.0.8"
      }
    },
    "diskSpace": {
      "status": "UP",
      "details": {
        "total": 500107862016,
        "free": 250053931008,
        "threshold": 10485760,
        "exists": true
      }
    },
    "livenessState": {
      "status": "UP"
    },
    "readinessState": {
      "status": "UP"
    }
  }
}
```

---

## 3007-3012: Custom HealthIndicator

การสร้าง HealthIndicator เองเพื่อตรวจสอบ components ที่ Spring ยังไม่รองรับ

### Database Custom HealthIndicator

```java
// DatabaseHealthIndicator.java
package com.example.health.indicator;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.boot.actuate.health.Health;
import org.springframework.boot.actuate.health.HealthIndicator;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.stereotype.Component;

import java.util.HashMap;
import java.util.Map;

@Slf4j
@Component("database")
@RequiredArgsConstructor
public class DatabaseHealthIndicator implements HealthIndicator {

    private final JdbcTemplate jdbcTemplate;

    @Override
    public Health health() {
        Map<String, Object> details = new HashMap<>();
        
        try {
            long start = System.currentTimeMillis();
            
            // ทดสอบ query
            Integer result = jdbcTemplate.queryForObject(
                "SELECT 1", Integer.class
            );
            
            long responseTime = System.currentTimeMillis() - start;
            
            // ดึงข้อมูลเพิ่มเติม
            String version = jdbcTemplate.queryForObject(
                "SELECT version()", String.class
            );
            
            details.put("responseTime", responseTime + "ms");
            details.put("version", version);
            details.put("status", "Connected");
            
            // เตือนถ้า query ช้าเกินไป
            if (responseTime > 1000) {
                return Health.up()
                    .withDetails(details)
                    .withDetail("warning", "Database response time is slow: " + responseTime + "ms")
                    .build();
            }
            
            return Health.up().withDetails(details).build();
            
        } catch (Exception e) {
            log.error("Database health check failed", e);
            details.put("error", e.getMessage());
            return Health.down()
                .withDetails(details)
                .withException(e)
                .build();
        }
    }
}
```

### Redis HealthIndicator

```java
// RedisHealthIndicator.java
package com.example.health.indicator;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.boot.actuate.health.Health;
import org.springframework.boot.actuate.health.HealthIndicator;
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.data.redis.connection.RedisServerCommands;
import org.springframework.stereotype.Component;

import java.util.Properties;

@Slf4j
@Component("redis")
@RequiredArgsConstructor
public class RedisHealthIndicator implements HealthIndicator {

    private final RedisConnectionFactory redisConnectionFactory;

    @Override
    public Health health() {
        try {
            var connection = redisConnectionFactory.getConnection();
            RedisServerCommands serverCommands = connection.serverCommands();
            
            // PING test
            String pong = connection.ping();
            
            // ดึง server info
            Properties info = serverCommands.info("server");
            String version = info.getProperty("redis_version");
            String mode = info.getProperty("redis_mode");
            String connectedClients = info.getProperty("connected_clients");
            String usedMemory = info.getProperty("used_memory_human");
            
            Properties memoryInfo = serverCommands.info("memory");
            String maxMemory = memoryInfo.getProperty("maxmemory_human");
            
            connection.close();
            
            return Health.up()
                .withDetail("version", version)
                .withDetail("mode", mode)
                .withDetail("connectedClients", connectedClients)
                .withDetail("usedMemory", usedMemory)
                .withDetail("maxMemory", maxMemory)
                .withDetail("ping", pong)
                .build();
                
        } catch (Exception e) {
            log.error("Redis health check failed", e);
            return Health.down()
                .withDetail("error", e.getMessage())
                .build();
        }
    }
}
```

### Kafka HealthIndicator

```java
// KafkaHealthIndicator.java
package com.example.health.indicator;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.apache.kafka.clients.admin.AdminClient;
import org.apache.kafka.clients.admin.DescribeClusterResult;
import org.springframework.boot.actuate.health.Health;
import org.springframework.boot.actuate.health.HealthIndicator;
import org.springframework.kafka.core.KafkaAdmin;
import org.springframework.stereotype.Component;

import java.util.concurrent.TimeUnit;

@Slf4j
@Component("kafka")
@RequiredArgsConstructor
public class KafkaHealthIndicator implements HealthIndicator {

    private final KafkaAdmin kafkaAdmin;

    @Override
    public Health health() {
        try (AdminClient adminClient = AdminClient.create(kafkaAdmin.getConfigurationProperties())) {
            
            DescribeClusterResult clusterResult = adminClient.describeCluster();
            
            String clusterId = clusterResult.clusterId()
                .get(5, TimeUnit.SECONDS);
            int nodeCount = clusterResult.nodes()
                .get(5, TimeUnit.SECONDS).size();
            String controllerId = clusterResult.controller()
                .get(5, TimeUnit.SECONDS).idString();
            
            return Health.up()
                .withDetail("clusterId", clusterId)
                .withDetail("nodeCount", nodeCount)
                .withDetail("controllerId", controllerId)
                .build();
                
        } catch (Exception e) {
            log.error("Kafka health check failed", e);
            return Health.down()
                .withDetail("error", e.getMessage())
                .build();
        }
    }
}
```

### External API HealthIndicator

```java
// PaymentGatewayHealthIndicator.java
package com.example.health.indicator;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.boot.actuate.health.Health;
import org.springframework.boot.actuate.health.HealthIndicator;
import org.springframework.stereotype.Component;
import org.springframework.web.client.RestTemplate;

import java.time.Duration;
import java.time.Instant;

@Slf4j
@Component("paymentGateway")
@RequiredArgsConstructor
public class PaymentGatewayHealthIndicator implements HealthIndicator {

    private final RestTemplate restTemplate;
    private final String paymentGatewayUrl = "https://api.payment-gateway.com/health";

    @Override
    public Health health() {
        Instant start = Instant.now();
        
        try {
            // เรียก health endpoint ของ payment gateway
            var response = restTemplate.getForEntity(
                paymentGatewayUrl,
                String.class
            );
            
            long responseTime = Duration.between(start, Instant.now()).toMillis();
            
            if (response.getStatusCode().is2xxSuccessful()) {
                Health.Builder builder = Health.up()
                    .withDetail("url", paymentGatewayUrl)
                    .withDetail("responseTime", responseTime + "ms")
                    .withDetail("statusCode", response.getStatusCode().value());
                
                // เตือนถ้าช้า
                if (responseTime > 2000) {
                    builder.withDetail("warning", "Slow response: " + responseTime + "ms");
                }
                
                return builder.build();
            } else {
                return Health.down()
                    .withDetail("url", paymentGatewayUrl)
                    .withDetail("statusCode", response.getStatusCode().value())
                    .withDetail("responseTime", responseTime + "ms")
                    .build();
            }
            
        } catch (Exception e) {
            log.error("Payment gateway health check failed", e);
            return Health.down()
                .withDetail("url", paymentGatewayUrl)
                .withDetail("error", e.getMessage())
                .withDetail("responseTime", Duration.between(start, Instant.now()).toMillis() + "ms")
                .build();
        }
    }
}
```

---

## 3013-3018: Composite Health Checks

การรวม health indicators หลายตัวเข้าด้วยกัน

### Composite Health Indicator

```java
// StorageHealthIndicator.java
package com.example.health.indicator;

import lombok.RequiredArgsConstructor;
import org.springframework.boot.actuate.health.*;
import org.springframework.stereotype.Component;

@Component("storage")
@RequiredArgsConstructor
public class StorageHealthIndicator implements CompositeHealthContributor {

    private final DatabaseHealthIndicator databaseHealth;
    private final RedisHealthIndicator redisHealth;
    private final S3HealthIndicator s3Health;

    @Override
    public HealthContributor getContributor(String name) {
        return switch (name) {
            case "database" -> databaseHealth;
            case "redis" -> redisHealth;
            case "s3" -> s3Health;
            default -> null;
        };
    }

    @Override
    public java.util.Iterator<NamedContributor<HealthContributor>> iterator() {
        return java.util.List.of(
            NamedContributor.of("database", databaseHealth),
            NamedContributor.of("redis", redisHealth),
            NamedContributor.of("s3", s3Health)
        ).iterator();
    }
}
```

### S3 HealthIndicator

```java
// S3HealthIndicator.java
package com.example.health.indicator;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.boot.actuate.health.Health;
import org.springframework.boot.actuate.health.HealthIndicator;
import org.springframework.stereotype.Component;
import software.amazon.awssdk.services.s3.S3Client;
import software.amazon.awssdk.services.s3.model.HeadBucketRequest;

@Slf4j
@Component("s3")
@RequiredArgsConstructor
public class S3HealthIndicator implements HealthIndicator {

    private final S3Client s3Client;
    private final String bucketName = "${app.s3.bucket-name}";

    @Override
    public Health health() {
        try {
            s3Client.headBucket(HeadBucketRequest.builder()
                .bucket(bucketName)
                .build());
            
            return Health.up()
                .withDetail("bucket", bucketName)
                .withDetail("accessible", true)
                .build();
                
        } catch (Exception e) {
            log.error("S3 health check failed for bucket: {}", bucketName, e);
            return Health.down()
                .withDetail("bucket", bucketName)
                .withDetail("error", e.getMessage())
                .build();
        }
    }
}
```

---

## 3019-3024: Health Groups สำหรับ Kubernetes Probes

Kubernetes ต้องการ probe ที่แยกกัน 3 ประเภท: Liveness, Readiness, Startup

```
Liveness Probe:  แอปยัง alive ไหม? ถ้า DOWN → restart container
Readiness Probe: แอปพร้อม serve traffic ไหม? ถ้า DOWN → remove from load balancer
Startup Probe:   แอป start เสร็จแล้วไหม? ใช้ระหว่าง startup เท่านั้น
```

### Health Groups Configuration

```yaml
# application.yml
management:
  endpoint:
    health:
      show-details: always
      group:
        # Liveness: ตรวจสอบว่าแอปทำงานได้ปกติ
        liveness:
          include: livenessState
          additional-path: server:/actuator/health/liveness  # แยก path
          
        # Readiness: ตรวจสอบว่าแอปพร้อม serve request
        readiness:
          include: readinessState,db,redis
          additional-path: server:/actuator/health/readiness
          
        # Startup: ตรวจสอบว่า startup สมบูรณ์
        startup:
          include: db,redis,kafka
          
        # External: สำหรับ external monitoring
        external:
          include: paymentGateway,emailService
          show-details: when-authorized
          roles: ADMIN
```

### Kubernetes Deployment Manifest

```yaml
# kubernetes/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spring-boot-app
spec:
  template:
    spec:
      containers:
        - name: app
          image: my-app:latest
          ports:
            - containerPort: 8080
          
          # Startup Probe: รอให้แอป start ก่อน
          startupProbe:
            httpGet:
              path: /actuator/health/startup
              port: 8080
            failureThreshold: 30     # รอสูงสุด 30*10=300 วินาที
            periodSeconds: 10
            timeoutSeconds: 5
          
          # Liveness Probe: ตรวจสอบว่าแอปยัง alive
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            initialDelaySeconds: 30   # รอหลัง start
            periodSeconds: 20
            timeoutSeconds: 5
            failureThreshold: 3      # restart หลัง fail 3 ครั้ง
          
          # Readiness Probe: ตรวจสอบว่าพร้อม serve traffic
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080
            initialDelaySeconds: 15
            periodSeconds: 10
            timeoutSeconds: 5
            failureThreshold: 3
          
          resources:
            requests:
              memory: "512Mi"
              cpu: "250m"
            limits:
              memory: "1Gi"
              cpu: "500m"
```

### Application Availability

```java
// ApplicationReadinessService.java
package com.example.health.service;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.boot.availability.ApplicationAvailability;
import org.springframework.boot.availability.AvailabilityChangeEvent;
import org.springframework.boot.availability.LivenessState;
import org.springframework.boot.availability.ReadinessState;
import org.springframework.context.ApplicationEventPublisher;
import org.springframework.stereotype.Service;

@Slf4j
@Service
@RequiredArgsConstructor
public class ApplicationReadinessService {

    private final ApplicationAvailability applicationAvailability;
    private final ApplicationEventPublisher eventPublisher;

    // ตรวจสอบสถานะ liveness
    public LivenessState getLivenessState() {
        return applicationAvailability.getLivenessState();
    }

    // ตรวจสอบสถานะ readiness
    public ReadinessState getReadinessState() {
        return applicationAvailability.getReadinessState();
    }

    // ทำให้แอปไม่ ready (เช่น เมื่อ maintenance)
    public void setNotReady(String reason) {
        log.warn("Setting application to NOT READY: {}", reason);
        AvailabilityChangeEvent.publish(
            eventPublisher,
            reason,
            ReadinessState.REFUSING_TRAFFIC
        );
    }

    // ทำให้แอป ready อีกครั้ง
    public void setReady() {
        log.info("Setting application to READY");
        AvailabilityChangeEvent.publish(
            eventPublisher,
            "Application is ready",
            ReadinessState.ACCEPTING_TRAFFIC
        );
    }

    // ทำให้แอป liveness เป็น broken (จะถูก restart โดย Kubernetes)
    public void setLivenessBroken(String reason) {
        log.error("Setting application liveness to BROKEN: {}", reason);
        AvailabilityChangeEvent.publish(
            eventPublisher,
            reason,
            LivenessState.BROKEN
        );
    }
}
```

---

## 3025-3030: Circuit Breaker State ใน Health Endpoint

การแสดงสถานะ Circuit Breaker ใน health endpoint

### Dependencies

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-circuitbreaker-resilience4j</artifactId>
</dependency>
<dependency>
    <groupId>io.github.resilience4j</groupId>
    <artifactId>resilience4j-spring-boot3</artifactId>
</dependency>
```

### Circuit Breaker Health Indicator

```java
// CircuitBreakerHealthIndicator.java
package com.example.health.indicator;

import io.github.resilience4j.circuitbreaker.CircuitBreaker;
import io.github.resilience4j.circuitbreaker.CircuitBreakerRegistry;
import lombok.RequiredArgsConstructor;
import org.springframework.boot.actuate.health.Health;
import org.springframework.boot.actuate.health.HealthIndicator;
import org.springframework.stereotype.Component;

import java.util.HashMap;
import java.util.Map;

@Component("circuitBreakers")
@RequiredArgsConstructor
public class CircuitBreakerHealthIndicator implements HealthIndicator {

    private final CircuitBreakerRegistry circuitBreakerRegistry;

    @Override
    public Health health() {
        Map<String, Object> details = new HashMap<>();
        boolean anyOpen = false;
        
        for (CircuitBreaker cb : circuitBreakerRegistry.getAllCircuitBreakers()) {
            CircuitBreaker.State state = cb.getState();
            CircuitBreaker.Metrics metrics = cb.getMetrics();
            
            Map<String, Object> cbDetails = Map.of(
                "state", state.name(),
                "failureRate", metrics.getFailureRate() + "%",
                "slowCallRate", metrics.getSlowCallRate() + "%",
                "numberOfSuccessfulCalls", metrics.getNumberOfSuccessfulCalls(),
                "numberOfFailedCalls", metrics.getNumberOfFailedCalls(),
                "numberOfSlowCalls", metrics.getNumberOfSlowCalls(),
                "bufferedCalls", metrics.getNumberOfBufferedCalls()
            );
            
            details.put(cb.getName(), cbDetails);
            
            if (state == CircuitBreaker.State.OPEN) {
                anyOpen = true;
            }
        }
        
        if (anyOpen) {
            return Health.down()
                .withDetails(details)
                .withDetail("message", "One or more circuit breakers are OPEN")
                .build();
        }
        
        return Health.up().withDetails(details).build();
    }
}
```

### Resilience4j Configuration

```yaml
# application.yml
resilience4j:
  circuitbreaker:
    instances:
      paymentService:
        slidingWindowSize: 10
        minimumNumberOfCalls: 5
        permittedNumberOfCallsInHalfOpenState: 3
        automaticTransitionFromOpenToHalfOpenEnabled: true
        waitDurationInOpenState: 10s
        failureRateThreshold: 50
        slowCallRateThreshold: 80
        slowCallDurationThreshold: 2s
        
      emailService:
        slidingWindowSize: 20
        failureRateThreshold: 60
        waitDurationInOpenState: 30s
  
  # เปิด health endpoint สำหรับ circuit breakers
  health-indicator:
    enabled: true
    instances:
      paymentService:
        register-health-indicator: true
      emailService:
        register-health-indicator: true
```

---

## 3031-3036: Health Check Aggregation Dashboard

### Health Aggregation Service

```java
// HealthAggregationService.java
package com.example.health.service;

import lombok.RequiredArgsConstructor;
import org.springframework.boot.actuate.health.*;
import org.springframework.stereotype.Service;

import java.time.LocalDateTime;
import java.util.*;

@Service
@RequiredArgsConstructor
public class HealthAggregationService {

    private final HealthEndpoint healthEndpoint;

    public HealthReport generateReport() {
        SystemHealth systemHealth = healthEndpoint.health();
        
        List<ComponentHealth> components = new ArrayList<>();
        
        if (systemHealth instanceof CompositeHealth compositeHealth) {
            compositeHealth.getComponents().forEach((name, health) -> {
                ComponentHealth componentHealth = ComponentHealth.builder()
                    .name(name)
                    .status(health.getStatus().getCode())
                    .details(health.getDetails())
                    .checkedAt(LocalDateTime.now())
                    .build();
                
                components.add(componentHealth);
            });
        }
        
        return HealthReport.builder()
            .overallStatus(systemHealth.getStatus().getCode())
            .components(components)
            .generatedAt(LocalDateTime.now())
            .build();
    }

    @lombok.Data
    @lombok.Builder
    public static class HealthReport {
        private String overallStatus;
        private List<ComponentHealth> components;
        private LocalDateTime generatedAt;
    }

    @lombok.Data
    @lombok.Builder
    public static class ComponentHealth {
        private String name;
        private String status;
        private Map<String, Object> details;
        private LocalDateTime checkedAt;
    }
}
```

### Health Dashboard Controller

```java
// HealthDashboardController.java
package com.example.health.controller;

import com.example.health.service.HealthAggregationService;
import lombok.RequiredArgsConstructor;
import org.springframework.http.ResponseEntity;
import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("/api/v1/health")
@RequiredArgsConstructor
public class HealthDashboardController {

    private final HealthAggregationService healthAggregationService;

    @GetMapping("/report")
    @PreAuthorize("hasRole('OPS')")
    public ResponseEntity<HealthAggregationService.HealthReport> getHealthReport() {
        return ResponseEntity.ok(healthAggregationService.generateReport());
    }

    @GetMapping("/summary")
    public ResponseEntity<Object> getHealthSummary() {
        var report = healthAggregationService.generateReport();
        
        long downCount = report.getComponents().stream()
            .filter(c -> "DOWN".equals(c.getStatus()))
            .count();
        long degradedCount = report.getComponents().stream()
            .filter(c -> !Set.of("UP", "DOWN").contains(c.getStatus()))
            .count();
        
        return ResponseEntity.ok(java.util.Map.of(
            "status", report.getOverallStatus(),
            "totalComponents", report.getComponents().size(),
            "downComponents", downCount,
            "degradedComponents", degradedCount,
            "generatedAt", report.getGeneratedAt()
        ));
    }
}
```

---

## 3037-3040: Alerting เมื่อ Health Degradation

### Health Change Event Listener

```java
// HealthChangeListener.java
package com.example.health.listener;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.boot.actuate.health.Status;
import org.springframework.context.event.EventListener;
import org.springframework.scheduling.annotation.Async;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Component;

import java.time.LocalDateTime;
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;

@Slf4j
@Component
@RequiredArgsConstructor
public class HealthChangeListener {

    private final AlertService alertService;
    private final HealthCheckService healthCheckService;
    
    // เก็บสถานะล่าสุดของแต่ละ component
    private final Map<String, String> lastKnownStatus = new ConcurrentHashMap<>();
    
    // ตรวจสอบ health ทุก 30 วินาที และ alert ถ้ามีการเปลี่ยนแปลง
    @Scheduled(fixedDelay = 30000)
    @Async
    public void checkHealthAndAlert() {
        Map<String, String> currentStatus = healthCheckService.getComponentStatuses();
        
        currentStatus.forEach((component, status) -> {
            String previousStatus = lastKnownStatus.get(component);
            
            if (previousStatus != null && !previousStatus.equals(status)) {
                // สถานะเปลี่ยนแปลง
                handleStatusChange(component, previousStatus, status);
            } else if (previousStatus == null && !"UP".equals(status)) {
                // component ใหม่ที่ไม่ healthy
                handleStatusChange(component, "UNKNOWN", status);
            }
            
            lastKnownStatus.put(component, status);
        });
    }

    private void handleStatusChange(String component, String from, String to) {
        log.warn("Health status changed for '{}': {} -> {}", component, from, to);
        
        if ("DOWN".equals(to)) {
            alertService.sendCriticalAlert(
                "Component DOWN: " + component,
                String.format("Component '%s' changed from %s to %s at %s",
                    component, from, to, LocalDateTime.now())
            );
        } else if ("UP".equals(to) && "DOWN".equals(from)) {
            alertService.sendRecoveryAlert(
                "Component Recovered: " + component,
                String.format("Component '%s' recovered at %s", component, LocalDateTime.now())
            );
        }
    }
}
```

### Alert Service

```java
// AlertService.java
package com.example.health.service;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.mail.SimpleMailMessage;
import org.springframework.mail.javamail.JavaMailSender;
import org.springframework.stereotype.Service;

@Slf4j
@Service
@RequiredArgsConstructor
public class AlertService {

    private final JavaMailSender mailSender;
    private final PagerDutyClient pagerDutyClient;
    private final SlackWebhookClient slackClient;

    public void sendCriticalAlert(String title, String message) {
        log.error("CRITICAL ALERT: {} - {}", title, message);
        
        // ส่ง PagerDuty incident
        pagerDutyClient.createIncident(
            PagerDutyClient.Severity.CRITICAL,
            title,
            message
        );
        
        // ส่ง Slack notification
        slackClient.sendToChannel(
            "#critical-alerts",
            ":red_circle: *CRITICAL*: " + title + "\n" + message
        );
        
        // ส่งอีเมล
        sendEmail("ops-team@company.com", "[CRITICAL] " + title, message);
    }

    public void sendWarningAlert(String title, String message) {
        log.warn("WARNING ALERT: {} - {}", title, message);
        
        slackClient.sendToChannel(
            "#monitoring",
            ":warning: *WARNING*: " + title + "\n" + message
        );
    }

    public void sendRecoveryAlert(String title, String message) {
        log.info("RECOVERY ALERT: {} - {}", title, message);
        
        // Resolve PagerDuty incident
        pagerDutyClient.resolveIncident(title);
        
        slackClient.sendToChannel(
            "#monitoring",
            ":white_check_mark: *RECOVERED*: " + title + "\n" + message
        );
    }

    private void sendEmail(String to, String subject, String body) {
        try {
            SimpleMailMessage message = new SimpleMailMessage();
            message.setTo(to);
            message.setSubject(subject);
            message.setText(body);
            mailSender.send(message);
        } catch (Exception e) {
            log.error("Failed to send alert email", e);
        }
    }
}
```

### Health Check Scheduled Task

```java
// HealthCheckService.java
package com.example.health.service;

import lombok.RequiredArgsConstructor;
import org.springframework.boot.actuate.health.*;
import org.springframework.stereotype.Service;

import java.util.HashMap;
import java.util.Map;

@Service
@RequiredArgsConstructor
public class HealthCheckService {

    private final HealthEndpoint healthEndpoint;

    public Map<String, String> getComponentStatuses() {
        Map<String, String> statuses = new HashMap<>();
        
        SystemHealth health = healthEndpoint.health();
        statuses.put("overall", health.getStatus().getCode());
        
        if (health instanceof CompositeHealth compositeHealth) {
            compositeHealth.getComponents().forEach((name, componentHealth) -> {
                statuses.put(name, componentHealth.getStatus().getCode());
            });
        }
        
        return statuses;
    }

    public boolean isHealthy() {
        return Status.UP.equals(healthEndpoint.health().getStatus());
    }

    public boolean isComponentHealthy(String componentName) {
        SystemHealth health = healthEndpoint.health();
        
        if (health instanceof CompositeHealth compositeHealth) {
            Health component = compositeHealth.getComponents().get(componentName);
            if (component != null) {
                return Status.UP.equals(component.getStatus());
            }
        }
        
        return false;
    }
}
```

### Maintenance Mode Controller

```java
// MaintenanceModeController.java
package com.example.health.controller;

import com.example.health.service.ApplicationReadinessService;
import lombok.RequiredArgsConstructor;
import org.springframework.http.ResponseEntity;
import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.web.bind.annotation.*;

import java.util.Map;

@RestController
@RequestMapping("/api/v1/maintenance")
@RequiredArgsConstructor
public class MaintenanceModeController {

    private final ApplicationReadinessService readinessService;

    @PostMapping("/enable")
    @PreAuthorize("hasRole('OPS')")
    public ResponseEntity<Map<String, String>> enableMaintenance(
        @RequestParam(defaultValue = "Scheduled maintenance") String reason
    ) {
        readinessService.setNotReady(reason);
        return ResponseEntity.ok(Map.of(
            "status", "MAINTENANCE",
            "message", "Application set to maintenance mode: " + reason,
            "readinessState", readinessService.getReadinessState().name()
        ));
    }

    @PostMapping("/disable")
    @PreAuthorize("hasRole('OPS')")
    public ResponseEntity<Map<String, String>> disableMaintenance() {
        readinessService.setReady();
        return ResponseEntity.ok(Map.of(
            "status", "READY",
            "message", "Application returned to normal operation",
            "readinessState", readinessService.getReadinessState().name()
        ));
    }

    @GetMapping("/status")
    public ResponseEntity<Map<String, String>> getStatus() {
        return ResponseEntity.ok(Map.of(
            "readinessState", readinessService.getReadinessState().name(),
            "livenessState", readinessService.getLivenessState().name()
        ));
    }
}
```

---

## Health Monitoring Best Practices

### สรุปหลักการ Health Monitoring ที่ดี

```yaml
# Best practices สำหรับ Health Monitoring

# 1. แยก Liveness และ Readiness ออกจากกัน
#    - Liveness: แอปทำงานได้ (ไม่ deadlock, ไม่ OOM)
#    - Readiness: แอปพร้อม serve traffic (DB connected, cache ready)

# 2. Timeout ที่เหมาะสม
#    - Health check ควร timeout เร็ว (< 5 วินาที)
#    - ไม่ให้ health check ทำให้ app ช้าลง

# 3. Caching
#    - Cache health results สั้นๆ (เช่น 10 วินาที)
#    - ป้องกัน health check spam

# 4. Graceful Degradation
#    - ถ้า external service down แต่ core ยัง work → status DEGRADED ไม่ใช่ DOWN
#    - แยก critical กับ non-critical dependencies

# 5. Information Security
#    - ซ่อน details จาก public (show-details: when-authorized)
#    - เปิดเฉพาะ status สำหรับ load balancer
```

### Health Check Caching

```java
// CachingHealthCheckConfig.java
package com.example.health.config;

import org.springframework.boot.actuate.health.Health;
import org.springframework.boot.actuate.health.HealthIndicator;
import org.springframework.cache.annotation.Cacheable;
import org.springframework.cache.annotation.EnableCaching;
import org.springframework.context.annotation.Configuration;

@Configuration
@EnableCaching
public class CachingHealthCheckConfig {

    // Wrapper ที่ cache health results
    public static abstract class CachedHealthIndicator implements HealthIndicator {
        
        @Override
        @Cacheable(value = "health-cache", key = "#root.targetClass.simpleName")
        public Health health() {
            return doHealthCheck();
        }
        
        protected abstract Health doHealthCheck();
    }
}
```

---

## สรุป Part 85

ในบทนี้เราได้เรียนรู้:

1. **Custom HealthIndicator** - สร้าง indicators สำหรับ DB, Redis, Kafka, External APIs
2. **Composite Health** - รวม health indicators หลายตัว
3. **Health Groups** - Liveness, Readiness, Startup probes สำหรับ Kubernetes
4. **Circuit Breaker Health** - แสดงสถานะ circuit breakers ใน health endpoint
5. **Health Dashboard** - รวบรวม health data และแสดงผล
6. **Alerting** - แจ้งเตือนเมื่อสถานะเปลี่ยนแปลง
7. **Maintenance Mode** - การควบคุม readiness state

---

*[← Part 84: Internationalization](./part-84-internationalization.md) | [Part 86: Next Topic →](./part-86-next-topic.md)*

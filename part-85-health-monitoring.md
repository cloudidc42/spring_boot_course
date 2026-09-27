# Part 85: Health Monitoring & Custom Health Indicators
## ขั้นตอนที่ 3001-3040

> **ระดับ:** ระดับโลก (World-Class)  
> **เวลาเรียน:** 4-5 ชั่วโมง  
> **เป้าหมาย:** Implement comprehensive health monitoring for production applications

---

## ขั้นตอนที่ 3001: Health Check Overview

```
Spring Boot Actuator Health:

  /actuator/health          → aggregate status
  /actuator/health/liveness → is app alive? (kill if DOWN)
  /actuator/health/readiness → ready to serve traffic?

Status levels:
  UP       = healthy
  DOWN     = unhealthy (alert!)
  DEGRADED = partial failure (warn)
  OUT_OF_SERVICE = intentionally offline
  UNKNOWN  = status unknown
```

---

## ขั้นตอนที่ 3002: Custom HealthIndicator

```java
// ตรวจสอบ external payment gateway
@Component
@RequiredArgsConstructor
public class PaymentGatewayHealthIndicator implements HealthIndicator {

    private final PaymentGatewayClient client;

    @Override
    public Health health() {
        try {
            long start = System.currentTimeMillis();
            boolean available = client.ping();
            long latency = System.currentTimeMillis() - start;

            if (!available) {
                return Health.down()
                    .withDetail("reason", "Payment gateway not responding")
                    .build();
            }

            if (latency > 1000) {
                return Health.status("DEGRADED")
                    .withDetail("latency_ms", latency)
                    .withDetail("threshold_ms", 1000)
                    .build();
            }

            return Health.up()
                .withDetail("latency_ms", latency)
                .build();

        } catch (Exception e) {
            return Health.down(e)
                .withDetail("reason", e.getMessage())
                .build();
        }
    }
}

// ตรวจสอบ disk space
@Component
public class DiskSpaceHealthIndicator implements HealthIndicator {

    private static final long WARNING_THRESHOLD = 500 * 1024 * 1024; // 500MB

    @Override
    public Health health() {
        File root = new File("/");
        long free = root.getFreeSpace();
        long total = root.getTotalSpace();
        double usedPercent = ((double)(total - free) / total) * 100;

        if (free < WARNING_THRESHOLD) {
            return Health.down()
                .withDetail("free_bytes", free)
                .withDetail("total_bytes", total)
                .withDetail("used_percent", String.format("%.1f%%", usedPercent))
                .withDetail("threshold_bytes", WARNING_THRESHOLD)
                .build();
        }

        return Health.up()
            .withDetail("free_bytes", free)
            .withDetail("used_percent", String.format("%.1f%%", usedPercent))
            .build();
    }
}
```

---

## ขั้นตอนที่ 3003: Health Groups Configuration

```yaml
management:
  endpoint:
    health:
      probes:
        enabled: true
      show-details: when-authorized
      show-components: when-authorized
      group:
        liveness:
          include: livenessState, diskSpace
          show-details: always
        readiness:
          include: readinessState, db, redis, kafka
          show-details: always
        deep:
          include: "*"
          show-details: always
  health:
    redis:
      enabled: true
    kafka:
      enabled: true
    db:
      enabled: true
```

---

## ขั้นตอนที่ 3004: Composite Health Check

```java
@Component("externalServicesHealth")
public class ExternalServicesHealthIndicator extends AbstractHealthIndicator {

    private final List<HealthIndicator> indicators;

    @Override
    protected void doHealthCheck(Health.Builder builder) {
        Map<String, Health> healths = new LinkedHashMap<>();
        boolean anyDown = false;
        boolean anyDegraded = false;

        for (HealthIndicator indicator : indicators) {
            Health h = indicator.health();
            healths.put(indicator.getClass().getSimpleName(), h);
            if (h.getStatus() == Status.DOWN) anyDown = true;
            if ("DEGRADED".equals(h.getStatus().getCode())) anyDegraded = true;
        }

        builder.withDetails(healths);

        if (anyDown) builder.down();
        else if (anyDegraded) builder.status("DEGRADED");
        else builder.up();
    }
}
```

---

## ขั้นตอนที่ 3005: Circuit Breaker State in Health

```java
@Component
@RequiredArgsConstructor
public class CircuitBreakerHealthIndicator implements HealthIndicator {

    private final CircuitBreakerRegistry registry;

    @Override
    public Health health() {
        Map<String, Object> details = new HashMap<>();
        boolean anyOpen = false;

        for (CircuitBreaker cb : registry.getAllCircuitBreakers()) {
            CircuitBreaker.State state = cb.getState();
            details.put(cb.getName(), Map.of(
                "state", state.name(),
                "failureRate", cb.getMetrics().getFailureRate(),
                "slowCallRate", cb.getMetrics().getSlowCallRate(),
                "bufferedCalls", cb.getMetrics().getNumberOfBufferedCalls()
            ));
            if (state == CircuitBreaker.State.OPEN) anyOpen = true;
        }

        Health.Builder builder = anyOpen ? Health.status("DEGRADED") : Health.up();
        return builder.withDetails(details).build();
    }
}
```

---

## ขั้นตอนที่ 3006-3040: Health Dashboard Controller

```java
@RestController
@RequestMapping("/api/v1/admin/health")
@PreAuthorize("hasRole('ADMIN')")
@RequiredArgsConstructor
public class HealthDashboardController {

    private final HealthEndpoint healthEndpoint;
    private final MeterRegistry meterRegistry;

    @GetMapping("/summary")
    public ResponseEntity<HealthSummary> getHealthSummary() {
        HealthComponent health = healthEndpoint.health();

        return ResponseEntity.ok(new HealthSummary(
            health.getStatus().getCode(),
            getComponentStatuses(health),
            getKeyMetrics()
        ));
    }

    private Map<String, String> getComponentStatuses(HealthComponent health) {
        if (health instanceof SystemHealth systemHealth) {
            return systemHealth.getComponents().entrySet().stream()
                .collect(Collectors.toMap(
                    Map.Entry::getKey,
                    e -> e.getValue().getStatus().getCode()
                ));
        }
        return Map.of();
    }

    private Map<String, Object> getKeyMetrics() {
        Map<String, Object> metrics = new HashMap<>();

        // JVM heap usage
        Gauge heap = meterRegistry.find("jvm.memory.used")
            .tag("area", "heap").gauge();
        if (heap != null) {
            metrics.put("heapUsedMb", heap.value() / (1024 * 1024));
        }

        // Active DB connections
        Gauge dbConns = meterRegistry.find("hikaricp.connections.active").gauge();
        if (dbConns != null) {
            metrics.put("dbActiveConnections", dbConns.value());
        }

        // HTTP error rate
        Counter errors = meterRegistry.find("http.server.requests")
            .tag("status", "5xx").counter();
        if (errors != null) {
            metrics.put("httpErrors5xx", errors.count());
        }

        return metrics;
    }
}

record HealthSummary(String status, Map<String, String> components, Map<String, Object> metrics) {}
```

---

*[← Part 84: Internationalization](./part-84-internationalization.md) | [Part 86: Integration Patterns →](./part-86-integration-patterns.md)*

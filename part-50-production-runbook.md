# Part 50: Production Runbook & Operations
## ขั้นตอนที่ 1601-1640

> **ระดับ:** ระดับโลก (World-Class)  
> **เวลาเรียน:** 4-5 ชั่วโมง  
> **เป้าหมาย:** Operate Spring Boot applications in production

---

## ขั้นตอนที่ 1601: Production Readiness Checklist

```
Before Going to Production:

Application:
  ✅ Health checks configured (liveness + readiness)
  ✅ Graceful shutdown configured
  ✅ Connection pool properly sized
  ✅ Timeouts configured (DB, HTTP clients, external services)
  ✅ Circuit breakers in place
  ✅ Retry logic with backoff
  ✅ Error handling comprehensive
  ✅ Secrets externalized (not in code/config files)
  ✅ Logging structured (JSON)
  ✅ Correlation IDs in logs

Database:
  ✅ Database migrations (Flyway)
  ✅ Connection pool tuned
  ✅ Indexes on frequently queried columns
  ✅ Slow query logging enabled
  ✅ Backup configured and tested
  ✅ N+1 queries eliminated

Infrastructure:
  ✅ Auto-scaling configured
  ✅ Load balancer health checks
  ✅ Multi-AZ deployment
  ✅ Disaster recovery plan
  ✅ Monitoring and alerting

Security:
  ✅ HTTPS everywhere
  ✅ Security headers
  ✅ Secrets rotation policy
  ✅ Penetration tested
  ✅ OWASP compliance checked
```

---

## ขั้นตอนที่ 1602: application-prod.yml

```yaml
spring:
  # Database
  datasource:
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
      connection-timeout: 30000
      idle-timeout: 600000
      max-lifetime: 1800000
  
  # JPA
  jpa:
    open-in-view: false
    properties:
      hibernate:
        jdbc:
          batch_size: 100
        order_inserts: true
        order_updates: true
  
  # Cache
  cache:
    type: redis
    redis:
      time-to-live: 3600000  # 1 hour
  
  # Graceful shutdown
  lifecycle:
    timeout-per-shutdown-phase: 30s

# Server
server:
  port: 8080
  shutdown: graceful
  tomcat:
    max-threads: 200
    min-spare-threads: 10
    accept-count: 100
  compression:
    enabled: true
    mime-types: application/json

# Actuator
management:
  server:
    port: 8090
  endpoints:
    web:
      base-path: /actuator
      exposure:
        include: health, info, prometheus, metrics, loggers
  endpoint:
    health:
      probes:
        enabled: true
      show-details: never

# Logging
logging:
  level:
    root: WARN
    com.myapp: INFO
    org.springframework.web: WARN
    org.hibernate.SQL: WARN
  pattern:
    console: '{"timestamp":"%d{ISO8601}","level":"%level","logger":"%logger","message":"%message","traceId":"%X{traceId}","spanId":"%X{spanId}"}%n'

# Virtual Threads (Java 21+)
spring:
  threads:
    virtual:
      enabled: true
```

---

## ขั้นตอนที่ 1603: JVM Startup Script

```bash
#!/bin/bash
# start.sh

APP_NAME="myapp"
JAR_FILE="app.jar"
LOG_DIR="/var/log/${APP_NAME}"
HEAP_DUMP_DIR="/var/dumps/${APP_NAME}"

mkdir -p "${LOG_DIR}" "${HEAP_DUMP_DIR}"

JVM_OPTS=(
    # Memory
    -Xms512m
    -Xmx2g
    -XX:+UseG1GC
    -XX:MaxGCPauseMillis=200
    -XX:+UseStringDeduplication
    
    # Container support
    -XX:+UseContainerSupport
    -XX:MaxRAMPercentage=75.0
    
    # Heap dumps on OOM
    -XX:+HeapDumpOnOutOfMemoryError
    -XX:HeapDumpPath="${HEAP_DUMP_DIR}/${APP_NAME}-$(date +%Y%m%d-%H%M%S).hprof"
    
    # GC logging
    -Xlog:gc*:file="${LOG_DIR}/gc.log":time,uptime:filecount=5,filesize=20m
    
    # Faster startup
    -Djava.security.egd=file:/dev/./urandom
    
    # Timezone
    -Duser.timezone=Asia/Bangkok
    
    # Spring profile
    -Dspring.profiles.active=prod
)

exec java "${JVM_OPTS[@]}" -jar "${JAR_FILE}"
```

---

## ขั้นตอนที่ 1604: Common Issues & Fixes

```bash
# ===== Memory Issues =====

# Check JVM heap usage
curl http://localhost:8090/actuator/metrics/jvm.memory.used

# Take heap dump
jcmd $(pgrep java) GC.heap_dump /tmp/heap.hprof
# Or via Actuator (if enabled)
curl -X POST http://localhost:8090/actuator/heapdump > heap.hprof

# Analyze with Eclipse MAT or jhat
jhat -port 7000 heap.hprof

# ===== Thread Issues =====

# Thread dump
jstack $(pgrep java) > /tmp/thread-dump.txt
# Or via Actuator
curl http://localhost:8090/actuator/threaddump > thread-dump.json

# Check for deadlocks in thread dump
grep -A 10 "deadlock" /tmp/thread-dump.txt

# ===== Database Issues =====

# Check active DB connections
SELECT count(*) FROM pg_stat_activity;

# Check slow queries
SELECT query, calls, total_exec_time, mean_exec_time
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 20;

# Kill long-running queries
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE state = 'active'
AND query_start < NOW() - INTERVAL '5 minutes';

# Check HikariCP pool status
curl http://localhost:8090/actuator/metrics/hikaricp.connections.active

# ===== CPU Issues =====

# Find what's using CPU
jstack $(pgrep java) | grep "nid=0x$(printf '%x' $TOP_THREAD)"

# Flame graph with async-profiler
./profiler.sh -d 60 -f /tmp/flamegraph.html $(pgrep java)
```

---

## ขั้นตอนที่ 1605: Graceful Shutdown

```java
// Ensure in-flight requests complete before shutdown
@Configuration
public class GracefulShutdownConfig {
    
    @Bean
    public GracefulShutdown gracefulShutdown() {
        return new GracefulShutdown();
    }
    
    @Bean
    public ConfigurableServletWebServerFactory webServerFactory(GracefulShutdown gracefulShutdown) {
        TomcatServletWebServerFactory factory = new TomcatServletWebServerFactory();
        factory.addConnectorCustomizers(gracefulShutdown);
        return factory;
    }
}

class GracefulShutdown implements TomcatConnectorCustomizer, ApplicationListener<ContextClosedEvent> {
    
    private volatile Connector connector;
    private final int waitTime = 30;
    
    @Override
    public void customize(Connector connector) {
        this.connector = connector;
    }
    
    @Override
    public void onApplicationEvent(ContextClosedEvent event) {
        log.info("Initiating graceful shutdown...");
        connector.pause();
        
        Executor executor = connector.getProtocolHandler().getExecutor();
        if (executor instanceof ThreadPoolExecutor threadPoolExecutor) {
            try {
                threadPoolExecutor.shutdown();
                if (!threadPoolExecutor.awaitTermination(waitTime, TimeUnit.SECONDS)) {
                    log.warn("Tomcat thread pool did not shut down gracefully within {} seconds", waitTime);
                }
            } catch (InterruptedException ex) {
                Thread.currentThread().interrupt();
            }
        }
    }
}
```

---

## ขั้นตอนที่ 1606: Feature Flags

```java
// Simple feature flag with Spring profiles
@Component
@ConditionalOnProperty(name = "feature.new-checkout", havingValue = "true")
public class NewCheckoutService implements CheckoutService { ... }

@Component
@ConditionalOnProperty(name = "feature.new-checkout", matchIfMissing = true, havingValue = "false")
public class OldCheckoutService implements CheckoutService { ... }

// Dynamic feature flags with Redis
@Service
@RequiredArgsConstructor
public class FeatureFlagService {
    
    private final RedisTemplate<String, String> redisTemplate;
    
    public boolean isEnabled(String feature) {
        String value = redisTemplate.opsForValue().get("feature:" + feature);
        return "true".equalsIgnoreCase(value);
    }
    
    public boolean isEnabledForUser(String feature, Long userId) {
        // Gradual rollout: enabled for users where userId % 100 < rolloutPercent
        String rolloutKey = "feature:" + feature + ":rollout";
        String rolloutStr = redisTemplate.opsForValue().get(rolloutKey);
        
        if (rolloutStr == null) return false;
        
        int rolloutPercent = Integer.parseInt(rolloutStr);
        return (userId % 100) < rolloutPercent;
    }
    
    public void enable(String feature) {
        redisTemplate.opsForValue().set("feature:" + feature, "true");
    }
    
    public void setRollout(String feature, int percent) {
        redisTemplate.opsForValue().set("feature:" + feature + ":rollout", String.valueOf(percent));
    }
}
```

---

## ขั้นตอนที่ 1607-1640: Rolling Deployment Strategy

```yaml
# Zero-downtime deployment with Kubernetes
apiVersion: apps/v1
kind: Deployment
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 25%        # Allow 25% extra pods during update
      maxUnavailable: 0%   # Never take pods down before new ones ready
  
  template:
    spec:
      containers:
        - name: myapp
          # Graceful shutdown
          lifecycle:
            preStop:
              exec:
                command: ["/bin/sh", "-c", "sleep 10"]
          
          # Must be healthy before receiving traffic
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
            failureThreshold: 3
          
          # Kill pod only when really dead
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            initialDelaySeconds: 60
            periodSeconds: 30
            failureThreshold: 6
      
      # Wait for all terminations
      terminationGracePeriodSeconds: 60

# Deployment Process:
# 1. Build new image with tag sha-abc123
# 2. kubectl set image deployment/myapp myapp=myregistry/myapp:sha-abc123
# 3. Kubernetes creates new pod
# 4. New pod passes readiness check
# 5. Kubernetes routes traffic to new pod
# 6. Old pod receives SIGTERM
# 7. Old pod finishes in-flight requests (preStop sleep)
# 8. Old pod terminates
# 9. Repeat for each pod
```

---

*[← Part 49: Clean Architecture](./part-49-clean-architecture.md) | [Part 51: Real-World Project →](./part-51-realworld-project.md)*

# Part 09: Logging ใน Spring Boot
## ขั้นตอนที่ 181-200

> **ระดับ:** พื้นฐาน (Beginner)  
> **เวลาเรียน:** 2-3 ชั่วโมง  
> **เป้าหมาย:** ใช้งาน Logging อย่างมืออาชีพ เพื่อ Debug และ Monitor Application

---

## ขั้นตอนที่ 181: Logging Framework ใน Spring Boot

```
Spring Boot ใช้ SLF4J (Simple Logging Facade for Java)
เป็น facade (interface) สำหรับ logging implementations

SLF4J API (Interface)
    ├── Logback (default ใน Spring Boot) ← ใช้นี้!
    ├── Log4j2
    └── JUL (Java Util Logging)

ไม่ต้องเพิ่ม dependency - มาพร้อม spring-boot-starter
spring-boot-starter-logging รวมอยู่ใน spring-boot-starter-web
```

---

## ขั้นตอนที่ 182: Log Levels

```
TRACE → DEBUG → INFO → WARN → ERROR → OFF

TRACE  - ละเอียดที่สุด, ใช้ debug อย่างลึกมาก
DEBUG  - ข้อมูลสำหรับ developer, ใช้ระหว่าง development
INFO   - ข้อมูลทั่วไป, ใน production ดูได้
WARN   - คำเตือน แต่ยังทำงานได้
ERROR  - ข้อผิดพลาด ต้องแก้ไข
OFF    - ปิด logging ทั้งหมด

ถ้าตั้งค่า level = INFO:
  ✅ INFO แสดง
  ✅ WARN แสดง
  ✅ ERROR แสดง
  ❌ DEBUG ไม่แสดง
  ❌ TRACE ไม่แสดง
```

---

## ขั้นตอนที่ 183: ใช้งาน Logger

```java
import lombok.extern.slf4j.Slf4j;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

// วิธีที่ 1: Manual (verbose)
public class ProductService {
    
    private static final Logger log = 
        LoggerFactory.getLogger(ProductService.class);
    
    public void processProduct(Long id) {
        log.trace("Trace: Processing product id: {}", id);
        log.debug("Debug: Product details: {}", id);
        log.info("Info: Product processed successfully: {}", id);
        log.warn("Warn: Product has low stock: {}", id);
        log.error("Error: Failed to process product: {}", id);
    }
}

// วิธีที่ 2: Lombok @Slf4j (แนะนำ!)
@Service
@Slf4j  // ← สร้าง private static final Logger log อัตโนมัติ
public class ProductService {
    
    public void processProduct(Long id) {
        log.info("Processing product: {}", id);
    }
}

// วิธีที่ 3: ใช้ Other Lombok annotations
@Slf4j              // SLF4J (แนะนำ)
@Log                // java.util.logging
@Log4j2             // Log4j2
@CommonsLog         // Apache Commons Logging
@JBossLog           // JBoss Logging
```

---

## ขั้นตอนที่ 184: Logging Best Practices

```java
@Service
@Slf4j
public class OrderService {
    
    public Order createOrder(CreateOrderRequest request) {
        
        // ✅ ใช้ placeholder {} แทน string concatenation
        log.info("Creating order for user: {}", request.getUserId());
        
        // ❌ อย่าใช้ string concatenation (performance ต่ำ)
        log.info("Creating order for user: " + request.getUserId());
        
        // ✅ ตรวจสอบ level ก่อน log complex objects
        if (log.isDebugEnabled()) {
            log.debug("Order details: {}", request.toJson());  // expensive operation
        }
        
        // ✅ Log exception พร้อม stack trace
        try {
            return processOrder(request);
        } catch (Exception e) {
            log.error("Failed to create order for user: {}", request.getUserId(), e);
            throw e;
        }
        
        // ❌ อย่า log แบบนี้ - เสีย stack trace!
        // log.error("Failed: " + e.getMessage());
    }
    
    // ✅ ไม่ log sensitive data!
    public void login(String username, String password) {
        log.info("Login attempt for user: {}", username);
        // ❌ log.info("Login: username={}, password={}", username, password);
        // ❌ password ห้าม log!
    }
    
    // ✅ ใช้ MDC สำหรับ contextual logging
    public Order processWithContext(Long orderId) {
        MDC.put("orderId", orderId.toString());
        MDC.put("userId", getCurrentUserId());
        
        try {
            log.info("Processing order");  // log จะมี orderId และ userId
            return doProcess(orderId);
        } finally {
            MDC.clear();  // ล้าง MDC หลังใช้
        }
    }
}
```

---

## ขั้นตอนที่ 185: Logging Configuration ใน application.yml

```yaml
logging:
  # Log levels
  level:
    root: INFO                           # default level
    com.example: DEBUG                   # application code
    com.example.controller: TRACE        # specific package
    org.springframework.web: DEBUG       # Spring Web
    org.springframework.security: DEBUG  # Spring Security
    org.hibernate.SQL: DEBUG             # SQL queries
    org.hibernate.type.descriptor.sql: TRACE  # SQL parameter values
    
  # Log file
  file:
    name: logs/application.log           # log file path
    max-size: 10MB                       # rotate เมื่อ > 10MB
    max-history: 30                      # เก็บ log 30 วัน
    total-size-cap: 1GB                  # สูงสุดรวม 1GB
  
  # Console pattern
  pattern:
    console: "%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n"
    file: "%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n"
  
  # Log groups (Spring Boot 2.1+)
  group:
    web: org.springframework.core.codec, org.springframework.http,
         org.springframework.web, org.springframework.boot.actuate.endpoint.web
    sql: org.hibernate.SQL, org.jooq.tools.LoggerListener
  
  level:
    web: DEBUG    # แทนที่จะระบุ packages ทีละตัว
    sql: DEBUG
```

---

## ขั้นตอนที่ 186: Logback Configuration (logback-spring.xml)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!-- src/main/resources/logback-spring.xml -->
<configuration>
    
    <!-- Import Spring Boot default configurations -->
    <include resource="org/springframework/boot/logging/logback/defaults.xml"/>
    
    <!-- Properties -->
    <springProperty name="appName" source="spring.application.name" defaultValue="app"/>
    
    <!-- Console Appender (for development) -->
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>
                %d{yyyy-MM-dd HH:mm:ss.SSS} 
                %highlight(%-5level) 
                [%blue(%thread)] 
                %yellow(%logger{40}) : 
                %msg%n
            </pattern>
        </encoder>
    </appender>
    
    <!-- File Appender with rolling -->
    <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>logs/${appName}.log</file>
        
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>logs/${appName}-%d{yyyy-MM-dd}.%i.log.gz</fileNamePattern>
            <timeBasedFileNamingAndTriggeringPolicy
                class="ch.qos.logback.core.rolling.SizeAndTimeBasedFNATP">
                <maxFileSize>100MB</maxFileSize>
            </timeBasedFileNamingAndTriggeringPolicy>
            <maxHistory>30</maxHistory>
            <totalSizeCap>3GB</totalSizeCap>
        </rollingPolicy>
        
        <encoder>
            <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} %-5level [%thread] %logger{40} : %msg%n</pattern>
        </encoder>
    </appender>
    
    <!-- JSON Appender (for production with ELK stack) -->
    <appender name="JSON_FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>logs/${appName}-json.log</file>
        
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>logs/${appName}-json-%d{yyyy-MM-dd}.log.gz</fileNamePattern>
            <maxHistory>7</maxHistory>
        </rollingPolicy>
        
        <encoder class="net.logstash.logback.encoder.LogstashEncoder">
            <includeMdc>true</includeMdc>
        </encoder>
    </appender>
    
    <!-- Async Appender (performance) -->
    <appender name="ASYNC_FILE" class="ch.qos.logback.classic.AsyncAppender">
        <queueSize>512</queueSize>
        <discardingThreshold>0</discardingThreshold>
        <neverBlock>false</neverBlock>
        <appender-ref ref="FILE"/>
    </appender>
    
    <!-- Profile-specific configuration -->
    <springProfile name="dev">
        <root level="INFO">
            <appender-ref ref="CONSOLE"/>
        </root>
        <logger name="com.example" level="DEBUG"/>
        <logger name="org.springframework.web" level="DEBUG"/>
    </springProfile>
    
    <springProfile name="prod">
        <root level="WARN">
            <appender-ref ref="ASYNC_FILE"/>
            <appender-ref ref="JSON_FILE"/>
        </root>
        <logger name="com.example" level="INFO"/>
    </springProfile>
    
    <!-- Specific loggers -->
    <logger name="org.hibernate.SQL" level="DEBUG" additivity="false">
        <appender-ref ref="CONSOLE"/>
    </logger>
    
</configuration>
```

---

## ขั้นตอนที่ 187: MDC (Mapped Diagnostic Context)

MDC ช่วยเพิ่ม contextual information ใน log ทุก line ในนั้น request

```java
// MDC Filter - ใส่ข้อมูลไว้ใน MDC สำหรับทุก request
@Component
@Slf4j
public class MdcFilter extends OncePerRequestFilter {
    
    @Override
    protected void doFilterInternal(
        HttpServletRequest request,
        HttpServletResponse response,
        FilterChain filterChain
    ) throws ServletException, IOException {
        
        try {
            // เพิ่มข้อมูลเข้า MDC
            String requestId = UUID.randomUUID().toString().substring(0, 8);
            MDC.put("requestId", requestId);
            MDC.put("method", request.getMethod());
            MDC.put("uri", request.getRequestURI());
            MDC.put("clientIp", getClientIp(request));
            
            // ดึง user จาก JWT (ถ้ามี)
            String userId = extractUserIdFromRequest(request);
            if (userId != null) {
                MDC.put("userId", userId);
            }
            
            // เพิ่ม requestId ใน response header
            response.addHeader("X-Request-ID", requestId);
            
            filterChain.doFilter(request, response);
            
        } finally {
            MDC.clear();  // ล้างเสมอ!
        }
    }
    
    private String getClientIp(HttpServletRequest request) {
        String xForwardedFor = request.getHeader("X-Forwarded-For");
        return xForwardedFor != null ? xForwardedFor.split(",")[0] : request.getRemoteAddr();
    }
    
    private String extractUserIdFromRequest(HttpServletRequest request) {
        // Extract from JWT
        return null;  // implement based on auth mechanism
    }
}
```

```xml
<!-- logback-spring.xml - ใช้ MDC values ใน pattern -->
<pattern>
    %d{yyyy-MM-dd HH:mm:ss} 
    [%X{requestId}]       <!-- MDC value -->
    [%X{userId}]          <!-- MDC value -->
    %-5level 
    %logger{36} 
    - %msg%n
</pattern>

<!-- Output example: -->
<!-- 2026-09-26 10:00:00 [abc123] [user-42] INFO  com.example.UserService - User created -->
```

---

## ขั้นตอนที่ 188: Structured Logging (JSON)

```xml
<!-- pom.xml - Logstash Logback Encoder -->
<dependency>
    <groupId>net.logstash.logback</groupId>
    <artifactId>logstash-logback-encoder</artifactId>
    <version>7.4</version>
</dependency>
```

```java
// Structured logging ด้วย StructuredArguments
import static net.logstash.logback.argument.StructuredArguments.*;

@Service
@Slf4j
public class OrderService {
    
    public Order createOrder(CreateOrderRequest request) {
        // Structured arguments
        log.info("Order created",
            keyValue("userId", request.getUserId()),
            keyValue("amount", request.getTotalAmount()),
            keyValue("currency", "THB"),
            keyValue("items", request.getItems().size())
        );
        
        // ใน JSON output จะเป็น:
        // {
        //   "message": "Order created",
        //   "userId": 42,
        //   "amount": 1500.00,
        //   "currency": "THB",
        //   "items": 3
        // }
    }
}
```

---

## ขั้นตอนที่ 189: Request/Response Logging

```java
// Log ทุก HTTP request/response
@Component
@Slf4j
public class HttpLoggingFilter extends OncePerRequestFilter {
    
    @Override
    protected void doFilterInternal(
        HttpServletRequest request,
        HttpServletResponse response,
        FilterChain filterChain
    ) throws ServletException, IOException {
        
        long startTime = System.currentTimeMillis();
        
        // Wrap request เพื่อ read body หลายครั้ง
        ContentCachingRequestWrapper wrappedRequest = 
            new ContentCachingRequestWrapper(request, 10000);
        ContentCachingResponseWrapper wrappedResponse = 
            new ContentCachingResponseWrapper(response);
        
        try {
            filterChain.doFilter(wrappedRequest, wrappedResponse);
        } finally {
            long duration = System.currentTimeMillis() - startTime;
            
            // Log request
            log.info(
                "HTTP {} {} - Status: {} - Duration: {}ms - Body: {}",
                request.getMethod(),
                request.getRequestURI(),
                response.getStatus(),
                duration,
                getRequestBody(wrappedRequest)
            );
            
            // Copy response body back
            wrappedResponse.copyBodyToResponse();
        }
    }
    
    private String getRequestBody(ContentCachingRequestWrapper request) {
        byte[] content = request.getContentAsByteArray();
        if (content.length == 0) return "";
        
        try {
            String body = new String(content, request.getCharacterEncoding());
            // Mask sensitive data
            return maskSensitiveData(body);
        } catch (Exception e) {
            return "[error reading body]";
        }
    }
    
    private String maskSensitiveData(String json) {
        // Hide passwords
        return json.replaceAll(
            "\"password\"\\s*:\\s*\"[^\"]+\"",
            "\"password\": \"***\""
        );
    }
    
    @Override
    protected boolean shouldNotFilter(HttpServletRequest request) {
        String path = request.getRequestURI();
        return path.startsWith("/actuator") || path.startsWith("/static");
    }
}
```

---

## ขั้นตอนที่ 190: Log4j2 Configuration (Alternative)

```xml
<!-- pom.xml - ใช้ Log4j2 แทน Logback -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter</artifactId>
    <exclusions>
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-logging</artifactId>
        </exclusion>
    </exclusions>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-log4j2</artifactId>
</dependency>
```

```xml
<!-- src/main/resources/log4j2-spring.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="WARN">
    
    <Properties>
        <Property name="LOG_PATTERN">
            %d{yyyy-MM-dd HH:mm:ss.SSS} %-5level [%t] %c{1} - %msg%n
        </Property>
    </Properties>
    
    <Appenders>
        <Console name="Console" target="SYSTEM_OUT">
            <PatternLayout pattern="${LOG_PATTERN}"/>
        </Console>
        
        <RollingFile name="RollingFile" fileName="logs/app.log"
            filePattern="logs/app-%d{yyyy-MM-dd}-%i.log.gz">
            <PatternLayout pattern="${LOG_PATTERN}"/>
            <Policies>
                <TimeBasedTriggeringPolicy/>
                <SizeBasedTriggeringPolicy size="10MB"/>
            </Policies>
            <DefaultRolloverStrategy max="30"/>
        </RollingFile>
        
        <!-- Async (Log4j2 มี async built-in) -->
        <Async name="Async" bufferSize="512">
            <AppenderRef ref="RollingFile"/>
        </Async>
    </Appenders>
    
    <Loggers>
        <Logger name="com.example" level="DEBUG" additivity="false">
            <AppenderRef ref="Console"/>
        </Logger>
        
        <Root level="INFO">
            <AppenderRef ref="Console"/>
            <AppenderRef ref="Async"/>
        </Root>
    </Loggers>
    
</Configuration>
```

---

## ขั้นตอนที่ 191: Audit Logging

```java
// สร้าง Audit Trail ที่ดี
@Component
@Slf4j
@RequiredArgsConstructor
public class AuditLogger {
    
    public void log(AuditEvent event) {
        log.info("AUDIT | action={} | resource={} | resourceId={} | userId={} | result={} | ip={}",
            event.getAction(),
            event.getResource(),
            event.getResourceId(),
            event.getUserId(),
            event.getResult(),
            event.getClientIp()
        );
    }
}

@Getter
@Builder
public class AuditEvent {
    private String action;       // CREATE, UPDATE, DELETE, LOGIN, LOGOUT
    private String resource;     // User, Product, Order
    private String resourceId;
    private String userId;
    private String result;       // SUCCESS, FAILURE
    private String clientIp;
    private LocalDateTime timestamp;
    private String details;
}

// ใช้ AOP สำหรับ automatic audit logging
@Aspect
@Component
@RequiredArgsConstructor
@Slf4j
public class AuditAspect {
    
    private final AuditLogger auditLogger;
    
    @Around("@annotation(Audited)")
    public Object audit(ProceedingJoinPoint joinPoint) throws Throwable {
        // Before
        String action = getActionName(joinPoint);
        String userId = getCurrentUserId();
        
        try {
            Object result = joinPoint.proceed();
            
            // Success
            auditLogger.log(AuditEvent.builder()
                .action(action)
                .userId(userId)
                .result("SUCCESS")
                .timestamp(LocalDateTime.now())
                .build());
            
            return result;
        } catch (Exception e) {
            // Failure
            auditLogger.log(AuditEvent.builder()
                .action(action)
                .userId(userId)
                .result("FAILURE")
                .details(e.getMessage())
                .build());
            throw e;
        }
    }
}

// Annotation สำหรับ marking methods ที่ต้อง audit
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Audited {
    String action() default "";
    String resource() default "";
}

// ใช้งาน
@Service
public class UserService {
    
    @Audited(action = "CREATE", resource = "User")
    public UserResponse create(CreateUserRequest request) {
        // จะถูก audit log อัตโนมัติ
    }
    
    @Audited(action = "DELETE", resource = "User")
    public void delete(Long id) {
        // ...
    }
}
```

---

## ขั้นตอนที่ 192: Performance Logging

```java
@Aspect
@Component
@Slf4j
public class PerformanceLoggingAspect {
    
    // Log execution time ของทุก public methods ใน @Service
    @Around("@within(org.springframework.stereotype.Service)")
    public Object logExecutionTime(ProceedingJoinPoint joinPoint) throws Throwable {
        
        String className = joinPoint.getTarget().getClass().getSimpleName();
        String methodName = joinPoint.getSignature().getName();
        
        long startTime = System.currentTimeMillis();
        
        try {
            Object result = joinPoint.proceed();
            
            long duration = System.currentTimeMillis() - startTime;
            
            if (duration > 1000) {
                log.warn("SLOW METHOD: {}.{} took {}ms", className, methodName, duration);
            } else if (log.isDebugEnabled()) {
                log.debug("Method {}.{} took {}ms", className, methodName, duration);
            }
            
            return result;
        } catch (Exception e) {
            long duration = System.currentTimeMillis() - startTime;
            log.error("Method {}.{} failed after {}ms: {}", 
                className, methodName, duration, e.getMessage());
            throw e;
        }
    }
}
```

---

## ขั้นตอนที่ 193: Centralized Log Management

```yaml
# docker-compose.yml - ELK Stack for local development
version: '3.8'

services:
  elasticsearch:
    image: elasticsearch:8.11.0
    environment:
      - discovery.type=single-node
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
      - xpack.security.enabled=false
    ports:
      - "9200:9200"
    volumes:
      - es-data:/usr/share/elasticsearch/data

  logstash:
    image: logstash:8.11.0
    volumes:
      - ./logstash/pipeline:/usr/share/logstash/pipeline
    ports:
      - "5000:5000/tcp"
      - "5000:5000/udp"
    depends_on:
      - elasticsearch

  kibana:
    image: kibana:8.11.0
    ports:
      - "5601:5601"
    environment:
      ELASTICSEARCH_URL: http://elasticsearch:9200
    depends_on:
      - elasticsearch

volumes:
  es-data:
```

```
# logstash/pipeline/logstash.conf
input {
  tcp {
    port => 5000
    codec => json_lines
  }
}

filter {
  date {
    match => ["@timestamp", "ISO8601"]
  }
}

output {
  elasticsearch {
    hosts => ["elasticsearch:9200"]
    index => "spring-boot-%{+YYYY.MM.dd}"
  }
  stdout {
    codec => rubydebug
  }
}
```

---

## ขั้นตอนที่ 194: Monitoring Logs กับ Actuator

```yaml
# application.yml
management:
  endpoints:
    web:
      exposure:
        include: loggers, logfile
  
  endpoint:
    loggers:
      enabled: true
    logfile:
      enabled: true
```

```bash
# GET /actuator/loggers - ดู logger ทั้งหมด
curl http://localhost:8080/actuator/loggers

# GET /actuator/loggers/{name} - ดู logger เฉพาะ
curl http://localhost:8080/actuator/loggers/com.example.service

# POST /actuator/loggers/{name} - เปลี่ยน log level แบบ runtime!
curl -X POST http://localhost:8080/actuator/loggers/com.example.service \
  -H "Content-Type: application/json" \
  -d '{"configuredLevel": "DEBUG"}'

# ✅ เปลี่ยน level ได้โดยไม่ต้อง restart!

# GET /actuator/logfile - ดู log file
curl http://localhost:8080/actuator/logfile
```

---

## ขั้นตอนที่ 195: Log Security Best Practices

```java
@Service
@Slf4j
public class SecurityService {
    
    // ✅ Log ข้อมูลที่ปลอดภัย
    public void login(String username, String password) {
        log.info("Login attempt: username={}", username);
        // ❌ NEVER log password!
    }
    
    // ✅ Mask sensitive data
    public void processCard(CreditCardRequest request) {
        String maskedCard = maskCard(request.getCardNumber());
        log.info("Processing card: {}", maskedCard);
        // Output: "Processing card: ****-****-****-1234"
    }
    
    private String maskCard(String cardNumber) {
        if (cardNumber == null || cardNumber.length() < 4) return "****";
        return "****-****-****-" + cardNumber.substring(cardNumber.length() - 4);
    }
    
    // ✅ ไม่ log PII (Personal Identifiable Information)
    public void updateProfile(Long userId, UserProfile profile) {
        log.info("Updating profile for userId: {}", userId);
        // ❌ NEVER log: fullName, birthDate, nationalId, etc.
    }
    
    // ✅ Log ข้อมูลที่เป็นประโยชน์สำหรับ audit
    public void deleteUser(Long userId) {
        log.warn("ADMIN ACTION: Deleting user id={}, triggeredBy={}", 
            userId, getCurrentAdminId());
    }
}
```

---

## ขั้นตอนที่ 196: Logging ใน Test

```java
@SpringBootTest
@Slf4j
class UserServiceTest {
    
    @Test
    void createUser_ShouldLogInfo() {
        // ใช้ Logback ListAppender สำหรับ test log output
        ListAppender<ILoggingEvent> listAppender = new ListAppender<>();
        Logger userServiceLogger = (Logger) LoggerFactory.getLogger(UserService.class);
        listAppender.start();
        userServiceLogger.addAppender(listAppender);
        
        // Act
        userService.create(request);
        
        // Assert log messages
        List<ILoggingEvent> logs = listAppender.list;
        assertThat(logs).anyMatch(log ->
            log.getLevel() == Level.INFO &&
            log.getMessage().contains("Creating user")
        );
    }
}
```

---

## ขั้นตอนที่ 197-200: สรุปและ Checklist

### สรุป Part 09

```
✅ SLF4J framework และ Logback
✅ Log levels (TRACE → ERROR)
✅ ใช้ @Slf4j annotation
✅ Best practices (placeholders, sensitive data)
✅ Logback configuration (logback-spring.xml)
✅ MDC สำหรับ contextual logging
✅ Structured logging (JSON)
✅ Request/Response logging filter
✅ Log4j2 เป็น alternative
✅ Audit logging ด้วย AOP
✅ Performance logging
✅ ELK Stack integration
✅ Actuator log management
✅ Security best practices
```

### Logging Checklist

```
Development:
  ✅ ใช้ @Slf4j annotation
  ✅ Log ข้อมูลสำคัญ (user actions, errors)
  ✅ ใช้ placeholders {} ไม่ใช่ string concatenation
  ✅ Log exceptions พร้อม stack trace
  ✅ ใช้ DEBUG level สำหรับ verbose info

Production:
  ✅ Level = INFO หรือ WARN
  ✅ Rolling file appender
  ✅ Async appender
  ✅ JSON format (สำหรับ ELK)
  ✅ MDC สำหรับ request context
  ✅ ไม่ log sensitive data (passwords, PII)
  ✅ Centralized log management (ELK, CloudWatch)
```

### แบบฝึกหัด

```
1. สร้าง logback-spring.xml ที่:
   - dev: console + color, DEBUG level
   - prod: file (rolling), INFO level, JSON format

2. สร้าง MDCFilter ที่ inject:
   - requestId (UUID)
   - userId (จาก JWT)
   - clientIp

3. สร้าง AuditAspect ที่ log ทุก POST, PUT, DELETE requests
   พร้อมข้อมูล: action, resource, userId, timestamp, result
```

---

*[← Part 08: Properties](./part-08-properties-profiles.md) | [Part 10: Testing →](./part-10-testing-basics.md)*

# Part 67: Logging Best Practices
## ขั้นตอนที่ 2281-2320

**ระดับ:** ระดับโลก (World-Class)
**เวลาเรียน:** 6-7 ชั่วโมง
**เป้าหมาย:** เชี่ยวชาญการทำ Structured Logging, Log Correlation, Sensitive Data Masking และการส่ง logs ไปยัง ELK Stack สำหรับ Spring Boot Applications ระดับ Production

---

## ขั้นตอนที่ 2281-2285: Structured Logging กับ Logback JSON

### ทำไมต้องใช้ Structured Logging?

Logging แบบดั้งเดิม (Plain Text) มีปัญหาในระบบ Production ขนาดใหญ่:
- **ค้นหายาก**: ต้อง parse text ที่ไม่มีรูปแบบตายตัว
- **Filter ไม่ได้**: ไม่สามารถ filter ตาม field เฉพาะ
- **Aggregate ยาก**: รวม logs จากหลาย service ลำบาก

Structured Logging แก้ปัญหาเหล่านี้โดยเขียน logs เป็น JSON:

```json
{
  "timestamp": "2024-01-15T10:30:00.000Z",
  "level": "INFO",
  "traceId": "abc123def456",
  "spanId": "789xyz",
  "userId": "user-001",
  "requestId": "req-789",
  "service": "order-service",
  "message": "Order created successfully",
  "orderId": 12345,
  "customerId": "cust-001",
  "totalAmount": 500.00
}
```

### Dependencies

```xml
<!-- pom.xml -->
<dependency>
    <groupId>net.logstash.logback</groupId>
    <artifactId>logstash-logback-encoder</artifactId>
    <version>7.4</version>
</dependency>

<!-- Micrometer Tracing สำหรับ Distributed Tracing -->
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-brave</artifactId>
</dependency>
<dependency>
    <groupId>io.zipkin.reporter2</groupId>
    <artifactId>zipkin-reporter-brave</artifactId>
</dependency>

<!-- Logback Masker -->
<dependency>
    <groupId>ch.qos.logback</groupId>
    <artifactId>logback-classic</artifactId>
</dependency>
```

### Logback Configuration สำหรับ JSON Output

```xml
<!-- src/main/resources/logback-spring.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <springProperty scope="context" name="appName" source="spring.application.name"/>
    <springProperty scope="context" name="activeProfile" source="spring.profiles.active" defaultValue="default"/>

    <!-- Console Appender สำหรับ Development -->
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder class="net.logstash.logback.encoder.LogstashEncoder">
            <!-- ข้อมูลเพิ่มเติมที่ต้องการใส่ใน log ทุก entry -->
            <customFields>{"service":"${appName}","env":"${activeProfile}"}</customFields>

            <!-- รูปแบบ timestamp -->
            <timestampPattern>yyyy-MM-dd'T'HH:mm:ss.SSS'Z'</timestampPattern>
            <timeZone>UTC</timeZone>

            <!-- ลบ fields ที่ไม่ต้องการ -->
            <excludeMdcKeyName>dont_log_me</excludeMdcKeyName>

            <!-- Field name สำหรับ message -->
            <messageFieldName>message</messageFieldName>

            <!-- ใส่ caller info (ชื่อ class, method, line) -->
            <includeCallerData>false</includeCallerData>

            <!-- Mask sensitive fields -->
            <provider class="com.example.logging.MaskingJsonProvider"/>
        </encoder>
    </appender>

    <!-- File Appender สำหรับ Production -->
    <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>logs/application.log</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
            <fileNamePattern>logs/application-%d{yyyy-MM-dd}.%i.log.gz</fileNamePattern>
            <maxFileSize>100MB</maxFileSize>
            <maxHistory>30</maxHistory>
            <totalSizeCap>3GB</totalSizeCap>
        </rollingPolicy>
        <encoder class="net.logstash.logback.encoder.LogstashEncoder">
            <customFields>{"service":"${appName}","env":"${activeProfile}"}</customFields>
        </encoder>
    </appender>

    <!-- Async Appender เพื่อไม่ให้ logging กระทบ Performance -->
    <appender name="ASYNC_FILE" class="ch.qos.logback.classic.AsyncAppender">
        <appender-ref ref="FILE"/>
        <queueSize>512</queueSize>
        <discardingThreshold>0</discardingThreshold>
        <includeCallerData>false</includeCallerData>
        <neverBlock>false</neverBlock>
    </appender>

    <!-- Logger สำหรับ SQL queries -->
    <logger name="org.hibernate.SQL" level="DEBUG" additivity="false">
        <appender-ref ref="CONSOLE"/>
    </logger>

    <!-- Logger สำหรับ Security events -->
    <logger name="com.example.security" level="INFO" additivity="false">
        <appender-ref ref="CONSOLE"/>
        <appender-ref ref="ASYNC_FILE"/>
    </logger>

    <!-- Root Logger -->
    <springProfile name="local,dev">
        <root level="DEBUG">
            <appender-ref ref="CONSOLE"/>
        </root>
    </springProfile>

    <springProfile name="prod">
        <root level="INFO">
            <appender-ref ref="ASYNC_FILE"/>
        </root>
    </springProfile>
</configuration>
```

---

## ขั้นตอนที่ 2286-2292: Log Correlation - traceId, spanId, userId, requestId

### Request Logging Filter

```java
// RequestLoggingFilter.java
@Component
@Order(Ordered.HIGHEST_PRECEDENCE)
public class RequestLoggingFilter implements Filter {

    private static final Logger log = LoggerFactory.getLogger(RequestLoggingFilter.class);

    @Override
    public void doFilter(
        ServletRequest request,
        ServletResponse response,
        FilterChain chain
    ) throws IOException, ServletException {

        HttpServletRequest httpRequest = (HttpServletRequest) request;
        HttpServletResponse httpResponse = (HttpServletResponse) response;

        // สร้าง requestId ถ้ายังไม่มี
        String requestId = httpRequest.getHeader("X-Request-ID");
        if (requestId == null || requestId.isEmpty()) {
            requestId = UUID.randomUUID().toString();
        }

        // ดึง userId จาก JWT token
        String userId = extractUserId(httpRequest);

        // ใส่ Correlation IDs ลงใน MDC (Mapped Diagnostic Context)
        MDC.put("requestId", requestId);
        MDC.put("userId", userId != null ? userId : "anonymous");
        MDC.put("clientIp", getClientIp(httpRequest));
        MDC.put("httpMethod", httpRequest.getMethod());
        MDC.put("requestUri", httpRequest.getRequestURI());

        // ใส่ requestId ใน response header ด้วย
        httpResponse.setHeader("X-Request-ID", requestId);

        long startTime = System.currentTimeMillis();

        try {
            // Log incoming request
            log.info("Incoming request: {} {}",
                httpRequest.getMethod(),
                httpRequest.getRequestURI()
            );

            chain.doFilter(request, response);

        } finally {
            long duration = System.currentTimeMillis() - startTime;

            // Log outgoing response
            log.info("Request completed: {} {} -> {} ({}ms)",
                httpRequest.getMethod(),
                httpRequest.getRequestURI(),
                httpResponse.getStatus(),
                duration
            );

            // ล้าง MDC หลังจบ request
            MDC.clear();
        }
    }

    private String extractUserId(HttpServletRequest request) {
        String authHeader = request.getHeader("Authorization");
        if (authHeader != null && authHeader.startsWith("Bearer ")) {
            try {
                // parse JWT token และดึง userId
                String token = authHeader.substring(7);
                return jwtParser.parseToken(token).getSubject();
            } catch (Exception e) {
                return null;
            }
        }
        return null;
    }

    private String getClientIp(HttpServletRequest request) {
        String xForwardedFor = request.getHeader("X-Forwarded-For");
        if (xForwardedFor != null && !xForwardedFor.isEmpty()) {
            return xForwardedFor.split(",")[0].trim();
        }
        return request.getRemoteAddr();
    }
}
```

### Structured Logger Wrapper

```java
// StructuredLogger.java - Utility สำหรับ Structured Logging
@Component
public class StructuredLogger {

    private final Logger log;

    public StructuredLogger(Class<?> clazz) {
        this.log = LoggerFactory.getLogger(clazz);
    }

    // Log ที่มี business context
    public void logBusinessEvent(
        String event,
        String entityType,
        Object entityId,
        Map<String, Object> attributes
    ) {
        StructuredArguments args = StructuredArguments.entries(attributes);

        log.info("Business event: {}",
            StructuredArguments.keyValue("event", event),
            StructuredArguments.keyValue("entityType", entityType),
            StructuredArguments.keyValue("entityId", entityId),
            args
        );
    }

    // Log Performance metrics
    public void logPerformance(
        String operation,
        long durationMs,
        boolean success,
        Map<String, Object> metadata
    ) {
        if (durationMs > 1000) {
            log.warn("Slow operation detected",
                StructuredArguments.keyValue("operation", operation),
                StructuredArguments.keyValue("durationMs", durationMs),
                StructuredArguments.keyValue("success", success)
            );
        } else {
            log.debug("Operation completed",
                StructuredArguments.keyValue("operation", operation),
                StructuredArguments.keyValue("durationMs", durationMs),
                StructuredArguments.keyValue("success", success)
            );
        }
    }
}

// การใช้งานใน Service
@Service
public class OrderService {

    private static final Logger log = LoggerFactory.getLogger(OrderService.class);

    public Order createOrder(CreateOrderRequest request) {
        log.info("Creating order",
            StructuredArguments.keyValue("customerId", request.getCustomerId()),
            StructuredArguments.keyValue("itemCount", request.getItems().size())
        );

        try {
            Order order = processOrder(request);

            log.info("Order created successfully",
                StructuredArguments.keyValue("orderId", order.getId()),
                StructuredArguments.keyValue("customerId", order.getCustomerId()),
                StructuredArguments.keyValue("totalAmount", order.getTotalAmount()),
                StructuredArguments.keyValue("status", order.getStatus())
            );

            return order;

        } catch (InsufficientStockException e) {
            log.warn("Order creation failed - insufficient stock",
                StructuredArguments.keyValue("customerId", request.getCustomerId()),
                StructuredArguments.keyValue("productId", e.getProductId()),
                StructuredArguments.keyValue("requestedQuantity", e.getRequestedQuantity()),
                StructuredArguments.keyValue("availableStock", e.getAvailableStock())
            );
            throw e;

        } catch (Exception e) {
            log.error("Unexpected error creating order",
                StructuredArguments.keyValue("customerId", request.getCustomerId()),
                e
            );
            throw e;
        }
    }
}
```

---

## ขั้นตอนที่ 2293-2298: Log Levels Strategy

### กลยุทธ์การใช้ Log Levels

ระบบ Production ที่ดีต้องมีกลยุทธ์ที่ชัดเจนว่าควร log อะไรที่ level ไหน:

```java
// LogLevelStrategyExample.java
@Service
@Slf4j
public class PaymentService {

    // ERROR: สถานการณ์ที่ต้องการการแก้ไขทันที
    // - System failures ที่กระทบ users
    // - Data corruption
    // - External dependency ล้มเหลว
    public void processPayment(PaymentRequest request) {
        try {
            externalPaymentGateway.charge(request);
        } catch (PaymentGatewayException e) {
            // ERROR: External system failure
            log.error("Payment gateway failure - manual intervention required",
                StructuredArguments.keyValue("orderId", request.getOrderId()),
                StructuredArguments.keyValue("amount", request.getAmount()),
                StructuredArguments.keyValue("gateway", request.getGateway()),
                StructuredArguments.keyValue("errorCode", e.getErrorCode()),
                e
            );
            throw new PaymentProcessingException("Payment failed", e);
        }
    }

    // WARN: สถานการณ์ที่น่าสงสัย แต่ยังทำงานได้
    // - Business rules violations
    // - Performance degradation
    // - Deprecated features being used
    // - Retry attempts
    public void retryPayment(String orderId, int attempt) {
        if (attempt > 3) {
            // WARN: ต้องการความสนใจแต่ยังไม่ถึงขั้น error
            log.warn("High number of payment retries",
                StructuredArguments.keyValue("orderId", orderId),
                StructuredArguments.keyValue("attempt", attempt),
                StructuredArguments.keyValue("maxAttempts", 5)
            );
        }
    }

    // INFO: Business events ที่สำคัญ
    // - Order created/confirmed/cancelled
    // - Payment processed
    // - User registered/login
    // - Significant state changes
    public PaymentResult confirmPayment(String paymentId, String transactionId) {
        // INFO: Business milestone
        log.info("Payment confirmed",
            StructuredArguments.keyValue("paymentId", paymentId),
            StructuredArguments.keyValue("transactionId", transactionId),
            StructuredArguments.keyValue("timestamp", Instant.now())
        );
        return updatePaymentStatus(paymentId, transactionId);
    }

    // DEBUG: ข้อมูลสำหรับ debugging (ปิดใน production)
    // - Method entry/exit
    // - Object states
    // - Intermediate calculations
    // - Database queries (SQL)
    private void validatePaymentRequest(PaymentRequest request) {
        log.debug("Validating payment request",
            StructuredArguments.keyValue("orderId", request.getOrderId()),
            StructuredArguments.keyValue("currency", request.getCurrency()),
            StructuredArguments.keyValue("validationRules", getValidationRules())
        );
    }

    // TRACE: ข้อมูลละเอียดมากๆ สำหรับ deep debugging
    // - Loop iterations
    // - Detailed algorithm steps
    // - All HTTP headers
    private void processPaymentItems(List<PaymentItem> items) {
        items.forEach(item -> {
            log.trace("Processing payment item",
                StructuredArguments.keyValue("itemId", item.getId()),
                StructuredArguments.keyValue("productId", item.getProductId()),
                StructuredArguments.keyValue("price", item.getPrice())
            );
        });
    }
}
```

### Log Level Guidelines

```yaml
# application.yml - Log Level Configuration
logging:
  level:
    root: INFO

    # Application logs
    com.example: INFO
    com.example.service: INFO
    com.example.repository: WARN  # ไม่ต้องการ SQL logs ใน prod

    # Framework logs
    org.springframework.web: WARN
    org.springframework.security: WARN
    org.hibernate.SQL: WARN        # เปิด DEBUG เมื่อต้องการดู SQL

    # ปิด verbose logs
    org.springframework.boot.autoconfigure: WARN
    com.zaxxer.hikari: WARN

---
# application-dev.yml
logging:
  level:
    com.example: DEBUG
    org.hibernate.SQL: DEBUG
    org.hibernate.type.descriptor.sql: TRACE  # แสดง parameter values
```

---

## ขั้นตอนที่ 2299-2305: Sensitive Data Masking ใน Logs

### ปัญหา: Logging ข้อมูลที่ไม่ควร Log

```java
// BAD: อย่าทำแบบนี้!
log.info("Processing payment for user: {}, card: {}, cvv: {}",
    userId, creditCardNumber, cvv);
// Output: Processing payment for user: john@email.com, card: 4532-1234-5678-9012, cvv: 123
```

### การ Mask ข้อมูล Sensitive

```java
// MaskingUtils.java
public final class MaskingUtils {

    private MaskingUtils() {}

    // Mask credit card: แสดงเฉพาะ 4 ตัวท้าย
    public static String maskCreditCard(String cardNumber) {
        if (cardNumber == null || cardNumber.length() < 4) {
            return "****";
        }
        String cleaned = cardNumber.replaceAll("[^0-9]", "");
        return "*".repeat(cleaned.length() - 4) + cleaned.substring(cleaned.length() - 4);
    }

    // Mask email: แสดงเฉพาะ 3 ตัวแรกและ domain
    public static String maskEmail(String email) {
        if (email == null || !email.contains("@")) {
            return "***@***.***";
        }
        String[] parts = email.split("@");
        String local = parts[0];
        String domain = parts[1];

        if (local.length() <= 3) {
            return local + "***@" + domain;
        }
        return local.substring(0, 3) + "***@" + domain;
    }

    // Mask phone: แสดงเฉพาะ 4 ตัวท้าย
    public static String maskPhone(String phone) {
        if (phone == null || phone.length() < 4) {
            return "****";
        }
        return "*".repeat(phone.length() - 4) + phone.substring(phone.length() - 4);
    }

    // Mask password: แสดงแค่ [PROTECTED]
    public static String maskPassword(String password) {
        return "[PROTECTED]";
    }

    // Mask sensitive JSON fields
    public static String maskJsonFields(String json, Set<String> sensitiveFields) {
        if (json == null) return null;

        String maskedJson = json;
        for (String field : sensitiveFields) {
            maskedJson = maskedJson.replaceAll(
                "\"" + field + "\"\\s*:\\s*\"[^\"]*\"",
                "\"" + field + "\": \"[MASKED]\""
            );
        }
        return maskedJson;
    }
}
```

### Custom Logback Masking Provider

```java
// MaskingJsonProvider.java - Logback JSON Provider ที่ Mask อัตโนมัติ
public class MaskingJsonProvider extends JsonProvider<ILoggingEvent> {

    private static final Set<String> SENSITIVE_FIELDS = Set.of(
        "password", "creditCard", "cvv", "ssn",
        "token", "secret", "apiKey", "privateKey"
    );

    private static final Pattern CREDIT_CARD_PATTERN =
        Pattern.compile("\\b(?:\\d{4}[-\\s]?){3}\\d{4}\\b");

    private static final Pattern EMAIL_PATTERN =
        Pattern.compile("\\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\\.[A-Z|a-z]{2,}\\b");

    @Override
    public void writeTo(JsonGenerator generator, ILoggingEvent event) throws IOException {
        // ดึง message และ mask sensitive data
        String message = event.getFormattedMessage();
        String maskedMessage = maskSensitiveData(message);

        generator.writeStringField("message", maskedMessage);

        // Mask MDC values ด้วย
        Map<String, String> mdc = event.getMDCPropertyMap();
        if (mdc != null) {
            for (Map.Entry<String, String> entry : mdc.entrySet()) {
                String key = entry.getKey();
                String value = isSensitiveField(key)
                    ? "[MASKED]"
                    : maskSensitiveData(entry.getValue());
                generator.writeStringField(key, value);
            }
        }
    }

    private String maskSensitiveData(String text) {
        if (text == null) return null;

        // Mask credit card numbers
        text = CREDIT_CARD_PATTERN.matcher(text)
            .replaceAll(m -> MaskingUtils.maskCreditCard(m.group()));

        return text;
    }

    private boolean isSensitiveField(String fieldName) {
        return SENSITIVE_FIELDS.stream()
            .anyMatch(sensitive -> fieldName.toLowerCase().contains(sensitive.toLowerCase()));
    }
}
```

### Masked DTO สำหรับ Logging

```java
// PaymentLoggable.java - DTO ที่ปลอดภัยสำหรับ log
public class PaymentLoggable {

    private final String orderId;
    private final String maskedCardNumber;
    private final String maskedEmail;
    private final BigDecimal amount;
    private final String currency;

    public static PaymentLoggable from(PaymentRequest request) {
        return new PaymentLoggable(
            request.getOrderId(),
            MaskingUtils.maskCreditCard(request.getCardNumber()),
            MaskingUtils.maskEmail(request.getEmail()),
            request.getAmount(),
            request.getCurrency()
        );
    }

    @Override
    public String toString() {
        return String.format(
            "Payment{orderId='%s', card='%s', email='%s', amount=%s %s}",
            orderId, maskedCardNumber, maskedEmail, amount, currency
        );
    }
}

// การใช้งาน
public void processPayment(PaymentRequest request) {
    // GOOD: ใช้ Loggable DTO ที่ mask ข้อมูลแล้ว
    log.info("Processing payment: {}", PaymentLoggable.from(request));

    // หรือใช้แบบ structured
    log.info("Processing payment",
        StructuredArguments.keyValue("orderId", request.getOrderId()),
        StructuredArguments.keyValue("card", MaskingUtils.maskCreditCard(request.getCardNumber())),
        StructuredArguments.keyValue("amount", request.getAmount())
    );
}
```

---

## ขั้นตอนที่ 2306-2312: ELK Stack Integration

### Docker Compose สำหรับ ELK Stack

```yaml
# docker-compose-elk.yml
version: '3.8'

services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.11.0
    container_name: elasticsearch
    environment:
      - node.name=elasticsearch
      - cluster.name=docker-cluster
      - discovery.type=single-node
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
      - xpack.security.enabled=false
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - elasticsearch_data:/usr/share/elasticsearch/data
    ports:
      - "9200:9200"
    networks:
      - elk

  logstash:
    image: docker.elastic.co/logstash/logstash:8.11.0
    container_name: logstash
    volumes:
      - ./logstash/pipeline:/usr/share/logstash/pipeline
      - ./logstash/config:/usr/share/logstash/config
    ports:
      - "5000:5000/tcp"   # TCP input
      - "5000:5000/udp"   # UDP input
      - "5044:5044"       # Beats input
      - "9600:9600"       # Logstash monitoring API
    environment:
      LS_JAVA_OPTS: "-Xmx256m -Xms256m"
    networks:
      - elk
    depends_on:
      - elasticsearch

  kibana:
    image: docker.elastic.co/kibana/kibana:8.11.0
    container_name: kibana
    ports:
      - "5601:5601"
    environment:
      ELASTICSEARCH_URL: http://elasticsearch:9200
      ELASTICSEARCH_HOSTS: '["http://elasticsearch:9200"]'
    networks:
      - elk
    depends_on:
      - elasticsearch

  filebeat:
    image: docker.elastic.co/beats/filebeat:8.11.0
    container_name: filebeat
    user: root
    volumes:
      - ./filebeat/filebeat.yml:/usr/share/filebeat/filebeat.yml:ro
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./logs:/app/logs:ro
    networks:
      - elk
    depends_on:
      - logstash

networks:
  elk:
    driver: bridge

volumes:
  elasticsearch_data:
```

### Logstash Pipeline Configuration

```ruby
# logstash/pipeline/application.conf
input {
  # รับ logs จาก Filebeat
  beats {
    port => 5044
  }

  # รับ logs โดยตรงจาก Spring Boot ผ่าน TCP
  tcp {
    port => 5000
    codec => json_lines
  }
}

filter {
  # Parse JSON logs
  if [message] =~ /^\{/ {
    json {
      source => "message"
      target => "parsed"
      remove_field => ["message"]
    }

    # ย้าย fields จาก parsed ขึ้นมา root level
    mutate {
      rename => {
        "[parsed][timestamp]" => "@timestamp"
        "[parsed][level]"     => "log_level"
        "[parsed][service]"   => "service"
        "[parsed][traceId]"   => "trace_id"
        "[parsed][spanId]"    => "span_id"
        "[parsed][userId]"    => "user_id"
        "[parsed][requestId]" => "request_id"
        "[parsed][message]"   => "message"
      }
    }
  }

  # แปลง log level เป็น uppercase
  mutate {
    uppercase => ["log_level"]
  }

  # เพิ่ม environment tag
  if [@metadata][pipeline] {
    mutate {
      add_tag => ["processed"]
    }
  }

  # Geo-location สำหรับ client IP
  if [client_ip] {
    geoip {
      source => "client_ip"
      target => "geo"
    }
  }

  # Drop health check logs (ลด noise)
  if [request_uri] =~ /\/actuator\/health/ {
    drop {}
  }
}

output {
  elasticsearch {
    hosts => ["elasticsearch:9200"]
    index => "spring-boot-logs-%{+YYYY.MM.dd}"
    # ใช้ template สำหรับ index mapping
    template_name => "spring-boot-logs"
    template_overwrite => true
  }

  # Debug output (ปิดใน production)
  # stdout { codec => rubydebug }
}
```

### Filebeat Configuration

```yaml
# filebeat/filebeat.yml
filebeat.inputs:
  - type: log
    enabled: true
    paths:
      - /app/logs/*.log
    json.keys_under_root: true
    json.add_error_key: true
    json.overwrite_keys: true
    multiline.pattern: '^{'
    multiline.negate: true
    multiline.match: after

  # Docker container logs
  - type: docker
    containers.ids:
      - "*"
    processors:
      - add_docker_metadata: ~
      - decode_json_fields:
          fields: ["message"]
          target: ""
          overwrite_keys: true

filebeat.config.modules:
  path: ${path.config}/modules.d/*.yml
  reload.enabled: false

processors:
  - add_host_metadata:
      when.not.contains.tags: forwarded
  - add_cloud_metadata: ~

output.logstash:
  hosts: ["logstash:5044"]

logging.level: warning
```

### Spring Boot Log to Logstash โดยตรง

```xml
<!-- logback-spring.xml - เพิ่ม Logstash Appender -->
<appender name="LOGSTASH" class="net.logstash.logback.appender.LogstashTcpSocketAppender">
    <destination>logstash:5000</destination>
    <encoder class="net.logstash.logback.encoder.LogstashEncoder">
        <customFields>{"service":"${appName}","env":"${activeProfile}"}</customFields>
    </encoder>
    <!-- Reconnect หาก connection หาย -->
    <reconnectionDelay>10 seconds</reconnectionDelay>
    <!-- Connection timeout -->
    <connectionTimeout>10 seconds</connectionTimeout>
    <!-- Async wrapper -->
    <writeBufferSize>8192</writeBufferSize>
</appender>
```

---

## ขั้นตอนที่ 2313-2317: Distributed Logging Patterns

### Correlation ID Propagation ระหว่าง Services

```java
// CorrelationIdInterceptor.java - สำหรับ RestTemplate
@Component
public class CorrelationIdInterceptor implements ClientHttpRequestInterceptor {

    @Override
    public ClientHttpResponse intercept(
        HttpRequest request,
        byte[] body,
        ClientHttpRequestExecution execution
    ) throws IOException {

        // ส่ง correlation headers ต่อไปยัง downstream service
        String traceId = MDC.get("traceId");
        String spanId = MDC.get("spanId");
        String requestId = MDC.get("requestId");
        String userId = MDC.get("userId");

        if (traceId != null) {
            request.getHeaders().set("X-Trace-ID", traceId);
        }
        if (requestId != null) {
            request.getHeaders().set("X-Request-ID", requestId);
        }
        if (userId != null) {
            request.getHeaders().set("X-User-ID", userId);
        }

        return execution.execute(request, body);
    }
}

// Config สำหรับ RestTemplate
@Configuration
public class RestTemplateConfig {

    @Bean
    public RestTemplate restTemplate(CorrelationIdInterceptor correlationIdInterceptor) {
        RestTemplate restTemplate = new RestTemplate();
        restTemplate.setInterceptors(List.of(correlationIdInterceptor));
        return restTemplate;
    }
}
```

### Log Aggregation Pattern

```java
// AggregatedLogEntry.java - รวม logs จากหลาย services
@Data
@Builder
public class AggregatedLogEntry {
    private String traceId;
    private String requestId;
    private String service;
    private String timestamp;
    private String level;
    private String message;
    private Map<String, Object> context;

    // สร้าง correlation link สำหรับ Kibana
    public String getKibanaLink(String kibanaUrl) {
        return String.format(
            "%s/app/discover#/?_g=()&_a=(query:(language:kuery,query:'traceId:\"%s\"'))",
            kibanaUrl, traceId
        );
    }
}
```

---

## ขั้นตอนที่ 2318-2320: Log-Based Alerting

### Kibana Alert Rules

```json
// kibana_alert_rule.json - ตัวอย่าง Alert Rule
{
  "name": "High Error Rate Alert",
  "rule_type_id": "logs.alert.document.count",
  "params": {
    "logView": {
      "type": "log-view-reference",
      "logViewId": "default"
    },
    "count": {
      "value": 10,
      "comparator": ">"
    },
    "timeUnit": "m",
    "timeSize": 5,
    "criteria": [
      {
        "field": "log_level",
        "comparator": "equals",
        "value": "ERROR"
      }
    ],
    "groupBy": ["service"]
  },
  "schedule": {
    "interval": "1m"
  },
  "actions": [
    {
      "actionTypeId": ".slack",
      "params": {
        "message": "⚠️ High error rate detected in service {{context.group}}: {{context.matchingDocuments}} errors in last 5 minutes"
      }
    }
  ]
}
```

### Custom Log Alerting กับ Spring

```java
// LogAlertService.java - ส่ง alert เมื่อพบ error patterns
@Service
@Slf4j
public class LogAlertService {

    private final SlackNotificationService slackService;
    private final AtomicInteger errorCount = new AtomicInteger(0);
    private volatile Instant windowStart = Instant.now();

    @EventListener(ApplicationReadyEvent.class)
    public void startMonitoring() {
        // Reset counter ทุก 5 นาที
        Executors.newScheduledThreadPool(1)
            .scheduleAtFixedRate(this::resetWindow, 5, 5, TimeUnit.MINUTES);
    }

    @EventListener(ErrorLoggingEvent.class)
    public void onError(ErrorLoggingEvent event) {
        int count = errorCount.incrementAndGet();

        // ส่ง alert เมื่อมี error มากกว่า threshold
        if (count >= 10) {
            String message = String.format(
                "🚨 High error rate: %d errors in last 5 minutes\n" +
                "Service: %s\n" +
                "Last error: %s",
                count,
                event.getService(),
                event.getMessage()
            );

            slackService.sendAlert("#alerts", message);
            errorCount.set(0); // reset หลัง alert
        }
    }

    private void resetWindow() {
        errorCount.set(0);
        windowStart = Instant.now();
    }
}
```

### Kibana Dashboard Queries

```
# KQL (Kibana Query Language) ตัวอย่าง

# หา errors ทั้งหมดใน 1 ชั่วโมงที่ผ่านมา
log_level: "ERROR" and @timestamp > now-1h

# หา slow requests (> 1000ms)
duration_ms > 1000 and http_method: "POST"

# ตาม trace ID
trace_id: "abc123def456"

# หา payment errors
service: "payment-service" and log_level: "ERROR"

# Security events
event_type: "AUTHENTICATION_FAILURE" or event_type: "AUTHORIZATION_FAILURE"
```

---

## สรุป Part 67: Logging Best Practices

### Logging Checklist

- [ ] ใช้ Structured Logging (JSON format)
- [ ] มี Correlation IDs ทุก request (traceId, requestId, userId)
- [ ] Mask ข้อมูล sensitive ทุกจุด (password, card number, email)
- [ ] กำหนด Log Level ตาม business importance
- [ ] ส่ง logs ไปยัง centralized system (ELK)
- [ ] มี Alerting สำหรับ error patterns
- [ ] ไม่ log ข้อมูลที่ไม่จำเป็น (ลด noise)
- [ ] มี Async Appender เพื่อไม่กระทบ performance

### Anti-patterns ที่ต้องหลีกเลี่ยง

```java
// ❌ BAD: Log ข้อมูล sensitive
log.info("User {} logged in with password {}", username, password);

// ❌ BAD: Log ใน loop โดยไม่จำเป็น
for (Order order : orders) {
    log.info("Processing order {}", order.getId()); // มาก noise เกินไป
}

// ❌ BAD: ไม่มี context
log.error("Something went wrong");

// ✅ GOOD: ข้อมูลครบ ปลอดภัย
log.info("User authenticated",
    StructuredArguments.keyValue("userId", userId),
    StructuredArguments.keyValue("loginMethod", "password")
);

// ✅ GOOD: Log สรุป ไม่ใช่ทีละ item
log.info("Processing orders batch",
    StructuredArguments.keyValue("batchSize", orders.size()),
    StructuredArguments.keyValue("batchId", batchId)
);

// ✅ GOOD: มี context ครบถ้วน
log.error("Payment processing failed",
    StructuredArguments.keyValue("orderId", orderId),
    StructuredArguments.keyValue("errorCode", e.getErrorCode()),
    e
);
```

---

*[← Part 66: Testing Strategies](./part-66-testing-strategies.md) | [Part 68: Configuration Management →](./part-68-configuration-management.md)*

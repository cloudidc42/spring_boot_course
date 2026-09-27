# Part 68: Configuration Management
## ขั้นตอนที่ 2321-2360

**ระดับ:** ระดับโลก (World-Class)
**เวลาเรียน:** 6-7 ชั่วโมง
**เป้าหมาย:** เชี่ยวชาญการจัดการ Configuration สำหรับระบบ Microservices ด้วย Spring Cloud Config Server, HashiCorp Vault, Feature Toggles และ Runtime Configuration Updates

---

## ขั้นตอนที่ 2321-2325: Spring Cloud Config Server

### ทำความเข้าใจ Centralized Configuration

ในระบบ Microservices ที่มีหลาย services และหลาย environment การจัดการ configuration แบบแยกไฟล์ในแต่ละ service ทำให้เกิดปัญหา:
- Config กระจัดกระจาย หาลำบาก
- ต้อง deploy ใหม่เมื่อ config เปลี่ยน
- ไม่มี audit trail ว่าใครเปลี่ยนอะไรเมื่อไหร่
- Secret management ไม่ปลอดภัย

Spring Cloud Config Server แก้ปัญหาเหล่านี้โดย:
- เก็บ config ทั้งหมดไว้ที่ Git repository
- Services ดึง config จาก Config Server
- รองรับ refresh config โดยไม่ต้อง restart

### Setup Config Server

```xml
<!-- config-server/pom.xml -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-config-server</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```

```java
// ConfigServerApplication.java
@SpringBootApplication
@EnableConfigServer
public class ConfigServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(ConfigServerApplication.class, args);
    }
}
```

```yaml
# config-server/src/main/resources/application.yml
server:
  port: 8888

spring:
  application:
    name: config-server

  cloud:
    config:
      server:
        git:
          # URL ของ Git repository ที่เก็บ config
          uri: https://github.com/your-org/spring-config-repo
          default-label: main
          # Clone ลง local เพื่อ performance
          clone-on-start: true
          # Force pull เมื่อ local มีการเปลี่ยนแปลง
          force-pull: true
          # Timeout สำหรับ git operations
          timeout: 10
          # Pattern สำหรับหลาย repositories
          repos:
            order-service:
              pattern: order-service/*
              uri: https://github.com/your-org/order-config
            payment-service:
              pattern: payment-service/*
              uri: https://github.com/your-org/payment-config
          # Authentication สำหรับ private repo
          username: ${GIT_USERNAME}
          password: ${GIT_PASSWORD}

  security:
    user:
      name: config-admin
      password: ${CONFIG_SERVER_PASSWORD}

management:
  endpoints:
    web:
      exposure:
        include: health,info,refresh
```

### Git Config Repository Structure

```
config-repo/
├── application.yml              # Config ที่ใช้กับทุก services
├── application-dev.yml          # Config สำหรับ dev environment (ทุก services)
├── application-prod.yml         # Config สำหรับ prod environment (ทุก services)
├── order-service.yml            # Config เฉพาะ order-service
├── order-service-dev.yml        # Config order-service สำหรับ dev
├── order-service-prod.yml       # Config order-service สำหรับ prod
├── payment-service.yml          # Config เฉพาะ payment-service
└── payment-service-prod.yml     # Config payment-service สำหรับ prod
```

```yaml
# config-repo/application.yml - Global config
app:
  timezone: Asia/Bangkok
  locale: th_TH

management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  endpoint:
    health:
      show-details: when_authorized

logging:
  level:
    root: INFO
```

```yaml
# config-repo/order-service.yml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/orders
    driver-class-name: org.postgresql.Driver
    hikari:
      maximum-pool-size: 10
      minimum-idle: 5
      connection-timeout: 30000

order:
  max-items-per-order: 50
  default-currency: THB
  notification:
    email:
      enabled: true
      template: order-confirmation
    sms:
      enabled: false
```

```yaml
# config-repo/order-service-prod.yml
spring:
  datasource:
    url: jdbc:postgresql://prod-db:5432/orders
    hikari:
      maximum-pool-size: 50
      minimum-idle: 10

order:
  notification:
    sms:
      enabled: true
```

### Config Client Setup

```xml
<!-- order-service/pom.xml -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-config</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-bootstrap</artifactId>
</dependency>
```

```yaml
# order-service/src/main/resources/bootstrap.yml
spring:
  application:
    name: order-service  # ใช้ชื่อนี้ในการดึง config
  profiles:
    active: ${SPRING_PROFILES_ACTIVE:dev}

  cloud:
    config:
      uri: http://config-server:8888
      username: config-admin
      password: ${CONFIG_SERVER_PASSWORD}
      # Retry ถ้า config server ไม่พร้อม
      fail-fast: true
      retry:
        max-attempts: 6
        initial-interval: 1000
        max-interval: 2000
        multiplier: 1.1
```

---

## ขั้นตอนที่ 2326-2332: @RefreshScope สำหรับ Runtime Config Updates

### การ Refresh Config โดยไม่ Restart

```java
// OrderConfig.java - Config Properties ที่ Refresh ได้
@Configuration
@ConfigurationProperties(prefix = "order")
@RefreshScope  // ทำให้ Bean นี้ถูกสร้างใหม่เมื่อมีการ refresh config
@Validated
public class OrderConfig {

    @NotNull
    @Min(1)
    @Max(100)
    private Integer maxItemsPerOrder;

    @NotBlank
    private String defaultCurrency;

    private NotificationConfig notification = new NotificationConfig();

    @Data
    public static class NotificationConfig {
        private EmailConfig email = new EmailConfig();
        private SmsConfig sms = new SmsConfig();
    }

    @Data
    public static class EmailConfig {
        private boolean enabled = true;
        private String template;
        private String fromAddress;
    }

    @Data
    public static class SmsConfig {
        private boolean enabled = false;
        private String provider;
    }

    // Getters and Setters...
}

// OrderService.java - ใช้ Config ที่ Refresh ได้
@Service
public class OrderService {

    private final OrderConfig orderConfig;

    public OrderService(OrderConfig orderConfig) {
        this.orderConfig = orderConfig;
    }

    public Order createOrder(CreateOrderRequest request) {
        // orderConfig จะได้ค่าใหม่อัตโนมัติหลัง refresh
        if (request.getItems().size() > orderConfig.getMaxItemsPerOrder()) {
            throw new ValidationException(
                "Too many items. Maximum allowed: " + orderConfig.getMaxItemsPerOrder()
            );
        }
        // ...
    }
}
```

### Refresh Config ผ่าน Actuator

```bash
# Refresh config ของ service เดียว
curl -X POST http://order-service:8080/actuator/refresh

# Response
["order.maxItemsPerOrder", "order.notification.sms.enabled"]
```

### Spring Cloud Bus สำหรับ Broadcast Refresh

```xml
<!-- เพิ่ม Spring Cloud Bus เพื่อ broadcast refresh ไปยังทุก instances -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-bus-amqp</artifactId>
</dependency>
```

```yaml
# application.yml
spring:
  rabbitmq:
    host: rabbitmq
    port: 5672
    username: ${RABBITMQ_USERNAME}
    password: ${RABBITMQ_PASSWORD}

  cloud:
    bus:
      enabled: true
      refresh:
        enabled: true
```

```bash
# Refresh ทุก services พร้อมกัน
curl -X POST http://config-server:8888/actuator/busrefresh

# Refresh เฉพาะ service ที่ต้องการ
curl -X POST http://config-server:8888/actuator/busrefresh/order-service
```

---

## ขั้นตอนที่ 2333-2340: HashiCorp Vault Integration

### ทำไมต้องใช้ Vault?

Git repository ไม่เหมาะสำหรับ secrets เช่น:
- Database passwords
- API keys
- JWT secrets
- TLS certificates

Vault แก้ปัญหาโดย:
- เข้ารหัส secrets ทั้งหมด
- มี audit log ทุก access
- รองรับ dynamic secrets (สร้าง credentials ชั่วคราว)
- มี lease management สำหรับ rotation

### Docker Compose Setup

```yaml
# docker-compose.yml
services:
  vault:
    image: vault:1.15
    container_name: vault
    cap_add:
      - IPC_LOCK
    environment:
      VAULT_DEV_ROOT_TOKEN_ID: myroot
      VAULT_DEV_LISTEN_ADDRESS: 0.0.0.0:8200
    ports:
      - "8200:8200"
    command: server -dev
```

### เพิ่ม Secrets ใน Vault

```bash
# Login
vault login myroot

# Enable KV secrets engine
vault secrets enable -path=secret kv-v2

# เพิ่ม secrets สำหรับ order-service
vault kv put secret/order-service/dev \
    db.password="dev-secret-password" \
    jwt.secret="dev-jwt-secret-key-256bits" \
    stripe.api-key="sk_test_xxxxx"

# สำหรับ prod
vault kv put secret/order-service/prod \
    db.password="prod-very-secret-password" \
    jwt.secret="prod-jwt-secret-key-256bits" \
    stripe.api-key="sk_live_xxxxx"

# ตรวจสอบ
vault kv get secret/order-service/dev
```

### Spring Boot + Vault Integration

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-vault-config</artifactId>
</dependency>
```

```yaml
# bootstrap.yml
spring:
  application:
    name: order-service
  profiles:
    active: ${SPRING_PROFILES_ACTIVE:dev}

  cloud:
    vault:
      uri: http://vault:8200
      authentication: TOKEN
      token: ${VAULT_TOKEN}
      # หรือใช้ AppRole (แนะนำสำหรับ production)
      # authentication: APPROLE
      # app-role:
      #   role-id: ${VAULT_ROLE_ID}
      #   secret-id: ${VAULT_SECRET_ID}

      # Paths ที่จะดึง secrets
      kv:
        enabled: true
        backend: secret
        profiles-path: /data
        default-context: order-service

      # Renew lease อัตโนมัติ
      fail-fast: true
```

### Dynamic Database Credentials กับ Vault

```yaml
# ใช้ Vault สร้าง database credentials แบบชั่วคราว
spring:
  cloud:
    vault:
      database:
        enabled: true
        role: order-service-role
        backend: database

# Vault Policy สำหรับ order-service
path "database/creds/order-service-role" {
  capabilities = ["read"]
}

path "secret/data/order-service/*" {
  capabilities = ["read"]
}
```

```java
// VaultConfig.java
@Configuration
public class VaultConfig {

    @Bean
    @ConfigurationProperties("spring.datasource")
    public DataSource dataSource(
        @Value("${spring.datasource.url}") String url,
        @Value("${spring.datasource.username}") String username,
        @Value("${spring.datasource.password}") String password
    ) {
        // Credentials มาจาก Vault โดยอัตโนมัติ
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl(url);
        config.setUsername(username);
        config.setPassword(password);
        return new HikariDataSource(config);
    }
}
```

---

## ขั้นตอนที่ 2341-2347: Configuration Validation

### @Validated สำหรับ Config Properties

```java
// DatabaseConfig.java - Config ที่มี Validation
@ConfigurationProperties(prefix = "app.database")
@Validated
@Data
public class DatabaseConfig {

    @NotBlank(message = "Database host must not be blank")
    private String host;

    @Min(value = 1, message = "Port must be at least 1")
    @Max(value = 65535, message = "Port must be at most 65535")
    private int port = 5432;

    @NotBlank
    private String name;

    @Min(1)
    @Max(200)
    private int maxPoolSize = 10;

    @Min(0)
    private int minIdle = 5;

    @DurationMin(seconds = 1)
    @DurationMax(seconds = 30)
    private Duration connectionTimeout = Duration.ofSeconds(10);

    // Custom validation
    @AssertTrue(message = "minIdle must be less than or equal to maxPoolSize")
    public boolean isPoolSizeValid() {
        return minIdle <= maxPoolSize;
    }
}

// CacheConfig.java - Cache Configuration
@ConfigurationProperties(prefix = "app.cache")
@Validated
@Data
public class CacheConfig {

    @NotNull
    private RedisConfig redis = new RedisConfig();

    @Min(60)
    @Max(86400)
    private int defaultTtlSeconds = 300;

    @Data
    public static class RedisConfig {
        @NotBlank
        private String host = "localhost";

        @Min(1)
        @Max(65535)
        private int port = 6379;

        @Min(0)
        @Max(15)
        private int database = 0;

        private String password;

        @Min(1)
        @Max(100)
        private int maxConnections = 10;
    }
}
```

### Custom Config Validator

```java
// ConfigurationValidator.java - Custom validator สำหรับ Config
@Component
public class ConfigurationValidator implements ApplicationListener<ApplicationReadyEvent> {

    private static final Logger log = LoggerFactory.getLogger(ConfigurationValidator.class);

    private final DatabaseConfig databaseConfig;
    private final CacheConfig cacheConfig;
    private final OrderConfig orderConfig;

    @Override
    public void onApplicationEvent(ApplicationReadyEvent event) {
        validateDatabaseConfig();
        validateCacheConfig();
        validateOrderConfig();
        log.info("All configurations validated successfully");
    }

    private void validateDatabaseConfig() {
        try {
            // ทดสอบ database connection
            DataSource dataSource = event.getApplicationContext().getBean(DataSource.class);
            dataSource.getConnection().close();
            log.info("Database connection validated: {}:{}/{}",
                databaseConfig.getHost(),
                databaseConfig.getPort(),
                databaseConfig.getName()
            );
        } catch (Exception e) {
            throw new IllegalStateException(
                "Failed to validate database configuration: " + e.getMessage(), e
            );
        }
    }

    private void validateCacheConfig() {
        // ตรวจสอบค่า config ที่สัมพันธ์กัน
        if (cacheConfig.getDefaultTtlSeconds() < 60) {
            throw new IllegalStateException(
                "Cache TTL too short. Minimum 60 seconds required."
            );
        }
    }

    private void validateOrderConfig() {
        if (orderConfig.getMaxItemsPerOrder() < 1) {
            throw new IllegalStateException(
                "maxItemsPerOrder must be at least 1"
            );
        }
    }
}
```

---

## ขั้นตอนที่ 2348-2354: Environment-Specific Configurations

### Multi-Environment Setup

```
src/main/resources/
├── application.yml           # Base config (ทุก environment)
├── application-local.yml     # Developer local machine
├── application-dev.yml       # Development server
├── application-staging.yml   # Staging/UAT environment
└── application-prod.yml      # Production
```

```yaml
# application.yml - Base Configuration
spring:
  application:
    name: order-service

  jpa:
    hibernate:
      ddl-auto: validate  # ใน prod ห้ามใช้ create/update
    show-sql: false
    properties:
      hibernate:
        format_sql: false

server:
  port: 8080
  shutdown: graceful  # Graceful shutdown

app:
  order:
    max-items: 50
    currency: THB
```

```yaml
# application-local.yml - สำหรับ Developer
spring:
  jpa:
    hibernate:
      ddl-auto: create-drop  # สร้างใหม่ทุกครั้งที่ start
    show-sql: true
    properties:
      hibernate:
        format_sql: true

  h2:
    console:
      enabled: true  # เปิด H2 Console

logging:
  level:
    com.example: DEBUG
    org.hibernate.SQL: DEBUG
```

```yaml
# application-dev.yml - Development Server
spring:
  datasource:
    url: jdbc:postgresql://dev-db:5432/orders_dev
    username: ${DB_USERNAME:dev_user}
    password: ${DB_PASSWORD:dev_password}

  jpa:
    hibernate:
      ddl-auto: update  # Update schema โดยอัตโนมัติ

logging:
  level:
    com.example: DEBUG
```

```yaml
# application-prod.yml - Production
spring:
  datasource:
    url: ${DATABASE_URL}
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
    hikari:
      maximum-pool-size: 50
      minimum-idle: 10
      connection-timeout: 20000
      idle-timeout: 300000
      max-lifetime: 1200000

  jpa:
    hibernate:
      ddl-auto: validate  # ห้ามเปลี่ยน schema อัตโนมัติ

server:
  ssl:
    enabled: true
    key-store: ${SSL_KEYSTORE_PATH}
    key-store-password: ${SSL_KEYSTORE_PASSWORD}

management:
  endpoint:
    health:
      show-details: never  # ไม่แสดง detail ใน prod

logging:
  level:
    root: WARN
    com.example: INFO
```

### Kubernetes ConfigMap Integration

```yaml
# k8s/configmap.yml
apiVersion: v1
kind: ConfigMap
metadata:
  name: order-service-config
  namespace: production
data:
  application.yml: |
    spring:
      datasource:
        url: jdbc:postgresql://postgres-service:5432/orders
      redis:
        host: redis-service
    app:
      order:
        max-items: 100

---
# k8s/secret.yml
apiVersion: v1
kind: Secret
metadata:
  name: order-service-secrets
  namespace: production
type: Opaque
stringData:
  DB_PASSWORD: "production-secret"
  JWT_SECRET: "jwt-secret-key"
  STRIPE_API_KEY: "sk_live_xxxx"

---
# k8s/deployment.yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  template:
    spec:
      containers:
        - name: order-service
          image: order-service:latest
          env:
            - name: SPRING_PROFILES_ACTIVE
              value: "prod"
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: order-service-secrets
                  key: DB_PASSWORD
            - name: JWT_SECRET
              valueFrom:
                secretKeyRef:
                  name: order-service-secrets
                  key: JWT_SECRET
          volumeMounts:
            - name: config-volume
              mountPath: /config
      volumes:
        - name: config-volume
          configMap:
            name: order-service-config
```

---

## ขั้นตอนที่ 2355-2360: Feature Toggles ผ่าน Configuration

### Feature Toggle Pattern

```java
// FeatureToggleConfig.java
@ConfigurationProperties(prefix = "app.features")
@RefreshScope  // ทำให้ toggle เปลี่ยนได้ runtime
@Data
public class FeatureToggleConfig {

    // Feature flags
    private boolean newCheckoutFlow = false;
    private boolean loyaltyPointsEnabled = false;
    private boolean expressDeliveryEnabled = true;
    private boolean aiRecommendationsEnabled = false;

    // Canary release: เปิดใช้เฉพาะ % ของ users
    private double aiRecommendationsRolloutPercentage = 0.0;

    // A/B Testing
    private String checkoutFlowVariant = "A";
}

// FeatureToggleService.java
@Service
@RefreshScope
public class FeatureToggleService {

    private final FeatureToggleConfig config;

    public FeatureToggleService(FeatureToggleConfig config) {
        this.config = config;
    }

    public boolean isEnabled(String featureName) {
        return switch (featureName) {
            case "new-checkout" -> config.isNewCheckoutFlow();
            case "loyalty-points" -> config.isLoyaltyPointsEnabled();
            case "express-delivery" -> config.isExpressDeliveryEnabled();
            case "ai-recommendations" -> config.isAiRecommendationsEnabled();
            default -> false;
        };
    }

    // Canary Release: เปิดใช้สำหรับ % ของ users
    public boolean isEnabledForUser(String featureName, String userId) {
        if (!isEnabled(featureName)) {
            return false;
        }

        if ("ai-recommendations".equals(featureName)) {
            // Hash userId เพื่อให้ user เดิมได้ผลเหมือนกันทุกครั้ง
            int hash = Math.abs(userId.hashCode()) % 100;
            return hash < (config.getAiRecommendationsRolloutPercentage() * 100);
        }

        return true;
    }
}
```

### การใช้ Feature Toggles ใน Controller

```java
// OrderController.java
@RestController
@RequestMapping("/api/orders")
public class OrderController {

    private final OrderService orderService;
    private final FeatureToggleService featureToggleService;
    private final NewCheckoutService newCheckoutService;

    @PostMapping("/checkout")
    public ResponseEntity<CheckoutResponse> checkout(
        @RequestBody CheckoutRequest request,
        @AuthenticationPrincipal UserDetails userDetails
    ) {
        String userId = userDetails.getUsername();

        // ใช้ new checkout flow ถ้า feature เปิด
        if (featureToggleService.isEnabledForUser("new-checkout", userId)) {
            log.info("Using new checkout flow for user {}", userId);
            return ResponseEntity.ok(newCheckoutService.process(request));
        }

        // Fallback ไปยัง old flow
        return ResponseEntity.ok(orderService.processCheckout(request));
    }

    @GetMapping("/recommendations")
    public ResponseEntity<List<ProductRecommendation>> getRecommendations(
        @AuthenticationPrincipal UserDetails userDetails
    ) {
        String userId = userDetails.getUsername();

        if (!featureToggleService.isEnabledForUser("ai-recommendations", userId)) {
            // Feature ยังไม่เปิด - return empty list
            return ResponseEntity.ok(Collections.emptyList());
        }

        return ResponseEntity.ok(aiRecommendationService.getRecommendations(userId));
    }
}
```

### Feature Toggle ใน Config File

```yaml
# application.yml
app:
  features:
    new-checkout-flow: false      # ยังไม่เปิด
    loyalty-points-enabled: true   # เปิดแล้ว
    express-delivery-enabled: true
    ai-recommendations-enabled: true
    ai-recommendations-rollout-percentage: 0.25  # เปิดแค่ 25%
    checkout-flow-variant: "B"

# Runtime change ผ่าน Config Server
# เปลี่ยนไฟล์ใน Git repo แล้ว refresh
# curl -X POST http://order-service/actuator/refresh
```

### Testing Feature Toggles

```java
// FeatureToggleTest.java
@SpringBootTest
class FeatureToggleTest {

    @Autowired
    private FeatureToggleService featureToggleService;

    @Test
    @DisplayName("Feature toggle ที่ปิดควร return false")
    void disabledFeature_ShouldReturnFalse() {
        assertThat(featureToggleService.isEnabled("new-checkout")).isFalse();
    }

    @Test
    @DisplayName("Canary release ควร distribute traffic ตาม percentage")
    void canaryRelease_ShouldDistributeTrafficCorrectly() {
        // สร้าง users จำนวนมากเพื่อตรวจสอบ distribution
        long enabledCount = IntStream.range(0, 1000)
            .mapToObj(i -> "user-" + i)
            .filter(userId -> featureToggleService.isEnabledForUser("ai-recommendations", userId))
            .count();

        // ควรได้ประมาณ 25% (±5% tolerance)
        assertThat(enabledCount).isBetween(200L, 300L);
    }

    @Test
    @DisplayName("Feature ที่ปิดอยู่ไม่ควรเปิดสำหรับ user ใดๆ")
    void disabledFeature_ShouldNotBeEnabledForAnyUser() {
        IntStream.range(0, 100)
            .mapToObj(i -> "user-" + i)
            .forEach(userId ->
                assertThat(featureToggleService.isEnabledForUser("new-checkout", userId))
                    .isFalse()
            );
    }
}
```

---

## สรุป Part 68: Configuration Management

### Config Management Checklist

- [ ] ใช้ Spring Cloud Config Server สำหรับ centralized config
- [ ] เก็บ config ใน Git (version control + audit trail)
- [ ] Secrets อยู่ใน Vault ไม่ใช่ Git
- [ ] ใช้ @RefreshScope สำหรับ config ที่ต้อง refresh ได้
- [ ] มี validation สำหรับ config properties
- [ ] แยก config ตาม environment อย่างชัดเจน
- [ ] ใช้ Feature Toggles สำหรับ gradual rollout
- [ ] Config Server ต้องมี HA setup ใน production

### Anti-patterns

```yaml
# ❌ BAD: Hardcode credentials ใน application.yml
spring:
  datasource:
    password: my-production-password

# ❌ BAD: ใช้ credentials เหมือนกันทุก environment
spring:
  datasource:
    url: jdbc:postgresql://prod-db:5432/orders

# ✅ GOOD: ใช้ Environment Variables หรือ Vault
spring:
  datasource:
    url: ${DATABASE_URL}
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
```

---

*[← Part 67: Logging Best Practices](./part-67-logging-best-practices.md) | [Part 69: Database Advanced →](./part-69-database-advanced.md)*

# Part 08: Application Properties และ Profiles
## ขั้นตอนที่ 156-180

> **ระดับ:** พื้นฐาน (Beginner)  
> **เวลาเรียน:** 2-3 ชั่วโมง  
> **เป้าหมาย:** จัดการ Configuration อย่างมืออาชีพ สำหรับหลาย Environment

---

## ขั้นตอนที่ 156: application.properties vs application.yml

```properties
# application.properties - traditional style
spring.application.name=myapp
spring.datasource.url=jdbc:postgresql://localhost:5432/mydb
spring.datasource.username=admin
spring.jpa.hibernate.ddl-auto=update
server.port=8080
```

```yaml
# application.yml - YAML style (อ่านง่ายกว่า)
spring:
  application:
    name: myapp
  datasource:
    url: jdbc:postgresql://localhost:5432/mydb
    username: admin
  jpa:
    hibernate:
      ddl-auto: update

server:
  port: 8080
```

### เปรียบเทียบ

```
Properties              YAML
─────────────────────────────────────────
  ง่ายกว่า              อ่านง่ายกว่า (hierarchical)
  ทุก IDE support       บาง IDE ต้องการ plugin
  ไม่รองรับ lists       รองรับ lists ได้ดี
  copy-paste ง่าย       ระวัง indentation
  ไม่ sensitive space   Sensitive ต่อ spaces
```

---

## ขั้นตอนที่ 157: YAML Syntax ที่ควรรู้

```yaml
# Basic types
string-value: hello world
quoted-string: "Hello World"   # ใช้เมื่อมี special characters
number: 42
float: 3.14
boolean-true: true
boolean-false: false
null-value: null               # หรือ ~
empty-string: ""

# Multi-line strings
description: |
  This is a multi-line string
  that preserves newlines

folded-description: >
  This is a folded string
  where newlines become spaces

# Lists
fruits:
  - apple
  - banana
  - cherry

# Inline list
colors: [red, green, blue]

# Maps
database:
  host: localhost
  port: 5432
  name: mydb

# Inline map
person: {name: John, age: 30}

# Nested
server:
  cors:
    allowed-origins:
      - http://localhost:3000
      - https://myapp.com
    allowed-methods:
      - GET
      - POST
      - PUT
      - DELETE

# References (anchors)
defaults: &defaults
  timeout: 30
  retry: 3

production:
  <<: *defaults    # Merge defaults
  timeout: 60      # Override

# Multiple documents in one file (Spring Profiles)
spring:
  application:
    name: myapp
---
spring:
  config:
    activate:
      on-profile: dev
  datasource:
    url: jdbc:h2:mem:devdb
---
spring:
  config:
    activate:
      on-profile: prod
  datasource:
    url: jdbc:postgresql://prod-host:5432/mydb
```

---

## ขั้นตอนที่ 158: @ConfigurationProperties

```java
// ดีกว่าการใช้ @Value หลายๆ ตัว
@Component
@ConfigurationProperties(prefix = "app")
@Data
@Validated  // ← Enable validation บน properties
public class AppProperties {
    
    @NotBlank
    private String name;
    
    @NotBlank  
    private String version;
    
    @Valid  // ← Validate nested objects
    private Security security = new Security();
    
    @Valid
    private Database database = new Database();
    
    private List<String> allowedOrigins = new ArrayList<>();
    
    private Map<String, String> features = new HashMap<>();
    
    @Data
    public static class Security {
        
        @NotBlank
        private String jwtSecret;
        
        @Min(1)
        private long jwtExpirationMs = 86400000L;  // 24 hours
        
        @Min(1)
        private long refreshExpirationMs = 604800000L;  // 7 days
    }
    
    @Data
    public static class Database {
        
        @NotBlank
        private String url;
        
        @NotBlank
        private String username;
        
        private String password;
        
        @Min(1)
        @Max(100)
        private int maxPoolSize = 10;
        
        @Min(0)
        private int minIdle = 2;
    }
}
```

```yaml
# application.yml
app:
  name: My Application
  version: 1.0.0
  
  security:
    jwt-secret: mySecretKey123456789012345678901234
    jwt-expiration-ms: 86400000
    refresh-expiration-ms: 604800000
  
  database:
    url: jdbc:postgresql://localhost:5432/mydb
    username: admin
    password: secret
    max-pool-size: 10
    min-idle: 2
  
  allowed-origins:
    - http://localhost:3000
    - https://myapp.com
  
  features:
    payment: enabled
    notifications: enabled
    analytics: disabled
```

---

## ขั้นตอนที่ 159: Profiles

Profiles ช่วยให้ config ต่างกันสำหรับแต่ละ environment

```
Environments:
  dev   → localhost, H2 database, verbose logging
  test  → CI/CD, in-memory database
  staging → production-like แต่ข้อมูลน้อยกว่า
  prod  → Production, real database, minimal logging
```

### สร้าง Profile-specific configurations

```
src/main/resources/
├── application.yml              ← Shared config
├── application-dev.yml          ← Dev config
├── application-test.yml         ← Test config
├── application-staging.yml      ← Staging config
└── application-prod.yml         ← Production config
```

```yaml
# application.yml (shared)
spring:
  application:
    name: myapp
  profiles:
    active: ${SPRING_PROFILES_ACTIVE:dev}  # default = dev

app:
  name: My Application
  version: 1.0.0

management:
  endpoints:
    web:
      exposure:
        include: health,info
```

```yaml
# application-dev.yml
spring:
  datasource:
    url: jdbc:h2:mem:devdb;DB_CLOSE_DELAY=-1
    driver-class-name: org.h2.Driver
    username: sa
    password: ""
  
  h2:
    console:
      enabled: true    # เปิด H2 Console ที่ /h2-console
  
  jpa:
    show-sql: true
    hibernate:
      ddl-auto: create-drop
    database-platform: org.hibernate.dialect.H2Dialect
  
  devtools:
    restart:
      enabled: true

logging:
  level:
    com.example: DEBUG
    org.springframework.web: DEBUG
    org.hibernate.SQL: DEBUG
    org.hibernate.type.descriptor.sql: TRACE

management:
  endpoints:
    web:
      exposure:
        include: "*"    # เปิดทุก endpoints ใน dev
```

```yaml
# application-test.yml
spring:
  datasource:
    url: jdbc:h2:mem:testdb;DB_CLOSE_DELAY=-1
    driver-class-name: org.h2.Driver
    username: sa
    password: ""
  
  jpa:
    hibernate:
      ddl-auto: create-drop
    show-sql: false

logging:
  level:
    root: WARN
    com.example: INFO
```

```yaml
# application-prod.yml
spring:
  datasource:
    url: ${DATABASE_URL}              # จาก environment variable
    username: ${DATABASE_USERNAME}
    password: ${DATABASE_PASSWORD}
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
      connection-timeout: 30000
  
  jpa:
    show-sql: false
    hibernate:
      ddl-auto: validate    # ไม่เปลี่ยน schema!
  
  devtools:
    restart:
      enabled: false

logging:
  level:
    root: WARN
    com.example: INFO

management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics  # จำกัด endpoints ใน prod
  endpoint:
    health:
      show-details: when-authorized   # ซ่อน details จาก public
```

---

## ขั้นตอนที่ 160: Profile Activation

```bash
# วิธีที่ 1: application.properties/yml
spring.profiles.active=dev

# วิธีที่ 2: Command line argument
java -jar app.jar --spring.profiles.active=prod

# วิธีที่ 3: JVM System Property
java -Dspring.profiles.active=prod -jar app.jar

# วิธีที่ 4: Environment Variable
export SPRING_PROFILES_ACTIVE=prod
java -jar app.jar

# วิธีที่ 5: Maven (สำหรับ mvn spring-boot:run)
mvn spring-boot:run -Dspring-boot.run.profiles=dev

# วิธีที่ 6: Programmatically
SpringApplication app = new SpringApplication(MyApp.class);
app.setAdditionalProfiles("dev", "local");
app.run(args);
```

### Activate หลาย Profiles พร้อมกัน

```bash
# หลาย profiles (คั่นด้วย comma)
spring.profiles.active=prod,metrics,featureX

# หรือ
--spring.profiles.active=prod,metrics,featureX
```

---

## ขั้นตอนที่ 161: @Profile Annotation

```java
// Component/Service/Repository สำหรับ profile เฉพาะ
@Service
@Profile("dev")
public class MockPaymentService implements PaymentService {
    @Override
    public PaymentResult process(PaymentRequest request) {
        // ไม่ charge เงินจริง ใช้แค่ใน dev
        return PaymentResult.success("mock-transaction-id");
    }
}

@Service
@Profile("prod")
public class StripePaymentService implements PaymentService {
    @Override
    public PaymentResult process(PaymentRequest request) {
        // Stripe real payment
        return stripeClient.charge(request);
    }
}

// หลาย profiles
@Component
@Profile({"dev", "test"})  // OR logic
public class DevTestComponent { }

// NOT profile
@Component
@Profile("!prod")  // ทุก profile ยกเว้น prod
public class NonProdComponent { }

// Complex expression (Spring 5.1+)
@Component
@Profile("cloud & (aws | gcp)")  // cloud AND (aws OR gcp)
public class CloudComponent { }
```

---

## ขั้นตอนที่ 162: Environment Abstraction

```java
@Component
@RequiredArgsConstructor
public class AppStartupInfo {
    
    private final Environment environment;
    
    @PostConstruct
    public void printInfo() {
        System.out.println("Active profiles: " + 
            Arrays.toString(environment.getActiveProfiles()));
        
        System.out.println("Default profiles: " + 
            Arrays.toString(environment.getDefaultProfiles()));
        
        System.out.println("Database URL: " + 
            environment.getProperty("spring.datasource.url", "NOT SET"));
        
        System.out.println("Is dev: " + 
            environment.acceptsProfiles(Profiles.of("dev")));
        
        // ดู property จาก environment variables
        System.out.println("JAVA_HOME: " + 
            environment.getProperty("JAVA_HOME"));
    }
}
```

---

## ขั้นตอนที่ 163: External Configuration

```
ลำดับการโหลด application.properties/yml:

1. /config/application.yml       (ใน directory ที่รัน jar)
2. application.yml               (ใน directory ที่รัน jar)
3. classpath:/config/application.yml   (ใน jar)
4. classpath:/application.yml         (ใน jar)

เหมาะสำหรับ: ไม่ต้องแก้ jar เมื่อ config เปลี่ยน
```

```bash
# ตัวอย่าง directory structure สำหรับ production:
/app/
├── myapp.jar
├── config/
│   └── application.yml          ← External config (override)
└── logs/
```

```bash
# หรือระบุ config location ชัดเจน
java -jar myapp.jar \
  --spring.config.location=file:/etc/myapp/application.yml

# หลายที่
java -jar myapp.jar \
  --spring.config.location=\
    classpath:/default.yml,\
    file:/etc/myapp/override.yml

# เพิ่ม config location โดยไม่ replace default
java -jar myapp.jar \
  --spring.config.additional-location=file:/etc/myapp/extra.yml
```

---

## ขั้นตอนที่ 164: Spring Cloud Config Server (Preview)

```yaml
# ใน production ใช้ Config Server แทน local files

# bootstrap.yml (โหลดก่อน application.yml)
spring:
  cloud:
    config:
      uri: http://config-server:8888
      label: main        # git branch
      profile: prod      # environment
```

```
Config Server จะดึง config จาก:
  - Git repository
  - File system
  - Vault
  - Database

ทุก application ดึง config จาก server เดียว → เปลี่ยน config ง่าย
```

---

## ขั้นตอนที่ 165: Property Encryption

```yaml
# ไม่ควร store plaintext passwords!

# ❌ แบบไม่ดี:
spring:
  datasource:
    password: myPlainTextPassword

# ✅ แบบดี 1: Environment Variables
spring:
  datasource:
    password: ${DATABASE_PASSWORD}   # จาก OS env var

# ✅ แบบดี 2: Jasypt Encryption
spring:
  datasource:
    password: ENC(encryptedPasswordHere)  # ต้องใช้ Jasypt library

# ✅ แบบดี 3: HashiCorp Vault (Enterprise)
spring:
  cloud:
    vault:
      uri: https://vault:8200
      token: ${VAULT_TOKEN}
```

```java
// ใช้ Jasypt สำหรับ encrypt properties
// pom.xml:
// com.github.ulisesbocchio:jasypt-spring-boot-starter:3.0.5

@SpringBootApplication
@EnableEncryptableProperties  // Jasypt
public class MyApp { }
```

```bash
# Encrypt password ด้วย Jasypt CLI
java -cp jasypt-1.9.3.jar \
  org.jasypt.intf.cli.JasyptPBEStringEncryptionCLI \
  input="mySecret" \
  password="masterKey" \
  algorithm=PBEWithMD5AndDES

# Output: e.g., 7FLYdFBhgFcRZ3wl19Sezw==

# ใส่ใน properties:
# spring.datasource.password=ENC(7FLYdFBhgFcRZ3wl19Sezw==)

# รัน app:
# java -Djasypt.encryptor.password=masterKey -jar app.jar
```

---

## ขั้นตอนที่ 166: Secrets Management

```java
// .env file (ไม่ควร commit!)
// .env
DATABASE_URL=jdbc:postgresql://localhost:5432/mydb
DATABASE_USERNAME=admin
DATABASE_PASSWORD=secret
JWT_SECRET=myJwtSecretKey123456789012345678901234

// Load .env file (ใช้ dotenv-java library)
// pom.xml: io.github.cdimascio:dotenv-java:3.0.0

Dotenv dotenv = Dotenv.configure()
    .ignoreIfMissing()  // ไม่ error ถ้าไม่มีไฟล์ (production ใช้ real env vars)
    .load();

// หรือใช้ Spring Boot DevTools plugin ที่รองรับ .env
```

---

## ขั้นตอนที่ 167: Conditional Properties

```yaml
# Spring Boot 2.4+ รองรับ conditional includes
spring:
  config:
    import:
      - optional:file:./config/local.properties  # โหลดถ้ามี
      - optional:classpath:extra.yml             # โหลดถ้ามี
      - vault://              # Spring Cloud Vault
```

```java
// @ConditionalOnProperty - สร้าง Bean เมื่อ property มีค่าที่กำหนด
@Configuration
public class FeatureConfig {
    
    @Bean
    @ConditionalOnProperty(name = "feature.ai.enabled", havingValue = "true")
    public AiService aiService() {
        return new AiServiceImpl();
    }
    
    @Bean
    @ConditionalOnProperty(
        name = "app.cache.type",
        havingValue = "redis"
    )
    public CacheService redisCacheService() {
        return new RedisCacheService();
    }
}
```

---

## ขั้นตอนที่ 168: Dynamic Property Refresh

```yaml
# ใช้ Spring Cloud Config + @RefreshScope
# แก้ properties ใน config server แล้วเรียก /actuator/refresh
# Bean จะ re-initialized

management:
  endpoints:
    web:
      exposure:
        include: refresh
```

```java
@RestController
@RefreshScope  // ← Bean จะ re-created เมื่อ properties เปลี่ยน
@RequiredArgsConstructor
public class ConfigController {
    
    @Value("${app.feature.new-ui}")
    private boolean newUiEnabled;
    
    @GetMapping("/config")
    public Map<String, Object> getConfig() {
        return Map.of("newUiEnabled", newUiEnabled);
    }
}

// POST /actuator/refresh → bean ถูก refresh → newUiEnabled อัปเดต
```

---

## ขั้นตอนที่ 169: Profile Groups (Spring Boot 2.4+)

```yaml
# สร้าง profile groups
spring:
  profiles:
    group:
      production:         # profile "production" = รวมหลาย profiles
        - prod
        - metrics
        - monitoring
      local:              # profile "local" = dev + fake services
        - dev
        - fake-payment
        - fake-email
```

```bash
# activate ทีเดียว
spring.profiles.active=production

# เท่ากับ:
spring.profiles.active=prod,metrics,monitoring
```

---

## ขั้นตอนที่ 170: Testing Configuration

```java
// ทดสอบด้วย specific properties
@SpringBootTest(properties = {
    "spring.datasource.url=jdbc:h2:mem:testdb",
    "app.feature.enabled=true"
})
class MyTest { }

// หรือใช้ @TestPropertySource
@SpringBootTest
@TestPropertySource(
    locations = "classpath:application-test.yml",
    properties = {
        "app.timeout=5",
        "feature.enabled=false"
    }
)
class MyTest { }

// ใช้ @ActiveProfiles
@SpringBootTest
@ActiveProfiles("test")
class IntegrationTest { }

// ทดสอบ @ConfigurationProperties
@SpringBootTest
class AppPropertiesTest {
    
    @Autowired
    private AppProperties appProperties;
    
    @Test
    void properties_ShouldBeLoaded() {
        assertEquals("myapp", appProperties.getName());
        assertNotNull(appProperties.getSecurity().getJwtSecret());
    }
}
```

---

## ขั้นตอนที่ 171: Property Validation

```java
@Component
@ConfigurationProperties(prefix = "app")
@Validated  // ← จำเป็น!
@Data
public class AppProperties {
    
    @NotBlank(message = "App name must not be blank")
    private String name;
    
    @Valid
    @NotNull
    private Security security;
    
    @Data
    public static class Security {
        
        @NotBlank(message = "JWT secret must not be blank")
        @Size(min = 32, message = "JWT secret must be at least 32 characters")
        private String jwtSecret;
        
        @Min(value = 1000, message = "JWT expiration must be at least 1 second (1000ms)")
        private long jwtExpirationMs;
    }
}
```

```
ถ้า properties ไม่ valid ที่ startup:
  APPLICATION FAILED TO START
  ***************************
  Description:
  Binding to target ... failed
  Property: app.security.jwt-secret
  Value: ""
  Reason: JWT secret must not be blank
```

---

## ขั้นตอนที่ 172: Complex Property Types

```java
@ConfigurationProperties(prefix = "app")
@Data
public class AppProperties {
    
    // Duration
    private Duration sessionTimeout = Duration.ofMinutes(30);
    
    // DataSize
    private DataSize maxUploadSize = DataSize.ofMegabytes(10);
    
    // Period
    private Period dataRetentionPeriod = Period.ofDays(90);
    
    // Charset
    private Charset encoding = StandardCharsets.UTF_8;
    
    // Resource
    private Resource configFile;
    
    // Pattern
    private Pattern emailPattern = Pattern.compile(".*@.*\\..*");
}
```

```yaml
app:
  session-timeout: 30m           # Duration: 30 minutes
  max-upload-size: 10MB          # DataSize: 10 megabytes
  data-retention-period: P90D    # Period: 90 days (ISO-8601)
  encoding: UTF-8
  config-file: classpath:config.json
  email-pattern: ".*@.*\\..*"
```

---

## ขั้นตอนที่ 173: Properties ที่ควรรู้จัก

```yaml
# สรุป Spring Boot properties ที่ใช้บ่อย

spring:
  application:
    name: myapp                    # ชื่อ app
  
  profiles:
    active: dev                    # active profile
  
  # Database
  datasource:
    url: jdbc:postgresql://...
    username: user
    password: pass
    hikari:
      maximum-pool-size: 10
  
  # JPA
  jpa:
    hibernate:
      ddl-auto: update
    show-sql: true
    open-in-view: false           # ปิดเพื่อ performance
  
  # Security
  security:
    user:
      name: admin
      password: admin
  
  # Cache
  cache:
    type: redis
  
  # Mail
  mail:
    host: smtp.gmail.com
    port: 587
    username: user@gmail.com
    password: apppassword
    properties:
      mail:
        smtp:
          auth: true
          starttls:
            enable: true
  
  # Web
  web:
    resources:
      static-locations: classpath:/static/
  
  # Multipart
  servlet:
    multipart:
      max-file-size: 10MB
      max-request-size: 10MB
  
  # Thymeleaf
  thymeleaf:
    cache: false                  # ปิด cache ใน dev
  
  # Actuator
  management:
    server:
      port: 8081                  # แยก port
    endpoints:
      web:
        exposure:
          include: health,info
  
  # Logging
  logging:
    level:
      root: INFO
      com.example: DEBUG
    file:
      name: logs/app.log
  
  # Server
  server:
    port: 8080
    servlet:
      context-path: /api
    compression:
      enabled: true
```

---

## ขั้นตอนที่ 174: Secrets จาก AWS Parameter Store

```yaml
# ใช้ Spring Cloud AWS
spring:
  cloud:
    aws:
      ssm:
        enabled: true
  config:
    import: "aws-parameterstore:"
```

```java
// Parameter Store path: /myapp/prod/database-url
@Value("${/myapp/prod/database-url}")
private String databaseUrl;

// หรือผ่าน ConfigurationProperties
@ConfigurationProperties(prefix = "/myapp/prod")
@Data
public class AwsProperties {
    private String databaseUrl;
    private String jwtSecret;
}
```

---

## ขั้นตอนที่ 175-180: สรุปและแบบฝึกหัด

### สรุป Part 08

```
✅ application.properties vs application.yml
✅ YAML Syntax
✅ @ConfigurationProperties
✅ Spring Profiles
✅ Profile activation (หลายวิธี)
✅ @Profile annotation
✅ Environment abstraction
✅ External configuration
✅ Property encryption (Jasypt)
✅ Dynamic property refresh (@RefreshScope)
✅ Profile groups
✅ Testing configuration
✅ Property validation
✅ Complex property types
✅ AWS Parameter Store integration
```

### Configuration Best Practices

```
✅ ใช้ .yml แทน .properties (อ่านง่ายกว่า)
✅ ใช้ @ConfigurationProperties แทน @Value หลายๆ ตัว
✅ ไม่ hardcode sensitive values (ใช้ env vars)
✅ Validate configuration ด้วย @Validated
✅ ใช้ Profile groups สำหรับ complex environments
✅ แยก configuration ตาม environment
✅ ใช้ Spring Cloud Config Server ใน production
✅ Encrypt sensitive properties

❌ ห้าม commit passwords/secrets ใน git
❌ ห้าม show-sql=true ใน production
❌ ห้าม ddl-auto=update ใน production (ใช้ validate)
❌ ห้าม expose actuator endpoints ทั้งหมดใน production
```

### แบบฝึกหัด

```
1. สร้าง application configuration สำหรับ 3 environments:
   - dev: H2 database, verbose logging, all actuator endpoints
   - test: H2 in-memory, minimal logging
   - prod: PostgreSQL, WARNING logging only, limited actuator

2. สร้าง @ConfigurationProperties สำหรับ:
   - Email settings (SMTP)
   - JWT settings (secret, expiration)
   - Upload settings (max size, allowed types, directory)
   พร้อม validation

3. ทดสอบว่า properties โหลดถูกต้องด้วย @SpringBootTest
```

---

*[← Part 07: Auto-configuration](./part-07-auto-configuration.md) | [Part 09: Logging →](./part-09-logging.md)*

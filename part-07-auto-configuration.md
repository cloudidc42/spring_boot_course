# Part 07: Spring Boot Auto-configuration
## ขั้นตอนที่ 136-155

> **ระดับ:** พื้นฐาน (Beginner)  
> **เวลาเรียน:** 2-3 ชั่วโมง  
> **เป้าหมาย:** เข้าใจ Auto-configuration กลไก และสร้าง Custom Auto-configuration

---

## ขั้นตอนที่ 136: Auto-configuration คืออะไร?

Auto-configuration คือกลไกที่ Spring Boot ใช้ config application อัตโนมัติตาม dependencies ที่มีใน classpath

```
เพิ่ม spring-boot-starter-web ใน pom.xml
        │
        ▼
Spring Boot ตรวจสอบ classpath
        │
        ▼
พบ: spring-webmvc, embedded tomcat, jackson
        │
        ▼
Auto-configure:
  ✓ EmbeddedTomcat (port 8080)
  ✓ DispatcherServlet
  ✓ Jackson ObjectMapper
  ✓ Error handling
  ✓ Static resources
  ✓ ViewResolver
        │
        ▼
Application ready!
```

---

## ขั้นตอนที่ 137: @EnableAutoConfiguration

```java
// @SpringBootApplication รวม 3 annotations
@SpringBootApplication
// = 
@SpringBootConfiguration    // เป็น @Configuration class
@EnableAutoConfiguration    // ← Auto-configuration!
@ComponentScan              // ← scan beans

// @EnableAutoConfiguration บอกให้ Spring Boot หา auto-configuration classes
// ที่ลงทะเบียนไว้ใน:
// - Spring Boot 2.x: META-INF/spring.factories
// - Spring Boot 3.x: META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
```

---

## ขั้นตอนที่ 138: ดู Auto-configuration ที่ทำงาน

```bash
# วิธีที่ 1: รันด้วย --debug flag
java -jar myapp.jar --debug

# Output จะมี section:
# ============================
# CONDITIONS EVALUATION REPORT
# ============================
# Positive matches (auto-configured):
#   DataSourceAutoConfiguration matched:
#     - @ConditionalOnClass found required class 'javax.sql.DataSource'
#     - @ConditionalOnMissingBean not found ...
# 
# Negative matches (not auto-configured):
#   MongoAutoConfiguration:
#     Did not match:
#       - @ConditionalOnClass did not find required class 'com.mongodb.MongoClient'

# วิธีที่ 2: actuator/conditions endpoint
curl http://localhost:8080/actuator/conditions

# วิธีที่ 3: Logging
logging.level.org.springframework.boot.autoconfigure=DEBUG
```

---

## ขั้นตอนที่ 139: @ConditionalOnXxx Annotations

```java
// Spring Boot ใช้ @Conditional annotations เพื่อตัดสินใจ auto-configure หรือไม่

@Configuration
@ConditionalOnClass(DataSource.class)        // มี DataSource class ใน classpath
@ConditionalOnMissingBean(DataSource.class)  // ไม่มี DataSource bean อยู่แล้ว
@ConditionalOnProperty(
    prefix = "spring.datasource",
    name = "url"                             // property นี้ถูกตั้งค่า
)
public class DataSourceAutoConfiguration {
    
    @Bean
    @ConditionalOnMissingBean
    public DataSource dataSource(DataSourceProperties properties) {
        return DataSourceBuilder.create()
            .url(properties.getUrl())
            .username(properties.getUsername())
            .password(properties.getPassword())
            .build();
    }
}
```

### ตัวอย่าง @Conditional annotations ทั้งหมด

```java
// Class-level
@ConditionalOnClass(name = "com.example.SomeClass")   // มี class นี้
@ConditionalOnMissingClass("com.example.SomeClass")   // ไม่มี class นี้

// Bean-level
@ConditionalOnBean(type = "DataSource")               // มี bean ประเภทนี้
@ConditionalOnMissingBean                             // ไม่มี bean นี้ (by type)

// Property-level
@ConditionalOnProperty(name = "feature.enabled", havingValue = "true")
@ConditionalOnProperty(name = "feature.enabled", matchIfMissing = true)

// Resource-level
@ConditionalOnResource(resources = "classpath:schema.sql")  // มี resource นี้

// Web-level
@ConditionalOnWebApplication                          // เป็น web application
@ConditionalOnNotWebApplication                       // ไม่ใช่ web application
@ConditionalOnWebApplication(type = Type.SERVLET)     // Servlet web app
@ConditionalOnWebApplication(type = Type.REACTIVE)    // Reactive web app

// Expression-level
@ConditionalOnExpression("${feature.enabled} && ${another.feature}")
```

---

## ขั้นตอนที่ 140: สร้าง Custom Auto-configuration

```
กรณีที่ต้องสร้าง Custom Auto-configuration:
1. สร้าง Library ที่ต้องการ Spring integration
2. สร้าง Starter ขึ้นมาเอง
3. Company internal library ที่ใช้ร่วมกันหลาย projects
```

### ตัวอย่าง: สร้าง SMS Service Library

```
my-sms-starter/
├── pom.xml
└── src/main/java/com/example/sms/
    ├── SmsProperties.java           ← Configuration properties
    ├── SmsService.java              ← The service
    ├── SmsAutoConfiguration.java    ← Auto-configuration
    └── SmsTemplate.java             ← Main template class
```

```java
// 1. Properties class
// com/example/sms/SmsProperties.java
@ConfigurationProperties(prefix = "sms")
@Data
public class SmsProperties {
    
    private boolean enabled = true;
    private String apiKey;
    private String apiSecret;
    private String provider = "twilio";
    private String defaultSender;
    private int timeout = 30;
}
```

```java
// 2. Service interface
// com/example/sms/SmsService.java
public interface SmsService {
    void send(String to, String message);
    void sendBatch(List<String> numbers, String message);
}
```

```java
// 3. Implementation
// com/example/sms/TwilioSmsService.java
@RequiredArgsConstructor
@Slf4j
public class TwilioSmsService implements SmsService {
    
    private final SmsProperties properties;
    
    @Override
    public void send(String to, String message) {
        log.info("Sending SMS to {} via Twilio", to);
        // Twilio SDK call
    }
    
    @Override
    public void sendBatch(List<String> numbers, String message) {
        numbers.forEach(number -> send(number, message));
    }
}
```

```java
// 4. Auto-configuration class
// com/example/sms/SmsAutoConfiguration.java
@Configuration
@EnableConfigurationProperties(SmsProperties.class)  // ← เปิดใช้ Properties
@ConditionalOnClass(SmsService.class)                 // ← มี SmsService ใน classpath
@ConditionalOnProperty(
    prefix = "sms",
    name = "enabled",
    havingValue = "true",
    matchIfMissing = true                             // ← default = enable
)
@AutoConfiguration  // Spring Boot 3.x (แทน @Configuration)
public class SmsAutoConfiguration {
    
    @Bean
    @ConditionalOnMissingBean(SmsService.class)       // ← ไม่สร้างถ้ามี custom bean แล้ว
    @ConditionalOnProperty(prefix = "sms", name = "api-key")
    public SmsService twilioSmsService(SmsProperties properties) {
        return new TwilioSmsService(properties);
    }
    
    @Bean
    @ConditionalOnBean(SmsService.class)
    public SmsTemplate smsTemplate(SmsService smsService, SmsProperties properties) {
        return new SmsTemplate(smsService, properties);
    }
}
```

```
# 5. ลงทะเบียน Auto-configuration
# src/main/resources/META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
com.example.sms.SmsAutoConfiguration
```

---

## ขั้นตอนที่ 141: Override Auto-configuration

```java
// Spring Boot จะไม่ auto-configure ถ้าเราสร้าง bean เองแล้ว

// 1. สร้าง custom DataSource → Spring Boot จะไม่ auto-configure DataSource
@Bean
public DataSource customDataSource() {
    HikariConfig config = new HikariConfig();
    config.setJdbcUrl("jdbc:postgresql://...");
    return new HikariDataSource(config);
}

// 2. Exclude specific auto-configuration
@SpringBootApplication(exclude = {
    SecurityAutoConfiguration.class,         // ปิด Security
    DataSourceAutoConfiguration.class,       // ปิด DataSource
    HibernateJpaAutoConfiguration.class      // ปิด JPA
})
public class MyApp { }

// หรือผ่าน properties
// spring.autoconfigure.exclude=
//   org.springframework.boot.autoconfigure.security.servlet.SecurityAutoConfiguration,
//   org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration
```

---

## ขั้นตอนที่ 142: DataSource Auto-configuration

```
เพิ่ม dependency: spring-boot-starter-data-jpa + postgresql driver
        │
        ▼
Spring Boot auto-detects:
  - DataSource class available
  - spring.datasource.url property set
        │
        ▼
Auto-configures:
  ✓ HikariCP DataSource (default pool)
  ✓ JPA EntityManagerFactory
  ✓ Spring Data JPA Repository support
  ✓ Transaction management
```

```yaml
# ตั้งค่า DataSource ใน application.yml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/mydb
    username: user
    password: pass
    # HikariCP settings
    hikari:
      pool-name: MyHikariCP
      maximum-pool-size: 10
      minimum-idle: 2
      connection-timeout: 30000        # 30 seconds
      idle-timeout: 600000             # 10 minutes
      max-lifetime: 1800000            # 30 minutes
      connection-test-query: SELECT 1  # PostgreSQL
```

---

## ขั้นตอนที่ 143: Web MVC Auto-configuration

```
Spring Boot WebMvcAutoConfiguration configures:
  ✓ ContentNegotiatingViewResolver
  ✓ BeanNameViewResolver
  ✓ MessageConverters (JSON, XML, etc.)
  ✓ RequestMappingHandlerAdapter
  ✓ ExceptionHandlerExceptionResolver
  ✓ Static resources (/static, /public, /resources)
  ✓ Default error page
  ✓ Favicon
```

```java
// Override WebMVC configuration
@Configuration
public class WebMvcConfig implements WebMvcConfigurer {
    
    @Override
    public void addViewControllers(ViewControllerRegistry registry) {
        registry.addViewController("/").setViewName("redirect:/api/health");
    }
    
    @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry) {
        registry.addResourceHandler("/uploads/**")
            .addResourceLocations("file:uploads/")
            .setCacheControl(CacheControl.maxAge(30, TimeUnit.DAYS));
    }
    
    @Override
    public void extendMessageConverters(List<HttpMessageConverter<?>> converters) {
        // เพิ่ม custom converter
    }
    
    @Override
    public void addFormatters(FormatterRegistry registry) {
        registry.addConverter(new StringToLocalDateConverter());
    }
}
```

---

## ขั้นตอนที่ 144: Security Auto-configuration

```
เพิ่ม spring-boot-starter-security
        │
        ▼
SpringBootWebSecurityConfiguration auto-configures:
  ✓ ทุก endpoints ต้องการ authentication
  ✓ Basic Authentication
  ✓ Form-based login (/login, /logout)
  ✓ Random password สำหรับ user "user" (ดูใน console log)
  ✓ CSRF protection
  ✓ HTTP headers (X-Frame-Options, X-XSS-Protection, etc.)
```

```bash
# Default password ใน log:
Using generated security password: 7a1d7b83-4d53-4b24-9b6c-...
```

```java
// Override Security configuration
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    
    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/auth/**").permitAll()
                .requestMatchers("/actuator/health").permitAll()
                .anyRequest().authenticated()
            )
            .csrf(csrf -> csrf.disable())  // สำหรับ REST API
            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            );
        
        return http.build();
    }
}
```

---

## ขั้นตอนที่ 145: Caching Auto-configuration

```yaml
# เปิดใช้ @EnableCaching และ auto-configure
spring:
  cache:
    type: redis   # หรือ caffeine, simple, none
    redis:
      time-to-live: 600000  # 10 minutes
```

```java
// Spring Boot จะ configure CacheManager อัตโนมัติ
// ตาม spring.cache.type ที่ตั้งค่า

@SpringBootApplication
@EnableCaching
public class MyApp { }

@Service
public class ProductService {
    
    @Cacheable(value = "products", key = "#id")
    public ProductResponse findById(Long id) {
        // จะ call เฉพาะตอน cache miss
        return productRepository.findById(id)...;
    }
    
    @CacheEvict(value = "products", key = "#id")
    public void delete(Long id) { }
    
    @CachePut(value = "products", key = "#result.id")
    public ProductResponse update(Long id, UpdateRequest req) { ... }
}
```

---

## ขั้นตอนที่ 146: Actuator Auto-configuration

```yaml
# management = actuator settings
management:
  endpoints:
    web:
      exposure:
        include: "*"      # เปิดทุก endpoints
        exclude: "env"    # ยกเว้น env (sensitive)
      base-path: /manage  # เปลี่ยน base path
  
  endpoint:
    health:
      show-details: always    # always, when-authorized, never
      show-components: always
    
    shutdown:
      enabled: true           # เปิด /actuator/shutdown
  
  server:
    port: 8081               # actuator บน port ต่างหาก (ความปลอดภัย)
  
  metrics:
    tags:
      application: ${spring.application.name}
```

### Custom Health Indicator

```java
// สร้าง custom health check
@Component
public class DatabaseHealthIndicator implements HealthIndicator {
    
    private final DataSource dataSource;
    
    public DatabaseHealthIndicator(DataSource dataSource) {
        this.dataSource = dataSource;
    }
    
    @Override
    public Health health() {
        try (Connection conn = dataSource.getConnection();
             Statement stmt = conn.createStatement()) {
            
            stmt.execute("SELECT 1");
            
            return Health.up()
                .withDetail("database", "PostgreSQL")
                .withDetail("status", "Available")
                .build();
        } catch (Exception e) {
            return Health.down()
                .withDetail("error", e.getMessage())
                .build();
        }
    }
}

// Custom health checks:
// GET /actuator/health → {
//   "status": "UP",
//   "components": {
//     "db": { "status": "UP", "details": {...} },
//     "database": { "status": "UP", "details": {...} }
//   }
// }
```

---

## ขั้นตอนที่ 147: JPA Auto-configuration

```
spring-boot-starter-data-jpa auto-configures:
  ✓ EntityManagerFactory
  ✓ JpaTransactionManager
  ✓ Spring Data JPA Repositories
  ✓ Hibernate dialect (auto-detected)
  ✓ Connection pool (HikariCP)
```

```yaml
spring:
  jpa:
    # DDL auto:
    # none     - ไม่ทำอะไร
    # validate - ตรวจสอบ schema ตรงกับ entity หรือไม่
    # update   - update schema (ไม่แนะนำใน production)
    # create   - drop + create ทุกครั้ง
    # create-drop - create ตอน start, drop ตอน stop
    hibernate:
      ddl-auto: validate   # แนะนำสำหรับ production
    
    properties:
      hibernate:
        dialect: org.hibernate.dialect.PostgreSQLDialect
        format_sql: true
        use_sql_comments: true
        
        # Performance
        jdbc:
          batch_size: 50          # Batch inserts
        order_inserts: true       # Order inserts for batching
        order_updates: true
        
        # Query cache
        cache:
          use_second_level_cache: true
          use_query_cache: true
          region:
            factory_class: org.hibernate.cache.jcache.JCacheRegionFactory
    
    show-sql: false            # ปิดใน production
    open-in-view: false        # ปิดเพื่อ performance ที่ดีกว่า
```

---

## ขั้นตอนที่ 148: Auto-configuration Order

```java
// กำหนดลำดับ auto-configuration
@AutoConfiguration(
    before = SecurityAutoConfiguration.class,   // ทำก่อน
    after = DataSourceAutoConfiguration.class   // ทำหลัง
)
public class MyAutoConfiguration { }

// หรือใช้ @AutoConfigureBefore / @AutoConfigureAfter (deprecated ใน Spring Boot 3.x)
```

---

## ขั้นตอนที่ 149: สร้าง Spring Boot Starter

Starter คือ dependency ที่รวม:
1. Library หลัก
2. Auto-configuration
3. Dependencies ที่จำเป็น

```
my-sms-spring-boot-starter/
├── pom.xml
└── src/main/
    ├── java/
    │   └── com/example/sms/starter/
    │       ├── SmsProperties.java
    │       ├── SmsService.java
    │       ├── TwilioSmsService.java
    │       └── SmsAutoConfiguration.java
    └── resources/
        └── META-INF/
            └── spring/
                └── org.springframework.boot.autoconfigure.AutoConfiguration.imports
```

```xml
<!-- pom.xml ของ starter -->
<project>
    <groupId>com.example</groupId>
    <artifactId>my-sms-spring-boot-starter</artifactId>
    <version>1.0.0</version>
    <packaging>jar</packaging>
    
    <dependencies>
        <!-- Spring Boot Auto-configuration (บังคับ) -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-autoconfigure</artifactId>
        </dependency>
        
        <!-- ถ้าใช้ @ConfigurationProperties (แนะนำ) -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-configuration-processor</artifactId>
            <optional>true</optional>
        </dependency>
        
        <!-- Twilio SDK (optional - ไม่บังคับถ้า user ไม่ใช้) -->
        <dependency>
            <groupId>com.twilio.sdk</groupId>
            <artifactId>twilio</artifactId>
            <version>9.9.1</version>
            <optional>true</optional>
        </dependency>
    </dependencies>
</project>
```

---

## ขั้นตอนที่ 150: Configuration Metadata

```java
// Spring Boot configuration processor สร้าง metadata
// ให้ IDE แสดง autocomplete ใน application.properties

// เพิ่มใน pom.xml:
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-configuration-processor</artifactId>
    <optional>true</optional>
</dependency>

// จากนั้น build project → จะสร้าง:
// target/classes/META-INF/spring-configuration-metadata.json
```

```json
// spring-configuration-metadata.json (auto-generated)
{
  "groups": [
    {
      "name": "app",
      "type": "com.example.AppProperties",
      "sourceType": "com.example.AppProperties"
    }
  ],
  "properties": [
    {
      "name": "app.name",
      "type": "java.lang.String",
      "description": "Application name",
      "sourceType": "com.example.AppProperties"
    }
  ]
}
```

---

## ขั้นตอนที่ 151: Relaxed Binding

Spring Boot รองรับ property binding แบบ flexible:

```
# สิ่งนี้ทั้งหมดมีความหมายเดียวกัน:
spring.datasource.url
spring.datasource.URL
spring.datasource.Url
SPRING_DATASOURCE_URL          ← Environment variable
spring.datasource-url
spring.datasource_url

# ใน @ConfigurationProperties:
@ConfigurationProperties(prefix = "my-app")
public class MyAppProperties {
    private String serverUrl;  // รับค่าจาก my-app.server-url
}
```

---

## ขั้นตอนที่ 152: Property Source Order (ลำดับความสำคัญ)

```
ลำดับจากสูงสุดถึงต่ำสุด:
(สูงกว่าจะ override ต่ำกว่า)

1. Devtools global settings
2. @TestPropertySource (Testing)
3. @SpringBootTest properties (Testing)
4. Command line arguments:
   java -jar app.jar --server.port=9090
5. SPRING_APPLICATION_JSON (environment variable/property)
6. ServletConfig init parameters
7. ServletContext init parameters
8. JNDI (java:comp/env)
9. Java System properties: -Dserver.port=9090
10. OS environment variables: SERVER_PORT=9090
11. application-{profile}.properties (outside jar)
12. application.properties (outside jar)
13. @PropertySource annotations
14. application-{profile}.properties (inside jar)
15. application.properties (inside jar)  ← Default
16. @SpringBootApplication default properties
```

```bash
# ตัวอย่างการ override:

# application.properties
server.port=8080

# Override ด้วย command line:
java -jar app.jar --server.port=9090

# Override ด้วย environment variable:
SERVER_PORT=9090 java -jar app.jar

# Override ด้วย System property:
java -Dserver.port=9090 -jar app.jar
```

---

## ขั้นตอนที่ 153: Embedded Container Configuration

```yaml
server:
  port: 8080
  servlet:
    context-path: /myapp       # http://localhost:8080/myapp/api/...
  
  # SSL/TLS
  ssl:
    enabled: true
    key-store: classpath:keystore.p12
    key-store-password: secret
    key-store-type: PKCS12
    key-alias: tomcat
  
  # Compression
  compression:
    enabled: true
    min-response-size: 1024    # compress responses > 1KB
    mime-types:
      - application/json
      - text/html
  
  # HTTP/2
  http2:
    enabled: true
  
  # Tomcat specific
  tomcat:
    max-threads: 200
    min-spare-threads: 10
    accept-count: 100
    max-connections: 10000
    connection-timeout: 20000
    
    # Access log
    accesslog:
      enabled: true
      directory: logs
      pattern: "%t %a %r %s (%D ms)"
```

---

## ขั้นตอนที่ 154: Jackson Auto-configuration

```yaml
spring:
  jackson:
    # Date/Time format
    date-format: "yyyy-MM-dd HH:mm:ss"
    time-zone: "Asia/Bangkok"
    
    # Serialization
    serialization:
      write-dates-as-timestamps: false
      indent-output: false
    
    # Deserialization  
    deserialization:
      fail-on-unknown-properties: false  # ไม่ error ถ้า JSON มี field ที่ไม่รู้จัก
    
    # Include
    default-property-inclusion: non_null  # ไม่แสดง null fields
    
    # Naming strategy
    property-naming-strategy: SNAKE_CASE  # camelCase → snake_case
```

```java
// Custom ObjectMapper configuration
@Configuration
public class JacksonConfig {
    
    @Bean
    @Primary
    public ObjectMapper objectMapper() {
        ObjectMapper mapper = new ObjectMapper();
        
        // Register Java 8 Date/Time support
        mapper.registerModule(new JavaTimeModule());
        
        // Configuration
        mapper.configure(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS, false);
        mapper.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);
        mapper.setSerializationInclusion(JsonInclude.Include.NON_NULL);
        
        // Date format
        mapper.setDateFormat(new SimpleDateFormat("yyyy-MM-dd HH:mm:ss"));
        mapper.setTimeZone(TimeZone.getTimeZone("Asia/Bangkok"));
        
        return mapper;
    }
}
```

---

## ขั้นตอนที่ 155: สรุป Auto-configuration

```
สิ่งที่เรียนรู้ใน Part 07:

✅ Auto-configuration mechanism
✅ @EnableAutoConfiguration
✅ ดู auto-configuration ที่ทำงาน (--debug flag, actuator)
✅ @ConditionalOnXxx annotations ทั้งหมด
✅ สร้าง Custom Auto-configuration
✅ Override auto-configuration
✅ DataSource, Security, JPA auto-configuration
✅ Caching, Actuator auto-configuration
✅ Auto-configuration order
✅ สร้าง Spring Boot Starter
✅ Configuration Metadata
✅ Relaxed Binding
✅ Property Source Order (ลำดับความสำคัญ)
✅ Embedded Container Configuration
✅ Jackson Auto-configuration
```

### แบบฝึกหัด

```
1. สร้าง Custom Auto-configuration สำหรับ Logging Service ที่:
   - Enable เมื่อ property "logging.custom.enabled=true"
   - มี LoggingProperties class
   - Auto-configure สำหรับทั้ง dev (console) และ prod (file)

2. สร้าง Health Indicator ที่ check:
   - External API availability
   - Disk space > 1GB
   - Memory usage < 80%

3. Override default Jackson config เพื่อ:
   - แสดงวันที่เป็น Thai format (dd/MM/yyyy)
   - Serialize enum เป็น lowercase
   - ซ่อน null fields
```

---

*[← Part 06: Dependency Injection](./part-06-dependency-injection.md) | [Part 08: Properties และ Profiles →](./part-08-properties-profiles.md)*

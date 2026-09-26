# Part 01: บทนำและ Spring Boot คืออะไร
## ขั้นตอนที่ 1-15

> **ระดับ:** พื้นฐาน (Beginner)  
> **เวลาเรียน:** 2-3 ชั่วโมง  
> **เป้าหมาย:** เข้าใจว่า Spring Boot คืออะไร ทำงานอย่างไร และทำไมต้องใช้

---

## ขั้นตอนที่ 1: Spring Framework คืออะไร?

### ประวัติและที่มา

Spring Framework ถูกสร้างขึ้นในปี 2003 โดย **Rod Johnson** เพื่อแก้ปัญหาความซับซ้อนของ Java EE (Enterprise Edition) ในสมัยนั้น

```
ปัญหาของ Java EE แบบเดิม:
┌─────────────────────────────────────────┐
│  EJB (Enterprise JavaBeans)             │
│  - Configuration ซับซ้อนมาก            │
│  - XML ยาวมาก                          │
│  - Test ยาก                            │
│  - Deploy ช้า                          │
│  - Performance ต่ำ                     │
└─────────────────────────────────────────┘

Spring แก้ปัญหาด้วย:
┌─────────────────────────────────────────┐
│  - Lightweight Container                │
│  - Dependency Injection (DI)            │
│  - Aspect-Oriented Programming (AOP)   │
│  - Simple Configuration                │
│  - Easy Testing                        │
└─────────────────────────────────────────┘
```

### Spring Ecosystem

```
Spring Ecosystem
├── Spring Core (IoC/DI)
├── Spring MVC (Web)
├── Spring Data (Database)
├── Spring Security (Authentication/Authorization)
├── Spring Cloud (Microservices)
├── Spring Batch (Batch Processing)
├── Spring Integration (EIP)
├── Spring WebFlux (Reactive)
└── Spring Boot (Auto-configuration)
```

---

## ขั้นตอนที่ 2: Spring Boot คืออะไร?

Spring Boot คือเฟรมเวิร์คที่ทำให้การสร้างแอพพลิเคชัน Spring ง่ายขึ้นมาก โดยหลักการ **"Convention over Configuration"**

### ก่อน Spring Boot (แบบเดิม)

```xml
<!-- web.xml - ต้องสร้างเอง -->
<web-app>
    <servlet>
        <servlet-name>dispatcher</servlet-name>
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
        <init-param>
            <param-name>contextConfigLocation</param-name>
            <param-value>/WEB-INF/spring/dispatcher-config.xml</param-value>
        </init-param>
        <load-on-startup>1</load-on-startup>
    </servlet>
    <servlet-mapping>
        <servlet-name>dispatcher</servlet-name>
        <url-pattern>/</url-pattern>
    </servlet-mapping>
</web-app>
```

```xml
<!-- dispatcher-config.xml - ต้องสร้างเอง -->
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:mvc="http://www.springframework.org/schema/mvc"
       xmlns:context="http://www.springframework.org/schema/context">
    
    <mvc:annotation-driven/>
    <context:component-scan base-package="com.example"/>
    
    <bean class="org.springframework.web.servlet.view.InternalResourceViewResolver">
        <property name="prefix" value="/WEB-INF/views/"/>
        <property name="suffix" value=".jsp"/>
    </bean>
</beans>
```

### หลัง Spring Boot (แบบใหม่)

```java
// เพียง 3 บรรทัด! ทุกอย่าง Auto-configure ให้หมด
@SpringBootApplication
public class MyApplication {
    public static void main(String[] args) {
        SpringApplication.run(MyApplication.class, args);
    }
}
```

---

## ขั้นตอนที่ 3: หลักการสำคัญของ Spring Boot

### 1. Auto-configuration
Spring Boot จะ configure สิ่งต่างๆ ให้อัตโนมัติตาม dependencies ที่มี

```
เพิ่ม spring-boot-starter-web →
Spring Boot จะ auto-configure:
  ✓ Tomcat embedded server
  ✓ Spring MVC
  ✓ Jackson (JSON serialization)
  ✓ DispatcherServlet
  ✓ Error handling
```

### 2. Starter Dependencies
แทนที่จะเพิ่ม dependencies แยกกัน ใช้ "starter" แทน

```xml
<!-- แบบเก่า - ต้องเพิ่มทีละ dependency -->
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-webmvc</artifactId>
    <version>6.1.0</version>
</dependency>
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-web</artifactId>
    <version>6.1.0</version>
</dependency>
<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
    <version>2.15.0</version>
</dependency>
<!-- ... อีกหลายตัว -->

<!-- แบบใหม่ - ใช้ starter แค่ตัวเดียว -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

### 3. Embedded Server
ไม่ต้อง deploy WAR ไป Tomcat แยก, รันตรงจาก JAR ได้เลย

```
แบบเก่า:
Build WAR → Deploy to Tomcat → Start Server

แบบใหม่:
Build JAR → java -jar app.jar (Tomcat ฝังอยู่ข้างใน!)
```

### 4. Production-ready Features
มี Actuator สำหรับ monitoring พร้อมใช้งานทันที

```
GET /actuator/health   → Health check
GET /actuator/metrics  → Application metrics
GET /actuator/info     → App information
```

---

## ขั้นตอนที่ 4: Spring Boot 3.x vs 2.x

```
Spring Boot 2.x (สิ้นสุด Support ปลาย 2023)
├── Java 8+ required
├── Jakarta EE 8 (javax.* namespace)
├── Spring Framework 5.x
└── Reactive ผ่าน Project Reactor

Spring Boot 3.x (ปัจจุบัน - แนะนำ!)
├── Java 17+ required (แนะนำ Java 21)
├── Jakarta EE 10 (jakarta.* namespace) ← เปลี่ยน!
├── Spring Framework 6.x
├── Native Image support ด้วย GraalVM
├── Virtual Threads (Project Loom) - Java 21
└── Observability ด้วย Micrometer
```

### การเปลี่ยนแปลงสำคัญ

```java
// Spring Boot 2.x
import javax.persistence.Entity;     // javax.*
import javax.validation.Valid;
import javax.servlet.http.HttpServletRequest;

// Spring Boot 3.x
import jakarta.persistence.Entity;   // jakarta.*  ← เปลี่ยน!
import jakarta.validation.Valid;
import jakarta.servlet.http.HttpServletRequest;
```

---

## ขั้นตอนที่ 5: Architecture ของ Spring Boot Application

```
┌─────────────────────────────────────────────────────────┐
│                    CLIENT (Browser/Mobile)               │
└─────────────────────────┬───────────────────────────────┘
                          │ HTTP Request
                          ▼
┌─────────────────────────────────────────────────────────┐
│                  Spring Boot Application                 │
│  ┌─────────────────────────────────────────────────┐   │
│  │              Controller Layer                    │   │
│  │  @RestController / @Controller                  │   │
│  │  รับ HTTP Request, ส่งต่อไป Service             │   │
│  └──────────────────────┬──────────────────────────┘   │
│                         │                               │
│  ┌──────────────────────▼──────────────────────────┐   │
│  │               Service Layer                      │   │
│  │  @Service                                        │   │
│  │  Business Logic ทั้งหมดอยู่ที่นี่              │   │
│  └──────────────────────┬──────────────────────────┘   │
│                         │                               │
│  ┌──────────────────────▼──────────────────────────┐   │
│  │             Repository Layer                     │   │
│  │  @Repository / JpaRepository                    │   │
│  │  ติดต่อ Database                                │   │
│  └──────────────────────┬──────────────────────────┘   │
│                         │                               │
└─────────────────────────┼───────────────────────────────┘
                          │
┌─────────────────────────▼───────────────────────────────┐
│                      Database                            │
│              MySQL / PostgreSQL / MongoDB                │
└─────────────────────────────────────────────────────────┘
```

---

## ขั้นตอนที่ 6: Spring Boot Starters ที่สำคัญ

```xml
<!-- Web Development -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>

<!-- Database + JPA -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>

<!-- Security -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>

<!-- Testing -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>

<!-- Validation -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-validation</artifactId>
</dependency>

<!-- Caching -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-cache</artifactId>
</dependency>

<!-- Actuator (Monitoring) -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>

<!-- Thymeleaf (Template Engine) -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-thymeleaf</artifactId>
</dependency>

<!-- WebSocket -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-websocket</artifactId>
</dependency>

<!-- Reactive Web -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-webflux</artifactId>
</dependency>

<!-- Mail -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-mail</artifactId>
</dependency>

<!-- Redis Cache -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>

<!-- MongoDB -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-mongodb</artifactId>
</dependency>

<!-- Batch Processing -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-batch</artifactId>
</dependency>
```

---

## ขั้นตอนที่ 7: เปรียบเทียบ Spring Boot กับ Framework อื่น

### Java Frameworks

| Feature | Spring Boot | Quarkus | Micronaut |
|---------|-------------|---------|-----------|
| Startup Time | ~2-5 วิ | ~0.1 วิ | ~0.5 วิ |
| Memory Usage | สูง | ต่ำมาก | ต่ำ |
| Community | ใหญ่มาก | กำลังโต | กำลังโต |
| Learning Curve | ปานกลาง | ปานกลาง | สูง |
| Job Market | สูงมาก | ปานกลาง | น้อย |
| Native Image | รองรับ | รองรับดี | รองรับ |

### เทียบกับภาษาอื่น

| Framework | ภาษา | Throughput | Use Case |
|-----------|------|-----------|---------|
| Spring Boot | Java | สูง | Enterprise, เสถียร |
| Node.js/Express | JavaScript | สูงมาก | Real-time, API |
| Django | Python | ปานกลาง | ML, Data Science |
| Ruby on Rails | Ruby | ปานกลาง | Rapid Development |
| ASP.NET Core | C# | สูงมาก | Microsoft Stack |
| FastAPI | Python | สูง | API, ML |

---

## ขั้นตอนที่ 8: ทำไมต้องเลือก Spring Boot?

### เหตุผลที่ควรใช้ Spring Boot

```
1. 🏢 ใช้งานใน Enterprise ระดับโลก
   - Netflix, Amazon, Google, Alibaba
   - ธนาคาร, รัฐบาล, บริษัทใหญ่ๆ

2. 💼 งานในตลาดสูงมาก
   - Java/Spring Boot Developer เป็นที่ต้องการมาก
   - เงินเดือนสูงกว่าภาษาอื่น

3. 🌟 Ecosystem สมบูรณ์
   - Spring Data, Security, Cloud, Batch
   - รองรับทุก Use Case

4. 📚 Documentation ดีมาก
   - docs.spring.io มีครบ
   - Community ใหญ่มาก

5. 🔒 Security ดีเยี่ยม
   - Spring Security
   - OAuth2, JWT built-in

6. ⚡ Performance สูง
   - Virtual Threads (Java 21)
   - Reactive WebFlux

7. 🐳 Cloud Native
   - Docker-friendly
   - Kubernetes-ready
   - Cloud providers support
```

---

## ขั้นตอนที่ 9: วงจรชีวิต Spring Application Context

```java
// Spring Boot Application Life Cycle
public class SpringBootLifecycle {
    
    // 1. JVM starts, main() called
    public static void main(String[] args) {
        
        // 2. SpringApplication.run() ถูกเรียก
        SpringApplication app = new SpringApplication(Application.class);
        
        // 3. ApplicationContext สร้างขึ้น
        // 4. Beans ทั้งหมดถูก scan และ create
        // 5. Auto-configuration ทำงาน
        // 6. Embedded Server เริ่ม
        // 7. ApplicationStartedEvent fired
        // 8. CommandLineRunner/ApplicationRunner ทำงาน
        // 9. ApplicationReadyEvent fired
        
        ConfigurableApplicationContext ctx = app.run(args);
        
        // 10. Application กำลัง running...
        
        // เมื่อ shutdown:
        // 11. ApplicationContext.close() ถูกเรียก
        // 12. Beans ถูก destroy
        // 13. JVM exits
    }
}
```

### Events ใน Spring Boot

```java
@Component
public class ApplicationEventListener {
    
    private static final Logger log = LoggerFactory.getLogger(ApplicationEventListener.class);
    
    // เรียกก่อน ApplicationContext ถูกสร้าง
    @EventListener(ApplicationStartingEvent.class)
    public void onStarting(ApplicationStartingEvent event) {
        log.info("Application is starting...");
    }
    
    // เรียกหลัง Environment ถูก prepared
    @EventListener(ApplicationEnvironmentPreparedEvent.class)
    public void onEnvironmentPrepared(ApplicationEnvironmentPreparedEvent event) {
        log.info("Environment is prepared");
    }
    
    // เรียกหลัง ApplicationContext ถูกสร้าง แต่ก่อน refresh
    @EventListener(ApplicationContextInitializedEvent.class)
    public void onContextInitialized(ApplicationContextInitializedEvent event) {
        log.info("Application context initialized");
    }
    
    // เรียกหลัง ApplicationContext ถูก refresh แต่ก่อน runner
    @EventListener(ApplicationStartedEvent.class)
    public void onStarted(ApplicationStartedEvent event) {
        log.info("Application started successfully");
    }
    
    // เรียกหลัง CommandLineRunner/ApplicationRunner ทำงานเสร็จ
    @EventListener(ApplicationReadyEvent.class)
    public void onReady(ApplicationReadyEvent event) {
        log.info("Application is ready to serve requests!");
    }
    
    // เรียกเมื่อเกิด error ระหว่าง startup
    @EventListener(ApplicationFailedEvent.class)
    public void onFailed(ApplicationFailedEvent event) {
        log.error("Application failed to start!", event.getException());
    }
}
```

---

## ขั้นตอนที่ 10: Spring Boot Annotations พื้นฐาน

```java
// === Class-level Annotations ===

// ทำให้ class เป็น Spring managed bean
@Component
// specializations ของ @Component:
@Service      // Business logic
@Repository   // Data access
@Controller   // Web MVC controller
@RestController // REST API controller (= @Controller + @ResponseBody)

// Configuration class
@Configuration

// Spring Boot main annotation (รวม 3 annotation)
@SpringBootApplication
// = @Configuration + @EnableAutoConfiguration + @ComponentScan

// === Method-level Annotations ===

// HTTP Method mapping
@GetMapping("/path")
@PostMapping("/path")
@PutMapping("/path")
@DeleteMapping("/path")
@PatchMapping("/path")
@RequestMapping(value = "/path", method = RequestMethod.GET)

// Bean definition
@Bean

// Scheduled task
@Scheduled(fixedRate = 5000)

// Transaction
@Transactional

// Caching
@Cacheable("cacheName")
@CacheEvict("cacheName")
@CachePut("cacheName")

// === Parameter-level Annotations ===

// Inject path variable
@PathVariable("id") Long id

// Inject query parameter
@RequestParam("name") String name

// Inject request body (JSON)
@RequestBody UserDTO userDTO

// Inject request header
@RequestHeader("Authorization") String token

// === Field-level Annotations ===

// Dependency Injection
@Autowired
@Inject  // JSR-330 standard

// Value from properties
@Value("${app.name}")
private String appName;

// === Validation Annotations ===
@NotNull
@NotBlank
@NotEmpty
@Min(1)
@Max(100)
@Size(min = 2, max = 50)
@Email
@Pattern(regexp = "^[0-9]{10}$")
@Valid  // Trigger validation
```

---

## ขั้นตอนที่ 11: เข้าใจ Dependency Injection (DI)

### ปัญหาโดยไม่มี DI

```java
// ❌ แบบเดิม - Tight Coupling
public class OrderService {
    
    // สร้าง object เองตรงๆ - ทำให้ test ยาก, เปลี่ยนยาก
    private PaymentService paymentService = new PaymentService();
    private EmailService emailService = new EmailService();
    private InventoryService inventoryService = new InventoryService();
    
    public void createOrder(Order order) {
        inventoryService.reduceStock(order);
        paymentService.processPayment(order);
        emailService.sendConfirmation(order);
    }
}
```

### แก้ด้วย DI

```java
// ✅ แบบใหม่ - Loose Coupling with DI

// Interface กำหนด contract
public interface PaymentService {
    void processPayment(Order order);
}

// Implementation
@Service
public class StripePaymentService implements PaymentService {
    @Override
    public void processPayment(Order order) {
        // Stripe payment logic
    }
}

// OrderService รับ dependencies ผ่าน Constructor
@Service
public class OrderService {
    
    private final PaymentService paymentService;
    private final EmailService emailService;
    private final InventoryService inventoryService;
    
    // Constructor Injection (แนะนำ!)
    public OrderService(
        PaymentService paymentService,
        EmailService emailService,
        InventoryService inventoryService
    ) {
        this.paymentService = paymentService;
        this.emailService = emailService;
        this.inventoryService = inventoryService;
    }
    
    public void createOrder(Order order) {
        inventoryService.reduceStock(order);
        paymentService.processPayment(order);
        emailService.sendConfirmation(order);
    }
}
```

### ประเภทของ Injection

```java
// 1. Constructor Injection (แนะนำมากที่สุด!)
@Service
public class UserService {
    
    private final UserRepository userRepository; // final ได้!
    private final EmailService emailService;
    
    public UserService(UserRepository userRepository, EmailService emailService) {
        this.userRepository = userRepository;
        this.emailService = emailService;
    }
}

// 2. Field Injection (ง่ายแต่ไม่แนะนำ - test ยาก)
@Service
public class UserService {
    
    @Autowired
    private UserRepository userRepository; // ไม่สามารถเป็น final ได้
    
    @Autowired
    private EmailService emailService;
}

// 3. Setter Injection (ใช้เมื่อ optional dependency)
@Service
public class UserService {
    
    private UserRepository userRepository;
    
    @Autowired
    public void setUserRepository(UserRepository userRepository) {
        this.userRepository = userRepository;
    }
}
```

---

## ขั้นตอนที่ 12: Spring Bean Scopes

```java
// 1. Singleton (default) - 1 instance ต่อ ApplicationContext
@Component
@Scope("singleton")  // หรือไม่ต้องใส่ เป็น default
public class SingletonService {
    private int count = 0;
    
    public void increment() { count++; }
    public int getCount() { return count; }
}

// 2. Prototype - สร้าง instance ใหม่ทุกครั้งที่ request
@Component
@Scope("prototype")
public class PrototypeService {
    // ใช้กับ stateful objects
}

// 3. Request - 1 instance ต่อ HTTP Request (Web only)
@Component
@Scope(value = WebApplicationContext.SCOPE_REQUEST,
       proxyMode = ScopedProxyMode.TARGET_CLASS)
public class RequestScopedService {
    // ใช้เก็บ request-specific data
}

// 4. Session - 1 instance ต่อ HTTP Session (Web only)
@Component
@Scope(value = WebApplicationContext.SCOPE_SESSION,
       proxyMode = ScopedProxyMode.TARGET_CLASS)
public class SessionScopedService {
    // ใช้เก็บ user session data
}

// 5. Application - 1 instance ต่อ ServletContext (Web only)
@Component
@Scope(value = WebApplicationContext.SCOPE_APPLICATION)
public class ApplicationScopedService {
    // คล้าย Singleton แต่ใน web context
}
```

---

## ขั้นตอนที่ 13: Auto-configuration ทำงานอย่างไร?

```java
// Spring Boot ดูว่า class ไหนอยู่ใน classpath
// แล้ว auto-configure ให้เหมาะสม

// ตัวอย่าง: DataSourceAutoConfiguration
@Configuration
@ConditionalOnClass({ DataSource.class, EmbeddedDatabaseType.class })  // ถ้ามี DataSource
@ConditionalOnMissingBean(type = "io.r2dbc.spi.ConnectionFactory")     // และไม่มี R2DBC
@Import({ DataSourceConfiguration.Hikari.class,
          DataSourceConfiguration.Tomcat.class,
          DataSourceConfiguration.Dbcp2.class,
          DataSourceConfiguration.OracleUcp.class,
          DataSourceConfiguration.Generic.class,
          DataSourceJmxConfiguration.class })
public class DataSourceAutoConfiguration {
    // Configure DataSource อัตโนมัติ
}

// เราดูได้ว่า auto-configuration อะไรทำงาน:
// 1. เพิ่ม actuator endpoint
// GET /actuator/conditions → แสดง auto-configuration report

// 2. ใช้ --debug flag
// java -jar app.jar --debug

// 3. ใช้ properties
// logging.level.org.springframework.boot.autoconfigure=DEBUG
```

### การ Disable Auto-configuration

```java
// Disable specific auto-configuration
@SpringBootApplication(exclude = {
    DataSourceAutoConfiguration.class,
    SecurityAutoConfiguration.class
})
public class MyApplication {
    public static void main(String[] args) {
        SpringApplication.run(MyApplication.class, args);
    }
}

// หรือใน properties
// spring.autoconfigure.exclude=org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration
```

---

## ขั้นตอนที่ 14: Application Context Types

```java
// 1. AnnotationConfigApplicationContext
// สำหรับ standalone Java application
ApplicationContext ctx = 
    new AnnotationConfigApplicationContext(AppConfig.class);

// 2. ClassPathXmlApplicationContext
// สำหรับ XML-based configuration (แบบเก่า)
ApplicationContext ctx = 
    new ClassPathXmlApplicationContext("applicationContext.xml");

// 3. AnnotationConfigServletWebServerApplicationContext
// สำหรับ Spring Boot Web Application (สร้างอัตโนมัติ)
// ใช้กับ spring-boot-starter-web

// 4. AnnotationConfigReactiveWebServerApplicationContext
// สำหรับ Spring Boot Reactive Application
// ใช้กับ spring-boot-starter-webflux

// ตรวจสอบ beans ใน context
@SpringBootApplication
public class MyApplication {
    public static void main(String[] args) {
        ApplicationContext ctx = SpringApplication.run(MyApplication.class, args);
        
        // ดู beans ทั้งหมด
        String[] beanNames = ctx.getBeanDefinitionNames();
        Arrays.sort(beanNames);
        for (String beanName : beanNames) {
            System.out.println(beanName);
        }
        
        System.out.println("Total beans: " + ctx.getBeanDefinitionCount());
    }
}
```

---

## ขั้นตอนที่ 15: สรุปและเตรียมตัวสำหรับ Part ต่อไป

### สิ่งที่เรียนรู้ใน Part 01

```
✅ Spring Framework คืออะไรและมาจากไหน
✅ Spring Boot คืออะไร และทำไมต้องใช้
✅ หลักการ Auto-configuration, Starter Dependencies, Embedded Server
✅ Spring Boot 3.x vs 2.x
✅ Architecture 3 Layers (Controller, Service, Repository)
✅ Starters ที่สำคัญ
✅ Life Cycle ของ Spring Boot Application
✅ Annotations พื้นฐาน
✅ Dependency Injection (DI) ทั้ง 3 แบบ
✅ Bean Scopes
✅ Auto-configuration mechanism
✅ Application Context Types
```

### ความรู้ที่ต้องมีก่อนเรียน Part 02

```java
// Java พื้นฐานที่ต้องรู้:

// 1. Class และ Object
public class Person {
    private String name;
    private int age;
    
    // Constructor
    public Person(String name, int age) {
        this.name = name;
        this.age = age;
    }
    
    // Getters
    public String getName() { return name; }
    public int getAge() { return age; }
}

// 2. Interface
public interface Greetable {
    void greet();
}

// 3. Inheritance
public class Employee extends Person implements Greetable {
    private String company;
    
    public Employee(String name, int age, String company) {
        super(name, age);
        this.company = company;
    }
    
    @Override
    public void greet() {
        System.out.println("Hello, I'm " + getName() + " from " + company);
    }
}

// 4. Generics
List<String> names = new ArrayList<>();
Map<String, Integer> ages = new HashMap<>();

// 5. Lambda expressions (Java 8+)
List<String> filtered = names.stream()
    .filter(name -> name.startsWith("J"))
    .collect(Collectors.toList());

// 6. Optional (Java 8+)
Optional<String> optional = Optional.of("Hello");
String value = optional.orElse("Default");
```

### แบบฝึกหัด

```
1. ไปดูเว็บไซต์ https://spring.io เพื่อทำความรู้จัก ecosystem
2. ทำความเข้าใจ Maven/Gradle project structure
3. ลอง search หา "Spring Boot" ใน job sites เพื่อดู requirements
4. อ่านเกี่ยวกับ HTTP Methods (GET, POST, PUT, DELETE, PATCH)
5. ทบทวน Java 8+ features: Lambda, Stream, Optional
```

### เตรียมตัวสำหรับ Part 02

ใน **Part 02** เราจะ:
- ติดตั้ง JDK 21
- ติดตั้ง IntelliJ IDEA
- ติดตั้ง Maven
- ติดตั้ง Docker
- ตั้งค่าต่างๆ
- สร้างโปรเจคแรกด้วย Spring Initializr

---

> **หมายเหตุ:** เนื้อหาทั้งหมดใน Part นี้เป็นทฤษฎีและพื้นฐาน  
> ใน Part ถัดไปเราจะเริ่มเขียนโค้ดจริงกัน!

---

*[← Back to README](./README.md) | [Part 02: ติดตั้ง Development Environment →](./part-02-setup-environment.md)*

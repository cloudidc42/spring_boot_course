# Part 06: Dependency Injection และ IoC Container
## ขั้นตอนที่ 111-135

> **ระดับ:** พื้นฐาน (Beginner)  
> **เวลาเรียน:** 3-4 ชั่วโมง  
> **เป้าหมาย:** เข้าใจและใช้งาน DI และ IoC Container อย่างมืออาชีพ

---

## ขั้นตอนที่ 111: IoC Container คืออะไร?

IoC (Inversion of Control) หมายถึงการ "ย้อนกลับ" การควบคุมการสร้าง objects จากโค้ดของเราไปยัง Framework

```
แบบดั้งเดิม (No IoC):
╔════════════════════════════════════╗
║  OrderService                      ║
║  ┌─────────────────────────────┐  ║
║  │  paymentService =           │  ║
║  │    new PaymentService();    │  ║
║  │                             │  ║
║  │  emailService =             │  ║
║  │    new EmailService();      │  ║
║  └─────────────────────────────┘  ║
╚════════════════════════════════════╝
- OrderService ต้องรู้ว่าสร้าง dependencies อย่างไร
- เปลี่ยน implementation ยาก
- Test ยาก (ต้อง instantiate จริงๆ)

แบบ IoC (Spring Container):
╔════════════════════════════════════════╗
║  Spring IoC Container                  ║
║  ┌───────────────────────────────┐    ║
║  │  OrderService                  │    ║
║  │  ← PaymentService (injected)  │    ║
║  │  ← EmailService (injected)    │    ║
║  └───────────────────────────────┘    ║
║                                        ║
║  Container จัดการ:                     ║
║  - สร้าง objects                       ║
║  - จัดการ lifecycle                    ║
║  - Inject dependencies                 ║
╚════════════════════════════════════════╝
- OrderService ไม่ต้องสร้าง dependencies เอง
- เปลี่ยน implementation ง่าย
- Test ง่าย (inject mock ได้)
```

---

## ขั้นตอนที่ 112: Spring Bean คืออะไร?

Spring Bean คือ object ที่ Spring Container จัดการ lifecycle และ dependency injection

```java
// วิธีสร้าง Bean:

// 1. @Component (และ stereotypes ของมัน)
@Component
public class MyComponent { }

@Service
public class MyService { }

@Repository
public class MyRepository { }

@Controller
public class MyController { }

@RestController
public class MyRestController { }

// 2. @Bean ใน @Configuration class
@Configuration
public class AppConfig {
    
    @Bean
    public MyService myService() {
        return new MyService();  // Spring จะ manage instance นี้
    }
    
    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
    
    // @Bean สามารถ depend กันได้
    @Bean
    public UserService userService(UserRepository repo, PasswordEncoder encoder) {
        return new UserService(repo, encoder);
    }
}

// 3. @Import
@SpringBootApplication
@Import(ExternalConfig.class)
public class MyApp { }
```

---

## ขั้นตอนที่ 113: @ComponentScan

```java
// Default: scan package ของ main class และ sub-packages
@SpringBootApplication  // = @ComponentScan(basePackages = "com.example.myapp")
public class MyApp { }

// Custom scan
@SpringBootApplication
@ComponentScan(
    basePackages = {"com.example.myapp", "com.example.shared"},  // หลาย packages
    excludeFilters = @ComponentScan.Filter(
        type = FilterType.ANNOTATION,
        classes = Repository.class  // ยกเว้น repositories
    )
)
public class MyApp { }
```

---

## ขั้นตอนที่ 114: Dependency Injection แบบต่างๆ

```java
// ======================================
// 1. Constructor Injection (แนะนำมากที่สุด!)
// ======================================
@Service
public class UserService {
    
    private final UserRepository userRepository;
    private final EmailService emailService;
    private final PasswordEncoder passwordEncoder;
    
    // Spring จะ inject อัตโนมัติเมื่อมี 1 constructor
    public UserService(
        UserRepository userRepository,
        EmailService emailService,
        PasswordEncoder passwordEncoder
    ) {
        this.userRepository = userRepository;
        this.emailService = emailService;
        this.passwordEncoder = passwordEncoder;
    }
}

// ใช้ Lombok ลด boilerplate
@Service
@RequiredArgsConstructor  // สร้าง constructor สำหรับ final fields อัตโนมัติ
public class UserService {
    
    private final UserRepository userRepository;      // final = inject via constructor
    private final EmailService emailService;
    private final PasswordEncoder passwordEncoder;
}

/*
ข้อดี Constructor Injection:
✅ Dependencies เป็น final (immutable)
✅ Test ง่ายมาก (inject mock ผ่าน constructor)
✅ ชัดเจนว่า class ต้องการอะไร
✅ ป้องกัน circular dependency (ตรวจพบตั้งแต่ startup)
✅ เหมาะกับ required dependencies
*/

// ======================================
// 2. Field Injection (ไม่แนะนำ!)
// ======================================
@Service
public class UserService {
    
    @Autowired  // Spring inject โดยตรงผ่าน reflection
    private UserRepository userRepository;
    
    @Autowired
    private EmailService emailService;
}

/*
ข้อเสีย Field Injection:
❌ ไม่สามารถทำเป็น final ได้
❌ Test ยากกว่า (ต้องใช้ @InjectMocks หรือ reflection)
❌ ซ่อน dependencies (ไม่เห็น ชัดเจนว่า class ต้องการอะไร)
❌ อาจเกิด NullPointerException ถ้าใช้นอก Spring context
*/

// ======================================
// 3. Setter Injection (ใช้เมื่อ optional dependency)
// ======================================
@Service
public class NotificationService {
    
    private EmailService emailService;
    private SmsService smsService;  // Optional
    
    @Autowired
    public void setEmailService(EmailService emailService) {
        this.emailService = emailService;
    }
    
    @Autowired(required = false)  // Optional dependency
    public void setSmsService(SmsService smsService) {
        this.smsService = smsService;
    }
    
    public void notify(String message) {
        emailService.send(message);
        if (smsService != null) {  // Check null for optional
            smsService.send(message);
        }
    }
}
```

---

## ขั้นตอนที่ 115: @Autowired และ @Qualifier

```java
// เมื่อมี interface หลาย implementations:

public interface PaymentGateway {
    void processPayment(BigDecimal amount);
}

@Component("stripeGateway")
public class StripePaymentGateway implements PaymentGateway {
    @Override
    public void processPayment(BigDecimal amount) {
        // Stripe implementation
    }
}

@Component("paypalGateway")
public class PayPalPaymentGateway implements PaymentGateway {
    @Override
    public void processPayment(BigDecimal amount) {
        // PayPal implementation
    }
}

// ปัญหา: Spring ไม่รู้จะ inject ตัวไหน!
@Service
public class OrderService {
    
    // ❌ Error: expected single matching bean but found 2
    @Autowired
    private PaymentGateway paymentGateway;
}

// แก้ด้วย @Qualifier
@Service
@RequiredArgsConstructor
public class OrderService {
    
    @Qualifier("stripeGateway")  // ระบุ bean name
    private final PaymentGateway paymentGateway;
    
    // หรือใช้ constructor
    public OrderService(@Qualifier("stripeGateway") PaymentGateway paymentGateway) {
        this.paymentGateway = paymentGateway;
    }
}

// แก้ด้วย @Primary (กำหนด default bean)
@Component
@Primary  // ← ใช้ตัวนี้เป็น default
public class StripePaymentGateway implements PaymentGateway { }
```

---

## ขั้นตอนที่ 116: Bean Scopes ลึกขึ้น

```java
// 1. Singleton (default)
@Component
// @Scope("singleton")  // ไม่ต้องใส่ เป็น default
public class ConfigService {
    private final Map<String, String> config = new HashMap<>();
    // 1 instance ตลอด lifetime ของ application
}

// 2. Prototype - สร้างใหม่ทุกครั้ง
@Component
@Scope("prototype")
public class TaskProcessor {
    private String taskName;
    
    // ทุกครั้งที่ inject จะได้ instance ใหม่
}

// 3. Request - 1 instance ต่อ HTTP Request
@Component
@Scope(
    value = WebApplicationContext.SCOPE_REQUEST,
    proxyMode = ScopedProxyMode.TARGET_CLASS  // จำเป็น!
)
public class RequestContext {
    private String currentUserId;
    // แต่ละ request มีของตัวเอง
}

// 4. Session - 1 instance ต่อ HTTP Session
@Component
@Scope(
    value = WebApplicationContext.SCOPE_SESSION,
    proxyMode = ScopedProxyMode.TARGET_CLASS
)
public class UserSessionData {
    private String username;
    private List<String> recentViews = new ArrayList<>();
}

// 5. Application - 1 instance ต่อ ServletContext
@Component
@Scope(WebApplicationContext.SCOPE_APPLICATION)
public class AppStatistics {
    private AtomicInteger requestCount = new AtomicInteger(0);
}
```

### Prototype in Singleton: ปัญหาและวิธีแก้

```java
// ❌ ปัญหา: Prototype bean ที่ inject ใน Singleton จะไม่ได้ instance ใหม่ทุกครั้ง!
@Service
public class OrderService {  // Singleton
    
    @Autowired
    private TaskProcessor processor;  // Prototype แต่ inject แค่ครั้งเดียว!
    
    public void processOrder() {
        // processor จะเป็น instance เดิมตลอด ❌
        processor.process();
    }
}

// ✅ วิธีแก้ 1: ใช้ ApplicationContext
@Service
@RequiredArgsConstructor
public class OrderService {
    
    private final ApplicationContext context;
    
    public void processOrder() {
        TaskProcessor processor = context.getBean(TaskProcessor.class);
        processor.process();  // ได้ instance ใหม่ทุกครั้ง ✅
    }
}

// ✅ วิธีแก้ 2: Lookup Method Injection
@Service
public abstract class OrderService {
    
    @Lookup
    public abstract TaskProcessor getTaskProcessor();
    
    public void processOrder() {
        TaskProcessor processor = getTaskProcessor();  // Spring override ให้
        processor.process();
    }
}

// ✅ วิธีแก้ 3: ObjectProvider (Spring 4.3+)
@Service
@RequiredArgsConstructor
public class OrderService {
    
    private final ObjectProvider<TaskProcessor> processorProvider;
    
    public void processOrder() {
        TaskProcessor processor = processorProvider.getObject();  // Lazy, ได้ใหม่ทุกครั้ง
        processor.process();
    }
}
```

---

## ขั้นตอนที่ 117: @Value Injection

```java
@Service
public class AppService {
    
    // จาก properties file
    @Value("${app.name}")
    private String appName;
    
    // ค่า default
    @Value("${app.timeout:30}")
    private int timeout;
    
    // SpEL (Spring Expression Language)
    @Value("#{systemProperties['user.home']}")
    private String userHome;
    
    @Value("#{T(java.lang.Math).PI}")
    private double pi;
    
    // อ่าน system environment
    @Value("${JAVA_HOME:not-set}")
    private String javaHome;
    
    // List และ Map
    @Value("${app.allowed-origins}")
    private List<String> allowedOrigins;
    
    @Value("#{${app.database-config}}")  // SpEL map literal
    private Map<String, String> databaseConfig;
    
    // Static field - ต้องใช้ setter
    private static String staticValue;
    
    @Value("${app.static-value}")
    public void setStaticValue(String value) {
        AppService.staticValue = value;
    }
}
```

---

## ขั้นตอนที่ 118: @Conditional Annotations

```java
// @ConditionalOnProperty - สร้าง bean เมื่อ property มีค่าที่กำหนด
@Configuration
public class FeatureConfig {
    
    @Bean
    @ConditionalOnProperty(
        name = "feature.payment.stripe.enabled",
        havingValue = "true",
        matchIfMissing = false
    )
    public PaymentGateway stripeGateway() {
        return new StripePaymentGateway();
    }
    
    // @ConditionalOnClass - สร้าง bean เมื่อ class อยู่ใน classpath
    @Bean
    @ConditionalOnClass(name = "com.stripe.Stripe")
    public StripeClient stripeClient() {
        return new StripeClient();
    }
    
    // @ConditionalOnMissingBean - สร้าง bean เมื่อไม่มี bean นี้อยู่แล้ว
    @Bean
    @ConditionalOnMissingBean(DataSource.class)
    public DataSource defaultDataSource() {
        return new EmbeddedDatabaseBuilder()
            .setType(EmbeddedDatabaseType.H2)
            .build();
    }
    
    // @ConditionalOnProfile - สร้าง bean เมื่อ profile นั้นๆ active
    @Bean
    @Profile("dev")
    public EmailService fakeEmailService() {
        return new FakeEmailService();  // ใช้ใน dev (ไม่ส่ง email จริง)
    }
    
    @Bean
    @Profile("!dev")  // ไม่ใช่ dev (prod, test)
    public EmailService realEmailService() {
        return new SmtpEmailService();
    }
    
    // @ConditionalOnBean - สร้าง bean เมื่อมี bean อื่นอยู่แล้ว
    @Bean
    @ConditionalOnBean(RedisConnectionFactory.class)
    public RedisCache redisCache(RedisConnectionFactory factory) {
        return new RedisCache(factory);
    }
}
```

---

## ขั้นตอนที่ 119: Bean Lifecycle

```java
@Component
@Slf4j
public class DatabaseConnection {
    
    private Connection connection;
    
    // Constructor - ถูกเรียกเมื่อ bean สร้าง
    public DatabaseConnection() {
        log.info("DatabaseConnection constructor called");
    }
    
    // @PostConstruct - ถูกเรียกหลัง DI เสร็จ
    @PostConstruct
    public void init() {
        log.info("Initializing database connection...");
        // เหมาะสำหรับ:
        // - เริ่มต้น connection
        // - Load ข้อมูล cache
        // - Validate configuration
        connection = createConnection();
    }
    
    // @PreDestroy - ถูกเรียกก่อน bean ถูก destroy
    @PreDestroy
    public void cleanup() {
        log.info("Closing database connection...");
        // เหมาะสำหรับ:
        // - ปิด connection
        // - Save state
        // - Release resources
        if (connection != null) {
            connection.close();
        }
    }
    
    private Connection createConnection() {
        // ...
    }
}
```

```java
// อีกวิธีระบุ lifecycle methods ใน @Bean
@Configuration
public class AppConfig {
    
    @Bean(initMethod = "start", destroyMethod = "stop")
    public MyService myService() {
        return new MyService();
    }
}

public class MyService {
    
    public void start() {
        // เริ่มต้น
    }
    
    public void stop() {
        // ทำความสะอาด
    }
}
```

### Bean Lifecycle Order

```
Spring Application Start
        │
        ▼
BeanDefinitionRegistryPostProcessor
        │
        ▼
BeanFactoryPostProcessor
        │
        ▼
BeanPostProcessor.postProcessBeforeInitialization()
        │
        ▼
Bean Constructor
        │
        ▼
Dependency Injection (@Autowired)
        │
        ▼
@PostConstruct
        │
        ▼
InitializingBean.afterPropertiesSet()
        │
        ▼
@Bean(initMethod = "...")
        │
        ▼
BeanPostProcessor.postProcessAfterInitialization()
        │
        ▼
Bean Ready (ApplicationContext.publishEvent)
        │
        ▼
... Application Running ...
        │
        ▼
@PreDestroy
        │
        ▼
DisposableBean.destroy()
        │
        ▼
@Bean(destroyMethod = "...")
```

---

## ขั้นตอนที่ 120: @Lazy Initialization

```java
// By default, Spring สร้าง Singleton beans ทั้งหมดตอน startup
// @Lazy ทำให้สร้างเมื่อถูกใช้งานครั้งแรก

@Service
@Lazy  // สร้างเมื่อถูกใช้ครั้งแรก
public class HeavyService {
    
    public HeavyService() {
        // Expensive initialization
        log.info("HeavyService initialized!");
    }
}

// Lazy ที่ injection point
@Service
public class MyService {
    
    @Lazy
    @Autowired
    private HeavyService heavyService;  // ไม่สร้างจนกว่าจะใช้
}

// Enable lazy initialization สำหรับทุก beans
// application.properties
// spring.main.lazy-initialization=true
```

---

## ขั้นตอนที่ 121: Circular Dependencies

```java
// ❌ Circular Dependency - Spring จะ error!
@Service
public class ServiceA {
    @Autowired
    private ServiceB serviceB;  // A depends on B
}

@Service
public class ServiceB {
    @Autowired
    private ServiceA serviceA;  // B depends on A  ← Circular!
}

// Error: The dependencies of some of the beans in the application context form a cycle:
// serviceA → serviceB → serviceA

// ✅ วิธีแก้ 1: Refactor (ดีที่สุด!)
// แยก shared logic ออกมาเป็น ServiceC
@Service
public class ServiceC {  // ← Shared logic
    // logic ที่ A และ B ต้องการ
}

@Service
public class ServiceA {
    @Autowired
    private ServiceC serviceC;
}

@Service
public class ServiceB {
    @Autowired
    private ServiceC serviceC;
}

// ✅ วิธีแก้ 2: @Lazy ที่ injection point
@Service
public class ServiceA {
    @Lazy
    @Autowired
    private ServiceB serviceB;  // ← Lazy injection
}

// ✅ วิธีแก้ 3: Setter injection
@Service
public class ServiceA {
    private ServiceB serviceB;
    
    @Autowired
    public void setServiceB(ServiceB serviceB) {
        this.serviceB = serviceB;
    }
}

// ✅ วิธีแก้ 4: ApplicationContext.getBean() (ไม่แนะนำ)
@Service
@RequiredArgsConstructor
public class ServiceA {
    private final ApplicationContext context;
    
    public void doSomething() {
        ServiceB serviceB = context.getBean(ServiceB.class);
        // ...
    }
}
```

---

## ขั้นตอนที่ 122: ApplicationContext API

```java
@Component
@RequiredArgsConstructor
public class BeanExplorer {
    
    private final ApplicationContext context;
    
    public void exploreContext() {
        // 1. Get bean by type
        UserService userService = context.getBean(UserService.class);
        
        // 2. Get bean by name
        UserService userSvc = (UserService) context.getBean("userService");
        
        // 3. Get bean by name and type (type-safe)
        UserService us = context.getBean("userService", UserService.class);
        
        // 4. Check if bean exists
        boolean exists = context.containsBean("userService");
        
        // 5. Get all bean names
        String[] beanNames = context.getBeanDefinitionNames();
        
        // 6. Get all beans of specific type
        Map<String, UserRepository> repos = context.getBeansOfType(UserRepository.class);
        
        // 7. Get beans with annotation
        Map<String, Object> services = context.getBeansWithAnnotation(Service.class);
        
        // 8. Check if bean is singleton
        boolean isSingleton = context.isSingleton("userService");
        boolean isPrototype = context.isPrototype("taskProcessor");
        
        // 9. Get application name
        String appName = context.getApplicationName();
        
        // 10. Get environment
        Environment env = context.getEnvironment();
        String dbUrl = env.getProperty("spring.datasource.url");
    }
}
```

---

## ขั้นตอนที่ 123: BeanFactory vs ApplicationContext

```
BeanFactory (Basic)           ApplicationContext (Advanced)
──────────────────────        ──────────────────────────────
- Basic DI                    - DI
- Lazy by default             - Eager by default
- No AOP support              - AOP support
- No Event support            - Event support
- Manual setup                - Auto-configuration
                              - I18n support
                              - Environment support
                              - Resource loading

ใน Spring Boot: ใช้ ApplicationContext เสมอ
BeanFactory อยู่ที่ base ของ ApplicationContext
```

---

## ขั้นตอนที่ 124: Event System

```java
// 1. สร้าง Custom Event
public class UserCreatedEvent extends ApplicationEvent {
    
    private final Long userId;
    private final String email;
    
    public UserCreatedEvent(Object source, Long userId, String email) {
        super(source);
        this.userId = userId;
        this.email = email;
    }
    
    // Getters
    public Long getUserId() { return userId; }
    public String getEmail() { return email; }
}

// 2. Publish Event
@Service
@RequiredArgsConstructor
public class UserService {
    
    private final UserRepository userRepository;
    private final ApplicationEventPublisher eventPublisher;
    
    @Transactional
    public UserResponse create(CreateUserRequest request) {
        User user = userRepository.save(userMapper.toEntity(request));
        
        // Publish event หลัง transaction commit
        eventPublisher.publishEvent(new UserCreatedEvent(this, user.getId(), user.getEmail()));
        
        return userMapper.toResponse(user);
    }
}

// 3. Listen to Event
@Component
@Slf4j
public class UserEventListener {
    
    private final EmailService emailService;
    
    // @EventListener - synchronous
    @EventListener
    public void handleUserCreated(UserCreatedEvent event) {
        log.info("User created: id={}", event.getUserId());
        emailService.sendWelcomeEmail(event.getEmail());
    }
    
    // @EventListener พร้อม condition
    @EventListener(condition = "#event.userId > 100")
    public void handleSpecialUser(UserCreatedEvent event) {
        // เฉพาะ user ที่มี id > 100
    }
    
    // @Async - asynchronous event handling
    @Async
    @EventListener
    public void handleUserCreatedAsync(UserCreatedEvent event) {
        // ทำงานใน background thread
        sendWelcomeEmailWithDelay(event.getEmail());
    }
    
    // @TransactionalEventListener - รอให้ transaction commit ก่อน
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void handleAfterCommit(UserCreatedEvent event) {
        // มั่นใจว่าข้อมูล save แล้ว จึงส่ง email
        emailService.sendWelcomeEmail(event.getEmail());
    }
}
```

---

## ขั้นตอนที่ 125: @Profile ขั้นสูง

```java
// Configuration สำหรับแต่ละ profile
@Configuration
@Profile("dev")
public class DevConfig {
    
    @Bean
    public DataSource h2DataSource() {
        return new EmbeddedDatabaseBuilder()
            .setType(EmbeddedDatabaseType.H2)
            .addScript("schema.sql")
            .addScript("data.sql")
            .build();
    }
    
    @Bean
    public EmailService fakeEmailService() {
        return email -> System.out.println("FAKE EMAIL to: " + email);
    }
}

@Configuration
@Profile("prod")
public class ProdConfig {
    
    @Bean
    public DataSource prodDataSource(
        @Value("${DATABASE_URL}") String url,
        @Value("${DATABASE_USER}") String user,
        @Value("${DATABASE_PASSWORD}") String password
    ) {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl(url);
        config.setUsername(user);
        config.setPassword(password);
        return new HikariDataSource(config);
    }
    
    @Bean
    public EmailService smtpEmailService() {
        return new SmtpEmailServiceImpl();
    }
}

// ใช้หลาย profiles
@Profile({"dev", "test"})  // dev หรือ test
public class DevTestConfig { }

@Profile("!prod")  // ทุก profile ยกเว้น prod
public class NonProdConfig { }

// Profile ที่ซ้อนกัน
@Profile("cloud & !aws")  // cloud แต่ไม่ใช่ aws
public class CloudConfig { }
```

---

## ขั้นตอนที่ 126: @Import และ @ImportResource

```java
// Import Configuration class อื่น
@SpringBootApplication
@Import({
    SecurityConfig.class,
    SwaggerConfig.class,
    CacheConfig.class
})
public class MyApp { }

// Import XML configuration (แบบเก่า)
@Configuration
@ImportResource("classpath:spring-legacy.xml")
public class LegacyConfig { }

// Import ผ่าน @EnableXxx
@SpringBootApplication
@EnableScheduling       // เปิดใช้ @Scheduled
@EnableAsync            // เปิดใช้ @Async
@EnableCaching          // เปิดใช้ @Cacheable
@EnableTransactionManagement  // เปิดใช้ @Transactional
@EnableJpaAuditing      // เปิดใช้ JPA Auditing
public class MyApp { }
```

---

## ขั้นตอนที่ 127: BeanPostProcessor

```java
// Custom BeanPostProcessor - ทำงานก่อน/หลังทุก bean ถูก initialize
@Component
@Slf4j
public class LoggingBeanPostProcessor implements BeanPostProcessor {
    
    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        log.trace("Before init: {}", beanName);
        return bean;  // return bean หรือ wrapper
    }
    
    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        log.trace("After init: {}", beanName);
        
        // ตัวอย่าง: Auto-validate configuration beans
        if (bean.getClass().isAnnotationPresent(ValidateOnStartup.class)) {
            validateBean(bean);
        }
        
        return bean;
    }
    
    private void validateBean(Object bean) {
        // validation logic
    }
}
```

---

## ขั้นตอนที่ 128: EnvironmentPostProcessor

```java
// ทำงานก่อน ApplicationContext สร้าง (ใช้สำหรับ custom property sources)
public class AppEnvironmentPostProcessor implements EnvironmentPostProcessor {
    
    @Override
    public void postProcessEnvironment(
        ConfigurableEnvironment environment,
        SpringApplication application
    ) {
        // เพิ่ม custom property source
        Map<String, Object> customProps = new HashMap<>();
        customProps.put("app.custom.property", "custom-value");
        
        environment.getPropertySources().addFirst(
            new MapPropertySource("custom", customProps)
        );
    }
}
```

```
// src/main/resources/META-INF/spring.factories (Spring Boot 2.x)
// หรือ
// src/main/resources/META-INF/spring/org.springframework.boot.env.EnvironmentPostProcessor 
// (Spring Boot 3.x)

com.example.myapp.AppEnvironmentPostProcessor
```

---

## ขั้นตอนที่ 129: Dependency Injection Best Practices

```java
// ✅ Best Practice 1: Single Responsibility Principle
// Bean ควรมี responsibility เดียว ไม่ควรมี dependencies มากเกินไป

@Service
@RequiredArgsConstructor
public class UserService {
    // ถ้ามีมากกว่า 5 dependencies อาจเป็นสัญญาณว่าควรแยก service
    private final UserRepository userRepository;
    private final PasswordEncoder passwordEncoder;
    private final UserMapper userMapper;
    private final EmailService emailService;
    // ← ถ้ามีมากกว่านี้ ควรพิจารณา refactor
}

// ✅ Best Practice 2: Program to Interface
public interface UserRepository extends JpaRepository<User, Long> { }

@Service
public class UserService {
    private final UserRepository repo;  // inject interface, not implementation
}

// ✅ Best Practice 3: Immutable Dependencies
@Service
@RequiredArgsConstructor
public class UserService {
    private final UserRepository repo;  // final = immutable after construction
}

// ✅ Best Practice 4: Use @ConfigurationProperties instead of many @Value
@Component
@ConfigurationProperties(prefix = "app.email")
@Data
public class EmailProperties {
    private String host;
    private int port;
    private String username;
    private String password;
    private boolean enabled;
}

// ✅ Best Practice 5: Keep Configuration Separate
@Configuration
public class ServiceConfig {
    
    @Bean
    public UserService userService(
        UserRepository repo,
        PasswordEncoder encoder,
        UserMapper mapper
    ) {
        return new UserService(repo, encoder, mapper);
    }
}
```

---

## ขั้นตอนที่ 130: Spring Expression Language (SpEL)

```java
// SpEL ใช้ได้ใน @Value, @ConditionalOnExpression, Thymeleaf, ฯลฯ

@Component
public class SpELExamples {
    
    // Arithmetic
    @Value("#{2 + 3}")          // 5
    private int sum;
    
    @Value("#{10 * 2.5}")       // 25.0
    private double product;
    
    // String
    @Value("#{'Hello ' + 'World'}")
    private String greeting;
    
    // Conditional
    @Value("#{2 > 1 ? 'yes' : 'no'}")
    private String result;
    
    // Access bean properties
    @Value("#{appProperties.name}")
    private String appName;
    
    // Call methods
    @Value("#{appProperties.name.toUpperCase()}")
    private String upperName;
    
    // System properties
    @Value("#{systemProperties['user.home']}")
    private String userHome;
    
    // Environment variables
    @Value("#{environment.getProperty('PATH')}")
    private String path;
    
    // Static methods
    @Value("#{T(java.lang.Math).random() * 100}")
    private double randomValue;
    
    @Value("#{T(java.time.LocalDateTime).now()}")
    private LocalDateTime now;
    
    // Regular expressions
    @Value("#{appProperties.email matches '[a-z]+@[a-z]+\\.[a-z]+'}")
    private boolean isValidEmail;
}
```

---

## ขั้นตอนที่ 131-135: Testing Dependency Injection

```java
// ======================================
// Unit Test (ไม่ใช้ Spring Context)
// ======================================
class UserServiceTest {
    
    // Mock dependencies
    @Mock
    private UserRepository userRepository;
    
    @Mock
    private EmailService emailService;
    
    @Mock
    private PasswordEncoder passwordEncoder;
    
    @InjectMocks  // สร้าง UserService และ inject mocks
    private UserService userService;
    
    @BeforeEach
    void setUp() {
        MockitoAnnotations.openMocks(this);
    }
    
    @Test
    void createUser_WhenEmailNotExists_ShouldCreateSuccessfully() {
        // Arrange
        CreateUserRequest request = new CreateUserRequest(
            "john", "john@test.com", "Password123!", "John", "Doe", null
        );
        
        User savedUser = User.builder()
            .id(1L)
            .username("john")
            .email("john@test.com")
            .build();
        
        when(userRepository.existsByEmail("john@test.com")).thenReturn(false);
        when(userRepository.existsByUsername("john")).thenReturn(false);
        when(passwordEncoder.encode(any())).thenReturn("hashed_password");
        when(userRepository.save(any())).thenReturn(savedUser);
        
        // Act
        UserResponse result = userService.create(request);
        
        // Assert
        assertNotNull(result);
        assertEquals("john", result.username());
        verify(userRepository).save(any(User.class));
        verify(emailService).sendWelcomeEmail("john@test.com");
    }
    
    @Test
    void createUser_WhenEmailExists_ShouldThrowException() {
        // Arrange
        when(userRepository.existsByEmail(any())).thenReturn(true);
        
        CreateUserRequest request = new CreateUserRequest(
            "john", "existing@test.com", "Password123!", null, null, null
        );
        
        // Act & Assert
        assertThrows(DuplicateResourceException.class, () -> userService.create(request));
        verify(userRepository, never()).save(any());
    }
}

// ======================================
// Integration Test (ใช้ Spring Context)
// ======================================
@SpringBootTest
@Transactional
class UserServiceIntegrationTest {
    
    @Autowired
    private UserService userService;
    
    @Autowired
    private UserRepository userRepository;
    
    @Test
    void createUser_ShouldPersistToDatabase() {
        // Arrange
        CreateUserRequest request = new CreateUserRequest(
            "john", "john@test.com", "Password123!", "John", "Doe", null
        );
        
        // Act
        UserResponse result = userService.create(request);
        
        // Assert
        assertNotNull(result.id());
        assertTrue(userRepository.existsByEmail("john@test.com"));
    }
}

// ======================================
// Slice Test (ทดสอบ layer เดียว)
// ======================================
@DataJpaTest  // เฉพาะ JPA layer
class UserRepositoryTest {
    
    @Autowired
    private UserRepository userRepository;
    
    @Test
    void findByEmail_WhenExists_ShouldReturnUser() {
        // Arrange
        User user = User.builder()
            .username("john")
            .email("john@test.com")
            .password("hashed")
            .build();
        userRepository.save(user);
        
        // Act
        Optional<User> found = userRepository.findByEmail("john@test.com");
        
        // Assert
        assertTrue(found.isPresent());
        assertEquals("john", found.get().getUsername());
    }
}
```

---

### สรุป Part 06

```
✅ IoC Container concept
✅ Spring Bean สร้างด้วย @Component, @Bean
✅ @ComponentScan
✅ DI แบบต่างๆ (Constructor, Field, Setter)
✅ @Autowired, @Qualifier, @Primary
✅ Bean Scopes (Singleton, Prototype, Request, Session)
✅ Prototype in Singleton ปัญหาและวิธีแก้
✅ @Value injection
✅ @Conditional annotations
✅ Bean Lifecycle (@PostConstruct, @PreDestroy)
✅ @Lazy initialization
✅ Circular dependencies
✅ ApplicationContext API
✅ Event System
✅ @Profile
✅ BeanPostProcessor
✅ SpEL
✅ Testing DI
```

---

*[← Part 05: REST API พื้นฐาน](./part-05-rest-api-basics.md) | [Part 07: Auto-configuration →](./part-07-auto-configuration.md)*

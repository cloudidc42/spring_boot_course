# Part 116: โปรเจค 76-80 — SaaS & Infrastructure Services

> **ระดับ:** โลก (World-Class) | **เวลาเรียนรู้:** 10-15 ชั่วโมง  
> **เป้าหมาย:** สร้างระบบ SaaS และ Infrastructure Services ที่ใช้งานได้จริงในระดับ Production

---

## โปรเจค 76: Multi-tenant SaaS Boilerplate

### ภาพรวมโปรเจค

ระบบ Multi-tenant SaaS Boilerplate เป็นโครงสร้างพื้นฐานสำหรับสร้างแอปพลิเคชัน SaaS ที่รองรับหลาย Tenant โดยใช้ Schema-per-Tenant approach ซึ่งแต่ละ Tenant จะมี Database Schema แยกกัน ทำให้ข้อมูลถูก isolate อย่างสมบูรณ์ ระบบนี้รวมถึงการจัดการ Tenant Registration, Billing Integration และ Admin Panel API

### โครงสร้างโปรเจค

```
multi-tenant-saas/
├── src/main/java/com/saas/
│   ├── tenant/
│   │   ├── Tenant.java
│   │   ├── TenantService.java
│   │   └── TenantController.java
│   ├── config/
│   │   ├── MultiTenantConfig.java
│   │   └── TenantContext.java
│   ├── billing/
│   │   ├── Subscription.java
│   │   └── BillingService.java
│   └── admin/
│       └── AdminController.java
```

### Entities

```java
// Tenant.java — หน่วยงาน/บริษัทที่ใช้บริการ SaaS
@Entity
@Table(name = "tenants", schema = "public")
public class Tenant {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(unique = true, nullable = false)
    private String subdomain; // tenant1.myapp.com

    @Column(unique = true, nullable = false)
    private String schemaName; // schema ใน PostgreSQL

    @Column(nullable = false)
    private String name;

    @Column(nullable = false)
    private String adminEmail;

    @Enumerated(EnumType.STRING)
    private TenantStatus status; // PENDING, ACTIVE, SUSPENDED, CANCELLED

    @Enumerated(EnumType.STRING)
    private PlanType plan; // FREE, STARTER, PROFESSIONAL, ENTERPRISE

    private LocalDateTime createdAt;
    private LocalDateTime activatedAt;
    private LocalDateTime suspendedAt;

    // จำนวน user ที่ใช้งานได้
    private Integer maxUsers;
    private Integer currentUsers;

    // Getters & Setters
}

// Subscription.java — ข้อมูลการสมัครใช้บริการและการชำระเงิน
@Entity
@Table(name = "subscriptions", schema = "public")
public class Subscription {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @OneToOne
    @JoinColumn(name = "tenant_id")
    private Tenant tenant;

    private String stripeCustomerId;
    private String stripeSubscriptionId;

    @Enumerated(EnumType.STRING)
    private PlanType plan;

    private LocalDate billingCycleStart;
    private LocalDate billingCycleEnd;
    private LocalDate nextBillingDate;

    private BigDecimal monthlyAmount;
    private String currency;

    @Enumerated(EnumType.STRING)
    private SubscriptionStatus status; // ACTIVE, PAST_DUE, CANCELLED, TRIALING

    private LocalDateTime trialEndDate;
    private LocalDateTime cancelledAt;

    // Getters & Setters
}

// TenantAwareBaseEntity.java — Base entity ที่ทุก tenant entity ต้อง extend
@MappedSuperclass
public abstract class TenantAwareBaseEntity {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;

    @PrePersist
    protected void onCreate() {
        createdAt = LocalDateTime.now();
        updatedAt = LocalDateTime.now();
    }

    @PreUpdate
    protected void onUpdate() {
        updatedAt = LocalDateTime.now();
    }
}
```

### Multi-tenant Configuration

```java
// TenantContext.java — เก็บ Tenant ปัจจุบันใน ThreadLocal
public class TenantContext {
    private static final ThreadLocal<String> CURRENT_TENANT = new ThreadLocal<>();

    public static void setCurrentTenant(String tenant) {
        CURRENT_TENANT.set(tenant);
    }

    public static String getCurrentTenant() {
        return CURRENT_TENANT.get();
    }

    public static void clear() {
        CURRENT_TENANT.remove();
    }
}

// MultiTenantDataSource.java — DataSource ที่สลับ Schema ตาม Tenant
@Component
public class MultiTenantDataSource extends AbstractRoutingDataSource {

    @Override
    protected Object determineCurrentLookupKey() {
        return TenantContext.getCurrentTenant();
    }
}

// TenantSchemaInterceptor.java — ตั้งค่า Schema ก่อน Query ทุก Query
@Component
public class TenantSchemaInterceptor implements HandlerInterceptor {

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) {
        String tenant = extractTenantFromRequest(request);
        if (tenant != null) {
            TenantContext.setCurrentTenant(tenant);
        }
        return true;
    }

    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response,
                                Object handler, Exception ex) {
        TenantContext.clear(); // สำคัญมาก! ต้อง clear หลังใช้งาน
    }

    private String extractTenantFromRequest(HttpServletRequest request) {
        // ดึง tenant จาก subdomain: tenant1.myapp.com
        String host = request.getServerName();
        String[] parts = host.split("\\.");
        if (parts.length >= 3) {
            return parts[0]; // subdomain
        }
        // หรือดึงจาก Header: X-Tenant-ID
        return request.getHeader("X-Tenant-ID");
    }
}

// MultiTenantJpaConfig.java — ตั้งค่า JPA สำหรับ Multi-tenant
@Configuration
@EnableJpaRepositories(basePackages = "com.saas")
public class MultiTenantJpaConfig {

    @Bean
    public MultiTenantConnectionProvider multiTenantConnectionProvider(DataSource dataSource) {
        return new SchemaBasedMultiTenantConnectionProvider(dataSource);
    }

    @Bean
    public CurrentTenantIdentifierResolver currentTenantIdentifierResolver() {
        return new TenantIdentifierResolver();
    }

    @Bean
    public LocalContainerEntityManagerFactoryBean entityManagerFactory(
            DataSource dataSource,
            MultiTenantConnectionProvider connectionProvider,
            CurrentTenantIdentifierResolver tenantResolver) {

        LocalContainerEntityManagerFactoryBean factory = new LocalContainerEntityManagerFactoryBean();
        factory.setDataSource(dataSource);
        factory.setPackagesToScan("com.saas");

        HibernateJpaVendorAdapter vendorAdapter = new HibernateJpaVendorAdapter();
        factory.setJpaVendorAdapter(vendorAdapter);

        Map<String, Object> properties = new HashMap<>();
        properties.put(Environment.MULTI_TENANT, MultiTenancyStrategy.SCHEMA);
        properties.put(Environment.MULTI_TENANT_CONNECTION_PROVIDER, connectionProvider);
        properties.put(Environment.MULTI_TENANT_IDENTIFIER_RESOLVER, tenantResolver);
        factory.setJpaPropertyMap(properties);

        return factory;
    }
}
```

### Service Layer

```java
// TenantService.java — จัดการการสร้างและบริหาร Tenant
@Service
@Transactional
public class TenantService {

    @Autowired
    private TenantRepository tenantRepository;

    @Autowired
    private SchemaCreationService schemaCreationService;

    @Autowired
    private BillingService billingService;

    @Autowired
    private EmailService emailService;

    // สมัคร Tenant ใหม่
    public Tenant registerTenant(TenantRegistrationRequest request) {
        // ตรวจสอบว่า subdomain ไม่ซ้ำ
        if (tenantRepository.existsBySubdomain(request.getSubdomain())) {
            throw new SubdomainAlreadyTakenException("Subdomain already taken: " + request.getSubdomain());
        }

        String schemaName = "tenant_" + request.getSubdomain().replaceAll("[^a-zA-Z0-9]", "_");

        Tenant tenant = new Tenant();
        tenant.setSubdomain(request.getSubdomain());
        tenant.setSchemaName(schemaName);
        tenant.setName(request.getCompanyName());
        tenant.setAdminEmail(request.getAdminEmail());
        tenant.setStatus(TenantStatus.PENDING);
        tenant.setPlan(request.getPlan() != null ? request.getPlan() : PlanType.FREE);
        tenant.setMaxUsers(getMaxUsersForPlan(tenant.getPlan()));
        tenant.setCurrentUsers(0);
        tenant.setCreatedAt(LocalDateTime.now());

        tenant = tenantRepository.save(tenant);

        // สร้าง Schema ใน Database
        schemaCreationService.createSchemaForTenant(schemaName);

        // สร้าง Stripe Customer สำหรับ Billing
        if (tenant.getPlan() != PlanType.FREE) {
            billingService.createStripeCustomer(tenant, request.getPaymentMethodId());
        }

        // Activate tenant
        tenant.setStatus(TenantStatus.ACTIVE);
        tenant.setActivatedAt(LocalDateTime.now());
        tenant = tenantRepository.save(tenant);

        // ส่ง Welcome Email
        emailService.sendWelcomeEmail(tenant.getAdminEmail(), tenant.getName(), tenant.getSubdomain());

        return tenant;
    }

    // อัปเกรด/ดาวน์เกรด Plan
    public Tenant changePlan(Long tenantId, PlanType newPlan) {
        Tenant tenant = tenantRepository.findById(tenantId)
                .orElseThrow(() -> new TenantNotFoundException("Tenant not found"));

        PlanType oldPlan = tenant.getPlan();
        tenant.setPlan(newPlan);
        tenant.setMaxUsers(getMaxUsersForPlan(newPlan));
        tenant = tenantRepository.save(tenant);

        // อัปเดต Stripe Subscription
        billingService.updateSubscription(tenant, newPlan);

        return tenant;
    }

    // ระงับ Tenant
    public void suspendTenant(Long tenantId, String reason) {
        Tenant tenant = tenantRepository.findById(tenantId)
                .orElseThrow(() -> new TenantNotFoundException("Tenant not found"));

        tenant.setStatus(TenantStatus.SUSPENDED);
        tenant.setSuspendedAt(LocalDateTime.now());
        tenantRepository.save(tenant);

        emailService.sendSuspensionEmail(tenant.getAdminEmail(), tenant.getName(), reason);
    }

    private int getMaxUsersForPlan(PlanType plan) {
        return switch (plan) {
            case FREE -> 5;
            case STARTER -> 25;
            case PROFESSIONAL -> 100;
            case ENTERPRISE -> Integer.MAX_VALUE;
        };
    }
}

// SchemaCreationService.java — สร้าง Database Schema สำหรับ Tenant ใหม่
@Service
public class SchemaCreationService {

    @Autowired
    private DataSource dataSource;

    @Autowired
    private ResourceLoader resourceLoader;

    public void createSchemaForTenant(String schemaName) {
        try (Connection conn = dataSource.getConnection()) {
            // สร้าง Schema
            conn.createStatement().execute("CREATE SCHEMA IF NOT EXISTS " + schemaName);

            // Run migration scripts สำหรับ tenant schema
            runMigrations(conn, schemaName);

        } catch (SQLException e) {
            throw new SchemaCreationException("Failed to create schema: " + schemaName, e);
        }
    }

    private void runMigrations(Connection conn, String schemaName) throws SQLException {
        conn.createStatement().execute("SET search_path TO " + schemaName);

        // สร้าง Tables พื้นฐานที่ทุก Tenant ต้องมี
        conn.createStatement().execute("""
            CREATE TABLE IF NOT EXISTS users (
                id BIGSERIAL PRIMARY KEY,
                email VARCHAR(255) UNIQUE NOT NULL,
                password_hash VARCHAR(255) NOT NULL,
                first_name VARCHAR(100),
                last_name VARCHAR(100),
                role VARCHAR(50) DEFAULT 'USER',
                is_active BOOLEAN DEFAULT TRUE,
                created_at TIMESTAMP DEFAULT NOW(),
                updated_at TIMESTAMP DEFAULT NOW()
            )
        """);

        conn.createStatement().execute("""
            CREATE TABLE IF NOT EXISTS user_sessions (
                id BIGSERIAL PRIMARY KEY,
                user_id BIGINT REFERENCES users(id),
                token VARCHAR(500) NOT NULL,
                expires_at TIMESTAMP NOT NULL,
                created_at TIMESTAMP DEFAULT NOW()
            )
        """);
    }

    public void deleteSchemaForTenant(String schemaName) {
        try (Connection conn = dataSource.getConnection()) {
            conn.createStatement().execute("DROP SCHEMA IF EXISTS " + schemaName + " CASCADE");
        } catch (SQLException e) {
            throw new SchemaCreationException("Failed to delete schema: " + schemaName, e);
        }
    }
}

// BillingService.java — จัดการ Billing ผ่าน Stripe
@Service
public class BillingService {

    @Value("${stripe.secret-key}")
    private String stripeSecretKey;

    @PostConstruct
    public void init() {
        Stripe.apiKey = stripeSecretKey;
    }

    public String createStripeCustomer(Tenant tenant, String paymentMethodId) {
        try {
            CustomerCreateParams params = CustomerCreateParams.builder()
                    .setEmail(tenant.getAdminEmail())
                    .setName(tenant.getName())
                    .setMetadata(Map.of("tenantId", String.valueOf(tenant.getId())))
                    .setPaymentMethod(paymentMethodId)
                    .setInvoiceSettings(
                            CustomerCreateParams.InvoiceSettings.builder()
                                    .setDefaultPaymentMethod(paymentMethodId)
                                    .build()
                    )
                    .build();

            Customer customer = Customer.create(params);
            return customer.getId();

        } catch (StripeException e) {
            throw new BillingException("Failed to create Stripe customer", e);
        }
    }

    public String createSubscription(String customerId, PlanType plan) {
        try {
            String priceId = getPriceIdForPlan(plan);

            SubscriptionCreateParams params = SubscriptionCreateParams.builder()
                    .setCustomer(customerId)
                    .addItem(SubscriptionCreateParams.Item.builder()
                            .setPrice(priceId)
                            .build())
                    .build();

            com.stripe.model.Subscription subscription =
                    com.stripe.model.Subscription.create(params);

            return subscription.getId();

        } catch (StripeException e) {
            throw new BillingException("Failed to create subscription", e);
        }
    }

    private String getPriceIdForPlan(PlanType plan) {
        return switch (plan) {
            case STARTER -> "price_starter_monthly";
            case PROFESSIONAL -> "price_professional_monthly";
            case ENTERPRISE -> "price_enterprise_monthly";
            default -> throw new IllegalArgumentException("No price for plan: " + plan);
        };
    }
}
```

### Controller Layer

```java
// TenantController.java — API สำหรับ Tenant Registration
@RestController
@RequestMapping("/api/tenants")
public class TenantController {

    @Autowired
    private TenantService tenantService;

    // สมัคร Tenant ใหม่ (Public API)
    @PostMapping("/register")
    public ResponseEntity<TenantResponse> register(@Valid @RequestBody TenantRegistrationRequest request) {
        Tenant tenant = tenantService.registerTenant(request);
        return ResponseEntity.status(HttpStatus.CREATED).body(TenantResponse.from(tenant));
    }

    // ดึงข้อมูล Tenant ปัจจุบัน (ต้องผ่าน Auth)
    @GetMapping("/current")
    @PreAuthorize("isAuthenticated()")
    public ResponseEntity<TenantResponse> getCurrentTenant() {
        String tenantId = TenantContext.getCurrentTenant();
        Tenant tenant = tenantService.getTenantBySchema(tenantId);
        return ResponseEntity.ok(TenantResponse.from(tenant));
    }

    // เปลี่ยน Plan
    @PutMapping("/{id}/plan")
    @PreAuthorize("hasRole('TENANT_ADMIN')")
    public ResponseEntity<TenantResponse> changePlan(
            @PathVariable Long id,
            @RequestBody ChangePlanRequest request) {
        Tenant tenant = tenantService.changePlan(id, request.getNewPlan());
        return ResponseEntity.ok(TenantResponse.from(tenant));
    }
}

// AdminController.java — API สำหรับ Super Admin จัดการ Tenant ทั้งหมด
@RestController
@RequestMapping("/api/admin/tenants")
@PreAuthorize("hasRole('SUPER_ADMIN')")
public class AdminController {

    @Autowired
    private TenantService tenantService;

    // ดูรายการ Tenant ทั้งหมด
    @GetMapping
    public ResponseEntity<Page<TenantResponse>> listTenants(
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size,
            @RequestParam(required = false) TenantStatus status) {
        Page<Tenant> tenants = tenantService.listTenants(page, size, status);
        return ResponseEntity.ok(tenants.map(TenantResponse::from));
    }

    // ระงับ Tenant
    @PostMapping("/{id}/suspend")
    public ResponseEntity<Void> suspend(@PathVariable Long id,
                                         @RequestBody SuspendRequest request) {
        tenantService.suspendTenant(id, request.getReason());
        return ResponseEntity.ok().build();
    }

    // ดู Statistics ทั้งหมด
    @GetMapping("/stats")
    public ResponseEntity<TenantStats> getStats() {
        return ResponseEntity.ok(tenantService.getOverallStats());
    }
}
```

### SQL Scripts

```sql
-- migrations/V1__create_public_schema.sql
-- สร้าง Tables หลักใน Public Schema

CREATE TABLE tenants (
    id BIGSERIAL PRIMARY KEY,
    subdomain VARCHAR(100) UNIQUE NOT NULL,
    schema_name VARCHAR(100) UNIQUE NOT NULL,
    name VARCHAR(255) NOT NULL,
    admin_email VARCHAR(255) NOT NULL,
    status VARCHAR(50) DEFAULT 'PENDING',
    plan VARCHAR(50) DEFAULT 'FREE',
    max_users INTEGER DEFAULT 5,
    current_users INTEGER DEFAULT 0,
    created_at TIMESTAMP DEFAULT NOW(),
    activated_at TIMESTAMP,
    suspended_at TIMESTAMP
);

CREATE TABLE subscriptions (
    id BIGSERIAL PRIMARY KEY,
    tenant_id BIGINT UNIQUE REFERENCES tenants(id),
    stripe_customer_id VARCHAR(255),
    stripe_subscription_id VARCHAR(255),
    plan VARCHAR(50),
    billing_cycle_start DATE,
    billing_cycle_end DATE,
    next_billing_date DATE,
    monthly_amount DECIMAL(10,2),
    currency VARCHAR(3) DEFAULT 'USD',
    status VARCHAR(50),
    trial_end_date TIMESTAMP,
    cancelled_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_tenants_subdomain ON tenants(subdomain);
CREATE INDEX idx_tenants_status ON tenants(status);
CREATE INDEX idx_subscriptions_status ON subscriptions(status);
```

---

## โปรเจค 77: Headless CMS

### ภาพรวมโปรเจค

Headless CMS (Content Management System) เป็นระบบจัดการเนื้อหาที่แยก Backend (Content Repository) ออกจาก Frontend (Presentation Layer) โดยสมบูรณ์ ระบบนี้รองรับ Dynamic Schema สำหรับ Content Types, การ Versioning, Media Library, Webhooks เมื่อ Publish และ Preview Mode สำหรับดูเนื้อหาก่อน Publish

### Entities

```java
// ContentType.java — กำหนดโครงสร้างของ Content เช่น Blog Post, Product, Page
@Entity
@Table(name = "content_types")
public class ContentType {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(unique = true, nullable = false)
    private String apiId; // blog-post, product, page (ใช้ใน API URL)

    @Column(nullable = false)
    private String displayName; // "Blog Post", "Product"

    private String description;

    // JSON ที่เก็บ Field Definitions: [{name, type, required, ...}]
    @Column(columnDefinition = "jsonb")
    private String fields;

    private boolean publishable; // มี draft/published state ไหม
    private boolean versionable; // เก็บ Version History ไหม

    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
}

// ContentEntry.java — เนื้อหาจริงของแต่ละ Content Type
@Entity
@Table(name = "content_entries")
public class ContentEntry {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne
    @JoinColumn(name = "content_type_id")
    private ContentType contentType;

    // ข้อมูลจริงในรูป JSON ตาม Schema ของ ContentType
    @Column(columnDefinition = "jsonb", nullable = false)
    private String data;

    @Enumerated(EnumType.STRING)
    private EntryStatus status; // DRAFT, PUBLISHED, ARCHIVED

    private String slug; // URL-friendly identifier

    private Long currentVersion;
    private Long publishedVersion;

    @ManyToOne
    @JoinColumn(name = "created_by")
    private User createdBy;

    @ManyToOne
    @JoinColumn(name = "updated_by")
    private User updatedBy;

    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
    private LocalDateTime publishedAt;
    private LocalDateTime scheduledAt; // สำหรับ Scheduled Publishing
}

// ContentVersion.java — เก็บประวัติการเปลี่ยนแปลง
@Entity
@Table(name = "content_versions")
public class ContentVersion {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne
    @JoinColumn(name = "entry_id")
    private ContentEntry entry;

    private Long versionNumber;

    @Column(columnDefinition = "jsonb")
    private String data; // snapshot ของ data ณ เวลานั้น

    private String changeSummary;

    @ManyToOne
    @JoinColumn(name = "created_by")
    private User createdBy;

    private LocalDateTime createdAt;
}

// MediaAsset.java — ไฟล์รูปภาพและสื่อต่างๆ
@Entity
@Table(name = "media_assets")
public class MediaAsset {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String filename;

    private String originalFilename;
    private String mimeType;
    private Long fileSize;

    // S3/Storage URL
    private String url;
    private String thumbnailUrl;

    // Image dimensions
    private Integer width;
    private Integer height;

    // Alternative text สำหรับ accessibility
    private String altText;
    private String caption;

    // Tags สำหรับ search
    @ElementCollection
    @CollectionTable(name = "media_tags")
    private Set<String> tags = new HashSet<>();

    @ManyToOne
    @JoinColumn(name = "uploaded_by")
    private User uploadedBy;

    private LocalDateTime createdAt;
}

// Webhook.java — กำหนด Webhook ที่จะ trigger เมื่อมี Event
@Entity
@Table(name = "webhooks")
public class Webhook {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String name;

    @Column(nullable = false)
    private String url;

    // Secret สำหรับ verify webhook signature
    private String secret;

    // Events ที่ต้อง trigger เช่น ["entry.publish", "entry.create"]
    @ElementCollection
    @CollectionTable(name = "webhook_events")
    private Set<String> events = new HashSet<>();

    private boolean active;
    private LocalDateTime createdAt;
}
```

### Service Layer

```java
// ContentTypeService.java — จัดการ Content Type Definitions
@Service
@Transactional
public class ContentTypeService {

    @Autowired
    private ContentTypeRepository contentTypeRepository;

    @Autowired
    private ObjectMapper objectMapper;

    // สร้าง Content Type ใหม่ พร้อม Field Definitions
    public ContentType createContentType(CreateContentTypeRequest request) {
        if (contentTypeRepository.existsByApiId(request.getApiId())) {
            throw new ContentTypeAlreadyExistsException("Content type already exists: " + request.getApiId());
        }

        // Validate field definitions
        validateFieldDefinitions(request.getFields());

        ContentType contentType = new ContentType();
        contentType.setApiId(request.getApiId());
        contentType.setDisplayName(request.getDisplayName());
        contentType.setDescription(request.getDescription());
        contentType.setPublishable(request.isPublishable());
        contentType.setVersionable(request.isVersionable());

        try {
            contentType.setFields(objectMapper.writeValueAsString(request.getFields()));
        } catch (JsonProcessingException e) {
            throw new InvalidFieldDefinitionException("Invalid field definitions");
        }

        contentType.setCreatedAt(LocalDateTime.now());
        contentType.setUpdatedAt(LocalDateTime.now());

        return contentTypeRepository.save(contentType);
    }

    private void validateFieldDefinitions(List<FieldDefinition> fields) {
        Set<String> fieldNames = new HashSet<>();
        for (FieldDefinition field : fields) {
            if (!fieldNames.add(field.getName())) {
                throw new InvalidFieldDefinitionException("Duplicate field name: " + field.getName());
            }
            // Validate field type
            if (!VALID_FIELD_TYPES.contains(field.getType())) {
                throw new InvalidFieldDefinitionException("Invalid field type: " + field.getType());
            }
        }
    }
}

// ContentEntryService.java — จัดการ Content Entries
@Service
@Transactional
public class ContentEntryService {

    @Autowired
    private ContentEntryRepository entryRepository;

    @Autowired
    private ContentVersionRepository versionRepository;

    @Autowired
    private ContentTypeRepository contentTypeRepository;

    @Autowired
    private WebhookService webhookService;

    @Autowired
    private ObjectMapper objectMapper;

    // สร้าง Entry ใหม่
    public ContentEntry createEntry(String contentTypeApiId, Map<String, Object> data, User user) {
        ContentType contentType = contentTypeRepository.findByApiId(contentTypeApiId)
                .orElseThrow(() -> new ContentTypeNotFoundException("Content type not found: " + contentTypeApiId));

        // Validate data ตาม field definitions
        validateEntryData(contentType, data);

        ContentEntry entry = new ContentEntry();
        entry.setContentType(contentType);
        entry.setStatus(EntryStatus.DRAFT);
        entry.setCurrentVersion(1L);
        entry.setCreatedBy(user);
        entry.setUpdatedBy(user);
        entry.setCreatedAt(LocalDateTime.now());
        entry.setUpdatedAt(LocalDateTime.now());

        // Generate slug จาก title field (ถ้ามี)
        if (data.containsKey("title")) {
            entry.setSlug(generateSlug(data.get("title").toString()));
        }

        try {
            entry.setData(objectMapper.writeValueAsString(data));
        } catch (JsonProcessingException e) {
            throw new InvalidEntryDataException("Invalid entry data");
        }

        entry = entryRepository.save(entry);

        // สร้าง Version snapshot
        if (contentType.isVersionable()) {
            saveVersion(entry, data, user, "Initial version");
        }

        return entry;
    }

    // Publish entry
    public ContentEntry publishEntry(Long entryId, User user) {
        ContentEntry entry = entryRepository.findById(entryId)
                .orElseThrow(() -> new EntryNotFoundException("Entry not found"));

        if (entry.getStatus() == EntryStatus.PUBLISHED) {
            return entry; // already published
        }

        entry.setStatus(EntryStatus.PUBLISHED);
        entry.setPublishedVersion(entry.getCurrentVersion());
        entry.setPublishedAt(LocalDateTime.now());
        entry.setUpdatedBy(user);
        entry = entryRepository.save(entry);

        // Trigger webhooks
        webhookService.triggerEvent("entry.publish", entry);

        return entry;
    }

    // ย้อนกลับไป Version ก่อนหน้า
    public ContentEntry revertToVersion(Long entryId, Long versionNumber, User user) {
        ContentEntry entry = entryRepository.findById(entryId)
                .orElseThrow(() -> new EntryNotFoundException("Entry not found"));

        ContentVersion version = versionRepository.findByEntryAndVersionNumber(entry, versionNumber)
                .orElseThrow(() -> new VersionNotFoundException("Version not found"));

        entry.setData(version.getData());
        entry.setCurrentVersion(entry.getCurrentVersion() + 1);
        entry.setUpdatedBy(user);
        entry.setUpdatedAt(LocalDateTime.now());
        entry.setStatus(EntryStatus.DRAFT); // revert จะกลับเป็น draft เสมอ

        entry = entryRepository.save(entry);

        // สร้าง Version ใหม่สำหรับ revert
        saveVersion(entry, null, user, "Reverted to version " + versionNumber);

        return entry;
    }

    private void saveVersion(ContentEntry entry, Map<String, Object> data, User user, String summary) {
        ContentVersion version = new ContentVersion();
        version.setEntry(entry);
        version.setVersionNumber(entry.getCurrentVersion());
        version.setData(entry.getData());
        version.setChangeSummary(summary);
        version.setCreatedBy(user);
        version.setCreatedAt(LocalDateTime.now());
        versionRepository.save(version);
    }

    private String generateSlug(String title) {
        return title.toLowerCase()
                .replaceAll("[^a-z0-9\\s-]", "")
                .replaceAll("\\s+", "-")
                .replaceAll("-+", "-")
                .trim();
    }
}

// WebhookService.java — ส่ง Webhook Events
@Service
public class WebhookService {

    @Autowired
    private WebhookRepository webhookRepository;

    @Autowired
    private RestTemplate restTemplate;

    @Autowired
    private ObjectMapper objectMapper;

    @Async
    public void triggerEvent(String eventType, Object payload) {
        List<Webhook> webhooks = webhookRepository.findByEventsContainingAndActive(eventType, true);

        for (Webhook webhook : webhooks) {
            try {
                sendWebhook(webhook, eventType, payload);
            } catch (Exception e) {
                // Log error แต่ไม่ fail main flow
                log.error("Failed to deliver webhook to {}: {}", webhook.getUrl(), e.getMessage());
            }
        }
    }

    private void sendWebhook(Webhook webhook, String eventType, Object payload) throws Exception {
        Map<String, Object> body = new HashMap<>();
        body.put("event", eventType);
        body.put("timestamp", Instant.now().toString());
        body.put("data", payload);

        String bodyJson = objectMapper.writeValueAsString(body);
        String signature = generateSignature(webhook.getSecret(), bodyJson);

        HttpHeaders headers = new HttpHeaders();
        headers.setContentType(MediaType.APPLICATION_JSON);
        headers.set("X-CMS-Signature", signature);
        headers.set("X-CMS-Event", eventType);

        HttpEntity<String> entity = new HttpEntity<>(bodyJson, headers);
        restTemplate.postForEntity(webhook.getUrl(), entity, String.class);
    }

    private String generateSignature(String secret, String body) throws Exception {
        Mac mac = Mac.getInstance("HmacSHA256");
        SecretKeySpec keySpec = new SecretKeySpec(secret.getBytes(), "HmacSHA256");
        mac.init(keySpec);
        byte[] hash = mac.doFinal(body.getBytes());
        return "sha256=" + HexFormat.of().formatHex(hash);
    }
}
```

### Controller Layer

```java
// ContentDeliveryController.java — Public API สำหรับ Frontend ดึงเนื้อหา
@RestController
@RequestMapping("/api/content")
public class ContentDeliveryController {

    @Autowired
    private ContentEntryService entryService;

    // ดึง Published Entries ของ Content Type นั้นๆ
    @GetMapping("/{contentType}")
    public ResponseEntity<Page<Map<String, Object>>> getEntries(
            @PathVariable String contentType,
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "10") int size,
            @RequestParam(required = false) String q,
            @RequestParam(required = false) String orderBy,
            @RequestParam(required = false) String preview) {

        boolean isPreview = "true".equals(preview);
        Page<ContentEntry> entries = entryService.getPublishedEntries(contentType, page, size, q, isPreview);
        return ResponseEntity.ok(entries.map(e -> entryService.parseEntryData(e)));
    }

    // ดึง Entry เดียวด้วย slug
    @GetMapping("/{contentType}/{slug}")
    public ResponseEntity<Map<String, Object>> getEntry(
            @PathVariable String contentType,
            @PathVariable String slug) {
        ContentEntry entry = entryService.getPublishedEntryBySlug(contentType, slug);
        return ResponseEntity.ok(entryService.parseEntryData(entry));
    }
}

// ContentManagementController.java — API สำหรับ CMS Admin จัดการเนื้อหา
@RestController
@RequestMapping("/api/cms")
@PreAuthorize("hasRole('EDITOR')")
public class ContentManagementController {

    @Autowired
    private ContentEntryService entryService;

    @PostMapping("/content-types/{contentType}/entries")
    public ResponseEntity<ContentEntryResponse> createEntry(
            @PathVariable String contentType,
            @RequestBody Map<String, Object> data,
            @AuthenticationPrincipal User user) {
        ContentEntry entry = entryService.createEntry(contentType, data, user);
        return ResponseEntity.status(HttpStatus.CREATED).body(ContentEntryResponse.from(entry));
    }

    @PostMapping("/entries/{id}/publish")
    public ResponseEntity<ContentEntryResponse> publish(
            @PathVariable Long id,
            @AuthenticationPrincipal User user) {
        ContentEntry entry = entryService.publishEntry(id, user);
        return ResponseEntity.ok(ContentEntryResponse.from(entry));
    }

    @GetMapping("/entries/{id}/versions")
    public ResponseEntity<List<ContentVersionResponse>> getVersions(@PathVariable Long id) {
        List<ContentVersion> versions = entryService.getVersionHistory(id);
        return ResponseEntity.ok(versions.stream().map(ContentVersionResponse::from).toList());
    }

    @PostMapping("/entries/{id}/revert/{version}")
    public ResponseEntity<ContentEntryResponse> revert(
            @PathVariable Long id,
            @PathVariable Long version,
            @AuthenticationPrincipal User user) {
        ContentEntry entry = entryService.revertToVersion(id, version, user);
        return ResponseEntity.ok(ContentEntryResponse.from(entry));
    }
}
```

### SQL Scripts

```sql
-- migrations/V1__cms_schema.sql
CREATE TABLE content_types (
    id BIGSERIAL PRIMARY KEY,
    api_id VARCHAR(100) UNIQUE NOT NULL,
    display_name VARCHAR(255) NOT NULL,
    description TEXT,
    fields JSONB DEFAULT '[]',
    publishable BOOLEAN DEFAULT TRUE,
    versionable BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE content_entries (
    id BIGSERIAL PRIMARY KEY,
    content_type_id BIGINT REFERENCES content_types(id),
    data JSONB NOT NULL DEFAULT '{}',
    status VARCHAR(50) DEFAULT 'DRAFT',
    slug VARCHAR(500),
    current_version BIGINT DEFAULT 1,
    published_version BIGINT,
    created_by BIGINT,
    updated_by BIGINT,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    published_at TIMESTAMP,
    scheduled_at TIMESTAMP
);

CREATE TABLE content_versions (
    id BIGSERIAL PRIMARY KEY,
    entry_id BIGINT REFERENCES content_entries(id) ON DELETE CASCADE,
    version_number BIGINT NOT NULL,
    data JSONB NOT NULL,
    change_summary TEXT,
    created_by BIGINT,
    created_at TIMESTAMP DEFAULT NOW(),
    UNIQUE(entry_id, version_number)
);

CREATE TABLE media_assets (
    id BIGSERIAL PRIMARY KEY,
    filename VARCHAR(500) NOT NULL,
    original_filename VARCHAR(500),
    mime_type VARCHAR(100),
    file_size BIGINT,
    url TEXT NOT NULL,
    thumbnail_url TEXT,
    width INTEGER,
    height INTEGER,
    alt_text VARCHAR(500),
    caption TEXT,
    uploaded_by BIGINT,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE webhooks (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    url TEXT NOT NULL,
    secret VARCHAR(255),
    active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_entries_type_status ON content_entries(content_type_id, status);
CREATE INDEX idx_entries_slug ON content_entries(slug);
CREATE INDEX idx_entries_data_gin ON content_entries USING gin(data);
```

---

## โปรเจค 78: Product Catalog Service

### ภาพรวมโปรเจค

Product Catalog Service เป็นระบบจัดการข้อมูลสินค้าครบวงจร รองรับ Products, Variants (ขนาด/สี), Attributes แบบ Dynamic, Pricing Rules, Bundles, Collections และ Import/Export ผ่าน CSV พร้อม Sync ข้อมูลไปยัง Elasticsearch สำหรับการค้นหาที่รวดเร็ว

### Entities

```java
// Product.java — ข้อมูลสินค้าหลัก
@Entity
@Table(name = "products")
public class Product {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(unique = true, nullable = false)
    private String sku; // Stock Keeping Unit

    @Column(nullable = false)
    private String name;

    private String description;
    private String shortDescription;

    @Column(nullable = false)
    private BigDecimal basePrice;

    private BigDecimal compareAtPrice; // ราคาก่อนลด (สำหรับแสดง strikethrough)
    private BigDecimal costPrice; // ราคาต้นทุน

    @ManyToOne
    @JoinColumn(name = "category_id")
    private Category category;

    @ManyToMany
    @JoinTable(name = "product_tags",
            joinColumns = @JoinColumn(name = "product_id"),
            inverseJoinColumns = @JoinColumn(name = "tag_id"))
    private Set<Tag> tags = new HashSet<>();

    @OneToMany(mappedBy = "product", cascade = CascadeType.ALL)
    private List<ProductVariant> variants = new ArrayList<>();

    @OneToMany(mappedBy = "product", cascade = CascadeType.ALL)
    private List<ProductImage> images = new ArrayList<>();

    @ElementCollection
    @CollectionTable(name = "product_attributes")
    @MapKeyColumn(name = "attr_key")
    @Column(name = "attr_value")
    private Map<String, String> attributes = new HashMap<>();

    @Enumerated(EnumType.STRING)
    private ProductStatus status; // DRAFT, ACTIVE, ARCHIVED

    private boolean featured;
    private Integer stockQuantity;
    private Integer lowStockThreshold;

    private Float weight;
    private String weightUnit;

    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
}

// ProductVariant.java — Variant ของสินค้า เช่น เสื้อสีแดงไซส์ L
@Entity
@Table(name = "product_variants")
public class ProductVariant {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne
    @JoinColumn(name = "product_id")
    private Product product;

    @Column(unique = true, nullable = false)
    private String variantSku;

    // Options เช่น {color: "Red", size: "L"}
    @ElementCollection
    @CollectionTable(name = "variant_options")
    @MapKeyColumn(name = "option_key")
    @Column(name = "option_value")
    private Map<String, String> options = new HashMap<>();

    private BigDecimal priceAdjustment; // ราคาเพิ่ม/ลดจาก base price
    private Integer stockQuantity;
    private String imageUrl;
    private boolean active;
}

// PricingRule.java — กฎการคำนวณราคา เช่น ส่วนลดตามปริมาณ
@Entity
@Table(name = "pricing_rules")
public class PricingRule {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String name;

    @Enumerated(EnumType.STRING)
    private RuleType type; // PERCENTAGE_DISCOUNT, FIXED_DISCOUNT, QUANTITY_BASED, BUNDLE

    // Conditions เช่น {"minQuantity": 10, "customerGroup": "WHOLESALE"}
    @Column(columnDefinition = "jsonb")
    private String conditions;

    // Actions เช่น {"discountPercent": 15}
    @Column(columnDefinition = "jsonb")
    private String actions;

    private Integer priority; // ลำดับความสำคัญเมื่อมีหลาย rule
    private LocalDate startDate;
    private LocalDate endDate;
    private boolean active;
}

// Bundle.java — กลุ่มสินค้าที่ขายร่วมกัน
@Entity
@Table(name = "bundles")
public class Bundle {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String name;

    private String description;
    private BigDecimal bundlePrice; // ราคาพิเศษสำหรับ bundle
    private BigDecimal regularPrice; // ราคาปกติถ้าซื้อแยก

    @OneToMany(mappedBy = "bundle", cascade = CascadeType.ALL)
    private List<BundleItem> items = new ArrayList<>();

    private boolean active;
    private LocalDateTime createdAt;
}
```

### Service Layer

```java
// ProductService.java — จัดการข้อมูลสินค้า
@Service
@Transactional
public class ProductService {

    @Autowired
    private ProductRepository productRepository;

    @Autowired
    private ProductSearchService searchService;

    @Autowired
    private CsvImportService csvImportService;

    // สร้างสินค้าใหม่
    public Product createProduct(CreateProductRequest request) {
        if (productRepository.existsBySku(request.getSku())) {
            throw new DuplicateSkuException("SKU already exists: " + request.getSku());
        }

        Product product = new Product();
        product.setSku(request.getSku());
        product.setName(request.getName());
        product.setDescription(request.getDescription());
        product.setBasePrice(request.getBasePrice());
        product.setStatus(ProductStatus.DRAFT);
        product.setCreatedAt(LocalDateTime.now());
        product.setUpdatedAt(LocalDateTime.now());

        // สร้าง default variant ถ้าไม่มี variants
        if (request.getVariants() == null || request.getVariants().isEmpty()) {
            ProductVariant defaultVariant = new ProductVariant();
            defaultVariant.setProduct(product);
            defaultVariant.setVariantSku(request.getSku() + "-DEFAULT");
            defaultVariant.setStockQuantity(request.getStockQuantity() != null ? request.getStockQuantity() : 0);
            defaultVariant.setActive(true);
            product.getVariants().add(defaultVariant);
        }

        product = productRepository.save(product);

        // Sync to Elasticsearch
        searchService.indexProduct(product);

        return product;
    }

    // Import products จาก CSV
    public ImportResult importFromCsv(MultipartFile file) throws IOException {
        List<CsvProductRow> rows = csvImportService.parseCsv(file.getInputStream());

        int created = 0, updated = 0, failed = 0;
        List<String> errors = new ArrayList<>();

        for (CsvProductRow row : rows) {
            try {
                if (productRepository.existsBySku(row.getSku())) {
                    updateProductFromCsv(row);
                    updated++;
                } else {
                    createProductFromCsv(row);
                    created++;
                }
            } catch (Exception e) {
                failed++;
                errors.add("Row " + row.getRowNumber() + ": " + e.getMessage());
            }
        }

        return new ImportResult(created, updated, failed, errors);
    }

    // Export products เป็น CSV
    public byte[] exportToCsv(ProductFilter filter) throws IOException {
        List<Product> products = productRepository.findAll(buildSpec(filter));

        StringWriter sw = new StringWriter();
        CSVPrinter printer = new CSVPrinter(sw, CSVFormat.DEFAULT.withHeader(
                "SKU", "Name", "Price", "Stock", "Status", "Category"));

        for (Product p : products) {
            printer.printRecord(
                    p.getSku(),
                    p.getName(),
                    p.getBasePrice(),
                    p.getStockQuantity(),
                    p.getStatus(),
                    p.getCategory() != null ? p.getCategory().getName() : ""
            );
        }

        printer.flush();
        return sw.toString().getBytes(StandardCharsets.UTF_8);
    }

    // คำนวณราคาจริงหลังจาก Pricing Rules
    public BigDecimal calculateFinalPrice(Product product, int quantity, String customerGroup) {
        List<PricingRule> applicableRules = pricingRuleRepository.findApplicableRules(
                product.getId(), quantity, customerGroup, LocalDate.now());

        BigDecimal price = product.getBasePrice();

        // เรียง rules ตาม priority
        applicableRules.sort(Comparator.comparing(PricingRule::getPriority));

        for (PricingRule rule : applicableRules) {
            price = applyRule(price, rule, quantity);
        }

        return price.max(BigDecimal.ZERO); // ราคาต้องไม่ติดลบ
    }
}

// ProductSearchService.java — Sync กับ Elasticsearch
@Service
public class ProductSearchService {

    @Autowired
    private ElasticsearchOperations elasticsearchOps;

    // Index สินค้าไปยัง Elasticsearch
    public void indexProduct(Product product) {
        ProductDocument doc = ProductDocument.builder()
                .id(product.getId())
                .sku(product.getSku())
                .name(product.getName())
                .description(product.getDescription())
                .price(product.getBasePrice().doubleValue())
                .category(product.getCategory() != null ? product.getCategory().getName() : null)
                .status(product.getStatus().name())
                .attributes(product.getAttributes())
                .tags(product.getTags().stream().map(Tag::getName).collect(Collectors.toSet()))
                .build();

        elasticsearchOps.save(doc);
    }

    // ค้นหาสินค้าแบบ Full-text
    public SearchHits<ProductDocument> search(String query, Map<String, String> filters, Pageable pageable) {
        NativeQuery nativeQuery = NativeQuery.builder()
                .withQuery(q -> q
                        .bool(b -> b
                                .must(m -> m.multiMatch(mm -> mm
                                        .query(query)
                                        .fields(List.of("name^3", "description", "sku^2", "tags"))
                                ))
                                .filter(buildFilters(filters))
                        )
                )
                .withPageable(pageable)
                .build();

        return elasticsearchOps.search(nativeQuery, ProductDocument.class);
    }
}
```

### Controller Layer

```java
@RestController
@RequestMapping("/api/catalog")
public class ProductCatalogController {

    @Autowired
    private ProductService productService;

    @GetMapping("/products")
    public ResponseEntity<Page<ProductResponse>> list(
            @RequestParam(required = false) String q,
            @RequestParam(required = false) Long categoryId,
            @RequestParam(required = false) BigDecimal minPrice,
            @RequestParam(required = false) BigDecimal maxPrice,
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size) {
        ProductFilter filter = new ProductFilter(q, categoryId, minPrice, maxPrice);
        Page<Product> products = productService.findProducts(filter, PageRequest.of(page, size));
        return ResponseEntity.ok(products.map(ProductResponse::from));
    }

    @PostMapping("/products")
    @PreAuthorize("hasRole('CATALOG_MANAGER')")
    public ResponseEntity<ProductResponse> create(@Valid @RequestBody CreateProductRequest request) {
        Product product = productService.createProduct(request);
        return ResponseEntity.status(HttpStatus.CREATED).body(ProductResponse.from(product));
    }

    @PostMapping("/products/import")
    @PreAuthorize("hasRole('CATALOG_MANAGER')")
    public ResponseEntity<ImportResult> importCsv(@RequestParam("file") MultipartFile file) throws IOException {
        return ResponseEntity.ok(productService.importFromCsv(file));
    }

    @GetMapping("/products/export")
    @PreAuthorize("hasRole('CATALOG_MANAGER')")
    public ResponseEntity<byte[]> exportCsv(ProductFilter filter) throws IOException {
        byte[] csv = productService.exportToCsv(filter);
        return ResponseEntity.ok()
                .header(HttpHeaders.CONTENT_DISPOSITION, "attachment; filename=products.csv")
                .contentType(MediaType.parseMediaType("text/csv"))
                .body(csv);
    }
}
```

---

## โปรเจค 79: Mini Search Engine

### ภาพรวมโปรเจค

Mini Search Engine เป็นระบบค้นหาเอกสารที่ใช้ Elasticsearch เป็น Backend รองรับ Full-text Search, Faceted Filters, Autocomplete, Spelling Correction, Relevance Tuning และ Search Analytics เพื่อติดตามพฤติกรรมการค้นหา

### Entities & Elasticsearch Documents

```java
// SearchDocument.java — Document ที่ถูก Index ใน Elasticsearch
@Document(indexName = "documents")
public class SearchDocument {
    @Id
    private String id;

    @Field(type = FieldType.Text, analyzer = "english")
    private String title;

    @Field(type = FieldType.Text, analyzer = "english")
    private String content;

    @Field(type = FieldType.Keyword)
    private String category;

    @Field(type = FieldType.Keyword)
    private List<String> tags;

    @Field(type = FieldType.Date)
    private LocalDateTime publishedAt;

    @Field(type = FieldType.Text, analyzer = "autocomplete", searchAnalyzer = "standard")
    private String titleSuggest; // สำหรับ autocomplete

    @Field(type = FieldType.Float)
    private Float boostScore; // manual boost

    @Field(type = FieldType.Keyword)
    private String sourceUrl;
}

// SearchQuery.java — เก็บ log การค้นหาสำหรับ Analytics
@Entity
@Table(name = "search_queries")
public class SearchQuery {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String query;

    private Integer resultsCount;
    private Boolean hasResults;
    private Long responseTimeMs;

    // Click data
    private String clickedDocumentId;
    private Integer clickPosition;

    private String userId;
    private String sessionId;
    private String ipAddress;

    private LocalDateTime searchedAt;
}
```

### Search Service

```java
// SearchService.java — ระบบค้นหาหลัก
@Service
public class SearchService {

    @Autowired
    private ElasticsearchOperations elasticsearchOps;

    @Autowired
    private SearchQueryRepository queryRepository;

    // Full-text search พร้อม facets
    public SearchResult search(SearchRequest request) {
        long startTime = System.currentTimeMillis();

        NativeQuery query = buildSearchQuery(request);

        SearchHits<SearchDocument> hits = elasticsearchOps.search(query, SearchDocument.class);

        long responseTime = System.currentTimeMillis() - startTime;

        // บันทึก Analytics
        logSearchQuery(request.getQuery(), (int) hits.getTotalHits(), responseTime, request.getSessionId());

        // สร้าง facets จาก aggregations
        Map<String, List<FacetValue>> facets = extractFacets(hits);

        return SearchResult.builder()
                .hits(hits.getSearchHits().stream()
                        .map(h -> SearchHitResponse.from(h))
                        .collect(Collectors.toList()))
                .totalHits(hits.getTotalHits())
                .facets(facets)
                .responseTimeMs(responseTime)
                .build();
    }

    private NativeQuery buildSearchQuery(SearchRequest request) {
        BoolQuery.Builder boolQuery = new BoolQuery.Builder();

        // Main search query
        if (StringUtils.hasText(request.getQuery())) {
            boolQuery.must(m -> m
                    .multiMatch(mm -> mm
                            .query(request.getQuery())
                            .fields(List.of("title^4", "content^2", "tags^3"))
                            .fuzziness("AUTO") // spelling correction
                            .type(TextQueryType.BestFields)
                    )
            );
        } else {
            boolQuery.must(m -> m.matchAll(ma -> ma));
        }

        // Filters
        if (request.getCategory() != null) {
            boolQuery.filter(f -> f.term(t -> t.field("category").value(request.getCategory())));
        }

        if (request.getTags() != null && !request.getTags().isEmpty()) {
            boolQuery.filter(f -> f.terms(t -> t
                    .field("tags")
                    .terms(tv -> tv.value(request.getTags().stream()
                            .map(FieldValue::of).collect(Collectors.toList())))
            ));
        }

        if (request.getDateFrom() != null) {
            boolQuery.filter(f -> f.range(r -> r
                    .field("publishedAt")
                    .gte(JsonData.of(request.getDateFrom().toString()))
            ));
        }

        // Build aggregations for facets
        return NativeQuery.builder()
                .withQuery(q -> q.bool(boolQuery.build()))
                .withAggregation("categories", a -> a.terms(t -> t.field("category").size(20)))
                .withAggregation("tags", a -> a.terms(t -> t.field("tags").size(50)))
                .withPageable(PageRequest.of(request.getPage(), request.getSize()))
                .withHighlightQuery(buildHighlightQuery())
                .build();
    }

    // Autocomplete suggestions
    public List<String> autocomplete(String prefix) {
        NativeQuery query = NativeQuery.builder()
                .withQuery(q -> q
                        .match(m -> m
                                .field("titleSuggest")
                                .query(prefix)
                        )
                )
                .withPageable(PageRequest.of(0, 10))
                .build();

        SearchHits<SearchDocument> hits = elasticsearchOps.search(query, SearchDocument.class);

        return hits.getSearchHits().stream()
                .map(h -> h.getContent().getTitle())
                .distinct()
                .collect(Collectors.toList());
    }

    // Spelling suggestions โดยใช้ Elasticsearch Suggest API
    public List<String> spellingSuggestions(String query) {
        // ใช้ term suggester ของ Elasticsearch
        SuggestQuery suggestQuery = SuggestQuery.builder()
                .withSuggester(s -> s
                        .term(t -> t
                                .field("content")
                                .suggestMode(SuggestMode.Missing)
                        )
                        .text(query)
                )
                .build();

        SearchHits<SearchDocument> result = elasticsearchOps.search(suggestQuery, SearchDocument.class);
        // Extract suggestions...
        return extractSuggestions(result);
    }

    private void logSearchQuery(String query, int resultsCount, long responseTime, String sessionId) {
        SearchQuery log = new SearchQuery();
        log.setQuery(query);
        log.setResultsCount(resultsCount);
        log.setHasResults(resultsCount > 0);
        log.setResponseTimeMs(responseTime);
        log.setSessionId(sessionId);
        log.setSearchedAt(LocalDateTime.now());
        queryRepository.save(log);
    }
}

// DocumentIndexingService.java — จัดการการ Index เอกสาร
@Service
public class DocumentIndexingService {

    @Autowired
    private ElasticsearchOperations elasticsearchOps;

    // Index เอกสารใหม่
    public void indexDocument(DocumentIndexRequest request) {
        SearchDocument doc = new SearchDocument();
        doc.setId(request.getId());
        doc.setTitle(request.getTitle());
        doc.setContent(request.getContent());
        doc.setCategory(request.getCategory());
        doc.setTags(request.getTags());
        doc.setPublishedAt(request.getPublishedAt());
        doc.setTitleSuggest(request.getTitle()); // ใช้สำหรับ autocomplete
        doc.setBoostScore(1.0f); // default boost
        doc.setSourceUrl(request.getSourceUrl());

        elasticsearchOps.save(doc);
    }

    // Bulk index หลายเอกสารพร้อมกัน
    public BulkIndexResult bulkIndex(List<DocumentIndexRequest> requests) {
        List<IndexQuery> queries = requests.stream()
                .map(req -> {
                    SearchDocument doc = mapToDocument(req);
                    return new IndexQueryBuilder()
                            .withId(doc.getId())
                            .withObject(doc)
                            .build();
                })
                .collect(Collectors.toList());

        List<IndexedObjectInformation> results = elasticsearchOps.bulkIndex(queries,
                IndexCoordinates.of("documents"));

        return new BulkIndexResult(results.size(), 0);
    }

    // Reindex ทั้งหมด (สำหรับ schema changes)
    @Async
    public void reindexAll() {
        // ลบ index เก่า
        elasticsearchOps.indexOps(SearchDocument.class).delete();

        // สร้าง index ใหม่พร้อม settings
        elasticsearchOps.indexOps(SearchDocument.class).createWithMapping();

        // Reindex จาก database
        // ...
    }
}
```

### Controller

```java
@RestController
@RequestMapping("/api/search")
public class SearchController {

    @Autowired
    private SearchService searchService;

    @Autowired
    private DocumentIndexingService indexingService;

    @GetMapping
    public ResponseEntity<SearchResult> search(
            @RequestParam String q,
            @RequestParam(required = false) String category,
            @RequestParam(required = false) List<String> tags,
            @RequestParam(required = false) String dateFrom,
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "10") int size) {

        SearchRequest request = SearchRequest.builder()
                .query(q).category(category).tags(tags)
                .page(page).size(size).build();

        return ResponseEntity.ok(searchService.search(request));
    }

    @GetMapping("/autocomplete")
    public ResponseEntity<List<String>> autocomplete(@RequestParam String q) {
        return ResponseEntity.ok(searchService.autocomplete(q));
    }

    @GetMapping("/suggest")
    public ResponseEntity<List<String>> suggest(@RequestParam String q) {
        return ResponseEntity.ok(searchService.spellingSuggestions(q));
    }

    @PostMapping("/index")
    @PreAuthorize("hasRole('INDEXER')")
    public ResponseEntity<Void> index(@RequestBody DocumentIndexRequest request) {
        indexingService.indexDocument(request);
        return ResponseEntity.ok().build();
    }

    @PostMapping("/index/bulk")
    @PreAuthorize("hasRole('INDEXER')")
    public ResponseEntity<BulkIndexResult> bulkIndex(@RequestBody List<DocumentIndexRequest> requests) {
        return ResponseEntity.ok(indexingService.bulkIndex(requests));
    }

    // Track clicks สำหรับ CTR analytics
    @PostMapping("/click")
    public ResponseEntity<Void> trackClick(@RequestBody ClickTrackingRequest request) {
        searchService.trackClick(request.getDocumentId(), request.getQuery(),
                request.getPosition(), request.getSessionId());
        return ResponseEntity.ok().build();
    }
}
```

---

## โปรเจค 80: Recommendation Engine

### ภาพรวมโปรเจค

Recommendation Engine เป็นระบบแนะนำสินค้า/เนื้อหาโดยใช้ Collaborative Filtering ทั้งแบบ User-based และ Item-based รวมถึง Content-based Filtering และ Hybrid Recommendations มีระบบ A/B Testing สำหรับทดสอบ Algorithm ต่างๆ และ Click-through Rate Tracking

### Entities

```java
// UserInteraction.java — เก็บ Interaction ของ User กับ Item
@Entity
@Table(name = "user_interactions")
public class UserInteraction {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private Long userId;

    @Column(nullable = false)
    private Long itemId;

    @Column(nullable = false)
    private String itemType; // product, article, video

    @Enumerated(EnumType.STRING)
    private InteractionType type; // VIEW, CLICK, PURCHASE, RATING, BOOKMARK

    private Float rating; // 1-5 สำหรับ explicit feedback
    private Integer viewDurationSeconds;

    private LocalDateTime interactedAt;
    private String sessionId;
}

// RecommendationModel.java — เก็บ trained model metadata
@Entity
@Table(name = "recommendation_models")
public class RecommendationModel {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String name;

    @Enumerated(EnumType.STRING)
    private ModelType type; // USER_BASED_CF, ITEM_BASED_CF, CONTENT_BASED, HYBRID

    @Enumerated(EnumType.STRING)
    private ModelStatus status; // TRAINING, ACTIVE, DEPRECATED

    private String modelPath; // path ไป serialized model
    private Float accuracy;

    private LocalDateTime trainedAt;
    private LocalDateTime activatedAt;
}

// ABTestExperiment.java — A/B Test สำหรับทดสอบ Algorithm
@Entity
@Table(name = "ab_test_experiments")
public class ABTestExperiment {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String name;

    private String description;

    // Traffic split: {control: 50, treatment: 50}
    @Column(columnDefinition = "jsonb")
    private String trafficSplit;

    // Variants: {control: "item_based_cf", treatment: "hybrid"}
    @Column(columnDefinition = "jsonb")
    private String variants;

    @Enumerated(EnumType.STRING)
    private ExperimentStatus status; // DRAFT, RUNNING, PAUSED, COMPLETED

    private LocalDateTime startDate;
    private LocalDateTime endDate;
}
```

### Recommendation Services

```java
// CollaborativeFilteringService.java — User-based และ Item-based CF
@Service
public class CollaborativeFilteringService {

    @Autowired
    private UserInteractionRepository interactionRepository;

    @Autowired
    private RedisTemplate<String, Object> redisTemplate;

    // User-based Collaborative Filtering
    // หาผู้ใช้ที่คล้ายกัน แล้วแนะนำสิ่งที่พวกเขาชอบ
    public List<Long> userBasedRecommendations(Long userId, int topN) {
        // ดึง Interaction Matrix จาก Cache หรือ DB
        Map<Long, Map<Long, Float>> userItemMatrix = getUserItemMatrix();

        Map<Long, Float> targetUserRatings = userItemMatrix.get(userId);
        if (targetUserRatings == null || targetUserRatings.isEmpty()) {
            return popularItems(topN); // Cold start: แนะนำ popular items
        }

        // คำนวณ Cosine Similarity กับ Users คนอื่น
        Map<Long, Double> userSimilarities = new HashMap<>();

        for (Map.Entry<Long, Map<Long, Float>> entry : userItemMatrix.entrySet()) {
            Long otherUserId = entry.getKey();
            if (otherUserId.equals(userId)) continue;

            double similarity = cosineSimilarity(targetUserRatings, entry.getValue());
            if (similarity > 0) {
                userSimilarities.put(otherUserId, similarity);
            }
        }

        // เรียงตาม similarity
        List<Long> similarUsers = userSimilarities.entrySet().stream()
                .sorted(Map.Entry.<Long, Double>comparingByValue().reversed())
                .limit(50) // Top 50 similar users
                .map(Map.Entry::getKey)
                .collect(Collectors.toList());

        // หา Items ที่ Similar Users ชอบ แต่ Target User ยังไม่เคยดู
        Set<Long> seenItems = new HashSet<>(targetUserRatings.keySet());

        Map<Long, Double> itemScores = new HashMap<>();
        for (Long similarUserId : similarUsers) {
            double similarity = userSimilarities.get(similarUserId);
            Map<Long, Float> similarUserRatings = userItemMatrix.get(similarUserId);

            for (Map.Entry<Long, Float> item : similarUserRatings.entrySet()) {
                if (!seenItems.contains(item.getKey())) {
                    itemScores.merge(item.getKey(),
                            similarity * item.getValue(),
                            Double::sum);
                }
            }
        }

        return itemScores.entrySet().stream()
                .sorted(Map.Entry.<Long, Double>comparingByValue().reversed())
                .limit(topN)
                .map(Map.Entry::getKey)
                .collect(Collectors.toList());
    }

    // Item-based Collaborative Filtering
    // หา Items ที่คล้ายกัน แนะนำตาม Items ที่ User เคย interact
    public List<Long> itemBasedRecommendations(Long userId, int topN) {
        List<UserInteraction> userInteractions = interactionRepository.findByUserIdOrderByInteractedAtDesc(userId);

        if (userInteractions.isEmpty()) {
            return popularItems(topN);
        }

        // ดึง Item Similarity Matrix (pre-computed)
        Map<Long, Double> candidateScores = new HashMap<>();

        // สำหรับ Item ที่ User เคย interact ล่าสุด
        List<Long> recentItems = userInteractions.stream()
                .limit(20)
                .map(UserInteraction::getItemId)
                .collect(Collectors.toList());

        for (Long itemId : recentItems) {
            // ดึง Similar Items จาก pre-computed matrix (Redis)
            List<ItemSimilarity> similarItems = getItemSimilarities(itemId);

            for (ItemSimilarity sim : similarItems) {
                if (!recentItems.contains(sim.getItemId())) {
                    candidateScores.merge(sim.getItemId(), sim.getSimilarity(), Double::sum);
                }
            }
        }

        return candidateScores.entrySet().stream()
                .sorted(Map.Entry.<Long, Double>comparingByValue().reversed())
                .limit(topN)
                .map(Map.Entry::getKey)
                .collect(Collectors.toList());
    }

    // คำนวณ Cosine Similarity
    private double cosineSimilarity(Map<Long, Float> a, Map<Long, Float> b) {
        Set<Long> commonItems = new HashSet<>(a.keySet());
        commonItems.retainAll(b.keySet());

        if (commonItems.isEmpty()) return 0.0;

        double dotProduct = 0, normA = 0, normB = 0;
        for (Long item : commonItems) {
            dotProduct += a.get(item) * b.get(item);
        }
        for (float v : a.values()) normA += v * v;
        for (float v : b.values()) normB += v * v;

        return dotProduct / (Math.sqrt(normA) * Math.sqrt(normB));
    }
}

// HybridRecommendationService.java — รวม Algorithms หลายตัว
@Service
public class HybridRecommendationService {

    @Autowired
    private CollaborativeFilteringService cfService;

    @Autowired
    private ContentBasedFilteringService contentService;

    @Autowired
    private ABTestService abTestService;

    // Hybrid Recommendations: รวม CF + Content-based
    public List<RecommendedItem> getRecommendations(Long userId, String itemType, int limit) {
        // ตรวจสอบว่า User อยู่ใน A/B Test group ไหน
        String algorithm = abTestService.getAlgorithmForUser(userId, "recommendation_algorithm");

        List<Long> itemIds;

        switch (algorithm) {
            case "user_cf" -> itemIds = cfService.userBasedRecommendations(userId, limit * 2);
            case "item_cf" -> itemIds = cfService.itemBasedRecommendations(userId, limit * 2);
            case "content" -> itemIds = contentService.getRecommendations(userId, limit * 2);
            default -> { // hybrid (default)
                List<Long> cfRecs = cfService.itemBasedRecommendations(userId, limit);
                List<Long> contentRecs = contentService.getRecommendations(userId, limit);
                itemIds = mergeRecommendations(cfRecs, contentRecs, limit * 2);
            }
        }

        // ดึงข้อมูล Items และสร้าง Response
        return buildRecommendedItems(itemIds, algorithm, limit);
    }

    // รวม Recommendations จาก 2 sources โดยให้น้ำหนัก CF = 60%, Content = 40%
    private List<Long> mergeRecommendations(List<Long> cfRecs, List<Long> contentRecs, int limit) {
        Map<Long, Double> scores = new LinkedHashMap<>();

        double cfWeight = 0.6;
        double contentWeight = 0.4;

        for (int i = 0; i < cfRecs.size(); i++) {
            double score = cfWeight * (1.0 - (double) i / cfRecs.size());
            scores.merge(cfRecs.get(i), score, Double::sum);
        }

        for (int i = 0; i < contentRecs.size(); i++) {
            double score = contentWeight * (1.0 - (double) i / contentRecs.size());
            scores.merge(contentRecs.get(i), score, Double::sum);
        }

        return scores.entrySet().stream()
                .sorted(Map.Entry.<Long, Double>comparingByValue().reversed())
                .limit(limit)
                .map(Map.Entry::getKey)
                .collect(Collectors.toList());
    }
}

// RecommendationController.java
@RestController
@RequestMapping("/api/recommendations")
public class RecommendationController {

    @Autowired
    private HybridRecommendationService recommendationService;

    @GetMapping("/users/{userId}")
    public ResponseEntity<List<RecommendedItem>> getRecommendations(
            @PathVariable Long userId,
            @RequestParam(defaultValue = "product") String itemType,
            @RequestParam(defaultValue = "10") int limit) {
        return ResponseEntity.ok(recommendationService.getRecommendations(userId, itemType, limit));
    }

    @PostMapping("/interactions")
    public ResponseEntity<Void> trackInteraction(@RequestBody InteractionRequest request) {
        recommendationService.trackInteraction(request);
        return ResponseEntity.ok().build();
    }

    @PostMapping("/click")
    public ResponseEntity<Void> trackClick(@RequestBody ClickRequest request) {
        recommendationService.trackClick(request.getUserId(), request.getItemId(),
                request.getExperimentId(), request.getVariant());
        return ResponseEntity.ok().build();
    }
}
```

### SQL Scripts

```sql
-- migrations/V1__recommendation_engine.sql
CREATE TABLE user_interactions (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL,
    item_id BIGINT NOT NULL,
    item_type VARCHAR(50) NOT NULL DEFAULT 'product',
    type VARCHAR(50) NOT NULL,
    rating FLOAT,
    view_duration_seconds INTEGER,
    interacted_at TIMESTAMP DEFAULT NOW(),
    session_id VARCHAR(255)
);

CREATE TABLE recommendation_models (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    type VARCHAR(50) NOT NULL,
    status VARCHAR(50) DEFAULT 'TRAINING',
    model_path VARCHAR(500),
    accuracy FLOAT,
    trained_at TIMESTAMP,
    activated_at TIMESTAMP
);

CREATE TABLE ab_test_experiments (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    traffic_split JSONB,
    variants JSONB,
    status VARCHAR(50) DEFAULT 'DRAFT',
    start_date TIMESTAMP,
    end_date TIMESTAMP
);

CREATE TABLE ab_test_assignments (
    user_id BIGINT NOT NULL,
    experiment_id BIGINT REFERENCES ab_test_experiments(id),
    variant VARCHAR(100) NOT NULL,
    assigned_at TIMESTAMP DEFAULT NOW(),
    PRIMARY KEY (user_id, experiment_id)
);

CREATE TABLE recommendation_clicks (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL,
    item_id BIGINT NOT NULL,
    experiment_id BIGINT,
    variant VARCHAR(100),
    position INTEGER,
    clicked_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_interactions_user_id ON user_interactions(user_id);
CREATE INDEX idx_interactions_item_id ON user_interactions(item_id);
CREATE INDEX idx_interactions_user_item ON user_interactions(user_id, item_id);
```

---

## สรุป Part 116

ใน Part นี้เราได้เรียนรู้การสร้าง SaaS & Infrastructure Services ระดับ Production:

| โปรเจค | เทคโนโลยีหลัก | ความยาก |
|--------|---------------|---------|
| 76. Multi-tenant SaaS | Schema-per-tenant, Stripe API | ⭐⭐⭐⭐⭐ |
| 77. Headless CMS | Dynamic schema, Webhooks, Versioning | ⭐⭐⭐⭐ |
| 78. Product Catalog | Elasticsearch sync, CSV Import/Export | ⭐⭐⭐⭐ |
| 79. Mini Search Engine | Elasticsearch, Facets, Autocomplete | ⭐⭐⭐⭐ |
| 80. Recommendation Engine | Collaborative Filtering, A/B Testing | ⭐⭐⭐⭐⭐ |

### Key Takeaways

1. **Multi-tenancy** — Schema isolation ให้ความปลอดภัยสูงสุด แต่ต้องบริหาร database connections อย่างระมัดระวัง
2. **Headless CMS** — Dynamic schemas ช่วยให้ flexible แต่ต้องมี validation layer ที่แข็งแกร่ง
3. **Elasticsearch** — เหมาะสำหรับ full-text search และ analytics แต่ต้องระวัง data consistency กับ primary DB
4. **Recommendation Systems** — Cold start problem เป็นความท้าทายหลัก ต้องมี fallback strategy

*[← Part 115](./part-115-advanced-microservices.md) | [Part 117 →](./part-117-media-entertainment.md)*

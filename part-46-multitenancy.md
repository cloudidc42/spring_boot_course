# Part 46: Multi-Tenancy
## ขั้นตอนที่ 1441-1480

> **ระดับ:** ระดับโลก (World-Class)  
> **เวลาเรียน:** 7-8 ชั่วโมง  
> **เป้าหมาย:** Build SaaS platform ที่รองรับหลาย tenant

---

## ขั้นตอนที่ 1441: Multi-Tenancy Concepts

```
Multi-Tenancy = Multiple customers (tenants) sharing same application

Strategies:
  1. Database per tenant
     ✅ Best isolation
     ✅ Easy per-tenant backups
     ❌ Expensive (1 DB per customer)
     ❌ Schema migration complexity
  
  2. Schema per tenant
     ✅ Good isolation
     ✅ Single DB instance
     ❌ Limited number of schemas
     ❌ Complex connection management
  
  3. Row-level isolation (Discriminator)
     ✅ Simple
     ✅ Easy to manage
     ❌ Less isolation
     ❌ All tenants see same table (filtered by tenant_id)
     
Use case:
  Database per tenant → Enterprise (high security)
  Schema per tenant   → Mid-market SaaS
  Row-level          → Simple SaaS with many small tenants
```

---

## ขั้นตอนที่ 1442: Row-Level Multi-Tenancy

```java
// TenantContext - thread-local tenant storage
public class TenantContext {
    
    private static final ThreadLocal<String> CURRENT_TENANT = new ThreadLocal<>();
    
    public static void setCurrentTenant(String tenantId) {
        CURRENT_TENANT.set(tenantId);
    }
    
    public static String getCurrentTenant() {
        return CURRENT_TENANT.get();
    }
    
    public static void clear() {
        CURRENT_TENANT.remove();
    }
}

// Extract tenant from JWT or subdomain
@Component
@RequiredArgsConstructor
public class TenantFilter extends OncePerRequestFilter {
    
    private final JwtService jwtService;
    
    @Override
    protected void doFilterInternal(
        HttpServletRequest request,
        HttpServletResponse response,
        FilterChain filterChain
    ) throws ServletException, IOException {
        
        String tenantId = extractTenant(request);
        
        if (tenantId != null) {
            TenantContext.setCurrentTenant(tenantId);
        }
        
        try {
            filterChain.doFilter(request, response);
        } finally {
            TenantContext.clear();
        }
    }
    
    private String extractTenant(HttpServletRequest request) {
        // Option 1: Subdomain (tenant1.myapp.com)
        String host = request.getServerName();
        if (host.contains(".")) {
            return host.split("\\.")[0];
        }
        
        // Option 2: Header
        String tenantHeader = request.getHeader("X-Tenant-ID");
        if (tenantHeader != null) return tenantHeader;
        
        // Option 3: JWT claim
        String token = extractToken(request);
        if (token != null) {
            return jwtService.extractClaim(token, claims -> claims.get("tenantId", String.class));
        }
        
        return null;
    }
}
```

---

## ขั้นตอนที่ 1443: BaseEntity with Tenant

```java
@MappedSuperclass
@EntityListeners(TenantEntityListener.class)
@Getter @Setter
public abstract class TenantAwareEntity {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(name = "tenant_id", nullable = false)
    private String tenantId;
    
    @CreationTimestamp
    @Column(name = "created_at", updatable = false)
    private LocalDateTime createdAt;
    
    @UpdateTimestamp
    @Column(name = "updated_at")
    private LocalDateTime updatedAt;
}

@Component
public class TenantEntityListener {
    
    @PrePersist
    public void setTenant(TenantAwareEntity entity) {
        if (entity.getTenantId() == null) {
            String tenantId = TenantContext.getCurrentTenant();
            if (tenantId == null) {
                throw new TenantContextMissingException("No tenant context set");
            }
            entity.setTenantId(tenantId);
        }
    }
}

// Product entity with tenant
@Entity
@Table(name = "products")
@FilterDef(name = "tenantFilter", parameters = @ParamDef(name = "tenantId", type = String.class))
@Filter(name = "tenantFilter", condition = "tenant_id = :tenantId")
public class Product extends TenantAwareEntity {
    private String name;
    private BigDecimal price;
    // ...
}
```

---

## ขั้นตอนที่ 1444: Hibernate Tenant Filter

```java
@Configuration
public class HibernateConfig {
    
    @Bean
    public HibernatePropertiesCustomizer hibernatePropertiesCustomizer() {
        return properties -> {
            properties.put("hibernate.session_factory.statement_inspector",
                (StatementInspector) sql -> {
                    // Log SQL with tenant context
                    return sql;
                });
        };
    }
}

// Apply filter automatically using AOP
@Aspect
@Component
@RequiredArgsConstructor
public class TenantFilterAspect {
    
    private final EntityManager em;
    
    @Around("@annotation(transactional)")
    public Object applyTenantFilter(
        ProceedingJoinPoint pjp,
        Transactional transactional
    ) throws Throwable {
        String tenantId = TenantContext.getCurrentTenant();
        
        if (tenantId != null) {
            Session session = em.unwrap(Session.class);
            Filter filter = session.enableFilter("tenantFilter");
            filter.setParameter("tenantId", tenantId);
        }
        
        try {
            return pjp.proceed();
        } finally {
            if (tenantId != null) {
                em.unwrap(Session.class).disableFilter("tenantFilter");
            }
        }
    }
}
```

---

## ขั้นตอนที่ 1445: Schema-per-Tenant

```java
// MultiTenantConnectionProvider
@Component
@RequiredArgsConstructor
public class SchemaBasedMultiTenantConnectionProvider 
    implements MultiTenantConnectionProvider<String> {
    
    private final DataSource dataSource;
    
    @Override
    public Connection getConnection(String tenantId) throws SQLException {
        Connection conn = dataSource.getConnection();
        conn.setSchema(tenantId);
        return conn;
    }
    
    @Override
    public void releaseConnection(String tenantId, Connection conn) throws SQLException {
        conn.setSchema("public");  // Reset to default
        conn.close();
    }
    
    @Override
    public boolean supportsAggressiveRelease() { return false; }
    
    @Override
    public Connection getAnyConnection() throws SQLException {
        return dataSource.getConnection();
    }
    
    @Override
    public void releaseAnyConnection(Connection conn) throws SQLException {
        conn.close();
    }
}

// TenantIdentifierResolver
@Component
public class TenantIdentifierResolver implements CurrentTenantIdentifierResolver<String> {
    
    @Override
    public String resolveCurrentTenantIdentifier() {
        String tenantId = TenantContext.getCurrentTenant();
        return tenantId != null ? tenantId : "public";
    }
    
    @Override
    public boolean validateExistingCurrentSessions() {
        return false;
    }
}

// JPA Config
@Configuration
public class MultiTenantJpaConfig {
    
    @Bean
    public LocalContainerEntityManagerFactoryBean entityManagerFactory(
        DataSource dataSource,
        SchemaBasedMultiTenantConnectionProvider connectionProvider,
        TenantIdentifierResolver tenantResolver
    ) {
        Map<String, Object> props = new HashMap<>();
        props.put(MULTI_TENANT_CONNECTION_PROVIDER, connectionProvider);
        props.put(MULTI_TENANT_IDENTIFIER_RESOLVER, tenantResolver);
        props.put(MULTI_TENANT, MultiTenancyStrategy.SCHEMA);
        
        LocalContainerEntityManagerFactoryBean factory = new LocalContainerEntityManagerFactoryBean();
        factory.setDataSource(dataSource);
        factory.setJpaPropertyMap(props);
        factory.setPackagesToScan("com.myapp");
        factory.setPersistenceProvider(new HibernatePersistenceProvider());
        
        return factory;
    }
}
```

---

## ขั้นตอนที่ 1446: Tenant Provisioning

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class TenantProvisioningService {
    
    private final DataSource dataSource;
    private final Flyway flyway;
    
    @Transactional
    public void createTenant(String tenantId) {
        log.info("Creating tenant schema: {}", tenantId);
        
        try (Connection conn = dataSource.getConnection()) {
            // Create schema
            conn.createStatement().execute("CREATE SCHEMA IF NOT EXISTS " + tenantId);
            
            // Run Flyway migrations on new schema
            Flyway tenantFlyway = Flyway.configure()
                .dataSource(dataSource)
                .schemas(tenantId)
                .locations("classpath:db/migration/tenant")
                .load();
            
            tenantFlyway.migrate();
            
            log.info("Tenant {} created successfully", tenantId);
        } catch (SQLException e) {
            throw new RuntimeException("Failed to create tenant: " + tenantId, e);
        }
    }
    
    @Transactional
    public void deleteTenant(String tenantId) {
        try (Connection conn = dataSource.getConnection()) {
            conn.createStatement().execute("DROP SCHEMA IF EXISTS " + tenantId + " CASCADE");
            log.info("Tenant {} deleted", tenantId);
        } catch (SQLException e) {
            throw new RuntimeException("Failed to delete tenant: " + tenantId, e);
        }
    }
}

// Tenant registration flow
@RestController
@RequestMapping("/api/v1/tenants")
@RequiredArgsConstructor
public class TenantController {
    
    private final TenantProvisioningService provisioningService;
    private final TenantRepository tenantRepository;
    
    @PostMapping("/register")
    public ResponseEntity<TenantResponse> register(@Valid @RequestBody CreateTenantRequest request) {
        // Create tenant record
        Tenant tenant = Tenant.builder()
            .slug(request.slug())
            .name(request.name())
            .email(request.email())
            .plan(Plan.FREE)
            .build();
        
        Tenant saved = tenantRepository.save(tenant);
        
        // Provision schema
        provisioningService.createTenant(saved.getSlug());
        
        return ResponseEntity.status(201).body(tenantMapper.toResponse(saved));
    }
}
```

---

## ขั้นตอนที่ 1447-1480: Tenant-Aware Services

```java
// All services automatically filtered by tenant
@Service
@RequiredArgsConstructor
@Transactional(readOnly = true)
public class ProductService {
    
    private final ProductRepository repository;
    
    // Returns ONLY products for current tenant (filter applied automatically)
    public Page<Product> findAll(Pageable pageable) {
        return repository.findAll(pageable);
    }
    
    @Transactional
    public Product create(CreateProductRequest request) {
        // tenantId set automatically by TenantEntityListener
        return repository.save(Product.builder()
            .name(request.name())
            .price(request.price())
            .build());
    }
}

// Multi-tenancy configuration summary
/*
  Row-level:
  ✅ Simple to implement
  ✅ Single database
  ❌ Data isolation via code (risk of bugs)
  
  Schema-per-tenant:
  ✅ True DB-level isolation
  ✅ Per-tenant backup possible
  ❌ Schema management complexity
  ❌ Limited by max schemas
  
  Database-per-tenant:
  ✅ Full isolation
  ✅ Best for regulated industries
  ❌ Expensive
  ❌ Complex connection pooling
  
  Best practices:
  1. Always validate tenant context
  2. Never trust client-provided tenant ID without validation
  3. Audit all cross-tenant operations
  4. Test tenant isolation thoroughly
*/
```

---

*[← Part 45: Security](./part-45-security.md) | [Part 47: Spring Batch Advanced →](./part-47-batch.md)*

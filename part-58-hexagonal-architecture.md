# Part 58: Hexagonal Architecture
## ขั้นตอนที่ 1921-1960

**ระดับ:** ระดับโลก (World-Class)
**เวลาเรียน:** 6-7 ชั่วโมง
**เป้าหมาย:** เข้าใจ Hexagonal Architecture (Ports & Adapters) และสามารถออกแบบระบบที่ Testable, Maintainable และ Framework-Independent

---

## ขั้นตอนที่ 1921-1925: Hexagonal Architecture คืออะไร?

### แนวคิดพื้นฐาน

Hexagonal Architecture หรือ Ports & Adapters Pattern ถูกคิดค้นโดย Alistair Cockburn ในปี 2005 หลักการคือ:

- **Core Domain** (Application) ไม่ขึ้นอยู่กับ Framework, Database, หรือ UI
- **Ports** คือ interfaces ที่ Core Domain กำหนด
- **Adapters** คือ implementations ที่เชื่อมต่อโลกภายนอกกับ Core

```
                    ┌─────────────────────────────────────┐
   REST Client ──── │ REST Adapter (Primary)               │
   CLI Tool    ──── │ CLI Adapter (Primary)                │
                    │                                      │
                    │    ┌──────────────────────────┐      │
                    │    │    Application Core      │      │
                    │    │  (Domain Logic)          │      │
                    │    │                          │      │
                    │    │  Ports (Interfaces)      │      │
                    │    └──────────────────────────┘      │
                    │                                      │
                    │ JPA Adapter (Secondary)  ──── DB     │
                    │ Email Adapter (Secondary) ─── SMTP   │
                    │ Redis Adapter (Secondary) ─── Cache  │
                    └─────────────────────────────────────┘
```

### Primary vs Secondary Adapters

| ประเภท | คำอธิบาย | ตัวอย่าง |
|--------|-----------|---------|
| **Primary (Driving)** | เรียกใช้ Application | REST Controller, CLI, Scheduler |
| **Secondary (Driven)** | ถูกเรียกโดย Application | Database, Email, Message Queue |

### โครงสร้าง Project

```
product-catalog-service/
├── src/main/java/com/example/catalog/
│   ├── domain/                          # Core Domain
│   │   ├── model/
│   │   │   ├── Product.java
│   │   │   ├── Category.java
│   │   │   └── Price.java
│   │   ├── service/
│   │   │   └── ProductService.java      # Application Service
│   │   └── port/
│   │       ├── in/                      # Primary Ports (Use Cases)
│   │       │   ├── CreateProductUseCase.java
│   │       │   ├── GetProductQuery.java
│   │       │   └── UpdatePriceUseCase.java
│   │       └── out/                     # Secondary Ports
│   │           ├── ProductRepository.java
│   │           ├── EmailNotificationPort.java
│   │           └── PricingServicePort.java
│   └── adapter/
│       ├── in/                          # Primary Adapters
│       │   ├── rest/
│       │   │   ├── ProductController.java
│       │   │   └── ProductRequest.java
│       │   └── cli/
│       │       └── ProductCliAdapter.java
│       └── out/                         # Secondary Adapters
│           ├── persistence/
│           │   ├── JpaProductRepository.java
│           │   ├── ProductJpaEntity.java
│           │   └── ProductMapper.java
│           ├── email/
│           │   └── SmtpEmailAdapter.java
│           └── pricing/
│               └── ExternalPricingAdapter.java
```

---

## ขั้นตอนที่ 1926-1933: Core Domain Model

### Domain Entities

```java
// Product.java - Domain Entity (ไม่มี Framework dependency!)
public class Product {
    
    private final ProductId id;
    private String name;
    private String description;
    private Price price;
    private Category category;
    private boolean active;
    private Instant createdAt;
    private Instant updatedAt;
    
    // Factory method - Business logic ใน domain
    public static Product create(String name, String description, 
                                  Price price, Category category) {
        if (name == null || name.isBlank()) {
            throw new InvalidProductException("Product name cannot be blank");
        }
        if (price == null || price.isNegative()) {
            throw new InvalidProductException("Price cannot be negative");
        }
        
        Product product = new Product();
        product.id = ProductId.generate();
        product.name = name;
        product.description = description;
        product.price = price;
        product.category = category;
        product.active = true;
        product.createdAt = Instant.now();
        product.updatedAt = Instant.now();
        
        return product;
    }
    
    // Domain methods
    public void updatePrice(Price newPrice, String reason) {
        if (newPrice == null || newPrice.isNegative()) {
            throw new InvalidProductException("New price cannot be negative");
        }
        
        Price oldPrice = this.price;
        this.price = newPrice;
        this.updatedAt = Instant.now();
        
        // Domain event
        DomainEvents.raise(new PriceChangedEvent(this.id, oldPrice, newPrice, reason));
    }
    
    public void deactivate() {
        if (!this.active) {
            throw new ProductAlreadyDeactivatedException(this.id.toString());
        }
        this.active = false;
        this.updatedAt = Instant.now();
        
        DomainEvents.raise(new ProductDeactivatedEvent(this.id));
    }
    
    public boolean isAvailableForPurchase() {
        return this.active && this.price != null && this.price.isPositive();
    }
    
    // Getters
    public ProductId getId() { return id; }
    public String getName() { return name; }
    public String getDescription() { return description; }
    public Price getPrice() { return price; }
    public Category getCategory() { return category; }
    public boolean isActive() { return active; }
    public Instant getCreatedAt() { return createdAt; }
    public Instant getUpdatedAt() { return updatedAt; }
}
```

```java
// Price.java - Value Object
public final class Price {
    
    private final BigDecimal amount;
    private final Currency currency;
    
    private Price(BigDecimal amount, Currency currency) {
        this.amount = amount;
        this.currency = currency;
    }
    
    public static Price of(BigDecimal amount, Currency currency) {
        if (amount == null) throw new IllegalArgumentException("Amount cannot be null");
        if (currency == null) throw new IllegalArgumentException("Currency cannot be null");
        return new Price(amount, currency);
    }
    
    public static Price thb(BigDecimal amount) {
        return new Price(amount, Currency.getInstance("THB"));
    }
    
    public boolean isNegative() {
        return amount.compareTo(BigDecimal.ZERO) < 0;
    }
    
    public boolean isPositive() {
        return amount.compareTo(BigDecimal.ZERO) > 0;
    }
    
    public Price add(Price other) {
        if (!this.currency.equals(other.currency)) {
            throw new CurrencyMismatchException(this.currency, other.currency);
        }
        return new Price(this.amount.add(other.amount), this.currency);
    }
    
    public Price multiply(int quantity) {
        return new Price(this.amount.multiply(BigDecimal.valueOf(quantity)), this.currency);
    }
    
    public BigDecimal getAmount() { return amount; }
    public Currency getCurrency() { return currency; }
    
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Price)) return false;
        Price price = (Price) o;
        return Objects.equals(amount, price.amount) && 
               Objects.equals(currency, price.currency);
    }
    
    @Override
    public int hashCode() {
        return Objects.hash(amount, currency);
    }
    
    @Override
    public String toString() {
        return amount + " " + currency.getCurrencyCode();
    }
}
```

```java
// ProductId.java - Value Object
public final class ProductId {
    
    private final String value;
    
    private ProductId(String value) {
        this.value = value;
    }
    
    public static ProductId generate() {
        return new ProductId(UUID.randomUUID().toString());
    }
    
    public static ProductId of(String value) {
        if (value == null || value.isBlank()) {
            throw new IllegalArgumentException("ProductId cannot be blank");
        }
        return new ProductId(value);
    }
    
    public String getValue() { return value; }
    
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof ProductId)) return false;
        ProductId productId = (ProductId) o;
        return Objects.equals(value, productId.value);
    }
    
    @Override
    public int hashCode() { return Objects.hash(value); }
    
    @Override
    public String toString() { return value; }
}
```

---

## ขั้นตอนที่ 1934-1938: Primary Ports (Use Cases)

```java
// CreateProductUseCase.java - Primary Port
public interface CreateProductUseCase {
    
    ProductId createProduct(CreateProductCommand command);
    
    record CreateProductCommand(
        String name,
        String description,
        BigDecimal priceAmount,
        String currencyCode,
        String categoryName
    ) {
        public CreateProductCommand {
            Objects.requireNonNull(name, "Name is required");
            Objects.requireNonNull(priceAmount, "Price is required");
        }
    }
}
```

```java
// GetProductQuery.java - Primary Port (Query side)
public interface GetProductQuery {
    
    Optional<ProductView> getProduct(ProductId productId);
    
    List<ProductView> getProductsByCategory(String category);
    
    Page<ProductView> searchProducts(ProductSearchCriteria criteria, Pageable pageable);
    
    record ProductView(
        String id,
        String name,
        String description,
        BigDecimal price,
        String currency,
        String category,
        boolean active,
        Instant createdAt
    ) {}
    
    record ProductSearchCriteria(
        String keyword,
        String category,
        BigDecimal minPrice,
        BigDecimal maxPrice,
        boolean activeOnly
    ) {}
}
```

```java
// UpdatePriceUseCase.java - Primary Port
public interface UpdatePriceUseCase {
    
    void updatePrice(UpdatePriceCommand command);
    
    record UpdatePriceCommand(
        ProductId productId,
        BigDecimal newPrice,
        String currency,
        String reason
    ) {
        public UpdatePriceCommand {
            Objects.requireNonNull(productId, "ProductId is required");
            Objects.requireNonNull(newPrice, "New price is required");
        }
    }
}
```

---

## ขั้นตอนที่ 1939-1944: Secondary Ports

```java
// ProductRepository.java - Secondary Port (Out)
public interface ProductRepository {
    
    ProductId save(Product product);
    
    Optional<Product> findById(ProductId id);
    
    List<Product> findByCategory(String category);
    
    Page<Product> search(GetProductQuery.ProductSearchCriteria criteria, 
                          Pageable pageable);
    
    boolean existsById(ProductId id);
    
    void delete(ProductId id);
}
```

```java
// EmailNotificationPort.java - Secondary Port (Out)
public interface EmailNotificationPort {
    
    void sendPriceChangeNotification(ProductId productId, 
                                      Price oldPrice, 
                                      Price newPrice,
                                      List<String> subscriberEmails);
    
    void sendProductDeactivationNotification(ProductId productId,
                                              List<String> adminEmails);
}
```

```java
// PricingServicePort.java - Secondary Port (Out)
public interface PricingServicePort {
    
    Optional<Price> getSuggestedPrice(ProductId productId);
    
    boolean isPriceValid(Price price, String category);
}
```

---

## ขั้นตอนที่ 1945-1950: Application Service (Core)

```java
// ProductService.java - Application Service (Core)
@Service
@Slf4j
public class ProductService implements CreateProductUseCase, 
                                       GetProductQuery, 
                                       UpdatePriceUseCase {
    
    private final ProductRepository productRepository;
    private final EmailNotificationPort emailNotification;
    private final PricingServicePort pricingService;
    
    // Constructor injection - no Spring annotations in core!
    public ProductService(ProductRepository productRepository,
                          EmailNotificationPort emailNotification,
                          PricingServicePort pricingService) {
        this.productRepository = productRepository;
        this.emailNotification = emailNotification;
        this.pricingService = pricingService;
    }
    
    @Override
    @Transactional
    public ProductId createProduct(CreateProductCommand command) {
        log.info("Creating product: {}", command.name());
        
        // Validate price with external service
        Price price = Price.of(
            command.priceAmount(),
            Currency.getInstance(command.currencyCode())
        );
        
        if (!pricingService.isPriceValid(price, command.categoryName())) {
            throw new InvalidPriceException(
                "ราคาไม่ถูกต้องสำหรับ category: " + command.categoryName()
            );
        }
        
        // Create domain object
        Product product = Product.create(
            command.name(),
            command.description(),
            price,
            new Category(command.categoryName())
        );
        
        // Save through port
        ProductId savedId = productRepository.save(product);
        
        log.info("Product created: {}", savedId);
        return savedId;
    }
    
    @Override
    @Transactional(readOnly = true)
    public Optional<ProductView> getProduct(ProductId productId) {
        return productRepository.findById(productId)
            .map(this::toView);
    }
    
    @Override
    @Transactional(readOnly = true)
    public List<ProductView> getProductsByCategory(String category) {
        return productRepository.findByCategory(category)
            .stream()
            .map(this::toView)
            .collect(Collectors.toList());
    }
    
    @Override
    @Transactional(readOnly = true)
    public Page<ProductView> searchProducts(ProductSearchCriteria criteria, 
                                             Pageable pageable) {
        return productRepository.search(criteria, pageable)
            .map(this::toView);
    }
    
    @Override
    @Transactional
    public void updatePrice(UpdatePriceCommand command) {
        log.info("Updating price for product: {}", command.productId());
        
        Product product = productRepository.findById(command.productId())
            .orElseThrow(() -> new ProductNotFoundException(command.productId().toString()));
        
        Price newPrice = Price.of(
            command.newPrice(),
            Currency.getInstance(command.currency())
        );
        
        Price oldPrice = product.getPrice();
        product.updatePrice(newPrice, command.reason());
        
        productRepository.save(product);
        
        // Notify subscribers ผ่าน secondary port
        List<String> subscribers = getSubscribedEmails(command.productId());
        if (!subscribers.isEmpty()) {
            emailNotification.sendPriceChangeNotification(
                command.productId(), oldPrice, newPrice, subscribers
            );
        }
        
        log.info("Price updated for product: {} from {} to {}",
            command.productId(), oldPrice, newPrice);
    }
    
    private ProductView toView(Product product) {
        return new ProductView(
            product.getId().getValue(),
            product.getName(),
            product.getDescription(),
            product.getPrice().getAmount(),
            product.getPrice().getCurrency().getCurrencyCode(),
            product.getCategory().getName(),
            product.isActive(),
            product.getCreatedAt()
        );
    }
    
    private List<String> getSubscribedEmails(ProductId productId) {
        // Business logic ในการดึง subscribers
        return List.of(); // simplified
    }
}
```

---

## ขั้นตอนที่ 1951-1954: Primary Adapters

### REST Adapter

```java
// ProductController.java - Primary Adapter (REST)
@RestController
@RequestMapping("/api/v1/products")
@Slf4j
public class ProductController {
    
    // Depend on Use Cases, not Service implementation!
    private final CreateProductUseCase createProductUseCase;
    private final GetProductQuery getProductQuery;
    private final UpdatePriceUseCase updatePriceUseCase;
    
    public ProductController(CreateProductUseCase createProductUseCase,
                              GetProductQuery getProductQuery,
                              UpdatePriceUseCase updatePriceUseCase) {
        this.createProductUseCase = createProductUseCase;
        this.getProductQuery = getProductQuery;
        this.updatePriceUseCase = updatePriceUseCase;
    }
    
    @PostMapping
    public ResponseEntity<CreateProductResponse> createProduct(
            @RequestBody @Valid CreateProductRequest request) {
        
        CreateProductUseCase.CreateProductCommand command = 
            new CreateProductUseCase.CreateProductCommand(
                request.name(),
                request.description(),
                request.price(),
                request.currency(),
                request.category()
            );
        
        ProductId productId = createProductUseCase.createProduct(command);
        
        return ResponseEntity
            .created(URI.create("/api/v1/products/" + productId.getValue()))
            .body(new CreateProductResponse(productId.getValue()));
    }
    
    @GetMapping("/{productId}")
    public ResponseEntity<ProductResponse> getProduct(@PathVariable String productId) {
        return getProductQuery.getProduct(ProductId.of(productId))
            .map(view -> ResponseEntity.ok(ProductResponse.from(view)))
            .orElse(ResponseEntity.notFound().build());
    }
    
    @GetMapping
    public ResponseEntity<Page<ProductResponse>> searchProducts(
            @ModelAttribute ProductSearchRequest request,
            Pageable pageable) {
        
        GetProductQuery.ProductSearchCriteria criteria = 
            new GetProductQuery.ProductSearchCriteria(
                request.keyword(),
                request.category(),
                request.minPrice(),
                request.maxPrice(),
                request.activeOnly()
            );
        
        Page<ProductResponse> page = getProductQuery
            .searchProducts(criteria, pageable)
            .map(ProductResponse::from);
        
        return ResponseEntity.ok(page);
    }
    
    @PutMapping("/{productId}/price")
    public ResponseEntity<Void> updatePrice(
            @PathVariable String productId,
            @RequestBody @Valid UpdatePriceRequest request) {
        
        UpdatePriceUseCase.UpdatePriceCommand command = 
            new UpdatePriceUseCase.UpdatePriceCommand(
                ProductId.of(productId),
                request.newPrice(),
                request.currency(),
                request.reason()
            );
        
        updatePriceUseCase.updatePrice(command);
        
        return ResponseEntity.ok().build();
    }
}
```

```java
// REST DTOs
public record CreateProductRequest(
    @NotBlank String name,
    String description,
    @NotNull @Positive BigDecimal price,
    @NotBlank String currency,
    @NotBlank String category
) {}

public record CreateProductResponse(String productId) {}

public record ProductResponse(
    String id,
    String name,
    String description,
    BigDecimal price,
    String currency,
    String category,
    boolean active
) {
    public static ProductResponse from(GetProductQuery.ProductView view) {
        return new ProductResponse(
            view.id(), view.name(), view.description(),
            view.price(), view.currency(), view.category(), view.active()
        );
    }
}
```

### CLI Adapter

```java
// ProductCliAdapter.java - Primary Adapter (CLI)
@Component
@Slf4j
public class ProductCliAdapter implements CommandLineRunner {
    
    private final CreateProductUseCase createProductUseCase;
    private final GetProductQuery getProductQuery;
    
    @Value("${app.cli.enabled:false}")
    private boolean cliEnabled;
    
    public ProductCliAdapter(CreateProductUseCase createProductUseCase,
                              GetProductQuery getProductQuery) {
        this.createProductUseCase = createProductUseCase;
        this.getProductQuery = getProductQuery;
    }
    
    @Override
    public void run(String... args) {
        if (!cliEnabled || args.length == 0) return;
        
        String command = args[0];
        
        switch (command) {
            case "create-product" -> handleCreateProduct(args);
            case "list-products" -> handleListProducts(args);
            case "get-product" -> handleGetProduct(args);
            default -> printHelp();
        }
    }
    
    private void handleCreateProduct(String[] args) {
        if (args.length < 5) {
            System.out.println("Usage: create-product <name> <price> <currency> <category>");
            return;
        }
        
        CreateProductUseCase.CreateProductCommand command = 
            new CreateProductUseCase.CreateProductCommand(
                args[1],
                args.length > 5 ? args[5] : "",
                new BigDecimal(args[2]),
                args[3],
                args[4]
            );
        
        ProductId productId = createProductUseCase.createProduct(command);
        System.out.println("Product created: " + productId.getValue());
    }
    
    private void handleListProducts(String[] args) {
        String category = args.length > 1 ? args[1] : null;
        
        List<GetProductQuery.ProductView> products = category != null
            ? getProductQuery.getProductsByCategory(category)
            : Collections.emptyList();
        
        System.out.println("Products (" + products.size() + "):");
        products.forEach(p -> 
            System.out.printf("  %s | %s | %s %s%n",
                p.id(), p.name(), p.price(), p.currency())
        );
    }
    
    private void handleGetProduct(String[] args) {
        if (args.length < 2) {
            System.out.println("Usage: get-product <productId>");
            return;
        }
        
        getProductQuery.getProduct(ProductId.of(args[1]))
            .ifPresentOrElse(
                p -> System.out.printf("Product: %s%nName: %s%nPrice: %s %s%n",
                    p.id(), p.name(), p.price(), p.currency()),
                () -> System.out.println("Product not found: " + args[1])
            );
    }
    
    private void printHelp() {
        System.out.println("Available commands:");
        System.out.println("  create-product <name> <price> <currency> <category>");
        System.out.println("  list-products [category]");
        System.out.println("  get-product <id>");
    }
}
```

---

## ขั้นตอนที่ 1955-1958: Secondary Adapters

### JPA Adapter

```java
// JpaProductRepository.java - Secondary Adapter (JPA)
@Repository
@Slf4j
public class JpaProductRepository implements ProductRepository {
    
    private final ProductJpaRepository jpaRepository;
    private final ProductMapper mapper;
    
    public JpaProductRepository(ProductJpaRepository jpaRepository,
                                  ProductMapper mapper) {
        this.jpaRepository = jpaRepository;
        this.mapper = mapper;
    }
    
    @Override
    public ProductId save(Product product) {
        ProductJpaEntity entity = mapper.toEntity(product);
        ProductJpaEntity saved = jpaRepository.save(entity);
        return ProductId.of(saved.getId());
    }
    
    @Override
    public Optional<Product> findById(ProductId id) {
        return jpaRepository.findById(id.getValue())
            .map(mapper::toDomain);
    }
    
    @Override
    public List<Product> findByCategory(String category) {
        return jpaRepository.findByCategoryName(category)
            .stream()
            .map(mapper::toDomain)
            .collect(Collectors.toList());
    }
    
    @Override
    public Page<Product> search(GetProductQuery.ProductSearchCriteria criteria, 
                                 Pageable pageable) {
        Specification<ProductJpaEntity> spec = buildSpecification(criteria);
        return jpaRepository.findAll(spec, pageable)
            .map(mapper::toDomain);
    }
    
    @Override
    public boolean existsById(ProductId id) {
        return jpaRepository.existsById(id.getValue());
    }
    
    @Override
    public void delete(ProductId id) {
        jpaRepository.deleteById(id.getValue());
    }
    
    private Specification<ProductJpaEntity> buildSpecification(
            GetProductQuery.ProductSearchCriteria criteria) {
        return Specification
            .where(hasKeyword(criteria.keyword()))
            .and(hasCategory(criteria.category()))
            .and(hasPriceRange(criteria.minPrice(), criteria.maxPrice()))
            .and(isActive(criteria.activeOnly()));
    }
    
    private Specification<ProductJpaEntity> hasKeyword(String keyword) {
        if (keyword == null || keyword.isBlank()) return null;
        return (root, query, cb) -> cb.or(
            cb.like(cb.lower(root.get("name")), "%" + keyword.toLowerCase() + "%"),
            cb.like(cb.lower(root.get("description")), "%" + keyword.toLowerCase() + "%")
        );
    }
    
    private Specification<ProductJpaEntity> hasCategory(String category) {
        if (category == null || category.isBlank()) return null;
        return (root, query, cb) -> 
            cb.equal(root.get("categoryName"), category);
    }
    
    private Specification<ProductJpaEntity> hasPriceRange(BigDecimal min, BigDecimal max) {
        if (min == null && max == null) return null;
        return (root, query, cb) -> {
            List<Predicate> predicates = new ArrayList<>();
            if (min != null) predicates.add(cb.greaterThanOrEqualTo(root.get("priceAmount"), min));
            if (max != null) predicates.add(cb.lessThanOrEqualTo(root.get("priceAmount"), max));
            return cb.and(predicates.toArray(new Predicate[0]));
        };
    }
    
    private Specification<ProductJpaEntity> isActive(boolean activeOnly) {
        if (!activeOnly) return null;
        return (root, query, cb) -> cb.isTrue(root.get("active"));
    }
}
```

```java
// ProductJpaEntity.java - JPA Entity (Framework-specific)
@Entity
@Table(name = "products")
public class ProductJpaEntity {
    
    @Id
    private String id;
    
    @Column(nullable = false)
    private String name;
    
    @Column(columnDefinition = "TEXT")
    private String description;
    
    @Column(nullable = false, precision = 10, scale = 2)
    private BigDecimal priceAmount;
    
    @Column(nullable = false, length = 3)
    private String priceCurrency;
    
    @Column(nullable = false)
    private String categoryName;
    
    private boolean active;
    
    @CreationTimestamp
    private Instant createdAt;
    
    @UpdateTimestamp
    private Instant updatedAt;
    
    // Getters and Setters
    public String getId() { return id; }
    public void setId(String id) { this.id = id; }
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    public String getDescription() { return description; }
    public void setDescription(String description) { this.description = description; }
    public BigDecimal getPriceAmount() { return priceAmount; }
    public void setPriceAmount(BigDecimal priceAmount) { this.priceAmount = priceAmount; }
    public String getPriceCurrency() { return priceCurrency; }
    public void setPriceCurrency(String priceCurrency) { this.priceCurrency = priceCurrency; }
    public String getCategoryName() { return categoryName; }
    public void setCategoryName(String categoryName) { this.categoryName = categoryName; }
    public boolean isActive() { return active; }
    public void setActive(boolean active) { this.active = active; }
    public Instant getCreatedAt() { return createdAt; }
    public Instant getUpdatedAt() { return updatedAt; }
}
```

```java
// ProductMapper.java - Mapping between Domain and JPA
@Component
public class ProductMapper {
    
    public ProductJpaEntity toEntity(Product product) {
        ProductJpaEntity entity = new ProductJpaEntity();
        entity.setId(product.getId().getValue());
        entity.setName(product.getName());
        entity.setDescription(product.getDescription());
        entity.setPriceAmount(product.getPrice().getAmount());
        entity.setPriceCurrency(product.getPrice().getCurrency().getCurrencyCode());
        entity.setCategoryName(product.getCategory().getName());
        entity.setActive(product.isActive());
        return entity;
    }
    
    public Product toDomain(ProductJpaEntity entity) {
        return Product.reconstitute(
            ProductId.of(entity.getId()),
            entity.getName(),
            entity.getDescription(),
            Price.of(entity.getPriceAmount(), 
                     Currency.getInstance(entity.getPriceCurrency())),
            new Category(entity.getCategoryName()),
            entity.isActive(),
            entity.getCreatedAt(),
            entity.getUpdatedAt()
        );
    }
}
```

### Email Adapter

```java
// SmtpEmailAdapter.java - Secondary Adapter (Email)
@Component
@Slf4j
public class SmtpEmailAdapter implements EmailNotificationPort {
    
    private final JavaMailSender mailSender;
    private final TemplateEngine templateEngine;
    
    @Value("${spring.mail.from}")
    private String fromEmail;
    
    public SmtpEmailAdapter(JavaMailSender mailSender, 
                             TemplateEngine templateEngine) {
        this.mailSender = mailSender;
        this.templateEngine = templateEngine;
    }
    
    @Override
    public void sendPriceChangeNotification(ProductId productId, 
                                             Price oldPrice, 
                                             Price newPrice,
                                             List<String> subscriberEmails) {
        log.info("Sending price change notification for product: {}", productId);
        
        Context context = new Context();
        context.setVariable("productId", productId.getValue());
        context.setVariable("oldPrice", oldPrice.toString());
        context.setVariable("newPrice", newPrice.toString());
        context.setVariable("changePercent", 
            calculateChangePercent(oldPrice, newPrice));
        
        String emailBody = templateEngine.process("price-change-email", context);
        
        subscriberEmails.forEach(email -> {
            try {
                MimeMessage message = mailSender.createMimeMessage();
                MimeMessageHelper helper = new MimeMessageHelper(message, true, "UTF-8");
                helper.setFrom(fromEmail);
                helper.setTo(email);
                helper.setSubject("ราคาสินค้าเปลี่ยนแปลง: " + productId.getValue());
                helper.setText(emailBody, true);
                mailSender.send(message);
            } catch (MessagingException e) {
                log.error("Failed to send email to: {}", email, e);
            }
        });
    }
    
    @Override
    public void sendProductDeactivationNotification(ProductId productId,
                                                     List<String> adminEmails) {
        log.info("Sending deactivation notification for product: {}", productId);
        
        adminEmails.forEach(email -> {
            try {
                SimpleMailMessage message = new SimpleMailMessage();
                message.setFrom(fromEmail);
                message.setTo(email);
                message.setSubject("สินค้าถูกปิดการใช้งาน: " + productId.getValue());
                message.setText("สินค้า ID: " + productId.getValue() + " ถูกปิดการใช้งานแล้ว");
                mailSender.send(message);
            } catch (Exception e) {
                log.error("Failed to send deactivation email to: {}", email, e);
            }
        });
    }
    
    private String calculateChangePercent(Price oldPrice, Price newPrice) {
        if (oldPrice.getAmount().compareTo(BigDecimal.ZERO) == 0) return "N/A";
        
        BigDecimal change = newPrice.getAmount().subtract(oldPrice.getAmount());
        BigDecimal percent = change.divide(oldPrice.getAmount(), 4, RoundingMode.HALF_UP)
            .multiply(BigDecimal.valueOf(100));
        
        return percent.setScale(2, RoundingMode.HALF_UP) + "%";
    }
}
```

---

## ขั้นตอนที่ 1959-1960: Testing แต่ละ Adapter

### Testing Core Domain (Unit Test)

```java
// ProductServiceTest.java - Testing Core
class ProductServiceTest {
    
    // ใช้ Mock adapters ไม่ใช่ real implementations
    private ProductRepository mockRepository;
    private EmailNotificationPort mockEmail;
    private PricingServicePort mockPricing;
    private ProductService productService;
    
    @BeforeEach
    void setUp() {
        mockRepository = mock(ProductRepository.class);
        mockEmail = mock(EmailNotificationPort.class);
        mockPricing = mock(PricingServicePort.class);
        productService = new ProductService(mockRepository, mockEmail, mockPricing);
    }
    
    @Test
    void createProduct_validInput_savesProduct() {
        // Given
        when(mockPricing.isPriceValid(any(), any())).thenReturn(true);
        when(mockRepository.save(any())).thenReturn(ProductId.generate());
        
        CreateProductUseCase.CreateProductCommand command = 
            new CreateProductUseCase.CreateProductCommand(
                "Test Product", "Description",
                new BigDecimal("100.00"), "THB", "Electronics"
            );
        
        // When
        ProductId result = productService.createProduct(command);
        
        // Then
        assertThat(result).isNotNull();
        verify(mockRepository, times(1)).save(any(Product.class));
        verify(mockPricing, times(1)).isPriceValid(any(), eq("Electronics"));
    }
    
    @Test
    void createProduct_invalidPrice_throwsException() {
        // Given
        when(mockPricing.isPriceValid(any(), any())).thenReturn(false);
        
        CreateProductUseCase.CreateProductCommand command = 
            new CreateProductUseCase.CreateProductCommand(
                "Test Product", "Description",
                new BigDecimal("999999.00"), "THB", "Electronics"
            );
        
        // When & Then
        assertThatThrownBy(() -> productService.createProduct(command))
            .isInstanceOf(InvalidPriceException.class);
        
        verify(mockRepository, never()).save(any());
    }
    
    @Test
    void updatePrice_sendsEmailToSubscribers() {
        // Given
        Product existingProduct = Product.create(
            "Test Product", "Description",
            Price.thb(new BigDecimal("100.00")),
            new Category("Electronics")
        );
        
        when(mockRepository.findById(any())).thenReturn(Optional.of(existingProduct));
        when(mockRepository.save(any())).thenReturn(existingProduct.getId());
        
        ProductId productId = existingProduct.getId();
        
        UpdatePriceUseCase.UpdatePriceCommand command = 
            new UpdatePriceUseCase.UpdatePriceCommand(
                productId,
                new BigDecimal("120.00"),
                "THB",
                "Price adjustment"
            );
        
        // When
        productService.updatePrice(command);
        
        // Then
        verify(mockRepository, times(1)).save(any(Product.class));
    }
}
```

### Testing REST Adapter

```java
// ProductControllerTest.java - Testing REST Adapter
@WebMvcTest(ProductController.class)
class ProductControllerTest {
    
    @Autowired
    private MockMvc mockMvc;
    
    @MockBean
    private CreateProductUseCase createProductUseCase;
    
    @MockBean
    private GetProductQuery getProductQuery;
    
    @MockBean
    private UpdatePriceUseCase updatePriceUseCase;
    
    @Autowired
    private ObjectMapper objectMapper;
    
    @Test
    void createProduct_validRequest_returns201() throws Exception {
        // Given
        ProductId newId = ProductId.generate();
        when(createProductUseCase.createProduct(any())).thenReturn(newId);
        
        CreateProductRequest request = new CreateProductRequest(
            "Test Product", "Description",
            new BigDecimal("100.00"), "THB", "Electronics"
        );
        
        // When & Then
        mockMvc.perform(post("/api/v1/products")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request)))
            .andExpect(status().isCreated())
            .andExpect(header().string("Location", 
                containsString("/api/v1/products/" + newId.getValue())))
            .andExpect(jsonPath("$.productId").value(newId.getValue()));
    }
    
    @Test
    void getProduct_existing_returns200() throws Exception {
        // Given
        String productId = UUID.randomUUID().toString();
        GetProductQuery.ProductView view = new GetProductQuery.ProductView(
            productId, "Test Product", "Description",
            new BigDecimal("100.00"), "THB", "Electronics",
            true, Instant.now()
        );
        
        when(getProductQuery.getProduct(ProductId.of(productId)))
            .thenReturn(Optional.of(view));
        
        // When & Then
        mockMvc.perform(get("/api/v1/products/{id}", productId))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.id").value(productId))
            .andExpect(jsonPath("$.name").value("Test Product"))
            .andExpect(jsonPath("$.price").value(100.00));
    }
    
    @Test
    void getProduct_notFound_returns404() throws Exception {
        // Given
        String productId = UUID.randomUUID().toString();
        when(getProductQuery.getProduct(any())).thenReturn(Optional.empty());
        
        // When & Then
        mockMvc.perform(get("/api/v1/products/{id}", productId))
            .andExpect(status().isNotFound());
    }
}
```

### Testing JPA Adapter

```java
// JpaProductRepositoryTest.java - Testing Secondary Adapter
@DataJpaTest
@Import(ProductMapper.class)
class JpaProductRepositoryTest {
    
    @Autowired
    private ProductJpaRepository jpaRepository;
    
    @Autowired
    private ProductMapper mapper;
    
    private JpaProductRepository productRepository;
    
    @BeforeEach
    void setUp() {
        productRepository = new JpaProductRepository(jpaRepository, mapper);
    }
    
    @Test
    void save_newProduct_persistsCorrectly() {
        // Given
        Product product = Product.create(
            "Test Product",
            "Description",
            Price.thb(new BigDecimal("100.00")),
            new Category("Electronics")
        );
        
        // When
        ProductId savedId = productRepository.save(product);
        
        // Then
        assertThat(savedId).isNotNull();
        
        Optional<Product> found = productRepository.findById(savedId);
        assertThat(found).isPresent();
        assertThat(found.get().getName()).isEqualTo("Test Product");
        assertThat(found.get().getPrice().getAmount()).isEqualByComparingTo("100.00");
    }
    
    @Test
    void search_byKeyword_returnsMatchingProducts() {
        // Given
        productRepository.save(Product.create(
            "iPhone 15", "Apple smartphone",
            Price.thb(new BigDecimal("35000.00")), new Category("Phones")
        ));
        productRepository.save(Product.create(
            "Samsung Galaxy", "Android smartphone",
            Price.thb(new BigDecimal("25000.00")), new Category("Phones")
        ));
        
        GetProductQuery.ProductSearchCriteria criteria = 
            new GetProductQuery.ProductSearchCriteria(
                "iPhone", null, null, null, true
            );
        
        // When
        Page<Product> results = productRepository.search(criteria, Pageable.ofSize(10));
        
        // Then
        assertThat(results.getTotalElements()).isEqualTo(1);
        assertThat(results.getContent().get(0).getName()).isEqualTo("iPhone 15");
    }
}
```

---

## การเปรียบเทียบ Hexagonal กับ Clean Architecture

| ด้าน | Hexagonal Architecture | Clean Architecture |
|------|----------------------|-------------------|
| **ผู้คิดค้น** | Alistair Cockburn | Robert Martin |
| **แนวคิดหลัก** | Ports & Adapters | Dependency Rule (Layers) |
| **โครงสร้าง** | Hexagon ที่มี ports | Circles (Entity, Use Case, Interface, Framework) |
| **Ports** | Primary (in), Secondary (out) | Interface Adapters, Use Cases |
| **คล้ายกัน** | - Business logic ที่ core | - Business logic ที่ core |
| | - ไม่ขึ้นกับ framework | - ไม่ขึ้นกับ framework |
| | - Testable | - Testable |

---

## สรุป Part 58

ในส่วนนี้เราได้เรียนรู้:

1. **Hexagonal Architecture** - แนวคิด Ports & Adapters
2. **Core Domain Model** - Entities, Value Objects ที่ไม่ขึ้น Framework
3. **Primary Ports** - Use Cases, Queries
4. **Secondary Ports** - Repository, Email, External Service interfaces
5. **Primary Adapters** - REST Controller, CLI
6. **Secondary Adapters** - JPA, SMTP Email
7. **Testing** - Unit test แต่ละ layer แยกกัน

---

*[← Part 57: Chaos Engineering](./part-57-chaos-engineering.md) | [Part 59: Reactive Patterns →](./part-59-reactive-patterns.md)*

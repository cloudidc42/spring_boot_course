# Part 34: CQRS - Command Query Responsibility Segregation
## ขั้นตอนที่ 971-1000

> **ระดับ:** ระดับโลก (World-Class)  
> **เวลาเรียน:** 6-7 ชั่วโมง  
> **เป้าหมาย:** แยก Read/Write model เพื่อ scalability สูงสุด

---

## ขั้นตอนที่ 971: CQRS Concepts

```
Traditional (Single Model):
  UserService.createUser() → Database
  UserService.findUser()   → Database
  (ใช้ Model เดียวทั้ง read/write)

CQRS:
  Command → Write Side → Command DB (normalized, consistent)
  Query  → Read Side  → Query DB (denormalized, fast read)
  
ทำไม CQRS?
  ✅ Scale read/write independently
  ✅ Optimize query model for reads
  ✅ Better separation of concerns
  ✅ Works well with Event Sourcing

เมื่อไหร่ใช้ CQRS?
  ✅ High read:write ratio (90:10)
  ✅ Complex query requirements
  ✅ Need to scale reads independently
  ✅ Large team (split read/write teams)
  
เมื่อไหร่ไม่ควรใช้?
  ❌ Simple CRUD apps
  ❌ Low traffic
  ❌ Small team
  ❌ Read == Write ratio
```

---

## ขั้นตอนที่ 972: Command and Query Objects

```java
// Commands (write side - intent to change state)
public record CreateProductCommand(
    String name,
    String description,
    BigDecimal price,
    Integer stock,
    Long categoryId
) {}

public record UpdateProductPriceCommand(
    Long productId,
    BigDecimal newPrice,
    String reason
) {}

public record DeactivateProductCommand(
    Long productId,
    String reason
) {}

// Queries (read side - no side effects)
public record GetProductByIdQuery(Long id) {}

public record SearchProductsQuery(
    String keyword,
    String status,
    BigDecimal minPrice,
    BigDecimal maxPrice,
    Long categoryId,
    int page,
    int size,
    String sortBy,
    String sortDir
) {}

public record GetProductsByCategoryQuery(
    Long categoryId,
    int page,
    int size
) {}
```

---

## ขั้นตอนที่ 973: Command Handlers

```java
// Command Handler Interface
public interface CommandHandler<C, R> {
    R handle(C command);
}

// Product Command Handlers
@Component
@RequiredArgsConstructor
@Transactional
public class CreateProductCommandHandler implements CommandHandler<CreateProductCommand, Long> {
    
    private final ProductRepository productRepository;
    private final CategoryRepository categoryRepository;
    private final ApplicationEventPublisher eventPublisher;
    
    @Override
    public Long handle(CreateProductCommand command) {
        Category category = null;
        if (command.categoryId() != null) {
            category = categoryRepository.findById(command.categoryId())
                .orElseThrow(() -> new ResourceNotFoundException("Category not found"));
        }
        
        Product product = Product.builder()
            .name(command.name())
            .description(command.description())
            .price(command.price())
            .stock(command.stock())
            .category(category)
            .status(ProductStatus.ACTIVE)
            .build();
        
        Product saved = productRepository.save(product);
        
        eventPublisher.publishEvent(new ProductCreatedEvent(saved));
        
        return saved.getId();
    }
}

@Component
@RequiredArgsConstructor
@Transactional
public class UpdateProductPriceCommandHandler implements CommandHandler<UpdateProductPriceCommand, Void> {
    
    private final ProductRepository productRepository;
    private final PriceChangeAuditRepository auditRepository;
    
    @Override
    public Void handle(UpdateProductPriceCommand command) {
        Product product = productRepository.findById(command.productId())
            .orElseThrow(() -> new ResourceNotFoundException("Product not found"));
        
        BigDecimal oldPrice = product.getPrice();
        product.setPrice(command.newPrice());
        productRepository.save(product);
        
        auditRepository.save(PriceChangeAudit.builder()
            .productId(command.productId())
            .oldPrice(oldPrice)
            .newPrice(command.newPrice())
            .reason(command.reason())
            .changedAt(LocalDateTime.now())
            .build());
        
        return null;
    }
}
```

---

## ขั้นตอนที่ 974: Query Handlers

```java
// Read-optimized view model
public record ProductDetailView(
    Long id,
    String name,
    String description,
    BigDecimal price,
    Integer stock,
    ProductStatus status,
    String categoryName,
    Double averageRating,
    Long reviewCount,
    List<String> tags
) {}

public record ProductListView(
    Long id,
    String name,
    BigDecimal price,
    Integer stock,
    String categoryName,
    ProductStatus status,
    Double averageRating
) {}

// Query Handler Interface
public interface QueryHandler<Q, R> {
    R handle(Q query);
}

@Component
@RequiredArgsConstructor
@Transactional(readOnly = true)
public class GetProductByIdQueryHandler implements QueryHandler<GetProductByIdQuery, ProductDetailView> {
    
    private final EntityManager em;
    
    @Override
    public ProductDetailView handle(GetProductByIdQuery query) {
        // Use native SQL for optimal read query (JOIN all needed data in one query)
        String sql = """
            SELECT 
                p.id, p.name, p.description, p.price, p.stock, p.status,
                c.name as category_name,
                AVG(r.rating) as avg_rating,
                COUNT(r.id) as review_count,
                STRING_AGG(t.name, ',') as tags
            FROM products p
            LEFT JOIN categories c ON p.category_id = c.id
            LEFT JOIN reviews r ON r.product_id = p.id
            LEFT JOIN product_tags pt ON pt.product_id = p.id
            LEFT JOIN tags t ON t.id = pt.tag_id
            WHERE p.id = :id
            GROUP BY p.id, p.name, p.description, p.price, p.stock, p.status, c.name
            """;
        
        return em.createNativeQuery(sql, "ProductDetailViewMapping")
            .setParameter("id", query.id())
            .getResultStream()
            .findFirst()
            .orElseThrow(() -> new ResourceNotFoundException("Product not found: " + query.id()));
    }
}

@Component
@RequiredArgsConstructor
@Transactional(readOnly = true)
public class SearchProductsQueryHandler implements QueryHandler<SearchProductsQuery, Page<ProductListView>> {
    
    private final EntityManager em;
    
    @Override
    public Page<ProductListView> handle(SearchProductsQuery query) {
        StringBuilder sql = new StringBuilder("""
            SELECT p.id, p.name, p.price, p.stock, c.name as category_name, p.status,
                   AVG(r.rating) as avg_rating
            FROM products p
            LEFT JOIN categories c ON p.category_id = c.id
            LEFT JOIN reviews r ON r.product_id = p.id
            WHERE 1=1
            """);
        
        Map<String, Object> params = new HashMap<>();
        
        if (query.keyword() != null) {
            sql.append("AND (p.name ILIKE :keyword OR p.description ILIKE :keyword) ");
            params.put("keyword", "%" + query.keyword() + "%");
        }
        
        if (query.status() != null) {
            sql.append("AND p.status = :status ");
            params.put("status", query.status());
        }
        
        if (query.categoryId() != null) {
            sql.append("AND p.category_id = :categoryId ");
            params.put("categoryId", query.categoryId());
        }
        
        sql.append("GROUP BY p.id, p.name, p.price, p.stock, c.name, p.status ");
        sql.append("ORDER BY ").append(query.sortBy()).append(" ").append(query.sortDir());
        
        var typedQuery = em.createNativeQuery(sql.toString(), "ProductListViewMapping");
        params.forEach(typedQuery::setParameter);
        
        long total = ((Number) typedQuery.getSingleResult()).longValue();
        typedQuery.setFirstResult(query.page() * query.size());
        typedQuery.setMaxResults(query.size());
        
        List<ProductListView> content = typedQuery.getResultList();
        return new PageImpl<>(content, PageRequest.of(query.page(), query.size()), total);
    }
}
```

---

## ขั้นตอนที่ 975: Command/Query Bus

```java
// Command Bus
@Component
@RequiredArgsConstructor
public class CommandBus {
    
    private final ApplicationContext context;
    
    @SuppressWarnings("unchecked")
    public <C, R> R dispatch(C command) {
        Class<?> commandClass = command.getClass();
        
        // Find handler by convention: CommandClass → CommandClassHandler
        String handlerName = commandClass.getSimpleName() + "Handler";
        
        Map<String, ?> handlers = context.getBeansOfType(CommandHandler.class);
        
        CommandHandler<C, R> handler = (CommandHandler<C, R>) handlers.values().stream()
            .filter(h -> h.getClass().getSimpleName().equals(handlerName))
            .findFirst()
            .orElseThrow(() -> new IllegalArgumentException("No handler for: " + commandClass.getName()));
        
        return handler.handle(command);
    }
}

// Query Bus
@Component
@RequiredArgsConstructor
public class QueryBus {
    
    private final ApplicationContext context;
    
    @SuppressWarnings("unchecked")
    public <Q, R> R dispatch(Q query) {
        String handlerName = query.getClass().getSimpleName() + "Handler";
        
        Map<String, ?> handlers = context.getBeansOfType(QueryHandler.class);
        
        QueryHandler<Q, R> handler = (QueryHandler<Q, R>) handlers.values().stream()
            .filter(h -> h.getClass().getSimpleName().equals(handlerName))
            .findFirst()
            .orElseThrow(() -> new IllegalArgumentException("No handler for: " + query.getClass().getName()));
        
        return handler.handle(query);
    }
}
```

---

## ขั้นตอนที่ 976: Controller with CQRS

```java
@RestController
@RequestMapping("/api/v1/products")
@RequiredArgsConstructor
public class ProductController {
    
    private final CommandBus commandBus;
    private final QueryBus queryBus;
    
    // Write endpoints - use Commands
    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public ResponseEntity<Map<String, Long>> create(
        @Valid @RequestBody CreateProductCommand command
    ) {
        Long id = commandBus.dispatch(command);
        return ResponseEntity
            .created(URI.create("/api/v1/products/" + id))
            .body(Map.of("id", id));
    }
    
    @PatchMapping("/{id}/price")
    public ResponseEntity<Void> updatePrice(
        @PathVariable Long id,
        @Valid @RequestBody UpdatePriceRequest request
    ) {
        commandBus.dispatch(new UpdateProductPriceCommand(id, request.newPrice(), request.reason()));
        return ResponseEntity.noContent().build();
    }
    
    @DeleteMapping("/{id}")
    public ResponseEntity<Void> deactivate(
        @PathVariable Long id,
        @RequestParam(required = false) String reason
    ) {
        commandBus.dispatch(new DeactivateProductCommand(id, reason));
        return ResponseEntity.noContent().build();
    }
    
    // Read endpoints - use Queries
    @GetMapping("/{id}")
    public ResponseEntity<ProductDetailView> findById(@PathVariable Long id) {
        return ResponseEntity.ok(queryBus.dispatch(new GetProductByIdQuery(id)));
    }
    
    @GetMapping
    public ResponseEntity<Page<ProductListView>> search(
        @RequestParam(required = false) String keyword,
        @RequestParam(required = false) String status,
        @RequestParam(defaultValue = "0") int page,
        @RequestParam(defaultValue = "20") int size
    ) {
        var query = new SearchProductsQuery(keyword, status, null, null, null,
            page, size, "id", "desc");
        return ResponseEntity.ok(queryBus.dispatch(query));
    }
}
```

---

## ขั้นตอนที่ 977-1000: Read Model Synchronization

```java
// When write side changes, sync read model
@Component
@RequiredArgsConstructor
public class ProductReadModelSync {
    
    private final ProductReadRepository readRepository;
    
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    @Async
    public void onProductCreated(ProductCreatedEvent event) {
        ProductReadModel readModel = ProductReadModel.builder()
            .id(event.getProduct().getId())
            .name(event.getProduct().getName())
            .price(event.getProduct().getPrice())
            .build();
        
        readRepository.save(readModel);
    }
    
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    @Async
    public void onPriceChanged(ProductPriceChangedEvent event) {
        readRepository.updatePrice(event.getProductId(), event.getNewPrice());
    }
}

// CQRS Pattern Summary
/*
  Benefits:
  ✅ Read model optimized purely for queries
  ✅ Write model enforces business rules
  ✅ Scale independently
  ✅ Clear separation of concerns
  ✅ Event sourcing compatible
  
  Trade-offs:
  ❌ More complexity
  ❌ Eventual consistency (read model lag)
  ❌ Two data models to maintain
  
  Best with:
  → High-traffic read-heavy systems
  → Complex reporting requirements
  → Large teams
  → Microservices architecture
*/
```

---

*[← Part 33: Domain-Driven Design](./part-33-ddd.md) | [Part 35: OAuth2 & OIDC →](./part-35-oauth2.md)*

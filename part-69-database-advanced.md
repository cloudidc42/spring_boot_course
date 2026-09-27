# Part 69: Database Advanced
## ขั้นตอนที่ 2361-2400

**ระดับ:** ระดับโลก (World-Class)
**เวลาเรียน:** 8-9 ชั่วโมง
**เป้าหมาย:** เชี่ยวชาญเทคนิค Database ขั้นสูงใน Spring Boot ตั้งแต่ JPA @EntityGraph, QueryDSL, Database Sharding, Read Replicas, Locking strategies และ JSON columns

---

## ขั้นตอนที่ 2361-2366: Advanced JPA - @EntityGraph และ FetchType

### N+1 Query Problem และการแก้ไข

ปัญหา N+1 เกิดขึ้นเมื่อ JPA โหลด parent entity แล้ว query ลูกแต่ละ entity แยกกัน:

```java
// ❌ BAD: N+1 Problem
@Entity
public class Order {
    @OneToMany(fetch = FetchType.LAZY)  // Default: LAZY
    private List<OrderItem> items;

    @ManyToOne(fetch = FetchType.LAZY)
    private Customer customer;
}

// เรียก findAll() ได้ 1 query
List<Order> orders = orderRepository.findAll();

// แต่เวลา access items หรือ customer จะเกิด query เพิ่มสำหรับแต่ละ order
for (Order order : orders) {
    order.getItems().size();      // Query สำหรับแต่ละ order
    order.getCustomer().getName(); // Query อีก 1 ต่อ order
}
// 1 (orders) + N (items) + N (customers) = 2N+1 queries !!
```

### แก้ด้วย @EntityGraph

```java
// OrderRepository.java
public interface OrderRepository extends JpaRepository<Order, Long> {

    // ใช้ @EntityGraph เพื่อ JOIN FETCH ใน single query
    @EntityGraph(attributePaths = {"items", "customer"})
    List<Order> findAll();

    // Named Entity Graph
    @EntityGraph(value = "Order.withItemsAndCustomer")
    Optional<Order> findById(Long id);

    // EntityGraph สำหรับ specific query
    @EntityGraph(attributePaths = {"items", "items.product", "customer"})
    @Query("SELECT o FROM Order o WHERE o.status = :status")
    List<Order> findByStatusWithDetails(@Param("status") OrderStatus status);

    // Fetch เฉพาะบาง fields
    @EntityGraph(attributePaths = {"customer"})
    @Query("SELECT o FROM Order o WHERE o.customerId = :customerId")
    List<Order> findByCustomerIdWithCustomer(@Param("customerId") String customerId);
}

// Order.java - ประกาศ Named Entity Graph
@Entity
@Table(name = "orders")
@NamedEntityGraph(
    name = "Order.withItemsAndCustomer",
    attributeNodes = {
        @NamedAttributeNode("items"),
        @NamedAttributeNode(value = "customer", subgraph = "customer.address")
    },
    subgraphs = {
        @NamedSubgraph(
            name = "customer.address",
            attributeNodes = @NamedAttributeNode("address")
        )
    }
)
public class Order {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String customerId;

    @OneToMany(cascade = CascadeType.ALL, fetch = FetchType.LAZY)
    @JoinColumn(name = "order_id")
    private List<OrderItem> items;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "customer_id")
    private Customer customer;

    // ...
}
```

### FetchType Strategy

```java
// FetchType Guidelines:
// - FetchType.LAZY (default สำหรับ @OneToMany, @ManyToMany): โหลดเมื่อต้องใช้
// - FetchType.EAGER (default สำหรับ @ManyToOne, @OneToOne): โหลดพร้อมกับ parent

@Entity
public class Product {
    // ✅ LAZY สำหรับ collections (มักมีหลาย items)
    @OneToMany(fetch = FetchType.LAZY)
    private List<Review> reviews;

    // ✅ LAZY สำหรับ ManyToOne ถ้าไม่จำเป็น
    @ManyToOne(fetch = FetchType.LAZY)
    private Category category;

    // ⚠️ EAGER ใช้ได้กับ @OneToOne ที่ต้องการ always
    @OneToOne(fetch = FetchType.LAZY)  // แนะนำ LAZY แม้กระทั่ง @OneToOne
    private ProductDetail detail;
}
```

---

## ขั้นตอนที่ 2367-2373: JPQL vs Criteria API vs QueryDSL

### JPQL (Java Persistence Query Language)

```java
// OrderRepository.java - JPQL queries
public interface OrderRepository extends JpaRepository<Order, Long> {

    // Simple JPQL
    @Query("SELECT o FROM Order o WHERE o.status = :status AND o.totalAmount > :minAmount")
    List<Order> findByStatusAndMinAmount(
        @Param("status") OrderStatus status,
        @Param("minAmount") BigDecimal minAmount
    );

    // JOIN กับ collection
    @Query("""
        SELECT DISTINCT o FROM Order o
        JOIN FETCH o.items i
        WHERE i.productId = :productId
        AND o.createdAt BETWEEN :startDate AND :endDate
        """)
    List<Order> findOrdersContainingProduct(
        @Param("productId") Long productId,
        @Param("startDate") LocalDateTime startDate,
        @Param("endDate") LocalDateTime endDate
    );

    // Aggregation
    @Query("""
        SELECT o.customerId, COUNT(o), SUM(o.totalAmount)
        FROM Order o
        WHERE o.status = 'DELIVERED'
        GROUP BY o.customerId
        HAVING COUNT(o) > :minOrders
        ORDER BY SUM(o.totalAmount) DESC
        """)
    List<Object[]> findTopCustomers(@Param("minOrders") long minOrders);

    // Native SQL query สำหรับ database-specific features
    @Query(
        value = """
            SELECT * FROM orders o
            WHERE o.metadata @> :metadata::jsonb
            AND o.created_at > NOW() - INTERVAL '7 days'
            """,
        nativeQuery = true
    )
    List<Order> findByMetadata(@Param("metadata") String metadata);

    // Pagination
    @Query(
        value = "SELECT o FROM Order o WHERE o.status = :status",
        countQuery = "SELECT COUNT(o) FROM Order o WHERE o.status = :status"
    )
    Page<Order> findByStatusPaged(@Param("status") OrderStatus status, Pageable pageable);
}
```

### Criteria API สำหรับ Dynamic Queries

```java
// OrderSpecification.java - Specifications for dynamic filtering
public class OrderSpecification {

    public static Specification<Order> hasStatus(OrderStatus status) {
        return (root, query, cb) ->
            status == null ? null : cb.equal(root.get("status"), status);
    }

    public static Specification<Order> hasCustomerId(String customerId) {
        return (root, query, cb) ->
            customerId == null ? null : cb.equal(root.get("customerId"), customerId);
    }

    public static Specification<Order> hasTotalAmountBetween(
        BigDecimal min, BigDecimal max
    ) {
        return (root, query, cb) -> {
            if (min == null && max == null) return null;
            if (min == null) return cb.lessThanOrEqualTo(root.get("totalAmount"), max);
            if (max == null) return cb.greaterThanOrEqualTo(root.get("totalAmount"), min);
            return cb.between(root.get("totalAmount"), min, max);
        };
    }

    public static Specification<Order> createdBetween(
        LocalDateTime start, LocalDateTime end
    ) {
        return (root, query, cb) -> {
            if (start == null && end == null) return null;
            if (start == null) return cb.lessThanOrEqualTo(root.get("createdAt"), end);
            if (end == null) return cb.greaterThanOrEqualTo(root.get("createdAt"), start);
            return cb.between(root.get("createdAt"), start, end);
        };
    }

    public static Specification<Order> containsProduct(Long productId) {
        return (root, query, cb) -> {
            query.distinct(true);  // ป้องกัน duplicate
            Join<Order, OrderItem> items = root.join("items", JoinType.INNER);
            return cb.equal(items.get("productId"), productId);
        };
    }
}

// การใช้งาน Specifications
@Service
public class OrderQueryService {

    private final OrderRepository orderRepository;

    public Page<Order> searchOrders(OrderSearchRequest request, Pageable pageable) {
        Specification<Order> spec = Specification
            .where(OrderSpecification.hasStatus(request.getStatus()))
            .and(OrderSpecification.hasCustomerId(request.getCustomerId()))
            .and(OrderSpecification.hasTotalAmountBetween(
                request.getMinAmount(),
                request.getMaxAmount()
            ))
            .and(OrderSpecification.createdBetween(
                request.getStartDate(),
                request.getEndDate()
            ));

        if (request.getProductId() != null) {
            spec = spec.and(OrderSpecification.containsProduct(request.getProductId()));
        }

        return orderRepository.findAll(spec, pageable);
    }
}
```

### QueryDSL - Type-safe Queries

```xml
<!-- pom.xml -->
<dependency>
    <groupId>com.querydsl</groupId>
    <artifactId>querydsl-jpa</artifactId>
    <classifier>jakarta</classifier>
</dependency>
<dependency>
    <groupId>com.querydsl</groupId>
    <artifactId>querydsl-apt</artifactId>
    <classifier>jakarta</classifier>
</dependency>

<!-- APT Plugin สำหรับ Generate Q-classes -->
<plugin>
    <groupId>com.mysema.maven</groupId>
    <artifactId>apt-maven-plugin</artifactId>
    <version>1.1.3</version>
    <executions>
        <execution>
            <goals><goal>process</goal></goals>
            <configuration>
                <outputDirectory>target/generated-sources/java</outputDirectory>
                <processor>com.querydsl.apt.jpa.JPAAnnotationProcessor</processor>
            </configuration>
        </execution>
    </executions>
</plugin>
```

```java
// OrderQueryRepository.java - QueryDSL Repository
@Repository
public class OrderQueryRepository {

    private final JPAQueryFactory queryFactory;

    public OrderQueryRepository(EntityManager entityManager) {
        this.queryFactory = new JPAQueryFactory(entityManager);
    }

    // Type-safe query (compile error ถ้า field ผิด)
    public List<Order> findOrdersWithFilters(OrderFilter filter) {
        QOrder order = QOrder.order;
        QOrderItem item = QOrderItem.orderItem;
        QCustomer customer = QCustomer.customer;

        BooleanBuilder where = new BooleanBuilder();

        if (filter.getStatus() != null) {
            where.and(order.status.eq(filter.getStatus()));
        }

        if (filter.getCustomerId() != null) {
            where.and(order.customerId.eq(filter.getCustomerId()));
        }

        if (filter.getMinAmount() != null) {
            where.and(order.totalAmount.goe(filter.getMinAmount()));
        }

        if (filter.getMaxAmount() != null) {
            where.and(order.totalAmount.loe(filter.getMaxAmount()));
        }

        if (filter.getSearchTerm() != null) {
            where.and(
                order.customerId.containsIgnoreCase(filter.getSearchTerm())
                    .or(customer.email.containsIgnoreCase(filter.getSearchTerm()))
            );
        }

        return queryFactory
            .selectFrom(order)
            .leftJoin(order.customer, customer).fetchJoin()
            .leftJoin(order.items, item).fetchJoin()
            .where(where)
            .orderBy(order.createdAt.desc())
            .fetch();
    }

    // Aggregation query
    public List<CustomerOrderSummary> getCustomerOrderSummaries() {
        QOrder order = QOrder.order;

        return queryFactory
            .select(Projections.constructor(
                CustomerOrderSummary.class,
                order.customerId,
                order.count(),
                order.totalAmount.sum(),
                order.totalAmount.avg()
            ))
            .from(order)
            .where(order.status.eq(OrderStatus.DELIVERED))
            .groupBy(order.customerId)
            .having(order.count().gt(5))
            .orderBy(order.totalAmount.sum().desc())
            .limit(100)
            .fetch();
    }

    // Pagination
    public QueryResults<Order> findOrdersPaged(
        OrderFilter filter,
        long offset,
        long limit
    ) {
        QOrder order = QOrder.order;
        BooleanBuilder where = buildWhereClause(filter, order);

        return queryFactory
            .selectFrom(order)
            .where(where)
            .orderBy(order.createdAt.desc())
            .offset(offset)
            .limit(limit)
            .fetchResults();
    }

    private BooleanBuilder buildWhereClause(OrderFilter filter, QOrder order) {
        BooleanBuilder where = new BooleanBuilder();
        if (filter.getStatus() != null) {
            where.and(order.status.eq(filter.getStatus()));
        }
        return where;
    }
}
```

---

## ขั้นตอนที่ 2374-2379: Optimistic vs Pessimistic Locking

### Optimistic Locking ด้วย @Version

```java
// Product.java - Optimistic Locking
@Entity
@Table(name = "products")
public class Product {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;
    private int stock;
    private BigDecimal price;

    // Version field สำหรับ Optimistic Locking
    @Version
    private Long version;

    public void decreaseStock(int quantity) {
        if (this.stock < quantity) {
            throw new InsufficientStockException(id, quantity, stock);
        }
        this.stock -= quantity;
    }
}

// Service ที่ใช้ Optimistic Locking
@Service
public class InventoryService {

    private final ProductRepository productRepository;

    @Retryable(
        retryFor = OptimisticLockingFailureException.class,
        maxAttempts = 3,
        backoff = @Backoff(delay = 100, multiplier = 2)
    )
    @Transactional
    public void decreaseStock(Long productId, int quantity) {
        Product product = productRepository.findById(productId)
            .orElseThrow(() -> new ProductNotFoundException(productId));

        product.decreaseStock(quantity);
        productRepository.save(product);
        // ถ้ามี concurrent update: JPA throw OptimisticLockingFailureException
        // @Retryable จะ retry อัตโนมัติ
    }
}
```

### Pessimistic Locking

```java
// ProductRepository.java - Pessimistic Locking
public interface ProductRepository extends JpaRepository<Product, Long> {

    // PESSIMISTIC_WRITE: Lock record สำหรับ update (SELECT ... FOR UPDATE)
    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("SELECT p FROM Product p WHERE p.id = :id")
    Optional<Product> findByIdForUpdate(@Param("id") Long id);

    // PESSIMISTIC_READ: Lock record สำหรับ read เท่านั้น (SELECT ... FOR SHARE)
    @Lock(LockModeType.PESSIMISTIC_READ)
    @Query("SELECT p FROM Product p WHERE p.id = :id")
    Optional<Product> findByIdForRead(@Param("id") Long id);

    // Lock timeout (PostgreSQL specific)
    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @QueryHints(@QueryHint(
        name = "jakarta.persistence.lock.timeout",
        value = "3000"  // 3 seconds timeout
    ))
    @Query("SELECT p FROM Product p WHERE p.id = :id")
    Optional<Product> findByIdWithTimeout(@Param("id") Long id);
}

// Service ใช้ Pessimistic Locking สำหรับ Critical Section
@Service
public class StockService {

    @Transactional(isolation = Isolation.READ_COMMITTED)
    public void transferStock(Long fromProductId, Long toProductId, int quantity) {
        // Lock ทั้ง 2 products ก่อน เพื่อป้องกัน deadlock ต้อง lock ตาม order เดิมเสมอ
        Long minId = Math.min(fromProductId, toProductId);
        Long maxId = Math.max(fromProductId, toProductId);

        Product first = productRepository.findByIdForUpdate(minId)
            .orElseThrow(() -> new ProductNotFoundException(minId));
        Product second = productRepository.findByIdForUpdate(maxId)
            .orElseThrow(() -> new ProductNotFoundException(maxId));

        Product from = fromProductId.equals(minId) ? first : second;
        Product to = toProductId.equals(minId) ? first : second;

        from.decreaseStock(quantity);
        to.increaseStock(quantity);

        productRepository.save(from);
        productRepository.save(to);
    }
}
```

### เมื่อไหร่ใช้อะไร?

```java
/*
Optimistic Locking ใช้เมื่อ:
- Conflict น้อย (read มากกว่า write)
- Performance สำคัญ (ไม่มี database lock)
- Web applications ทั่วไป
ตัวอย่าง: อัพเดท profile, บันทึก draft

Pessimistic Locking ใช้เมื่อ:
- Conflict สูง (หลาย users update data เดียวกัน)
- Critical operations (payment, stock deduction)
- ไม่ยอมรับ retry
ตัวอย่าง: การจอง seat, การ deduct stock
*/
```

---

## ขั้นตอนที่ 2380-2386: Read Replicas Routing

### AbstractRoutingDataSource สำหรับ Read/Write Splitting

```java
// DataSourceConfig.java
@Configuration
public class DataSourceConfig {

    @Bean
    @ConfigurationProperties("spring.datasource.primary")
    public DataSource primaryDataSource() {
        return DataSourceBuilder.create().build();
    }

    @Bean
    @ConfigurationProperties("spring.datasource.replica")
    public DataSource replicaDataSource() {
        return DataSourceBuilder.create().build();
    }

    @Bean
    @Primary
    public DataSource routingDataSource(
        @Qualifier("primaryDataSource") DataSource primary,
        @Qualifier("replicaDataSource") DataSource replica
    ) {
        Map<Object, Object> targetDataSources = new HashMap<>();
        targetDataSources.put(DataSourceType.PRIMARY, primary);
        targetDataSources.put(DataSourceType.REPLICA, replica);

        RoutingDataSource routingDataSource = new RoutingDataSource();
        routingDataSource.setTargetDataSources(targetDataSources);
        routingDataSource.setDefaultTargetDataSource(primary);

        return routingDataSource;
    }
}

// DataSourceType.java
public enum DataSourceType {
    PRIMARY, REPLICA
}

// DataSourceContextHolder.java - ThreadLocal สำหรับ routing
public class DataSourceContextHolder {

    private static final ThreadLocal<DataSourceType> context =
        new ThreadLocal<>();

    public static void setDataSourceType(DataSourceType type) {
        context.set(type);
    }

    public static DataSourceType getDataSourceType() {
        return context.get();
    }

    public static void clear() {
        context.remove();
    }
}

// RoutingDataSource.java
public class RoutingDataSource extends AbstractRoutingDataSource {

    @Override
    protected Object determineCurrentLookupKey() {
        DataSourceType type = DataSourceContextHolder.getDataSourceType();
        // ถ้าไม่มีการกำหนด ใช้ PRIMARY
        return type != null ? type : DataSourceType.PRIMARY;
    }
}

// ReadOnly Annotation สำหรับ Routing
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
public @interface ReadOnly {}

// ReadOnlyAspect.java - AOP สำหรับ auto-routing
@Aspect
@Component
@Order(1)  // ต้องทำก่อน @Transactional
public class ReadOnlyAspect {

    @Around("@annotation(ReadOnly) || @annotation(org.springframework.transaction.annotation.Transactional)")
    public Object routeDataSource(ProceedingJoinPoint joinPoint) throws Throwable {
        Transactional transactional = getTransactionalAnnotation(joinPoint);
        ReadOnly readOnly = getReadOnlyAnnotation(joinPoint);

        boolean isReadOnly = (transactional != null && transactional.readOnly())
            || readOnly != null;

        if (isReadOnly) {
            DataSourceContextHolder.setDataSourceType(DataSourceType.REPLICA);
        } else {
            DataSourceContextHolder.setDataSourceType(DataSourceType.PRIMARY);
        }

        try {
            return joinPoint.proceed();
        } finally {
            DataSourceContextHolder.clear();
        }
    }

    private Transactional getTransactionalAnnotation(ProceedingJoinPoint joinPoint) {
        MethodSignature signature = (MethodSignature) joinPoint.getSignature();
        Method method = signature.getMethod();
        return AnnotationUtils.findAnnotation(method, Transactional.class);
    }

    private ReadOnly getReadOnlyAnnotation(ProceedingJoinPoint joinPoint) {
        MethodSignature signature = (MethodSignature) joinPoint.getSignature();
        Method method = signature.getMethod();
        return AnnotationUtils.findAnnotation(method, ReadOnly.class);
    }
}
```

### การใช้ Read Replica ใน Service

```java
// OrderService.java
@Service
public class OrderService {

    @Transactional(readOnly = true)  // อ่านจาก Replica
    public Page<Order> findOrders(Pageable pageable) {
        return orderRepository.findAll(pageable);
    }

    @ReadOnly  // อ่านจาก Replica
    public Order findById(Long id) {
        return orderRepository.findById(id)
            .orElseThrow(() -> new OrderNotFoundException(id));
    }

    @Transactional  // เขียนไปที่ Primary
    public Order createOrder(CreateOrderRequest request) {
        // ...
    }
}
```

```yaml
# application.yml - Database Configuration
spring:
  datasource:
    primary:
      url: jdbc:postgresql://primary-db:5432/orders
      username: ${DB_USERNAME}
      password: ${DB_PASSWORD}
      hikari:
        maximum-pool-size: 20

    replica:
      url: jdbc:postgresql://replica-db:5432/orders
      username: ${DB_READONLY_USERNAME}
      password: ${DB_READONLY_PASSWORD}
      hikari:
        maximum-pool-size: 30  # Replica รองรับ read มากกว่า
```

---

## ขั้นตอนที่ 2387-2392: Bulk Operations Performance

### Batch Insert และ Update

```java
// BatchInsertService.java
@Service
public class BatchInsertService {

    private final EntityManager entityManager;

    @Transactional
    public void batchInsertOrders(List<Order> orders) {
        int batchSize = 50;

        for (int i = 0; i < orders.size(); i++) {
            entityManager.persist(orders.get(i));

            // Flush และ clear ทุก batch เพื่อไม่ให้ memory overflow
            if (i % batchSize == 0 && i > 0) {
                entityManager.flush();
                entityManager.clear();
            }
        }

        entityManager.flush();
        entityManager.clear();
    }

    // Batch Update ด้วย JPQL
    @Transactional
    @Modifying(clearAutomatically = true)
    public int cancelOrdersBefore(LocalDateTime cutoffDate) {
        return entityManager.createQuery(
            "UPDATE Order o SET o.status = :newStatus, o.updatedAt = :now " +
            "WHERE o.status = :oldStatus AND o.createdAt < :cutoff"
        )
        .setParameter("newStatus", OrderStatus.CANCELLED)
        .setParameter("now", LocalDateTime.now())
        .setParameter("oldStatus", OrderStatus.PENDING)
        .setParameter("cutoff", cutoffDate)
        .executeUpdate();
    }

    // JDBC Batch Insert สำหรับ performance สูงสุด
    @Transactional
    public void jdbcBatchInsert(List<Order> orders) {
        jdbcTemplate.batchUpdate(
            "INSERT INTO orders (customer_id, status, total_amount, created_at) VALUES (?, ?, ?, ?)",
            new BatchPreparedStatementSetter() {
                @Override
                public void setValues(PreparedStatement ps, int i) throws SQLException {
                    Order order = orders.get(i);
                    ps.setString(1, order.getCustomerId());
                    ps.setString(2, order.getStatus().name());
                    ps.setBigDecimal(3, order.getTotalAmount());
                    ps.setTimestamp(4, Timestamp.valueOf(order.getCreatedAt()));
                }

                @Override
                public int getBatchSize() {
                    return orders.size();
                }
            }
        );
    }
}
```

### Pagination สำหรับ Large Dataset

```java
// DataExportService.java - Process large data ด้วย Pagination
@Service
public class DataExportService {

    private final OrderRepository orderRepository;

    // ❌ BAD: โหลดทั้งหมดในครั้งเดียว - OutOfMemoryError
    public List<Order> exportAllOrders_BAD() {
        return orderRepository.findAll(); // อาจมีล้าน records!
    }

    // ✅ GOOD: ใช้ Pagination
    public void exportAllOrders(Consumer<List<Order>> processor) {
        int pageSize = 1000;
        int page = 0;

        Page<Order> orderPage;
        do {
            orderPage = orderRepository.findAll(PageRequest.of(page, pageSize));
            processor.accept(orderPage.getContent());
            page++;
        } while (orderPage.hasNext());
    }

    // ✅ BETTER: ใช้ Stream กับ ScrollQuery (Spring Data 3.x)
    @Transactional(readOnly = true)
    public void streamOrders(Consumer<Order> processor) {
        try (Stream<Order> stream = orderRepository.findAllByStatusOrderByCreatedAt(
            OrderStatus.DELIVERED
        )) {
            stream.forEach(processor);
        }
    }
}
```

---

## ขั้นตอนที่ 2393-2400: JSON Columns กับ PostgreSQL

### Setup JSON Column Support

```xml
<!-- pom.xml -->
<dependency>
    <groupId>com.vladmihalcea</groupId>
    <artifactId>hibernate-types-60</artifactId>
    <version>2.21.1</version>
</dependency>
```

```java
// Order.java - ใช้ JSON Column
@Entity
@Table(name = "orders")
@TypeDef(name = "json", typeClass = JsonStringType.class)
@TypeDef(name = "jsonb", typeClass = JsonBinaryType.class)
public class Order {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String customerId;

    // JSON column สำหรับ shipping address
    @Type(type = "jsonb")
    @Column(name = "shipping_address", columnDefinition = "jsonb")
    private ShippingAddress shippingAddress;

    // JSON array สำหรับ tags
    @Type(type = "jsonb")
    @Column(name = "tags", columnDefinition = "jsonb")
    private List<String> tags = new ArrayList<>();

    // JSON object สำหรับ metadata
    @Type(type = "jsonb")
    @Column(name = "metadata", columnDefinition = "jsonb")
    private Map<String, Object> metadata = new HashMap<>();

    // Nested JSON
    @Type(type = "jsonb")
    @Column(name = "payment_details", columnDefinition = "jsonb")
    private PaymentDetails paymentDetails;
}

// ShippingAddress.java - POJO สำหรับ JSON
@Data
@NoArgsConstructor
@AllArgsConstructor
public class ShippingAddress {
    private String street;
    private String city;
    private String province;
    private String postalCode;
    private String country;
    private Double latitude;
    private Double longitude;

    public String getFullAddress() {
        return String.format("%s, %s, %s %s, %s",
            street, city, province, postalCode, country);
    }
}

// PaymentDetails.java
@Data
public class PaymentDetails {
    private String method;  // "CREDIT_CARD", "PROMPTPAY", "COD"
    private String transactionId;
    private String maskedCardNumber;
    private String bankCode;
    private LocalDateTime paidAt;
    private Map<String, String> additionalInfo;
}
```

### Query JSON Fields ใน PostgreSQL

```java
// OrderRepository.java - Native queries สำหรับ JSONB
public interface OrderRepository extends JpaRepository<Order, Long> {

    // ค้นหาตาม JSON field
    @Query(
        value = "SELECT * FROM orders WHERE shipping_address->>'city' = :city",
        nativeQuery = true
    )
    List<Order> findByShippingCity(@Param("city") String city);

    // ค้นหาตาม nested JSON field
    @Query(
        value = """
            SELECT * FROM orders
            WHERE payment_details->>'method' = :paymentMethod
            AND created_at > :since
            """,
        nativeQuery = true
    )
    List<Order> findByPaymentMethodSince(
        @Param("paymentMethod") String paymentMethod,
        @Param("since") LocalDateTime since
    );

    // ค้นหา order ที่มี tag ที่ระบุ
    @Query(
        value = "SELECT * FROM orders WHERE tags @> :tag::jsonb",
        nativeQuery = true
    )
    List<Order> findByTag(@Param("tag") String tag);  // tag = '["express"]'

    // Full-text search ใน JSON
    @Query(
        value = """
            SELECT * FROM orders
            WHERE metadata @> :criteria::jsonb
            """,
        nativeQuery = true
    )
    List<Order> findByMetadata(@Param("criteria") String criteria);

    // Aggregate JSON data
    @Query(
        value = """
            SELECT
                shipping_address->>'city' as city,
                COUNT(*) as order_count,
                SUM(total_amount) as total_revenue
            FROM orders
            WHERE status = 'DELIVERED'
            GROUP BY shipping_address->>'city'
            ORDER BY total_revenue DESC
            LIMIT :limit
            """,
        nativeQuery = true
    )
    List<Object[]> getRevenueByCity(@Param("limit") int limit);
}
```

### JSON Index สำหรับ Performance

```sql
-- ใน migration file (Flyway)
-- Index บน JSON field ที่ query บ่อย
CREATE INDEX idx_orders_shipping_city
ON orders ((shipping_address->>'city'));

-- GIN Index สำหรับ containment queries (@>)
CREATE INDEX idx_orders_tags
ON orders USING GIN (tags);

CREATE INDEX idx_orders_metadata
ON orders USING GIN (metadata);

-- Composite index
CREATE INDEX idx_orders_payment_method_status
ON orders (status, (payment_details->>'method'));
```

### JSON Column Service

```java
// OrderMetadataService.java
@Service
@Transactional
public class OrderMetadataService {

    private final OrderRepository orderRepository;

    // เพิ่ม metadata ให้ Order
    public Order addMetadata(Long orderId, String key, Object value) {
        Order order = orderRepository.findById(orderId)
            .orElseThrow(() -> new OrderNotFoundException(orderId));

        order.getMetadata().put(key, value);
        return orderRepository.save(order);
    }

    // Update shipping address
    public Order updateShippingAddress(Long orderId, ShippingAddress address) {
        Order order = orderRepository.findById(orderId)
            .orElseThrow(() -> new OrderNotFoundException(orderId));

        if (order.getStatus() != OrderStatus.PENDING) {
            throw new IllegalStateException(
                "Cannot update shipping address for order in status: " + order.getStatus()
            );
        }

        order.setShippingAddress(address);
        return orderRepository.save(order);
    }

    // ค้นหา orders ตาม metadata
    @Transactional(readOnly = true)
    public List<Order> findByMetadataCriteria(Map<String, Object> criteria) {
        try {
            String criteriaJson = new ObjectMapper().writeValueAsString(criteria);
            return orderRepository.findByMetadata(criteriaJson);
        } catch (JsonProcessingException e) {
            throw new IllegalArgumentException("Invalid criteria", e);
        }
    }
}
```

---

## สรุป Part 69: Database Advanced

### Performance Checklist

- [ ] ใช้ @EntityGraph แทน FetchType.EAGER เพื่อหลีกเลี่ยง N+1
- [ ] ใช้ QueryDSL สำหรับ complex dynamic queries
- [ ] กำหนด Locking strategy ให้เหมาะสม (Optimistic vs Pessimistic)
- [ ] Route read queries ไปยัง Read Replica
- [ ] ใช้ Batch operations สำหรับ bulk data
- [ ] Index JSON columns ที่ query บ่อย
- [ ] ใช้ Pagination แทนการโหลดข้อมูลทั้งหมด
- [ ] Monitor slow queries ด้วย PostgreSQL pg_stat_statements

### Database Anti-patterns

```java
// ❌ BAD: โหลดทั้ง collection แล้วกรองใน Java
List<Order> allOrders = orderRepository.findAll();
List<Order> filtered = allOrders.stream()
    .filter(o -> o.getStatus() == OrderStatus.PENDING)
    .collect(toList());

// ✅ GOOD: กรองใน Database
List<Order> pendingOrders = orderRepository.findByStatus(OrderStatus.PENDING);

// ❌ BAD: โหลด entity ทั้งหมดเพื่อ update เดียว
Product product = productRepository.findById(id).get();
product.setPrice(newPrice);
productRepository.save(product); // โหลด + update ทุก field

// ✅ GOOD: Partial update ด้วย @Modifying
@Modifying
@Query("UPDATE Product p SET p.price = :price WHERE p.id = :id")
int updatePrice(@Param("id") Long id, @Param("price") BigDecimal price);
```

---

*[← Part 68: Configuration Management](./part-68-configuration-management.md) | [Part 70: Cloud Native →](./part-70-cloud-native.md)*

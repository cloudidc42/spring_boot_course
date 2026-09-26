# Part 11: Spring Data JPA
## ขั้นตอนที่ 231-265

> **ระดับ:** พื้นฐาน-กลาง (Beginner-Intermediate)  
> **เวลาเรียน:** 5-6 ชั่วโมง  
> **เป้าหมาย:** ใช้ Spring Data JPA สำหรับ database operations อย่างมืออาชีพ

---

## ขั้นตอนที่ 231: JPA Concepts

```
JPA (Jakarta Persistence API) = สเปค / Interface
Hibernate = Implementation ที่ดีที่สุด (Spring Boot ใช้เป็น default)
Spring Data JPA = wrapper ที่ทำให้ใช้งาน JPA ง่ายขึ้น

Flow:
Code → Spring Data JPA → JPA API → Hibernate → JDBC → Database
```

---

## ขั้นตอนที่ 232: Entity Basics

```java
@Entity
@Table(name = "users", 
    indexes = {
        @Index(name = "idx_users_email", columnList = "email"),
        @Index(name = "idx_users_username", columnList = "username")
    },
    uniqueConstraints = {
        @UniqueConstraint(name = "uk_users_email", columnNames = "email"),
        @UniqueConstraint(name = "uk_users_username", columnNames = "username")
    }
)
@Getter @Setter @Builder @NoArgsConstructor @AllArgsConstructor
@EntityListeners(AuditingEntityListener.class)
public class User {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(name = "username", nullable = false, length = 50)
    private String username;
    
    @Column(name = "email", nullable = false)
    private String email;
    
    @Column(name = "password_hash", nullable = false)
    private String password;
    
    @Column(name = "first_name", length = 100)
    private String firstName;
    
    @Column(name = "last_name", length = 100)
    private String lastName;
    
    @Enumerated(EnumType.STRING)
    @Column(name = "role", nullable = false)
    private UserRole role = UserRole.USER;
    
    @Column(name = "is_active", nullable = false)
    private boolean active = true;
    
    @Column(name = "last_login_at")
    private LocalDateTime lastLoginAt;
    
    @CreatedDate
    @Column(name = "created_at", updatable = false)
    private LocalDateTime createdAt;
    
    @LastModifiedDate
    @Column(name = "updated_at")
    private LocalDateTime updatedAt;
    
    @CreatedBy
    @Column(name = "created_by", updatable = false)
    private String createdBy;
    
    @LastModifiedBy
    @Column(name = "updated_by")
    private String updatedBy;
    
    @Version  // Optimistic locking
    private Long version;
}
```

---

## ขั้นตอนที่ 233: BaseEntity ด้วย @MappedSuperclass

```java
@MappedSuperclass
@Getter
@EntityListeners(AuditingEntityListener.class)
public abstract class BaseEntity {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @CreatedDate
    @Column(updatable = false)
    private LocalDateTime createdAt;
    
    @LastModifiedDate
    private LocalDateTime updatedAt;
    
    @CreatedBy
    @Column(updatable = false)
    private String createdBy;
    
    @LastModifiedBy
    private String updatedBy;
    
    @Version
    private Long version;
}

// Entity extends BaseEntity
@Entity
@Table(name = "products")
@Getter @Setter @Builder @NoArgsConstructor @AllArgsConstructor
public class Product extends BaseEntity {
    
    @Column(nullable = false)
    private String name;
    
    private String description;
    
    @Column(nullable = false, precision = 10, scale = 2)
    private BigDecimal price;
    
    @Column(nullable = false)
    private Integer stock;
    
    @Enumerated(EnumType.STRING)
    private ProductStatus status = ProductStatus.ACTIVE;
}
```

---

## ขั้นตอนที่ 234: JPA Repository

```java
// JpaRepository<Entity, ID>
public interface UserRepository extends JpaRepository<User, Long> {
    
    // ===== Derived Query Methods =====
    
    // findBy + field name
    Optional<User> findByEmail(String email);
    Optional<User> findByUsername(String username);
    
    // Multiple conditions
    Optional<User> findByEmailAndActive(String email, boolean active);
    List<User> findByRoleAndActive(UserRole role, boolean active);
    
    // Boolean checks
    boolean existsByEmail(String email);
    boolean existsByUsername(String username);
    boolean existsByEmailOrUsername(String email, String username);
    
    // Count
    long countByRole(UserRole role);
    long countByActive(boolean active);
    
    // Delete
    void deleteByEmail(String email);
    
    // Like / Contains
    List<User> findByUsernameLike(String pattern);        // LIKE 'pattern'
    List<User> findByUsernameContaining(String keyword);  // LIKE '%keyword%'
    List<User> findByUsernameStartingWith(String prefix); // LIKE 'prefix%'
    
    // Comparison
    List<User> findByCreatedAtAfter(LocalDateTime date);
    List<User> findByCreatedAtBefore(LocalDateTime date);
    List<User> findByCreatedAtBetween(LocalDateTime start, LocalDateTime end);
    
    // Order By
    List<User> findByRoleOrderByCreatedAtDesc(UserRole role);
    List<User> findByActiveOrderByUsernameAsc(boolean active);
    
    // Top / First
    Optional<User> findFirstByOrderByCreatedAtDesc();  // Latest user
    List<User> findTop10ByRoleOrderByCreatedAtDesc(UserRole role);
    
    // Pagination
    Page<User> findByRole(UserRole role, Pageable pageable);
    Page<User> findByActive(boolean active, Pageable pageable);
    
    // ===== @Query Methods =====
    
    @Query("SELECT u FROM User u WHERE u.email = :email AND u.active = true")
    Optional<User> findActiveByEmail(@Param("email") String email);
    
    @Query("SELECT u FROM User u WHERE u.createdAt >= :since ORDER BY u.createdAt DESC")
    List<User> findRecentUsers(@Param("since") LocalDateTime since);
    
    // Native SQL
    @Query(value = "SELECT * FROM users WHERE email ILIKE %:keyword%", nativeQuery = true)
    List<User> searchByEmail(@Param("keyword") String keyword);
    
    // Count query for pagination
    @Query(value = "SELECT u FROM User u WHERE u.role = :role",
           countQuery = "SELECT count(u) FROM User u WHERE u.role = :role")
    Page<User> findByRoleWithCount(@Param("role") UserRole role, Pageable pageable);
    
    // ===== Modifying Queries =====
    
    @Modifying
    @Transactional
    @Query("UPDATE User u SET u.active = false WHERE u.id = :id")
    int deactivateUser(@Param("id") Long id);
    
    @Modifying
    @Transactional
    @Query("UPDATE User u SET u.lastLoginAt = :time WHERE u.id = :id")
    void updateLastLogin(@Param("id") Long id, @Param("time") LocalDateTime time);
    
    @Modifying
    @Transactional
    @Query("DELETE FROM User u WHERE u.active = false AND u.updatedAt < :before")
    int deleteInactiveUsers(@Param("before") LocalDateTime before);
}
```

---

## ขั้นตอนที่ 235: Pagination และ Sorting

```java
// Service
@Service
@RequiredArgsConstructor
public class UserService {
    
    private final UserRepository userRepository;
    
    public Page<UserResponse> findAll(int page, int size, String sortBy, String sortDir) {
        
        // Sorting
        Sort sort = sortDir.equalsIgnoreCase("asc") 
            ? Sort.by(sortBy).ascending()
            : Sort.by(sortBy).descending();
        
        // Pageable
        Pageable pageable = PageRequest.of(page, size, sort);
        
        // Query
        Page<User> userPage = userRepository.findAll(pageable);
        
        // Map to response
        return userPage.map(this::toResponse);
    }
    
    // Multiple sort fields
    public Page<UserResponse> findWithMultiSort(int page, int size) {
        Sort sort = Sort.by(
            Sort.Order.desc("createdAt"),
            Sort.Order.asc("username")
        );
        
        Pageable pageable = PageRequest.of(page, size, sort);
        return userRepository.findAll(pageable).map(this::toResponse);
    }
}
```

```java
// Controller
@RestController
@RequestMapping("/api/v1/users")
@RequiredArgsConstructor
public class UserController {
    
    private final UserService userService;
    
    @GetMapping
    public ResponseEntity<ApiResponse<PageResponse<UserResponse>>> findAll(
        @RequestParam(defaultValue = "0") int page,
        @RequestParam(defaultValue = "20") @Max(100) int size,
        @RequestParam(defaultValue = "createdAt") String sortBy,
        @RequestParam(defaultValue = "desc") String sortDir
    ) {
        Page<UserResponse> result = userService.findAll(page, size, sortBy, sortDir);
        return ResponseEntity.ok(ApiResponse.success(PageResponse.of(result)));
    }
}
```

```java
// PageResponse wrapper
public record PageResponse<T>(
    List<T> content,
    int page,
    int size,
    long totalElements,
    int totalPages,
    boolean first,
    boolean last
) {
    public static <T> PageResponse<T> of(Page<T> page) {
        return new PageResponse<>(
            page.getContent(),
            page.getNumber(),
            page.getSize(),
            page.getTotalElements(),
            page.getTotalPages(),
            page.isFirst(),
            page.isLast()
        );
    }
}
```

---

## ขั้นตอนที่ 236: JPA Specifications (Dynamic Queries)

```java
// UserSpec
public class UserSpec {
    
    public static Specification<User> withUsername(String username) {
        return (root, query, cb) -> username == null ? null 
            : cb.like(cb.lower(root.get("username")), "%" + username.toLowerCase() + "%");
    }
    
    public static Specification<User> withEmail(String email) {
        return (root, query, cb) -> email == null ? null
            : cb.like(cb.lower(root.get("email")), "%" + email.toLowerCase() + "%");
    }
    
    public static Specification<User> withRole(UserRole role) {
        return (root, query, cb) -> role == null ? null
            : cb.equal(root.get("role"), role);
    }
    
    public static Specification<User> isActive(Boolean active) {
        return (root, query, cb) -> active == null ? null
            : cb.equal(root.get("active"), active);
    }
    
    public static Specification<User> createdAfter(LocalDateTime date) {
        return (root, query, cb) -> date == null ? null
            : cb.greaterThanOrEqualTo(root.get("createdAt"), date);
    }
}

// Repository extends JpaSpecificationExecutor
public interface UserRepository extends JpaRepository<User, Long>, JpaSpecificationExecutor<User> { }

// Service
public Page<UserResponse> search(UserSearchRequest request, Pageable pageable) {
    Specification<User> spec = Specification
        .where(UserSpec.withUsername(request.username()))
        .and(UserSpec.withEmail(request.email()))
        .and(UserSpec.withRole(request.role()))
        .and(UserSpec.isActive(request.active()))
        .and(UserSpec.createdAfter(request.createdAfter()));
    
    return userRepository.findAll(spec, pageable).map(this::toResponse);
}
```

---

## ขั้นตอนที่ 237: Entity Relationships

### One-to-Many

```java
// User has many Orders
@Entity
public class User extends BaseEntity {
    // ...
    
    @OneToMany(mappedBy = "user", cascade = CascadeType.ALL, orphanRemoval = true)
    @OrderBy("createdAt DESC")
    private List<Order> orders = new ArrayList<>();
    
    // Helper methods
    public void addOrder(Order order) {
        orders.add(order);
        order.setUser(this);
    }
    
    public void removeOrder(Order order) {
        orders.remove(order);
        order.setUser(null);
    }
}

@Entity
@Table(name = "orders")
public class Order extends BaseEntity {
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id", nullable = false)
    private User user;
    
    @Column(nullable = false, precision = 10, scale = 2)
    private BigDecimal totalAmount;
    
    @Enumerated(EnumType.STRING)
    private OrderStatus status = OrderStatus.PENDING;
    
    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<OrderItem> items = new ArrayList<>();
}
```

### Many-to-Many

```java
// User has many Roles, Role belongs to many Users
@Entity
public class User extends BaseEntity {
    
    @ManyToMany(fetch = FetchType.LAZY)
    @JoinTable(
        name = "user_roles",
        joinColumns = @JoinColumn(name = "user_id"),
        inverseJoinColumns = @JoinColumn(name = "role_id")
    )
    private Set<Role> roles = new HashSet<>();
}

@Entity
@Table(name = "roles")
public class Role extends BaseEntity {
    
    @Column(unique = true, nullable = false)
    private String name;
    
    @ManyToMany(mappedBy = "roles")
    private Set<User> users = new HashSet<>();
}
```

### One-to-One

```java
@Entity
public class User extends BaseEntity {
    
    @OneToOne(mappedBy = "user", cascade = CascadeType.ALL, fetch = FetchType.LAZY)
    private UserProfile profile;
}

@Entity
@Table(name = "user_profiles")
public class UserProfile extends BaseEntity {
    
    @OneToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id", nullable = false, unique = true)
    private User user;
    
    private String bio;
    private String avatarUrl;
    private String phone;
}
```

---

## ขั้นตอนที่ 238: Fetch Strategies และ N+1 Problem

```java
// ❌ N+1 Problem
List<Order> orders = orderRepository.findAll();
for (Order order : orders) {
    // สร้าง SQL query ใหม่ทุก iteration!
    System.out.println(order.getUser().getUsername()); 
}

// ✅ Solution 1: JOIN FETCH
@Query("SELECT o FROM Order o JOIN FETCH o.user WHERE o.status = :status")
List<Order> findByStatusWithUser(@Param("status") OrderStatus status);

// ✅ Solution 2: @EntityGraph
@EntityGraph(attributePaths = {"user", "items"})
List<Order> findByStatus(OrderStatus status);

// ✅ Solution 3: @BatchSize
@OneToMany(mappedBy = "order")
@BatchSize(size = 100)
private List<OrderItem> items;

// ✅ Solution 4: Projections (fetch only needed fields)
public interface OrderSummary {
    Long getId();
    String getStatus();
    BigDecimal getTotalAmount();
    String getUserUsername();  // Nested: user.username → getUserUsername()
}

List<OrderSummary> findProjectedByStatus(OrderStatus status);
```

---

## ขั้นตอนที่ 239: Transactions

```java
@Service
@RequiredArgsConstructor
@Transactional(readOnly = true)  // Default: read-only ประหยัด resources
public class OrderService {
    
    private final OrderRepository orderRepository;
    private final ProductRepository productRepository;
    private final UserRepository userRepository;
    
    // Read operations inherit readOnly = true
    public OrderResponse findById(Long id) {
        return orderRepository.findById(id)
            .map(this::toResponse)
            .orElseThrow(() -> new ResourceNotFoundException("Order", "id", id));
    }
    
    // Write operations override to readOnly = false
    @Transactional
    public OrderResponse create(Long userId, CreateOrderRequest request) {
        User user = userRepository.findById(userId)
            .orElseThrow(() -> new ResourceNotFoundException("User", "id", userId));
        
        Order order = new Order();
        order.setUser(user);
        order.setStatus(OrderStatus.PENDING);
        
        BigDecimal total = BigDecimal.ZERO;
        
        for (var item : request.items()) {
            Product product = productRepository.findById(item.productId())
                .orElseThrow(() -> new ResourceNotFoundException("Product", "id", item.productId()));
            
            if (product.getStock() < item.quantity()) {
                throw new InsufficientStockException(product.getName(), item.quantity());
            }
            
            // Reduce stock
            product.setStock(product.getStock() - item.quantity());
            productRepository.save(product);
            
            // Add item
            OrderItem orderItem = new OrderItem();
            orderItem.setProduct(product);
            orderItem.setQuantity(item.quantity());
            orderItem.setPrice(product.getPrice());
            order.addItem(orderItem);
            
            total = total.add(product.getPrice().multiply(BigDecimal.valueOf(item.quantity())));
        }
        
        order.setTotalAmount(total);
        Order saved = orderRepository.save(order);
        
        return toResponse(saved);
    }
    
    // Transaction with specific propagation
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void logOrderCreated(Long orderId) {
        // Creates a new separate transaction
        auditLogRepository.save(new AuditLog("ORDER_CREATED", orderId));
    }
    
    // Rollback only for specific exceptions
    @Transactional(rollbackFor = Exception.class)
    public void riskyOperation() { }
    
    // No rollback for certain exceptions
    @Transactional(noRollbackFor = ValidationException.class)
    public void partiallyRolledBack() { }
}
```

---

## ขั้นตอนที่ 240: Custom Repository Implementation

```java
// Custom interface
public interface UserRepositoryCustom {
    List<User> searchUsers(String keyword, UserRole role, Boolean active);
    Map<UserRole, Long> countUsersByRole();
}

// Implementation
@RequiredArgsConstructor
public class UserRepositoryCustomImpl implements UserRepositoryCustom {
    
    private final EntityManager em;
    
    @Override
    public List<User> searchUsers(String keyword, UserRole role, Boolean active) {
        CriteriaBuilder cb = em.getCriteriaBuilder();
        CriteriaQuery<User> query = cb.createQuery(User.class);
        Root<User> root = query.from(User.class);
        
        List<Predicate> predicates = new ArrayList<>();
        
        if (keyword != null && !keyword.isBlank()) {
            String pattern = "%" + keyword.toLowerCase() + "%";
            predicates.add(cb.or(
                cb.like(cb.lower(root.get("username")), pattern),
                cb.like(cb.lower(root.get("email")), pattern)
            ));
        }
        
        if (role != null) {
            predicates.add(cb.equal(root.get("role"), role));
        }
        
        if (active != null) {
            predicates.add(cb.equal(root.get("active"), active));
        }
        
        query.where(predicates.toArray(new Predicate[0]));
        query.orderBy(cb.desc(root.get("createdAt")));
        
        return em.createQuery(query).getResultList();
    }
    
    @Override
    public Map<UserRole, Long> countUsersByRole() {
        String jpql = "SELECT u.role, COUNT(u) FROM User u GROUP BY u.role";
        
        return em.createQuery(jpql, Object[].class)
            .getResultList()
            .stream()
            .collect(Collectors.toMap(
                row -> (UserRole) row[0],
                row -> (Long) row[1]
            ));
    }
}

// Main repository extends both
public interface UserRepository extends JpaRepository<User, Long>, 
                                        JpaSpecificationExecutor<User>,
                                        UserRepositoryCustom {
    // ...
}
```

---

## ขั้นตอนที่ 241: Auditing Configuration

```java
@Configuration
@EnableJpaAuditing(auditorAwareRef = "auditorAware")
@EnableJpaRepositories
public class JpaConfig {
    
    @Bean
    AuditorAware<String> auditorAware() {
        return () -> Optional.ofNullable(SecurityContextHolder.getContext())
            .map(SecurityContext::getAuthentication)
            .filter(Authentication::isAuthenticated)
            .map(Authentication::getName)
            .or(() -> Optional.of("system"));
    }
}
```

---

## ขั้นตอนที่ 242-265: Complete Product Repository Example

```java
// Product Entity
@Entity
@Table(name = "products",
    indexes = {
        @Index(columnList = "name"),
        @Index(columnList = "category_id"),
        @Index(columnList = "status,price")
    }
)
@Getter @Setter @Builder @NoArgsConstructor @AllArgsConstructor
@EntityListeners(AuditingEntityListener.class)
public class Product extends BaseEntity {
    
    @Column(nullable = false, length = 255)
    private String name;
    
    @Column(columnDefinition = "TEXT")
    private String description;
    
    @Column(nullable = false, precision = 10, scale = 2)
    private BigDecimal price;
    
    @Column(nullable = false)
    @PositiveOrZero
    private Integer stock;
    
    @Column(name = "image_url")
    private String imageUrl;
    
    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private ProductStatus status = ProductStatus.ACTIVE;
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "category_id")
    private Category category;
    
    @OneToMany(mappedBy = "product", cascade = CascadeType.ALL)
    private List<ProductImage> images = new ArrayList<>();
    
    @Column(name = "view_count", nullable = false)
    private Long viewCount = 0L;
}

// Product Repository
public interface ProductRepository extends JpaRepository<Product, Long>, 
                                           JpaSpecificationExecutor<Product> {
    
    Page<Product> findByStatus(ProductStatus status, Pageable pageable);
    
    Page<Product> findByCategory(Category category, Pageable pageable);
    
    @Query("SELECT p FROM Product p WHERE p.price BETWEEN :min AND :max AND p.status = 'ACTIVE'")
    Page<Product> findByPriceRange(@Param("min") BigDecimal min, 
                                   @Param("max") BigDecimal max, 
                                   Pageable pageable);
    
    @Query("SELECT p FROM Product p WHERE p.stock <= :threshold AND p.status = 'ACTIVE'")
    List<Product> findLowStockProducts(@Param("threshold") int threshold);
    
    @Modifying
    @Query("UPDATE Product p SET p.viewCount = p.viewCount + 1 WHERE p.id = :id")
    void incrementViewCount(@Param("id") Long id);
    
    // Search
    @Query("""
        SELECT p FROM Product p
        WHERE (:keyword IS NULL OR LOWER(p.name) LIKE LOWER(CONCAT('%', :keyword, '%')))
        AND (:status IS NULL OR p.status = :status)
        AND (:minPrice IS NULL OR p.price >= :minPrice)
        AND (:maxPrice IS NULL OR p.price <= :maxPrice)
        """)
    Page<Product> search(
        @Param("keyword") String keyword,
        @Param("status") ProductStatus status,
        @Param("minPrice") BigDecimal minPrice,
        @Param("maxPrice") BigDecimal maxPrice,
        Pageable pageable
    );
}

// Product Service
@Service
@RequiredArgsConstructor
@Transactional(readOnly = true)
@Slf4j
public class ProductService {
    
    private final ProductRepository productRepository;
    private final CategoryRepository categoryRepository;
    private final ProductMapper productMapper;
    
    public Page<ProductResponse> search(ProductSearchRequest request, Pageable pageable) {
        return productRepository.search(
            request.keyword(),
            request.status(),
            request.minPrice(),
            request.maxPrice(),
            pageable
        ).map(productMapper::toResponse);
    }
    
    public ProductDetailResponse findById(Long id) {
        Product product = productRepository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("Product", "id", id));
        
        // Async: increment view count
        productRepository.incrementViewCount(id);
        
        return productMapper.toDetailResponse(product);
    }
    
    @Transactional
    public ProductResponse create(CreateProductRequest request) {
        Category category = null;
        if (request.categoryId() != null) {
            category = categoryRepository.findById(request.categoryId())
                .orElseThrow(() -> new ResourceNotFoundException("Category", "id", request.categoryId()));
        }
        
        Product product = productMapper.toEntity(request);
        product.setCategory(category);
        
        Product saved = productRepository.save(product);
        log.info("Product created: {} (id={})", saved.getName(), saved.getId());
        
        return productMapper.toResponse(saved);
    }
    
    @Transactional
    public ProductResponse update(Long id, UpdateProductRequest request) {
        Product product = productRepository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("Product", "id", id));
        
        productMapper.updateFromRequest(request, product);
        
        if (request.categoryId() != null) {
            Category category = categoryRepository.findById(request.categoryId())
                .orElseThrow(() -> new ResourceNotFoundException("Category", "id", request.categoryId()));
            product.setCategory(category);
        }
        
        return productMapper.toResponse(productRepository.save(product));
    }
    
    @Transactional
    public void delete(Long id) {
        if (!productRepository.existsById(id)) {
            throw new ResourceNotFoundException("Product", "id", id);
        }
        productRepository.deleteById(id);
    }
    
    public List<ProductResponse> findLowStock(int threshold) {
        return productRepository.findLowStockProducts(threshold)
            .stream()
            .map(productMapper::toResponse)
            .toList();
    }
}
```

---

*[← Part 10: Testing](./part-10-testing-basics.md) | [Part 12: Database Setup →](./part-12-database-setup.md)*

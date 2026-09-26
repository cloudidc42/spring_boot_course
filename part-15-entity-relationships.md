# Part 15: Entity Relationships
## ขั้นตอนที่ 361-395

> **ระดับ:** กลาง (Intermediate)  
> **เวลาเรียน:** 5-6 ชั่วโมง  
> **เป้าหมาย:** จัดการ Entity Relationships อย่างถูกต้องและมีประสิทธิภาพ

---

## ขั้นตอนที่ 361: One-to-Many - Order System

```java
// Order Entity
@Entity
@Table(name = "orders")
@Getter @Setter @Builder @NoArgsConstructor @AllArgsConstructor
@EntityListeners(AuditingEntityListener.class)
public class Order extends BaseEntity {
    
    @Column(name = "order_number", unique = true, nullable = false)
    private String orderNumber;
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id", nullable = false)
    private User user;
    
    @Column(nullable = false, precision = 10, scale = 2)
    private BigDecimal totalAmount;
    
    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private OrderStatus status = OrderStatus.PENDING;
    
    @Column(name = "shipping_address", columnDefinition = "TEXT")
    private String shippingAddress;
    
    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
    @OrderBy("id ASC")
    private List<OrderItem> items = new ArrayList<>();
    
    // Helper method
    public void addItem(OrderItem item) {
        items.add(item);
        item.setOrder(this);
        recalculateTotal();
    }
    
    public void removeItem(OrderItem item) {
        items.remove(item);
        item.setOrder(null);
        recalculateTotal();
    }
    
    private void recalculateTotal() {
        this.totalAmount = items.stream()
            .map(item -> item.getPrice().multiply(BigDecimal.valueOf(item.getQuantity())))
            .reduce(BigDecimal.ZERO, BigDecimal::add);
    }
}

// OrderItem Entity
@Entity
@Table(name = "order_items")
@Getter @Setter @Builder @NoArgsConstructor @AllArgsConstructor
public class OrderItem {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "order_id", nullable = false)
    private Order order;
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "product_id", nullable = false)
    private Product product;
    
    @Column(nullable = false)
    @Min(1)
    private Integer quantity;
    
    @Column(nullable = false, precision = 10, scale = 2)
    private BigDecimal price;  // snapshot ราคา ณ ขณะสั่ง
    
    @Column(precision = 10, scale = 2)
    public BigDecimal getSubtotal() {
        return price.multiply(BigDecimal.valueOf(quantity));
    }
}
```

---

## ขั้นตอนที่ 362: Many-to-Many - Tags

```java
// Tag Entity
@Entity
@Table(name = "tags")
@Getter @Setter @Builder @NoArgsConstructor @AllArgsConstructor
public class Tag extends BaseEntity {
    
    @Column(unique = true, nullable = false)
    private String name;
    
    @Column(unique = true, nullable = false)
    private String slug;
    
    @ManyToMany(mappedBy = "tags")
    private Set<Product> products = new HashSet<>();
}

// Product Entity - add tags
@Entity
public class Product extends BaseEntity {
    // ...
    
    @ManyToMany(fetch = FetchType.LAZY)
    @JoinTable(
        name = "product_tags",
        joinColumns = @JoinColumn(name = "product_id"),
        inverseJoinColumns = @JoinColumn(name = "tag_id")
    )
    private Set<Tag> tags = new HashSet<>();
    
    public void addTag(Tag tag) {
        tags.add(tag);
        tag.getProducts().add(this);
    }
    
    public void removeTag(Tag tag) {
        tags.remove(tag);
        tag.getProducts().remove(this);
    }
}
```

---

## ขั้นตอนที่ 363: One-to-One - User Profile

```java
// UserProfile Entity
@Entity
@Table(name = "user_profiles")
@Getter @Setter @Builder @NoArgsConstructor @AllArgsConstructor
@EntityListeners(AuditingEntityListener.class)
public class UserProfile extends BaseEntity {
    
    @OneToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id", unique = true)
    private User user;
    
    private String bio;
    
    @Column(name = "avatar_url")
    private String avatarUrl;
    
    private String phone;
    
    @Column(name = "birth_date")
    private LocalDate birthDate;
    
    @Embedded
    private Address address;
    
    @Column(name = "social_links", columnDefinition = "jsonb")
    @JdbcTypeCode(SqlTypes.JSON)
    private Map<String, String> socialLinks = new HashMap<>();
}

// User Entity
@Entity
public class User extends BaseEntity {
    // ...
    
    @OneToOne(mappedBy = "user", cascade = CascadeType.ALL, fetch = FetchType.LAZY)
    private UserProfile profile;
    
    public void setProfile(UserProfile profile) {
        if (profile != null) {
            profile.setUser(this);
        }
        this.profile = profile;
    }
}

// Embeddable
@Embeddable
@Getter @Setter @Builder @NoArgsConstructor @AllArgsConstructor
public class Address {
    
    @Column(name = "street_address")
    private String street;
    
    private String city;
    
    @Column(name = "state_province")
    private String state;
    
    @Column(name = "postal_code")
    private String postalCode;
    
    @Column(name = "country_code")
    private String country;
}
```

---

## ขั้นตอนที่ 364: Inheritance - Single Table

```java
// Base notification
@Entity
@Table(name = "notifications")
@Inheritance(strategy = InheritanceType.SINGLE_TABLE)
@DiscriminatorColumn(name = "type", discriminatorType = DiscriminatorType.STRING)
@Getter @Setter
public abstract class Notification extends BaseEntity {
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id")
    private User user;
    
    private String title;
    private String message;
    private boolean read = false;
    private LocalDateTime readAt;
}

@Entity
@DiscriminatorValue("EMAIL")
public class EmailNotification extends Notification {
    private String emailAddress;
    private String subject;
}

@Entity
@DiscriminatorValue("PUSH")
public class PushNotification extends Notification {
    private String deviceToken;
    private String payload;
}

@Entity
@DiscriminatorValue("SMS")
public class SmsNotification extends Notification {
    private String phoneNumber;
    private int characterCount;
}
```

---

## ขั้นตอนที่ 365: Inheritance - Table Per Class

```java
@Entity
@Inheritance(strategy = InheritanceType.TABLE_PER_CLASS)
public abstract class Payment extends BaseEntity {
    private BigDecimal amount;
    private PaymentStatus status;
    private LocalDateTime paidAt;
}

@Entity
@Table(name = "credit_card_payments")
public class CreditCardPayment extends Payment {
    private String cardNumber;   // last 4 digits
    private String cardBrand;
    private String expiryMonth;
    private String expiryYear;
}

@Entity
@Table(name = "bank_transfer_payments")
public class BankTransferPayment extends Payment {
    private String bankCode;
    private String accountNumber;
    private String referenceCode;
}
```

---

## ขั้นตอนที่ 366: Composite Key

```java
// Composite Key Class
@Embeddable
@Getter @Setter @EqualsAndHashCode
@NoArgsConstructor @AllArgsConstructor
public class UserRoleId implements Serializable {
    
    @Column(name = "user_id")
    private Long userId;
    
    @Column(name = "role_id")
    private Long roleId;
}

// Entity with Composite Key
@Entity
@Table(name = "user_roles")
@Getter @Setter
public class UserRole {
    
    @EmbeddedId
    private UserRoleId id;
    
    @ManyToOne(fetch = FetchType.LAZY)
    @MapsId("userId")
    @JoinColumn(name = "user_id")
    private User user;
    
    @ManyToOne(fetch = FetchType.LAZY)
    @MapsId("roleId")
    @JoinColumn(name = "role_id")
    private Role role;
    
    private LocalDateTime assignedAt = LocalDateTime.now();
    private String assignedBy;
}
```

---

## ขั้นตอนที่ 367: Fetch Optimization

```java
// Repository with optimized queries
public interface OrderRepository extends JpaRepository<Order, Long> {
    
    // Load order with items in one query
    @Query("SELECT o FROM Order o JOIN FETCH o.items WHERE o.id = :id")
    Optional<Order> findByIdWithItems(@Param("id") Long id);
    
    // Load order with items and products
    @Query("""
        SELECT DISTINCT o FROM Order o 
        JOIN FETCH o.items i 
        JOIN FETCH i.product 
        WHERE o.user.id = :userId
        """)
    List<Order> findByUserIdWithItemsAndProducts(@Param("userId") Long userId);
    
    // Use @EntityGraph
    @EntityGraph(attributePaths = {"items", "items.product", "user"})
    Optional<Order> findWithDetailsById(Long id);
    
    // Named EntityGraph
    @NamedEntityGraph(
        name = "Order.withItems",
        attributeNodes = @NamedAttributeNode(value = "items", 
            subgraph = "items.product"),
        subgraphs = @NamedSubgraph(
            name = "items.product",
            attributeNodes = @NamedAttributeNode("product")
        )
    )
    @EntityGraph("Order.withItems")
    List<Order> findByStatus(OrderStatus status);
}
```

---

## ขั้นตอนที่ 368: Cascade Operations

```java
// Cascade Types
@OneToMany(
    mappedBy = "order", 
    cascade = {
        CascadeType.PERSIST,  // save parent → save children
        CascadeType.MERGE,    // update parent → update children
        CascadeType.REMOVE,   // delete parent → delete children
        CascadeType.REFRESH,  // refresh parent → refresh children
        CascadeType.DETACH    // detach parent → detach children
    },
    // CascadeType.ALL = all above
    orphanRemoval = true  // remove orphan children
)
private List<OrderItem> items;

// ระวัง! CascadeType.REMOVE อันตราย
// ถ้า User มี Orders มาก การลบ User จะลบ Orders ทั้งหมด
// ใช้ soft delete แทน
```

---

## ขั้นตอนที่ 369-395: Complete E-Commerce Data Model

```sql
-- Final database schema
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    username VARCHAR(50) UNIQUE NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    role VARCHAR(20) NOT NULL DEFAULT 'USER',
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP,
    version BIGINT DEFAULT 0
);

CREATE TABLE user_profiles (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT UNIQUE REFERENCES users(id) ON DELETE CASCADE,
    bio TEXT,
    avatar_url VARCHAR(500),
    phone VARCHAR(20),
    birth_date DATE,
    street_address VARCHAR(255),
    city VARCHAR(100),
    postal_code VARCHAR(20),
    country_code CHAR(2)
);

CREATE TABLE categories (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(100) UNIQUE NOT NULL,
    slug VARCHAR(100) UNIQUE NOT NULL,
    parent_id BIGINT REFERENCES categories(id),
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE products (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    price DECIMAL(10,2) NOT NULL CHECK (price > 0),
    stock INTEGER NOT NULL DEFAULT 0 CHECK (stock >= 0),
    image_url VARCHAR(500),
    status VARCHAR(20) DEFAULT 'ACTIVE',
    category_id BIGINT REFERENCES categories(id),
    view_count BIGINT DEFAULT 0,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP,
    version BIGINT DEFAULT 0
);

CREATE TABLE tags (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(50) UNIQUE NOT NULL,
    slug VARCHAR(50) UNIQUE NOT NULL
);

CREATE TABLE product_tags (
    product_id BIGINT REFERENCES products(id) ON DELETE CASCADE,
    tag_id BIGINT REFERENCES tags(id) ON DELETE CASCADE,
    PRIMARY KEY (product_id, tag_id)
);

CREATE TABLE orders (
    id BIGSERIAL PRIMARY KEY,
    order_number VARCHAR(50) UNIQUE NOT NULL,
    user_id BIGINT REFERENCES users(id),
    total_amount DECIMAL(10,2) NOT NULL,
    status VARCHAR(20) DEFAULT 'PENDING',
    shipping_address TEXT,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP,
    version BIGINT DEFAULT 0
);

CREATE TABLE order_items (
    id BIGSERIAL PRIMARY KEY,
    order_id BIGINT REFERENCES orders(id) ON DELETE CASCADE,
    product_id BIGINT REFERENCES products(id),
    quantity INTEGER NOT NULL CHECK (quantity > 0),
    price DECIMAL(10,2) NOT NULL
);
```

```java
// OrderService - creating order with items
@Service
@RequiredArgsConstructor
@Transactional
@Slf4j
public class OrderService {
    
    private final OrderRepository orderRepository;
    private final UserRepository userRepository;
    private final ProductRepository productRepository;
    private final OrderMapper orderMapper;
    
    public OrderResponse create(Long userId, CreateOrderRequest request) {
        User user = userRepository.findById(userId)
            .orElseThrow(() -> new ResourceNotFoundException("User", "id", userId));
        
        Order order = new Order();
        order.setUser(user);
        order.setOrderNumber(generateOrderNumber());
        order.setStatus(OrderStatus.PENDING);
        order.setShippingAddress(request.shippingAddress());
        
        for (var itemReq : request.items()) {
            Product product = productRepository.findById(itemReq.productId())
                .orElseThrow(() -> new ResourceNotFoundException("Product", "id", itemReq.productId()));
            
            if (product.getStock() < itemReq.quantity()) {
                throw new InsufficientStockException(product.getName(), itemReq.quantity());
            }
            
            product.setStock(product.getStock() - itemReq.quantity());
            
            OrderItem item = OrderItem.builder()
                .product(product)
                .quantity(itemReq.quantity())
                .price(product.getPrice())
                .build();
            
            order.addItem(item);  // recalculates total
        }
        
        Order saved = orderRepository.save(order);
        log.info("Order created: {} for user {}", saved.getOrderNumber(), userId);
        
        return orderMapper.toResponse(saved);
    }
    
    @Transactional(readOnly = true)
    public OrderDetailResponse findById(Long id) {
        Order order = orderRepository.findByIdWithItems(id)
            .orElseThrow(() -> new ResourceNotFoundException("Order", "id", id));
        return orderMapper.toDetailResponse(order);
    }
    
    @Transactional
    public OrderResponse updateStatus(Long id, OrderStatus newStatus) {
        Order order = orderRepository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("Order", "id", id));
        
        validateStatusTransition(order.getStatus(), newStatus);
        order.setStatus(newStatus);
        
        return orderMapper.toResponse(orderRepository.save(order));
    }
    
    private void validateStatusTransition(OrderStatus current, OrderStatus next) {
        Map<OrderStatus, Set<OrderStatus>> allowed = Map.of(
            OrderStatus.PENDING, Set.of(OrderStatus.CONFIRMED, OrderStatus.CANCELLED),
            OrderStatus.CONFIRMED, Set.of(OrderStatus.SHIPPED, OrderStatus.CANCELLED),
            OrderStatus.SHIPPED, Set.of(OrderStatus.DELIVERED),
            OrderStatus.DELIVERED, Set.of(),
            OrderStatus.CANCELLED, Set.of()
        );
        
        if (!allowed.getOrDefault(current, Set.of()).contains(next)) {
            throw new BusinessException(
                String.format("Cannot transition order from %s to %s", current, next)
            );
        }
    }
    
    private String generateOrderNumber() {
        return "ORD-" + LocalDate.now().format(DateTimeFormatter.BASIC_ISO_DATE) 
            + "-" + UUID.randomUUID().toString().substring(0, 8).toUpperCase();
    }
}
```

---

*[← Part 14: Validation](./part-14-validation.md) | [Part 16: Spring Security →](./part-16-spring-security.md)*

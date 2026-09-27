# Part 101: โปรเจค 1-5 — E-Commerce & FinTech
## โปรเจคที่ 1-5

> **ระดับ:** ระดับโลก | **เวลาเรียนรู้:** 10-15 ชั่วโมง | **โปรเจค:** 5 โปรเจคสมบูรณ์

ในส่วนนี้เราจะสร้างระบบ E-Commerce และ FinTech ที่ใช้งานได้จริงในระดับ Production โดยแต่ละโปรเจคจะมีโค้ดครบถ้วน พร้อม Entity, Service, Controller, และ Database Migration

---

## โปรเจคที่ 1: Full E-Commerce Platform

### ภาพรวมระบบ

ระบบ E-Commerce ครบวงจรที่รองรับการจัดการสินค้า ตะกร้าสินค้า การสั่งซื้อ การชำระเงินผ่าน Stripe และการติดตามสถานะออเดอร์ ระบบนี้ออกแบบมาให้รองรับการใช้งานจริงโดยมีการลดสต็อกสินค้าอัตโนมัติเมื่อมีการสั่งซื้อ

### โครงสร้าง Entity

```java
// Product.java
@Entity
@Table(name = "products")
@Data
@NoArgsConstructor
@AllArgsConstructor
public class Product {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String name;

    @Column(columnDefinition = "TEXT")
    private String description;

    @Column(nullable = false, precision = 10, scale = 2)
    private BigDecimal price;

    @Column(nullable = false)
    private Integer stockQuantity;

    @Column(unique = true)
    private String sku;

    private String imageUrl;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "category_id")
    private Category category;

    @Column(nullable = false)
    private Boolean active = true;

    @CreatedDate
    private LocalDateTime createdAt;

    @LastModifiedDate
    private LocalDateTime updatedAt;
}

// Category.java
@Entity
@Table(name = "categories")
@Data
@NoArgsConstructor
public class Category {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, unique = true)
    private String name;

    private String description;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "parent_id")
    private Category parent;

    @OneToMany(mappedBy = "parent")
    private List<Category> children = new ArrayList<>();
}

// Cart.java
@Entity
@Table(name = "carts")
@Data
@NoArgsConstructor
public class Cart {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @OneToOne
    @JoinColumn(name = "user_id", unique = true)
    private User user;

    @OneToMany(mappedBy = "cart", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<CartItem> items = new ArrayList<>();

    @CreatedDate
    private LocalDateTime createdAt;

    @LastModifiedDate
    private LocalDateTime updatedAt;

    public BigDecimal getTotalAmount() {
        return items.stream()
            .map(item -> item.getProduct().getPrice().multiply(BigDecimal.valueOf(item.getQuantity())))
            .reduce(BigDecimal.ZERO, BigDecimal::add);
    }
}

// CartItem.java
@Entity
@Table(name = "cart_items")
@Data
@NoArgsConstructor
public class CartItem {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "cart_id")
    private Cart cart;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "product_id")
    private Product product;

    @Column(nullable = false)
    private Integer quantity;
}

// Order.java
@Entity
@Table(name = "orders")
@Data
@NoArgsConstructor
public class Order {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(unique = true, nullable = false)
    private String orderNumber;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id")
    private User user;

    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL)
    private List<OrderItem> items = new ArrayList<>();

    @Enumerated(EnumType.STRING)
    private OrderStatus status = OrderStatus.PENDING;

    @Column(precision = 10, scale = 2)
    private BigDecimal totalAmount;

    @Column(precision = 10, scale = 2)
    private BigDecimal shippingCost;

    @Embedded
    private ShippingAddress shippingAddress;

    @OneToOne(mappedBy = "order", cascade = CascadeType.ALL)
    private Payment payment;

    @CreatedDate
    private LocalDateTime createdAt;

    @LastModifiedDate
    private LocalDateTime updatedAt;
}

// OrderStatus Enum
public enum OrderStatus {
    PENDING, CONFIRMED, PROCESSING, SHIPPED, DELIVERED, CANCELLED, REFUNDED
}

// OrderItem.java
@Entity
@Table(name = "order_items")
@Data
@NoArgsConstructor
public class OrderItem {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne
    @JoinColumn(name = "order_id")
    private Order order;

    @ManyToOne
    @JoinColumn(name = "product_id")
    private Product product;

    private Integer quantity;

    @Column(precision = 10, scale = 2)
    private BigDecimal unitPrice;

    @Column(precision = 10, scale = 2)
    private BigDecimal subtotal;
}

// Payment.java
@Entity
@Table(name = "payments")
@Data
@NoArgsConstructor
public class Payment {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @OneToOne
    @JoinColumn(name = "order_id")
    private Order order;

    private String stripePaymentIntentId;
    private String stripeChargeId;

    @Enumerated(EnumType.STRING)
    private PaymentStatus status = PaymentStatus.PENDING;

    @Column(precision = 10, scale = 2)
    private BigDecimal amount;

    private String currency = "THB";

    @CreatedDate
    private LocalDateTime createdAt;
}

// ShippingAddress.java (Embeddable)
@Embeddable
@Data
@NoArgsConstructor
public class ShippingAddress {
    private String recipientName;
    private String addressLine1;
    private String addressLine2;
    private String city;
    private String province;
    private String postalCode;
    private String phoneNumber;
}
```

### Repository Layer

```java
// ProductRepository.java
@Repository
public interface ProductRepository extends JpaRepository<Product, Long> {

    Page<Product> findByActiveTrue(Pageable pageable);

    Page<Product> findByCategoryIdAndActiveTrue(Long categoryId, Pageable pageable);

    @Query("SELECT p FROM Product p WHERE p.active = true AND " +
           "(LOWER(p.name) LIKE LOWER(CONCAT('%', :keyword, '%')) OR " +
           "LOWER(p.description) LIKE LOWER(CONCAT('%', :keyword, '%')))")
    Page<Product> searchProducts(@Param("keyword") String keyword, Pageable pageable);

    @Query("SELECT p FROM Product p WHERE p.stockQuantity <= :threshold AND p.active = true")
    List<Product> findLowStockProducts(@Param("threshold") int threshold);

    @Modifying
    @Query("UPDATE Product p SET p.stockQuantity = p.stockQuantity - :quantity WHERE p.id = :id AND p.stockQuantity >= :quantity")
    int deductStock(@Param("id") Long id, @Param("quantity") int quantity);

    Optional<Product> findBySku(String sku);
}

// OrderRepository.java
@Repository
public interface OrderRepository extends JpaRepository<Order, Long> {

    Page<Order> findByUserIdOrderByCreatedAtDesc(Long userId, Pageable pageable);

    Optional<Order> findByOrderNumber(String orderNumber);

    @Query("SELECT o FROM Order o WHERE o.status = :status ORDER BY o.createdAt DESC")
    Page<Order> findByStatus(@Param("status") OrderStatus status, Pageable pageable);

    @Query("SELECT o FROM Order o WHERE o.user.id = :userId AND o.status = :status")
    List<Order> findByUserIdAndStatus(@Param("userId") Long userId, @Param("status") OrderStatus status);

    @Query("SELECT SUM(o.totalAmount) FROM Order o WHERE o.status = 'DELIVERED' AND " +
           "o.createdAt BETWEEN :startDate AND :endDate")
    BigDecimal calculateRevenue(@Param("startDate") LocalDateTime startDate,
                                @Param("endDate") LocalDateTime endDate);
}

// CartRepository.java
@Repository
public interface CartRepository extends JpaRepository<Cart, Long> {
    Optional<Cart> findByUserId(Long userId);
}
```

### Service Layer

```java
// ProductService.java
@Service
@Transactional
@RequiredArgsConstructor
public class ProductService {

    private final ProductRepository productRepository;
    private final CategoryRepository categoryRepository;

    public Page<ProductDTO> getAllProducts(Pageable pageable) {
        return productRepository.findByActiveTrue(pageable)
            .map(this::toDTO);
    }

    public Page<ProductDTO> searchProducts(String keyword, Pageable pageable) {
        return productRepository.searchProducts(keyword, pageable)
            .map(this::toDTO);
    }

    public ProductDTO getProductById(Long id) {
        Product product = productRepository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("Product not found: " + id));
        return toDTO(product);
    }

    public ProductDTO createProduct(CreateProductRequest request) {
        Category category = categoryRepository.findById(request.getCategoryId())
            .orElseThrow(() -> new ResourceNotFoundException("Category not found"));

        Product product = new Product();
        product.setName(request.getName());
        product.setDescription(request.getDescription());
        product.setPrice(request.getPrice());
        product.setStockQuantity(request.getStockQuantity());
        product.setSku(request.getSku());
        product.setImageUrl(request.getImageUrl());
        product.setCategory(category);

        return toDTO(productRepository.save(product));
    }

    public ProductDTO updateProduct(Long id, UpdateProductRequest request) {
        Product product = productRepository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("Product not found: " + id));

        if (request.getName() != null) product.setName(request.getName());
        if (request.getPrice() != null) product.setPrice(request.getPrice());
        if (request.getStockQuantity() != null) product.setStockQuantity(request.getStockQuantity());

        return toDTO(productRepository.save(product));
    }

    @Transactional
    public void deductStock(Long productId, int quantity) {
        int updated = productRepository.deductStock(productId, quantity);
        if (updated == 0) {
            throw new InsufficientStockException("Insufficient stock for product: " + productId);
        }
    }

    private ProductDTO toDTO(Product product) {
        return ProductDTO.builder()
            .id(product.getId())
            .name(product.getName())
            .description(product.getDescription())
            .price(product.getPrice())
            .stockQuantity(product.getStockQuantity())
            .sku(product.getSku())
            .imageUrl(product.getImageUrl())
            .categoryName(product.getCategory() != null ? product.getCategory().getName() : null)
            .build();
    }
}

// CartService.java
@Service
@Transactional
@RequiredArgsConstructor
public class CartService {

    private final CartRepository cartRepository;
    private final ProductRepository productRepository;

    public CartDTO getCart(Long userId) {
        Cart cart = cartRepository.findByUserId(userId)
            .orElseGet(() -> createNewCart(userId));
        return toDTO(cart);
    }

    public CartDTO addItem(Long userId, Long productId, int quantity) {
        Cart cart = cartRepository.findByUserId(userId)
            .orElseGet(() -> createNewCart(userId));

        Product product = productRepository.findById(productId)
            .orElseThrow(() -> new ResourceNotFoundException("Product not found: " + productId));

        if (product.getStockQuantity() < quantity) {
            throw new InsufficientStockException("Not enough stock available");
        }

        CartItem existingItem = cart.getItems().stream()
            .filter(item -> item.getProduct().getId().equals(productId))
            .findFirst()
            .orElse(null);

        if (existingItem != null) {
            existingItem.setQuantity(existingItem.getQuantity() + quantity);
        } else {
            CartItem newItem = new CartItem();
            newItem.setCart(cart);
            newItem.setProduct(product);
            newItem.setQuantity(quantity);
            cart.getItems().add(newItem);
        }

        return toDTO(cartRepository.save(cart));
    }

    public CartDTO updateItem(Long userId, Long productId, int quantity) {
        Cart cart = cartRepository.findByUserId(userId)
            .orElseThrow(() -> new ResourceNotFoundException("Cart not found"));

        CartItem item = cart.getItems().stream()
            .filter(i -> i.getProduct().getId().equals(productId))
            .findFirst()
            .orElseThrow(() -> new ResourceNotFoundException("Item not in cart"));

        if (quantity <= 0) {
            cart.getItems().remove(item);
        } else {
            item.setQuantity(quantity);
        }

        return toDTO(cartRepository.save(cart));
    }

    public void clearCart(Long userId) {
        Cart cart = cartRepository.findByUserId(userId)
            .orElseThrow(() -> new ResourceNotFoundException("Cart not found"));
        cart.getItems().clear();
        cartRepository.save(cart);
    }

    private Cart createNewCart(Long userId) {
        Cart cart = new Cart();
        // Note: In real app, load user from UserRepository
        cartRepository.save(cart);
        return cart;
    }

    private CartDTO toDTO(Cart cart) {
        return CartDTO.builder()
            .id(cart.getId())
            .items(cart.getItems().stream().map(this::toItemDTO).collect(Collectors.toList()))
            .totalAmount(cart.getTotalAmount())
            .itemCount(cart.getItems().size())
            .build();
    }

    private CartItemDTO toItemDTO(CartItem item) {
        return CartItemDTO.builder()
            .productId(item.getProduct().getId())
            .productName(item.getProduct().getName())
            .price(item.getProduct().getPrice())
            .quantity(item.getQuantity())
            .subtotal(item.getProduct().getPrice().multiply(BigDecimal.valueOf(item.getQuantity())))
            .build();
    }
}

// OrderService.java
@Service
@Transactional
@RequiredArgsConstructor
public class OrderService {

    private final OrderRepository orderRepository;
    private final CartService cartService;
    private final ProductService productService;
    private final PaymentService paymentService;

    public OrderDTO createOrder(Long userId, CreateOrderRequest request) {
        CartDTO cart = cartService.getCart(userId);

        if (cart.getItems().isEmpty()) {
            throw new BadRequestException("Cart is empty");
        }

        // Create order
        Order order = new Order();
        order.setOrderNumber(generateOrderNumber());
        order.setStatus(OrderStatus.PENDING);
        order.setShippingAddress(mapAddress(request.getShippingAddress()));

        // Add items and deduct stock
        BigDecimal total = BigDecimal.ZERO;
        for (CartItemDTO cartItem : cart.getItems()) {
            productService.deductStock(cartItem.getProductId(), cartItem.getQuantity());

            OrderItem orderItem = new OrderItem();
            orderItem.setOrder(order);
            orderItem.setQuantity(cartItem.getQuantity());
            orderItem.setUnitPrice(cartItem.getPrice());
            orderItem.setSubtotal(cartItem.getSubtotal());
            order.getItems().add(orderItem);

            total = total.add(cartItem.getSubtotal());
        }

        order.setTotalAmount(total);
        order.setShippingCost(calculateShipping(total));

        Order savedOrder = orderRepository.save(order);

        // Clear cart after order creation
        cartService.clearCart(userId);

        return toDTO(savedOrder);
    }

    public OrderDTO updateStatus(Long orderId, OrderStatus status) {
        Order order = orderRepository.findById(orderId)
            .orElseThrow(() -> new ResourceNotFoundException("Order not found: " + orderId));

        validateStatusTransition(order.getStatus(), status);
        order.setStatus(status);

        return toDTO(orderRepository.save(order));
    }

    public Page<OrderDTO> getUserOrders(Long userId, Pageable pageable) {
        return orderRepository.findByUserIdOrderByCreatedAtDesc(userId, pageable)
            .map(this::toDTO);
    }

    private String generateOrderNumber() {
        return "ORD-" + System.currentTimeMillis() + "-" + 
               (int)(Math.random() * 1000);
    }

    private BigDecimal calculateShipping(BigDecimal orderTotal) {
        return orderTotal.compareTo(new BigDecimal("1000")) >= 0 ?
               BigDecimal.ZERO : new BigDecimal("50");
    }

    private void validateStatusTransition(OrderStatus current, OrderStatus next) {
        Map<OrderStatus, Set<OrderStatus>> allowed = Map.of(
            OrderStatus.PENDING, Set.of(OrderStatus.CONFIRMED, OrderStatus.CANCELLED),
            OrderStatus.CONFIRMED, Set.of(OrderStatus.PROCESSING, OrderStatus.CANCELLED),
            OrderStatus.PROCESSING, Set.of(OrderStatus.SHIPPED),
            OrderStatus.SHIPPED, Set.of(OrderStatus.DELIVERED)
        );

        if (!allowed.getOrDefault(current, Set.of()).contains(next)) {
            throw new BadRequestException("Invalid status transition: " + current + " -> " + next);
        }
    }

    private OrderDTO toDTO(Order order) {
        return OrderDTO.builder()
            .id(order.getId())
            .orderNumber(order.getOrderNumber())
            .status(order.getStatus())
            .totalAmount(order.getTotalAmount())
            .shippingCost(order.getShippingCost())
            .createdAt(order.getCreatedAt())
            .items(order.getItems().stream().map(this::toItemDTO).collect(Collectors.toList()))
            .build();
    }

    private OrderItemDTO toItemDTO(OrderItem item) {
        return OrderItemDTO.builder()
            .productId(item.getProduct().getId())
            .productName(item.getProduct().getName())
            .quantity(item.getQuantity())
            .unitPrice(item.getUnitPrice())
            .subtotal(item.getSubtotal())
            .build();
    }

    private ShippingAddress mapAddress(ShippingAddressRequest req) {
        ShippingAddress address = new ShippingAddress();
        address.setRecipientName(req.getRecipientName());
        address.setAddressLine1(req.getAddressLine1());
        address.setCity(req.getCity());
        address.setProvince(req.getProvince());
        address.setPostalCode(req.getPostalCode());
        address.setPhoneNumber(req.getPhoneNumber());
        return address;
    }
}
```

### REST Controller

```java
// ProductController.java
@RestController
@RequestMapping("/api/v1/products")
@RequiredArgsConstructor
@Tag(name = "Products", description = "Product management APIs")
public class ProductController {

    private final ProductService productService;

    @GetMapping
    public ResponseEntity<Page<ProductDTO>> getAllProducts(
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size,
            @RequestParam(defaultValue = "id") String sortBy) {
        Pageable pageable = PageRequest.of(page, size, Sort.by(sortBy));
        return ResponseEntity.ok(productService.getAllProducts(pageable));
    }

    @GetMapping("/search")
    public ResponseEntity<Page<ProductDTO>> searchProducts(
            @RequestParam String keyword,
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size) {
        Pageable pageable = PageRequest.of(page, size);
        return ResponseEntity.ok(productService.searchProducts(keyword, pageable));
    }

    @GetMapping("/{id}")
    public ResponseEntity<ProductDTO> getProduct(@PathVariable Long id) {
        return ResponseEntity.ok(productService.getProductById(id));
    }

    @PostMapping
    @PreAuthorize("hasRole('ADMIN')")
    public ResponseEntity<ProductDTO> createProduct(
            @Valid @RequestBody CreateProductRequest request) {
        ProductDTO product = productService.createProduct(request);
        return ResponseEntity.status(HttpStatus.CREATED).body(product);
    }

    @PutMapping("/{id}")
    @PreAuthorize("hasRole('ADMIN')")
    public ResponseEntity<ProductDTO> updateProduct(
            @PathVariable Long id,
            @Valid @RequestBody UpdateProductRequest request) {
        return ResponseEntity.ok(productService.updateProduct(id, request));
    }

    @DeleteMapping("/{id}")
    @PreAuthorize("hasRole('ADMIN')")
    public ResponseEntity<Void> deleteProduct(@PathVariable Long id) {
        productService.deleteProduct(id);
        return ResponseEntity.noContent().build();
    }
}

// CartController.java
@RestController
@RequestMapping("/api/v1/cart")
@RequiredArgsConstructor
public class CartController {

    private final CartService cartService;

    @GetMapping
    public ResponseEntity<CartDTO> getCart(@AuthenticationPrincipal UserPrincipal user) {
        return ResponseEntity.ok(cartService.getCart(user.getId()));
    }

    @PostMapping("/items")
    public ResponseEntity<CartDTO> addItem(
            @AuthenticationPrincipal UserPrincipal user,
            @Valid @RequestBody AddToCartRequest request) {
        return ResponseEntity.ok(cartService.addItem(user.getId(), request.getProductId(), request.getQuantity()));
    }

    @PutMapping("/items/{productId}")
    public ResponseEntity<CartDTO> updateItem(
            @AuthenticationPrincipal UserPrincipal user,
            @PathVariable Long productId,
            @RequestParam int quantity) {
        return ResponseEntity.ok(cartService.updateItem(user.getId(), productId, quantity));
    }

    @DeleteMapping("/items/{productId}")
    public ResponseEntity<CartDTO> removeItem(
            @AuthenticationPrincipal UserPrincipal user,
            @PathVariable Long productId) {
        return ResponseEntity.ok(cartService.updateItem(user.getId(), productId, 0));
    }

    @DeleteMapping
    public ResponseEntity<Void> clearCart(@AuthenticationPrincipal UserPrincipal user) {
        cartService.clearCart(user.getId());
        return ResponseEntity.noContent().build();
    }
}

// OrderController.java
@RestController
@RequestMapping("/api/v1/orders")
@RequiredArgsConstructor
public class OrderController {

    private final OrderService orderService;

    @PostMapping
    public ResponseEntity<OrderDTO> createOrder(
            @AuthenticationPrincipal UserPrincipal user,
            @Valid @RequestBody CreateOrderRequest request) {
        OrderDTO order = orderService.createOrder(user.getId(), request);
        return ResponseEntity.status(HttpStatus.CREATED).body(order);
    }

    @GetMapping
    public ResponseEntity<Page<OrderDTO>> getUserOrders(
            @AuthenticationPrincipal UserPrincipal user,
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "10") int size) {
        Pageable pageable = PageRequest.of(page, size);
        return ResponseEntity.ok(orderService.getUserOrders(user.getId(), pageable));
    }

    @GetMapping("/{id}")
    public ResponseEntity<OrderDTO> getOrder(
            @AuthenticationPrincipal UserPrincipal user,
            @PathVariable Long id) {
        return ResponseEntity.ok(orderService.getOrderById(id, user.getId()));
    }

    @PatchMapping("/{id}/status")
    @PreAuthorize("hasRole('ADMIN')")
    public ResponseEntity<OrderDTO> updateStatus(
            @PathVariable Long id,
            @RequestParam OrderStatus status) {
        return ResponseEntity.ok(orderService.updateStatus(id, status));
    }
}
```

### Flyway Migration

```sql
-- V1__init_ecommerce.sql
CREATE TABLE categories (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL UNIQUE,
    description TEXT,
    parent_id BIGINT REFERENCES categories(id),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE products (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    price DECIMAL(10,2) NOT NULL,
    stock_quantity INT NOT NULL DEFAULT 0,
    sku VARCHAR(100) UNIQUE,
    image_url VARCHAR(500),
    category_id BIGINT REFERENCES categories(id),
    active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE carts (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT UNIQUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE cart_items (
    id BIGSERIAL PRIMARY KEY,
    cart_id BIGINT NOT NULL REFERENCES carts(id) ON DELETE CASCADE,
    product_id BIGINT NOT NULL REFERENCES products(id),
    quantity INT NOT NULL CHECK (quantity > 0)
);

CREATE TABLE orders (
    id BIGSERIAL PRIMARY KEY,
    order_number VARCHAR(50) NOT NULL UNIQUE,
    user_id BIGINT NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'PENDING',
    total_amount DECIMAL(10,2),
    shipping_cost DECIMAL(10,2) DEFAULT 0,
    recipient_name VARCHAR(255),
    address_line1 VARCHAR(255),
    address_line2 VARCHAR(255),
    city VARCHAR(100),
    province VARCHAR(100),
    postal_code VARCHAR(20),
    phone_number VARCHAR(20),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE order_items (
    id BIGSERIAL PRIMARY KEY,
    order_id BIGINT NOT NULL REFERENCES orders(id),
    product_id BIGINT NOT NULL REFERENCES products(id),
    quantity INT NOT NULL,
    unit_price DECIMAL(10,2) NOT NULL,
    subtotal DECIMAL(10,2) NOT NULL
);

CREATE TABLE payments (
    id BIGSERIAL PRIMARY KEY,
    order_id BIGINT NOT NULL REFERENCES orders(id),
    stripe_payment_intent_id VARCHAR(255),
    stripe_charge_id VARCHAR(255),
    status VARCHAR(20) NOT NULL DEFAULT 'PENDING',
    amount DECIMAL(10,2) NOT NULL,
    currency VARCHAR(10) DEFAULT 'THB',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_products_category ON products(category_id);
CREATE INDEX idx_products_sku ON products(sku);
CREATE INDEX idx_orders_user ON orders(user_id);
CREATE INDEX idx_orders_status ON orders(status);
```

### Docker Compose

```yaml
# docker-compose.yml (ecommerce snippet)
services:
  db:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: ecommerce_db
      POSTGRES_USER: ecommerce_user
      POSTGRES_PASSWORD: ecommerce_pass
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

  app:
    build: .
    ports:
      - "8080:8080"
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://db:5432/ecommerce_db
      SPRING_DATASOURCE_USERNAME: ecommerce_user
      SPRING_DATASOURCE_PASSWORD: ecommerce_pass
      STRIPE_SECRET_KEY: sk_test_your_stripe_key
    depends_on:
      - db
      - redis
```

### ตัวอย่าง API Calls

```bash
# สร้างสินค้าใหม่
curl -X POST http://localhost:8080/api/v1/products \
  -H "Authorization: Bearer {token}" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "iPhone 15 Pro",
    "description": "Apple iPhone 15 Pro 256GB",
    "price": 45900.00,
    "stockQuantity": 50,
    "sku": "IPHONE-15-PRO-256",
    "categoryId": 1
  }'

# เพิ่มสินค้าลงตะกร้า
curl -X POST http://localhost:8080/api/v1/cart/items \
  -H "Authorization: Bearer {token}" \
  -H "Content-Type: application/json" \
  -d '{"productId": 1, "quantity": 2}'

# สร้างออเดอร์
curl -X POST http://localhost:8080/api/v1/orders \
  -H "Authorization: Bearer {token}" \
  -H "Content-Type: application/json" \
  -d '{
    "shippingAddress": {
      "recipientName": "สมชาย ใจดี",
      "addressLine1": "123 ถนนสุขุมวิท",
      "city": "กรุงเทพมหานคร",
      "province": "กรุงเทพมหานคร",
      "postalCode": "10110",
      "phoneNumber": "0812345678"
    }
  }'

# ดูสถานะออเดอร์
curl http://localhost:8080/api/v1/orders \
  -H "Authorization: Bearer {token}"
```

---

## โปรเจคที่ 2: Digital Wallet Service

### ภาพรวมระบบ

ระบบกระเป๋าเงินดิจิทัลที่รองรับการจัดการยอดเงิน, การโอนเงินระหว่างผู้ใช้, ประวัติการทำธุรกรรม, การเติมเงิน และการถอนเงิน ระบบใช้ Idempotency Keys เพื่อป้องกันการทำธุรกรรมซ้ำ และใช้ Database Transaction เพื่อความถูกต้องของข้อมูล

### Entity Classes

```java
// Wallet.java
@Entity
@Table(name = "wallets")
@Data
@NoArgsConstructor
public class Wallet {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @OneToOne
    @JoinColumn(name = "user_id", unique = true)
    private User user;

    @Column(nullable = false, precision = 15, scale = 2)
    private BigDecimal balance = BigDecimal.ZERO;

    @Column(nullable = false, unique = true)
    private String walletNumber;

    @Enumerated(EnumType.STRING)
    private WalletStatus status = WalletStatus.ACTIVE;

    @Version
    private Long version; // Optimistic locking

    @CreatedDate
    private LocalDateTime createdAt;

    @LastModifiedDate
    private LocalDateTime updatedAt;
}

// WalletTransaction.java
@Entity
@Table(name = "wallet_transactions")
@Data
@NoArgsConstructor
public class WalletTransaction {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "wallet_id")
    private Wallet wallet;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "related_wallet_id")
    private Wallet relatedWallet; // For transfers

    @Enumerated(EnumType.STRING)
    private TransactionType type;

    @Column(nullable = false, precision = 15, scale = 2)
    private BigDecimal amount;

    @Column(precision = 15, scale = 2)
    private BigDecimal balanceBefore;

    @Column(precision = 15, scale = 2)
    private BigDecimal balanceAfter;

    @Enumerated(EnumType.STRING)
    private TransactionStatus status = TransactionStatus.COMPLETED;

    private String description;

    @Column(unique = true)
    private String idempotencyKey;

    private String referenceId;

    @CreatedDate
    private LocalDateTime createdAt;
}

// Enums
public enum TransactionType {
    TOP_UP, WITHDRAWAL, TRANSFER_IN, TRANSFER_OUT, PAYMENT, REFUND
}

public enum TransactionStatus {
    PENDING, COMPLETED, FAILED, REVERSED
}

public enum WalletStatus {
    ACTIVE, SUSPENDED, CLOSED
}
```

### Repository

```java
// WalletRepository.java
@Repository
public interface WalletRepository extends JpaRepository<Wallet, Long> {
    Optional<Wallet> findByUserId(Long userId);
    Optional<Wallet> findByWalletNumber(String walletNumber);

    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("SELECT w FROM Wallet w WHERE w.id = :id")
    Optional<Wallet> findByIdWithLock(@Param("id") Long id);
}

// WalletTransactionRepository.java
@Repository
public interface WalletTransactionRepository extends JpaRepository<WalletTransaction, Long> {
    Page<WalletTransaction> findByWalletIdOrderByCreatedAtDesc(Long walletId, Pageable pageable);

    Optional<WalletTransaction> findByIdempotencyKey(String idempotencyKey);

    @Query("SELECT SUM(t.amount) FROM WalletTransaction t " +
           "WHERE t.wallet.id = :walletId AND t.type = :type " +
           "AND t.status = 'COMPLETED' AND t.createdAt >= :since")
    BigDecimal sumByTypeAndPeriod(@Param("walletId") Long walletId,
                                  @Param("type") TransactionType type,
                                  @Param("since") LocalDateTime since);
}
```

### Service Layer

```java
// WalletService.java
@Service
@RequiredArgsConstructor
@Slf4j
public class WalletService {

    private final WalletRepository walletRepository;
    private final WalletTransactionRepository transactionRepository;

    @Transactional
    public WalletDTO topUp(Long userId, BigDecimal amount, String idempotencyKey) {
        // Check idempotency
        Optional<WalletTransaction> existing = transactionRepository.findByIdempotencyKey(idempotencyKey);
        if (existing.isPresent()) {
            log.info("Duplicate top-up request: {}", idempotencyKey);
            Wallet wallet = walletRepository.findByUserId(userId).orElseThrow();
            return toDTO(wallet);
        }

        Wallet wallet = walletRepository.findByIdWithLock(
            walletRepository.findByUserId(userId)
                .orElseThrow(() -> new ResourceNotFoundException("Wallet not found"))
                .getId()
        ).orElseThrow();

        validateWallet(wallet);
        validateAmount(amount);

        BigDecimal before = wallet.getBalance();
        wallet.setBalance(before.add(amount));
        walletRepository.save(wallet);

        createTransaction(wallet, null, TransactionType.TOP_UP, amount, before, wallet.getBalance(), 
                         "Top up", idempotencyKey);

        return toDTO(wallet);
    }

    @Transactional
    public TransferResult transfer(Long fromUserId, String toWalletNumber, 
                                   BigDecimal amount, String idempotencyKey) {
        // Check idempotency
        if (transactionRepository.findByIdempotencyKey(idempotencyKey).isPresent()) {
            throw new DuplicateTransactionException("Transaction already processed");
        }

        Wallet fromWallet = walletRepository.findByUserId(fromUserId)
            .orElseThrow(() -> new ResourceNotFoundException("Source wallet not found"));

        Wallet toWallet = walletRepository.findByWalletNumber(toWalletNumber)
            .orElseThrow(() -> new ResourceNotFoundException("Destination wallet not found"));

        if (fromWallet.getId().equals(toWallet.getId())) {
            throw new BadRequestException("Cannot transfer to same wallet");
        }

        // Lock both wallets in consistent order to prevent deadlocks
        Long firstId = Math.min(fromWallet.getId(), toWallet.getId());
        Long secondId = Math.max(fromWallet.getId(), toWallet.getId());

        Wallet lockedFirst = walletRepository.findByIdWithLock(firstId).orElseThrow();
        Wallet lockedSecond = walletRepository.findByIdWithLock(secondId).orElseThrow();

        Wallet lockedFrom = lockedFirst.getId().equals(fromWallet.getId()) ? lockedFirst : lockedSecond;
        Wallet lockedTo = lockedFirst.getId().equals(toWallet.getId()) ? lockedFirst : lockedSecond;

        validateWallet(lockedFrom);
        validateAmount(amount);

        if (lockedFrom.getBalance().compareTo(amount) < 0) {
            throw new InsufficientBalanceException("Insufficient balance");
        }

        BigDecimal fromBefore = lockedFrom.getBalance();
        BigDecimal toBefore = lockedTo.getBalance();

        lockedFrom.setBalance(fromBefore.subtract(amount));
        lockedTo.setBalance(toBefore.add(amount));

        walletRepository.save(lockedFrom);
        walletRepository.save(lockedTo);

        createTransaction(lockedFrom, lockedTo, TransactionType.TRANSFER_OUT, amount,
                         fromBefore, lockedFrom.getBalance(), "Transfer out", idempotencyKey);
        createTransaction(lockedTo, lockedFrom, TransactionType.TRANSFER_IN, amount,
                         toBefore, lockedTo.getBalance(), "Transfer in", idempotencyKey + "_IN");

        return new TransferResult(toDTO(lockedFrom), amount, toWalletNumber);
    }

    @Transactional
    public WalletDTO withdraw(Long userId, BigDecimal amount, String idempotencyKey) {
        Wallet wallet = walletRepository.findByIdWithLock(
            walletRepository.findByUserId(userId)
                .orElseThrow(() -> new ResourceNotFoundException("Wallet not found"))
                .getId()
        ).orElseThrow();

        validateWallet(wallet);
        validateAmount(amount);

        if (wallet.getBalance().compareTo(amount) < 0) {
            throw new InsufficientBalanceException("Insufficient balance for withdrawal");
        }

        BigDecimal before = wallet.getBalance();
        wallet.setBalance(before.subtract(amount));
        walletRepository.save(wallet);

        createTransaction(wallet, null, TransactionType.WITHDRAWAL, amount, before, 
                         wallet.getBalance(), "Withdrawal", idempotencyKey);

        return toDTO(wallet);
    }

    @Transactional(readOnly = true)
    public Page<TransactionDTO> getTransactionHistory(Long userId, Pageable pageable) {
        Wallet wallet = walletRepository.findByUserId(userId)
            .orElseThrow(() -> new ResourceNotFoundException("Wallet not found"));

        return transactionRepository.findByWalletIdOrderByCreatedAtDesc(wallet.getId(), pageable)
            .map(this::toTransactionDTO);
    }

    private void createTransaction(Wallet wallet, Wallet relatedWallet, TransactionType type,
                                   BigDecimal amount, BigDecimal before, BigDecimal after,
                                   String description, String idempotencyKey) {
        WalletTransaction tx = new WalletTransaction();
        tx.setWallet(wallet);
        tx.setRelatedWallet(relatedWallet);
        tx.setType(type);
        tx.setAmount(amount);
        tx.setBalanceBefore(before);
        tx.setBalanceAfter(after);
        tx.setDescription(description);
        tx.setIdempotencyKey(idempotencyKey);
        transactionRepository.save(tx);
    }

    private void validateWallet(Wallet wallet) {
        if (wallet.getStatus() != WalletStatus.ACTIVE) {
            throw new WalletException("Wallet is not active: " + wallet.getStatus());
        }
    }

    private void validateAmount(BigDecimal amount) {
        if (amount == null || amount.compareTo(BigDecimal.ZERO) <= 0) {
            throw new BadRequestException("Amount must be positive");
        }
    }

    private WalletDTO toDTO(Wallet wallet) {
        return WalletDTO.builder()
            .id(wallet.getId())
            .walletNumber(wallet.getWalletNumber())
            .balance(wallet.getBalance())
            .status(wallet.getStatus())
            .build();
    }

    private TransactionDTO toTransactionDTO(WalletTransaction tx) {
        return TransactionDTO.builder()
            .id(tx.getId())
            .type(tx.getType())
            .amount(tx.getAmount())
            .balanceBefore(tx.getBalanceBefore())
            .balanceAfter(tx.getBalanceAfter())
            .description(tx.getDescription())
            .createdAt(tx.getCreatedAt())
            .build();
    }
}
```

### REST Controller

```java
// WalletController.java
@RestController
@RequestMapping("/api/v1/wallet")
@RequiredArgsConstructor
public class WalletController {

    private final WalletService walletService;

    @GetMapping
    public ResponseEntity<WalletDTO> getWallet(@AuthenticationPrincipal UserPrincipal user) {
        return ResponseEntity.ok(walletService.getWallet(user.getId()));
    }

    @PostMapping("/top-up")
    public ResponseEntity<WalletDTO> topUp(
            @AuthenticationPrincipal UserPrincipal user,
            @Valid @RequestBody TopUpRequest request,
            @RequestHeader("Idempotency-Key") String idempotencyKey) {
        return ResponseEntity.ok(walletService.topUp(user.getId(), request.getAmount(), idempotencyKey));
    }

    @PostMapping("/transfer")
    public ResponseEntity<TransferResult> transfer(
            @AuthenticationPrincipal UserPrincipal user,
            @Valid @RequestBody TransferRequest request,
            @RequestHeader("Idempotency-Key") String idempotencyKey) {
        return ResponseEntity.ok(
            walletService.transfer(user.getId(), request.getToWalletNumber(), 
                                   request.getAmount(), idempotencyKey));
    }

    @PostMapping("/withdraw")
    public ResponseEntity<WalletDTO> withdraw(
            @AuthenticationPrincipal UserPrincipal user,
            @Valid @RequestBody WithdrawRequest request,
            @RequestHeader("Idempotency-Key") String idempotencyKey) {
        return ResponseEntity.ok(walletService.withdraw(user.getId(), request.getAmount(), idempotencyKey));
    }

    @GetMapping("/transactions")
    public ResponseEntity<Page<TransactionDTO>> getTransactions(
            @AuthenticationPrincipal UserPrincipal user,
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size) {
        Pageable pageable = PageRequest.of(page, size);
        return ResponseEntity.ok(walletService.getTransactionHistory(user.getId(), pageable));
    }
}
```

### Flyway Migration

```sql
-- V2__init_wallet.sql
CREATE TABLE wallets (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL UNIQUE,
    balance DECIMAL(15,2) NOT NULL DEFAULT 0.00,
    wallet_number VARCHAR(20) NOT NULL UNIQUE,
    status VARCHAR(20) NOT NULL DEFAULT 'ACTIVE',
    version BIGINT DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE wallet_transactions (
    id BIGSERIAL PRIMARY KEY,
    wallet_id BIGINT NOT NULL REFERENCES wallets(id),
    related_wallet_id BIGINT REFERENCES wallets(id),
    type VARCHAR(30) NOT NULL,
    amount DECIMAL(15,2) NOT NULL,
    balance_before DECIMAL(15,2),
    balance_after DECIMAL(15,2),
    status VARCHAR(20) NOT NULL DEFAULT 'COMPLETED',
    description VARCHAR(255),
    idempotency_key VARCHAR(255) UNIQUE,
    reference_id VARCHAR(255),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_wallet_user ON wallets(user_id);
CREATE INDEX idx_tx_wallet ON wallet_transactions(wallet_id);
CREATE INDEX idx_tx_idempotency ON wallet_transactions(idempotency_key);
```

### ตัวอย่าง API Calls

```bash
# เติมเงิน
curl -X POST http://localhost:8080/api/v1/wallet/top-up \
  -H "Authorization: Bearer {token}" \
  -H "Idempotency-Key: unique-key-12345" \
  -H "Content-Type: application/json" \
  -d '{"amount": 1000.00}'

# โอนเงิน
curl -X POST http://localhost:8080/api/v1/wallet/transfer \
  -H "Authorization: Bearer {token}" \
  -H "Idempotency-Key: transfer-key-67890" \
  -H "Content-Type: application/json" \
  -d '{"toWalletNumber": "WALL-001234", "amount": 500.00}'

# ดูประวัติ
curl http://localhost:8080/api/v1/wallet/transactions \
  -H "Authorization: Bearer {token}"
```

---

## โปรเจคที่ 3: Invoice Management System

### ภาพรวมระบบ

ระบบจัดการใบแจ้งหนี้สำหรับธุรกิจ รองรับการสร้าง แก้ไข และส่งใบแจ้งหนี้ มีการคำนวณภาษีอัตโนมัติ สร้าง PDF และติดตามสถานะการชำระเงิน พร้อมระบบแจ้งเตือนเมื่อครบกำหนดชำระ

### Entity Classes

```java
// Invoice.java
@Entity
@Table(name = "invoices")
@Data
@NoArgsConstructor
public class Invoice {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(unique = true, nullable = false)
    private String invoiceNumber;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "customer_id")
    private Customer customer;

    @Enumerated(EnumType.STRING)
    private InvoiceStatus status = InvoiceStatus.DRAFT;

    @OneToMany(mappedBy = "invoice", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<InvoiceLineItem> lineItems = new ArrayList<>();

    @Column(precision = 10, scale = 2)
    private BigDecimal subtotal;

    @Column(precision = 5, scale = 2)
    private BigDecimal taxRate = new BigDecimal("7.00"); // VAT 7%

    @Column(precision = 10, scale = 2)
    private BigDecimal taxAmount;

    @Column(precision = 10, scale = 2)
    private BigDecimal totalAmount;

    @Column(precision = 10, scale = 2)
    private BigDecimal paidAmount = BigDecimal.ZERO;

    private LocalDate issueDate;
    private LocalDate dueDate;
    private LocalDate paidDate;

    private String notes;
    private String pdfUrl;

    @CreatedDate
    private LocalDateTime createdAt;

    @LastModifiedDate
    private LocalDateTime updatedAt;
}

// InvoiceLineItem.java
@Entity
@Table(name = "invoice_line_items")
@Data
@NoArgsConstructor
public class InvoiceLineItem {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "invoice_id")
    private Invoice invoice;

    private String description;
    private Integer quantity;

    @Column(precision = 10, scale = 2)
    private BigDecimal unitPrice;

    @Column(precision = 10, scale = 2)
    private BigDecimal amount;
}

// Customer.java
@Entity
@Table(name = "customers")
@Data
@NoArgsConstructor
public class Customer {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;
    private String email;
    private String phone;
    private String taxId;
    private String address;
    private String city;
    private String country = "Thailand";
}

public enum InvoiceStatus {
    DRAFT, SENT, VIEWED, PARTIAL_PAID, PAID, OVERDUE, CANCELLED
}
```

### Service Layer

```java
// InvoiceService.java
@Service
@Transactional
@RequiredArgsConstructor
@Slf4j
public class InvoiceService {

    private final InvoiceRepository invoiceRepository;
    private final CustomerRepository customerRepository;
    private final PdfGeneratorService pdfGeneratorService;
    private final EmailService emailService;

    public InvoiceDTO createInvoice(CreateInvoiceRequest request) {
        Customer customer = customerRepository.findById(request.getCustomerId())
            .orElseThrow(() -> new ResourceNotFoundException("Customer not found"));

        Invoice invoice = new Invoice();
        invoice.setInvoiceNumber(generateInvoiceNumber());
        invoice.setCustomer(customer);
        invoice.setIssueDate(request.getIssueDate());
        invoice.setDueDate(request.getDueDate());
        invoice.setNotes(request.getNotes());
        invoice.setTaxRate(request.getTaxRate() != null ? request.getTaxRate() : new BigDecimal("7.00"));

        // Add line items
        for (LineItemRequest itemReq : request.getLineItems()) {
            InvoiceLineItem item = new InvoiceLineItem();
            item.setInvoice(invoice);
            item.setDescription(itemReq.getDescription());
            item.setQuantity(itemReq.getQuantity());
            item.setUnitPrice(itemReq.getUnitPrice());
            item.setAmount(itemReq.getUnitPrice().multiply(BigDecimal.valueOf(itemReq.getQuantity())));
            invoice.getLineItems().add(item);
        }

        calculateTotals(invoice);

        return toDTO(invoiceRepository.save(invoice));
    }

    public InvoiceDTO sendInvoice(Long id) {
        Invoice invoice = invoiceRepository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("Invoice not found"));

        if (invoice.getStatus() != InvoiceStatus.DRAFT) {
            throw new BadRequestException("Only draft invoices can be sent");
        }

        // Generate PDF
        String pdfUrl = pdfGeneratorService.generateInvoicePdf(invoice);
        invoice.setPdfUrl(pdfUrl);
        invoice.setStatus(InvoiceStatus.SENT);

        // Send email
        emailService.sendInvoice(invoice.getCustomer().getEmail(), invoice);

        return toDTO(invoiceRepository.save(invoice));
    }

    public InvoiceDTO recordPayment(Long id, BigDecimal amount, LocalDate paymentDate) {
        Invoice invoice = invoiceRepository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("Invoice not found"));

        if (invoice.getStatus() == InvoiceStatus.PAID || 
            invoice.getStatus() == InvoiceStatus.CANCELLED) {
            throw new BadRequestException("Cannot record payment for this invoice");
        }

        BigDecimal newPaid = invoice.getPaidAmount().add(amount);

        if (newPaid.compareTo(invoice.getTotalAmount()) > 0) {
            throw new BadRequestException("Payment exceeds invoice total");
        }

        invoice.setPaidAmount(newPaid);

        if (newPaid.compareTo(invoice.getTotalAmount()) == 0) {
            invoice.setStatus(InvoiceStatus.PAID);
            invoice.setPaidDate(paymentDate);
        } else {
            invoice.setStatus(InvoiceStatus.PARTIAL_PAID);
        }

        return toDTO(invoiceRepository.save(invoice));
    }

    @Scheduled(cron = "0 0 9 * * *")
    public void checkOverdueInvoices() {
        List<Invoice> overdueInvoices = invoiceRepository.findOverdueInvoices(LocalDate.now());

        for (Invoice invoice : overdueInvoices) {
            invoice.setStatus(InvoiceStatus.OVERDUE);
            invoiceRepository.save(invoice);
            emailService.sendOverdueAlert(invoice.getCustomer().getEmail(), invoice);
            log.info("Marked invoice {} as overdue", invoice.getInvoiceNumber());
        }
    }

    private void calculateTotals(Invoice invoice) {
        BigDecimal subtotal = invoice.getLineItems().stream()
            .map(InvoiceLineItem::getAmount)
            .reduce(BigDecimal.ZERO, BigDecimal::add);

        BigDecimal tax = subtotal.multiply(invoice.getTaxRate())
            .divide(new BigDecimal("100"), 2, RoundingMode.HALF_UP);

        invoice.setSubtotal(subtotal);
        invoice.setTaxAmount(tax);
        invoice.setTotalAmount(subtotal.add(tax));
    }

    private String generateInvoiceNumber() {
        String year = String.valueOf(LocalDate.now().getYear());
        String month = String.format("%02d", LocalDate.now().getMonthValue());
        long count = invoiceRepository.countByCreatedAtYear(LocalDate.now().getYear()) + 1;
        return String.format("INV-%s%s-%04d", year, month, count);
    }

    private InvoiceDTO toDTO(Invoice invoice) {
        return InvoiceDTO.builder()
            .id(invoice.getId())
            .invoiceNumber(invoice.getInvoiceNumber())
            .customerName(invoice.getCustomer().getName())
            .status(invoice.getStatus())
            .subtotal(invoice.getSubtotal())
            .taxAmount(invoice.getTaxAmount())
            .totalAmount(invoice.getTotalAmount())
            .paidAmount(invoice.getPaidAmount())
            .balance(invoice.getTotalAmount().subtract(invoice.getPaidAmount()))
            .issueDate(invoice.getIssueDate())
            .dueDate(invoice.getDueDate())
            .pdfUrl(invoice.getPdfUrl())
            .lineItems(invoice.getLineItems().stream().map(this::toLineItemDTO).collect(Collectors.toList()))
            .build();
    }

    private LineItemDTO toLineItemDTO(InvoiceLineItem item) {
        return LineItemDTO.builder()
            .description(item.getDescription())
            .quantity(item.getQuantity())
            .unitPrice(item.getUnitPrice())
            .amount(item.getAmount())
            .build();
    }
}
```

### REST Controller

```java
// InvoiceController.java
@RestController
@RequestMapping("/api/v1/invoices")
@RequiredArgsConstructor
public class InvoiceController {

    private final InvoiceService invoiceService;

    @PostMapping
    public ResponseEntity<InvoiceDTO> createInvoice(@Valid @RequestBody CreateInvoiceRequest request) {
        return ResponseEntity.status(HttpStatus.CREATED).body(invoiceService.createInvoice(request));
    }

    @GetMapping("/{id}")
    public ResponseEntity<InvoiceDTO> getInvoice(@PathVariable Long id) {
        return ResponseEntity.ok(invoiceService.getInvoice(id));
    }

    @GetMapping
    public ResponseEntity<Page<InvoiceDTO>> listInvoices(
            @RequestParam(required = false) InvoiceStatus status,
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size) {
        return ResponseEntity.ok(invoiceService.listInvoices(status, PageRequest.of(page, size)));
    }

    @PutMapping("/{id}")
    public ResponseEntity<InvoiceDTO> updateInvoice(
            @PathVariable Long id,
            @Valid @RequestBody UpdateInvoiceRequest request) {
        return ResponseEntity.ok(invoiceService.updateInvoice(id, request));
    }

    @PostMapping("/{id}/send")
    public ResponseEntity<InvoiceDTO> sendInvoice(@PathVariable Long id) {
        return ResponseEntity.ok(invoiceService.sendInvoice(id));
    }

    @PostMapping("/{id}/payments")
    public ResponseEntity<InvoiceDTO> recordPayment(
            @PathVariable Long id,
            @Valid @RequestBody RecordPaymentRequest request) {
        return ResponseEntity.ok(invoiceService.recordPayment(id, request.getAmount(), request.getPaymentDate()));
    }

    @GetMapping("/{id}/pdf")
    public ResponseEntity<byte[]> downloadPdf(@PathVariable Long id) {
        byte[] pdf = invoiceService.generatePdf(id);
        return ResponseEntity.ok()
            .header(HttpHeaders.CONTENT_DISPOSITION, "attachment; filename=invoice-" + id + ".pdf")
            .contentType(MediaType.APPLICATION_PDF)
            .body(pdf);
    }
}
```

---

## โปรเจคที่ 4: Subscription Management System

### ภาพรวมระบบ

ระบบจัดการการสมัครสมาชิกสำหรับ SaaS ที่รองรับแผนบริการหลายระดับ (FREE/PRO/ENTERPRISE), วงจรชีวิตของการสมัครสมาชิก (ทดลอง/ใช้งาน/ยกเลิก/หมดอายุ), รอบการเรียกเก็บเงิน และการอัพเกรด/ดาวน์เกรดแผน

### Entity Classes

```java
// Plan.java
@Entity
@Table(name = "plans")
@Data
@NoArgsConstructor
public class Plan {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Enumerated(EnumType.STRING)
    @Column(unique = true)
    private PlanType type;

    private String name;
    private String description;

    @Column(precision = 10, scale = 2)
    private BigDecimal monthlyPrice;

    @Column(precision = 10, scale = 2)
    private BigDecimal annualPrice;

    private Integer maxUsers;
    private Integer maxProjects;
    private Long storageGb;

    @ElementCollection
    @CollectionTable(name = "plan_features")
    private List<String> features = new ArrayList<>();

    private Boolean active = true;
}

// Subscription.java
@Entity
@Table(name = "subscriptions")
@Data
@NoArgsConstructor
public class Subscription {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id")
    private User user;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "plan_id")
    private Plan plan;

    @Enumerated(EnumType.STRING)
    private SubscriptionStatus status = SubscriptionStatus.TRIAL;

    @Enumerated(EnumType.STRING)
    private BillingCycle billingCycle = BillingCycle.MONTHLY;

    private LocalDate startDate;
    private LocalDate trialEndDate;
    private LocalDate currentPeriodStart;
    private LocalDate currentPeriodEnd;
    private LocalDate cancelledAt;
    private LocalDate endDate;

    private Boolean autoRenew = true;
    private String stripeSubscriptionId;
    private String stripeCustomerId;

    @OneToMany(mappedBy = "subscription", cascade = CascadeType.ALL)
    private List<BillingRecord> billingHistory = new ArrayList<>();

    @CreatedDate
    private LocalDateTime createdAt;
}

// BillingRecord.java
@Entity
@Table(name = "billing_records")
@Data
@NoArgsConstructor
public class BillingRecord {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "subscription_id")
    private Subscription subscription;

    @Column(precision = 10, scale = 2)
    private BigDecimal amount;

    @Enumerated(EnumType.STRING)
    private BillingStatus status = BillingStatus.PENDING;

    private LocalDate billingDate;
    private LocalDate paidDate;
    private String invoiceId;
    private String failureReason;

    @CreatedDate
    private LocalDateTime createdAt;
}

// Enums
public enum PlanType { FREE, PRO, ENTERPRISE }
public enum SubscriptionStatus { TRIAL, ACTIVE, PAST_DUE, CANCELLED, EXPIRED }
public enum BillingCycle { MONTHLY, ANNUAL }
public enum BillingStatus { PENDING, PAID, FAILED, REFUNDED }
```

### Service Layer

```java
// SubscriptionService.java
@Service
@Transactional
@RequiredArgsConstructor
@Slf4j
public class SubscriptionService {

    private final SubscriptionRepository subscriptionRepository;
    private final PlanRepository planRepository;
    private final BillingRecordRepository billingRecordRepository;
    private final EmailService emailService;

    public SubscriptionDTO subscribe(Long userId, SubscribeRequest request) {
        // Check if user already has active subscription
        subscriptionRepository.findActiveByUserId(userId).ifPresent(s -> {
            throw new ConflictException("User already has an active subscription");
        });

        Plan plan = planRepository.findByType(request.getPlanType())
            .orElseThrow(() -> new ResourceNotFoundException("Plan not found"));

        Subscription subscription = new Subscription();
        subscription.setUser(loadUser(userId));
        subscription.setPlan(plan);
        subscription.setBillingCycle(request.getBillingCycle());
        subscription.setStartDate(LocalDate.now());
        subscription.setAutoRenew(true);

        // Set trial period for non-free plans
        if (plan.getType() != PlanType.FREE) {
            subscription.setStatus(SubscriptionStatus.TRIAL);
            subscription.setTrialEndDate(LocalDate.now().plusDays(14));
            subscription.setCurrentPeriodStart(LocalDate.now());
            subscription.setCurrentPeriodEnd(LocalDate.now().plusDays(14));
        } else {
            subscription.setStatus(SubscriptionStatus.ACTIVE);
            subscription.setCurrentPeriodStart(LocalDate.now());
            subscription.setCurrentPeriodEnd(LocalDate.now().plusYears(100)); // Unlimited for free
        }

        return toDTO(subscriptionRepository.save(subscription));
    }

    public SubscriptionDTO upgrade(Long userId, PlanType newPlanType) {
        Subscription current = subscriptionRepository.findActiveByUserId(userId)
            .orElseThrow(() -> new ResourceNotFoundException("No active subscription"));

        Plan newPlan = planRepository.findByType(newPlanType)
            .orElseThrow(() -> new ResourceNotFoundException("Plan not found"));

        if (getPlanRank(newPlan.getType()) <= getPlanRank(current.getPlan().getType())) {
            throw new BadRequestException("New plan must be higher tier for upgrade");
        }

        // Calculate prorated amount
        BigDecimal proratedAmount = calculateProration(current, newPlan);

        // Update subscription
        current.setPlan(newPlan);
        current.setStatus(SubscriptionStatus.ACTIVE);

        // Create billing record for upgrade
        if (proratedAmount.compareTo(BigDecimal.ZERO) > 0) {
            createBillingRecord(current, proratedAmount, LocalDate.now());
        }

        emailService.sendUpgradeConfirmation(current.getUser().getEmail(), newPlan);

        return toDTO(subscriptionRepository.save(current));
    }

    public SubscriptionDTO downgrade(Long userId, PlanType newPlanType) {
        Subscription current = subscriptionRepository.findActiveByUserId(userId)
            .orElseThrow(() -> new ResourceNotFoundException("No active subscription"));

        Plan newPlan = planRepository.findByType(newPlanType)
            .orElseThrow(() -> new ResourceNotFoundException("Plan not found"));

        if (getPlanRank(newPlan.getType()) >= getPlanRank(current.getPlan().getType())) {
            throw new BadRequestException("New plan must be lower tier for downgrade");
        }

        // Downgrade takes effect at end of current period
        current.setPlan(newPlan);
        // Note: Status stays ACTIVE, change takes effect at renewal

        return toDTO(subscriptionRepository.save(current));
    }

    public void cancel(Long userId, boolean immediately) {
        Subscription subscription = subscriptionRepository.findActiveByUserId(userId)
            .orElseThrow(() -> new ResourceNotFoundException("No active subscription"));

        if (immediately) {
            subscription.setStatus(SubscriptionStatus.CANCELLED);
            subscription.setEndDate(LocalDate.now());
        } else {
            subscription.setAutoRenew(false);
            subscription.setCancelledAt(LocalDate.now());
            // Will be cancelled at period end via scheduler
        }

        subscriptionRepository.save(subscription);
        emailService.sendCancellationConfirmation(subscription.getUser().getEmail(), subscription);
    }

    @Scheduled(cron = "0 0 1 * * *")
    public void processRenewals() {
        List<Subscription> due = subscriptionRepository.findDueForRenewal(LocalDate.now());

        for (Subscription sub : due) {
            try {
                if (!sub.getAutoRenew()) {
                    sub.setStatus(SubscriptionStatus.EXPIRED);
                    sub.setEndDate(LocalDate.now());
                } else {
                    // Process billing
                    BigDecimal amount = sub.getBillingCycle() == BillingCycle.MONTHLY ?
                        sub.getPlan().getMonthlyPrice() : sub.getPlan().getAnnualPrice();

                    createBillingRecord(sub, amount, LocalDate.now());

                    // Extend period
                    if (sub.getBillingCycle() == BillingCycle.MONTHLY) {
                        sub.setCurrentPeriodStart(LocalDate.now());
                        sub.setCurrentPeriodEnd(LocalDate.now().plusMonths(1));
                    } else {
                        sub.setCurrentPeriodStart(LocalDate.now());
                        sub.setCurrentPeriodEnd(LocalDate.now().plusYears(1));
                    }
                    sub.setStatus(SubscriptionStatus.ACTIVE);
                }
                subscriptionRepository.save(sub);
            } catch (Exception e) {
                log.error("Failed to process renewal for subscription {}: {}", sub.getId(), e.getMessage());
                sub.setStatus(SubscriptionStatus.PAST_DUE);
                subscriptionRepository.save(sub);
            }
        }
    }

    private BigDecimal calculateProration(Subscription current, Plan newPlan) {
        long totalDays = ChronoUnit.DAYS.between(current.getCurrentPeriodStart(), current.getCurrentPeriodEnd());
        long remainingDays = ChronoUnit.DAYS.between(LocalDate.now(), current.getCurrentPeriodEnd());

        BigDecimal dailyRate = newPlan.getMonthlyPrice().divide(BigDecimal.valueOf(totalDays), 2, RoundingMode.HALF_UP);
        BigDecimal currentDailyRate = current.getPlan().getMonthlyPrice().divide(BigDecimal.valueOf(totalDays), 2, RoundingMode.HALF_UP);

        return dailyRate.subtract(currentDailyRate).multiply(BigDecimal.valueOf(remainingDays));
    }

    private int getPlanRank(PlanType type) {
        return switch (type) {
            case FREE -> 0;
            case PRO -> 1;
            case ENTERPRISE -> 2;
        };
    }

    private void createBillingRecord(Subscription sub, BigDecimal amount, LocalDate date) {
        BillingRecord record = new BillingRecord();
        record.setSubscription(sub);
        record.setAmount(amount);
        record.setBillingDate(date);
        record.setStatus(BillingStatus.PENDING);
        billingRecordRepository.save(record);
    }

    private SubscriptionDTO toDTO(Subscription sub) {
        return SubscriptionDTO.builder()
            .id(sub.getId())
            .planType(sub.getPlan().getType())
            .planName(sub.getPlan().getName())
            .status(sub.getStatus())
            .billingCycle(sub.getBillingCycle())
            .currentPeriodStart(sub.getCurrentPeriodStart())
            .currentPeriodEnd(sub.getCurrentPeriodEnd())
            .autoRenew(sub.getAutoRenew())
            .build();
    }

    private User loadUser(Long userId) {
        // Load from UserRepository in real implementation
        User user = new User();
        user.setId(userId);
        return user;
    }
}
```

### REST Controller

```java
// SubscriptionController.java
@RestController
@RequestMapping("/api/v1/subscriptions")
@RequiredArgsConstructor
public class SubscriptionController {

    private final SubscriptionService subscriptionService;

    @GetMapping("/plans")
    public ResponseEntity<List<PlanDTO>> getPlans() {
        return ResponseEntity.ok(subscriptionService.getAvailablePlans());
    }

    @PostMapping
    public ResponseEntity<SubscriptionDTO> subscribe(
            @AuthenticationPrincipal UserPrincipal user,
            @Valid @RequestBody SubscribeRequest request) {
        return ResponseEntity.status(HttpStatus.CREATED)
            .body(subscriptionService.subscribe(user.getId(), request));
    }

    @GetMapping("/current")
    public ResponseEntity<SubscriptionDTO> getCurrentSubscription(
            @AuthenticationPrincipal UserPrincipal user) {
        return ResponseEntity.ok(subscriptionService.getCurrentSubscription(user.getId()));
    }

    @PostMapping("/upgrade")
    public ResponseEntity<SubscriptionDTO> upgrade(
            @AuthenticationPrincipal UserPrincipal user,
            @RequestParam PlanType planType) {
        return ResponseEntity.ok(subscriptionService.upgrade(user.getId(), planType));
    }

    @PostMapping("/downgrade")
    public ResponseEntity<SubscriptionDTO> downgrade(
            @AuthenticationPrincipal UserPrincipal user,
            @RequestParam PlanType planType) {
        return ResponseEntity.ok(subscriptionService.downgrade(user.getId(), planType));
    }

    @DeleteMapping
    public ResponseEntity<Void> cancel(
            @AuthenticationPrincipal UserPrincipal user,
            @RequestParam(defaultValue = "false") boolean immediately) {
        subscriptionService.cancel(user.getId(), immediately);
        return ResponseEntity.noContent().build();
    }

    @GetMapping("/billing")
    public ResponseEntity<Page<BillingRecordDTO>> getBillingHistory(
            @AuthenticationPrincipal UserPrincipal user,
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "10") int size) {
        return ResponseEntity.ok(subscriptionService.getBillingHistory(user.getId(), PageRequest.of(page, size)));
    }
}
```

---

## โปรเจคที่ 5: Coupon & Discount System

### ภาพรวมระบบ

ระบบจัดการคูปองและส่วนลดที่รองรับการสร้างโค้ดคูปอง ส่วนลดแบบเปอร์เซ็นต์หรือจำนวนเงิน จำกัดจำนวนการใช้งาน กำหนดระยะเวลา คูปองสำหรับหมวดหมู่สินค้า และการสร้างคูปองจำนวนมาก

### Entity Classes

```java
// Coupon.java
@Entity
@Table(name = "coupons")
@Data
@NoArgsConstructor
public class Coupon {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(unique = true, nullable = false)
    private String code;

    private String name;
    private String description;

    @Enumerated(EnumType.STRING)
    private DiscountType discountType;

    @Column(precision = 10, scale = 2)
    private BigDecimal discountValue;

    @Column(precision = 10, scale = 2)
    private BigDecimal minimumOrderAmount;

    @Column(precision = 10, scale = 2)
    private BigDecimal maximumDiscountAmount; // Cap for percentage discounts

    private Integer usageLimit; // null = unlimited
    private Integer usageCount = 0;
    private Integer usageLimitPerUser;

    private LocalDateTime validFrom;
    private LocalDateTime validTo;

    private Boolean active = true;
    private Boolean isSingleUse = false;

    @ManyToMany
    @JoinTable(name = "coupon_categories",
               joinColumns = @JoinColumn(name = "coupon_id"),
               inverseJoinColumns = @JoinColumn(name = "category_id"))
    private List<Category> applicableCategories = new ArrayList<>();

    @OneToMany(mappedBy = "coupon")
    private List<CouponUsage> usages = new ArrayList<>();

    @CreatedDate
    private LocalDateTime createdAt;
}

// CouponUsage.java
@Entity
@Table(name = "coupon_usages",
       uniqueConstraints = @UniqueConstraint(columnNames = {"coupon_id", "user_id", "order_id"}))
@Data
@NoArgsConstructor
public class CouponUsage {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne
    @JoinColumn(name = "coupon_id")
    private Coupon coupon;

    @Column(nullable = false)
    private Long userId;

    @Column(nullable = false)
    private Long orderId;

    @Column(precision = 10, scale = 2)
    private BigDecimal discountApplied;

    @CreatedDate
    private LocalDateTime usedAt;
}

public enum DiscountType {
    PERCENTAGE, FIXED_AMOUNT
}
```

### Service Layer

```java
// CouponService.java
@Service
@Transactional
@RequiredArgsConstructor
public class CouponService {

    private final CouponRepository couponRepository;
    private final CouponUsageRepository couponUsageRepository;

    public CouponDTO createCoupon(CreateCouponRequest request) {
        if (couponRepository.existsByCode(request.getCode())) {
            throw new ConflictException("Coupon code already exists: " + request.getCode());
        }

        Coupon coupon = new Coupon();
        coupon.setCode(request.getCode().toUpperCase());
        coupon.setName(request.getName());
        coupon.setDescription(request.getDescription());
        coupon.setDiscountType(request.getDiscountType());
        coupon.setDiscountValue(request.getDiscountValue());
        coupon.setMinimumOrderAmount(request.getMinimumOrderAmount());
        coupon.setMaximumDiscountAmount(request.getMaximumDiscountAmount());
        coupon.setUsageLimit(request.getUsageLimit());
        coupon.setUsageLimitPerUser(request.getUsageLimitPerUser());
        coupon.setValidFrom(request.getValidFrom());
        coupon.setValidTo(request.getValidTo());
        coupon.setIsSingleUse(request.getIsSingleUse() != null && request.getIsSingleUse());

        return toDTO(couponRepository.save(coupon));
    }

    public List<CouponDTO> bulkGenerateCoupons(BulkCouponRequest request) {
        List<Coupon> coupons = new ArrayList<>();

        for (int i = 0; i < request.getCount(); i++) {
            Coupon coupon = new Coupon();
            coupon.setCode(request.getPrefix() + generateRandomCode(8));
            coupon.setName(request.getName());
            coupon.setDiscountType(request.getDiscountType());
            coupon.setDiscountValue(request.getDiscountValue());
            coupon.setValidFrom(request.getValidFrom());
            coupon.setValidTo(request.getValidTo());
            coupon.setUsageLimit(1); // Single use
            coupon.setIsSingleUse(true);
            coupons.add(coupon);
        }

        return couponRepository.saveAll(coupons).stream()
            .map(this::toDTO)
            .collect(Collectors.toList());
    }

    public DiscountResult applyCoupon(String code, Long userId, Long orderId, 
                                      BigDecimal orderAmount, List<Long> categoryIds) {
        Coupon coupon = couponRepository.findByCodeAndActiveTrue(code)
            .orElseThrow(() -> new CouponException("Invalid or inactive coupon code"));

        // Validate coupon
        validateCoupon(coupon, userId, orderId, orderAmount, categoryIds);

        // Calculate discount
        BigDecimal discount = calculateDiscount(coupon, orderAmount);

        // Record usage
        CouponUsage usage = new CouponUsage();
        usage.setCoupon(coupon);
        usage.setUserId(userId);
        usage.setOrderId(orderId);
        usage.setDiscountApplied(discount);
        couponUsageRepository.save(usage);

        // Increment usage count
        coupon.setUsageCount(coupon.getUsageCount() + 1);
        couponRepository.save(coupon);

        return new DiscountResult(coupon.getCode(), discount, orderAmount.subtract(discount));
    }

    public void validateCouponCode(String code, Long userId, BigDecimal orderAmount, List<Long> categoryIds) {
        Coupon coupon = couponRepository.findByCodeAndActiveTrue(code)
            .orElseThrow(() -> new CouponException("Invalid or inactive coupon code"));
        validateCoupon(coupon, userId, null, orderAmount, categoryIds);
    }

    private void validateCoupon(Coupon coupon, Long userId, Long orderId, 
                                 BigDecimal orderAmount, List<Long> categoryIds) {
        LocalDateTime now = LocalDateTime.now();

        if (coupon.getValidFrom() != null && now.isBefore(coupon.getValidFrom())) {
            throw new CouponException("Coupon is not yet valid");
        }

        if (coupon.getValidTo() != null && now.isAfter(coupon.getValidTo())) {
            throw new CouponException("Coupon has expired");
        }

        if (coupon.getUsageLimit() != null && coupon.getUsageCount() >= coupon.getUsageLimit()) {
            throw new CouponException("Coupon has reached its usage limit");
        }

        if (coupon.getMinimumOrderAmount() != null && 
            orderAmount.compareTo(coupon.getMinimumOrderAmount()) < 0) {
            throw new CouponException("Order amount is below minimum required: " + coupon.getMinimumOrderAmount());
        }

        if (coupon.getUsageLimitPerUser() != null) {
            int userUsageCount = couponUsageRepository.countByUserIdAndCouponId(userId, coupon.getId());
            if (userUsageCount >= coupon.getUsageLimitPerUser()) {
                throw new CouponException("Coupon usage limit per user exceeded");
            }
        }

        // Check category restriction
        if (!coupon.getApplicableCategories().isEmpty() && categoryIds != null) {
            boolean hasApplicableCategory = coupon.getApplicableCategories().stream()
                .anyMatch(c -> categoryIds.contains(c.getId()));
            if (!hasApplicableCategory) {
                throw new CouponException("Coupon not applicable for selected items");
            }
        }
    }

    private BigDecimal calculateDiscount(Coupon coupon, BigDecimal orderAmount) {
        BigDecimal discount;

        if (coupon.getDiscountType() == DiscountType.PERCENTAGE) {
            discount = orderAmount.multiply(coupon.getDiscountValue())
                .divide(new BigDecimal("100"), 2, RoundingMode.HALF_UP);

            // Apply maximum cap if set
            if (coupon.getMaximumDiscountAmount() != null &&
                discount.compareTo(coupon.getMaximumDiscountAmount()) > 0) {
                discount = coupon.getMaximumDiscountAmount();
            }
        } else {
            discount = coupon.getDiscountValue();
        }

        // Discount cannot exceed order amount
        return discount.min(orderAmount);
    }

    private String generateRandomCode(int length) {
        String chars = "ABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789";
        StringBuilder sb = new StringBuilder();
        for (int i = 0; i < length; i++) {
            sb.append(chars.charAt((int)(Math.random() * chars.length())));
        }
        return sb.toString();
    }

    private CouponDTO toDTO(Coupon coupon) {
        return CouponDTO.builder()
            .id(coupon.getId())
            .code(coupon.getCode())
            .name(coupon.getName())
            .discountType(coupon.getDiscountType())
            .discountValue(coupon.getDiscountValue())
            .minimumOrderAmount(coupon.getMinimumOrderAmount())
            .usageLimit(coupon.getUsageLimit())
            .usageCount(coupon.getUsageCount())
            .validFrom(coupon.getValidFrom())
            .validTo(coupon.getValidTo())
            .active(coupon.getActive())
            .build();
    }
}
```

### REST Controller

```java
// CouponController.java
@RestController
@RequestMapping("/api/v1/coupons")
@RequiredArgsConstructor
public class CouponController {

    private final CouponService couponService;

    @PostMapping
    @PreAuthorize("hasRole('ADMIN')")
    public ResponseEntity<CouponDTO> createCoupon(@Valid @RequestBody CreateCouponRequest request) {
        return ResponseEntity.status(HttpStatus.CREATED).body(couponService.createCoupon(request));
    }

    @PostMapping("/bulk-generate")
    @PreAuthorize("hasRole('ADMIN')")
    public ResponseEntity<List<CouponDTO>> bulkGenerate(@Valid @RequestBody BulkCouponRequest request) {
        return ResponseEntity.ok(couponService.bulkGenerateCoupons(request));
    }

    @GetMapping
    @PreAuthorize("hasRole('ADMIN')")
    public ResponseEntity<Page<CouponDTO>> listCoupons(
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size) {
        return ResponseEntity.ok(couponService.listCoupons(PageRequest.of(page, size)));
    }

    @PostMapping("/validate")
    public ResponseEntity<ValidateCouponResponse> validateCoupon(
            @AuthenticationPrincipal UserPrincipal user,
            @Valid @RequestBody ValidateCouponRequest request) {
        couponService.validateCouponCode(request.getCode(), user.getId(),
                                          request.getOrderAmount(), request.getCategoryIds());
        BigDecimal discount = couponService.previewDiscount(request.getCode(), request.getOrderAmount());
        return ResponseEntity.ok(new ValidateCouponResponse(true, discount, request.getOrderAmount().subtract(discount)));
    }

    @PostMapping("/apply")
    public ResponseEntity<DiscountResult> applyCoupon(
            @AuthenticationPrincipal UserPrincipal user,
            @Valid @RequestBody ApplyCouponRequest request) {
        return ResponseEntity.ok(couponService.applyCoupon(
            request.getCode(), user.getId(), request.getOrderId(),
            request.getOrderAmount(), request.getCategoryIds()));
    }

    @PatchMapping("/{id}/deactivate")
    @PreAuthorize("hasRole('ADMIN')")
    public ResponseEntity<Void> deactivateCoupon(@PathVariable Long id) {
        couponService.deactivateCoupon(id);
        return ResponseEntity.noContent().build();
    }
}
```

### Flyway Migration สำหรับ Coupons

```sql
-- V5__init_coupons.sql
CREATE TABLE coupons (
    id BIGSERIAL PRIMARY KEY,
    code VARCHAR(50) NOT NULL UNIQUE,
    name VARCHAR(255),
    description TEXT,
    discount_type VARCHAR(20) NOT NULL,
    discount_value DECIMAL(10,2) NOT NULL,
    minimum_order_amount DECIMAL(10,2),
    maximum_discount_amount DECIMAL(10,2),
    usage_limit INT,
    usage_count INT NOT NULL DEFAULT 0,
    usage_limit_per_user INT,
    valid_from TIMESTAMP,
    valid_to TIMESTAMP,
    active BOOLEAN DEFAULT TRUE,
    is_single_use BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE coupon_categories (
    coupon_id BIGINT NOT NULL REFERENCES coupons(id),
    category_id BIGINT NOT NULL REFERENCES categories(id),
    PRIMARY KEY (coupon_id, category_id)
);

CREATE TABLE coupon_usages (
    id BIGSERIAL PRIMARY KEY,
    coupon_id BIGINT NOT NULL REFERENCES coupons(id),
    user_id BIGINT NOT NULL,
    order_id BIGINT NOT NULL,
    discount_applied DECIMAL(10,2) NOT NULL,
    used_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE (coupon_id, user_id, order_id)
);

CREATE INDEX idx_coupon_code ON coupons(code);
CREATE INDEX idx_coupon_active ON coupons(active, valid_from, valid_to);
CREATE INDEX idx_coupon_usage_user ON coupon_usages(user_id, coupon_id);
```

### ตัวอย่าง API Calls

```bash
# สร้างคูปอง
curl -X POST http://localhost:8080/api/v1/coupons \
  -H "Authorization: Bearer {admin_token}" \
  -H "Content-Type: application/json" \
  -d '{
    "code": "SAVE20",
    "name": "ลด 20% สำหรับสมาชิกใหม่",
    "discountType": "PERCENTAGE",
    "discountValue": 20,
    "minimumOrderAmount": 500,
    "maximumDiscountAmount": 200,
    "usageLimit": 100,
    "usageLimitPerUser": 1,
    "validFrom": "2024-01-01T00:00:00",
    "validTo": "2024-12-31T23:59:59"
  }'

# สร้างคูปองจำนวนมาก
curl -X POST http://localhost:8080/api/v1/coupons/bulk-generate \
  -H "Authorization: Bearer {admin_token}" \
  -H "Content-Type: application/json" \
  -d '{
    "count": 50,
    "prefix": "PROMO-",
    "name": "Promotional Coupon",
    "discountType": "FIXED_AMOUNT",
    "discountValue": 100,
    "validFrom": "2024-06-01T00:00:00",
    "validTo": "2024-06-30T23:59:59"
  }'

# ตรวจสอบคูปอง
curl -X POST http://localhost:8080/api/v1/coupons/validate \
  -H "Authorization: Bearer {user_token}" \
  -H "Content-Type: application/json" \
  -d '{"code": "SAVE20", "orderAmount": 1500}'

# ใช้คูปอง
curl -X POST http://localhost:8080/api/v1/coupons/apply \
  -H "Authorization: Bearer {user_token}" \
  -H "Content-Type: application/json" \
  -d '{"code": "SAVE20", "orderId": 123, "orderAmount": 1500}'
```

---

*[← Part 100: หลักสูตรสมบูรณ์](./part-100-whats-next.md) | [Part 102: Business Systems →](./part-102-business-systems.md)*

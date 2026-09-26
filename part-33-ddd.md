# Part 33: Domain-Driven Design (DDD)
## ขั้นตอนที่ 931-970

> **ระดับ:** ระดับโลก (World-Class)  
> **เวลาเรียน:** 8-10 ชั่วโมง  
> **เป้าหมาย:** ออกแบบ software ด้วย DDD

---

## ขั้นตอนที่ 931: DDD Concepts

```
Domain-Driven Design:
  ทำให้ code สะท้อน business domain ที่แท้จริง

Core Concepts:
  Domain       = ส่วนของ business ที่เราแก้ปัญหา
  Subdomain    = แบ่ง domain ออกเป็นส่วนย่อย
  Bounded Context = ขอบเขตของ model (1 context = 1 microservice)
  Ubiquitous Language = ภาษาร่วมระหว่าง dev + business

Building Blocks:
  Entity       = Object ที่มี identity (id) - เปลี่ยนได้
  Value Object = Object ที่ไม่มี id - ไม่เปลี่ยนได้
  Aggregate    = กลุ่ม Entity/VO ที่มี Aggregate Root
  Repository   = ส่วน persistence ของ Aggregate
  Domain Event = เหตุการณ์ที่เกิดใน domain
  Domain Service = Logic ที่ไม่เหมาะอยู่ใน Entity

E-Commerce Example:
  Order Bounded Context:
    Aggregate Root: Order
    Entities: OrderItem
    Value Objects: Money, Address, OrderStatus
    Domain Events: OrderCreated, OrderShipped, OrderCancelled
```

---

## ขั้นตอนที่ 932: Value Objects

```java
// Value Object - immutable, no identity
public record Money(BigDecimal amount, Currency currency) {
    
    // Self-validate in constructor
    public Money {
        Objects.requireNonNull(amount, "Amount must not be null");
        Objects.requireNonNull(currency, "Currency must not be null");
        if (amount.compareTo(BigDecimal.ZERO) < 0) {
            throw new IllegalArgumentException("Amount cannot be negative");
        }
        amount = amount.setScale(currency.getDefaultFractionDigits(), RoundingMode.HALF_UP);
    }
    
    public static Money of(BigDecimal amount, Currency currency) {
        return new Money(amount, currency);
    }
    
    public static Money thb(BigDecimal amount) {
        return new Money(amount, Currency.getInstance("THB"));
    }
    
    public static Money zero(Currency currency) {
        return new Money(BigDecimal.ZERO, currency);
    }
    
    public Money add(Money other) {
        assertSameCurrency(other);
        return new Money(amount.add(other.amount), currency);
    }
    
    public Money subtract(Money other) {
        assertSameCurrency(other);
        return new Money(amount.subtract(other.amount), currency);
    }
    
    public Money multiply(int factor) {
        return new Money(amount.multiply(BigDecimal.valueOf(factor)), currency);
    }
    
    public boolean isGreaterThan(Money other) {
        assertSameCurrency(other);
        return amount.compareTo(other.amount) > 0;
    }
    
    private void assertSameCurrency(Money other) {
        if (!currency.equals(other.currency)) {
            throw new IllegalArgumentException("Cannot operate on different currencies");
        }
    }
}

// Address Value Object
@Embeddable
public record Address(
    String street,
    String city,
    String state,
    String country,
    String postalCode
) {
    public Address {
        Objects.requireNonNull(street, "Street is required");
        Objects.requireNonNull(city, "City is required");
        Objects.requireNonNull(country, "Country is required");
    }
    
    public String formatted() {
        return String.format("%s, %s, %s %s, %s", street, city, state, postalCode, country);
    }
}
```

---

## ขั้นตอนที่ 933: Domain Events

```java
// Base domain event
public abstract class DomainEvent {
    private final String eventId = UUID.randomUUID().toString();
    private final LocalDateTime occurredAt = LocalDateTime.now();
    
    public String getEventId() { return eventId; }
    public LocalDateTime getOccurredAt() { return occurredAt; }
    public abstract String getEventType();
}

// Specific events
public class OrderCreatedEvent extends DomainEvent {
    private final Long orderId;
    private final Long userId;
    private final Money totalAmount;
    private final List<OrderItemSnapshot> items;
    
    public OrderCreatedEvent(Order order) {
        this.orderId = order.getId();
        this.userId = order.getUserId();
        this.totalAmount = order.getTotalAmount();
        this.items = order.getItems().stream()
            .map(OrderItemSnapshot::from)
            .toList();
    }
    
    @Override
    public String getEventType() { return "ORDER_CREATED"; }
}

public class OrderStatusChangedEvent extends DomainEvent {
    private final Long orderId;
    private final OrderStatus from;
    private final OrderStatus to;
    
    @Override
    public String getEventType() { return "ORDER_STATUS_CHANGED"; }
}
```

---

## ขั้นตอนที่ 934: Aggregate Root

```java
@Entity
@Table(name = "orders")
@Getter
public class Order extends AbstractAggregateRoot<Order> {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(name = "order_number", unique = true, nullable = false)
    private String orderNumber;
    
    @Column(name = "user_id", nullable = false)
    private Long userId;
    
    @Embedded
    @AttributeOverrides({
        @AttributeOverride(name = "amount", column = @Column(name = "total_amount")),
        @AttributeOverride(name = "currency", column = @Column(name = "currency"))
    })
    private Money totalAmount;
    
    @Enumerated(EnumType.STRING)
    private OrderStatus status;
    
    @Embedded
    private Address shippingAddress;
    
    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<OrderItem> items = new ArrayList<>();
    
    @CreationTimestamp
    private LocalDateTime createdAt;
    
    @UpdateTimestamp
    private LocalDateTime updatedAt;
    
    // Factory method
    public static Order create(Long userId, Address shippingAddress) {
        Order order = new Order();
        order.userId = userId;
        order.orderNumber = generateOrderNumber();
        order.status = OrderStatus.PENDING;
        order.shippingAddress = shippingAddress;
        order.totalAmount = Money.thb(BigDecimal.ZERO);
        
        // Register domain event
        order.registerEvent(new OrderCreatedEvent(order));
        
        return order;
    }
    
    // Business methods - protect invariants
    public void addItem(Long productId, String productName, Money unitPrice, int quantity) {
        if (status != OrderStatus.PENDING) {
            throw new DomainException("Cannot add items to " + status + " order");
        }
        
        // Find existing item or create new
        Optional<OrderItem> existing = items.stream()
            .filter(i -> i.getProductId().equals(productId))
            .findFirst();
        
        if (existing.isPresent()) {
            existing.get().increaseQuantity(quantity);
        } else {
            items.add(new OrderItem(this, productId, productName, unitPrice, quantity));
        }
        
        recalculateTotal();
    }
    
    public void removeItem(Long productId) {
        if (status != OrderStatus.PENDING) {
            throw new DomainException("Cannot remove items from " + status + " order");
        }
        
        items.removeIf(i -> i.getProductId().equals(productId));
        recalculateTotal();
    }
    
    public void confirm() {
        if (status != OrderStatus.PENDING) {
            throw new DomainException("Can only confirm PENDING orders");
        }
        if (items.isEmpty()) {
            throw new DomainException("Cannot confirm empty order");
        }
        
        changeStatus(OrderStatus.CONFIRMED);
    }
    
    public void ship(String trackingNumber) {
        if (status != OrderStatus.CONFIRMED) {
            throw new DomainException("Can only ship CONFIRMED orders");
        }
        
        changeStatus(OrderStatus.SHIPPED);
    }
    
    public void deliver() {
        if (status != OrderStatus.SHIPPED) {
            throw new DomainException("Can only deliver SHIPPED orders");
        }
        
        changeStatus(OrderStatus.DELIVERED);
    }
    
    public void cancel(String reason) {
        if (status == OrderStatus.DELIVERED || status == OrderStatus.CANCELLED) {
            throw new DomainException("Cannot cancel " + status + " order");
        }
        
        changeStatus(OrderStatus.CANCELLED);
    }
    
    private void changeStatus(OrderStatus newStatus) {
        OrderStatus oldStatus = this.status;
        this.status = newStatus;
        registerEvent(new OrderStatusChangedEvent(id, oldStatus, newStatus));
    }
    
    private void recalculateTotal() {
        totalAmount = items.stream()
            .map(OrderItem::getSubtotal)
            .reduce(Money.thb(BigDecimal.ZERO), Money::add);
    }
    
    private static String generateOrderNumber() {
        return "ORD-" + LocalDate.now().format(DateTimeFormatter.ofPattern("yyyyMMdd")) 
            + "-" + UUID.randomUUID().toString().substring(0, 8).toUpperCase();
    }
}
```

---

## ขั้นตอนที่ 935: Application Service

```java
@Service
@RequiredArgsConstructor
@Transactional
public class OrderApplicationService {
    
    private final OrderRepository orderRepository;
    private final ProductRepository productRepository;
    private final UserRepository userRepository;
    private final ApplicationEventPublisher eventPublisher;
    
    public OrderResponse createOrder(Long userId, CreateOrderCommand command) {
        // Validate user
        User user = userRepository.findById(userId)
            .orElseThrow(() -> new ResourceNotFoundException("User not found"));
        
        // Create order aggregate
        Order order = Order.create(userId, command.shippingAddress());
        
        // Add items
        for (var itemCmd : command.items()) {
            Product product = productRepository.findById(itemCmd.productId())
                .orElseThrow(() -> new ResourceNotFoundException("Product not found: " + itemCmd.productId()));
            
            order.addItem(
                product.getId(),
                product.getName(),
                Money.thb(product.getPrice()),
                itemCmd.quantity()
            );
        }
        
        order.confirm();
        
        // Save - Spring Data handles domain events via @DomainEvents
        Order saved = orderRepository.save(order);
        
        return orderMapper.toResponse(saved);
    }
    
    public void cancelOrder(Long orderId, Long userId, String reason) {
        Order order = findOrder(orderId, userId);
        order.cancel(reason);
        orderRepository.save(order);
    }
    
    private Order findOrder(Long orderId, Long userId) {
        return orderRepository.findByIdAndUserId(orderId, userId)
            .orElseThrow(() -> new ResourceNotFoundException("Order not found"));
    }
}
```

---

## ขั้นตอนที่ 936-970: Domain Event Handlers

```java
// Spring publishes domain events automatically after save (via @DomainEvents)
@Component
@RequiredArgsConstructor
@Slf4j
public class OrderEventHandler {
    
    private final NotificationService notificationService;
    private final InventoryService inventoryService;
    private final EmailService emailService;
    
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void onOrderCreated(OrderCreatedEvent event) {
        log.info("Order created: {}", event.getOrderId());
        
        // These run after transaction commits successfully
        notificationService.notifyOrderCreated(event);
        inventoryService.reserveStock(event);
    }
    
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    @Async
    public void sendOrderConfirmationEmail(OrderCreatedEvent event) {
        emailService.sendOrderConfirmation(event.getOrderId());
    }
    
    @EventListener
    public void onStatusChanged(OrderStatusChangedEvent event) {
        if (event.getTo() == OrderStatus.SHIPPED) {
            notificationService.notifyOrderShipped(event.getOrderId());
        }
    }
}
```

---

*[← Part 32: Spring WebFlux](./part-32-webflux.md) | [Part 34: CQRS Pattern →](./part-34-cqrs.md)*

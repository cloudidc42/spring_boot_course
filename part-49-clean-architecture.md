# Part 49: Clean Architecture with Spring Boot
## ขั้นตอนที่ 1561-1600

> **ระดับ:** ระดับโลก (World-Class)  
> **เวลาเรียน:** 8-10 ชั่วโมง  
> **เป้าหมาย:** Structure code ตาม Clean Architecture principles

---

## ขั้นตอนที่ 1561: Clean Architecture Concepts

```
Clean Architecture (Robert C. Martin):
  
  Dependency Rule:
  Source code dependencies MUST point inward
  Outer layers depend on inner layers, never reverse
  
  Layers (outer → inner):
  ┌────────────────────────────────────┐
  │  Frameworks & Drivers              │
  │  (Spring, JPA, REST, DB)           │
  │  ┌────────────────────────────┐    │
  │  │  Interface Adapters        │    │
  │  │  (Controllers, Presenters) │    │
  │  │  ┌──────────────────────┐ │    │
  │  │  │  Use Cases           │ │    │
  │  │  │  (Application Logic) │ │    │
  │  │  │  ┌──────────────┐   │ │    │
  │  │  │  │  Entities    │   │ │    │
  │  │  │  │  (Domain)    │   │ │    │
  │  │  │  └──────────────┘   │ │    │
  │  │  └──────────────────────┘ │    │
  │  └────────────────────────────┘    │
  └────────────────────────────────────┘
  
Benefits:
  ✅ Testable without frameworks
  ✅ Independent of database
  ✅ Independent of UI
  ✅ Independent of external agencies
  ✅ Easy to understand business logic
```

---

## ขั้นตอนที่ 1562: Clean Architecture Project Structure

```
src/main/java/com/myapp/
├── domain/                   # Pure business logic (no dependencies)
│   ├── entity/
│   │   ├── Order.java
│   │   └── Product.java
│   ├── valueobject/
│   │   ├── Money.java
│   │   └── Address.java
│   ├── repository/
│   │   ├── OrderRepository.java     (interface only!)
│   │   └── ProductRepository.java   (interface only!)
│   ├── service/
│   │   └── OrderDomainService.java
│   └── event/
│       ├── OrderCreatedEvent.java
│       └── OrderStatusChangedEvent.java
│
├── application/              # Use cases (orchestrate domain)
│   ├── usecase/
│   │   ├── CreateOrderUseCase.java
│   │   ├── CancelOrderUseCase.java
│   │   └── GetOrderUseCase.java
│   ├── port/
│   │   ├── input/            # Driven ports (API)
│   │   │   └── OrderInputPort.java
│   │   └── output/           # Driving ports (DB, messaging)
│   │       ├── OrderOutputPort.java
│   │       └── PaymentOutputPort.java
│   └── dto/
│       ├── CreateOrderCommand.java
│       └── OrderResult.java
│
├── infrastructure/           # Implementations
│   ├── persistence/
│   │   ├── entity/
│   │   │   └── OrderJpaEntity.java
│   │   ├── repository/
│   │   │   └── OrderJpaRepository.java
│   │   └── adapter/
│   │       └── OrderRepositoryAdapter.java  (implements domain port)
│   ├── messaging/
│   │   └── KafkaOrderPublisher.java
│   ├── payment/
│   │   └── StripePaymentAdapter.java
│   └── config/
│       └── BeanConfig.java
│
└── presentation/             # Web layer
    ├── rest/
    │   ├── OrderController.java
    │   └── mapper/
    │       └── OrderWebMapper.java
    └── dto/
        ├── CreateOrderRequest.java
        └── OrderResponse.java
```

---

## ขั้นตอนที่ 1563: Domain Layer (No framework dependencies)

```java
// Domain Entity - pure Java, no Spring annotations
public class Order {
    
    private final OrderId id;
    private final CustomerId customerId;
    private OrderStatus status;
    private final List<OrderItem> items;
    private Money totalAmount;
    private Address deliveryAddress;
    private final OrderCreatedAt createdAt;
    
    private final List<OrderEvent> domainEvents = new ArrayList<>();
    
    // Factory method
    public static Order create(CustomerId customerId, Address address, List<OrderItem> items) {
        if (items == null || items.isEmpty()) {
            throw new OrderDomainException("Order must have at least one item");
        }
        
        Order order = new Order(
            new OrderId(UUID.randomUUID()),
            customerId,
            OrderStatus.PENDING,
            items,
            address,
            new OrderCreatedAt(LocalDateTime.now())
        );
        
        order.calculateTotal();
        order.domainEvents.add(new OrderCreatedEvent(order.id, customerId, order.totalAmount));
        
        return order;
    }
    
    public void confirm() {
        if (status != OrderStatus.PENDING) {
            throw new OrderDomainException("Can only confirm PENDING orders, current: " + status);
        }
        status = OrderStatus.CONFIRMED;
        domainEvents.add(new OrderConfirmedEvent(id));
    }
    
    public void ship(TrackingNumber trackingNumber) {
        if (status != OrderStatus.CONFIRMED) {
            throw new OrderDomainException("Can only ship CONFIRMED orders, current: " + status);
        }
        status = OrderStatus.SHIPPED;
        domainEvents.add(new OrderShippedEvent(id, trackingNumber));
    }
    
    private void calculateTotal() {
        totalAmount = items.stream()
            .map(OrderItem::subtotal)
            .reduce(Money.ZERO, Money::add);
    }
    
    public List<OrderEvent> getDomainEvents() {
        return Collections.unmodifiableList(domainEvents);
    }
    
    public void clearDomainEvents() {
        domainEvents.clear();
    }
    
    // Getters only
    public OrderId getId() { return id; }
    public OrderStatus getStatus() { return status; }
    public Money getTotalAmount() { return totalAmount; }
}
```

---

## ขั้นตอนที่ 1564: Application Layer (Use Cases)

```java
// Port (interface) - what use case needs from infrastructure
public interface OrderRepositoryPort {
    Order save(Order order);
    Optional<Order> findById(OrderId orderId);
    List<Order> findByCustomerId(CustomerId customerId);
}

public interface PaymentPort {
    PaymentResult processPayment(CustomerId customerId, Money amount, PaymentDetails details);
}

public interface OrderEventPublisherPort {
    void publish(List<OrderEvent> events);
}

// Use Case
@Service
@RequiredArgsConstructor
public class CreateOrderUseCase {
    
    private final OrderRepositoryPort orderRepository;
    private final ProductRepositoryPort productRepository;
    private final PaymentPort paymentPort;
    private final OrderEventPublisherPort eventPublisher;
    
    public CreateOrderResult execute(CreateOrderCommand command) {
        // Validate products exist and have stock
        List<OrderItem> items = command.items().stream().map(itemCmd -> {
            Product product = productRepository.findById(new ProductId(itemCmd.productId()))
                .orElseThrow(() -> new ProductNotFoundException(itemCmd.productId()));
            
            if (product.getStock() < itemCmd.quantity()) {
                throw new InsufficientStockException(product.getId(), itemCmd.quantity());
            }
            
            return new OrderItem(
                product.getId(),
                product.getName(),
                product.getPrice(),
                itemCmd.quantity()
            );
        }).toList();
        
        // Create order domain object
        Order order = Order.create(
            new CustomerId(command.customerId()),
            command.address(),
            items
        );
        
        // Process payment
        PaymentResult payment = paymentPort.processPayment(
            order.getCustomerId(),
            order.getTotalAmount(),
            command.paymentDetails()
        );
        
        if (!payment.isSuccessful()) {
            throw new PaymentFailedException(payment.getFailureReason());
        }
        
        order.confirm();
        
        // Persist
        Order saved = orderRepository.save(order);
        
        // Publish events
        eventPublisher.publish(saved.getDomainEvents());
        saved.clearDomainEvents();
        
        return new CreateOrderResult(saved.getId().value(), saved.getStatus());
    }
}
```

---

## ขั้นตอนที่ 1565: Infrastructure Layer (Adapters)

```java
// Adapter: implements domain port using JPA
@Component
@RequiredArgsConstructor
public class OrderRepositoryAdapter implements OrderRepositoryPort {
    
    private final OrderJpaRepository jpaRepository;
    private final OrderPersistenceMapper mapper;
    
    @Override
    public Order save(Order order) {
        OrderJpaEntity entity = mapper.toEntity(order);
        OrderJpaEntity saved = jpaRepository.save(entity);
        return mapper.toDomain(saved);
    }
    
    @Override
    public Optional<Order> findById(OrderId orderId) {
        return jpaRepository.findById(orderId.value())
            .map(mapper::toDomain);
    }
    
    @Override
    public List<Order> findByCustomerId(CustomerId customerId) {
        return jpaRepository.findByCustomerId(customerId.value()).stream()
            .map(mapper::toDomain)
            .toList();
    }
}

// Mapper: converts between domain and JPA entities
@Component
public class OrderPersistenceMapper {
    
    public Order toDomain(OrderJpaEntity entity) {
        return Order.reconstitute(
            new OrderId(entity.getId()),
            new CustomerId(entity.getCustomerId()),
            OrderStatus.valueOf(entity.getStatus()),
            entity.getItems().stream().map(this::itemToDomain).toList(),
            addressToDomain(entity),
            new OrderCreatedAt(entity.getCreatedAt())
        );
    }
    
    public OrderJpaEntity toEntity(Order order) {
        OrderJpaEntity entity = new OrderJpaEntity();
        entity.setId(order.getId().value());
        entity.setCustomerId(order.getCustomerId().value());
        entity.setStatus(order.getStatus().name());
        entity.setTotalAmount(order.getTotalAmount().amount());
        entity.setCurrency(order.getTotalAmount().currency().getCurrencyCode());
        return entity;
    }
}
```

---

## ขั้นตอนที่ 1566: Presentation Layer

```java
// Web Controller calls Use Case
@RestController
@RequestMapping("/api/v1/orders")
@RequiredArgsConstructor
public class OrderController {
    
    private final CreateOrderUseCase createOrderUseCase;
    private final GetOrderUseCase getOrderUseCase;
    private final OrderWebMapper webMapper;
    
    @PostMapping
    public ResponseEntity<OrderWebResponse> createOrder(
        @Valid @RequestBody CreateOrderWebRequest webRequest,
        Authentication auth
    ) {
        CreateOrderCommand command = webMapper.toCommand(webRequest, getUserId(auth));
        CreateOrderResult result = createOrderUseCase.execute(command);
        
        return ResponseEntity
            .created(URI.create("/api/v1/orders/" + result.orderId()))
            .body(webMapper.toResponse(result));
    }
    
    @GetMapping("/{id}")
    public ResponseEntity<OrderWebResponse> getOrder(@PathVariable UUID id) {
        OrderResult result = getOrderUseCase.execute(new GetOrderQuery(new OrderId(id)));
        return ResponseEntity.ok(webMapper.toResponse(result));
    }
}
```

---

## ขั้นตอนที่ 1567-1600: Testing Clean Architecture

```java
// Domain tests - zero Spring context!
class OrderTest {
    
    @Test
    void create_WithValidItems_ShouldBePending() {
        List<OrderItem> items = List.of(
            new OrderItem(new ProductId(1L), "iPhone", Money.thb(BigDecimal.valueOf(45000)), 1)
        );
        
        Order order = Order.create(
            new CustomerId(1L),
            new Address("123 Main St", "Bangkok", null, "Thailand", "10000"),
            items
        );
        
        assertThat(order.getStatus()).isEqualTo(OrderStatus.PENDING);
        assertThat(order.getTotalAmount().amount()).isEqualByComparingTo("45000");
        assertThat(order.getDomainEvents()).hasSize(1);
        assertThat(order.getDomainEvents().get(0)).isInstanceOf(OrderCreatedEvent.class);
    }
    
    @Test
    void confirm_WhenPending_ShouldSucceed() {
        Order order = createTestOrder();
        order.confirm();
        assertThat(order.getStatus()).isEqualTo(OrderStatus.CONFIRMED);
    }
    
    @Test
    void confirm_WhenNotPending_ShouldThrow() {
        Order order = createTestOrder();
        order.confirm();  // Set to CONFIRMED
        
        assertThatThrownBy(order::confirm)
            .isInstanceOf(OrderDomainException.class)
            .hasMessageContaining("PENDING");
    }
}

// Use Case tests - mock ports
@ExtendWith(MockitoExtension.class)
class CreateOrderUseCaseTest {
    
    @Mock private OrderRepositoryPort orderRepository;
    @Mock private PaymentPort paymentPort;
    @Mock private OrderEventPublisherPort eventPublisher;
    
    @InjectMocks private CreateOrderUseCase useCase;
    
    @Test
    void execute_WithValidOrder_ShouldReturnSuccess() {
        when(paymentPort.processPayment(any(), any(), any()))
            .thenReturn(PaymentResult.success("PAY-123"));
        when(orderRepository.save(any())).thenAnswer(i -> i.getArguments()[0]);
        
        CreateOrderResult result = useCase.execute(validCommand());
        
        assertThat(result.orderId()).isNotNull();
        verify(eventPublisher).publish(any());
    }
    
    @Test
    void execute_WhenPaymentFails_ShouldThrow() {
        when(paymentPort.processPayment(any(), any(), any()))
            .thenReturn(PaymentResult.failed("Insufficient funds"));
        
        assertThatThrownBy(() -> useCase.execute(validCommand()))
            .isInstanceOf(PaymentFailedException.class);
        
        verify(orderRepository, never()).save(any());
    }
}
```

---

*[← Part 48: Advanced Caching](./part-48-advanced-caching.md) | [Part 50: Production Runbook →](./part-50-production-runbook.md)*

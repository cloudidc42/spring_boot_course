# Part 97: Capstone Project Implementation Guide
## ขั้นตอนที่ 3481-3520

**ระดับ: World-Class Professional**

---

## บทนำ: การ Implement Capstone Project

ใน Part นี้เราจะ implement E-Commerce Platform ที่ออกแบบไว้ใน Part 96 โดยใช้หลักการ **Domain-Driven Design (DDD)** เพื่อให้โค้ดมีโครงสร้างที่ชัดเจนและบำรุงรักษาง่าย

---

## ขั้นตอนที่ 3481: User Service Implementation

### Project Structure

```
user-service/
├── src/main/java/com/ecommerce/user/
│   ├── domain/
│   │   ├── model/
│   │   │   ├── User.java
│   │   │   ├── UserAddress.java
│   │   │   └── UserStatus.java
│   │   ├── repository/
│   │   │   └── UserRepository.java
│   │   ├── service/
│   │   │   └── UserDomainService.java
│   │   └── event/
│   │       ├── UserRegisteredEvent.java
│   │       └── UserUpdatedEvent.java
│   ├── application/
│   │   ├── command/
│   │   │   ├── RegisterUserCommand.java
│   │   │   └── UpdateProfileCommand.java
│   │   ├── query/
│   │   │   └── GetUserQuery.java
│   │   └── service/
│   │       ├── UserCommandService.java
│   │       └── UserQueryService.java
│   ├── infrastructure/
│   │   ├── persistence/
│   │   │   └── JpaUserRepository.java
│   │   ├── messaging/
│   │   │   └── UserEventPublisher.java
│   │   └── cache/
│   │       └── UserCacheService.java
│   └── presentation/
│       ├── controller/
│       │   └── UserController.java
│       └── dto/
│           ├── RegisterUserRequest.java
│           └── UserResponse.java
```

### Domain Model

```java
// domain/model/User.java
package com.ecommerce.user.domain.model;

import com.ecommerce.common.domain.BaseEntity;
import jakarta.persistence.*;
import lombok.*;
import org.springframework.security.crypto.password.PasswordEncoder;

import java.time.Instant;
import java.util.ArrayList;
import java.util.List;

@Entity
@Table(name = "users")
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class User extends BaseEntity {

    @Column(name = "email", unique = true, nullable = false)
    private String email;

    @Column(name = "username", unique = true, nullable = false)
    private String username;

    @Column(name = "password_hash", nullable = false)
    private String passwordHash;

    @Column(name = "first_name")
    private String firstName;

    @Column(name = "last_name")
    private String lastName;

    @Column(name = "phone")
    private String phone;

    @Enumerated(EnumType.STRING)
    @Column(name = "status", nullable = false)
    private UserStatus status = UserStatus.ACTIVE;

    @Column(name = "email_verified")
    private boolean emailVerified = false;

    @Column(name = "last_login_at")
    private Instant lastLoginAt;

    @OneToMany(mappedBy = "user", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<UserAddress> addresses = new ArrayList<>();

    @Transient
    private List<Object> domainEvents = new ArrayList<>();

    // Factory Method - เป็น DDD Best Practice
    public static User register(
        String email,
        String username,
        String rawPassword,
        String firstName,
        String lastName,
        PasswordEncoder passwordEncoder
    ) {
        // Validate Business Rules
        validateEmail(email);
        validateUsername(username);
        validatePassword(rawPassword);

        User user = new User();
        user.email = email.toLowerCase().trim();
        user.username = username.trim();
        user.passwordHash = passwordEncoder.encode(rawPassword);
        user.firstName = firstName;
        user.lastName = lastName;
        user.status = UserStatus.ACTIVE;

        // Raise Domain Event
        user.domainEvents.add(new UserRegisteredEvent(user.getId(), email, username));

        return user;
    }

    // Business Method
    public void updateProfile(String firstName, String lastName, String phone) {
        this.firstName = firstName;
        this.lastName = lastName;
        this.phone = phone;
        this.domainEvents.add(new UserProfileUpdatedEvent(this.getId()));
    }

    public void recordLogin() {
        this.lastLoginAt = Instant.now();
    }

    public void verifyEmail() {
        if (this.emailVerified) {
            throw new IllegalStateException("Email already verified");
        }
        this.emailVerified = true;
        this.domainEvents.add(new UserEmailVerifiedEvent(this.getId(), this.email));
    }

    public boolean isActive() {
        return this.status == UserStatus.ACTIVE;
    }

    public List<Object> pullDomainEvents() {
        List<Object> events = new ArrayList<>(this.domainEvents);
        this.domainEvents.clear();
        return events;
    }

    // Validation Methods
    private static void validateEmail(String email) {
        if (email == null || !email.matches("^[\\w.-]+@[\\w.-]+\\.[a-zA-Z]{2,}$")) {
            throw new InvalidEmailException("Invalid email format: " + email);
        }
    }

    private static void validateUsername(String username) {
        if (username == null || username.length() < 3 || username.length() > 50) {
            throw new InvalidUsernameException("Username must be 3-50 characters");
        }
        if (!username.matches("^[a-zA-Z0-9_-]+$")) {
            throw new InvalidUsernameException("Username can only contain letters, numbers, _ and -");
        }
    }

    private static void validatePassword(String password) {
        if (password == null || password.length() < 8) {
            throw new WeakPasswordException("Password must be at least 8 characters");
        }
    }
}
```

### Application Service

```java
// application/service/UserCommandService.java
package com.ecommerce.user.application.service;

import com.ecommerce.user.application.command.RegisterUserCommand;
import com.ecommerce.user.domain.model.User;
import com.ecommerce.user.domain.repository.UserRepository;
import com.ecommerce.user.infrastructure.messaging.UserEventPublisher;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.UUID;

@Slf4j
@Service
@RequiredArgsConstructor
public class UserCommandService {

    private final UserRepository userRepository;
    private final PasswordEncoder passwordEncoder;
    private final UserEventPublisher eventPublisher;

    @Transactional
    public UUID registerUser(RegisterUserCommand command) {
        log.info("Registering user with email: {}", command.email());

        // Check Uniqueness
        if (userRepository.existsByEmail(command.email())) {
            throw new EmailAlreadyExistsException(command.email());
        }
        if (userRepository.existsByUsername(command.username())) {
            throw new UsernameAlreadyExistsException(command.username());
        }

        // Create Domain Object
        User user = User.register(
            command.email(),
            command.username(),
            command.password(),
            command.firstName(),
            command.lastName(),
            passwordEncoder
        );

        // Persist
        User savedUser = userRepository.save(user);

        // Publish Events (Transactional Outbox Pattern)
        savedUser.pullDomainEvents().forEach(event ->
            eventPublisher.publish(event)
        );

        log.info("User registered successfully with ID: {}", savedUser.getId());
        return savedUser.getId();
    }

    @Transactional
    public void updateProfile(UUID userId, UpdateProfileCommand command) {
        User user = userRepository.findById(userId)
            .orElseThrow(() -> new UserNotFoundException(userId));

        user.updateProfile(command.firstName(), command.lastName(), command.phone());
        userRepository.save(user);

        user.pullDomainEvents().forEach(event -> eventPublisher.publish(event));
    }
}
```

---

## ขั้นตอนที่ 3482: Event-Driven Order Processing

### Order Domain

```java
// order-service/domain/model/Order.java
package com.ecommerce.order.domain.model;

import com.ecommerce.common.domain.BaseEntity;
import jakarta.persistence.*;
import lombok.*;

import java.math.BigDecimal;
import java.time.Instant;
import java.util.*;

@Entity
@Table(name = "orders")
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class Order extends BaseEntity {

    @Column(name = "order_number", unique = true, nullable = false)
    private String orderNumber;

    @Column(name = "user_id", nullable = false)
    private UUID userId;

    @Enumerated(EnumType.STRING)
    @Column(name = "status", nullable = false)
    private OrderStatus status = OrderStatus.PENDING;

    @Column(name = "subtotal", nullable = false, precision = 12, scale = 2)
    private BigDecimal subtotal;

    @Column(name = "discount_amount", precision = 12, scale = 2)
    private BigDecimal discountAmount = BigDecimal.ZERO;

    @Column(name = "shipping_cost", precision = 12, scale = 2)
    private BigDecimal shippingCost = BigDecimal.ZERO;

    @Column(name = "tax_amount", precision = 12, scale = 2)
    private BigDecimal taxAmount = BigDecimal.ZERO;

    @Column(name = "total_amount", nullable = false, precision = 12, scale = 2)
    private BigDecimal totalAmount;

    @Column(name = "currency", nullable = false)
    private String currency = "THB";

    @Column(name = "shipping_address", columnDefinition = "jsonb")
    @Convert(converter = JsonbConverter.class)
    private Address shippingAddress;

    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
    @OrderBy("id ASC")
    private List<OrderItem> items = new ArrayList<>();

    @Column(name = "idempotency_key", unique = true)
    private String idempotencyKey;

    @Transient
    private List<DomainEvent> domainEvents = new ArrayList<>();

    // Factory Method
    public static Order create(
        UUID userId,
        List<OrderItemData> itemsData,
        Address shippingAddress,
        String idempotencyKey
    ) {
        if (itemsData == null || itemsData.isEmpty()) {
            throw new EmptyOrderException("Order must have at least one item");
        }

        Order order = new Order();
        order.orderNumber = generateOrderNumber();
        order.userId = userId;
        order.shippingAddress = shippingAddress;
        order.idempotencyKey = idempotencyKey;
        order.status = OrderStatus.PENDING;

        // Add Items
        for (OrderItemData itemData : itemsData) {
            order.items.add(OrderItem.create(order, itemData));
        }

        // Calculate Totals
        order.recalculateTotals();

        // Raise Event
        order.domainEvents.add(OrderCreatedEvent.builder()
            .orderId(order.getId().toString())
            .orderNumber(order.orderNumber)
            .userId(userId.toString())
            .totalAmount(order.totalAmount)
            .currency(order.currency)
            .items(itemsData)
            .build());

        return order;
    }

    // State Machine Transitions
    public void confirm() {
        validateTransition(OrderStatus.CONFIRMED);
        this.status = OrderStatus.CONFIRMED;
        domainEvents.add(new OrderConfirmedEvent(getId().toString(), orderNumber));
    }

    public void startProcessing() {
        validateTransition(OrderStatus.PROCESSING);
        this.status = OrderStatus.PROCESSING;
    }

    public void ship(String trackingNumber) {
        validateTransition(OrderStatus.SHIPPED);
        this.status = OrderStatus.SHIPPED;
        domainEvents.add(OrderShippedEvent.builder()
            .orderId(getId().toString())
            .trackingNumber(trackingNumber)
            .build());
    }

    public void deliver() {
        validateTransition(OrderStatus.DELIVERED);
        this.status = OrderStatus.DELIVERED;
        domainEvents.add(new OrderDeliveredEvent(getId().toString(), orderNumber));
    }

    public void cancel(String reason) {
        if (!canBeCancelled()) {
            throw new OrderCannotBeCancelledException(
                "Order " + orderNumber + " cannot be cancelled in status: " + status
            );
        }
        this.status = OrderStatus.CANCELLED;
        domainEvents.add(OrderCancelledEvent.builder()
            .orderId(getId().toString())
            .orderNumber(orderNumber)
            .reason(reason)
            .build());
    }

    private void validateTransition(OrderStatus newStatus) {
        if (!status.canTransitionTo(newStatus)) {
            throw new InvalidOrderStatusTransitionException(status, newStatus);
        }
    }

    private boolean canBeCancelled() {
        return status == OrderStatus.PENDING || status == OrderStatus.CONFIRMED;
    }

    private void recalculateTotals() {
        this.subtotal = items.stream()
            .map(OrderItem::getTotalPrice)
            .reduce(BigDecimal.ZERO, BigDecimal::add);

        this.totalAmount = subtotal
            .add(shippingCost)
            .add(taxAmount)
            .subtract(discountAmount);
    }

    private static String generateOrderNumber() {
        // Format: ORD-20240101-XXXXX
        return String.format("ORD-%s-%s",
            java.time.LocalDate.now().format(java.time.format.DateTimeFormatter.ofPattern("yyyyMMdd")),
            String.format("%05d", (int)(Math.random() * 99999))
        );
    }

    public List<DomainEvent> pullDomainEvents() {
        List<DomainEvent> events = new ArrayList<>(domainEvents);
        domainEvents.clear();
        return events;
    }
}
```

### Order Status State Machine

```java
// domain/model/OrderStatus.java
public enum OrderStatus {
    PENDING {
        @Override
        public Set<OrderStatus> allowedTransitions() {
            return Set.of(CONFIRMED, CANCELLED);
        }
    },
    CONFIRMED {
        @Override
        public Set<OrderStatus> allowedTransitions() {
            return Set.of(PROCESSING, CANCELLED);
        }
    },
    PROCESSING {
        @Override
        public Set<OrderStatus> allowedTransitions() {
            return Set.of(SHIPPED);
        }
    },
    SHIPPED {
        @Override
        public Set<OrderStatus> allowedTransitions() {
            return Set.of(DELIVERED);
        }
    },
    DELIVERED {
        @Override
        public Set<OrderStatus> allowedTransitions() {
            return Set.of(REFUNDED);  // Return/Refund only after delivery
        }
    },
    CANCELLED {
        @Override
        public Set<OrderStatus> allowedTransitions() {
            return Set.of();  // Terminal state
        }
    },
    REFUNDED {
        @Override
        public Set<OrderStatus> allowedTransitions() {
            return Set.of();  // Terminal state
        }
    };

    public abstract Set<OrderStatus> allowedTransitions();

    public boolean canTransitionTo(OrderStatus newStatus) {
        return allowedTransitions().contains(newStatus);
    }
}
```

---

## ขั้นตอนที่ 3483: Kafka Event Processing

### Event Publisher

```java
// infrastructure/messaging/OrderEventPublisher.java
package com.ecommerce.order.infrastructure.messaging;

import com.ecommerce.common.event.DomainEvent;
import com.fasterxml.jackson.databind.ObjectMapper;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.kafka.support.SendResult;
import org.springframework.stereotype.Component;

import java.util.concurrent.CompletableFuture;

@Slf4j
@Component
@RequiredArgsConstructor
public class OrderEventPublisher {

    private final KafkaTemplate<String, Object> kafkaTemplate;
    private final ObjectMapper objectMapper;

    // Topics
    private static final String ORDER_EVENTS_TOPIC = "order.events";
    private static final String ORDER_CREATED_TOPIC = "order.created";
    private static final String ORDER_CANCELLED_TOPIC = "order.cancelled";

    public void publish(DomainEvent event) {
        String topic = resolveTopicFor(event);
        String key = event.getAggregateId();

        log.info("Publishing event {} to topic {}", event.getEventType(), topic);

        CompletableFuture<SendResult<String, Object>> future =
            kafkaTemplate.send(topic, key, event);

        future.whenComplete((result, ex) -> {
            if (ex == null) {
                log.debug("Event {} published successfully to partition {}",
                    event.getEventId(),
                    result.getRecordMetadata().partition()
                );
            } else {
                log.error("Failed to publish event {}: {}",
                    event.getEventId(), ex.getMessage(), ex);
                // Store in Outbox for retry
                storeInOutbox(event);
            }
        });
    }

    private String resolveTopicFor(DomainEvent event) {
        return switch (event) {
            case OrderCreatedEvent e -> ORDER_CREATED_TOPIC;
            case OrderCancelledEvent e -> ORDER_CANCELLED_TOPIC;
            default -> ORDER_EVENTS_TOPIC;
        };
    }

    private void storeInOutbox(DomainEvent event) {
        // Implement Transactional Outbox Pattern
        // Save to outbox table for retry
    }
}
```

### Kafka Consumer (Inventory Service)

```java
// inventory-service/infrastructure/messaging/OrderEventConsumer.java
package com.ecommerce.inventory.infrastructure.messaging;

import com.ecommerce.common.event.OrderCreatedEvent;
import com.ecommerce.inventory.application.service.InventoryService;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.kafka.annotation.RetryableTopic;
import org.springframework.kafka.support.KafkaHeaders;
import org.springframework.messaging.handler.annotation.Header;
import org.springframework.messaging.handler.annotation.Payload;
import org.springframework.retry.annotation.Backoff;
import org.springframework.stereotype.Component;

@Slf4j
@Component
@RequiredArgsConstructor
public class OrderEventConsumer {

    private final InventoryService inventoryService;

    /**
     * Retry ด้วย Exponential Backoff
     * ถ้า Retry ครบ 3 ครั้งแล้วยังล้มเหลว ส่งไป Dead Letter Topic
     */
    @RetryableTopic(
        attempts = "3",
        backoff = @Backoff(delay = 1000, multiplier = 2.0),
        dltTopicSuffix = ".dlt"
    )
    @KafkaListener(
        topics = "order.created",
        groupId = "inventory-service",
        containerFactory = "orderKafkaListenerContainerFactory"
    )
    public void handleOrderCreated(
        @Payload OrderCreatedEvent event,
        @Header(KafkaHeaders.RECEIVED_PARTITION) int partition,
        @Header(KafkaHeaders.OFFSET) long offset
    ) {
        log.info("Processing OrderCreated event: orderId={}, partition={}, offset={}",
            event.getAggregateId(), partition, offset);

        try {
            inventoryService.reserveStock(event);
            log.info("Stock reserved for order: {}", event.getAggregateId());
        } catch (InsufficientStockException e) {
            log.warn("Insufficient stock for order: {}", event.getAggregateId());
            // Publish OrderInventoryFailedEvent
            inventoryService.handleInsufficientStock(event);
        }
    }

    @KafkaListener(
        topics = "order.cancelled",
        groupId = "inventory-service"
    )
    public void handleOrderCancelled(@Payload OrderCancelledEvent event) {
        log.info("Processing OrderCancelled event: orderId={}", event.getAggregateId());
        inventoryService.releaseReservedStock(event.getAggregateId());
    }

    // Dead Letter Topic Handler
    @KafkaListener(
        topics = "order.created.dlt",
        groupId = "inventory-service-dlt"
    )
    public void handleOrderCreatedDLT(@Payload OrderCreatedEvent event) {
        log.error("Order event in DLT, manual intervention needed: orderId={}",
            event.getAggregateId());
        // Alert Team, Create Incident
        alertService.createIncident("OrderCreated DLT", event);
    }
}
```

---

## ขั้นตอนที่ 3484: Real-time Inventory Management

```java
// inventory-service/domain/model/Inventory.java
@Entity
@Table(name = "inventory")
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class Inventory extends BaseEntity {

    @Column(name = "product_id", nullable = false)
    private UUID productId;

    @Column(name = "variant_id")
    private UUID variantId;

    @Column(name = "total_quantity", nullable = false)
    private int totalQuantity;

    @Column(name = "reserved_quantity", nullable = false)
    private int reservedQuantity = 0;

    @Column(name = "reorder_point")
    private int reorderPoint = 10;

    @Column(name = "reorder_quantity")
    private int reorderQuantity = 100;

    // Computed: เหลือขายได้จริง
    public int getAvailableQuantity() {
        return totalQuantity - reservedQuantity;
    }

    // Reserve Stock (สำหรับ Order ที่ Pending)
    public void reserve(int quantity) {
        if (getAvailableQuantity() < quantity) {
            throw new InsufficientStockException(
                String.format("Cannot reserve %d, only %d available",
                    quantity, getAvailableQuantity())
            );
        }
        this.reservedQuantity += quantity;
    }

    // Release Reserved (ถ้า Order ถูก Cancel)
    public void release(int quantity) {
        if (this.reservedQuantity < quantity) {
            throw new IllegalStateException("Cannot release more than reserved");
        }
        this.reservedQuantity -= quantity;
    }

    // Commit (เมื่อ Order ถูก Confirm จริงๆ)
    public void commit(int quantity) {
        if (this.reservedQuantity < quantity) {
            throw new IllegalStateException("Cannot commit more than reserved");
        }
        this.reservedQuantity -= quantity;
        this.totalQuantity -= quantity;
    }

    // Restock
    public void addStock(int quantity) {
        if (quantity <= 0) {
            throw new IllegalArgumentException("Quantity must be positive");
        }
        this.totalQuantity += quantity;
    }

    public boolean needsReorder() {
        return getAvailableQuantity() <= reorderPoint;
    }
}
```

```java
// application/service/InventoryService.java
@Slf4j
@Service
@RequiredArgsConstructor
public class InventoryService {

    private final InventoryRepository inventoryRepository;
    private final RedisTemplate<String, Object> redisTemplate;
    private final InventoryEventPublisher eventPublisher;

    /**
     * Reserve stock ด้วย Optimistic Locking
     * ป้องกัน Race Condition เมื่อมี Concurrent Orders
     */
    @Transactional
    @Retryable(
        value = OptimisticLockingFailureException.class,
        maxAttempts = 3,
        backoff = @Backoff(delay = 100, multiplier = 2)
    )
    public void reserveStock(OrderCreatedEvent event) {
        for (var item : event.getItems()) {
            Inventory inventory = inventoryRepository
                .findByProductIdAndVariantId(item.getProductId(), item.getVariantId())
                .orElseThrow(() -> new InventoryNotFoundException(item.getProductId()));

            inventory.reserve(item.getQuantity());
            inventoryRepository.save(inventory);

            // Update Redis Cache
            updateStockCache(inventory);

            // Check if reorder needed
            if (inventory.needsReorder()) {
                eventPublisher.publish(new LowStockEvent(
                    inventory.getProductId(),
                    inventory.getAvailableQuantity()
                ));
            }
        }

        log.info("Stock reserved for order: {}", event.getAggregateId());
    }

    private void updateStockCache(Inventory inventory) {
        String key = "stock:" + inventory.getProductId();
        redisTemplate.opsForValue().set(
            key,
            inventory.getAvailableQuantity(),
            Duration.ofMinutes(10)
        );
    }

    /**
     * Real-time stock check ผ่าน Redis
     * ลด Load บน Database
     */
    public int getAvailableStock(UUID productId) {
        String key = "stock:" + productId;
        Integer cached = (Integer) redisTemplate.opsForValue().get(key);

        if (cached != null) {
            return cached;
        }

        // Cache miss: ดึงจาก DB
        Inventory inventory = inventoryRepository
            .findByProductId(productId)
            .orElse(null);

        if (inventory == null) return 0;

        // Cache result
        redisTemplate.opsForValue().set(key, inventory.getAvailableQuantity(), Duration.ofMinutes(10));
        return inventory.getAvailableQuantity();
    }
}
```

---

## ขั้นตอนที่ 3485: Payment Processing with Idempotency

```java
// payment-service/application/service/PaymentService.java
@Slf4j
@Service
@RequiredArgsConstructor
public class PaymentService {

    private final PaymentRepository paymentRepository;
    private final StripePaymentGateway stripeGateway;
    private final PaymentEventPublisher eventPublisher;

    /**
     * Idempotent Payment Processing
     * ใช้ idempotencyKey เพื่อป้องกันการชำระเงินซ้ำ
     * 
     * ถ้า Request เดิมมาซ้ำ (Network Retry) จะ return ผลลัพธ์เดิม
     * แทนที่จะชำระเงินซ้ำ
     */
    @Transactional
    public PaymentResult processPayment(ProcessPaymentCommand command) {
        String idempotencyKey = command.getIdempotencyKey();

        // Check if already processed
        return paymentRepository
            .findByIdempotencyKey(idempotencyKey)
            .map(existingPayment -> {
                log.info("Duplicate payment request with key: {}, returning existing result",
                    idempotencyKey);
                return toPaymentResult(existingPayment);
            })
            .orElseGet(() -> createNewPayment(command));
    }

    private PaymentResult createNewPayment(ProcessPaymentCommand command) {
        // Create Payment Record
        Payment payment = Payment.create(
            command.getOrderId(),
            command.getUserId(),
            command.getAmount(),
            command.getCurrency(),
            command.getPaymentMethod(),
            command.getIdempotencyKey()
        );
        paymentRepository.save(payment);

        try {
            // Call Payment Gateway
            StripePaymentIntent intent = stripeGateway.createPaymentIntent(
                StripeRequest.builder()
                    .amount(command.getAmount())
                    .currency(command.getCurrency())
                    .paymentMethodId(command.getPaymentMethodId())
                    .idempotencyKey(command.getIdempotencyKey())
                    .metadata(Map.of(
                        "orderId", command.getOrderId().toString(),
                        "userId", command.getUserId().toString()
                    ))
                    .build()
            );

            // Update Payment
            payment.complete(intent.getId(), intent.getClientSecret());
            paymentRepository.save(payment);

            // Publish Success Event
            eventPublisher.publish(PaymentCompletedEvent.builder()
                .orderId(command.getOrderId())
                .paymentId(payment.getId())
                .amount(command.getAmount())
                .build());

            return PaymentResult.success(payment.getId(), intent.getClientSecret());

        } catch (StripeException e) {
            log.error("Payment failed for order {}: {}", command.getOrderId(), e.getMessage());

            payment.fail(e.getMessage());
            paymentRepository.save(payment);

            eventPublisher.publish(PaymentFailedEvent.builder()
                .orderId(command.getOrderId())
                .reason(e.getMessage())
                .build());

            return PaymentResult.failed(e.getMessage());
        }
    }
}
```

---

## ขั้นตอนที่ 3486: Search Service Implementation

```java
// search-service/infrastructure/elasticsearch/ProductSearchRepository.java
package com.ecommerce.search.infrastructure.elasticsearch;

import co.elastic.clients.elasticsearch.ElasticsearchClient;
import co.elastic.clients.elasticsearch._types.query_dsl.*;
import co.elastic.clients.elasticsearch.core.SearchRequest;
import co.elastic.clients.elasticsearch.core.SearchResponse;
import com.ecommerce.search.domain.model.ProductDocument;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Repository;

import java.util.List;

@Slf4j
@Repository
@RequiredArgsConstructor
public class ProductSearchRepository {

    private final ElasticsearchClient esClient;
    private static final String INDEX = "products";

    public SearchResult<ProductDocument> search(ProductSearchQuery query) throws Exception {
        SearchRequest.Builder requestBuilder = new SearchRequest.Builder()
            .index(INDEX);

        // Build Query
        BoolQuery.Builder boolQuery = QueryBuilders.bool();

        // Full-text search
        if (query.getKeyword() != null && !query.getKeyword().isEmpty()) {
            boolQuery.must(QueryBuilders.multiMatch(m -> m
                .query(query.getKeyword())
                .fields("name^3", "description^1", "brand^2", "tags^2")
                .fuzziness("AUTO")
                .type(TextQueryType.BestFields)
            ));
        }

        // Category filter
        if (query.getCategoryId() != null) {
            boolQuery.filter(QueryBuilders.term(t -> t
                .field("categoryId")
                .value(query.getCategoryId().toString())
            ));
        }

        // Price range filter
        if (query.getMinPrice() != null || query.getMaxPrice() != null) {
            RangeQuery.Builder rangeBuilder = QueryBuilders.range().field("price");
            if (query.getMinPrice() != null) rangeBuilder.gte(JsonData.of(query.getMinPrice()));
            if (query.getMaxPrice() != null) rangeBuilder.lte(JsonData.of(query.getMaxPrice()));
            boolQuery.filter(rangeBuilder.build()._toQuery());
        }

        // Status filter
        boolQuery.filter(QueryBuilders.term(t -> t
            .field("status")
            .value("ACTIVE")
        ));

        requestBuilder.query(boolQuery.build()._toQuery());

        // Aggregations (Facets)
        requestBuilder
            .aggregations("categories", a -> a
                .terms(t -> t.field("categoryId").size(20))
            )
            .aggregations("brands", a -> a
                .terms(t -> t.field("brand.keyword").size(20))
            )
            .aggregations("price_ranges", a -> a
                .range(r -> r
                    .field("price")
                    .ranges(
                        rv -> rv.to("500"),
                        rv -> rv.from("500").to("1000"),
                        rv -> rv.from("1000").to("5000"),
                        rv -> rv.from("5000")
                    )
                )
            );

        // Pagination
        requestBuilder
            .from(query.getPage() * query.getSize())
            .size(query.getSize());

        // Sorting
        if ("price_asc".equals(query.getSort())) {
            requestBuilder.sort(s -> s.field(f -> f.field("price").order(SortOrder.Asc)));
        } else if ("price_desc".equals(query.getSort())) {
            requestBuilder.sort(s -> s.field(f -> f.field("price").order(SortOrder.Desc)));
        } else {
            // Default: relevance + popularity
            requestBuilder.sort(s -> s.score(sc -> sc.order(SortOrder.Desc)));
        }

        SearchResponse<ProductDocument> response =
            esClient.search(requestBuilder.build(), ProductDocument.class);

        return SearchResult.<ProductDocument>builder()
            .hits(response.hits().hits().stream()
                .map(hit -> hit.source())
                .toList())
            .total(response.hits().total().value())
            .aggregations(parseAggregations(response.aggregations()))
            .build();
    }
}
```

---

## ขั้นตอนที่ 3487: Recommendation Engine

```java
// recommendation-service/domain/service/RecommendationService.java
@Slf4j
@Service
@RequiredArgsConstructor
public class RecommendationService {

    private final RedisTemplate<String, Object> redisTemplate;
    private final ProductViewRepository viewRepository;
    private final OrderItemRepository orderItemRepository;

    /**
     * Collaborative Filtering แบบง่าย
     * "ลูกค้าที่ซื้อสิ่งนี้ ยังซื้อ..."
     */
    public List<UUID> getCollaborativeRecommendations(
        UUID productId,
        int limit
    ) {
        String cacheKey = "rec:collab:" + productId;

        // Check Cache
        List<UUID> cached = (List<UUID>) redisTemplate.opsForValue().get(cacheKey);
        if (cached != null) return cached;

        // Find users who bought this product
        List<UUID> userIds = orderItemRepository.findUsersByProductId(productId);

        if (userIds.isEmpty()) return List.of();

        // Find what else those users bought (excluding current product)
        List<UUID> recommendations = orderItemRepository
            .findFrequentlyBoughtTogether(productId, userIds, limit);

        // Cache for 1 hour
        redisTemplate.opsForValue().set(cacheKey, recommendations, Duration.ofHours(1));

        return recommendations;
    }

    /**
     * Content-based Filtering
     * แนะนำสินค้าในหมวดหมู่เดียวกัน ที่ Rating ดี
     */
    public List<UUID> getContentBasedRecommendations(UUID productId, int limit) {
        return productRepository.findSimilarProducts(productId, limit);
    }

    /**
     * Personalized Recommendations
     * อิงจาก History ของ User นั้น
     */
    public List<UUID> getPersonalizedRecommendations(UUID userId, int limit) {
        String cacheKey = "rec:personal:" + userId;

        // Check Cache
        List<UUID> cached = (List<UUID>) redisTemplate.opsForValue().get(cacheKey);
        if (cached != null) return cached;

        // Get user's purchase history
        List<UUID> purchasedProducts = orderItemRepository
            .findProductIdsByUserId(userId, 50);

        // Get categories user likes
        List<UUID> preferredCategories = productRepository
            .findCategoriesByProductIds(purchasedProducts);

        // Get top products in those categories
        List<UUID> recommendations = productRepository
            .findTopProductsInCategories(preferredCategories, purchasedProducts, limit);

        // Cache for 30 minutes
        redisTemplate.opsForValue().set(cacheKey, recommendations, Duration.ofMinutes(30));

        return recommendations;
    }
}
```

---

## ขั้นตอนที่ 3488: API Gateway Configuration

```java
// api-gateway/src/main/resources/application.yml
spring:
  cloud:
    gateway:
      routes:
        # User Service Routes
        - id: user-service
          uri: lb://USER-SERVICE
          predicates:
            - Path=/api/v1/users/**,/api/v1/auth/**
          filters:
            - name: CircuitBreaker
              args:
                name: userServiceCB
                fallbackUri: forward:/fallback/user
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 100
                redis-rate-limiter.burstCapacity: 200
                redis-rate-limiter.requestedTokens: 1

        # Product Service Routes
        - id: product-service
          uri: lb://PRODUCT-SERVICE
          predicates:
            - Path=/api/v1/products/**,/api/v1/categories/**
          filters:
            - name: CircuitBreaker
              args:
                name: productServiceCB
                fallbackUri: forward:/fallback/product
            - AddRequestHeader=X-Service-Name, api-gateway
            - name: Retry
              args:
                retries: 3
                statuses: BAD_GATEWAY,SERVICE_UNAVAILABLE
                methods: GET
                backoff:
                  firstBackoff: 100ms
                  maxBackoff: 1000ms
                  factor: 2

        # Order Service Routes (Authentication Required)
        - id: order-service
          uri: lb://ORDER-SERVICE
          predicates:
            - Path=/api/v1/orders/**
          filters:
            - AuthenticationFilter
            - name: CircuitBreaker
              args:
                name: orderServiceCB
                fallbackUri: forward:/fallback/order

      # Global Filters
      default-filters:
        - name: RequestId
        - DedupeResponseHeader=Access-Control-Allow-Credentials Access-Control-Allow-Origin
        - name: GlobalRateLimiter

      globalcors:
        corsConfigurations:
          '[/**]':
            allowedOrigins:
              - "https://www.ecommerce.example.com"
              - "https://admin.ecommerce.example.com"
            allowedMethods:
              - GET
              - POST
              - PUT
              - PATCH
              - DELETE
              - OPTIONS
            allowedHeaders: "*"
            allowCredentials: true
            maxAge: 3600
```

---

## ขั้นตอนที่ 3489: Integration Testing

```java
// tests/OrderFlowIntegrationTest.java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Testcontainers
@ActiveProfiles("test")
class OrderFlowIntegrationTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16")
        .withDatabaseName("testdb")
        .withUsername("test")
        .withPassword("test");

    @Container
    static KafkaContainer kafka = new KafkaContainer(
        DockerImageName.parse("confluentinc/cp-kafka:7.5.0")
    );

    @Container
    static GenericContainer<?> redis = new GenericContainer<>("redis:7-alpine")
        .withExposedPorts(6379);

    @Autowired
    private TestRestTemplate restTemplate;

    @Autowired
    private KafkaTemplate<String, Object> kafkaTemplate;

    @Test
    @DisplayName("Complete Order Flow: Create -> Confirm -> Ship -> Deliver")
    void testCompleteOrderFlow() throws Exception {
        // Given: User and Products exist
        UUID userId = createTestUser();
        UUID productId = createTestProduct(100);  // 100 in stock

        // When: Create Order
        CreateOrderRequest orderRequest = CreateOrderRequest.builder()
            .userId(userId)
            .items(List.of(
                OrderItemRequest.builder()
                    .productId(productId)
                    .quantity(2)
                    .build()
            ))
            .shippingAddress(TestAddresses.BANGKOK)
            .idempotencyKey(UUID.randomUUID().toString())
            .build();

        ResponseEntity<ApiResponse<OrderResponse>> createResponse = restTemplate
            .postForEntity("/api/v1/orders", orderRequest, 
                new ParameterizedTypeReference<>() {});

        assertThat(createResponse.getStatusCode()).isEqualTo(HttpStatus.CREATED);
        UUID orderId = createResponse.getBody().getData().getId();

        // Then: Verify Stock Reserved
        Thread.sleep(1000);  // Wait for Kafka processing
        int availableStock = getAvailableStock(productId);
        assertThat(availableStock).isEqualTo(98);  // 100 - 2 reserved

        // Confirm Order
        restTemplate.postForEntity(
            "/api/v1/orders/" + orderId + "/confirm",
            null, Void.class
        );

        // Ship Order
        ShipOrderRequest shipRequest = ShipOrderRequest.builder()
            .trackingNumber("TH123456789")
            .build();
        restTemplate.postForEntity(
            "/api/v1/orders/" + orderId + "/ship",
            shipRequest, Void.class
        );

        // Verify Final State
        ResponseEntity<ApiResponse<OrderResponse>> getResponse = restTemplate
            .getForEntity("/api/v1/orders/" + orderId,
                new ParameterizedTypeReference<>() {});

        assertThat(getResponse.getBody().getData().getStatus())
            .isEqualTo(OrderStatus.SHIPPED);
        assertThat(getResponse.getBody().getData().getTrackingNumber())
            .isEqualTo("TH123456789");
    }
}
```

---

## สรุป Part 97

ในส่วนนี้เราได้ Implement:

1. **User Service** ด้วย DDD Pattern
2. **Order Domain** ด้วย State Machine
3. **Kafka Event Processing** สำหรับ Async Communication
4. **Inventory Management** ด้วย Optimistic Locking
5. **Payment Processing** ด้วย Idempotency
6. **Search Service** ด้วย Elasticsearch
7. **Recommendation Engine** ด้วย Redis Caching
8. **API Gateway** Configuration
9. **Integration Tests** ด้วย Testcontainers

---

*[← Part 96: Capstone Design](./part-96-capstone-design.md) | [Part 98: Interview Preparation →](./part-98-interview-prep.md)*

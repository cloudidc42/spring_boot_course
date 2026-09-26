# Part 39: Saga Pattern - Distributed Transactions
## ขั้นตอนที่ 1161-1200

> **ระดับ:** ระดับโลก (World-Class)  
> **เวลาเรียน:** 7-8 ชั่วโมง  
> **เป้าหมาย:** จัดการ distributed transactions ใน microservices

---

## ขั้นตอนที่ 1161: The Problem with Distributed Transactions

```
Traditional (Monolith):
  @Transactional
  public void createOrder() {
    orderRepo.save(order);    // Same DB
    inventoryRepo.reduce();   // Same DB
    paymentRepo.charge();     // Same DB
  }
  // If any fails → all rollback ✅

Microservices Problem:
  OrderService     → Order DB (PostgreSQL)
  InventoryService → Inventory DB (MySQL)
  PaymentService   → Payment DB (MongoDB)
  
  How to ensure all succeed or all fail?
  ❌ 2-Phase Commit (2PC) = slow, not scalable
  ✅ Saga Pattern = sequence of local transactions

Saga Approaches:
  1. Choreography  = Services react to events (no coordinator)
  2. Orchestration = Saga orchestrator controls flow
```

---

## ขั้นตอนที่ 1162: Choreography-Based Saga

```
Order Flow (Choreography):
  1. OrderService creates order → publishes OrderCreated
  2. InventoryService listens → reserves stock → publishes StockReserved (or StockFailed)
  3. PaymentService listens → charges payment → publishes PaymentProcessed (or PaymentFailed)
  4. OrderService listens → confirms order → publishes OrderConfirmed
  
Compensating Transactions (if failure):
  PaymentFailed → InventoryService releases stock
  StockFailed   → OrderService cancels order
```

```java
// OrderService - Choreography
@Service
@RequiredArgsConstructor
@Slf4j
public class OrderSagaService {
    
    private final OrderRepository orderRepository;
    private final KafkaTemplate<String, Object> kafkaTemplate;
    
    @Transactional
    public Order createOrder(CreateOrderRequest request) {
        Order order = Order.builder()
            .userId(request.userId())
            .status(OrderStatus.PENDING)
            .items(mapItems(request.items()))
            .build();
        
        Order saved = orderRepository.save(order);
        
        // Publish event to start saga
        kafkaTemplate.send("orders.created", String.valueOf(saved.getId()),
            new OrderCreatedEvent(saved.getId(), request.userId(), request.items()));
        
        return saved;
    }
    
    // Listen for payment confirmation
    @KafkaListener(topics = "payments.processed")
    @Transactional
    public void onPaymentProcessed(PaymentProcessedEvent event, Acknowledgment ack) {
        orderRepository.findById(event.getOrderId()).ifPresent(order -> {
            order.setStatus(OrderStatus.CONFIRMED);
            orderRepository.save(order);
            
            log.info("Order {} confirmed after payment", event.getOrderId());
        });
        ack.acknowledge();
    }
    
    // Compensating transaction - payment failed
    @KafkaListener(topics = "payments.failed")
    @Transactional
    public void onPaymentFailed(PaymentFailedEvent event, Acknowledgment ack) {
        orderRepository.findById(event.getOrderId()).ifPresent(order -> {
            order.setStatus(OrderStatus.CANCELLED);
            order.setCancellationReason("Payment failed: " + event.getReason());
            orderRepository.save(order);
            
            // Publish to release inventory
            kafkaTemplate.send("orders.cancelled", String.valueOf(order.getId()),
                new OrderCancelledEvent(order.getId(), "Payment failed"));
            
            log.error("Order {} cancelled: payment failed", event.getOrderId());
        });
        ack.acknowledge();
    }
}

// InventoryService - listens and responds
@Component
@RequiredArgsConstructor
@Slf4j
public class InventoryEventHandler {
    
    private final InventoryService inventoryService;
    private final KafkaTemplate<String, Object> kafkaTemplate;
    
    @KafkaListener(topics = "orders.created")
    @Transactional
    public void onOrderCreated(OrderCreatedEvent event, Acknowledgment ack) {
        try {
            inventoryService.reserveStock(event.getItems());
            
            kafkaTemplate.send("inventory.reserved", String.valueOf(event.getOrderId()),
                new StockReservedEvent(event.getOrderId()));
            
        } catch (InsufficientStockException ex) {
            kafkaTemplate.send("inventory.failed", String.valueOf(event.getOrderId()),
                new StockFailedEvent(event.getOrderId(), ex.getMessage()));
        }
        ack.acknowledge();
    }
    
    // Compensating: release reserved stock if order cancelled
    @KafkaListener(topics = "orders.cancelled")
    @Transactional
    public void onOrderCancelled(OrderCancelledEvent event, Acknowledgment ack) {
        inventoryService.releaseReservation(event.getOrderId());
        ack.acknowledge();
    }
}
```

---

## ขั้นตอนที่ 1163: Orchestration-Based Saga

```java
// Saga Orchestrator - controls the entire flow
@Component
@RequiredArgsConstructor
@Slf4j
public class CreateOrderSagaOrchestrator {
    
    private final SagaStateRepository sagaStateRepository;
    private final KafkaTemplate<String, Object> kafkaTemplate;
    
    // Start the saga
    @Transactional
    public void start(String sagaId, CreateOrderRequest request) {
        SagaState state = SagaState.builder()
            .sagaId(sagaId)
            .sagaType("CREATE_ORDER")
            .currentStep("RESERVE_INVENTORY")
            .status(SagaStatus.IN_PROGRESS)
            .payload(serialize(request))
            .build();
        
        sagaStateRepository.save(state);
        
        // Step 1: Reserve inventory
        kafkaTemplate.send("inventory.reserve", sagaId,
            new ReserveInventoryCommand(sagaId, request.items()));
    }
    
    // Handle inventory reserved → move to payment
    @KafkaListener(topics = "inventory.reserved.reply")
    @Transactional
    public void onInventoryReserved(StockReservedEvent event, Acknowledgment ack) {
        SagaState state = sagaStateRepository.findBySagaId(event.getSagaId())
            .orElseThrow(() -> new IllegalStateException("Saga not found: " + event.getSagaId()));
        
        state.setCurrentStep("PROCESS_PAYMENT");
        sagaStateRepository.save(state);
        
        CreateOrderRequest request = deserialize(state.getPayload(), CreateOrderRequest.class);
        
        // Step 2: Process payment
        kafkaTemplate.send("payment.process", event.getSagaId(),
            new ProcessPaymentCommand(event.getSagaId(), request.userId(), request.totalAmount()));
        
        ack.acknowledge();
    }
    
    // Handle payment processed → confirm order
    @KafkaListener(topics = "payment.processed.reply")
    @Transactional
    public void onPaymentProcessed(PaymentProcessedEvent event, Acknowledgment ack) {
        SagaState state = sagaStateRepository.findBySagaId(event.getSagaId()).orElseThrow();
        
        state.setCurrentStep("CONFIRM_ORDER");
        sagaStateRepository.save(state);
        
        // Step 3: Confirm order
        kafkaTemplate.send("order.confirm", event.getSagaId(),
            new ConfirmOrderCommand(event.getSagaId(), event.getOrderId()));
        
        ack.acknowledge();
    }
    
    // Handle order confirmed → saga complete
    @KafkaListener(topics = "order.confirmed.reply")
    @Transactional
    public void onOrderConfirmed(OrderConfirmedEvent event, Acknowledgment ack) {
        SagaState state = sagaStateRepository.findBySagaId(event.getSagaId()).orElseThrow();
        
        state.setStatus(SagaStatus.COMPLETED);
        state.setCurrentStep("DONE");
        state.setCompletedAt(LocalDateTime.now());
        sagaStateRepository.save(state);
        
        log.info("Saga {} completed successfully", event.getSagaId());
        ack.acknowledge();
    }
    
    // ===== Compensating Transactions =====
    
    @KafkaListener(topics = "inventory.failed.reply")
    @Transactional
    public void onInventoryFailed(StockFailedEvent event, Acknowledgment ack) {
        compensate(event.getSagaId(), "INVENTORY_FAILED", event.getReason());
        ack.acknowledge();
    }
    
    @KafkaListener(topics = "payment.failed.reply")
    @Transactional
    public void onPaymentFailed(PaymentFailedEvent event, Acknowledgment ack) {
        // Compensate: release inventory
        kafkaTemplate.send("inventory.release", event.getSagaId(),
            new ReleaseInventoryCommand(event.getSagaId()));
        
        compensate(event.getSagaId(), "PAYMENT_FAILED", event.getReason());
        ack.acknowledge();
    }
    
    private void compensate(String sagaId, String failureReason, String details) {
        sagaStateRepository.findBySagaId(sagaId).ifPresent(state -> {
            state.setStatus(SagaStatus.FAILED);
            state.setFailureReason(failureReason + ": " + details);
            state.setCompletedAt(LocalDateTime.now());
            sagaStateRepository.save(state);
            
            log.error("Saga {} failed at {}: {}", sagaId, state.getCurrentStep(), details);
        });
    }
}
```

---

## ขั้นตอนที่ 1164: Saga State Storage

```java
@Entity
@Table(name = "saga_states")
@Getter @Setter @Builder @NoArgsConstructor @AllArgsConstructor
public class SagaState {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(name = "saga_id", unique = true, nullable = false)
    private String sagaId;
    
    @Column(name = "saga_type", nullable = false)
    private String sagaType;
    
    @Column(name = "current_step")
    private String currentStep;
    
    @Enumerated(EnumType.STRING)
    private SagaStatus status;
    
    @Column(columnDefinition = "jsonb")
    @JdbcTypeCode(SqlTypes.JSON)
    private String payload;
    
    @Column(name = "failure_reason")
    private String failureReason;
    
    @CreationTimestamp
    @Column(name = "created_at")
    private LocalDateTime createdAt;
    
    @Column(name = "completed_at")
    private LocalDateTime completedAt;
}

public enum SagaStatus {
    IN_PROGRESS, COMPLETED, FAILED, COMPENSATING
}
```

---

## ขั้นตอนที่ 1165-1200: Idempotency

```java
// Make handlers idempotent - safe to process same message twice
@Component
@RequiredArgsConstructor
public class IdempotencyFilter {
    
    private final ProcessedMessageRepository processedRepo;
    
    public boolean isAlreadyProcessed(String messageId) {
        return processedRepo.existsByMessageId(messageId);
    }
    
    @Transactional
    public void markAsProcessed(String messageId) {
        processedRepo.save(ProcessedMessage.builder()
            .messageId(messageId)
            .processedAt(LocalDateTime.now())
            .build());
    }
}

// Use in listener
@KafkaListener(topics = "payments.processed")
@Transactional
public void onPaymentProcessed(
    @Payload PaymentProcessedEvent event,
    @Header(KafkaHeaders.RECEIVED_KEY) String messageId,
    Acknowledgment ack
) {
    if (idempotencyFilter.isAlreadyProcessed(messageId)) {
        log.info("Skipping duplicate message: {}", messageId);
        ack.acknowledge();
        return;
    }
    
    // Process...
    
    idempotencyFilter.markAsProcessed(messageId);
    ack.acknowledge();
}

/*
 * Saga Pattern Summary:
 * 
 * Choreography:
 *   ✅ Simple, decoupled
 *   ❌ Hard to track overall saga state
 *   ❌ Hard to debug
 *   Best for: Simple 2-3 step flows
 * 
 * Orchestration:
 *   ✅ Clear flow visible in one place
 *   ✅ Easier to debug
 *   ✅ Better monitoring
 *   ❌ Orchestrator is a coupling point
 *   Best for: Complex multi-step flows
 * 
 * Key principles:
 *   1. Compensating transactions for rollback
 *   2. Idempotent handlers
 *   3. Track saga state persistently
 *   4. Handle partial failures gracefully
 */
```

---

*[← Part 38: Event Sourcing](./part-38-event-sourcing.md) | [Part 40: GraphQL →](./part-40-graphql.md)*

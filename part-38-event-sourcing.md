# Part 38: Event Sourcing
## ขั้นตอนที่ 1126-1160

> **ระดับ:** ระดับโลก (World-Class)  
> **เวลาเรียน:** 7-8 ชั่วโมง  
> **เป้าหมาย:** Store state as sequence of events

---

## ขั้นตอนที่ 1126: Event Sourcing Concepts

```
Traditional (State-based):
  - Store current state: {id:1, status:"SHIPPED", amount:500}
  - ❌ No history of how we got here

Event Sourcing:
  - Store events: [OrderCreated, PaymentProcessed, OrderShipped]
  - Replay events to get current state
  
Benefits:
  ✅ Full audit trail
  ✅ Time travel (replay to any point in time)
  ✅ Event-driven by nature
  ✅ No update/delete → append-only
  ✅ Debug complex state changes

Trade-offs:
  ❌ Complex to implement
  ❌ Event schema evolution
  ❌ Eventual consistency for queries (need CQRS)
  ❌ Event store can grow large (use snapshots)

Event Store:
  - Append-only storage for events
  - Key: aggregateId + version
  - Support EventStoreDB, Axon Framework, custom
```

---

## ขั้นตอนที่ 1127: Event Store Implementation

```java
// Domain Event base
@MappedSuperclass
@Getter
public abstract class DomainEvent {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(name = "aggregate_id", nullable = false)
    private String aggregateId;
    
    @Column(name = "aggregate_type", nullable = false)
    private String aggregateType;
    
    @Column(name = "event_type", nullable = false)
    private String eventType;
    
    @Column(nullable = false)
    private Long version;
    
    @Column(columnDefinition = "jsonb", nullable = false)
    @JdbcTypeCode(SqlTypes.JSON)
    private String payload;
    
    @Column(name = "occurred_at", nullable = false)
    private LocalDateTime occurredAt = LocalDateTime.now();
    
    @Column(name = "correlation_id")
    private String correlationId;
    
    public abstract Object getPayloadObject();
}

// Concrete events
@Entity
@Table(name = "order_events")
@Getter @Setter @Builder @NoArgsConstructor @AllArgsConstructor
public class OrderEvent extends DomainEvent {
    
    @Override
    @Transient
    public Object getPayloadObject() {
        try {
            return objectMapper.readValue(getPayload(), Object.class);
        } catch (JsonProcessingException e) {
            throw new RuntimeException(e);
        }
    }
}

// Event Store Repository
@Repository
public interface EventStoreRepository extends JpaRepository<OrderEvent, Long> {
    
    List<OrderEvent> findByAggregateIdOrderByVersionAsc(String aggregateId);
    
    List<OrderEvent> findByAggregateIdAndVersionGreaterThanOrderByVersionAsc(
        String aggregateId, Long afterVersion
    );
    
    Optional<OrderEvent> findTopByAggregateIdOrderByVersionDesc(String aggregateId);
}

// Event Store Service
@Service
@RequiredArgsConstructor
@Slf4j
public class EventStore {
    
    private final EventStoreRepository repository;
    private final ObjectMapper objectMapper;
    private final ApplicationEventPublisher eventPublisher;
    
    @Transactional
    public void append(String aggregateId, String aggregateType, 
                       String eventType, Object payload, Long expectedVersion) {
        
        // Optimistic concurrency check
        Optional<OrderEvent> lastEvent = repository.findTopByAggregateIdOrderByVersionDesc(aggregateId);
        long currentVersion = lastEvent.map(e -> e.getVersion()).orElse(0L);
        
        if (expectedVersion != null && !expectedVersion.equals(currentVersion)) {
            throw new ConcurrencyException(
                "Expected version " + expectedVersion + " but current is " + currentVersion);
        }
        
        OrderEvent event = OrderEvent.builder()
            .aggregateId(aggregateId)
            .aggregateType(aggregateType)
            .eventType(eventType)
            .version(currentVersion + 1)
            .payload(serialize(payload))
            .occurredAt(LocalDateTime.now())
            .build();
        
        repository.save(event);
        
        // Publish for read model updates
        eventPublisher.publishEvent(event);
        
        log.debug("Appended event {} v{} to aggregate {}", eventType, event.getVersion(), aggregateId);
    }
    
    @Transactional(readOnly = true)
    public List<OrderEvent> getEvents(String aggregateId) {
        return repository.findByAggregateIdOrderByVersionAsc(aggregateId);
    }
    
    @Transactional(readOnly = true)
    public List<OrderEvent> getEventsSince(String aggregateId, Long sinceVersion) {
        return repository.findByAggregateIdAndVersionGreaterThanOrderByVersionAsc(
            aggregateId, sinceVersion);
    }
    
    private String serialize(Object obj) {
        try {
            return objectMapper.writeValueAsString(obj);
        } catch (JsonProcessingException e) {
            throw new RuntimeException("Failed to serialize event payload", e);
        }
    }
}
```

---

## ขั้นตอนที่ 1128: Event-Sourced Aggregate

```java
// Order event payloads
public record OrderCreatedPayload(Long userId, String orderNumber, Address shippingAddress) {}
public record OrderItemAddedPayload(Long productId, String name, BigDecimal price, int quantity) {}
public record OrderConfirmedPayload(Money totalAmount) {}
public record OrderShippedPayload(String trackingNumber) {}
public record OrderCancelledPayload(String reason) {}

// Event-Sourced Order Aggregate
public class Order {
    
    private Long id;
    private String aggregateId;
    private Long version = 0L;
    
    // Current state (derived from events)
    private String orderNumber;
    private Long userId;
    private OrderStatus status;
    private List<OrderItem> items = new ArrayList<>();
    private Money totalAmount;
    private Address shippingAddress;
    
    // Uncommitted events (will be saved)
    private final List<Object> uncommittedEvents = new ArrayList<>();
    
    // Private constructor - use factory methods
    private Order() {}
    
    // ===== Factory: Reconstruct from events =====
    public static Order fromEvents(List<OrderEvent> events) {
        Order order = new Order();
        events.forEach(order::apply);
        return order;
    }
    
    // ===== Commands (produce events) =====
    
    public static Order create(Long userId, Address shippingAddress) {
        Order order = new Order();
        order.aggregateId = UUID.randomUUID().toString();
        order.raiseEvent("ORDER_CREATED", 
            new OrderCreatedPayload(userId, generateOrderNumber(), shippingAddress));
        return order;
    }
    
    public void addItem(Long productId, String name, Money unitPrice, int quantity) {
        if (status != OrderStatus.PENDING) {
            throw new DomainException("Cannot add items to " + status + " order");
        }
        raiseEvent("ORDER_ITEM_ADDED", new OrderItemAddedPayload(productId, name, unitPrice.amount(), quantity));
    }
    
    public void confirm() {
        if (status != OrderStatus.PENDING || items.isEmpty()) {
            throw new DomainException("Cannot confirm: status=" + status + ", items=" + items.size());
        }
        raiseEvent("ORDER_CONFIRMED", new OrderConfirmedPayload(totalAmount));
    }
    
    public void ship(String trackingNumber) {
        if (status != OrderStatus.CONFIRMED) {
            throw new DomainException("Cannot ship: status=" + status);
        }
        raiseEvent("ORDER_SHIPPED", new OrderShippedPayload(trackingNumber));
    }
    
    public void cancel(String reason) {
        if (status == OrderStatus.DELIVERED || status == OrderStatus.CANCELLED) {
            throw new DomainException("Cannot cancel: status=" + status);
        }
        raiseEvent("ORDER_CANCELLED", new OrderCancelledPayload(reason));
    }
    
    // ===== Event application (state transitions) =====
    
    private void apply(OrderEvent event) {
        version = event.getVersion();
        
        switch (event.getEventType()) {
            case "ORDER_CREATED" -> {
                OrderCreatedPayload payload = deserialize(event.getPayload(), OrderCreatedPayload.class);
                this.aggregateId = event.getAggregateId();
                this.userId = payload.userId();
                this.orderNumber = payload.orderNumber();
                this.shippingAddress = payload.shippingAddress();
                this.status = OrderStatus.PENDING;
                this.totalAmount = Money.thb(BigDecimal.ZERO);
            }
            case "ORDER_ITEM_ADDED" -> {
                OrderItemAddedPayload payload = deserialize(event.getPayload(), OrderItemAddedPayload.class);
                items.add(new OrderItem(payload.productId(), payload.name(), 
                    Money.thb(payload.price()), payload.quantity()));
                recalculateTotal();
            }
            case "ORDER_CONFIRMED" -> this.status = OrderStatus.CONFIRMED;
            case "ORDER_SHIPPED" -> this.status = OrderStatus.SHIPPED;
            case "ORDER_CANCELLED" -> this.status = OrderStatus.CANCELLED;
        }
    }
    
    private void raiseEvent(String type, Object payload) {
        uncommittedEvents.add(new EventEnvelope(aggregateId, "ORDER", type, payload, version));
        // Apply to self immediately
        // (simplified - in real implementation create an OrderEvent and apply it)
    }
    
    public List<Object> getUncommittedEvents() {
        return Collections.unmodifiableList(uncommittedEvents);
    }
    
    public void markEventsAsCommitted() {
        uncommittedEvents.clear();
    }
    
    private void recalculateTotal() {
        totalAmount = items.stream()
            .map(i -> i.getUnitPrice().multiply(i.getQuantity()))
            .reduce(Money.thb(BigDecimal.ZERO), Money::add);
    }
    
    private static String generateOrderNumber() {
        return "ORD-" + LocalDate.now().format(DateTimeFormatter.ofPattern("yyyyMMdd"))
            + "-" + UUID.randomUUID().toString().substring(0, 8).toUpperCase();
    }
}
```

---

## ขั้นตอนที่ 1129: Snapshot Pattern

```java
// For long-lived aggregates, save snapshot to avoid replaying all events
@Entity
@Table(name = "order_snapshots")
@Getter @Setter @Builder @NoArgsConstructor @AllArgsConstructor
public class OrderSnapshot {
    
    @Id
    @GeneratedValue
    private Long id;
    
    @Column(name = "aggregate_id", nullable = false)
    private String aggregateId;
    
    @Column(nullable = false)
    private Long version;
    
    @Column(columnDefinition = "jsonb", nullable = false)
    @JdbcTypeCode(SqlTypes.JSON)
    private String state;  // Serialized Order state
    
    @Column(name = "created_at")
    private LocalDateTime createdAt = LocalDateTime.now();
}

@Service
@RequiredArgsConstructor
public class OrderRepository {
    
    private final EventStore eventStore;
    private final OrderSnapshotRepository snapshotRepository;
    private static final int SNAPSHOT_THRESHOLD = 100;
    
    public Optional<Order> findById(String aggregateId) {
        // 1. Load latest snapshot
        Optional<OrderSnapshot> snapshot = snapshotRepository
            .findTopByAggregateIdOrderByVersionDesc(aggregateId);
        
        if (snapshot.isPresent()) {
            Order order = deserialize(snapshot.get().getState(), Order.class);
            
            // 2. Load events after snapshot
            List<OrderEvent> events = eventStore.getEventsSince(aggregateId, snapshot.get().getVersion());
            events.forEach(e -> order.applyFromHistory(e));
            
            return Optional.of(order);
        } else {
            // No snapshot - load all events
            List<OrderEvent> events = eventStore.getEvents(aggregateId);
            if (events.isEmpty()) return Optional.empty();
            
            return Optional.of(Order.fromEvents(events));
        }
    }
    
    public void save(Order order) {
        // Save uncommitted events
        for (Object event : order.getUncommittedEvents()) {
            EventEnvelope env = (EventEnvelope) event;
            eventStore.append(env.aggregateId(), env.aggregateType(), 
                env.eventType(), env.payload(), env.expectedVersion());
        }
        order.markEventsAsCommitted();
        
        // Save snapshot if threshold reached
        if (order.getVersion() % SNAPSHOT_THRESHOLD == 0) {
            snapshotRepository.save(OrderSnapshot.builder()
                .aggregateId(order.getAggregateId())
                .version(order.getVersion())
                .state(serialize(order))
                .build());
        }
    }
}
```

---

## ขั้นตอนที่ 1130-1160: Projections

```java
// Read model projection - built from events
@Component
@RequiredArgsConstructor
@Slf4j
public class OrderProjection {
    
    private final OrderViewRepository viewRepository;
    private final ObjectMapper objectMapper;
    
    @TransactionalEventListener
    public void on(OrderEvent event) {
        switch (event.getEventType()) {
            case "ORDER_CREATED" -> {
                OrderCreatedPayload p = deserialize(event, OrderCreatedPayload.class);
                viewRepository.save(OrderView.builder()
                    .id(event.getAggregateId())
                    .userId(p.userId())
                    .orderNumber(p.orderNumber())
                    .status("PENDING")
                    .totalAmount(BigDecimal.ZERO)
                    .createdAt(event.getOccurredAt())
                    .build());
            }
            case "ORDER_CONFIRMED" -> {
                viewRepository.findById(event.getAggregateId())
                    .ifPresent(view -> {
                        view.setStatus("CONFIRMED");
                        viewRepository.save(view);
                    });
            }
            case "ORDER_SHIPPED" -> {
                OrderShippedPayload p = deserialize(event, OrderShippedPayload.class);
                viewRepository.findById(event.getAggregateId())
                    .ifPresent(view -> {
                        view.setStatus("SHIPPED");
                        view.setTrackingNumber(p.trackingNumber());
                        view.setShippedAt(event.getOccurredAt());
                        viewRepository.save(view);
                    });
            }
        }
    }
    
    // Rebuild projection from scratch
    @Transactional
    public void rebuildAll() {
        log.info("Rebuilding all order projections...");
        viewRepository.deleteAll();
        
        eventStore.getAll().forEach(this::on);
        
        log.info("Projection rebuild complete");
    }
}
```

---

*[← Part 37: OpenTelemetry](./part-37-opentelemetry.md) | [Part 39: Saga Pattern →](./part-39-saga.md)*

# Part 65: Data Consistency in Distributed Systems
## ขั้นตอนที่ 2201-2240

**ระดับ:** ระดับโลก (World-Class)
**เวลาเรียน:** 5-6 ชั่วโมง
**เป้าหมาย:** เข้าใจและ Implement Data Consistency Patterns สำหรับระบบ Distributed ระดับ Production

---

## บทนำ

ในระบบ Distributed System การรักษา Data Consistency เป็นความท้าทายที่ยิ่งใหญ่ที่สุด เราต้องเลือกระหว่าง Consistency, Availability และ Partition Tolerance (CAP Theorem)

---

## ขั้นตอนที่ 2201-2205: CAP Theorem

### อธิบาย

CAP Theorem บอกว่าระบบ Distributed สามารถรับประกันได้แค่ 2 ใน 3 คุณสมบัติ:

- **Consistency (C):** ทุก Node เห็นข้อมูลเหมือนกันทุกเวลา
- **Availability (A):** ทุก Request ได้รับ Response (ไม่ใช่ Error)
- **Partition Tolerance (P):** ระบบทำงานต่อได้แม้ Network แบ่งเป็นส่วนๆ

```
           Consistency
               |
               |
    CA         |         CP
  (SQL DB)     |      (HBase, Zookeeper)
               |
    -----------+----------
               |
    AP         |
  (Cassandra,  |
   DynamoDB)   |
               |
         Partition Tolerance (Always needed in production)
```

### PACELC Extension

PACELC เป็นการขยาย CAP: "เมื่อมี Partition เลือก Availability หรือ Consistency แต่เมื่อ Normal เลือก Latency หรือ Consistency"

```java
// DatabaseChoiceGuide.java - แสดงแนวทางการเลือก Database
public class DatabaseChoiceGuide {
    
    /*
     * CA Systems (เลือก Consistency + Availability, ไม่ Partition Tolerant):
     * - MySQL, PostgreSQL, Oracle
     * - ดีสำหรับ: Traditional OLTP, Financial Systems (Single Data Center)
     * - ข้อจำกัด: ไม่สามารถ Scale แบบ Distributed ได้ดี
     *
     * CP Systems (เลือก Consistency + Partition Tolerance):
     * - Apache Zookeeper, HBase, MongoDB (strong consistency mode)
     * - ดีสำหรับ: Distributed Coordination, Leader Election
     * - ข้อจำกัด: อาจไม่ Available เมื่อมี Partition
     *
     * AP Systems (เลือก Availability + Partition Tolerance):
     * - Apache Cassandra, DynamoDB, CouchDB
     * - ดีสำหรับ: High-scale, Eventually Consistent Use Cases
     * - ข้อจำกัด: Eventual Consistency ต้องจัดการ Conflicts
     */
}
```

---

## ขั้นตอนที่ 2206-2210: Eventual Consistency Patterns

### อธิบาย

Eventual Consistency หมายความว่าข้อมูลจะ Consistent ในที่สุด แต่อาจมีช่วงเวลาที่ Nodes เห็นข้อมูลต่างกัน

```java
// EventuallyConsistentOrderService.java
@Service
@RequiredArgsConstructor
@Slf4j
public class EventuallyConsistentOrderService {
    
    private final OrderRepository orderRepository;
    private final ApplicationEventPublisher eventPublisher;
    
    @Transactional
    public Order createOrder(CreateOrderRequest request) {
        // 1. บันทึก Order ลง Local Database (Immediately Consistent)
        Order order = Order.builder()
            .id(UUID.randomUUID().toString())
            .customerId(request.getCustomerId())
            .status(OrderStatus.PENDING)
            .build();
        
        Order saved = orderRepository.save(order);
        
        // 2. Publish Event (จะถูก Propagate ไปยัง Services อื่น Eventually)
        OrderCreatedEvent event = new OrderCreatedEvent(
            saved.getId(),
            saved.getCustomerId(),
            request.getItems(),
            Instant.now()
        );
        
        eventPublisher.publishEvent(event);
        
        // หมายเหตุ: ณ จุดนี้ Inventory Service ยังไม่รู้เรื่อง Order นี้
        // แต่ในที่สุดจะรับ Event และ Update ตัวเอง
        
        return saved;
    }
    
    // Read your own writes: อ่านจาก Cache ที่มีข้อมูลล่าสุด
    public Optional<Order> getOrderWithConsistency(String orderId) {
        // ลอง Read จาก Primary ก่อน (Strongly Consistent)
        return orderRepository.findById(orderId);
        
        // Note: ใน HA Setup อาจต้อง Read จาก Primary Node โดยเฉพาะ
        // เพื่อให้ Read your own writes ทำงานได้
    }
}
```

```java
// ReadRepairService.java - Read Repair Pattern
@Service
@RequiredArgsConstructor
@Slf4j
public class ReadRepairService {
    
    private final List<DataReplica> replicas;
    private final DataRepairQueue repairQueue;
    
    // อ่านจากหลาย Replicas และ Repair ตัวที่ Stale
    public Optional<Order> readWithRepair(String orderId) {
        List<CompletableFuture<Optional<Order>>> futures = replicas.stream()
            .map(replica -> CompletableFuture.supplyAsync(
                () -> replica.findById(orderId)))
            .collect(Collectors.toList());
        
        // รอทุก Replica ตอบกลับ
        List<Optional<Order>> results = futures.stream()
            .map(CompletableFuture::join)
            .collect(Collectors.toList());
        
        // หา Version ล่าสุด
        Optional<Order> latest = results.stream()
            .filter(Optional::isPresent)
            .map(Optional::get)
            .max(Comparator.comparing(Order::getVersion));
        
        if (latest.isEmpty()) {
            return Optional.empty();
        }
        
        Order latestOrder = latest.get();
        
        // Repair Replicas ที่ Stale
        for (int i = 0; i < replicas.size(); i++) {
            Optional<Order> replicaResult = results.get(i);
            if (replicaResult.isEmpty() || 
                replicaResult.get().getVersion() < latestOrder.getVersion()) {
                
                final DataReplica staleReplica = replicas.get(i);
                log.info("Scheduling read repair for replica: {}", i);
                
                // Async Repair (ไม่รอ)
                repairQueue.schedule(() -> 
                    staleReplica.update(latestOrder));
            }
        }
        
        return latest;
    }
}
```

---

## ขั้นตอนที่ 2211-2215: Two-Phase Commit (2PC) และปัญหา

### อธิบาย

2PC เป็น Protocol สำหรับ Distributed Transaction:
1. **Phase 1 (Prepare):** Coordinator ถาม Participants ทุกตัวว่า "พร้อม Commit ไหม?"
2. **Phase 2 (Commit/Abort):** ถ้าทุกตัวพร้อม → Commit, ถ้ามีตัวใดปฏิเสธ → Abort

### ปัญหาของ 2PC

```
ปัญหา 1: Blocking Protocol
- ถ้า Coordinator ล้มเหลวหลัง Phase 1 → Participants ค้าง

ปัญหา 2: Single Point of Failure
- Coordinator คือ SPOF

ปัญหา 3: Performance
- ต้องรอทุก Participant ตอบ → Latency สูง

ปัญหา 4: Not Suitable for Microservices
- ต้องการ Distributed Transaction Manager ที่ Services ทั้งหมด Support
```

```java
// TwoPhaseCommitDemo.java - แสดง 2PC (ไม่แนะนำสำหรับ Microservices)
@Service
@RequiredArgsConstructor
@Slf4j
public class TwoPhaseCommitCoordinator {
    
    private final List<TransactionParticipant> participants;
    
    public boolean executeDistributedTransaction(TransactionRequest request) {
        String transactionId = UUID.randomUUID().toString();
        
        // Phase 1: Prepare
        log.info("2PC Phase 1: Preparing transaction {}", transactionId);
        List<PrepareResult> prepareResults = new ArrayList<>();
        
        for (TransactionParticipant participant : participants) {
            try {
                PrepareResult result = participant.prepare(transactionId, request);
                prepareResults.add(result);
                
                if (!result.isReady()) {
                    log.warn("Participant {} not ready, aborting", 
                             participant.getName());
                    // Abort ทุก Participant ที่ Prepare แล้ว
                    abortAll(transactionId, prepareResults);
                    return false;
                }
            } catch (Exception e) {
                log.error("Participant {} failed in prepare phase", 
                          participant.getName());
                abortAll(transactionId, prepareResults);
                return false;
            }
        }
        
        // Phase 2: Commit
        log.info("2PC Phase 2: Committing transaction {}", transactionId);
        boolean allCommitted = true;
        
        for (TransactionParticipant participant : participants) {
            try {
                participant.commit(transactionId);
            } catch (Exception e) {
                // ปัญหาใหญ่: บาง Participant Commit แล้ว บางตัวยังไม่ได้
                log.error("CRITICAL: Participant {} failed to commit transaction {}", 
                          participant.getName(), transactionId);
                // ต้อง Manual Recovery หรือใช้ Compensating Transaction
                allCommitted = false;
            }
        }
        
        return allCommitted;
    }
    
    private void abortAll(String transactionId, 
                           List<PrepareResult> preparedParticipants) {
        preparedParticipants.stream()
            .filter(PrepareResult::isReady)
            .forEach(result -> {
                try {
                    result.getParticipant().abort(transactionId);
                } catch (Exception e) {
                    log.error("Failed to abort participant: {}", e.getMessage());
                }
            });
    }
}
```

---

## ขั้นตอนที่ 2216-2220: Saga Pattern vs 2PC

### Saga Pattern เป็นทางเลือกที่ดีกว่า

```
2PC:
- Synchronous
- Blocking
- All-or-nothing
- Tight coupling

Saga:
- Asynchronous  
- Non-blocking
- Eventual consistency
- Loose coupling
```

### Choreography Saga

```java
// OrderCreatedEventHandler.java - Choreography Saga
@Service
@RequiredArgsConstructor
@Slf4j
public class OrderSagaChoreography {
    
    private final KafkaTemplate<String, Object> kafkaTemplate;
    private final OrderRepository orderRepository;
    
    // Step 1: Order Created → Reserve Inventory
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void onOrderCreated(OrderCreatedEvent event) {
        log.info("Saga Step 1: Reserving inventory for order {}", 
                 event.getOrderId());
        
        kafkaTemplate.send("inventory.reserve", 
            new ReserveInventoryCommand(
                event.getOrderId(),
                event.getItems()
            ));
    }
    
    // Step 2: Inventory Reserved → Process Payment
    @KafkaListener(topics = "inventory.reserved")
    public void onInventoryReserved(InventoryReservedEvent event) {
        log.info("Saga Step 2: Processing payment for order {}", 
                 event.getOrderId());
        
        kafkaTemplate.send("payment.process",
            new ProcessPaymentCommand(
                event.getOrderId(),
                event.getTotalAmount()
            ));
    }
    
    // Step 3: Payment Processed → Confirm Order
    @KafkaListener(topics = "payment.processed")
    @Transactional
    public void onPaymentProcessed(PaymentProcessedEvent event) {
        log.info("Saga Step 3: Confirming order {}", event.getOrderId());
        
        orderRepository.updateStatus(event.getOrderId(), OrderStatus.CONFIRMED);
        
        kafkaTemplate.send("order.confirmed",
            new OrderConfirmedEvent(event.getOrderId()));
    }
    
    // Compensation: Payment Failed → Release Inventory
    @KafkaListener(topics = "payment.failed")
    public void onPaymentFailed(PaymentFailedEvent event) {
        log.warn("Saga Compensation: Payment failed for order {}, releasing inventory",
                event.getOrderId());
        
        kafkaTemplate.send("inventory.release",
            new ReleaseInventoryCommand(event.getOrderId()));
        
        orderRepository.updateStatus(event.getOrderId(), OrderStatus.PAYMENT_FAILED);
    }
    
    // Compensation: Inventory Not Available → Cancel Order
    @KafkaListener(topics = "inventory.insufficient")
    @Transactional
    public void onInventoryInsufficient(InventoryInsufficientEvent event) {
        log.warn("Saga Compensation: Insufficient inventory for order {}, cancelling",
                event.getOrderId());
        
        orderRepository.updateStatus(event.getOrderId(), OrderStatus.CANCELLED);
    }
}
```

### Orchestration Saga

```java
// OrderSagaOrchestrator.java - Orchestration Saga
@Service
@RequiredArgsConstructor
@Slf4j
public class OrderSagaOrchestrator {
    
    private final InventoryService inventoryService;
    private final PaymentService paymentService;
    private final OrderRepository orderRepository;
    private final SagaStateRepository sagaStateRepository;
    
    @Transactional
    public OrderResult executeCreateOrderSaga(CreateOrderRequest request) {
        SagaState sagaState = SagaState.start(
            "create-order-" + UUID.randomUUID(), 
            request
        );
        sagaStateRepository.save(sagaState);
        
        String reservationId = null;
        String paymentId = null;
        
        try {
            // Step 1: Reserve Inventory
            sagaState.setCurrentStep("RESERVE_INVENTORY");
            sagaStateRepository.save(sagaState);
            
            InventoryResult inventory = inventoryService.reserve(request.getItems());
            reservationId = inventory.getReservationId();
            sagaState.setData("reservationId", reservationId);
            
            // Step 2: Process Payment
            sagaState.setCurrentStep("PROCESS_PAYMENT");
            sagaStateRepository.save(sagaState);
            
            PaymentResult payment = paymentService.charge(request.getPaymentDetails());
            paymentId = payment.getPaymentId();
            sagaState.setData("paymentId", paymentId);
            
            // Step 3: Confirm Order
            sagaState.setCurrentStep("CONFIRM_ORDER");
            sagaStateRepository.save(sagaState);
            
            Order order = Order.builder()
                .customerId(request.getCustomerId())
                .status(OrderStatus.CONFIRMED)
                .paymentId(paymentId)
                .reservationId(reservationId)
                .build();
            
            Order saved = orderRepository.save(order);
            
            sagaState.setStatus(SagaStatus.COMPLETED);
            sagaStateRepository.save(sagaState);
            
            return OrderResult.success(saved);
            
        } catch (InventoryException e) {
            log.error("Saga failed at RESERVE_INVENTORY: {}", e.getMessage());
            sagaState.setStatus(SagaStatus.FAILED);
            sagaState.setFailureReason(e.getMessage());
            sagaStateRepository.save(sagaState);
            
            return OrderResult.failed("INVENTORY_UNAVAILABLE");
            
        } catch (PaymentException e) {
            log.error("Saga failed at PROCESS_PAYMENT: {}, compensating...", 
                     e.getMessage());
            
            // Compensate: Release Inventory
            if (reservationId != null) {
                try {
                    inventoryService.release(reservationId);
                    sagaState.setData("compensation.inventory", "RELEASED");
                } catch (Exception ce) {
                    log.error("Compensation failed for inventory: {}", ce.getMessage());
                    sagaState.setData("compensation.inventory", "FAILED: " + ce.getMessage());
                }
            }
            
            sagaState.setStatus(SagaStatus.COMPENSATED);
            sagaStateRepository.save(sagaState);
            
            return OrderResult.failed("PAYMENT_FAILED");
        }
    }
    
    // Resume Failed Saga จาก Checkpoint
    @Scheduled(fixedDelay = 60000)
    public void resumeStuckSagas() {
        List<SagaState> stuckSagas = sagaStateRepository
            .findByStatusAndCreatedAtBefore(
                SagaStatus.IN_PROGRESS,
                Instant.now().minus(Duration.ofMinutes(5))
            );
        
        stuckSagas.forEach(saga -> {
            log.warn("Found stuck saga: {} at step: {}", 
                    saga.getId(), saga.getCurrentStep());
            // Resume หรือ Compensate ตาม Business Logic
        });
    }
}
```

---

## ขั้นตอนที่ 2221-2225: Outbox Pattern ด้วย Debezium CDC

### อธิบาย

Outbox Pattern แก้ปัญหา Dual Write (บันทึก DB + ส่ง Event ต้องทำพร้อมกัน):

```
ปัญหาเดิม (Dual Write):
DB.save(order)         ← ถ้า 1 สำเร็จ แต่ 2 ล้ม = Inconsistent
Kafka.send(event)      

Outbox Pattern:
DB.save(order)         ← เป็น Transaction เดียวกัน!
DB.save(outbox_event)  

CDC (Debezium) อ่าน outbox_event จาก DB แล้วส่งไป Kafka อัตโนมัติ
```

```java
// OutboxEntity.java
@Entity
@Table(name = "outbox_events")
@Data
@Builder
public class OutboxEvent {
    
    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private String id;
    
    @Column(nullable = false)
    private String aggregateId;      // ID ของ Entity (เช่น Order ID)
    
    @Column(nullable = false)
    private String aggregateType;    // ประเภท Entity (เช่น "Order")
    
    @Column(nullable = false)
    private String eventType;        // ประเภท Event (เช่น "OrderCreated")
    
    @Column(nullable = false)
    @Lob
    private String payload;          // JSON Payload
    
    @Column(nullable = false)
    private Instant createdAt;
    
    @Column
    private Instant processedAt;     // เมื่อ CDC ประมวลผลแล้ว (optional tracking)
    
    private String correlationId;    // สำหรับ Tracing
}
```

```java
// OutboxOrderService.java
@Service
@RequiredArgsConstructor
@Slf4j
public class OutboxOrderService {
    
    private final OrderRepository orderRepository;
    private final OutboxRepository outboxRepository;
    private final ObjectMapper objectMapper;
    
    @Transactional  // ACID: Order + Outbox บันทึกใน Transaction เดียวกัน
    public Order createOrder(CreateOrderRequest request) {
        // 1. บันทึก Order
        Order order = Order.builder()
            .id(UUID.randomUUID().toString())
            .customerId(request.getCustomerId())
            .status(OrderStatus.PENDING)
            .items(request.getItems())
            .build();
        
        Order saved = orderRepository.save(order);
        
        // 2. บันทึก Event ลงใน Outbox (ใน Transaction เดียวกัน!)
        OrderCreatedEventPayload payload = OrderCreatedEventPayload.builder()
            .orderId(saved.getId())
            .customerId(saved.getCustomerId())
            .items(saved.getItems())
            .createdAt(Instant.now())
            .build();
        
        OutboxEvent outboxEvent = OutboxEvent.builder()
            .aggregateId(saved.getId())
            .aggregateType("Order")
            .eventType("OrderCreated")
            .payload(toJson(payload))
            .createdAt(Instant.now())
            .correlationId(MDC.get("traceId"))
            .build();
        
        outboxRepository.save(outboxEvent);
        
        // Debezium จะอ่าน outbox_events table แล้วส่ง Event ไปยัง Kafka
        // ไม่ต้องส่ง Kafka ที่นี่!
        
        log.info("Order {} created with outbox event", saved.getId());
        return saved;
    }
    
    private String toJson(Object obj) {
        try {
            return objectMapper.writeValueAsString(obj);
        } catch (JsonProcessingException e) {
            throw new RuntimeException("Failed to serialize event", e);
        }
    }
}
```

```yaml
# debezium-connector.json - Debezium MySQL Connector
{
  "name": "outbox-connector",
  "config": {
    "connector.class": "io.debezium.connector.mysql.MySqlConnector",
    "database.hostname": "mysql",
    "database.port": "3306",
    "database.user": "debezium",
    "database.password": "${DB_PASSWORD}",
    "database.server.id": "184054",
    "database.server.name": "order-service",
    "table.include.list": "order_service.outbox_events",
    "transforms": "outbox",
    "transforms.outbox.type": "io.debezium.transforms.outbox.EventRouter",
    "transforms.outbox.table.fields.additional.placement": "correlationId:header:correlationId",
    "transforms.outbox.route.topic.replacement": "${routedByValue}.events",
    "value.converter": "org.apache.kafka.connect.json.JsonConverter",
    "value.converter.schemas.enable": "false",
    "key.converter": "org.apache.kafka.connect.storage.StringConverter"
  }
}
```

```yaml
# docker-compose-debezium.yml
version: '3.8'
services:
  debezium:
    image: debezium/connect:2.5
    ports:
      - "8083:8083"
    environment:
      BOOTSTRAP_SERVERS: kafka:9092
      GROUP_ID: 1
      CONFIG_STORAGE_TOPIC: debezium_configs
      OFFSET_STORAGE_TOPIC: debezium_offsets
      STATUS_STORAGE_TOPIC: debezium_statuses
    depends_on:
      - kafka
      - mysql
```

---

## ขั้นตอนที่ 2226-2230: Compensating Transactions

```java
// CompensationService.java - Compensating Transactions
@Service
@RequiredArgsConstructor
@Slf4j
public class CompensationService {
    
    private final OrderRepository orderRepository;
    private final InventoryService inventoryService;
    private final PaymentService paymentService;
    private final CompensationLog compensationLog;
    
    @Transactional
    public void compensateOrder(String orderId, CompensationReason reason) {
        log.info("Starting compensation for order: {} reason: {}", 
                orderId, reason);
        
        Order order = orderRepository.findById(orderId)
            .orElseThrow(() -> new NotFoundException("Order: " + orderId));
        
        List<CompensationStep> steps = new ArrayList<>();
        
        // กำหนด Compensation Steps ตาม State ปัจจุบัน
        if (order.getPaymentId() != null) {
            steps.add(new CompensationStep("REFUND_PAYMENT", () ->
                paymentService.refund(order.getPaymentId(), 
                                      "Compensation: " + reason)));
        }
        
        if (order.getReservationId() != null) {
            steps.add(new CompensationStep("RELEASE_INVENTORY", () ->
                inventoryService.release(order.getReservationId())));
        }
        
        steps.add(new CompensationStep("CANCEL_ORDER", () -> {
            order.setStatus(OrderStatus.CANCELLED);
            order.setCancellationReason(reason.toString());
            orderRepository.save(order);
        }));
        
        // Execute Compensation Steps
        executeCompensationSteps(orderId, steps);
    }
    
    private void executeCompensationSteps(String orderId, 
                                            List<CompensationStep> steps) {
        for (CompensationStep step : steps) {
            try {
                log.info("Executing compensation step: {} for order: {}", 
                        step.getName(), orderId);
                
                step.execute();
                
                compensationLog.record(orderId, step.getName(), "SUCCESS");
                
            } catch (Exception e) {
                log.error("Compensation step {} failed for order {}: {}",
                         step.getName(), orderId, e.getMessage());
                
                compensationLog.record(orderId, step.getName(), 
                                      "FAILED: " + e.getMessage());
                
                // ไม่หยุด ยังคงพยายาม Compensate Steps อื่น
                // แต่ต้องแจ้ง Operations Team
                alertOpsTeam(orderId, step.getName(), e);
            }
        }
    }
    
    @Data
    @AllArgsConstructor
    private static class CompensationStep {
        private String name;
        private Runnable action;
        
        void execute() {
            action.run();
        }
    }
}
```

---

## ขั้นตอนที่ 2231-2235: Idempotency ใน Distributed Systems

### อธิบาย

Idempotency หมายความว่า Operation เดิมถูกเรียกกี่ครั้งก็ให้ผลลัพธ์เหมือนกัน สำคัญมากเพราะ Network ทำให้ Request อาจถูกส่งซ้ำ

```java
// IdempotencyKey.java
@Entity
@Table(name = "idempotency_keys")
@Data
@Builder
public class IdempotencyKey {
    
    @Id
    private String key;
    
    @Column(nullable = false)
    private String requestHash;      // Hash ของ Request Body
    
    @Column(nullable = false)
    @Lob
    private String responseBody;     // Cached Response
    
    @Column(nullable = false)
    private int responseStatus;
    
    @Column(nullable = false)
    private Instant createdAt;
    
    @Column(nullable = false)
    private Instant expiresAt;
}
```

```java
// IdempotencyService.java
@Service
@RequiredArgsConstructor
@Slf4j
public class IdempotencyService {
    
    private final IdempotencyKeyRepository repository;
    private final ObjectMapper objectMapper;
    
    public <T> T executeIdempotently(String idempotencyKey, 
                                      Object request, 
                                      Supplier<T> operation) {
        
        String requestHash = hashRequest(request);
        
        // ตรวจสอบว่า Key เคยใช้แล้วหรือยัง
        Optional<IdempotencyKey> existing = repository
            .findByKeyAndExpiresAtAfter(idempotencyKey, Instant.now());
        
        if (existing.isPresent()) {
            IdempotencyKey cached = existing.get();
            
            // ตรวจสอบว่า Request เหมือนกันไหม
            if (!cached.getRequestHash().equals(requestHash)) {
                throw new IdempotencyConflictException(
                    "Idempotency key reused with different request");
            }
            
            log.info("Returning cached response for idempotency key: {}", 
                    idempotencyKey);
            
            try {
                return objectMapper.readValue(cached.getResponseBody(), 
                    new TypeReference<T>() {});
            } catch (JsonProcessingException e) {
                throw new RuntimeException("Failed to deserialize cached response", e);
            }
        }
        
        // Execute Operation
        T result = operation.get();
        
        // Cache Result
        IdempotencyKey keyRecord = IdempotencyKey.builder()
            .key(idempotencyKey)
            .requestHash(requestHash)
            .responseBody(toJson(result))
            .responseStatus(200)
            .createdAt(Instant.now())
            .expiresAt(Instant.now().plus(Duration.ofDays(7)))
            .build();
        
        repository.save(keyRecord);
        
        return result;
    }
    
    private String hashRequest(Object request) {
        try {
            String json = objectMapper.writeValueAsString(request);
            MessageDigest md = MessageDigest.getInstance("SHA-256");
            byte[] hash = md.digest(json.getBytes(StandardCharsets.UTF_8));
            return Base64.getEncoder().encodeToString(hash);
        } catch (Exception e) {
            throw new RuntimeException("Failed to hash request", e);
        }
    }
    
    private String toJson(Object obj) {
        try {
            return objectMapper.writeValueAsString(obj);
        } catch (JsonProcessingException e) {
            throw new RuntimeException("Failed to serialize response", e);
        }
    }
}
```

```java
// IdempotencyFilter.java - Middleware สำหรับ HTTP Level Idempotency
@Component
@RequiredArgsConstructor
@Slf4j
public class IdempotencyFilter extends OncePerRequestFilter {
    
    private final IdempotencyService idempotencyService;
    
    @Override
    protected void doFilterInternal(HttpServletRequest request, 
                                     HttpServletResponse response,
                                     FilterChain filterChain) 
            throws ServletException, IOException {
        
        String idempotencyKey = request.getHeader("Idempotency-Key");
        
        // ตรวจสอบเฉพาะ POST/PUT/PATCH
        if (idempotencyKey == null || 
            !isIdempotencyRequired(request.getMethod())) {
            filterChain.doFilter(request, response);
            return;
        }
        
        // ตรวจสอบว่า Request นี้เคยทำแล้วหรือไม่
        Optional<CachedResponse> cached = idempotencyService
            .findCachedResponse(idempotencyKey);
        
        if (cached.isPresent()) {
            log.info("Returning idempotent response for key: {}", 
                    idempotencyKey);
            
            CachedResponse cachedResp = cached.get();
            response.setStatus(cachedResp.getStatus());
            response.setContentType("application/json");
            response.getWriter().write(cachedResp.getBody());
            return;
        }
        
        // Wrap Response เพื่อ Capture ก่อน Send
        ContentCachingResponseWrapper wrappedResponse = 
            new ContentCachingResponseWrapper(response);
        
        filterChain.doFilter(request, wrappedResponse);
        
        // Cache Response ถ้า Success
        if (wrappedResponse.getStatus() < 400) {
            byte[] responseBody = wrappedResponse.getContentAsByteArray();
            idempotencyService.cacheResponse(
                idempotencyKey,
                wrappedResponse.getStatus(),
                new String(responseBody, StandardCharsets.UTF_8)
            );
        }
        
        wrappedResponse.copyBodyToResponse();
    }
    
    private boolean isIdempotencyRequired(String method) {
        return "POST".equals(method) || 
               "PUT".equals(method) || 
               "PATCH".equals(method);
    }
}
```

---

## ขั้นตอนที่ 2236-2240: Testing Eventual Consistency

```java
// EventualConsistencyTest.java
@SpringBootTest
@ActiveProfiles("test")
class EventualConsistencyTest {
    
    @Autowired
    private OrderService orderService;
    
    @Autowired
    private InventoryService inventoryService;
    
    @Autowired
    private KafkaTestListener kafkaTestListener;
    
    @Test
    void orderCreationShouldEventuallyUpdateInventory() throws Exception {
        // Arrange
        String productId = "PROD-001";
        int initialInventory = inventoryService.getAvailableQuantity(productId);
        
        // Act: Create Order
        CreateOrderRequest request = CreateOrderRequest.builder()
            .customerId("CUST-001")
            .items(List.of(OrderItem.of(productId, 2)))
            .build();
        
        Order order = orderService.createOrder(request);
        
        // Assert: Order Created
        assertThat(order.getStatus()).isEqualTo(OrderStatus.PENDING);
        
        // ณ จุดนี้ Inventory ยังไม่ได้ Update (Eventual)
        
        // รอ Event Propagation (Test ต้องรอ แต่ Production ไม่ต้องรอ)
        await()
            .atMost(Duration.ofSeconds(10))
            .pollInterval(Duration.ofMillis(200))
            .untilAsserted(() -> {
                int currentInventory = inventoryService
                    .getAvailableQuantity(productId);
                
                assertThat(currentInventory)
                    .isEqualTo(initialInventory - 2)
                    .as("Inventory should be reduced by 2 eventually");
            });
    }
    
    @Test
    void compensationShouldRestoreInventoryOnPaymentFailure() throws Exception {
        String productId = "PROD-002";
        int initialInventory = inventoryService.getAvailableQuantity(productId);
        
        // Create Order แต่ Payment จะ Fail
        CreateOrderRequest request = CreateOrderRequest.builder()
            .customerId("CUST-002")
            .items(List.of(OrderItem.of(productId, 1)))
            .paymentDetails(PaymentDetails.invalid()) // Invalid Payment
            .build();
        
        Order order = orderService.createOrder(request);
        
        // รอ Saga Completion
        await()
            .atMost(Duration.ofSeconds(15))
            .untilAsserted(() -> {
                Order updated = orderService.getOrder(order.getId());
                assertThat(updated.getStatus())
                    .isIn(OrderStatus.PAYMENT_FAILED, OrderStatus.CANCELLED)
                    .as("Order should be failed or cancelled");
            });
        
        // ตรวจสอบว่า Inventory ถูก Restore
        await()
            .atMost(Duration.ofSeconds(15))
            .untilAsserted(() -> {
                int currentInventory = inventoryService
                    .getAvailableQuantity(productId);
                
                assertThat(currentInventory)
                    .isEqualTo(initialInventory)
                    .as("Inventory should be restored after payment failure");
            });
    }
    
    @Test
    void idempotentOrderCreation() {
        String idempotencyKey = "order-" + UUID.randomUUID();
        CreateOrderRequest request = createTestRequest();
        
        // สร้าง Order ครั้งแรก
        Order firstOrder = orderService.createOrderIdempotently(
            idempotencyKey, request);
        
        // สร้าง Order ครั้งที่สอง (ต้องได้ Order เดิม)
        Order secondOrder = orderService.createOrderIdempotently(
            idempotencyKey, request);
        
        assertThat(firstOrder.getId()).isEqualTo(secondOrder.getId());
        assertThat(firstOrder.getStatus()).isEqualTo(secondOrder.getStatus());
        
        // ตรวจสอบว่า Order ถูกสร้างแค่ครั้งเดียว
        long orderCount = orderRepository.countByIdempotencyKey(idempotencyKey);
        assertThat(orderCount).isEqualTo(1L);
    }
    
    @Test
    void outboxEventShouldBePublishedToKafka() throws Exception {
        String productId = "PROD-003";
        
        Order order = orderService.createOrder(CreateOrderRequest.builder()
            .customerId("CUST-003")
            .items(List.of(OrderItem.of(productId, 1)))
            .build());
        
        // ตรวจสอบว่า Event ถูกส่งไป Kafka ผ่าน Outbox
        await()
            .atMost(Duration.ofSeconds(10))
            .untilAsserted(() -> {
                List<String> events = kafkaTestListener
                    .getEvents("Order.events");
                
                boolean found = events.stream()
                    .anyMatch(e -> e.contains(order.getId()) && 
                                  e.contains("OrderCreated"));
                
                assertThat(found)
                    .isTrue()
                    .as("OrderCreated event should be in Kafka");
            });
    }
}
```

---

## สรุป Data Consistency Patterns

| Pattern | เมื่อใช้ | Trade-off |
|---------|---------|---------|
| Strong Consistency (2PC) | Financial, Critical Data | Performance, Availability |
| Eventual Consistency | Scale, High Availability | Complexity, Stale Reads |
| Saga (Choreography) | Independent Services | Hard to track, debugging |
| Saga (Orchestration) | Central Control needed | Orchestrator is SPOF |
| Outbox Pattern | Event Publishing + DB | Requires CDC setup |
| Idempotency | Retry-safe operations | Storage overhead |
| Compensating Transactions | Long-running processes | Complex logic |

---

## Decision Tree สำหรับเลือก Pattern

```
ต้องการ Strong Consistency?
├── YES: ใช้ Database Transactions (ACID)
│   └── ข้ามหลาย Services? → 2PC (ระวัง Performance)
└── NO: Eventual Consistency ยอมรับได้?
    ├── YES: Saga Pattern
    │   ├── Services Independent? → Choreography Saga
    │   └── Central Control? → Orchestration Saga
    └── Event Publishing?
        └── YES: Outbox Pattern + CDC (Debezium)
```

---

*[← Part 64: Resilience Patterns](./part-64-resilience-patterns.md) | [Part 66: Advanced Security →](./part-66-advanced-security.md)*

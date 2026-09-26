# Part 26: Message Queue - Kafka & RabbitMQ
## ขั้นตอนที่ 701-735

> **ระดับ:** สูง (Advanced)  
> **เวลาเรียน:** 6-7 ชั่วโมง  
> **เป้าหมาย:** Event-Driven Architecture ด้วย Kafka และ RabbitMQ

---

## ขั้นตอนที่ 701: Event-Driven Architecture

```
ทำไมต้อง Message Queue?
  - Decouple services (ส่ง event ไม่ต้องรู้ว่าใครรับ)
  - Async processing (ไม่ต้องรอ)
  - Resilience (ถ้า consumer ล่ม message ยังอยู่ใน queue)
  - Scalability (consumer scale independently)

Kafka:
  ✅ High throughput (millions of messages/sec)
  ✅ Persistent messages (replay)
  ✅ Multiple consumer groups
  ✅ Best for: event streaming, audit logs, analytics

RabbitMQ:
  ✅ Complex routing (direct, topic, fanout, headers)
  ✅ Message acknowledgment
  ✅ Dead letter queue
  ✅ Best for: task queue, notifications, complex routing
```

---

## ขั้นตอนที่ 702: Apache Kafka Setup

```yaml
# docker-compose.yml
services:
  zookeeper:
    image: confluentinc/cp-zookeeper:7.5.0
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
    ports:
      - "2181:2181"

  kafka:
    image: confluentinc/cp-kafka:7.5.0
    depends_on:
      - zookeeper
    ports:
      - "9092:9092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_AUTO_CREATE_TOPICS_ENABLE: false

  kafka-ui:
    image: provectuslabs/kafka-ui:latest
    depends_on:
      - kafka
    ports:
      - "8090:8080"
    environment:
      KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: kafka:9092
```

---

## ขั้นตอนที่ 703: Kafka Dependencies

```xml
<dependency>
    <groupId>org.springframework.kafka</groupId>
    <artifactId>spring-kafka</artifactId>
</dependency>
```

```yaml
# application.yml
spring:
  kafka:
    bootstrap-servers: localhost:9092
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
      acks: all  # Wait for all replicas
      retries: 3
      properties:
        spring.json.add.type.headers: false
    
    consumer:
      group-id: order-service
      auto-offset-reset: earliest
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
      properties:
        spring.json.trusted.packages: "com.myapp.*"
    
    topics:
      order-created: orders.created
      order-shipped: orders.shipped
      payment-processed: payments.processed
```

---

## ขั้นตอนที่ 704: Kafka Configuration

```java
@Configuration
@EnableKafka
public class KafkaConfig {
    
    @Value("${spring.kafka.bootstrap-servers}")
    private String bootstrapServers;
    
    // Producer
    @Bean
    public ProducerFactory<String, Object> producerFactory() {
        Map<String, Object> config = new HashMap<>();
        config.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
        config.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        config.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, JsonSerializer.class);
        config.put(ProducerConfig.ACKS_CONFIG, "all");
        config.put(ProducerConfig.RETRIES_CONFIG, 3);
        config.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);
        return new DefaultKafkaProducerFactory<>(config);
    }
    
    @Bean
    public KafkaTemplate<String, Object> kafkaTemplate() {
        return new KafkaTemplate<>(producerFactory());
    }
    
    // Consumer
    @Bean
    public ConsumerFactory<String, Object> consumerFactory() {
        Map<String, Object> config = new HashMap<>();
        config.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
        config.put(ConsumerConfig.GROUP_ID_CONFIG, "order-service");
        config.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        config.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, JsonDeserializer.class);
        config.put(JsonDeserializer.TRUSTED_PACKAGES, "com.myapp.*");
        config.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
        return new DefaultKafkaConsumerFactory<>(config);
    }
    
    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, Object> kafkaListenerContainerFactory() {
        ConcurrentKafkaListenerContainerFactory<String, Object> factory = 
            new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactory());
        factory.setConcurrency(3);  // 3 concurrent consumers
        factory.getContainerProperties().setAckMode(ContainerProperties.AckMode.MANUAL_IMMEDIATE);
        
        // Error handling
        factory.setCommonErrorHandler(new DefaultErrorHandler(
            new DeadLetterPublishingRecoverer(kafkaTemplate()),
            new FixedBackOff(1000L, 3)  // 3 retries, 1s delay
        ));
        
        return factory;
    }
    
    // Create topics
    @Bean
    public NewTopic orderCreatedTopic() {
        return TopicBuilder.name("orders.created")
            .partitions(3)
            .replicas(1)
            .build();
    }
    
    @Bean
    public NewTopic orderShippedTopic() {
        return TopicBuilder.name("orders.shipped")
            .partitions(3)
            .replicas(1)
            .build();
    }
}
```

---

## ขั้นตอนที่ 705: Kafka Producer

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class OrderEventProducer {
    
    private final KafkaTemplate<String, Object> kafkaTemplate;
    
    public void publishOrderCreated(Order order) {
        OrderCreatedEvent event = OrderCreatedEvent.builder()
            .orderId(order.getId())
            .orderNumber(order.getOrderNumber())
            .userId(order.getUser().getId())
            .totalAmount(order.getTotalAmount())
            .items(order.getItems().stream()
                .map(i -> new OrderItemEvent(i.getProduct().getId(), i.getQuantity(), i.getPrice()))
                .toList())
            .occurredAt(LocalDateTime.now())
            .build();
        
        // Send with key = orderId (ensures ordering within partition)
        var future = kafkaTemplate.send("orders.created", 
            String.valueOf(order.getId()), event);
        
        future.whenComplete((result, ex) -> {
            if (ex != null) {
                log.error("Failed to publish OrderCreated: {}", ex.getMessage(), ex);
            } else {
                log.info("Published OrderCreated: topic={}, partition={}, offset={}",
                    result.getRecordMetadata().topic(),
                    result.getRecordMetadata().partition(),
                    result.getRecordMetadata().offset());
            }
        });
    }
    
    public void publishOrderShipped(Long orderId, String trackingNumber) {
        OrderShippedEvent event = new OrderShippedEvent(orderId, trackingNumber, LocalDateTime.now());
        kafkaTemplate.send("orders.shipped", String.valueOf(orderId), event);
    }
}
```

---

## ขั้นตอนที่ 706: Kafka Consumer

```java
@Component
@RequiredArgsConstructor
@Slf4j
public class OrderEventConsumer {
    
    private final NotificationService notificationService;
    private final InventoryService inventoryService;
    private final EmailService emailService;
    
    @KafkaListener(
        topics = "orders.created",
        groupId = "notification-service",
        containerFactory = "kafkaListenerContainerFactory"
    )
    public void handleOrderCreated(
        @Payload OrderCreatedEvent event,
        @Header(KafkaHeaders.RECEIVED_TOPIC) String topic,
        @Header(KafkaHeaders.RECEIVED_PARTITION) int partition,
        @Header(KafkaHeaders.OFFSET) long offset,
        Acknowledgment ack
    ) {
        log.info("Received OrderCreated: orderId={}, topic={}, partition={}, offset={}",
            event.getOrderId(), topic, partition, offset);
        
        try {
            // Process event
            notificationService.sendOrderConfirmation(event);
            emailService.sendOrderConfirmationEmail(event.getUserEmail(), mapToResponse(event));
            
            // Manual ack - only ack after successful processing
            ack.acknowledge();
            
        } catch (Exception e) {
            log.error("Error processing OrderCreated: {}", e.getMessage(), e);
            // Don't ack - message will be retried or go to DLT
            throw e;
        }
    }
    
    @KafkaListener(
        topics = "orders.created",
        groupId = "inventory-service"  // Different consumer group!
    )
    public void updateInventory(OrderCreatedEvent event, Acknowledgment ack) {
        log.info("Updating inventory for order: {}", event.getOrderId());
        
        // Each item in order reduces inventory
        for (var item : event.getItems()) {
            inventoryService.reduceStock(item.getProductId(), item.getQuantity());
        }
        
        ack.acknowledge();
    }
    
    // Dead Letter Topic consumer
    @KafkaListener(topics = "orders.created.DLT", groupId = "dlq-handler")
    public void handleDlq(OrderCreatedEvent event, Acknowledgment ack) {
        log.error("DLQ received failed order event: {}", event.getOrderId());
        // Alert, manual review, or alternative processing
        ack.acknowledge();
    }
}
```

---

## ขั้นตอนที่ 707: RabbitMQ Setup

```yaml
# docker-compose.yml
services:
  rabbitmq:
    image: rabbitmq:3-management
    ports:
      - "5672:5672"
      - "15672:15672"  # Management UI
    environment:
      RABBITMQ_DEFAULT_USER: admin
      RABBITMQ_DEFAULT_PASS: admin123
```

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-amqp</artifactId>
</dependency>
```

---

## ขั้นตอนที่ 708: RabbitMQ Configuration

```java
@Configuration
public class RabbitMQConfig {
    
    public static final String ORDER_EXCHANGE = "orders.exchange";
    public static final String NOTIFICATION_QUEUE = "notification.queue";
    public static final String EMAIL_QUEUE = "email.queue";
    public static final String DLQ = "orders.dlq";
    public static final String DLQ_EXCHANGE = "orders.dlq.exchange";
    
    // Exchange
    @Bean
    public TopicExchange orderExchange() {
        return ExchangeBuilder.topicExchange(ORDER_EXCHANGE)
            .durable(true)
            .build();
    }
    
    @Bean
    public DirectExchange dlqExchange() {
        return ExchangeBuilder.directExchange(DLQ_EXCHANGE).durable(true).build();
    }
    
    // Dead Letter Queue
    @Bean
    public Queue dlq() {
        return QueueBuilder.durable(DLQ).build();
    }
    
    @Bean
    public Binding dlqBinding() {
        return BindingBuilder.bind(dlq()).to(dlqExchange()).with(DLQ);
    }
    
    // Notification Queue
    @Bean
    public Queue notificationQueue() {
        return QueueBuilder.durable(NOTIFICATION_QUEUE)
            .withArgument("x-dead-letter-exchange", DLQ_EXCHANGE)
            .withArgument("x-dead-letter-routing-key", DLQ)
            .withArgument("x-message-ttl", 3600000)  // 1 hour
            .build();
    }
    
    @Bean
    public Binding notificationBinding() {
        return BindingBuilder.bind(notificationQueue())
            .to(orderExchange())
            .with("orders.#");  // All order events
    }
    
    // Email Queue
    @Bean
    public Queue emailQueue() {
        return QueueBuilder.durable(EMAIL_QUEUE).build();
    }
    
    @Bean
    public Binding emailBinding() {
        return BindingBuilder.bind(emailQueue())
            .to(orderExchange())
            .with("orders.created");  // Only order.created
    }
    
    // Message Converter
    @Bean
    public MessageConverter messageConverter(ObjectMapper objectMapper) {
        return new Jackson2JsonMessageConverter(objectMapper);
    }
    
    @Bean
    public RabbitTemplate rabbitTemplate(ConnectionFactory factory, MessageConverter converter) {
        RabbitTemplate template = new RabbitTemplate(factory);
        template.setMessageConverter(converter);
        template.setConfirmCallback((correlation, ack, reason) -> {
            if (!ack) log.error("Message not delivered: {}", reason);
        });
        return template;
    }
}
```

---

## ขั้นตอนที่ 709: RabbitMQ Producer/Consumer

```java
// Producer
@Service
@RequiredArgsConstructor
public class OrderMessagePublisher {
    
    private final RabbitTemplate rabbitTemplate;
    
    public void publishOrderCreated(OrderCreatedEvent event) {
        rabbitTemplate.convertAndSend(
            RabbitMQConfig.ORDER_EXCHANGE, 
            "orders.created",
            event
        );
    }
}

// Consumer
@Component
@RequiredArgsConstructor
@Slf4j
public class OrderMessageConsumer {
    
    private final NotificationService notificationService;
    
    @RabbitListener(queues = RabbitMQConfig.NOTIFICATION_QUEUE)
    public void handleOrderCreated(OrderCreatedEvent event, 
                                    Channel channel,
                                    @Header(AmqpHeaders.DELIVERY_TAG) long deliveryTag) {
        try {
            notificationService.sendOrderConfirmation(event);
            channel.basicAck(deliveryTag, false);  // Acknowledge
        } catch (Exception e) {
            log.error("Processing failed: {}", e.getMessage());
            channel.basicNack(deliveryTag, false, false);  // Reject, don't requeue → DLQ
        }
    }
}
```

---

## ขั้นตอนที่ 710-735: Transactional Outbox Pattern

```java
// Ensure message is sent exactly once, even if app crashes
@Entity
@Table(name = "outbox_events")
@Getter @Setter @Builder @NoArgsConstructor @AllArgsConstructor
public class OutboxEvent extends BaseEntity {
    
    @Column(name = "aggregate_type")
    private String aggregateType;  // e.g., "ORDER"
    
    @Column(name = "aggregate_id")
    private String aggregateId;
    
    @Column(name = "event_type")
    private String eventType;  // e.g., "ORDER_CREATED"
    
    @Column(columnDefinition = "jsonb")
    @JdbcTypeCode(SqlTypes.JSON)
    private String payload;
    
    @Column(name = "processed_at")
    private LocalDateTime processedAt;
    
    private boolean processed = false;
}

// In OrderService - save event in same transaction as order
@Transactional
public OrderResponse create(Long userId, CreateOrderRequest request) {
    // ... create order ...
    Order saved = orderRepository.save(order);
    
    // Save event in outbox (same transaction!)
    OutboxEvent event = OutboxEvent.builder()
        .aggregateType("ORDER")
        .aggregateId(String.valueOf(saved.getId()))
        .eventType("ORDER_CREATED")
        .payload(objectMapper.writeValueAsString(new OrderCreatedEvent(saved)))
        .build();
    
    outboxRepository.save(event);
    return orderMapper.toResponse(saved);
}

// Outbox Processor - runs separately
@Component
@RequiredArgsConstructor
@Slf4j
public class OutboxEventProcessor {
    
    private final OutboxRepository outboxRepository;
    private final KafkaTemplate<String, Object> kafkaTemplate;
    
    @Scheduled(fixedDelay = 1000)
    @Transactional
    public void processOutbox() {
        List<OutboxEvent> pending = outboxRepository
            .findTop100ByProcessedFalseOrderByCreatedAt();
        
        for (OutboxEvent event : pending) {
            try {
                kafkaTemplate.send(
                    "orders." + event.getEventType().toLowerCase().replace("_", "."),
                    event.getAggregateId(),
                    event.getPayload()
                );
                
                event.setProcessed(true);
                event.setProcessedAt(LocalDateTime.now());
                outboxRepository.save(event);
                
            } catch (Exception e) {
                log.error("Failed to publish outbox event {}: {}", event.getId(), e.getMessage());
            }
        }
    }
}
```

---

*[← Part 25: Microservices Intro](./part-25-microservices-intro.md) | [Part 27: API Documentation →](./part-27-api-documentation.md)*

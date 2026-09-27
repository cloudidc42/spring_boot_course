# Part 74: Event-Driven Architecture Advanced Patterns
## ขั้นตอนที่ 2561-2600

**ระดับ:** ระดับโลก (World-Class)
**เวลาเรียน:** 5-6 ชั่วโมง
**เป้าหมาย:** เรียนรู้ Advanced Patterns ใน Event-Driven Architecture ตั้งแต่ Event Versioning, Apache Avro, Kafka Streams, Event Replay, Dead Letter Queue และ Exactly-Once Semantics

---

## สารบัญ

1. [Event Versioning and Schema Evolution](#event-versioning)
2. [Apache Avro with Schema Registry](#avro)
3. [Kafka Streams for Real-Time Processing](#kafka-streams)
4. [Event Replay and Reprocessing](#event-replay)
5. [Dead Letter Queue Handling](#dlq)
6. [Exactly-Once Semantics](#exactly-once)

---

## ขั้นตอนที่ 2561: Event Versioning Strategies {#event-versioning}

### ทำไม Event Versioning ถึงสำคัญ?

ใน Event-Driven Architecture เมื่อ producer เปลี่ยน event schema โดยไม่ระวัง consumers ทั้งหมดที่ subscribe จะพัง เพราะ events ถูกเก็บใน Kafka log ซึ่งอาจมี consumer ที่ต้อง replay events เก่า

### 3 Strategies สำหรับ Event Versioning

**1. Versioned Event Type**
```java
// events/OrderCreatedV1.java
public record OrderCreatedV1(
    Long orderId,
    String customerId,
    BigDecimal amount  // v1 field name
) {}

// events/OrderCreatedV2.java  
public record OrderCreatedV2(
    Long orderId,
    String customerId,
    BigDecimal totalAmount,    // renamed
    String currency,           // new field
    List<OrderItem> items      // new field
) {}
```

**2. Single Event Type with Optional Fields**
```java
// events/OrderCreated.java - backward compatible
@JsonInclude(JsonInclude.Include.NON_NULL)
public class OrderCreated {
    private final String schemaVersion;  // "1.0", "2.0"
    private final Long orderId;
    private final String customerId;
    
    // v1 field (deprecated in v2)
    @Deprecated
    private final BigDecimal amount;
    
    // v2+ fields
    private final BigDecimal totalAmount;
    private final String currency;
    private final List<OrderItem> items;
}
```

**3. Upcasting (แนะนำสำหรับ Axon Framework)**
```java
package com.example.events.upcaster;

import org.axonframework.serialization.upcasting.event.SingleEventUpcaster;
import org.dom4j.Document;

// แปลง v1 → v2 event โดยอัตโนมัติ
public class OrderCreatedEventUpcaster extends SingleEventUpcaster {

    @Override
    protected boolean canUpcast(EventRepresentation intermediateRepresentation) {
        return intermediateRepresentation.getType().getName()
            .equals("com.example.events.OrderCreatedV1");
    }

    @Override
    protected EventRepresentation doUpcast(EventRepresentation ir) {
        return ir.withType("com.example.events.OrderCreatedV2")
            .withPayload(transformPayload(ir.getPayload()));
    }

    private Document transformPayload(Document v1Payload) {
        // Rename "amount" to "totalAmount"
        v1Payload.getRootElement()
            .element("amount")
            .setName("totalAmount");
        
        // Add default currency
        v1Payload.getRootElement()
            .addElement("currency")
            .setText("THB");
        
        return v1Payload;
    }
}
```

---

## ขั้นตอนที่ 2565: Apache Avro with Schema Registry {#avro}

### ทำไมต้องใช้ Avro + Schema Registry?

1. **Binary Format** - Compact กว่า JSON มาก (ลด network overhead)
2. **Schema Enforcement** - ไม่สามารถส่ง event ที่ไม่ตรงกับ schema ได้
3. **Schema Evolution** - รองรับการเพิ่ม/ลบ field อย่างปลอดภัย
4. **Schema Registry** - เก็บ schema history และ compatibility rules

### Maven Dependencies

```xml
<!-- pom.xml -->
<dependencies>
    <dependency>
        <groupId>org.apache.avro</groupId>
        <artifactId>avro</artifactId>
        <version>1.11.3</version>
    </dependency>
    <dependency>
        <groupId>io.confluent</groupId>
        <artifactId>kafka-avro-serializer</artifactId>
        <version>7.5.0</version>
    </dependency>
</dependencies>

<build>
    <plugins>
        <plugin>
            <groupId>org.apache.avro</groupId>
            <artifactId>avro-maven-plugin</artifactId>
            <version>1.11.3</version>
            <executions>
                <execution>
                    <phase>generate-sources</phase>
                    <goals><goal>schema</goal></goals>
                    <configuration>
                        <sourceDirectory>src/main/avro</sourceDirectory>
                        <outputDirectory>target/generated-sources/avro</outputDirectory>
                    </configuration>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

### Avro Schema Definition

```json
// src/main/avro/OrderCreated.avsc
{
  "namespace": "com.example.events",
  "type": "record",
  "name": "OrderCreated",
  "doc": "Event published when a new order is created",
  "fields": [
    {
      "name": "eventId",
      "type": "string",
      "doc": "Unique ID for this event (UUID)"
    },
    {
      "name": "orderId",
      "type": "long",
      "doc": "Order identifier"
    },
    {
      "name": "customerId",
      "type": "string",
      "doc": "Customer identifier"
    },
    {
      "name": "totalAmount",
      "type": {
        "type": "bytes",
        "logicalType": "decimal",
        "precision": 10,
        "scale": 2
      },
      "doc": "Total order amount"
    },
    {
      "name": "currency",
      "type": "string",
      "default": "THB",
      "doc": "Currency code"
    },
    {
      "name": "status",
      "type": {
        "type": "enum",
        "name": "OrderStatus",
        "symbols": ["PENDING", "CONFIRMED", "PROCESSING", "SHIPPED", "DELIVERED", "CANCELLED"]
      }
    },
    {
      "name": "items",
      "type": {
        "type": "array",
        "items": {
          "type": "record",
          "name": "OrderItem",
          "fields": [
            {"name": "productId", "type": "string"},
            {"name": "quantity", "type": "int"},
            {"name": "price", "type": {"type": "bytes", "logicalType": "decimal", "precision": 10, "scale": 2}}
          ]
        }
      }
    },
    {
      "name": "createdAt",
      "type": {
        "type": "long",
        "logicalType": "timestamp-millis"
      },
      "doc": "Event creation timestamp in milliseconds"
    },
    {
      "name": "metadata",
      "type": ["null", {"type": "map", "values": "string"}],
      "default": null,
      "doc": "Optional additional metadata"
    }
  ]
}
```

### Kafka Configuration พร้อม Schema Registry

```java
package com.example.config;

import io.confluent.kafka.serializers.KafkaAvroSerializer;
import io.confluent.kafka.serializers.KafkaAvroDeserializer;
import org.apache.kafka.clients.producer.ProducerConfig;
import org.apache.kafka.clients.consumer.ConsumerConfig;
import org.springframework.kafka.core.*;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class KafkaAvroConfig {

    @Value("${spring.kafka.bootstrap-servers}")
    private String bootstrapServers;

    @Value("${schema.registry.url}")
    private String schemaRegistryUrl;

    @Bean
    public ProducerFactory<String, Object> avroProducerFactory() {
        Map<String, Object> config = new HashMap<>();
        config.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
        config.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, 
            org.apache.kafka.common.serialization.StringSerializer.class);
        config.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, 
            KafkaAvroSerializer.class);
        config.put("schema.registry.url", schemaRegistryUrl);
        
        // ส่ง schema name ใน header
        config.put("auto.register.schemas", true);
        config.put("use.latest.version", false);
        
        // Reliability settings
        config.put(ProducerConfig.ACKS_CONFIG, "all");
        config.put(ProducerConfig.RETRIES_CONFIG, 3);
        config.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);
        
        return new DefaultKafkaProducerFactory<>(config);
    }

    @Bean
    public ConsumerFactory<String, Object> avroConsumerFactory() {
        Map<String, Object> config = new HashMap<>();
        config.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
        config.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG,
            org.apache.kafka.common.serialization.StringDeserializer.class);
        config.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG,
            KafkaAvroDeserializer.class);
        config.put("schema.registry.url", schemaRegistryUrl);
        config.put("specific.avro.reader", true);  // ใช้ generated classes
        
        // Offset management
        config.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
        config.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);
        
        return new DefaultKafkaConsumerFactory<>(config);
    }
}
```

### Event Publisher

```java
package com.example.events;

import com.example.events.avro.OrderCreated;
import com.example.events.avro.OrderStatus;
import org.apache.avro.specific.SpecificRecord;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Component;
import java.util.UUID;

@Component
public class OrderEventPublisher {

    private final KafkaTemplate<String, Object> kafkaTemplate;

    public OrderEventPublisher(KafkaTemplate<String, Object> kafkaTemplate) {
        this.kafkaTemplate = kafkaTemplate;
    }

    public void publishOrderCreated(Order order) {
        // สร้าง Avro event (generated class)
        OrderCreated event = OrderCreated.newBuilder()
            .setEventId(UUID.randomUUID().toString())
            .setOrderId(order.getId())
            .setCustomerId(order.getCustomerId())
            .setTotalAmount(order.getTotalAmount().unscaledValue().toByteArray())
            .setCurrency("THB")
            .setStatus(OrderStatus.PENDING)
            .setItems(convertItems(order.getItems()))
            .setCreatedAt(System.currentTimeMillis())
            .build();

        // ส่งโดยใช้ orderId เป็น key (partition ด้วย order)
        kafkaTemplate.send("order-events", order.getId().toString(), event)
            .whenComplete((result, ex) -> {
                if (ex != null) {
                    log.error("Failed to publish OrderCreated event for orderId: {}", 
                        order.getId(), ex);
                } else {
                    log.info("Published OrderCreated event: orderId={}, partition={}, offset={}",
                        order.getId(),
                        result.getRecordMetadata().partition(),
                        result.getRecordMetadata().offset());
                }
            });
    }

    private List<com.example.events.avro.OrderItem> convertItems(List<OrderItem> items) {
        return items.stream()
            .map(item -> com.example.events.avro.OrderItem.newBuilder()
                .setProductId(item.getProductId())
                .setQuantity(item.getQuantity())
                .setPrice(item.getPrice().unscaledValue().toByteArray())
                .build())
            .collect(Collectors.toList());
    }
}
```

---

## ขั้นตอนที่ 2570: Kafka Streams for Real-Time Processing {#kafka-streams}

### Kafka Streams Application

```java
package com.example.streams;

import com.example.events.avro.OrderCreated;
import com.example.events.avro.OrderStats;
import org.apache.kafka.streams.StreamsBuilder;
import org.apache.kafka.streams.kstream.*;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import java.time.Duration;

@Configuration
public class OrderStreamProcessor {

    @Bean
    public KStream<String, OrderCreated> orderStream(StreamsBuilder streamsBuilder) {
        
        // อ่าน events จาก topic
        KStream<String, OrderCreated> orders = streamsBuilder
            .stream("order-events", 
                Consumed.with(Serdes.String(), avroSerde(OrderCreated.class)));

        // 1. Filter orders ที่มีมูลค่าสูง
        KStream<String, OrderCreated> highValueOrders = orders
            .filter((key, order) -> {
                BigDecimal amount = new BigDecimal(
                    new java.math.BigInteger(order.getTotalAmount().array()), 2);
                return amount.compareTo(new BigDecimal("10000")) > 0;
            });

        // ส่งไปยัง VIP processing topic
        highValueOrders.to("high-value-orders",
            Produced.with(Serdes.String(), avroSerde(OrderCreated.class)));

        // 2. Aggregate: นับ orders และรวมยอดต่อชั่วโมง
        KTable<Windowed<String>, OrderStats> hourlyStats = orders
            .groupBy(
                (key, order) -> order.getCustomerId().toString(),
                Grouped.with(Serdes.String(), avroSerde(OrderCreated.class))
            )
            .windowedBy(TimeWindows.ofSizeWithNoGrace(Duration.ofHours(1)))
            .aggregate(
                OrderStats::new,  // initializer
                (customerId, order, stats) -> {
                    stats.setCustomerId(customerId);
                    stats.setOrderCount(stats.getOrderCount() + 1);
                    BigDecimal amount = new BigDecimal(
                        new java.math.BigInteger(order.getTotalAmount().array()), 2);
                    stats.setTotalAmount(
                        (stats.getTotalAmount() == null ? BigDecimal.ZERO : stats.getTotalAmount())
                            .add(amount));
                    return stats;
                },
                Materialized.<String, OrderStats, WindowStore<Bytes, byte[]>>as("hourly-order-stats")
                    .withValueSerde(avroSerde(OrderStats.class))
            );

        // 3. Join: Enrich order ด้วย customer info
        KTable<String, CustomerInfo> customers = streamsBuilder
            .table("customer-info",
                Consumed.with(Serdes.String(), avroSerde(CustomerInfo.class)));

        KStream<String, EnrichedOrder> enrichedOrders = orders
            .join(
                customers,
                (order, customer) -> EnrichedOrder.builder()
                    .order(order)
                    .customerName(customer.getName())
                    .customerTier(customer.getTier())
                    .build(),
                Joined.with(Serdes.String(), avroSerde(OrderCreated.class), avroSerde(CustomerInfo.class))
            );

        enrichedOrders.to("enriched-orders",
            Produced.with(Serdes.String(), avroSerde(EnrichedOrder.class)));

        return orders;
    }

    // Interactive Query - query state store
    @Bean
    public QueryableStoreType<ReadOnlyWindowStore<String, OrderStats>> storeType() {
        return QueryableStoreTypes.windowStore();
    }
}
```

### Interactive Queries API

```java
package com.example.streams;

import org.apache.kafka.streams.KafkaStreams;
import org.apache.kafka.streams.StoreQueryParameters;
import org.apache.kafka.streams.state.*;
import org.springframework.kafka.config.StreamsBuilderFactoryBean;
import org.springframework.web.bind.annotation.*;
import java.time.Instant;

@RestController
@RequestMapping("/api/stats")
public class OrderStatsController {

    private final StreamsBuilderFactoryBean factoryBean;

    @GetMapping("/customer/{customerId}/hourly")
    public OrderStats getHourlyStats(
            @PathVariable String customerId,
            @RequestParam String windowStart) {
        
        KafkaStreams streams = factoryBean.getKafkaStreams();
        
        // Query state store
        ReadOnlyWindowStore<String, OrderStats> store = streams.store(
            StoreQueryParameters.fromNameAndType(
                "hourly-order-stats",
                QueryableStoreTypes.windowStore()
            )
        );

        Instant start = Instant.parse(windowStart);
        Instant end = start.plusSeconds(3600);
        
        // หา stats สำหรับ customer ในช่วงเวลาที่กำหนด
        WindowStoreIterator<OrderStats> iterator = store.fetch(
            customerId, start, end);
        
        if (iterator.hasNext()) {
            KeyValue<Long, OrderStats> kv = iterator.next();
            iterator.close();
            return kv.value;
        }
        
        return null;
    }
}
```

---

## ขั้นตอนที่ 2575: Event Replay and Reprocessing {#event-replay}

### Kafka Consumer Offset Management

```java
package com.example.events;

import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.apache.kafka.common.TopicPartition;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.kafka.listener.ConsumerSeekAware;
import org.springframework.stereotype.Component;
import java.util.Map;

@Component
public class ReplayableEventConsumer implements ConsumerSeekAware {

    private ConsumerSeekCallback seekCallback;

    @Override
    public void registerSeekCallback(ConsumerSeekCallback callback) {
        this.seekCallback = callback;
    }

    @Override
    public void onPartitionsAssigned(Map<TopicPartition, Long> assignments,
                                      ConsumerSeekCallback callback) {
        // ตรวจสอบว่าต้อง replay หรือไม่
        if (shouldReplay()) {
            // Seek ไป timestamp ที่ต้องการ replay
            assignments.forEach((tp, offset) -> {
                callback.seekToTimestamp(tp.topic(), tp.partition(), 
                    getReplayFromTimestamp());
            });
        }
    }

    @KafkaListener(topics = "order-events", groupId = "order-processor")
    public void processEvent(ConsumerRecord<String, OrderCreated> record) {
        log.info("Processing event: offset={}, partition={}", 
            record.offset(), record.partition());
        // process event...
    }

    // Replay events จาก specific timestamp
    public void replayFrom(String topic, long fromTimestamp) {
        seekCallback.seekToTimestamp(topic, 0, fromTimestamp);
        seekCallback.seekToTimestamp(topic, 1, fromTimestamp);
        // ... for each partition
    }
}
```

### Event Replay Service

```java
package com.example.replay;

import org.apache.kafka.clients.admin.AdminClient;
import org.apache.kafka.clients.consumer.KafkaConsumer;
import org.apache.kafka.common.TopicPartition;
import org.springframework.stereotype.Service;
import java.time.Instant;
import java.util.*;

@Service
public class EventReplayService {

    private final KafkaConsumer<String, Object> replayConsumer;
    private final EventProcessor eventProcessor;
    private final AdminClient adminClient;

    public ReplayResult replayEvents(String topic, Instant fromTime, Instant toTime) {
        log.info("Starting event replay for topic={} from={} to={}", 
            topic, fromTime, toTime);
        
        // หา partitions ทั้งหมดของ topic
        List<TopicPartition> partitions = getTopicPartitions(topic);
        
        // กำหนด offset ที่ต้องการเริ่ม
        Map<TopicPartition, Long> timestampsToSearch = new HashMap<>();
        partitions.forEach(tp -> timestampsToSearch.put(tp, fromTime.toEpochMilli()));
        
        // หา offsets สำหรับ timestamp ที่กำหนด
        Map<TopicPartition, OffsetAndTimestamp> offsetsForTimes = 
            replayConsumer.offsetsForTimes(timestampsToSearch);
        
        // Seek ไป offset เหล่านั้น
        offsetsForTimes.forEach((tp, offsetAndTimestamp) -> {
            if (offsetAndTimestamp != null) {
                replayConsumer.seek(tp, offsetAndTimestamp.offset());
            }
        });
        
        // Process events จนถึง toTime
        int processedCount = 0;
        int errorCount = 0;
        boolean done = false;
        
        while (!done) {
            var records = replayConsumer.poll(Duration.ofSeconds(1));
            
            if (records.isEmpty()) break;
            
            for (var record : records) {
                // หยุดเมื่อถึงเวลาที่กำหนด
                if (record.timestamp() > toTime.toEpochMilli()) {
                    done = true;
                    break;
                }
                
                try {
                    eventProcessor.reprocess(record.value(), record.offset(), record.partition());
                    processedCount++;
                } catch (Exception e) {
                    log.error("Failed to replay event at offset={}", record.offset(), e);
                    errorCount++;
                }
            }
        }
        
        return new ReplayResult(processedCount, errorCount, fromTime, toTime);
    }

    private List<TopicPartition> getTopicPartitions(String topic) {
        return replayConsumer.partitionsFor(topic).stream()
            .map(pi -> new TopicPartition(pi.topic(), pi.partition()))
            .collect(Collectors.toList());
    }
}
```

### Idempotent Event Processing

```java
package com.example.events;

import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.stereotype.Component;
import java.time.Duration;

@Component
public class IdempotentEventProcessor {

    private final StringRedisTemplate redisTemplate;
    private final OrderService orderService;

    // ใช้ Redis เก็บ event IDs ที่ process แล้ว
    public void processOrderCreated(OrderCreated event) {
        String eventId = event.getEventId().toString();
        String redisKey = "processed-events:" + eventId;
        
        // Check ว่า event ถูก process แล้วหรือยัง
        Boolean isNew = redisTemplate.opsForValue()
            .setIfAbsent(redisKey, "1", Duration.ofDays(7));
        
        if (Boolean.FALSE.equals(isNew)) {
            log.warn("Duplicate event detected: eventId={}, skipping", eventId);
            return;  // Skip duplicate
        }
        
        try {
            // Process event
            orderService.handleOrderCreated(event);
            log.info("Successfully processed event: eventId={}", eventId);
        } catch (Exception e) {
            // ลบ Redis key เพื่อให้ retry ได้
            redisTemplate.delete(redisKey);
            throw e;
        }
    }
}
```

---

## ขั้นตอนที่ 2580: Dead Letter Queue Handling {#dlq}

### DLQ Configuration

```java
package com.example.config;

import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.kafka.listener.DeadLetterPublishingRecoverer;
import org.springframework.kafka.listener.DefaultErrorHandler;
import org.springframework.util.backoff.FixedBackOff;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class KafkaErrorHandlingConfig {

    @Bean
    public DefaultErrorHandler errorHandler(
            KafkaTemplate<Object, Object> template) {
        
        // DLQ publisher
        DeadLetterPublishingRecoverer recoverer = new DeadLetterPublishingRecoverer(
            template,
            (record, exception) -> {
                // ส่งไปยัง topic ชื่อเดิม + ".DLT"
                return new org.apache.kafka.common.TopicPartition(
                    record.topic() + ".DLT",
                    record.partition()
                );
            }
        );

        // Retry 3 ครั้ง ก่อนส่งไป DLQ
        DefaultErrorHandler errorHandler = new DefaultErrorHandler(
            recoverer,
            new FixedBackOff(1000L, 3)  // retry 3 ครั้ง ห่างกัน 1 วินาที
        );

        // Exception ประเภทไหนที่ไม่ต้อง retry
        errorHandler.addNotRetryableExceptions(
            IllegalArgumentException.class,
            NullPointerException.class
        );

        return errorHandler;
    }
}
```

### DLQ Consumer และ Reprocessing

```java
package com.example.dlq;

import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.kafka.support.KafkaHeaders;
import org.springframework.messaging.handler.annotation.Header;
import org.springframework.stereotype.Component;

@Component
public class DeadLetterQueueConsumer {

    private final DlqRepository dlqRepository;
    private final AlertService alertService;
    private final ManualReprocessingService reprocessingService;

    // Monitor DLQ
    @KafkaListener(
        topics = "order-events.DLT",
        groupId = "dlq-monitor"
    )
    public void processDLQ(
            ConsumerRecord<String, Object> record,
            @Header(KafkaHeaders.DLT_EXCEPTION_CAUSE_FQCN) String exceptionClass,
            @Header(KafkaHeaders.DLT_EXCEPTION_MESSAGE) String errorMessage,
            @Header(KafkaHeaders.DLT_ORIGINAL_TOPIC) String originalTopic,
            @Header(KafkaHeaders.DLT_ORIGINAL_PARTITION) int originalPartition,
            @Header(KafkaHeaders.DLT_ORIGINAL_OFFSET) long originalOffset) {

        log.error("Message in DLQ: topic={}, partition={}, offset={}, error={}",
            originalTopic, originalPartition, originalOffset, errorMessage);

        // บันทึกลง database เพื่อ tracking
        DlqEntry entry = DlqEntry.builder()
            .eventKey(record.key())
            .eventPayload(record.value().toString())
            .originalTopic(originalTopic)
            .originalPartition(originalPartition)
            .originalOffset(originalOffset)
            .exceptionClass(exceptionClass)
            .errorMessage(errorMessage)
            .receivedAt(Instant.now())
            .status(DlqStatus.PENDING)
            .build();
        
        dlqRepository.save(entry);

        // ส่ง alert
        alertService.sendAlert(Alert.builder()
            .severity("WARNING")
            .message("Message sent to DLQ: " + originalTopic)
            .details(errorMessage)
            .build());
    }
}
```

### DLQ Management API

```java
package com.example.dlq;

import org.springframework.web.bind.annotation.*;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;

@RestController
@RequestMapping("/api/admin/dlq")
@PreAuthorize("hasRole('ADMIN')")
public class DlqManagementController {

    private final DlqRepository dlqRepository;
    private final ManualReprocessingService reprocessingService;
    private final KafkaTemplate<String, Object> kafkaTemplate;

    // ดู DLQ entries
    @GetMapping
    public Page<DlqEntry> getDlqEntries(
            @RequestParam(required = false) DlqStatus status,
            Pageable pageable) {
        if (status != null) {
            return dlqRepository.findByStatus(status, pageable);
        }
        return dlqRepository.findAll(pageable);
    }

    // Replay event หนึ่งรายการ
    @PostMapping("/{id}/replay")
    public ResponseEntity<Map<String, Object>> replayEvent(@PathVariable Long id) {
        DlqEntry entry = dlqRepository.findById(id)
            .orElseThrow(() -> new NotFoundException("DLQ entry not found: " + id));

        try {
            // ส่ง message กลับไปยัง original topic
            kafkaTemplate.send(
                entry.getOriginalTopic(),
                entry.getEventKey(),
                entry.getEventPayload()
            );
            
            entry.setStatus(DlqStatus.REPLAYED);
            entry.setReplayedAt(Instant.now());
            dlqRepository.save(entry);
            
            return ResponseEntity.ok(Map.of(
                "status", "REPLAYED",
                "entryId", id,
                "topic", entry.getOriginalTopic()
            ));
        } catch (Exception e) {
            return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
                .body(Map.of("error", e.getMessage()));
        }
    }

    // Replay ทั้งหมดที่ pending
    @PostMapping("/replay-all")
    public ResponseEntity<Map<String, Object>> replayAll() {
        List<DlqEntry> pendingEntries = dlqRepository.findByStatus(DlqStatus.PENDING);
        
        int replayed = 0;
        int failed = 0;
        
        for (DlqEntry entry : pendingEntries) {
            try {
                kafkaTemplate.send(
                    entry.getOriginalTopic(),
                    entry.getEventKey(),
                    entry.getEventPayload()
                );
                entry.setStatus(DlqStatus.REPLAYED);
                replayed++;
            } catch (Exception e) {
                log.error("Failed to replay DLQ entry {}: {}", entry.getId(), e.getMessage());
                failed++;
            }
        }
        
        dlqRepository.saveAll(pendingEntries);
        
        return ResponseEntity.ok(Map.of(
            "totalEntries", pendingEntries.size(),
            "replayed", replayed,
            "failed", failed
        ));
    }

    // ลบ entries เก่า
    @DeleteMapping("/cleanup")
    public ResponseEntity<Map<String, Object>> cleanup(
            @RequestParam(defaultValue = "30") int daysOld) {
        Instant cutoff = Instant.now().minus(Duration.ofDays(daysOld));
        int deleted = dlqRepository.deleteByReceivedAtBeforeAndStatus(
            cutoff, DlqStatus.REPLAYED);
        
        return ResponseEntity.ok(Map.of("deleted", deleted));
    }
}
```

---

## ขั้นตอนที่ 2585: Exactly-Once Semantics {#exactly-once}

### Exactly-Once ใน Kafka Producer

```java
package com.example.events;

import org.apache.kafka.clients.producer.ProducerConfig;
import org.springframework.kafka.core.DefaultKafkaProducerFactory;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import java.util.Map;
import java.util.HashMap;

@Configuration
public class ExactlyOnceConfig {

    @Bean
    public ProducerFactory<String, Object> exactlyOnceProducerFactory() {
        Map<String, Object> config = new HashMap<>();
        config.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "kafka:9092");
        config.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        config.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, JsonSerializer.class);
        
        // Exactly-once settings
        config.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);
        config.put(ProducerConfig.ACKS_CONFIG, "all");
        config.put(ProducerConfig.RETRIES_CONFIG, Integer.MAX_VALUE);
        config.put(ProducerConfig.MAX_IN_FLIGHT_REQUESTS_PER_CONNECTION, 1);
        config.put(ProducerConfig.TRANSACTIONAL_ID_CONFIG, "order-producer-1");
        
        DefaultKafkaProducerFactory<String, Object> factory = 
            new DefaultKafkaProducerFactory<>(config);
        factory.setTransactionIdPrefix("order-tx-");
        
        return factory;
    }
}
```

### Transactional Event Publishing

```java
package com.example.events;

import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.transaction.annotation.Transactional;
import org.springframework.stereotype.Service;

@Service
public class TransactionalOrderService {

    private final OrderRepository orderRepository;
    private final KafkaTemplate<String, Object> kafkaTemplate;

    // @Transactional จะ coordinate ทั้ง DB transaction และ Kafka transaction
    @Transactional
    public Order createOrderWithEvent(CreateOrderRequest request) {
        // 1. Save to database
        Order order = orderRepository.save(Order.builder()
            .customerId(request.getCustomerId())
            .status(OrderStatus.PENDING)
            .build());

        // 2. Publish event - ถ้า exception เกิด จะ rollback ทั้งคู่
        OrderCreated event = buildEvent(order);
        kafkaTemplate.send("order-events", order.getId().toString(), event);

        // ถ้า exception เกิดที่นี่:
        // - Database transaction จะ rollback
        // - Kafka transaction จะ abort
        // → ไม่มี partial state

        return order;
    }

    // Outbox Pattern สำหรับ Exactly-Once ที่ reliable กว่า
    @Transactional
    public Order createOrderWithOutbox(CreateOrderRequest request) {
        // 1. Save order
        Order order = orderRepository.save(Order.builder()
            .customerId(request.getCustomerId())
            .status(OrderStatus.PENDING)
            .build());

        // 2. Save event ลง outbox table (ใน same transaction)
        OutboxEvent outboxEvent = OutboxEvent.builder()
            .aggregateId(order.getId().toString())
            .aggregateType("Order")
            .eventType("OrderCreated")
            .payload(serializeEvent(buildEvent(order)))
            .status(OutboxStatus.PENDING)
            .createdAt(Instant.now())
            .build();
        
        outboxEventRepository.save(outboxEvent);
        
        // Polling publisher จะ poll outbox table และส่งไป Kafka
        return order;
    }
}
```

### Outbox Pattern Publisher

```java
package com.example.outbox;

import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Component;
import org.springframework.transaction.annotation.Transactional;
import java.util.List;

@Component
public class OutboxEventPublisher {

    private final OutboxEventRepository outboxRepository;
    private final KafkaTemplate<String, Object> kafkaTemplate;

    @Scheduled(fixedDelay = 1000) // Poll ทุก 1 วินาที
    @Transactional
    public void publishPendingEvents() {
        List<OutboxEvent> pendingEvents = outboxRepository
            .findByStatusOrderByCreatedAtAsc(OutboxStatus.PENDING, 
                org.springframework.data.domain.PageRequest.of(0, 100));
        
        for (OutboxEvent event : pendingEvents) {
            try {
                // ส่ง event ไป Kafka
                kafkaTemplate.send(
                    getTopicForEventType(event.getEventType()),
                    event.getAggregateId(),
                    deserializePayload(event.getPayload())
                ).get(5, TimeUnit.SECONDS); // รอ confirm
                
                // Mark as published
                event.setStatus(OutboxStatus.PUBLISHED);
                event.setPublishedAt(Instant.now());
                outboxRepository.save(event);
                
            } catch (Exception e) {
                log.error("Failed to publish outbox event: {}", event.getId(), e);
                event.setRetryCount(event.getRetryCount() + 1);
                event.setLastError(e.getMessage());
                
                if (event.getRetryCount() >= 5) {
                    event.setStatus(OutboxStatus.FAILED);
                }
                
                outboxRepository.save(event);
            }
        }
    }

    private String getTopicForEventType(String eventType) {
        return switch (eventType) {
            case "OrderCreated" -> "order-events";
            case "OrderShipped" -> "shipment-events";
            case "PaymentProcessed" -> "payment-events";
            default -> "general-events";
        };
    }
}
```

### Kafka Streams Exactly-Once

```java
package com.example.streams;

import org.apache.kafka.streams.StreamsConfig;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import java.util.Properties;

@Configuration
public class ExactlyOnceStreamConfig {

    @Bean(name = KafkaStreamsDefaultConfiguration.DEFAULT_STREAMS_CONFIG_BEAN_NAME)
    public KafkaStreamsConfiguration streamsConfig() {
        Map<String, Object> props = new HashMap<>();
        props.put(StreamsConfig.APPLICATION_ID_CONFIG, "order-processor");
        props.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, "kafka:9092");
        
        // Exactly-once processing guarantee
        props.put(StreamsConfig.PROCESSING_GUARANTEE_CONFIG, 
            StreamsConfig.EXACTLY_ONCE_V2);  // ใช้ V2 สำหรับ Kafka 2.5+
        
        // Commit interval
        props.put(StreamsConfig.COMMIT_INTERVAL_MS_CONFIG, 100);
        
        // Replication factor
        props.put(StreamsConfig.REPLICATION_FACTOR_CONFIG, 3);
        
        return new KafkaStreamsConfiguration(props);
    }
}
```

---

## ขั้นตอนที่ 2590: Event Schema Registry Best Practices

### Schema Compatibility Rules

```java
package com.example.schema;

import io.confluent.kafka.schemaregistry.client.SchemaRegistryClient;
import io.confluent.kafka.schemaregistry.client.rest.exceptions.RestClientException;
import org.springframework.stereotype.Service;

@Service
public class SchemaRegistryService {

    private final SchemaRegistryClient schemaRegistryClient;

    public void setCompatibilityMode(String subject, CompatibilityMode mode) {
        try {
            schemaRegistryClient.updateCompatibility(subject, mode.name());
            log.info("Set compatibility for {} to {}", subject, mode);
        } catch (RestClientException | IOException e) {
            throw new SchemaRegistryException("Failed to set compatibility", e);
        }
    }

    public boolean isCompatible(String subject, String newSchema) {
        try {
            return schemaRegistryClient.testCompatibility(
                subject, 
                new org.apache.avro.Schema.Parser().parse(newSchema)
            );
        } catch (Exception e) {
            log.error("Schema compatibility check failed", e);
            return false;
        }
    }

    // ตัวอย่าง compatibility modes
    public enum CompatibilityMode {
        NONE,               // ไม่ check
        BACKWARD,           // consumers ใหม่ อ่าน data เก่าได้ (add optional fields)
        FORWARD,            // consumers เก่า อ่าน data ใหม่ได้ (remove optional fields)
        FULL,               // ทั้ง backward และ forward
        BACKWARD_TRANSITIVE,// backward ข้ามทุก version
        FORWARD_TRANSITIVE, // forward ข้ามทุก version
        FULL_TRANSITIVE     // full ข้ามทุก version
    }
}
```

---

## สรุปสิ่งที่เรียนรู้

ใน Part นี้เราได้เรียนรู้:

1. **Event Versioning** - 3 strategies สำหรับ evolve events โดยไม่ break consumers
2. **Apache Avro** - Schema definition, code generation และ Schema Registry
3. **Kafka Streams** - Real-time processing, windowing, joins และ interactive queries
4. **Event Replay** - การ replay events จาก specific timestamp
5. **Dead Letter Queue** - การจัดการ failed events และ manual reprocessing
6. **Exactly-Once** - Idempotent producer, transactions และ Outbox pattern

### Best Practices
- ใช้ Avro + Schema Registry สำหรับ production event schemas
- กำหนด BACKWARD compatibility เป็น minimum
- ใช้ Outbox Pattern แทน dual-write สำหรับ exactly-once
- เก็บ DLQ monitoring dashboard ใน Grafana
- Idempotent consumer ด้วย Redis สำหรับ replay safety

---

*[← Part 73: Zero-Downtime Deployment](./part-73-zero-downtime.md) | [Part 75: Security Advanced →](./part-75-security-advanced.md)*

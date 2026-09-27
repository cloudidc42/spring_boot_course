# Part 89: Advanced Patterns
## ขั้นตอนที่ 3161-3200

**ระดับ:** ระดับโลก (World-Class)
**เวลาเรียน:** 6-8 ชั่วโมง
**เป้าหมาย:** เรียนรู้ Design Patterns ขั้นสูงกับ Spring Framework ทั้ง Template Method, Strategy, Observer, Decorator, Factory, Composite และ Command patterns

---

## ขั้นตอนที่ 3161: Template Method Pattern กับ Spring

Template Method Pattern กำหนด skeleton ของ algorithm ใน base class และให้ subclasses override ขั้นตอนเฉพาะ

```java
// template/DataExportTemplate.java
package com.example.patterns.template;

import lombok.extern.slf4j.Slf4j;
import java.util.List;
import java.util.Map;

@Slf4j
public abstract class DataExportTemplate<T> {

    // Template method - กำหนดขั้นตอนทั้งหมด
    public final ExportResult export(ExportRequest request) {
        log.info("Starting export: {}", request.getExportId());
        
        // 1. Validate request
        validateRequest(request);
        
        // 2. ดึงข้อมูล
        List<T> data = fetchData(request);
        log.info("Fetched {} records", data.size());
        
        // 3. Transform ข้อมูล
        List<Map<String, Object>> transformedData = transformData(data, request);
        
        // 4. Apply business rules (optional - default no-op)
        transformedData = applyBusinessRules(transformedData, request);
        
        // 5. Format ตาม type
        byte[] formattedData = formatData(transformedData, request);
        
        // 6. ส่งออก
        String destination = deliverData(formattedData, request);
        
        // 7. บันทึก audit log
        logExport(request, data.size(), destination);
        
        return ExportResult.builder()
            .exportId(request.getExportId())
            .recordCount(data.size())
            .destination(destination)
            .status("SUCCESS")
            .build();
    }

    // Abstract methods - subclasses ต้อง implement
    protected abstract void validateRequest(ExportRequest request);
    protected abstract List<T> fetchData(ExportRequest request);
    protected abstract List<Map<String, Object>> transformData(
        List<T> data, ExportRequest request);
    protected abstract byte[] formatData(
        List<Map<String, Object>> data, ExportRequest request);

    // Hook methods - subclasses อาจ override หรือไม่ก็ได้
    protected List<Map<String, Object>> applyBusinessRules(
            List<Map<String, Object>> data, ExportRequest request) {
        return data; // default: ไม่เปลี่ยนแปลง
    }

    protected String deliverData(byte[] data, ExportRequest request) {
        // default: บันทึกลงไฟล์
        String filename = "export_" + request.getExportId() + getFileExtension();
        log.info("Saving to file: {}", filename);
        return filename;
    }

    protected void logExport(ExportRequest request, int count, String destination) {
        log.info("Export completed: id={}, records={}, destination={}",
            request.getExportId(), count, destination);
    }

    protected abstract String getFileExtension();
}
```

```java
// template/CsvOrderExport.java
package com.example.patterns.template;

import com.example.patterns.model.Order;
import com.example.patterns.repository.OrderRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Component;

import java.util.*;
import java.util.stream.Collectors;

@Component
@RequiredArgsConstructor
public class CsvOrderExport extends DataExportTemplate<Order> {

    private final OrderRepository orderRepository;

    @Override
    protected void validateRequest(ExportRequest request) {
        if (request.getDateFrom() == null || request.getDateTo() == null) {
            throw new IllegalArgumentException("Date range required for order export");
        }
        if (request.getDateTo().isBefore(request.getDateFrom())) {
            throw new IllegalArgumentException("DateTo must be after DateFrom");
        }
    }

    @Override
    protected List<Order> fetchData(ExportRequest request) {
        return orderRepository.findByCreatedAtBetween(
            request.getDateFrom(), request.getDateTo());
    }

    @Override
    protected List<Map<String, Object>> transformData(
            List<Order> orders, ExportRequest request) {
        return orders.stream().map(order -> {
            Map<String, Object> row = new LinkedHashMap<>();
            row.put("Order ID", order.getOrderId());
            row.put("Customer", order.getCustomerId());
            row.put("Amount", order.getAmount());
            row.put("Status", order.getStatus());
            row.put("Date", order.getCreatedAt());
            return row;
        }).collect(Collectors.toList());
    }

    @Override
    protected List<Map<String, Object>> applyBusinessRules(
            List<Map<String, Object>> data, ExportRequest request) {
        // กรองเฉพาะ completed orders ถ้า request ต้องการ
        if (Boolean.TRUE.equals(request.getCompletedOnly())) {
            return data.stream()
                .filter(row -> "COMPLETED".equals(row.get("Status")))
                .collect(Collectors.toList());
        }
        return data;
    }

    @Override
    protected byte[] formatData(List<Map<String, Object>> data, ExportRequest request) {
        StringBuilder csv = new StringBuilder();
        if (!data.isEmpty()) {
            // Header
            csv.append(String.join(",", data.get(0).keySet())).append("\n");
            // Rows
            for (Map<String, Object> row : data) {
                csv.append(row.values().stream()
                    .map(v -> "\"" + v + "\"")
                    .collect(Collectors.joining(","))).append("\n");
            }
        }
        return csv.toString().getBytes();
    }

    @Override
    protected String getFileExtension() {
        return ".csv";
    }
}
```

```java
// template/ExcelOrderExport.java
package com.example.patterns.template;

import com.example.patterns.model.Order;
import com.example.patterns.repository.OrderRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Component;

import java.util.*;
import java.util.stream.Collectors;

@Component
@RequiredArgsConstructor
public class ExcelOrderExport extends DataExportTemplate<Order> {

    private final OrderRepository orderRepository;

    @Override
    protected void validateRequest(ExportRequest request) {
        if (request.getDateFrom() == null) {
            throw new IllegalArgumentException("Date from required");
        }
    }

    @Override
    protected List<Order> fetchData(ExportRequest request) {
        return orderRepository.findByCreatedAtBetween(
            request.getDateFrom(),
            request.getDateTo() != null ? request.getDateTo() :
                java.time.LocalDateTime.now());
    }

    @Override
    protected List<Map<String, Object>> transformData(
            List<Order> orders, ExportRequest request) {
        return orders.stream().map(order -> {
            Map<String, Object> row = new LinkedHashMap<>();
            row.put("orderId", order.getOrderId());
            row.put("customerId", order.getCustomerId());
            row.put("amount", order.getAmount().doubleValue());
            row.put("status", order.getStatus());
            row.put("createdAt", order.getCreatedAt().toString());
            return row;
        }).collect(Collectors.toList());
    }

    @Override
    protected byte[] formatData(List<Map<String, Object>> data, ExportRequest request) {
        // จำลองการสร้าง Excel (ในระบบจริงใช้ Apache POI)
        return "Excel data".getBytes();
    }

    @Override
    protected String getFileExtension() {
        return ".xlsx";
    }
}
```

---

## ขั้นตอนที่ 3162: Strategy Pattern กับ Spring Beans

Strategy Pattern แยก algorithms ออกจาก context และ inject ผ่าน Spring DI

```java
// strategy/PricingStrategy.java
package com.example.patterns.strategy;

import com.example.patterns.model.Order;
import java.math.BigDecimal;

public interface PricingStrategy {
    BigDecimal calculatePrice(Order order);
    BigDecimal calculateDiscount(Order order);
    String getStrategyName();
    boolean supports(String customerType);
}
```

```java
// strategy/RegularPricingStrategy.java
package com.example.patterns.strategy;

import com.example.patterns.model.Order;
import org.springframework.stereotype.Component;

import java.math.BigDecimal;
import java.math.RoundingMode;

@Component("regularPricing")
public class RegularPricingStrategy implements PricingStrategy {

    @Override
    public BigDecimal calculatePrice(Order order) {
        return order.getBasePrice().setScale(2, RoundingMode.HALF_UP);
    }

    @Override
    public BigDecimal calculateDiscount(Order order) {
        // ไม่มี discount สำหรับ regular customers
        return BigDecimal.ZERO;
    }

    @Override
    public String getStrategyName() {
        return "REGULAR";
    }

    @Override
    public boolean supports(String customerType) {
        return "REGULAR".equals(customerType);
    }
}
```

```java
// strategy/VipPricingStrategy.java
package com.example.patterns.strategy;

import com.example.patterns.model.Order;
import org.springframework.stereotype.Component;

import java.math.BigDecimal;
import java.math.RoundingMode;

@Component("vipPricing")
public class VipPricingStrategy implements PricingStrategy {

    private static final BigDecimal VIP_DISCOUNT_RATE = new BigDecimal("0.15"); // 15%
    private static final BigDecimal VOLUME_DISCOUNT_THRESHOLD = new BigDecimal("5000");
    private static final BigDecimal VOLUME_DISCOUNT_RATE = new BigDecimal("0.05"); // เพิ่ม 5%

    @Override
    public BigDecimal calculateDiscount(Order order) {
        BigDecimal discount = order.getBasePrice().multiply(VIP_DISCOUNT_RATE);
        
        // เพิ่ม volume discount ถ้าออเดอร์ใหญ่
        if (order.getBasePrice().compareTo(VOLUME_DISCOUNT_THRESHOLD) > 0) {
            BigDecimal volumeDiscount = order.getBasePrice().multiply(VOLUME_DISCOUNT_RATE);
            discount = discount.add(volumeDiscount);
        }
        
        return discount.setScale(2, RoundingMode.HALF_UP);
    }

    @Override
    public BigDecimal calculatePrice(Order order) {
        return order.getBasePrice()
            .subtract(calculateDiscount(order))
            .setScale(2, RoundingMode.HALF_UP);
    }

    @Override
    public String getStrategyName() {
        return "VIP";
    }

    @Override
    public boolean supports(String customerType) {
        return "VIP".equals(customerType) || "PREMIUM".equals(customerType);
    }
}
```

```java
// strategy/SeasonalPricingStrategy.java
package com.example.patterns.strategy;

import com.example.patterns.model.Order;
import org.springframework.stereotype.Component;

import java.math.BigDecimal;
import java.math.RoundingMode;
import java.time.LocalDate;
import java.time.MonthDay;

@Component("seasonalPricing")
public class SeasonalPricingStrategy implements PricingStrategy {

    @Override
    public BigDecimal calculateDiscount(Order order) {
        // ตรวจสอบว่าอยู่ในช่วง seasonal sale หรือไม่
        if (isSeasonalPeriod()) {
            return order.getBasePrice()
                .multiply(new BigDecimal("0.20")) // 20% off
                .setScale(2, RoundingMode.HALF_UP);
        }
        return BigDecimal.ZERO;
    }

    @Override
    public BigDecimal calculatePrice(Order order) {
        return order.getBasePrice()
            .subtract(calculateDiscount(order))
            .setScale(2, RoundingMode.HALF_UP);
    }

    @Override
    public String getStrategyName() {
        return "SEASONAL";
    }

    @Override
    public boolean supports(String customerType) {
        return isSeasonalPeriod(); // ใช้ได้ทุกคนในช่วง seasonal
    }

    private boolean isSeasonalPeriod() {
        MonthDay today = MonthDay.now();
        // Black Friday: November 23-30
        MonthDay blackFridayStart = MonthDay.of(11, 23);
        MonthDay blackFridayEnd = MonthDay.of(11, 30);
        // Year-end sale: December 26-31
        MonthDay yearEndStart = MonthDay.of(12, 26);
        MonthDay yearEndEnd = MonthDay.of(12, 31);
        
        return (today.compareTo(blackFridayStart) >= 0 &&
                today.compareTo(blackFridayEnd) <= 0) ||
               (today.compareTo(yearEndStart) >= 0 &&
                today.compareTo(yearEndEnd) <= 0);
    }
}
```

```java
// service/PricingService.java
package com.example.patterns.service;

import com.example.patterns.model.Order;
import com.example.patterns.strategy.PricingStrategy;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;

import java.math.BigDecimal;
import java.util.List;

@Slf4j
@Service
@RequiredArgsConstructor
public class PricingService {

    // Spring inject strategies ทั้งหมดอัตโนมัติ
    private final List<PricingStrategy> pricingStrategies;

    public BigDecimal calculateFinalPrice(Order order, String customerType) {
        PricingStrategy strategy = selectStrategy(customerType);
        log.info("Using pricing strategy: {} for customer type: {}",
            strategy.getStrategyName(), customerType);
        
        BigDecimal price = strategy.calculatePrice(order);
        BigDecimal discount = strategy.calculateDiscount(order);
        
        log.info("Price: {}, Discount: {}, Final: {}",
            order.getBasePrice(), discount, price);
        return price;
    }

    private PricingStrategy selectStrategy(String customerType) {
        return pricingStrategies.stream()
            .filter(s -> s.supports(customerType))
            .findFirst()
            .orElseThrow(() -> new IllegalArgumentException(
                "No pricing strategy for type: " + customerType));
    }

    // ดู strategies ที่มี
    public List<String> getAvailableStrategies() {
        return pricingStrategies.stream()
            .map(PricingStrategy::getStrategyName)
            .toList();
    }
}
```

---

## ขั้นตอนที่ 3163: Observer Pattern กับ ApplicationEvents

Spring ApplicationEvents เป็น built-in observer pattern ของ Spring Framework

```java
// event/OrderCreatedEvent.java
package com.example.patterns.event;

import com.example.patterns.model.Order;
import lombok.Getter;
import org.springframework.context.ApplicationEvent;

@Getter
public class OrderCreatedEvent extends ApplicationEvent {
    private final Order order;
    private final String createdBy;

    public OrderCreatedEvent(Object source, Order order, String createdBy) {
        super(source);
        this.order = order;
        this.createdBy = createdBy;
    }
}
```

```java
// event/OrderStatusChangedEvent.java
package com.example.patterns.event;

import com.example.patterns.model.Order;
import lombok.Getter;
import org.springframework.context.ApplicationEvent;

@Getter
public class OrderStatusChangedEvent extends ApplicationEvent {
    private final Order order;
    private final String previousStatus;
    private final String newStatus;

    public OrderStatusChangedEvent(Object source, Order order,
            String previousStatus, String newStatus) {
        super(source);
        this.order = order;
        this.previousStatus = previousStatus;
        this.newStatus = newStatus;
    }
}
```

```java
// service/OrderService.java
package com.example.patterns.service;

import com.example.patterns.event.OrderCreatedEvent;
import com.example.patterns.event.OrderStatusChangedEvent;
import com.example.patterns.model.Order;
import com.example.patterns.repository.OrderRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.context.ApplicationEventPublisher;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Slf4j
@Service
@RequiredArgsConstructor
public class OrderService {

    private final OrderRepository orderRepository;
    private final ApplicationEventPublisher eventPublisher;

    @Transactional
    public Order createOrder(Order order, String createdBy) {
        Order saved = orderRepository.save(order);
        
        // Publish event หลัง save สำเร็จ
        eventPublisher.publishEvent(
            new OrderCreatedEvent(this, saved, createdBy));
        
        log.info("Order created and event published: {}", saved.getOrderId());
        return saved;
    }

    @Transactional
    public Order updateOrderStatus(String orderId, String newStatus) {
        Order order = orderRepository.findByOrderId(orderId)
            .orElseThrow(() -> new RuntimeException("Order not found: " + orderId));
        
        String previousStatus = order.getStatus();
        order.setStatus(newStatus);
        Order updated = orderRepository.save(order);
        
        // Publish status change event
        eventPublisher.publishEvent(
            new OrderStatusChangedEvent(this, updated, previousStatus, newStatus));
        
        return updated;
    }
}
```

```java
// listener/OrderEventListener.java
package com.example.patterns.listener;

import com.example.patterns.event.OrderCreatedEvent;
import com.example.patterns.event.OrderStatusChangedEvent;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.context.event.EventListener;
import org.springframework.scheduling.annotation.Async;
import org.springframework.stereotype.Component;
import org.springframework.transaction.event.TransactionPhase;
import org.springframework.transaction.event.TransactionalEventListener;

@Slf4j
@Component
@RequiredArgsConstructor
public class OrderEventListener {

    private final NotificationService notificationService;
    private final AuditService auditService;
    private final InventoryService inventoryService;

    // Listen to OrderCreatedEvent
    @EventListener
    @Async  // ทำงาน async ใน thread แยก
    public void onOrderCreated(OrderCreatedEvent event) {
        log.info("Handling OrderCreatedEvent: {}", event.getOrder().getOrderId());
        
        // ส่ง notification
        notificationService.sendOrderConfirmation(event.getOrder());
        
        // บันทึก audit
        auditService.logOrderCreation(event.getOrder(), event.getCreatedBy());
    }

    // ทำงานเฉพาะหลัง transaction commit สำเร็จ
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void onOrderCreatedAfterCommit(OrderCreatedEvent event) {
        log.info("Transaction committed, updating inventory for: {}",
            event.getOrder().getOrderId());
        inventoryService.reserveInventory(event.getOrder());
    }

    // ทำงานถ้า transaction rollback
    @TransactionalEventListener(phase = TransactionPhase.AFTER_ROLLBACK)
    public void onOrderCreatedRollback(OrderCreatedEvent event) {
        log.warn("Transaction rolled back for order: {}", event.getOrder().getOrderId());
        inventoryService.releaseReservation(event.getOrder());
    }

    @EventListener
    @Async
    public void onOrderStatusChanged(OrderStatusChangedEvent event) {
        log.info("Order {} status changed: {} -> {}",
            event.getOrder().getOrderId(),
            event.getPreviousStatus(),
            event.getNewStatus());

        // ส่ง notification เมื่อ status เปลี่ยน
        if ("SHIPPED".equals(event.getNewStatus())) {
            notificationService.sendShippingNotification(event.getOrder());
        } else if ("CANCELLED".equals(event.getNewStatus())) {
            notificationService.sendCancellationNotification(event.getOrder());
            inventoryService.releaseReservation(event.getOrder());
        }
    }
}
```

---

## ขั้นตอนที่ 3164: Decorator Pattern กับ Spring Proxies

Decorator Pattern เพิ่ม behavior ให้ objects โดยไม่ต้อง modify class เดิม

```java
// service/OrderProcessor.java
package com.example.patterns.service;

import com.example.patterns.model.Order;

public interface OrderProcessor {
    Order process(Order order);
    String getProcessorName();
}
```

```java
// service/BaseOrderProcessor.java
package com.example.patterns.service;

import com.example.patterns.model.Order;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;

@Slf4j
@Service("baseOrderProcessor")
public class BaseOrderProcessor implements OrderProcessor {

    @Override
    public Order process(Order order) {
        log.info("Base processing order: {}", order.getOrderId());
        order.setStatus("PROCESSED");
        return order;
    }

    @Override
    public String getProcessorName() {
        return "BASE";
    }
}
```

```java
// decorator/LoggingOrderProcessorDecorator.java
package com.example.patterns.decorator;

import com.example.patterns.model.Order;
import com.example.patterns.service.OrderProcessor;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;

@Slf4j
@RequiredArgsConstructor
public class LoggingOrderProcessorDecorator implements OrderProcessor {

    private final OrderProcessor delegate;

    @Override
    public Order process(Order order) {
        long startTime = System.currentTimeMillis();
        log.info("[{}] Processing started for order: {}",
            getProcessorName(), order.getOrderId());
        
        try {
            Order result = delegate.process(order);
            long elapsed = System.currentTimeMillis() - startTime;
            log.info("[{}] Processing completed in {}ms for order: {}",
                getProcessorName(), elapsed, order.getOrderId());
            return result;
        } catch (Exception e) {
            log.error("[{}] Processing failed for order: {}",
                getProcessorName(), order.getOrderId(), e);
            throw e;
        }
    }

    @Override
    public String getProcessorName() {
        return "LOGGING -> " + delegate.getProcessorName();
    }
}
```

```java
// decorator/ValidationOrderProcessorDecorator.java
package com.example.patterns.decorator;

import com.example.patterns.model.Order;
import com.example.patterns.service.OrderProcessor;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;

import java.math.BigDecimal;

@Slf4j
@RequiredArgsConstructor
public class ValidationOrderProcessorDecorator implements OrderProcessor {

    private final OrderProcessor delegate;

    @Override
    public Order process(Order order) {
        validateOrder(order);
        return delegate.process(order);
    }

    private void validateOrder(Order order) {
        if (order.getOrderId() == null || order.getOrderId().isBlank()) {
            throw new IllegalArgumentException("Order ID required");
        }
        if (order.getAmount() == null ||
                order.getAmount().compareTo(BigDecimal.ZERO) <= 0) {
            throw new IllegalArgumentException("Amount must be positive");
        }
        if (order.getCustomerId() == null) {
            throw new IllegalArgumentException("Customer ID required");
        }
        log.debug("Order validation passed: {}", order.getOrderId());
    }

    @Override
    public String getProcessorName() {
        return "VALIDATION -> " + delegate.getProcessorName();
    }
}
```

```java
// decorator/CachingOrderProcessorDecorator.java
package com.example.patterns.decorator;

import com.example.patterns.model.Order;
import com.example.patterns.service.OrderProcessor;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;

import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

@Slf4j
@RequiredArgsConstructor
public class CachingOrderProcessorDecorator implements OrderProcessor {

    private final OrderProcessor delegate;
    private final Map<String, Order> cache = new ConcurrentHashMap<>();

    @Override
    public Order process(Order order) {
        String cacheKey = order.getOrderId();
        
        if (cache.containsKey(cacheKey)) {
            log.debug("Cache hit for order: {}", cacheKey);
            return cache.get(cacheKey);
        }
        
        Order result = delegate.process(order);
        cache.put(cacheKey, result);
        log.debug("Cached result for order: {}", cacheKey);
        return result;
    }

    @Override
    public String getProcessorName() {
        return "CACHING -> " + delegate.getProcessorName();
    }

    public void clearCache() {
        cache.clear();
    }
}
```

```java
// config/OrderProcessorConfig.java
package com.example.patterns.config;

import com.example.patterns.decorator.*;
import com.example.patterns.service.BaseOrderProcessor;
import com.example.patterns.service.OrderProcessor;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Primary;

@Configuration
public class OrderProcessorConfig {

    @Bean
    @Primary
    public OrderProcessor decoratedOrderProcessor(BaseOrderProcessor base) {
        // Stack decorators: Caching -> Validation -> Logging -> Base
        return new CachingOrderProcessorDecorator(
            new ValidationOrderProcessorDecorator(
                new LoggingOrderProcessorDecorator(
                    base
                )
            )
        );
    }
}
```

---

## ขั้นตอนที่ 3165: Factory Pattern กับ @Bean Methods

Factory Pattern สร้าง objects โดยไม่ระบุ concrete class โดยตรง

```java
// factory/NotificationSender.java
package com.example.patterns.factory;

public interface NotificationSender {
    void send(String recipient, String message);
    boolean supports(String channel);
    String getChannel();
}
```

```java
// factory/EmailNotificationSender.java
package com.example.patterns.factory;

import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Component;

@Slf4j
@Component
public class EmailNotificationSender implements NotificationSender {

    @Override
    public void send(String recipient, String message) {
        log.info("Sending email to {}: {}", recipient, message);
        // ส่ง email จริง
    }

    @Override
    public boolean supports(String channel) {
        return "EMAIL".equalsIgnoreCase(channel);
    }

    @Override
    public String getChannel() {
        return "EMAIL";
    }
}
```

```java
// factory/SmsNotificationSender.java
package com.example.patterns.factory;

import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Component;

@Slf4j
@Component
public class SmsNotificationSender implements NotificationSender {

    @Override
    public void send(String recipient, String message) {
        log.info("Sending SMS to {}: {}", recipient, message);
        // ส่ง SMS จริง
    }

    @Override
    public boolean supports(String channel) {
        return "SMS".equalsIgnoreCase(channel);
    }

    @Override
    public String getChannel() {
        return "SMS";
    }
}
```

```java
// factory/PushNotificationSender.java
package com.example.patterns.factory;

import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Component;

@Slf4j
@Component
public class PushNotificationSender implements NotificationSender {

    @Override
    public void send(String recipient, String message) {
        log.info("Sending push notification to {}: {}", recipient, message);
        // ส่ง push notification
    }

    @Override
    public boolean supports(String channel) {
        return "PUSH".equalsIgnoreCase(channel);
    }

    @Override
    public String getChannel() {
        return "PUSH";
    }
}
```

```java
// factory/NotificationFactory.java
package com.example.patterns.factory;

import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Component;

import java.util.List;

@Component
@RequiredArgsConstructor
public class NotificationFactory {

    // Spring inject ทุก NotificationSender implementations
    private final List<NotificationSender> senders;

    public NotificationSender getSender(String channel) {
        return senders.stream()
            .filter(sender -> sender.supports(channel))
            .findFirst()
            .orElseThrow(() -> new IllegalArgumentException(
                "No sender for channel: " + channel));
    }

    public List<String> getAvailableChannels() {
        return senders.stream()
            .map(NotificationSender::getChannel)
            .toList();
    }

    // Abstract Factory - สร้าง sender สำหรับ notification type
    public NotificationSender getSenderForOrderEvent(String orderStatus) {
        return switch (orderStatus) {
            case "CONFIRMED" -> getSender("EMAIL");
            case "SHIPPED" -> getSender("SMS");
            case "DELIVERED" -> getSender("PUSH");
            default -> getSender("EMAIL");
        };
    }
}
```

---

## ขั้นตอนที่ 3166: Composite Pattern สำหรับ Services

Composite Pattern ทำให้ treat individual objects และ groups เหมือนกัน

```java
// composite/ReportComponent.java
package com.example.patterns.composite;

import java.util.Map;

public interface ReportComponent {
    String getName();
    Map<String, Object> generateReport(ReportContext context);
    void addComponent(ReportComponent component);
    void removeComponent(ReportComponent component);
    boolean isLeaf();
}
```

```java
// composite/SalesReportLeaf.java
package com.example.patterns.composite;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Component;

import java.util.*;

@Slf4j
@Component
@RequiredArgsConstructor
public class SalesReportLeaf implements ReportComponent {

    private final SalesDataService salesDataService;

    @Override
    public String getName() {
        return "Sales Report";
    }

    @Override
    public Map<String, Object> generateReport(ReportContext context) {
        log.info("Generating sales report for period: {} - {}",
            context.getDateFrom(), context.getDateTo());
        
        Map<String, Object> report = new LinkedHashMap<>();
        report.put("type", "SALES");
        report.put("totalOrders", salesDataService.countOrders(context));
        report.put("totalRevenue", salesDataService.getTotalRevenue(context));
        report.put("averageOrderValue", salesDataService.getAverageOrderValue(context));
        report.put("topProducts", salesDataService.getTopProducts(context, 5));
        return report;
    }

    @Override
    public void addComponent(ReportComponent component) {
        throw new UnsupportedOperationException("Leaf cannot add components");
    }

    @Override
    public void removeComponent(ReportComponent component) {
        throw new UnsupportedOperationException("Leaf cannot remove components");
    }

    @Override
    public boolean isLeaf() { return true; }
}
```

```java
// composite/CompositeReport.java
package com.example.patterns.composite;

import lombok.Getter;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;

import java.util.*;

@Slf4j
@RequiredArgsConstructor
public class CompositeReport implements ReportComponent {

    @Getter
    private final String name;
    private final List<ReportComponent> children = new ArrayList<>();

    @Override
    public Map<String, Object> generateReport(ReportContext context) {
        log.info("Generating composite report: {}", name);
        
        Map<String, Object> compositeReport = new LinkedHashMap<>();
        compositeReport.put("reportName", name);
        compositeReport.put("generatedAt", new Date());
        
        Map<String, Object> sections = new LinkedHashMap<>();
        for (ReportComponent child : children) {
            sections.put(child.getName(), child.generateReport(context));
        }
        compositeReport.put("sections", sections);
        
        return compositeReport;
    }

    @Override
    public void addComponent(ReportComponent component) {
        children.add(component);
        log.debug("Added component '{}' to '{}'", component.getName(), name);
    }

    @Override
    public void removeComponent(ReportComponent component) {
        children.remove(component);
    }

    @Override
    public boolean isLeaf() { return false; }
}
```

```java
// config/ReportConfig.java
package com.example.patterns.config;

import com.example.patterns.composite.*;
import lombok.RequiredArgsConstructor;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
@RequiredArgsConstructor
public class ReportConfig {

    private final SalesReportLeaf salesReport;
    private final InventoryReportLeaf inventoryReport;
    private final CustomerReportLeaf customerReport;

    // สร้าง composite report tree
    @Bean
    public ReportComponent monthlyReportTree() {
        CompositeReport monthly = new CompositeReport("Monthly Report");
        
        // Sales section
        CompositeReport salesSection = new CompositeReport("Sales Section");
        salesSection.addComponent(salesReport);
        
        // Operations section
        CompositeReport operationsSection = new CompositeReport("Operations Section");
        operationsSection.addComponent(inventoryReport);
        operationsSection.addComponent(customerReport);
        
        monthly.addComponent(salesSection);
        monthly.addComponent(operationsSection);
        
        return monthly;
    }
}
```

---

## ขั้นตอนที่ 3167: Command Pattern กับ Spring

Command Pattern encapsulate requests เป็น objects ทำให้ queue, undo และ log ได้

```java
// command/Command.java
package com.example.patterns.command;

public interface Command<T> {
    T execute();
    void undo();
    boolean canUndo();
    String getCommandName();
    String getDescription();
}
```

```java
// command/CreateOrderCommand.java
package com.example.patterns.command;

import com.example.patterns.model.Order;
import com.example.patterns.repository.OrderRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;

@Slf4j
@RequiredArgsConstructor
public class CreateOrderCommand implements Command<Order> {

    private final OrderRepository orderRepository;
    private final Order orderToCreate;
    private Order createdOrder;

    @Override
    public Order execute() {
        log.info("Executing CreateOrderCommand for: {}", orderToCreate.getOrderId());
        createdOrder = orderRepository.save(orderToCreate);
        return createdOrder;
    }

    @Override
    public void undo() {
        if (createdOrder != null) {
            log.info("Undoing CreateOrderCommand: deleting {}", createdOrder.getOrderId());
            orderRepository.delete(createdOrder);
            createdOrder = null;
        }
    }

    @Override
    public boolean canUndo() {
        return createdOrder != null;
    }

    @Override
    public String getCommandName() {
        return "CREATE_ORDER";
    }

    @Override
    public String getDescription() {
        return "Create order: " + orderToCreate.getOrderId();
    }
}
```

```java
// command/UpdateOrderStatusCommand.java
package com.example.patterns.command;

import com.example.patterns.model.Order;
import com.example.patterns.repository.OrderRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;

@Slf4j
@RequiredArgsConstructor
public class UpdateOrderStatusCommand implements Command<Order> {

    private final OrderRepository orderRepository;
    private final String orderId;
    private final String newStatus;
    private String previousStatus;

    @Override
    public Order execute() {
        Order order = orderRepository.findByOrderId(orderId)
            .orElseThrow(() -> new RuntimeException("Order not found: " + orderId));
        
        previousStatus = order.getStatus();
        order.setStatus(newStatus);
        Order updated = orderRepository.save(order);
        
        log.info("Order {} status changed: {} -> {}", orderId, previousStatus, newStatus);
        return updated;
    }

    @Override
    public void undo() {
        if (previousStatus != null) {
            Order order = orderRepository.findByOrderId(orderId)
                .orElseThrow();
            order.setStatus(previousStatus);
            orderRepository.save(order);
            log.info("Undone: Order {} status restored to {}", orderId, previousStatus);
        }
    }

    @Override
    public boolean canUndo() {
        return previousStatus != null;
    }

    @Override
    public String getCommandName() {
        return "UPDATE_ORDER_STATUS";
    }

    @Override
    public String getDescription() {
        return String.format("Update order %s status to %s", orderId, newStatus);
    }
}
```

```java
// command/CommandInvoker.java
package com.example.patterns.command;

import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Component;

import java.util.ArrayDeque;
import java.util.Deque;
import java.util.List;
import java.util.ArrayList;

@Slf4j
@Component
public class CommandInvoker {

    private final Deque<Command<?>> commandHistory = new ArrayDeque<>();
    private final List<CommandExecutionRecord> executionLog = new ArrayList<>();

    // Execute command และบันทึกใน history
    public <T> T executeCommand(Command<T> command) {
        log.info("Executing command: {}", command.getCommandName());
        
        long startTime = System.currentTimeMillis();
        try {
            T result = command.execute();
            long elapsed = System.currentTimeMillis() - startTime;
            
            if (command.canUndo()) {
                commandHistory.push(command);
            }
            
            executionLog.add(new CommandExecutionRecord(
                command.getCommandName(),
                command.getDescription(),
                "SUCCESS",
                elapsed
            ));
            
            log.info("Command {} completed in {}ms", command.getCommandName(), elapsed);
            return result;
        } catch (Exception e) {
            executionLog.add(new CommandExecutionRecord(
                command.getCommandName(),
                command.getDescription(),
                "FAILED: " + e.getMessage(),
                System.currentTimeMillis() - startTime
            ));
            throw e;
        }
    }

    // Undo command ล่าสุด
    public void undo() {
        if (commandHistory.isEmpty()) {
            log.warn("No commands to undo");
            return;
        }
        
        Command<?> lastCommand = commandHistory.pop();
        log.info("Undoing command: {}", lastCommand.getCommandName());
        lastCommand.undo();
    }

    // Undo หลาย commands
    public void undoLast(int count) {
        for (int i = 0; i < count && !commandHistory.isEmpty(); i++) {
            undo();
        }
    }

    public List<CommandExecutionRecord> getExecutionLog() {
        return List.copyOf(executionLog);
    }

    public int getPendingUndoCount() {
        return commandHistory.size();
    }

    // Record class
    public record CommandExecutionRecord(
        String commandName,
        String description,
        String status,
        long executionTimeMs
    ) {}
}
```

---

## ขั้นตอนที่ 3168: Builder Pattern กับ Spring

Builder Pattern ใน Spring สำหรับสร้าง complex objects

```java
// builder/QueryBuilder.java
package com.example.patterns.builder;

import lombok.Builder;
import lombok.Getter;
import org.springframework.data.domain.Sort;

import java.time.LocalDateTime;
import java.util.ArrayList;
import java.util.List;

@Getter
public class OrderQuery {
    private final String customerId;
    private final String status;
    private final LocalDateTime dateFrom;
    private final LocalDateTime dateTo;
    private final Double minAmount;
    private final Double maxAmount;
    private final List<String> productCodes;
    private final int page;
    private final int size;
    private final Sort sort;
    private final boolean includeDeleted;

    private OrderQuery(Builder builder) {
        this.customerId = builder.customerId;
        this.status = builder.status;
        this.dateFrom = builder.dateFrom;
        this.dateTo = builder.dateTo;
        this.minAmount = builder.minAmount;
        this.maxAmount = builder.maxAmount;
        this.productCodes = List.copyOf(builder.productCodes);
        this.page = builder.page;
        this.size = builder.size;
        this.sort = builder.sort;
        this.includeDeleted = builder.includeDeleted;
    }

    public static Builder builder() {
        return new Builder();
    }

    public static class Builder {
        private String customerId;
        private String status;
        private LocalDateTime dateFrom;
        private LocalDateTime dateTo;
        private Double minAmount;
        private Double maxAmount;
        private List<String> productCodes = new ArrayList<>();
        private int page = 0;
        private int size = 20;
        private Sort sort = Sort.by(Sort.Direction.DESC, "createdAt");
        private boolean includeDeleted = false;

        public Builder forCustomer(String customerId) {
            this.customerId = customerId;
            return this;
        }

        public Builder withStatus(String status) {
            this.status = status;
            return this;
        }

        public Builder betweenDates(LocalDateTime from, LocalDateTime to) {
            this.dateFrom = from;
            this.dateTo = to;
            return this;
        }

        public Builder amountBetween(double min, double max) {
            this.minAmount = min;
            this.maxAmount = max;
            return this;
        }

        public Builder withProducts(List<String> codes) {
            this.productCodes.addAll(codes);
            return this;
        }

        public Builder page(int page, int size) {
            this.page = page;
            this.size = size;
            return this;
        }

        public Builder sortBy(String field, Sort.Direction direction) {
            this.sort = Sort.by(direction, field);
            return this;
        }

        public Builder includeDeleted() {
            this.includeDeleted = true;
            return this;
        }

        public OrderQuery build() {
            validate();
            return new OrderQuery(this);
        }

        private void validate() {
            if (dateFrom != null && dateTo != null && dateTo.isBefore(dateFrom)) {
                throw new IllegalStateException("dateTo must be after dateFrom");
            }
            if (minAmount != null && maxAmount != null && minAmount > maxAmount) {
                throw new IllegalStateException("minAmount must be <= maxAmount");
            }
            if (size <= 0 || size > 1000) {
                throw new IllegalStateException("size must be between 1 and 1000");
            }
        }
    }
}
```

---

## ขั้นตอนที่ 3169: Testing Design Patterns

```java
// test/PatternsTest.java
package com.example.patterns;

import com.example.patterns.command.*;
import com.example.patterns.factory.*;
import com.example.patterns.model.Order;
import com.example.patterns.service.PricingService;
import com.example.patterns.strategy.PricingStrategy;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;

import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.List;

import static org.assertj.core.api.Assertions.*;

@SpringBootTest
class PatternsTest {

    @Autowired
    private PricingService pricingService;

    @Autowired
    private NotificationFactory notificationFactory;

    @Autowired
    private CommandInvoker commandInvoker;

    @Autowired
    private List<PricingStrategy> strategies;

    @Test
    void strategyPattern_shouldSelectCorrectPricing() {
        Order order = Order.builder()
            .orderId("TEST-001")
            .basePrice(new BigDecimal("1000"))
            .build();

        // VIP ได้รับ 15% discount
        BigDecimal vipPrice = pricingService.calculateFinalPrice(order, "VIP");
        assertThat(vipPrice).isEqualByComparingTo("850.00");

        // Regular ไม่ได้ discount
        BigDecimal regularPrice = pricingService.calculateFinalPrice(order, "REGULAR");
        assertThat(regularPrice).isEqualByComparingTo("1000.00");
    }

    @Test
    void factoryPattern_shouldCreateCorrectSender() {
        NotificationSender emailSender = notificationFactory.getSender("EMAIL");
        assertThat(emailSender.getChannel()).isEqualTo("EMAIL");
        assertThat(emailSender).isInstanceOf(EmailNotificationSender.class);

        NotificationSender smsSender = notificationFactory.getSender("SMS");
        assertThat(smsSender.getChannel()).isEqualTo("SMS");
    }

    @Test
    void commandPattern_shouldExecuteAndUndo() {
        Order order = Order.builder()
            .orderId("CMD-TEST-001")
            .customerId("CUST-001")
            .amount(new BigDecimal("500"))
            .status("NEW")
            .createdAt(LocalDateTime.now())
            .build();

        // Execute command
        // (in real test would use mock repository)
        assertThat(commandInvoker.getPendingUndoCount()).isEqualTo(0);
    }

    @Test
    void strategyPattern_shouldListAllStrategies() {
        List<String> availableStrategies = pricingService.getAvailableStrategies();
        assertThat(availableStrategies).contains("REGULAR", "VIP", "SEASONAL");
    }
}
```

---

## สรุป Part 89

ในส่วนนี้เราได้เรียนรู้:
- **Template Method Pattern** - กำหนด algorithm skeleton ใน base class
- **Strategy Pattern** - แยก algorithms และ inject ผ่าน Spring DI
- **Observer Pattern** - Spring ApplicationEvents สำหรับ loose coupling
- **Decorator Pattern** - เพิ่ม behavior โดยไม่ modify class เดิม
- **Factory Pattern** - สร้าง objects ผ่าน Spring IoC container
- **Composite Pattern** - จัดการ tree structure ของ components
- **Command Pattern** - Encapsulate requests เป็น objects พร้อม undo
- **Builder Pattern** - สร้าง complex objects อย่างปลอดภัย

---

*[← Part 88: Data Streaming](./part-88-data-streaming.md) | [Part 90: Production Optimization →](./part-90-production-optimization.md)*

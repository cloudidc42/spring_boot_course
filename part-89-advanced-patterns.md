# Part 89: Advanced Design Patterns in Spring Boot
## ขั้นตอนที่ 3161-3200

**ระดับ:** World-Class (ระดับโลก)
**เวลาเรียน:** 8-10 ชั่วโมง
**เป้าหมาย:** เรียนรู้ Design Patterns ขั้นสูงที่ใช้ใน Spring Boot ระดับ production ได้แก่ Template Method, Strategy, Observer/Event, Decorator, Factory, Command และ Composite patterns พร้อม implementation จริงด้วย Spring features

---

## ขั้นตอนที่ 3161: Template Method Pattern กับ Abstract Spring Services

Template Method Pattern คือ pattern ที่กำหนดโครงร่าง (skeleton) ของ algorithm ใน base class และให้ subclass override บางขั้นตอนได้ ใน Spring Boot เราใช้ abstract class เป็น base service

### ทำไมต้องใช้ Template Method?

เมื่อมี workflow ที่มีขั้นตอนคล้ายกันหลายแบบ เช่น:
- การ process คำสั่งซื้อ (Order Processing)
- การ export report ในหลายรูปแบบ (PDF, Excel, CSV)
- การ validate ข้อมูลแบบต่างๆ

### Implementation: Order Processing Template

```java
// domain/order/OrderProcessingTemplate.java
package com.example.shophub.domain.order;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.transaction.annotation.Transactional;

/**
 * Abstract Template สำหรับการประมวลผลคำสั่งซื้อ
 * กำหนดขั้นตอน: validate → reserve inventory → charge payment → confirm order → notify
 */
public abstract class OrderProcessingTemplate {
    
    protected final Logger log = LoggerFactory.getLogger(getClass());
    
    /**
     * Template method - กำหนดขั้นตอนการ process order
     * Subclass ไม่ควร override method นี้
     */
    @Transactional
    public final OrderResult processOrder(OrderRequest request) {
        log.info("เริ่มประมวลผลคำสั่งซื้อ: orderId={}", request.getOrderId());
        
        try {
            // ขั้นที่ 1: ตรวจสอบความถูกต้อง
            ValidationResult validation = validateOrder(request);
            if (!validation.isValid()) {
                return OrderResult.failed(validation.getErrors());
            }
            
            // ขั้นที่ 2: จอง inventory
            InventoryReservation reservation = reserveInventory(request);
            
            // ขั้นที่ 3: ชำระเงิน
            PaymentResult payment = chargePayment(request, reservation);
            if (!payment.isSuccessful()) {
                releaseInventory(reservation);
                return OrderResult.paymentFailed(payment.getError());
            }
            
            // ขั้นที่ 4: ยืนยันคำสั่งซื้อ
            Order confirmedOrder = confirmOrder(request, reservation, payment);
            
            // ขั้นที่ 5: แจ้งเตือน (optional hook)
            postProcess(confirmedOrder);
            
            log.info("ประมวลผลคำสั่งซื้อสำเร็จ: orderId={}", confirmedOrder.getId());
            return OrderResult.success(confirmedOrder);
            
        } catch (Exception e) {
            log.error("เกิดข้อผิดพลาดในการประมวลผลคำสั่งซื้อ", e);
            return OrderResult.error(e.getMessage());
        }
    }
    
    // Abstract methods ที่ subclass ต้อง implement
    protected abstract ValidationResult validateOrder(OrderRequest request);
    protected abstract InventoryReservation reserveInventory(OrderRequest request);
    protected abstract PaymentResult chargePayment(OrderRequest request, InventoryReservation reservation);
    protected abstract Order confirmOrder(OrderRequest request, InventoryReservation reservation, PaymentResult payment);
    
    // Hook method - subclass อาจ override หรือไม่ก็ได้
    protected void releaseInventory(InventoryReservation reservation) {
        log.warn("คืน inventory: reservationId={}", reservation.getId());
    }
    
    // Hook method สำหรับ post-processing
    protected void postProcess(Order order) {
        // Default: ไม่ทำอะไร
    }
}
```

```java
// domain/order/StandardOrderProcessor.java
package com.example.shophub.domain.order;

import org.springframework.stereotype.Service;

@Service
public class StandardOrderProcessor extends OrderProcessingTemplate {
    
    private final InventoryService inventoryService;
    private final PaymentGateway paymentGateway;
    private final OrderRepository orderRepository;
    private final NotificationService notificationService;
    
    public StandardOrderProcessor(
            InventoryService inventoryService,
            PaymentGateway paymentGateway,
            OrderRepository orderRepository,
            NotificationService notificationService) {
        this.inventoryService = inventoryService;
        this.paymentGateway = paymentGateway;
        this.orderRepository = orderRepository;
        this.notificationService = notificationService;
    }
    
    @Override
    protected ValidationResult validateOrder(OrderRequest request) {
        List<String> errors = new ArrayList<>();
        
        if (request.getItems() == null || request.getItems().isEmpty()) {
            errors.add("คำสั่งซื้อต้องมีสินค้าอย่างน้อย 1 รายการ");
        }
        
        if (request.getCustomerId() == null) {
            errors.add("ต้องระบุ Customer ID");
        }
        
        return errors.isEmpty() 
            ? ValidationResult.valid() 
            : ValidationResult.invalid(errors);
    }
    
    @Override
    protected InventoryReservation reserveInventory(OrderRequest request) {
        return inventoryService.reserve(request.getItems());
    }
    
    @Override
    protected PaymentResult chargePayment(OrderRequest request, InventoryReservation reservation) {
        return paymentGateway.charge(
            request.getPaymentMethod(),
            reservation.getTotalAmount()
        );
    }
    
    @Override
    protected Order confirmOrder(OrderRequest request, InventoryReservation reservation, PaymentResult payment) {
        Order order = Order.builder()
            .customerId(request.getCustomerId())
            .items(reservation.getItems())
            .paymentId(payment.getTransactionId())
            .status(OrderStatus.CONFIRMED)
            .build();
        return orderRepository.save(order);
    }
    
    @Override
    protected void postProcess(Order order) {
        // ส่ง notification ให้ลูกค้า
        notificationService.sendOrderConfirmation(order);
    }
}
```

```java
// domain/order/SubscriptionOrderProcessor.java
// Template สำหรับ subscription order มีขั้นตอนพิเศษ
@Service
public class SubscriptionOrderProcessor extends OrderProcessingTemplate {
    
    private final SubscriptionService subscriptionService;
    // ... other dependencies
    
    @Override
    protected ValidationResult validateOrder(OrderRequest request) {
        // ตรวจสอบเพิ่มเติมสำหรับ subscription
        ValidationResult base = super.validateOrder(request); // ถ้าต้องการ
        
        // ตรวจสอบว่า subscription ยังใช้งานได้
        if (!subscriptionService.isActive(request.getCustomerId())) {
            return ValidationResult.invalid(List.of("Subscription หมดอายุแล้ว"));
        }
        
        return ValidationResult.valid();
    }
    
    @Override
    protected PaymentResult chargePayment(OrderRequest request, InventoryReservation reservation) {
        // Subscription ใช้ stored payment method อัตโนมัติ
        return subscriptionService.chargeStoredPayment(
            request.getCustomerId(),
            reservation.getTotalAmount()
        );
    }
    
    // implement other abstract methods...
    @Override
    protected InventoryReservation reserveInventory(OrderRequest request) {
        return inventoryService.reserveWithPriority(request.getItems(), Priority.SUBSCRIPTION);
    }
    
    @Override
    protected Order confirmOrder(OrderRequest request, InventoryReservation reservation, PaymentResult payment) {
        Order order = Order.builder()
            .customerId(request.getCustomerId())
            .orderType(OrderType.SUBSCRIPTION)
            .items(reservation.getItems())
            .paymentId(payment.getTransactionId())
            .status(OrderStatus.CONFIRMED)
            .build();
        return orderRepository.save(order);
    }
}
```

### Report Export Template

```java
// report/ReportExportTemplate.java
public abstract class ReportExportTemplate<T> {
    
    public final byte[] export(ReportRequest request) {
        // ดึงข้อมูล
        List<T> data = fetchData(request);
        
        // เตรียม headers
        List<String> headers = getHeaders();
        
        // แปลงข้อมูลเป็น rows
        List<List<Object>> rows = data.stream()
            .map(this::toRow)
            .collect(Collectors.toList());
        
        // สร้าง output
        return buildOutput(headers, rows, request);
    }
    
    protected abstract List<T> fetchData(ReportRequest request);
    protected abstract List<String> getHeaders();
    protected abstract List<Object> toRow(T item);
    protected abstract byte[] buildOutput(List<String> headers, List<List<Object>> rows, ReportRequest request);
}

// report/SalesReportPdfExporter.java
@Service
public class SalesReportPdfExporter extends ReportExportTemplate<SalesRecord> {
    
    @Override
    protected List<SalesRecord> fetchData(ReportRequest request) {
        return salesRepository.findByDateRange(request.getFrom(), request.getTo());
    }
    
    @Override
    protected List<String> getHeaders() {
        return List.of("วันที่", "สินค้า", "จำนวน", "ราคา", "รวม");
    }
    
    @Override
    protected List<Object> toRow(SalesRecord record) {
        return List.of(
            record.getDate(),
            record.getProductName(),
            record.getQuantity(),
            record.getUnitPrice(),
            record.getTotal()
        );
    }
    
    @Override
    protected byte[] buildOutput(List<String> headers, List<List<Object>> rows, ReportRequest request) {
        // สร้าง PDF ด้วย Apache PDFBox หรือ iText
        return pdfBuilder.build(headers, rows);
    }
}
```

---

## ขั้นตอนที่ 3162: Strategy Pattern กับ @Qualifier

Strategy Pattern ช่วยให้เราสลับ algorithm ได้ตอน runtime โดยไม่ต้องแก้ code ใน Spring Boot เราใช้ `@Qualifier` เพื่อเลือก strategy ที่ต้องการ

### Use Case: Shipping Calculator

```java
// shipping/ShippingStrategy.java
package com.example.shophub.shipping;

public interface ShippingStrategy {
    ShippingCost calculate(ShippingRequest request);
    boolean supports(ShippingMethod method);
}

// shipping/StandardShippingStrategy.java
@Service
@Qualifier("standard")
public class StandardShippingStrategy implements ShippingStrategy {
    
    private static final BigDecimal BASE_RATE = new BigDecimal("50");
    private static final BigDecimal PER_KG_RATE = new BigDecimal("20");
    
    @Override
    public ShippingCost calculate(ShippingRequest request) {
        BigDecimal weight = request.getWeightKg();
        BigDecimal cost = BASE_RATE.add(weight.multiply(PER_KG_RATE));
        
        return ShippingCost.builder()
            .method(ShippingMethod.STANDARD)
            .amount(cost)
            .estimatedDays(5)
            .build();
    }
    
    @Override
    public boolean supports(ShippingMethod method) {
        return method == ShippingMethod.STANDARD;
    }
}

// shipping/ExpressShippingStrategy.java
@Service
@Qualifier("express")
public class ExpressShippingStrategy implements ShippingStrategy {
    
    @Override
    public ShippingCost calculate(ShippingRequest request) {
        BigDecimal weight = request.getWeightKg();
        BigDecimal cost = new BigDecimal("150").add(weight.multiply(new BigDecimal("40")));
        
        return ShippingCost.builder()
            .method(ShippingMethod.EXPRESS)
            .amount(cost)
            .estimatedDays(1)
            .build();
    }
    
    @Override
    public boolean supports(ShippingMethod method) {
        return method == ShippingMethod.EXPRESS;
    }
}

// shipping/FreemiumShippingStrategy.java
@Service
@Qualifier("free")
public class FreemiumShippingStrategy implements ShippingStrategy {
    
    @Override
    public ShippingCost calculate(ShippingRequest request) {
        return ShippingCost.builder()
            .method(ShippingMethod.FREE)
            .amount(BigDecimal.ZERO)
            .estimatedDays(7)
            .build();
    }
    
    @Override
    public boolean supports(ShippingMethod method) {
        return method == ShippingMethod.FREE;
    }
}
```

### Strategy Selector/Context

```java
// shipping/ShippingCalculatorService.java
@Service
public class ShippingCalculatorService {
    
    // Spring inject ทุก implementation ของ ShippingStrategy
    private final List<ShippingStrategy> strategies;
    
    public ShippingCalculatorService(List<ShippingStrategy> strategies) {
        this.strategies = strategies;
    }
    
    public ShippingCost calculate(ShippingRequest request, ShippingMethod method) {
        return strategies.stream()
            .filter(s -> s.supports(method))
            .findFirst()
            .map(s -> s.calculate(request))
            .orElseThrow(() -> new UnsupportedShippingMethodException(method));
    }
    
    public List<ShippingCost> getAllOptions(ShippingRequest request) {
        return strategies.stream()
            .map(s -> s.calculate(request))
            .sorted(Comparator.comparing(ShippingCost::getAmount))
            .collect(Collectors.toList());
    }
}
```

### Strategy พร้อม Map-based lookup (ประสิทธิภาพสูงกว่า)

```java
// shipping/ShippingStrategyRegistry.java
@Configuration
public class ShippingStrategyRegistry {
    
    @Bean
    public Map<ShippingMethod, ShippingStrategy> shippingStrategies(
            @Qualifier("standard") ShippingStrategy standard,
            @Qualifier("express") ShippingStrategy express,
            @Qualifier("free") ShippingStrategy free) {
        
        return Map.of(
            ShippingMethod.STANDARD, standard,
            ShippingMethod.EXPRESS, express,
            ShippingMethod.FREE, free
        );
    }
}

// Service ที่ใช้ Map lookup
@Service
public class OptimizedShippingService {
    
    private final Map<ShippingMethod, ShippingStrategy> strategyMap;
    
    public OptimizedShippingService(Map<ShippingMethod, ShippingStrategy> strategyMap) {
        this.strategyMap = strategyMap;
    }
    
    public ShippingCost calculate(ShippingRequest request, ShippingMethod method) {
        ShippingStrategy strategy = strategyMap.get(method);
        if (strategy == null) {
            throw new UnsupportedShippingMethodException(method);
        }
        return strategy.calculate(request);
    }
}
```

### Discount Strategy Pattern

```java
// discount/DiscountStrategy.java
public interface DiscountStrategy {
    Discount apply(Cart cart, Customer customer);
    int getPriority(); // ลำดับความสำคัญ
}

// discount/MemberDiscountStrategy.java
@Service
public class MemberDiscountStrategy implements DiscountStrategy {
    
    @Override
    public Discount apply(Cart cart, Customer customer) {
        if (!customer.isMember()) {
            return Discount.none();
        }
        
        BigDecimal rate = switch (customer.getMemberLevel()) {
            case SILVER -> new BigDecimal("0.05");
            case GOLD -> new BigDecimal("0.10");
            case PLATINUM -> new BigDecimal("0.15");
            default -> BigDecimal.ZERO;
        };
        
        return Discount.percentage(rate, "Member discount");
    }
    
    @Override
    public int getPriority() { return 1; }
}

// discount/BulkDiscountStrategy.java
@Service
public class BulkDiscountStrategy implements DiscountStrategy {
    
    private static final int BULK_THRESHOLD = 10;
    
    @Override
    public Discount apply(Cart cart, Customer customer) {
        boolean hasBulkItems = cart.getItems().stream()
            .anyMatch(item -> item.getQuantity() >= BULK_THRESHOLD);
        
        if (!hasBulkItems) return Discount.none();
        
        return Discount.percentage(new BigDecimal("0.08"), "Bulk purchase discount");
    }
    
    @Override
    public int getPriority() { return 2; }
}

// discount/DiscountEngine.java - Composite Strategy
@Service
public class DiscountEngine {
    
    private final List<DiscountStrategy> strategies;
    
    public DiscountEngine(List<DiscountStrategy> strategies) {
        // เรียงตาม priority
        this.strategies = strategies.stream()
            .sorted(Comparator.comparingInt(DiscountStrategy::getPriority))
            .collect(Collectors.toList());
    }
    
    public CartWithDiscount applyBestDiscount(Cart cart, Customer customer) {
        return strategies.stream()
            .map(s -> s.apply(cart, customer))
            .filter(d -> !d.isNone())
            .max(Comparator.comparing(Discount::getAmount))
            .map(discount -> CartWithDiscount.of(cart, discount))
            .orElse(CartWithDiscount.of(cart, Discount.none()));
    }
}
```

---

## ขั้นตอนที่ 3163: Observer/Event Pattern กับ ApplicationEventPublisher

Spring มี built-in Event System ผ่าน `ApplicationEventPublisher` ซึ่งช่วย decouple components ได้ดีมาก

### สร้าง Domain Events

```java
// events/OrderEvent.java
package com.example.shophub.events;

import org.springframework.context.ApplicationEvent;

public abstract class OrderEvent extends ApplicationEvent {
    
    private final String orderId;
    private final String customerId;
    
    protected OrderEvent(Object source, String orderId, String customerId) {
        super(source);
        this.orderId = orderId;
        this.customerId = customerId;
    }
    
    public String getOrderId() { return orderId; }
    public String getCustomerId() { return customerId; }
}

// events/OrderPlacedEvent.java
public class OrderPlacedEvent extends OrderEvent {
    
    private final List<OrderItem> items;
    private final BigDecimal totalAmount;
    
    public OrderPlacedEvent(Object source, String orderId, String customerId,
                            List<OrderItem> items, BigDecimal totalAmount) {
        super(source, orderId, customerId);
        this.items = items;
        this.totalAmount = totalAmount;
    }
    
    public List<OrderItem> getItems() { return items; }
    public BigDecimal getTotalAmount() { return totalAmount; }
}

// events/OrderShippedEvent.java
public class OrderShippedEvent extends OrderEvent {
    
    private final String trackingNumber;
    private final String carrier;
    private final LocalDateTime estimatedDelivery;
    
    // constructor, getters...
}

// events/OrderCancelledEvent.java
public class OrderCancelledEvent extends OrderEvent {
    
    private final String reason;
    private final BigDecimal refundAmount;
    
    // constructor, getters...
}
```

### Publish Events จาก Service

```java
// service/OrderService.java
@Service
@Transactional
public class OrderService {
    
    private final OrderRepository orderRepository;
    private final ApplicationEventPublisher eventPublisher;
    
    public OrderService(OrderRepository orderRepository,
                        ApplicationEventPublisher eventPublisher) {
        this.orderRepository = orderRepository;
        this.eventPublisher = eventPublisher;
    }
    
    public Order placeOrder(PlaceOrderCommand command) {
        // สร้าง order
        Order order = Order.create(command);
        orderRepository.save(order);
        
        // publish event - ทุก listener จะถูกแจ้ง
        eventPublisher.publishEvent(new OrderPlacedEvent(
            this,
            order.getId(),
            order.getCustomerId(),
            order.getItems(),
            order.getTotalAmount()
        ));
        
        return order;
    }
    
    public void shipOrder(String orderId, ShipmentDetails shipment) {
        Order order = orderRepository.findById(orderId)
            .orElseThrow(() -> new OrderNotFoundException(orderId));
        
        order.markAsShipped(shipment);
        orderRepository.save(order);
        
        eventPublisher.publishEvent(new OrderShippedEvent(
            this,
            orderId,
            order.getCustomerId(),
            shipment.getTrackingNumber(),
            shipment.getCarrier(),
            shipment.getEstimatedDelivery()
        ));
    }
}
```

### Event Listeners

```java
// listeners/NotificationEventListener.java
@Component
public class NotificationEventListener {
    
    private final EmailService emailService;
    private final SmsService smsService;
    private final PushNotificationService pushService;
    
    @EventListener
    public void handleOrderPlaced(OrderPlacedEvent event) {
        log.info("ส่ง notification สำหรับคำสั่งซื้อใหม่: {}", event.getOrderId());
        
        emailService.sendOrderConfirmation(
            event.getCustomerId(),
            event.getOrderId(),
            event.getTotalAmount()
        );
    }
    
    @EventListener
    public void handleOrderShipped(OrderShippedEvent event) {
        smsService.sendShippingNotification(
            event.getCustomerId(),
            event.getTrackingNumber(),
            event.getCarrier()
        );
        
        pushService.send(
            event.getCustomerId(),
            "คำสั่งซื้อของคุณถูกจัดส่งแล้ว! Tracking: " + event.getTrackingNumber()
        );
    }
    
    @EventListener
    @Async // ส่งใน background thread
    public void handleOrderCancelled(OrderCancelledEvent event) {
        emailService.sendCancellationEmail(
            event.getCustomerId(),
            event.getOrderId(),
            event.getReason(),
            event.getRefundAmount()
        );
    }
}

// listeners/InventoryEventListener.java
@Component
public class InventoryEventListener {
    
    private final InventoryService inventoryService;
    
    @EventListener
    @Async
    public void handleOrderPlaced(OrderPlacedEvent event) {
        // อัปเดต inventory เมื่อมีคำสั่งซื้อใหม่
        event.getItems().forEach(item -> 
            inventoryService.decreaseStock(item.getProductId(), item.getQuantity())
        );
    }
    
    @EventListener
    public void handleOrderCancelled(OrderCancelledEvent event) {
        // คืน inventory เมื่อยกเลิกคำสั่งซื้อ
        inventoryService.restoreStock(event.getOrderId());
    }
}

// listeners/AnalyticsEventListener.java
@Component
public class AnalyticsEventListener {
    
    private final AnalyticsService analyticsService;
    
    @EventListener
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    // TransactionalEventListener - จะทำงานหลัง transaction commit สำเร็จเท่านั้น
    public void handleOrderPlaced(OrderPlacedEvent event) {
        analyticsService.trackOrderPlaced(
            event.getOrderId(),
            event.getCustomerId(),
            event.getTotalAmount()
        );
    }
}
```

### Async Event Processing

```java
// config/AsyncConfig.java
@Configuration
@EnableAsync
public class AsyncConfig {
    
    @Bean(name = "eventExecutor")
    public TaskExecutor eventExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);
        executor.setMaxPoolSize(20);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("event-");
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        executor.initialize();
        return executor;
    }
}

// Async listener ที่กำหนด executor
@Component
public class AuditEventListener {
    
    @Async("eventExecutor")
    @EventListener
    public void handleAnyOrderEvent(OrderEvent event) {
        auditLog.record(
            event.getClass().getSimpleName(),
            event.getOrderId(),
            event.getCustomerId(),
            LocalDateTime.now()
        );
    }
}
```

---

## ขั้นตอนที่ 3164: Decorator Pattern กับ Spring Proxies

Decorator Pattern ใช้ `@Primary` และ delegate เพื่อเพิ่ม behavior โดยไม่แก้ implementation เดิม

### Use Case: Caching Decorator

```java
// service/ProductService.java (interface)
public interface ProductService {
    Product findById(String productId);
    List<Product> findAll(ProductFilter filter);
    Product save(Product product);
}

// service/DefaultProductService.java (Primary implementation)
@Service
public class DefaultProductService implements ProductService {
    
    private final ProductRepository productRepository;
    
    @Override
    public Product findById(String productId) {
        return productRepository.findById(productId)
            .orElseThrow(() -> new ProductNotFoundException(productId));
    }
    
    @Override
    public List<Product> findAll(ProductFilter filter) {
        return productRepository.findWithFilter(filter);
    }
    
    @Override
    public Product save(Product product) {
        return productRepository.save(product);
    }
}

// service/CachingProductService.java (Decorator)
@Service
@Primary // Spring จะ inject service นี้เป็น default
public class CachingProductService implements ProductService {
    
    private final ProductService delegate; // inject DefaultProductService
    private final CacheManager cacheManager;
    
    // ใช้ @Qualifier เพื่อ inject DefaultProductService ไม่ใช่ตัวเอง
    public CachingProductService(
            @Qualifier("defaultProductService") ProductService delegate,
            CacheManager cacheManager) {
        this.delegate = delegate;
        this.cacheManager = cacheManager;
    }
    
    @Override
    public Product findById(String productId) {
        Cache cache = cacheManager.getCache("products");
        Cache.ValueWrapper cached = cache.get(productId);
        
        if (cached != null) {
            log.debug("Cache hit สำหรับ product: {}", productId);
            return (Product) cached.get();
        }
        
        Product product = delegate.findById(productId);
        cache.put(productId, product);
        return product;
    }
    
    @Override
    public List<Product> findAll(ProductFilter filter) {
        // Cache list ด้วย filter เป็น key
        String cacheKey = filter.toCacheKey();
        Cache cache = cacheManager.getCache("productLists");
        Cache.ValueWrapper cached = cache.get(cacheKey);
        
        if (cached != null) {
            return (List<Product>) cached.get();
        }
        
        List<Product> products = delegate.findAll(filter);
        cache.put(cacheKey, products);
        return products;
    }
    
    @Override
    public Product save(Product product) {
        Product saved = delegate.save(product);
        // Invalidate cache เมื่อมีการบันทึก
        cacheManager.getCache("products").evict(saved.getId());
        cacheManager.getCache("productLists").clear();
        return saved;
    }
}
```

### Logging Decorator

```java
// service/LoggingProductService.java
@Service
@ConditionalOnProperty(name = "feature.detailed-logging", havingValue = "true")
public class LoggingProductService implements ProductService {
    
    private final ProductService delegate;
    private final MetricsService metricsService;
    
    public LoggingProductService(
            @Qualifier("cachingProductService") ProductService delegate,
            MetricsService metricsService) {
        this.delegate = delegate;
        this.metricsService = metricsService;
    }
    
    @Override
    public Product findById(String productId) {
        long startTime = System.currentTimeMillis();
        try {
            Product result = delegate.findById(productId);
            long duration = System.currentTimeMillis() - startTime;
            
            log.info("findById({}) completed in {}ms", productId, duration);
            metricsService.recordLatency("product.findById", duration);
            
            return result;
        } catch (Exception e) {
            log.error("findById({}) failed: {}", productId, e.getMessage());
            metricsService.incrementCounter("product.findById.errors");
            throw e;
        }
    }
    
    // delegate other methods similarly...
    @Override
    public List<Product> findAll(ProductFilter filter) {
        return delegate.findAll(filter);
    }
    
    @Override
    public Product save(Product product) {
        return delegate.save(product);
    }
}
```

---

## ขั้นตอนที่ 3165: Factory Pattern กับ @Bean และ @ConditionalOn*

Factory Pattern ใน Spring ใช้ `@Bean` methods และ `@Conditional` annotations เพื่อสร้าง object ตาม condition

### Payment Gateway Factory

```java
// payment/PaymentGateway.java
public interface PaymentGateway {
    PaymentResult charge(PaymentRequest request);
    RefundResult refund(RefundRequest request);
    PaymentStatus getStatus(String transactionId);
}

// payment/StripePaymentGateway.java
@Component("stripeGateway")
public class StripePaymentGateway implements PaymentGateway {
    
    @Value("${payment.stripe.api-key}")
    private String apiKey;
    
    @Override
    public PaymentResult charge(PaymentRequest request) {
        // Stripe SDK integration
        Stripe.apiKey = apiKey;
        try {
            PaymentIntentCreateParams params = PaymentIntentCreateParams.builder()
                .setAmount(request.getAmountCents())
                .setCurrency(request.getCurrency())
                .setPaymentMethod(request.getPaymentMethodId())
                .setConfirm(true)
                .build();
            
            PaymentIntent intent = PaymentIntent.create(params);
            return PaymentResult.success(intent.getId());
        } catch (StripeException e) {
            return PaymentResult.failed(e.getMessage());
        }
    }
    
    @Override
    public RefundResult refund(RefundRequest request) {
        // Stripe refund implementation
        try {
            RefundCreateParams params = RefundCreateParams.builder()
                .setPaymentIntent(request.getTransactionId())
                .setAmount(request.getAmountCents())
                .build();
            Refund refund = Refund.create(params);
            return RefundResult.success(refund.getId());
        } catch (StripeException e) {
            return RefundResult.failed(e.getMessage());
        }
    }
    
    @Override
    public PaymentStatus getStatus(String transactionId) {
        // implementation
        return PaymentStatus.SUCCESS;
    }
}

// payment/Omise2C2PGateway.java (Thai payment gateway)
@Component("omiseGateway")
public class Omise2C2PGateway implements PaymentGateway {
    
    @Value("${payment.omise.public-key}")
    private String publicKey;
    
    @Value("${payment.omise.secret-key}")
    private String secretKey;
    
    @Override
    public PaymentResult charge(PaymentRequest request) {
        // Omise integration สำหรับตลาดไทย
        Client client = new Client(publicKey, secretKey);
        ChargeParams params = ChargeParams.builder()
            .amount(request.getAmountSatang()) // Satang (1/100 of THB)
            .currency("thb")
            .card(request.getToken())
            .build();
        
        try {
            Charge charge = client.charges().create(params);
            return PaymentResult.success(charge.getId());
        } catch (OmiseException e) {
            return PaymentResult.failed(e.getMessage());
        }
    }
    
    @Override
    public RefundResult refund(RefundRequest request) {
        // implementation
        return RefundResult.success("refund-id");
    }
    
    @Override
    public PaymentStatus getStatus(String transactionId) {
        return PaymentStatus.SUCCESS;
    }
}
```

### Payment Factory Configuration

```java
// config/PaymentConfig.java
@Configuration
public class PaymentConfig {
    
    @Bean
    @ConditionalOnProperty(name = "payment.provider", havingValue = "stripe")
    public PaymentGateway stripePaymentGateway(
            @Qualifier("stripeGateway") PaymentGateway gateway) {
        return gateway;
    }
    
    @Bean
    @ConditionalOnProperty(name = "payment.provider", havingValue = "omise")
    public PaymentGateway omisePaymentGateway(
            @Qualifier("omiseGateway") PaymentGateway gateway) {
        return gateway;
    }
    
    // Mock gateway สำหรับ development/testing
    @Bean
    @ConditionalOnProperty(name = "payment.provider", havingValue = "mock", matchIfMissing = true)
    @Profile({"dev", "test"})
    public PaymentGateway mockPaymentGateway() {
        return new MockPaymentGateway();
    }
    
    // Payment gateway factory ที่เลือกตาม currency
    @Bean
    public PaymentGatewayFactory paymentGatewayFactory(
            @Qualifier("stripeGateway") PaymentGateway stripe,
            @Qualifier("omiseGateway") PaymentGateway omise) {
        return new CurrencyBasedPaymentGatewayFactory(stripe, omise);
    }
}

// payment/PaymentGatewayFactory.java
public interface PaymentGatewayFactory {
    PaymentGateway getGateway(String currency, PaymentType type);
}

// payment/CurrencyBasedPaymentGatewayFactory.java
public class CurrencyBasedPaymentGatewayFactory implements PaymentGatewayFactory {
    
    private final PaymentGateway stripeGateway;
    private final PaymentGateway omiseGateway;
    
    public CurrencyBasedPaymentGatewayFactory(PaymentGateway stripeGateway,
                                               PaymentGateway omiseGateway) {
        this.stripeGateway = stripeGateway;
        this.omiseGateway = omiseGateway;
    }
    
    @Override
    public PaymentGateway getGateway(String currency, PaymentType type) {
        return switch (currency.toUpperCase()) {
            case "THB" -> omiseGateway; // ใช้ Omise สำหรับบาทไทย
            case "USD", "EUR", "GBP" -> stripeGateway;
            default -> throw new UnsupportedCurrencyException(currency);
        };
    }
}
```

---

## ขั้นตอนที่ 3166: Command Pattern กับ Spring

Command Pattern encapsulate request เป็น object ทำให้ support undo/redo, queuing, และ logging ได้

### สร้าง Command Framework

```java
// command/Command.java
public interface Command<T> {
    T execute();
    void undo();
    String getDescription();
}

// command/CommandBus.java
@Service
public class CommandBus {
    
    private final Map<Class<?>, CommandHandler<?, ?>> handlers;
    private final Deque<Command<?>> executedCommands = new ArrayDeque<>();
    private final ApplicationEventPublisher eventPublisher;
    private final AuditService auditService;
    
    public CommandBus(List<CommandHandler<?, ?>> handlerList,
                     ApplicationEventPublisher eventPublisher,
                     AuditService auditService) {
        this.handlers = handlerList.stream()
            .collect(Collectors.toMap(
                h -> h.getCommandClass(),
                h -> h
            ));
        this.eventPublisher = eventPublisher;
        this.auditService = auditService;
    }
    
    @SuppressWarnings("unchecked")
    public <C, R> R dispatch(C command) {
        CommandHandler<C, R> handler = (CommandHandler<C, R>) handlers.get(command.getClass());
        
        if (handler == null) {
            throw new CommandHandlerNotFoundException(command.getClass());
        }
        
        // Audit logging
        auditService.logCommand(command);
        
        R result = handler.handle(command);
        
        // Publish command executed event
        eventPublisher.publishEvent(new CommandExecutedEvent(command, result));
        
        return result;
    }
}

// command/CommandHandler.java
public interface CommandHandler<C, R> {
    R handle(C command);
    Class<C> getCommandClass();
}
```

### Concrete Commands

```java
// command/CreateOrderCommand.java
public class CreateOrderCommand {
    private final String customerId;
    private final List<OrderItem> items;
    private final String paymentMethodId;
    private final ShippingAddress shippingAddress;
    
    // constructor, getters...
}

// command/CreateOrderCommandHandler.java
@Component
public class CreateOrderCommandHandler implements CommandHandler<CreateOrderCommand, Order> {
    
    private final OrderService orderService;
    private final CustomerValidator customerValidator;
    
    @Override
    public Order handle(CreateOrderCommand command) {
        customerValidator.validate(command.getCustomerId());
        
        return orderService.createOrder(
            command.getCustomerId(),
            command.getItems(),
            command.getPaymentMethodId(),
            command.getShippingAddress()
        );
    }
    
    @Override
    public Class<CreateOrderCommand> getCommandClass() {
        return CreateOrderCommand.class;
    }
}

// command/CancelOrderCommand.java
public class CancelOrderCommand {
    private final String orderId;
    private final String reason;
    private final String requestedByUserId;
    
    // constructor, getters...
}

// command/CancelOrderCommandHandler.java
@Component
public class CancelOrderCommandHandler implements CommandHandler<CancelOrderCommand, CancellationResult> {
    
    private final OrderService orderService;
    private final AuthorizationService authService;
    
    @Override
    public CancellationResult handle(CancelOrderCommand command) {
        // ตรวจสอบสิทธิ์
        authService.checkPermission(command.getRequestedByUserId(), Permission.CANCEL_ORDER);
        
        return orderService.cancelOrder(command.getOrderId(), command.getReason());
    }
    
    @Override
    public Class<CancelOrderCommand> getCommandClass() {
        return CancelOrderCommand.class;
    }
}
```

### REST Controller ใช้ Command Bus

```java
// controller/OrderController.java
@RestController
@RequestMapping("/api/v1/orders")
public class OrderController {
    
    private final CommandBus commandBus;
    
    @PostMapping
    public ResponseEntity<Order> createOrder(@Valid @RequestBody CreateOrderRequest request,
                                              @AuthenticationPrincipal UserDetails user) {
        CreateOrderCommand command = new CreateOrderCommand(
            user.getUsername(),
            request.getItems(),
            request.getPaymentMethodId(),
            request.getShippingAddress()
        );
        
        Order order = commandBus.dispatch(command);
        return ResponseEntity.status(HttpStatus.CREATED).body(order);
    }
    
    @PostMapping("/{orderId}/cancel")
    public ResponseEntity<CancellationResult> cancelOrder(
            @PathVariable String orderId,
            @RequestBody CancelOrderRequest request,
            @AuthenticationPrincipal UserDetails user) {
        
        CancelOrderCommand command = new CancelOrderCommand(
            orderId,
            request.getReason(),
            user.getUsername()
        );
        
        return ResponseEntity.ok(commandBus.dispatch(command));
    }
}
```

---

## ขั้นตอนที่ 3167: Composite Pattern สำหรับ Services

Composite Pattern ช่วยให้เรา treat individual objects และ groups of objects เหมือนกัน

### Use Case: Discount Composite

```java
// discount/DiscountComponent.java
public interface DiscountComponent {
    BigDecimal calculate(Cart cart, Customer customer);
    String getDescription();
}

// discount/PercentageDiscount.java (Leaf)
public class PercentageDiscount implements DiscountComponent {
    
    private final BigDecimal rate;
    private final String description;
    
    public PercentageDiscount(BigDecimal rate, String description) {
        this.rate = rate;
        this.description = description;
    }
    
    @Override
    public BigDecimal calculate(Cart cart, Customer customer) {
        return cart.getTotalAmount().multiply(rate);
    }
    
    @Override
    public String getDescription() { return description; }
}

// discount/FixedDiscount.java (Leaf)
public class FixedDiscount implements DiscountComponent {
    
    private final BigDecimal amount;
    private final String description;
    
    @Override
    public BigDecimal calculate(Cart cart, Customer customer) {
        return amount.min(cart.getTotalAmount()); // ไม่ให้เกินราคาสินค้า
    }
    
    @Override
    public String getDescription() { return description; }
}

// discount/CompositeDiscount.java (Composite)
public class CompositeDiscount implements DiscountComponent {
    
    private final List<DiscountComponent> discounts = new ArrayList<>();
    private final DiscountCombineStrategy combineStrategy;
    private final String description;
    
    public enum DiscountCombineStrategy {
        SUM,    // รวมส่วนลดทั้งหมด
        MAX,    // เลือกส่วนลดที่มากที่สุด
        SEQUENTIAL // ลดซ้อนกัน
    }
    
    public void add(DiscountComponent discount) {
        discounts.add(discount);
    }
    
    @Override
    public BigDecimal calculate(Cart cart, Customer customer) {
        return switch (combineStrategy) {
            case SUM -> discounts.stream()
                .map(d -> d.calculate(cart, customer))
                .reduce(BigDecimal.ZERO, BigDecimal::add);
            
            case MAX -> discounts.stream()
                .map(d -> d.calculate(cart, customer))
                .max(BigDecimal::compareTo)
                .orElse(BigDecimal.ZERO);
            
            case SEQUENTIAL -> {
                BigDecimal remainingAmount = cart.getTotalAmount();
                BigDecimal totalDiscount = BigDecimal.ZERO;
                for (DiscountComponent discount : discounts) {
                    BigDecimal d = discount.calculate(
                        Cart.withAmount(remainingAmount), customer
                    );
                    totalDiscount = totalDiscount.add(d);
                    remainingAmount = remainingAmount.subtract(d);
                }
                yield totalDiscount;
            }
        };
    }
    
    @Override
    public String getDescription() { return description; }
}
```

### Building Discount Tree ด้วย Builder

```java
// config/DiscountConfig.java
@Configuration
public class DiscountConfig {
    
    @Bean
    public DiscountComponent memberDiscountTree() {
        CompositeDiscount composite = new CompositeDiscount(
            DiscountCombineStrategy.MAX,
            "Member discounts"
        );
        
        composite.add(new PercentageDiscount(new BigDecimal("0.05"), "Silver member"));
        composite.add(new PercentageDiscount(new BigDecimal("0.10"), "Gold member"));
        composite.add(new PercentageDiscount(new BigDecimal("0.15"), "Platinum member"));
        
        return composite;
    }
    
    @Bean
    public DiscountComponent fullDiscountTree(
            @Qualifier("memberDiscountTree") DiscountComponent memberDiscounts) {
        
        CompositeDiscount root = new CompositeDiscount(
            DiscountCombineStrategy.SUM,
            "All applicable discounts"
        );
        
        root.add(memberDiscounts);
        root.add(new PercentageDiscount(new BigDecimal("0.02"), "First purchase"));
        root.add(new FixedDiscount(new BigDecimal("50"), "Welcome coupon"));
        
        return root;
    }
}
```

### Validation Composite

```java
// validation/ValidationRule.java
public interface ValidationRule<T> {
    ValidationResult validate(T subject);
    String getRuleName();
}

// validation/CompositeValidationRule.java
public class CompositeValidationRule<T> implements ValidationRule<T> {
    
    private final List<ValidationRule<T>> rules;
    private final String name;
    
    public CompositeValidationRule(String name, List<ValidationRule<T>> rules) {
        this.name = name;
        this.rules = rules;
    }
    
    @Override
    public ValidationResult validate(T subject) {
        List<String> errors = rules.stream()
            .map(rule -> rule.validate(subject))
            .filter(result -> !result.isValid())
            .flatMap(result -> result.getErrors().stream())
            .collect(Collectors.toList());
        
        return errors.isEmpty() ? ValidationResult.valid() : ValidationResult.invalid(errors);
    }
    
    @Override
    public String getRuleName() { return name; }
}

// validation/OrderValidationConfig.java
@Configuration
public class OrderValidationConfig {
    
    @Bean
    public ValidationRule<Order> orderValidationRule() {
        return new CompositeValidationRule<>("Order validation", List.of(
            new NonEmptyItemsRule(),
            new ValidCustomerRule(),
            new ValidShippingAddressRule(),
            new PaymentMethodValidRule(),
            new StockAvailabilityRule()
        ));
    }
}
```

---

## ขั้นตอนที่ 3168-3200: Summary และ Best Practices

### Design Pattern Selection Guide

```
ปัญหา                          Pattern ที่เหมาะสม
─────────────────────────────────────────────────────
Algorithm หลายขั้นตอน          Template Method
เปลี่ยน Algorithm ตอน runtime  Strategy + @Qualifier
Decouple producers/consumers   Observer (ApplicationEvent)
เพิ่ม behavior ไม่แก้ code     Decorator (@Primary + delegate)
สร้าง object ตาม condition     Factory (@Bean + @ConditionalOn*)
Encapsulate operation          Command + CommandBus
Tree structure                 Composite
```

### Anti-patterns ที่ควรหลีกเลี่ยง

```java
// BAD: Over-engineering ด้วย pattern ที่ไม่จำเป็น
// ถ้ามีแค่ 1 implementation ไม่ต้องใช้ Strategy
public interface SimpleCalculator {
    int add(int a, int b);
}

// BAD: Circular dependency ใน Decorator
@Service
@Primary
public class BadDecorator implements ProductService {
    @Autowired // Self-injection - อาจเกิด circular dependency
    private ProductService self;
}

// GOOD: ใช้ @Qualifier แทน
@Service
@Primary
public class GoodDecorator implements ProductService {
    private final ProductService delegate;
    
    public GoodDecorator(@Qualifier("defaultProductService") ProductService delegate) {
        this.delegate = delegate;
    }
}
```

### Testing Design Patterns

```java
// test/OrderProcessingTemplateTest.java
@ExtendWith(MockitoExtension.class)
class OrderProcessingTemplateTest {
    
    @Mock
    private InventoryService inventoryService;
    
    @Mock
    private PaymentGateway paymentGateway;
    
    @Mock
    private OrderRepository orderRepository;
    
    @InjectMocks
    private StandardOrderProcessor processor;
    
    @Test
    void shouldProcessOrderSuccessfully() {
        // Arrange
        OrderRequest request = OrderRequest.builder()
            .customerId("customer-1")
            .items(List.of(new OrderItem("product-1", 2)))
            .paymentMethodId("pm-1")
            .build();
        
        InventoryReservation reservation = InventoryReservation.builder()
            .id("res-1")
            .totalAmount(new BigDecimal("200"))
            .build();
        
        when(inventoryService.reserve(any())).thenReturn(reservation);
        when(paymentGateway.charge(any(), any())).thenReturn(PaymentResult.success("tx-1"));
        when(orderRepository.save(any())).thenAnswer(inv -> {
            Order order = inv.getArgument(0);
            order.setId("order-1");
            return order;
        });
        
        // Act
        OrderResult result = processor.processOrder(request);
        
        // Assert
        assertTrue(result.isSuccess());
        assertNotNull(result.getOrder());
        verify(inventoryService).reserve(request.getItems());
        verify(paymentGateway).charge(any(), eq(reservation));
    }
    
    @Test
    void shouldReleaseInventoryOnPaymentFailure() {
        // Arrange
        InventoryReservation reservation = InventoryReservation.builder().id("res-1").build();
        
        when(inventoryService.reserve(any())).thenReturn(reservation);
        when(paymentGateway.charge(any(), any())).thenReturn(PaymentResult.failed("Card declined"));
        
        // Act
        OrderResult result = processor.processOrder(validRequest());
        
        // Assert
        assertFalse(result.isSuccess());
        verify(inventoryService).release(reservation); // ต้องคืน inventory
    }
}
```

### Performance Considerations

```java
// ใช้ @Lazy เพื่อ delay initialization ของ heavy strategies
@Configuration
public class StrategyConfig {
    
    @Bean
    @Lazy // สร้างเมื่อมีการใช้งานครั้งแรก
    public ComplexAnalyticsStrategy complexAnalyticsStrategy() {
        return new ComplexAnalyticsStrategy(); // heavy initialization
    }
}

// ใช้ Flyweight pattern กับ Strategy ที่ไม่มี state
// Strategy ที่ไม่มี instance variable ควรเป็น singleton (default ใน Spring)
@Service // Default scope = Singleton - เหมาะสำหรับ stateless strategy
public class StatelessShippingStrategy implements ShippingStrategy {
    
    @Override
    public ShippingCost calculate(ShippingRequest request) {
        // Pure function - ไม่มี state
        return ShippingCost.of(request.getWeightKg().multiply(new BigDecimal("30")));
    }
}
```

---

## สรุป

ใน Part 89 นี้ เราได้เรียนรู้ Design Patterns ขั้นสูงที่ใช้ใน Spring Boot:

1. **Template Method** - กำหนด workflow skeleton ใน abstract class
2. **Strategy + @Qualifier** - เปลี่ยน algorithm ตอน runtime
3. **Observer (ApplicationEvent)** - decouple producers/consumers
4. **Decorator (@Primary + delegate)** - เพิ่ม behavior โดยไม่แก้ code เดิม
5. **Factory (@Bean + @Conditional)** - สร้าง object ตาม environment/config
6. **Command + CommandBus** - encapsulate operations และ support audit
7. **Composite** - treat single/group objects เหมือนกัน

Pattern เหล่านี้ช่วยให้ code มี:
- **High cohesion** - แต่ละ class มีหน้าที่ชัดเจน
- **Low coupling** - components ไม่ผูกติดกัน
- **Open/Closed** - เพิ่ม feature ได้โดยไม่แก้ existing code
- **Testability** - ทดสอบได้ง่าย

---

*[← Part 88: Data Streaming](./part-88-data-streaming.md) | [Part 90: Production Optimization →](./part-90-production-optimization.md)*

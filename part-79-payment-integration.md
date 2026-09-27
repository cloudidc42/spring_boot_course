# Part 79: Payment Integration
## ขั้นตอนที่ 2761-2800

**ระดับ:** ระดับโลก (World-Class)
**เวลาเรียน:** 5-6 ชั่วโมง
**เป้าหมาย:** เรียนรู้การ integrate ระบบชำระเงินด้วย Stripe, จัดการ Webhook, Idempotent operations, Payment state machine, Refunds และ PCI compliance

---

## ขั้นตอนที่ 2761: แนะนำ Payment Architecture

การออกแบบระบบชำระเงินต้องคำนึงถึงความปลอดภัย ความเชื่อถือได้ และ PCI DSS compliance ห้าม store card number โดยตรง ให้ใช้ token ของ payment gateway แทน

```xml
<!-- pom.xml -->
<dependencies>
    <dependency>
        <groupId>com.stripe</groupId>
        <artifactId>stripe-java</artifactId>
        <version>25.1.0</version>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.statemachine</groupId>
        <artifactId>spring-statemachine-core</artifactId>
        <version>4.0.0</version>
    </dependency>
</dependencies>
```

```yaml
# application.yml
stripe:
  api-key: ${STRIPE_SECRET_KEY}
  webhook-secret: ${STRIPE_WEBHOOK_SECRET}
  publishable-key: ${STRIPE_PUBLISHABLE_KEY}

app:
  payment:
    currency: THB
    capture-mode: automatic  # automatic หรือ manual
    idempotency-key-ttl: 24h
    refund-window-days: 30
```

---

## ขั้นตอนที่ 2762: Stripe Configuration

```java
// config/StripeConfig.java
package com.example.payment.config;

import com.stripe.Stripe;
import jakarta.annotation.PostConstruct;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Configuration;

@Configuration
public class StripeConfig {

    @Value("${stripe.api-key}")
    private String apiKey;

    @PostConstruct
    public void initStripe() {
        Stripe.apiKey = apiKey;
        Stripe.setAppInfo("MyApp", "1.0.0", "https://example.com");
    }
}
```

---

## ขั้นตอนที่ 2763: Payment Entity และ State Machine

```java
// entity/Payment.java
package com.example.payment.entity;

import jakarta.persistence.*;
import lombok.*;
import java.math.BigDecimal;
import java.time.LocalDateTime;

@Entity
@Table(name = "payments", indexes = {
    @Index(name = "idx_payment_order_id", columnList = "order_id"),
    @Index(name = "idx_payment_stripe_intent", columnList = "stripe_payment_intent_id"),
    @Index(name = "idx_payment_status", columnList = "status"),
    @Index(name = "idx_payment_idempotency_key", columnList = "idempotency_key", unique = true)
})
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class Payment {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "order_id", nullable = false)
    private Long orderId;

    @Column(name = "user_id", nullable = false)
    private String userId;

    @Column(name = "idempotency_key", nullable = false, unique = true)
    private String idempotencyKey;

    @Column(nullable = false, precision = 12, scale = 2)
    private BigDecimal amount;

    @Column(nullable = false, length = 3)
    private String currency;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private PaymentStatus status;

    @Column(name = "stripe_payment_intent_id")
    private String stripePaymentIntentId;

    @Column(name = "stripe_charge_id")
    private String stripeChargeId;

    @Column(name = "payment_method_type")
    private String paymentMethodType;  // card, promptpay, etc.

    @Column(name = "failure_code")
    private String failureCode;

    @Column(name = "failure_message")
    private String failureMessage;

    @Column(name = "metadata", columnDefinition = "jsonb")
    private String metadata;

    @Column(name = "created_at", nullable = false)
    private LocalDateTime createdAt;

    @Column(name = "updated_at")
    private LocalDateTime updatedAt;

    @Column(name = "completed_at")
    private LocalDateTime completedAt;

    @Version
    private Long version;  // Optimistic locking

    @PrePersist
    protected void onCreate() {
        createdAt = LocalDateTime.now();
        updatedAt = LocalDateTime.now();
    }

    @PreUpdate
    protected void onUpdate() {
        updatedAt = LocalDateTime.now();
    }
}
```

```java
// entity/PaymentStatus.java
package com.example.payment.entity;

public enum PaymentStatus {
    INITIATED,      // สร้าง payment intent แล้ว
    PROCESSING,     // กำลังประมวลผล
    REQUIRES_ACTION, // ต้องการ 3D Secure
    SUCCEEDED,      // ชำระเงินสำเร็จ
    FAILED,         // ล้มเหลว
    CANCELED,       // ยกเลิก
    REFUNDING,      // กำลัง refund
    REFUNDED,       // refund แล้ว
    PARTIALLY_REFUNDED  // refund บางส่วน
}
```

---

## ขั้นตอนที่ 2764: Payment Service หลัก

```java
// service/PaymentService.java
package com.example.payment.service;

import com.example.payment.dto.*;
import com.example.payment.entity.*;
import com.example.payment.repository.PaymentRepository;
import com.stripe.exception.StripeException;
import com.stripe.model.PaymentIntent;
import com.stripe.param.PaymentIntentCreateParams;
import com.stripe.param.PaymentIntentConfirmParams;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.math.BigDecimal;
import java.util.HashMap;
import java.util.Map;
import java.util.UUID;

@Slf4j
@Service
@RequiredArgsConstructor
public class PaymentService {

    private final PaymentRepository paymentRepository;

    @Value("${app.payment.currency}")
    private String defaultCurrency;

    // สร้าง Payment Intent (ขั้นตอนที่ 1)
    @Transactional
    public PaymentIntentResponse createPaymentIntent(CreatePaymentRequest request)
            throws StripeException {

        // สร้าง idempotency key เพื่อป้องกัน duplicate payment
        String idempotencyKey = request.getIdempotencyKey() != null
            ? request.getIdempotencyKey()
            : "order-" + request.getOrderId() + "-" + UUID.randomUUID();

        // ตรวจสอบ duplicate
        if (paymentRepository.existsByIdempotencyKey(idempotencyKey)) {
            Payment existing = paymentRepository.findByIdempotencyKey(idempotencyKey)
                .orElseThrow();
            log.info("Returning existing payment for idempotency key: {}", idempotencyKey);
            return new PaymentIntentResponse(
                existing.getStripePaymentIntentId(),
                null,  // ไม่ return client secret ซ้ำ
                existing.getStatus().name(),
                existing.getAmount()
            );
        }

        // แปลงเป็น smallest currency unit (สตางค์)
        long amountInSmallestUnit = request.getAmount()
            .multiply(new BigDecimal("100"))
            .longValue();

        // สร้าง metadata
        Map<String, String> metadata = new HashMap<>();
        metadata.put("orderId", request.getOrderId().toString());
        metadata.put("userId", request.getUserId());
        metadata.put("idempotencyKey", idempotencyKey);

        // สร้าง Payment Intent ใน Stripe
        PaymentIntentCreateParams params = PaymentIntentCreateParams.builder()
            .setAmount(amountInSmallestUnit)
            .setCurrency(defaultCurrency.toLowerCase())
            .setAutomaticPaymentMethods(
                PaymentIntentCreateParams.AutomaticPaymentMethods.builder()
                    .setEnabled(true)
                    .build()
            )
            .putAllMetadata(metadata)
            .setDescription("Payment for order #" + request.getOrderId())
            .build();

        PaymentIntent intent = PaymentIntent.create(params);

        // บันทึกลง database
        Payment payment = Payment.builder()
            .orderId(request.getOrderId())
            .userId(request.getUserId())
            .idempotencyKey(idempotencyKey)
            .amount(request.getAmount())
            .currency(defaultCurrency)
            .status(PaymentStatus.INITIATED)
            .stripePaymentIntentId(intent.getId())
            .build();

        paymentRepository.save(payment);

        log.info("Created payment intent {} for order {}", intent.getId(), request.getOrderId());

        return new PaymentIntentResponse(
            intent.getId(),
            intent.getClientSecret(),
            intent.getStatus(),
            request.getAmount()
        );
    }

    // ดึงสถานะ payment
    public Payment getPaymentByOrderId(Long orderId) {
        return paymentRepository.findByOrderId(orderId)
            .orElseThrow(() -> new PaymentNotFoundException("ไม่พบข้อมูลการชำระเงินสำหรับ order: " + orderId));
    }
}
```

---

## ขั้นตอนที่ 2765: Webhook Handler

Webhook สำคัญมากสำหรับ Stripe เพราะ payment อาจสำเร็จหลังจาก user ออกจาก app ไปแล้ว

```java
// controller/StripeWebhookController.java
package com.example.payment.controller;

import com.example.payment.service.WebhookProcessingService;
import com.stripe.exception.SignatureVerificationException;
import com.stripe.model.*;
import com.stripe.net.Webhook;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@Slf4j
@RestController
@RequestMapping("/api/v1/webhooks/stripe")
@RequiredArgsConstructor
public class StripeWebhookController {

    private final WebhookProcessingService webhookService;

    @Value("${stripe.webhook-secret}")
    private String webhookSecret;

    @PostMapping
    public ResponseEntity<String> handleWebhook(
            @RequestBody String payload,
            @RequestHeader("Stripe-Signature") String sigHeader) {

        // 1. ตรวจสอบ signature เพื่อยืนยันว่า event มาจาก Stripe จริง
        Event event;
        try {
            event = Webhook.constructEvent(payload, sigHeader, webhookSecret);
        } catch (SignatureVerificationException e) {
            log.warn("Invalid Stripe webhook signature");
            return ResponseEntity.status(HttpStatus.BAD_REQUEST)
                .body("Invalid signature");
        }

        log.info("Received Stripe webhook: type={}, id={}", event.getType(), event.getId());

        // 2. Process event แบบ async เพื่อตอบ Stripe ได้เร็ว (ภายใน 30 วินาที)
        webhookService.processAsync(event);

        // 3. ตอบ 200 ทันทีเพื่อบอก Stripe ว่าได้รับแล้ว
        return ResponseEntity.ok("Received");
    }
}
```

```java
// service/WebhookProcessingService.java
package com.example.payment.service;

import com.example.payment.entity.*;
import com.example.payment.repository.PaymentRepository;
import com.stripe.model.*;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.scheduling.annotation.Async;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.time.LocalDateTime;
import java.util.Optional;

@Slf4j
@Service
@RequiredArgsConstructor
public class WebhookProcessingService {

    private final PaymentRepository paymentRepository;
    private final OrderService orderService;

    @Async
    public void processAsync(Event event) {
        try {
            processEvent(event);
        } catch (Exception e) {
            log.error("Failed to process webhook event {}: {}", event.getId(), e.getMessage(), e);
            // บันทึก failed event เพื่อ retry ภายหลัง
        }
    }

    @Transactional
    public void processEvent(Event event) {
        // ตรวจสอบ idempotency - ถ้า event นี้เคย process แล้วให้ skip
        if (isAlreadyProcessed(event.getId())) {
            log.info("Skipping already-processed event: {}", event.getId());
            return;
        }

        switch (event.getType()) {
            case "payment_intent.succeeded" -> handlePaymentSucceeded(event);
            case "payment_intent.payment_failed" -> handlePaymentFailed(event);
            case "payment_intent.requires_action" -> handleRequiresAction(event);
            case "charge.refunded" -> handleChargeRefunded(event);
            case "charge.dispute.created" -> handleDisputeCreated(event);
            default -> log.debug("Unhandled event type: {}", event.getType());
        }

        markEventAsProcessed(event.getId());
    }

    private void handlePaymentSucceeded(Event event) {
        PaymentIntent intent = (PaymentIntent) event.getDataObjectDeserializer()
            .getObject().orElseThrow();

        Optional<Payment> paymentOpt = paymentRepository
            .findByStripePaymentIntentId(intent.getId());

        if (paymentOpt.isEmpty()) {
            log.warn("Payment not found for intent: {}", intent.getId());
            return;
        }

        Payment payment = paymentOpt.get();

        // อัปเดตสถานะ
        payment.setStatus(PaymentStatus.SUCCEEDED);
        payment.setStripeChargeId(intent.getLatestCharge());
        payment.setCompletedAt(LocalDateTime.now());

        if (intent.getPaymentMethod() != null) {
            payment.setPaymentMethodType(intent.getPaymentMethod());
        }

        paymentRepository.save(payment);

        // แจ้ง order service ว่าชำระเงินสำเร็จ
        orderService.confirmPayment(payment.getOrderId(), payment.getId());

        log.info("Payment succeeded: intentId={}, orderId={}",
            intent.getId(), payment.getOrderId());
    }

    private void handlePaymentFailed(Event event) {
        PaymentIntent intent = (PaymentIntent) event.getDataObjectDeserializer()
            .getObject().orElseThrow();

        paymentRepository.findByStripePaymentIntentId(intent.getId())
            .ifPresent(payment -> {
                payment.setStatus(PaymentStatus.FAILED);
                payment.setFailureCode(intent.getLastPaymentError() != null
                    ? intent.getLastPaymentError().getCode() : "unknown");
                payment.setFailureMessage(intent.getLastPaymentError() != null
                    ? intent.getLastPaymentError().getMessage() : "Payment failed");
                paymentRepository.save(payment);

                log.info("Payment failed: intentId={}, code={}",
                    intent.getId(), payment.getFailureCode());
            });
    }

    private void handleRequiresAction(Event event) {
        PaymentIntent intent = (PaymentIntent) event.getDataObjectDeserializer()
            .getObject().orElseThrow();

        paymentRepository.findByStripePaymentIntentId(intent.getId())
            .ifPresent(payment -> {
                payment.setStatus(PaymentStatus.REQUIRES_ACTION);
                paymentRepository.save(payment);
            });
    }

    private void handleChargeRefunded(Event event) {
        Charge charge = (Charge) event.getDataObjectDeserializer()
            .getObject().orElseThrow();

        paymentRepository.findByStripeChargeId(charge.getId())
            .ifPresent(payment -> {
                boolean fullyRefunded = charge.getRefunded();
                payment.setStatus(fullyRefunded
                    ? PaymentStatus.REFUNDED
                    : PaymentStatus.PARTIALLY_REFUNDED);
                paymentRepository.save(payment);

                log.info("Charge refunded: chargeId={}, fullyRefunded={}",
                    charge.getId(), fullyRefunded);
            });
    }

    private void handleDisputeCreated(Event event) {
        Dispute dispute = (Dispute) event.getDataObjectDeserializer()
            .getObject().orElseThrow();
        log.warn("Dispute created for charge: {}, amount: {}",
            dispute.getCharge(), dispute.getAmount());
        // ส่ง alert ไปยัง team
    }

    private boolean isAlreadyProcessed(String eventId) {
        // ตรวจสอบจาก processed_events table
        return false; // placeholder
    }

    private void markEventAsProcessed(String eventId) {
        // บันทึกลง processed_events table
    }
}
```

---

## ขั้นตอนที่ 2766: Refund Service

```java
// service/RefundService.java
package com.example.payment.service;

import com.example.payment.dto.RefundRequest;
import com.example.payment.dto.RefundResponse;
import com.example.payment.entity.*;
import com.example.payment.repository.*;
import com.stripe.exception.StripeException;
import com.stripe.model.Refund;
import com.stripe.param.RefundCreateParams;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.time.temporal.ChronoUnit;

@Slf4j
@Service
@RequiredArgsConstructor
public class RefundService {

    private final PaymentRepository paymentRepository;
    private final RefundRecordRepository refundRecordRepository;

    @Value("${app.payment.refund-window-days}")
    private int refundWindowDays;

    @Transactional
    public RefundResponse processRefund(Long paymentId, RefundRequest request)
            throws StripeException {

        Payment payment = paymentRepository.findById(paymentId)
            .orElseThrow(() -> new PaymentNotFoundException("ไม่พบ payment: " + paymentId));

        // ตรวจสอบว่า payment ชำระเงินสำเร็จแล้ว
        if (payment.getStatus() != PaymentStatus.SUCCEEDED) {
            throw new IllegalStateException(
                "ไม่สามารถ refund payment ที่มีสถานะ: " + payment.getStatus()
            );
        }

        // ตรวจสอบ refund window
        if (payment.getCompletedAt() != null) {
            long daysSincePayment = ChronoUnit.DAYS.between(
                payment.getCompletedAt(), LocalDateTime.now()
            );
            if (daysSincePayment > refundWindowDays) {
                throw new IllegalStateException(
                    String.format("เกิน %d วันที่กำหนดสำหรับการ refund", refundWindowDays)
                );
            }
        }

        // คำนวณยอด refund
        BigDecimal refundAmount = request.getAmount() != null
            ? request.getAmount()
            : payment.getAmount();  // refund ทั้งหมด

        // ตรวจสอบว่าไม่ refund เกินยอดที่ชำระ
        BigDecimal alreadyRefunded = refundRecordRepository
            .sumRefundedAmount(paymentId);
        BigDecimal maxRefundable = payment.getAmount().subtract(alreadyRefunded);

        if (refundAmount.compareTo(maxRefundable) > 0) {
            throw new IllegalArgumentException(
                String.format("ยอด refund (%s) เกินยอดที่สามารถ refund ได้ (%s)",
                    refundAmount, maxRefundable)
            );
        }

        // แปลงเป็น smallest unit
        long amountInCents = refundAmount.multiply(new BigDecimal("100")).longValue();

        // สร้าง Refund ใน Stripe
        RefundCreateParams params = RefundCreateParams.builder()
            .setCharge(payment.getStripeChargeId())
            .setAmount(amountInCents)
            .putMetadata("paymentId", paymentId.toString())
            .putMetadata("reason", request.getReason())
            .build();

        Refund stripeRefund = Refund.create(params);

        // บันทึก refund record
        RefundRecord refundRecord = RefundRecord.builder()
            .paymentId(paymentId)
            .stripeRefundId(stripeRefund.getId())
            .amount(refundAmount)
            .reason(request.getReason())
            .status("PENDING")
            .createdAt(LocalDateTime.now())
            .build();

        refundRecordRepository.save(refundRecord);

        // อัปเดตสถานะ payment
        payment.setStatus(PaymentStatus.REFUNDING);
        paymentRepository.save(payment);

        log.info("Refund initiated: paymentId={}, amount={}, stripeRefundId={}",
            paymentId, refundAmount, stripeRefund.getId());

        return new RefundResponse(
            refundRecord.getId(),
            stripeRefund.getId(),
            refundAmount,
            "PENDING"
        );
    }
}
```

---

## ขั้นตอนที่ 2767: Payment State Machine

```java
// statemachine/PaymentStateMachineConfig.java
package com.example.payment.statemachine;

import com.example.payment.entity.PaymentStatus;
import org.springframework.context.annotation.Configuration;
import org.springframework.statemachine.config.*;
import org.springframework.statemachine.config.builders.*;

import java.util.EnumSet;

@Configuration
@EnableStateMachine
public class PaymentStateMachineConfig
        extends StateMachineConfigurerAdapter<PaymentStatus, PaymentEvent> {

    @Override
    public void configure(StateMachineStateConfigurer<PaymentStatus, PaymentEvent> states)
            throws Exception {
        states
            .withStates()
            .initial(PaymentStatus.INITIATED)
            .states(EnumSet.allOf(PaymentStatus.class))
            .end(PaymentStatus.SUCCEEDED)
            .end(PaymentStatus.FAILED)
            .end(PaymentStatus.CANCELED)
            .end(PaymentStatus.REFUNDED);
    }

    @Override
    public void configure(StateMachineTransitionConfigurer<PaymentStatus, PaymentEvent> transitions)
            throws Exception {
        transitions
            // INITIATED -> PROCESSING
            .withExternal()
                .source(PaymentStatus.INITIATED)
                .target(PaymentStatus.PROCESSING)
                .event(PaymentEvent.PROCESS)
            .and()
            // PROCESSING -> REQUIRES_ACTION (3D Secure)
            .withExternal()
                .source(PaymentStatus.PROCESSING)
                .target(PaymentStatus.REQUIRES_ACTION)
                .event(PaymentEvent.REQUIRES_ACTION)
            .and()
            // REQUIRES_ACTION -> PROCESSING (user ยืนยัน 3D Secure)
            .withExternal()
                .source(PaymentStatus.REQUIRES_ACTION)
                .target(PaymentStatus.PROCESSING)
                .event(PaymentEvent.CONFIRM)
            .and()
            // PROCESSING -> SUCCEEDED
            .withExternal()
                .source(PaymentStatus.PROCESSING)
                .target(PaymentStatus.SUCCEEDED)
                .event(PaymentEvent.SUCCEED)
            .and()
            // PROCESSING -> FAILED
            .withExternal()
                .source(PaymentStatus.PROCESSING)
                .target(PaymentStatus.FAILED)
                .event(PaymentEvent.FAIL)
            .and()
            // INITIATED -> CANCELED
            .withExternal()
                .source(PaymentStatus.INITIATED)
                .target(PaymentStatus.CANCELED)
                .event(PaymentEvent.CANCEL)
            .and()
            // SUCCEEDED -> REFUNDING
            .withExternal()
                .source(PaymentStatus.SUCCEEDED)
                .target(PaymentStatus.REFUNDING)
                .event(PaymentEvent.REFUND)
            .and()
            // REFUNDING -> REFUNDED
            .withExternal()
                .source(PaymentStatus.REFUNDING)
                .target(PaymentStatus.REFUNDED)
                .event(PaymentEvent.REFUND_COMPLETE)
            .and()
            // REFUNDING -> PARTIALLY_REFUNDED
            .withExternal()
                .source(PaymentStatus.REFUNDING)
                .target(PaymentStatus.PARTIALLY_REFUNDED)
                .event(PaymentEvent.PARTIAL_REFUND_COMPLETE);
    }
}
```

```java
// statemachine/PaymentEvent.java
package com.example.payment.statemachine;

public enum PaymentEvent {
    PROCESS,
    REQUIRES_ACTION,
    CONFIRM,
    SUCCEED,
    FAIL,
    CANCEL,
    REFUND,
    REFUND_COMPLETE,
    PARTIAL_REFUND_COMPLETE
}
```

---

## ขั้นตอนที่ 2768: Idempotent Payment Operations

```java
// service/IdempotencyService.java
package com.example.payment.service;

import lombok.RequiredArgsConstructor;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.stereotype.Service;

import java.time.Duration;
import java.util.Optional;

@Service
@RequiredArgsConstructor
public class IdempotencyService {

    private final RedisTemplate<String, Object> redisTemplate;
    private static final String KEY_PREFIX = "idempotency:";
    private static final Duration TTL = Duration.ofHours(24);

    // ตรวจสอบว่า request นี้เคยทำแล้วหรือยัง
    public Optional<Object> getStoredResult(String idempotencyKey) {
        Object result = redisTemplate.opsForValue().get(KEY_PREFIX + idempotencyKey);
        return Optional.ofNullable(result);
    }

    // บันทึกผลลัพธ์
    public void storeResult(String idempotencyKey, Object result) {
        redisTemplate.opsForValue().set(
            KEY_PREFIX + idempotencyKey, result, TTL
        );
    }

    // Distributed lock เพื่อป้องกัน concurrent requests
    public boolean tryLock(String idempotencyKey) {
        Boolean success = redisTemplate.opsForValue()
            .setIfAbsent(KEY_PREFIX + "lock:" + idempotencyKey, "locked",
                Duration.ofSeconds(30));
        return Boolean.TRUE.equals(success);
    }

    public void releaseLock(String idempotencyKey) {
        redisTemplate.delete(KEY_PREFIX + "lock:" + idempotencyKey);
    }
}
```

---

## ขั้นตอนที่ 2769: Payment Controller

```java
// controller/PaymentController.java
package com.example.payment.controller;

import com.example.payment.dto.*;
import com.example.payment.service.*;
import jakarta.validation.Valid;
import lombok.RequiredArgsConstructor;
import org.springframework.http.ResponseEntity;
import org.springframework.security.core.annotation.AuthenticationPrincipal;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/v1/payments")
@RequiredArgsConstructor
public class PaymentController {

    private final PaymentService paymentService;
    private final RefundService refundService;
    private final IdempotencyService idempotencyService;

    // สร้าง Payment Intent
    @PostMapping("/intent")
    public ResponseEntity<PaymentIntentResponse> createIntent(
            @Valid @RequestBody CreatePaymentRequest request,
            @RequestHeader(value = "Idempotency-Key", required = false) String idempotencyKey,
            @AuthenticationPrincipal UserDetails user) throws Exception {

        request.setUserId(user.getUsername());
        if (idempotencyKey != null) {
            request.setIdempotencyKey(idempotencyKey);
        }

        // ตรวจสอบ idempotency
        if (idempotencyKey != null) {
            var cached = idempotencyService.getStoredResult(idempotencyKey);
            if (cached.isPresent()) {
                return ResponseEntity.ok((PaymentIntentResponse) cached.get());
            }
        }

        var response = paymentService.createPaymentIntent(request);

        if (idempotencyKey != null) {
            idempotencyService.storeResult(idempotencyKey, response);
        }

        return ResponseEntity.ok(response);
    }

    // ขอ refund
    @PostMapping("/{paymentId}/refund")
    public ResponseEntity<RefundResponse> refund(
            @PathVariable Long paymentId,
            @Valid @RequestBody RefundRequest request) throws Exception {
        return ResponseEntity.ok(refundService.processRefund(paymentId, request));
    }

    // ดูสถานะ payment ของ order
    @GetMapping("/orders/{orderId}")
    public ResponseEntity<PaymentStatusResponse> getPaymentStatus(
            @PathVariable Long orderId) {
        var payment = paymentService.getPaymentByOrderId(orderId);
        return ResponseEntity.ok(new PaymentStatusResponse(
            payment.getId(),
            payment.getStatus().name(),
            payment.getAmount(),
            payment.getCurrency(),
            payment.getCompletedAt()
        ));
    }
}
```

---

## ขั้นตอนที่ 2770: DTO Classes

```java
// dto/CreatePaymentRequest.java
package com.example.payment.dto;

import jakarta.validation.constraints.*;
import lombok.Data;
import java.math.BigDecimal;

@Data
public class CreatePaymentRequest {

    @NotNull(message = "กรุณาระบุ order ID")
    private Long orderId;

    private String userId;

    @NotNull(message = "กรุณาระบุยอดเงิน")
    @DecimalMin(value = "1.00", message = "ยอดเงินต้องมากกว่า 1 บาท")
    @DecimalMax(value = "1000000.00", message = "ยอดเงินสูงสุด 1,000,000 บาท")
    private BigDecimal amount;

    private String idempotencyKey;
    private String returnUrl;
}
```

```java
// dto/RefundRequest.java
package com.example.payment.dto;

import jakarta.validation.constraints.*;
import lombok.Data;
import java.math.BigDecimal;

@Data
public class RefundRequest {

    @DecimalMin(value = "1.00", message = "ยอด refund ต้องมากกว่า 1 บาท")
    private BigDecimal amount;  // null = refund ทั้งหมด

    @NotBlank(message = "กรุณาระบุเหตุผล")
    @Size(max = 500)
    private String reason;
}
```

---

## ขั้นตอนที่ 2771: PCI Compliance Considerations

```java
// security/PaymentSecurityService.java
package com.example.payment.security;

import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;

@Slf4j
@Service
public class PaymentSecurityService {

    /**
     * PCI DSS Compliance notes:
     *
     * 1. ห้าม store card number, CVV, หรือ PIN โดยตรง
     *    - ใช้ Stripe token หรือ payment method ID แทน
     *
     * 2. ต้องเข้ารหัส sensitive data ทั้งหมด
     *    - TLS 1.2+ สำหรับทุก API call
     *    - Encrypt ที่ rest สำหรับ payment records
     *
     * 3. Access control
     *    - Role-based access ไปยัง payment endpoints
     *    - Audit log ทุก payment operation
     *
     * 4. Monitoring
     *    - Alert เมื่อมี unusual payment patterns
     *    - Track failed payment attempts
     */

    // Mask card number สำหรับ logging (เก็บแค่ 4 ตัวท้าย)
    public String maskCardNumber(String cardNumber) {
        if (cardNumber == null || cardNumber.length() < 4) return "****";
        return "**** **** **** " + cardNumber.substring(cardNumber.length() - 4);
    }

    // ตรวจสอบว่า IP ไม่ได้ถูก blacklist
    public boolean isIpAllowed(String ipAddress) {
        // ตรวจสอบจาก IP blacklist
        return true; // placeholder
    }

    // ตรวจสอบ velocity limit (ป้องกัน fraud)
    public boolean checkVelocityLimit(String userId, java.math.BigDecimal amount) {
        // ตรวจสอบว่า user ไม่ได้ชำระเงินมากเกินไปใน timeframe
        return true; // placeholder
    }
}
```

```java
// repository/PaymentRepository.java
package com.example.payment.repository;

import com.example.payment.entity.Payment;
import com.example.payment.entity.PaymentStatus;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;

import java.time.LocalDateTime;
import java.util.List;
import java.util.Optional;

public interface PaymentRepository extends JpaRepository<Payment, Long> {

    Optional<Payment> findByIdempotencyKey(String idempotencyKey);

    boolean existsByIdempotencyKey(String idempotencyKey);

    Optional<Payment> findByStripePaymentIntentId(String intentId);

    Optional<Payment> findByStripeChargeId(String chargeId);

    Optional<Payment> findByOrderId(Long orderId);

    List<Payment> findByUserIdAndStatus(String userId, PaymentStatus status);

    @Query("SELECT p FROM Payment p WHERE p.status = :status AND p.createdAt < :before")
    List<Payment> findStalePayments(PaymentStatus status, LocalDateTime before);
}
```

---

## ขั้นตอนที่ 2772: ทดสอบ Payment Service

```java
// test/PaymentServiceTest.java
package com.example.payment.service;

import com.example.payment.dto.CreatePaymentRequest;
import com.example.payment.entity.Payment;
import com.example.payment.entity.PaymentStatus;
import com.example.payment.repository.PaymentRepository;
import com.stripe.model.PaymentIntent;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.*;
import org.mockito.junit.jupiter.MockitoExtension;

import java.math.BigDecimal;
import java.util.Optional;

import static org.assertj.core.api.Assertions.*;
import static org.mockito.ArgumentMatchers.*;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)
class PaymentServiceTest {

    @Mock
    private PaymentRepository paymentRepository;

    @InjectMocks
    private PaymentService paymentService;

    @Test
    void shouldReturnExistingPaymentForDuplicateRequest() throws Exception {
        // Given
        String idempotencyKey = "order-123-uuid";
        Payment existingPayment = Payment.builder()
            .stripePaymentIntentId("pi_existing")
            .status(PaymentStatus.INITIATED)
            .amount(new BigDecimal("1000.00"))
            .build();

        when(paymentRepository.existsByIdempotencyKey(idempotencyKey)).thenReturn(true);
        when(paymentRepository.findByIdempotencyKey(idempotencyKey))
            .thenReturn(Optional.of(existingPayment));

        CreatePaymentRequest request = new CreatePaymentRequest();
        request.setOrderId(123L);
        request.setUserId("user-1");
        request.setAmount(new BigDecimal("1000.00"));
        request.setIdempotencyKey(idempotencyKey);

        // When
        var response = paymentService.createPaymentIntent(request);

        // Then
        assertThat(response.getPaymentIntentId()).isEqualTo("pi_existing");
        assertThat(response.getClientSecret()).isNull(); // ไม่ return client secret ซ้ำ
        verify(paymentRepository, never()).save(any()); // ไม่ save ใหม่
    }
}
```

---

## สรุป

ในส่วนนี้เราได้เรียนรู้:

1. **Stripe Integration** - สร้าง Payment Intent และจัดการ lifecycle
2. **Webhook Handling** - รับและ verify Stripe webhook ด้วย signature
3. **Idempotency** - ป้องกัน duplicate payment ด้วย idempotency key
4. **Payment State Machine** - จัดการ state transition ด้วย Spring State Machine
5. **Refunds** - จัดการ refund และ partial refund
6. **Dispute Handling** - รับ dispute notification จาก Stripe
7. **PCI Compliance** - Best practices สำหรับ card data security

---

*[← Part 78: Email & Notifications](./part-78-email-notifications.md) | [Part 80: Real-time Features →](./part-80-realtime-features.md)*

# Part 111: โปรเจค 51-55 — Loyalty & Marketing Systems

**ระดับ:** ระดับโลก (World-Class)
**เวลา:** 10-15 ชั่วโมง
**เป้าหมาย:** สร้างระบบ Loyalty และ Marketing ที่ครบครัน ตั้งแต่ Points System, Gift Cards, Referral Programs, Affiliate Marketing จนถึง Price Comparison Service

---

*[← Part 110](./part-110-ecommerce-advanced.md) | [Part 112: Food & Health →](./part-112-food-health.md)*

---

## โปรเจค 51: Loyalty Points System

### ภาพรวมระบบ

ระบบ Loyalty Points เป็นหัวใจสำคัญของธุรกิจ e-commerce สมัยใหม่ ช่วยให้ลูกค้ากลับมาซื้อซ้ำโดยการสะสมแต้มจากการซื้อสินค้า แลกคะแนนเป็นส่วนลด และเลื่อนระดับสมาชิกตามยอดการใช้งาน ระบบนี้รองรับ:

- **การสะสมแต้ม**: คำนวณแต้มจากยอดซื้อตามอัตราแต่ละ Tier
- **การแลกแต้ม**: แลกแต้มเป็นส่วนลดหรือของรางวัล
- **ระดับสมาชิก**: Bronze / Silver / Gold / Platinum พร้อม Benefits
- **วันหมดอายุแต้ม**: แต้มหมดอายุตามนโยบายที่กำหนด
- **การโอนแต้ม**: โอนแต้มระหว่างสมาชิก
- **Leaderboard**: จัดอันดับสมาชิกที่มีแต้มสูงสุด

### Flyway Migration

```sql
-- V1__create_loyalty_tables.sql
CREATE TABLE loyalty_members (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL UNIQUE,
    member_number VARCHAR(20) NOT NULL UNIQUE,
    tier VARCHAR(20) NOT NULL DEFAULT 'BRONZE',
    total_points_earned BIGINT NOT NULL DEFAULT 0,
    available_points BIGINT NOT NULL DEFAULT 0,
    lifetime_spend DECIMAL(15,2) NOT NULL DEFAULT 0,
    tier_progress DECIMAL(5,2) NOT NULL DEFAULT 0,
    tier_expiry_date DATE,
    enrolled_at TIMESTAMP NOT NULL DEFAULT NOW(),
    last_activity_at TIMESTAMP,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE loyalty_transactions (
    id BIGSERIAL PRIMARY KEY,
    member_id BIGINT NOT NULL REFERENCES loyalty_members(id),
    transaction_type VARCHAR(30) NOT NULL,
    points BIGINT NOT NULL,
    balance_after BIGINT NOT NULL,
    reference_type VARCHAR(50),
    reference_id VARCHAR(100),
    description VARCHAR(500),
    expiry_date DATE,
    expired BOOLEAN NOT NULL DEFAULT FALSE,
    partner_id BIGINT,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE loyalty_tiers (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(20) NOT NULL UNIQUE,
    min_spend DECIMAL(15,2) NOT NULL,
    max_spend DECIMAL(15,2),
    earn_rate DECIMAL(5,3) NOT NULL,
    bonus_multiplier DECIMAL(5,2) NOT NULL DEFAULT 1.0,
    benefits JSONB,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE loyalty_partners (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    code VARCHAR(50) NOT NULL UNIQUE,
    earn_rate DECIMAL(5,3) NOT NULL DEFAULT 1.0,
    active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE point_transfers (
    id BIGSERIAL PRIMARY KEY,
    from_member_id BIGINT NOT NULL REFERENCES loyalty_members(id),
    to_member_id BIGINT NOT NULL REFERENCES loyalty_members(id),
    points BIGINT NOT NULL,
    fee_points BIGINT NOT NULL DEFAULT 0,
    status VARCHAR(20) NOT NULL DEFAULT 'COMPLETED',
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

INSERT INTO loyalty_tiers (name, min_spend, max_spend, earn_rate, bonus_multiplier, benefits) VALUES
('BRONZE',   0,       999.99,  1.0, 1.0, '{"birthday_bonus": 100, "free_shipping": false}'),
('SILVER',   1000,    4999.99, 1.5, 1.2, '{"birthday_bonus": 200, "free_shipping": true, "priority_support": false}'),
('GOLD',     5000,    19999.99,2.0, 1.5, '{"birthday_bonus": 500, "free_shipping": true, "priority_support": true}'),
('PLATINUM', 20000,   NULL,    3.0, 2.0, '{"birthday_bonus": 1000, "free_shipping": true, "priority_support": true, "vip_events": true}');
```

### Entity

```java
// LoyaltyMember.java
package com.loyalty.entity;

import jakarta.persistence.*;
import lombok.*;
import org.hibernate.annotations.JdbcTypeCode;
import org.hibernate.type.SqlTypes;
import java.math.BigDecimal;
import java.time.LocalDate;
import java.time.LocalDateTime;

@Entity
@Table(name = "loyalty_members")
@Getter @Setter @Builder
@NoArgsConstructor @AllArgsConstructor
public class LoyaltyMember {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "user_id", nullable = false, unique = true)
    private Long userId;

    @Column(name = "member_number", nullable = false, unique = true)
    private String memberNumber;

    @Enumerated(EnumType.STRING)
    @Column(name = "tier", nullable = false)
    private TierLevel tier = TierLevel.BRONZE;

    @Column(name = "total_points_earned", nullable = false)
    private Long totalPointsEarned = 0L;

    @Column(name = "available_points", nullable = false)
    private Long availablePoints = 0L;

    @Column(name = "lifetime_spend", nullable = false)
    private BigDecimal lifetimeSpend = BigDecimal.ZERO;

    @Column(name = "tier_expiry_date")
    private LocalDate tierExpiryDate;

    @Column(name = "enrolled_at", nullable = false)
    private LocalDateTime enrolledAt = LocalDateTime.now();

    @Column(name = "last_activity_at")
    private LocalDateTime lastActivityAt;

    @Column(name = "updated_at")
    private LocalDateTime updatedAt;
}

// LoyaltyTransaction.java
@Entity
@Table(name = "loyalty_transactions")
@Getter @Setter @Builder
@NoArgsConstructor @AllArgsConstructor
public class LoyaltyTransaction {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "member_id", nullable = false)
    private LoyaltyMember member;

    @Enumerated(EnumType.STRING)
    @Column(name = "transaction_type", nullable = false)
    private TransactionType transactionType;

    @Column(name = "points", nullable = false)
    private Long points;

    @Column(name = "balance_after", nullable = false)
    private Long balanceAfter;

    @Column(name = "reference_type")
    private String referenceType;

    @Column(name = "reference_id")
    private String referenceId;

    @Column(name = "description")
    private String description;

    @Column(name = "expiry_date")
    private LocalDate expiryDate;

    @Column(name = "expired")
    private Boolean expired = false;

    @Column(name = "created_at", nullable = false)
    private LocalDateTime createdAt = LocalDateTime.now();
}

// TierLevel.java
public enum TierLevel {
    BRONZE, SILVER, GOLD, PLATINUM
}

// TransactionType.java
public enum TransactionType {
    EARN_PURCHASE, EARN_PARTNER, EARN_BONUS, EARN_REFERRAL,
    REDEEM, EXPIRE, TRANSFER_OUT, TRANSFER_IN, ADJUST
}
```

### Repository

```java
// LoyaltyMemberRepository.java
package com.loyalty.repository;

import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import java.util.List;
import java.util.Optional;

public interface LoyaltyMemberRepository extends JpaRepository<LoyaltyMember, Long> {
    Optional<LoyaltyMember> findByUserId(Long userId);
    Optional<LoyaltyMember> findByMemberNumber(String memberNumber);

    @Query("SELECT m FROM LoyaltyMember m ORDER BY m.availablePoints DESC LIMIT :limit")
    List<LoyaltyMember> findTopByAvailablePoints(int limit);

    @Query("SELECT m FROM LoyaltyMember m WHERE m.tier = :tier ORDER BY m.availablePoints DESC")
    List<LoyaltyMember> findByTierOrderByPoints(TierLevel tier);
}

// LoyaltyTransactionRepository.java
public interface LoyaltyTransactionRepository extends JpaRepository<LoyaltyTransaction, Long> {
    List<LoyaltyTransaction> findByMemberIdOrderByCreatedAtDesc(Long memberId);

    @Query("SELECT t FROM LoyaltyTransaction t WHERE t.member.id = :memberId " +
           "AND t.expired = false AND t.expiryDate <= :expiryDate")
    List<LoyaltyTransaction> findExpiringPoints(Long memberId, LocalDate expiryDate);
}
```

### Service

```java
// LoyaltyService.java
package com.loyalty.service;

import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import java.math.BigDecimal;
import java.math.RoundingMode;
import java.time.LocalDate;
import java.time.LocalDateTime;
import java.util.*;

@Service
@RequiredArgsConstructor
public class LoyaltyService {

    private final LoyaltyMemberRepository memberRepository;
    private final LoyaltyTransactionRepository transactionRepository;
    private final LoyaltyTierRepository tierRepository;
    private final LoyaltyPartnerRepository partnerRepository;

    @Transactional
    public LoyaltyMember enroll(Long userId) {
        if (memberRepository.findByUserId(userId).isPresent()) {
            throw new IllegalStateException("User already enrolled in loyalty program");
        }
        String memberNumber = "LM" + System.currentTimeMillis();
        LoyaltyMember member = LoyaltyMember.builder()
                .userId(userId)
                .memberNumber(memberNumber)
                .tier(TierLevel.BRONZE)
                .availablePoints(0L)
                .totalPointsEarned(0L)
                .lifetimeSpend(BigDecimal.ZERO)
                .enrolledAt(LocalDateTime.now())
                .build();
        return memberRepository.save(member);
    }

    @Transactional
    public LoyaltyTransaction earnPoints(Long userId, BigDecimal purchaseAmount, String orderId) {
        LoyaltyMember member = getMember(userId);
        LoyaltyTier tier = tierRepository.findByName(member.getTier().name())
                .orElseThrow(() -> new RuntimeException("Tier not found"));

        long pointsEarned = purchaseAmount
                .multiply(BigDecimal.valueOf(tier.getEarnRate()))
                .setScale(0, RoundingMode.DOWN)
                .longValueExact();

        member.setAvailablePoints(member.getAvailablePoints() + pointsEarned);
        member.setTotalPointsEarned(member.getTotalPointsEarned() + pointsEarned);
        member.setLifetimeSpend(member.getLifetimeSpend().add(purchaseAmount));
        member.setLastActivityAt(LocalDateTime.now());
        member.setUpdatedAt(LocalDateTime.now());

        updateTier(member);
        memberRepository.save(member);

        LoyaltyTransaction tx = LoyaltyTransaction.builder()
                .member(member)
                .transactionType(TransactionType.EARN_PURCHASE)
                .points(pointsEarned)
                .balanceAfter(member.getAvailablePoints())
                .referenceType("ORDER")
                .referenceId(orderId)
                .description("Points earned from purchase: " + orderId)
                .expiryDate(LocalDate.now().plusYears(1))
                .build();
        return transactionRepository.save(tx);
    }

    @Transactional
    public LoyaltyTransaction redeemPoints(Long userId, Long pointsToRedeem, String redemptionRef) {
        LoyaltyMember member = getMember(userId);
        if (member.getAvailablePoints() < pointsToRedeem) {
            throw new IllegalArgumentException("Insufficient points. Available: " + member.getAvailablePoints());
        }

        member.setAvailablePoints(member.getAvailablePoints() - pointsToRedeem);
        member.setLastActivityAt(LocalDateTime.now());
        member.setUpdatedAt(LocalDateTime.now());
        memberRepository.save(member);

        LoyaltyTransaction tx = LoyaltyTransaction.builder()
                .member(member)
                .transactionType(TransactionType.REDEEM)
                .points(-pointsToRedeem)
                .balanceAfter(member.getAvailablePoints())
                .referenceType("REDEMPTION")
                .referenceId(redemptionRef)
                .description("Points redeemed: " + pointsToRedeem + " pts")
                .build();
        return transactionRepository.save(tx);
    }

    @Transactional
    public PointTransferResult transferPoints(Long fromUserId, Long toUserId, Long points) {
        LoyaltyMember from = getMember(fromUserId);
        LoyaltyMember to = getMember(toUserId);

        long fee = (long) (points * 0.05); // 5% transfer fee
        long totalDeduction = points + fee;

        if (from.getAvailablePoints() < totalDeduction) {
            throw new IllegalArgumentException("Insufficient points including transfer fee");
        }

        from.setAvailablePoints(from.getAvailablePoints() - totalDeduction);
        to.setAvailablePoints(to.getAvailablePoints() + points);
        from.setUpdatedAt(LocalDateTime.now());
        to.setUpdatedAt(LocalDateTime.now());

        memberRepository.saveAll(List.of(from, to));

        return new PointTransferResult(points, fee, from.getAvailablePoints(), to.getAvailablePoints());
    }

    @Transactional
    public void expirePoints() {
        LocalDate today = LocalDate.now();
        // Find all non-expired transactions past their expiry date
        List<LoyaltyMember> allMembers = memberRepository.findAll();
        for (LoyaltyMember member : allMembers) {
            List<LoyaltyTransaction> expiringTxs =
                transactionRepository.findExpiringPoints(member.getId(), today);
            for (LoyaltyTransaction tx : expiringTxs) {
                long expiredPoints = tx.getPoints();
                member.setAvailablePoints(
                    Math.max(0, member.getAvailablePoints() - expiredPoints));
                tx.setExpired(true);
                transactionRepository.save(tx);
            }
            if (!expiringTxs.isEmpty()) {
                member.setUpdatedAt(LocalDateTime.now());
                memberRepository.save(member);
            }
        }
    }

    public List<LeaderboardEntry> getLeaderboard(int limit) {
        List<LoyaltyMember> topMembers = memberRepository.findTopByAvailablePoints(limit);
        List<LeaderboardEntry> leaderboard = new ArrayList<>();
        int rank = 1;
        for (LoyaltyMember m : topMembers) {
            leaderboard.add(new LeaderboardEntry(rank++, m.getMemberNumber(),
                m.getTier(), m.getAvailablePoints()));
        }
        return leaderboard;
    }

    private void updateTier(LoyaltyMember member) {
        BigDecimal spend = member.getLifetimeSpend();
        TierLevel newTier;
        if (spend.compareTo(BigDecimal.valueOf(20000)) >= 0) newTier = TierLevel.PLATINUM;
        else if (spend.compareTo(BigDecimal.valueOf(5000)) >= 0) newTier = TierLevel.GOLD;
        else if (spend.compareTo(BigDecimal.valueOf(1000)) >= 0) newTier = TierLevel.SILVER;
        else newTier = TierLevel.BRONZE;
        member.setTier(newTier);
        if (newTier != TierLevel.BRONZE) {
            member.setTierExpiryDate(LocalDate.now().plusYears(1));
        }
    }

    private LoyaltyMember getMember(Long userId) {
        return memberRepository.findByUserId(userId)
                .orElseThrow(() -> new RuntimeException("Member not found for userId: " + userId));
    }

    public record PointTransferResult(long points, long fee, long fromBalance, long toBalance) {}
    public record LeaderboardEntry(int rank, String memberNumber, TierLevel tier, long points) {}
}
```

### Controller

```java
// LoyaltyController.java
package com.loyalty.controller;

import lombok.RequiredArgsConstructor;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import java.math.BigDecimal;
import java.util.List;

@RestController
@RequestMapping("/api/v1/loyalty")
@RequiredArgsConstructor
public class LoyaltyController {

    private final LoyaltyService loyaltyService;

    @PostMapping("/enroll")
    public ResponseEntity<LoyaltyMember> enroll(@RequestParam Long userId) {
        return ResponseEntity.ok(loyaltyService.enroll(userId));
    }

    @PostMapping("/earn")
    public ResponseEntity<LoyaltyTransaction> earnPoints(@RequestBody EarnPointsRequest request) {
        return ResponseEntity.ok(loyaltyService.earnPoints(
            request.userId(), request.purchaseAmount(), request.orderId()));
    }

    @PostMapping("/redeem")
    public ResponseEntity<LoyaltyTransaction> redeemPoints(@RequestBody RedeemPointsRequest request) {
        return ResponseEntity.ok(loyaltyService.redeemPoints(
            request.userId(), request.points(), request.redemptionRef()));
    }

    @PostMapping("/transfer")
    public ResponseEntity<LoyaltyService.PointTransferResult> transferPoints(
            @RequestBody TransferPointsRequest request) {
        return ResponseEntity.ok(loyaltyService.transferPoints(
            request.fromUserId(), request.toUserId(), request.points()));
    }

    @GetMapping("/leaderboard")
    public ResponseEntity<List<LoyaltyService.LeaderboardEntry>> getLeaderboard(
            @RequestParam(defaultValue = "10") int limit) {
        return ResponseEntity.ok(loyaltyService.getLeaderboard(limit));
    }

    @GetMapping("/members/{userId}")
    public ResponseEntity<LoyaltyMember> getMember(@PathVariable Long userId) {
        return ResponseEntity.ok(loyaltyService.getMember(userId));
    }

    @GetMapping("/members/{userId}/transactions")
    public ResponseEntity<List<LoyaltyTransaction>> getTransactions(@PathVariable Long userId) {
        return ResponseEntity.ok(loyaltyService.getTransactions(userId));
    }

    record EarnPointsRequest(Long userId, BigDecimal purchaseAmount, String orderId) {}
    record RedeemPointsRequest(Long userId, Long points, String redemptionRef) {}
    record TransferPointsRequest(Long fromUserId, Long toUserId, Long points) {}
}
```

### docker-compose.yml

```yaml
# docker-compose.yml (Loyalty Points System)
version: '3.8'
services:
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: loyalty_db
      POSTGRES_USER: loyalty_user
      POSTGRES_PASSWORD: loyalty_pass
    ports:
      - "5432:5432"
    volumes:
      - loyalty_pg_data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

  loyalty-service:
    build: .
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/loyalty_db
      SPRING_DATASOURCE_USERNAME: loyalty_user
      SPRING_DATASOURCE_PASSWORD: loyalty_pass
      SPRING_REDIS_HOST: redis
      SPRING_REDIS_PORT: 6379
    ports:
      - "8080:8080"
    depends_on:
      - postgres
      - redis

volumes:
  loyalty_pg_data:
```

---

## โปรเจค 52: Gift Card Management

### ภาพรวมระบบ

ระบบจัดการ Gift Card ช่วยให้ธุรกิจสามารถออก Gift Card ในรูปแบบดิจิทัล สามารถซื้อ ส่งทางอีเมล และใช้งานในระบบ checkout ได้อย่างสะดวก ระบบรองรับ:

- **การสร้าง Gift Card**: ออก Gift Card ด้วยมูลค่าต่าง ๆ
- **การจัดการยอดคงเหลือ**: ตรวจสอบและอัปเดต Balance
- **การซื้อ Gift Card**: Flow การซื้อพร้อม Payment Integration
- **การส่งอีเมล**: ส่ง Gift Card ให้ผู้รับผ่านอีเมล
- **การใช้งานที่ Checkout**: ใช้บางส่วนหรือทั้งหมด
- **การออก Bulk**: ออก Gift Card จำนวนมากสำหรับโปรโมชัน

### Flyway Migration

```sql
-- V1__create_gift_card_tables.sql
CREATE TABLE gift_card_designs (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    description TEXT,
    image_url VARCHAR(500),
    active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE gift_cards (
    id BIGSERIAL PRIMARY KEY,
    code VARCHAR(50) NOT NULL UNIQUE,
    initial_amount DECIMAL(10,2) NOT NULL,
    current_balance DECIMAL(10,2) NOT NULL,
    currency VARCHAR(3) NOT NULL DEFAULT 'THB',
    status VARCHAR(20) NOT NULL DEFAULT 'ACTIVE',
    design_id BIGINT REFERENCES gift_card_designs(id),
    purchased_by_user_id BIGINT,
    recipient_email VARCHAR(255),
    recipient_name VARCHAR(200),
    personal_message TEXT,
    expiry_date DATE,
    activated_at TIMESTAMP,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE gift_card_transactions (
    id BIGSERIAL PRIMARY KEY,
    gift_card_id BIGINT NOT NULL REFERENCES gift_cards(id),
    transaction_type VARCHAR(20) NOT NULL,
    amount DECIMAL(10,2) NOT NULL,
    balance_after DECIMAL(10,2) NOT NULL,
    order_id VARCHAR(100),
    user_id BIGINT,
    description VARCHAR(500),
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE bulk_gift_card_batches (
    id BIGSERIAL PRIMARY KEY,
    batch_name VARCHAR(200) NOT NULL,
    quantity INT NOT NULL,
    amount_per_card DECIMAL(10,2) NOT NULL,
    total_amount DECIMAL(12,2) NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'PROCESSING',
    created_by BIGINT,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    completed_at TIMESTAMP
);

INSERT INTO gift_card_designs (name, description, image_url) VALUES
('Classic', 'Classic gift card design', '/images/gift-card-classic.jpg'),
('Birthday', 'Happy Birthday theme', '/images/gift-card-birthday.jpg'),
('Holiday', 'Holiday season theme', '/images/gift-card-holiday.jpg'),
('Corporate', 'Corporate gift design', '/images/gift-card-corporate.jpg');
```

### Entity & Service

```java
// GiftCard.java
package com.giftcard.entity;

import jakarta.persistence.*;
import lombok.*;
import java.math.BigDecimal;
import java.time.LocalDate;
import java.time.LocalDateTime;

@Entity
@Table(name = "gift_cards")
@Getter @Setter @Builder
@NoArgsConstructor @AllArgsConstructor
public class GiftCard {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "code", nullable = false, unique = true)
    private String code;

    @Column(name = "initial_amount", nullable = false)
    private BigDecimal initialAmount;

    @Column(name = "current_balance", nullable = false)
    private BigDecimal currentBalance;

    @Column(name = "currency", nullable = false)
    private String currency = "THB";

    @Enumerated(EnumType.STRING)
    @Column(name = "status", nullable = false)
    private GiftCardStatus status = GiftCardStatus.ACTIVE;

    @Column(name = "purchased_by_user_id")
    private Long purchasedByUserId;

    @Column(name = "recipient_email")
    private String recipientEmail;

    @Column(name = "recipient_name")
    private String recipientName;

    @Column(name = "personal_message")
    private String personalMessage;

    @Column(name = "expiry_date")
    private LocalDate expiryDate;

    @Column(name = "activated_at")
    private LocalDateTime activatedAt;

    @Column(name = "created_at", nullable = false)
    private LocalDateTime createdAt = LocalDateTime.now();

    @Column(name = "updated_at")
    private LocalDateTime updatedAt;
}

public enum GiftCardStatus { INACTIVE, ACTIVE, PARTIALLY_USED, DEPLETED, EXPIRED, CANCELLED }

// GiftCardService.java
package com.giftcard.service;

import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import java.math.BigDecimal;
import java.security.SecureRandom;
import java.time.LocalDate;
import java.time.LocalDateTime;
import java.util.*;

@Service
@RequiredArgsConstructor
public class GiftCardService {

    private final GiftCardRepository giftCardRepository;
    private final GiftCardTransactionRepository txRepository;
    private final EmailService emailService;
    private static final SecureRandom RANDOM = new SecureRandom();

    @Transactional
    public GiftCard createGiftCard(BigDecimal amount, String currency, Long purchasedByUserId,
                                    String recipientEmail, String recipientName,
                                    String personalMessage, Long designId) {
        String code = generateCode();
        GiftCard card = GiftCard.builder()
                .code(code)
                .initialAmount(amount)
                .currentBalance(amount)
                .currency(currency)
                .status(GiftCardStatus.ACTIVE)
                .purchasedByUserId(purchasedByUserId)
                .recipientEmail(recipientEmail)
                .recipientName(recipientName)
                .personalMessage(personalMessage)
                .expiryDate(LocalDate.now().plusYears(2))
                .activatedAt(LocalDateTime.now())
                .createdAt(LocalDateTime.now())
                .build();
        card = giftCardRepository.save(card);

        if (recipientEmail != null) {
            emailService.sendGiftCard(card);
        }
        return card;
    }

    @Transactional
    public GiftCardRedemptionResult redeem(String code, BigDecimal amountToUse, String orderId, Long userId) {
        GiftCard card = giftCardRepository.findByCode(code)
                .orElseThrow(() -> new RuntimeException("Gift card not found: " + code));

        if (card.getStatus() == GiftCardStatus.EXPIRED ||
            (card.getExpiryDate() != null && card.getExpiryDate().isBefore(LocalDate.now()))) {
            throw new IllegalStateException("Gift card has expired");
        }
        if (card.getStatus() == GiftCardStatus.DEPLETED ||
            card.getStatus() == GiftCardStatus.CANCELLED) {
            throw new IllegalStateException("Gift card is not valid: " + card.getStatus());
        }
        if (card.getCurrentBalance().compareTo(BigDecimal.ZERO) <= 0) {
            throw new IllegalStateException("Gift card has no balance");
        }

        BigDecimal actualAmount = amountToUse.min(card.getCurrentBalance());
        card.setCurrentBalance(card.getCurrentBalance().subtract(actualAmount));
        card.setUpdatedAt(LocalDateTime.now());

        if (card.getCurrentBalance().compareTo(BigDecimal.ZERO) == 0) {
            card.setStatus(GiftCardStatus.DEPLETED);
        } else if (actualAmount.compareTo(card.getInitialAmount()) < 0) {
            card.setStatus(GiftCardStatus.PARTIALLY_USED);
        }
        giftCardRepository.save(card);

        GiftCardTransaction tx = GiftCardTransaction.builder()
                .giftCard(card)
                .transactionType("REDEEM")
                .amount(actualAmount)
                .balanceAfter(card.getCurrentBalance())
                .orderId(orderId)
                .userId(userId)
                .description("Redemption for order: " + orderId)
                .build();
        txRepository.save(tx);

        return new GiftCardRedemptionResult(actualAmount, card.getCurrentBalance(),
            amountToUse.subtract(actualAmount));
    }

    @Transactional
    public List<GiftCard> bulkCreate(int quantity, BigDecimal amountPerCard, String currency,
                                      Long createdBy) {
        List<GiftCard> cards = new ArrayList<>();
        for (int i = 0; i < quantity; i++) {
            GiftCard card = GiftCard.builder()
                    .code(generateCode())
                    .initialAmount(amountPerCard)
                    .currentBalance(amountPerCard)
                    .currency(currency)
                    .status(GiftCardStatus.INACTIVE)
                    .purchasedByUserId(createdBy)
                    .expiryDate(LocalDate.now().plusYears(1))
                    .createdAt(LocalDateTime.now())
                    .build();
            cards.add(card);
        }
        return giftCardRepository.saveAll(cards);
    }

    public GiftCardBalanceResult checkBalance(String code) {
        GiftCard card = giftCardRepository.findByCode(code)
                .orElseThrow(() -> new RuntimeException("Gift card not found"));
        return new GiftCardBalanceResult(card.getCode(), card.getCurrentBalance(),
            card.getCurrency(), card.getStatus(), card.getExpiryDate());
    }

    private String generateCode() {
        String chars = "ABCDEFGHJKLMNPQRSTUVWXYZ23456789";
        StringBuilder sb = new StringBuilder("GC-");
        for (int i = 0; i < 4; i++) {
            for (int j = 0; j < 4; j++) sb.append(chars.charAt(RANDOM.nextInt(chars.length())));
            if (i < 3) sb.append("-");
        }
        return sb.toString();
    }

    public record GiftCardRedemptionResult(BigDecimal amountUsed, BigDecimal remainingBalance, BigDecimal remaining) {}
    public record GiftCardBalanceResult(String code, BigDecimal balance, String currency,
                                         GiftCardStatus status, LocalDate expiryDate) {}
}
```

### Controller

```java
// GiftCardController.java
package com.giftcard.controller;

import lombok.RequiredArgsConstructor;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import java.math.BigDecimal;
import java.util.List;

@RestController
@RequestMapping("/api/v1/gift-cards")
@RequiredArgsConstructor
public class GiftCardController {

    private final GiftCardService giftCardService;

    @PostMapping
    public ResponseEntity<GiftCard> create(@RequestBody CreateGiftCardRequest request) {
        return ResponseEntity.ok(giftCardService.createGiftCard(
            request.amount(), request.currency(), request.purchasedByUserId(),
            request.recipientEmail(), request.recipientName(),
            request.personalMessage(), request.designId()));
    }

    @GetMapping("/{code}/balance")
    public ResponseEntity<GiftCardService.GiftCardBalanceResult> checkBalance(
            @PathVariable String code) {
        return ResponseEntity.ok(giftCardService.checkBalance(code));
    }

    @PostMapping("/{code}/redeem")
    public ResponseEntity<GiftCardService.GiftCardRedemptionResult> redeem(
            @PathVariable String code, @RequestBody RedeemRequest request) {
        return ResponseEntity.ok(giftCardService.redeem(
            code, request.amount(), request.orderId(), request.userId()));
    }

    @PostMapping("/bulk")
    public ResponseEntity<List<GiftCard>> bulkCreate(@RequestBody BulkCreateRequest request) {
        return ResponseEntity.ok(giftCardService.bulkCreate(
            request.quantity(), request.amountPerCard(),
            request.currency(), request.createdBy()));
    }

    record CreateGiftCardRequest(BigDecimal amount, String currency, Long purchasedByUserId,
                                  String recipientEmail, String recipientName,
                                  String personalMessage, Long designId) {}
    record RedeemRequest(BigDecimal amount, String orderId, Long userId) {}
    record BulkCreateRequest(int quantity, BigDecimal amountPerCard, String currency, Long createdBy) {}
}
```

### docker-compose.yml

```yaml
# docker-compose.yml (Gift Card Management)
version: '3.8'
services:
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: giftcard_db
      POSTGRES_USER: giftcard_user
      POSTGRES_PASSWORD: giftcard_pass
    ports:
      - "5432:5432"
    volumes:
      - giftcard_pg_data:/var/lib/postgresql/data

  mailhog:
    image: mailhog/mailhog
    ports:
      - "1025:1025"
      - "8025:8025"

  giftcard-service:
    build: .
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/giftcard_db
      SPRING_DATASOURCE_USERNAME: giftcard_user
      SPRING_DATASOURCE_PASSWORD: giftcard_pass
      SPRING_MAIL_HOST: mailhog
      SPRING_MAIL_PORT: 1025
    ports:
      - "8080:8080"
    depends_on:
      - postgres
      - mailhog

volumes:
  giftcard_pg_data:
```

---

## โปรเจค 53: Referral Program System

### ภาพรวมระบบ

ระบบ Referral Program ช่วยให้ธุรกิจเติบโตโดยใช้เครือข่ายลูกค้าเดิม ผู้ใช้งานแต่ละคนจะได้รับ Referral Code เฉพาะตัว เมื่อมีคนสมัครผ่าน Code นั้นและทำการซื้อ ทั้งผู้แนะนำและผู้ถูกแนะนำจะได้รับรางวัล ระบบยังมีการตรวจจับการโกงด้วยการตรวจสอบ IP และ Device

- **Referral Codes**: Code เฉพาะตัวสำหรับแต่ละผู้ใช้
- **Tracking**: ติดตามการ Click และการสมัคร
- **Rewards**: รางวัลเป็น Cash / Credits / Discount
- **Multi-level**: รองรับ Referral หลายชั้น
- **Fraud Detection**: ตรวจจับ IP/Device ซ้ำ

### Flyway Migration

```sql
-- V1__create_referral_tables.sql
CREATE TABLE referral_programs (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    description TEXT,
    referrer_reward_type VARCHAR(20) NOT NULL,
    referrer_reward_value DECIMAL(10,2) NOT NULL,
    referee_reward_type VARCHAR(20) NOT NULL,
    referee_reward_value DECIMAL(10,2) NOT NULL,
    min_purchase_amount DECIMAL(10,2),
    max_reward_per_user DECIMAL(10,2),
    active BOOLEAN NOT NULL DEFAULT TRUE,
    start_date TIMESTAMP,
    end_date TIMESTAMP,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE referral_codes (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL UNIQUE,
    code VARCHAR(20) NOT NULL UNIQUE,
    program_id BIGINT NOT NULL REFERENCES referral_programs(id),
    total_referrals INT NOT NULL DEFAULT 0,
    successful_referrals INT NOT NULL DEFAULT 0,
    total_rewards_earned DECIMAL(12,2) NOT NULL DEFAULT 0,
    active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE referrals (
    id BIGSERIAL PRIMARY KEY,
    referral_code_id BIGINT NOT NULL REFERENCES referral_codes(id),
    referee_user_id BIGINT,
    referee_email VARCHAR(255),
    status VARCHAR(20) NOT NULL DEFAULT 'PENDING',
    ip_address VARCHAR(45),
    device_fingerprint VARCHAR(200),
    user_agent TEXT,
    click_at TIMESTAMP NOT NULL DEFAULT NOW(),
    converted_at TIMESTAMP,
    first_purchase_at TIMESTAMP,
    first_purchase_amount DECIMAL(10,2),
    referrer_reward_issued BOOLEAN NOT NULL DEFAULT FALSE,
    referee_reward_issued BOOLEAN NOT NULL DEFAULT FALSE,
    fraud_flagged BOOLEAN NOT NULL DEFAULT FALSE,
    fraud_reason VARCHAR(500),
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE referral_rewards (
    id BIGSERIAL PRIMARY KEY,
    referral_id BIGINT NOT NULL REFERENCES referrals(id),
    user_id BIGINT NOT NULL,
    reward_type VARCHAR(20) NOT NULL,
    reward_value DECIMAL(10,2) NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'PENDING',
    issued_at TIMESTAMP,
    expires_at TIMESTAMP,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_referral_codes_user ON referral_codes(user_id);
CREATE INDEX idx_referrals_code ON referrals(referral_code_id);
CREATE INDEX idx_referrals_ip ON referrals(ip_address);
CREATE INDEX idx_referrals_device ON referrals(device_fingerprint);
```

### Entity & Service

```java
// ReferralCode.java
package com.referral.entity;

import jakarta.persistence.*;
import lombok.*;
import java.math.BigDecimal;
import java.time.LocalDateTime;

@Entity
@Table(name = "referral_codes")
@Getter @Setter @Builder
@NoArgsConstructor @AllArgsConstructor
public class ReferralCode {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "user_id", nullable = false, unique = true)
    private Long userId;

    @Column(name = "code", nullable = false, unique = true)
    private String code;

    @Column(name = "program_id", nullable = false)
    private Long programId;

    @Column(name = "total_referrals")
    private int totalReferrals = 0;

    @Column(name = "successful_referrals")
    private int successfulReferrals = 0;

    @Column(name = "total_rewards_earned")
    private BigDecimal totalRewardsEarned = BigDecimal.ZERO;

    @Column(name = "active")
    private boolean active = true;

    @Column(name = "created_at")
    private LocalDateTime createdAt = LocalDateTime.now();
}

// Referral.java
@Entity
@Table(name = "referrals")
@Getter @Setter @Builder
@NoArgsConstructor @AllArgsConstructor
public class Referral {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "referral_code_id", nullable = false)
    private ReferralCode referralCode;

    @Column(name = "referee_user_id")
    private Long refereeUserId;

    @Column(name = "referee_email")
    private String refereeEmail;

    @Enumerated(EnumType.STRING)
    @Column(name = "status", nullable = false)
    private ReferralStatus status = ReferralStatus.PENDING;

    @Column(name = "ip_address")
    private String ipAddress;

    @Column(name = "device_fingerprint")
    private String deviceFingerprint;

    @Column(name = "fraud_flagged")
    private boolean fraudFlagged = false;

    @Column(name = "fraud_reason")
    private String fraudReason;

    @Column(name = "first_purchase_amount")
    private BigDecimal firstPurchaseAmount;

    @Column(name = "converted_at")
    private LocalDateTime convertedAt;

    @Column(name = "created_at")
    private LocalDateTime createdAt = LocalDateTime.now();
}

public enum ReferralStatus { PENDING, CLICKED, REGISTERED, CONVERTED, REWARDED, FRAUD }

// ReferralService.java
@Service
@RequiredArgsConstructor
public class ReferralService {

    private final ReferralCodeRepository codeRepository;
    private final ReferralRepository referralRepository;
    private final ReferralRewardRepository rewardRepository;
    private final ReferralProgramRepository programRepository;

    @Transactional
    public ReferralCode generateCode(Long userId) {
        return codeRepository.findByUserId(userId).orElseGet(() -> {
            ReferralProgram activeProgram = programRepository.findFirstByActiveTrue()
                    .orElseThrow(() -> new RuntimeException("No active referral program"));
            String code = generateUniqueCode(userId);
            ReferralCode rc = ReferralCode.builder()
                    .userId(userId)
                    .code(code)
                    .programId(activeProgram.getId())
                    .build();
            return codeRepository.save(rc);
        });
    }

    @Transactional
    public Referral trackClick(String code, String ipAddress, String deviceFingerprint,
                                String userAgent) {
        ReferralCode referralCode = codeRepository.findByCode(code)
                .orElseThrow(() -> new RuntimeException("Invalid referral code"));

        // Fraud detection: check if same IP already used this code
        boolean fraudulent = detectFraud(referralCode.getId(), ipAddress, deviceFingerprint,
                                          referralCode.getUserId());

        Referral referral = Referral.builder()
                .referralCode(referralCode)
                .status(fraudulent ? ReferralStatus.FRAUD : ReferralStatus.CLICKED)
                .ipAddress(ipAddress)
                .deviceFingerprint(deviceFingerprint)
                .fraudFlagged(fraudulent)
                .fraudReason(fraudulent ? "Duplicate IP or device fingerprint detected" : null)
                .build();

        referralCode.setTotalReferrals(referralCode.getTotalReferrals() + 1);
        codeRepository.save(referralCode);
        return referralRepository.save(referral);
    }

    @Transactional
    public void registerReferee(Long referralId, Long refereeUserId, String email) {
        Referral referral = referralRepository.findById(referralId)
                .orElseThrow(() -> new RuntimeException("Referral not found"));
        if (!referral.isFraudFlagged()) {
            referral.setRefereeUserId(refereeUserId);
            referral.setRefereeEmail(email);
            referral.setStatus(ReferralStatus.REGISTERED);
            referralRepository.save(referral);
        }
    }

    @Transactional
    public void processConversion(Long referralId, BigDecimal purchaseAmount) {
        Referral referral = referralRepository.findById(referralId)
                .orElseThrow(() -> new RuntimeException("Referral not found"));
        if (referral.isFraudFlagged() || referral.getStatus() == ReferralStatus.CONVERTED) return;

        ReferralProgram program = programRepository.findById(referral.getReferralCode().getProgramId())
                .orElseThrow();

        if (program.getMinPurchaseAmount() != null &&
            purchaseAmount.compareTo(program.getMinPurchaseAmount()) < 0) return;

        referral.setStatus(ReferralStatus.CONVERTED);
        referral.setFirstPurchaseAmount(purchaseAmount);
        referral.setConvertedAt(LocalDateTime.now());
        referralRepository.save(referral);

        // Issue rewards
        issueReward(referral, referral.getReferralCode().getUserId(),
            program.getReferrerRewardType(), program.getReferrerRewardValue());
        issueReward(referral, referral.getRefereeUserId(),
            program.getRefereeRewardType(), program.getRefereeRewardValue());

        referral.getReferralCode().setSuccessfulReferrals(
            referral.getReferralCode().getSuccessfulReferrals() + 1);
        codeRepository.save(referral.getReferralCode());
    }

    private void issueReward(Referral referral, Long userId, String rewardType, BigDecimal value) {
        ReferralReward reward = ReferralReward.builder()
                .referralId(referral.getId())
                .userId(userId)
                .rewardType(rewardType)
                .rewardValue(value)
                .status("ISSUED")
                .issuedAt(LocalDateTime.now())
                .expiresAt(LocalDateTime.now().plusMonths(6))
                .build();
        rewardRepository.save(reward);
    }

    private boolean detectFraud(Long codeId, String ip, String fingerprint, Long referrerId) {
        // Check if this IP has already been used for this code
        long ipCount = referralRepository.countByReferralCodeIdAndIpAddress(codeId, ip);
        if (ipCount > 0) return true;
        // Check if fingerprint already used
        if (fingerprint != null) {
            long fpCount = referralRepository.countByReferralCodeIdAndDeviceFingerprint(codeId, fingerprint);
            if (fpCount > 0) return true;
        }
        return false;
    }

    private String generateUniqueCode(Long userId) {
        String base = Long.toString(userId, 36).toUpperCase();
        String suffix = Long.toString(System.currentTimeMillis(), 36).toUpperCase().substring(4);
        return ("REF-" + base + suffix).substring(0, Math.min(15, base.length() + suffix.length() + 4));
    }
}
```

### Controller

```java
// ReferralController.java
@RestController
@RequestMapping("/api/v1/referrals")
@RequiredArgsConstructor
public class ReferralController {

    private final ReferralService referralService;

    @PostMapping("/codes/generate")
    public ResponseEntity<ReferralCode> generateCode(@RequestParam Long userId) {
        return ResponseEntity.ok(referralService.generateCode(userId));
    }

    @PostMapping("/track")
    public ResponseEntity<Referral> trackClick(@RequestBody TrackClickRequest request,
            HttpServletRequest httpRequest) {
        String ip = httpRequest.getRemoteAddr();
        return ResponseEntity.ok(referralService.trackClick(
            request.code(), ip, request.deviceFingerprint(), request.userAgent()));
    }

    @PostMapping("/{referralId}/register")
    public ResponseEntity<Void> register(@PathVariable Long referralId,
            @RequestBody RegisterRequest request) {
        referralService.registerReferee(referralId, request.userId(), request.email());
        return ResponseEntity.ok().build();
    }

    @PostMapping("/{referralId}/convert")
    public ResponseEntity<Void> convert(@PathVariable Long referralId,
            @RequestBody ConvertRequest request) {
        referralService.processConversion(referralId, request.purchaseAmount());
        return ResponseEntity.ok().build();
    }

    @GetMapping("/codes/{userId}")
    public ResponseEntity<ReferralCode> getCode(@PathVariable Long userId) {
        return ResponseEntity.ok(referralService.generateCode(userId));
    }

    record TrackClickRequest(String code, String deviceFingerprint, String userAgent) {}
    record RegisterRequest(Long userId, String email) {}
    record ConvertRequest(java.math.BigDecimal purchaseAmount) {}
}
```

### docker-compose.yml

```yaml
# docker-compose.yml (Referral Program)
version: '3.8'
services:
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: referral_db
      POSTGRES_USER: referral_user
      POSTGRES_PASSWORD: referral_pass
    ports:
      - "5432:5432"
    volumes:
      - referral_pg_data:/var/lib/postgresql/data

  referral-service:
    build: .
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/referral_db
      SPRING_DATASOURCE_USERNAME: referral_user
      SPRING_DATASOURCE_PASSWORD: referral_pass
    ports:
      - "8080:8080"
    depends_on:
      - postgres

volumes:
  referral_pg_data:
```

---

## โปรเจค 54: Affiliate Marketing Platform

### ภาพรวมระบบ

ระบบ Affiliate Marketing ช่วยให้ธุรกิจขยายการตลาดผ่านเครือข่าย Publisher ที่หลากหลาย Affiliates จะได้รับ Link พิเศษสำหรับโปรโมตสินค้า ระบบติดตาม Click และ Conversion อัตโนมัติ คำนวณค่า Commission และจัดการ Payout ให้กับ Affiliates ทั้งหมด

- **Affiliate Management**: จัดการโปรไฟล์และข้อมูล Affiliate
- **Tracking Links**: สร้างและจัดการ Link ติดตาม
- **Click & Conversion**: ติดตาม Click และ Conversion แบบ Real-time
- **Commission Calculation**: คำนวณค่า Commission ตามประเภท
- **Payout Management**: จัดการการจ่ายเงิน Affiliate
- **Performance Reports**: รายงานผลประสิทธิภาพ

### Flyway Migration

```sql
-- V1__create_affiliate_tables.sql
CREATE TABLE affiliates (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL UNIQUE,
    affiliate_code VARCHAR(20) NOT NULL UNIQUE,
    company_name VARCHAR(200),
    website_url VARCHAR(500),
    commission_rate DECIMAL(5,2) NOT NULL DEFAULT 10.00,
    commission_type VARCHAR(20) NOT NULL DEFAULT 'PERCENTAGE',
    status VARCHAR(20) NOT NULL DEFAULT 'PENDING',
    payment_method VARCHAR(20),
    payment_details JSONB,
    total_clicks BIGINT NOT NULL DEFAULT 0,
    total_conversions BIGINT NOT NULL DEFAULT 0,
    total_commission_earned DECIMAL(14,2) NOT NULL DEFAULT 0,
    pending_payout DECIMAL(14,2) NOT NULL DEFAULT 0,
    approved_at TIMESTAMP,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE affiliate_links (
    id BIGSERIAL PRIMARY KEY,
    affiliate_id BIGINT NOT NULL REFERENCES affiliates(id),
    name VARCHAR(200) NOT NULL,
    destination_url TEXT NOT NULL,
    tracking_code VARCHAR(50) NOT NULL UNIQUE,
    campaign VARCHAR(100),
    total_clicks BIGINT NOT NULL DEFAULT 0,
    unique_clicks BIGINT NOT NULL DEFAULT 0,
    total_conversions BIGINT NOT NULL DEFAULT 0,
    active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE affiliate_clicks (
    id BIGSERIAL PRIMARY KEY,
    link_id BIGINT NOT NULL REFERENCES affiliate_links(id),
    affiliate_id BIGINT NOT NULL REFERENCES affiliates(id),
    ip_address VARCHAR(45),
    user_agent TEXT,
    referer VARCHAR(1000),
    click_token VARCHAR(100) NOT NULL UNIQUE,
    is_unique BOOLEAN NOT NULL DEFAULT TRUE,
    clicked_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE affiliate_conversions (
    id BIGSERIAL PRIMARY KEY,
    click_id BIGINT REFERENCES affiliate_clicks(id),
    affiliate_id BIGINT NOT NULL REFERENCES affiliates(id),
    link_id BIGINT NOT NULL REFERENCES affiliate_links(id),
    order_id VARCHAR(100) NOT NULL,
    order_amount DECIMAL(12,2) NOT NULL,
    commission_amount DECIMAL(10,2) NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'PENDING',
    converted_at TIMESTAMP NOT NULL DEFAULT NOW(),
    approved_at TIMESTAMP,
    paid_at TIMESTAMP
);

CREATE TABLE affiliate_payouts (
    id BIGSERIAL PRIMARY KEY,
    affiliate_id BIGINT NOT NULL REFERENCES affiliates(id),
    amount DECIMAL(12,2) NOT NULL,
    payment_method VARCHAR(20) NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'PROCESSING',
    reference VARCHAR(100),
    period_from DATE NOT NULL,
    period_to DATE NOT NULL,
    conversions_count INT NOT NULL,
    processed_at TIMESTAMP,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_affiliate_clicks_token ON affiliate_clicks(click_token);
CREATE INDEX idx_affiliate_conversions_order ON affiliate_conversions(order_id);
```

### Entity & Service

```java
// Affiliate.java
package com.affiliate.entity;

import jakarta.persistence.*;
import lombok.*;
import org.hibernate.annotations.JdbcTypeCode;
import org.hibernate.type.SqlTypes;
import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.Map;

@Entity
@Table(name = "affiliates")
@Getter @Setter @Builder
@NoArgsConstructor @AllArgsConstructor
public class Affiliate {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "user_id", nullable = false, unique = true)
    private Long userId;

    @Column(name = "affiliate_code", nullable = false, unique = true)
    private String affiliateCode;

    @Column(name = "company_name")
    private String companyName;

    @Column(name = "commission_rate", nullable = false)
    private BigDecimal commissionRate = BigDecimal.valueOf(10);

    @Column(name = "commission_type", nullable = false)
    private String commissionType = "PERCENTAGE";

    @Enumerated(EnumType.STRING)
    @Column(name = "status", nullable = false)
    private AffiliateStatus status = AffiliateStatus.PENDING;

    @Column(name = "total_clicks")
    private Long totalClicks = 0L;

    @Column(name = "total_conversions")
    private Long totalConversions = 0L;

    @Column(name = "total_commission_earned")
    private BigDecimal totalCommissionEarned = BigDecimal.ZERO;

    @Column(name = "pending_payout")
    private BigDecimal pendingPayout = BigDecimal.ZERO;

    @JdbcTypeCode(SqlTypes.JSON)
    @Column(name = "payment_details")
    private Map<String, Object> paymentDetails;

    @Column(name = "created_at")
    private LocalDateTime createdAt = LocalDateTime.now();
}

public enum AffiliateStatus { PENDING, ACTIVE, SUSPENDED, REJECTED }

// AffiliateService.java
@Service
@RequiredArgsConstructor
public class AffiliateService {

    private final AffiliateRepository affiliateRepository;
    private final AffiliateLinkRepository linkRepository;
    private final AffiliateClickRepository clickRepository;
    private final AffiliateConversionRepository conversionRepository;
    private final AffiliatePayoutRepository payoutRepository;

    @Transactional
    public Affiliate register(Long userId, String companyName, String websiteUrl,
                               BigDecimal commissionRate) {
        String code = "AFF-" + Long.toString(userId, 36).toUpperCase()
                    + Long.toString(System.currentTimeMillis(), 36).toUpperCase().substring(5);
        Affiliate affiliate = Affiliate.builder()
                .userId(userId)
                .affiliateCode(code.substring(0, Math.min(20, code.length())))
                .companyName(companyName)
                .commissionRate(commissionRate != null ? commissionRate : BigDecimal.valueOf(10))
                .status(AffiliateStatus.PENDING)
                .build();
        return affiliateRepository.save(affiliate);
    }

    @Transactional
    public AffiliateLink createLink(Long affiliateId, String name, String destinationUrl,
                                     String campaign) {
        Affiliate affiliate = affiliateRepository.findById(affiliateId)
                .orElseThrow(() -> new RuntimeException("Affiliate not found"));
        if (affiliate.getStatus() != AffiliateStatus.ACTIVE) {
            throw new IllegalStateException("Affiliate is not active");
        }
        String trackingCode = "TRK-" + affiliateId + "-"
                + Long.toString(System.currentTimeMillis(), 36).toUpperCase();
        AffiliateLink link = AffiliateLink.builder()
                .affiliate(affiliate)
                .name(name)
                .destinationUrl(destinationUrl)
                .trackingCode(trackingCode.substring(0, Math.min(50, trackingCode.length())))
                .campaign(campaign)
                .build();
        return linkRepository.save(link);
    }

    @Transactional
    public String trackClick(String trackingCode, String ipAddress, String userAgent, String referer) {
        AffiliateLink link = linkRepository.findByTrackingCode(trackingCode)
                .orElseThrow(() -> new RuntimeException("Invalid tracking link"));

        String clickToken = java.util.UUID.randomUUID().toString();
        boolean isUnique = !clickRepository.existsByLinkIdAndIpAddress(link.getId(), ipAddress);

        AffiliateClick click = AffiliateClick.builder()
                .link(link)
                .affiliateId(link.getAffiliate().getId())
                .ipAddress(ipAddress)
                .userAgent(userAgent)
                .referer(referer)
                .clickToken(clickToken)
                .isUnique(isUnique)
                .build();
        clickRepository.save(click);

        link.setTotalClicks(link.getTotalClicks() + 1);
        if (isUnique) link.setUniqueClicks(link.getUniqueClicks() + 1);
        linkRepository.save(link);

        link.getAffiliate().setTotalClicks(link.getAffiliate().getTotalClicks() + 1);
        affiliateRepository.save(link.getAffiliate());

        return link.getDestinationUrl() + "?click_token=" + clickToken;
    }

    @Transactional
    public AffiliateConversion recordConversion(String clickToken, String orderId,
                                                  BigDecimal orderAmount) {
        AffiliateClick click = clickRepository.findByClickToken(clickToken)
                .orElseThrow(() -> new RuntimeException("Click token not found"));

        Affiliate affiliate = affiliateRepository.findById(click.getAffiliateId())
                .orElseThrow();

        BigDecimal commission;
        if ("PERCENTAGE".equals(affiliate.getCommissionType())) {
            commission = orderAmount.multiply(affiliate.getCommissionRate())
                    .divide(BigDecimal.valueOf(100), 2, java.math.RoundingMode.HALF_UP);
        } else {
            commission = affiliate.getCommissionRate(); // Fixed amount
        }

        AffiliateConversion conversion = AffiliateConversion.builder()
                .click(click)
                .affiliateId(affiliate.getId())
                .linkId(click.getLink().getId())
                .orderId(orderId)
                .orderAmount(orderAmount)
                .commissionAmount(commission)
                .status("PENDING")
                .build();
        conversionRepository.save(conversion);

        affiliate.setTotalConversions(affiliate.getTotalConversions() + 1);
        affiliate.setPendingPayout(affiliate.getPendingPayout().add(commission));
        affiliateRepository.save(affiliate);

        return conversion;
    }

    @Transactional
    public AffiliatePayout processPayout(Long affiliateId,
                                          java.time.LocalDate periodFrom,
                                          java.time.LocalDate periodTo) {
        Affiliate affiliate = affiliateRepository.findById(affiliateId)
                .orElseThrow();
        BigDecimal pendingAmount = affiliate.getPendingPayout();
        if (pendingAmount.compareTo(BigDecimal.ZERO) <= 0) {
            throw new IllegalStateException("No pending payout");
        }

        AffiliatePayout payout = AffiliatePayout.builder()
                .affiliateId(affiliateId)
                .amount(pendingAmount)
                .paymentMethod(affiliate.getPaymentMethod() != null ?
                    affiliate.getPaymentMethod() : "BANK_TRANSFER")
                .status("PROCESSING")
                .periodFrom(periodFrom)
                .periodTo(periodTo)
                .conversionsCount(affiliate.getTotalConversions().intValue())
                .build();
        payoutRepository.save(payout);

        affiliate.setTotalCommissionEarned(
            affiliate.getTotalCommissionEarned().add(pendingAmount));
        affiliate.setPendingPayout(BigDecimal.ZERO);
        affiliateRepository.save(affiliate);

        return payout;
    }

    public AffiliatePerformanceReport getPerformanceReport(Long affiliateId) {
        Affiliate affiliate = affiliateRepository.findById(affiliateId)
                .orElseThrow();
        List<AffiliateLink> links = linkRepository.findByAffiliateId(affiliateId);
        return new AffiliatePerformanceReport(
            affiliate.getAffiliateCode(),
            affiliate.getTotalClicks(),
            affiliate.getTotalConversions(),
            affiliate.getTotalCommissionEarned(),
            affiliate.getPendingPayout(),
            links.size(),
            affiliate.getTotalClicks() > 0 ?
                (double) affiliate.getTotalConversions() / affiliate.getTotalClicks() * 100 : 0.0
        );
    }

    public record AffiliatePerformanceReport(String code, long clicks, long conversions,
        BigDecimal totalEarned, BigDecimal pendingPayout, int activeLinks, double conversionRate) {}
}
```

### Controller

```java
// AffiliateController.java
@RestController
@RequestMapping("/api/v1/affiliates")
@RequiredArgsConstructor
public class AffiliateController {

    private final AffiliateService affiliateService;

    @PostMapping("/register")
    public ResponseEntity<Affiliate> register(@RequestBody RegisterAffiliateRequest request) {
        return ResponseEntity.ok(affiliateService.register(
            request.userId(), request.companyName(), request.websiteUrl(), request.commissionRate()));
    }

    @PostMapping("/{affiliateId}/links")
    public ResponseEntity<AffiliateLink> createLink(@PathVariable Long affiliateId,
            @RequestBody CreateLinkRequest request) {
        return ResponseEntity.ok(affiliateService.createLink(
            affiliateId, request.name(), request.destinationUrl(), request.campaign()));
    }

    @GetMapping("/track/{trackingCode}")
    public ResponseEntity<Void> trackAndRedirect(@PathVariable String trackingCode,
            HttpServletRequest request, HttpServletResponse response) throws Exception {
        String redirectUrl = affiliateService.trackClick(
            trackingCode, request.getRemoteAddr(),
            request.getHeader("User-Agent"), request.getHeader("Referer"));
        response.sendRedirect(redirectUrl);
        return ResponseEntity.status(302).build();
    }

    @PostMapping("/conversions")
    public ResponseEntity<AffiliateConversion> recordConversion(
            @RequestBody ConversionRequest request) {
        return ResponseEntity.ok(affiliateService.recordConversion(
            request.clickToken(), request.orderId(), request.orderAmount()));
    }

    @GetMapping("/{affiliateId}/performance")
    public ResponseEntity<AffiliateService.AffiliatePerformanceReport> getPerformance(
            @PathVariable Long affiliateId) {
        return ResponseEntity.ok(affiliateService.getPerformanceReport(affiliateId));
    }

    @PostMapping("/{affiliateId}/payout")
    public ResponseEntity<AffiliatePayout> processPayout(@PathVariable Long affiliateId,
            @RequestBody PayoutRequest request) {
        return ResponseEntity.ok(affiliateService.processPayout(
            affiliateId, request.periodFrom(), request.periodTo()));
    }

    record RegisterAffiliateRequest(Long userId, String companyName, String websiteUrl,
                                     java.math.BigDecimal commissionRate) {}
    record CreateLinkRequest(String name, String destinationUrl, String campaign) {}
    record ConversionRequest(String clickToken, String orderId, java.math.BigDecimal orderAmount) {}
    record PayoutRequest(java.time.LocalDate periodFrom, java.time.LocalDate periodTo) {}
}
```

### docker-compose.yml

```yaml
# docker-compose.yml (Affiliate Marketing)
version: '3.8'
services:
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: affiliate_db
      POSTGRES_USER: affiliate_user
      POSTGRES_PASSWORD: affiliate_pass
    ports:
      - "5432:5432"
    volumes:
      - affiliate_pg_data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

  affiliate-service:
    build: .
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/affiliate_db
      SPRING_DATASOURCE_USERNAME: affiliate_user
      SPRING_DATASOURCE_PASSWORD: affiliate_pass
      SPRING_REDIS_HOST: redis
    ports:
      - "8080:8080"
    depends_on:
      - postgres
      - redis

volumes:
  affiliate_pg_data:
```

---

## โปรเจค 55: Price Comparison Service

### ภาพรวมระบบ

Price Comparison Service ช่วยให้ผู้บริโภคเปรียบเทียบราคาสินค้าจากหลายแหล่งได้ในที่เดียว ระบบดึงข้อมูลราคาจากร้านค้าต่าง ๆ เก็บประวัติราคา แจ้งเตือนเมื่อราคาลดลง และให้ API สำหรับ Browser Extension ที่ช่วยให้ผู้ใช้เห็นราคาเปรียบเทียบขณะช็อปปิ้งออนไลน์

- **Products & Sources**: สินค้าและแหล่งราคาจากหลายร้าน
- **Price History**: ประวัติราคาสำหรับวิเคราะห์แนวโน้ม
- **Price Drop Alerts**: แจ้งเตือนเมื่อราคาลดลงตามเป้า
- **Best Deal Finder**: ค้นหาราคาดีที่สุดอัตโนมัติ
- **Browser Extension API**: API สำหรับ Extension

### Flyway Migration

```sql
-- V1__create_price_comparison_tables.sql
CREATE TABLE products (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(500) NOT NULL,
    description TEXT,
    brand VARCHAR(200),
    category VARCHAR(100),
    barcode VARCHAR(100),
    image_url VARCHAR(1000),
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE price_sources (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    website_url VARCHAR(500) NOT NULL,
    logo_url VARCHAR(500),
    active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE product_prices (
    id BIGSERIAL PRIMARY KEY,
    product_id BIGINT NOT NULL REFERENCES products(id),
    source_id BIGINT NOT NULL REFERENCES price_sources(id),
    price DECIMAL(12,2) NOT NULL,
    original_price DECIMAL(12,2),
    discount_percentage DECIMAL(5,2),
    currency VARCHAR(3) NOT NULL DEFAULT 'THB',
    product_url TEXT,
    in_stock BOOLEAN NOT NULL DEFAULT TRUE,
    last_updated TIMESTAMP NOT NULL DEFAULT NOW(),
    UNIQUE (product_id, source_id)
);

CREATE TABLE price_history (
    id BIGSERIAL PRIMARY KEY,
    product_id BIGINT NOT NULL REFERENCES products(id),
    source_id BIGINT NOT NULL REFERENCES price_sources(id),
    price DECIMAL(12,2) NOT NULL,
    currency VARCHAR(3) NOT NULL DEFAULT 'THB',
    recorded_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE price_alerts (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL,
    product_id BIGINT NOT NULL REFERENCES products(id),
    target_price DECIMAL(12,2) NOT NULL,
    currency VARCHAR(3) NOT NULL DEFAULT 'THB',
    notification_email VARCHAR(255),
    notification_type VARCHAR(20) NOT NULL DEFAULT 'EMAIL',
    active BOOLEAN NOT NULL DEFAULT TRUE,
    triggered_at TIMESTAMP,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_product_prices_product ON product_prices(product_id);
CREATE INDEX idx_price_history_product_date ON price_history(product_id, recorded_at DESC);
CREATE INDEX idx_price_alerts_product ON price_alerts(product_id, active);
```

### Entity & Service

```java
// Product.java
package com.pricecompare.entity;

import jakarta.persistence.*;
import lombok.*;
import java.time.LocalDateTime;

@Entity
@Table(name = "products")
@Getter @Setter @Builder
@NoArgsConstructor @AllArgsConstructor
public class Product {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "name", nullable = false)
    private String name;

    @Column(name = "description")
    private String description;

    @Column(name = "brand")
    private String brand;

    @Column(name = "category")
    private String category;

    @Column(name = "barcode")
    private String barcode;

    @Column(name = "image_url")
    private String imageUrl;

    @Column(name = "created_at")
    private LocalDateTime createdAt = LocalDateTime.now();
}

// ProductPrice.java
@Entity
@Table(name = "product_prices")
@Getter @Setter @Builder
@NoArgsConstructor @AllArgsConstructor
public class ProductPrice {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "product_id", nullable = false)
    private Long productId;

    @Column(name = "source_id", nullable = false)
    private Long sourceId;

    @Column(name = "price", nullable = false)
    private java.math.BigDecimal price;

    @Column(name = "original_price")
    private java.math.BigDecimal originalPrice;

    @Column(name = "discount_percentage")
    private java.math.BigDecimal discountPercentage;

    @Column(name = "product_url")
    private String productUrl;

    @Column(name = "in_stock")
    private boolean inStock = true;

    @Column(name = "last_updated")
    private LocalDateTime lastUpdated = LocalDateTime.now();
}

// PriceAlert.java
@Entity
@Table(name = "price_alerts")
@Getter @Setter @Builder
@NoArgsConstructor @AllArgsConstructor
public class PriceAlert {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "user_id", nullable = false)
    private Long userId;

    @Column(name = "product_id", nullable = false)
    private Long productId;

    @Column(name = "target_price", nullable = false)
    private java.math.BigDecimal targetPrice;

    @Column(name = "notification_email")
    private String notificationEmail;

    @Column(name = "active")
    private boolean active = true;

    @Column(name = "triggered_at")
    private LocalDateTime triggeredAt;

    @Column(name = "created_at")
    private LocalDateTime createdAt = LocalDateTime.now();
}

// PriceComparisonService.java
@Service
@RequiredArgsConstructor
public class PriceComparisonService {

    private final ProductRepository productRepository;
    private final ProductPriceRepository priceRepository;
    private final PriceHistoryRepository historyRepository;
    private final PriceAlertRepository alertRepository;
    private final PriceSourceRepository sourceRepository;
    private final EmailService emailService;

    @Transactional
    public void updatePrice(Long productId, Long sourceId, java.math.BigDecimal newPrice,
                             String productUrl, boolean inStock) {
        ProductPrice existing = priceRepository.findByProductIdAndSourceId(productId, sourceId)
                .orElse(null);

        java.math.BigDecimal oldPrice = existing != null ? existing.getPrice() : null;

        if (existing == null) {
            existing = ProductPrice.builder()
                    .productId(productId).sourceId(sourceId)
                    .price(newPrice).productUrl(productUrl).inStock(inStock).build();
        } else {
            existing.setPrice(newPrice);
            existing.setProductUrl(productUrl);
            existing.setInStock(inStock);
            existing.setLastUpdated(java.time.LocalDateTime.now());
        }
        priceRepository.save(existing);

        // Record history
        PriceHistory history = PriceHistory.builder()
                .productId(productId).sourceId(sourceId).price(newPrice).build();
        historyRepository.save(history);

        // Check alerts if price dropped
        if (oldPrice != null && newPrice.compareTo(oldPrice) < 0) {
            checkAndTriggerAlerts(productId, newPrice);
        }
    }

    public PriceComparisonResult compareByProduct(Long productId) {
        Product product = productRepository.findById(productId)
                .orElseThrow(() -> new RuntimeException("Product not found"));
        List<ProductPrice> prices = priceRepository.findByProductIdOrderByPrice(productId);

        ProductPrice best = prices.stream()
                .filter(ProductPrice::isInStock)
                .min(java.util.Comparator.comparing(ProductPrice::getPrice))
                .orElse(null);

        return new PriceComparisonResult(product, prices, best);
    }

    public PriceComparisonResult findByBarcode(String barcode) {
        Product product = productRepository.findByBarcode(barcode)
                .orElseThrow(() -> new RuntimeException("Product not found for barcode: " + barcode));
        return compareByProduct(product.getId());
    }

    public List<PriceHistory> getPriceHistory(Long productId, Long sourceId, int days) {
        java.time.LocalDateTime since = java.time.LocalDateTime.now().minusDays(days);
        return historyRepository.findByProductIdAndSourceIdAndRecordedAtAfterOrderByRecordedAt(
            productId, sourceId, since);
    }

    @Transactional
    public PriceAlert createAlert(Long userId, Long productId, java.math.BigDecimal targetPrice,
                                   String email) {
        PriceAlert alert = PriceAlert.builder()
                .userId(userId).productId(productId)
                .targetPrice(targetPrice).notificationEmail(email).build();
        return alertRepository.save(alert);
    }

    private void checkAndTriggerAlerts(Long productId, java.math.BigDecimal currentPrice) {
        List<PriceAlert> alerts = alertRepository
                .findByProductIdAndActiveAndTargetPriceGreaterThanEqual(
                    productId, true, currentPrice);
        for (PriceAlert alert : alerts) {
            alert.setActive(false);
            alert.setTriggeredAt(java.time.LocalDateTime.now());
            alertRepository.save(alert);
            emailService.sendPriceAlert(alert.getNotificationEmail(), productId, currentPrice);
        }
    }

    public List<Product> searchProducts(String query) {
        return productRepository.findByNameContainingIgnoreCaseOrBrandContainingIgnoreCase(
            query, query);
    }

    public record PriceComparisonResult(Product product, List<ProductPrice> prices,
                                         ProductPrice bestDeal) {}
}
```

### Controller

```java
// PriceComparisonController.java
@RestController
@RequestMapping("/api/v1/prices")
@RequiredArgsConstructor
public class PriceComparisonController {

    private final PriceComparisonService service;

    @GetMapping("/compare/{productId}")
    public ResponseEntity<PriceComparisonService.PriceComparisonResult> compare(
            @PathVariable Long productId) {
        return ResponseEntity.ok(service.compareByProduct(productId));
    }

    @GetMapping("/barcode/{barcode}")
    public ResponseEntity<PriceComparisonService.PriceComparisonResult> findByBarcode(
            @PathVariable String barcode) {
        return ResponseEntity.ok(service.findByBarcode(barcode));
    }

    @GetMapping("/history/{productId}/{sourceId}")
    public ResponseEntity<List<PriceHistory>> getHistory(
            @PathVariable Long productId, @PathVariable Long sourceId,
            @RequestParam(defaultValue = "30") int days) {
        return ResponseEntity.ok(service.getPriceHistory(productId, sourceId, days));
    }

    @PostMapping("/alerts")
    public ResponseEntity<PriceAlert> createAlert(@RequestBody CreateAlertRequest request) {
        return ResponseEntity.ok(service.createAlert(
            request.userId(), request.productId(), request.targetPrice(), request.email()));
    }

    @GetMapping("/search")
    public ResponseEntity<List<Product>> search(@RequestParam String q) {
        return ResponseEntity.ok(service.searchProducts(q));
    }

    // Browser Extension API endpoint
    @GetMapping("/extension/check")
    public ResponseEntity<BrowserExtensionResponse> extensionCheck(
            @RequestParam String url, @RequestParam(required = false) String barcode) {
        try {
            PriceComparisonService.PriceComparisonResult result = barcode != null ?
                service.findByBarcode(barcode) : null;
            if (result == null) return ResponseEntity.noContent().build();
            return ResponseEntity.ok(new BrowserExtensionResponse(
                result.product().getName(), result.prices().size(),
                result.bestDeal() != null ? result.bestDeal().getPrice() : null,
                result.product().getId()));
        } catch (Exception e) {
            return ResponseEntity.noContent().build();
        }
    }

    record CreateAlertRequest(Long userId, Long productId,
                               java.math.BigDecimal targetPrice, String email) {}
    record BrowserExtensionResponse(String productName, int sourceCount,
                                     java.math.BigDecimal bestPrice, Long productId) {}
}
```

### docker-compose.yml

```yaml
# docker-compose.yml (Price Comparison Service)
version: '3.8'
services:
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: pricecompare_db
      POSTGRES_USER: price_user
      POSTGRES_PASSWORD: price_pass
    ports:
      - "5432:5432"
    volumes:
      - price_pg_data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    command: redis-server --maxmemory 256mb --maxmemory-policy allkeys-lru

  elasticsearch:
    image: elasticsearch:8.11.0
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    ports:
      - "9200:9200"
    volumes:
      - price_es_data:/usr/share/elasticsearch/data

  price-service:
    build: .
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/pricecompare_db
      SPRING_DATASOURCE_USERNAME: price_user
      SPRING_DATASOURCE_PASSWORD: price_pass
      SPRING_REDIS_HOST: redis
      SPRING_ELASTICSEARCH_URIS: http://elasticsearch:9200
    ports:
      - "8080:8080"
    depends_on:
      - postgres
      - redis
      - elasticsearch

volumes:
  price_pg_data:
  price_es_data:
```

---

*[← Part 110](./part-110-ecommerce-advanced.md) | [Part 112: Food & Health →](./part-112-food-health.md)*

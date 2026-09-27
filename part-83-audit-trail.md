# Part 83: Audit Trail
## ขั้นตอนที่ 2921-2960

**ระดับ:** ระดับโลก (World-Class)
**เวลาเรียน:** 7-9 ชั่วโมง
**เป้าหมาย:** เรียนรู้การสร้างระบบ Audit Trail ที่ครบวงจร ตั้งแต่การใช้ Spring Data JPA Auditing, Hibernate Envers, ไปจนถึงการ publish audit events ไปยัง Kafka และการทำ GDPR-compliant audit

---

## 2921-2927: Spring Data JPA Auditing พื้นฐาน

Audit Trail คือการบันทึกว่าใคร ทำอะไร เมื่อไหร่ กับข้อมูลใด เป็นสิ่งที่ขาดไม่ได้ในระบบ enterprise

### เปิดใช้งาน JPA Auditing

```java
// AuditingConfig.java
package com.example.audit.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.domain.AuditorAware;
import org.springframework.data.jpa.repository.config.EnableJpaAuditing;

@Configuration
@EnableJpaAuditing(auditorAwareRef = "auditorProvider")
public class AuditingConfig {

    @Bean
    public AuditorAware<String> auditorProvider() {
        return new SecurityAuditorAware();
    }
}
```

### AuditorAware Implementation

```java
// SecurityAuditorAware.java
package com.example.audit.config;

import org.springframework.data.domain.AuditorAware;
import org.springframework.security.core.Authentication;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.security.core.userdetails.UserDetails;

import java.util.Optional;

public class SecurityAuditorAware implements AuditorAware<String> {

    @Override
    public Optional<String> getCurrentAuditor() {
        Authentication authentication = SecurityContextHolder.getContext().getAuthentication();
        
        if (authentication == null || !authentication.isAuthenticated()) {
            return Optional.of("SYSTEM");
        }
        
        Object principal = authentication.getPrincipal();
        
        if (principal instanceof UserDetails userDetails) {
            return Optional.of(userDetails.getUsername());
        }
        
        if (principal instanceof String username) {
            return Optional.of(username);
        }
        
        return Optional.of("UNKNOWN");
    }
}
```

### Auditable Base Entity

```java
// AuditableEntity.java
package com.example.audit.entity;

import jakarta.persistence.*;
import lombok.Getter;
import lombok.Setter;
import org.springframework.data.annotation.*;
import org.springframework.data.jpa.domain.support.AuditingEntityListener;

import java.time.LocalDateTime;

@Getter
@Setter
@MappedSuperclass
@EntityListeners(AuditingEntityListener.class)
public abstract class AuditableEntity {

    @CreatedDate
    @Column(name = "created_at", nullable = false, updatable = false)
    private LocalDateTime createdAt;

    @CreatedBy
    @Column(name = "created_by", nullable = false, updatable = false, length = 100)
    private String createdBy;

    @LastModifiedDate
    @Column(name = "updated_at")
    private LocalDateTime updatedAt;

    @LastModifiedBy
    @Column(name = "updated_by", length = 100)
    private String updatedBy;

    @Version
    @Column(name = "version")
    private Long version;
}
```

### Entity ที่ extend AuditableEntity

```java
// Product.java
package com.example.audit.entity;

import jakarta.persistence.*;
import lombok.*;

import java.math.BigDecimal;

@Entity
@Table(name = "products")
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class Product extends AuditableEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String name;

    @Column(length = 1000)
    private String description;

    @Column(nullable = false, precision = 10, scale = 2)
    private BigDecimal price;

    @Column(nullable = false)
    private Integer stock;

    @Enumerated(EnumType.STRING)
    private ProductStatus status;
}
```

---

## 2928-2933: Custom Audit Log Table

การสร้าง Audit Log table แยกต่างหากเพื่อบันทึกประวัติการเปลี่ยนแปลง

### Audit Log Entity

```java
// AuditLog.java
package com.example.audit.entity;

import jakarta.persistence.*;
import lombok.*;

import java.time.LocalDateTime;
import java.util.Map;

@Entity
@Table(name = "audit_logs", indexes = {
    @Index(name = "idx_audit_entity_type", columnList = "entity_type"),
    @Index(name = "idx_audit_entity_id", columnList = "entity_id"),
    @Index(name = "idx_audit_performed_by", columnList = "performed_by"),
    @Index(name = "idx_audit_performed_at", columnList = "performed_at"),
    @Index(name = "idx_audit_action", columnList = "action")
})
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class AuditLog {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "entity_type", nullable = false, length = 100)
    private String entityType;

    @Column(name = "entity_id", nullable = false, length = 100)
    private String entityId;

    @Enumerated(EnumType.STRING)
    @Column(name = "action", nullable = false, length = 20)
    private AuditAction action;

    @Column(name = "performed_by", nullable = false, length = 100)
    private String performedBy;

    @Column(name = "performed_at", nullable = false)
    private LocalDateTime performedAt;

    @Column(name = "ip_address", length = 45)
    private String ipAddress;

    @Column(name = "user_agent", length = 500)
    private String userAgent;

    @Column(name = "old_values", columnDefinition = "jsonb")
    @Convert(converter = JsonConverter.class)
    private Map<String, Object> oldValues;

    @Column(name = "new_values", columnDefinition = "jsonb")
    @Convert(converter = JsonConverter.class)
    private Map<String, Object> newValues;

    @Column(name = "changed_fields", length = 1000)
    private String changedFields;

    @Column(name = "session_id", length = 100)
    private String sessionId;

    @Column(name = "request_id", length = 100)
    private String requestId;

    @Column(name = "correlation_id", length = 100)
    private String correlationId;

    public enum AuditAction {
        CREATE, READ, UPDATE, DELETE, LOGIN, LOGOUT, EXPORT, IMPORT
    }
}
```

### JSON Converter สำหรับ Map

```java
// JsonConverter.java
package com.example.audit.converter;

import com.fasterxml.jackson.core.type.TypeReference;
import com.fasterxml.jackson.databind.ObjectMapper;
import jakarta.persistence.AttributeConverter;
import jakarta.persistence.Converter;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;

import java.util.Map;

@Slf4j
@Converter
@RequiredArgsConstructor
public class JsonConverter implements AttributeConverter<Map<String, Object>, String> {

    private final ObjectMapper objectMapper;

    @Override
    public String convertToDatabaseColumn(Map<String, Object> attribute) {
        if (attribute == null) return null;
        try {
            return objectMapper.writeValueAsString(attribute);
        } catch (Exception e) {
            log.error("Error converting map to JSON", e);
            return null;
        }
    }

    @Override
    public Map<String, Object> convertToEntityAttribute(String dbData) {
        if (dbData == null || dbData.isBlank()) return null;
        try {
            return objectMapper.readValue(dbData, new TypeReference<Map<String, Object>>() {});
        } catch (Exception e) {
            log.error("Error converting JSON to map", e);
            return null;
        }
    }
}
```

### Audit Log Service

```java
// AuditLogService.java
package com.example.audit.service;

import com.example.audit.entity.AuditLog;
import com.example.audit.entity.AuditLog.AuditAction;
import com.example.audit.repository.AuditLogRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.scheduling.annotation.Async;
import org.springframework.stereotype.Service;
import org.springframework.web.context.request.RequestContextHolder;
import org.springframework.web.context.request.ServletRequestAttributes;

import java.time.LocalDateTime;
import java.util.Map;

@Slf4j
@Service
@RequiredArgsConstructor
public class AuditLogService {

    private final AuditLogRepository auditLogRepository;
    private final ObjectMapper objectMapper;
    private final AuditorAware<String> auditorAware;

    @Async
    public void log(
        String entityType,
        String entityId,
        AuditAction action,
        Object oldEntity,
        Object newEntity
    ) {
        try {
            Map<String, Object> oldValues = toMap(oldEntity);
            Map<String, Object> newValues = toMap(newEntity);
            String changedFields = findChangedFields(oldValues, newValues);

            AuditLog auditLog = AuditLog.builder()
                .entityType(entityType)
                .entityId(entityId)
                .action(action)
                .performedBy(auditorAware.getCurrentAuditor().orElse("UNKNOWN"))
                .performedAt(LocalDateTime.now())
                .ipAddress(getClientIp())
                .userAgent(getUserAgent())
                .oldValues(oldValues)
                .newValues(newValues)
                .changedFields(changedFields)
                .sessionId(getSessionId())
                .requestId(getRequestId())
                .build();

            auditLogRepository.save(auditLog);
        } catch (Exception e) {
            log.error("Failed to save audit log for {}/{}", entityType, entityId, e);
        }
    }

    @SuppressWarnings("unchecked")
    private Map<String, Object> toMap(Object obj) {
        if (obj == null) return null;
        return objectMapper.convertValue(obj, Map.class);
    }

    private String findChangedFields(
        Map<String, Object> oldValues,
        Map<String, Object> newValues
    ) {
        if (oldValues == null || newValues == null) return null;
        
        return newValues.entrySet().stream()
            .filter(entry -> {
                Object oldVal = oldValues.get(entry.getKey());
                Object newVal = entry.getValue();
                return !java.util.Objects.equals(oldVal, newVal);
            })
            .map(Map.Entry::getKey)
            .collect(java.util.stream.Collectors.joining(","));
    }

    private String getClientIp() {
        try {
            var attrs = (ServletRequestAttributes) RequestContextHolder.currentRequestAttributes();
            String xff = attrs.getRequest().getHeader("X-Forwarded-For");
            return xff != null ? xff.split(",")[0].trim() : attrs.getRequest().getRemoteAddr();
        } catch (Exception e) {
            return "UNKNOWN";
        }
    }

    private String getUserAgent() {
        try {
            var attrs = (ServletRequestAttributes) RequestContextHolder.currentRequestAttributes();
            return attrs.getRequest().getHeader("User-Agent");
        } catch (Exception e) {
            return null;
        }
    }

    private String getSessionId() {
        try {
            var attrs = (ServletRequestAttributes) RequestContextHolder.currentRequestAttributes();
            return attrs.getRequest().getSession(false) != null ?
                attrs.getRequest().getSession().getId() : null;
        } catch (Exception e) {
            return null;
        }
    }

    private String getRequestId() {
        try {
            var attrs = (ServletRequestAttributes) RequestContextHolder.currentRequestAttributes();
            return attrs.getRequest().getHeader("X-Request-Id");
        } catch (Exception e) {
            return null;
        }
    }
}
```

---

## 2934-2939: Hibernate Envers สำหรับ Entity History

Hibernate Envers เป็น module ที่บันทึกประวัติทุกการเปลี่ยนแปลงของ entity โดยอัตโนมัติ

### Dependencies

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.hibernate.orm</groupId>
    <artifactId>hibernate-envers</artifactId>
</dependency>
```

### Entity ที่ track ด้วย Envers

```java
// Order.java
package com.example.audit.entity;

import jakarta.persistence.*;
import lombok.*;
import org.hibernate.envers.Audited;
import org.hibernate.envers.NotAudited;

import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.List;

@Entity
@Table(name = "orders")
@Audited  // บอกให้ Envers track entity นี้
@Getter
@Setter
@NoArgsConstructor
public class Order extends AuditableEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, unique = true)
    private String orderNumber;

    @Column(nullable = false)
    private String customerId;

    @Enumerated(EnumType.STRING)
    private OrderStatus status;

    @Column(nullable = false, precision = 12, scale = 2)
    private BigDecimal totalAmount;

    // Track การเปลี่ยนแปลงของ items ด้วย
    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL)
    @Audited  // Audit relation ด้วย
    private List<OrderItem> items;

    // ไม่ track field นี้ใน audit
    @NotAudited
    @Column(name = "internal_notes", length = 2000)
    private String internalNotes;
}
```

### Envers Configuration

```yaml
# application.yml
spring:
  jpa:
    properties:
      org:
        hibernate:
          envers:
            audit_table_suffix: _audit    # ชื่อ table จะเป็น orders_audit
            revision_field_name: rev      # ชื่อ column สำหรับ revision
            revision_type_field_name: rev_type  # 0=ADD, 1=MOD, 2=DEL
            store_data_at_delete: true    # เก็บข้อมูลก่อนลบ
            global_with_modified_flag: true  # บันทึกว่า field ไหนเปลี่ยน
```

### Custom Revision Entity

```java
// CustomRevisionEntity.java
package com.example.audit.entity;

import jakarta.persistence.*;
import lombok.*;
import org.hibernate.envers.RevisionEntity;
import org.hibernate.envers.RevisionNumber;
import org.hibernate.envers.RevisionTimestamp;

@Entity
@Table(name = "revisions")
@RevisionEntity(CustomRevisionListener.class)
@Getter
@Setter
public class CustomRevisionEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "revision_seq")
    @SequenceGenerator(name = "revision_seq", sequenceName = "revision_sequence")
    @RevisionNumber
    private int id;

    @RevisionTimestamp
    private long timestamp;

    // ข้อมูลเพิ่มเติมที่ต้องการเก็บ
    @Column(name = "username", length = 100)
    private String username;

    @Column(name = "ip_address", length = 45)
    private String ipAddress;

    @Column(name = "user_agent", length = 500)
    private String userAgent;
}
```

### Custom Revision Listener

```java
// CustomRevisionListener.java
package com.example.audit.entity;

import org.hibernate.envers.RevisionListener;
import org.springframework.security.core.Authentication;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.web.context.request.RequestContextHolder;
import org.springframework.web.context.request.ServletRequestAttributes;

public class CustomRevisionListener implements RevisionListener {

    @Override
    public void newRevision(Object revisionEntity) {
        CustomRevisionEntity revision = (CustomRevisionEntity) revisionEntity;
        
        // บันทึก username
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        if (auth != null && auth.isAuthenticated()) {
            revision.setUsername(auth.getName());
        }
        
        // บันทึก IP
        try {
            var attrs = (ServletRequestAttributes) RequestContextHolder.currentRequestAttributes();
            String xff = attrs.getRequest().getHeader("X-Forwarded-For");
            revision.setIpAddress(xff != null ? xff.split(",")[0].trim() : 
                attrs.getRequest().getRemoteAddr());
            revision.setUserAgent(attrs.getRequest().getHeader("User-Agent"));
        } catch (Exception ignored) {}
    }
}
```

### Envers Query Service

```java
// EntityHistoryService.java
package com.example.audit.service;

import com.example.audit.entity.*;
import jakarta.persistence.EntityManager;
import lombok.RequiredArgsConstructor;
import org.hibernate.envers.AuditReader;
import org.hibernate.envers.AuditReaderFactory;
import org.hibernate.envers.DefaultRevisionEntity;
import org.hibernate.envers.query.AuditEntity;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.Date;
import java.util.List;

@Service
@RequiredArgsConstructor
@Transactional(readOnly = true)
public class EntityHistoryService {

    private final EntityManager entityManager;

    // ดึงประวัติทั้งหมดของ entity
    public List<Order> getOrderHistory(Long orderId) {
        AuditReader reader = AuditReaderFactory.get(entityManager);
        
        return reader.createQuery()
            .forRevisionsOfEntity(Order.class, true, true)
            .add(AuditEntity.id().eq(orderId))
            .addOrder(AuditEntity.revisionNumber().asc())
            .getResultList();
    }

    // ดึง revision ที่ N ของ entity
    public Order getOrderAtRevision(Long orderId, int revisionNumber) {
        AuditReader reader = AuditReaderFactory.get(entityManager);
        return reader.find(Order.class, orderId, revisionNumber);
    }

    // ดึง entity ณ เวลาที่กำหนด
    public Order getOrderAtTime(Long orderId, Date date) {
        AuditReader reader = AuditReaderFactory.get(entityManager);
        return reader.find(Order.class, orderId, date);
    }

    // ดึงรายการ revisions ทั้งหมดของ entity
    public List<Number> getOrderRevisions(Long orderId) {
        AuditReader reader = AuditReaderFactory.get(entityManager);
        return reader.getRevisions(Order.class, orderId);
    }

    // ดึง entities ที่ถูก modify โดย user คนนี้
    public List<Object[]> getChangesBy(String username) {
        AuditReader reader = AuditReaderFactory.get(entityManager);
        
        return reader.createQuery()
            .forRevisionsOfEntity(Order.class, false, true)
            .add(AuditEntity.revisionProperty("username").eq(username))
            .getResultList();
    }

    // ดึง entities ที่ถูก modify หลังจากเวลาที่กำหนด
    public List<Order> getOrdersModifiedAfter(Date date) {
        AuditReader reader = AuditReaderFactory.get(entityManager);
        
        return reader.createQuery()
            .forRevisionsOfEntity(Order.class, true, false)
            .add(AuditEntity.revisionProperty("timestamp").gt(date.getTime()))
            .getResultList();
    }
}
```

---

## 2940-2945: Audit Search และ Reporting

### Audit Log Repository

```java
// AuditLogRepository.java
package com.example.audit.repository;

import com.example.audit.entity.AuditLog;
import com.example.audit.entity.AuditLog.AuditAction;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.JpaSpecificationExecutor;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;

import java.time.LocalDateTime;
import java.util.List;

public interface AuditLogRepository extends JpaRepository<AuditLog, Long>,
    JpaSpecificationExecutor<AuditLog> {

    Page<AuditLog> findByEntityTypeAndEntityId(
        String entityType,
        String entityId,
        Pageable pageable
    );

    Page<AuditLog> findByPerformedBy(String performedBy, Pageable pageable);

    Page<AuditLog> findByActionAndPerformedAtBetween(
        AuditAction action,
        LocalDateTime from,
        LocalDateTime to,
        Pageable pageable
    );

    @Query("""
        SELECT al FROM AuditLog al
        WHERE al.entityType = :entityType
          AND al.performedAt >= :from
          AND al.performedAt <= :to
        ORDER BY al.performedAt DESC
        """)
    Page<AuditLog> findByEntityTypeAndDateRange(
        @Param("entityType") String entityType,
        @Param("from") LocalDateTime from,
        @Param("to") LocalDateTime to,
        Pageable pageable
    );

    @Query("""
        SELECT al.performedBy, COUNT(al), al.action
        FROM AuditLog al
        WHERE al.performedAt >= :from
        GROUP BY al.performedBy, al.action
        ORDER BY COUNT(al) DESC
        """)
    List<Object[]> getUserActivitySummary(@Param("from") LocalDateTime from);
}
```

### Audit Search API

```java
// AuditLogController.java
package com.example.audit.controller;

import com.example.audit.entity.AuditLog;
import com.example.audit.entity.AuditLog.AuditAction;
import com.example.audit.repository.AuditLogRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.jpa.domain.Specification;
import org.springframework.data.web.PageableDefault;
import org.springframework.format.annotation.DateTimeFormat;
import org.springframework.http.ResponseEntity;
import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.web.bind.annotation.*;

import java.time.LocalDateTime;

@RestController
@RequestMapping("/api/v1/audit-logs")
@RequiredArgsConstructor
@PreAuthorize("hasRole('AUDITOR')")
public class AuditLogController {

    private final AuditLogRepository auditLogRepository;

    @GetMapping
    public ResponseEntity<Page<AuditLog>> searchAuditLogs(
        @RequestParam(required = false) String entityType,
        @RequestParam(required = false) String entityId,
        @RequestParam(required = false) String performedBy,
        @RequestParam(required = false) AuditAction action,
        @RequestParam(required = false) 
            @DateTimeFormat(iso = DateTimeFormat.ISO.DATE_TIME) LocalDateTime from,
        @RequestParam(required = false) 
            @DateTimeFormat(iso = DateTimeFormat.ISO.DATE_TIME) LocalDateTime to,
        @PageableDefault(size = 20, sort = "performedAt") Pageable pageable
    ) {
        Specification<AuditLog> spec = buildSpec(entityType, entityId, performedBy, action, from, to);
        return ResponseEntity.ok(auditLogRepository.findAll(spec, pageable));
    }

    @GetMapping("/{entityType}/{entityId}")
    public ResponseEntity<Page<AuditLog>> getEntityHistory(
        @PathVariable String entityType,
        @PathVariable String entityId,
        @PageableDefault(size = 20) Pageable pageable
    ) {
        return ResponseEntity.ok(
            auditLogRepository.findByEntityTypeAndEntityId(entityType, entityId, pageable)
        );
    }

    private Specification<AuditLog> buildSpec(
        String entityType, String entityId, String performedBy,
        AuditAction action, LocalDateTime from, LocalDateTime to
    ) {
        return (root, query, cb) -> {
            var predicates = new java.util.ArrayList<jakarta.persistence.criteria.Predicate>();
            
            if (entityType != null) 
                predicates.add(cb.equal(root.get("entityType"), entityType));
            if (entityId != null)
                predicates.add(cb.equal(root.get("entityId"), entityId));
            if (performedBy != null)
                predicates.add(cb.like(root.get("performedBy"), "%" + performedBy + "%"));
            if (action != null)
                predicates.add(cb.equal(root.get("action"), action));
            if (from != null)
                predicates.add(cb.greaterThanOrEqualTo(root.get("performedAt"), from));
            if (to != null)
                predicates.add(cb.lessThanOrEqualTo(root.get("performedAt"), to));
            
            return cb.and(predicates.toArray(new jakarta.persistence.criteria.Predicate[0]));
        };
    }
}
```

---

## 2946-2951: GDPR-Compliant Audit

ภายใต้ GDPR เราต้องสามารถลบข้อมูลส่วนตัวของ user ได้ แต่ยังต้องรักษา audit trail ไว้

### Anonymization Service

```java
// AuditAnonymizationService.java
package com.example.audit.service;

import com.example.audit.entity.AuditLog;
import com.example.audit.repository.AuditLogRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.time.LocalDateTime;
import java.util.*;

@Slf4j
@Service
@RequiredArgsConstructor
public class AuditAnonymizationService {

    private final AuditLogRepository auditLogRepository;
    private final PseudonymizationService pseudonymizationService;

    // ทำ pseudonymization ของ audit logs เมื่อได้รับ GDPR request
    @Transactional
    public void anonymizeUser(String userId) {
        log.info("Anonymizing audit logs for user: {}", userId);
        
        // หา audit logs ที่เกี่ยวกับ user นี้
        List<AuditLog> logs = auditLogRepository.findByPerformedBy(userId, 
            org.springframework.data.domain.Pageable.unpaged()).getContent();
        
        // สร้าง pseudonym ที่คงที่สำหรับ user นี้
        String pseudonym = pseudonymizationService.getPseudonym(userId);
        
        for (AuditLog auditLog : logs) {
            // แทนที่ username ด้วย pseudonym
            auditLog.setPerformedBy(pseudonym);
            
            // ลบ IP address
            auditLog.setIpAddress("ANONYMIZED");
            
            // ลบ User Agent
            auditLog.setUserAgent(null);
            
            // ลบข้อมูลส่วนตัวใน old/new values
            anonymizePersonalData(auditLog);
        }
        
        auditLogRepository.saveAll(logs);
        log.info("Anonymized {} audit logs for user: {}", logs.size(), pseudonym);
    }

    private void anonymizePersonalData(AuditLog log) {
        // Fields ที่ถือว่าเป็น personal data
        Set<String> personalFields = Set.of(
            "email", "phoneNumber", "fullName", "dateOfBirth",
            "address", "nationalId", "passport"
        );
        
        if (log.getOldValues() != null) {
            anonymizeFields(log.getOldValues(), personalFields);
        }
        if (log.getNewValues() != null) {
            anonymizeFields(log.getNewValues(), personalFields);
        }
    }

    private void anonymizeFields(Map<String, Object> values, Set<String> fieldsToAnonymize) {
        fieldsToAnonymize.forEach(field -> {
            if (values.containsKey(field)) {
                values.put(field, "[REDACTED]");
            }
        });
    }

    // ลบ audit logs เก่าตามนโยบายการเก็บข้อมูล
    @Scheduled(cron = "0 0 2 1 * *")  // รันทุกวันที่ 1 ของเดือน
    @Transactional
    public void purgeOldAuditLogs() {
        int retentionDays = 365 * 7;  // 7 ปี ตาม requirement
        LocalDateTime cutoff = LocalDateTime.now().minusDays(retentionDays);
        
        log.info("Purging audit logs older than: {}", cutoff);
        // ลบ logs ที่เก่ากว่า retention period
        // auditLogRepository.deleteByPerformedAtBefore(cutoff);
        log.info("Audit log purge completed");
    }
}
```

### Pseudonymization Service

```java
// PseudonymizationService.java
package com.example.audit.service;

import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;

import java.security.MessageDigest;
import java.util.Base64;

@Service
@RequiredArgsConstructor
public class PseudonymizationService {

    private final String saltValue = "audit-salt-v1";  // ในระบบจริงเก็บใน secrets

    // สร้าง pseudonym ที่ deterministic แต่ไม่สามารถ reverse ได้
    public String getPseudonym(String userId) {
        try {
            MessageDigest digest = MessageDigest.getInstance("SHA-256");
            String input = saltValue + userId;
            byte[] hash = digest.digest(input.getBytes());
            return "ANON-" + Base64.getUrlEncoder()
                .withoutPadding()
                .encodeToString(hash)
                .substring(0, 16)
                .toUpperCase();
        } catch (Exception e) {
            throw new RuntimeException("Failed to create pseudonym", e);
        }
    }
}
```

---

## 2952-2957: Audit Event Publishing ไปยัง Kafka

### Audit Event

```java
// AuditEvent.java
package com.example.audit.event;

import com.example.audit.entity.AuditLog.AuditAction;
import lombok.*;

import java.time.LocalDateTime;
import java.util.Map;

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class AuditEvent {
    private String eventId;
    private String entityType;
    private String entityId;
    private AuditAction action;
    private String performedBy;
    private LocalDateTime performedAt;
    private String ipAddress;
    private Map<String, Object> oldValues;
    private Map<String, Object> newValues;
    private String changedFields;
    private String correlationId;
    private Map<String, String> metadata;
}
```

### Kafka Audit Producer

```java
// KafkaAuditProducer.java
package com.example.audit.kafka;

import com.example.audit.event.AuditEvent;
import com.fasterxml.jackson.databind.ObjectMapper;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.kafka.support.SendResult;
import org.springframework.stereotype.Component;

import java.util.UUID;
import java.util.concurrent.CompletableFuture;

@Slf4j
@Component
@RequiredArgsConstructor
public class KafkaAuditProducer {

    private static final String AUDIT_TOPIC = "audit-events";
    
    private final KafkaTemplate<String, AuditEvent> kafkaTemplate;

    public void publishAuditEvent(AuditEvent event) {
        if (event.getEventId() == null) {
            event.setEventId(UUID.randomUUID().toString());
        }
        
        // ใช้ entityType + entityId เป็น partition key เพื่อให้ events ของ entity เดียวกัน
        // ไปยัง partition เดียวกัน (รักษา ordering)
        String key = event.getEntityType() + ":" + event.getEntityId();
        
        CompletableFuture<SendResult<String, AuditEvent>> future = 
            kafkaTemplate.send(AUDIT_TOPIC, key, event);
        
        future.whenComplete((result, ex) -> {
            if (ex != null) {
                log.error("Failed to publish audit event for {}/{}", 
                    event.getEntityType(), event.getEntityId(), ex);
                // Fallback: บันทึกลง database โดยตรง
                fallbackSaveToDatabase(event);
            } else {
                log.debug("Published audit event: {} to partition: {}",
                    event.getEventId(),
                    result.getRecordMetadata().partition());
            }
        });
    }

    private void fallbackSaveToDatabase(AuditEvent event) {
        // บันทึกลง outbox table เพื่อ retry ทีหลัง
        log.warn("Using database fallback for audit event: {}", event.getEventId());
    }
}
```

### Kafka Audit Consumer (สำหรับ Audit Service แยก)

```java
// KafkaAuditConsumer.java
package com.example.audit.kafka;

import com.example.audit.entity.AuditLog;
import com.example.audit.event.AuditEvent;
import com.example.audit.repository.AuditLogRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.kafka.support.Acknowledgment;
import org.springframework.messaging.handler.annotation.Payload;
import org.springframework.stereotype.Component;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;

@Slf4j
@Component
@RequiredArgsConstructor
public class KafkaAuditConsumer {

    private final AuditLogRepository auditLogRepository;

    @KafkaListener(
        topics = "audit-events",
        groupId = "audit-service",
        containerFactory = "batchKafkaListenerContainerFactory"
    )
    @Transactional
    public void consumeAuditEvents(
        @Payload List<AuditEvent> events,
        Acknowledgment acknowledgment
    ) {
        log.info("Received {} audit events", events.size());
        
        try {
            List<AuditLog> auditLogs = events.stream()
                .map(this::toAuditLog)
                .toList();
            
            auditLogRepository.saveAll(auditLogs);
            acknowledgment.acknowledge();
            
            log.info("Saved {} audit events to database", auditLogs.size());
        } catch (Exception e) {
            log.error("Failed to process audit events", e);
            // ไม่ acknowledge เพื่อให้ retry
        }
    }

    private AuditLog toAuditLog(AuditEvent event) {
        return AuditLog.builder()
            .entityType(event.getEntityType())
            .entityId(event.getEntityId())
            .action(event.getAction())
            .performedBy(event.getPerformedBy())
            .performedAt(event.getPerformedAt())
            .ipAddress(event.getIpAddress())
            .oldValues(event.getOldValues())
            .newValues(event.getNewValues())
            .changedFields(event.getChangedFields())
            .correlationId(event.getCorrelationId())
            .build();
    }
}
```

---

## 2958-2960: Audit Aspect สำหรับ Auto-logging

```java
// AuditAspect.java
package com.example.audit.aspect;

import com.example.audit.annotation.Auditable;
import com.example.audit.entity.AuditLog.AuditAction;
import com.example.audit.kafka.KafkaAuditProducer;
import com.example.audit.event.AuditEvent;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.reflect.MethodSignature;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.stereotype.Component;

import java.lang.reflect.Method;
import java.time.LocalDateTime;
import java.util.UUID;

@Slf4j
@Aspect
@Component
@RequiredArgsConstructor
public class AuditAspect {

    private final KafkaAuditProducer auditProducer;

    @Around("@annotation(auditable)")
    public Object auditMethod(ProceedingJoinPoint joinPoint, Auditable auditable) throws Throwable {
        String entityType = auditable.entityType();
        AuditAction action = auditable.action();
        
        String performedBy = SecurityContextHolder.getContext()
            .getAuthentication() != null ?
            SecurityContextHolder.getContext().getAuthentication().getName() :
            "SYSTEM";
        
        Object result = null;
        try {
            result = joinPoint.proceed();
            
            // Publish audit event หลังจาก method สำเร็จ
            AuditEvent event = AuditEvent.builder()
                .eventId(UUID.randomUUID().toString())
                .entityType(entityType)
                .action(action)
                .performedBy(performedBy)
                .performedAt(LocalDateTime.now())
                .build();
            
            auditProducer.publishAuditEvent(event);
            
        } catch (Exception e) {
            log.error("Audit method failed: {}", joinPoint.getSignature().getName(), e);
            throw e;
        }
        
        return result;
    }
}
```

```java
// Auditable.java - annotation
package com.example.audit.annotation;

import com.example.audit.entity.AuditLog.AuditAction;

import java.lang.annotation.*;

@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Auditable {
    String entityType();
    AuditAction action() default AuditAction.UPDATE;
    String entityIdParam() default "";
}
```

---

## สรุป Part 83

ในบทนี้เราได้เรียนรู้:

1. **Spring Data JPA Auditing** - @CreatedBy, @LastModifiedBy กับ AuditorAware
2. **Hibernate Envers** - Entity history tracking อัตโนมัติ
3. **Custom Audit Log** - บันทึก audit ใน table แยก พร้อม JSON fields
4. **GDPR Compliance** - Anonymization และ pseudonymization
5. **Kafka Integration** - Publish audit events แบบ async
6. **Audit Aspect** - Auto-logging ด้วย @Auditable annotation

---

*[← Part 82: Rate Limiting](./part-82-rate-limiting.md) | [Part 84: Internationalization →](./part-84-internationalization.md)*

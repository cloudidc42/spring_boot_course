# Part 115: โปรเจค 71-75 — Analytics & Monitoring

**ระดับ:** ระดับโลก (World-Class)
**เวลา:** 10-15 ชั่วโมง
**เป้าหมาย:** สร้างระบบ Analytics และ Monitoring ระดับ Enterprise ครอบคลุม Analytics Dashboard, A/B Testing, Application Monitoring, Log Management และ Data Pipeline Service

---

*[← Part 114: Education & Learning](./part-114-education-learning.md) | [Part 116: Advanced APIs →](./part-116-advanced-apis.md)*

---

## โปรเจค 71: Analytics Dashboard API

### ภาพรวมระบบ

Analytics Dashboard API เป็นระบบรวบรวมและวิเคราะห์ข้อมูลพฤติกรรมผู้ใช้จากเว็บไซต์และแอปพลิเคชัน ระบบรองรับการติดตาม Pageviews, Click Events, Conversions วิเคราะห์ Session, Funnel และ Retention Cohorts พร้อม Export ข้อมูล

- **Event Tracking**: บันทึก Events ทุกประเภท
- **Session Analysis**: วิเคราะห์ Session ผู้ใช้
- **Funnel Analysis**: วิเคราะห์ Conversion Funnel
- **Retention Cohorts**: คำนวณ Retention Rate
- **Custom Reports**: สร้างรายงานแบบกำหนดเอง
- **Export to CSV/Excel**: Export ข้อมูลสำหรับ BI Tools

### Flyway Migration

```sql
-- V1__create_analytics_tables.sql
CREATE TABLE analytics_sites (
    id BIGSERIAL PRIMARY KEY,
    site_id VARCHAR(50) NOT NULL UNIQUE,
    name VARCHAR(200) NOT NULL,
    domain VARCHAR(500),
    owner_id BIGINT NOT NULL,
    tracking_enabled BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE analytics_events (
    id BIGSERIAL PRIMARY KEY,
    site_id VARCHAR(50) NOT NULL,
    session_id VARCHAR(100) NOT NULL,
    visitor_id VARCHAR(100),
    user_id BIGINT,
    event_type VARCHAR(50) NOT NULL,
    event_name VARCHAR(200),
    page_url TEXT,
    referrer TEXT,
    properties JSONB,
    ip_address VARCHAR(45),
    user_agent TEXT,
    country VARCHAR(100),
    city VARCHAR(100),
    device_type VARCHAR(20),
    browser VARCHAR(100),
    os VARCHAR(100),
    screen_resolution VARCHAR(20),
    occurred_at TIMESTAMP NOT NULL DEFAULT NOW()
) PARTITION BY RANGE (occurred_at);

CREATE TABLE analytics_sessions (
    id BIGSERIAL PRIMARY KEY,
    site_id VARCHAR(50) NOT NULL,
    session_id VARCHAR(100) NOT NULL UNIQUE,
    visitor_id VARCHAR(100),
    user_id BIGINT,
    started_at TIMESTAMP NOT NULL,
    ended_at TIMESTAMP,
    duration_seconds INT,
    page_views INT DEFAULT 0,
    events_count INT DEFAULT 0,
    entry_page TEXT,
    exit_page TEXT,
    is_bounce BOOLEAN DEFAULT FALSE,
    referrer TEXT,
    utm_source VARCHAR(200),
    utm_medium VARCHAR(200),
    utm_campaign VARCHAR(200),
    country VARCHAR(100),
    device_type VARCHAR(20),
    browser VARCHAR(100),
    converted BOOLEAN DEFAULT FALSE
);

CREATE TABLE funnel_definitions (
    id BIGSERIAL PRIMARY KEY,
    site_id VARCHAR(50) NOT NULL,
    name VARCHAR(200) NOT NULL,
    description TEXT,
    steps JSONB NOT NULL,
    created_by BIGINT,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE custom_reports (
    id BIGSERIAL PRIMARY KEY,
    site_id VARCHAR(50) NOT NULL,
    name VARCHAR(200) NOT NULL,
    description TEXT,
    report_type VARCHAR(50) NOT NULL,
    config JSONB NOT NULL,
    created_by BIGINT,
    is_public BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

-- Create partitions for current and next month
CREATE TABLE analytics_events_2026_01 PARTITION OF analytics_events
    FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
CREATE TABLE analytics_events_2026_02 PARTITION OF analytics_events
    FOR VALUES FROM ('2026-02-01') TO ('2026-03-01');

CREATE INDEX idx_events_site_type_time ON analytics_events(site_id, event_type, occurred_at DESC);
CREATE INDEX idx_events_session ON analytics_events(session_id);
CREATE INDEX idx_sessions_site_started ON analytics_sessions(site_id, started_at DESC);
CREATE INDEX idx_sessions_visitor ON analytics_sessions(visitor_id);
```

### Entity

```java
// AnalyticsEvent.java
package com.analytics.entity;

import jakarta.persistence.*;
import lombok.*;
import org.hibernate.annotations.JdbcTypeCode;
import org.hibernate.type.SqlTypes;
import java.time.LocalDateTime;
import java.util.Map;

@Entity
@Table(name = "analytics_events")
@Getter @Setter @Builder
@NoArgsConstructor @AllArgsConstructor
public class AnalyticsEvent {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "site_id", nullable = false)
    private String siteId;

    @Column(name = "session_id", nullable = false)
    private String sessionId;

    @Column(name = "visitor_id")
    private String visitorId;

    @Column(name = "user_id")
    private Long userId;

    @Column(name = "event_type", nullable = false)
    private String eventType;

    @Column(name = "event_name")
    private String eventName;

    @Column(name = "page_url")
    private String pageUrl;

    @Column(name = "referrer")
    private String referrer;

    @JdbcTypeCode(SqlTypes.JSON)
    @Column(name = "properties")
    private Map<String, Object> properties;

    @Column(name = "country")
    private String country;

    @Column(name = "device_type")
    private String deviceType;

    @Column(name = "browser")
    private String browser;

    @Column(name = "occurred_at", nullable = false)
    private LocalDateTime occurredAt = LocalDateTime.now();
}

// AnalyticsSession.java
@Entity
@Table(name = "analytics_sessions")
@Getter @Setter @Builder
@NoArgsConstructor @AllArgsConstructor
public class AnalyticsSession {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "site_id", nullable = false)
    private String siteId;

    @Column(name = "session_id", nullable = false, unique = true)
    private String sessionId;

    @Column(name = "visitor_id")
    private String visitorId;

    @Column(name = "user_id")
    private Long userId;

    @Column(name = "started_at", nullable = false)
    private LocalDateTime startedAt;

    @Column(name = "ended_at")
    private LocalDateTime endedAt;

    @Column(name = "duration_seconds")
    private Integer durationSeconds;

    @Column(name = "page_views")
    private Integer pageViews = 0;

    @Column(name = "events_count")
    private Integer eventsCount = 0;

    @Column(name = "entry_page")
    private String entryPage;

    @Column(name = "utm_source")
    private String utmSource;

    @Column(name = "utm_medium")
    private String utmMedium;

    @Column(name = "utm_campaign")
    private String utmCampaign;

    @Column(name = "is_bounce")
    private Boolean isBounce = false;

    @Column(name = "converted")
    private Boolean converted = false;
}
```

### Service

```java
// AnalyticsService.java
package com.analytics.service;

import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import java.time.LocalDate;
import java.time.LocalDateTime;
import java.util.*;

@Service
@RequiredArgsConstructor
public class AnalyticsService {

    private final AnalyticsEventRepository eventRepository;
    private final AnalyticsSessionRepository sessionRepository;
    private final FunnelRepository funnelRepository;

    @Transactional
    public AnalyticsEvent track(String siteId, String sessionId, String visitorId,
                                  Long userId, String eventType, String eventName,
                                  String pageUrl, String referrer,
                                  Map<String, Object> properties, String country,
                                  String deviceType, String browser) {
        AnalyticsEvent event = AnalyticsEvent.builder()
                .siteId(siteId).sessionId(sessionId).visitorId(visitorId).userId(userId)
                .eventType(eventType).eventName(eventName).pageUrl(pageUrl)
                .referrer(referrer).properties(properties).country(country)
                .deviceType(deviceType).browser(browser).occurredAt(LocalDateTime.now()).build();
        event = eventRepository.save(event);

        // Update or create session
        updateSession(siteId, sessionId, visitorId, userId, pageUrl, referrer,
                      eventType, country, deviceType, browser);
        return event;
    }

    private void updateSession(String siteId, String sessionId, String visitorId, Long userId,
                                String pageUrl, String referrer, String eventType,
                                String country, String deviceType, String browser) {
        AnalyticsSession session = sessionRepository.findBySessionId(sessionId)
                .orElseGet(() -> AnalyticsSession.builder()
                    .siteId(siteId).sessionId(sessionId).visitorId(visitorId).userId(userId)
                    .startedAt(LocalDateTime.now()).entryPage(pageUrl).build());

        session.setEndedAt(LocalDateTime.now());
        if (session.getStartedAt() != null) {
            long seconds = java.time.Duration.between(session.getStartedAt(),
                session.getEndedAt()).getSeconds();
            session.setDurationSeconds((int) seconds);
        }
        session.setEventsCount(session.getEventsCount() + 1);
        if ("pageview".equalsIgnoreCase(eventType)) {
            session.setPageViews(session.getPageViews() + 1);
            session.setIsBounce(session.getPageViews() <= 1);
        }
        if ("conversion".equalsIgnoreCase(eventType)) {
            session.setConverted(true);
        }
        sessionRepository.save(session);
    }

    public DashboardMetrics getDashboardMetrics(String siteId, LocalDate from, LocalDate to) {
        LocalDateTime fromDt = from.atStartOfDay();
        LocalDateTime toDt = to.atTime(23, 59, 59);

        long totalSessions = sessionRepository.countBySiteIdAndStartedAtBetween(
            siteId, fromDt, toDt);
        long uniqueVisitors = sessionRepository.countDistinctVisitorsBySiteIdAndStartedAtBetween(
            siteId, fromDt, toDt);
        long pageViews = eventRepository.countBySiteIdAndEventTypeAndOccurredAtBetween(
            siteId, "pageview", fromDt, toDt);
        long conversions = sessionRepository.countBySiteIdAndConvertedTrueAndStartedAtBetween(
            siteId, fromDt, toDt);
        double bounceRate = totalSessions > 0 ?
            (double) sessionRepository.countBySiteIdAndIsBounceAndStartedAtBetween(
                siteId, true, fromDt, toDt) / totalSessions * 100 : 0.0;
        double avgSessionDuration = sessionRepository.avgDurationBySiteIdAndStartedAtBetween(
            siteId, fromDt, toDt);

        return new DashboardMetrics(totalSessions, uniqueVisitors, pageViews, conversions,
            bounceRate, avgSessionDuration);
    }

    public List<FunnelStep> analyzeFunnel(Long funnelId, LocalDate from, LocalDate to) {
        FunnelDefinition funnel = funnelRepository.findById(funnelId).orElseThrow();
        LocalDateTime fromDt = from.atStartOfDay();
        LocalDateTime toDt = to.atTime(23, 59, 59);

        List<Map<String, Object>> steps = funnel.getSteps();
        List<FunnelStep> result = new ArrayList<>();

        long prevCount = -1;
        for (Map<String, Object> step : steps) {
            String eventType = (String) step.get("eventType");
            String eventName = (String) step.get("eventName");
            long count = eventRepository.countDistinctSessionsByEventTypeAndName(
                funnel.getSiteId(), eventType, eventName, fromDt, toDt);
            double dropOff = prevCount > 0 ? (1.0 - (double) count / prevCount) * 100 : 0.0;
            result.add(new FunnelStep((String) step.get("name"), count, dropOff));
            prevCount = count;
        }
        return result;
    }

    public RetentionCohort getRetentionCohort(String siteId, LocalDate cohortStart, int weeks) {
        List<List<Double>> matrix = new ArrayList<>();
        for (int week = 0; week < weeks; week++) {
            LocalDate cohortWeekStart = cohortStart.plusWeeks(week);
            LocalDate cohortWeekEnd = cohortWeekStart.plusDays(6);
            List<String> cohortVisitors = sessionRepository.findVisitorsBySiteIdAndDateRange(
                siteId, cohortWeekStart.atStartOfDay(), cohortWeekEnd.atTime(23, 59, 59));

            List<Double> retention = new ArrayList<>();
            retention.add(100.0); // 100% on week 0
            for (int returnWeek = 1; returnWeek <= weeks - week - 1; returnWeek++) {
                LocalDate retWeekStart = cohortWeekStart.plusWeeks(returnWeek);
                LocalDate retWeekEnd = retWeekStart.plusDays(6);
                long returned = sessionRepository.countReturnedVisitors(
                    siteId, cohortVisitors, retWeekStart.atStartOfDay(),
                    retWeekEnd.atTime(23, 59, 59));
                double rate = cohortVisitors.isEmpty() ? 0.0 :
                    (double) returned / cohortVisitors.size() * 100;
                retention.add(Math.round(rate * 10.0) / 10.0);
            }
            matrix.add(retention);
        }
        return new RetentionCohort(cohortStart, weeks, matrix);
    }

    public byte[] exportToCsv(String siteId, LocalDate from, LocalDate to, String exportType) {
        LocalDateTime fromDt = from.atStartOfDay();
        LocalDateTime toDt = to.atTime(23, 59, 59);

        StringBuilder sb = new StringBuilder();
        if ("events".equals(exportType)) {
            sb.append("id,site_id,session_id,event_type,event_name,page_url,occurred_at\n");
            List<AnalyticsEvent> events = eventRepository
                    .findBySiteIdAndOccurredAtBetweenOrderByOccurredAt(siteId, fromDt, toDt);
            for (AnalyticsEvent e : events) {
                sb.append(String.format("%d,%s,%s,%s,%s,%s,%s\n",
                    e.getId(), e.getSiteId(), e.getSessionId(), e.getEventType(),
                    e.getEventName() != null ? e.getEventName() : "",
                    e.getPageUrl() != null ? e.getPageUrl() : "",
                    e.getOccurredAt()));
            }
        }
        return sb.toString().getBytes(java.nio.charset.StandardCharsets.UTF_8);
    }

    public record DashboardMetrics(long totalSessions, long uniqueVisitors, long pageViews,
        long conversions, double bounceRate, double avgSessionDurationSeconds) {}
    public record FunnelStep(String stepName, long count, double dropOffRate) {}
    public record RetentionCohort(LocalDate cohortStart, int weeks, List<List<Double>> matrix) {}
}
```

### Controller

```java
// AnalyticsController.java
package com.analytics.controller;

import lombok.RequiredArgsConstructor;
import org.springframework.http.HttpHeaders;
import org.springframework.http.MediaType;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import java.time.LocalDate;
import java.util.List;
import java.util.Map;

@RestController
@RequestMapping("/api/v1/analytics")
@RequiredArgsConstructor
public class AnalyticsController {

    private final AnalyticsService analyticsService;

    @PostMapping("/track")
    public ResponseEntity<Void> track(@RequestBody TrackEventRequest request,
            jakarta.servlet.http.HttpServletRequest httpRequest) {
        String ip = httpRequest.getRemoteAddr();
        String ua = httpRequest.getHeader("User-Agent");
        analyticsService.track(request.siteId(), request.sessionId(), request.visitorId(),
            request.userId(), request.eventType(), request.eventName(), request.pageUrl(),
            request.referrer(), request.properties(), null, request.deviceType(), null);
        return ResponseEntity.ok().build();
    }

    @GetMapping("/{siteId}/dashboard")
    public ResponseEntity<AnalyticsService.DashboardMetrics> getDashboard(
            @PathVariable String siteId,
            @RequestParam(required = false) LocalDate from,
            @RequestParam(required = false) LocalDate to) {
        LocalDate endDate = to != null ? to : LocalDate.now();
        LocalDate startDate = from != null ? from : endDate.minusDays(30);
        return ResponseEntity.ok(analyticsService.getDashboardMetrics(siteId, startDate, endDate));
    }

    @GetMapping("/funnels/{funnelId}/analysis")
    public ResponseEntity<List<AnalyticsService.FunnelStep>> analyzeFunnel(
            @PathVariable Long funnelId,
            @RequestParam(required = false) LocalDate from,
            @RequestParam(required = false) LocalDate to) {
        LocalDate endDate = to != null ? to : LocalDate.now();
        LocalDate startDate = from != null ? from : endDate.minusDays(30);
        return ResponseEntity.ok(analyticsService.analyzeFunnel(funnelId, startDate, endDate));
    }

    @GetMapping("/{siteId}/retention")
    public ResponseEntity<AnalyticsService.RetentionCohort> getRetention(
            @PathVariable String siteId,
            @RequestParam(required = false) LocalDate cohortStart,
            @RequestParam(defaultValue = "8") int weeks) {
        LocalDate start = cohortStart != null ? cohortStart : LocalDate.now().minusWeeks(weeks);
        return ResponseEntity.ok(analyticsService.getRetentionCohort(siteId, start, weeks));
    }

    @GetMapping("/{siteId}/export")
    public ResponseEntity<byte[]> export(@PathVariable String siteId,
            @RequestParam String type,
            @RequestParam(required = false) LocalDate from,
            @RequestParam(required = false) LocalDate to) {
        LocalDate endDate = to != null ? to : LocalDate.now();
        LocalDate startDate = from != null ? from : endDate.minusDays(30);
        byte[] data = analyticsService.exportToCsv(siteId, startDate, endDate, type);
        return ResponseEntity.ok()
                .header(HttpHeaders.CONTENT_DISPOSITION,
                    "attachment; filename=analytics-" + type + ".csv")
                .contentType(MediaType.parseMediaType("text/csv"))
                .body(data);
    }

    record TrackEventRequest(String siteId, String sessionId, String visitorId, Long userId,
        String eventType, String eventName, String pageUrl, String referrer,
        Map<String, Object> properties, String deviceType) {}
}
```

### docker-compose.yml

```yaml
# docker-compose.yml (Analytics Dashboard API)
version: '3.8'
services:
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: analytics_db
      POSTGRES_USER: analytics_user
      POSTGRES_PASSWORD: analytics_pass
    ports:
      - "5432:5432"
    volumes:
      - analytics_pg_data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

  kafka:
    image: confluentinc/cp-kafka:7.5.0
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
    ports:
      - "9092:9092"
    depends_on:
      - zookeeper

  zookeeper:
    image: confluentinc/cp-zookeeper:7.5.0
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181

  analytics-service:
    build: .
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/analytics_db
      SPRING_DATASOURCE_USERNAME: analytics_user
      SPRING_DATASOURCE_PASSWORD: analytics_pass
      SPRING_REDIS_HOST: redis
      SPRING_KAFKA_BOOTSTRAP_SERVERS: kafka:9092
    ports:
      - "8080:8080"
    depends_on:
      - postgres
      - redis
      - kafka

volumes:
  analytics_pg_data:
```

---

## โปรเจค 72: A/B Testing Platform

### ภาพรวมระบบ

A/B Testing Platform ช่วยให้ทีมผลิตภัณฑ์ทดสอบฟีเจอร์ใหม่กับกลุ่มผู้ใช้แบบควบคุม กำหนดสัดส่วน Traffic สำหรับแต่ละ Variant ติดตาม Conversion คำนวณ Statistical Significance และเลือก Winner อัตโนมัติ

- **Experiments**: สร้างและจัดการการทดลอง
- **Variants**: กำหนด Variant และสัดส่วน Traffic
- **Assignment**: จัดสรรผู้ใช้ไปยัง Variant แบบ Deterministic
- **Conversion Tracking**: บันทึก Conversion Events
- **Statistical Significance**: คำนวณนัยสำคัญทางสถิติ
- **Winner Selection**: เลือก Winner อัตโนมัติหรือ Manual

### Flyway Migration

```sql
-- V1__create_ab_testing_tables.sql
CREATE TABLE experiments (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    description TEXT,
    hypothesis TEXT,
    status VARCHAR(20) NOT NULL DEFAULT 'DRAFT',
    target_metric VARCHAR(200) NOT NULL,
    secondary_metrics TEXT,
    traffic_percentage INT NOT NULL DEFAULT 100,
    min_sample_size INT DEFAULT 1000,
    confidence_level DECIMAL(5,3) DEFAULT 0.95,
    created_by BIGINT NOT NULL,
    started_at TIMESTAMP,
    ended_at TIMESTAMP,
    winner_variant_id BIGINT,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE experiment_variants (
    id BIGSERIAL PRIMARY KEY,
    experiment_id BIGINT NOT NULL REFERENCES experiments(id),
    name VARCHAR(100) NOT NULL,
    description TEXT,
    is_control BOOLEAN NOT NULL DEFAULT FALSE,
    traffic_split DECIMAL(5,2) NOT NULL DEFAULT 50.00,
    config JSONB,
    impressions BIGINT DEFAULT 0,
    conversions BIGINT DEFAULT 0,
    total_value DECIMAL(14,2) DEFAULT 0,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE experiment_assignments (
    id BIGSERIAL PRIMARY KEY,
    experiment_id BIGINT NOT NULL REFERENCES experiments(id),
    variant_id BIGINT NOT NULL REFERENCES experiment_variants(id),
    user_id BIGINT,
    anonymous_id VARCHAR(100),
    assigned_at TIMESTAMP NOT NULL DEFAULT NOW(),
    UNIQUE (experiment_id, user_id),
    UNIQUE (experiment_id, anonymous_id)
);

CREATE TABLE experiment_events (
    id BIGSERIAL PRIMARY KEY,
    experiment_id BIGINT NOT NULL REFERENCES experiments(id),
    variant_id BIGINT NOT NULL REFERENCES experiment_variants(id),
    assignment_id BIGINT REFERENCES experiment_assignments(id),
    event_type VARCHAR(30) NOT NULL,
    event_name VARCHAR(200),
    value DECIMAL(12,2),
    properties JSONB,
    occurred_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_assignments_experiment_user ON experiment_assignments(experiment_id, user_id);
CREATE INDEX idx_experiment_events_variant ON experiment_events(variant_id, event_type);
```

### Entity & Service

```java
// ABTestingService.java
package com.abtesting.service;

import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.*;

@Service
@RequiredArgsConstructor
public class ABTestingService {

    private final ExperimentRepository experimentRepository;
    private final ExperimentVariantRepository variantRepository;
    private final ExperimentAssignmentRepository assignmentRepository;
    private final ExperimentEventRepository eventRepository;

    @Transactional
    public Experiment createExperiment(Long createdBy, String name, String description,
                                        String hypothesis, String targetMetric,
                                        int trafficPercentage) {
        Experiment experiment = Experiment.builder()
                .name(name).description(description).hypothesis(hypothesis)
                .targetMetric(targetMetric).trafficPercentage(trafficPercentage)
                .status("DRAFT").createdBy(createdBy).build();
        return experimentRepository.save(experiment);
    }

    @Transactional
    public ExperimentVariant addVariant(Long experimentId, String name, String description,
                                         boolean isControl, BigDecimal trafficSplit,
                                         Map<String, Object> config) {
        ExperimentVariant variant = ExperimentVariant.builder()
                .experimentId(experimentId).name(name).description(description)
                .isControl(isControl).trafficSplit(trafficSplit).config(config).build();
        return variantRepository.save(variant);
    }

    @Transactional
    public ExperimentAssignment assignVariant(Long experimentId, Long userId, String anonymousId) {
        // Check existing assignment
        Optional<ExperimentAssignment> existing = userId != null ?
            assignmentRepository.findByExperimentIdAndUserId(experimentId, userId) :
            assignmentRepository.findByExperimentIdAndAnonymousId(experimentId, anonymousId);

        if (existing.isPresent()) return existing.get();

        Experiment experiment = experimentRepository.findById(experimentId).orElseThrow();
        if (!"RUNNING".equals(experiment.getStatus())) {
            throw new IllegalStateException("Experiment is not running");
        }

        // Deterministic assignment based on user ID hash
        String hashKey = userId != null ? userId.toString() : anonymousId;
        Long variantId = assignVariantDeterministically(experimentId, hashKey,
            experiment.getTrafficPercentage());
        if (variantId == null) return null; // Not in experiment traffic

        ExperimentAssignment assignment = ExperimentAssignment.builder()
                .experimentId(experimentId).variantId(variantId)
                .userId(userId).anonymousId(anonymousId).build();
        assignment = assignmentRepository.save(assignment);

        // Record impression
        ExperimentVariant variant = variantRepository.findById(variantId).orElseThrow();
        variant.setImpressions(variant.getImpressions() + 1);
        variantRepository.save(variant);

        return assignment;
    }

    @Transactional
    public void recordConversion(Long experimentId, Long userId, String anonymousId,
                                   String eventName, BigDecimal value) {
        ExperimentAssignment assignment = userId != null ?
            assignmentRepository.findByExperimentIdAndUserId(experimentId, userId).orElse(null) :
            assignmentRepository.findByExperimentIdAndAnonymousId(experimentId, anonymousId).orElse(null);

        if (assignment == null) return;

        ExperimentEvent event = ExperimentEvent.builder()
                .experimentId(experimentId).variantId(assignment.getVariantId())
                .assignmentId(assignment.getId()).eventType("CONVERSION")
                .eventName(eventName).value(value).build();
        eventRepository.save(event);

        ExperimentVariant variant = variantRepository.findById(assignment.getVariantId()).orElseThrow();
        variant.setConversions(variant.getConversions() + 1);
        if (value != null) variant.setTotalValue(variant.getTotalValue().add(value));
        variantRepository.save(variant);
    }

    public StatisticalSignificance calculateSignificance(Long experimentId) {
        List<ExperimentVariant> variants = variantRepository.findByExperimentId(experimentId);
        ExperimentVariant control = variants.stream()
                .filter(ExperimentVariant::getIsControl).findFirst()
                .orElseThrow(() -> new RuntimeException("No control variant"));

        List<VariantResult> results = new ArrayList<>();
        for (ExperimentVariant v : variants) {
            double cvr = v.getImpressions() > 0 ?
                (double) v.getConversions() / v.getImpressions() : 0.0;
            double improvement = control.getImpressions() > 0 && control.getConversions() > 0 ?
                (cvr - (double) control.getConversions() / control.getImpressions()) /
                ((double) control.getConversions() / control.getImpressions()) * 100 : 0.0;
            double pValue = calculatePValue(control, v);
            boolean significant = pValue < 0.05;
            results.add(new VariantResult(v.getName(), v.getImpressions(), v.getConversions(),
                cvr * 100, improvement, pValue, significant));
        }
        return new StatisticalSignificance(experimentId, results);
    }

    private double calculatePValue(ExperimentVariant control, ExperimentVariant variant) {
        if (control.getImpressions() == 0 || variant.getImpressions() == 0) return 1.0;
        double p1 = (double) control.getConversions() / control.getImpressions();
        double p2 = (double) variant.getConversions() / variant.getImpressions();
        double pooledP = (double) (control.getConversions() + variant.getConversions()) /
                         (control.getImpressions() + variant.getImpressions());
        if (pooledP == 0 || pooledP == 1) return 1.0;
        double se = Math.sqrt(pooledP * (1 - pooledP) *
                    (1.0 / control.getImpressions() + 1.0 / variant.getImpressions()));
        double z = se > 0 ? Math.abs(p2 - p1) / se : 0;
        // Simplified p-value from z-score using normal distribution approximation
        return 2 * (1 - normalCdf(z));
    }

    private double normalCdf(double z) {
        return 0.5 * (1 + erf(z / Math.sqrt(2)));
    }

    private double erf(double x) {
        double t = 1.0 / (1.0 + 0.5 * Math.abs(x));
        double tau = t * Math.exp(-x * x - 1.26551223 + t * (1.00002368 + t * (0.37409196 +
            t * (0.09678418 + t * (-0.18628806 + t * (0.27886807 + t * (-1.13520398 +
            t * (1.48851587 + t * (-0.82215223 + t * 0.17087294)))))))));
        return x >= 0 ? 1 - tau : tau - 1;
    }

    private Long assignVariantDeterministically(Long experimentId, String hashKey,
                                                  int trafficPercentage) {
        int hash = Math.abs((experimentId + hashKey).hashCode()) % 100;
        if (hash >= trafficPercentage) return null;

        List<ExperimentVariant> variants = variantRepository.findByExperimentId(experimentId);
        double cumulative = 0;
        double normalizedHash = (double) hash / trafficPercentage * 100;
        for (ExperimentVariant v : variants) {
            cumulative += v.getTrafficSplit().doubleValue();
            if (normalizedHash < cumulative) return v.getId();
        }
        return variants.isEmpty() ? null : variants.get(variants.size() - 1).getId();
    }

    public record VariantResult(String name, long impressions, long conversions,
        double conversionRate, double improvement, double pValue, boolean isSignificant) {}
    public record StatisticalSignificance(Long experimentId, List<VariantResult> variants) {}
}
```

### Controller

```java
// ABTestingController.java
@RestController
@RequestMapping("/api/v1/experiments")
@RequiredArgsConstructor
public class ABTestingController {

    private final ABTestingService service;

    @PostMapping
    public ResponseEntity<Experiment> create(@RequestBody CreateExperimentRequest request) {
        return ResponseEntity.ok(service.createExperiment(request.createdBy(), request.name(),
            request.description(), request.hypothesis(), request.targetMetric(),
            request.trafficPercentage()));
    }

    @PostMapping("/{experimentId}/variants")
    public ResponseEntity<ExperimentVariant> addVariant(@PathVariable Long experimentId,
            @RequestBody AddVariantRequest request) {
        return ResponseEntity.ok(service.addVariant(experimentId, request.name(),
            request.description(), request.isControl(), request.trafficSplit(), request.config()));
    }

    @PostMapping("/{experimentId}/assign")
    public ResponseEntity<ExperimentAssignment> assign(@PathVariable Long experimentId,
            @RequestBody AssignRequest request) {
        return ResponseEntity.ok(service.assignVariant(experimentId,
            request.userId(), request.anonymousId()));
    }

    @PostMapping("/{experimentId}/convert")
    public ResponseEntity<Void> recordConversion(@PathVariable Long experimentId,
            @RequestBody ConversionRequest request) {
        service.recordConversion(experimentId, request.userId(), request.anonymousId(),
            request.eventName(), request.value());
        return ResponseEntity.ok().build();
    }

    @GetMapping("/{experimentId}/significance")
    public ResponseEntity<ABTestingService.StatisticalSignificance> getSignificance(
            @PathVariable Long experimentId) {
        return ResponseEntity.ok(service.calculateSignificance(experimentId));
    }

    record CreateExperimentRequest(Long createdBy, String name, String description,
        String hypothesis, String targetMetric, int trafficPercentage) {}
    record AddVariantRequest(String name, String description, boolean isControl,
        BigDecimal trafficSplit, Map<String, Object> config) {}
    record AssignRequest(Long userId, String anonymousId) {}
    record ConversionRequest(Long userId, String anonymousId, String eventName, BigDecimal value) {}
}
```

### docker-compose.yml

```yaml
# docker-compose.yml (A/B Testing Platform)
version: '3.8'
services:
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: abtest_db
      POSTGRES_USER: abtest_user
      POSTGRES_PASSWORD: abtest_pass
    ports:
      - "5432:5432"
    volumes:
      - abtest_pg_data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

  abtest-service:
    build: .
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/abtest_db
      SPRING_DATASOURCE_USERNAME: abtest_user
      SPRING_DATASOURCE_PASSWORD: abtest_pass
      SPRING_REDIS_HOST: redis
    ports:
      - "8080:8080"
    depends_on:
      - postgres
      - redis

volumes:
  abtest_pg_data:
```

---

## โปรเจค 73: Application Monitoring System

### ภาพรวมระบบ

Application Monitoring System ติดตามสถานะและประสิทธิภาพของ Services ทุกตัวในระบบ บันทึก Uptime ตรวจสอบ Health Checks จัดการ Incidents สร้าง Status Page API และส่งการแจ้งเตือนเมื่อเกิดปัญหา

### Flyway Migration

```sql
-- V1__create_monitoring_tables.sql
CREATE TABLE monitored_services (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    description TEXT,
    url VARCHAR(1000),
    check_type VARCHAR(30) NOT NULL DEFAULT 'HTTP',
    check_interval_seconds INT NOT NULL DEFAULT 60,
    timeout_seconds INT NOT NULL DEFAULT 30,
    expected_status_code INT DEFAULT 200,
    expected_response_content TEXT,
    team_id BIGINT,
    alert_threshold_minutes INT DEFAULT 5,
    status VARCHAR(20) NOT NULL DEFAULT 'UNKNOWN',
    last_check_at TIMESTAMP,
    last_success_at TIMESTAMP,
    uptime_percentage DECIMAL(7,4) DEFAULT 100.0000,
    active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE health_checks (
    id BIGSERIAL PRIMARY KEY,
    service_id BIGINT NOT NULL REFERENCES monitored_services(id),
    status VARCHAR(20) NOT NULL,
    response_time_ms INT,
    status_code INT,
    error_message TEXT,
    checked_at TIMESTAMP NOT NULL DEFAULT NOW()
) PARTITION BY RANGE (checked_at);

CREATE TABLE incidents (
    id BIGSERIAL PRIMARY KEY,
    service_id BIGINT NOT NULL REFERENCES monitored_services(id),
    title VARCHAR(300) NOT NULL,
    description TEXT,
    severity VARCHAR(20) NOT NULL DEFAULT 'MEDIUM',
    status VARCHAR(20) NOT NULL DEFAULT 'OPEN',
    started_at TIMESTAMP NOT NULL DEFAULT NOW(),
    resolved_at TIMESTAMP,
    root_cause TEXT,
    resolution_notes TEXT,
    downtime_minutes INT,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE sla_definitions (
    id BIGSERIAL PRIMARY KEY,
    service_id BIGINT NOT NULL UNIQUE REFERENCES monitored_services(id),
    uptime_target_percentage DECIMAL(7,4) NOT NULL DEFAULT 99.9,
    response_time_target_ms INT DEFAULT 2000,
    period_type VARCHAR(20) DEFAULT 'MONTHLY',
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE alert_channels (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    channel_type VARCHAR(20) NOT NULL,
    config JSONB NOT NULL,
    active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

-- Create partition for health checks
CREATE TABLE health_checks_2026_09 PARTITION OF health_checks
    FOR VALUES FROM ('2026-09-01') TO ('2026-10-01');
CREATE TABLE health_checks_2026_10 PARTITION OF health_checks
    FOR VALUES FROM ('2026-10-01') TO ('2026-11-01');

CREATE INDEX idx_health_checks_service_time ON health_checks(service_id, checked_at DESC);
CREATE INDEX idx_incidents_service_status ON incidents(service_id, status);
```

### Entity & Service

```java
// MonitoringService.java
package com.monitoring.service;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import org.springframework.web.client.RestTemplate;
import java.math.BigDecimal;
import java.math.RoundingMode;
import java.time.LocalDateTime;
import java.util.*;

@Service
@RequiredArgsConstructor
@Slf4j
public class MonitoringService {

    private final MonitoredServiceRepository serviceRepository;
    private final HealthCheckRepository healthCheckRepository;
    private final IncidentRepository incidentRepository;
    private final AlertChannelRepository alertChannelRepository;
    private final NotificationService notificationService;
    private final RestTemplate restTemplate;

    @Scheduled(fixedDelay = 30000) // Every 30 seconds
    @Transactional
    public void runHealthChecks() {
        List<MonitoredService> services = serviceRepository.findByActiveTrue();
        for (MonitoredService service : services) {
            try {
                performHealthCheck(service);
            } catch (Exception e) {
                log.error("Error checking service {}: {}", service.getName(), e.getMessage());
            }
        }
    }

    @Transactional
    public void performHealthCheck(MonitoredService service) {
        long startTime = System.currentTimeMillis();
        String newStatus;
        int responseCode = 0;
        String errorMessage = null;

        try {
            if ("HTTP".equals(service.getCheckType())) {
                org.springframework.http.ResponseEntity<String> response =
                    restTemplate.getForEntity(service.getUrl(), String.class);
                responseCode = response.getStatusCode().value();
                int expectedCode = service.getExpectedStatusCode() != null ?
                    service.getExpectedStatusCode() : 200;
                newStatus = responseCode == expectedCode ? "UP" : "DOWN";
            } else {
                newStatus = "UP"; // Simplified for non-HTTP checks
            }
        } catch (Exception e) {
            newStatus = "DOWN";
            errorMessage = e.getMessage();
        }

        int responseTime = (int) (System.currentTimeMillis() - startTime);
        HealthCheck check = HealthCheck.builder()
                .serviceId(service.getId()).status(newStatus)
                .responseTimeMs(responseTime).statusCode(responseCode > 0 ? responseCode : null)
                .errorMessage(errorMessage).build();
        healthCheckRepository.save(check);

        String previousStatus = service.getStatus();
        service.setStatus(newStatus);
        service.setLastCheckAt(LocalDateTime.now());
        if ("UP".equals(newStatus)) service.setLastSuccessAt(LocalDateTime.now());

        updateUptimePercentage(service);
        serviceRepository.save(service);

        // Handle status change
        if ("UP".equals(previousStatus) && "DOWN".equals(newStatus)) {
            createIncident(service);
        } else if ("DOWN".equals(previousStatus) && "UP".equals(newStatus)) {
            resolveIncident(service);
        }
    }

    private void createIncident(MonitoredService service) {
        Incident incident = Incident.builder()
                .serviceId(service.getId())
                .title(service.getName() + " is DOWN")
                .severity("HIGH")
                .status("OPEN").build();
        incident = incidentRepository.save(incident);
        notificationService.sendDowntimeAlert(service, incident);
    }

    private void resolveIncident(MonitoredService service) {
        List<Incident> openIncidents = incidentRepository
                .findByServiceIdAndStatus(service.getId(), "OPEN");
        for (Incident incident : openIncidents) {
            LocalDateTime now = LocalDateTime.now();
            long minutes = java.time.Duration.between(incident.getStartedAt(), now).toMinutes();
            incident.setStatus("RESOLVED");
            incident.setResolvedAt(now);
            incident.setDowntimeMinutes((int) minutes);
            incidentRepository.save(incident);
            notificationService.sendRecoveryAlert(service, incident);
        }
    }

    private void updateUptimePercentage(MonitoredService service) {
        LocalDateTime thirtyDaysAgo = LocalDateTime.now().minusDays(30);
        long totalChecks = healthCheckRepository.countByServiceIdAndCheckedAtAfter(
            service.getId(), thirtyDaysAgo);
        long upChecks = healthCheckRepository.countByServiceIdAndStatusAndCheckedAtAfter(
            service.getId(), "UP", thirtyDaysAgo);
        if (totalChecks > 0) {
            BigDecimal uptime = BigDecimal.valueOf(upChecks)
                    .divide(BigDecimal.valueOf(totalChecks), 6, RoundingMode.HALF_UP)
                    .multiply(BigDecimal.valueOf(100));
            service.setUptimePercentage(uptime);
        }
    }

    public StatusPageResponse getStatusPage() {
        List<MonitoredService> services = serviceRepository.findByActiveTrueOrderByName();
        List<Incident> activeIncidents = incidentRepository.findByStatus("OPEN");
        Map<String, String> serviceStatuses = new LinkedHashMap<>();
        for (MonitoredService s : services) {
            serviceStatuses.put(s.getName(), s.getStatus());
        }
        return new StatusPageResponse(serviceStatuses, activeIncidents.size(),
            activeIncidents.isEmpty() ? "All systems operational" : "Some systems degraded");
    }

    public SlaReport getSlaReport(Long serviceId, java.time.LocalDate month) {
        MonitoredService service = serviceRepository.findById(serviceId).orElseThrow();
        SlaDefinition sla = slaRepository.findByServiceId(serviceId).orElse(null);
        if (sla == null) throw new RuntimeException("No SLA defined for this service");

        LocalDateTime monthStart = month.withDayOfMonth(1).atStartOfDay();
        LocalDateTime monthEnd = month.withDayOfMonth(month.lengthOfMonth()).atTime(23, 59, 59);

        long totalChecks = healthCheckRepository.countByServiceIdAndCheckedAtBetween(
            serviceId, monthStart, monthEnd);
        long upChecks = healthCheckRepository.countByServiceIdAndStatusAndCheckedAtBetween(
            serviceId, "UP", monthStart, monthEnd);

        double actualUptime = totalChecks > 0 ? (double) upChecks / totalChecks * 100 : 0.0;
        boolean metSla = actualUptime >= sla.getUptimeTargetPercentage().doubleValue();
        double downtime = (100.0 - actualUptime) / 100.0 * month.lengthOfMonth() * 24 * 60;

        return new SlaReport(service.getName(), month, sla.getUptimeTargetPercentage().doubleValue(),
            actualUptime, metSla, (int) downtime);
    }

    private SlaDefinitionRepository slaRepository;

    public record StatusPageResponse(Map<String, String> services, int activeIncidents,
                                      String overallStatus) {}
    public record SlaReport(String serviceName, java.time.LocalDate month, double targetUptime,
        double actualUptime, boolean metSla, int downtimeMinutes) {}
}
```

### Controller & docker-compose.yml

```java
// MonitoringController.java
@RestController
@RequestMapping("/api/v1/monitoring")
@RequiredArgsConstructor
public class MonitoringController {

    private final MonitoringService monitoringService;

    @GetMapping("/status")
    public ResponseEntity<MonitoringService.StatusPageResponse> getStatusPage() {
        return ResponseEntity.ok(monitoringService.getStatusPage());
    }

    @PostMapping("/services/{serviceId}/check")
    public ResponseEntity<Void> triggerCheck(@PathVariable Long serviceId) {
        MonitoredService service = serviceRepository.findById(serviceId).orElseThrow();
        monitoringService.performHealthCheck(service);
        return ResponseEntity.ok().build();
    }

    @GetMapping("/services/{serviceId}/sla")
    public ResponseEntity<MonitoringService.SlaReport> getSlaReport(
            @PathVariable Long serviceId,
            @RequestParam(required = false) java.time.LocalDate month) {
        java.time.LocalDate reportMonth = month != null ? month :
            java.time.LocalDate.now().withDayOfMonth(1);
        return ResponseEntity.ok(monitoringService.getSlaReport(serviceId, reportMonth));
    }

    private MonitoredServiceRepository serviceRepository;
}
```

```yaml
# docker-compose.yml (Application Monitoring System)
version: '3.8'
services:
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: monitoring_db
      POSTGRES_USER: monitor_user
      POSTGRES_PASSWORD: monitor_pass
    ports:
      - "5432:5432"
    volumes:
      - monitoring_pg_data:/var/lib/postgresql/data

  monitoring-service:
    build: .
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/monitoring_db
      SPRING_DATASOURCE_USERNAME: monitor_user
      SPRING_DATASOURCE_PASSWORD: monitor_pass
      ALERT_SLACK_WEBHOOK: ${SLACK_WEBHOOK_URL}
      ALERT_EMAIL: ${ALERT_EMAIL}
    ports:
      - "8080:8080"
    depends_on:
      - postgres

volumes:
  monitoring_pg_data:
```

---

## โปรเจค 74: Log Management Service

### ภาพรวมระบบ

Log Management Service รวบรวม Log จาก Services ทั้งหมดในระบบ Parse Log แบบ Structured ค้นหา Full-text กำหนด Retention Policy และแจ้งเตือนเมื่อพบ Error Pattern ที่น่าสงสัย

### Flyway Migration

```sql
-- V1__create_log_management_tables.sql
CREATE TABLE log_sources (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    source_type VARCHAR(30) NOT NULL DEFAULT 'APPLICATION',
    environment VARCHAR(20) DEFAULT 'PRODUCTION',
    token VARCHAR(100) NOT NULL UNIQUE,
    retention_days INT NOT NULL DEFAULT 30,
    active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE log_entries (
    id BIGSERIAL PRIMARY KEY,
    source_id BIGINT NOT NULL REFERENCES log_sources(id),
    log_level VARCHAR(10) NOT NULL,
    message TEXT NOT NULL,
    service_name VARCHAR(200),
    hostname VARCHAR(200),
    trace_id VARCHAR(100),
    span_id VARCHAR(100),
    user_id BIGINT,
    request_id VARCHAR(100),
    fields JSONB,
    stack_trace TEXT,
    logged_at TIMESTAMP NOT NULL DEFAULT NOW()
) PARTITION BY RANGE (logged_at);

CREATE TABLE log_alerts (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    source_id BIGINT REFERENCES log_sources(id),
    pattern TEXT NOT NULL,
    log_level VARCHAR(10),
    threshold_count INT NOT NULL DEFAULT 1,
    window_minutes INT NOT NULL DEFAULT 5,
    notification_channel VARCHAR(200),
    active BOOLEAN DEFAULT TRUE,
    last_triggered_at TIMESTAMP,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE log_alert_occurrences (
    id BIGSERIAL PRIMARY KEY,
    alert_id BIGINT NOT NULL REFERENCES log_alerts(id),
    match_count INT NOT NULL,
    triggered_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE log_retention_jobs (
    id BIGSERIAL PRIMARY KEY,
    source_id BIGINT NOT NULL REFERENCES log_sources(id),
    deleted_count BIGINT NOT NULL DEFAULT 0,
    ran_at TIMESTAMP NOT NULL DEFAULT NOW()
);

-- Partitions
CREATE TABLE log_entries_2026_09 PARTITION OF log_entries
    FOR VALUES FROM ('2026-09-01') TO ('2026-10-01');
CREATE TABLE log_entries_2026_10 PARTITION OF log_entries
    FOR VALUES FROM ('2026-10-01') TO ('2026-11-01');

CREATE INDEX idx_log_entries_source_level_time ON log_entries(source_id, log_level, logged_at DESC);
CREATE INDEX idx_log_entries_trace ON log_entries(trace_id) WHERE trace_id IS NOT NULL;
CREATE INDEX idx_log_entries_message_fts ON log_entries USING gin(to_tsvector('english', message));
```

### Entity & Service

```java
// LogManagementService.java
package com.logmanagement.service;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import java.time.LocalDateTime;
import java.util.*;

@Service
@RequiredArgsConstructor
@Slf4j
public class LogManagementService {

    private final LogSourceRepository sourceRepository;
    private final LogEntryRepository entryRepository;
    private final LogAlertRepository alertRepository;
    private final LogAlertOccurrenceRepository occurrenceRepository;
    private final NotificationService notificationService;

    @Transactional
    public List<LogEntry> ingestBatch(String sourceToken, List<LogIngestionRequest> logRequests) {
        LogSource source = sourceRepository.findByToken(sourceToken)
                .orElseThrow(() -> new RuntimeException("Invalid source token"));
        if (!source.getActive()) {
            throw new IllegalStateException("Log source is disabled");
        }

        List<LogEntry> entries = new ArrayList<>();
        for (LogIngestionRequest req : logRequests) {
            LogEntry entry = LogEntry.builder()
                    .sourceId(source.getId())
                    .logLevel(req.level() != null ? req.level().toUpperCase() : "INFO")
                    .message(req.message()).serviceName(req.serviceName())
                    .hostname(req.hostname()).traceId(req.traceId()).spanId(req.spanId())
                    .userId(req.userId()).requestId(req.requestId())
                    .fields(req.fields()).stackTrace(req.stackTrace())
                    .loggedAt(req.loggedAt() != null ? req.loggedAt() : LocalDateTime.now())
                    .build();
            entries.add(entry);
        }
        entries = entryRepository.saveAll(entries);
        checkAlerts(source.getId(), entries);
        return entries;
    }

    public LogSearchResult search(Long sourceId, String query, String level,
                                   LocalDateTime from, LocalDateTime to,
                                   String traceId, int page, int size) {
        List<LogEntry> entries;
        if (query != null && !query.isEmpty()) {
            entries = entryRepository.fullTextSearch(sourceId, query, level, from, to,
                traceId, page, size);
        } else {
            entries = entryRepository.findByFilters(sourceId, level, from, to, traceId, page, size);
        }
        long total = entryRepository.countByFilters(sourceId, query, level, from, to, traceId);
        return new LogSearchResult(entries, total, page, size);
    }

    private void checkAlerts(Long sourceId, List<LogEntry> newEntries) {
        List<LogAlert> activeAlerts = alertRepository
                .findBySourceIdOrSourceIdIsNullAndActiveTrue(sourceId);

        for (LogAlert alert : activeAlerts) {
            LocalDateTime windowStart = LocalDateTime.now().minusMinutes(alert.getWindowMinutes());
            long matchCount = newEntries.stream()
                    .filter(e -> alert.getLogLevel() == null ||
                                 alert.getLogLevel().equalsIgnoreCase(e.getLogLevel()))
                    .filter(e -> e.getMessage().toLowerCase()
                                  .contains(alert.getPattern().toLowerCase()))
                    .count();

            if (matchCount == 0) continue;

            long totalMatches = entryRepository.countBySourceIdAndMessageContainingAndLoggedAtAfter(
                sourceId, alert.getPattern(), windowStart);
            if (alert.getLogLevel() != null) {
                totalMatches = Math.min(totalMatches,
                    entryRepository.countBySourceIdAndLogLevelAndLoggedAtAfter(
                        sourceId, alert.getLogLevel(), windowStart));
            }

            if (totalMatches >= alert.getThresholdCount()) {
                LogAlertOccurrence occurrence = LogAlertOccurrence.builder()
                        .alertId(alert.getId()).matchCount((int) totalMatches).build();
                occurrenceRepository.save(occurrence);
                alert.setLastTriggeredAt(LocalDateTime.now());
                alertRepository.save(alert);
                notificationService.sendLogAlert(alert, (int) totalMatches);
            }
        }
    }

    @Scheduled(cron = "0 0 2 * * *")
    @Transactional
    public void enforceRetentionPolicies() {
        List<LogSource> sources = sourceRepository.findByActiveTrue();
        for (LogSource source : sources) {
            LocalDateTime cutoff = LocalDateTime.now().minusDays(source.getRetentionDays());
            long deleted = entryRepository.deleteBySourceIdAndLoggedAtBefore(
                source.getId(), cutoff);
            if (deleted > 0) {
                LogRetentionJob job = LogRetentionJob.builder()
                        .sourceId(source.getId()).deletedCount(deleted).build();
                log.info("Retention: deleted {} logs for source {}", deleted, source.getName());
            }
        }
    }

    public LogStats getStats(Long sourceId, LocalDateTime from, LocalDateTime to) {
        Map<String, Long> byLevel = new LinkedHashMap<>();
        for (String level : List.of("ERROR", "WARN", "INFO", "DEBUG")) {
            byLevel.put(level, entryRepository.countBySourceIdAndLogLevelAndLoggedAtBetween(
                sourceId, level, from, to));
        }
        long total = byLevel.values().stream().mapToLong(Long::longValue).sum();
        return new LogStats(total, byLevel);
    }

    public record LogIngestionRequest(String level, String message, String serviceName,
        String hostname, String traceId, String spanId, Long userId, String requestId,
        Map<String, Object> fields, String stackTrace, LocalDateTime loggedAt) {}
    public record LogSearchResult(List<LogEntry> entries, long total, int page, int size) {}
    public record LogStats(long totalLogs, Map<String, Long> byLevel) {}
}
```

### Controller & docker-compose.yml

```java
// LogManagementController.java
@RestController
@RequestMapping("/api/v1/logs")
@RequiredArgsConstructor
public class LogManagementController {

    private final LogManagementService service;

    @PostMapping("/ingest")
    public ResponseEntity<Void> ingest(@RequestHeader("X-Source-Token") String token,
            @RequestBody List<LogManagementService.LogIngestionRequest> logs) {
        service.ingestBatch(token, logs);
        return ResponseEntity.ok().build();
    }

    @GetMapping("/search")
    public ResponseEntity<LogManagementService.LogSearchResult> search(
            @RequestParam Long sourceId,
            @RequestParam(required = false) String query,
            @RequestParam(required = false) String level,
            @RequestParam(required = false) LocalDateTime from,
            @RequestParam(required = false) LocalDateTime to,
            @RequestParam(required = false) String traceId,
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "100") int size) {
        return ResponseEntity.ok(service.search(sourceId, query, level, from, to, traceId,
            page, size));
    }

    @GetMapping("/stats/{sourceId}")
    public ResponseEntity<LogManagementService.LogStats> getStats(
            @PathVariable Long sourceId,
            @RequestParam(required = false) LocalDateTime from,
            @RequestParam(required = false) LocalDateTime to) {
        LocalDateTime endDt = to != null ? to : LocalDateTime.now();
        LocalDateTime startDt = from != null ? from : endDt.minusHours(24);
        return ResponseEntity.ok(service.getStats(sourceId, startDt, endDt));
    }
}
```

```yaml
# docker-compose.yml (Log Management Service)
version: '3.8'
services:
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: logmanagement_db
      POSTGRES_USER: log_user
      POSTGRES_PASSWORD: log_pass
    ports:
      - "5432:5432"
    volumes:
      - log_pg_data:/var/lib/postgresql/data

  elasticsearch:
    image: elasticsearch:8.11.0
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    ports:
      - "9200:9200"
    volumes:
      - log_es_data:/usr/share/elasticsearch/data

  logmanagement-service:
    build: .
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/logmanagement_db
      SPRING_DATASOURCE_USERNAME: log_user
      SPRING_DATASOURCE_PASSWORD: log_pass
      SPRING_ELASTICSEARCH_URIS: http://elasticsearch:9200
    ports:
      - "8080:8080"
    depends_on:
      - postgres
      - elasticsearch

volumes:
  log_pg_data:
  log_es_data:
```

---

## โปรเจค 75: Data Pipeline Service

### ภาพรวมระบบ

Data Pipeline Service ช่วยให้ทีม Data Engineer สร้างและจัดการ Pipeline สำหรับ Data Transformation อัตโนมัติ รองรับการกำหนด Steps, Schedule, Error Handling, Retry และ Data Quality Checks เพื่อให้มั่นใจว่าข้อมูลที่ผ่าน Pipeline มีคุณภาพ

- **Pipeline Definitions**: กำหนด Pipeline และ Steps
- **Scheduling**: รันตาม Schedule อัตโนมัติ
- **Run History**: ประวัติการรันพร้อมสถิติ
- **Data Transformation**: ประมวลผลข้อมูล
- **Error Handling & Retry**: จัดการ Error อัตโนมัติ
- **Data Quality Checks**: ตรวจสอบคุณภาพข้อมูล

### Flyway Migration

```sql
-- V1__create_data_pipeline_tables.sql
CREATE TABLE pipelines (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(200) NOT NULL UNIQUE,
    description TEXT,
    status VARCHAR(20) NOT NULL DEFAULT 'ACTIVE',
    schedule_cron VARCHAR(100),
    timeout_minutes INT DEFAULT 60,
    max_retries INT DEFAULT 3,
    retry_delay_seconds INT DEFAULT 300,
    notification_on_failure TEXT,
    created_by BIGINT,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE pipeline_steps (
    id BIGSERIAL PRIMARY KEY,
    pipeline_id BIGINT NOT NULL REFERENCES pipelines(id) ON DELETE CASCADE,
    name VARCHAR(200) NOT NULL,
    step_type VARCHAR(50) NOT NULL,
    config JSONB NOT NULL,
    step_order INT NOT NULL,
    depends_on_step_id BIGINT REFERENCES pipeline_steps(id),
    retry_count INT DEFAULT 0,
    max_retries INT DEFAULT 3,
    timeout_seconds INT DEFAULT 300,
    active BOOLEAN DEFAULT TRUE
);

CREATE TABLE pipeline_runs (
    id BIGSERIAL PRIMARY KEY,
    pipeline_id BIGINT NOT NULL REFERENCES pipelines(id),
    run_number BIGINT NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'PENDING',
    triggered_by VARCHAR(50) DEFAULT 'SCHEDULE',
    started_at TIMESTAMP,
    completed_at TIMESTAMP,
    duration_seconds INT,
    records_processed BIGINT DEFAULT 0,
    records_failed BIGINT DEFAULT 0,
    error_message TEXT,
    retry_attempt INT DEFAULT 0,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE step_runs (
    id BIGSERIAL PRIMARY KEY,
    pipeline_run_id BIGINT NOT NULL REFERENCES pipeline_runs(id),
    step_id BIGINT NOT NULL REFERENCES pipeline_steps(id),
    status VARCHAR(20) NOT NULL DEFAULT 'PENDING',
    started_at TIMESTAMP,
    completed_at TIMESTAMP,
    duration_seconds INT,
    records_in BIGINT DEFAULT 0,
    records_out BIGINT DEFAULT 0,
    records_failed BIGINT DEFAULT 0,
    error_message TEXT,
    output_metadata JSONB
);

CREATE TABLE data_quality_checks (
    id BIGSERIAL PRIMARY KEY,
    pipeline_id BIGINT NOT NULL REFERENCES pipelines(id),
    check_name VARCHAR(200) NOT NULL,
    check_type VARCHAR(50) NOT NULL,
    config JSONB NOT NULL,
    severity VARCHAR(20) NOT NULL DEFAULT 'ERROR',
    active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE quality_check_results (
    id BIGSERIAL PRIMARY KEY,
    pipeline_run_id BIGINT NOT NULL REFERENCES pipeline_runs(id),
    check_id BIGINT NOT NULL REFERENCES data_quality_checks(id),
    status VARCHAR(20) NOT NULL,
    actual_value DECIMAL(20,4),
    expected_value DECIMAL(20,4),
    message TEXT,
    checked_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_pipeline_runs_pipeline_status ON pipeline_runs(pipeline_id, status);
CREATE INDEX idx_step_runs_run ON step_runs(pipeline_run_id);
```

### Entity & Service

```java
// DataPipelineService.java
package com.pipeline.service;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import java.time.LocalDateTime;
import java.util.*;

@Service
@RequiredArgsConstructor
@Slf4j
public class DataPipelineService {

    private final PipelineRepository pipelineRepository;
    private final PipelineStepRepository stepRepository;
    private final PipelineRunRepository runRepository;
    private final StepRunRepository stepRunRepository;
    private final DataQualityCheckRepository qualityCheckRepository;
    private final QualityCheckResultRepository qualityResultRepository;
    private final StepExecutorFactory executorFactory;
    private final NotificationService notificationService;

    @Transactional
    public Pipeline createPipeline(String name, String description, String scheduleCron,
                                    int timeoutMinutes, int maxRetries, Long createdBy) {
        Pipeline pipeline = Pipeline.builder()
                .name(name).description(description).scheduleCron(scheduleCron)
                .timeoutMinutes(timeoutMinutes).maxRetries(maxRetries)
                .status("ACTIVE").createdBy(createdBy).build();
        return pipelineRepository.save(pipeline);
    }

    @Transactional
    public PipelineStep addStep(Long pipelineId, String name, String stepType,
                                 Map<String, Object> config, int stepOrder, Long dependsOn) {
        PipelineStep step = PipelineStep.builder()
                .pipelineId(pipelineId).name(name).stepType(stepType)
                .config(config).stepOrder(stepOrder).dependsOnStepId(dependsOn).build();
        return stepRepository.save(step);
    }

    @Transactional
    public PipelineRun triggerRun(Long pipelineId, String triggeredBy) {
        Pipeline pipeline = pipelineRepository.findById(pipelineId)
                .orElseThrow(() -> new RuntimeException("Pipeline not found"));
        if (!"ACTIVE".equals(pipeline.getStatus())) {
            throw new IllegalStateException("Pipeline is not active");
        }

        long runNumber = runRepository.countByPipelineId(pipelineId) + 1;
        PipelineRun run = PipelineRun.builder()
                .pipelineId(pipelineId).runNumber(runNumber)
                .status("RUNNING").triggeredBy(triggeredBy)
                .startedAt(LocalDateTime.now()).build();
        run = runRepository.save(run);

        executeRun(run, pipeline);
        return run;
    }

    private void executeRun(PipelineRun run, Pipeline pipeline) {
        List<PipelineStep> steps = stepRepository
                .findByPipelineIdAndActiveTrueOrderByStepOrder(pipeline.getId());

        long totalProcessed = 0, totalFailed = 0;
        boolean pipelineFailed = false;

        for (PipelineStep step : steps) {
            StepRun stepRun = StepRun.builder()
                    .pipelineRunId(run.getId()).stepId(step.getId())
                    .status("RUNNING").startedAt(LocalDateTime.now()).build();
            stepRun = stepRunRepository.save(stepRun);

            try {
                StepExecutionResult result = executorFactory.getExecutor(step.getStepType())
                        .execute(step, run);
                stepRun.setStatus("COMPLETED");
                stepRun.setRecordsIn(result.recordsIn());
                stepRun.setRecordsOut(result.recordsOut());
                stepRun.setRecordsFailed(result.recordsFailed());
                stepRun.setOutputMetadata(result.metadata());
                totalProcessed += result.recordsOut();
                totalFailed += result.recordsFailed();
            } catch (Exception e) {
                stepRun.setStatus("FAILED");
                stepRun.setErrorMessage(e.getMessage());
                pipelineFailed = true;
                log.error("Step {} failed: {}", step.getName(), e.getMessage());
            } finally {
                LocalDateTime now = LocalDateTime.now();
                stepRun.setCompletedAt(now);
                if (stepRun.getStartedAt() != null) {
                    stepRun.setDurationSeconds((int) java.time.Duration.between(
                        stepRun.getStartedAt(), now).getSeconds());
                }
                stepRunRepository.save(stepRun);
            }
            if (pipelineFailed) break;
        }

        // Run quality checks
        if (!pipelineFailed) {
            runQualityChecks(run);
        }

        LocalDateTime now = LocalDateTime.now();
        run.setStatus(pipelineFailed ? "FAILED" : "COMPLETED");
        run.setCompletedAt(now);
        run.setRecordsProcessed(totalProcessed);
        run.setRecordsFailed(totalFailed);
        if (run.getStartedAt() != null) {
            run.setDurationSeconds((int) java.time.Duration.between(run.getStartedAt(), now).getSeconds());
        }
        runRepository.save(run);

        if (pipelineFailed) {
            handleFailure(run, pipeline);
        }
    }

    private void runQualityChecks(PipelineRun run) {
        List<DataQualityCheck> checks = qualityCheckRepository
                .findByPipelineIdAndActiveTrue(run.getPipelineId());
        for (DataQualityCheck check : checks) {
            try {
                QualityCheckResult result = performQualityCheck(check, run);
                qualityResultRepository.save(result);
                if ("FAILED".equals(result.getStatus()) && "ERROR".equals(check.getSeverity())) {
                    run.setStatus("QUALITY_FAILED");
                    runRepository.save(run);
                    break;
                }
            } catch (Exception e) {
                log.error("Quality check {} failed: {}", check.getCheckName(), e.getMessage());
            }
        }
    }

    private QualityCheckResult performQualityCheck(DataQualityCheck check, PipelineRun run) {
        // Simplified quality check execution
        QualityCheckResult result = QualityCheckResult.builder()
                .pipelineRunId(run.getId()).checkId(check.getId())
                .status("PASSED").message("Check passed").build();
        return result;
    }

    private void handleFailure(PipelineRun run, Pipeline pipeline) {
        if (run.getRetryAttempt() < pipeline.getMaxRetries()) {
            run.setRetryAttempt(run.getRetryAttempt() + 1);
            runRepository.save(run);
            // Schedule retry (simplified)
            log.info("Scheduling retry {} for pipeline {}", run.getRetryAttempt(), pipeline.getName());
        } else {
            notificationService.sendPipelineFailureAlert(pipeline, run);
        }
    }

    public PipelineStats getPipelineStats(Long pipelineId) {
        List<PipelineRun> recentRuns = runRepository
                .findByPipelineIdOrderByCreatedAtDesc(pipelineId, 20);
        long successful = recentRuns.stream().filter(r -> "COMPLETED".equals(r.getStatus())).count();
        double successRate = recentRuns.isEmpty() ? 0.0 :
            (double) successful / recentRuns.size() * 100;
        OptionalDouble avgDuration = recentRuns.stream()
                .filter(r -> r.getDurationSeconds() != null)
                .mapToInt(PipelineRun::getDurationSeconds).average();
        return new PipelineStats(recentRuns.size(), (int) successful, successRate,
            avgDuration.orElse(0.0));
    }

    public record StepExecutionResult(long recordsIn, long recordsOut, long recordsFailed,
                                       Map<String, Object> metadata) {}
    public record PipelineStats(int totalRuns, int successfulRuns, double successRate,
                                 double avgDurationSeconds) {}
}
```

### Controller

```java
// DataPipelineController.java
@RestController
@RequestMapping("/api/v1/pipelines")
@RequiredArgsConstructor
public class DataPipelineController {

    private final DataPipelineService service;

    @PostMapping
    public ResponseEntity<Pipeline> createPipeline(@RequestBody CreatePipelineRequest request) {
        return ResponseEntity.ok(service.createPipeline(request.name(), request.description(),
            request.scheduleCron(), request.timeoutMinutes(), request.maxRetries(),
            request.createdBy()));
    }

    @PostMapping("/{pipelineId}/steps")
    public ResponseEntity<PipelineStep> addStep(@PathVariable Long pipelineId,
            @RequestBody AddStepRequest request) {
        return ResponseEntity.ok(service.addStep(pipelineId, request.name(), request.stepType(),
            request.config(), request.stepOrder(), request.dependsOnStepId()));
    }

    @PostMapping("/{pipelineId}/run")
    public ResponseEntity<PipelineRun> triggerRun(@PathVariable Long pipelineId,
            @RequestParam(defaultValue = "MANUAL") String triggeredBy) {
        return ResponseEntity.ok(service.triggerRun(pipelineId, triggeredBy));
    }

    @GetMapping("/{pipelineId}/stats")
    public ResponseEntity<DataPipelineService.PipelineStats> getStats(
            @PathVariable Long pipelineId) {
        return ResponseEntity.ok(service.getPipelineStats(pipelineId));
    }

    @GetMapping("/{pipelineId}/runs")
    public ResponseEntity<List<PipelineRun>> getRuns(@PathVariable Long pipelineId) {
        return ResponseEntity.ok(service.getRecentRuns(pipelineId));
    }

    record CreatePipelineRequest(String name, String description, String scheduleCron,
        int timeoutMinutes, int maxRetries, Long createdBy) {}
    record AddStepRequest(String name, String stepType, Map<String, Object> config,
        int stepOrder, Long dependsOnStepId) {}
}
```

### docker-compose.yml

```yaml
# docker-compose.yml (Data Pipeline Service)
version: '3.8'
services:
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: pipeline_db
      POSTGRES_USER: pipeline_user
      POSTGRES_PASSWORD: pipeline_pass
    ports:
      - "5432:5432"
    volumes:
      - pipeline_pg_data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

  kafka:
    image: confluentinc/cp-kafka:7.5.0
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
    ports:
      - "9092:9092"
    depends_on:
      - zookeeper

  zookeeper:
    image: confluentinc/cp-zookeeper:7.5.0
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181

  pipeline-service:
    build: .
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/pipeline_db
      SPRING_DATASOURCE_USERNAME: pipeline_user
      SPRING_DATASOURCE_PASSWORD: pipeline_pass
      SPRING_REDIS_HOST: redis
      SPRING_KAFKA_BOOTSTRAP_SERVERS: kafka:9092
    ports:
      - "8080:8080"
    depends_on:
      - postgres
      - redis
      - kafka

volumes:
  pipeline_pg_data:
```

---

*[← Part 114: Education & Learning](./part-114-education-learning.md) | [Part 116: Advanced APIs →](./part-116-advanced-apis.md)*

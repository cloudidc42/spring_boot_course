# Part 96: Capstone Project Design - World-Class E-Commerce Platform
## ขั้นตอนที่ 3441-3480

**ระดับ: World-Class Professional**

---

## บทนำ: Capstone Project คืออะไร?

Capstone Project คือโปรเจกต์สุดท้ายที่รวบรวมทุกสิ่งที่เรียนมาตลอดหลักสูตร เราจะสร้าง **E-Commerce Platform ระดับ World-Class** ที่สามารถรองรับการใช้งานจริงในระดับ Enterprise ได้

โปรเจกต์นี้จะครอบคลุม:
- Microservices Architecture ที่ซับซ้อน
- Event-Driven System
- Real-time Processing
- High Availability และ Fault Tolerance
- Performance Optimization
- Security Best Practices

---

## ขั้นตอนที่ 3441: System Architecture Overview

### ภาพรวมสถาปัตยกรรมระบบ

ระบบ E-Commerce Platform ของเราจะประกอบด้วย Microservices หลายตัวที่ทำงานร่วมกัน แต่ละ Service มีหน้าที่เฉพาะและสื่อสารกันผ่าน Event Bus

```
┌─────────────────────────────────────────────────────────────────┐
│                        API Gateway Layer                         │
│              (Kong / Spring Cloud Gateway)                       │
└─────────────────────┬───────────────────────────────────────────┘
                       │
        ┌──────────────┼──────────────┐
        ▼              ▼              ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│   User       │ │  Product     │ │   Order      │
│   Service    │ │  Service     │ │   Service    │
│  :8081       │ │  :8082       │ │  :8083       │
└──────┬───────┘ └──────┬───────┘ └──────┬───────┘
       │                │                │
       ▼                ▼                ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│  PostgreSQL  │ │ Elasticsearch│ │  PostgreSQL  │
│  (Users DB) │ │ + PostgreSQL │ │  (Orders DB) │
└──────────────┘ └──────────────┘ └──────────────┘
       │                │                │
       └────────────────┼────────────────┘
                        ▼
              ┌──────────────────┐
              │   Apache Kafka   │
              │  (Event Bus)     │
              └─────────┬────────┘
                        │
        ┌───────────────┼───────────────┐
        ▼               ▼               ▼
┌─────────────┐ ┌─────────────┐ ┌─────────────┐
│  Inventory  │ │  Payment    │ │Notification │
│  Service   │ │  Service    │ │  Service    │
│  :8084      │ │  :8085      │ │  :8086      │
└─────────────┘ └─────────────┘ └─────────────┘
```

### Service Breakdown

| Service | หน้าที่ | Technology |
|---------|---------|-----------|
| User Service | จัดการ User, Auth, Profile | Spring Boot + PostgreSQL + Redis |
| Product Service | จัดการ Product Catalog | Spring Boot + Elasticsearch + PostgreSQL |
| Order Service | จัดการ Order Lifecycle | Spring Boot + PostgreSQL + Kafka |
| Inventory Service | จัดการ Stock Level | Spring Boot + PostgreSQL + Redis |
| Payment Service | จัดการการชำระเงิน | Spring Boot + PostgreSQL + Stripe |
| Notification Service | ส่ง Email/SMS/Push | Spring Boot + Kafka + SendGrid |
| Search Service | Full-text Search | Spring Boot + Elasticsearch |
| Recommendation Service | แนะนำสินค้า | Spring Boot + Redis + ML Model |

---

## ขั้นตอนที่ 3442: Technology Stack Decisions

### เหตุผลในการเลือก Technology Stack

**Backend Framework: Spring Boot 3.2**
- Ecosystem ที่สมบูรณ์และมี Community ใหญ่
- Native Support สำหรับ GraalVM Native Image
- Virtual Threads (Project Loom) ช่วยเพิ่ม Throughput
- Spring Security, Data, Cloud ทำงานร่วมกันได้ดี

**Database Strategy: Polyglot Persistence**
```yaml
# แต่ละ Service เลือก Database ที่เหมาะสมที่สุด
databases:
  user-service:
    primary: PostgreSQL 16  # ACID compliance สำหรับ User Data
    cache: Redis 7          # Session และ Token caching
    
  product-service:
    primary: PostgreSQL 16  # Product metadata
    search: Elasticsearch 8 # Full-text search และ facets
    
  order-service:
    primary: PostgreSQL 16  # ACID transactions
    events: Apache Kafka    # Event sourcing
    
  inventory-service:
    primary: PostgreSQL 16  # Consistent stock levels
    cache: Redis 7          # Real-time stock updates
```

**Message Broker: Apache Kafka**
- High Throughput สำหรับ Event Streaming
- Durable Message Storage
- Replay capability สำหรับ Event Sourcing
- Topic Partitioning สำหรับ Horizontal Scaling

**Service Mesh: Istio**
- Automatic mTLS ระหว่าง Services
- Traffic Management และ Load Balancing
- Observability (Metrics, Tracing, Logging)
- Circuit Breaking

---

## ขั้นตอนที่ 3443: Parent POM Structure

```xml
<!-- pom.xml (Root/Parent) -->
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 
         http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.ecommerce.platform</groupId>
    <artifactId>ecommerce-platform</artifactId>
    <version>1.0.0-SNAPSHOT</version>
    <packaging>pom</packaging>

    <name>E-Commerce Platform - Capstone Project</name>
    <description>World-Class E-Commerce Platform built with Spring Boot</description>

    <modules>
        <module>common-lib</module>
        <module>api-gateway</module>
        <module>user-service</module>
        <module>product-service</module>
        <module>order-service</module>
        <module>inventory-service</module>
        <module>payment-service</module>
        <module>notification-service</module>
        <module>search-service</module>
        <module>recommendation-service</module>
    </modules>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.2.0</version>
        <relativePath/>
    </parent>

    <properties>
        <java.version>21</java.version>
        <spring-cloud.version>2023.0.0</spring-cloud.version>
        <testcontainers.version>1.19.3</testcontainers.version>
        <mapstruct.version>1.5.5.Final</mapstruct.version>
        <openapi.version>2.3.0</openapi.version>
    </properties>

    <dependencyManagement>
        <dependencies>
            <!-- Spring Cloud BOM -->
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>${spring-cloud.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>

            <!-- Testcontainers BOM -->
            <dependency>
                <groupId>org.testcontainers</groupId>
                <artifactId>testcontainers-bom</artifactId>
                <version>${testcontainers.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>

            <!-- Common Library -->
            <dependency>
                <groupId>com.ecommerce.platform</groupId>
                <artifactId>common-lib</artifactId>
                <version>${project.version}</version>
            </dependency>
        </dependencies>
    </dependencyManagement>

    <dependencies>
        <!-- ทุก Module มี dependencies เหล่านี้ -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
        <dependency>
            <groupId>io.micrometer</groupId>
            <artifactId>micrometer-tracing-bridge-otel</artifactId>
        </dependency>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>
</project>
```

---

## ขั้นตอนที่ 3444: Data Model Design (ERD)

### User Domain

```sql
-- Users Table
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    username VARCHAR(100) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    first_name VARCHAR(100),
    last_name VARCHAR(100),
    phone VARCHAR(20),
    status VARCHAR(20) DEFAULT 'ACTIVE' CHECK (status IN ('ACTIVE', 'INACTIVE', 'SUSPENDED')),
    email_verified BOOLEAN DEFAULT false,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    last_login_at TIMESTAMP WITH TIME ZONE,
    version BIGINT DEFAULT 0  -- สำหรับ Optimistic Locking
);

-- User Addresses Table
CREATE TABLE user_addresses (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    label VARCHAR(50),  -- 'HOME', 'WORK', 'OTHER'
    recipient_name VARCHAR(200) NOT NULL,
    address_line1 VARCHAR(255) NOT NULL,
    address_line2 VARCHAR(255),
    city VARCHAR(100) NOT NULL,
    state VARCHAR(100),
    postal_code VARCHAR(20) NOT NULL,
    country_code CHAR(2) NOT NULL,
    phone VARCHAR(20),
    is_default BOOLEAN DEFAULT false,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Indexes
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_username ON users(username);
CREATE INDEX idx_users_status ON users(status);
CREATE INDEX idx_user_addresses_user_id ON user_addresses(user_id);
```

### Product Domain

```sql
-- Product Categories
CREATE TABLE categories (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    parent_id UUID REFERENCES categories(id),
    name VARCHAR(200) NOT NULL,
    slug VARCHAR(200) UNIQUE NOT NULL,
    description TEXT,
    image_url VARCHAR(500),
    sort_order INT DEFAULT 0,
    is_active BOOLEAN DEFAULT true,
    path VARCHAR(1000),  -- Materialized path: /electronics/phones/
    depth INT DEFAULT 0,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Products Table
CREATE TABLE products (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    category_id UUID NOT NULL REFERENCES categories(id),
    name VARCHAR(500) NOT NULL,
    slug VARCHAR(500) UNIQUE NOT NULL,
    description TEXT,
    short_description VARCHAR(500),
    sku VARCHAR(100) UNIQUE,
    brand VARCHAR(200),
    
    -- Pricing
    base_price DECIMAL(12, 2) NOT NULL,
    sale_price DECIMAL(12, 2),
    currency CHAR(3) DEFAULT 'THB',
    
    -- Status
    status VARCHAR(20) DEFAULT 'DRAFT' CHECK (
        status IN ('DRAFT', 'ACTIVE', 'INACTIVE', 'DISCONTINUED')
    ),
    
    -- SEO
    meta_title VARCHAR(200),
    meta_description VARCHAR(500),
    
    -- Metadata
    attributes JSONB DEFAULT '{}',
    tags TEXT[],
    
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    published_at TIMESTAMP WITH TIME ZONE,
    version BIGINT DEFAULT 0
);

-- Product Variants (Size, Color, etc.)
CREATE TABLE product_variants (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_id UUID NOT NULL REFERENCES products(id) ON DELETE CASCADE,
    sku VARCHAR(100) UNIQUE NOT NULL,
    name VARCHAR(200),
    attributes JSONB NOT NULL DEFAULT '{}',  -- {"color": "Red", "size": "XL"}
    price_adjustment DECIMAL(12, 2) DEFAULT 0,
    weight_grams INT,
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Product Images
CREATE TABLE product_images (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_id UUID NOT NULL REFERENCES products(id) ON DELETE CASCADE,
    variant_id UUID REFERENCES product_variants(id),
    url VARCHAR(500) NOT NULL,
    alt_text VARCHAR(200),
    sort_order INT DEFAULT 0,
    is_primary BOOLEAN DEFAULT false
);

-- Indexes
CREATE INDEX idx_products_category ON products(category_id);
CREATE INDEX idx_products_status ON products(status);
CREATE INDEX idx_products_slug ON products(slug);
CREATE INDEX idx_products_tags ON products USING gin(tags);
CREATE INDEX idx_products_attributes ON products USING gin(attributes);
```

### Order Domain

```sql
-- Orders Table
CREATE TABLE orders (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_number VARCHAR(50) UNIQUE NOT NULL,
    user_id UUID NOT NULL,
    
    -- Status
    status VARCHAR(30) DEFAULT 'PENDING' CHECK (
        status IN ('PENDING', 'CONFIRMED', 'PROCESSING', 'SHIPPED', 
                   'DELIVERED', 'CANCELLED', 'REFUNDED')
    ),
    
    -- Pricing
    subtotal DECIMAL(12, 2) NOT NULL,
    discount_amount DECIMAL(12, 2) DEFAULT 0,
    shipping_cost DECIMAL(12, 2) DEFAULT 0,
    tax_amount DECIMAL(12, 2) DEFAULT 0,
    total_amount DECIMAL(12, 2) NOT NULL,
    currency CHAR(3) DEFAULT 'THB',
    
    -- Addresses (Snapshot at order time)
    shipping_address JSONB NOT NULL,
    billing_address JSONB,
    
    -- Shipping
    shipping_method VARCHAR(100),
    tracking_number VARCHAR(200),
    estimated_delivery DATE,
    
    -- Notes
    customer_notes TEXT,
    internal_notes TEXT,
    
    -- Idempotency
    idempotency_key VARCHAR(100) UNIQUE,
    
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    confirmed_at TIMESTAMP WITH TIME ZONE,
    shipped_at TIMESTAMP WITH TIME ZONE,
    delivered_at TIMESTAMP WITH TIME ZONE,
    cancelled_at TIMESTAMP WITH TIME ZONE,
    
    version BIGINT DEFAULT 0
);

-- Order Items
CREATE TABLE order_items (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id UUID NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
    product_id UUID NOT NULL,
    variant_id UUID,
    
    -- Snapshot ณ เวลาที่สั่ง
    product_name VARCHAR(500) NOT NULL,
    product_sku VARCHAR(100),
    variant_attributes JSONB,
    
    quantity INT NOT NULL CHECK (quantity > 0),
    unit_price DECIMAL(12, 2) NOT NULL,
    discount_amount DECIMAL(12, 2) DEFAULT 0,
    total_price DECIMAL(12, 2) NOT NULL
);

-- Order Events (Event Sourcing)
CREATE TABLE order_events (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id UUID NOT NULL,
    event_type VARCHAR(100) NOT NULL,
    event_data JSONB NOT NULL,
    occurred_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    created_by VARCHAR(100)
);

CREATE INDEX idx_orders_user_id ON orders(user_id);
CREATE INDEX idx_orders_status ON orders(status);
CREATE INDEX idx_orders_created_at ON orders(created_at DESC);
CREATE INDEX idx_order_events_order_id ON order_events(order_id);
```

---

## ขั้นตอนที่ 3445: Common Library Design

```java
// common-lib/src/main/java/com/ecommerce/common/domain/BaseEntity.java
package com.ecommerce.common.domain;

import jakarta.persistence.*;
import lombok.Getter;
import lombok.Setter;
import org.springframework.data.annotation.CreatedDate;
import org.springframework.data.annotation.LastModifiedDate;
import org.springframework.data.jpa.domain.support.AuditingEntityListener;

import java.time.Instant;
import java.util.UUID;

@Getter
@Setter
@MappedSuperclass
@EntityListeners(AuditingEntityListener.class)
public abstract class BaseEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    @Column(name = "id", updatable = false, nullable = false)
    private UUID id;

    @CreatedDate
    @Column(name = "created_at", nullable = false, updatable = false)
    private Instant createdAt;

    @LastModifiedDate
    @Column(name = "updated_at", nullable = false)
    private Instant updatedAt;

    @Version
    @Column(name = "version")
    private Long version;
}
```

```java
// common-lib/src/main/java/com/ecommerce/common/api/ApiResponse.java
package com.ecommerce.common.api;

import com.fasterxml.jackson.annotation.JsonInclude;
import lombok.Builder;
import lombok.Data;

import java.time.Instant;

@Data
@Builder
@JsonInclude(JsonInclude.Include.NON_NULL)
public class ApiResponse<T> {

    private boolean success;
    private String message;
    private T data;
    private ErrorDetails error;
    private PageInfo pagination;
    
    @Builder.Default
    private Instant timestamp = Instant.now();
    
    private String requestId;

    // Static factory methods
    public static <T> ApiResponse<T> success(T data) {
        return ApiResponse.<T>builder()
            .success(true)
            .data(data)
            .build();
    }

    public static <T> ApiResponse<T> success(T data, String message) {
        return ApiResponse.<T>builder()
            .success(true)
            .message(message)
            .data(data)
            .build();
    }

    public static <T> ApiResponse<T> error(String message, String code) {
        return ApiResponse.<T>builder()
            .success(false)
            .error(ErrorDetails.builder()
                .code(code)
                .message(message)
                .build())
            .build();
    }

    @Data
    @Builder
    public static class ErrorDetails {
        private String code;
        private String message;
        private Object details;
    }

    @Data
    @Builder
    public static class PageInfo {
        private int page;
        private int size;
        private long totalElements;
        private int totalPages;
        private boolean first;
        private boolean last;
    }
}
```

```java
// common-lib/src/main/java/com/ecommerce/common/event/DomainEvent.java
package com.ecommerce.common.event;

import lombok.Getter;
import lombok.experimental.SuperBuilder;

import java.time.Instant;
import java.util.UUID;

@Getter
@SuperBuilder
public abstract class DomainEvent {

    private final String eventId;
    private final String eventType;
    private final String aggregateId;
    private final String aggregateType;
    private final Instant occurredAt;
    private final int version;
    private final String correlationId;
    private final String causationId;

    protected DomainEvent(String aggregateId, String aggregateType) {
        this.eventId = UUID.randomUUID().toString();
        this.eventType = this.getClass().getSimpleName();
        this.aggregateId = aggregateId;
        this.aggregateType = aggregateType;
        this.occurredAt = Instant.now();
        this.version = 1;
        this.correlationId = UUID.randomUUID().toString();
        this.causationId = null;
    }
}
```

---

## ขั้นตอนที่ 3446: API Design Principles

### REST API Standards

**URL Structure:**
```
/api/v1/{resource}
/api/v1/{resource}/{id}
/api/v1/{resource}/{id}/{sub-resource}
```

**HTTP Methods:**
```
GET    /api/v1/products          # List with pagination
GET    /api/v1/products/{id}     # Get by ID
POST   /api/v1/products          # Create
PUT    /api/v1/products/{id}     # Full Update
PATCH  /api/v1/products/{id}     # Partial Update
DELETE /api/v1/products/{id}     # Delete (Soft Delete)
```

### API Versioning Strategy

```java
// config/WebConfig.java - API Versioning
@Configuration
public class ApiVersionConfig {

    /**
     * เราใช้ URL-based versioning (/api/v1/, /api/v2/)
     * เหตุผล:
     * - ง่ายต่อการ Debug และ Test
     * - Cache-friendly
     * - ชัดเจนสำหรับ Developer
     */
    @Bean
    public RequestMappingHandlerMapping requestMappingHandlerMapping() {
        return new RequestMappingHandlerMapping();
    }
}

// BaseController.java
@RequestMapping("/api/v1")
public abstract class BaseV1Controller {
    // V1 Controllers ทั้งหมด extend class นี้
}

@RequestMapping("/api/v2")
public abstract class BaseV2Controller {
    // V2 Controllers ทั้งหมด extend class นี้
}
```

### Pagination Standard

```java
// common/PageRequest.java
@Data
public class PageRequest {
    
    @Min(0)
    @RequestParam(defaultValue = "0")
    private int page = 0;
    
    @Min(1)
    @Max(100)
    @RequestParam(defaultValue = "20")
    private int size = 20;
    
    @RequestParam(required = false)
    private String sortBy;
    
    @RequestParam(defaultValue = "DESC")
    private String sortDirection = "DESC";
    
    public org.springframework.data.domain.PageRequest toSpringPageRequest() {
        if (sortBy != null && !sortBy.isEmpty()) {
            Sort sort = Sort.by(
                Sort.Direction.fromString(sortDirection),
                sortBy
            );
            return org.springframework.data.domain.PageRequest.of(page, size, sort);
        }
        return org.springframework.data.domain.PageRequest.of(page, size);
    }
}
```

---

## ขั้นตอนที่ 3447: Non-Functional Requirements

### Availability: 99.99% (4 Nines)

```
99.99% uptime = 52.6 minutes downtime per year
```

**Strategy:**
```yaml
# kubernetes/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  replicas: 3  # ต้องมีอย่างน้อย 3 replicas
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0  # Zero downtime deployment
  template:
    spec:
      affinity:
        # กระจาย Pods ไปหลาย Availability Zones
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - order-service
            topologyKey: topology.kubernetes.io/zone
      containers:
      - name: order-service
        image: ecommerce/order-service:latest
        readinessProbe:
          httpGet:
            path: /actuator/health/readiness
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
          failureThreshold: 3
        livenessProbe:
          httpGet:
            path: /actuator/health/liveness
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
          limits:
            memory: "1Gi"
            cpu: "1000m"
```

### Latency: p99 < 100ms

```java
// config/PerformanceConfig.java
@Configuration
public class PerformanceConfig {

    /**
     * Virtual Threads สำหรับ I/O-bound Operations
     * ช่วยลด Thread Context Switching overhead
     */
    @Bean(TaskExecutionAutoConfiguration.APPLICATION_TASK_EXECUTOR_BEAN_NAME)
    public AsyncTaskExecutor asyncTaskExecutor() {
        return new TaskExecutorAdapter(Executors.newVirtualThreadPerTaskExecutor());
    }
    
    /**
     * Connection Pool Tuning
     * ปรับตาม Traffic Pattern
     */
    @Bean
    @ConfigurationProperties("spring.datasource.hikari")
    public HikariConfig hikariConfig() {
        HikariConfig config = new HikariConfig();
        config.setMaximumPoolSize(50);
        config.setMinimumIdle(10);
        config.setConnectionTimeout(3000);     // 3 seconds
        config.setIdleTimeout(300000);          // 5 minutes
        config.setMaxLifetime(1800000);         // 30 minutes
        config.setKeepaliveTime(60000);         // 1 minute
        return config;
    }
}
```

---

## ขั้นตอนที่ 3448: Capacity Planning

### Traffic Estimation

```
สมมุติว่าเราเป็น E-Commerce ขนาดกลางในไทย:

Daily Active Users (DAU): 500,000
Peak Hours: 12:00-14:00, 19:00-22:00
Peak Multiplier: 5x จาก Average

Average Requests/User/Day: 50
Total Daily Requests: 500,000 × 50 = 25,000,000

Average RPS = 25,000,000 / 86,400 = ~290 RPS
Peak RPS = 290 × 5 = ~1,450 RPS

กำหนด Safety Buffer = 2x
Target RPS = 1,450 × 2 = ~3,000 RPS
```

### Scaling Strategy

```java
// HPA Configuration
// kubernetes/hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: order-service-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-service
  minReplicas: 3
  maxReplicas: 50
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  - type: External
    external:
      metric:
        name: kafka_consumer_lag
        selector:
          matchLabels:
            consumer_group: order-service
      target:
        type: AverageValue
        averageValue: "1000"
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
      - type: Percent
        value: 100  # Double instances ได้ภายใน 1 นาที
        periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300  # รอ 5 นาทีก่อน Scale Down
      policies:
      - type: Pods
        value: 2
        periodSeconds: 60
```

---

## ขั้นตอนที่ 3449: Database Scaling Strategy

### Read Replica Setup

```java
// config/DataSourceConfig.java
@Configuration
public class DataSourceConfig {

    /**
     * Primary-Replica Pattern
     * Write ไปที่ Primary, Read จาก Replica
     * ช่วยลด Load บน Primary ได้ถึง 70-80%
     */
    @Bean
    @Primary
    @ConfigurationProperties("spring.datasource.primary")
    public DataSource primaryDataSource() {
        return DataSourceBuilder.create().build();
    }

    @Bean
    @ConfigurationProperties("spring.datasource.replica")
    public DataSource replicaDataSource() {
        return DataSourceBuilder.create().build();
    }

    @Bean
    public DataSource routingDataSource(
        @Qualifier("primaryDataSource") DataSource primary,
        @Qualifier("replicaDataSource") DataSource replica
    ) {
        Map<Object, Object> dataSourceMap = new HashMap<>();
        dataSourceMap.put(DataSourceType.PRIMARY, primary);
        dataSourceMap.put(DataSourceType.REPLICA, replica);

        RoutingDataSource routingDataSource = new RoutingDataSource();
        routingDataSource.setTargetDataSources(dataSourceMap);
        routingDataSource.setDefaultTargetDataSource(primary);
        return routingDataSource;
    }
}

// RoutingDataSource.java
public class RoutingDataSource extends AbstractRoutingDataSource {

    @Override
    protected Object determineCurrentLookupKey() {
        // ReadOnly transaction ไปที่ Replica
        return TransactionSynchronizationManager.isCurrentTransactionReadOnly()
            ? DataSourceType.REPLICA
            : DataSourceType.PRIMARY;
    }
}
```

### Caching Strategy

```java
// config/CacheConfig.java
@Configuration
@EnableCaching
public class CacheConfig {

    /**
     * Multi-Level Caching:
     * L1: Local Cache (Caffeine) - เร็วมาก แต่ไม่ share ระหว่าง instances
     * L2: Distributed Cache (Redis) - ช้ากว่าแต่ share ระหว่าง instances
     */
    @Bean
    public CacheManager cacheManager(RedisConnectionFactory redisConnectionFactory) {
        // L1 Cache Configuration
        CaffeineCache productL1Cache = new CaffeineCache(
            "products",
            Caffeine.newBuilder()
                .maximumSize(10_000)
                .expireAfterWrite(Duration.ofMinutes(5))
                .recordStats()
                .build()
        );

        // L2 Cache Configuration
        RedisCacheConfiguration redisCacheConfig = RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(30))
            .serializeKeysWith(RedisSerializationContext.SerializationPair
                .fromSerializer(new StringRedisSerializer()))
            .serializeValuesWith(RedisSerializationContext.SerializationPair
                .fromSerializer(new GenericJackson2JsonRedisSerializer()));

        return new TwoLevelCacheManager(productL1Cache, redisCacheConfig);
    }
}
```

---

## ขั้นตอนที่ 3450: Security Architecture

### Defense in Depth

```
Layer 1: Edge Security (WAF, DDoS Protection)
    - AWS WAF / Cloudflare
    - Rate Limiting ระดับ IP
    
Layer 2: API Gateway
    - Authentication (JWT Validation)
    - Authorization
    - Rate Limiting ระดับ User
    - Request Validation
    
Layer 3: Service Mesh (mTLS)
    - Service-to-Service Authentication
    - Encrypted Traffic
    
Layer 4: Application Security
    - Input Validation
    - SQL Injection Prevention
    - Business Logic Validation
    
Layer 5: Data Security
    - Encryption at Rest
    - PII Data Masking
    - Audit Logging
```

```java
// security/SecurityConfig.java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
            .csrf(AbstractHttpConfigurer::disable)
            .sessionManagement(session ->
                session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/v1/auth/**").permitAll()
                .requestMatchers(HttpMethod.GET, "/api/v1/products/**").permitAll()
                .requestMatchers("/actuator/health").permitAll()
                .requestMatchers("/actuator/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer(oauth2 ->
                oauth2.jwt(jwt -> jwt.jwtAuthenticationConverter(jwtConverter())))
            .addFilterBefore(
                new RateLimitingFilter(rateLimiter()),
                UsernamePasswordAuthenticationFilter.class
            )
            .build();
    }

    @Bean
    public Bucket rateLimiter() {
        // Bucket4j Rate Limiting
        // 100 requests per minute per user
        return Bucket.builder()
            .addLimit(Bandwidth.classic(100, Refill.greedy(100, Duration.ofMinutes(1))))
            .build();
    }
}
```

---

## ขั้นตอนที่ 3451: Monitoring & Observability Design

### Three Pillars of Observability

```yaml
# docker-compose.monitoring.yml
version: '3.8'

services:
  # Metrics: Prometheus + Grafana
  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./monitoring/prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
      - "9090:9090"

  grafana:
    image: grafana/grafana:latest
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    ports:
      - "3000:3000"

  # Traces: Jaeger (OpenTelemetry)
  jaeger:
    image: jaegertracing/all-in-one:latest
    ports:
      - "16686:16686"   # UI
      - "4317:4317"     # OTLP gRPC

  # Logs: ELK Stack
  elasticsearch:
    image: elasticsearch:8.11.0
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
    ports:
      - "9200:9200"

  logstash:
    image: logstash:8.11.0
    volumes:
      - ./monitoring/logstash.conf:/usr/share/logstash/pipeline/logstash.conf

  kibana:
    image: kibana:8.11.0
    ports:
      - "5601:5601"
```

```java
// config/ObservabilityConfig.java
@Configuration
public class ObservabilityConfig {

    /**
     * Custom Business Metrics
     * นอกจาก Default Metrics จาก Actuator
     */
    @Bean
    public MeterRegistry businessMetrics(MeterRegistry registry) {
        // Counter สำหรับ Order Events
        Counter.builder("orders.created")
            .description("Total orders created")
            .tag("currency", "THB")
            .register(registry);

        Counter.builder("orders.failed")
            .description("Total failed orders")
            .register(registry);

        // Gauge สำหรับ Active Sessions
        Gauge.builder("users.active_sessions", sessionService, SessionService::getActiveCount)
            .description("Current active user sessions")
            .register(registry);

        // Timer สำหรับ Payment Processing
        Timer.builder("payment.processing.duration")
            .description("Payment processing duration")
            .publishPercentiles(0.5, 0.95, 0.99)
            .register(registry);

        return registry;
    }
}
```

---

## ขั้นตอนที่ 3452: Deployment Architecture

### Kubernetes Multi-Region Setup

```yaml
# kubernetes/namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: ecommerce-prod
  labels:
    environment: production
    team: platform-engineering
    
---
# ConfigMap for shared config
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: ecommerce-prod
data:
  SPRING_PROFILES_ACTIVE: "production"
  LOG_LEVEL: "INFO"
  KAFKA_BOOTSTRAP_SERVERS: "kafka-cluster:9092"
  
---
# Secret (ใน Production ใช้ Vault หรือ AWS Secrets Manager)
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
  namespace: ecommerce-prod
type: Opaque
stringData:
  DB_PASSWORD: "${DB_PASSWORD}"
  JWT_SECRET: "${JWT_SECRET}"
  STRIPE_API_KEY: "${STRIPE_API_KEY}"
```

---

## สรุป Part 96

ในส่วนนี้เราได้ออกแบบ:

1. **System Architecture** - Microservices ที่ communicate ผ่าน Kafka
2. **Technology Stack** - Spring Boot 3.2, PostgreSQL, Elasticsearch, Redis, Kafka
3. **Data Models** - User, Product, Order Domain
4. **API Standards** - RESTful, Versioning, Pagination
5. **NFRs** - 99.99% Availability, p99 < 100ms
6. **Capacity Planning** - รองรับ 3,000 RPS
7. **Security Architecture** - Defense in Depth
8. **Observability** - Metrics, Traces, Logs

ใน Part 97 เราจะลงมือ implement แต่ละ Service อย่างละเอียด

---

*[← Part 95: Advanced Architecture Patterns](./part-95-advanced-architecture.md) | [Part 97: Capstone Implementation →](./part-97-capstone-implementation.md)*

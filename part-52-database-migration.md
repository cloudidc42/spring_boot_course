# Part 52: Database Migration with Flyway
## ขั้นตอนที่ 1681-1720

**ระดับ:** Advanced  
**เวลาเรียน:** 3-4 ชั่วโมง  
**เป้าหมาย:** เข้าใจและใช้งาน Flyway สำหรับ Database Migration อย่างมืออาชีพ ครอบคลุม versioning, zero-downtime migration, data migration และการจัดการ multi-environment

---

## ขั้นตอนที่ 1681: ทำไมต้องใช้ Database Migration?

ปัญหาที่พบบ่อยใน production คือ schema ของฐานข้อมูลเปลี่ยนแปลงไม่ synchronized กับโค้ด การใช้ Flyway ช่วยแก้ปัญหานี้ด้วย:
- **Version Control สำหรับ DB**: ติดตามการเปลี่ยนแปลงทุกอย่างใน Git
- **Repeatable**: ทุก environment ได้ schema เหมือนกัน
- **Auditable**: รู้ว่า migration ไหน run ไปแล้วและเมื่อไร
- **Automated**: ทำงานอัตโนมัติเมื่อ application start

### เพิ่ม Flyway ใน pom.xml

```xml
<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-core</artifactId>
</dependency>
<!-- สำหรับ PostgreSQL 10+ -->
<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-database-postgresql</artifactId>
</dependency>
```

### Configuration พื้นฐาน

```yaml
# application.yml
spring:
  flyway:
    enabled: true
    locations: classpath:db/migration
    baseline-on-migrate: true
    baseline-version: 0
    validate-on-migrate: true
    out-of-order: false  # ไม่อนุญาต migration version ย้อนหลัง
    table: flyway_schema_history  # ชื่อ table สำหรับเก็บประวัติ
```

---

## ขั้นตอนที่ 1682: Naming Convention และ File Structure

Flyway ใช้ naming convention ที่สำคัญมาก:

```
V{version}__{description}.sql
R__{description}.sql    <- Repeatable migrations
U{version}__{description}.sql  <- Undo migrations
```

```
src/main/resources/
└── db/
    └── migration/
        ├── V1__initial_schema.sql
        ├── V2__add_categories.sql
        ├── V3__add_product_indexes.sql
        ├── V4__add_user_address.sql
        ├── V4.1__add_user_preferences.sql  <- decimal version ได้
        ├── V5__rename_columns.sql
        └── R__views_and_procedures.sql     <- Repeatable: run เมื่อ checksum เปลี่ยน
```

กฎสำคัญ:
- Version ต้องเรียงลำดับขึ้น (ห้ามซ้ำ, ห้ามลด)
- Description ใช้ underscore แทน space
- สองขีด `__` คั่นระหว่าง version กับ description

---

## ขั้นตอนที่ 1683: Migration แรก - Initial Schema

```sql
-- V1__initial_schema.sql
-- สร้าง schema ของแอปพลิเคชัน

CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    username VARCHAR(50) NOT NULL,
    email VARCHAR(100) NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CONSTRAINT uq_users_username UNIQUE (username),
    CONSTRAINT uq_users_email UNIQUE (email)
);

CREATE TABLE products (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    price DECIMAL(10,2) NOT NULL,
    stock_qty INT NOT NULL DEFAULT 0,
    active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CONSTRAINT chk_products_price CHECK (price > 0),
    CONSTRAINT chk_products_stock CHECK (stock_qty >= 0)
);

-- สร้าง updated_at trigger สำหรับ auto-update
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ language 'plpgsql';

CREATE TRIGGER update_users_updated_at
    BEFORE UPDATE ON users
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at_column();
```

---

## ขั้นตอนที่ 1684: Adding Columns (Backward Compatible)

เพิ่ม column ใหม่อย่างปลอดภัยโดยไม่กระทบ application เดิม:

```sql
-- V4__add_user_profile_fields.sql
-- เพิ่ม column ที่ nullable เพื่อ backward compatibility

ALTER TABLE users 
    ADD COLUMN IF NOT EXISTS first_name VARCHAR(50),
    ADD COLUMN IF NOT EXISTS last_name VARCHAR(50),
    ADD COLUMN IF NOT EXISTS phone_number VARCHAR(20),
    ADD COLUMN IF NOT EXISTS avatar_url VARCHAR(500),
    ADD COLUMN IF NOT EXISTS birth_date DATE;

-- เพิ่ม column ที่มี default value (ไม่ต้องระบุในโค้ดเดิม)
ALTER TABLE users
    ADD COLUMN IF NOT EXISTS email_verified BOOLEAN NOT NULL DEFAULT FALSE,
    ADD COLUMN IF NOT EXISTS role VARCHAR(20) NOT NULL DEFAULT 'CUSTOMER';

COMMENT ON COLUMN users.role IS 'User role: CUSTOMER, ADMIN, SELLER';
```

### หลักการ Backward Compatible Changes

การเปลี่ยนแปลงที่ **ปลอดภัย** (backward compatible):
- เพิ่ม nullable column ใหม่
- เพิ่ม column ที่มี default value
- เพิ่ม table ใหม่
- เพิ่ม index ใหม่
- ขยาย VARCHAR ให้ใหญ่ขึ้น

การเปลี่ยนแปลงที่ **อันตราย** (non-backward compatible):
- ลบ column
- เปลี่ยนชื่อ column (โดยตรง)
- เปลี่ยน data type
- เพิ่ม NOT NULL constraint ในตารางที่มีข้อมูลอยู่แล้ว

---

## ขั้นตอนที่ 1685: Renaming Columns (Zero-Downtime Strategy)

การเปลี่ยนชื่อ column ที่ถูกต้องต้องทำ 3 ขั้นตอนใน 3 deployment:

```sql
-- Phase 1: V5__rename_stock_qty_phase1.sql
-- เพิ่ม column ใหม่ (ทั้ง column เดิมและใหม่ต้องอยู่พร้อมกัน)
ALTER TABLE products 
    ADD COLUMN IF NOT EXISTS stock_quantity INT;

-- Copy ข้อมูลจาก column เดิม
UPDATE products SET stock_quantity = stock_qty WHERE stock_quantity IS NULL;

-- สร้าง trigger เพื่อ sync ข้อมูลระหว่าง transition
CREATE OR REPLACE FUNCTION sync_stock_columns()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'INSERT' OR TG_OP = 'UPDATE' THEN
        IF NEW.stock_qty IS DISTINCT FROM OLD.stock_qty THEN
            NEW.stock_quantity = NEW.stock_qty;
        END IF;
        IF NEW.stock_quantity IS DISTINCT FROM OLD.stock_quantity THEN
            NEW.stock_qty = NEW.stock_quantity;
        END IF;
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER sync_stock_trigger
    BEFORE INSERT OR UPDATE ON products
    FOR EACH ROW EXECUTE FUNCTION sync_stock_columns();
```

```sql
-- Phase 2: ไม่มี migration - Deploy application ที่ใช้ column ใหม่ (stock_quantity)
-- application ต้องรองรับทั้งสอง column ในช่วงนี้
```

```sql
-- Phase 3: V6__rename_stock_qty_phase3.sql
-- ลบ trigger และ column เดิม หลังจาก verify ว่า application ใหม่ทำงานถูกต้อง

DROP TRIGGER IF EXISTS sync_stock_trigger ON products;
DROP FUNCTION IF EXISTS sync_stock_columns();

-- เพิ่ม constraint บน column ใหม่
ALTER TABLE products 
    ALTER COLUMN stock_quantity SET NOT NULL,
    ALTER COLUMN stock_quantity SET DEFAULT 0;

-- ลบ column เดิม
ALTER TABLE products DROP COLUMN IF EXISTS stock_qty;
```

---

## ขั้นตอนที่ 1686: Splitting Tables (Zero-Downtime)

แยกตารางขนาดใหญ่ออกเป็น 2 ตาราง:

```sql
-- V7__split_user_address_phase1.sql
-- แยก address ออกจาก users table

CREATE TABLE user_addresses (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    address_type VARCHAR(20) NOT NULL DEFAULT 'SHIPPING',
    address_line1 VARCHAR(255) NOT NULL,
    address_line2 VARCHAR(255),
    city VARCHAR(100) NOT NULL,
    postal_code VARCHAR(20) NOT NULL,
    country VARCHAR(100) NOT NULL DEFAULT 'Thailand',
    is_default BOOLEAN NOT NULL DEFAULT FALSE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_user_addresses_user_id ON user_addresses(user_id);

-- Migrate ข้อมูลที่มีอยู่ (ถ้า users table มี address fields)
INSERT INTO user_addresses (user_id, address_line1, city, postal_code, country, is_default)
SELECT 
    id,
    COALESCE(address, 'N/A'),
    COALESCE(city, 'Unknown'),
    COALESCE(postal_code, '00000'),
    'Thailand',
    TRUE
FROM users
WHERE address IS NOT NULL;
```

---

## ขั้นตอนที่ 1687: Data Migrations

Migration ที่ต้อง transform ข้อมูล:

```sql
-- V8__normalize_email_data.sql
-- Normalize email ให้เป็นตัวพิมพ์เล็กทั้งหมด

-- ตรวจสอบว่ามี duplicate emails ไหม (หลัง normalize)
DO $$
DECLARE
    duplicate_count INT;
BEGIN
    SELECT COUNT(*) INTO duplicate_count
    FROM (
        SELECT LOWER(email) as normalized_email
        FROM users
        GROUP BY LOWER(email)
        HAVING COUNT(*) > 1
    ) duplicates;
    
    IF duplicate_count > 0 THEN
        RAISE EXCEPTION 'Found % duplicate emails after normalization', duplicate_count;
    END IF;
END $$;

-- อัปเดต email ให้เป็น lowercase
UPDATE users SET email = LOWER(email);

-- เพิ่ม index สำหรับ case-insensitive search
CREATE UNIQUE INDEX IF NOT EXISTS idx_users_email_lower 
ON users (LOWER(email));
```

```sql
-- V9__backfill_product_slugs.sql
-- สร้าง URL slug สำหรับสินค้าทุกตัว

ALTER TABLE products 
    ADD COLUMN IF NOT EXISTS slug VARCHAR(250);

-- สร้าง function สำหรับ generate slug
CREATE OR REPLACE FUNCTION generate_slug(input_text TEXT)
RETURNS TEXT AS $$
BEGIN
    RETURN LOWER(
        REGEXP_REPLACE(
            REGEXP_REPLACE(input_text, '[^a-zA-Z0-9\s-]', '', 'g'),
            '\s+', '-', 'g'
        )
    );
END;
$$ LANGUAGE plpgsql;

-- Backfill slug สำหรับ products ที่มีอยู่
UPDATE products 
SET slug = generate_slug(name) || '-' || id
WHERE slug IS NULL;

-- เพิ่ม constraint หลังจาก backfill เสร็จ
ALTER TABLE products 
    ALTER COLUMN slug SET NOT NULL;

CREATE UNIQUE INDEX IF NOT EXISTS idx_products_slug ON products(slug);
```

---

## ขั้นตอนที่ 1688: Java-Based Flyway Callbacks

เมื่อต้องการทำ logic ที่ซับซ้อนเกินกว่า SQL จะจัดการได้:

```java
// src/main/java/com/example/migration/callbacks/AfterMigrateCallback.java
package com.example.migration.callbacks;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.flywaydb.core.api.callback.Callback;
import org.flywaydb.core.api.callback.Context;
import org.flywaydb.core.api.callback.Event;
import org.springframework.stereotype.Component;

@Component
@RequiredArgsConstructor
@Slf4j
public class AfterMigrateCallback implements Callback {
    
    private final CacheWarmupService cacheWarmupService;
    private final SearchIndexService searchIndexService;
    
    @Override
    public boolean supports(Event event, Context context) {
        // ทำงานหลัง migration สำเร็จ
        return event == Event.AFTER_MIGRATE;
    }
    
    @Override
    public boolean canHandleInTransaction(Event event, Context context) {
        return false;  // ทำงานนอก transaction
    }
    
    @Override
    public void handle(Event event, Context context) {
        if (event == Event.AFTER_MIGRATE) {
            log.info("Migration completed, warming up caches...");
            try {
                cacheWarmupService.warmup();
                searchIndexService.reindex();
                log.info("Post-migration tasks completed successfully");
            } catch (Exception e) {
                log.error("Post-migration task failed (non-fatal): {}", e.getMessage());
            }
        }
    }
    
    @Override
    public String getCallbackName() {
        return "AfterMigrateCallback";
    }
}
```

```java
// src/main/java/com/example/migration/callbacks/BeforeMigrateCallback.java
package com.example.migration.callbacks;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.flywaydb.core.api.callback.Callback;
import org.flywaydb.core.api.callback.Context;
import org.flywaydb.core.api.callback.Event;
import org.springframework.stereotype.Component;

@Component
@RequiredArgsConstructor
@Slf4j
public class BeforeMigrateCallback implements Callback {
    
    private final DatabaseBackupService backupService;
    
    @Override
    public boolean supports(Event event, Context context) {
        return event == Event.BEFORE_MIGRATE;
    }
    
    @Override
    public boolean canHandleInTransaction(Event event, Context context) {
        return false;
    }
    
    @Override
    public void handle(Event event, Context context) {
        if (event == Event.BEFORE_MIGRATE) {
            log.info("Starting migration, creating backup snapshot...");
            backupService.createSnapshot("pre-migration-" + System.currentTimeMillis());
            log.info("Backup created");
        }
    }
    
    @Override
    public String getCallbackName() {
        return "BeforeMigrateCallback";
    }
}
```

### Register Callbacks

```java
// src/main/java/com/example/config/FlywayConfig.java
package com.example.config;

import lombok.RequiredArgsConstructor;
import org.flywaydb.core.api.callback.Callback;
import org.springframework.boot.autoconfigure.flyway.FlywayConfigurationCustomizer;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.util.List;

@Configuration
@RequiredArgsConstructor
public class FlywayConfig {
    
    private final List<Callback> flywayCallbacks;
    
    @Bean
    public FlywayConfigurationCustomizer flywayConfigurationCustomizer() {
        return config -> config.callbacks(
            flywayCallbacks.toArray(new Callback[0])
        );
    }
}
```

---

## ขั้นตอนที่ 1689: Java-Based Migrations

บางครั้ง SQL ไม่พอ ใช้ Java ได้:

```java
// src/main/java/db/migration/V10__EncryptSensitiveData.java
package db.migration;

import org.flywaydb.core.api.migration.BaseJavaMigration;
import org.flywaydb.core.api.migration.Context;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;

import java.sql.PreparedStatement;
import java.sql.ResultSet;
import java.sql.Statement;

/**
 * Java-based migration สำหรับ encrypt ข้อมูล sensitive
 * ใช้ BCrypt สำหรับ hash passwords ที่ยังไม่ได้ hash
 */
public class V10__EncryptSensitiveData extends BaseJavaMigration {
    
    private static final BCryptPasswordEncoder encoder = new BCryptPasswordEncoder(12);
    
    @Override
    public void migrate(Context context) throws Exception {
        var connection = context.getConnection();
        
        // ดึง users ที่มี plain text password
        String selectSql = "SELECT id, password_hash FROM users " +
                          "WHERE password_hash NOT LIKE '$2a$%'";
        
        try (Statement stmt = connection.createStatement();
             ResultSet rs = stmt.executeQuery(selectSql);
             PreparedStatement updateStmt = connection.prepareStatement(
                 "UPDATE users SET password_hash = ? WHERE id = ?")) {
            
            int count = 0;
            while (rs.next()) {
                long userId = rs.getLong("id");
                String plainPassword = rs.getString("password_hash");
                
                String hashed = encoder.encode(plainPassword);
                updateStmt.setString(1, hashed);
                updateStmt.setLong(2, userId);
                updateStmt.addBatch();
                
                count++;
                if (count % 100 == 0) {
                    updateStmt.executeBatch();
                }
            }
            
            if (count % 100 != 0) {
                updateStmt.executeBatch();
            }
            
            System.out.printf("Encrypted passwords for %d users%n", count);
        }
    }
}
```

---

## ขั้นตอนที่ 1690: Handling Migration Failures

เมื่อ migration ล้มเหลว ต้องรู้วิธีแก้ไข:

```java
// src/main/java/com/example/config/FlywayRepairConfig.java
package com.example.config;

import lombok.extern.slf4j.Slf4j;
import org.flywaydb.core.Flyway;
import org.springframework.boot.ApplicationArguments;
import org.springframework.boot.ApplicationRunner;
import org.springframework.context.annotation.Profile;
import org.springframework.stereotype.Component;

/**
 * เรียกใช้ Flyway repair mode เมื่อ migration ล้มเหลว
 * ใช้กับ profile "repair" เท่านั้น
 */
@Component
@Profile("repair")
@Slf4j
public class FlywayRepairRunner implements ApplicationRunner {
    
    private final Flyway flyway;
    
    public FlywayRepairRunner(Flyway flyway) {
        this.flyway = flyway;
    }
    
    @Override
    public void run(ApplicationArguments args) {
        log.warn("Running Flyway repair...");
        flyway.repair();
        log.info("Flyway repair completed");
    }
}
```

### วิธีแก้ไข Failed Migration

```bash
# 1. ดู migration history
SELECT * FROM flyway_schema_history ORDER BY installed_rank DESC;

# 2. ถ้า migration ล้มเหลวจะมีสถานะ 'F'
# repair จะลบ failed migration record และทำให้ลอง run ใหม่ได้
./mvnw flyway:repair

# 3. แก้ไข SQL script แล้ว run ใหม่
./mvnw flyway:migrate
```

### ป้องกัน Migration ล้มเหลว

```sql
-- ใช้ transaction และ DO block เพื่อ rollback เมื่อเกิด error
DO $$
BEGIN
    -- Validate ข้อมูลก่อน migration
    IF EXISTS (SELECT 1 FROM products WHERE price <= 0) THEN
        RAISE EXCEPTION 'Invalid product prices found. Migration aborted.';
    END IF;
    
    -- ทำการ migration
    ALTER TABLE products ADD COLUMN IF NOT EXISTS discounted_price DECIMAL(10,2);
    UPDATE products SET discounted_price = price * 0.9 WHERE active = TRUE;
    
    RAISE NOTICE 'Migration completed successfully';
END $$;
```

---

## ขั้นตอนที่ 1691: Multi-Environment Migration Management

จัดการ migration สำหรับหลาย environment:

```yaml
# application-dev.yml
spring:
  flyway:
    locations: classpath:db/migration,classpath:db/testdata
    clean-disabled: false  # อนุญาต flyway:clean ใน dev
    out-of-order: true     # อนุญาต out-of-order ใน dev

---
# application-test.yml
spring:
  flyway:
    locations: classpath:db/migration,classpath:db/testdata
    clean-on-validation-error: true  # clean แล้ว re-migrate เมื่อ test

---
# application-prod.yml
spring:
  flyway:
    locations: classpath:db/migration
    clean-disabled: true   # ห้าม flyway:clean ใน production เด็ดขาด
    out-of-order: false
    validate-on-migrate: true
    baseline-on-migrate: false
```

### Test Data Migrations (Dev/Test เท่านั้น)

```sql
-- src/main/resources/db/testdata/V100__test_seed_data.sql
-- migration นี้จะ run เฉพาะใน dev และ test

INSERT INTO users (username, email, password_hash, role) VALUES
    ('admin', 'admin@test.com', '$2a$12$...hashed...', 'ADMIN'),
    ('testuser', 'user@test.com', '$2a$12$...hashed...', 'CUSTOMER')
ON CONFLICT (username) DO NOTHING;

INSERT INTO products (name, price, stock_qty) VALUES
    ('Test Product 1', 100.00, 50),
    ('Test Product 2', 250.00, 30)
ON CONFLICT DO NOTHING;
```

---

## ขั้นตอนที่ 1692: Repeatable Migrations

Repeatable migrations จะ run ใหม่ทุกครั้งที่ checksum เปลี่ยน:

```sql
-- src/main/resources/db/migration/R__views.sql
-- สร้าง/อัปเดต database views
-- จะ run ใหม่อัตโนมัติเมื่อไฟล์นี้เปลี่ยน

CREATE OR REPLACE VIEW product_summary AS
SELECT 
    p.id,
    p.name,
    p.sku,
    p.price,
    p.sale_price,
    COALESCE(p.sale_price, p.price) as effective_price,
    p.stock_quantity,
    p.active,
    c.name as category_name,
    c.id as category_id,
    p.created_at
FROM products p
LEFT JOIN categories c ON c.id = p.category_id
WHERE p.active = TRUE;

CREATE OR REPLACE VIEW order_summary AS
SELECT 
    o.id,
    o.order_number,
    o.status,
    o.total_amount,
    o.created_at,
    u.username,
    u.email,
    COUNT(oi.id) as item_count
FROM orders o
JOIN users u ON u.id = o.user_id
JOIN order_items oi ON oi.order_id = o.id
GROUP BY o.id, o.order_number, o.status, o.total_amount, 
         o.created_at, u.username, u.email;
```

```sql
-- src/main/resources/db/migration/R__functions.sql
-- Functions และ Stored Procedures

CREATE OR REPLACE FUNCTION calculate_cart_total(p_cart_id BIGINT)
RETURNS DECIMAL(12,2) AS $$
DECLARE
    v_total DECIMAL(12,2);
BEGIN
    SELECT COALESCE(SUM(
        COALESCE(p.sale_price, p.price) * ci.quantity
    ), 0)
    INTO v_total
    FROM cart_items ci
    JOIN products p ON p.id = ci.product_id
    WHERE ci.cart_id = p_cart_id
      AND p.active = TRUE;
    
    RETURN v_total;
END;
$$ LANGUAGE plpgsql;
```

---

## ขั้นตอนที่ 1693: Flyway Maven Plugin

```xml
<!-- pom.xml - Flyway Maven Plugin -->
<plugin>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-maven-plugin</artifactId>
    <version>10.4.1</version>
    <configuration>
        <url>jdbc:postgresql://localhost:5432/ecommerce_db</url>
        <user>${db.username}</user>
        <password>${db.password}</password>
        <locations>
            <location>classpath:db/migration</location>
        </locations>
    </configuration>
</plugin>
```

### คำสั่ง Flyway ที่ใช้บ่อย

```bash
# ดู migration status
./mvnw flyway:info

# Run pending migrations
./mvnw flyway:migrate

# Validate migrations (ตรวจสอบ checksum)
./mvnw flyway:validate

# Repair failed migrations
./mvnw flyway:repair

# ล้าง database (DEV ONLY! อันตรายมากใน production)
./mvnw flyway:clean

# Baseline existing database
./mvnw flyway:baseline
```

---

## ขั้นตอนที่ 1694: Testing Migrations

```java
// src/test/java/com/example/migration/MigrationTest.java
package com.example.migration;

import org.flywaydb.core.Flyway;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.test.context.ActiveProfiles;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest
@ActiveProfiles("test")
class MigrationTest {
    
    @Autowired
    private Flyway flyway;
    
    @Autowired
    private JdbcTemplate jdbcTemplate;
    
    @Test
    void allMigrationsShouldSucceed() {
        // ตรวจสอบว่า migration ทั้งหมดสำเร็จ
        var info = flyway.info();
        var failedMigrations = java.util.Arrays.stream(info.all())
            .filter(m -> m.getState().isFailed())
            .toList();
        
        assertThat(failedMigrations)
            .withFailMessage("Found failed migrations: %s", failedMigrations)
            .isEmpty();
    }
    
    @Test
    void usersTableShouldHaveExpectedColumns() {
        // ตรวจสอบ schema หลัง migration
        var columns = jdbcTemplate.queryForList(
            "SELECT column_name FROM information_schema.columns " +
            "WHERE table_name = 'users' ORDER BY column_name",
            String.class
        );
        
        assertThat(columns).containsAll(
            java.util.List.of("id", "username", "email", "created_at", "updated_at")
        );
    }
    
    @Test
    void migrationsShouldBeIdempotentAfterRepair() {
        // ทดสอบว่า repair แล้ว re-migrate ได้
        flyway.repair();
        var result = flyway.migrate();
        assertThat(result.success).isTrue();
    }
}
```

---

## ขั้นตอนที่ 1695: Large Table Migration Strategy

สำหรับตารางที่มีข้อมูลล้านๆ แถว ต้องระวังเป็นพิเศษ:

```sql
-- V15__add_index_to_large_table.sql
-- เพิ่ม index แบบ CONCURRENTLY เพื่อไม่ lock table

-- PostgreSQL: สร้าง index แบบ non-blocking
-- หมายเหตุ: CONCURRENTLY ทำงานนอก transaction
CREATE INDEX CONCURRENTLY IF NOT EXISTS 
    idx_orders_created_at ON orders(created_at);

-- สร้าง partial index เพื่อประหยัดพื้นที่
CREATE INDEX CONCURRENTLY IF NOT EXISTS
    idx_orders_pending ON orders(user_id, created_at)
    WHERE status = 'PENDING';
```

```java
// src/main/java/db/migration/V16__BatchUpdateProducts.java
package db.migration;

import org.flywaydb.core.api.migration.BaseJavaMigration;
import org.flywaydb.core.api.migration.Context;

import java.sql.PreparedStatement;
import java.sql.ResultSet;
import java.sql.Statement;
import java.util.ArrayList;
import java.util.List;

/**
 * Batch update สำหรับตารางที่มีข้อมูลจำนวนมาก
 * ทำ batch เพื่อหลีกเลี่ยง lock timeout
 */
public class V16__BatchUpdateProducts extends BaseJavaMigration {
    
    private static final int BATCH_SIZE = 1000;
    
    @Override
    public void migrate(Context context) throws Exception {
        var connection = context.getConnection();
        
        // เพิ่ม column ใหม่
        try (Statement stmt = connection.createStatement()) {
            stmt.execute(
                "ALTER TABLE products ADD COLUMN IF NOT EXISTS search_vector TSVECTOR"
            );
        }
        
        // อ่าน IDs ทั้งหมด
        List<Long> allIds = new ArrayList<>();
        try (Statement stmt = connection.createStatement();
             ResultSet rs = stmt.executeQuery("SELECT id FROM products")) {
            while (rs.next()) {
                allIds.add(rs.getLong("id"));
            }
        }
        
        // อัปเดตทีละ batch
        String updateSql = "UPDATE products SET search_vector = " +
            "to_tsvector('english', name || ' ' || COALESCE(description, '')) " +
            "WHERE id = ?";
        
        try (PreparedStatement stmt = connection.prepareStatement(updateSql)) {
            for (int i = 0; i < allIds.size(); i++) {
                stmt.setLong(1, allIds.get(i));
                stmt.addBatch();
                
                if ((i + 1) % BATCH_SIZE == 0) {
                    stmt.executeBatch();
                    // Commit แต่ละ batch เพื่อปลดล็อก
                    connection.commit();
                    System.out.printf("Processed %d / %d products%n", 
                        i + 1, allIds.size());
                }
            }
            stmt.executeBatch();
        }
    }
}
```

---

## ขั้นตอนที่ 1696: Migration Best Practices Summary

### Checklist ก่อน Production Migration

```markdown
## Pre-Migration Checklist

- [ ] Backup database แล้ว
- [ ] Test migration ใน staging environment แล้ว
- [ ] Migration เป็น backward compatible (หรือมี deployment plan)
- [ ] ไม่มี long-running transactions ใน migration
- [ ] ตรวจสอบ execution time ใน staging (ถ้า > 30 วินาที ต้องหาวิธีอื่น)
- [ ] มี rollback plan ถ้า migration ล้มเหลว
- [ ] Notify team ก่อน deploy ถ้า migration กระทบการใช้งาน
```

```java
// src/main/java/com/example/config/FlywayHealthCheck.java
package com.example.config;

import lombok.RequiredArgsConstructor;
import org.flywaydb.core.Flyway;
import org.flywaydb.core.api.MigrationInfo;
import org.springframework.boot.actuate.health.Health;
import org.springframework.boot.actuate.health.HealthIndicator;
import org.springframework.stereotype.Component;

import java.util.Arrays;

@Component
@RequiredArgsConstructor
public class FlywayHealthCheck implements HealthIndicator {
    
    private final Flyway flyway;
    
    @Override
    public Health health() {
        var info = flyway.info();
        var pending = Arrays.stream(info.pending()).count();
        var failed = Arrays.stream(info.all())
            .filter(m -> m.getState().isFailed())
            .count();
        
        if (failed > 0) {
            return Health.down()
                .withDetail("failedMigrations", failed)
                .build();
        }
        
        if (pending > 0) {
            return Health.up()
                .withDetail("pendingMigrations", pending)
                .withDetail("status", "migrations_pending")
                .build();
        }
        
        return Health.up()
            .withDetail("migrationsApplied", info.applied().length)
            .withDetail("status", "up_to_date")
            .build();
    }
}
```

---

## สรุปท้ายบท

ในส่วนนี้เราได้เรียนรู้:

1. **Flyway setup** และ configuration สำหรับ Spring Boot
2. **Naming convention** และโครงสร้างไฟล์ migration
3. **Zero-downtime migration** strategies สำหรับ renaming, splitting tables
4. **Data migrations** ทั้งแบบ SQL และ Java
5. **Callbacks** สำหรับ pre/post migration tasks
6. **Multi-environment** management
7. **Batch processing** สำหรับ large tables
8. **Health checks** และ monitoring

---

*[← Part 51: Real-World Project](./part-51-realworld-project.md) | [Part 53: GraalVM Native →](./part-53-graalvm-native.md)*

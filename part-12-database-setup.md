# Part 12: Database Setup - PostgreSQL & MySQL
## ขั้นตอนที่ 266-295

> **ระดับ:** พื้นฐาน-กลาง (Beginner-Intermediate)  
> **เวลาเรียน:** 3-4 ชั่วโมง  
> **เป้าหมาย:** ตั้งค่า PostgreSQL และ MySQL กับ Spring Boot อย่างถูกต้อง

---

## ขั้นตอนที่ 266: PostgreSQL Setup

```yaml
# docker-compose.yml
version: '3.8'
services:
  postgres:
    image: postgres:16-alpine
    container_name: spring_postgres
    environment:
      POSTGRES_DB: springdb
      POSTGRES_USER: springuser
      POSTGRES_PASSWORD: springpass
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./sql/init.sql:/docker-entrypoint-initdb.d/init.sql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U springuser -d springdb"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  postgres_data:
```

```bash
# Start
docker-compose up -d postgres

# Connect
psql -h localhost -U springuser -d springdb
```

---

## ขั้นตอนที่ 267: application.yml สำหรับ PostgreSQL

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/springdb
    username: springuser
    password: springpass
    driver-class-name: org.postgresql.Driver
    hikari:
      pool-name: SpringHikariCP
      minimum-idle: 5
      maximum-pool-size: 20
      idle-timeout: 300000      # 5 minutes
      connection-timeout: 20000 # 20 seconds
      max-lifetime: 1200000     # 20 minutes
      auto-commit: false
      connection-test-query: SELECT 1

  jpa:
    hibernate:
      ddl-auto: validate  # production: validate, dev: update or create-drop
    show-sql: false       # true ใน dev เท่านั้น
    properties:
      hibernate:
        dialect: org.hibernate.dialect.PostgreSQLDialect
        format_sql: true
        jdbc:
          batch_size: 50
          order_inserts: true
          order_updates: true
        cache:
          use_second_level_cache: false
    open-in-view: false   # IMPORTANT: ปิด เพื่อ performance
```

---

## ขั้นตอนที่ 268: Flyway Database Migration

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-core</artifactId>
</dependency>
<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-database-postgresql</artifactId>
</dependency>
```

```yaml
# application.yml
spring:
  flyway:
    enabled: true
    locations: classpath:db/migration
    baseline-on-migrate: true
    validate-on-migrate: true
    out-of-order: false
```

```sql
-- src/main/resources/db/migration/V1__create_users_table.sql
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    username VARCHAR(50) NOT NULL,
    email VARCHAR(255) NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    first_name VARCHAR(100),
    last_name VARCHAR(100),
    role VARCHAR(20) NOT NULL DEFAULT 'USER',
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    last_login_at TIMESTAMP,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP,
    created_by VARCHAR(100),
    updated_by VARCHAR(100),
    version BIGINT NOT NULL DEFAULT 0,
    
    CONSTRAINT uk_users_email UNIQUE (email),
    CONSTRAINT uk_users_username UNIQUE (username),
    CONSTRAINT ck_users_role CHECK (role IN ('USER', 'ADMIN', 'MODERATOR'))
);

CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_username ON users(username);
CREATE INDEX idx_users_role ON users(role);
```

```sql
-- V2__create_products_table.sql
CREATE TABLE categories (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    slug VARCHAR(100) NOT NULL,
    description TEXT,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP,
    version BIGINT NOT NULL DEFAULT 0,
    
    CONSTRAINT uk_categories_slug UNIQUE (slug)
);

CREATE TABLE products (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    price DECIMAL(10, 2) NOT NULL,
    stock INTEGER NOT NULL DEFAULT 0,
    image_url VARCHAR(500),
    status VARCHAR(20) NOT NULL DEFAULT 'ACTIVE',
    category_id BIGINT REFERENCES categories(id),
    view_count BIGINT NOT NULL DEFAULT 0,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP,
    created_by VARCHAR(100),
    updated_by VARCHAR(100),
    version BIGINT NOT NULL DEFAULT 0,
    
    CONSTRAINT ck_products_price CHECK (price > 0),
    CONSTRAINT ck_products_stock CHECK (stock >= 0),
    CONSTRAINT ck_products_status CHECK (status IN ('ACTIVE', 'INACTIVE', 'OUT_OF_STOCK'))
);

CREATE INDEX idx_products_category ON products(category_id);
CREATE INDEX idx_products_status ON products(status);
CREATE INDEX idx_products_price ON products(price);
```

```sql
-- V3__seed_data.sql  (Initial data)
INSERT INTO categories (name, slug, description) VALUES
    ('Electronics', 'electronics', 'Electronic devices and accessories'),
    ('Clothing', 'clothing', 'Fashion and apparel'),
    ('Books', 'books', 'Books and educational materials');

INSERT INTO users (username, email, password_hash, first_name, last_name, role) VALUES
    ('admin', 'admin@example.com', '$2a$12$placeholder', 'Admin', 'User', 'ADMIN');
```

---

## ขั้นตอนที่ 269: Liquibase (Alternative to Flyway)

```xml
<dependency>
    <groupId>org.liquibase</groupId>
    <artifactId>liquibase-core</artifactId>
</dependency>
```

```yaml
# application.yml
spring:
  liquibase:
    change-log: classpath:db/changelog/db.changelog-master.yaml
```

```yaml
# src/main/resources/db/changelog/db.changelog-master.yaml
databaseChangeLog:
  - include:
      file: db/changelog/changes/001-create-users.yaml
  - include:
      file: db/changelog/changes/002-create-products.yaml
```

```yaml
# 001-create-users.yaml
databaseChangeLog:
  - changeSet:
      id: 001
      author: developer
      changes:
        - createTable:
            tableName: users
            columns:
              - column:
                  name: id
                  type: BIGINT
                  autoIncrement: true
                  constraints:
                    primaryKey: true
              - column:
                  name: username
                  type: VARCHAR(50)
                  constraints:
                    nullable: false
                    unique: true
              - column:
                  name: email
                  type: VARCHAR(255)
                  constraints:
                    nullable: false
                    unique: true
```

---

## ขั้นตอนที่ 270: MySQL Setup

```yaml
# docker-compose.yml (MySQL)
services:
  mysql:
    image: mysql:8.0
    container_name: spring_mysql
    environment:
      MYSQL_ROOT_PASSWORD: rootpass
      MYSQL_DATABASE: springdb
      MYSQL_USER: springuser
      MYSQL_PASSWORD: springpass
    ports:
      - "3306:3306"
    volumes:
      - mysql_data:/var/lib/mysql
    command: --default-authentication-plugin=mysql_native_password
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 10s
      timeout: 5s
      retries: 5
```

```xml
<!-- pom.xml MySQL Connector -->
<dependency>
    <groupId>com.mysql</groupId>
    <artifactId>mysql-connector-j</artifactId>
    <scope>runtime</scope>
</dependency>
```

```yaml
# application.yml สำหรับ MySQL
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/springdb?useSSL=false&serverTimezone=UTC&allowPublicKeyRetrieval=true
    username: springuser
    password: springpass
    driver-class-name: com.mysql.cj.jdbc.Driver

  jpa:
    properties:
      hibernate:
        dialect: org.hibernate.dialect.MySQLDialect
```

---

## ขั้นตอนที่ 271: Multiple Datasources

```java
@Configuration
public class DataSourceConfig {
    
    @Bean
    @Primary
    @ConfigurationProperties("spring.datasource.primary")
    public DataSource primaryDataSource() {
        return DataSourceBuilder.create().build();
    }
    
    @Bean
    @ConfigurationProperties("spring.datasource.secondary")
    public DataSource secondaryDataSource() {
        return DataSourceBuilder.create().build();
    }
}
```

```yaml
spring:
  datasource:
    primary:
      url: jdbc:postgresql://localhost:5432/primary_db
      username: user1
      password: pass1
    secondary:
      url: jdbc:postgresql://localhost:5432/analytics_db
      username: user2
      password: pass2
```

---

## ขั้นตอนที่ 272: Connection Pool Monitoring

```java
// Monitor HikariCP pool
@Component
@RequiredArgsConstructor
@Slf4j
public class DataSourceHealthChecker {
    
    private final DataSource dataSource;
    
    @Scheduled(fixedDelay = 60000)  // ทุก 1 นาที
    public void logPoolStats() {
        if (dataSource instanceof HikariDataSource hikari) {
            HikariPoolMXBean pool = hikari.getHikariPoolMXBean();
            log.debug("DB Pool: active={}, idle={}, waiting={}, total={}",
                pool.getActiveConnections(),
                pool.getIdleConnections(),
                pool.getThreadsAwaitingConnection(),
                pool.getTotalConnections()
            );
        }
    }
}
```

---

## ขั้นตอนที่ 273: Database Configuration สำหรับ Profile ต่างๆ

```yaml
# application-dev.yml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/springdb_dev
  jpa:
    hibernate.ddl-auto: create-drop  # dev: สร้าง schema ใหม่ทุก restart
    show-sql: true

# application-test.yml
spring:
  datasource:
    url: jdbc:h2:mem:testdb
    driver-class-name: org.h2.Driver
  jpa:
    hibernate.ddl-auto: create-drop

# application-prod.yml
spring:
  datasource:
    url: ${DATABASE_URL}
    username: ${DATABASE_USERNAME}
    password: ${DATABASE_PASSWORD}
    hikari:
      maximum-pool-size: 50
  jpa:
    hibernate.ddl-auto: validate  # production: validate เท่านั้น
    show-sql: false
```

---

## ขั้นตอนที่ 274-295: Full Database Best Practices

### Soft Delete Pattern

```java
@Entity
@Where(clause = "deleted_at IS NULL")  // Hibernate filter
@SQLDelete(sql = "UPDATE products SET deleted_at = NOW() WHERE id = ?")
public class Product extends BaseEntity {
    
    @Column(name = "deleted_at")
    private LocalDateTime deletedAt;
}

// Repository
public interface ProductRepository extends JpaRepository<Product, Long> {
    // findAll() จะ filter deleted_at IS NULL อัตโนมัติ
    
    // หา deleted records
    @Query(value = "SELECT * FROM products WHERE deleted_at IS NOT NULL", nativeQuery = true)
    List<Product> findDeleted();
}

// Service
@Transactional
public void delete(Long id) {
    productRepository.deleteById(id);  // เรียก SQL DELETE แต่ @SQLDelete จะ soft delete
}
```

### Optimistic Locking

```java
@Entity
public class Product extends BaseEntity {
    
    @Version
    private Long version;  // JPA จะ check version เมื่อ update
}

// เมื่อ 2 users update พร้อมกัน:
// User A read version=1, User B read version=1
// User A save version=2 (success)
// User B save version=1 (fail! OptimisticLockException)
```

### Generated Schema

```bash
# Generate DDL ออกมาดู (ไม่ execute)
# application-schema.yml
spring:
  jpa:
    properties:
      javax.persistence.schema-generation.scripts.action: create
      javax.persistence.schema-generation.scripts.create-target: create-schema.sql
```

---

*[← Part 11: Spring Data JPA](./part-11-spring-data-jpa.md) | [Part 13: CRUD Operations →](./part-13-crud-operations.md)*

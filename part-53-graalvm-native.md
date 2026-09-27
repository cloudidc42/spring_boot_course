# Part 53: Spring Boot with GraalVM Native Image
## ขั้นตอนที่ 1721-1760

**ระดับ:** Advanced  
**เวลาเรียน:** 4-5 ชั่วโมง  
**เป้าหมาย:** เรียนรู้การสร้าง Native Image ด้วย GraalVM เพื่อให้ Spring Boot application ใช้ memory น้อยลงและ start ไวขึ้นอย่างมาก

---

## ขั้นตอนที่ 1721: ทำความเข้าใจ GraalVM Native Image

**GraalVM Native Image** แปลง Java application เป็น native executable ที่:
- Start ใน **milliseconds** (ปกติ < 100ms) แทนที่จะเป็นหลายวินาที
- ใช้ **memory น้อยกว่า 70-80%** เทียบกับ JVM
- ไม่ต้องการ JVM ในการ run (standalone binary)
- เหมาะสำหรับ microservices, serverless, container

### เปรียบเทียบ JVM vs Native

| ด้าน | JVM Mode | Native Image |
|------|----------|--------------|
| Startup time | 3-10 วินาที | 50-200 ms |
| Memory (RSS) | 300-500 MB | 50-150 MB |
| Throughput (steady state) | สูงกว่า (JIT) | ต่ำกว่าเล็กน้อย |
| Build time | ไม่กี่วินาที | 3-10 นาที |
| Debugging | ง่าย | ยากกว่า |

### สิ่งที่ Native Image ไม่รองรับ (ตามค่าเริ่มต้น)

- Dynamic class loading หลัง build time
- Reflection ที่ไม่ได้ register ไว้
- Dynamic proxy ที่ไม่ได้ register ไว้
- JNI แบบ dynamic

---

## ขั้นตอนที่ 1722: Setup Maven สำหรับ Native Build

```xml
<!-- pom.xml -->
<project>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.2.0</version>
    </parent>
    
    <properties>
        <java.version>21</java.version>
    </properties>
    
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jpa</artifactId>
        </dependency>
        <!-- Native Image support สำหรับ PostgreSQL -->
        <dependency>
            <groupId>org.postgresql</groupId>
            <artifactId>postgresql</artifactId>
        </dependency>
    </dependencies>
    
    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
            <!-- Native Build Tools Plugin -->
            <plugin>
                <groupId>org.graalvm.buildtools</groupId>
                <artifactId>native-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
    
    <profiles>
        <!-- Profile สำหรับ native build -->
        <profile>
            <id>native</id>
            <build>
                <plugins>
                    <plugin>
                        <groupId>org.graalvm.buildtools</groupId>
                        <artifactId>native-maven-plugin</artifactId>
                        <configuration>
                            <imageName>app-native</imageName>
                            <buildArgs>
                                <arg>--initialize-at-build-time=org.slf4j.LoggerFactory</arg>
                                <arg>-H:+ReportExceptionStackTraces</arg>
                                <arg>--no-fallback</arg>
                            </buildArgs>
                        </configuration>
                        <executions>
                            <execution>
                                <id>build-native</id>
                                <goals>
                                    <goal>compile-no-fork</goal>
                                </goals>
                                <phase>package</phase>
                            </execution>
                        </executions>
                    </plugin>
                </plugins>
            </build>
        </profile>
    </profiles>
</project>
```

---

## ขั้นตอนที่ 1723: AOT Processing ใน Spring Boot 3

Spring Boot 3 รองรับ **Ahead-of-Time (AOT) processing** ซึ่งเป็นก้าวสำคัญสู่ Native Image:

```java
// Application หลักไม่ต้องเปลี่ยน
@SpringBootApplication
public class NativeApplication {
    public static void main(String[] args) {
        SpringApplication.run(NativeApplication.class, args);
    }
}
```

AOT processing จะทำสิ่งเหล่านี้อัตโนมัติ:
- วิเคราะห์ Spring beans และสร้าง configuration แบบ static
- สร้าง proxy classes สำหรับ Spring features
- Register reflection hints สำหรับ classes ที่ใช้
- สร้าง resource hints สำหรับ files ที่ต้องการ

### ทดสอบ AOT ก่อน build native

```bash
# Run application ด้วย AOT mode (ตรวจสอบปัญหาก่อน build จริง)
./mvnw spring-boot:process-aot
./mvnw spring-boot:run -Dspring-boot.run.arguments="--spring.aot.enabled=true"
```

---

## ขั้นตอนที่ 1724: Reflection Hints

Reflection ที่ใช้ใน runtime ต้อง register ไว้ล่วงหน้า:

```java
// src/main/java/com/example/hints/AppRuntimeHints.java
package com.example.hints;

import com.example.dto.response.ProductResponse;
import com.example.dto.response.UserResponse;
import com.example.domain.entity.Product;
import com.example.domain.entity.User;
import org.springframework.aot.hint.MemberCategory;
import org.springframework.aot.hint.RuntimeHints;
import org.springframework.aot.hint.RuntimeHintsRegistrar;
import org.springframework.context.annotation.ImportRuntimeHints;
import org.springframework.stereotype.Component;

/**
 * Register runtime hints สำหรับ classes ที่ใช้ reflection
 * เช่น classes ที่ถูก serialize/deserialize ด้วย Jackson
 */
@Component
@ImportRuntimeHints(AppRuntimeHints.class)
public class AppRuntimeHints implements RuntimeHintsRegistrar {
    
    @Override
    public void registerHints(RuntimeHints hints, ClassLoader classLoader) {
        // Register reflection สำหรับ entity classes
        hints.reflection()
            .registerType(Product.class, 
                MemberCategory.INVOKE_PUBLIC_CONSTRUCTORS,
                MemberCategory.INVOKE_PUBLIC_METHODS,
                MemberCategory.DECLARED_FIELDS)
            .registerType(User.class,
                MemberCategory.INVOKE_PUBLIC_CONSTRUCTORS,
                MemberCategory.INVOKE_PUBLIC_METHODS,
                MemberCategory.DECLARED_FIELDS);
        
        // Register reflection สำหรับ DTO classes
        hints.reflection()
            .registerType(ProductResponse.class,
                MemberCategory.INVOKE_PUBLIC_CONSTRUCTORS,
                MemberCategory.INVOKE_PUBLIC_METHODS,
                MemberCategory.DECLARED_FIELDS)
            .registerType(UserResponse.class,
                MemberCategory.INVOKE_PUBLIC_CONSTRUCTORS,
                MemberCategory.INVOKE_PUBLIC_METHODS,
                MemberCategory.DECLARED_FIELDS);
        
        // Register resources ที่ต้องการ
        hints.resources()
            .registerPattern("db/migration/*.sql")
            .registerPattern("messages/*.properties")
            .registerPattern("static/**");
        
        // Register proxies ถ้าใช้ JDK dynamic proxy
        hints.proxies()
            .registerJdkProxy(
                com.example.domain.repository.ProductRepository.class
            );
    }
}
```

---

## ขั้นตอนที่ 1725: @RegisterReflectionForBinding

Annotation ที่ง่ายกว่าสำหรับ serialization/deserialization:

```java
// src/main/java/com/example/NativeApplication.java
package com.example;

import com.example.dto.request.*;
import com.example.dto.response.*;
import org.springframework.aot.hint.annotation.RegisterReflectionForBinding;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

// Register ทุก DTO classes ที่ต้องการ JSON binding
@RegisterReflectionForBinding({
    // Request DTOs
    LoginRequest.class,
    RegisterRequest.class,
    ProductRequest.class,
    CartItemRequest.class,
    CreateOrderRequest.class,
    
    // Response DTOs
    AuthResponse.class,
    ProductResponse.class,
    CartResponse.class,
    OrderResponse.class,
    ErrorResponse.class,
    PageResponse.class
})
@SpringBootApplication
public class NativeApplication {
    public static void main(String[] args) {
        SpringApplication.run(NativeApplication.class, args);
    }
}
```

---

## ขั้นตอนที่ 1726: Handling Common Native Image Issues

### ปัญหาที่ 1: Reflection ใน Library

```java
// ปัญหา: Jackson ต้องการ reflection สำหรับ deserialization
// แก้ไข: เพิ่ม @JsonCreator และ @JsonProperty

import com.fasterxml.jackson.annotation.JsonCreator;
import com.fasterxml.jackson.annotation.JsonProperty;
import lombok.Value;

@Value
public class ProductRequest {
    String name;
    BigDecimal price;
    
    // ต้องมี @JsonCreator ใน Native Image
    @JsonCreator
    public ProductRequest(
            @JsonProperty("name") String name,
            @JsonProperty("price") BigDecimal price) {
        this.name = name;
        this.price = price;
    }
}
```

### ปัญหาที่ 2: Dynamic Proxy

```java
// src/main/java/com/example/hints/ProxyHints.java
package com.example.hints;

import org.springframework.aot.hint.RuntimeHints;
import org.springframework.aot.hint.RuntimeHintsRegistrar;

public class ProxyHints implements RuntimeHintsRegistrar {
    
    @Override
    public void registerHints(RuntimeHints hints, ClassLoader classLoader) {
        // Register JDK proxy สำหรับ Repository interfaces
        hints.proxies()
            .registerJdkProxy(
                org.springframework.data.repository.Repository.class,
                org.springframework.data.jpa.repository.JpaRepository.class
            );
        
        // Register proxy chain ที่ซับซ้อน
        hints.proxies()
            .registerJdkProxy(
                com.example.service.PaymentService.class,
                org.springframework.transaction.interceptor.TransactionalProxy.class
            );
    }
}
```

### ปัญหาที่ 3: Resource Files

```java
// Register resource patterns
hints.resources()
    .registerPattern("*.properties")
    .registerPattern("config/**")
    .registerResourceBundle("messages");

// Register specific files
hints.resources()
    .registerPattern("META-INF/spring/*.imports")
    .registerPattern("META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports");
```

---

## ขั้นตอนที่ 1727: Serialization Hints

```java
// src/main/java/com/example/hints/SerializationHints.java
package com.example.hints;

import org.springframework.aot.hint.RuntimeHints;
import org.springframework.aot.hint.RuntimeHintsRegistrar;

import java.util.List;
import java.util.Map;

public class SerializationHints implements RuntimeHintsRegistrar {
    
    @Override
    public void registerHints(RuntimeHints hints, ClassLoader classLoader) {
        // Register Java standard library classes ที่ต้องการ serialization
        hints.serialization()
            .registerType(java.util.ArrayList.class)
            .registerType(java.util.HashMap.class)
            .registerType(java.math.BigDecimal.class)
            .registerType(java.time.LocalDateTime.class)
            .registerType(java.util.UUID.class);
        
        // Register custom classes สำหรับ Java serialization
        hints.serialization()
            .registerType(com.example.domain.entity.User.class)
            .registerType(com.example.domain.entity.Product.class);
    }
}
```

---

## ขั้นตอนที่ 1728: Native Configuration Files

สร้าง hint files ด้วยตัวเองเมื่อ automatic hints ไม่พอ:

```json
// src/main/resources/META-INF/native-image/reflect-config.json
[
  {
    "name": "com.example.domain.entity.Product",
    "allDeclaredConstructors": true,
    "allPublicConstructors": true,
    "allDeclaredMethods": true,
    "allPublicMethods": true,
    "allDeclaredFields": true,
    "allPublicFields": true
  },
  {
    "name": "com.example.dto.response.ProductResponse",
    "allDeclaredConstructors": true,
    "allPublicConstructors": true,
    "allDeclaredMethods": true,
    "allPublicMethods": true,
    "allDeclaredFields": true
  }
]
```

```json
// src/main/resources/META-INF/native-image/resource-config.json
{
  "resources": {
    "includes": [
      {"pattern": "\\Qapplication.yml\\E"},
      {"pattern": "\\Qapplication-prod.yml\\E"},
      {"pattern": "db/migration/.*\\.sql"},
      {"pattern": "static/.*"},
      {"pattern": "templates/.*\\.html"}
    ]
  },
  "bundles": [
    {"name": "messages"}
  ]
}
```

```json
// src/main/resources/META-INF/native-image/proxy-config.json
[
  {
    "interfaces": [
      "com.example.domain.repository.ProductRepository",
      "org.springframework.data.repository.Repository"
    ]
  }
]
```

---

## ขั้นตอนที่ 1729: Build Native Image

```bash
# วิธีที่ 1: Build ด้วย Maven (ต้องติดตั้ง GraalVM JDK)
./mvnw -Pnative native:compile

# วิธีที่ 2: Build ผ่าน Spring Boot plugin
./mvnw spring-boot:build-image -Pnative

# วิธีที่ 3: ใช้ Buildpacks (ไม่ต้องติดตั้ง GraalVM local)
./mvnw spring-boot:build-image \
  -Dspring-boot.build-image.imageName=ecommerce-api:native

# ดู build options
./mvnw help:describe -Dplugin=org.graalvm.buildtools:native-maven-plugin -Ddetail
```

### ติดตั้ง GraalVM

```bash
# ติดตั้งผ่าน SDKMAN (แนะนำ)
sdk install java 21.0.2-graalce
sdk use java 21.0.2-graalce

# ตรวจสอบ
java -version
# Output: OpenJDK Runtime Environment GraalVM CE ...

# ติดตั้ง native-image tool
gu install native-image
native-image --version
```

---

## ขั้นตอนที่ 1730: Docker Multi-Stage Build สำหรับ Native Image

```dockerfile
# Dockerfile.native
# Stage 1: Build stage ด้วย GraalVM
FROM ghcr.io/graalvm/native-image-community:21 AS builder

WORKDIR /app

# Copy Maven wrapper และ pom.xml ก่อน (cache layer)
COPY .mvn/ .mvn/
COPY mvnw pom.xml ./

# Download dependencies (cache layer ถ้า pom.xml ไม่เปลี่ยน)
RUN ./mvnw dependency:go-offline -q

# Copy source code
COPY src ./src

# Build native image
RUN ./mvnw -Pnative native:compile -DskipTests

# Stage 2: Runtime stage (tiny image)
FROM debian:bookworm-slim

WORKDIR /app

# Copy native executable
COPY --from=builder /app/target/app-native ./app

# Non-root user สำหรับ security
RUN addgroup --system --gid 1001 appgroup && \
    adduser --system --uid 1001 --gid 1001 appuser
USER appuser

EXPOSE 8080

ENTRYPOINT ["./app"]
```

```dockerfile
# Dockerfile.native-distroless (เล็กที่สุด)
FROM ghcr.io/graalvm/native-image-community:21 AS builder
WORKDIR /app
COPY . .
RUN ./mvnw -Pnative native:compile -DskipTests

# Distroless: ไม่มี shell, ไม่มี package manager = secure มาก
FROM gcr.io/distroless/base-debian12
COPY --from=builder /app/target/app-native /app
EXPOSE 8080
ENTRYPOINT ["/app"]
```

```yaml
# docker-compose.native.yml
version: '3.9'

services:
  app-native:
    build:
      context: .
      dockerfile: Dockerfile.native
    container_name: ecommerce-native
    ports:
      - "8080:8080"
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/ecommerce_db
    depends_on:
      - postgres
    # เห็นความต่างของ startup time ชัดเจน
    healthcheck:
      test: ["CMD", "wget", "--spider", "-q", "http://localhost:8080/actuator/health"]
      start_period: 5s  # Native image start ไว ไม่ต้องรอนาน
      interval: 10s
```

---

## ขั้นตอนที่ 1731: Benchmark - Native vs JVM

```java
// src/test/java/com/example/benchmark/StartupBenchmarkTest.java
package com.example.benchmark;

import org.junit.jupiter.api.Test;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.web.client.TestRestTemplate;
import org.springframework.boot.test.web.server.LocalServerPort;

/**
 * ทดสอบ startup time (ดู log เพื่อเปรียบเทียบ)
 * Native: "Started Application in 0.089 seconds"
 * JVM:    "Started Application in 4.312 seconds"
 */
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class StartupBenchmarkTest {
    
    @LocalServerPort
    private int port;
    
    @Test
    void applicationShouldStartSuccessfully() {
        // แค่การ start ก็แสดง startup time แล้ว
        System.out.println("Application started on port: " + port);
    }
}
```

### Script เปรียบเทียบ Memory

```bash
#!/bin/bash
# scripts/benchmark-memory.sh

echo "=== JVM Mode ==="
java -jar target/app.jar &
JVM_PID=$!
sleep 10
JVM_MEM=$(cat /proc/$JVM_PID/status | grep VmRSS | awk '{print $2}')
echo "JVM RSS Memory: ${JVM_MEM} kB"
kill $JVM_PID

echo ""
echo "=== Native Mode ==="
./target/app-native &
NATIVE_PID=$!
sleep 2
NATIVE_MEM=$(cat /proc/$NATIVE_PID/status | grep VmRSS | awk '{print $2}')
echo "Native RSS Memory: ${NATIVE_MEM} kB"
kill $NATIVE_PID

echo ""
echo "=== Results ==="
RATIO=$(echo "scale=2; $JVM_MEM / $NATIVE_MEM" | bc)
echo "JVM uses ${RATIO}x more memory than Native"
```

### ผลลัพธ์ที่คาดหวัง

```
=== Typical Benchmark Results ===

Application: Spring Boot E-Commerce API

JVM Mode:
  Startup time: 4.3 seconds
  RSS Memory: 380 MB
  First request latency: 250ms (JIT warmup)
  Steady-state throughput: 15,000 req/s

Native Image:
  Startup time: 0.09 seconds  (48x faster!)
  RSS Memory: 78 MB            (79% less memory!)
  First request latency: 8ms   (no JIT warmup)
  Steady-state throughput: 13,000 req/s (13% less)

Conclusion: Native เหมาะสำหรับ:
  - Microservices ที่ต้อง scale quickly
  - Serverless functions (Lambda, Cloud Run)
  - CLI tools
  - Short-lived processes
  
JVM เหมาะกว่าสำหรับ:
  - Long-running services ที่ต้อง peak performance
  - Applications ที่มี complex dynamic behavior
```

---

## ขั้นตอนที่ 1732: Common Issues และ Solutions

### Issue 1: ClassNotFoundException ใน Native

```java
// ปัญหา: Dynamic class loading
Class<?> clazz = Class.forName("com.example.SomeClass");

// แก้ไข: Register ใน hints
hints.reflection()
    .registerType(SomeClass.class, MemberCategory.values());

// หรือใช้ constant string ที่ known ณ compile time
// Spring AOT จะตรวจจับ pattern นี้ได้อัตโนมัติ
```

### Issue 2: Missing Properties File

```java
// ปัญหา: Properties file ไม่ถูก include ใน native
// แก้ไข:
hints.resources()
    .registerPattern("custom.properties")
    .registerPattern("config/app.properties");
```

### Issue 3: Hibernate กับ Native

```java
// src/main/java/com/example/config/HibernateNativeConfig.java
package com.example.config;

import org.springframework.aot.hint.RuntimeHints;
import org.springframework.aot.hint.RuntimeHintsRegistrar;

public class HibernateNativeConfig implements RuntimeHintsRegistrar {
    
    @Override
    public void registerHints(RuntimeHints hints, ClassLoader classLoader) {
        // Hibernate ต้องการ reflection สำหรับ entity classes
        hints.resources()
            .registerPattern("hibernate.properties")
            .registerPattern("META-INF/persistence.xml");
        
        // Register dialect classes
        hints.reflection()
            .registerType(
                org.hibernate.dialect.PostgreSQLDialect.class,
                org.springframework.aot.hint.MemberCategory.INVOKE_PUBLIC_CONSTRUCTORS
            );
    }
}
```

### Issue 4: Logback/SLF4J

```java
// ปัญหา: Logback ใช้ reflection หนัก
// แก้ไขด้วย configuration:

// src/main/resources/logback-spring.xml
// ใช้ simple configuration สำหรับ native
```

```xml
<!-- src/main/resources/logback-native.xml -->
<configuration>
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>%d{HH:mm:ss} %-5level %logger{36} - %msg%n</pattern>
        </encoder>
    </appender>
    
    <root level="INFO">
        <appender-ref ref="CONSOLE"/>
    </root>
</configuration>
```

---

## ขั้นตอนที่ 1733: Tracing Agent สำหรับ Automatic Hints

ใช้ GraalVM Tracing Agent เพื่อ collect hints อัตโนมัติ:

```bash
# Step 1: Run application ด้วย tracing agent
java -agentlib:native-image-agent=config-output-dir=src/main/resources/META-INF/native-image \
     -jar target/app.jar

# Step 2: ทำ API calls ทุกๆ endpoint เพื่อ trace
curl http://localhost:8080/api/v1/products
curl -X POST http://localhost:8080/api/v1/auth/login \
     -H "Content-Type: application/json" \
     -d '{"username":"admin","password":"password"}'
# ... ทดสอบทุก feature

# Step 3: หยุด application
# Agent จะสร้างไฟล์:
# - reflect-config.json
# - proxy-config.json
# - resource-config.json
# - jni-config.json
# - serialization-config.json

# Step 4: Build native image โดยใช้ config ที่ได้
./mvnw -Pnative native:compile
```

---

## ขั้นตอนที่ 1734: Gradle Setup สำหรับ Native

```kotlin
// build.gradle.kts
plugins {
    id("org.springframework.boot") version "3.2.0"
    id("io.spring.dependency-management") version "1.1.4"
    id("org.graalvm.buildtools.native") version "0.9.28"
    kotlin("jvm") version "1.9.22"
    kotlin("plugin.spring") version "1.9.22"
    kotlin("plugin.jpa") version "1.9.22"
}

graalvmNative {
    binaries {
        named("main") {
            imageName.set("app-native")
            buildArgs.add("--initialize-at-build-time=org.slf4j.LoggerFactory")
            buildArgs.add("-H:+ReportExceptionStackTraces")
            buildArgs.add("--no-fallback")
            // เพิ่ม memory สำหรับ build (Native Image ใช้ memory เยอะ)
            jvmArgs.add("-Xmx6g")
        }
    }
}
```

```bash
# Build ด้วย Gradle
./gradlew nativeCompile

# Run native image
./build/native/nativeCompile/app-native

# Build Docker image
./gradlew bootBuildImage
```

---

## ขั้นตอนที่ 1735: CI/CD Pipeline สำหรับ Native Build

```yaml
# .github/workflows/native-build.yml
name: Build Native Image

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  build-native:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup GraalVM
        uses: graalvm/setup-graalvm@v1
        with:
          java-version: '21'
          distribution: 'graalvm-community'
          native-image: 'true'
      
      - name: Cache Maven packages
        uses: actions/cache@v3
        with:
          path: ~/.m2/repository
          key: ${{ runner.os }}-maven-${{ hashFiles('**/pom.xml') }}
      
      - name: Build Native Image
        run: ./mvnw -Pnative native:compile -DskipTests
        timeout-minutes: 30
      
      - name: Test Native Image
        run: |
          ./target/app-native &
          APP_PID=$!
          sleep 5
          curl -f http://localhost:8080/actuator/health
          kill $APP_PID
      
      - name: Measure startup time
        run: |
          START=$(date +%s%3N)
          ./target/app-native &
          APP_PID=$!
          until curl -sf http://localhost:8080/actuator/health > /dev/null; do
            sleep 0.1
          done
          END=$(date +%s%3N)
          echo "Startup time: $((END - START)) ms"
          kill $APP_PID
      
      - name: Build Docker Image
        run: |
          docker build -f Dockerfile.native -t ecommerce-native:${{ github.sha }} .
      
      - name: Upload native binary
        uses: actions/upload-artifact@v3
        with:
          name: native-binary
          path: target/app-native
```

---

## ขั้นตอนที่ 1736: Serverless ด้วย Native Image

```java
// src/main/java/com/example/function/ProductFunction.java
package com.example.function;

import com.example.service.ProductService;
import lombok.RequiredArgsConstructor;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.util.function.Function;

/**
 * Spring Cloud Function สำหรับ Serverless deployment
 * Native Image ทำให้ Cold start < 100ms (เหมาะมากสำหรับ Lambda)
 */
@Configuration
@RequiredArgsConstructor
public class ProductFunction {
    
    private final ProductService productService;
    
    @Bean
    public Function<Long, ProductResponse> getProduct() {
        return id -> productService.getProductById(id);
    }
    
    @Bean
    public Function<ProductFilterCriteria, java.util.List<ProductResponse>> searchProducts() {
        return criteria -> productService.searchProducts(criteria, 
            org.springframework.data.domain.Pageable.unpaged())
            .getContent();
    }
}
```

```yaml
# serverless.yml (AWS Lambda)
service: ecommerce-native-api

provider:
  name: aws
  runtime: provided.al2  # ต้องใช้ custom runtime สำหรับ native binary
  architecture: arm64    # Graviton2 - ประหยัดต้นทุน 20%

functions:
  getProduct:
    handler: com.example.function.ProductFunction::getProduct
    events:
      - http:
          path: /products/{id}
          method: get
    environment:
      SPRING_DATASOURCE_URL: !Sub "${DatabaseUrl}"
```

---

## สรุปท้ายบท

ในส่วนนี้เราได้เรียนรู้:

1. **GraalVM Native Image** คืออะไรและประโยชน์ที่ได้รับ
2. **AOT Processing** ใน Spring Boot 3
3. **Runtime Hints** สำหรับ reflection, resources, proxies
4. **Multi-stage Docker build** สำหรับ native image
5. **Benchmark** เปรียบเทียบ JVM vs Native
6. **Common issues** และวิธีแก้ไข
7. **Tracing Agent** สำหรับ automatic hint generation
8. **CI/CD Pipeline** สำหรับ native build

Native Image เป็น technology ที่น่าสนใจมาก โดยเฉพาะสำหรับ microservices และ serverless ที่ต้องการ startup time ที่ไวและ memory ที่น้อย

---

*[← Part 52: Database Migration](./part-52-database-migration.md) | [Part 54: Virtual Threads →](./part-54-virtual-threads.md)*

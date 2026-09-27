# Part 72: Contract Testing - Consumer-Driven Contract Testing
## ขั้นตอนที่ 2481-2520

**ระดับ:** ระดับโลก (World-Class)
**เวลาเรียน:** 4-5 ชั่วโมง
**เป้าหมาย:** เรียนรู้การทำ Consumer-Driven Contract Testing ด้วย Spring Cloud Contract และ Pact เพื่อให้ทีมต่างๆ พัฒนา microservices ได้อิสระโดยไม่ทำให้ integration พัง

---

## สารบัญ

1. [Consumer-Driven Contract Testing คืออะไร](#concept)
2. [Spring Cloud Contract: Writing Contracts](#spring-cloud-contract)
3. [Auto-Generated Stubs](#stubs)
4. [Running Contract Tests in CI/CD](#cicd)
5. [Pact Framework](#pact)
6. [Contract Evolution and Versioning](#versioning)
7. [Breaking Changes Detection](#breaking-changes)

---

## ขั้นตอนที่ 2481: Consumer-Driven Contract Testing คืออะไร? {#concept}

### ปัญหาที่ Contract Testing แก้

ในระบบ Microservices เราต้องการให้แต่ละ service develop และ deploy ได้อิสระ แต่ปัญหาคือเมื่อ Producer (API provider) เปลี่ยน API โดยไม่แจ้ง Consumer (API caller) ก็จะพัง

**แบบดั้งเดิม (Integration Testing):**
```
Consumer Service → [Integration Test Environment] → Producer Service
                   ← ต้องรอ Producer พร้อม →
```

**แบบ Contract Testing:**
```
Consumer Service → [Contract] → Producer Service
     ↓                              ↓
  Consumer Test               Provider Verification
  (ทดสอบ stub)                 (ทดสอบจาก contract)
```

### ประเภทของ Contract Testing

1. **Consumer-Driven (CDCT)** - Consumer เป็นผู้กำหนด contract (แนะนำ)
2. **Provider-Driven** - Provider กำหนด contract แล้ว consumer ต้องตาม
3. **Bidirectional** - ทั้งสองฝ่ายช่วยกันกำหนด

---

## ขั้นตอนที่ 2482: Spring Cloud Contract Setup

### Maven Dependencies

```xml
<!-- Producer pom.xml -->
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-contract-verifier</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>

<build>
    <plugins>
        <plugin>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-contract-maven-plugin</artifactId>
            <extensions>true</extensions>
            <configuration>
                <!-- Base class สำหรับ generated tests -->
                <baseClassForTests>
                    com.example.producer.BaseContractTest
                </baseClassForTests>
                <!-- ไดเรกทอรีที่เก็บ contracts -->
                <contractsDirectory>
                    ${project.basedir}/src/test/resources/contracts
                </contractsDirectory>
            </configuration>
        </plugin>
    </plugins>
</build>
```

```xml
<!-- Consumer pom.xml -->
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-contract-stub-runner</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

---

## ขั้นตอนที่ 2483: Writing Contracts in Groovy DSL {#spring-cloud-contract}

### โครงสร้างไดเรกทอรี

```
producer-service/
├── src/
│   ├── main/java/...
│   └── test/
│       ├── java/
│       │   └── com/example/producer/
│       │       └── BaseContractTest.java
│       └── resources/
│           └── contracts/
│               ├── orders/
│               │   ├── get_order_by_id.groovy
│               │   ├── create_order.groovy
│               │   └── order_not_found.groovy
│               └── products/
│                   └── get_product.groovy
```

### Contract: Get Order by ID

```groovy
// src/test/resources/contracts/orders/get_order_by_id.groovy
package contracts.orders

import org.springframework.cloud.contract.spec.Contract

Contract.make {
    description "Should return order when valid ID is provided"
    
    request {
        method GET()
        url "/api/orders/1"
        headers {
            contentType(applicationJson())
            header("Authorization", "Bearer valid-token")
        }
    }
    
    response {
        status 200
        headers {
            contentType(applicationJson())
        }
        body(
            id: 1,
            customerId: "CUST-001",
            status: "CONFIRMED",
            totalAmount: 1500.00,
            items: [
                [
                    productId: "PROD-001",
                    productName: "Spring Boot Book",
                    quantity: 2,
                    price: 750.00
                ]
            ],
            createdAt: $(producer(regex('[0-9]{4}-[0-9]{2}-[0-9]{2}T[0-9]{2}:[0-9]{2}:[0-9]{2}.*')),
                        consumer('2024-01-15T10:00:00Z'))
        )
    }
}
```

### Contract: Create Order

```groovy
// src/test/resources/contracts/orders/create_order.groovy
package contracts.orders

import org.springframework.cloud.contract.spec.Contract

Contract.make {
    description "Should create order and return created order with ID"
    
    request {
        method POST()
        url "/api/orders"
        headers {
            contentType(applicationJson())
            header("Authorization", "Bearer valid-token")
        }
        body(
            customerId: "CUST-001",
            items: [
                [
                    productId: "PROD-001",
                    quantity: 2
                ]
            ],
            shippingAddress: [
                street: "123 Silom Road",
                city: "Bangkok",
                postalCode: "10500"
            ]
        )
        bodyMatchers {
            jsonPath('$.customerId', byRegex('[A-Z]+-[0-9]+'))
            jsonPath('$.items', byType {
                minOccurrence(1)
            })
        }
    }
    
    response {
        status 201
        headers {
            contentType(applicationJson())
        }
        body(
            // $(producer(...), consumer(...)) = ค่าที่ producer ใช้จริง, ค่าที่ consumer expect
            id: $(producer(regex('[0-9]+')), consumer(100)),
            customerId: fromRequest().body('$.customerId'),
            status: "PENDING",
            totalAmount: $(producer(regex('[0-9]+\\.[0-9]+')), consumer(1500.00)),
            createdAt: $(producer(regex('.*')), consumer('2024-01-15T10:00:00Z'))
        )
    }
}
```

### Contract: Order Not Found

```groovy
// src/test/resources/contracts/orders/order_not_found.groovy
package contracts.orders

import org.springframework.cloud.contract.spec.Contract

Contract.make {
    description "Should return 404 when order does not exist"
    
    request {
        method GET()
        url "/api/orders/99999"
        headers {
            header("Authorization", "Bearer valid-token")
        }
    }
    
    response {
        status 404
        headers {
            contentType(applicationJson())
        }
        body(
            error: "ORDER_NOT_FOUND",
            message: $(producer(regex('.*')), consumer("Order not found: 99999")),
            timestamp: $(producer(regex('.*')), consumer('2024-01-15T10:00:00Z'))
        )
    }
}
```

### YAML Contract Alternative

```yaml
# src/test/resources/contracts/orders/get_all_orders.yml
description: "Should return paginated list of orders"
request:
  method: GET
  url: /api/orders
  queryParameters:
    page: 0
    size: 10
  headers:
    Authorization: "Bearer valid-token"
response:
  status: 200
  headers:
    Content-Type: application/json
  body:
    content:
      - id: 1
        customerId: "CUST-001"
        status: "CONFIRMED"
        totalAmount: 1500.00
      - id: 2
        customerId: "CUST-002"
        status: "PENDING"
        totalAmount: 500.00
    pageable:
      pageNumber: 0
      pageSize: 10
    totalElements: 2
    totalPages: 1
  matchers:
    body:
      - path: $.content[*].id
        type: by_regex
        value: "[0-9]+"
      - path: $.content[*].totalAmount
        type: by_regex
        value: "[0-9]+\\.[0-9]+"
```

---

## ขั้นตอนที่ 2485: Base Test Class สำหรับ Producer

```java
package com.example.producer;

import com.example.producer.controller.OrderController;
import com.example.producer.service.OrderService;
import io.restassured.module.mockmvc.RestAssuredMockMvc;
import org.junit.jupiter.api.BeforeEach;
import org.mockito.Mockito;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.mock.mockito.MockBean;
import org.springframework.web.context.WebApplicationContext;

import java.math.BigDecimal;
import java.time.Instant;
import java.util.List;
import java.util.Optional;

import static org.mockito.ArgumentMatchers.*;

// Base class ที่ generated tests จะ extend
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.MOCK)
public abstract class BaseContractTest {

    @Autowired
    private WebApplicationContext context;

    @MockBean
    private OrderService orderService;

    @BeforeEach
    public void setup() {
        // Setup RestAssured
        RestAssuredMockMvc.webAppContextSetup(context);
        
        // Mock data ที่ตรงกับ contract
        Order mockOrder = Order.builder()
            .id(1L)
            .customerId("CUST-001")
            .status(OrderStatus.CONFIRMED)
            .totalAmount(new BigDecimal("1500.00"))
            .items(List.of(
                OrderItem.builder()
                    .productId("PROD-001")
                    .productName("Spring Boot Book")
                    .quantity(2)
                    .price(new BigDecimal("750.00"))
                    .build()
            ))
            .createdAt(Instant.parse("2024-01-15T10:00:00Z"))
            .build();

        // Stub สำหรับ get order by id
        Mockito.when(orderService.findById(1L))
            .thenReturn(Optional.of(mockOrder));
        
        Mockito.when(orderService.findById(99999L))
            .thenReturn(Optional.empty());

        // Stub สำหรับ create order
        Mockito.when(orderService.createOrder(any(CreateOrderRequest.class)))
            .thenReturn(Order.builder()
                .id(100L)
                .customerId("CUST-001")
                .status(OrderStatus.PENDING)
                .totalAmount(new BigDecimal("1500.00"))
                .createdAt(Instant.parse("2024-01-15T10:00:00Z"))
                .build());

        // Stub สำหรับ get all orders
        Mockito.when(orderService.findAll(any()))
            .thenReturn(new PageImpl<>(List.of(
                mockOrder,
                Order.builder()
                    .id(2L)
                    .customerId("CUST-002")
                    .status(OrderStatus.PENDING)
                    .totalAmount(new BigDecimal("500.00"))
                    .createdAt(Instant.now())
                    .build()
            )));
    }
}
```

---

## ขั้นตอนที่ 2488: Auto-Generated Stubs {#stubs}

เมื่อ Producer รัน `mvn install` Spring Cloud Contract จะ generate:
1. **Test classes** - สำหรับ verify ว่า producer ทำตาม contract
2. **Stub JAR** - สำหรับ consumer ใช้ในการ test

### Generated Test (Auto-Generated โดย Plugin)

```java
// Target/generated-test-sources/contracts/com/example/producer/ContractVerifierTest.java
// นี่คือโค้ดที่ generate โดยอัตโนมัติ - ไม่ต้องเขียนเอง
public class OrdersTest extends BaseContractTest {

    @Test
    public void validate_get_order_by_id() throws Exception {
        // given:
        MockMvcRequestSpecification request = given()
            .header("Content-Type", "application/json")
            .header("Authorization", "Bearer valid-token");

        // when:
        ResponseOptions response = given().spec(request)
            .get("/api/orders/1");

        // then:
        assertThat(response.statusCode()).isEqualTo(200);
        assertThat(response.header("Content-Type"))
            .matches("application/json.*");
        
        // and:
        DocumentContext parsedJson = JsonPath.parse(response.getBody().asString());
        assertThatJson(parsedJson).field("['id']").isEqualTo(1);
        assertThatJson(parsedJson).field("['customerId']").isEqualTo("CUST-001");
        assertThatJson(parsedJson).field("['status']").isEqualTo("CONFIRMED");
        assertThatJson(parsedJson).field("['totalAmount']").isEqualTo(1500.0);
    }
}
```

### Consumer Test ด้วย Stub Runner

```java
package com.example.consumer;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.cloud.contract.stubrunner.spring.AutoConfigureStubRunner;
import org.springframework.cloud.contract.stubrunner.spring.StubRunnerProperties;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest
@AutoConfigureStubRunner(
    // Stub จาก Maven repository หรือ local
    ids = "com.example:order-service:+:stubs:8090",
    stubsMode = StubRunnerProperties.StubsMode.LOCAL
)
class OrderClientContractTest {

    @Autowired
    private OrderClient orderClient;  // Feign client หรือ RestTemplate

    @Test
    void shouldGetOrderById() {
        // stub จะถูก start บน port 8090 โดยอัตโนมัติ
        Order order = orderClient.getOrder(1L);
        
        assertThat(order.getId()).isEqualTo(1L);
        assertThat(order.getCustomerId()).isEqualTo("CUST-001");
        assertThat(order.getStatus()).isEqualTo(OrderStatus.CONFIRMED);
        assertThat(order.getTotalAmount()).isEqualByComparingTo("1500.00");
    }

    @Test
    void shouldHandleOrderNotFound() {
        assertThatThrownBy(() -> orderClient.getOrder(99999L))
            .isInstanceOf(OrderNotFoundException.class)
            .hasMessageContaining("ORDER_NOT_FOUND");
    }

    @Test
    void shouldCreateOrder() {
        CreateOrderRequest request = CreateOrderRequest.builder()
            .customerId("CUST-001")
            .items(List.of(new OrderItem("PROD-001", 2)))
            .build();

        Order created = orderClient.createOrder(request);
        
        assertThat(created.getId()).isNotNull();
        assertThat(created.getStatus()).isEqualTo(OrderStatus.PENDING);
    }
}
```

### Feign Client (Consumer)

```java
package com.example.consumer.client;

import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.*;

// ใช้ url จาก property ได้ หรือ hardcode สำหรับ test
@FeignClient(
    name = "order-service",
    url = "${order.service.url:http://localhost:8090}"
)
public interface OrderClient {

    @GetMapping("/api/orders/{id}")
    Order getOrder(@PathVariable Long id);

    @PostMapping("/api/orders")
    Order createOrder(@RequestBody CreateOrderRequest request);

    @GetMapping("/api/orders")
    Page<Order> getAllOrders(Pageable pageable);
}
```

---

## ขั้นตอนที่ 2492: Running Contract Tests in CI/CD {#cicd}

### GitHub Actions Pipeline

```yaml
# .github/workflows/contract-tests.yml
name: Contract Tests

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  producer-contract-test:
    name: Producer - Verify Contracts
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up JDK 21
        uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
          cache: 'maven'

      - name: Run contract tests (Producer)
        working-directory: ./order-service
        run: mvn test -pl . -Dtest="*ContractTest*"

      - name: Publish stubs to Maven local
        working-directory: ./order-service
        run: mvn install -DskipTests=false

      - name: Upload stubs artifact
        uses: actions/upload-artifact@v4
        with:
          name: order-service-stubs
          path: order-service/target/*.jar

  consumer-contract-test:
    name: Consumer - Test Against Stubs
    runs-on: ubuntu-latest
    needs: producer-contract-test
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up JDK 21
        uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
          cache: 'maven'

      - name: Download stubs
        uses: actions/download-artifact@v4
        with:
          name: order-service-stubs
          path: ~/.m2/repository/com/example/order-service

      - name: Run consumer contract tests
        working-directory: ./consumer-service
        run: mvn test -Dtest="*ContractTest*"
```

### Makefile สำหรับ Local Development

```makefile
# Makefile
.PHONY: contract-test-producer contract-test-consumer publish-stubs

# Producer: verify contracts + publish stubs
contract-test-producer:
	cd order-service && mvn spring-cloud-contract:generateStubs spring-cloud-contract:pushStubsToScm

# Consumer: run tests against stubs
contract-test-consumer:
	cd consumer-service && mvn test -Dtest="*ContractTest*"

# Publish stubs to local maven repo
publish-stubs:
	cd order-service && mvn install -DskipTests

# Run everything
contract-test-all: publish-stubs contract-test-consumer
```

---

## ขั้นตอนที่ 2496: Pact Framework Alternative {#pact}

Pact เป็น framework ที่ใช้งานได้ใน multiple languages และมี Pact Broker สำหรับเก็บ contracts

### Pact Dependencies

```xml
<!-- Consumer pom.xml -->
<dependency>
    <groupId>au.com.dius.pact.consumer</groupId>
    <artifactId>junit5</artifactId>
    <version>4.6.7</version>
    <scope>test</scope>
</dependency>

<!-- Producer pom.xml -->
<dependency>
    <groupId>au.com.dius.pact.provider</groupId>
    <artifactId>junit5spring</artifactId>
    <version>4.6.7</version>
    <scope>test</scope>
</dependency>
```

### Pact Consumer Test

```java
package com.example.consumer;

import au.com.dius.pact.consumer.dsl.PactDslWithProvider;
import au.com.dius.pact.consumer.junit5.PactConsumerTestExt;
import au.com.dius.pact.consumer.junit5.PactTestFor;
import au.com.dius.pact.core.model.RequestResponsePact;
import au.com.dius.pact.core.model.annotations.Pact;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.springframework.web.client.RestTemplate;

import static au.com.dius.pact.consumer.dsl.LambdaDsl.newJsonBody;
import static org.assertj.core.api.Assertions.assertThat;

@ExtendWith(PactConsumerTestExt.class)
@PactTestFor(providerName = "order-service", port = "8080")
class OrderClientPactTest {

    // กำหนด contract ใน code
    @Pact(consumer = "notification-service")
    public RequestResponsePact getOrderPact(PactDslWithProvider builder) {
        return builder
            .given("order with id 1 exists")
            .uponReceiving("a request to get order 1")
                .path("/api/orders/1")
                .method("GET")
                .headers("Authorization", "Bearer valid-token")
            .willRespondWith()
                .status(200)
                .headers(Map.of("Content-Type", "application/json"))
                .body(newJsonBody(body -> {
                    body.numberType("id", 1);
                    body.stringType("customerId", "CUST-001");
                    body.stringMatcher("status", "CONFIRMED|PENDING|CANCELLED", "CONFIRMED");
                    body.decimalType("totalAmount", 1500.00);
                    body.array("items", items -> {
                        items.object(item -> {
                            item.stringType("productId", "PROD-001");
                            item.numberType("quantity", 2);
                            item.decimalType("price", 750.00);
                        });
                    });
                }).build())
            .toPact();
    }

    @Pact(consumer = "notification-service")
    public RequestResponsePact orderNotFoundPact(PactDslWithProvider builder) {
        return builder
            .given("order with id 99999 does not exist")
            .uponReceiving("a request for non-existent order")
                .path("/api/orders/99999")
                .method("GET")
            .willRespondWith()
                .status(404)
                .body(newJsonBody(body -> {
                    body.stringType("error", "ORDER_NOT_FOUND");
                    body.stringType("message");
                }).build())
            .toPact();
    }

    @Test
    @PactTestFor(pactMethod = "getOrderPact")
    void shouldGetOrder(MockServer mockServer) {
        // RestTemplate ชี้ไปที่ mock server
        RestTemplate restTemplate = new RestTemplate();
        String url = mockServer.getUrl() + "/api/orders/1";
        
        // เพิ่ม Authorization header
        HttpHeaders headers = new HttpHeaders();
        headers.set("Authorization", "Bearer valid-token");
        HttpEntity<?> entity = new HttpEntity<>(headers);
        
        ResponseEntity<Order> response = restTemplate.exchange(
            url, HttpMethod.GET, entity, Order.class);
        
        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.OK);
        assertThat(response.getBody().getId()).isEqualTo(1L);
        assertThat(response.getBody().getStatus()).isEqualTo("CONFIRMED");
    }

    @Test
    @PactTestFor(pactMethod = "orderNotFoundPact")
    void shouldHandleOrderNotFound(MockServer mockServer) {
        RestTemplate restTemplate = new RestTemplate();
        
        assertThatThrownBy(() -> 
            restTemplate.getForObject(
                mockServer.getUrl() + "/api/orders/99999", 
                ErrorResponse.class)
        ).isInstanceOf(HttpClientErrorException.NotFound.class);
    }
}
```

### Pact Provider Verification

```java
package com.example.producer;

import au.com.dius.pact.provider.junit5.PactVerificationContext;
import au.com.dius.pact.provider.junit5.PactVerificationInvocationContextProvider;
import au.com.dius.pact.provider.junitsupport.*;
import au.com.dius.pact.provider.junitsupport.loader.*;
import au.com.dius.pact.provider.spring.junit5.MockMvcTestTarget;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.TestTemplate;
import org.junit.jupiter.api.extension.ExtendWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.boot.test.mock.mockito.MockBean;
import org.springframework.test.web.servlet.MockMvc;

import static org.mockito.ArgumentMatchers.*;
import static org.mockito.Mockito.when;

@WebMvcTest(OrderController.class)
// โหลด pact จาก Pact Broker
@Provider("order-service")
@PactBroker(
    url = "${PACT_BROKER_URL:http://localhost:9292}",
    authentication = @PactBrokerAuth(token = "${PACT_BROKER_TOKEN:}")
)
// หรือโหลดจาก local file
// @PactFolder("src/test/resources/pacts")
class OrderProviderPactTest {

    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private OrderService orderService;

    @BeforeEach
    void before(PactVerificationContext context) {
        context.setTarget(new MockMvcTestTarget(mockMvc));
    }

    @TestTemplate
    @ExtendWith(PactVerificationInvocationContextProvider.class)
    void pactVerificationTestTemplate(PactVerificationContext context) {
        context.verifyInteraction();
    }

    // State handlers - setup mock data ตาม state ที่ consumer กำหนด
    @State("order with id 1 exists")
    void orderExists() {
        when(orderService.findById(1L)).thenReturn(Optional.of(
            Order.builder()
                .id(1L)
                .customerId("CUST-001")
                .status(OrderStatus.CONFIRMED)
                .totalAmount(new BigDecimal("1500.00"))
                .items(List.of(
                    OrderItem.builder()
                        .productId("PROD-001")
                        .quantity(2)
                        .price(new BigDecimal("750.00"))
                        .build()
                ))
                .build()
        ));
    }

    @State("order with id 99999 does not exist")
    void orderNotFound() {
        when(orderService.findById(99999L)).thenReturn(Optional.empty());
    }
}
```

### Pact Broker Setup

```yaml
# docker-compose-pact.yml
version: '3.8'

services:
  pact-broker:
    image: pactfoundation/pact-broker:2.107.0
    ports:
      - "9292:9292"
    environment:
      PACT_BROKER_DATABASE_URL: "postgres://pact:password@postgres/pact_broker"
      PACT_BROKER_LOG_LEVEL: INFO
      PACT_BROKER_SQL_LOG_LEVEL: DEBUG
    depends_on:
      - postgres

  postgres:
    image: postgres:15
    environment:
      POSTGRES_USER: pact
      POSTGRES_PASSWORD: password
      POSTGRES_DB: pact_broker
    volumes:
      - pact-db:/var/lib/postgresql/data

volumes:
  pact-db:
```

---

## ขั้นตอนที่ 2500: Contract Evolution and Versioning {#versioning}

### Semantic Versioning สำหรับ Contracts

```groovy
// Contract version 1.0 - Initial
// src/test/resources/contracts/orders/v1/get_order.groovy
Contract.make {
    description "v1.0: Get order - original format"
    
    request {
        method GET()
        url "/api/v1/orders/1"
    }
    
    response {
        status 200
        body(
            id: 1,
            customerId: "CUST-001",
            status: "CONFIRMED",
            amount: 1500.00  // v1 field name
        )
    }
}
```

```groovy
// Contract version 2.0 - Breaking change (rename field)
// src/test/resources/contracts/orders/v2/get_order.groovy
Contract.make {
    description "v2.0: Get order - new format with renamed field"
    
    request {
        method GET()
        url "/api/v2/orders/1"
    }
    
    response {
        status 200
        body(
            id: 1,
            customerId: "CUST-001",
            status: "CONFIRMED",
            totalAmount: 1500.00  // v2 field name (renamed from 'amount')
        )
    }
}
```

### Backward Compatible Contract Evolution

```groovy
// Contract ที่รองรับทั้ง old และ new consumers
Contract.make {
    description "Get order with backward-compatible response"
    
    request {
        method GET()
        url "/api/orders/1"
    }
    
    response {
        status 200
        body(
            id: 1,
            customerId: "CUST-001",
            status: "CONFIRMED",
            // เพิ่ม field ใหม่ แต่ไม่ลบ field เก่า
            totalAmount: 1500.00,
            // Deprecated fields - ยังคงส่งไปเพื่อ backward compatibility
            // amount: 1500.00  // ลบออกแล้ว - breaking change!
        )
    }
}
```

### Contract Migration Helper

```java
package com.example.producer.migration;

import org.springframework.stereotype.Component;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.node.ObjectNode;

// Transformer สำหรับ response เพื่อ support หลาย version
@Component
public class OrderResponseTransformer {

    private final ObjectMapper objectMapper;

    public OrderResponseTransformer(ObjectMapper objectMapper) {
        this.objectMapper = objectMapper;
    }

    // Transform response ตาม API version ที่ consumer ต้องการ
    public OrderResponse transformForVersion(Order order, String apiVersion) {
        return switch (apiVersion) {
            case "v1" -> OrderResponse.builder()
                .id(order.getId())
                .customerId(order.getCustomerId())
                .status(order.getStatus().name())
                .amount(order.getTotalAmount())  // v1: field name 'amount'
                .build();
            case "v2", null -> OrderResponse.builder()
                .id(order.getId())
                .customerId(order.getCustomerId())
                .status(order.getStatus().name())
                .totalAmount(order.getTotalAmount())  // v2+: field name 'totalAmount'
                .build();
            default -> throw new UnsupportedApiVersionException("Version: " + apiVersion);
        };
    }
}
```

---

## ขั้นตอนที่ 2505: Breaking Changes Detection {#breaking-changes}

### Contract Compatibility Check

```java
package com.example.producer.contract;

import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.cloud.contract.verifier.dsl.Contract;
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

// Test ที่ตรวจสอบว่า contract ยังคง compatible
@SpringBootTest
class ContractCompatibilityTest {

    @Test
    void shouldNotBreakExistingContracts() {
        // โหลด contract เก่าจาก version control
        List<ContractDefinition> oldContracts = loadContracts("v1.0.0");
        
        // โหลด contract ใหม่
        List<ContractDefinition> newContracts = loadContracts("current");
        
        // ตรวจสอบ breaking changes
        ContractCompatibilityChecker checker = new ContractCompatibilityChecker();
        List<BreakingChange> breakingChanges = checker.findBreakingChanges(
            oldContracts, newContracts);
        
        assertThat(breakingChanges)
            .withFailMessage("Breaking changes detected: %s", breakingChanges)
            .isEmpty();
    }
}
```

### Custom Breaking Change Detector

```java
package com.example.producer.contract;

import java.util.ArrayList;
import java.util.List;

public class ContractCompatibilityChecker {

    public List<BreakingChange> findBreakingChanges(
            List<ContractDefinition> oldContracts,
            List<ContractDefinition> newContracts) {
        
        List<BreakingChange> changes = new ArrayList<>();
        
        for (ContractDefinition oldContract : oldContracts) {
            ContractDefinition newContract = findMatchingContract(
                newContracts, oldContract.getName());
            
            if (newContract == null) {
                // Contract ถูกลบ - breaking change!
                changes.add(new BreakingChange(
                    BreakingChangeType.CONTRACT_REMOVED,
                    "Contract removed: " + oldContract.getName()
                ));
                continue;
            }
            
            // ตรวจสอบ response fields
            checkResponseFields(oldContract, newContract, changes);
            
            // ตรวจสอบ request URL
            checkRequestUrl(oldContract, newContract, changes);
            
            // ตรวจสอบ status code
            checkStatusCode(oldContract, newContract, changes);
        }
        
        return changes;
    }

    private void checkResponseFields(ContractDefinition old, 
                                      ContractDefinition newDef,
                                      List<BreakingChange> changes) {
        Set<String> oldFields = old.getResponseBodyFields();
        Set<String> newFields = newDef.getResponseBodyFields();
        
        // Field ที่ถูกลบ = breaking change
        for (String field : oldFields) {
            if (!newFields.contains(field)) {
                changes.add(new BreakingChange(
                    BreakingChangeType.RESPONSE_FIELD_REMOVED,
                    "Field removed from response: " + field + 
                    " in contract: " + old.getName()
                ));
            }
        }
        
        // Field ที่ถูก rename = breaking change
        // (ตรวจสอบด้วย semantic analysis หรือ annotation)
    }

    private void checkRequestUrl(ContractDefinition old,
                                   ContractDefinition newDef,
                                   List<BreakingChange> changes) {
        if (!old.getRequestUrl().equals(newDef.getRequestUrl())) {
            changes.add(new BreakingChange(
                BreakingChangeType.REQUEST_URL_CHANGED,
                String.format("URL changed from '%s' to '%s' in contract: %s",
                    old.getRequestUrl(), 
                    newDef.getRequestUrl(), 
                    old.getName())
            ));
        }
    }
}
```

### Git Hook สำหรับ Check Breaking Changes

```bash
#!/bin/bash
# .git/hooks/pre-push

echo "Checking for contract breaking changes..."

# Run contract compatibility check
mvn -pl order-service test -Dtest=ContractCompatibilityTest -q

if [ $? -ne 0 ]; then
    echo "ERROR: Breaking contract changes detected!"
    echo "Please update the contract version or maintain backward compatibility."
    echo "Run: mvn -pl order-service test -Dtest=ContractCompatibilityTest for details"
    exit 1
fi

echo "Contract compatibility check passed."
exit 0
```

---

## ขั้นตอนที่ 2510: Messaging Contract Testing

### Contract สำหรับ Kafka Messages

```groovy
// src/test/resources/contracts/messaging/order-created.groovy
package contracts.messaging

import org.springframework.cloud.contract.spec.Contract

Contract.make {
    description "Should publish OrderCreated event when order is created"
    
    label "order.created"  // trigger label สำหรับ consumer
    
    input {
        triggeredBy("createOrder()")  // method ที่ trigger message
    }
    
    outputMessage {
        sentTo("order-events")  // Kafka topic
        body(
            eventType: "ORDER_CREATED",
            orderId: $(producer(regex('[0-9]+')), consumer(1)),
            customerId: "CUST-001",
            totalAmount: 1500.00,
            items: [
                [
                    productId: "PROD-001",
                    quantity: 2
                ]
            ],
            timestamp: $(producer(regex('.*')), consumer('2024-01-15T10:00:00Z'))
        )
        headers {
            messagingContentType(applicationJson())
            header("event-type", "ORDER_CREATED")
        }
    }
}
```

### Consumer Messaging Test

```java
package com.example.consumer;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.cloud.contract.stubrunner.StubTrigger;
import org.springframework.cloud.contract.stubrunner.spring.AutoConfigureStubRunner;
import org.springframework.cloud.contract.stubrunner.spring.StubRunnerProperties;

@SpringBootTest
@AutoConfigureStubRunner(
    ids = "com.example:order-service:+:stubs",
    stubsMode = StubRunnerProperties.StubsMode.LOCAL
)
class OrderEventConsumerContractTest {

    @Autowired
    private StubTrigger stubTrigger;

    @Autowired
    private NotificationService notificationService;

    @Test
    void shouldProcessOrderCreatedEvent() throws Exception {
        // Trigger message ผ่าน stub
        stubTrigger.trigger("order.created");
        
        // รอ message ถูก process
        Thread.sleep(500);
        
        // Verify ว่า consumer process message ถูกต้อง
        verify(notificationService).sendOrderConfirmation(
            argThat(event -> 
                event.getOrderId() != null && 
                event.getEventType().equals("ORDER_CREATED")
            )
        );
    }
}
```

---

## ขั้นตอนที่ 2515: Contract Test Report

```java
package com.example.producer.contract;

import org.junit.jupiter.api.AfterAll;
import org.junit.jupiter.api.TestInfo;
import java.util.ArrayList;
import java.util.List;

// Generates contract test report
public class ContractTestReporter {

    private static final List<ContractTestResult> results = new ArrayList<>();

    public static void recordResult(String contractName, boolean passed, String details) {
        results.add(new ContractTestResult(contractName, passed, details));
    }

    @AfterAll
    static void generateReport() {
        System.out.println("\n=== Contract Test Report ===");
        System.out.println("Total: " + results.size());
        System.out.println("Passed: " + results.stream().filter(r -> r.isPassed()).count());
        System.out.println("Failed: " + results.stream().filter(r -> !r.isPassed()).count());
        System.out.println("\nDetails:");
        results.forEach(r -> {
            String icon = r.isPassed() ? "✓" : "✗";
            System.out.printf("  %s %s%n", icon, r.getContractName());
            if (!r.isPassed()) {
                System.out.printf("    Error: %s%n", r.getDetails());
            }
        });
        System.out.println("============================\n");
    }

    record ContractTestResult(String contractName, boolean passed, String details) {}
}
```

---

## สรุปสิ่งที่เรียนรู้

ใน Part นี้เราได้เรียนรู้:

1. **Contract Testing Concept** - ทำไม consumer-driven contract testing ถึงสำคัญกว่า integration test
2. **Spring Cloud Contract** - การเขียน contract ใน Groovy/YAML และ auto-generate tests
3. **Stub Generation** - การสร้าง stub JAR สำหรับ consumer ใช้ test
4. **Pact Framework** - Alternative ที่ใช้งานได้หลาย language พร้อม Pact Broker
5. **CI/CD Integration** - การ integrate contract test ใน pipeline
6. **Contract Evolution** - การ evolve contract โดยไม่ break consumers
7. **Breaking Change Detection** - การตรวจหา breaking changes อัตโนมัติ

### Best Practices
- Consumer กำหนด contract, ไม่ใช่ Provider
- ใช้ semantic versioning สำหรับ contracts
- เก็บ contracts ใน version control
- Run contract tests ทั้ง consumer และ producer ใน CI
- ตรวจสอบ breaking changes ก่อน merge

---

*[← Part 71: Observability](./part-71-observability.md) | [Part 73: Zero-Downtime Deployment →](./part-73-zero-downtime.md)*

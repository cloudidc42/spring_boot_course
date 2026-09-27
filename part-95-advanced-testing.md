# Part 95: Advanced Testing in Production-Like Environments
## ขั้นตอนที่ 3401-3440

**ระดับ:** World-Class (ระดับโลก)
**เวลาเรียน:** 6-8 ชั่วโมง
**เป้าหมาย:** เรียนรู้ advanced testing techniques ที่ใช้ในระบบ production จริง ครอบคลุม Chaos testing, Load testing automation, Contract testing, Visual regression testing สำหรับ APIs, Synthetic monitoring และ Testing with production data snapshots

---

## ขั้นตอนที่ 3401: ภาพรวม Advanced Testing Pyramid

```
        /\
       /  \
      / E2E \          ← น้อยที่สุด แต่ครอบคลุมมากที่สุด
     /--------\
    / Contract  \      ← ตรวจสอบ API contracts ระหว่าง services
   /------------\
  / Integration   \    ← ทดสอบ components ร่วมกัน
 /----------------\
/    Unit Tests    \   ← มากที่สุด, เร็วที่สุด
\__________________/

Advanced Layers:
- Chaos Testing (ทดสอบ resilience)
- Load Testing (ทดสอบ performance)
- Synthetic Monitoring (ทดสอบ production)
- Contract Testing (ทดสอบ API compatibility)
```

## ขั้นตอนที่ 3402: Chaos Testing ด้วย Chaos Monkey

Chaos Engineering คือการ inject failures โดยตั้งใจ เพื่อค้นหาจุดอ่อนก่อนที่ production จะพัง

### ติดตั้ง Chaos Monkey สำหรับ Spring Boot

```xml
<!-- pom.xml -->
<dependency>
    <groupId>de.codecentric</groupId>
    <artifactId>chaos-monkey-spring-boot</artifactId>
    <version>3.0.1</version>
</dependency>
```

```yaml
# application.yml
chaos:
  monkey:
    enabled: true
    assaults:
      level: 5                    # ความรุนแรง (1-10)
      latency-active: true
      latency-range-start: 1000  # ms
      latency-range-end: 5000    # ms
      exceptions-active: true
      exception:
        type: java.lang.RuntimeException
        arguments:
          - type: java.lang.String
            value: "Chaos Monkey Exception!"
      kill-application-active: false  # อย่าเปิดใน production!
      memory-active: false
    watcher:
      controller: true
      restController: true
      service: true
      repository: true
      component: true
```

```java
// ChaosMonkeyConfig.java - Custom chaos settings
@Configuration
@Profile("chaos")
public class ChaosMonkeyConfig {

    @Bean
    public ChaosMonkeyRequestScope chaosMonkeyRequestScope(
            AssaultProperties assaultProperties,
            ChaosMonkeySettings settings) {
        return new ChaosMonkeyRequestScope(settings, assaultProperties);
    }
}
```

### Kubernetes Chaos Engineering ด้วย Chaos Mesh

```yaml
# chaos/pod-failure-experiment.yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: PodChaos
metadata:
  name: user-service-pod-failure
  namespace: shophub-staging
spec:
  action: pod-failure
  mode: one            # จำนวน pods ที่จะถูก chaos
  value: "1"
  duration: "30s"
  selector:
    namespaces:
      - shophub-staging
    labelSelectors:
      app: user-service
  scheduler:
    cron: "@every 10m"  # chaos ทุก 10 นาที
```

```yaml
# chaos/network-partition-experiment.yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: NetworkChaos
metadata:
  name: order-to-inventory-delay
  namespace: shophub-staging
spec:
  action: delay
  mode: all
  selector:
    namespaces:
      - shophub-staging
    labelSelectors:
      app: order-service
  delay:
    latency: "2s"
    jitter: "500ms"
    correlation: "50"
  direction: to
  target:
    selector:
      namespaces:
        - shophub-staging
      labelSelectors:
        app: inventory-service
  duration: "5m"
```

```yaml
# chaos/memory-stress-experiment.yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: StressChaos
metadata:
  name: product-service-memory-stress
  namespace: shophub-staging
spec:
  mode: one
  selector:
    namespaces:
      - shophub-staging
    labelSelectors:
      app: product-service
  stressors:
    memory:
      workers: 4
      size: "256MB"
  duration: "2m"
```

### Chaos Test ที่ Automated

```java
// ChaosTestSuite.java - Automated chaos experiments
@SpringBootTest
@ActiveProfiles("chaos-test")
@Slf4j
class ChaosResilienceTest {

    @Autowired
    private RecommendationService recommendationService;

    @Autowired
    private ChaosMonkeySettings chaosSettings;

    @Autowired
    private AssaultProperties assaultProperties;

    @Test
    @DisplayName("Service should handle latency injection gracefully")
    void shouldHandleLatencyGracefully() throws Exception {
        // Enable latency assault
        assaultProperties.setLatencyActive(true);
        assaultProperties.setLatencyRangeStart(2000);
        assaultProperties.setLatencyRangeEnd(3000);
        assaultProperties.setLevel(5);

        // Service ต้องตอบสนองภายใน timeout
        assertTimeout(Duration.ofSeconds(10), () -> {
            List<ProductDto> recommendations = 
                    recommendationService.getPersonalizedRecommendations(1L, 10);
            // ควรได้ fallback recommendations
            assertThat(recommendations).isNotEmpty();
        });

        // Disable after test
        assaultProperties.setLatencyActive(false);
    }

    @Test
    @DisplayName("Service should use fallback when exceptions are injected")
    void shouldUseFallbackOnException() {
        // Enable exception assault
        assaultProperties.setExceptionsActive(true);
        assaultProperties.setLevel(8);

        // Service ไม่ควร throw exception
        assertDoesNotThrow(() -> {
            List<ProductDto> products = 
                    recommendationService.getPersonalizedRecommendations(1L, 10);
            assertThat(products).isNotNull();
        });

        assaultProperties.setExceptionsActive(false);
    }
}
```

## ขั้นตอนที่ 3403: Load Testing Automation ใน CI/CD

Load testing อัตโนมัติช่วยตรวจจับ performance regression ก่อน deploy

### Gatling Load Test

```scala
// src/gatling/scala/simulations/OrderServiceSimulation.scala
package simulations

import io.gatling.core.Predef._
import io.gatling.http.Predef._
import scala.concurrent.duration._

class OrderServiceSimulation extends Simulation {

  val httpProtocol = http
    .baseUrl(System.getProperty("baseUrl", "http://localhost:8080"))
    .acceptHeader("application/json")
    .contentTypeHeader("application/json")
    .header("Authorization", s"Bearer ${System.getProperty("authToken", "test-token")}")

  // สร้าง orders scenario
  val createOrderScenario = scenario("Create Order")
    .exec(
      http("Create Order")
        .post("/api/orders")
        .body(StringBody("""
          {
            "items": [
              {"productId": 1, "quantity": 2},
              {"productId": 2, "quantity": 1}
            ],
            "shippingAddress": "123 Test St, Bangkok"
          }
        """))
        .check(status.is(201))
        .check(jsonPath("$.data.orderNumber").saveAs("orderNumber"))
    )
    .pause(1)
    .exec(
      http("Get Order Status")
        .get("/api/orders/${orderNumber}")
        .check(status.is(200))
        .check(jsonPath("$.data.status").is("PENDING"))
    )

  // Product browsing scenario
  val browseProductsScenario = scenario("Browse Products")
    .exec(
      http("List Products")
        .get("/api/products?page=0&size=20")
        .check(status.is(200))
        .check(jsonPath("$.data.content").exists)
    )
    .pause(2)
    .exec(
      http("Get Product Detail")
        .get("/api/products/1")
        .check(status.is(200))
    )
    .pause(1)
    .exec(
      http("Get Recommendations")
        .get("/api/recommendations/1?limit=10")
        .check(status.is(200))
    )

  // Performance SLA thresholds
  val successThreshold = 0.99  // 99% success rate
  val p95Threshold = 1000       // 95th percentile < 1000ms
  val p99Threshold = 2000       // 99th percentile < 2000ms

  setUp(
    // Normal load
    browseProductsScenario.inject(
      rampUsersPerSec(1).to(50).during(2.minutes),
      constantUsersPerSec(50).during(5.minutes)
    ).protocols(httpProtocol),
    
    // Transaction load
    createOrderScenario.inject(
      rampUsersPerSec(1).to(10).during(2.minutes),
      constantUsersPerSec(10).during(5.minutes)
    ).protocols(httpProtocol)
  )
  .assertions(
    global.responseTime.percentile3.lt(p99Threshold),   // P99 < 2000ms
    global.responseTime.percentile2.lt(p95Threshold),   // P95 < 1000ms
    global.successfulRequests.percent.gt(successThreshold * 100),
    details("Create Order").responseTime.mean.lt(500),
    details("Get Recommendations").responseTime.mean.lt(200)
  )
}
```

### Stress Test

```scala
// StressTestSimulation.scala
class StressTestSimulation extends Simulation {

  val httpProtocol = http
    .baseUrl(System.getProperty("baseUrl", "http://localhost:8080"))

  val stressScenario = scenario("Stress Test")
    .exec(
      http("Health Check under stress")
        .get("/actuator/health")
        .check(status.is(200))
        .check(jsonPath("$.status").is("UP"))
    )

  setUp(
    stressScenario.inject(
      rampUsersPerSec(10).to(500).during(5.minutes),   // ramp up
      constantUsersPerSec(500).during(10.minutes),       // hold
      rampUsersPerSec(500).to(0).during(2.minutes)       // ramp down
    ).protocols(httpProtocol)
  )
  .assertions(
    // สูงสุด 1% error rate ภายใต้ stress
    global.failedRequests.percent.lt(1),
    global.responseTime.percentile3.lt(5000)
  )
}
```

### Integration กับ CI/CD

```yaml
# .github/workflows/performance-test.yml
name: Performance Test

on:
  push:
    branches: [main, develop]
  schedule:
    - cron: '0 6 * * 1'  # ทุกวันจันทร์ 6am

jobs:
  load-test:
    name: Load Test
    runs-on: ubuntu-latest
    services:
      app:
        image: myregistry.azurecr.io/shophub/user-service:latest
        ports:
          - 8081:8080
        env:
          SPRING_PROFILES_ACTIVE: test

    steps:
      - uses: actions/checkout@v4

      - name: Wait for app to be ready
        run: |
          for i in {1..30}; do
            if curl -f http://localhost:8081/actuator/health; then
              break
            fi
            sleep 2
          done

      - name: Run Gatling tests
        run: |
          mvn gatling:test \
            -Dgatling.simulationClass=simulations.OrderServiceSimulation \
            -DbaseUrl=http://localhost:8081 \
            -DauthToken=${{ secrets.TEST_AUTH_TOKEN }}

      - name: Upload Gatling Report
        uses: actions/upload-artifact@v3
        if: always()
        with:
          name: gatling-report
          path: target/gatling/

      - name: Check performance thresholds
        run: |
          # Parse Gatling result and fail if SLA not met
          RESULT=$(cat target/gatling/*/js/stats.json | jq '.stats.percentiles3.value')
          echo "P99 latency: ${RESULT}ms"
          if [ "$RESULT" -gt 2000 ]; then
            echo "FAIL: P99 latency $RESULT ms exceeds threshold 2000ms"
            exit 1
          fi

      - name: Comment PR with results
        if: github.event_name == 'pull_request'
        uses: actions/github-script@v6
        with:
          script: |
            const fs = require('fs');
            // Parse and post results as PR comment
```

## ขั้นตอนที่ 3404: Contract Testing ด้วย Pact

Contract testing ตรวจสอบว่า API ระหว่าง services ยังทำงาน compatible กันอยู่

### Consumer Contract (Order Service → Inventory Service)

```java
// order-service/src/test/java/com/shophub/order/contract/InventoryClientContractTest.java
@ExtendWith(PactConsumerTestExt.class)
@PactTestFor(providerName = "inventory-service", port = "8084")
class InventoryClientContractTest {

    @Pact(consumer = "order-service")
    public RequestResponsePact checkAndReserveStockPact(PactDslWithProvider builder) {
        return builder
                .given("sufficient stock available for SKU-001 and SKU-002")
                .uponReceiving("a request to check and reserve stock")
                .path("/api/inventory/check-and-reserve")
                .method("POST")
                .headers(Map.of("Content-Type", "application/json"))
                .body(new PactDslJsonBody()
                        .array("items")
                            .object()
                                .stringValue("sku", "SKU-001")
                                .numberValue("quantity", 2)
                                .integerMatching("productId", 1)
                            .closeObject()
                            .object()
                                .stringValue("sku", "SKU-002")
                                .numberValue("quantity", 1)
                                .integerMatching("productId", 2)
                            .closeObject()
                        .closeArray()
                )
                .willRespondWith()
                .status(200)
                .headers(Map.of("Content-Type", "application/json"))
                .body(new PactDslJsonBody()
                        .booleanValue("success", true)
                        .object("data")
                            .booleanValue("available", true)
                            .stringMatcher("message", ".*", "Stock reserved successfully")
                        .closeObject()
                )
                .toPact();
    }

    @Test
    @PactTestFor(pactMethod = "checkAndReserveStockPact")
    void shouldSuccessfullyCheckAndReserveStock(MockServer mockServer) {
        // ใช้ Feign client กับ mock server
        InventoryClient client = Feign.builder()
                .decoder(new JacksonDecoder())
                .encoder(new JacksonEncoder())
                .target(InventoryClient.class, mockServer.getUrl());

        StockCheckRequest request = new StockCheckRequest();
        request.setItems(List.of(
                new StockCheckRequest.Item(1L, "SKU-001", 2),
                new StockCheckRequest.Item(2L, "SKU-002", 1)
        ));

        ApiResponse<StockCheckResponse> response = client.checkAndReserveStock(request);

        assertThat(response.isSuccess()).isTrue();
        assertThat(response.getData().isAvailable()).isTrue();
    }

    @Pact(consumer = "order-service")
    public RequestResponsePact insufficientStockPact(PactDslWithProvider builder) {
        return builder
                .given("insufficient stock for SKU-001")
                .uponReceiving("a request when stock is insufficient")
                .path("/api/inventory/check-and-reserve")
                .method("POST")
                .body(new PactDslJsonBody()
                        .array("items")
                            .object()
                                .stringValue("sku", "SKU-001")
                                .numberValue("quantity", 100)
                                .integerMatching("productId", 1)
                            .closeObject()
                        .closeArray()
                )
                .willRespondWith()
                .status(200)
                .body(new PactDslJsonBody()
                        .booleanValue("success", true)
                        .object("data")
                            .booleanValue("available", false)
                            .stringMatcher("message", ".*Insufficient.*", "Insufficient stock for SKU-001")
                        .closeObject()
                )
                .toPact();
    }
}
```

### Provider Verification (Inventory Service)

```java
// inventory-service/src/test/java/com/shophub/inventory/contract/InventoryProviderContractTest.java
@Provider("inventory-service")
@PactBroker(
    url = "${PACT_BROKER_URL:http://localhost:9292}",
    authentication = @PactBrokerAuth(token = "${PACT_BROKER_TOKEN}")
)
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class InventoryProviderContractTest {

    @LocalServerPort
    private int port;

    @MockBean
    private InventoryRepository inventoryRepository;

    @BeforeEach
    void setUp(PactVerificationContext context) {
        context.setTarget(new HttpTestTarget("localhost", port));
    }

    @TestTemplate
    @ExtendWith(PactVerificationInvocationContextProvider.class)
    void verifyPact(PactVerificationContext context) {
        context.verifyInteraction();
    }

    // ตั้งค่า state สำหรับแต่ละ test
    @State("sufficient stock available for SKU-001 and SKU-002")
    public void sufficientStockState() {
        Inventory inventory1 = Inventory.builder()
                .sku("SKU-001").productId(1L)
                .availableQuantity(50).reservedQuantity(0)
                .reorderLevel(10).build();

        Inventory inventory2 = Inventory.builder()
                .sku("SKU-002").productId(2L)
                .availableQuantity(100).reservedQuantity(0)
                .reorderLevel(10).build();

        when(inventoryRepository.findBySku("SKU-001")).thenReturn(Optional.of(inventory1));
        when(inventoryRepository.findBySku("SKU-002")).thenReturn(Optional.of(inventory2));
        when(inventoryRepository.findBySkuWithLock(any())).thenAnswer(inv -> {
            String sku = inv.getArgument(0);
            return "SKU-001".equals(sku) ? Optional.of(inventory1) : Optional.of(inventory2);
        });
        when(inventoryRepository.save(any())).thenAnswer(inv -> inv.getArgument(0));
    }

    @State("insufficient stock for SKU-001")
    public void insufficientStockState() {
        Inventory inventory = Inventory.builder()
                .sku("SKU-001").productId(1L)
                .availableQuantity(5).reservedQuantity(0)
                .reorderLevel(10).build();

        when(inventoryRepository.findBySku("SKU-001")).thenReturn(Optional.of(inventory));
    }
}
```

### Pact Broker Setup

```yaml
# docker-compose.test.yml
services:
  pact-broker:
    image: pactfoundation/pact-broker:latest
    ports:
      - "9292:9292"
    environment:
      PACT_BROKER_DATABASE_URL: "postgres://pact:password@postgres/pact_broker"
      PACT_BROKER_DATABASE_ADAPTER: postgres
      PACT_BROKER_BASIC_AUTH_USERNAME: admin
      PACT_BROKER_BASIC_AUTH_PASSWORD: admin
    depends_on:
      - postgres

  postgres:
    image: postgres:15
    environment:
      POSTGRES_DB: pact_broker
      POSTGRES_USER: pact
      POSTGRES_PASSWORD: password
```

## ขั้นตอนที่ 3405: Visual Regression Testing สำหรับ APIs

"Visual regression" สำหรับ API หมายถึงการตรวจสอบว่า response structure ไม่เปลี่ยนแปลงโดยไม่ตั้งใจ

### Schema Validation Testing

```java
// ApiSchemaValidationTest.java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@ActiveProfiles("test")
class ApiSchemaValidationTest {

    @Autowired
    private TestRestTemplate restTemplate;

    @LocalServerPort
    private int port;

    private ObjectMapper objectMapper = new ObjectMapper();

    @Test
    void productListSchemaIsStable() throws Exception {
        ResponseEntity<String> response = restTemplate.getForEntity(
                "/api/products?page=0&size=5", String.class);

        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.OK);

        // ตรวจสอบว่า schema ยังเหมือนเดิม
        JsonNode body = objectMapper.readTree(response.getBody());
        
        assertThat(body.has("success")).isTrue();
        assertThat(body.has("data")).isTrue();
        assertThat(body.get("data").has("content")).isTrue();
        assertThat(body.get("data").has("totalElements")).isTrue();
        assertThat(body.get("data").has("totalPages")).isTrue();

        // ตรวจสอบ product fields
        JsonNode firstProduct = body.get("data").get("content").get(0);
        assertThat(firstProduct.has("id")).isTrue();
        assertThat(firstProduct.has("sku")).isTrue();
        assertThat(firstProduct.has("name")).isTrue();
        assertThat(firstProduct.has("price")).isTrue();
        assertThat(firstProduct.has("category")).isTrue();

        // ตรวจสอบว่าไม่มี field ที่ไม่ควรเห็น (sensitive data)
        assertThat(firstProduct.has("internalCost")).isFalse();
        assertThat(firstProduct.has("supplierId")).isFalse();
    }

    @Test
    void orderResponseSchemaIsConsistent() throws Exception {
        // สร้าง order ก่อน
        CreateOrderRequest request = new CreateOrderRequest();
        request.setItems(List.of(new CreateOrderRequest.Item(1L, 1)));
        request.setShippingAddress("Bangkok");

        ResponseEntity<String> createResponse = restTemplate.postForEntity(
                "/api/orders", request, String.class);

        assertThat(createResponse.getStatusCode()).isEqualTo(HttpStatus.CREATED);

        JsonNode orderBody = objectMapper.readTree(createResponse.getBody());
        JsonNode order = orderBody.get("data");

        // Required fields
        assertAll(
                () -> assertThat(order.has("id")).isTrue(),
                () -> assertThat(order.has("orderNumber")).isTrue(),
                () -> assertThat(order.has("status")).isTrue(),
                () -> assertThat(order.has("totalAmount")).isTrue(),
                () -> assertThat(order.has("items")).isTrue(),
                () -> assertThat(order.has("createdAt")).isTrue()
        );

        // Field types
        assertThat(order.get("id").isNumber()).isTrue();
        assertThat(order.get("orderNumber").isTextual()).isTrue();
        assertThat(order.get("totalAmount").isNumber()).isTrue();
        assertThat(order.get("items").isArray()).isTrue();
    }
}
```

### API Snapshot Testing

```java
// ApiSnapshotTest.java - บันทึกและเปรียบเทียบ API responses
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class ApiSnapshotTest {

    @Autowired
    private TestRestTemplate restTemplate;

    private static final Path SNAPSHOTS_DIR = 
            Paths.get("src/test/resources/api-snapshots");

    @Test
    void productApiResponseMatchesSnapshot() throws Exception {
        ResponseEntity<String> response = restTemplate.getForEntity(
                "/api/products/1", String.class);

        String snapshotFile = "product-detail-snapshot.json";
        Path snapshotPath = SNAPSHOTS_DIR.resolve(snapshotFile);

        if (!Files.exists(snapshotPath)) {
            // สร้าง snapshot ครั้งแรก
            Files.createDirectories(SNAPSHOTS_DIR);
            Files.writeString(snapshotPath, 
                    prettyPrint(response.getBody()));
            log.info("Created snapshot: {}", snapshotFile);
        } else {
            // เปรียบเทียบกับ snapshot
            String expected = Files.readString(snapshotPath);
            assertJsonEquals(expected, response.getBody());
        }
    }

    private void assertJsonEquals(String expected, String actual) throws Exception {
        ObjectMapper mapper = new ObjectMapper();
        JsonNode expectedNode = mapper.readTree(expected);
        JsonNode actualNode = mapper.readTree(actual);

        // เปรียบเทียบ structure (ไม่สนใจ timestamps และ IDs)
        assertSchemaMatch(expectedNode, actualNode, "");
    }

    private void assertSchemaMatch(JsonNode expected, JsonNode actual, String path) {
        if (expected.isObject()) {
            expected.fieldNames().forEachRemaining(field -> {
                assertThat(actual.has(field))
                        .as("Field '%s%s' should exist", path, field)
                        .isTrue();
                assertSchemaMatch(expected.get(field), actual.get(field), path + field + ".");
            });
        } else if (expected.isArray() && expected.size() > 0) {
            assertThat(actual.isArray())
                    .as("Path '%s' should be array", path)
                    .isTrue();
        }
    }

    private String prettyPrint(String json) throws Exception {
        ObjectMapper mapper = new ObjectMapper();
        return mapper.writerWithDefaultPrettyPrinter()
                .writeValueAsString(mapper.readTree(json));
    }
}
```

## ขั้นตอนที่ 3406: Synthetic Monitoring

Synthetic monitoring คือการส่ง requests จำลองไปยัง production ตลอดเวลา เพื่อตรวจสอบว่าระบบยังทำงานอยู่

```java
// SyntheticMonitor.java
@Component
@RequiredArgsConstructor
@Slf4j
public class SyntheticMonitor {

    private final RestTemplate restTemplate;
    private final MeterRegistry meterRegistry;
    private final AlertService alertService;

    @Value("${monitoring.base-url:https://api.shophub.com}")
    private String baseUrl;

    // ทดสอบทุก 1 นาที
    @Scheduled(fixedRate = 60000)
    public void runHealthChecks() {
        checkEndpoint("health", "/actuator/health", "UP", 
                resp -> ((Map) resp.getBody()).get("status").equals("UP"));
        
        checkEndpoint("product-list", "/api/products?page=0&size=1", null,
                resp -> resp.getStatusCode().is2xxSuccessful());
        
        checkEndpoint("product-search", "/api/products?keyword=laptop", null,
                resp -> resp.getStatusCode().is2xxSuccessful());
    }

    // ทดสอบ user journey ทุก 5 นาที
    @Scheduled(fixedRate = 300000)
    public void runUserJourneyCheck() {
        runWithMetrics("user-journey", () -> {
            // 1. Login
            String token = performLogin();
            if (token == null) {
                alertService.alert("CRITICAL", "Login endpoint failure");
                return;
            }

            // 2. Browse products
            List<Long> productIds = browseProducts(token);
            if (productIds.isEmpty()) {
                alertService.alert("HIGH", "Product listing failure");
                return;
            }

            // 3. Check product detail
            boolean productOk = checkProductDetail(token, productIds.get(0));
            if (!productOk) {
                alertService.alert("HIGH", "Product detail failure");
            }

            log.info("User journey check passed");
        });
    }

    private void checkEndpoint(String name, String path, String expectedValue,
                                 java.util.function.Predicate<ResponseEntity<Map>> checker) {
        Timer timer = meterRegistry.timer("synthetic.check.duration", "endpoint", name);
        Counter successCounter = meterRegistry.counter("synthetic.check.success", "endpoint", name);
        Counter failureCounter = meterRegistry.counter("synthetic.check.failure", "endpoint", name);

        try {
            timer.record(() -> {
                ResponseEntity<Map> response = restTemplate.getForEntity(
                        baseUrl + path, Map.class);
                
                if (checker.test(response)) {
                    successCounter.increment();
                } else {
                    failureCounter.increment();
                    alertService.alert("HIGH", "Endpoint check failed: " + name);
                }
            });
        } catch (Exception e) {
            failureCounter.increment();
            log.error("Synthetic check failed for {}: {}", name, e.getMessage());
            alertService.alert("CRITICAL", "Endpoint unreachable: " + name);
        }
    }

    private void runWithMetrics(String checkName, Runnable check) {
        long startTime = System.currentTimeMillis();
        try {
            check.run();
            long duration = System.currentTimeMillis() - startTime;
            meterRegistry.timer("synthetic.journey.duration", "journey", checkName)
                    .record(duration, TimeUnit.MILLISECONDS);
        } catch (Exception e) {
            log.error("Journey check failed: {}", checkName, e);
            alertService.alert("CRITICAL", "User journey failure: " + e.getMessage());
        }
    }

    private String performLogin() {
        try {
            Map<String, String> loginRequest = Map.of(
                    "email", "synthetic-test@shophub.com",
                    "password", "synthetic-test-pass"
            );
            ResponseEntity<Map> response = restTemplate.postForEntity(
                    baseUrl + "/api/auth/login", loginRequest, Map.class);

            if (response.getStatusCode().is2xxSuccessful()) {
                return (String) ((Map) response.getBody().get("data")).get("token");
            }
        } catch (Exception e) {
            log.error("Login failed in synthetic test: {}", e.getMessage());
        }
        return null;
    }

    private List<Long> browseProducts(String token) {
        try {
            HttpHeaders headers = new HttpHeaders();
            headers.set("Authorization", "Bearer " + token);
            HttpEntity<Void> entity = new HttpEntity<>(headers);

            ResponseEntity<Map> response = restTemplate.exchange(
                    baseUrl + "/api/products?page=0&size=5",
                    HttpMethod.GET, entity, Map.class);

            if (response.getStatusCode().is2xxSuccessful()) {
                Map data = (Map) response.getBody().get("data");
                List<Map> content = (List<Map>) data.get("content");
                return content.stream()
                        .map(p -> ((Number) p.get("id")).longValue())
                        .collect(Collectors.toList());
            }
        } catch (Exception e) {
            log.error("Product browsing failed in synthetic test: {}", e.getMessage());
        }
        return Collections.emptyList();
    }

    private boolean checkProductDetail(String token, Long productId) {
        try {
            HttpHeaders headers = new HttpHeaders();
            headers.set("Authorization", "Bearer " + token);
            HttpEntity<Void> entity = new HttpEntity<>(headers);

            ResponseEntity<Map> response = restTemplate.exchange(
                    baseUrl + "/api/products/" + productId,
                    HttpMethod.GET, entity, Map.class);

            return response.getStatusCode().is2xxSuccessful();
        } catch (Exception e) {
            log.error("Product detail check failed: {}", e.getMessage());
            return false;
        }
    }
}
```

### AlertService สำหรับ Synthetic Monitoring

```java
// AlertService.java
@Service
@RequiredArgsConstructor
@Slf4j
public class AlertService {

    private final SlackWebhookClient slackClient;
    private final PagerDutyClient pagerDutyClient;

    // Anti-flood: ไม่แจ้งเตือนซ้ำในช่วงเวลาสั้น ๆ
    private final Map<String, Instant> lastAlertTimes = new ConcurrentHashMap<>();
    private static final Duration ALERT_COOLDOWN = Duration.ofMinutes(5);

    public void alert(String severity, String message) {
        String alertKey = severity + ":" + message;
        Instant lastAlert = lastAlertTimes.get(alertKey);

        if (lastAlert != null && 
                Duration.between(lastAlert, Instant.now()).compareTo(ALERT_COOLDOWN) < 0) {
            log.debug("Alert suppressed (cooldown): {}", message);
            return;
        }

        lastAlertTimes.put(alertKey, Instant.now());

        log.warn("ALERT [{}]: {}", severity, message);

        String emoji = switch (severity) {
            case "CRITICAL" -> "🚨";
            case "HIGH" -> "⚠️";
            case "MEDIUM" -> "⚡";
            default -> "ℹ️";
        };

        slackClient.send(String.format("%s [%s] %s", emoji, severity, message));

        if ("CRITICAL".equals(severity)) {
            pagerDutyClient.triggerIncident(message);
        }
    }
}
```

## ขั้นตอนที่ 3407: Testing กับ Production Data Snapshots

ใช้ข้อมูลจาก production ใน test environment เพื่อความใกล้เคียงกับ real-world scenarios

### Database Snapshot Strategy

```bash
#!/bin/bash
# scripts/create-test-snapshot.sh
# สร้าง anonymized snapshot จาก production data

set -e

PROD_HOST="prod-db.shophub.internal"
TEST_HOST="test-db.shophub.internal"
DB_NAME="shophub"
SNAPSHOT_DATE=$(date +%Y%m%d)
SNAPSHOT_FILE="snapshot_${SNAPSHOT_DATE}.sql"

echo "Creating production snapshot..."

# Dump production data (specific tables only)
pg_dump \
  --host=$PROD_HOST \
  --username=readonly_user \
  --dbname=$DB_NAME \
  --table=products \
  --table=categories \
  --table=inventory \
  --no-owner \
  --no-privileges \
  --format=custom \
  --file=/tmp/${SNAPSHOT_FILE}

echo "Anonymizing sensitive data..."

# Restore ไปยัง test DB ก่อน
pg_restore \
  --host=$TEST_HOST \
  --username=admin \
  --dbname=test_snapshot \
  --clean \
  /tmp/${SNAPSHOT_FILE}

# Anonymize PII data ใน test DB
psql --host=$TEST_HOST --username=admin --dbname=test_snapshot << 'SQL'
  -- Anonymize user data
  UPDATE users SET
    email = 'user_' || id || '@test.example.com',
    first_name = 'Test',
    last_name = 'User_' || id,
    phone = '000-000-' || LPAD(CAST(id AS VARCHAR), 4, '0');

  -- Anonymize order data
  UPDATE orders SET
    shipping_address = 'Test Address ' || id || ', Bangkok';

  -- Remove payment info
  DELETE FROM payment_methods;
  DELETE FROM payment_transactions;
  
  -- ลด scale ลง (เอาเฉพาะ 10% เพื่อความเร็วใน test)
  DELETE FROM orders WHERE id NOT IN (
    SELECT id FROM orders ORDER BY created_at DESC LIMIT 10000
  );
  
  VACUUM ANALYZE;
SQL

echo "Snapshot created and anonymized: $SNAPSHOT_FILE"
echo "Uploading to S3..."
aws s3 cp /tmp/${SNAPSHOT_FILE} s3://shophub-test-snapshots/${SNAPSHOT_FILE}
echo "Done!"
```

### TestContainers กับ Production Snapshot

```java
// ProductionSnapshotIT.java
@SpringBootTest
@ActiveProfiles("snapshot-test")
@Testcontainers
class ProductionSnapshotIT {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15")
            .withInitScript("production-snapshot-anonymized.sql")
            .withDatabaseName("shophub_test")
            .withUsername("test")
            .withPassword("test");

    @DynamicPropertySource
    static void registerDataSourceProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Autowired
    private ProductService productService;

    @Autowired
    private OrderService orderService;

    @Test
    void shouldHandleProductionDataVolume() {
        // ทดสอบกับ data volume จริงจาก production
        Page<ProductDto> products = productService.searchProducts("", null, 
                PageRequest.of(0, 20));
        
        assertThat(products.getTotalElements()).isGreaterThan(1000);
        assertThat(products.getContent()).hasSize(20);
    }

    @Test
    void queryPerformanceShouldMeetSLA() {
        long startTime = System.currentTimeMillis();
        
        // Query ที่ complex พอสมควร
        Page<ProductDto> results = productService.searchProducts(
                "laptop", "electronics", PageRequest.of(0, 10));
        
        long duration = System.currentTimeMillis() - startTime;
        
        // SLA: search ต้องเร็วกว่า 200ms
        assertThat(duration).isLessThan(200);
        assertThat(results).isNotNull();
    }

    @Test
    void reportGenerationWithRealData() {
        // ทดสอบ report generation กับ data จริง
        LocalDate start = LocalDate.now().minusMonths(1);
        LocalDate end = LocalDate.now();
        
        SalesReport report = orderService.generateSalesReport(start, end);
        
        assertThat(report).isNotNull();
        assertThat(report.getTotalOrders()).isGreaterThan(0);
        assertThat(report.getTotalRevenue()).isGreaterThan(BigDecimal.ZERO);
    }
}
```

### Data Masking Utilities

```java
// DataMaskingService.java
@Service
public class DataMaskingService {

    // Mask email: john.doe@example.com → j***@example.com
    public String maskEmail(String email) {
        if (email == null || !email.contains("@")) return email;
        String[] parts = email.split("@");
        String local = parts[0];
        String domain = parts[1];
        return local.charAt(0) + "***@" + domain;
    }

    // Mask phone: 0812345678 → 081****678
    public String maskPhone(String phone) {
        if (phone == null || phone.length() < 7) return phone;
        int visibleStart = 3;
        int visibleEnd = 3;
        String masked = phone.substring(0, visibleStart) +
                "*".repeat(phone.length() - visibleStart - visibleEnd) +
                phone.substring(phone.length() - visibleEnd);
        return masked;
    }

    // Mask credit card: 4111111111111234 → ****1234
    public String maskCreditCard(String cardNumber) {
        if (cardNumber == null || cardNumber.length() < 4) return cardNumber;
        return "****" + cardNumber.substring(cardNumber.length() - 4);
    }

    // Deterministic fake name (เหมือนกันทุกครั้งสำหรับ user id เดิม)
    public String getFakeName(Long userId) {
        String[] firstNames = {"สมชาย", "สมศรี", "วิชัย", "นภา", "กมล", "ปราณี"};
        String[] lastNames = {"ใจดี", "รักชาติ", "สุขใจ", "วงศ์ทอง", "พงษ์ไทย"};
        int firstIdx = (int) (userId % firstNames.length);
        int lastIdx = (int) ((userId / firstNames.length) % lastNames.length);
        return firstNames[firstIdx] + " " + lastNames[lastIdx];
    }
}
```

## ขั้นตอนที่ 3408: Test Environment Management

```java
// TestEnvironmentManager.java - จัดการ test environments
@Configuration
@Profile("integration-test")
public class TestEnvironmentConfig {

    // เริ่ม Kafka container สำหรับ integration tests
    @Bean
    public KafkaContainer kafkaContainer() {
        KafkaContainer kafka = new KafkaContainer(
                DockerImageName.parse("confluentinc/cp-kafka:7.5.0"));
        kafka.start();
        return kafka;
    }

    @Bean
    public GenericContainer<?> redisContainer() {
        GenericContainer<?> redis = new GenericContainer<>(
                DockerImageName.parse("redis:7-alpine"))
                .withExposedPorts(6379);
        redis.start();
        return redis;
    }

    @Bean
    @DependsOn("kafkaContainer")
    public KafkaTemplate<String, Object> testKafkaTemplate(KafkaContainer kafka) {
        Map<String, Object> props = new HashMap<>();
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, kafka.getBootstrapServers());
        props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, JsonSerializer.class);

        return new KafkaTemplate<>(new DefaultKafkaProducerFactory<>(props));
    }
}
```

### Test Utilities

```java
// TestDataFactory.java - สร้าง test data ง่าย ๆ
@Component
public class TestDataFactory {

    @Autowired
    private UserRepository userRepository;

    @Autowired
    private ProductRepository productRepository;

    @Autowired
    private InventoryRepository inventoryRepository;

    @Autowired
    private PasswordEncoder passwordEncoder;

    @Transactional
    public User createTestUser(String email) {
        return userRepository.save(User.builder()
                .username("testuser_" + System.currentTimeMillis())
                .email(email)
                .password(passwordEncoder.encode("Test1234!"))
                .firstName("Test")
                .lastName("User")
                .roles(Set.of("ROLE_USER"))
                .active(true)
                .build());
    }

    @Transactional
    public Product createTestProduct(String sku) {
        Product product = productRepository.save(Product.builder()
                .sku(sku)
                .name("Test Product " + sku)
                .description("Test description")
                .price(new BigDecimal("99.99"))
                .category("electronics")
                .available(true)
                .build());

        inventoryRepository.save(Inventory.builder()
                .sku(sku)
                .productId(product.getId())
                .availableQuantity(100)
                .reservedQuantity(0)
                .reorderLevel(10)
                .build());

        return product;
    }

    @Transactional
    public void cleanupTestData() {
        inventoryRepository.deleteAll();
        productRepository.deleteAll();
        userRepository.deleteAll();
    }
}
```

## ขั้นตอนที่ 3409-3440: สรุปและ Testing Strategy

### Testing สำหรับ Microservices

```java
// Integration Test สำหรับ Order Flow ทั้ง end-to-end
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Testcontainers
@ActiveProfiles("integration-test")
class OrderFlowIntegrationTest {

    @Autowired
    private TestRestTemplate restTemplate;

    @Autowired
    private TestDataFactory testDataFactory;

    private User testUser;
    private Product testProduct;

    @BeforeEach
    void setUp() {
        testUser = testDataFactory.createTestUser("test@flow.com");
        testProduct = testDataFactory.createTestProduct("TEST-SKU-001");
    }

    @AfterEach
    void tearDown() {
        testDataFactory.cleanupTestData();
    }

    @Test
    void completeOrderFlowShouldWork() throws Exception {
        // 1. Login
        LoginRequest loginRequest = new LoginRequest("test@flow.com", "Test1234!");
        ResponseEntity<ApiResponse<AuthResponse>> loginResponse = restTemplate.postForEntity(
                "/api/auth/login", loginRequest,
                new ParameterizedTypeReference<>() {});

        assertThat(loginResponse.getStatusCode()).isEqualTo(HttpStatus.OK);
        String token = loginResponse.getBody().getData().getToken();

        // 2. สร้าง order
        HttpHeaders headers = new HttpHeaders();
        headers.set("Authorization", "Bearer " + token);

        CreateOrderRequest orderRequest = new CreateOrderRequest();
        orderRequest.setItems(List.of(new CreateOrderRequest.Item(testProduct.getId(), 2)));
        orderRequest.setShippingAddress("123 Test St, Bangkok");

        HttpEntity<CreateOrderRequest> entity = new HttpEntity<>(orderRequest, headers);
        ResponseEntity<ApiResponse<OrderDto>> orderResponse = restTemplate.exchange(
                "/api/orders", HttpMethod.POST, entity,
                new ParameterizedTypeReference<>() {});

        assertThat(orderResponse.getStatusCode()).isEqualTo(HttpStatus.CREATED);
        String orderNumber = orderResponse.getBody().getData().getOrderNumber();
        assertThat(orderNumber).startsWith("ORD-");

        // 3. ตรวจสอบ order status
        ResponseEntity<ApiResponse<OrderDto>> statusResponse = restTemplate.exchange(
                "/api/orders/" + orderNumber, HttpMethod.GET,
                new HttpEntity<>(headers),
                new ParameterizedTypeReference<>() {});

        assertThat(statusResponse.getStatusCode()).isEqualTo(HttpStatus.OK);
        assertThat(statusResponse.getBody().getData().getStatus()).isEqualTo("PENDING");

        // 4. รอให้ Kafka process events (async)
        Thread.sleep(2000);

        // 5. ตรวจสอบว่า inventory ถูกอัปเดต
        ResponseEntity<Map> inventoryResponse = restTemplate.getForEntity(
                "/api/inventory/TEST-SKU-001", Map.class);
        
        // Available stock ควรลดลง 2 หน่วย
        assertThat((int) ((Map) inventoryResponse.getBody().get("data"))
                .get("availableQuantity")).isEqualTo(98);
    }
}
```

### Test Coverage Report

```xml
<!-- pom.xml - JaCoCo configuration -->
<plugin>
    <groupId>org.jacoco</groupId>
    <artifactId>jacoco-maven-plugin</artifactId>
    <version>0.8.11</version>
    <configuration>
        <excludes>
            <exclude>**/*Application.class</exclude>
            <exclude>**/*Config.class</exclude>
            <exclude>**/dto/**</exclude>
            <exclude>**/entity/**</exclude>
        </excludes>
    </configuration>
    <executions>
        <execution>
            <goals>
                <goal>prepare-agent</goal>
            </goals>
        </execution>
        <execution>
            <id>report</id>
            <phase>test</phase>
            <goals>
                <goal>report</goal>
            </goals>
        </execution>
        <execution>
            <id>check</id>
            <goals>
                <goal>check</goal>
            </goals>
            <configuration>
                <rules>
                    <rule>
                        <element>BUNDLE</element>
                        <limits>
                            <limit>
                                <counter>LINE</counter>
                                <value>COVEREDRATIO</value>
                                <minimum>0.80</minimum>  <!-- 80% line coverage -->
                            </limit>
                            <limit>
                                <counter>BRANCH</counter>
                                <value>COVEREDRATIO</value>
                                <minimum>0.70</minimum>  <!-- 70% branch coverage -->
                            </limit>
                        </limits>
                    </rule>
                </rules>
            </configuration>
        </execution>
    </executions>
</plugin>
```

### Complete Testing Checklist

```yaml
# testing-checklist.yml
unit-tests:
  - service layer coverage > 80%
  - repository layer mocked
  - edge cases covered
  - happy path + error paths

integration-tests:
  - database integration (TestContainers)
  - Kafka messaging (EmbeddedKafka)
  - Redis caching
  - REST endpoint tests

contract-tests:
  - consumer contracts published to Pact Broker
  - provider verifications passing
  - all service pairs covered

performance-tests:
  - p95 < 200ms for read endpoints
  - p99 < 1000ms for write endpoints
  - 99.9% success rate under normal load
  - graceful degradation under overload

chaos-tests:
  - latency injection (circuit breaker triggers)
  - exception injection (fallbacks work)
  - service unavailability (graceful degradation)

synthetic-monitoring:
  - health check every 1 minute
  - user journey every 5 minutes
  - alert on 2+ consecutive failures
  - PagerDuty for critical alerts
```

---

*[← Part 94: Machine Learning Integration](./part-94-machine-learning.md) | [Part 96: Next Chapter →](./part-96-next.md)*

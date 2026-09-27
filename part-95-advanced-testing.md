# Part 95: Advanced Testing Strategies for Spring Boot
## ขั้นตอนที่ 3401-3440

**ระดับ:** World-Class (ระดับโลก)
**เวลาเรียน:** 8-10 ชั่วโมง
**เป้าหมาย:** เรียนรู้ testing strategies ขั้นสูงสำหรับ production systems ครอบคลุม Chaos testing, Load testing CI gates, Contract testing automation, Synthetic monitoring, Production data snapshots, Shift-left security testing และ Testcontainers compose

---

## ขั้นตอนที่ 3401: Chaos Testing กับ Chaos Monkey for Spring Boot

Chaos Engineering คือการทดสอบความแข็งแกร่งของระบบโดยจงใจใส่ความผิดพลาดเข้าไป เพื่อค้นหาจุดอ่อนก่อนที่จะเกิดขึ้นจริงใน production

### Dependencies

```xml
<!-- pom.xml -->
<dependency>
    <groupId>de.codecentric</groupId>
    <artifactId>chaos-monkey-spring-boot</artifactId>
    <version>3.1.0</version>
</dependency>
```

### Configuration

```yaml
# application-chaos.yml
chaos:
  monkey:
    enabled: true
    assaults:
      level: 3                  # 1-10, likelihood ของ assault
      latency-active: true      # เพิ่ม latency แบบสุ่ม
      latency-range-start: 1000 # 1 วินาที
      latency-range-end: 5000   # 5 วินาที
      exceptions-active: false  # ยังไม่เปิด exceptions
      kill-application-active: false
    watcher:
      service: true             # Assault ที่ @Service beans
      rest-controller: false
      repository: true          # Assault ที่ @Repository beans
      component: false
```

### Chaos Testing Scenarios

```java
// chaos/ChaosTestScenario.java
@SpringBootTest
@ActiveProfiles("chaos")
@TestPropertySource(properties = {
    "chaos.monkey.enabled=true",
    "chaos.monkey.assaults.level=5",
    "chaos.monkey.assaults.latency-active=true"
})
class ChaosLatencyTest {
    
    @Autowired
    private OrderService orderService;
    
    @Autowired
    private ChaosMonkeySettings chaosMonkeySettings;
    
    @Test
    @DisplayName("Order service ควร fallback เมื่อ latency สูง")
    void orderServiceShouldHandleHighLatency() {
        // Enable chaos
        chaosMonkeySettings.getAssaultProperties().setLatencyActive(true);
        chaosMonkeySettings.getAssaultProperties().setLatencyRangeStart(2000);
        chaosMonkeySettings.getAssaultProperties().setLatencyRangeEnd(3000);
        
        // ทดสอบว่า service ยังทำงานได้ภายใน timeout
        long start = System.currentTimeMillis();
        
        assertDoesNotThrow(() -> {
            OrderResult result = orderService.processOrder(createTestOrder());
            // อาจล้มเหลว แต่ไม่ควร throw exception ที่ unhandled
        });
        
        long duration = System.currentTimeMillis() - start;
        assertThat(duration).isLessThan(5000); // ต้องมี timeout < 5s
    }
    
    @Test
    @DisplayName("ระบบควร return fallback response เมื่อ repository เกิดข้อผิดพลาด")
    void systemShouldReturnFallbackOnRepositoryError() {
        // Enable exception assault
        chaosMonkeySettings.getAssaultProperties().setExceptionsActive(true);
        chaosMonkeySettings.getAssaultProperties().setException(
            new RuntimeException("Chaos: Database simulated failure")
        );
        
        // ProductService ควรมี fallback
        List<Product> products = productService.findFeaturedProducts();
        
        // ไม่ควรได้ null - ควรได้ empty list หรือ cached data
        assertThat(products).isNotNull();
    }
}
```

### Resilience Patterns ที่รองรับ Chaos

```java
// service/ResilientProductService.java
@Service
public class ResilientProductService {
    
    private final ProductRepository productRepository;
    private final Cache<String, Product> localCache;
    
    // Circuit breaker + fallback
    @CircuitBreaker(name = "productService", fallbackMethod = "getProductFallback")
    @TimeLimiter(name = "productService")
    @Retry(name = "productService")
    public CompletableFuture<Product> findById(String productId) {
        return CompletableFuture.supplyAsync(() -> 
            productRepository.findById(productId)
                .orElseThrow(() -> new ProductNotFoundException(productId))
        );
    }
    
    // Fallback method
    public CompletableFuture<Product> getProductFallback(String productId, Exception e) {
        log.warn("Circuit breaker open for product {}, using fallback: {}", 
            productId, e.getMessage());
        
        // ลอง local cache ก่อน
        Product cached = localCache.getIfPresent(productId);
        if (cached != null) {
            return CompletableFuture.completedFuture(cached);
        }
        
        // Return stub product
        return CompletableFuture.completedFuture(
            Product.stub(productId, "Product temporarily unavailable")
        );
    }
}
```

---

## ขั้นตอนที่ 3402: Automated Load Testing ใน CI กับ k6 Gates

### k6 Load Test Script

```javascript
// tests/load/api-load-test.js
import http from 'k6/http';
import { check, sleep, group } from 'k6';
import { Rate, Trend, Counter } from 'k6/metrics';

// Custom metrics
const errorRate = new Rate('error_rate');
const apiLatency = new Trend('api_latency', true);
const successfulOrders = new Counter('successful_orders');

export const options = {
    stages: [
        { duration: '1m', target: 50 },    // Ramp up
        { duration: '3m', target: 100 },   // Normal load
        { duration: '1m', target: 200 },   // Stress test
        { duration: '2m', target: 100 },   // Back to normal
        { duration: '1m', target: 0 },     // Ramp down
    ],
    
    // Performance gates - test ล้มเหลวถ้าไม่ผ่าน
    thresholds: {
        http_req_duration: [
            'p(95)<500',     // 95% requests < 500ms
            'p(99)<1000',    // 99% requests < 1s
        ],
        http_req_failed: ['rate<0.01'],   // Error rate < 1%
        error_rate: ['rate<0.05'],         // Custom error rate < 5%
        api_latency: ['p(90)<300'],
    },
};

const BASE_URL = __ENV.BASE_URL || 'http://localhost:8080';

export default function() {
    group('Product browsing', () => {
        // Browse products
        const productsRes = http.get(`${BASE_URL}/api/v1/products?limit=20`);
        
        check(productsRes, {
            'products status 200': (r) => r.status === 200,
            'products response time < 200ms': (r) => r.timings.duration < 200,
        });
        
        errorRate.add(productsRes.status !== 200);
        apiLatency.add(productsRes.timings.duration);
        
        sleep(0.5);
        
        // Get product detail
        if (productsRes.status === 200) {
            const products = JSON.parse(productsRes.body);
            if (products.content && products.content.length > 0) {
                const productId = products.content[0].id;
                
                const detailRes = http.get(`${BASE_URL}/api/v1/products/${productId}`);
                check(detailRes, {
                    'product detail status 200': (r) => r.status === 200,
                    'product detail < 150ms': (r) => r.timings.duration < 150,
                });
            }
        }
    });
    
    sleep(1);
    
    group('User flow', () => {
        // Login
        const loginRes = http.post(`${BASE_URL}/api/v1/auth/login`, 
            JSON.stringify({
                email: 'test@example.com',
                password: 'testpassword'
            }),
            { headers: { 'Content-Type': 'application/json' } }
        );
        
        if (loginRes.status === 200) {
            const token = JSON.parse(loginRes.body).accessToken;
            const headers = { 
                'Authorization': `Bearer ${token}`,
                'Content-Type': 'application/json'
            };
            
            // Add to cart
            const cartRes = http.post(`${BASE_URL}/api/v1/cart/items`,
                JSON.stringify({ productId: 'prod-1', quantity: 1 }),
                { headers }
            );
            
            check(cartRes, {
                'add to cart status 200': (r) => r.status === 200,
            });
        }
    });
    
    sleep(2);
}

export function handleSummary(data) {
    return {
        'stdout': textSummary(data, { indent: ' ', enableColors: true }),
        'load-test-results.json': JSON.stringify(data),
    };
}
```

### CI Pipeline Integration

```yaml
# .github/workflows/load-test.yml
name: Load Testing Gate

on:
  pull_request:
    branches: [main, staging]
  schedule:
    - cron: '0 2 * * *'  # Daily at 2 AM

jobs:
  load-test:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Start application
      run: |
        docker-compose -f docker-compose.test.yml up -d
        ./scripts/wait-for-healthy.sh http://localhost:8080/actuator/health 120
    
    - name: Run k6 load test
      uses: grafana/k6-action@v0.3.1
      with:
        filename: tests/load/api-load-test.js
        flags: --out json=results/load-test.json
      env:
        BASE_URL: http://localhost:8080
    
    - name: Analyze results
      run: |
        python3 scripts/analyze-load-test.py results/load-test.json
    
    - name: Upload results
      uses: actions/upload-artifact@v3
      with:
        name: load-test-results
        path: results/
    
    - name: Fail if thresholds exceeded
      run: |
        if [ -f results/threshold-failures.txt ]; then
          echo "Load test FAILED! Thresholds exceeded:"
          cat results/threshold-failures.txt
          exit 1
        fi
```

---

## ขั้นตอนที่ 3403: Contract Testing กับ Pact Broker

Contract testing ตรวจสอบว่า consumer และ provider ตกลงเรื่อง API contract ตรงกัน

### Consumer Contract Test

```java
// consumer/OrderServiceContractTest.java
@ExtendWith(PactConsumerTestExt.class)
@PactTestFor(providerName = "ProductService", port = "8080")
class OrderServiceContractTest {
    
    @Pact(consumer = "OrderService")
    public RequestResponsePact createPact(PactDslWithProvider builder) {
        return builder
            .given("product exists with id prod-1")
            .uponReceiving("a request for product details")
                .path("/api/v1/products/prod-1")
                .method("GET")
                .headers(Map.of("Accept", "application/json"))
            .willRespondWith()
                .status(200)
                .headers(Map.of("Content-Type", "application/json;charset=UTF-8"))
                .body(new PactDslJsonBody()
                    .stringType("id", "prod-1")
                    .stringType("name", "Sample Product")
                    .decimalType("price", 199.99)
                    .booleanType("inStock", true)
                    .integerType("stockQuantity", 100))
            .toPact();
    }
    
    @Test
    @PactTestFor(pactMethod = "createPact")
    void testGetProductById(MockServer mockServer) {
        // Arrange
        WebClient client = WebClient.create(mockServer.getUrl());
        
        // Act
        Product product = client.get()
            .uri("/api/v1/products/prod-1")
            .retrieve()
            .bodyToMono(Product.class)
            .block();
        
        // Assert
        assertThat(product).isNotNull();
        assertThat(product.getId()).isEqualTo("prod-1");
        assertThat(product.getPrice()).isEqualByComparingTo("199.99");
    }
    
    @Pact(consumer = "OrderService")
    public RequestResponsePact createNotFoundPact(PactDslWithProvider builder) {
        return builder
            .given("product does not exist")
            .uponReceiving("a request for non-existent product")
                .path("/api/v1/products/nonexistent")
                .method("GET")
            .willRespondWith()
                .status(404)
                .body(new PactDslJsonBody()
                    .stringType("error", "Product not found")
                    .stringType("productId", "nonexistent"))
            .toPact();
    }
}
```

### Provider Verification

```java
// provider/ProductServicePactVerificationTest.java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Provider("ProductService")
@PactBroker(
    url = "${pact.broker.url}",
    authentication = @PactBrokerAuth(
        username = "${pact.broker.username}",
        password = "${pact.broker.password}"
    )
)
class ProductServicePactVerificationTest {
    
    @LocalServerPort
    private int port;
    
    @Autowired
    private ProductRepository productRepository;
    
    @BeforeEach
    void setUp(PactVerificationContext context) {
        context.setTarget(new HttpTestTarget("localhost", port));
    }
    
    @TestTemplate
    @ExtendWith(PactVerificationInvocationContextProvider.class)
    void pactVerificationTestTemplate(PactVerificationContext context) {
        context.verifyInteraction();
    }
    
    // State setup methods
    @State("product exists with id prod-1")
    public void productExistsState() {
        // Create test data
        productRepository.save(Product.builder()
            .id("prod-1")
            .name("Sample Product")
            .price(new BigDecimal("199.99"))
            .inStock(true)
            .stockQuantity(100)
            .build());
    }
    
    @State("product does not exist")
    public void productDoesNotExistState() {
        // Clean up any existing test product
        productRepository.deleteById("nonexistent");
    }
    
    @AfterEach
    void cleanUp() {
        productRepository.deleteById("prod-1");
    }
}
```

### Pact Broker CI Pipeline

```yaml
# .github/workflows/contract-tests.yml
name: Contract Testing

on:
  push:
    branches: [main, develop]
  pull_request:

jobs:
  consumer-tests:
    name: Consumer Contract Tests
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    
    - name: Run consumer contract tests
      run: mvn test -pl order-service -Dtest=*ContractTest
    
    - name: Publish pacts to broker
      run: |
        mvn pact:publish \
          -Dpact.broker.url=${{ secrets.PACT_BROKER_URL }} \
          -Dpact.broker.token=${{ secrets.PACT_BROKER_TOKEN }} \
          -Dpact.consumer.version=${{ github.sha }}

  provider-tests:
    name: Provider Verification
    needs: consumer-tests
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    
    - name: Verify contracts
      run: |
        mvn test -pl product-service -Dtest=*PactVerification \
          -Dpact.broker.url=${{ secrets.PACT_BROKER_URL }} \
          -Dpact.broker.token=${{ secrets.PACT_BROKER_TOKEN }}
    
    - name: Can I deploy?
      run: |
        pact-broker can-i-deploy \
          --pacticipant ProductService \
          --version ${{ github.sha }} \
          --to-environment production \
          --broker-base-url ${{ secrets.PACT_BROKER_URL }}
```

---

## ขั้นตอนที่ 3404: Synthetic Monitoring กับ Playwright

Synthetic monitoring ใช้ automated browser tests เพื่อ monitor production continuously

```typescript
// tests/synthetic/checkout-flow.spec.ts
import { test, expect } from '@playwright/test';

test.describe('Critical user flows - Production Monitoring', () => {
    
    test('Complete checkout flow', async ({ page }) => {
        const startTime = Date.now();
        
        // 1. Navigate to shop
        await page.goto(process.env.APP_URL || 'https://shophub.example.com');
        await expect(page).toHaveTitle(/ShopHub/);
        
        // 2. Search for product
        await page.fill('[data-testid="search-input"]', 'test product');
        await page.click('[data-testid="search-button"]');
        await expect(page.locator('[data-testid="search-results"]')).toBeVisible();
        
        // 3. Add to cart
        await page.click('[data-testid="product-card"]:first-child [data-testid="add-to-cart"]');
        await expect(page.locator('[data-testid="cart-count"]')).toContainText('1');
        
        // 4. Login
        await page.click('[data-testid="login-button"]');
        await page.fill('[data-testid="email-input"]', process.env.TEST_USER_EMAIL!);
        await page.fill('[data-testid="password-input"]', process.env.TEST_USER_PASSWORD!);
        await page.click('[data-testid="submit-login"]');
        
        await expect(page.locator('[data-testid="user-menu"]')).toBeVisible({ timeout: 5000 });
        
        // 5. Checkout
        await page.click('[data-testid="cart-icon"]');
        await page.click('[data-testid="checkout-button"]');
        
        // Check checkout page loaded
        await expect(page).toHaveURL(/\/checkout/);
        
        const flowDuration = Date.now() - startTime;
        console.log(`Checkout flow completed in ${flowDuration}ms`);
        
        // Assert performance SLO
        expect(flowDuration).toBeLessThan(30000); // 30s max
    });
    
    test('API health check', async ({ request }) => {
        const response = await request.get('/actuator/health');
        
        expect(response.status()).toBe(200);
        
        const health = await response.json();
        expect(health.status).toBe('UP');
        
        // Check specific components
        expect(health.components.db.status).toBe('UP');
        expect(health.components.redis.status).toBe('UP');
    });
    
    test('Search latency SLO', async ({ page }) => {
        await page.goto('/');
        
        const searchStart = Date.now();
        await page.fill('[data-testid="search-input"]', 'laptop');
        await page.click('[data-testid="search-button"]');
        await page.waitForResponse(res => res.url().includes('/api/v1/products/search'));
        
        const searchLatency = Date.now() - searchStart;
        expect(searchLatency).toBeLessThan(1000); // Search < 1s
    });
});
```

### Playwright CI/CD Schedule

```yaml
# .github/workflows/synthetic-monitoring.yml
name: Synthetic Monitoring

on:
  schedule:
    - cron: '*/15 * * * *'  # ทุก 15 นาที

jobs:
  synthetic-tests:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Install Playwright
      run: npx playwright install chromium
    
    - name: Run synthetic tests
      run: |
        npx playwright test tests/synthetic/ \
          --reporter=json \
          --output=synthetic-results.json
      env:
        APP_URL: https://shophub.example.com
        TEST_USER_EMAIL: ${{ secrets.SYNTHETIC_USER_EMAIL }}
        TEST_USER_PASSWORD: ${{ secrets.SYNTHETIC_USER_PASSWORD }}
    
    - name: Alert on failure
      if: failure()
      uses: slackapi/slack-github-action@v1.24.0
      with:
        payload: |
          {
            "text": "⚠️ Synthetic monitoring FAILED! Check production: https://shophub.example.com",
            "attachments": [{"color": "danger", "title": "Test run failed"}]
          }
      env:
        SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
```

---

## ขั้นตอนที่ 3405: Testing กับ Production Data Snapshots (Masked)

```java
// testdata/ProductionDataMaskingService.java
@Service
public class ProductionDataMaskingService {
    
    public void createMaskedSnapshot(String sourceDb, String targetDb) {
        log.info("กำลังสร้าง masked snapshot จาก {} ไป {}", sourceDb, targetDb);
        
        // 1. Copy production schema + data
        copySchema(sourceDb, targetDb);
        copyData(sourceDb, targetDb);
        
        // 2. Mask sensitive data
        maskPersonalData(targetDb);
        maskPaymentData(targetDb);
        maskCredentials(targetDb);
        
        log.info("Masked snapshot สร้างเสร็จแล้ว");
    }
    
    private void maskPersonalData(String db) {
        jdbcTemplate.update("""
            UPDATE users SET
                email = CONCAT('user', id, '@masked.example.com'),
                phone = CONCAT('09', SUBSTRING(MD5(RANDOM()::TEXT), 1, 8)),
                first_name = 'TestFirst' || id::TEXT,
                last_name = 'TestLast' || id::TEXT,
                address = 'Masked Address',
                national_id = REPEAT('*', 13)
            WHERE true
        """);
    }
    
    private void maskPaymentData(String db) {
        jdbcTemplate.update("""
            UPDATE payment_methods SET
                card_number_masked = CONCAT('****-****-****-', RIGHT(card_number, 4)),
                card_number = NULL,
                cvv = NULL,
                holder_name = 'MASKED HOLDER'
            WHERE true
        """);
    }
    
    private void maskCredentials(String db) {
        jdbcTemplate.update("""
            UPDATE users SET
                password_hash = '$2a$10$maskedPasswordHashForTesting...'
            WHERE true
        """);
    }
}
```

### Testcontainers กับ Production-like Data

```java
// test/integration/OrderServiceIntegrationTest.java
@SpringBootTest
@Testcontainers
class OrderServiceIntegrationTest {
    
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15")
        .withDatabaseName("shophub_test")
        .withUsername("test")
        .withPassword("test")
        .withInitScript("test-data/masked-production-snapshot.sql"); // ใช้ masked snapshot
    
    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }
    
    @Autowired
    private OrderService orderService;
    
    @Test
    void shouldCreateOrderWithProductionLikeData() {
        // Test ด้วยข้อมูลที่คล้าย production
        PlaceOrderCommand command = PlaceOrderCommand.builder()
            .customerId("user1@masked.example.com") // masked email
            .items(List.of(new OrderItem("prod-real-id-from-snapshot", 1)))
            .build();
        
        Order order = orderService.placeOrder(command);
        
        assertThat(order).isNotNull();
        assertThat(order.getStatus()).isEqualTo(OrderStatus.CONFIRMED);
    }
}
```

---

## ขั้นตอนที่ 3406: Shift-Left Security Testing ใน CI

"Shift-left" หมายถึงการทำ security testing ตั้งแต่ development phase แทนที่จะรอทำที่ production

### SAST (Static Application Security Testing)

```yaml
# .github/workflows/security.yml
name: Security Testing

on: [push, pull_request]

jobs:
  sast:
    name: Static Security Analysis
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v4
    
    # SpotBugs + Find Security Bugs
    - name: Run SpotBugs Security Analysis
      run: mvn spotbugs:check -Dspotbugs.plugins=com.h3xstream.findsecbugs:findsecbugs-plugin:1.12.0
    
    # OWASP Dependency Check
    - name: Check Dependencies for Vulnerabilities
      run: |
        mvn org.owasp:dependency-check-maven:check \
          -DfailBuildOnCVSS=7 \
          -DsuppressionsLocation=.owasp-suppressions.xml
    
    # Trivy container scan
    - name: Scan Docker image
      uses: aquasecurity/trivy-action@master
      with:
        image-ref: 'shophub/api:${{ github.sha }}'
        format: 'sarif'
        output: 'trivy-results.sarif'
        severity: 'CRITICAL,HIGH'
        exit-code: '1'
    
    # Secret scanning
    - name: Detect secrets
      uses: trufflesecurity/trufflehog@main
      with:
        path: ./
        base: main
        head: HEAD
        extra_args: --debug --only-verified
```

### Security Unit Tests

```java
// security/SecurityTest.java
@SpringBootTest
@AutoConfigureMockMvc
class SecurityTest {
    
    @Autowired
    private MockMvc mockMvc;
    
    @Test
    @DisplayName("ต้องป้องกัน SQL Injection")
    void shouldPreventSqlInjection() throws Exception {
        String maliciousInput = "'; DROP TABLE users; --";
        
        mockMvc.perform(get("/api/v1/products/search")
                .param("q", maliciousInput))
            .andExpect(status().isOk()) // ต้องไม่ crash
            .andExpect(jsonPath("$.error").doesNotExist()); // ไม่มี SQL error
    }
    
    @Test
    @DisplayName("ต้องป้องกัน XSS")
    void shouldPreventXss() throws Exception {
        String xssPayload = "<script>alert('XSS')</script>";
        
        mockMvc.perform(post("/api/v1/products")
                .contentType(MediaType.APPLICATION_JSON)
                .content("""
                    {
                        "name": "%s",
                        "price": 100
                    }
                    """.formatted(xssPayload))
                .with(user("admin").roles("ADMIN")))
            .andExpect(status().isCreated())
            .andExpect(jsonPath("$.name").value(not(containsString("<script>"))));
    }
    
    @Test
    @DisplayName("ต้องป้องกัน Mass Assignment")
    void shouldPreventMassAssignment() throws Exception {
        // User ไม่ควรสามารถตั้งค่า admin=true ได้
        mockMvc.perform(put("/api/v1/users/me")
                .contentType(MediaType.APPLICATION_JSON)
                .content("""
                    {
                        "name": "Test",
                        "role": "ADMIN",
                        "admin": true
                    }
                    """)
                .with(user("regular-user")))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.role").value(not("ADMIN")));
    }
    
    @Test
    @DisplayName("ต้องมี rate limiting")
    void shouldEnforceRateLimiting() throws Exception {
        // ส่ง requests เยอะๆ แล้วต้องได้ 429
        for (int i = 0; i < 100; i++) {
            mockMvc.perform(post("/api/v1/auth/login")
                    .contentType(MediaType.APPLICATION_JSON)
                    .content("""{"email":"test@test.com","password":"wrong"}"""));
        }
        
        // Request สุดท้ายควรได้ 429 Too Many Requests
        mockMvc.perform(post("/api/v1/auth/login")
                .contentType(MediaType.APPLICATION_JSON)
                .content("""{"email":"test@test.com","password":"wrong"}"""))
            .andExpect(status().isTooManyRequests());
    }
    
    @Test
    @DisplayName("ต้องป้องกัน IDOR (Insecure Direct Object Reference)")
    void shouldPreventIdor() throws Exception {
        // User A ไม่ควรเข้าถึง orders ของ User B
        String userAToken = getTokenForUser("user-a");
        String userBOrderId = "order-of-user-b";
        
        mockMvc.perform(get("/api/v1/orders/" + userBOrderId)
                .header("Authorization", "Bearer " + userAToken))
            .andExpect(status().isForbidden()); // ต้องได้ 403
    }
}
```

---

## ขั้นตอนที่ 3407: Testcontainers Compose สำหรับ Integration Tests

```java
// test/config/IntegrationTestConfig.java
@TestConfiguration
public class IntegrationTestConfig {
    
    // Testcontainers Compose - ใช้ docker-compose.test.yml
    @Container
    static DockerComposeContainer<?> compose = new DockerComposeContainer<>(
        new File("docker-compose.test.yml"))
        .withExposedService("postgres", 5432, 
            Wait.forListeningPort().withStartupTimeout(Duration.ofMinutes(2)))
        .withExposedService("redis", 6379, 
            Wait.forListeningPort())
        .withExposedService("kafka", 9092, 
            Wait.forListeningPort());
    
    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", 
            () -> "jdbc:postgresql://" + 
                compose.getServiceHost("postgres", 5432) + ":" + 
                compose.getServicePort("postgres", 5432) + "/shophub");
        
        registry.add("spring.redis.host", 
            () -> compose.getServiceHost("redis", 6379));
        registry.add("spring.redis.port", 
            () -> compose.getServicePort("redis", 6379));
        
        registry.add("spring.kafka.bootstrap-servers", 
            () -> compose.getServiceHost("kafka", 9092) + ":" + 
                compose.getServicePort("kafka", 9092));
    }
}

// docker-compose.test.yml
```

```yaml
# docker-compose.test.yml
version: '3.8'
services:
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: shophub
      POSTGRES_USER: test
      POSTGRES_PASSWORD: test
    ports:
    - "5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U test"]
      interval: 5s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    ports:
    - "6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 5s
      retries: 5

  kafka:
    image: confluentinc/cp-kafka:7.5.0
    environment:
      KAFKA_NODE_ID: 1
      KAFKA_PROCESS_ROLES: 'broker,controller'
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: 'CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT'
      KAFKA_CONTROLLER_QUORUM_VOTERS: '1@kafka:9093'
      KAFKA_LISTENERS: 'PLAINTEXT://kafka:9092,CONTROLLER://kafka:9093'
      KAFKA_ADVERTISED_LISTENERS: 'PLAINTEXT://kafka:9092'
      KAFKA_CONTROLLER_LISTENER_NAMES: 'CONTROLLER'
      KAFKA_AUTO_CREATE_TOPICS_ENABLE: 'true'
    ports:
    - "9092"
```

### Full Integration Test

```java
// test/OrderFlowIntegrationTest.java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Import(IntegrationTestConfig.class)
@ActiveProfiles("test")
class OrderFlowIntegrationTest {
    
    @Autowired
    private TestRestTemplate restTemplate;
    
    @Autowired
    private KafkaConsumerTestHelper kafkaHelper;
    
    @Autowired
    private OrderRepository orderRepository;
    
    @Test
    @DisplayName("Complete order flow: browse → add to cart → checkout → payment → confirmation")
    void completeOrderFlow() throws InterruptedException {
        // 1. Browse products
        ResponseEntity<ProductPage> products = restTemplate.getForEntity(
            "/api/v1/products?category=electronics", ProductPage.class);
        assertThat(products.getStatusCode()).isEqualTo(HttpStatus.OK);
        assertThat(products.getBody().getContent()).isNotEmpty();
        
        String productId = products.getBody().getContent().get(0).getId();
        
        // 2. Login
        ResponseEntity<AuthResponse> login = restTemplate.postForEntity(
            "/api/v1/auth/login",
            new LoginRequest("customer@test.com", "password123"),
            AuthResponse.class);
        
        String token = login.getBody().getAccessToken();
        HttpHeaders headers = new HttpHeaders();
        headers.setBearerAuth(token);
        
        // 3. Place order
        PlaceOrderRequest orderRequest = PlaceOrderRequest.builder()
            .items(List.of(new OrderItemRequest(productId, 1)))
            .paymentMethodId("pm-test-visa")
            .shippingAddressId("addr-1")
            .build();
        
        ResponseEntity<Order> orderResponse = restTemplate.exchange(
            "/api/v1/orders",
            HttpMethod.POST,
            new HttpEntity<>(orderRequest, headers),
            Order.class);
        
        assertThat(orderResponse.getStatusCode()).isEqualTo(HttpStatus.CREATED);
        String orderId = orderResponse.getBody().getId();
        
        // 4. Verify Kafka events
        kafkaHelper.waitForMessage("order-placed", orderId, Duration.ofSeconds(10));
        
        // 5. Verify order in DB
        Order savedOrder = orderRepository.findById(orderId).orElseThrow();
        assertThat(savedOrder.getStatus()).isEqualTo(OrderStatus.CONFIRMED);
        
        // 6. Get order details
        ResponseEntity<Order> getOrder = restTemplate.exchange(
            "/api/v1/orders/" + orderId,
            HttpMethod.GET,
            new HttpEntity<>(headers),
            Order.class);
        
        assertThat(getOrder.getStatusCode()).isEqualTo(HttpStatus.OK);
        assertThat(getOrder.getBody().getItems()).hasSize(1);
        assertThat(getOrder.getBody().getItems().get(0).getProductId()).isEqualTo(productId);
    }
}
```

---

## ขั้นตอนที่ 3408-3440: Testing Best Practices Summary

### Test Pyramid

```
               /\
              /  \
             / E2E \      ← น้อย, ช้า, แพง, confidence สูง
            /--------\
           / Integration\  ← ปานกลาง
          /--------------\
         /  Unit Tests    \  ← เยอะ, เร็ว, ถูก
        /------------------\
```

### Testing Checklist

```
Unit Tests (70%):
  ✅ Service logic ทุก branch
  ✅ Domain model invariants
  ✅ Utility functions
  ✅ Exception handling
  ✅ Boundary conditions

Integration Tests (20%):
  ✅ Repository operations
  ✅ API endpoints (MockMvc)
  ✅ Message queue integration
  ✅ Cache behavior
  ✅ External service mocking

E2E Tests (10%):
  ✅ Critical user flows
  ✅ Cross-service integration
  ✅ Database seeded with test data
  ✅ Real browser testing (Playwright)

Special Tests:
  ✅ Contract tests (Pact)
  ✅ Performance tests (k6)
  ✅ Security tests (SAST + DAST)
  ✅ Chaos tests
  ✅ Synthetic monitoring
```

### Test Performance Tips

```java
// ใช้ @DirtiesContext อย่างระมัดระวัง - ช้ามาก
// BAD:
@DirtiesContext // recreates Spring context every test
class SlowTest { }

// GOOD: ใช้ @Transactional สำหรับ rollback แทน
@Transactional // rollback after each test
class FastTest { }

// ใช้ TestEntityManager แทน JpaRepository ใน test
@DataJpaTest
class RepositoryTest {
    @Autowired
    TestEntityManager entityManager;
    
    @Test
    void shouldFindByEmail() {
        User user = entityManager.persistAndFlush(new User("test@test.com"));
        Optional<User> found = userRepository.findByEmail("test@test.com");
        assertThat(found).isPresent();
    }
}
```

---

## สรุป

Part 95 ครอบคลุม Advanced Testing Strategies:

1. **Chaos Testing** - Chaos Monkey ทดสอบ resilience
2. **Load Testing CI Gates** - k6 ป้องกัน performance regression
3. **Contract Testing** - Pact ตรวจสอบ API contracts
4. **Synthetic Monitoring** - Playwright monitor production
5. **Production Data** - Masked snapshots สำหรับ realistic tests
6. **Shift-Left Security** - SAST, dependency check ใน CI
7. **Testcontainers Compose** - Integration tests ที่สมจริง

Testing ที่ดีคือ safety net ที่ช่วยให้ ship code ได้อย่างมั่นใจ

---

*[← Part 94: Machine Learning](./part-94-machine-learning.md) | [Part 96: Capstone Design →](./part-96-capstone-design.md)*

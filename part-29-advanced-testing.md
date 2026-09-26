# Part 29: Advanced Testing
## ขั้นตอนที่ 796-825

> **ระดับ:** สูง (Advanced)  
> **เวลาเรียน:** 5-6 ชั่วโมง  
> **เป้าหมาย:** Contract Testing, Property-Based Testing, Performance Testing

---

## ขั้นตอนที่ 796: Integration Testing Best Practices

```java
// Shared TestContainers across tests (performance)
@SpringBootTest
@Testcontainers
@ActiveProfiles("test")
public abstract class BaseIntegrationTest {
    
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine")
        .withDatabaseName("testdb")
        .withUsername("test")
        .withPassword("test")
        .withReuse(true);  // Reuse container between test classes
    
    @Container
    static GenericContainer<?> redis = new GenericContainer<>("redis:7-alpine")
        .withExposedPorts(6379)
        .withReuse(true);
    
    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
        registry.add("spring.data.redis.host", redis::getHost);
        registry.add("spring.data.redis.port", () -> redis.getMappedPort(6379));
    }
}

// Use in test classes
class ProductServiceIntegrationTest extends BaseIntegrationTest {
    
    @Autowired
    private ProductService productService;
    
    @Autowired
    private ProductRepository productRepository;
    
    @Test
    @Transactional
    void createAndFind_ShouldWork() {
        // ...
    }
}
```

---

## ขั้นตอนที่ 797: Pact - Consumer-Driven Contract Testing

```xml
<dependency>
    <groupId>au.com.dius.pact.consumer</groupId>
    <artifactId>junit5</artifactId>
    <version>4.6.7</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>au.com.dius.pact.provider</groupId>
    <artifactId>junit5spring</artifactId>
    <version>4.6.7</version>
    <scope>test</scope>
</dependency>
```

```java
// Consumer side (order-service consuming user-service)
@ExtendWith(PactConsumerTestExt.class)
@PactTestFor(providerName = "user-service")
class UserClientPactTest {
    
    @Pact(consumer = "order-service")
    public RequestResponsePact getUserByIdPact(PactDslWithProvider builder) {
        return builder
            .given("user with id 1 exists")
            .uponReceiving("get user by id 1")
            .path("/api/v1/users/1")
            .method("GET")
            .headers("Authorization", "Bearer token")
            .willRespondWith()
            .status(200)
            .headers(Map.of("Content-Type", "application/json"))
            .body(new PactDslJsonBody()
                .numberType("id", 1)
                .stringType("username", "john")
                .stringType("email", "john@test.com")
                .booleanType("active", true))
            .toPact();
    }
    
    @Test
    @PactTestFor(pactMethod = "getUserByIdPact")
    void testGetUserById(MockServer mockServer) {
        UserClient client = new UserClient(mockServer.getUrl());
        ApiResponse<UserResponse> response = client.getUserById(1L);
        
        assertThat(response.getData().getId()).isEqualTo(1L);
        assertThat(response.getData().getUsername()).isEqualTo("john");
    }
}

// Provider side (user-service verifying contract)
@Provider("user-service")
@PactBroker(url = "http://pact-broker:9292")
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class UserServicePactProviderTest {
    
    @LocalServerPort
    int port;
    
    @BeforeEach
    void setUp(PactVerificationContext context) {
        context.setTarget(new HttpTestTarget("localhost", port));
    }
    
    @TestTemplate
    @ExtendWith(PactVerificationInvocationContextProvider.class)
    void verifyPact(PactVerificationContext context) {
        context.verifyInteraction();
    }
    
    @State("user with id 1 exists")
    void setupUser() {
        // Create test user in DB
        userRepository.save(User.builder()
            .id(1L).username("john").email("john@test.com")
            .build());
    }
}
```

---

## ขั้นตอนที่ 798: Property-Based Testing

```xml
<dependency>
    <groupId>net.jqwik</groupId>
    <artifactId>jqwik</artifactId>
    <version>1.8.2</version>
    <scope>test</scope>
</dependency>
```

```java
@ExtendWith(PactVerificationInvocationContextProvider.class)
class PricingServicePropertyTest {
    
    @Property
    void applyDiscount_ShouldNeverBeNegative(
        @ForAll @BigRange(min = "0.01", max = "999999.99") BigDecimal price,
        @ForAll @IntRange(min = 0, max = 100) int discountPercent
    ) {
        BigDecimal discounted = pricingService.applyDiscount(price, discountPercent);
        assertThat(discounted).isGreaterThanOrEqualTo(BigDecimal.ZERO);
    }
    
    @Property
    void applyDiscount_100Percent_ShouldBeZero(
        @ForAll @BigRange(min = "0.01", max = "999999.99") BigDecimal price
    ) {
        BigDecimal discounted = pricingService.applyDiscount(price, 100);
        assertThat(discounted).isEqualByComparingTo(BigDecimal.ZERO);
    }
    
    @Property
    void applyDiscount_0Percent_ShouldBeOriginal(
        @ForAll @BigRange(min = "0.01", max = "999999.99") BigDecimal price
    ) {
        BigDecimal discounted = pricingService.applyDiscount(price, 0);
        assertThat(discounted).isEqualByComparingTo(price);
    }
    
    @Property
    void palindrome_ShouldReverse(@ForAll String text) {
        String reversed = stringService.reverse(text);
        assertThat(stringService.reverse(reversed)).isEqualTo(text);
    }
}
```

---

## ขั้นตอนที่ 799: End-to-End Testing with REST-Assured

```xml
<dependency>
    <groupId>io.rest-assured</groupId>
    <artifactId>rest-assured</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>io.rest-assured</groupId>
    <artifactId>spring-mock-mvc</artifactId>
    <scope>test</scope>
</dependency>
```

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class ProductApiE2ETest extends BaseIntegrationTest {
    
    @LocalServerPort
    int port;
    
    @Autowired
    AuthService authService;
    
    String accessToken;
    
    @BeforeEach
    void setUp() {
        RestAssured.port = port;
        RestAssured.basePath = "/api/v1";
        
        // Login
        AuthResponse auth = authService.login(new LoginRequest("admin@test.com", "Admin123!"));
        accessToken = auth.accessToken();
    }
    
    @Test
    @Order(1)
    void createProduct_ShouldReturnCreated() {
        given()
            .auth().oauth2(accessToken)
            .contentType(ContentType.JSON)
            .body("""
                {
                  "name": "iPhone 15",
                  "price": 45000,
                  "stock": 100
                }
                """)
        .when()
            .post("/products")
        .then()
            .statusCode(201)
            .body("success", equalTo(true))
            .body("data.name", equalTo("iPhone 15"))
            .body("data.price", equalTo(45000.0f))
            .header("Location", containsString("/api/v1/products/"));
    }
    
    @Test
    void searchProducts_ShouldReturnPaginatedResults() {
        given()
            .queryParam("keyword", "iPhone")
            .queryParam("page", "0")
            .queryParam("size", "10")
        .when()
            .get("/products")
        .then()
            .statusCode(200)
            .body("data.content", not(empty()))
            .body("data.meta.page", equalTo(0))
            .body("data.meta.size", equalTo(10));
    }
    
    @Test
    void createProduct_WithoutAuth_ShouldReturn401() {
        given()
            .contentType(ContentType.JSON)
            .body("""{"name": "Test", "price": 100, "stock": 1}""")
        .when()
            .post("/products")
        .then()
            .statusCode(401);
    }
}
```

---

## ขั้นตอนที่ 800: Test Data Management

```java
// @Sql to load/clean test data
@SpringBootTest
@Sql(scripts = "/test-data/users.sql", executionPhase = Sql.ExecutionPhase.BEFORE_TEST_METHOD)
@Sql(scripts = "/test-data/cleanup.sql", executionPhase = Sql.ExecutionPhase.AFTER_TEST_METHOD)
class UserServiceTest { }

// DatabaseRider (easier data setup)
@DBUnit(schema = "public", caseSensitiveTableNames = false)
@DataSet(value = "users.yml", strategy = SeedStrategy.CLEAN_INSERT)
@ExpectedDataSet("users-after.yml")
@Test
void updateUser_ShouldChangeData() { }

// Faker for realistic test data
@Component
public class TestDataFactory {
    
    private final Faker faker = new Faker();
    
    public CreateUserRequest randomUser() {
        return new CreateUserRequest(
            faker.internet().username(),
            faker.internet().emailAddress(),
            "Password123!",
            faker.name().firstName(),
            faker.name().lastName(),
            null
        );
    }
    
    public CreateProductRequest randomProduct(Long categoryId) {
        return new CreateProductRequest(
            faker.commerce().productName(),
            faker.lorem().paragraph(),
            BigDecimal.valueOf(faker.number().randomDouble(2, 10, 100000)),
            faker.number().numberBetween(0, 1000),
            null,
            categoryId
        );
    }
}
```

---

## ขั้นตอนที่ 801-825: Mutation Testing

```xml
<!-- PIT Mutation Testing -->
<plugin>
    <groupId>org.pitest</groupId>
    <artifactId>pitest-maven</artifactId>
    <version>1.15.2</version>
    <dependencies>
        <dependency>
            <groupId>org.pitest</groupId>
            <artifactId>pitest-junit5-plugin</artifactId>
            <version>1.2.1</version>
        </dependency>
    </dependencies>
    <configuration>
        <targetClasses>
            <param>com.myapp.service.*</param>
        </targetClasses>
        <targetTests>
            <param>com.myapp.service.*Test</param>
        </targetTests>
        <mutationThreshold>70</mutationThreshold>
        <coverageThreshold>80</coverageThreshold>
    </configuration>
</plugin>
```

```bash
# Run mutation testing
mvn test-compile org.pitest:pitest-maven:mutationCoverage

# Report at: target/pit-reports/index.html
# Mutation Score: percentage of mutants killed by tests
# Higher = Better tests quality
```

### Testing Strategy Summary

```
Testing Pyramid:
  Unit Tests (70%):      Fast, isolated, pure logic
  Integration Tests (20%): Repository, Service with real DB
  E2E Tests (10%):       Full API with real server

Key principles:
  ✅ Tests should be deterministic (same result every run)
  ✅ Tests should be independent (no shared state)
  ✅ Tests should be fast (< 10 min total)
  ✅ Write tests before fixing bugs (TDD)
  ✅ Coverage: 80% line, 70% mutation score

Test naming: methodName_Given_Expected
  createUser_WhenEmailExists_ShouldThrowException
  findById_WhenProductExists_ShouldReturnProduct
```

---

*[← Part 28: Performance](./part-28-performance.md) | [Part 30: AOP Programming →](./part-30-aop-programming.md)*

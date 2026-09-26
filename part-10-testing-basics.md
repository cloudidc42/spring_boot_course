# Part 10: การทดสอบ (Testing) พื้นฐาน
## ขั้นตอนที่ 201-230

> **ระดับ:** พื้นฐาน (Beginner)  
> **เวลาเรียน:** 4-5 ชั่วโมง  
> **เป้าหมาย:** เขียน Tests ที่ดีและมีประสิทธิภาพด้วย JUnit 5 และ Mockito

---

## ขั้นตอนที่ 201: Testing Pyramid

```
                    ┌────────────────┐
                    │   E2E Tests    │  ← น้อย แต่ครอบคลุม user journey
                    └────────────────┘
                 ┌──────────────────────┐
                 │  Integration Tests   │  ← ปานกลาง
                 └──────────────────────┘
            ┌──────────────────────────────┐
            │         Unit Tests           │  ← มาก, เร็ว, ถูก
            └──────────────────────────────┘

Unit Tests:      ทดสอบแต่ละ function/method แยกกัน
Integration Tests: ทดสอบการทำงานร่วมกันของหลาย components
E2E Tests:       ทดสอบ user journey ทั้งหมด
```

---

## ขั้นตอนที่ 202: Test Dependencies

```xml
<!-- pom.xml - spring-boot-starter-test รวมทุกอย่าง -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>

<!-- รวม:
  - JUnit 5 (Jupiter) - test framework
  - Mockito - mocking framework
  - MockMvc - Spring MVC testing
  - AssertJ - fluent assertions
  - Hamcrest - matchers
  - JsonPath - JSON testing
  - H2 - in-memory database
  - TestContainers support
-->

<!-- เพิ่ม TestContainers สำหรับ integration tests -->
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>junit-jupiter</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>postgresql</artifactId>
    <scope>test</scope>
</dependency>
```

---

## ขั้นตอนที่ 203: JUnit 5 Basics

```java
import org.junit.jupiter.api.*;
import static org.junit.jupiter.api.Assertions.*;

class CalculatorTest {
    
    private Calculator calculator;
    
    // @BeforeAll - รัน 1 ครั้งก่อนทุก tests ใน class
    @BeforeAll
    static void setUpAll() {
        System.out.println("Setting up test class");
    }
    
    // @BeforeEach - รัน ก่อน test แต่ละ method
    @BeforeEach
    void setUp() {
        calculator = new Calculator();
    }
    
    // @AfterEach - รัน หลัง test แต่ละ method
    @AfterEach
    void tearDown() {
        // cleanup
    }
    
    // @AfterAll - รัน 1 ครั้งหลังทุก tests ใน class
    @AfterAll
    static void tearDownAll() {
        System.out.println("Test class complete");
    }
    
    // @Test - basic test
    @Test
    void add_ShouldReturnSum() {
        int result = calculator.add(2, 3);
        assertEquals(5, result);
    }
    
    // @DisplayName - ชื่อที่แสดงใน test report
    @Test
    @DisplayName("Should throw exception when dividing by zero")
    void divide_ByZero_ShouldThrowException() {
        assertThrows(ArithmeticException.class, () -> {
            calculator.divide(10, 0);
        });
    }
    
    // @Disabled - skip test
    @Test
    @Disabled("Not implemented yet")
    void futureFeatureTest() { }
    
    // @Tag - group tests
    @Test
    @Tag("slow")
    void slowTest() { }
    
    // @RepeatedTest - รัน test หลายครั้ง
    @RepeatedTest(5)
    void repeatedTest(RepetitionInfo info) {
        System.out.println("Running iteration: " + info.getCurrentRepetition());
    }
    
    // @ParameterizedTest - ทดสอบด้วยข้อมูลหลายชุด
    @ParameterizedTest
    @ValueSource(ints = {1, 2, 3, 4, 5})
    void isPositive_ShouldReturnTrue(int number) {
        assertTrue(calculator.isPositive(number));
    }
    
    @ParameterizedTest
    @CsvSource({
        "2, 3, 5",
        "0, 0, 0",
        "-1, 1, 0",
        "100, 200, 300"
    })
    void add_WithMultipleInputs(int a, int b, int expected) {
        assertEquals(expected, calculator.add(a, b));
    }
    
    @ParameterizedTest
    @MethodSource("provideAddTestData")
    void add_WithMethodSource(int a, int b, int expected) {
        assertEquals(expected, calculator.add(a, b));
    }
    
    static Stream<Arguments> provideAddTestData() {
        return Stream.of(
            Arguments.of(2, 3, 5),
            Arguments.of(0, 0, 0),
            Arguments.of(-1, 1, 0)
        );
    }
}
```

---

## ขั้นตอนที่ 204: AssertJ Assertions (แนะนำมากกว่า JUnit assertions)

```java
import static org.assertj.core.api.Assertions.*;

class AssertJExamplesTest {
    
    @Test
    void stringAssertions() {
        String name = "John Doe";
        
        assertThat(name)
            .isNotNull()
            .isNotEmpty()
            .startsWith("John")
            .endsWith("Doe")
            .contains("n D")
            .hasSize(8)
            .isEqualTo("John Doe");
    }
    
    @Test
    void numberAssertions() {
        int score = 85;
        
        assertThat(score)
            .isGreaterThan(0)
            .isLessThan(100)
            .isGreaterThanOrEqualTo(85)
            .isBetween(80, 90)
            .isEqualTo(85);
    }
    
    @Test
    void listAssertions() {
        List<String> fruits = List.of("Apple", "Banana", "Cherry");
        
        assertThat(fruits)
            .isNotEmpty()
            .hasSize(3)
            .contains("Apple", "Banana")
            .containsExactly("Apple", "Banana", "Cherry")
            .doesNotContain("Mango")
            .allMatch(f -> f.length() > 3)
            .anyMatch(f -> f.startsWith("A"));
    }
    
    @Test
    void objectAssertions() {
        User user = new User(1L, "john", "john@test.com");
        
        assertThat(user)
            .isNotNull()
            .hasFieldOrPropertyWithValue("username", "john")
            .hasFieldOrPropertyWithValue("email", "john@test.com");
        
        // Extracting fields
        assertThat(user)
            .extracting(User::getUsername, User::getEmail)
            .containsExactly("john", "john@test.com");
    }
    
    @Test
    void exceptionAssertions() {
        assertThatThrownBy(() -> {
            throw new RuntimeException("Test error");
        })
        .isInstanceOf(RuntimeException.class)
        .hasMessage("Test error")
        .hasMessageContaining("error");
        
        // Alternative
        assertThatExceptionOfType(RuntimeException.class)
            .isThrownBy(() -> {
                throw new RuntimeException("Test error");
            })
            .withMessage("Test error");
        
        // No exception
        assertThatNoException()
            .isThrownBy(() -> {
                // should not throw
            });
    }
}
```

---

## ขั้นตอนที่ 205: Mockito - Mocking Framework

```java
import org.mockito.Mock;
import org.mockito.InjectMocks;
import org.mockito.junit.jupiter.MockitoExtension;
import static org.mockito.Mockito.*;
import static org.mockito.BDDMockito.*;

@ExtendWith(MockitoExtension.class)  // ← ใช้ Mockito ใน JUnit 5
class UserServiceTest {
    
    @Mock
    private UserRepository userRepository;   // Mock
    
    @Mock
    private EmailService emailService;        // Mock
    
    @Mock
    private PasswordEncoder passwordEncoder;  // Mock
    
    @InjectMocks  // สร้าง UserService และ inject mocks
    private UserService userService;
    
    @Test
    void createUser_WhenValidRequest_ShouldCreateSuccessfully() {
        // Arrange (Given)
        CreateUserRequest request = new CreateUserRequest(
            "john", "john@test.com", "Password123!", "John", "Doe", null
        );
        
        User savedUser = User.builder()
            .id(1L)
            .username("john")
            .email("john@test.com")
            .build();
        
        // Stubbing (กำหนดพฤติกรรม mock)
        when(userRepository.existsByEmail("john@test.com")).thenReturn(false);
        when(userRepository.existsByUsername("john")).thenReturn(false);
        when(passwordEncoder.encode("Password123!")).thenReturn("$2a$12$hashedPassword");
        when(userRepository.save(any(User.class))).thenReturn(savedUser);
        
        // Act (When)
        UserResponse result = userService.create(request);
        
        // Assert (Then)
        assertThat(result).isNotNull();
        assertThat(result.username()).isEqualTo("john");
        assertThat(result.email()).isEqualTo("john@test.com");
        
        // Verify interactions
        verify(userRepository).existsByEmail("john@test.com");
        verify(userRepository).existsByUsername("john");
        verify(passwordEncoder).encode("Password123!");
        verify(userRepository).save(any(User.class));
        verify(emailService).sendWelcomeEmail("john@test.com");
        
        // Verify not called
        verifyNoMoreInteractions(emailService);
    }
    
    @Test
    void createUser_WhenEmailExists_ShouldThrowDuplicateException() {
        // Arrange
        CreateUserRequest request = new CreateUserRequest(
            "john", "existing@test.com", "Password123!", null, null, null
        );
        
        when(userRepository.existsByEmail("existing@test.com")).thenReturn(true);
        
        // Act & Assert
        assertThatThrownBy(() -> userService.create(request))
            .isInstanceOf(DuplicateResourceException.class)
            .hasMessageContaining("existing@test.com");
        
        // ตรวจสอบว่า save ไม่ถูกเรียก
        verify(userRepository, never()).save(any());
        verify(emailService, never()).sendWelcomeEmail(any());
    }
}
```

---

## ขั้นตอนที่ 206: Mockito Stubbing Patterns

```java
@Test
void mockitoStubbing() {
    
    // 1. thenReturn - return ค่า
    when(repo.findById(1L)).thenReturn(Optional.of(user));
    
    // 2. thenReturn หลายค่า (call ต่อไป return ค่าต่างกัน)
    when(repo.findById(any())).thenReturn(
        Optional.of(user1),    // call ที่ 1
        Optional.of(user2),    // call ที่ 2
        Optional.empty()       // call ที่ 3+
    );
    
    // 3. thenThrow - throw exception
    when(repo.findById(999L)).thenThrow(new RuntimeException("Not found"));
    
    // 4. thenAnswer - dynamic response
    when(repo.save(any())).thenAnswer(invocation -> {
        User user = invocation.getArgument(0);
        user.setId(1L);  // Simulate auto-generated ID
        return user;
    });
    
    // 5. doReturn, doThrow สำหรับ void methods
    doNothing().when(emailService).sendEmail(any());
    doThrow(new RuntimeException("Email failed")).when(emailService).sendEmail("bad@test.com");
    
    // 6. ArgumentCaptor - ดักจับ arguments
    ArgumentCaptor<User> userCaptor = ArgumentCaptor.forClass(User.class);
    
    userService.create(request);
    
    verify(repo).save(userCaptor.capture());
    User capturedUser = userCaptor.getValue();
    assertThat(capturedUser.getEmail()).isEqualTo("john@test.com");
    
    // 7. BDD style
    given(repo.findById(1L)).willReturn(Optional.of(user));
    // then()
    then(repo).should().findById(1L);
    
    // 8. Matchers
    when(repo.findById(anyLong())).thenReturn(Optional.of(user));
    when(repo.findByEmail(anyString())).thenReturn(Optional.empty());
    when(repo.save(argThat(u -> u.getEmail().endsWith("@test.com"))))
        .thenReturn(user);
}
```

---

## ขั้นตอนที่ 207: @SpringBootTest

```java
// Full integration test - โหลด complete ApplicationContext
@SpringBootTest
class UserServiceIntegrationTest {
    
    @Autowired
    private UserService userService;
    
    @Autowired
    private UserRepository userRepository;
    
    @Test
    @Transactional
    void createUser_ShouldPersistToDatabase() {
        // Arrange
        CreateUserRequest request = new CreateUserRequest(
            "john_" + UUID.randomUUID(), // unique username
            "john_" + UUID.randomUUID() + "@test.com", // unique email
            "Password123!", "John", "Doe", null
        );
        
        // Act
        UserResponse result = userService.create(request);
        
        // Assert
        assertThat(result.id()).isNotNull();
        
        // Verify in database
        Optional<User> found = userRepository.findById(result.id());
        assertThat(found).isPresent();
        assertThat(found.get().getEmail()).isEqualTo(request.email());
    }
}
```

```java
// @SpringBootTest กับ WebEnvironment
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class UserApiIntegrationTest {
    
    @Autowired
    private TestRestTemplate restTemplate;
    
    @LocalServerPort
    private int port;
    
    @Test
    void getUser_ShouldReturn200() {
        ResponseEntity<UserResponse> response = restTemplate.getForEntity(
            "http://localhost:" + port + "/api/v1/users/1",
            UserResponse.class
        );
        
        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.OK);
        assertThat(response.getBody()).isNotNull();
    }
}
```

---

## ขั้นตอนที่ 208: @WebMvcTest (Controller Layer Testing)

```java
// @WebMvcTest - ทดสอบ Controller layer เท่านั้น
// ไม่โหลด @Service, @Repository
@WebMvcTest(UserController.class)
class UserControllerTest {
    
    @Autowired
    private MockMvc mockMvc;
    
    @Autowired
    private ObjectMapper objectMapper;
    
    @MockBean  // Mock สำหรับ Spring Context
    private UserService userService;
    
    @Test
    void getUser_WhenExists_ShouldReturn200() throws Exception {
        // Arrange
        UserResponse user = new UserResponse(
            1L, "john", "john@test.com", "John", "Doe", null, "USER", true, LocalDateTime.now()
        );
        when(userService.findById(1L)).thenReturn(user);
        
        // Act & Assert
        mockMvc.perform(get("/api/v1/users/1")
                .accept(MediaType.APPLICATION_JSON))
            .andDo(print())  // print request/response
            .andExpect(status().isOk())
            .andExpect(content().contentType(MediaType.APPLICATION_JSON))
            .andExpect(jsonPath("$.id").value(1))
            .andExpect(jsonPath("$.username").value("john"))
            .andExpect(jsonPath("$.email").value("john@test.com"));
    }
    
    @Test
    void getUser_WhenNotExists_ShouldReturn404() throws Exception {
        when(userService.findById(999L))
            .thenThrow(new ResourceNotFoundException("User", "id", 999L));
        
        mockMvc.perform(get("/api/v1/users/999"))
            .andExpect(status().isNotFound());
    }
    
    @Test
    void createUser_WithValidData_ShouldReturn201() throws Exception {
        // Arrange
        CreateUserRequest request = new CreateUserRequest(
            "john", "john@test.com", "Password123!", "John", "Doe", null
        );
        
        UserResponse response = new UserResponse(
            1L, "john", "john@test.com", "John", "Doe", null, "USER", true, LocalDateTime.now()
        );
        
        when(userService.create(any())).thenReturn(response);
        
        // Act & Assert
        mockMvc.perform(post("/api/v1/users")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request)))
            .andExpect(status().isCreated())
            .andExpect(jsonPath("$.id").value(1))
            .andExpect(jsonPath("$.username").value("john"));
    }
    
    @Test
    void createUser_WithInvalidEmail_ShouldReturn400() throws Exception {
        CreateUserRequest request = new CreateUserRequest(
            "john", "not-an-email", "Password123!", null, null, null
        );
        
        mockMvc.perform(post("/api/v1/users")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request)))
            .andExpect(status().isBadRequest())
            .andExpect(jsonPath("$.fieldErrors.email").exists());
    }
    
    @Test
    void createUser_WithBlankUsername_ShouldReturn400() throws Exception {
        CreateUserRequest request = new CreateUserRequest(
            "", "john@test.com", "Password123!", null, null, null
        );
        
        mockMvc.perform(post("/api/v1/users")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request)))
            .andExpect(status().isBadRequest())
            .andExpect(jsonPath("$.fieldErrors.username").isNotEmpty());
    }
}
```

---

## ขั้นตอนที่ 209: @DataJpaTest (Repository Layer Testing)

```java
// @DataJpaTest - ทดสอบ Repository/JPA layer
// Auto-configure H2, Spring Data, Hibernate, Transactions
@DataJpaTest
class UserRepositoryTest {
    
    @Autowired
    private UserRepository userRepository;
    
    @Autowired
    private TestEntityManager entityManager;
    
    private User testUser;
    
    @BeforeEach
    void setUp() {
        testUser = User.builder()
            .username("john")
            .email("john@test.com")
            .password("hashed")
            .role(UserRole.USER)
            .build();
    }
    
    @Test
    void findByEmail_WhenExists_ShouldReturnUser() {
        // Arrange
        entityManager.persistAndFlush(testUser);
        
        // Act
        Optional<User> found = userRepository.findByEmail("john@test.com");
        
        // Assert
        assertThat(found).isPresent();
        assertThat(found.get().getUsername()).isEqualTo("john");
    }
    
    @Test
    void findByEmail_WhenNotExists_ShouldReturnEmpty() {
        Optional<User> found = userRepository.findByEmail("notexist@test.com");
        assertThat(found).isEmpty();
    }
    
    @Test
    void existsByEmail_WhenExists_ShouldReturnTrue() {
        entityManager.persistAndFlush(testUser);
        
        boolean exists = userRepository.existsByEmail("john@test.com");
        
        assertThat(exists).isTrue();
    }
    
    @Test
    void save_ShouldPersistUser() {
        User saved = userRepository.save(testUser);
        
        assertThat(saved.getId()).isNotNull();
        assertThat(saved.getCreatedAt()).isNotNull();
    }
    
    @Test
    void findAll_ShouldReturnAllUsers() {
        entityManager.persist(testUser);
        entityManager.persist(User.builder()
            .username("jane")
            .email("jane@test.com")
            .password("hashed")
            .build());
        entityManager.flush();
        
        List<User> users = userRepository.findAll();
        
        assertThat(users).hasSize(2);
    }
    
    @Test
    void findByRole_ShouldReturnMatchingUsers() {
        entityManager.persist(testUser);  // USER role
        entityManager.persist(User.builder()
            .username("admin")
            .email("admin@test.com")
            .password("hashed")
            .role(UserRole.ADMIN)
            .build());
        entityManager.flush();
        
        List<User> users = userRepository.findByRole(UserRole.USER);
        
        assertThat(users).hasSize(1);
        assertThat(users.get(0).getUsername()).isEqualTo("john");
    }
    
    // Test @Query method
    @Test
    void findActiveUserByEmail_ShouldReturnActiveUsers() {
        User activeUser = entityManager.persist(testUser);  // active = true by default
        User inactiveUser = entityManager.persist(User.builder()
            .username("inactive")
            .email("inactive@test.com")
            .password("hashed")
            .active(false)
            .build());
        entityManager.flush();
        
        Optional<User> found = userRepository.findActiveUserByEmail("john@test.com");
        assertThat(found).isPresent();
        
        Optional<User> notFound = userRepository.findActiveUserByEmail("inactive@test.com");
        assertThat(notFound).isEmpty();
    }
}
```

---

## ขั้นตอนที่ 210: TestContainers (Real Database Testing)

```java
// ทดสอบกับ PostgreSQL จริง ไม่ใช่ H2!
@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)  // ไม่ใช้ H2
@Testcontainers
class UserRepositoryIntegrationTest {
    
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine")
        .withDatabaseName("testdb")
        .withUsername("test")
        .withPassword("test");
    
    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }
    
    @Autowired
    private UserRepository userRepository;
    
    @Test
    void crud_WithRealDatabase() {
        User user = User.builder()
            .username("john")
            .email("john@test.com")
            .password("hashed")
            .build();
        
        User saved = userRepository.save(user);
        assertThat(saved.getId()).isNotNull();
        
        Optional<User> found = userRepository.findByEmail("john@test.com");
        assertThat(found).isPresent();
        
        userRepository.delete(saved);
        assertThat(userRepository.findById(saved.getId())).isEmpty();
    }
}
```

---

## ขั้นตอนที่ 211: Testing Security

```java
// เพิ่ม dependency:
// spring-security-test

@WebMvcTest(UserController.class)
class UserControllerSecurityTest {
    
    @Autowired
    private MockMvc mockMvc;
    
    @MockBean
    private UserService userService;
    
    @MockBean
    private JwtService jwtService;
    
    @Test
    void getUser_WithoutAuth_ShouldReturn401() throws Exception {
        mockMvc.perform(get("/api/v1/users/1"))
            .andExpect(status().isUnauthorized());
    }
    
    @Test
    @WithMockUser(username = "john", roles = {"USER"})  // Mock authenticated user
    void getUser_WithAuth_ShouldReturn200() throws Exception {
        when(userService.findById(1L)).thenReturn(mockUserResponse());
        
        mockMvc.perform(get("/api/v1/users/1"))
            .andExpect(status().isOk());
    }
    
    @Test
    @WithMockUser(username = "user", roles = {"USER"})
    void deleteUser_AsUser_ShouldReturn403() throws Exception {
        mockMvc.perform(delete("/api/v1/users/1"))
            .andExpect(status().isForbidden());
    }
    
    @Test
    @WithMockUser(username = "admin", roles = {"ADMIN"})
    void deleteUser_AsAdmin_ShouldReturn200() throws Exception {
        doNothing().when(userService).delete(1L);
        
        mockMvc.perform(delete("/api/v1/users/1"))
            .andExpect(status().isOk());
    }
    
    // Test JWT
    @Test
    void getUser_WithValidJwt_ShouldReturn200() throws Exception {
        String token = "valid.jwt.token";
        when(jwtService.validateToken(token)).thenReturn(true);
        when(jwtService.extractUsername(token)).thenReturn("john");
        when(userService.findById(1L)).thenReturn(mockUserResponse());
        
        mockMvc.perform(get("/api/v1/users/1")
                .header("Authorization", "Bearer " + token))
            .andExpect(status().isOk());
    }
}
```

---

## ขั้นตอนที่ 212: Testing Async Methods

```java
@SpringBootTest
class AsyncServiceTest {
    
    @Autowired
    private AsyncEmailService emailService;
    
    @Test
    void sendEmailAsync_ShouldComplete() throws Exception {
        CompletableFuture<Boolean> result = emailService.sendAsync("john@test.com", "Hello");
        
        // รอให้ async task เสร็จ (timeout 5 วิ)
        Boolean sent = result.get(5, TimeUnit.SECONDS);
        
        assertThat(sent).isTrue();
    }
    
    // ใช้ Awaitility สำหรับ async testing
    @Test
    void processInBackground_ShouldComplete() {
        emailService.processInBackground("test@test.com");
        
        await()
            .atMost(5, TimeUnit.SECONDS)
            .untilAsserted(() -> {
                verify(emailRepository, times(1)).save(any());
            });
    }
}
```

---

## ขั้นตอนที่ 213: Test Data Builders

```java
// Builder pattern สำหรับ test data
public class UserTestDataBuilder {
    
    private Long id = 1L;
    private String username = "testuser";
    private String email = "test@test.com";
    private String password = "$2a$12$hashedPassword";
    private String firstName = "Test";
    private String lastName = "User";
    private UserRole role = UserRole.USER;
    private boolean active = true;
    
    public static UserTestDataBuilder aUser() {
        return new UserTestDataBuilder();
    }
    
    public UserTestDataBuilder withId(Long id) {
        this.id = id;
        return this;
    }
    
    public UserTestDataBuilder withUsername(String username) {
        this.username = username;
        return this;
    }
    
    public UserTestDataBuilder withEmail(String email) {
        this.email = email;
        return this;
    }
    
    public UserTestDataBuilder asAdmin() {
        this.role = UserRole.ADMIN;
        return this;
    }
    
    public UserTestDataBuilder inactive() {
        this.active = false;
        return this;
    }
    
    public User build() {
        return User.builder()
            .id(id)
            .username(username)
            .email(email)
            .password(password)
            .firstName(firstName)
            .lastName(lastName)
            .role(role)
            .active(active)
            .build();
    }
}

// ใช้งาน
@Test
void test() {
    User admin = aUser().withId(1L).withEmail("admin@test.com").asAdmin().build();
    User inactive = aUser().withId(2L).inactive().build();
}
```

---

## ขั้นตอนที่ 214: Test Configuration

```java
// Test-specific configuration
@TestConfiguration
public class TestConfig {
    
    @Bean
    @Primary  // Override main bean
    public EmailService fakeEmailService() {
        return (email) -> {
            System.out.println("FAKE EMAIL: " + email);
        };
    }
    
    @Bean
    public Clock fixedClock() {
        return Clock.fixed(
            Instant.parse("2026-01-01T00:00:00Z"),
            ZoneId.of("UTC")
        );
    }
}

// ใช้ใน test
@SpringBootTest
@Import(TestConfig.class)
class MyTest {
    
    @Autowired
    private EmailService emailService;  // จะได้ fakeEmailService
}
```

---

## ขั้นตอนที่ 215: Code Coverage

```xml
<!-- pom.xml - JaCoCo สำหรับ code coverage -->
<plugin>
    <groupId>org.jacoco</groupId>
    <artifactId>jacoco-maven-plugin</artifactId>
    <version>0.8.11</version>
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
        <!-- Fail ถ้า coverage ต่ำกว่า threshold -->
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
                                <minimum>0.80</minimum>  <!-- 80% minimum -->
                            </limit>
                        </limits>
                    </rule>
                </rules>
            </configuration>
        </execution>
    </executions>
</plugin>
```

```bash
# Generate coverage report
mvn test jacoco:report

# ดู report ที่:
# target/site/jacoco/index.html
```

---

## ขั้นตอนที่ 216-230: Complete Test Example

```java
// Complete test ที่ cover Controller, Service, Repository
// สำหรับ User management feature

// ===== Unit Test: Service =====
@ExtendWith(MockitoExtension.class)
class UserServiceTest {
    
    @Mock UserRepository userRepository;
    @Mock EmailService emailService;
    @Mock PasswordEncoder passwordEncoder;
    @Mock UserMapper userMapper;
    
    @InjectMocks
    UserService userService;
    
    @Test
    @DisplayName("Create: Success case")
    void create_Success() {
        // Arrange
        var request = new CreateUserRequest("john", "john@test.com", "Pass123!", null, null, null);
        var entity = User.builder().id(1L).username("john").email("john@test.com").build();
        var response = new UserResponse(1L, "john", "john@test.com", null, null, null, "USER", true, LocalDateTime.now());
        
        given(userRepository.existsByEmail(any())).willReturn(false);
        given(userRepository.existsByUsername(any())).willReturn(false);
        given(userMapper.toEntity(any())).willReturn(entity);
        given(passwordEncoder.encode(any())).willReturn("hashed");
        given(userRepository.save(any())).willReturn(entity);
        given(userMapper.toResponse(any())).willReturn(response);
        
        // Act
        var result = userService.create(request);
        
        // Assert
        assertThat(result.id()).isEqualTo(1L);
        then(userRepository).should().save(any());
    }
    
    @Test
    @DisplayName("Create: Duplicate email throws exception")
    void create_DuplicateEmail_ThrowsException() {
        var request = new CreateUserRequest("john", "dup@test.com", "Pass123!", null, null, null);
        given(userRepository.existsByEmail("dup@test.com")).willReturn(true);
        
        assertThatThrownBy(() -> userService.create(request))
            .isInstanceOf(DuplicateResourceException.class);
        
        then(userRepository).should(never()).save(any());
    }
    
    @Test
    @DisplayName("FindById: Not found throws exception")
    void findById_NotFound_ThrowsException() {
        given(userRepository.findById(999L)).willReturn(Optional.empty());
        
        assertThatThrownBy(() -> userService.findById(999L))
            .isInstanceOf(ResourceNotFoundException.class)
            .hasMessageContaining("999");
    }
}

// ===== Slice Test: Controller =====
@WebMvcTest(UserController.class)
@WithMockUser  // Default mock user
class UserControllerTest {
    
    @Autowired MockMvc mockMvc;
    @Autowired ObjectMapper objectMapper;
    @MockBean UserService userService;
    
    @Test
    void getAll_ShouldReturn200() throws Exception {
        var users = Page.empty();
        given(userService.findAll(any())).willReturn(users);
        
        mockMvc.perform(get("/api/v1/users"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.success").value(true));
    }
    
    @Test
    void create_ValidRequest_ShouldReturn201() throws Exception {
        var request = new CreateUserRequest("john", "john@test.com", "Pass123!", "John", "Doe", null);
        var response = new UserResponse(1L, "john", "john@test.com", "John", "Doe", null, "USER", true, LocalDateTime.now());
        
        given(userService.create(any())).willReturn(response);
        
        mockMvc.perform(post("/api/v1/users")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request)))
            .andExpect(status().isCreated())
            .andExpect(jsonPath("$.data.id").value(1));
    }
}

// ===== Integration Test: Repository =====
@DataJpaTest
class UserRepositoryTest {
    
    @Autowired UserRepository repo;
    @Autowired TestEntityManager em;
    
    @Test
    void findByEmail_Success() {
        em.persistAndFlush(User.builder()
            .username("john")
            .email("john@test.com")
            .password("hashed")
            .build());
        
        assertThat(repo.findByEmail("john@test.com")).isPresent();
        assertThat(repo.findByEmail("other@test.com")).isEmpty();
    }
}
```

### Testing Checklist

```
Unit Tests:
  ✅ ทดสอบ happy path
  ✅ ทดสอบ error cases
  ✅ ทดสอบ edge cases (null, empty, max)
  ✅ Verify interactions กับ mocks
  ✅ Test naming: methodName_Scenario_ExpectedResult

Integration Tests:
  ✅ @DataJpaTest สำหรับ repository
  ✅ @WebMvcTest สำหรับ controller
  ✅ @SpringBootTest สำหรับ full integration
  ✅ ใช้ TestContainers สำหรับ real DB

General:
  ✅ 80%+ code coverage
  ✅ Tests รันเร็ว (< 1 นาที ต่อ unit test suite)
  ✅ Tests เป็น independent (ไม่พึ่งกัน)
  ✅ ใช้ @Transactional ใน integration tests
  ✅ Clean up test data
```

---

*[← Part 09: Logging](./part-09-logging.md) | [Part 11: Spring Data JPA →](./part-11-spring-data-jpa.md)*

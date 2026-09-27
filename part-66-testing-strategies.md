# Part 66: Testing Strategies
## ขั้นตอนที่ 2241-2280

**ระดับ:** ระดับโลก (World-Class)
**เวลาเรียน:** 7-8 ชั่วโมง
**เป้าหมาย:** เชี่ยวชาญกลยุทธ์การทดสอบแบบครบวงจร ตั้งแต่ Unit Test ไปจนถึง Contract Testing และ Mutation Testing สำหรับ Spring Boot Applications ระดับ Production

---

## ขั้นตอนที่ 2241-2245: Test Pyramid และแนวคิดพื้นฐาน

### ทำความเข้าใจ Test Pyramid

Test Pyramid คือแนวคิดหลักในการวางกลยุทธ์การทดสอบ โดยแบ่งออกเป็น 3 ชั้น:

```
           /\
          /  \
         / E2E\          ← น้อยที่สุด แต่ครอบคลุม Business Flow
        /------\
       / Integr.\        ← ปานกลาง ทดสอบ Component ร่วมกัน
      /----------\
     /  Unit Tests \     ← มากที่สุด เร็วที่สุด ราคาถูกที่สุด
    /--------------\
```

| ชั้น | จำนวน | ความเร็ว | ความครอบคลุม | ราคา |
|------|--------|----------|-------------|------|
| Unit | 70% | < 1ms | เฉพาะ Logic | ต่ำ |
| Integration | 20% | 100ms-1s | หลาย Component | กลาง |
| E2E | 10% | 1-30s | ทั้ง System | สูง |

### Maven Dependencies สำหรับ Testing

```xml
<!-- pom.xml -->
<dependencies>
    <!-- Spring Boot Test Starter (รวม JUnit 5, Mockito, AssertJ) -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>

    <!-- TestContainers - ทดสอบกับ Real Infrastructure -->
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
    <dependency>
        <groupId>org.testcontainers</groupId>
        <artifactId>kafka</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.testcontainers</groupId>
        <artifactId>redis</artifactId>
        <scope>test</scope>
    </dependency>

    <!-- Spring Cloud Contract -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-contract-verifier</artifactId>
        <scope>test</scope>
    </dependency>

    <!-- Awaitility - สำหรับ Async Testing -->
    <dependency>
        <groupId>org.awaitility</groupId>
        <artifactId>awaitility</artifactId>
        <scope>test</scope>
    </dependency>

    <!-- WireMock - Mock HTTP Services -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-contract-wiremock</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>

<!-- PIT Mutation Testing Plugin -->
<plugin>
    <groupId>org.pitest</groupId>
    <artifactId>pitest-maven</artifactId>
    <version>1.15.0</version>
    <dependencies>
        <dependency>
            <groupId>org.pitest</groupId>
            <artifactId>pitest-junit5-plugin</artifactId>
            <version>1.2.1</version>
        </dependency>
    </dependencies>
    <configuration>
        <targetClasses>
            <param>com.example.*</param>
        </targetClasses>
        <targetTests>
            <param>com.example.*</param>
        </targetTests>
        <mutationThreshold>80</mutationThreshold>
        <coverageThreshold>85</coverageThreshold>
    </configuration>
</plugin>
```

---

## ขั้นตอนที่ 2246-2252: Unit Testing ขั้นสูง

### Domain Model สำหรับตัวอย่าง

```java
// Order.java - Domain Entity
@Entity
@Table(name = "orders")
public class Order {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String customerId;

    @OneToMany(cascade = CascadeType.ALL, orphanRemoval = true)
    private List<OrderItem> items = new ArrayList<>();

    @Enumerated(EnumType.STRING)
    private OrderStatus status = OrderStatus.PENDING;

    private BigDecimal totalAmount;
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;

    public void addItem(OrderItem item) {
        items.add(item);
        recalculateTotal();
    }

    public void removeItem(Long itemId) {
        items.removeIf(item -> item.getId().equals(itemId));
        recalculateTotal();
    }

    public void confirm() {
        if (status != OrderStatus.PENDING) {
            throw new IllegalStateException(
                "Cannot confirm order in status: " + status
            );
        }
        this.status = OrderStatus.CONFIRMED;
        this.updatedAt = LocalDateTime.now();
    }

    public void cancel(String reason) {
        if (status == OrderStatus.DELIVERED) {
            throw new IllegalStateException("Cannot cancel delivered order");
        }
        this.status = OrderStatus.CANCELLED;
        this.updatedAt = LocalDateTime.now();
    }

    private void recalculateTotal() {
        this.totalAmount = items.stream()
            .map(item -> item.getPrice().multiply(BigDecimal.valueOf(item.getQuantity())))
            .reduce(BigDecimal.ZERO, BigDecimal::add);
    }

    // Getters and Setters...
}

// OrderService.java
@Service
@Transactional
public class OrderService {

    private final OrderRepository orderRepository;
    private final ProductService productService;
    private final NotificationService notificationService;
    private final OrderEventPublisher eventPublisher;

    public OrderService(
        OrderRepository orderRepository,
        ProductService productService,
        NotificationService notificationService,
        OrderEventPublisher eventPublisher
    ) {
        this.orderRepository = orderRepository;
        this.productService = productService;
        this.notificationService = notificationService;
        this.eventPublisher = eventPublisher;
    }

    public Order createOrder(CreateOrderRequest request) {
        // ตรวจสอบ stock
        request.getItems().forEach(item -> {
            Product product = productService.findById(item.getProductId())
                .orElseThrow(() -> new ProductNotFoundException(item.getProductId()));

            if (product.getStock() < item.getQuantity()) {
                throw new InsufficientStockException(item.getProductId(), item.getQuantity());
            }
        });

        Order order = new Order();
        order.setCustomerId(request.getCustomerId());

        request.getItems().forEach(item -> {
            Product product = productService.findById(item.getProductId()).get();
            OrderItem orderItem = new OrderItem(
                product.getId(),
                product.getName(),
                product.getPrice(),
                item.getQuantity()
            );
            order.addItem(orderItem);
            productService.decreaseStock(product.getId(), item.getQuantity());
        });

        Order savedOrder = orderRepository.save(order);
        eventPublisher.publishOrderCreated(savedOrder);
        notificationService.notifyOrderCreated(savedOrder);

        return savedOrder;
    }

    public Order confirmOrder(Long orderId) {
        Order order = orderRepository.findById(orderId)
            .orElseThrow(() -> new OrderNotFoundException(orderId));
        order.confirm();
        Order savedOrder = orderRepository.save(order);
        eventPublisher.publishOrderConfirmed(savedOrder);
        return savedOrder;
    }
}
```

### Unit Test ครอบคลุม Business Logic

```java
// OrderServiceTest.java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {

    @Mock
    private OrderRepository orderRepository;

    @Mock
    private ProductService productService;

    @Mock
    private NotificationService notificationService;

    @Mock
    private OrderEventPublisher eventPublisher;

    @InjectMocks
    private OrderService orderService;

    // Test Data Builders
    private Product createProduct(Long id, int stock, BigDecimal price) {
        Product product = new Product();
        product.setId(id);
        product.setName("Product " + id);
        product.setStock(stock);
        product.setPrice(price);
        return product;
    }

    private CreateOrderRequest createOrderRequest(String customerId, Long productId, int quantity) {
        OrderItemRequest itemRequest = new OrderItemRequest(productId, quantity);
        return new CreateOrderRequest(customerId, List.of(itemRequest));
    }

    @Test
    @DisplayName("createOrder - ควรสร้าง Order สำเร็จเมื่อ stock เพียงพอ")
    void createOrder_WhenSufficientStock_ShouldCreateSuccessfully() {
        // Arrange
        Product product = createProduct(1L, 10, new BigDecimal("100.00"));
        CreateOrderRequest request = createOrderRequest("customer-1", 1L, 5);

        when(productService.findById(1L)).thenReturn(Optional.of(product));
        when(orderRepository.save(any(Order.class))).thenAnswer(inv -> {
            Order order = inv.getArgument(0);
            order.setId(1L);
            return order;
        });

        // Act
        Order result = orderService.createOrder(request);

        // Assert
        assertThat(result).isNotNull();
        assertThat(result.getCustomerId()).isEqualTo("customer-1");
        assertThat(result.getItems()).hasSize(1);
        assertThat(result.getTotalAmount()).isEqualByComparingTo("500.00");
        assertThat(result.getStatus()).isEqualTo(OrderStatus.PENDING);

        // ตรวจสอบว่า method ที่สำคัญถูกเรียก
        verify(productService).decreaseStock(1L, 5);
        verify(orderRepository).save(any(Order.class));
        verify(eventPublisher).publishOrderCreated(any(Order.class));
        verify(notificationService).notifyOrderCreated(any(Order.class));
    }

    @Test
    @DisplayName("createOrder - ควร throw exception เมื่อ stock ไม่พอ")
    void createOrder_WhenInsufficientStock_ShouldThrowException() {
        // Arrange
        Product product = createProduct(1L, 3, new BigDecimal("100.00"));
        CreateOrderRequest request = createOrderRequest("customer-1", 1L, 5);

        when(productService.findById(1L)).thenReturn(Optional.of(product));

        // Act & Assert
        assertThatThrownBy(() -> orderService.createOrder(request))
            .isInstanceOf(InsufficientStockException.class)
            .hasMessageContaining("1");  // product id

        // ตรวจสอบว่าไม่มีการ save order
        verify(orderRepository, never()).save(any());
        verify(eventPublisher, never()).publishOrderCreated(any());
    }

    @Test
    @DisplayName("createOrder - ควร throw exception เมื่อไม่พบ product")
    void createOrder_WhenProductNotFound_ShouldThrowException() {
        // Arrange
        CreateOrderRequest request = createOrderRequest("customer-1", 999L, 1);
        when(productService.findById(999L)).thenReturn(Optional.empty());

        // Act & Assert
        assertThatThrownBy(() -> orderService.createOrder(request))
            .isInstanceOf(ProductNotFoundException.class);
    }

    @Test
    @DisplayName("confirmOrder - ควร confirm Order ที่มีสถานะ PENDING")
    void confirmOrder_WhenPending_ShouldConfirmSuccessfully() {
        // Arrange
        Order order = new Order();
        order.setId(1L);
        order.setStatus(OrderStatus.PENDING);

        when(orderRepository.findById(1L)).thenReturn(Optional.of(order));
        when(orderRepository.save(any(Order.class))).thenReturn(order);

        // Act
        Order result = orderService.confirmOrder(1L);

        // Assert
        assertThat(result.getStatus()).isEqualTo(OrderStatus.CONFIRMED);
        verify(eventPublisher).publishOrderConfirmed(order);
    }

    @Test
    @DisplayName("confirmOrder - ควร throw exception เมื่อ Order ไม่ใช่ PENDING")
    void confirmOrder_WhenNotPending_ShouldThrowException() {
        // Arrange
        Order order = new Order();
        order.setId(1L);
        order.setStatus(OrderStatus.CANCELLED);

        when(orderRepository.findById(1L)).thenReturn(Optional.of(order));

        // Act & Assert
        assertThatThrownBy(() -> orderService.confirmOrder(1L))
            .isInstanceOf(IllegalStateException.class)
            .hasMessageContaining("CANCELLED");

        verify(eventPublisher, never()).publishOrderConfirmed(any());
    }
}
```

---

## ขั้นตอนที่ 2253-2258: Mockito Advanced - Spies, Argument Captors

### การใช้ Spy และ ArgumentCaptor

```java
// OrderServiceAdvancedTest.java
@ExtendWith(MockitoExtension.class)
class OrderServiceAdvancedTest {

    @Mock
    private OrderRepository orderRepository;

    @Mock
    private ProductService productService;

    @Spy
    private NotificationService notificationService = new EmailNotificationService();

    @Mock
    private OrderEventPublisher eventPublisher;

    @InjectMocks
    private OrderService orderService;

    @Captor
    private ArgumentCaptor<Order> orderCaptor;

    @Captor
    private ArgumentCaptor<OrderCreatedEvent> eventCaptor;

    @Test
    @DisplayName("ใช้ ArgumentCaptor เพื่อตรวจสอบ Object ที่ส่งไปยัง Repository")
    void createOrder_ShouldSaveOrderWithCorrectData() {
        // Arrange
        Product product = createProduct(1L, 10, new BigDecimal("250.00"));
        CreateOrderRequest request = createOrderRequest("cust-123", 1L, 3);

        when(productService.findById(1L)).thenReturn(Optional.of(product));
        when(orderRepository.save(any())).thenAnswer(i -> i.getArgument(0));

        // Act
        orderService.createOrder(request);

        // Assert - ใช้ ArgumentCaptor ตรวจสอบ Order ที่ถูก save
        verify(orderRepository).save(orderCaptor.capture());
        Order capturedOrder = orderCaptor.getValue();

        assertThat(capturedOrder.getCustomerId()).isEqualTo("cust-123");
        assertThat(capturedOrder.getTotalAmount()).isEqualByComparingTo("750.00");
        assertThat(capturedOrder.getItems()).hasSize(1);
        assertThat(capturedOrder.getItems().get(0).getQuantity()).isEqualTo(3);
    }

    @Test
    @DisplayName("ใช้ ArgumentCaptor กับ Event Publisher")
    void createOrder_ShouldPublishEventWithCorrectData() {
        // Arrange
        Product product = createProduct(1L, 10, new BigDecimal("100.00"));
        CreateOrderRequest request = createOrderRequest("cust-456", 1L, 2);

        when(productService.findById(1L)).thenReturn(Optional.of(product));
        when(orderRepository.save(any())).thenAnswer(i -> {
            Order o = i.getArgument(0);
            o.setId(99L);
            return o;
        });

        // Act
        orderService.createOrder(request);

        // Assert - ตรวจสอบ Event ที่ถูก publish
        verify(eventPublisher).publishOrderCreated(orderCaptor.capture());
        Order publishedOrder = orderCaptor.getValue();

        assertThat(publishedOrder.getId()).isEqualTo(99L);
        assertThat(publishedOrder.getCustomerId()).isEqualTo("cust-456");
    }

    @Test
    @DisplayName("ใช้ Spy เพื่อ override บาง method แต่คง behavior อื่น")
    void createOrder_WithSpy_ShouldCallRealNotification() {
        // Arrange
        Product product = createProduct(1L, 10, new BigDecimal("100.00"));
        CreateOrderRequest request = createOrderRequest("cust-789", 1L, 1);

        when(productService.findById(1L)).thenReturn(Optional.of(product));
        when(orderRepository.save(any())).thenAnswer(i -> i.getArgument(0));

        // Spy: override เฉพาะ method ที่ต้องการ, method อื่นใช้ real implementation
        doNothing().when(notificationService).sendEmail(anyString(), anyString());

        // Act
        orderService.createOrder(request);

        // Assert
        verify(notificationService).notifyOrderCreated(any(Order.class));
        verify(notificationService).sendEmail(eq("cust-789@example.com"), anyString());
    }

    @Test
    @DisplayName("Stubbing Chain - mock method ที่ถูกเรียกหลายครั้ง")
    void createOrder_WithMultipleProducts_ShouldCallProductServiceMultipleTimes() {
        // Arrange - ใช้ thenReturn chain
        Product product1 = createProduct(1L, 10, new BigDecimal("100.00"));
        Product product2 = createProduct(2L, 5, new BigDecimal("200.00"));

        // สร้าง request ที่มี 2 items
        List<OrderItemRequest> items = List.of(
            new OrderItemRequest(1L, 2),
            new OrderItemRequest(2L, 1)
        );
        CreateOrderRequest request = new CreateOrderRequest("cust-001", items);

        // Stubbing Chain: เรียกครั้งแรกได้ product1, ครั้งที่สองได้ product2
        when(productService.findById(1L)).thenReturn(Optional.of(product1));
        when(productService.findById(2L)).thenReturn(Optional.of(product2));
        when(orderRepository.save(any())).thenAnswer(i -> i.getArgument(0));

        // Act
        Order result = orderService.createOrder(request);

        // Assert
        assertThat(result.getItems()).hasSize(2);
        assertThat(result.getTotalAmount()).isEqualByComparingTo("400.00"); // 100*2 + 200*1
        verify(productService, times(2)).findById(anyLong());
        verify(productService).decreaseStock(1L, 2);
        verify(productService).decreaseStock(2L, 1);
    }
}
```

---

## ขั้นตอนที่ 2259-2265: Spring Boot Test Slices

### @WebMvcTest - ทดสอบ Web Layer เท่านั้น

```java
// OrderControllerTest.java
@WebMvcTest(OrderController.class)
class OrderControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private OrderService orderService;

    @Autowired
    private ObjectMapper objectMapper;

    @Test
    @DisplayName("POST /api/orders - ควรสร้าง Order และ return 201")
    void createOrder_ShouldReturn201WithOrderData() throws Exception {
        // Arrange
        CreateOrderRequest request = new CreateOrderRequest(
            "customer-1",
            List.of(new OrderItemRequest(1L, 2))
        );

        Order createdOrder = new Order();
        createdOrder.setId(1L);
        createdOrder.setCustomerId("customer-1");
        createdOrder.setStatus(OrderStatus.PENDING);
        createdOrder.setTotalAmount(new BigDecimal("200.00"));

        when(orderService.createOrder(any(CreateOrderRequest.class)))
            .thenReturn(createdOrder);

        // Act & Assert
        mockMvc.perform(post("/api/orders")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request)))
            .andExpect(status().isCreated())
            .andExpect(jsonPath("$.id").value(1))
            .andExpect(jsonPath("$.customerId").value("customer-1"))
            .andExpect(jsonPath("$.status").value("PENDING"))
            .andExpect(jsonPath("$.totalAmount").value(200.00))
            .andDo(print());
    }

    @Test
    @DisplayName("POST /api/orders - ควร return 400 เมื่อ request ไม่ valid")
    void createOrder_WhenInvalidRequest_ShouldReturn400() throws Exception {
        // Request ที่ไม่มี customerId
        String invalidRequest = """
            {
                "items": []
            }
            """;

        mockMvc.perform(post("/api/orders")
                .contentType(MediaType.APPLICATION_JSON)
                .content(invalidRequest))
            .andExpect(status().isBadRequest())
            .andExpect(jsonPath("$.errors").exists());
    }

    @Test
    @DisplayName("GET /api/orders/{id} - ควร return 404 เมื่อไม่พบ Order")
    void getOrder_WhenNotFound_ShouldReturn404() throws Exception {
        when(orderService.findById(999L))
            .thenThrow(new OrderNotFoundException(999L));

        mockMvc.perform(get("/api/orders/999"))
            .andExpect(status().isNotFound())
            .andExpect(jsonPath("$.message").value(containsString("999")));
    }

    @Test
    @DisplayName("GET /api/orders - ควร return paginated list")
    void getOrders_ShouldReturnPaginatedList() throws Exception {
        List<Order> orders = List.of(
            createOrder(1L, "cust-1", OrderStatus.PENDING),
            createOrder(2L, "cust-2", OrderStatus.CONFIRMED)
        );

        when(orderService.findAll(any(Pageable.class)))
            .thenReturn(new PageImpl<>(orders));

        mockMvc.perform(get("/api/orders")
                .param("page", "0")
                .param("size", "10"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.content").isArray())
            .andExpect(jsonPath("$.content.length()").value(2))
            .andExpect(jsonPath("$.totalElements").value(2));
    }

    @Test
    @WithMockUser(roles = "ADMIN")
    @DisplayName("DELETE /api/orders/{id} - Admin ควรสามารถลบ Order ได้")
    void deleteOrder_AsAdmin_ShouldReturn204() throws Exception {
        doNothing().when(orderService).deleteOrder(1L);

        mockMvc.perform(delete("/api/orders/1"))
            .andExpect(status().isNoContent());
    }

    @Test
    @WithMockUser(roles = "USER")
    @DisplayName("DELETE /api/orders/{id} - User ทั่วไปไม่ควรลบ Order ได้")
    void deleteOrder_AsUser_ShouldReturn403() throws Exception {
        mockMvc.perform(delete("/api/orders/1"))
            .andExpect(status().isForbidden());
    }

    private Order createOrder(Long id, String customerId, OrderStatus status) {
        Order order = new Order();
        order.setId(id);
        order.setCustomerId(customerId);
        order.setStatus(status);
        return order;
    }
}
```

### @DataJpaTest - ทดสอบ Repository Layer

```java
// OrderRepositoryTest.java
@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
@Testcontainers
class OrderRepositoryTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15")
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
    private OrderRepository orderRepository;

    @Autowired
    private TestEntityManager entityManager;

    @BeforeEach
    void setUp() {
        orderRepository.deleteAll();
    }

    @Test
    @DisplayName("ควรหา Order ตาม customerId ได้")
    void findByCustomerId_ShouldReturnOrders() {
        // Arrange - สร้าง test data
        Order order1 = createAndPersistOrder("cust-001", OrderStatus.PENDING);
        Order order2 = createAndPersistOrder("cust-001", OrderStatus.CONFIRMED);
        createAndPersistOrder("cust-002", OrderStatus.PENDING); // order ของ customer อื่น

        entityManager.flush();
        entityManager.clear();

        // Act
        List<Order> result = orderRepository.findByCustomerId("cust-001");

        // Assert
        assertThat(result).hasSize(2);
        assertThat(result).allMatch(o -> o.getCustomerId().equals("cust-001"));
    }

    @Test
    @DisplayName("ควรหา Order ตาม status ได้")
    void findByStatus_ShouldReturnFilteredOrders() {
        // Arrange
        createAndPersistOrder("cust-001", OrderStatus.PENDING);
        createAndPersistOrder("cust-002", OrderStatus.PENDING);
        createAndPersistOrder("cust-003", OrderStatus.CONFIRMED);

        entityManager.flush();

        // Act
        List<Order> pendingOrders = orderRepository.findByStatus(OrderStatus.PENDING);

        // Assert
        assertThat(pendingOrders).hasSize(2);
        assertThat(pendingOrders).allMatch(o -> o.getStatus() == OrderStatus.PENDING);
    }

    @Test
    @DisplayName("Custom query - หา Order ที่มียอดรวมเกิน threshold")
    void findByTotalAmountGreaterThan_ShouldWork() {
        // Arrange
        Order smallOrder = createOrderWithAmount("cust-001", new BigDecimal("100.00"));
        Order largeOrder = createOrderWithAmount("cust-002", new BigDecimal("1000.00"));
        entityManager.persist(smallOrder);
        entityManager.persist(largeOrder);
        entityManager.flush();

        // Act
        List<Order> result = orderRepository.findByTotalAmountGreaterThan(new BigDecimal("500.00"));

        // Assert
        assertThat(result).hasSize(1);
        assertThat(result.get(0).getTotalAmount()).isEqualByComparingTo("1000.00");
    }

    private Order createAndPersistOrder(String customerId, OrderStatus status) {
        Order order = new Order();
        order.setCustomerId(customerId);
        order.setStatus(status);
        order.setTotalAmount(BigDecimal.ZERO);
        order.setCreatedAt(LocalDateTime.now());
        return entityManager.persist(order);
    }

    private Order createOrderWithAmount(String customerId, BigDecimal amount) {
        Order order = new Order();
        order.setCustomerId(customerId);
        order.setStatus(OrderStatus.PENDING);
        order.setTotalAmount(amount);
        order.setCreatedAt(LocalDateTime.now());
        return order;
    }
}
```

### @JsonTest - ทดสอบ JSON Serialization/Deserialization

```java
// OrderJsonTest.java
@JsonTest
class OrderJsonTest {

    @Autowired
    private JacksonTester<Order> orderTester;

    @Autowired
    private JacksonTester<CreateOrderRequest> requestTester;

    @Test
    @DisplayName("Order ควร serialize เป็น JSON ถูกต้อง")
    void serialize_Order_ShouldProduceCorrectJson() throws Exception {
        Order order = new Order();
        order.setId(1L);
        order.setCustomerId("cust-001");
        order.setStatus(OrderStatus.CONFIRMED);
        order.setTotalAmount(new BigDecimal("500.00"));
        order.setCreatedAt(LocalDateTime.of(2024, 1, 15, 10, 30, 0));

        JsonContent<Order> result = orderTester.write(order);

        assertThat(result).hasJsonPathNumberValue("$.id", 1);
        assertThat(result).hasJsonPathStringValue("$.customerId", "cust-001");
        assertThat(result).hasJsonPathStringValue("$.status", "CONFIRMED");
        assertThat(result).hasJsonPathNumberValue("$.totalAmount", 500.00);
        assertThat(result).doesNotHaveJsonPath("$.password"); // field ที่ไม่ควรเปิดเผย
    }

    @Test
    @DisplayName("JSON ควร deserialize เป็น CreateOrderRequest ถูกต้อง")
    void deserialize_JsonToCreateOrderRequest_ShouldWork() throws Exception {
        String json = """
            {
                "customerId": "cust-001",
                "items": [
                    {"productId": 1, "quantity": 3},
                    {"productId": 2, "quantity": 1}
                ]
            }
            """;

        CreateOrderRequest result = requestTester.parseObject(json);

        assertThat(result.getCustomerId()).isEqualTo("cust-001");
        assertThat(result.getItems()).hasSize(2);
        assertThat(result.getItems().get(0).getProductId()).isEqualTo(1L);
        assertThat(result.getItems().get(0).getQuantity()).isEqualTo(3);
    }

    @Test
    @DisplayName("Order ที่มี null fields ควรไม่แสดงใน JSON")
    void serialize_OrderWithNullFields_ShouldExcludeNulls() throws Exception {
        Order order = new Order();
        order.setId(1L);
        order.setCustomerId("cust-001");
        // items, totalAmount ไม่ได้ set

        JsonContent<Order> result = orderTester.write(order);

        assertThat(result).doesNotHaveJsonPath("$.updatedAt");
    }
}
```

---

## ขั้นตอนที่ 2266-2272: TestContainers - ทดสอบกับ Real Infrastructure

### Integration Test กับ PostgreSQL, Redis, Kafka

```java
// OrderIntegrationTest.java - ทดสอบ Full Stack
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Testcontainers
@ActiveProfiles("test")
class OrderIntegrationTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15-alpine")
        .withDatabaseName("orders_test")
        .withUsername("testuser")
        .withPassword("testpass")
        .withInitScript("sql/schema.sql");

    @Container
    static GenericContainer<?> redis = new GenericContainer<>("redis:7-alpine")
        .withExposedPorts(6379);

    @Container
    static KafkaContainer kafka = new KafkaContainer(
        DockerImageName.parse("confluentinc/cp-kafka:7.4.0")
    );

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        // Database
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);

        // Redis
        registry.add("spring.redis.host", redis::getHost);
        registry.add("spring.redis.port", () -> redis.getMappedPort(6379));

        // Kafka
        registry.add("spring.kafka.bootstrap-servers", kafka::getBootstrapServers);
    }

    @Autowired
    private TestRestTemplate restTemplate;

    @Autowired
    private OrderRepository orderRepository;

    @Autowired
    private ProductRepository productRepository;

    @BeforeEach
    void setUp() {
        orderRepository.deleteAll();
        productRepository.deleteAll();

        // เตรียม test data
        Product product = new Product("Test Product", new BigDecimal("100.00"), 50);
        productRepository.save(product);
    }

    @Test
    @DisplayName("Integration: สร้าง Order แบบ end-to-end")
    void createOrder_FullFlow_ShouldWork() {
        // Arrange
        Product product = productRepository.findAll().get(0);
        CreateOrderRequest request = new CreateOrderRequest(
            "integration-test-customer",
            List.of(new OrderItemRequest(product.getId(), 3))
        );

        // Act
        ResponseEntity<OrderResponse> response = restTemplate.postForEntity(
            "/api/orders",
            request,
            OrderResponse.class
        );

        // Assert HTTP Response
        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.CREATED);
        assertThat(response.getBody()).isNotNull();
        assertThat(response.getBody().getStatus()).isEqualTo("PENDING");

        // ตรวจสอบใน Database โดยตรง
        Long orderId = response.getBody().getId();
        Order savedOrder = orderRepository.findById(orderId).orElseThrow();
        assertThat(savedOrder.getTotalAmount()).isEqualByComparingTo("300.00");

        // ตรวจสอบว่า stock ลดลง
        Product updatedProduct = productRepository.findById(product.getId()).orElseThrow();
        assertThat(updatedProduct.getStock()).isEqualTo(47); // 50 - 3
    }

    @Test
    @DisplayName("Integration: ทดสอบ Cache กับ Redis")
    void getOrder_SecondCall_ShouldReturnCachedResult() {
        // Arrange - สร้าง Order ก่อน
        Order order = new Order();
        order.setCustomerId("cache-test-customer");
        order.setStatus(OrderStatus.CONFIRMED);
        order.setTotalAmount(new BigDecimal("500.00"));
        Order saved = orderRepository.save(order);

        // Act - เรียก 2 ครั้ง
        ResponseEntity<OrderResponse> firstResponse = restTemplate.getForEntity(
            "/api/orders/" + saved.getId(), OrderResponse.class
        );
        ResponseEntity<OrderResponse> secondResponse = restTemplate.getForEntity(
            "/api/orders/" + saved.getId(), OrderResponse.class
        );

        // Assert - ทั้งสองควรได้ผลเหมือนกัน (อันที่สองมาจาก cache)
        assertThat(firstResponse.getStatusCode()).isEqualTo(HttpStatus.OK);
        assertThat(secondResponse.getStatusCode()).isEqualTo(HttpStatus.OK);
        assertThat(firstResponse.getBody().getId()).isEqualTo(secondResponse.getBody().getId());
    }
}
```

### Kafka Integration Test

```java
// OrderKafkaIntegrationTest.java
@SpringBootTest
@Testcontainers
@EmbeddedKafka(partitions = 1, topics = {"order-events"})
class OrderKafkaIntegrationTest {

    @Autowired
    private OrderService orderService;

    @Autowired
    private KafkaTemplate<String, Object> kafkaTemplate;

    private KafkaConsumer<String, String> consumer;

    @BeforeEach
    void setUp() {
        Properties props = new Properties();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, embeddedKafka.getBrokersAsString());
        props.put(ConsumerConfig.GROUP_ID_CONFIG, "test-group");
        props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);

        consumer = new KafkaConsumer<>(props);
        consumer.subscribe(Collections.singletonList("order-events"));
    }

    @Test
    @DisplayName("สร้าง Order ควร publish event ไปยัง Kafka")
    void createOrder_ShouldPublishKafkaEvent() throws Exception {
        // Arrange
        CreateOrderRequest request = /* ... */;

        // Act
        orderService.createOrder(request);

        // Assert - ตรวจสอบ Kafka message
        await()
            .atMost(Duration.ofSeconds(10))
            .until(() -> {
                ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(500));
                return records.count() > 0;
            });

        ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(1000));
        assertThat(records.count()).isGreaterThan(0);

        ConsumerRecord<String, String> record = records.iterator().next();
        assertThat(record.topic()).isEqualTo("order-events");

        ObjectMapper mapper = new ObjectMapper();
        OrderEvent event = mapper.readValue(record.value(), OrderEvent.class);
        assertThat(event.getEventType()).isEqualTo("ORDER_CREATED");
    }
}
```

---

## ขั้นตอนที่ 2273-2276: Contract Testing กับ Spring Cloud Contract

### Producer Side (Server) - กำหนด Contract

```groovy
// src/test/resources/contracts/order/shouldReturnOrderById.groovy
import org.springframework.cloud.contract.spec.Contract

Contract.make {
    description "ควร return Order เมื่อส่ง valid order id"

    request {
        method GET()
        url "/api/orders/1"
        headers {
            contentType applicationJson()
            header "Authorization": "Bearer valid-token"
        }
    }

    response {
        status OK()
        headers {
            contentType applicationJson()
        }
        body(
            id: 1,
            customerId: "customer-001",
            status: "CONFIRMED",
            totalAmount: 500.00
        )
        bodyMatchers {
            jsonPath('$.id', byType())
            jsonPath('$.status', byRegex('PENDING|CONFIRMED|CANCELLED|DELIVERED'))
        }
    }
}
```

```groovy
// src/test/resources/contracts/order/shouldReturn404WhenOrderNotFound.groovy
Contract.make {
    description "ควร return 404 เมื่อไม่พบ Order"

    request {
        method GET()
        url "/api/orders/999"
    }

    response {
        status NOT_FOUND()
        headers {
            contentType applicationJson()
        }
        body(
            message: $(consumer(anyNonBlankString()), producer("Order not found: 999")),
            errorCode: "ORDER_NOT_FOUND"
        )
    }
}
```

### Contract Test Base Class

```java
// ContractVerifierBase.java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@AutoConfigureMessageVerifier
public abstract class ContractVerifierBase {

    @Autowired
    private OrderController orderController;

    @MockBean
    private OrderService orderService;

    @LocalServerPort
    private int port;

    @BeforeEach
    void setUp() {
        RestAssured.baseURI = "http://localhost:" + port;

        // กำหนด stub data สำหรับ contract tests
        Order order = new Order();
        order.setId(1L);
        order.setCustomerId("customer-001");
        order.setStatus(OrderStatus.CONFIRMED);
        order.setTotalAmount(new BigDecimal("500.00"));

        when(orderService.findById(1L)).thenReturn(Optional.of(order));
        when(orderService.findById(999L))
            .thenThrow(new OrderNotFoundException(999L));
    }
}
```

### Consumer Side - ใช้ Contract เป็น Stub

```java
// OrderClientTest.java - Consumer ทดสอบ API Client
@SpringBootTest
@AutoConfigureStubRunner(
    ids = "com.example:order-service:+:stubs:8090",
    stubsMode = StubRunnerProperties.StubsMode.LOCAL
)
class OrderClientTest {

    @Autowired
    private OrderServiceClient orderServiceClient;

    @Test
    @DisplayName("OrderClient ควร parse response จาก Order Service ได้ถูกต้อง")
    void getOrder_ShouldReturnOrderFromStub() {
        // Act - เรียก stub server ที่มาจาก contract
        OrderDto order = orderServiceClient.getOrder(1L);

        // Assert
        assertThat(order.getId()).isEqualTo(1L);
        assertThat(order.getCustomerId()).isEqualTo("customer-001");
        assertThat(order.getStatus()).isEqualTo("CONFIRMED");
    }

    @Test
    @DisplayName("OrderClient ควรจัดการ 404 ได้ถูกต้อง")
    void getOrder_WhenNotFound_ShouldThrowException() {
        assertThatThrownBy(() -> orderServiceClient.getOrder(999L))
            .isInstanceOf(OrderNotFoundException.class);
    }
}
```

---

## ขั้นตอนที่ 2277-2280: Mutation Testing กับ PIT

### ทำความเข้าใจ Mutation Testing

Mutation Testing คือการทดสอบคุณภาพของ Test Suite โดยการ "สร้าง bug" ใน code แล้วดูว่า tests สามารถจับ bug เหล่านั้นได้หรือไม่

```
Code เดิม:                    Mutation:
if (stock >= quantity)  →     if (stock > quantity)   (เปลี่ยน >= เป็น >)
return total * 2        →     return total + 2        (เปลี่ยน * เป็น +)
list.add(item)         →     // list.add(item)        (ลบ statement)
```

ถ้า Test สามารถ "ฆ่า" (kill) mutation ได้ = Test ดี
ถ้า mutation "รอด" (survive) = Test ต้องปรับปรุง

### Configuration PIT Mutation Testing

```xml
<!-- pom.xml -->
<plugin>
    <groupId>org.pitest</groupId>
    <artifactId>pitest-maven</artifactId>
    <version>1.15.0</version>
    <dependencies>
        <dependency>
            <groupId>org.pitest</groupId>
            <artifactId>pitest-junit5-plugin</artifactId>
            <version>1.2.1</version>
        </dependency>
    </dependencies>
    <configuration>
        <!-- กำหนด classes ที่จะทดสอบ -->
        <targetClasses>
            <param>com.example.order.service.*</param>
            <param>com.example.order.domain.*</param>
        </targetClasses>
        <!-- กำหนด test classes -->
        <targetTests>
            <param>com.example.order.*Test</param>
        </targetTests>
        <!-- Mutation Operators ที่จะใช้ -->
        <mutators>
            <mutator>STRONGER</mutator>
        </mutators>
        <!-- Threshold requirements -->
        <mutationThreshold>75</mutationThreshold>
        <coverageThreshold>80</coverageThreshold>
        <!-- Output format -->
        <outputFormats>
            <outputFormat>HTML</outputFormat>
            <outputFormat>XML</outputFormat>
        </outputFormats>
        <!-- Exclude generated code -->
        <excludedClasses>
            <excludedClass>*.config.*</excludedClass>
            <excludedClass>*.*Application</excludedClass>
        </excludedClasses>
        <!-- Parallel execution -->
        <threads>4</threads>
        <!-- Timeout factor -->
        <timeoutFactor>1.25</timeoutFactor>
        <timeoutConst>3000</timeoutConst>
    </configuration>
</plugin>
```

### รัน Mutation Testing

```bash
# รัน mutation testing
mvn test-compile org.pitest:pitest-maven:mutationCoverage

# รัน เฉพาะ class ที่ต้องการ
mvn org.pitest:pitest-maven:mutationCoverage \
    -DtargetClasses="com.example.order.service.OrderService" \
    -DtargetTests="com.example.order.service.OrderServiceTest"

# Report อยู่ที่: target/pit-reports/*/index.html
```

### Test Coverage Configuration

```yaml
# .github/workflows/test.yml
name: Test Coverage Check

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Run tests with coverage
        run: mvn test jacoco:report

      - name: Check coverage threshold
        run: mvn jacoco:check

      - name: Run mutation testing
        run: mvn org.pitest:pitest-maven:mutationCoverage

      - name: Upload coverage report
        uses: codecov/codecov-action@v3
        with:
          file: target/site/jacoco/jacoco.xml
```

```xml
<!-- JaCoCo Coverage Check -->
<plugin>
    <groupId>org.jacoco</groupId>
    <artifactId>jacoco-maven-plugin</artifactId>
    <configuration>
        <rules>
            <rule>
                <element>CLASS</element>
                <excludes>
                    <exclude>*.*Application</exclude>
                    <exclude>*.config.*</exclude>
                    <exclude>*.dto.*</exclude>
                </excludes>
                <limits>
                    <limit>
                        <counter>LINE</counter>
                        <value>COVEREDRATIO</value>
                        <minimum>0.85</minimum>
                    </limit>
                    <limit>
                        <counter>BRANCH</counter>
                        <value>COVEREDRATIO</value>
                        <minimum>0.80</minimum>
                    </limit>
                </limits>
            </rule>
        </rules>
    </configuration>
</plugin>
```

---

## สรุป Part 66: Testing Strategies

### Best Practices สำหรับ Testing Strategy

| แนวทาง | รายละเอียด |
|--------|-----------|
| **Test Naming** | ใช้ `methodName_condition_expectedResult` |
| **Arrange-Act-Assert** | แยก 3 ส่วนให้ชัดเจนด้วย comment |
| **Test Data** | ใช้ Builder Pattern หรือ Factory Method |
| **Mocking** | Mock ที่ boundary เท่านั้น, ไม่ mock internal |
| **Coverage** | 85% line, 80% branch เป็น minimum |
| **Speed** | Unit < 10ms, Integration < 1s, E2E < 30s |

### Testing Checklist

- [ ] Unit tests ครอบคลุม business logic ทั้งหมด
- [ ] Integration tests ทดสอบกับ real database/cache
- [ ] Contract tests ป้องกัน breaking changes
- [ ] Mutation score > 75%
- [ ] Test coverage > 85%
- [ ] No flaky tests (tests ที่ fail บางครั้ง)
- [ ] Tests รันได้ใน CI/CD pipeline

---

*[← Part 65: Security Advanced](./part-65-security-advanced.md) | [Part 67: Logging Best Practices →](./part-67-logging-best-practices.md)*

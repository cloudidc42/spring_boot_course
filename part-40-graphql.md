# Part 40: GraphQL with Spring Boot
## ขั้นตอนที่ 1201-1240

> **ระดับ:** ระดับโลก (World-Class)  
> **เวลาเรียน:** 5-6 ชั่วโมง  
> **เป้าหมาย:** Build flexible APIs ด้วย GraphQL

---

## ขั้นตอนที่ 1201: GraphQL vs REST

```
REST:
  GET /users/1            → returns all user fields
  GET /users/1/orders     → extra request needed
  GET /users/1/profile    → extra request needed
  Over-fetching, Under-fetching problem

GraphQL:
  Single endpoint: POST /graphql
  Client specifies exactly what it needs:
  
  query {
    user(id: 1) {
      name
      email
      orders {
        id
        totalAmount
        status
      }
    }
  }
  
Benefits:
  ✅ No over-fetching
  ✅ No under-fetching
  ✅ Single request for related data
  ✅ Self-documenting (schema)
  ✅ Strong typing
  
Trade-offs:
  ❌ More complex than REST
  ❌ N+1 problem (use DataLoader)
  ❌ Caching is harder
  ❌ Large learning curve
```

---

## ขั้นตอนที่ 1202: Dependencies

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-graphql</artifactId>
</dependency>

<!-- WebSocket support for subscriptions -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-websocket</artifactId>
</dependency>
```

---

## ขั้นตอนที่ 1203: Schema Definition

```graphql
# src/main/resources/graphql/schema.graphqls

# Scalars
scalar DateTime
scalar BigDecimal
scalar UUID

# Types
type Product {
    id: ID!
    name: String!
    description: String
    price: BigDecimal!
    stock: Int!
    status: ProductStatus!
    category: Category
    reviews: [Review!]!
    averageRating: Float
    createdAt: DateTime!
}

type Category {
    id: ID!
    name: String!
    description: String
    products(page: Int, size: Int): ProductPage!
}

type Review {
    id: ID!
    rating: Int!
    comment: String
    author: User!
    createdAt: DateTime!
}

type User {
    id: ID!
    username: String!
    email: String!
    orders: [Order!]!
}

type Order {
    id: ID!
    orderNumber: String!
    status: OrderStatus!
    totalAmount: BigDecimal!
    items: [OrderItem!]!
    user: User!
    createdAt: DateTime!
}

type OrderItem {
    product: Product!
    quantity: Int!
    unitPrice: BigDecimal!
    subtotal: BigDecimal!
}

# Pagination
type ProductPage {
    content: [Product!]!
    totalElements: Int!
    totalPages: Int!
    pageNumber: Int!
    pageSize: Int!
}

# Enums
enum ProductStatus { ACTIVE, INACTIVE, OUT_OF_STOCK }
enum OrderStatus { PENDING, CONFIRMED, SHIPPED, DELIVERED, CANCELLED }

# Input types
input CreateProductInput {
    name: String!
    description: String
    price: BigDecimal!
    stock: Int!
    categoryId: ID
}

input UpdateProductInput {
    name: String
    description: String
    price: BigDecimal
    stock: Int
}

input ProductFilter {
    keyword: String
    status: ProductStatus
    categoryId: ID
    minPrice: BigDecimal
    maxPrice: BigDecimal
}

# Root types
type Query {
    product(id: ID!): Product
    products(filter: ProductFilter, page: Int = 0, size: Int = 20): ProductPage!
    category(id: ID!): Category
    categories: [Category!]!
    me: User
    order(id: ID!): Order
    myOrders: [Order!]!
}

type Mutation {
    createProduct(input: CreateProductInput!): Product!
    updateProduct(id: ID!, input: UpdateProductInput!): Product!
    deleteProduct(id: ID!): Boolean!
    
    createOrder(items: [OrderItemInput!]!): Order!
    cancelOrder(id: ID!, reason: String): Order!
}

type Subscription {
    orderStatusChanged(orderId: ID!): Order!
    newOrderForAdmin: Order!
}
```

---

## ขั้นตอนที่ 1204: Query Resolvers

```java
@Controller
@RequiredArgsConstructor
public class ProductGraphQLController {
    
    private final ProductService productService;
    
    @QueryMapping
    public Product product(@Argument Long id) {
        return productService.findById(id);
    }
    
    @QueryMapping
    public ProductPage products(
        @Argument ProductFilter filter,
        @Argument int page,
        @Argument int size
    ) {
        return productService.search(filter, page, size);
    }
    
    // Field resolver for nested type
    @SchemaMapping(typeName = "Product", field = "reviews")
    public List<Review> reviews(Product product) {
        // N+1 problem here - fix with DataLoader
        return reviewService.findByProductId(product.id());
    }
    
    @SchemaMapping(typeName = "Product", field = "averageRating")
    public Double averageRating(Product product) {
        return reviewService.getAverageRating(product.id());
    }
    
    @SchemaMapping(typeName = "Category", field = "products")
    public ProductPage categoryProducts(
        Category category,
        @Argument int page,
        @Argument int size
    ) {
        return productService.findByCategory(category.id(), page, size);
    }
}

@Controller
@RequiredArgsConstructor
public class OrderGraphQLController {
    
    private final OrderService orderService;
    
    @QueryMapping
    @PreAuthorize("isAuthenticated()")
    public List<Order> myOrders(Authentication auth) {
        return orderService.findByUser(auth.getName());
    }
    
    @MutationMapping
    @PreAuthorize("isAuthenticated()")
    public Order createOrder(
        @Argument List<OrderItemInput> items,
        Authentication auth
    ) {
        return orderService.create(auth.getName(), items);
    }
    
    @MutationMapping
    public Order cancelOrder(@Argument Long id, @Argument String reason) {
        return orderService.cancel(id, reason);
    }
}
```

---

## ขั้นตอนที่ 1205: DataLoader - Fix N+1 Problem

```java
@Configuration
public class DataLoaderConfig {
    
    @Bean
    public BatchLoaderRegistry batchLoaderRegistry(ReviewRepository reviewRepository) {
        return registry -> {
            // Batch load reviews for multiple products at once
            registry.<Long, List<Review>>forTypePair(Long.class, List.class)
                .withName("reviewsByProductId")
                .registerBatchLoader((productIds, env) -> {
                    
                    Map<Long, List<Review>> reviewMap = reviewRepository
                        .findByProductIdIn(new HashSet<>(productIds))
                        .stream()
                        .collect(Collectors.groupingBy(r -> r.getProductId()));
                    
                    return Mono.just(
                        productIds.stream()
                            .map(id -> reviewMap.getOrDefault(id, List.of()))
                            .toList()
                    );
                });
        };
    }
}

// Use DataLoader in resolver
@SchemaMapping(typeName = "Product", field = "reviews")
public CompletableFuture<List<Review>> reviews(
    Product product,
    DataLoader<Long, List<Review>> reviewsLoader
) {
    return reviewsLoader.load(product.id());
}
```

---

## ขั้นตอนที่ 1206: Mutations

```java
@Controller
@RequiredArgsConstructor
public class ProductMutationController {
    
    private final ProductService productService;
    
    @MutationMapping
    @PreAuthorize("hasRole('ADMIN')")
    public Product createProduct(@Argument @Valid CreateProductInput input) {
        return productService.create(input);
    }
    
    @MutationMapping
    @PreAuthorize("hasRole('ADMIN')")
    public Product updateProduct(
        @Argument Long id,
        @Argument UpdateProductInput input
    ) {
        return productService.update(id, input);
    }
    
    @MutationMapping
    @PreAuthorize("hasRole('ADMIN')")
    public boolean deleteProduct(@Argument Long id) {
        productService.delete(id);
        return true;
    }
}
```

---

## ขั้นตอนที่ 1207: Subscriptions (Real-time)

```java
@Controller
@RequiredArgsConstructor
public class OrderSubscriptionController {
    
    private final OrderService orderService;
    
    @SubscriptionMapping
    @PreAuthorize("isAuthenticated()")
    public Flux<Order> orderStatusChanged(
        @Argument Long orderId,
        Authentication auth
    ) {
        return orderService.subscribeToOrderStatus(orderId, auth.getName());
    }
    
    @SubscriptionMapping
    @PreAuthorize("hasRole('ADMIN')")
    public Flux<Order> newOrderForAdmin() {
        return orderService.subscribeToNewOrders();
    }
}

// In OrderService
@Service
public class OrderService {
    
    private final Sinks.Many<Order> orderSink = Sinks.many().multicast().directAllOrNothing();
    
    public Flux<Order> subscribeToOrderStatus(Long orderId, String userId) {
        return orderSink.asFlux()
            .filter(o -> o.getId().equals(orderId))
            .filter(o -> o.getUserId().equals(getUserId(userId)));
    }
    
    public Flux<Order> subscribeToNewOrders() {
        return orderSink.asFlux();
    }
    
    @Transactional
    public Order create(String username, List<OrderItemInput> items) {
        Order order = // create order...
        
        // Publish to subscribers
        orderSink.tryEmitNext(order);
        
        return order;
    }
}
```

---

## ขั้นตอนที่ 1208: Configuration

```yaml
# application.yml
spring:
  graphql:
    path: /graphql
    websocket:
      path: /graphql-ws
    graphiql:
      enabled: true  # GraphiQL IDE at /graphiql
      path: /graphiql
    schema:
      locations: classpath:graphql/**
      file-extensions: .graphqls,.gqls
    cors:
      allowed-origins: "http://localhost:3000"
```

---

## ขั้นตอนที่ 1209-1240: Error Handling

```java
@ControllerAdvice
public class GraphQLExceptionHandler {
    
    @ExceptionHandler(ResourceNotFoundException.class)
    public GraphQLError handleNotFound(ResourceNotFoundException ex) {
        return GraphQLError.newError()
            .errorType(ErrorType.NOT_FOUND)
            .message(ex.getMessage())
            .build();
    }
    
    @ExceptionHandler(ValidationException.class)
    public GraphQLError handleValidation(ValidationException ex) {
        return GraphQLError.newError()
            .errorType(ErrorType.BAD_REQUEST)
            .message(ex.getMessage())
            .build();
    }
    
    @ExceptionHandler(AccessDeniedException.class)
    public GraphQLError handleForbidden(AccessDeniedException ex) {
        return GraphQLError.newError()
            .errorType(ErrorType.FORBIDDEN)
            .message("Access denied")
            .build();
    }
}

/*
 * GraphQL Query Example (client side):
 * 
 * query GetProductWithReviews($id: ID!) {
 *   product(id: $id) {
 *     id
 *     name
 *     price
 *     reviews {
 *       rating
 *       comment
 *       author {
 *         username
 *       }
 *     }
 *     averageRating
 *   }
 * }
 * 
 * mutation CreateProduct($input: CreateProductInput!) {
 *   createProduct(input: $input) {
 *     id
 *     name
 *     price
 *   }
 * }
 */
```

---

*[← Part 39: Saga Pattern](./part-39-saga.md) | [Part 41: Service Mesh →](./part-41-service-mesh.md)*

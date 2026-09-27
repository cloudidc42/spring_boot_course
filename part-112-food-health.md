# Part 112: โปรเจค 56-60 — Food & Health Applications

**ระดับ:** ระดับโลก (World-Class)
**เวลา:** 10-15 ชั่วโมง
**เป้าหมาย:** สร้างแอปพลิเคชันด้านอาหารและสุขภาพที่ครบครัน ตั้งแต่ Food Delivery, Recipe Management, Meal Planner, Pharmacy Management จนถึง Blood Bank Management System

---

*[← Part 111: Loyalty & Marketing](./part-111-loyalty-marketing.md) | [Part 113: Fitness & Wellness →](./part-113-fitness-wellness.md)*

---

## โปรเจค 56: Food Delivery App

### ภาพรวมระบบ

Food Delivery App เป็นระบบสั่งอาหารออนไลน์ที่รองรับทุกขั้นตอน ตั้งแต่การเรียกดูร้านอาหารและเมนู การเพิ่มสินค้าในตะกร้า การสั่งซื้อ การจัดสรรคนส่ง ไปจนถึงการติดตามสถานะการจัดส่งแบบ Real-time ระบบยังรองรับ Surge Pricing ในช่วงที่มีความต้องการสูง

- **ร้านอาหารและเมนู**: จัดการร้านและรายการเมนูอาหาร
- **ตะกร้าสินค้า**: เพิ่ม/ลบรายการ ปรับจำนวน
- **การสั่งซื้อ**: สร้างคำสั่งซื้อพร้อม Payment
- **การจัดสรรคนส่ง**: จัดสรรไรเดอร์อัตโนมัติตามตำแหน่ง
- **Delivery Tracking**: ติดตามสถานะแบบ Real-time
- **Surge Pricing**: ปรับราคาค่าส่งตาม Demand

### Flyway Migration

```sql
-- V1__create_food_delivery_tables.sql
CREATE TABLE restaurants (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    description TEXT,
    cuisine_type VARCHAR(100),
    address TEXT NOT NULL,
    latitude DECIMAL(10,8),
    longitude DECIMAL(11,8),
    phone VARCHAR(20),
    image_url VARCHAR(500),
    rating DECIMAL(3,2) DEFAULT 0,
    total_ratings INT DEFAULT 0,
    min_order_amount DECIMAL(8,2) DEFAULT 0,
    delivery_fee DECIMAL(8,2) DEFAULT 0,
    estimated_delivery_minutes INT DEFAULT 30,
    is_open BOOLEAN DEFAULT TRUE,
    active BOOLEAN DEFAULT TRUE,
    owner_id BIGINT,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE menu_categories (
    id BIGSERIAL PRIMARY KEY,
    restaurant_id BIGINT NOT NULL REFERENCES restaurants(id),
    name VARCHAR(100) NOT NULL,
    display_order INT DEFAULT 0,
    active BOOLEAN DEFAULT TRUE
);

CREATE TABLE menu_items (
    id BIGSERIAL PRIMARY KEY,
    restaurant_id BIGINT NOT NULL REFERENCES restaurants(id),
    category_id BIGINT REFERENCES menu_categories(id),
    name VARCHAR(200) NOT NULL,
    description TEXT,
    price DECIMAL(10,2) NOT NULL,
    image_url VARCHAR(500),
    is_available BOOLEAN DEFAULT TRUE,
    preparation_time_mins INT DEFAULT 15,
    calories INT,
    is_vegetarian BOOLEAN DEFAULT FALSE,
    is_spicy BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE orders (
    id BIGSERIAL PRIMARY KEY,
    order_number VARCHAR(30) NOT NULL UNIQUE,
    customer_id BIGINT NOT NULL,
    restaurant_id BIGINT NOT NULL REFERENCES restaurants(id),
    driver_id BIGINT,
    status VARCHAR(30) NOT NULL DEFAULT 'PENDING',
    delivery_address TEXT NOT NULL,
    delivery_latitude DECIMAL(10,8),
    delivery_longitude DECIMAL(11,8),
    subtotal DECIMAL(10,2) NOT NULL,
    delivery_fee DECIMAL(8,2) NOT NULL,
    surge_multiplier DECIMAL(4,2) DEFAULT 1.0,
    total_amount DECIMAL(10,2) NOT NULL,
    special_instructions TEXT,
    estimated_delivery_at TIMESTAMP,
    delivered_at TIMESTAMP,
    cancelled_at TIMESTAMP,
    cancellation_reason VARCHAR(500),
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE order_items (
    id BIGSERIAL PRIMARY KEY,
    order_id BIGINT NOT NULL REFERENCES orders(id),
    menu_item_id BIGINT NOT NULL REFERENCES menu_items(id),
    name VARCHAR(200) NOT NULL,
    price DECIMAL(10,2) NOT NULL,
    quantity INT NOT NULL,
    subtotal DECIMAL(10,2) NOT NULL,
    special_request TEXT
);

CREATE TABLE drivers (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL UNIQUE,
    name VARCHAR(200) NOT NULL,
    phone VARCHAR(20) NOT NULL,
    vehicle_type VARCHAR(50),
    license_plate VARCHAR(20),
    status VARCHAR(20) NOT NULL DEFAULT 'OFFLINE',
    current_latitude DECIMAL(10,8),
    current_longitude DECIMAL(11,8),
    rating DECIMAL(3,2) DEFAULT 5.0,
    total_deliveries INT DEFAULT 0,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE order_ratings (
    id BIGSERIAL PRIMARY KEY,
    order_id BIGINT NOT NULL UNIQUE REFERENCES orders(id),
    customer_id BIGINT NOT NULL,
    food_rating INT CHECK (food_rating BETWEEN 1 AND 5),
    delivery_rating INT CHECK (delivery_rating BETWEEN 1 AND 5),
    comment TEXT,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_orders_customer ON orders(customer_id);
CREATE INDEX idx_orders_restaurant ON orders(restaurant_id);
CREATE INDEX idx_orders_driver ON orders(driver_id);
CREATE INDEX idx_orders_status ON orders(status);
```

### Entity

```java
// Restaurant.java
package com.fooddelivery.entity;

import jakarta.persistence.*;
import lombok.*;
import java.math.BigDecimal;
import java.time.LocalDateTime;

@Entity
@Table(name = "restaurants")
@Getter @Setter @Builder
@NoArgsConstructor @AllArgsConstructor
public class Restaurant {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "name", nullable = false)
    private String name;

    @Column(name = "cuisine_type")
    private String cuisineType;

    @Column(name = "address", nullable = false)
    private String address;

    @Column(name = "latitude")
    private BigDecimal latitude;

    @Column(name = "longitude")
    private BigDecimal longitude;

    @Column(name = "rating")
    private BigDecimal rating = BigDecimal.ZERO;

    @Column(name = "total_ratings")
    private Integer totalRatings = 0;

    @Column(name = "min_order_amount")
    private BigDecimal minOrderAmount = BigDecimal.ZERO;

    @Column(name = "delivery_fee")
    private BigDecimal deliveryFee = BigDecimal.ZERO;

    @Column(name = "estimated_delivery_minutes")
    private Integer estimatedDeliveryMinutes = 30;

    @Column(name = "is_open")
    private Boolean isOpen = true;

    @Column(name = "active")
    private Boolean active = true;

    @Column(name = "created_at")
    private LocalDateTime createdAt = LocalDateTime.now();
}

// Order.java
@Entity
@Table(name = "orders")
@Getter @Setter @Builder
@NoArgsConstructor @AllArgsConstructor
public class Order {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "order_number", nullable = false, unique = true)
    private String orderNumber;

    @Column(name = "customer_id", nullable = false)
    private Long customerId;

    @Column(name = "restaurant_id", nullable = false)
    private Long restaurantId;

    @Column(name = "driver_id")
    private Long driverId;

    @Enumerated(EnumType.STRING)
    @Column(name = "status", nullable = false)
    private OrderStatus status = OrderStatus.PENDING;

    @Column(name = "delivery_address", nullable = false)
    private String deliveryAddress;

    @Column(name = "subtotal", nullable = false)
    private BigDecimal subtotal;

    @Column(name = "delivery_fee", nullable = false)
    private BigDecimal deliveryFee;

    @Column(name = "surge_multiplier")
    private BigDecimal surgeMultiplier = BigDecimal.ONE;

    @Column(name = "total_amount", nullable = false)
    private BigDecimal totalAmount;

    @OneToMany(cascade = CascadeType.ALL, fetch = FetchType.LAZY)
    @JoinColumn(name = "order_id")
    private java.util.List<OrderItem> items;

    @Column(name = "estimated_delivery_at")
    private LocalDateTime estimatedDeliveryAt;

    @Column(name = "delivered_at")
    private LocalDateTime deliveredAt;

    @Column(name = "created_at")
    private LocalDateTime createdAt = LocalDateTime.now();
}

public enum OrderStatus {
    PENDING, CONFIRMED, PREPARING, READY_FOR_PICKUP,
    PICKED_UP, ON_THE_WAY, DELIVERED, CANCELLED
}

// Driver.java
@Entity
@Table(name = "drivers")
@Getter @Setter @Builder
@NoArgsConstructor @AllArgsConstructor
public class Driver {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "user_id", nullable = false, unique = true)
    private Long userId;

    @Column(name = "name", nullable = false)
    private String name;

    @Column(name = "phone", nullable = false)
    private String phone;

    @Enumerated(EnumType.STRING)
    @Column(name = "status", nullable = false)
    private DriverStatus status = DriverStatus.OFFLINE;

    @Column(name = "current_latitude")
    private BigDecimal currentLatitude;

    @Column(name = "current_longitude")
    private BigDecimal currentLongitude;

    @Column(name = "rating")
    private BigDecimal rating = BigDecimal.valueOf(5.0);

    @Column(name = "total_deliveries")
    private Integer totalDeliveries = 0;
}

public enum DriverStatus { OFFLINE, AVAILABLE, BUSY }
```

### Service

```java
// FoodDeliveryService.java
package com.fooddelivery.service;

import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import java.math.BigDecimal;
import java.math.RoundingMode;
import java.time.LocalDateTime;
import java.util.*;

@Service
@RequiredArgsConstructor
public class FoodDeliveryService {

    private final RestaurantRepository restaurantRepository;
    private final MenuItemRepository menuItemRepository;
    private final OrderRepository orderRepository;
    private final OrderItemRepository orderItemRepository;
    private final DriverRepository driverRepository;

    @Transactional
    public Order placeOrder(Long customerId, Long restaurantId, String deliveryAddress,
                             List<OrderItemRequest> items, String specialInstructions) {
        Restaurant restaurant = restaurantRepository.findById(restaurantId)
                .orElseThrow(() -> new RuntimeException("Restaurant not found"));
        if (!restaurant.getIsOpen()) {
            throw new IllegalStateException("Restaurant is currently closed");
        }

        BigDecimal subtotal = BigDecimal.ZERO;
        List<OrderItem> orderItems = new ArrayList<>();

        for (OrderItemRequest itemReq : items) {
            MenuItem menuItem = menuItemRepository.findById(itemReq.menuItemId())
                    .orElseThrow(() -> new RuntimeException("Menu item not found: " + itemReq.menuItemId()));
            if (!menuItem.getIsAvailable()) {
                throw new IllegalStateException("Item unavailable: " + menuItem.getName());
            }
            BigDecimal itemSubtotal = menuItem.getPrice().multiply(BigDecimal.valueOf(itemReq.quantity()));
            subtotal = subtotal.add(itemSubtotal);
            orderItems.add(OrderItem.builder()
                    .menuItemId(menuItem.getId())
                    .name(menuItem.getName())
                    .price(menuItem.getPrice())
                    .quantity(itemReq.quantity())
                    .subtotal(itemSubtotal)
                    .specialRequest(itemReq.specialRequest())
                    .build());
        }

        if (subtotal.compareTo(restaurant.getMinOrderAmount()) < 0) {
            throw new IllegalArgumentException("Order below minimum: " + restaurant.getMinOrderAmount());
        }

        BigDecimal surgeMultiplier = calculateSurgeMultiplier(restaurantId);
        BigDecimal deliveryFee = restaurant.getDeliveryFee().multiply(surgeMultiplier)
                .setScale(2, RoundingMode.HALF_UP);
        BigDecimal total = subtotal.add(deliveryFee);

        String orderNumber = "ORD-" + System.currentTimeMillis();
        Order order = Order.builder()
                .orderNumber(orderNumber)
                .customerId(customerId)
                .restaurantId(restaurantId)
                .status(OrderStatus.PENDING)
                .deliveryAddress(deliveryAddress)
                .subtotal(subtotal)
                .deliveryFee(deliveryFee)
                .surgeMultiplier(surgeMultiplier)
                .totalAmount(total)
                .specialInstructions(specialInstructions)
                .estimatedDeliveryAt(LocalDateTime.now().plusMinutes(
                    restaurant.getEstimatedDeliveryMinutes()))
                .items(orderItems)
                .build();
        order = orderRepository.save(order);

        // Auto assign driver
        assignDriver(order);
        return order;
    }

    @Transactional
    public Order updateOrderStatus(Long orderId, OrderStatus newStatus) {
        Order order = orderRepository.findById(orderId)
                .orElseThrow(() -> new RuntimeException("Order not found"));
        order.setStatus(newStatus);
        if (newStatus == OrderStatus.DELIVERED) {
            order.setDeliveredAt(LocalDateTime.now());
            if (order.getDriverId() != null) {
                driverRepository.findById(order.getDriverId()).ifPresent(driver -> {
                    driver.setStatus(DriverStatus.AVAILABLE);
                    driver.setTotalDeliveries(driver.getTotalDeliveries() + 1);
                    driverRepository.save(driver);
                });
            }
        }
        return orderRepository.save(order);
    }

    private void assignDriver(Order order) {
        List<Driver> availableDrivers = driverRepository.findByStatus(DriverStatus.AVAILABLE);
        if (!availableDrivers.isEmpty()) {
            Driver driver = availableDrivers.get(0); // Simplified: first available
            driver.setStatus(DriverStatus.BUSY);
            driverRepository.save(driver);
            order.setDriverId(driver.getId());
            order.setStatus(OrderStatus.CONFIRMED);
            orderRepository.save(order);
        }
    }

    private BigDecimal calculateSurgeMultiplier(Long restaurantId) {
        // Count active orders in last 30 mins
        LocalDateTime threshold = LocalDateTime.now().minusMinutes(30);
        long activeOrders = orderRepository.countByRestaurantIdAndCreatedAtAfterAndStatusNot(
            restaurantId, threshold, OrderStatus.CANCELLED);
        if (activeOrders > 50) return BigDecimal.valueOf(1.5);
        if (activeOrders > 30) return BigDecimal.valueOf(1.25);
        return BigDecimal.ONE;
    }

    public List<Restaurant> searchRestaurants(String query, String cuisineType) {
        if (cuisineType != null) return restaurantRepository.findByCuisineTypeAndActiveTrue(cuisineType);
        if (query != null) return restaurantRepository.findByNameContainingIgnoreCaseAndActiveTrue(query);
        return restaurantRepository.findByActiveTrueOrderByRatingDesc();
    }

    public List<MenuItem> getMenuByRestaurant(Long restaurantId) {
        return menuItemRepository.findByRestaurantIdAndIsAvailableTrue(restaurantId);
    }

    public record OrderItemRequest(Long menuItemId, int quantity, String specialRequest) {}
}
```

### Controller

```java
// FoodDeliveryController.java
package com.fooddelivery.controller;

import lombok.RequiredArgsConstructor;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import java.util.List;

@RestController
@RequestMapping("/api/v1/food")
@RequiredArgsConstructor
public class FoodDeliveryController {

    private final FoodDeliveryService service;

    @GetMapping("/restaurants")
    public ResponseEntity<List<Restaurant>> searchRestaurants(
            @RequestParam(required = false) String q,
            @RequestParam(required = false) String cuisine) {
        return ResponseEntity.ok(service.searchRestaurants(q, cuisine));
    }

    @GetMapping("/restaurants/{id}/menu")
    public ResponseEntity<List<MenuItem>> getMenu(@PathVariable Long id) {
        return ResponseEntity.ok(service.getMenuByRestaurant(id));
    }

    @PostMapping("/orders")
    public ResponseEntity<Order> placeOrder(@RequestBody PlaceOrderRequest request) {
        return ResponseEntity.ok(service.placeOrder(
            request.customerId(), request.restaurantId(), request.deliveryAddress(),
            request.items(), request.specialInstructions()));
    }

    @PutMapping("/orders/{orderId}/status")
    public ResponseEntity<Order> updateStatus(@PathVariable Long orderId,
            @RequestParam OrderStatus status) {
        return ResponseEntity.ok(service.updateOrderStatus(orderId, status));
    }

    @GetMapping("/orders/{orderId}")
    public ResponseEntity<Order> getOrder(@PathVariable Long orderId) {
        return ResponseEntity.ok(service.getOrder(orderId));
    }

    record PlaceOrderRequest(Long customerId, Long restaurantId, String deliveryAddress,
                              List<FoodDeliveryService.OrderItemRequest> items,
                              String specialInstructions) {}
}
```

### docker-compose.yml

```yaml
# docker-compose.yml (Food Delivery App)
version: '3.8'
services:
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: fooddelivery_db
      POSTGRES_USER: food_user
      POSTGRES_PASSWORD: food_pass
    ports:
      - "5432:5432"
    volumes:
      - food_pg_data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

  kafka:
    image: confluentinc/cp-kafka:7.5.0
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
    ports:
      - "9092:9092"
    depends_on:
      - zookeeper

  zookeeper:
    image: confluentinc/cp-zookeeper:7.5.0
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181

  food-delivery-service:
    build: .
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/fooddelivery_db
      SPRING_DATASOURCE_USERNAME: food_user
      SPRING_DATASOURCE_PASSWORD: food_pass
      SPRING_REDIS_HOST: redis
      SPRING_KAFKA_BOOTSTRAP_SERVERS: kafka:9092
    ports:
      - "8080:8080"
    depends_on:
      - postgres
      - redis
      - kafka

volumes:
  food_pg_data:
```

---

## โปรเจค 57: Recipe Management App

### ภาพรวมระบบ

Recipe Management App เป็นแอปสำหรับจัดการสูตรอาหาร ช่วยให้ผู้ใช้สามารถบันทึกสูตรอาหารพร้อมวัตถุดิบและขั้นตอนการทำ คำนวณข้อมูลโภชนาการ วางแผนมื้ออาหาร และสร้างรายการซื้อของอัตโนมัติ

- **สูตรอาหาร**: บันทึกสูตร วัตถุดิบ ขั้นตอน
- **ข้อมูลโภชนาการ**: คำนวณแคลอรี และ Macros
- **Meal Planning**: วางแผนมื้ออาหารรายสัปดาห์
- **Shopping List**: สร้างรายการซื้อของอัตโนมัติ
- **Dietary Filters**: กรองตาม Vegan / Gluten-free ฯลฯ

### Flyway Migration

```sql
-- V1__create_recipe_tables.sql
CREATE TABLE recipes (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL,
    title VARCHAR(500) NOT NULL,
    description TEXT,
    instructions TEXT NOT NULL,
    prep_time_minutes INT,
    cook_time_minutes INT,
    servings INT DEFAULT 1,
    difficulty VARCHAR(20) DEFAULT 'MEDIUM',
    cuisine VARCHAR(100),
    image_url VARCHAR(500),
    is_public BOOLEAN DEFAULT TRUE,
    is_vegetarian BOOLEAN DEFAULT FALSE,
    is_vegan BOOLEAN DEFAULT FALSE,
    is_gluten_free BOOLEAN DEFAULT FALSE,
    total_calories DECIMAL(8,2),
    total_protein_g DECIMAL(8,2),
    total_carbs_g DECIMAL(8,2),
    total_fat_g DECIMAL(8,2),
    rating DECIMAL(3,2) DEFAULT 0,
    total_ratings INT DEFAULT 0,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE ingredients (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(200) NOT NULL UNIQUE,
    category VARCHAR(100),
    calories_per_100g DECIMAL(8,2),
    protein_per_100g DECIMAL(8,2),
    carbs_per_100g DECIMAL(8,2),
    fat_per_100g DECIMAL(8,2),
    unit VARCHAR(50) DEFAULT 'g'
);

CREATE TABLE recipe_ingredients (
    id BIGSERIAL PRIMARY KEY,
    recipe_id BIGINT NOT NULL REFERENCES recipes(id) ON DELETE CASCADE,
    ingredient_id BIGINT NOT NULL REFERENCES ingredients(id),
    quantity DECIMAL(10,3) NOT NULL,
    unit VARCHAR(50) NOT NULL,
    notes VARCHAR(200)
);

CREATE TABLE meal_plans (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL,
    name VARCHAR(200) NOT NULL,
    week_start_date DATE NOT NULL,
    week_end_date DATE NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE meal_plan_items (
    id BIGSERIAL PRIMARY KEY,
    meal_plan_id BIGINT NOT NULL REFERENCES meal_plans(id) ON DELETE CASCADE,
    recipe_id BIGINT NOT NULL REFERENCES recipes(id),
    day_of_week INT NOT NULL CHECK (day_of_week BETWEEN 1 AND 7),
    meal_type VARCHAR(20) NOT NULL,
    servings INT DEFAULT 1
);

CREATE TABLE favorite_recipes (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL,
    recipe_id BIGINT NOT NULL REFERENCES recipes(id),
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    UNIQUE (user_id, recipe_id)
);
```

### Entity & Service

```java
// Recipe.java
package com.recipe.entity;

import jakarta.persistence.*;
import lombok.*;
import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.List;

@Entity
@Table(name = "recipes")
@Getter @Setter @Builder
@NoArgsConstructor @AllArgsConstructor
public class Recipe {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "user_id", nullable = false)
    private Long userId;

    @Column(name = "title", nullable = false)
    private String title;

    @Column(name = "description")
    private String description;

    @Column(name = "instructions", nullable = false, columnDefinition = "TEXT")
    private String instructions;

    @Column(name = "prep_time_minutes")
    private Integer prepTimeMinutes;

    @Column(name = "cook_time_minutes")
    private Integer cookTimeMinutes;

    @Column(name = "servings")
    private Integer servings = 1;

    @Column(name = "difficulty")
    private String difficulty = "MEDIUM";

    @Column(name = "cuisine")
    private String cuisine;

    @Column(name = "is_vegetarian")
    private Boolean isVegetarian = false;

    @Column(name = "is_vegan")
    private Boolean isVegan = false;

    @Column(name = "is_gluten_free")
    private Boolean isGlutenFree = false;

    @Column(name = "total_calories")
    private BigDecimal totalCalories;

    @Column(name = "total_protein_g")
    private BigDecimal totalProteinG;

    @Column(name = "total_carbs_g")
    private BigDecimal totalCarbsG;

    @Column(name = "total_fat_g")
    private BigDecimal totalFatG;

    @Column(name = "rating")
    private BigDecimal rating = BigDecimal.ZERO;

    @OneToMany(cascade = CascadeType.ALL, fetch = FetchType.LAZY)
    @JoinColumn(name = "recipe_id")
    private List<RecipeIngredient> ingredients;

    @Column(name = "created_at")
    private LocalDateTime createdAt = LocalDateTime.now();
}

// RecipeService.java
@Service
@RequiredArgsConstructor
public class RecipeService {

    private final RecipeRepository recipeRepository;
    private final IngredientRepository ingredientRepository;
    private final RecipeIngredientRepository recipeIngRepository;
    private final MealPlanRepository mealPlanRepository;
    private final MealPlanItemRepository mealPlanItemRepository;

    @Transactional
    public Recipe createRecipe(Long userId, CreateRecipeRequest request) {
        Recipe recipe = Recipe.builder()
                .userId(userId)
                .title(request.title())
                .description(request.description())
                .instructions(request.instructions())
                .prepTimeMinutes(request.prepTimeMinutes())
                .cookTimeMinutes(request.cookTimeMinutes())
                .servings(request.servings())
                .difficulty(request.difficulty())
                .cuisine(request.cuisine())
                .isVegetarian(request.isVegetarian())
                .isVegan(request.isVegan())
                .isGlutenFree(request.isGlutenFree())
                .build();

        List<RecipeIngredient> ingredients = new ArrayList<>();
        BigDecimal totalCal = BigDecimal.ZERO, totalProt = BigDecimal.ZERO;
        BigDecimal totalCarbs = BigDecimal.ZERO, totalFat = BigDecimal.ZERO;

        for (var ingReq : request.ingredients()) {
            Ingredient ingredient = ingredientRepository.findById(ingReq.ingredientId())
                    .orElseThrow(() -> new RuntimeException("Ingredient not found: " + ingReq.ingredientId()));
            RecipeIngredient ri = RecipeIngredient.builder()
                    .ingredientId(ingredient.getId())
                    .quantity(ingReq.quantity())
                    .unit(ingReq.unit())
                    .notes(ingReq.notes())
                    .build();
            ingredients.add(ri);

            // Calculate nutrition per quantity
            BigDecimal factor = ingReq.quantity().divide(BigDecimal.valueOf(100), 4,
                java.math.RoundingMode.HALF_UP);
            if (ingredient.getCaloriesPer100g() != null)
                totalCal = totalCal.add(ingredient.getCaloriesPer100g().multiply(factor));
            if (ingredient.getProteinPer100g() != null)
                totalProt = totalProt.add(ingredient.getProteinPer100g().multiply(factor));
            if (ingredient.getCarbsPer100g() != null)
                totalCarbs = totalCarbs.add(ingredient.getCarbsPer100g().multiply(factor));
            if (ingredient.getFatPer100g() != null)
                totalFat = totalFat.add(ingredient.getFatPer100g().multiply(factor));
        }

        recipe.setTotalCalories(totalCal.setScale(2, java.math.RoundingMode.HALF_UP));
        recipe.setTotalProteinG(totalProt.setScale(2, java.math.RoundingMode.HALF_UP));
        recipe.setTotalCarbsG(totalCarbs.setScale(2, java.math.RoundingMode.HALF_UP));
        recipe.setTotalFatG(totalFat.setScale(2, java.math.RoundingMode.HALF_UP));
        recipe.setIngredients(ingredients);

        return recipeRepository.save(recipe);
    }

    @Transactional
    public MealPlan createMealPlan(Long userId, String name, java.time.LocalDate weekStart) {
        MealPlan plan = MealPlan.builder()
                .userId(userId)
                .name(name)
                .weekStartDate(weekStart)
                .weekEndDate(weekStart.plusDays(6))
                .build();
        return mealPlanRepository.save(plan);
    }

    @Transactional
    public MealPlanItem addMealToPlan(Long planId, Long recipeId, int dayOfWeek,
                                       String mealType, int servings) {
        MealPlanItem item = MealPlanItem.builder()
                .mealPlanId(planId).recipeId(recipeId)
                .dayOfWeek(dayOfWeek).mealType(mealType).servings(servings).build();
        return mealPlanItemRepository.save(item);
    }

    public List<ShoppingItem> generateShoppingList(Long mealPlanId) {
        List<MealPlanItem> planItems = mealPlanItemRepository.findByMealPlanId(mealPlanId);
        Map<Long, ShoppingItem> aggregated = new HashMap<>();

        for (MealPlanItem planItem : planItems) {
            Recipe recipe = recipeRepository.findById(planItem.getRecipeId()).orElseThrow();
            for (RecipeIngredient ri : recipe.getIngredients()) {
                BigDecimal scaledQty = ri.getQuantity()
                        .multiply(BigDecimal.valueOf(planItem.getServings()));
                aggregated.merge(ri.getIngredientId(),
                    new ShoppingItem(ri.getIngredientId(), "", scaledQty, ri.getUnit()),
                    (existing, newItem) -> new ShoppingItem(existing.ingredientId(),
                        existing.name(), existing.quantity().add(newItem.quantity()), existing.unit()));
            }
        }

        return new ArrayList<>(aggregated.values());
    }

    public List<Recipe> searchRecipes(String query, Boolean vegetarian, Boolean vegan,
                                       Boolean glutenFree, String cuisine) {
        return recipeRepository.searchWithFilters(query, vegetarian, vegan, glutenFree, cuisine);
    }

    public record CreateRecipeRequest(String title, String description, String instructions,
        Integer prepTimeMinutes, Integer cookTimeMinutes, Integer servings, String difficulty,
        String cuisine, Boolean isVegetarian, Boolean isVegan, Boolean isGlutenFree,
        List<IngredientRequest> ingredients) {}
    public record IngredientRequest(Long ingredientId, BigDecimal quantity, String unit, String notes) {}
    public record ShoppingItem(Long ingredientId, String name, BigDecimal quantity, String unit) {}
}
```

### Controller

```java
// RecipeController.java
@RestController
@RequestMapping("/api/v1/recipes")
@RequiredArgsConstructor
public class RecipeController {

    private final RecipeService recipeService;

    @PostMapping
    public ResponseEntity<Recipe> create(@RequestParam Long userId,
            @RequestBody RecipeService.CreateRecipeRequest request) {
        return ResponseEntity.ok(recipeService.createRecipe(userId, request));
    }

    @GetMapping("/search")
    public ResponseEntity<List<Recipe>> search(
            @RequestParam(required = false) String q,
            @RequestParam(required = false) Boolean vegetarian,
            @RequestParam(required = false) Boolean vegan,
            @RequestParam(required = false) Boolean glutenFree,
            @RequestParam(required = false) String cuisine) {
        return ResponseEntity.ok(recipeService.searchRecipes(q, vegetarian, vegan, glutenFree, cuisine));
    }

    @PostMapping("/meal-plans")
    public ResponseEntity<MealPlan> createMealPlan(@RequestParam Long userId,
            @RequestParam String name, @RequestParam java.time.LocalDate weekStart) {
        return ResponseEntity.ok(recipeService.createMealPlan(userId, name, weekStart));
    }

    @PostMapping("/meal-plans/{planId}/items")
    public ResponseEntity<MealPlanItem> addToMealPlan(@PathVariable Long planId,
            @RequestBody AddMealRequest request) {
        return ResponseEntity.ok(recipeService.addMealToPlan(
            planId, request.recipeId(), request.dayOfWeek(), request.mealType(), request.servings()));
    }

    @GetMapping("/meal-plans/{planId}/shopping-list")
    public ResponseEntity<List<RecipeService.ShoppingItem>> getShoppingList(
            @PathVariable Long planId) {
        return ResponseEntity.ok(recipeService.generateShoppingList(planId));
    }

    record AddMealRequest(Long recipeId, int dayOfWeek, String mealType, int servings) {}
}
```

### docker-compose.yml

```yaml
# docker-compose.yml (Recipe Management App)
version: '3.8'
services:
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: recipe_db
      POSTGRES_USER: recipe_user
      POSTGRES_PASSWORD: recipe_pass
    ports:
      - "5432:5432"
    volumes:
      - recipe_pg_data:/var/lib/postgresql/data

  recipe-service:
    build: .
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/recipe_db
      SPRING_DATASOURCE_USERNAME: recipe_user
      SPRING_DATASOURCE_PASSWORD: recipe_pass
    ports:
      - "8080:8080"
    depends_on:
      - postgres

volumes:
  recipe_pg_data:
```

---

## โปรเจค 58: Meal Planner & Nutrition Tracker

### ภาพรวมระบบ

Nutrition Tracker ช่วยให้ผู้ใช้บันทึกอาหารที่ทานประจำวัน ติดตามแคลอรีและสารอาหาร บันทึกการดื่มน้ำ น้ำหนัก และรายงานความก้าวหน้าตามเป้าหมายสุขภาพ

### Flyway Migration

```sql
-- V1__create_nutrition_tracker_tables.sql
CREATE TABLE user_goals (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL UNIQUE,
    daily_calorie_target INT NOT NULL DEFAULT 2000,
    protein_target_g DECIMAL(8,2),
    carbs_target_g DECIMAL(8,2),
    fat_target_g DECIMAL(8,2),
    water_target_ml INT DEFAULT 2000,
    weight_goal_kg DECIMAL(5,2),
    activity_level VARCHAR(20) DEFAULT 'MODERATE',
    goal_type VARCHAR(20) DEFAULT 'MAINTAIN',
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE food_diary_entries (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL,
    diary_date DATE NOT NULL,
    meal_type VARCHAR(20) NOT NULL,
    food_name VARCHAR(300) NOT NULL,
    serving_size DECIMAL(10,2) NOT NULL,
    serving_unit VARCHAR(50) NOT NULL,
    calories DECIMAL(8,2) NOT NULL,
    protein_g DECIMAL(8,2),
    carbs_g DECIMAL(8,2),
    fat_g DECIMAL(8,2),
    fiber_g DECIMAL(8,2),
    sodium_mg DECIMAL(10,2),
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE water_logs (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL,
    log_date DATE NOT NULL,
    amount_ml INT NOT NULL,
    logged_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE weight_logs (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL,
    weight_kg DECIMAL(5,2) NOT NULL,
    body_fat_percentage DECIMAL(5,2),
    notes TEXT,
    logged_at DATE NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_food_diary_user_date ON food_diary_entries(user_id, diary_date);
CREATE INDEX idx_water_logs_user_date ON water_logs(user_id, log_date);
CREATE INDEX idx_weight_logs_user ON weight_logs(user_id, logged_at DESC);
```

### Entity & Service

```java
// FoodDiaryEntry.java
package com.nutrition.entity;

import jakarta.persistence.*;
import lombok.*;
import java.math.BigDecimal;
import java.time.LocalDate;
import java.time.LocalDateTime;

@Entity
@Table(name = "food_diary_entries")
@Getter @Setter @Builder
@NoArgsConstructor @AllArgsConstructor
public class FoodDiaryEntry {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "user_id", nullable = false)
    private Long userId;

    @Column(name = "diary_date", nullable = false)
    private LocalDate diaryDate;

    @Column(name = "meal_type", nullable = false)
    private String mealType; // BREAKFAST, LUNCH, DINNER, SNACK

    @Column(name = "food_name", nullable = false)
    private String foodName;

    @Column(name = "serving_size", nullable = false)
    private BigDecimal servingSize;

    @Column(name = "serving_unit", nullable = false)
    private String servingUnit;

    @Column(name = "calories", nullable = false)
    private BigDecimal calories;

    @Column(name = "protein_g")
    private BigDecimal proteinG;

    @Column(name = "carbs_g")
    private BigDecimal carbsG;

    @Column(name = "fat_g")
    private BigDecimal fatG;

    @Column(name = "created_at")
    private LocalDateTime createdAt = LocalDateTime.now();
}

// NutritionTrackerService.java
@Service
@RequiredArgsConstructor
public class NutritionTrackerService {

    private final FoodDiaryRepository diaryRepository;
    private final WaterLogRepository waterLogRepository;
    private final WeightLogRepository weightLogRepository;
    private final UserGoalRepository goalRepository;

    @Transactional
    public FoodDiaryEntry logFood(Long userId, LocalDate date, String mealType, String foodName,
                                   BigDecimal servingSize, String servingUnit, BigDecimal calories,
                                   BigDecimal protein, BigDecimal carbs, BigDecimal fat) {
        FoodDiaryEntry entry = FoodDiaryEntry.builder()
                .userId(userId).diaryDate(date).mealType(mealType)
                .foodName(foodName).servingSize(servingSize).servingUnit(servingUnit)
                .calories(calories).proteinG(protein).carbsG(carbs).fatG(fat).build();
        return diaryRepository.save(entry);
    }

    @Transactional
    public WaterLog logWater(Long userId, int amountMl) {
        WaterLog log = WaterLog.builder()
                .userId(userId).logDate(LocalDate.now()).amountMl(amountMl).build();
        return waterLogRepository.save(log);
    }

    @Transactional
    public WeightLog logWeight(Long userId, BigDecimal weightKg, BigDecimal bodyFatPercent,
                                String notes) {
        WeightLog log = WeightLog.builder()
                .userId(userId).weightKg(weightKg)
                .bodyFatPercentage(bodyFatPercent)
                .notes(notes).loggedAt(LocalDate.now()).build();
        return weightLogRepository.save(log);
    }

    public DailySummary getDailySummary(Long userId, LocalDate date) {
        List<FoodDiaryEntry> entries = diaryRepository.findByUserIdAndDiaryDate(userId, date);
        UserGoal goals = goalRepository.findByUserId(userId).orElse(null);

        BigDecimal totalCal = entries.stream().map(FoodDiaryEntry::getCalories)
                .reduce(BigDecimal.ZERO, BigDecimal::add);
        BigDecimal totalProt = entries.stream().map(e -> e.getProteinG() != null ? e.getProteinG() : BigDecimal.ZERO)
                .reduce(BigDecimal.ZERO, BigDecimal::add);
        BigDecimal totalCarbs = entries.stream().map(e -> e.getCarbsG() != null ? e.getCarbsG() : BigDecimal.ZERO)
                .reduce(BigDecimal.ZERO, BigDecimal::add);
        BigDecimal totalFat = entries.stream().map(e -> e.getFatG() != null ? e.getFatG() : BigDecimal.ZERO)
                .reduce(BigDecimal.ZERO, BigDecimal::add);

        int totalWater = waterLogRepository.findByUserIdAndLogDate(userId, date).stream()
                .mapToInt(WaterLog::getAmountMl).sum();

        return new DailySummary(date, entries, totalCal, totalProt, totalCarbs, totalFat,
            totalWater, goals);
    }

    public WeeklyReport getWeeklyReport(Long userId, LocalDate weekStart) {
        LocalDate weekEnd = weekStart.plusDays(6);
        List<FoodDiaryEntry> weekEntries = diaryRepository
                .findByUserIdAndDiaryDateBetween(userId, weekStart, weekEnd);
        List<WeightLog> weightLogs = weightLogRepository
                .findByUserIdAndLoggedAtBetween(userId, weekStart, weekEnd);

        BigDecimal avgCal = weekEntries.stream().map(FoodDiaryEntry::getCalories)
                .reduce(BigDecimal.ZERO, BigDecimal::add)
                .divide(BigDecimal.valueOf(7), 2, java.math.RoundingMode.HALF_UP);

        return new WeeklyReport(weekStart, weekEnd, avgCal, weekEntries.size(), weightLogs);
    }

    public record DailySummary(LocalDate date, List<FoodDiaryEntry> entries,
        BigDecimal totalCalories, BigDecimal totalProtein, BigDecimal totalCarbs,
        BigDecimal totalFat, int waterMl, UserGoal goals) {}
    public record WeeklyReport(LocalDate from, LocalDate to, BigDecimal avgCalories,
        int totalFoodEntries, List<WeightLog> weightLogs) {}
}
```

### Controller & docker-compose.yml

```java
// NutritionController.java
@RestController
@RequestMapping("/api/v1/nutrition")
@RequiredArgsConstructor
public class NutritionController {

    private final NutritionTrackerService service;

    @PostMapping("/diary")
    public ResponseEntity<FoodDiaryEntry> logFood(@RequestBody LogFoodRequest request) {
        return ResponseEntity.ok(service.logFood(request.userId(), request.date(),
            request.mealType(), request.foodName(), request.servingSize(), request.servingUnit(),
            request.calories(), request.protein(), request.carbs(), request.fat()));
    }

    @PostMapping("/water")
    public ResponseEntity<WaterLog> logWater(@RequestParam Long userId,
            @RequestParam int amountMl) {
        return ResponseEntity.ok(service.logWater(userId, amountMl));
    }

    @PostMapping("/weight")
    public ResponseEntity<WeightLog> logWeight(@RequestBody LogWeightRequest request) {
        return ResponseEntity.ok(service.logWeight(request.userId(), request.weightKg(),
            request.bodyFatPercent(), request.notes()));
    }

    @GetMapping("/summary/{userId}")
    public ResponseEntity<NutritionTrackerService.DailySummary> getDailySummary(
            @PathVariable Long userId,
            @RequestParam(required = false) java.time.LocalDate date) {
        return ResponseEntity.ok(service.getDailySummary(userId,
            date != null ? date : java.time.LocalDate.now()));
    }

    @GetMapping("/weekly/{userId}")
    public ResponseEntity<NutritionTrackerService.WeeklyReport> getWeeklyReport(
            @PathVariable Long userId, @RequestParam java.time.LocalDate weekStart) {
        return ResponseEntity.ok(service.getWeeklyReport(userId, weekStart));
    }

    record LogFoodRequest(Long userId, java.time.LocalDate date, String mealType,
        String foodName, java.math.BigDecimal servingSize, String servingUnit,
        java.math.BigDecimal calories, java.math.BigDecimal protein,
        java.math.BigDecimal carbs, java.math.BigDecimal fat) {}
    record LogWeightRequest(Long userId, java.math.BigDecimal weightKg,
        java.math.BigDecimal bodyFatPercent, String notes) {}
}
```

```yaml
# docker-compose.yml (Nutrition Tracker)
version: '3.8'
services:
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: nutrition_db
      POSTGRES_USER: nutrition_user
      POSTGRES_PASSWORD: nutrition_pass
    ports:
      - "5432:5432"
    volumes:
      - nutrition_pg_data:/var/lib/postgresql/data

  nutrition-service:
    build: .
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/nutrition_db
      SPRING_DATASOURCE_USERNAME: nutrition_user
      SPRING_DATASOURCE_PASSWORD: nutrition_pass
    ports:
      - "8080:8080"
    depends_on:
      - postgres

volumes:
  nutrition_pg_data:
```

---

## โปรเจค 59: Pharmacy Management System

### ภาพรวมระบบ

ระบบบริหารร้านขายยาครบวงจร จัดการสต็อกยา ใบสั่งยา การจ่ายยา ติดตามวันหมดอายุ สั่งซื้อจากซัพพลายเออร์ และบันทึกยาควบคุม

### Flyway Migration

```sql
-- V1__create_pharmacy_tables.sql
CREATE TABLE medicines (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(300) NOT NULL,
    generic_name VARCHAR(300),
    brand VARCHAR(200),
    category VARCHAR(100),
    dosage_form VARCHAR(100),
    strength VARCHAR(100),
    barcode VARCHAR(100) UNIQUE,
    requires_prescription BOOLEAN DEFAULT FALSE,
    is_controlled_substance BOOLEAN DEFAULT FALSE,
    unit_price DECIMAL(10,2) NOT NULL,
    reorder_level INT DEFAULT 10,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE medicine_stock (
    id BIGSERIAL PRIMARY KEY,
    medicine_id BIGINT NOT NULL REFERENCES medicines(id),
    batch_number VARCHAR(100) NOT NULL,
    quantity INT NOT NULL DEFAULT 0,
    expiry_date DATE NOT NULL,
    supplier_id BIGINT,
    purchase_price DECIMAL(10,2),
    received_at TIMESTAMP NOT NULL DEFAULT NOW(),
    UNIQUE (medicine_id, batch_number)
);

CREATE TABLE prescriptions (
    id BIGSERIAL PRIMARY KEY,
    patient_id BIGINT NOT NULL,
    doctor_id BIGINT,
    doctor_name VARCHAR(200),
    doctor_license VARCHAR(100),
    prescription_date DATE NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'PENDING',
    notes TEXT,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE prescription_items (
    id BIGSERIAL PRIMARY KEY,
    prescription_id BIGINT NOT NULL REFERENCES prescriptions(id),
    medicine_id BIGINT NOT NULL REFERENCES medicines(id),
    quantity INT NOT NULL,
    dosage_instructions TEXT,
    duration_days INT,
    dispensed_quantity INT DEFAULT 0
);

CREATE TABLE dispensing_records (
    id BIGSERIAL PRIMARY KEY,
    prescription_id BIGINT REFERENCES prescriptions(id),
    medicine_id BIGINT NOT NULL REFERENCES medicines(id),
    patient_id BIGINT NOT NULL,
    pharmacist_id BIGINT NOT NULL,
    quantity_dispensed INT NOT NULL,
    batch_number VARCHAR(100),
    total_amount DECIMAL(10,2) NOT NULL,
    dispensed_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE controlled_substance_logs (
    id BIGSERIAL PRIMARY KEY,
    medicine_id BIGINT NOT NULL REFERENCES medicines(id),
    action VARCHAR(20) NOT NULL,
    quantity INT NOT NULL,
    reference_id BIGINT,
    pharmacist_id BIGINT NOT NULL,
    notes TEXT,
    logged_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_medicine_stock_expiry ON medicine_stock(expiry_date);
CREATE INDEX idx_dispensing_patient ON dispensing_records(patient_id);
```

### Entity & Service

```java
// PharmacyService.java
package com.pharmacy.service;

import lombok.RequiredArgsConstructor;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import java.math.BigDecimal;
import java.time.LocalDate;
import java.time.LocalDateTime;
import java.util.List;

@Service
@RequiredArgsConstructor
public class PharmacyService {

    private final MedicineRepository medicineRepository;
    private final MedicineStockRepository stockRepository;
    private final PrescriptionRepository prescriptionRepository;
    private final DispensingRecordRepository dispensingRepository;
    private final ControlledSubstanceLogRepository controlledLogRepository;
    private final NotificationService notificationService;

    @Transactional
    public MedicineStock receiveStock(Long medicineId, String batchNumber, int quantity,
                                       LocalDate expiryDate, BigDecimal purchasePrice) {
        MedicineStock stock = stockRepository.findByMedicineIdAndBatchNumber(medicineId, batchNumber)
                .orElse(MedicineStock.builder()
                    .medicineId(medicineId).batchNumber(batchNumber)
                    .quantity(0).expiryDate(expiryDate).purchasePrice(purchasePrice).build());
        stock.setQuantity(stock.getQuantity() + quantity);
        return stockRepository.save(stock);
    }

    @Transactional
    public DispensingRecord dispense(Long prescriptionId, Long medicineId, Long patientId,
                                      Long pharmacistId, int quantity) {
        Medicine medicine = medicineRepository.findById(medicineId)
                .orElseThrow(() -> new RuntimeException("Medicine not found"));

        // Check total available stock
        int totalStock = stockRepository.findByMedicineIdOrderByExpiryDate(medicineId)
                .stream().mapToInt(MedicineStock::getQuantity).sum();
        if (totalStock < quantity) {
            throw new IllegalStateException("Insufficient stock. Available: " + totalStock);
        }

        // Deduct from oldest batch (FEFO - First Expired First Out)
        int remaining = quantity;
        String usedBatch = null;
        for (MedicineStock batch : stockRepository.findByMedicineIdOrderByExpiryDate(medicineId)) {
            if (remaining <= 0) break;
            int deduct = Math.min(remaining, batch.getQuantity());
            batch.setQuantity(batch.getQuantity() - deduct);
            stockRepository.save(batch);
            remaining -= deduct;
            if (usedBatch == null) usedBatch = batch.getBatchNumber();
        }

        BigDecimal totalAmount = medicine.getUnitPrice().multiply(BigDecimal.valueOf(quantity));
        DispensingRecord record = DispensingRecord.builder()
                .prescriptionId(prescriptionId).medicineId(medicineId)
                .patientId(patientId).pharmacistId(pharmacistId)
                .quantityDispensed(quantity).batchNumber(usedBatch)
                .totalAmount(totalAmount).build();
        dispensingRepository.save(record);

        // Log controlled substance
        if (medicine.getIsControlledSubstance()) {
            ControlledSubstanceLog log = ControlledSubstanceLog.builder()
                    .medicineId(medicineId).action("DISPENSE")
                    .quantity(quantity).referenceId(record.getId())
                    .pharmacistId(pharmacistId)
                    .notes("Dispensed to patient " + patientId).build();
            controlledLogRepository.save(log);
        }

        return record;
    }

    @Scheduled(cron = "0 0 8 * * *") // Run every day at 8 AM
    public void checkExpiringMedicines() {
        LocalDate thirtyDaysFromNow = LocalDate.now().plusDays(30);
        List<MedicineStock> expiring = stockRepository
                .findByExpiryDateBeforeAndQuantityGreaterThan(thirtyDaysFromNow, 0);
        if (!expiring.isEmpty()) {
            notificationService.notifyExpiringMedicines(expiring);
        }
    }

    @Scheduled(cron = "0 0 9 * * *")
    public void checkLowStock() {
        List<Medicine> allMedicines = medicineRepository.findAll();
        for (Medicine med : allMedicines) {
            int totalStock = stockRepository.findByMedicineId(med.getId())
                    .stream().mapToInt(MedicineStock::getQuantity).sum();
            if (totalStock <= med.getReorderLevel()) {
                notificationService.notifyLowStock(med, totalStock);
            }
        }
    }

    public List<MedicineStock> getExpiringStock(int daysAhead) {
        return stockRepository.findByExpiryDateBeforeAndQuantityGreaterThan(
            LocalDate.now().plusDays(daysAhead), 0);
    }
}
```

### Controller

```java
// PharmacyController.java
@RestController
@RequestMapping("/api/v1/pharmacy")
@RequiredArgsConstructor
public class PharmacyController {

    private final PharmacyService pharmacyService;

    @PostMapping("/stock/receive")
    public ResponseEntity<MedicineStock> receiveStock(@RequestBody ReceiveStockRequest request) {
        return ResponseEntity.ok(pharmacyService.receiveStock(request.medicineId(),
            request.batchNumber(), request.quantity(), request.expiryDate(), request.purchasePrice()));
    }

    @PostMapping("/dispense")
    public ResponseEntity<DispensingRecord> dispense(@RequestBody DispenseRequest request) {
        return ResponseEntity.ok(pharmacyService.dispense(request.prescriptionId(),
            request.medicineId(), request.patientId(), request.pharmacistId(), request.quantity()));
    }

    @GetMapping("/stock/expiring")
    public ResponseEntity<List<MedicineStock>> getExpiring(
            @RequestParam(defaultValue = "30") int daysAhead) {
        return ResponseEntity.ok(pharmacyService.getExpiringStock(daysAhead));
    }

    record ReceiveStockRequest(Long medicineId, String batchNumber, int quantity,
        java.time.LocalDate expiryDate, BigDecimal purchasePrice) {}
    record DispenseRequest(Long prescriptionId, Long medicineId, Long patientId,
        Long pharmacistId, int quantity) {}
}
```

### docker-compose.yml

```yaml
# docker-compose.yml (Pharmacy Management)
version: '3.8'
services:
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: pharmacy_db
      POSTGRES_USER: pharmacy_user
      POSTGRES_PASSWORD: pharmacy_pass
    ports:
      - "5432:5432"
    volumes:
      - pharmacy_pg_data:/var/lib/postgresql/data

  pharmacy-service:
    build: .
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/pharmacy_db
      SPRING_DATASOURCE_USERNAME: pharmacy_user
      SPRING_DATASOURCE_PASSWORD: pharmacy_pass
    ports:
      - "8080:8080"
    depends_on:
      - postgres

volumes:
  pharmacy_pg_data:
```

---

## โปรเจค 60: Blood Bank Management System

### ภาพรวมระบบ

ระบบบริหารจัดการธนาคารเลือดแบบครบวงจร จัดการข้อมูลผู้บริจาคเลือด การบริจาค สต็อกเลือดตามกรุ๊ป การรับ Request เลือดจากโรงพยาบาล การจับคู่ความเข้ากันได้ ติดตามวันหมดอายุ และแจ้งเตือนผู้บริจาคเมื่อต้องการเลือด

### Flyway Migration

```sql
-- V1__create_blood_bank_tables.sql
CREATE TABLE blood_donors (
    id BIGSERIAL PRIMARY KEY,
    donor_number VARCHAR(20) NOT NULL UNIQUE,
    first_name VARCHAR(100) NOT NULL,
    last_name VARCHAR(100) NOT NULL,
    blood_type VARCHAR(5) NOT NULL,
    date_of_birth DATE NOT NULL,
    gender VARCHAR(10),
    phone VARCHAR(20) NOT NULL,
    email VARCHAR(255),
    address TEXT,
    weight_kg DECIMAL(5,2),
    is_eligible BOOLEAN DEFAULT TRUE,
    last_donation_date DATE,
    total_donations INT DEFAULT 0,
    registered_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE blood_donations (
    id BIGSERIAL PRIMARY KEY,
    donor_id BIGINT NOT NULL REFERENCES blood_donors(id),
    donation_number VARCHAR(30) NOT NULL UNIQUE,
    blood_type VARCHAR(5) NOT NULL,
    volume_ml INT NOT NULL DEFAULT 450,
    donation_date DATE NOT NULL,
    staff_id BIGINT,
    status VARCHAR(20) NOT NULL DEFAULT 'COLLECTED',
    test_result VARCHAR(20),
    notes TEXT,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE blood_inventory (
    id BIGSERIAL PRIMARY KEY,
    donation_id BIGINT NOT NULL UNIQUE REFERENCES blood_donations(id),
    blood_type VARCHAR(5) NOT NULL,
    component_type VARCHAR(50) NOT NULL DEFAULT 'WHOLE_BLOOD',
    volume_ml INT NOT NULL,
    expiry_date DATE NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'AVAILABLE',
    location VARCHAR(100),
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE blood_requests (
    id BIGSERIAL PRIMARY KEY,
    request_number VARCHAR(30) NOT NULL UNIQUE,
    hospital_id BIGINT NOT NULL,
    hospital_name VARCHAR(300) NOT NULL,
    patient_name VARCHAR(200) NOT NULL,
    blood_type VARCHAR(5) NOT NULL,
    component_type VARCHAR(50) DEFAULT 'WHOLE_BLOOD',
    quantity_units INT NOT NULL,
    urgency VARCHAR(20) NOT NULL DEFAULT 'ROUTINE',
    status VARCHAR(20) NOT NULL DEFAULT 'PENDING',
    required_by TIMESTAMP,
    fulfilled_at TIMESTAMP,
    notes TEXT,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE blood_dispensing (
    id BIGSERIAL PRIMARY KEY,
    request_id BIGINT NOT NULL REFERENCES blood_requests(id),
    inventory_id BIGINT NOT NULL REFERENCES blood_inventory(id),
    dispensed_by BIGINT NOT NULL,
    dispensed_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_blood_inventory_type_status ON blood_inventory(blood_type, status);
CREATE INDEX idx_blood_inventory_expiry ON blood_inventory(expiry_date);
CREATE INDEX idx_blood_requests_urgency ON blood_requests(urgency, status);
```

### Entity & Service

```java
// BloodBankService.java
package com.bloodbank.service;

import lombok.RequiredArgsConstructor;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import java.time.LocalDate;
import java.time.LocalDateTime;
import java.util.*;

@Service
@RequiredArgsConstructor
public class BloodBankService {

    private final BloodDonorRepository donorRepository;
    private final BloodDonationRepository donationRepository;
    private final BloodInventoryRepository inventoryRepository;
    private final BloodRequestRepository requestRepository;
    private final BloodDispensingRepository dispensingRepository;
    private final NotificationService notificationService;

    // Blood type compatibility map
    private static final Map<String, List<String>> COMPATIBILITY = Map.of(
        "A+",  List.of("A+", "AB+"),
        "A-",  List.of("A+", "A-", "AB+", "AB-"),
        "B+",  List.of("B+", "AB+"),
        "B-",  List.of("B+", "B-", "AB+", "AB-"),
        "AB+", List.of("AB+"),
        "AB-", List.of("AB+", "AB-"),
        "O+",  List.of("A+", "B+", "AB+", "O+"),
        "O-",  List.of("A+", "A-", "B+", "B-", "AB+", "AB-", "O+", "O-")
    );

    @Transactional
    public BloodDonor registerDonor(String firstName, String lastName, String bloodType,
                                     LocalDate dob, String phone, String email) {
        String donorNumber = "DN-" + System.currentTimeMillis();
        BloodDonor donor = BloodDonor.builder()
                .donorNumber(donorNumber.substring(0, Math.min(20, donorNumber.length())))
                .firstName(firstName).lastName(lastName).bloodType(bloodType)
                .dateOfBirth(dob).phone(phone).email(email).isEligible(true).build();
        return donorRepository.save(donor);
    }

    @Transactional
    public BloodDonation recordDonation(Long donorId, int volumeMl, Long staffId) {
        BloodDonor donor = donorRepository.findById(donorId)
                .orElseThrow(() -> new RuntimeException("Donor not found"));
        if (!donor.getIsEligible()) {
            throw new IllegalStateException("Donor is not eligible to donate");
        }
        // Check 56-day rule
        if (donor.getLastDonationDate() != null &&
            donor.getLastDonationDate().plusDays(56).isAfter(LocalDate.now())) {
            throw new IllegalStateException("Donor must wait 56 days between donations");
        }

        String donationNumber = "DON-" + System.currentTimeMillis();
        BloodDonation donation = BloodDonation.builder()
                .donorId(donorId).donationNumber(donationNumber)
                .bloodType(donor.getBloodType()).volumeMl(volumeMl)
                .donationDate(LocalDate.now()).staffId(staffId)
                .status("COLLECTED").build();
        donation = donationRepository.save(donation);

        donor.setLastDonationDate(LocalDate.now());
        donor.setTotalDonations(donor.getTotalDonations() + 1);
        donorRepository.save(donor);

        // Add to inventory
        BloodInventory inventory = BloodInventory.builder()
                .donationId(donation.getId()).bloodType(donor.getBloodType())
                .componentType("WHOLE_BLOOD").volumeMl(volumeMl)
                .expiryDate(LocalDate.now().plusDays(35))
                .status("PENDING_TESTING").build();
        inventoryRepository.save(inventory);

        return donation;
    }

    @Transactional
    public List<BloodInventory> fulfillRequest(Long requestId, Long dispensedBy) {
        BloodRequest request = requestRepository.findById(requestId)
                .orElseThrow(() -> new RuntimeException("Request not found"));

        List<String> compatibleTypes = COMPATIBILITY.getOrDefault(request.getBloodType(),
            List.of(request.getBloodType()));

        List<BloodInventory> available = inventoryRepository
                .findByBloodTypeInAndComponentTypeAndStatusOrderByExpiryDate(
                    compatibleTypes, request.getComponentType(), "AVAILABLE");

        if (available.size() < request.getQuantityUnits()) {
            throw new IllegalStateException("Insufficient blood units. Available: " + available.size()
                + " Required: " + request.getQuantityUnits());
        }

        List<BloodInventory> dispensed = new ArrayList<>();
        for (int i = 0; i < request.getQuantityUnits(); i++) {
            BloodInventory unit = available.get(i);
            unit.setStatus("DISPENSED");
            unit.setUpdatedAt(LocalDateTime.now());
            inventoryRepository.save(unit);

            BloodDispensing disp = BloodDispensing.builder()
                    .requestId(requestId).inventoryId(unit.getId())
                    .dispensedBy(dispensedBy).build();
            dispensingRepository.save(disp);
            dispensed.add(unit);
        }

        request.setStatus("FULFILLED");
        request.setFulfilledAt(LocalDateTime.now());
        requestRepository.save(request);

        return dispensed;
    }

    public Map<String, Integer> getInventorySummary() {
        Map<String, Integer> summary = new LinkedHashMap<>();
        for (String bt : List.of("A+","A-","B+","B-","AB+","AB-","O+","O-")) {
            int count = inventoryRepository.countByBloodTypeAndStatus(bt, "AVAILABLE");
            summary.put(bt, count);
        }
        return summary;
    }

    @Scheduled(cron = "0 0 7 * * *")
    public void notifyExpiringBlood() {
        LocalDate threeDays = LocalDate.now().plusDays(3);
        List<BloodInventory> expiring = inventoryRepository
                .findByExpiryDateBeforeAndStatus(threeDays, "AVAILABLE");
        if (!expiring.isEmpty()) {
            notificationService.notifyBloodExpiring(expiring);
        }
    }

    @Transactional
    public void notifyDonorsForBloodType(String bloodType) {
        List<BloodDonor> eligibleDonors = donorRepository
                .findByBloodTypeAndIsEligibleAndLastDonationDateBefore(
                    bloodType, true, LocalDate.now().minusDays(56));
        for (BloodDonor donor : eligibleDonors) {
            if (donor.getEmail() != null) {
                notificationService.sendDonationRequest(donor);
            }
        }
    }
}
```

### Controller

```java
// BloodBankController.java
@RestController
@RequestMapping("/api/v1/blood-bank")
@RequiredArgsConstructor
public class BloodBankController {

    private final BloodBankService service;

    @PostMapping("/donors")
    public ResponseEntity<BloodDonor> registerDonor(@RequestBody RegisterDonorRequest request) {
        return ResponseEntity.ok(service.registerDonor(request.firstName(), request.lastName(),
            request.bloodType(), request.dateOfBirth(), request.phone(), request.email()));
    }

    @PostMapping("/donations")
    public ResponseEntity<BloodDonation> recordDonation(@RequestBody RecordDonationRequest request) {
        return ResponseEntity.ok(service.recordDonation(
            request.donorId(), request.volumeMl(), request.staffId()));
    }

    @PostMapping("/requests/{requestId}/fulfill")
    public ResponseEntity<List<BloodInventory>> fulfill(@PathVariable Long requestId,
            @RequestParam Long dispensedBy) {
        return ResponseEntity.ok(service.fulfillRequest(requestId, dispensedBy));
    }

    @GetMapping("/inventory/summary")
    public ResponseEntity<Map<String, Integer>> getInventorySummary() {
        return ResponseEntity.ok(service.getInventorySummary());
    }

    @PostMapping("/notify/{bloodType}")
    public ResponseEntity<Void> notifyDonors(@PathVariable String bloodType) {
        service.notifyDonorsForBloodType(bloodType);
        return ResponseEntity.ok().build();
    }

    record RegisterDonorRequest(String firstName, String lastName, String bloodType,
        java.time.LocalDate dateOfBirth, String phone, String email) {}
    record RecordDonationRequest(Long donorId, int volumeMl, Long staffId) {}
}
```

### docker-compose.yml

```yaml
# docker-compose.yml (Blood Bank Management)
version: '3.8'
services:
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: bloodbank_db
      POSTGRES_USER: bloodbank_user
      POSTGRES_PASSWORD: bloodbank_pass
    ports:
      - "5432:5432"
    volumes:
      - bloodbank_pg_data:/var/lib/postgresql/data

  mailhog:
    image: mailhog/mailhog
    ports:
      - "1025:1025"
      - "8025:8025"

  bloodbank-service:
    build: .
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/bloodbank_db
      SPRING_DATASOURCE_USERNAME: bloodbank_user
      SPRING_DATASOURCE_PASSWORD: bloodbank_pass
      SPRING_MAIL_HOST: mailhog
      SPRING_MAIL_PORT: 1025
    ports:
      - "8080:8080"
    depends_on:
      - postgres
      - mailhog

volumes:
  bloodbank_pg_data:
```

---

*[← Part 111: Loyalty & Marketing](./part-111-loyalty-marketing.md) | [Part 113: Fitness & Wellness →](./part-113-fitness-wellness.md)*

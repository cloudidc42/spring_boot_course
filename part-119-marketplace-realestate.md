# Part 119: โปรเจค 91-95 — Marketplace & Real Estate

> **ระดับ:** โลก (World-Class) | **เวลาเรียนรู้:** 10-15 ชั่วโมง  
> **เป้าหมาย:** สร้างระบบ Marketplace & Real Estate Platforms ที่ใช้งานได้จริงในระดับ Production

---

## โปรเจค 91: Multi-vendor Marketplace

### ภาพรวมโปรเจค

Multi-vendor Marketplace เป็นแพลตฟอร์มตลาดออนไลน์ที่รองรับหลาย Vendor สามารถขายสินค้าได้พร้อมกัน ระบบรองรับการจัดการ Vendors, Products ของแต่ละ Vendor, คำสั่งซื้อที่แบ่งตาม Vendor, การจ่ายเงินให้ Vendor (Payouts), การบริหาร Commission, Vendor Dashboard และ Store Pages

### Entities

```java
// Vendor.java — ร้านค้าใน Marketplace
@Entity
@Table(name = "vendors")
public class Vendor {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @OneToOne
    @JoinColumn(name = "user_id")
    private User user;

    @Column(unique = true, nullable = false)
    private String storeName;

    @Column(unique = true)
    private String storeSlug;

    private String description;
    private String logoUrl;
    private String bannerUrl;
    private String email;
    private String phone;
    private String address;

    // Payment info for payouts
    private String bankAccount;
    private String bankName;
    private String taxId;

    @Enumerated(EnumType.STRING)
    private VendorStatus status; // PENDING, ACTIVE, SUSPENDED

    private Float commissionRate; // % ที่ Marketplace หัก
    private Float averageRating;
    private Long totalOrders;
    private BigDecimal totalRevenue;
    private BigDecimal pendingPayout;

    private LocalDateTime createdAt;
    private LocalDateTime verifiedAt;
}

// VendorProduct.java — สินค้าของ Vendor
@Entity
@Table(name = "vendor_products")
public class VendorProduct {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne
    @JoinColumn(name = "vendor_id")
    private Vendor vendor;

    @Column(nullable = false)
    private String name;

    private String description;

    @Column(nullable = false)
    private BigDecimal price;

    private BigDecimal compareAtPrice;
    private Integer stockQuantity;

    @ElementCollection
    @CollectionTable(name = "vendor_product_images")
    private List<String> imageUrls = new ArrayList<>();

    private String category;

    @Enumerated(EnumType.STRING)
    private ProductStatus status; // DRAFT, ACTIVE, SOLD_OUT, ARCHIVED

    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
}

// MarketplaceOrder.java — คำสั่งซื้อหลัก (รวมทุก Vendor)
@Entity
@Table(name = "marketplace_orders")
public class MarketplaceOrder {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(unique = true, nullable = false)
    private String orderNumber;

    @ManyToOne
    @JoinColumn(name = "customer_id")
    private User customer;

    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL)
    private List<VendorOrder> vendorOrders = new ArrayList<>();

    private BigDecimal totalAmount;
    private BigDecimal shippingAmount;
    private String currency;

    // Shipping address
    private String shippingName;
    private String shippingAddress;
    private String shippingCity;
    private String shippingPostalCode;
    private String shippingCountry;

    @Enumerated(EnumType.STRING)
    private OrderStatus status; // PENDING_PAYMENT, PAID, PROCESSING, SHIPPED, DELIVERED, CANCELLED

    // Payment
    private String stripePaymentIntentId;
    private LocalDateTime paidAt;

    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
}

// VendorOrder.java — ส่วนของ Order ที่เป็นของ Vendor นั้นๆ
@Entity
@Table(name = "vendor_orders")
public class VendorOrder {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne
    @JoinColumn(name = "order_id")
    private MarketplaceOrder order;

    @ManyToOne
    @JoinColumn(name = "vendor_id")
    private Vendor vendor;

    @OneToMany(mappedBy = "vendorOrder", cascade = CascadeType.ALL)
    private List<VendorOrderItem> items = new ArrayList<>();

    private BigDecimal subtotal;
    private BigDecimal commissionAmount;
    private BigDecimal vendorAmount; // subtotal - commission

    @Enumerated(EnumType.STRING)
    private VendorOrderStatus status; // PENDING, CONFIRMED, SHIPPED, DELIVERED, CANCELLED

    private String trackingNumber;
    private String shippingCarrier;

    @Enumerated(EnumType.STRING)
    private PayoutStatus payoutStatus; // PENDING, SCHEDULED, PAID

    private LocalDateTime payoutScheduledAt;
    private LocalDateTime paidOutAt;
    private String payoutReference;
}

// VendorPayout.java — บันทึกการจ่ายเงินให้ Vendor
@Entity
@Table(name = "vendor_payouts")
public class VendorPayout {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne
    @JoinColumn(name = "vendor_id")
    private Vendor vendor;

    private BigDecimal amount;
    private String currency;

    @Enumerated(EnumType.STRING)
    private PayoutStatus status; // SCHEDULED, PROCESSING, COMPLETED, FAILED

    private String bankAccount;
    private String transactionReference;

    private LocalDateTime scheduledAt;
    private LocalDateTime processedAt;
    private String notes;
}
```

### Service Layer

```java
// MarketplaceOrderService.java — จัดการคำสั่งซื้อ
@Service
@Transactional
public class MarketplaceOrderService {

    @Autowired
    private MarketplaceOrderRepository orderRepository;

    @Autowired
    private VendorOrderRepository vendorOrderRepository;

    @Autowired
    private VendorRepository vendorRepository;

    @Autowired
    private PaymentService paymentService;

    // สร้างคำสั่งซื้อ — แยกตาม Vendor อัตโนมัติ
    public MarketplaceOrder createOrder(CreateOrderRequest request, Long customerId) {
        MarketplaceOrder order = new MarketplaceOrder();
        order.setOrderNumber(generateOrderNumber());
        order.setCustomer(new User(customerId));
        order.setStatus(OrderStatus.PENDING_PAYMENT);
        order.setCreatedAt(LocalDateTime.now());

        // Copy shipping info
        order.setShippingName(request.getShippingName());
        order.setShippingAddress(request.getShippingAddress());
        order.setShippingCity(request.getShippingCity());
        order.setShippingPostalCode(request.getShippingPostalCode());
        order.setShippingCountry(request.getShippingCountry());

        // Group items by vendor
        Map<Long, List<OrderItemRequest>> itemsByVendor = request.getItems().stream()
                .collect(Collectors.groupingBy(OrderItemRequest::getVendorId));

        BigDecimal totalAmount = BigDecimal.ZERO;

        for (Map.Entry<Long, List<OrderItemRequest>> entry : itemsByVendor.entrySet()) {
            Vendor vendor = vendorRepository.findById(entry.getKey()).orElseThrow();

            VendorOrder vendorOrder = new VendorOrder();
            vendorOrder.setOrder(order);
            vendorOrder.setVendor(vendor);
            vendorOrder.setStatus(VendorOrderStatus.PENDING);
            vendorOrder.setPayoutStatus(PayoutStatus.PENDING);

            BigDecimal subtotal = BigDecimal.ZERO;
            List<VendorOrderItem> orderItems = new ArrayList<>();

            for (OrderItemRequest item : entry.getValue()) {
                VendorProduct product = productRepository.findById(item.getProductId()).orElseThrow();
                BigDecimal itemTotal = product.getPrice().multiply(BigDecimal.valueOf(item.getQuantity()));
                subtotal = subtotal.add(itemTotal);

                VendorOrderItem orderItem = new VendorOrderItem();
                orderItem.setVendorOrder(vendorOrder);
                orderItem.setProduct(product);
                orderItem.setQuantity(item.getQuantity());
                orderItem.setUnitPrice(product.getPrice());
                orderItem.setTotal(itemTotal);
                orderItems.add(orderItem);
            }

            vendorOrder.setItems(orderItems);
            vendorOrder.setSubtotal(subtotal);

            // คำนวณ Commission
            BigDecimal commission = subtotal.multiply(
                    BigDecimal.valueOf(vendor.getCommissionRate() / 100));
            vendorOrder.setCommissionAmount(commission);
            vendorOrder.setVendorAmount(subtotal.subtract(commission));

            order.getVendorOrders().add(vendorOrder);
            totalAmount = totalAmount.add(subtotal);
        }

        order.setTotalAmount(totalAmount);
        order = orderRepository.save(order);

        // สร้าง Stripe Payment Intent
        String paymentIntentId = paymentService.createPaymentIntent(
                totalAmount, order.getCurrency(), order.getOrderNumber());
        order.setStripePaymentIntentId(paymentIntentId);

        return orderRepository.save(order);
    }

    // Webhook จาก Stripe: Payment สำเร็จ
    @Transactional
    public void onPaymentSuccess(String paymentIntentId) {
        MarketplaceOrder order = orderRepository.findByStripePaymentIntentId(paymentIntentId)
                .orElseThrow();

        order.setStatus(OrderStatus.PAID);
        order.setPaidAt(LocalDateTime.now());

        // อัปเดต VendorOrders เป็น CONFIRMED
        for (VendorOrder vendorOrder : order.getVendorOrders()) {
            vendorOrder.setStatus(VendorOrderStatus.CONFIRMED);

            // Schedule payout หลังจากส่งสินค้าเรียบร้อย
            vendorOrder.setPayoutScheduledAt(LocalDateTime.now().plusDays(7));
        }

        orderRepository.save(order);

        // แจ้งแต่ละ Vendor
        for (VendorOrder vo : order.getVendorOrders()) {
            emailService.sendNewOrderNotification(vo.getVendor().getEmail(), order.getOrderNumber());
        }
    }
}

// VendorPayoutService.java — จัดการ Payout ให้ Vendor
@Service
public class VendorPayoutService {

    @Autowired
    private VendorOrderRepository vendorOrderRepository;

    @Autowired
    private VendorPayoutRepository payoutRepository;

    // Process Payouts ที่ถึงกำหนด (Scheduled Task)
    @Scheduled(cron = "0 0 10 * * *") // ทุกวัน เวลา 10 โมง
    public void processScheduledPayouts() {
        List<VendorOrder> dueOrders = vendorOrderRepository
                .findByPayoutStatusAndPayoutScheduledAtBefore(
                        PayoutStatus.SCHEDULED, LocalDateTime.now());

        Map<Long, BigDecimal> vendorPayouts = new HashMap<>();

        for (VendorOrder order : dueOrders) {
            vendorPayouts.merge(order.getVendor().getId(),
                    order.getVendorAmount(), BigDecimal::add);
            order.setPayoutStatus(PayoutStatus.PROCESSING);
        }

        vendorOrderRepository.saveAll(dueOrders);

        // สร้าง Payout records และโอนเงิน
        for (Map.Entry<Long, BigDecimal> entry : vendorPayouts.entrySet()) {
            processVendorPayout(entry.getKey(), entry.getValue());
        }
    }

    private void processVendorPayout(Long vendorId, BigDecimal amount) {
        Vendor vendor = vendorRepository.findById(vendorId).orElseThrow();

        VendorPayout payout = new VendorPayout();
        payout.setVendor(vendor);
        payout.setAmount(amount);
        payout.setCurrency("THB");
        payout.setStatus(PayoutStatus.PROCESSING);
        payout.setBankAccount(vendor.getBankAccount());
        payout.setScheduledAt(LocalDateTime.now());
        payoutRepository.save(payout);

        // Transfer via Bank API หรือ Stripe Connect
        // ...
    }
}

// MarketplaceController.java
@RestController
@RequestMapping("/api/marketplace")
public class MarketplaceController {

    @Autowired
    private MarketplaceOrderService orderService;

    @PostMapping("/orders")
    @PreAuthorize("isAuthenticated()")
    public ResponseEntity<OrderResponse> createOrder(
            @RequestBody CreateOrderRequest request,
            @AuthenticationPrincipal User user) {
        MarketplaceOrder order = orderService.createOrder(request, user.getId());
        return ResponseEntity.status(HttpStatus.CREATED).body(OrderResponse.from(order));
    }

    @GetMapping("/vendors/{slug}/store")
    public ResponseEntity<StorePageResponse> getStorePage(@PathVariable String slug) {
        return ResponseEntity.ok(orderService.getStorePage(slug));
    }

    @GetMapping("/vendors/{id}/dashboard")
    @PreAuthorize("hasRole('VENDOR')")
    public ResponseEntity<VendorDashboardResponse> getVendorDashboard(
            @PathVariable Long id,
            @AuthenticationPrincipal User user) {
        return ResponseEntity.ok(orderService.getVendorDashboard(id, user.getId()));
    }
}
```

---

## โปรเจค 92: Real Estate Listing Platform

### ภาพรวมโปรเจค

Real Estate Listing Platform เป็นระบบประกาศซื้อ-ขาย-เช่าอสังหาริมทรัพย์ รองรับ Properties หลายประเภท, การค้นหาด้วย Location/ราคา/จำนวนห้อง, Favorites, Inquiry Form, Agent Profiles และ Featured Listings

### Entities

```java
// Property.java — ข้อมูลอสังหาริมทรัพย์
@Entity
@Table(name = "properties")
public class Property {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String title;

    @Column(columnDefinition = "text")
    private String description;

    @Enumerated(EnumType.STRING)
    private PropertyType propertyType; // HOUSE, CONDO, APARTMENT, TOWNHOUSE, LAND, COMMERCIAL

    @Enumerated(EnumType.STRING)
    private ListingType listingType; // FOR_SALE, FOR_RENT, FOR_SALE_OR_RENT

    @Column(nullable = false)
    private BigDecimal price; // ราคาขาย หรือ ค่าเช่าต่อเดือน

    private BigDecimal pricePerSqm;
    private Float areaSquareMeters;
    private Float landAreaSquareWa; // ตารางวา (สำหรับที่ดิน)

    private Integer bedrooms;
    private Integer bathrooms;
    private Integer parkingSpaces;
    private Integer floors;

    // Location
    private String addressLine;
    private String district;
    private String province;
    private String postalCode;
    private String country;
    private Double latitude;
    private Double longitude;

    // Features
    @ElementCollection
    @CollectionTable(name = "property_features")
    private Set<String> features = new HashSet<>(); // POOL, GYM, GARDEN, SECURITY

    @OneToMany(mappedBy = "property", cascade = CascadeType.ALL)
    private List<PropertyImage> images = new ArrayList<>();

    @ManyToOne
    @JoinColumn(name = "agent_id")
    private Agent agent;

    @Enumerated(EnumType.STRING)
    private PropertyStatus status; // AVAILABLE, UNDER_OFFER, SOLD, RENTED, INACTIVE

    private boolean featured;
    private Integer viewCount;
    private Integer favoriteCount;

    private LocalDateTime listedAt;
    private LocalDateTime updatedAt;
}

// Agent.java — นายหน้าอสังหาริมทรัพย์
@Entity
@Table(name = "agents")
public class Agent {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @OneToOne
    @JoinColumn(name = "user_id")
    private User user;

    private String displayName;
    private String bio;
    private String profileImageUrl;
    private String licenseNumber;
    private String phone;
    private String email;

    private Float averageRating;
    private Integer totalReviews;
    private Long propertiesListed;
    private Long propertiesSold;

    private boolean featured;
    private LocalDateTime createdAt;
}

// PropertyInquiry.java — การติดต่อสอบถามเกี่ยวกับทรัพย์
@Entity
@Table(name = "property_inquiries")
public class PropertyInquiry {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne
    @JoinColumn(name = "property_id")
    private Property property;

    private String inquirerName;
    private String inquirerEmail;
    private String inquirerPhone;

    @Column(columnDefinition = "text", nullable = false)
    private String message;

    @Enumerated(EnumType.STRING)
    private InquiryType type; // GENERAL, VIEWING_REQUEST, PRICE_OFFER

    @Enumerated(EnumType.STRING)
    private InquiryStatus status; // NEW, REPLIED, CLOSED

    private LocalDateTime createdAt;
    private LocalDateTime respondedAt;
}
```

### Service Layer

```java
// PropertyService.java — จัดการ Property Listings
@Service
@Transactional
public class PropertyService {

    @Autowired
    private PropertyRepository propertyRepository;

    @Autowired
    private PropertyInquiryRepository inquiryRepository;

    // ค้นหา Properties ด้วย filters หลายอย่าง
    public Page<Property> searchProperties(PropertySearchRequest request, Pageable pageable) {
        Specification<Property> spec = Specification.where(null);

        spec = spec.and(PropertySpecs.hasStatus(PropertyStatus.AVAILABLE));

        if (request.getListingType() != null) {
            spec = spec.and(PropertySpecs.hasListingType(request.getListingType()));
        }

        if (request.getPropertyType() != null) {
            spec = spec.and(PropertySpecs.hasPropertyType(request.getPropertyType()));
        }

        if (request.getMinPrice() != null) {
            spec = spec.and(PropertySpecs.hasPriceGreaterThan(request.getMinPrice()));
        }

        if (request.getMaxPrice() != null) {
            spec = spec.and(PropertySpecs.hasPriceLessThan(request.getMaxPrice()));
        }

        if (request.getMinBedrooms() != null) {
            spec = spec.and(PropertySpecs.hasMinBedrooms(request.getMinBedrooms()));
        }

        if (request.getProvince() != null) {
            spec = spec.and(PropertySpecs.inProvince(request.getProvince()));
        }

        // Geo search — หาทรัพย์ในรัศมี X km จากพิกัดที่กำหนด
        if (request.getLat() != null && request.getLng() != null && request.getRadiusKm() != null) {
            spec = spec.and(PropertySpecs.withinRadius(
                    request.getLat(), request.getLng(), request.getRadiusKm()));
        }

        // เพิ่ม view count
        Page<Property> results = propertyRepository.findAll(spec, pageable);
        return results;
    }

    // ส่ง Inquiry
    public PropertyInquiry submitInquiry(Long propertyId, InquiryRequest request) {
        Property property = propertyRepository.findById(propertyId).orElseThrow();

        PropertyInquiry inquiry = new PropertyInquiry();
        inquiry.setProperty(property);
        inquiry.setInquirerName(request.getName());
        inquiry.setInquirerEmail(request.getEmail());
        inquiry.setInquirerPhone(request.getPhone());
        inquiry.setMessage(request.getMessage());
        inquiry.setType(request.getType());
        inquiry.setStatus(InquiryStatus.NEW);
        inquiry.setCreatedAt(LocalDateTime.now());

        inquiry = inquiryRepository.save(inquiry);

        // แจ้ง Agent
        if (property.getAgent() != null) {
            emailService.sendInquiryNotification(
                    property.getAgent().getEmail(),
                    property.getTitle(),
                    request.getName(),
                    request.getMessage()
            );
        }

        return inquiry;
    }

    // เพิ่ม/ลบ Favorite
    public boolean toggleFavorite(Long userId, Long propertyId) {
        Optional<UserFavorite> existing = favoriteRepository.findByUserIdAndPropertyId(userId, propertyId);

        if (existing.isPresent()) {
            favoriteRepository.delete(existing.get());
            propertyRepository.decrementFavoriteCount(propertyId);
            return false;
        } else {
            UserFavorite favorite = new UserFavorite(userId, propertyId);
            favoriteRepository.save(favorite);
            propertyRepository.incrementFavoriteCount(propertyId);
            return true;
        }
    }
}

// PropertyController.java
@RestController
@RequestMapping("/api/properties")
public class PropertyController {

    @Autowired
    private PropertyService propertyService;

    @GetMapping
    public ResponseEntity<Page<PropertySummaryResponse>> search(
            PropertySearchRequest request,
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size,
            @RequestParam(defaultValue = "listedAt") String sortBy) {
        Page<Property> properties = propertyService.searchProperties(
                request, PageRequest.of(page, size, Sort.by(sortBy).descending()));
        return ResponseEntity.ok(properties.map(PropertySummaryResponse::from));
    }

    @PostMapping("/{id}/inquiry")
    public ResponseEntity<InquiryResponse> submitInquiry(
            @PathVariable Long id,
            @Valid @RequestBody InquiryRequest request) {
        PropertyInquiry inquiry = propertyService.submitInquiry(id, request);
        return ResponseEntity.status(HttpStatus.CREATED).body(InquiryResponse.from(inquiry));
    }

    @PostMapping("/{id}/favorite")
    @PreAuthorize("isAuthenticated()")
    public ResponseEntity<FavoriteResponse> toggleFavorite(
            @PathVariable Long id,
            @AuthenticationPrincipal User user) {
        boolean isFavorite = propertyService.toggleFavorite(user.getId(), id);
        return ResponseEntity.ok(new FavoriteResponse(id, isFavorite));
    }

    @GetMapping("/featured")
    public ResponseEntity<List<PropertySummaryResponse>> getFeatured(
            @RequestParam(defaultValue = "10") int limit) {
        return ResponseEntity.ok(propertyService.getFeaturedProperties(limit)
                .stream().map(PropertySummaryResponse::from).toList());
    }
}
```

---

## โปรเจค 93: Car Rental System

### ภาพรวมโปรเจค

Car Rental System เป็นระบบให้เช่ารถยนต์ครบวงจร รองรับการจัดการ Vehicles, ปฏิทินความพร้อม, การจอง, Pricing หลายประเภท (รายวัน/สัปดาห์/เดือน), Extras (ประกัน/GPS), รายงานความเสียหาย และการจัดการ Fleet

### Entities

```java
// Vehicle.java — ข้อมูลรถยนต์ในระบบ
@Entity
@Table(name = "vehicles")
public class Vehicle {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(unique = true, nullable = false)
    private String licensePlate;

    @Column(nullable = false)
    private String make; // Toyota, Honda
    private String model;
    private Integer year;
    private String color;

    @Enumerated(EnumType.STRING)
    private VehicleCategory category; // ECONOMY, COMPACT, SEDAN, SUV, LUXURY, VAN

    @Enumerated(EnumType.STRING)
    private TransmissionType transmission; // AUTOMATIC, MANUAL

    private Integer seats;
    private String fuelType; // PETROL, DIESEL, ELECTRIC, HYBRID
    private Integer engineCC;

    // Pricing
    private BigDecimal dailyRate;
    private BigDecimal weeklyRate;   // rate สำหรับ 7 วัน
    private BigDecimal monthlyRate;  // rate สำหรับ 30 วัน
    private BigDecimal securityDeposit;

    @Enumerated(EnumType.STRING)
    private VehicleStatus status; // AVAILABLE, RENTED, MAINTENANCE, RETIRED

    private String imageUrl;
    private String locationCode; // BKK, CNX, HKT
    private Long currentKilometers;

    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
}

// Reservation.java — การจองรถ
@Entity
@Table(name = "reservations")
public class Reservation {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(unique = true, nullable = false)
    private String reservationNumber;

    @ManyToOne
    @JoinColumn(name = "vehicle_id")
    private Vehicle vehicle;

    @ManyToOne
    @JoinColumn(name = "customer_id")
    private User customer;

    private LocalDate pickupDate;
    private LocalDate returnDate;
    private Integer totalDays;

    private String pickupLocation;
    private String returnLocation;

    // Pricing breakdown
    private BigDecimal baseRentalAmount;
    private BigDecimal extrasAmount;
    private BigDecimal taxAmount;
    private BigDecimal totalAmount;
    private BigDecimal depositAmount;

    @Enumerated(EnumType.STRING)
    private ReservationStatus status; // PENDING, CONFIRMED, ACTIVE, COMPLETED, CANCELLED

    // Actual pickup/return
    private LocalDateTime actualPickupAt;
    private Long pickupKilometers;
    private LocalDateTime actualReturnAt;
    private Long returnKilometers;

    @OneToMany(mappedBy = "reservation", cascade = CascadeType.ALL)
    private List<ReservationExtra> extras = new ArrayList<>();

    private String paymentIntentId;
    private LocalDateTime createdAt;
}

// DamageReport.java — รายงานความเสียหาย
@Entity
@Table(name = "damage_reports")
public class DamageReport {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne
    @JoinColumn(name = "reservation_id")
    private Reservation reservation;

    @ManyToOne
    @JoinColumn(name = "vehicle_id")
    private Vehicle vehicle;

    @Column(columnDefinition = "text", nullable = false)
    private String description;

    @ElementCollection
    @CollectionTable(name = "damage_photos")
    private List<String> photoUrls = new ArrayList<>();

    private BigDecimal estimatedRepairCost;
    private BigDecimal chargedAmount;

    @Enumerated(EnumType.STRING)
    private DamageStatus status; // REPORTED, ASSESSED, REPAIRED, CHARGED

    private LocalDateTime reportedAt;
    private LocalDateTime repairedAt;
}
```

### Service Layer

```java
// CarRentalService.java — จัดการการจองรถ
@Service
@Transactional
public class CarRentalService {

    @Autowired
    private VehicleRepository vehicleRepository;

    @Autowired
    private ReservationRepository reservationRepository;

    // ตรวจสอบความพร้อมของรถ
    public List<Vehicle> checkAvailability(String locationCode, LocalDate pickupDate,
                                            LocalDate returnDate, VehicleCategory category) {
        return vehicleRepository.findAvailableVehicles(
                locationCode, pickupDate, returnDate, category);
    }

    // สร้างการจอง
    public Reservation createReservation(CreateReservationRequest request, Long customerId) {
        Vehicle vehicle = vehicleRepository.findById(request.getVehicleId()).orElseThrow();

        // ตรวจสอบว่ารถว่างอยู่
        boolean isAvailable = reservationRepository.isVehicleAvailable(
                request.getVehicleId(), request.getPickupDate(), request.getReturnDate());

        if (!isAvailable) {
            throw new VehicleNotAvailableException("Vehicle is not available for selected dates");
        }

        int totalDays = (int) ChronoUnit.DAYS.between(request.getPickupDate(), request.getReturnDate());
        if (totalDays < 1) {
            throw new InvalidDateRangeException("Return date must be after pickup date");
        }

        // คำนวณราคา
        BigDecimal baseAmount = calculateRentalCost(vehicle, totalDays);

        // คำนวณ Extras
        BigDecimal extrasAmount = BigDecimal.ZERO;
        List<ReservationExtra> extras = new ArrayList<>();

        for (ExtraRequest extraReq : request.getExtras()) {
            RentalExtra extra = rentalExtraRepository.findById(extraReq.getExtraId()).orElseThrow();
            BigDecimal extraTotal = extra.getDailyRate().multiply(BigDecimal.valueOf(totalDays));
            extrasAmount = extrasAmount.add(extraTotal);

            ReservationExtra reservationExtra = new ReservationExtra();
            reservationExtra.setExtra(extra);
            reservationExtra.setDailyRate(extra.getDailyRate());
            reservationExtra.setTotalAmount(extraTotal);
            extras.add(reservationExtra);
        }

        BigDecimal taxAmount = baseAmount.add(extrasAmount)
                .multiply(BigDecimal.valueOf(0.07)); // VAT 7%
        BigDecimal totalAmount = baseAmount.add(extrasAmount).add(taxAmount);

        Reservation reservation = new Reservation();
        reservation.setReservationNumber(generateReservationNumber());
        reservation.setVehicle(vehicle);
        reservation.setCustomer(new User(customerId));
        reservation.setPickupDate(request.getPickupDate());
        reservation.setReturnDate(request.getReturnDate());
        reservation.setTotalDays(totalDays);
        reservation.setPickupLocation(request.getPickupLocation());
        reservation.setReturnLocation(request.getReturnLocation());
        reservation.setBaseRentalAmount(baseAmount);
        reservation.setExtrasAmount(extrasAmount);
        reservation.setTaxAmount(taxAmount);
        reservation.setTotalAmount(totalAmount);
        reservation.setDepositAmount(vehicle.getSecurityDeposit());
        reservation.setStatus(ReservationStatus.PENDING);
        reservation.setExtras(extras);
        reservation.setCreatedAt(LocalDateTime.now());

        return reservationRepository.save(reservation);
    }

    // คำนวณราคาเช่า
    private BigDecimal calculateRentalCost(Vehicle vehicle, int days) {
        if (days >= 30) {
            return vehicle.getMonthlyRate().multiply(BigDecimal.valueOf(days / 30.0));
        } else if (days >= 7) {
            return vehicle.getWeeklyRate().multiply(BigDecimal.valueOf(days / 7.0));
        } else {
            return vehicle.getDailyRate().multiply(BigDecimal.valueOf(days));
        }
    }

    // Record Pickup
    public Reservation recordPickup(Long reservationId, Long currentKm) {
        Reservation reservation = reservationRepository.findById(reservationId).orElseThrow();
        reservation.setActualPickupAt(LocalDateTime.now());
        reservation.setPickupKilometers(currentKm);
        reservation.setStatus(ReservationStatus.ACTIVE);
        reservation.getVehicle().setStatus(VehicleStatus.RENTED);
        return reservationRepository.save(reservation);
    }

    // Record Return
    public Reservation recordReturn(Long reservationId, Long returnKm, String conditionNotes) {
        Reservation reservation = reservationRepository.findById(reservationId).orElseThrow();
        reservation.setActualReturnAt(LocalDateTime.now());
        reservation.setReturnKilometers(returnKm);
        reservation.setStatus(ReservationStatus.COMPLETED);
        reservation.getVehicle().setStatus(VehicleStatus.AVAILABLE);
        reservation.getVehicle().setCurrentKilometers(returnKm);
        return reservationRepository.save(reservation);
    }
}

// CarRentalController.java
@RestController
@RequestMapping("/api/rentals")
public class CarRentalController {

    @Autowired
    private CarRentalService carRentalService;

    @GetMapping("/vehicles/available")
    public ResponseEntity<List<VehicleResponse>> checkAvailability(
            @RequestParam String location,
            @RequestParam LocalDate pickupDate,
            @RequestParam LocalDate returnDate,
            @RequestParam(required = false) VehicleCategory category) {
        return ResponseEntity.ok(carRentalService.checkAvailability(location, pickupDate, returnDate, category)
                .stream().map(VehicleResponse::from).toList());
    }

    @PostMapping("/reservations")
    @PreAuthorize("isAuthenticated()")
    public ResponseEntity<ReservationResponse> createReservation(
            @Valid @RequestBody CreateReservationRequest request,
            @AuthenticationPrincipal User user) {
        Reservation reservation = carRentalService.createReservation(request, user.getId());
        return ResponseEntity.status(HttpStatus.CREATED).body(ReservationResponse.from(reservation));
    }

    @PostMapping("/reservations/{id}/pickup")
    @PreAuthorize("hasRole('RENTAL_STAFF')")
    public ResponseEntity<ReservationResponse> recordPickup(
            @PathVariable Long id,
            @RequestBody PickupRequest request) {
        Reservation reservation = carRentalService.recordPickup(id, request.getCurrentKm());
        return ResponseEntity.ok(ReservationResponse.from(reservation));
    }

    @PostMapping("/reservations/{id}/return")
    @PreAuthorize("hasRole('RENTAL_STAFF')")
    public ResponseEntity<ReservationResponse> recordReturn(
            @PathVariable Long id,
            @RequestBody ReturnRequest request) {
        Reservation reservation = carRentalService.recordReturn(id, request.getReturnKm(),
                request.getConditionNotes());
        return ResponseEntity.ok(ReservationResponse.from(reservation));
    }
}
```

---

## โปรเจค 94: Insurance Management System

### ภาพรวมโปรเจค

Insurance Management System เป็นระบบจัดการประกันภัยครบวงจร รองรับ Policies หลาย Coverage Types, Claims Processing Workflow, การคำนวณ Premium, Renewal Reminders และการจัดเก็บเอกสาร

### Entities

```java
// InsurancePolicy.java — กรมธรรม์ประกันภัย
@Entity
@Table(name = "insurance_policies")
public class InsurancePolicy {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(unique = true, nullable = false)
    private String policyNumber;

    @ManyToOne
    @JoinColumn(name = "customer_id")
    private User customer;

    @Enumerated(EnumType.STRING)
    private InsuranceType type; // HEALTH, LIFE, AUTO, HOME, TRAVEL

    @Column(nullable = false)
    private String productName;

    @Column(nullable = false)
    private BigDecimal coverageAmount;

    @Column(nullable = false)
    private BigDecimal annualPremium;

    private BigDecimal monthlyPremium;

    @Enumerated(EnumType.STRING)
    private PaymentFrequency paymentFrequency; // MONTHLY, QUARTERLY, ANNUALLY

    private LocalDate startDate;
    private LocalDate endDate;
    private LocalDate nextPaymentDate;

    @Enumerated(EnumType.STRING)
    private PolicyStatus status; // PENDING, ACTIVE, EXPIRED, CANCELLED, LAPSED

    @OneToMany(mappedBy = "policy", cascade = CascadeType.ALL)
    private List<PolicyCoverage> coverages = new ArrayList<>();

    // Documents
    @ElementCollection
    @CollectionTable(name = "policy_documents")
    private List<String> documentUrls = new ArrayList<>();

    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
}

// Claim.java — การเคลม
@Entity
@Table(name = "claims")
public class Claim {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(unique = true, nullable = false)
    private String claimNumber;

    @ManyToOne
    @JoinColumn(name = "policy_id")
    private InsurancePolicy policy;

    @Column(columnDefinition = "text", nullable = false)
    private String description;

    @Enumerated(EnumType.STRING)
    private ClaimType claimType;

    @Column(nullable = false)
    private BigDecimal claimedAmount;

    private BigDecimal approvedAmount;
    private BigDecimal paidAmount;

    @Enumerated(EnumType.STRING)
    private ClaimStatus status; // SUBMITTED, UNDER_REVIEW, APPROVED, REJECTED, PAID, CLOSED

    private LocalDate incidentDate;
    private LocalDateTime submittedAt;
    private LocalDateTime reviewedAt;
    private LocalDateTime paidAt;

    @OneToMany(mappedBy = "claim", cascade = CascadeType.ALL)
    private List<ClaimDocument> documents = new ArrayList<>();

    private String assessorNotes;
    private String rejectionReason;
}
```

### Service Layer

```java
// InsuranceService.java — จัดการ Policy และ Claims
@Service
@Transactional
public class InsuranceService {

    @Autowired
    private InsurancePolicyRepository policyRepository;

    @Autowired
    private ClaimRepository claimRepository;

    // คำนวณ Premium
    public PremiumCalculation calculatePremium(PremiumCalculationRequest request) {
        BigDecimal basePremium = getBasePremium(request.getInsuranceType(), request.getCoverageAmount());

        // Risk factors
        BigDecimal multiplier = BigDecimal.ONE;

        if (request.getAge() != null) {
            if (request.getAge() > 60) multiplier = multiplier.multiply(new BigDecimal("1.5"));
            else if (request.getAge() > 45) multiplier = multiplier.multiply(new BigDecimal("1.2"));
        }

        if (Boolean.TRUE.equals(request.getSmoker())) {
            multiplier = multiplier.multiply(new BigDecimal("1.3"));
        }

        BigDecimal annualPremium = basePremium.multiply(multiplier)
                .setScale(2, RoundingMode.HALF_UP);

        return PremiumCalculation.builder()
                .annualPremium(annualPremium)
                .monthlyPremium(annualPremium.divide(BigDecimal.valueOf(12), 2, RoundingMode.HALF_UP))
                .build();
    }

    // ยื่นเคลม
    public Claim submitClaim(Long policyId, SubmitClaimRequest request, Long customerId) {
        InsurancePolicy policy = policyRepository.findById(policyId).orElseThrow();

        if (!policy.getCustomer().getId().equals(customerId)) {
            throw new AccessDeniedException("Not your policy");
        }

        if (policy.getStatus() != PolicyStatus.ACTIVE) {
            throw new PolicyNotActiveException("Policy is not active");
        }

        Claim claim = new Claim();
        claim.setClaimNumber(generateClaimNumber());
        claim.setPolicy(policy);
        claim.setDescription(request.getDescription());
        claim.setClaimType(request.getClaimType());
        claim.setClaimedAmount(request.getClaimedAmount());
        claim.setStatus(ClaimStatus.SUBMITTED);
        claim.setIncidentDate(request.getIncidentDate());
        claim.setSubmittedAt(LocalDateTime.now());

        return claimRepository.save(claim);
    }

    // Process Claim
    public Claim processClaim(Long claimId, ProcessClaimRequest request, Long assessorId) {
        Claim claim = claimRepository.findById(claimId).orElseThrow();

        if (request.isApproved()) {
            claim.setStatus(ClaimStatus.APPROVED);
            claim.setApprovedAmount(request.getApprovedAmount());
            claim.setAssessorNotes(request.getNotes());
        } else {
            claim.setStatus(ClaimStatus.REJECTED);
            claim.setRejectionReason(request.getRejectionReason());
        }

        claim.setReviewedAt(LocalDateTime.now());
        return claimRepository.save(claim);
    }

    // Renewal reminders
    @Scheduled(cron = "0 0 9 * * *")
    public void sendRenewalReminders() {
        List<InsurancePolicy> expiring = policyRepository
                .findByEndDateBetween(LocalDate.now(), LocalDate.now().plusDays(30));

        for (InsurancePolicy policy : expiring) {
            int daysLeft = (int) ChronoUnit.DAYS.between(LocalDate.now(), policy.getEndDate());
            emailService.sendRenewalReminder(
                    policy.getCustomer().getEmail(),
                    policy.getPolicyNumber(),
                    policy.getProductName(),
                    daysLeft
            );
        }
    }
}

// InsuranceController.java
@RestController
@RequestMapping("/api/insurance")
public class InsuranceController {

    @Autowired
    private InsuranceService insuranceService;

    @PostMapping("/calculate-premium")
    public ResponseEntity<PremiumCalculation> calculate(
            @RequestBody PremiumCalculationRequest request) {
        return ResponseEntity.ok(insuranceService.calculatePremium(request));
    }

    @PostMapping("/policies/{id}/claims")
    @PreAuthorize("isAuthenticated()")
    public ResponseEntity<ClaimResponse> submitClaim(
            @PathVariable Long id,
            @RequestBody SubmitClaimRequest request,
            @AuthenticationPrincipal User user) {
        Claim claim = insuranceService.submitClaim(id, request, user.getId());
        return ResponseEntity.status(HttpStatus.CREATED).body(ClaimResponse.from(claim));
    }

    @PutMapping("/claims/{id}/process")
    @PreAuthorize("hasRole('ASSESSOR')")
    public ResponseEntity<ClaimResponse> processClaim(
            @PathVariable Long id,
            @RequestBody ProcessClaimRequest request,
            @AuthenticationPrincipal User user) {
        Claim claim = insuranceService.processClaim(id, request, user.getId());
        return ResponseEntity.ok(ClaimResponse.from(claim));
    }
}
```

---

## โปรเจค 95: Chatbot Backend Service

### ภาพรวมโปรเจค

Chatbot Backend Service เป็นระบบ Backend สำหรับ Chatbot รองรับการกำหนด Intent Definitions, Entity Extraction, จัดการ Conversation Sessions, Context Management, การส่งต่อให้ Human Agent, การ Integrate กับ NLP Services (Dialogflow/Rasa) และ Analytics

### Entities

```java
// Intent.java — กำหนด Intent ที่ Bot รองรับ
@Entity
@Table(name = "intents")
public class Intent {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(unique = true, nullable = false)
    private String name; // GREETING, HELP, ORDER_STATUS, COMPLAINT

    private String description;

    // Training phrases สำหรับ NLP
    @ElementCollection
    @CollectionTable(name = "intent_training_phrases")
    private List<String> trainingPhrases = new ArrayList<>();

    // Response templates
    @ElementCollection
    @CollectionTable(name = "intent_responses")
    private List<String> responses = new ArrayList<>();

    @ElementCollection
    @CollectionTable(name = "intent_entities")
    private Set<String> requiredEntities = new HashSet<>();

    private boolean requiresHandoff; // ต้อง escalate ให้ human หรือเปล่า
    private boolean active;
    private LocalDateTime createdAt;
}

// ConversationSession.java — Session การสนทนา
@Entity
@Table(name = "conversation_sessions")
public class ConversationSession {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(unique = true, nullable = false)
    private String sessionId;

    private Long userId;
    private String channel; // WEB, LINE, FACEBOOK, API

    @Enumerated(EnumType.STRING)
    private SessionStatus status; // ACTIVE, HANDED_OFF, CLOSED

    // Context ที่สะสมระหว่างการสนทนา
    @Column(columnDefinition = "jsonb")
    private String context;

    // Long-term memory
    @Column(columnDefinition = "jsonb")
    private String userMemory;

    private String assignedAgentId; // เมื่อ hand off ให้ human

    private LocalDateTime startedAt;
    private LocalDateTime lastActivityAt;
    private LocalDateTime endedAt;
}

// ConversationMessage.java — ข้อความในการสนทนา
@Entity
@Table(name = "conversation_messages")
public class ConversationMessage {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne
    @JoinColumn(name = "session_id")
    private ConversationSession session;

    @Enumerated(EnumType.STRING)
    private MessageRole role; // USER, BOT, AGENT

    @Column(columnDefinition = "text", nullable = false)
    private String content;

    private String detectedIntent;
    private Float intentConfidence;

    // Extracted entities เช่น {"orderId": "123", "date": "2024-01-01"}
    @Column(columnDefinition = "jsonb")
    private String extractedEntities;

    private boolean handedOff;
    private LocalDateTime sentAt;
}
```

### Chatbot Service

```java
// ChatbotService.java — Core Chatbot Logic
@Service
public class ChatbotService {

    @Autowired
    private ConversationSessionRepository sessionRepository;

    @Autowired
    private ConversationMessageRepository messageRepository;

    @Autowired
    private IntentRepository intentRepository;

    @Autowired
    private NlpService nlpService; // Dialogflow หรือ Rasa client

    @Autowired
    private ObjectMapper objectMapper;

    // ประมวลผล User Message
    public ChatbotResponse processMessage(String sessionId, String userMessage, Long userId) {
        ConversationSession session = getOrCreateSession(sessionId, userId);

        // บันทึก User Message
        ConversationMessage userMsg = saveMessage(session, MessageRole.USER, userMessage);

        // Detect Intent ผ่าน NLP
        NlpResult nlpResult = nlpService.detectIntent(session.getSessionId(), userMessage,
                parseContext(session.getContext()));

        userMsg.setDetectedIntent(nlpResult.getIntentName());
        userMsg.setIntentConfidence(nlpResult.getConfidence());
        userMsg.setExtractedEntities(toJson(nlpResult.getEntities()));
        messageRepository.save(userMsg);

        // อัปเดต Context
        updateContext(session, nlpResult);

        // ตรวจสอบว่าต้อง Escalate ไหม
        if (shouldHandOff(session, nlpResult)) {
            return handleHandOff(session);
        }

        // สร้าง Bot Response
        String botResponse = generateResponse(session, nlpResult);

        ConversationMessage botMsg = saveMessage(session, MessageRole.BOT, botResponse);

        session.setLastActivityAt(LocalDateTime.now());
        sessionRepository.save(session);

        return ChatbotResponse.builder()
                .sessionId(sessionId)
                .message(botResponse)
                .intent(nlpResult.getIntentName())
                .confidence(nlpResult.getConfidence())
                .entities(nlpResult.getEntities())
                .requiresHandoff(false)
                .build();
    }

    private boolean shouldHandOff(ConversationSession session, NlpResult nlpResult) {
        // Hand off ถ้า intent ต้องการ
        if (nlpResult.getIntentName() != null) {
            Intent intent = intentRepository.findByName(nlpResult.getIntentName()).orElse(null);
            if (intent != null && intent.isRequiresHandoff()) return true;
        }

        // Hand off ถ้า confidence ต่ำเกินไป
        if (nlpResult.getConfidence() < 0.5f) {
            // ตรวจสอบ consecutive low confidence
            long lowConfidenceCount = messageRepository.countRecentLowConfidenceMessages(
                    session.getId(), 0.5f, 3);
            if (lowConfidenceCount >= 3) return true;
        }

        // User ขอพูดกับ Human
        if (nlpResult.getIntentName() != null &&
                nlpResult.getIntentName().equals("SPEAK_TO_HUMAN")) return true;

        return false;
    }

    private ChatbotResponse handleHandOff(ConversationSession session) {
        session.setStatus(SessionStatus.HANDED_OFF);
        sessionRepository.save(session);

        // แจ้ง Agent ที่ว่าง
        notifyAvailableAgent(session);

        return ChatbotResponse.builder()
                .sessionId(session.getSessionId())
                .message("กำลังเชื่อมต่อกับเจ้าหน้าที่ กรุณารอสักครู่...")
                .requiresHandoff(true)
                .build();
    }

    private ConversationSession getOrCreateSession(String sessionId, Long userId) {
        return sessionRepository.findBySessionId(sessionId)
                .orElseGet(() -> {
                    ConversationSession session = new ConversationSession();
                    session.setSessionId(sessionId);
                    session.setUserId(userId);
                    session.setStatus(SessionStatus.ACTIVE);
                    session.setContext("{}");
                    session.setStartedAt(LocalDateTime.now());
                    session.setLastActivityAt(LocalDateTime.now());
                    return sessionRepository.save(session);
                });
    }

    // Analytics: Intent distribution
    public Map<String, Long> getIntentAnalytics(LocalDateTime from, LocalDateTime to) {
        return messageRepository.countByIntentBetween(from, to).stream()
                .collect(Collectors.toMap(
                        r -> (String) r[0],
                        r -> (Long) r[1]
                ));
    }
}

// ChatbotController.java
@RestController
@RequestMapping("/api/chatbot")
public class ChatbotController {

    @Autowired
    private ChatbotService chatbotService;

    @PostMapping("/message")
    public ResponseEntity<ChatbotResponse> processMessage(
            @RequestBody ChatMessageRequest request,
            @AuthenticationPrincipal(required = false) User user) {

        Long userId = user != null ? user.getId() : null;
        ChatbotResponse response = chatbotService.processMessage(
                request.getSessionId(),
                request.getMessage(),
                userId
        );

        return ResponseEntity.ok(response);
    }

    @GetMapping("/sessions/{sessionId}/history")
    public ResponseEntity<List<MessageResponse>> getHistory(@PathVariable String sessionId) {
        List<ConversationMessage> messages = chatbotService.getSessionHistory(sessionId);
        return ResponseEntity.ok(messages.stream().map(MessageResponse::from).toList());
    }

    @PostMapping("/sessions/{sessionId}/handoff")
    public ResponseEntity<Void> requestHandoff(@PathVariable String sessionId) {
        chatbotService.requestHandoff(sessionId);
        return ResponseEntity.ok().build();
    }

    @GetMapping("/analytics/intents")
    @PreAuthorize("hasRole('ANALYST')")
    public ResponseEntity<Map<String, Long>> getIntentAnalytics(
            @RequestParam LocalDateTime from,
            @RequestParam LocalDateTime to) {
        return ResponseEntity.ok(chatbotService.getIntentAnalytics(from, to));
    }
}
```

### SQL Scripts

```sql
-- migrations/V1__marketplace_realestate.sql
CREATE TABLE vendors (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT UNIQUE NOT NULL,
    store_name VARCHAR(255) UNIQUE NOT NULL,
    store_slug VARCHAR(255) UNIQUE,
    description TEXT,
    logo_url TEXT,
    banner_url TEXT,
    email VARCHAR(255),
    commission_rate FLOAT DEFAULT 10.0,
    status VARCHAR(50) DEFAULT 'PENDING',
    total_orders BIGINT DEFAULT 0,
    total_revenue DECIMAL(15,2) DEFAULT 0,
    pending_payout DECIMAL(15,2) DEFAULT 0,
    created_at TIMESTAMP DEFAULT NOW(),
    verified_at TIMESTAMP
);

CREATE TABLE properties (
    id BIGSERIAL PRIMARY KEY,
    title VARCHAR(500) NOT NULL,
    description TEXT,
    property_type VARCHAR(50) NOT NULL,
    listing_type VARCHAR(50) NOT NULL,
    price DECIMAL(15,2) NOT NULL,
    area_square_meters FLOAT,
    bedrooms INTEGER,
    bathrooms INTEGER,
    district VARCHAR(255),
    province VARCHAR(255),
    latitude DOUBLE PRECISION,
    longitude DOUBLE PRECISION,
    status VARCHAR(50) DEFAULT 'AVAILABLE',
    featured BOOLEAN DEFAULT FALSE,
    view_count INTEGER DEFAULT 0,
    favorite_count INTEGER DEFAULT 0,
    listed_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE conversation_sessions (
    id BIGSERIAL PRIMARY KEY,
    session_id VARCHAR(255) UNIQUE NOT NULL,
    user_id BIGINT,
    channel VARCHAR(50),
    status VARCHAR(50) DEFAULT 'ACTIVE',
    context JSONB DEFAULT '{}',
    user_memory JSONB,
    assigned_agent_id VARCHAR(255),
    started_at TIMESTAMP DEFAULT NOW(),
    last_activity_at TIMESTAMP DEFAULT NOW(),
    ended_at TIMESTAMP
);

CREATE TABLE conversation_messages (
    id BIGSERIAL PRIMARY KEY,
    session_id BIGINT REFERENCES conversation_sessions(id),
    role VARCHAR(20) NOT NULL,
    content TEXT NOT NULL,
    detected_intent VARCHAR(255),
    intent_confidence FLOAT,
    extracted_entities JSONB,
    handed_off BOOLEAN DEFAULT FALSE,
    sent_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_sessions_session_id ON conversation_sessions(session_id);
CREATE INDEX idx_messages_session ON conversation_messages(session_id, sent_at);
CREATE INDEX idx_properties_province ON properties(province, status);
CREATE INDEX idx_properties_price ON properties(price, listing_type);
```

---

## สรุป Part 119

ใน Part นี้เราได้สร้าง Marketplace & Real Estate Platforms ระดับ Production:

| โปรเจค | เทคโนโลยีหลัก | ความยาก |
|--------|---------------|---------|
| 91. Multi-vendor Marketplace | Order splitting, Commission, Vendor payouts | ⭐⭐⭐⭐⭐ |
| 92. Real Estate Platform | Geo search, Agent profiles, Inquiries | ⭐⭐⭐⭐ |
| 93. Car Rental System | Availability calendar, Dynamic pricing, Fleet management | ⭐⭐⭐⭐ |
| 94. Insurance Management | Premium calculation, Claims workflow, Renewals | ⭐⭐⭐⭐ |
| 95. Chatbot Backend | Intent detection, Context management, Human handoff | ⭐⭐⭐⭐⭐ |

### Key Takeaways

1. **Multi-vendor Marketplace** — การแบ่ง Order ตาม Vendor ต้องทำใน Single Transaction เพื่อ data integrity
2. **Geo Search** — PostgreSQL PostGIS extension หรือ Haversine formula ใช้สำหรับ radius search
3. **Chatbot** — Context management เป็น challenge ใหญ่ — ต้องเก็บ state ของ conversation อย่างชัดเจน
4. **Insurance** — Regulatory compliance ต้องการ audit trail ที่ครบถ้วนทุก transaction

*[← Part 118](./part-118-workflow-automation.md) | [Part 120 →](./part-120-ai-advanced-projects.md)*

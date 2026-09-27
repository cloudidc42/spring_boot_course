# Part 105: โปรเจค 21-25 — Booking & Marketplace

> **ระดับ:** ระดับโลก | **เวลาเรียนรู้:** 10-15 ชั่วโมง | **โปรเจค:** 5 โปรเจคสมบูรณ์

ในส่วนสุดท้ายของคอร์สนี้ เราจะสร้างระบบ Booking และ Marketplace ที่ครอบคลุมการประมูล การจองนัดหมาย แบบสำรวจ ระบบรีวิว และ Support Ticket

---

## โปรเจคที่ 21: Auction Platform

### ภาพรวมระบบ

แพลตฟอร์มประมูลออนไลน์ที่รองรับรายการสินค้า การประมูล Auto-bid, นาฬิกาจับเวลา การแจ้งเตือนผู้ชนะ การเชื่อมต่อการชำระเงิน และประวัติการประมูล ระบบใช้ WebSocket เพื่อแสดงราคาเรียลไทม์

### Entity Classes

```java
// AuctionItem.java
@Entity
@Table(name = "auction_items")
@Data
@NoArgsConstructor
public class AuctionItem {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String title;

    @Column(columnDefinition = "TEXT")
    private String description;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "seller_id")
    private User seller;

    private String category;

    @Column(precision = 12, scale = 2)
    private BigDecimal startingPrice;

    @Column(precision = 12, scale = 2)
    private BigDecimal reservePrice; // Minimum acceptable price (hidden)

    @Column(precision = 12, scale = 2)
    private BigDecimal buyNowPrice; // Optional Buy It Now price

    @Column(precision = 12, scale = 2)
    private BigDecimal currentPrice;

    @Column(precision = 12, scale = 2)
    private BigDecimal minimumBidIncrement = new BigDecimal("10.00");

    @Enumerated(EnumType.STRING)
    private AuctionStatus status = AuctionStatus.DRAFT;

    private LocalDateTime auctionStartTime;
    private LocalDateTime auctionEndTime;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "current_winner_id")
    private User currentWinner;

    private Integer bidCount = 0;
    private Integer viewCount = 0;

    @ElementCollection
    @CollectionTable(name = "auction_item_images")
    private List<String> imageUrls = new ArrayList<>();

    @OneToMany(mappedBy = "item")
    @OrderBy("bidAmount DESC")
    private List<Bid> bids = new ArrayList<>();

    @CreatedDate
    private LocalDateTime createdAt;
}

// Bid.java
@Entity
@Table(name = "bids")
@Data
@NoArgsConstructor
public class Bid {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "item_id")
    private AuctionItem item;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "bidder_id")
    private User bidder;

    @Column(precision = 12, scale = 2, nullable = false)
    private BigDecimal bidAmount;

    @Column(precision = 12, scale = 2)
    private BigDecimal maxAutoBidAmount; // For auto-bidding

    @Enumerated(EnumType.STRING)
    private BidStatus status = BidStatus.ACTIVE;

    @Column(nullable = false)
    private Boolean isAutoBid = false;

    @CreatedDate
    private LocalDateTime bidTime;
}

// AutoBidConfig.java
@Entity
@Table(name = "auto_bid_configs",
       uniqueConstraints = @UniqueConstraint(columnNames = {"item_id", "user_id"}))
@Data
@NoArgsConstructor
public class AutoBidConfig {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne
    @JoinColumn(name = "item_id")
    private AuctionItem item;

    @ManyToOne
    @JoinColumn(name = "user_id")
    private User user;

    @Column(precision = 12, scale = 2)
    private BigDecimal maxAmount;

    private Boolean active = true;
}

// AuctionPayment.java
@Entity
@Table(name = "auction_payments")
@Data
@NoArgsConstructor
public class AuctionPayment {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @OneToOne
    @JoinColumn(name = "item_id")
    private AuctionItem item;

    @ManyToOne
    @JoinColumn(name = "winner_id")
    private User winner;

    @Column(precision = 12, scale = 2)
    private BigDecimal amount;

    @Enumerated(EnumType.STRING)
    private PaymentStatus status = PaymentStatus.PENDING;

    private LocalDateTime paymentDeadline;
    private LocalDateTime paidAt;
    private String stripePaymentIntentId;
}

public enum AuctionStatus { DRAFT, SCHEDULED, LIVE, ENDED, SOLD, CANCELLED, RESERVE_NOT_MET }
public enum BidStatus { ACTIVE, OUTBID, WINNING, WON, LOST }
public enum PaymentStatus { PENDING, PAID, FAILED, REFUNDED }
```

### Repository Layer

```java
// AuctionItemRepository.java
@Repository
public interface AuctionItemRepository extends JpaRepository<AuctionItem, Long> {
    Page<AuctionItem> findByStatus(AuctionStatus status, Pageable pageable);

    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("SELECT a FROM AuctionItem a WHERE a.id = :id")
    Optional<AuctionItem> findByIdWithLock(@Param("id") Long id);

    @Query("SELECT a FROM AuctionItem a WHERE a.status = 'LIVE' AND a.auctionEndTime < :now")
    List<AuctionItem> findExpiredLiveAuctions(@Param("now") LocalDateTime now);

    @Query("SELECT a FROM AuctionItem a WHERE a.status = 'SCHEDULED' AND a.auctionStartTime <= :now")
    List<AuctionItem> findDueToStart(@Param("now") LocalDateTime now);
}

// BidRepository.java
@Repository
public interface BidRepository extends JpaRepository<Bid, Long> {
    List<Bid> findByItemIdOrderByBidAmountDesc(Long itemId);
    Optional<Bid> findTopByItemIdOrderByBidAmountDesc(Long itemId);
    Page<Bid> findByBidderIdOrderByBidTimeDesc(Long bidderId, Pageable pageable);
    Optional<BigDecimal> findMaxBidAmountByItemId(Long itemId);
}

// AutoBidConfigRepository.java
@Repository
public interface AutoBidConfigRepository extends JpaRepository<AutoBidConfig, Long> {
    List<AutoBidConfig> findByItemIdAndActiveTrue(Long itemId);
    Optional<AutoBidConfig> findByItemIdAndUserId(Long itemId, Long userId);
}
```

### Service Layer

```java
// AuctionService.java
@Service
@RequiredArgsConstructor
@Slf4j
public class AuctionService {

    private final AuctionItemRepository itemRepository;
    private final BidRepository bidRepository;
    private final AutoBidConfigRepository autoBidConfigRepository;
    private final AuctionPaymentRepository paymentRepository;
    private final EmailService emailService;
    private final SimpMessagingTemplate messagingTemplate; // WebSocket

    @Transactional
    public BidResult placeBid(Long itemId, Long bidderId, BigDecimal bidAmount) {
        AuctionItem item = itemRepository.findByIdWithLock(itemId)
            .orElseThrow(() -> new ResourceNotFoundException("Auction item not found"));

        // Validate auction state
        if (item.getStatus() != AuctionStatus.LIVE) {
            throw new BadRequestException("Auction is not live");
        }

        if (LocalDateTime.now().isAfter(item.getAuctionEndTime())) {
            throw new BadRequestException("Auction has ended");
        }

        if (bidderId.equals(item.getSeller().getId())) {
            throw new BadRequestException("Seller cannot bid on their own item");
        }

        // Check minimum bid
        BigDecimal minBid = item.getCurrentPrice() != null ?
            item.getCurrentPrice().add(item.getMinimumBidIncrement()) :
            item.getStartingPrice();

        if (bidAmount.compareTo(minBid) < 0) {
            throw new BadRequestException("Bid must be at least " + minBid);
        }

        // Check Buy Now
        if (item.getBuyNowPrice() != null && bidAmount.compareTo(item.getBuyNowPrice()) >= 0) {
            return processBuyNow(item, bidderId, item.getBuyNowPrice());
        }

        // Outbid previous winner
        if (item.getCurrentWinner() != null && !item.getCurrentWinner().getId().equals(bidderId)) {
            notifyOutbid(item.getCurrentWinner(), item);
        }

        // Create bid
        Bid bid = new Bid();
        bid.setItem(item);
        User bidder = new User();
        bidder.setId(bidderId);
        bid.setBidder(bidder);
        bid.setBidAmount(bidAmount);
        bidRepository.save(bid);

        // Update item
        item.setCurrentPrice(bidAmount);
        item.setCurrentWinner(bidder);
        item.setBidCount(item.getBidCount() + 1);
        itemRepository.save(item);

        // Extend auction if bid placed near end (anti-sniping)
        if (ChronoUnit.MINUTES.between(LocalDateTime.now(), item.getAuctionEndTime()) < 5) {
            item.setAuctionEndTime(item.getAuctionEndTime().plusMinutes(5));
            itemRepository.save(item);
        }

        // Trigger auto-bids from other users
        triggerAutoBids(item, bidderId);

        // Broadcast via WebSocket
        broadcastBidUpdate(item);

        return BidResult.builder()
            .itemId(itemId)
            .currentPrice(item.getCurrentPrice())
            .bidCount(item.getBidCount())
            .auctionEndTime(item.getAuctionEndTime())
            .isWinning(true)
            .build();
    }

    @Transactional
    public void setAutoBid(Long itemId, Long userId, BigDecimal maxAmount) {
        AuctionItem item = itemRepository.findById(itemId)
            .orElseThrow(() -> new ResourceNotFoundException("Auction item not found"));

        if (item.getStatus() != AuctionStatus.LIVE) {
            throw new BadRequestException("Can only set auto-bid on live auctions");
        }

        AutoBidConfig config = autoBidConfigRepository.findByItemIdAndUserId(itemId, userId)
            .orElse(new AutoBidConfig());

        config.setItem(item);
        User user = new User();
        user.setId(userId);
        config.setUser(user);
        config.setMaxAmount(maxAmount);
        config.setActive(true);
        autoBidConfigRepository.save(config);

        // Trigger immediate auto-bid if needed
        triggerAutoBids(item, -1L);
    }

    private void triggerAutoBids(AuctionItem item, Long excludeUserId) {
        List<AutoBidConfig> configs = autoBidConfigRepository.findByItemIdAndActiveTrue(item.getId());

        // Sort by max amount (highest first) and exclude current winner
        configs.stream()
            .filter(c -> !c.getUser().getId().equals(excludeUserId))
            .filter(c -> !c.getUser().getId().equals(item.getCurrentWinner() != null ?
                        item.getCurrentWinner().getId() : -1L))
            .filter(c -> {
                BigDecimal needed = item.getCurrentPrice().add(item.getMinimumBidIncrement());
                return c.getMaxAmount().compareTo(needed) >= 0;
            })
            .max(Comparator.comparing(AutoBidConfig::getMaxAmount))
            .ifPresent(config -> {
                BigDecimal autoBidAmount = item.getCurrentPrice().add(item.getMinimumBidIncrement());

                Bid autoBid = new Bid();
                autoBid.setItem(item);
                autoBid.setBidder(config.getUser());
                autoBid.setBidAmount(autoBidAmount);
                autoBid.setIsAutoBid(true);
                autoBid.setMaxAutoBidAmount(config.getMaxAmount());
                bidRepository.save(autoBid);

                item.setCurrentPrice(autoBidAmount);
                item.setCurrentWinner(config.getUser());
                item.setBidCount(item.getBidCount() + 1);
                itemRepository.save(item);

                broadcastBidUpdate(item);
            });
    }

    @Scheduled(fixedRate = 30000) // Every 30 seconds
    @Transactional
    public void processEndedAuctions() {
        List<AuctionItem> endedItems = itemRepository.findExpiredLiveAuctions(LocalDateTime.now());

        for (AuctionItem item : endedItems) {
            try {
                if (item.getCurrentWinner() != null) {
                    // Check if reserve price was met
                    if (item.getReservePrice() != null &&
                        item.getCurrentPrice().compareTo(item.getReservePrice()) < 0) {
                        item.setStatus(AuctionStatus.RESERVE_NOT_MET);
                        emailService.notifyReserveNotMet(item.getSeller().getEmail(), item);
                    } else {
                        item.setStatus(AuctionStatus.ENDED);
                        createAuctionPayment(item);
                        emailService.notifyWinner(item.getCurrentWinner().getEmail(), item);
                        emailService.notifySellerSale(item.getSeller().getEmail(), item);
                    }
                } else {
                    item.setStatus(AuctionStatus.ENDED);
                }
                itemRepository.save(item);
                broadcastAuctionEnded(item);
            } catch (Exception e) {
                log.error("Error processing ended auction {}: {}", item.getId(), e.getMessage());
            }
        }
    }

    @Scheduled(fixedRate = 60000)
    @Transactional
    public void startScheduledAuctions() {
        itemRepository.findDueToStart(LocalDateTime.now()).forEach(item -> {
            item.setStatus(AuctionStatus.LIVE);
            itemRepository.save(item);
            log.info("Started auction for item: {}", item.getId());
        });
    }

    private BidResult processBuyNow(AuctionItem item, Long buyerId, BigDecimal price) {
        item.setStatus(AuctionStatus.SOLD);
        item.setCurrentPrice(price);
        User buyer = new User();
        buyer.setId(buyerId);
        item.setCurrentWinner(buyer);
        itemRepository.save(item);

        createAuctionPayment(item);
        emailService.notifyBuyNow(buyer.getEmail(), item);

        broadcastAuctionEnded(item);

        return BidResult.builder()
            .itemId(item.getId())
            .currentPrice(price)
            .auctionEndTime(item.getAuctionEndTime())
            .isWinning(true)
            .buyNowPurchased(true)
            .build();
    }

    private void createAuctionPayment(AuctionItem item) {
        AuctionPayment payment = new AuctionPayment();
        payment.setItem(item);
        payment.setWinner(item.getCurrentWinner());
        payment.setAmount(item.getCurrentPrice());
        payment.setPaymentDeadline(LocalDateTime.now().plusDays(3));
        paymentRepository.save(payment);
    }

    private void broadcastBidUpdate(AuctionItem item) {
        BidUpdateMessage msg = BidUpdateMessage.builder()
            .itemId(item.getId())
            .currentPrice(item.getCurrentPrice())
            .bidCount(item.getBidCount())
            .auctionEndTime(item.getAuctionEndTime())
            .build();
        messagingTemplate.convertAndSend("/topic/auction/" + item.getId(), msg);
    }

    private void broadcastAuctionEnded(AuctionItem item) {
        messagingTemplate.convertAndSend("/topic/auction/" + item.getId() + "/ended",
            Map.of("status", item.getStatus(), "finalPrice", item.getCurrentPrice()));
    }

    private void notifyOutbid(User outbidUser, AuctionItem item) {
        emailService.sendOutbidNotification(outbidUser.getEmail(), item);
    }
}
```

### REST Controller

```java
// AuctionController.java
@RestController
@RequestMapping("/api/v1/auctions")
@RequiredArgsConstructor
public class AuctionController {

    private final AuctionService auctionService;

    @GetMapping
    public ResponseEntity<Page<AuctionItemDTO>> getLiveAuctions(
            @RequestParam(required = false) String category,
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size) {
        return ResponseEntity.ok(auctionService.getLiveAuctions(category, PageRequest.of(page, size)));
    }

    @GetMapping("/{id}")
    public ResponseEntity<AuctionItemDetailDTO> getAuction(@PathVariable Long id) {
        return ResponseEntity.ok(auctionService.getAuctionDetail(id));
    }

    @PostMapping
    @PreAuthorize("hasRole('SELLER') or hasRole('ADMIN')")
    public ResponseEntity<AuctionItemDTO> createAuction(
            @Valid @RequestBody CreateAuctionRequest request,
            @AuthenticationPrincipal UserPrincipal user) {
        return ResponseEntity.status(HttpStatus.CREATED)
            .body(auctionService.createAuction(request, user.getId()));
    }

    @PostMapping("/{id}/bids")
    public ResponseEntity<BidResult> placeBid(
            @PathVariable Long id,
            @AuthenticationPrincipal UserPrincipal user,
            @Valid @RequestBody PlaceBidRequest request) {
        return ResponseEntity.ok(auctionService.placeBid(id, user.getId(), request.getBidAmount()));
    }

    @PostMapping("/{id}/auto-bid")
    public ResponseEntity<Void> setAutoBid(
            @PathVariable Long id,
            @AuthenticationPrincipal UserPrincipal user,
            @Valid @RequestBody SetAutoBidRequest request) {
        auctionService.setAutoBid(id, user.getId(), request.getMaxAmount());
        return ResponseEntity.ok().build();
    }

    @GetMapping("/{id}/bids")
    public ResponseEntity<Page<BidDTO>> getBidHistory(
            @PathVariable Long id,
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size) {
        return ResponseEntity.ok(auctionService.getBidHistory(id, PageRequest.of(page, size)));
    }

    @GetMapping("/my-bids")
    public ResponseEntity<Page<BidDTO>> getMyBids(
            @AuthenticationPrincipal UserPrincipal user,
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size) {
        return ResponseEntity.ok(auctionService.getMyBids(user.getId(), PageRequest.of(page, size)));
    }
}
```

---

## โปรเจคที่ 22: Appointment Booking System

### ภาพรวมระบบ

ระบบจองนัดหมายที่รองรับผู้ให้บริการ บริการ ช่วงเวลา การจอง การแจ้งเตือน การยกเลิก และปฏิทินความพร้อม ใช้งานได้กับธุรกิจที่ต้องการระบบนัดหมาย เช่น คลินิก ร้านเสริมสวย และที่ปรึกษา

### Entity Classes

```java
// ServiceProvider.java
@Entity
@Table(name = "service_providers")
@Data
@NoArgsConstructor
public class ServiceProvider {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String businessName;
    private String description;
    private String category; // BEAUTY, MEDICAL, CONSULTING, FITNESS, etc.
    private String address;
    private String phone;
    private String email;
    private String website;
    private String logoUrl;

    private Double averageRating = 0.0;
    private Integer reviewCount = 0;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "owner_id")
    private User owner;

    @OneToMany(mappedBy = "provider", cascade = CascadeType.ALL)
    private List<ServiceOffering> services = new ArrayList<>();

    @OneToMany(mappedBy = "provider", cascade = CascadeType.ALL)
    private List<ProviderSchedule> schedules = new ArrayList<>();

    private Boolean active = true;
}

// ServiceOffering.java
@Entity
@Table(name = "service_offerings")
@Data
@NoArgsConstructor
public class ServiceOffering {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "provider_id")
    private ServiceProvider provider;

    private String name;
    private String description;
    private Integer durationMinutes;

    @Column(precision = 10, scale = 2)
    private BigDecimal price;

    private String category;
    private Boolean active = true;
    private String imageUrl;
}

// ProviderSchedule.java
@Entity
@Table(name = "provider_schedules")
@Data
@NoArgsConstructor
public class ProviderSchedule {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "provider_id")
    private ServiceProvider provider;

    private Integer dayOfWeek; // 1=Monday, 7=Sunday
    private LocalTime startTime;
    private LocalTime endTime;
    private Boolean isWorking = true;
    private Integer slotDurationMinutes = 30;
}

// TimeSlot.java
@Entity
@Table(name = "time_slots")
@Data
@NoArgsConstructor
public class TimeSlot {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "provider_id")
    private ServiceProvider provider;

    private LocalDate slotDate;
    private LocalTime startTime;
    private LocalTime endTime;

    @Enumerated(EnumType.STRING)
    private SlotStatus status = SlotStatus.AVAILABLE;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "booking_id")
    private Appointment booking;
}

// Appointment.java
@Entity
@Table(name = "appointments")
@Data
@NoArgsConstructor
public class Appointment {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(unique = true)
    private String confirmationCode;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "provider_id")
    private ServiceProvider provider;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "service_id")
    private ServiceOffering service;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "customer_id")
    private User customer;

    private LocalDate appointmentDate;
    private LocalTime startTime;
    private LocalTime endTime;

    @Enumerated(EnumType.STRING)
    private AppointmentStatus status = AppointmentStatus.CONFIRMED;

    @Column(precision = 10, scale = 2)
    private BigDecimal price;

    private String customerNotes;
    private String internalNotes;
    private String cancellationReason;

    private Boolean reminderSent = false;

    @CreatedDate
    private LocalDateTime createdAt;
}

// BlockedTime.java (for unavailability)
@Entity
@Table(name = "blocked_times")
@Data
@NoArgsConstructor
public class BlockedTime {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne
    @JoinColumn(name = "provider_id")
    private ServiceProvider provider;

    private LocalDate blockedDate;
    private LocalTime startTime;
    private LocalTime endTime;
    private Boolean allDay = false;
    private String reason;
}

public enum SlotStatus { AVAILABLE, BOOKED, BLOCKED }
public enum AppointmentStatus { PENDING, CONFIRMED, COMPLETED, CANCELLED, NO_SHOW }
```

### Service Layer

```java
// AppointmentService.java
@Service
@Transactional
@RequiredArgsConstructor
@Slf4j
public class AppointmentService {

    private final AppointmentRepository appointmentRepository;
    private final ServiceProviderRepository providerRepository;
    private final ServiceOfferingRepository serviceRepository;
    private final BlockedTimeRepository blockedTimeRepository;
    private final EmailService emailService;

    public List<AvailableSlot> getAvailableSlots(Long providerId, Long serviceId, LocalDate date) {
        ServiceProvider provider = providerRepository.findById(providerId)
            .orElseThrow(() -> new ResourceNotFoundException("Provider not found"));

        ServiceOffering service = serviceRepository.findById(serviceId)
            .orElseThrow(() -> new ResourceNotFoundException("Service not found"));

        // Get provider schedule for that day
        ProviderSchedule schedule = getScheduleForDay(provider, date.getDayOfWeek().getValue());
        if (schedule == null || !schedule.getIsWorking()) {
            return Collections.emptyList();
        }

        // Get existing appointments
        List<Appointment> existingBookings = appointmentRepository
            .findByProviderIdAndDateAndStatusNot(providerId, date, AppointmentStatus.CANCELLED);

        // Get blocked times
        List<BlockedTime> blockedTimes = blockedTimeRepository
            .findByProviderIdAndDate(providerId, date);

        // Generate slots
        List<AvailableSlot> slots = generateSlots(schedule, service.getDurationMinutes());

        // Remove booked/blocked slots
        return slots.stream()
            .filter(slot -> !isSlotTaken(slot, existingBookings, service.getDurationMinutes()))
            .filter(slot -> !isSlotBlocked(slot, blockedTimes))
            .filter(slot -> slot.getStartTime().isAfter(LocalTime.now()) || !date.equals(LocalDate.now()))
            .collect(Collectors.toList());
    }

    public AppointmentDTO bookAppointment(BookAppointmentRequest request, Long customerId) {
        ServiceProvider provider = providerRepository.findById(request.getProviderId())
            .orElseThrow(() -> new ResourceNotFoundException("Provider not found"));

        ServiceOffering service = serviceRepository.findById(request.getServiceId())
            .orElseThrow(() -> new ResourceNotFoundException("Service not found"));

        // Check availability
        boolean isAvailable = checkSlotAvailability(
            provider.getId(), request.getDate(),
            request.getStartTime(), service.getDurationMinutes());

        if (!isAvailable) {
            throw new ConflictException("This time slot is no longer available");
        }

        Appointment appointment = new Appointment();
        appointment.setConfirmationCode(generateConfirmationCode());
        appointment.setProvider(provider);
        appointment.setService(service);
        User customer = new User();
        customer.setId(customerId);
        appointment.setCustomer(customer);
        appointment.setAppointmentDate(request.getDate());
        appointment.setStartTime(request.getStartTime());
        appointment.setEndTime(request.getStartTime().plusMinutes(service.getDurationMinutes()));
        appointment.setPrice(service.getPrice());
        appointment.setCustomerNotes(request.getNotes());

        Appointment saved = appointmentRepository.save(appointment);

        // Send confirmation
        emailService.sendAppointmentConfirmation(customer.getEmail(), saved);

        // Schedule reminder
        scheduleReminder(saved);

        return toDTO(saved);
    }

    public AppointmentDTO cancelAppointment(Long appointmentId, Long userId, String reason) {
        Appointment appointment = appointmentRepository.findById(appointmentId)
            .orElseThrow(() -> new ResourceNotFoundException("Appointment not found"));

        boolean isCustomer = appointment.getCustomer().getId().equals(userId);
        boolean isProvider = appointment.getProvider().getOwner().getId().equals(userId);

        if (!isCustomer && !isProvider) {
            throw new ForbiddenException("Not authorized to cancel this appointment");
        }

        if (appointment.getStatus() == AppointmentStatus.COMPLETED ||
            appointment.getStatus() == AppointmentStatus.CANCELLED) {
            throw new BadRequestException("Cannot cancel this appointment");
        }

        appointment.setStatus(AppointmentStatus.CANCELLED);
        appointment.setCancellationReason(reason);

        // Send notification
        emailService.sendCancellationNotice(appointment.getCustomer().getEmail(), appointment);

        return toDTO(appointmentRepository.save(appointment));
    }

    @Scheduled(cron = "0 0 * * * *") // Every hour
    public void sendReminders() {
        LocalDateTime reminderTime = LocalDateTime.now().plusHours(24);
        LocalDate reminderDate = reminderTime.toLocalDate();
        LocalTime startOfHour = reminderTime.toLocalTime().withMinute(0).withSecond(0);
        LocalTime endOfHour = startOfHour.plusHours(1);

        List<Appointment> upcoming = appointmentRepository.findUpcomingForReminder(
            reminderDate, startOfHour, endOfHour);

        for (Appointment apt : upcoming) {
            emailService.sendAppointmentReminder(apt.getCustomer().getEmail(), apt);
            apt.setReminderSent(true);
            appointmentRepository.save(apt);
        }
    }

    private List<AvailableSlot> generateSlots(ProviderSchedule schedule, int serviceDuration) {
        List<AvailableSlot> slots = new ArrayList<>();
        LocalTime current = schedule.getStartTime();

        while (current.plusMinutes(serviceDuration).compareTo(schedule.getEndTime()) <= 0) {
            slots.add(new AvailableSlot(current, current.plusMinutes(serviceDuration)));
            current = current.plusMinutes(schedule.getSlotDurationMinutes());
        }
        return slots;
    }

    private boolean isSlotTaken(AvailableSlot slot, List<Appointment> bookings, int duration) {
        return bookings.stream().anyMatch(b ->
            b.getStartTime().compareTo(slot.getStartTime()) < 0 && 
                b.getEndTime().compareTo(slot.getStartTime()) > 0 ||
            b.getStartTime().compareTo(slot.getStartTime()) >= 0 && 
                b.getStartTime().compareTo(slot.getEndTime()) < 0
        );
    }

    private boolean isSlotBlocked(AvailableSlot slot, List<BlockedTime> blocked) {
        return blocked.stream().anyMatch(b ->
            b.isAllDay() ||
            (b.getStartTime().compareTo(slot.getStartTime()) <= 0 &&
             b.getEndTime().compareTo(slot.getStartTime()) > 0)
        );
    }

    private boolean checkSlotAvailability(Long providerId, LocalDate date, LocalTime time, int duration) {
        List<Appointment> conflicts = appointmentRepository.findConflicts(
            providerId, date, time, time.plusMinutes(duration));
        return conflicts.isEmpty();
    }

    private void scheduleReminder(Appointment appointment) {
        // In production: use a scheduler or message queue
        log.info("Reminder scheduled for appointment: {}", appointment.getConfirmationCode());
    }

    private String generateConfirmationCode() {
        return "APT-" + UUID.randomUUID().toString().substring(0, 8).toUpperCase();
    }

    private ProviderSchedule getScheduleForDay(ServiceProvider provider, int dayOfWeek) {
        return provider.getSchedules().stream()
            .filter(s -> s.getDayOfWeek() == dayOfWeek)
            .findFirst().orElse(null);
    }

    private AppointmentDTO toDTO(Appointment a) {
        return AppointmentDTO.builder()
            .id(a.getId())
            .confirmationCode(a.getConfirmationCode())
            .providerName(a.getProvider().getBusinessName())
            .serviceName(a.getService().getName())
            .appointmentDate(a.getAppointmentDate())
            .startTime(a.getStartTime())
            .endTime(a.getEndTime())
            .status(a.getStatus())
            .price(a.getPrice())
            .build();
    }
}
```

---

## โปรเจคที่ 23: Survey & Poll Platform

### ภาพรวมระบบ

แพลตฟอร์มสร้างแบบสำรวจและโพลที่รองรับประเภทคำถามหลายแบบ (เลือกเดียว/หลายตัว/คะแนน/ข้อความ) การเก็บคำตอบ การวิเคราะห์ผล และการตั้งค่าแบบสำรวจ Public/Private

### Entity Classes

```java
// Survey.java
@Entity
@Table(name = "surveys")
@Data
@NoArgsConstructor
public class Survey {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String title;

    private String description;
    private String welcomeMessage;
    private String thankYouMessage;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "creator_id")
    private User creator;

    @Enumerated(EnumType.STRING)
    private SurveyStatus status = SurveyStatus.DRAFT;

    private Boolean isPublic = false;
    private Boolean isAnonymous = false;
    private Boolean allowMultipleSubmissions = false;

    private LocalDateTime startDate;
    private LocalDateTime endDate;

    private String accessCode; // For private surveys
    private Integer responseLimit;
    private Integer responseCount = 0;

    @OneToMany(mappedBy = "survey", cascade = CascadeType.ALL)
    @OrderBy("displayOrder ASC")
    private List<SurveyQuestion> questions = new ArrayList<>();

    @CreatedDate
    private LocalDateTime createdAt;
}

// SurveyQuestion.java
@Entity
@Table(name = "survey_questions")
@Data
@NoArgsConstructor
public class SurveyQuestion {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "survey_id")
    private Survey survey;

    private String questionText;

    @Enumerated(EnumType.STRING)
    private QuestionType type;

    private Boolean required = true;
    private Integer displayOrder;

    private String helpText;
    private Integer minRating; // For RATING type
    private Integer maxRating;

    @OneToMany(mappedBy = "question", cascade = CascadeType.ALL)
    @OrderBy("optionOrder ASC")
    private List<QuestionOption> options = new ArrayList<>();
}

// QuestionOption.java
@Entity
@Table(name = "question_options")
@Data
@NoArgsConstructor
public class QuestionOption {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne
    @JoinColumn(name = "question_id")
    private SurveyQuestion question;

    private String optionText;
    private Integer optionOrder;
    private String value; // Optional value mapping
}

// SurveyResponse.java
@Entity
@Table(name = "survey_responses")
@Data
@NoArgsConstructor
public class SurveyResponse {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "survey_id")
    private Survey survey;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "respondent_id")
    private User respondent; // Null if anonymous

    private String respondentEmail; // For tracking even if anonymous
    private String sessionId; // For anonymous tracking

    @OneToMany(mappedBy = "response", cascade = CascadeType.ALL)
    private List<QuestionAnswer> answers = new ArrayList<>();

    @Column(precision = 5, scale = 2)
    private BigDecimal completionPercentage = BigDecimal.ZERO;

    @Enumerated(EnumType.STRING)
    private ResponseStatus status = ResponseStatus.COMPLETED;

    @CreatedDate
    private LocalDateTime submittedAt;
}

// QuestionAnswer.java
@Entity
@Table(name = "question_answers")
@Data
@NoArgsConstructor
public class QuestionAnswer {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "response_id")
    private SurveyResponse response;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "question_id")
    private SurveyQuestion question;

    @Column(columnDefinition = "TEXT")
    private String textAnswer;

    private Integer numericAnswer;

    @ElementCollection
    @CollectionTable(name = "answer_selected_options")
    private List<Long> selectedOptionIds = new ArrayList<>();
}

public enum SurveyStatus { DRAFT, ACTIVE, CLOSED, ARCHIVED }
public enum QuestionType { SINGLE_CHOICE, MULTIPLE_CHOICE, RATING, TEXT, LONG_TEXT, DATE, SCALE }
public enum ResponseStatus { PARTIAL, COMPLETED }
```

### Service Layer

```java
// SurveyService.java
@Service
@Transactional
@RequiredArgsConstructor
public class SurveyService {

    private final SurveyRepository surveyRepository;
    private final SurveyResponseRepository responseRepository;
    private final QuestionAnswerRepository answerRepository;

    public SurveyDTO createSurvey(CreateSurveyRequest request, Long creatorId) {
        Survey survey = new Survey();
        survey.setTitle(request.getTitle());
        survey.setDescription(request.getDescription());
        survey.setWelcomeMessage(request.getWelcomeMessage());
        survey.setThankYouMessage(request.getThankYouMessage());
        User creator = new User();
        creator.setId(creatorId);
        survey.setCreator(creator);
        survey.setIsPublic(request.getIsPublic());
        survey.setIsAnonymous(request.getIsAnonymous());
        survey.setAllowMultipleSubmissions(request.getAllowMultipleSubmissions());
        survey.setStartDate(request.getStartDate());
        survey.setEndDate(request.getEndDate());
        survey.setResponseLimit(request.getResponseLimit());

        if (!request.getIsPublic()) {
            survey.setAccessCode(generateAccessCode());
        }

        int order = 0;
        for (QuestionRequest qReq : request.getQuestions()) {
            SurveyQuestion question = new SurveyQuestion();
            question.setSurvey(survey);
            question.setQuestionText(qReq.getQuestionText());
            question.setType(qReq.getType());
            question.setRequired(qReq.getRequired());
            question.setDisplayOrder(order++);
            question.setHelpText(qReq.getHelpText());
            question.setMinRating(qReq.getMinRating());
            question.setMaxRating(qReq.getMaxRating());

            if (qReq.getOptions() != null) {
                int optOrder = 0;
                for (String optText : qReq.getOptions()) {
                    QuestionOption opt = new QuestionOption();
                    opt.setQuestion(question);
                    opt.setOptionText(optText);
                    opt.setOptionOrder(optOrder++);
                    question.getOptions().add(opt);
                }
            }
            survey.getQuestions().add(question);
        }

        return toSurveyDTO(surveyRepository.save(survey));
    }

    public ResponseDTO submitResponse(Long surveyId, SubmitResponseRequest request, Long userId) {
        Survey survey = surveyRepository.findById(surveyId)
            .orElseThrow(() -> new ResourceNotFoundException("Survey not found"));

        validateSurveyAccess(survey, request.getAccessCode(), userId);

        if (!survey.getAllowMultipleSubmissions() && userId != null) {
            if (responseRepository.existsBySurveyIdAndRespondentId(surveyId, userId)) {
                throw new ConflictException("You have already submitted a response to this survey");
            }
        }

        if (survey.getResponseLimit() != null && survey.getResponseCount() >= survey.getResponseLimit()) {
            throw new ConflictException("Survey has reached its response limit");
        }

        SurveyResponse response = new SurveyResponse();
        response.setSurvey(survey);

        if (userId != null && !survey.getIsAnonymous()) {
            User respondent = new User();
            respondent.setId(userId);
            response.setRespondent(respondent);
        }
        response.setRespondentEmail(request.getRespondentEmail());

        // Validate required questions
        Map<Long, QuestionAnswerRequest> answerMap = request.getAnswers().stream()
            .collect(Collectors.toMap(QuestionAnswerRequest::getQuestionId, a -> a));

        for (SurveyQuestion question : survey.getQuestions()) {
            QuestionAnswerRequest answerReq = answerMap.get(question.getId());
            if (question.getRequired() && answerReq == null) {
                throw new BadRequestException("Question is required: " + question.getQuestionText());
            }

            if (answerReq != null) {
                QuestionAnswer answer = new QuestionAnswer();
                answer.setResponse(response);
                answer.setQuestion(question);
                answer.setTextAnswer(answerReq.getTextAnswer());
                answer.setNumericAnswer(answerReq.getNumericAnswer());
                answer.setSelectedOptionIds(answerReq.getSelectedOptionIds());
                response.getAnswers().add(answer);
            }
        }

        survey.setResponseCount(survey.getResponseCount() + 1);
        surveyRepository.save(survey);

        SurveyResponse saved = responseRepository.save(response);

        return ResponseDTO.builder()
            .id(saved.getId())
            .surveyTitle(survey.getTitle())
            .submittedAt(saved.getSubmittedAt())
            .thankYouMessage(survey.getThankYouMessage())
            .build();
    }

    public SurveyAnalytics getAnalytics(Long surveyId) {
        Survey survey = surveyRepository.findById(surveyId)
            .orElseThrow(() -> new ResourceNotFoundException("Survey not found"));

        List<SurveyResponse> responses = responseRepository.findBySurveyId(surveyId);

        Map<Long, QuestionAnalytics> questionAnalytics = new LinkedHashMap<>();

        for (SurveyQuestion question : survey.getQuestions()) {
            List<QuestionAnswer> answers = answerRepository.findByQuestionId(question.getId());
            QuestionAnalytics qAnalytics = analyzeQuestion(question, answers);
            questionAnalytics.put(question.getId(), qAnalytics);
        }

        return SurveyAnalytics.builder()
            .surveyId(surveyId)
            .surveyTitle(survey.getTitle())
            .totalResponses(responses.size())
            .completionRate(calculateCompletionRate(responses))
            .questionAnalytics(questionAnalytics)
            .build();
    }

    private QuestionAnalytics analyzeQuestion(SurveyQuestion question, List<QuestionAnswer> answers) {
        QuestionAnalytics analytics = new QuestionAnalytics();
        analytics.setQuestionId(question.getId());
        analytics.setQuestionText(question.getQuestionText());
        analytics.setType(question.getType());
        analytics.setResponseCount(answers.size());

        switch (question.getType()) {
            case SINGLE_CHOICE, MULTIPLE_CHOICE -> {
                Map<Long, Long> optionCounts = answers.stream()
                    .flatMap(a -> a.getSelectedOptionIds().stream())
                    .collect(Collectors.groupingBy(id -> id, Collectors.counting()));

                Map<String, Long> labeledCounts = new LinkedHashMap<>();
                for (QuestionOption opt : question.getOptions()) {
                    labeledCounts.put(opt.getOptionText(), optionCounts.getOrDefault(opt.getId(), 0L));
                }
                analytics.setOptionCounts(labeledCounts);
            }
            case RATING -> {
                OptionalDouble avg = answers.stream()
                    .filter(a -> a.getNumericAnswer() != null)
                    .mapToInt(QuestionAnswer::getNumericAnswer)
                    .average();
                analytics.setAverageRating(avg.orElse(0.0));

                Map<Integer, Long> ratingDistribution = answers.stream()
                    .filter(a -> a.getNumericAnswer() != null)
                    .collect(Collectors.groupingBy(QuestionAnswer::getNumericAnswer, Collectors.counting()));
                analytics.setRatingDistribution(ratingDistribution);
            }
            case TEXT, LONG_TEXT -> {
                List<String> textResponses = answers.stream()
                    .filter(a -> a.getTextAnswer() != null && !a.getTextAnswer().isBlank())
                    .map(QuestionAnswer::getTextAnswer)
                    .limit(50)
                    .collect(Collectors.toList());
                analytics.setTextResponses(textResponses);
            }
        }

        return analytics;
    }

    private void validateSurveyAccess(Survey survey, String accessCode, Long userId) {
        if (survey.getStatus() != SurveyStatus.ACTIVE) {
            throw new BadRequestException("Survey is not active");
        }
        if (survey.getEndDate() != null && LocalDateTime.now().isAfter(survey.getEndDate())) {
            throw new BadRequestException("Survey has ended");
        }
        if (!survey.getIsPublic() && (accessCode == null || !accessCode.equals(survey.getAccessCode()))) {
            throw new ForbiddenException("Invalid access code for this survey");
        }
    }

    private double calculateCompletionRate(List<SurveyResponse> responses) {
        if (responses.isEmpty()) return 0.0;
        long completed = responses.stream()
            .filter(r -> r.getStatus() == ResponseStatus.COMPLETED).count();
        return (double) completed / responses.size() * 100;
    }

    private String generateAccessCode() {
        return String.format("%06d", (int)(Math.random() * 1000000));
    }

    private SurveyDTO toSurveyDTO(Survey s) {
        return SurveyDTO.builder()
            .id(s.getId())
            .title(s.getTitle())
            .status(s.getStatus())
            .isPublic(s.getIsPublic())
            .responseCount(s.getResponseCount())
            .accessCode(!s.getIsPublic() ? s.getAccessCode() : null)
            .questionCount(s.getQuestions().size())
            .createdAt(s.getCreatedAt())
            .build();
    }
}
```

---

## โปรเจคที่ 24: Review & Rating System

### ภาพรวมระบบ

ระบบรีวิวและให้คะแนนที่รองรับการรีวิวสินค้า/บริการ ดาวให้คะแนน ข้อดีข้อเสีย ป้ายการซื้อจริง การโหวตว่าเป็นประโยชน์ และการรายงานเนื้อหาที่ไม่เหมาะสม

### Entity Classes

```java
// Review.java
@Entity
@Table(name = "reviews")
@Data
@NoArgsConstructor
public class Review {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "reviewer_id")
    private User reviewer;

    private String targetType; // PRODUCT, SERVICE, PLACE, etc.
    private Long targetId;

    private Integer rating; // 1-5

    private String title;

    @Column(columnDefinition = "TEXT")
    private String content;

    @ElementCollection
    @CollectionTable(name = "review_pros")
    private List<String> pros = new ArrayList<>();

    @ElementCollection
    @CollectionTable(name = "review_cons")
    private List<String> cons = new ArrayList<>();

    @ElementCollection
    @CollectionTable(name = "review_images")
    private List<String> imageUrls = new ArrayList<>();

    private Boolean verifiedPurchase = false;

    @Enumerated(EnumType.STRING)
    private ReviewStatus status = ReviewStatus.PUBLISHED;

    private Integer helpfulVotes = 0;
    private Integer notHelpfulVotes = 0;
    private Integer reportCount = 0;

    @CreatedDate
    private LocalDateTime createdAt;

    @LastModifiedDate
    private LocalDateTime updatedAt;
}

// ReviewHelpfulVote.java
@Entity
@Table(name = "review_helpful_votes",
       uniqueConstraints = @UniqueConstraint(columnNames = {"review_id", "user_id"}))
@Data
@NoArgsConstructor
public class ReviewHelpfulVote {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne
    @JoinColumn(name = "review_id")
    private Review review;

    @ManyToOne
    @JoinColumn(name = "user_id")
    private User user;

    private Boolean helpful;
}

// ReviewReport.java
@Entity
@Table(name = "review_reports")
@Data
@NoArgsConstructor
public class ReviewReport {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne
    @JoinColumn(name = "review_id")
    private Review review;

    @ManyToOne
    @JoinColumn(name = "reporter_id")
    private User reporter;

    private String reason; // SPAM, FAKE, INAPPROPRIATE, etc.
    private String description;

    @Enumerated(EnumType.STRING)
    private ReportStatus status = ReportStatus.PENDING;

    @CreatedDate
    private LocalDateTime reportedAt;
}

public enum ReviewStatus { PENDING, PUBLISHED, HIDDEN, DELETED }
public enum ReportStatus { PENDING, REVIEWED, DISMISSED, ACTION_TAKEN }
```

### Service Layer

```java
// ReviewService.java
@Service
@Transactional
@RequiredArgsConstructor
public class ReviewService {

    private final ReviewRepository reviewRepository;
    private final ReviewHelpfulVoteRepository voteRepository;
    private final ReviewReportRepository reportRepository;

    public ReviewDTO createReview(CreateReviewRequest request, Long reviewerId) {
        // Prevent duplicate reviews
        if (reviewRepository.existsByReviewerIdAndTargetTypeAndTargetId(
                reviewerId, request.getTargetType(), request.getTargetId())) {
            throw new ConflictException("You have already reviewed this item");
        }

        Review review = new Review();
        User reviewer = new User();
        reviewer.setId(reviewerId);
        review.setReviewer(reviewer);
        review.setTargetType(request.getTargetType());
        review.setTargetId(request.getTargetId());
        review.setRating(request.getRating());
        review.setTitle(request.getTitle());
        review.setContent(request.getContent());
        review.setPros(request.getPros() != null ? request.getPros() : new ArrayList<>());
        review.setCons(request.getCons() != null ? request.getCons() : new ArrayList<>());
        review.setImageUrls(request.getImageUrls() != null ? request.getImageUrls() : new ArrayList<>());

        // Check if verified purchase (integrate with OrderService)
        boolean isVerified = checkVerifiedPurchase(reviewerId, request.getTargetType(), request.getTargetId());
        review.setVerifiedPurchase(isVerified);

        Review saved = reviewRepository.save(review);
        updateTargetRating(request.getTargetType(), request.getTargetId());

        return toDTO(saved);
    }

    public Page<ReviewDTO> getReviews(String targetType, Long targetId,
                                       Integer minRating, String sortBy, Pageable pageable) {
        Page<Review> reviews;

        if (minRating != null) {
            reviews = reviewRepository.findByTargetAndMinRating(targetType, targetId, minRating, pageable);
        } else if ("HELPFUL".equals(sortBy)) {
            reviews = reviewRepository.findByTargetOrderByHelpful(targetType, targetId, pageable);
        } else {
            reviews = reviewRepository.findByTargetTypeAndTargetIdAndStatus(
                targetType, targetId, ReviewStatus.PUBLISHED, pageable);
        }

        return reviews.map(this::toDTO);
    }

    public RatingSummary getRatingSummary(String targetType, Long targetId) {
        List<Review> reviews = reviewRepository.findByTargetTypeAndTargetIdAndStatus(
            targetType, targetId, ReviewStatus.PUBLISHED);

        if (reviews.isEmpty()) {
            return RatingSummary.builder().averageRating(0.0).totalReviews(0).build();
        }

        double avg = reviews.stream().mapToInt(Review::getRating).average().orElse(0.0);

        Map<Integer, Long> distribution = reviews.stream()
            .collect(Collectors.groupingBy(Review::getRating, Collectors.counting()));

        long verifiedCount = reviews.stream().filter(Review::getVerifiedPurchase).count();

        return RatingSummary.builder()
            .averageRating(Math.round(avg * 10.0) / 10.0)
            .totalReviews(reviews.size())
            .verifiedReviews((int) verifiedCount)
            .ratingDistribution(distribution)
            .fiveStar((int) distribution.getOrDefault(5, 0L))
            .fourStar((int) distribution.getOrDefault(4, 0L))
            .threeStar((int) distribution.getOrDefault(3, 0L))
            .twoStar((int) distribution.getOrDefault(2, 0L))
            .oneStar((int) distribution.getOrDefault(1, 0L))
            .build();
    }

    public void markHelpful(Long reviewId, Long userId, boolean helpful) {
        Review review = reviewRepository.findById(reviewId)
            .orElseThrow(() -> new ResourceNotFoundException("Review not found"));

        ReviewHelpfulVote existing = voteRepository.findByReviewIdAndUserId(reviewId, userId).orElse(null);

        if (existing != null) {
            if (existing.getHelpful() != helpful) {
                // Change vote
                if (existing.getHelpful()) {
                    review.setHelpfulVotes(review.getHelpfulVotes() - 1);
                    review.setNotHelpfulVotes(review.getNotHelpfulVotes() + 1);
                } else {
                    review.setHelpfulVotes(review.getHelpfulVotes() + 1);
                    review.setNotHelpfulVotes(review.getNotHelpfulVotes() - 1);
                }
                existing.setHelpful(helpful);
                voteRepository.save(existing);
            }
        } else {
            ReviewHelpfulVote vote = new ReviewHelpfulVote();
            vote.setReview(review);
            User user = new User();
            user.setId(userId);
            vote.setUser(user);
            vote.setHelpful(helpful);
            voteRepository.save(vote);

            if (helpful) review.setHelpfulVotes(review.getHelpfulVotes() + 1);
            else review.setNotHelpfulVotes(review.getNotHelpfulVotes() + 1);
        }

        reviewRepository.save(review);
    }

    public ReviewReportDTO reportReview(Long reviewId, Long reporterId, String reason, String description) {
        Review review = reviewRepository.findById(reviewId)
            .orElseThrow(() -> new ResourceNotFoundException("Review not found"));

        ReviewReport report = new ReviewReport();
        report.setReview(review);
        User reporter = new User();
        reporter.setId(reporterId);
        report.setReporter(reporter);
        report.setReason(reason);
        report.setDescription(description);
        reportRepository.save(report);

        review.setReportCount(review.getReportCount() + 1);

        // Auto-hide if too many reports
        if (review.getReportCount() >= 5) {
            review.setStatus(ReviewStatus.HIDDEN);
        }

        reviewRepository.save(review);

        return ReviewReportDTO.builder()
            .reportId(report.getId())
            .reviewId(reviewId)
            .status(report.getStatus())
            .build();
    }

    private boolean checkVerifiedPurchase(Long userId, String targetType, Long targetId) {
        // In production, integrate with OrderService to verify purchase
        return false;
    }

    private void updateTargetRating(String targetType, Long targetId) {
        // Update rating in the target entity (Product, Service, etc.)
        // This would typically call the relevant service
    }

    private ReviewDTO toDTO(Review r) {
        return ReviewDTO.builder()
            .id(r.getId())
            .reviewerName(r.getReviewer().getUsername())
            .rating(r.getRating())
            .title(r.getTitle())
            .content(r.getContent())
            .pros(r.getPros())
            .cons(r.getCons())
            .imageUrls(r.getImageUrls())
            .verifiedPurchase(r.getVerifiedPurchase())
            .helpfulVotes(r.getHelpfulVotes())
            .createdAt(r.getCreatedAt())
            .build();
    }
}
```

### REST Controller

```java
// ReviewController.java
@RestController
@RequestMapping("/api/v1/reviews")
@RequiredArgsConstructor
public class ReviewController {

    private final ReviewService reviewService;

    @GetMapping
    public ResponseEntity<Page<ReviewDTO>> getReviews(
            @RequestParam String targetType,
            @RequestParam Long targetId,
            @RequestParam(required = false) Integer minRating,
            @RequestParam(required = false, defaultValue = "RECENT") String sortBy,
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "10") int size) {
        return ResponseEntity.ok(reviewService.getReviews(targetType, targetId, minRating, sortBy,
                PageRequest.of(page, size)));
    }

    @GetMapping("/summary")
    public ResponseEntity<RatingSummary> getRatingSummary(
            @RequestParam String targetType,
            @RequestParam Long targetId) {
        return ResponseEntity.ok(reviewService.getRatingSummary(targetType, targetId));
    }

    @PostMapping
    public ResponseEntity<ReviewDTO> createReview(
            @Valid @RequestBody CreateReviewRequest request,
            @AuthenticationPrincipal UserPrincipal user) {
        return ResponseEntity.status(HttpStatus.CREATED).body(reviewService.createReview(request, user.getId()));
    }

    @PostMapping("/{id}/helpful")
    public ResponseEntity<Void> markHelpful(
            @PathVariable Long id,
            @RequestParam boolean helpful,
            @AuthenticationPrincipal UserPrincipal user) {
        reviewService.markHelpful(id, user.getId(), helpful);
        return ResponseEntity.ok().build();
    }

    @PostMapping("/{id}/report")
    public ResponseEntity<ReviewReportDTO> reportReview(
            @PathVariable Long id,
            @Valid @RequestBody ReportReviewRequest request,
            @AuthenticationPrincipal UserPrincipal user) {
        return ResponseEntity.ok(reviewService.reportReview(id, user.getId(),
                request.getReason(), request.getDescription()));
    }
}
```

---

## โปรเจคที่ 25: Support Ticket System

### ภาพรวมระบบ

ระบบ Support Ticket ที่รองรับการสร้างตั๋ว หมวดหมู่ ระดับความสำคัญ การมอบหมายงาน การติดตาม SLA ความคิดเห็น เวลาแก้ไข และรายงานประสิทธิภาพของทีม

### Entity Classes

```java
// Ticket.java
@Entity
@Table(name = "tickets")
@Data
@NoArgsConstructor
public class Ticket {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(unique = true, nullable = false)
    private String ticketNumber;

    @Column(nullable = false)
    private String subject;

    @Column(columnDefinition = "TEXT")
    private String description;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "reporter_id")
    private User reporter;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "category_id")
    private TicketCategory category;

    @Enumerated(EnumType.STRING)
    private TicketPriority priority = TicketPriority.MEDIUM;

    @Enumerated(EnumType.STRING)
    private TicketStatus status = TicketStatus.OPEN;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "assigned_to")
    private User assignedTo;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "assigned_team_id")
    private SupportTeam team;

    private LocalDateTime firstResponseAt;
    private LocalDateTime resolvedAt;
    private LocalDateTime closedAt;
    private LocalDateTime dueDate; // SLA deadline

    private Integer customerRating; // 1-5 CSAT
    private String customerFeedback;

    @ElementCollection
    @CollectionTable(name = "ticket_attachments")
    private List<String> attachmentUrls = new ArrayList<>();

    @OneToMany(mappedBy = "ticket", cascade = CascadeType.ALL)
    @OrderBy("createdAt ASC")
    private List<TicketComment> comments = new ArrayList<>();

    @CreatedDate
    private LocalDateTime createdAt;

    @LastModifiedDate
    private LocalDateTime updatedAt;
}

// TicketCategory.java
@Entity
@Table(name = "ticket_categories")
@Data
@NoArgsConstructor
public class TicketCategory {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;
    private String description;
    private Integer slaHours; // Target resolution time
    private String defaultAssignedTeam;
}

// TicketComment.java
@Entity
@Table(name = "ticket_comments")
@Data
@NoArgsConstructor
public class TicketComment {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "ticket_id")
    private Ticket ticket;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "author_id")
    private User author;

    @Column(columnDefinition = "TEXT")
    private String content;

    private Boolean isInternal = false; // Internal notes not visible to customer

    @ElementCollection
    @CollectionTable(name = "comment_attachments")
    private List<String> attachmentUrls = new ArrayList<>();

    @CreatedDate
    private LocalDateTime createdAt;
}

// SupportTeam.java
@Entity
@Table(name = "support_teams")
@Data
@NoArgsConstructor
public class SupportTeam {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;
    private String description;

    @ManyToMany
    @JoinTable(name = "team_members",
               joinColumns = @JoinColumn(name = "team_id"),
               inverseJoinColumns = @JoinColumn(name = "user_id"))
    private List<User> members = new ArrayList<>();
}

// SLAPolicy.java
@Entity
@Table(name = "sla_policies")
@Data
@NoArgsConstructor
public class SLAPolicy {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    @Enumerated(EnumType.STRING)
    private TicketPriority priority;

    private Integer firstResponseHours;
    private Integer resolutionHours;

    private Boolean active = true;
}

public enum TicketPriority { LOW, MEDIUM, HIGH, URGENT }
public enum TicketStatus { OPEN, IN_PROGRESS, WAITING_FOR_CUSTOMER, RESOLVED, CLOSED }
```

### Service Layer

```java
// TicketService.java
@Service
@Transactional
@RequiredArgsConstructor
@Slf4j
public class TicketService {

    private final TicketRepository ticketRepository;
    private final TicketCategoryRepository categoryRepository;
    private final TicketCommentRepository commentRepository;
    private final SLAPolicyRepository slaPolicyRepository;
    private final EmailService emailService;

    public TicketDTO createTicket(CreateTicketRequest request, Long reporterId) {
        TicketCategory category = categoryRepository.findById(request.getCategoryId())
            .orElseThrow(() -> new ResourceNotFoundException("Category not found"));

        // Calculate SLA deadline
        SLAPolicy sla = slaPolicyRepository.findByPriorityAndActive(request.getPriority(), true)
            .orElse(null);

        Ticket ticket = new Ticket();
        ticket.setTicketNumber(generateTicketNumber());
        ticket.setSubject(request.getSubject());
        ticket.setDescription(request.getDescription());
        User reporter = new User();
        reporter.setId(reporterId);
        ticket.setReporter(reporter);
        ticket.setCategory(category);
        ticket.setPriority(request.getPriority());
        ticket.setAttachmentUrls(request.getAttachmentUrls() != null ? request.getAttachmentUrls() : new ArrayList<>());

        if (sla != null) {
            ticket.setDueDate(LocalDateTime.now().plusHours(sla.getResolutionHours()));
        }

        Ticket saved = ticketRepository.save(ticket);

        // Auto-assign based on category
        autoAssign(saved, category);

        // Send confirmation to reporter
        emailService.sendTicketConfirmation(reporter.getEmail(), saved);

        return toDTO(saved);
    }

    public TicketCommentDTO addComment(Long ticketId, Long authorId, 
                                        String content, boolean internal, List<String> attachments) {
        Ticket ticket = ticketRepository.findById(ticketId)
            .orElseThrow(() -> new ResourceNotFoundException("Ticket not found"));

        if (ticket.getStatus() == TicketStatus.CLOSED) {
            throw new BadRequestException("Cannot comment on closed ticket. Please create a new ticket.");
        }

        TicketComment comment = new TicketComment();
        comment.setTicket(ticket);
        User author = new User();
        author.setId(authorId);
        comment.setAuthor(author);
        comment.setContent(content);
        comment.setIsInternal(internal);
        comment.setAttachmentUrls(attachments != null ? attachments : new ArrayList<>());

        // Track first response time
        boolean isStaff = isStaff(authorId);
        if (isStaff && ticket.getFirstResponseAt() == null) {
            ticket.setFirstResponseAt(LocalDateTime.now());
        }

        // Auto-update status if customer replies to waiting ticket
        if (!isStaff && ticket.getStatus() == TicketStatus.WAITING_FOR_CUSTOMER) {
            ticket.setStatus(TicketStatus.IN_PROGRESS);
        }

        ticketRepository.save(ticket);
        TicketComment saved = commentRepository.save(comment);

        // Notify relevant parties
        if (isStaff && !internal) {
            emailService.notifyCustomerNewComment(ticket.getReporter().getEmail(), ticket, saved);
        } else if (!isStaff) {
            if (ticket.getAssignedTo() != null) {
                emailService.notifyAgentNewComment(ticket.getAssignedTo().getEmail(), ticket, saved);
            }
        }

        return toCommentDTO(saved);
    }

    public TicketDTO updateTicketStatus(Long ticketId, TicketStatus status, Long agentId) {
        Ticket ticket = ticketRepository.findById(ticketId)
            .orElseThrow(() -> new ResourceNotFoundException("Ticket not found"));

        TicketStatus oldStatus = ticket.getStatus();
        ticket.setStatus(status);

        if (status == TicketStatus.RESOLVED) {
            ticket.setResolvedAt(LocalDateTime.now());
        } else if (status == TicketStatus.CLOSED) {
            ticket.setClosedAt(LocalDateTime.now());
        }

        Ticket saved = ticketRepository.save(ticket);

        // Notify customer
        emailService.notifyStatusChange(ticket.getReporter().getEmail(), ticket, oldStatus, status);

        return toDTO(saved);
    }

    public TicketDTO assignTicket(Long ticketId, Long assignToId, Long teamId) {
        Ticket ticket = ticketRepository.findById(ticketId)
            .orElseThrow(() -> new ResourceNotFoundException("Ticket not found"));

        if (assignToId != null) {
            User assignTo = new User();
            assignTo.setId(assignToId);
            ticket.setAssignedTo(assignTo);
        }

        if (teamId != null) {
            SupportTeam team = new SupportTeam();
            team.setId(teamId);
            ticket.setTeam(team);
        }

        if (ticket.getStatus() == TicketStatus.OPEN) {
            ticket.setStatus(TicketStatus.IN_PROGRESS);
        }

        return toDTO(ticketRepository.save(ticket));
    }

    public TicketDTO submitCsat(Long ticketId, Long customerId, int rating, String feedback) {
        Ticket ticket = ticketRepository.findById(ticketId)
            .orElseThrow(() -> new ResourceNotFoundException("Ticket not found"));

        if (!ticket.getReporter().getId().equals(customerId)) {
            throw new ForbiddenException("Only the ticket reporter can submit CSAT");
        }

        if (ticket.getStatus() != TicketStatus.RESOLVED && ticket.getStatus() != TicketStatus.CLOSED) {
            throw new BadRequestException("Can only rate resolved or closed tickets");
        }

        if (rating < 1 || rating > 5) {
            throw new BadRequestException("Rating must be between 1 and 5");
        }

        ticket.setCustomerRating(rating);
        ticket.setCustomerFeedback(feedback);
        ticket.setStatus(TicketStatus.CLOSED);
        ticket.setClosedAt(LocalDateTime.now());

        return toDTO(ticketRepository.save(ticket));
    }

    @Scheduled(cron = "0 0/15 * * * *") // Every 15 minutes
    public void checkSLABreaches() {
        LocalDateTime now = LocalDateTime.now();
        List<Ticket> overdueTickets = ticketRepository.findOverdueTickets(now);

        for (Ticket ticket : overdueTickets) {
            if (ticket.getAssignedTo() != null) {
                emailService.sendSLABreachAlert(ticket.getAssignedTo().getEmail(), ticket);
            }
            log.warn("SLA BREACH - Ticket: {} Priority: {} Created: {}",
                ticket.getTicketNumber(), ticket.getPriority(), ticket.getCreatedAt());
        }
    }

    public AgentPerformanceReport getAgentPerformance(Long agentId, LocalDate from, LocalDate to) {
        LocalDateTime start = from.atStartOfDay();
        LocalDateTime end = to.plusDays(1).atStartOfDay();

        List<Ticket> assignedTickets = ticketRepository.findByAssignedToIdAndCreatedAtBetween(agentId, start, end);

        long resolved = assignedTickets.stream()
            .filter(t -> t.getStatus() == TicketStatus.RESOLVED || t.getStatus() == TicketStatus.CLOSED)
            .count();

        OptionalDouble avgFirstResponse = assignedTickets.stream()
            .filter(t -> t.getFirstResponseAt() != null)
            .mapToLong(t -> ChronoUnit.MINUTES.between(t.getCreatedAt(), t.getFirstResponseAt()))
            .average();

        OptionalDouble avgResolutionTime = assignedTickets.stream()
            .filter(t -> t.getResolvedAt() != null)
            .mapToLong(t -> ChronoUnit.HOURS.between(t.getCreatedAt(), t.getResolvedAt()))
            .average();

        OptionalDouble avgCsat = assignedTickets.stream()
            .filter(t -> t.getCustomerRating() != null)
            .mapToInt(Ticket::getCustomerRating)
            .average();

        long slaBreaches = assignedTickets.stream()
            .filter(t -> t.getDueDate() != null && t.getResolvedAt() != null &&
                        t.getResolvedAt().isAfter(t.getDueDate()))
            .count();

        return AgentPerformanceReport.builder()
            .agentId(agentId)
            .periodFrom(from)
            .periodTo(to)
            .totalTickets(assignedTickets.size())
            .resolvedTickets((int) resolved)
            .resolutionRate(assignedTickets.isEmpty() ? 0.0 : (double) resolved / assignedTickets.size() * 100)
            .avgFirstResponseMinutes(avgFirstResponse.orElse(0.0))
            .avgResolutionHours(avgResolutionTime.orElse(0.0))
            .avgCsatScore(avgCsat.orElse(0.0))
            .slaBreaches((int) slaBreaches)
            .build();
    }

    private void autoAssign(Ticket ticket, TicketCategory category) {
        // Auto-assign logic based on category, priority, and workload
        log.info("Auto-assigning ticket {} in category {}", ticket.getTicketNumber(), category.getName());
    }

    private boolean isStaff(Long userId) {
        // Check if user has staff/agent role
        return true; // Simplified - check via SecurityContext in real implementation
    }

    private String generateTicketNumber() {
        String date = LocalDate.now().format(DateTimeFormatter.ofPattern("yyyyMMdd"));
        long count = ticketRepository.countByCreatedAtToday() + 1;
        return String.format("TKT-%s-%04d", date, count);
    }

    private TicketDTO toDTO(Ticket t) {
        return TicketDTO.builder()
            .id(t.getId())
            .ticketNumber(t.getTicketNumber())
            .subject(t.getSubject())
            .priority(t.getPriority())
            .status(t.getStatus())
            .categoryName(t.getCategory() != null ? t.getCategory().getName() : null)
            .reporterName(t.getReporter().getUsername())
            .assignedToName(t.getAssignedTo() != null ? t.getAssignedTo().getUsername() : null)
            .dueDate(t.getDueDate())
            .firstResponseAt(t.getFirstResponseAt())
            .resolvedAt(t.getResolvedAt())
            .createdAt(t.getCreatedAt())
            .commentCount(t.getComments().size())
            .build();
    }

    private TicketCommentDTO toCommentDTO(TicketComment c) {
        return TicketCommentDTO.builder()
            .id(c.getId())
            .authorName(c.getAuthor().getUsername())
            .content(c.getContent())
            .isInternal(c.getIsInternal())
            .createdAt(c.getCreatedAt())
            .build();
    }
}
```

### REST Controller

```java
// TicketController.java
@RestController
@RequestMapping("/api/v1/tickets")
@RequiredArgsConstructor
public class TicketController {

    private final TicketService ticketService;

    @PostMapping
    public ResponseEntity<TicketDTO> createTicket(
            @Valid @RequestBody CreateTicketRequest request,
            @AuthenticationPrincipal UserPrincipal user) {
        return ResponseEntity.status(HttpStatus.CREATED).body(ticketService.createTicket(request, user.getId()));
    }

    @GetMapping
    public ResponseEntity<Page<TicketDTO>> getTickets(
            @RequestParam(required = false) TicketStatus status,
            @RequestParam(required = false) TicketPriority priority,
            @RequestParam(required = false) Long assignedTo,
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size) {
        return ResponseEntity.ok(ticketService.getTickets(status, priority, assignedTo, PageRequest.of(page, size)));
    }

    @GetMapping("/{id}")
    public ResponseEntity<TicketDetailDTO> getTicket(@PathVariable Long id) {
        return ResponseEntity.ok(ticketService.getTicketDetail(id));
    }

    @PostMapping("/{id}/comments")
    public ResponseEntity<TicketCommentDTO> addComment(
            @PathVariable Long id,
            @Valid @RequestBody AddCommentRequest request,
            @AuthenticationPrincipal UserPrincipal user) {
        return ResponseEntity.status(HttpStatus.CREATED)
            .body(ticketService.addComment(id, user.getId(), request.getContent(),
                                           request.isInternal(), request.getAttachmentUrls()));
    }

    @PatchMapping("/{id}/status")
    @PreAuthorize("hasRole('SUPPORT_AGENT') or hasRole('ADMIN')")
    public ResponseEntity<TicketDTO> updateStatus(
            @PathVariable Long id,
            @RequestParam TicketStatus status,
            @AuthenticationPrincipal UserPrincipal user) {
        return ResponseEntity.ok(ticketService.updateTicketStatus(id, status, user.getId()));
    }

    @PatchMapping("/{id}/assign")
    @PreAuthorize("hasRole('SUPPORT_ADMIN')")
    public ResponseEntity<TicketDTO> assignTicket(
            @PathVariable Long id,
            @RequestParam(required = false) Long agentId,
            @RequestParam(required = false) Long teamId) {
        return ResponseEntity.ok(ticketService.assignTicket(id, agentId, teamId));
    }

    @PostMapping("/{id}/csat")
    public ResponseEntity<TicketDTO> submitCsat(
            @PathVariable Long id,
            @AuthenticationPrincipal UserPrincipal user,
            @RequestParam int rating,
            @RequestParam(required = false) String feedback) {
        return ResponseEntity.ok(ticketService.submitCsat(id, user.getId(), rating, feedback));
    }

    @GetMapping("/my-tickets")
    public ResponseEntity<Page<TicketDTO>> getMyTickets(
            @AuthenticationPrincipal UserPrincipal user,
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "10") int size) {
        return ResponseEntity.ok(ticketService.getMyTickets(user.getId(), PageRequest.of(page, size)));
    }

    @GetMapping("/reports/agent-performance")
    @PreAuthorize("hasRole('SUPPORT_ADMIN')")
    public ResponseEntity<AgentPerformanceReport> getAgentPerformance(
            @RequestParam Long agentId,
            @RequestParam @DateTimeFormat(iso = DateTimeFormat.ISO.DATE) LocalDate from,
            @RequestParam @DateTimeFormat(iso = DateTimeFormat.ISO.DATE) LocalDate to) {
        return ResponseEntity.ok(ticketService.getAgentPerformance(agentId, from, to));
    }
}
```

### Flyway Migration

```sql
-- V25__init_support_tickets.sql
CREATE TABLE ticket_categories (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    description TEXT,
    sla_hours INT DEFAULT 24,
    default_assigned_team VARCHAR(100)
);

CREATE TABLE support_teams (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    description TEXT
);

CREATE TABLE sla_policies (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    priority VARCHAR(20) NOT NULL,
    first_response_hours INT,
    resolution_hours INT,
    active BOOLEAN DEFAULT TRUE,
    UNIQUE(priority, active)
);

INSERT INTO sla_policies (name, priority, first_response_hours, resolution_hours)
VALUES
    ('Low Priority SLA', 'LOW', 24, 72),
    ('Medium Priority SLA', 'MEDIUM', 8, 48),
    ('High Priority SLA', 'HIGH', 4, 24),
    ('Urgent Priority SLA', 'URGENT', 1, 4);

CREATE TABLE tickets (
    id BIGSERIAL PRIMARY KEY,
    ticket_number VARCHAR(30) NOT NULL UNIQUE,
    subject VARCHAR(500) NOT NULL,
    description TEXT,
    reporter_id BIGINT NOT NULL,
    category_id BIGINT REFERENCES ticket_categories(id),
    priority VARCHAR(20) NOT NULL DEFAULT 'MEDIUM',
    status VARCHAR(30) NOT NULL DEFAULT 'OPEN',
    assigned_to BIGINT,
    assigned_team_id BIGINT REFERENCES support_teams(id),
    first_response_at TIMESTAMP,
    resolved_at TIMESTAMP,
    closed_at TIMESTAMP,
    due_date TIMESTAMP,
    customer_rating INT CHECK (customer_rating BETWEEN 1 AND 5),
    customer_feedback TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE ticket_comments (
    id BIGSERIAL PRIMARY KEY,
    ticket_id BIGINT NOT NULL REFERENCES tickets(id),
    author_id BIGINT NOT NULL,
    content TEXT NOT NULL,
    is_internal BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE ticket_attachments (
    ticket_id BIGINT NOT NULL REFERENCES tickets(id),
    attachment_url VARCHAR(500) NOT NULL
);

CREATE TABLE team_members (
    team_id BIGINT NOT NULL REFERENCES support_teams(id),
    user_id BIGINT NOT NULL,
    PRIMARY KEY (team_id, user_id)
);

CREATE INDEX idx_tickets_status ON tickets(status);
CREATE INDEX idx_tickets_priority ON tickets(priority);
CREATE INDEX idx_tickets_reporter ON tickets(reporter_id);
CREATE INDEX idx_tickets_assigned ON tickets(assigned_to);
CREATE INDEX idx_tickets_due_date ON tickets(due_date);
CREATE INDEX idx_comments_ticket ON ticket_comments(ticket_id);
```

### ตัวอย่าง API Calls

```bash
# สร้าง Support Ticket
curl -X POST http://localhost:8080/api/v1/tickets \
  -H "Authorization: Bearer {token}" \
  -H "Content-Type: application/json" \
  -d '{
    "subject": "ไม่สามารถเข้าสู่ระบบได้",
    "description": "พยายาม login แล้วได้รับ error 401 ตลอด แม้จะใช้รหัสผ่านที่ถูกต้อง",
    "categoryId": 2,
    "priority": "HIGH"
  }'

# ดู Tickets ของตัวเอง
curl http://localhost:8080/api/v1/tickets/my-tickets \
  -H "Authorization: Bearer {token}"

# ตอบ Ticket
curl -X POST http://localhost:8080/api/v1/tickets/1/comments \
  -H "Authorization: Bearer {agent_token}" \
  -H "Content-Type: application/json" \
  -d '{
    "content": "ทีม Support ได้รับ Ticket ของคุณแล้ว กำลังตรวจสอบปัญหา",
    "internal": false
  }'

# เปลี่ยนสถานะ Ticket
curl -X PATCH "http://localhost:8080/api/v1/tickets/1/status?status=RESOLVED" \
  -H "Authorization: Bearer {agent_token}"

# ให้คะแนน CSAT
curl -X POST "http://localhost:8080/api/v1/tickets/1/csat?rating=5&feedback=แก้ไขได้รวดเร็วมาก" \
  -H "Authorization: Bearer {token}"

# ดูรายงาน Agent Performance
curl "http://localhost:8080/api/v1/tickets/reports/agent-performance?agentId=5&from=2024-01-01&to=2024-01-31" \
  -H "Authorization: Bearer {admin_token}"
```

---

## สรุปคอร์ส 100 Real-World Projects

คุณได้เรียนรู้และสร้างโปรเจคครบ 25 โปรเจคในส่วนนี้ ครอบคลุมระบบที่ใช้งานจริงหลากหลายประเภท:

### สิ่งที่ได้เรียนรู้

| หมวดหมู่ | โปรเจค | ทักษะหลัก |
|-----------|---------|-----------|
| E-Commerce & FinTech | 1-5 | JPA, Transaction, Idempotency |
| Business Systems | 6-10 | Complex Queries, Scheduling |
| Operations | 11-15 | Stock Management, GPA Calc |
| Platform & SaaS | 16-20 | CMS, Forum, LMS |
| Booking & Marketplace | 21-25 | WebSocket, SLA, Rating |

### Best Practices ที่ใช้ตลอดคอร์ส

```java
// 1. ใช้ DTO แยกจาก Entity เสมอ
// 2. ใช้ @Transactional ในทุก Service method ที่เปลี่ยนข้อมูล
// 3. ใช้ Pessimistic Lock สำหรับ concurrent operations
// 4. ใช้ Flyway สำหรับ Database Migration
// 5. ใช้ @Scheduled สำหรับ Background Jobs
// 6. ใช้ Custom Exceptions
// 7. ใช้ Builder Pattern สำหรับ DTO
// 8. ใช้ Specification Pattern สำหรับ Complex Queries
// 9. ใช้ Idempotency Keys สำหรับ Financial Operations
// 10. ใช้ Email Notifications สำหรับ User Feedback
```

---

*[← Part 104: Platform & SaaS](./part-104-platform-saas.md) | [Part 100: หลักสูตรสมบูรณ์](./part-100-whats-next.md)*

# Part 106: โปรเจค 26-30 — Real-time & Communication Systems

> **ระดับ:** ระดับโลก | **เวลาเรียนรู้:** 10-15 ชั่วโมง | **โปรเจค:** 26-30

---

## โปรเจค 26: Real-time Chat Application

### ภาพรวม
ระบบแชทแบบ Real-time ที่รองรับห้องสนทนา (Rooms), ข้อความส่วนตัว (Private Messages), สถานะออนไลน์ (Online Presence), ประวัติข้อความ, การแชร์ไฟล์, การยืนยันการอ่าน (Read Receipts) และตัวแสดงการพิมพ์ (Typing Indicators) โดยใช้ WebSocket และ STOMP Protocol

### Dependencies (pom.xml)
```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-websocket</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-security</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-redis</artifactId>
    </dependency>
    <dependency>
        <groupId>org.postgresql</groupId>
        <artifactId>postgresql</artifactId>
    </dependency>
</dependencies>
```

### Flyway Migration
```sql
-- V1__create_chat_tables.sql
CREATE TABLE chat_users (
    id BIGSERIAL PRIMARY KEY,
    username VARCHAR(50) UNIQUE NOT NULL,
    display_name VARCHAR(100) NOT NULL,
    avatar_url VARCHAR(500),
    status VARCHAR(20) DEFAULT 'OFFLINE',
    last_seen_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE chat_rooms (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    description TEXT,
    type VARCHAR(20) NOT NULL DEFAULT 'PUBLIC',
    created_by BIGINT REFERENCES chat_users(id),
    avatar_url VARCHAR(500),
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE room_members (
    id BIGSERIAL PRIMARY KEY,
    room_id BIGINT REFERENCES chat_rooms(id) ON DELETE CASCADE,
    user_id BIGINT REFERENCES chat_users(id) ON DELETE CASCADE,
    role VARCHAR(20) DEFAULT 'MEMBER',
    joined_at TIMESTAMP DEFAULT NOW(),
    UNIQUE(room_id, user_id)
);

CREATE TABLE chat_messages (
    id BIGSERIAL PRIMARY KEY,
    room_id BIGINT REFERENCES chat_rooms(id) ON DELETE CASCADE,
    sender_id BIGINT REFERENCES chat_users(id),
    content TEXT,
    message_type VARCHAR(20) DEFAULT 'TEXT',
    file_url VARCHAR(500),
    file_name VARCHAR(255),
    file_size BIGINT,
    reply_to_id BIGINT REFERENCES chat_messages(id),
    is_deleted BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE message_reads (
    id BIGSERIAL PRIMARY KEY,
    message_id BIGINT REFERENCES chat_messages(id) ON DELETE CASCADE,
    user_id BIGINT REFERENCES chat_users(id) ON DELETE CASCADE,
    read_at TIMESTAMP DEFAULT NOW(),
    UNIQUE(message_id, user_id)
);

CREATE INDEX idx_messages_room_created ON chat_messages(room_id, created_at DESC);
CREATE INDEX idx_room_members_user ON room_members(user_id);
```

### Entity Classes
```java
// ChatUser.java
@Entity
@Table(name = "chat_users")
@Data
@NoArgsConstructor
public class ChatUser {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(unique = true, nullable = false)
    private String username;

    @Column(name = "display_name", nullable = false)
    private String displayName;

    @Column(name = "avatar_url")
    private String avatarUrl;

    @Enumerated(EnumType.STRING)
    private UserStatus status = UserStatus.OFFLINE;

    @Column(name = "last_seen_at")
    private LocalDateTime lastSeenAt;

    @Column(name = "created_at")
    private LocalDateTime createdAt = LocalDateTime.now();

    public enum UserStatus { ONLINE, AWAY, BUSY, OFFLINE }
}

// ChatRoom.java
@Entity
@Table(name = "chat_rooms")
@Data
@NoArgsConstructor
public class ChatRoom {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String name;

    private String description;

    @Enumerated(EnumType.STRING)
    private RoomType type = RoomType.PUBLIC;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "created_by")
    private ChatUser createdBy;

    @Column(name = "avatar_url")
    private String avatarUrl;

    @Column(name = "created_at")
    private LocalDateTime createdAt = LocalDateTime.now();

    public enum RoomType { PUBLIC, PRIVATE, DIRECT }
}

// ChatMessage.java
@Entity
@Table(name = "chat_messages")
@Data
@NoArgsConstructor
public class ChatMessage {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "room_id", nullable = false)
    private ChatRoom room;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "sender_id")
    private ChatUser sender;

    @Column(columnDefinition = "TEXT")
    private String content;

    @Enumerated(EnumType.STRING)
    @Column(name = "message_type")
    private MessageType messageType = MessageType.TEXT;

    @Column(name = "file_url")
    private String fileUrl;

    @Column(name = "file_name")
    private String fileName;

    @Column(name = "file_size")
    private Long fileSize;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "reply_to_id")
    private ChatMessage replyTo;

    @Column(name = "is_deleted")
    private Boolean isDeleted = false;

    @Column(name = "created_at")
    private LocalDateTime createdAt = LocalDateTime.now();

    public enum MessageType { TEXT, IMAGE, FILE, SYSTEM }
}
```

### WebSocket Configuration
```java
// WebSocketConfig.java
@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {

    @Override
    public void configureMessageBroker(MessageBrokerRegistry registry) {
        registry.enableSimpleBroker("/topic", "/queue");
        registry.setApplicationDestinationPrefixes("/app");
        registry.setUserDestinationPrefix("/user");
    }

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/ws/chat")
                .setAllowedOriginPatterns("*")
                .withSockJS();
    }
}
```

### Repository
```java
// ChatMessageRepository.java
@Repository
public interface ChatMessageRepository extends JpaRepository<ChatMessage, Long> {

    @Query("SELECT m FROM ChatMessage m WHERE m.room.id = :roomId AND m.isDeleted = false ORDER BY m.createdAt DESC")
    Page<ChatMessage> findByRoomIdOrderByCreatedAtDesc(@Param("roomId") Long roomId, Pageable pageable);

    @Query("SELECT COUNT(m) FROM ChatMessage m LEFT JOIN MessageRead mr ON mr.message.id = m.id AND mr.user.id = :userId " +
           "WHERE m.room.id = :roomId AND mr.id IS NULL AND m.sender.id != :userId AND m.isDeleted = false")
    Long countUnreadMessages(@Param("roomId") Long roomId, @Param("userId") Long userId);
}

// ChatRoomRepository.java
@Repository
public interface ChatRoomRepository extends JpaRepository<ChatRoom, Long> {

    @Query("SELECT r FROM ChatRoom r JOIN r.members m WHERE m.user.id = :userId")
    List<ChatRoom> findRoomsByUserId(@Param("userId") Long userId);

    @Query("SELECT r FROM ChatRoom r WHERE r.type = 'PUBLIC' ORDER BY r.createdAt DESC")
    List<ChatRoom> findPublicRooms();
}
```

### Service
```java
// ChatService.java
@Service
@RequiredArgsConstructor
@Slf4j
public class ChatService {

    private final ChatMessageRepository messageRepository;
    private final ChatRoomRepository roomRepository;
    private final RoomMemberRepository roomMemberRepository;
    private final SimpMessagingTemplate messagingTemplate;
    private final RedisTemplate<String, Object> redisTemplate;

    private static final String ONLINE_USERS_KEY = "chat:online_users";
    private static final String TYPING_KEY_PREFIX = "chat:typing:";

    public ChatMessage sendMessage(Long roomId, Long senderId, SendMessageRequest request) {
        ChatRoom room = roomRepository.findById(roomId)
                .orElseThrow(() -> new ResourceNotFoundException("Room not found"));
        ChatUser sender = userRepository.findById(senderId)
                .orElseThrow(() -> new ResourceNotFoundException("User not found"));

        ChatMessage message = new ChatMessage();
        message.setRoom(room);
        message.setSender(sender);
        message.setContent(request.getContent());
        message.setMessageType(request.getMessageType());

        if (request.getReplyToId() != null) {
            messageRepository.findById(request.getReplyToId())
                    .ifPresent(message::setReplyTo);
        }

        ChatMessage saved = messageRepository.save(message);

        // Broadcast to room topic
        MessagePayload payload = MessagePayload.from(saved);
        messagingTemplate.convertAndSend("/topic/room." + roomId, payload);

        log.info("Message sent to room {} by user {}", roomId, senderId);
        return saved;
    }

    public void updateTypingStatus(Long roomId, Long userId, boolean isTyping) {
        String key = TYPING_KEY_PREFIX + roomId;
        String userIdStr = String.valueOf(userId);

        if (isTyping) {
            redisTemplate.opsForSet().add(key, userIdStr);
            redisTemplate.expire(key, Duration.ofSeconds(5));
        } else {
            redisTemplate.opsForSet().remove(key, userIdStr);
        }

        Set<Object> typingUsers = redisTemplate.opsForSet().members(key);
        messagingTemplate.convertAndSend("/topic/room." + roomId + ".typing",
                new TypingEvent(roomId, typingUsers));
    }

    public void markMessagesAsRead(Long roomId, Long userId, Long lastMessageId) {
        List<ChatMessage> unreadMessages = messageRepository
                .findUnreadMessagesUpTo(roomId, userId, lastMessageId);

        List<MessageRead> reads = unreadMessages.stream().map(msg -> {
            MessageRead read = new MessageRead();
            read.setMessage(msg);
            read.setUser(new ChatUser(userId));
            return read;
        }).collect(Collectors.toList());

        messageReadRepository.saveAll(reads);

        // Notify senders about read receipts
        messagingTemplate.convertAndSend("/topic/room." + roomId + ".read",
                new ReadReceiptEvent(roomId, userId, lastMessageId));
    }

    public void setUserOnline(Long userId) {
        redisTemplate.opsForSet().add(ONLINE_USERS_KEY, String.valueOf(userId));
        userRepository.updateStatus(userId, ChatUser.UserStatus.ONLINE);
        broadcastPresenceUpdate(userId, ChatUser.UserStatus.ONLINE);
    }

    public void setUserOffline(Long userId) {
        redisTemplate.opsForSet().remove(ONLINE_USERS_KEY, String.valueOf(userId));
        userRepository.updateStatusAndLastSeen(userId, ChatUser.UserStatus.OFFLINE, LocalDateTime.now());
        broadcastPresenceUpdate(userId, ChatUser.UserStatus.OFFLINE);
    }

    private void broadcastPresenceUpdate(Long userId, ChatUser.UserStatus status) {
        messagingTemplate.convertAndSend("/topic/presence",
                new PresenceEvent(userId, status));
    }

    public Page<ChatMessage> getRoomMessages(Long roomId, int page, int size) {
        return messageRepository.findByRoomIdOrderByCreatedAtDesc(
                roomId, PageRequest.of(page, size));
    }
}
```

### Controller
```java
// ChatController.java
@Controller
@RequiredArgsConstructor
@Slf4j
public class ChatController {

    private final ChatService chatService;

    @MessageMapping("/room/{roomId}/send")
    public void sendMessage(@DestinationVariable Long roomId,
                            @Payload SendMessageRequest request,
                            Principal principal) {
        Long userId = getUserId(principal);
        chatService.sendMessage(roomId, userId, request);
    }

    @MessageMapping("/room/{roomId}/typing")
    public void typing(@DestinationVariable Long roomId,
                       @Payload TypingRequest request,
                       Principal principal) {
        Long userId = getUserId(principal);
        chatService.updateTypingStatus(roomId, userId, request.isTyping());
    }

    @MessageMapping("/room/{roomId}/read")
    public void markRead(@DestinationVariable Long roomId,
                         @Payload ReadRequest request,
                         Principal principal) {
        Long userId = getUserId(principal);
        chatService.markMessagesAsRead(roomId, userId, request.getLastMessageId());
    }
}

// ChatRestController.java
@RestController
@RequestMapping("/api/chat")
@RequiredArgsConstructor
public class ChatRestController {

    private final ChatService chatService;
    private final FileStorageService fileStorageService;

    @GetMapping("/rooms/{roomId}/messages")
    public ResponseEntity<Page<MessageDTO>> getRoomMessages(
            @PathVariable Long roomId,
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "50") int size) {
        return ResponseEntity.ok(chatService.getRoomMessages(roomId, page, size)
                .map(MessageDTO::from));
    }

    @PostMapping("/rooms/{roomId}/files")
    public ResponseEntity<FileUploadResponse> uploadFile(
            @PathVariable Long roomId,
            @RequestParam("file") MultipartFile file,
            Authentication auth) {
        String fileUrl = fileStorageService.upload(file, "chat/" + roomId);
        return ResponseEntity.ok(new FileUploadResponse(fileUrl, file.getOriginalFilename(), file.getSize()));
    }

    @PostMapping("/rooms")
    public ResponseEntity<ChatRoomDTO> createRoom(@RequestBody @Valid CreateRoomRequest request,
                                                   Authentication auth) {
        return ResponseEntity.status(HttpStatus.CREATED)
                .body(chatService.createRoom(request, getCurrentUserId(auth)));
    }

    @GetMapping("/rooms")
    public ResponseEntity<List<ChatRoomDTO>> getMyRooms(Authentication auth) {
        return ResponseEntity.ok(chatService.getUserRooms(getCurrentUserId(auth)));
    }
}
```

### Docker Compose
```yaml
# docker-compose.yml (Project 26)
version: '3.8'
services:
  chat-app:
    build: .
    ports:
      - "8080:8080"
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/chatdb
      SPRING_REDIS_HOST: redis
    depends_on:
      - postgres
      - redis

  postgres:
    image: postgres:15
    environment:
      POSTGRES_DB: chatdb
      POSTGRES_USER: chat
      POSTGRES_PASSWORD: chatpass
    ports:
      - "5432:5432"
    volumes:
      - chat_data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

volumes:
  chat_data:
```

---

## โปรเจค 27: Ride Hailing Backend

### ภาพรวม
ระบบ Backend สำหรับบริการเรียกรถ (Ride Hailing) ครบวงจร รองรับการจัดการคนขับ (Drivers) และผู้โดยสาร (Riders), การร้องขอเดินทาง, การจับคู่คนขับโดยอิงจากระยะทาง (Proximity Matching), วงจรชีวิตการเดินทาง (Trip Lifecycle), การคำนวณค่าโดยสาร (Fare Calculation) และระบบให้คะแนน (Rating)

### Flyway Migration
```sql
-- V1__create_ridehailing_tables.sql
CREATE TABLE drivers (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    phone VARCHAR(20) UNIQUE NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    license_number VARCHAR(50) UNIQUE NOT NULL,
    status VARCHAR(20) DEFAULT 'OFFLINE',
    rating DECIMAL(3,2) DEFAULT 5.00,
    total_trips INTEGER DEFAULT 0,
    current_lat DECIMAL(10,8),
    current_lng DECIMAL(11,8),
    last_location_update TIMESTAMP,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE vehicles (
    id BIGSERIAL PRIMARY KEY,
    driver_id BIGINT UNIQUE REFERENCES drivers(id),
    make VARCHAR(50) NOT NULL,
    model VARCHAR(50) NOT NULL,
    year INTEGER NOT NULL,
    plate_number VARCHAR(20) UNIQUE NOT NULL,
    vehicle_type VARCHAR(20) DEFAULT 'STANDARD',
    color VARCHAR(30),
    capacity INTEGER DEFAULT 4
);

CREATE TABLE riders (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    phone VARCHAR(20) UNIQUE NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    rating DECIMAL(3,2) DEFAULT 5.00,
    total_trips INTEGER DEFAULT 0,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE trips (
    id BIGSERIAL PRIMARY KEY,
    rider_id BIGINT REFERENCES riders(id),
    driver_id BIGINT REFERENCES drivers(id),
    status VARCHAR(20) DEFAULT 'REQUESTED',
    pickup_lat DECIMAL(10,8) NOT NULL,
    pickup_lng DECIMAL(11,8) NOT NULL,
    pickup_address VARCHAR(500),
    dropoff_lat DECIMAL(10,8) NOT NULL,
    dropoff_lng DECIMAL(11,8) NOT NULL,
    dropoff_address VARCHAR(500),
    estimated_distance DECIMAL(10,2),
    estimated_duration INTEGER,
    actual_distance DECIMAL(10,2),
    actual_duration INTEGER,
    base_fare DECIMAL(10,2),
    distance_fare DECIMAL(10,2),
    time_fare DECIMAL(10,2),
    total_fare DECIMAL(10,2),
    surge_multiplier DECIMAL(4,2) DEFAULT 1.00,
    vehicle_type VARCHAR(20) DEFAULT 'STANDARD',
    rider_rating INTEGER,
    driver_rating INTEGER,
    requested_at TIMESTAMP DEFAULT NOW(),
    accepted_at TIMESTAMP,
    started_at TIMESTAMP,
    completed_at TIMESTAMP,
    cancelled_at TIMESTAMP,
    cancellation_reason TEXT
);
```

### Entity Classes
```java
// Driver.java
@Entity
@Table(name = "drivers")
@Data
@NoArgsConstructor
public class Driver {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String name;

    @Column(unique = true, nullable = false)
    private String phone;

    @Column(unique = true, nullable = false)
    private String email;

    @Column(name = "license_number", unique = true)
    private String licenseNumber;

    @Enumerated(EnumType.STRING)
    private DriverStatus status = DriverStatus.OFFLINE;

    @Column(precision = 3, scale = 2)
    private BigDecimal rating = BigDecimal.valueOf(5.00);

    @Column(name = "total_trips")
    private Integer totalTrips = 0;

    @Column(name = "current_lat", precision = 10, scale = 8)
    private BigDecimal currentLat;

    @Column(name = "current_lng", precision = 11, scale = 8)
    private BigDecimal currentLng;

    @Column(name = "last_location_update")
    private LocalDateTime lastLocationUpdate;

    @OneToOne(mappedBy = "driver", cascade = CascadeType.ALL)
    private Vehicle vehicle;

    public enum DriverStatus { ONLINE, OFFLINE, ON_TRIP, BUSY }
}

// Trip.java
@Entity
@Table(name = "trips")
@Data
@NoArgsConstructor
public class Trip {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "rider_id")
    private Rider rider;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "driver_id")
    private Driver driver;

    @Enumerated(EnumType.STRING)
    private TripStatus status = TripStatus.REQUESTED;

    @Column(name = "pickup_lat", precision = 10, scale = 8)
    private BigDecimal pickupLat;

    @Column(name = "pickup_lng", precision = 11, scale = 8)
    private BigDecimal pickupLng;

    @Column(name = "pickup_address")
    private String pickupAddress;

    @Column(name = "dropoff_lat", precision = 10, scale = 8)
    private BigDecimal dropoffLat;

    @Column(name = "dropoff_lng", precision = 11, scale = 8)
    private BigDecimal dropoffLng;

    @Column(name = "dropoff_address")
    private String dropoffAddress;

    @Column(name = "total_fare", precision = 10, scale = 2)
    private BigDecimal totalFare;

    @Column(name = "surge_multiplier", precision = 4, scale = 2)
    private BigDecimal surgeMultiplier = BigDecimal.ONE;

    @Column(name = "rider_rating")
    private Integer riderRating;

    @Column(name = "driver_rating")
    private Integer driverRating;

    @Column(name = "requested_at")
    private LocalDateTime requestedAt = LocalDateTime.now();

    @Column(name = "accepted_at")
    private LocalDateTime acceptedAt;

    @Column(name = "started_at")
    private LocalDateTime startedAt;

    @Column(name = "completed_at")
    private LocalDateTime completedAt;

    public enum TripStatus { REQUESTED, ACCEPTED, DRIVER_ARRIVING, STARTED, COMPLETED, CANCELLED }
}
```

### Service
```java
// TripService.java
@Service
@RequiredArgsConstructor
@Slf4j
@Transactional
public class TripService {

    private final TripRepository tripRepository;
    private final DriverRepository driverRepository;
    private final FareCalculationService fareService;
    private final DriverMatchingService matchingService;
    private final NotificationService notificationService;

    public Trip requestTrip(Long riderId, TripRequest request) {
        // Validate rider doesn't have active trip
        if (tripRepository.hasActiveTrip(riderId)) {
            throw new BusinessException("Rider already has an active trip");
        }

        Trip trip = new Trip();
        trip.setRider(new Rider(riderId));
        trip.setPickupLat(request.getPickupLat());
        trip.setPickupLng(request.getPickupLng());
        trip.setPickupAddress(request.getPickupAddress());
        trip.setDropoffLat(request.getDropoffLat());
        trip.setDropoffLng(request.getDropoffLng());
        trip.setDropoffAddress(request.getDropoffAddress());
        trip.setVehicleType(request.getVehicleType());
        trip.setStatus(Trip.TripStatus.REQUESTED);

        // Calculate fare estimate
        FareEstimate estimate = fareService.calculateEstimate(
                request.getPickupLat(), request.getPickupLng(),
                request.getDropoffLat(), request.getDropoffLng(),
                request.getVehicleType());
        trip.setEstimatedDistance(estimate.getDistance());
        trip.setEstimatedDuration(estimate.getDuration());
        trip.setTotalFare(estimate.getFare());
        trip.setSurgeMultiplier(fareService.getSurgeMultiplier(request.getPickupLat(), request.getPickupLng()));

        Trip savedTrip = tripRepository.save(trip);

        // Find and notify nearby drivers asynchronously
        matchingService.findAndNotifyDrivers(savedTrip);

        return savedTrip;
    }

    public Trip acceptTrip(Long tripId, Long driverId) {
        Trip trip = tripRepository.findById(tripId)
                .orElseThrow(() -> new ResourceNotFoundException("Trip not found"));

        if (trip.getStatus() != Trip.TripStatus.REQUESTED) {
            throw new BusinessException("Trip is no longer available");
        }

        Driver driver = driverRepository.findById(driverId)
                .orElseThrow(() -> new ResourceNotFoundException("Driver not found"));

        trip.setDriver(driver);
        trip.setStatus(Trip.TripStatus.ACCEPTED);
        trip.setAcceptedAt(LocalDateTime.now());

        driver.setStatus(Driver.DriverStatus.BUSY);
        driverRepository.save(driver);

        notificationService.notifyRider(trip.getRider().getId(),
                "Driver " + driver.getName() + " is on the way!");

        return tripRepository.save(trip);
    }

    public Trip startTrip(Long tripId, Long driverId) {
        Trip trip = getAndValidateTrip(tripId, driverId, Trip.TripStatus.ACCEPTED);
        trip.setStatus(Trip.TripStatus.STARTED);
        trip.setStartedAt(LocalDateTime.now());

        Driver driver = trip.getDriver();
        driver.setStatus(Driver.DriverStatus.ON_TRIP);
        driverRepository.save(driver);

        return tripRepository.save(trip);
    }

    public Trip completeTrip(Long tripId, Long driverId, CompleteTripRequest request) {
        Trip trip = getAndValidateTrip(tripId, driverId, Trip.TripStatus.STARTED);

        trip.setStatus(Trip.TripStatus.COMPLETED);
        trip.setCompletedAt(LocalDateTime.now());
        trip.setActualDistance(request.getActualDistance());

        // Recalculate actual fare
        BigDecimal actualFare = fareService.calculateActualFare(
                trip.getActualDistance(), request.getActualDuration(),
                trip.getVehicleType(), trip.getSurgeMultiplier());
        trip.setTotalFare(actualFare);

        Driver driver = trip.getDriver();
        driver.setStatus(Driver.DriverStatus.ONLINE);
        driver.setTotalTrips(driver.getTotalTrips() + 1);
        driverRepository.save(driver);

        return tripRepository.save(trip);
    }

    private Trip getAndValidateTrip(Long tripId, Long driverId, Trip.TripStatus expectedStatus) {
        Trip trip = tripRepository.findById(tripId)
                .orElseThrow(() -> new ResourceNotFoundException("Trip not found"));
        if (!trip.getDriver().getId().equals(driverId)) {
            throw new AccessDeniedException("Not authorized for this trip");
        }
        if (trip.getStatus() != expectedStatus) {
            throw new BusinessException("Invalid trip status transition");
        }
        return trip;
    }
}

// FareCalculationService.java
@Service
public class FareCalculationService {

    private static final Map<String, BigDecimal[]> FARE_CONFIG = Map.of(
        "STANDARD", new BigDecimal[]{new BigDecimal("2.50"), new BigDecimal("1.20"), new BigDecimal("0.25")},
        "PREMIUM",  new BigDecimal[]{new BigDecimal("4.00"), new BigDecimal("2.00"), new BigDecimal("0.40")},
        "XL",       new BigDecimal[]{new BigDecimal("3.50"), new BigDecimal("1.60"), new BigDecimal("0.30")}
    );

    public FareEstimate calculateEstimate(BigDecimal pickupLat, BigDecimal pickupLng,
                                          BigDecimal dropoffLat, BigDecimal dropoffLng,
                                          String vehicleType) {
        double distanceKm = calculateHaversineDistance(
                pickupLat.doubleValue(), pickupLng.doubleValue(),
                dropoffLat.doubleValue(), dropoffLng.doubleValue());
        int durationMinutes = (int)(distanceKm * 3); // rough estimate

        BigDecimal[] config = FARE_CONFIG.getOrDefault(vehicleType, FARE_CONFIG.get("STANDARD"));
        BigDecimal baseFare = config[0];
        BigDecimal perKm = config[1];
        BigDecimal perMinute = config[2];

        BigDecimal fare = baseFare
                .add(perKm.multiply(BigDecimal.valueOf(distanceKm)))
                .add(perMinute.multiply(BigDecimal.valueOf(durationMinutes)));

        return new FareEstimate(BigDecimal.valueOf(distanceKm), durationMinutes, fare.setScale(2, RoundingMode.HALF_UP));
    }

    private double calculateHaversineDistance(double lat1, double lng1, double lat2, double lng2) {
        final double R = 6371;
        double dLat = Math.toRadians(lat2 - lat1);
        double dLng = Math.toRadians(lng2 - lng1);
        double a = Math.sin(dLat/2) * Math.sin(dLat/2)
                + Math.cos(Math.toRadians(lat1)) * Math.cos(Math.toRadians(lat2))
                * Math.sin(dLng/2) * Math.sin(dLng/2);
        return R * 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1-a));
    }

    public BigDecimal getSurgeMultiplier(BigDecimal lat, BigDecimal lng) {
        // In real scenario: check demand/supply ratio in area
        return BigDecimal.ONE;
    }
}
```

### Controller
```java
// TripController.java
@RestController
@RequestMapping("/api/trips")
@RequiredArgsConstructor
public class TripController {

    private final TripService tripService;

    @PostMapping
    public ResponseEntity<TripDTO> requestTrip(@RequestBody @Valid TripRequest request,
                                                Authentication auth) {
        Trip trip = tripService.requestTrip(getRiderId(auth), request);
        return ResponseEntity.status(HttpStatus.CREATED).body(TripDTO.from(trip));
    }

    @PostMapping("/{tripId}/accept")
    public ResponseEntity<TripDTO> acceptTrip(@PathVariable Long tripId, Authentication auth) {
        Trip trip = tripService.acceptTrip(tripId, getDriverId(auth));
        return ResponseEntity.ok(TripDTO.from(trip));
    }

    @PostMapping("/{tripId}/start")
    public ResponseEntity<TripDTO> startTrip(@PathVariable Long tripId, Authentication auth) {
        Trip trip = tripService.startTrip(tripId, getDriverId(auth));
        return ResponseEntity.ok(TripDTO.from(trip));
    }

    @PostMapping("/{tripId}/complete")
    public ResponseEntity<TripDTO> completeTrip(@PathVariable Long tripId,
                                                 @RequestBody CompleteTripRequest request,
                                                 Authentication auth) {
        Trip trip = tripService.completeTrip(tripId, getDriverId(auth), request);
        return ResponseEntity.ok(TripDTO.from(trip));
    }

    @PostMapping("/{tripId}/cancel")
    public ResponseEntity<TripDTO> cancelTrip(@PathVariable Long tripId,
                                               @RequestBody CancelTripRequest request,
                                               Authentication auth) {
        Trip trip = tripService.cancelTrip(tripId, getCurrentUserId(auth), request.getReason());
        return ResponseEntity.ok(TripDTO.from(trip));
    }

    @PostMapping("/{tripId}/rate")
    public ResponseEntity<Void> rateTrip(@PathVariable Long tripId,
                                          @RequestBody @Valid RatingRequest request,
                                          Authentication auth) {
        tripService.rateTrip(tripId, getCurrentUserId(auth), request);
        return ResponseEntity.ok().build();
    }

    @PutMapping("/drivers/location")
    public ResponseEntity<Void> updateDriverLocation(@RequestBody LocationUpdate location,
                                                       Authentication auth) {
        tripService.updateDriverLocation(getDriverId(auth), location);
        return ResponseEntity.ok().build();
    }

    @GetMapping("/nearby-drivers")
    public ResponseEntity<List<DriverDTO>> getNearbyDrivers(
            @RequestParam BigDecimal lat,
            @RequestParam BigDecimal lng,
            @RequestParam(defaultValue = "5") Double radiusKm) {
        return ResponseEntity.ok(tripService.getNearbyDrivers(lat, lng, radiusKm));
    }
}
```

---

## โปรเจค 28: Live Notification Service

### ภาพรวม
บริการแจ้งเตือนแบบ Real-time ที่รองรับหลายช่องทางการส่ง ได้แก่ Push Notification, Email, SMS และ In-App Notification พร้อม Template Engine, การตั้งเวลาส่ง (Scheduling), การติดตามสถานะ อ่าน/ยังไม่อ่าน และการส่งจำนวนมาก (Bulk Send)

### Flyway Migration
```sql
-- V1__create_notification_tables.sql
CREATE TABLE notification_templates (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(100) UNIQUE NOT NULL,
    type VARCHAR(20) NOT NULL,
    subject VARCHAR(200),
    body_template TEXT NOT NULL,
    variables JSONB,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE notifications (
    id BIGSERIAL PRIMARY KEY,
    recipient_id BIGINT NOT NULL,
    template_id BIGINT REFERENCES notification_templates(id),
    type VARCHAR(20) NOT NULL,
    channel VARCHAR(20) NOT NULL,
    subject VARCHAR(200),
    body TEXT NOT NULL,
    status VARCHAR(20) DEFAULT 'PENDING',
    priority VARCHAR(10) DEFAULT 'NORMAL',
    scheduled_at TIMESTAMP,
    sent_at TIMESTAMP,
    read_at TIMESTAMP,
    error_message TEXT,
    metadata JSONB,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE notification_preferences (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT UNIQUE NOT NULL,
    email_enabled BOOLEAN DEFAULT TRUE,
    push_enabled BOOLEAN DEFAULT TRUE,
    sms_enabled BOOLEAN DEFAULT FALSE,
    in_app_enabled BOOLEAN DEFAULT TRUE,
    quiet_hours_start TIME,
    quiet_hours_end TIME,
    timezone VARCHAR(50) DEFAULT 'UTC'
);

CREATE INDEX idx_notifications_recipient ON notifications(recipient_id, created_at DESC);
CREATE INDEX idx_notifications_status ON notifications(status, scheduled_at);
```

### Entity & Service
```java
// Notification.java
@Entity
@Table(name = "notifications")
@Data
@NoArgsConstructor
public class Notification {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "recipient_id", nullable = false)
    private Long recipientId;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "template_id")
    private NotificationTemplate template;

    @Enumerated(EnumType.STRING)
    private NotificationType type;

    @Enumerated(EnumType.STRING)
    private NotificationChannel channel;

    private String subject;

    @Column(columnDefinition = "TEXT")
    private String body;

    @Enumerated(EnumType.STRING)
    private NotificationStatus status = NotificationStatus.PENDING;

    @Enumerated(EnumType.STRING)
    private Priority priority = Priority.NORMAL;

    @Column(name = "scheduled_at")
    private LocalDateTime scheduledAt;

    @Column(name = "sent_at")
    private LocalDateTime sentAt;

    @Column(name = "read_at")
    private LocalDateTime readAt;

    @Column(name = "error_message")
    private String errorMessage;

    @Type(JsonType.class)
    @Column(columnDefinition = "jsonb")
    private Map<String, Object> metadata;

    public enum NotificationType { SYSTEM, MARKETING, TRANSACTIONAL, ALERT }
    public enum NotificationChannel { EMAIL, SMS, PUSH, IN_APP }
    public enum NotificationStatus { PENDING, SCHEDULED, SENT, DELIVERED, FAILED, READ }
    public enum Priority { LOW, NORMAL, HIGH, URGENT }
}

// NotificationService.java
@Service
@RequiredArgsConstructor
@Slf4j
public class NotificationService {

    private final NotificationRepository notificationRepository;
    private final NotificationTemplateRepository templateRepository;
    private final EmailNotificationSender emailSender;
    private final SmsNotificationSender smsSender;
    private final PushNotificationSender pushSender;
    private final SimpMessagingTemplate messagingTemplate;
    private final TemplateEngine templateEngine;

    public Notification send(SendNotificationRequest request) {
        Notification notification = buildNotification(request);
        notificationRepository.save(notification);

        if (request.getScheduledAt() != null && request.getScheduledAt().isAfter(LocalDateTime.now())) {
            notification.setStatus(Notification.NotificationStatus.SCHEDULED);
            notification.setScheduledAt(request.getScheduledAt());
        } else {
            sendImmediately(notification);
        }

        return notificationRepository.save(notification);
    }

    public void sendBulk(BulkNotificationRequest request) {
        List<Notification> notifications = request.getRecipientIds().stream()
                .map(recipientId -> {
                    SendNotificationRequest req = request.toSingleRequest(recipientId);
                    return buildNotification(req);
                })
                .collect(Collectors.toList());

        notificationRepository.saveAll(notifications);

        // Process asynchronously
        CompletableFuture.runAsync(() ->
                notifications.forEach(this::sendImmediately));
    }

    private void sendImmediately(Notification notification) {
        try {
            switch (notification.getChannel()) {
                case EMAIL -> emailSender.send(notification);
                case SMS -> smsSender.send(notification);
                case PUSH -> pushSender.send(notification);
                case IN_APP -> deliverInApp(notification);
            }
            notification.setStatus(Notification.NotificationStatus.SENT);
            notification.setSentAt(LocalDateTime.now());
        } catch (Exception e) {
            log.error("Failed to send notification {}: {}", notification.getId(), e.getMessage());
            notification.setStatus(Notification.NotificationStatus.FAILED);
            notification.setErrorMessage(e.getMessage());
        }
        notificationRepository.save(notification);
    }

    private void deliverInApp(Notification notification) {
        messagingTemplate.convertAndSendToUser(
                String.valueOf(notification.getRecipientId()),
                "/queue/notifications",
                NotificationPayload.from(notification));
    }

    private Notification buildNotification(SendNotificationRequest request) {
        Notification notification = new Notification();
        notification.setRecipientId(request.getRecipientId());
        notification.setChannel(request.getChannel());
        notification.setType(request.getType());
        notification.setPriority(request.getPriority());
        notification.setMetadata(request.getMetadata());

        if (request.getTemplateId() != null) {
            NotificationTemplate template = templateRepository.findById(request.getTemplateId())
                    .orElseThrow(() -> new ResourceNotFoundException("Template not found"));
            notification.setTemplate(template);
            notification.setSubject(renderTemplate(template.getSubject(), request.getVariables()));
            notification.setBody(renderTemplate(template.getBodyTemplate(), request.getVariables()));
        } else {
            notification.setSubject(request.getSubject());
            notification.setBody(request.getBody());
        }

        return notification;
    }

    private String renderTemplate(String template, Map<String, Object> variables) {
        if (template == null || variables == null) return template;
        Context context = new Context();
        context.setVariables(variables);
        return templateEngine.process(template, context);
    }

    @Scheduled(fixedRate = 60000)
    public void processScheduledNotifications() {
        List<Notification> due = notificationRepository
                .findDueScheduledNotifications(LocalDateTime.now());
        due.forEach(this::sendImmediately);
    }

    public long markAsRead(Long userId, Long notificationId) {
        return notificationRepository.markAsRead(notificationId, userId, LocalDateTime.now());
    }

    public long markAllAsRead(Long userId) {
        return notificationRepository.markAllAsRead(userId, LocalDateTime.now());
    }
}
```

### Controller
```java
// NotificationController.java
@RestController
@RequestMapping("/api/notifications")
@RequiredArgsConstructor
public class NotificationController {

    private final NotificationService notificationService;

    @PostMapping
    public ResponseEntity<NotificationDTO> send(@RequestBody @Valid SendNotificationRequest request) {
        return ResponseEntity.status(HttpStatus.CREATED)
                .body(NotificationDTO.from(notificationService.send(request)));
    }

    @PostMapping("/bulk")
    public ResponseEntity<Void> sendBulk(@RequestBody @Valid BulkNotificationRequest request) {
        notificationService.sendBulk(request);
        return ResponseEntity.accepted().build();
    }

    @GetMapping("/my")
    public ResponseEntity<Page<NotificationDTO>> getMyNotifications(
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size,
            Authentication auth) {
        return ResponseEntity.ok(notificationService.getUserNotifications(
                getCurrentUserId(auth), PageRequest.of(page, size)));
    }

    @GetMapping("/my/unread-count")
    public ResponseEntity<Map<String, Long>> getUnreadCount(Authentication auth) {
        long count = notificationService.getUnreadCount(getCurrentUserId(auth));
        return ResponseEntity.ok(Map.of("count", count));
    }

    @PutMapping("/{id}/read")
    public ResponseEntity<Void> markAsRead(@PathVariable Long id, Authentication auth) {
        notificationService.markAsRead(getCurrentUserId(auth), id);
        return ResponseEntity.ok().build();
    }

    @PutMapping("/read-all")
    public ResponseEntity<Map<String, Long>> markAllAsRead(Authentication auth) {
        long updated = notificationService.markAllAsRead(getCurrentUserId(auth));
        return ResponseEntity.ok(Map.of("updated", updated));
    }
}
```

---

## โปรเจค 29: IoT Device Management

### ภาพรวม
ระบบจัดการอุปกรณ์ IoT ที่รองรับการลงทะเบียนอุปกรณ์, ประเภทอุปกรณ์, การรับข้อมูล Telemetry, การส่งคำสั่งไปยังอุปกรณ์ (Commands), การแจ้งเตือนเมื่อเกิดการละเมิดขีดจำกัด (Threshold Alerts), กลุ่มอุปกรณ์ (Device Groups) และการอัปเดต Firmware

### Flyway Migration
```sql
-- V1__create_iot_tables.sql
CREATE TABLE device_types (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(100) UNIQUE NOT NULL,
    description TEXT,
    telemetry_schema JSONB,
    command_schema JSONB,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE devices (
    id BIGSERIAL PRIMARY KEY,
    device_id VARCHAR(100) UNIQUE NOT NULL,
    name VARCHAR(100) NOT NULL,
    type_id BIGINT REFERENCES device_types(id),
    status VARCHAR(20) DEFAULT 'INACTIVE',
    firmware_version VARCHAR(50),
    last_seen_at TIMESTAMP,
    last_telemetry JSONB,
    location_lat DECIMAL(10,8),
    location_lng DECIMAL(11,8),
    tags JSONB,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE device_groups (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    description TEXT,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE device_group_members (
    device_id BIGINT REFERENCES devices(id) ON DELETE CASCADE,
    group_id BIGINT REFERENCES device_groups(id) ON DELETE CASCADE,
    PRIMARY KEY (device_id, group_id)
);

CREATE TABLE telemetry_data (
    id BIGSERIAL PRIMARY KEY,
    device_id BIGINT REFERENCES devices(id) ON DELETE CASCADE,
    data JSONB NOT NULL,
    received_at TIMESTAMP DEFAULT NOW()
) PARTITION BY RANGE (received_at);

CREATE TABLE telemetry_data_2024 PARTITION OF telemetry_data
    FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');

CREATE TABLE device_commands (
    id BIGSERIAL PRIMARY KEY,
    device_id BIGINT REFERENCES devices(id),
    command_name VARCHAR(100) NOT NULL,
    parameters JSONB,
    status VARCHAR(20) DEFAULT 'PENDING',
    sent_at TIMESTAMP,
    acknowledged_at TIMESTAMP,
    result JSONB,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE alert_rules (
    id BIGSERIAL PRIMARY KEY,
    device_type_id BIGINT REFERENCES device_types(id),
    name VARCHAR(100) NOT NULL,
    metric_path VARCHAR(200) NOT NULL,
    operator VARCHAR(10) NOT NULL,
    threshold_value DECIMAL(20,6) NOT NULL,
    severity VARCHAR(20) DEFAULT 'WARNING',
    enabled BOOLEAN DEFAULT TRUE
);

CREATE TABLE device_alerts (
    id BIGSERIAL PRIMARY KEY,
    device_id BIGINT REFERENCES devices(id),
    rule_id BIGINT REFERENCES alert_rules(id),
    actual_value DECIMAL(20,6),
    status VARCHAR(20) DEFAULT 'ACTIVE',
    triggered_at TIMESTAMP DEFAULT NOW(),
    resolved_at TIMESTAMP
);
```

### Entity & Service
```java
// Device.java
@Entity
@Table(name = "devices")
@Data
@NoArgsConstructor
public class Device {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "device_id", unique = true, nullable = false)
    private String deviceId;

    @Column(nullable = false)
    private String name;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "type_id")
    private DeviceType type;

    @Enumerated(EnumType.STRING)
    private DeviceStatus status = DeviceStatus.INACTIVE;

    @Column(name = "firmware_version")
    private String firmwareVersion;

    @Column(name = "last_seen_at")
    private LocalDateTime lastSeenAt;

    @Type(JsonType.class)
    @Column(name = "last_telemetry", columnDefinition = "jsonb")
    private Map<String, Object> lastTelemetry;

    @Type(JsonType.class)
    @Column(columnDefinition = "jsonb")
    private Map<String, String> tags;

    @ManyToMany
    @JoinTable(name = "device_group_members",
            joinColumns = @JoinColumn(name = "device_id"),
            inverseJoinColumns = @JoinColumn(name = "group_id"))
    private Set<DeviceGroup> groups = new HashSet<>();

    public enum DeviceStatus { ACTIVE, INACTIVE, MAINTENANCE, ERROR }
}

// IoTService.java
@Service
@RequiredArgsConstructor
@Slf4j
public class IoTService {

    private final DeviceRepository deviceRepository;
    private final TelemetryRepository telemetryRepository;
    private final DeviceCommandRepository commandRepository;
    private final AlertRuleRepository alertRuleRepository;
    private final DeviceAlertRepository alertRepository;
    private final MqttGateway mqttGateway;
    private final SimpMessagingTemplate wsTemplate;

    @Transactional
    public void ingestTelemetry(String deviceId, Map<String, Object> data) {
        Device device = deviceRepository.findByDeviceId(deviceId)
                .orElseThrow(() -> new ResourceNotFoundException("Device not found: " + deviceId));

        // Save telemetry
        TelemetryData telemetry = new TelemetryData();
        telemetry.setDevice(device);
        telemetry.setData(data);
        telemetryRepository.save(telemetry);

        // Update device last seen and last telemetry
        device.setLastSeenAt(LocalDateTime.now());
        device.setLastTelemetry(data);
        device.setStatus(Device.DeviceStatus.ACTIVE);
        deviceRepository.save(device);

        // Check alert rules
        checkAlertRules(device, data);

        // Stream to WebSocket clients
        wsTemplate.convertAndSend("/topic/device." + deviceId + ".telemetry", data);
    }

    public DeviceCommand sendCommand(Long deviceId, SendCommandRequest request) {
        Device device = deviceRepository.findById(deviceId)
                .orElseThrow(() -> new ResourceNotFoundException("Device not found"));

        DeviceCommand command = new DeviceCommand();
        command.setDevice(device);
        command.setCommandName(request.getCommandName());
        command.setParameters(request.getParameters());
        command.setStatus(DeviceCommand.CommandStatus.PENDING);

        DeviceCommand saved = commandRepository.save(command);

        // Send via MQTT
        String topic = "devices/" + device.getDeviceId() + "/commands";
        mqttGateway.sendToMqtt(topic, buildCommandPayload(saved));
        saved.setStatus(DeviceCommand.CommandStatus.SENT);
        saved.setSentAt(LocalDateTime.now());

        return commandRepository.save(saved);
    }

    private void checkAlertRules(Device device, Map<String, Object> telemetryData) {
        if (device.getType() == null) return;

        List<AlertRule> rules = alertRuleRepository.findByDeviceTypeAndEnabled(
                device.getType().getId(), true);

        for (AlertRule rule : rules) {
            Object value = extractValue(telemetryData, rule.getMetricPath());
            if (value instanceof Number numValue) {
                double actual = numValue.doubleValue();
                boolean violated = checkThreshold(actual, rule.getOperator(), rule.getThresholdValue().doubleValue());

                if (violated) {
                    createAlert(device, rule, actual);
                } else {
                    resolveAlert(device, rule);
                }
            }
        }
    }

    private boolean checkThreshold(double actual, String operator, double threshold) {
        return switch (operator) {
            case ">" -> actual > threshold;
            case ">=" -> actual >= threshold;
            case "<" -> actual < threshold;
            case "<=" -> actual <= threshold;
            case "==" -> actual == threshold;
            default -> false;
        };
    }
}
```

### Controller
```java
// IoTController.java
@RestController
@RequestMapping("/api/iot")
@RequiredArgsConstructor
public class IoTController {

    private final IoTService iotService;

    @PostMapping("/devices/{deviceId}/telemetry")
    public ResponseEntity<Void> ingestTelemetry(
            @PathVariable String deviceId,
            @RequestBody Map<String, Object> data,
            @RequestHeader("X-Device-Token") String token) {
        iotService.validateDeviceToken(deviceId, token);
        iotService.ingestTelemetry(deviceId, data);
        return ResponseEntity.ok().build();
    }

    @PostMapping("/devices/{deviceId}/commands")
    public ResponseEntity<CommandDTO> sendCommand(
            @PathVariable Long deviceId,
            @RequestBody @Valid SendCommandRequest request) {
        return ResponseEntity.ok(CommandDTO.from(iotService.sendCommand(deviceId, request)));
    }

    @GetMapping("/devices")
    public ResponseEntity<Page<DeviceDTO>> listDevices(
            @RequestParam(required = false) String status,
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size) {
        return ResponseEntity.ok(iotService.listDevices(status, PageRequest.of(page, size)));
    }

    @GetMapping("/devices/{id}/telemetry")
    public ResponseEntity<List<TelemetryDTO>> getDeviceTelemetry(
            @PathVariable Long id,
            @RequestParam @DateTimeFormat(iso = DateTimeFormat.ISO.DATE_TIME) LocalDateTime from,
            @RequestParam @DateTimeFormat(iso = DateTimeFormat.ISO.DATE_TIME) LocalDateTime to) {
        return ResponseEntity.ok(iotService.getDeviceTelemetry(id, from, to));
    }

    @GetMapping("/devices/{id}/alerts")
    public ResponseEntity<List<AlertDTO>> getDeviceAlerts(@PathVariable Long id) {
        return ResponseEntity.ok(iotService.getDeviceAlerts(id));
    }

    @PostMapping("/groups/{groupId}/commands")
    public ResponseEntity<Void> sendGroupCommand(
            @PathVariable Long groupId,
            @RequestBody @Valid SendCommandRequest request) {
        iotService.sendGroupCommand(groupId, request);
        return ResponseEntity.accepted().build();
    }
}
```

---

## โปรเจค 30: Live Dashboard API

### ภาพรวม
API สำหรับ Dashboard แบบ Real-time ที่รองรับการเก็บ Metrics, การรวมข้อมูลแบบ Real-time, การ Streaming ผ่าน SSE (Server-Sent Events), Widget ที่กำหนดค่าได้, ข้อมูล Time-series และการแจ้งเตือนเมื่อเกินขีดจำกัด (Alert Thresholds)

### Flyway Migration
```sql
-- V1__create_dashboard_tables.sql
CREATE TABLE dashboards (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    owner_id BIGINT NOT NULL,
    is_public BOOLEAN DEFAULT FALSE,
    refresh_interval INTEGER DEFAULT 30,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE widgets (
    id BIGSERIAL PRIMARY KEY,
    dashboard_id BIGINT REFERENCES dashboards(id) ON DELETE CASCADE,
    name VARCHAR(100) NOT NULL,
    type VARCHAR(50) NOT NULL,
    config JSONB NOT NULL,
    position_x INTEGER DEFAULT 0,
    position_y INTEGER DEFAULT 0,
    width INTEGER DEFAULT 4,
    height INTEGER DEFAULT 3,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE metrics (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    labels JSONB,
    value DOUBLE PRECISION NOT NULL,
    unit VARCHAR(50),
    recorded_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE alert_thresholds (
    id BIGSERIAL PRIMARY KEY,
    widget_id BIGINT REFERENCES widgets(id) ON DELETE CASCADE,
    metric_name VARCHAR(200) NOT NULL,
    operator VARCHAR(10) NOT NULL,
    threshold DOUBLE PRECISION NOT NULL,
    severity VARCHAR(20) DEFAULT 'WARNING',
    message TEXT,
    enabled BOOLEAN DEFAULT TRUE
);

CREATE INDEX idx_metrics_name_recorded ON metrics(name, recorded_at DESC);
CREATE INDEX idx_metrics_recorded_at ON metrics(recorded_at DESC);
```

### Entity & Service
```java
// Metric.java
@Entity
@Table(name = "metrics")
@Data
@NoArgsConstructor
public class Metric {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String name;

    @Type(JsonType.class)
    @Column(columnDefinition = "jsonb")
    private Map<String, String> labels;

    @Column(nullable = false)
    private Double value;

    private String unit;

    @Column(name = "recorded_at")
    private LocalDateTime recordedAt = LocalDateTime.now();
}

// DashboardService.java
@Service
@RequiredArgsConstructor
@Slf4j
public class DashboardService {

    private final MetricRepository metricRepository;
    private final DashboardRepository dashboardRepository;
    private final WidgetRepository widgetRepository;
    private final AlertThresholdRepository thresholdRepository;
    private final List<SseEmitter> emitters = new CopyOnWriteArrayList<>();

    public void recordMetric(String name, double value, Map<String, String> labels) {
        Metric metric = new Metric();
        metric.setName(name);
        metric.setValue(value);
        metric.setLabels(labels);
        metricRepository.save(metric);

        // Check thresholds
        checkThresholds(name, value);

        // Push to SSE clients
        broadcastMetricUpdate(name, value, labels);
    }

    public SseEmitter subscribe(Long dashboardId) {
        SseEmitter emitter = new SseEmitter(Long.MAX_VALUE);
        emitters.add(emitter);

        emitter.onCompletion(() -> emitters.remove(emitter));
        emitter.onTimeout(() -> emitters.remove(emitter));
        emitter.onError(e -> emitters.remove(emitter));

        // Send initial data
        try {
            Dashboard dashboard = dashboardRepository.findById(dashboardId)
                    .orElseThrow(() -> new ResourceNotFoundException("Dashboard not found"));
            DashboardSnapshot snapshot = buildSnapshot(dashboard);
            emitter.send(SseEmitter.event().name("snapshot").data(snapshot));
        } catch (IOException e) {
            emitters.remove(emitter);
        }

        return emitter;
    }

    private void broadcastMetricUpdate(String name, double value, Map<String, String> labels) {
        MetricUpdate update = new MetricUpdate(name, value, labels, LocalDateTime.now());
        List<SseEmitter> deadEmitters = new ArrayList<>();

        for (SseEmitter emitter : emitters) {
            try {
                emitter.send(SseEmitter.event()
                        .name("metric")
                        .data(update));
            } catch (IOException e) {
                deadEmitters.add(emitter);
            }
        }
        emitters.removeAll(deadEmitters);
    }

    public AggregatedMetrics getAggregatedMetrics(String metricName,
                                                    LocalDateTime from,
                                                    LocalDateTime to,
                                                    String aggregation,
                                                    String interval) {
        List<MetricPoint> raw = metricRepository.findByNameAndTimeRange(metricName, from, to);
        return aggregate(raw, aggregation, interval);
    }

    private AggregatedMetrics aggregate(List<MetricPoint> points,
                                         String aggregation,
                                         String interval) {
        Map<LocalDateTime, List<Double>> grouped = points.stream()
                .collect(Collectors.groupingBy(
                        p -> truncateToInterval(p.getRecordedAt(), interval),
                        Collectors.mapping(MetricPoint::getValue, Collectors.toList())));

        List<TimeSeriesPoint> series = grouped.entrySet().stream()
                .sorted(Map.Entry.comparingByKey())
                .map(e -> new TimeSeriesPoint(e.getKey(), computeAggregation(e.getValue(), aggregation)))
                .collect(Collectors.toList());

        return new AggregatedMetrics(series);
    }

    private double computeAggregation(List<Double> values, String aggregation) {
        return switch (aggregation.toUpperCase()) {
            case "AVG" -> values.stream().mapToDouble(Double::doubleValue).average().orElse(0);
            case "SUM" -> values.stream().mapToDouble(Double::doubleValue).sum();
            case "MAX" -> values.stream().mapToDouble(Double::doubleValue).max().orElse(0);
            case "MIN" -> values.stream().mapToDouble(Double::doubleValue).min().orElse(0);
            case "COUNT" -> values.size();
            default -> values.stream().mapToDouble(Double::doubleValue).average().orElse(0);
        };
    }

    private void checkThresholds(String metricName, double value) {
        thresholdRepository.findByMetricNameAndEnabled(metricName, true).forEach(threshold -> {
            boolean violated = checkOperator(value, threshold.getOperator(), threshold.getThreshold());
            if (violated) {
                log.warn("Threshold violated for metric {}: {} {} {}",
                        metricName, value, threshold.getOperator(), threshold.getThreshold());
                // Trigger alert notification
            }
        });
    }
}
```

### Controller
```java
// DashboardController.java
@RestController
@RequestMapping("/api/dashboards")
@RequiredArgsConstructor
public class DashboardController {

    private final DashboardService dashboardService;

    @GetMapping("/{id}/stream")
    public SseEmitter streamDashboard(@PathVariable Long id) {
        return dashboardService.subscribe(id);
    }

    @PostMapping("/metrics")
    public ResponseEntity<Void> recordMetric(@RequestBody @Valid RecordMetricRequest request) {
        dashboardService.recordMetric(request.getName(), request.getValue(), request.getLabels());
        return ResponseEntity.ok().build();
    }

    @PostMapping("/metrics/batch")
    public ResponseEntity<Void> recordMetricBatch(@RequestBody List<RecordMetricRequest> requests) {
        requests.forEach(r -> dashboardService.recordMetric(r.getName(), r.getValue(), r.getLabels()));
        return ResponseEntity.ok().build();
    }

    @GetMapping("/metrics/{name}/aggregate")
    public ResponseEntity<AggregatedMetrics> getAggregatedMetrics(
            @PathVariable String name,
            @RequestParam @DateTimeFormat(iso = DateTimeFormat.ISO.DATE_TIME) LocalDateTime from,
            @RequestParam @DateTimeFormat(iso = DateTimeFormat.ISO.DATE_TIME) LocalDateTime to,
            @RequestParam(defaultValue = "AVG") String aggregation,
            @RequestParam(defaultValue = "1h") String interval) {
        return ResponseEntity.ok(dashboardService.getAggregatedMetrics(name, from, to, aggregation, interval));
    }

    @PostMapping
    public ResponseEntity<DashboardDTO> createDashboard(
            @RequestBody @Valid CreateDashboardRequest request, Authentication auth) {
        return ResponseEntity.status(HttpStatus.CREATED)
                .body(dashboardService.createDashboard(request, getCurrentUserId(auth)));
    }

    @PostMapping("/{dashboardId}/widgets")
    public ResponseEntity<WidgetDTO> addWidget(
            @PathVariable Long dashboardId,
            @RequestBody @Valid CreateWidgetRequest request) {
        return ResponseEntity.status(HttpStatus.CREATED)
                .body(dashboardService.addWidget(dashboardId, request));
    }

    @GetMapping("/{id}")
    public ResponseEntity<DashboardDetailDTO> getDashboard(@PathVariable Long id) {
        return ResponseEntity.ok(dashboardService.getDashboardDetail(id));
    }
}
```

### Docker Compose (สำหรับทุกโปรเจคในส่วนนี้)
```yaml
# docker-compose.yml (Part 106)
version: '3.8'
services:
  app:
    build: .
    ports:
      - "8080:8080"
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/appdb
      SPRING_REDIS_HOST: redis
      SPRING_RABBITMQ_HOST: rabbitmq
    depends_on:
      - postgres
      - redis
      - rabbitmq

  postgres:
    image: postgres:15
    environment:
      POSTGRES_DB: appdb
      POSTGRES_USER: appuser
      POSTGRES_PASSWORD: apppass
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    command: redis-server --appendonly yes
    volumes:
      - redis_data:/data

  rabbitmq:
    image: rabbitmq:3-management
    ports:
      - "5672:5672"
      - "15672:15672"
    environment:
      RABBITMQ_DEFAULT_USER: admin
      RABBITMQ_DEFAULT_PASS: admin123

volumes:
  postgres_data:
  redis_data:
```

---

## สรุป Part 106

| โปรเจค | เทคโนโลยีหลัก | ความซับซ้อน |
|--------|--------------|------------|
| 26. Real-time Chat | WebSocket, STOMP, Redis | สูง |
| 27. Ride Hailing | Geolocation, State Machine | สูงมาก |
| 28. Notification Service | Multi-channel, Scheduling | สูง |
| 29. IoT Management | MQTT, Telemetry, Alerts | สูงมาก |
| 30. Live Dashboard | SSE, Time-series, Aggregation | สูง |

---

## Navigation

- [← Part 105](part-105-advanced-patterns.md)
- [Part 107: Developer Tools →](part-107-developer-tools.md)
- [กลับหน้าหลัก](README.md)

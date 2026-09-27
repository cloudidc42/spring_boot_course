# Part 110: โปรเจค 46-50 — Logistics & Tracking

> **ระดับ:** ระดับโลก | **เวลาเรียนรู้:** 10-15 ชั่วโมง | **โปรเจค:** 46-50

---

## โปรเจค 46: Logistics & Delivery Tracking

### ภาพรวม
ระบบ Logistics และติดตามการจัดส่งแบบครบวงจร รองรับการจัดการ Shipments, Carriers, Tracking Events, สถานะการจัดส่ง (Delivery Status), หลักฐานการจัดส่ง (Proof of Delivery), การเพิ่มประสิทธิภาพเส้นทาง (Route Optimization) และการแจ้งเตือนลูกค้า

### Dependencies (pom.xml)
```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-redis</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-mail</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-websocket</artifactId>
    </dependency>
    <dependency>
        <groupId>com.google.maps</groupId>
        <artifactId>google-maps-services</artifactId>
        <version>2.2.0</version>
    </dependency>
</dependencies>
```

### Flyway Migration
```sql
-- V1__create_logistics_tables.sql
CREATE TABLE carriers (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    code VARCHAR(20) UNIQUE NOT NULL,
    tracking_url_template VARCHAR(500),
    api_endpoint VARCHAR(500),
    api_key_encrypted VARCHAR(500),
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE shipments (
    id BIGSERIAL PRIMARY KEY,
    tracking_number VARCHAR(100) UNIQUE NOT NULL,
    carrier_id BIGINT REFERENCES carriers(id),
    sender_name VARCHAR(200),
    sender_address TEXT,
    sender_phone VARCHAR(20),
    recipient_name VARCHAR(200) NOT NULL,
    recipient_address TEXT NOT NULL,
    recipient_phone VARCHAR(20),
    recipient_email VARCHAR(200),
    origin_city VARCHAR(100),
    destination_city VARCHAR(100),
    status VARCHAR(30) DEFAULT 'CREATED',
    weight DECIMAL(10,3),
    dimensions JSONB,
    declared_value DECIMAL(12,2),
    service_type VARCHAR(50),
    estimated_delivery DATE,
    actual_delivery TIMESTAMP,
    instructions TEXT,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE tracking_events (
    id BIGSERIAL PRIMARY KEY,
    shipment_id BIGINT REFERENCES shipments(id) ON DELETE CASCADE,
    event_code VARCHAR(50),
    description VARCHAR(500) NOT NULL,
    location VARCHAR(200),
    location_lat DECIMAL(10,8),
    location_lng DECIMAL(11,8),
    event_time TIMESTAMP NOT NULL,
    recorded_by VARCHAR(100),
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE proof_of_delivery (
    id BIGSERIAL PRIMARY KEY,
    shipment_id BIGINT UNIQUE REFERENCES shipments(id),
    received_by VARCHAR(200),
    signature_image_url VARCHAR(500),
    photo_urls JSONB,
    delivery_lat DECIMAL(10,8),
    delivery_lng DECIMAL(11,8),
    notes TEXT,
    delivered_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE delivery_notifications (
    id BIGSERIAL PRIMARY KEY,
    shipment_id BIGINT REFERENCES shipments(id),
    channel VARCHAR(20) NOT NULL,
    recipient VARCHAR(200) NOT NULL,
    message TEXT NOT NULL,
    status VARCHAR(20) DEFAULT 'PENDING',
    sent_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_shipments_tracking ON shipments(tracking_number);
CREATE INDEX idx_tracking_events_shipment ON tracking_events(shipment_id, event_time DESC);
```

### Entity Classes
```java
// Shipment.java
@Entity
@Table(name = "shipments")
@Data
@NoArgsConstructor
public class Shipment {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "tracking_number", unique = true, nullable = false)
    private String trackingNumber;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "carrier_id")
    private Carrier carrier;

    @Column(name = "recipient_name", nullable = false)
    private String recipientName;

    @Column(name = "recipient_address", columnDefinition = "TEXT")
    private String recipientAddress;

    @Column(name = "recipient_email")
    private String recipientEmail;

    @Enumerated(EnumType.STRING)
    private ShipmentStatus status = ShipmentStatus.CREATED;

    @Column(name = "estimated_delivery")
    private LocalDate estimatedDelivery;

    @Column(name = "actual_delivery")
    private LocalDateTime actualDelivery;

    @OneToMany(mappedBy = "shipment", cascade = CascadeType.ALL, orphanRemoval = true)
    @OrderBy("eventTime DESC")
    private List<TrackingEvent> trackingEvents = new ArrayList<>();

    @Column(name = "created_at")
    private LocalDateTime createdAt = LocalDateTime.now();

    public enum ShipmentStatus {
        CREATED, PICKED_UP, IN_TRANSIT, OUT_FOR_DELIVERY,
        DELIVERED, FAILED_DELIVERY, RETURNED, CANCELLED
    }
}

// TrackingEvent.java
@Entity
@Table(name = "tracking_events")
@Data
@NoArgsConstructor
public class TrackingEvent {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "shipment_id", nullable = false)
    private Shipment shipment;

    @Column(name = "event_code")
    private String eventCode;

    @Column(nullable = false)
    private String description;

    private String location;

    @Column(name = "location_lat", precision = 10, scale = 8)
    private BigDecimal locationLat;

    @Column(name = "location_lng", precision = 11, scale = 8)
    private BigDecimal locationLng;

    @Column(name = "event_time", nullable = false)
    private LocalDateTime eventTime;

    @Column(name = "recorded_by")
    private String recordedBy;
}
```

### Service
```java
// ShipmentTrackingService.java
@Service
@RequiredArgsConstructor
@Slf4j
public class ShipmentTrackingService {

    private final ShipmentRepository shipmentRepository;
    private final TrackingEventRepository eventRepository;
    private final ProofOfDeliveryRepository podRepository;
    private final NotificationService notificationService;
    private final SimpMessagingTemplate wsTemplate;
    private final RedisTemplate<String, Object> redisTemplate;

    private static final String TRACKING_CACHE_PREFIX = "tracking:";

    @Transactional
    public Shipment createShipment(CreateShipmentRequest request) {
        Shipment shipment = new Shipment();
        shipment.setTrackingNumber(generateTrackingNumber(request.getCarrierCode()));
        shipment.setCarrier(carrierRepository.findByCode(request.getCarrierCode())
                .orElseThrow(() -> new ResourceNotFoundException("Carrier not found")));
        shipment.setSenderName(request.getSenderName());
        shipment.setSenderAddress(request.getSenderAddress());
        shipment.setRecipientName(request.getRecipientName());
        shipment.setRecipientAddress(request.getRecipientAddress());
        shipment.setRecipientEmail(request.getRecipientEmail());
        shipment.setRecipientPhone(request.getRecipientPhone());
        shipment.setOriginCity(request.getOriginCity());
        shipment.setDestinationCity(request.getDestinationCity());
        shipment.setServiceType(request.getServiceType());
        shipment.setEstimatedDelivery(calculateEstimatedDelivery(request));
        shipment.setWeight(request.getWeight());

        Shipment saved = shipmentRepository.save(shipment);

        // Add initial tracking event
        addTrackingEvent(saved.getId(), new AddTrackingEventRequest(
                "SHIPMENT_CREATED", "Shipment created and awaiting pickup",
                request.getOriginCity(), null, null, LocalDateTime.now(), "SYSTEM"));

        // Send notification to recipient
        if (saved.getRecipientEmail() != null) {
            notificationService.sendShipmentCreatedNotification(saved);
        }

        return saved;
    }

    @Transactional
    public TrackingEvent addTrackingEvent(Long shipmentId, AddTrackingEventRequest request) {
        Shipment shipment = shipmentRepository.findById(shipmentId)
                .orElseThrow(() -> new ResourceNotFoundException("Shipment not found"));

        TrackingEvent event = new TrackingEvent();
        event.setShipment(shipment);
        event.setEventCode(request.getEventCode());
        event.setDescription(request.getDescription());
        event.setLocation(request.getLocation());
        event.setLocationLat(request.getLat());
        event.setLocationLng(request.getLng());
        event.setEventTime(request.getEventTime());
        event.setRecordedBy(request.getRecordedBy());

        TrackingEvent saved = eventRepository.save(event);

        // Update shipment status based on event code
        updateShipmentStatus(shipment, request.getEventCode());

        // Invalidate cache
        redisTemplate.delete(TRACKING_CACHE_PREFIX + shipment.getTrackingNumber());

        // Broadcast real-time update
        wsTemplate.convertAndSend("/topic/tracking." + shipment.getTrackingNumber(), event);

        // Send notification
        sendStatusUpdateNotification(shipment, event);

        return saved;
    }

    public ProofOfDelivery recordDelivery(Long shipmentId, RecordDeliveryRequest request) {
        Shipment shipment = shipmentRepository.findById(shipmentId)
                .orElseThrow(() -> new ResourceNotFoundException("Shipment not found"));

        ProofOfDelivery pod = new ProofOfDelivery();
        pod.setShipment(shipment);
        pod.setReceivedBy(request.getReceivedBy());
        pod.setSignatureImageUrl(request.getSignatureImageUrl());
        pod.setPhotoUrls(request.getPhotoUrls());
        pod.setDeliveryLat(request.getLat());
        pod.setDeliveryLng(request.getLng());
        pod.setNotes(request.getNotes());

        ProofOfDelivery saved = podRepository.save(pod);

        // Update shipment status to DELIVERED
        shipment.setStatus(Shipment.ShipmentStatus.DELIVERED);
        shipment.setActualDelivery(LocalDateTime.now());
        shipmentRepository.save(shipment);

        // Add tracking event
        addTrackingEvent(shipmentId, new AddTrackingEventRequest(
                "DELIVERED", "Package delivered to " + request.getReceivedBy(),
                shipment.getDestinationCity(), request.getLat(), request.getLng(),
                LocalDateTime.now(), "DRIVER"));

        return saved;
    }

    public ShipmentTracking getTracking(String trackingNumber) {
        // Check cache
        Object cached = redisTemplate.opsForValue().get(TRACKING_CACHE_PREFIX + trackingNumber);
        if (cached instanceof ShipmentTracking st) return st;

        Shipment shipment = shipmentRepository.findByTrackingNumber(trackingNumber)
                .orElseThrow(() -> new ResourceNotFoundException("Tracking number not found: " + trackingNumber));

        List<TrackingEvent> events = eventRepository.findByShipmentId(shipment.getId());
        ShipmentTracking tracking = new ShipmentTracking(shipment, events);

        // Cache for 5 minutes
        redisTemplate.opsForValue().set(TRACKING_CACHE_PREFIX + trackingNumber,
                tracking, Duration.ofMinutes(5));

        return tracking;
    }

    private void updateShipmentStatus(Shipment shipment, String eventCode) {
        Shipment.ShipmentStatus newStatus = switch (eventCode) {
            case "PICKED_UP" -> Shipment.ShipmentStatus.PICKED_UP;
            case "IN_TRANSIT", "ARRIVED_AT_FACILITY" -> Shipment.ShipmentStatus.IN_TRANSIT;
            case "OUT_FOR_DELIVERY" -> Shipment.ShipmentStatus.OUT_FOR_DELIVERY;
            case "DELIVERED" -> Shipment.ShipmentStatus.DELIVERED;
            case "DELIVERY_FAILED" -> Shipment.ShipmentStatus.FAILED_DELIVERY;
            default -> shipment.getStatus();
        };

        if (newStatus != shipment.getStatus()) {
            shipment.setStatus(newStatus);
            shipmentRepository.save(shipment);
        }
    }

    private String generateTrackingNumber(String carrierCode) {
        String timestamp = String.valueOf(System.currentTimeMillis()).substring(3);
        String random = String.format("%06d", new Random().nextInt(999999));
        return carrierCode.toUpperCase() + timestamp + random;
    }
}
```

### Controller
```java
// ShipmentController.java
@RestController
@RequestMapping("/api/shipments")
@RequiredArgsConstructor
public class ShipmentController {

    private final ShipmentTrackingService trackingService;

    @PostMapping
    public ResponseEntity<ShipmentDTO> createShipment(@RequestBody @Valid CreateShipmentRequest request) {
        return ResponseEntity.status(HttpStatus.CREATED)
                .body(ShipmentDTO.from(trackingService.createShipment(request)));
    }

    @GetMapping("/{trackingNumber}/track")
    public ResponseEntity<ShipmentTrackingDTO> track(@PathVariable String trackingNumber) {
        return ResponseEntity.ok(ShipmentTrackingDTO.from(trackingService.getTracking(trackingNumber)));
    }

    @PostMapping("/{id}/events")
    public ResponseEntity<TrackingEventDTO> addEvent(
            @PathVariable Long id,
            @RequestBody @Valid AddTrackingEventRequest request) {
        return ResponseEntity.status(HttpStatus.CREATED)
                .body(TrackingEventDTO.from(trackingService.addTrackingEvent(id, request)));
    }

    @PostMapping("/{id}/delivery")
    public ResponseEntity<ProofOfDeliveryDTO> recordDelivery(
            @PathVariable Long id,
            @RequestBody @Valid RecordDeliveryRequest request) {
        return ResponseEntity.ok(ProofOfDeliveryDTO.from(trackingService.recordDelivery(id, request)));
    }

    @GetMapping("/{id}/proof-of-delivery")
    public ResponseEntity<ProofOfDeliveryDTO> getProofOfDelivery(@PathVariable Long id) {
        return ResponseEntity.ok(ProofOfDeliveryDTO.from(trackingService.getProofOfDelivery(id)));
    }
}
```

---

## โปรเจค 47: Fleet Management System

### ภาพรวม
ระบบจัดการยานพาหนะ (Fleet) ครบวงจรรองรับรถยนต์, คนขับ, เส้นทาง (Routes), การติดตาม GPS (Last Known Position), กำหนดการบำรุงรักษา (Maintenance Schedules), บันทึกการใช้เชื้อเพลิง (Fuel Logs) และรายงานการเดินทาง (Trip Reports)

### Flyway Migration
```sql
-- V1__create_fleet_tables.sql
CREATE TABLE fleet_drivers (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    employee_id VARCHAR(50) UNIQUE NOT NULL,
    phone VARCHAR(20),
    email VARCHAR(100) UNIQUE,
    license_number VARCHAR(50),
    license_expiry DATE,
    status VARCHAR(20) DEFAULT 'AVAILABLE',
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE vehicles (
    id BIGSERIAL PRIMARY KEY,
    plate_number VARCHAR(20) UNIQUE NOT NULL,
    make VARCHAR(50),
    model VARCHAR(50),
    year INTEGER,
    vin VARCHAR(50) UNIQUE,
    vehicle_type VARCHAR(30),
    fuel_type VARCHAR(20),
    capacity INTEGER,
    status VARCHAR(20) DEFAULT 'AVAILABLE',
    current_driver_id BIGINT REFERENCES fleet_drivers(id),
    last_lat DECIMAL(10,8),
    last_lng DECIMAL(11,8),
    last_location_update TIMESTAMP,
    odometer DECIMAL(10,2) DEFAULT 0,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE routes (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    description TEXT,
    waypoints JSONB,
    total_distance DECIMAL(10,2),
    estimated_duration INTEGER,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE fleet_trips (
    id BIGSERIAL PRIMARY KEY,
    vehicle_id BIGINT REFERENCES vehicles(id),
    driver_id BIGINT REFERENCES fleet_drivers(id),
    route_id BIGINT REFERENCES routes(id),
    purpose TEXT,
    start_location VARCHAR(500),
    end_location VARCHAR(500),
    start_odometer DECIMAL(10,2),
    end_odometer DECIMAL(10,2),
    distance_driven DECIMAL(10,2),
    status VARCHAR(20) DEFAULT 'PLANNED',
    started_at TIMESTAMP,
    completed_at TIMESTAMP,
    notes TEXT,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE maintenance_schedules (
    id BIGSERIAL PRIMARY KEY,
    vehicle_id BIGINT REFERENCES vehicles(id),
    maintenance_type VARCHAR(100) NOT NULL,
    description TEXT,
    interval_km DECIMAL(10,2),
    interval_days INTEGER,
    last_performed_at TIMESTAMP,
    last_odometer DECIMAL(10,2),
    next_due_at TIMESTAMP,
    next_due_odometer DECIMAL(10,2),
    status VARCHAR(20) DEFAULT 'UPCOMING',
    priority VARCHAR(10) DEFAULT 'NORMAL'
);

CREATE TABLE fuel_logs (
    id BIGSERIAL PRIMARY KEY,
    vehicle_id BIGINT REFERENCES vehicles(id),
    driver_id BIGINT REFERENCES fleet_drivers(id),
    fuel_type VARCHAR(20),
    liters DECIMAL(8,3) NOT NULL,
    cost_per_liter DECIMAL(8,4),
    total_cost DECIMAL(10,2),
    odometer DECIMAL(10,2),
    station_name VARCHAR(200),
    location VARCHAR(500),
    fueled_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_fleet_trips_vehicle ON fleet_trips(vehicle_id, started_at DESC);
CREATE INDEX idx_maintenance_next_due ON maintenance_schedules(next_due_at, status);
```

### Service
```java
// FleetManagementService.java
@Service
@RequiredArgsConstructor
@Slf4j
public class FleetManagementService {

    private final VehicleRepository vehicleRepository;
    private final FleetTripRepository tripRepository;
    private final MaintenanceScheduleRepository maintenanceRepository;
    private final FuelLogRepository fuelLogRepository;
    private final RedisTemplate<String, Object> redisTemplate;

    private static final String VEHICLE_LOCATION_KEY = "fleet:location:";

    public void updateVehicleLocation(Long vehicleId, BigDecimal lat, BigDecimal lng) {
        Vehicle vehicle = vehicleRepository.findById(vehicleId)
                .orElseThrow(() -> new ResourceNotFoundException("Vehicle not found"));

        vehicle.setLastLat(lat);
        vehicle.setLastLng(lng);
        vehicle.setLastLocationUpdate(LocalDateTime.now());
        vehicleRepository.save(vehicle);

        // Cache in Redis for fast access
        LocationData location = new LocationData(vehicleId, lat, lng, LocalDateTime.now());
        redisTemplate.opsForValue().set(VEHICLE_LOCATION_KEY + vehicleId,
                location, Duration.ofMinutes(10));
    }

    public FleetTrip startTrip(Long vehicleId, Long driverId, StartTripRequest request) {
        Vehicle vehicle = vehicleRepository.findById(vehicleId)
                .orElseThrow(() -> new ResourceNotFoundException("Vehicle not found"));

        if (vehicle.getStatus() == Vehicle.VehicleStatus.IN_USE) {
            throw new BusinessException("Vehicle is already in use");
        }

        FleetDriver driver = driverRepository.findById(driverId)
                .orElseThrow(() -> new ResourceNotFoundException("Driver not found"));

        FleetTrip trip = new FleetTrip();
        trip.setVehicle(vehicle);
        trip.setDriver(driver);
        trip.setStartLocation(request.getStartLocation());
        trip.setStartOdometer(vehicle.getOdometer());
        trip.setPurpose(request.getPurpose());
        trip.setStatus(FleetTrip.TripStatus.IN_PROGRESS);
        trip.setStartedAt(LocalDateTime.now());

        if (request.getRouteId() != null) {
            trip.setRoute(new Route(request.getRouteId()));
        }

        FleetTrip saved = tripRepository.save(trip);

        vehicle.setStatus(Vehicle.VehicleStatus.IN_USE);
        vehicle.setCurrentDriver(driver);
        vehicleRepository.save(vehicle);

        return saved;
    }

    public FleetTrip completeTrip(Long tripId, CompleteTripRequest request) {
        FleetTrip trip = tripRepository.findById(tripId)
                .orElseThrow(() -> new ResourceNotFoundException("Trip not found"));

        Vehicle vehicle = trip.getVehicle();
        BigDecimal distanceDriven = request.getEndOdometer().subtract(trip.getStartOdometer());

        trip.setEndLocation(request.getEndLocation());
        trip.setEndOdometer(request.getEndOdometer());
        trip.setDistanceDriven(distanceDriven);
        trip.setStatus(FleetTrip.TripStatus.COMPLETED);
        trip.setCompletedAt(LocalDateTime.now());
        trip.setNotes(request.getNotes());

        vehicle.setOdometer(request.getEndOdometer());
        vehicle.setStatus(Vehicle.VehicleStatus.AVAILABLE);
        vehicle.setCurrentDriver(null);
        vehicleRepository.save(vehicle);

        // Check if any maintenance is now due
        checkMaintenanceDue(vehicle);

        return tripRepository.save(trip);
    }

    public FuelLog recordFuel(Long vehicleId, RecordFuelRequest request) {
        Vehicle vehicle = vehicleRepository.findById(vehicleId)
                .orElseThrow(() -> new ResourceNotFoundException("Vehicle not found"));

        FuelLog log = new FuelLog();
        log.setVehicle(vehicle);
        log.setDriver(request.getDriverId() != null ? new FleetDriver(request.getDriverId()) : null);
        log.setFuelType(request.getFuelType());
        log.setLiters(request.getLiters());
        log.setCostPerLiter(request.getCostPerLiter());
        log.setTotalCost(request.getLiters().multiply(request.getCostPerLiter()));
        log.setOdometer(request.getOdometer());
        log.setStationName(request.getStationName());
        log.setFueledAt(request.getFueledAt() != null ? request.getFueledAt() : LocalDateTime.now());

        // Update vehicle odometer if higher than current
        if (request.getOdometer().compareTo(vehicle.getOdometer()) > 0) {
            vehicle.setOdometer(request.getOdometer());
            vehicleRepository.save(vehicle);
        }

        return fuelLogRepository.save(log);
    }

    public FleetReport generateFleetReport(LocalDateTime from, LocalDateTime to) {
        List<Vehicle> vehicles = vehicleRepository.findAll();
        List<VehicleReport> vehicleReports = vehicles.stream().map(vehicle -> {
            List<FleetTrip> trips = tripRepository.findByVehicleIdAndDateRange(vehicle.getId(), from, to);
            List<FuelLog> fuelLogs = fuelLogRepository.findByVehicleIdAndDateRange(vehicle.getId(), from, to);

            BigDecimal totalDistance = trips.stream()
                    .filter(t -> t.getDistanceDriven() != null)
                    .map(FleetTrip::getDistanceDriven)
                    .reduce(BigDecimal.ZERO, BigDecimal::add);

            BigDecimal totalFuelCost = fuelLogs.stream()
                    .filter(f -> f.getTotalCost() != null)
                    .map(FuelLog::getTotalCost)
                    .reduce(BigDecimal.ZERO, BigDecimal::add);

            BigDecimal totalFuelLiters = fuelLogs.stream()
                    .map(FuelLog::getLiters)
                    .reduce(BigDecimal.ZERO, BigDecimal::add);

            return new VehicleReport(vehicle, trips.size(), totalDistance, totalFuelLiters, totalFuelCost);
        }).collect(Collectors.toList());

        return new FleetReport(from, to, vehicleReports);
    }

    private void checkMaintenanceDue(Vehicle vehicle) {
        List<MaintenanceSchedule> schedules = maintenanceRepository.findByVehicleId(vehicle.getId());
        schedules.forEach(schedule -> {
            boolean odometerDue = schedule.getNextDueOdometer() != null
                    && vehicle.getOdometer().compareTo(schedule.getNextDueOdometer()) >= 0;
            boolean timeDue = schedule.getNextDueAt() != null
                    && LocalDateTime.now().isAfter(schedule.getNextDueAt());

            if (odometerDue || timeDue) {
                schedule.setStatus(MaintenanceSchedule.MaintenanceStatus.OVERDUE);
                maintenanceRepository.save(schedule);
                log.warn("Maintenance OVERDUE for vehicle {} - {}", vehicle.getPlateNumber(),
                        schedule.getMaintenanceType());
            }
        });
    }
}
```

### Controller
```java
// FleetController.java
@RestController
@RequestMapping("/api/fleet")
@RequiredArgsConstructor
public class FleetController {

    private final FleetManagementService fleetService;

    @PutMapping("/vehicles/{id}/location")
    public ResponseEntity<Void> updateLocation(
            @PathVariable Long id,
            @RequestBody LocationUpdateRequest request) {
        fleetService.updateVehicleLocation(id, request.getLat(), request.getLng());
        return ResponseEntity.ok().build();
    }

    @PostMapping("/trips/start")
    public ResponseEntity<TripDTO> startTrip(@RequestBody @Valid StartTripRequest request) {
        return ResponseEntity.status(HttpStatus.CREATED)
                .body(TripDTO.from(fleetService.startTrip(request.getVehicleId(),
                        request.getDriverId(), request)));
    }

    @PostMapping("/trips/{id}/complete")
    public ResponseEntity<TripDTO> completeTrip(@PathVariable Long id,
                                                  @RequestBody @Valid CompleteTripRequest request) {
        return ResponseEntity.ok(TripDTO.from(fleetService.completeTrip(id, request)));
    }

    @PostMapping("/vehicles/{id}/fuel")
    public ResponseEntity<FuelLogDTO> recordFuel(@PathVariable Long id,
                                                  @RequestBody @Valid RecordFuelRequest request) {
        return ResponseEntity.status(HttpStatus.CREATED)
                .body(FuelLogDTO.from(fleetService.recordFuel(id, request)));
    }

    @GetMapping("/vehicles/{id}/maintenance")
    public ResponseEntity<List<MaintenanceDTO>> getMaintenanceSchedule(@PathVariable Long id) {
        return ResponseEntity.ok(fleetService.getMaintenanceSchedule(id).stream()
                .map(MaintenanceDTO::from).collect(Collectors.toList()));
    }

    @GetMapping("/reports")
    public ResponseEntity<FleetReportDTO> getReport(
            @RequestParam @DateTimeFormat(iso = DateTimeFormat.ISO.DATE_TIME) LocalDateTime from,
            @RequestParam @DateTimeFormat(iso = DateTimeFormat.ISO.DATE_TIME) LocalDateTime to) {
        return ResponseEntity.ok(FleetReportDTO.from(fleetService.generateFleetReport(from, to)));
    }
}
```

---

## โปรเจค 48: Asset Management System

### ภาพรวม
ระบบจัดการสินทรัพย์ (Assets) ครอบคลุม IT/อุปกรณ์/อสังหาริมทรัพย์ รองรับการมอบหมาย (Assignments), การคำนวณค่าเสื่อมราคา (Depreciation), บันทึกการบำรุงรักษา (Maintenance Records), ป้าย QR Code (QR Code Labels) และกระบวนการจำหน่าย (Disposal Workflow)

### Flyway Migration
```sql
-- V1__create_asset_management_tables.sql
CREATE TABLE asset_categories (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(100) UNIQUE NOT NULL,
    description TEXT,
    depreciation_method VARCHAR(20) DEFAULT 'STRAIGHT_LINE',
    useful_life_years INTEGER DEFAULT 5,
    salvage_value_percent DECIMAL(5,2) DEFAULT 10.00
);

CREATE TABLE assets (
    id BIGSERIAL PRIMARY KEY,
    asset_tag VARCHAR(50) UNIQUE NOT NULL,
    name VARCHAR(200) NOT NULL,
    category_id BIGINT REFERENCES asset_categories(id),
    serial_number VARCHAR(100),
    model VARCHAR(100),
    manufacturer VARCHAR(100),
    purchase_date DATE,
    purchase_cost DECIMAL(12,2),
    current_value DECIMAL(12,2),
    status VARCHAR(30) DEFAULT 'AVAILABLE',
    condition VARCHAR(20) DEFAULT 'GOOD',
    location VARCHAR(200),
    department VARCHAR(100),
    assigned_to BIGINT,
    qr_code_url VARCHAR(500),
    notes TEXT,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE asset_assignments (
    id BIGSERIAL PRIMARY KEY,
    asset_id BIGINT REFERENCES assets(id),
    assigned_to_id BIGINT NOT NULL,
    assigned_to_name VARCHAR(200),
    assigned_by_id BIGINT,
    assigned_at TIMESTAMP DEFAULT NOW(),
    returned_at TIMESTAMP,
    return_condition VARCHAR(20),
    notes TEXT
);

CREATE TABLE asset_maintenance (
    id BIGSERIAL PRIMARY KEY,
    asset_id BIGINT REFERENCES assets(id),
    maintenance_type VARCHAR(50) NOT NULL,
    description TEXT,
    performed_by VARCHAR(200),
    cost DECIMAL(10,2),
    performed_at TIMESTAMP NOT NULL,
    next_maintenance_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE asset_disposals (
    id BIGSERIAL PRIMARY KEY,
    asset_id BIGINT REFERENCES assets(id),
    disposal_type VARCHAR(30) NOT NULL,
    disposal_reason TEXT,
    disposal_value DECIMAL(10,2),
    disposal_date DATE,
    approved_by BIGINT,
    approved_at TIMESTAMP,
    status VARCHAR(20) DEFAULT 'PENDING',
    notes TEXT,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE depreciation_records (
    id BIGSERIAL PRIMARY KEY,
    asset_id BIGINT REFERENCES assets(id),
    fiscal_year INTEGER NOT NULL,
    book_value_start DECIMAL(12,2),
    depreciation_amount DECIMAL(12,2),
    book_value_end DECIMAL(12,2),
    recorded_at TIMESTAMP DEFAULT NOW(),
    UNIQUE(asset_id, fiscal_year)
);
```

### Service
```java
// AssetManagementService.java
@Service
@RequiredArgsConstructor
@Slf4j
public class AssetManagementService {

    private final AssetRepository assetRepository;
    private final AssetAssignmentRepository assignmentRepository;
    private final AssetMaintenanceRepository maintenanceRepository;
    private final AssetDisposalRepository disposalRepository;
    private final DepreciationRepository depreciationRepository;
    private final QrCodeService qrCodeService;
    private final S3Service s3Service;

    @Transactional
    public Asset createAsset(CreateAssetRequest request) {
        Asset asset = new Asset();
        asset.setAssetTag(generateAssetTag(request.getCategoryId()));
        asset.setName(request.getName());
        asset.setCategory(new AssetCategory(request.getCategoryId()));
        asset.setSerialNumber(request.getSerialNumber());
        asset.setModel(request.getModel());
        asset.setManufacturer(request.getManufacturer());
        asset.setPurchaseDate(request.getPurchaseDate());
        asset.setPurchaseCost(request.getPurchaseCost());
        asset.setCurrentValue(request.getPurchaseCost());
        asset.setLocation(request.getLocation());
        asset.setDepartment(request.getDepartment());
        asset.setStatus(Asset.AssetStatus.AVAILABLE);
        asset.setCondition(Asset.AssetCondition.GOOD);

        Asset saved = assetRepository.save(asset);

        // Generate QR Code
        try {
            String qrContent = "asset:" + saved.getAssetTag();
            byte[] qrBytes = qrCodeService.generateQrCode(qrContent, 300, "#000000", "#FFFFFF");
            String qrKey = "assets/qr/" + saved.getAssetTag() + ".png";
            String qrUrl = s3Service.upload(qrBytes, qrKey, "image/png");
            saved.setQrCodeUrl(qrUrl);
            saved = assetRepository.save(saved);
        } catch (Exception e) {
            log.warn("Failed to generate QR code for asset {}: {}", saved.getAssetTag(), e.getMessage());
        }

        return saved;
    }

    @Transactional
    public AssetAssignment assignAsset(Long assetId, AssignAssetRequest request, Long assignedBy) {
        Asset asset = assetRepository.findById(assetId)
                .orElseThrow(() -> new ResourceNotFoundException("Asset not found"));

        if (asset.getStatus() == Asset.AssetStatus.ASSIGNED) {
            throw new BusinessException("Asset is already assigned to someone");
        }
        if (asset.getStatus() != Asset.AssetStatus.AVAILABLE) {
            throw new BusinessException("Asset is not available for assignment (status: " + asset.getStatus() + ")");
        }

        AssetAssignment assignment = new AssetAssignment();
        assignment.setAsset(asset);
        assignment.setAssignedToId(request.getAssignedToId());
        assignment.setAssignedToName(request.getAssignedToName());
        assignment.setAssignedById(assignedBy);
        assignment.setNotes(request.getNotes());

        AssetAssignment saved = assignmentRepository.save(assignment);

        asset.setStatus(Asset.AssetStatus.ASSIGNED);
        asset.setAssignedTo(request.getAssignedToId());
        assetRepository.save(asset);

        return saved;
    }

    @Transactional
    public AssetAssignment returnAsset(Long assetId, ReturnAssetRequest request) {
        Asset asset = assetRepository.findById(assetId)
                .orElseThrow(() -> new ResourceNotFoundException("Asset not found"));

        AssetAssignment currentAssignment = assignmentRepository
                .findCurrentAssignment(assetId)
                .orElseThrow(() -> new ResourceNotFoundException("No active assignment found"));

        currentAssignment.setReturnedAt(LocalDateTime.now());
        currentAssignment.setReturnCondition(request.getCondition());
        currentAssignment.setNotes(request.getNotes());
        assignmentRepository.save(currentAssignment);

        asset.setStatus(Asset.AssetStatus.AVAILABLE);
        asset.setAssignedTo(null);
        asset.setCondition(Asset.AssetCondition.valueOf(request.getCondition()));
        assetRepository.save(asset);

        return currentAssignment;
    }

    public BigDecimal calculateDepreciation(Long assetId, int year) {
        Asset asset = assetRepository.findById(assetId)
                .orElseThrow(() -> new ResourceNotFoundException("Asset not found"));

        AssetCategory category = asset.getCategory();
        BigDecimal purchaseCost = asset.getPurchaseCost();
        BigDecimal salvageValue = purchaseCost.multiply(category.getSalvageValuePercent())
                .divide(BigDecimal.valueOf(100), 2, RoundingMode.HALF_UP);

        return switch (category.getDepreciationMethod()) {
            case "STRAIGHT_LINE" -> calculateStraightLine(purchaseCost, salvageValue,
                    category.getUsefulLifeYears());
            case "DECLINING_BALANCE" -> calculateDecliningBalance(asset, year, category.getUsefulLifeYears());
            default -> calculateStraightLine(purchaseCost, salvageValue, category.getUsefulLifeYears());
        };
    }

    private BigDecimal calculateStraightLine(BigDecimal cost, BigDecimal salvage, int lifeYears) {
        return cost.subtract(salvage).divide(BigDecimal.valueOf(lifeYears), 2, RoundingMode.HALF_UP);
    }

    @Scheduled(cron = "0 0 0 1 1 *") // Every Jan 1st
    public void recordAnnualDepreciation() {
        int currentYear = LocalDate.now().getYear() - 1; // Record for previous year
        List<Asset> activeAssets = assetRepository.findByStatusNot(Asset.AssetStatus.DISPOSED);

        activeAssets.forEach(asset -> {
            try {
                BigDecimal depreciation = calculateDepreciation(asset.getId(), currentYear);
                BigDecimal newValue = asset.getCurrentValue().subtract(depreciation)
                        .max(BigDecimal.ZERO);

                DepreciationRecord record = new DepreciationRecord();
                record.setAsset(asset);
                record.setFiscalYear(currentYear);
                record.setBookValueStart(asset.getCurrentValue());
                record.setDepreciationAmount(depreciation);
                record.setBookValueEnd(newValue);
                depreciationRepository.save(record);

                asset.setCurrentValue(newValue);
                assetRepository.save(asset);
            } catch (Exception e) {
                log.error("Failed to record depreciation for asset {}: {}", asset.getAssetTag(), e.getMessage());
            }
        });
    }
}
```

### Controller
```java
// AssetController.java
@RestController
@RequestMapping("/api/assets")
@RequiredArgsConstructor
public class AssetController {

    private final AssetManagementService assetService;

    @PostMapping
    public ResponseEntity<AssetDTO> createAsset(@RequestBody @Valid CreateAssetRequest request) {
        return ResponseEntity.status(HttpStatus.CREATED)
                .body(AssetDTO.from(assetService.createAsset(request)));
    }

    @PostMapping("/{id}/assign")
    public ResponseEntity<AssetAssignmentDTO> assignAsset(
            @PathVariable Long id,
            @RequestBody @Valid AssignAssetRequest request,
            Authentication auth) {
        return ResponseEntity.ok(AssetAssignmentDTO.from(
                assetService.assignAsset(id, request, getCurrentUserId(auth))));
    }

    @PostMapping("/{id}/return")
    public ResponseEntity<AssetAssignmentDTO> returnAsset(
            @PathVariable Long id,
            @RequestBody @Valid ReturnAssetRequest request) {
        return ResponseEntity.ok(AssetAssignmentDTO.from(assetService.returnAsset(id, request)));
    }

    @GetMapping("/{id}/qr-code")
    public ResponseEntity<byte[]> getQrCode(@PathVariable Long id) throws Exception {
        Asset asset = assetService.getAsset(id);
        byte[] qrBytes = assetService.getQrCode(asset.getAssetTag());
        return ResponseEntity.ok()
                .contentType(MediaType.IMAGE_PNG)
                .body(qrBytes);
    }

    @GetMapping("/scan/{assetTag}")
    public ResponseEntity<AssetDetailDTO> scanQrCode(@PathVariable String assetTag) {
        return ResponseEntity.ok(assetService.getAssetByTag(assetTag));
    }

    @PostMapping("/{id}/dispose")
    public ResponseEntity<AssetDisposalDTO> initiateDisposal(
            @PathVariable Long id,
            @RequestBody @Valid InitiateDisposalRequest request,
            Authentication auth) {
        return ResponseEntity.status(HttpStatus.CREATED)
                .body(AssetDisposalDTO.from(assetService.initiateDisposal(id, request, getCurrentUserId(auth))));
    }
}
```

---

## โปรเจค 49: QR Code Generator Service

### ภาพรวม
บริการสร้าง QR Code สำหรับ URLs, ข้อความ, vCards, WiFi; ปรับแต่งสี/โลโก้, การสร้างแบบจำนวนมาก (Bulk Generation), การติดตามการ Scan (Scan Tracking) และ Analytics

### Flyway Migration
```sql
-- V1__create_qrcode_tables.sql
CREATE TABLE qr_codes (
    id BIGSERIAL PRIMARY KEY,
    code VARCHAR(50) UNIQUE NOT NULL,
    content_type VARCHAR(20) NOT NULL,
    content TEXT NOT NULL,
    size INTEGER DEFAULT 300,
    foreground_color VARCHAR(7) DEFAULT '#000000',
    background_color VARCHAR(7) DEFAULT '#FFFFFF',
    error_correction VARCHAR(5) DEFAULT 'H',
    with_logo BOOLEAN DEFAULT FALSE,
    logo_url VARCHAR(500),
    image_url VARCHAR(500),
    scan_count INTEGER DEFAULT 0,
    owner_id BIGINT,
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE qr_scans (
    id BIGSERIAL PRIMARY KEY,
    qr_code_id BIGINT REFERENCES qr_codes(id) ON DELETE CASCADE,
    ip_address VARCHAR(45),
    user_agent TEXT,
    country_code VARCHAR(3),
    city VARCHAR(100),
    device_type VARCHAR(20),
    scanned_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE qr_bulk_jobs (
    id BIGSERIAL PRIMARY KEY,
    job_name VARCHAR(100),
    total_count INTEGER NOT NULL,
    generated_count INTEGER DEFAULT 0,
    status VARCHAR(20) DEFAULT 'PENDING',
    template JSONB,
    result_zip_url VARCHAR(500),
    created_by BIGINT,
    created_at TIMESTAMP DEFAULT NOW(),
    completed_at TIMESTAMP
);
```

### Service
```java
// QrCodeGeneratorService.java
@Service
@RequiredArgsConstructor
@Slf4j
public class QrCodeGeneratorService {

    private final QrCodeRepository qrCodeRepository;
    private final QrScanRepository scanRepository;
    private final QrBulkJobRepository bulkJobRepository;
    private final S3Service s3Service;
    private final GeoIpService geoIpService;

    public QrCode generateQrCode(GenerateQrRequest request, Long ownerId) throws Exception {
        String content = buildContent(request);
        byte[] qrImageBytes = renderQrCode(content, request);

        // Upload image to S3
        String code = generateCode();
        String imageKey = "qrcodes/" + code + ".png";
        String imageUrl = s3Service.upload(qrImageBytes, imageKey, "image/png");

        QrCode qrCode = new QrCode();
        qrCode.setCode(code);
        qrCode.setContentType(request.getContentType());
        qrCode.setContent(content);
        qrCode.setSize(request.getSize() != null ? request.getSize() : 300);
        qrCode.setForegroundColor(request.getForegroundColor() != null ? request.getForegroundColor() : "#000000");
        qrCode.setBackgroundColor(request.getBackgroundColor() != null ? request.getBackgroundColor() : "#FFFFFF");
        qrCode.setWithLogo(request.isWithLogo());
        qrCode.setLogoUrl(request.getLogoUrl());
        qrCode.setImageUrl(imageUrl);
        qrCode.setOwnerId(ownerId);

        return qrCodeRepository.save(qrCode);
    }

    private byte[] renderQrCode(String content, GenerateQrRequest request) throws Exception {
        int size = request.getSize() != null ? request.getSize() : 300;
        ErrorCorrectionLevel errorLevel = request.isWithLogo() ? ErrorCorrectionLevel.H : ErrorCorrectionLevel.M;

        Map<EncodeHintType, Object> hints = new EnumMap<>(EncodeHintType.class);
        hints.put(EncodeHintType.ERROR_CORRECTION, errorLevel);
        hints.put(EncodeHintType.MARGIN, 2);
        hints.put(EncodeHintType.CHARACTER_SET, "UTF-8");

        QRCodeWriter writer = new QRCodeWriter();
        BitMatrix matrix = writer.encode(content, BarcodeFormat.QR_CODE, size, size, hints);

        Color fg = Color.decode(request.getForegroundColor() != null ? request.getForegroundColor() : "#000000");
        Color bg = Color.decode(request.getBackgroundColor() != null ? request.getBackgroundColor() : "#FFFFFF");

        BufferedImage image = new BufferedImage(size, size, BufferedImage.TYPE_INT_RGB);
        for (int x = 0; x < size; x++) {
            for (int y = 0; y < size; y++) {
                image.setRGB(x, y, matrix.get(x, y) ? fg.getRGB() : bg.getRGB());
            }
        }

        // Overlay logo if requested
        if (request.isWithLogo() && request.getLogoUrl() != null) {
            image = overlayLogo(image, request.getLogoUrl(), size);
        }

        ByteArrayOutputStream baos = new ByteArrayOutputStream();
        ImageIO.write(image, "PNG", baos);
        return baos.toByteArray();
    }

    private String buildContent(GenerateQrRequest request) {
        return switch (request.getContentType().toUpperCase()) {
            case "URL" -> request.getUrl();
            case "TEXT" -> request.getText();
            case "VCARD" -> buildVCard(request.getVcard());
            case "WIFI" -> buildWifiString(request.getWifi());
            case "EMAIL" -> "mailto:" + request.getEmail().getAddress() + "?subject=" +
                            URLEncoder.encode(request.getEmail().getSubject(), StandardCharsets.UTF_8);
            case "PHONE" -> "tel:" + request.getPhone();
            default -> throw new BusinessException("Unsupported content type: " + request.getContentType());
        };
    }

    private String buildVCard(VCardData vcard) {
        return "BEGIN:VCARD\n" +
               "VERSION:3.0\n" +
               "FN:" + vcard.getFullName() + "\n" +
               "TEL:" + vcard.getPhone() + "\n" +
               "EMAIL:" + vcard.getEmail() + "\n" +
               (vcard.getOrganization() != null ? "ORG:" + vcard.getOrganization() + "\n" : "") +
               (vcard.getUrl() != null ? "URL:" + vcard.getUrl() + "\n" : "") +
               "END:VCARD";
    }

    private String buildWifiString(WifiData wifi) {
        return "WIFI:T:" + wifi.getSecurityType() + ";S:" + wifi.getSsid() + ";P:" + wifi.getPassword() + ";;";
    }

    public void recordScan(String code, HttpServletRequest request) {
        QrCode qrCode = qrCodeRepository.findByCode(code)
                .orElseThrow(() -> new ResourceNotFoundException("QR Code not found: " + code));

        String ip = extractIpAddress(request);
        QrScan scan = new QrScan();
        scan.setQrCode(qrCode);
        scan.setIpAddress(ip);
        scan.setUserAgent(request.getHeader("User-Agent"));

        GeoLocation location = geoIpService.lookup(ip);
        if (location != null) {
            scan.setCountryCode(location.getCountryCode());
            scan.setCity(location.getCity());
        }

        scanRepository.save(scan);
        qrCodeRepository.incrementScanCount(qrCode.getId());
    }

    @Async
    public void generateBulk(Long jobId) {
        QrBulkJob job = bulkJobRepository.findById(jobId)
                .orElseThrow(() -> new ResourceNotFoundException("Job not found"));

        job.setStatus("PROCESSING");
        bulkJobRepository.save(job);

        try {
            List<Map<String, Object>> items = (List<Map<String, Object>>) job.getTemplate().get("items");
            ByteArrayOutputStream zipBaos = new ByteArrayOutputStream();
            ZipOutputStream zipOut = new ZipOutputStream(zipBaos);

            for (int i = 0; i < items.size(); i++) {
                Map<String, Object> item = items.get(i);
                GenerateQrRequest req = mapToRequest(item);
                byte[] qrBytes = renderQrCode(buildContent(req), req);

                String fileName = "qr_" + (i + 1) + "_" + item.getOrDefault("name", "item") + ".png";
                ZipEntry entry = new ZipEntry(fileName);
                zipOut.putNextEntry(entry);
                zipOut.write(qrBytes);
                zipOut.closeEntry();

                job.setGeneratedCount(i + 1);
                bulkJobRepository.save(job);
            }

            zipOut.close();
            String zipKey = "qrcodes/bulk/" + jobId + ".zip";
            String zipUrl = s3Service.upload(zipBaos.toByteArray(), zipKey, "application/zip");

            job.setResultZipUrl(zipUrl);
            job.setStatus("COMPLETED");
            job.setCompletedAt(LocalDateTime.now());

        } catch (Exception e) {
            job.setStatus("FAILED");
            log.error("Bulk QR generation failed for job {}: {}", jobId, e.getMessage());
        }

        bulkJobRepository.save(job);
    }
}
```

### Controller
```java
// QrCodeController.java
@RestController
@RequestMapping("/api/qrcodes")
@RequiredArgsConstructor
public class QrCodeController {

    private final QrCodeGeneratorService qrService;

    @PostMapping("/generate")
    public ResponseEntity<QrCodeDTO> generate(@RequestBody @Valid GenerateQrRequest request,
                                               Authentication auth) throws Exception {
        Long ownerId = auth != null ? getCurrentUserId(auth) : null;
        return ResponseEntity.status(HttpStatus.CREATED)
                .body(QrCodeDTO.from(qrService.generateQrCode(request, ownerId)));
    }

    @GetMapping("/{code}/image")
    public ResponseEntity<byte[]> getImage(@PathVariable String code) {
        byte[] image = qrService.getQrImage(code);
        return ResponseEntity.ok()
                .contentType(MediaType.IMAGE_PNG)
                .body(image);
    }

    @GetMapping("/{code}/scan")
    public ResponseEntity<Void> trackScan(@PathVariable String code, HttpServletRequest request) {
        String content = qrService.getScanContent(code);
        qrService.recordScan(code, request);
        return ResponseEntity.status(HttpStatus.FOUND)
                .location(URI.create(content.startsWith("http") ? content : "https://" + content))
                .build();
    }

    @GetMapping("/{code}/analytics")
    public ResponseEntity<QrAnalyticsDTO> getAnalytics(@PathVariable String code) {
        return ResponseEntity.ok(qrService.getAnalytics(code));
    }

    @PostMapping("/bulk")
    public ResponseEntity<BulkJobDTO> createBulkJob(@RequestBody @Valid BulkQrRequest request,
                                                      Authentication auth) {
        return ResponseEntity.status(HttpStatus.ACCEPTED)
                .body(BulkJobDTO.from(qrService.createBulkJob(request, getCurrentUserId(auth))));
    }
}
```

---

## โปรเจค 50: Barcode & Label Service

### ภาพรวม
บริการสร้าง Barcode (Code128/EAN13/QR), Template สำหรับป้าย (Label Templates), การพิมพ์ PDF จำนวนมาก (Bulk Print PDF), กระบวนการติดฉลากสินค้า (Product Labeling Workflow) และ API สำหรับค้นหาสินค้าด้วยการ Scan

### Flyway Migration
```sql
-- V1__create_barcode_label_tables.sql
CREATE TABLE label_templates (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(100) UNIQUE NOT NULL,
    description TEXT,
    width_mm DECIMAL(6,2) NOT NULL,
    height_mm DECIMAL(6,2) NOT NULL,
    barcode_type VARCHAR(20) DEFAULT 'CODE_128',
    template_config JSONB NOT NULL,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE product_labels (
    id BIGSERIAL PRIMARY KEY,
    product_code VARCHAR(100) NOT NULL,
    product_name VARCHAR(200) NOT NULL,
    barcode_value VARCHAR(200) NOT NULL,
    barcode_type VARCHAR(20) DEFAULT 'CODE_128',
    label_data JSONB,
    template_id BIGINT REFERENCES label_templates(id),
    barcode_image_url VARCHAR(500),
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE barcode_scan_log (
    id BIGSERIAL PRIMARY KEY,
    barcode_value VARCHAR(200) NOT NULL,
    barcode_type VARCHAR(20),
    scanned_by VARCHAR(100),
    scan_location VARCHAR(200),
    scan_result JSONB,
    scanned_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE print_jobs (
    id BIGSERIAL PRIMARY KEY,
    template_id BIGINT REFERENCES label_templates(id),
    label_ids JSONB,
    total_labels INTEGER,
    status VARCHAR(20) DEFAULT 'PENDING',
    pdf_url VARCHAR(500),
    created_by BIGINT,
    created_at TIMESTAMP DEFAULT NOW(),
    completed_at TIMESTAMP
);
```

### Service
```java
// BarcodeService.java
@Service
@RequiredArgsConstructor
@Slf4j
public class BarcodeService {

    private final LabelTemplateRepository templateRepository;
    private final ProductLabelRepository labelRepository;
    private final PrintJobRepository printJobRepository;
    private final BarcodeScanLogRepository scanLogRepository;
    private final S3Service s3Service;

    public byte[] generateBarcode(String value, String barcodeType, int width, int height) throws Exception {
        BarcodeFormat format = switch (barcodeType.toUpperCase()) {
            case "CODE_128" -> BarcodeFormat.CODE_128;
            case "EAN_13" -> BarcodeFormat.EAN_13;
            case "EAN_8" -> BarcodeFormat.EAN_8;
            case "QR_CODE" -> BarcodeFormat.QR_CODE;
            case "CODE_39" -> BarcodeFormat.CODE_39;
            case "UPC_A" -> BarcodeFormat.UPC_A;
            case "ITF" -> BarcodeFormat.ITF;
            default -> BarcodeFormat.CODE_128;
        };

        Map<EncodeHintType, Object> hints = new HashMap<>();
        hints.put(EncodeHintType.MARGIN, 5);

        MultiFormatWriter writer = new MultiFormatWriter();
        BitMatrix matrix = writer.encode(value, format, width, height, hints);

        BufferedImage image = MatrixToImageWriter.toBufferedImage(matrix);
        ByteArrayOutputStream baos = new ByteArrayOutputStream();
        ImageIO.write(image, "PNG", baos);
        return baos.toByteArray();
    }

    public ProductLabel createProductLabel(CreateProductLabelRequest request) throws Exception {
        byte[] barcodeBytes = generateBarcode(request.getBarcodeValue(),
                request.getBarcodeType(), 400, 150);

        String barcodeKey = "barcodes/" + UUID.randomUUID() + ".png";
        String barcodeUrl = s3Service.upload(barcodeBytes, barcodeKey, "image/png");

        ProductLabel label = new ProductLabel();
        label.setProductCode(request.getProductCode());
        label.setProductName(request.getProductName());
        label.setBarcodeValue(request.getBarcodeValue());
        label.setBarcodeType(request.getBarcodeType());
        label.setLabelData(request.getLabelData());
        label.setBarcodeImageUrl(barcodeUrl);

        if (request.getTemplateId() != null) {
            label.setTemplate(new LabelTemplate(request.getTemplateId()));
        }

        return labelRepository.save(label);
    }

    @Async
    public void generateBulkPdf(Long printJobId) {
        PrintJob job = printJobRepository.findById(printJobId)
                .orElseThrow(() -> new ResourceNotFoundException("Print job not found"));

        job.setStatus("PROCESSING");
        printJobRepository.save(job);

        try {
            List<Long> labelIds = (List<Long>) job.getLabelIds();
            List<ProductLabel> labels = labelRepository.findAllById(labelIds);
            LabelTemplate template = job.getTemplate();

            // Create PDF with labels
            PDDocument document = new PDDocument();
            PDRectangle pageSize = new PDRectangle(
                    template.getWidthMm() * 2.835f, // mm to points
                    template.getHeightMm() * 2.835f);

            int labelsPerPage = calculateLabelsPerPage(template);
            int currentIndex = 0;

            while (currentIndex < labels.size()) {
                PDPage page = new PDPage(new PDRectangle(PDRectangle.A4));
                document.addPage(page);

                try (PDPageContentStream cs = new PDPageContentStream(document, page)) {
                    int labelsOnThisPage = Math.min(labelsPerPage, labels.size() - currentIndex);
                    for (int i = 0; i < labelsOnThisPage; i++) {
                        ProductLabel label = labels.get(currentIndex + i);
                        renderLabelOnPage(document, cs, label, template, i, labelsPerPage);
                    }
                }
                currentIndex += labelsPerPage;
            }

            ByteArrayOutputStream baos = new ByteArrayOutputStream();
            document.save(baos);
            document.close();

            String pdfKey = "labels/print_" + printJobId + ".pdf";
            String pdfUrl = s3Service.upload(baos.toByteArray(), pdfKey, "application/pdf");

            job.setPdfUrl(pdfUrl);
            job.setStatus("COMPLETED");
            job.setCompletedAt(LocalDateTime.now());

        } catch (Exception e) {
            log.error("Bulk PDF generation failed for job {}: {}", printJobId, e.getMessage());
            job.setStatus("FAILED");
        }

        printJobRepository.save(job);
    }

    public ScanResult scanBarcode(String barcodeValue, String scannerLocation, String scannedBy) {
        // Log the scan
        BarcodeScanLog scanLog = new BarcodeScanLog();
        scanLog.setBarcodeValue(barcodeValue);
        scanLog.setScannedBy(scannedBy);
        scanLog.setScanLocation(scannerLocation);

        // Lookup product
        Optional<ProductLabel> label = labelRepository.findByBarcodeValue(barcodeValue);
        Map<String, Object> result = new HashMap<>();

        if (label.isPresent()) {
            result.put("found", true);
            result.put("productCode", label.get().getProductCode());
            result.put("productName", label.get().getProductName());
            result.put("labelData", label.get().getLabelData());
        } else {
            result.put("found", false);
            result.put("message", "Product not found for barcode: " + barcodeValue);
        }

        scanLog.setScanResult(result);
        scanLogRepository.save(scanLog);

        return new ScanResult(label.isPresent(), result);
    }
}
```

### Controller
```java
// BarcodeController.java
@RestController
@RequestMapping("/api/barcodes")
@RequiredArgsConstructor
public class BarcodeController {

    private final BarcodeService barcodeService;

    @GetMapping("/generate")
    public ResponseEntity<byte[]> generateBarcode(
            @RequestParam String value,
            @RequestParam(defaultValue = "CODE_128") String type,
            @RequestParam(defaultValue = "400") int width,
            @RequestParam(defaultValue = "150") int height) throws Exception {
        byte[] barcode = barcodeService.generateBarcode(value, type, width, height);
        return ResponseEntity.ok()
                .contentType(MediaType.IMAGE_PNG)
                .body(barcode);
    }

    @PostMapping("/labels")
    public ResponseEntity<ProductLabelDTO> createLabel(
            @RequestBody @Valid CreateProductLabelRequest request) throws Exception {
        return ResponseEntity.status(HttpStatus.CREATED)
                .body(ProductLabelDTO.from(barcodeService.createProductLabel(request)));
    }

    @PostMapping("/print-jobs")
    public ResponseEntity<PrintJobDTO> createPrintJob(
            @RequestBody @Valid CreatePrintJobRequest request, Authentication auth) {
        PrintJob job = barcodeService.createPrintJob(request, getCurrentUserId(auth));
        barcodeService.generateBulkPdf(job.getId());
        return ResponseEntity.status(HttpStatus.ACCEPTED)
                .body(PrintJobDTO.from(job));
    }

    @GetMapping("/print-jobs/{id}/download")
    public ResponseEntity<Void> downloadPdf(@PathVariable Long id) {
        String pdfUrl = barcodeService.getPdfDownloadUrl(id);
        return ResponseEntity.status(HttpStatus.FOUND)
                .location(URI.create(pdfUrl))
                .build();
    }

    @PostMapping("/scan")
    public ResponseEntity<ScanResultDTO> scanBarcode(@RequestBody @Valid ScanRequest request) {
        return ResponseEntity.ok(ScanResultDTO.from(
                barcodeService.scanBarcode(
                        request.getBarcodeValue(),
                        request.getScanLocation(),
                        request.getScannedBy())));
    }

    @GetMapping("/scan/{value}/lookup")
    public ResponseEntity<ScanResultDTO> lookupBarcode(@PathVariable String value) {
        return ResponseEntity.ok(ScanResultDTO.from(
                barcodeService.scanBarcode(value, "API", "system")));
    }
}
```

### Docker Compose (Part 110)
```yaml
# docker-compose.yml (Part 110)
version: '3.8'
services:
  app:
    build: .
    ports:
      - "8080:8080"
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/logisticsdb
      SPRING_REDIS_HOST: redis
      AWS_S3_BUCKET: logistics-bucket
      AWS_REGION: ap-southeast-1
      GOOGLE_MAPS_API_KEY: ${GOOGLE_MAPS_API_KEY}
    depends_on:
      - postgres
      - redis

  postgres:
    image: postgres:15
    environment:
      POSTGRES_DB: logisticsdb
      POSTGRES_USER: logistics
      POSTGRES_PASSWORD: logisticspass
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

  localstack:
    image: localstack/localstack
    ports:
      - "4566:4566"
    environment:
      SERVICES: s3
      DEFAULT_REGION: ap-southeast-1
    volumes:
      - localstack_data:/var/lib/localstack

volumes:
  postgres_data:
  redis_data:
  localstack_data:
```

---

## สรุป Part 110

| โปรเจค | เทคโนโลยีหลัก | ความซับซ้อน |
|--------|--------------|------------|
| 46. Logistics Tracking | Tracking Events, POD, WebSocket | สูง |
| 47. Fleet Management | GPS Tracking, Depreciation | สูง |
| 48. Asset Management | QR Code, Depreciation, Disposal | สูง |
| 49. QR Code Generator | ZXing, Logo Overlay, Bulk | กลาง |
| 50. Barcode & Label | PDFBox, Barcode Types, Scan API | กลาง |

---

## สรุปภาพรวม Parts 106-110

| Part | หัวข้อ | โปรเจค | เทคโนโลยีหลัก |
|------|--------|--------|--------------|
| 106 | Real-time & Communication | 26-30 | WebSocket, STOMP, MQTT, SSE |
| 107 | Developer Tools | 31-35 | Redis, OAuth2, ZXing, GeoIP |
| 108 | Content & Document | 36-40 | PDFBox, POI, Elasticsearch, S3 |
| 109 | Data & Aggregation | 41-45 | RSS, CoinGecko, Feed Algorithm |
| 110 | Logistics & Tracking | 46-50 | GPS, Barcode, QR Code, Fleet |

---

## Navigation

- [← Part 109: Data & Aggregation](part-109-data-aggregation.md)
- [Part 111: Advanced Patterns →](part-111-advanced-patterns.md)
- [กลับหน้าหลัก](README.md)

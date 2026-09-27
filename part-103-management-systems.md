# Part 103: โปรเจค 11-15 — Operations & Management

> **ระดับ:** ระดับโลก | **เวลาเรียนรู้:** 10-15 ชั่วโมง | **โปรเจค:** 5 โปรเจคสมบูรณ์

ในส่วนนี้เราจะสร้างระบบจัดการด้านการดำเนินงาน ครอบคลุมระบบสต็อก โรงเรียน อีเวนต์ ท่องเที่ยว และฟิตเนส

---

## โปรเจคที่ 11: Inventory Management System

### ภาพรวมระบบ

ระบบจัดการคลังสินค้าที่รองรับสินค้า คลังสินค้า การเคลื่อนไหวสต็อก (รับเข้า/จ่ายออก/โอนย้าย) การแจ้งเตือนสต็อกต่ำ ใบสั่งซื้อ และการประเมินมูลค่าสต็อกแบบ FIFO

### Entity Classes

```java
// Warehouse.java
@Entity
@Table(name = "warehouses")
@Data
@NoArgsConstructor
public class Warehouse {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, unique = true)
    private String code;

    private String name;
    private String address;
    private String managerName;
    private String phone;
    private Boolean active = true;
}

// InventoryProduct.java
@Entity
@Table(name = "inventory_products")
@Data
@NoArgsConstructor
public class InventoryProduct {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(unique = true, nullable = false)
    private String sku;

    private String name;
    private String description;
    private String category;
    private String unit; // PIECE, KG, LITER, BOX, etc.
    private String barcode;

    @Column(precision = 10, scale = 2)
    private BigDecimal costPrice;

    @Column(precision = 10, scale = 2)
    private BigDecimal sellingPrice;

    private Integer reorderPoint; // Alert threshold
    private Integer reorderQuantity; // Suggested reorder qty
    private Boolean trackInventory = true;

    @CreatedDate
    private LocalDateTime createdAt;
}

// StockLevel.java (per warehouse per product)
@Entity
@Table(name = "stock_levels",
       uniqueConstraints = @UniqueConstraint(columnNames = {"product_id", "warehouse_id"}))
@Data
@NoArgsConstructor
public class StockLevel {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "product_id")
    private InventoryProduct product;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "warehouse_id")
    private Warehouse warehouse;

    @Column(nullable = false)
    private Integer quantity = 0;

    @Column(nullable = false)
    private Integer reservedQuantity = 0;

    @Version
    private Long version;

    public Integer getAvailableQuantity() {
        return quantity - reservedQuantity;
    }
}

// StockMovement.java
@Entity
@Table(name = "stock_movements")
@Data
@NoArgsConstructor
public class StockMovement {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "product_id")
    private InventoryProduct product;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "from_warehouse_id")
    private Warehouse fromWarehouse;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "to_warehouse_id")
    private Warehouse toWarehouse;

    @Enumerated(EnumType.STRING)
    private MovementType type; // IN, OUT, TRANSFER, ADJUSTMENT

    private Integer quantity;

    @Column(precision = 10, scale = 2)
    private BigDecimal unitCost;

    @Column(precision = 12, scale = 2)
    private BigDecimal totalCost;

    private String reference; // PO number, order number, etc.
    private String reason;
    private String performedBy;

    @CreatedDate
    private LocalDateTime createdAt;
}

// PurchaseOrder.java
@Entity
@Table(name = "purchase_orders")
@Data
@NoArgsConstructor
public class PurchaseOrder {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(unique = true, nullable = false)
    private String poNumber;

    private String supplierName;
    private String supplierEmail;

    @Enumerated(EnumType.STRING)
    private POStatus status = POStatus.DRAFT;

    @OneToMany(mappedBy = "purchaseOrder", cascade = CascadeType.ALL)
    private List<POLineItem> lineItems = new ArrayList<>();

    @Column(precision = 12, scale = 2)
    private BigDecimal totalAmount;

    private LocalDate expectedDeliveryDate;
    private LocalDate receivedDate;

    private String notes;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "warehouse_id")
    private Warehouse warehouse;

    @CreatedDate
    private LocalDateTime createdAt;
}

// POLineItem.java
@Entity
@Table(name = "po_line_items")
@Data
@NoArgsConstructor
public class POLineItem {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne
    @JoinColumn(name = "purchase_order_id")
    private PurchaseOrder purchaseOrder;

    @ManyToOne
    @JoinColumn(name = "product_id")
    private InventoryProduct product;

    private Integer orderedQuantity;
    private Integer receivedQuantity = 0;

    @Column(precision = 10, scale = 2)
    private BigDecimal unitCost;

    @Column(precision = 12, scale = 2)
    private BigDecimal totalCost;
}

public enum MovementType { IN, OUT, TRANSFER, ADJUSTMENT }
public enum POStatus { DRAFT, SENT, PARTIAL_RECEIVED, FULLY_RECEIVED, CANCELLED }
```

### Service Layer

```java
// InventoryService.java
@Service
@Transactional
@RequiredArgsConstructor
@Slf4j
public class InventoryService {

    private final InventoryProductRepository productRepository;
    private final StockLevelRepository stockLevelRepository;
    private final StockMovementRepository movementRepository;
    private final PurchaseOrderRepository purchaseOrderRepository;

    public StockMovementDTO receiveStock(Long productId, Long warehouseId,
                                         Integer quantity, BigDecimal unitCost, String poNumber) {
        InventoryProduct product = productRepository.findById(productId)
            .orElseThrow(() -> new ResourceNotFoundException("Product not found"));

        StockLevel stockLevel = stockLevelRepository.findByProductIdAndWarehouseId(productId, warehouseId)
            .orElseGet(() -> createStockLevel(productId, warehouseId));

        stockLevel.setQuantity(stockLevel.getQuantity() + quantity);
        stockLevelRepository.save(stockLevel);

        StockMovement movement = new StockMovement();
        movement.setProduct(product);
        movement.setToWarehouse(stockLevel.getWarehouse());
        movement.setType(MovementType.IN);
        movement.setQuantity(quantity);
        movement.setUnitCost(unitCost);
        movement.setTotalCost(unitCost != null ? unitCost.multiply(BigDecimal.valueOf(quantity)) : null);
        movement.setReference(poNumber);

        return toMovementDTO(movementRepository.save(movement));
    }

    public StockMovementDTO issueStock(Long productId, Long warehouseId,
                                        Integer quantity, String reason, String reference) {
        StockLevel stockLevel = stockLevelRepository.findByProductIdAndWarehouseIdWithLock(productId, warehouseId)
            .orElseThrow(() -> new ResourceNotFoundException("Stock not found"));

        if (stockLevel.getAvailableQuantity() < quantity) {
            throw new InsufficientStockException("Insufficient stock. Available: " + stockLevel.getAvailableQuantity());
        }

        stockLevel.setQuantity(stockLevel.getQuantity() - quantity);
        stockLevelRepository.save(stockLevel);

        StockMovement movement = new StockMovement();
        movement.setProduct(stockLevel.getProduct());
        movement.setFromWarehouse(stockLevel.getWarehouse());
        movement.setType(MovementType.OUT);
        movement.setQuantity(quantity);
        movement.setReason(reason);
        movement.setReference(reference);

        // Calculate FIFO cost
        BigDecimal fifoCost = calculateFIFOCost(productId, warehouseId, quantity);
        movement.setTotalCost(fifoCost);

        checkAndAlertLowStock(stockLevel);

        return toMovementDTO(movementRepository.save(movement));
    }

    public StockMovementDTO transferStock(Long productId, Long fromWarehouseId,
                                           Long toWarehouseId, Integer quantity) {
        StockLevel fromStock = stockLevelRepository.findByProductIdAndWarehouseIdWithLock(productId, fromWarehouseId)
            .orElseThrow(() -> new ResourceNotFoundException("Source stock not found"));

        if (fromStock.getAvailableQuantity() < quantity) {
            throw new InsufficientStockException("Insufficient stock for transfer");
        }

        StockLevel toStock = stockLevelRepository.findByProductIdAndWarehouseId(productId, toWarehouseId)
            .orElseGet(() -> createStockLevel(productId, toWarehouseId));

        fromStock.setQuantity(fromStock.getQuantity() - quantity);
        toStock.setQuantity(toStock.getQuantity() + quantity);

        stockLevelRepository.save(fromStock);
        stockLevelRepository.save(toStock);

        StockMovement movement = new StockMovement();
        movement.setProduct(fromStock.getProduct());
        movement.setFromWarehouse(fromStock.getWarehouse());
        movement.setToWarehouse(toStock.getWarehouse());
        movement.setType(MovementType.TRANSFER);
        movement.setQuantity(quantity);

        return toMovementDTO(movementRepository.save(movement));
    }

    public List<LowStockAlert> getLowStockAlerts() {
        return stockLevelRepository.findLowStockItems().stream()
            .map(sl -> LowStockAlert.builder()
                .productId(sl.getProduct().getId())
                .productName(sl.getProduct().getName())
                .sku(sl.getProduct().getSku())
                .warehouseName(sl.getWarehouse().getName())
                .currentStock(sl.getQuantity())
                .reorderPoint(sl.getProduct().getReorderPoint())
                .suggestedReorderQty(sl.getProduct().getReorderQuantity())
                .build())
            .collect(Collectors.toList());
    }

    public StockValuationReport getStockValuation(Long warehouseId) {
        List<StockLevel> stockLevels = warehouseId != null ?
            stockLevelRepository.findByWarehouseId(warehouseId) :
            stockLevelRepository.findAll();

        BigDecimal totalValue = BigDecimal.ZERO;
        List<StockValuationItem> items = new ArrayList<>();

        for (StockLevel sl : stockLevels) {
            BigDecimal avgCost = calculateAverageCost(sl.getProduct().getId(), sl.getWarehouse().getId());
            BigDecimal value = avgCost.multiply(BigDecimal.valueOf(sl.getQuantity()));
            totalValue = totalValue.add(value);

            items.add(StockValuationItem.builder()
                .sku(sl.getProduct().getSku())
                .productName(sl.getProduct().getName())
                .quantity(sl.getQuantity())
                .avgCost(avgCost)
                .totalValue(value)
                .build());
        }

        return StockValuationReport.builder()
            .totalValue(totalValue)
            .items(items)
            .generatedAt(LocalDateTime.now())
            .build();
    }

    private BigDecimal calculateFIFOCost(Long productId, Long warehouseId, int quantity) {
        List<StockMovement> inMovements = movementRepository
            .findFIFOMovements(productId, warehouseId, MovementType.IN);

        BigDecimal totalCost = BigDecimal.ZERO;
        int remaining = quantity;

        for (StockMovement mov : inMovements) {
            if (remaining <= 0) break;
            int used = Math.min(remaining, mov.getQuantity());
            if (mov.getUnitCost() != null) {
                totalCost = totalCost.add(mov.getUnitCost().multiply(BigDecimal.valueOf(used)));
            }
            remaining -= used;
        }

        return totalCost;
    }

    private BigDecimal calculateAverageCost(Long productId, Long warehouseId) {
        BigDecimal avgCost = movementRepository.calculateAverageCost(productId, warehouseId);
        return avgCost != null ? avgCost : BigDecimal.ZERO;
    }

    private void checkAndAlertLowStock(StockLevel stockLevel) {
        InventoryProduct product = stockLevel.getProduct();
        if (product.getReorderPoint() != null && stockLevel.getQuantity() <= product.getReorderPoint()) {
            log.warn("LOW STOCK ALERT: Product {} (SKU: {}) at warehouse {} - Current: {}, Reorder Point: {}",
                product.getName(), product.getSku(), stockLevel.getWarehouse().getName(),
                stockLevel.getQuantity(), product.getReorderPoint());
        }
    }

    private StockLevel createStockLevel(Long productId, Long warehouseId) {
        StockLevel sl = new StockLevel();
        sl.setProduct(productRepository.findById(productId).orElseThrow());
        sl.setWarehouse(new Warehouse()); // simplified
        sl.setQuantity(0);
        return sl;
    }

    private StockMovementDTO toMovementDTO(StockMovement m) {
        return StockMovementDTO.builder()
            .id(m.getId())
            .productName(m.getProduct().getName())
            .type(m.getType())
            .quantity(m.getQuantity())
            .reference(m.getReference())
            .createdAt(m.getCreatedAt())
            .build();
    }
}
```

### REST Controller

```java
// InventoryController.java
@RestController
@RequestMapping("/api/v1/inventory")
@RequiredArgsConstructor
public class InventoryController {

    private final InventoryService inventoryService;

    @PostMapping("/stock/receive")
    public ResponseEntity<StockMovementDTO> receiveStock(@Valid @RequestBody ReceiveStockRequest request) {
        return ResponseEntity.ok(inventoryService.receiveStock(
            request.getProductId(), request.getWarehouseId(),
            request.getQuantity(), request.getUnitCost(), request.getPoNumber()));
    }

    @PostMapping("/stock/issue")
    public ResponseEntity<StockMovementDTO> issueStock(@Valid @RequestBody IssueStockRequest request) {
        return ResponseEntity.ok(inventoryService.issueStock(
            request.getProductId(), request.getWarehouseId(),
            request.getQuantity(), request.getReason(), request.getReference()));
    }

    @PostMapping("/stock/transfer")
    public ResponseEntity<StockMovementDTO> transferStock(@Valid @RequestBody TransferStockRequest request) {
        return ResponseEntity.ok(inventoryService.transferStock(
            request.getProductId(), request.getFromWarehouseId(),
            request.getToWarehouseId(), request.getQuantity()));
    }

    @GetMapping("/stock/levels")
    public ResponseEntity<List<StockLevelDTO>> getStockLevels(
            @RequestParam(required = false) Long warehouseId,
            @RequestParam(required = false) String sku) {
        return ResponseEntity.ok(inventoryService.getStockLevels(warehouseId, sku));
    }

    @GetMapping("/alerts/low-stock")
    public ResponseEntity<List<LowStockAlert>> getLowStockAlerts() {
        return ResponseEntity.ok(inventoryService.getLowStockAlerts());
    }

    @GetMapping("/reports/valuation")
    public ResponseEntity<StockValuationReport> getStockValuation(
            @RequestParam(required = false) Long warehouseId) {
        return ResponseEntity.ok(inventoryService.getStockValuation(warehouseId));
    }

    @GetMapping("/movements")
    public ResponseEntity<Page<StockMovementDTO>> getMovements(
            @RequestParam(required = false) Long productId,
            @RequestParam(required = false) MovementType type,
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size) {
        return ResponseEntity.ok(inventoryService.getMovements(productId, type, PageRequest.of(page, size)));
    }
}
```

---

## โปรเจคที่ 12: School Management System

### ภาพรวมระบบ

ระบบจัดการโรงเรียนที่รองรับข้อมูลนักเรียน ครู ชั้นเรียน วิชา การลงทะเบียน คะแนน การเข้าเรียน รายงานผลการเรียน และระบบแจ้งเตือนผู้ปกครอง

### Entity Classes

```java
// Student.java
@Entity
@Table(name = "students")
@Data
@NoArgsConstructor
public class Student {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(unique = true)
    private String studentId;

    private String firstName;
    private String lastName;
    private LocalDate dateOfBirth;
    private String gender;
    private String email;
    private String phone;
    private String address;

    @Enumerated(EnumType.STRING)
    private StudentStatus status = StudentStatus.ACTIVE;

    // Parent/Guardian
    private String parentName;
    private String parentPhone;
    private String parentEmail;

    @OneToMany(mappedBy = "student")
    private List<Enrollment> enrollments = new ArrayList<>();

    @CreatedDate
    private LocalDateTime createdAt;
}

// Teacher.java
@Entity
@Table(name = "teachers")
@Data
@NoArgsConstructor
public class Teacher {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(unique = true)
    private String teacherId;

    private String firstName;
    private String lastName;
    private String email;
    private String phone;
    private String specialization;
    private String qualification;

    @Column(precision = 12, scale = 2)
    private BigDecimal salary;

    private LocalDate hireDate;
    private Boolean active = true;
}

// SchoolClass.java
@Entity
@Table(name = "school_classes")
@Data
@NoArgsConstructor
public class SchoolClass {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name; // e.g., "6A", "Grade 10"
    private String gradeLevel;
    private Integer academicYear;
    private Integer maxStudents;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "homeroom_teacher_id")
    private Teacher homeroomTeacher;

    @OneToMany(mappedBy = "schoolClass")
    private List<Enrollment> enrollments = new ArrayList<>();
}

// Subject.java
@Entity
@Table(name = "subjects")
@Data
@NoArgsConstructor
public class Subject {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(unique = true)
    private String subjectCode;

    private String name;
    private String description;
    private Integer creditHours;
    private String gradeLevel;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "teacher_id")
    private Teacher teacher;
}

// Enrollment.java
@Entity
@Table(name = "enrollments",
       uniqueConstraints = @UniqueConstraint(columnNames = {"student_id", "school_class_id", "academic_year"}))
@Data
@NoArgsConstructor
public class Enrollment {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "student_id")
    private Student student;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "school_class_id")
    private SchoolClass schoolClass;

    private Integer academicYear;

    @Enumerated(EnumType.STRING)
    private EnrollmentStatus status = EnrollmentStatus.ACTIVE;

    @OneToMany(mappedBy = "enrollment")
    private List<Grade> grades = new ArrayList<>();

    @CreatedDate
    private LocalDateTime enrolledAt;
}

// Grade.java
@Entity
@Table(name = "grades")
@Data
@NoArgsConstructor
public class Grade {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "enrollment_id")
    private Enrollment enrollment;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "subject_id")
    private Subject subject;

    @Column(precision = 5, scale = 2)
    private BigDecimal midtermScore;

    @Column(precision = 5, scale = 2)
    private BigDecimal finalScore;

    @Column(precision = 5, scale = 2)
    private BigDecimal assignmentScore;

    @Column(precision = 5, scale = 2)
    private BigDecimal totalScore;

    private String letterGrade; // A, B+, B, C+, C, D, F

    private Integer semester;
    private Integer academicYear;

    @CreatedDate
    private LocalDateTime createdAt;
}

// Attendance (for school)
@Entity
@Table(name = "school_attendance")
@Data
@NoArgsConstructor
public class SchoolAttendance {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne
    @JoinColumn(name = "student_id")
    private Student student;

    @ManyToOne
    @JoinColumn(name = "subject_id")
    private Subject subject;

    private LocalDate attendanceDate;

    @Enumerated(EnumType.STRING)
    private AttendanceType type = AttendanceType.PRESENT;

    private String notes;
}

public enum StudentStatus { ACTIVE, GRADUATED, TRANSFERRED, SUSPENDED, DROPPED }
public enum EnrollmentStatus { ACTIVE, COMPLETED, WITHDRAWN }
public enum AttendanceType { PRESENT, ABSENT, LATE, EXCUSED }
```

### Service Layer

```java
// SchoolService.java
@Service
@Transactional
@RequiredArgsConstructor
public class SchoolService {

    private final StudentRepository studentRepository;
    private final TeacherRepository teacherRepository;
    private final EnrollmentRepository enrollmentRepository;
    private final GradeRepository gradeRepository;
    private final SchoolAttendanceRepository attendanceRepository;
    private final EmailService emailService;

    public EnrollmentDTO enrollStudent(Long studentId, Long classId, int academicYear) {
        Student student = studentRepository.findById(studentId)
            .orElseThrow(() -> new ResourceNotFoundException("Student not found"));

        SchoolClass schoolClass = schoolClassRepository.findById(classId)
            .orElseThrow(() -> new ResourceNotFoundException("Class not found"));

        // Check capacity
        long enrolled = enrollmentRepository.countBySchoolClassIdAndStatus(classId, EnrollmentStatus.ACTIVE);
        if (schoolClass.getMaxStudents() != null && enrolled >= schoolClass.getMaxStudents()) {
            throw new ConflictException("Class is at maximum capacity");
        }

        Enrollment enrollment = new Enrollment();
        enrollment.setStudent(student);
        enrollment.setSchoolClass(schoolClass);
        enrollment.setAcademicYear(academicYear);

        return toEnrollmentDTO(enrollmentRepository.save(enrollment));
    }

    public GradeDTO recordGrade(RecordGradeRequest request) {
        Enrollment enrollment = enrollmentRepository.findById(request.getEnrollmentId())
            .orElseThrow(() -> new ResourceNotFoundException("Enrollment not found"));

        Subject subject = subjectRepository.findById(request.getSubjectId())
            .orElseThrow(() -> new ResourceNotFoundException("Subject not found"));

        Grade grade = gradeRepository.findByEnrollmentIdAndSubjectIdAndSemesterAndAcademicYear(
            request.getEnrollmentId(), request.getSubjectId(),
            request.getSemester(), request.getAcademicYear())
            .orElse(new Grade());

        grade.setEnrollment(enrollment);
        grade.setSubject(subject);
        grade.setSemester(request.getSemester());
        grade.setAcademicYear(request.getAcademicYear());
        grade.setMidtermScore(request.getMidtermScore());
        grade.setFinalScore(request.getFinalScore());
        grade.setAssignmentScore(request.getAssignmentScore());

        // Calculate total and letter grade
        BigDecimal total = calculateTotalScore(grade);
        grade.setTotalScore(total);
        grade.setLetterGrade(calculateLetterGrade(total));

        Grade saved = gradeRepository.save(grade);

        // Notify parent if failing
        if (grade.getLetterGrade().equals("F")) {
            Student student = enrollment.getStudent();
            emailService.sendFailingGradeAlert(student.getParentEmail(), student, subject, grade);
        }

        return toGradeDTO(saved);
    }

    public ReportCard generateReportCard(Long studentId, int semester, int academicYear) {
        Student student = studentRepository.findById(studentId)
            .orElseThrow(() -> new ResourceNotFoundException("Student not found"));

        List<Grade> grades = gradeRepository.findByStudentIdAndSemesterAndYear(studentId, semester, academicYear);

        BigDecimal gpa = calculateGPA(grades);

        return ReportCard.builder()
            .studentId(student.getStudentId())
            .studentName(student.getFirstName() + " " + student.getLastName())
            .semester(semester)
            .academicYear(academicYear)
            .gpa(gpa)
            .grades(grades.stream().map(this::toGradeDTO).collect(Collectors.toList()))
            .attendance(getAttendanceSummary(studentId, semester, academicYear))
            .build();
    }

    public AttendanceSummary getAttendanceSummary(Long studentId, int semester, int academicYear) {
        List<SchoolAttendance> records = attendanceRepository
            .findByStudentIdAndSemesterYear(studentId, academicYear);

        long present = records.stream().filter(a -> a.getType() == AttendanceType.PRESENT).count();
        long absent = records.stream().filter(a -> a.getType() == AttendanceType.ABSENT).count();
        long late = records.stream().filter(a -> a.getType() == AttendanceType.LATE).count();

        double attendanceRate = records.isEmpty() ? 0 :
            ((double)(present + late) / records.size()) * 100;

        return AttendanceSummary.builder()
            .totalDays(records.size())
            .presentDays((int) present)
            .absentDays((int) absent)
            .lateDays((int) late)
            .attendanceRate(Math.round(attendanceRate * 100.0) / 100.0)
            .build();
    }

    private BigDecimal calculateTotalScore(Grade grade) {
        BigDecimal midterm = grade.getMidtermScore() != null ? grade.getMidtermScore() : BigDecimal.ZERO;
        BigDecimal finalScore = grade.getFinalScore() != null ? grade.getFinalScore() : BigDecimal.ZERO;
        BigDecimal assignment = grade.getAssignmentScore() != null ? grade.getAssignmentScore() : BigDecimal.ZERO;

        // Midterm 30%, Final 50%, Assignment 20%
        return midterm.multiply(new BigDecimal("0.30"))
            .add(finalScore.multiply(new BigDecimal("0.50")))
            .add(assignment.multiply(new BigDecimal("0.20")));
    }

    private String calculateLetterGrade(BigDecimal score) {
        double s = score.doubleValue();
        if (s >= 80) return "A";
        if (s >= 75) return "B+";
        if (s >= 70) return "B";
        if (s >= 65) return "C+";
        if (s >= 60) return "C";
        if (s >= 55) return "D+";
        if (s >= 50) return "D";
        return "F";
    }

    private BigDecimal calculateGPA(List<Grade> grades) {
        if (grades.isEmpty()) return BigDecimal.ZERO;

        double total = grades.stream()
            .mapToDouble(g -> letterGradeToPoints(g.getLetterGrade()))
            .average().orElse(0.0);

        return BigDecimal.valueOf(total).setScale(2, RoundingMode.HALF_UP);
    }

    private double letterGradeToPoints(String grade) {
        return switch (grade) {
            case "A" -> 4.0;
            case "B+" -> 3.5;
            case "B" -> 3.0;
            case "C+" -> 2.5;
            case "C" -> 2.0;
            case "D+" -> 1.5;
            case "D" -> 1.0;
            default -> 0.0;
        };
    }

    private EnrollmentDTO toEnrollmentDTO(Enrollment e) {
        return EnrollmentDTO.builder()
            .id(e.getId())
            .studentName(e.getStudent().getFirstName() + " " + e.getStudent().getLastName())
            .className(e.getSchoolClass().getName())
            .academicYear(e.getAcademicYear())
            .status(e.getStatus())
            .build();
    }

    private GradeDTO toGradeDTO(Grade g) {
        return GradeDTO.builder()
            .subjectName(g.getSubject().getName())
            .midtermScore(g.getMidtermScore())
            .finalScore(g.getFinalScore())
            .totalScore(g.getTotalScore())
            .letterGrade(g.getLetterGrade())
            .build();
    }
}
```

---

## โปรเจคที่ 13: Event Management Platform

### ภาพรวมระบบ

แพลตฟอร์มจัดการอีเวนต์ที่รองรับการสร้างอีเวนต์ ตั๋วหลายประเภท การลงทะเบียน การ Check-in ด้วย QR Code การจัดการวิทยากร ตารางกิจกรรม และการจำกัดจำนวนที่นั่ง

### Entity Classes

```java
// Event.java
@Entity
@Table(name = "events")
@Data
@NoArgsConstructor
public class Event {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String title;

    @Column(columnDefinition = "TEXT")
    private String description;

    private String venue;
    private String address;
    private String city;

    private LocalDateTime startDateTime;
    private LocalDateTime endDateTime;

    @Enumerated(EnumType.STRING)
    private EventStatus status = EventStatus.DRAFT;

    private String bannerImageUrl;

    @Enumerated(EnumType.STRING)
    private EventType type;

    private String organizer;
    private String organizerEmail;

    @OneToMany(mappedBy = "event", cascade = CascadeType.ALL)
    private List<TicketType> ticketTypes = new ArrayList<>();

    @OneToMany(mappedBy = "event", cascade = CascadeType.ALL)
    private List<Speaker> speakers = new ArrayList<>();

    @OneToMany(mappedBy = "event", cascade = CascadeType.ALL)
    private List<AgendaItem> agenda = new ArrayList<>();

    @CreatedDate
    private LocalDateTime createdAt;
}

// TicketType.java
@Entity
@Table(name = "ticket_types")
@Data
@NoArgsConstructor
public class TicketType {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "event_id")
    private Event event;

    private String name; // GENERAL, VIP, STUDENT, EARLY_BIRD
    private String description;

    @Column(precision = 10, scale = 2)
    private BigDecimal price;

    private Integer totalQuantity;
    private Integer soldQuantity = 0;

    private LocalDateTime saleStartDate;
    private LocalDateTime saleEndDate;

    public Integer getAvailableQuantity() {
        return totalQuantity - soldQuantity;
    }
}

// Registration.java
@Entity
@Table(name = "registrations")
@Data
@NoArgsConstructor
public class Registration {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(unique = true)
    private String registrationCode;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "event_id")
    private Event event;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "ticket_type_id")
    private TicketType ticketType;

    private String attendeeName;
    private String attendeeEmail;
    private String attendeePhone;

    @Column(precision = 10, scale = 2)
    private BigDecimal amountPaid;

    @Enumerated(EnumType.STRING)
    private RegistrationStatus status = RegistrationStatus.CONFIRMED;

    private String qrCode; // Base64 encoded QR

    private LocalDateTime checkedInAt;
    private String checkedInBy;

    @CreatedDate
    private LocalDateTime registeredAt;
}

// Speaker.java
@Entity
@Table(name = "speakers")
@Data
@NoArgsConstructor
public class Speaker {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "event_id")
    private Event event;

    private String name;
    private String bio;
    private String organization;
    private String jobTitle;
    private String photoUrl;
    private String linkedInUrl;
}

// AgendaItem.java
@Entity
@Table(name = "agenda_items")
@Data
@NoArgsConstructor
public class AgendaItem {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "event_id")
    private Event event;

    private String title;
    private String description;
    private LocalDateTime startTime;
    private LocalDateTime endTime;
    private String location;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "speaker_id")
    private Speaker speaker;

    private Integer displayOrder;
}

public enum EventStatus { DRAFT, PUBLISHED, CANCELLED, COMPLETED }
public enum EventType { CONFERENCE, WORKSHOP, SEMINAR, NETWORKING, CONCERT, SPORTS }
public enum RegistrationStatus { PENDING, CONFIRMED, CANCELLED, ATTENDED, NO_SHOW }
```

### Service Layer

```java
// EventService.java
@Service
@Transactional
@RequiredArgsConstructor
public class EventService {

    private final EventRepository eventRepository;
    private final TicketTypeRepository ticketTypeRepository;
    private final RegistrationRepository registrationRepository;
    private final QRCodeService qrCodeService;
    private final EmailService emailService;

    public RegistrationDTO registerForEvent(RegisterEventRequest request) {
        Event event = eventRepository.findById(request.getEventId())
            .orElseThrow(() -> new ResourceNotFoundException("Event not found"));

        if (event.getStatus() != EventStatus.PUBLISHED) {
            throw new BadRequestException("Event is not open for registration");
        }

        if (event.getStartDateTime().isBefore(LocalDateTime.now())) {
            throw new BadRequestException("Event has already started");
        }

        TicketType ticketType = ticketTypeRepository.findByIdWithLock(request.getTicketTypeId())
            .orElseThrow(() -> new ResourceNotFoundException("Ticket type not found"));

        if (ticketType.getAvailableQuantity() <= 0) {
            throw new ConflictException("No tickets available for this ticket type");
        }

        // Check registration dates
        LocalDateTime now = LocalDateTime.now();
        if (ticketType.getSaleStartDate() != null && now.isBefore(ticketType.getSaleStartDate())) {
            throw new BadRequestException("Ticket sale has not started yet");
        }
        if (ticketType.getSaleEndDate() != null && now.isAfter(ticketType.getSaleEndDate())) {
            throw new BadRequestException("Ticket sale has ended");
        }

        // Check duplicate registration
        if (registrationRepository.existsByEventIdAndAttendeeEmail(event.getId(), request.getAttendeeEmail())) {
            throw new ConflictException("This email is already registered for this event");
        }

        String regCode = generateRegistrationCode();
        String qrCode = qrCodeService.generateQRCode(regCode);

        Registration registration = new Registration();
        registration.setRegistrationCode(regCode);
        registration.setEvent(event);
        registration.setTicketType(ticketType);
        registration.setAttendeeName(request.getAttendeeName());
        registration.setAttendeeEmail(request.getAttendeeEmail());
        registration.setAttendeePhone(request.getAttendeePhone());
        registration.setAmountPaid(ticketType.getPrice());
        registration.setQrCode(qrCode);

        ticketType.setSoldQuantity(ticketType.getSoldQuantity() + 1);
        ticketTypeRepository.save(ticketType);

        Registration saved = registrationRepository.save(registration);

        // Send confirmation email with QR code
        emailService.sendRegistrationConfirmation(saved.getAttendeeEmail(), saved);

        return toRegistrationDTO(saved);
    }

    public CheckInResult checkIn(String registrationCode, String staffId) {
        Registration registration = registrationRepository.findByRegistrationCode(registrationCode)
            .orElseThrow(() -> new ResourceNotFoundException("Registration not found"));

        if (registration.getStatus() == RegistrationStatus.CANCELLED) {
            return new CheckInResult(false, "Registration is cancelled");
        }

        if (registration.getCheckedInAt() != null) {
            return new CheckInResult(false, "Already checked in at " + registration.getCheckedInAt());
        }

        registration.setCheckedInAt(LocalDateTime.now());
        registration.setCheckedInBy(staffId);
        registration.setStatus(RegistrationStatus.ATTENDED);
        registrationRepository.save(registration);

        return new CheckInResult(true, "Check-in successful for " + registration.getAttendeeName());
    }

    public EventStatistics getEventStatistics(Long eventId) {
        Event event = eventRepository.findById(eventId)
            .orElseThrow(() -> new ResourceNotFoundException("Event not found"));

        List<Registration> registrations = registrationRepository.findByEventId(eventId);

        long totalRegistrations = registrations.size();
        long checkedIn = registrations.stream()
            .filter(r -> r.getCheckedInAt() != null).count();
        long cancelled = registrations.stream()
            .filter(r -> r.getStatus() == RegistrationStatus.CANCELLED).count();

        BigDecimal totalRevenue = registrations.stream()
            .filter(r -> r.getStatus() != RegistrationStatus.CANCELLED && r.getAmountPaid() != null)
            .map(Registration::getAmountPaid)
            .reduce(BigDecimal.ZERO, BigDecimal::add);

        Map<String, Long> byTicketType = registrations.stream()
            .collect(Collectors.groupingBy(r -> r.getTicketType().getName(), Collectors.counting()));

        return EventStatistics.builder()
            .eventId(eventId)
            .eventTitle(event.getTitle())
            .totalRegistrations(totalRegistrations)
            .checkedIn(checkedIn)
            .cancelled(cancelled)
            .attendanceRate(totalRegistrations > 0 ? (double) checkedIn / totalRegistrations * 100 : 0)
            .totalRevenue(totalRevenue)
            .byTicketType(byTicketType)
            .build();
    }

    private String generateRegistrationCode() {
        return "EVT-" + UUID.randomUUID().toString().substring(0, 8).toUpperCase();
    }

    private RegistrationDTO toRegistrationDTO(Registration r) {
        return RegistrationDTO.builder()
            .id(r.getId())
            .registrationCode(r.getRegistrationCode())
            .eventTitle(r.getEvent().getTitle())
            .ticketType(r.getTicketType().getName())
            .attendeeName(r.getAttendeeName())
            .attendeeEmail(r.getAttendeeEmail())
            .amountPaid(r.getAmountPaid())
            .status(r.getStatus())
            .qrCode(r.getQrCode())
            .registeredAt(r.getRegisteredAt())
            .build();
    }
}
```

---

## โปรเจคที่ 14: Travel Booking System

### ภาพรวมระบบ

ระบบจองท่องเที่ยวที่รองรับจุดหมายปลายทาง ทัวร์ แพ็คเกจ การจอง ข้อมูลผู้โดยสาร การชำระเงิน การสร้าง Itinerary และนโยบายการยกเลิก

### Entity Classes

```java
// Destination.java
@Entity
@Table(name = "destinations")
@Data
@NoArgsConstructor
public class Destination {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;
    private String country;
    private String city;
    private String description;
    private String imageUrl;
    private Boolean popular = false;
}

// TourPackage.java
@Entity
@Table(name = "tour_packages")
@Data
@NoArgsConstructor
public class TourPackage {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String title;
    private String description;
    private Integer durationDays;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "destination_id")
    private Destination destination;

    @Column(precision = 10, scale = 2)
    private BigDecimal pricePerPerson;

    @Column(precision = 10, scale = 2)
    private BigDecimal childPrice;

    private Integer maxGroupSize;
    private String difficulty; // EASY, MODERATE, HARD
    private String highlights;

    @ElementCollection
    @CollectionTable(name = "package_inclusions")
    private List<String> inclusions = new ArrayList<>();

    @ElementCollection
    @CollectionTable(name = "package_exclusions")
    private List<String> exclusions = new ArrayList<>();

    private Boolean available = true;
}

// TourBooking.java
@Entity
@Table(name = "tour_bookings")
@Data
@NoArgsConstructor
public class TourBooking {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(unique = true)
    private String bookingReference;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "package_id")
    private TourPackage tourPackage;

    private LocalDate departureDate;
    private Integer adultsCount;
    private Integer childrenCount;

    @OneToMany(mappedBy = "booking", cascade = CascadeType.ALL)
    private List<Passenger> passengers = new ArrayList<>();

    @Column(precision = 12, scale = 2)
    private BigDecimal totalAmount;

    @Column(precision = 12, scale = 2)
    private BigDecimal depositAmount;

    @Column(precision = 12, scale = 2)
    private BigDecimal paidAmount = BigDecimal.ZERO;

    @Enumerated(EnumType.STRING)
    private BookingStatus status = BookingStatus.PENDING;

    private String contactName;
    private String contactEmail;
    private String contactPhone;

    private String specialRequests;
    private String cancellationReason;
    private LocalDateTime cancelledAt;

    @Column(precision = 10, scale = 2)
    private BigDecimal cancellationFee;

    @CreatedDate
    private LocalDateTime createdAt;
}

// Passenger.java
@Entity
@Table(name = "passengers")
@Data
@NoArgsConstructor
public class Passenger {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne
    @JoinColumn(name = "booking_id")
    private TourBooking booking;

    private String firstName;
    private String lastName;
    private LocalDate dateOfBirth;
    private String passportNumber;
    private String nationality;
    private Boolean isAdult = true;
}

public enum BookingStatus { PENDING, CONFIRMED, PARTIALLY_PAID, FULLY_PAID, CANCELLED, COMPLETED }
```

### Service Layer

```java
// TravelService.java
@Service
@Transactional
@RequiredArgsConstructor
public class TravelService {

    private final TourPackageRepository packageRepository;
    private final TourBookingRepository bookingRepository;
    private final EmailService emailService;

    public BookingDTO createBooking(CreateBookingRequest request) {
        TourPackage tourPackage = packageRepository.findById(request.getPackageId())
            .orElseThrow(() -> new ResourceNotFoundException("Tour package not found"));

        if (!tourPackage.getAvailable()) {
            throw new BadRequestException("Tour package is not available");
        }

        // Check availability for departure date
        long existingBookings = bookingRepository.countConfirmedBookings(
            request.getPackageId(), request.getDepartureDate());

        if (tourPackage.getMaxGroupSize() != null) {
            int totalPassengers = request.getAdultsCount() + 
                                  (request.getChildrenCount() != null ? request.getChildrenCount() : 0);
            if (existingBookings + totalPassengers > tourPackage.getMaxGroupSize()) {
                throw new ConflictException("Not enough availability for selected date");
            }
        }

        BigDecimal adultTotal = tourPackage.getPricePerPerson()
            .multiply(BigDecimal.valueOf(request.getAdultsCount()));
        BigDecimal childTotal = tourPackage.getChildPrice() != null && request.getChildrenCount() != null ?
            tourPackage.getChildPrice().multiply(BigDecimal.valueOf(request.getChildrenCount())) :
            BigDecimal.ZERO;
        BigDecimal totalAmount = adultTotal.add(childTotal);
        BigDecimal deposit = totalAmount.multiply(new BigDecimal("0.30"));

        TourBooking booking = new TourBooking();
        booking.setBookingReference(generateBookingRef());
        booking.setTourPackage(tourPackage);
        booking.setDepartureDate(request.getDepartureDate());
        booking.setAdultsCount(request.getAdultsCount());
        booking.setChildrenCount(request.getChildrenCount());
        booking.setTotalAmount(totalAmount);
        booking.setDepositAmount(deposit);
        booking.setContactName(request.getContactName());
        booking.setContactEmail(request.getContactEmail());
        booking.setContactPhone(request.getContactPhone());
        booking.setSpecialRequests(request.getSpecialRequests());

        // Add passengers
        for (PassengerRequest pasReq : request.getPassengers()) {
            Passenger passenger = new Passenger();
            passenger.setBooking(booking);
            passenger.setFirstName(pasReq.getFirstName());
            passenger.setLastName(pasReq.getLastName());
            passenger.setDateOfBirth(pasReq.getDateOfBirth());
            passenger.setPassportNumber(pasReq.getPassportNumber());
            passenger.setNationality(pasReq.getNationality());
            passenger.setIsAdult(pasReq.getIsAdult());
            booking.getPassengers().add(passenger);
        }

        TourBooking saved = bookingRepository.save(booking);
        emailService.sendBookingConfirmation(saved.getContactEmail(), saved);

        return toBookingDTO(saved);
    }

    public BookingDTO cancelBooking(String bookingReference, String reason) {
        TourBooking booking = bookingRepository.findByBookingReference(bookingReference)
            .orElseThrow(() -> new ResourceNotFoundException("Booking not found"));

        if (booking.getStatus() == BookingStatus.CANCELLED) {
            throw new BadRequestException("Booking is already cancelled");
        }

        // Calculate cancellation fee
        long daysUntilDeparture = ChronoUnit.DAYS.between(LocalDate.now(), booking.getDepartureDate());
        BigDecimal cancellationFee = calculateCancellationFee(booking.getTotalAmount(), daysUntilDeparture);

        booking.setStatus(BookingStatus.CANCELLED);
        booking.setCancellationReason(reason);
        booking.setCancelledAt(LocalDateTime.now());
        booking.setCancellationFee(cancellationFee);

        emailService.sendCancellationNotice(booking.getContactEmail(), booking, cancellationFee);

        return toBookingDTO(bookingRepository.save(booking));
    }

    public Itinerary generateItinerary(String bookingReference) {
        TourBooking booking = bookingRepository.findByBookingReference(bookingReference)
            .orElseThrow(() -> new ResourceNotFoundException("Booking not found"));

        TourPackage pkg = booking.getTourPackage();
        List<ItineraryDay> days = new ArrayList<>();

        for (int i = 0; i < pkg.getDurationDays(); i++) {
            ItineraryDay day = new ItineraryDay();
            day.setDay(i + 1);
            day.setDate(booking.getDepartureDate().plusDays(i));
            day.setTitle("Day " + (i + 1) + " - " + pkg.getDestination().getCity());
            day.setDescription("Explore " + pkg.getDestination().getName());
            days.add(day);
        }

        return Itinerary.builder()
            .bookingReference(bookingReference)
            .tourName(pkg.getTitle())
            .departureDate(booking.getDepartureDate())
            .returnDate(booking.getDepartureDate().plusDays(pkg.getDurationDays() - 1))
            .contactName(booking.getContactName())
            .days(days)
            .inclusions(pkg.getInclusions())
            .build();
    }

    private BigDecimal calculateCancellationFee(BigDecimal totalAmount, long daysUntilDeparture) {
        if (daysUntilDeparture >= 30) return BigDecimal.ZERO;
        if (daysUntilDeparture >= 15) return totalAmount.multiply(new BigDecimal("0.25"));
        if (daysUntilDeparture >= 7) return totalAmount.multiply(new BigDecimal("0.50"));
        return totalAmount.multiply(new BigDecimal("0.80")); // < 7 days: 80% fee
    }

    private String generateBookingRef() {
        return "TRV-" + LocalDate.now().format(DateTimeFormatter.ofPattern("yyyyMMdd")) +
               "-" + String.format("%04d", (int)(Math.random() * 10000));
    }

    private BookingDTO toBookingDTO(TourBooking b) {
        return BookingDTO.builder()
            .id(b.getId())
            .bookingReference(b.getBookingReference())
            .tourName(b.getTourPackage().getTitle())
            .departureDate(b.getDepartureDate())
            .status(b.getStatus())
            .totalAmount(b.getTotalAmount())
            .depositAmount(b.getDepositAmount())
            .paidAmount(b.getPaidAmount())
            .passengerCount(b.getAdultsCount() + (b.getChildrenCount() != null ? b.getChildrenCount() : 0))
            .build();
    }
}
```

---

## โปรเจคที่ 15: Gym Management System

### ภาพรวมระบบ

ระบบจัดการฟิตเนสที่รองรับสมาชิก แผนสมาชิก ตารางเรียน การจองคลาส การมอบหมาย Trainer การติดตาม Check-in และการแจ้งเตือนชำระเงิน

### Entity Classes

```java
// Member.java
@Entity
@Table(name = "members")
@Data
@NoArgsConstructor
public class Member {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(unique = true)
    private String memberCode;

    private String firstName;
    private String lastName;
    private String email;
    private String phone;
    private LocalDate dateOfBirth;
    private String gender;

    @Enumerated(EnumType.STRING)
    private MemberStatus status = MemberStatus.ACTIVE;

    @OneToMany(mappedBy = "member")
    private List<MembershipSubscription> subscriptions = new ArrayList<>();

    private String emergencyContactName;
    private String emergencyContactPhone;
    private String healthNotes;

    @CreatedDate
    private LocalDateTime joinedAt;
}

// MembershipPlan.java
@Entity
@Table(name = "membership_plans")
@Data
@NoArgsConstructor
public class MembershipPlan {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;
    private String description;
    private Integer durationDays;

    @Column(precision = 10, scale = 2)
    private BigDecimal price;

    private Integer classesAllowed; // null = unlimited
    private Boolean allowGuestPass;
    private Integer guestPassCount;
    private Boolean includesTrainer;

    @ElementCollection
    @CollectionTable(name = "plan_features")
    private List<String> features = new ArrayList<>();

    private Boolean active = true;
}

// MembershipSubscription.java
@Entity
@Table(name = "membership_subscriptions")
@Data
@NoArgsConstructor
public class MembershipSubscription {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne
    @JoinColumn(name = "member_id")
    private Member member;

    @ManyToOne
    @JoinColumn(name = "plan_id")
    private MembershipPlan plan;

    private LocalDate startDate;
    private LocalDate endDate;

    @Enumerated(EnumType.STRING)
    private SubscriptionStatus status = SubscriptionStatus.ACTIVE;

    @Column(precision = 10, scale = 2)
    private BigDecimal amountPaid;

    private Integer classesUsed = 0;

    private LocalDateTime lastPaymentDate;
    private LocalDateTime nextPaymentDate;
}

// FitnessClass.java
@Entity
@Table(name = "fitness_classes")
@Data
@NoArgsConstructor
public class FitnessClass {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;
    private String description;
    private String type; // YOGA, PILATES, ZUMBA, SPIN, BOXING
    private Integer durationMinutes;
    private Integer maxCapacity;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "trainer_id")
    private Trainer trainer;

    private LocalDateTime scheduledAt;
    private String location; // Room/Studio name
    private Boolean active = true;

    @OneToMany(mappedBy = "fitnessClass")
    private List<ClassBooking> bookings = new ArrayList<>();

    public Integer getAvailableSpots() {
        long confirmed = bookings.stream()
            .filter(b -> b.getStatus() == BookingStatus.CONFIRMED)
            .count();
        return maxCapacity - (int) confirmed;
    }
}

// Trainer.java
@Entity
@Table(name = "trainers")
@Data
@NoArgsConstructor
public class Trainer {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String firstName;
    private String lastName;
    private String email;
    private String phone;
    private String specialization;
    private String certification;
    private String bio;
    private String photoUrl;
    private Boolean available = true;
}

// ClassBooking.java
@Entity
@Table(name = "class_bookings",
       uniqueConstraints = @UniqueConstraint(columnNames = {"member_id", "fitness_class_id"}))
@Data
@NoArgsConstructor
public class ClassBooking {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne
    @JoinColumn(name = "member_id")
    private Member member;

    @ManyToOne
    @JoinColumn(name = "fitness_class_id")
    private FitnessClass fitnessClass;

    @Enumerated(EnumType.STRING)
    private BookingStatus status = BookingStatus.CONFIRMED;

    private Boolean attended = false;

    @CreatedDate
    private LocalDateTime bookedAt;
}

// GymCheckIn.java
@Entity
@Table(name = "gym_checkins")
@Data
@NoArgsConstructor
public class GymCheckIn {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne
    @JoinColumn(name = "member_id")
    private Member member;

    @CreatedDate
    private LocalDateTime checkInTime;

    private LocalDateTime checkOutTime;
    private String method; // CARD, APP, FINGERPRINT
}

public enum MemberStatus { ACTIVE, SUSPENDED, EXPIRED, CANCELLED }
public enum SubscriptionStatus { ACTIVE, EXPIRED, CANCELLED, SUSPENDED }
public enum BookingStatus { CONFIRMED, WAITLISTED, CANCELLED }
```

### Service Layer

```java
// GymService.java
@Service
@Transactional
@RequiredArgsConstructor
@Slf4j
public class GymService {

    private final MemberRepository memberRepository;
    private final MembershipSubscriptionRepository subscriptionRepository;
    private final MembershipPlanRepository planRepository;
    private final FitnessClassRepository classRepository;
    private final ClassBookingRepository classBookingRepository;
    private final GymCheckInRepository checkInRepository;
    private final EmailService emailService;

    public MemberDTO enrollMembership(Long memberId, Long planId) {
        Member member = memberRepository.findById(memberId)
            .orElseThrow(() -> new ResourceNotFoundException("Member not found"));

        MembershipPlan plan = planRepository.findById(planId)
            .orElseThrow(() -> new ResourceNotFoundException("Plan not found"));

        // Expire existing active subscription
        subscriptionRepository.findActiveByMemberId(memberId).ifPresent(sub -> {
            sub.setStatus(SubscriptionStatus.CANCELLED);
            subscriptionRepository.save(sub);
        });

        MembershipSubscription subscription = new MembershipSubscription();
        subscription.setMember(member);
        subscription.setPlan(plan);
        subscription.setStartDate(LocalDate.now());
        subscription.setEndDate(LocalDate.now().plusDays(plan.getDurationDays()));
        subscription.setAmountPaid(plan.getPrice());
        subscription.setNextPaymentDate(LocalDateTime.now().plusDays(plan.getDurationDays()));
        subscription.setLastPaymentDate(LocalDateTime.now());

        subscriptionRepository.save(subscription);
        member.setStatus(MemberStatus.ACTIVE);
        memberRepository.save(member);

        emailService.sendMembershipConfirmation(member.getEmail(), subscription);

        return toMemberDTO(member);
    }

    public ClassBookingDTO bookClass(Long memberId, Long classId) {
        Member member = memberRepository.findById(memberId)
            .orElseThrow(() -> new ResourceNotFoundException("Member not found"));

        FitnessClass fitnessClass = classRepository.findById(classId)
            .orElseThrow(() -> new ResourceNotFoundException("Class not found"));

        // Check active membership
        MembershipSubscription sub = subscriptionRepository.findActiveByMemberId(memberId)
            .orElseThrow(() -> new BadRequestException("No active membership"));

        if (sub.getStatus() != SubscriptionStatus.ACTIVE) {
            throw new BadRequestException("Membership is not active");
        }

        // Check if already booked
        if (classBookingRepository.existsByMemberIdAndFitnessClassId(memberId, classId)) {
            throw new ConflictException("Already booked this class");
        }

        // Check class limits
        MembershipPlan plan = sub.getPlan();
        if (plan.getClassesAllowed() != null && sub.getClassesUsed() >= plan.getClassesAllowed()) {
            throw new BadRequestException("Class limit reached for current membership");
        }

        ClassBooking booking = new ClassBooking();
        booking.setMember(member);
        booking.setFitnessClass(fitnessClass);

        // Check capacity - add to waitlist if full
        if (fitnessClass.getAvailableSpots() <= 0) {
            booking.setStatus(BookingStatus.WAITLISTED);
        }

        // Increment classes used
        if (plan.getClassesAllowed() != null) {
            sub.setClassesUsed(sub.getClassesUsed() + 1);
            subscriptionRepository.save(sub);
        }

        ClassBooking saved = classBookingRepository.save(booking);

        if (booking.getStatus() == BookingStatus.CONFIRMED) {
            emailService.sendClassBookingConfirmation(member.getEmail(), fitnessClass);
        }

        return toClassBookingDTO(saved);
    }

    public GymCheckInDTO checkIn(Long memberId, String method) {
        Member member = memberRepository.findById(memberId)
            .orElseThrow(() -> new ResourceNotFoundException("Member not found"));

        // Validate active membership
        MembershipSubscription sub = subscriptionRepository.findActiveByMemberId(memberId)
            .orElseThrow(() -> new BadRequestException("No active membership. Please renew."));

        if (sub.getEndDate().isBefore(LocalDate.now())) {
            sub.setStatus(SubscriptionStatus.EXPIRED);
            subscriptionRepository.save(sub);
            throw new BadRequestException("Membership has expired. Please renew.");
        }

        GymCheckIn checkIn = new GymCheckIn();
        checkIn.setMember(member);
        checkIn.setMethod(method);

        return toCheckInDTO(checkInRepository.save(checkIn));
    }

    @Scheduled(cron = "0 0 8 * * *") // Every day at 8am
    public void sendExpirationReminders() {
        LocalDate threeDaysFromNow = LocalDate.now().plusDays(3);
        List<MembershipSubscription> expiringSoon = subscriptionRepository
            .findByStatusAndEndDateBefore(SubscriptionStatus.ACTIVE, threeDaysFromNow);

        for (MembershipSubscription sub : expiringSoon) {
            emailService.sendMembershipExpirationReminder(sub.getMember().getEmail(), sub);
            log.info("Sent expiration reminder to member: {}", sub.getMember().getEmail());
        }
    }

    public List<FitnessClassDTO> getUpcomingClasses(String type, LocalDate date) {
        LocalDateTime start = date != null ? date.atStartOfDay() : LocalDateTime.now();
        LocalDateTime end = date != null ? date.plusDays(7).atStartOfDay() : LocalDateTime.now().plusDays(7);

        return classRepository.findUpcomingClasses(type, start, end).stream()
            .map(this::toClassDTO)
            .collect(Collectors.toList());
    }

    private MemberDTO toMemberDTO(Member m) {
        return MemberDTO.builder()
            .id(m.getId())
            .memberCode(m.getMemberCode())
            .fullName(m.getFirstName() + " " + m.getLastName())
            .email(m.getEmail())
            .status(m.getStatus())
            .build();
    }

    private ClassBookingDTO toClassBookingDTO(ClassBooking b) {
        return ClassBookingDTO.builder()
            .id(b.getId())
            .className(b.getFitnessClass().getName())
            .scheduledAt(b.getFitnessClass().getScheduledAt())
            .trainerName(b.getFitnessClass().getTrainer().getFirstName() + " " + 
                        b.getFitnessClass().getTrainer().getLastName())
            .status(b.getStatus())
            .bookedAt(b.getBookedAt())
            .build();
    }

    private GymCheckInDTO toCheckInDTO(GymCheckIn c) {
        return GymCheckInDTO.builder()
            .id(c.getId())
            .memberName(c.getMember().getFirstName() + " " + c.getMember().getLastName())
            .checkInTime(c.getCheckInTime())
            .method(c.getMethod())
            .build();
    }

    private FitnessClassDTO toClassDTO(FitnessClass fc) {
        return FitnessClassDTO.builder()
            .id(fc.getId())
            .name(fc.getName())
            .type(fc.getType())
            .scheduledAt(fc.getScheduledAt())
            .durationMinutes(fc.getDurationMinutes())
            .trainerName(fc.getTrainer() != null ?
                fc.getTrainer().getFirstName() + " " + fc.getTrainer().getLastName() : null)
            .availableSpots(fc.getAvailableSpots())
            .maxCapacity(fc.getMaxCapacity())
            .build();
    }
}
```

### REST Controller

```java
// GymController.java
@RestController
@RequestMapping("/api/v1/gym")
@RequiredArgsConstructor
public class GymController {

    private final GymService gymService;

    @GetMapping("/classes")
    public ResponseEntity<List<FitnessClassDTO>> getUpcomingClasses(
            @RequestParam(required = false) String type,
            @RequestParam(required = false) @DateTimeFormat(iso = DateTimeFormat.ISO.DATE) LocalDate date) {
        return ResponseEntity.ok(gymService.getUpcomingClasses(type, date));
    }

    @PostMapping("/classes/{classId}/book")
    public ResponseEntity<ClassBookingDTO> bookClass(
            @AuthenticationPrincipal UserPrincipal user,
            @PathVariable Long classId) {
        return ResponseEntity.ok(gymService.bookClass(user.getId(), classId));
    }

    @DeleteMapping("/classes/{classId}/book")
    public ResponseEntity<Void> cancelBooking(
            @AuthenticationPrincipal UserPrincipal user,
            @PathVariable Long classId) {
        gymService.cancelClassBooking(user.getId(), classId);
        return ResponseEntity.noContent().build();
    }

    @PostMapping("/members/{memberId}/membership")
    public ResponseEntity<MemberDTO> enrollMembership(
            @PathVariable Long memberId,
            @RequestParam Long planId) {
        return ResponseEntity.ok(gymService.enrollMembership(memberId, planId));
    }

    @PostMapping("/check-in")
    public ResponseEntity<GymCheckInDTO> checkIn(
            @AuthenticationPrincipal UserPrincipal user,
            @RequestParam(defaultValue = "APP") String method) {
        return ResponseEntity.ok(gymService.checkIn(user.getId(), method));
    }

    @GetMapping("/members/{memberId}/history")
    public ResponseEntity<Page<GymCheckInDTO>> getCheckInHistory(
            @PathVariable Long memberId,
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size) {
        return ResponseEntity.ok(gymService.getCheckInHistory(memberId, PageRequest.of(page, size)));
    }

    @GetMapping("/plans")
    public ResponseEntity<List<MembershipPlanDTO>> getMembershipPlans() {
        return ResponseEntity.ok(gymService.getMembershipPlans());
    }
}
```

### Flyway Migration

```sql
-- V15__init_gym.sql
CREATE TABLE membership_plans (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    description TEXT,
    duration_days INT NOT NULL,
    price DECIMAL(10,2) NOT NULL,
    classes_allowed INT,
    allow_guest_pass BOOLEAN DEFAULT FALSE,
    guest_pass_count INT DEFAULT 0,
    includes_trainer BOOLEAN DEFAULT FALSE,
    active BOOLEAN DEFAULT TRUE
);

CREATE TABLE members (
    id BIGSERIAL PRIMARY KEY,
    member_code VARCHAR(20) UNIQUE,
    first_name VARCHAR(100) NOT NULL,
    last_name VARCHAR(100) NOT NULL,
    email VARCHAR(255) UNIQUE,
    phone VARCHAR(20),
    date_of_birth DATE,
    gender VARCHAR(10),
    status VARCHAR(20) DEFAULT 'ACTIVE',
    emergency_contact_name VARCHAR(255),
    emergency_contact_phone VARCHAR(20),
    health_notes TEXT,
    joined_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE membership_subscriptions (
    id BIGSERIAL PRIMARY KEY,
    member_id BIGINT NOT NULL REFERENCES members(id),
    plan_id BIGINT NOT NULL REFERENCES membership_plans(id),
    start_date DATE NOT NULL,
    end_date DATE NOT NULL,
    status VARCHAR(20) DEFAULT 'ACTIVE',
    amount_paid DECIMAL(10,2),
    classes_used INT DEFAULT 0,
    last_payment_date TIMESTAMP,
    next_payment_date TIMESTAMP
);

CREATE TABLE trainers (
    id BIGSERIAL PRIMARY KEY,
    first_name VARCHAR(100) NOT NULL,
    last_name VARCHAR(100) NOT NULL,
    email VARCHAR(255),
    phone VARCHAR(20),
    specialization VARCHAR(255),
    certification TEXT,
    bio TEXT,
    photo_url VARCHAR(500),
    available BOOLEAN DEFAULT TRUE
);

CREATE TABLE fitness_classes (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    type VARCHAR(50),
    duration_minutes INT,
    max_capacity INT,
    trainer_id BIGINT REFERENCES trainers(id),
    scheduled_at TIMESTAMP NOT NULL,
    location VARCHAR(100),
    active BOOLEAN DEFAULT TRUE
);

CREATE TABLE class_bookings (
    id BIGSERIAL PRIMARY KEY,
    member_id BIGINT NOT NULL REFERENCES members(id),
    fitness_class_id BIGINT NOT NULL REFERENCES fitness_classes(id),
    status VARCHAR(20) DEFAULT 'CONFIRMED',
    attended BOOLEAN DEFAULT FALSE,
    booked_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE (member_id, fitness_class_id)
);

CREATE TABLE gym_checkins (
    id BIGSERIAL PRIMARY KEY,
    member_id BIGINT NOT NULL REFERENCES members(id),
    check_in_time TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    check_out_time TIMESTAMP,
    method VARCHAR(20)
);
```

### ตัวอย่าง API Calls

```bash
# สมัครแพ็คเกจ
curl -X POST "http://localhost:8080/api/v1/gym/members/1/membership?planId=2" \
  -H "Authorization: Bearer {token}"

# ดูคลาสที่จะมีในสัปดาห์นี้
curl "http://localhost:8080/api/v1/gym/classes?type=YOGA" \
  -H "Authorization: Bearer {token}"

# จองคลาส
curl -X POST http://localhost:8080/api/v1/gym/classes/5/book \
  -H "Authorization: Bearer {token}"

# Check-in ฟิตเนส
curl -X POST "http://localhost:8080/api/v1/gym/check-in?method=APP" \
  -H "Authorization: Bearer {token}"

# ดูประวัติ check-in
curl http://localhost:8080/api/v1/gym/members/1/history \
  -H "Authorization: Bearer {token}"
```

---

*[← Part 102: Business Management Systems](./part-102-business-systems.md) | [Part 104: Platform & SaaS →](./part-104-platform-saas.md)*

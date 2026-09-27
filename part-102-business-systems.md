# Part 102: โปรเจค 6-10 — Business Management Systems

> **ระดับ:** ระดับโลก | **เวลาเรียนรู้:** 10-15 ชั่วโมง | **โปรเจค:** 5 โปรเจคสมบูรณ์

ในส่วนนี้เราจะสร้างระบบจัดการธุรกิจที่ใช้งานจริงในองค์กร ครอบคลุมระบบ HR, CRM, โรงพยาบาล, โรงแรม และร้านอาหาร

---

## โปรเจคที่ 6: HR Management System

### ภาพรวมระบบ

ระบบจัดการทรัพยากรบุคคลครบวงจร รองรับข้อมูลพนักงาน แผนก ตำแหน่งงาน การลา การติดตามการเข้างาน และการคำนวณเงินเดือนเบื้องต้น ระบบนี้ออกแบบมาให้ใช้งานได้จริงในองค์กรขนาดกลางถึงใหญ่

### Entity Classes

```java
// Department.java
@Entity
@Table(name = "departments")
@Data
@NoArgsConstructor
public class Department {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, unique = true)
    private String name;

    private String description;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "manager_id")
    private Employee manager;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "parent_department_id")
    private Department parentDepartment;

    @OneToMany(mappedBy = "department")
    private List<Employee> employees = new ArrayList<>();

    @Column(nullable = false)
    private Boolean active = true;

    @CreatedDate
    private LocalDateTime createdAt;
}

// Position.java
@Entity
@Table(name = "positions")
@Data
@NoArgsConstructor
public class Position {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String title;

    private String description;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "department_id")
    private Department department;

    @Column(precision = 12, scale = 2)
    private BigDecimal minSalary;

    @Column(precision = 12, scale = 2)
    private BigDecimal maxSalary;

    private String level; // JUNIOR, MID, SENIOR, LEAD, MANAGER
}

// Employee.java
@Entity
@Table(name = "employees")
@Data
@NoArgsConstructor
public class Employee {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(unique = true, nullable = false)
    private String employeeCode;

    private String firstName;
    private String lastName;

    @Column(unique = true)
    private String email;

    private String phone;
    private LocalDate dateOfBirth;
    private String nationalId;
    private String gender;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "department_id")
    private Department department;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "position_id")
    private Position position;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "manager_id")
    private Employee manager;

    @Column(precision = 12, scale = 2)
    private BigDecimal baseSalary;

    private LocalDate hireDate;
    private LocalDate terminationDate;

    @Enumerated(EnumType.STRING)
    private EmploymentStatus status = EmploymentStatus.ACTIVE;

    @Enumerated(EnumType.STRING)
    private EmploymentType employmentType = EmploymentType.FULL_TIME;

    private String profileImageUrl;

    @Embedded
    private Address address;

    @OneToMany(mappedBy = "employee")
    private List<LeaveRequest> leaveRequests = new ArrayList<>();

    @CreatedDate
    private LocalDateTime createdAt;
}

// LeaveRequest.java
@Entity
@Table(name = "leave_requests")
@Data
@NoArgsConstructor
public class LeaveRequest {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "employee_id")
    private Employee employee;

    @Enumerated(EnumType.STRING)
    private LeaveType leaveType;

    private LocalDate startDate;
    private LocalDate endDate;

    private Integer totalDays;
    private String reason;

    @Enumerated(EnumType.STRING)
    private LeaveStatus status = LeaveStatus.PENDING;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "approved_by")
    private Employee approvedBy;

    private String rejectionReason;
    private LocalDateTime approvedAt;

    @CreatedDate
    private LocalDateTime createdAt;
}

// Attendance.java
@Entity
@Table(name = "attendance")
@Data
@NoArgsConstructor
public class Attendance {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "employee_id")
    private Employee employee;

    @Column(nullable = false)
    private LocalDate workDate;

    private LocalTime checkInTime;
    private LocalTime checkOutTime;

    @Enumerated(EnumType.STRING)
    private AttendanceStatus status = AttendanceStatus.PRESENT;

    private String location;
    private String notes;

    // Calculated fields
    private Double totalWorkHours;
    private Double overtimeHours;
}

// Enums
public enum EmploymentStatus { ACTIVE, ON_LEAVE, SUSPENDED, TERMINATED }
public enum EmploymentType { FULL_TIME, PART_TIME, CONTRACT, INTERN }
public enum LeaveType { ANNUAL, SICK, MATERNITY, PATERNITY, UNPAID, EMERGENCY }
public enum LeaveStatus { PENDING, APPROVED, REJECTED, CANCELLED }
public enum AttendanceStatus { PRESENT, ABSENT, LATE, HALF_DAY, ON_LEAVE, HOLIDAY }
```

### Repository Layer

```java
// EmployeeRepository.java
@Repository
public interface EmployeeRepository extends JpaRepository<Employee, Long> {

    Page<Employee> findByDepartmentIdAndStatus(Long deptId, EmploymentStatus status, Pageable pageable);

    Optional<Employee> findByEmployeeCode(String code);

    Optional<Employee> findByEmail(String email);

    @Query("SELECT e FROM Employee e WHERE e.status = 'ACTIVE' AND " +
           "(LOWER(e.firstName) LIKE LOWER(CONCAT('%', :q, '%')) OR " +
           "LOWER(e.lastName) LIKE LOWER(CONCAT('%', :q, '%')) OR " +
           "LOWER(e.email) LIKE LOWER(CONCAT('%', :q, '%')))")
    Page<Employee> searchEmployees(@Param("q") String query, Pageable pageable);

    @Query("SELECT COUNT(e) FROM Employee e WHERE e.department.id = :deptId AND e.status = 'ACTIVE'")
    long countActiveByDepartment(@Param("deptId") Long deptId);

    List<Employee> findByManagerIdAndStatus(Long managerId, EmploymentStatus status);

    @Query("SELECT e FROM Employee e WHERE e.hireDate BETWEEN :from AND :to")
    List<Employee> findNewHires(@Param("from") LocalDate from, @Param("to") LocalDate to);
}

// LeaveRequestRepository.java
@Repository
public interface LeaveRequestRepository extends JpaRepository<LeaveRequest, Long> {

    Page<LeaveRequest> findByEmployeeIdOrderByCreatedAtDesc(Long employeeId, Pageable pageable);

    Page<LeaveRequest> findByStatus(LeaveStatus status, Pageable pageable);

    @Query("SELECT lr FROM LeaveRequest lr WHERE lr.employee.manager.id = :managerId AND lr.status = 'PENDING'")
    List<LeaveRequest> findPendingByManager(@Param("managerId") Long managerId);

    @Query("SELECT COALESCE(SUM(lr.totalDays), 0) FROM LeaveRequest lr " +
           "WHERE lr.employee.id = :empId AND lr.leaveType = :type AND lr.status = 'APPROVED' " +
           "AND YEAR(lr.startDate) = :year")
    int getTotalLeaveDays(@Param("empId") Long empId, @Param("type") LeaveType type, @Param("year") int year);

    @Query("SELECT lr FROM LeaveRequest lr WHERE lr.startDate <= :date AND lr.endDate >= :date " +
           "AND lr.status = 'APPROVED'")
    List<LeaveRequest> findEmployeesOnLeave(@Param("date") LocalDate date);
}

// AttendanceRepository.java
@Repository
public interface AttendanceRepository extends JpaRepository<Attendance, Long> {

    Optional<Attendance> findByEmployeeIdAndWorkDate(Long empId, LocalDate date);

    List<Attendance> findByEmployeeIdAndWorkDateBetween(Long empId, LocalDate from, LocalDate to);

    @Query("SELECT a FROM Attendance a WHERE a.workDate = :date AND a.status = 'ABSENT'")
    List<Attendance> findAbsencesByDate(@Param("date") LocalDate date);

    @Query("SELECT AVG(a.totalWorkHours) FROM Attendance a WHERE a.employee.id = :empId " +
           "AND a.workDate BETWEEN :from AND :to")
    Double getAverageWorkHours(@Param("empId") Long empId, @Param("from") LocalDate from, @Param("to") LocalDate to);
}
```

### Service Layer

```java
// HRService.java
@Service
@Transactional
@RequiredArgsConstructor
@Slf4j
public class HRService {

    private final EmployeeRepository employeeRepository;
    private final DepartmentRepository departmentRepository;
    private final PositionRepository positionRepository;
    private final LeaveRequestRepository leaveRequestRepository;
    private final AttendanceRepository attendanceRepository;
    private final EmailService emailService;

    public EmployeeDTO createEmployee(CreateEmployeeRequest request) {
        if (employeeRepository.findByEmail(request.getEmail()).isPresent()) {
            throw new ConflictException("Email already in use: " + request.getEmail());
        }

        Department dept = departmentRepository.findById(request.getDepartmentId())
            .orElseThrow(() -> new ResourceNotFoundException("Department not found"));

        Position position = positionRepository.findById(request.getPositionId())
            .orElseThrow(() -> new ResourceNotFoundException("Position not found"));

        Employee employee = new Employee();
        employee.setEmployeeCode(generateEmployeeCode(dept));
        employee.setFirstName(request.getFirstName());
        employee.setLastName(request.getLastName());
        employee.setEmail(request.getEmail());
        employee.setPhone(request.getPhone());
        employee.setDateOfBirth(request.getDateOfBirth());
        employee.setDepartment(dept);
        employee.setPosition(position);
        employee.setBaseSalary(request.getBaseSalary());
        employee.setHireDate(request.getHireDate() != null ? request.getHireDate() : LocalDate.now());
        employee.setEmploymentType(request.getEmploymentType());

        Employee saved = employeeRepository.save(employee);

        // Send welcome email
        emailService.sendWelcomeEmail(saved.getEmail(), saved.getFirstName());

        return toDTO(saved);
    }

    public LeaveRequestDTO applyForLeave(Long employeeId, ApplyLeaveRequest request) {
        Employee employee = employeeRepository.findById(employeeId)
            .orElseThrow(() -> new ResourceNotFoundException("Employee not found"));

        // Check leave balance
        int usedDays = leaveRequestRepository.getTotalLeaveDays(
            employeeId, request.getLeaveType(), LocalDate.now().getYear());
        int allowedDays = getLeaveAllowance(request.getLeaveType());
        int requestedDays = calculateWorkDays(request.getStartDate(), request.getEndDate());

        if (usedDays + requestedDays > allowedDays) {
            throw new BadRequestException("Insufficient leave balance. Used: " + usedDays + 
                                          ", Allowed: " + allowedDays);
        }

        LeaveRequest leave = new LeaveRequest();
        leave.setEmployee(employee);
        leave.setLeaveType(request.getLeaveType());
        leave.setStartDate(request.getStartDate());
        leave.setEndDate(request.getEndDate());
        leave.setTotalDays(requestedDays);
        leave.setReason(request.getReason());

        LeaveRequest saved = leaveRequestRepository.save(leave);

        // Notify manager
        if (employee.getManager() != null) {
            emailService.notifyLeaveRequest(employee.getManager().getEmail(), saved);
        }

        return toLeaveDTO(saved);
    }

    public LeaveRequestDTO approveLeave(Long leaveId, Long approverId, boolean approved, String reason) {
        LeaveRequest leave = leaveRequestRepository.findById(leaveId)
            .orElseThrow(() -> new ResourceNotFoundException("Leave request not found"));

        if (leave.getStatus() != LeaveStatus.PENDING) {
            throw new BadRequestException("Leave request is not pending");
        }

        Employee approver = employeeRepository.findById(approverId)
            .orElseThrow(() -> new ResourceNotFoundException("Approver not found"));

        leave.setApprovedBy(approver);
        leave.setApprovedAt(LocalDateTime.now());

        if (approved) {
            leave.setStatus(LeaveStatus.APPROVED);
            emailService.sendLeaveApproval(leave.getEmployee().getEmail(), leave);
        } else {
            leave.setStatus(LeaveStatus.REJECTED);
            leave.setRejectionReason(reason);
            emailService.sendLeaveRejection(leave.getEmployee().getEmail(), leave, reason);
        }

        return toLeaveDTO(leaveRequestRepository.save(leave));
    }

    public AttendanceDTO recordCheckIn(Long employeeId, String location) {
        Employee employee = employeeRepository.findById(employeeId)
            .orElseThrow(() -> new ResourceNotFoundException("Employee not found"));

        LocalDate today = LocalDate.now();
        Optional<Attendance> existing = attendanceRepository.findByEmployeeIdAndWorkDate(employeeId, today);

        if (existing.isPresent() && existing.get().getCheckInTime() != null) {
            throw new BadRequestException("Already checked in today");
        }

        Attendance attendance = existing.orElse(new Attendance());
        attendance.setEmployee(employee);
        attendance.setWorkDate(today);
        attendance.setCheckInTime(LocalTime.now());
        attendance.setLocation(location);

        LocalTime standardStart = LocalTime.of(9, 0);
        attendance.setStatus(LocalTime.now().isAfter(standardStart.plusMinutes(15)) ?
            AttendanceStatus.LATE : AttendanceStatus.PRESENT);

        return toAttendanceDTO(attendanceRepository.save(attendance));
    }

    public AttendanceDTO recordCheckOut(Long employeeId) {
        LocalDate today = LocalDate.now();
        Attendance attendance = attendanceRepository.findByEmployeeIdAndWorkDate(employeeId, today)
            .orElseThrow(() -> new BadRequestException("No check-in record for today"));

        if (attendance.getCheckOutTime() != null) {
            throw new BadRequestException("Already checked out today");
        }

        attendance.setCheckOutTime(LocalTime.now());

        // Calculate work hours
        long minutes = ChronoUnit.MINUTES.between(attendance.getCheckInTime(), LocalTime.now());
        double workHours = minutes / 60.0;
        attendance.setTotalWorkHours(workHours);

        // Calculate overtime (> 8 hours)
        double overtime = Math.max(0, workHours - 8.0);
        attendance.setOvertimeHours(overtime);

        return toAttendanceDTO(attendanceRepository.save(attendance));
    }

    public PayrollSummary calculatePayroll(Long employeeId, int month, int year) {
        Employee employee = employeeRepository.findById(employeeId)
            .orElseThrow(() -> new ResourceNotFoundException("Employee not found"));

        LocalDate from = LocalDate.of(year, month, 1);
        LocalDate to = from.withDayOfMonth(from.lengthOfMonth());

        List<Attendance> attendances = attendanceRepository.findByEmployeeIdAndWorkDateBetween(employeeId, from, to);

        int presentDays = (int) attendances.stream()
            .filter(a -> a.getStatus() == AttendanceStatus.PRESENT || a.getStatus() == AttendanceStatus.LATE)
            .count();

        double totalOvertime = attendances.stream()
            .mapToDouble(a -> a.getOvertimeHours() != null ? a.getOvertimeHours() : 0)
            .sum();

        // Get approved leave days
        int leaveDays = leaveRequestRepository.getTotalLeaveDays(employeeId, null, year);

        BigDecimal dailyRate = employee.getBaseSalary().divide(BigDecimal.valueOf(26), 2, RoundingMode.HALF_UP);
        BigDecimal hourlyRate = employee.getBaseSalary().divide(BigDecimal.valueOf(208), 2, RoundingMode.HALF_UP);
        BigDecimal overtimePay = hourlyRate.multiply(new BigDecimal("1.5"))
            .multiply(BigDecimal.valueOf(totalOvertime));

        BigDecimal grossPay = dailyRate.multiply(BigDecimal.valueOf(presentDays)).add(overtimePay);

        return PayrollSummary.builder()
            .employeeId(employeeId)
            .employeeName(employee.getFirstName() + " " + employee.getLastName())
            .month(month)
            .year(year)
            .baseSalary(employee.getBaseSalary())
            .presentDays(presentDays)
            .overtimeHours(totalOvertime)
            .overtimePay(overtimePay)
            .grossPay(grossPay)
            .build();
    }

    private int calculateWorkDays(LocalDate from, LocalDate to) {
        int days = 0;
        LocalDate current = from;
        while (!current.isAfter(to)) {
            DayOfWeek day = current.getDayOfWeek();
            if (day != DayOfWeek.SATURDAY && day != DayOfWeek.SUNDAY) {
                days++;
            }
            current = current.plusDays(1);
        }
        return days;
    }

    private int getLeaveAllowance(LeaveType type) {
        return switch (type) {
            case ANNUAL -> 15;
            case SICK -> 30;
            case MATERNITY -> 90;
            case PATERNITY -> 15;
            case UNPAID -> 30;
            case EMERGENCY -> 3;
        };
    }

    private String generateEmployeeCode(Department dept) {
        String prefix = dept.getName().substring(0, Math.min(3, dept.getName().length())).toUpperCase();
        long count = employeeRepository.countActiveByDepartment(dept.getId()) + 1;
        return String.format("%s%04d", prefix, count);
    }

    private EmployeeDTO toDTO(Employee emp) {
        return EmployeeDTO.builder()
            .id(emp.getId())
            .employeeCode(emp.getEmployeeCode())
            .fullName(emp.getFirstName() + " " + emp.getLastName())
            .email(emp.getEmail())
            .departmentName(emp.getDepartment() != null ? emp.getDepartment().getName() : null)
            .positionTitle(emp.getPosition() != null ? emp.getPosition().getTitle() : null)
            .status(emp.getStatus())
            .hireDate(emp.getHireDate())
            .build();
    }

    private LeaveRequestDTO toLeaveDTO(LeaveRequest lr) {
        return LeaveRequestDTO.builder()
            .id(lr.getId())
            .employeeName(lr.getEmployee().getFirstName() + " " + lr.getEmployee().getLastName())
            .leaveType(lr.getLeaveType())
            .startDate(lr.getStartDate())
            .endDate(lr.getEndDate())
            .totalDays(lr.getTotalDays())
            .status(lr.getStatus())
            .reason(lr.getReason())
            .build();
    }

    private AttendanceDTO toAttendanceDTO(Attendance att) {
        return AttendanceDTO.builder()
            .id(att.getId())
            .workDate(att.getWorkDate())
            .checkInTime(att.getCheckInTime())
            .checkOutTime(att.getCheckOutTime())
            .status(att.getStatus())
            .totalWorkHours(att.getTotalWorkHours())
            .build();
    }
}
```

### REST Controller

```java
// HRController.java
@RestController
@RequestMapping("/api/v1/hr")
@RequiredArgsConstructor
public class HRController {

    private final HRService hrService;

    // Employee endpoints
    @PostMapping("/employees")
    @PreAuthorize("hasRole('HR_ADMIN')")
    public ResponseEntity<EmployeeDTO> createEmployee(@Valid @RequestBody CreateEmployeeRequest request) {
        return ResponseEntity.status(HttpStatus.CREATED).body(hrService.createEmployee(request));
    }

    @GetMapping("/employees")
    public ResponseEntity<Page<EmployeeDTO>> listEmployees(
            @RequestParam(required = false) Long departmentId,
            @RequestParam(required = false) String search,
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size) {
        return ResponseEntity.ok(hrService.listEmployees(departmentId, search, PageRequest.of(page, size)));
    }

    @GetMapping("/employees/{id}")
    public ResponseEntity<EmployeeDTO> getEmployee(@PathVariable Long id) {
        return ResponseEntity.ok(hrService.getEmployee(id));
    }

    @PutMapping("/employees/{id}")
    @PreAuthorize("hasRole('HR_ADMIN')")
    public ResponseEntity<EmployeeDTO> updateEmployee(
            @PathVariable Long id,
            @Valid @RequestBody UpdateEmployeeRequest request) {
        return ResponseEntity.ok(hrService.updateEmployee(id, request));
    }

    // Leave management endpoints
    @PostMapping("/leave/apply")
    public ResponseEntity<LeaveRequestDTO> applyLeave(
            @AuthenticationPrincipal UserPrincipal user,
            @Valid @RequestBody ApplyLeaveRequest request) {
        return ResponseEntity.ok(hrService.applyForLeave(user.getId(), request));
    }

    @GetMapping("/leave/pending")
    @PreAuthorize("hasRole('MANAGER') or hasRole('HR_ADMIN')")
    public ResponseEntity<List<LeaveRequestDTO>> getPendingLeaves(
            @AuthenticationPrincipal UserPrincipal user) {
        return ResponseEntity.ok(hrService.getPendingLeaves(user.getId()));
    }

    @PatchMapping("/leave/{id}/approve")
    @PreAuthorize("hasRole('MANAGER') or hasRole('HR_ADMIN')")
    public ResponseEntity<LeaveRequestDTO> processLeave(
            @PathVariable Long id,
            @AuthenticationPrincipal UserPrincipal user,
            @RequestParam boolean approved,
            @RequestParam(required = false) String reason) {
        return ResponseEntity.ok(hrService.approveLeave(id, user.getId(), approved, reason));
    }

    // Attendance endpoints
    @PostMapping("/attendance/check-in")
    public ResponseEntity<AttendanceDTO> checkIn(
            @AuthenticationPrincipal UserPrincipal user,
            @RequestParam(required = false) String location) {
        return ResponseEntity.ok(hrService.recordCheckIn(user.getId(), location));
    }

    @PostMapping("/attendance/check-out")
    public ResponseEntity<AttendanceDTO> checkOut(@AuthenticationPrincipal UserPrincipal user) {
        return ResponseEntity.ok(hrService.recordCheckOut(user.getId()));
    }

    @GetMapping("/attendance")
    public ResponseEntity<List<AttendanceDTO>> getAttendance(
            @AuthenticationPrincipal UserPrincipal user,
            @RequestParam @DateTimeFormat(iso = DateTimeFormat.ISO.DATE) LocalDate from,
            @RequestParam @DateTimeFormat(iso = DateTimeFormat.ISO.DATE) LocalDate to) {
        return ResponseEntity.ok(hrService.getAttendanceHistory(user.getId(), from, to));
    }

    // Payroll
    @GetMapping("/payroll/{employeeId}")
    @PreAuthorize("hasRole('HR_ADMIN') or @securityService.isSelf(#employeeId, authentication)")
    public ResponseEntity<PayrollSummary> getPayroll(
            @PathVariable Long employeeId,
            @RequestParam int month,
            @RequestParam int year) {
        return ResponseEntity.ok(hrService.calculatePayroll(employeeId, month, year));
    }
}
```

### Flyway Migration

```sql
-- V6__init_hr.sql
CREATE TABLE departments (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL UNIQUE,
    description TEXT,
    manager_id BIGINT,
    parent_department_id BIGINT REFERENCES departments(id),
    active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE positions (
    id BIGSERIAL PRIMARY KEY,
    title VARCHAR(255) NOT NULL,
    description TEXT,
    department_id BIGINT REFERENCES departments(id),
    min_salary DECIMAL(12,2),
    max_salary DECIMAL(12,2),
    level VARCHAR(20)
);

CREATE TABLE employees (
    id BIGSERIAL PRIMARY KEY,
    employee_code VARCHAR(20) NOT NULL UNIQUE,
    first_name VARCHAR(100) NOT NULL,
    last_name VARCHAR(100) NOT NULL,
    email VARCHAR(255) UNIQUE,
    phone VARCHAR(20),
    date_of_birth DATE,
    national_id VARCHAR(20),
    gender VARCHAR(10),
    department_id BIGINT REFERENCES departments(id),
    position_id BIGINT REFERENCES positions(id),
    manager_id BIGINT REFERENCES employees(id),
    base_salary DECIMAL(12,2),
    hire_date DATE,
    termination_date DATE,
    status VARCHAR(20) DEFAULT 'ACTIVE',
    employment_type VARCHAR(20) DEFAULT 'FULL_TIME',
    profile_image_url VARCHAR(500),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE leave_requests (
    id BIGSERIAL PRIMARY KEY,
    employee_id BIGINT NOT NULL REFERENCES employees(id),
    leave_type VARCHAR(20) NOT NULL,
    start_date DATE NOT NULL,
    end_date DATE NOT NULL,
    total_days INT NOT NULL,
    reason TEXT,
    status VARCHAR(20) DEFAULT 'PENDING',
    approved_by BIGINT REFERENCES employees(id),
    rejection_reason TEXT,
    approved_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE attendance (
    id BIGSERIAL PRIMARY KEY,
    employee_id BIGINT NOT NULL REFERENCES employees(id),
    work_date DATE NOT NULL,
    check_in_time TIME,
    check_out_time TIME,
    status VARCHAR(20) DEFAULT 'PRESENT',
    location VARCHAR(255),
    notes TEXT,
    total_work_hours DOUBLE PRECISION,
    overtime_hours DOUBLE PRECISION,
    UNIQUE (employee_id, work_date)
);
```

---

## โปรเจคที่ 7: CRM System

### ภาพรวมระบบ

ระบบจัดการลูกค้าสัมพันธ์ (CRM) ที่รองรับการติดตาม Leads, ข้อมูลติดต่อ, บริษัทลูกค้า, Pipeline การขาย, กิจกรรมต่างๆ (การโทร/อีเมล/ประชุม) และการคาดการณ์มูลค่าการขาย

### Entity Classes

```java
// Company.java
@Entity
@Table(name = "companies")
@Data
@NoArgsConstructor
public class Company {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String name;

    private String website;
    private String industry;
    private String phone;
    private String address;
    private String city;
    private String country;
    private Integer employeeCount;

    @Column(precision = 15, scale = 2)
    private BigDecimal annualRevenue;

    @OneToMany(mappedBy = "company")
    private List<Contact> contacts = new ArrayList<>();

    @CreatedDate
    private LocalDateTime createdAt;
}

// Contact.java
@Entity
@Table(name = "contacts")
@Data
@NoArgsConstructor
public class Contact {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String firstName;
    private String lastName;
    private String email;
    private String phone;
    private String jobTitle;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "company_id")
    private Company company;

    @Enumerated(EnumType.STRING)
    private ContactStatus status = ContactStatus.ACTIVE;

    private String notes;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "owner_id")
    private User owner;

    @OneToMany(mappedBy = "contact")
    private List<Activity> activities = new ArrayList<>();

    @CreatedDate
    private LocalDateTime createdAt;
}

// Lead.java
@Entity
@Table(name = "leads")
@Data
@NoArgsConstructor
public class Lead {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String firstName;
    private String lastName;
    private String email;
    private String phone;
    private String company;
    private String jobTitle;

    @Enumerated(EnumType.STRING)
    private LeadStatus status = LeadStatus.NEW;

    @Enumerated(EnumType.STRING)
    private LeadSource source;

    private String notes;

    @Column(precision = 10, scale = 2)
    private BigDecimal estimatedValue;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "owner_id")
    private User owner;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "converted_to_contact_id")
    private Contact convertedContact;

    private LocalDateTime convertedAt;

    @CreatedDate
    private LocalDateTime createdAt;
}

// Deal.java
@Entity
@Table(name = "deals")
@Data
@NoArgsConstructor
public class Deal {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String title;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "contact_id")
    private Contact contact;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "company_id")
    private Company company;

    @Enumerated(EnumType.STRING)
    private DealStage stage = DealStage.NEW;

    @Column(precision = 15, scale = 2)
    private BigDecimal value;

    @Column(precision = 5, scale = 2)
    private BigDecimal probability; // 0-100%

    private LocalDate expectedCloseDate;
    private LocalDate closedDate;

    private String notes;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "owner_id")
    private User owner;

    @OneToMany(mappedBy = "deal")
    private List<Activity> activities = new ArrayList<>();

    @CreatedDate
    private LocalDateTime createdAt;
}

// Activity.java
@Entity
@Table(name = "activities")
@Data
@NoArgsConstructor
public class Activity {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Enumerated(EnumType.STRING)
    private ActivityType type;

    private String subject;

    @Column(columnDefinition = "TEXT")
    private String description;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "contact_id")
    private Contact contact;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "deal_id")
    private Deal deal;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "performed_by")
    private User performedBy;

    private LocalDateTime scheduledAt;
    private LocalDateTime completedAt;

    @Enumerated(EnumType.STRING)
    private ActivityStatus status = ActivityStatus.PLANNED;

    @CreatedDate
    private LocalDateTime createdAt;
}

// Enums
public enum LeadStatus { NEW, CONTACTED, QUALIFIED, UNQUALIFIED, CONVERTED }
public enum LeadSource { WEBSITE, REFERRAL, SOCIAL_MEDIA, EMAIL_CAMPAIGN, COLD_CALL, EVENT, OTHER }
public enum DealStage { NEW, QUALIFIED, PROPOSAL, NEGOTIATION, WON, LOST }
public enum ActivityType { CALL, EMAIL, MEETING, NOTE, TASK }
public enum ActivityStatus { PLANNED, COMPLETED, CANCELLED }
public enum ContactStatus { ACTIVE, INACTIVE }
```

### Service Layer

```java
// CRMService.java
@Service
@Transactional
@RequiredArgsConstructor
public class CRMService {

    private final LeadRepository leadRepository;
    private final ContactRepository contactRepository;
    private final CompanyRepository companyRepository;
    private final DealRepository dealRepository;
    private final ActivityRepository activityRepository;

    public ContactDTO convertLead(Long leadId, Long assignedTo) {
        Lead lead = leadRepository.findById(leadId)
            .orElseThrow(() -> new ResourceNotFoundException("Lead not found"));

        if (lead.getStatus() == LeadStatus.CONVERTED) {
            throw new BadRequestException("Lead already converted");
        }

        // Create or find company
        Company company = companyRepository.findByName(lead.getCompany())
            .orElseGet(() -> {
                Company c = new Company();
                c.setName(lead.getCompany());
                return companyRepository.save(c);
            });

        // Create contact from lead
        Contact contact = new Contact();
        contact.setFirstName(lead.getFirstName());
        contact.setLastName(lead.getLastName());
        contact.setEmail(lead.getEmail());
        contact.setPhone(lead.getPhone());
        contact.setJobTitle(lead.getJobTitle());
        contact.setCompany(company);

        Contact savedContact = contactRepository.save(contact);

        // Update lead status
        lead.setStatus(LeadStatus.CONVERTED);
        lead.setConvertedContact(savedContact);
        lead.setConvertedAt(LocalDateTime.now());
        leadRepository.save(lead);

        return toContactDTO(savedContact);
    }

    public DealDTO updateDealStage(Long dealId, DealStage newStage) {
        Deal deal = dealRepository.findById(dealId)
            .orElseThrow(() -> new ResourceNotFoundException("Deal not found"));

        DealStage oldStage = deal.getStage();
        deal.setStage(newStage);

        if (newStage == DealStage.WON || newStage == DealStage.LOST) {
            deal.setClosedDate(LocalDate.now());
            // Update probability
            deal.setProbability(newStage == DealStage.WON ? new BigDecimal("100") : BigDecimal.ZERO);
        }

        // Auto-update probability based on stage
        if (newStage != DealStage.WON && newStage != DealStage.LOST) {
            deal.setProbability(getStageProbability(newStage));
        }

        // Create activity log
        Activity activity = new Activity();
        activity.setDeal(deal);
        activity.setType(ActivityType.NOTE);
        activity.setSubject("Deal stage changed: " + oldStage + " -> " + newStage);
        activity.setStatus(ActivityStatus.COMPLETED);
        activity.setCompletedAt(LocalDateTime.now());
        activityRepository.save(activity);

        return toDealDTO(dealRepository.save(deal));
    }

    public PipelineSummary getPipelineSummary(Long ownerId) {
        List<Deal> deals = ownerId != null ?
            dealRepository.findByOwnerIdAndStageNotIn(ownerId, List.of(DealStage.WON, DealStage.LOST)) :
            dealRepository.findByStageNotIn(List.of(DealStage.WON, DealStage.LOST));

        Map<DealStage, List<Deal>> byStage = deals.stream()
            .collect(Collectors.groupingBy(Deal::getStage));

        BigDecimal totalValue = deals.stream()
            .map(Deal::getValue)
            .filter(Objects::nonNull)
            .reduce(BigDecimal.ZERO, BigDecimal::add);

        BigDecimal weightedValue = deals.stream()
            .filter(d -> d.getValue() != null && d.getProbability() != null)
            .map(d -> d.getValue().multiply(d.getProbability()).divide(new BigDecimal("100"), 2, RoundingMode.HALF_UP))
            .reduce(BigDecimal.ZERO, BigDecimal::add);

        Map<String, StageSummary> stageSummaries = new LinkedHashMap<>();
        for (DealStage stage : DealStage.values()) {
            if (stage != DealStage.WON && stage != DealStage.LOST) {
                List<Deal> stageDeals = byStage.getOrDefault(stage, List.of());
                BigDecimal stageValue = stageDeals.stream()
                    .map(Deal::getValue)
                    .filter(Objects::nonNull)
                    .reduce(BigDecimal.ZERO, BigDecimal::add);
                stageSummaries.put(stage.name(), new StageSummary(stageDeals.size(), stageValue));
            }
        }

        return PipelineSummary.builder()
            .totalDeals(deals.size())
            .totalValue(totalValue)
            .weightedValue(weightedValue)
            .byStage(stageSummaries)
            .build();
    }

    private BigDecimal getStageProbability(DealStage stage) {
        return switch (stage) {
            case NEW -> new BigDecimal("10");
            case QUALIFIED -> new BigDecimal("25");
            case PROPOSAL -> new BigDecimal("50");
            case NEGOTIATION -> new BigDecimal("75");
            default -> BigDecimal.ZERO;
        };
    }

    private ContactDTO toContactDTO(Contact c) {
        return ContactDTO.builder()
            .id(c.getId())
            .fullName(c.getFirstName() + " " + c.getLastName())
            .email(c.getEmail())
            .phone(c.getPhone())
            .companyName(c.getCompany() != null ? c.getCompany().getName() : null)
            .build();
    }

    private DealDTO toDealDTO(Deal d) {
        return DealDTO.builder()
            .id(d.getId())
            .title(d.getTitle())
            .stage(d.getStage())
            .value(d.getValue())
            .probability(d.getProbability())
            .expectedCloseDate(d.getExpectedCloseDate())
            .build();
    }
}
```

### REST Controller (CRM)

```java
// CRMController.java
@RestController
@RequestMapping("/api/v1/crm")
@RequiredArgsConstructor
public class CRMController {

    private final CRMService crmService;

    // Leads
    @PostMapping("/leads")
    public ResponseEntity<LeadDTO> createLead(@Valid @RequestBody CreateLeadRequest request) {
        return ResponseEntity.status(HttpStatus.CREATED).body(crmService.createLead(request));
    }

    @GetMapping("/leads")
    public ResponseEntity<Page<LeadDTO>> listLeads(
            @RequestParam(required = false) LeadStatus status,
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size) {
        return ResponseEntity.ok(crmService.listLeads(status, PageRequest.of(page, size)));
    }

    @PostMapping("/leads/{id}/convert")
    public ResponseEntity<ContactDTO> convertLead(
            @PathVariable Long id,
            @AuthenticationPrincipal UserPrincipal user) {
        return ResponseEntity.ok(crmService.convertLead(id, user.getId()));
    }

    // Deals / Pipeline
    @PostMapping("/deals")
    public ResponseEntity<DealDTO> createDeal(@Valid @RequestBody CreateDealRequest request) {
        return ResponseEntity.status(HttpStatus.CREATED).body(crmService.createDeal(request));
    }

    @PatchMapping("/deals/{id}/stage")
    public ResponseEntity<DealDTO> updateDealStage(
            @PathVariable Long id,
            @RequestParam DealStage stage) {
        return ResponseEntity.ok(crmService.updateDealStage(id, stage));
    }

    @GetMapping("/pipeline")
    public ResponseEntity<PipelineSummary> getPipeline(
            @RequestParam(required = false) Long ownerId) {
        return ResponseEntity.ok(crmService.getPipelineSummary(ownerId));
    }

    // Activities
    @PostMapping("/activities")
    public ResponseEntity<ActivityDTO> logActivity(@Valid @RequestBody LogActivityRequest request) {
        return ResponseEntity.status(HttpStatus.CREATED).body(crmService.logActivity(request));
    }

    @GetMapping("/contacts/{contactId}/activities")
    public ResponseEntity<Page<ActivityDTO>> getContactActivities(
            @PathVariable Long contactId,
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size) {
        return ResponseEntity.ok(crmService.getContactActivities(contactId, PageRequest.of(page, size)));
    }
}
```

---

## โปรเจคที่ 8: Hospital Management System

### ภาพรวมระบบ

ระบบจัดการโรงพยาบาลที่รองรับข้อมูลผู้ป่วย แพทย์ การนัดหมาย เวชระเบียน ใบสั่งยา การเรียกเก็บเงิน และการจัดการหอผู้ป่วย

### Entity Classes

```java
// Patient.java
@Entity
@Table(name = "patients")
@Data
@NoArgsConstructor
public class Patient {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(unique = true)
    private String patientId; // HN number

    private String firstName;
    private String lastName;
    private LocalDate dateOfBirth;
    private String gender;
    private String bloodType;
    private String nationalId;
    private String phone;
    private String email;
    private String address;
    private String emergencyContactName;
    private String emergencyContactPhone;

    @Column(columnDefinition = "TEXT")
    private String allergies;

    @Column(columnDefinition = "TEXT")
    private String chronicConditions;

    @OneToMany(mappedBy = "patient")
    private List<Appointment> appointments = new ArrayList<>();

    @OneToMany(mappedBy = "patient")
    private List<MedicalRecord> medicalRecords = new ArrayList<>();

    @CreatedDate
    private LocalDateTime createdAt;
}

// Doctor.java
@Entity
@Table(name = "doctors")
@Data
@NoArgsConstructor
public class Doctor {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(unique = true)
    private String doctorCode;

    private String firstName;
    private String lastName;
    private String specialization;
    private String licenseNumber;
    private String phone;
    private String email;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "department_id")
    private Department department;

    @Column(precision = 10, scale = 2)
    private BigDecimal consultationFee;

    private Boolean available = true;

    @OneToMany(mappedBy = "doctor")
    private List<Appointment> appointments = new ArrayList<>();
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
    private String appointmentNumber;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "patient_id")
    private Patient patient;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "doctor_id")
    private Doctor doctor;

    private LocalDateTime appointmentDateTime;

    @Enumerated(EnumType.STRING)
    private AppointmentStatus status = AppointmentStatus.SCHEDULED;

    private String type; // OPD, IPD, EMERGENCY
    private String chiefComplaint;
    private String notes;

    @OneToOne(mappedBy = "appointment", cascade = CascadeType.ALL)
    private MedicalRecord medicalRecord;

    @CreatedDate
    private LocalDateTime createdAt;
}

// MedicalRecord.java
@Entity
@Table(name = "medical_records")
@Data
@NoArgsConstructor
public class MedicalRecord {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "patient_id")
    private Patient patient;

    @OneToOne
    @JoinColumn(name = "appointment_id")
    private Appointment appointment;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "doctor_id")
    private Doctor doctor;

    @Column(columnDefinition = "TEXT")
    private String diagnosis;

    @Column(columnDefinition = "TEXT")
    private String symptoms;

    @Column(columnDefinition = "TEXT")
    private String treatment;

    @Column(columnDefinition = "TEXT")
    private String notes;

    private String vitalSigns; // JSON string: BP, HR, Temp, SpO2

    @OneToMany(mappedBy = "medicalRecord", cascade = CascadeType.ALL)
    private List<Prescription> prescriptions = new ArrayList<>();

    @CreatedDate
    private LocalDateTime createdAt;
}

// Prescription.java
@Entity
@Table(name = "prescriptions")
@Data
@NoArgsConstructor
public class Prescription {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne
    @JoinColumn(name = "medical_record_id")
    private MedicalRecord medicalRecord;

    private String medicineName;
    private String dosage;
    private String frequency;
    private Integer duration; // Days
    private String instructions;

    @Column(precision = 10, scale = 2)
    private BigDecimal price;
}

public enum AppointmentStatus {
    SCHEDULED, CONFIRMED, CHECKED_IN, IN_PROGRESS, COMPLETED, CANCELLED, NO_SHOW
}
```

### Service Layer

```java
// HospitalService.java
@Service
@Transactional
@RequiredArgsConstructor
public class HospitalService {

    private final PatientRepository patientRepository;
    private final DoctorRepository doctorRepository;
    private final AppointmentRepository appointmentRepository;
    private final MedicalRecordRepository medicalRecordRepository;

    public AppointmentDTO bookAppointment(BookAppointmentRequest request) {
        Patient patient = patientRepository.findById(request.getPatientId())
            .orElseThrow(() -> new ResourceNotFoundException("Patient not found"));

        Doctor doctor = doctorRepository.findById(request.getDoctorId())
            .orElseThrow(() -> new ResourceNotFoundException("Doctor not found"));

        // Check doctor availability
        boolean hasConflict = appointmentRepository.hasConflict(
            request.getDoctorId(),
            request.getAppointmentDateTime(),
            request.getAppointmentDateTime().plusMinutes(30));

        if (hasConflict) {
            throw new ConflictException("Doctor is not available at this time");
        }

        Appointment appointment = new Appointment();
        appointment.setAppointmentNumber("APT-" + System.currentTimeMillis());
        appointment.setPatient(patient);
        appointment.setDoctor(doctor);
        appointment.setAppointmentDateTime(request.getAppointmentDateTime());
        appointment.setType(request.getType());
        appointment.setChiefComplaint(request.getChiefComplaint());

        return toAppointmentDTO(appointmentRepository.save(appointment));
    }

    public MedicalRecordDTO createMedicalRecord(Long appointmentId, CreateMedicalRecordRequest request) {
        Appointment appointment = appointmentRepository.findById(appointmentId)
            .orElseThrow(() -> new ResourceNotFoundException("Appointment not found"));

        if (appointment.getStatus() != AppointmentStatus.IN_PROGRESS &&
            appointment.getStatus() != AppointmentStatus.CHECKED_IN) {
            throw new BadRequestException("Appointment must be in progress to create medical record");
        }

        MedicalRecord record = new MedicalRecord();
        record.setPatient(appointment.getPatient());
        record.setAppointment(appointment);
        record.setDoctor(appointment.getDoctor());
        record.setDiagnosis(request.getDiagnosis());
        record.setSymptoms(request.getSymptoms());
        record.setTreatment(request.getTreatment());
        record.setNotes(request.getNotes());
        record.setVitalSigns(request.getVitalSigns());

        // Add prescriptions
        for (PrescriptionRequest presReq : request.getPrescriptions()) {
            Prescription pres = new Prescription();
            pres.setMedicalRecord(record);
            pres.setMedicineName(presReq.getMedicineName());
            pres.setDosage(presReq.getDosage());
            pres.setFrequency(presReq.getFrequency());
            pres.setDuration(presReq.getDuration());
            pres.setInstructions(presReq.getInstructions());
            pres.setPrice(presReq.getPrice());
            record.getPrescriptions().add(pres);
        }

        appointment.setStatus(AppointmentStatus.COMPLETED);
        appointmentRepository.save(appointment);

        return toMedicalRecordDTO(medicalRecordRepository.save(record));
    }

    public BillingDTO generateBill(Long appointmentId) {
        Appointment appointment = appointmentRepository.findById(appointmentId)
            .orElseThrow(() -> new ResourceNotFoundException("Appointment not found"));

        MedicalRecord record = appointment.getMedicalRecord();
        BigDecimal consultationFee = appointment.getDoctor().getConsultationFee();

        BigDecimal medicineCost = BigDecimal.ZERO;
        if (record != null) {
            medicineCost = record.getPrescriptions().stream()
                .filter(p -> p.getPrice() != null)
                .map(Prescription::getPrice)
                .reduce(BigDecimal.ZERO, BigDecimal::add);
        }

        BigDecimal totalAmount = consultationFee.add(medicineCost);

        return BillingDTO.builder()
            .appointmentNumber(appointment.getAppointmentNumber())
            .patientName(appointment.getPatient().getFirstName() + " " + appointment.getPatient().getLastName())
            .doctorName(appointment.getDoctor().getFirstName() + " " + appointment.getDoctor().getLastName())
            .consultationFee(consultationFee)
            .medicineCost(medicineCost)
            .totalAmount(totalAmount)
            .build();
    }

    private AppointmentDTO toAppointmentDTO(Appointment apt) {
        return AppointmentDTO.builder()
            .id(apt.getId())
            .appointmentNumber(apt.getAppointmentNumber())
            .patientName(apt.getPatient().getFirstName() + " " + apt.getPatient().getLastName())
            .doctorName(apt.getDoctor().getFirstName() + " " + apt.getDoctor().getLastName())
            .appointmentDateTime(apt.getAppointmentDateTime())
            .status(apt.getStatus())
            .type(apt.getType())
            .build();
    }

    private MedicalRecordDTO toMedicalRecordDTO(MedicalRecord record) {
        return MedicalRecordDTO.builder()
            .id(record.getId())
            .diagnosis(record.getDiagnosis())
            .symptoms(record.getSymptoms())
            .treatment(record.getTreatment())
            .prescriptions(record.getPrescriptions().stream()
                .map(p -> PrescriptionDTO.builder()
                    .medicineName(p.getMedicineName())
                    .dosage(p.getDosage())
                    .frequency(p.getFrequency())
                    .duration(p.getDuration())
                    .build())
                .collect(Collectors.toList()))
            .createdAt(record.getCreatedAt())
            .build();
    }
}
```

---

## โปรเจคที่ 9: Hotel Booking System

### ภาพรวมระบบ

ระบบจองโรงแรมที่รองรับประเภทห้องพัก ราคา สถานะห้อง การจอง (check-in/check-out) การจัดการแขก การขอบริการในห้อง และการออกใบเสร็จ

### Entity Classes

```java
// RoomType.java
@Entity
@Table(name = "room_types")
@Data
@NoArgsConstructor
public class RoomType {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name; // STANDARD, DELUXE, SUITE, PENTHOUSE
    private String description;
    private Integer capacity;
    private Integer bedCount;
    private String bedType;

    @Column(precision = 10, scale = 2)
    private BigDecimal basePrice;

    @Column(precision = 10, scale = 2)
    private BigDecimal weekendPrice;

    @ElementCollection
    @CollectionTable(name = "room_type_amenities")
    private List<String> amenities = new ArrayList<>();
}

// Room.java
@Entity
@Table(name = "rooms")
@Data
@NoArgsConstructor
public class Room {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(unique = true, nullable = false)
    private String roomNumber;

    private Integer floor;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "room_type_id")
    private RoomType roomType;

    @Enumerated(EnumType.STRING)
    private RoomStatus status = RoomStatus.AVAILABLE;

    private String notes;
    private LocalDate lastCleanedDate;
}

// Reservation.java
@Entity
@Table(name = "reservations")
@Data
@NoArgsConstructor
public class Reservation {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(unique = true, nullable = false)
    private String confirmationNumber;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "guest_id")
    private Guest guest;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "room_id")
    private Room room;

    private LocalDate checkInDate;
    private LocalDate checkOutDate;

    private LocalDateTime actualCheckIn;
    private LocalDateTime actualCheckOut;

    private Integer numberOfGuests;

    @Enumerated(EnumType.STRING)
    private ReservationStatus status = ReservationStatus.CONFIRMED;

    @Column(precision = 10, scale = 2)
    private BigDecimal totalAmount;

    @Column(precision = 10, scale = 2)
    private BigDecimal depositAmount;

    private String specialRequests;

    @OneToMany(mappedBy = "reservation", cascade = CascadeType.ALL)
    private List<RoomServiceRequest> serviceRequests = new ArrayList<>();

    @CreatedDate
    private LocalDateTime createdAt;
}

// Guest.java
@Entity
@Table(name = "guests")
@Data
@NoArgsConstructor
public class Guest {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String firstName;
    private String lastName;
    private String email;
    private String phone;
    private String nationality;
    private String passportNumber;
    private String address;

    @Enumerated(EnumType.STRING)
    private GuestType type = GuestType.REGULAR; // REGULAR, VIP, BLACKLISTED

    private Integer totalStays = 0;
    private String notes;
}

// RoomServiceRequest.java
@Entity
@Table(name = "room_service_requests")
@Data
@NoArgsConstructor
public class RoomServiceRequest {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "reservation_id")
    private Reservation reservation;

    private String serviceType; // FOOD, HOUSEKEEPING, MAINTENANCE, AMENITY
    private String description;

    @Column(precision = 10, scale = 2)
    private BigDecimal cost;

    @Enumerated(EnumType.STRING)
    private ServiceStatus status = ServiceStatus.PENDING;

    @CreatedDate
    private LocalDateTime requestedAt;

    private LocalDateTime completedAt;
}

public enum RoomStatus { AVAILABLE, OCCUPIED, CLEANING, MAINTENANCE, OUT_OF_ORDER }
public enum ReservationStatus { PENDING, CONFIRMED, CHECKED_IN, CHECKED_OUT, CANCELLED, NO_SHOW }
public enum GuestType { REGULAR, VIP, BLACKLISTED }
public enum ServiceStatus { PENDING, IN_PROGRESS, COMPLETED, CANCELLED }
```

### Service Layer

```java
// HotelService.java
@Service
@Transactional
@RequiredArgsConstructor
public class HotelService {

    private final RoomRepository roomRepository;
    private final ReservationRepository reservationRepository;
    private final GuestRepository guestRepository;
    private final RoomServiceRequestRepository serviceRequestRepository;

    public List<RoomDTO> searchAvailableRooms(LocalDate checkIn, LocalDate checkOut,
                                               String roomTypeName, Integer guests) {
        return roomRepository.findAvailableRooms(checkIn, checkOut, roomTypeName, guests)
            .stream().map(this::toRoomDTO).collect(Collectors.toList());
    }

    public ReservationDTO createReservation(CreateReservationRequest request) {
        Guest guest = guestRepository.findById(request.getGuestId())
            .orElseThrow(() -> new ResourceNotFoundException("Guest not found"));

        Room room = roomRepository.findById(request.getRoomId())
            .orElseThrow(() -> new ResourceNotFoundException("Room not found"));

        // Check availability
        boolean isAvailable = !reservationRepository.hasConflict(
            request.getRoomId(), request.getCheckInDate(), request.getCheckOutDate());

        if (!isAvailable) {
            throw new ConflictException("Room is not available for selected dates");
        }

        // Calculate total
        long nights = ChronoUnit.DAYS.between(request.getCheckInDate(), request.getCheckOutDate());
        BigDecimal totalAmount = calculateRoomRate(room, request.getCheckInDate(), request.getCheckOutDate())
            .multiply(BigDecimal.valueOf(nights));

        Reservation reservation = new Reservation();
        reservation.setConfirmationNumber("HTL-" + System.currentTimeMillis());
        reservation.setGuest(guest);
        reservation.setRoom(room);
        reservation.setCheckInDate(request.getCheckInDate());
        reservation.setCheckOutDate(request.getCheckOutDate());
        reservation.setNumberOfGuests(request.getNumberOfGuests());
        reservation.setTotalAmount(totalAmount);
        reservation.setDepositAmount(totalAmount.multiply(new BigDecimal("0.3"))); // 30% deposit
        reservation.setSpecialRequests(request.getSpecialRequests());

        return toReservationDTO(reservationRepository.save(reservation));
    }

    public ReservationDTO checkIn(Long reservationId) {
        Reservation reservation = reservationRepository.findById(reservationId)
            .orElseThrow(() -> new ResourceNotFoundException("Reservation not found"));

        if (reservation.getStatus() != ReservationStatus.CONFIRMED) {
            throw new BadRequestException("Reservation must be confirmed before check-in");
        }

        reservation.setStatus(ReservationStatus.CHECKED_IN);
        reservation.setActualCheckIn(LocalDateTime.now());

        Room room = reservation.getRoom();
        room.setStatus(RoomStatus.OCCUPIED);
        roomRepository.save(room);

        // Update guest total stays
        Guest guest = reservation.getGuest();
        guest.setTotalStays(guest.getTotalStays() + 1);
        guestRepository.save(guest);

        return toReservationDTO(reservationRepository.save(reservation));
    }

    public FolioBill checkOut(Long reservationId) {
        Reservation reservation = reservationRepository.findById(reservationId)
            .orElseThrow(() -> new ResourceNotFoundException("Reservation not found"));

        if (reservation.getStatus() != ReservationStatus.CHECKED_IN) {
            throw new BadRequestException("Guest is not checked in");
        }

        reservation.setStatus(ReservationStatus.CHECKED_OUT);
        reservation.setActualCheckOut(LocalDateTime.now());

        Room room = reservation.getRoom();
        room.setStatus(RoomStatus.CLEANING);
        roomRepository.save(room);

        reservationRepository.save(reservation);

        // Generate folio/bill
        return generateFolio(reservation);
    }

    private FolioBill generateFolio(Reservation reservation) {
        BigDecimal serviceTotal = reservation.getServiceRequests().stream()
            .filter(s -> s.getStatus() == ServiceStatus.COMPLETED && s.getCost() != null)
            .map(RoomServiceRequest::getCost)
            .reduce(BigDecimal.ZERO, BigDecimal::add);

        return FolioBill.builder()
            .confirmationNumber(reservation.getConfirmationNumber())
            .guestName(reservation.getGuest().getFirstName() + " " + reservation.getGuest().getLastName())
            .roomNumber(reservation.getRoom().getRoomNumber())
            .checkInDate(reservation.getCheckInDate())
            .checkOutDate(reservation.getCheckOutDate())
            .roomCharges(reservation.getTotalAmount())
            .serviceCharges(serviceTotal)
            .totalDue(reservation.getTotalAmount().add(serviceTotal).subtract(reservation.getDepositAmount()))
            .build();
    }

    private BigDecimal calculateRoomRate(Room room, LocalDate checkIn, LocalDate checkOut) {
        RoomType roomType = room.getRoomType();
        LocalDate current = checkIn;
        BigDecimal total = BigDecimal.ZERO;

        while (current.isBefore(checkOut)) {
            boolean isWeekend = current.getDayOfWeek() == DayOfWeek.SATURDAY ||
                               current.getDayOfWeek() == DayOfWeek.SUNDAY;
            total = total.add(isWeekend ? roomType.getWeekendPrice() : roomType.getBasePrice());
            current = current.plusDays(1);
        }

        long nights = ChronoUnit.DAYS.between(checkIn, checkOut);
        return nights > 0 ? total.divide(BigDecimal.valueOf(nights), 2, RoundingMode.HALF_UP) : total;
    }

    private ReservationDTO toReservationDTO(Reservation r) {
        return ReservationDTO.builder()
            .id(r.getId())
            .confirmationNumber(r.getConfirmationNumber())
            .guestName(r.getGuest().getFirstName() + " " + r.getGuest().getLastName())
            .roomNumber(r.getRoom().getRoomNumber())
            .checkInDate(r.getCheckInDate())
            .checkOutDate(r.getCheckOutDate())
            .status(r.getStatus())
            .totalAmount(r.getTotalAmount())
            .build();
    }

    private RoomDTO toRoomDTO(Room room) {
        return RoomDTO.builder()
            .id(room.getId())
            .roomNumber(room.getRoomNumber())
            .floor(room.getFloor())
            .typeName(room.getRoomType().getName())
            .basePrice(room.getRoomType().getBasePrice())
            .status(room.getStatus())
            .build();
    }
}
```

---

## โปรเจคที่ 10: Restaurant POS System

### ภาพรวมระบบ

ระบบ POS สำหรับร้านอาหารที่รองรับเมนูอาหาร โต๊ะ การรับออเดอร์ คิวครัว การแบ่งจ่ายบิล รายงานยอดขายรายวัน และการตัดสต็อกวัตถุดิบ

### Entity Classes

```java
// MenuItem.java
@Entity
@Table(name = "menu_items")
@Data
@NoArgsConstructor
public class MenuItem {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String name;

    private String description;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "category_id")
    private MenuCategory category;

    @Column(precision = 8, scale = 2)
    private BigDecimal price;

    @Column(precision = 8, scale = 2)
    private BigDecimal cost;

    private String imageUrl;
    private Boolean available = true;
    private Integer preparationTimeMinutes;

    @ElementCollection
    @CollectionTable(name = "menu_item_tags")
    private List<String> tags = new ArrayList<>(); // SPICY, VEG, POPULAR, etc.
}

// RestaurantTable.java
@Entity
@Table(name = "restaurant_tables")
@Data
@NoArgsConstructor
public class RestaurantTable {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(unique = true)
    private String tableNumber;

    private Integer capacity;
    private String location; // INDOOR, OUTDOOR, PRIVATE_ROOM

    @Enumerated(EnumType.STRING)
    private TableStatus status = TableStatus.AVAILABLE;

    private String currentOrderId;
}

// RestaurantOrder.java
@Entity
@Table(name = "restaurant_orders")
@Data
@NoArgsConstructor
public class RestaurantOrder {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(unique = true)
    private String orderNumber;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "table_id")
    private RestaurantTable table;

    private Integer numberOfGuests;

    @Enumerated(EnumType.STRING)
    private OrderStatus status = OrderStatus.OPEN;

    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL)
    private List<RestaurantOrderItem> items = new ArrayList<>();

    @Column(precision = 10, scale = 2)
    private BigDecimal subtotal;

    @Column(precision = 10, scale = 2)
    private BigDecimal taxAmount;

    @Column(precision = 5, scale = 2)
    private BigDecimal taxRate = new BigDecimal("7.00");

    @Column(precision = 10, scale = 2)
    private BigDecimal serviceCharge;

    @Column(precision = 5, scale = 2)
    private BigDecimal serviceChargeRate = new BigDecimal("10.00");

    @Column(precision = 10, scale = 2)
    private BigDecimal totalAmount;

    @Column(precision = 10, scale = 2)
    private BigDecimal discountAmount = BigDecimal.ZERO;

    @Enumerated(EnumType.STRING)
    private PaymentMethod paymentMethod;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "cashier_id")
    private User cashier;

    @CreatedDate
    private LocalDateTime createdAt;

    private LocalDateTime closedAt;
}

// RestaurantOrderItem.java
@Entity
@Table(name = "restaurant_order_items")
@Data
@NoArgsConstructor
public class RestaurantOrderItem {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne
    @JoinColumn(name = "order_id")
    private RestaurantOrder order;

    @ManyToOne
    @JoinColumn(name = "menu_item_id")
    private MenuItem menuItem;

    private Integer quantity;

    @Column(precision = 8, scale = 2)
    private BigDecimal unitPrice;

    @Column(precision = 10, scale = 2)
    private BigDecimal subtotal;

    private String specialInstructions;

    @Enumerated(EnumType.STRING)
    private KitchenStatus kitchenStatus = KitchenStatus.PENDING;

    private LocalDateTime sentToKitchenAt;
    private LocalDateTime readyAt;
    private LocalDateTime servedAt;
}

public enum TableStatus { AVAILABLE, OCCUPIED, RESERVED, CLEANING }
public enum OrderStatus { OPEN, BILLED, PAID, CANCELLED }
public enum KitchenStatus { PENDING, PREPARING, READY, SERVED }
public enum PaymentMethod { CASH, CREDIT_CARD, QR_CODE, WALLET }
```

### Service Layer

```java
// RestaurantService.java
@Service
@Transactional
@RequiredArgsConstructor
public class RestaurantService {

    private final RestaurantOrderRepository orderRepository;
    private final RestaurantTableRepository tableRepository;
    private final MenuItemRepository menuItemRepository;

    public RestaurantOrderDTO openTable(Long tableId, Integer guests) {
        RestaurantTable table = tableRepository.findById(tableId)
            .orElseThrow(() -> new ResourceNotFoundException("Table not found"));

        if (table.getStatus() != TableStatus.AVAILABLE) {
            throw new ConflictException("Table is not available");
        }

        RestaurantOrder order = new RestaurantOrder();
        order.setOrderNumber("ORD-" + LocalDate.now().format(DateTimeFormatter.BASIC_ISO_DATE) + "-" +
                            String.format("%04d", (int)(Math.random() * 10000)));
        order.setTable(table);
        order.setNumberOfGuests(guests);

        RestaurantOrder saved = orderRepository.save(order);

        table.setStatus(TableStatus.OCCUPIED);
        table.setCurrentOrderId(order.getOrderNumber());
        tableRepository.save(table);

        return toOrderDTO(saved);
    }

    public RestaurantOrderDTO addItems(Long orderId, List<AddMenuItemRequest> items) {
        RestaurantOrder order = orderRepository.findById(orderId)
            .orElseThrow(() -> new ResourceNotFoundException("Order not found"));

        if (order.getStatus() != OrderStatus.OPEN) {
            throw new BadRequestException("Order is not open");
        }

        for (AddMenuItemRequest itemReq : items) {
            MenuItem menuItem = menuItemRepository.findById(itemReq.getMenuItemId())
                .orElseThrow(() -> new ResourceNotFoundException("Menu item not found: " + itemReq.getMenuItemId()));

            if (!menuItem.getAvailable()) {
                throw new BadRequestException("Menu item is not available: " + menuItem.getName());
            }

            RestaurantOrderItem orderItem = new RestaurantOrderItem();
            orderItem.setOrder(order);
            orderItem.setMenuItem(menuItem);
            orderItem.setQuantity(itemReq.getQuantity());
            orderItem.setUnitPrice(menuItem.getPrice());
            orderItem.setSubtotal(menuItem.getPrice().multiply(BigDecimal.valueOf(itemReq.getQuantity())));
            orderItem.setSpecialInstructions(itemReq.getSpecialInstructions());
            orderItem.setSentToKitchenAt(LocalDateTime.now());
            order.getItems().add(orderItem);
        }

        calculateOrderTotals(order);
        return toOrderDTO(orderRepository.save(order));
    }

    public RestaurantOrderDTO updateKitchenStatus(Long orderId, Long itemId, KitchenStatus status) {
        RestaurantOrder order = orderRepository.findById(orderId)
            .orElseThrow(() -> new ResourceNotFoundException("Order not found"));

        RestaurantOrderItem item = order.getItems().stream()
            .filter(i -> i.getId().equals(itemId))
            .findFirst()
            .orElseThrow(() -> new ResourceNotFoundException("Order item not found"));

        item.setKitchenStatus(status);
        if (status == KitchenStatus.READY) {
            item.setReadyAt(LocalDateTime.now());
        } else if (status == KitchenStatus.SERVED) {
            item.setServedAt(LocalDateTime.now());
        }

        return toOrderDTO(orderRepository.save(order));
    }

    public BillDTO generateBill(Long orderId) {
        RestaurantOrder order = orderRepository.findById(orderId)
            .orElseThrow(() -> new ResourceNotFoundException("Order not found"));

        if (order.getStatus() != OrderStatus.OPEN) {
            throw new BadRequestException("Order cannot be billed");
        }

        calculateOrderTotals(order);
        order.setStatus(OrderStatus.BILLED);
        orderRepository.save(order);

        return BillDTO.builder()
            .orderNumber(order.getOrderNumber())
            .tableNumber(order.getTable().getTableNumber())
            .subtotal(order.getSubtotal())
            .serviceCharge(order.getServiceCharge())
            .taxAmount(order.getTaxAmount())
            .discountAmount(order.getDiscountAmount())
            .totalAmount(order.getTotalAmount())
            .items(order.getItems().stream().map(this::toBillItemDTO).collect(Collectors.toList()))
            .build();
    }

    public RestaurantOrderDTO processPayment(Long orderId, PaymentMethod method, BigDecimal amountPaid) {
        RestaurantOrder order = orderRepository.findById(orderId)
            .orElseThrow(() -> new ResourceNotFoundException("Order not found"));

        if (order.getStatus() != OrderStatus.BILLED) {
            throw new BadRequestException("Order must be billed before payment");
        }

        if (amountPaid.compareTo(order.getTotalAmount()) < 0) {
            throw new BadRequestException("Insufficient payment amount");
        }

        order.setStatus(OrderStatus.PAID);
        order.setPaymentMethod(method);
        order.setClosedAt(LocalDateTime.now());

        // Free up the table
        RestaurantTable table = order.getTable();
        table.setStatus(TableStatus.CLEANING);
        table.setCurrentOrderId(null);
        tableRepository.save(table);

        return toOrderDTO(orderRepository.save(order));
    }

    public DailySalesReport getDailySalesReport(LocalDate date) {
        LocalDateTime start = date.atStartOfDay();
        LocalDateTime end = date.plusDays(1).atStartOfDay();

        List<RestaurantOrder> paidOrders = orderRepository.findPaidOrdersBetween(start, end);

        BigDecimal totalRevenue = paidOrders.stream()
            .map(RestaurantOrder::getTotalAmount)
            .reduce(BigDecimal.ZERO, BigDecimal::add);

        Map<String, Long> itemSalesCount = paidOrders.stream()
            .flatMap(o -> o.getItems().stream())
            .collect(Collectors.groupingBy(
                i -> i.getMenuItem().getName(),
                Collectors.summingLong(i -> i.getQuantity().longValue())
            ));

        String topSellingItem = itemSalesCount.entrySet().stream()
            .max(Map.Entry.comparingByValue())
            .map(Map.Entry::getKey)
            .orElse("N/A");

        return DailySalesReport.builder()
            .date(date)
            .totalOrders(paidOrders.size())
            .totalRevenue(totalRevenue)
            .averageOrderValue(paidOrders.isEmpty() ? BigDecimal.ZERO :
                totalRevenue.divide(BigDecimal.valueOf(paidOrders.size()), 2, RoundingMode.HALF_UP))
            .topSellingItem(topSellingItem)
            .itemSalesCount(itemSalesCount)
            .build();
    }

    private void calculateOrderTotals(RestaurantOrder order) {
        BigDecimal subtotal = order.getItems().stream()
            .map(RestaurantOrderItem::getSubtotal)
            .reduce(BigDecimal.ZERO, BigDecimal::add);

        BigDecimal serviceCharge = subtotal.multiply(order.getServiceChargeRate())
            .divide(new BigDecimal("100"), 2, RoundingMode.HALF_UP);

        BigDecimal taxBase = subtotal.add(serviceCharge);
        BigDecimal tax = taxBase.multiply(order.getTaxRate())
            .divide(new BigDecimal("100"), 2, RoundingMode.HALF_UP);

        order.setSubtotal(subtotal);
        order.setServiceCharge(serviceCharge);
        order.setTaxAmount(tax);
        order.setTotalAmount(subtotal.add(serviceCharge).add(tax).subtract(order.getDiscountAmount()));
    }

    private RestaurantOrderDTO toOrderDTO(RestaurantOrder order) {
        return RestaurantOrderDTO.builder()
            .id(order.getId())
            .orderNumber(order.getOrderNumber())
            .tableNumber(order.getTable().getTableNumber())
            .status(order.getStatus())
            .subtotal(order.getSubtotal())
            .totalAmount(order.getTotalAmount())
            .items(order.getItems().stream().map(this::toOrderItemDTO).collect(Collectors.toList()))
            .build();
    }

    private OrderItemDTO toOrderItemDTO(RestaurantOrderItem item) {
        return OrderItemDTO.builder()
            .menuItemName(item.getMenuItem().getName())
            .quantity(item.getQuantity())
            .unitPrice(item.getUnitPrice())
            .subtotal(item.getSubtotal())
            .kitchenStatus(item.getKitchenStatus())
            .build();
    }

    private BillItemDTO toBillItemDTO(RestaurantOrderItem item) {
        return BillItemDTO.builder()
            .name(item.getMenuItem().getName())
            .quantity(item.getQuantity())
            .unitPrice(item.getUnitPrice())
            .subtotal(item.getSubtotal())
            .build();
    }
}
```

### REST Controller (Restaurant POS)

```java
// RestaurantController.java
@RestController
@RequestMapping("/api/v1/restaurant")
@RequiredArgsConstructor
public class RestaurantController {

    private final RestaurantService restaurantService;

    @GetMapping("/tables")
    public ResponseEntity<List<TableDTO>> getTables() {
        return ResponseEntity.ok(restaurantService.getAllTables());
    }

    @PostMapping("/orders/open-table")
    public ResponseEntity<RestaurantOrderDTO> openTable(
            @RequestParam Long tableId,
            @RequestParam(defaultValue = "1") Integer guests) {
        return ResponseEntity.ok(restaurantService.openTable(tableId, guests));
    }

    @PostMapping("/orders/{id}/items")
    public ResponseEntity<RestaurantOrderDTO> addItems(
            @PathVariable Long id,
            @Valid @RequestBody List<AddMenuItemRequest> items) {
        return ResponseEntity.ok(restaurantService.addItems(id, items));
    }

    @GetMapping("/kitchen/queue")
    public ResponseEntity<List<KitchenOrderDTO>> getKitchenQueue() {
        return ResponseEntity.ok(restaurantService.getKitchenQueue());
    }

    @PatchMapping("/kitchen/orders/{orderId}/items/{itemId}")
    public ResponseEntity<RestaurantOrderDTO> updateKitchenStatus(
            @PathVariable Long orderId,
            @PathVariable Long itemId,
            @RequestParam KitchenStatus status) {
        return ResponseEntity.ok(restaurantService.updateKitchenStatus(orderId, itemId, status));
    }

    @PostMapping("/orders/{id}/bill")
    public ResponseEntity<BillDTO> generateBill(@PathVariable Long id) {
        return ResponseEntity.ok(restaurantService.generateBill(id));
    }

    @PostMapping("/orders/{id}/pay")
    public ResponseEntity<RestaurantOrderDTO> processPayment(
            @PathVariable Long id,
            @RequestParam PaymentMethod method,
            @RequestParam BigDecimal amountPaid) {
        return ResponseEntity.ok(restaurantService.processPayment(id, method, amountPaid));
    }

    @GetMapping("/reports/daily")
    @PreAuthorize("hasRole('MANAGER')")
    public ResponseEntity<DailySalesReport> getDailyReport(
            @RequestParam(defaultValue = "#{T(java.time.LocalDate).now()}") LocalDate date) {
        return ResponseEntity.ok(restaurantService.getDailySalesReport(date));
    }
}
```

### Flyway Migration

```sql
-- V10__init_restaurant.sql
CREATE TABLE menu_categories (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    display_order INT DEFAULT 0
);

CREATE TABLE menu_items (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    category_id BIGINT REFERENCES menu_categories(id),
    price DECIMAL(8,2) NOT NULL,
    cost DECIMAL(8,2),
    image_url VARCHAR(500),
    available BOOLEAN DEFAULT TRUE,
    preparation_time_minutes INT DEFAULT 15
);

CREATE TABLE restaurant_tables (
    id BIGSERIAL PRIMARY KEY,
    table_number VARCHAR(10) NOT NULL UNIQUE,
    capacity INT NOT NULL,
    location VARCHAR(50),
    status VARCHAR(20) DEFAULT 'AVAILABLE',
    current_order_id VARCHAR(50)
);

CREATE TABLE restaurant_orders (
    id BIGSERIAL PRIMARY KEY,
    order_number VARCHAR(50) NOT NULL UNIQUE,
    table_id BIGINT NOT NULL REFERENCES restaurant_tables(id),
    number_of_guests INT DEFAULT 1,
    status VARCHAR(20) DEFAULT 'OPEN',
    subtotal DECIMAL(10,2),
    tax_amount DECIMAL(10,2),
    tax_rate DECIMAL(5,2) DEFAULT 7.00,
    service_charge DECIMAL(10,2),
    service_charge_rate DECIMAL(5,2) DEFAULT 10.00,
    total_amount DECIMAL(10,2),
    discount_amount DECIMAL(10,2) DEFAULT 0,
    payment_method VARCHAR(20),
    cashier_id BIGINT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    closed_at TIMESTAMP
);

CREATE TABLE restaurant_order_items (
    id BIGSERIAL PRIMARY KEY,
    order_id BIGINT NOT NULL REFERENCES restaurant_orders(id),
    menu_item_id BIGINT NOT NULL REFERENCES menu_items(id),
    quantity INT NOT NULL,
    unit_price DECIMAL(8,2) NOT NULL,
    subtotal DECIMAL(10,2) NOT NULL,
    special_instructions TEXT,
    kitchen_status VARCHAR(20) DEFAULT 'PENDING',
    sent_to_kitchen_at TIMESTAMP,
    ready_at TIMESTAMP,
    served_at TIMESTAMP
);
```

### ตัวอย่าง API Calls

```bash
# เปิดโต๊ะ
curl -X POST "http://localhost:8080/api/v1/restaurant/orders/open-table?tableId=5&guests=3" \
  -H "Authorization: Bearer {token}"

# เพิ่มเมนู
curl -X POST http://localhost:8080/api/v1/restaurant/orders/1/items \
  -H "Authorization: Bearer {token}" \
  -H "Content-Type: application/json" \
  -d '[
    {"menuItemId": 1, "quantity": 2, "specialInstructions": "ไม่เผ็ด"},
    {"menuItemId": 5, "quantity": 1}
  ]'

# ดูคิวครัว
curl http://localhost:8080/api/v1/restaurant/kitchen/queue \
  -H "Authorization: Bearer {token}"

# อัพเดทสถานะอาหาร
curl -X PATCH "http://localhost:8080/api/v1/restaurant/kitchen/orders/1/items/3?status=READY" \
  -H "Authorization: Bearer {token}"

# ออกบิล
curl -X POST http://localhost:8080/api/v1/restaurant/orders/1/bill \
  -H "Authorization: Bearer {token}"

# ชำระเงิน
curl -X POST "http://localhost:8080/api/v1/restaurant/orders/1/pay?method=QR_CODE&amountPaid=350" \
  -H "Authorization: Bearer {token}"

# รายงานยอดขายวันนี้
curl "http://localhost:8080/api/v1/restaurant/reports/daily?date=2024-01-15" \
  -H "Authorization: Bearer {admin_token}"
```

---

*[← Part 101: E-Commerce & FinTech](./part-101-ecommerce-projects.md) | [Part 103: Operations & Management →](./part-103-management-systems.md)*

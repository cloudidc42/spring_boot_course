# Part 118: โปรเจค 86-90 — Workflow & Automation

> **ระดับ:** โลก (World-Class) | **เวลาเรียนรู้:** 10-15 ชั่วโมง  
> **เป้าหมาย:** สร้างระบบ Workflow & Automation ที่ใช้งานได้จริงในระดับองค์กร

---

## โปรเจค 86: Workflow Automation Engine

### ภาพรวมโปรเจค

Workflow Automation Engine เป็นระบบสร้างและรัน Workflow อัตโนมัติ ผู้ใช้สามารถกำหนด Workflow ที่ประกอบด้วย Steps, Conditions และ Branches ได้ ระบบรองรับ Triggers หลายประเภท (Manual, Scheduled, Webhook), เก็บ Execution History, รองรับ Approval Steps และส่ง Notifications ผ่านช่องทางต่างๆ

### Entities

```java
// WorkflowDefinition.java — Blueprint ของ Workflow
@Entity
@Table(name = "workflow_definitions")
public class WorkflowDefinition {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String name;

    private String description;

    // JSON ที่เก็บ Step definitions, conditions, branches
    @Column(columnDefinition = "jsonb", nullable = false)
    private String steps;

    // Trigger configuration
    @Enumerated(EnumType.STRING)
    private TriggerType triggerType; // MANUAL, SCHEDULED, WEBHOOK, EVENT

    @Column(columnDefinition = "jsonb")
    private String triggerConfig; // {"cron": "0 9 * * MON"} หรือ {"webhookPath": "/hr/new-employee"}

    @Enumerated(EnumType.STRING)
    private WorkflowStatus status; // DRAFT, ACTIVE, DISABLED

    private Integer version;
    private Long createdBy;

    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
}

// WorkflowExecution.java — Instance ของการรัน Workflow
@Entity
@Table(name = "workflow_executions")
public class WorkflowExecution {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne
    @JoinColumn(name = "workflow_id")
    private WorkflowDefinition workflow;

    @Enumerated(EnumType.STRING)
    private ExecutionStatus status; // PENDING, RUNNING, WAITING_APPROVAL, COMPLETED, FAILED, CANCELLED

    // Input data ที่ trigger ส่งมา
    @Column(columnDefinition = "jsonb")
    private String inputData;

    // Context data ที่สะสมระหว่างรัน
    @Column(columnDefinition = "jsonb")
    private String contextData;

    private String currentStepId;
    private String errorMessage;

    private Long triggeredBy; // userId ที่ trigger หรือ null ถ้าเป็น scheduled

    private LocalDateTime startedAt;
    private LocalDateTime completedAt;
    private Long durationMs;
}

// WorkflowStepExecution.java — ผลการรัน Step แต่ละ Step
@Entity
@Table(name = "workflow_step_executions")
public class WorkflowStepExecution {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne
    @JoinColumn(name = "execution_id")
    private WorkflowExecution execution;

    @Column(nullable = false)
    private String stepId;

    private String stepName;
    private String stepType; // ACTION, CONDITION, APPROVAL, NOTIFICATION, DELAY

    @Enumerated(EnumType.STRING)
    private StepStatus status; // PENDING, RUNNING, COMPLETED, FAILED, SKIPPED

    @Column(columnDefinition = "jsonb")
    private String inputData;

    @Column(columnDefinition = "jsonb")
    private String outputData;

    private String errorMessage;

    private LocalDateTime startedAt;
    private LocalDateTime completedAt;
    private Long durationMs;
}

// ApprovalRequest.java — คำขออนุมัติใน Approval Step
@Entity
@Table(name = "approval_requests")
public class ApprovalRequest {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne
    @JoinColumn(name = "execution_id")
    private WorkflowExecution execution;

    private String stepId;

    @ManyToOne
    @JoinColumn(name = "approver_id")
    private User approver;

    @Enumerated(EnumType.STRING)
    private ApprovalStatus status; // PENDING, APPROVED, REJECTED

    private String comment;
    private LocalDateTime requestedAt;
    private LocalDateTime respondedAt;
    private LocalDateTime expiresAt;
}
```

### Workflow Engine

```java
// WorkflowEngine.java — Core engine ที่รัน Workflow
@Service
public class WorkflowEngine {

    @Autowired
    private WorkflowExecutionRepository executionRepository;

    @Autowired
    private WorkflowStepExecutionRepository stepExecutionRepository;

    @Autowired
    private StepExecutorFactory stepExecutorFactory;

    @Autowired
    private ObjectMapper objectMapper;

    // เริ่มรัน Workflow
    @Transactional
    public WorkflowExecution startExecution(WorkflowDefinition workflow, Map<String, Object> inputData, Long triggeredBy) {
        WorkflowExecution execution = new WorkflowExecution();
        execution.setWorkflow(workflow);
        execution.setStatus(ExecutionStatus.RUNNING);
        execution.setInputData(toJson(inputData));
        execution.setContextData(toJson(new HashMap<>(inputData)));
        execution.setTriggeredBy(triggeredBy);
        execution.setStartedAt(LocalDateTime.now());

        execution = executionRepository.save(execution);

        // รัน Workflow แบบ Async
        executeWorkflowAsync(execution.getId());

        return execution;
    }

    @Async
    public void executeWorkflowAsync(Long executionId) {
        WorkflowExecution execution = executionRepository.findById(executionId).orElseThrow();

        try {
            List<StepDefinition> steps = parseSteps(execution.getWorkflow().getSteps());
            Map<String, Object> context = parseJson(execution.getContextData());

            for (StepDefinition step : steps) {
                if (shouldSkipStep(step, context)) {
                    recordSkippedStep(execution, step);
                    continue;
                }

                StepResult result = executeStep(execution, step, context);

                if (result.isWaitingForApproval()) {
                    execution.setStatus(ExecutionStatus.WAITING_APPROVAL);
                    execution.setCurrentStepId(step.getId());
                    executionRepository.save(execution);
                    return; // หยุดรอ Approval
                }

                if (!result.isSuccess()) {
                    execution.setStatus(ExecutionStatus.FAILED);
                    execution.setErrorMessage(result.getErrorMessage());
                    execution.setCompletedAt(LocalDateTime.now());
                    executionRepository.save(execution);
                    return;
                }

                // Update context ด้วย output ของ step
                context.putAll(result.getOutputData());
            }

            // Workflow เสร็จสมบูรณ์
            execution.setStatus(ExecutionStatus.COMPLETED);
            execution.setCompletedAt(LocalDateTime.now());
            execution.setDurationMs(Duration.between(
                    execution.getStartedAt(), execution.getCompletedAt()).toMillis());
            executionRepository.save(execution);

        } catch (Exception e) {
            execution.setStatus(ExecutionStatus.FAILED);
            execution.setErrorMessage(e.getMessage());
            execution.setCompletedAt(LocalDateTime.now());
            executionRepository.save(execution);
        }
    }

    private StepResult executeStep(WorkflowExecution execution, StepDefinition step,
                                    Map<String, Object> context) {
        WorkflowStepExecution stepExec = new WorkflowStepExecution();
        stepExec.setExecution(execution);
        stepExec.setStepId(step.getId());
        stepExec.setStepName(step.getName());
        stepExec.setStepType(step.getType());
        stepExec.setStatus(StepStatus.RUNNING);
        stepExec.setInputData(toJson(context));
        stepExec.setStartedAt(LocalDateTime.now());
        stepExec = stepExecutionRepository.save(stepExec);

        StepExecutor executor = stepExecutorFactory.getExecutor(step.getType());
        StepResult result = executor.execute(step, context);

        stepExec.setStatus(result.isSuccess() ? StepStatus.COMPLETED : StepStatus.FAILED);
        stepExec.setOutputData(toJson(result.getOutputData()));
        stepExec.setErrorMessage(result.getErrorMessage());
        stepExec.setCompletedAt(LocalDateTime.now());
        stepExec.setDurationMs(Duration.between(stepExec.getStartedAt(), stepExec.getCompletedAt()).toMillis());
        stepExecutionRepository.save(stepExec);

        return result;
    }

    // Resume execution หลังจาก Approval
    @Transactional
    public void resumeAfterApproval(Long executionId, boolean approved, String comment) {
        WorkflowExecution execution = executionRepository.findById(executionId).orElseThrow();

        if (approved) {
            execution.setStatus(ExecutionStatus.RUNNING);
            executionRepository.save(execution);
            executeWorkflowAsync(executionId);
        } else {
            execution.setStatus(ExecutionStatus.CANCELLED);
            execution.setErrorMessage("Rejected by approver: " + comment);
            execution.setCompletedAt(LocalDateTime.now());
            executionRepository.save(execution);
        }
    }
}

// WorkflowController.java
@RestController
@RequestMapping("/api/workflows")
public class WorkflowController {

    @Autowired
    private WorkflowEngine workflowEngine;

    @Autowired
    private WorkflowDefinitionService definitionService;

    @PostMapping
    @PreAuthorize("hasRole('WORKFLOW_ADMIN')")
    public ResponseEntity<WorkflowResponse> create(@RequestBody CreateWorkflowRequest request) {
        WorkflowDefinition workflow = definitionService.create(request);
        return ResponseEntity.status(HttpStatus.CREATED).body(WorkflowResponse.from(workflow));
    }

    @PostMapping("/{id}/trigger")
    @PreAuthorize("isAuthenticated()")
    public ResponseEntity<ExecutionResponse> trigger(
            @PathVariable Long id,
            @RequestBody(required = false) Map<String, Object> inputData,
            @AuthenticationPrincipal User user) {
        WorkflowDefinition workflow = definitionService.findById(id);
        WorkflowExecution execution = workflowEngine.startExecution(workflow, inputData, user.getId());
        return ResponseEntity.status(HttpStatus.ACCEPTED).body(ExecutionResponse.from(execution));
    }

    @GetMapping("/executions/{id}")
    public ResponseEntity<ExecutionDetailResponse> getExecution(@PathVariable Long id) {
        return ResponseEntity.ok(workflowEngine.getExecutionDetail(id));
    }

    @PostMapping("/approvals/{id}/respond")
    @PreAuthorize("isAuthenticated()")
    public ResponseEntity<Void> respondToApproval(
            @PathVariable Long id,
            @RequestBody ApprovalResponse response,
            @AuthenticationPrincipal User user) {
        workflowEngine.processApprovalResponse(id, user.getId(), response.isApproved(), response.getComment());
        return ResponseEntity.ok().build();
    }
}
```

### SQL Scripts

```sql
-- migrations/V1__workflow_engine.sql
CREATE TABLE workflow_definitions (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    steps JSONB NOT NULL DEFAULT '[]',
    trigger_type VARCHAR(50) DEFAULT 'MANUAL',
    trigger_config JSONB,
    status VARCHAR(50) DEFAULT 'DRAFT',
    version INTEGER DEFAULT 1,
    created_by BIGINT,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE workflow_executions (
    id BIGSERIAL PRIMARY KEY,
    workflow_id BIGINT REFERENCES workflow_definitions(id),
    status VARCHAR(50) DEFAULT 'PENDING',
    input_data JSONB,
    context_data JSONB,
    current_step_id VARCHAR(255),
    error_message TEXT,
    triggered_by BIGINT,
    started_at TIMESTAMP DEFAULT NOW(),
    completed_at TIMESTAMP,
    duration_ms BIGINT
);

CREATE TABLE workflow_step_executions (
    id BIGSERIAL PRIMARY KEY,
    execution_id BIGINT REFERENCES workflow_executions(id) ON DELETE CASCADE,
    step_id VARCHAR(255) NOT NULL,
    step_name VARCHAR(255),
    step_type VARCHAR(50),
    status VARCHAR(50) DEFAULT 'PENDING',
    input_data JSONB,
    output_data JSONB,
    error_message TEXT,
    started_at TIMESTAMP,
    completed_at TIMESTAMP,
    duration_ms BIGINT
);

CREATE TABLE approval_requests (
    id BIGSERIAL PRIMARY KEY,
    execution_id BIGINT REFERENCES workflow_executions(id),
    step_id VARCHAR(255),
    approver_id BIGINT NOT NULL,
    status VARCHAR(50) DEFAULT 'PENDING',
    comment TEXT,
    requested_at TIMESTAMP DEFAULT NOW(),
    responded_at TIMESTAMP,
    expires_at TIMESTAMP
);

CREATE INDEX idx_executions_workflow ON workflow_executions(workflow_id, status);
CREATE INDEX idx_approvals_approver ON approval_requests(approver_id, status);
```

---

## โปรเจค 87: Contract Management System

### ภาพรวมโปรเจค

Contract Management System เป็นระบบจัดการสัญญาครบวงจร รองรับการสร้างสัญญา, จัดการคู่สัญญา (Parties), Milestones, Payment Schedules, E-signature Flow, Version Control, การติดตามวันหมดอายุ และ Clause Library สำหรับนำ Clause มาใช้ซ้ำ

### Entities

```java
// Contract.java — สัญญาหลัก
@Entity
@Table(name = "contracts")
public class Contract {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(unique = true, nullable = false)
    private String contractNumber;

    @Column(nullable = false)
    private String title;

    @Column(columnDefinition = "text")
    private String content; // สัญญาในรูป Markdown หรือ HTML

    @Enumerated(EnumType.STRING)
    private ContractType type; // SERVICE_AGREEMENT, NDA, EMPLOYMENT, PURCHASE

    @Enumerated(EnumType.STRING)
    private ContractStatus status; // DRAFT, PENDING_REVIEW, PENDING_SIGNATURE, ACTIVE, EXPIRED, TERMINATED

    @ManyToOne
    @JoinColumn(name = "created_by")
    private User createdBy;

    private BigDecimal totalValue;
    private String currency;

    private LocalDate startDate;
    private LocalDate endDate;
    private Integer noticeDaysBeforeExpiry;
    private boolean autoRenew;

    @OneToMany(mappedBy = "contract", cascade = CascadeType.ALL)
    private List<ContractParty> parties = new ArrayList<>();

    @OneToMany(mappedBy = "contract", cascade = CascadeType.ALL)
    private List<ContractMilestone> milestones = new ArrayList<>();

    @OneToMany(mappedBy = "contract", cascade = CascadeType.ALL)
    private List<ContractVersion> versions = new ArrayList<>();

    private Integer currentVersion;
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
}

// ContractParty.java — คู่สัญญา
@Entity
@Table(name = "contract_parties")
public class ContractParty {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne
    @JoinColumn(name = "contract_id")
    private Contract contract;

    private String partyName;
    private String partyType; // INDIVIDUAL, COMPANY
    private String role; // BUYER, SELLER, SERVICE_PROVIDER, CLIENT

    private String email;
    private String phone;
    private String address;
    private String taxId;

    // E-Signature
    private String signatureStatus; // PENDING, SIGNED, DECLINED
    private String signatureImageUrl;
    private String signatureIpAddress;
    private LocalDateTime signedAt;
    private String signatureToken; // Token สำหรับ signing link
}

// ContractMilestone.java — จุดสำคัญในสัญญา
@Entity
@Table(name = "contract_milestones")
public class ContractMilestone {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne
    @JoinColumn(name = "contract_id")
    private Contract contract;

    @Column(nullable = false)
    private String name;

    private String description;
    private LocalDate dueDate;
    private BigDecimal paymentAmount;

    @Enumerated(EnumType.STRING)
    private MilestoneStatus status; // PENDING, IN_PROGRESS, COMPLETED, DELAYED

    private LocalDateTime completedAt;
    private String completionNote;
}

// ContractClause.java — Clause Library
@Entity
@Table(name = "contract_clauses")
public class ContractClause {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String name;

    private String category; // LIABILITY, IP, CONFIDENTIALITY, TERMINATION

    @Column(columnDefinition = "text", nullable = false)
    private String content;

    private String description;
    private boolean active;
    private Long usageCount;
    private LocalDateTime createdAt;
}
```

### Service Layer

```java
// ContractService.java — จัดการสัญญา
@Service
@Transactional
public class ContractService {

    @Autowired
    private ContractRepository contractRepository;

    @Autowired
    private ContractVersionRepository versionRepository;

    @Autowired
    private ESignatureService signatureService;

    @Autowired
    private EmailService emailService;

    // สร้างสัญญาใหม่
    public Contract createContract(CreateContractRequest request, User creator) {
        Contract contract = new Contract();
        contract.setContractNumber(generateContractNumber());
        contract.setTitle(request.getTitle());
        contract.setContent(request.getContent());
        contract.setType(request.getType());
        contract.setStatus(ContractStatus.DRAFT);
        contract.setCreatedBy(creator);
        contract.setTotalValue(request.getTotalValue());
        contract.setCurrency(request.getCurrency());
        contract.setStartDate(request.getStartDate());
        contract.setEndDate(request.getEndDate());
        contract.setCurrentVersion(1);
        contract.setCreatedAt(LocalDateTime.now());
        contract.setUpdatedAt(LocalDateTime.now());

        contract = contractRepository.save(contract);

        // บันทึก Version 1
        saveVersion(contract, request.getContent(), creator, "Initial version");

        return contract;
    }

    // ส่งสัญญาให้ลงนาม
    public Contract sendForSignature(Long contractId, User sender) {
        Contract contract = contractRepository.findById(contractId).orElseThrow();

        if (contract.getStatus() != ContractStatus.DRAFT &&
                contract.getStatus() != ContractStatus.PENDING_REVIEW) {
            throw new InvalidContractStatusException("Contract is not ready for signature");
        }

        contract.setStatus(ContractStatus.PENDING_SIGNATURE);
        contract = contractRepository.save(contract);

        // สร้าง Signature Token และส่ง Email ให้แต่ละ Party
        for (ContractParty party : contract.getParties()) {
            String token = signatureService.generateSigningToken(contract.getId(), party.getId());
            party.setSignatureToken(token);
            party.setSignatureStatus("PENDING");

            emailService.sendSigningRequest(
                    party.getEmail(),
                    party.getPartyName(),
                    contract.getTitle(),
                    token
            );
        }

        return contract;
    }

    // Party ลงนามสัญญา
    public ContractParty signContract(String signingToken, String signatureImageBase64,
                                       String ipAddress) {
        ContractParty party = contractPartyRepository.findBySignatureToken(signingToken)
                .orElseThrow(() -> new InvalidSigningTokenException("Invalid signing token"));

        if (party.getSignedAt() != null) {
            throw new AlreadySignedException("Already signed");
        }

        // บันทึก Signature
        String signatureImageUrl = uploadSignatureImage(signatureImageBase64, party.getId());

        party.setSignatureStatus("SIGNED");
        party.setSignatureImageUrl(signatureImageUrl);
        party.setSignatureIpAddress(ipAddress);
        party.setSignedAt(LocalDateTime.now());
        party.setSignatureToken(null); // ลบ token หลังใช้แล้ว
        contractPartyRepository.save(party);

        // ตรวจสอบว่าทุก Party ลงนามแล้วหรือยัง
        checkAllPartiesSigned(party.getContract());

        return party;
    }

    private void checkAllPartiesSigned(Contract contract) {
        boolean allSigned = contract.getParties().stream()
                .allMatch(p -> "SIGNED".equals(p.getSignatureStatus()));

        if (allSigned) {
            contract.setStatus(ContractStatus.ACTIVE);
            contractRepository.save(contract);

            // แจ้งเตือนทุก Party ว่าสัญญา Active แล้ว
            for (ContractParty party : contract.getParties()) {
                emailService.sendContractActivatedEmail(party.getEmail(), contract.getTitle());
            }
        }
    }

    // ตรวจหาสัญญาที่กำลังจะหมดอายุ (ใช้กับ Scheduled Task)
    @Scheduled(cron = "0 0 9 * * *") // ทุกวันเวลา 9 โมง
    public void checkExpiringContracts() {
        List<Contract> expiring = contractRepository.findContractsExpiringSoon(
                LocalDate.now().plusDays(30));

        for (Contract contract : expiring) {
            int daysLeft = (int) ChronoUnit.DAYS.between(LocalDate.now(), contract.getEndDate());

            for (ContractParty party : contract.getParties()) {
                emailService.sendExpiryReminder(party.getEmail(), contract.getTitle(),
                        daysLeft, contract.isAutoRenew());
            }
        }
    }

    private String generateContractNumber() {
        return "CNT-" + Year.now().getValue() + "-" +
                String.format("%06d", contractRepository.count() + 1);
    }
}
```

---

## โปรเจค 88: Compliance Management System

### ภาพรวมโปรเจค

Compliance Management System เป็นระบบจัดการการปฏิบัติตามกฎระเบียบและมาตรฐาน รองรับการกำหนด Requirements, Controls, Assessments, การเก็บ Evidence, Risk Scoring, Audit Trails, Reporting และ Periodic Reviews

### Entities

```java
// ComplianceRequirement.java — ข้อกำหนดที่ต้องปฏิบัติตาม เช่น GDPR Article 5
@Entity
@Table(name = "compliance_requirements")
public class ComplianceRequirement {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String code; // GDPR-5.1, ISO27001-A.5.1

    @Column(nullable = false)
    private String title;

    @Column(columnDefinition = "text")
    private String description;

    private String framework; // GDPR, ISO27001, SOC2, HIPAA, PCI-DSS

    @Enumerated(EnumType.STRING)
    private RequirementSeverity severity; // CRITICAL, HIGH, MEDIUM, LOW

    @ManyToOne
    @JoinColumn(name = "parent_id")
    private ComplianceRequirement parent;

    private boolean active;
    private LocalDateTime createdAt;
}

// ComplianceControl.java — การควบคุมที่ implement เพื่อตอบสนอง Requirement
@Entity
@Table(name = "compliance_controls")
public class ComplianceControl {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String name;

    private String description;

    @Enumerated(EnumType.STRING)
    private ControlType type; // PREVENTIVE, DETECTIVE, CORRECTIVE

    @ManyToMany
    @JoinTable(name = "control_requirements")
    private Set<ComplianceRequirement> requirements = new HashSet<>();

    @Enumerated(EnumType.STRING)
    private ControlStatus status; // IMPLEMENTED, PARTIALLY_IMPLEMENTED, NOT_IMPLEMENTED

    private Long ownerId; // User ที่รับผิดชอบ
    private Float effectivenessScore; // 0-100

    private LocalDate lastReviewedAt;
    private LocalDate nextReviewDate;
    private LocalDateTime createdAt;
}

// ComplianceAssessment.java — การประเมินความสอดคล้อง
@Entity
@Table(name = "compliance_assessments")
public class ComplianceAssessment {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String name;

    private String framework;

    @Enumerated(EnumType.STRING)
    private AssessmentStatus status; // PLANNED, IN_PROGRESS, COMPLETED

    private Float overallScore;
    private String assessorName;

    private LocalDate startDate;
    private LocalDate endDate;
    private LocalDate nextAssessmentDate;

    @OneToMany(mappedBy = "assessment", cascade = CascadeType.ALL)
    private List<AssessmentFinding> findings = new ArrayList<>();

    private LocalDateTime createdAt;
}

// Evidence.java — หลักฐานสนับสนุนการปฏิบัติตามกฎระเบียบ
@Entity
@Table(name = "compliance_evidence")
public class Evidence {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne
    @JoinColumn(name = "control_id")
    private ComplianceControl control;

    @Column(nullable = false)
    private String title;

    private String description;
    private String fileUrl;
    private String fileType;
    private String uploadedBy;

    @Enumerated(EnumType.STRING)
    private EvidenceStatus status; // VALID, EXPIRED, PENDING_REVIEW

    private LocalDate validUntil;
    private LocalDateTime uploadedAt;
}
```

### Service Layer

```java
// ComplianceService.java — จัดการการ Assess และ Scoring
@Service
@Transactional
public class ComplianceService {

    @Autowired
    private ComplianceControlRepository controlRepository;

    @Autowired
    private ComplianceAssessmentRepository assessmentRepository;

    @Autowired
    private AuditLogService auditLogService;

    // คำนวณ Risk Score สำหรับ Control แต่ละตัว
    public Float calculateRiskScore(Long controlId) {
        ComplianceControl control = controlRepository.findById(controlId).orElseThrow();

        Float score = 0f;

        // ตรวจสอบสถานะ Implementation
        switch (control.getStatus()) {
            case IMPLEMENTED -> score += 50;
            case PARTIALLY_IMPLEMENTED -> score += 25;
            case NOT_IMPLEMENTED -> score += 0;
        }

        // Effectiveness score
        if (control.getEffectivenessScore() != null) {
            score += control.getEffectivenessScore() * 0.3f;
        }

        // Evidence recency
        long validEvidenceCount = evidenceRepository.countValidByControlId(controlId);
        score += Math.min(validEvidenceCount * 5, 20f);

        return Math.min(score, 100f);
    }

    // สร้าง Compliance Report
    public ComplianceReport generateReport(String framework) {
        List<ComplianceControl> controls = controlRepository.findByFramework(framework);

        Map<ControlStatus, Long> statusBreakdown = controls.stream()
                .collect(Collectors.groupingBy(ComplianceControl::getStatus, Collectors.counting()));

        long total = controls.size();
        long implemented = statusBreakdown.getOrDefault(ControlStatus.IMPLEMENTED, 0L);
        float overallScore = total > 0 ? (float) implemented / total * 100 : 0;

        List<ControlGap> gaps = controls.stream()
                .filter(c -> c.getStatus() != ControlStatus.IMPLEMENTED)
                .map(c -> ControlGap.from(c, calculateRiskScore(c.getId())))
                .sorted(Comparator.comparing(ControlGap::getRiskScore).reversed())
                .collect(Collectors.toList());

        return ComplianceReport.builder()
                .framework(framework)
                .totalControls(total)
                .implementedControls(implemented)
                .overallScore(overallScore)
                .statusBreakdown(statusBreakdown)
                .gaps(gaps)
                .generatedAt(LocalDateTime.now())
                .build();
    }

    // Periodic Review reminder
    @Scheduled(cron = "0 0 8 * * MON") // ทุกวันจันทร์ เวลา 8 โมง
    public void sendReviewReminders() {
        List<ComplianceControl> dueForReview = controlRepository
                .findByNextReviewDateBefore(LocalDate.now().plusDays(7));

        for (ComplianceControl control : dueForReview) {
            User owner = userRepository.findById(control.getOwnerId()).orElse(null);
            if (owner != null) {
                emailService.sendReviewReminder(owner.getEmail(), control.getName(),
                        control.getNextReviewDate());
            }
        }
    }
}

// ComplianceController.java
@RestController
@RequestMapping("/api/compliance")
@PreAuthorize("hasRole('COMPLIANCE_OFFICER')")
public class ComplianceController {

    @Autowired
    private ComplianceService complianceService;

    @GetMapping("/report/{framework}")
    public ResponseEntity<ComplianceReport> getReport(@PathVariable String framework) {
        return ResponseEntity.ok(complianceService.generateReport(framework));
    }

    @PostMapping("/assessments")
    public ResponseEntity<AssessmentResponse> createAssessment(
            @RequestBody CreateAssessmentRequest request) {
        ComplianceAssessment assessment = complianceService.createAssessment(request);
        return ResponseEntity.status(HttpStatus.CREATED).body(AssessmentResponse.from(assessment));
    }

    @PostMapping("/controls/{id}/evidence")
    public ResponseEntity<EvidenceResponse> uploadEvidence(
            @PathVariable Long id,
            @RequestParam("file") MultipartFile file,
            @RequestBody EvidenceRequest request) throws IOException {
        Evidence evidence = complianceService.uploadEvidence(id, file, request);
        return ResponseEntity.status(HttpStatus.CREATED).body(EvidenceResponse.from(evidence));
    }
}
```

---

## โปรเจค 89: Expense Tracker & Approvals

### ภาพรวมโปรเจค

Expense Tracker & Approvals เป็นระบบจัดการค่าใช้จ่ายของพนักงาน รองรับการยื่น Expense Claims, อัปโหลดใบเสร็จ, จัดหมวดหมู่ค่าใช้จ่าย, Approval Workflow แบบ Manager Chain, การเบิกคืนเงิน, กำหนด Budget Limits และสร้าง Reports

### Entities

```java
// ExpenseClaim.java — คำขอเบิกค่าใช้จ่าย
@Entity
@Table(name = "expense_claims")
public class ExpenseClaim {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(unique = true, nullable = false)
    private String claimNumber;

    @ManyToOne
    @JoinColumn(name = "employee_id")
    private User employee;

    @Column(nullable = false)
    private String title;

    private String description;

    @OneToMany(mappedBy = "claim", cascade = CascadeType.ALL)
    private List<ExpenseItem> items = new ArrayList<>();

    private BigDecimal totalAmount;
    private String currency;

    @Enumerated(EnumType.STRING)
    private ClaimStatus status; // DRAFT, SUBMITTED, PENDING_APPROVAL, APPROVED, REJECTED, REIMBURSED

    // Approval chain
    private Long currentApproverId;
    private int approvalLevel;

    private String rejectionReason;
    private LocalDateTime submittedAt;
    private LocalDateTime approvedAt;
    private LocalDateTime reimbursedAt;
    private LocalDateTime createdAt;
}

// ExpenseItem.java — รายการค่าใช้จ่ายแต่ละรายการ
@Entity
@Table(name = "expense_items")
public class ExpenseItem {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne
    @JoinColumn(name = "claim_id")
    private ExpenseClaim claim;

    @Column(nullable = false)
    private String description;

    @ManyToOne
    @JoinColumn(name = "category_id")
    private ExpenseCategory category; // TRAVEL, MEALS, ACCOMMODATION, SUPPLIES

    @Column(nullable = false)
    private BigDecimal amount;

    private String currency;
    private LocalDate expenseDate;

    // ใบเสร็จ
    private String receiptUrl;
    private String receiptS3Key;

    private String merchant;
    private String notes;
}

// ExpenseApproval.java — ประวัติการอนุมัติ
@Entity
@Table(name = "expense_approvals")
public class ExpenseApproval {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne
    @JoinColumn(name = "claim_id")
    private ExpenseClaim claim;

    @ManyToOne
    @JoinColumn(name = "approver_id")
    private User approver;

    private int approvalLevel;

    @Enumerated(EnumType.STRING)
    private ApprovalDecision decision; // APPROVED, REJECTED, DELEGATED

    private String comment;
    private LocalDateTime decidedAt;
}
```

### Service Layer

```java
// ExpenseService.java — จัดการ Expense Claims
@Service
@Transactional
public class ExpenseService {

    @Autowired
    private ExpenseClaimRepository claimRepository;

    @Autowired
    private ExpenseApprovalRepository approvalRepository;

    @Autowired
    private BudgetService budgetService;

    @Autowired
    private S3Client s3Client;

    @Autowired
    private EmailService emailService;

    // ยื่น Claim
    public ExpenseClaim submitClaim(Long claimId, Long employeeId) {
        ExpenseClaim claim = claimRepository.findById(claimId).orElseThrow();

        if (!claim.getEmployee().getId().equals(employeeId)) {
            throw new AccessDeniedException("Not your claim");
        }

        if (claim.getStatus() != ClaimStatus.DRAFT) {
            throw new InvalidClaimStatusException("Claim is not in DRAFT status");
        }

        // ตรวจสอบ Budget Limit
        budgetService.checkBudgetLimit(
                claim.getEmployee().getDepartmentId(),
                claim.getTotalAmount());

        claim.setStatus(ClaimStatus.SUBMITTED);
        claim.setSubmittedAt(LocalDateTime.now());
        claim.setApprovalLevel(1);

        // หา Manager คนแรก
        User firstApprover = findApprover(claim.getEmployee(), 1);
        claim.setCurrentApproverId(firstApprover.getId());

        claim = claimRepository.save(claim);

        // แจ้ง Manager
        emailService.sendApprovalRequest(
                firstApprover.getEmail(),
                claim.getEmployee().getName(),
                claim.getTotalAmount(),
                claim.getId()
        );

        return claim;
    }

    // Approve หรือ Reject
    public ExpenseClaim processApproval(Long claimId, Long approverId,
                                         boolean approved, String comment) {
        ExpenseClaim claim = claimRepository.findById(claimId).orElseThrow();

        if (!claim.getCurrentApproverId().equals(approverId)) {
            throw new NotAuthorizedException("Not the current approver");
        }

        // บันทึกผลการ Approve
        ExpenseApproval approval = new ExpenseApproval();
        approval.setClaim(claim);
        approval.setApprover(new User(approverId));
        approval.setApprovalLevel(claim.getApprovalLevel());
        approval.setDecision(approved ? ApprovalDecision.APPROVED : ApprovalDecision.REJECTED);
        approval.setComment(comment);
        approval.setDecidedAt(LocalDateTime.now());
        approvalRepository.save(approval);

        if (!approved) {
            claim.setStatus(ClaimStatus.REJECTED);
            claim.setRejectionReason(comment);
            emailService.sendClaimRejected(claim.getEmployee().getEmail(),
                    claim.getTotalAmount(), comment);
        } else {
            // ตรวจสอบว่าต้องผ่าน Manager Level ถัดไปหรือเปล่า
            User nextApprover = findApprover(claim.getEmployee(), claim.getApprovalLevel() + 1);

            if (nextApprover != null && claim.getTotalAmount().compareTo(getThresholdForLevel(claim.getApprovalLevel() + 1)) >= 0) {
                // ส่งต่อ Manager คนถัดไป
                claim.setApprovalLevel(claim.getApprovalLevel() + 1);
                claim.setCurrentApproverId(nextApprover.getId());
                claim.setStatus(ClaimStatus.PENDING_APPROVAL);
                emailService.sendApprovalRequest(nextApprover.getEmail(),
                        claim.getEmployee().getName(), claim.getTotalAmount(), claim.getId());
            } else {
                // Approved สมบูรณ์
                claim.setStatus(ClaimStatus.APPROVED);
                claim.setApprovedAt(LocalDateTime.now());
                emailService.sendClaimApproved(claim.getEmployee().getEmail(), claim.getTotalAmount());
            }
        }

        return claimRepository.save(claim);
    }

    // Mark as Reimbursed
    public ExpenseClaim markAsReimbursed(Long claimId, String paymentReference) {
        ExpenseClaim claim = claimRepository.findById(claimId).orElseThrow();
        claim.setStatus(ClaimStatus.REIMBURSED);
        claim.setReimbursedAt(LocalDateTime.now());
        return claimRepository.save(claim);
    }
}

// ExpenseController.java
@RestController
@RequestMapping("/api/expenses")
public class ExpenseController {

    @Autowired
    private ExpenseService expenseService;

    @PostMapping("/claims")
    @PreAuthorize("isAuthenticated()")
    public ResponseEntity<ClaimResponse> createClaim(
            @RequestBody CreateClaimRequest request,
            @AuthenticationPrincipal User user) {
        ExpenseClaim claim = expenseService.createClaim(request, user.getId());
        return ResponseEntity.status(HttpStatus.CREATED).body(ClaimResponse.from(claim));
    }

    @PostMapping("/claims/{id}/submit")
    @PreAuthorize("isAuthenticated()")
    public ResponseEntity<ClaimResponse> submit(
            @PathVariable Long id,
            @AuthenticationPrincipal User user) {
        ExpenseClaim claim = expenseService.submitClaim(id, user.getId());
        return ResponseEntity.ok(ClaimResponse.from(claim));
    }

    @PostMapping("/claims/{id}/approve")
    @PreAuthorize("hasRole('MANAGER')")
    public ResponseEntity<ClaimResponse> approve(
            @PathVariable Long id,
            @RequestBody ApprovalRequest request,
            @AuthenticationPrincipal User user) {
        ExpenseClaim claim = expenseService.processApproval(id, user.getId(),
                request.isApproved(), request.getComment());
        return ResponseEntity.ok(ClaimResponse.from(claim));
    }

    @PostMapping("/claims/{id}/receipt")
    @PreAuthorize("isAuthenticated()")
    public ResponseEntity<Void> uploadReceipt(
            @PathVariable Long id,
            @RequestParam("file") MultipartFile file,
            @AuthenticationPrincipal User user) throws IOException {
        expenseService.uploadReceipt(id, file, user.getId());
        return ResponseEntity.ok().build();
    }

    @GetMapping("/reports/monthly")
    @PreAuthorize("hasRole('FINANCE')")
    public ResponseEntity<MonthlyReport> getMonthlyReport(
            @RequestParam int year,
            @RequestParam int month) {
        return ResponseEntity.ok(expenseService.generateMonthlyReport(year, month));
    }
}
```

---

## โปรเจค 90: Budget Management System

### ภาพรวมโปรเจค

Budget Management System เป็นระบบจัดการงบประมาณขององค์กร รองรับการสร้าง Budgets รายแผนก, การ Allocate งบประมาณ, ติดตามยอดใช้จ่ายจริง, Variance Reports, คำของบประมาณเพิ่มเติม, Approval Workflow และการแจ้งเตือนเมื่อใกล้เต็ม Budget

### Entities

```java
// Budget.java — งบประมาณหลักของปีงบประมาณ
@Entity
@Table(name = "budgets")
public class Budget {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String name;

    private String description;
    private Integer fiscalYear;

    @Column(nullable = false)
    private BigDecimal totalAmount;

    private BigDecimal allocatedAmount;
    private BigDecimal spentAmount;
    private String currency;

    @Enumerated(EnumType.STRING)
    private BudgetStatus status; // PLANNING, APPROVED, ACTIVE, CLOSED

    private LocalDate startDate;
    private LocalDate endDate;

    @ManyToOne
    @JoinColumn(name = "department_id")
    private Department department;

    @OneToMany(mappedBy = "budget", cascade = CascadeType.ALL)
    private List<BudgetAllocation> allocations = new ArrayList<>();

    // Alert thresholds
    private Float warningThresholdPercent; // แจ้งเตือนเมื่อใช้ถึง % นี้
    private Float criticalThresholdPercent;

    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
}

// BudgetAllocation.java — การแบ่ง Budget ให้กับ Category หรือ Project
@Entity
@Table(name = "budget_allocations")
public class BudgetAllocation {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne
    @JoinColumn(name = "budget_id")
    private Budget budget;

    @Column(nullable = false)
    private String category; // PERSONNEL, TECHNOLOGY, MARKETING, TRAVEL

    private String description;
    private BigDecimal allocatedAmount;
    private BigDecimal spentAmount;
    private BigDecimal remainingAmount;

    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
}

// BudgetRequest.java — คำของบประมาณเพิ่มเติม
@Entity
@Table(name = "budget_requests")
public class BudgetRequest {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne
    @JoinColumn(name = "budget_id")
    private Budget budget;

    @ManyToOne
    @JoinColumn(name = "requested_by")
    private User requestedBy;

    @Column(nullable = false)
    private String title;

    private String justification;
    private BigDecimal requestedAmount;
    private String category;

    @Enumerated(EnumType.STRING)
    private RequestStatus status; // PENDING, APPROVED, REJECTED

    private String approverComment;
    private Long approverId;

    private LocalDateTime requestedAt;
    private LocalDateTime decidedAt;
}

// ActualSpending.java — ยอดใช้จ่ายจริง
@Entity
@Table(name = "actual_spendings")
public class ActualSpending {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne
    @JoinColumn(name = "budget_id")
    private Budget budget;

    private String category;
    private BigDecimal amount;
    private String currency;

    private String description;
    private String referenceType; // EXPENSE_CLAIM, PURCHASE_ORDER, INVOICE
    private Long referenceId;

    private LocalDate spendingDate;
    private LocalDateTime recordedAt;
}
```

### Service Layer

```java
// BudgetService.java — จัดการงบประมาณ
@Service
@Transactional
public class BudgetService {

    @Autowired
    private BudgetRepository budgetRepository;

    @Autowired
    private ActualSpendingRepository spendingRepository;

    @Autowired
    private BudgetAlertService alertService;

    // บันทึกการใช้จ่าย
    public void recordSpending(Long budgetId, String category, BigDecimal amount,
                                String description, String referenceType, Long referenceId) {
        Budget budget = budgetRepository.findById(budgetId).orElseThrow();

        // หา Allocation ที่ตรงกัน
        BudgetAllocation allocation = budget.getAllocations().stream()
                .filter(a -> a.getCategory().equals(category))
                .findFirst()
                .orElseThrow(() -> new AllocationNotFoundException(
                        "No allocation found for category: " + category));

        if (allocation.getRemainingAmount().compareTo(amount) < 0) {
            throw new InsufficientBudgetException(
                    "Insufficient budget. Requested: " + amount + ", Available: " + allocation.getRemainingAmount());
        }

        // อัปเดต Allocation
        allocation.setSpentAmount(allocation.getSpentAmount().add(amount));
        allocation.setRemainingAmount(allocation.getRemainingAmount().subtract(amount));

        // อัปเดต Budget รวม
        budget.setSpentAmount(budget.getSpentAmount().add(amount));

        // บันทึก Actual Spending
        ActualSpending spending = new ActualSpending();
        spending.setBudget(budget);
        spending.setCategory(category);
        spending.setAmount(amount);
        spending.setDescription(description);
        spending.setReferenceType(referenceType);
        spending.setReferenceId(referenceId);
        spending.setSpendingDate(LocalDate.now());
        spending.setRecordedAt(LocalDateTime.now());
        spendingRepository.save(spending);

        budgetRepository.save(budget);

        // ตรวจสอบ Alert Thresholds
        checkAlertThresholds(budget, allocation);
    }

    // สร้าง Variance Report
    public VarianceReport generateVarianceReport(Long budgetId) {
        Budget budget = budgetRepository.findById(budgetId).orElseThrow();

        List<VarianceItem> items = budget.getAllocations().stream()
                .map(a -> {
                    BigDecimal variance = a.getSpentAmount().subtract(a.getAllocatedAmount());
                    float variancePercent = a.getAllocatedAmount().compareTo(BigDecimal.ZERO) > 0 ?
                            variance.divide(a.getAllocatedAmount(), 4, RoundingMode.HALF_UP)
                                    .multiply(BigDecimal.valueOf(100)).floatValue() : 0;

                    return VarianceItem.builder()
                            .category(a.getCategory())
                            .allocated(a.getAllocatedAmount())
                            .actual(a.getSpentAmount())
                            .variance(variance)
                            .variancePercent(variancePercent)
                            .status(variance.compareTo(BigDecimal.ZERO) > 0 ? "OVER" :
                                    variancePercent < -10 ? "UNDER" : "ON_TRACK")
                            .build();
                })
                .collect(Collectors.toList());

        return VarianceReport.builder()
                .budgetId(budgetId)
                .budgetName(budget.getName())
                .fiscalYear(budget.getFiscalYear())
                .totalAllocated(budget.getAllocatedAmount())
                .totalActual(budget.getSpentAmount())
                .totalVariance(budget.getSpentAmount().subtract(budget.getAllocatedAmount()))
                .items(items)
                .generatedAt(LocalDateTime.now())
                .build();
    }

    // Check thresholds และส่ง Alert
    private void checkAlertThresholds(Budget budget, BudgetAllocation allocation) {
        if (budget.getWarningThresholdPercent() == null) return;

        float spentPercent = budget.getSpentAmount()
                .divide(budget.getTotalAmount(), 4, RoundingMode.HALF_UP)
                .multiply(BigDecimal.valueOf(100))
                .floatValue();

        if (spentPercent >= budget.getCriticalThresholdPercent()) {
            alertService.sendCriticalAlert(budget, spentPercent);
        } else if (spentPercent >= budget.getWarningThresholdPercent()) {
            alertService.sendWarningAlert(budget, spentPercent);
        }
    }

    // ตรวจสอบ Budget Limit สำหรับ Expense Claims
    public void checkBudgetLimit(Long departmentId, BigDecimal amount) {
        Budget activeBudget = budgetRepository.findActiveBudgetByDepartment(
                departmentId, LocalDate.now())
                .orElseThrow(() -> new NoBudgetException("No active budget for department"));

        if (activeBudget.getSpentAmount().add(amount).compareTo(activeBudget.getTotalAmount()) > 0) {
            throw new BudgetExceededException("This expense would exceed the department budget");
        }
    }
}

// BudgetController.java
@RestController
@RequestMapping("/api/budget")
public class BudgetController {

    @Autowired
    private BudgetService budgetService;

    @PostMapping
    @PreAuthorize("hasRole('BUDGET_ADMIN')")
    public ResponseEntity<BudgetResponse> create(@RequestBody CreateBudgetRequest request) {
        Budget budget = budgetService.create(request);
        return ResponseEntity.status(HttpStatus.CREATED).body(BudgetResponse.from(budget));
    }

    @GetMapping("/{id}/variance-report")
    @PreAuthorize("hasAnyRole('BUDGET_ADMIN', 'FINANCE_MANAGER')")
    public ResponseEntity<VarianceReport> varianceReport(@PathVariable Long id) {
        return ResponseEntity.ok(budgetService.generateVarianceReport(id));
    }

    @PostMapping("/requests")
    @PreAuthorize("isAuthenticated()")
    public ResponseEntity<BudgetRequestResponse> submitBudgetRequest(
            @RequestBody CreateBudgetRequestRequest request,
            @AuthenticationPrincipal User user) {
        BudgetRequest br = budgetService.submitBudgetRequest(request, user.getId());
        return ResponseEntity.status(HttpStatus.CREATED).body(BudgetRequestResponse.from(br));
    }

    @PostMapping("/requests/{id}/approve")
    @PreAuthorize("hasRole('CFO')")
    public ResponseEntity<BudgetRequestResponse> approveBudgetRequest(
            @PathVariable Long id,
            @RequestBody ApprovalRequest request,
            @AuthenticationPrincipal User user) {
        BudgetRequest br = budgetService.processRequest(id, user.getId(),
                request.isApproved(), request.getComment());
        return ResponseEntity.ok(BudgetRequestResponse.from(br));
    }
}
```

### SQL Scripts

```sql
-- migrations/V1__budget_management.sql
CREATE TABLE budgets (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    fiscal_year INTEGER,
    total_amount DECIMAL(15,2) NOT NULL,
    allocated_amount DECIMAL(15,2) DEFAULT 0,
    spent_amount DECIMAL(15,2) DEFAULT 0,
    currency VARCHAR(3) DEFAULT 'THB',
    status VARCHAR(50) DEFAULT 'PLANNING',
    start_date DATE,
    end_date DATE,
    department_id BIGINT,
    warning_threshold_percent FLOAT,
    critical_threshold_percent FLOAT,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE budget_allocations (
    id BIGSERIAL PRIMARY KEY,
    budget_id BIGINT REFERENCES budgets(id) ON DELETE CASCADE,
    category VARCHAR(100) NOT NULL,
    description TEXT,
    allocated_amount DECIMAL(15,2) NOT NULL,
    spent_amount DECIMAL(15,2) DEFAULT 0,
    remaining_amount DECIMAL(15,2),
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE budget_requests (
    id BIGSERIAL PRIMARY KEY,
    budget_id BIGINT REFERENCES budgets(id),
    requested_by BIGINT NOT NULL,
    title VARCHAR(500) NOT NULL,
    justification TEXT,
    requested_amount DECIMAL(15,2) NOT NULL,
    category VARCHAR(100),
    status VARCHAR(50) DEFAULT 'PENDING',
    approver_comment TEXT,
    approver_id BIGINT,
    requested_at TIMESTAMP DEFAULT NOW(),
    decided_at TIMESTAMP
);

CREATE TABLE actual_spendings (
    id BIGSERIAL PRIMARY KEY,
    budget_id BIGINT REFERENCES budgets(id),
    category VARCHAR(100),
    amount DECIMAL(15,2) NOT NULL,
    currency VARCHAR(3) DEFAULT 'THB',
    description TEXT,
    reference_type VARCHAR(50),
    reference_id BIGINT,
    spending_date DATE,
    recorded_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_budgets_dept ON budgets(department_id, status);
CREATE INDEX idx_spendings_budget ON actual_spendings(budget_id, spending_date DESC);
```

---

## สรุป Part 118

ใน Part นี้เราได้สร้าง Workflow & Automation Systems ระดับองค์กร:

| โปรเจค | เทคโนโลยีหลัก | ความยาก |
|--------|---------------|---------|
| 86. Workflow Engine | Step execution, Approval flow, Async processing | ⭐⭐⭐⭐⭐ |
| 87. Contract Management | E-signature, Versioning, Expiry tracking | ⭐⭐⭐⭐ |
| 88. Compliance Management | Risk scoring, Evidence collection, Reporting | ⭐⭐⭐⭐ |
| 89. Expense Tracker | Multi-level approval, Receipt upload, Reimbursement | ⭐⭐⭐⭐ |
| 90. Budget Management | Allocation tracking, Variance reporting, Alerts | ⭐⭐⭐⭐ |

### Key Takeaways

1. **Workflow Engine** — State machine approach ทำให้ง่ายต่อการ debug และ resume หลัง failure
2. **Approval Chains** — ต้องมี timeout handling เพื่อไม่ให้ workflow ค้างอยู่นานเกินไป
3. **Budget Tracking** — Real-time tracking ต้องการ atomic operations เพื่อป้องกัน race conditions
4. **Compliance** — Audit trail ต้องครบถ้วนและ immutable เพื่อผ่านการตรวจสอบ

*[← Part 117](./part-117-media-entertainment.md) | [Part 119 →](./part-119-marketplace-realestate.md)*

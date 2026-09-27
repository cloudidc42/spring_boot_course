# Part 120: โปรเจค 96-100 — AI-Powered & Advanced Projects

> **ระดับ:** โลก (World-Class) | **เวลาเรียนรู้:** 10-15 ชั่วโมง  
> **เป้าหมาย:** สร้างระบบที่ขับเคลื่อนด้วย AI และ Enterprise Integration Platform ระดับ Production

---

## โปรเจค 96: AI-Powered Product Search

### ภาพรวมโปรเจค

AI-Powered Product Search เป็นระบบค้นหาสินค้าที่ใช้ AI ช่วยในการเข้าใจภาษาธรรมชาติ ผู้ใช้สามารถพิมพ์คำค้นหาแบบภาษาพูดได้ เช่น "เสื้อผ้าสีแดงสำหรับงานแต่ง ราคาไม่เกิน 2000 บาท" ระบบใช้ Spring AI สำหรับ Semantic Search ผ่าน Embeddings, Query Expansion, Personalized Ranking และ Spell Correction

### Dependencies

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-openai-spring-boot-starter</artifactId>
    <version>1.0.0</version>
</dependency>
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-pgvector-store-spring-boot-starter</artifactId>
    <version>1.0.0</version>
</dependency>
```

### Entities & Documents

```java
// ProductEmbedding.java — เก็บ Vector Embedding ของสินค้า
@Entity
@Table(name = "product_embeddings")
public class ProductEmbedding {
    @Id
    private Long productId;

    // Embedding vector (1536 dimensions สำหรับ text-embedding-ada-002)
    @Column(columnDefinition = "vector(1536)")
    private float[] embedding;

    // Text ที่ใช้สร้าง embedding
    private String embeddingText;

    private LocalDateTime updatedAt;
}

// SearchAnalytics.java — บันทึก Query และผลลัพธ์สำหรับ Analytics
@Entity
@Table(name = "ai_search_analytics")
public class SearchAnalytics {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String originalQuery;
    private String processedQuery;
    private String expandedQuery;

    @Column(columnDefinition = "jsonb")
    private String extractedFilters; // {"color": "red", "maxPrice": 2000}

    private Integer resultsCount;
    private Long userId;
    private Long responseTimeMs;
    private String model;

    private LocalDateTime searchedAt;
}
```

### AI Search Service

```java
// AiProductSearchService.java — ระบบค้นหาด้วย AI
@Service
public class AiProductSearchService {

    @Autowired
    private ChatClient chatClient;

    @Autowired
    private EmbeddingModel embeddingModel;

    @Autowired
    private VectorStore vectorStore; // pgvector

    @Autowired
    private ProductRepository productRepository;

    @Autowired
    private SearchAnalyticsRepository analyticsRepository;

    // Natural Language Search — เข้าใจภาษาพูด
    public AiSearchResult naturalLanguageSearch(String query, Long userId) {
        long startTime = System.currentTimeMillis();

        // Step 1: ใช้ AI แยกแยะ Intent และ Filters จาก Natural Language Query
        SearchIntent intent = extractSearchIntent(query);

        // Step 2: สร้าง Embedding สำหรับ Semantic Search
        float[] queryEmbedding = embeddingModel.embed(query);

        // Step 3: ค้นหาด้วย Semantic Similarity
        List<Document> semanticResults = vectorStore.similaritySearch(
                SearchRequest.query(query)
                        .withTopK(50)
                        .withSimilarityThreshold(0.7)
        );

        // Step 4: กรองตาม Extracted Filters
        List<Long> productIds = semanticResults.stream()
                .map(d -> Long.parseLong(d.getMetadata().get("productId").toString()))
                .collect(Collectors.toList());

        List<Product> products = productRepository.findByIdInWithFilters(
                productIds, intent.getFilters());

        // Step 5: Personalized Ranking
        if (userId != null) {
            products = personalizeRanking(products, userId);
        }

        long responseTime = System.currentTimeMillis() - startTime;

        // บันทึก Analytics
        saveAnalytics(query, intent, products.size(), userId, responseTime);

        return AiSearchResult.builder()
                .products(products.stream().map(ProductResponse::from).collect(Collectors.toList()))
                .extractedFilters(intent.getFilters())
                .expandedQuery(intent.getExpandedQuery())
                .totalResults(products.size())
                .responseTimeMs(responseTime)
                .build();
    }

    // ดึง Intent และ Filters จาก Natural Language
    private SearchIntent extractSearchIntent(String query) {
        String systemPrompt = """
            คุณคือ AI ที่ช่วยวิเคราะห์คำค้นหาสินค้า
            จากคำค้นหาที่ได้รับ ให้สกัด:
            1. category: ประเภทสินค้าหลัก
            2. color: สีที่ต้องการ (ถ้ามี)
            3. minPrice: ราคาต่ำสุด (ถ้ามี)
            4. maxPrice: ราคาสูงสุด (ถ้ามี)
            5. occasion: โอกาสการใช้งาน (ถ้ามี)
            6. expandedQuery: คำค้นหาที่ขยายความแล้ว ภาษาอังกฤษ
            
            ตอบกลับเป็น JSON เท่านั้น
            """;

        String response = chatClient.prompt()
                .system(systemPrompt)
                .user(query)
                .call()
                .content();

        return parseSearchIntent(response);
    }

    // Personalize Ranking ตามประวัติของ User
    private List<Product> personalizeRanking(List<Product> products, Long userId) {
        // ดึง User Preference Profile
        UserPreferenceProfile profile = getUserPreferenceProfile(userId);

        return products.stream()
                .sorted(Comparator.comparingDouble(p -> -calculatePersonalizedScore(p, profile)))
                .collect(Collectors.toList());
    }

    // Index Product ใน Vector Store
    public void indexProduct(Product product) {
        String embeddingText = String.format(
                "Product: %s. Category: %s. Description: %s. Tags: %s",
                product.getName(),
                product.getCategory(),
                product.getDescription(),
                String.join(", ", product.getTags())
        );

        Document doc = new Document(
                embeddingText,
                Map.of(
                        "productId", product.getId().toString(),
                        "name", product.getName(),
                        "price", product.getBasePrice().toString(),
                        "category", product.getCategory()
                )
        );

        vectorStore.add(List.of(doc));
    }
}

// AiSearchController.java
@RestController
@RequestMapping("/api/ai-search")
public class AiSearchController {

    @Autowired
    private AiProductSearchService searchService;

    @GetMapping("/products")
    public ResponseEntity<AiSearchResult> search(
            @RequestParam String q,
            @AuthenticationPrincipal(required = false) User user) {
        Long userId = user != null ? user.getId() : null;
        return ResponseEntity.ok(searchService.naturalLanguageSearch(q, userId));
    }

    @PostMapping("/products/index/{id}")
    @PreAuthorize("hasRole('ADMIN')")
    public ResponseEntity<Void> indexProduct(@PathVariable Long id) {
        searchService.indexProductById(id);
        return ResponseEntity.ok().build();
    }
}
```

---

## โปรเจค 97: Intelligent Document Processing

### ภาพรวมโปรเจค

Intelligent Document Processing เป็นระบบประมวลผลเอกสารด้วย AI รองรับ OCR ผ่าน Tesseract หรือ AWS Textract, สกัดข้อมูลจากเอกสาร, จัดประเภทเอกสาร, ตรวจสอบความถูกต้อง, Workflow Routing และสร้าง Structured Output จากเอกสารที่ไม่มี Structure

### Entities

```java
// DocumentProcessingJob.java — งานประมวลผลเอกสาร
@Entity
@Table(name = "document_processing_jobs")
public class DocumentProcessingJob {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(unique = true, nullable = false)
    private String jobId;

    private String originalFilename;
    private String s3Key;
    private String fileType; // PDF, JPEG, PNG

    @Enumerated(EnumType.STRING)
    private DocumentType documentType; // INVOICE, RECEIPT, CONTRACT, ID_CARD, MEDICAL_REPORT

    @Enumerated(EnumType.STRING)
    private JobStatus status; // QUEUED, PROCESSING, COMPLETED, FAILED

    // Raw OCR text
    @Column(columnDefinition = "text")
    private String ocrText;

    // Extracted structured data
    @Column(columnDefinition = "jsonb")
    private String extractedData;

    // Validation results
    @Column(columnDefinition = "jsonb")
    private String validationResults;

    private Float confidenceScore;
    private String processingEngine; // TESSERACT, AWS_TEXTRACT

    private Long submittedBy;
    private LocalDateTime submittedAt;
    private LocalDateTime processedAt;
    private Long processingTimeMs;
}
```

### Document Processing Service

```java
// DocumentProcessingService.java — ประมวลผลเอกสารด้วย AI
@Service
public class DocumentProcessingService {

    @Autowired
    private ChatClient chatClient;

    @Autowired
    private S3Client s3Client;

    @Autowired
    private DocumentProcessingJobRepository jobRepository;

    // เริ่มประมวลผลเอกสาร
    @Async
    public void processDocument(String jobId) {
        DocumentProcessingJob job = jobRepository.findByJobId(jobId).orElseThrow();

        try {
            job.setStatus(JobStatus.PROCESSING);
            jobRepository.save(job);

            long startTime = System.currentTimeMillis();

            // Step 1: ดาวน์โหลดไฟล์จาก S3
            byte[] fileBytes = downloadFromS3(job.getS3Key());

            // Step 2: OCR
            String ocrText = performOcr(fileBytes, job.getFileType());
            job.setOcrText(ocrText);

            // Step 3: ใช้ AI จัดประเภทเอกสาร
            DocumentType docType = classifyDocument(ocrText);
            job.setDocumentType(docType);

            // Step 4: สกัดข้อมูลตาม Document Type
            Map<String, Object> extractedData = extractStructuredData(ocrText, docType);
            job.setExtractedData(toJson(extractedData));

            // Step 5: Validate ข้อมูลที่สกัดมา
            ValidationResult validation = validateExtractedData(extractedData, docType);
            job.setValidationResults(toJson(validation));

            job.setConfidenceScore(validation.getOverallConfidence());
            job.setStatus(JobStatus.COMPLETED);
            job.setProcessingTimeMs(System.currentTimeMillis() - startTime);
            job.setProcessedAt(LocalDateTime.now());

        } catch (Exception e) {
            job.setStatus(JobStatus.FAILED);
            log.error("Failed to process document: {}", jobId, e);
        }

        jobRepository.save(job);
    }

    // ใช้ AI สกัดข้อมูล Structured จาก OCR Text
    private Map<String, Object> extractStructuredData(String ocrText, DocumentType docType) {
        String schema = getSchemaForDocumentType(docType);

        String prompt = String.format("""
                จากข้อความที่ได้จาก OCR ต่อไปนี้ กรุณาสกัดข้อมูลและตอบกลับเป็น JSON
                ตาม Schema ที่กำหนด:
                
                Schema: %s
                
                OCR Text:
                %s
                
                ตอบกลับเป็น JSON เท่านั้น ถ้าไม่พบข้อมูลให้ใส่ null
                """, schema, ocrText);

        String response = chatClient.prompt()
                .user(prompt)
                .call()
                .content();

        try {
            return objectMapper.readValue(response, Map.class);
        } catch (Exception e) {
            throw new ExtractionException("Failed to parse AI response", e);
        }
    }

    // OCR ด้วย AWS Textract
    private String performOcr(byte[] fileBytes, String fileType) {
        if ("PDF".equals(fileType)) {
            return performTextractOcr(fileBytes);
        }
        return performTesseractOcr(fileBytes);
    }

    private String performTextractOcr(byte[] fileBytes) {
        TextractClient textractClient = TextractClient.builder()
                .region(Region.AP_SOUTHEAST_1)
                .build();

        DetectDocumentTextRequest request = DetectDocumentTextRequest.builder()
                .document(Document.builder()
                        .bytes(SdkBytes.fromByteArray(fileBytes))
                        .build())
                .build();

        DetectDocumentTextResponse response = textractClient.detectDocumentText(request);

        return response.blocks().stream()
                .filter(b -> b.blockType() == BlockType.LINE)
                .map(Block::text)
                .collect(Collectors.joining("\n"));
    }

    private String getSchemaForDocumentType(DocumentType docType) {
        return switch (docType) {
            case INVOICE -> """
                {
                  "invoiceNumber": "string",
                  "invoiceDate": "date (YYYY-MM-DD)",
                  "vendorName": "string",
                  "vendorTaxId": "string",
                  "buyerName": "string",
                  "items": [{"description": "string", "quantity": "number", "unitPrice": "number", "total": "number"}],
                  "subtotal": "number",
                  "vat": "number",
                  "totalAmount": "number"
                }
                """;
            case ID_CARD -> """
                {
                  "idNumber": "string",
                  "fullName": "string",
                  "dateOfBirth": "date",
                  "address": "string",
                  "issueDate": "date",
                  "expiryDate": "date"
                }
                """;
            default -> "{}";
        };
    }
}

// DocumentController.java
@RestController
@RequestMapping("/api/documents")
public class DocumentController {

    @Autowired
    private DocumentProcessingService processingService;

    @PostMapping("/upload")
    @PreAuthorize("isAuthenticated()")
    public ResponseEntity<JobResponse> uploadDocument(
            @RequestParam("file") MultipartFile file,
            @AuthenticationPrincipal User user) throws IOException {
        DocumentProcessingJob job = processingService.submitDocument(file, user.getId());
        return ResponseEntity.status(HttpStatus.ACCEPTED).body(JobResponse.from(job));
    }

    @GetMapping("/jobs/{jobId}")
    @PreAuthorize("isAuthenticated()")
    public ResponseEntity<JobDetailResponse> getJobStatus(@PathVariable String jobId) {
        DocumentProcessingJob job = processingService.getJob(jobId);
        return ResponseEntity.ok(JobDetailResponse.from(job));
    }
}
```

---

## โปรเจค 98: AI Customer Support Bot

### ภาพรวมโปรเจค

AI Customer Support Bot เป็นระบบ Customer Support อัตโนมัติที่ใช้ Spring AI + ChatGPT รองรับ RAG (Retrieval Augmented Generation) pattern ด้วย Vector Search เพื่อดึงข้อมูลจาก Knowledge Base, เก็บ Conversation History, Escalation ให้ Human Agent และ Feedback Learning

### Knowledge Base Setup

```java
// KnowledgeBase.java — เนื้อหาสำหรับ RAG
@Entity
@Table(name = "knowledge_base")
public class KnowledgeBase {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String title;

    @Column(columnDefinition = "text", nullable = false)
    private String content;

    private String category; // FAQ, POLICY, PRODUCT, TROUBLESHOOTING

    @ElementCollection
    @CollectionTable(name = "kb_tags")
    private Set<String> tags = new HashSet<>();

    @Enumerated(EnumType.STRING)
    private KbStatus status; // DRAFT, PUBLISHED

    private Long viewCount;
    private Float helpfulRating;
    private Long helpfulVotes;

    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
}

// SupportTicket.java — Ticket เมื่อ Bot ส่งต่อให้ Human
@Entity
@Table(name = "support_tickets")
public class SupportTicket {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(unique = true, nullable = false)
    private String ticketNumber;

    private Long customerId;
    private String conversationSessionId;

    @Column(nullable = false)
    private String subject;

    @Column(columnDefinition = "text")
    private String description;

    @Enumerated(EnumType.STRING)
    private TicketPriority priority; // LOW, MEDIUM, HIGH, URGENT

    @Enumerated(EnumType.STRING)
    private TicketStatus status; // OPEN, IN_PROGRESS, RESOLVED, CLOSED

    private Long assignedAgentId;
    private LocalDateTime createdAt;
    private LocalDateTime resolvedAt;
}
```

### AI Support Service with RAG

```java
// AiSupportService.java — AI Support ด้วย RAG Pattern
@Service
public class AiSupportService {

    @Autowired
    private ChatClient chatClient;

    @Autowired
    private VectorStore vectorStore;

    @Autowired
    private KnowledgeBaseRepository kbRepository;

    @Autowired
    private ConversationHistoryService historyService;

    @Autowired
    private SupportTicketService ticketService;

    // ตอบคำถาม Customer ด้วย RAG
    public SupportResponse answerQuestion(String sessionId, String question, Long customerId) {
        // Step 1: ดึง Conversation History
        List<Message> history = historyService.getRecentHistory(sessionId, 10);

        // Step 2: ค้นหา Knowledge ที่เกี่ยวข้องจาก Vector Store (RAG)
        List<Document> relevantDocs = vectorStore.similaritySearch(
                SearchRequest.query(question)
                        .withTopK(5)
                        .withSimilarityThreshold(0.75)
        );

        // Step 3: สร้าง Context จาก Knowledge Base
        String knowledgeContext = relevantDocs.stream()
                .map(Document::getContent)
                .collect(Collectors.joining("\n\n---\n\n"));

        // Step 4: สร้าง System Prompt พร้อม Context
        String systemPrompt = String.format("""
                คุณคือ AI Customer Support ของบริษัทเรา ชื่อ "นวล"
                
                กฎการตอบ:
                1. ตอบเป็นภาษาเดียวกับที่ลูกค้าใช้
                2. ใช้ข้อมูลจาก Knowledge Base ที่ให้มาในการตอบ
                3. ถ้าไม่แน่ใจหรือไม่มีข้อมูล ให้บอกว่าจะส่งต่อให้เจ้าหน้าที่
                4. ตอบกระชับ ชัดเจน เป็นมิตร
                5. ห้ามแต่งข้อมูลที่ไม่มีใน Knowledge Base
                
                Knowledge Base:
                %s
                """, knowledgeContext);

        // Step 5: เรียก ChatGPT พร้อม Conversation History
        ChatResponse chatResponse = chatClient.prompt()
                .system(systemPrompt)
                .messages(convertToAiMessages(history))
                .user(question)
                .call()
                .chatResponse();

        String answer = chatResponse.getResult().getOutput().getContent();

        // Step 6: บันทึก Message ใน History
        historyService.saveUserMessage(sessionId, question, customerId);
        historyService.saveBotMessage(sessionId, answer);

        // Step 7: ตรวจสอบว่าต้อง Escalate ไหม
        boolean shouldEscalate = detectEscalationNeed(question, answer);

        if (shouldEscalate) {
            SupportTicket ticket = ticketService.createFromConversation(sessionId, customerId, question);
            return SupportResponse.builder()
                    .answer(answer + "\n\nฉันจะส่งต่อคำถามของคุณให้เจ้าหน้าที่ดูแล กรุณารอสักครู่")
                    .escalated(true)
                    .ticketNumber(ticket.getTicketNumber())
                    .sourceDocs(relevantDocs.stream()
                            .map(d -> d.getMetadata().get("title").toString())
                            .collect(Collectors.toList()))
                    .build();
        }

        return SupportResponse.builder()
                .answer(answer)
                .escalated(false)
                .sourceDocs(relevantDocs.stream()
                        .map(d -> d.getMetadata().get("title").toString())
                        .collect(Collectors.toList()))
                .build();
    }

    // Index Knowledge Base Articles ไปยัง Vector Store
    public void indexKnowledgeBaseArticle(KnowledgeBase article) {
        Document doc = new Document(
                article.getTitle() + "\n\n" + article.getContent(),
                Map.of(
                        "id", article.getId().toString(),
                        "title", article.getTitle(),
                        "category", article.getCategory()
                )
        );
        vectorStore.add(List.of(doc));
    }

    // Feedback Learning — บันทึกว่าคำตอบมีประโยชน์หรือไม่
    public void recordFeedback(String sessionId, boolean helpful, String comment) {
        historyService.updateLastMessageFeedback(sessionId, helpful, comment);

        // ถ้าไม่มีประโยชน์ ให้ flag สำหรับ human review
        if (!helpful) {
            reviewService.flagForReview(sessionId, comment);
        }
    }

    private boolean detectEscalationNeed(String question, String answer) {
        String lowerQ = question.toLowerCase();
        String lowerA = answer.toLowerCase();

        // Escalate ถ้าคำถามเกี่ยวกับ complaint หรือ refund
        if (lowerQ.contains("คืนเงิน") || lowerQ.contains("refund") ||
                lowerQ.contains("ร้องเรียน") || lowerQ.contains("complaint")) {
            return true;
        }

        // Escalate ถ้า bot ตอบว่าไม่แน่ใจ
        if (lowerA.contains("ไม่แน่ใจ") || lowerA.contains("ไม่ทราบ") ||
                lowerA.contains("โปรดติดต่อ")) {
            return true;
        }

        return false;
    }
}

// AiSupportController.java
@RestController
@RequestMapping("/api/support")
public class AiSupportController {

    @Autowired
    private AiSupportService supportService;

    @PostMapping("/chat")
    public ResponseEntity<SupportResponse> chat(
            @RequestBody SupportChatRequest request,
            @AuthenticationPrincipal(required = false) User user) {
        Long customerId = user != null ? user.getId() : null;
        SupportResponse response = supportService.answerQuestion(
                request.getSessionId(), request.getQuestion(), customerId);
        return ResponseEntity.ok(response);
    }

    @PostMapping("/feedback")
    public ResponseEntity<Void> submitFeedback(@RequestBody FeedbackRequest request) {
        supportService.recordFeedback(request.getSessionId(),
                request.isHelpful(), request.getComment());
        return ResponseEntity.ok().build();
    }

    @PostMapping("/knowledge-base")
    @PreAuthorize("hasRole('SUPPORT_ADMIN')")
    public ResponseEntity<KbResponse> addKnowledge(
            @RequestBody CreateKbRequest request) {
        KnowledgeBase kb = supportService.createAndIndexKnowledge(request);
        return ResponseEntity.status(HttpStatus.CREATED).body(KbResponse.from(kb));
    }
}
```

---

## โปรเจค 99: Real-time Fraud Detection

### ภาพรวมโปรเจค

Real-time Fraud Detection เป็นระบบตรวจจับการทุจริตแบบ Real-time รองรับ Transaction Scoring, การ Integrate กับ ML Model, Rules Engine (Velocity Checks, Geo-anomaly), Real-time Blocking, Case Management และ Alert System

### Entities

```java
// FraudRule.java — กฎสำหรับตรวจจับการทุจริต
@Entity
@Table(name = "fraud_rules")
public class FraudRule {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String name;

    private String description;

    @Enumerated(EnumType.STRING)
    private RuleType type; // VELOCITY_CHECK, GEO_ANOMALY, AMOUNT_THRESHOLD, PATTERN_MATCH

    // Rule definition ใน JSON
    @Column(columnDefinition = "jsonb", nullable = false)
    private String ruleDefinition;

    private Integer riskScore; // 0-100 ที่ rule นี้ contribute
    private Integer priority;

    @Enumerated(EnumType.STRING)
    private RuleAction action; // SCORE, BLOCK, FLAG

    private boolean active;
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
}

// TransactionRiskScore.java — คะแนน Risk ของ Transaction
@Entity
@Table(name = "transaction_risk_scores")
public class TransactionRiskScore {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(unique = true, nullable = false)
    private String transactionId;

    private Long userId;
    private BigDecimal amount;
    private String currency;
    private String merchantCode;
    private String ipAddress;
    private String deviceFingerprint;
    private String country;

    private Integer riskScore; // 0-100
    @Column(columnDefinition = "jsonb")
    private String triggeredRules; // รายการ rules ที่ trigger

    @Enumerated(EnumType.STRING)
    private RiskDecision decision; // APPROVE, REVIEW, BLOCK

    private String mlModelVersion;
    private Float mlScore;

    private Long processingTimeMs;
    private LocalDateTime evaluatedAt;
}

// FraudCase.java — Case สำหรับ Manual Review
@Entity
@Table(name = "fraud_cases")
public class FraudCase {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(unique = true, nullable = false)
    private String caseNumber;

    private String transactionId;
    private Long userId;
    private BigDecimal amount;
    private Integer riskScore;

    @Column(columnDefinition = "jsonb")
    private String evidence; // transactions, patterns, notes

    @Enumerated(EnumType.STRING)
    private CaseStatus status; // OPEN, IN_REVIEW, CONFIRMED_FRAUD, FALSE_POSITIVE, CLOSED

    private Long assignedTo;
    private String resolution;

    private LocalDateTime createdAt;
    private LocalDateTime resolvedAt;
}
```

### Fraud Detection Engine

```java
// FraudDetectionService.java — Real-time Fraud Detection
@Service
public class FraudDetectionService {

    @Autowired
    private FraudRuleRepository ruleRepository;

    @Autowired
    private TransactionRiskScoreRepository riskScoreRepository;

    @Autowired
    private FraudCaseRepository caseRepository;

    @Autowired
    private MlModelClient mlModelClient;

    @Autowired
    private RedisTemplate<String, Object> redisTemplate;

    @Autowired
    private AlertService alertService;

    // ประเมิน Risk ของ Transaction แบบ Real-time
    public RiskAssessmentResult assessTransaction(TransactionData transaction) {
        long startTime = System.currentTimeMillis();

        int totalRiskScore = 0;
        List<String> triggeredRules = new ArrayList<>();

        // 1. Rules Engine Evaluation
        List<FraudRule> activeRules = ruleRepository.findByActiveOrderByPriorityAsc(true);

        for (FraudRule rule : activeRules) {
            RuleEvalResult result = evaluateRule(rule, transaction);

            if (result.isTriggered()) {
                totalRiskScore += rule.getRiskScore();
                triggeredRules.add(rule.getName());

                // ถ้า rule กำหนดให้ BLOCK ทันที
                if (rule.getAction() == RuleAction.BLOCK) {
                    return buildBlockedResult(transaction, rule.getName(), totalRiskScore,
                            triggeredRules, startTime);
                }
            }
        }

        // 2. ML Model Scoring
        Float mlScore = mlModelClient.scoreTransaction(transaction);
        if (mlScore > 0.8f) {
            totalRiskScore = Math.min(totalRiskScore + 30, 100);
            triggeredRules.add("ML_HIGH_RISK_PREDICTION");
        }

        // Cap score ที่ 100
        totalRiskScore = Math.min(totalRiskScore, 100);

        // 3. ตัดสินใจ
        RiskDecision decision;
        if (totalRiskScore >= 80) {
            decision = RiskDecision.BLOCK;
            alertService.sendFraudAlert(transaction, totalRiskScore);
        } else if (totalRiskScore >= 50) {
            decision = RiskDecision.REVIEW;
            createFraudCase(transaction, totalRiskScore, triggeredRules);
        } else {
            decision = RiskDecision.APPROVE;
        }

        long processingTime = System.currentTimeMillis() - startTime;

        // บันทึกผล
        saveRiskScore(transaction, totalRiskScore, decision, triggeredRules, mlScore, processingTime);

        return RiskAssessmentResult.builder()
                .transactionId(transaction.getTransactionId())
                .riskScore(totalRiskScore)
                .decision(decision)
                .triggeredRules(triggeredRules)
                .processingTimeMs(processingTime)
                .build();
    }

    private RuleEvalResult evaluateRule(FraudRule rule, TransactionData tx) {
        return switch (rule.getType()) {
            case VELOCITY_CHECK -> checkVelocity(rule, tx);
            case GEO_ANOMALY -> checkGeoAnomaly(rule, tx);
            case AMOUNT_THRESHOLD -> checkAmountThreshold(rule, tx);
            default -> RuleEvalResult.notTriggered();
        };
    }

    // Velocity Check — ตรวจสอบความถี่การทำธุรกรรม
    private RuleEvalResult checkVelocity(FraudRule rule, TransactionData tx) {
        try {
            VelocityRule config = objectMapper.readValue(rule.getRuleDefinition(), VelocityRule.class);
            String key = "velocity:" + tx.getUserId() + ":" + config.getTimeWindowMinutes();

            Long txCount = redisTemplate.opsForValue().increment(key);
            if (txCount == 1) {
                redisTemplate.expire(key, Duration.ofMinutes(config.getTimeWindowMinutes()));
            }

            if (txCount != null && txCount > config.getMaxTransactions()) {
                return RuleEvalResult.triggered("Too many transactions: " + txCount);
            }

            return RuleEvalResult.notTriggered();

        } catch (Exception e) {
            log.error("Velocity check failed", e);
            return RuleEvalResult.notTriggered();
        }
    }

    // Geo Anomaly — ตรวจสอบตำแหน่งที่ผิดปกติ
    private RuleEvalResult checkGeoAnomaly(FraudRule rule, TransactionData tx) {
        // ดึง last transaction location
        String lastCountryKey = "last_country:" + tx.getUserId();
        String lastCountry = (String) redisTemplate.opsForValue().get(lastCountryKey);

        if (lastCountry != null && !lastCountry.equals(tx.getCountry())) {
            // Check time difference — ถ้าสั้นเกินกว่าจะเดินทางได้
            String lastTxTimeKey = "last_tx_time:" + tx.getUserId();
            String lastTxTimeStr = (String) redisTemplate.opsForValue().get(lastTxTimeKey);

            if (lastTxTimeStr != null) {
                LocalDateTime lastTxTime = LocalDateTime.parse(lastTxTimeStr);
                long minutesSinceLast = ChronoUnit.MINUTES.between(lastTxTime, LocalDateTime.now());

                if (minutesSinceLast < 60) { // น้อยกว่า 1 ชั่วโมงแต่คนละประเทศ
                    return RuleEvalResult.triggered(
                            "Geographic anomaly: " + lastCountry + " -> " + tx.getCountry() +
                                    " in " + minutesSinceLast + " minutes");
                }
            }
        }

        // อัปเดต last location
        redisTemplate.opsForValue().set(lastCountryKey, tx.getCountry(), Duration.ofDays(7));
        redisTemplate.opsForValue().set("last_tx_time:" + tx.getUserId(),
                LocalDateTime.now().toString(), Duration.ofDays(7));

        return RuleEvalResult.notTriggered();
    }
}

// FraudDetectionController.java
@RestController
@RequestMapping("/api/fraud")
public class FraudDetectionController {

    @Autowired
    private FraudDetectionService fraudDetectionService;

    // ประเมิน Transaction ก่อนอนุมัติ
    @PostMapping("/assess")
    public ResponseEntity<RiskAssessmentResult> assess(@RequestBody TransactionData transaction) {
        return ResponseEntity.ok(fraudDetectionService.assessTransaction(transaction));
    }

    // Update Case Status
    @PutMapping("/cases/{id}")
    @PreAuthorize("hasRole('FRAUD_ANALYST')")
    public ResponseEntity<CaseResponse> updateCase(
            @PathVariable Long id,
            @RequestBody UpdateCaseRequest request) {
        FraudCase fraudCase = fraudDetectionService.updateCase(id, request);
        return ResponseEntity.ok(CaseResponse.from(fraudCase));
    }

    @GetMapping("/dashboard")
    @PreAuthorize("hasRole('FRAUD_ANALYST')")
    public ResponseEntity<FraudDashboard> getDashboard() {
        return ResponseEntity.ok(fraudDetectionService.getDashboardStats());
    }
}
```

---

## โปรเจค 100: Complete Enterprise Integration Platform

### ภาพรวมโปรเจค

Enterprise Integration Platform เป็น Capstone Project ที่รวมทุกระบบที่เรียนมาใน Course นี้เข้าด้วยกัน ประกอบด้วย SSO (OAuth2), Shared Notification Service, Unified Audit Log, API Gateway, Health Dashboard, Multi-tenant Support และ Complete Microservices Architecture

### Architecture Overview

```
                    ┌─────────────────────────────────────────────┐
                    │           API Gateway (Spring Cloud Gateway)  │
                    │     Rate Limiting | Auth | Load Balancing     │
                    └─────────────────────────────────────────────┘
                                          │
              ┌───────────────────────────┼──────────────────────────┐
              │                           │                          │
    ┌─────────────────┐       ┌──────────────────┐      ┌──────────────────┐
    │  Auth Service   │       │  Tenant Service  │      │ Notification Svc │
    │  (OAuth2 SSO)   │       │  (Multi-tenant)  │      │  (Email/SMS/Push)│
    └─────────────────┘       └──────────────────┘      └──────────────────┘
              │                           │                          │
    ┌─────────────────┐       ┌──────────────────┐      ┌──────────────────┐
    │  Audit Service  │       │  Business Logic  │      │  Health Monitor  │
    │ (Unified Audit) │       │   Services...    │      │   Dashboard      │
    └─────────────────┘       └──────────────────┘      └──────────────────┘
```

### Core Integration Services

```java
// UnifiedAuditService.java — Centralized Audit Log สำหรับทุก Service
@Service
public class UnifiedAuditService {

    @Autowired
    private AuditLogRepository auditLogRepository;

    @Autowired
    private KafkaTemplate<String, AuditEvent> kafkaTemplate;

    // บันทึก Audit Event จากทุก Service
    public void recordEvent(AuditEvent event) {
        AuditLog log = new AuditLog();
        log.setEventId(UUID.randomUUID().toString());
        log.setService(event.getService());
        log.setAction(event.getAction());
        log.setResourceType(event.getResourceType());
        log.setResourceId(event.getResourceId());
        log.setUserId(event.getUserId());
        log.setTenantId(event.getTenantId());
        log.setIpAddress(event.getIpAddress());
        log.setOldValue(toJson(event.getOldValue()));
        log.setNewValue(toJson(event.getNewValue()));
        log.setSuccess(event.isSuccess());
        log.setErrorMessage(event.getErrorMessage());
        log.setTimestamp(LocalDateTime.now());

        auditLogRepository.save(log);

        // Publish ไปยัง Kafka สำหรับ Real-time processing
        kafkaTemplate.send("audit-events", event);
    }

    // Spring AOP สำหรับ Auto-audit @Auditable methods
    @Around("@annotation(auditable)")
    public Object auditMethod(ProceedingJoinPoint joinPoint, Auditable auditable) throws Throwable {
        String userId = SecurityContextHolder.getContext().getAuthentication().getName();
        Object result = null;
        boolean success = true;
        String error = null;

        try {
            result = joinPoint.proceed();
        } catch (Exception e) {
            success = false;
            error = e.getMessage();
            throw e;
        } finally {
            AuditEvent event = AuditEvent.builder()
                    .service(auditable.service())
                    .action(auditable.action())
                    .resourceType(auditable.resourceType())
                    .userId(userId)
                    .success(success)
                    .errorMessage(error)
                    .build();
            recordEvent(event);
        }

        return result;
    }
}

// SharedNotificationService.java — Centralized Notification Service
@Service
public class SharedNotificationService {

    @Autowired
    private EmailService emailService;

    @Autowired
    private SmsService smsService;

    @Autowired
    private PushNotificationService pushService;

    @Autowired
    private NotificationPreferenceRepository prefRepository;

    // ส่ง Notification ผ่านทุก Channels
    public void sendNotification(NotificationRequest request) {
        NotificationPreference pref = prefRepository.findByUserId(request.getUserId())
                .orElse(NotificationPreference.defaultPreference());

        if (pref.isEmailEnabled() && request.getChannels().contains("EMAIL")) {
            emailService.send(request.getUserEmail(), request.getSubject(),
                    request.getEmailBody());
        }

        if (pref.isSmsEnabled() && request.getChannels().contains("SMS") &&
                request.getPhone() != null) {
            smsService.send(request.getPhone(), request.getSmsBody());
        }

        if (pref.isPushEnabled() && request.getChannels().contains("PUSH")) {
            pushService.send(request.getUserId(), request.getPushTitle(),
                    request.getPushBody(), request.getPushData());
        }
    }
}

// ApiGatewayConfig.java — Spring Cloud Gateway Configuration
@Configuration
public class ApiGatewayConfig {

    @Bean
    public RouteLocator routeLocator(RouteLocatorBuilder builder) {
        return builder.routes()
                // Auth Service
                .route("auth-service", r -> r
                        .path("/api/auth/**")
                        .filters(f -> f
                                .rewritePath("/api/auth/(?<remaining>.*)", "/api/${remaining}")
                                .addResponseHeader("X-Gateway-Service", "auth"))
                        .uri("lb://auth-service"))

                // User Service
                .route("user-service", r -> r
                        .path("/api/users/**")
                        .filters(f -> f
                                .requestRateLimiter(c -> c
                                        .setRateLimiter(redisRateLimiter())
                                        .setKeyResolver(userKeyResolver()))
                                .addResponseHeader("X-Gateway-Service", "user"))
                        .uri("lb://user-service"))

                // Business Services (ตัวอย่าง)
                .route("product-service", r -> r
                        .path("/api/products/**", "/api/catalog/**")
                        .uri("lb://product-service"))

                .route("order-service", r -> r
                        .path("/api/orders/**")
                        .uri("lb://order-service"))

                .build();
    }

    @Bean
    public RedisRateLimiter redisRateLimiter() {
        return new RedisRateLimiter(10, 20); // 10 requests/sec, burst 20
    }
}

// HealthDashboardService.java — รวม Health ของทุก Service
@Service
public class HealthDashboardService {

    @Autowired
    private DiscoveryClient discoveryClient;

    @Autowired
    private WebClient webClient;

    // ดึง Health Status ของทุก Service
    public List<ServiceHealth> getAllServiceHealth() {
        List<String> services = discoveryClient.getServices();

        return services.parallelStream()
                .map(this::checkServiceHealth)
                .collect(Collectors.toList());
    }

    private ServiceHealth checkServiceHealth(String serviceName) {
        try {
            String healthUrl = "lb://" + serviceName + "/actuator/health";

            ResponseEntity<Map> response = webClient.get()
                    .uri(healthUrl)
                    .retrieve()
                    .toEntity(Map.class)
                    .block(Duration.ofSeconds(5));

            boolean isUp = response != null && response.getStatusCode().is2xxSuccessful();

            return ServiceHealth.builder()
                    .serviceName(serviceName)
                    .status(isUp ? "UP" : "DOWN")
                    .responseTimeMs(System.currentTimeMillis()) // simplified
                    .checkedAt(LocalDateTime.now())
                    .build();

        } catch (Exception e) {
            return ServiceHealth.builder()
                    .serviceName(serviceName)
                    .status("DOWN")
                    .error(e.getMessage())
                    .checkedAt(LocalDateTime.now())
                    .build();
        }
    }
}

// EnterpriseIntegrationController.java — Main Integration API
@RestController
@RequestMapping("/api/enterprise")
public class EnterpriseIntegrationController {

    @Autowired
    private HealthDashboardService healthService;

    @Autowired
    private UnifiedAuditService auditService;

    @Autowired
    private SharedNotificationService notificationService;

    @GetMapping("/health")
    public ResponseEntity<List<ServiceHealth>> getSystemHealth() {
        return ResponseEntity.ok(healthService.getAllServiceHealth());
    }

    @GetMapping("/audit-logs")
    @PreAuthorize("hasRole('SUPER_ADMIN')")
    public ResponseEntity<Page<AuditLogResponse>> getAuditLogs(
            @RequestParam(required = false) String service,
            @RequestParam(required = false) String userId,
            @RequestParam(required = false) LocalDateTime from,
            @RequestParam(required = false) LocalDateTime to,
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "50") int size) {
        return ResponseEntity.ok(auditService.search(service, userId, from, to,
                PageRequest.of(page, size)));
    }

    @PostMapping("/notifications/send")
    @PreAuthorize("hasRole('SYSTEM')")
    public ResponseEntity<Void> sendNotification(@RequestBody NotificationRequest request) {
        notificationService.sendNotification(request);
        return ResponseEntity.ok().build();
    }
}
```

### SQL Scripts for Integration Platform

```sql
-- migrations/V1__enterprise_platform.sql
CREATE TABLE audit_logs (
    id BIGSERIAL PRIMARY KEY,
    event_id VARCHAR(255) UNIQUE NOT NULL,
    service VARCHAR(100) NOT NULL,
    action VARCHAR(100) NOT NULL,
    resource_type VARCHAR(100),
    resource_id VARCHAR(255),
    user_id VARCHAR(255),
    tenant_id VARCHAR(255),
    ip_address VARCHAR(50),
    old_value JSONB,
    new_value JSONB,
    success BOOLEAN DEFAULT TRUE,
    error_message TEXT,
    timestamp TIMESTAMP DEFAULT NOW()
) PARTITION BY RANGE (timestamp);

-- สร้าง Partitions ทุกเดือน (เพิ่มประสิทธิภาพสำหรับ large data)
CREATE TABLE audit_logs_2024_01 PARTITION OF audit_logs
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');
CREATE TABLE audit_logs_2024_02 PARTITION OF audit_logs
    FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');

CREATE TABLE notification_logs (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL,
    channel VARCHAR(50) NOT NULL,
    subject VARCHAR(500),
    body TEXT,
    status VARCHAR(50) DEFAULT 'SENT',
    error_message TEXT,
    sent_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE service_health_logs (
    id BIGSERIAL PRIMARY KEY,
    service_name VARCHAR(100) NOT NULL,
    status VARCHAR(20) NOT NULL,
    response_time_ms BIGINT,
    error TEXT,
    checked_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_audit_service ON audit_logs(service, timestamp DESC);
CREATE INDEX idx_audit_user ON audit_logs(user_id, timestamp DESC);
CREATE INDEX idx_audit_tenant ON audit_logs(tenant_id, timestamp DESC);
CREATE INDEX idx_notification_user ON notification_logs(user_id, sent_at DESC);
```

---

## 🎉 ยินดีด้วย! คุณสำเร็จการศึกษา 100 Real-World Projects แล้ว!

### ฉลองความสำเร็จของคุณ

คุณได้เดินทางผ่านความท้าทายมากมายตลอด 120 Parts ของ Course นี้ และได้สร้างโปรเจคจริงถึง **100 โปรเจค** ที่ครอบคลุมทุกด้านของ Spring Boot Development ระดับ Production

### สิ่งที่คุณได้เรียนรู้

#### Foundation (โปรเจค 1-20)
ตั้งแต่ REST APIs พื้นฐาน, Spring Data JPA, Spring Security, JWT Authentication, File Upload, Email Service จนถึง Redis Caching และ Async Processing — คุณได้วางรากฐานที่แข็งแกร่ง

#### Intermediate (โปรเจค 21-50)
WebSocket, Reactive Programming, DDD, CQRS, OAuth2, Event Sourcing, Saga Pattern, GraphQL — คุณได้ก้าวสู่การออกแบบ Architecture ระดับสูง

#### Advanced (โปรเจค 51-75)
Microservices, API Gateway, Distributed Tracing, Circuit Breaker, Service Mesh, Zero-downtime Deployment, gRPC, Chaos Engineering — คุณได้เชี่ยวชาญระบบซับซ้อน

#### World-class (โปรเจค 76-100)
Multi-tenant SaaS, Headless CMS, Live Streaming, AI Integration, Fraud Detection, Enterprise Integration Platform — คุณได้สร้างระบบระดับโลก

### สถิติความสำเร็จของคุณ

| หัวข้อ | จำนวน |
|--------|-------|
| โปรเจคที่สร้าง | 100 |
| บรรทัดโค้ดที่เขียน | 50,000+ |
| Technologies ที่ใช้ | 40+ |
| Design Patterns | 25+ |
| Database Technologies | PostgreSQL, Redis, MongoDB, Elasticsearch |
| Cloud Services | AWS S3, SES, MediaConvert, Textract |
| AI/ML Integration | Spring AI, OpenAI, Vector DB |

### เส้นทางต่อไป

ตอนนี้คุณมีความรู้และทักษะที่จำเป็นสำหรับ:

1. **Senior/Lead Java Developer** — ออกแบบและสร้างระบบขนาดใหญ่
2. **Solution Architect** — วางแผน Architecture สำหรับองค์กร
3. **Tech Lead** — นำทีมในการพัฒนาระบบ Complex
4. **Startup Founder** — สร้าง SaaS Product ของตัวเอง

### คำแนะนำสุดท้าย

> "การเรียนรู้ที่แท้จริงเกิดจากการลงมือทำ อย่าหยุดสร้าง อย่าหยุดเรียนรู้ เทคโนโลยีเปลี่ยนแปลงอยู่เสมอ แต่หลักการพื้นฐานในการออกแบบระบบที่ดียังคงเหมือนเดิม — ทำให้ Simple, ทำให้ Testable, ทำให้ Scalable"

**ขอบคุณที่ศึกษา Course นี้จนจบ ขอให้โชคดีในการ Coding ครับ/ค่ะ** 🚀

---

## สรุป Part 120 และ Course ทั้งหมด

| โปรเจค | เทคโนโลยีหลัก | ความยาก |
|--------|---------------|---------|
| 96. AI Product Search | Spring AI, Vector Embeddings, RAG | ⭐⭐⭐⭐⭐ |
| 97. Document Processing | OCR (Tesseract/Textract), AI Extraction | ⭐⭐⭐⭐⭐ |
| 98. AI Support Bot | Spring AI, RAG Pattern, Vector Search | ⭐⭐⭐⭐⭐ |
| 99. Fraud Detection | ML Integration, Redis Velocity Check, Real-time | ⭐⭐⭐⭐⭐ |
| 100. Enterprise Platform | API Gateway, Unified Audit, Multi-tenant, Full Integration | ⭐⭐⭐⭐⭐ |

### Key Takeaways สุดท้าย

1. **Spring AI** — Framework ที่ทำให้การ Integrate AI models เข้ากับ Spring Boot เป็นเรื่องง่าย
2. **RAG Pattern** — Vector Search + LLM เป็น Pattern ที่ทรงพลังสำหรับ Knowledge-based AI
3. **Real-time Fraud Detection** — Redis เป็น Key สำหรับ Velocity Checks เพราะ Performance สูง
4. **Enterprise Integration** — API Gateway + Unified Audit + Shared Services = Enterprise-grade Platform
5. **AI ไม่ใช่ Silver Bullet** — AI ช่วยได้มาก แต่ต้องมี proper validation, monitoring และ fallback

*[← Part 119](./part-119-marketplace-realestate.md) | [กลับไปหน้าหลัก: README](./README.md)*

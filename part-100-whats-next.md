# Part 100: What's Next - Course Completion
## ขั้นตอนที่ 3601-3640

**ระดับ: World-Class Professional | หลักสูตรสมบูรณ์**

---

```
╔═══════════════════════════════════════════════════════════════════╗
║                                                                   ║
║   🎓 ยินดีด้วย! คุณเรียนจบหลักสูตร Spring Boot ครบ 100 ตอน!   ║
║                                                                   ║
║   จาก Beginner → Professional → World-Class Developer            ║
║                                                                   ║
╚═══════════════════════════════════════════════════════════════════╝
```

---

## ขั้นตอนที่ 3601: บทสรุปการเดินทาง 100 ตอน

### Learning Path Recap

เส้นทางที่คุณผ่านมา 100 ตอนนี้ไม่ใช่แค่การเรียน Technology แต่คือการพัฒนาตัวเองให้เป็น Developer ระดับ World-Class

```
Phase 1: Foundation (ตอนที่ 1-20)
"เริ่มต้นจากศูนย์"

├── Part 01: Introduction to Spring Boot
│   └── เข้าใจว่า Spring Boot คืออะไร ทำไมถึงสำคัญ
│
├── Part 02: Setup Environment
│   └── JDK, IDE, Maven/Gradle พร้อมใช้งาน
│
├── Part 03: First Application
│   └── "Hello, World!" ตัวแรกของคุณ
│
├── Part 04: Project Structure
│   └── เข้าใจโครงสร้าง Spring Boot Project
│
├── Part 05: REST API Basics
│   └── Controller, GET/POST/PUT/DELETE
│
├── Part 06: Dependency Injection
│   └── IoC Container, @Autowired, @Component
│
├── Part 07: Auto Configuration
│   └── Magic ของ Spring Boot ถูกเปิดเผย
│
├── Part 08: Properties & Profiles
│   └── application.properties, Dev/Prod Profile
│
├── Part 09: Logging
│   └── Logback, SLF4J, Structured Logging
│
├── Part 10: Testing Basics
│   └── JUnit 5, Mockito, @SpringBootTest
│
├── Part 11-15: Spring Data JPA
│   └── Entity, Repository, CRUD, Relationships, Validation
│
└── Part 16-20: Security & File Upload
    └── Spring Security, JWT, File Handling, Email Service

Phase 1 Milestone: สร้าง REST API ที่มี CRUD + Security ได้

---

Phase 2: Professional (ตอนที่ 21-60)
"พัฒนาทักษะอย่างจริงจัง"

├── Part 21-30: Advanced Spring Features
│   └── Caching, Scheduling, Async, Events
│
├── Part 31-40: Microservices Foundation
│   └── Spring Cloud, Eureka, Feign Client, Config Server
│
├── Part 41-50: Message Broker & Event-Driven
│   └── Kafka, RabbitMQ, Event Sourcing
│
└── Part 51-60: Performance & Testing
    └── Load Testing, Profiling, Test Containers

Phase 2 Milestone: สร้าง Microservices ที่ Production-ready ได้

---

Phase 3: World-Class (ตอนที่ 61-100)
"ระดับ Enterprise"

├── Part 61-70: Cloud Native & DevOps
│   └── Docker, Kubernetes, CI/CD, Cloud
│
├── Part 71-80: Advanced Patterns
│   └── Observability, Contract Testing, Zero-Downtime, Search
│
├── Part 81-90: Real-world Features
│   └── Payment, Real-time, Advanced Security, Performance
│
├── Part 91-95: Architecture Excellence
│   └── DDD, Clean Architecture, SAGA, CQRS
│
└── Part 96-100: Capstone & Career
    └── Capstone Design, Implementation, Interview, Career, This!

Phase 3 Milestone: Design และ Build Enterprise System ได้
```

---

## ขั้นตอนที่ 3602: Knowledge Summary

### สิ่งที่คุณรู้แล้วตอนนี้

```java
// Technology Stack ที่คุณ Master แล้ว:
public class YourSkillSet {
    
    // Core
    Java java = new Java(version: 21, features: [
        "Virtual Threads", "Records", "Sealed Classes",
        "Pattern Matching", "Text Blocks"
    ]);
    
    SpringBoot springBoot = new SpringBoot(version: "3.2", areas: [
        "Auto-configuration", "Actuator", "Testing",
        "Security", "Data", "Cloud", "Kafka"
    ]);
    
    // Data Storage
    Database database = new PolyglotPersistence(
        relational: "PostgreSQL 16",
        cache: "Redis 7",
        search: "Elasticsearch 8",
        messaging: "Apache Kafka 3.6"
    );
    
    // Cloud & DevOps
    Infrastructure infrastructure = new Infrastructure(
        container: "Docker",
        orchestration: "Kubernetes",
        cicd: "GitHub Actions",
        monitoring: "Prometheus + Grafana + Jaeger"
    );
    
    // Architecture Patterns
    Patterns patterns = new Patterns([
        "Microservices", "Event-Driven Architecture",
        "CQRS", "Event Sourcing", "SAGA",
        "Domain-Driven Design", "Clean Architecture",
        "Hexagonal Architecture"
    ]);
    
    // Testing
    Testing testing = new Testing([
        "Unit Testing (JUnit 5 + Mockito)",
        "Integration Testing (Testcontainers)",
        "Contract Testing (Pact)",
        "Load Testing (k6)",
        "E2E Testing"
    ]);
}
```

---

## ขั้นตอนที่ 3603: Next Technologies to Explore

### หลังจาก Master Spring Boot แล้ว ควรเรียนอะไรต่อ?

**1. Reactive Programming**

```java
// Spring WebFlux + Project Reactor
// สำหรับ High-throughput, Low-latency systems

@RestController
@RequestMapping("/api/v1/products")
public class ReactiveProductController {
    
    private final ReactiveProductService productService;
    
    // Flux = 0..N items
    @GetMapping
    public Flux<ProductResponse> getAllProducts() {
        return productService.findAll()
            .map(this::toResponse)
            .doOnNext(p -> log.debug("Streaming product: {}", p.getId()));
    }
    
    // Mono = 0..1 item
    @GetMapping("/{id}")
    public Mono<ProductResponse> getProduct(@PathVariable UUID id) {
        return productService.findById(id)
            .map(this::toResponse)
            .switchIfEmpty(Mono.error(new ProductNotFoundException(id)));
    }
    
    // Server-Sent Events สำหรับ Real-time Updates
    @GetMapping(value = "/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<ServerSentEvent<ProductResponse>> streamProducts() {
        return productService.findAll()
            .map(product -> ServerSentEvent.<ProductResponse>builder()
                .id(product.getId().toString())
                .event("product-update")
                .data(toResponse(product))
                .build());
    }
}
```

**2. GraalVM Native Image**

```xml
<!-- pom.xml: เพิ่ม Native Support -->
<plugin>
    <groupId>org.graalvm.buildtools</groupId>
    <artifactId>native-maven-plugin</artifactId>
</plugin>

<!-- Build Native Image -->
<!-- mvn -Pnative native:compile -->
<!-- 
ผลลัพธ์: Startup Time จาก 5s -> 0.1s
Memory จาก 256MB -> 50MB
ดีมากสำหรับ Serverless Functions
-->
```

**3. Spring AI**

```java
// AI-Powered Features ใน Spring Boot
@Service
@RequiredArgsConstructor
public class ProductRecommendationAI {
    
    private final ChatClient chatClient;
    private final VectorStore vectorStore;  // pgvector or Chroma
    
    /**
     * RAG (Retrieval Augmented Generation)
     * ผสม Product Database กับ AI
     */
    public String generateProductDescription(UUID productId) {
        Product product = productRepository.findById(productId).orElseThrow();
        
        // สร้าง Description ด้วย AI
        String prompt = String.format("""
            เขียน Product Description ภาษาไทยที่น่าสนใจ สำหรับสินค้า:
            ชื่อ: %s
            หมวดหมู่: %s
            ราคา: %s บาท
            คุณสมบัติ: %s
            
            ความยาว: 2-3 ประโยค น่าดึงดูด
            """,
            product.getName(),
            product.getCategory().getName(),
            product.getPrice(),
            product.getAttributes()
        );
        
        return chatClient.prompt()
            .user(prompt)
            .call()
            .content();
    }
    
    /**
     * Semantic Product Search
     * ค้นหาสินค้าด้วยความหมาย ไม่ใช่แค่ Keyword
     */
    public List<Product> semanticSearch(String query) {
        // Convert query to vector
        List<Document> similarDocs = vectorStore.similaritySearch(
            SearchRequest.query(query).withTopK(10)
        );
        
        List<UUID> productIds = similarDocs.stream()
            .map(doc -> UUID.fromString(doc.getMetadata().get("productId").toString()))
            .toList();
        
        return productRepository.findAllById(productIds);
    }
}
```

**4. Kotlin + Spring Boot**

```kotlin
// Kotlin ทำงานได้ดีกับ Spring Boot
// Code กระชับกว่า Java มาก

@RestController
@RequestMapping("/api/v1/products")
class ProductController(
    private val productService: ProductService
) {
    
    @GetMapping("/{id}")
    fun getProduct(@PathVariable id: UUID): ResponseEntity<ProductResponse> {
        val product = productService.findById(id)
            ?: return ResponseEntity.notFound().build()
        return ResponseEntity.ok(product.toResponse())
    }
    
    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    fun createProduct(@Valid @RequestBody request: CreateProductRequest): ProductResponse {
        return productService.createProduct(request)
    }
}

// Data Classes แทน Lombok
data class CreateProductRequest(
    @field:NotBlank val name: String,
    @field:NotNull @field:Positive val price: BigDecimal,
    val description: String? = null
)

// Extension Functions ที่สะอาดมาก
fun Product.toResponse() = ProductResponse(
    id = this.id,
    name = this.name,
    price = this.price,
    status = this.status
)
```

---

## ขั้นตอนที่ 3604: Community Resources

### เข้าร่วม Community

**GitHub:**
```
Organizations ที่น่าติดตาม:
- spring-projects (Spring Official)
- spring-attic (Older Spring Projects)
- spring-cloud (Spring Cloud)
- micrometer-metrics (Metrics)
- testcontainers (Testing)

Topics to follow:
- #spring-boot
- #java
- #microservices
- #kubernetes
```

**Stack Overflow:**
```
Tags ที่ควร Follow:
- [spring-boot]
- [spring-security]
- [spring-data-jpa]
- [kafka]
- [microservices]

เป้าหมาย:
- ตอบคำถาม 1 ข้อต่อวัน
- สร้าง Reputation 10,000+ ใน 2 ปี
- Badge: Gold in [spring-boot]
```

**Blogs ที่ต้องอ่าน:**
```
1. spring.io/blog - Official Spring Blog
   - Release Notes ทุก Version
   - Best Practices
   - Community Spotlights

2. baeldung.com
   - Spring Boot Deep Dives
   - Java Tutorials
   - คุณภาพสูงมาก

3. reflectoring.io
   - Practical Spring Boot
   - Testing Guides

4. blog.sebastian-daschner.com
   - Enterprise Java Patterns

5. microservices.io
   - Microservices Patterns (Chris Richardson)
```

---

## ขั้นตอนที่ 3605: Recommended Books

### หนังสือที่ต้องอ่านต่อ

```
ระดับ Intermediate:
📚 Spring Boot in Action (Craig Walls)
   - ครอบคลุม Spring Boot ทั้งหมด

📚 Spring Security in Action (Laurentiu Spilca)
   - Security ลึกมาก

📚 Cloud Native Spring in Action (Thomas Vitale)
   - Spring + Cloud Native

---

ระดับ Advanced:
📚 Designing Data-Intensive Applications (Martin Kleppmann)
   - ดีที่สุดสำหรับ System Design
   - Database, Streaming, Distributed Systems

📚 Building Microservices (Sam Newman, 2nd Edition)
   - Comprehensive Microservices Guide

📚 Release It! (Michael Nygard, 2nd Edition)
   - Production-ready Software Patterns

📚 Clean Architecture (Robert C. Martin)
   - Software Architecture Principles

---

ระดับ Leadership:
📚 The Staff Engineer's Path (Tanya Reilly)
   - สำหรับ Senior → Staff transition

📚 An Elegant Puzzle (Will Larson)
   - Engineering Management

📚 Accelerate (Nicole Forsgren et al.)
   - DevOps and High Performance Teams
```

---

## ขั้นตอนที่ 3606: Final Project Ideas

### Portfolio Projects ที่ควรสร้าง

**Project 1: Microservices E-Commerce (จาก Capstone)**
```
สิ่งที่ต้องมี:
✅ 5+ Microservices
✅ Event-driven Architecture
✅ JWT Authentication
✅ Kubernetes Deployment
✅ CI/CD Pipeline
✅ Monitoring Dashboard
✅ API Documentation
✅ 80%+ Test Coverage

GitHub Repository Structure:
ecommerce-platform/
├── README.md (Architecture Diagram, Setup Guide)
├── docker-compose.yml
├── kubernetes/
├── user-service/
├── product-service/
├── order-service/
├── inventory-service/
├── payment-service/
└── docs/
    ├── api.yaml (OpenAPI)
    └── architecture.png
```

**Project 2: Real-time Stock Market Dashboard**
```java
// WebSocket + Kafka + Redis
@Component
public class StockPriceSimulator {
    
    private final KafkaTemplate<String, StockUpdate> kafkaTemplate;
    private final Random random = new Random();
    
    @Scheduled(fixedDelay = 100)  // ทุก 100ms
    public void simulateStockUpdates() {
        List<String> stocks = List.of("PTT", "SCB", "KBANK", "AOT", "CPALL");
        
        stocks.forEach(symbol -> {
            StockUpdate update = StockUpdate.builder()
                .symbol(symbol)
                .price(getNewPrice(symbol))
                .change(random.nextDouble(-5, 5))
                .volume(random.nextLong(1000, 100000))
                .timestamp(Instant.now())
                .build();
            
            kafkaTemplate.send("stock.updates", symbol, update);
        });
    }
}

@Controller
public class StockWebSocketController {
    
    @MessageMapping("/subscribe/{symbol}")
    @SendToUser("/queue/stock-updates")
    public Flux<StockUpdate> subscribeToStock(@DestinationVariable String symbol) {
        return stockService.getStockStream(symbol);
    }
}
```

**Project 3: Developer Tools - API Mock Server**
```java
// สร้าง Tool ที่ Developer อื่นใช้
@RestController
@RequestMapping("/api/mock")
public class MockApiController {
    
    /**
     * Dynamic Mock API
     * Developer กำหนด Response ที่ต้องการได้
     */
    @PostMapping("/configure")
    public MockConfiguration configure(@RequestBody MockConfig config) {
        // บันทึก Config ว่า Path นี้ต้อง Return อะไร
        return mockService.addConfiguration(config);
    }
    
    // Dynamic Endpoint ที่รับทุก Method, ทุก Path
    @RequestMapping(value = "/{path:**}", method = {
        RequestMethod.GET, RequestMethod.POST,
        RequestMethod.PUT, RequestMethod.DELETE
    })
    public ResponseEntity<Object> handleMockRequest(
        HttpServletRequest request,
        @PathVariable String path
    ) {
        return mockService.handleRequest(request, path);
    }
}
```

---

## ขั้นตอนที่ 3607: 30-60-90 Day Plan After Course

### แผนการพัฒนาหลังเรียนจบ

```
30 วันแรก: Consolidate Learning
├── ทบทวน 5 Topics ที่ยังไม่ชัดเจน
├── Build 1 Personal Project โดยใช้สิ่งที่เรียน
├── Setup GitHub Profile ให้น่าสนใจ
└── เริ่มเขียน Blog Post แรก

30-60 วัน: Apply and Share
├── Contribute 1 Open Source Issue
├── เข้าร่วม Local Developer Meetup
├── Publish 2 Blog Posts
└── สร้าง Demo Video สำหรับ Portfolio Project

60-90 วัน: Level Up
├── สมัคร Job ที่ใช้ Skills ที่เรียนมา
├── เตรียมตัวสัมภาษณ์ด้วย Part 98
├── Network กับ Senior Developers
└── เริ่ม Learning Path ถัดไป (Reactive หรือ AI)
```

---

## ขั้นตอนที่ 3608: Measuring Your Progress

### Self-Assessment Checklist

```
Junior Level (ทำได้ทั้งหมดก่อนจะเลื่อนขั้น):
□ Build REST API ด้วย Spring Boot จาก Scratch
□ Implement CRUD ด้วย Spring Data JPA
□ เพิ่ม Spring Security + JWT Authentication
□ เขียน Unit Tests ด้วย JUnit + Mockito
□ Deploy ด้วย Docker

Mid-level (เพิ่มเติม):
□ Design Event-Driven System ด้วย Kafka
□ Implement Circuit Breaker Pattern
□ ทำ Database Optimization (ดู Explain Plan)
□ ใช้ Redis Cache อย่างถูกต้อง
□ เขียน Integration Tests ด้วย Testcontainers

Senior Level (เพิ่มเติม):
□ Design Microservices Architecture
□ Implement SAGA/CQRS Pattern
□ ทำ Performance Profiling และแก้ไข
□ Lead Code Review ได้
□ Write Architecture Decision Records

Staff Level (เพิ่มเติม):
□ Drive Technical Standards ใน Organization
□ Mentor Senior Engineers ได้
□ Present Technical Vision ต่อ Leadership
□ Contribute to Open Source Project
□ Speak at Conference/Meetup
```

---

## ขั้นตอนที่ 3609-3640: Final Words and Celebration

### คำอวยพรสำหรับการเดินทางต่อ

```
╔═══════════════════════════════════════════════════════════════════════════╗
║                                                                           ║
║   ขอแสดงความยินดีอย่างจริงใจกับทุกคนที่เรียนจบหลักสูตรนี้!           ║
║                                                                           ║
║   คุณได้ผ่านการเดินทาง 100 ตอน                                          ║
║   จาก "สวัสดี, World!" ไปสู่การออกแบบระบบระดับ Enterprise               ║
║                                                                           ║
║   สิ่งที่คุณทำได้ตอนนี้:                                                ║
║   ✅ Build Microservices ที่ Production-ready                             ║
║   ✅ Design Systems ที่รองรับ Millions of Users                          ║
║   ✅ Implement Security Best Practices                                    ║
║   ✅ Optimize Performance ที่ Scale                                      ║
║   ✅ Lead Technical Decisions                                             ║
║                                                                           ║
╚═══════════════════════════════════════════════════════════════════════════╝
```

### สิ่งที่สำคัญที่สุดที่ได้เรียนรู้

```
1. Technology เปลี่ยนแปลงเสมอ แต่ Principles ยังคงอยู่
   SOLID, DRY, KISS ยังคงสำคัญไม่ว่าจะใช้ Framework อะไร

2. Code ที่ดีคือ Code ที่อ่านง่าย ไม่ใช่ Code ที่เขียนเร็ว
   Write for the next developer, not just for the computer

3. Testing ไม่ใช่ทางเลือก แต่คือ Professional Standard
   คนที่ไม่เขียน Test คือคนที่ไม่ Professional

4. Performance Optimization ต้องวัดก่อนแก้
   Premature optimization is the root of all evil

5. Communication Skills สำคัญเท่ากับ Technical Skills
   ความสามารถในการอธิบาย Technical Concept ให้คนอื่นเข้าใจ

6. การแบ่งปันความรู้ทำให้คุณเก่งขึ้น ไม่ใช่อ่อนแอลง
   Teaching is the best way to learn

7. Career Growth ไม่ใช่ Race แต่เป็น Journey ของตัวเอง
   ทุกคนมีเส้นทางและ Timeline ของตัวเอง
```

### Code ที่จะพาคุณไปไกล

```java
/**
 * The Mindset of a World-Class Developer
 * 
 * หลักการที่ต้องพกติดตัวตลอดอาชีพ
 */
public interface WorldClassDeveloper {
    
    /**
     * เรียนรู้อยู่เสมอ - Technology เปลี่ยนทุกวัน
     * ถ้าหยุดเรียน คือเริ่มถอยหลัง
     */
    void continuousLearning();
    
    /**
     * แบ่งปันความรู้ - Community ที่ดีสร้างจาก Giving
     * Blog, OSS, Mentoring, Speaking
     */
    void shareKnowledge();
    
    /**
     * รับผิดชอบต่อ Code ของตัวเอง
     * Own your failures, Learn from them
     */
    void takeOwnership();
    
    /**
     * ให้ความสำคัญกับ User ก่อน Technology
     * Technology เป็นเครื่องมือ ไม่ใช่เป้าหมาย
     */
    void userFirst();
    
    /**
     * ช่วยเหลือเพื่อนร่วมทีม
     * Team Success > Individual Success
     */
    void liftOthersUp();
}
```

### เส้นทางต่อจากนี้

```
Short-term (1-3 เดือน):
→ Build Capstone Project ให้สมบูรณ์
→ Deploy บน Kubernetes
→ Publish บน GitHub

Medium-term (3-12 เดือน):
→ Contribute to Open Source
→ Write 10 Technical Blog Posts
→ Speak at 1 Meetup
→ Get 1 Star on GitHub Project

Long-term (1-5 ปี):
→ Become recognized Expert in Java/Spring Boot
→ Lead Technical Team
→ Influence Technical Decisions
→ Build Products that Impact Real Users

Dream (5+ ปี):
→ Create Open Source Library ที่คนอื่นใช้
→ Speak at International Conference
→ Mentor Next Generation of Developers
→ Build Company หรือ Product ของตัวเอง
```

---

### คำขอบคุณ

```
ขอบคุณที่ศรัทธาในหลักสูตรนี้และใช้เวลาศึกษา

การเรียนรู้ 100 ตอนไม่ใช่เรื่องง่าย มีหลายครั้งที่รู้สึกท้อ
แต่คุณยังคงเดินต่อ - นั่นคือสิ่งที่ทำให้คุณแตกต่างจากคนอื่น

"The journey of a thousand miles begins with a single step"
- Lao Tzu

คุณเริ่มก้าวแรกแล้ว อย่าหยุด

Good luck on your journey to become a World-Class Developer! 🚀
```

---

## Quick Reference: Key Commands

```bash
# Spring Boot
mvn spring-boot:run
mvn spring-boot:build-image  # Build Docker Image
mvn test                      # Run Tests
mvn verify -Pintegration-test # Run Integration Tests

# Docker
docker build -t myapp:latest .
docker run -p 8080:8080 myapp:latest
docker-compose up -d

# Kubernetes
kubectl apply -f kubernetes/
kubectl get pods -n ecommerce-prod
kubectl logs -f deployment/order-service
kubectl rollout status deployment/order-service
kubectl rollout undo deployment/order-service  # Rollback

# Git
git log --oneline --graph   # Beautiful history
git bisect start            # Binary search for bug
git blame -L 10,20 file.java  # Who wrote this?

# Database
psql -h localhost -U postgres
\l                           # List databases
\c mydb                      # Connect to database
\d table_name                # Describe table
EXPLAIN ANALYZE SELECT ...;  # Query plan

# Kafka
kafka-topics.sh --list --bootstrap-server localhost:9092
kafka-console-consumer.sh --topic order.events --from-beginning
kafka-consumer-groups.sh --describe --group order-service

# Redis
redis-cli
MONITOR                      # Watch all commands
INFO memory                  # Memory stats
KEYS pattern*                # Find keys (ระวัง Production!)
SCAN 0 MATCH pattern* COUNT 100  # Better alternative
```

---

## Resources Summary

```
Official Documentation:
- spring.io/docs
- docs.spring.io/spring-boot/docs/current/reference/html/
- docs.spring.io/spring-security/reference/

Community:
- stackoverflow.com/questions/tagged/spring-boot
- github.com/spring-projects/spring-boot/discussions
- reddit.com/r/SpringBoot

Learning:
- spring.academy (Official Spring Learning Platform)
- baeldung.com (Best Spring Tutorial Site)
- reflectoring.io (Practical Spring Boot)

Tools:
- start.spring.io (Project Generator)
- editor.swagger.io (API Design)
- excalidraw.com (Architecture Diagrams)
- dbdiagram.io (ERD Design)
```

---

## จบแล้ว! แต่การเรียนรู้ยังไม่จบ...

```
╔═══════════════════════════════════════════════════════╗
║                                                       ║
║   The best code you'll ever write                     ║
║   is the code you haven't written yet.               ║
║                                                       ║
║   Keep building. Keep learning. Keep sharing.        ║
║                                                       ║
║   See you in the next course!                        ║
║                                                       ║
╚═══════════════════════════════════════════════════════╝
```

---

*[← Part 99: Career Growth](./part-99-career-growth.md) | [กลับไปตอนแรก: Part 01: Introduction →](./part-01-introduction.md)*

---

**หลักสูตร Spring Boot: จาก Beginner สู่ World-Class Professional**
*100 ตอน | 3,640 ขั้นตอน | ครบทุก Skill ที่ต้องใช้ใน Production*

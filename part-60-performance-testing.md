# Part 60: Performance Testing
## ขั้นตอนที่ 2001-2040

**ระดับ:** ระดับโลก (World-Class)
**เวลาเรียน:** 6-7 ชั่วโมง
**เป้าหมาย:** เชี่ยวชาญการทดสอบ Performance ด้วย k6, Gatling, JMeter และการวิเคราะห์ JVM Memory สำหรับ Spring Boot Applications

---

## ขั้นตอนที่ 2001-2005: ทำความเข้าใจ Performance Testing

### ประเภทของ Performance Testing

| ประเภท | คำอธิบาย | วัตถุประสงค์ |
|--------|-----------|-------------|
| **Load Test** | ทดสอบด้วย load ปกติ | ตรวจสอบ response time ที่ normal load |
| **Stress Test** | เพิ่ม load จนเกินความสามารถ | หา breaking point |
| **Spike Test** | เพิ่ม load อย่างรวดเร็ว | ทดสอบ auto-scaling |
| **Soak Test** | ทดสอบนานๆ | หา memory leak |
| **Volume Test** | ทดสอบกับข้อมูลจำนวนมาก | ตรวจสอบ DB performance |

### Key Metrics ที่ต้องวัด

- **Throughput** (RPS) - requests per second
- **Response Time** - P50, P95, P99, P99.9
- **Error Rate** - % ของ failed requests
- **Concurrency** - จำนวน concurrent users
- **CPU/Memory Usage** - resource consumption

---

## ขั้นตอนที่ 2006-2013: k6 Load Testing

### k6 Installation และ Basic Test

```bash
# ติดตั้ง k6
brew install k6                        # macOS
choco install k6                       # Windows
apt-get install k6                     # Ubuntu/Debian

# รัน test
k6 run script.js
k6 run --vus 100 --duration 30s script.js
```

### k6 Load Test Script

```javascript
// k6-load-test.js
import http from 'k6/http';
import { check, sleep, group } from 'k6';
import { Counter, Rate, Trend, Gauge } from 'k6/metrics';

// Custom metrics
const orderCreatedRate = new Rate('order_created_rate');
const paymentSuccessRate = new Rate('payment_success_rate');
const apiLatency = new Trend('api_latency', true);
const activeUsers = new Gauge('active_users');

// Test configuration
export const options = {
  stages: [
    { duration: '1m', target: 50 },   // Ramp up to 50 VUs in 1 min
    { duration: '3m', target: 50 },   // Stay at 50 VUs for 3 mins
    { duration: '1m', target: 100 },  // Ramp up to 100 VUs
    { duration: '3m', target: 100 },  // Stay at 100 VUs
    { duration: '2m', target: 0 },    // Ramp down
  ],
  
  thresholds: {
    'http_req_duration': [
      'p(95)<500',          // 95th percentile ต้องน้อยกว่า 500ms
      'p(99)<1000',         // 99th percentile ต้องน้อยกว่า 1000ms
    ],
    'http_req_failed': ['rate<0.01'],  // Error rate ต้องน้อยกว่า 1%
    'order_created_rate': ['rate>0.95'], // Order success rate ต้องมากกว่า 95%
  },
};

const BASE_URL = __ENV.BASE_URL || 'http://localhost:8080';

// Shared data
const products = ['PROD-001', 'PROD-002', 'PROD-003', 'PROD-004', 'PROD-005'];
const users = ['user-001', 'user-002', 'user-003', 'user-004', 'user-005'];

export default function () {
  const userId = users[Math.floor(Math.random() * users.length)];
  const productId = products[Math.floor(Math.random() * products.length)];
  
  activeUsers.add(1);
  
  group('Browse Products', () => {
    // List products
    const listResponse = http.get(`${BASE_URL}/api/v1/products`, {
      headers: { 'Accept': 'application/json' },
    });
    
    check(listResponse, {
      'products list status 200': (r) => r.status === 200,
      'products list has items': (r) => JSON.parse(r.body).length > 0,
    });
    
    apiLatency.add(listResponse.timings.duration, { endpoint: 'list_products' });
    
    sleep(0.5);
    
    // Get single product
    const productResponse = http.get(`${BASE_URL}/api/v1/products/${productId}`);
    
    check(productResponse, {
      'product detail status 200': (r) => r.status === 200,
      'product has name': (r) => JSON.parse(r.body).name !== undefined,
    });
    
    apiLatency.add(productResponse.timings.duration, { endpoint: 'get_product' });
    sleep(1);
  });
  
  group('Create Order', () => {
    const orderPayload = JSON.stringify({
      userId: userId,
      productId: productId,
      quantity: Math.floor(Math.random() * 3) + 1,
      idempotencyKey: `order-${__VU}-${__ITER}`,
    });
    
    const startTime = Date.now();
    const orderResponse = http.post(
      `${BASE_URL}/api/v1/orders`,
      orderPayload,
      {
        headers: {
          'Content-Type': 'application/json',
          'X-Idempotency-Key': `order-${__VU}-${__ITER}`,
        },
      }
    );
    
    const orderCreated = check(orderResponse, {
      'order created 201': (r) => r.status === 201,
      'order has id': (r) => {
        try {
          return JSON.parse(r.body).orderId !== undefined;
        } catch (e) {
          return false;
        }
      },
    });
    
    orderCreatedRate.add(orderCreated);
    apiLatency.add(Date.now() - startTime, { endpoint: 'create_order' });
    
    if (!orderCreated) {
      console.log(`Order failed: ${orderResponse.status} - ${orderResponse.body}`);
    }
    
    sleep(2);
  });
  
  activeUsers.add(-1);
}

// Setup ก่อนเริ่ม test
export function setup() {
  const healthResponse = http.get(`${BASE_URL}/actuator/health`);
  if (healthResponse.status !== 200) {
    throw new Error('Application is not healthy!');
  }
  console.log('Application health check passed');
  return { startTime: new Date().toISOString() };
}

// Teardown หลัง test
export function teardown(data) {
  console.log(`Test started at: ${data.startTime}`);
  console.log(`Test ended at: ${new Date().toISOString()}`);
}
```

### k6 Stress Test

```javascript
// k6-stress-test.js
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  stages: [
    { duration: '2m', target: 100 },   // Normal load
    { duration: '5m', target: 100 },   // Hold normal
    { duration: '2m', target: 200 },   // Scale up
    { duration: '5m', target: 200 },   // Hold high
    { duration: '2m', target: 300 },   // Stress
    { duration: '5m', target: 300 },   // Hold stress
    { duration: '2m', target: 400 },   // Breaking point
    { duration: '5m', target: 400 },   // Hold breaking point
    { duration: '5m', target: 0 },     // Recovery
  ],
  
  thresholds: {
    'http_req_failed': ['rate<0.1'],   // Stress test: อนุญาต 10% errors
    'http_req_duration': ['p(95)<2000'], // 95th percentile ต้องน้อยกว่า 2000ms
  },
};

const BASE_URL = __ENV.BASE_URL || 'http://localhost:8080';

export default function () {
  const res = http.get(`${BASE_URL}/api/v1/products`);
  
  check(res, {
    'status is 200': (r) => r.status === 200,
    'response time < 1s': (r) => r.timings.duration < 1000,
  });
  
  sleep(1);
}
```

### k6 Spike Test

```javascript
// k6-spike-test.js
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  stages: [
    { duration: '1m', target: 10 },    // Normal
    { duration: '10s', target: 500 },  // Sudden spike!
    { duration: '2m', target: 500 },   // Hold spike
    { duration: '10s', target: 10 },   // Drop back
    { duration: '2m', target: 10 },    // Recovery
  ],
};

const BASE_URL = __ENV.BASE_URL || 'http://localhost:8080';

export default function () {
  const res = http.get(`${BASE_URL}/api/v1/products`);
  
  check(res, {
    'status is 200': (r) => r.status === 200,
  });
  
  sleep(0.1);
}
```

---

## ขั้นตอนที่ 2014-2019: Gatling Simulation

### Gatling Setup

```xml
<!-- pom.xml - Gatling plugin -->
<build>
  <plugins>
    <plugin>
      <groupId>io.gatling</groupId>
      <artifactId>gatling-maven-plugin</artifactId>
      <version>4.6.0</version>
      <configuration>
        <simulationsFolder>src/test/scala/simulations</simulationsFolder>
        <resultsFolder>target/gatling</resultsFolder>
      </configuration>
    </plugin>
  </plugins>
</build>

<dependencies>
  <dependency>
    <groupId>io.gatling.highcharts</groupId>
    <artifactId>gatling-charts-highcharts</artifactId>
    <version>3.9.5</version>
    <scope>test</scope>
  </dependency>
</dependencies>
```

### Gatling Simulation (Scala)

```scala
// src/test/scala/simulations/ProductCatalogSimulation.scala
package simulations

import io.gatling.core.Predef._
import io.gatling.http.Predef._
import scala.concurrent.duration._

class ProductCatalogSimulation extends Simulation {

  // HTTP Protocol configuration
  val httpProtocol = http
    .baseUrl("http://localhost:8080")
    .acceptHeader("application/json")
    .contentTypeHeader("application/json")
    .acceptEncodingHeader("gzip, deflate")
    .userAgentHeader("Gatling/3.9")
    .header("X-Request-ID", session => java.util.UUID.randomUUID().toString)

  // Test data feeder
  val productFeeder = csv("data/products.csv").random
  val userFeeder = csv("data/users.csv").circular

  // Scenarios
  val browseProducts = scenario("Browse Products")
    .exec(
      http("List Products")
        .get("/api/v1/products")
        .check(status.is(200))
        .check(jsonPath("$[*]").count.gte(1))
        .check(responseTimeInMillis.lte(500))
    )
    .pause(1.second, 3.seconds)
    .feed(productFeeder)
    .exec(
      http("Get Product Detail")
        .get("/api/v1/products/${productId}")
        .check(status.is(200))
        .check(jsonPath("$.name").exists)
        .check(jsonPath("$.price").exists)
    )
    .pause(2.seconds)

  val createOrder = scenario("Create Order")
    .feed(userFeeder)
    .feed(productFeeder)
    .exec(
      http("Create Order")
        .post("/api/v1/orders")
        .body(StringBody(
          """{
            "userId": "${userId}",
            "productId": "${productId}",
            "quantity": 1,
            "idempotencyKey": "${userId}-${productId}-${__threadId}"
          }"""
        ))
        .check(status.in(201, 409))  // 201 = created, 409 = insufficient stock
        .check(jsonPath("$.orderId").optional.saveAs("orderId"))
    )
    .pause(1.second)
    .doIf(session => session.contains("orderId")) {
      exec(
        http("Get Order Status")
          .get("/api/v1/orders/${orderId}")
          .check(status.is(200))
          .check(jsonPath("$.status").in("PENDING", "PROCESSING", "COMPLETED"))
      )
    }

  val searchProducts = scenario("Search Products")
    .exec(
      http("Search Electronics")
        .get("/api/v1/products?category=electronics&keyword=phone")
        .check(status.is(200))
        .check(responseTimeInMillis.lte(200))
    )
    .pause(1.second)
    .exec(
      http("Search Clothing")
        .get("/api/v1/products?category=clothing")
        .check(status.is(200))
    )

  // Setup - รัน scenarios พร้อมกัน
  setUp(
    browseProducts.inject(
      atOnceUsers(10),                           // 10 users ทันที
      rampUsers(50).during(1.minute),            // เพิ่มเป็น 50 ใน 1 นาที
      constantUsersPerSec(20).during(3.minutes), // 20 users/s นาน 3 นาที
    ),
    
    createOrder.inject(
      nothingFor(30.seconds),                    // รอ 30 วินาทีก่อน
      rampUsers(30).during(2.minutes),
      constantUsersPerSec(10).during(3.minutes),
    ),
    
    searchProducts.inject(
      rampUsersPerSec(1).to(20).during(1.minute),
      constantUsersPerSec(20).during(3.minutes),
    ),
  )
  .protocols(httpProtocol)
  .assertions(
    global.responseTime.percentile(95).lt(500),   // P95 < 500ms
    global.responseTime.percentile(99).lt(1000),  // P99 < 1000ms
    global.successfulRequests.percent.gte(99),    // Success rate >= 99%
    global.requestsPerSec.gte(50),                // ต้องรับได้อย่างน้อย 50 RPS
  )
}
```

### รัน Gatling

```bash
# รัน simulation
mvn gatling:test

# รัน specific simulation
mvn gatling:test -Dgatling.simulationClass=simulations.ProductCatalogSimulation

# ดู report ที่ target/gatling/<simulation-name>/index.html
open target/gatling/ProductCatalogSimulation*/index.html
```

---

## ขั้นตอนที่ 2020-2025: JMeter Test Plans

### JMeter Configuration (XML)

```xml
<!-- ProductAPILoadTest.jmx -->
<?xml version="1.0" encoding="UTF-8"?>
<jmeterTestPlan version="1.2" properties="5.0" jmeter="5.6.2">
  <hashTree>
    <TestPlan testname="Product API Load Test" enabled="true">
      <stringProp name="TestPlan.comments">Spring Boot API Performance Test</stringProp>
      
      <hashTree>
        <!-- Thread Group - Normal Load -->
        <ThreadGroup testname="Normal Load - Browse" enabled="true">
          <stringProp name="ThreadGroup.num_threads">50</stringProp>
          <stringProp name="ThreadGroup.ramp_time">60</stringProp>
          <stringProp name="ThreadGroup.duration">180</stringProp>
          <boolProp name="ThreadGroup.same_user_on_next_iteration">true</boolProp>
          
          <hashTree>
            <!-- HTTP Config -->
            <ConfigTestElement testname="HTTP Config" enabled="true">
              <stringProp name="HTTPSampler.domain">localhost</stringProp>
              <stringProp name="HTTPSampler.port">8080</stringProp>
              <stringProp name="HTTPSampler.protocol">http</stringProp>
            </ConfigTestElement>
            
            <!-- List Products -->
            <HTTPSamplerProxy testname="List Products" enabled="true">
              <stringProp name="HTTPSampler.path">/api/v1/products</stringProp>
              <stringProp name="HTTPSampler.method">GET</stringProp>
              <boolProp name="HTTPSampler.use_keepalive">true</boolProp>
            </HTTPSamplerProxy>
            
            <!-- Response Assertion -->
            <ResponseAssertion testname="Response Code 200" enabled="true">
              <stringProp name="Assertion.test_field">Assertion.response_code</stringProp>
              <stringProp name="Assertion.custom_message">Expected 200 OK</stringProp>
              <collectionProp name="Asserion.test_strings">
                <stringProp name="49586">200</stringProp>
              </collectionProp>
            </ResponseAssertion>
            
            <!-- Duration Assertion -->
            <DurationAssertion testname="Response Time < 500ms" enabled="true">
              <stringProp name="DurationAssertion.duration">500</stringProp>
            </DurationAssertion>
            
            <!-- Think Time -->
            <UniformRandomTimer testname="Think Time" enabled="true">
              <stringProp name="ConstantTimer.delay">1000</stringProp>
              <stringProp name="RandomTimer.range">2000</stringProp>
            </UniformRandomTimer>
          </hashTree>
        </ThreadGroup>
        
        <!-- Listeners -->
        <ResultCollector testname="Aggregate Report" enabled="true">
          <boolProp name="ResultCollector.error_logging">false</boolProp>
          <objProp>
            <name>saveConfig</name>
            <value class="SampleSaveConfiguration">
              <time>true</time>
              <latency>true</latency>
              <responseCode>true</responseCode>
              <responseMessage>true</responseMessage>
              <threadName>true</threadName>
              <dataType>true</dataType>
              <encoding>false</encoding>
              <assertions>true</assertions>
              <subresults>true</subresults>
              <responseData>false</responseData>
            </value>
          </objProp>
          <stringProp name="filename">results/aggregate-report.csv</stringProp>
        </ResultCollector>
      </hashTree>
    </TestPlan>
  </hashTree>
</jmeterTestPlan>
```

### JMeter CLI

```bash
# รัน JMeter แบบ headless
jmeter -n -t ProductAPILoadTest.jmx \
  -l results/test-results.csv \
  -e -o results/report/ \
  -Jhost=localhost \
  -Jport=8080

# ดู Dashboard
open results/report/index.html
```

---

## ขั้นตอนที่ 2026-2030: Spring Boot Profiling ด้วย async-profiler

### ตั้งค่า Spring Boot สำหรับ Profiling

```yaml
# application-profiling.yml
management:
  endpoints:
    web:
      exposure:
        include: "*"
  endpoint:
    health:
      show-details: always

spring:
  jpa:
    show-sql: true
    properties:
      hibernate:
        generate_statistics: true
        
logging:
  level:
    org.hibernate.stat: DEBUG
    org.springframework.jdbc: DEBUG
```

### Async-Profiler Integration

```java
// ProfilingController.java
@RestController
@RequestMapping("/internal/profiling")
@Profile("profiling")
@Slf4j
public class ProfilingController {
    
    @Value("${profiling.output-dir:/tmp/profiles}")
    private String outputDir;
    
    private volatile AsyncProfiler profiler;
    
    @PostConstruct
    void init() {
        try {
            profiler = AsyncProfiler.getInstance();
        } catch (Exception e) {
            log.warn("async-profiler not available: {}", e.getMessage());
        }
    }
    
    /**
     * เริ่ม CPU profiling
     */
    @PostMapping("/start/cpu")
    public ResponseEntity<String> startCpuProfiling(
            @RequestParam(defaultValue = "60") int durationSeconds) {
        
        if (profiler == null) {
            return ResponseEntity.badRequest().body("async-profiler not available");
        }
        
        try {
            String outputFile = outputDir + "/cpu-" + 
                System.currentTimeMillis() + ".html";
            
            profiler.execute(
                "start,event=cpu,file=" + outputFile + ",interval=1000000"
            );
            
            // Auto-stop หลัง duration
            CompletableFuture.delayedExecutor(durationSeconds, TimeUnit.SECONDS)
                .execute(() -> {
                    try {
                        profiler.execute("stop");
                        log.info("CPU profiling saved to: {}", outputFile);
                    } catch (Exception e) {
                        log.error("Failed to stop profiling", e);
                    }
                });
            
            return ResponseEntity.ok("CPU profiling started. Output: " + outputFile);
            
        } catch (Exception e) {
            return ResponseEntity.internalServerError()
                .body("Failed to start profiling: " + e.getMessage());
        }
    }
    
    /**
     * เริ่ม Allocation profiling
     */
    @PostMapping("/start/alloc")
    public ResponseEntity<String> startAllocProfiling() {
        if (profiler == null) {
            return ResponseEntity.badRequest().body("async-profiler not available");
        }
        
        try {
            String outputFile = outputDir + "/alloc-" + 
                System.currentTimeMillis() + ".html";
            
            profiler.execute("start,event=alloc,file=" + outputFile);
            
            return ResponseEntity.ok("Allocation profiling started. Output: " + outputFile);
        } catch (Exception e) {
            return ResponseEntity.internalServerError()
                .body("Failed: " + e.getMessage());
        }
    }
    
    /**
     * หยุด profiling
     */
    @PostMapping("/stop")
    public ResponseEntity<String> stopProfiling() {
        if (profiler == null) {
            return ResponseEntity.badRequest().body("async-profiler not available");
        }
        
        try {
            profiler.execute("stop");
            return ResponseEntity.ok("Profiling stopped");
        } catch (Exception e) {
            return ResponseEntity.internalServerError()
                .body("Failed: " + e.getMessage());
        }
    }
}
```

### Async-Profiler Command Line

```bash
# ดาวน์โหลด async-profiler
wget https://github.com/async-profiler/async-profiler/releases/download/v3.0/async-profiler-3.0-linux-x64.tar.gz
tar xzf async-profiler-3.0-linux-x64.tar.gz

# หา PID ของ Spring Boot
PID=$(jps | grep MyApplication | cut -d' ' -f1)

# CPU Profiling 30 วินาที
./profiler.sh -d 30 -f /tmp/cpu-profile.html $PID

# Allocation Profiling
./profiler.sh -e alloc -d 30 -f /tmp/alloc-profile.html $PID

# Lock Profiling
./profiler.sh -e lock -d 30 -f /tmp/lock-profile.html $PID

# Wall-clock Profiling (ดู thread blocking)
./profiler.sh -e wall -d 30 -f /tmp/wall-profile.html $PID

# JVM flags สำหรับ JIT profiling
java -XX:+UnlockDiagnosticVMOptions \
     -XX:+PreserveFramePointer \
     -agentpath:/path/to/libasyncProfiler.so=start,event=cpu,file=/tmp/profile.html \
     -jar myapp.jar
```

---

## ขั้นตอนที่ 2031-2034: JVM Memory Analysis

### JVM Flags สำหรับ Production

```bash
# JVM flags แนะนำสำหรับ Spring Boot
java \
  # Heap size
  -Xms2g \
  -Xmx4g \
  
  # G1GC (แนะนำสำหรับ >= Java 11)
  -XX:+UseG1GC \
  -XX:MaxGCPauseMillis=200 \
  -XX:G1HeapRegionSize=16m \
  
  # GC Logging
  -Xlog:gc*:file=/var/log/app/gc.log:time,uptime:filecount=10,filesize=100m \
  
  # OOM Handling
  -XX:+HeapDumpOnOutOfMemoryError \
  -XX:HeapDumpPath=/var/log/app/heapdump.hprof \
  -XX:+ExitOnOutOfMemoryError \
  
  # JFR (Java Flight Recorder)
  -XX:+FlightRecorder \
  -XX:StartFlightRecording=duration=60s,filename=/var/log/app/recording.jfr \
  
  # Spring Boot optimizations
  -Dspring.jmx.enabled=false \
  -Dfile.encoding=UTF-8 \
  
  -jar myapp.jar
```

### Memory Analysis Tools

```bash
# Heap dump analysis ด้วย jmap
jmap -dump:format=b,file=heapdump.hprof $PID

# Histogram ของ objects ใน heap
jmap -histo $PID | head -50

# GC Stats
jstat -gc $PID 1000 10  # ทุก 1 วินาที 10 ครั้ง
jstat -gcutil $PID 1000 10

# Thread dump
jstack $PID > threaddump.txt

# JVM stats
jcmd $PID VM.native_memory summary
jcmd $PID GC.heap_info
```

### Memory Leak Detection

```java
// MemoryLeakDetector.java
@Component
@Slf4j
public class MemoryLeakDetector {
    
    private final MeterRegistry meterRegistry;
    
    public MemoryLeakDetector(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
        setupMemoryMetrics();
    }
    
    private void setupMemoryMetrics() {
        // Heap usage
        Gauge.builder("jvm.memory.heap.used")
            .register(meterRegistry)
            .set(() -> {
                MemoryMXBean memoryBean = ManagementFactory.getMemoryMXBean();
                return (double) memoryBean.getHeapMemoryUsage().getUsed() / (1024 * 1024);
            });
        
        // Non-heap
        Gauge.builder("jvm.memory.nonheap.used")
            .register(meterRegistry)
            .set(() -> {
                MemoryMXBean memoryBean = ManagementFactory.getMemoryMXBean();
                return (double) memoryBean.getNonHeapMemoryUsage().getUsed() / (1024 * 1024);
            });
        
        // GC stats
        ManagementFactory.getGarbageCollectorMXBeans().forEach(gcBean -> {
            Gauge.builder("jvm.gc.collection.count")
                .tag("gc", gcBean.getName())
                .register(meterRegistry)
                .set((double) gcBean.getCollectionCount());
            
            Gauge.builder("jvm.gc.collection.time.ms")
                .tag("gc", gcBean.getName())
                .register(meterRegistry)
                .set((double) gcBean.getCollectionTime());
        });
    }
    
    @Scheduled(fixedRate = 60000)
    public void checkForMemoryLeak() {
        MemoryMXBean memBean = ManagementFactory.getMemoryMXBean();
        MemoryUsage heapUsage = memBean.getHeapMemoryUsage();
        
        double usedMB = (double) heapUsage.getUsed() / (1024 * 1024);
        double maxMB = (double) heapUsage.getMax() / (1024 * 1024);
        double usagePercent = (usedMB / maxMB) * 100;
        
        log.info("Heap Usage: {:.1f}MB / {:.1f}MB ({:.1f}%)",
            usedMB, maxMB, usagePercent);
        
        if (usagePercent > 85) {
            log.warn("HIGH HEAP USAGE: {:.1f}% - potential memory leak!", usagePercent);
        }
        
        if (usagePercent > 95) {
            log.error("CRITICAL HEAP USAGE: {:.1f}% - OOM imminent!", usagePercent);
            // Trigger heap dump
            triggerHeapDump();
        }
    }
    
    private void triggerHeapDump() {
        try {
            String fileName = "/tmp/emergency-heapdump-" + 
                System.currentTimeMillis() + ".hprof";
            
            HotSpotDiagnosticMXBean bean = ManagementFactory
                .newPlatformMXBeanProxy(
                    ManagementFactory.getPlatformMBeanServer(),
                    "com.sun.management:type=HotSpotDiagnostic",
                    HotSpotDiagnosticMXBean.class
                );
            
            bean.dumpHeap(fileName, true);
            log.error("Emergency heap dump saved to: {}", fileName);
            
        } catch (IOException e) {
            log.error("Failed to create heap dump", e);
        }
    }
}
```

---

## ขั้นตอนที่ 2035-2038: Identifying Bottlenecks

### Slow Query Detection

```java
// SlowQueryInterceptor.java
@Component
@Slf4j
public class SlowQueryInterceptor implements MethodInterceptor {
    
    private static final long SLOW_QUERY_THRESHOLD_MS = 500;
    private final MeterRegistry meterRegistry;
    
    public SlowQueryInterceptor(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
    }
    
    @Override
    public Object invoke(MethodInvocation invocation) throws Throwable {
        long startTime = System.currentTimeMillis();
        
        try {
            Object result = invocation.proceed();
            long duration = System.currentTimeMillis() - startTime;
            
            recordQueryMetric(invocation.getMethod().getName(), duration, false);
            
            if (duration > SLOW_QUERY_THRESHOLD_MS) {
                log.warn("SLOW QUERY DETECTED: {} took {}ms",
                    invocation.getMethod().getDeclaringClass().getSimpleName() + 
                    "." + invocation.getMethod().getName(),
                    duration);
            }
            
            return result;
            
        } catch (Exception e) {
            long duration = System.currentTimeMillis() - startTime;
            recordQueryMetric(invocation.getMethod().getName(), duration, true);
            throw e;
        }
    }
    
    private void recordQueryMetric(String methodName, long duration, boolean error) {
        Timer.builder("db.query.duration")
            .tag("method", methodName)
            .tag("error", String.valueOf(error))
            .register(meterRegistry)
            .record(duration, TimeUnit.MILLISECONDS);
    }
}
```

### N+1 Query Detection

```java
// HibernateStatisticsMonitor.java
@Component
@Slf4j
public class HibernateStatisticsMonitor {
    
    private final Statistics hibernateStats;
    private final MeterRegistry meterRegistry;
    
    public HibernateStatisticsMonitor(EntityManagerFactory entityManagerFactory,
                                       MeterRegistry meterRegistry) {
        this.hibernateStats = entityManagerFactory.unwrap(SessionFactory.class)
            .getStatistics();
        this.hibernateStats.setStatisticsEnabled(true);
        this.meterRegistry = meterRegistry;
    }
    
    @Scheduled(fixedRate = 30000)
    public void reportStatistics() {
        long queryCount = hibernateStats.getQueryExecutionCount();
        long loadCount = hibernateStats.getEntityLoadCount();
        long flushCount = hibernateStats.getFlushCount();
        
        log.info("Hibernate Stats - Queries: {}, Entity loads: {}, Flushes: {}",
            queryCount, loadCount, flushCount);
        
        // ตรวจสอบ N+1
        String[] slowQueries = hibernateStats.getQueries();
        if (slowQueries.length > 0) {
            log.warn("Slow queries detected: {}", slowQueries.length);
            for (String query : slowQueries) {
                log.warn("  Slow query: {}", query);
            }
        }
        
        // Cache hit rate
        double secondLevelCacheHitRatio = hibernateStats.getSecondLevelCacheHitCount() > 0
            ? (double) hibernateStats.getSecondLevelCacheHitCount() / 
              (hibernateStats.getSecondLevelCacheHitCount() + 
               hibernateStats.getSecondLevelCacheMissCount())
            : 0.0;
        
        log.info("L2 Cache Hit Rate: {:.2f}%", secondLevelCacheHitRatio * 100);
        
        meterRegistry.gauge("hibernate.query.count", queryCount);
        meterRegistry.gauge("hibernate.cache.hit_rate", secondLevelCacheHitRatio);
    }
}
```

---

## ขั้นตอนที่ 2039-2040: Performance Optimization Workflow

### Performance Optimization Checklist

```java
// PerformanceOptimizationService.java
@Service
@Slf4j
public class PerformanceOptimizationService {
    
    /**
     * 1. Connection Pool Optimization
     * HikariCP Configuration
     */
    @Bean
    public DataSource optimizedDataSource() {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl("jdbc:postgresql://localhost:5432/mydb");
        config.setUsername("postgres");
        config.setPassword("password");
        
        // Pool size = (Core count * 2) + effective_spindle_count
        int coreCount = Runtime.getRuntime().availableProcessors();
        int poolSize = coreCount * 2 + 1;
        
        config.setMaximumPoolSize(poolSize);
        config.setMinimumIdle(poolSize / 2);
        config.setConnectionTimeout(30000);
        config.setIdleTimeout(600000);
        config.setMaxLifetime(1800000);
        config.setConnectionTestQuery("SELECT 1");
        
        // Performance settings
        config.addDataSourceProperty("cachePrepStmts", "true");
        config.addDataSourceProperty("prepStmtCacheSize", "250");
        config.addDataSourceProperty("prepStmtCacheSqlLimit", "2048");
        config.addDataSourceProperty("useServerPrepStmts", "true");
        
        log.info("Connection pool configured with size: {}", poolSize);
        
        return new HikariDataSource(config);
    }
    
    /**
     * 2. Caching Strategy
     * Cache frequently accessed, rarely changed data
     */
    @Bean
    public CacheManager optimizedCacheManager(RedisConnectionFactory factory) {
        RedisCacheConfiguration config = RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(10))
            .serializeKeysWith(RedisSerializationContext.SerializationPair
                .fromSerializer(new StringRedisSerializer()))
            .serializeValuesWith(RedisSerializationContext.SerializationPair
                .fromSerializer(new GenericJackson2JsonRedisSerializer()))
            .disableCachingNullValues();
        
        Map<String, RedisCacheConfiguration> configs = new HashMap<>();
        configs.put("products", config.entryTtl(Duration.ofMinutes(30)));
        configs.put("categories", config.entryTtl(Duration.ofHours(1)));
        configs.put("users", config.entryTtl(Duration.ofMinutes(5)));
        
        return RedisCacheManager.builder(factory)
            .cacheDefaults(config)
            .withInitialCacheConfigurations(configs)
            .build();
    }
}
```

### Performance Testing Pipeline

```yaml
# .github/workflows/performance-test.yml
name: Performance Tests

on:
  push:
    branches: [main]
  schedule:
    - cron: '0 2 * * *'  # ทุกวันตี 2

jobs:
  performance-test:
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_DB: perftest
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
        ports:
          - 5432:5432
          
      redis:
        image: redis:7
        ports:
          - 6379:6379
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up JDK 21
        uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
      
      - name: Build application
        run: mvn package -DskipTests
      
      - name: Start application
        run: |
          java -jar target/myapp.jar \
            -Dspring.profiles.active=perftest \
            -Xms512m -Xmx1g &
          sleep 30  # รอให้ start
          curl --retry 10 --retry-delay 3 http://localhost:8080/actuator/health
      
      - name: Install k6
        run: |
          sudo gpg -k
          sudo gpg --no-default-keyring --keyring /usr/share/keyrings/k6-archive-keyring.gpg \
            --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys C5AD17C747E3415A3642D57D77C6C491D6AC1D69
          echo "deb [signed-by=/usr/share/keyrings/k6-archive-keyring.gpg] https://dl.k6.io/deb stable main" \
            | sudo tee /etc/apt/sources.list.d/k6.list
          sudo apt-get update
          sudo apt-get install k6
      
      - name: Run k6 load test
        run: |
          k6 run \
            --out json=k6-results.json \
            --summary-export=k6-summary.json \
            performance-tests/k6-load-test.js
      
      - name: Check performance thresholds
        run: |
          # ตรวจสอบว่า test ผ่าน threshold
          python3 scripts/check-perf-results.py k6-summary.json
      
      - name: Upload results
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: performance-results
          path: |
            k6-results.json
            k6-summary.json
```

### Performance Monitoring Dashboard

```java
// PerformanceDashboardController.java
@RestController
@RequestMapping("/internal/performance")
@Profile("monitoring")
@Slf4j
public class PerformanceDashboardController {
    
    private final MeterRegistry meterRegistry;
    
    public PerformanceDashboardController(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
    }
    
    @GetMapping("/summary")
    public Map<String, Object> getPerformanceSummary() {
        Map<String, Object> summary = new LinkedHashMap<>();
        
        // HTTP metrics
        Timer requestTimer = meterRegistry.find("http.server.requests").timer();
        if (requestTimer != null) {
            summary.put("http", Map.of(
                "count", requestTimer.count(),
                "p50_ms", requestTimer.percentile(0.50, TimeUnit.MILLISECONDS),
                "p95_ms", requestTimer.percentile(0.95, TimeUnit.MILLISECONDS),
                "p99_ms", requestTimer.percentile(0.99, TimeUnit.MILLISECONDS),
                "max_ms", requestTimer.max(TimeUnit.MILLISECONDS)
            ));
        }
        
        // JVM metrics
        MemoryMXBean memBean = ManagementFactory.getMemoryMXBean();
        MemoryUsage heap = memBean.getHeapMemoryUsage();
        summary.put("jvm", Map.of(
            "heap_used_mb", heap.getUsed() / (1024 * 1024),
            "heap_max_mb", heap.getMax() / (1024 * 1024),
            "heap_usage_percent", (double) heap.getUsed() / heap.getMax() * 100,
            "threads", ManagementFactory.getThreadMXBean().getThreadCount()
        ));
        
        // Database metrics
        Gauge dbPoolActive = meterRegistry.find("hikaricp.connections.active").gauge();
        Gauge dbPoolIdle = meterRegistry.find("hikaricp.connections.idle").gauge();
        if (dbPoolActive != null) {
            summary.put("database", Map.of(
                "pool_active", dbPoolActive.value(),
                "pool_idle", dbPoolIdle != null ? dbPoolIdle.value() : 0
            ));
        }
        
        // Cache metrics
        Counter cacheHits = meterRegistry.find("cache.gets")
            .tag("result", "hit").counter();
        Counter cacheMisses = meterRegistry.find("cache.gets")
            .tag("result", "miss").counter();
        
        if (cacheHits != null && cacheMisses != null) {
            double total = cacheHits.count() + cacheMisses.count();
            summary.put("cache", Map.of(
                "hit_rate", total > 0 ? cacheHits.count() / total : 0.0,
                "hits", cacheHits.count(),
                "misses", cacheMisses.count()
            ));
        }
        
        summary.put("timestamp", Instant.now());
        return summary;
    }
    
    @GetMapping("/recommendations")
    public List<String> getOptimizationRecommendations() {
        List<String> recommendations = new ArrayList<>();
        
        // ตรวจสอบ Cache Hit Rate
        Counter cacheHits = meterRegistry.find("cache.gets").tag("result", "hit").counter();
        Counter cacheMisses = meterRegistry.find("cache.gets").tag("result", "miss").counter();
        
        if (cacheHits != null && cacheMisses != null) {
            double total = cacheHits.count() + cacheMisses.count();
            double hitRate = total > 0 ? cacheHits.count() / total : 0.0;
            
            if (hitRate < 0.5) {
                recommendations.add("Cache hit rate ต่ำ (" + 
                    String.format("%.1f%%", hitRate * 100) + 
                    ") - พิจารณาเพิ่ม TTL หรือ cache more data");
            }
        }
        
        // ตรวจสอบ Heap Usage
        MemoryMXBean memBean = ManagementFactory.getMemoryMXBean();
        MemoryUsage heap = memBean.getHeapMemoryUsage();
        double heapUsage = (double) heap.getUsed() / heap.getMax();
        
        if (heapUsage > 0.8) {
            recommendations.add("Heap usage สูง (" + 
                String.format("%.1f%%", heapUsage * 100) + 
                ") - พิจารณาเพิ่ม -Xmx หรือหา memory leak");
        }
        
        // ตรวจสอบ P99 Latency
        Timer reqTimer = meterRegistry.find("http.server.requests").timer();
        if (reqTimer != null) {
            double p99Ms = reqTimer.percentile(0.99, TimeUnit.MILLISECONDS);
            if (p99Ms > 1000) {
                recommendations.add("P99 latency สูง (" + 
                    String.format("%.0f", p99Ms) + "ms) " +
                    "- ตรวจสอบ slow queries และ blocking operations");
            }
        }
        
        if (recommendations.isEmpty()) {
            recommendations.add("ระบบทำงานได้ดี ไม่พบ bottleneck ที่ชัดเจน");
        }
        
        return recommendations;
    }
}
```

---

## สรุป Part 60

ในส่วนนี้เราได้เรียนรู้:

1. **Performance Testing Types** - Load, Stress, Spike, Soak Testing
2. **k6** - Load/Stress/Spike test scripts, custom metrics, thresholds
3. **Gatling** - Scala simulation, advanced scenarios, assertions
4. **JMeter** - Test plans, headless execution
5. **async-profiler** - CPU, memory, lock profiling
6. **JVM Memory Analysis** - GC tuning, heap dumps, memory leak detection
7. **Bottleneck Identification** - Slow queries, N+1, connection pools
8. **CI/CD Integration** - Automated performance testing pipeline

---

## สิ่งที่ต้องทำต่อ

หลังจากจบ Parts 56-60 แล้ว คุณควรจะสามารถ:

- ออกแบบ Distributed Systems ที่ปลอดภัยจาก Race Conditions
- ทดสอบความแข็งแกร่งด้วย Chaos Engineering
- ใช้ Hexagonal Architecture สร้างโค้ดที่ Testable
- เขียน Reactive Systems ด้วย Spring WebFlux
- ทดสอบ Performance และ Optimize ระบบได้อย่างมืออาชีพ

---

*[← Part 59: Reactive Patterns](./part-59-reactive-patterns.md) | [Part 61: Coming Soon →](./)*
